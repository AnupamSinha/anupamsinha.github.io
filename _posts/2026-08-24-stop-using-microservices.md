---
title: "Stop Using Microservices — A Senior Architect's Confession"
date: 2026-08-24
categories: [Spring Boot, Architecture]
tags: [microservices, software-architecture, spring-boot, monolith, engineering]
description: "Why your default choice of microservices is probably wrong, and what 17 years of Java architecture taught me about choosing the right approach"
mermaid: true
---
## My Confession

I have sinned. Between 2016 and 2021, I designed at least four systems as microservices that had no business being microservices. I did it because that's what "modern architecture" meant. I did it because conference talks told me monoliths were legacy. I did it because job postings demanded microservices experience.

Every single one of those systems would have been better as a modular monolith.

This isn't an anti-microservices post. I've built genuinely good microservices architectures where they were the right fit. This is an anti-default-microservices post. The default should be a well-structured monolith, and you should earn your way into microservices through genuine need — not resume-driven development.

## The Hidden Costs Nobody Mentions in Conferences

Conference talks about microservices show the happy path. Here's the talk nobody gives:

## Cost 1: Distributed Debugging Is a Nightmare

In a monolith, a bug is a stack trace. In microservices, a bug is a detective novel.

```java
// Monolith: one stack trace tells you everything
java.lang.NullPointerException: Order.customerId is null
    at com.company.billing.InvoiceService.createInvoice(InvoiceService.java:47)
    at com.company.order.OrderService.completeOrder(OrderService.java:123)
    at com.company.api.OrderController.submit(OrderController.java:34)
```

Compare that to the microservices version of the same bug:

```
// Microservices: good luck
Order Service logs: "Order ORD-12345 completed, emitting event"
Billing Service logs: "Received OrderCompleted event for ORD-12345"
Billing Service logs: "ERROR: Failed to create invoice - customer not found"
Customer Service logs: "GET /customers/null - 404 Not Found"
```

Now figure out *why* customerId is null. Was it null in the original request? Did the Order Service fail to enrich it? Did the event serialization drop it? Did the Customer Service return null for a valid ID due to a cache issue?

In a monolith, you set a breakpoint and step through. In microservices, you correlate logs across services using trace IDs — assuming your distributed tracing is properly configured, which in my experience, it never fully is.

**Real cost:** Our team spent an average of 3.5 hours debugging issues that would have taken 20 minutes in a monolith. Multiply that across 4-5 bugs per week, and you're losing an entire developer's productive time.

## Cost 2: Data Consistency Becomes Your Full-Time Job

The moment you split your data across services, you enter the world of eventual consistency. For some domains, that's fine. For many business applications, it's a source of endless bugs.

```java
// The saga pattern - what should be a simple transaction
@SagaOrchestrator
public class CreateOrderSaga {

    private final OrderService orderService;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;

    public OrderResult execute(CreateOrderCommand command) {
        // Step 1: Reserve inventory
        ReservationResult reservation = inventoryService.reserve(command.items());
        if (!reservation.isSuccess()) {
            return OrderResult.failed("Insufficient inventory");
        }

        // Step 2: Process payment
        PaymentResult payment = paymentService.charge(command.paymentDetails());
        if (!payment.isSuccess()) {
            // Compensate: release inventory
            inventoryService.release(reservation.id());
            return OrderResult.failed("Payment failed");
        }

        // Step 3: Create order
        try {
            Order order = orderService.create(command, reservation.id(), payment.id());
            return OrderResult.success(order);
        } catch (Exception e) {
            // Compensate: refund payment AND release inventory
            paymentService.refund(payment.id());
            inventoryService.release(reservation.id());
            return OrderResult.failed("Order creation failed");
        }
    }
}
```

Compare to the monolith version:

```java
// Monolith: one transaction, done
@Service
@Transactional
public class OrderService {

    public Order createOrder(CreateOrderCommand command) {
        inventoryRepository.reserve(command.items());
        Payment payment = paymentProcessor.charge(command.paymentDetails());
        Order order = orderRepository.save(new Order(command, payment));
        return order;
    }
    // If anything fails, the whole transaction rolls back. That's it.
}
```

The saga pattern adds hundreds of lines of compensation logic for something that's a 5-line transactional method in a monolith. And sagas still have edge cases — what if the compensating transaction fails? Now you need a dead letter queue, manual intervention workflows, and reconciliation jobs.

## Cost 3: The Operational Overhead Tax

Every microservice requires:

- Its own CI/CD pipeline
- Its own health checks and readiness probes
- Its own logging configuration
- Its own alerting rules
- Its own dependency management
- Its own database migrations
- Its own secret management
- Its own network policies
- Its own resource requests and limits

For a team of 6-8 developers managing 20+ services, this operational overhead consumes 40-60% of engineering capacity. I've seen it repeatedly across companies in Singapore's fintech and e-commerce sectors.

## Cost 4: The Integration Testing Problem

```java
// To test one user journey end-to-end, you need all of these running:
@SpringBootTest
@Testcontainers
class OrderJourneyIntegrationTest {

    @Container
    static PostgreSQLContainer<?> orderDb = new PostgreSQLContainer<>("postgres:15");

    @Container
    static PostgreSQLContainer<?> inventoryDb = new PostgreSQLContainer<>("postgres:15");

    @Container
    static PostgreSQLContainer<?> customerDb = new PostgreSQLContainer<>("postgres:15");

    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @Container
    static GenericContainer<?> orderService = new GenericContainer<>("order-service:latest");

    @Container
    static GenericContainer<?> inventoryService = new GenericContainer<>("inventory-service:latest");

    @Container
    static GenericContainer<?> paymentService = new GenericContainer<>("payment-service:latest");

    // Your test setup is now more complex than the actual test
}
```

In a monolith, you start one application context and test the whole flow. In microservices, your test infrastructure becomes a project of its own.

## Cost 5: Network Is Not Free

Every inter-service call adds:

**Latency** — 1-5ms per hop in a well-configured cluster, more with TLS

**Failure modes** — Timeouts, circuit breakers, retries, bulkheads

**Serialization cost** — JSON marshaling/unmarshaling on both ends

**Bandwidth** — Data transferred over the network that could be a method call

A request that touches 5 services accumulates 5-25ms of pure network overhead, plus serialization time, plus the cognitive overhead of managing failure at every boundary.

## When Microservices ARE the Right Choice

I'm not saying never use microservices. Here's when they genuinely make sense:

**1. Organizational scaling**

You have 100+ developers across multiple teams who need to deploy independently without coordinating. Conway's Law is real, and microservices align architecture with team boundaries.

**2. Genuinely different scaling requirements**

Your search service handles 10,000 req/s while your admin panel handles 10 req/s. Scaling them independently saves real money.

**3. Technology heterogeneity is required**

One component genuinely needs Python's ML libraries while the rest is Java. Different runtime requirements justify separate services.

**4. Fault isolation is critical**

A failure in your recommendation engine should not bring down your checkout flow. If the blast radius of a component failure is unacceptable, isolate it.

**5. Different deployment cadences with different risk profiles**

Your payment processing changes quarterly (heavily regulated) while your notification templates change daily. Independent deployment cycles with different testing rigor justify separation.

## The Modular Monolith Alternative

Here's what I recommend as the default architecture for teams under 30 developers:

```java
// Spring Modulith gives you boundaries without the distributed systems tax
@SpringBootApplication
@Modulithic(
    sharedModules = {"shared"},
    systemName = "E-Commerce Platform"
)
public class ECommerceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ECommerceApplication.class, args);
    }
}
```

Module structure with enforced boundaries:

```java
// Each module exposes only its public API
// Internal classes are package-private and inaccessible to other modules

// order/api/OrderApi.java - the public contract
public interface OrderApi {
    OrderDto createOrder(CreateOrderCommand command);
    Optional<OrderDto> findById(OrderId id);
}

// order/internal/OrderServiceImpl.java - hidden implementation
@Service
class OrderServiceImpl implements OrderApi {

    private final OrderRepository repository;
    private final ApplicationEventPublisher events;

    @Override
    @Transactional
    public OrderDto createOrder(CreateOrderCommand command) {
        Order order = Order.create(command);
        repository.save(order);
        events.publishEvent(new OrderCreated(order.id(), order.customerId()));
        return OrderDto.from(order);
    }
}
```

You get:

- **Clean boundaries** — enforced at compile time by ArchUnit/Modulith verification
- **Single deployment** — one artifact, one pipeline, one log stream
- **ACID transactions** — where your business logic actually needs them
- **Simple debugging** — one process, one debugger, full stack traces
- **Easy extraction** — if a module genuinely needs to become a service, the boundary already exists

## The Decision Checklist

Before choosing microservices, answer honestly:

**Team size** — Do you have more than 30 developers who can't coordinate releases?

**Traffic patterns** — Do components genuinely have 10x+ different scaling needs?

**Data independence** — Can each service own its data without constantly querying others?

**Operational maturity** — Do you have dedicated platform/DevOps engineers for the infrastructure overhead?

**Deployment independence** — Do teams actually need to deploy on completely different schedules?

If you answered "no" to most of these, a modular monolith is your better bet.

## The Career Incentive Problem

Let's address the elephant in the room. Microservices are resume-driven development. "Designed microservices architecture with 30 services on Kubernetes" looks better on a resume than "maintained and evolved a well-structured monolith."

Job postings demand "microservices experience." Architects get hired for complexity, not simplicity. This incentive structure pushes our industry toward unnecessary complexity.

I've started being honest in interviews: "I simplified a 42-service architecture into a modular monolith and saved $2M/year." The companies worth working for value that. The ones that don't aren't places where I want to make architecture decisions anyway.

## The Extraction Path: Monolith to Microservice

The beauty of starting with a modular monolith is that extraction is straightforward when genuinely needed:

```java
// Step 1: You already have clean module boundaries and event-based communication
@ApplicationModuleListener
void onOrderCreated(OrderCreated event) {
    // This handler already works asynchronously
    inventoryService.reserveForOrder(event.orderId(), event.items());
}

// Step 2: When you need to extract, swap the event infrastructure
// ApplicationEventPublisher -> Kafka/RabbitMQ
// Direct module API calls -> REST/gRPC clients
// Shared database -> separate database with data migration

// Step 3: Deploy the extracted module as a service
// The boundary already exists - you're just adding a network layer
```

Going from a modular monolith to microservices is a well-understood, low-risk operation. Going from a distributed mess back to sanity (which I've done) is a 4-month project.

## My Rule of Thumb

**Start with a modular monolith. Extract services only when you can articulate a specific, measurable benefit that justifies the operational cost.**

"It might need to scale independently someday" is not a justification. "This component currently handles 50x the traffic of everything else and has different SLA requirements" is.

## Conclusion

After 17 years of building systems in Java, here's my honest position: the best microservices architecture I ever designed was the one where I said "we don't need microservices here" and built a modular monolith instead.

The team shipped faster. The system was more reliable. The cloud bill was reasonable. And nobody woke up at 3 AM because a cascading failure across 8 services brought down the entire platform because one database connection pool was exhausted.

Simplicity is the ultimate sophistication. Start simple. Earn your complexity.
