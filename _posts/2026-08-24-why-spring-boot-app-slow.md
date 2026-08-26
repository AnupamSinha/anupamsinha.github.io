---
title: "Why Your Spring Boot App Is Slow (And How I Fixed 12 Common Causes)"
date: 2026-08-24
categories: [Spring Boot, Performance]
tags: [spring-boot, performance, java, debugging, production]
description: "A field guide to diagnosing and fixing real production performance issues in Spring Boot — based on 17 years of firefighting Java applications in production."
mermaid: true
---
## These Are Not Theoretical Problems

Every performance issue in this post is something I've personally diagnosed and fixed in production systems. Not blog posts I've read. Not Stack Overflow answers I've bookmarked. Real incidents, real fixes, real production impact.

After 17 years building Java systems in Singapore — across banking, fintech, e-commerce, and logistics — I've developed a nose for these problems. Most of them follow predictable patterns. Let me save you some 3 AM debugging sessions.

## 1. N+1 Queries — The Silent Performance Killer

This is by far the most common performance issue I encounter. It's insidious because it doesn't show up in unit tests or with small datasets.

### The Problem

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
}

// In your service
List<Order> orders = orderRepository.findAll(); // 1 query
for (Order order : orders) {
    order.getItems().size(); // N queries, one per order
}
```

With 100 orders, that's 101 database round trips. With 10,000 orders in a pagination call that forgot to limit? Good luck.

### The Fix

```java
// Option 1: JOIN FETCH in JPQL
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") OrderStatus status);

// Option 2: EntityGraph
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findByStatus(OrderStatus status);

// Option 3: Batch fetching (for existing code you can't easily refactor)
// In application.yml
// spring.jpa.properties.hibernate.default_batch_fetch_size: 50
```

### How to Detect It

Enable SQL logging in dev/staging (never production):

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
    show-sql: false # Use p6spy instead for formatted output
```

Better yet, use a library like `datasource-proxy` with a query count assertion in integration tests:

```java
@Test
void shouldLoadOrdersInSingleQuery() {
    QueryCountHolder.clear();
    orderService.getRecentOrders(PageRequest.of(0, 50));
    assertThat(QueryCountHolder.getGrandTotal().getSelect()).isLessThanOrEqualTo(3);
}
```

## 2. Connection Pool Starvation

### The Problem

Default HikariCP pool size is 10 connections. If your app has 200 concurrent requests and each holds a connection for 100ms, you'll see requests queueing. Under load, this cascades — requests timeout waiting for connections, threads pile up, GC kicks in, everything spirals.

### The Fix

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 30
      minimum-idle: 10
      connection-timeout: 3000    # Fail fast, don't wait forever
      idle-timeout: 600000        # 10 minutes
      max-lifetime: 1800000       # 30 minutes
      leak-detection-threshold: 60000  # Alert if connection held > 60s
```

**The formula I use** — `pool_size = (core_count * 2) + effective_spindle_count`. For cloud databases with SSD storage, start with 20–30 for typical web apps and tune from there.

**Critical** — Your connection pool size multiplied by your pod count must not exceed your database's `max_connections`. I've seen production outages where auto-scaling added pods until the database hit connection limits.

### How to Monitor

```java
@Scheduled(fixedRate = 30000)
public void logPoolStats() {
    HikariPoolMXBean poolProxy = hikariDataSource.getHikariPoolMXBean();
    log.info("Pool stats: active={}, idle={}, waiting={}, total={}",
            poolProxy.getActiveConnections(),
            poolProxy.getIdleConnections(),
            poolProxy.getThreadsAwaitingConnection(),
            poolProxy.getTotalConnections());
}
```

If `threadsAwaitingConnection` is consistently above 0, your pool is undersized.

## 3. Synchronous Calls That Should Be Async

### The Problem

Your API handler calls three external services sequentially:

```java
public OrderResponse processOrder(OrderRequest request) {
    InventoryResult inventory = inventoryService.check(request);   // 150ms
    PaymentResult payment = paymentService.charge(request);         // 300ms
    ShippingResult shipping = shippingService.schedule(request);    // 200ms
    return buildResponse(inventory, payment, shipping);             // Total: 650ms
}
```

### The Fix

If these calls are independent, parallelize them:

```java
public OrderResponse processOrder(OrderRequest request) {
    CompletableFuture<InventoryResult> inventoryFuture =
            CompletableFuture.supplyAsync(() -> inventoryService.check(request), taskExecutor);
    CompletableFuture<PaymentResult> paymentFuture =
            CompletableFuture.supplyAsync(() -> paymentService.charge(request), taskExecutor);
    CompletableFuture<ShippingResult> shippingFuture =
            CompletableFuture.supplyAsync(() -> shippingService.schedule(request), taskExecutor);

    CompletableFuture.allOf(inventoryFuture, paymentFuture, shippingFuture).join();

    return buildResponse(
            inventoryFuture.join(),
            paymentFuture.join(),
            shippingFuture.join()
    ); // Total: ~300ms (max of the three)
}
```

**Important** — Always use a dedicated `TaskExecutor` with bounded thread pool. Never use `ForkJoinPool.commonPool()` for IO-bound work.

```java
@Bean
public TaskExecutor externalServiceExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(100);
    executor.setThreadNamePrefix("ext-service-");
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.initialize();
    return executor;
}
```

## 4. Missing or Misconfigured Caching

### The Problem

Every request for user profile data hits the database, even though profiles change once a week.

### The Fix

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
                .maximumSize(10000)
                .expireAfterWrite(Duration.ofMinutes(15))
                .recordStats());
        return manager;
    }
}

@Service
public class UserProfileService {

    @Cacheable(value = "userProfiles", key = "#userId")
    public UserProfile getUserProfile(Long userId) {
        return userProfileRepository.findById(userId)
                .orElseThrow(() -> new NotFoundException("User not found: " + userId));
    }

    @CacheEvict(value = "userProfiles", key = "#userId")
    public void updateProfile(Long userId, UpdateProfileRequest request) {
        // update logic
    }
}
```

**Common mistake** — Caching mutable data without eviction. If you cache something, you must evict it when it changes. Otherwise you'll get stale data bugs that are incredibly hard to reproduce.

**Another mistake** — Using `@Cacheable` on methods called internally within the same class. Spring AOP proxies don't intercept self-calls. Either inject the bean into itself or use AspectJ weaving.

## 5. Slow JSON Serialization

### The Problem

You're returning massive object graphs with lazy-loaded relationships. Jackson serializes everything, triggering additional queries and creating 5MB response payloads.

### The Fix

Use DTOs. Always.

```java
// Don't return entities from controllers
@GetMapping("/orders/{id}")
public OrderDTO getOrder(@PathVariable Long id) {
    Order order = orderService.findById(id);
    return OrderDTO.from(order); // Explicit mapping, no accidental lazy loading
}

// Projections for read-heavy endpoints
public interface OrderSummaryProjection {
    Long getId();
    String getStatus();
    BigDecimal getTotalAmount();
    LocalDateTime getCreatedAt();
}

@Query("SELECT o.id as id, o.status as status, o.totalAmount as totalAmount, " +
       "o.createdAt as createdAt FROM Order o WHERE o.userId = :userId")
List<OrderSummaryProjection> findSummariesByUserId(@Param("userId") Long userId);
```

Also configure Jackson for speed:

```yaml
spring:
  jackson:
    serialization:
      write-dates-as-timestamps: false
    deserialization:
      fail-on-unknown-properties: false
    default-property-inclusion: non_null  # Don't serialize nulls
```

## 6. Component Scanning Bloat

### The Problem

Your main application class sits in `com.company` and scans everything — including 200 configuration classes, test utilities accidentally on the classpath, and third-party auto-configurations you don't need.

### The Fix

```java
@SpringBootApplication(scanBasePackages = "com.company.myapp")
@EnableAutoConfiguration(exclude = {
    DataSourceAutoConfiguration.class,      // If you configure manually
    SecurityAutoConfiguration.class,         // If you handle security differently
    MailSenderAutoConfiguration.class,       // If you don't send emails
    JmxAutoConfiguration.class              // If you don't use JMX
})
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

For startup time specifically, use Spring Boot's lazy initialization:

```yaml
spring:
  main:
    lazy-initialization: true  # Only for dev/staging, be careful in production
```

## 7. Too Many Interceptors and Filters

### The Problem

Every request passes through 15 filters and interceptors — logging, authentication, rate limiting, request tracking, tenant resolution, feature flags, A/B testing, input sanitization... Each adds 2–5ms. That's 30–75ms of overhead before your handler even starts.

### The Fix

- Audit your filter chain. Remove anything that isn't needed on every request
- Use path-specific filters instead of global ones
- Combine related filters into a single filter with multiple responsibilities
- Order matters — put short-circuit filters (auth, rate limit) first

```java
@Bean
public FilterRegistrationBean<HeavyAuditFilter> auditFilter() {
    FilterRegistrationBean<HeavyAuditFilter> registration = new FilterRegistrationBean<>();
    registration.setFilter(new HeavyAuditFilter());
    registration.addUrlPatterns("/api/admin/*"); // Only admin endpoints
    registration.setOrder(100);
    return registration;
}
```

## 8. Wrong Garbage Collector

### The Problem

You're still using the default GC (G1GC is default since Java 9, but some apps override it). For latency-sensitive apps, G1GC can cause 50–200ms pauses that show up as p99 spikes.

### The Fix

For most Spring Boot web applications:

**Heap < 4GB, throughput-focused** — G1GC (default) is fine

**Heap > 4GB, latency-sensitive** — ZGC (Java 17+)

```
JAVA_OPTS="-XX:+UseZGC -XX:+ZGenerational -Xmx8g -Xms8g"
```

**High throughput, can tolerate pauses** — Parallel GC

```
JAVA_OPTS="-XX:+UseParallelGC -Xmx4g -Xms4g"
```

ZGC keeps pause times under 1ms regardless of heap size. I've seen p99 latency drop from 200ms to 15ms just by switching from G1GC to ZGC on a 12GB heap service.

## 9. Missing Database Indexes

### The Problem

Your query filters on `status` and `created_at` but there's no composite index. The database does a full table scan on 50 million rows.

### The Fix

```java
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_orders_status_created", columnList = "status, created_at DESC"),
    @Index(name = "idx_orders_user_status", columnList = "user_id, status"),
    @Index(name = "idx_orders_tracking", columnList = "tracking_number", unique = true)
})
public class Order {
    // ...
}
```

**Rules I follow** —

- Index every column used in WHERE clauses
- Composite indexes follow the "equality first, range last" principle
- Check `EXPLAIN ANALYZE` for your top 20 queries
- Unused indexes slow down writes. Drop them.
- Partial indexes for queries that filter on a fixed value (e.g., `WHERE status = 'ACTIVE'`)

## 10. Oversized API Payloads

### The Problem

Your `/api/products` endpoint returns every field including 50KB base64-encoded images, full audit history, and internal metadata. Mobile clients on 3G are timing out.

### The Fix

```java
// Sparse fieldsets
@GetMapping("/products")
public List<?> getProducts(
        @RequestParam(defaultValue = "id,name,price,thumbnail") String fields) {
    return productService.findAll().stream()
            .map(p -> filterFields(p, fields.split(",")))
            .collect(Collectors.toList());
}

// Pagination with sensible defaults
@GetMapping("/orders")
public Page<OrderSummaryDTO> getOrders(
        @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC)
        Pageable pageable) {
    return orderService.findSummaries(pageable);
}
```

Also enable compression:

```yaml
server:
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/plain
    min-response-size: 1024
```

## 11. Thread Pool Starvation (Tomcat)

### The Problem

Default Tomcat thread pool is 200 threads. If downstream services are slow (500ms per call) and you have 400 concurrent users, all threads are blocked waiting for responses. New requests queue indefinitely.

### The Fix

```yaml
server:
  tomcat:
    threads:
      max: 400
      min-spare: 50
    accept-count: 100
    connection-timeout: 5000
```

**Better fix** — Don't block threads. Use WebClient (reactive) for external calls:

```java
@Bean
public WebClient webClient() {
    return WebClient.builder()
            .baseUrl("https://downstream-service.com")
            .clientConnector(new ReactorClientHttpConnector(
                    HttpClient.create()
                            .responseTimeout(Duration.ofSeconds(3))
                            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)))
            .build();
}
```

And set proper timeouts. A call without a timeout is a memory leak waiting to happen.

## 12. Cold Start Overhead

### The Problem

Your Spring Boot app takes 45 seconds to start. In Kubernetes with rolling deployments and auto-scaling, this means requests hit pods that aren't ready.

### The Fix

```yaml
# Readiness probe that waits for full initialization
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    readinessstate:
      enabled: true
    livenessstate:
      enabled: true
```

For actual startup time reduction:

- Use Spring Boot 3.2+ with AOT compilation for GraalVM
- Enable class data sharing: `-XX:+UseAppCDS -XX:SharedArchiveFile=app-cds.jsa`
- Use `spring.main.lazy-initialization=true` carefully
- Remove unused auto-configurations
- Consider Spring Boot's new virtual threads support (Java 21) to reduce the thread pool overhead

```java
// Java 21 virtual threads — eliminates thread pool sizing entirely
@Bean
public TomcatProtocolHandlerCustomizer<?> virtualThreadCustomizer() {
    return protocolHandler -> {
        protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
    };
}
```

## My Performance Investigation Checklist

When I get called into a performance issue, here's my systematic approach:

1. **Metrics first** — Check APM dashboards for p50/p95/p99 latency. Where's the time going?
2. **Database queries** — Enable slow query log. Is it one query or death by a thousand cuts?
3. **Connection pools** — Are connections exhausted? Are threads waiting?
4. **GC logs** — Are there long pauses correlating with latency spikes?
5. **Thread dump** — What are threads doing? Blocked on IO? Locked on a monitor?
6. **Heap dump** — If memory is the issue, what's consuming it?
7. **Network** — Latency to downstream services. DNS resolution times. Connection reuse.

In my experience, 70% of Spring Boot performance issues are database-related (N+1, missing indexes, pool exhaustion). Start there.

## Final Thought

Performance is not an afterthought you bolt on later. It's a constraint you design for from the start. Choose the right data structures, set proper timeouts, paginate everything, cache what makes sense, and monitor relentlessly.

The fastest code is the code you don't run. Question every abstraction layer, every framework magic, every "it's just one more query." These add up faster than you think.
