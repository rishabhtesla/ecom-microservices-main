# 26. JPA, Hibernate, Transactions, and Data Performance

## 1. Entity lifecycle

```text
Transient -> persist -> Managed
Managed   -> detach  -> Detached
Managed   -> remove  -> Removed
```

The persistence context is the first-level cache and identity map. Dirty checking compares managed state at flush time and generates SQL changes. A transaction commit normally flushes first; `flush()` sends SQL without committing the database transaction.

`save()` can persist/merge depending on entity state. `saveAndFlush()` forces a flush but does not commit. Do not use it as a substitute for transaction design.

## 2. `@Transactional` mechanics

Spring wraps the target in a proxy:

```text
caller -> transactional proxy -> begin -> target -> flush/commit
```

Typical propagation:

| Propagation | Meaning |
|---|---|
| `REQUIRED` | Join current or create one |
| `REQUIRES_NEW` | Suspend current and create independent one |
| `SUPPORTS` | Join if present |
| `MANDATORY` | Fail without an existing transaction |
| `NESTED` | Savepoint when supported |

Isolation controls visibility/locking, not distributed consistency. `READ_COMMITTED` is common; `SERIALIZABLE` can reduce anomalies at a throughput cost.

Unchecked exceptions typically roll back by default. Explicitly configure checked-exception behavior:

```java
@Transactional(rollbackFor = PaymentRejectedException.class)
public void checkout(...) { ... }
```

Self-invocation bypasses the proxy:

```java
public void outer() {
    inner(); // no proxy interception
}

@Transactional
public void inner() { ... }
```

Move the method to another bean or use programmatic `TransactionTemplate` when a boundary must be explicit.

## 3. N+1 and query shape

```text
select orders
select items for order 1
select items for order 2
...
```

Fix with a deliberate query:

```java
@Query("""
       select distinct o from Order o
       left join fetch o.items
       where o.id = :id
       """)
Optional<Order> findWithItems(@Param("id") Long id);
```

Other options include `@EntityGraph`, batch fetching, DTO projections, and explicit read models. Joining multiple collection “bags” can cause `MultipleBagFetchException` or row multiplication; fetch one collection, use sets carefully, or query in stages.

### Pagination

- `Page` performs a count query and gives total pages;
- `Slice` checks whether another page exists;
- keyset/cursor pagination avoids expensive deep offsets.

Use stable indexed ordering for cursor pagination:

```sql
WHERE (created_at, id) < (:lastCreatedAt, :lastId)
ORDER BY created_at DESC, id DESC
LIMIT :size
```

Disable Open Session in View for service APIs where lazy access after the transaction hides query boundaries:

```yaml
spring:
  jpa:
    open-in-view: false
```

## 4. Concurrency and locking

Optimistic locking:

```java
@Version
private long version;
```

Concurrent updates fail with an optimistic-lock exception; retry or present a conflict according to business rules. Pessimistic locking:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Inventory> findForUpdateByProductId(String productId);
```

It holds database locks and can deadlock. Keep transactions short, lock rows in a consistent order, index predicates, and set lock/time limits.

## 5. Connection pools and migrations

Hikari properties are capacity controls:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 2000
      max-lifetime: 1800000
      leak-detection-threshold: 10000
```

Pool size must account for database limits and all application replicas. A larger pool can increase database contention rather than throughput.

Flyway/Liquibase migrations should be backward compatible during rolling deploys. The repository's database names and profiles are documented in [`service-catalog.md`](../reference/service-catalog.md).

## 6. Redis, Cassandra, and Kafka integration

Redis is appropriate for cache/rate limiting and short-lived coordination, not as a silent replacement for the order database. Cassandra requires query-first partition design and bounded partitions. Kafka consumers need explicit acknowledgment, error handler, retry/DLQ, and idempotency policy.

The additional examples are under [`additional/kafka`](../../additional/kafka). The production Order publication still needs an outbox as explained in [`19`](../19-distributed-data-and-consistency/README.md).

## Interview Q&A

**What is the difference between flush and commit?**
Flush synchronizes pending persistence-context changes to SQL execution; commit makes the database transaction durable/visible according to its isolation and durability rules.

**Why not solve N+1 by making every association eager?**
It moves the cost to every query, can create huge graphs/cartesian products, and makes fetch behavior implicit. Query each use case deliberately.

**Does `REQUIRES_NEW` guarantee a distributed transaction?**
No. It creates a separate local transaction and can commit even if the outer transaction later rolls back.
