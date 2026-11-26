# 19. Distributed Data, Consistency, and Reliable Messaging

## Database ownership

The primary services demonstrate database-per-service:

```text
Product -> PostgreSQL productdb
Order   -> PostgreSQL orderdb
User    -> MongoDB ecom_user
```

The key rule is ownership: another service may call an API or consume an event, but it does not join the owner's tables.

## ACID and BASE

ACID protects a local transaction:

- Atomicity;
- Consistency;
- Isolation;
- Durability.

BASE accepts basically available, soft, eventually convergent state. It is not “no consistency”; it is a choice to resolve consistency across time and boundaries rather than one global transaction.

## Strong, eventual, and causal consistency

```text
strong:    read immediately sees committed write
eventual:  replicas converge after propagation
causal:    if B was caused by A, observers do not see B before A
```

Notification can be eventual. Payment authorization and order state transitions need explicit business rules even if propagation is asynchronous.

## CAP and PACELC

CAP describes the partition case:

```text
partition -> choose consistency or availability
```

PACELC adds normal behavior:

```text
partition -> A or C
else      -> latency or consistency
```

Quorum reasoning:

```text
N replicas, R read quorum, W write quorum
R + W > N  -> overlapping read/write quorum
```

Application conflicts, stale caches, and repair still need design.

## 2PC versus saga

2PC holds participant resources while a coordinator prepares and commits. Coordinator failure can block participants and the protocol is expensive across independent services.

A saga uses local transactions and compensations:

```text
reserve stock
authorize payment
confirm order

if payment fails:
  release stock
  mark order PAYMENT_FAILED
```

Choreography is event-driven but can become implicit and hard to trace. Orchestration makes workflow state explicit but adds a coordinator. Use semantic locks such as `PENDING_PAYMENT` to prevent incompatible operations while a saga is in progress.

## Transactional outbox

The current Order Service calls:

```java
Order savedOrder = orderRepository.save(order);
cartService.clearCart(userId);
streamBridge.send("createOrder-out-0", event);
```

That is a dual-write gap. A production outbox:

```sql
CREATE TABLE outbox_event (
    event_id UUID PRIMARY KEY,
    aggregate_id VARCHAR(100) NOT NULL,
    event_type VARCHAR(200) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    published_at TIMESTAMPTZ
);
```

The order and outbox row commit together. A polling publisher sends rows and marks them. Duplicate publication is possible, so consumers remain idempotent.

## CDC and Debezium

```text
database WAL/binlog
  -> Debezium connector
  -> Kafka
  -> downstream read models/search/analytics
```

CDC is useful for integration with existing databases and rebuilding read models. It emits data changes, not necessarily domain facts. Use an ACL/translator if consumers need `OrderCreated` rather than `orders` row changes.

## Inbox/idempotent consumer

```sql
CREATE TABLE inbox_event (
    consumer_name VARCHAR(100),
    event_id UUID,
    received_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (consumer_name, event_id)
);
```

Insert the event ID and apply the side effect in one local transaction. A duplicate primary-key failure means the effect already happened.

## CQRS and event sourcing

CQRS:

```text
commands -> write aggregate -> events
events   -> read projection
queries  -> materialized read model
```

Event sourcing stores events as the source of truth:

```text
OrderCreated -> PaymentAuthorized -> Shipped
```

It needs event versioning, snapshots, replay tooling, correction events, and storage governance. The repository uses state-based JPA; do not claim event sourcing is implemented.

## Cassandra, Redis, search, and grids

- Cassandra: partition-key-first modeling, known query paths, bounded partitions.
- Redis: low-latency cache/rate limiter/short-lived state.
- Elasticsearch/OpenSearch: search index and analytical read patterns, not the transactional source of truth.
- Coherence/in-memory grids: distributed cache/data-grid use cases with explicit invalidation and ownership.

## Interview Q&A

**Why is an outbox safer than publishing directly after `save`?**
Because the business change and the intent to publish commit in one local transaction. A relay can retry the intent after a process crash.

**Does an outbox provide exactly-once delivery?**
No. It usually provides at-least-once publication. Exactly-once business effect comes from idempotency and unique constraints.

**When is CQRS worth it?**
When read and write models, scaling, or query shapes differ substantially. It is not a default for every CRUD service because projections add lag and operational work.
