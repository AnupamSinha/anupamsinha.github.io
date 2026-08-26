---
title: "Idempotency in Distributed Systems — The Pattern That Prevents Disasters"
date: 2026-08-24
categories: [Spring Boot, Architecture]
tags: [distributed-systems, spring-boot, idempotency, java, microservices]
description: "How to ensure your APIs handle retries gracefully, prevent double charges, and avoid duplicate orders — with full Spring Boot implementation examples."
mermaid: true
---
## The $4.2 Million Bug

In 2019, a fintech company I was consulting for had a payment gateway that occasionally timed out. Their mobile app retried the request. The gateway processed it again. Two charges. Sometimes three.

In one weekend, they double-charged 1,847 customers totaling $4.2 million in duplicate transactions. Their customer support team worked 72 hours straight processing refunds while their CTO drafted an incident report for the regulators.

The fix? A 200-line implementation of idempotency keys. Something that should have been there from day one.

This is not an edge case. In distributed systems, **the network is unreliable** — timeouts happen, retries happen, messages get delivered more than once. If your system isn't idempotent, it's just a matter of time before something goes wrong.

---

## What Idempotency Actually Means

An operation is idempotent if performing it multiple times produces the same result as performing it once.

**Naturally idempotent operations:**
- `GET /users/123` — Reading data (safe and idempotent)
- `PUT /users/123 {"name": "Alice"}` — Setting to a specific value
- `DELETE /users/123` — Deleting (already gone is the same as deleting)

**NOT naturally idempotent:**
- `POST /payments` — Creates a new payment each time
- `POST /orders` — Creates a new order each time
- `PATCH /accounts/123/balance {"add": 100}` — Adds 100 each time

The challenge is making non-idempotent operations behave idempotently without changing their semantics.

---

## Pattern 1: Idempotency Keys

The most universal pattern. The client generates a unique key and sends it with the request. The server uses this key to detect duplicates.

### How It Works

1. Client generates a UUID (idempotency key)
2. Client sends request with key in header: `Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000`
3. Server checks if this key has been seen before
4. If new: process request, store result with key
5. If seen: return the stored result without reprocessing

### Spring Boot Implementation

```java
@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    private final PaymentService paymentService;
    private final IdempotencyService idempotencyService;

    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody @Valid PaymentRequest request) {
        
        // Check for existing result
        Optional<PaymentResponse> existing = idempotencyService.getExistingResult(
            idempotencyKey, PaymentResponse.class);
        
        if (existing.isPresent()) {
            return ResponseEntity.ok(existing.get());
        }
        
        // Process new payment
        PaymentResponse response = paymentService.processPayment(request);
        
        // Store result for future duplicate detection
        idempotencyService.storeResult(idempotencyKey, response, Duration.ofHours(24));
        
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

### The Idempotency Service

```java
@Service
public class IdempotencyService {
    
    private final StringRedisTemplate redisTemplate;
    private final ObjectMapper objectMapper;
    
    private static final String KEY_PREFIX = "idempotency:";
    private static final String LOCK_PREFIX = "idempotency-lock:";
    
    public <T> Optional<T> getExistingResult(String idempotencyKey, Class<T> type) {
        String stored = redisTemplate.opsForValue().get(KEY_PREFIX + idempotencyKey);
        if (stored == null) return Optional.empty();
        
        try {
            return Optional.of(objectMapper.readValue(stored, type));
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Corrupted idempotency record", e);
        }
    }
    
    public <T> void storeResult(String idempotencyKey, T result, Duration ttl) {
        try {
            String serialized = objectMapper.writeValueAsString(result);
            redisTemplate.opsForValue().set(
                KEY_PREFIX + idempotencyKey, serialized, ttl);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Failed to serialize result", e);
        }
    }
    
    public boolean acquireLock(String idempotencyKey, Duration lockDuration) {
        Boolean acquired = redisTemplate.opsForValue().setIfAbsent(
            LOCK_PREFIX + idempotencyKey, "locked", lockDuration);
        return Boolean.TRUE.equals(acquired);
    }
    
    public void releaseLock(String idempotencyKey) {
        redisTemplate.delete(LOCK_PREFIX + idempotencyKey);
    }
}
```

### Handling Concurrent Duplicate Requests

There's a race condition: what if two identical requests arrive simultaneously, before the first one finishes processing? You need a lock:

```java
@PostMapping
public ResponseEntity<PaymentResponse> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody @Valid PaymentRequest request) {
    
    // Check for existing result first (fast path)
    Optional<PaymentResponse> existing = idempotencyService.getExistingResult(
        idempotencyKey, PaymentResponse.class);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get());
    }
    
    // Acquire lock to prevent concurrent processing
    if (!idempotencyService.acquireLock(idempotencyKey, Duration.ofSeconds(30))) {
        // Another request with same key is being processed
        throw new ConflictException("Request is already being processed");
    }
    
    try {
        // Double-check after acquiring lock
        existing = idempotencyService.getExistingResult(idempotencyKey, PaymentResponse.class);
        if (existing.isPresent()) {
            return ResponseEntity.ok(existing.get());
        }
        
        PaymentResponse response = paymentService.processPayment(request);
        idempotencyService.storeResult(idempotencyKey, response, Duration.ofHours(24));
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
        
    } finally {
        idempotencyService.releaseLock(idempotencyKey);
    }
}
```

---

## Pattern 2: Database Unique Constraints

When you need guaranteed deduplication with ACID properties, use the database itself.

### Implementation

```java
@Entity
@Table(name = "payment_transactions")
public class PaymentTransaction {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "idempotency_key", unique = true, nullable = false)
    private String idempotencyKey;
    
    @Column(name = "amount", nullable = false)
    private BigDecimal amount;
    
    @Column(name = "currency", nullable = false)
    private String currency;
    
    @Column(name = "status", nullable = false)
    @Enumerated(EnumType.STRING)
    private PaymentStatus status;
    
    @Column(name = "created_at", nullable = false)
    private Instant createdAt;
    
    @Column(name = "response_payload")
    private String responsePayload;
}
```

```java
@Service
@Transactional
public class PaymentService {
    
    private final PaymentTransactionRepository repository;
    private final PaymentGateway gateway;
    
    public PaymentResponse processPayment(String idempotencyKey, PaymentRequest request) {
        // Try to find existing transaction
        Optional<PaymentTransaction> existing = repository.findByIdempotencyKey(idempotencyKey);
        
        if (existing.isPresent()) {
            return toResponse(existing.get());
        }
        
        // Create transaction record first (claims the idempotency key)
        PaymentTransaction transaction = new PaymentTransaction();
        transaction.setIdempotencyKey(idempotencyKey);
        transaction.setAmount(request.amount());
        transaction.setCurrency(request.currency());
        transaction.setStatus(PaymentStatus.PENDING);
        transaction.setCreatedAt(Instant.now());
        
        try {
            repository.save(transaction); // Unique constraint prevents duplicates
        } catch (DataIntegrityViolationException e) {
            // Concurrent insert — fetch and return existing
            return toResponse(repository.findByIdempotencyKey(idempotencyKey).orElseThrow());
        }
        
        // Process with external gateway
        try {
            GatewayResponse gatewayResponse = gateway.charge(request);
            transaction.setStatus(PaymentStatus.COMPLETED);
            transaction.setResponsePayload(serialize(gatewayResponse));
            repository.save(transaction);
            return toResponse(transaction);
        } catch (Exception e) {
            transaction.setStatus(PaymentStatus.FAILED);
            repository.save(transaction);
            throw new PaymentProcessingException("Payment failed", e);
        }
    }
}
```

**When to use database constraints over Redis:**
- When you need ACID guarantees (financial transactions)
- When the idempotency key must survive Redis failures
- When you need to query idempotent operations later (audit trails)

---

## Pattern 3: Redis-Based Deduplication for Events

In event-driven architectures, consumers might receive the same message multiple times (at-least-once delivery). Redis is ideal for high-throughput deduplication:

```java
@Component
public class OrderEventConsumer {
    
    private final StringRedisTemplate redis;
    private final OrderService orderService;
    private static final Duration DEDUP_WINDOW = Duration.ofHours(24);
    
    @KafkaListener(topics = "order-events")
    public void handleOrderEvent(ConsumerRecord<String, OrderEvent> record) {
        String eventId = record.value().eventId();
        String dedupKey = "event-processed:" + eventId;
        
        // SET NX - only succeeds if key doesn't exist
        Boolean isNew = redis.opsForValue().setIfAbsent(dedupKey, "1", DEDUP_WINDOW);
        
        if (Boolean.FALSE.equals(isNew)) {
            log.info("Duplicate event detected, skipping: {}", eventId);
            return;
        }
        
        try {
            orderService.processOrderEvent(record.value());
        } catch (Exception e) {
            // Remove dedup key so retry can reprocess
            redis.delete(dedupKey);
            throw e;
        }
    }
}
```

**Important:** Notice that we remove the dedup key on failure. This ensures that if processing fails, the message can be retried. Only successful processing is considered "done."

---

## Pattern 4: Conditional Updates (Optimistic Locking)

For balance updates, counters, or any state modification, use version-based conditional updates:

```java
@Entity
@Table(name = "accounts")
public class Account {
    @Id
    private Long id;
    
    @Column(name = "balance")
    private BigDecimal balance;
    
    @Version
    private Long version;
}
```

```java
@Repository
public interface AccountRepository extends JpaRepository<Account, Long> {
    
    @Modifying
    @Query("UPDATE Account a SET a.balance = a.balance + :amount, a.version = a.version + 1 " +
           "WHERE a.id = :id AND a.version = :expectedVersion")
    int updateBalance(@Param("id") Long id, 
                      @Param("amount") BigDecimal amount,
                      @Param("expectedVersion") Long expectedVersion);
}
```

```java
@Service
public class AccountService {
    
    private final AccountRepository accountRepository;
    
    @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
    @Transactional
    public void credit(Long accountId, BigDecimal amount, String transactionRef) {
        Account account = accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
        
        int updated = accountRepository.updateBalance(
            accountId, amount, account.getVersion());
        
        if (updated == 0) {
            throw new OptimisticLockException(
                "Account was modified concurrently, retrying");
        }
    }
}
```

This pattern makes the update itself idempotent — if the version has already advanced, the update is a no-op (returns 0 rows updated), preventing double-application of the same change.

---

## A Complete Payment Processing Example

Let me put it all together in a production-grade implementation:

```java
@Service
@Slf4j
public class PaymentOrchestrator {
    
    private final IdempotencyService idempotencyService;
    private final PaymentTransactionRepository transactionRepo;
    private final PaymentGateway gateway;
    private final EventPublisher eventPublisher;
    
    @Transactional
    public PaymentResult processPayment(String idempotencyKey, PaymentCommand command) {
        // Layer 1: Check Redis cache for completed results
        Optional<PaymentResult> cached = idempotencyService.getExistingResult(
            idempotencyKey, PaymentResult.class);
        if (cached.isPresent()) {
            log.info("Returning cached result for key: {}", idempotencyKey);
            return cached.get();
        }
        
        // Layer 2: Check database for in-progress or completed transactions
        Optional<PaymentTransaction> existingTx = transactionRepo.findByIdempotencyKey(
            idempotencyKey);
        if (existingTx.isPresent()) {
            return handleExistingTransaction(existingTx.get());
        }
        
        // Layer 3: Acquire distributed lock for new processing
        if (!idempotencyService.acquireLock(idempotencyKey, Duration.ofSeconds(60))) {
            throw new PaymentInProgressException(idempotencyKey);
        }
        
        try {
            return executePayment(idempotencyKey, command);
        } finally {
            idempotencyService.releaseLock(idempotencyKey);
        }
    }
    
    private PaymentResult executePayment(String idempotencyKey, PaymentCommand command) {
        // Create DB record (claims the idempotency key via unique constraint)
        PaymentTransaction transaction = PaymentTransaction.builder()
            .idempotencyKey(idempotencyKey)
            .amount(command.amount())
            .currency(command.currency())
            .merchantId(command.merchantId())
            .status(PaymentStatus.INITIATED)
            .createdAt(Instant.now())
            .build();
        
        transactionRepo.save(transaction);
        
        // Call payment gateway
        GatewayResponse gatewayResponse = gateway.charge(ChargeRequest.from(command));
        
        // Update transaction
        transaction.setStatus(PaymentStatus.COMPLETED);
        transaction.setGatewayRef(gatewayResponse.referenceId());
        transactionRepo.save(transaction);
        
        // Build result
        PaymentResult result = PaymentResult.success(
            transaction.getId(), gatewayResponse.referenceId(), command.amount());
        
        // Cache in Redis for fast subsequent lookups
        idempotencyService.storeResult(idempotencyKey, result, Duration.ofHours(24));
        
        // Publish event
        eventPublisher.publish(new PaymentCompletedEvent(transaction.getId(), idempotencyKey));
        
        return result;
    }
    
    private PaymentResult handleExistingTransaction(PaymentTransaction tx) {
        return switch (tx.getStatus()) {
            case COMPLETED -> PaymentResult.success(
                tx.getId(), tx.getGatewayRef(), tx.getAmount());
            case FAILED -> PaymentResult.failed(tx.getId(), tx.getFailureReason());
            case INITIATED, PROCESSING -> throw new PaymentInProgressException(
                tx.getIdempotencyKey());
        };
    }
}
```

---

## Testing Idempotency

Testing is where most teams fall short. Here's how to properly verify idempotent behavior:

```java
@SpringBootTest
@Testcontainers
class PaymentIdempotencyTest {
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @Autowired
    private PaymentOrchestrator orchestrator;
    
    @Autowired
    private PaymentTransactionRepository transactionRepo;
    
    @Test
    @DisplayName("Same idempotency key returns same result without reprocessing")
    void duplicateRequestReturnsCachedResult() {
        String idempotencyKey = UUID.randomUUID().toString();
        PaymentCommand command = new PaymentCommand(
            BigDecimal.valueOf(100), "SGD", "merchant-1");
        
        // First call - processes payment
        PaymentResult first = orchestrator.processPayment(idempotencyKey, command);
        assertThat(first.status()).isEqualTo("COMPLETED");
        
        // Second call - returns cached result
        PaymentResult second = orchestrator.processPayment(idempotencyKey, command);
        assertThat(second).isEqualTo(first);
        
        // Verify only one transaction was created
        List<PaymentTransaction> transactions = transactionRepo.findAll();
        assertThat(transactions).hasSize(1);
    }
    
    @Test
    @DisplayName("Concurrent duplicate requests don't create duplicate transactions")
    void concurrentDuplicatesAreHandled() throws Exception {
        String idempotencyKey = UUID.randomUUID().toString();
        PaymentCommand command = new PaymentCommand(
            BigDecimal.valueOf(50), "SGD", "merchant-2");
        
        int threadCount = 10;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        CountDownLatch latch = new CountDownLatch(1);
        
        List<Future<PaymentResult>> futures = new ArrayList<>();
        for (int i = 0; i < threadCount; i++) {
            futures.add(executor.submit(() -> {
                latch.await(); // All threads start simultaneously
                return orchestrator.processPayment(idempotencyKey, command);
            }));
        }
        
        latch.countDown(); // Release all threads
        
        // Collect results (some may throw ConflictException)
        List<PaymentResult> successes = new ArrayList<>();
        for (Future<PaymentResult> future : futures) {
            try {
                successes.add(future.get());
            } catch (ExecutionException e) {
                // Expected for threads that couldn't acquire lock
                assertThat(e.getCause())
                    .isInstanceOf(PaymentInProgressException.class);
            }
        }
        
        // Verify exactly one transaction
        assertThat(transactionRepo.findAll()).hasSize(1);
    }
    
    @Test
    @DisplayName("Different idempotency keys create separate transactions")
    void differentKeysCreateSeparateTransactions() {
        PaymentCommand command = new PaymentCommand(
            BigDecimal.valueOf(100), "SGD", "merchant-1");
        
        PaymentResult first = orchestrator.processPayment(UUID.randomUUID().toString(), command);
        PaymentResult second = orchestrator.processPayment(UUID.randomUUID().toString(), command);
        
        assertThat(first.transactionId()).isNotEqualTo(second.transactionId());
        assertThat(transactionRepo.findAll()).hasSize(2);
    }
}
```

---

## Key Design Decisions

**How long should you store idempotency keys?**
- Payment APIs — 24-48 hours minimum (covers retry windows and support investigation)
- Order creation — 1-7 days (covers UI retry behavior)
- Event deduplication — depends on your retry policy (typically 24 hours)

**Should the response include a "was this a duplicate" indicator?**
- Some APIs return `201 Created` for new and `200 OK` for duplicate. I prefer this approach — it gives clients useful signal without changing the response body.

**What if the client sends a different request body with the same idempotency key?**
- Return `422 Unprocessable Entity` with a clear error. The key is bound to the first request's parameters. Stripe does this, and it's the right call.

**Where should the idempotency key come from?**
- Always client-generated. If the server generates it, you've lost the deduplication guarantee — the whole point is that the client can safely retry with the same key.

---

## Lessons From the Field

After implementing idempotency in payment systems, e-commerce platforms, and event-driven architectures, here's what I've learned:

1. **Start with idempotency from day one** — Retrofitting it is 10x harder than building it in
2. **Layer your defenses** — Redis for speed, database for durability, locks for concurrency
3. **Test with chaos** — Kill processes mid-transaction, simulate network partitions, send bursts of duplicates
4. **Monitor deduplication rates** — A spike in duplicates often signals a client-side bug or network issue
5. **Document your idempotency contract** — Make it crystal clear in your API docs which endpoints are idempotent and how clients should generate keys

Idempotency isn't glamorous. You won't see it on a conference keynote. But it's the pattern that prevents your $4.2 million weekend.
