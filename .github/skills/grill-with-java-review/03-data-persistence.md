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

