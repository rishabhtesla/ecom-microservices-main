# 23. Containerization, Kubernetes, Releases, and CI/CD

## 1. Container image basics

A production image should be reproducible, minimal, non-root, and observable:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/order-*.jar app.jar
USER 10001
EXPOSE 8083
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Use a multi-stage build when Maven compilation belongs in the image build:

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /src
COPY pom.xml .
COPY src src
RUN ./mvnw -q -DskipTests package

FROM eclipse-temurin:21-jre
COPY --from=build /src/target/*.jar /app/app.jar
USER 10001
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Pin a base image digest in a controlled pipeline, generate an SBOM, scan dependencies and images, and avoid copying `.env`, Git metadata, or credentials. The repository has Docker Compose but no tracked Kubernetes manifests or CI workflows.

## 2. Kubernetes primitives

```text
Deployment -> ReplicaSets -> Pods
Service    -> stable DNS and virtual IP
Ingress/Gateway -> external routing
ConfigMap  -> non-secret configuration
Secret     -> secret material (still protect RBAC/encryption)
HPA        -> CPU/memory/custom metric scaling
KEDA       -> event/queue-driven scaling
```

### Deployment blueprint

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 2
  selector:
    matchLabels: {app: order-service}
  template:
    metadata:
      labels: {app: order-service}
    spec:
      containers:
        - name: order
          image: registry.example/order:sha-abc123
          ports:
            - containerPort: 8083
          readinessProbe:
            httpGet: {path: /actuator/health/readiness, port: 8083}
          livenessProbe:
            httpGet: {path: /actuator/health/liveness, port: 8083}
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 15"]
          resources:
            requests: {cpu: "250m", memory: "512Mi"}
            limits: {cpu: "1", memory: "1Gi"}
```

`preStop` alone is not graceful shutdown. The application must stop accepting work, drain connections, finish bounded in-flight work, and close consumers.

## 3. Zero-downtime database changes

Use expand-and-contract:

```text
1. add nullable/new column
2. deploy code that writes both
3. backfill in batches
4. switch reads to new column
5. stop old writes
6. enforce constraint
7. remove old column later
```

Flyway and Liquibase migrations must be ordered, observable, and safe when a deployment rolls back. Never make an application version depend on a destructive migration that an older version cannot tolerate during a rolling update.

## 4. Release strategies

| Strategy | Strength | Main risk |
|---|---|---|
| Rolling | Simple, resource efficient | Old/new versions coexist |
| Blue-green | Fast rollback, isolated environments | Double capacity and data compatibility |
| Canary | Small blast radius, measured | Routing/metrics complexity |
| Shadow | Test real traffic without user response | Side effects must be disabled |
| Feature flag | Decouple deploy from release | Flag lifecycle and hidden paths |

Canary decisions need automated health checks: error rate, p95 latency, saturation, business success, and dependency behavior.

## 5. KEDA and event scaling

**[Blueprint]**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: notification-scaler
spec:
  scaleTargetRef:
    name: notification
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: notification-group
        topic: order.exchange
        lagThreshold: "100"
```

Scale on backlog only when consumers are idempotent and downstream capacity is protected.

## 6. Testing pyramid

```text
many fast unit tests
service slice tests
integration tests with Testcontainers
consumer/provider contract tests
few cross-service E2E tests
```

Testcontainers should provision real PostgreSQL/Kafka/Redis dependencies for behavior that mocks cannot verify. Contract tests catch incompatible APIs without making every CI run a full environment.

## 7. CI/CD pipeline

**[Blueprint]**

```yaml
name: build
on: [push, pull_request]
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - run: ./mvnw -B verify
      - run: ./mvnw -B test
      - run: docker build --tag "$IMAGE" .
      - run: docker scout cves "$IMAGE"
```

A real pipeline also signs artifacts, publishes an SBOM, runs contract/schema compatibility checks, scans IaC, pushes immutable image tags, deploys to a controlled environment, performs smoke tests, and supports rollback. Secrets belong in the CI secret store or workload identity, not YAML.

## Interview Q&A

**Why do rolling deployments require backward-compatible changes?**
Old and new pods serve traffic at the same time. APIs, events, and database schemas must work for both until the old version drains.

**What is the difference between readiness and liveness?**
Readiness controls routing; liveness controls restart. Confusing them can cause either traffic to unhealthy pods or restart loops during dependency outages.

**Why use immutable image tags?**
`latest` can point to different content over time. A digest or commit tag makes a deployment reproducible and rollbackable.
