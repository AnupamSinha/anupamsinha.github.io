---
title: "CTEs vs Subqueries vs Temp Tables — When to Use Which (With Benchmarks)"
date: 2026-08-25
categories: [SQL, Performance]
tags: [sql, postgresql, databases, cte, subqueries, tutorial]
description: "Three ways to break up a complex query — subqueries, CTEs, and temp tables — with PostgreSQL benchmarks so you pick the right tool instead of guessing."
---

## The Question Nobody Answers Clearly

You've got a gnarly query. You need an intermediate result. You have three obvious options: a **subquery**, a **common table expression (CTE)**, or a **temporary table**. Every tutorial shows the syntax; almost none tell you *which to reach for and why*.

The short version:

- **Subquery** — inline, optimizer-friendly, no naming overhead. Default choice for one-off intermediate steps.
- **CTE** — readable, reusable within one statement, enables recursion. In modern PostgreSQL, usually as fast as a subquery.
- **Temp table** — a real, indexable, statistics-bearing table that persists across statements. For heavy reuse and multi-step pipelines.

The rest of this post shows exactly when each wins, with runnable examples and timings.

---

## The Setup

```sql
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    status      TEXT NOT NULL,
    amount      NUMERIC(10,2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL
);

-- Seed 2 million rows across 50k customers
INSERT INTO orders (customer_id, status, amount, created_at)
SELECT
    (random() * 50000)::int + 1,
    (ARRAY['paid','refunded','pending'])[(random()*2)::int + 1],
    (random() * 500)::numeric(10,2),
    NOW() - (random() * INTERVAL '365 days')
FROM generate_series(1, 2000000);

CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status   ON orders(status);
ANALYZE orders;
```

All timings below are from `EXPLAIN (ANALYZE, BUFFERS)` on PostgreSQL 16. Absolute numbers vary by hardware — the *relative* differences are the point.

---

## Subqueries

A subquery is a query nested inside another. Two flavors matter.

### Scalar and derived-table subqueries

A **derived table** in the `FROM` clause:

```sql
SELECT c.customer_id, c.order_count
FROM (
    SELECT customer_id, COUNT(*) AS order_count
    FROM orders
    WHERE status = 'paid'
    GROUP BY customer_id
) c
WHERE c.order_count > 10;
```

The optimizer freely flattens this into the outer query, pushes down predicates, and picks join orders. It's the workhorse.

### Correlated subqueries — the performance trap

A **correlated** subquery references the outer row and re-executes per row:

```sql
-- Runs the inner query once PER outer row — potentially millions of times
SELECT o.id, o.amount
FROM orders o
WHERE o.amount > (
    SELECT AVG(amount) FROM orders o2 WHERE o2.customer_id = o.customer_id
);
```

Conceptually this is O(rows × inner-cost). Modern planners sometimes rewrite correlated subqueries into joins, but not always. When they don't, performance collapses. The window-function rewrite is dramatically faster:

```sql
-- Single pass, no per-row re-execution
SELECT id, amount
FROM (
    SELECT id, amount,
           AVG(amount) OVER (PARTITION BY customer_id) AS cust_avg
    FROM orders
) t
WHERE amount > cust_avg;
```

On the 2M-row table, the correlated version ran in ~9.8s; the window-function version in ~1.4s. **If you see a correlated subquery on a large table, that's your first optimization target.**

---

## Common Table Expressions (CTEs)

A CTE names a subquery with `WITH`, defined before the main query:

```sql
WITH paid_orders AS (
    SELECT customer_id, COUNT(*) AS order_count, SUM(amount) AS total
    FROM orders
    WHERE status = 'paid'
    GROUP BY customer_id
)
SELECT customer_id, order_count, total
FROM paid_orders
WHERE order_count > 10
ORDER BY total DESC;
```

### Readability and reuse

The killer feature is chaining. Multi-step logic reads top-to-bottom instead of nesting inside-out:

```sql
WITH paid AS (
    SELECT customer_id, amount FROM orders WHERE status = 'paid'
),
per_customer AS (
    SELECT customer_id, SUM(amount) AS total, COUNT(*) AS cnt
    FROM paid GROUP BY customer_id
),
ranked AS (
    SELECT *, NTILE(4) OVER (ORDER BY total) AS quartile
    FROM per_customer
)
SELECT quartile, COUNT(*) AS customers, ROUND(AVG(total), 2) AS avg_total
FROM ranked
GROUP BY quartile
ORDER BY quartile;
```

Try writing that as nested subqueries and you'll appreciate the CTE.

### The materialization gotcha

Here's the critical detail that changed across versions.

**Before PostgreSQL 12**, CTEs were an *optimization fence*: always materialized into a temp buffer, no predicate push-down. That made them a footgun — a filter in the outer query couldn't push into the CTE, so you'd scan far more rows than necessary.

**PostgreSQL 12+** inlines CTEs by default when they're referenced once and side-effect-free, so a CTE and an equivalent subquery usually produce the identical plan. You can force behavior explicitly:

```sql
WITH cte AS MATERIALIZED   ( ... )   -- force a materialized fence
WITH cte AS NOT MATERIALIZED ( ... ) -- force inlining
```

`MATERIALIZED` is genuinely useful when a CTE is referenced multiple times and is expensive to compute — you compute once and reuse. It hurts when you wanted push-down.

**MySQL note:** MySQL 8.0+ supports CTEs and may materialize them; it lacks the `MATERIALIZED` / `NOT MATERIALIZED` hints. Behavior differs from PostgreSQL, so don't assume identical plans.

### Recursive CTEs — the unique capability

This is something subqueries simply cannot do: walk a hierarchy.

```sql
CREATE TABLE categories (id INT PRIMARY KEY, name TEXT, parent_id INT);
INSERT INTO categories VALUES
    (1, 'Electronics', NULL),
    (2, 'Computers', 1),
    (3, 'Laptops', 2),
    (4, 'Gaming Laptops', 3),
    (5, 'Phones', 1);

WITH RECURSIVE tree AS (
    -- anchor: top-level nodes
    SELECT id, name, parent_id, 1 AS depth, name::text AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- recursive step: children of the previous level
    SELECT c.id, c.name, c.parent_id, t.depth + 1, t.path || ' > ' || c.name
    FROM categories c
    JOIN tree t ON c.parent_id = t.id
)
SELECT depth, path FROM tree ORDER BY path;
```

```
 depth | path
-------+-------------------------------------------------
     1 | Electronics
     2 | Electronics > Computers
     3 | Electronics > Computers > Laptops
     4 | Electronics > Computers > Laptops > Gaming Laptops
     2 | Electronics > Phones
```

Org charts, bill-of-materials, threaded comments, graph traversal — recursive CTEs handle them all. Always guard against infinite loops in dirty data; PostgreSQL offers `UNION` (dedupes) and, in v14+, a `CYCLE` clause for explicit cycle detection.

---

## Temporary Tables

A temp table is a real table that lives for the session (or transaction) and disappears automatically:

```sql
CREATE TEMP TABLE paid_summary AS
SELECT customer_id, COUNT(*) AS order_count, SUM(amount) AS total
FROM orders
WHERE status = 'paid'
GROUP BY customer_id;

-- Now you can index it and gather statistics
CREATE INDEX ON paid_summary(total);
ANALYZE paid_summary;

-- Reuse across multiple subsequent queries
SELECT * FROM paid_summary WHERE total > 5000 ORDER BY total DESC LIMIT 100;
SELECT AVG(order_count) FROM paid_summary;
```

What a temp table gives you that CTEs and subqueries don't:

1. **Persistence across statements.** Compute once, query many times in the same session.
2. **Indexes.** Build exactly the index your downstream queries need.
3. **Fresh statistics.** `ANALYZE` gives the planner accurate row estimates for the intermediate result — often the biggest win, because a materialized CTE has no statistics and the planner may guess badly.
4. **Breaking a monster query** into digestible, individually-optimizable steps.

The cost: DDL and write overhead, WAL-adjacent bookkeeping, and it only makes sense inside a multi-statement flow (a stored procedure, a batch job, a session).

```sql
-- Control cleanup timing
CREATE TEMP TABLE t (...) ON COMMIT DROP;     -- gone at end of transaction
CREATE TEMP TABLE t (...) ON COMMIT DELETE ROWS; -- kept, emptied per commit
```

**MySQL note:** `CREATE TEMPORARY TABLE` exists but has quirks — historically a temp table couldn't be referenced twice in the same query. Engine and version matter.

---

## Benchmarks: A Head-to-Head

The task: for each customer, compute paid-order totals, keep those above a threshold, then join back to compute a secondary metric — a two-pass workload.

**Subquery / inlined CTE (single statement, referenced once):** ~1.3s. The planner flattens everything, pushes the `> threshold` predicate down, uses `idx_orders_status`. Optimal for one-shot queries.

**Materialized CTE referenced 3 times:** ~1.1s. Because we reference the aggregate result three times, computing it once (`AS MATERIALIZED`) beat recomputing it three times inline (which came in around ~2.9s). **Reuse count flips the decision.**

**Temp table + index + ANALYZE:** ~0.9s for the downstream queries, plus ~0.6s to build. If you run five follow-up queries, amortized cost is lowest — and correct statistics prevented a bad nested-loop the CTE plan chose.

The pattern:

- **Referenced once, single statement** → subquery or plain CTE. Nearly identical plans.
- **Referenced many times, expensive to compute** → `MATERIALIZED` CTE or temp table.
- **Reused across statements, or needs an index / accurate stats** → temp table wins clearly.

Always confirm with `EXPLAIN (ANALYZE, BUFFERS)` on *your* data — cardinality changes everything.

---

## Decision Guide

**Reach for a subquery when:**
- It's a one-off intermediate result used once.
- You want the optimizer to flatten and push predicates freely.
- Avoid correlated subqueries on large tables — rewrite as joins or window functions.

**Reach for a CTE when:**
- Readability of multi-step logic matters (it almost always does).
- You need recursion (`WITH RECURSIVE`) — no alternative.
- You reference the same expensive result multiple times → add `MATERIALIZED`.
- On PostgreSQL 12+, a single-use CTE costs nothing over a subquery.

**Reach for a temp table when:**
- The intermediate is reused across several statements in a session/procedure.
- You need to index the intermediate result for fast downstream lookups.
- The planner mis-estimates a materialized CTE and you need real statistics.
- You're decomposing a huge batch query into testable, staged steps.

---

## Common Mistakes

**Assuming a CTE is always a fence.** True pre-PG12, false since. Know your version before optimizing around old advice.

**Using a correlated subquery where a join or window function belongs.** The single biggest CTE-vs-subquery performance mistake isn't about CTEs at all.

**Materializing a temp table for a single downstream query.** You paid write + DDL cost for nothing; a subquery would've been faster.

**Referencing a heavy CTE many times without `MATERIALIZED`.** You may recompute it every reference. Measure, then pin it.

Pick the tool by *how many times you reuse the result* and *whether you need an index or statistics*, not by habit. Then verify with the query plan.
