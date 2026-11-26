# Microservices Learning Guide

This directory is the interview revision guide for the e-commerce system. Each chapter follows the same pattern:

1. **Concept** — what the technology or design pattern solves.
2. **How it works** — the request, event, or deployment flow.
3. **Where it appears here** — exact modules and files to open.
4. **Code walkthrough** — representative code from this repository or a clearly labelled production blueprint.
5. **Interview prompts** — questions you should be able to answer without opening the notes.

For a single continuous revision pass, start with the [Complete Interview Guide](00-complete-interview-guide/README.md). It contains every topic from the expanded syllabus in one document and links to the deeper chapter for code and trade-offs.

## Status labels

| Label | Meaning |
|---|---|
| **Implemented** | Present in the primary services and wired into the local platform. |
| **Demonstrated** | Present in `additional/` as a focused example, not necessarily part of the production composition. |
| **Blueprint** | A production-grade extension explained here but not currently implemented in this checkout. |
| **Gap** | A deliberate limitation or hardening task you should be able to identify and explain. |

## Learning path

| Stage | Chapter | Outcome |
|---:|---|---|
| 0 | [Service catalog](reference/service-catalog.md) | Know every service, port, endpoint, data store, and source location. |
| 1 | [Foundations](01-microservices-foundations/README.md) | Explain why and when to use microservices. |
| 2 | [Spring Boot and REST](02-spring-boot-and-rest/README.md) | Trace an HTTP request through controller, service, repository, and DTO. |
| 3 | [Domain and data](03-domain-design-and-data/README.md) | Define bounded contexts and database ownership. |
| 4 | [Discovery and config](04-service-discovery-and-config/README.md) | Explain runtime service location and externalized configuration. |
| 5 | [Gateway](05-api-gateway/README.md) | Explain edge routing, filters, authentication, and failure handling. |
| 6 | [Communication](06-interservice-communication/README.md) | Choose synchronous HTTP clients deliberately. |
| 7 | [Events](07-event-driven-architecture/README.md) | Explain Kafka topics, groups, offsets, and eventual consistency. |
| 8 | [Security](08-security-and-keycloak/README.md) | Explain OAuth2 flows and JWT validation end to end. |
| 9 | [Resilience](09-resilience-and-reliability/README.md) | Prevent cascading failures and duplicate side effects. |
| 10 | [Observability](10-observability/README.md) | Correlate logs, metrics, and traces across services. |
| 11 | [Testing](11-testing-and-quality/README.md) | Design tests that catch distributed-system failures. |
| 12 | [Docker](12-docker-and-local-platform/README.md) | Reproduce the platform locally and reason about container networking. |
| 13 | [Kubernetes](13-kubernetes/README.md) | Convert services into deployable workloads. |
| 14 | [Production](14-production-architecture/README.md) | Discuss hardening, delivery, payments, and migrations. |
| 15 | [Interview playbook](15-interview-playbook/README.md) | Give a concise, technically accurate project explanation. |

## Expanded deep dives

| Area | Deep dive |
|---|---|
| Architecture, decomposition, and Conway's Law | [16 — Architecture patterns](16-architecture-patterns/README.md) |
| REST maturity, gRPC, GraphQL, contracts, and schema evolution | [17 — Communication and contracts](17-communication-and-contracts/README.md) |
| Load balancing, discovery, and service mesh | [18 — Discovery and mesh](18-discovery-load-balancing-mesh/README.md) |
| ACID/BASE, CAP/PACELC, sagas, outbox, CDC, CQRS, and event sourcing | [19 — Distributed data](19-distributed-data-and-consistency/README.md) |
| Resilience, caching, backpressure, and traffic management | [20 — Resilience and caching](20-resilience-and-caching/README.md) |
| OpenTelemetry, SLOs, synthetic monitoring, and chaos | [21 — Observability and chaos](21-observability-and-chaos/README.md) |
| Zero trust, mTLS, SPIFFE, OPA, secrets, and OWASP API risks | [22 — Zero-trust security](22-zero-trust-security/README.md) |
| Multi-stage images, K8s releases, feature flags, and migrations | [23 — Deployment and DevOps](23-deployment-and-devops/README.md) |
| IoC, bean lifecycle, scopes, auto-configuration, and profiles | [24 — Spring Core internals](24-spring-core-internals/README.md) |
| MVC/WebFlux lifecycle, servers, validation, and HTTP clients | [25 — Spring Web and HTTP](25-spring-web-and-http/README.md) |
| Transactions, Hibernate state, query tuning, locking, and pools | [26 — Spring Data/JPA](26-spring-data-jpa-and-transactions/README.md) |
| AOP, async execution, scheduling, and resilience annotations | [27 — AOP, async, and resilience](27-spring-aop-async-and-resilience/README.md) |
| Security filter chain, authentication, authorization, CORS, and CSRF | [28 — Spring Security internals](28-spring-security-internals/README.md) |
| Actuator, OpenTelemetry, startup optimization, AOT, and operations | [29 — Actuator and production operations](29-actuator-and-production-operations/README.md) |
| JUnit, Mockito, slice tests, Testcontainers, and WireMock | [30 — Spring testing](30-spring-testing/README.md) |

## Runtime map

The main request path is:

```text
client
  -> gateway:8080
  -> product:8081 / user:8082 / order:8083
  -> PostgreSQL or MongoDB
order
  -> Kafka topic order.exchange
  -> notification consumer
```

The platform path is:

```text
services
  -> Config Server:8888 for configuration
  -> Eureka:8761 for discovery
  -> Keycloak:8443 for OAuth2/OIDC
  -> Zipkin:9411 for traces
  -> Prometheus:9090 and Grafana:3000 for metrics
  -> Loki/Grafana Alloy for logs
```

## How to use this for interviews

For every technology, memorize four sentences:

1. The problem it solves.
2. The trade-off it introduces.
3. Where this repository uses it.
4. What you would change for production.

For example: “Eureka removes hard-coded instance addresses through client-side discovery. It adds registry availability and staleness concerns. The order service resolves `http://product-service` through a load-balanced `RestClient.Builder`. In production I would compare it with Kubernetes Service DNS and define a registry failure strategy.”

## Local setup preflight

Run the commands below from the repository root before debugging an application failure:

```bash
git status --short
java -version
docker version
docker compose -f deploy/docker/docker-compose.yml config
```

Then inspect the environment and service health:

```bash
docker compose -f deploy/docker/docker-compose.yml ps
curl -sS http://localhost:8761/actuator/health
curl -sS http://localhost:8888/actuator/health
curl -sS http://localhost:9411/health
```

The Compose file is a learning environment, not a secure production deployment. Do not reuse its default credentials, anonymous Grafana settings, public management endpoints, or development Keycloak configuration outside a local machine.
