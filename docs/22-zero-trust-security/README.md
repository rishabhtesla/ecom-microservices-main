# 22. Zero-Trust Security and API Protection

## 1. Identity layers

```text
authentication: who is the caller?
authorization:  what may the caller do?
auditing:       what did the caller do and why?
```

OAuth 2.0 is an authorization framework. OIDC adds identity claims and an ID token. A resource server should validate an access token, not treat an ID token as an API credential.

The repository's Gateway and services use Keycloak/OIDC examples under [`additional/keycloak`](../../additional/keycloak) and JWT resource-server configuration in the service configuration.

## 2. Token types and validation

```text
access token  -> short-lived API authorization
refresh token -> obtain a new access token; protect more carefully
ID token      -> client identity/session information
```

Validate:

- signature against issuer JWKS;
- issuer;
- audience/resource;
- expiry and not-before;
- algorithm allow-list;
- scopes/roles;
- required tenant or subject claims.

JWKS keys rotate. Cache keys with refresh-on-unknown-kid behavior, but do not disable signature validation when the identity provider is unavailable.

## 3. RBAC, ABAC, and policy

RBAC maps roles to permissions:

```text
ROLE_ADMIN -> product:write
ROLE_USER  -> order:create
```

ABAC evaluates attributes:

```text
allow if subject.tenant == order.tenant
       && action == "read"
       && order.owner == subject.id
```

OPA/Rego can centralize policy decisions, but policy calls add latency and an availability dependency. Cache only decisions whose security freshness is acceptable.

## 4. Service-to-service trust

Zero trust means every request is authenticated and authorized regardless of network location. Options include:

- OAuth client credentials;
- token relay when the downstream needs user context;
- token exchange for a narrower audience;
- mTLS workload identity;
- SPIFFE/SPIRE-issued workload identities.

Do not blindly forward a user token to every downstream service. Audience restriction and least privilege limit blast radius.

## 5. Do not trust headers

The repository currently has an `X-User-ID`-based order path documented as a learning/demo gap. A caller can forge this header unless a trusted edge strips and re-adds it. Prefer:

```java
String subject = authentication.getName();
```

and authorize the subject against the order owner. If a gateway propagates identity, downstream services must still authenticate the gateway and validate the propagated assertion.

## 6. Secret management

Do not commit passwords or client secrets. Use:

```text
Kubernetes Secret -> workload identity/secret store CSI -> process
Vault/Azure Key Vault -> short-lived or rotated credentials
```

Secrets need rotation, access audit, least privilege, and a break-glass process. Environment variables reduce source leakage but are not automatically safe; they can appear in diagnostics and process metadata.

## 7. API protection

- validate size, type, range, and business constraints;
- reject unknown fields where appropriate;
- normalize Unicode and canonicalize paths;
- enforce content type and request limits;
- protect against SSRF by allow-listing destinations;
- prevent mass assignment by using request DTOs;
- rate-limit by identity and network dimensions;
- use CSRF protection for cookie-authenticated browser sessions;
- use secure CORS allow-lists, never arbitrary origins with credentials;
- return generic errors without stack traces.

### OWASP API risks to discuss

Broken object-level authorization, broken function-level authorization, unrestricted resource consumption, SSRF, security misconfiguration, improper inventory, and unsafe consumption of APIs are particularly relevant to microservices.

## 8. Spring Security shape

```java
@Bean
SecurityFilterChain apiSecurity(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health/**").permitAll()
            .requestMatchers(HttpMethod.POST, "/api/orders").hasAuthority("SCOPE_orders:write")
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()))
        .build();
}
```

Disabling CSRF is appropriate for a stateless bearer-token API only when it is not also accepting browser cookies. Use method-level authorization for object/business rules:

```java
@PreAuthorize("@orderAuthorization.canRead(#orderId, authentication)")
public OrderResponse get(String orderId) { ... }
```

## Interview Q&A

**Why is mTLS not enough for authorization?**
mTLS authenticates the workload connection. It does not say whether that workload may read this tenant's order or perform this business action.

**What is the difference between OAuth and OIDC?**
OAuth delegates authorization; OIDC standardizes authentication and identity claims on top of OAuth.

**When should CSRF be enabled?**
When browsers automatically attach credentials such as cookies. A bearer token manually attached by a non-browser client has a different CSRF threat model, but XSS/token theft still matter.
