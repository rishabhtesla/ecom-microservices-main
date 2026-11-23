# 13. Kubernetes Deployment

## Current repository status

This checkout contains Docker Compose deployment material but no tracked Kubernetes Deployment, Service, Ingress, Helm, or Kustomize manifests. The material in this chapter is a deployment blueprint: it teaches the concepts and gives a safe starting manifest without pretending that Kubernetes deployment has already been validated here.

## Map the Compose model to Kubernetes

| Compose concept | Kubernetes concept |
|---|---|
| Container | Pod/container |
| Service name and bridge DNS | `Service` and cluster DNS |
| Environment variables | `ConfigMap` and `Secret` |
| `restart` | Deployment controller and kubelet |
| `depends_on` | Readiness probes, retries, and controllers |
| Port mapping | Service port/targetPort and Ingress |
| Volume | PersistentVolumeClaim |
| Multiple replicas | Deployment `replicas` |
| Docker healthcheck | Liveness/readiness/startup probes |

Use managed Kafka, PostgreSQL, MongoDB, Keycloak, and observability services where possible. Running every stateful dependency inside the same Kubernetes cluster is a separate reliability project.

## Workload types

- **Deployment:** stateless Gateway, Product, User, Order, Notification, Config Server, and Eureka processes.
- **Service:** stable virtual IP and DNS for a Deployment.
- **Ingress/Gateway API:** external HTTP entry point and TLS.
- **ConfigMap:** non-secret configuration.
- **Secret:** credentials and tokens, ideally backed by an external secret manager.
- **HorizontalPodAutoscaler:** change replica count based on measured resource or custom metrics.
- **StatefulSet:** stateful infrastructure only when managed alternatives are not available.
- **Job/CronJob:** migrations, cleanup, or scheduled reconciliation.

## A product deployment blueprint

The following is intentionally generic. Replace the image, namespace, registry, secret names, and resource values after measuring the service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  labels:
    app.kubernetes.io/name: product-service
    app.kubernetes.io/component: application
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: product-service
  template:
    metadata:
      labels:
        app.kubernetes.io/name: product-service
    spec:
      containers:
        - name: product-service
          image: registry.example.com/ecommerce/product-service:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8081
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: product-database
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: product-database
                  key: password
            - name: SPRING_PROFILES_ACTIVE
              value: kubernetes
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: http
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http
            periodSeconds: 20
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /actuator/health
              port: http
            periodSeconds: 5
            failureThreshold: 30
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 768Mi
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: product-service
  ports:
    - name: http
      port: 8081
      targetPort: http
```

Other services follow the same shape with their own image, port, configuration, and database secret. The Kubernetes Service DNS name is `product-service`, so a client can use:

```text
http://product-service:8081/api/products/42
```

If the application still has Eureka enabled, decide whether it registers with Eureka in the cluster or whether Kubernetes DNS replaces discovery. Running both without a clear policy increases operational complexity.

## Configuration and secrets

Non-secret values:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  SPRING_PROFILES_ACTIVE: kubernetes
  SPRING_CLOUD_CONFIG_URI: http://config-server:8888
  KAFKA_BOOTSTRAP_SERVERS: kafka:9092
```

Credentials should come from a Secret or external secret operator:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-database
type: Opaque
stringData:
  username: replace-me
  password: replace-me
```

Do not commit real values. Native Kubernetes Secrets are base64-encoded, not automatically encrypted for every access path. Use encryption at rest, RBAC, rotation, and a managed secret store.

## Probes and graceful rollout

Spring Boot should expose readiness/liveness groups explicitly. A readiness failure removes a pod from service traffic; it should not necessarily restart the process. A liveness failure restarts a process that cannot recover. A startup probe protects slow JVM startup from premature liveness restarts.

Rolling updates should:

1. start a new pod;
2. wait for readiness;
3. shift traffic;
4. terminate the old pod gracefully;
5. stop if the new revision fails readiness.

Add a `PodDisruptionBudget` for critical services and set a termination grace period long enough to finish in-flight requests.

## Gateway exposure

Expose only the Gateway through an Ingress or Gateway API:

```text
Internet -> TLS Ingress -> gateway-service:8080
                         -> ClusterIP services
```

Keep Product, User, Order, Config Server, Eureka, databases, and brokers internal unless an explicit operational requirement exists.

## Scaling and resource design

Scale on a business signal when possible:

- gateway requests per second;
- order request latency or queue depth;
- Kafka consumer lag for Notification;
- database connection saturation.

CPU-only HPA can miss a queue backlog. Define minimum/maximum replicas, stabilization windows, and a scale-down policy. Capacity-plan databases and brokers separately; scaling application pods cannot fix a saturated database.

## Network and identity

- Use namespaces and RBAC per environment.
- Apply NetworkPolicies: gateway can reach business services; only required services can reach databases.
- Use workload identity for cloud resources.
- Encrypt ingress and service traffic where threat model requires it.
- Restrict egress to payment providers and identity infrastructure.
- Use image pull credentials and signed images.

## Helm and Kustomize

Helm is useful for reusable parameterized charts. Kustomize is useful for base plus environment overlays:

```text
deploy/k8s/
  base/
    product-deployment.yaml
    product-service.yaml
  overlays/
    development/
    staging/
    production/
```

Keep environment differences in values/overlays, not copied manifests. Validate with `kubectl apply --dry-run=server`, schema validation, policy checks, and a disposable namespace.

## Deployment sequence

```bash
kubectl create namespace ecommerce
kubectl apply -n ecommerce -f config/
kubectl apply -n ecommerce -f secrets/       # from a secure source
kubectl apply -n ecommerce -f services/
kubectl rollout status deployment/product-service -n ecommerce
kubectl get pods,svc -n ecommerce
kubectl port-forward svc/gateway-service 8080:8080 -n ecommerce
```

Do not encode startup order with shell sleeps. Let readiness, retries, and controller reconciliation handle dependencies.

## Interview questions

**Deployment versus Service?**  
Deployment manages replicated application pods and rolling updates. Service gives those pods a stable virtual IP and DNS name.

**Readiness versus liveness?**  
Readiness controls traffic eligibility. Liveness controls whether the process should be restarted. Mixing them causes either bad traffic or restart loops.

**Would you run Kafka in the same cluster?**  
Only with a team prepared to operate storage, replication, upgrades, capacity, and recovery. A managed Kafka service is often the safer production choice.
