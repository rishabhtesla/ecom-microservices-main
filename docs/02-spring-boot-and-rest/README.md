# 2. Spring Boot and REST API Development

## The request-processing model

The common Spring Boot path is:

```text
HTTP request
  -> embedded server
  -> DispatcherServlet / WebFlux handler
  -> @RestController
  -> application service
  -> repository or downstream client
  -> DTO
  -> HTTP response
```

Spring Boot supplies auto-configuration, dependency injection, configuration binding, an embedded server, and production endpoints through Actuator. Spring MVC is used by Product, User, and Order; the Gateway uses the reactive WebFlux stack.

## Module responsibilities

Keep the layers understandable:

| Layer | Responsibility | Example |
|---|---|---|
| Controller | HTTP mapping, headers, status codes, input boundary | `ProductController` |
| DTO | External request/response contract | `ProductRequest`, `ProductResponse` |
| Service | Business rules and orchestration | `ProductService`, `OrderService` |
| Repository | Persistence abstraction | `ProductRepository`, `OrderRepository` |
| Entity/model | Data owned by the service | `Product`, `Order`, `OrderItem` |
| Client | Remote service contract | `ProductServiceClient`, `UserServiceClient` |

Do not expose JPA entities as public API contracts. A schema change inside a service should not silently change every caller.

## A complete controller example

This is the style used by the Product Service:

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    @GetMapping("/simulate")
    public ResponseEntity<String> simulateFailure(
            @RequestParam(defaultValue = "false") boolean fail) {
        if (fail) {
            throw new RuntimeException("Simulated Failure For Testing");
        }
        return ResponseEntity.ok("Product Service is OK");
    }

    @PostMapping
    public ResponseEntity<ProductResponse> createProduct(
            @RequestBody ProductRequest productRequest) {
        return new ResponseEntity<>(
                productService.createProduct(productRequest),
                HttpStatus.CREATED);
    }

    @GetMapping
    public ResponseEntity<List<ProductResponse>> getProducts() {
        return ResponseEntity.ok(productService.getAllProducts());
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProductById(
            @PathVariable String id) {
        return productService.getProductById(id)
                .map(ResponseEntity::ok)
                .orElseGet(() -> ResponseEntity.notFound().build());
    }

    @PutMapping("/{id}")
    public ResponseEntity<ProductResponse> updateProduct(
            @PathVariable Long id,
            @RequestBody ProductRequest productRequest) {
        return productService.updateProduct(id, productRequest)
                .map(ResponseEntity::ok)
                .orElseGet(() -> ResponseEntity.notFound().build());
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        boolean deleted = productService.deleteProduct(id);
        return deleted
                ? ResponseEntity.noContent().build()
                : ResponseEntity.notFound().build();
    }

    @GetMapping("/search")
    public ResponseEntity<List<ProductResponse>> searchProducts(
            @RequestParam String keyword) {
        return ResponseEntity.ok(productService.searchProducts(keyword));
    }
}
```

The source is [`product/controllers/ProductController.java`](../../product/src/main/java/com/ecommerce/product/controllers/ProductController.java). Notice that the controller delegates business work and maps `Optional` to `404`; it does not contain repository logic.

## Service and repository flow

The service creates a new entity, saves it, and maps it back to a response:

```java
public ProductResponse createProduct(ProductRequest request) {
    Product product = new Product();
    updateProductFromRequest(product, request);
    Product savedProduct = productRepository.save(product);
    return mapToProductResponse(savedProduct);
}
```

The repository is a Spring Data interface:

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByActiveTrue();

    @Query("""
        SELECT p FROM products p
        WHERE p.active = true
          AND p.stockQuantity > 0
          AND LOWER(p.name) LIKE LOWER(CONCAT('%', :keyword, '%'))
        """)
    List<Product> searchProducts(@Param("keyword") String keyword);

    Optional<Product> findByIdAndActiveTrue(Long id);
}
```

The checked-in repository has the same query as a single-line string. The formatted version above makes the JPQL intent easier to review.

## REST design rules

Use the HTTP contract deliberately:

| Situation | Response |
|---|---|
| Resource created | `201 Created`, preferably with a `Location` header |
| Successful read/update | `200 OK` |
| Successful delete | `204 No Content` |
| Resource absent | `404 Not Found` |
| Invalid request | `400 Bad Request` with a structured error |
| Unauthenticated | `401 Unauthorized` |
| Authenticated but forbidden | `403 Forbidden` |
| Conflict, such as duplicate idempotency key | `409 Conflict` |
| Unexpected server failure | `500 Internal Server Error`, without stack-trace details |

For a production API, add:

- `@Valid` on request DTOs and Bean Validation annotations;
- a `@RestControllerAdvice` that returns a stable error envelope;
- pagination for collection endpoints;
- explicit sorting and filtering rules;
- API versioning when compatibility requires it;
- correlation/request IDs;
- an idempotency key on checkout;
- OpenAPI documentation and contract tests.

### Structured error example

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Request validation failed",
  "status": 400,
  "detail": "One or more fields are invalid",
  "instance": "/api/products",
  "errors": {
    "price": "must be greater than zero"
  },
  "traceId": "4f3c..."
}
```

## Configuration and profiles

Local service bootstrap imports the Config Server:

```yaml
spring:
  application:
    name: product-service
  config:
    import: optional:configserver:http://localhost:8888
```

Docker overrides the host name with `http://config-server:8888`. The service-specific files then supply the port, database URL, Eureka URL, Actuator, and tracing settings.

Use profiles for environment differences, not for business behavior. Prefer typed `@ConfigurationProperties` for a group of settings over many scattered `@Value` fields.

## Actuator and health

Actuator provides operational endpoints such as `/actuator/health`, `/actuator/info`, and `/actuator/prometheus`. The project currently exposes `*` in several learning configurations. In production:

1. expose only health/readiness and metrics as needed;
2. place management endpoints on a separate port or protected network;
3. never return database credentials or internal topology from health details;
4. distinguish liveness from readiness;
5. make downstream dependency health checks intentional.

## What to say in an interview

**Why use DTOs?**  
DTOs create an anti-corruption boundary. They prevent persistence annotations, internal fields, and lazy relationships from becoming a public contract.

**Where should validation live?**  
Shape validation belongs at the API boundary. Business invariants belong in the application/domain service and must also protect non-HTTP entry points such as Kafka consumers.

**Why use `ResponseEntity`?**  
It makes status and headers explicit. In a larger codebase, a consistent exception handler can reduce controller noise while preserving a stable contract.
