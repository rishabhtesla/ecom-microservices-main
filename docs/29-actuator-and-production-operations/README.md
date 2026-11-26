# 29. Actuator, Micrometer, and Production Operations

## 1. Actuator endpoints

Useful endpoints include:

```text
/actuator/health
/actuator/health/liveness
/actuator/health/readiness
/actuator/metrics
/actuator/prometheus
/actuator/info
/actuator/threaddump
/actuator/heapdump
```

Expose only what operations needs:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true
```

Protect diagnostics with a separate management port/network and strong authentication. Heap/thread dumps can reveal secrets and customer data.

## 2. Custom health indicators

```java
@Component
class PaymentProviderHealth implements HealthIndicator {
    @Override
    public Health health() {
        if (provider.isReachable()) {
            return Health.up().withDetail("provider", "reachable").build();
        }
        return Health.down()
                .withDetail("provider", "unreachable")
                .build();
    }
}
```

Do not make liveness depend on every remote dependency. A provider outage should normally make a payment capability unavailable or not-ready according to the service contract, not cause every pod to restart.

## 3. Micrometer Observation

Metrics should have stable names and low-cardinality tags:

```java
Timer.Sample sample = Timer.start(registry);
try {
    return checkout();
} finally {
    sample.stop(Timer.builder("checkout.duration")
            .tag("outcome", outcome)
            .register(registry));
}
```

The Observation API can create metrics and tracing around the same operation. Prefer framework instrumentation for HTTP/database calls and add business observations at meaningful boundaries.

## 4. Tracing and MDC

```text
incoming traceparent
 -> server span
 -> child client/database/message spans
 -> W3C context on outbound request
```

MDC correlation IDs help logs but are thread-local. Ensure async executors and reactive context carry them, and clear them after work. Never use a caller-provided correlation ID as an authorization identity.

The local observability stack includes Prometheus, Grafana, Zipkin, Loki, and Alloy definitions under [`deploy/docker`](../../deploy/docker). The Compose Prometheus configuration uses `host.docker.internal` targets; verify reachability from the Prometheus container rather than assuming host networking.

## 5. Startup optimization

### Lazy initialization

```yaml
spring:
  main:
    lazy-initialization: true
```

This can reduce startup time but moves bean failures to first use. Pair it with smoke tests and explicit critical-bean validation.

### AOT/native image/CDS

Spring AOT generates runtime hints and optimized initialization paths. GraalVM native images reduce startup/memory for some workloads but require reflection/resource/proxy configuration and can increase build complexity. Class Data Sharing improves class loading for supported JVM deployments.

Layered JARs and Buildpacks/Jib make dependency layers reusable:

```text
dependencies layer
spring-loader layer
application classes layer
```

Do not optimize startup before measuring cold start, warm latency, image size, and operational cost.

## 6. Graceful shutdown

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s
```

Graceful shutdown requires platform coordination:

```text
mark not-ready
stop new requests/consumption
drain in-flight work
commit/ack bounded messages
close clients and pools
terminate
```

Kubernetes `terminationGracePeriodSeconds` must exceed the application drain timeout. `preStop` sleeps are only a routing propagation buffer.

## 7. Null safety and modern Java/Spring

JSpecify annotations such as `@NullMarked` and `@Nullable` communicate null contracts to tools, especially across library boundaries. They do not replace runtime validation. Verify support in the exact Spring/Boot/compiler versions before adopting a repository-wide policy; this project currently targets Boot 3.4.x.

## Interview Q&A

**Why should readiness fail before liveness during shutdown?**
Readiness stops new traffic while allowing in-flight work to drain. Liveness restarting immediately would interrupt that work.

**What is the value of an Observation versus a metric?**
An Observation represents one operation and can produce correlated metrics, traces, and logs; a metric is an aggregate measurement.

**Why can lazy initialization hide deployment failures?**
The application can report “started” before an invalid bean or missing secret is used. A startup smoke test must exercise critical paths.
