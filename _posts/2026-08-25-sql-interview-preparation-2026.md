---
title: "The Complete SQL Interview Preparation Guide (2026)"
date: 2026-08-25
categories: [SQL, Interviews]
tags: [sql, postgresql, mysql, databases, interview, career]
description: "The SQL interview questions that actually get asked, the patterns behind them, and the reasoning interviewers listen for — fundamentals to hard problems, PostgreSQL-first."
---

## How SQL Interviews Actually Work

SQL rounds aren't about memorizing syntax. Interviewers are checking three things: do you understand set-based thinking (not row-by-row loops), can you reason about *why* a query behaves the way it does, and do you know how it performs. A candidate who writes a working query and then says "this will do a sequential scan unless we index `customer_id`" clears the bar that a candidate who just writes working SQL doesn't.

This guide follows that structure: fundamentals you must not fumble, the core query patterns, the performance conversation, and the hard problems that separate strong candidates. Everything is PostgreSQL; MySQL differences are noted where they'd trip you up.

Our working schema:

```sql
CREATE TABLE employees (
    id         INT PRIMARY KEY,
    name       TEXT,
    dept_id    INT,
    manager_id INT,          -- self-reference
    salary     NUMERIC(10,2),
    hire_date  DATE
);

CREATE TABLE departments (
    id   INT PRIMARY KEY,
    name TEXT
);

CREATE TABLE orders (
    id          INT PRIMARY KEY,
    customer_id INT,
    amount      NUMERIC(10,2),
    created_at  TIMESTAMPTZ
);
```

---

## Part 1 — Fundamentals You Cannot Fumble

### The Logical Order of Operations

If you learn one thing, learn this. SQL is *written* in one order and *executed* in another:

```
FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

This single fact answers a dozen interview questions:

- **Why can't I use a `SELECT` alias in `WHERE`?** Because `WHERE` runs before `SELECT` — the alias doesn't exist yet.
- **When do I use `WHERE` vs `HAVING`?** `WHERE` filters rows before grouping; `HAVING` filters groups after aggregation.
- **Why can't I filter on a window function in `WHERE`?** Windows compute after `GROUP BY`/`HAVING`, so wrap it in a subquery.

```sql
-- WHERE filters rows, HAVING filters groups
SELECT dept_id, AVG(salary) AS avg_salary
FROM employees
WHERE hire_date >= '2020-01-01'   -- pre-aggregation row filter
GROUP BY dept_id
HAVING AVG(salary) > 80000;       -- post-aggregation group filter
```

### JOIN Types

Be able to explain each in one sentence and predict row counts.

```sql
-- INNER: only matching rows on both sides
SELECT e.name, d.name FROM employees e
JOIN departments d ON d.id = e.dept_id;

-- LEFT: all left rows, NULLs where no right match
SELECT e.name, d.name FROM employees e
LEFT JOIN departments d ON d.id = e.dept_id;

-- Find employees with no department (anti-join)
SELECT e.name FROM employees e
LEFT JOIN departments d ON d.id = e.dept_id
WHERE d.id IS NULL;
```

That last pattern — LEFT JOIN plus `WHERE right.key IS NULL` — is the classic "find rows in A with no match in B" and gets asked constantly.

### NULL Semantics

The most common correctness trap in interviews.

```sql
-- NULL is never equal to anything, including NULL
SELECT NULL = NULL;      -- NULL (not true!)
SELECT NULL <> 5;        -- NULL
WHERE col = NULL         -- matches nothing; use IS NULL

-- COUNT(*) counts rows; COUNT(col) skips NULLs
SELECT COUNT(*), COUNT(manager_id) FROM employees;  -- differ if any NULL managers

-- NOT IN with a NULL in the list returns zero rows — know why
SELECT * FROM employees WHERE id NOT IN (SELECT manager_id FROM employees);
-- If any manager_id is NULL, this returns nothing. Use NOT EXISTS instead.
```

If you can explain the `NOT IN` NULL trap unprompted, you signal seniority immediately.

---

## Part 2 — The Core Query Patterns

### Aggregation With GROUP BY

```sql
-- Count and total per department, only departments with 3+ employees
SELECT d.name, COUNT(*) AS headcount, SUM(e.salary) AS payroll
FROM employees e
JOIN departments d ON d.id = e.dept_id
GROUP BY d.name
HAVING COUNT(*) >= 3
ORDER BY payroll DESC;
```

**Interview trap:** every non-aggregated column in `SELECT` must appear in `GROUP BY` (in standard SQL and Postgres). MySQL historically let you select un-grouped columns and returned arbitrary values — a footgun; `ONLY_FULL_GROUP_BY` mode (default in modern MySQL) now enforces the standard.

### The Second-Highest Salary — The Classic

You *will* see a variant of this. Know multiple approaches.

```sql
-- Approach 1: DISTINCT + OFFSET
SELECT DISTINCT salary FROM employees
ORDER BY salary DESC
OFFSET 1 LIMIT 1;

-- Approach 2: DENSE_RANK (handles ties correctly, generalizes to Nth)
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) t
WHERE rnk = 2
LIMIT 1;
```

The follow-up is always "now the Nth highest" and "now per department." The `DENSE_RANK` version answers both — add `PARTITION BY dept_id` for per-group. Explain *why* `DENSE_RANK` over `RANK` (ties shouldn't skip the number you want).

### Top-N Per Group

```sql
-- Highest-paid employee in each department
SELECT dept_id, name, salary FROM (
    SELECT dept_id, name, salary,
           ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
    FROM employees
) t
WHERE rn = 1;
```

`ROW_NUMBER` for exactly one, `RANK`/`DENSE_RANK` if ties should all qualify. This pattern is the single most reusable window-function idiom in interviews.

### Self-Joins

```sql
-- Employees who earn more than their manager
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS mgr_salary
FROM employees e
JOIN employees m ON m.id = e.manager_id
WHERE e.salary > m.salary;
```

The trick is aliasing the same table twice and thinking of them as two independent copies.

### Duplicates

```sql
-- Find duplicate emails
SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;

-- Delete duplicates, keeping the lowest id (Postgres)
DELETE FROM users a
USING users b
WHERE a.email = b.email AND a.id > b.id;
```

---

## Part 3 — Window Functions (The Differentiator)

Window functions are the fastest way to signal you're past the junior tier.

```sql
-- Running total of orders per customer
SELECT customer_id, created_at, amount,
       SUM(amount) OVER (
           PARTITION BY customer_id ORDER BY created_at
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total
FROM orders;

-- Month-over-month change with LAG
SELECT created_at::date AS day, amount,
       amount - LAG(amount) OVER (ORDER BY created_at) AS delta
FROM orders;
```

Know the three ranking functions cold and their tie behavior:

```sql
SELECT name, salary,
       ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn,   -- 1,2,3,4 always unique
       RANK()       OVER (ORDER BY salary DESC) AS rnk,  -- 1,1,3,4 (skips)
       DENSE_RANK() OVER (ORDER BY salary DESC) AS drnk  -- 1,1,2,3 (no gaps)
FROM employees;
```

**The window-function trap:** you cannot put a window function in `WHERE`. Wrap it:

```sql
SELECT * FROM (
    SELECT name, salary, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
    FROM employees
) t WHERE rn <= 3;
```

---

## Part 4 — CTEs and Recursion

### Common Table Expressions

CTEs make complex queries readable — interviewers appreciate clarity.

```sql
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees GROUP BY dept_id
)
SELECT e.name, e.salary, d.avg_salary
FROM employees e
JOIN dept_avg d ON d.dept_id = e.dept_id
WHERE e.salary > d.avg_salary;   -- earns above their department average
```

### Recursive CTEs — Hierarchies

The org chart / category tree question. Know the anchor + recursive structure.

```sql
WITH RECURSIVE org AS (
    -- anchor: top-level (no manager)
    SELECT id, name, manager_id, 1 AS level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    -- recursive: employees reporting to someone already in `org`
    SELECT e.id, e.name, e.manager_id, o.level + 1
    FROM employees e
    JOIN org o ON o.id = e.manager_id
)
SELECT * FROM org ORDER BY level, id;
```

Explain the mental model: the anchor seeds the result, then the recursive term runs repeatedly against the rows produced last iteration until it produces none. Both PostgreSQL and MySQL 8.0+ support this.

---

## Part 5 — The Performance Conversation

This is where mid-level candidates and senior candidates diverge. Expect "how would you make this faster?"

### Reading EXPLAIN

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42;
```

Talk through what you'd look for:
- **Seq Scan** on a large table with a selective filter → missing index.
- **Index Scan / Index Only Scan** → good; the latter means the index covered the query.
- **Nested Loop** vs **Hash Join** vs **Merge Join** → know when each is appropriate (nested loop for small outer + indexed inner; hash for big unsorted joins).
- Big gap between **estimated and actual rows** → stale statistics; suggest `ANALYZE`.

### Indexing Reasoning

```sql
-- This query
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 10;

-- benefits from a composite index matching filter + sort
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at DESC);
```

Key points to articulate: composite index column order matters (equality columns first, then the range/sort column); an index on `(a, b)` helps queries filtering on `a` or `a,b` but not `b` alone (the leftmost-prefix rule); indexes speed reads but cost writes and storage.

### The Classic "Why Is This Slow"

Be ready to name the usual suspects: no index on the filter/join column, a function wrapping an indexed column (`WHERE LOWER(email) = ...`), a leading-wildcard `LIKE '%x'`, `OFFSET`-heavy pagination, the N+1 pattern from the application, and stale statistics. Naming these fluently is worth more than any single clever query.

---

## Part 6 — Hard Problems Worth Practicing

### Gaps and Islands

Find consecutive-day login streaks. The trick: `date - row_number` is constant within a run.

```sql
SELECT user_id, MIN(login_date) AS start, MAX(login_date) AS finish, COUNT(*) AS len
FROM (
    SELECT user_id, login_date,
           login_date - (ROW_NUMBER() OVER (
               PARTITION BY user_id ORDER BY login_date))::int AS grp
    FROM logins
) t
GROUP BY user_id, grp;
```

### Median Without a Built-in

```sql
-- PostgreSQL has percentile_cont — know it exists
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY salary) AS median FROM employees;
```

If asked to do it *without* the built-in, the window-function approach (pick the middle row(s) by `ROW_NUMBER` compared against `COUNT(*)`) is the fallback — be ready to sketch it.

### Pivoting

```sql
-- Count orders per status as columns using FILTER (Postgres, clean)
SELECT customer_id,
       COUNT(*) FILTER (WHERE amount > 100) AS big_orders,
       COUNT(*) FILTER (WHERE amount <= 100) AS small_orders
FROM orders
GROUP BY customer_id;
```

The `FILTER` clause is cleaner than `CASE WHEN` inside aggregates and shows you know modern SQL. MySQL uses `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`.

### Running Deduplication / Latest Record Per Key

```sql
-- Latest order per customer (DISTINCT ON is a Postgres gem)
SELECT DISTINCT ON (customer_id) customer_id, id, amount, created_at
FROM orders
ORDER BY customer_id, created_at DESC;
```

`DISTINCT ON` is PostgreSQL-specific and elegant; the portable equivalent is the `ROW_NUMBER() = 1` pattern. Mention both.

---

## Part 7 — Behavioral and Reasoning Questions

Interviewers also probe how you think:

- **"When would you denormalize?"** When read performance from repeated joins outweighs the cost of maintaining redundant data, and you accept the write/consistency burden. Have a concrete example ready.
- **"Transaction isolation levels?"** Know `READ COMMITTED` (Postgres default) vs `REPEATABLE READ` vs `SERIALIZABLE`, and the anomalies each prevents (dirty read, non-repeatable read, phantom read).
- **"How do you prevent a lost update?"** Optimistic locking with a version column, or `SELECT ... FOR UPDATE` pessimistic locking.
- **"Index tradeoffs?"** Faster reads, slower writes, more storage, and they can become stale.

---

## The Night-Before Checklist

- Logical order of operations — recite it.
- JOIN types and the LEFT JOIN + `IS NULL` anti-join.
- NULL semantics and the `NOT IN` trap.
- `WHERE` vs `HAVING`.
- Second-highest / Nth-highest / top-N-per-group via `DENSE_RANK`/`ROW_NUMBER`.
- Window functions: the three rankings, `LAG`/`LEAD`, running totals, and why they can't go in `WHERE`.
- CTEs and recursive CTEs for hierarchies.
- Reading `EXPLAIN ANALYZE` and naming the common causes of slow queries.
- `DISTINCT ON`, `FILTER`, `percentile_cont` as the modern-Postgres flourishes.

Practice writing these by hand, out loud, explaining the *why* as you go. The candidate who narrates "I'm using `DENSE_RANK` here because ties shouldn't consume a rank, and this will need an index on `salary` to avoid a sort" is the one who gets the offer. Working SQL is table stakes; reasoning about it is the interview.
