# Reference: API / REST Layer — eclipse-regional-platform

## Table of Contents
1. Controller Conventions
2. DTOs and Validation
3. Exception Handling and Error Envelope
4. OpenAPI / Swagger
5. Pagination, Sorting, Filtering
6. Idempotency and Concurrency
7. Versioning and Deprecation
8. Anti-patterns Catalogue

---

## 1. Controller Conventions

### Checklist
- [ ] One `@RestController` per resource / aggregate; URL begins with `/api/v{n}/...`
- [ ] Controllers contain **no business logic** — translate HTTP ↔ service call only
- [ ] Constructor injection (`@RequiredArgsConstructor`) — no `@Autowired` field injection
- [ ] HTTP methods used per REST semantics: GET (read), POST (create / RPC for actions),
  PUT (full replace), PATCH (partial), DELETE (remove)
- [ ] Status codes correct: 200 (read), 201 + `Location` (create), 202 (async accepted),
  204 (no body), 400 (validation), 401 (unauth), 403 (forbidden), 404 (not found),
  409 (conflict), 422 (semantic), 429 (rate limit), 5xx for server errors only
- [ ] Always returns `ResponseEntity<DTO>` or a typed DTO — never `Object`, `Map`, or JPA entity
- [ ] No `HttpServletRequest` / `HttpServletResponse` parameters unless absolutely required

```java
// ✅ CORRECT
@RestController
@RequestMapping("/api/v1/accounts")
@RequiredArgsConstructor
@Validated
class AccountController {
    private final AccountService service;

    @PostMapping
    ResponseEntity<AccountView> create(@Valid @RequestBody CreateAccountRequest req) {
        AccountView v = service.create(req);
        return ResponseEntity.created(URI.create("/api/v1/accounts/" + v.id())).body(v);
    }
}
```

---

## 2. DTOs and Validation

### Checklist
- [ ] DTOs are **records** (Java 17+) — immutable by default; no Lombok `@Data` on DTOs
- [ ] **Never** expose JPA entities through the API — use a dedicated DTO + MapStruct mapper
- [ ] **Property-level authorization (API3)**: separate request / response DTOs; never bind admin-only
  fields (`role`, `tenantId`, `internalNotes`) on a customer-facing endpoint
- [ ] Bean Validation annotations on every field: `@NotBlank`, `@NotNull`, `@Size`, `@Pattern`,
  `@Email`, `@Min`, `@Max`, `@DecimalMin`, `@Digits`, `@Past`, `@Future`
- [ ] Cross-field validation via custom `@Constraint` annotations or method-level `@AssertTrue`
- [ ] `@Validated` at the class level for path/query parameter validation; `@Valid` on `@RequestBody`
- [ ] Money types use `BigDecimal` with `@Digits(integer=16, fraction=2)` — never `double` / `float`
- [ ] Date-time fields use `Instant` / `OffsetDateTime` / `LocalDate` — never `Date`
- [ ] Currency carried as ISO 4217 code (`@Pattern("^[A-Z]{3}$")`) alongside amount

```java
// ✅ CORRECT — record DTO + validation
public record CreateAccountRequest(
    @NotBlank @Size(max = 140) String displayName,
    @NotNull AccountType type,
    @NotNull @Pattern(regexp = "^[A-Z]{3}$") String currency,
    @NotNull @DecimalMin("0.00") @Digits(integer = 16, fraction = 2) BigDecimal openingBalance) { }
```

### Anti-patterns
```java
// ❌ Exposing entity directly — leaks DB columns, audit fields, version
@GetMapping("/{id}") AccountEntity get(@PathVariable UUID id) { return repo.findById(id).orElseThrow(); }

// ❌ Mass assignment via copyProperties
BeanUtils.copyProperties(req, entity); // copies anything the client sends

// ❌ Money as double
double amount;
```

---

## 3. Exception Handling and Error Envelope

### Checklist
- [ ] Single `@RestControllerAdvice` per application maps exceptions → standardised body
- [ ] Body conforms to RFC 7807 `application/problem+json` (Spring's `ProblemDetail`) or the project's
  `ErrorResponse` envelope — but never leaks stack traces, SQL, or class names
- [ ] `MethodArgumentNotValidException` returns 400 with field-level errors (no internals)
- [ ] `ConstraintViolationException` returns 400; `DataIntegrityViolationException` mapped to
  409 with a generic message — never echo SQL state
- [ ] Domain exceptions extend a sealed hierarchy (`BusinessException`, `NotFoundException`,
  `ConflictException`); each maps to a deterministic HTTP code
- [ ] Generic `Exception` handler returns 500 with a correlation id and **no** technical detail;
  full stack trace logged via SLF4J at ERROR
- [ ] Correlation id (traceId) attached to every error response and to MDC

```java
// ✅ CORRECT
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    ProblemDetail notFound(NotFoundException e) {
        return ProblemDetail.forStatusAndDetail(NOT_FOUND, e.getCode());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail validation(MethodArgumentNotValidException e) {
        var pd = ProblemDetail.forStatusAndDetail(BAD_REQUEST, "Validation failed");
        pd.setProperty("errors", e.getBindingResult().getFieldErrors().stream()
            .map(f -> Map.of("field", f.getField(), "message", f.getDefaultMessage())).toList());
        return pd;
    }

    @ExceptionHandler(Exception.class)
    ProblemDetail unexpected(Exception e) {
        log.error("Unhandled error", e);
        var pd = ProblemDetail.forStatusAndDetail(INTERNAL_SERVER_ERROR, "Internal error");
        pd.setProperty("traceId", MDC.get("traceId"));
        return pd;
    }
}
```

### Anti-patterns
```java
// ❌ Returns full stack to client
return ResponseEntity.status(500).body(e.getMessage() + "\n" + Arrays.toString(e.getStackTrace()));

// ❌ Swallows root cause
catch (Exception e) { return ResponseEntity.badRequest().build(); }
```

---

## 4. OpenAPI / Swagger

### Checklist
- [ ] springdoc-openapi (or springfox legacy) generates spec at build time; spec checked into
  `<ctx>-api/openapi.yaml` and is the contract source of truth
- [ ] Every endpoint has `@Operation(summary, description)`; every DTO field has `@Schema(description, example)`
- [ ] Security schemes (`bearerAuth`, `mTLS`) declared; every endpoint links to required scheme + scopes
- [ ] Error responses documented (`@ApiResponse(responseCode = "400" / "401" / "403" / "404" / "409")`)
- [ ] Swagger UI **disabled** in `prod` profile (`springdoc.swagger-ui.enabled=false`)
- [ ] Breaking changes increment major version path (`/api/v2/...`); deprecated routes flagged via `@Deprecated`
  and `Deprecation` / `Sunset` headers

---

## 5. Pagination, Sorting, Filtering

### Checklist
- [ ] List endpoints **always paginated** — never return unbounded collections
- [ ] Pagination via `Pageable` (Spring Data) or explicit `page` + `size` query params with `@Max(100)` cap
- [ ] Response includes `totalElements`, `totalPages`, `number`, `size`, `content`
- [ ] Sort fields validated against an allowlist — never pass raw `Sort` from request to repository
- [ ] Filter parameters typed (records, `@RequestParam` with binding) — not free-form `Map<String,String>`

```java
// ✅ CORRECT — sort allowlist
private static final Set<String> SORTABLE = Set.of("createdAt", "amount", "status");

@GetMapping
PageView<TxnView> list(
    @RequestParam(defaultValue = "0") @Min(0) int page,
    @RequestParam(defaultValue = "20") @Min(1) @Max(100) int size,
    @RequestParam(defaultValue = "createdAt") String sort) {
    if (!SORTABLE.contains(sort)) throw new BusinessException("INVALID_SORT");
    return service.list(PageRequest.of(page, size, Sort.by(sort).descending()));
}
```

---

## 6. Idempotency and Concurrency

### Checklist
- [ ] All POST endpoints that mutate money / create entities accept an `Idempotency-Key` header
- [ ] Server stores `(idempotencyKey, requestHash) → response` for ≥ 24h (Redis or DB)
- [ ] Replay with same key + same body → return stored response with same status
- [ ] Replay with same key + **different** body → 422 `IDEMPOTENCY_CONFLICT`
- [ ] PUT endpoints support optimistic locking via `If-Match` ETag → maps to JPA `@Version`
- [ ] 409 returned on `OptimisticLockException` with guidance to re-fetch

---

## 7. Versioning and Deprecation

### Checklist
- [ ] URI versioning (`/api/v1/`, `/api/v2/`) — chosen consistently for the platform
- [ ] No accept-header juggling unless explicitly chosen project-wide
- [ ] Deprecation lifecycle: announce → `Deprecation` + `Sunset` headers → removal in next major release
- [ ] Old version maintained for the contractual deprecation window (typically 6 months)
- [ ] Contract tests (Spring Cloud Contract / Pact) exist for every consumer

---

## 8. Anti-patterns Catalogue

| Anti-pattern                                              | Correct Alternative                                  |
| --------------------------------------------------------- | ---------------------------------------------------- |
| Returning JPA entity from controller                      | DTO record + MapStruct                               |
| `Map<String, Object>` request body                        | Typed record DTO with Bean Validation                |
| Business logic inside `@RestController`                   | Delegate to `@Service`                               |
| `@Autowired` field injection                              | Constructor injection (`@RequiredArgsConstructor`)   |
| `double` / `float` for money                              | `BigDecimal` with `@Digits`                          |
| `Date` / `Calendar`                                       | `Instant` / `OffsetDateTime` / `LocalDate`           |
| Unbounded list endpoint                                   | `Pageable` + `@Max(100)` size                        |
| Free-form sort param                                      | Allowlist of sortable fields                         |
| Returning stack trace to client                           | RFC 7807 `ProblemDetail` + traceId                   |
| Different DTO shape per environment                       | One DTO; environment-only fields stripped server-side |
| Swagger UI exposed in `prod`                              | `springdoc.swagger-ui.enabled=false` for `prod`      |


-------------------------------------------------------------

