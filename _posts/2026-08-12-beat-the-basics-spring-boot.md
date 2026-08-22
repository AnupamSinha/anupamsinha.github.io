---
title: "Beat the Basics — Spring Boot from Zero to Confident"
date: 2026-08-12
categories: [Java, Spring]
tags: [spring-boot, java, rest-api, dependency-injection, jpa, spring-security, microservices, backend, tutorial]
description: "A multi-chapter guide to mastering Spring Boot fundamentals. Covers project setup, dependency injection, REST APIs, data access with JPA, exception handling, configuration, security basics, and testing — all with working code examples."
---

## Why This Guide?

Spring Boot is the backbone of most enterprise Java applications. But jumping in can feel overwhelming — starters, auto-configuration, annotations everywhere. This guide breaks it down chapter by chapter with real, runnable examples.

By the end, you'll understand how Spring Boot works under the hood, not just how to copy-paste from Stack Overflow.

---

## Chapter 1: What Spring Boot Actually Does

Spring Boot is an opinionated layer on top of the Spring Framework. It eliminates boilerplate by:

- **Auto-configuring** beans based on what's on your classpath
- **Embedding** a server (Tomcat/Jetty/Undertow) so you don't need a WAR deployment
- **Providing starters** — curated dependency bundles that just work together

### The Minimal Application

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

`@SpringBootApplication` is a shortcut for three annotations:

| Annotation | Purpose |
|-----------|---------|
| `@Configuration` | Marks the class as a source of bean definitions |
| `@EnableAutoConfiguration` | Tells Spring Boot to guess configuration based on classpath |
| `@ComponentScan` | Scans the current package and sub-packages for components |

### How Auto-Configuration Works

When you add `spring-boot-starter-web` to your `pom.xml`, Spring Boot:

1. Sees `DispatcherServlet` on the classpath
2. Automatically configures an embedded Tomcat server on port 8080
3. Registers the servlet, error handling, and static resource mappings

No XML. No manual bean wiring. It just works.

---

## Chapter 2: Dependency Injection — The Core Pattern

Everything in Spring revolves around **Inversion of Control (IoC)**. You don't create objects — the framework creates and injects them for you.

### Constructor Injection (Preferred)

```java
@Service
public class OrderService {

    private final PaymentGateway paymentGateway;
    private final InventoryService inventoryService;

    // Spring injects these automatically
    public OrderService(PaymentGateway paymentGateway, InventoryService inventoryService) {
        this.paymentGateway = paymentGateway;
        this.inventoryService = inventoryService;
    }

    public OrderResult placeOrder(Order order) {
        inventoryService.reserve(order.getItems());
        return paymentGateway.charge(order.getTotal());
    }
}
```

### Why Constructor Injection Over @Autowired on Fields?

- **Immutability** — fields can be `final`
- **Testability** — easy to pass mocks in unit tests without reflection
- **Fail-fast** — missing dependencies cause startup failure, not runtime NPEs

### Bean Scopes

```java
@Component
@Scope("prototype") // New instance every time it's injected
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
}
```

| Scope | Behavior |
|-------|----------|
| `singleton` (default) | One instance per Spring container |
| `prototype` | New instance on every injection |
| `request` | One instance per HTTP request |
| `session` | One instance per HTTP session |

---

## Chapter 3: Building REST APIs

### A Complete Controller

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    public List<ProductDto> getAll() {
        return productService.findAll();
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductDto> getById(@PathVariable Long id) {
        return productService.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductDto create(@Valid @RequestBody CreateProductRequest request) {
        return productService.create(request);
    }

    @PutMapping("/{id}")
    public ProductDto update(@PathVariable Long id, @Valid @RequestBody UpdateProductRequest request) {
        return productService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        productService.delete(id);
    }
}
```

### Request Validation with Bean Validation

```java
public record CreateProductRequest(
    @NotBlank(message = "Name is required")
    String name,

    @Positive(message = "Price must be positive")
    BigDecimal price,

    @Size(max = 500, message = "Description too long")
    String description
) {}
```

### Key Annotations Explained

| Annotation | What It Does |
|-----------|--------------|
| `@RestController` | `@Controller` + `@ResponseBody` — returns JSON by default |
| `@RequestMapping` | Base path for all endpoints in this controller |
| `@PathVariable` | Extracts value from the URL path |
| `@RequestBody` | Deserializes JSON request body into an object |
| `@Valid` | Triggers Bean Validation on the annotated parameter |

---

## Chapter 4: Data Access with Spring Data JPA

### Entity Definition

```java
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    private String description;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;

    // Constructors, getters, setters
    protected Product() {} // JPA requires no-arg constructor

    public Product(String name, BigDecimal price, String description) {
        this.name = name;
        this.price = price;
        this.description = description;
    }
}
```

### Repository — Zero Boilerplate

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Spring Data generates the query from the method name
    List<Product> findByNameContainingIgnoreCase(String keyword);

    // Custom query when method names get too complex
    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :min AND :max ORDER BY p.price")
    List<Product> findByPriceRange(@Param("min") BigDecimal min, @Param("max") BigDecimal max);

    // Native SQL when you need it
    @Query(value = "SELECT * FROM products WHERE created_at > :since", nativeQuery = true)
    List<Product> findRecentProducts(@Param("since") LocalDateTime since);
}
```

### How It Works Under the Hood

1. Spring Data sees your interface extending `JpaRepository`
2. At startup, it creates a **proxy implementation** using `SimpleJpaRepository`
3. Method names like `findByNameContainingIgnoreCase` are parsed into JPQL queries
4. You get full CRUD + pagination + sorting with zero implementation code

### application.yml Configuration

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate  # Never use 'update' or 'create' in production
    show-sql: false
    properties:
      hibernate:
        format_sql: true
```

---

## Chapter 5: Exception Handling Done Right

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            LocalDateTime.now()
        );
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult()
                .getFieldErrors()
                .stream()
                .collect(Collectors.toMap(
                    FieldError::getField,
                    FieldError::getDefaultMessage,
                    (first, second) -> first
                ));

        return new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Validation failed",
            LocalDateTime.now(),
            errors
        );
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneric(Exception ex) {
        // Log the full stack trace but don't expose it to the client
        log.error("Unexpected error", ex);
        return new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "An unexpected error occurred",
            LocalDateTime.now()
        );
    }
}
```

### Custom Exception

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, Long id) {
        super(String.format("%s not found with id: %d", resource, id));
    }
}
```

### Error Response DTO

```java
public record ErrorResponse(
    int status,
    String message,
    LocalDateTime timestamp,
    Map<String, String> fieldErrors
) {
    public ErrorResponse(int status, String message, LocalDateTime timestamp) {
        this(status, message, timestamp, null);
    }
}
```

---

## Chapter 6: Configuration and Profiles

### Externalized Configuration

Spring Boot reads configuration from multiple sources in order of precedence:

1. Command-line arguments
2. Environment variables
3. `application-{profile}.yml`
4. `application.yml`

### Custom Configuration Properties

```java
@Configuration
@ConfigurationProperties(prefix = "app.notifications")
public class NotificationConfig {

    private boolean enabled = true;
    private int retryAttempts = 3;
    private Duration timeout = Duration.ofSeconds(30);
    private Map<String, String> templates = new HashMap<>();

    // Getters and setters
}
```

```yaml
# application.yml
app:
  notifications:
    enabled: true
    retry-attempts: 5
    timeout: 45s
    templates:
      welcome: "Welcome to {appName}, {userName}!"
      reset: "Click here to reset your password: {link}"
```

### Profiles for Environment-Specific Config

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true

logging:
  level:
    com.myapp: DEBUG
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

logging:
  level:
    com.myapp: WARN
```

Activate with: `--spring.profiles.active=prod` or `SPRING_PROFILES_ACTIVE=prod`

---

## Chapter 7: Security Basics

### Minimal Security Configuration (Spring Security 6.x)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable()) // Disable for REST APIs using tokens
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### UserDetailsService Implementation

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User.builder()
                .username(user.getEmail())
                .password(user.getPassword())
                .roles(user.getRoles().toArray(new String[0]))
                .build();
    }
}
```

### What's Happening Behind the Scenes

1. Every request passes through a **filter chain**
2. `UsernamePasswordAuthenticationFilter` extracts credentials
3. `AuthenticationManager` delegates to your `UserDetailsService`
4. `PasswordEncoder` verifies the password hash
5. On success, a `SecurityContext` is populated for the request

---

## Chapter 8: Testing Spring Boot Applications

### Unit Test (No Spring Context)

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private InventoryService inventoryService;

    @InjectMocks
    private OrderService orderService;

    @Test
    void placeOrder_shouldChargeAndReserve() {
        Order order = new Order(List.of(new Item("Widget", 2)), BigDecimal.valueOf(49.99));
        when(paymentGateway.charge(any())).thenReturn(OrderResult.success());

        OrderResult result = orderService.placeOrder(order);

        assertThat(result.isSuccess()).isTrue();
        verify(inventoryService).reserve(order.getItems());
        verify(paymentGateway).charge(order.getTotal());
    }
}
```

### Integration Test (Full Context)

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class ProductControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ProductRepository productRepository;

    @BeforeEach
    void setup() {
        productRepository.deleteAll();
    }

    @Test
    void createProduct_shouldReturn201() throws Exception {
        String requestBody = """
            {
                "name": "Wireless Mouse",
                "price": 29.99,
                "description": "Ergonomic wireless mouse"
            }
            """;

        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.name").value("Wireless Mouse"))
                .andExpect(jsonPath("$.price").value(29.99));
    }

    @Test
    void getProduct_notFound_shouldReturn404() throws Exception {
        mockMvc.perform(get("/api/v1/products/999"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.message").value("Product not found with id: 999"));
    }
}
```

### Slice Tests (Fast, Focused)

```java
// Only loads JPA-related beans
@DataJpaTest
class ProductRepositoryTest {

    @Autowired
    private ProductRepository productRepository;

    @Test
    void findByPriceRange_shouldReturnMatchingProducts() {
        productRepository.save(new Product("Cheap", BigDecimal.valueOf(5.00), ""));
        productRepository.save(new Product("Mid", BigDecimal.valueOf(50.00), ""));
        productRepository.save(new Product("Expensive", BigDecimal.valueOf(500.00), ""));

        List<Product> results = productRepository.findByPriceRange(
                BigDecimal.valueOf(10), BigDecimal.valueOf(100));

        assertThat(results).hasSize(1);
        assertThat(results.get(0).getName()).isEqualTo("Mid");
    }
}
```

### Testing Cheat Sheet

| Annotation | What It Loads | Use Case |
|-----------|--------------|----------|
| `@SpringBootTest` | Full application context | End-to-end integration tests |
| `@WebMvcTest` | Only web layer (controllers, filters) | Controller logic tests |
| `@DataJpaTest` | Only JPA layer (repos, entities) | Repository query tests |
| `@MockBean` | Replaces a bean with a Mockito mock | Isolating layers in integration tests |

---

## Chapter 9: Actuator and Production Readiness

### Add the Starter

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Configure Endpoints

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: when-authorized
  health:
    db:
      enabled: true
    diskSpace:
      enabled: true
```

### Custom Health Indicator

```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentGateway paymentGateway;

    public PaymentGatewayHealthIndicator(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    @Override
    public Health health() {
        try {
            paymentGateway.ping();
            return Health.up()
                    .withDetail("provider", "Stripe")
                    .withDetail("responseTime", "45ms")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

### Key Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/actuator/health` | Liveness/readiness for Kubernetes probes |
| `/actuator/metrics` | Application metrics (JVM, HTTP, custom) |
| `/actuator/prometheus` | Prometheus-compatible metrics export |
| `/actuator/info` | App version, git commit, build info |

---

## Wrapping Up

Spring Boot removes the ceremony so you can focus on business logic. Here's what we covered:

| Chapter | Key Takeaway |
|---------|-------------|
| 1. What Spring Boot Does | Auto-configuration eliminates boilerplate |
| 2. Dependency Injection | Constructor injection for testable, immutable services |
| 3. REST APIs | Annotations map HTTP semantics cleanly |
| 4. Data Access | Spring Data JPA generates implementations from interfaces |
| 5. Exception Handling | `@RestControllerAdvice` for consistent error responses |
| 6. Configuration | Profiles separate environment concerns |
| 7. Security | Filter chains protect endpoints declaratively |
| 8. Testing | Slice tests keep feedback loops fast |
| 9. Actuator | Production observability out of the box |

The best way to learn this is hands-on. Clone a starter, build a small CRUD API, break things, fix things. The framework rewards experimentation.

---

> Next steps: explore Spring Boot with **virtual threads** (Project Loom), **GraalVM native images**, and **Spring Modulith** for modular monoliths.
{: .prompt-tip }
