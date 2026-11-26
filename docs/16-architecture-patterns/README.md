# 16. Architecture, Decomposition, and DDD Deep Dive

This chapter answers the architecture questions that usually come before implementation questions: “Why are these services separate?”, “How large should a service be?”, and “How would you migrate a monolith safely?”

## 1. Deployment shape is not design quality

| Shape | Deployment | Transaction boundary | Best use |
|---|---|---|---|
| Monolith | One artifact | Usually local | Small team, early product, tightly coupled domain |
| Modular monolith | One artifact, strict modules | Local unless deliberately split | Discovering boundaries safely |
| Microservices | Independent artifacts | Local per service | Independent ownership/scaling/deployment |
| Distributed monolith | Many artifacts, synchronized changes | Accidental network transaction | Anti-pattern |

A distributed monolith often has all the operational cost of microservices and none of the autonomy:

```text
gateway -> order -> user -> profile -> database
                    |
                    -> product -> database
```

If every request needs every service, every service must be deployed together, or every service shares a database, the boundary is probably misplaced.

## 2. Domain-Driven Design vocabulary

### Ubiquitous language

Write a glossary before designing endpoints:

| Term | Product meaning | Order meaning |
|---|---|---|
| Product | Catalog item and stock | Snapshot referenced by ID |
| Price | Current catalog price | Price captured on order item |
| User | Profile and identity link | Buyer identified by trusted subject |
| Status | Active/inactive catalog state | Pending, confirmed, cancelled, payment state |

The same word can have different models in different bounded contexts. Sharing a Java entity to avoid mapping is often a coupling mistake.

### Context map

```text
Product context --published API/event--> Order context
User context    --published API/event--> Order context
Order context   --OrderCreated event--> Notification context
Identity        --OIDC/JWT-----------> Gateway and services
```

An anti-corruption layer translates a foreign model:

```java
public final class ProductAcl {
    private final LegacyCatalogClient legacy;

    public CatalogProduct find(String id) {
        LegacyProduct source = legacy.get(id);
        return new CatalogProduct(
                source.code(),
                source.displayName(),
                Money.of(source.amount(), source.currency()));
    }
}
```

The ACL should translate semantics, not just rename fields. It is also a good place to handle legacy error codes and compatibility behavior.

### Aggregates

An aggregate protects invariants. If an Order must have at least one item and its total must equal the sum of line totals, those rules belong at the Order boundary. Do not load an entire customer history into the aggregate to update one order.

## 3. Decomposition by capability and subdomain

Use business capabilities first:

```text
catalog
identity/profile
checkout/order
notification
```

Then classify each subdomain:

```text
core:       checkout/pricing rules if they differentiate the business
supporting: catalog administration
generic:    identity, email, payments provider integration
```

A generic capability may be an external platform rather than a custom service. A service boundary needs an owner, a data owner, an API/event contract, and an operational owner.

### A decomposition scorecard

Give a candidate boundary a score from 1-5:

| Question | Strong boundary signal |
|---|---|
| Does it have a distinct business capability? | Yes |
| Does one team own it? | Yes |
| Does it have an independent data model? | Yes |
| Does it change at a different rate? | Yes |
| Can it expose a small stable contract? | Yes |
| Can it fail/degrade independently? | Yes |
| Does it require frequent distributed transactions? | No |

If the last answer is “yes,” keep the code together or redesign the workflow before extracting.

## 4. Strangler migration

The monolith is [`additional/ecom-application`](../../additional/ecom-application). A safe migration:

```text
Phase 1: modularize catalog inside monolith
Phase 2: define Product API and tests
Phase 3: route reads to Product Service
Phase 4: dual-read/compare and observe
Phase 5: move writes and database ownership
Phase 6: remove old catalog path
```

Use a facade or gateway to send one route to the new service while leaving other routes in the monolith. Avoid a “big bang” rewrite.

## 5. Sidecar and ambassador patterns

### Sidecar

```yaml
containers:
  - name: order
  - name: telemetry-sidecar
```

The sidecar can collect logs, expose metrics, or proxy mTLS. It shares the pod lifecycle but has its own CPU/memory and failure modes.

### Ambassador

```text
application -> localhost:9000 -> legacy payment adapter
```

The ambassador hides a remote protocol, retries, or credential flow from the application. Do not use a sidecar merely to avoid designing a clear application client.

## 6. Conway's Law and service size

Conway's Law predicts that architecture follows communication paths. Define:

- team ownership;
- on-call responsibility;
- service and database ownership;
- API review authority;
- deploy/release independence.

The Two-Pizza Team rule is a heuristic for communication, not a reason to create a service per person. Prefer a cohesive service that a team can understand and operate over an arbitrary line-count target.

## 7. Architecture interview answers

**How do you identify a distributed monolith?**
I look for synchronized deployments, shared schema writes, long synchronous call chains, common release trains, and failure propagation. Multiple Spring Boot projects are not enough.

**Why is an ACL useful during migration?**
It prevents legacy models and assumptions from leaking into the new bounded context. That lets the new model evolve while the old system remains stable.

**When would you keep a modular monolith?**
When the team is small, boundaries are uncertain, transactions are tightly coupled, or independent scaling/deployment is not yet valuable. I would enforce module boundaries and extract later with evidence.
