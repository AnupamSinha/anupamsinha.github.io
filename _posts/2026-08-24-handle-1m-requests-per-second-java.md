---
title: "How I Handle 1 Million Requests/Second with Java"
date: 2026-08-24
categories: [Spring Boot, Performance]
tags: [java, performance, high-throughput, spring-boot, scalability]
description: "A performance engineering deep-dive covering virtual threads, connection pooling, async I/O, caching layers, read replicas, message batching, zero-copy serialization, and load shedding — with real JMH benchmarks and before/after metrics."
mermaid: true
---
## The Starting Point

In 2022, our fintech platform in Singapore was handling 50,000 requests per second. The business told us to prepare for 1 million. I remember thinking: "We'll just add more instances." That naive assumption cost us three months of wasted effort before we got serious about performance engineering.

After 17 years of building Java systems, I've learned that horizontal scaling alone doesn't get you to 1M RPS. You need fundamental changes in how your application processes requests. This is the story of how we got there — the techniques, the trade-offs, and the actual numbers.

## The Architecture Before Optimization

Our starting point was a typical Spring Boot microservice:

- Spring Boot 3.1 with traditional thread-per-request model
- PostgreSQL with HikariCP (default settings)
- Redis for session management
- Kafka for async processing
- Running on 12 instances (4 vCPU, 8GB RAM each)

**Baseline metrics (per instance):**

**Throughput** — 4,200 RPS

**P99 latency** — 340ms

**Thread count** — 200 platform threads

**CPU utilization** — 78%

**GC pause (P99)** — 45ms

Total cluster throughput: ~50,000 RPS. We needed 20x improvement without 20x the infrastructure cost.

## Technique 1: Virtual Threads (Project Loom)

The single biggest win came from migrating to Java 21 virtual threads. Our thread-per-request model was burning platform threads on I/O waits — database calls, Redis lookups, HTTP calls to downstream services.

### The Migration

```java
@Configuration
public class VirtualThreadConfig {

    @Bean
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
        return protocolHandler -> {
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        };
    }

    @Bean
    public AsyncTaskExecutor applicationTaskExecutor() {
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }
}
```

In Spring Boot 3.2+, it's even simpler:

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

### The Gotcha: Pinned Threads

Virtual threads get "pinned" to carrier threads inside `synchronized` blocks. We found this in our connection pool wrapper:

```java
// BAD: synchronized pins the virtual thread
public synchronized Connection getConnection() {
    return pool.borrowConnection();
}

// GOOD: ReentrantLock doesn't pin
private final ReentrantLock lock = new ReentrantLock();

public Connection getConnection() {
    lock.lock();
    try {
        return pool.borrowConnection();
    } finally {
        lock.unlock();
    }
}
```

We used `-Djdk.tracePinnedThreads=short` during testing to catch all pinning points.

### JMH Benchmark: Thread Model Comparison

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 5, time = 2)
@Measurement(iterations = 10, time = 5)
public class ThreadModelBenchmark {

    private static final int SIMULATED_IO_MS = 10;

    @Benchmark
    @Threads(200)
    public void platformThreads_200(Blackhole bh) throws InterruptedException {
        Thread.sleep(SIMULATED_IO_MS); // Simulate I/O
        bh.consume(processRequest());
    }

    @Benchmark
    public void virtualThreads_unlimited(Blackhole bh) throws Exception {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            var futures = IntStream.range(0, 10_000)
                .mapToObj(i -> executor.submit(() -> {
                    Thread.sleep(SIMULATED_IO_MS);
                    return processRequest();
                }))
                .toList();

            for (var future : futures) {
                bh.consume(future.get());
            }
        }
    }

    private String processRequest() {
        return "processed";
    }
}
```

**Results:**

**Platform threads (200)** — 18,500 ops/sec

**Virtual threads (unlimited)** — 142,000 ops/sec

**Improvement** — 7.7x throughput

## Technique 2: Connection Pool Tuning

Default HikariCP settings are conservative. With virtual threads handling thousands of concurrent requests, the connection pool becomes the bottleneck.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 50
      minimum-idle: 50
      connection-timeout: 3000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 30000
```

### The Formula

The optimal pool size isn't "as many as possible." It's:

**Pool Size = (Core Count * 2) + Effective Spindle Count**

For our 4-core instances with SSDs: `(4 * 2) + 1 = 9` for a single database. But with read replicas (covered later), we split this:

- **Write pool** — 10 connections
- **Read pool** — 40 connections (spread across 4 replicas)

### Custom Connection Pool with Routing

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource routingDataSource(
            @Qualifier("writeDataSource") DataSource writeDs,
            @Qualifier("readDataSource") DataSource readDs) {

        var routingDs = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                    ? "READ" : "WRITE";
            }
        };

        routingDs.setTargetDataSources(Map.of("WRITE", writeDs, "READ", readDs));
        routingDs.setDefaultTargetDataSource(writeDs);
        return routingDs;
    }

    @Bean("readDataSource")
    public DataSource readDataSource() {
        var config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://read-replica:5432/app");
        config.setMaximumPoolSize(40);
        config.setReadOnly(true);
        return new HikariDataSource(config);
    }
}
```

**Impact:** Read latency dropped from 12ms to 3ms as we eliminated write-lock contention on read queries.

## Technique 3: Multi-Layer Response Caching

We implemented three caching layers, each serving a different access pattern:

```java
@Service
public class ProductService {

    private final Cache<String, Product> localCache;  // Caffeine L1
    private final RedisTemplate<String, Product> redisTemplate;  // Redis L2
    private final ProductRepository repository;  // Database L3

    public ProductService(RedisTemplate<String, Product> redisTemplate,
                          ProductRepository repository) {
        this.localCache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofSeconds(30))
            .recordStats()
            .build();
        this.redisTemplate = redisTemplate;
        this.repository = repository;
    }

    public Product getProduct(String id) {
        // L1: In-process cache (sub-microsecond)
        Product product = localCache.getIfPresent(id);
        if (product != null) return product;

        // L2: Distributed cache (sub-millisecond)
        product = redisTemplate.opsForValue().get("product:" + id);
        if (product != null) {
            localCache.put(id, product);
            return product;
        }

        // L3: Database (milliseconds)
        product = repository.findById(id).orElseThrow();
        redisTemplate.opsForValue().set("product:" + id, product, Duration.ofMinutes(5));
        localCache.put(id, product);
        return product;
    }
}
```

### Cache Hit Rates (Production)

**L1 (Caffeine)** — 72% hit rate, < 1μs latency

**L2 (Redis)** — 23% hit rate, 0.3ms latency

**L3 (Database)** — 5% of requests, 4ms latency

**Effective read latency** — 0.25ms (weighted average)

## Technique 4: Async I/O for Non-Critical Paths

Not every operation needs to be synchronous. We identified three categories:

- **Critical path** — Must complete before response (payment validation)
- **Important but deferrable** — Can fail independently (audit logging, analytics)
- **Fire and forget** — Best effort (metrics, notifications)

```java
@Service
public class OrderProcessingService {

    private final ExecutorService asyncExecutor = 
        Executors.newVirtualThreadPerTaskExecutor();
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Transactional
    public OrderResponse processOrder(OrderRequest request) {
        // Critical path — synchronous
        Order order = validateAndCreateOrder(request);
        PaymentResult payment = chargePayment(order);

        // Fire-and-forget — async, non-blocking
        asyncExecutor.submit(() -> auditLog(order, payment));
        asyncExecutor.submit(() -> updateAnalytics(order));

        // Important but deferrable — via Kafka
        kafkaTemplate.send("order-events",
            order.getId(),
            new OrderEvent(order, OrderStatus.CONFIRMED));

        return new OrderResponse(order.getId(), payment.getTransactionId());
    }
}
```

**Impact:** P99 response time dropped from 340ms to 85ms by moving non-critical work off the request path.

## Technique 5: Message Batching

Instead of sending individual Kafka messages, we batch them:

```java
@Component
public class BatchingEventPublisher {

    private final BlockingQueue<OrderEvent> buffer = new LinkedBlockingQueue<>(10_000);
    private final KafkaTemplate<String, List<OrderEvent>> kafkaTemplate;

    @Scheduled(fixedDelay = 50) // Flush every 50ms
    public void flushBatch() {
        List<OrderEvent> batch = new ArrayList<>(500);
        buffer.drainTo(batch, 500); // Max 500 per batch

        if (!batch.isEmpty()) {
            kafkaTemplate.send("order-events-batch", batch)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("Batch send failed, requeueing {} events", batch.size(), ex);
                        buffer.addAll(batch); // Requeue on failure
                    }
                });
        }
    }

    public void publish(OrderEvent event) {
        if (!buffer.offer(event)) {
            log.warn("Event buffer full, applying backpressure");
            throw new ServiceOverloadedException("Event buffer full");
        }
    }
}
```

**Impact:** Kafka producer throughput increased from 15,000 to 180,000 messages/sec. Network calls reduced by 98%.

## Technique 6: Zero-Copy Serialization

JSON serialization was consuming 12% of our CPU. We switched hot paths to Protocol Buffers with zero-copy:

```protobuf
syntax = "proto3";

message ProductResponse {
    string id = 1;
    string name = 2;
    int64 price_cents = 3;
    int32 stock_quantity = 4;
    int64 last_updated_epoch = 5;
}
```

```java
@Configuration
public class ProtobufConfig {

    @Bean
    public ProtobufHttpMessageConverter protobufHttpMessageConverter() {
        return new ProtobufHttpMessageConverter();
    }
}

@RestController
@RequestMapping("/api/v2/products")
public class ProductControllerV2 {

    @GetMapping(value = "/{id}", produces = "application/x-protobuf")
    public ProductResponse getProduct(@PathVariable String id) {
        Product product = productService.getProduct(id);
        return ProductResponse.newBuilder()
            .setId(product.getId())
            .setName(product.getName())
            .setPriceCents(product.getPriceCents())
            .setStockQuantity(product.getStockQuantity())
            .setLastUpdatedEpoch(product.getLastUpdated().toEpochMilli())
            .build();
    }
}
```

### JMH Benchmark: Serialization Comparison

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
public class SerializationBenchmark {

    private Product product;
    private ObjectMapper jackson;

    @Setup
    public void setup() {
        product = createSampleProduct();
        jackson = new ObjectMapper();
    }

    @Benchmark
    public byte[] jackson_serialize() throws Exception {
        return jackson.writeValueAsBytes(product);
    }

    @Benchmark
    public byte[] protobuf_serialize() {
        return toProtobuf(product).toByteArray();
    }
}
```

**Results:**

**Jackson JSON** — 1.2M ops/sec, avg payload 847 bytes

**Protobuf** — 8.4M ops/sec, avg payload 312 bytes

**Improvement** — 7x faster serialization, 63% smaller payloads

## Technique 7: Load Shedding

At scale, you must protect your system from itself. When approaching capacity, it's better to reject some requests cleanly than to degrade everyone's experience.

```java
@Component
public class LoadSheddingFilter implements WebFilter {

    private final AtomicInteger activeRequests = new AtomicInteger(0);
    private final int maxConcurrentRequests;
    private final MeterRegistry meterRegistry;

    public LoadSheddingFilter(
            @Value("${app.max-concurrent-requests:5000}") int maxConcurrent,
            MeterRegistry meterRegistry) {
        this.maxConcurrentRequests = maxConcurrent;
        this.meterRegistry = meterRegistry;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        int current = activeRequests.incrementAndGet();

        if (current > maxConcurrentRequests) {
            activeRequests.decrementAndGet();
            meterRegistry.counter("requests.shed").increment();

            exchange.getResponse().setStatusCode(HttpStatus.SERVICE_UNAVAILABLE);
            exchange.getResponse().getHeaders().set("Retry-After", "1");
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange)
            .doFinally(signal -> activeRequests.decrementAndGet());
    }
}
```

### Priority-Based Shedding

Not all requests are equal. Paying customers get priority:

```java
@Component
public class PriorityLoadShedder {

    private final Semaphore highPriority = new Semaphore(4000);
    private final Semaphore lowPriority = new Semaphore(1000);

    public <T> T execute(RequestPriority priority, Supplier<T> task) {
        Semaphore semaphore = priority == RequestPriority.HIGH ? highPriority : lowPriority;

        if (!semaphore.tryAcquire()) {
            throw new LoadSheddedException(
                "System at capacity for priority: " + priority);
        }

        try {
            return task.get();
        } finally {
            semaphore.release();
        }
    }
}
```

## Technique 8: Database Read Replicas

We configured PostgreSQL with 4 read replicas and routed read-only transactions automatically:

```java
@Service
public class ProductQueryService {

    private final ProductRepository productRepository;

    @Transactional(readOnly = true) // Routes to read replica
    public Page<Product> searchProducts(String query, Pageable pageable) {
        return productRepository.findByNameContaining(query, pageable);
    }

    @Transactional(readOnly = true)
    public Optional<Product> findById(String id) {
        return productRepository.findById(id);
    }
}
```

The routing datasource (shown earlier) intercepts `readOnly = true` and sends those queries to replicas. This gave us 5x read capacity without touching a single query.

## The Final Numbers

After implementing all eight techniques across 6 weeks:

**Cluster size** — 12 instances → 8 instances (33% reduction)

**Throughput (per instance)** — 4,200 RPS → 132,000 RPS

**Total cluster throughput** — 50K RPS → 1,056,000 RPS

**P99 latency** — 340ms → 28ms

**P50 latency** — 45ms → 4ms

**CPU utilization** — 78% → 52%

**Monthly infrastructure cost** — Reduced by 41%

## Impact Breakdown by Technique

**Virtual threads** — 7.7x throughput improvement

**Connection pool tuning** — 3x reduction in connection wait time

**Multi-layer caching** — 95% of reads served without DB hit

**Async I/O** — 75% reduction in P99 latency

**Message batching** — 12x Kafka throughput improvement

**Protobuf serialization** — 7x serialization speed, 63% less bandwidth

**Load shedding** — Zero cascading failures under peak load

**Read replicas** — 5x read capacity

## Lessons Learned

- **Measure first, optimize second.** We wasted two weeks optimizing JSON serialization before realizing I/O waits were 80% of our latency. Profiling with async-profiler would have shown this in minutes.

- **Virtual threads aren't magic.** They're transformative for I/O-bound workloads, but they won't help CPU-bound processing. Know your bottleneck.

- **Cache invalidation is harder than caching.** Our L1 Caffeine cache occasionally served stale data for up to 30 seconds. For most use cases, that was acceptable. For pricing, we used pub/sub invalidation.

- **Load shedding saved us on launch day.** Traffic spiked to 1.4M RPS (40% above plan). Without load shedding, we would have had a cascading failure. Instead, 6% of low-priority requests got a clean 503 with a retry header.

- **Don't optimize everything.** Only 3 of our 47 endpoints needed all these techniques. The other 44 run fine on default settings. Profile-guided optimization beats premature optimization every time.

## Where to Start

If you're just beginning this journey:

1. **Enable virtual threads** — Biggest bang for buck if you're I/O-bound
2. **Add Caffeine L1 cache** — Trivial to implement, immediate impact
3. **Tune your connection pool** — Most apps leave default settings untouched
4. **Profile with async-profiler** — Know your actual bottleneck before optimizing

The rest becomes relevant once you're pushing past 100K RPS.
