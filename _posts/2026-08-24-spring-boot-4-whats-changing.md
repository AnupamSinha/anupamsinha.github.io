---
title: "Spring Boot 4.0: Everything That's Changing (And How to Prepare)"
date: 2026-08-24
categories: [Spring Boot, Fundamentals]
tags: [spring-boot, java, migration, spring-framework, java-21]
description: "A practical migration guide covering Java 21 baseline, Jakarta EE 11, Hibernate 7, virtual threads by default, and what you need to do now to be ready"
mermaid: true
---
## The Big Picture

Spring Boot 4.0 is shaping up to be the most significant major release since Spring Boot 2.0 dropped Java 6/7 support. After living through the Spring Boot 2 to 3 migration (Jakarta namespace, anyone?), I want to make sure we're all better prepared this time.

I've been tracking the Spring team's roadmaps, GitHub milestones, and conference talks throughout 2025. Here's everything that's changing, what it means for your applications, and a practical migration checklist so you're not scrambling when it drops.

## Baseline: Java 21 Minimum

The biggest headline: **Spring Boot 4.0 requires Java 21 as a minimum.** Not Java 17, not Java 11 — Java 21.

This isn't surprising. Spring Framework 7 (which underpins Boot 4) already declared this. But for teams still running Java 17 in production, this is your wake-up call.

**What this enables:**

- Virtual threads as a first-class citizen (Project Loom)
- Pattern matching (switch expressions, record patterns)
- Sealed classes for domain modeling
- Record patterns in instanceof and switch
- Sequenced collections

**What you need to do now:**

```java
// If your code still does this, start migrating today
if (obj instanceof String) {
    String s = (String) obj; // Java 8 style
    process(s);
}

// Java 21 style - cleaner, more expressive
if (obj instanceof String s) {
    process(s);
}

// Even better with switch
switch (obj) {
    case String s -> process(s);
    case Integer i -> calculate(i);
    case null -> handleNull();
    default -> handleUnknown(obj);
}
```

**Action item:** Run your test suite against Java 21 now. Most codebases work without changes, but third-party libraries may have compatibility issues. Better to discover them today.

## Jakarta EE 11 (Up from Jakarta EE 10)

Spring Boot 3 moved from `javax.*` to `jakarta.*`. Spring Boot 4 takes the next step: Jakarta EE 11 as the baseline.

Key changes in Jakarta EE 11:

**Servlet 6.1** — New API methods, better async support

**Jakarta Persistence 3.2** — Enhanced criteria API, improved entity graphs

**Jakarta Validation 3.1** — New constraint annotations, better generic support

**Jakarta JSON-P 2.1 / JSON-B 3.0** — Updated JSON processing APIs

```java
// Jakarta Persistence 3.2 - cleaner criteria queries
@Repository
public class OrderRepository {

    @PersistenceContext
    private EntityManager em;

    public List<Order> findByStatus(OrderStatus status, int limit) {
        // New simplified criteria API in JPA 3.2
        return em.createQuery(
            "SELECT o FROM Order o WHERE o.status = :status ORDER BY o.createdAt DESC",
            Order.class
        )
        .setParameter("status", status)
        .setMaxResults(limit)
        .getResultList();
    }
}
```

**Migration impact:** If you survived the `javax` to `jakarta` migration in Boot 3, this is relatively minor. Most changes are additive, not breaking. The main risk is third-party libraries that haven't updated to Jakarta EE 11 yet.

## Hibernate 7: The Big One

This is where I expect the most migration pain. Hibernate 7 brings significant changes:

**Removed deprecated APIs** — Everything deprecated in Hibernate 6.x is gone

**New type system** — Improved handling of Java 21 types (records, sealed classes)

**Stateless session improvements** — Better for batch processing

**Criteria API overhaul** — More aligned with JPA 3.2

```java
// Hibernate 7 - better record support for projections
public record OrderSummary(
    Long id,
    String customerName,
    BigDecimal total,
    OrderStatus status,
    LocalDateTime createdAt
) {}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Direct record projection - improved in Hibernate 7
    @Query("""
        SELECT new com.company.OrderSummary(
            o.id, c.name, o.total, o.status, o.createdAt
        )
        FROM Order o JOIN o.customer c
        WHERE o.status = :status
        """)
    List<OrderSummary> findSummariesByStatus(@Param("status") OrderStatus status);
}
```

**What's being removed:**

- `@Type` annotation (replaced by `@JdbcTypeCode` and custom type contributors)
- Legacy `SessionFactory` bootstrap methods
- `hibernate.hbm2ddl.auto` alias spellings (use the canonical forms)
- Several deprecated `Criteria` API methods

**Action item:** Run your application with Hibernate 6.6+ and enable deprecation warnings. Fix every warning before Boot 4 arrives:

```properties
# Add to application.properties now
spring.jpa.properties.hibernate.log_slow_query=1000
spring.jpa.properties.hibernate.show_deprecation_warnings=true
```

## Virtual Threads as Default

This is the change I'm most excited about. Spring Boot 4 is expected to make virtual threads the default for request handling when running on Java 21+.

```java
// In Spring Boot 3.2+, you opt in:
spring.threads.virtual.enabled=true

// In Spring Boot 4.0, this is the default behavior
// Every request handler runs on a virtual thread
// No more reactive programming just for scalability
```

What this means in practice:

```java
// This blocking code becomes perfectly scalable in Boot 4
@RestController
public class OrderController {

    @GetMapping("/orders/{id}")
    public OrderDto getOrder(@PathVariable Long id) {
        // This database call blocks a virtual thread, not a platform thread
        // The JVM can handle millions of concurrent requests like this
        Order order = orderRepository.findById(id)
            .orElseThrow(() -> new OrderNotFoundException(id));

        // This external API call also blocks — but it's fine now
        CustomerDto customer = customerClient.getCustomer(order.customerId());

        return OrderDto.from(order, customer);
    }
}
```

**The reactive question:** Does this kill WebFlux/reactive programming?

Not entirely, but for most CRUD applications, you no longer need reactive to achieve high concurrency. Virtual threads give you the scalability of reactive with the simplicity of imperative code.

**When you still want reactive:**

- Streaming responses (Server-Sent Events, WebSocket feeds)
- Backpressure is critical (bounded buffer requirements)
- You're already invested in a reactive stack and it's working well

**Migration consideration:** If you adopted WebFlux purely for scalability, you can now simplify back to Spring MVC:

```java
// Before: WebFlux for scalability (complex)
@GetMapping("/orders/{id}")
public Mono<OrderDto> getOrder(@PathVariable Long id) {
    return orderRepository.findById(id)
        .flatMap(order -> customerClient.getCustomer(order.customerId())
            .map(customer -> OrderDto.from(order, customer)))
        .switchIfEmpty(Mono.error(new OrderNotFoundException(id)));
}

// After: Spring MVC with virtual threads (simple, same throughput)
@GetMapping("/orders/{id}")
public OrderDto getOrder(@PathVariable Long id) {
    Order order = orderRepository.findById(id)
        .orElseThrow(() -> new OrderNotFoundException(id));
    CustomerDto customer = customerClient.getCustomer(order.customerId());
    return OrderDto.from(order, customer);
}
```

## Removal of Deprecated APIs

Spring Boot 4 is doing a major cleanup. Everything deprecated in the 3.x line gets removed.

**Key deprecations to address now:**

```java
// DEPRECATED: RestTemplate configuration
@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder.build();
}

// PREFERRED: RestClient (Spring Boot 3.2+)
@Bean
public RestClient restClient(RestClient.Builder builder) {
    return builder
        .baseUrl("https://api.example.com")
        .defaultHeader("Accept", MediaType.APPLICATION_JSON_VALUE)
        .build();
}
```

```java
// DEPRECATED: WebSecurityConfigurerAdapter (removed in Boot 3, but some patterns linger)
// Make sure you're using the component-based security configuration

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

```java
// DEPRECATED: Old property names
// spring.redis.* -> spring.data.redis.*
// spring.elasticsearch.* -> spring.data.elasticsearch.*
// management.metrics.* -> management.observations.*

// Check your application.properties for deprecated keys:
// Run: ./gradlew bootRun and look for deprecation warnings in logs
```

## AI-First Support: Spring AI Integration

Spring Boot 4 is expected to include first-class Spring AI auto-configuration out of the box. This means:

```java
// Expected: Auto-configuration for AI clients
@RestController
public class AssistantController {

    private final ChatClient chatClient;

    public AssistantController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are a helpful assistant for our e-commerce platform")
            .build();
    }

    @PostMapping("/api/assistant/query")
    public AssistantResponse query(@RequestBody UserQuery query) {
        String response = chatClient.prompt()
            .user(query.message())
            .call()
            .content();

        return new AssistantResponse(response);
    }
}
```

```properties
# Expected auto-configuration properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.model=gpt-4o
spring.ai.vectorstore.pgvector.dimensions=1536
```

This makes adding AI capabilities to existing Spring Boot applications trivial — no separate dependency management or manual configuration needed.

## Observability Improvements

Spring Boot 4 doubles down on the Micrometer Observation API:

```java
// Enhanced automatic instrumentation
@RestController
@Observed(name = "order.controller") // Automatic metrics + traces
public class OrderController {

    @GetMapping("/orders")
    public Page<OrderDto> listOrders(Pageable pageable) {
        // Automatically observed: HTTP metrics, trace spans, log correlation
        return orderService.findAll(pageable).map(OrderDto::from);
    }
}
```

Expected improvements:

- Better default dashboards for Grafana
- Automatic SLO/SLI metric generation
- Improved trace context propagation across virtual threads
- Native OpenTelemetry support without bridges

## The Migration Checklist: What to Do NOW

Don't wait for Boot 4 to drop. Start preparing today:

## Phase 1: Foundation (Do This Quarter)

**1. Upgrade to Java 21 in CI/CD**

```xml
<!-- pom.xml -->
<properties>
    <java.version>21</java.version>
</properties>
```

Run your full test suite. Fix any Java 21 incompatibilities.

**2. Upgrade to latest Spring Boot 3.4.x**

Stay current with Boot 3.x. Each minor release removes additional deprecation warnings and moves APIs closer to the Boot 4 target.

**3. Enable virtual threads experimentally**

```properties
spring.threads.virtual.enabled=true
```

Run load tests. Identify any code that assumes ThreadLocal behavior or uses `synchronized` blocks on I/O paths.

**4. Audit deprecated API usage**

```bash
# Gradle
./gradlew compileJava -Xlint:deprecation

# Maven
mvn compile -Dmaven.compiler.showDeprecation=true
```

## Phase 2: Dependencies (Next Quarter)

**5. Update Hibernate to 6.6+**

Fix all Hibernate deprecation warnings. Replace `@Type` annotations. Test thoroughly.

**6. Migrate from RestTemplate to RestClient**

```java
// Replace every RestTemplate usage
@Service
public class PaymentClient {

    private final RestClient restClient;

    public PaymentClient(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("https://payment-gateway.example.com")
            .defaultHeader("Authorization", "Bearer " + apiKey)
            .build();
    }

    public PaymentResult charge(PaymentRequest request) {
        return restClient.post()
            .uri("/v1/charges")
            .body(request)
            .retrieve()
            .body(PaymentResult.class);
    }
}
```

**7. Review ThreadLocal usage**

Virtual threads don't carry ThreadLocal across scheduling boundaries the same way. Audit any `ThreadLocal` usage:

```java
// This pattern can break with virtual threads
private static final ThreadLocal<RequestContext> context = new ThreadLocal<>();

// Prefer: ScopedValue (Java 21 preview, expected stable in 25)
// Or: Pass context explicitly through method parameters
// Or: Use Spring's RequestContextHolder (which is virtual-thread aware)
```

## Phase 3: Architecture (Before Boot 4 GA)

**8. Evaluate your reactive codebase**

If you're using WebFlux purely for concurrency, plan a migration back to Spring MVC with virtual threads. The code will be simpler and more maintainable.

**9. Test with GraalVM native image**

Boot 4 will have improved native image support. Start testing now:

```bash
./mvnw -Pnative native:compile
```

Fix any reflection or proxy issues.

**10. Update test infrastructure**

Ensure your test libraries are compatible. JUnit 5.11+, Testcontainers latest, Mockito 5.x.

## What NOT to Worry About

**Your business logic doesn't change.** The Spring programming model remains the same. `@RestController`, `@Service`, `@Repository` — all still work exactly as expected.

**Spring Data repositories are fine.** The abstraction layer insulates you from Hibernate changes.

**Your deployment model doesn't change.** Containers, Kubernetes, cloud platforms — all the same.

## Timeline and Strategy

Based on Spring's historical patterns:

**Spring Framework 7 GA** — Expected mid-2026

**Spring Boot 4.0 GA** — Expected late 2026

**Spring Boot 3.x maintenance** — Likely supported until mid-2028

You have time, but the migration window is your friend. Start now with Phase 1, and by the time Boot 4 drops, you'll need a day to upgrade, not a quarter.

## Final Thoughts

The Boot 2 to 3 migration caught many teams off guard because of the `javax` to `jakarta` namespace change. Boot 4 doesn't have a single disruptive change like that, but it has many smaller changes that compound.

The team that starts preparing now — upgrading to Java 21, fixing deprecations, testing virtual threads — will have a smooth migration day. The team that waits will be debugging Hibernate 7 breaking changes under pressure.

Don't be the second team.
