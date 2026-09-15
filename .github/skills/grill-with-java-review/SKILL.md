# eclipse-regional-platform Whole Project Java Review Skill

## Purpose

When invoked with:

`/skill:grill-with-java-review`

perform a **complete whole-project backend Java/Spring Boot code review** using the authoritative Grill reference files maintained in the central Git repository.

The review MUST cover the complete application project.

The review MUST NOT be limited to current Git changes.

The user does not need to provide additional review instructions.

The skill must:

1. Identify the complete project/repository root.
2. Discover the complete project structure.
3. Identify all relevant Java/Spring Boot source code.
4. Identify configuration, test, build, deployment, messaging, and integration files.
5. Identify the affected architectural layers.
6. Retrieve the applicable Grill reference files from the central Git repository.
7. Read the actual contents of the applicable Grill reference files.
8. Review the complete project against those rules.
9. Report only confirmed findings supported by the actual project code.
10. Generate a complete Markdown review.
11. Automatically create or update `GRILLED_JAVA_REVIEW.md` in the repository root.
12. Verify that the review file was successfully created or updated.
13. Never modify production/source/configuration/test files.
14. Never ask the user for additional review instructions or confirmation.

---

# Step 1 — Identify Project Root

First determine the repository root:

```bash
git rev-parse --show-toplevel
```

Store the returned path as:

```text
REPOSITORY_ROOT
```

The complete project review must be performed relative to this directory.

If the project is not a Git repository, identify the workspace/project root using the available environment information and clearly mention that Git repository metadata was unavailable.

---

# Step 2 — Whole Project Review Scope

This skill performs a **COMPLETE PROJECT CODE REVIEW**.

The review MUST NOT be limited to:

```text
git diff
git diff --cached
git status
```

Git changes may be inspected for additional context, but Git changes are NOT the review boundary.

A clean Git working tree does NOT mean that there is nothing to review.

The skill must inspect the complete project.

Review all relevant project components, including where applicable:

```text
src/main/java
src/main/resources
src/test/java
src/test/resources
pom.xml
build.gradle
build.gradle.kts
settings.gradle
settings.gradle.kts
Docker files
Kubernetes files
Helm files
application.yml
application.yaml
application.properties
security configuration
database migration files
Kafka configuration
Feign/WebClient configuration
REST controllers
DTOs
services
repositories
entities
mappers
exception handlers
configuration classes
messaging classes
integration clients
tests
```

Do not limit the review to files modified in Git.

---

# Step 3 — Discover the Complete Project

Before reviewing findings, inspect the project structure.

Identify:

* Maven modules
* Gradle modules
* Java source directories
* Resource directories
* Test directories
* Configuration files
* Build files
* Deployment files
* Database migration files
* Messaging/integration files
* Security configuration
* API/controller packages
* Service/business packages
* Repository/persistence packages
* DTO/model packages
* Exception-handling packages
* External service clients
* Kafka producers/consumers
* Scheduled jobs
* Configuration classes

For a Maven project, inspect:

```text
pom.xml
```

For a Gradle project, inspect:

```text
build.gradle
build.gradle.kts
settings.gradle
settings.gradle.kts
```

If the project contains multiple modules, review all relevant modules.

---

# Step 4 — Central Grill Reference Repository

The Grill reference files are maintained in a central Git repository.

They are not required to be permanently stored in the application repository.

Central Grill repository:

```text
https://github.com/Sangeetha-sivanandam/CodeReview
```

Expected Grill directory:

```text
.github/skills/grill-review/grill-with-java-review/
```

Expected reference files:

```text
01-security.md
02-api-rest.md
03-data-persistence.md
04-service-business-logic.md
05-observability.md
06-testing.md
07-build-deploy.md
08-messaging-integration.md
```

The reference files in the central repository are the authoritative Grill review rules.

---

# Step 5 — Retrieve Grill Reference Files

Before starting the code review, retrieve the applicable Grill reference files from the central repository.

Use an available Git, repository, remote-file, or repository-access mechanism provided by the current environment.

The retrieval process must:

1. Access the configured central Grill repository.
2. Retrieve the latest available reference files.
3. Read the actual contents of the retrieved files.
4. Use their actual contents as the authoritative review rules.

Do not:

* Invent reference-file contents.
* Reconstruct reference-file contents from memory.
* Claim that a reference file was read when it was not retrieved.
* Assume that a reference file contains a rule without reading it.

If temporary retrieval is required, temporary files may be used.

Temporary Grill reference files must NOT be permanently added to the application repository.

Do not commit the Grill reference files into the application repository unless explicitly requested by the user.

---

# Step 6 — Required Grill References

Use the following mapping:

| Review Area                                  | Grill Reference                |
| -------------------------------------------- | ------------------------------ |
| Security/authentication/authorization        | `01-security.md`               |
| Controller/REST/DTO/validation/OpenAPI       | `02-api-rest.md`               |
| JPA/Hibernate/repository/SQL/transactions    | `03-data-persistence.md`       |
| Service/business logic/domain/mapping        | `04-service-business-logic.md` |
| Logging/MDC/metrics/tracing                  | `05-observability.md`          |
| Unit/integration/security tests              | `06-testing.md`                |
| Maven/Gradle/Docker/Kubernetes/configuration | `07-build-deploy.md`           |
| Kafka/Feign/WebClient/messaging              | `08-messaging-integration.md`  |

Read every reference relevant to the discovered project components.

If the project contains all applicable areas, read all eight reference files.

---

# Step 7 — Missing Reference Handling

If a required reference file cannot be retrieved:

1. Do not invent its contents.
2. Continue with successfully retrieved references.
3. Clearly identify the missing reference in the review.
4. Do not claim that the missing reference was read.

Example:

```text
Reference unavailable:
03-data-persistence.md could not be retrieved from the central Grill repository.
```

If none of the required Grill references can be retrieved, do not claim that a complete Grill-based review was performed.

Clearly report that the authoritative Grill references were unavailable.

---

# Step 8 — Architectural Classification

Classify the project into applicable architectural areas:

* Security
* API / REST
* DTO / validation
* Service / business logic
* Persistence
* Observability
* Testing
* Build / deployment
* Messaging
* External integrations

Review each applicable architectural area.

Do not skip an area merely because there are no current Git changes in that area.

---

# Step 9 — Security Review

Using `01-security.md`, where applicable, review the complete project for confirmed issues involving:

* Authentication bypass
* Authorization bypass
* BOLA/IDOR
* BFLA
* JWT/OAuth2 implementation
* TLS/mTLS configuration
* Hardcoded secrets
* API keys
* Passwords
* Private keys
* Weak cryptography
* Disabled security
* CORS
* CSRF
* SSRF
* Injection
* Unsafe deserialization
* Sensitive information exposure
* Input validation
* Security filter configuration
* Access-control configuration
* Security-related endpoints

Do not report a security vulnerability unless it is supported by actual project code/configuration.

---

# Step 10 — API / REST Review

Using `02-api-rest.md`, where applicable, review:

* Controllers
* REST endpoints
* DTOs
* Request validation
* Response models
* Exception handling
* HTTP status codes
* API contracts
* OpenAPI configuration
* Path variables
* Query parameters
* Request parameters
* Pagination
* Authorization
* Idempotency
* Entity exposure

Check for:

* Missing validation
* Incorrect HTTP status codes
* Incorrect error handling
* Sensitive fields in responses
* Entity exposure
* DTO/entity coupling
* Missing authorization
* Missing pagination where required
* Missing idempotency where required
* Incorrect REST design
* Invalid path/query handling
* API documentation problems

Only report confirmed issues.

---

# Step 11 — Data Persistence Review

Using `03-data-persistence.md`, where applicable, review:

* Entities
* Repositories
* JPA mappings
* Hibernate configuration
* SQL
* JPQL
* Native queries
* Transactions
* Database configuration
* Migration scripts

Check for:

* SQL injection
* Unsafe native queries
* JPQL issues
* Incorrect `@Transactional` usage
* Transaction boundary problems
* N+1 queries
* Lazy/eager loading problems
* Missing pagination
* Incorrect relationships
* Incorrect cascade behavior
* Locking problems
* Concurrency issues
* Migration problems
* Sensitive data persistence
* Connection handling

Do not claim a persistence vulnerability unless confirmed by actual code.

---

# Step 12 — Service / Business Logic Review

Using `04-service-business-logic.md`, where applicable, review:

* Service classes
* Business rules
* Domain logic
* State transitions
* Mapping logic
* Transaction boundaries
* Retry logic
* Compensation logic
* External service handling

Check for:

* Incorrect business rules
* State transition errors
* Race conditions
* Idempotency problems
* Duplicate processing
* Partial failures
* Exception handling
* Retry problems
* Compensation/rollback problems
* External service failures
* Incorrect mapping
* Business invariant violations

---

# Step 13 — Observability Review

Using `05-observability.md`, where applicable, review:

* Logging
* MDC
* Correlation IDs
* Metrics
* Tracing
* Actuator configuration

Check for sensitive information such as:

* Passwords
* OTPs
* JWTs
* Session tokens
* CVV
* Full PAN
* Full account numbers
* Sensitive customer information
* Authentication credentials

Also check:

* Missing correlation IDs
* Incorrect log levels
* Poor structured logging
* Excessive metric cardinality
* Unsafe Actuator exposure

Only report issues supported by actual code/configuration.

---

# Step 14 — Testing Review

Using `06-testing.md`, review the complete test suite.

Check for:

* Unit tests
* Controller tests
* Service tests
* Repository tests
* Integration tests
* Security tests
* Validation tests
* Negative cases
* Boundary cases
* Exception cases
* External service failure tests
* Messaging failure tests
* Idempotency tests
* Authorization tests
* Transaction rollback tests

Assess:

* Existing test coverage
* Critical business flows
* Security-sensitive flows
* Failure scenarios
* Edge cases

Do not automatically classify missing tests as HIGH severity.

Only recommend tests that are relevant to actual project behavior.

---

# Step 15 — Build / Deployment Review

Using `07-build-deploy.md`, review:

* Maven/Gradle configuration
* Dependencies
* Plugins
* Build configuration
* Docker configuration
* Kubernetes configuration
* Helm configuration
* Application configuration
* Profiles
* Health probes
* Resource configuration
* Actuator configuration

Check for:

* Vulnerable dependencies
* Unnecessary dependencies
* Unsafe production configuration
* Secrets
* Resource-limit problems
* Missing health probes
* Unsafe profiles
* Debug configuration
* Unsafe Actuator exposure

Do not claim a dependency vulnerability unless it is actually confirmed.

---

# Step 16 — Messaging / Integration Review

Using `08-messaging-integration.md`, review:

* Kafka producers
* Kafka consumers
* Event models
* Messaging configuration
* Feign clients
* WebClient clients
* External integrations
* Retry configuration
* Timeout configuration
* Circuit breakers
* Bulkheads

Check for:

* Duplicate event processing
* Missing idempotency
* Retry problems
* Dead-letter handling
* Event ID problems
* Outbox problems
* Missing timeouts
* Circuit-breaker issues
* Bulkhead issues
* SSRF
* Unsafe external response handling
* Partial failures

Only report confirmed issues.

---

# Step 17 — Cross-Cutting Security Checks

Review the entire project for:

## Secrets

Never allow:

* Passwords
* API keys
* Access tokens
* Private keys
* Signing keys
* Client secrets
* Production credentials

Check:

* Java source
* Properties
* YAML
* JSON
* Docker files
* Kubernetes files
* Build files
* CI/CD configuration

Do not expose secrets in the generated review itself.

If a secret is found, identify its location without reproducing the secret value.

---

# Step 18 — Error Handling

Review the complete project for:

* Empty catch blocks
* Swallowed exceptions
* `printStackTrace()`
* `System.out.println()`
* Internal stack traces returned to clients
* Framework internals exposed to clients
* Incorrect exception mapping
* Inconsistent error responses

Use the project's existing exception-handling standard.

---

# Step 19 — Logging

Review the complete project for unsafe logging.

Do not log:

* Passwords
* OTP
* JWT
* Session tokens
* CVV
* Full PAN
* Full account number
* Sensitive customer information

Prefer:

* Structured logging
* Parameterized logging
* Appropriate log levels
* Correlation IDs

---

# Step 20 — Code Quality

Review the complete project for:

* Constructor injection
* Business logic inside controllers
* Static mutable state
* Poor exception handling
* Incorrect transaction boundaries
* Entity exposure
* Duplicate logic
* Unsafe configuration
* Inconsistent project conventions
* Dead code
* Unused code
* Excessive complexity
* Poor separation of concerns

Do not report purely stylistic issues as HIGH or CRITICAL.

---

# Step 21 — Critical Security Findings

Use `CRITICAL` only when actually confirmed.

Examples:

* Authentication bypass
* Authorization bypass
* SQL/command injection
* XXE
* Unsafe deserialization
* Trust-all TLS
* Hostname verification disabled
* Hardcoded production secrets
* Critical sensitive-data exposure
* Broken cryptographic verification
* Production private keys committed
* Authentication disabled on sensitive endpoints

Do not inflate severity.

---

# Step 22 — Validate Every Finding

Every finding must contain:

* Severity
* File
* Line/location
* Actual problem
* Impact
* Co