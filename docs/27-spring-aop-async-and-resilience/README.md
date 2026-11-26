# 27. Spring AOP, Asynchronous Work, and Resilience

## 1. AOP vocabulary

```text
aspect:     cross-cutting module
join point: method execution (in Spring proxy AOP)
pointcut:   which join points match
advice:     behavior before/after/around a match
```

```java
@Aspect
@Component
class TimingAspect {
    @Around("@annotation(Monitored)")
    Object time(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.nanoTime();
        try {
            return joinPoint.proceed();
        } finally {
            record(joinPoint.getSignature().toShortString(),
                    System.nanoTime() - start);
        }
    }
}
```

Spring AOP is proxy-based, so private methods, final methods/classes (for some proxy strategies), and internal calls do not behave like external proxy calls. Keep pointcuts narrow and avoid advice that changes business semantics invisibly.

## 2. `@Async` and context propagation

```java
@Configuration
@EnableAsync
class AsyncConfig {
    @Bean
    TaskExecutor orderExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("order-");
        executor.initialize();
        return executor;
    }
}
```

```java
@Async("orderExecutor")
public CompletableFuture<Void> publishAudit(Order order) {
    // bounded asynchronous work
    return CompletableFuture.completedFuture(null);
}
```

Always bound queues and handle rejected work. MDC, security context, and tracing context do not automatically follow every executor. A `TaskDecorator` can capture and restore them, then clear them in `finally` to avoid thread-local leakage.

Never use `@Async` to pretend a required transaction is still active. Load the data needed by the async task and define its own transaction/idempotency boundary.

## 3. Scheduling and distributed locks

```java
@Scheduled(fixedDelayString = "${outbox.poll-delay}")
public void publishOutbox() { ... }
```

Every replica runs a scheduled method unless coordination is added. ShedLock or a platform scheduler can ensure one owner:

```text
schedule fires on replica A and B
 -> distributed lock
 -> only lock holder executes
```

The lock store, lease duration, clock behavior, and crash recovery must be tested. For high-value workflows, a durable queue is often safer than an in-process scheduler.

## 4. Resilience4j composition

```java
@CircuitBreaker(name = "product")
@Retry(name = "product")
@Bulkhead(name = "product", type = Bulkhead.Type.SEMAPHORE)
public ProductResponse product(String id) { ... }
```

Annotation order matters. Configure instances explicitly:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      product:
        sliding-window-size: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
  retry:
    instances:
      product:
        max-attempts: 2
        wait-duration: 100ms
```

Do not retry after a business response such as `409`, and do not place a fallback that turns an authorization failure into success.

Spring Framework/Spring Boot versions may provide native retry or concurrency annotations in newer releases. Check the exact dependency version before copying version-specific examples; this repository currently targets Spring Boot 3.4.x.

## 5. Reactive and asynchronous failure

For `Mono`/`Flux`, use operators that preserve a bounded timeout and map errors explicitly:

```java
return client.get()
        .retrieve()
        .bodyToMono(ProductResponse.class)
        .timeout(Duration.ofSeconds(2))
        .retryWhen(Retry.backoff(1, Duration.ofMillis(100))
                .filter(this::isTransient));
```

Avoid blocking inside the reactive chain. A retry of a cold publisher can repeat side effects; make the operation idempotent.

## Interview Q&A

**Why does `@Async` sometimes appear not to work?**
A self-invocation bypasses the proxy, and the method may be called before the async proxy is applied. Call through another Spring bean and configure the executor explicitly.

**Why is an unbounded async queue dangerous?**
It converts overload into unbounded memory growth and increasing latency. Bounded queues make saturation visible and allow rejection/load shedding.

**Should circuit breakers and retries always be combined?**
No. Retries can amplify failure; a breaker may be enough for a non-idempotent or latency-sensitive call. Use measured policy per dependency.
