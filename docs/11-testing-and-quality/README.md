# 11. Testing and Quality

## Test the boundary you own

Distributed applications need more than controller unit tests. Use a test pyramid:

```text
many fast unit tests
  -> slice/API tests
  -> consumer/provider contract tests
  -> integration tests with real infrastructure
  -> a small number of end-to-end tests
```

Each layer catches a different failure. A green `contextLoads` test only proves that a Spring context can start with the current classpath.

## What exists today

Each primary module has a Spring Boot test class such as:

```java
@SpringBootTest
class ProductApplicationTests {
    @Test
    void contextLoads() {
    }
}
```

The Postman collection at the repository root provides a manual API workflow. The focused `additional/` modules also contain generated context tests. Treat these as a baseline, not complete distributed-system coverage.

Run one module:

```bash
cd product
./mvnw test

cd ../order
./mvnw test
```

Run all independent modules:

```bash
for module in configserver eureka gateway product user order notification; do
  (cd "$module" && ./mvnw test) || exit 1
done
```

## Unit tests

Test business rules without Spring:

```java
@ExtendWith(MockitoExtension.class)
class ProductServiceTest {
    @Mock
    private ProductRepository repository;

    @InjectMocks
    private ProductService service;

    @Test
    void inactiveProductIsNotReturned() {
        when(repository.findByIdAndActiveTrue(7L))
                .thenReturn(Optional.empty());

        assertThat(service.getProductById("7")).isEmpty();
    }
}
```

Prioritize:

- price and stock validation;
- soft delete behavior;
- order total and line-item mapping;
- empty-cart handling;
- retry/fallback behavior;
- event payload construction;
- idempotency and duplicate handling.

## Web/API slice tests

Use `@WebMvcTest` for MVC controllers and `WebTestClient`/`@WebFluxTest` for the reactive gateway. Verify:

- routes and methods;
- validation errors;
- required headers;
- status codes;
- authorization decisions;
- problem response shape.

Do not mock the entire application so heavily that serialization and routing are never exercised.

## Persistence integration tests

Use Testcontainers for PostgreSQL and MongoDB so tests run against the same database engines as local deployment:

```java
@Testcontainers
class ProductRepositoryIT {
    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:14");
}
```

Test migrations, indexes, query semantics, transaction boundaries, and optimistic locking. Avoid relying on H2 to prove PostgreSQL behavior; H2 can accept different SQL and transaction semantics.

H2 is commented out in the primary JPA POMs. It can still be useful for very fast repository experiments, but it should not be the only persistence integration test when production uses PostgreSQL.

## Contract tests

The Order Service depends on Product and User API shapes. A provider change should not break Order at runtime. Use consumer-driven contracts to verify:

```text
Order consumer expects:
  GET /api/products/{id}
  200 -> ProductResponse fields
  404 -> unavailable product behavior

Product provider verifies:
  it still satisfies the contract
```

For Kafka, test event schema compatibility and unknown fields. Keep event fixtures versioned.

## Messaging tests

Test:

- publisher sends the expected destination and payload;
- consumer handles a valid event;
- malformed payload goes to an error/DLQ path;
- duplicate event ID does not repeat the side effect;
- consumer restart resumes from the committed offset;
- partition key preserves the required order.

Use an embedded broker only when it behaves close enough to production; a containerized Kafka test is often more representative.

## End-to-end test

Keep a small happy path:

```text
create a user
create a product
add product to cart
create order
assert order response
assert OrderCreatedEvent is consumed
assert notification record/log/metric exists
```

Add negative paths:

- invalid JWT;
- insufficient stock;
- missing user;
- product service unavailable;
- duplicate checkout idempotency key;
- Kafka unavailable after order commit.

## Quality gates

For every service:

- compile and unit tests;
- static analysis and formatting;
- dependency vulnerability scan;
- API/contract compatibility check;
- container image scan;
- migration check;
- smoke test with health/readiness endpoints.

Never let a successful build hide skipped tests. Report test count, skipped count, and integration infrastructure status.

## Interview questions

**Why are contract tests valuable in microservices?**  
They verify the exact interaction between independently deployed producer and consumer without requiring every service to run in one slow end-to-end environment.

**Why not use only end-to-end tests?**  
They are slow, brittle, expensive to diagnose, and cover too few failure combinations. Lower-level tests give faster and more precise feedback.

**What would you test for a retry?**  
Transient failure is retried up to the configured bound, permanent failure is not retried, a fallback is returned only after attempts are exhausted, and a retried write is not duplicated.
