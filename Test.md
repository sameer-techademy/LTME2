# Java Full-Stack Engineering: Production-Grade Data & Persistence Study Guide
---

## Table of Contents

1. [JPA Persistence Context & Entity Lifecycle](#1-jpa-persistence-context--entity-lifecycle)
2. [JPA Entity Modeling & Association Design](#2-jpa-entity-modeling--association-design)
3. [Spring Data JPA: Repositories, Queries & Transactions](#3-spring-data-jpa-repositories-queries--transactions)
4. [JPA Performance: N+1, Batching & Locking](#4-jpa-performance-n1-batching--locking)
5. [PostgreSQL Query Optimization & Indexing](#5-postgresql-query-optimization--indexing)
6. [Concurrency Control: Transactions, Isolation & Deadlocks](#6-concurrency-control-transactions-isolation--deadlocks)
7. [Multi-Store Data Engineering: JDBC, NoSQL & Polyglot Persistence](#7-multi-store-data-engineering-jdbc-nosql--polyglot-persistence)
8. [Spring Batch & Large-Scale ETL](#8-spring-batch--large-scale-etl)
9. [Reporting Databases, Window Functions & Summary Tables](#9-reporting-databases-window-functions--summary-tables)
10. [Data Security: PII Masking, Tokenization & Audit](#10-data-security-pii-masking-tokenization--audit)
11. [Cross-Cutting Production Patterns](#11-cross-cutting-production-patterns)
12. [Advanced Reading & References](#12-advanced-reading--references)

---

## 1. JPA Persistence Context & Entity Lifecycle

### 1.1 The Persistence Context — What It Actually Is

The Persistence Context (PC) is the in-memory unit-of-work that JPA maintains between your application code and the database. Every entity you load, save, or modify within an active session lives inside this context. It is not a cache in the conventional sense — it is a **first-level identity map** that enforces object identity and tracks mutations automatically.

**Core Guarantees:**
- Within a single PC, loading the same entity by ID twice returns the **same object reference**
- All changes to managed entities are automatically detected (dirty checking) and flushed to the database at the right time
- The PC acts as a write buffer — SQL is not necessarily issued when you call `save()`, but when the PC is flushed

```
Application Code
      │
      ▼
┌─────────────────────────────────────┐
│         Persistence Context          │
│                                     │
│  Entity A (managed) ─── snapshot    │
│  Entity B (managed) ─── snapshot    │
│  Entity C (new)                     │
│                                     │
│  Dirty Checking compares           │
│  current state vs snapshot          │
│  at flush time → generates SQL      │
└──────────────┬──────────────────────┘
               │ flush
               ▼
         Database (via JDBC)
```

### 1.2 Entity Lifecycle States

Understanding entity state is foundational to reasoning about what SQL will be generated, when, and why.

| State | Description | What triggers transition |
|---|---|---|
| **Transient** | Object exists in JVM but is unknown to JPA. No identity, no tracking. | `new Entity()` |
| **Managed** | Entity is tracked by the active PC. All changes are visible and will be flushed. | `persist()`, `merge()`, `find()`, `JPQL result` |
| **Detached** | Entity was once managed but the PC closed or it was explicitly detached. Changes are **not** tracked. | PC close, `detach()`, serialization |
| **Removed** | Entity is scheduled for deletion. Will be deleted on flush. | `remove()` |

**Critical lifecycle rules engineers break in production:**

1. **Calling `save()` on a detached entity does not throw an error — it silently merges**, potentially overwriting concurrent changes if no `@Version` is present.
2. **Accessing a lazy collection on a detached entity throws `LazyInitializationException`** — this is the single most common JPA runtime error in distributed systems.
3. **Entities returned from one transaction and passed into another are detached** — any mutation will not be tracked unless explicitly re-attached via `merge()`.

### 1.3 Dirty Checking Internals

Dirty checking is how JPA detects which managed entities have changed and need to be flushed as UPDATE statements. It works by capturing a **snapshot** of each entity's state at the moment it becomes managed.

**What gets snapshotted:**
- All basic field values (primitives, Strings, value types)
- State of embedded objects
- FK values of associations

**What dirty checking cannot detect natively:**
- Mutations to mutable types without overriding `equals`/`hashCode` (e.g., modifying a `byte[]` in-place)
- Changes to detached entities
- Changes made directly via JDBC/native SQL — the PC has no visibility into these

**Collection dirty tracking:** Hibernate wraps `Set`, `List`, and `Map` fields with its own proxy types (`PersistentSet`, `PersistentList`, etc.). These proxies detect add/remove operations. If you **replace the collection reference** entirely (e.g., `entity.setItems(new ArrayList<>())`), the dirty detection machinery is bypassed and the original collection is orphaned.

### 1.4 Flush Modes — When SQL Is Actually Issued

Flush is the act of synchronizing the PC's in-memory state with the database. The flush mode controls when this happens.

| Flush Mode | Behavior | Risk |
|---|---|---|
| `AUTO` (default) | Flushes before any query that could be affected by pending changes | Can generate unexpected SQL mid-transaction |
| `COMMIT` | Flushes only when the transaction commits | Stale reads possible if queries run before flush |
| `MANUAL` | Flushes only when explicitly called | Full developer control, easy to forget |

**`FlushMode.AUTO` edge case:** When Hibernate detects that a pending INSERT or UPDATE could affect the result of an upcoming query, it flushes preemptively. This is good for correctness but can make SQL appear at unexpected times. In bulk processing scenarios, switch to `COMMIT` or manage flush manually.

### 1.5 Second-Level Cache vs. Persistence Context

The PC is the **first-level cache** — it is scoped to a single session/transaction and cleared when it closes. The **second-level cache (L2C)** is optional, cross-session, and shared across the application.

**How they interact:**
- An entity found in the PC is returned directly — the L2C is never consulted
- When the PC misses (entity not loaded in this session), Hibernate checks L2C before going to DB
- Write-behind behavior and L2C invalidation interact: if a transaction modifies an entity, its L2C entry is invalidated on commit

**L2C pitfalls in production:**
- L2C entries can become **stale** when multiple application nodes write to the same DB without a distributed cache invalidation strategy
- In payments and financial systems, the default position should be **no L2C on transactional entities** unless you explicitly understand the invalidation model

### 1.6 Snapshot Mismatch and Detached Entity Leaks

A common failure pattern in layered architectures:

```java
// Anti-pattern: Detached entity leaking across service boundaries
@Service
public class PaymentService {
    public Payment getPaymentForAudit(Long id) {
        return paymentRepository.findById(id).orElseThrow(); // transaction ends here
        // entity is now DETACHED
    }
}

@Service
public class AuditService {
    public void auditPayment(Payment payment) {
        payment.setAuditTimestamp(Instant.now()); // mutation on detached entity — NOT tracked
        // no save() call = change silently lost
    }
}
```

**Correct pattern:** Use DTOs at service boundaries. Never pass managed entities across transaction boundaries unless the receiving method explicitly re-opens a transaction and merges.

### 1.7 Optimistic Locking with `@Version`

In concurrent systems, optimistic locking is the primary mechanism to prevent **lost updates** — where two transactions read the same entity, both modify it, and one silently overwrites the other's changes.

```java
@Entity
public class Wallet {
    @Id
    private Long id;

    @Version
    private Long version; // Hibernate manages this automatically

    private BigDecimal balance;
}
```

**How it works:**
1. On load, Hibernate captures the current `version` value
2. On flush (UPDATE), Hibernate adds `WHERE id = ? AND version = ?` to the SQL
3. If another transaction committed a change between your load and your flush, the version no longer matches → zero rows updated → `OptimisticLockException` is thrown

**Without `@Version` on mutable shared entities in a payment system, concurrent writes will silently overwrite each other.** This is a correctness bug, not a performance issue.

### 1.8 IDENTITY Generation Strategy and Batch Insert Implications

When using `@GeneratedValue(strategy = GenerationType.IDENTITY)`, the database assigns the ID on INSERT. Hibernate must issue the INSERT immediately to discover the ID — **this disables JDBC batch inserts**.

For bulk insert scenarios (e.g., settlement record creation), prefer `SEQUENCE` strategy:

```java
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "payment_seq")
@SequenceGenerator(name = "payment_seq", sequenceName = "payment_id_seq", allocationSize = 50)
```

`allocationSize = 50` means Hibernate fetches 50 IDs at a time from the DB sequence, then assigns them in-memory — allowing JDBC batching to proceed uninterrupted.

---

## 2. JPA Entity Modeling & Association Design

### 2.1 Value Types vs. Entities

Not everything in your domain model should be an `@Entity`. Choosing incorrectly impacts schema design, query complexity, and transaction behavior.

| Type | Use when | JPA construct |
|---|---|---|
| **Entity** | Object has independent identity and lifecycle; can be shared or referenced from multiple places | `@Entity` with `@Id` |
| **Embeddable** | Object is a value with no independent lifecycle; always owned by one entity | `@Embeddable` + `@Embedded` |
| **Element Collection** | Simple value list/set that doesn't need its own entity | `@ElementCollection` |

**Embeddable example — modeling an address or money amount:**

```java
@Embeddable
public class Money {
    private BigDecimal amount;
    
    @Enumerated(EnumType.STRING)
    private Currency currency;
}

@Entity
public class Payment {
    @Embedded
    private Money settlementAmount;
}
```

**`equals`/`hashCode` on Embeddables:** Embeddables must correctly implement `equals` and `hashCode` based on all fields. Hibernate uses these to detect changes to embedded objects during dirty checking. An Embeddable with no `equals` override will always appear dirty.

### 2.2 Association Mapping — Design Principles

#### Ownership and `mappedBy`

Every bidirectional association has exactly one **owning side** and one **inverse side**. The owning side controls the FK column in the database. The inverse side uses `mappedBy` and has no corresponding column.

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items;
}

@Entity
public class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id") // owning side — controls the FK
    private Order order;
}
```

**Critical rule:** Changes to the inverse side (`order.items.add(item)`) **only** take effect in the database if you also set the owning side (`item.setOrder(order)`). Forgetting this produces inconsistencies that are invisible within the same session but fail after reload.

#### Fetch Strategy Selection

| Fetch Type | Default on | Behavior | Use When |
|---|---|---|---|
| `LAZY` | `@OneToMany`, `@ManyToMany` | Association not loaded until accessed | Always — unless you have a proven need |
| `EAGER` | `@ManyToOne`, `@OneToOne` | Association always loaded with parent | Avoid stacking multiple EAGER on one entity |

**EAGER is not inherently harmful** — it becomes problematic when multiple EAGER associations stack on a single entity, multiplying the JOIN complexity of every single query that loads it, regardless of whether the associations are needed.

#### Collection Type Selection

| Type | Behavior | Use When |
|---|---|---|
| `List` | Ordered, allows duplicates, generates index column | You need ordered results or positional access |
| `Set` | Unordered, no duplicates, uses equals/hashCode | Default for most to-many associations |
| `Bag` (untyped List without `@OrderColumn`) | Allows duplicates, no order — becomes a Bag in Hibernate | Avoid — produces cartesian products in bulk fetch |

**For large collections (thousands of items), avoid loading the full collection in memory.** Use JPQL queries against the child entity directly with pagination rather than navigating `parent.getChildren()`.

### 2.3 Cascade Semantics

Cascading tells JPA to propagate operations from a parent to its associated children automatically.

| Cascade Type | Propagates | Risk |
|---|---|---|
| `PERSIST` | `persist()` calls | Safe for lifecycle-owned children |
| `MERGE` | `merge()` calls | Safe with care |
| `REMOVE` | `remove()` calls | **Dangerous on shared associations** |
| `ALL` | All of the above + REFRESH + DETACH | Use only when children are truly lifecycle-owned |

**`orphanRemoval = true`:** When an entity is removed from the parent's collection, it will be automatically deleted. This is only safe when the child entity is exclusively owned by one parent. Sharing a child between two parents with `orphanRemoval` active on either side will cause unexpected deletions.

**Cascade delete surprises in production:** Setting `cascade = CascadeType.ALL` on a `@OneToMany` with a large collection can trigger thousands of DELETE statements at flush time when the parent is removed. In high-volume systems, always use explicit bulk DELETE queries instead.

### 2.4 Bidirectional Associations and JSON Serialization

Bidirectional associations cause **infinite recursion** when naively serialized to JSON (each side references the other). Common solutions:

```java
// Option 1: Break the cycle with @JsonIgnore on the inverse side
@OneToMany(mappedBy = "customer")
@JsonIgnore
private List<Order> orders;

// Option 2: Use @JsonManagedReference / @JsonBackReference
@OneToMany(mappedBy = "customer")
@JsonManagedReference
private List<Order> orders;

@ManyToOne
@JsonBackReference
private Customer customer;

// Option 3 (preferred): Use DTOs at the API layer and never serialize entities directly
```

**Best practice:** Never expose JPA entities directly in API responses. Map to DTOs explicitly. This avoids serialization cycles, decouples API contracts from schema changes, and prevents lazy-load triggers during serialization.

### 2.5 Missing FK Indexes — A Silent Performance Killer

Every FK column should have a database index unless it is guaranteed to be used exclusively in bulk operations. Without an index on the FK:

- `JOIN` queries perform full table scans on the child table
- `ON DELETE CASCADE` operations iterate the child table row by row
- `findByParentId` queries degrade linearly with data volume

JPA does not automatically create FK indexes. You must declare them explicitly:

```java
@Table(indexes = {
    @Index(name = "idx_order_item_order_id", columnList = "order_id"),
    @Index(name = "idx_order_item_product_id", columnList = "product_id")
})
```

---

## 3. Spring Data JPA: Repositories, Queries & Transactions

### 3.1 Repository Abstraction — What It Actually Does

Spring Data JPA generates repository implementations at runtime, based on method signatures, annotations, and a set of well-defined conventions. The actual SQL or JPQL is **generated from the method name**, not written by you.

```java
public interface PaymentRepository extends JpaRepository<Payment, Long> {
    // Derived query — Spring Data generates JPQL from the method name
    List<Payment> findByMerchantIdAndStatusAndCreatedAtBetween(
        Long merchantId, PaymentStatus status, 
        LocalDateTime from, LocalDateTime to
    );
}
```

**Derived query limitations:** Method names grow unwieldy for complex queries. Beyond 3–4 conditions, switch to `@Query` with explicit JPQL or native SQL. Derived queries for complex conditions also produce inefficient SQL that ignores selectivity estimates.

### 3.2 Query Approaches and When to Use Each

| Approach | Best for | Limitations |
|---|---|---|
| Derived query methods | Simple field-equality / range queries | Gets unwieldy beyond ~3 conditions |
| `@Query` (JPQL) | Moderate complexity; entity-centric; portable | Cannot use all DB-specific features |
| `@Query(nativeQuery = true)` | Full SQL power; DB-specific optimizations | Tied to schema; bypasses entity lifecycle |
| `Specification` (JPA Criteria) | Dynamic, runtime-built predicates | Verbose API; use for complex filtering UIs |
| `QueryDSL` | Compile-safe, complex dynamic queries | Requires code generation |

**JPQL vs native SQL:** JPQL operates on entities and their mapped fields. Native SQL bypasses the entity model entirely and returns raw result sets. For bulk reporting, analytics, and complex aggregations, native SQL is generally more appropriate. For operations that need to interact with the entity lifecycle, JPQL or the Criteria API is correct.

**Pagination with `JOIN FETCH` is broken:**

```java
// This will generate a warning and potentially incorrect results:
@Query("SELECT p FROM Payment p JOIN FETCH p.items WHERE p.merchant.id = :id")
Page<Payment> findWithItemsByMerchant(@Param("id") Long id, Pageable pageable);
```

When you use `JOIN FETCH` with `Pageable`, Hibernate cannot apply the LIMIT at the SQL level without loading all rows first (because the JOIN creates duplicates). Hibernate warns you about this and performs in-memory pagination — this defeats the purpose. Use separate queries or entity graphs with count queries.

### 3.3 DTO Projections — Fetch Only What You Need

Loading full entities for read-only API responses wastes memory and generates SQL that selects unnecessary columns. Use projections:

```java
// Interface-based projection
public interface PaymentSummary {
    Long getId();
    String getStatus();
    BigDecimal getAmount();
    String getMerchantName(); // can span associations
}

public interface PaymentRepository extends JpaRepository<Payment, Long> {
    List<PaymentSummary> findProjectedByMerchantId(Long merchantId);
}
```

Or with explicit JPQL and constructor expression:

```java
@Query("SELECT new com.example.dto.PaymentSummaryDto(p.id, p.status, p.amount, m.name) " +
       "FROM Payment p JOIN p.merchant m WHERE p.merchant.id = :id")
List<PaymentSummaryDto> findSummariesByMerchant(@Param("id") Long id);
```

### 3.4 Transaction Management in Depth

#### `@Transactional` — What It Does and Doesn't Do

`@Transactional` on a Spring bean method creates a proxy that begins a transaction before the method executes and commits (or rolls back) after it returns. **It only works when called from outside the bean** — calling a `@Transactional` method from within the same class bypasses the proxy entirely.

```java
@Service
public class PaymentService {
    
    // ✗ Anti-pattern: calling @Transactional from within the same class
    public void processPayments(List<Long> ids) {
        for (Long id : ids) {
            processOne(id); // @Transactional on processOne is ignored here
        }
    }

    @Transactional
    public void processOne(Long id) { ... }
}
```

#### Propagation Semantics

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Joins existing transaction; creates new one if none exists |
| `REQUIRES_NEW` | Always creates a new transaction; **suspends** the current one |
| `SUPPORTS` | Joins if exists; non-transactional if not |
| `NOT_SUPPORTED` | Suspends current transaction; runs non-transactionally |
| `NEVER` | Throws if a transaction already exists |
| `MANDATORY` | Throws if no transaction exists |
| `NESTED` | Savepoint within existing transaction; partial rollback possible |

**`REQUIRES_NEW` suspends, not commits, the outer transaction.** If the outer transaction rolls back after the inner completes, the inner's changes persist independently. This is intentional for cases like audit logging, but can cause inconsistency if misapplied to business operations.

#### `@Transactional(readOnly = true)`

Declaring a transaction as read-only does several things:
- Hints to Hibernate to skip dirty checking at flush time (performance improvement)
- Hints to the JDBC driver that no writes will occur (some drivers optimize connection handling)
- Enables routing to read replicas in multi-datasource setups

**It does not make the transaction literally read-only at the DB level by default.** If you need DB-enforced read-only, configure it at the JDBC connection level.

#### Rollback Rules

By default, Spring only rolls back on `RuntimeException` and `Error`. Checked exceptions do **not** trigger rollback unless explicitly declared:

```java
@Transactional(rollbackFor = {PaymentException.class, SettlementException.class})
public void settlePayment(Long id) throws PaymentException { ... }
```

**Exceptions inside `@Async` methods and lambda streams do not propagate to the calling transaction** — they are swallowed or surface in a different thread context. Always explicitly handle exceptions in async code and design for idempotency.

### 3.5 Bulk Operations and the Persistence Context

JPQL bulk UPDATE and DELETE statements **bypass the entity lifecycle entirely**:

```java
@Modifying
@Query("UPDATE Payment p SET p.status = :status WHERE p.batchId = :batchId")
int updateStatusByBatch(@Param("status") PaymentStatus status, @Param("batchId") Long batchId);
```

Consequences:
- Entity lifecycle callbacks (`@PreUpdate`, `@PostUpdate`) are **not fired**
- `@Version` is **not incremented** — optimistic lock conflicts will not be detected
- The PC is **not updated** — if you load any of the affected entities before or after, they will show stale state

**Always call `entityManager.clear()` or reload affected entities after a bulk operation** if they might be accessed again in the same session.

### 3.6 Locking in Spring Data JPA

```java
// Pessimistic write lock — blocks other readers and writers on the same row
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT w FROM Wallet w WHERE w.id = :id")
Optional<Wallet> findByIdForUpdate(@Param("id") Long id);

// Optimistic lock — version check at flush time
@Lock(LockModeType.OPTIMISTIC)
Optional<Wallet> findById(Long id);
```

Configure lock timeouts to avoid indefinite blocking:

```java
@QueryHints({@QueryHint(name = "javax.persistence.lock.timeout", value = "3000")})
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Wallet> findByIdForUpdate(Long id);
```

---

## 4. JPA Performance: N+1, Batching & Locking

### 4.1 The N+1 Problem — Mechanics and Detection

The N+1 problem occurs when loading a collection of entities, then triggering a separate query for each entity's lazy association. With N entities, you get 1 (load list) + N (load association per entity) = N+1 queries.

```java
// Loading 100 payments, then accessing merchant on each → 101 queries
List<Payment> payments = paymentRepository.findAll(); // 1 query
for (Payment p : payments) {
    String name = p.getMerchant().getName(); // 1 query per payment → 100 queries
}
```

**How to detect it:**
- Enable SQL logging: `spring.jpa.show-sql=true` and `logging.level.org.hibernate.SQL=DEBUG`
- Count the number of SELECT statements in a single request
- Use `spring.jpa.properties.hibernate.generate_statistics=true` and check `session.statistics`
- In production, use APM tools (Datadog, Dynatrace) to track query counts per transaction

### 4.2 Solving N+1 — Fetch Join, Entity Graph, Batch Size

#### Option 1: JPQL JOIN FETCH

```java
@Query("SELECT DISTINCT p FROM Payment p JOIN FETCH p.merchant WHERE p.status = :status")
List<Payment> findWithMerchantByStatus(@Param("status") PaymentStatus status);
```

**Important limitations:**
- `JOIN FETCH` on a collection association (`OneToMany`) with `DISTINCT` works but Hibernate still loads all rows before deduplication
- You **cannot safely paginate** with `JOIN FETCH` on collection associations (see Section 3.2)
- Fetching multiple collection associations with `JOIN FETCH` in a single query produces a **cartesian product** — always JOIN FETCH only one collection per query

#### Option 2: `@EntityGraph`

```java
@EntityGraph(attributePaths = {"merchant", "items"})
List<Payment> findByStatus(PaymentStatus status);
```

Entity graphs are equivalent to JOIN FETCH but declared outside the query string. They still create JOINs internally and carry the same pagination limitation. They do not eliminate N+1 if the graph structure conflicts with default fetch modes on nested associations.

#### Option 3: `@BatchSize` — Lazy Loading in Batches

```java
@OneToMany(mappedBy = "payment", fetch = FetchType.LAZY)
@BatchSize(size = 50)
private List<PaymentItem> items;
```

With `@BatchSize(size = 50)`, when you first access `items` for one Payment, Hibernate fetches the items for up to 50 Payments at once using an SQL `IN` clause. This turns N queries into `ceil(N/50)` queries.

**`@BatchSize` for to-one vs to-many:**
- For to-one (`@ManyToOne`), `@BatchSize` is applied at the entity level, not the association level
- For to-many, it is applied at the association level as shown above

### 4.3 JDBC Batch Inserts and Updates

Batching collects multiple INSERT or UPDATE statements and sends them to the database in a single network round-trip. This dramatically reduces overhead for bulk writes.

**Required configuration:**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true   # groups inserts of the same type
        order_updates: true   # groups updates of the same type
        generate_statistics: true  # for verifying batching is active
```

**What breaks batching:**
- `GenerationType.IDENTITY` — must use `SEQUENCE` (see Section 1.8)
- Mixing entity types within a batch without `order_inserts = true`
- Very large batches that exhaust PC memory — use `flush()` and `clear()` after each batch window

```java
@Transactional
public void bulkInsert(List<Transaction> transactions) {
    for (int i = 0; i < transactions.size(); i++) {
        entityManager.persist(transactions.get(i));
        if (i > 0 && i % 50 == 0) {
            entityManager.flush();
            entityManager.clear(); // prevents PC memory growth
        }
    }
}
```

**Batch updates and deletes:** The same batching mechanism applies to UPDATEs. For DELETE operations at very high volume, consider using bulk JPQL DELETE queries (acknowledging the lifecycle bypass tradeoff from Section 3.5).

### 4.4 Batching Operational Constraints

When designing bulk operations, account for:

| Constraint | Consideration |
|---|---|
| **Memory** | PC grows with each persisted entity; flush+clear periodically |
| **Transaction duration** | Long transactions hold DB locks; use chunked processing |
| **Retry behavior** | A mid-batch failure may leave partial state; design for idempotency |
| **DB lock contention** | Multiple batch threads writing the same rows will deadlock |
| **Indexing impact** | Heavy inserts can slow down index maintenance; consider disabling non-critical indexes for bulk loads |

### 4.5 Optimistic vs. Pessimistic Locking — Choosing Correctly

| Strategy | Mechanism | Use When |
|---|---|---|
| **Optimistic** | `@Version` field, fail at flush time | Low contention; writes are infrequent relative to reads |
| **Pessimistic Read** | DB `SELECT FOR SHARE` | You need to prevent writes while reading, but allow other reads |
| **Pessimistic Write** | DB `SELECT FOR UPDATE` | You need exclusive access; contention is expected |

**Decision guidance for financial systems:**

```
                  ┌─────────────────────────────────────────────┐
                  │         Is contention expected?             │
                  └─────────────┬───────────────────────────────┘
                                │
           ┌────────────────────┴────────────────────┐
           │ No / low contention                     │ Yes / moderate-high
           ▼                                         ▼
  Use @Version (optimistic)             Is operation latency-sensitive?
  Handle OptimisticLockException              │
  with retry                          ┌───────┴────────┐
                                      │ Yes            │ No
                                      ▼                ▼
                               Consider         PESSIMISTIC_WRITE
                               queuing          with lock timeout
                               pattern
```

**Pessimistic locking pitfalls:**
- `PESSIMISTIC_WRITE` without a lock timeout (`javax.persistence.lock.timeout`) can cause threads to block indefinitely if the lock holder is slow or fails
- Lock ordering matters for deadlock prevention: always acquire locks on entities in a consistent order across all code paths
- Never hold pessimistic locks across user-facing wait times (e.g., waiting for an HTTP response before committing)

---

## 5. PostgreSQL Query Optimization & Indexing

### 5.1 How the Query Planner Works

PostgreSQL uses a **cost-based optimizer** to choose among candidate query plans. It estimates costs using:

- **Table statistics** collected by `ANALYZE` (row counts, column value distributions, histograms)
- **Page estimates** from `pg_class`
- **Configuration parameters** like `random_page_cost`, `effective_cache_size`, `work_mem`

When the planner makes a wrong choice, it is almost always due to **stale or inaccurate statistics**. Running `ANALYZE` after bulk loads, or increasing `default_statistics_target` for skewed columns, is frequently the fix.

### 5.2 Reading EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT p.id, m.name, p.amount
FROM payments p
JOIN merchants m ON m.id = p.merchant_id
WHERE p.status = 'PENDING'
  AND p.created_at >= NOW() - INTERVAL '7 days';
```

**Key fields to interpret:**

| Field | Meaning |
|---|---|
| `Seq Scan` | Full table scan — no useful index found |
| `Index Scan` | Walks the index tree and fetches heap pages per row |
| `Index Only Scan` | All needed data in the index — no heap fetch |
| `Bitmap Index Scan` | Collects qualifying row pointers from index, then batch-fetches heap pages |
| `Bitmap Heap Scan` | The heap-fetch phase after Bitmap Index Scan |
| `rows=X (actual rows=Y)` | Planner estimate vs actual — large deviation = stats issue |
| `Buffers: shared hit=X read=Y` | Cache hits vs disk reads |
| `cost=startup..total` | Planner's cost estimate; lower is better |

**A large discrepancy between estimated and actual rows is the most common root cause of poor query plans.** Fix it by running `ANALYZE payments` or increasing `ALTER TABLE payments ALTER COLUMN status SET STATISTICS 500`.

### 5.3 Index Types and When to Use Each

#### B-Tree (default)

Supports `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN`, `IS NULL`, pattern prefix (`LIKE 'abc%'`). The overwhelming default for most production use cases.

```sql
-- Single column
CREATE INDEX idx_payments_status ON payments(status);

-- Composite — left-to-right prefix rule applies
CREATE INDEX idx_payments_merchant_status_date
    ON payments(merchant_id, status, created_at DESC);
```

**Composite index key rule:** A composite index `(A, B, C)` is usable by queries filtering on `A`, `A+B`, or `A+B+C`. It is **not** usable for filtering on `B` alone or `C` alone. Put the highest-selectivity, most-commonly-filtered column first.

**Index ordering matters for sort elimination:** If your query sorts by `created_at DESC`, defining the index with `DESC` can allow the planner to satisfy the ORDER BY without a separate sort step.

#### Partial Index

Indexes a subset of rows, saving space and maintenance overhead.

```sql
-- Index only PENDING payments — much smaller than full status index
CREATE INDEX idx_payments_pending
    ON payments(created_at, merchant_id)
    WHERE status = 'PENDING';
```

For columns with very skewed distributions (e.g., 99% of rows are `SETTLED`, 1% are `PENDING`), a partial index on the rare value is far more selective and efficient than a full index.

#### INCLUDE Columns (Covering Index / Index-Only Scan)

Add non-key columns to the index leaf pages so the query can be satisfied entirely from the index without accessing the heap.

```sql
CREATE INDEX idx_payments_merchant_covering
    ON payments(merchant_id, created_at)
    INCLUDE (amount, status, currency);
```

This enables **index-only scans** for queries selecting `amount`, `status`, and `currency` filtered by `merchant_id`. Requirements: the **visibility map** must mark the page as all-visible; run `VACUUM` regularly.

#### GIN / GiST — for Full-Text and JSONB

```sql
-- JSONB field indexing
CREATE INDEX idx_payments_metadata ON payments USING GIN(metadata);

-- Full-text
CREATE INDEX idx_payments_description_fts 
    ON payments USING GIN(to_tsvector('english', description));
```

### 5.4 Index Maintenance — Operational Reality

Indexes are not free. They consume storage and incur write overhead on every INSERT, UPDATE, and DELETE.

| Problem | Symptom | Fix |
|---|---|---|
| **Index bloat** | Index grows even after deletes/updates | `REINDEX CONCURRENTLY` |
| **Dead tuples** | Queries slow over time post bulk delete | `VACUUM ANALYZE` |
| **Poor fill factor** | Frequent updates fragment index pages | `WITH (fillfactor = 70)` |
| **Misestimation from skew** | Planner picks wrong plan | Increase statistics target for that column |
| **Unused indexes** | Write overhead with no read benefit | Monitor `pg_stat_user_indexes`; drop unused |

```sql
-- Find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY schemaname, tablename;
```

### 5.5 Query Shape and Index Compatibility

Index usage can be silently defeated by query shape. Common patterns that break index usage:

```sql
-- ✗ Function on indexed column defeats index
WHERE UPPER(email) = 'USER@EXAMPLE.COM'
-- ✓ Fix: use functional index or store normalized form
CREATE INDEX idx_users_email_upper ON users(UPPER(email));

-- ✗ Implicit cast defeats index
WHERE payment_id = '12345'  -- payment_id is integer, '12345' is text
-- ✓ Fix: use correct type
WHERE payment_id = 12345

-- ✗ LIKE with leading wildcard
WHERE description LIKE '%payment%'
-- ✓ Fix: use GIN full-text index or pg_trgm for arbitrary substring search

-- ✗ OR across different columns (planner may choose seq scan)
WHERE status = 'PENDING' OR merchant_id = 5
-- ✓ Fix: UNION of two index scans, or reconsider the query
```

### 5.6 Planner Statistics and `pg_statistic`

The planner builds internal statistics from `ANALYZE`. For skewed distributions (e.g., one merchant has 80% of all payments), the default statistics may be insufficient.

```sql
-- Check column statistics
SELECT * FROM pg_stats WHERE tablename = 'payments' AND attname = 'merchant_id';

-- Increase statistics target for a critical column
ALTER TABLE payments ALTER COLUMN merchant_id SET STATISTICS 500;
ANALYZE payments;
```

A high discrepancy between estimated and actual row counts in `EXPLAIN ANALYZE` is your signal that the planner is working with stale or insufficient statistics.

---

## 6. Concurrency Control: Transactions, Isolation & Deadlocks

### 6.1 Transaction Isolation Levels — What Each Actually Prevents

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Write Skew |
|---|---|---|---|---|
| `READ UNCOMMITTED` | Possible | Possible | Possible | Possible |
| `READ COMMITTED` | Prevented | Possible | Possible | Possible |
| `REPEATABLE READ` | Prevented | Prevented | Prevented* | Possible |
| `SERIALIZABLE` | Prevented | Prevented | Prevented | Prevented |

*In PostgreSQL, `REPEATABLE READ` prevents phantom reads through MVCC snapshots (differs from the SQL standard definition).

**Choosing isolation per use case:**

| Use Case | Recommended Level | Reasoning |
|---|---|---|
| Normal OLTP (payment create, status update) | `READ_COMMITTED` | Sufficient correctness; minimizes lock contention |
| Balance reads within a multi-step operation | `REPEATABLE_READ` | Prevents non-repeatable reads without full serialization |
| Settlement batch | `REPEATABLE_READ` or `SERIALIZABLE` | Need consistent view across entire settlement window |
| Audit / reporting query | `READ_COMMITTED` with `readOnly=true` | Snapshot consistency at statement level; no contention |

Set isolation level in Spring:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public SettlementResult runSettlement(Long merchantId, LocalDate date) { ... }
```

### 6.2 MVCC in PostgreSQL — Reads Never Block Writes

PostgreSQL implements **Multi-Version Concurrency Control (MVCC)**. Readers never block writers; writers never block readers. Each transaction sees a consistent snapshot of the data as of its start time (or the start of each statement, at `READ_COMMITTED`).

**Implications:**
- A long-running read transaction holds a snapshot that prevents `VACUUM` from reclaiming dead tuples → table bloat
- High write load against a table being read by a long transaction will accumulate dead versions → performance degrades
- This is why settlement jobs should not run during peak OLTP hours unless on a replica

### 6.3 Deadlocks — How They Happen and How to Prevent Them

A deadlock occurs when two or more transactions each hold a lock that the other is waiting for.

```
Transaction A:                      Transaction B:
LOCK wallet_id = 100                LOCK wallet_id = 200
(waiting for wallet_id = 200)       (waiting for wallet_id = 100)
          ↑_____________________________↓
                    DEADLOCK
```

PostgreSQL detects deadlocks automatically and rolls back the **victim** transaction (the one with the lower cost to abort). Your application receives a `SQLState 40P01` (or `DeadlockException` via Spring).

**Prevention strategies:**

1. **Consistent lock ordering:** Always acquire locks on the same entities in the same order across all code paths
2. **Keep transactions short:** Minimize the time locks are held
3. **Lock at the start:** Acquire all needed locks at transaction start rather than lazily as processing proceeds
4. **Use advisory locks** for coarse-grained mutual exclusion (e.g., one settlement job per merchant at a time)

```java
// Always lock wallets in ascending ID order to prevent deadlock
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    Long first = Math.min(fromId, toId);
    Long second = Math.max(fromId, toId);
    Wallet w1 = walletRepository.findByIdForUpdate(first);
    Wallet w2 = walletRepository.findByIdForUpdate(second);
    // ... proceed with transfer
}
```

### 6.4 Retry Strategies for Lock Failures

Both optimistic lock failures (`OptimisticLockException`) and deadlock errors are **transient** — they should be retried, not reported as application errors.

```java
@Service
public class PaymentService {

    @Retryable(
        include = {OptimisticLockingFailureException.class, DeadlockLoserDataAccessException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2, random = true)
    )
    @Transactional
    public void processPayment(Long paymentId) {
        // ... payment logic
    }

    @Recover
    public void recoverFromLockFailure(OptimisticLockingFailureException e, Long paymentId) {
        log.error("Payment {} failed after retries: {}", paymentId, e.getMessage());
        // alert, dead-letter queue, etc.
    }
}
```

**Retry + idempotency:** A retry must be safe to execute multiple times. Ensure all state-changing operations are idempotent — typically via a unique idempotency key that the DB will reject on duplicate:

```sql
CREATE UNIQUE INDEX idx_payments_idempotency ON payments(idempotency_key);
```

### 6.5 Detecting and Diagnosing Lock Contention

```sql
-- See what is currently blocked and what is blocking it
SELECT
    blocked.pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- Check lock modes held
SELECT pid, mode, granted, relation::regclass
FROM pg_locks
WHERE NOT granted;
```

### 6.6 Isolation in Spring — Mapping to JPA and JDBC

**Important:** Setting `@Transactional(isolation = ...)` configures the **JDBC connection's isolation level**, not a JPA-level concept. This requires that the connection is actually obtained with the requested level. Some connection pools reset isolation per checkout — verify this in your pool configuration.

For JDBC-level lock timeout:

```java
@PersistenceContext
private EntityManager em;

public void lockWithTimeout(Long id, int timeoutMs) {
    Map<String, Object> hints = Map.of("javax.persistence.lock.timeout", timeoutMs);
    Wallet wallet = em.find(Wallet.class, id, LockModeType.PESSIMISTIC_WRITE, hints);
}
```

**Mixing DB locks with distributed locks (e.g., Redis):** If your system acquires a Redis lock and then a DB lock, or vice versa, you can create a two-resource deadlock class. Always define and document lock acquisition order across both systems, and set timeouts on both levels.

---

## 7. Multi-Store Data Engineering: JDBC, NoSQL & Polyglot Persistence

### 7.1 Why Polyglot Persistence

A single data store rarely satisfies all the requirements of a production payment or financial system. Different workloads have fundamentally different access patterns:

| Workload | Requirement | Best-fit Store |
|---|---|---|
| OLTP — payment create/update | Strong consistency, FK integrity, low latency | PostgreSQL / MySQL |
| Session / rate-limit state | Fast reads/writes, TTL, atomic counters | Redis |
| Time-series metrics | High write throughput, range queries | TimescaleDB / InfluxDB |
| Audit / event log | Append-only, high volume, full-text search | Elasticsearch / OpenSearch |
| Document-centric config | Flexible schema, nested queries | MongoDB |
| Wide-column analytics | Partition-aware, linear scalability | Cassandra |

Choosing the right store for each concern is a systems design decision that affects consistency guarantees, operational complexity, and failure modes — not just performance.

### 7.2 CAP Theorem and PACELC — The Right Mental Model

The **CAP theorem** states that a distributed system can provide at most two of: **Consistency**, **Availability**, and **Partition Tolerance**. Since network partitions are unavoidable in production, the real choice is between **CP** (favor consistency during partition) and **AP** (favor availability during partition).

**CAP is a simplified model.** The **PACELC framework** is more useful in practice: even when there is no partition, you face a trade-off between **Latency** and **Consistency**.

| System | CAP Choice | PACELC Trade-off |
|---|---|---|
| PostgreSQL | CP | Low latency over perfect consistency (READ_COMMITTED) |
| Redis (cluster) | AP | Favors latency; replica lag possible |
| Cassandra | AP (tunable) | Tunable via consistency levels; favors latency by default |
| MongoDB | CP (tunable) | Write concern controls durability vs latency |

**Operational implication:** An AP system like Redis can serve stale data after a network event. Do not use Redis as the system of record for financial balances. Use it as a cache or rate-limit counter, with the authoritative state always in a CP store.

### 7.3 JDBC Streaming for Large Result Sets

Standard JPA/JDBC queries load the entire result set into memory. For large datasets (millions of rows), this causes OOM errors or excessive GC pressure.

```java
// Stream large result sets from JDBC without loading all rows into memory
@QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "" + Integer.MIN_VALUE))
@Query(value = "SELECT * FROM transactions WHERE batch_id = ?1", nativeQuery = true)
Stream<Transaction> streamByBatchId(Long batchId);

// Always close the stream in a try-with-resources
@Transactional(readOnly = true)
public void processBatch(Long batchId) {
    try (Stream<Transaction> txns = txnRepository.streamByBatchId(batchId)) {
        txns.forEach(txn -> process(txn));
    }
}
```

**JDBC streaming limitations:**
- The DB connection is held open for the duration of the stream — long streams hold connections from the pool
- Multi-threaded access to the same stream is not safe
- Some JDBC drivers require dedicated connection settings (e.g., MySQL requires `fetchSize = Integer.MIN_VALUE` for true streaming; PostgreSQL requires `autoCommit = false`)
- Server-side cursors (PostgreSQL) vs client-side buffering (MySQL) behave differently — understand your driver's behavior

### 7.4 Redis — Production Patterns and Pitfalls

#### Correct Use Cases

```java
// Rate limiting — atomic increment with TTL
@Component
public class RateLimiter {
    private final StringRedisTemplate redis;

    public boolean allow(String merchantId) {
        String key = "rate:merchant:" + merchantId;
        Long count = redis.opsForValue().increment(key);
        if (count == 1) {
            redis.expire(key, Duration.ofSeconds(60));
        }
        return count <= MAX_REQUESTS_PER_MINUTE;
    }
}
```

#### Critical Pitfalls

**Hot key problem:** A single Redis key receiving extremely high throughput (e.g., a global counter) becomes a bottleneck. In Redis Cluster, hot keys are constrained to a single shard. Solutions: local in-memory counters that flush to Redis periodically, key sharding with aggregation.

**Replica lag:** Redis replication is asynchronous. After a write to the primary, a read from a replica may return the old value for a brief window. For operations requiring read-your-own-writes, always read from the primary.

**Eviction under memory pressure:** When Redis reaches `maxmemory`, it evicts keys according to the configured policy (`allkeys-lru`, `volatile-lru`, etc.). If your rate limiter or session keys are evicted, operations that depend on them will behave as if the data never existed (e.g., rate limit resets). Use `noeviction` for data that must never be silently dropped, and handle `OOM` errors explicitly.

**Redis failover scenarios:** During a Redis Sentinel or Cluster failover, there is a brief window where writes may fail. Application code using Redis must handle `RedisConnectionException` gracefully and implement fallback behavior (fail open or fail closed, depending on the use case).

### 7.5 Cassandra — Production Patterns and Pitfalls

Cassandra's data model is fundamentally query-driven: **you design tables around queries, not entities**.

```
// Anti-pattern: thinking relationally
Table: transactions (id, merchant_id, user_id, amount, status, created_at)
// You cannot efficiently query by merchant_id AND date range without a full scan

// Cassandra pattern: table per query
Table: transactions_by_merchant (
    merchant_id    -- partition key
    created_at     -- clustering key
    transaction_id
    amount
    status
)
PRIMARY KEY ((merchant_id), created_at, transaction_id)
```

#### Consistency Levels

| Level | Description | Use When |
|---|---|---|
| `ONE` | Responds when one replica acknowledges | Fastest; suitable for non-critical reads |
| `QUORUM` | Majority of replicas | Default for balanced consistency+availability |
| `ALL` | All replicas must respond | Strongest consistency; single node failure = failure |
| `LOCAL_QUORUM` | Quorum within local datacenter | Multi-DC deployments |

For financial writes, use at least `QUORUM`. For critical reads (balance checks), use `QUORUM` or `LOCAL_QUORUM`.

#### Cassandra Pitfalls

**Tombstone buildup:** Deletes in Cassandra do not immediately remove data — they create tombstones that are cleaned up during compaction. High-delete workloads cause tombstone accumulation that degrades read performance. Design schemas to avoid frequent deletes; use TTL for expiring data instead of explicit deletes.

**Partition size limits:** A single Cassandra partition should not grow beyond ~100MB or ~100K rows. Unbounded time-series data stored under a single partition key (e.g., `merchant_id` alone) will eventually exceed this. Add a time bucket to the partition key: `(merchant_id, year_month)`.

**Write concern vs read repair:** Cassandra trades consistency for availability. If a write succeeds at `QUORUM` but one replica is down, that replica will be out of sync until read repair or hinted handoff restores it. Design reads to tolerate brief inconsistency, or use `QUORUM` reads to detect and repair divergence.

### 7.6 MongoDB — Production Patterns and Pitfalls

MongoDB suits use cases with flexible or nested document schemas (e.g., merchant configuration, product catalogs, event payloads).

#### Write Concerns

```java
MongoCollection<Document> collection = db.getCollection("events")
    .withWriteConcern(WriteConcern.MAJORITY); // ensures durability across replica set
```

`WriteConcern.MAJORITY` ensures the write is acknowledged by the majority of replica set members before returning. For financial audit events, always use `MAJORITY` — the default `WriteConcern.ACKNOWLEDGED` only confirms the primary received the write.

#### Pitfalls

**Unbounded document growth:** Embedding unbounded arrays in a document (e.g., all events for a payment appended to the payment document) will eventually hit the 16MB BSON limit and degrade performance as the document grows. Use a separate collection for high-volume child records.

**Secondary index design:** Like relational databases, MongoDB queries without index support perform collection scans. For compound indexes, the ESR rule applies: **Equality fields first, Sort fields second, Range fields last**.

**Lack of joins — design implication:** MongoDB does not support traditional joins (only `$lookup` in aggregation, which is expensive). Denormalize strategically: embed data that is always read together, reference data that has independent lifecycle.

### 7.7 Cross-Store Consistency — Outbox and CDC

When writing to multiple stores in a single business operation, distributed consistency requires explicit coordination. Two common patterns:

#### Transactional Outbox Pattern

```
Business Transaction:
  1. INSERT into payments table
  2. INSERT into outbox table (in the SAME DB transaction)

Background process:
  3. Read unprocessed outbox entries
  4. Publish to Redis / Kafka / Cassandra
  5. Mark outbox entry as processed
```

This guarantees that the event is published if and only if the business transaction committed, because both writes are atomic within the same DB transaction.

#### Change Data Capture (CDC)

Tools like Debezium tail the PostgreSQL WAL (Write-Ahead Log) and publish change events to Kafka. Downstream consumers (Redis cache updater, Elasticsearch indexer, Cassandra writer) react to these events.

CDC provides eventual consistency with minimal application coupling, but adds infrastructure complexity and requires handling out-of-order or duplicate events in consumers.

---

## 8. Spring Batch & Large-Scale ETL

### 8.1 Spring Batch Core Model

Spring Batch provides a structured model for building reliable, restartable batch pipelines:

```
┌──────────────────────────────────────────────────────────────┐
│                         JOB                                  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    STEP                              │   │
│  │                                                      │   │
│  │  ┌────────────┐   ┌─────────────┐   ┌────────────┐  │   │
│  │  │   ItemReader│──▶│ItemProcessor│──▶│ItemWriter  │  │   │
│  │  │            │   │             │   │            │  │   │
│  │  │ chunk-size │   │ transforms  │   │ commits DB │  │   │
│  │  └────────────┘   └─────────────┘   └────────────┘  │   │
│  │                                                      │   │
│  │  Commit Interval = chunk size                       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  JobRepository: metadata, restartability, execution history  │
└──────────────────────────────────────────────────────────────┘
```

**Transaction boundaries in Spring Batch:**
- Each **chunk** is wrapped in its own transaction
- If a chunk fails, only that chunk's transaction is rolled back
- The Job and Step metadata is committed to `JobRepository` independently
- A failed job can be **restarted** from the last successfully committed chunk

### 8.2 Reader Selection

| Reader | When to Use | Limitations |
|---|---|---|
| `JdbcCursorItemReader` | Large datasets; server-side cursor; steady memory | Holds DB connection open; long transactions |
| `JdbcPagingItemReader` | When cursor is not suitable; stable sort required | Requires a stable sort key; may miss rows if data changes during read |
| `FlatFileItemReader` | CSV, delimited files | Requires schema stability |
| `JpaPagingItemReader` | JPA entity reads with Spring Data | Load full entities; slower than JDBC for bulk |

**`JdbcCursorItemReader` with certain drivers:** MySQL with server-side cursors requires `fetchSize = Integer.MIN_VALUE` and `autoCommit = false`. Without these settings, it loads the entire result set into memory, defeating the purpose.

**`JdbcPagingItemReader` and stable sort:** Paging is only correct when the sort key is stable across the duration of the job. If rows are inserted or updated in the sorted column while the job runs, pages will shift and rows may be skipped or double-processed. Always use an immutable column (e.g., `id`, `created_at`) as the sort key.

### 8.3 Designing for Idempotency and Restartability

A batch job **will** fail at some point — network timeout, DB connection drop, out-of-memory, application restart. The job must be safe to restart.

**Idempotent writer pattern:**

```java
@Bean
public ItemWriter<Transaction> idempotentTransactionWriter(DataSource dataSource) {
    JdbcBatchItemWriter<Transaction> writer = new JdbcBatchItemWriter<>();
    writer.setDataSource(dataSource);
    writer.setSql(
        "INSERT INTO processed_transactions (id, merchant_id, amount, status) " +
        "VALUES (:id, :merchantId, :amount, :status) " +
        "ON CONFLICT (id) DO UPDATE SET status = EXCLUDED.status"  // PostgreSQL UPSERT
    );
    writer.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>());
    return writer;
}
```

**Note:** `ON CONFLICT ... DO UPDATE` is PostgreSQL-specific. For portability, use `MERGE` (ANSI SQL) or check for existence before insert.

### 8.4 Partitioning for Parallel Execution

For very large datasets, partition the data and process each partition in parallel:

```java
@Bean
public Step partitionedSettlementStep(StepBuilderFactory steps,
                                       PartitionHandler partitionHandler) {
    return steps.get("settlementStep.master")
        .partitioner("settlementStep.worker", new MerchantPartitioner(merchantRepository))
        .partitionHandler(partitionHandler)
        .build();
}

// Each partition processes a range of merchant IDs
public class MerchantPartitioner implements Partitioner {
    public Map<String, ExecutionContext> partition(int gridSize) {
        List<Long> merchantIds = merchantRepository.findAllActiveIds();
        // Split merchant list into gridSize buckets
        // Each ExecutionContext holds minId/maxId for one worker
    }
}
```

**Parallelism and DB locking:** Parallel partitions reading and writing the same rows will deadlock. Ensure partitions are disjoint — each worker owns an exclusive range of keys.

### 8.5 Chunk Sizing and Memory Management

Chunk size is the number of items read, processed, and written in a single transaction. Choosing it correctly is a balance:

| Chunk Size | Characteristic |
|---|---|
| Too small (1–10) | High transaction overhead; many DB round trips |
| Too large (10,000+) | Large in-memory buffer; long-held locks; bigger rollback cost on failure |
| Practical range | 50–500 for most OLTP-adjacent operations; 1000–5000 for pure INSERT pipelines |

**Memory management within a step:** If your Processor or Writer loads additional data per item (e.g., merchant enrichment lookups), that memory accumulates within the chunk window. Use `@StepScope` components and clear caches between chunks using `ChunkListener.afterChunk()`.

### 8.6 Monitoring and Operational Safeguards

Production batch jobs need observability hooks:

```java
@Bean
public Step settleStep(StepBuilderFactory steps) {
    return steps.get("settleStep")
        .<RawTransaction, ProcessedTransaction>chunk(200)
        .reader(transactionReader())
        .processor(settlementProcessor())
        .writer(settlementWriter())
        .listener(new SettlementChunkListener()) // metrics, alerting
        .faultTolerant()
            .skip(MerchantNotFoundException.class).skipLimit(100)  // tolerate bad data
            .retry(TransientDataAccessException.class).retryLimit(3)
        .build();
}
```

**Skip policy vs retry policy:**
- **Skip:** The item is logged and skipped; processing continues. Use for bad-data errors (validation failures, missing lookups)
- **Retry:** The entire chunk is retried. Use for transient errors (DB connection drops, lock timeouts)

**Separate DataSource for batch:** Long-running batch jobs should use a dedicated connection pool and schema (OLTP vs reporting/ETL separation). This prevents batch workloads from saturating the OLTP connection pool.

---

## 9. Reporting Databases, Window Functions & Summary Tables

### 9.1 OLTP vs. OLAP — Why They Conflict

OLTP and reporting workloads have fundamentally different characteristics:

| Dimension | OLTP | OLAP / Reporting |
|---|---|---|
| Query pattern | Point lookups, narrow row sets | Full scans, aggregations across millions of rows |
| Indexes | Many narrow indexes for FK/PK lookups | Fewer, broader indexes optimized for range scans |
| Latency target | Milliseconds | Seconds to minutes acceptable |
| Concurrency | Thousands of concurrent short transactions | Few concurrent but long-running queries |
| Schema design | Normalized (3NF) | Denormalized (star/snowflake, summary tables) |

Running heavy reporting queries against your OLTP database during business hours will degrade transactional performance. The canonical solution is **read replica routing**: route reporting queries to a standby replica, keeping the primary for writes.

### 9.2 Read Replica Architecture and Replica Lag

```
Primary DB (writes)
    │
    │ WAL streaming replication
    ▼
Replica DB (reads — reporting, dashboards, exports)
```

**Replica lag** is the delay between a commit on the primary and its availability on the replica. Lag is typically milliseconds in steady state, but spikes during heavy write periods or when the replica is under load.

**Implications of replica lag:**

- After writing a payment, a query to the replica may not see it immediately
- Settlement queries that need a consistent snapshot (e.g., total for today) must account for potential lag
- For real-time balance checks, always read from the primary

**Monitoring lag:**

```sql
-- On the replica
SELECT EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) AS lag_seconds;
```

Lag spikes should trigger alerts — they indicate either replication bottleneck or replica under load.

### 9.3 Window Functions — SQL-Level Analytics

Window functions perform calculations across a set of rows that are related to the current row, without collapsing rows like GROUP BY does.

```sql
-- Running total per merchant
SELECT
    merchant_id,
    created_at,
    amount,
    SUM(amount) OVER (
        PARTITION BY merchant_id
        ORDER BY created_at
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM payments
WHERE status = 'SETTLED';

-- Rank merchants by volume this month
SELECT
    merchant_id,
    total_volume,
    RANK() OVER (ORDER BY total_volume DESC) AS volume_rank
FROM merchant_daily_summary
WHERE summary_date = CURRENT_DATE;
```

**Performance of window functions:**

- Window functions run after WHERE and GROUP BY but before LIMIT
- The `OVER` clause defines the window — each distinct window is a separate sort + scan
- For large tables, ensure the `ORDER BY` inside `OVER` aligns with an available index to avoid an explicit sort step
- Queries with multiple window functions over different partitions/orderings each require their own sort pass

**`ORDER BY` and index alignment:** If your window function orders by `created_at`, and you have an index on `(merchant_id, created_at)`, the planner may use the index to skip the sort entirely. Verify with `EXPLAIN ANALYZE`.

### 9.4 Summary Tables and Materialized Views

For dashboards and reports that aggregate millions of rows, pre-computing summary tables is the standard production approach.

```sql
-- Summary table updated incrementally by ETL
CREATE TABLE merchant_daily_summary (
    merchant_id   BIGINT NOT NULL,
    summary_date  DATE NOT NULL,
    currency      VARCHAR(3) NOT NULL,
    total_amount  NUMERIC(18, 4),
    tx_count      INTEGER,
    avg_amount    NUMERIC(18, 4),
    PRIMARY KEY (merchant_id, summary_date, currency)
);

-- Idempotent upsert from ETL job
INSERT INTO merchant_daily_summary (merchant_id, summary_date, currency, total_amount, tx_count)
SELECT
    merchant_id,
    DATE(created_at) AS summary_date,
    currency,
    SUM(amount),
    COUNT(*)
FROM payments
WHERE status = 'SETTLED'
  AND DATE(created_at) = :target_date
GROUP BY merchant_id, DATE(created_at), currency
ON CONFLICT (merchant_id, summary_date, currency)
DO UPDATE SET
    total_amount = EXCLUDED.total_amount,
    tx_count = EXCLUDED.tx_count,
    avg_amount = EXCLUDED.total_amount / EXCLUDED.tx_count;
```

**Multi-currency aggregation:** Never sum amounts across currencies in a single aggregate without first converting to a base currency. Store currency alongside every amount in summary tables, and aggregate per currency, then convert at the application layer using a reliable exchange rate snapshot.

### 9.5 ETL Idempotency and Late-Arriving Data

ETL pipelines feeding summary tables must be idempotent — running the same ETL job twice for the same date must produce the same result, not double the counts.

**Late-arriving events:** A settlement that occurred on Day 1 may not be processed and written to the DB until Day 2 (e.g., due to batch delays or backpressure). A naive ETL that only processes `WHERE created_at = yesterday` will miss these. Strategies:

- Process a rolling window (e.g., last 3 days) on each run, using upsert
- Maintain an `etl_processed_at` watermark and reprocess any rows updated since the last run
- Design the summary table to allow correction (not append-only for mutable facts)

**Repository query decomposition:** Very large repository query methods that combine filtering, joining, aggregating, and sorting should be decomposed. Each query should do one thing clearly. Use `@Query` with explicit SQL for complex aggregations rather than derived method names.

---

## 10. Data Security: PII Masking, Tokenization & Audit

### 10.1 PII in a Payment System — What Must Be Protected

In a payment system, PII includes (at minimum): full card numbers (PAN), CVV, cardholder name, bank account numbers, national IDs (Aadhaar, SSN), email, phone, and IP address. The key principle: **PII should not exist in plaintext in your operational database longer than necessary**.

**Three primary protection strategies:**

| Strategy | What It Does | Use When |
|---|---|---|
| **Masking** | Replaces part of the data with a fixed character for display | Showing partial card number in UI (`**** **** **** 4321`) |
| **Tokenization** | Replaces sensitive data with a non-reversible token | Storing card references without storing the actual PAN |
| **Encryption** | Encrypts the data; original can be recovered with the key | Storing data that must be retrieved in original form |

These are complements, not alternatives. A well-designed system uses all three at appropriate layers.

### 10.2 Data Masking — Correctness Requirements

Masking is applied at the display layer. It must be correct — incorrect masking can leak more data than intended or produce invalid output.

**Regex pitfalls:**

```java
// ✗ Broken — group $3 does not exist in this pattern (only 2 groups)
String maskedAadhaar = aadhaar.replaceAll("(\\d{4})(\\d{4})(\\d{4})", "****-****-$3");

// ✓ Correct — keep only the last 4 digits
String maskedAadhaar = aadhaar.replaceAll("(\\d{8})(\\d{4})", "****-****-$2");

// ✗ Overly broad email regex that breaks on valid edge cases
String maskedEmail = email.replaceAll("(.)(.*)(@.*)", "$1****$3");
// Breaks on single-character local parts (e.g., "a@example.com" → "a****@example.com" — fine)
// Breaks on empty local parts (invalid email, but should not throw)

// ✓ Robust email masking
public static String maskEmail(String email) {
    if (email == null || !email.contains("@")) return "***";
    int atIndex = email.indexOf('@');
    String local = email.substring(0, atIndex);
    String domain = email.substring(atIndex);
    if (local.length() <= 2) return local.charAt(0) + "***" + domain;
    return local.charAt(0) + "*".repeat(local.length() - 2) + local.charAt(local.length() - 1) + domain;
}
```

**Masking converter in JPA — edge cases:** A JPA `AttributeConverter` that masks on read will mask the data in all read contexts, including when the application legitimately needs the original value. Converters that encrypt/decrypt are appropriate for at-rest encryption; converters that mask are a code smell — masking should happen at the serialization/display layer.

### 10.3 Tokenization Architecture

Tokenization replaces a sensitive value with a surrogate token. The mapping is stored in a secure token vault. The token is safe to store in your operational DB; the PAN is stored only in the vault.

```
┌──────────────┐   PAN   ┌──────────────────┐   token   ┌────────────────┐
│  Payment     │────────▶│   Token Vault     │──────────▶│  Payment DB    │
│  Gateway     │         │  (secure store)   │           │  (token only)  │
└──────────────┘         └──────────────────┘           └────────────────┘

On retrieval:
┌────────────────┐  token  ┌──────────────────┐  PAN   ┌───────────────┐
│  Payment DB    │────────▶│   Token Vault     │───────▶│  Application  │
└────────────────┘         └──────────────────┘        └───────────────┘
```

**Token vault outages:** The token vault is a critical dependency. If the vault is unavailable, any operation requiring PAN retrieval (e.g., card-on-file charging) will fail. Design for this with:

- Circuit breaker around vault calls
- Graceful degradation (queue the operation for retry rather than failing immediately)
- SLA monitoring with alerts on vault latency (P95, P99)

**KMS/Vault latency impact:** Encryption/decryption calls to a KMS or Vault add latency to every operation that touches encrypted fields. In high-throughput systems, cache decrypted values locally in memory for the duration of a request, but never persist them or log them.

### 10.4 Encryption at Rest and in Transit

**At rest:** Use column-level encryption for sensitive fields via AES-256 or a managed KMS (AWS KMS, GCP Cloud KMS, HashiCorp Vault):

```java
@Convert(converter = EncryptedStringConverter.class)
@Column(name = "account_number")
private String accountNumber;

@Component
public class EncryptedStringConverter implements AttributeConverter<String, String> {
    @Autowired
    private EncryptionService encryptionService;

    public String convertToDatabaseColumn(String attribute) {
        return encryptionService.encrypt(attribute);
    }

    public String convertToEntityAttribute(String dbData) {
        return encryptionService.decrypt(dbData);
    }
}
```

**Robust error handling around crypto failures:**

```java
public String decrypt(String ciphertext) {
    try {
        return kmsClient.decrypt(ciphertext);
    } catch (KMSException e) {
        // Never swallow silently — alert and fail fast
        metricRegistry.counter("kms.decrypt.failure").increment();
        throw new DataAccessException("Decryption failed — KMS unavailable", e);
    }
}
```

### 10.5 Audit Logging — What to Capture and What Not to

Audit logs record who did what to which data, and when. In financial systems, audit is both a compliance requirement and an operational tool.

**What must be audited:**
- Authentication events (login, logout, MFA failures)
- Authorization decisions (access granted/denied for sensitive operations)
- Write operations on financial entities (payment create, status change, refund, reversal)
- Data access to PII fields (who viewed a card number, when)
- Administrative operations (configuration changes, user role modifications)

**What to avoid auditing:**
- High-frequency read operations (health checks, list queries) — they create noise and storage overhead
- Sensitive field values in log payloads — audit events should log the fact of access, not reproduce the sensitive value

**Audit event structure:**

```java
@Entity
@Table(name = "audit_events")
public class AuditEvent {
    @Id @GeneratedValue
    private Long id;

    private String principalId;   // who
    private String action;         // what operation
    private String resourceType;   // type of entity
    private String resourceId;     // which entity
    private Instant occurredAt;    // when
    private String outcome;        // SUCCESS / FAILURE
    private String ipAddress;      // from where

    // ✗ Never store: the actual PAN, balance, or decrypted value
}
```

**PII leaks in exception paths:** Exception messages and stack traces can inadvertently include PII (e.g., a validation error that echoes the input value, or a query that includes the raw card number). Sanitize exception messages before logging.

### 10.6 Observability for Security-Sensitive Operations

Security instrumentation is essential for detecting anomalies:

```java
@Aspect
@Component
public class SensitiveOperationAspect {

    private final MeterRegistry registry;

    @Around("@annotation(SensitiveOperation)")
    public Object monitor(ProceedingJoinPoint pjp) throws Throwable {
        Timer.Sample sample = Timer.start(registry);
        try {
            Object result = pjp.proceed();
            registry.counter("sensitive_ops", "operation", pjp.getSignature().getName(),
                             "outcome", "success").increment();
            return result;
        } catch (Exception e) {
            registry.counter("sensitive_ops", "operation", pjp.getSignature().getName(),
                             "outcome", "failure").increment();
            throw e;
        } finally {
            sample.stop(registry.timer("sensitive_ops.latency"));
        }
    }
}
```

**Interpreting P95/P99 spikes on sensitive operations:** A P95 latency spike on a tokenization or decryption endpoint often points to KMS/Vault latency, not application code. Correlate with KMS latency metrics and connection pool saturation before investigating application logic.

**Over-auditing risks:** Auditing every read access to PII can itself create a compliance issue if the audit log accumulates sensitive values or if the audit system is not adequately protected. Define a clear audit retention policy and ensure audit logs are stored in an access-controlled, tamper-evident store.

---

## 11. Cross-Cutting Production Patterns

### 11.1 Idempotency — The Foundation of Reliable Systems

Idempotency means an operation can be applied multiple times with the same effect as applying it once. In distributed systems where retries, timeouts, and duplicate deliveries are inevitable, **every state-changing operation must be idempotent**.

**Implementing idempotency keys:**

```java
@Entity
@Table(name = "payments",
       uniqueConstraints = @UniqueConstraint(columnNames = "idempotency_key"))
public class Payment {
    @Column(name = "idempotency_key", nullable = false, unique = true)
    private String idempotencyKey; // caller-supplied; UUID is common
}

@Transactional
public PaymentResult createPayment(CreatePaymentRequest req) {
    return paymentRepository.findByIdempotencyKey(req.getIdempotencyKey())
        .map(existing -> PaymentResult.of(existing)) // return cached result
        .orElseGet(() -> {
            Payment payment = createAndSave(req);
            return PaymentResult.of(payment);
        });
}
```

The unique constraint ensures that even under concurrent retries, only one payment record is created. A duplicate-key exception from the DB is caught and handled by returning the existing result.

### 11.2 Error Handling and Exception Hierarchy

Production code must distinguish between recoverable and non-recoverable errors:

| Error Class | Examples | Correct Response |
|---|---|---|
| **Transient** | DB connection timeout, lock contention, network blip | Retry with exponential backoff |
| **Business constraint** | Insufficient balance, duplicate idempotency key | Return domain error, no retry |
| **Validation** | Missing required field, invalid card format | Reject immediately, no retry |
| **System fault** | OOM, corrupt data, KMS unavailable | Alert, circuit break, fail fast |

**Mapping DB errors to HTTP responses:**

```java
@ExceptionHandler(DataIntegrityViolationException.class)
public ResponseEntity<ErrorResponse> handleConstraintViolation(DataIntegrityViolationException e) {
    if (isDuplicateKeyException(e)) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse("DUPLICATE_REQUEST", "Idempotency key already used"));
    }
    return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
        .body(new ErrorResponse("CONSTRAINT_VIOLATION", "Data integrity error"));
}

@ExceptionHandler(OptimisticLockingFailureException.class)
public ResponseEntity<ErrorResponse> handleOptimisticLock(OptimisticLockingFailureException e) {
    return ResponseEntity.status(HttpStatus.CONFLICT)
        .body(new ErrorResponse("CONCURRENT_MODIFICATION", "Please retry the operation"));
}
```

### 11.3 Pagination — Never Return Unbounded Result Sets

Every API endpoint that returns a list must support and enforce pagination. Unbounded queries against large tables are a denial-of-service risk and a performance antipattern.

```java
// Offset-based pagination — simple but degrades at large offsets
@GetMapping("/payments")
public Page<PaymentSummary> listPayments(
        @RequestParam Long merchantId,
        Pageable pageable) { // Spring resolves ?page=0&size=20&sort=createdAt,desc
    return paymentRepository.findSummariesByMerchant(merchantId, pageable);
}

// Cursor-based pagination — scales better for large datasets
@GetMapping("/payments")
public CursorPage<PaymentSummary> listPayments(
        @RequestParam Long merchantId,
        @RequestParam(required = false) Long afterId) {
    return paymentRepository.findAfterIdByMerchant(merchantId, afterId, PAGE_SIZE);
}
```

**Offset pagination degrades at large pages:** `LIMIT 20 OFFSET 10000` requires the DB to read and discard 10,000 rows before returning your 20. For large datasets or high-page-number access, cursor-based pagination (keyset pagination) is significantly more efficient.

### 11.4 Connection Pool Sizing and Monitoring

JDBC connection pools are a critical resource. Undersizing causes request queuing and timeout; oversizing causes DB resource exhaustion.

**HikariCP configuration (Spring Boot default):**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # typically (2 * CPU_cores) + disk spindle count
      minimum-idle: 5
      connection-timeout: 3000     # fail fast if no connection available in 3s
      idle-timeout: 600000         # release idle connections after 10 min
      max-lifetime: 1800000        # recycle connections after 30 min (before DB reaps them)
      validation-timeout: 1000
```

**Pool saturation monitoring:**

```java
// Expose HikariCP metrics to Micrometer
@Bean
public MeterRegistryCustomizer<MeterRegistry> hikariMetrics(DataSource dataSource) {
    return registry -> ((HikariDataSource) dataSource).setMetricRegistry(
        registry // exposes hikaricp.connections.active, .pending, .idle, etc.
    );
}
```

Alert on `hikaricp.connections.pending > 0` for sustained periods — this indicates the pool is exhausted and requests are queuing.

### 11.5 Distributed Tracing for Data Operations

In a microservices system, a single user-facing request may touch multiple services and databases. Distributed tracing connects all these operations into a single trace.

```java
// Propagate trace context through async operations
@Async
public void processPaymentAsync(Long paymentId, Span parentSpan) {
    try (Scope scope = tracer.scopeManager().activate(parentSpan)) {
        Span childSpan = tracer.buildSpan("processPayment")
            .asChildOf(parentSpan)
            .start();
        try {
            paymentService.process(paymentId);
        } finally {
            childSpan.finish();
        }
    }
}
```

**Key data operations to instrument:**
- All DB queries (auto-instrumented by Spring Boot Actuator + Micrometer with JDBC observability)
- Cache hits/misses (Redis operations)
- External service calls (KMS, token vault, fraud engine)
- Batch step transitions and chunk completions

### 11.6 Designing for Failure — Checklist

Before deploying any data-intensive feature to production, verify:

- [ ] All state-changing operations are idempotent
- [ ] Every `@Transactional` method is tested for rollback behavior
- [ ] Optimistic lock failures and deadlocks are retried automatically
- [ ] Lock timeouts are configured on all pessimistic lock queries
- [ ] No unbounded queries or paginated endpoints without a maximum page size
- [ ] Bulk operations use `flush()` and `clear()` to prevent PC memory growth
- [ ] JDBC streaming connections are closed in `try-with-resources`
- [ ] Summary table ETL is idempotent (upsert, not insert)
- [ ] PII is never logged in plaintext
- [ ] Exception messages are sanitized before external propagation
- [ ] Connection pool metrics are exposed and alerting is configured
- [ ] Replica lag monitoring is in place for reporting DB reads

---

## 12. Advanced Reading & References

### 12.1 JPA and Hibernate

| Resource | Type | Link |
|---|---|---|
| **Java Persistence with Hibernate** — Bauer, King, Gregory | 📘 Book | [Manning](https://www.manning.com/books/java-persistence-with-hibernate-second-edition) |
| **High-Performance Java Persistence** — Mihalcea | 📘 Book | [leanpub.com/high-performance-java-persistence](https://leanpub.com/high-performance-java-persistence) |
| **Hibernate Official Documentation** | 📄 Docs | [hibernate.org/orm/documentation](https://hibernate.org/orm/documentation/) |
| **Spring Data JPA Reference** | 📄 Docs | [docs.spring.io/spring-data/jpa/docs/current/reference/html](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/) |
| **Vlad Mihalcea Blog** | 📝 Blog | [vladmihalcea.com](https://vladmihalcea.com/blog/) |

### 12.2 PostgreSQL Internals and Performance

| Resource | Type | Link |
|---|---|---|
| **PostgreSQL: Up and Running** — Obe, Hsu | 📘 Book | [O'Reilly](https://www.oreilly.com/library/view/postgresql-up-and/9781492057994/) |
| **The Internals of PostgreSQL** — Suzuki | 📄 Free Book | [interdb.jp/pg](https://www.interdb.jp/pg/) |
| **Use the Index, Luke** — Winand | 📄 Free Book | [use-the-index-luke.com](https://use-the-index-luke.com/) |
| **PostgreSQL EXPLAIN docs** | 📄 Docs | [postgresql.org/docs/current/using-explain.html](https://www.postgresql.org/docs/current/using-explain.html) |
| **explain.depesz.com** — EXPLAIN visualizer | 🛠 Tool | [explain.depesz.com](https://explain.depesz.com/) |
| **pev2** — EXPLAIN plan explorer | 🛠 Tool | [github.com/dalibo/pev2](https://github.com/dalibo/pev2) |

### 12.3 Concurrency and Distributed Systems

| Resource | Type | Link |
|---|---|---|
| **Designing Data-Intensive Applications** — Kleppmann | 📘 Book | [O'Reilly](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/) |
| **Java Concurrency in Practice** — Goetz et al. | 📘 Book | [Addison-Wesley](https://jcip.net/) |
| **Spring Retry** | 📄 Docs | [github.com/spring-projects/spring-retry](https://github.com/spring-projects/spring-retry) |
| **PostgreSQL Lock Monitoring** | 📄 Docs | [postgresql.org/docs/current/monitoring-locks.html](https://www.postgresql.org/docs/current/monitoring-locks.html) |
| **Database Internals** — Petrov | 📘 Book | [O'Reilly](https://www.oreilly.com/library/view/database-internals/9781492040330/) |

### 12.4 Spring Batch and ETL

| Resource | Type | Link |
|---|---|---|
| **Spring Batch Reference Documentation** | 📄 Docs | [docs.spring.io/spring-batch/docs/current/reference/html](https://docs.spring.io/spring-batch/docs/current/reference/html/) |
| **The Definitive Guide to Spring Batch** — Minella | 📘 Book | [Apress](https://link.springer.com/book/10.1007/978-1-4842-3724-3) |
| **Spring Batch GitHub** | 🛠 Tool | [github.com/spring-projects/spring-batch](https://github.com/spring-projects/spring-batch) |

### 12.5 NoSQL and Polyglot Persistence

| Resource | Type | Link |
|---|---|---|
| **Redis in Action** — Carlson | 📘 Book | [Manning](https://www.manning.com/books/redis-in-action) |
| **Cassandra: The Definitive Guide** — Carpenter, Hewitt | 📘 Book | [O'Reilly](https://www.oreilly.com/library/view/cassandra-the-definitive/9781098115159/) |
| **MongoDB: The Definitive Guide** — Chodorow | 📘 Book | [O'Reilly](https://www.oreilly.com/library/view/mongodb-the-definitive/9781491954454/) |
| **Debezium (CDC)** | 📄 Docs | [debezium.io/documentation](https://debezium.io/documentation/) |
| **Redis Cluster spec** | 📄 Docs | [redis.io/docs/manual/scaling](https://redis.io/docs/manual/scaling/) |

### 12.6 Data Security and Compliance

| Resource | Type | Link |
|---|---|---|
| **PCI DSS v4.0 Requirements** | 📄 Standard | [pcisecuritystandards.org](https://www.pcisecuritystandards.org/document_library/) |
| **OWASP Top 10** | 📄 Guide | [owasp.org/Top10](https://owasp.org/www-project-top-ten/) |
| **Vault by HashiCorp** | 📄 Docs | [developer.hashicorp.com/vault/docs](https://developer.hashicorp.com/vault/docs) |
| **Spring Security Reference** | 📄 Docs | [docs.spring.io/spring-security/reference](https://docs.spring.io/spring-security/reference/) |
| **AWS KMS Developer Guide** | 📄 Docs | [docs.aws.amazon.com/kms](https://docs.aws.amazon.com/kms/latest/developerguide/) |

### 12.7 Cross-Cutting / General Architecture

| Resource | Type | Link |
|---|---|---|
| **Site Reliability Engineering** — Google | 📘 Book | [sre.google/sre-book](https://sre.google/sre-book/table-of-contents/) |
| **Release It!** — Nygard | 📘 Book | [Pragmatic Bookshelf](https://pragprog.com/titles/mnee2/release-it-second-edition/) |
| **Patterns of Enterprise Application Architecture** — Fowler | 📘 Book | [martinfowler.com](https://martinfowler.com/books/eaa.html) |
| **microservices.io** — Richardson | 📝 Site | [microservices.io](https://microservices.io/patterns/) |

---

## Quick Reference: When to Use What

```
┌─────────────────────┬─────────────────────────┬────────────────────────────────────────────────┐
│ Problem             │ Tool / Pattern           │ When / What                                    │
├─────────────────────┼─────────────────────────┼────────────────────────────────────────────────┤
│ Entity mutations    │ JPA @Transactional       │ All write operations needing lifecycle tracking │
│ tracking            │ + Persistence Context   │ Dirty checking, cascade, version control        │
├─────────────────────┼─────────────────────────┼────────────────────────────────────────────────┤
│ N+1 queries         │ JOIN FETCH / @BatchSize  │ Collection associations in list queries         │
│                     │ / EntityGraph            │ Use @BatchSize for to-one on large lists        │
├─────────────────────┼─────────────────────────┼────────────────────────────────────────────────┤
│ Bulk inserts        │ JDBC batching + SEQUENCE │ High-volume writes; settlement record creation  │
│                     │ + flush/clear cycles     │ Avoid IDENTITY strategy                         │
├─────────────────────┼─────────────────────────┼────────────────────────────────────────────────┤
│ Concurrent writes   │ @Version (optimistic)    │ Low contention; single entity update            │
│                     │ SELECT FOR UPDATE        │ High contention; explicit coordination needed   │
│                     │ (pessimistic)            │ Always set lock timeout                         │
├─────────────────────┼─────────────────────────┼────────────────────────────────────────────────┤
│ Query performance   │ EXPLAIN ANALYZE          │ Diagnose slow queries                           │
│                     │ Composite / partial /    │ Design indexes to match query shape             │
│                     │ covering indexes         │ Monitor via pg_stat_user_indexes                │
├─────────────────────┼─────────────────────────┼────────────────────────────────────────────────┤
│ Large result sets   │ JDBC streaming /         │ ETL, exports, reporting pipelines               │
│                     │ Spring Batch             │ Never load millions of rows into memory         │
├─────────────────────┼─────────────────────────┼────────────────────────────────────────────────┤
│ Reporting workload  │ Read replica + summary   │ Dashboards, aggregations, settlement reports    │
│                     │ tables + window funcs    │ Separate from OLTP; idempotent ETL              │
├─────────────────────┼─────────────────────────┼────────────────────────────────────────────────┤
│ PII protection      │ Masking + Tokenization   │ Display: mask. Storage: tokenize. Transit:      │
│                     │ + Encryption + Audit     │ TLS. At rest: column-level encryption           │
├─────────────────────┼─────────────────────────┼────────────────────────────────────────────────┤
│ Retry / duplicate   │ Idempotency keys         │ All payment creation and state-change ops       │
│ prevention          │ + unique constraint      │ Return cached result on duplicate key           │
├─────────────────────┼─────────────────────────┼────────────────────────────────────────────────┤
│ Multi-store writes  │ Outbox pattern / CDC     │ DB + Kafka / Redis / Elasticsearch sync         │
│                     │ (Debezium)               │ Guarantees at-least-once delivery               │
└─────────────────────┴─────────────────────────┴────────────────────────────────────────────────┘
```

---
