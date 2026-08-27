---
title: "How to Read a Query Execution Plan (EXPLAIN ANALYZE Deep Dive)"
date: 2026-08-25
categories: [SQL, Performance]
tags: [sql, postgresql, performance, explain, query-tuning, tutorial]
description: "Learn to read a PostgreSQL execution plan top-to-bottom and inside-out — node types, cost vs actual time, rows and loops, buffers, and the red flags."
---
## The Plan Is the Ground Truth

When a query is slow, opinions are cheap and the execution plan is free. `EXPLAIN` shows you the strategy the planner chose; `EXPLAIN ANALYZE` runs it and shows you what actually happened. Everything about tuning starts here.

The trick most people never learn: **a plan is a tree, and you read it inside-out and bottom-up.** The most indented node runs first; its output feeds its parent. Timings in the parent *include* the children. Once that clicks, plans stop being intimidating.

Set up data we can run real plans against:

```sql
CREATE TABLE customers (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    country TEXT NOT NULL
);

CREATE TABLE orders (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id  BIGINT NOT NULL REFERENCES customers(id),
    status       TEXT NOT NULL,
    total_cents  BIGINT NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO customers (country)
SELECT (ARRAY['SG','IN','US','DE','JP'])[1 + (g % 5)]
FROM generate_series(1, 200000) g;

INSERT INTO orders (customer_id, status, total_cents, created_at)
SELECT 1 + (random() * 199999)::bigint,
       (ARRAY['PENDING','PAID','SHIPPED','CANCELLED'])[1 + (random() * 3)::int],
       (random() * 500000)::bigint,
       now() - (random() * interval '365 days')
FROM generate_series(1, 3000000) g;

ANALYZE customers;
ANALYZE orders;
```

---

## EXPLAIN vs EXPLAIN ANALYZE — and Always Use BUFFERS

```sql
-- Estimates only. Does NOT run the query. Safe on anything.
EXPLAIN SELECT * FROM orders WHERE status = 'PAID';

-- Actually runs the query and reports real timing + row counts.
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'PAID';

-- The one you should default to:
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT * FROM orders WHERE status = 'PAID';
```

- `ANALYZE` — runs the statement. **Careful:** for `INSERT`/`UPDATE`/`DELETE` it will modify data. Wrap those in a transaction and roll back:

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS) DELETE FROM orders WHERE status = 'CANCELLED';
ROLLBACK;
```

- `BUFFERS` — shows pages read from cache (`shared hit`) vs disk (`read`). This separates "slow because of I/O" from "slow because of CPU."
- `VERBOSE` — adds output columns and schema-qualified names.
- `FORMAT JSON` — machine-readable, great for tooling.

---

## Anatomy of a Single Node

Take the simplest possible plan:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE id = 12345;
```

```
Index Scan using orders_pkey on orders
  (cost=0.43..8.45 rows=1 width=45)
  (actual time=0.028..0.029 rows=1 loops=1)
  Index Cond: (id = 12345)
  Buffers: shared hit=4
Planning Time: 0.071 ms
Execution Time: 0.052 ms
```

Decode every number:

- **`Index Scan using orders_pkey`** — the node type and which index.
- **`cost=0.43..8.45`** — planner's estimate in arbitrary units. First number is *startup cost* (work before the first row), second is *total cost* (work to return all rows). Costs are for comparing plans, not real time.
- **`rows=1`** — estimated rows this node will emit.
- **`width=45`** — estimated average bytes per row.
- **`actual time=0.028..0.029`** — real milliseconds: time to first row .. time to last row.
- **`rows=1 loops=1`** — actual rows emitted, and how many times this node ran.
- **`Buffers: shared hit=4`** — 4 pages found in cache, 0 read from disk.
- **`Planning Time` / `Execution Time`** — planning the query vs running it.

The single most useful comparison on any node: **estimated `rows` vs actual `rows`**. If they diverge by 10x or more, the planner is working with bad information and probably picked a bad plan.

---

## The Multiplication Trap — `actual time` and `loops`

This is the number one misreading of plans. On a nested node, **`actual time` is per-loop, and total time for that node is `actual time × loops`.**

```sql
EXPLAIN ANALYZE
SELECT c.country, o.total_cents
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE c.country = 'JP';
```

```
Nested Loop  (actual time=0.045..812.339 rows=598210 loops=1)
  ->  Seq Scan on customers c  (actual time=0.012..28.114 rows=40000 loops=1)
        Filter: (country = 'JP'::text)
  ->  Index Scan using idx_orders_customer on orders o
        (actual time=0.008..0.017 rows=15 loops=40000)
        Index Cond: (customer_id = c.id)
```

Look at the inner `Index Scan`: `actual time=0.017` looks instant. But `loops=40000` — it ran forty thousand times. Real cost ≈ `0.017 ms × 40000 = ~680 ms`. That inner index scan, not the seq scan, is where the time went. Always multiply by `loops` before deciding a node is cheap.

---

## Scan Node Types — What Each One Means

**Seq Scan** — read the whole table, row by row. Good for small tables or when returning most rows. Bad when a selective index exists.

```
Seq Scan on orders  (actual time=0.014..430.2 rows=750000 loops=1)
  Filter: (status = 'PAID'::text)
  Rows Removed by Filter: 2250000
```

`Rows Removed by Filter` counts rows read then thrown away. A big number here on a big table screams "missing index."

**Index Scan** — walk the index, then fetch each matching row from the heap. Great for selective predicates.

**Index Only Scan** — answer entirely from the index, no heap fetch. The best case. Watch `Heap Fetches` — it should be low or zero.

```
Index Only Scan using idx_orders_cover on orders
  Heap Fetches: 0
```

**Bitmap Heap Scan / Bitmap Index Scan** — a two-phase middle ground. The bitmap index scan builds a bitmap of matching page locations; the bitmap heap scan reads those pages in physical order (sequential I/O instead of random). PostgreSQL picks this when a predicate matches too many rows for a plain index scan but too few for a full seq scan.

```
Bitmap Heap Scan on orders  (actual time=45.1..210.4 rows=300000 loops=1)
  Recheck Cond: (status = 'PENDING'::text)
  Heap Blocks: exact=28431
  ->  Bitmap Index Scan on idx_orders_status  (actual time=40.2..40.2 rows=300000 loops=1)
        Index Cond: (status = 'PENDING'::text)
```

---

## Join Node Types — And When Each Is Right

**Nested Loop** — for each row from the outer input, probe the inner input. Efficient only when the outer side is small and the inner side has an index. Deadly when the outer side is large (that multiplication trap).

**Hash Join** — build a hash table from the smaller side, then stream the larger side against it. The workhorse for joining two large unindexed sets. Watch for the hash spilling to disk.

```
Hash Join  (actual time=52.1..690.4 rows=598210 loops=1)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (actual time=0.01..210.3 rows=3000000 loops=1)
  ->  Hash  (actual time=51.9..51.9 rows=40000 loops=1)
        Buckets: 65536  Batches: 1  Memory Usage: 2560kB
        ->  Seq Scan on customers c  (actual time=0.01..30.2 rows=40000 loops=1)
```

`Batches: 1` means the hash fit in memory. `Batches: 4` (or more) means it spilled — raise `work_mem` or reduce the build side.

**Merge Join** — both inputs sorted on the join key, then merged in one pass. Efficient for large, pre-sorted or index-ordered inputs.

The planner chooses based on cost estimates. If it picks a Nested Loop over millions of rows because it *thinks* the outer side is small (but it's actually huge), you have a statistics problem — fix estimates, don't fight the planner with hints.

---

## Aggregation, Sort, and the Disk-Spill Red Flag

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, count(*), sum(total_cents)
FROM orders
GROUP BY customer_id
ORDER BY sum(total_cents) DESC
LIMIT 10;
```

```
Limit  (actual time=980.2..980.2 rows=10 loops=1)
  ->  Sort  (actual time=980.2..980.2 rows=10 loops=1)
        Sort Key: (sum(total_cents)) DESC
        Sort Method: top-N heapsort  Memory: 27kB
        ->  HashAggregate  (actual time=812.4..905.1 rows=200000 loops=1)
              Group Key: customer_id
              Batches: 1  Memory Usage: 24593kB
              ->  Seq Scan on orders (actual time=0.01..210.4 rows=3000000 loops=1)
```

Read the `Sort Method`:

- **`quicksort Memory: ...`** — sorted in RAM. Fine.
- **`top-N heapsort Memory: ...`** — optimized `ORDER BY ... LIMIT`. Fine, often great.
- **`external merge Disk: ...kB`** — spilled to disk. **Red flag.** Raise `work_mem` or add an index that provides the order.

For grouping, `HashAggregate` with `Batches: 1` is in-memory and good. `Batches: 8` means the hash aggregate spilled (PostgreSQL 13+). Same remedy: more `work_mem` or fewer groups.

---

## Reading a Full Plan, Step by Step

Here's a realistic multi-node plan. We read it inside-out.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.country, count(*) AS orders, sum(o.total_cents) AS revenue
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.created_at >= now() - interval '30 days'
GROUP BY c.country
ORDER BY revenue DESC;
```

```
Sort  (actual time=1120.4..1120.4 rows=5 loops=1)                          [6]
  Sort Key: (sum(o.total_cents)) DESC
  Sort Method: quicksort  Memory: 25kB
  ->  HashAggregate  (actual time=1120.2..1120.3 rows=5 loops=1)           [5]
        Group Key: c.country
        ->  Hash Join  (actual time=210.1..980.6 rows=246000 loops=1)     [4]
              Hash Cond: (o.customer_id = c.id)
              ->  Bitmap Heap Scan on orders o                            [2]
                    (actual time=52.3..610.2 rows=246000 loops=1)
                    Recheck Cond: (created_at >= now() - interval '30 days')
                    ->  Bitmap Index Scan on idx_orders_created           [1]
                          (actual time=48.1..48.1 rows=246000 loops=1)
              ->  Hash  (actual time=98.2..98.2 rows=200000 loops=1)      [3]
                    ->  Seq Scan on customers c (rows=200000 loops=1)
Planning Time: 0.32 ms
Execution Time: 1121.0 ms
```

Reading order (most indented first):

1. **[1] Bitmap Index Scan** on `idx_orders_created` finds the recent orders' page locations.
2. **[2] Bitmap Heap Scan** reads those pages → 246k recent orders.
3. **[3] Hash** builds a hash table over all 200k customers.
4. **[4] Hash Join** matches recent orders to customers → 246k joined rows.
5. **[5] HashAggregate** groups by country → 5 rows.
6. **[6] Sort** orders the 5 rows by revenue.

Where's the time? The join produces its last row at 980ms, and the heap scan runs to 610ms. The bulk of the cost is reading 246k recent orders and joining. If this were too slow, a covering index on `(created_at) INCLUDE (customer_id, total_cents)` could turn steps 1–2 into an index-only scan and drop the heap reads.

---

## The Red Flags Checklist

When scanning a plan, these are the patterns that cost you:

- **Estimated rows ≠ actual rows (>10x off).** Stale statistics. Run `ANALYZE`; raise the column's statistics target; add extended statistics for correlated columns.
- **`Seq Scan` + large `Rows Removed by Filter` on a big table.** Missing or unusable index.
- **`Nested Loop` with large `loops` on the inner node.** The multiplication trap — usually a bad row estimate made the planner think the outer side was tiny.
- **`Sort Method: external merge Disk:`** or **`HashAggregate ... Batches: >1`.** Spilled to disk. Raise `work_mem` or provide an ordering index.
- **`Hash ... Batches: >1`.** Hash join spilled. Same remedy.
- **`Buffers: ... read=` large.** Heavy disk I/O — data doesn't fit in cache, or the query touches far more pages than it should.
- **`Heap Fetches` high on an Index Only Scan.** Visibility map stale — `VACUUM` the table.
- **`Rows Removed by Join Filter` large.** The join condition is matching too much before filtering.

---

## Turning a Plan Into a Fix

The workflow that turns reading into results:

1. Get the plan: `EXPLAIN (ANALYZE, BUFFERS)`.
2. Find the node with the largest *total* time (remember `actual time × loops`).
3. Ask *why* it's expensive: too many rows scanned? bad estimate? disk spill? random I/O?
4. Apply the matching remedy: index, rewrite, `ANALYZE`, `work_mem`, or covering index.
5. Re-run the plan and confirm the expensive node changed shape.

```sql
-- Before: Seq Scan removing millions of rows
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'CANCELLED' AND created_at > now() - interval '7 days';

CREATE INDEX idx_orders_status_created ON orders (status, created_at);
ANALYZE orders;

-- After: Index Scan touching only the matching slice
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'CANCELLED' AND created_at > now() - interval '7 days';
```

Confirm the `Seq Scan` became an `Index Scan`, `Rows Removed by Filter` dropped to near zero, and `Buffers` fell. That before/after is the proof your change worked — not the wall-clock alone, which varies with cache warmth.

---

## MySQL Notes

MySQL's equivalents:

```sql
-- Estimated plan
EXPLAIN SELECT * FROM orders WHERE status = 'PAID';

-- Actual execution (MySQL 8.0.18+), like PostgreSQL's EXPLAIN ANALYZE
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'PAID';

-- Rich JSON plan with cost details
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE status = 'PAID';
```

Key column mappings in classic `EXPLAIN`:

- **`type`** — access method: `ALL` (full scan, worst), `index`, `range`, `ref`, `eq_ref`, `const` (best).
- **`rows`** — estimated rows examined.
- **`filtered`** — percentage of rows kept after the `WHERE` (low = wasteful).
- **`Extra`** — `Using index` (covering, good), `Using filesort` (sort, watch it), `Using temporary` (temp table, watch it), `Using where`.

`Using filesort` and `Using temporary` together on a big query are MySQL's version of the disk-spill red flag.

---

## Final Thought

An execution plan is not something to skim — it's a precise, honest report of what the database did. Read it inside-out, multiply `actual time` by `loops`, compare estimated to actual rows, and watch for disk spills. Do that consistently and query tuning stops being guesswork and becomes a short list of mechanical checks.

The plan already tells you what's wrong. You just have to learn its language.
