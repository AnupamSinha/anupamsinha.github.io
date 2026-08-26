---
title: "Building Production-Ready REST APIs — The Complete Checklist"
date: 2026-08-24
categories: [Spring Boot, Fundamentals]
tags: [rest-api, spring-boot, production, java, best-practices]
description: "Everything between "it works on my machine" and "it handles 10,000 requests per second in production" — the full checklist with Spring Boot implementations."
mermaid: true
---
## The Gap Nobody Talks About

You've built a REST API. It handles CRUD operations. Your Postman tests pass. Your unit tests are green. You deploy to production and then...

- A client sends a 50MB JSON payload and your service OOMs
- Someone hits your endpoint 10,000 times per second and overwhelms your database
- A deployment causes 30 seconds of 502 errors
- Your error responses are inconsistent and your mobile team is furious
- A network partition causes duplicate orders

There's an enormous gap between "the API works" and "the API is production-ready." After building and maintaining APIs that serve millions of requests daily across multiple organizations in Singapore, here's the complete checklist I've developed.

---

## 1. API Versioning

Decide on a versioning strategy before your first client integrates. Changing it later is painful.

### Strategy: URI Path Versioning (My Recommendation)

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {
    
    @GetMapping("/{id}")
    public OrderResponseV1 getOrder(@PathVariable Long id) {
        return orderService.getOrderV1(id);
    }
}

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 {
    
    @GetMapping("/{id}")
    public OrderResponseV2 getOrder(@PathVariable Long id) {
        // V2 includes additional fields, different structure
        return orderService.getOrderV2(id);
    }
}
```

**Why path versioning over header versioning:**
- Visible and explicit — no hidden behavior
- Easy to test (just change the URL)
- Load balancers and CDNs can route by path
- API documentation is clearer

**Rule of thumb:** Support the current version and one version back. Deprecate with 6-month notice. Never remove without migration support.

---

## 2. Pagination

Never return unbounded lists. Ever. A table with 10 rows today will have 10 million tomorrow.

### Cursor-Based Pagination (Preferred for Large Datasets)

```java
@GetMapping("/orders")
public CursorPage<OrderResponse> getOrders(
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") @Max(100) int limit) {
    
    return orderService.getOrders(cursor, limit);
}

// The response structure
public record CursorPage<T>(
    List<T> data,
    String nextCursor,
    boolean hasMore
) {}
```

```java
@Service
public class OrderService {
    
    public CursorPage<OrderResponse> getOrders(String cursor, int limit) {
        Long afterId = cursor != null ? decodeCursor(cursor) : 0L;
        
        List<Order> orders = orderRepository.findByIdGreaterThanOrderByIdAsc(
            afterId, PageRequest.of(0, limit + 1)); // Fetch one extra to check hasMore
        
        boolean hasMore = orders.size() > limit;
        List<Order> pageData = hasMore ? orders.subList(0, limit) : orders;
        
        String nextCursor = hasMore ? 
            encodeCursor(pageData.get(pageData.size() - 1).getId()) : null;
        
        return new CursorPage<>(
            pageData.stream().map(this::toResponse).toList(),
            nextCursor,
            hasMore
        );
    }
    
    private String encodeCursor(Long id) {
        return Base64.getUrlEncoder().encodeToString(id.toString().getBytes());
    }
    
    private Long decodeCursor(String cursor) {
        return Long.parseLong(new String(Base64.getUrlDecoder().decode(cursor)));
    }
}
```

**Why cursor over offset:**
- Offset pagination breaks when data is inserted/deleted between pages
- `OFFSET 1000000` is expensive — the DB still scans those rows
- Cursors are stable regardless of concurrent modifications

---

## 3. Standardized Error Responses

Nothing makes frontend developers angrier than inconsistent error formats. Standardize from day one using RFC 7807 Problem Details:

```java
public record ProblemDetail(
    String type,
    String title,
    int status,
    String detail,
    String instance,
    Map<String, Object> extensions
) {}
```

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ProblemDetail> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {
        ProblemDetail problem = new ProblemDetail(
            "https://api.example.com/problems/resource-not-found",
            "Resource Not Found",
            404,
            ex.getMessage(),
            request.getRequestURI(),
            Map.of("resourceType", ex.getResourceType(), "resourceId", ex.getResourceId())
        );
        return ResponseEntity.status(404).body(problem);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ProblemDetail> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {
        List<Map<String, String>> fieldErrors = ex.getBindingResult()
            .getFieldErrors().stream()
            .map(fe -> Map.of(
                "field", fe.getField(),
                "message", fe.getDefaultMessage(),
                "rejectedValue", String.valueOf(fe.getRejectedValue())
            ))
            .toList();
        
        ProblemDetail problem = new ProblemDetail(
            "https://api.example.com/problems/validation-error",
            "Validation Failed",
            422,
            "One or more fields failed validation",
            request.getRequestURI(),
            Map.of("errors", fieldErrors)
        );
        return ResponseEntity.status(422).body(problem);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleUnexpected(
            Exception ex, HttpServletRequest request) {
        log.error("Unexpected error at {}", request.getRequestURI(), ex);
        ProblemDetail problem = new ProblemDetail(
            "https://api.example.com/problems/internal-error",
            "Internal Server Error",
            500,
            "An unexpected error occurred. Please try again later.",
            request.getRequestURI(),
            Map.of("traceId", MDC.get("traceId"))
        );
        return ResponseEntity.status(500).body(problem);
    }
}
```

**Important:** Never leak stack traces or internal details in production error responses. Include a trace ID so you can correlate with logs.

---

## 4. Rate Limiting

Protect your API from abuse and ensure fair usage. Implement at both the application level and (ideally) the gateway level.

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {
    
    private final StringRedisTemplate redis;
    
    private static final int MAX_REQUESTS = 100;
    private static final Duration WINDOW = Duration.ofMinutes(1);
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
            HttpServletResponse response, FilterChain chain) 
            throws ServletException, IOException {
        
        String clientId = extractClientId(request);
        String key = "ratelimit:" + clientId + ":" + currentWindow();
        
        Long count = redis.opsForValue().increment(key);
        if (count == 1) {
            redis.expire(key, WINDOW.plusSeconds(10));
        }
        
        // Set rate limit headers
        response.setHeader("X-RateLimit-Limit", String.valueOf(MAX_REQUESTS));
        response.setHeader("X-RateLimit-Remaining", 
            String.valueOf(Math.max(0, MAX_REQUESTS - count)));
        response.setHeader("X-RateLimit-Reset", 
            String.valueOf(nextWindowReset()));
        
        if (count > MAX_REQUESTS) {
            response.setStatus(429);
            response.setContentType("application/problem+json");
            response.getWriter().write("""
                {"type":"https://api.example.com/problems/rate-limited",
                 "title":"Too Many Requests",
                 "status":429,
                 "detail":"Rate limit exceeded. Try again in %d seconds."}
                """.formatted(secondsUntilReset()));
            return;
        }
        
        chain.doFilter(request, response);
    }
    
    private String currentWindow() {
        return String.valueOf(Instant.now().getEpochSecond() / WINDOW.getSeconds());
    }
}
```

**Always include rate limit headers** — even when the client is under the limit. This lets well-behaved clients implement backoff before hitting 429s.

---

## 5. Request Validation

Validate everything at the boundary. Never trust client input.

```java
public record CreateOrderRequest(
    @NotNull(message = "Customer ID is required")
    Long customerId,
    
    @NotEmpty(message = "At least one item is required")
    @Size(max = 50, message = "Maximum 50 items per order")
    List<@Valid OrderItemRequest> items,
    
    @NotBlank(message = "Shipping address is required")
    @Size(max = 500, message = "Address too long")
    String shippingAddress,
    
    @Pattern(regexp = "^[A-Z]{3}$", message = "Invalid currency code")
    String currency
) {}

public record OrderItemRequest(
    @NotNull Long productId,
    @Min(value = 1, message = "Quantity must be at least 1")
    @Max(value = 999, message = "Quantity too large")
    Integer quantity
) {}
```

### Protect Against Payload Attacks

```yaml
# application.yml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
  codec:
    max-in-memory-size: 1MB

server:
  max-http-request-header-size: 16KB
  tomcat:
    max-swallow-size: 2MB
    max-http-form-post-size: 2MB
```

---

## 6. Health Checks

Production systems need more than a 200 OK — they need readiness and liveness probes that mean something.

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    private final DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            conn.createStatement().execute("SELECT 1");
            return Health.up()
                .withDetail("database", "reachable")
                .withDetail("pool.active", getActiveConnections())
                .withDetail("pool.idle", getIdleConnections())
                .build();
        } catch (SQLException e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

@Component
public class ExternalServiceHealthIndicator implements HealthIndicator {
    
    private final WebClient webClient;
    
    @Override
    public Health health() {
        try {
            webClient.get()
                .uri("/health")
                .retrieve()
                .toBodilessEntity()
                .block(Duration.ofSeconds(3));
            return Health.up().build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("service", "payment-gateway")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

**Separate liveness from readiness:**
- **Liveness** (`/actuator/health/liveness`) — Is the process alive? (restart if no)
- **Readiness** (`/actuator/health/readiness`) — Can it serve traffic? (remove from load balancer if no)

---

## 7. API Documentation (OpenAPI)

If your API isn't documented, it doesn't exist. Use SpringDoc to auto-generate OpenAPI specs:

```java
@Operation(
    summary = "Create a new order",
    description = "Creates a new order. Requires idempotency key for safe retries.",
    responses = {
        @ApiResponse(responseCode = "201", description = "Order created"),
        @ApiResponse(responseCode = "409", description = "Duplicate idempotency key"),
        @ApiResponse(responseCode = "422", description = "Validation failed")
    }
)
@PostMapping
public ResponseEntity<OrderResponse> createOrder(
        @Parameter(description = "Unique key for idempotent retries", required = true)
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody @Valid CreateOrderRequest request) {
    // ...
}
```

---

## 8. Security Headers and CORS

```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .headers(headers -> headers
                .contentTypeOptions(Customizer.withDefaults())
                .frameOptions(frame -> frame.deny())
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubDomains(true))
                .xssProtection(xss -> xss.headerValue(
                    XXssProtectionHeaderWriter.HeaderValue.ENABLED_MODE_BLOCK))
            )
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .build();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of(
            "https://app.example.com",
            "https://admin.example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        config.setAllowedHeaders(List.of(
            "Authorization", "Content-Type", "Idempotency-Key"));
        config.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

**Never use `allowedOrigins("*")` in production.** Whitelist explicitly.

---

## 9. Compression and Performance

```yaml
server:
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/plain
    min-response-size: 1024

spring:
  jackson:
    default-property-inclusion: non_null  # Don't serialize nulls
    serialization:
      write-dates-as-timestamps: false
```

### Response Compression with ETags

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Bean
    public FilterRegistrationBean<ShallowEtagHeaderFilter> etagFilter() {
        FilterRegistrationBean<ShallowEtagHeaderFilter> registration = 
            new FilterRegistrationBean<>();
        registration.setFilter(new ShallowEtagHeaderFilter());
        registration.addUrlPatterns("/api/*");
        return registration;
    }
}
```

ETags let clients cache responses and only fetch when data changes — reducing bandwidth and server load significantly for read-heavy APIs.

---

## 10. Timeouts and Circuit Breakers

Never call an external service without a timeout. Never let a failing service take down your system.

```java
@Configuration
public class ResilienceConfig {
    
    @Bean
    public CircuitBreakerConfig circuitBreakerConfig() {
        return CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowSize(10)
            .minimumNumberOfCalls(5)
            .build();
    }
}

@Service
public class PaymentGatewayClient {
    
    private final WebClient webClient;
    private final CircuitBreaker circuitBreaker;
    
    public PaymentResponse charge(PaymentRequest request) {
        return circuitBreaker.executeSupplier(() -> 
            webClient.post()
                .uri("/charges")
                .bodyValue(request)
                .retrieve()
                .bodyToMono(PaymentResponse.class)
                .timeout(Duration.ofSeconds(5))
                .block()
        );
    }
}
```

### Configure Timeouts at Every Layer

**Connection timeout** — How long to wait for TCP connection (2-5 seconds)

**Read timeout** — How long to wait for response data (5-30 seconds depending on operation)

**Connection pool timeout** — How long to wait for an available connection from pool (1-3 seconds)

```java
@Bean
public WebClient paymentWebClient() {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
        .responseTimeout(Duration.ofSeconds(5))
        .doOnConnected(conn -> conn
            .addHandlerLast(new ReadTimeoutHandler(5))
            .addHandlerLast(new WriteTimeoutHandler(3)));
    
    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .baseUrl("https://payment-gateway.example.com")
        .build();
}
```

---

## 11. Request/Response Logging and Tracing

You can't debug production issues without proper request tracing.

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestTracingFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        
        String traceId = request.getHeader("X-Trace-Id");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString().replace("-", "").substring(0, 16);
        }
        
        MDC.put("traceId", traceId);
        response.setHeader("X-Trace-Id", traceId);
        
        long start = System.currentTimeMillis();
        try {
            chain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - start;
            log.info("HTTP {} {} - {} ({}ms)", 
                request.getMethod(), request.getRequestURI(),
                response.getStatus(), duration);
            MDC.clear();
        }
    }
}
```

---

## 12. Graceful Shutdown

Ensure in-flight requests complete during deployments:

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

This tells Spring Boot to:
1. Stop accepting new requests
2. Wait up to 30 seconds for active requests to complete
3. Then shut down

Combined with Kubernetes rolling deployments, this eliminates deployment-related errors.

---

## The Complete Checklist

Here's your go-to checklist before declaring an API production-ready:

**Contract and Design:**
- API versioning strategy decided and implemented
- Consistent URL naming conventions (plural nouns, kebab-case)
- Pagination on all list endpoints
- Standardized error response format (RFC 7807)
- OpenAPI documentation generated and published

**Security:**
- Authentication (OAuth 2.0 / JWT)
- Authorization (endpoint-level and resource-level)
- CORS configured with explicit whitelist
- Security headers set (HSTS, X-Content-Type-Options, X-Frame-Options)
- Input validation on all endpoints
- Request size limits configured
- Rate limiting enabled

**Reliability:**
- Idempotency keys for mutating operations
- Timeouts on all external calls
- Circuit breakers for downstream services
- Health checks (liveness + readiness)
- Graceful shutdown configured
- Retry logic with exponential backoff for clients

**Observability:**
- Request/response logging with trace IDs
- Metrics (request count, latency percentiles, error rates)
- Distributed tracing (OpenTelemetry)
- Alerting on error rate spikes and latency degradation

**Performance:**
- Response compression enabled
- ETags for cacheable responses
- Database query optimization (N+1, indexes, connection pooling)
- Async processing for long-running operations

This isn't theoretical. Every item on this list has cost me (or my team) hours of production debugging when it was missing. The initial investment pays for itself after the first production incident you prevent.
