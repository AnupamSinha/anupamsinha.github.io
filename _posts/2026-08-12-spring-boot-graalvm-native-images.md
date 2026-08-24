---
title: "Spring Boot with GraalVM Native Images — Instant Startup, Minimal Memory"
date: 2026-08-12
categories: [Spring Boot, Performance]
tags: [spring-boot, graalvm, native-image, aot, performance, cloud-native, containers, serverless, java-21]
description: "A practical guide to compiling Spring Boot applications into GraalVM native images. Covers setup, AOT compilation, handling reflection, testing native builds, and real-world trade-offs for cloud-native and serverless deployments."
---

## Why Native Images?

A typical Spring Boot app on the JVM:

- **Starts in 2-5 seconds** (class loading, bean initialization, JIT warm-up)
- **Uses 200-500MB RAM** at baseline
- **Takes time to reach peak performance** (JIT compilation needs warm-up)

For long-running servers, this is fine. But for:

- **Serverless functions** (Lambda, Cloud Functions) — you pay for cold start
- **Kubernetes scaling** — new pods need to serve traffic immediately
- **CLI tools** — users expect instant response

...startup time and memory footprint matter a lot.

### GraalVM Native Image Changes the Game

| Metric | JVM | Native Image |
|--------|-----|-------------|
| Startup time | 2-5 seconds | 30-80 milliseconds |
| Memory (RSS) | 200-500MB | 40-80MB |
| Peak throughput | Higher (JIT optimized) | Lower (AOT compiled) |
| Build time | Seconds | Minutes |

You trade peak throughput for instant startup and minimal memory.

---

## How Native Image Compilation Works

Instead of shipping bytecode that the JVM interprets and JIT-compiles at runtime, GraalVM's `native-image` tool performs **Ahead-of-Time (AOT) compilation**:

1. **Starts from `main()`** and traces all reachable code paths
2. **Resolves everything at build time** — no dynamic class loading
3. **Eliminates dead code** — only includes what's actually used
4. **Produces a standalone binary** — no JVM needed at runtime

```
Source Code → Bytecode → native-image → Platform Binary
                              ↓
                    Static analysis (closed-world assumption)
                    Initializes classes at build time
                    Strips unreachable code
```

---

## Setting Up Your Project

### Prerequisites

- **Java 21+** (GraalVM JDK or regular JDK with native-image installed)
- **Spring Boot 3.0+** (built-in AOT support)
- **GraalVM native-image tool** (`gu install native-image` or via SDKMAN)

### pom.xml Configuration

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

### Build the Native Image

```bash
# Compile to native binary
./mvnw -Pnative native:compile

# Or build a native container image (no local GraalVM needed)
./mvnw -Pnative spring-boot:build-image
```

The `-Pnative` profile is provided by `spring-boot-starter-parent` — it activates AOT processing and native compilation.

---

## Spring Boot's AOT Engine

Spring Boot 3.x includes an AOT processing step that prepares your app for native compilation:

### What AOT Processing Does

```java
// At build time, Spring's AOT engine:
// 1. Evaluates all @Conditional annotations
// 2. Resolves bean definitions
// 3. Generates optimized code for bean instantiation
// 4. Produces reflection/resource/proxy configuration for GraalVM
```

### Generated Output (target/spring-aot)

```
target/spring-aot/main/
├── sources/          # Generated Java code for bean registration
├── resources/
│   └── META-INF/
│       └── native-image/
│           ├── reflect-config.json    # Reflection metadata
│           ├── resource-config.json   # Resources to include
│           ├── proxy-config.json      # Dynamic proxies
│           └── serialization-config.json
```

This is what makes Spring Boot work with native images out of the box — the framework generates the hints GraalVM needs.

---

## The Closed-World Constraint

Native images operate under the **closed-world assumption**: everything must be known at build time. This conflicts with Java's dynamic features:

### What Requires Special Handling

| Feature | Problem | Solution |
|---------|---------|----------|
| Reflection | Classes accessed via reflection aren't found by static analysis | Register in `reflect-config.json` or use `@RegisterReflectionForBinding` |
| Dynamic proxies | JDK proxies created at runtime | Register in `proxy-config.json` |
| Resource loading | `getResource()` files aren't bundled | Register in `resource-config.json` |
| Serialization | Classes serialized/deserialized dynamically | Register in `serialization-config.json` |

### Registering Reflection Hints

```java
@Configuration
@RegisterReflectionForBinding({
    OrderDto.class,
    CustomerDto.class,
    PaymentResponse.class
})
public class NativeHintsConfig {
}
```

### RuntimeHints for Complex Cases

```java
@Component
public class MyRuntimeHints implements RuntimeHintsRegistrar {

    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Register reflection
        hints.reflection().registerType(ExternalLibraryClass.class,
                MemberCategory.PUBLIC_FIELDS,
                MemberCategory.INVOKE_PUBLIC_METHODS);

        // Register resources
        hints.resources().registerPattern("templates/*.html");
        hints.resources().registerPattern("validation-messages.properties");

        // Register proxies
        hints.proxies().registerJdkProxy(MyServiceInterface.class);
    }
}
```

Register it:

```java
@SpringBootApplication
@ImportRuntimeHints(MyRuntimeHints.class)
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

---

## Common Pitfalls and Fixes

### 1. Jackson Serialization Fails at Runtime

```java
// Problem: Jackson uses reflection to serialize/deserialize
// Fix: Use records (automatically registered) or add hints

// Records work out of the box in most cases
public record ProductDto(Long id, String name, BigDecimal price) {}

// For regular classes, register them
@RegisterReflectionForBinding(LegacyDto.class)
```

### 2. JPA Entities Need Registration

```java
@Entity
@Table(name = "orders")
public class Order {
    // Entities are usually handled by Spring's AOT engine
    // But if you use native queries with projections:
}

// Interface projections may need hints
public interface OrderSummaryProjection {
    String getCustomerName();
    BigDecimal getTotal();
}
```

### 3. Conditional Beans Resolved at Build Time

```java
// This works fine — evaluated at AOT build time
@ConditionalOnProperty(name = "features.notifications", havingValue = "true")
@Service
public class NotificationService { }

// DANGER: This won't work if the condition changes between build and runtime
// Build with the same profile you'll run with
```

### 4. Libraries Without Native Support

Some libraries rely heavily on reflection and don't ship GraalVM metadata:

```xml
<!-- Check if your library supports native images -->
<!-- Many now ship metadata in: META-INF/native-image/ -->

<!-- For libraries that don't, use the reachability metadata repository -->
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <configuration>
        <metadataRepository>
            <enabled>true</enabled>
        </metadataRepository>
    </configuration>
</plugin>
```

---

## Testing Native Images

### AOT-Processed Tests

```java
@SpringBootTest
class OrderServiceNativeTest {
    // Standard tests run in AOT mode during native compilation
    // If they pass in AOT mode, they'll work in the native image
}
```

### Run Tests in Native Mode

```bash
# Compile and run tests as a native image
./mvnw -Pnative native:test
```

This compiles your tests into a native binary and runs them — catching AOT issues before deployment.

### GraalVM Tracing Agent (for discovering reflection usage)

```bash
# Run your app with the tracing agent to discover dynamic features
java -agentlib:native-image-agent=config-output-dir=src/main/resources/META-INF/native-image \
     -jar target/myapp.jar

# Exercise all code paths (run tests, hit endpoints)
# Agent records all reflection/resource/proxy usage
```

---

## Dockerfile for Native Images

### Multi-Stage Build

```dockerfile
# Stage 1: Build the native image
FROM ghcr.io/graalvm/native-image-community:21 AS builder

WORKDIR /app
COPY pom.xml .
COPY src ./src
COPY .mvn ./.mvn
COPY mvnw .

RUN ./mvnw -Pnative native:compile -DskipTests

# Stage 2: Minimal runtime image
FROM debian:bookworm-slim

WORKDIR /app
COPY --from=builder /app/target/myapp .

# No JVM needed!
EXPOSE 8080
ENTRYPOINT ["./myapp"]
```

### Resulting Image Size

```
Traditional (JVM + app):     ~350MB
Native (distroless):         ~80MB
Native (static + scratch):   ~50MB
```

---

## Spring Boot Build-Image (Easier Path)

If you don't want to manage Dockerfiles, Spring Boot's buildpack support handles everything:

```bash
# Builds a native container image using Cloud Native Buildpacks
./mvnw -Pnative spring-boot:build-image \
    -Dspring-boot.build-image.imageName=myapp:native
```

```bash
# Run it
docker run -p 8080:8080 myapp:native

# Startup log:
# Started MyApp in 0.056 seconds (process running for 0.062)
```

---

## Real-World Performance Comparison

A REST API with Spring Data JPA, 5 endpoints, PostgreSQL:

| Metric | JVM (Java 21) | Native Image |
|--------|--------------|--------------|
| Startup | 2.8s | 0.055s |
| Memory (idle) | 280MB | 52MB |
| Memory (under load) | 450MB | 95MB |
| Throughput (peak) | 12,400 req/s | 9,800 req/s |
| Build time | 8s | 3m 45s |
| Binary size | 21MB JAR + JVM | 78MB standalone |

---

## When to Use Native Images

### Use native images when:

- **Serverless / FaaS** — cold starts matter, you pay per invocation
- **Edge deployments** — memory constrained environments
- **Kubernetes autoscaling** — pods must serve traffic immediately
- **CLI tools** — users expect instant response
- **High-density hosting** — pack more instances per node

### Stick with JVM when:

- **Long-running services** — JIT warm-up gives better peak throughput
- **Heavy reflection** — too many hints to maintain
- **Rapid development** — native builds take minutes, JVM takes seconds
- **Dynamic plugins** — loading code at runtime isn't supported

---

## Migration Checklist

- [ ] Upgrade to Spring Boot 3.2+ and Java 21+
- [ ] Add `native-maven-plugin` to your build
- [ ] Run `./mvnw -Pnative native:test` — fix any AOT test failures
- [ ] Run the tracing agent against your integration test suite
- [ ] Replace `synchronized` with `ReentrantLock` where possible
- [ ] Replace problematic libraries (check the [GraalVM Reachability Metadata Repository](https://github.com/oracle/graalvm-reachability-metadata))
- [ ] Test with production-like data — edge cases surface at runtime
- [ ] Compare throughput under load — verify the trade-off is acceptable
- [ ] Set up CI/CD with native builds (expect 5-10 min build times)

---

> Native images are a deployment optimization, not a development workflow. Develop and test on the JVM for fast feedback, then compile to native for production deployment.
{: .prompt-tip }
