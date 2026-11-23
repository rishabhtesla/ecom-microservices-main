# 1. Microservices Foundations

## What is a microservice?

A microservice is a small, independently deployable application that owns a cohesive business capability. “Small” is not a line-count rule. The useful boundaries are:

- one business responsibility;
- one team or ownership boundary;
- one deployable process;
- one data ownership boundary;
- an explicit API or event contract.

The goal is not to maximize the number of services. The goal is to make a change in one business capability deployable without rebuilding and redeploying unrelated capabilities.

## Monolith versus microservices

| Concern | Modular monolith | Microservices |
|---|---|---|
| Deployment | One artifact | Many independently deployed artifacts |
| Runtime calls | In-process method calls | Network calls and messages |
| Data | Often one database | Database ownership per service |
| Failure | Usually local | Partial failure is normal |
| Scaling | Scale the whole application | Scale the hot capability |
| Transactions | Local ACID transaction is easy | Cross-service transaction needs a saga/outbox strategy |
| Operations | One process to observe | Discovery, configuration, tracing, and automation are required |

Microservices trade deployment and team autonomy for distributed-systems complexity. An interviewer usually wants to hear both sides.

## The boundaries in this project

The primary application uses these bounded contexts:

```text
Product context
  catalog, price, stock, active flag

User context
  profile, address, Keycloak identity reference, role

Order context
  cart, order, order item, order status

Notification context
  reaction to OrderCreatedEvent

Platform context
  gateway, discovery, configuration, observability
```

The services do not share Java entity classes or database tables. Order stores a `productId` and a `userId`; it obtains the information it needs through an API call or an event instead of joining another service's tables.

## A request across the system

For `POST /api/orders`, the logical sequence is:

1. The client calls the gateway on port `8080`.
2. The gateway authenticates the bearer token and chooses the `ORDER-SERVICE` route.
3. Order Service reads the cart from its PostgreSQL database.
4. Order Service creates an order and its items in its own transaction boundary.
5. Order Service publishes an `OrderCreatedEvent` to Kafka.
6. Notification Service consumes the event independently.

The synchronous part gives the caller an immediate result. The asynchronous part keeps notification work out of the checkout request.

## Synchronous and asynchronous communication

Use synchronous HTTP when the caller needs an answer to continue:

```text
Order Service --GET product--> Product Service
Order Service --GET user-----> User Service
```

Use an event when the producer should not know every consumer or when work can complete later:

```text
Order Service --OrderCreatedEvent--> Kafka --> Notification Service
                                              --> Payment Service (future)
                                              --> Analytics (future)
```

An event is a statement of fact, not a remote method call. `OrderCreatedEvent` should contain the data consumers need and remain compatible as the producer evolves.

## Distributed-systems vocabulary

### Partial failure

One service, broker, database, or network hop can fail while the rest of the system is healthy. Every network call therefore needs a timeout, a bounded retry policy, and a defined fallback or error response.

### Eventual consistency

After an order is committed, notification state may become visible milliseconds or seconds later. The system is correct if consumers eventually process the event and can retry safely. Do not promise read-after-write consistency across services unless you explicitly design for it.

### CAP trade-off

During a network partition, a distributed system chooses between:

- **Consistency:** reject or delay operations rather than return potentially stale data.
- **Availability:** continue responding, possibly with stale data.

CAP does not mean “pick two features.” It describes what happens when a partition exists. Normal operation can provide both consistency and availability.

### Idempotency

An operation is idempotent when repeating the same request or event does not create an additional business effect. `PUT /products/{id}` is normally idempotent; `POST /orders` is not unless it accepts an idempotency key and stores the result.

### Coupling

There are several forms:

- **Temporal coupling:** both services must be available at the same time.
- **Schema coupling:** one service must know another's exact JSON shape.
- **Deployment coupling:** both services must be released together.
- **Operational coupling:** one service's resource exhaustion affects another.

Events reduce temporal coupling but introduce schema/versioning and replay concerns.

## Where to open the code

| Concept | File |
|---|---|
| Gateway boundary | [`gateway/src/main/java/com/ecommerce/gateway/GatewayConfig.java`](../../gateway/src/main/java/com/ecommerce/gateway/GatewayConfig.java) |
| Product bounded context | [`product/src/main/java/com/ecommerce/product/controllers/ProductController.java`](../../product/src/main/java/com/ecommerce/product/controllers/ProductController.java) |
| Order aggregate and event | [`order/src/main/java/com/ecommerce/order/services/OrderService.java`](../../order/src/main/java/com/ecommerce/order/services/OrderService.java) |
| Event consumer | [`notification/src/main/java/com/ecommerce/notification/OrderEventConsumer.java`](../../notification/src/main/java/com/ecommerce/notification/OrderEventConsumer.java) |
| Monolithic starting point | [`additional/ecom-application`](../../additional/ecom-application) |

## Decomposition exercise: monolith to services

The [`additional/ecom-application`](../../additional/ecom-application) module contains product, user, cart, and order controllers in one Spring Boot application. A safe extraction sequence is:

1. Identify a business capability with a clear data owner.
2. Define its API and tests while it is still in the monolith.
3. Move the capability and its database writes behind an adapter.
4. Introduce a network boundary and contract tests.
5. Move the data store ownership.
6. Introduce events only after the synchronous boundary is stable.
7. Remove the old code path and measure the operational cost.

Do not split a monolith by technical layers such as “controller service” and “repository service.” Split it by business capability.

## Common design patterns

| Pattern | Why it matters here |
|---|---|
| API Gateway | One public entry point and edge policy |
| Service discovery | Find changing service instances without hard-coded host addresses |
| Externalized configuration | Change environment-specific values without rebuilding every service |
| Database per service | Preserve ownership and independent schema evolution |
| Event-driven integration | Decouple notification from checkout |
| Circuit breaker | Stop repeatedly calling an unhealthy dependency |
| Transactional outbox | Prevent a database commit and event publish from diverging |
| Saga | Coordinate a multi-service business process without a global transaction |
| Bulkhead | Prevent one dependency from consuming all threads/connections |
| Distributed tracing | Follow one request across multiple processes |

## Interview questions

**Why not make everything asynchronous?**  
Asynchronous messaging improves decoupling but makes the user experience, error handling, ordering, and debugging harder. Use synchronous calls for a required immediate answer and events for independent consequences.

**What is the biggest cost of microservices?**  
Operations and consistency. A local method call becomes a network call with latency, timeouts, retries, authentication, versioning, and partial failure.

**How do you choose a service boundary?**  
Start with domain language, ownership, change frequency, transaction boundaries, and data ownership. A service boundary that constantly requires distributed joins is probably wrong.

**How do you keep services independent?**  
Own separate data, publish stable contracts, avoid shared entity libraries, automate build/deploy, and observe each service independently.
