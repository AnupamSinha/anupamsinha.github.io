---
title: "Building Event-Driven Architecture from Scratch — The Complete Guide"
date: 2026-08-24
categories: [Spring Boot, Architecture]
tags: [event-driven, spring-boot, kafka, microservices, architecture]
description: "A progressive guide to event-driven systems — starting from domain events and building up to Kafka, event sourcing, CQRS, and eventual consistency with full Spring Boot implementations at every stage."
mermaid: true
---
## Why Events Matter

Every significant system I've built in 17 years of enterprise Java in Singapore has eventually adopted event-driven patterns. Not because it's trendy — because synchronous, coupled systems hit a wall.

That wall looks like this: your order service directly calls inventory, payment, notification, and analytics services. One service goes down and the entire order flow breaks. You want to add a loyalty points service, and now you're modifying the order service — a service owned by a different team, deployed on a different cadence, with its own test suite you don't fully understand.

Events solve this by inverting the dependency. The order service publishes what happened. Everyone else reacts independently. Adding a new consumer doesn't require changing the producer. Services fail independently. You can replay events to rebuild state.

Let me build this up progressively — from simple in-process events to a full Kafka-based event-sourced system.

---

## Level 1: Domain Events (In-Process)

The simplest form. Events within a single Spring Boot application to decouple modules.

### Define the Event

```java
public record OrderPlacedEvent(
    String orderId,
    String customerId,
    List<OrderLineItem> items,
    BigDecimal totalAmount,
    Instant occurredAt
) {
    public OrderPlacedEvent {
        Objects.requireNonNull(orderId);
        Objects.requireNonNull(customerId);
        occurredAt = occurredAt != null ? occurredAt : Instant.now();
    }
}

public record OrderLineItem(
    String productId,
    int quantity,
    BigDecimal unitPrice
) {}
```

### Publish from Your Domain Logic

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order placeOrder(PlaceOrderCommand command) {
        // Business logic
        Order order = Order.create(
            command.customerId(),
            command.items(),
            command.shippingAddress()
        );
        order.validate();
        orderRepository.save(order);

        // Publish event AFTER successful persistence
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
```

### React with Listeners

```java
@Component
public class InventoryReservationListener {

    private final InventoryService inventoryService;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlaced(OrderPlacedEvent event) {
        inventoryService.reserveItems(event.orderId(), event.items());
    }
}

@Component
public class OrderNotificationListener {

    private final NotificationService notificationService;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void onOrderPlaced(OrderPlacedEvent event) {
        notificationService.sendOrderConfirmation(event.customerId(), event.orderId());
    }
}

@Component
public class AnalyticsListener {

    private final AnalyticsService analyticsService;

    @EventListener
    @Async
    public void onOrderPlaced(OrderPlacedEvent event) {
        analyticsService.trackPurchase(event.customerId(), event.totalAmount());
    }
}
```

### Why @TransactionalEventListener?

With `@EventListener`, the handler runs inside the same transaction. If notification fails, the order creation rolls back. That's rarely what you want.

`@TransactionalEventListener(phase = AFTER_COMMIT)` ensures the event fires only after the transaction commits successfully. The order is persisted before any listeners run.

---

## Level 2: Event Bus with Transactional Outbox

In-process events are great within one application. But they don't survive restarts and can't cross service boundaries. The transactional outbox pattern bridges this gap.

### The Problem

```java
// DANGEROUS: Not atomic
@Transactional
public void placeOrder(PlaceOrderCommand command) {
    orderRepository.save(order);          // This succeeds
    kafkaTemplate.send("orders", event);  // This might fail — now DB and Kafka are inconsistent
}
```

If Kafka is down when you try to publish, your database has the order but no event was emitted. Downstream services never know about it.

### The Outbox Solution

Write the event to a database table in the same transaction as your business data. A separate process polls the outbox and publishes to Kafka.

```java
@Entity
@Table(name = "event_outbox")
public class OutboxEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String aggregateType;

    @Column(nullable = false)
    private String aggregateId;

    @Column(nullable = false)
    private String eventType;

    @Column(columnDefinition = "TEXT", nullable = false)
    private String payload;

    @Column(nullable = false)
    private Instant createdAt;

    @Column(nullable = false)
    private boolean published;

    private Instant publishedAt;
}
```

### Writing to the Outbox

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    @Transactional  // Single transaction covers both writes
    public Order placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command);
        orderRepository.save(order);

        OutboxEvent outboxEvent = new OutboxEvent();
        outboxEvent.setAggregateType("Order");
        outboxEvent.setAggregateId(order.getId());
        outboxEvent.setEventType("OrderPlaced");
        outboxEvent.setPayload(objectMapper.writeValueAsString(
            new OrderPlacedEvent(order.getId(), order.getCustomerId(),
                order.getItems(), order.getTotal(), Instant.now())
        ));
        outboxEvent.setCreatedAt(Instant.now());
        outboxEvent.setPublished(false);

        outboxRepository.save(outboxEvent);

        return order;
    }
}
```

### Publishing from the Outbox

```java
@Component
public class OutboxPublisher {

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000)  // Poll every second
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pending = outboxRepository
            .findTop100ByPublishedFalseOrderByCreatedAtAsc();

        for (OutboxEvent event : pending) {
            try {
                String topic = event.getAggregateType().toLowerCase() + "-events";
                kafkaTemplate.send(topic, event.getAggregateId(), event.getPayload())
                    .get(5, TimeUnit.SECONDS);  // Wait for ack

                event.setPublished(true);
                event.setPublishedAt(Instant.now());
                outboxRepository.save(event);
            } catch (Exception ex) {
                log.error("Failed to publish event {}", event.getId(), ex);
                break;  // Stop processing — maintain order
            }
        }
    }
}
```

This guarantees at-least-once delivery. Events may be published more than once (if the publisher crashes after Kafka ack but before marking as published), so consumers must be idempotent.

---

## Level 3: Kafka Integration — Distributed Event Streaming

Now we scale beyond a single service.

### Kafka Producer Configuration

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "all");  // Wait for all replicas
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);  // Exactly-once semantics
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

### Kafka Consumer with Error Handling

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "inventory-service");
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);  // Manual commit
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate()),
            new FixedBackOff(2000L, 3)
        ));
        return factory;
    }
}
```

### Idempotent Consumer

```java
@Component
public class InventoryEventHandler {

    private final InventoryService inventoryService;
    private final ProcessedEventRepository processedEventRepo;

    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void handle(ConsumerRecord<String, String> record, Acknowledgment ack) {
        String eventId = extractEventId(record);

        // Idempotency check
        if (processedEventRepo.existsById(eventId)) {
            log.info("Event {} already processed, skipping", eventId);
            ack.acknowledge();
            return;
        }

        try {
            OrderEvent event = objectMapper.readValue(record.value(), OrderEvent.class);

            switch (event.getType()) {
                case "OrderPlaced" -> inventoryService.reserveStock(event);
                case "OrderCancelled" -> inventoryService.releaseStock(event);
            }

            processedEventRepo.save(new ProcessedEvent(eventId, Instant.now()));
            ack.acknowledge();

        } catch (Exception ex) {
            log.error("Failed to process event {}", eventId, ex);
            // Don't ack — will be retried by error handler
            throw new RuntimeException(ex);
        }
    }
}
```

---

## Level 4: Event Sourcing — Events as the Source of Truth

Instead of storing the current state, store every event that led to the current state. The current state is derived by replaying events.

### Event Store

```java
@Entity
@Table(name = "event_store")
public class StoredEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long sequenceNumber;

    @Column(nullable = false)
    private String aggregateId;

    @Column(nullable = false)
    private String aggregateType;

    @Column(nullable = false)
    private int version;

    @Column(nullable = false)
    private String eventType;

    @Column(columnDefinition = "TEXT", nullable = false)
    private String eventData;

    @Column(nullable = false)
    private Instant occurredAt;
}
```

### Aggregate Rebuilt from Events

```java
public class OrderAggregate {

    private String orderId;
    private String customerId;
    private OrderStatus status;
    private List<OrderLineItem> items = new ArrayList<>();
    private BigDecimal totalAmount = BigDecimal.ZERO;
    private int version = 0;

    // Rebuild from event history
    public static OrderAggregate reconstitute(List<DomainEvent> events) {
        OrderAggregate aggregate = new OrderAggregate();
        for (DomainEvent event : events) {
            aggregate.apply(event);
            aggregate.version++;
        }
        return aggregate;
    }

    // Apply event to update state
    private void apply(DomainEvent event) {
        switch (event) {
            case OrderCreatedEvent e -> {
                this.orderId = e.orderId();
                this.customerId = e.customerId();
                this.status = OrderStatus.CREATED;
                this.items = new ArrayList<>(e.items());
                this.totalAmount = e.totalAmount();
            }
            case OrderConfirmedEvent e -> {
                this.status = OrderStatus.CONFIRMED;
            }
            case OrderShippedEvent e -> {
                this.status = OrderStatus.SHIPPED;
            }
            case OrderCancelledEvent e -> {
                this.status = OrderStatus.CANCELLED;
            }
            default -> throw new IllegalArgumentException("Unknown event: " + event.getClass());
        }
    }

    // Command handler that produces new events
    public List<DomainEvent> confirm() {
        if (this.status != OrderStatus.CREATED) {
            throw new IllegalStateException("Can only confirm orders in CREATED status");
        }
        OrderConfirmedEvent event = new OrderConfirmedEvent(this.orderId, Instant.now());
        apply(event);
        return List.of(event);
    }
}
```

### Event Store Repository

```java
@Service
public class EventStoreService {

    private final EventStoreRepository repository;
    private final ObjectMapper objectMapper;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public void saveEvents(String aggregateId, String aggregateType,
                          List<DomainEvent> events, int expectedVersion) {
        // Optimistic concurrency check
        int currentVersion = repository.findMaxVersionByAggregateId(aggregateId)
            .orElse(0);

        if (currentVersion != expectedVersion) {
            throw new ConcurrencyException(
                "Expected version %d but found %d".formatted(expectedVersion, currentVersion));
        }

        int version = expectedVersion;
        for (DomainEvent event : events) {
            version++;
            StoredEvent stored = new StoredEvent();
            stored.setAggregateId(aggregateId);
            stored.setAggregateType(aggregateType);
            stored.setVersion(version);
            stored.setEventType(event.getClass().getSimpleName());
            stored.setEventData(objectMapper.writeValueAsString(event));
            stored.setOccurredAt(event.occurredAt());
            repository.save(stored);
        }

        // Publish for projections and external consumers
        events.forEach(eventPublisher::publishEvent);
    }

    public OrderAggregate loadAggregate(String aggregateId) {
        List<StoredEvent> storedEvents = repository
            .findByAggregateIdOrderByVersionAsc(aggregateId);

        List<DomainEvent> events = storedEvents.stream()
            .map(this::deserializeEvent)
            .toList();

        return OrderAggregate.reconstitute(events);
    }
}
```

### When Event Sourcing Makes Sense

- **Audit requirements** — financial systems, healthcare, compliance
- **Temporal queries** — "What was the order state at 3:00 PM yesterday?"
- **Event replay** — rebuild read models, fix bugs by replaying corrected logic
- **Complex domain logic** — event history reveals business intent

### When It's Overkill

- **Simple CRUD** — if your domain is "save and retrieve," events add complexity for no gain
- **No audit needs** — if nobody cares about history, don't pay the storage cost
- **Team unfamiliarity** — event sourcing has a steep learning curve

---

## Level 5: CQRS — Separating Reads from Writes

Command Query Responsibility Segregation pairs naturally with event sourcing. The write side stores events. The read side maintains optimized projections.

### Write Side (Command)

```java
@RestController
@RequestMapping("/api/orders")
public class OrderCommandController {

    private final EventStoreService eventStore;

    @PostMapping
    public ResponseEntity<String> createOrder(@RequestBody CreateOrderCommand command) {
        String orderId = UUID.randomUUID().toString();

        OrderCreatedEvent event = new OrderCreatedEvent(
            orderId, command.customerId(), command.items(),
            command.calculateTotal(), Instant.now()
        );

        eventStore.saveEvents(orderId, "Order", List.of(event), 0);

        return ResponseEntity.status(HttpStatus.CREATED).body(orderId);
    }

    @PostMapping("/{orderId}/confirm")
    public ResponseEntity<Void> confirmOrder(@PathVariable String orderId) {
        OrderAggregate order = eventStore.loadAggregate(orderId);
        List<DomainEvent> newEvents = order.confirm();
        eventStore.saveEvents(orderId, "Order", newEvents, order.getVersion());

        return ResponseEntity.ok().build();
    }
}
```

### Read Side (Query) — Projection

```java
@Component
public class OrderReadModelProjection {

    private final OrderReadRepository readRepository;

    @EventListener
    @Async
    public void on(OrderCreatedEvent event) {
        OrderReadModel readModel = new OrderReadModel();
        readModel.setOrderId(event.orderId());
        readModel.setCustomerId(event.customerId());
        readModel.setStatus("CREATED");
        readModel.setTotalAmount(event.totalAmount());
        readModel.setItemCount(event.items().size());
        readModel.setCreatedAt(event.occurredAt());
        readRepository.save(readModel);
    }

    @EventListener
    @Async
    public void on(OrderConfirmedEvent event) {
        OrderReadModel readModel = readRepository.findById(event.orderId())
            .orElseThrow();
        readModel.setStatus("CONFIRMED");
        readModel.setConfirmedAt(event.occurredAt());
        readRepository.save(readModel);
    }

    @EventListener
    @Async
    public void on(OrderShippedEvent event) {
        OrderReadModel readModel = readRepository.findById(event.orderId())
            .orElseThrow();
        readModel.setStatus("SHIPPED");
        readModel.setShippedAt(event.occurredAt());
        readRepository.save(readModel);
    }
}

@RestController
@RequestMapping("/api/orders")
public class OrderQueryController {

    private final OrderReadRepository readRepository;

    @GetMapping("/{orderId}")
    public OrderReadModel getOrder(@PathVariable String orderId) {
        return readRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    @GetMapping("/customer/{customerId}")
    public List<OrderReadModel> getCustomerOrders(@PathVariable String customerId) {
        return readRepository.findByCustomerIdOrderByCreatedAtDesc(customerId);
    }
}
```

### The Power of Multiple Projections

With CQRS, you can maintain multiple read models optimized for different queries:

- **OrderReadModel** in PostgreSQL — for individual order lookups
- **CustomerOrderSummary** in Redis — for fast dashboard rendering
- **OrderAnalytics** in Elasticsearch — for full-text search and aggregations
- **OrderTimeline** in a time-series DB — for temporal analysis

Each projection subscribes to the same events and builds its own optimized view. You can add new projections without touching the write side.

---

## Level 6: Eventual Consistency — Embracing the Reality

In a distributed event-driven system, you give up strong consistency. The read model may lag behind the write model by milliseconds to seconds.

### Handling the Read Lag

```java
@RestController
public class OrderController {

    @PostMapping("/api/orders")
    public ResponseEntity<OrderCreatedResponse> createOrder(@RequestBody CreateOrderCommand cmd) {
        String orderId = orderCommandService.createOrder(cmd);

        // Return the write model data immediately — don't query the read model
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(new OrderCreatedResponse(orderId, "CREATED", cmd.items()));
    }

    @GetMapping("/api/orders/{orderId}")
    public ResponseEntity<OrderReadModel> getOrder(
            @PathVariable String orderId,
            @RequestHeader(value = "X-Min-Version", required = false) Integer minVersion) {

        OrderReadModel order = orderReadRepository.findById(orderId)
            .orElseThrow();

        // Client can request a minimum version to ensure consistency
        if (minVersion != null && order.getVersion() < minVersion) {
            return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .header("Retry-After", "1")
                .build();
        }

        return ResponseEntity.ok(order);
    }
}
```

### Strategies for Eventual Consistency

**Optimistic UI** — return the command result immediately. The UI shows the expected state. If the projection catches up with a different result, notify the user.

**Polling with version** — client sends the expected version. Server returns 503 if the read model hasn't caught up yet.

**Websocket notifications** — push updates to clients when projections complete.

**Causal consistency** — include a version/timestamp in the command response. Subsequent reads must see at least that version.

---

## The Architecture at Scale

When you put all levels together, the architecture looks like this:

**Write Path:**
Client → API Gateway → Command Service → Event Store (DB) → Outbox Publisher → Kafka

**Read Path:**
Kafka → Projection Services → Read Databases (Postgres, Redis, Elasticsearch)
Client → API Gateway → Query Service → Read Databases

**Key Properties:**
- Write side and read side scale independently
- Adding new consumers doesn't change producers
- Events are the single source of truth
- Any read model can be rebuilt from scratch by replaying events
- Services fail independently without cascading

---

## Common Mistakes I've Seen

- **Events as commands** — "CreateOrderEvent" is wrong. Events are past tense: "OrderCreatedEvent"
- **Fat events with everything** — include only what happened, not the entire aggregate state
- **No idempotency** — at-least-once delivery means consumers MUST handle duplicates
- **Ignoring event schema evolution** — add fields, never remove. Use versioning
- **Event sourcing everywhere** — not every service needs it. Use it where the audit trail and temporal queries justify the complexity

Start at Level 1. Move to Level 2 when you cross service boundaries. Add Kafka when you need durability and fan-out. Adopt event sourcing only where the domain demands it. CQRS when read and write patterns diverge significantly.

The progression is the point. Don't start at Level 5.
