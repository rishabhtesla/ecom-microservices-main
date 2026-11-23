# 6. Inter-Service Communication

## Choose the interaction from the business need

There are two broad choices:

| Choice | Use when | Cost |
|---|---|---|
| Synchronous HTTP | The caller needs a result before it can continue | Latency and temporal coupling |
| Asynchronous event | The producer announces a fact and consumers can process later | Eventual consistency, replay, ordering, and duplicate handling |

The Order Service demonstrates synchronous HTTP to obtain product and user information, then asynchronous Kafka publication for notification.

## Service discovery removes hard-coded hosts

The Order Service uses logical base URLs:

```java
@HttpExchange
public interface ProductServiceClient {
    @GetExchange("/api/products/{id}")
    ProductResponse getProductDetails(@PathVariable String id);
}
```

```java
@Bean
@LoadBalanced
public RestClient.Builder restClientBuilder() {
    return RestClient.builder();
}
```

The concrete client uses `http://product-service`:

```java
RestClient restClient = restClientBuilder
        .baseUrl("http://product-service")
        .defaultStatusHandler(
                HttpStatusCode::is4xxClientError,
                (request, response) -> Optional.empty())
        .build();

RestClientAdapter adapter = RestClientAdapter.create(restClient);
HttpServiceProxyFactory factory = HttpServiceProxyFactory
        .builderFor(adapter)
        .build();

return factory.createClient(ProductServiceClient.class);
```

Because the injected builder is load-balanced, the logical host is resolved through Eureka. The same pattern is used for [`UserServiceClient`](../../order/src/main/java/com/ecommerce/order/clients/UserServiceClient.java).

## Spring HTTP client options demonstrated here

### Spring HTTP interfaces and RestClient

This is the primary application's current style. HTTP interfaces keep the endpoint contract in one place, while `RestClient` provides the blocking request execution. It is concise and works well for request/response calls.

Add in production:

- connect, read, and connection-pool timeouts;
- status mapping for 404, 409, 429, and 5xx;
- authentication propagation or service credentials;
- metrics around downstream latency;
- a bounded circuit breaker;
- schema-compatible DTOs.

### OpenFeign

The focused example uses:

```java
@FeignClient(name = "provider")
public interface ProviderFeignClient {
    @GetMapping("/instance-info")
    String getInstanceInfo();
}
```

See [`additional/interservice/consumer`](../../additional/interservice/consumer). Feign is declarative and familiar in Spring Cloud applications, but the same timeout, retry, error, and observability design still applies.

### RestTemplate

The example uses a logical URL:

```java
private static final String PROVIDER_URL = "http://provider";

public String getInstanceInfo() {
    return restTemplate.getForObject(
            PROVIDER_URL + "/instance-info", String.class);
}
```

`RestTemplate` is blocking and is retained for existing applications. Prefer the newer `RestClient` for new blocking code unless the project has a reason to standardize on another client.

### WebClient

The `additional/interservice/consumer` module also contains `WebClient` configuration and a client. WebClient is appropriate for reactive pipelines, streaming responses, or when the whole call path is non-blocking. Do not mix reactive and blocking calls without understanding thread-pool pressure.

## The Order Service call graph

`CartService.addToCart` performs:

```text
1. ProductServiceClient.getProductDetails(productId)
2. validate stock
3. UserServiceClient.getUserDetails(userId)
4. persist or update CartItem
```

This is a chain of network calls inside a business operation. A product timeout and a user timeout have different operational meanings and should be measured separately.

## Error handling

Translate remote failures intentionally:

```text
404 product -> product unavailable / validation response
404 user    -> user does not exist
429         -> caller should back off
503/timeout -> dependency unavailable; do not pretend cart was saved
invalid JSON -> contract/versioning incident
```

Do not catch `Exception` and return a success-shaped default. Preserve the distinction between “the business rule rejected this request” and “the dependency could not answer.”

## Propagating identity and trace context

An internal call should carry:

- a service identity or access token;
- the trace/span context;
- a request/correlation ID;
- tenant context only if it is trusted and validated.

The primary Order module includes a custom `ClientHttpRequestInterceptor` that injects the current tracing context using Micrometer's `Tracer` and `Propagator`. Prefer the framework's supported propagation mechanism where possible and test it across gateway, HTTP, and Kafka boundaries.

## Versioning strategies

When a provider changes:

1. add fields rather than changing the meaning of existing fields;
2. make new fields optional for old consumers;
3. support old and new endpoints during migration;
4. use consumer-driven contract tests;
5. version events when the compatibility break is unavoidable.

Do not share a single DTO JAR across all services just to avoid writing mapping code. Shared libraries can silently force synchronized releases.

## Timeouts and backpressure

Every client should have:

```text
connection timeout: how long to establish a connection
read/response timeout: how long to wait for a response
pool timeout: how long to wait for a client connection
overall deadline: maximum business-operation budget
```

The downstream timeout must be shorter than the caller's deadline. Otherwise a gateway request can expire while internal threads continue doing work.

## Interview questions

**Feign or RestClient?**  
Neither is automatically more reliable. Feign is declarative and convenient; RestClient plus HTTP interfaces is a modern, explicit option. I choose based on project standards, reactive/blocking needs, timeout support, and observability integration.

**How do you prevent a slow dependency from exhausting the service?**  
Bound connection pools and threads, set deadlines, use circuit breakers/bulkheads, apply backpressure, and degrade or reject work deliberately.

**How do you authenticate service-to-service calls?**  
Use a service identity, usually OAuth2 client credentials or mTLS, validate the token at the receiving service, and authorize the requested operation. Do not rely on an untrusted forwarded user header.
