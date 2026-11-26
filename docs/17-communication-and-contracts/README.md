# 17. Communication, API Design, and Contracts

## REST maturity in practice

The Product Service is a practical level-2 REST API:

```java
@GetMapping("/{id}")
public ResponseEntity<ProductResponse> getProductById(
        @PathVariable String id) {
    return productService.getProductById(id)
            .map(ResponseEntity::ok)
            .orElseGet(() -> ResponseEntity.notFound().build());
}
```

It uses resources, verbs, content negotiation through JSON, and status codes. It does not need to implement hypermedia to be useful REST.

### REST design checklist

- resource nouns, not verbs in paths;
- idempotent `PUT`/`DELETE`;
- `201` for creation and `Location` when useful;
- `400` for malformed/invalid input;
- `401` versus `403` distinction;
- `409` for business conflicts;
- pagination and stable sorting for collections;
- RFC 7807-style problem details;
- explicit timeout and retry behavior for clients.

## gRPC deep dive

**[Blueprint]**

```proto
syntax = "proto3";

package product.v1;

service ProductService {
  rpc Get(GetRequest) returns (Product);
  rpc ListStock(stream StockQuery) returns (stream StockUpdate);
}

message GetRequest {
  string id = 1;
}

message Product {
  string id = 1;
  string name = 2;
  int64 price_minor = 3;
  string currency = 4;
}
```

Protobuf field numbers are part of the wire contract. Never reuse a removed field number; reserve it. Add optional fields compatibly. HTTP/2 multiplexing and streaming reduce connection overhead, but load balancers, browser support, debugging tools, and gateway translation must be considered.

## GraphQL Federation

**[Blueprint]**

```graphql
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Money!
}

extend type Query {
  product(id: ID!): Product
}
```

Federated subgraphs own fields and a router composes the graph. Guard against:

- resolver N+1 calls;
- unbounded query depth/complexity;
- authorization at field and object level;
- inconsistent nullability;
- schema ownership conflict;
- cache keys that omit user/tenant context.

The repository is REST-based; GraphQL is a design option, not an implemented module.

## RabbitMQ, Kafka, queues, and pub/sub

| Requirement | Better fit |
|---|---|
| Work queue with one worker handling each task | RabbitMQ queue or Kafka consumer group |
| Replay from a durable log | Kafka |
| Complex routing and short-lived brokered messages | RabbitMQ |
| Independent consumers with their own position | Kafka consumer groups |
| Simple request/reply | REST or gRPC |

The Order Service publishes to Kafka through Spring Cloud Stream:

```yaml
spring:
  cloud:
    stream:
      bindings:
        createOrder-out-0:
          destination: order.exchange
          content-type: application/json
```

The Notification Service subscribes with `notification-group`. A different group gets an independent subscription.

## Event notification versus carried state

```json
// notification
{"eventType":"OrderCreated","orderId":"42"}

// carried state
{
  "eventType": "OrderCreated",
  "eventId": "evt-1",
  "orderId": "42",
  "userId": "u-7",
  "totalAmount": 119.98,
  "items": [{"productId":"p-1","quantity":2,"unitPrice":59.99}]
}
```

Carried state reduces callback coupling but increases payload/schema ownership. Include an event version and avoid leaking entities.

## Gateway patterns

The repository's Gateway is a generic edge gateway. A BFF would split APIs by client:

```text
/web-api  -> web BFF -> Product/User/Order
/mobile-api -> mobile BFF -> Product/User/Order
```

Request aggregation reduces client round trips but increases gateway latency and dependency coupling. Protocol translation is useful when public REST maps to internal gRPC or legacy systems. Edge caching is safe only when cache keys include all authorization/tenant dimensions.

## Contract evolution

### URI/header/media type

```text
URI:         /api/v2/orders
Header:      X-API-Version: 2
Media type:  Accept: application/vnd.shop.order.v2+json
```

Prefer additive changes. Use expand-and-contract for renames:

```text
1. producer emits old and new field
2. consumers learn new field
3. migrate all consumers
4. stop old field
5. remove old schema
```

### Contract test

**[Blueprint]**

```java
@Pact(consumer = "order-service", provider = "product-service")
public RequestResponsePact productContract(
        PactDslWithProvider builder) {
    return builder
            .given("product 42 exists")
            .uponReceiving("lookup product")
                .method("GET")
                .path("/api/products/42")
            .willRespondWith()
                .status(200)
                .body(newJsonBody(body -> {
                    body.stringType("name");
                    body.numberType("stockQuantity");
                }).build())
            .toPact();
}
```

### Schema registry compatibility

**[Blueprint]**

```json
{
  "type": "record",
  "name": "OrderCreated",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "orderId", "type": "long"},
    {"name": "currency", "type": "string", "default": "USD"}
  ]
}
```

Backward compatibility lets new consumers read old records. Forward compatibility lets old consumers read new records. Full compatibility checks both. Adding a field with a default is safer than renaming/removing a required field. Enforce compatibility in CI instead of discovering it after a producer release.

## Interview Q&A

**Why is a shared DTO library risky?**
It can force synchronized version releases and expose internal models. Share stable primitives only when the ownership and compatibility policy are explicit.

**Why does a consumer-driven contract not replace E2E tests?**
It verifies an interaction boundary. E2E verifies a business flow across many boundaries. Both have different failure coverage.
