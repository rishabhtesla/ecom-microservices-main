# 30. Testing Spring Boot Microservices

## 1. Test pyramid and test ownership

```text
unit              -> pure domain rules, fast
slice              -> one Spring layer
integration        -> real database/broker/client boundary
contract            -> provider/consumer compatibility
E2E                -> critical business journeys
```

Each level answers a different question. A passing controller unit test does not prove Jackson validation, transaction behavior, or a real PostgreSQL query.

## 2. Unit tests with JUnit and Mockito

```java
@ExtendWith(MockitoExtension.class)
class OrderTotalTest {
    @Test
    void sumsLineItems() {
        Order order = new Order();
        order.addItem(new OrderItem("p-1", 2, new BigDecimal("10.00")));

        assertThat(order.total()).isEqualByComparingTo("20.00");
    }
}
```

Mock collaborators, not the object under test. Prefer AssertJ's readable assertions. Test business edge cases: zero/negative quantity, currency mismatch, duplicate command, invalid state transition.

## 3. Spring slice tests

### MVC

```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {
    @Autowired MockMvc mvc;
    @MockitoBean ProductService service;

    @Test
    void rejectsBlankName() throws Exception {
        mvc.perform(post("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"name":"","price":10,"currency":"USD"}
                    """))
            .andExpect(status().isBadRequest());
    }
}
```

`@WebMvcTest` loads the web layer, not a real database. In newer Boot versions, `@MockitoBean`/`@MockitoSpyBean` replace older `@MockBean`/`@SpyBean` usage; use the annotation supported by the project's exact version.

### JPA

```java
@DataJpaTest
class ProductRepositoryTest {
    @Autowired ProductRepository repository;

    @Test
    void findsOnlyAvailableProducts() { ... }
}
```

Slice tests verify mapping/query behavior and normally roll back each test. They do not prove connection pool sizing or production migration compatibility.

### JSON

```java
@JsonTest
class OrderEventJsonTest {
    @Autowired JacksonTester<OrderCreatedEvent> json;

    @Test
    void serializesStableContract() throws Exception {
        assertThat(json.write(event).getJson())
            .extractingJsonPathStringValue("@.eventType")
            .isEqualTo("OrderCreated");
    }
}
```

## 4. Integration tests with Testcontainers

```java
@Testcontainers
@SpringBootTest
class ProductPostgresIT {
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:16");

    @Test
    void persistsUsingRealPostgres() { ... }
}
```

Use real PostgreSQL to catch dialect, constraint, transaction, and query issues. Similar containers can exercise Kafka, Redis, and Cassandra. Keep containers reusable where CI isolation allows it, but never share mutable state between tests.

## 5. HTTP stubs and WireMock

Stub external behavior, not your own service implementation:

```java
stubFor(get(urlEqualTo("/payments/42"))
    .willReturn(aResponse()
        .withStatus(200)
        .withHeader("Content-Type", "application/json")
        .withBody("""
            {"status":"AUTHORIZED"}
            """)));
```

Test timeout, 429, 5xx, malformed response, and slow response behavior. Verify the client sends the expected method, headers, path, and idempotency key.

## 6. Contract tests

Consumer-driven contracts describe the interaction a consumer relies on:

```text
consumer publishes expectation
provider verifies against its running implementation
broker shares result with release pipeline
```

They catch path/status/field incompatibilities earlier than E2E. Add provider states (“product exists”, “product is out of stock”) and avoid over-specifying fields that the consumer does not use.

For Kafka, schema compatibility tests should validate event fields, defaults, enum evolution, and headers. A schema registry is a runtime governance component; a JSON fixture alone is not enough.

## 7. E2E checkout test

```text
create test user
obtain token
create/find product
add to cart
submit checkout with idempotency key
assert order state
assert event/notification eventually appears
```

Use polling with a bounded deadline for eventual consistency, not arbitrary sleeps. Clean test data by tenant/run ID and isolate payment provider side effects.

## 8. Test reliability

- control clock/time zone and random IDs;
- wait on observable state, not fixed sleeps;
- avoid test order dependence;
- clean database/topics/cache;
- capture logs and container diagnostics on failure;
- run unit/slice tests on every change and heavier integration/E2E tests in appropriate CI stages;
- test graceful shutdown, retries, duplicate messages, and rollback paths.

## Interview Q&A

**What does `@SpringBootTest` prove that a unit test does not?**
It verifies application context wiring and, depending on configuration, real framework integration. It is slower and still does not prove production infrastructure behavior unless that infrastructure is included.

**Why are mocks dangerous for persistence tests?**
They cannot reveal SQL dialect errors, missing indexes, transaction boundaries, constraints, or lazy-loading behavior.

**How do you test at-least-once consumers?**
Deliver the same event twice, assert the business effect occurs once, and verify the inbox/unique-idempotency mechanism.
