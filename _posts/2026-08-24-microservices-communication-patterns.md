---
title: "7 Microservices Communication Patterns — When to Use Which"
date: 2026-08-24
categories: [Spring Boot, Microservices]
tags: [microservices, spring-boot, kafka, system-design, architecture]
description: "A decision guide for choosing between REST, gRPC, Kafka, event-driven, saga, API gateway, and service mesh — with Spring Boot implementations, failure modes, and real-world tradeoffs."
mermaid: true
---
## The Communication Problem

Microservices are simple individually. The complexity lives in how they talk to each other. Pick the wrong communication pattern and you'll end up with a distributed monolith — all the operational overhead of microservices with none of the benefits.

Over 17 years of building distributed systems in Singapore's financial services sector, I've seen teams default to REST for everything. REST is fine for CRUD. It's terrible for event propagation, long-running workflows, and high-throughput data pipelines.

Here are the 7 patterns I reach for, when each one fits, and critically, when they don't.

---

## Pattern 1: Synchronous REST (HTTP/JSON)

The default. The one everyone knows. And the one most often misused.

### Spring Boot Implementation

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final RestClient restClient;

    public OrderController(RestClient.Builder builder) {
        this.restClient = builder.baseUrl("http://inventory-service").build();
    }

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest request) {
        // Synchronous call to inventory service
        InventoryResponse inventory = restClient.post()
            .uri("/api/inventory/reserve")
            .body(new ReserveRequest(request.getSkus()))
            .retrieve()
            .body(InventoryResponse.class);

        if (!inventory.isAvailable()) {
            return ResponseEntity.badRequest()
                .body(OrderResponse.outOfStock());
        }

        Order order = orderService.create(request, inventory);
        return ResponseEntity.ok(OrderResponse.from(order));
    }
}
```

### When to Use

- **Simple CRUD operations** — creating, reading, updating resources
- **Low latency requirements** where you need an immediate response
- **Public APIs** consumed by external clients
- **Synchronous workflows** where the caller must wait for the result

### When NOT to Use

- **High fan-out calls** — if one request triggers 5+ downstream calls, latency compounds
- **Long-running operations** — anything over 2-3 seconds should be async
- **Unreliable networks** — REST gives you temporal coupling; both services must be up simultaneously
- **Event notification** — if you're polling REST endpoints for changes, you're doing it wrong

### Failure Handling

```java
@Configuration
public class ResilienceConfig {

    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        return builder
            .baseUrl("http://inventory-service")
            .requestInterceptor(new RetryInterceptor(3, Duration.ofMillis(500)))
            .build();
    }
}

// With Resilience4j for circuit breaking
@CircuitBreaker(name = "inventory", fallbackMethod = "fallbackInventory")
@Retry(name = "inventory")
@TimeLimiter(name = "inventory")
public CompletableFuture<InventoryResponse> checkInventory(List<String> skus) {
    return CompletableFuture.supplyAsync(() ->
        restClient.post()
            .uri("/api/inventory/check")
            .body(new CheckRequest(skus))
            .retrieve()
            .body(InventoryResponse.class)
    );
}

public CompletableFuture<InventoryResponse> fallbackInventory(List<String> skus, Exception ex) {
    log.warn("Inventory service unavailable, returning cached data", ex);
    return CompletableFuture.completedFuture(cachedInventoryService.getLastKnown(skus));
}
```

---

## Pattern 2: gRPC (High-Performance Internal Communication)

When REST's JSON serialization and HTTP/1.1 overhead becomes a bottleneck, gRPC delivers binary Protocol Buffers over HTTP/2 with streaming support.

### Proto Definition

```protobuf
syntax = "proto3";

service PricingService {
    rpc CalculatePrice (PriceRequest) returns (PriceResponse);
    rpc StreamPrices (PriceRequest) returns (stream PriceUpdate); // server streaming
}

message PriceRequest {
    string product_id = 1;
    string currency = 2;
    int32 quantity = 3;
}

message PriceResponse {
    string product_id = 1;
    int64 price_cents = 2;
    string currency = 3;
}
```

### Spring Boot gRPC Server

```java
@GrpcService
public class PricingGrpcService extends PricingServiceGrpc.PricingServiceImplBase {

    private final PricingEngine pricingEngine;

    @Override
    public void calculatePrice(PriceRequest request,
                               StreamObserver<PriceResponse> responseObserver) {
        BigDecimal price = pricingEngine.calculate(
            request.getProductId(),
            request.getCurrency(),
            request.getQuantity()
        );

        responseObserver.onNext(PriceResponse.newBuilder()
            .setProductId(request.getProductId())
            .setPriceCents(price.multiply(BigDecimal.valueOf(100)).longValue())
            .setCurrency(request.getCurrency())
            .build());
        responseObserver.onCompleted();
    }
}
```

### When to Use

- **Internal service-to-service calls** with high throughput requirements
- **Polyglot environments** — proto files generate clients in any language
- **Streaming data** — real-time price feeds, log streaming, live updates
- **Low-latency paths** — binary serialization is 5-10x faster than JSON

### When NOT to Use

- **Public-facing APIs** — browsers can't natively call gRPC (need gRPC-Web or a gateway)
- **Simple CRUD** where REST is perfectly adequate
- **Teams unfamiliar with protobuf** — the learning curve is real
- **Debugging ease is priority** — binary payloads are harder to inspect than JSON

### Failure Handling

gRPC has built-in deadlines (not timeouts — deadlines propagate across service hops):

```java
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("pricing-service:9090")
    .usePlaintext()
    .build();

PricingServiceGrpc.PricingServiceBlockingStub stub =
    PricingServiceGrpc.newBlockingStub(channel)
        .withDeadlineAfter(500, TimeUnit.MILLISECONDS);
```

---

## Pattern 3: Asynchronous Messaging (Kafka)

When you need decoupled, durable communication that survives service restarts and handles backpressure gracefully.

### Spring Boot Producer

```java
@Service
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderCreated(Order order) {
        OrderEvent event = new OrderEvent(
            order.getId(),
            OrderEventType.CREATED,
            order.toEventPayload(),
            Instant.now()
        );

        kafkaTemplate.send("order-events", order.getId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish order event: {}", order.getId(), ex);
                    outboxService.storeForRetry(event); // transactional outbox fallback
                } else {
                    log.debug("Published order event to partition {}",
                        result.getRecordMetadata().partition());
                }
            });
    }
}
```

### Spring Boot Consumer

```java
@Component
public class InventoryEventConsumer {

    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderEvent(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {

        log.info("Processing order event: {} from partition {} offset {}",
            event.getOrderId(), partition, offset);

        switch (event.getType()) {
            case CREATED -> inventoryService.reserveStock(event.getPayload());
            case CANCELLED -> inventoryService.releaseStock(event.getPayload());
            default -> log.warn("Unknown event type: {}", event.getType());
        }
    }
}
```

### When to Use

- **Decoupling services** — producer doesn't know or care who consumes
- **High throughput** — Kafka handles millions of events per second
- **Event replay** — consumers can reprocess events from any offset
- **Cross-team boundaries** — teams evolve independently

### When NOT to Use

- **Request-reply patterns** — if you need an immediate response, messaging adds latency
- **Simple point-to-point** — for two services, Kafka's operational overhead may not be justified
- **Ordering matters globally** — Kafka guarantees order only within a partition
- **Sub-millisecond latency** — messaging adds 5-50ms of latency

### Failure Handling

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent>
            kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate),
            new FixedBackOff(1000L, 3)  // 3 retries, 1 second apart
        ));
        return factory;
    }
}
```

Dead letter topics catch messages that fail after retries. You can then alert, inspect, and replay them.

---

## Pattern 4: Event-Driven (Domain Events)

More than messaging — this is about designing your system around events as first-class citizens.

### Spring Boot Implementation with ApplicationEventPublisher

```java
// Domain event
public record OrderPlacedEvent(
    String orderId,
    String customerId,
    List<LineItem> items,
    BigDecimal totalAmount,
    Instant occurredAt
) {}

// Publishing service
@Service
@Transactional
public class OrderService {

    private final ApplicationEventPublisher eventPublisher;
    private final OrderRepository orderRepository;

    public Order placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command);
        orderRepository.save(order);

        eventPublisher.publishEvent(new OrderPlacedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getItems(),
            order.getTotal(),
            Instant.now()
        ));

        return order;
    }
}

// Listeners react independently
@Component
public class NotificationListener {

    @EventListener
    @Async
    public void onOrderPlaced(OrderPlacedEvent event) {
        emailService.sendOrderConfirmation(event.customerId(), event.orderId());
    }
}

@Component
public class AnalyticsListener {

    @EventListener
    @Async
    public void onOrderPlaced(OrderPlacedEvent event) {
        analyticsService.trackPurchase(event.customerId(), event.totalAmount());
    }
}
```

### When to Use

- **Multiple reactions to one action** — order placed triggers notification, analytics, inventory update
- **Loose coupling within a service** — modules react without direct dependencies
- **Audit trail requirements** — events naturally form a log
- **Eventual consistency is acceptable**

### When NOT to Use

- **Strong consistency required** — if all reactions must succeed or none, events are the wrong tool
- **Simple linear workflows** — if A always calls B which calls C, direct calls are clearer
- **Debugging complex flows** — event-driven systems are harder to trace

---

## Pattern 5: Saga Pattern (Distributed Transactions)

When a business operation spans multiple services and you need all-or-nothing semantics without distributed transactions (which don't scale).

### Orchestration-Based Saga in Spring Boot

```java
@Service
public class OrderSagaOrchestrator {

    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final ShippingService shippingService;

    @Transactional
    public OrderResult executeOrderSaga(OrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        List<Runnable> compensations = new ArrayList<>();

        try {
            // Step 1: Reserve inventory
            ReservationResult reservation = inventoryService.reserve(request.getItems());
            compensations.add(() -> inventoryService.cancelReservation(reservation.getId()));

            // Step 2: Process payment
            PaymentResult payment = paymentService.charge(request.getPaymentDetails());
            compensations.add(() -> paymentService.refund(payment.getTransactionId()));

            // Step 3: Schedule shipping
            ShipmentResult shipment = shippingService.schedule(request.getShippingAddress());
            compensations.add(() -> shippingService.cancel(shipment.getShipmentId()));

            return OrderResult.success(sagaId, reservation, payment, shipment);

        } catch (Exception ex) {
            log.error("Saga {} failed at step, executing compensations", sagaId, ex);
            executeCompensations(compensations);
            return OrderResult.failed(sagaId, ex.getMessage());
        }
    }

    private void executeCompensations(List<Runnable> compensations) {
        // Execute in reverse order
        Lists.reverse(compensations).forEach(compensation -> {
            try {
                compensation.run();
            } catch (Exception ex) {
                log.error("Compensation failed — manual intervention required", ex);
                alertService.raiseIncident("SAGA_COMPENSATION_FAILURE", ex);
            }
        });
    }
}
```

### When to Use

- **Multi-service transactions** — payment + inventory + shipping must all succeed
- **Long-running operations** — processes that take minutes or hours
- **When 2PC (XA) is impractical** — which is always in microservices

### When NOT to Use

- **Single service operations** — use local ACID transactions
- **Idempotency isn't guaranteed** — compensations must be safe to retry
- **Simple fire-and-forget** — don't overcomplicate notifications

---

## Pattern 6: API Gateway Pattern

A single entry point that routes, aggregates, and cross-cuts concerns.

### Spring Cloud Gateway Implementation

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .circuitBreaker(c -> c
                        .setName("orderService")
                        .setFallbackUri("forward:/fallback/orders"))
                    .retry(retryConfig -> retryConfig
                        .setRetries(3)
                        .setStatuses(HttpStatus.SERVICE_UNAVAILABLE))
                    .requestRateLimiter(rl -> rl
                        .setRateLimiter(redisRateLimiter()))
                    .addRequestHeader("X-Request-Source", "gateway"))
                .uri("lb://order-service"))
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .tokenRelay())
                .uri("lb://user-service"))
            .build();
    }

    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(100, 200); // 100 req/s, burst 200
    }
}
```

### When to Use

- **Authentication/authorization** at the edge
- **Rate limiting** before requests hit backend services
- **Response aggregation** — combine multiple service responses into one
- **Protocol translation** — REST externally, gRPC internally
- **API versioning** — route v1/v2 to different backends

### When NOT to Use

- **Internal service-to-service calls** — services should call each other directly
- **Simple applications** — a gateway adds latency and a single point of failure
- **When it becomes a bottleneck** — all traffic funnels through one layer

---

## Pattern 7: Service Mesh (Infrastructure-Level Communication)

When communication concerns (retries, mTLS, observability) are pushed to infrastructure instead of application code.

### Istio VirtualService Configuration

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: v2
          weight: 90
        - destination:
            host: order-service
            subset: v1
          weight: 10
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure
      timeout: 10s
```

### Spring Boot Application — Zero Communication Code

```java
// With service mesh, your application code has NO retry/circuit-breaker logic
@Service
public class OrderService {

    private final RestClient restClient;

    public InventoryStatus checkInventory(String sku) {
        // Just a plain HTTP call — mesh handles retries, timeouts, mTLS
        return restClient.get()
            .uri("http://inventory-service/api/inventory/{sku}", sku)
            .retrieve()
            .body(InventoryStatus.class);
    }
}
```

### When to Use

- **Large-scale deployments** (50+ services) where per-service resilience config is unmanageable
- **mTLS everywhere** without changing application code
- **Canary deployments and traffic splitting**
- **Uniform observability** — every call is traced without instrumentation

### When NOT to Use

- **Small teams/few services** — the operational overhead of a mesh is enormous
- **Latency-sensitive paths** — sidecar proxy adds 1-3ms per hop
- **Simple environments** — if you have 5 services, a mesh is overkill
- **Teams without Kubernetes expertise**

---

## My Decision Matrix

**Start with REST** for synchronous request-reply between 2-3 services.

**Add Kafka** when you need decoupling, event replay, or high throughput.

**Use gRPC** for latency-sensitive internal calls or streaming.

**Implement Sagas** when business transactions span multiple services.

**Deploy an API Gateway** when you have external consumers needing auth, rate limiting, and aggregation.

**Consider a Service Mesh** only when you have 50+ services and a platform team to manage it.

**Use Event-Driven patterns** within services to decouple modules and enable multiple reactions to business events.

The most common mistake I see: starting with a service mesh and saga orchestration for a system with 3 services. Start simple. Add complexity only when the problem demands it.
