# Reference: Testing — eclipse-regional-platform

## Table of Contents
1. Test Pyramid and Conventions
2. Unit Tests (JUnit 5 + Mockito + AssertJ)
3. Slice Tests (`@WebMvcTest`, `@DataJpaTest`, etc.)
4. Integration Tests (`@SpringBootTest` + Testcontainers)
5. Contract Tests (Spring Cloud Contract / Pact)
6. External Service Stubs (WireMock)
7. Security and Authorization Tests
8. Performance / Load Tests
9. Anti-patterns Catalogue

---

## 1. Test Pyramid and Conventions

### Checklist
- [ ] Pyramid: ~70% unit, ~20% integration / slice, ~10% E2E
- [ ] Test class naming: `*Test.java` (unit, run by Surefire), `*IT.java` (integration, run by Failsafe)
- [ ] Test methods named with `@DisplayName` describing behaviour: `"rejects transfer when balance insufficient"`
- [ ] AssertJ used for assertions — fluent, descriptive failure messages
- [ ] No `@SpringBootTest` when a `@WebMvcTest` / `@DataJpaTest` slice will do (startup cost matters)
- [ ] No production-only conditions inside tests (`if (env == prod)`); use Spring profiles + `@ActiveProfiles("test")`
- [ ] `@DirtiesContext` justified — it forces a full context reload and slows the suite
- [ ] Tests deterministic — no time / random / network dependencies without injection
- [ ] Coverage targets per layer documented in the main `SKILL.md` (Step 3.6)

---

## 2. Unit Tests

### Checklist
- [ ] Plain JUnit 5 (`@Test`) — no Spring context
- [ ] Mockito (`@Mock`, `@InjectMocks`) for collaborators; `@ExtendWith(MockitoExtension.class)`
- [ ] Each test follows **Arrange-Act-Assert (Given-When-Then)** structure
- [ ] One logical assertion per test (multiple AssertJ chained calls are fine)
- [ ] Parameterised tests (`@ParameterizedTest` + `@CsvSource` / `@MethodSource`) for boundary tables
- [ ] `Clock` injected via constructor and provided as `Clock.fixed(...)` in tests
- [ ] No `Thread.sleep` — use `Awaitility` for waits
- [ ] No production logging suppressed without justification

```java
// ✅ CORRECT — unit test
@ExtendWith(MockitoExtension.class)
class TransferServiceTest {

    @Mock AccountRepository accounts;
    @Mock LedgerPort ledger;
    @InjectMocks TransferService service;

    @Test
    @DisplayName("rejects transfer when balance insufficient")
    void rejectsWhenInsufficient() {
        var from = AccountFixture.with(Money.of("10.00", "MYR"));
        var to   = AccountFixture.empty();
        when(accounts.loadForUpdate(from.id())).thenReturn(Optional.of(from));
        when(accounts.loadForUpdate(to.id())).thenReturn(Optional.of(to));

        assertThatThrownBy(() -> service.execute(
                new TransferCommand(from.id(), to.id(), Money.of("50.00", "MYR")), "key-1"))
            .isInstanceOf(BusinessException.class)
            .hasMessage("INSUFFICIENT_FUNDS");

        verifyNoInteractions(ledger);
    }
}
```

---

## 3. Slice Tests

### `@WebMvcTest` (Controller layer)
- [ ] Controller tested in isolation with `MockMvc` + mocked service
- [ ] Validation errors covered (`status().isBadRequest()` + body fields)
- [ ] Security tested via `@WithMockUser` / `@WithMockJwt` / `SecurityMockMvcRequestPostProcessors`
- [ ] Error advice covered — 4xx / 5xx mappings asserted

### `@DataJpaTest` (Repository layer)
- [ ] Runs against **Testcontainers** PostgreSQL / Oracle — not H2 (engine-specific SQL differs)
- [ ] `@AutoConfigureTestDatabase(replace = NONE)` to keep the real engine
- [ ] Queries with non-trivial JPQL / native SQL asserted on a seeded dataset
- [ ] N+1 detected via `Statistics` from `SessionFactory`

```java
// ✅ CORRECT — slice test for controller
@WebMvcTest(AccountController.class)
class AccountControllerTest {
    @Autowired MockMvc mvc;
    @MockBean AccountService service;

    @Test
    @WithMockUser(authorities = "SCOPE_account.read")
    void returns200WithBody() throws Exception {
        when(service.get(any())).thenReturn(new AccountView(...));
        mvc.perform(get("/api/v1/accounts/{id}", UUID.randomUUID()))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.id").exists());
    }
}
```

---

## 4. Integration Tests (`@SpringBootTest` + Testcontainers)

### Checklist
- [ ] Full-context test reserved for cross-layer flows (controller → service → DB → message bus)
- [ ] Testcontainers spin up real PostgreSQL / Oracle, Kafka, Redis, Vault — never embedded H2 for prod-engine code
- [ ] Containers reused across tests via `@Container(static = true)` or Singleton pattern to keep suite fast
- [ ] `@DynamicPropertySource` wires container hostnames / ports into Spring properties
- [ ] Liquibase / Flyway migrations applied on container startup — tests verify migrations are forward-compatible
- [ ] Outbox / Kafka publication asserted by consuming from a test consumer
- [ ] Tests run on every PR via CI; tagged `@Tag("integration")` and Failsafe-bound

```java
// ✅ CORRECT — Testcontainers + Spring Boot Test
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class TransferFlowIT {

    @Container static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    @Container static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
}
```

---

## 5. Contract Tests

### Checklist
- [ ] **Producer side**: Spring Cloud Contract (Groovy / YAML) or Pact contracts generated; verified in build
- [ ] **Consumer side**: stubs published to a stub runner / Pact Broker; consumer tests run against published stub
- [ ] Contract changes follow consumer-driven flow — producer cannot break a registered consumer
- [ ] Contracts versioned alongside the API; breaking change → new major version
- [ ] Contracts cover happy path **and** error envelopes (4xx, 5xx, validation errors)

---

## 6. External Service Stubs (WireMock)

### Checklist
- [ ] WireMock used for every external HTTP dependency in integration tests — never call real partner
- [ ] Stubs cover: 2xx, 4xx, 5xx, slow response (timeout), connection reset
- [ ] Resilience4j circuit-breaker / retry behaviour tested via WireMock scenarios
- [ ] Stub mappings stored as JSON files alongside tests for reuse
- [ ] WireMock recorder used to capture real partner traffic once → replayed offline (never recorded in prod)

---

## 7. Security and Authorization Tests

### Checklist
- [ ] Every privileged endpoint has a negative test: unauthenticated → 401, wrong role → 403
- [ ] BOLA / object-level tests: actor A cannot read or mutate actor B's data
- [ ] Tenant isolation tests: tenant X cannot access tenant Y's data even with matching path id
- [ ] JWT validation tests: expired token → 401, tampered signature → 401, wrong audience → 401, wrong issuer → 401
- [ ] CORS rejection tests for unauthorised origin
- [ ] Idempotency-key conflict tests (same key + different body → 422)
- [ ] CSRF protection tests for any cookie-auth surface
- [ ] Rate-limit tests (`429 Too Many Requests`)

---

## 8. Performance / Load Tests

### Checklist
- [ ] Load tests written in **Gatling** or **k6** — checked into `perf/` directory
- [ ] Baseline scenarios cover top-5 critical flows (login, balance enquiry, transfer, statement, FX)
- [ ] SLOs defined (p95 latency, error rate, throughput); test asserts against SLO
- [ ] Run nightly in `perf` environment; results published to Grafana / report
- [ ] No load test ever runs against `prod`

---

## 9. Anti-patterns Catalogue

| Anti-pattern                                            | Correct Alternative                                     |
| ------------------------------------------------------- | ------------------------------------------------------- |
| `@SpringBootTest` everywhere                            | Slice tests (`@WebMvcTest`, `@DataJpaTest`)             |
| H2 to stand in for PostgreSQL / Oracle                  | Testcontainers with the real engine + version           |
| `Thread.sleep(...)` to wait for async                   | `Awaitility.await().atMost(...).until(...)`             |
| Mocking the class under test                            | Test the real instance; mock only its collaborators     |
| `verify(repo, times(1))` everywhere                     | Verify only behaviour that matters; otherwise omit      |
| `@DirtiesContext` per test                              | Properly scope mocks / reset state                      |
| Hardcoded date/time in assertions                       | Inject `Clock`; use `Clock.fixed(...)`                  |
| Calling real partner API in test                        | WireMock stub                                           |
| No negative auth tests on a new endpoint                | Add unauth (401) + forbidden (403) cases                |
| Tests dependent on execution order                      | Make each test self-contained                           |


-----------------------------------------------------------------------

