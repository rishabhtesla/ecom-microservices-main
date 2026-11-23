# 9. Resilience and Reliability

## Distributed failure is normal

In a microservices system, failure can happen at every boundary:

```text
client -> gateway -> discovery -> service -> database
                         |
                         -> broker -> consumer -> email provider
```

The correct response is not to retry everything. It is to define a bounded failure budget and choose a behavior for each operation:

- fail fast;
- retry transient failures;
- open a circuit;
- return a degraded response;
- queue for later;
- reject excess load;
- compensate a previous business action.

## Timeout first

A retry without a timeout can multiply work while the original request is still running. Set:

1. connection timeout;
2. response/read timeout;
3. overall request deadline;
4. database query timeout;
5. broker publish/consume timeout.

The downstream budget must be shorter than the upstream budget. If the gateway has a 2-second deadline, an internal client should not wait 5 seconds.

## Circuit breaker

A circuit breaker has three useful states:

```text
CLOSED      calls flow; failures are measured
OPEN        calls fail fast; dependency is protected
HALF_OPEN   a small probe set tests recovery
```

The Order configuration uses a count-based sliding window:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      productService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        automaticTransitionFromOpenToHalfOpenEnabled: true
        slidingWindowType: count_based
```

The Gateway uses the same conceptual policy for its `ecomBreaker` route and forwards product failures to `/fallback/products`.

Choose thresholds from observed latency/error data. A circuit breaker is not a replacement for fixing an unhealthy dependency.

## Retry

The current Cart Service uses:

```java
@Retry(name = "retryBreaker", fallbackMethod = "addToCartFallBack")
public boolean addToCart(String userId, CartItemRequest request) {
    ProductResponse product = productServiceClient
            .getProductDetails(request.getProductId());
    // validate and persist cart item
}

public boolean addToCartFallBack(
        String userId,
        CartItemRequest request,
        Exception exception) {
    exception.printStackTrace();
    return false;
}
```

The configured policy is five attempts with a five-second wait. This is a useful demonstration, but adding a cart item is a write and can create duplicate side effects if the remote call or persistence operation is not idempotent.

Production retry rules:

- retry only transient errors such as connection reset or `503`;
- do not retry validation errors, `401`, or `404`;
- use exponential backoff and jitter;
- cap attempts and total elapsed time;
- retry reads more readily than writes;
- use an idempotency key for retried writes;
- log the final failure with dependency and attempt count.

## Rate limiting and load shedding

Rate limiting protects a service from a caller consuming all capacity. The gateway uses Redis-backed token-bucket settings of 10 replenished tokens per second and a burst of 20.

Resilience4j also configures a rate limiter for `/message`:

```yaml
ratelimiter:
  instances:
    rateBreaker:
      timeout-duration: 0
      limit-refresh-period: 4s
      limit-for-period: 2
```

With `timeout-duration: 0`, a request that cannot obtain permission immediately fails rather than waiting and building a queue. That is a form of load shedding.

Use separate budgets per tenant, user, or endpoint. Return `429 Too Many Requests` with a retry hint when the caller can recover.

## Bulkheads

A bulkhead isolates resources:

- separate thread pools for slow dependencies;
- bounded HTTP connection pools;
- database connection limits;
- separate Kafka consumer concurrency;
- per-tenant quotas.

Without a bulkhead, a slow Product Service can consume every Order Service worker and make user/profile operations fail too.

## Fallback design

A fallback is valid only when the degraded answer is correct:

```text
catalog unavailable -> 503 or cached catalog marked stale
recommendations down -> omit recommendations
payment unavailable -> keep order pending; never mark paid
notification down -> order remains committed; retry notification
```

Do not turn an exception into `200 OK` with an empty result if that could be interpreted as “there are no products.”

## Idempotency

For checkout:

```text
client sends Idempotency-Key: checkout-abc
Order Service stores (userId, idempotencyKey, response)
repeat request returns the stored response
```

Make the key unique per user or tenant, expire old keys safely, and bind the stored result to the original request fingerprint. For event consumers, use `eventId` or a natural business key with a unique database constraint.

## Saga for order and payment

The payment extension can be choreographed:

```text
OrderCreated/PendingPayment
  -> PaymentAuthorized -> OrderConfirmed
  -> PaymentFailed     -> OrderPaymentFailed
```

Orchestrated:

```text
CheckoutSaga
  -> reserve stock
  -> authorize payment
  -> confirm order
  -> compensate stock if payment fails
```

Every step needs a timeout, retry policy, and compensating action. A compensation is a new business action, not a database rollback.

## Reliability checklist

- Define SLOs for gateway latency, checkout success, and event lag.
- Measure dependency error rates separately from caller errors.
- Set timeouts before adding retries.
- Add jitter to avoid synchronized retry storms.
- Use circuit breakers and bulkheads around remote calls.
- Make writes idempotent.
- Persist events through an outbox where correctness matters.
- Test broker, database, and dependency outages.
- Alert on DLQs, circuit-open duration, and consumer lag.

## Interview questions

**What is retry amplification?**  
If each of several upstream layers retries a failing request, one original request can become many downstream attempts. Set one owner for retries, cap total attempts, and propagate a deadline.

**When should a circuit breaker open?**  
When failure or slow-call rates exceed a threshold over enough samples that continuing calls would harm the dependency or caller. It should be based on measured behavior, not a random number.

**What is the difference between a circuit breaker and a timeout?**  
A timeout limits one call's waiting time. A circuit breaker uses a history of calls to stop sending future traffic to a failing dependency.
