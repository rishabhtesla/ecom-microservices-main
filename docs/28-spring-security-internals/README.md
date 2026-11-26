# 28. Spring Security Internals

## 1. Request security flow

```text
Servlet container
 -> DelegatingFilterProxy
 -> FilterChainProxy
 -> matching SecurityFilterChain
 -> authentication filters
 -> SecurityContext
 -> authorization filters
 -> controller
```

In a reactive Gateway the equivalent concepts use a WebFlux security chain. The implementation style differs, but authentication must still happen before authorization and the security context must be propagated safely.

## 2. Authentication components

```text
AuthenticationManager
  -> AuthenticationProvider
      -> UserDetailsService / JWT decoder / external IdP
  -> authenticated Authentication
  -> SecurityContext
```

For username/password, use a slow password hash such as BCrypt or Argon2 and never log credentials. For JWT resource servers, the application validates a signed token and maps scopes/claims to authorities; it should not call the identity provider for every request.

## 3. Stateless JWT resource server

```java
@Bean
SecurityFilterChain resourceServer(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(s -> s
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health/**").permitAll()
            .requestMatchers(HttpMethod.POST, "/api/products/**")
                .hasAuthority("SCOPE_products:write")
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()));
    return http.build();
}
```

`permitAll` means the endpoint does not require authentication; it does not mean input validation or rate limiting is unnecessary. Keep actuator exposure narrower than application APIs.

## 4. Authorities and claims

Keycloak commonly places client roles under nested claims. A converter should be null-safe and explicit:

```java
final class KeycloakAuthorities
        implements Converter<Jwt, Collection<GrantedAuthority>> {
    @Override
    public Collection<GrantedAuthority> convert(Jwt jwt) {
        Map<String, Object> resourceAccess =
                jwt.getClaimAsMap("resource_access");
        if (resourceAccess == null) {
            return List.of();
        }
        // Validate expected map/list types before converting.
        return extractRoles(resourceAccess).stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .toList();
    }
}
```

Do not treat an untrusted display name as an identity key. Use the stable `sub` claim, then apply ownership/tenant authorization in the domain.

## 5. Method security

```java
@EnableMethodSecurity
@Configuration
class MethodSecurityConfig {}

@PreAuthorize("hasAuthority('SCOPE_orders:read')")
public OrderResponse getOrder(String id) { ... }
```

Method rules are useful but do not replace object-level checks:

```java
@PreAuthorize("@orderAuthorization.canRead(#id, authentication)")
```

Centralize complex authorization in a typed policy service rather than long SpEL strings.

## 6. CORS, CSRF, and sessions

- CORS controls which browser origins may read responses; it is not authentication.
- CSRF protects cookie-authenticated browser actions from cross-site requests.
- `STATELESS` prevents server sessions from storing authentication state.

If a service accepts both cookies and bearer tokens, choosing one blanket CSRF setting is usually wrong. Model each client flow explicitly.

## 7. OAuth flows to know

| Flow | Typical use |
|---|---|
| Authorization Code + PKCE | Browser/mobile user sign-in |
| Client Credentials | Service-to-service, no user |
| Refresh token | Obtain new access token without user login |
| Token exchange | Narrow/audience-specific downstream token |

Authorization Code and PKCE examples are in [`additional/keycloak/keycloak-oauth-demos`](../../additional/keycloak/keycloak-oauth-demos). Client secrets must not be embedded in public clients.

## 8. Security failure checklist

- trust `X-User-ID` or similar caller-controlled headers;
- accept any JWT issuer/audience;
- disable TLS verification in clients;
- put secrets in `application.yml`;
- expose `/actuator/heapdump`;
- use broad admin Keycloak credentials for runtime requests;
- return stack traces;
- make authorization decisions only at the gateway.

The repository documents these as learning/demo gaps in [`service-catalog.md`](../reference/service-catalog.md). Treat the Compose credentials as local-only and rotate anything reused elsewhere.

## Interview Q&A

**What is the difference between authentication and authorization?**
Authentication establishes identity; authorization decides whether that identity can perform a specific action on a specific resource.

**Why validate audience as well as signature?**
A valid token issued for another API is still not authorized for this resource. Audience validation prevents token confusion across services.

**Why is gateway-only authorization insufficient?**
Services may be reached through another route, a job, or a compromised internal caller. Each service must enforce its own trust boundary.
