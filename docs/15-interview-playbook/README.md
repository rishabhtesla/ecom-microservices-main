# 15. Java Microservices Interview Playbook

## A 60-second project explanation

> This is an e-commerce system split into Product, User, Order, and Notification services, fronted by a Spring Cloud Gateway. Product and Order use PostgreSQL, User uses MongoDB, and Order publishes an `OrderCreatedEvent` to Kafka for Notification. Eureka provides discovery and Config Server externalizes configuration. Keycloak issues OAuth2/OIDC tokens; the gateway validates JWTs. Resilience4j provides retries, circuit breakers, and rate limiting, while Micrometer, Zipkin, Prometheus, Grafana, Loki, and Alloy provide observability. The next production steps are an outbox, idempotent checkout, payment state management, Kubernetes manifests, contract tests, and secret management.

That answer is short, accurate, and gives the interviewer several directions to explore.

## A five-minute architecture explanation

Use this order:

1. **Entry point:** client calls Gateway on `8080`.
2. **Security:** Keycloak issues a JWT; Gateway validates issuer/signature/expiry and maps roles.
3. **Routing:** Gateway resolves Product/User/Order through Eureka.
4. **Data:** Product and Order own PostgreSQL databases; User owns a MongoDB database.
5. **Communication:** Order calls Product/User synchronously with a load-balanced RestClient HTTP interface.
6. **Events:** Order publishes `OrderCreatedEvent` through Spring Cloud Stream to Kafka; Notification consumes it in a group.
7. **Reliability:** circuit breakers, retries, rate limits, timeouts, idempotency, and an outbox are discussed separately because not all are complete in this checkout.
8. **Operations:** Config Server, Actuator, Micrometer, Zipkin, Prometheus, Grafana, Loki, and Docker Compose support the platform.
9. **Production evolution:** Kubernetes, managed data/brokers, secrets, CI/CD, payment integration, and contract tests.

## Project-specific questions and answers

| Question | Strong answer |
|---|---|
| Why separate Product and Order? | They have different business ownership, data models, scaling patterns, and change rates. Order references product IDs rather than joining Product's database. |
| Why does Order use both HTTP and Kafka? | It needs a synchronous product/user answer during cart validation, but notification is an independent consequence that can be eventually consistent. |
| Why Eureka? | It removes fixed instance addresses and enables discovery/load balancing. In Kubernetes, Service DNS may replace it, but the failure and timeout concerns remain. |
| Why Config Server? | It centralizes versioned service configuration and supports environment-specific values/refresh. Secrets still belong in a secret manager. |
| Why an API gateway? | A stable external boundary for routing, authentication, rate limiting, and observability while keeping internal topology private. |
| Is the gateway enough for security? | No. Receiving services and consumers must also authenticate/authorize where appropriate; a forwarded user header is not identity. |
| What does a JWT provide? | A signed token representation. The resource server validates signature and claims; OAuth2 defines how the token is obtained and used. |
| Authorization Code versus Client Credentials? | Authorization Code represents an end user, usually with PKCE for public clients. Client Credentials represents a machine/service with no user. |
| What does a circuit breaker solve? | It stops repeatedly calling an unhealthy dependency and gives it recovery time. It does not fix the dependency or replace timeouts. |
| Why not retry every failure? | Retries amplify load and can duplicate writes. Retry only transient failures, with deadlines, jitter, bounds, and idempotency. |
| What is the outbox pattern? | Save the business change and an event row in one local transaction, then relay the event to Kafka. It closes the database/message dual-write gap. |
| At-least-once delivery implication? | A consumer can see duplicates, so it must use an event ID/business key and make its side effect idempotent. |
| How would you add payment? | Add a Payment Service with provider idempotency keys, signed webhook verification, local payment state, explicit order transitions, and payment events. |
| What should be in a health check? | Liveness should indicate process recoverability; readiness should indicate whether the instance can safely receive traffic. Do not expose secrets. |
| How do you debug one failed checkout? | Start with trace/correlation ID, inspect gateway auth/routing, follow Order's downstream spans, check database commit, Kafka publish/lag, consumer logs, and DLQ. |
| How do you scale Notification? | Increase consumer instances up to partition count and monitor lag; use a stable event key and ensure the handler is idempotent. |
| What is the current Kubernetes status? | No tracked manifests are present. I can describe the Deployment/Service/ConfigMap/Secret/probe/HPA design and would validate it in a disposable namespace. |
| What is the current test status? | The repository has module context tests and a Postman collection, but it needs unit, contract, Testcontainers, messaging, and failure-path coverage for production confidence. |

## Code locations to memorize

| Talking point | File |
|---|---|
| Gateway routes and filters | [`gateway/GatewayConfig.java`](../../gateway/src/main/java/com/ecommerce/gateway/GatewayConfig.java) |
| Gateway JWT policy | [`gateway/security/SecurityConfig.java`](../../gateway/src/main/java/com/ecommerce/gateway/security/SecurityConfig.java) |
| Product REST boundary | [`product/controllers/ProductController.java`](../../product/src/main/java/com/ecommerce/product/controllers/ProductController.java) |
| Product persistence | [`product/repositories/ProductRepository.java`](../../product/src/main/java/com/ecommerce/product/repositories/ProductRepository.java) |
| Order-to-service clients | [`order/clients`](../../order/src/main/java/com/ecommerce/order/clients) |
| Order event producer | [`order/services/OrderService.java`](../../order/src/main/java/com/ecommerce/order/services/OrderService.java) |
| Notification consumer | [`notification/OrderEventConsumer.java`](../../notification/src/main/java/com/ecommerce/notification/OrderEventConsumer.java) |
| Keycloak Admin API | [`user/services/KeyCloakAdminService.java`](../../user/src/main/java/com/ecommerce/user/services/KeyCloakAdminService.java) |
| Resilience configuration | [`configserver/src/main/resources/config/order-service.yml`](../../configserver/src/main/resources/config/order-service.yml) |
| Local platform | [`deploy/docker/docker-compose.yml`](../../deploy/docker/docker-compose.yml) |

## Whiteboard sequence: checkout

```text
Client
  POST /api/cart + JWT
    -> Gateway validates JWT
    -> Order resolves Product through Eureka/RestClient
    -> Order resolves User through Eureka/RestClient
    -> Order stores CartItem

Client
  POST /api/orders + idempotency key
    -> Order loads cart and price snapshots
    -> Order writes order + items + outbox in one transaction
    -> relay publishes OrderCreated
    -> Notification consumes in notification-group
    -> Payment/analytics may consume independently
```

When asked about the current implementation, say that the checked-in code publishes through `StreamBridge` after saving and clearing the cart; the outbox in the diagram is the production improvement.

## Red flags to call out proactively

Showing awareness of limitations is stronger than claiming perfection:

- database names in Compose and service configuration are not aligned;
- Redis is used by the gateway limiter but absent from Compose;
- local images are referenced by fixed published names;
- default credentials and exposed management endpoints are learning-only;
- a user ID header is trusted instead of deriving identity from JWT;
- the custom JWT role parser is not null/type safe;
- checkout has no outbox or idempotency key;
- Notification currently logs instead of delivering/persisting notifications;
- tests are mostly context-load smoke tests;
- Kubernetes manifests and CI/CD workflows are not tracked;
- primary POMs are Spring Boot 3.4.x/Java 21, not Boot 4/Spring 7.

## Final revision checklist

Before an interview, be able to explain without notes:

- service boundaries and database ownership;
- a request through Gateway, Eureka, and Config Server;
- RestClient/Feign/RestTemplate trade-offs;
- Kafka topic, partition, offset, and group behavior;
- at-least-once delivery and idempotent consumers;
- JWT validation and OAuth2 grant selection;
- circuit breaker states and retry amplification;
- outbox and saga patterns;
- logs, metrics, and traces for one checkout;
- Docker network DNS and readiness;
- Kubernetes Deployment versus Service and readiness versus liveness;
- payment service security and state transitions;
- how you would test a dependency outage;
- what is implemented versus a production blueprint in this repository.

## A concise “what would you improve?” answer

> First I would make the local platform deterministic: align database names, add or remove Redis rate limiting, fix the image/runtime port mismatch, and remove default credentials from the deployment path. Next I would make checkout reliable with an idempotency key, a transactional outbox, Kafka retry/DLQ, and contract/Testcontainers tests. Then I would add Kubernetes manifests, managed dependencies, secret rotation, SLO dashboards, and a payment state machine with signed webhooks.
