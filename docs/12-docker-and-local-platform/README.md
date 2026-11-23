# 12. Docker and the Local Platform

## Why containerize each service?

A container packages an application with a predictable runtime and process boundary. It does not make an application stateless or production-ready by itself. A container still needs:

- configuration injected at runtime;
- a health/readiness contract;
- resource limits;
- logs sent to stdout or a supported collector;
- graceful shutdown;
- a non-root user where possible;
- a versioned image and supply-chain scan.

## Compose topology

The main file is [`deploy/docker/docker-compose.yml`](../../deploy/docker/docker-compose.yml). It creates:

```text
backend network:
  postgres, pgadmin, mongo, zookeeper, kafka, rabbitmq,
  keycloak, zipkin, config-server, eureka,
  gateway-service, user-service, product-service,
  order-service, notification-service

loki network:
  read, write, backend, loki-gateway, alloy, minio,
  prometheus, grafana, flog
```

Within the Docker network, a service talks to `postgres`, `mongo`, `kafka`, `eureka`, or `config-server` by service name. `localhost` inside a container means that same container, not the host and not another Compose service.

## Image build strategies

The repository demonstrates three strategies:

| Strategy | Script | Notes |
|---|---|---|
| Package JARs | [`build-projects.sh`](../../deploy/docker/build-projects.sh) | Compile/package each independent Maven module |
| Cloud Native Buildpacks | [`build-images-buildpacks.sh`](../../deploy/docker/build-images-buildpacks.sh) | Spring Boot creates an OCI image |
| Jib | [`build-images-jib.sh`](../../deploy/docker/build-images-jib.sh) | Builds container layers without a handwritten Dockerfile |

The Maven POMs configure Jib images such as `dcode007/product-service`. `jib:build` normally pushes to a registry; use a local Jib goal or configure registry credentials intentionally for local work.

The checked-in [`notification/Dockerfile`](../../notification/Dockerfile) is a simple example:

```dockerfile
FROM eclipse-temurin:23-jdk-ubi9-minimal
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8888
ENTRYPOINT ["java", "-jar", "app.jar"]
```

It should be treated as a learning artifact: the project targets Java 21, the service configuration uses port `8084`, and the Dockerfile exposes `8888`. Image runtime version and port metadata should be aligned before using it as a deployment contract.

## Local startup

The Compose file references published images and environment variables. A safe preflight is:

```bash
cd deploy/docker
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs -f gateway-service
```

For local builds, first build images with names matching the Compose file, or change Compose to use local `build:` contexts. Do not silently mix a newly built product service with an old gateway image.

## Environment configuration

The main Compose services consume values such as:

| Variable | Consumer | Example local value |
|---|---|---|
| `DB_USER` | Product, Order | `embarkx` |
| `DB_PASSWORD` | Product, Order | local-only password |
| `MONGO_URI` | User | `mongodb://mongo:27017/ecom_user` |
| `RABBITMQ_HOST` | Config Server | `rabbitmq` |
| `RABBITMQ_PORT` | Config Server | `5672` |
| `RABBITMQ_USERNAME` / `RABBITMQ_PASSWORD` | Config Server | local-only credentials |
| `RABBITMQ_VHOST` | Config Server | `/` or configured vhost |
| `ZIPKIN_URL` | Gateway | `http://zipkin:9411/api/v2/spans` |

Use a local untracked `.env` file or a secret manager. Never commit real passwords, tokens, private keys, or registry credentials.

## Known Compose gaps

Before calling the stack “working,” resolve these checked-in issues:

1. `init-multi-db.sql` creates `productdb` and `orderdb`; the Docker-specific Config Server files use those names, while the default local files use `product` and `order`. Verify the active profile.
2. Gateway config enables a Redis rate limiter, but Redis is not a Compose service.
3. The Compose file uses published `dcode007/*` images rather than `build:` contexts.
4. `depends_on` controls startup order, not application readiness. Health checks and retrying startup are still required.
5. The notification Dockerfile exposes `8888` while its application config uses `8084`.
6. The Config Server healthcheck probes `8761`, although Config Server listens on `8888`.
7. Development passwords, anonymous Grafana administration, broad Actuator exposure, and `start-dev` Keycloak are not production settings.
8. Host-mounted log paths and Docker socket access give Alloy powerful local permissions.

## Container health

A healthcheck should test the application endpoint, not merely that a process exists:

```yaml
healthcheck:
  test: ["CMD", "wget", "--spider",
         "http://localhost:8081/actuator/health"]
  interval: 10s
  timeout: 5s
  retries: 5
```

Use separate liveness and readiness semantics:

- **Liveness:** restart only when the process cannot recover.
- **Readiness:** remove from traffic while dependencies/configuration are unavailable.

Do not make liveness depend on every downstream service or a temporary broker outage; that can cause restart storms.

## Production image checklist

- Pin base image digests or trusted versions.
- Use a supported JRE image and Java version.
- Run as a non-root user.
- Copy only the built artifact and required runtime files.
- Generate SBOM and scan the image.
- Do not bake environment secrets into layers.
- Handle SIGTERM and graceful shutdown.
- Set JVM/container memory-aware options.
- Define a read-only filesystem where compatible.
- Publish immutable version tags.

## Interview questions

**Why does `localhost` fail in Compose?**  
Each container has its own network namespace. Use the Compose service name and the container port, such as `http://postgres:5432` or `http://eureka:8761`.

**What is the difference between a Docker image and a container?**  
An image is an immutable package/layer template. A container is a running instance with a writable runtime layer and process/network configuration.

**Does `depends_on` mean the dependency is ready?**  
No. It starts containers in order. Application-level healthchecks, retries, and readiness gates are still required.
