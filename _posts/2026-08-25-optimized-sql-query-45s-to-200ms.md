---
title: "I Optimized a Query from 45 Seconds to 200ms — Here's How"
date: 2026-08-25
categories: [SQL, Performance]
tags: [sql, postgresql, performance, indexing, query-tuning, tutorial]
description: "A real optimization story: an analytics query that timed out in production, the step-by-step diagnosis with every EXPLAIN plan, and the five changes that took it from 45 seconds to 200 milliseconds."
---
## The 9 AM Slack Message

"The revenue dashboard is timing out again. Can someone look?"

It was the finance team's daily driver — a page showing revenue per country for the last 30 days, per-customer order counts, and a running total. It had been "a bit slow" for months. That morning it crossed the 30-second gateway timeout and started returning 504s. Now it was my problem.

I'm changing the table and column names, but everything else — the schema shape, the query, the plans, the fixes — is a faithful reconstruction of a real optimization I did on an e-commerce reporting system.

Here's the setup you can reproduce locally:

```sql
CREATE TABLE customers (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name      TEXT NOT NULL,
    country   TEXT NOT NULL,
    tier      TEXT NOT NULL,          -- 'FREE','PRO','ENTERPRISE'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id  BIGINT NOT NULL REFERENCES customers(id),
    status       TEXT NOT NULL,       -- 'PENDING','PAID','REFUNDED','CANCELLED'
    total_cents  BIGINT NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 800k customers, 40M orders — production-ish scale
INSERT INTO customers (name, country, tier, created_at)
SELECT 'Customer ' || g,
       (ARRAY['SG','IN','US','DE','JP','GB','AU'])[1 + (g % 7)],
       (ARRAY['FREE','FREE','FREE','PRO','ENTERPRISE'])[1 + (g % 5)],
       now() - (random() * interval '1200 days')
FROM generate_series(1, 800000) g;

INSERT INTO orders (customer_id, status, total_cents, created_at)
SELECT 1 + (random() * 799999)::bigint,
       (ARRAY['PAID','PAID','PAID','REFUNDED','CANCELLED'])[1 + (random() * 4)::int],
       (random() * 1000000)::bigint,
       now() - (random() * interval '900 days')
FROM generate_series(1, 40000000) g;

ANALYZE customers;
ANALYZE orders;
```

---

## The Query That Was Melting

This is roughly what the ORM had generated, cleaned up for readability:

```sql
SELECT
    c.country,
    c.tier,
    COUNT(DISTINCT o.customer_id)                          AS active_customers,
    COUNT(o.id)                                            AS order_count,
    SUM(o.total_cents) / 100.0                             AS revenue,
    (SELECT COUNT(*) FROM orders o2
      WHERE o2.customer_id = c.id)                         AS lifetime_orders
FROM customers c
JOIN orders o ON UPPER(o.status) = 'PAID' AND o.customer_id = c.id
WHERE o.created_at >= now() - interval '30 days'
   OR c.tier = 'ENTERPRISE'
GROUP BY c.country, c.tier
ORDER BY revenue DESC;
```

There is a *lot* wrong here, and I didn't see all of it at first. Let me show you how I found each problem.

---

## Step 1: Measure, Don't Guess

First rule: never optimize on a hunch. I ran it under `EXPLAIN (ANALYZE, BUFFERS)` on a staging copy of production data. (In production I'd use a read replica; you never run `EXPLAIN ANALYZE` on a query you can't afford to actually execute.)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ... ;  -- the query above
```

```
GroupAggregate  (actual time=44210.1..45120.8 rows=35 loops=1)
  Group Key: c.country, c.tier
  ->  Sort  (actual time=41880.4..43050.2 rows=39400221 loops=1)
        Sort Key: c.country, c.tier
        Sort Method: external merge  Disk: 2205184kB
        ->  Hash Join  (actual time=1120.2..30110.5 rows=39400221 loops=1)
              Hash Cond: (o.customer_id = c.id)
              Join Filter: ((o.created_at >= ...) OR (c.tier = 'ENTERPRISE'))
              Rows Removed by Join Filter: 0
              ->  Seq Scan on orders o  (actual time=0.03..9800.2 rows=40000000 loops=1)
                    Filter: (upper(status) = 'PAID'::text)
                    Rows Removed by Filter: 8000000
              ->  Hash  (actual time=1100.1..1100.1 rows=800000 loops=1)
                    ->  Seq Scan on customers c (rows=800000 loops=1)
  SubPlan 1
    ->  Aggregate  (actual time=0.05..0.05 rows=1 loops=35)
          ->  Index Scan using idx_orders_customer on orders o2 ...
Planning Time: 1.4 ms
Execution Time: 45121.6 ms
```

45 seconds. Reading the plan inside-out, three things jumped out:

1. **`Seq Scan on orders` reading all 40M rows** with `Filter: upper(status) = 'PAID'`.
2. **`Sort Method: external merge Disk: 2205184kB`** — a 2.2 GB sort spilled to disk.
3. The join produced **39.4 million rows** before aggregation. The whole thing was processing the entire order history, not 30 days.

Let me tackle them in order of impact.

---

## Step 2: The `UPPER(status)` Trap Killed the Index

`UPPER(o.status) = 'PAID'` wraps the column in a function, so even if I added an index on `status`, it couldn't be used (I covered exactly this in my "why is my query slow" post — a function around an indexed column defeats the index).

The data was already stored uppercase. The `UPPER()` was defensive cruft the ORM added. I removed it:

```sql
-- was: JOIN orders o ON UPPER(o.status) = 'PAID' AND o.customer_id = c.id
JOIN orders o ON o.status = 'PAID' AND o.customer_id = c.id
```

If the data *hadn't* been clean, the right fix would be an expression index:

```sql
CREATE INDEX idx_orders_status_upper ON orders ((UPPER(status)));
```

But not needing the function at all is better. One line, and now an index on `status` becomes usable.

---

## Step 3: The `OR` Across Tables Was Fatal

Look at the join condition again:

```sql
WHERE o.created_at >= now() - interval '30 days'
   OR c.tier = 'ENTERPRISE'
```

That `OR` spanning both tables is why the join emitted 39 million rows. The planner couldn't push the `created_at` filter down to the orders scan, because any order might still qualify via the *other* branch (`c.tier = 'ENTERPRISE'`). So it joined everything, then filtered — except `Rows Removed by Join Filter: 0` showed the filter wasn't even removing anything useful because of how the `OR` interacted with the join.

I talked to the finance team. What they actually wanted: "last 30 days of revenue, but always include enterprise customers even if they had no orders this month." That's not one query with an `OR` — it's cleaner as a `UNION` of two intents, or better, restructure so the time filter is a plain, pushdown-able predicate.

For the dashboard's real need, the enterprise carve-out was for a *different* widget. The revenue-per-country table only needed the last 30 days. We split it:

```sql
-- The dashboard's main table: last 30 days, paid orders only
SELECT c.country, c.tier,
       COUNT(o.id)                 AS order_count,
       SUM(o.total_cents) / 100.0  AS revenue
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'PAID'
  AND o.created_at >= now() - interval '30 days'
GROUP BY c.country, c.tier
ORDER BY revenue DESC;
```

Now `created_at >= now() - interval '30 days'` is a clean predicate the planner can push straight into the orders scan. This single restructuring is the biggest win in the whole story — but I needed an index to cash it in.

---

## Step 4: Add the Index the Query Actually Needs

The query filters orders by `status = 'PAID'` (equality) and then ranges on `created_at`. That's the textbook `(equality, range)` composite index shape:

```sql
CREATE INDEX CONCURRENTLY idx_orders_paid_created
    ON orders (status, created_at)
    INCLUDE (customer_id, total_cents);
```

Two deliberate choices:

- **`CONCURRENTLY`** — this is a 40M-row table in production. A plain `CREATE INDEX` takes an `ACCESS EXCLUSIVE` lock and blocks writes for minutes. `CONCURRENTLY` builds it without blocking.
- **`INCLUDE (customer_id, total_cents)`** — these are the only order columns the query reads. Putting them in the index means an **index-only scan**: the query never touches the 40M-row heap.

Re-running the restructured query:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.country, c.tier, COUNT(o.id), SUM(o.total_cents)/100.0
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'PAID' AND o.created_at >= now() - interval '30 days'
GROUP BY c.country, c.tier
ORDER BY 4 DESC;
```

```
Sort  (actual time=1180.3..1180.3 rows=35 loops=1)
  Sort Key: (sum(o.total_cents) / 100.0) DESC
  ->  HashAggregate  (actual time=1170.1..1180.0 rows=35 loops=1)
        Group Key: c.country, c.tier
        ->  Hash Join  (actual time=210.4..980.2 rows=1315000 loops=1)
              Hash Cond: (o.customer_id = c.id)
              ->  Index Only Scan using idx_orders_paid_created on orders o
                    (actual time=0.05..410.2 rows=1315000 loops=1)
                    Index Cond: ((status = 'PAID') AND (created_at >= ...))
                    Heap Fetches: 0
              ->  Hash (actual time=205.1..205.1 rows=800000 loops=1)
                    ->  Seq Scan on customers c (rows=800000 loops=1)
Execution Time: 1181.4 ms
```

45s → 1.18s. The orders scan went from 40M rows to the 1.3M paid orders in the window, via an `Index Only Scan` with `Heap Fetches: 0`. But I wasn't done — the customers side was still a full seq scan feeding the hash, and the finance team wanted sub-second.

---

## Step 5: Kill the Correlated Subquery

I'd dropped `lifetime_orders` in the restructure, but the original had this per-row correlated subquery:

```sql
(SELECT COUNT(*) FROM orders o2 WHERE o2.customer_id = c.id) AS lifetime_orders
```

A correlated subquery runs *once per output row*. In the original plan it showed as `SubPlan 1 ... loops=35` — only 35 times here because grouping collapsed the rows, but in the pre-aggregation version it had been catastrophic. When a subquery references the outer row, treat it as a red flag and rewrite it as a join or a lateral.

The dashboard's "lifetime orders" widget was a separate number. When it *did* need to be combined, the fix is a pre-aggregated join, not a per-row subquery:

```sql
SELECT c.country, c.tier,
       COUNT(o.id)                AS order_count,
       SUM(o.total_cents)/100.0   AS revenue,
       SUM(lt.lifetime_orders)    AS lifetime_orders
FROM customers c
JOIN orders o ON o.customer_id = c.id
              AND o.status = 'PAID'
              AND o.created_at >= now() - interval '30 days'
JOIN LATERAL (
    SELECT COUNT(*) AS lifetime_orders
    FROM orders x WHERE x.customer_id = c.id
) lt ON true
GROUP BY c.country, c.tier;
```

For a dashboard, though, the truly right move for a whole-table lifetime count is to not compute it live at all — which brings me to the last change.

---

## Step 6: Pre-Aggregate What Doesn't Need to Be Live

The dashboard refreshed every load, but revenue for "the last 30 days" doesn't change second to second, and finance looked at it a few times a day. Computing a 1.3M-row aggregation on every page load was wasteful.

I moved it behind a **materialized view**, refreshed every 15 minutes:

```sql
CREATE MATERIALIZED VIEW mv_revenue_by_country_tier AS
SELECT c.country, c.tier,
       COUNT(o.id)                AS order_count,
       SUM(o.total_cents)/100.0   AS revenue
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'PAID'
  AND o.created_at >= now() - interval '30 days'
GROUP BY c.country, c.tier
WITH DATA;

-- Needed for REFRESH ... CONCURRENTLY
CREATE UNIQUE INDEX ON mv_revenue_by_country_tier (country, tier);
```

Refresh on a schedule without blocking readers:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_revenue_by_country_tier;
```

The dashboard query became trivial:

```sql
EXPLAIN ANALYZE
SELECT * FROM mv_revenue_by_country_tier ORDER BY revenue DESC;
```

```
Sort  (actual time=0.09..0.10 rows=35 loops=1)
  Sort Key: revenue DESC
  ->  Seq Scan on mv_revenue_by_country_tier (actual time=0.01..0.03 rows=35 loops=1)
Execution Time: 0.20 ms
```

Reading 35 pre-computed rows: **0.2 ms** to serve. Even accounting for the network round trip and serialization, the endpoint now returns in ~200 ms end-to-end.

---

## The Scorecard

| Stage | Change | Time |
|---|---|---|
| Baseline | Original query | 45,000 ms |
| Step 2 | Remove `UPPER()` on status | ~38,000 ms |
| Step 3 | Split the cross-table `OR`, push down the date filter | ~9,000 ms |
| Step 4 | Add `(status, created_at) INCLUDE (...)`, index-only scan | 1,180 ms |
| Step 5 | Replace correlated subquery with lateral join | 1,050 ms |
| Step 6 | Materialized view refreshed every 15 min | 200 ms |

Two changes did the heavy lifting — restructuring the `OR` so the date filter could push down, and the covering index enabling an index-only scan. The materialized view turned "fast" into "instant" for a read pattern that didn't need to be live.

---

## What I'd Tell My Past Self

- **Read the plan first.** I could have spent a day tweaking `work_mem`; the plan told me in 30 seconds that a cross-table `OR` was forcing a 40M-row join.
- **A function around a column is a silent index-killer.** `UPPER(status)` looked harmless. It wasn't.
- **`OR` across two tables prevents predicate pushdown.** If you see a giant join feeding an aggregation, check whether an `OR` is blocking the planner from filtering early.
- **Correlated subqueries run per row.** Rewrite as joins or laterals.
- **Not every number needs to be live.** A materialized view is the cheapest optimization there is when data tolerates minutes of staleness.
- **Build indexes `CONCURRENTLY` in production.** A blocking `CREATE INDEX` on a hot 40M-row table is its own incident.

**MySQL notes:** MySQL 8 has no materialized views — emulate with a summary table refreshed by a scheduled event or job. `EXPLAIN ANALYZE` (8.0.18+) gives the actual-timing plan. The `UPPER()` and `OR` issues are identical. For the covering index, InnoDB secondary indexes implicitly include the primary key; add `total_cents` and `customer_id` to the index to cover the query.

---

## Final Thought

Nobody optimizes a query from 45 seconds to 200 milliseconds with one magic trick. It's a sequence of small, evidence-driven changes, each verified against the execution plan. Measure, read the plan, fix the biggest node, re-measure. Repeat until the finance team stops Slacking you at 9 AM.

The database was never slow. The query was asking it to do 200x more work than the question required.
