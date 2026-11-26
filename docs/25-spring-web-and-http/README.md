# 25. Spring Web, Embedded Servers, and HTTP Clients

## 1. Spring MVC request lifecycle

```text
HTTP request
 -> servlet Filter chain
 -> DispatcherServlet
 -> HandlerMapping
 -> HandlerAdapter
 -> controller
 -> HttpMessageConverter (Jackson)
 -> response
```

An exception flows through `HandlerExceptionResolver`. `@RestControllerAdvice` is the application-friendly way to translate validation and domain errors consistently.

### Filter versus interceptor versus AOP

| Extension | Scope | Typical use |
|---|---|---|
| Servlet `Filter` | Before/after servlet chain | security, raw request, correlation ID |
| `OncePerRequestFilter` | Once per dispatch | bearer extraction, request logging |
| `HandlerInterceptor` | MVC handler | route-aware authorization/metrics |
| Spring AOP | proxied method | transactions, service policies |

Do not put business rules in a filter that cannot understand the controller/domain.

## 2. Validation and problem details

```java
public record CreateProductRequest(
        @NotBlank String name,
        @Positive BigDecimal price,
        @NotBlank String currency) {}

@PostMapping
ResponseEntity<ProductResponse> create(
        @Valid @RequestBody CreateProductRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED)
            .body(service.create(request));
}
```

Use custom `ConstraintValidator` for reusable semantic checks, and return a stable error shape:

```json
{
  "type": "https://example.test/problems/validation",
  "title": "Validation failed",
  "status": 400,
  "detail": "One or more fields are invalid",
  "instance": "/api/products",
  "errors": [{"field":"price","message":"must be positive"}]
}
```

Spring's `ProblemDetail` supports RFC 7807-style responses. Do not expose stack traces or internal SQL.

## 3. Embedded Tomcat and concurrency

Tomcat accepts connections and dispatches request work to a worker pool. Important controls include:

```yaml
server:
  tomcat:
    threads:
      max: 200
    accept-count: 100
    max-connections: 8192
  shutdown: graceful
```

These are capacity settings, not performance guarantees. Size the pool with downstream latency, database pool size, CPU, and memory in mind. A request thread waiting on a database connection still consumes capacity.

Virtual threads can make blocking I/O cheaper in suitable applications:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

They do not make a database unlimited, remove the need for timeouts, or make CPU-heavy work parallel for free. Test pinning and library behavior.

## 4. WebFlux and backpressure

```java
public Flux<ProductResponse> streamProducts() {
    return repository.findAll()
            .map(mapper::toResponse);
}
```

WebFlux uses non-blocking pipelines (commonly Netty event loops). Blocking JDBC or file calls on event-loop threads can stall many requests; isolate them on a bounded scheduler or use non-blocking drivers.

Choose MVC plus virtual threads for conventional blocking applications when simplicity and ecosystem compatibility matter. Choose WebFlux when the workload is naturally streaming/non-blocking and the team can operate reactive code correctly.

## 5. HTTP clients

The repository demonstrates RestClient, WebClient, HTTP interfaces, RestTemplate, and OpenFeign under [`additional/interservice`](../../additional/interservice). Order uses HTTP interfaces and a load-balanced `RestClient.Builder`.

### RestClient

```java
ProductResponse product = restClient.get()
        .uri("/api/products/{id}", id)
        .retrieve()
        .onStatus(HttpStatusCode::is4xxClientError,
                (request, response) -> {
                    throw new ProductNotFoundException(id);
                })
        .body(ProductResponse.class);
```

### WebClient

```java
Mono<ProductResponse> product = webClient.get()
        .uri("/api/products/{id}", id)
        .retrieve()
        .bodyToMono(ProductResponse.class)
        .timeout(Duration.ofSeconds(2));
```

### Declarative HTTP interface

```java
@HttpExchange("/api/products")
interface ProductClient {
    @GetExchange("/{id}")
    ProductResponse find(@PathVariable String id);
}
```

Configure connection pooling, maximum idle time, connect/response timeout, TLS trust, and request interceptors. Propagate trace context without logging authorization headers.

## 6. API versioning

URI, header, and media-type versioning each have operational tradeoffs. Version behavior, not every internal class. Additive response fields are normally compatible; changing meaning or removing required fields is not.

## Interview Q&A

**Why use `OncePerRequestFilter` instead of a plain filter?**
Servlet dispatches can invoke filters more than once for the same logical request. The base class provides once-per-request behavior for cross-cutting request logic.

**Why can increasing Tomcat threads make outages worse?**
More concurrent requests can overwhelm the database or downstream service, increasing queueing and latency. Capacity must be balanced across the whole dependency graph.

**When is WebFlux a poor choice?**
When most calls are blocking, the team is unfamiliar with reactive debugging, or the problem does not benefit from non-blocking concurrency.
