---
title: "Distributed Locking: 5 Approaches Compared (Redis, ZooKeeper, DB, ShedLock, Hazelcast)"
date: 2026-08-24
categories: [Spring Boot, Architecture]
tags: [distributed-systems, spring-boot, redis, java, microservices]
description: "A deep comparison of distributed lock implementations with benchmarks, failure modes, production war stories, and Spring Boot code for each approach"
mermaid: true
---
Distributed locking is one of those problems that seems solved until it isn't. You Google "distributed lock Spring Boot," copy the first Stack Overflow answer, deploy to production, and six months later you're debugging a situation where two instances both held the "lock" simultaneously and corrupted your data.

I've used all five approaches in production over 17 years. Each has bitten me in different ways. This post gives you the full picture — not just the happy path code, but the failure modes that will haunt you at 3 AM.

## When You Actually Need Distributed Locks

Before we compare implementations, let's be clear about when you need a distributed lock:

- **Preventing duplicate processing** — ensuring only one instance processes a scheduled job
- **Resource coordination** — only one instance should write to a shared file or external API with rate limits
- **Critical sections** — bank account transfers, inventory deductions, sequential ID generation
- **Leader election** — picking one instance as the "primary" for coordination tasks

If you can solve your problem with idempotent operations or optimistic locking, do that instead. Distributed locks add latency, complexity, and failure modes. They're a last resort, not a first choice.

## Approach 1: Redis (Redisson)

Redis is the most popular choice for distributed locking due to its speed and simplicity. Redisson provides a battle-tested implementation of the Redlock algorithm.

```java
@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
                .setAddress("redis://localhost:6379")
                .setConnectionMinimumIdleSize(5)
                .setConnectionPoolSize(10);
        return Redisson.create(config);
    }
}

@Service
@RequiredArgsConstructor
@Slf4j
public class RedisDistributedLock {

    private final RedissonClient redissonClient;

    public <T> T executeWithLock(String lockName, Duration waitTime,
                                  Duration leaseTime, Supplier<T> action) {
        RLock lock = redissonClient.getLock(lockName);

        try {
            boolean acquired = lock.tryLock(
                    waitTime.toMillis(), leaseTime.toMillis(), TimeUnit.MILLISECONDS);

            if (!acquired) {
                throw new LockAcquisitionException(
                        "Failed to acquire lock: " + lockName);
            }

            log.debug("Lock acquired: {} by thread {}", lockName,
                    Thread.currentThread().getName());
            return action.get();

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new LockAcquisitionException("Interrupted while waiting for lock", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
                log.debug("Lock released: {}", lockName);
            }
        }
    }

    /**
     * Fair lock variant - guarantees FIFO ordering
     */
    public <T> T executeWithFairLock(String lockName, Duration waitTime,
                                      Duration leaseTime, Supplier<T> action) {
        RLock lock = redissonClient.getFairLock(lockName);

        try {
            boolean acquired = lock.tryLock(
                    waitTime.toMillis(), leaseTime.toMillis(), TimeUnit.MILLISECONDS);
            if (!acquired) {
                throw new LockAcquisitionException("Failed to acquire fair lock: " + lockName);
            }
            return action.get();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new LockAcquisitionException("Interrupted", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

**Usage in a service:**

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final RedisDistributedLock distributedLock;
    private final AccountRepository accountRepository;

    public TransferResult transfer(String fromAccountId, String toAccountId,
                                    BigDecimal amount) {
        // Lock both accounts in consistent order to prevent deadlocks
        String lockKey = "transfer:" + Stream.of(fromAccountId, toAccountId)
                .sorted()
                .collect(Collectors.joining(":"));

        return distributedLock.executeWithLock(
                lockKey,
                Duration.ofSeconds(5),   // wait up to 5s
                Duration.ofSeconds(30),  // auto-release after 30s
                () -> doTransfer(fromAccountId, toAccountId, amount)
        );
    }

    private TransferResult doTransfer(String from, String to, BigDecimal amount) {
        Account fromAccount = accountRepository.findById(from).orElseThrow();
        Account toAccount = accountRepository.findById(to).orElseThrow();

        if (fromAccount.getBalance().compareTo(amount) < 0) {
            return TransferResult.insufficientFunds();
        }

        fromAccount.debit(amount);
        toAccount.credit(amount);
        accountRepository.saveAll(List.of(fromAccount, toAccount));

        return TransferResult.success();
    }
}
```

**Characteristics:**

- **Acquisition latency** — 1-5ms (single Redis), 5-15ms (Redlock with 3+ nodes)
- **Throughput** — 50K-100K lock acquisitions/second per Redis node
- **Failure mode** — if Redis master fails before replicating the lock key to a replica, two clients can hold the same lock (solved by Redlock, but at cost of latency)
- **Best for** — high-throughput scenarios where occasional lock failure is tolerable, or with Redlock for stronger guarantees

**My production experience:** Redis locks are fast and work well 99.9% of the time. The 0.1% hits you during failovers. I had a case where Redis Sentinel promoted a replica that didn't have the lock key, and two instances ran a payment batch simultaneously. Lost $14K before we caught it. After that, we moved payment-critical locks to ZooKeeper.

## Approach 2: ZooKeeper (Curator)

ZooKeeper provides the strongest consistency guarantees for distributed locking. The sequential ephemeral node approach ensures exactly-once lock acquisition even during failures.

```java
@Configuration
public class ZooKeeperConfig {

    @Bean
    public CuratorFramework curatorFramework(
            @Value("${zookeeper.connection-string}") String connectionString) {
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(connectionString)
                .sessionTimeoutMs(30000)
                .connectionTimeoutMs(10000)
                .retryPolicy(new ExponentialBackoffRetry(1000, 3))
                .build();
        client.start();
        return client;
    }
}

@Service
@RequiredArgsConstructor
@Slf4j
public class ZooKeeperDistributedLock {

    private final CuratorFramework curatorFramework;

    public <T> T executeWithLock(String lockPath, Duration waitTime,
                                  Supplier<T> action) {
        InterProcessMutex mutex = new InterProcessMutex(
                curatorFramework, "/locks/" + lockPath);

        try {
            boolean acquired = mutex.acquire(waitTime.toMillis(), TimeUnit.MILLISECONDS);
            if (!acquired) {
                throw new LockAcquisitionException(
                        "Failed to acquire ZK lock: " + lockPath);
            }

            log.debug("ZK lock acquired: {}", lockPath);
            return action.get();

        } catch (Exception e) {
            if (e instanceof LockAcquisitionException) throw (LockAcquisitionException) e;
            throw new LockAcquisitionException("ZK lock error", e);
        } finally {
            try {
                if (mutex.isAcquiredInThisProcess()) {
                    mutex.release();
                    log.debug("ZK lock released: {}", lockPath);
                }
            } catch (Exception e) {
                log.error("Failed to release ZK lock: {}", lockPath, e);
            }
        }
    }

    /**
     * Read-write lock for scenarios where reads can be concurrent
     * but writes must be exclusive
     */
    public <T> T executeWithReadLock(String lockPath, Duration waitTime,
                                      Supplier<T> action) {
        InterProcessReadWriteLock rwLock = new InterProcessReadWriteLock(
                curatorFramework, "/locks/" + lockPath);

        try {
            boolean acquired = rwLock.readLock()
                    .acquire(waitTime.toMillis(), TimeUnit.MILLISECONDS);
            if (!acquired) {
                throw new LockAcquisitionException("Failed to acquire read lock");
            }
            return action.get();
        } catch (Exception e) {
            if (e instanceof LockAcquisitionException) throw (LockAcquisitionException) e;
            throw new LockAcquisitionException("ZK read lock error", e);
        } finally {
            try {
                rwLock.readLock().release();
            } catch (Exception ignored) {}
        }
    }
}
```

**Characteristics:**

- **Acquisition latency** — 10-50ms (ZooKeeper consensus overhead)
- **Throughput** — 5K-15K lock acquisitions/second (limited by ZK write throughput)
- **Failure mode** — if the lock holder's session expires (GC pause, network partition), ZK automatically releases the lock. The holder might still think it has the lock (fencing tokens solve this)
- **Best for** — critical operations where correctness trumps performance (financial transactions, leader election)

**My production experience:** ZooKeeper is rock-solid for correctness but operationally heavy. The ZK cluster itself needs care — if you lose quorum, all locks freeze. We ran a 5-node ZK cluster for payment processing locks and it worked flawlessly for 3 years. The downside: ZK is another piece of infrastructure to maintain, monitor, and upgrade.

## Approach 3: Database (PostgreSQL Advisory Locks)

Often overlooked, database advisory locks are perfect when you already have a database and don't want to introduce new infrastructure.

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class DatabaseDistributedLock {

    private final JdbcTemplate jdbcTemplate;
    private final DataSource dataSource;

    /**
     * PostgreSQL advisory lock using session-level locking
     * Lock is automatically released when the connection is returned to the pool
     */
    public <T> T executeWithAdvisoryLock(long lockId, Supplier<T> action) {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);

            // Try to acquire advisory lock (non-blocking)
            try (PreparedStatement stmt = connection.prepareStatement(
                    "SELECT pg_try_advisory_lock(?)")) {
                stmt.setLong(1, lockId);
                ResultSet rs = stmt.executeQuery();
                rs.next();
                boolean acquired = rs.getBoolean(1);

                if (!acquired) {
                    throw new LockAcquisitionException(
                            "Failed to acquire advisory lock: " + lockId);
                }
            }

            try {
                T result = action.get();
                connection.commit();
                return result;
            } catch (Exception e) {
                connection.rollback();
                throw e;
            } finally {
                // Release advisory lock
                try (PreparedStatement stmt = connection.prepareStatement(
                        "SELECT pg_advisory_unlock(?)")) {
                    stmt.setLong(1, lockId);
                    stmt.executeQuery();
                }
            }
        } catch (SQLException e) {
            throw new LockAcquisitionException("Database lock error", e);
        }
    }

    /**
     * Table-based lock with timeout (works with any database)
     */
    public <T> T executeWithTableLock(String lockName, Duration timeout,
                                       Supplier<T> action) {
        Instant deadline = Instant.now().plus(timeout);

        while (Instant.now().isBefore(deadline)) {
            try {
                int inserted = jdbcTemplate.update(
                        "INSERT INTO distributed_locks (lock_name, locked_by, locked_at, expires_at) " +
                        "VALUES (?, ?, NOW(), NOW() + INTERVAL '30 seconds') " +
                        "ON CONFLICT (lock_name) DO NOTHING",
                        lockName, getInstanceId());

                if (inserted > 0) {
                    try {
                        return action.get();
                    } finally {
                        jdbcTemplate.update(
                                "DELETE FROM distributed_locks WHERE lock_name = ? AND locked_by = ?",
                                lockName, getInstanceId());
                    }
                }

                // Check if existing lock has expired
                int cleaned = jdbcTemplate.update(
                        "DELETE FROM distributed_locks WHERE lock_name = ? AND expires_at < NOW()",
                        lockName);
                if (cleaned > 0) continue;

                Thread.sleep(100); // Retry after short delay
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new LockAcquisitionException("Interrupted", e);
            }
        }

        throw new LockAcquisitionException("Timeout acquiring table lock: " + lockName);
    }

    private String getInstanceId() {
        return ManagementFactory.getRuntimeMXBean().getName();
    }
}
```

The table DDL:

```sql
CREATE TABLE distributed_locks (
    lock_name VARCHAR(255) PRIMARY KEY,
    locked_by VARCHAR(255) NOT NULL,
    locked_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_locks_expires ON distributed_locks(expires_at);
```

**Characteristics:**

- **Acquisition latency** — 2-10ms (advisory locks), 5-50ms (table-based with retries)
- **Throughput** — 10K-30K acquisitions/second (limited by database transaction throughput)
- **Failure mode** — advisory locks release on connection close (good). Table-based locks can become stale if the holder crashes without cleanup (solved by expiry column)
- **Best for** — applications that already have a database but don't need extreme lock throughput. Great for scheduled jobs

**My production experience:** PostgreSQL advisory locks are underrated. For scheduled job coordination (ensuring only one instance runs a daily report), they're perfect. Zero additional infrastructure. The one gotcha: if you use connection pooling (HikariCP), make sure you're not accidentally returning a connection to the pool while still holding an advisory lock. Use `pg_try_advisory_xact_lock` for transaction-scoped locks to avoid this.

## Approach 4: ShedLock

ShedLock is purpose-built for the most common distributed lock use case: ensuring scheduled tasks run on only one instance. It's not a general-purpose lock — it's specifically designed for `@Scheduled` methods.

```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class ShedLockConfig {

    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
                JdbcTemplateLockProvider.Configuration.builder()
                        .withJdbcTemplate(new JdbcTemplate(dataSource))
                        .usingDbTime()
                        .build()
        );
    }

    // Alternative: Redis-based lock provider
    @Bean
    @ConditionalOnProperty(name = "shedlock.provider", havingValue = "redis")
    public LockProvider redisLockProvider(RedisConnectionFactory connectionFactory) {
        return new RedisLockProvider(connectionFactory);
    }
}

@Service
@Slf4j
public class ScheduledTasks {

    @Autowired
    private ReportService reportService;

    @Autowired
    private DataCleanupService cleanupService;

    /**
     * Daily report generation - must run on exactly one instance
     * lockAtLeastFor prevents execution on other nodes even if this one finishes quickly
     */
    @Scheduled(cron = "0 0 2 * * *") // 2 AM daily
    @SchedulerLock(name = "generateDailyReport",
            lockAtLeastFor = "5m",
            lockAtMostFor = "30m")
    public void generateDailyReport() {
        log.info("Starting daily report generation");
        reportService.generateAndEmail();
        log.info("Daily report generation complete");
    }

    /**
     * Cleanup expired sessions every 15 minutes
     */
    @Scheduled(fixedRate = 900_000) // 15 minutes
    @SchedulerLock(name = "cleanupExpiredSessions",
            lockAtLeastFor = "5m",
            lockAtMostFor = "14m")
    public void cleanupExpiredSessions() {
        log.info("Starting session cleanup");
        int cleaned = cleanupService.removeExpiredSessions();
        log.info("Cleaned {} expired sessions", cleaned);
    }

    /**
     * Retry failed payment notifications every 5 minutes
     */
    @Scheduled(fixedRate = 300_000)
    @SchedulerLock(name = "retryFailedNotifications",
            lockAtLeastFor = "2m",
            lockAtMostFor = "4m")
    public void retryFailedNotifications() {
        log.info("Retrying failed notifications");
        reportService.retryFailed();
    }
}
```

The DDL for ShedLock (PostgreSQL):

```sql
CREATE TABLE shedlock (
    name VARCHAR(64) NOT NULL PRIMARY KEY,
    lock_until TIMESTAMP NOT NULL,
    locked_at TIMESTAMP NOT NULL,
    locked_by VARCHAR(255) NOT NULL
);
```

**Characteristics:**

- **Acquisition latency** — 2-5ms (simple SELECT + UPDATE)
- **Throughput** — N/A (designed for low-frequency scheduled tasks)
- **Failure mode** — `lockAtMostFor` ensures the lock is released even if the holder crashes. Minimal risk of deadlocks
- **Best for** — scheduled tasks that must run on exactly one instance (the 90% use case for distributed locks in microservices)

**My production experience:** ShedLock is what I reach for first. In 90% of cases where developers think they need a "distributed lock," what they actually need is "don't run this cron job on all 5 instances simultaneously." ShedLock handles that perfectly with minimal configuration. It's boring in the best way.

## Approach 5: Hazelcast (In-Memory Data Grid)

Hazelcast provides distributed locks as part of its in-memory data grid. The advantage: no external infrastructure required — the lock state lives in the application cluster itself.

```java
@Configuration
public class HazelcastConfig {

    @Bean
    public HazelcastInstance hazelcastInstance() {
        Config config = new Config();
        config.setClusterName("my-application");

        // Configure network discovery
        NetworkConfig networkConfig = config.getNetworkConfig();
        JoinConfig joinConfig = networkConfig.getJoin();
        joinConfig.getMulticastConfig().setEnabled(false);
        joinConfig.getTcpIpConfig()
                .setEnabled(true)
                .addMember("10.0.0.1")
                .addMember("10.0.0.2")
                .addMember("10.0.0.3");

        // Configure CP subsystem for strong consistency (Hazelcast 4+)
        config.getCPSubsystemConfig()
                .setCPMemberCount(3)
                .setGroupSize(3);

        return Hazelcast.newHazelcastInstance(config);
    }
}

@Service
@RequiredArgsConstructor
@Slf4j
public class HazelcastDistributedLock {

    private final HazelcastInstance hazelcastInstance;

    /**
     * Standard lock using CP subsystem (strongly consistent)
     */
    public <T> T executeWithLock(String lockName, Duration waitTime,
                                  Supplier<T> action) {
        FencedLock lock = hazelcastInstance.getCPSubsystem()
                .getLock(lockName);

        try {
            boolean acquired = lock.tryLock(
                    waitTime.toMillis(), TimeUnit.MILLISECONDS);

            if (!acquired) {
                throw new LockAcquisitionException(
                        "Failed to acquire Hazelcast lock: " + lockName);
            }

            long fenceToken = lock.getFence();
            log.debug("Hazelcast lock acquired: {}, fence: {}", lockName, fenceToken);

            return action.get();

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new LockAcquisitionException("Interrupted", e);
        } finally {
            try {
                lock.unlock();
            } catch (Exception e) {
                log.warn("Failed to unlock Hazelcast lock: {}", lockName, e);
            }
        }
    }

    /**
     * Distributed semaphore for limiting concurrent access
     */
    public <T> T executeWithSemaphore(String name, int permits,
                                       Duration waitTime, Supplier<T> action) {
        ISemaphore semaphore = hazelcastInstance.getCPSubsystem()
                .getSemaphore(name);
        semaphore.init(permits);

        try {
            boolean acquired = semaphore.tryAcquire(1,
                    waitTime.toMillis(), TimeUnit.MILLISECONDS);
            if (!acquired) {
                throw new LockAcquisitionException("Semaphore not available: " + name);
            }
            return action.get();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new LockAcquisitionException("Interrupted", e);
        } finally {
            semaphore.release(1);
        }
    }
}
```

**Characteristics:**

- **Acquisition latency** — 5-20ms (depends on cluster size and network)
- **Throughput** — 20K-50K acquisitions/second (limited by CP subsystem consensus)
- **Failure mode** — with CP subsystem enabled, uses Raft consensus. Lock is released when holder's session expires. Without CP subsystem, locks are AP (eventually consistent) and can be lost during split-brain
- **Best for** — applications that already use Hazelcast for caching or already need an in-memory data grid. Avoids adding Redis/ZK as separate infrastructure

**My production experience:** Hazelcast locks are convenient when you're already running Hazelcast. The CP subsystem (added in Hazelcast 4) makes them genuinely safe for critical sections. Before CP subsystem, Hazelcast locks were AP — during network partitions, both sides could acquire the "same" lock. Always enable CP subsystem if you use Hazelcast for locking.

## Head-to-Head Comparison

**Redis (Redisson)**
- **Latency** — 1-5ms (single node), 5-15ms (Redlock)
- **Throughput** — 50K-100K/s
- **Consistency** — AP (single node), CP-ish (Redlock)
- **Operational cost** — Low (Redis is common infrastructure)
- **Failure recovery** — Manual (or auto via expiry)

**ZooKeeper (Curator)**
- **Latency** — 10-50ms
- **Throughput** — 5K-15K/s
- **Consistency** — CP (linearizable)
- **Operational cost** — High (dedicated ZK cluster)
- **Failure recovery** — Automatic (ephemeral nodes)

**Database (Advisory Locks)**
- **Latency** — 2-10ms
- **Throughput** — 10K-30K/s
- **Consistency** — CP (ACID guarantees)
- **Operational cost** — Zero (you already have a DB)
- **Failure recovery** — Automatic (connection close)

**ShedLock**
- **Latency** — 2-5ms
- **Throughput** — N/A (scheduled tasks only)
- **Consistency** — Eventual (but sufficient for its use case)
- **Operational cost** — Zero (uses existing DB or Redis)
- **Failure recovery** — Automatic (lockAtMostFor)

**Hazelcast (CP Subsystem)**
- **Latency** — 5-20ms
- **Throughput** — 20K-50K/s
- **Consistency** — CP (Raft-based)
- **Operational cost** — Medium (embedded, but needs CP config)
- **Failure recovery** — Automatic (session expiry)

## My Decision Framework

After years of production experience, here's my decision tree:

**"I need to coordinate scheduled tasks"** → ShedLock. Don't overthink it.

**"I need general-purpose locks and already have Redis"** → Redisson. Fast, simple, handles 99% of cases.

**"I need absolute correctness for financial operations"** → ZooKeeper or PostgreSQL advisory locks. ZK if you need high throughput; DB if you want zero additional infrastructure.

**"I already run Hazelcast"** → Hazelcast CP subsystem. Don't add Redis just for locking.

**"I need locks AND I need them under 5ms"** → Redis single-node with careful monitoring. Accept the tiny window of inconsistency during failovers.

## The Fencing Token Pattern

Regardless of which approach you use, fencing tokens protect against a subtle bug: a lock holder that pauses (GC, network), loses the lock, but then resumes and acts as if it still holds the lock.

```java
@Service
@RequiredArgsConstructor
public class FencedLockService {

    private final RedissonClient redissonClient;
    private final JdbcTemplate jdbcTemplate;

    public void updateWithFencing(String resourceId, String newValue) {
        RLock lock = redissonClient.getLock("resource:" + resourceId);

        try {
            lock.lock();

            // Get a monotonically increasing token
            Long fenceToken = redissonClient.getAtomicLong(
                    "fence:" + resourceId).incrementAndGet();

            // Conditional update: only succeeds if our token is the latest
            int updated = jdbcTemplate.update(
                    "UPDATE resources SET value = ?, fence_token = ? " +
                    "WHERE id = ? AND fence_token < ?",
                    newValue, fenceToken, resourceId, fenceToken);

            if (updated == 0) {
                log.warn("Fencing token rejected update for resource {}. " +
                         "Another holder has written since our lock acquisition.",
                         resourceId);
            }
        } finally {
            lock.unlock();
        }
    }
}
```

The fence token ensures that even if the lock implementation fails and two clients believe they hold the lock, only the most recent one's writes will be accepted by downstream storage.

## Final Thoughts

The "best" distributed lock doesn't exist. What exists is the right trade-off for your specific situation. In my experience, most teams overthink this. Start with ShedLock for scheduled tasks. Add Redisson if you need general-purpose locks. Only reach for ZooKeeper when you have a genuine linearizability requirement.

And whatever you choose — add monitoring. Alert on lock acquisition failures, lock hold times exceeding thresholds, and lock timeouts. The lock implementation that you don't monitor is the one that will wake you up at 3 AM.
