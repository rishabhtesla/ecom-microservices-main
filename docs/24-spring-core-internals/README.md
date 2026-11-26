# 24. Spring Core and Boot Internals

This chapter focuses on what Spring does at startup and why apparently simple annotations have proxy, lifecycle, and conditional-configuration consequences.

## 1. `BeanFactory` and `ApplicationContext`

`BeanFactory` is the basic IoC container. `ApplicationContext` builds on it with events, message resolution, resource loading, environment/profile support, and automatic post-processor registration. Spring Boot creates an `ApplicationContext`, registers bean definitions, applies post-processors, and starts the application.

Constructor injection is the default recommendation:

```java
@Service
public class CheckoutService {
    private final ProductClient products;
    private final OrderRepository orders;

    public CheckoutService(ProductClient products, OrderRepository orders) {
        this.products = products;
        this.orders = orders;
    }
}
```

It makes required dependencies explicit, supports immutable fields, and exposes cycles at design time. Setter injection is suitable for optional dependencies. Field injection hides dependencies and makes isolated tests harder.

Use `@Qualifier` when several beans have the same type:

```java
public CheckoutService(@Qualifier("catalogClient") ProductClient client) { ... }
```

Use `@Primary` only when there is a genuine default. `@Lazy` defers initialization but can move failures from startup to first use.

## 2. Bean lifecycle

The simplified lifecycle is:

```text
instantiate
 -> dependency/property population
 -> Aware callbacks
 -> BeanPostProcessor before initialization
 -> @PostConstruct / InitializingBean
 -> BeanPostProcessor after initialization
 -> ready for use
 -> @PreDestroy / DisposableBean on shutdown
```

Example:

```java
@Component
public class ConnectionPoolOwner {
    @PostConstruct
    void start() { /* validate configuration and create resources */ }

    @PreDestroy
    void stop() { /* close resources */ }
}
```

`BeanFactoryPostProcessor` changes bean definitions before instances are created. `BeanPostProcessor` wraps or changes instances and is how many annotations become proxies. Do not perform remote calls in a post-processor unless startup ordering and failure behavior are deliberate.

## 3. Proxies and scopes

| Scope | Meaning |
|---|---|
| singleton | One instance per context |
| prototype | New instance per lookup |
| request/session | Web request/session lifetime |
| application | Servlet application lifetime |

Injecting a prototype directly into a singleton resolves it once. Use `ObjectProvider<T>`:

```java
@Component
class JobRunner {
    private final ObjectProvider<JobContext> contexts;

    JobRunner(ObjectProvider<JobContext> contexts) {
        this.contexts = contexts;
    }

    void run() {
        JobContext context = contexts.getObject();
        // new prototype context for this run
    }
}
```

Scoped proxies (`ScopedProxyMode.TARGET_CLASS`) inject a proxy that resolves the current request/session object.

## 4. Circular dependencies

Constructor cycles are design problems:

```text
A -> B -> A
```

Breaking the cycle with lazy or setter injection can hide the problem. Extract a third abstraction, publish an event, or reverse ownership. Spring Boot disables circular references by default in modern versions because silent cycles make startup and design failures harder to reason about.

## 5. Auto-configuration

`@SpringBootApplication` combines component scanning, auto-configuration import, and configuration support. Auto-configurations are activated by conditions:

```java
@AutoConfiguration
@ConditionalOnClass(RedisTemplate.class)
@ConditionalOnMissingBean(CacheManager.class)
public class ProductCacheAutoConfiguration {
    @Bean
    CacheManager productCacheManager() { ... }
}
```

Modern Boot libraries list auto-configurations in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Conditions such as `@ConditionalOnProperty`, `@ConditionalOnBean`, and `@ConditionalOnMissingBean` allow application code to override defaults.

A custom starter normally contains:

```text
my-feature-autoconfigure
  @AutoConfiguration + conditions + beans
my-feature-spring-boot-starter
  dependency on autoconfigure and runtime libraries
```

Keep auto-configuration opt-in or safely conditional; never replace an application bean unexpectedly.

## 6. Externalized configuration

Property precedence depends on source and profile. Prefer typed configuration:

```java
@ConfigurationProperties("checkout")
@Validated
public record CheckoutProperties(
        @NotNull Duration timeout,
        @Min(1) int maxItems) {}
```

`@Value` is useful for a single property but becomes difficult to validate and refactor across a large service. Profiles select environment-specific behavior, not secrets. Secrets should come from a secret manager or protected runtime injection.

The repository uses Spring Cloud Config and service-specific files under [`configserver/src/main/resources/config`](../../configserver/src/main/resources/config). `optional:configserver` is convenient locally but can allow a service to start without required configuration; production should fail clearly when critical configuration is missing.

## 7. Interview Q&A

**Why does `@Transactional` not work on a private method?**
Spring typically applies it through a proxy. Calls must pass through the proxy and the method must be interceptable; a private method or self-invocation bypasses the proxy.

**What is the difference between `@Configuration` and `@Component`?**
Both create candidates, but `@Configuration` is designed for bean factory methods and preserves inter-bean method semantics through configuration enhancement.

**How do you debug unexpected auto-configuration?**
Enable the condition evaluation report (`--debug`), inspect the matched/missed conditions, and override or exclude the specific auto-configuration rather than guessing.
