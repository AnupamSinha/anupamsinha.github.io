---
title: "Database Sharding Strategies — Range, Hash, and Directory-Based"
date: 2026-08-25
categories: [SQL, System Design]
tags: [sql, postgresql, databases, sharding, system-design]
description: "A practical deep dive into range, hash, and directory-based sharding — routing, rebalancing, cross-shard queries, and global IDs."
---

## Why Shard at All

A single database node has a ceiling: disk, RAM, connections, write throughput, and the physics of one machine. Vertical scaling (a bigger box) buys you time, not forever. **Sharding** is horizontal partitioning across independent database nodes, where each node — a *shard* — holds a disjoint subset of the rows.

The distinction that trips people up: **partitioning** splits a table within one database; **sharding** splits it across many databases that don't share storage or a query planner. A partitioned table can still be joined trivially. Two shards cannot — they're separate servers.

You shard when one of these is true:

- Write throughput exceeds what one primary can absorb (replicas only scale reads).
- The working set no longer fits in RAM, so cache hit ratios collapse.
- A single table is too large to index, vacuum, or back up in a reasonable window.
- You need data locality (EU users in EU, tenant isolation, blast-radius reduction).

Sharding is expensive in complexity. Exhaust read replicas, connection pooling, caching, and archiving first. When you do shard, the entire game is picking a **shard key** and a **routing strategy**.

---

## The Shard Key Is the Decision That Outlives You

Every sharding scheme routes a row to a shard by applying a function to a **shard key**:

```
shard_id = route(shard_key)
```

The shard key is chosen once and is brutal to change later, because changing it means physically moving most of your data. Pick a key that:

1. **Is present in almost every query** so you can route to a single shard instead of fanning out to all of them.
2. **Has high cardinality** so data spreads evenly.
3. **Distributes writes evenly over time** — no monotonic hotspots.

For a multi-tenant SaaS, `tenant_id` is usually right: every query already filters by tenant. For a social app, `user_id`. For events, resist the temptation to shard by `created_at` (a range key) unless time-range queries dominate — it creates a write hotspot on "today."

Let's use an `orders` table sharded by `customer_id` across 4 shards.

```sql
-- Same DDL on every shard. The shards are peers, not a hierarchy.
CREATE TABLE orders (
    id           BIGINT NOT NULL,
    customer_id  BIGINT NOT NULL,
    status       TEXT   NOT NULL,
    total_cents  BIGINT NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (customer_id, id)   -- shard key leads the PK
);
```

Note the composite primary key led by the shard key. A plain global auto-increment `id` doesn't work across shards — you'd get collisions. We'll handle ID generation below.

---

## Strategy 1 — Range-Based Sharding

Assign contiguous ranges of the shard key to shards.

```
customer_id       1 ..  250000  -> shard 0
customer_id  250001 ..  500000  -> shard 1
customer_id  500001 ..  750000  -> shard 2
customer_id  750001 .. 1000000  -> shard 3
```

Routing is a lookup against a range map:

```java
// Range-based router. Ranges live in config or a metadata table.
record ShardRange(long lo, long hi, int shardId) {}

int routeByRange(long customerId, List<ShardRange> ranges) {
    for (ShardRange r : ranges) {
        if (customerId >= r.lo() && customerId <= r.hi()) {
            return r.shardId();
        }
    }
    throw new IllegalStateException("No shard for customer " + customerId);
}
```

**Strengths**

- **Range queries stay local.** `WHERE customer_id BETWEEN 300000 AND 320000` hits exactly one shard.
- **Trivial to add capacity at the top:** new IDs go to a new shard without moving old data.
- Human-readable mapping; easy to reason about which data lives where.

**Weaknesses**

- **Hotspots.** If IDs are monotonically increasing, all new writes land on the highest shard. The last shard is on fire while shard 0 is idle.
- **Uneven distribution** when key density varies (a few whale tenants dominate one range).

Range sharding shines for time-series *reads* and natural-order data, but you must combat the write hotspot. In PostgreSQL you'd often express the per-shard layout with native range partitioning inside each shard:

```sql
-- Within a single shard you can range-partition by time for vacuum/drop efficiency
CREATE TABLE orders (
    id BIGINT NOT NULL,
    customer_id BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026_08 PARTITION OF orders
    FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
```

---

## Strategy 2 — Hash-Based Sharding

Hash the shard key and take it modulo the shard count.

```
shard_id = hash(customer_id) % N
```

```java
int routeByHash(long customerId, int shardCount) {
    // Use a stable hash, not Object.hashCode() which varies by JVM/type.
    long h = fmix64(customerId);          // e.g. Murmur3 finalizer
    int shard = (int) Math.floorMod(h, shardCount);
    return shard;
}

// Murmur3 64-bit finalizer — cheap, well-distributed, deterministic.
static long fmix64(long k) {
    k ^= (k >>> 33);
    k *= 0xff51afd7ed558ccdL;
    k ^= (k >>> 33);
    k *= 0xc4ceb9fe1a85ec53L;
    k ^= (k >>> 33);
    return k;
}
```

You can compute the same hash inside Postgres to verify routing:

```sql
-- Postgres has a stable hash for integers used by hash partitioning.
SELECT abs(hashint8(customer_id)) % 4 AS shard_id
FROM (VALUES (101), (102), (103), (104)) AS t(customer_id);
```

**Strengths**

- **Even distribution.** A good hash spreads writes and storage uniformly regardless of key density.
- **No hotspots** from monotonic keys — sequential IDs scatter across all shards.

**Weaknesses**

- **Range queries fan out** to every shard, then merge client-side. `BETWEEN` is now N queries.
- **Resharding is catastrophic with plain modulo.** Change `N` from 4 to 5 and `hash % N` moves almost every row. This is the classic trap.

### The Resharding Trap and Consistent Hashing

With `hash % 4 → hash % 5`, roughly 80% of keys change shards. You cannot do that live. The fix is **consistent hashing** (or its cousin, **hash slots**), which limits movement to `~1/N` of keys when you add a node.

The pragmatic industry pattern is **fixed logical slots** (what Redis Cluster and Vitess effectively do): hash into a large fixed number of slots (say 4096), then map slots to physical shards. Adding a shard just reassigns some slots.

```java
// Two-level mapping: key -> slot (fixed) -> physical shard (movable).
static final int SLOTS = 4096;

int slotFor(long customerId) {
    return (int) Math.floorMod(fmix64(customerId), SLOTS);
}

int routeViaSlots(long customerId, int[] slotToShard) {
    return slotToShard[slotFor(customerId)];
}
```

Now moving capacity is a matter of migrating a handful of slots and updating `slotToShard`, not rehashing the universe. PostgreSQL's native `PARTITION BY HASH` uses exactly this modulus/remainder idea:

```sql
CREATE TABLE orders (customer_id BIGINT, id BIGINT) PARTITION BY HASH (customer_id);
CREATE TABLE orders_p0 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE orders_p1 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE orders_p2 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE orders_p3 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

---

## Strategy 3 — Directory-Based Sharding

Keep an explicit lookup table (the *directory*) that maps each key — or key group — to a shard. Routing is a query, not a formula.

```sql
-- The directory lives in its own highly-available, cached store.
CREATE TABLE shard_directory (
    tenant_id   BIGINT PRIMARY KEY,
    shard_id    INT NOT NULL,
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Look up where a tenant lives:
SELECT shard_id FROM shard_directory WHERE tenant_id = 90210;
```

```java
// Directory router with a local cache; the directory is the source of truth.
class DirectoryRouter {
    private final Cache<Long, Integer> cache;      // e.g. Caffeine, short TTL
    private final DirectoryStore store;            // backed by DB / etcd / ZK

    int route(long tenantId) {
        Integer cached = cache.getIfPresent(tenantId);
        if (cached != null) return cached;
        int shard = store.lookup(tenantId);         // authoritative
        cache.put(tenantId, shard);
        return shard;
    }
}
```

**Strengths**

- **Maximum flexibility.** You can place any tenant on any shard, pin a whale to a dedicated shard, or colocate related tenants.
- **Rebalancing is a directory update plus a data move** — no formula constraints.
- Enables **heterogeneous shards** (big tenants on big hardware).

**Weaknesses**

- **The directory is a new single point of failure** and a lookup on the hot path. You must cache it aggressively and make it highly available.
- Cache invalidation during migrations is subtle — a stale entry routes to the wrong shard mid-move.

Directory-based is the choice for serious multi-tenant systems because business reality (whales, compliance, per-tenant SLAs) rarely fits a clean formula.

---

## Cross-Shard Queries — The Cost You Pay Forever

Once sharded, any query that doesn't include the shard key becomes a **scatter-gather**: run on all shards, merge results in the app.

```java
// Scatter-gather aggregation across shards. Runs in parallel, merges in app.
long totalPaid(List<DataSource> shards) {
    return shards.parallelStream()
        .mapToLong(ds -> queryOneShard(ds,
            "SELECT COALESCE(SUM(total_cents),0) FROM orders WHERE status='PAID'"))
        .sum();  // SUM merges cleanly; COUNT too.
}
```

Some operations merge cleanly (`SUM`, `COUNT`, `MIN`, `MAX`). Others are painful:

- **`AVG`** — never sum the averages. Fetch `SUM` and `COUNT` per shard, divide once at the end.
- **`ORDER BY ... LIMIT n`** — each shard returns its top `n`, then you merge and re-limit. Pulling `n` from every shard is the only correct way.
- **`DISTINCT` / `GROUP BY`** across shards needs a merge step, and high-cardinality groups can blow up memory.
- **JOINs across shards** are the real pain. Either colocate joined data on the same shard (shard both tables by the same key) or denormalize.

**Cross-shard transactions** lose single-node ACID. You're now in two-phase commit or saga territory, both of which trade latency and complexity for atomicity. The winning move is designing so that a transaction touches exactly one shard — which is why the shard key must appear in your writes.

---

## Global IDs Without a Central Sequence

Auto-increment is dead across shards. Three viable patterns:

```java
// 1) Snowflake-style 64-bit IDs: time-ordered, sortable, no coordination.
//    [ 41 bits ms timestamp | 10 bits node id | 12 bits per-ms sequence ]
long nextId(long nodeId, AtomicLong seq, long epoch) {
    long ts = System.currentTimeMillis() - epoch;
    long s  = seq.getAndIncrement() & 0xFFF;      // 4096 per ms per node
    return (ts << 22) | (nodeId << 12) | s;
}
```

```sql
-- 2) UUIDv7: time-ordered UUID, index-friendly (Postgres 18+ has gen_uuid_v7()).
--    Random UUIDv4 hurts B-tree locality; prefer v7 for primary keys.
SELECT gen_random_uuid();   -- v4, available now via pgcrypto/core
```

```sql
-- 3) Per-shard offset sequences: each shard's sequence steps by shard count.
--    Shard 0 -> 1,5,9,...  Shard 1 -> 2,6,10,...  (start N, increment N)
CREATE SEQUENCE orders_id_seq START WITH 1 INCREMENT BY 4;  -- on shard 0
```

Snowflake and UUIDv7 are time-sortable, which keeps B-tree inserts append-friendly on each shard. Avoid random UUIDv4 as a primary key at scale — it scatters inserts across the index and destroys cache locality.

---

## MySQL Notes

- MySQL has no native hash/range *sharding* either; you shard at the application or middleware layer (Vitess is the reference implementation and speaks the MySQL protocol).
- MySQL `PARTITION BY HASH`/`RANGE` exists but is **single-server partitioning**, same distinction as Postgres — it does not distribute across hosts.
- **Vitess** implements directory-style routing via *vindexes* and hides scatter-gather behind the MySQL wire protocol, which is why it's the go-to for large MySQL fleets.
- For global IDs, MySQL shops commonly use the increment-offset trick (`auto_increment_increment` / `auto_increment_offset`) or Snowflake generators.

---

## Choosing — A Practical Rule of Thumb

- **Range** — when range/time queries dominate and you can tolerate or engineer around write hotspots. Great for archival and time-series.
- **Hash (via fixed slots)** — the sane default for even write distribution when most access is by exact key. Use slots, never raw `% N`.
- **Directory** — when tenants are unequal, compliance/locality matters, or you need surgical placement and rebalancing. The standard for mature multi-tenant SaaS.

And the meta-rule: **don't shard until you must.** Every join, transaction, and analytics query gets harder the moment your data spans hosts. Shard on a key that lives in your queries, keep transactions on one shard, and make IDs coordination-free from day one.
