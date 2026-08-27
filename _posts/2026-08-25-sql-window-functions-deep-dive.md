---
title: "Window Functions in SQL — The Feature That Separates Juniors from Seniors"
date: 2026-08-25
categories: [SQL, Analytics]
tags: [sql, postgresql, databases, window-functions, tutorial]
description: "Running totals, rankings, moving averages, and gaps-and-islands — a complete, code-heavy guide to SQL window functions without collapsing your rows."
---

## The Aha Moment

Here's the query that makes window functions click. You want each order alongside the customer's running total — but you don't want to lose the individual rows. `GROUP BY` can't do this; it collapses rows. A subquery per row is slow and ugly. A window function does it in one clean expression:

```sql
SELECT
    customer_id,
    id AS order_id,
    amount,
    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY id
    ) AS running_total
FROM orders;
```

That `OVER (...)` clause is the whole game. It says: "compute an aggregate, but over a *window* of related rows, and keep every row." Once you internalize windows, a huge class of reporting queries that used to require self-joins or application-side loops becomes a few lines of SQL.

Everything here is PostgreSQL. MySQL 8.0+ supports the same syntax; I note differences where they matter.

---

## The Sample Data

```sql
CREATE TABLE sales (
    id          INT PRIMARY KEY,
    region      TEXT,
    salesperson TEXT,
    sale_date   DATE,
    amount      NUMERIC(10,2)
);

INSERT INTO sales (id, region, salesperson, sale_date, amount) VALUES
    (1, 'West', 'Alice', '2026-01-05', 100),
    (2, 'West', 'Alice', '2026-01-12', 150),
    (3, 'West', 'Bob',   '2026-01-08', 200),
    (4, 'West', 'Bob',   '2026-01-20', 200),
    (5, 'East', 'Carol', '2026-01-03', 300),
    (6, 'East', 'Carol', '2026-01-19', 120),
    (7, 'East', 'Dave',  '2026-01-11',  90),
    (8, 'East', 'Dave',  '2026-01-25', 400);
```

---

## Anatomy of the OVER Clause

Every window function has three optional parts inside `OVER`:

```sql
function() OVER (
    PARTITION BY <cols>   -- split rows into groups (like GROUP BY, but rows survive)
    ORDER BY <cols>       -- order rows within each partition
    <frame>               -- which rows around the current one to include
)
```

- **PARTITION BY** divides the result into independent groups. The function restarts for each partition. Omit it and the whole result is one partition.
- **ORDER BY** defines row order within a partition — essential for running totals, rankings, and `LAG`/`LEAD`.
- **Frame** (like `ROWS BETWEEN ...`) narrows the window to a sliding subset. This is where most people get surprised.

---

## Ranking Functions: ROW_NUMBER, RANK, DENSE_RANK

These three look similar and behave differently on ties. Learn the distinction cold — it's a classic interview question and a real source of bugs.

```sql
SELECT
    region,
    salesperson,
    amount,
    ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC) AS row_num,
    RANK()       OVER (PARTITION BY region ORDER BY amount DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY region ORDER BY amount DESC) AS dense_rnk
FROM sales;
```

For the West region (amounts 200, 200, 150, 100):

```
 region | salesperson | amount | row_num | rnk | dense_rnk
--------+-------------+--------+---------+-----+-----------
 West   | Bob         | 200.00 |       1 |   1 |         1
 West   | Bob         | 200.00 |       2 |   1 |         1
 West   | Alice       | 150.00 |       3 |   3 |         2
 West   | Alice       | 100.00 |       4 |   4 |         3
```

- **ROW_NUMBER** — always unique, 1..N. Ties are broken arbitrarily (add a tiebreaker to `ORDER BY` for determinism).
- **RANK** — ties share a rank, then it *skips* (1, 1, 3, 4).
- **DENSE_RANK** — ties share a rank, no gaps (1, 1, 2, 3).

### Top-N-per-group

The canonical use of `ROW_NUMBER`: get the top 2 sales per region. Window functions can't go in `WHERE`, so filter in an outer query or CTE:

```sql
SELECT region, salesperson, amount
FROM (
    SELECT region, salesperson, amount,
           ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC) AS rn
    FROM sales
) ranked
WHERE rn <= 2;
```

Want to *include* ties for 2nd place? Swap `ROW_NUMBER` for `RANK`.

---

## Running Totals and the Frame Clause

A running total is `SUM(...) OVER (ORDER BY ...)`. But there's a subtlety in the default frame.

```sql
SELECT
    salesperson,
    sale_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY salesperson
        ORDER BY sale_date
    ) AS running_total
FROM sales
ORDER BY salesperson, sale_date;
```

When you specify `ORDER BY` but no explicit frame, the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. That gives a running total — good. But `RANGE` treats **ties in the order column as one unit**, which can surprise you when timestamps repeat. For a strict row-by-row cumulative sum, be explicit with `ROWS`:

```sql
SUM(amount) OVER (
    PARTITION BY salesperson
    ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
) AS running_total
```

Rule: **use `ROWS` for physical row counts; use `RANGE` for value-based windows.** When in doubt, specify `ROWS` explicitly to avoid the tie surprise.

---

## Moving Averages with a Sliding Frame

A frame can be a fixed-size window that slides. A 3-row moving average (current row plus the two before it):

```sql
SELECT
    sale_date,
    amount,
    AVG(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3
FROM sales
ORDER BY sale_date;
```

The frame bounds you'll use most:

- `UNBOUNDED PRECEDING` — start of the partition
- `N PRECEDING` — N rows before the current row
- `CURRENT ROW`
- `N FOLLOWING` — N rows after the current row
- `UNBOUNDED FOLLOWING` — end of the partition

A centered 3-row average: `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING`. A total that looks ahead to the partition end: `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING`.

---

## LAG and LEAD: Peeking at Neighboring Rows

`LAG` reads a previous row's value; `LEAD` reads a following row's value — without a self-join. Perfect for period-over-period comparisons.

```sql
SELECT
    salesperson,
    sale_date,
    amount,
    LAG(amount) OVER (PARTITION BY salesperson ORDER BY sale_date) AS prev_amount,
    amount - LAG(amount) OVER (PARTITION BY salesperson ORDER BY sale_date) AS delta
FROM sales
ORDER BY salesperson, sale_date;
```

The first row of each partition has no previous row, so `prev_amount` is NULL. Supply a default and an offset with the extra arguments — `LAG(amount, 1, 0)` returns 0 instead of NULL for the first row:

```sql
LAG(amount, 1, 0) OVER (PARTITION BY salesperson ORDER BY sale_date)
```

### Percent change month over month

```sql
SELECT
    sale_date,
    amount,
    ROUND(
        100.0 * (amount - LAG(amount) OVER (ORDER BY sale_date))
        / NULLIF(LAG(amount) OVER (ORDER BY sale_date), 0),
        1
    ) AS pct_change
FROM sales
ORDER BY sale_date;
```

Note the `NULLIF(..., 0)` guarding against division by zero — a habit worth building.

---

## FIRST_VALUE, LAST_VALUE, and NTH_VALUE

Grab a specific row's value from within the window. `FIRST_VALUE` is straightforward:

```sql
SELECT
    region,
    salesperson,
    amount,
    FIRST_VALUE(salesperson) OVER (
        PARTITION BY region ORDER BY amount DESC
    ) AS top_seller
FROM sales;
```

`LAST_VALUE` has a famous trap. With the default frame (`... AND CURRENT ROW`), "last value" means "last value *so far*", which is just the current row. To get the true last value in the partition, extend the frame to the end:

```sql
LAST_VALUE(salesperson) OVER (
    PARTITION BY region
    ORDER BY amount DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
) AS bottom_seller
```

This frame gotcha bites everyone once. Remember: `LAST_VALUE` almost always needs an explicit full-partition frame.

---

## NTILE: Bucketing Rows

`NTILE(n)` splits an ordered partition into `n` roughly equal buckets — great for quartiles, deciles, and percentile bands.

```sql
SELECT
    salesperson,
    amount,
    NTILE(4) OVER (ORDER BY amount) AS quartile
FROM sales;
```

If rows don't divide evenly, earlier buckets get the extra rows. `NTILE(4)` over 10 rows yields buckets of size 3, 3, 2, 2.

---

## PERCENT_RANK and CUME_DIST

For relative standing:

- `PERCENT_RANK()` — relative rank in `[0, 1]`: `(rank - 1) / (total_rows - 1)`.
- `CUME_DIST()` — cumulative distribution: fraction of rows with a value ≤ the current row.

```sql
SELECT
    salesperson,
    amount,
    ROUND(PERCENT_RANK() OVER (ORDER BY amount)::numeric, 2) AS pct_rank,
    ROUND(CUME_DIST()    OVER (ORDER BY amount)::numeric, 2) AS cume_dist
FROM sales;
```

These power "this sale is in the top 10%" style analytics without a second pass over the data.

---

## Named Windows with WINDOW

When several functions share the same window spec, repeating `OVER (PARTITION BY ... ORDER BY ...)` is noisy. Define it once with a `WINDOW` clause:

```sql
SELECT
    region,
    salesperson,
    amount,
    ROW_NUMBER() OVER w   AS rn,
    RANK()       OVER w   AS rnk,
    SUM(amount)  OVER w   AS running_total
FROM sales
WINDOW w AS (PARTITION BY region ORDER BY amount DESC);
```

Cleaner, and there's a single place to change the spec. Supported in PostgreSQL and MySQL 8.0+.

---

## Gaps and Islands: The Advanced Pattern

A senior-level classic. Given a sequence, find contiguous runs ("islands") and the gaps between them. The trick: `row_number - value` is constant within a consecutive run.

```sql
CREATE TABLE logins (user_id INT, login_date DATE);
INSERT INTO logins VALUES
    (1, '2026-01-01'), (1, '2026-01-02'), (1, '2026-01-03'),  -- island 1
    (1, '2026-01-06'), (1, '2026-01-07');                     -- island 2

SELECT
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*)        AS streak_length
FROM (
    SELECT
        user_id,
        login_date,
        login_date - (ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY login_date
        ))::int AS grp
    FROM logins
) t
GROUP BY user_id, grp
ORDER BY streak_start;
```

```
 user_id | streak_start | streak_end | streak_length
---------+--------------+------------+---------------
       1 | 2026-01-01   | 2026-01-03 |             3
       1 | 2026-01-06   | 2026-01-07 |             2
```

Because consecutive dates increase by 1 and `ROW_NUMBER` also increases by 1, their difference (`grp`) stays constant within a run and jumps at every gap. Grouping by `grp` collapses each island. This pattern shows up everywhere: login streaks, uptime windows, consecutive-status runs.

---

## Where Window Functions Run in the Pipeline

Understanding the logical order of operations explains two common errors:

```
FROM → WHERE → GROUP BY → HAVING → window functions → SELECT → ORDER BY
```

Two consequences:

1. **You can't reference a window function in `WHERE` or `GROUP BY`** — they haven't been computed yet. Wrap the query in a subquery/CTE and filter in the outer layer.
2. **Window functions see the post-`GROUP BY` rows.** You can layer a window function on top of an aggregate:

```sql
SELECT
    region,
    SUM(amount) AS region_total,
    SUM(SUM(amount)) OVER () AS grand_total,
    ROUND(100.0 * SUM(amount) / SUM(SUM(amount)) OVER (), 1) AS pct_of_total
FROM sales
GROUP BY region;
```

That nested `SUM(SUM(amount)) OVER ()` — "sum of the group sums across all groups" — computes each region's share of the grand total in a single query. It looks strange the first time, but it's exactly the aggregate-then-window layering the pipeline permits.

---

## Performance Notes

- Window functions require a sort per distinct `PARTITION BY / ORDER BY`. An index matching that ordering lets PostgreSQL skip the sort. Check with `EXPLAIN ANALYZE` for a `WindowAgg` node fed by an `Index Scan` rather than a `Sort`.
- Reusing the exact same window spec across multiple functions (or via a named `WINDOW`) lets the planner compute them in one pass instead of several.
- Window functions don't reduce row count, so a huge partition still materializes a lot of rows. Filter with `WHERE` *before* the window whenever the logic allows.

---

## Quick Reference

**ROW_NUMBER / RANK / DENSE_RANK** — sequential position; differ on ties.

**SUM / AVG / COUNT OVER** — running totals and moving windows via the frame clause.

**LAG / LEAD** — read neighboring rows for period-over-period math.

**FIRST_VALUE / LAST_VALUE / NTH_VALUE** — pull a specific row's value; watch `LAST_VALUE`'s frame.

**NTILE / PERCENT_RANK / CUME_DIST** — bucketing and relative standing.

**Frame clause** — `ROWS` for physical rows, `RANGE` for value ranges; the default frame surprises people.

Window functions turn multi-pass, self-joining, application-side logic into declarative SQL. That fluency is exactly what separates a junior query from a senior one.
