# Reference: Observability — eclipse-regional-platform

## Table of Contents
1. Structured Logging (SLF4J + Logback JSON)
2. MDC and Correlation IDs
3. Metrics (Micrometer + Prometheus)
4. Distributed Tracing (OpenTelemetry / Micrometer Tracing)
5. Health Checks and Actuator
6. Audit Logging
7. PII / Secret Redaction
8. Anti-patterns Catalogue

---

## 1. Structured Logging

### Checklist
- [ ] SLF4J is the only logging API used (`org.slf4j.Logger`) — never `java.util.logging`, `System.out`, `printStackTrace()`
- [ ] Logback configuration (`logback-spring.xml`) ships JSON to stdout in all non-local environments
  (via `logstash-logback-encoder` or `LogstashEncoder`)
- [ ] Log levels per environment:
  - `local` / `dev` — DEBUG on application packages, INFO on framework
  - `sit` / `uat` — INFO on application, WARN on framework
  - `stg` / `prod` — INFO on application, WARN on framework (DEBUG only via runtime change via Actuator + audit log)
- [ ] Parameterised messages (`log.info("created account {}", id)`) — never `String.format` or concatenation
- [ ] Exception logging: `log.error("op failed for {}", id, ex)` (exception is the **last** argument, not in the format string)
- [ ] No log statement inside a tight loop without a rate limiter / sampler
- [ ] Logger declared `private static final` per class — no static loggers via Lombok `@Slf4j` when MDC behaviour differs per instance

```java
// ✅ CORRECT
private static final Logger log = LoggerFactory.getLogger(TransferService.class);

log.info("transfer initiated id={} amount={} {}", transfer.id(), money.amount(), money.currency());
log.error("transfer failed id={}", transfer.id(), ex);
```

---

## 2. MDC and Correlation IDs

### Checklist
- [ ] A servlet / reactive filter populates MDC at request entry with:
  `traceId`, `spanId`, `requestId`, `tenantId`, `customerHash` (hashed customer id), `clientIp` (truncated /24 for IPv4)
- [ ] MDC cleared in a `finally` block — never leaks across requests / thread reuse
- [ ] `TaskDecorator` registered on every executor so MDC propagates across `@Async` and `CompletableFuture` boundaries
- [ ] Logback pattern includes `%X{traceId}` and `%X{tenantId}` in every log line
- [ ] Outbound HTTP / Kafka headers carry `traceparent` (W3C) — auto-handled by Micrometer Tracing / OpenTelemetry
- [ ] Never put raw PII (email, phone, full name) in MDC

---

## 3. Metrics (Micrometer + Prometheus)

### Checklist
- [ ] Micrometer registry exposed at `/actuator/prometheus` — only on the management port, secured with `ROLE_ACTUATOR`
- [ ] Counters / timers / gauges named per Micrometer convention: `service.operation.outcome` (dot-separated)
- [ ] **Tags / labels carry low-cardinality values only** — `outcome`, `status_code`, `endpoint`, `tenantId` (if bounded)
  — never raw `customerId`, request URI with path variables, or error message text
- [ ] HTTP server metrics enabled (`management.metrics.web.server.request.autotime.enabled=true`)
- [ ] Custom business metrics added for every critical flow (transfer count, FX trade volume, fraud-flag rate)
- [ ] SLO metrics aligned with the service's SLI definitions; alert rules in Prometheus / Grafana
- [ ] Histograms (`publishPercentileHistogram(true)`) on latency-critical timers — not just percentiles
- [ ] Common tags applied via `MeterRegistryCustomizer` (`region`, `env`, `service`, `version`)

```java
// ✅ CORRECT — bounded tags
Timer.builder("transfer.execute")
     .description("Transfer execution latency")
     .tag("outcome", outcome)        // SUCCESS / FAILURE / TIMEOUT
     .tag("currency", currency)      // ~150 ISO codes — bounded
     .publishPercentileHistogram()
     .register(registry)
     .record(elapsed);
```

### Anti-patterns
```java
// ❌ Unbounded tag — explodes Prometheus cardinality
registry.counter("api.calls", "userId", userId).increment();

// ❌ Tag = raw error message
registry.counter("errors", "message", e.getMessage()).increment();
```

---

## 4. Distributed Tracing

### Checklist
- [ ] Micrometer Tracing (Brave or OpenTelemetry bridge) auto-instruments Spring MVC, WebClient, RestTemplate, Feign, JDBC, Kafka
- [ ] W3C `traceparent` propagated outbound to all downstream services and partners
- [ ] Manual spans added for business-significant operations via `Observation` API
- [ ] Span attributes carry **no PII** — `customer.id.hash`, `account.id` (UUID), `amount` (yes — needed for tracing) but **never** PAN / CVV
- [ ] Sampling rate environment-appropriate: 100% in `sit`/`uat`, 10–25% in `prod` (head-based) or tail-based on errors
- [ ] Errors annotated on span (`span.error(throwable)`)

---

## 5. Health Checks and Actuator

### Checklist
- [ ] Management endpoints on a **separate port** (`management.server.port=8081`) so they're never exposed via the public LB
- [ ] `management.endpoints.web.exposure.include` is an **explicit allowlist** — never `*` in prod
  Allowed in prod: `health`, `info`, `prometheus`, `metrics`
- [ ] Spring Boot 3 group `health.liveness.include` / `health.readiness.include` configured for K8s probes
- [ ] Custom `HealthIndicator` per critical downstream (DB, Kafka, partner API) — but does **not** call deep partner APIs in liveness
- [ ] `management.endpoint.env.show-values=NEVER`; `management.endpoint.configprops.show-values=NEVER`
- [ ] Actuator chain protected by Spring Security → distinct `SecurityFilterChain` requiring `ROLE_ACTUATOR`
- [ ] `info` endpoint contains build / git info via `spring-boot-starter-actuator` + `git-commit-id-plugin` — **no secrets**

```yaml
# ✅ CORRECT — prod actuator config
management:
  server.port: 8081
  endpoints.web.exposure.include: health,info,prometheus
  endpoint:
    health.probes.enabled: true
    env.show-values: NEVER
    configprops.show-values: NEVER
  health:
    livenessstate.enabled: true
    readinessstate.enabled: true
```

---

## 6. Audit Logging

### Checklist
- [ ] Dedicated audit appender (`AUDIT`) writes to a separate file / Kafka topic — distinct from application log
- [ ] Audit events emitted for: login success/failure, password change, consent grant/revoke, role change,
  funds movement, admin action, configuration change, data export
- [ ] Audit record contains: `eventId` (UUID), `eventType`, `timestamp` (UTC), `actor` (hashed), `target`,
  `outcome`, `traceId`, `tenantId`, `region`, `ipAddress` (truncated), `userAgent`
- [ ] Audit appender is **append-only**; tamper-evident via sequence number + HMAC chain or write to immutable store
- [ ] Audit log retention per regulatory requirement (BNM RMiT typically ≥ 7 years)
- [ ] Audit log never contains PAN, CVV, password, token

---

## 7. PII / Secret Redaction

### Checklist
- [ ] Logback `LayoutWrappingEncoder` / pattern converter masks known patterns:
  - 13–19 digit sequences → `****-****-****-1234`
  - Email → `j***@example.com`
  - JWT-shaped strings → `[REDACTED_TOKEN]`
- [ ] Jackson `@JsonIgnore` / `@ToString.Exclude` on sensitive DTO / entity fields
- [ ] Custom `toString()` on domain objects holding sensitive data → never includes the secret
- [ ] Sentry / Crashlytics-equivalent error reporters configured with PII scrubber
- [ ] No `log.debug("request body: {}", json)` — even at DEBUG; log a redacted summary instead

---

## 8. Anti-patterns Catalogue

| Anti-pattern                                              | Correct Alternative                                          |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| `System.out.println` / `e.printStackTrace()`              | SLF4J `log.error("...", ex)`                                 |
| String concatenation in log message                       | Parameterised `{}` placeholders                              |
| Logging full request / response body                      | Redacted summary; never PII                                  |
| `customerId` / email as Micrometer tag                    | Hashed id; or omit                                           |
| Tag = full error message                                  | Tag = `outcome` (SUCCESS / FAILURE / TIMEOUT)                |
| Actuator on public port                                   | Separate `management.server.port`                            |
| `management.endpoints.web.exposure.include: "*"` in prod  | Allowlist: `health,info,prometheus`                          |
| Audit log mixed with application log                      | Dedicated `AUDIT` appender, append-only sink                 |
| 100% trace sampling in prod                               | 10–25% head sampling or tail-based on errors                 |
| `@Slf4j` Lombok mixed with manual MDC handling            | Explicit logger + central MDC filter                         |


--------------------------------------------------------------

