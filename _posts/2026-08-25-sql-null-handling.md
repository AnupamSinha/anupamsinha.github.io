---
title: "NULL in SQL — The Silent Bug in Half Your Queries"
date: 2026-08-25
categories: [SQL, Fundamentals]
tags: [sql, postgresql, databases, null, data-integrity, tutorial]
description: "NULL means unknown, and that single fact breaks equality, NOT IN, aggregates, uniqueness, and joins in ways that pass code review and fail in production."
---

## NULL Is Not a Value

The root of every NULL bug is one misconception: treating NULL as if it were a value like `0` or `''`. It isn't. **NULL means "unknown."** It's the absence of a value, a placeholder for "we don't know."

Once you adopt that framing, the weird behavior becomes logical. Is an unknown number equal to 5? Unknown. Is an unknown greater than 10? Unknown. Are two unknowns equal to each other? Still unknown. SQL encodes "unknown" as a third truth value, and that third value silently reshapes your query results.

This is PostgreSQL, but the NULL semantics here are ANSI SQL and apply to MySQL, SQL Server, and Oracle alike (with a couple of vendor quirks I'll flag).

---

## Three-Valued Logic

SQL Boolean logic has **three** outcomes, not two: `TRUE`, `FALSE`, and `UNKNOWN`. Any comparison involving NULL yields `UNKNOWN`.

```sql
SELECT
    NULL = NULL      AS eq,        -- NULL (unknown), NOT true
    NULL <> NULL     AS neq,       -- NULL
    NULL = 5         AS eq_five,   -- NULL
    NULL > 5         AS gt_five,   -- NULL
    NULL + 5         AS arith;     -- NULL (arithmetic with NULL is NULL)
```

```
 eq  | neq | eq_five | gt_five | arith
-----+-----+---------+---------+-------
     |     |         |         |
```

Every column is NULL (displayed as blank). The critical rule: **`WHERE` keeps a row only when the condition is `TRUE`.** `UNKNOWN` is not `TRUE`, so the row is dropped — exactly as if it were `FALSE`. That asymmetry is where rows silently disappear.

### AND / OR / NOT with UNKNOWN

```sql
SELECT
    (TRUE  AND NULL) AS t_and,   -- NULL  (could be either, so unknown)
    (FALSE AND NULL) AS f_and,   -- FALSE (false regardless)
    (TRUE  OR  NULL) AS t_or,    -- TRUE  (true regardless)
    (FALSE OR  NULL) AS f_or,    -- NULL
    (NOT NULL)       AS not_n;   -- NULL
```

`FALSE AND NULL` is `FALSE` and `TRUE OR NULL` is `TRUE` because the other operand alone decides the outcome. Everything else with NULL stays `UNKNOWN`.

---

## Testing for NULL: IS NULL, Never = NULL

Because `= NULL` is always `UNKNOWN`, you must use `IS NULL` / `IS NOT NULL`:

```sql
-- WRONG: returns zero rows, always
SELECT * FROM users WHERE deleted_at = NULL;

-- CORRECT
SELECT * FROM users WHERE deleted_at IS NULL;
```

`WHERE deleted_at = NULL` doesn't error — it just silently matches nothing, which is worse than an error because it looks like it ran fine. (MySQL has a non-standard `<=>` null-safe equality operator; PostgreSQL uses `IS NOT DISTINCT FROM`, covered below.)

---

## The NOT IN Landmine

This is the NULL bug that has shipped in production at nearly every company. `NOT IN` with a NULL in the list returns **no rows** — even rows you'd expect to match.

```sql
CREATE TABLE employees (id INT, name TEXT, dept_id INT);
INSERT INTO employees VALUES
    (1, 'Alice', 10),
    (2, 'Bob',   20),
    (3, 'Carol', NULL);   -- one NULL dept_id

-- Intent: employees NOT in departments 10 or (unknown)
-- Actual result: ZERO rows
SELECT name FROM employees
WHERE dept_id NOT IN (10, NULL);
```

Why zero rows? `dept_id NOT IN (10, NULL)` expands to `dept_id <> 10 AND dept_id <> NULL`. That second term is always `UNKNOWN`, so the whole `AND` can never be `TRUE`. Every row is dropped.

This bites hardest with a subquery, because you often don't realize the subquery can return NULLs:

```sql
-- If ANY row of the subquery is NULL, this returns nothing
SELECT name FROM employees
WHERE dept_id NOT IN (SELECT dept_id FROM some_other_table);
```

The fixes, in order of preference:

```sql
-- 1. NOT EXISTS — immune to the NULL problem, usually the best plan too
SELECT e.name FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM some_other_table o WHERE o.dept_id = e.dept_id
);

-- 2. LEFT JOIN / IS NULL anti-join
SELECT e.name FROM employees e
LEFT JOIN some_other_table o ON o.dept_id = e.dept_id
WHERE o.dept_id IS NULL;

-- 3. If you must use NOT IN, exclude NULLs from the list
SELECT name FROM employees
WHERE dept_id NOT IN (SELECT dept_id FROM some_other_table WHERE dept_id IS NOT NULL);
```

**Make `NOT EXISTS` your default for "not in a set" logic.** It behaves correctly with NULLs and the optimizer handles it well.

---

## NULL in Joins

A join condition is a comparison, so `NULL = NULL` is `UNKNOWN` — meaning **NULLs never match in a join**, not even to each other.

```sql
-- Rows where either side's join key is NULL simply don't match
SELECT e.name, d.name AS dept
FROM employees e
JOIN departments d ON e.dept_id = d.id;   -- Carol (NULL dept_id) is excluded
```

Carol drops out of an inner join. A `LEFT JOIN` keeps her but with NULL department columns. If you genuinely need NULLs to match NULLs in a join, use `IS NOT DISTINCT FROM`:

```sql
-- Treats NULL = NULL as TRUE for matching purposes
SELECT a.id, b.id
FROM a JOIN b ON a.key IS NOT DISTINCT FROM b.key;
```

`IS NOT DISTINCT FROM` is null-safe equality: it returns `TRUE` when both sides are NULL, `FALSE` when one is NULL, and normal equality otherwise. Its complement is `IS DISTINCT FROM`.

---

## NULL in Aggregates

Aggregates **skip NULLs** (except `COUNT(*)`). This changes results in ways that look like bugs.

```sql
CREATE TABLE scores (student TEXT, score INT);
INSERT INTO scores VALUES ('A', 90), ('B', NULL), ('C', 80), ('D', NULL);

SELECT
    COUNT(*)      AS rows_total,    -- 4
    COUNT(score)  AS scored,        -- 2 (NULLs skipped)
    SUM(score)    AS sum,           -- 170
    AVG(score)    AS avg;           -- 85.0  (170/2, NOT 170/4)
```

`AVG` divides by 2, not 4 — because only 2 scores are known. If a missing score should count as 0, say so explicitly with `COALESCE`:

```sql
SELECT AVG(COALESCE(score, 0)) AS avg_zeros;  -- 42.5 (170/4)
```

One more subtlety: `SUM` over an all-NULL (or empty) set returns **NULL**, not `0`:

```sql
SELECT SUM(score) FROM scores WHERE score < 0;  -- NULL, not 0
SELECT COALESCE(SUM(score), 0) FROM scores WHERE score < 0;  -- 0
```

Wrap money and count sums in `COALESCE(..., 0)` when a NULL total would break downstream math or JSON.

---

## COALESCE, NULLIF, and the Toolbox

**`COALESCE(a, b, c, ...)`** returns the first non-NULL argument. The go-to for defaults:

```sql
SELECT COALESCE(nickname, first_name, 'Anonymous') AS display_name FROM users;
SELECT COALESCE(discount, 0) * price AS discount_amount FROM cart;
```

**`NULLIF(a, b)`** returns NULL when `a = b`, else `a`. The classic use is guarding division by zero:

```sql
-- Avoids "division by zero"; yields NULL instead when denominator is 0
SELECT revenue / NULLIF(order_count, 0) AS avg_order_value FROM stats;
```

**PostgreSQL-specific:** `column IS NOT DISTINCT FROM value` for null-safe equality, and `GREATEST` / `LEAST` ignore NULLs among their arguments in PostgreSQL (behavior differs in some other engines — check before relying on it).

**MySQL note:** MySQL adds `IFNULL(a, b)` (two-arg COALESCE) and `ISNULL(a)`; the null-safe equality operator is `<=>`.

---

## Concatenation and String Traps

In standard SQL, concatenating NULL with `||` produces NULL — one missing piece poisons the whole string:

```sql
-- If middle_name is NULL, the whole thing is NULL
SELECT first_name || ' ' || middle_name || ' ' || last_name AS full_name FROM people;

-- Fix: coalesce each nullable part, or use CONCAT_WS
SELECT CONCAT_WS(' ', first_name, middle_name, last_name) AS full_name FROM people;
```

`CONCAT_WS` (concatenate with separator) **skips NULL arguments** and doesn't emit a doubled separator — the clean way to build names, addresses, and CSV lines from nullable columns. Plain `CONCAT(...)` also treats NULL as an empty string, unlike `||`.

---

## Sorting: Where Do NULLs Go?

NULLs sort as if larger than everything by default in PostgreSQL, so they land last on `ASC` and first on `DESC`. You can override this:

```sql
SELECT name, score FROM scores ORDER BY score DESC NULLS LAST;
SELECT name, score FROM scores ORDER BY score ASC  NULLS FIRST;
```

**MySQL note:** MySQL sorts NULLs as *smallest* (first on `ASC`) and lacks `NULLS FIRST/LAST`; emulate with `ORDER BY col IS NULL, col`.

---

## NULL, UNIQUE Constraints, and Primary Keys

A `UNIQUE` constraint permits **multiple NULLs**, because two unknowns aren't considered equal (they aren't "duplicates"):

```sql
CREATE TABLE accounts (id INT PRIMARY KEY, email TEXT UNIQUE);
INSERT INTO accounts VALUES (1, NULL), (2, NULL);  -- both succeed!
```

That surprises people expecting "unique email" to also mean "at most one row without an email." If you need "unique when present, and at most one NULL", you need extra rules. PostgreSQL 15+ offers `UNIQUE NULLS NOT DISTINCT` to treat NULLs as equal (thus allowing only one):

```sql
-- PostgreSQL 15+: only ONE NULL email allowed
CREATE TABLE accounts2 (id INT PRIMARY KEY, email TEXT UNIQUE NULLS NOT DISTINCT);
```

And a `PRIMARY KEY` column can never be NULL — `NOT NULL` is implied.

---

## CHECK Constraints and NULL

A `CHECK` constraint passes when it evaluates to `TRUE` *or* `UNKNOWN` — it only rejects `FALSE`. So NULLs slip past checks unless you explicitly forbid them:

```sql
CREATE TABLE products (
    id    INT PRIMARY KEY,
    price NUMERIC CHECK (price > 0)   -- NULL price is ALLOWED (check is UNKNOWN)
);
INSERT INTO products VALUES (1, NULL);  -- succeeds despite CHECK (price > 0)
```

If price must be positive *and* present, add `NOT NULL`:

```sql
price NUMERIC NOT NULL CHECK (price > 0)
```

This is a subtle data-integrity hole. Audit `CHECK` constraints on nullable columns and add `NOT NULL` wherever "unknown" isn't a legal state.

---

## DISTINCT and GROUP BY Treat NULLs as Equal

Here's an apparent contradiction worth internalizing. For *comparisons*, `NULL <> NULL`. But for **grouping and deduplication**, SQL treats all NULLs as *the same group*:

```sql
SELECT DISTINCT dept_id FROM employees;  -- NULL appears exactly ONCE

SELECT dept_id, COUNT(*) FROM employees GROUP BY dept_id;  -- all NULLs in one group
```

So NULLs are "not equal" when you compare them with `=`, but "the same" when you `GROUP BY` or `DISTINCT`. Both behaviors are intentional; keep the two contexts separate in your head.

---

## A Practical Checklist

Before shipping a query that touches nullable columns, run through this:

- Using `= NULL` anywhere? Replace with `IS NULL`.
- Any `NOT IN (subquery)`? Assume the subquery can return NULL and switch to `NOT EXISTS`.
- Any `AVG` / `SUM` where missing should mean zero? `COALESCE` the input or the result.
- Building strings from nullable columns? Use `CONCAT_WS`, not `||`.
- Dividing by a column that could be zero? Wrap the denominator in `NULLIF(..., 0)`.
- Joining on columns that can be NULL? Decide whether NULLs should match (`IS NOT DISTINCT FROM`) or drop.
- `ORDER BY` on a nullable column? Add explicit `NULLS FIRST` / `NULLS LAST`.
- Relying on `UNIQUE` to prevent duplicate NULLs? It doesn't (pre-PG15) — add the right constraint.
- `CHECK` on a nullable column? Add `NOT NULL` if "unknown" is illegal.

NULL isn't a flaw in SQL — it's a precise way to model "unknown." The bugs come from forgetting that and treating it like a value. Keep the "unknown" framing front of mind and the silent failures become predictable, catchable, and preventable.
