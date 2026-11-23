# 8. Security, OAuth2, JWT, and Keycloak

## Authentication and authorization

- **Authentication:** Who is the caller?
- **Authorization:** What may that caller do?
- **Identity provider:** System that authenticates users and issues tokens.
- **Resource server:** API that validates access tokens.
- **OAuth2 client:** Application requesting a token to call a resource.
- **OIDC:** Identity layer on top of OAuth2, including user identity claims.

In this repository, Keycloak is the identity provider, the Gateway is a JWT resource server, and the User Service calls the Keycloak Admin API to provision users.

## Token flow in the main application

```text
user/client
  -> Keycloak realm ecom-app
  <- access token (JWT)
  -> Gateway Authorization: Bearer <token>
  -> signature/issuer/audience validation
  -> route authorization
  -> downstream service
```

A JWT is signed, not encrypted. Never put passwords, card data, or sensitive secrets in claims. Validate signature, issuer, expiration, and audience; then authorize based on trusted claims.

## Gateway resource-server configuration

The active security chain is:

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(
            ServerHttpSecurity http) {
        return http
                .csrf(ServerHttpSecurity.CsrfSpec::disable)
                .authorizeExchange(exchange -> exchange
                        .anyExchange().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2
                        .jwt(jwt -> jwt
                                .jwtAuthenticationConverter(
                                        grantedAuthoritiesExtractor())))
                .build();
    }
}
```

The issuer is configured in [`gateway-service-docker.yml`](../../configserver/src/main/resources/config/gateway-service-docker.yml):

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8443/realms/ecom-app
          jwk-set-uri: http://keycloak:8080/realms/ecom-app/protocol/openid-connect/certs
```

The host-visible issuer and container-visible JWK URL illustrate an important deployment concern: token `iss` must match what the resource server expects, while the JWK endpoint must be reachable from the service network.

## Role mapping

Keycloak can put client roles under `resource_access`. The current converter reads the `oauth2-pkce` client roles and creates Spring authorities such as `ROLE_USER`. The route matchers for product/order/user roles are currently commented out, so successful authentication is enough for all active gateway routes.

Production role mapping should:

1. handle a missing `resource_access` claim without throwing;
2. verify the expected client ID;
3. distinguish realm roles from client roles;
4. use least-privilege matchers;
5. test a token for every role and route.

Do not use a raw unchecked cast from a JWT claim as an authorization decision without null/type checks.

## User provisioning flow

`UserService.addUser` currently:

```text
1. obtain an admin access token from Keycloak
2. create a Keycloak user
3. save the returned Keycloak ID in MongoDB
4. assign the USER client role
5. save the local profile
```

The implementation is in [`KeyCloakAdminService`](../../user/src/main/java/com/ecommerce/user/services/KeyCloakAdminService.java). It calls:

```text
POST /realms/{realm}/protocol/openid-connect/token
POST /admin/realms/{realm}/users
GET  /admin/realms/{realm}/clients/{clientId}/roles/{role}
POST /admin/realms/{realm}/users/{userId}/role-mappings/clients/{clientId}
```

This is useful for learning the Admin API, but the production design should use a confidential service account with narrowly scoped admin permissions. Do not use a human admin password or the resource-owner-password grant for service automation.

## OAuth2 flows demonstrated

### Authorization Code

Use for a server-side web application:

```text
browser -> authorization endpoint
browser <- authorization code
backend -> token endpoint with code + client secret
backend <- access/refresh tokens
backend -> resource API
```

The example is under [`auth-code-flow`](../../additional/keycloak/keycloak-oauth-demos/auth-code-flow).

### Authorization Code with PKCE

Use for a browser/mobile public client that cannot safely store a client secret:

```text
client creates code_verifier
client sends code_challenge
user authenticates
client receives code
client exchanges code + code_verifier
```

The React example uses `oauth2-pkce` and is in [`pkce-flow`](../../additional/keycloak/keycloak-oauth-demos/pkce-flow). PKCE protects the authorization code if it is intercepted; it does not replace TLS or token validation.

### Client Credentials

Use for machine-to-machine calls with no end user:

```text
service A -> Keycloak token endpoint (client ID + secret)
Keycloak -> access token
service A -> service B with bearer token
service B -> validates token and service scope
```

The client/resource example is in [`client-credentials`](../../additional/keycloak/keycloak-oauth-demos/client-credentials).

### Refresh token

Used by a trusted client to obtain a new access token without asking the user to log in again. Store refresh tokens securely, rotate/revoke them, and do not expose them to an untrusted browser unless the architecture explicitly supports it.

## CORS and browser security

The PKCE resource server configures CORS for the React origin. Check origins exactly, including trailing slashes, and allow only required methods/headers. Never use `*` with credentials. Use CSRF protection for cookie-authenticated browser sessions; disabling CSRF is appropriate for a stateless bearer-token API only when the rest of the threat model supports it.

## Keycloak local configuration

The Compose setup starts Keycloak in development mode with local bootstrap credentials. Realm backups are stored under [`additional/keycloak/keycloak-backups`](../../additional/keycloak/keycloak-backups). A repeatable local setup should import the realm explicitly and provision clients/roles through version-controlled automation rather than manual clicking.

## Security gaps to be able to explain

- Default passwords are present in local Compose/configuration. Rotate them and use a secret manager outside local learning.
- One OAuth2 demo contains a client secret in a properties file. Treat it as compromised and rotate it; never copy this pattern into a real repository.
- User provisioning obtains an admin token with a username/password flow and broad privileges.
- `X-User-ID` is trusted by Order Service instead of being derived from a validated token.
- Services behind the gateway are not consistently shown as resource servers.
- The custom role extractor needs null-safe parsing and focused authorization tests.
- Actuator endpoints and Grafana are broadly exposed in the local stack.

## Interview questions

**OAuth2 versus JWT?**  
OAuth2 is an authorization protocol and token acquisition framework. JWT is a token representation. OAuth2 access tokens can be JWTs, but OAuth2 and JWT are not synonyms.

**Why validate JWTs at the resource server?**  
The API that makes the authorization decision must validate the token itself. Trusting only the gateway creates a dangerous single enforcement point.

**What is PKCE for?**  
It binds the authorization-code exchange to a one-time verifier held by the client, protecting public clients from authorization-code interception.

**What is the difference between 401 and 403?**  
`401` means the request lacks valid authentication. `403` means the caller is authenticated but lacks permission.
