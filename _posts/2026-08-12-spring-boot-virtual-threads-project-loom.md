---
title: "Spring Boot with Virtual Threads — Project Loom in Practice"
date: 2026-08-12
categories: [Java, Spring]
tags: [spring-boot, virtual-threads, project-loom, java-21, concurrency, performance, reactive, backend]
description: "Learn how to enable and use virtual threads in Spring Boot. Understand what Project Loom changes, when virtual threads help, and how to migrate from platform threads with practical examples."
image:
  path: /assets/img/posts/developer_activity_bv83.svg
  alt: Spring Boot Virtual Threads
---

## What Problem Does Project Loom Solve?

Traditional Java concurrency uses **platform threads** — each one maps 1:1 to an OS thread. They're expensive (~1MB stack each) and limited. A typical server tops out around 200-500 concurrent threads.

This creates the **thread-per-request bottleneck**: if your app handles 10,000 concurrent requests and each waits on a database call, you need 10,000 threads. That's not feasible with platform threads, which is exactly why reactive frameworks (WebFlux, RxJava) were created — they avoid blocking threads entirely.

**Virtual threads** solve this differently. They're lightweight (~1KB), managed by the JVM, and can exist in the millions. When a virtual thread blocks on I/O, the JVM unmounts it from the carrier (platform) thread, freeing that thread to run other virtual threads.

> You get the simplicity of blocking code with the scalability of non-blocking code.
{: .prompt-info }

---

## The Before and After

### Before: Platform Threads (Thread-Per-Request)

```
10,000 requests → 10,000 platform threads → OOM or thread pool exhaustion
```

### After: Virtual Threads

```
10,000 requests → 10,000 virtual threads → ~30 carrier threads doing the actual work
```

Same blocking code. Same readability. Massively better scalability.

---

## Enabling Virtual Threads in Spring Boot

### Requirements

- **Java 21+** (virtual threads are a final feature since JDK 21)
- **Spring Boot 3.2+** (first-class virtual thread support)

### One Property — That's It

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

This single property switches Tomcat's thread pool to use virtual threads for request handling. Every incoming HTTP request now runs on a virtual thread instead of a platform thread.

---

## What Happens Under the Hood

When you set `spring.threads.virtual.enabled=true`, Spring Boot configures:

1. **Tomcat's executor** to use `Executors.newVirtualThreadPerTaskExecutor()`
2. **Async task execution** (`@Async`) to use virtual threads
3. **Scheduling** (`@Scheduled`) to use virtual threads

```java
// What Spring Boot configures for you
@Bean
public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
    return protocolHandler -> {
        protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
    };
}
```

---

## Writing Code with Virtual Threads

The best part? **Your code doesn't change.** Regular blocking code now scales:

### REST Controller — Unchanged

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final OrderService orderService;
    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;

    public OrderController(OrderService orderService, PaymentClient paymentClient,
                           InventoryClient inventoryClient) {
        this.orderService = orderService;
        this.paymentClient = paymentClient;
        this.inventoryClient = inventoryClient;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody OrderRequest request) {
        // These blocking calls are now cheap — virtual thread gets unmounted during I/O wait
        InventoryStatus inventory = inventoryClient.check(request.items());   // blocks ~50ms
        PaymentResult payment = paymentClient.charge(request.paymentInfo());  // blocks ~200ms
        Order order = orderService.save(request, inventory, payment);         // blocks ~20ms

        return ResponseEntity.status(HttpStatus.CREATED).body(OrderResponse.from(order));
    }
}
```

With platform threads, those three blocking calls would hold an expensive thread for ~270ms. With virtual threads, the carrier thread is freed during each I/O wait.

---

## Structured Concurrency (Preview in Java 21+)

When you need to call multiple services in parallel:

```java
@Service
public class OrderService {

    public OrderSummary getOrderSummary(Long orderId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

            Subtask<Order> orderTask = scope.fork(() -> orderRepository.findById(orderId).orElseThrow());
            Subtask<CustomerProfile> customerTask = scope.fork(() -> customerClient.getProfile(orderId));
            Subtask<List<ShipmentStatus>> shipmentTask = scope.fork(() -> shipmentClient.getStatus(orderId));

            scope.join();           // Wait for all
            scope.throwIfFailed();  // Propagate exceptions

            return new OrderSummary(
                orderTask.get(),
                customerTask.get(),
                shipmentTask.get()
            );
        }
    }
}
```

All three calls run concurrently on virtual threads. If one fails, the scope cancels the others. Clean, readable, no CompletableFuture chains.

---

## When Virtual Threads Help (and When They Don't)

### Great for (I/O-bound workloads)

- REST APIs calling downstream services
- Database queries
- File I/O operations
- Message queue consumers
- Any thread-per-request server handling many concurrent connections

### Not helpful for (CPU-bound workloads)

- Heavy computation (image processing, encryption)
- Tight loops without I/O
- Already non-blocking code (WebFlux/Reactor)

### The Rule of Thumb

> If your threads spend most of their time **waiting**, virtual threads help. If they spend most of their time **computing**, they don't.
{: .prompt-tip }

---

## Things to Watch Out For

### 1. Synchronized Blocks Pin the Carrier Thread

```java
// BAD — pins the carrier thread during I/O inside synchronized
synchronized (lock) {
    database.query();  // Virtual thread cannot unmount here
}

// GOOD — use ReentrantLock instead
lock.lock();
try {
    database.query();  // Virtual thread can unmount normally
} finally {
    lock.unlock();
}
```

### 2. ThreadLocal Overhead

Virtual threads are cheap to create but `ThreadLocal` storage is per-thread. With millions of virtual threads, heavy `ThreadLocal` usage causes memory pressure.

```java
// Be cautious — this creates a value per virtual thread
private static final ThreadLocal<ExpensiveObject> cache = ThreadLocal.withInitial(ExpensiveObject::new);
```

Consider scoped values (preview feature) as a replacement:

```java
private static final ScopedValue<RequestContext> CONTEXT = ScopedValue.newInstance();

ScopedValue.runWhere(CONTEXT, new RequestContext(user), () -> {
    // Available to this virtual thread and any it spawns
    processRequest();
});
```

### 3. Connection Pool Sizing

With platform threads, 200 threads and a 200-connection pool was balanced. With virtual threads, you might have 10,000 concurrent requests but your database can only handle 200 connections.

```yaml
# Ensure your connection pool is the bottleneck you're aware of
spring:
  datasource:
    hikari:
      maximum-pool-size: 50  # Don't set this to 10,000!
```

---

## Benchmarking: Platform vs Virtual Threads

A simple benchmark with 5,000 concurrent requests to an endpoint that sleeps 200ms (simulating I/O):

| Metric | Platform Threads (200 pool) | Virtual Threads |
|--------|---------------------------|-----------------|
| Throughput | ~950 req/s | ~4,800 req/s |
| Avg Latency | 1,050ms | 210ms |
| P99 Latency | 4,200ms | 250ms |
| Memory | ~600MB | ~180MB |

The platform thread pool creates queuing. Virtual threads eliminate it.

---

## Migration Checklist

- [ ] Upgrade to Java 21+ and Spring Boot 3.2+
- [ ] Add `spring.threads.virtual.enabled: true`
- [ ] Audit `synchronized` blocks — replace with `ReentrantLock` where they guard I/O
- [ ] Review `ThreadLocal` usage — consider scoped values
- [ ] Verify connection pool sizes are appropriately bounded
- [ ] Remove reactive code that was only there for scalability (if desired)
- [ ] Load test to validate improvement

---

## Should You Ditch WebFlux?

Not necessarily. Virtual threads eliminate the primary reason most teams adopted WebFlux (scalability without thread exhaustion). But WebFlux still has advantages:

- **Backpressure** built into the reactive streams contract
- **Streaming responses** (SSE, WebSocket handling)
- **Existing codebase** — migrating working reactive code isn't free

For new projects, virtual threads with blocking code give you 90% of the scalability benefit with much simpler code. For existing WebFlux apps that work well, there's no urgent reason to rewrite.

---

> Virtual threads don't make your code faster — they make your server handle more concurrent requests by eliminating thread pool queuing. The individual request latency stays the same; aggregate throughput improves dramatically.
{: .prompt-warning }
