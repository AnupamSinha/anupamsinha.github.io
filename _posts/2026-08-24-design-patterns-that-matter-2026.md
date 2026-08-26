---
title: "Design Patterns That Actually Matter in 2026 (Skip the Rest)"
date: 2026-08-24
categories: [Java, Fundamentals]
tags: [design-patterns, java, spring-boot, software-engineering, clean-code]
description: "An opinionated filter of the Gang of Four patterns — which ones you use daily in modern Spring Boot, and which ones the framework already handles for you"
mermaid: true
---
Every Java developer has read (or at least skimmed) the Gang of Four book. 23 patterns, each with a UML diagram that looks like it belongs in a PhD thesis. But here's what nobody tells you in 2026: Spring Boot and modern Java have absorbed many of these patterns into the framework itself. You don't implement them — you just use them.

After 17 years of Java development, I've noticed a clear split. About 6-7 patterns come up in my code every week. Another 5-6 are handled by the framework. The rest? I haven't consciously implemented them in years. Let me give you the honest filter based on what I actually see in production codebases.

## Patterns You Actually Implement (Daily Use)

### 1. Strategy Pattern — The Most Useful Pattern in Existence

This is the one pattern I reach for constantly. Any time you have multiple algorithms or behaviors that should be interchangeable, Strategy is your answer.

**Real-world example: Payment processing with multiple providers**

```java
public interface PaymentStrategy {
    PaymentResult process(PaymentRequest request);
    boolean supports(PaymentMethod method);
}

@Component
public class StripePaymentStrategy implements PaymentStrategy {

    private final StripeClient stripeClient;

    public StripePaymentStrategy(StripeClient stripeClient) {
        this.stripeClient = stripeClient;
    }

    @Override
    public PaymentResult process(PaymentRequest request) {
        StripeCharge charge = stripeClient.createCharge(
            request.getAmount(),
            request.getCurrency(),
            request.getCardToken()
        );
        return new PaymentResult(charge.getId(), charge.getStatus());
    }

    @Override
    public boolean supports(PaymentMethod method) {
        return method == PaymentMethod.CREDIT_CARD;
    }
}

@Component
public class PayNowPaymentStrategy implements PaymentStrategy {

    private final PayNowClient payNowClient;

    public PayNowPaymentStrategy(PayNowClient payNowClient) {
        this.payNowClient = payNowClient;
    }

    @Override
    public PaymentResult process(PaymentRequest request) {
        QrPayment payment = payNowClient.initiatePayment(
            request.getAmount(),
            request.getReference()
        );
        return new PaymentResult(payment.getTransactionId(), "PENDING");
    }

    @Override
    public boolean supports(PaymentMethod method) {
        return method == PaymentMethod.PAYNOW;
    }
}

@Component
public class GrabPayPaymentStrategy implements PaymentStrategy {

    private final GrabPayClient grabPayClient;

    public GrabPayPaymentStrategy(GrabPayClient grabPayClient) {
        this.grabPayClient = grabPayClient;
    }

    @Override
    public PaymentResult process(PaymentRequest request) {
        GrabTransaction txn = grabPayClient.charge(
            request.getUserToken(),
            request.getAmount()
        );
        return new PaymentResult(txn.getId(), txn.getStatus());
    }

    @Override
    public boolean supports(PaymentMethod method) {
        return method == PaymentMethod.GRABPAY;
    }
}
```

The resolver that picks the right strategy:

```java
@Component
public class PaymentProcessor {

    private final List<PaymentStrategy> strategies;

    public PaymentProcessor(List<PaymentStrategy> strategies) {
        this.strategies = strategies;
    }

    public PaymentResult processPayment(PaymentRequest request) {
        return strategies.stream()
            .filter(s -> s.supports(request.getPaymentMethod()))
            .findFirst()
            .orElseThrow(() -> new UnsupportedPaymentMethodException(
                request.getPaymentMethod()))
            .process(request);
    }
}
```

Spring Boot injects all implementations of `PaymentStrategy` automatically. Adding a new payment method means adding one class — zero changes to existing code. This is the Open/Closed Principle in action.

### 2. Builder Pattern — For Anything With More Than 3 Fields

Java records handle simple DTOs. But for complex objects with optional fields, validation, and computed properties, Builder is still king.

```java
public class OrderBuilder {

    private String customerId;
    private String currency = "SGD";
    private ShippingAddress shippingAddress;
    private BillingAddress billingAddress;
    private final List<OrderLineItem> items = new ArrayList<>();
    private String couponCode;
    private String notes;
    private DeliveryPreference deliveryPreference = DeliveryPreference.STANDARD;

    public static OrderBuilder forCustomer(String customerId) {
        OrderBuilder builder = new OrderBuilder();
        builder.customerId = customerId;
        return builder;
    }

    public OrderBuilder currency(String currency) {
        this.currency = currency;
        return this;
    }

    public OrderBuilder shippingAddress(ShippingAddress address) {
        this.shippingAddress = address;
        return this;
    }

    public OrderBuilder billingAddress(BillingAddress address) {
        this.billingAddress = address;
        return this;
    }

    public OrderBuilder addItem(String productId, int quantity, BigDecimal price) {
        this.items.add(new OrderLineItem(productId, quantity, price));
        return this;
    }

    public OrderBuilder withCoupon(String couponCode) {
        this.couponCode = couponCode;
        return this;
    }

    public OrderBuilder deliveryPreference(DeliveryPreference preference) {
        this.deliveryPreference = preference;
        return this;
    }

    public OrderBuilder notes(String notes) {
        this.notes = notes;
        return this;
    }

    public Order build() {
        validate();
        BigDecimal subtotal = calculateSubtotal();
        BigDecimal discount = applyCoupon(subtotal);
        BigDecimal shipping = calculateShipping();
        BigDecimal total = subtotal.subtract(discount).add(shipping);

        return new Order(
            UUID.randomUUID().toString(),
            customerId,
            List.copyOf(items),
            shippingAddress,
            billingAddress != null ? billingAddress : BillingAddress.from(shippingAddress),
            currency,
            subtotal,
            discount,
            shipping,
            total,
            deliveryPreference,
            notes,
            Instant.now()
        );
    }

    private void validate() {
        if (customerId == null || customerId.isBlank()) {
            throw new IllegalStateException("Customer ID is required");
        }
        if (items.isEmpty()) {
            throw new IllegalStateException("Order must have at least one item");
        }
        if (shippingAddress == null) {
            throw new IllegalStateException("Shipping address is required");
        }
    }

    private BigDecimal calculateSubtotal() {
        return items.stream()
            .map(item -> item.price().multiply(BigDecimal.valueOf(item.quantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    private BigDecimal applyCoupon(BigDecimal subtotal) {
        // Coupon logic here
        return BigDecimal.ZERO;
    }

    private BigDecimal calculateShipping() {
        return deliveryPreference == DeliveryPreference.EXPRESS 
            ? new BigDecimal("15.00") 
            : new BigDecimal("5.00");
    }
}
```

Usage:

```java
Order order = OrderBuilder.forCustomer("cust-123")
    .addItem("prod-1", 2, new BigDecimal("29.99"))
    .addItem("prod-2", 1, new BigDecimal("49.99"))
    .shippingAddress(address)
    .withCoupon("SAVE10")
    .deliveryPreference(DeliveryPreference.EXPRESS)
    .build();
```

Note: I don't use Lombok's `@Builder` for domain objects. It skips validation. For DTOs and test data? Lombok is fine.

### 3. Observer/Event Pattern — Spring's ApplicationEvent

This is how you decouple components without adding a message broker. Spring's event system is Observer pattern built into the framework.

```java
// The event
public record OrderCompletedEvent(
    String orderId,
    String customerId,
    BigDecimal totalAmount,
    Instant completedAt
) {}

// Publisher — in your service
@Service
public class OrderService {

    private final ApplicationEventPublisher eventPublisher;
    private final OrderRepository repository;

    public OrderService(ApplicationEventPublisher eventPublisher, 
                       OrderRepository repository) {
        this.eventPublisher = eventPublisher;
        this.repository = repository;
    }

    @Transactional
    public Order completeOrder(String orderId) {
        Order order = repository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        order.markCompleted();
        Order saved = repository.save(order);

        eventPublisher.publishEvent(new OrderCompletedEvent(
            saved.getId(),
            saved.getCustomerId(),
            saved.getTotalAmount(),
            Instant.now()
        ));

        return saved;
    }
}

// Listeners — completely decoupled
@Component
public class OrderNotificationListener {

    private final EmailService emailService;

    public OrderNotificationListener(EmailService emailService) {
        this.emailService = emailService;
    }

    @EventListener
    public void onOrderCompleted(OrderCompletedEvent event) {
        emailService.sendOrderConfirmation(event.customerId(), event.orderId());
    }
}

@Component
public class OrderAnalyticsListener {

    private final AnalyticsService analyticsService;

    public OrderAnalyticsListener(AnalyticsService analyticsService) {
        this.analyticsService = analyticsService;
    }

    @Async
    @EventListener
    public void onOrderCompleted(OrderCompletedEvent event) {
        analyticsService.trackPurchase(event.customerId(), event.totalAmount());
    }
}

@Component
public class LoyaltyPointsListener {

    private final LoyaltyService loyaltyService;

    public LoyaltyPointsListener(LoyaltyService loyaltyService) {
        this.loyaltyService = loyaltyService;
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderCompleted(OrderCompletedEvent event) {
        int points = event.totalAmount().intValue();
        loyaltyService.awardPoints(event.customerId(), points);
    }
}
```

Key decisions here:

- Use `@EventListener` for synchronous in-process events
- Use `@Async @EventListener` for fire-and-forget async processing
- Use `@TransactionalEventListener` when the listener should only execute after the transaction commits (prevents awarding loyalty points for a rolled-back order)

### 4. Template Method — For Standardized Workflows

When multiple processes share the same structure but differ in specific steps:

```java
public abstract class DataImportTemplate<T> {

    private static final Logger log = LoggerFactory.getLogger(DataImportTemplate.class);

    @Transactional
    public ImportResult execute(InputStream source) {
        log.info("Starting import: {}", getImportName());
        
        List<T> rawRecords = parse(source);
        log.info("Parsed {} records", rawRecords.size());

        List<T> validRecords = rawRecords.stream()
            .filter(this::validate)
            .toList();
        log.info("Validated {} records ({} rejected)", 
            validRecords.size(), rawRecords.size() - validRecords.size());

        List<T> transformed = validRecords.stream()
            .map(this::transform)
            .toList();

        int saved = persist(transformed);
        log.info("Persisted {} records", saved);

        postProcess(transformed);

        return new ImportResult(rawRecords.size(), validRecords.size(), saved);
    }

    protected abstract String getImportName();
    protected abstract List<T> parse(InputStream source);
    protected abstract boolean validate(T record);
    protected abstract T transform(T record);
    protected abstract int persist(List<T> records);

    // Hook method — override if needed
    protected void postProcess(List<T> records) {
        // Default: do nothing
    }
}

@Component
public class CustomerCsvImport extends DataImportTemplate<CustomerRecord> {

    private final CustomerRepository repository;

    public CustomerCsvImport(CustomerRepository repository) {
        this.repository = repository;
    }

    @Override
    protected String getImportName() {
        return "Customer CSV Import";
    }

    @Override
    protected List<CustomerRecord> parse(InputStream source) {
        // CSV parsing logic
        return CsvParser.parse(source, CustomerRecord.class);
    }

    @Override
    protected boolean validate(CustomerRecord record) {
        return record.email() != null 
            && record.email().contains("@")
            && record.name() != null 
            && !record.name().isBlank();
    }

    @Override
    protected CustomerRecord transform(CustomerRecord record) {
        return new CustomerRecord(
            record.name().trim(),
            record.email().toLowerCase().trim(),
            record.phone()
        );
    }

    @Override
    protected int persist(List<CustomerRecord> records) {
        List<Customer> entities = records.stream()
            .map(Customer::fromRecord)
            .toList();
        return repository.saveAll(entities).size();
    }
}
```

### 5. Decorator Pattern — For Cross-Cutting Concerns

Decorators wrap existing behavior with additional functionality. In Spring Boot, you see this with caching, retry, and logging decorators:

```java
public interface PricingService {
    BigDecimal calculatePrice(String productId, String customerId);
}

@Component
@Primary
public class CachedPricingService implements PricingService {

    private final PricingService delegate;
    private final Cache<String, BigDecimal> priceCache;

    public CachedPricingService(
            @Qualifier("corePricingService") PricingService delegate) {
        this.delegate = delegate;
        this.priceCache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .build();
    }

    @Override
    public BigDecimal calculatePrice(String productId, String customerId) {
        String cacheKey = productId + ":" + customerId;
        return priceCache.get(cacheKey, 
            key -> delegate.calculatePrice(productId, customerId));
    }
}

@Component("corePricingService")
public class CorePricingService implements PricingService {

    private final ProductRepository productRepository;
    private final DiscountEngine discountEngine;
    private final TaxCalculator taxCalculator;

    public CorePricingService(ProductRepository productRepository,
                              DiscountEngine discountEngine,
                              TaxCalculator taxCalculator) {
        this.productRepository = productRepository;
        this.discountEngine = discountEngine;
        this.taxCalculator = taxCalculator;
    }

    @Override
    public BigDecimal calculatePrice(String productId, String customerId) {
        Product product = productRepository.findById(productId)
            .orElseThrow();
        
        BigDecimal basePrice = product.getBasePrice();
        BigDecimal discount = discountEngine.calculate(customerId, product);
        BigDecimal afterDiscount = basePrice.subtract(discount);
        BigDecimal tax = taxCalculator.calculate(afterDiscount, product.getCategory());
        
        return afterDiscount.add(tax);
    }
}
```

The `@Primary` annotation ensures that when other components autowire `PricingService`, they get the cached version. The core implementation is only injected explicitly via `@Qualifier`.

## Patterns the Framework Handles (Don't Implement Yourself)

### Singleton — Spring Does This

Every `@Component`, `@Service`, `@Repository` in Spring is a singleton by default. You don't write singleton boilerplate anymore. Period.

```java
// DON'T DO THIS in 2026
public class ConnectionPool {
    private static ConnectionPool instance;
    private ConnectionPool() {}
    public static synchronized ConnectionPool getInstance() { ... }
}

// DO THIS — Spring manages the lifecycle
@Component
public class ConnectionPool {
    // Automatically singleton-scoped
}
```

### Factory Method — Use Spring's ObjectProvider

When you need to create instances dynamically, use `ObjectProvider` or prototype scope instead of writing your own factory:

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class ReportGenerator {
    private final String reportType;
    
    public ReportGenerator(@Value("#{null}") String reportType) {
        this.reportType = reportType;
    }
}

@Service
public class ReportService {
    
    private final ObjectProvider<ReportGenerator> generatorProvider;
    
    public ReportService(ObjectProvider<ReportGenerator> generatorProvider) {
        this.generatorProvider = generatorProvider;
    }
    
    public Report generate(String type) {
        ReportGenerator generator = generatorProvider.getObject(type);
        return generator.generate();
    }
}
```

### Proxy Pattern — Spring AOP

`@Transactional`, `@Cacheable`, `@Async`, `@Retryable` — these are all proxy-based decorators. Spring creates proxy objects at runtime. You don't implement Proxy pattern; you use annotations.

### Chain of Responsibility — Spring Security Filter Chain

The entire security filter chain is Chain of Responsibility. You add filters, they execute in order. You never implement the chain yourself.

## Patterns That Are Mostly Irrelevant

### Abstract Factory

In 17 years, I've implemented Abstract Factory exactly twice. Both times, I later refactored it to Strategy + simple factory method because Abstract Factory is almost always over-engineering for what's actually needed.

### Visitor

Unless you're building a compiler or AST processor, you'll never need this. It's the most academic pattern in the GoF book.

### Memento

State management in modern apps is handled by event sourcing, database snapshots, or frontend state libraries. Memento as described in the GoF book doesn't map to how we build software today.

### Flyweight

The JVM handles string interning and object pooling. Connection pools exist as libraries. You don't implement Flyweight manually in 2026.

### Mediator

In the GoF sense, Mediator is largely replaced by event buses, Spring Events, or message brokers. The pattern exists, but the implementation is always a framework feature, not custom code.

## My Rule of Thumb

Before implementing any pattern, ask:

1. **Does Spring already provide this?** — Check for annotations and framework features first.
2. **Is this solving a real problem or a hypothetical one?** — Don't add Strategy for two variants. A simple `if` statement is fine until there are three or more.
3. **Will the next developer understand this?** — Patterns should clarify intent, not obscure it. If you need a comment explaining which pattern this is, you might be forcing it.

The best code I've seen in production uses maybe 5-6 patterns consistently and uses them well. The worst code tries to use all 23 "because we might need it someday."

## Final Thoughts

Design patterns aren't a checklist. They're a vocabulary. Knowing all 23 means you can recognize them when reading code. But writing code? You'll reach for Strategy, Builder, Observer, Template Method, and Decorator. That covers 90% of what you need.

The framework handles Singleton, Factory, Proxy, and Chain of Responsibility. And the rest? They're either absorbed into libraries, replaced by modern language features, or solutions to problems that modern architectures simply don't have.

Focus on the patterns that solve your actual problems. Skip the rest.
