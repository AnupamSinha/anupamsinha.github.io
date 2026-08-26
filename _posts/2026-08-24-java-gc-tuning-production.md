---
title: "Java Memory Management: GC Tuning That Actually Works in Production"
date: 2026-08-24
categories: [Java, Performance]
tags: [java, performance, jvm, garbage-collection, spring-boot]
description: "A production-focused guide to JVM garbage collection — choosing between G1/ZGC/Shenandoah, heap sizing for containers, GC log analysis, finding memory leaks in Spring Boot, and JFR profiling techniques that solved real outages."
mermaid: true
---
## Why Textbook GC Theory Fails in Production

Every Java developer learns about Young Gen, Old Gen, and the basic collector algorithms. That knowledge helps you pass interviews. It doesn't help you at 2 AM when your production service is doing 4-second GC pauses and your SLAs are burning.

In 17 years of running Java services in Singapore — fintech platforms processing millions of transactions, e-commerce systems handling flash sales — I've learned that GC tuning is empirical. You measure, you adjust, you measure again. There is no universal "best" configuration.

But there are patterns. There are collectors that fit specific workloads. There are container-specific traps that catch everyone once. And there are diagnostic techniques that find the root cause in minutes instead of days.

---

## Choosing Your Collector: G1 vs ZGC vs Shenandoah

### G1 (Garbage First) — The Safe Default

G1 has been the default collector since Java 9. It works well for most workloads without tuning.

**Best for:**
- Heap sizes 4-32 GB
- Applications where 50-200ms pause times are acceptable
- Teams that don't want to become GC experts
- Mixed workloads (moderate allocation rate, moderate live set)

**Key JVM flags:**

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m
-XX:InitiatingHeapOccupancyPercent=45
```

**What G1 does well:** It divides the heap into regions and collects the regions with the most garbage first (hence the name). It gives you a pause-time target, and it mostly hits it.

**Where G1 struggles:** If your live data set is large (>50% of heap), G1 spends more time in mixed collections. If your allocation rate is extremely high, G1's concurrent marking can't keep up, triggering full GCs.

### ZGC — Sub-Millisecond Pauses

ZGC is a low-latency collector that keeps pauses under 1ms regardless of heap size. Production-ready since Java 15.

**Best for:**
- Large heaps (32 GB to 16 TB)
- Latency-sensitive applications (trading systems, real-time APIs)
- Applications where P99 latency matters more than throughput
- Services with large live data sets

**Key JVM flags:**

```bash
-XX:+UseZGC
-XX:+ZGenerational          # Java 21+ — generational ZGC, significantly better
-Xmx16g
-XX:SoftMaxHeapSize=12g     # ZGC will try to stay under this
```

**The tradeoff:** ZGC uses more CPU for concurrent work and more memory (colored pointers use extra metadata). Throughput is 5-15% lower than G1 for batch workloads. For latency-sensitive services, that's a worthwhile trade.

### Shenandoah — The Low-Pause Alternative

Shenandoah offers similar pause times to ZGC with different implementation trade-offs. Available in OpenJDK (not Oracle JDK builds).

**Best for:**
- Teams on Red Hat/Adoptium builds
- Workloads similar to ZGC's sweet spot
- When ZGC's memory overhead is unacceptable

**Key JVM flags:**

```bash
-XX:+UseShenandoahGC
-XX:ShenandoahGCHeuristics=adaptive
-Xmx8g
```

### My Decision Process

**Start with G1.** If P99 latency is over your SLA, check GC logs. If GC pauses are the cause, switch to ZGC (Java 21+ with generational mode). Only reach for Shenandoah if you're on a distribution that doesn't ship ZGC or you have specific memory overhead constraints.

---

## Heap Sizing: The Art of Not Too Big, Not Too Small

### The Golden Ratio

Your heap should be 3-4× your live data set. The live data set is the memory consumed after a full GC (all garbage collected, only reachable objects remain).

**Finding your live data set:**

```bash
# Force a full GC and check heap usage after
jcmd <pid> GC.run
jcmd <pid> GC.heap_info
```

Or from GC logs, look at the heap occupancy after a Full GC:

```
[Full GC (Metadata GC Threshold) 1.2G->450M(4G), 0.8s]
```

Live set = 450 MB. Target heap = 450 × 3.5 ≈ 1.6 GB minimum.

### Why Too Big Is Bad

"Just give it 32 GB" seems safe. But:
- Larger heaps mean longer full GC pauses when they do happen
- More memory pressure on the OS, less page cache for file I/O
- In containers, you're paying for memory you don't use
- Compressed oops (object pointers) stop working above ~32 GB, inflating object headers

### Why Too Small Is Bad

- Frequent GC cycles consume CPU
- Promotion failures trigger stop-the-world Full GCs
- Under memory pressure, G1 abandons concurrent marking and falls back to serial Full GC

### The Container Trap

```bash
# WRONG: JVM doesn't know about cgroup limits (older Java versions)
docker run -m 4g myapp:latest -Xmx4g  # OOM killed!
```

The JVM process uses heap + metaspace + thread stacks + native memory + GC overhead. If you set `-Xmx` equal to the container limit, the process exceeds the limit and gets OOM-killed by the kernel.

**The correct approach:**

```bash
# Leave 25-30% headroom for non-heap memory
docker run -m 4g myapp:latest \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=75.0
```

Or explicitly:

```bash
docker run -m 4g myapp:latest \
  -Xms3g -Xmx3g \
  -XX:MaxMetaspaceSize=256m \
  -XX:ReservedCodeCacheSize=256m \
  -Xss512k
```

**Memory budget for a 4 GB container:**
- **Heap** — 3 GB
- **Metaspace** — 256 MB (class metadata)
- **Thread stacks** — 200 threads × 512 KB = 100 MB
- **Code cache** — 256 MB (JIT compiled code)
- **Native/Direct** — remaining ~400 MB
- **Total** — ~4 GB (fits within cgroup limit)

### Java 17+ Container Awareness

Modern Java (10+) automatically detects container limits:

```bash
# Java reads cgroup limits and adjusts ergonomics
java -XX:+PrintFlagsFinal -version 2>&1 | grep -i "maxheapsize"
```

But I still recommend explicit sizing. "Automatic" means the JVM picks defaults that may not match your workload.

---

## GC Logging: Your Production Diagnostic Lifeline

Always enable GC logging in production. The overhead is negligible (<1% CPU). The diagnostic value when things go wrong is enormous.

### Java 17+ Unified GC Logging

```bash
-Xlog:gc*=info:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=10,filesize=100m
```

This gives you:
- **gc*** — all GC-related events
- **info level** — enough detail without flooding
- **Timestamps** — correlate with application events
- **File rotation** — 10 files × 100 MB = 1 GB max disk usage

### Reading GC Logs — What to Look For

**Healthy G1 output:**

```
[2024-03-15T10:23:45.123+0800][gc] GC(1234) Pause Young (Normal) (G1 Evacuation Pause) 
    2048M->512M(4096M) 12.345ms
```

**Key fields:**
- **Pause Young (Normal)** — regular young generation collection
- **2048M->512M** — heap went from 2 GB used to 512 MB used
- **12.345ms** — pause time (under 200ms target = healthy)

**Danger signs:**

```
[gc] GC(5678) Pause Full (Allocation Failure) 3800M->3200M(4096M) 4.567s
```

- **Pause Full** — stop-the-world full collection (bad)
- **Allocation Failure** — young gen couldn't find space (very bad)
- **3800M->3200M** — only 600 MB freed (live set is huge, possible leak)
- **4.567s** — 4.5 second pause (SLA is dead)

### GC Log Analysis Tools

Don't read raw logs. Use:
- **GCEasy** (gceasy.io) — upload logs, get visual analysis
- **GCViewer** — open source desktop tool
- **JClarity Censum** — now part of Microsoft (for Azure)

### Metrics I Alert On

```java
// Expose via Micrometer for Prometheus/Grafana
@Configuration
public class GcMetricsConfig {

    @Bean
    MeterBinder gcMetrics() {
        return new JvmGcMetrics();  // Spring Boot auto-configures this with Actuator
    }
}
```

**Alert thresholds I use:**
- **GC pause > 500ms** — warning
- **GC pause > 2s** — critical
- **Full GC count > 0 per hour** — investigate
- **Heap after GC > 80% of max** — memory leak probable
- **GC time > 5% of wall clock** — throughput degrading

---

## Finding Memory Leaks in Spring Boot

### The Usual Suspects

**1. Unbounded caches without eviction:**

```java
// LEAK: This grows forever
private final Map<String, Object> cache = new HashMap<>();

public Object getCached(String key) {
    return cache.computeIfAbsent(key, k -> expensiveComputation(k));
}
```

Fix: Use Caffeine with size/time bounds:

```java
private final Cache<String, Object> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(30))
    .recordStats()
    .build();
```

**2. Event listeners never unregistered:**

```java
// LEAK: Every request registers a listener, never removed
@Component
public class DangerousComponent {

    @Autowired
    private ApplicationEventPublisher publisher;

    public void processRequest(Request req) {
        // This anonymous class holds a reference to 'req' forever
        publisher.publishEvent(new CustomEvent(this, req));
    }
}
```

**3. ThreadLocal not cleaned in servlet containers:**

```java
// LEAK in Tomcat thread pool
private static final ThreadLocal<List<AuditEntry>> auditLog = 
    ThreadLocal.withInitial(ArrayList::new);

public void handleRequest() {
    auditLog.get().add(new AuditEntry(...));
    // If you never call auditLog.remove(), the list grows across requests
    // because Tomcat reuses threads
}
```

**4. JPA session cache (first-level cache):**

```java
// LEAK: Processing millions of rows in one transaction
@Transactional
public void processAllOrders() {
    List<Order> orders = orderRepository.findAll(); // Loads EVERYTHING into memory
    for (Order order : orders) {
        order.setStatus("PROCESSED");
        // JPA keeps all entities in persistence context
    }
}
```

Fix: Use pagination and flush/clear:

```java
@Transactional
public void processAllOrders() {
    int page = 0;
    Page<Order> batch;
    do {
        batch = orderRepository.findAll(PageRequest.of(page++, 1000));
        for (Order order : batch) {
            order.setStatus("PROCESSED");
        }
        entityManager.flush();
        entityManager.clear(); // Release entities from persistence context
    } while (batch.hasNext());
}
```

**5. Classpath scanning / reflection caches:**

Spring Boot's component scanning and reflection utilities cache class metadata. In applications with dynamic classloading (plugin systems, scripting engines), this can leak metaspace.

### Diagnosing with Heap Dumps

```bash
# Trigger heap dump (safe in production — just causes a GC pause)
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# Or automatically on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/heap-dump.hprof
```

Open with Eclipse MAT (Memory Analyzer Tool):
1. Look at **Dominator Tree** — what objects retain the most memory
2. Check **Leak Suspects** report — MAT auto-identifies likely leaks
3. Examine **Retained Heap** — the total memory freed if this object were garbage collected

---

## JFR Profiling: The Production-Safe Profiler

Java Flight Recorder (JFR) is built into the JVM. It has <2% overhead and is designed for always-on production use.

### Starting JFR

```bash
# Start recording for 5 minutes
jcmd <pid> JFR.start name=profile duration=5m filename=/tmp/profile.jfr

# Or configure at startup for continuous recording
-XX:StartFlightRecording=disk=true,maxsize=500m,maxage=24h,dumponexit=true,filename=/var/log/app/flight.jfr
```

### What JFR Captures

- **GC events** — every collection with before/after heap sizes
- **CPU profiling** — method-level hot spots sampled every 10ms
- **Allocation profiling** — which methods allocate the most
- **Thread events** — contention, parking, sleeping
- **I/O events** — file and socket reads/writes with duration
- **Exceptions** — every exception thrown (even caught ones)

### Analyzing with JDK Mission Control

```bash
# Open JFR recording in JMC
jmc /tmp/profile.jfr
```

Or programmatically with the JFR streaming API (Java 14+):

```java
@Component
public class JfrMonitor {

    @PostConstruct
    public void startMonitoring() {
        try (var stream = new RecordingStream()) {
            stream.enable("jdk.GCPhasePause").withThreshold(Duration.ofMillis(100));
            stream.enable("jdk.CPULoad").withPeriod(Duration.ofSeconds(5));

            stream.onEvent("jdk.GCPhasePause", event -> {
                Duration pause = event.getDuration("duration");
                if (pause.toMillis() > 200) {
                    log.warn("Long GC pause: {}ms", pause.toMillis());
                    alertService.notify("GC_LONG_PAUSE", pause.toMillis());
                }
            });

            stream.onEvent("jdk.CPULoad", event -> {
                float jvmUser = event.getFloat("jvmUser");
                if (jvmUser > 0.8) {
                    log.warn("High CPU: {}%", jvmUser * 100);
                }
            });

            stream.startAsync();
        }
    }
}
```

### The Allocation Hot Path

The most actionable JFR analysis for GC issues is the allocation profile. It shows which methods create the most garbage:

**Common offenders in Spring Boot:**
- **JSON serialization** — Jackson creates intermediate `byte[]` arrays
- **String concatenation in loops** — use StringBuilder
- **Autoboxing** — `int` to `Integer` in collections
- **Stream().collect()** — intermediate collections
- **Logging with string formatting** — `log.debug("Got " + obj)` allocates even when DEBUG is off

Fix: Use parameterized logging:

```java
// BAD: Allocates string even if debug is disabled
log.debug("Processing order " + orderId + " with " + items.size() + " items");

// GOOD: No allocation if debug is disabled
log.debug("Processing order {} with {} items", orderId, items.size());
```

---

## Fixing OOM in Containers

### The OOM Kill vs Java OOM

**Java OOM (OutOfMemoryError):** The JVM tried to allocate memory and the heap was full. Application gets the error. You can catch it (though you usually shouldn't).

**Container OOM Kill (OOMKilled):** The kernel cgroup enforcer terminated the process because total RSS exceeded the container memory limit. No exception. No graceful shutdown. Process is dead.

### Diagnosing Which One Happened

```bash
# Check if container was OOM killed
kubectl describe pod myapp-xyz | grep -A5 "Last State"
# Or
docker inspect <container_id> | grep OOMKilled
```

If it's a container OOM kill but your heap is within limits, the leak is in native memory:

**Native memory consumers:**
- **Direct ByteBuffers** — NIO allocations outside heap
- **Thread stacks** — each thread uses 512KB-1MB
- **Metaspace** — class metadata (grows with dynamic class generation)
- **JIT code cache** — compiled methods
- **Native libraries** — JNI, compression (zlib), SSL

### Tracking Native Memory

```bash
# Enable Native Memory Tracking (5-10% overhead)
-XX:NativeMemoryTracking=detail

# Check usage
jcmd <pid> VM.native_memory summary
```

Output:

```
Total: reserved=4500MB, committed=3200MB
-                 Java Heap (reserved=3072MB, committed=3072MB)
-                     Class (reserved=256MB, committed=180MB)
-                    Thread (reserved=200MB, committed=200MB)
-                      Code (reserved=256MB, committed=150MB)
-                        GC (reserved=300MB, committed=300MB)
-                  Internal (reserved=50MB, committed=50MB)
-                    Symbol (reserved=20MB, committed=20MB)
-    Native Memory Tracking (reserved=10MB, committed=10MB)
```

If committed total > container limit, you'll get OOM killed.

### Production Configuration Template

Here's the configuration I use for Spring Boot services in Kubernetes with a 4 GB container:

```bash
java \
  -Xms2g -Xmx2g \
  -XX:+UseZGC \
  -XX:+ZGenerational \
  -XX:MaxMetaspaceSize=256m \
  -XX:ReservedCodeCacheSize=256m \
  -XX:MaxDirectMemorySize=256m \
  -Xss512k \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/app/heap-dump.hprof \
  -Xlog:gc*=info:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=50m \
  -XX:+ExitOnOutOfMemoryError \
  -XX:NativeMemoryTracking=summary \
  -XX:StartFlightRecording=disk=true,maxsize=200m,maxage=12h \
  -jar app.jar
```

**Memory budget:**
- **Heap** — 2 GB
- **Metaspace** — 256 MB
- **Code cache** — 256 MB
- **Direct memory** — 256 MB
- **Thread stacks** — 200 threads × 512 KB = 100 MB
- **GC overhead** — ~300 MB (ZGC needs extra)
- **Buffer** — ~600 MB for spikes and native libs
- **Total** — ~3.8 GB (within 4 GB cgroup limit)

---

## Quick Wins That Always Help

**1. Set -Xms equal to -Xmx**
Eliminates heap resizing pauses. The JVM starts with full heap and never grows/shrinks.

**2. Use -XX:+ExitOnOutOfMemoryError**
In containers, a half-alive JVM after OOM is worse than a clean restart. Let it die, let Kubernetes reschedule.

**3. Enable string deduplication for G1**
```bash
-XX:+UseStringDeduplication  # G1 only — deduplicates identical strings
```
Can save 5-15% heap in applications with many duplicate strings (HTTP headers, JSON field names).

**4. Tune Metaspace for Spring Boot**
Spring apps load many classes. Set metaspace explicitly to avoid resizing:
```bash
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m
```

**5. Right-size your thread pools**
Each thread costs 512KB-1MB of stack. 500 threads = 500 MB just for stacks. Use async I/O (WebFlux, virtual threads in Java 21) to reduce thread count.

---

## The Tuning Loop

1. **Baseline** — Run under realistic load, collect GC logs and JFR data
2. **Identify** — What's the problem? Latency? Throughput? OOMs?
3. **Hypothesize** — "If I increase heap / switch collectors / reduce allocation, it should improve"
4. **Change one thing** — Never tune multiple parameters simultaneously
5. **Measure** — Same load, compare before/after
6. **Iterate** — Repeat until SLAs are met

GC tuning is not a one-time activity. Your traffic patterns change. Your data grows. Your code evolves. Revisit your GC configuration quarterly, or whenever latency SLAs start slipping.

The best GC configuration is the one you understand, can explain, and can adjust when the workload changes.
