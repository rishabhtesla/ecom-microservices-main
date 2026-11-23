# 3. Domain Design and Data Ownership

## Start with the business language

Microservices should follow business boundaries, not technical layers. In this application:

- Product owns catalog and stock-facing data.
- User owns profile data and the link to the identity provider.
- Order owns carts, orders, items, totals, and order status.
- Notification reacts to an order fact and owns notification delivery state when that feature is added.

This is a bounded-context view. A field with the same name can have a different meaning in another context. `userId` in Order is a reference, not permission to query the User database directly.

## Database-per-service rule

Each service owns its schema and persistence lifecycle:

```text
Product Service -> product database
Order Service   -> order database
User Service    -> ecom_user Mongo database
```

The service may expose a read model or API for another service, but another service must not connect directly to its database. This preserves independent migrations and prevents accidental coupling through joins.

The current configuration is in:

- [`product-service.yml`](../../configserver/src/main/resources/config/product-service.yml)
- [`order-service.yml`](../../configserver/src/main/resources/config/order-service.yml)
- [`user-service.yml`](../../configserver/src/main/resources/config/user-service.yml)

## Relational and document models

Product uses JPA:

```java
@Entity(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private BigDecimal price;
    private Integer stockQuantity;
    private Boolean active = true;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

Order models an aggregate with child items:

```java
@Entity(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String userId;
    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    private OrderStatus status = OrderStatus.PENDING;

    @OneToMany(mappedBy = "order",
            cascade = CascadeType.ALL,
            orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}
```

User uses Spring Data MongoDB because a profile with an embedded address can be stored naturally as a document. The database choice should follow access patterns and ownership, not a blanket “SQL is always better” or “NoSQL is always faster” rule.

## Aggregate and transaction boundary

An aggregate is the consistency boundary for a business operation. In this repository, an Order and its OrderItems are saved together. The cart-to-order operation also clears the cart and sends an event, which exposes an important distributed-systems issue:

```text
save order
clear cart
publish event
```

If the database commit succeeds but the process crashes before Kafka publish, the order exists but notification never hears about it. If the event is published and the database transaction later rolls back, consumers see a fact that is not durable.

### Production solution: transactional outbox

Add an `outbox_event` table in the Order database:

```text
Order transaction:
  1. save Order and OrderItems
  2. insert OutboxEvent(status=NEW, aggregateId=orderId)
  3. commit once

Relay:
  4. poll NEW rows
  5. publish to Kafka
  6. mark SENT, with retry and dead-letter handling
```

The relay must be idempotent and consumers must tolerate duplicate events. Exactly-once business behavior is normally achieved through idempotency, not through a magical network-wide transaction.

## Checkout consistency

The current order flow calculates the total from cart items and marks the order `CONFIRMED`. A production checkout should explicitly define:

1. whether prices are snapshotted at cart time or checkout time;
2. how stock is reserved and released;
3. what happens if payment fails;
4. whether the order is `PENDING_PAYMENT` before authorization;
5. how duplicate checkout requests are detected;
6. how an expired cart is handled.

## Payment service blueprint

The syllabus mentions payments, but this checkout does not contain a payment module. A safe extension would be:

```text
Order Service
  -> OrderCreated / PaymentRequested
Payment Service
  -> calls provider with idempotency key
  -> stores provider reference and payment status
  -> publishes PaymentAuthorized or PaymentFailed
Order Service
  -> transitions order to CONFIRMED or PAYMENT_FAILED
```

Do not store raw card numbers or CVV. Store a provider token, payment intent ID, status, amount, and audit metadata. Webhook handling must verify the provider signature and be idempotent.

## Monolith-to-microservices extraction

The monolithic example under [`additional/ecom-application`](../../additional/ecom-application) is useful because it shows the original in-process calls. Extract one vertical slice at a time:

| Step | Action | Safety check |
|---:|---|---|
| 1 | Define Product API inside the monolith | Controller/service tests |
| 2 | Introduce a product repository boundary | No caller accesses product tables |
| 3 | Run the new Product Service beside the monolith | Contract and shadow traffic |
| 4 | Move product data ownership | Migration and rollback plan |
| 5 | Switch callers to service discovery | Metrics show no old path |
| 6 | Remove duplicated product logic | Consumer contract tests remain |

The strangler pattern is safer than a rewrite: route one capability to the new service, observe it, and progressively remove the old path.

## Data patterns to know

| Pattern | Use |
|---|---|
| Shared database | Transitional only; creates schema and deployment coupling |
| Database per service | Default ownership model |
| Read replica/read model | Scale or reshape reads without changing write ownership |
| CQRS | Separate read and write models when their requirements differ significantly |
| Outbox | Atomically record a local change and a future event |
| Inbox/deduplication table | Remember consumed event IDs and make handlers idempotent |
| Saga choreography | Services react to each other's events |
| Saga orchestration | A coordinator commands steps and compensations |

## Interview questions

**Why is a cross-service database join a problem?**  
It bypasses ownership, creates a runtime dependency on another service's schema, and makes independent deployment impossible.

**How do you keep an order total correct?**  
Persist line-item price snapshots, calculate with `BigDecimal`, define rounding/currency rules, and make the checkout operation idempotent. Do not trust a client-supplied total.

**What is the difference between a saga and a distributed transaction?**  
A saga is a sequence of local transactions with events or commands and compensating actions. It accepts eventual consistency rather than holding locks across services.

**Where would you place the outbox?**  
In the same database and local transaction as the business aggregate whose change must produce an event.
