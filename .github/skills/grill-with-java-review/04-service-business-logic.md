# Reference: Service / Business Logic Layer — eclipse-regional-platform

## Table of Contents
1. Service Layer Conventions
2. Domain Model (DDD-lite)
3. Mapping (Entity ↔ Domain ↔ DTO)
4. Idempotency, Outbox, and Saga
5. Money, Time, Locale
6. Configuration via `@ConfigurationProperties`
7. Concurrency and Async Execution
8. Anti-patterns Catalogue

---

## 1. Service Layer Conventions

### Checklist
- [ ] `@Service` classes live in `application` package; **all** orchestration / business logic lives here
- [ ] Constructor injection only; field is `private final`
- [ ] Each service has a single responsibility; aim for ≤ 250 LoC per class
- [ ] Public methods accept domain types or DTOs — **never** JPA entities from outside the persistence boundary
- [ ] Public methods return DTOs / domain objects — never `Optional<Entity>` to callers outside the package
- [ ] `@Transactional` boundaries are explicit (see [03-data-persistence.md](03-data-persistence.md))
- [ ] Cross-cutting concerns (logging, metrics, authorization) handled via annotations / aspects — not inline
- [ ] Service throws domain exceptions (`BusinessException`, `NotFoundException`) — never `RuntimeException` with a string

```java
// ✅ CORRECT — thin orchestration, explicit ports
@Service
@RequiredArgsConstructor
@Transactional
class TransferService {

    private final AccountRepository accounts;
    private final LedgerPort ledger;
    private final OutboxPublisher outbox;
    private final TransferMapper mapper;

    @PreAuthorize("@tenantGuard.canAccess(#cmd.fromAccountId())")
    public TransferView execute(TransferCommand cmd, String idempotencyKey) {
        return idempotency.runOnce(idempotencyKey, () -> {
            Account from = accounts.loadForUpdate(cmd.fromAccountId()).orElseThrow(NotFoundException::new);
            Account to   = accounts.loadForUpdate(cmd.toAccountId()).orElseThrow(NotFoundException::new);
            Transfer t = from.transferTo(to, cmd.amount());      // domain rule
            ledger.post(t);
            outbox.publish(new TransferCompleted(t.id()));        // transactional outbox
            return mapper.toView(t);
        });
    }
}
```

---

## 2. Domain Model (DDD-lite)

### Checklist
- [ ] Domain classes in `domain` package have **zero Spring / JPA / Jackson dependencies** — pure Java
- [ ] Invariants enforced inside the aggregate root constructor / methods (`transferTo`, `credit`, `close`)
  — not in the service layer
- [ ] Value objects (`Money`, `AccountNumber`, `Iban`) are immutable records with self-validation
- [ ] Domain events are records implementing a `DomainEvent` marker; raised inside the aggregate and
  published after commit (via outbox / `ApplicationEventPublisher.publishEvent`)
- [ ] No leak of infrastructure types (`Page`, `Pageable`, `ResponseEntity`) into the domain layer
- [ ] No anaemic model — entities own their behaviour; services orchestrate them

```java
// ✅ CORRECT — Money value object with invariants
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.scale() > currency.getDefaultFractionDigits())
            throw new IllegalArgumentException("Scale exceeds currency");
    }
    public Money add(Money other) {
        require(currency.equals(other.currency), "Currency mismatch");
        return new Money(amount.add(other.amount), currency);
    }
}
```

---

## 3. Mapping (Entity ↔ Domain ↔ DTO)

### Checklist
- [ ] **MapStruct** (`@Mapper(componentModel = "spring")`) used for all non-trivial mappings; generated code reviewed
- [ ] No `BeanUtils.copyProperties` anywhere — silent and untyped
- [ ] Mappers live next to the layer they output (`api` package for view DTOs, `infrastructure` for entity adapters)
- [ ] Field omissions explicit via `@Mapping(target = "x", ignore = true)` — visible in code review
- [ ] No conditional logic in mappers; transformation should be pure — branching belongs in the service

---

## 4. Idempotency, Outbox, and Saga

### Idempotency Checklist
- [ ] Every state-mutating endpoint accepts `Idempotency-Key` header (see [02-api-rest.md](02-api-rest.md))
- [ ] Server stores `(key, requestHash, response, status)` in Redis (TTL ≥ 24h) or DB table with unique index on `(tenantId, key)`
- [ ] Same key + same body → return cached response with same status
- [ ] Same key + different body → 422 `IDEMPOTENCY_CONFLICT`
- [ ] In-flight request with same key → block / 409 (do not double-execute)

### Transactional Outbox Checklist
- [ ] Any service that mutates DB **and** publishes to Kafka uses the **outbox pattern** — never a direct
  Kafka publish inside `@Transactional` (publish-before-commit risk)
- [ ] `outbox_event` table written in the same TX as the business change
- [ ] Separate poller / Debezium CDC ships events to Kafka; deletes / marks rows processed
- [ ] Consumer deduplicates by `event_id`

### Saga Checklist (for multi-service workflows)
- [ ] Long-running flows modelled as a saga (orchestration or choreography) — never a synchronous chain of HTTP calls
- [ ] Each step has a compensating action; failures trigger compensation
- [ ] Saga state persisted (DB / Camunda / Temporal)
- [ ] Timeouts and dead-letter handling defined per step

---

## 5. Money, Time, Locale

### Checklist
- [ ] All money is `BigDecimal` + ISO-4217 currency; carried as the `Money` value object
- [ ] Rounding mode explicit (`HALF_EVEN` for bankers' rounding) — never default
- [ ] Scale validated against currency's `defaultFractionDigits`
- [ ] FX rates fetched from the rate service with effective-time; no hardcoded rates
- [ ] All time is `Instant` (UTC) at storage and transport; convert to `ZoneId` only at presentation
- [ ] `Clock` injected (`@Bean Clock clock() { return Clock.systemUTC(); }`) — never `Instant.now()` directly in code
  under test
- [ ] User-facing locale derived from the `Accept-Language` header or tenant config — never hardcoded `en_US`

---

## 6. Configuration via `@ConfigurationProperties`

### Checklist
- [ ] Typed configuration via `@ConfigurationProperties` records — no `@Value("${...}")` sprinkled across services
- [ ] `@Validated` on the properties class; fields have Bean Validation annotations
- [ ] One properties class per bounded context; nested records for grouping
- [ ] No mutable static config; properties bean is immutable
- [ ] `application-<env>.yml` provides overrides only — never the canonical schema

```java
// ✅ CORRECT
@ConfigurationProperties(prefix = "transfer")
@Validated
public record TransferProperties(
    @NotNull @DurationMin(seconds = 1) Duration timeout,
    @NotNull @DecimalMin("0.01") BigDecimal minAmount,
    @NotNull @DecimalMax("1000000.00") BigDecimal maxAmount,
    @NotNull Retry retry) {

    public record Retry(@Min(0) int maxAttempts, @NotNull Duration backoff) { }
}
```

---

## 7. Concurrency and Async Execution

### Checklist
- [ ] `@Async` methods use a **named, bounded** `ThreadPoolTaskExecutor` — never the default
- [ ] Thread pool sized via config; rejection policy explicit (`CallerRunsPolicy` for backpressure or `AbortPolicy`)
- [ ] No `CompletableFuture.supplyAsync` without an explicit executor
- [ ] Virtual threads (`Executors.newVirtualThreadPerTaskExecutor`) considered for IO-bound work on JDK 21+
- [ ] `MDC` propagated across async boundaries (`TaskDecorator` configured)
- [ ] No shared mutable state between threads; use immutable records or `ConcurrentHashMap`
- [ ] `synchronized` rare; prefer concurrent collections or explicit locks with timeout
- [ ] Scheduled tasks (`@Scheduled`) marked `@SchedulerLock` (ShedLock) in clustered deployments — never run on every pod

---

## 8. Anti-patterns Catalogue

| Anti-pattern                                       | Correct Alternative                                          |
| -------------------------------------------------- | ------------------------------------------------------------ |
| Business logic inside controller                   | Move to `@Service`; controller is HTTP-only                  |
| Anaemic entity + huge service method               | Push invariants into the aggregate root                      |
| `BeanUtils.copyProperties(src, dst)`               | MapStruct mapper                                             |
| Direct Kafka publish inside `@Transactional`       | Transactional outbox + separate publisher                    |
| `double` for money                                 | `BigDecimal` inside a `Money` value object                   |
| `Instant.now()` in code under test                 | Inject `Clock`; use `Clock.fixed(...)` in tests              |
| `@Value("${x}")` scattered everywhere              | `@ConfigurationProperties` record                            |
| `@Async` on default executor                       | Named bounded `ThreadPoolTaskExecutor`                       |
| `@Scheduled` in a multi-pod deployment without lock | ShedLock `@SchedulerLock`                                    |
| Self-invocation of `@Transactional` method         | Call through an injected proxy or split into two beans       |
| Domain class depending on Spring / JPA / Jackson   | Keep `domain` package framework-free                         |


----------------------------------------------------------------

