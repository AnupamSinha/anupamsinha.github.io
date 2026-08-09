---
layout: single
title: "Java 18 to Java 21 Migration: What Changed and What You Need to Know"
date: 2024-08-09
categories: [java]
tags: [java, java21, migration, virtual-threads, pattern-matching, lts]
description: "A version-by-version breakdown of the features that matter, with code examples to get you migrating today."
header:
  overlay_color: "#0d1b2a"
  overlay_filter: "0.7"
  overlay_image: https://images.unsplash.com/photo-1515879218367-8466d910auj8?w=1600
  teaser: https://images.unsplash.com/photo-1515879218367-8466d910auj8?w=400
excerpt: "Java 21 is the most significant LTS release since Java 8. Here's everything that changed from 18 to 21 — version by version, with real code examples."
toc: true
toc_sticky: true
toc_label: "Table of Contents"
toc_icon: "java"
---

Java 21 is a Long-Term Support (LTS) release — the first since Java 17. If you're still on Java 18, 19, or 20 (all short-term releases), migrating to 21 gives you years of support, security patches, and the culmination of features that were in preview across those intermediate versions.

Let's walk through what each version introduced and what's now production-ready in 21.

---

## Java 18 (March 2022) — The Stepping Stone

![Java 18](https://images.unsplash.com/photo-1461749280684-dccba630e2f6?w=800){: .align-center}

### Simple Web Server (JEP 408)

A built-in, zero-dependency HTTP file server for prototyping and testing:

```bash
jwebserver --port 8080 --directory /path/to/files
```

Or programmatically:

```java
var server = SimpleFileServer.createFileServer(
    new InetSocketAddress(8080),
    Path.of("/tmp/www"),
    OutputLevel.INFO
);
server.start();
```

### UTF-8 by Default (JEP 400)

No more charset surprises across platforms. `Charset.defaultCharset()` now returns UTF-8 on all operating systems.

{: .notice--warning}
**Migration note**: If your code relied on platform-specific default charset (e.g., Windows-1252 on older Windows), you may need to explicitly specify the charset where needed.

### Code Snippets in JavaDoc (JEP 413)

Replace old `<pre>{@code ...}</pre>` with cleaner syntax:

```java
/**
 * Example usage:
 * {@snippet :
 *     var list = List.of("one", "two", "three");
 *     list.forEach(System.out::println);
 * }
 */
```

---

## Java 19 (September 2022) — Previews Take Shape

### Virtual Threads (Preview — JEP 425)

The first look at Project Loom's virtual threads:

```java
// Preview in 19, finalized in 21
Thread.startVirtualThread(() -> {
    System.out.println("Running on: " + Thread.currentThread());
});
```

### Structured Concurrency (Incubator — JEP 428)

A new model for managing multiple concurrent tasks as a single unit of work.

### Record Patterns (Preview — JEP 405)

Deconstruct records directly in pattern matching:

```java
record Point(int x, int y) {}

if (obj instanceof Point(int x, int y)) {
    System.out.println("x = " + x + ", y = " + y);
}
```

---

## Java 20 (March 2023) — Refinement Cycle

Java 20 was primarily about refining preview features. No major new finals, but important second previews for:

- Virtual Threads (Second Preview)
- Record Patterns (Second Preview)
- Pattern Matching for Switch (Fourth Preview)
- Scoped Values (Incubator)

{: .notice--info}
**Migration note**: If you adopted preview features in 19, check for API changes in 20. The refinements were subtle but breaking in some cases.

---

## Java 21 (September 2023) — The LTS Destination

![Java 21 LTS](https://images.unsplash.com/photo-1504639725590-34d0984388bd?w=800){: .align-center}

Everything comes together. Here's what's now production-ready:

### Virtual Threads — Finalized (JEP 444)

The headline feature. Lightweight threads managed by the JVM, not the OS.

```java
// Create 100,000 virtual threads without breaking a sweat
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i ->
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        })
    );
}
```

{: .notice--success}
**Why it matters**: Traditional threads are expensive (~1MB stack each). Virtual threads are cheap (few KB). You can now use simple blocking I/O code without worrying about thread pool exhaustion.

**Migration action**: Replace `Executors.newFixedThreadPool()` in I/O-heavy services with `Executors.newVirtualThreadPerTaskExecutor()`. No reactive frameworks needed for scalability.

### Record Patterns — Finalized (JEP 440)

Deconstruct record values directly in `instanceof` and `switch`:

```java
record Address(String city, String country) {}
record Customer(String name, Address address) {}

// Nested deconstruction
if (customer instanceof Customer(String name, Address(String city, var country))) {
    System.out.println(name + " lives in " + city);
}
```

### Pattern Matching for Switch — Finalized (JEP 441)

No more chains of `instanceof` checks:

```java
// Before (Java 17 style)
if (obj instanceof String s) {
    process(s);
} else if (obj instanceof Integer i) {
    process(i);
} else if (obj instanceof null) {
    handleNull();
}

// After (Java 21)
switch (obj) {
    case String s  -> process(s);
    case Integer i -> process(i);
    case null      -> handleNull();
    default        -> handleUnknown(obj);
}
```

**Guarded patterns** with `when`:

```java
switch (shape) {
    case Circle c when c.radius() > 10 -> "large circle";
    case Circle c                       -> "small circle";
    case Rectangle r                    -> "rectangle";
    default                             -> "unknown";
}
```

### Sequenced Collections (JEP 431)

New interfaces: `SequencedCollection`, `SequencedSet`, `SequencedMap`.

```java
// Before — inconsistent across collection types
list.get(0);                    // first element of List
set.iterator().next();          // first element of SortedSet
deque.getFirst();               // first element of Deque

// After — unified API
collection.getFirst();
collection.getLast();
collection.reversed();
```

### Sealed Classes with Pattern Matching — Full Power

Combine sealed classes (finalized in 17) with switch pattern matching (finalized in 21):

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}
record Circle(double radius) implements Shape {}
record Rectangle(double w, double h) implements Shape {}
record Triangle(double base, double height) implements Shape {}

double area(Shape shape) {
    return switch (shape) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.w() * r.h();
        case Triangle t  -> 0.5 * t.base() * t.height();
        // No default needed — compiler knows it's exhaustive
    };
}
```

---

## Migration Checklist: Java 18 → Java 21

| Step | Action |
|------|--------|
| 1 | Update build tool (`pom.xml` or `build.gradle`) source/target to 21 |
| 2 | Update CI/CD base image to JDK 21 (e.g., `eclipse-temurin:21-jdk`) |
| 3 | Run full build — fix any removed/deprecated API usage |
| 4 | Check for `SecurityManager` usage — deprecated for removal |
| 5 | Replace thread pools with virtual threads in I/O-heavy services |
| 6 | Refactor `instanceof` chains to switch pattern matching |
| 7 | Adopt `SequencedCollection` where you access first/last elements |
| 8 | Remove `--enable-preview` flags for features now finalized |
| 9 | Update dependencies (Spring Boot 3.2+, Hibernate 6.4+ support Java 21) |
| 10 | Run tests, performance benchmarks, and verify GC behavior |

---

## Common Pitfalls

### 1. Virtual threads + synchronized blocks

Virtual threads can get pinned to carrier threads inside `synchronized` blocks. Prefer `ReentrantLock` for critical sections in virtual-thread-heavy code:

```java
// ❌ Avoid with virtual threads
synchronized (lock) {
    blockingIOCall();
}

// ✅ Prefer
private final ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    blockingIOCall();
} finally {
    lock.unlock();
}
```

### 2. Preview API changes between versions

If you used record patterns in Java 19, note that named patterns were removed in later previews. Always check the final JEP spec, not the preview version.

### 3. Third-party library compatibility

Some libraries (Lombok, Byte Buddy, older ASM versions) may need updates for Java 21 bytecode. Update them before upgrading the JDK.

---

## Should You Skip Straight from 17 to 21?

If you're still on Java 17 (the previous LTS), absolutely go directly to 21. Features from 18, 19, and 20 that survived are all included in 21. There's no need to stop at intermediate versions.

---

## Wrapping Up

Java 21 is the most significant LTS release since Java 8. Virtual threads alone change how you think about concurrency. Pattern matching makes your code more expressive. Sequenced collections fix a long-standing API gap.

The migration from 18 to 21 is relatively smooth — no major breaking changes, mostly additive features. The biggest wins come from adopting the new patterns, not just bumping the version number.

---

*Found this useful? Share it with your team. Got questions? Drop a comment below.*
