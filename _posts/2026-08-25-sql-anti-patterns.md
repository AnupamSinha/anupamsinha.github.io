---
title: "SQL Anti-Patterns — 10 Mistakes That Will Get You Rejected in Code Review"
date: 2026-08-25
categories: [SQL, Best Practices]
tags: [sql, postgresql, mysql, databases, anti-patterns, code-review]
description: "The 10 SQL habits senior reviewers flag on sight — from SELECT * to NOT IN with NULLs — and the correct patterns to replace them."
---

## Why This List Exists

Most SQL bugs don't announce themselves. They pass tests against a 100-row seed database, sail through code review if the reviewer is skimming, and then quietly return wrong results or table-lock production at 3x traffic. The queries below are the ones I flag in almost every review. None of them are exotic. They're the everyday habits that separate code that works on your laptop from code that survives a real workload.

Everything here is PostgreSQL-first. Where MySQL behaves differently — and it often does — I call it out.

Sample schema we'll use throughout:

```sql
CREATE TABLE customers (
    id          BIGINT PRIMARY KEY,
    email       TEXT NOT NULL,
    country     TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    id           BIGINT PRIMARY KEY,
    customer_id  BIGINT NOT NULL REFERENCES customers(id),
    status       TEXT NOT NULL,
    total_cents  BIGINT NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 1. SELECT * in Application Code

`SELECT *` is fine at the psql prompt. In application code it's a landmine.

```sql
-- The anti-pattern
SELECT * FROM orders WHERE customer_id = 42;
```

Three problems. First, you pull columns you don't need — including large `TEXT`/`JSONB` blobs — inflating network and memory. Second, adding a column upstream silently changes your result shape, which breaks positional deserialization and ORDs that map by index. Third, it defeats **covering indexes**: if your index contains exactly the columns a query needs, PostgreSQL can serve it index-only, but `SELECT *` forces a heap fetch every time.

```sql
-- The fix: name what you need
SELECT id, status, total_cents
FROM orders
WHERE customer_id = 42;
```

Now this index can serve the query without touching the table:

```sql
CREATE INDEX idx_orders_customer
    ON orders (customer_id) INCLUDE (status, total_cents);
```

Reviewer's rule: `SELECT *` in a migration, a view, or a hot query is an automatic comment.

---

## 2. Implicit Type Coercion That Kills Indexes

You have an index on `orders.id` (a `BIGINT`). Then someone binds a string:

```sql
-- The anti-pattern: comparing bigint to text
SELECT * FROM orders WHERE id = '42';
```

PostgreSQL is usually smart enough to coerce a literal here, but the truly dangerous version is a function or cast on the *indexed column*:

```sql
-- Index on created_at is now useless
SELECT * FROM orders WHERE created_at::date = '2026-08-25';
```

Casting `created_at` to `date` means the index on `created_at` can't be used — PostgreSQL has to compute the expression for every row. Rewrite as a range so the raw column stays comparable:

```sql
SELECT * FROM orders
WHERE created_at >= '2026-08-25'
  AND created_at <  '2026-08-26';
```

If you genuinely need to query by an expression, index the expression itself:

```sql
CREATE INDEX idx_orders_created_date ON orders ((created_at::date));
```

**MySQL note:** the same rule applies, and MySQL's silent type juggling is worse — comparing a numeric column to a string can coerce the *column*, not the literal, dropping the index and sometimes changing results. Always match types.

---

## 3. Leading Wildcards on LIKE

```sql
-- The anti-pattern: leading % means full scan
SELECT * FROM customers WHERE email LIKE '%@gmail.com';
```

A B-tree index is sorted left to right. A leading `%` means the engine has no prefix to seek on, so it scans every row. A trailing wildcard is index-friendly:

```sql
SELECT * FROM customers WHERE email LIKE 'anupam%';  -- can use the index
```

For genuine substring or suffix search in PostgreSQL, use a trigram index instead of hoping the B-tree helps:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_customers_email_trgm
    ON customers USING gin (email gin_trgm_ops);

-- Now this is index-accelerated
SELECT * FROM customers WHERE email LIKE '%@gmail.com';
```

For full-text needs, reach for `tsvector`/`tsquery`. Don't paper over a search requirement with `LIKE '%...%'` and call it done.

---

## 4. The N+1 Query in Disguise

This one rarely shows up in the SQL file — it hides in the application loop.

```java
// The anti-pattern
List<Order> orders = orderRepo.findByCustomer(customerId);
for (Order o : orders) {
    o.setItems(itemRepo.findByOrderId(o.getId())); // one query per order
}
```

One hundred orders means one hundred and one round trips. Each trip pays network latency and planning overhead. Collapse it into a single set-based query:

```sql
SELECT o.id AS order_id, i.id AS item_id, i.sku, i.qty
FROM orders o
JOIN order_items i ON i.order_id = o.id
WHERE o.customer_id = 42
ORDER BY o.id;
```

Or, if you truly need it split, batch the second query with `IN` / `= ANY`:

```sql
SELECT * FROM order_items WHERE order_id = ANY($1);  -- $1 = array of order ids
```

In ORMs, this is `JOIN FETCH` (JPA), `.includes` (ActiveRecord), or `selectinload` (SQLAlchemy). If a reviewer sees a repository call inside a loop, expect a comment.

---

## 5. OFFSET-Based Pagination on Large Tables

```sql
-- The anti-pattern: gets slower every page
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100000;
```

`OFFSET 100000` doesn't skip rows for free — the engine generates and discards all 100,000 rows before returning the 20 you want. Page 1 is instant; page 5,000 crawls. Use **keyset (seek) pagination** anchored on the last row you saw:

```sql
SELECT * FROM orders
WHERE (created_at, id) < ($1, $2)   -- last row's created_at and id
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Now every page is equally fast because a matching index seeks directly to the boundary:

```sql
CREATE INDEX idx_orders_created_id ON orders (created_at DESC, id DESC);
```

The tradeoff is you can't jump to an arbitrary page number — but "page 5,000" is almost never a real user need. Infinite scroll and "next" buttons map perfectly to keyset.

---

## 6. Functions Wrapped Around Indexed Columns

Closely related to #2, but common enough to stand alone.

```sql
-- The anti-pattern
SELECT * FROM customers WHERE LOWER(email) = 'anupam@example.com';
SELECT * FROM orders    WHERE EXTRACT(YEAR FROM created_at) = 2026;
```

Any function on the column makes the plain-column index unusable. Two fixes, depending on the case:

Store or index the normalized form:

```sql
-- Expression index for case-insensitive lookup
CREATE INDEX idx_customers_email_lower ON customers (LOWER(email));

-- Or use a case-insensitive type
ALTER TABLE customers ALTER COLUMN email TYPE citext;
```

Convert the year predicate to a range so the raw column is compared directly:

```sql
SELECT * FROM orders
WHERE created_at >= '2026-01-01'
  AND created_at <  '2027-01-01';
```

The pattern to internalize: **keep the indexed column bare on one side of the comparison.**

---

## 7. NOT IN With a Nullable Subquery

This is a correctness bug, not just a performance one — and it's brutal because it looks right.

```sql
-- The anti-pattern
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
```

If any `customer_id` in that subquery is `NULL`, the entire `NOT IN` returns **zero rows**. Here's why: `id NOT IN (1, 2, NULL)` expands to `id <> 1 AND id <> 2 AND id <> NULL`, and `id <> NULL` is `UNKNOWN`, which can never be true. One stray NULL silently empties your result.

Use `NOT EXISTS`, which has clean three-valued-logic semantics and usually optimizes better anyway:

```sql
SELECT c.*
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

Rule: **prefer `NOT EXISTS` over `NOT IN` for subqueries, always.** Reserve `NOT IN` for hardcoded, guaranteed-non-null literal lists.

---

## 8. Counting Rows With COUNT(*) on Huge Tables for Pagination

```sql
-- The anti-pattern: exact count on every page load
SELECT COUNT(*) FROM orders WHERE status = 'shipped';
```

In PostgreSQL, `COUNT(*)` with a filter must scan every matching row (MVCC means there's no O(1) row count). On a big table behind a "showing X of N results" widget, this dominates your page latency. Options, in order of preference:

Drop the exact count. Most UIs don't need it — "load more" doesn't care about `N`.

If you need an approximation, read the planner's estimate:

```sql
SELECT reltuples::bigint AS approx_rows
FROM pg_class WHERE relname = 'orders';
```

Or run `EXPLAIN` and parse the estimated row count. For filtered approximate counts, `EXPLAIN (FORMAT JSON) SELECT ...` gives you the estimate without executing.

If you truly need exact, filtered counts frequently, maintain a **summary table** or a materialized rollup updated by trigger or a periodic job. Don't recompute from scratch on every request.

---

## 9. Storing Comma-Separated Values in a Column

```sql
-- The anti-pattern
CREATE TABLE products (
    id     BIGINT PRIMARY KEY,
    name   TEXT,
    tags   TEXT   -- 'electronics,sale,featured'
);
```

The moment you write `WHERE tags LIKE '%sale%'` you've lost: no index, false matches (`on-sale` matches `sale`), no referential integrity, and painful updates. This violates first normal form. Model the relationship properly:

```sql
CREATE TABLE tags (
    id   BIGINT PRIMARY KEY,
    name TEXT UNIQUE
);

CREATE TABLE product_tags (
    product_id BIGINT REFERENCES products(id),
    tag_id     BIGINT REFERENCES tags(id),
    PRIMARY KEY (product_id, tag_id)
);
```

If the values are genuinely schemaless and belong together, PostgreSQL gives you a real array or JSONB with indexes to back them:

```sql
ALTER TABLE products ADD COLUMN tags TEXT[];
CREATE INDEX idx_products_tags ON products USING gin (tags);

SELECT * FROM products WHERE tags @> ARRAY['sale'];  -- index-backed containment
```

A CSV in a `TEXT` column is never the right answer.

---

## 10. Relying on Implicit Ordering

```sql
-- The anti-pattern: assumes rows come back "in order"
SELECT id, status FROM orders LIMIT 10;
```

SQL result order is **undefined** without an explicit `ORDER BY`. It might look sorted by primary key today because of a sequential scan, then change after a `VACUUM`, an index-only scan, or a parallel plan. Any query whose result you consume in a specific order — pagination, "latest N", top sellers — must say so:

```sql
SELECT id, status FROM orders
ORDER BY created_at DESC, id DESC   -- tiebreaker for determinism
LIMIT 10;
```

Note the tiebreaker. If `created_at` has duplicates, ordering by it alone is still non-deterministic across executions. Add a unique column (`id`) to make the order total and stable — this also makes keyset pagination correct.

---

## Bonus: A Few More Reviewers Flag Instantly

**Swallowing NULLs in aggregates without thinking.** `AVG(total_cents)` skips NULL rows; that may or may not be what you want. Be deliberate — `COALESCE` first if NULL should count as zero.

**`WHERE col = NULL`.** Never equals anything. Use `IS NULL`.

**Unbounded `IN` lists.** `WHERE id IN (... 10,000 literals ...)` blows up planning time. Pass an array and use `= ANY($1)`, or join against a `VALUES`/temp table.

**No `LIMIT` on exploratory or admin queries against big tables.** One `SELECT * FROM events` can pin a connection and spike memory.

**`SELECT DISTINCT` to paper over a bad join.** If you added `DISTINCT` because a join started duplicating rows, fix the join or use `EXISTS` — `DISTINCT` hides the real cardinality bug and sorts the entire result.

---

## The Reviewer's Mental Checklist

Before you approve (or submit) a query, run these in your head:

- Does it name its columns instead of `SELECT *`?
- Is every indexed column compared bare, with no function or cast wrapping it?
- Does any `LIKE` start with `%`, and if so is a trigram/FTS index behind it?
- Is there a query inside a loop that should be a join or a batched `IN`?
- Does pagination use keyset instead of large `OFFSET`?
- Any `NOT IN` over a subquery that could contain NULL?
- Is result order pinned with a deterministic `ORDER BY` plus tiebreaker?
- Are relationships modeled as tables/arrays, not CSV strings?

None of these require deep wizardry. They're the difference between SQL that demos and SQL that ships. Internalize the ten, and your queries stop being the thing that gets flagged — and start being the thing juniors copy.
