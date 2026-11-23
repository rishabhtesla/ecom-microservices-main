# 5. API Gateway

## What belongs at the edge?

An API gateway is the controlled boundary between external clients and internal services. It is a good place for cross-cutting edge concerns:

- route selection;
- TLS termination and header policy;
- authentication and coarse authorization;
- request correlation;
- rate limiting;
- protocol translation;
- response and request size limits;
- safe retries for idempotent reads;
- a consistent public API.

It should not become a second monolith containing product, user, or order business rules.

## The gateway in this repository

The gateway is a reactive Spring Cloud Gateway application on port `8080`. Its route definitions are Java-based in [`GatewayConfig`](../../gateway/src/main/java/com/ecommerce/gateway/GatewayConfig.java):

```java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route("product-service", r -> r
                    .path("/api/products/**")
                    .filters(f -> f
                            .retry(retry -> retry
                                    .setRetries(10)
                                    .setMethods(HttpMethod.GET))
                            .requestRateLimiter(config -> config
                                    .setRateLimiter(redisRateLimiter())
                                    .setKeyResolver(hostNameKeyResolver()))
                            .circuitBreaker(config -> config
                                    .setName("ecomBreaker")
                                    .setFallbackUri(
                                            "forward:/fallback/products")))
                    .uri("lb://PRODUCT-SERVICE"))
            .route("user-service", r -> r
                    .path("/api/users/**")
                    .uri("lb://USER-SERVICE"))
            .route("order-service", r -> r
                    .path("/api/orders/**", "/api/cart/**")
                    .uri("lb://ORDER-SERVICE"))
            .route("eureka-server", r -> r
                    .path("/eureka/main")
                    .filters(f -> f.rewritePath(
                            "/eureka/main", "/"))
                    .uri("http://localhost:8761"))
            .route("eureka-server-static", r -> r
                    .path("/eureka/**")
                    .uri("http://localhost:8761"))
            .build();
}
```

`lb://` tells Spring Cloud LoadBalancer to resolve a logical Eureka service name. The public path is preserved for the primary API routes, so `/api/products/42` is sent to Product Service with the same path.

## Request flow

```text
client
  -> gateway:8080
      1. match path predicate
      2. authenticate JWT
      3. resolve service through Eureka
      4. apply rate limit/retry/circuit breaker
      5. forward request
      6. return response or fallback
```

The client should know the gateway URL, not every internal service address. Internal services can remain private in a container network or cluster.

## JWT resource-server security

The gateway configures WebFlux security:

```java
@Bean
public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
    return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(exchange -> exchange
                    .anyExchange().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2
                    .jwt(jwt -> jwt
                            .jwtAuthenticationConverter(
                                    grantedAuthoritiesExtractor())))
            .build();
}
```

The issuer and JWK settings are in `gateway-service-docker.yml`. The gateway validates the JWT signature and standard claims using Keycloak metadata; it should not trust a client-provided user ID header as identity.

The current role converter reads client roles from the `resource_access` claim for the `oauth2-pkce` client and prefixes them with `ROLE_`. The role-specific matchers are commented out, so the current active policy is “any valid token may access any gateway route.”

For production, make authorization explicit:

```java
.pathMatchers(HttpMethod.GET, "/api/products/**").hasAuthority("ROLE_CATALOG_READ")
.pathMatchers(HttpMethod.POST, "/api/products/**").hasAuthority("ROLE_CATALOG_WRITE")
.pathMatchers("/api/orders/**", "/api/cart/**").hasAuthority("ROLE_ORDER_USER")
.anyExchange().authenticated()
```

Use the exact authority format produced by the JWT converter. Test the claim shape from a real token rather than assuming realm roles and client roles are represented identically.

## Rate limiting

The gateway creates:

```java
new RedisRateLimiter(10, 20, 1)
```

The usual interpretation is a replenish rate of 10 tokens per second, a burst capacity of 20, and one token per request. The key resolver currently uses the remote host name:

```java
@Bean
public KeyResolver hostNameKeyResolver() {
    return exchange -> Mono.just(
            exchange.getRequest()
                    .getRemoteAddress()
                    .getHostName());
}
```

This is suitable for a demonstration, not a trustworthy identity boundary. Behind a proxy, many users may share one address; forwarded headers can also be spoofed unless the trusted proxy is configured. A production key might use a validated subject, API key, or tenant ID.

The Compose file does not define Redis, so the rate limiter is an explicit local-platform gap. Add Redis with persistence/HA requirements or use a different gateway limiter before enabling this route in a clean environment.

## Retry and circuit breaker

Only `GET` is retried in the product route, which avoids replaying a non-idempotent write. The circuit breaker forwards failures to:

```java
@RestController
public class FallbackController {
    @GetMapping("/fallback/products")
    public ResponseEntity<List<String>> productsFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(Collections.singletonList(
                        "Product service is unavailable"));
    }
}
```

The fallback should be a deliberate degraded response. Never return a successful-looking empty product list when the catalog is unavailable.

## Gateway versus service security

Gateway authentication is useful for client traffic, but it is not sufficient by itself:

- internal services may be reachable accidentally;
- a message consumer has no gateway;
- defense in depth protects against misrouting;
- service-to-service calls need their own identity and authorization.

Use network policy, mTLS or service tokens, and resource-server validation in services that accept direct traffic. Pass only trusted identity claims, not arbitrary headers from the original client.

## Gateway production checklist

- Add a real Redis deployment or disable the Redis limiter locally.
- Replace `localhost` Eureka routes with a network-resolvable address.
- Set connect, response, and pool timeouts.
- Retry only safe, idempotent operations.
- Bound request body and header sizes.
- Add correlation IDs and propagate W3C trace context.
- Return stable problem responses.
- Protect or remove the Eureka proxy route.
- Restrict Actuator endpoints.
- Test route authorization for every HTTP method.

## Interview questions

**Why use a gateway instead of exposing every service?**  
It gives clients one stable endpoint and centralizes edge concerns. It also reduces the public attack surface and allows internal topology to change.

**Should every business rule be in the gateway?**  
No. The gateway can make coarse access decisions, but the owning service must enforce business authorization and invariants.

**Why should a POST not be retried automatically?**  
A network timeout does not tell us whether the server committed the request. Retrying can create duplicates unless the operation has an idempotency key and deduplication.
