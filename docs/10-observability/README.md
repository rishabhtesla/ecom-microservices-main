# 10. Observability: Logs, Metrics, and Traces

## The three signals

| Signal | Answers | Local component |
|---|---|---|
| Logs | What did this process observe or do? | Logback files, Loki, Grafana Alloy |
| Metrics | How often and how quickly is it happening? | Actuator, Micrometer, Prometheus, Grafana |
| Traces | Which calls formed one request? | Micrometer Tracing, Zipkin |

Observability is more than collecting data. The signal must be correlated with a service, environment, operation, status, and trace/request ID.

## Actuator and Micrometer

Product, User, and Order include Actuator, Prometheus registry, and Micrometer tracing dependencies. Their configuration exposes management endpoints and enables tracing sampling for the local demo:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
  tracing:
    sampling:
      probability: 1.0
```

Local URLs:

```text
http://localhost:8081/actuator/health
http://localhost:8081/actuator/prometheus
http://localhost:8082/actuator/prometheus
http://localhost:8083/actuator/prometheus
```

Expose only the endpoints needed by the platform in production. A public `env`, `beans`, `heapdump`, or unrestricted health-detail endpoint can leak secrets and topology.

## Prometheus scrape model

The checked-in configuration scrapes Product, User, and Order:

```yaml
scrape_configs:
  - job_name: 'product-service'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 3s
    static_configs:
      - targets: ['host.docker.internal:8081']
        labels:
          application: 'product-service'
```

This is static local discovery and assumes the application processes are reachable through `host.docker.internal`; it is not the same as scraping the Compose containers by service DNS. In Kubernetes, use service discovery or a Prometheus operator and label services consistently. A three-second interval is useful for a demo but can be expensive at scale.

Useful metrics:

- request count by route and status;
- request latency percentiles;
- downstream client latency/error count;
- circuit-breaker state and calls;
- database pool usage;
- JVM heap and GC;
- Kafka consumer lag;
- order creation success/failure;
- notification delivery success/failure.

Avoid high-cardinality labels such as raw user IDs, order IDs, or arbitrary URLs.

## Distributed tracing

Gateway and services include Micrometer tracing with a Zipkin reporter. The Docker gateway config sends spans to:

```yaml
management:
  zipkin:
    tracing:
      endpoint: ${ZIPKIN_URL:http://zipkin:9411/api/v2/spans}
```

The Order Service also injects the current propagation context into a blocking RestClient request:

```java
propagator.inject(
        tracer.currentTraceContext().context(),
        request.getHeaders(),
        (carrier, key, value) -> carrier.add(key, value));
```

The trace should look like:

```text
gateway span
  -> order HTTP server span
      -> product client/server span
      -> user client/server span
      -> database span
      -> Kafka publish span
          -> notification consume span
```

Use a standard propagation format such as W3C Trace Context. Sample intelligently in production: keep all errors and slow traces, sample normal high-volume traffic.

## Logging with Loki and Alloy

Services write rolling files such as:

```yaml
logging:
  file:
    name: logs/${spring.application.name}.log
  logback:
    rollingpolicy:
      max-file-size: 5MB
      max-history: 7
```

Grafana Alloy discovers Docker containers and tails files, forwarding them to Loki through `loki-gateway`. Grafana is provisioned with Loki as a data source in the Compose stack.

A production log should be structured:

```json
{
  "timestamp": "2026-01-01T10:00:00Z",
  "level": "INFO",
  "service": "order-service",
  "event": "order_created",
  "orderId": "42",
  "traceId": "4f3c...",
  "durationMs": 84
}
```

Never log passwords, access tokens, card data, or full sensitive user profiles. Use a stable event name and fields that are searchable.

## Debugging playbook

When checkout fails:

1. Capture the request time, route, user/tenant context, and trace ID.
2. Search the gateway log for authentication/routing failure.
3. Open the trace in Zipkin and find the slow/error span.
4. Check Order Service logs and database pool metrics.
5. Inspect Product/User downstream spans.
6. If the order committed, check Kafka publish status and consumer lag.
7. Check Notification logs and DLQ/retry metrics.

Do not start by reading every log file. Start from a trace or correlation ID and follow the dependency graph.

## Production observability checklist

- Use structured JSON logs and centralized retention policies.
- Redact secrets before shipping logs.
- Define SLOs and alert thresholds.
- Create dashboards for RED metrics: rate, errors, duration.
- Add USE metrics for resources: utilization, saturation, errors.
- Propagate trace context through HTTP and Kafka.
- Keep error and slow-request traces at a useful sampling rate.
- Monitor event lag and outbox/Dead Letter Queue size.
- Separate business metrics from infrastructure metrics.

## Interview questions

**Logs versus traces?**  
Logs are discrete records with local context. A trace connects spans from multiple processes into one causal request. A trace can point you to the right logs; neither replaces metrics.

**Why are percentiles better than averages for latency?**  
An average can hide a slow tail. p95/p99 show the experience of the slowest meaningful fraction of requests and help set timeout/SLO policies.

**What is a high-cardinality metric label?**  
A label with many unique values, such as order ID. It can explode time-series count and cost. Put such values in logs/traces instead.
