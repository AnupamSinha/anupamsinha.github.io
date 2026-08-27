---
title: "GROUP BY, HAVING, and Aggregations — The Complete Mental Model"
date: 2026-08-25
categories: [SQL, Fundamentals]
tags: [sql, postgresql, databases, group-by, aggregation, tutorial]
description: "Build the mental model of how SQL groups, aggregates, and filters — then GROUPING SETS, ROLLUP, FILTER, and DISTINCT aggregates all fall into place."
---

## The One Idea That Explains Everything

`GROUP BY` does exactly one thing: it **collapses many rows into one row per group**. Everything confusing about aggregation — why you can't `SELECT` a non-grouped column, why `WHERE` and `HAVING` are different, why `COUNT(*)` and `COUNT(col)` disagree — follows from that single fact.

After grouping, each group is a single output row. So the only things you can put in the `SELECT` list are: (a) the columns you grouped by (constant within the group), and (b) aggregate functions that reduce the group's many values to one. Ask "does this produce one value per group?" and every rule answers itself.

PostgreSQL syntax throughout; MySQL differences noted.

---

## The Data

```sql
CREATE TABLE sales (
    id       INT PRIMARY KEY,
    region   TEXT,
    product  TEXT,
    channel  TEXT,
    quantity INT,
    amount   NUMERIC(10,2)
);

INSERT INTO sales VALUES
    (1, 'West', 'Widget',  'online',  3, 150.00),
    (2, 'West', 'Widget',  'store',   1,  50.00),
    (3, 'West', 'Gadget',  'online',  2, 200.00),
    (4, 'East', 'Widget',  'online',  5, 250.00),
    (5, 'East', 'Gadget',  'store',   1, 100.00),
    (6, 'East', 'Gadget',  'online',  4, 400.00),
    (7, 'East', 'Widget',  'store',   2, NULL);   -- NULL amount on purpose
```

---

## The Execution Order That Fixes Most Confusion

SQL is written `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ...`, but it *executes* in a different logical order. Memorize this and half your bugs vanish:

```
1. FROM      -- pick the tables
2. WHERE     -- filter individual rows (before grouping)
3. GROUP BY  -- collapse rows into groups
4. HAVING    -- filter whole groups (after grouping)
5. SELECT    -- compute output columns and aggregates
6. ORDER BY  -- sort the final rows
```

Two consequences drop out immediately:

- **`WHERE` filters rows; `HAVING` filters groups.** `WHERE` runs before grouping, so it cannot see aggregates. `HAVING` runs after, so it can.
- **`SELECT` aliases aren't visible in `WHERE`/`GROUP BY`/`HAVING`** in standard SQL, because `SELECT` is computed later. (PostgreSQL allows a `SELECT` alias in `GROUP BY` and `ORDER BY` as an extension, but not in `WHERE`.)

---

## Basic Aggregation

```sql
SELECT
    region,
    COUNT(*)       AS num_sales,
    SUM(quantity)  AS total_qty,
    SUM(amount)    AS revenue,
    AVG(amount)    AS avg_amount,
    MIN(amount)    AS min_amount,
    MAX(amount)    AS max_amount
FROM sales
GROUP BY region;
```

```
 region | num_sales | total_qty | revenue | avg_amount | min_amount | max_amount
--------+-----------+-----------+---------+------------+------------+-----------
 East   |         4 |        12 |  750.00 |   250.0000 |     100.00 |     400.00
 West   |         3 |         6 |  400.00 |   133.3333 |      50.00 |     200.00
```

Look closely at East: 4 sales, but `revenue = 750` and `avg_amount = 250` — computed over only **3** non-NULL amounts. That's the NULL rule of aggregates.

---

## The NULL Rule Every Aggregate Follows

**Aggregate functions ignore NULLs** (except `COUNT(*)`). This is the single most misunderstood behavior in aggregation.

```sql
SELECT
    COUNT(*)        AS count_all_rows,   -- counts rows, NULLs included
    COUNT(amount)   AS count_non_null,   -- counts non-NULL amounts only
    SUM(amount)     AS sum_amount,       -- NULLs skipped
    AVG(amount)     AS avg_amount        -- SUM / COUNT(non-null), NOT / COUNT(*)
FROM sales
WHERE region = 'East';
```

```
 count_all_rows | count_non_null | sum_amount | avg_amount
----------------+----------------+------------+-----------
              4 |              3 |     750.00 |   250.0000
```

The gotcha: `AVG(amount)` divides by the count of **non-NULL** values (3), not total rows (4). If you want NULLs treated as zero, coalesce first:

```sql
-- Treat missing amounts as 0 in the average
SELECT AVG(COALESCE(amount, 0)) AS avg_treating_null_as_zero
FROM sales WHERE region = 'East';   -- 187.50, divides by 4
```

Decide deliberately: is a missing amount "unknown" (exclude it, default behavior) or "zero" (coalesce it)? The two give different answers.

---

## WHERE vs HAVING — Now Obvious

`WHERE` filters rows before grouping. `HAVING` filters groups after. Use each for what it can see.

```sql
-- WHERE: drop rows before grouping (online sales only)
-- HAVING: drop groups after aggregating (regions with revenue > 300)
SELECT region, SUM(amount) AS revenue
FROM sales
WHERE channel = 'online'          -- per-row filter, runs first
GROUP BY region
HAVING SUM(amount) > 300;         -- per-group filter, runs after
```

A frequent inefficiency is filtering in `HAVING` what belongs in `WHERE`:

```sql
-- WASTEFUL: groups everything, then discards non-online groups
SELECT region, SUM(amount) FROM sales
GROUP BY region, channel
HAVING channel = 'online';        -- works, but should be WHERE

-- BETTER: filter rows first, less work for GROUP BY
SELECT region, SUM(amount) FROM sales
WHERE channel = 'online'
GROUP BY region;
```

Rule: **if a condition doesn't involve an aggregate, it belongs in `WHERE`.** Reserve `HAVING` for conditions on aggregate results.

---

## The GROUP BY Column Rule

Every non-aggregated column in `SELECT` must appear in `GROUP BY`. Otherwise the value is ambiguous — which of the group's many `product` values should the single output row show?

```sql
-- ERROR: column "sales.product" must appear in the GROUP BY clause
SELECT region, product, SUM(amount)
FROM sales
GROUP BY region;
```

Fix by grouping at the grain you actually want:

```sql
SELECT region, product, SUM(amount) AS revenue
FROM sales
GROUP BY region, product
ORDER BY region, product;
```

### The functional-dependency shortcut

PostgreSQL is smart about primary keys. If you `GROUP BY` a primary key, you may `SELECT` any other column of that table without listing it, because the PK functionally determines them:

```sql
-- Legal in PostgreSQL: name is functionally dependent on the PK id
SELECT c.id, c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id;   -- c.name allowed because c.id is the PK
```

**MySQL note:** Old MySQL silently allowed *any* bare column in the `SELECT` and returned an arbitrary value from the group — a notorious source of wrong results. Modern MySQL enables `ONLY_FULL_GROUP_BY` by default, aligning it with the standard. Don't rely on the old lax behavior.

---

## COUNT: The Four Variants

```sql
SELECT
    COUNT(*)                  AS all_rows,        -- every row
    COUNT(amount)             AS non_null_amount, -- rows where amount IS NOT NULL
    COUNT(DISTINCT product)   AS distinct_products,
    COUNT(DISTINCT region)    AS distinct_regions
FROM sales;
```

- `COUNT(*)` — counts rows, period. Doesn't care about NULLs.
- `COUNT(col)` — counts rows where `col` is not NULL.
- `COUNT(DISTINCT col)` — counts distinct non-NULL values.

Using `COUNT(col)` when you meant `COUNT(*)` silently undercounts whenever `col` has NULLs. Pick intentionally.

---

## DISTINCT Aggregates and FILTER

`DISTINCT` works inside most aggregates:

```sql
SELECT region,
       SUM(amount)              AS revenue,
       SUM(DISTINCT amount)     AS sum_of_distinct_amounts,
       COUNT(DISTINCT product)  AS distinct_products
FROM sales
GROUP BY region;
```

PostgreSQL's `FILTER` clause is the clean way to do **conditional aggregation** — multiple sub-aggregates in one pass, one column each:

```sql
SELECT
    region,
    COUNT(*)                                    AS total_sales,
    COUNT(*) FILTER (WHERE channel = 'online')  AS online_sales,
    COUNT(*) FILTER (WHERE channel = 'store')   AS store_sales,
    SUM(amount) FILTER (WHERE channel = 'online') AS online_revenue
FROM sales
GROUP BY region;
```

```
 region | total_sales | online_sales | store_sales | online_revenue
--------+-------------+--------------+-------------+---------------
 East   |           4 |            2 |           2 |         650.00
 West   |           3 |            2 |           1 |         350.00
```

Before `FILTER` existed, people wrote `SUM(CASE WHEN channel='online' THEN amount END)`. That still works everywhere (including MySQL) and is worth knowing:

```sql
-- Portable equivalent using CASE
SELECT
    region,
    SUM(CASE WHEN channel = 'online' THEN amount ELSE 0 END) AS online_revenue,
    COUNT(CASE WHEN channel = 'online' THEN 1 END)           AS online_sales
FROM sales
GROUP BY region;
```

`FILTER` reads better and is standard SQL; `CASE` is the portable fallback.

---

## String and Array Aggregation

Aggregates aren't just numbers. Collapse a group's rows into a single string or array:

```sql
SELECT
    region,
    STRING_AGG(DISTINCT product, ', ' ORDER BY product) AS products,
    ARRAY_AGG(id ORDER BY id)                           AS sale_ids
FROM sales
GROUP BY region;
```

```
 region | products        | sale_ids
--------+-----------------+-----------
 East   | Gadget, Widget  | {4,5,6,7}
 West   | Gadget, Widget  | {1,2,3}
```

You can `ORDER BY` *inside* the aggregate to control element order — handy for building deterministic lists. PostgreSQL also has `JSON_AGG` / `JSONB_AGG` to assemble JSON documents per group. **MySQL note:** the equivalent is `GROUP_CONCAT(...)`.

---

## GROUPING SETS, ROLLUP, and CUBE — Multiple Levels at Once

Need subtotals *and* grand totals in one query? Instead of `UNION`-ing several `GROUP BY` queries, ask for multiple grouping levels directly.

`GROUPING SETS` lists exactly which groupings you want:

```sql
SELECT region, product, SUM(amount) AS revenue
FROM sales
GROUP BY GROUPING SETS (
    (region, product),   -- detail
    (region),            -- subtotal per region
    ()                   -- grand total
)
ORDER BY region NULLS LAST, product NULLS LAST;
```

```
 region | product | revenue
--------+---------+--------
 East   | Gadget  |  500.00
 East   | Widget  |  250.00
 East   | NULL    |  750.00   <- region subtotal
 West   | Gadget  |  200.00
 West   | Widget  |  200.00
 West   | NULL    |  400.00   <- region subtotal
 NULL   | NULL    | 1150.00   <- grand total
```

`ROLLUP(a, b)` is shorthand for the hierarchical sets `(a,b), (a), ()` — perfect for drill-down reports. `CUBE(a, b)` generates *all* combinations: `(a,b), (a), (b), ()`.

```sql
SELECT region, product, SUM(amount) FROM sales
GROUP BY ROLLUP (region, product);   -- detail + region subtotals + grand total
```

### Telling subtotal NULLs from data NULLs

A subtotal row has NULL in the rolled-up column — but so might a real data NULL. `GROUPING()` disambiguates: it returns 1 when the column was aggregated away for that row, 0 otherwise.

```sql
SELECT
    CASE WHEN GROUPING(region) = 1 THEN 'ALL REGIONS' ELSE region END AS region,
    CASE WHEN GROUPING(product) = 1 THEN 'ALL PRODUCTS' ELSE product END AS product,
    SUM(amount) AS revenue
FROM sales
GROUP BY ROLLUP (region, product)
ORDER BY GROUPING(region), region, GROUPING(product), product;
```

This labels subtotal rows clearly instead of leaving confusing NULLs. **MySQL note:** MySQL supports `WITH ROLLUP` (different syntax) but not `GROUPING SETS` or `CUBE`.

---

## Aggregates Without GROUP BY

Omit `GROUP BY` entirely and the whole table is one group — a single-row result:

```sql
SELECT COUNT(*), SUM(amount), AVG(amount) FROM sales;  -- one row
```

This is why you can't mix a bare column with an aggregate and no `GROUP BY` — there's exactly one output row, and a raw column would have many candidate values.

---

## Putting It Together

A realistic report exercising the whole model — filter rows, group, conditionally aggregate, filter groups, sort:

```sql
SELECT
    region,
    COUNT(*)                                       AS sales,
    SUM(amount)                                    AS revenue,
    COUNT(*) FILTER (WHERE amount IS NULL)         AS missing_amount,
    ROUND(AVG(amount), 2)                          AS avg_amount,
    STRING_AGG(DISTINCT product, ', ')             AS products
FROM sales
WHERE quantity > 0                 -- row filter first
GROUP BY region                    -- collapse to one row per region
HAVING SUM(amount) > 300           -- keep only high-revenue regions
ORDER BY revenue DESC;             -- sort the final rows
```

Read it in execution order — `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY` — and every clause has an obvious job.

---

## The Mental Model, Summarized

- `GROUP BY` collapses rows into one row per group. Everything else follows.
- `WHERE` filters rows (pre-group, no aggregates). `HAVING` filters groups (post-group, aggregates allowed).
- Aggregates ignore NULLs; `COUNT(*)` doesn't. `AVG` divides by non-NULL count.
- Non-aggregated `SELECT` columns must be in `GROUP BY` (or functionally dependent on a grouped PK).
- `FILTER` (or `CASE`) gives conditional aggregation in a single pass.
- `GROUPING SETS` / `ROLLUP` / `CUBE` produce subtotals and grand totals without unions; `GROUPING()` labels them.

Hold "one row per group" in your head and you'll never again guess whether something goes in `WHERE` or `HAVING`.
