# 18. Discovery, Load Balancing, and Service Mesh

## Discovery models

### Client-side discovery: Eureka

```text
client -> registry -> instance list -> client chooses instance
```

The repository uses:

```java
.uri("lb://PRODUCT-SERVICE")
```

and:

```java
@Bean
@LoadBalanced
public RestClient.Builder restClientBuilder() {
    return RestClient.builder();
}
```

Each service registers a lease and fetches the registry. A registry outage can affect new discovery while cached instances may continue to work.

### Server-side discovery: Kubernetes DNS

```text
client -> product-service.default.svc.cluster.local
       -> Kubernetes Service -> ready Product pod
```

The client does not choose an instance; the platform does. Kubernetes CoreDNS resolves the stable Service name.

### Consul and other registries

Consul combines service catalog, health checks, and key/value features. Eureka is Spring-friendly and client-side. Choose based on platform, multi-language requirements, operational ownership, and failure behavior.

## Health, heartbeat, and readiness

Registration is not readiness:

```text
process started -> registration -> dependencies ready -> readiness=true
```

Use:

- liveness to restart a stuck process;
- readiness to remove a process from routing;
- startup probe to protect slow startup.

Do not make liveness fail because Kafka is temporarily unavailable; that can cause restart storms.

## Load balancing algorithms

```text
round robin        A B C A B C
least connections  choose instance with lowest active count
weighted           A receives 70%, B 30%
consistent hash    key=order-42 maps to stable node
sticky session     same client tends to same node
```

Round robin is a default, not a proof of fairness. Long requests and uneven instance capacity need least-connections or weights. Consistent hashing is useful for cache locality and partition ownership. Avoid sticky sessions unless unavoidable; they reduce failover flexibility.

## Service mesh architecture

```text
control plane
  -> certificates, routes, policy, telemetry config

data plane
  Order app -> Order Envoy -> Product Envoy -> Product app
```

Istio and Linkerd are control-plane ecosystems; Envoy is a common proxy. Sidecar injection gives every pod a proxy, while ambient/sidecarless modes change the operational model.

### Mesh capabilities

- workload mTLS;
- service identity and authorization;
- retries/timeouts;
- circuit breaking and outlier detection;
- traffic splitting;
- canary/A-B routing;
- telemetry and access logs.

### Transparent retries warning

If the application, gateway, and mesh all retry, one request can produce many downstream attempts. Assign retry ownership per operation and propagate a deadline.

## Canary traffic

**[Blueprint]**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts: [product-service]
  http:
    - route:
        - destination:
            host: product-service
            subset: stable
          weight: 95
        - destination:
            host: product-service
            subset: canary
          weight: 5
```

The 5% route is useful only when metrics, logs, traces, and rollback automation are ready. Do not send payment side effects to a shadow target without isolating writes.

## mTLS and identity

TLS encrypts traffic; mTLS also authenticates both workloads. A mesh can issue short-lived workload certificates. SPIFFE/SPIRE provides a workload identity model independent of IP address.

## Interview Q&A

**Why might Kubernetes DNS be preferable to Eureka inside Kubernetes?**
It removes an application-level registry dependency and uses a platform primitive. Eureka remains reasonable for a multi-platform or non-Kubernetes estate.

**What does a mesh not solve?**
It does not define business idempotency, domain authorization, correct transaction boundaries, or safe retries for non-idempotent operations.
