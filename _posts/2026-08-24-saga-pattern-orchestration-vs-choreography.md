---
title: "Saga Pattern Deep Dive: Orchestration vs Choreography (With Code)"
date: 2026-08-24
categories: [Spring Boot, Architecture]
tags: [saga-pattern, microservices, spring-boot, kafka, distributed-systems]
description: "Implement the same business process two ways — orchestration and choreography — with full Spring Boot and Kafka code. Compare complexity, debugging, coupling, and failure handling to know when to choose which."
mermaid: true
---
## The Distributed Transaction Problem

The moment you split a monolith into microservices, you lose ACID transactions across service boundaries. You can't wrap a database call in Service A and another in Service B inside a single `@Transactional`. They're different databases, different processes, often different machines.

The classic example: an e-commerce order that needs to:

1. **Create the order** (Order Service)
2. **Process payment** (Payment Service)
3. **Reserve inventory** (Inventory Service)
4. **Schedule shipping** (Shipping Service)

If payment succeeds but inventory reservation fails, you need to refund the payment. If shipping fails, you need to release inventory and refund payment. This is the Saga pattern — a sequence of local transactions with compensating actions for rollback.

After 17 years building distributed systems in Singapore, I've implemented both orchestration and choreography sagas in production. They solve the same problem with fundamentally different tradeoffs. Let me show you both, with working code.

## Choreography: Event-Driven Saga

In choreography, there's no central coordinator. Each service listens for events, performs its work, and publishes the next event. Services react to what happened without any service knowing the full workflow.

### The Event Flow

```
Order Created → Payment Service listens → Payment Processed
Payment Processed → Inventory Service listens → Inventory Reserved
Inventory Reserved → Shipping Service listens → Shipping Scheduled
Shipping Scheduled → Order Service listens → Order Completed

// Failure compensation flows:
Payment Failed → Order Service listens → Order Cancelled
Inventory Failed → Payment Service listens → Payment Refunded
Shipping Failed → Inventory Service listens → Inventory Released
                → Payment Service listens → Payment Refunded
```

### Event Definitions

```java
public sealed interface OrderEvent {

    record OrderCreated(String orderId, String userId, List<OrderItem> items,
                        BigDecimal totalAmount) implements OrderEvent {}

    record OrderCompleted(String orderId) implements OrderEvent {}

    record OrderCancelled(String orderId, String reason) implements OrderEvent {}
}

public sealed interface PaymentEvent {

    record PaymentProcessed(String orderId, String paymentId,
                            BigDecimal amount) implements PaymentEvent {}

    record PaymentFailed(String orderId, String reason) implements PaymentEvent {}

    record PaymentRefunded(String orderId, String paymentId,
                           BigDecimal amount) implements PaymentEvent {}
}

public sealed interface InventoryEvent {

    record InventoryReserved(String orderId,
                             Map<String, Integer> reservedItems) implements InventoryEvent {}

    record InventoryReservationFailed(String orderId,
                                      String reason) implements InventoryEvent {}

    record InventoryReleased(String orderId) implements InventoryEvent {}
}

public sealed interface ShippingEvent {

    record ShippingScheduled(String orderId, String trackingNumber,
                             LocalDate estimatedDelivery) implements ShippingEvent {}

    record ShippingFailed(String orderId, String reason) implements ShippingEvent {}
}
```

### Order Service (Choreography)

```java
@Service
public class OrderServiceChoreography {

    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.builder()
                .id(UUID.randomUUID().toString())
                .userId(request.getUserId())
                .items(request.getItems())
                .totalAmount(request.calculateTotal())
                .status(OrderStatus.PENDING)
                .build();

        orderRepository.save(order);

        // Publish event — other services react
        kafkaTemplate.send("order-events", order.getId(),
                new OrderEvent.OrderCreated(order.getId(), order.getUserId(),
                        order.getItems(), order.getTotalAmount()));

        return order;
    }

    @KafkaListener(topics = "shipping-events", groupId = "order-service")
    public void handleShippingEvent(ShippingEvent event) {
        switch (event) {
            case ShippingEvent.ShippingScheduled e -> {
                Order order = orderRepository.findById(e.orderId()).orElseThrow();
                order.setStatus(OrderStatus.COMPLETED);
                order.setTrackingNumber(e.trackingNumber());
                orderRepository.save(order);
            }
            case ShippingEvent.ShippingFailed e -> {
                Order order = orderRepository.findById(e.orderId()).orElseThrow();
                order.setStatus(OrderStatus.FAILED);
                order.setFailureReason(e.reason());
                orderRepository.save(order);
            }
        }
    }

    @KafkaListener(topics = "payment-events", groupId = "order-service")
    public void handlePaymentEvent(PaymentEvent event) {
        if (event instanceof PaymentEvent.PaymentFailed e) {
            Order order = orderRepository.findById(e.orderId()).orElseThrow();
            order.setStatus(OrderStatus.CANCELLED);
            order.setFailureReason("Payment failed: " + e.reason());
            orderRepository.save(order);

            kafkaTemplate.send("order-events", e.orderId(),
                    new OrderEvent.OrderCancelled(e.orderId(), e.reason()));
        }
    }
}
```

### Payment Service (Choreography)

```java
@Service
public class PaymentServiceChoreography {

    private final PaymentRepository paymentRepository;
    private final PaymentGateway paymentGateway;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "order-events", groupId = "payment-service")
    public void handleOrderEvent(OrderEvent event) {
        if (event instanceof OrderEvent.OrderCreated e) {
            processPayment(e);
        }
    }

    @KafkaListener(topics = "inventory-events", groupId = "payment-service")
    public void handleInventoryEvent(InventoryEvent event) {
        if (event instanceof InventoryEvent.InventoryReservationFailed e) {
            // Compensate: refund the payment
            refundPayment(e.orderId(), "Inventory unavailable");
        }
    }

    @KafkaListener(topics = "shipping-events", groupId = "payment-service")
    public void handleShippingEvent(ShippingEvent event) {
        if (event instanceof ShippingEvent.ShippingFailed e) {
            // Compensate: refund the payment
            refundPayment(e.orderId(), "Shipping failed");
        }
    }

    private void processPayment(OrderEvent.OrderCreated event) {
        try {
            PaymentResult result = paymentGateway.charge(
                    event.userId(), event.totalAmount());

            Payment payment = Payment.builder()
                    .id(result.getPaymentId())
                    .orderId(event.orderId())
                    .amount(event.totalAmount())
                    .status(PaymentStatus.COMPLETED)
                    .build();
            paymentRepository.save(payment);

            kafkaTemplate.send("payment-events", event.orderId(),
                    new PaymentEvent.PaymentProcessed(event.orderId(),
                            result.getPaymentId(), event.totalAmount()));

        } catch (PaymentException e) {
            kafkaTemplate.send("payment-events", event.orderId(),
                    new PaymentEvent.PaymentFailed(event.orderId(), e.getMessage()));
        }
    }

    private void refundPayment(String orderId, String reason) {
        Payment payment = paymentRepository.findByOrderId(orderId).orElseThrow();
        paymentGateway.refund(payment.getId(), payment.getAmount());
        payment.setStatus(PaymentStatus.REFUNDED);
        paymentRepository.save(payment);

        kafkaTemplate.send("payment-events", orderId,
                new PaymentEvent.PaymentRefunded(orderId, payment.getId(), payment.getAmount()));
    }
}
```

### Inventory Service (Choreography)

```java
@Service
public class InventoryServiceChoreography {

    private final InventoryRepository inventoryRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "payment-events", groupId = "inventory-service")
    public void handlePaymentEvent(PaymentEvent event) {
        if (event instanceof PaymentEvent.PaymentProcessed e) {
            reserveInventory(e.orderId());
        }
    }

    @KafkaListener(topics = "shipping-events", groupId = "inventory-service")
    public void handleShippingEvent(ShippingEvent event) {
        if (event instanceof ShippingEvent.ShippingFailed e) {
            releaseInventory(e.orderId());
        }
    }

    private void reserveInventory(String orderId) {
        // Fetch order details (in practice, the event would carry items)
        try {
            Map<String, Integer> reserved = doReservation(orderId);

            kafkaTemplate.send("inventory-events", orderId,
                    new InventoryEvent.InventoryReserved(orderId, reserved));

        } catch (InsufficientStockException e) {
            kafkaTemplate.send("inventory-events", orderId,
                    new InventoryEvent.InventoryReservationFailed(orderId, e.getMessage()));
        }
    }

    private void releaseInventory(String orderId) {
        // Release previously reserved stock
        inventoryRepository.releaseReservation(orderId);

        kafkaTemplate.send("inventory-events", orderId,
                new InventoryEvent.InventoryReleased(orderId));
    }

    private Map<String, Integer> doReservation(String orderId) {
        // Actual inventory reservation logic with optimistic locking
        // ...
        return Map.of(); // placeholder
    }
}
```

## Orchestration: Central Coordinator Saga

In orchestration, a central **Saga Orchestrator** knows the full workflow, tells each service what to do, and handles compensation on failure.

### The Saga State Machine

```java
public enum SagaStep {
    STARTED,
    PAYMENT_PENDING,
    PAYMENT_COMPLETED,
    INVENTORY_PENDING,
    INVENTORY_RESERVED,
    SHIPPING_PENDING,
    SHIPPING_SCHEDULED,
    COMPLETED,
    COMPENSATING_SHIPPING,
    COMPENSATING_INVENTORY,
    COMPENSATING_PAYMENT,
    FAILED
}
```

```java
@Entity
@Table(name = "order_sagas")
public class OrderSaga {

    @Id
    private String sagaId;
    private String orderId;

    @Enumerated(EnumType.STRING)
    private SagaStep currentStep;

    private String paymentId;
    private String reservationId;
    private String trackingNumber;
    private String failureReason;

    private LocalDateTime startedAt;
    private LocalDateTime completedAt;
    private int version; // Optimistic locking
}
```

### The Saga Orchestrator

```java
@Service
public class OrderSagaOrchestrator {

    private final OrderSagaRepository sagaRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final OrderRepository orderRepository;

    @Transactional
    public String startSaga(Order order) {
        OrderSaga saga = OrderSaga.builder()
                .sagaId(UUID.randomUUID().toString())
                .orderId(order.getId())
                .currentStep(SagaStep.STARTED)
                .startedAt(LocalDateTime.now())
                .build();
        sagaRepository.save(saga);

        // First step: initiate payment
        advanceToPayment(saga, order);
        return saga.getSagaId();
    }

    private void advanceToPayment(OrderSaga saga, Order order) {
        saga.setCurrentStep(SagaStep.PAYMENT_PENDING);
        sagaRepository.save(saga);

        kafkaTemplate.send("payment-commands", saga.getSagaId(),
                new PaymentCommand.ProcessPayment(saga.getSagaId(), order.getId(),
                        order.getUserId(), order.getTotalAmount()));
    }

    private void advanceToInventory(OrderSaga saga) {
        saga.setCurrentStep(SagaStep.INVENTORY_PENDING);
        sagaRepository.save(saga);

        kafkaTemplate.send("inventory-commands", saga.getSagaId(),
                new InventoryCommand.ReserveItems(saga.getSagaId(), saga.getOrderId()));
    }

    private void advanceToShipping(OrderSaga saga) {
        saga.setCurrentStep(SagaStep.SHIPPING_PENDING);
        sagaRepository.save(saga);

        kafkaTemplate.send("shipping-commands", saga.getSagaId(),
                new ShippingCommand.ScheduleShipment(saga.getSagaId(), saga.getOrderId()));
    }

    private void completeSaga(OrderSaga saga) {
        saga.setCurrentStep(SagaStep.COMPLETED);
        saga.setCompletedAt(LocalDateTime.now());
        sagaRepository.save(saga);

        Order order = orderRepository.findById(saga.getOrderId()).orElseThrow();
        order.setStatus(OrderStatus.COMPLETED);
        order.setTrackingNumber(saga.getTrackingNumber());
        orderRepository.save(order);
    }

    // --- Reply Handlers ---

    @KafkaListener(topics = "saga-replies", groupId = "saga-orchestrator")
    public void handleReply(SagaReply reply) {
        OrderSaga saga = sagaRepository.findById(reply.getSagaId()).orElseThrow();

        switch (reply) {
            case SagaReply.PaymentSuccess r -> {
                saga.setPaymentId(r.paymentId());
                saga.setCurrentStep(SagaStep.PAYMENT_COMPLETED);
                sagaRepository.save(saga);
                advanceToInventory(saga);
            }
            case SagaReply.PaymentFailed r -> {
                saga.setFailureReason("Payment failed: " + r.reason());
                failSaga(saga);
            }
            case SagaReply.InventoryReserved r -> {
                saga.setReservationId(r.reservationId());
                saga.setCurrentStep(SagaStep.INVENTORY_RESERVED);
                sagaRepository.save(saga);
                advanceToShipping(saga);
            }
            case SagaReply.InventoryFailed r -> {
                saga.setFailureReason("Inventory failed: " + r.reason());
                compensatePayment(saga);
            }
            case SagaReply.ShippingScheduled r -> {
                saga.setTrackingNumber(r.trackingNumber());
                completeSaga(saga);
            }
            case SagaReply.ShippingFailed r -> {
                saga.setFailureReason("Shipping failed: " + r.reason());
                compensateInventory(saga);
            }
            case SagaReply.CompensationComplete r -> {
                handleCompensationComplete(saga, r);
            }
            default -> log.warn("Unexpected reply for saga {}: {}", saga.getSagaId(), reply);
        }
    }

    // --- Compensation ---

    private void compensateInventory(OrderSaga saga) {
        saga.setCurrentStep(SagaStep.COMPENSATING_INVENTORY);
        sagaRepository.save(saga);

        kafkaTemplate.send("inventory-commands", saga.getSagaId(),
                new InventoryCommand.ReleaseItems(saga.getSagaId(), saga.getOrderId()));
    }

    private void compensatePayment(OrderSaga saga) {
        saga.setCurrentStep(SagaStep.COMPENSATING_PAYMENT);
        sagaRepository.save(saga);

        kafkaTemplate.send("payment-commands", saga.getSagaId(),
                new PaymentCommand.RefundPayment(saga.getSagaId(), saga.getOrderId(),
                        saga.getPaymentId()));
    }

    private void handleCompensationComplete(OrderSaga saga, SagaReply.CompensationComplete reply) {
        switch (saga.getCurrentStep()) {
            case COMPENSATING_INVENTORY -> compensatePayment(saga);
            case COMPENSATING_PAYMENT -> failSaga(saga);
            default -> log.warn("Unexpected compensation complete in step: {}",
                    saga.getCurrentStep());
        }
    }

    private void failSaga(OrderSaga saga) {
        saga.setCurrentStep(SagaStep.FAILED);
        saga.setCompletedAt(LocalDateTime.now());
        sagaRepository.save(saga);

        Order order = orderRepository.findById(saga.getOrderId()).orElseThrow();
        order.setStatus(OrderStatus.FAILED);
        order.setFailureReason(saga.getFailureReason());
        orderRepository.save(order);
    }
}
```

### Command and Reply Definitions

```java
public sealed interface PaymentCommand {
    record ProcessPayment(String sagaId, String orderId,
                          String userId, BigDecimal amount) implements PaymentCommand {}
    record RefundPayment(String sagaId, String orderId,
                         String paymentId) implements PaymentCommand {}
}

public sealed interface InventoryCommand {
    record ReserveItems(String sagaId, String orderId) implements InventoryCommand {}
    record ReleaseItems(String sagaId, String orderId) implements InventoryCommand {}
}

public sealed interface ShippingCommand {
    record ScheduleShipment(String sagaId, String orderId) implements ShippingCommand {}
}

public sealed interface SagaReply {
    String getSagaId();

    record PaymentSuccess(String sagaId, String paymentId) implements SagaReply {
        public String getSagaId() { return sagaId; }
    }
    record PaymentFailed(String sagaId, String reason) implements SagaReply {
        public String getSagaId() { return sagaId; }
    }
    record InventoryReserved(String sagaId, String reservationId) implements SagaReply {
        public String getSagaId() { return sagaId; }
    }
    record InventoryFailed(String sagaId, String reason) implements SagaReply {
        public String getSagaId() { return sagaId; }
    }
    record ShippingScheduled(String sagaId, String trackingNumber) implements SagaReply {
        public String getSagaId() { return sagaId; }
    }
    record ShippingFailed(String sagaId, String reason) implements SagaReply {
        public String getSagaId() { return sagaId; }
    }
    record CompensationComplete(String sagaId, String step) implements SagaReply {
        public String getSagaId() { return sagaId; }
    }
}
```

### Payment Service (Orchestration)

```java
@Service
public class PaymentServiceOrchestration {

    private final PaymentGateway paymentGateway;
    private final PaymentRepository paymentRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "payment-commands", groupId = "payment-service")
    public void handleCommand(PaymentCommand command) {
        switch (command) {
            case PaymentCommand.ProcessPayment cmd -> processPayment(cmd);
            case PaymentCommand.RefundPayment cmd -> refundPayment(cmd);
        }
    }

    private void processPayment(PaymentCommand.ProcessPayment cmd) {
        try {
            PaymentResult result = paymentGateway.charge(cmd.userId(), cmd.amount());

            Payment payment = Payment.builder()
                    .id(result.getPaymentId())
                    .orderId(cmd.orderId())
                    .amount(cmd.amount())
                    .status(PaymentStatus.COMPLETED)
                    .build();
            paymentRepository.save(payment);

            kafkaTemplate.send("saga-replies", cmd.sagaId(),
                    new SagaReply.PaymentSuccess(cmd.sagaId(), result.getPaymentId()));

        } catch (PaymentException e) {
            kafkaTemplate.send("saga-replies", cmd.sagaId(),
                    new SagaReply.PaymentFailed(cmd.sagaId(), e.getMessage()));
        }
    }

    private void refundPayment(PaymentCommand.RefundPayment cmd) {
        Payment payment = paymentRepository.findById(cmd.paymentId()).orElseThrow();
        paymentGateway.refund(payment.getId(), payment.getAmount());
        payment.setStatus(PaymentStatus.REFUNDED);
        paymentRepository.save(payment);

        kafkaTemplate.send("saga-replies", cmd.sagaId(),
                new SagaReply.CompensationComplete(cmd.sagaId(), "payment"));
    }
}
```

## The Comparison: When to Use Which

### Complexity

**Choreography** — Complexity is distributed. Each service only knows about its immediate neighbors. Simple to start with 2–3 services. Becomes a tangled web at 5+ services because no single place shows the full flow.

**Orchestration** — Complexity is centralized. The orchestrator can get large, but the workflow is explicit and readable. Participant services are simpler — they just handle commands and reply.

### Debugging

**Choreography** — Debugging a failed order requires tracing events across multiple services, topics, and consumer groups. You need distributed tracing (Jaeger/Zipkin) and a way to correlate events by order ID across all services.

**Orchestration** — Query the saga table. The current step, all collected data, and failure reason are in one place. Dramatically easier to debug.

### Coupling

**Choreography** — Services are loosely coupled through events. Payment Service doesn't know Inventory Service exists. Adding a new step (e.g., fraud check) means adding a new listener — existing services don't change.

**Orchestration** — Services are decoupled from each other but coupled to the orchestrator. Adding a step requires modifying the orchestrator. Participant services only know about commands and replies, not the workflow.

### Failure Handling

**Choreography** — Each service must know what to compensate when. Compensation logic is scattered. If you add a new step, you must update multiple services' failure handlers. Missing a compensation path means inconsistent state.

**Orchestration** — All compensation logic lives in one place. The orchestrator knows exactly what was done and what needs undoing. It's a state machine — easy to reason about all paths.

### Scalability

**Choreography** — Scales independently. No single point of coordination.

**Orchestration** — The orchestrator is a bottleneck if not designed carefully. In practice, it's rarely the bottleneck because it's just routing messages, but it does need high availability.

## My Decision Framework

After implementing both approaches in production systems:

**Choose Choreography when** —

- You have 2–3 services in the workflow
- Teams are highly autonomous and own their services completely
- The workflow is unlikely to change frequently
- You have mature distributed tracing and event monitoring
- Services are developed by different teams who don't want a shared orchestrator

**Choose Orchestration when** —

- You have 4+ services in the workflow
- The business process has complex branching or conditional steps
- You need clear visibility into saga state for operations/support
- Compensation logic is complex and ordering matters
- One team owns the end-to-end business process
- You need to easily modify the workflow without touching all services

### My Default Choice

For most systems I build, I default to **orchestration**. The operational visibility alone is worth the centralization. When something fails at 3 AM, I want to query one table and know exactly what happened, what step failed, and what compensations ran.

Choreography is elegant in theory but painful in practice once you hit 5+ services. The "just add a listener" simplicity turns into "where does this event go and who handles failures?"

## Handling Edge Cases

### Idempotency

Both approaches must handle duplicate messages (Kafka at-least-once delivery):

```java
@Service
public class IdempotentPaymentProcessor {

    private final ProcessedCommandRepository processedCommandRepo;

    @Transactional
    public void processPayment(PaymentCommand.ProcessPayment cmd) {
        // Check if we've already processed this
        if (processedCommandRepo.existsBySagaIdAndStep(cmd.sagaId(), "process-payment")) {
            log.info("Already processed payment for saga {}. Skipping.", cmd.sagaId());
            return;
        }

        // Process payment...

        // Mark as processed
        processedCommandRepo.save(new ProcessedCommand(cmd.sagaId(), "process-payment"));
    }
}
```

### Timeout Handling

What if a service never responds?

```java
@Scheduled(fixedDelay = 60000)
@SchedulerLock(name = "saga-timeout-check")
public void checkSagaTimeouts() {
    LocalDateTime timeout = LocalDateTime.now().minusMinutes(5);

    List<OrderSaga> stuckSagas = sagaRepository
            .findByCurrentStepNotInAndStartedAtBefore(
                    List.of(SagaStep.COMPLETED, SagaStep.FAILED), timeout);

    stuckSagas.forEach(saga -> {
        log.warn("Saga {} stuck at step {} for > 5 minutes. Initiating compensation.",
                saga.getSagaId(), saga.getCurrentStep());
        // Begin compensation from current step
        initiateCompensation(saga);
    });
}
```

### Saga State Persistence

In orchestration, the saga state survives pod restarts because it's persisted in the database. On startup, the orchestrator can resume any in-progress sagas.

In choreography, if a service crashes between consuming an event and publishing its response, you need Kafka's transactional producer (exactly-once semantics) or manual deduplication.

## Final Thoughts

The Saga pattern is not optional in microservices — if you have distributed writes, you need distributed consistency. The question is only how you coordinate it.

Both approaches work in production. The choice is about your team's operational maturity, debugging needs, and how many services participate in the workflow.

Start with orchestration unless you have a compelling reason not to. You can always decompose an orchestrator into choreography later. Going the other direction — introducing an orchestrator into a choreography system — is much harder because you're fighting existing event contracts.
