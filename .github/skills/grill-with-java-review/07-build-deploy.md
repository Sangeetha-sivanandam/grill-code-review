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

