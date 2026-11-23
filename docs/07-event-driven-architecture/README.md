# 7. Event-Driven Architecture with Kafka

## Why use an event here?

Checkout should not wait for email, invoice generation, analytics, or seller notifications. Order Service publishes a fact; independent consumers react:

```text
Order committed
  -> OrderCreatedEvent
  -> notification consumer
  -> future payment/analytics/search consumers
```

The producer does not need a direct URL for every consumer. Kafka stores the event long enough for consumers to process it, replay it, or catch up after a restart.

## Kafka vocabulary

| Term | Meaning |
|---|---|
| Broker | Kafka server that stores and serves records |
| Topic | Named stream of records |
| Partition | Ordered append-only log inside a topic |
| Offset | Position of a record within a partition |
| Producer | Application that publishes records |
| Consumer | Application that reads records |
| Consumer group | Consumers sharing work; one partition is assigned to one member at a time |
| Key | Value used to choose a partition and preserve per-key ordering |
| Retention | How long records remain available for reads/replays |

Kafka ordering is per partition, not globally across a topic. If all events for an order must be ordered, use the order ID as the key.

## Spring Cloud Stream publisher

Order Service configures an output binding:

```yaml
spring:
  cloud:
    stream:
      bindings:
        createOrder-out-0:
          destination: order.exchange
          content-type: application/json
      kafka:
        binder:
          brokers: kafka:9092
```

The service sends the typed event through `StreamBridge`:

```java
OrderCreatedEvent event = new OrderCreatedEvent(
        savedOrder.getId(),
        savedOrder.getUserId(),
        savedOrder.getStatus(),
        mapToOrderItemDTOs(savedOrder.getItems()),
        savedOrder.getTotalAmount(),
        savedOrder.getCreatedAt());

streamBridge.send("createOrder-out-0", event);
```

The complete workflow is in [`order/services/OrderService.java`](../../order/src/main/java/com/ecommerce/order/services/OrderService.java).

## Spring Cloud Stream consumer

Notification Service declares a function and binds it to the same destination:

```yaml
spring:
  cloud:
    function:
      definition: orderCreated
    stream:
      bindings:
        orderCreated-in-0:
          destination: order.exchange
          group: notification-group
          content-type: application/json
      kafka:
        binder:
          brokers: localhost:9092
```

```java
@Bean
public Consumer<OrderCreatedEvent> orderCreated() {
    return event -> {
        log.info("Received order created event for order: {}",
                event.getOrderId());
        log.info("Received order created event for user id: {}",
                event.getUserId());
    };
}
```

The group means multiple Notification Service instances share the work instead of each receiving every record. A different consumer group, such as a future `analytics-group`, receives its own copy of the logical stream.

## Event contract design

An event should be self-contained enough for its consumers:

```json
{
  "eventType": "OrderCreated",
  "eventVersion": 1,
  "eventId": "uuid",
  "occurredAt": "2026-01-01T10:00:00Z",
  "orderId": 42,
  "userId": "user-7",
  "status": "CONFIRMED",
  "totalAmount": 119.98,
  "items": [
    {
      "productId": "12",
      "quantity": 2,
      "unitPrice": 59.99
    }
  ]
}
```

Avoid leaking JPA entities or internal database column names into the event. Include an event ID for deduplication, a version for evolution, and an occurrence timestamp. Use a schema registry or reviewed JSON/Avro/Protobuf contracts as the system grows.

## Delivery semantics

### At-most-once

Commit the offset before processing. A crash can lose the event, but no duplicate is expected.

### At-least-once

Process first and commit the offset after success. A crash between processing and offset commit can deliver the event again. This is the practical default for business events, so consumers must be idempotent.

### Exactly-once

Kafka can provide exactly-once behavior for particular Kafka read/process/write pipelines, but external side effects such as email or payment providers still need idempotency. “Exactly once” is not a substitute for an idempotency key.

## Outbox and inbox

The current Order Service publishes after `save` and cart clearing. That creates a database/message dual-write gap. The production design is:

```text
local database transaction:
  order rows + outbox row

relay:
  outbox row -> Kafka

consumer:
  if eventId already processed: acknowledge and stop
  else perform effect + record eventId atomically
```

For email, store a notification record with a unique `eventId` or `(orderId, notificationType)` constraint before calling the provider. Retry the provider call with the provider's idempotency key.

## Failure handling

Define behavior for:

- malformed payload;
- unknown event version;
- transient downstream failure;
- poison message that always fails;
- broker outage;
- consumer rebalance;
- duplicate event;
- out-of-order event.

Use bounded retries with backoff and a dead-letter topic. Alert on DLQ growth. Do not retry a poison message forever and block an entire partition.

## Kafka versus RabbitMQ in this project

The Compose file starts both:

- **Kafka:** used by Order and Notification for business events and by the focused Kafka demos.
- **RabbitMQ:** used by Spring Cloud Bus configuration refresh and retained for course demonstrations.

Kafka is optimized for durable ordered streams and replay. RabbitMQ is optimized for brokered work queues and routing patterns. Choose based on delivery, replay, ordering, throughput, and operational requirements—not popularity.

## Additional Kafka examples

The [`additional/kafka`](../../additional/kafka) module demonstrates:

- function-style `Supplier` and `Consumer`;
- JSON and text bindings;
- consumer groups;
- a partitioned topic declaration;
- an uppercase function;
- an alternative `@KafkaListener` example in commented code.

Read [`producer/application.yml`](../../additional/kafka/producer/src/main/resources/application.yml) and [`consumer/application.yml`](../../additional/kafka/consumer/src/main/resources/application.yml) together; the destination and binding names must match.

## Interview questions

**Why use a consumer group?**  
It lets multiple instances share partitions for horizontal scaling while each group receives its own logical subscription.

**How do you handle duplicate events?**  
Give each event a stable ID, store processed IDs or use a unique business key, make the handler side effect idempotent, and acknowledge only after durable success.

**How do you preserve ordering?**  
Use a stable key such as `orderId`, ensure related records land in the same partition, and avoid assuming ordering across partitions or topics.

**What happens when a consumer is down?**  
Kafka retains records according to the retention policy. When the consumer returns, it resumes from its committed offset, subject to retention and rebalance behavior.
