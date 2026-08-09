---
title: "Java 18 to Java 21 Migration Guide — Features, Code Examples, Checklist"
date: 2026-08-09
categories: [Java, Migration]
tags: [java, java21, java-migration, lts, virtual-threads, pattern-matching, spring-boot, record-patterns, sequenced-collections, sealed-classes]
description: "Complete Java 18 to Java 21 LTS migration guide for developers. Covers virtual threads, pattern matching, record patterns, sequenced collections, sealed classes — with official JEP links, Spring Boot compatibility, code examples, and step-by-step migration checklist."
image:
  path: /assets/img/posts/version_control_9bpv.svg
  alt: Java 18 to Java 21 Migration Guide
pin: true
img_path: /assets/img/posts/
---

![Developer Activity](developer_activity_bv83.svg)
_Migrating your Java application to the latest LTS_

## Overview

| | Details |
|---|---|
| **Target** | Java 21 (LTS) |
| **Source Versions** | Java 18, 19, 20 |
| **Release Date** | September 19, 2023 |
| **LTS Support** | Oracle Premier Support until September 2028 |
| **Previous LTS** | Java 17 (September 2021) |
| **Official Release Notes** | [JDK 21 — OpenJDK](https://openjdk.org/projects/jdk/21/) |
| **Oracle Migration Guide** | [Migration Guide — Java SE 21](https://docs.oracle.com/en/java/javase/21/migrate/) |

---

## Why Migrate to Java 21?

Java 21 is the first Long-Term Support release since Java 17. Versions 18, 19, and 20 were short-term releases (6 months of support each). Migrating to 21 gives you:

- Years of security patches and support
- All finalized features that were in preview across 18–20
- Framework compatibility (Spring Boot 3.2+, Quarkus 3.6+, Micronaut 4.2+)
- Performance improvements in GC, JIT, and startup time

---

## Java 18 (March 2022) — Key Features

> Official release page: [JDK 18 — OpenJDK](https://openjdk.org/projects/jdk/18/)
{: .prompt-info }

### Simple Web Server — [JEP 408](https://openjdk.org/jeps/408)

A built-in, zero-dependency HTTP file server for prototyping and testing.

```bash
jwebserver --port 8080 --directory /path/to/files
```

Programmatic usage:

```java
var server = SimpleFileServer.createFileServer(
    new InetSocketAddress(8080),
    Path.of("/tmp/www"),
    OutputLevel.INFO
);
server.start();
```

### UTF-8 by Default — [JEP 400](https://openjdk.org/jeps/400)

`Charset.defaultCharset()` now returns UTF-8 on all operating systems. No more platform-specific charset surprises.

> If your code relied on platform-specific default charset (e.g., Windows-1252), you may need to explicitly specify charsets where needed.
{: .prompt-warning }

### Code Snippets in JavaDoc — [JEP 413](https://openjdk.org/jeps/413)

Replace verbose `<pre>{@code ...}</pre>` with cleaner syntax:

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

## Java 19 (September 2022) — Preview Features

> Official release page: [JDK 19 — OpenJDK](https://openjdk.org/projects/jdk/19/)
{: .prompt-info }

### Virtual Threads (Preview) — [JEP 425](https://openjdk.org/jeps/425)

First look at Project Loom's lightweight threads:

```java
Thread.startVirtualThread(() -> {
    System.out.println("Running on: " + Thread.currentThread());
});
```

### Structured Concurrency (Incubator) — [JEP 428](https://openjdk.org/jeps/428)

Manages multiple concurrent tasks as a single unit of work — improves error handling and cancellation.

### Record Patterns (Preview) — [JEP 405](https://openjdk.org/jeps/405)

Deconstruct records directly in pattern matching:

```java
record Point(int x, int y) {}

if (obj instanceof Point(int x, int y)) {
    System.out.println("x = " + x + ", y = " + y);
}
```

---

## Java 20 (March 2023) — Refinement Cycle

> Official release page: [JDK 20 — OpenJDK](https://openjdk.org/projects/jdk/20/)
{: .prompt-info }

Java 20 refined preview features from 19. No new finalized features, but important iterations:

| Feature | JEP | Status in 20 |
|---------|-----|--------------|
| Virtual Threads | [JEP 436](https://openjdk.org/jeps/436) | Second Preview |
| Record Patterns | [JEP 432](https://openjdk.org/jeps/432) | Second Preview |
| Pattern Matching for Switch | [JEP 433](https://openjdk.org/jeps/433) | Fourth Preview |
| Scoped Values | [JEP 429](https://openjdk.org/jeps/429) | Incubator |

> If you adopted preview features in 19, check for API changes. Refinements between previews can be subtle but breaking.
{: .prompt-warning }

---

## Java 21 (September 2023) — The LTS Destination

![Cloud Hosting](/assets/img/posts/cloud_hosting_aodd.svg){: width="450" }
_Everything comes together in Java 21_

> Official release page: [JDK 21 — OpenJDK](https://openjdk.org/projects/jdk/21/)
{: .prompt-info }

### Finalized Features Summary

| Feature | JEP | Category |
|---------|-----|----------|
| Virtual Threads | [JEP 444](https://openjdk.org/jeps/444) | Concurrency |
| Record Patterns | [JEP 440](https://openjdk.org/jeps/440) | Language |
| Pattern Matching for Switch | [JEP 441](https://openjdk.org/jeps/441) | Language |
| Sequenced Collections | [JEP 431](https://openjdk.org/jeps/431) | Collections |
| Generational ZGC | [JEP 439](https://openjdk.org/jeps/439) | GC |
| Key Encapsulation Mechanism API | [JEP 452](https://openjdk.org/jeps/452) | Security |

---

### Virtual Threads — [JEP 444](https://openjdk.org/jeps/444)

The headline feature. Lightweight threads managed by the JVM, not the OS.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i ->
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        })
    );
}
```

> Traditional threads cost ~1MB stack each. Virtual threads cost only a few KB. Use simple blocking I/O without worrying about thread pool exhaustion.
{: .prompt-tip }

**Migration action**: Replace `Executors.newFixedThreadPool()` in I/O-heavy services with `Executors.newVirtualThreadPerTaskExecutor()`.

**Further reading**:
- [Virtual Threads — Oracle Docs](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
- [Exploring Java's Virtual Threads — Oracle Blog](https://blogs.oracle.com/javamagazine/post/java-virtual-threads)

---

### Record Patterns — [JEP 440](https://openjdk.org/jeps/440)

Deconstruct record values directly in `instanceof` and `switch`:

```java
record Address(String city, String country) {}
record Customer(String name, Address address) {}

// Nested deconstruction
if (customer instanceof Customer(String name, Address(String city, var country))) {
    System.out.println(name + " lives in " + city);
}
```

---

### Pattern Matching for Switch — [JEP 441](https://openjdk.org/jeps/441)

No more chains of `instanceof` checks:

```java
// Before (Java 17 style)
if (obj instanceof String s) {
    process(s);
} else if (obj instanceof Integer i) {
    process(i);
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

**Oracle Docs**: [Pattern Matching for switch — Java SE 21](https://docs.oracle.com/en/java/javase/21/language/pattern-matching-switch.html)

---

### Sequenced Collections — [JEP 431](https://openjdk.org/jeps/431)

New interfaces: `SequencedCollection`, `SequencedSet`, `SequencedMap`.

```java
// Before — inconsistent
list.get(0);
set.iterator().next();
deque.getFirst();

// After — unified API
collection.getFirst();
collection.getLast();
collection.reversed();
```

---

### Sealed Classes + Pattern Matching

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

## Framework Compatibility

| Framework | Minimum Version for Java 21 | Reference |
|-----------|---------------------------|-----------|
| Spring Boot | 3.2+ | [Spring Boot 3.2 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.2-Release-Notes) |
| Spring Framework | 6.1+ | [Spring Framework 6.1](https://spring.io/blog/2023/11/16/spring-framework-6-1-goes-ga) |
| Hibernate | 6.4+ | [Hibernate ORM 6.4](https://hibernate.org/orm/releases/6.4/) |
| Quarkus | 3.6+ | [Quarkus](https://quarkus.io/) |
| Micronaut | 4.2+ | [Micronaut](https://micronaut.io/) |
| Lombok | 1.18.30+ | [Lombok Changelog](https://projectlombok.org/changelog) |
| JUnit 5 | 5.10+ | [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/) |

---

## Spring Boot + Virtual Threads

Spring Boot 3.2+ has built-in support for virtual threads. Enable with a single property:

```properties
# application.properties
spring.threads.virtual.enabled=true
```

This configures the embedded Tomcat/Jetty to use virtual threads for request handling — massive scalability with zero code change.

**Reference**: [Spring Boot Virtual Threads](https://docs.spring.io/spring-boot/reference/features/spring-application.html#features.spring-application.virtual-threads)

---

## Migration Checklist

| # | Step | Details |
|---|------|---------|
| 1 | **Update build tool** | Set `source` and `target` to 21 in `pom.xml` or `build.gradle` |
| 2 | **Update CI/CD image** | Use `eclipse-temurin:21-jdk` or equivalent |
| 3 | **Run full build** | Fix removed/deprecated API usage |
| 4 | **Check SecurityManager** | Deprecated for removal — plan alternatives |
| 5 | **Adopt virtual threads** | Replace fixed thread pools in I/O-heavy services |
| 6 | **Refactor instanceof chains** | Use switch pattern matching |
| 7 | **Use SequencedCollection** | Where you access first/last elements |
| 8 | **Remove --enable-preview** | For features now finalized |
| 9 | **Update frameworks** | Spring Boot 3.2+, Hibernate 6.4+ |
| 10 | **Update libraries** | Lombok, Byte Buddy, ASM compatibility |
| 11 | **Run tests & benchmarks** | Verify GC behavior and performance |

---

## Common Pitfalls

### 1. Virtual Threads + Synchronized Blocks

Virtual threads can get **pinned** to carrier threads inside `synchronized` blocks. Prefer `ReentrantLock`:

```java
// Avoid with virtual threads
synchronized (lock) {
    blockingIOCall();
}

// Prefer
private final ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    blockingIOCall();
} finally {
    lock.unlock();
}
```

**Deep dive**: [JEP 491 — Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491) (targeted for future releases)

### 2. Preview API Breaking Changes

Features evolve between preview rounds. Don't rely on preview behavior from 19/20 — always reference the final JEP.

### 3. Third-Party Library Compatibility

Libraries that manipulate bytecode need updates:
- **Lombok**: 1.18.30+
- **Byte Buddy**: 1.14.9+
- **ASM**: 9.6+

Update these **before** upgrading the JDK.

---

## Build Tool Configuration

### Maven

```xml
<properties>
    <java.version>21</java.version>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
</properties>
```

### Gradle

```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

### Dockerfile

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
# build stage

FROM eclipse-temurin:21-jre-alpine
# runtime stage
```

---

## Further Resources

| Resource | Link |
|----------|------|
| OpenJDK 21 Project Page | [openjdk.org/projects/jdk/21](https://openjdk.org/projects/jdk/21/) |
| Oracle Java SE 21 Docs | [docs.oracle.com/en/java/javase/21](https://docs.oracle.com/en/java/javase/21/) |
| Oracle Migration Guide | [Java SE 21 Migration Guide](https://docs.oracle.com/en/java/javase/21/migrate/) |
| Java Almanac (Diff Tool) | [javaalmanac.io/jdk/21](https://javaalmanac.io/jdk/21/) |
| Baeldung — Java 21 Features | [baeldung.com/java-lts-21-new-features](https://www.baeldung.com/java-lts-21-new-features) |
| Spring Boot 3.2 Release Notes | [GitHub Wiki](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.2-Release-Notes) |
| Virtual Threads — Oracle Blog | [blogs.oracle.com](https://blogs.oracle.com/javamagazine/post/java-virtual-threads) |
| JDK Download (Temurin) | [adoptium.net](https://adoptium.net/) |
