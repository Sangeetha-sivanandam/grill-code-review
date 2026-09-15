# Reference: Messaging & Integration — eclipse-regional-platform

## Table of Contents
1. Kafka Producers
2. Kafka Consumers
3. Transactional Outbox Pattern
4. RabbitMQ (where used)
5. Feign / WebClient / RestTemplate Clients
6. Resilience4j — Circuit Breaker, Retry, Bulkhead, Rate Limiter, Time Limiter
7. SSRF, Timeouts, and Connection Limits
8. Anti-patterns Catalogue

---

## 1. Kafka Producers

### Checklist
- [ ] Topic name follows convention: `<domain>.<entity>.<event>.v<major>` e.g. `transfer.transaction.completed.v1`
- [ ] Key chosen to preserve ordering per aggregate (e.g. `accountId` for account events) — never `null` on
  topics requiring order
- [ ] Schema enforced via Confluent Schema Registry (Avro / Protobuf) or JSON Schema — never raw maps
- [ ] Producer `acks=all`, `enable.idempotence=true`, `retries=Integer.MAX_VALUE`, `max.in.flight.requests.per.connection=5`
- [ ] `compression.type=lz4` or `zstd`
- [ ] Transactional producer (`transactional.id` set) when writing to multiple partitions atomically
- [ ] **Never publish in `@Transactional` boundary** — use the outbox pattern (Section 3)
- [ ] Headers carry `traceparent`, `tenantId`, `eventId`, `eventVersion`, `producedAt`
- [ ] Payload contains **no PII / CHD** unless topic is in the compliant data class and encrypted

```java
// ✅ CORRECT — idempotent producer config (yml)
spring.kafka.producer:
  acks: all
  enable-idempotence: true
  retries: 2147483647
  compression-type: lz4
  properties:
    max.in.flight.requests.per.connection: 5
    delivery.timeout.ms: 120000
```

---

## 2. Kafka Consumers

### Checklist
- [ ] Consumer group id stable and unique per service (`<service>.<topic>.consumer`)
- [ ] `enable.auto.commit=false`; commits are **manual** after successful processing
- [ ] `isolation.level=read_committed` when working with transactional producers
- [ ] Handler is **idempotent** — deduplicates by `eventId` (Redis / DB) before applying
- [ ] Poison-pill handling: `DefaultErrorHandler` configured with `DeadLetterPublishingRecoverer` →
  routes to `<topic>.DLT`
- [ ] Retry policy: exponential backoff bounded (`ExponentialBackOffWithMaxRetries`); after exhaustion → DLT
- [ ] Concurrency tuned per partition count; never more consumers than partitions
- [ ] Long-running work moved off the poll thread; commit only after work completes
- [ ] `max.poll.records` + `max.poll.interval.ms` tuned so the consumer never gets evicted from the group
- [ ] MDC populated from incoming headers (`traceparent`, `tenantId`)
- [ ] Metrics: lag (`kafka.consumer.records.lag`), processing time, retry count, DLT count
- [ ] Replay procedure documented (`kafka-consumer-groups.sh --reset-offsets`)

```java
// ✅ CORRECT — manual ack + dedup + DLT
@KafkaListener(topics = "${app.kafka.topic.transfer-completed}",
               groupId = "${app.kafka.group}",
               concurrency = "3")
void onTransferCompleted(ConsumerRecord<String, TransferCompleted> rec, Acknowledgment ack) {
    String eventId = headerString(rec, "eventId");
    if (deduper.alreadyProcessed(eventId)) { ack.acknowledge(); return; }
    handler.handle(rec.value());
    deduper.markProcessed(eventId);
    ack.acknowledge();
}
```

---

## 3. Transactional Outbox Pattern

### Checklist
- [ ] Service writes domain change **and** an `outbox_event` row in the **same** DB transaction
- [ ] Outbox row contains: `eventId` (UUID), `aggregateType`, `aggregateId`, `eventType`, `eventVersion`,
  `payload` (JSON / Avro bytes), `headers`, `createdAt`, `processedAt`
- [ ] Separate poller (`@Scheduled` with `@SchedulerLock`) or Debezium CDC publishes to Kafka, then marks the row processed
- [ ] Publisher tolerates retries (Kafka idempotent producer + `eventId` for consumer dedup)
- [ ] Outbox table indexed on `(processedAt, createdAt)` for poll efficiency
- [ ] Old processed rows pruned by a separate job per retention policy
- [ ] **Never** publish directly to Kafka inside `@Transactional` — TX may roll back after publish, causing a phantom event

---

## 4. RabbitMQ (where used)

### Checklist
- [ ] Exchanges, queues, bindings declared in code (`Declarables` bean) or via Terraform — never via ad-hoc UI clicks
- [ ] Queues durable, messages persistent (`MessageDeliveryMode.PERSISTENT`)
- [ ] Publisher confirms enabled (`spring.rabbitmq.publisher-confirm-type: correlated`) and consumer acks manual
- [ ] DLX (dead-letter exchange) bound for every queue; retry queue with TTL for backoff
- [ ] Per-queue prefetch (`spring.rabbitmq.listener.simple.prefetch`) tuned to avoid memory blow-up
- [ ] Connection / channel pool sized per consumer concurrency

---

## 5. Feign / WebClient / RestTemplate Clients

### Checklist
- [ ] **OpenFeign** preferred for synchronous REST to internal services; **WebClient** for reactive paths
- [ ] One client interface per downstream service; lives in `infrastructure.client`
- [ ] Connection + read timeouts **always set** — never default (`Integer.MAX_VALUE` or infinite)
  - Connect: ≤ 3s
  - Read: ≤ 10s (or downstream SLO)
- [ ] HTTP connection pool configured (Apache HC5 / OkHttp via Feign / Reactor Netty for WebClient)
- [ ] Retry, circuit breaker, bulkhead applied via **Resilience4j** (Section 6) — not Feign's built-in retry
- [ ] `traceparent` propagated automatically (Micrometer Tracing instrumentation present)
- [ ] mTLS used for partner / payment-network calls (see [01-security.md](01-security.md) Section 4)
- [ ] Request / response logging gated by feature flag — never logs PII / token / PAN
- [ ] Response size bounded (`Content-Length` check) to mitigate API10 (Unsafe Consumption)

```java
// ✅ CORRECT — Feign client with timeouts + Resilience4j
@FeignClient(name = "ledger", url = "${ledger.base-url}", configuration = LedgerFeignConfig.class)
interface LedgerClient {
    @CircuitBreaker(name = "ledger")
    @Retry(name = "ledger")
    @Bulkhead(name = "ledger")
    @TimeLimiter(name = "ledger")
    @PostMapping("/postings")
    PostingResponse post(@RequestBody PostingRequest req);
}
```

---

## 6. Resilience4j

### Checklist
- [ ] Circuit breaker configured per downstream (failure rate threshold, slow-call threshold, sliding window)
- [ ] Fallback method always defined; returns a typed degraded response — never `null` or rethrow
- [ ] Retry: bounded attempts (≤ 3), exponential backoff with jitter, **only** on transient errors
  (5xx / IOException) — never on 4xx
- [ ] Bulkhead isolates downstream slowdowns from the rest of the service (semaphore or thread-pool)
- [ ] Time limiter wraps async calls; aligned with downstream SLO and HTTP read timeout
- [ ] Rate limiter on inbound API surface (per-tenant) for endpoints sensitive to abuse
- [ ] All Resilience4j components register Micrometer metrics — alerting on open circuit / high reject rate
- [ ] Config in `application.yml` per environment — sensible defaults in base, tightened in prod

```yaml
# ✅ CORRECT — Resilience4j config sample
resilience4j:
  circuitbreaker.instances.ledger:
    sliding-window-type: COUNT_BASED
    sliding-window-size: 50
    failure-rate-threshold: 50
    slow-call-rate-threshold: 50
    slow-call-duration-threshold: 2s
    permitted-number-of-calls-in-half-open-state: 5
    wait-duration-in-open-state: 30s
  retry.instances.ledger:
    max-attempts: 3
    wait-duration: 200ms
    exponential-backoff-multiplier: 2
    retry-exceptions: [java.io.IOException, feign.RetryableException]
  timelimiter.instances.ledger:
    timeout-duration: 5s
```

---

## 7. SSRF, Timeouts, and Connection Limits

### Checklist
- [ ] Outbound destinations come from typed `@ConfigurationProperties` — never from request input
- [ ] If a use case must accept a URL from input (webhook, callback), the URL is validated against an **allowlist**
  of hosts; private / link-local / metadata IPs (`169.254.169.254`, `127.0.0.0/8`, `10.0.0.0/8`, IPv6 equivalents) rejected
- [ ] DNS resolution + connection limit per host enforced via the HTTP client (Apache HC5 `connPerRoute`)
- [ ] Max response size enforced (e.g. WebClient `ExchangeFilterFunction` + `DataBufferLimitException`)
- [ ] Decompression bombs guarded against (`Content-Encoding: gzip` with bounded decompressed size)

---

## 8. Anti-patterns Catalogue

| Anti-pattern                                              | Correct Alternative                                          |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| `kafkaTemplate.send(...)` inside `@Transactional`         | Transactional outbox + separate publisher                    |
| Consumer auto-commits before processing                   | `enable.auto.commit=false` + manual `Acknowledgment.acknowledge()` |
| Retry policy with unbounded attempts                      | Bounded attempts + exponential backoff + DLT                 |
| Retry on 4xx errors                                       | Retry only on 5xx / `IOException` / explicit transient list  |
| No DLT / DLQ wiring                                       | `DeadLetterPublishingRecoverer` → `<topic>.DLT`              |
| Default Feign timeouts                                    | Explicit connect + read timeouts                             |
| Direct call to external host from request input           | Allowlist + private-IP rejection                             |
| `RestTemplate` with no `ResponseErrorHandler`             | Custom handler that throws typed exceptions                  |
| Circuit breaker with no fallback                          | Typed degraded fallback method                               |
| Kafka payload carrying PAN / CVV                          | Tokenise; carry only token + masked PAN                      |
| Consumer concurrency > partition count                    | Concurrency ≤ partition count                                |


-------------------------------------------------------------

