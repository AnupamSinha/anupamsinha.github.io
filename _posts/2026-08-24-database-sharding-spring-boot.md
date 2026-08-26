---
title: "Database Sharding with Spring Boot — When One Database Isn't Enough"
date: 2026-08-24
categories: [Spring Boot, Data]
tags: [database, spring-boot, sharding, scalability, system-design]
description: "A practical guide to implementing database sharding in Spring Boot — covering when to shard, which strategy to pick, building a shard router with AbstractRoutingDataSource, and the simpler alternatives you should try first."
mermaid: true
---
## The Moment You Realize One Database Won't Cut It

I've been building enterprise systems in Singapore for 17 years, and I'll tell you this — most teams shard too early. But when you genuinely need it, nothing else will save you.

Here's the pattern I've seen repeatedly: your PostgreSQL instance is handling 500 million rows in a single table. Queries that used to take 20ms now take 2 seconds. You've already added indexes, optimized queries, thrown more RAM at the box. Your DBA is sending you passive-aggressive Slack messages about vacuum times.

That's when sharding enters the conversation.

But before we jump into implementation, let me be clear about when you actually need this — and what to try first.

## Try These Before You Shard

Sharding adds operational complexity that will haunt your team for years. Before you go down that path, exhaust these options:

**Read Replicas** — If your problem is read throughput, add replicas. Spring Boot makes this trivial with `@Transactional(readOnly = true)` routing to a read replica. This handles 80% of scaling problems I've encountered.

**Table Partitioning** — PostgreSQL native partitioning (range or hash) gives you many benefits of sharding without the distributed complexity. Your application code doesn't change at all.

**Vertical Partitioning** — Move large blob columns or audit trails to separate tables. Sometimes a 2GB row width is your actual problem.

**Connection Pool Tuning** — I've seen teams blame "database scaling limits" when their HikariCP pool was configured with 10 connections for 200 concurrent requests.

**Archival** — Do you really need 5 years of transaction history in your hot table? Move old data to cold storage.

If you've genuinely tried all of these and you're still hitting limits — welcome to the world of sharding.

## When to Shard: The Decision Framework

Here's my rule of thumb after doing this across multiple fintech and e-commerce platforms:

**Shard when ALL of these are true** —

- Single table exceeds 500M–1B rows
- Write throughput exceeds what a single primary can handle
- You've exhausted vertical scaling (largest available instance)
- Table partitioning doesn't solve the query patterns
- Your data has a natural partition key (tenant_id, region, user_id)

If your data doesn't have a natural partition key, think very hard before sharding. Cross-shard queries will become your new nightmare.

## Sharding Strategies

### 1. Range-Based Sharding

Partition data by ranges of a key value.

**Example** — Users 1–1,000,000 go to Shard A, 1,000,001–2,000,000 to Shard B.

**Pros** — Simple to understand, range queries within a shard are efficient

**Cons** — Hot spots (new users always hit the latest shard), uneven distribution over time, rebalancing is painful

**Best for** — Time-series data, sequential IDs with even access patterns

### 2. Hash-Based Sharding

Apply a hash function to the partition key and use modulo to determine the shard.

**Example** — `shard = hash(user_id) % number_of_shards`

**Pros** — Even distribution, no hot spots, predictable routing

**Cons** — Range queries require scatter-gather across all shards, adding shards requires rehashing (unless you use consistent hashing)

**Best for** — User data, transactional data with point lookups

### 3. Directory-Based Sharding

Maintain a lookup table that maps each entity to its shard.

**Example** — A `shard_mapping` table stores `tenant_id → shard_identifier`

**Pros** — Maximum flexibility, easy rebalancing (just update the directory), supports irregular distributions

**Cons** — The directory itself becomes a single point of failure and a potential bottleneck, extra lookup on every request

**Best for** — Multi-tenant SaaS applications where tenants vary wildly in size

## Implementing a Shard Router in Spring Boot

Here's the core implementation using Spring's `AbstractRoutingDataSource`. This is production-tested code I've used across multiple projects.

### Step 1: Define the Shard Context

```java
public class ShardContext {

    private static final ThreadLocal<String> currentShard = new ThreadLocal<>();

    public static void setShard(String shardId) {
        currentShard.set(shardId);
    }

    public static String getShard() {
        return currentShard.get();
    }

    public static void clear() {
        currentShard.remove();
    }
}
```

### Step 2: Create the Routing DataSource

```java
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

public class ShardRoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        String shard = ShardContext.getShard();
        if (shard == null) {
            return "default"; // fallback to primary shard
        }
        return shard;
    }
}
```

### Step 3: Configure Multiple DataSources

```java
@Configuration
public class ShardDataSourceConfig {

    @Bean
    public DataSource dataSource() {
        ShardRoutingDataSource routingDataSource = new ShardRoutingDataSource();

        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("shard-0", createDataSource("jdbc:postgresql://shard0-host:5432/orders"));
        targetDataSources.put("shard-1", createDataSource("jdbc:postgresql://shard1-host:5432/orders"));
        targetDataSources.put("shard-2", createDataSource("jdbc:postgresql://shard2-host:5432/orders"));

        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(targetDataSources.get("shard-0"));
        routingDataSource.afterPropertiesSet();

        return routingDataSource;
    }

    private DataSource createDataSource(String url) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername("app_user");
        config.setPassword("${SHARD_DB_PASSWORD}");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(3000);
        config.setIdleTimeout(600000);
        return new HikariDataSource(config);
    }
}
```

### Step 4: Build the Shard Resolution Strategy

```java
@Component
public class HashBasedShardResolver {

    private static final int TOTAL_SHARDS = 3;

    public String resolveShard(Long userId) {
        int shardIndex = Math.abs(Long.hashCode(userId) % TOTAL_SHARDS);
        return "shard-" + shardIndex;
    }
}
```

### Step 5: Create an Interceptor for Automatic Routing

```java
@Aspect
@Component
public class ShardRoutingAspect {

    private final HashBasedShardResolver shardResolver;

    public ShardRoutingAspect(HashBasedShardResolver shardResolver) {
        this.shardResolver = shardResolver;
    }

    @Around("@annotation(sharded)")
    public Object routeToShard(ProceedingJoinPoint joinPoint, Sharded sharded) throws Throwable {
        Object[] args = joinPoint.getArgs();
        Long userId = extractUserId(args, sharded.paramIndex());

        String shard = shardResolver.resolveShard(userId);
        ShardContext.setShard(shard);

        try {
            return joinPoint.proceed();
        } finally {
            ShardContext.clear();
        }
    }

    private Long extractUserId(Object[] args, int paramIndex) {
        return (Long) args[paramIndex];
    }
}
```

### Step 6: Use the Custom Annotation

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Sharded {
    int paramIndex() default 0;
}
```

### Step 7: Apply It in Your Service Layer

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Sharded(paramIndex = 0)
    @Transactional
    public Order createOrder(Long userId, OrderRequest request) {
        Order order = Order.builder()
                .userId(userId)
                .items(request.getItems())
                .totalAmount(request.calculateTotal())
                .status(OrderStatus.CREATED)
                .build();
        return orderRepository.save(order);
    }

    @Sharded(paramIndex = 0)
    @Transactional(readOnly = true)
    public List<Order> getOrdersByUser(Long userId) {
        return orderRepository.findByUserId(userId);
    }
}
```

## Handling Cross-Shard Queries

This is where sharding gets painful. When a query can't be routed to a single shard, you have three options:

### Option 1: Scatter-Gather Pattern

Query all shards in parallel and merge results. Use this for admin dashboards or reports.

```java
@Service
public class CrossShardQueryService {

    private final List<DataSource> allShards;
    private final ExecutorService executor;

    public CrossShardQueryService(List<DataSource> allShards) {
        this.allShards = allShards;
        this.executor = Executors.newFixedThreadPool(allShards.size());
    }

    public List<Order> findOrdersAcrossShards(OrderSearchCriteria criteria) {
        List<CompletableFuture<List<Order>>> futures = allShards.stream()
                .map(shard -> CompletableFuture.supplyAsync(
                        () -> queryOnShard(shard, criteria), executor))
                .collect(Collectors.toList());

        return futures.stream()
                .map(CompletableFuture::join)
                .flatMap(Collection::stream)
                .sorted(Comparator.comparing(Order::getCreatedAt).reversed())
                .limit(criteria.getLimit())
                .collect(Collectors.toList());
    }

    private List<Order> queryOnShard(DataSource shard, OrderSearchCriteria criteria) {
        JdbcTemplate jdbc = new JdbcTemplate(shard);
        return jdbc.query(
                "SELECT * FROM orders WHERE status = ? AND created_at > ? LIMIT ?",
                new OrderRowMapper(),
                criteria.getStatus().name(),
                criteria.getFromDate(),
                criteria.getLimit()
        );
    }
}
```

### Option 2: Maintain a Global Index

Keep a lightweight index table in a non-sharded database that maps entities to shards.

```java
@Entity
@Table(name = "order_shard_index")
public class OrderShardIndex {

    @Id
    private String orderId;
    private Long userId;
    private String shardId;
    private LocalDateTime createdAt;
    private String status;
}
```

This lets you find which shard an order lives on without querying all shards.

### Option 3: Denormalize into a Read-Optimized Store

For complex cross-shard analytics, stream shard changes into Elasticsearch or a data warehouse. Don't fight the sharding model — work around it.

## Rebalancing: The Hard Problem

When you need to add shards or redistribute data:

**Consistent Hashing** — Use a hash ring instead of simple modulo. When you add a shard, only ~1/N of the data needs to move.

```java
public class ConsistentHashShardResolver {

    private final TreeMap<Long, String> ring = new TreeMap<>();
    private static final int VIRTUAL_NODES = 150;

    public void addShard(String shardId) {
        for (int i = 0; i < VIRTUAL_NODES; i++) {
            long hash = hash(shardId + "-" + i);
            ring.put(hash, shardId);
        }
    }

    public String resolveShard(Long userId) {
        long hash = hash(String.valueOf(userId));
        Map.Entry<Long, String> entry = ring.ceilingEntry(hash);
        if (entry == null) {
            entry = ring.firstEntry();
        }
        return entry.getValue();
    }

    private long hash(String key) {
        return Hashing.murmur3_128().hashString(key, StandardCharsets.UTF_8).asLong();
    }
}
```

**Double-Write Migration** — During rebalancing:

1. Start writing to both old and new shard locations
2. Background job copies existing data to new locations
3. Verify data consistency
4. Switch reads to new locations
5. Stop writing to old locations
6. Clean up old data

This is non-trivial. Budget 2–4 weeks for a safe rebalancing operation on a production system.

## Common Pitfalls I've Seen

**Choosing the wrong shard key** — I once saw a team shard by `country_code`. Singapore had 80% of the data. That's not sharding, that's just a slower single database with extra steps.

**Auto-increment IDs across shards** — Use UUIDs or a distributed ID generator (Snowflake pattern). Don't rely on database sequences.

**Ignoring connection pool math** — If you have 3 shards with 20 connections each, that's 60 connections from your app server. Multiply by your pod count. Plan accordingly.

**No shard-aware testing** — Your integration tests must cover cross-shard scenarios. I've seen teams that only test against a single test database and discover routing bugs in production.

**Transactions across shards** — Distributed transactions (2PC) are fragile and slow. Design your data model so that transactions stay within a single shard. If you can't, look at the Saga pattern instead.

## My Recommended Architecture

For most teams starting with sharding, here's what I recommend:

1. **Start with 4 shards** — Gives you room to grow without immediate rebalancing
2. **Use consistent hashing** — Saves you pain later when adding shards
3. **Shard by tenant/user ID** — Natural partition key that keeps related data together
4. **Keep a global metadata database** — For user lookups, shard directory, and cross-cutting concerns
5. **Use Flyway/Liquibase per shard** — Each shard gets its own migration track
6. **Monitor per-shard metrics separately** — Connection pool utilization, query times, disk usage

## Final Thoughts

Sharding is an organizational decision as much as a technical one. It affects your deployment pipeline, your testing strategy, your on-call runbooks, and your team's cognitive load.

Do it when you must. Not before. And when you do, invest in the tooling to make it manageable — automated shard provisioning, centralized migration management, and comprehensive per-shard observability.

The best sharding implementation is the one your team can operate at 3 AM when something goes wrong.
