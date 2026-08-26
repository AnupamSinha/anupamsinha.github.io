---
title: "The Architecture Decision That Saved My Company $2M/Year"
date: 2026-08-24
categories: [Spring Boot, Architecture]
tags: [software-architecture, microservices, spring-boot, cloud-costs, tech-lead]
description: "How migrating from an over-engineered microservices architecture to Spring Modulith cut our cloud bill by 70% and made our team happier"
mermaid: true
---
## The Architecture We Inherited

In early 2023, I joined a fintech company in Singapore as Technical Architect. The platform processed payment reconciliations for mid-size merchants — roughly 50,000 transactions per day at peak. Not small, but not Netflix either.

What greeted me was a Kubernetes cluster running 42 microservices. Forty-two. For a platform that, functionally, did four things: ingest transaction data, reconcile payments, generate reports, and send notifications.

The previous architect had followed the "microservices-first" playbook religiously. Every bounded context — no matter how small — got its own service, its own database, its own deployment pipeline. The result was impressive on architecture diagrams and catastrophic on the AWS bill.

## The Numbers That Triggered The Alarm

When I pulled the cloud cost reports, here's what I found:

**Monthly AWS Spend** — $247,000

**EKS Cluster (42 services)** — $89,000/month

**RDS Instances (18 databases)** — $54,000/month

**Data Transfer (inter-service)** — $31,000/month

**Kafka (MSK for event bus)** — $28,000/month

**Monitoring & Logging (Datadog)** — $22,000/month

**CI/CD Pipeline (build minutes)** — $12,000/month

**Other (NAT Gateway, Load Balancers, etc.)** — $11,000/month

That's nearly $3M/year for a platform serving 200 concurrent users and processing 50K transactions daily. The platform was profitable, but the margins were being eaten alive by infrastructure costs.

## The Real Pain Points Beyond Cost

Cost was the trigger, but the operational pain was worse:

- **Deployment took 4 hours** for a full release across all services
- **Debugging a single transaction** required tracing across 7-8 services
- **One developer** spent 80% of their time just maintaining Kubernetes configs
- **Incident response** averaged 45 minutes because nobody could trace causality
- **New feature development** required changes in 4+ repos for anything meaningful
- **Data consistency issues** appeared weekly due to eventual consistency between services that should have been strongly consistent

The team of 8 developers was spending 60% of their time on infrastructure and operational concerns, not business logic.

## The Decision Framework

I didn't just say "let's go monolith." I used a structured approach to evaluate our options:

**Step 1: Identify actual scaling boundaries**

I mapped every service's traffic patterns over 3 months. The result was telling:

- 3 services handled 90% of the traffic (ingestion, reconciliation, API gateway)
- 27 services received fewer than 100 requests/hour
- 12 services were purely internal (no external traffic at all)

**Step 2: Map data coupling**

I traced database queries and found:

- 14 services shared data through synchronous API calls that could have been module-level method calls
- 8 services used Kafka events just to maintain data copies that could live in shared tables
- Only 3 services had genuinely independent data models

**Step 3: Calculate the "microservice tax" per service**

For each service, the fixed operational cost was roughly:

**Base container resources** — $120/month (even idle)

**Database instance** — $200-500/month

**CI/CD pipeline** — $50/month

**Monitoring/alerting** — $80/month

**Developer cognitive overhead** — unquantifiable but real

When a service handles 50 requests/hour, paying $500+/month in fixed costs makes no sense.

**Step 4: Define what genuinely needs independent deployment**

After honest analysis, only 5 services needed independent scaling or deployment:

1. Transaction ingestion (bursty traffic)
2. Payment gateway integration (third-party dependency isolation)
3. Report generation (CPU-intensive, long-running)
4. Notification service (independent SLA)
5. API Gateway (entry point)

Everything else was **organizational complexity masquerading as technical necessity**.

## The Migration: Spring Modulith

I chose Spring Modulith as the target architecture. Here's why:

```java
// Project structure - clean module boundaries without network overhead
com.company.reconciliation/
├── transaction/          // Module: Transaction ingestion & storage
│   ├── TransactionModule.java
│   ├── api/
│   ├── internal/
│   └── events/
├── reconciliation/       // Module: Core reconciliation logic
│   ├── ReconciliationModule.java
│   ├── api/
│   ├── internal/
│   └── events/
├── merchant/            // Module: Merchant management
│   ├── MerchantModule.java
│   ├── api/
│   └── internal/
├── reporting/           // Module: Report generation
│   ├── ReportingModule.java
│   ├── api/
│   └── internal/
└── shared/              // Shared kernel
    ├── domain/
    └── infrastructure/
```

Spring Modulith gave us **logical boundaries with compile-time enforcement** without the network overhead:

```java
@ApplicationModule(
    allowedDependencies = {"shared", "transaction::api", "merchant::api"}
)
class ReconciliationModule {
}
```

The key insight: module boundaries are enforced at compile time using ArchUnit rules. You get the same separation guarantees without serialization, network latency, or distributed transaction headaches.

```java
@Configuration
public class ModulithArchitectureRules {

    @Bean
    ApplicationModuleTest.Customizer customizer() {
        return test -> test
            .verifyModularity()
            .verifyNoCircularDependencies();
    }
}
```

## Inter-Module Communication

Instead of REST calls and Kafka events between 42 services, we used Spring's ApplicationEventPublisher for async communication and direct method calls for synchronous needs:

```java
// Publishing domain events - same guarantees, zero network overhead
@Service
@Transactional
public class ReconciliationService {

    private final ApplicationEventPublisher events;
    private final TransactionApi transactionApi; // Module's public API

    public ReconciliationResult reconcile(ReconciliationRequest request) {
        List<Transaction> transactions = transactionApi.findUnreconciled(
            request.merchantId(), request.dateRange()
        );

        ReconciliationResult result = performReconciliation(transactions);

        // Domain event - handled asynchronously but in-process
        events.publishEvent(new ReconciliationCompleted(
            result.id(), result.merchantId(), result.matchedCount()
        ));

        return result;
    }
}

// Event listener in another module - clean separation maintained
@Component
class NotificationEventListener {

    @ApplicationModuleListener
    void onReconciliationCompleted(ReconciliationCompleted event) {
        // Send merchant notification
        notificationService.notifyMerchant(
            event.merchantId(),
            "Reconciliation completed: " + event.matchedCount() + " matched"
        );
    }
}
```

## The Migration Strategy: Strangler Fig, Module by Module

We didn't do a big bang rewrite. Over 4 months:

**Month 1:** Built the modular monolith skeleton, migrated the 12 internal-only services (lowest risk)

**Month 2:** Consolidated the 14 tightly-coupled services into 4 modules, merged their databases

**Month 3:** Migrated the merchant management and reporting clusters

**Month 4:** Cut over the remaining services, decommissioned old infrastructure

Each module was wrapped with integration tests before migration. We ran old and new in parallel for 2 weeks per batch, comparing outputs.

## The After: Cost Breakdown

**Monthly AWS Spend** — $73,000 (down from $247,000)

**ECS (3 services + 1 monolith)** — $18,000/month

**RDS (3 databases, consolidated)** — $15,000/month

**Data Transfer** — $4,000/month

**Kafka (retained for external integrations only)** — $8,000/month

**Monitoring (switched to Grafana stack)** — $6,000/month

**CI/CD** — $3,000/month

**Other** — $19,000/month

**Annual savings: $2.09M**

## What We Kept as Separate Services

Remember those 5 services I identified as genuinely needing independence?

1. **Transaction Ingestion** — Kept separate, scales independently during batch uploads
2. **Payment Gateway Proxy** — Kept separate, isolates third-party failures
3. **Report Generation** — Moved to ECS Fargate tasks (spin up on demand, scale to zero)
4. **Notification Service** — Merged into the monolith (turned out it didn't need independence)
5. **API Gateway** — Replaced with AWS API Gateway (managed service)

The final architecture: 1 modular monolith + 2 independent services + managed services where appropriate.

## Performance Improvements

The consolidation wasn't just about cost:

**API Response Time (P95)** — 340ms down to 45ms (no inter-service network hops)

**Deployment Time** — 4 hours down to 12 minutes

**Incident Resolution** — 45 minutes down to 8 minutes (single process, single log stream)

**New Feature Delivery** — 3 weeks average down to 4 days

**On-Call Alerts/Week** — 23 down to 3

## Lessons Learned

**1. Microservices are an organizational scaling pattern, not a technical one**

If you have 8 developers, you don't need 42 services. You need clean module boundaries. Microservices solve the problem of 200 developers needing to deploy independently.

**2. The "microservice tax" is real and compounds**

Every service you add carries a fixed cost floor — infrastructure, monitoring, cognitive overhead. For low-traffic services, this tax dominates the actual compute cost.

**3. Data coupling is the honest test**

If two services constantly need each other's data synchronously, they're not independent services. They're one service with a network boundary inserted for no reason.

**4. Cloud costs are a trailing indicator**

By the time the bill is painful, you've accumulated months of architectural debt. Review costs quarterly as an architecture concern, not just a finance one.

**5. "But what if we need to scale?"**

We can always extract a module into a separate service later. Spring Modulith's event-based communication makes this straightforward — swap ApplicationEventPublisher for Kafka, and the module becomes a service. Going modular monolith doesn't close the door to microservices. It just delays the complexity until you actually need it.

## The Framework for Your Decision

Ask yourself these questions:

- Do you have more than 50 developers needing independent deployments?
- Do your services genuinely have different scaling profiles?
- Is your data model truly independent between services?
- Can you afford the operational overhead per service?
- Are you spending more time on infrastructure than business logic?

If you answered "no" to the first four and "yes" to the last one, you might be over-architected.

## Final Thought

The best architecture is the one your team can actually operate and evolve. A perfectly designed microservices architecture that your team can't debug, deploy, or afford isn't good architecture — it's expensive theater.

That $2M/year went back into hiring two more developers and giving the existing team a significant raise. The team is happier, shipping faster, and sleeping better. That's what good architecture actually looks like.
