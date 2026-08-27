---
title: "Why Your SQL Query Is Slow — 12 Common Causes"
date: 2026-08-25
categories: [SQL, Performance]
tags: [sql, postgresql, performance, indexing, query-tuning, tutorial]
description: "A practical, code-heavy guide to diagnosing slow SQL — twelve common causes with the query, the EXPLAIN output, and the fix."
---
## Slow Is a Symptom, Not a Diagnosis

"The query is slow" tells you nothing. A query can be slow because it scans too many rows, sorts on disk, waits on locks, returns too much data, or gets a bad plan from stale statistics. The job is to find *which* one.

Before we go through the twelve causes, set up a reproducible baseline. Everything in this post uses this schema:

```sql
CREATE TABLE customers (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       TEXT NOT NULL,
    country     TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id  BIGINT NOT NULL REFERENCES customers(id),
    status       TEXT NOT NULL,          -- 'PENDING','PAID','SHIPPED','CANCELLED'
    total_cents  BIGINT NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 500k customers, 5M orders
INSERT INTO customers (email, country, created_at)
SELECT 'user' || g || '@example.com',
       (ARRAY['SG','IN','US','DE','JP'])[1 + (g % 5)],
       now() - (random() * interval '730 days')
FROM generate_series(1, 500000) g;

INSERT INTO orders (customer_id, status, total_cents, created_at)
SELECT 1 + (random() * 499999)::bigint,
       (ARRAY['PENDING','PAID','SHIPPED','CANCELLED'])[1 + (random() * 3)::int],
       (random() * 500000)::bigint,
       now() - (random() * interval '365 days')
FROM generate_series(1, 5000000) g;

ANALYZE customers;
ANALYZE orders;
```

Your primary diagnostic tool is `EXPLAIN (ANALYZE, BUFFERS)`. `ANALYZE` actually runs the query and reports real timing and row counts. `BUFFERS` shows you how much data was read from cache versus disk. Never trust plain `EXPLAIN` alone — estimated rows lie, actual rows don't.

---

## 1. Full Table Scan on a Filtered Query

The most common cause. You filter on a column that has no index, so the database reads every row.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'PENDING';
```

```
Seq Scan on orders  (cost=0.00..97331.00 rows=1248667 width=45)
                    (actual time=0.031..612.443 rows=1250412 loops=1)
  Filter: (status = 'PENDING'::text)
  Rows Removed by Filter: 3749588
  Buffers: shared hit=2048 read=45283
Planning Time: 0.096 ms
Execution Time: 668.201 ms
```

`Seq Scan` plus `Rows Removed by Filter: 3749588` is the tell. The database read 5M rows to return 1.25M. If the result set were small, an index would help enormously:

```sql
CREATE INDEX idx_orders_status ON orders (status);
```

But here's the nuance: this filter matches 25% of the table. PostgreSQL will *correctly* ignore an index for a low-selectivity predicate, because a scan is cheaper than millions of random index lookups. Indexing pays off when your predicate is *selective* — it returns a small fraction of rows. We'll come back to this in cause #6.

**MySQL note:** InnoDB clusters the table on the primary key, so a "full scan" walks the clustered index. The `EXPLAIN` output shows `type: ALL` for a full scan versus `ref`/`range` when an index is used.

---

## 2. A Function or Cast Wrapped Around an Indexed Column

You have an index, but you accidentally made it useless by transforming the column.

```sql
CREATE INDEX idx_orders_created ON orders (created_at);

-- This CANNOT use the index:
EXPLAIN ANALYZE
SELECT * FROM orders WHERE DATE(created_at) = '2025-06-01';
```

```
Seq Scan on orders  (cost=0.00..109831.00 rows=25000 width=45)
  Filter: (date(created_at) = '2025-06-01'::date)
```

The index is on `created_at`, not on `DATE(created_at)`. Wrapping the column in a function forces a scan. Rewrite as a range so the raw column is exposed to the planner:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE created_at >= '2025-06-01' AND created_at < '2025-06-02';
```

```
Index Scan using idx_orders_created on orders
  (actual time=0.028..3.114 rows=13698 loops=1)
  Index Cond: ((created_at >= '2025-06-01') AND (created_at < '2025-06-02'))
```

Same logic applies to `WHERE LOWER(email) = ...`, `WHERE total_cents + 100 > ...`, and implicit casts like `WHERE customer_id = '42'` (string vs bigint). If you genuinely need the function, build an **expression index**:

```sql
CREATE INDEX idx_orders_created_date ON orders ((DATE(created_at)));
```

**MySQL note:** MySQL 8.0.13+ supports functional indexes with `CREATE INDEX ... ((DATE(created_at)))`. Older versions need a generated column.

---

## 3. Leading Wildcard LIKE

```sql
SELECT * FROM customers WHERE email LIKE '%example.com';
```

A B-tree index sorts left-to-right. `LIKE 'abc%'` can use it (the prefix is known); `LIKE '%abc'` cannot, because the leading characters are unknown. This becomes a full scan.

For suffix or substring search, use a trigram index:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_customers_email_trgm ON customers USING gin (email gin_trgm_ops);

EXPLAIN ANALYZE
SELECT * FROM customers WHERE email LIKE '%user123%';
```

```
Bitmap Heap Scan on customers  (actual time=0.412..0.907 rows=11 loops=1)
  Recheck Cond: (email ~~ '%user123%'::text)
  ->  Bitmap Index Scan on idx_customers_email_trgm  (actual time=0.388..0.388 rows=11 loops=1)
        Index Cond: (email ~~ '%user123%'::text)
```

For real full-text needs, reach for `tsvector`/`tsquery` and a GIN index instead.

---

## 4. Wrong Column Order in a Composite Index

Composite indexes are ordered. `(a, b)` can serve `WHERE a = ?`, `WHERE a = ? AND b = ?`, and `ORDER BY a, b`, but **not** `WHERE b = ?` alone.

```sql
CREATE INDEX idx_orders_cust_status ON orders (customer_id, status);

-- Uses the index (leading column present):
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42 AND status = 'PAID';

-- Does NOT use it efficiently (skips the leading column):
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'PAID';
```

Rule of thumb: **equality columns first, then the column you range-scan or sort on**. If most of your queries filter by `status` and then range on `created_at`, the index should be `(status, created_at)`, not `(created_at, status)`.

```sql
CREATE INDEX idx_orders_status_created ON orders (status, created_at DESC);

EXPLAIN ANALYZE
SELECT * FROM orders
WHERE status = 'PAID' AND created_at >= now() - interval '7 days'
ORDER BY created_at DESC;
```

The planner can now satisfy the filter and the sort from a single index range.

---

## 5. Sorting That Spills to Disk

Sorting is fine until the data doesn't fit in `work_mem`, at which point PostgreSQL sorts on disk — orders of magnitude slower.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders ORDER BY total_cents DESC LIMIT 100;
```

```
Limit  (actual time=1450.211..1450.230 rows=100 loops=1)
  ->  Sort  (actual time=1450.209..1450.219 rows=100 loops=1)
        Sort Key: total_cents DESC
        Sort Method: external merge  Disk: 204800kB
        ->  Seq Scan on orders (actual time=0.018..430.5 rows=5000000 loops=1)
```

`Sort Method: external merge  Disk: 204800kB` means it spilled ~200MB to disk. Two fixes.

Fix A — give the sort more memory (per-operation, be careful in production):

```sql
SET work_mem = '256MB';
```

Fix B — let an index provide the order so no sort is needed at all:

```sql
CREATE INDEX idx_orders_total ON orders (total_cents DESC);
```

```
Limit  (actual time=0.041..0.128 rows=100 loops=1)
  ->  Index Scan using idx_orders_total on orders (actual time=0.039..0.113 rows=100 loops=1)
```

`Sort Method: quicksort  Memory: ...` is in-memory and fine. `external merge Disk:` is the red flag.

---

## 6. Bad Row Estimates from Stale Statistics

The planner picks a plan based on statistics gathered by `ANALYZE`. When those are stale, estimates diverge from reality and you get a bad plan.

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'PENDING';
--  ... rows=12 ...  (estimated)   actual rows=1250412
```

When the estimated `rows` is wildly off from actual `rows`, suspect statistics. Fix:

```sql
ANALYZE orders;

-- For skewed columns, raise the statistics target so the planner keeps more histogram buckets:
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
ANALYZE orders;
```

For correlated columns (e.g., `country` and `status` predict each other), PostgreSQL 10+ supports extended statistics:

```sql
CREATE STATISTICS orders_stat (dependencies, ndistinct)
    ON status, customer_id FROM orders;
ANALYZE orders;
```

Autovacuum normally keeps stats fresh, but after a bulk load or a large `DELETE`, run `ANALYZE` manually. A wrong estimate is the root cause of a surprising number of "the query got slow overnight" incidents.

---

## 7. Implicit Type Mismatch in a JOIN

```sql
-- customer_id is BIGINT in orders, but the join key came in as TEXT
SELECT o.* FROM orders o
JOIN customers c ON o.customer_id::text = c.id::text
WHERE c.country = 'SG';
```

Casting both sides to `text` throws away the ability to use integer indexes and forces a hash or nested loop over converted values. Keep join keys in their native type and matching types on both sides. This also shows up when an ORM maps a numeric FK to a string.

---

## 8. OR Conditions That Defeat Indexes

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE status = 'PENDING' OR customer_id = 42;
```

An `OR` across two different columns often can't be served by a single index cleanly, and the planner may fall back to a scan. Two options.

Rewrite as a `UNION` of two index-friendly queries:

```sql
SELECT * FROM orders WHERE status = 'PENDING'
UNION
SELECT * FROM orders WHERE customer_id = 42;
```

Or rely on a **bitmap OR**, which PostgreSQL can do if both columns are indexed:

```sql
CREATE INDEX idx_orders_status ON orders (status);
CREATE INDEX idx_orders_customer ON orders (customer_id);
```

```
BitmapOr
  ->  Bitmap Index Scan on idx_orders_status
  ->  Bitmap Index Scan on idx_orders_customer
```

Check the plan. If you see a `Seq Scan` for an `OR`, restructure the query.

---

## 9. SELECT * When You Need Three Columns

`SELECT *` forces the database to read wide rows off the heap even when a **covering index** could have answered the query from the index alone.

```sql
-- Covering index: filter columns + returned columns
CREATE INDEX idx_orders_cover
    ON orders (status, created_at) INCLUDE (total_cents);

EXPLAIN (ANALYZE, BUFFERS)
SELECT total_cents FROM orders
WHERE status = 'PAID' AND created_at >= now() - interval '30 days';
```

```
Index Only Scan using idx_orders_cover on orders (actual time=0.03..8.9 rows=...)
  Heap Fetches: 0
```

`Index Only Scan` with `Heap Fetches: 0` means the query never touched the table heap. `SELECT *` would have forced a heap fetch for every row. Select only the columns you need — it reduces I/O, network transfer, and memory.

**MySQL note:** With InnoDB's clustered index, secondary indexes implicitly "cover" the primary key. Add the extra columns to the secondary index to get covering behavior.

---

## 10. Pagination with Large OFFSET

```sql
-- Page 5000, 20 rows per page
EXPLAIN ANALYZE
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 100000;
```

```
Limit  (actual time=95.411..95.430 rows=20 loops=1)
  ->  Index Scan using orders_pkey on orders (actual rows=100020 loops=1)
```

`OFFSET 100000` means the database reads and discards 100,000 rows before returning 20. The deeper the page, the slower it gets. Use **keyset pagination** (seek method) instead:

```sql
-- First page
SELECT * FROM orders ORDER BY id LIMIT 20;

-- Next page: remember the last id you saw (e.g., 100020)
SELECT * FROM orders WHERE id > 100020 ORDER BY id LIMIT 20;
```

```
Index Scan using orders_pkey on orders (actual rows=20 loops=1)
  Index Cond: (id > 100020)
```

Constant time regardless of how deep you page. The tradeoff is you can't jump to an arbitrary page number, but that's rarely needed in real UIs (infinite scroll, "load more").

---

## 11. Lock Contention and Blocking

Sometimes the query itself is fast — it's *waiting*. A long-running transaction holds a lock, and everyone else queues.

```sql
-- Who is blocking whom
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

Common culprits: an `UPDATE` inside a transaction that stays open while the app does slow work, or a migration taking `ACCESS EXCLUSIVE` on a hot table. Fixes:

- Keep transactions short; do I/O and business logic *outside* the transaction.
- Set `lock_timeout` and `statement_timeout` so runaway statements fail fast.
- For DDL, use `ADD COLUMN ... DEFAULT` (fast, metadata-only in PG 11+) and `CREATE INDEX CONCURRENTLY` to avoid blocking writes.

```sql
SET lock_timeout = '3s';
SET statement_timeout = '30s';
```

If latency spikes correlate with specific times and the CPU is idle, suspect locking before you touch the query.

---

## 12. Table and Index Bloat

PostgreSQL uses MVCC: `UPDATE` and `DELETE` leave dead tuples behind until vacuum reclaims them. On a heavily churned table, the heap fills with dead rows, and even indexed scans read more pages than they should.

```sql
-- Rough bloat check
SELECT relname,
       n_live_tup,
       n_dead_tup,
       round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

If `dead_pct` is high (say, over 20%) and `last_autovacuum` is stale, autovacuum is falling behind:

```sql
-- Reclaim space without an exclusive lock
VACUUM (ANALYZE) orders;

-- Rebuild a bloated index online
REINDEX INDEX CONCURRENTLY idx_orders_status_created;

-- Make autovacuum more aggressive on a hot table
ALTER TABLE orders SET (autovacuum_vacuum_scale_factor = 0.02,
                        autovacuum_analyze_scale_factor = 0.01);
```

Bloat is the classic "it was fast last month, now it's slow and nothing changed" cause. Nothing changed in your code — the physical layout degraded.

---

## A Diagnostic Flow You Can Actually Follow

When a query is slow, I work through this in order:

1. **Run `EXPLAIN (ANALYZE, BUFFERS)`.** Look at actual time and where it accumulates.
2. **Estimated vs actual rows.** Big gap? Suspect stale statistics (#6). Run `ANALYZE`.
3. **Seq Scan on a big table with a selective filter?** Missing or unusable index (#1, #2, #3, #4).
4. **`Sort Method: external merge Disk:`?** Sort spill (#5). Add an ordering index or raise `work_mem`.
5. **`Heap Fetches` high on an otherwise indexed query?** Consider a covering index (#9).
6. **Fast plan but slow wall-clock?** Locking (#11) or the app fetching too much data over the wire.
7. **Was fine, now slow, no code change?** Bloat (#12) or statistics drift.

Find these views and keep them handy:

```sql
-- Slowest queries by total time (requires pg_stat_statements)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT round(total_exec_time::numeric, 1) AS total_ms,
       calls,
       round(mean_exec_time::numeric, 2) AS mean_ms,
       query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

That top-20 list is where the wins are. Optimize the query that runs 2 million times a day, not the one that runs once at midnight.

---

## Final Thought

Slow SQL almost always comes down to reading more data than necessary, sorting more than necessary, or waiting on something. `EXPLAIN (ANALYZE, BUFFERS)` tells you which. Learn to read it, keep `pg_stat_statements` running, and check `ANALYZE` freshness before you blame the query.

The fastest query is the one that reads the fewest rows to produce its result. Everything in this post is a variation on that single idea.
