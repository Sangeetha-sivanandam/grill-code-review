# Reference: Security Layer — eclipse-regional-platform

## Table of Contents
1. Spring Security Configuration (HttpSecurity / SecurityFilterChain)
2. Authentication — OAuth2 Resource Server / JWT
3. Authorization — Method Security (`@PreAuthorize`) and Object-Level Checks
4. mTLS for Service-to-Service and Partner APIs
5. Cryptography — Hashing, Encryption, Random
6. Secrets Management
7. Input Validation, Deserialization, and Injection Prevention
8. CORS, CSRF, Headers, and Session Management
9. Secure Logging Policy
10. Anti-patterns Catalogue

---

## 1. Spring Security Configuration

### Checklist
- [ ] One `SecurityFilterChain` bean per surface (public API, internal API, Actuator) — distinct rules
- [ ] `authorizeHttpRequests()` is **deny-by-default**: ends with `.anyRequest().authenticated()` (or `.denyAll()`)
- [ ] No `permitAll()` on state-changing endpoints — only on `/health`, `/info`, `/openapi.yaml`, login pages
- [ ] CSRF disabled **only** for stateless API surfaces using bearer tokens; left enabled for cookie-auth flows
- [ ] Session creation policy `STATELESS` for token-authenticated REST APIs
- [ ] HSTS, X-Content-Type-Options, X-Frame-Options, Referrer-Policy headers enabled via `headers()` DSL
- [ ] Actuator chain isolated: requires `ROLE_ACTUATOR` and management port distinct from application port

```java
// ✅ CORRECT — stateless REST API chain
@Bean
@Order(1)
SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .csrf(CsrfConfigurer::disable) // stateless bearer-token API
        .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
        .authorizeHttpRequests(a -> a
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(jwtAuthConverter())))
        .headers(h -> h
            .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true).maxAgeInSeconds(31_536_000))
            .contentTypeOptions(Customizer.withDefaults())
            .frameOptions(FrameOptionsConfig::deny));
    return http.build();
}
```

### Anti-patterns
```java
// ❌ Wildcard + CSRF disabled on a cookie-authenticated surface
http.csrf().disable().authorizeHttpRequests().anyRequest().permitAll();

// ❌ Stateful session for a REST API → CSRF / fixation risk
http.sessionManagement(s -> s.sessionCreationPolicy(IF_REQUIRED));
```

---

## 2. Authentication — OAuth2 Resource Server / JWT

### Checklist
- [ ] JWT validation uses a `JwtDecoder` configured with **JWKS URL** (auto-rotating) — not a hardcoded HMAC secret
- [ ] `issuer-uri` matches the trusted IdP exactly; `aud` (audience) verified
- [ ] Algorithm allowlist — only `RS256` / `ES256`; `none` and HS-family rejected for inbound tokens from external IdPs
- [ ] `OAuth2TokenValidator` chain checks: timestamp, issuer, audience, required scopes/claims, clock skew ≤ 60s
- [ ] Refresh tokens stored server-side (Redis) with rotation; revocation list honoured
- [ ] No JWT logged in full — log only the `jti` and `sub` (hashed)
- [ ] Login throttling / lockout enforced (Bucket4j or Resilience4j rate limiter on auth endpoints)

```java
// ✅ CORRECT — JWKS-backed decoder with audience + timestamp validation
@Bean
JwtDecoder jwtDecoder(@Value("${security.oauth2.issuer-uri}") String issuer,
                      @Value("${security.oauth2.audience}") String audience) {
    NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuer);
    OAuth2TokenValidator<Jwt> validators = new DelegatingOAuth2TokenValidator<>(
        new JwtTimestampValidator(Duration.ofSeconds(60)),
        new JwtIssuerValidator(issuer),
        new JwtClaimValidator<List<String>>("aud", aud -> aud != null && aud.contains(audience))
    );
    decoder.setJwtValidator(validators);
    return decoder;
}
```

### Anti-patterns
```java
// ❌ HMAC secret hardcoded
NimbusJwtDecoder.withSecretKey(new SecretKeySpec("supersecret".getBytes(), "HmacSHA256")).build();

// ❌ Algorithm "none" silently accepted
JWT.require(Algorithm.none()).build().verify(token);

// ❌ No audience or issuer validation
NimbusJwtDecoder.withJwkSetUri(jwks).build(); // uses default validator only
```

---

## 3. Authorization — Method Security & Object-Level Checks

### Checklist
- [ ] `@EnableMethodSecurity(prePostEnabled = true)` on a security config class
- [ ] Every privileged service / controller method carries `@PreAuthorize(...)` — never relies on URL-only rules
- [ ] **Object-level (BOLA / API1)**: every `findById`, update, delete checks `entity.ownerId == currentUser.id`
  or tenant via a guard / `@PostAuthorize("returnObject.tenantId == authentication.tenantId")`
- [ ] Role names use `ROLE_` prefix consistently; authorities mapped from JWT claims via `JwtAuthenticationConverter`
- [ ] No authorization decisions made in the controller body — delegate to method security or a `PermissionEvaluator`
- [ ] No `@Secured`/`@RolesAllowed` mixed with `@PreAuthorize` in the same module — pick one

```java
// ✅ CORRECT — method security + tenant guard
@PreAuthorize("hasAuthority('SCOPE_account.read') and @tenantGuard.canAccess(#accountId)")
public AccountView getAccount(UUID accountId) { ... }
```

### Anti-patterns
```java
// ❌ Trusting client-supplied tenantId
@GetMapping("/accounts/{id}")
AccountView get(@PathVariable UUID id, @RequestParam UUID tenantId) { ... }

// ❌ Authorization in the controller body
if (!user.getRoles().contains("ADMIN")) throw new AccessDeniedException("...");
```

---

## 4. mTLS for Service-to-Service and Partner APIs

### Checklist
- [ ] Inbound mTLS enforced via Tomcat / Netty SSL config requiring client auth on partner-facing endpoints
- [ ] Outbound mTLS configured per `RestTemplate` / `WebClient` / Feign client via a dedicated `SslBundle`
- [ ] Client certs and keystore passwords sourced from Vault / KMS — never from `application.yml`
- [ ] Truststore contains only the explicit CA chain for the partner — never a system truststore for partner calls
- [ ] Hostname verification **always on** — never `NoopHostnameVerifier`
- [ ] Cert rotation runbook exists; expiry alerted via Micrometer gauge

```java
// ✅ CORRECT — Spring Boot SslBundle (Boot 3.1+)
@Bean
WebClient partnerWebClient(SslBundles bundles) {
    SslBundle bundle = bundles.getBundle("partner-mtls");
    HttpClient http = HttpClient.create().secure(s -> s.sslContext(bundle.createSslContext()));
    return WebClient.builder().clientConnector(new ReactorClientHttpConnector(http)).build();
}
```

### Anti-patterns
```java
// ❌ Trust-all SSL context — disables the entire purpose of TLS
SSLContext ctx = SSLContext.getInstance("TLS");
ctx.init(null, new TrustManager[]{ new X509TrustManager() {
    public void checkClientTrusted(X509Certificate[] c, String t) {}
    public void checkServerTrusted(X509Certificate[] c, String t) {}
    public X509Certificate[] getAcceptedIssuers() { return new X509Certificate[0]; }
}}, new SecureRandom());
```

---

## 5. Cryptography — Hashing, Encryption, Random

### Checklist
- [ ] Passwords hashed with `BCryptPasswordEncoder` (cost ≥ 12) or `Argon2PasswordEncoder` — never MD5 / SHA-1 / SHA-256
- [ ] Symmetric encryption uses **AES-256-GCM** (authenticated) — never ECB / CBC without HMAC
- [ ] IV / nonce generated per message via `SecureRandom`; never reused
- [ ] Asymmetric crypto: RSA ≥ 2048 (prefer 3072) or ECDSA P-256 / P-384
- [ ] Key derivation uses PBKDF2 (≥ 600k iterations), Argon2, or HKDF — never plain hashing
- [ ] Random IDs / tokens generated via `SecureRandom` or `UUID.randomUUID()` — never `Math.random()` or `Random`
- [ ] Master keys stored in HSM / Cloud KMS; data keys wrapped (envelope encryption)

```java
// ✅ CORRECT — AES-256-GCM with random IV
byte[] iv = new byte[12];
SecureRandom.getInstanceStrong().nextBytes(iv);
Cipher c = Cipher.getInstance("AES/GCM/NoPadding");
c.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
```

### Anti-patterns
```java
// ❌ ECB mode — leaks plaintext patterns
Cipher.getInstance("AES/ECB/PKCS5Padding");

// ❌ MD5 for password
MessageDigest.getInstance("MD5").digest(password.getBytes());

// ❌ Predictable randomness
new Random().nextLong(); // for a security token
```

---

## 6. Secrets Management

### Checklist
- [ ] No secret in source, `application*.yml`, `Dockerfile`, or git history
- [ ] Secrets fetched from Spring Cloud Config Server (encrypted), HashiCorp Vault, AWS Secrets Manager,
  Azure Key Vault, or Kubernetes Secrets mounted as files
- [ ] Secret rotation supported — application reloads via `@RefreshScope` or pod restart
- [ ] Secrets never echoed in `/actuator/env` — `management.endpoint.env.show-values=NEVER` and / or
  `EnvironmentEndpoint.Sanitizer` configured
- [ ] CI/CD pipeline injects secrets at deploy time — never bakes them into images
- [ ] Secret scanning (gitleaks / trufflehog) runs in pre-commit and CI

---

## 7. Input Validation, Deserialization, and Injection

### Checklist
- [ ] Every controller method input annotated `@Valid` / `@Validated`; DTO fields carry Bean Validation
  (`@NotBlank`, `@Size`, `@Pattern`, `@Email`, `@Positive`, `@Min`, `@Max`)
- [ ] No `@RequestParam Map<String,String>` "catch-all" parameters bypassing validation
- [ ] Path / query params bound to typed objects — never raw `String` for IDs that should be `UUID` / `Long`
- [ ] **SQL Injection**: all queries use JPA / parameterized JDBC — never string-concat user input into JPQL or
  native SQL; sort/order params validated against an allowlist of column names
- [ ] **NoSQL Injection**: same rule for MongoDB / Elasticsearch query builders
- [ ] **Command Injection**: no `Runtime.exec`, `ProcessBuilder` with user input — use Java APIs
- [ ] **XXE**: `DocumentBuilderFactory`, `SAXParserFactory`, `TransformerFactory` hardened with
  `setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)` and external entity disabled
- [ ] **Java Deserialization**: no `ObjectInputStream.readObject` on untrusted bytes; Jackson `DefaultTyping`
  disabled; XStream / SnakeYAML use safe constructors
- [ ] **SSRF**: outbound HTTP destinations validated against allowlist; user-supplied URLs rejected unless explicitly trusted
- [ ] **Path Traversal**: file paths normalized via `Path.normalize()` and verified to start with the allowed root

```java
// ✅ CORRECT — record DTO with Bean Validation
public record TransferRequest(
    @NotNull UUID fromAccountId,
    @NotNull UUID toAccountId,
    @NotNull @DecimalMin("0.01") @Digits(integer = 16, fraction = 2) BigDecimal amount,
    @NotBlank @Size(max = 140) String reference) { }

@PostMapping("/transfers")
TransferResponse transfer(@Valid @RequestBody TransferRequest req) { ... }
```

### Anti-patterns
```java
// ❌ String-concatenated JPQL — SQL injection
em.createQuery("from Account where id = '" + id + "'").getResultList();

// ❌ Unvalidated sort column → injection / leak
"ORDER BY " + sortField; // sortField from query string
```

---

## 8. CORS, CSRF, Headers, Session

### Checklist
- [ ] CORS configured per-endpoint via `CorsConfigurationSource` — never `@CrossOrigin(origins = "*")` on
  authenticated routes
- [ ] Allowed origins are an explicit allowlist; `allowCredentials=true` only when wildcards absent
- [ ] CSRF enabled on cookie-authenticated UIs; `CookieCsrfTokenRepository.withHttpOnlyFalse()` for SPA flows
- [ ] Cookies: `HttpOnly`, `Secure`, `SameSite=Lax` (or `Strict` for sensitive)
- [ ] Session fixation protection on (default in Spring Security) — `migrateSession`
- [ ] Concurrent session control configured for stateful flows (`maximumSessions(1)`)
- [ ] `Content-Security-Policy` set for any HTML-serving endpoint

---

## 9. Secure Logging Policy

### Forbidden in logs (any level, including DEBUG/TRACE)
- Full PAN, CVV, expiry
- Full account number (mask to `****1234`)
- Passwords, OTP, TAC, JWT, refresh tokens, API keys
- Customer raw email / phone — hash or mask
- Stack traces of business exceptions returned to user (log + return generic message)

### Required
- [ ] SLF4J only; no `System.out` / `printStackTrace`
- [ ] Parameterized logging (`log.info("user={}", id)`) — not concatenation
- [ ] MDC populated by a filter: `traceId`, `spanId`, `tenantId`, `customerHash`
- [ ] Audit appender separate from application log; tamper-evident (sequence + signature)
- [ ] PII redaction filter (Logback `PatternLayout` converter or `MaskingMessageProvider`) catches accidental leaks

---

## 10. Anti-patterns Catalogue

| Anti-pattern                                              | Correct Alternative                                     |
| --------------------------------------------------------- | ------------------------------------------------------- |
| `@CrossOrigin(origins = "*")` on auth endpoint            | Per-endpoint `CorsConfigurationSource` with allowlist   |
| `http.csrf().disable()` on cookie-auth UI                 | Keep CSRF; switch SPA to bearer + STATELESS             |
| `@PreAuthorize("permitAll()")` on POST/PUT/DELETE         | Explicit role / scope / tenant check                    |
| Trust-all `X509TrustManager`                              | Explicit truststore + hostname verification             |
| MD5 / SHA-1 / SHA-256 for password                        | `BCryptPasswordEncoder(12)` or Argon2                   |
| `new Random()` for security tokens                        | `SecureRandom.getInstanceStrong()` or `UUID.randomUUID` |
| Hardcoded keystore password in YAML                       | Vault / Secrets Manager via `${...}`                    |
| Returning JPA entity from controller                      | DTO + MapStruct                                         |
| `e.printStackTrace()`                                     | `log.error("op failed for {}", id, e)`                  |
| Logging full request body                                 | Log a redacted summary; never PAN/PII                   |
| `ObjectMapper().enableDefaultTyping()`                    | Use `@JsonTypeInfo` per class with explicit type list   |
| `DocumentBuilderFactory.newInstance()` (no hardening)     | Disable DOCTYPE + external entities                     |


----------------------------------------------



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



# Reference: Data Persistence Layer — eclipse-regional-platform

## Table of Contents
1. JPA Entity Conventions
2. Repository and Query Patterns
3. Transactions and Isolation
4. Performance — N+1, Fetch Strategy, Pagination
5. Liquibase / Flyway Migrations
6. Connection Pool (HikariCP) and Datasource
7. Caching (Spring Cache + Redis)
8. PII / CHD at Rest
9. Anti-patterns Catalogue

---

## 1. JPA Entity Conventions

### Checklist
- [ ] Entities live in `infrastructure.persistence` — never exposed outside the persistence boundary
- [ ] No `@Data` Lombok on entities (lazy-init traps in `equals`/`hashCode`/`toString`)
  — use `@Getter`, `@Setter`, and explicit `equals`/`hashCode` based on the **business key**
  (or `id` only after persistence)
- [ ] `@Id` is a stable surrogate key — UUID v7 or DB sequence; never a natural key that can change
- [ ] `@Version` field present on aggregates that need optimistic locking
- [ ] All associations default to `LAZY` — `FetchType.EAGER` is forbidden unless the relationship is 1-1
  AND always required AND tiny
- [ ] `@OneToMany` / `@ManyToMany` use `Set` (not `List`) when ordering doesn't matter, to avoid
  Hibernate's "delete-all-then-reinsert" on collection mutation
- [ ] `cascade = CascadeType.ALL` only inside an aggregate boundary; never across aggregates
- [ ] Auditing fields (`createdAt`, `updatedAt`, `createdBy`, `updatedBy`) populated via Spring Data
  `@EntityListeners(AuditingEntityListener.class)` — never set manually
- [ ] Database column names explicit (`@Column(name = "...")`) — survive refactors

```java
// ✅ CORRECT — minimal entity skeleton
@Entity
@Table(name = "account")
@Getter @Setter @NoArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class AccountEntity {

    @Id @Column(name = "id", nullable = false, updatable = false)
    private UUID id;

    @Version
    private long version;

    @Column(name = "tenant_id", nullable = false)
    private UUID tenantId;

    @CreatedDate    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @Override public boolean equals(Object o) {
        return o instanceof AccountEntity that && Objects.equals(id, that.id);
    }
    @Override public int hashCode() { return Objects.hash(id); }
}
```

---

## 2. Repository and Query Patterns

### Checklist
- [ ] Repositories extend `JpaRepository<E, ID>` — package-private interface in `infrastructure.persistence`
- [ ] Domain layer talks to a port (interface in `domain` / `application`) — repository implements / adapts it
- [ ] Custom queries use `@Query` JPQL — **never** string-concatenate user input
- [ ] Native queries justified in code comment; parameter-bound (`:name`) only
- [ ] `@Modifying(clearAutomatically = true, flushAutomatically = true)` on bulk update / delete
- [ ] `Specification<T>` or QueryDSL for dynamic filters — never build JPQL via `String +`
- [ ] Stream-returning queries (`@QueryHints` with cursor) used for large reports — never load 1M rows into memory
- [ ] `findAll()` without `Pageable` is forbidden in production code paths

```java
// ✅ CORRECT — JPQL with named parameters and projection
public interface AccountRepository extends JpaRepository<AccountEntity, UUID> {

    @Query("""
        select new com.eclipse.regional.account.application.AccountSummary(a.id, a.displayName, a.currency)
        from AccountEntity a
        where a.tenantId = :tenantId
          and a.status = :status
        """)
    Page<AccountSummary> summariesByStatus(UUID tenantId, AccountStatus status, Pageable pageable);
}
```

### Anti-patterns
```java
// ❌ String-concatenated query — SQL injection
em.createQuery("from AccountEntity where id = '" + id + "'");

// ❌ Native query with concatenation
em.createNativeQuery("SELECT * FROM account WHERE name = '" + name + "'");

// ❌ Unbounded fetch
List<AccountEntity> all = repo.findAll();
```

---

## 3. Transactions and Isolation

### Checklist
- [ ] `@Transactional` on `@Service` methods only — **never** on controllers or repositories
- [ ] Read-only operations marked `@Transactional(readOnly = true)` — enables Hibernate optimisations
- [ ] Default propagation `REQUIRED`; `REQUIRES_NEW` only when necessary (e.g. audit must commit even if outer rolls back)
- [ ] Isolation level explicit when business requires it (`SERIALIZABLE` for balance debits, otherwise `READ_COMMITTED`)
- [ ] Long-running operations split — never hold a transaction during an external HTTP / Kafka call
- [ ] `@Transactional(rollbackFor = Exception.class)` if checked exceptions must roll back
- [ ] No self-invocation of `@Transactional` methods (Spring proxy bypass) — call via injected bean or `AopContext.currentProxy()`

```java
// ✅ CORRECT — read-only on query method
@Service
@RequiredArgsConstructor
@Transactional
class AccountService {
    private final AccountRepository repo;

    @Transactional(readOnly = true)
    public AccountView get(UUID id) { ... }

    public AccountView credit(UUID id, BigDecimal amount) { ... } // write, default propagation
}
```

### Anti-patterns
```java
// ❌ External call inside a transaction — holds DB connection
@Transactional
public void process(...) {
    repo.save(...);
    httpClient.callPartner(...); // network latency × DB connection wait
}
```

---

## 4. Performance — N+1, Fetch Strategy, Pagination

### Checklist
- [ ] `@OneToMany`, `@ManyToOne`, `@ManyToMany` all `LAZY` — verified
- [ ] Fetch joins (`join fetch`) or `@EntityGraph` used to preload associations needed by a use case
- [ ] N+1 detection enabled in tests (`hibernate.generate_statistics=true` + assertion on query count)
- [ ] Hibernate stats / `p6spy` / `datasource-proxy` reviewed for any new high-volume read path
- [ ] Native pagination via `Pageable` — never load all and slice in Java
- [ ] Cursor-based pagination (keyset) for very large or growing tables — no `OFFSET 100000`
- [ ] No `EntityManager.flush()` in a loop (autoflush only)
- [ ] Batch inserts use `hibernate.jdbc.batch_size` ≥ 50 and `order_inserts = true`

---

## 5. Liquibase / Flyway Migrations

### Checklist
- [ ] Every schema change ships as a Liquibase changeset (`db/changelog/<ticket>.xml`) or Flyway migration
  (`db/migration/V<n>__<name>.sql`) — checked into VCS
- [ ] Changesets are **immutable** once merged — never edit a deployed changeset; add a follow-up
- [ ] Author + ticket id present in each changeset
- [ ] Backward-compatible: roll forward only — no destructive change without a deprecation step
  (add column → backfill → switch reads → remove old column in a later release)
- [ ] Indexes for every foreign key and every column in a `WHERE` / `ORDER BY` of a hot query
- [ ] No `DROP TABLE` / `DROP COLUMN` in the same release that introduces the replacement
- [ ] Liquibase `preconditions` guard environment-specific changes
- [ ] Migration tested against a Testcontainers DB matching production engine + version

---

## 6. Connection Pool (HikariCP)

### Checklist
- [ ] `maximumPoolSize` sized per service — typically `(core_count * 2) + effective_disk_count`,
  capped by DB's per-service connection budget
- [ ] `connectionTimeout` < HTTP request timeout (e.g. 3s connection vs 30s request)
- [ ] `leakDetectionThreshold` set in non-prod (≥ 30s) to surface unreleased connections
- [ ] `validationTimeout`, `maxLifetime` tuned shorter than DB / network idle-kill window
- [ ] Health indicator enabled (`management.health.db.enabled=true`)
- [ ] No second datasource opened ad-hoc — multi-datasource configured via `@ConfigurationProperties`

---

## 7. Caching (Spring Cache + Redis)

### Checklist
- [ ] `@Cacheable` keys include all parameters that affect the result — including `tenantId`
- [ ] TTL set per cache region (`spring.cache.redis.time-to-live`); default ≤ 10m for mutable data
- [ ] `@CacheEvict` called on every mutation path; `allEntries=true` only when scope is small
- [ ] No PII / CHD / token cached in Redis without encryption
- [ ] Cache stampede mitigated via `@Cacheable(sync = true)` or external lock
- [ ] Cache hit/miss metrics exposed via Micrometer

---

## 8. PII / CHD at Rest

### Checklist
- [ ] PAN, CVV, full SSN / NRIC **never** persisted; if PAN must be stored (PCI scope), it is tokenised
  via the platform tokenisation service
- [ ] Sensitive columns encrypted via Hibernate `@ColumnTransformer` (KMS-wrapped key) or DB-native TDE
- [ ] Searchable encrypted fields use deterministic encryption (HMAC-prefixed) — separate index column
- [ ] Soft-delete fields (`deleted_at`) cover sensitive entities; hard delete via batch job after
  retention window per PDPA / GDPR
- [ ] Audit log table is append-only (DB user has no UPDATE/DELETE); reviewed quarterly
- [ ] Cross-region replication respects data residency (ID data → ID region only)

---

## 9. Anti-patterns Catalogue

| Anti-pattern                                            | Correct Alternative                                          |
| ------------------------------------------------------- | ------------------------------------------------------------ |
| `@Data` on JPA entity                                   | `@Getter @Setter` + explicit `equals/hashCode`               |
| `FetchType.EAGER` on collections                        | LAZY + `@EntityGraph` per use case                           |
| `@Transactional` on controller                          | Move to service; controller is HTTP-only                     |
| `findAll()` without `Pageable`                          | `Page<T> findAll(Pageable p)` capped at 100                  |
| `OFFSET 100000`                                         | Keyset / cursor pagination                                   |
| String-concat JPQL / native SQL                         | Named parameters + `Specification` / QueryDSL                |
| External HTTP call inside `@Transactional`              | Move outside TX or use saga / outbox pattern                 |
| Editing a merged Liquibase changeset                    | Add a new changeset                                          |
| Encrypting with column-level reversible cipher in DB    | KMS-wrapped envelope + Hibernate `@ColumnTransformer`        |
| Storing PAN / CVV                                       | Tokenise via platform token service                          |
| Two datasources opened in code                          | Configure via `@ConfigurationProperties` + `@Primary` bean   |


----------------------------------------------------------------



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



# Reference: Build & Deploy — eclipse-regional-platform

## Table of Contents
1. Maven / Gradle Build
2. Spring Profiles and `application-*.yml`
3. Dockerfile and Image Hardening
4. Kubernetes / Helm Deployment
5. Secrets and Config in Pipeline
6. CI/CD Pipeline Gates
7. Release Branching and Versioning
8. Anti-patterns Catalogue

---

## 1. Maven / Gradle Build

### Checklist
- [ ] Single parent POM (or root `build.gradle.kts`) declares all versions via BOM imports
  (`spring-boot-dependencies`, `spring-cloud-dependencies`)
- [ ] No `<version>` overrides for Spring-managed artifacts unless justified in a comment
- [ ] No `SNAPSHOT` dependencies in release branches
- [ ] `pom.xml` / `gradle.lockfile` committed; `./mvnw verify` / `./gradlew check` reproducible
- [ ] Plugins pinned to exact versions; no `LATEST` / `RELEASE`
- [ ] Java toolchain pinned (`<source>17</source>` / `java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }`)
- [ ] Surefire (unit `*Test`) and Failsafe (integration `*IT`) configured separately; both run in CI
- [ ] JaCoCo configured with per-module coverage thresholds matching `SKILL.md` Step 3.6
- [ ] Spotless / Checkstyle / google-java-format applied via build plugin — formatting failures fail the build
- [ ] OWASP Dependency-Check and Snyk plugins integrated; build fails on CVE ≥ HIGH

---

## 2. Spring Profiles and `application-*.yml`

### Checklist
- [ ] Profile files: `application.yml` (defaults) + one per environment
  (`application-local.yml`, `-dev.yml`, `-sit.yml`, `-uat.yml`, `-stg.yml`, `-prod.yml`) + regional variants
- [ ] Active profile set via env var `SPRING_PROFILES_ACTIVE` — never hardcoded in code
- [ ] Profile files contain **placeholders** for secrets: `${DB_PASSWORD}` — not the secret itself
- [ ] `application-prod.yml` overrides only — same key shape as base; no surprise toggles
- [ ] `local` profile may stub external services (Testcontainers / WireMock); `prod` must point to real services
- [ ] `@Profile` on configuration beans gates env-specific wiring (e.g. mock partner adapter for `local`)
- [ ] No `if (profile.equals("prod"))` branching in business code — use `@Profile` on the bean instead
- [ ] Region selected via separate profile (`my`, `sg`, `id`, `ph`) — composable with env profile
  (e.g. `SPRING_PROFILES_ACTIVE=prod,id`)
- [ ] Property precedence understood: env var > command-line > profile yml > base yml

### Prod-only restrictions to verify
```yaml
spring:
  jpa.show-sql: false                       # NEVER true in prod
  jpa.properties.hibernate.format_sql: false
  h2.console.enabled: false                 # NEVER true in prod

management:
  endpoint.env.show-values: NEVER
  endpoints.web.exposure.include: health,info,prometheus
  endpoint.health.show-details: when-authorized

springdoc:
  swagger-ui.enabled: false                 # NEVER true in prod
  api-docs.enabled: false

logging.level.org.hibernate.SQL: WARN       # never DEBUG in prod
```

---

## 3. Dockerfile and Image Hardening

### Checklist
- [ ] Multi-stage build: builder stage compiles, runtime stage holds only the JRE + jar
- [ ] Base image is an explicit minor version of a trusted distroless / Temurin / Eclipse JRE image
  (`eclipse-temurin:17-jre-jammy` or `gcr.io/distroless/java17`) — never `:latest`
- [ ] Non-root user (`USER 1000` or `USER nonroot`); container fails to start as root
- [ ] No package managers, shells, or compilers in runtime stage (distroless preferred)
- [ ] `HEALTHCHECK` defined or delegated to Kubernetes probes
- [ ] Layered jar (`spring-boot:build-image` or `layertools`) — dependencies cached separately from application classes
- [ ] No secrets in image layers (`docker history` clean); `.dockerignore` excludes `.env`, `*.pem`, `*.jks`
- [ ] Trivy / Snyk image scan passes — no CVE ≥ HIGH
- [ ] Image labelled with `org.opencontainers.image.*` metadata (source, revision, version, created)
- [ ] Image signed (cosign / Notary) and verified on cluster admission

```dockerfile
# ✅ CORRECT — distroless multi-stage
FROM eclipse-temurin:17-jdk-jammy AS builder
WORKDIR /build
COPY ../../../../../../AppData/Local/Temp .
RUN ./mvnw -B -ntp -Dmaven.test.skip package spring-boot:repackage

FROM gcr.io/distroless/java17-debian12:nonroot
WORKDIR /app
COPY --from=builder /build/target/*.jar /app/app.jar
EXPOSE 8080
USER nonroot
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app/app.jar"]
```

---

## 4. Kubernetes / Helm Deployment

### Checklist
- [ ] Resource requests + limits set (CPU + memory) — no unbounded pods
- [ ] `securityContext`:
  - `runAsNonRoot: true`
  - `runAsUser: 1000` (or distroless's `65532`)
  - `readOnlyRootFilesystem: true`
  - `allowPrivilegeEscalation: false`
  - `capabilities.drop: [ALL]`
  - `seccompProfile.type: RuntimeDefault`
- [ ] Liveness probe → `/actuator/health/liveness` (cheap; no downstream calls)
- [ ] Readiness probe → `/actuator/health/readiness` (includes critical deps)
- [ ] Startup probe set for slow-warming services (avoids liveness flapping during boot)
- [ ] Pod Disruption Budget (PDB) defined; `minAvailable` ≥ 1
- [ ] HorizontalPodAutoscaler tuned per service; based on CPU + custom metric (e.g. Kafka consumer lag)
- [ ] NetworkPolicy applied — default-deny; explicit allow per egress / ingress
- [ ] Pod Security Standard `restricted` enforced via PodSecurityAdmission label
- [ ] Service mesh (Istio / Linkerd) mTLS in `STRICT` mode within the cluster
- [ ] Secrets via `ExternalSecrets` operator (Vault / AWS / Azure) — never `kubectl create secret` checked into git
- [ ] Image pull policy `IfNotPresent` with digest pinning (`sha256:...`) in prod manifests

---

## 5. Secrets and Config in Pipeline

### Checklist
- [ ] CI secrets stored in the platform's vault (GitHub Actions secrets / GitLab CI variables masked) — never echoed
- [ ] Build never prints secrets — `--quiet` / `-q` flags; secrets passed via env var
- [ ] No long-lived cloud credentials in pipeline; use OIDC federation (GitHub → AWS / Azure) for short-lived tokens
- [ ] Container registry push credentials scoped to a single repository
- [ ] Pre-commit hook + CI step run `gitleaks` / `trufflehog` to catch accidental secret commits
- [ ] `.gitignore` excludes `*.env`, `*.pem`, `*.p12`, `*.jks`, `*.key`, `application-local.yml`

---

## 6. CI/CD Pipeline Gates

### Required gates before merge to `main` / release branch
1. Format check (Spotless / google-java-format)
2. Static analysis (SonarQube quality gate — zero new criticals)
3. Compile + unit tests (Surefire) — coverage gate per [06-testing.md](06-testing.md)
4. Integration tests (Failsafe + Testcontainers)
5. Contract tests (producer + consumer side)
6. Dependency vulnerability scan (OWASP Dependency-Check / Snyk) — fail on HIGH
7. Container image build + Trivy scan — fail on HIGH
8. Helm template + `kubeval` / `kube-linter`
9. (Release branch only) Smoke test against `uat` after deploy

### Deployment checklist
- [ ] Blue/green or canary rollout — never direct replace in prod
- [ ] DB migrations applied **before** new app version starts taking traffic; forward-compatible only
- [ ] Feature flags used for risky changes; default OFF in prod
- [ ] Rollback procedure documented and tested per release

---

## 7. Release Branching and Versioning

### Checklist
- [ ] Trunk-based development with short-lived feature branches; PRs ≤ 400 LoC reviewed
- [ ] Release branches `release/<YYYY.MM>` cut from `main`; only fixes back-ported
- [ ] Semantic-ish versioning aligned with monthly release: `2026.06.0` (release), `2026.06.1` (hotfix)
- [ ] Tag immutable on release (`git tag -s`)
- [ ] Changelog generated from conventional commits
- [ ] Hotfix PRs require CHANGE-RECORD ticket + approval from on-call + service owner

### Cadence
- R5 → 2026.05 — May 2026
- R6 → 2026.06 — June 2026
- R7 → 2026.07 — July 2026

---

## 8. Anti-patterns Catalogue

| Anti-pattern                                              | Correct Alternative                                          |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| `SNAPSHOT` dependency on release branch                   | Pin to released version                                      |
| `:latest` Docker base tag                                 | Explicit minor version + digest pin                          |
| Container runs as root                                    | `USER 1000` / distroless `nonroot`                           |
| Secrets in `application-prod.yml`                         | `${VAR}` placeholder + Vault / Secrets Manager               |
| `actuator/*` exposed publicly                             | Separate management port + auth + allowlist                  |
| Swagger UI / H2 console / `show-sql` in prod              | Disable in `application-prod.yml`                            |
| `if (env == "prod")` in business code                     | `@Profile` on configuration beans                            |
| No pod resource limits                                    | `requests` + `limits` for CPU + memory                       |
| Liveness probe calls partner API                          | Liveness shallow; readiness owns downstream checks           |
| Schema migration removes column in same release as code   | Two-phase: add → backfill → switch → remove in next release  |
| Direct deploy-replace in prod                             | Blue/green or canary                                         |


----------------------------------------------------------------------



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

