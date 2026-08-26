---
title: "Java Concurrency Patterns Every Senior Developer Must Know"
date: 2026-08-24
categories: [Java, Fundamentals]
tags: [java, concurrency, multithreading, spring-boot, software-engineering]
description: "A practical guide to thread safety patterns that survive production traffic — covering immutability, concurrent collections, CompletableFuture, rate limiting, and atomic operations with real bug examples."
mermaid: true
---
## Why Most Concurrency Code Fails in Production

After 17 years of building high-throughput systems in Singapore's fintech and e-commerce space, I've reviewed thousands of pull requests. The pattern is consistent: concurrency bugs don't show up in unit tests. They show up at 3 AM when traffic spikes hit your payment service.

The problem isn't that developers don't understand `synchronized`. It's that they reach for the wrong tool. They use heavy locks where atomic operations suffice. They share mutable state when immutability solves the problem entirely. They write CompletableFuture chains that silently swallow exceptions.

This post covers the patterns I actually use in production. Not textbook theory — working code that handles real load.

---

## Pattern 1: Immutability as Your First Line of Defense

The cheapest concurrency fix is eliminating shared mutable state entirely. If an object can't change after construction, no thread can corrupt it.

```java
public final class PricingContext {
    private final String currency;
    private final BigDecimal exchangeRate;
    private final Instant validUntil;
    private final List<String> applicableDiscounts;

    public PricingContext(String currency, BigDecimal exchangeRate,
                          Instant validUntil, List<String> applicableDiscounts) {
        this.currency = currency;
        this.exchangeRate = exchangeRate;
        this.validUntil = validUntil;
        this.applicableDiscounts = List.copyOf(applicableDiscounts); // defensive copy
    }

    public String getCurrency() { return currency; }
    public BigDecimal getExchangeRate() { return exchangeRate; }
    public Instant getValidUntil() { return validUntil; }
    public List<String> getApplicableDiscounts() { return applicableDiscounts; }
}
```

Key rules for immutability:

- **Mark the class `final`** — prevents subclasses from adding mutable state
- **Make all fields `final`** — compiler enforces single assignment
- **Defensive copy collections** — `List.copyOf()` returns an unmodifiable copy
- **No setters, ever** — if you need a modified version, return a new object

With Java 16+ records, this gets even simpler:

```java
public record PricingContext(
    String currency,
    BigDecimal exchangeRate,
    Instant validUntil,
    List<String> applicableDiscounts
) {
    public PricingContext {
        applicableDiscounts = List.copyOf(applicableDiscounts);
    }
}
```

I pass immutable objects between threads without any synchronization. Zero locks, zero bugs, zero performance overhead.

---

## Pattern 2: ConcurrentHashMap vs Synchronized — Choosing Correctly

I see this bug constantly in code reviews:

```java
// BUG: check-then-act is NOT atomic
private final Map<String, UserSession> sessions = new ConcurrentHashMap<>();

public UserSession getOrCreateSession(String userId) {
    if (!sessions.containsKey(userId)) {        // Thread A checks here
        sessions.put(userId, new UserSession()); // Thread B also passes check, creates duplicate
    }
    return sessions.get(userId);
}
```

`ConcurrentHashMap` makes individual operations thread-safe, but compound operations (check-then-act) are still racy. The fix:

```java
public UserSession getOrCreateSession(String userId) {
    return sessions.computeIfAbsent(userId, k -> new UserSession());
}
```

`computeIfAbsent` is atomic — the lambda executes at most once per key under the segment lock.

### When to Use ConcurrentHashMap

- High read-to-write ratio (caches, session stores)
- Independent keys (no cross-key invariants)
- You need concurrent iteration without `ConcurrentModificationException`

### When synchronized Collections Are Better

```java
// When you need atomic compound operations across multiple keys
private final Map<String, Account> accounts = Collections.synchronizedMap(new HashMap<>());

public void transfer(String from, String to, BigDecimal amount) {
    synchronized (accounts) {
        Account source = accounts.get(from);
        Account target = accounts.get(to);
        source.debit(amount);
        target.credit(amount);
    }
}
```

If your invariant spans multiple keys (like a bank transfer), `ConcurrentHashMap` won't help. You need explicit synchronization over the entire operation.

### The ConcurrentHashMap.compute Pitfall

```java
// DANGER: Never do blocking I/O inside compute
sessions.compute(userId, (key, existing) -> {
    // This holds a segment lock!
    UserProfile profile = userService.fetchFromDatabase(key); // BLOCKING CALL
    return new UserSession(profile);
});
```

The lambda in `compute` runs while holding a hash bucket lock. Blocking I/O here will throttle your entire map. Fetch the data first, then compute.

---

## Pattern 3: CompletableFuture Composition for Async Pipelines

Spring Boot services often need to call multiple downstream APIs. Sequential calls kill latency. CompletableFuture lets you parallelize:

```java
@Service
public class OrderEnrichmentService {

    private final UserServiceClient userClient;
    private final InventoryServiceClient inventoryClient;
    private final PricingServiceClient pricingClient;

    public CompletableFuture<EnrichedOrder> enrichOrder(Order order) {
        CompletableFuture<UserProfile> userFuture =
            CompletableFuture.supplyAsync(() -> userClient.getProfile(order.getUserId()));

        CompletableFuture<InventoryStatus> inventoryFuture =
            CompletableFuture.supplyAsync(() -> inventoryClient.checkStock(order.getSkus()));

        CompletableFuture<PricingResult> pricingFuture =
            CompletableFuture.supplyAsync(() -> pricingClient.calculate(order));

        return CompletableFuture.allOf(userFuture, inventoryFuture, pricingFuture)
            .thenApply(ignored -> new EnrichedOrder(
                order,
                userFuture.join(),
                inventoryFuture.join(),
                pricingFuture.join()
            ));
    }
}
```

### The Silent Exception Killer

The most common CompletableFuture bug: exceptions vanish silently.

```java
// BUG: Exception is swallowed — nobody sees it
CompletableFuture.supplyAsync(() -> riskyOperation())
    .thenApply(result -> transform(result));
// If riskyOperation throws, the future completes exceptionally
// but nobody handles it
```

Always attach exception handling:

```java
CompletableFuture.supplyAsync(() -> riskyOperation())
    .thenApply(result -> transform(result))
    .exceptionally(ex -> {
        log.error("Operation failed", ex);
        return fallbackValue();
    });
```

Or better, use `handle` for both success and failure:

```java
CompletableFuture.supplyAsync(() -> callExternalApi())
    .orTimeout(5, TimeUnit.SECONDS)
    .handle((result, ex) -> {
        if (ex != null) {
            log.warn("API call failed, using cache", ex);
            return cachedValue();
        }
        return result;
    });
```

### Custom Thread Pools — Don't Use ForkJoinPool for I/O

```java
// BAD: ForkJoinPool.commonPool() is shared across your entire JVM
CompletableFuture.supplyAsync(() -> httpCall());

// GOOD: Dedicated pool for I/O operations
private static final ExecutorService IO_POOL =
    Executors.newFixedThreadPool(20, new ThreadFactoryBuilder()
        .setNameFormat("io-pool-%d")
        .setDaemon(true)
        .build());

CompletableFuture.supplyAsync(() -> httpCall(), IO_POOL);
```

The common ForkJoinPool has CPU-count threads. If all of them block on HTTP calls, your entire application stalls — including unrelated parallel streams.

---

## Pattern 4: Semaphore for Rate Limiting

When you need to limit concurrent access to a resource (database connection pool, third-party API with rate limits), `Semaphore` is cleaner than custom counter logic:

```java
@Component
public class RateLimitedApiClient {

    private final Semaphore semaphore = new Semaphore(10); // max 10 concurrent calls
    private final RestTemplate restTemplate;

    public ApiResponse callExternalApi(ApiRequest request) {
        boolean acquired = false;
        try {
            acquired = semaphore.tryAcquire(2, TimeUnit.SECONDS);
            if (!acquired) {
                throw new RateLimitExceededException(
                    "Could not acquire permit within timeout");
            }
            return restTemplate.postForObject("/api/endpoint", request, ApiResponse.class);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new ServiceException("Interrupted while waiting for rate limit", e);
        } finally {
            if (acquired) {
                semaphore.release();
            }
        }
    }
}
```

### Why not just use a connection pool?

Connection pools limit connections. Semaphores limit concurrency at any point in your code. I use them for:

- **Third-party API limits** — "Only 10 concurrent requests allowed"
- **Expensive computations** — limit CPU-bound work to avoid starving other threads
- **Bulkhead pattern** — isolate resource usage between services

### Fairness Matters

```java
// Fair semaphore: longest-waiting thread gets the permit first
private final Semaphore semaphore = new Semaphore(10, true);
```

Without fairness, under heavy load, some threads may starve. The fair version adds slight overhead but prevents starvation.

---

## Pattern 5: ReadWriteLock for Read-Heavy Workloads

If your data structure is read 95% of the time and written 5%, a single `synchronized` block is overkill. `ReadWriteLock` allows concurrent reads while still ensuring exclusive writes:

```java
@Component
public class ConfigurationCache {

    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private Map<String, String> configMap = new HashMap<>();

    public String getConfig(String key) {
        lock.readLock().lock();
        try {
            return configMap.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }

    public void refreshConfig(Map<String, String> newConfig) {
        lock.writeLock().lock();
        try {
            configMap = new HashMap<>(newConfig);
        } finally {
            lock.writeLock().unlock();
        }
    }

    public Map<String, String> getAllConfig() {
        lock.readLock().lock();
        try {
            return Map.copyOf(configMap); // return immutable snapshot
        } finally {
            lock.readLock().unlock();
        }
    }
}
```

Multiple threads can call `getConfig` simultaneously. Only `refreshConfig` blocks everyone.

### StampedLock — The Optimistic Alternative (Java 8+)

For extreme read performance, `StampedLock` offers optimistic reads:

```java
private final StampedLock lock = new StampedLock();
private double x, y;

public double distanceFromOrigin() {
    long stamp = lock.tryOptimisticRead();  // non-blocking
    double currentX = x, currentY = y;
    if (!lock.validate(stamp)) {            // check if write occurred
        stamp = lock.readLock();            // fall back to read lock
        try {
            currentX = x;
            currentY = y;
        } finally {
            lock.unlockRead(stamp);
        }
    }
    return Math.sqrt(currentX * currentX + currentY * currentY);
}
```

The optimistic read takes no lock at all. If no write happened during the read, you're done. In read-heavy scenarios, this eliminates locking overhead entirely.

---

## Pattern 6: Atomic Operations — Lock-Free Performance

For simple counters and accumulators, atomic classes outperform locks by using CPU-level compare-and-swap (CAS):

```java
@Component
public class MetricsCollector {

    private final AtomicLong requestCount = new AtomicLong(0);
    private final AtomicLong errorCount = new AtomicLong(0);
    private final LongAdder highThroughputCounter = new LongAdder(); // Java 8+

    public void recordRequest() {
        requestCount.incrementAndGet();
        highThroughputCounter.increment();
    }

    public void recordError() {
        errorCount.incrementAndGet();
    }

    public double getErrorRate() {
        long total = requestCount.get();
        if (total == 0) return 0.0;
        return (double) errorCount.get() / total;
    }

    public long getThroughput() {
        return highThroughputCounter.sum();
    }
}
```

### AtomicLong vs LongAdder

- **AtomicLong** — single CAS, good for low-to-moderate contention, exact real-time value
- **LongAdder** — spreads updates across cells, dramatically better under high contention, but `sum()` is eventually consistent

Use `LongAdder` for metrics counters where exact point-in-time accuracy isn't critical. Use `AtomicLong` when you need precise reads (like generating sequence numbers).

### AtomicReference for Lock-Free State Machines

```java
public class CircuitBreaker {

    private enum State { CLOSED, OPEN, HALF_OPEN }

    private final AtomicReference<State> state = new AtomicReference<>(State.CLOSED);
    private final AtomicInteger failureCount = new AtomicInteger(0);

    public boolean allowRequest() {
        return state.get() != State.OPEN;
    }

    public void recordFailure() {
        if (failureCount.incrementAndGet() >= 5) {
            state.compareAndSet(State.CLOSED, State.OPEN);
            // Only one thread transitions the state
        }
    }

    public void attemptReset() {
        state.compareAndSet(State.OPEN, State.HALF_OPEN);
    }
}
```

`compareAndSet` is the key — it succeeds only if the current value matches the expected value. No locks needed.

---

## Common Production Bugs I've Fixed

### Bug 1: The Lazy Singleton Race

```java
// BROKEN double-checked locking (pre-Java 5 semantics aside, still commonly botched)
public class ConnectionPool {
    private static ConnectionPool instance; // Missing volatile!

    public static ConnectionPool getInstance() {
        if (instance == null) {
            synchronized (ConnectionPool.class) {
                if (instance == null) {
                    instance = new ConnectionPool(); // partially constructed object visible
                }
            }
        }
        return instance;
    }
}
```

Without `volatile`, the JVM may reorder the write to `instance` before the constructor completes. Another thread sees a non-null but partially constructed object. Fix: add `volatile`, or better, use an enum or holder pattern:

```java
public class ConnectionPool {
    private static class Holder {
        static final ConnectionPool INSTANCE = new ConnectionPool();
    }

    public static ConnectionPool getInstance() {
        return Holder.INSTANCE;
    }
}
```

### Bug 2: Thread-Local Memory Leaks in Tomcat

```java
// LEAK: ThreadLocal not cleaned in a thread pool
private static final ThreadLocal<RequestContext> context = new ThreadLocal<>();

public void handleRequest(HttpServletRequest request) {
    context.set(new RequestContext(request));
    // ... process request
    // If an exception occurs before remove(), the context leaks
}
```

Tomcat reuses threads. If you don't clean up `ThreadLocal`, the old `RequestContext` (and everything it references) stays alive forever. Always use try-finally:

```java
public void handleRequest(HttpServletRequest request) {
    context.set(new RequestContext(request));
    try {
        processRequest();
    } finally {
        context.remove(); // ALWAYS clean up
    }
}
```

### Bug 3: Publishing `this` in Constructor

```java
// BUG: 'this' escapes before object is fully constructed
public class EventListener {
    private final List<String> events;

    public EventListener(EventBus bus) {
        bus.register(this); // Another thread can call onEvent before events is assigned!
        this.events = new ArrayList<>();
    }

    public void onEvent(String event) {
        events.add(event); // NullPointerException!
    }
}
```

Never pass `this` to another thread or register callbacks in a constructor. Use a factory method instead.

---

## My Concurrency Checklist for Code Reviews

- **Is the state truly shared?** If not, no synchronization needed
- **Can it be immutable?** If yes, use records or final fields
- **Is it a single atomic operation?** Use AtomicLong/AtomicReference
- **Is it read-heavy?** Use ReadWriteLock or StampedLock
- **Is it a cache/map?** Use ConcurrentHashMap with compute methods
- **Does it need rate limiting?** Use Semaphore
- **Is it async I/O?** Use CompletableFuture with dedicated pools
- **Does it hold locks during I/O?** Refactor immediately

Thread safety isn't about sprinkling `synchronized` everywhere. It's about choosing the right primitive for the right problem, and whenever possible, designing your way out of the problem entirely.
