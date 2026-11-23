# 4. Service Discovery and Centralized Configuration

## Why these are separate concerns

Service discovery answers **where is an instance?** Configuration answers **how should it behave?**

```text
Eureka:
  product-service -> 10.0.2.17:8081
  product-service -> 10.0.2.18:8081

Config Server:
  product-service + docker profile -> database URL, tracing, Actuator, resilience
```

Do not put secrets or environment-specific host names into Java source. Do not use the registry as a configuration database.

## Eureka server

The registry is enabled by the `eureka` module. Its learning configuration disables self-registration and registry fetching:

```yaml
spring:
  application:
    name: eureka

server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```

The server application is [`EurekaApplication`](../../eureka/src/main/java/com/ecommerce/eureka/EurekaApplication.java); the service port is declared in [`eureka/application.yml`](../../eureka/src/main/resources/application.yml).

## Eureka clients

Product, User, Order, and Gateway receive this kind of configuration from Config Server:

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka:8761/eureka/
    register-with-eureka: true
    fetch-registry: true
```

At startup, each client:

1. reads its application name;
2. loads configuration;
3. registers host, port, and health metadata;
4. periodically renews its lease;
5. fetches the registry;
6. resolves a logical service name at call time.

The `lb://PRODUCT-SERVICE` gateway URI and the load-balanced `http://product-service` RestClient URL both depend on this logical name.

## Spring Cloud Config Server

The Config Server is enabled with `@EnableConfigServer`:

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigserverApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigserverApplication.class, args);
    }
}
```

The primary service files are in [`configserver/src/main/resources/config`](../../configserver/src/main/resources/config). Docker uses the classpath-native profile:

```yaml
spring:
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/config

server:
  port: 8888
```

The local non-Docker file currently contains a machine-specific Windows filesystem path. Treat the Docker classpath profile or an environment-substituted location as the portable option.

## Config client startup

Each primary service has a local and Docker bootstrap file:

```yaml
spring:
  application:
    name: order-service
  config:
    import: optional:configserver:http://localhost:8888
```

Docker changes only the host name:

```yaml
spring:
  config:
    import: optional:configserver:http://config-server:8888
```

`optional:` lets the application start when Config Server is unavailable. That is convenient for development but can hide a broken production bootstrap. In production, fail fast unless a service has a documented safe local configuration.

## Configuration lookup

For a service named `order-service` and profile `docker`, Spring Cloud Config looks for combinations such as:

```text
order-service-docker.yml
order-service.yml
application-docker.yml
application.yml
```

The exact precedence depends on the client and active profile. Keep shared defaults small, make service-specific values explicit, and document which values must be supplied by secrets or environment variables.

## Refresh and Spring Cloud Bus

`MessageController` is annotated with `@RefreshScope` and reads `app.message` from configuration:

```java
@RestController
@RefreshScope
public class MessageController {
    @Value("${app.message}")
    private String message;

    @GetMapping("/message")
    public String getMessage() {
        return message;
    }
}
```

The Config Server includes the Spring Cloud Bus AMQP dependency. A production refresh design should define:

- who may call refresh endpoints;
- which properties are safe to refresh;
- whether all instances must receive the event;
- how to audit a configuration change;
- how to roll back a bad value.

## Discovery versus Kubernetes DNS

Eureka is client-side discovery: the application fetches a registry and load-balances calls. Kubernetes usually supplies server-side discovery through a stable `Service` DNS name:

```text
Eureka:      lb://PRODUCT-SERVICE
Kubernetes:  http://product-service.default.svc.cluster.local
```

The application still needs timeouts and retry policies in both cases. Kubernetes does not remove distributed-systems failure.

## Production hardening

- Run multiple Eureka instances if Eureka remains a dependency.
- Set explicit lease, eviction, and health-check behavior.
- Do not advertise loopback or container-only addresses accidentally.
- Store Config Server content in version-controlled, reviewed configuration.
- Use a secret manager for passwords, tokens, and keys.
- Fail fast when required configuration is missing.
- Encrypt configuration in transit and at rest.
- Restrict Config Server and Eureka endpoints to the platform network.
- Add readiness checks that verify the service can obtain required configuration.

## Interview questions

**What happens if Eureka is down after a service has started?**  
Existing clients may continue using a cached registry, but new registrations and fresh discovery can fail. The application needs a strategy for stale instances and registry recovery.

**Why use Config Server if environment variables exist?**  
Environment variables are excellent for deployment-specific secrets and overrides. Config Server provides versioned, shared, service-specific configuration and coordinated refresh. They complement each other.

**Why is `optional:configserver` risky?**  
It can let a service start with incomplete defaults, causing a later runtime failure or unsafe behavior. For required configuration, fail fast and make the dependency visible.
