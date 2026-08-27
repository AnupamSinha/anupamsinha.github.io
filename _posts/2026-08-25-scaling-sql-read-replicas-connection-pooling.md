---
title: "Read Replicas, Connection Pooling, and Scaling SQL to Millions of Users"
date: 2026-08-25
categories: [SQL, System Design]
tags: [sql, postgresql, databases, replication, connection-pooling, system-design]
description: "How streaming replication, replica lag, read/write splitting, and PgBouncer connection pooling scale SQL from thousands to millions of users."
---

## The Scaling Ladder

Most teams reach for sharding way too early. The honest scaling ladder for a relational database looks like this:

1. **Fix your queries and indexes.** Most "we need to scale" problems are a missing index.
2. **Add connection pooling.** A shocking amount of database pain is connection exhaustion, not query load.
3. **Add read replicas** and route read traffic off the primary.
4. **Cache** hot reads outside the database entirely.
5. **Shard** — only when a single primary can't absorb the write volume.

Rungs 2 and 3 are what carry you from thousands to millions of users on a single logical database. This post is about doing them correctly, because both have sharp edges.

---

## How Streaming Replication Works

PostgreSQL replication ships the **Write-Ahead Log (WAL)**. Every change is written to the WAL first; replicas connect to the primary, stream those WAL records, and replay them to stay in sync.

```
        writes                  WAL stream
client ───────▶ PRIMARY ───────────────────▶ REPLICA 1 (read-only)
                   │       WAL stream
                   └───────────────────────▶ REPLICA 2 (read-only)
```

Minimal primary config for streaming replication:

```conf
# postgresql.conf on the primary
wal_level = replica
max_wal_senders = 10          # how many replicas/backups can stream at once
wal_keep_size = 1024          # MB of WAL to retain for lagging replicas
hot_standby = on              # replicas can serve read queries while replaying
```

A replica is created from a base backup and then follows the stream:

```bash
# Take a consistent base backup and set up standby signal + connection info
pg_basebackup -h primary-host -U replicator -D /var/lib/pg/data -R -P
# The -R flag writes standby.signal + primary_conninfo automatically
```

Replicas are **read-only**. Any `INSERT`/`UPDATE`/`DELETE` sent to a replica errors out with `cannot execute ... in a read-only transaction`. That's a feature — it's how the app enforces the split.

---

## Synchronous vs Asynchronous — Pick Your Poison

By default replication is **asynchronous**: the primary commits and acknowledges the client *before* the replica has the data. Fast, but the replica is momentarily behind — this is **replica lag**.

```conf
# Asynchronous (default): primary does NOT wait for replicas. Lowest latency,
# small window of potential data loss on primary failure.
synchronous_commit = on
synchronous_standby_names = ''      # empty => async

# Synchronous: primary waits for at least one named standby to confirm.
# Zero data loss on failover, but every commit pays a network round trip.
synchronous_standby_names = 'FIRST 1 (replica1, replica2)'
```

The trade-off is fundamental and you can't cheat it:

- **Async** — low write latency, but a failover can lose the last few unreplicated commits, and reads on the replica can be stale.
- **Sync** — no data loss on failover, but commit latency now includes a round trip to a standby, and if the sync standby goes down, commits can **block** entirely.

Most systems run async and design the application to tolerate a little staleness. Use sync replication only for data where losing a committed write is unacceptable (payments, ledgers) — and keep at least two candidate standbys so one failure doesn't stall all writes.

---

## Read/Write Splitting in the Application

The core pattern: writes and read-your-writes go to the primary; everything else goes to a replica.

```java
// Two DataSources. Route by whether the operation writes or needs fresh data.
@Configuration
class RoutingDataSourceConfig {

    @Bean @Primary
    DataSource routingDataSource(
            @Qualifier("primary") DataSource primary,
            @Qualifier("replica") DataSource replica) {

        var router = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                // Set per-request/per-transaction by an interceptor or annotation.
                return TransactionSynchronizationManager
                        .isCurrentTransactionReadOnly() ? "replica" : "primary";
            }
        };
        router.setTargetDataSources(Map.of("primary", primary, "replica", replica));
        router.setDefaultTargetDataSource(primary);
        return router;
    }
}
```

With Spring, `@Transactional(readOnly = true)` becomes the routing signal:

```java
@Service
class OrderService {

    @Transactional(readOnly = true)          // -> routed to a read replica
    public List<Order> recentOrders(long customerId) {
        return orderRepo.findRecent(customerId);
    }

    @Transactional                            // -> routed to the primary
    public Order placeOrder(NewOrder cmd) {
        return orderRepo.save(cmd.toEntity());
    }
}
```

This is clean, but it introduces the single most common replica bug: **read-your-own-writes**.

---

## The Read-Your-Writes Trap

User updates their profile (write → primary), then the next request reads the profile (read → replica). If the replica hasn't caught up, the user sees their *old* data and assumes the save failed.

```
t0  POST /profile   -> PRIMARY commits "new name"
t1  GET  /profile   -> REPLICA (still 40ms behind) returns "old name"  ❌
```

Three defenses, in increasing sophistication:

```java
// 1) Sticky reads: after a write, route this user's reads to the primary
//    for a short window (e.g. a few seconds).
if (recentlyWrote(userId)) {
    return primary.query(...);
}
return replica.query(...);
```

```sql
-- 2) Wait for the replica to reach the write's WAL position (LSN).
--    Capture the LSN on the primary right after committing:
SELECT pg_current_wal_lsn();          -- e.g. '0/16B3748'

-- Then on the replica, only read once it has replayed past that LSN:
SELECT pg_last_wal_replay_lsn() >= '0/16B3748'::pg_lsn AS caught_up;
```

```java
// 3) Bounded staleness: check lag and fall back to primary if the replica
//    is too far behind for this operation's tolerance.
Duration lag = replicaLag();          // measured below
DataSource ds = lag.compareTo(Duration.ofSeconds(1)) < 0 ? replica : primary;
```

The pragmatic default in most apps: route reads that immediately follow a write (or belong to the same user session) to the primary, and send everything else to replicas.

---

## Measuring Replica Lag — Do It Continuously

You cannot manage lag you don't measure. On the replica:

```sql
-- Time-based lag: how old is the data the replica is serving?
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;

-- Byte-based lag from the primary's perspective (run on the primary):
SELECT client_addr,
       state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

Alert on both. Time-based lag tells you staleness (user-facing correctness); byte-based lag tells you whether a replica is falling behind faster than it can catch up (a runaway that will eventually break replication if WAL is recycled). Long-running queries on a replica can also stall replay via replication conflicts — watch `max_standby_streaming_delay`.

---

## Connection Pooling — The Cheapest Big Win

Every PostgreSQL connection is a **separate OS process** with real memory overhead (work_mem, catalog caches, etc.). A few hundred connections can consume gigabytes and starve the machine. Yet `max_connections` is often set to 100–500, and web apps happily open thousands.

The fix is a pool. There are two layers, and you usually need both:

- **Application pool** (HikariCP) — reuses connections within one app instance.
- **Server-side pooler** (PgBouncer) — multiplexes thousands of client connections onto a small number of real Postgres connections, across all app instances.

### Sizing the Application Pool

The classic mistake is a huge pool. More connections than the database has cores usually makes throughput *worse* due to contention. A well-known starting formula:

```
pool_size ≈ (core_count * 2) + effective_spindle_count
```

```java
// HikariCP — small, correctly sized pool beats a giant one.
HikariConfig cfg = new HikariConfig();
cfg.setJdbcUrl("jdbc:postgresql://pgbouncer:6432/app");
cfg.setMaximumPoolSize(10);              // NOT 200. Start small, measure.
cfg.setMinimumIdle(10);                  // avoid churn; keep it flat
cfg.setConnectionTimeout(3_000);         // fail fast if pool is exhausted
cfg.setMaxLifetime(1_800_000);           // recycle before server-side timeouts
cfg.setLeakDetectionThreshold(20_000);   // log connections held too long
```

If ten app instances each open 10 connections, that's 100 real connections — which is why you also need PgBouncer in front, so the database sees a bounded number regardless of instance count.

---

## PgBouncer and Pool Modes

PgBouncer sits between your app and Postgres and pools connections. Its **pool mode** is the setting that matters most:

```ini
# pgbouncer.ini
[databases]
app = host=primary-host port=5432 dbname=app

[pgbouncer]
listen_port = 6432
pool_mode = transaction        # session | transaction | statement
max_client_conn = 10000        # clients can connect this many...
default_pool_size = 20         # ...but only 20 real server conns per db/user
```

- **session** — a server connection is tied to a client for the whole session. Safe but barely better than no pooler for connection count.
- **transaction** — a server connection is assigned only for the duration of a transaction, then returned. This is the sweet spot: thousands of clients, a handful of server connections.
- **statement** — returns after every statement; breaks multi-statement transactions. Rarely used.

The catch with **transaction mode**: features that rely on session state break, because a client doesn't keep the same server connection between transactions.

```sql
-- These DO NOT work reliably in transaction pooling mode:
--   * Session-level PREPARE / server-side prepared statements (unless configured)
--   * SET session variables expected to persist across transactions
--   * advisory locks held across transactions
--   * LISTEN / NOTIFY
-- Keep such logic inside a single transaction, or use a session-mode pool for it.
```

For JDBC, disable driver-managed server-side prepared statements (or enable PgBouncer's prepared-statement support in recent versions) when using transaction mode:

```java
cfg.addDataSourceProperty("prepareThreshold", "0");   // avoid server-side prepares
```

---

## Putting It Together — A Reference Topology

```
                   ┌─────────────────────────────┐
   app instances   │  HikariCP (10 conns each)   │
   x N ────────────┤                             │
                   └──────────────┬──────────────┘
                                  │ 6432
                          ┌───────▼────────┐
                          │   PgBouncer    │  transaction pooling
                          │ default_pool=20│
                          └───┬────────┬───┘
                    writes    │        │   reads
                        ┌─────▼──┐  ┌──▼──────────┐
                        │PRIMARY │  │  REPLICAS   │  (read-only, async)
                        └────────┘  └─────────────┘
```

- Writes and read-your-writes → primary.
- Bulk reads, reports, dashboards → replicas.
- Every connection path goes through PgBouncer so the database never sees more than a bounded number of real connections.
- Monitor replica lag; fall back to primary when lag exceeds an operation's tolerance.

---

## MySQL Notes

- MySQL replication ships the **binary log (binlog)** rather than WAL, but the mental model is the same: primary logs changes, replicas replay them, and async is the default.
- Read-your-writes has the same trap. MySQL exposes GTIDs and `WAIT_FOR_EXECUTED_GTID_SET()` (analogous to waiting for a Postgres LSN) to gate reads until a replica has caught up.
- **ProxySQL** is the common read/write splitter and connection multiplexer for MySQL, filling roughly the role PgBouncer + routing plays here.
- MySQL connections are threads, not processes, so the per-connection memory cost is lower than Postgres — but pooling still matters for churn and contention.

---

## The Takeaways

- Pooling is the highest return-on-effort scaling change you can make: small HikariCP pools plus PgBouncer in transaction mode lets a modest database serve enormous client fan-out.
- Read replicas scale reads, not writes, and they're **eventually consistent** — design for read-your-writes with sticky reads or LSN waits.
- Measure replica lag continuously and route around it; staleness is a correctness bug, not just a latency one.
- Do all of this before you even think about sharding. Most millions-of-users systems run on one primary plus replicas plus a pooler for a very long time.
