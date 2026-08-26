---
title: "Migrating from Monolith to Microservices — A Battle-Tested Playbook"
date: 2026-08-24
categories: [Spring Boot, Architecture]
tags: [microservices, monolith, migration, spring-boot, architecture]
description: "The real migration strategy from someone who's done it three times — covering the strangler fig pattern, bounded context identification, database decomposition, and rollback planning."
mermaid: true
---
## The Migration That Changed My Perspective

In 2019, I led the decomposition of a 600,000-line Java monolith for a payments company in Singapore. The monolith was 8 years old, had 47 developers working on it, and deployed once every two weeks (when things went well). Deployments took 4 hours and required 6 people on a call.

The business wanted faster feature delivery. The CTO wanted microservices. I wanted to not lose my job.

Here's what actually worked — not the textbook version, but the battle-scarred reality.

## Step 0: Should You Even Migrate?

Before I share the how, let me be direct: most teams should NOT migrate to microservices. You should only consider it when:

- **Deployment bottleneck** — Teams block each other on releases
- **Scaling mismatch** — One module needs 50x the resources of another
- **Team autonomy** — You have 4+ teams that need independent release cycles
- **Technology diversity** — Part of your system genuinely needs a different tech stack

If your problem is "the code is messy," microservices won't fix it. They'll just distribute the mess across a network. Clean your monolith first.

In our case, we had all four signals. The payments module needed 20x the compute of the reporting module, and 8 teams were stepping on each other's code daily.

## Step 1: Identify Bounded Contexts

Don't decompose by technical layer (service, repository, controller). Decompose by business domain. This is where most migrations fail — wrong boundaries create chatty services that are worse than the monolith.

### How We Found Our Boundaries

We used a technique called Event Storming combined with dependency analysis:

```java
// Tool to analyze module coupling in the monolith
@Component
public class CouplingAnalyzer {

    public CouplingReport analyze(String basePackage) {
        var report = new CouplingReport();

        // Scan all classes and track cross-package dependencies
        var classes = new ClassPathScanningCandidateComponentProvider(true)
            .findCandidateComponents(basePackage);

        for (var beanDef : classes) {
            Class<?> clazz = Class.forName(beanDef.getBeanClassName());
            String sourceModule = extractModule(clazz.getPackageName());

            for (Field field : clazz.getDeclaredFields()) {
                String targetModule = extractModule(field.getType().getPackageName());
                if (!sourceModule.equals(targetModule)) {
                    report.addDependency(sourceModule, targetModule, clazz.getSimpleName());
                }
            }
        }
        return report;
    }

    private String extractModule(String packageName) {
        // com.company.app.payments.service -> "payments"
        String[] parts = packageName.split("\\.");
        return parts.length > 3 ? parts[3] : "core";
    }
}
```

### Our Resulting Bounded Contexts

After two weeks of analysis and three event storming sessions:

**Payment Processing** — Transaction creation, payment gateway integration, settlement

**Account Management** — User profiles, KYC, account lifecycle

**Notification** — Email, SMS, push notifications, templates

**Reporting** — Analytics, regulatory reports, statement generation

**Merchant Portal** — Merchant onboarding, configuration, dashboard

**Risk & Compliance** — Fraud detection, AML screening, rule engine

The key insight: we initially had "User" as a shared concept. But the Payment module only needed user ID and payment methods. The Account module needed full profile data. The Risk module needed transaction history. Each bounded context has its own view of what a "user" means.

## Step 2: Strangler Fig Pattern — The Only Safe Approach

Don't do a big-bang rewrite. I've seen three rewrites attempted. All three failed. Instead, use the strangler fig pattern: gradually route traffic from the monolith to new services while the monolith continues running.

### The Architecture

```
[Load Balancer]
      |
[API Gateway / Router]
      |
      ├── /api/payments/*  → New Payment Service (microservice)
      ├── /api/accounts/*  → Monolith (still handling this)
      ├── /api/reports/*   → Monolith (still handling this)
      └── /api/merchants/* → Monolith (still handling this)
```

### The Router Implementation

```java
@Configuration
public class StranglerFigRouter {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // Migrated: route to new microservice
            .route("payment-service", r -> r
                .path("/api/payments/**")
                .and()
                .header("X-Migration-Phase", "new")
                .uri("lb://payment-service"))

            // Shadow mode: send to both, return monolith response
            .route("payment-shadow", r -> r
                .path("/api/payments/**")
                .and()
                .header("X-Migration-Phase", "shadow")
                .filters(f -> f.filter(shadowTrafficFilter()))
                .uri("lb://monolith"))

            // Default: everything else stays with monolith
            .route("monolith-default", r -> r
                .path("/api/**")
                .uri("lb://monolith"))
            .build();
    }
}
```

### Shadow Traffic — The Validation Phase

Before cutting over, we ran both systems in parallel. The monolith served the real response, while the new service processed the same request and we compared results:

```java
@Component
public class ShadowTrafficComparator {

    private final WebClient monolithClient;
    private final WebClient microserviceClient;
    private final MeterRegistry metrics;

    public void compareResponses(ServerHttpRequest request) {
        // Execute both calls
        var monolithResponse = monolithClient.method(request.getMethod())
            .uri(request.getURI())
            .headers(h -> h.addAll(request.getHeaders()))
            .retrieve()
            .bodyToMono(String.class);

        var microserviceResponse = microserviceClient.method(request.getMethod())
            .uri(request.getURI())
            .headers(h -> h.addAll(request.getHeaders()))
            .retrieve()
            .bodyToMono(String.class);

        Mono.zip(monolithResponse, microserviceResponse)
            .subscribe(tuple -> {
                boolean match = compareJson(tuple.getT1(), tuple.getT2());
                metrics.counter("shadow.comparison",
                    "match", String.valueOf(match),
                    "endpoint", request.getURI().getPath()
                ).increment();

                if (!match) {
                    log.warn("Shadow mismatch on {}: monolith={}, service={}",
                        request.getURI(), tuple.getT1(), tuple.getT2());
                }
            });
    }
}
```

We ran shadow mode for 3 weeks. Initially, we had a 12% mismatch rate. By week 3, we were at 0.02% (edge cases in date formatting).

## Step 3: Database Decomposition

This is the hardest part. The monolith has one database with foreign keys across domains. You can't just split it overnight.

### Phase 1: Logical Separation (Weeks 1-4)

Create schema boundaries within the same database:

```sql
-- Create schemas for each bounded context
CREATE SCHEMA payments;
CREATE SCHEMA accounts;
CREATE SCHEMA notifications;

-- Move tables to appropriate schemas
ALTER TABLE transactions SET SCHEMA payments;
ALTER TABLE payment_methods SET SCHEMA payments;
ALTER TABLE users SET SCHEMA accounts;
ALTER TABLE user_profiles SET SCHEMA accounts;

-- Create views for cross-schema reads (temporary bridge)
CREATE VIEW payments.user_payment_info AS
SELECT id, email, default_payment_method_id
FROM accounts.users;
```

### Phase 2: Data Duplication via Events (Weeks 4-8)

```java
// Account service publishes events when user data changes
@Service
public class AccountEventPublisher {

    private final KafkaTemplate<String, AccountEvent> kafkaTemplate;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onAccountUpdated(AccountUpdatedEvent event) {
        kafkaTemplate.send("account-events",
            event.getUserId(),
            new AccountEvent(
                event.getUserId(),
                event.getEmail(),
                event.getPaymentMethodIds(),
                Instant.now()
            ));
    }
}

// Payment service maintains its own read model of user data
@Service
public class AccountDataConsumer {

    private final PaymentUserRepository paymentUserRepository;

    @KafkaListener(topics = "account-events", groupId = "payment-service")
    public void onAccountEvent(AccountEvent event) {
        PaymentUser user = paymentUserRepository.findById(event.userId())
            .orElse(new PaymentUser(event.userId()));

        user.setEmail(event.email());
        user.setPaymentMethodIds(event.paymentMethodIds());
        user.setLastSyncedAt(event.timestamp());

        paymentUserRepository.save(user);
    }
}
```

### Phase 3: Physical Separation (Weeks 8-12)

```java
// Payment service now has its own database
@Configuration
@Profile("microservice")
public class PaymentDataSourceConfig {

    @Bean
    @Primary
    public DataSource paymentDataSource() {
        var config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://payment-db:5432/payments");
        config.setMaximumPoolSize(20);
        return new HikariDataSource(config);
    }
}
```

### The Dual-Write Problem

During migration, some writes need to update both the old and new database. We used the Outbox Pattern:

```java
@Service
public class PaymentService {

    private final PaymentRepository paymentRepository;
    private final OutboxRepository outboxRepository;

    @Transactional
    public Payment createPayment(PaymentRequest request) {
        Payment payment = new Payment(request);
        paymentRepository.save(payment);

        // Same transaction — atomically consistent
        outboxRepository.save(new OutboxEvent(
            "payment.created",
            payment.getId().toString(),
            toJson(payment)
        ));

        return payment;
    }
}

// Separate process polls outbox and publishes to Kafka
@Scheduled(fixedDelay = 100)
public void publishOutboxEvents() {
    List<OutboxEvent> unpublished = outboxRepository.findUnpublished(100);

    for (OutboxEvent event : unpublished) {
        kafkaTemplate.send(event.getTopic(), event.getKey(), event.getPayload())
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    event.markPublished();
                    outboxRepository.save(event);
                }
            });
    }
}
```

## Step 4: Feature Toggles for Cutover

Never rely on deployment for cutover. Use feature toggles so you can switch between monolith and microservice at runtime:

```java
@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    private final FeatureToggleService toggles;
    private final MonolithPaymentClient monolithClient;
    private final PaymentService microservicePayment;

    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestBody PaymentRequest request) {

        if (toggles.isEnabled("payment.use-microservice", request.getMerchantId())) {
            return ResponseEntity.ok(microservicePayment.process(request));
        } else {
            return ResponseEntity.ok(monolithClient.process(request));
        }
    }
}
```

```java
@Service
public class FeatureToggleService {

    private final RedisTemplate<String, String> redis;

    public boolean isEnabled(String feature, String segmentKey) {
        // Check kill switch first (instant rollback)
        String killSwitch = redis.opsForValue().get("toggle:kill:" + feature);
        if ("true".equals(killSwitch)) {
            return false; // Route everything back to monolith
        }

        // Percentage-based rollout
        String percentage = redis.opsForValue().get("toggle:rollout:" + feature);
        if (percentage != null) {
            int rolloutPercent = Integer.parseInt(percentage);
            int hash = Math.abs(segmentKey.hashCode() % 100);
            return hash < rolloutPercent;
        }

        return false;
    }
}
```

Our rollout was:
- **Week 1** — 1% of traffic (canary)
- **Week 2** — 10% of traffic (early validation)
- **Week 3** — 50% of traffic (load testing the new service under real load)
- **Week 4** — 100% of traffic
- **Week 5** — Monolith payment code deprecated (but kept as fallback for 3 more months)

## Step 5: The Rollback Strategy

Every cutover needs a rollback plan that can execute in under 60 seconds:

```java
@RestController
@RequestMapping("/internal/migration")
public class MigrationController {

    private final FeatureToggleService toggleService;
    private final MeterRegistry metrics;

    @PostMapping("/rollback/{service}")
    @PreAuthorize("hasRole('MIGRATION_ADMIN')")
    public ResponseEntity<String> rollback(@PathVariable String service) {
        // Kill switch — instant rollback to monolith
        toggleService.setKillSwitch(service, true);

        // Record rollback event
        metrics.counter("migration.rollback", "service", service).increment();

        // Alert the team
        alertService.send(AlertLevel.P1,
            "Migration rollback triggered for: " + service);

        return ResponseEntity.ok("Rolled back " + service + " to monolith");
    }

    @PostMapping("/resume/{service}")
    @PreAuthorize("hasRole('MIGRATION_ADMIN')")
    public ResponseEntity<String> resume(@PathVariable String service,
                                          @RequestParam int percentage) {
        toggleService.setKillSwitch(service, false);
        toggleService.setRolloutPercentage(service, percentage);
        return ResponseEntity.ok("Resumed " + service + " at " + percentage + "%");
    }
}
```

We triggered rollback three times during the migration:
1. **Week 2** — New service had a memory leak under sustained load. Rollback in 8 seconds.
2. **Week 3** — Edge case in currency conversion. Rollback in 12 seconds. Fixed and resumed next day.
3. **Week 4** — False alarm. Monitoring showed error spike but it was a downstream timeout, not our service.

## Step 6: Shared-Nothing Architecture

The most critical principle: services must not share databases, caches, or in-memory state. Every shared resource becomes a coupling point that negates the benefits of microservices.

```java
// WRONG: Shared cache that couples services
@Cacheable(cacheNames = "users", key = "#userId")
public User getUser(String userId) {
    return userRepository.findById(userId);
}

// RIGHT: Each service owns its data and exposes it via API
@Service
public class PaymentUserService {

    private final Cache<String, PaymentUser> localCache;
    private final AccountServiceClient accountClient;

    public PaymentUser getPaymentUser(String userId) {
        return localCache.get(userId, key -> {
            // Fetch from account service API, cache locally
            AccountResponse response = accountClient.getAccount(userId);
            return new PaymentUser(response.id(), response.email(),
                                   response.paymentMethods());
        });
    }
}
```

### Communication Patterns

**Synchronous (HTTP/gRPC)** — Use only when you need the response to continue processing

**Asynchronous (Events)** — Use for notifications, data replication, eventual consistency

**Choreography** — Services react to events independently (simpler, harder to trace)

**Orchestration** — A central orchestrator coordinates the workflow (more complex, easier to trace)

We chose orchestration for the payment flow (needed consistency) and choreography for notifications and analytics (eventual consistency acceptable).

## The Timeline

Here's how long each phase actually took us:

**Bounded context analysis** — 3 weeks (2 weeks Event Storming + 1 week dependency analysis)

**First service extraction (Notifications)** — 6 weeks (lowest risk, minimal data coupling)

**Payment service extraction** — 14 weeks (complex domain, critical path, database decomposition)

**Account service extraction** — 10 weeks

**Full migration (6 services)** — 11 months (not the 6 months we originally estimated)

## What We Got Wrong

- **Underestimated data migration.** We allocated 20% of effort. It consumed 45%. Cross-domain foreign keys were everywhere.

- **Started with the hardest service.** We should have extracted Notifications first (we eventually did). It had the least coupling and gave the team confidence.

- **Didn't invest in observability early enough.** When you go from one monolith to 6 services, you need distributed tracing from day one. We added it in month 4 and wished we had it from month 1.

- **Shared libraries became coupling.** We created a `commons` library with DTOs. Every change to it required coordinating deployments across services. Eventually, we duplicated DTOs per service and accepted the redundancy.

## What We Got Right

- **Strangler fig, not big bang.** The monolith ran in production the entire time. We never had a "migration weekend."

- **Feature toggles for everything.** Rollback was always one Redis command away.

- **Shadow traffic validation.** We caught 200+ edge cases before they hit production.

- **Starting with team boundaries.** Each service was owned by one team. Conway's Law actually helped us here.

## The End Result

**Deployment frequency** — Every 2 weeks → Multiple times per day (per service)

**Deployment duration** — 4 hours → 8 minutes

**Team independence** — 47 developers blocking each other → 8 teams deploying independently

**Incident blast radius** — One bug takes everything down → Failures contained to one service

**Infrastructure cost** — 15% increase (more overhead per service) but offset by independent scaling

Was it worth it? For our scale and team size — absolutely. But I wouldn't recommend it for a team under 20 developers with a codebase under 200K lines. The operational complexity tax is real.
