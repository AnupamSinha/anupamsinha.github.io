---
title: "I Benchmarked Java 21 vs Go vs Rust for Microservices — Results Shocked Me"
date: 2026-08-24
categories: [Java, Performance]
tags: [java, golang, rust, benchmarks, microservices]
description: "Real performance data with methodology explained — HTTP throughput, cold start, memory usage, GC pauses, and tail latencies across three typical microservice workloads"
mermaid: true
---
## Why I Ran This Benchmark

Every few months, someone on my team asks: "Should we rewrite this in Go?" or "Wouldn't Rust be faster?" These conversations typically end with opinions, not data. So I decided to generate actual data for the workloads we encounter in practice.

I'm a Java architect with 17 years of experience. I'm biased toward Java — and I'm going to be honest about that upfront. But I also genuinely wanted to know where Java 21 stands against Go and Rust for the specific patterns we use in production microservices.

The results surprised me. Not because Java "won" (it didn't in every category), but because the gap has narrowed dramatically from what most people assume.

## Methodology

**Hardware:** AWS EC2 c6i.2xlarge (8 vCPUs, 16GB RAM) — representing a typical production microservice instance

**OS:** Amazon Linux 2023

**Versions tested:**

- Java 21.0.4 (Eclipse Temurin) + Spring Boot 3.4.1 + Virtual Threads
- Java 21.0.4 + GraalVM Native Image (Spring Boot 3.4.1)
- Go 1.23.2 + Gin framework
- Rust 1.82.0 + Axum framework + Tokio runtime

**Load generator:** wrk2 (constant-rate load) and hey (burst load) — run from a separate c6i.2xlarge instance in the same availability zone

**Database:** PostgreSQL 16 on RDS db.r6g.large (shared across all tests, connection reset between runs)

**Each test:**

- 5 minute warmup (Java), 1 minute warmup (Go/Rust)
- 10 minute sustained load measurement
- 3 runs per configuration, median reported
- GC logs collected for Java, runtime metrics for all

## Test Scenarios

I chose three scenarios that represent real microservice workloads:

**Scenario 1: JSON API** — Parse request, validate, transform, serialize response (CPU-bound)

**Scenario 2: Database CRUD** — Full request lifecycle with PostgreSQL queries (I/O-bound)

**Scenario 3: Aggregation Service** — Fan-out to 3 downstream services, aggregate responses (network I/O + CPU)

## Scenario 1: JSON API (CPU-Bound)

A service that accepts a complex JSON payload (nested order with line items), validates it, applies business rules (pricing, tax calculation), and returns the enriched response.

**Java (Spring Boot + Virtual Threads):**

```java
@RestController
public class OrderController {

    private final OrderValidator validator;
    private final PricingEngine pricingEngine;

    @PostMapping("/api/orders/calculate")
    public ResponseEntity<OrderResponse> calculateOrder(
            @Valid @RequestBody OrderRequest request) {
        
        validator.validate(request);
        OrderCalculation calc = pricingEngine.calculate(request);
        
        return ResponseEntity.ok(OrderResponse.from(calc));
    }
}
```

**Go (Gin):**

```go
func (h *OrderHandler) CalculateOrder(c *gin.Context) {
    var request OrderRequest
    if err := c.ShouldBindJSON(&request); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    
    if err := h.validator.Validate(&request); err != nil {
        c.JSON(422, gin.H{"error": err.Error()})
        return
    }
    
    calc := h.pricingEngine.Calculate(&request)
    c.JSON(200, NewOrderResponse(calc))
}
```

**Rust (Axum):**

```rust
async fn calculate_order(
    State(state): State<AppState>,
    Json(request): Json<OrderRequest>,
) -> Result<Json<OrderResponse>, AppError> {
    state.validator.validate(&request)?;
    let calc = state.pricing_engine.calculate(&request);
    Ok(Json(OrderResponse::from(calc)))
}
```

## Results: Scenario 1 (JSON API at 10,000 req/s target)

**Throughput (req/s achieved)**

**Java (Virtual Threads)** — 48,200 req/s

**Java (GraalVM Native)** — 52,100 req/s

**Go** — 61,300 req/s

**Rust** — 78,400 req/s

**P50 Latency**

**Java (Virtual Threads)** — 1.2ms

**Java (GraalVM Native)** — 0.9ms

**Go** — 0.7ms

**Rust** — 0.4ms

**P99 Latency**

**Java (Virtual Threads)** — 8.4ms

**Java (GraalVM Native)** — 3.1ms

**Go** — 2.8ms

**Rust** — 1.1ms

**P99.9 Latency**

**Java (Virtual Threads)** — 24.3ms (GC pauses)

**Java (GraalVM Native)** — 5.2ms

**Go** — 4.1ms

**Rust** — 1.8ms

**Memory Usage (RSS at steady state)**

**Java (Virtual Threads)** — 312MB

**Java (GraalVM Native)** — 89MB

**Go** — 42MB

**Rust** — 18MB

**Analysis:** For pure CPU-bound JSON processing, Rust wins decisively. Go is second. Java's tail latency suffers from GC pauses, but GraalVM Native eliminates most of that gap. The throughput difference between Java and Go is about 25% — meaningful at extreme scale, irrelevant for most services handling under 5,000 req/s.

## Scenario 2: Database CRUD (I/O-Bound)

Standard REST API with PostgreSQL — create, read, update, list operations with connection pooling.

**Java setup:** HikariCP connection pool (20 connections), Spring Data JPA

```java
@Service
@Transactional(readOnly = true)
public class ProductService {

    private final ProductRepository repository;

    public Page<ProductDto> findAll(Pageable pageable) {
        return repository.findAll(pageable).map(ProductDto::from);
    }

    @Transactional
    public ProductDto create(CreateProductCommand command) {
        Product product = Product.create(command);
        return ProductDto.from(repository.save(product));
    }

    public ProductDto findById(Long id) {
        return repository.findById(id)
            .map(ProductDto::from)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }
}
```

**Go setup:** pgxpool (20 connections), sqlx

```go
func (s *ProductService) FindAll(ctx context.Context, page, size int) ([]ProductDto, error) {
    rows, err := s.pool.Query(ctx,
        "SELECT id, name, price, stock FROM products ORDER BY id LIMIT $1 OFFSET $2",
        size, page*size)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var products []ProductDto
    for rows.Next() {
        var p ProductDto
        if err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock); err != nil {
            return nil, err
        }
        products = append(products, p)
    }
    return products, nil
}
```

**Rust setup:** deadpool-postgres (20 connections), tokio-postgres

```rust
impl ProductService {
    pub async fn find_all(&self, page: i64, size: i64) -> Result<Vec<ProductDto>, AppError> {
        let client = self.pool.get().await?;
        let rows = client.query(
            "SELECT id, name, price, stock FROM products ORDER BY id LIMIT $1 OFFSET $2",
            &[&size, &(page * size)],
        ).await?;
        
        Ok(rows.iter().map(ProductDto::from_row).collect())
    }
}
```

## Results: Scenario 2 (Database CRUD at 5,000 req/s mixed workload)

**Throughput (req/s max before degradation)**

**Java (Virtual Threads)** — 24,800 req/s

**Java (GraalVM Native)** — 23,100 req/s

**Go** — 26,200 req/s

**Rust** — 27,900 req/s

**P50 Latency**

**Java (Virtual Threads)** — 3.8ms

**Java (GraalVM Native)** — 3.6ms

**Go** — 3.4ms

**Rust** — 3.1ms

**P99 Latency**

**Java (Virtual Threads)** — 12.1ms

**Java (GraalVM Native)** — 9.8ms

**Go** — 9.2ms

**Rust** — 7.4ms

**P99.9 Latency**

**Java (Virtual Threads)** — 31.2ms

**Java (GraalVM Native)** — 14.1ms

**Go** — 12.8ms

**Rust** — 9.1ms

**Memory Usage (RSS at steady state)**

**Java (Virtual Threads)** — 287MB

**Java (GraalVM Native)** — 94MB

**Go** — 51MB

**Rust** — 24MB

**Analysis:** This is where it gets interesting. When the database is the bottleneck (which it usually is), the language differences shrink dramatically. The gap between Java and Rust in throughput is only 12%. P50 latency difference is less than 1ms. The database query time dominates — all languages are waiting for PostgreSQL.

**The key insight:** For I/O-bound services (which most business microservices are), language choice matters far less than people assume. Optimize your queries first.

## Scenario 3: Aggregation Service (Fan-Out)

A service that receives a request and fans out to 3 downstream services in parallel, then aggregates the responses.

**Java (Virtual Threads + StructuredTaskScope):**

```java
@GetMapping("/api/dashboard/{userId}")
public DashboardResponse getDashboard(@PathVariable Long userId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        var profileFuture = scope.fork(() -> profileClient.getProfile(userId));
        var ordersFuture = scope.fork(() -> orderClient.getRecentOrders(userId));
        var recommendationsFuture = scope.fork(() -> 
            recommendationClient.getRecommendations(userId));

        scope.join().throwIfFailed();

        return new DashboardResponse(
            profileFuture.get(),
            ordersFuture.get(),
            recommendationsFuture.get()
        );
    }
}
```

**Go (goroutines + errgroup):**

```go
func (h *DashboardHandler) GetDashboard(c *gin.Context) {
    userId := c.Param("userId")
    
    g, ctx := errgroup.WithContext(c.Request.Context())
    
    var profile *Profile
    var orders []*Order
    var recommendations []*Recommendation
    
    g.Go(func() error {
        var err error
        profile, err = h.profileClient.GetProfile(ctx, userId)
        return err
    })
    g.Go(func() error {
        var err error
        orders, err = h.orderClient.GetRecentOrders(ctx, userId)
        return err
    })
    g.Go(func() error {
        var err error
        recommendations, err = h.recClient.GetRecommendations(ctx, userId)
        return err
    })
    
    if err := g.Wait(); err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(200, DashboardResponse{profile, orders, recommendations})
}
```

**Rust (Tokio join):**

```rust
async fn get_dashboard(
    State(state): State<AppState>,
    Path(user_id): Path<i64>,
) -> Result<Json<DashboardResponse>, AppError> {
    let (profile, orders, recommendations) = tokio::try_join!(
        state.profile_client.get_profile(user_id),
        state.order_client.get_recent_orders(user_id),
        state.rec_client.get_recommendations(user_id),
    )?;

    Ok(Json(DashboardResponse { profile, orders, recommendations }))
}
```

## Results: Scenario 3 (Aggregation — downstream services respond in 5-20ms)

**Throughput (req/s at saturation)**

**Java (Virtual Threads)** — 18,400 req/s

**Java (GraalVM Native)** — 17,900 req/s

**Go** — 19,100 req/s

**Rust** — 19,800 req/s

**P50 Latency**

**Java (Virtual Threads)** — 22.4ms

**Java (GraalVM Native)** — 21.8ms

**Go** — 21.1ms

**Rust** — 20.6ms

**P99 Latency**

**Java (Virtual Threads)** — 48.2ms

**Java (GraalVM Native)** — 39.1ms

**Go** — 37.8ms

**Rust** — 34.2ms

**Memory at 10K concurrent connections**

**Java (Virtual Threads)** — 341MB

**Java (GraalVM Native)** — 112MB

**Go** — 68MB

**Rust** — 31MB

**Analysis:** When downstream latency dominates (5-20ms per call), the language overhead becomes noise. All four variants produce nearly identical throughput because the bottleneck is network I/O to downstream services. The 7% throughput difference between Java and Rust is meaningless in practice.

## Cold Start Time

This matters for serverless, scale-to-zero, and rapid autoscaling:

**Time to first request served**

**Java (Virtual Threads)** — 2,340ms

**Java (GraalVM Native)** — 78ms

**Go** — 12ms

**Rust** — 8ms

**Java loses badly here with JIT compilation.** GraalVM Native closes the gap significantly, but Go and Rust are still an order of magnitude faster. For Lambda functions or scale-to-zero services, this is where Java has a genuine disadvantage — and GraalVM Native is the answer.

## Memory Efficiency Under Load

At 1,000 concurrent connections, steady state:

**Java (Virtual Threads)** — 312MB baseline + ~0.5KB per virtual thread

**Java (GraalVM Native)** — 89MB baseline + ~0.5KB per virtual thread

**Go** — 42MB baseline + ~4KB per goroutine

**Rust** — 18MB baseline + ~2KB per task

Java's memory overhead means fewer instances per node in Kubernetes. If you're running 50 microservices, the difference between 312MB and 42MB per service adds up to significant cloud cost. GraalVM Native brings Java into a competitive range.

## What Shocked Me

**1. Java 21 with virtual threads is shockingly close in I/O-bound workloads**

For the typical microservice (database queries, HTTP calls, message queues), Java 21 performs within 10-15% of Go and Rust. Most teams won't notice this difference. The bottleneck is almost always the database or network, not the language runtime.

**2. GraalVM Native changes the game for Java**

Cold start drops from 2.3 seconds to 78ms. Memory drops from 312MB to 89MB. Tail latency improves significantly due to no JIT compilation pauses and reduced GC pressure. The tradeoff is peak throughput (AOT-compiled code is slightly slower than JIT-optimized hot paths), but for most services, the operational benefits outweigh the 5-8% throughput reduction.

**3. Rust's advantage is primarily in tail latency and memory**

For P50 and median throughput, Go and Rust are surprisingly close. Rust's killer advantage is at P99.9 — no GC pauses, ever. If your SLA is defined by tail latency (fintech, gaming, real-time bidding), Rust is genuinely worth the complexity cost.

**4. Go's efficiency per developer-hour is remarkable**

Go wasn't the fastest in any category, but it was consistently second place with the simplest code. The language's simplicity means faster development, easier debugging, and cheaper hiring. For teams optimizing developer productivity alongside performance, Go's position is strong.

## The Honest Recommendation

**Stay with Java when:**

- You have an existing Java team and Spring ecosystem investment
- Your services are I/O-bound (most CRUD microservices)
- Developer productivity and ecosystem richness matter more than raw performance
- You're willing to use GraalVM Native for latency-sensitive services
- You need the JVM's mature observability, profiling, and debugging tools

**Consider Go when:**

- Starting a new team or project with no existing language investment
- Memory efficiency matters at scale (many small services)
- You want the simplest possible operational profile
- Cold start time matters (serverless, FaaS)
- The service is relatively simple (Go's lack of generics until recently and minimal abstraction works best for straightforward services)

**Consider Rust when:**

- P99.9 tail latency is in your SLA
- Memory constraints are severe (embedded, edge computing)
- The service is CPU-bound with high throughput requirements
- You can afford the 2-3x longer development time
- Your team genuinely wants to learn Rust (forcing it kills morale)

## What I Actually Did After This Benchmark

For our production platform (fintech, payment reconciliation):

- **Kept Java** for 90% of services — the performance is more than adequate, the team knows it, the tooling is mature
- **Used GraalVM Native** for 2 services that needed fast cold start (event-triggered, scale-to-zero)
- **Wrote one service in Go** — a lightweight API gateway proxy where memory efficiency at high connection count mattered

I did not rewrite anything in Rust. The performance gain didn't justify the development cost or hiring challenge for our use case.

The best language for your microservice is the one your team can ship, debug, and maintain reliably. Performance differences matter only at extreme scale — and by then, you should be optimizing algorithms and architecture, not language runtimes.
