# Service Catalog and Code Map

This is the fastest page to open before an interview. It maps the business capability to its process, data, infrastructure, and source files.

## Primary services

| Service | Port | Business role | Persistence | Important dependencies | Start class |
|---|---:|---|---|---|---|
| Gateway | 8080 | Reactive API gateway, routing, JWT resource-server security, rate limiting, retries, circuit-breaker fallback | None | Eureka, Keycloak, Redis rate limiter, Config Server | [`GatewayApplication`](../../gateway/src/main/java/com/ecommerce/gateway/GatewayApplication.java) |
| Product | 8081 | Product catalog and inventory fields | PostgreSQL `product` locally, `productdb` in Docker profile | Eureka, Config Server, Actuator, Zipkin | [`ProductApplication`](../../product/src/main/java/com/ecommerce/product/ProductApplication.java) |
| User | 8082 | User profiles and Keycloak administration integration | MongoDB `ecom_user` | Eureka, Keycloak Admin API, Config Server | [`UserApplication`](../../user/src/main/java/com/ecommerce/user/UserApplication.java) |
| Order | 8083 | Cart, checkout, order persistence, event publication | PostgreSQL `order` locally, `orderdb` in Docker profile | Product/User over discovery, Kafka, Eureka, Config Server, Resilience4j | [`OrderApplication`](../../order/src/main/java/com/ecommerce/order/OrderApplication.java) |
| Notification | 8084 | Consumes order-created events and logs notification work | None in current implementation | Kafka | [`NotificationApplication`](../../notification/src/main/java/com/ecommerce/notification/NotificationApplication.java) |
| Config Server | 8888 | Serves externalized service configuration | Native classpath/filesystem | RabbitMQ Spring Cloud Bus | [`ConfigserverApplication`](../../configserver/src/main/java/com/ecommerce/configserver/ConfigserverApplication.java) |
| Eureka | 8761 | Service registry and discovery | In memory | Eureka clients | [`EurekaApplication`](../../eureka/src/main/java/com/ecommerce/eureka/EurekaApplication.java) |

The port and discovery settings live in:

- [`eureka/src/main/resources/application.yml`](../../eureka/src/main/resources/application.yml)
- [`configserver/src/main/resources/config/product-service.yml`](../../configserver/src/main/resources/config/product-service.yml)
- [`configserver/src/main/resources/config/user-service.yml`](../../configserver/src/main/resources/config/user-service.yml)
- [`configserver/src/main/resources/config/order-service.yml`](../../configserver/src/main/resources/config/order-service.yml)
- [`configserver/src/main/resources/config/gateway-service.yml`](../../configserver/src/main/resources/config/gateway-service.yml)

## External infrastructure

| Component | Default port | Used by | Purpose |
|---|---:|---|---|
| PostgreSQL | 5432 | Product, Order | Relational persistence and local multi-database bootstrap |
| MongoDB | 27017 | User | Document persistence |
| Kafka | 9092 | Order, Notification, additional Kafka demos | Business event transport |
| ZooKeeper | 2181 | Local Kafka image | Kafka coordination in the Compose setup |
| RabbitMQ | 5672 / 15672 | Config Server and optional messaging examples | Spring Cloud Bus and broker demonstrations |
| Keycloak | 8443 from host, 8080 inside network | Gateway, User, OAuth2 demos | OIDC issuer and admin API |
| Zipkin | 9411 | Gateway and services | Distributed trace UI and collector |
| Prometheus | 9090 | Product, User, Order | Metrics scraping |
| Grafana | 3000 | Prometheus/Loki | Dashboards |
| Loki/Grafana Alloy | 3100 / 12345 | Docker and application logs | Centralized log collection |
| pgAdmin | 5050 | PostgreSQL | Local database inspection |

The complete dependency graph is in [`deploy/docker/docker-compose.yml`](../../deploy/docker/docker-compose.yml).

## Public API through the gateway

The gateway routes these paths using service discovery:

| Method | Path | Destination | Source |
|---|---|---|---|
| `GET`, `POST`, `PUT`, `DELETE` | `/api/products/**` | `lb://PRODUCT-SERVICE` | [`GatewayConfig`](../../gateway/src/main/java/com/ecommerce/gateway/GatewayConfig.java) |
| `GET`, `POST`, `PUT` | `/api/users/**` | `lb://USER-SERVICE` | [`GatewayConfig`](../../gateway/src/main/java/com/ecommerce/gateway/GatewayConfig.java) |
| `GET`, `POST`, `DELETE` | `/api/orders/**`, `/api/cart/**` | `lb://ORDER-SERVICE` | [`GatewayConfig`](../../gateway/src/main/java/com/ecommerce/gateway/GatewayConfig.java) |
| `GET` | `/eureka/**` | Eureka | [`GatewayConfig`](../../gateway/src/main/java/com/ecommerce/gateway/GatewayConfig.java) |

### Product API

Implemented in [`ProductController`](../../product/src/main/java/com/ecommerce/product/controllers/ProductController.java):

```text
POST   /api/products
GET    /api/products
GET    /api/products/{id}
PUT    /api/products/{id}
DELETE /api/products/{id}       # soft delete: active=false
GET    /api/products/search?keyword=...
GET    /api/products/simulate?fail=true
```

### User API

Implemented in [`UserController`](../../user/src/main/java/com/ecommerce/user/controllers/UserController.java):

```text
GET  /api/users
GET  /api/users/{id}
POST /api/users
PUT  /api/users/{id}
```

### Cart and order API

Implemented in [`CartController`](../../order/src/main/java/com/ecommerce/order/controller/CartController.java) and [`OrderController`](../../order/src/main/java/com/ecommerce/order/controller/OrderController.java):

```text
POST   /api/cart                       # X-User-ID header + CartItemRequest
GET    /api/cart                       # X-User-ID header
DELETE /api/cart/items/{productId}     # X-User-ID header
POST   /api/orders                     # X-User-ID header
```

The current header is a teaching simplification. In production, derive the user identity from the validated JWT subject rather than trusting a caller-supplied `X-User-ID`.

## End-to-end order flow

1. The client sends `POST /api/cart` with `X-User-ID`.
2. Order Service calls Product Service and User Service through load-balanced HTTP interfaces.
3. The cart is persisted in the Order database.
4. `POST /api/orders` reads the cart, creates an order, clears the cart, and sends `OrderCreatedEvent` through `StreamBridge`.
5. Kafka delivers `order.exchange` to the `notification-group`.
6. Notification Service logs the order and user identifiers. Email, invoice, and seller notifications are currently extension points.

The publisher is [`OrderService`](../../order/src/main/java/com/ecommerce/order/services/OrderService.java); the consumer is [`OrderEventConsumer`](../../notification/src/main/java/com/ecommerce/notification/OrderEventConsumer.java).

## Additional learning modules

| Topic | Location |
|---|---|
| Monolithic baseline | [`additional/ecom-application`](../../additional/ecom-application) |
| Config Server native profile | [`additional/config-mgmt`](../../additional/config-mgmt) |
| Feign, RestTemplate, RestClient, WebClient, HTTP interfaces | [`additional/interservice`](../../additional/interservice) |
| Kafka producers/consumers and function style | [`additional/kafka`](../../additional/kafka) |
| Authorization code, client credentials, PKCE | [`additional/keycloak/keycloak-oauth-demos`](../../additional/keycloak/keycloak-oauth-demos) |

## Known inconsistencies to know before running

- The default service configuration uses `product` and `order`, while the Docker-specific profile uses the `productdb` and `orderdb` databases created by Compose. Verify that `SPRING_PROFILES_ACTIVE=docker` is loaded when running containers.
- The Compose gateway uses Redis-backed rate limiting, but the Compose file does not define a Redis service.
- The main Compose file points to published images (`dcode007/*`); local image builds and image names must agree.
- The notification service has a Dockerfile that exposes `8888`, although its configured application port is `8084`.
- The Config Server Compose healthcheck probes port `8761` instead of its own configured port `8888`.
- Several actuator endpoints are exposed broadly for learning. Restrict them behind authentication and network policy in production.
- The current notification consumer logs an event but does not persist a notification or call an email provider.
