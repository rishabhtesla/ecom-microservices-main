# 21. Observability, Diagnostics, and Chaos Engineering

Observability asks whether internal state can be inferred from outputs. Monitoring asks whether known signals are within acceptable bounds. Both are needed for production operations.

## 1. Structured logs

Prefer one JSON event per meaningful action:

```json
{
  "timestamp": "2025-01-15T10:12:34Z",
  "level": "INFO",
  "service": "order-service",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "event": "order.created",
  "orderId": "42",
  "durationMs": 83
}
```

Never put passwords, access tokens, card numbers, or full PII in logs. Correlation IDs are not credentials.

The local stack includes Loki and Alloy configuration under [`deploy/docker`](../../deploy/docker). The service log format and collector pipeline must agree on field names.

## 2. Metrics

### RED and USE

For request endpoints:

- **Rate:** requests per second;
- **Errors:** failed request rate;
- **Duration:** latency distribution, especially p50/p95/p99.

For resources:

- **Utilization:** how busy it is;
- **Saturation:** queue/backlog pressure;
- **Errors:** resource-specific failures.

Do not use only averages. A p99 checkout latency alert can reveal a problem hidden by a healthy average.

Spring Boot exposes Micrometer metrics and Prometheus endpoints. Typical names should include low-cardinality labels:

```text
http.server.requests{method,uri,status}
order_checkout_total{outcome}
```

Never label metrics by raw `userId`, `orderId`, or arbitrary URL values; cardinality can exhaust the metrics backend.

## 3. Tracing

```text
trace: checkout request
  span: gateway
    span: order controller
      span: product HTTP call
      span: user HTTP call
      span: database query
      span: Kafka publish
```

W3C Trace Context uses `traceparent` and optional `tracestate`. Propagate context across HTTP and messaging. A new trace should not be created for every downstream call.

The Order client has custom propagation in [`RestClientConfig`](../../order/src/main/java/com/ecommerce/order/clients/RestClientConfig.java). When replacing custom propagation with OpenTelemetry, verify that headers, baggage, sampling, and log MDC remain consistent.

## 4. Health and diagnostics

| Signal | Question | Failure action |
|---|---|---|
| Liveness | Is the process irrecoverably stuck? | Restart |
| Readiness | Should traffic be sent now? | Remove from routing |
| Startup | Has slow initialization completed? | Delay liveness/readiness |

Expose only safe management endpoints. A heap dump or thread dump can contain secrets and user data and should not be public.

## 5. SLOs and alert design

An SLI measures behavior, such as successful checkout requests. An SLO is the target, such as 99.9% availability over 30 days. An error budget is the allowed failure budget:

```text
monthly budget = total eligible minutes * (1 - SLO)
```

Alert on user impact and fast-burn budget consumption, not every log line.

## 6. Synthetic monitoring

A synthetic checkout should use a test account, non-production payment provider, and data cleanup:

```text
login -> browse product -> add to cart -> checkout test item
```

Mark synthetic traffic so it does not distort business metrics.

## 7. Chaos engineering

Chaos is a hypothesis-driven experiment:

```text
Hypothesis: checkout remains available when notification is down.
Steady state: checkout success rate and latency SLO.
Failure: stop notification consumer for 10 minutes.
Guardrail: no payment duplicate and error budget limit.
Rollback: restore consumer.
```

Progress from local fault injection to a controlled staging experiment. Test latency, packet loss, dependency refusal, broker lag, expired certificates, and exhausted database pools. Do not start with destructive production experiments.

## Interview Q&A

**What is the difference between logs, metrics, and traces?**
Logs provide event detail, metrics provide cheap aggregated trends/alerts, and traces show causal request paths. A trace does not replace business audit logs.

**Why is high-cardinality data dangerous in metrics?**
Each label combination creates a time series. IDs can create unbounded series, causing memory, storage, and query-cost failures.

**What makes a chaos experiment safe?**
A measurable steady state, a bounded fault, explicit guardrails, rollback, and a hypothesis that changes engineering decisions.
