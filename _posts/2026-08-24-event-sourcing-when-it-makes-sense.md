---
title: "Event Sourcing: When It Makes Sense and When It'll Ruin Your Project"
date: 2026-08-24
categories: [Spring Boot, Architecture]
tags: [event-sourcing, architecture, spring-boot, cqrs, distributed-systems]
description: "An honest evaluation of event sourcing — when it shines, when it's overkill, the hidden costs nobody warns you about, and a decision framework to help you choose."
mermaid: true
---
## My Event Sourcing Journey

I've implemented event sourcing in three production systems over my career. One was a massive success (trade settlement platform). One was borderline (e-commerce order management). One was a regrettable disaster (internal HR tool). The difference wasn't the technology — it was whether the problem actually needed event sourcing.

After 17 years building Java systems, I've developed strong opinions about when to use event sourcing and when it will actively harm your project. This article is the guide I wish someone had given me before that HR tool project.

## What Event Sourcing Actually Is

Instead of storing the current state of an entity, you store the sequence of events that led to that state. The current state is derived by replaying events.

Traditional approach:

```
Account Table: { id: 123, balance: 750, status: ACTIVE }
```

Event-sourced approach:

```
Event Store:
1. AccountCreated { id: 123, owner: "John" }
2. MoneyDeposited { id: 123, amount: 1000 }
3. MoneyWithdrawn { id: 123, amount: 200 }
4. MoneyWithdrawn { id: 123, amount: 50 }
```

Current balance? Replay: 0 + 1000 - 200 - 50 = 750.

### Basic Implementation in Spring Boot

```java
// The Event
public sealed interface AccountEvent {
    String accountId();
    Instant occurredAt();

    record AccountCreated(String accountId, String owner, Instant occurredAt)
        implements AccountEvent {}

    record MoneyDeposited(String accountId, BigDecimal amount, Instant occurredAt)
        implements AccountEvent {}

    record MoneyWithdrawn(String accountId, BigDecimal amount, Instant occurredAt)
        implements AccountEvent {}

    record AccountClosed(String accountId, String reason, Instant occurredAt)
        implements AccountEvent {}
}
```

```java
// The Aggregate — rebuilds state from events
public class AccountAggregate {

    private String accountId;
    private String owner;
    private BigDecimal balance = BigDecimal.ZERO;
    private AccountStatus status = AccountStatus.NEW;
    private List<AccountEvent> uncommittedEvents = new ArrayList<>();

    // Rebuild from history
    public static AccountAggregate fromHistory(List<AccountEvent> events) {
        var aggregate = new AccountAggregate();
        events.forEach(aggregate::apply);
        return aggregate;
    }

    // Command handler — validates business rules, produces events
    public void deposit(BigDecimal amount) {
        if (status != AccountStatus.ACTIVE) {
            throw new IllegalStateException("Cannot deposit to " + status + " account");
        }
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }

        raiseEvent(new MoneyDeposited(accountId, amount, Instant.now()));
    }

    public void withdraw(BigDecimal amount) {
        if (status != AccountStatus.ACTIVE) {
            throw new IllegalStateException("Cannot withdraw from " + status + " account");
        }
        if (balance.compareTo(amount) < 0) {
            throw new InsufficientFundsException(accountId, balance, amount);
        }

        raiseEvent(new MoneyWithdrawn(accountId, amount, Instant.now()));
    }

    // Event handlers — pure state transitions, no validation
    private void apply(AccountEvent event) {
        switch (event) {
            case AccountCreated e -> {
                this.accountId = e.accountId();
                this.owner = e.owner();
                this.status = AccountStatus.ACTIVE;
            }
            case MoneyDeposited e -> {
                this.balance = this.balance.add(e.amount());
            }
            case MoneyWithdrawn e -> {
                this.balance = this.balance.subtract(e.amount());
            }
            case AccountClosed e -> {
                this.status = AccountStatus.CLOSED;
            }
        }
    }

    private void raiseEvent(AccountEvent event) {
        apply(event);
        uncommittedEvents.add(event);
    }

    public List<AccountEvent> getUncommittedEvents() {
        return Collections.unmodifiableList(uncommittedEvents);
    }
}
```

```java
// The Event Store
@Repository
public class JdbcEventStore implements EventStore {

    private final JdbcTemplate jdbc;
    private final ObjectMapper objectMapper;

    @Override
    @Transactional
    public void save(String aggregateId, List<AccountEvent> events, long expectedVersion) {
        long currentVersion = getCurrentVersion(aggregateId);

        if (currentVersion != expectedVersion) {
            throw new OptimisticConcurrencyException(
                "Expected version " + expectedVersion + " but found " + currentVersion);
        }

        long version = expectedVersion;
        for (AccountEvent event : events) {
            version++;
            jdbc.update("""
                INSERT INTO event_store (aggregate_id, version, event_type, payload, occurred_at)
                VALUES (?, ?, ?, ?::jsonb, ?)
                """,
                aggregateId,
                version,
                event.getClass().getSimpleName(),
                objectMapper.writeValueAsString(event),
                event.occurredAt()
            );
        }
    }

    @Override
    public List<AccountEvent> load(String aggregateId) {
        return jdbc.query("""
            SELECT event_type, payload FROM event_store
            WHERE aggregate_id = ?
            ORDER BY version ASC
            """,
            (rs, row) -> deserialize(rs.getString("event_type"), rs.getString("payload")),
            aggregateId
        );
    }
}
```

## When Event Sourcing Shines

### 1. Audit Trail is a Business Requirement

If your domain requires a complete, tamper-proof audit trail — financial services, healthcare, legal — event sourcing gives you this for free. Every state change is recorded with who did it, when, and why.

Our trade settlement platform needed to answer: "Why was this trade settled at $X?" With event sourcing, we could replay the exact sequence of events (price quotes, margin calculations, fee applications) that produced the final number. Regulators loved it.

### 2. Temporal Queries

"What was the account balance at 3:47 PM on March 15th?" With traditional state storage, you'd need expensive audit tables or change data capture. With event sourcing, you replay events up to that timestamp:

```java
public BigDecimal getBalanceAt(String accountId, Instant timestamp) {
    List<AccountEvent> events = eventStore.loadUntil(accountId, timestamp);
    AccountAggregate aggregate = AccountAggregate.fromHistory(events);
    return aggregate.getBalance();
}
```

### 3. Event Replay for Bug Fixes

Found a calculation bug in production? Fix the code, replay events from the beginning, and get the correct state. We did this twice on the settlement platform — recalculated 3 months of settlement amounts after finding a rounding error, without losing any business data.

### 4. Multiple Read Models from Same Events

Different consumers need different views of the same data:

```java
// Read model 1: Account balance (for the mobile app)
@Component
public class BalanceProjection {

    @EventHandler
    public void on(MoneyDeposited event) {
        balanceRepository.addToBalance(event.accountId(), event.amount());
    }

    @EventHandler
    public void on(MoneyWithdrawn event) {
        balanceRepository.subtractFromBalance(event.accountId(), event.amount());
    }
}

// Read model 2: Transaction history (for statements)
@Component
public class TransactionHistoryProjection {

    @EventHandler
    public void on(MoneyDeposited event) {
        transactionRepository.save(new Transaction(
            event.accountId(), "DEPOSIT", event.amount(), event.occurredAt()));
    }
}

// Read model 3: Analytics (for the dashboard)
@Component
public class AnalyticsProjection {

    @EventHandler
    public void on(MoneyDeposited event) {
        analyticsRepository.incrementDailyDeposits(event.occurredAt().toLocalDate());
        analyticsRepository.addToDailyVolume(event.occurredAt().toLocalDate(), event.amount());
    }
}
```

### 5. Complex Domain with Business Rules that Evolve

When your domain logic changes frequently, event sourcing lets you add new projections without migrating old data. The events are immutable facts — only the interpretation changes.

## When Event Sourcing Will Ruin Your Project

### 1. Simple CRUD Applications

If your domain is "user submits form, data gets saved, user reads data back" — event sourcing adds enormous complexity for zero benefit. An HR tool for tracking employee leave requests does not need events. It needs a `leave_requests` table.

Our HR tool disaster: we event-sourced leave requests. The "events" were: `LeaveRequested`, `LeaveApproved`, `LeaveDenied`. The read model was exactly the same as what a simple table would have been. We spent 3x the development time for no business value.

### 2. Small Teams (Under 4 Developers)

Event sourcing requires maintaining:
- Event store
- Projections (read models)
- Snapshot mechanism
- Event versioning strategy
- Idempotent event handlers
- Eventual consistency handling in the UI

That's significant cognitive overhead. A team of 2-3 developers will spend more time fighting the architecture than building features.

### 3. Tight Deadlines

If you need to ship in 6 weeks, don't introduce event sourcing for the first time. The learning curve is steep, and the gotchas (event versioning, eventual consistency, snapshot management) will eat your schedule.

### 4. When Strong Consistency is Required Everywhere

Event sourcing naturally leads to eventual consistency between the write model and read models. If your business cannot tolerate even milliseconds of stale data across views, event sourcing creates constant friction.

## The Hidden Costs Nobody Warns You About

### Cost 1: Eventual Consistency in the UI

User deposits money. Your command succeeds. The event is published. The projection updates. But there's a lag — maybe 50ms, maybe 500ms under load. The user refreshes and doesn't see their deposit.

```java
// The workaround: "write-through" read after write
@Service
public class AccountQueryService {

    private final EventStore eventStore;
    private final BalanceReadRepository readRepo;

    public AccountBalance getBalance(String accountId, Long afterVersion) {
        AccountBalance cached = readRepo.findById(accountId);

        if (cached != null && cached.getVersion() >= afterVersion) {
            return cached; // Projection is caught up
        }

        // Projection is behind — rebuild from events
        List<AccountEvent> events = eventStore.load(accountId);
        return AccountAggregate.fromHistory(events).toBalance();
    }
}
```

This "simple" workaround adds complexity to every read path.

### Cost 2: Snapshot Management

An account with 100,000 events takes seconds to rebuild from scratch. You need snapshots:

```java
@Service
public class SnapshotService {

    private final SnapshotRepository snapshotRepository;
    private final EventStore eventStore;

    public AccountAggregate loadAggregate(String accountId) {
        // Load latest snapshot
        Optional<Snapshot> snapshot = snapshotRepository.findLatest(accountId);

        AccountAggregate aggregate;
        long fromVersion;

        if (snapshot.isPresent()) {
            aggregate = deserialize(snapshot.get().getState());
            fromVersion = snapshot.get().getVersion();
        } else {
            aggregate = new AccountAggregate();
            fromVersion = 0;
        }

        // Replay only events after snapshot
        List<AccountEvent> recentEvents = eventStore.loadAfterVersion(accountId, fromVersion);
        recentEvents.forEach(aggregate::apply);

        // Create new snapshot if too many events since last one
        if (recentEvents.size() > 100) {
            snapshotRepository.save(new Snapshot(
                accountId,
                aggregate.getVersion(),
                serialize(aggregate)
            ));
        }

        return aggregate;
    }
}
```

Now you need a strategy for when to snapshot, how to handle snapshot corruption, and how to rebuild snapshots when your aggregate structure changes.

### Cost 3: Event Versioning

Your events are immutable. But your domain evolves. Version 1 of `MoneyDeposited` didn't include `currency`. Version 2 does. You need upcasters:

```java
@Component
public class EventUpcasterChain {

    private final List<EventUpcaster> upcasters = List.of(
        new MoneyDepositedV1ToV2Upcaster(),
        new AccountCreatedV1ToV2Upcaster()
    );

    public AccountEvent upcast(String eventType, int version, JsonNode payload) {
        JsonNode current = payload;
        int currentVersion = version;

        for (EventUpcaster upcaster : upcasters) {
            if (upcaster.canUpcast(eventType, currentVersion)) {
                current = upcaster.upcast(current);
                currentVersion++;
            }
        }

        return deserialize(eventType, currentVersion, current);
    }
}

public class MoneyDepositedV1ToV2Upcaster implements EventUpcaster {

    @Override
    public boolean canUpcast(String eventType, int version) {
        return "MoneyDeposited".equals(eventType) && version == 1;
    }

    @Override
    public JsonNode upcast(JsonNode v1) {
        ObjectNode v2 = (ObjectNode) v1.deepCopy();
        v2.put("currency", "SGD"); // Default for existing events
        v2.put("schemaVersion", 2);
        return v2;
    }
}
```

Every schema change requires an upcaster. Over years, you accumulate dozens. Testing all permutations becomes a burden.

### Cost 4: Debugging Complexity

When something goes wrong in a traditional system, you look at the row in the database. With event sourcing:

1. Find the events for the aggregate
2. Replay them to find which event caused the bad state
3. Check if the projection is behind
4. Check if an upcaster introduced a bug
5. Check if an event was published but the handler failed
6. Check if idempotency logic incorrectly skipped an event

Debugging time increases 3-5x in my experience.

### Cost 5: Storage Growth

Every state change creates a new event. A high-volume system generates billions of events. You need:
- Event archiving strategy
- Partition/shard strategy for the event store
- Compaction or cleanup policies (if legally permitted)

## The Decision Framework

I use this framework to decide whether event sourcing is appropriate:

### Score Each Factor (0-3)

**Audit requirement** — 0: no audit needed, 3: regulatory compliance requires full history

**Temporal queries** — 0: never need point-in-time data, 3: frequently need historical state

**Multiple read models** — 0: one view of data, 3: 5+ different consumers need different views

**Event replay value** — 0: no replay scenarios, 3: replay is core to operations

**Domain complexity** — 0: simple CRUD, 3: complex state machines with many transitions

**Team size** — 0: 1-2 developers, 1: 3-5, 2: 6-10, 3: 10+

**Timeline** — 0: shipping in weeks, 1: months, 2: quarters, 3: long-term investment

### Scoring Guide

**Score 0-7** — Don't use event sourcing. Use a traditional approach with an audit log table if you need history.

**Score 8-13** — Consider event sourcing for specific subdomains, not the whole system. Hybrid approach.

**Score 14-21** — Event sourcing is likely a good fit. Invest in the infrastructure.

### Our Three Projects Scored

**Trade Settlement Platform** — Audit: 3, Temporal: 3, Read Models: 3, Replay: 3, Complexity: 3, Team: 3, Timeline: 3. **Score: 21.** Excellent fit.

**E-commerce Orders** — Audit: 2, Temporal: 1, Read Models: 2, Replay: 1, Complexity: 2, Team: 2, Timeline: 2. **Score: 12.** Borderline — we'd probably use a simpler approach if starting over.

**HR Leave Tool** — Audit: 1, Temporal: 0, Read Models: 0, Replay: 0, Complexity: 1, Team: 1, Timeline: 0. **Score: 3.** Should never have been event-sourced.

## The Hybrid Approach: Best of Both Worlds

For most teams, I recommend a hybrid: traditional state storage with an event log for the domains that need it.

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;      // Primary state (traditional)
    private final OrderEventLog orderEventLog;          // Event log (for audit/replay)

    @Transactional
    public Order updateOrderStatus(String orderId, OrderStatus newStatus, String reason) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        OrderStatus previousStatus = order.getStatus();
        order.setStatus(newStatus);
        order.setUpdatedAt(Instant.now());
        orderRepository.save(order);

        // Log the event for audit — but it's not the source of truth
        orderEventLog.append(new OrderStatusChanged(
            orderId, previousStatus, newStatus, reason, Instant.now()
        ));

        return order;
    }
}
```

You get:
- Simple reads (query the table directly)
- Full audit trail (event log)
- Temporal queries (replay event log when needed)
- No eventual consistency headaches for the primary read path

You lose:
- Ability to derive state purely from events
- Multiple independent read models
- Event replay to fix bugs

For 80% of projects, this trade-off is correct.

## If You Do Choose Event Sourcing

Use a proven framework. Don't build your own event store (I made that mistake):

- **Axon Framework** — Full-featured, Spring Boot integration, good for teams new to ES
- **EventStoreDB** — Purpose-built event store, great performance, built-in projections
- **Marten** — If you're using PostgreSQL (it uses JSONB for event storage)

Invest in:
- Comprehensive integration tests that replay events through full lifecycle
- Event schema registry with forward/backward compatibility rules
- Monitoring for projection lag
- Tooling to inspect and replay events in production

## Final Thoughts

Event sourcing is a powerful pattern for the right problem. But "powerful" doesn't mean "universally applicable." I've seen teams adopt it because it sounds architecturally elegant, then spend months fighting its complexity for a domain that needed a simple database.

Be honest about your requirements. Score them. If the number says no, trust it. A well-designed traditional system beats a poorly-implemented event-sourced system every single time.
