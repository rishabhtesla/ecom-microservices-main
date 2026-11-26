# Complete Java Microservices Interview Guide

This is the single, end-to-end revision document for the e-commerce repository. It explains the architecture concepts, the Spring internals, the distributed-systems patterns, the interview questions, and the code locations to open.

## How to read this guide

Every section uses three labels:

- **[Repository]** means the pattern exists in the checked-in code. Open the linked file and trace it.
- **[Demonstration]** means the pattern exists under `additional/` as a focused example.
- **[Blueprint]** means the pattern is explained with new example code because this repository does not implement it yet.

The primary modules currently use Spring Boot 3.4.x, Spring Cloud 2024.0.x, and Java 21. Some syllabus items refer to Spring Framework 7/Spring Boot 4. Those items are explained as version-aware concepts; they should not be described as already used by this checkout until the Maven POMs are upgraded and tested.

## Repository in one minute

```text
client
  -> Gateway :8080
      -> Product :8081  -> PostgreSQL productdb
      -> User    :8082  -> MongoDB ecom_user
      -> Order   :8083  -> PostgreSQL orderdb
                             -> Product/User over discovery
                             -> Kafka order.exchange
                                  -> Notification :8084

Config Server :8888
Eureka       :8761
Keycloak     :8443
Zipkin       :9411
Prometheus   :9090
Grafana      :3000
```

The fastest code tour is:

1. [`gateway/GatewayConfig.java`](../../gateway/src/main/java/com/ecommerce/gateway/GatewayConfig.java)
2. [`gateway/security/SecurityConfig.java`](../../gateway/src/main/java/com/ecommerce/gateway/security/SecurityConfig.java)
3. [`product/controllers/ProductController.java`](../../product/src/main/java/com/ecommerce/product/controllers/ProductController.java)
4. [`order/clients`](../../order/src/main/java/com/ecommerce/order/clients)
5. [`order/services/OrderService.java`](../../order/src/main/java/com/ecommerce/order/services/OrderService.java)
6. [`notification/OrderEventConsumer.java`](../../notification/src/main/java/com/ecommerce/notification/OrderEventConsumer.java)
7. [`deploy/docker/docker-compose.yml`](../../deploy/docker/docker-compose.yml)

---

# Part I - Distributed-system architecture

## 1. Monolith, modular monolith, microservices, and distributed monolith

### Monolith

A monolith is one deployable application. Its code may be well designed or badly designed; “monolith” describes deployment shape, not code quality.

Advantages:

- one process and one deployment;
- simple local transactions;
- easy refactoring across modules;
- low network and operational overhead.

Costs:

- the whole application often scales and deploys together;
- a shared database can create hidden coupling;
- a failure or memory leak can affect every capability;
- team ownership becomes difficult as the codebase grows.

### Modular monolith

A modular monolith has strict internal modules and one deployment. Product, User, and Order can have separate packages, interfaces, tests, and ownership while still calling each other in-process.

It is often the best starting point. Establish boundaries before adding a network boundary.

### Microservices

Each service is independently deployable and owns a cohesive business capability. The boundary introduces network latency, versioning, partial failure, authentication, observability, and eventual consistency.

### Distributed monolith anti-pattern

A distributed monolith has many processes but still behaves like one application:

- every service must be deployed together;
- services share one database schema;
- every request requires a long synchronous call chain;
- shared libraries force synchronized releases;
- one central team owns all changes;
- a failure in one service blocks all business flows.

The number of repositories is not proof of microservice independence. Ask whether each service can be changed, deployed, scaled, and recovered independently.

## 2. Domain-Driven Design

### Ubiquitous language

Use the business vocabulary consistently in requirements, code, APIs, events, and tests. If the business says “reserve stock,” do not call the same action “decrement inventory” in one service and “hold quantity” in another without defining the distinction.

### Bounded context

A bounded context is a boundary inside which a model and its terms have one meaning. This repository has:

```text
Product context:
  name, price, stockQuantity, active

User context:
  profile, address, Keycloak identity

Order context:
  cart, order, order item, total, status

Notification context:
  reactions to order facts and delivery state
```

The Order Service stores `productId` and `userId`; it does not own Product or User entities.

### Aggregate

An aggregate is a consistency boundary with one aggregate root. `Order` is the root and `OrderItem` children are saved with it:

```java
@OneToMany(mappedBy = "order",
        cascade = CascadeType.ALL,
        orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();
```

Do not make every database table a distributed aggregate. Keep the aggregate small enough to update atomically.

### Domain event

A domain event records a fact that happened: `OrderCreated`, `PaymentAuthorized`, or `StockReserved`. It is not a command such as `SendEmailNow`. Events should have an ID, version, timestamp, aggregate ID, and stable payload.

## 3. Decomposition strategies

### By business capability

Find an independently valuable capability such as catalog, identity, checkout, or notification. This is the primary decomposition in the repository.

### By subdomain

DDD distinguishes:

- **Core subdomain:** differentiates the business; invest custom design here.
- **Supporting subdomain:** important but not the main differentiator.
- **Generic subdomain:** use a product or standard solution, such as identity or email.

Do not split a generic subdomain into a custom service unless there is a real requirement.

### Strangler Fig pattern

Route one capability from the old monolith to a new service, observe it, then remove the old path:

```text
old monolith
  -> facade/router
       /product -> Product Service
       /legacy  -> monolith
```

The monolithic baseline is [`additional/ecom-application`](../../additional/ecom-application).

### Anti-Corruption Layer

An ACL translates a legacy or foreign model so it does not leak into the new domain:

```java
Product legacy = legacyClient.getProduct(id);
CatalogItem item = new CatalogItem(
        legacy.code(),
        Money.of(legacy.amount(), legacy.currency()));
```

The mapper is a boundary, not a cosmetic DTO conversion. It protects the new model from old naming and semantics.

### Sidecar and Ambassador

A sidecar runs beside the application and provides cross-cutting behavior such as telemetry, mTLS, or proxying. An ambassador sidecar represents an external dependency, for example a local proxy that handles a legacy database or payment connection.

The repository does not ship sidecars. Istio/Envoy examples belong to the deployment platform, not the business service.

## 4. Granularity and team design

### Conway's Law

Systems tend to reflect the communication structure of the organization. If Product, Identity, and Checkout are owned by separate teams with clear APIs, the architecture is more likely to have those boundaries.

### Two-pizza team rule

A team should be small enough to communicate directly, often described as roughly 5-10 people. It is a heuristic, not a capacity formula. The real goal is clear ownership of code, data, on-call, and product outcomes.

### Cohesion versus coupling

High cohesion means things that change together live together. Low coupling means one service does not need to know another service's implementation.

Warning signs:

- an Order change requires editing Product database tables;
- services share internal entity classes;
- a request needs five synchronous downstream calls;
- one team owns all service releases;
- every event consumer depends on an exact producer class.

### Interview Q&A

**Q: Why not split every table into a service?**
**A:** A service boundary should follow business ownership and transaction boundaries. A table-by-table split creates chatty calls and distributed transactions without creating useful autonomy.

**Q: Is a modular monolith a failure?**
**A:** No. It is often the safest way to discover domain boundaries while keeping local transactions and simple operations. Extract only when independent scaling, ownership, or deployment justifies the cost.

---

# Part II - Communication, APIs, and contracts

## 5. REST and the Richardson Maturity Model

The model is a vocabulary for how much of HTTP a service uses:

0. **The Swamp of POX:** one endpoint, often RPC-like payloads.
1. **Resources:** separate URIs such as `/products/42`.
2. **HTTP verbs and status codes:** `GET`, `POST`, `PUT`, `DELETE`, `201`, `404`, `409`.
3. **Hypermedia controls:** responses guide the next valid actions with links.

Most practical Spring APIs are level 2. Level 3 is useful when clients need discoverable workflows, but it is not mandatory for a well-designed API.

The Product API demonstrates resource paths and status codes:

```text
POST   /api/products       -> 201
GET    /api/products/{id}  -> 200 or 404
PUT    /api/products/{id}  -> 200 or 404
DELETE /api/products/{id}  -> 204 or 404
```

## 6. gRPC

gRPC uses Protocol Buffers over HTTP/2. It supports unary calls, server streaming, client streaming, and bidirectional streaming.

**[Blueprint]**

```proto
syntax = "proto3";

package catalog.v1;

service CatalogService {
  rpc GetProduct(GetProductRequest) returns (Product);
  rpc WatchStock(WatchStockRequest) returns (stream StockUpdate);
}

message GetProductRequest {
  string product_id = 1;
}

message Product {
  string product_id = 1;
  string name = 2;
  int64 price_minor = 3;
  string currency = 4;
}
```

Use gRPC for controlled service-to-service contracts, high-throughput internal calls, or streaming. Use REST for broad browser/client compatibility and simple public APIs. gRPC still needs deadlines, authentication, retries, load balancing, and observability.

## 7. GraphQL Federation

GraphQL exposes a client-selected graph. Federation lets multiple subgraphs own parts of the graph:

```text
Gateway graph:
  Product subgraph -> catalog
  User subgraph    -> profiles
  Order subgraph   -> checkout
```

It can solve over-fetching and aggregation for client applications, but it introduces schema governance, query-cost control, N+1 resolver risks, and authorization complexity. The repository uses REST; no GraphQL implementation is present.

## 8. Brokers and event streaming

### Message broker versus event stream

RabbitMQ is commonly used for brokered work queues and routing. Kafka is a durable partitioned log that supports replay and independent consumer offsets. Both can implement pub/sub, but the operational and semantic model differs.

### Point-to-point queue

One logical message is handled by one consumer in a competing-consumer group. Use it for work distribution.

### Pub/sub

Every subscription receives a copy. In Kafka, separate consumer groups provide independent subscriptions.

### Event notification versus event-carried state transfer

```text
Event notification:
  OrderCreated { orderId: 42 }
  consumer calls Order API for details

Event-carried state transfer:
  OrderCreated { orderId, userId, total, items, occurredAt }
  consumer can act without a synchronous callback
```

The repository's `OrderCreatedEvent` carries order details, which reduces consumer coupling to a live Order API. It still needs versioning and idempotency.

## 9. Gateway, BFF, aggregation, and protocol translation

The Gateway routes the primary API:

```java
.route("product-service", r -> r
        .path("/api/products/**")
        .uri("lb://PRODUCT-SERVICE"))
```

A **Backend for Frontend** is a client-specific gateway, such as one API optimized for mobile and another for a web application. A generic gateway should not grow client-specific business logic accidentally.

**Request aggregation** combines multiple downstream responses into one response. It reduces browser round trips but increases gateway latency and failure coupling.

**Protocol translation** converts external REST/JSON to internal gRPC, events, or a legacy protocol. Keep the translation at the boundary.

**Edge caching** can cache public catalog reads. Do not cache personalized or authorization-sensitive responses without a correct key and invalidation strategy.

## 10. API evolution and contract management

### Versioning options

| Style | Example | Trade-off |
|---|---|---|
| URI | `/api/v2/products` | Visible and easy to route; duplicates paths |
| Header | `X-API-Version: 2` | Keeps URI stable; less discoverable |
| Media type | `Accept: application/vnd.shop.product-v2+json` | Explicit representation; tooling is harder |

Version only when semantics or compatibility require it. Adding an optional response field is usually not a breaking change.

### Expand and contract

```text
expand:
  deploy readers that understand old and new fields
  write both representations if needed
contract:
  migrate all consumers
  stop producing the old field
  remove old code/schema
```

Never rename a field and deploy producer and all consumers as one distributed transaction.

### Consumer-driven contracts

**[Blueprint]**

```java
@Pact(consumer = "order-service", provider = "product-service")
public RequestResponsePact productLookup(PactDslWithProvider builder) {
    return builder
        .given("product 42 exists")
        .uponReceiving("a product lookup")
            .path("/api/products/42")
            .method("GET")
        .willRespondWith()
            .status(200)
            .body(newJsonBody(body -> {
                body.stringType("name");
                body.numberType("stockQuantity");
            }).build())
        .toPact();
}
```

Pact and Spring Cloud Contract verify the interaction without starting the entire platform. Provider verification must run in the provider build.

### Schema registry

**[Blueprint]** An Avro event schema can be registered and checked for compatibility:

```json
{
  "type": "record",
  "name": "OrderCreated",
  "namespace": "com.ecommerce.events.v1",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "orderId", "type": "long"},
    {"name": "totalAmount", "type": "double"}
  ]
}
```

Compatibility policies:

- **Backward:** new consumers can read old data.
- **Forward:** old consumers can read new data.
- **Full:** both directions remain compatible.

Adding an optional field with a default is safer than renaming/removing a required field. JSON Schema, Avro, and Protobuf each have different compatibility rules; enforce them in CI.

### Interview Q&A

**Q: Why is an API contract more than a Java interface?**
**A:** The contract includes wire format, status codes, headers, timeouts, authorization, error shape, and compatibility behavior. A shared Java interface cannot verify all of those.

**Q: When would you choose gRPC over REST?**
**A:** For controlled internal clients that benefit from typed compact messages or streaming. I would keep REST at the public edge unless clients and infrastructure already support gRPC well.

---

# Part III - Discovery, load balancing, and service mesh

## 11. Discovery models

### Client-side discovery

The client queries a registry and selects an instance:

```text
Order -> Eureka registry -> product instances -> selected Product instance
```

The repository uses this model with `lb://PRODUCT-SERVICE` and a load-balanced `RestClient.Builder`.

### Server-side discovery

The client calls a stable endpoint; a load balancer or platform routes it:

```text
Order -> product-service.default.svc -> Kubernetes Service -> Product pod
```

Kubernetes DNS/CoreDNS is the common server-side discovery model.

### Health and heartbeats

Registries need registration, renewal, eviction, and health behavior. A process being alive does not mean it is ready to handle traffic. Use health checks that distinguish liveness from readiness.

## 12. Load-balancing algorithms

| Algorithm | Behavior | Use |
|---|---|---|
| Round robin | Rotate instances equally | Similar instance capacity |
| Least connections | Choose the least busy instance | Long-lived/variable requests |
| Weighted | Send proportionally by weight | Unequal capacity or canary |
| Consistent hashing | Map a key to a stable subset | Cache locality, partition ownership |
| Sticky sessions | Keep a client on one instance | Avoid when possible; hurts failover/scaling |

The repository uses Spring Cloud LoadBalancer but does not configure every algorithm explicitly.

## 13. Service mesh

A service mesh moves east-west networking concerns into proxies:

```text
Order application -> Envoy sidecar -> Product Envoy sidecar -> Product application
```

- **Data plane:** sidecar proxies that handle traffic.
- **Control plane:** distributes policies, certificates, and routes.
- **Istio/Linkerd/Envoy:** common ecosystem choices.

Sidecar injection is convenient but increases CPU, latency, debugging complexity, and upgrade surface. An ambassador proxy is a sidecar dedicated to one external integration.

Mesh features:

- transparent mTLS;
- retries/timeouts;
- traffic splitting;
- canary/A-B routing;
- telemetry;
- authorization policy.

Do not enable mesh-level retries blindly if the application client already retries; retry amplification is still possible.

### Interview Q&A

**Q: Eureka or Kubernetes DNS?**
**A:** Eureka provides application-level client-side discovery and can work outside Kubernetes. Kubernetes Service DNS is platform-level server-side discovery. I choose one primary model and still configure timeouts and health behavior.

**Q: What is the cost of a service mesh?**
**A:** More operational components, proxy resource usage, policy debugging, and another failure surface. It is valuable when uniform mTLS, routing, and telemetry across many services justify that cost.

---

# Part IV - Distributed data and consistency

## 14. Database patterns

### Database per service

Product owns Product tables; Order owns Order tables; User owns Mongo documents. This is the preferred boundary.

### Shared database anti-pattern

It can be a transitional step during extraction, but direct cross-service table access creates schema, transaction, deployment, and security coupling.

### Polyglot persistence

Choose by access pattern:

- relational databases for transactions and constraints;
- MongoDB for document-shaped profiles;
- Cassandra for high-write distributed queries with known partition keys;
- Redis for low-latency cache/state;
- Elasticsearch/OpenSearch for search and analytics-oriented indexing;
- in-memory grids such as Coherence for distributed near-cache/use cases.

The repository uses PostgreSQL and MongoDB. Cassandra, Elasticsearch, Coherence, and Redis are syllabus extensions rather than active primary-service implementations.

## 15. Consistency models

| Model | Meaning |
|---|---|
| Strong | Reads observe the latest committed write |
| Eventual | Replicas converge if writes stop |
| Causal | Cause-and-effect ordering is preserved |
| ACID | Atomicity, consistency, isolation, durability for a transaction |
| BASE | Basically available, soft state, eventual consistency |

### CAP

During a network partition, a distributed system chooses consistency or availability for the affected operation. CAP is about partition behavior, not a general ranking of database products.

### PACELC

PACELC adds normal operation:

```text
If Partition:
  choose Availability or Consistency
Else:
  choose Latency or Consistency
```

This explains why a system might trade a slightly stale local read for lower latency even when no partition exists.

### Quorum and tunable consistency

For a replicated store with `N` replicas:

```text
R + W > N  -> read and write quorums overlap
```

This is a useful reasoning rule, not a guarantee of application-level correctness. Conflicts, clocks, repair, and failure modes still matter.

## 16. Distributed transactions and sagas

### Two-phase commit

2PC has a coordinator:

```text
prepare all participants
commit all participants
```

It can block during coordinator failure, holds resources, has poor latency/scaling characteristics, and requires compatible participants. It is rarely a good default across independent microservices.

### Saga choreography

Services react to events:

```text
OrderCreated -> PaymentAuthorized -> StockReserved -> OrderConfirmed
                  PaymentFailed -> OrderCancelled
```

It has low central coordination but can become difficult to understand as participants grow.

### Saga orchestration

A coordinator sends commands and records state:

```java
class CheckoutSaga {
    void start(OrderId orderId) {
        commandBus.send(new ReserveStock(orderId));
    }

    void onStockReserved(StockReserved event) {
        commandBus.send(new AuthorizePayment(event.orderId()));
    }

    void onPaymentFailed(PaymentFailed event) {
        commandBus.send(new ReleaseStock(event.orderId()));
    }
}
```

The coordinator must be durable and idempotent. A saga does not roll back time; compensating transactions create new business actions.

### Semantic locks

A business state such as `PENDING_PAYMENT` can prevent conflicting work while the saga is in progress. This is a semantic lock because it is enforced by domain state, not a database lock held across services.

## 17. Reliable messaging and data synchronization

### Dual-write problem

Writing a database row and publishing a message separately can leave one successful and the other failed.

### Transactional outbox

**[Blueprint]**

```sql
CREATE TABLE outbox_event (
    event_id UUID PRIMARY KEY,
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(100) NOT NULL,
    event_type VARCHAR(200) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    published_at TIMESTAMP WITH TIME ZONE
);
```

The Order transaction writes the order and outbox row together. A polling publisher claims unpublished rows, sends them to Kafka, and marks them published. Use row locking or a claim token when multiple relays run.

### CDC/Debezium

Change Data Capture reads database transaction logs and publishes changes without application dual writes:

```text
PostgreSQL WAL -> Debezium connector -> Kafka -> consumers
```

CDC is excellent for integration and replication, but raw row changes are not always good business events. Define whether consumers need `OrderCreated` or a table-level change.

### Idempotent consumer and inbox

```sql
CREATE TABLE consumed_event (
    consumer_name VARCHAR(100),
    event_id UUID,
    consumed_at TIMESTAMP WITH TIME ZONE NOT NULL,
    PRIMARY KEY (consumer_name, event_id)
);
```

Insert the event ID and apply the business effect in one local transaction. If the insert conflicts, acknowledge the duplicate without repeating the effect.

## 18. Query and read optimization

### CQRS

Command and Query Responsibility Segregation separates write models from read models:

```text
commands -> Order aggregate -> events
events   -> denormalized order read model
queries  -> read model
```

CQRS is useful when read and write workloads/models differ significantly. It adds synchronization and rebuild complexity.

### Event sourcing

Store the sequence of domain events as the source of truth and rebuild state by replay:

```text
OrderCreated
ItemAdded
PaymentAuthorized
OrderShipped
```

It provides audit/replay benefits but requires event versioning, snapshots, correction strategy, and careful storage growth. The repository uses state-based JPA, not event sourcing.

### API composition and materialized views

An API composer calls multiple services and assembles a response. A materialized view is updated asynchronously from events and serves fast reads. Use a materialized view when repeated runtime joins are too slow or fragile.

### Interview Q&A

**Q: Outbox or CDC?**
**A:** Outbox makes business events explicit and is controlled by application semantics. CDC is powerful for transaction-log synchronization and existing systems. They can coexist, but do not expose raw table changes as domain facts without a translation boundary.

**Q: Why is eventual consistency acceptable for notification?**
**A:** Notification is a consequence, not the checkout commit. The order can return immediately while notification retries independently; the user-facing contract must state the delay and failure behavior.

---

# Part V - Resilience, fault tolerance, and caching

## 19. Circuit breakers, retries, timeouts, and fallbacks

### Circuit states

```text
CLOSED    calls flow; failures are measured
OPEN      calls fail fast
HALF_OPEN limited probes test recovery
```

The repository configures Resilience4j in Order:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      productService:
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
  retry:
    instances:
      retryBreaker:
        maxAttempts: 5
        waitDuration: 5s
```

### Retry with exponential backoff and jitter

```text
delay = min(cap, base * 2^attempt) + random_jitter
```

Jitter prevents thousands of clients from retrying at the same instant. A retry budget limits how much additional traffic retries may create.

Retry only transient errors, only safe operations, and only within an overall deadline. The current Cart Service retry is a learning example and should be made idempotent before production use.

### Timeouts

Separate connection timeout, pool-acquisition timeout, read/response timeout, and end-to-end deadline. A timeout is not a failure response by itself; the caller must decide whether to retry, fallback, queue, or return an error.

### Fallbacks

Return stale catalog data only if it is explicitly marked stale. Never return `200` with an empty list when the real answer is “catalog unavailable.” Keep payment and inventory failures conservative.

## 20. Isolation and load protection

### Bulkhead

- **Thread-pool isolation:** separate executor for a dependency; isolates blocking work.
- **Semaphore isolation:** cap concurrent calls without creating a separate pool; useful for short work.

### Rate-limiting algorithms

| Algorithm | Meaning |
|---|---|
| Token bucket | Tokens refill; bursts are allowed up to bucket capacity |
| Leaky bucket | Work leaves at a steady rate; excess waits or drops |
| Fixed window | Count resets at fixed boundaries; boundary bursts are possible |
| Sliding window | Counts recent intervals more smoothly |

The Gateway uses a Redis token-bucket limiter. Redis is not currently defined in the Compose file, so this is a documented local-platform gap.

### Load shedding and backpressure

Reject work early with `429` or `503` when capacity is exhausted. Backpressure lets a producer slow down when a consumer cannot keep up. In reactive streams, demand is signaled through the publisher/subscriber protocol; in messaging, queue depth and consumer lag are operational backpressure signals.

## 21. Caching

### Patterns

| Pattern | Read/write path | Risk |
|---|---|---|
| Cache-aside | App reads cache, loads DB on miss, writes cache | Stampede and stale data |
| Read-through | Cache loader reads DB | Cache owns loader integration |
| Write-through | App writes cache and cache writes DB | Higher write latency |
| Write-behind | Cache writes later | Data loss/durability risk |

**Blueprint, cache-aside:**

```java
public ProductResponse getProduct(String id) {
    ProductResponse cached = redis.get("product:" + id);
    if (cached != null) {
        return cached;
    }
    ProductResponse loaded = productClient.getProductDetails(id);
    redis.set("product:" + id, loaded, Duration.ofMinutes(5));
    return loaded;
}
```

### Distributed cache

Redis is a common distributed cache. Coherence is an in-memory data-grid option. Treat cache data as disposable unless the design explicitly makes it durable.

### Cache hazards

- **Stampede/thundering herd:** many requests miss together; use single-flight locking, early refresh, or jittered TTL.
- **Penetration:** repeated requests for missing keys; cache negative results with a short TTL and validate IDs.
- **Avalanche:** many keys expire together; use randomized TTLs and staggered refresh.
- **Stale data:** publish invalidation events or use versioned keys.

Choose TTL, LRU/LFU eviction, maximum size, and invalidation rules from business correctness, not a default five minutes.

### Interview Q&A

**Q: Should checkout use a cache for price?**
**A:** It may use a cache for a read optimization, but checkout must validate authoritative price/stock and persist a price snapshot. A stale cache must not authorize payment for the wrong amount.

**Q: What is the danger of retrying a cache miss loader?**
**A:** It can multiply database traffic during an outage. Add request coalescing, a bounded loader pool, and a fallback policy.

---

# Part VI - Observability and debugging

## 22. Structured logging

Log JSON with stable fields:

```json
{
  "timestamp": "2026-01-01T10:00:00Z",
  "level": "INFO",
  "service": "order-service",
  "event": "order_created",
  "orderId": "42",
  "traceId": "4f3c...",
  "durationMs": 84
}
```

Use correlation IDs for a business request and trace IDs for distributed causal context. Never log passwords, tokens, card data, or sensitive profile fields.

The repository writes rolling service logs and collects them with Grafana Alloy/Loki. See [`docs/10-observability`](../10-observability/README.md).

## 23. Metrics

### RED method

- **Rate:** requests or events per second.
- **Errors:** failed requests/events.
- **Duration:** latency distribution, especially p95/p99.

### USE method

- **Utilization:** how busy a resource is.
- **Saturation:** queued or blocked work.
- **Errors:** resource failures.

Product, User, and Order expose Micrometer/Prometheus endpoints. Avoid high-cardinality labels such as raw order IDs.

## 24. Tracing and OpenTelemetry

A trace is a tree of spans. W3C Trace Context propagates `traceparent` and related headers:

```text
Gateway span
  -> Order HTTP span
      -> Product HTTP span
      -> User HTTP span
      -> PostgreSQL span
      -> Kafka publish span
          -> Notification consume span
```

Micrometer Tracing and Zipkin are present in the repository. OpenTelemetry is a vendor-neutral alternative or foundation. Use an OTel collector to route telemetry to Zipkin, Jaeger, or a managed backend.

## 25. Health and diagnostics

- **Liveness:** should the process be restarted?
- **Readiness:** should it receive traffic?
- **Startup:** is it still initializing?

Add custom `HealthIndicator` implementations only for dependencies whose state should remove the instance from traffic. Do not make liveness depend on every remote service.

Synthetic monitoring sends real probes such as “create/read product” or “checkout with a test account.” Chaos engineering intentionally injects faults:

```text
kill Product instance
add 500 ms latency
drop Kafka connection
fill connection pool
expire a certificate
```

Start in a controlled environment with a steady-state hypothesis, abort criteria, and rollback plan.

### Interview Q&A

**Q: How do you debug one failed checkout?**
**A:** Start from the trace/correlation ID, inspect gateway authentication and routing, follow Order's dependency spans, verify the database transaction, inspect Kafka publish/consumer lag, then check Notification/DLQ. Metrics show whether the issue is isolated or systemic.

---

# Part VII - Security and zero trust

## 26. OAuth2, OIDC, and JWT

- **OAuth2:** authorization framework and token acquisition.
- **OIDC:** identity layer on OAuth2.
- **JWT:** signed token representation.
- **Access token:** short-lived token sent to APIs.
- **Refresh token:** longer-lived credential used to obtain new access tokens; protect and rotate it.
- **JWKS:** public keys used by resource servers to verify JWT signatures.

The Gateway is configured as a JWT resource server:

```java
.oauth2ResourceServer(oauth2 -> oauth2
        .jwt(jwt -> jwt.jwtAuthenticationConverter(
                grantedAuthoritiesExtractor())))
```

Validate issuer, signature, expiration, not-before, audience, and required scopes/roles. Rotate signing keys through JWKS with overlap so old tokens can be verified during rollout.

## 27. RBAC, ABAC, and policy engines

- **RBAC:** role such as `ROLE_ORDER_USER`.
- **ABAC:** decision uses attributes such as tenant, region, resource owner, amount, and time.
- **OPA:** external policy engine using policy-as-code, commonly Rego.

RBAC is simple and predictable. ABAC handles multi-tenant/resource ownership rules but requires trusted attributes and careful policy testing.

**Blueprint, OPA decision input:**

```json
{
  "subject": {"id": "u-7", "roles": ["ORDER_USER"], "tenant": "t-1"},
  "action": "order.read",
  "resource": {"owner": "u-7", "tenant": "t-1"}
}
```

## 28. Zero-trust service-to-service trust

Zero trust means every request is authenticated and authorized based on identity and policy; network location alone is not trust.

Options:

- OAuth2 client credentials;
- mTLS with workload certificates;
- service-mesh identity;
- SPIFFE/SPIRE workload identities;
- token exchange when a service needs a delegated user context;
- token relay when a downstream service must act on behalf of the user.

The current Order flow trusts `X-User-ID`, which is a teaching simplification and not a zero-trust identity boundary.

## 29. Secrets and API protection

Use Vault, AWS Secrets Manager, Azure Key Vault, or another secret manager for database passwords, client secrets, signing keys, and provider credentials. Rotate without rebuilding images.

API protections:

- validate shape, range, size, and content type;
- reject unknown/unsafe input where appropriate;
- apply object-level authorization;
- rate limit authentication and expensive endpoints;
- enforce request body limits;
- protect against SSRF, injection, mass assignment, and insecure direct object references;
- use WAF/DDoS controls at the edge;
- return safe errors without stack traces.

The repository contains development credentials and a client secret in demo configuration. Treat those as compromised and never reuse them.

### Interview Q&A

**Q: Why is a valid JWT not enough?**
**A:** Authentication only establishes a caller identity. Authorization must check the action, resource, tenant, and current business state.

**Q: mTLS or OAuth2?**
**A:** mTLS proves workload identity at the transport layer. OAuth2 provides token claims, scopes, and delegated/user context. Many platforms use both.

---

# Part VIII - Containers, Kubernetes, and DevOps

## 30. Docker packaging

Use multi-stage builds to keep compilers out of the runtime image:

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /workspace
COPY . .
RUN ./mvnw -q -DskipTests package

FROM eclipse-temurin:21-jre
USER 10001
WORKDIR /app
COPY --from=build /workspace/target/*.jar app.jar
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Jib and Buildpacks can produce layered images without maintaining this Dockerfile. Pin image versions, generate an SBOM, scan, sign, and do not bake secrets into layers.

## 31. Kubernetes primitives

- **Pod:** smallest schedulable unit.
- **Deployment:** manages stateless replicas and rollouts.
- **Service:** stable DNS/IP for pods.
- **Ingress/Gateway API:** external HTTP/TLS routing.
- **ConfigMap:** non-secret config.
- **Secret:** credential material; use external secret integration.
- **HPA:** scales replicas from resource/custom metrics.
- **KEDA:** scales from event sources such as Kafka lag.

The repository has a complete blueprint in [`docs/13-kubernetes`](../13-kubernetes/README.md) but no tracked production manifests.

## 32. Release strategies

| Strategy | Behavior | Risk/control |
|---|---|---|
| Rolling | Replace instances gradually | Default; needs readiness |
| Blue-green | Switch traffic between two environments | Fast rollback; double capacity |
| Canary | Send a small percentage to new version | Gradual evidence; needs metrics |
| Shadow/dark launch | Copy traffic without affecting response | Test real load; side effects must be isolated |
| Feature flag | Deploy code disabled, enable selectively | Fast business rollback; flag lifecycle needed |

Use LaunchDarkly, Unleash, or a controlled internal flag service. A feature flag is not a permanent replacement for versioning.

## 33. Zero-downtime operations

Graceful shutdown:

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s
```

Kubernetes adds a `preStop` hook and connection draining. Readiness must fail before termination so new traffic stops while in-flight requests finish.

For database migrations, use expand-and-contract:

```text
1. add nullable/new column
2. deploy code that writes old and new forms
3. backfill
4. deploy readers using new form
5. stop old writes
6. remove old column later
```

Use Flyway or Liquibase instead of `hibernate.ddl-auto=update` in production.

## 34. Testing pyramid

```text
unit -> slice -> integration/Testcontainers -> contract -> small E2E
```

Use KEDA for event backlog scaling, not CPU alone, when Kafka consumer lag is the real bottleneck.

---

# Part IX - Spring Core and Boot internals

## 35. IoC container and dependency injection

The Spring container creates objects, resolves dependencies, applies post-processors, and manages lifecycle. `BeanFactory` is the minimal container; `ApplicationContext` adds events, messages, resource loading, and enterprise integration.

Prefer constructor injection:

```java
@Service
public class OrderService {
    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

Avoid field injection because dependencies are hidden and tests need reflection. Setter injection is suitable for optional/reconfigurable dependencies. `@Primary` selects a default bean; `@Qualifier` selects a named bean; `@Lazy` delays creation and can reduce startup cost but may move failures to first use.

Circular dependencies usually indicate unclear ownership. Spring Boot disables circular references by default because resolving them can produce partially initialized objects and hide design problems. Refactor, introduce an interface/event boundary, or use `ObjectProvider` only when lazy lookup is genuinely required.

## 36. Bean lifecycle and extension points

Simplified lifecycle:

```text
instantiate
  -> populate properties
  -> aware callbacks
  -> BeanPostProcessor.beforeInitialization
  -> @PostConstruct / InitializingBean
  -> BeanPostProcessor.afterInitialization
  -> ready for use
  -> @PreDestroy / DisposableBean
```

Use `@PostConstruct` for initialization that needs injected dependencies. Use `@PreDestroy` for cleanup. `BeanFactoryPostProcessor` changes bean definitions before beans are instantiated; `BeanPostProcessor` wraps or modifies bean instances.

**[Blueprint]**

```java
@Component
class AuditBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(
            Object bean, String name) {
        return bean;
    }
}
```

Spring 7/Boot 4-era `BeanRegistrar` and functional registration are useful for programmatic bean registration. Treat them as version-specific APIs and verify imports against the exact release train.

## 37. Scopes and proxies

Default scope is singleton. Other scopes include prototype, request, session, and application. A request-scoped bean injected into a singleton needs a scoped proxy or provider:

```java
@Bean
@RequestScope
public RequestContext requestContext() {
    return new RequestContext();
}
```

Inject a prototype into a singleton with `ObjectProvider<T>`:

```java
private final ObjectProvider<CommandContext> contexts;

CommandContext newContext() {
    return contexts.getObject();
}
```

`@Lookup` is another option. Scoped proxies delay resolution until the correct scope exists.

## 38. Auto-configuration and starters

`@SpringBootApplication` combines:

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

Boot discovers auto-configurations through metadata such as `AutoConfiguration.imports`. Conditions make configuration selective:

```java
@AutoConfiguration
@ConditionalOnClass(RedisClient.class)
@ConditionalOnMissingBean(CacheClient.class)
@ConditionalOnProperty(
        name = "app.cache.enabled",
        havingValue = "true")
class CacheAutoConfiguration {
}
```

A custom starter normally has:

```text
my-feature-spring-boot-autoconfigure
my-feature-spring-boot-starter
```

The starter depends on the auto-config module and exposes a simple dependency for consumers. Keep auto-configuration conditional and override-friendly.

## 39. Configuration and profiles

Prefer typed configuration:

```java
@ConfigurationProperties("payment")
@Validated
public record PaymentProperties(
        @NotBlank String baseUrl,
        @Min(1) Duration timeout) {
}
```

`@Value` is convenient for one value but becomes hard to validate and test across many settings. Use `application.yml` hierarchy, profiles, Config Server, environment variables, and a secret manager deliberately. Profiles select environment behavior; they should not hide business logic.

---

# Part X - Spring Web, embedded servers, and HTTP clients

## 40. Spring MVC request lifecycle

```text
Servlet container
  -> Filter chain
  -> DispatcherServlet
  -> HandlerMapping
  -> HandlerAdapter
  -> Controller
  -> HttpMessageConverter (Jackson)
  -> response
```

`HandlerExceptionResolver` and `@RestControllerAdvice` convert failures into responses. `ViewResolver` matters for server-rendered views; REST APIs usually write the response body directly.

## 41. Filters, interceptors, and AOP

| Mechanism | Scope | Typical use |
|---|---|---|
| Servlet Filter | Before/after servlet processing | request IDs, raw HTTP concerns |
| OncePerRequestFilter | One execution per request | authentication/logging |
| HandlerInterceptor | Around MVC handler | handler-level checks |
| Spring AOP | Method proxy | transactions, retries, metrics |

Order matters. Do not put authorization in a logging filter and assume it is the only security layer.

## 42. REST validation and ProblemDetail

```java
public record ProductRequest(
        @NotBlank String name,
        @Positive BigDecimal price,
        @PositiveOrZero Integer stockQuantity) {
}

@PostMapping
public ResponseEntity<ProductResponse> create(
        @Valid @RequestBody ProductRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED)
            .body(service.create(request));
}
```

Use `@RestControllerAdvice` to produce RFC 7807 `ProblemDetail` responses with stable error codes, field errors, and a trace ID. Custom constraints should implement `ConstraintValidator`.

Boot 4 API versioning properties and APIs are version-sensitive. The design choices remain URI, header, or media-type versioning; verify exact property names for the chosen Boot version.

## 43. Embedded Tomcat and concurrency

Tomcat has connectors, acceptors, polling, and worker threads. Key settings include:

```yaml
server:
  tomcat:
    threads:
      max: 200
    accept-count: 100
    max-connections: 8192
  shutdown: graceful
```

Tune from load tests and downstream capacity. More threads do not fix a slow database and can increase memory/context switching.

Virtual threads can make blocking I/O easier to scale:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

They do not make CPU work faster or remove database/broker limits. Avoid pinning virtual threads with long synchronized/blocking native operations.

## 44. WebFlux and Netty

WebFlux uses `Mono<T>` for zero/one result and `Flux<T>` for many results. Netty uses event loops, so blocking calls must not execute on event-loop threads.

Use WebFlux for streaming, high concurrency with non-blocking dependencies, or an already reactive path. Use MVC with a bounded thread pool or virtual threads for conventional blocking JPA applications. Do not choose WebFlux merely because it sounds faster.

## 45. HTTP clients

The repository demonstrates:

- `RestClient`: synchronous fluent client;
- `WebClient`: reactive client under `additional/interservice`;
- `@HttpExchange`: declarative interface backed by `RestClient`;
- OpenFeign and RestTemplate under `additional/interservice`.

Every client needs connection pooling, connection/read/deadline timeouts, error mapping, authentication propagation, trace propagation, and bounded retries.

---

# Part XI - Spring Data, JPA, Hibernate, and transactions

## 46. `@Transactional` and proxy mechanics

Spring typically applies `@Transactional` through an AOP proxy. A call from outside the proxy enters the transaction; a self-invocation such as `this.otherTransactionalMethod()` bypasses the proxy.

```java
@Transactional
public void createOrder() {
    saveOrder();
}

// Calling this method from inside the same instance does not re-enter
// a separate proxy transaction.
@Transactional(propagation = REQUIRES_NEW)
public void saveAudit() {
}
```

Visibility/final-method limitations depend on proxy type. Put transaction boundaries on public service methods and split collaborating services when propagation truly differs.

### Propagation

| Mode | Meaning |
|---|---|
| REQUIRED | Join current or create one |
| REQUIRES_NEW | Suspend current and create independent one |
| SUPPORTS | Join if present, otherwise none |
| MANDATORY | Require an existing transaction |
| NESTED | Savepoint when supported |

### Isolation

`READ_COMMITTED`, `REPEATABLE_READ`, and `SERIALIZABLE` trade concurrency for anomaly prevention. Choose based on lost updates, dirty reads, phantom reads, and database behavior—not a universal “highest is best.”

By default, unchecked exceptions usually trigger rollback. Use `rollbackFor` for checked exceptions when that is the business requirement.

For explicit multi-step control:

```java
transactionTemplate.execute(status -> {
    Order order = repository.save(new Order());
    outboxRepository.save(eventFor(order));
    return order;
});
```

## 47. JPA entity lifecycle

```text
Transient -> persist/save -> Managed/Persistent
Managed -> detach/clear -> Detached
Managed -> remove -> Removed
```

The persistence context is the first-level cache. Hibernate dirty checking compares managed state at flush and generates SQL changes. `flush()` synchronizes SQL with the database transaction; `commit()` completes the transaction. `saveAndFlush()` forces a flush earlier but does not commit.

Second-level cache is shared across persistence contexts and must be configured with correct invalidation; do not enable it for mutable hot data without measuring consistency.

## 48. Query optimization

### N+1 query problem

Loading one order and then one query per item/customer causes N+1 round trips. Fix with:

- `JOIN FETCH`;
- `@EntityGraph`;
- `@BatchSize`;
- DTO/interface projections;
- explicit read models.

Avoid multiple collection fetch joins that create cartesian products and `MultipleBagFetchException`.

### Pagination

- `Page` includes a count query;
- `Slice` tells whether another slice exists;
- keyset/cursor pagination uses a stable indexed key and avoids deep offset scans.

Disable Open Session in View (`spring.jpa.open-in-view=false`) for explicit transaction boundaries and to avoid lazy loads during response serialization.

## 49. Locking

Optimistic locking adds a version:

```java
@Version
private long version;
```

Conflicting updates throw `OptimisticLockException`; retry or tell the client to reload. Pessimistic locking uses database locks:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Stock> findForUpdate(Long productId);
```

Keep lock order consistent and transactions short to prevent deadlocks. Use optimistic locking for most web edits; use pessimistic locking for tightly controlled reservation operations when the database supports it.

## 50. Connection pools and migrations

Hikari settings include `maximum-pool-size`, `minimum-idle`, `connection-timeout`, `max-lifetime`, and `leak-detection-threshold`. Pool size should fit database capacity and concurrent query behavior. A large pool can overload the database.

Use Flyway/Liquibase migrations and expand-and-contract deployment. The repository currently uses `ddl-auto: update` for learning.

## 51. NoSQL, Redis, and Kafka integrations

### Cassandra

Model around partition keys and known query paths. Avoid unbounded partitions and joins. Consistency is tunable per operation.

### Redis

Use `RedisTemplate` or Spring Cache for cache-aside data, rate limiting, locks with care, and short-lived state. Do not use a cache as the only source of payment truth.

### Kafka

Use `@KafkaListener`, `KafkaTemplate`, Spring Cloud Stream, `DefaultErrorHandler`, `DeadLetterPublishingRecoverer`, and manual acknowledgements when you need explicit error/retry control. The repository uses Spring Cloud Stream `StreamBridge` and functional consumers.

---

# Part XII - Spring AOP, asynchronous execution, and resilience

## 52. Spring AOP

An aspect combines a pointcut and advice:

```java
@Aspect
@Component
class TimingAspect {
    @Around("execution(* com.ecommerce..service..*(..))")
    Object time(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.nanoTime();
        try {
            return joinPoint.proceed();
        } finally {
            long elapsed = System.nanoTime() - start;
            log.debug("method={} nanos={}",
                    joinPoint.getSignature(), elapsed);
        }
    }
}
```

Advice types include `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, and `@Around`. Proxies do not intercept private methods, final methods/classes under common proxy modes, or internal `this` calls.

## 53. Async and scheduling

```java
@EnableAsync
@Configuration
class AsyncConfig {
    @Bean
    Executor applicationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(32);
        executor.setQueueCapacity(100);
        executor.initialize();
        return executor;
    }
}
```

`@Async` returns immediately and moves work to an executor. Propagate MDC/security context with a `TaskDecorator`, bound the queue, and handle exceptions. Do not use an unbounded executor for payment or notification work.

`@Scheduled` supports fixed rate, fixed delay, and cron. In a multi-instance deployment, every instance may run the task. Use ShedLock or a distributed scheduler for one-at-a-time jobs.

## 54. Resilience annotations

The repository uses:

```java
@Retry(name = "retryBreaker", fallbackMethod = "addToCartFallBack")
```

Resilience4j also offers `@CircuitBreaker`, `@Bulkhead`, `@RateLimiter`, and `@TimeLimiter`. Spring Framework 7/Boot 4-era native retry/concurrency annotations are version-specific; do not mix them with Resilience4j accidentally. Define which layer owns retries and metrics.

---

# Part XIII - Spring Security internals

## 55. Security filter chain

```text
request
  -> DelegatingFilterProxy
  -> FilterChainProxy
  -> selected SecurityFilterChain
  -> authentication filters
  -> authorization
  -> controller
```

`SecurityFilterChain` is configured as a bean. `AuthenticationManager` delegates to `AuthenticationProvider`; providers can use `UserDetailsService`, an external OIDC/JWT decoder, or another identity system.

## 56. JWT resource server

The Gateway's `oauth2ResourceServer().jwt()` validates bearer tokens. A resource server should:

1. parse the bearer token;
2. locate the issuer/JWKS keys;
3. verify the signature;
4. validate time/audience/issuer claims;
5. convert scopes/roles into authorities;
6. apply request/method authorization.

Method-level authorization:

```java
@EnableMethodSecurity
@Configuration
class MethodSecurityConfig {
}

@PreAuthorize("hasAuthority('SCOPE_orders:read')")
public OrderResponse findOrder(long id) {
    return service.find(id);
}
```

Use `@PostAuthorize` carefully; it runs after method execution and must not expose sensitive results before authorization is evaluated.

## 57. CORS, CSRF, and sessions

- Enable CORS for explicit trusted origins/methods/headers.
- CSRF protects cookie-authenticated browser sessions. A stateless bearer-token API may disable it when no browser cookie authentication is involved.
- Use `SessionCreationPolicy.STATELESS` for JWT resource servers.
- Never combine `allowCredentials=true` with wildcard origins.

---

# Part XIV - Actuator and production operations

## 58. Actuator endpoints

Important endpoints:

```text
/actuator/health
/actuator/metrics
/actuator/prometheus
/actuator/info
/actuator/threaddump
/actuator/heapdump
```

Secure management endpoints, expose only what operators need, and separate liveness/readiness groups. Heap dumps and environment/config endpoints are sensitive.

## 59. Micrometer Observation and OpenTelemetry

Micrometer Observation connects an operation to metrics, traces, and context:

```java
Observation.createNotStarted("catalog.lookup", registry)
        .lowCardinalityKeyValue("operation", "get")
        .observe(() -> client.getProductDetails(id));
```

Use `@Observed`, `@Timed`, and `@Counted` for deliberate business/technical observations. Keep labels low-cardinality. Use W3C Trace Context and MDC correlation IDs.

## 60. Startup and build optimization

Options include:

- lazy initialization (`spring.main.lazy-initialization=true`);
- Class Data Sharing;
- Spring AOT;
- GraalVM native image;
- layered JARs from Buildpacks/Jib.

Measure startup, memory, throughput, reflection compatibility, and build time. Lazy initialization can make a health check pass while moving a configuration failure to first request.

JSpecify annotations such as `@NullMarked` and `@Nullable` improve static null-safety when supported by the exact toolchain. Treat them as compile-time contracts, not runtime validation.

---

# Part XV - Testing in Spring Boot

## 61. Unit testing

Use JUnit and Mockito for fast business tests:

```java
@ExtendWith(MockitoExtension.class)
class ProductServiceTest {
    @Mock ProductRepository repository;
    @InjectMocks ProductService service;

    @Test
    void missingActiveProductIsNotReturned() {
        when(repository.findByIdAndActiveTrue(7L))
                .thenReturn(Optional.empty());
        assertThat(service.getProductById("7")).isEmpty();
    }
}
```

AssertJ provides readable assertions. Keep tests independent of Spring unless container wiring is the behavior under test.

## 62. Slice tests

| Annotation | Scope |
|---|---|
| `@WebMvcTest` | MVC controllers, MockMvc, converters, advice |
| `@DataJpaTest` | JPA mappings and repositories |
| `@JsonTest` | Jackson serialization/deserialization |
| `@SpringBootTest` | Full application context |

Boot 3.4+/Boot 4 direction replaces `@MockBean`/`@SpyBean` with `@MockitoBean`/`@MockitoSpyBean` in version-specific APIs. Verify the exact version before changing a project.

## 63. Testcontainers and WireMock

```java
@Testcontainers
class OrderIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:14");
}
```

`@ServiceConnection` can wire supported containers automatically in newer Spring Boot versions. Use Testcontainers for PostgreSQL, Kafka, Cassandra, and Redis behavior that an in-memory mock cannot prove. Use WireMock for external HTTP contracts and failure responses.

## 64. Contract and E2E tests

Contract tests verify provider/consumer compatibility. E2E tests should be few but cover the business path:

```text
user -> product -> cart -> order -> OrderCreated -> notification
```

Add failure cases: invalid JWT, missing product, dependency timeout, duplicate checkout, Kafka outage after DB commit, and duplicate event delivery.

### Interview Q&A

**Q: Why not test only with `@SpringBootTest`?**
**A:** It is slower and less precise. Unit and slice tests give faster feedback; contract and container tests catch wire/infrastructure issues; a small E2E suite validates the complete business path.

**Q: Why Testcontainers instead of H2?**
**A:** H2 can differ from PostgreSQL in SQL, indexes, locking, and transaction behavior. Use the same engine for persistence integration tests.

---

# Final interview answer framework

For any topic, answer in this order:

1. **Problem:** what failure or complexity does it solve?
2. **Mechanism:** how does it work internally?
3. **Trade-off:** what new cost or failure mode does it introduce?
4. **Repository:** where is it used here?
5. **Production:** what would you harden or measure?

Example:

> Eureka solves dynamic service location. A client registers and renews a lease, fetches instances, and load-balances a logical name such as `PRODUCT-SERVICE`. It introduces registry availability and stale-instance concerns. This repository uses it through Gateway `lb://PRODUCT-SERVICE` and Order's load-balanced `RestClient`. In Kubernetes I would normally use Service DNS, secure the registry if it remains, add readiness, and monitor discovery failures.

## High-value questions to rehearse

1. Why is a distributed monolith worse than a modular monolith?
2. How did you choose Product, User, Order, and Notification boundaries?
3. What is the difference between a domain event and a command?
4. Why use Kafka for notification but HTTP for cart validation?
5. How do you evolve a REST/event contract without synchronized deployment?
6. How do consumer groups and partitions affect Kafka scaling and ordering?
7. What problem does the outbox solve?
8. How do you make a consumer idempotent?
9. Why is 2PC usually avoided?
10. When would you choose choreography versus orchestration?
11. What happens when Product Service is slow?
12. What is retry amplification and how do you prevent it?
13. What is the difference between a circuit breaker and a timeout?
14. How do you prevent a cache stampede?
15. What is the difference between liveness and readiness?
16. How do you trace one request across HTTP and Kafka?
17. Why is a valid JWT not sufficient authorization?
18. How do mTLS and OAuth2 complement each other?
19. Why does a database transaction not cover Kafka automatically?
20. What is the difference between `flush()` and commit?
21. Why does `@Transactional` fail on self-invocation?
22. How does Spring Boot auto-configuration decide what to create?
23. What is the difference between a Filter, Interceptor, and Aspect?
24. When should you use WebFlux instead of MVC?
25. What does a Kubernetes readiness probe do?
26. How would you perform a zero-downtime schema migration?
27. What would you test with Testcontainers?
28. Which syllabus features are not implemented in this checkout?

The honest answer to the last question is part of the interview preparation: payments, Kubernetes manifests, outbox/CDC, saga orchestration, schema registry, service mesh, CI/CD, and contract-test suites are documented blueprints or future work here, not claimed as completed runtime features.
