# 14. Production Architecture and Missing Course Pieces

## What “production-grade” adds

The project demonstrates the important building blocks, but production readiness is a system of controls around the code:

```text
secure identity + secret rotation
reproducible builds + signed images
safe database migrations
contract/integration tests
timeouts + idempotency + outbox
health/readiness + autoscaling
logs + metrics + traces + alerts
backups + disaster recovery
canary/rollback deployment
runbooks + ownership + SLOs
```

Microservices without these controls are distributed code, not a reliable platform.

## Reference production shape

```text
CDN/WAF
  -> managed ingress/API gateway
      -> Gateway deployment
          -> Product/User/Order deployments
              -> managed PostgreSQL/MongoDB
              -> managed Kafka
              -> Keycloak or enterprise IdP

all workloads
  -> OpenTelemetry collector
      -> metrics/logs/traces backend
  -> secret manager
  -> image registry and policy admission
```

Use private networking, least-privilege identities, and separate production data from development data.

## Coverage matrix

| Course topic | Repository status | Interview wording |
|---|---|---|
| Spring Boot REST | Implemented | Product/User/Order controllers and DTOs |
| JPA/PostgreSQL | Implemented | Product and Order persistence |
| MongoDB | Implemented | User profiles |
| API Gateway | Implemented | Spring Cloud Gateway with discovery and filters |
| Eureka | Implemented | Registry and load-balanced logical URLs |
| Config Server | Implemented | Native classpath config and Docker profile |
| Kafka/Cloud Stream | Implemented | Order event and notification consumer |
| RabbitMQ | Supporting infrastructure/demo | Config Bus and optional messaging examples |
| Keycloak/OAuth2 | Implemented/demo | JWT resource server, Admin API, auth code, PKCE, client credentials |
| Resilience4j | Implemented partially | Retry, circuit breaker, rate limiter; hardening remains |
| Zipkin/Micrometer | Implemented partially | HTTP tracing and Zipkin endpoint |
| Prometheus/Grafana/Loki | Local platform | Scraping and log collection examples |
| Docker/Compose | Implemented | Multi-container local environment |
| Kubernetes | Blueprint only | No tracked manifests yet |
| Payment service | Blueprint only | Add an idempotent provider integration and payment events |
| Outbox/Saga | Blueprint only | Required for reliable cross-service workflows |
| Schema registry/DLQ | Blueprint only | Add as event volume and compatibility requirements grow |
| CI/CD | Blueprint only | Add build, scan, test, deploy, and rollback workflows |

## Payment service design

The syllabus names a payment service, but the main codebase does not contain one. A production design should:

1. create a payment intent using a provider idempotency key;
2. persist a local payment record with `PENDING` status;
3. verify signed provider webhooks;
4. publish `PaymentAuthorized` or `PaymentFailed`;
5. never store raw card data;
6. reconcile provider and local records;
7. provide a refund flow and audit trail.

Order state should be explicit:

```text
PENDING_PAYMENT -> CONFIRMED
PENDING_PAYMENT -> PAYMENT_FAILED
CONFIRMED       -> CANCELLED
CONFIRMED       -> REFUND_PENDING -> REFUNDED
```

Avoid changing `CONFIRMED` before payment authorization merely because the order row was inserted.

## Safe database delivery

Do not rely on `hibernate.ddl-auto=update` for production schema management. Use a migration tool such as Flyway or Liquibase:

```text
V1__create_product.sql
V2__add_product_search_index.sql
V3__add_outbox_event.sql
```

Deploy backward-compatible schema changes first, then application code, then cleanup migrations. Test rollback and restore, not only forward migration.

## CI/CD pipeline

A service pipeline should:

1. check formatting and compile;
2. run unit and slice tests;
3. run integration/contract tests;
4. scan dependencies and source;
5. build a reproducible image;
6. generate an SBOM and scan the image;
7. sign and push an immutable image;
8. deploy to a disposable environment;
9. run smoke tests;
10. promote through approval/canary;
11. monitor SLOs and roll back automatically when safe.

Because the modules are independent Maven projects, a change to Product should not require rebuilding every service unless a shared platform contract changed.

## Security hardening

- Rotate all local/default credentials.
- Remove secrets from properties and Git history.
- Replace admin-password automation with a least-privilege service account.
- Validate JWT issuer, audience, expiry, signature, and role claims.
- Derive user identity from the token, not `X-User-ID`.
- Secure internal service traffic and management endpoints.
- Add dependency and container vulnerability scanning.
- Restrict egress and network access.
- Audit privileged actions and payment transitions.

## Availability and disaster recovery

Define:

- RTO: how quickly a service must recover;
- RPO: how much data loss is acceptable;
- backup frequency and retention;
- restore verification;
- region/zone strategy;
- broker replay/recovery plan;
- database failover;
- runbook ownership.

An application with three replicas but one database instance is not three-way available.

## Version and framework reality check

The primary modules currently declare Spring Boot `3.4.x`, Spring Cloud `2024.0.x`, and Java `21` in their Maven POMs. The additional inter-service example declares Java `24`. If the course has moved its teaching material to Spring Framework 7/Spring Boot 4, treat that as a separate upgrade task; do not describe this checkout as Boot 4 until its POMs, APIs, tests, and deployment images have actually been migrated.

Framework upgrades should be handled incrementally:

1. align the Java runtime and build plugins;
2. align the Spring Boot and Spring Cloud release train;
3. resolve namespace/API/deprecation changes;
4. run module and integration tests;
5. rebuild images and verify runtime probes;
6. update these notes and the service catalog.

## Recommended implementation roadmap

### Phase 1: make local behavior deterministic

- reconcile database names;
- add Redis or disable the gateway limiter;
- align Dockerfile port and Java version;
- add `.env.example` without secrets;
- add readiness checks and startup retries.

### Phase 2: make business workflows reliable

- add idempotent checkout;
- add transactional outbox and inbox/deduplication;
- add Kafka retry/DLQ;
- introduce payment state machine and provider webhook handling;
- add contract and Testcontainers tests.

### Phase 3: make the platform deployable

- add Kubernetes base/overlays or a Helm chart;
- add secret-manager integration and network policies;
- add image scanning/signing;
- add CI/CD with canary and rollback;
- define SLO dashboards and runbooks.

## Interview questions

**What is the first thing you would fix?**  
I would make the local deployment deterministic and secure: reconcile database names, remove default secrets, add health/readiness behavior, and ensure the gateway's Redis dependency exists. Then I would add an outbox and idempotent checkout before adding more services.

**How do you evolve a microservice without downtime?**  
Use backward-compatible API/event changes, expand-migrate-contract database changes, contract tests, rolling/canary deployment, and observable rollback criteria.

**What is the difference between high availability and resilience?**  
High availability keeps capacity running through component failure. Resilience includes absorbing, degrading, recovering from, and learning from failures across the whole system.
