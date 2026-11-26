# 20. Resilience, Fault Tolerance, and Caching

The goal of resilience is not to hide every failure. It is to preserve important behavior, fail quickly when recovery is impossible, and make overload visible.

## 1. Timeouts first

Every remote call needs a deadline:

```text
connect timeout: time to establish a connection
read/response timeout: time waiting for bytes
overall deadline: maximum time for the complete operation
```

Without a timeout, a finite thread pool can be exhausted by a slow dependency. A downstream timeout should become a bounded, observable error rather than an indefinitely hanging request.

The Order HTTP client is configured in [`RestClientConfig`](../../order/src/main/java/com/ecommerce/order/clients/RestClientConfig.java). Configure connect, response, and pool-acquisition timeouts together; one missing timeout can still create a stuck call.

## 2. Circuit breaker states

```text
Closed  --failure threshold--> Open
Open    --wait duration-------> Half-open
Half-open --healthy probes----> Closed
Half-open --failed probe------> Open
```

The Gateway applies a circuit breaker and fallback around the Product route. A circuit breaker should:

- count only failures that indicate dependency trouble;
- use a rolling window large enough to avoid reacting to one request;
- expose state-transition metrics;
- have a fallback that is honest about stale/missing data;
- avoid retrying a request after the circuit has already rejected it.

Fallbacks are not always cached success. A product lookup may return a stale cache, while a payment authorization should usually fail closed.

## 3. Retry with backoff and jitter

```java
Duration delay = base.multipliedBy(1L << attempt);
Duration jittered = delay.plus(random.nextLong(0, 250));
```

Retry only transient failures and only idempotent operations, or attach an idempotency key. Bound attempts and total time. A retry budget limits the percentage of traffic spent on retries so an outage does not multiply load.

Bad retry stack:

```text
client: 3 attempts x gateway: 3 x mesh: 3 = 27 downstream calls
```

Assign retry ownership and propagate a deadline.

## 4. Bulkheads, rate limits, and backpressure

### Bulkhead

- thread-pool isolation: work has a dedicated executor and queue;
- semaphore isolation: limits concurrent calls with less context switching.

Keep checkout capacity separate from low-priority catalog enrichment. One dependency must not consume every worker.

### Rate limiting

| Algorithm | Behavior |
|---|---|
| Fixed window | Simple, boundary bursts |
| Sliding window | More accurate, more state |
| Token bucket | Allows bounded bursts and sustained rate |
| Leaky bucket | Smooths output at a fixed rate |

The Gateway uses a Redis-backed request-rate limiter in [`GatewayConfig`](../../gateway/src/main/java/com/ecommerce/gateway/GatewayConfig.java). Redis itself must be deployed and monitored; the main Compose file does not currently define it.

### Backpressure and load shedding

Reactive streams propagate demand. Queues should have explicit maximum size, age limits, and dead-letter behavior. Shed optional work first (recommendations, analytics) and preserve checkout/authentication capacity.

## 5. Caching patterns

```text
cache-aside:
  read cache -> miss -> database -> populate cache

read-through:
  application asks cache; cache loader reads database

write-through:
  write cache and backing store together

write-behind:
  cache acknowledges, later flushes backing store
```

Cache-aside is easy to add but has stale-read and stampede risks. Write-behind is dangerous for authoritative order/payment state unless durability and replay are designed.

### Cache hazards

- **stampede/thundering herd:** many requests miss together;
- **penetration:** repeated requests for nonexistent keys;
- **avalanche:** many keys expire together;
- **invalidation race:** stale response overwrites fresh cache value.

Mitigations include jittered TTLs, negative caching, request coalescing, stale-while-revalidate, versioned values, and bounded cache capacity.

## 6. Redis example

**[Blueprint]**

```java
@Cacheable(cacheNames = "products", key = "#id", unless = "#result == null")
public ProductResponse find(String id) {
    return productRepository.findById(id)
            .map(mapper::toResponse)
            .orElse(null);
}

@CacheEvict(cacheNames = "products", key = "#id")
public void update(String id, UpdateProduct command) {
    // update authoritative database
}
```

Do not cache authorization decisions longer than the security policy allows. Include tenant and locale in keys where relevant.

## Interview Q&A

**Why are timeouts more fundamental than circuit breakers?**
A breaker cannot protect a pool from calls that never finish unless the call has a timeout. The timeout bounds resource occupation; the breaker prevents repeatedly starting known-bad calls.

**Should every failure be retried?**
No. Validation errors, authorization errors, and deterministic business conflicts should not be retried. Retry transient transport failures only when the operation is safe or idempotent.

**Why can a cache return incorrect data even after invalidation?**
Concurrent reads/writes, replication lag, delayed invalidation messages, and an older request finishing after a newer write can all reintroduce stale values.
