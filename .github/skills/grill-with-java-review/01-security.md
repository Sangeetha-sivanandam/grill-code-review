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

