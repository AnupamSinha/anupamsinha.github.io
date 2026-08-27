---
title: "Database Indexing Explained — B-Trees, When to Index, and When It Hurts"
date: 2026-08-25
categories: [SQL, Performance]
tags: [sql, postgresql, performance, indexing, b-tree, tutorial]
description: "How the B-tree works, why an index turns O(n) into O(log n), and the cases where adding an index actually makes your database slower."
---
## An Index Is a Data Structure, Not Magic

Most developers treat indexes as a checkbox: query slow, add index, query fast. That works until it doesn't — until the index is ignored, or writes crawl, or the "obvious" index made no difference. To use indexes well you need to know what they physically are and what they cost.

An index is a separate, ordered data structure that lets the database find rows without scanning the whole table. The default in PostgreSQL, MySQL, SQL Server, and Oracle is the **B-tree** (more precisely a B+tree). Almost everything in this post is about B-trees, because that's what you'll use 95% of the time.

We'll use this table throughout:

```sql
CREATE TABLE users (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       TEXT NOT NULL,
    country     TEXT NOT NULL,
    status      TEXT NOT NULL,      -- 'ACTIVE','SUSPENDED','DELETED'
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO users (email, country, status, created_at)
SELECT 'user' || g || '@example.com',
       (ARRAY['SG','IN','US','DE','JP'])[1 + (g % 5)],
       (ARRAY['ACTIVE','ACTIVE','ACTIVE','SUSPENDED','DELETED'])[1 + (g % 5)],
       now() - (random() * interval '1000 days')
FROM generate_series(1, 2000000) g;

ANALYZE users;
```

---

## How a B-Tree Actually Works

A B+tree is a balanced tree of pages (nodes). Each page holds many sorted keys. Internal pages point to child pages; leaf pages hold the keys plus a pointer to the actual row (in PostgreSQL, a `ctid` heap location; in InnoDB, the primary key).

Picture a B-tree on an integer column with 2 million rows:

```
                       [ Root page ]
                     500k        1.5M
                    /             \
        [ Internal ]               [ Internal ]
       100k  ...  400k            1.0M  ...  1.9M
        /            \             /            \
   [ Leaf pages: sorted keys -> row pointers, linked left-to-right ]
   ...  42  43  44 ...        ...  1500123 ... 
```

To find `id = 1500123`, the database:

1. Reads the root page, compares, follows the pointer toward the 1.5M subtree.
2. Reads one internal page, narrows the range.
3. Reads one leaf page, finds the key, follows the pointer to the row.

That's **3–4 page reads** to locate a row among 2 million. A sequential scan would read tens of thousands of pages. This is the difference between O(log n) and O(n).

Two consequences fall out of the structure:

- **Leaf pages are linked in sorted order.** That's why a B-tree can serve range queries (`BETWEEN`, `>`, `<`) and `ORDER BY` without a separate sort — it just walks the leaves.
- **The tree stays shallow.** With ~hundreds of keys per page, even a billion rows is only 4–5 levels deep. Index lookups stay fast as tables grow.

```sql
CREATE INDEX idx_users_id_demo ON users (id);  -- (the PK already has one)

EXPLAIN ANALYZE SELECT * FROM users WHERE id = 1500123;
```

```
Index Scan using users_pkey on users  (cost=0.43..8.45 rows=1 width=64)
                                       (actual time=0.021..0.022 rows=1 loops=1)
  Index Cond: (id = 1500123)
Execution Time: 0.041 ms
```

Four page reads, 0.04 ms. That's the B-tree earning its keep.

---

## Selectivity — The One Number That Decides Everything

An index helps when a query is **selective**: it returns a small fraction of the table. If a predicate matches most rows, an index scan (random I/O, one lookup per row) is *slower* than reading the table sequentially.

Estimate selectivity as `matching_rows / total_rows`:

```sql
SELECT status, count(*), round(count(*) * 100.0 / sum(count(*)) OVER (), 1) AS pct
FROM users GROUP BY status;
```

```
  status   | count  | pct
-----------+--------+------
 ACTIVE    | 1200000| 60.0
 SUSPENDED |  400000| 20.0
 DELETED   |  400000| 20.0
```

An index on `status` is nearly useless for `WHERE status = 'ACTIVE'` (60% of rows) — PostgreSQL will pick a `Seq Scan`. But `WHERE email = 'user42@example.com'` matches one row out of two million: extremely selective, perfect for an index.

This is why **indexing a boolean or a low-cardinality enum rarely helps** on its own. High-cardinality columns (email, UUID, timestamps used with ranges) are where B-trees shine.

---

## Composite Indexes and the Leftmost-Prefix Rule

A composite index `(a, b, c)` sorts by `a`, then `b` within equal `a`, then `c`. This is exactly like sorting a phone book by last name, then first name. You can look people up by last name, or by last + first name — but you cannot use that ordering to find everyone with a given *first* name.

```sql
CREATE INDEX idx_users_country_status_created
    ON users (country, status, created_at);
```

This index can serve:

```sql
-- leading column
WHERE country = 'SG'
-- leading + second
WHERE country = 'SG' AND status = 'ACTIVE'
-- leading + second + third range
WHERE country = 'SG' AND status = 'ACTIVE' AND created_at > now() - interval '30 days'
-- and ORDER BY that matches the index order
WHERE country = 'SG' ORDER BY status, created_at
```

It **cannot** efficiently serve:

```sql
WHERE status = 'ACTIVE'                 -- skips leading column
WHERE created_at > now() - interval '1 day'   -- skips two leading columns
```

The design rule, in order:

1. **Equality columns first** (`=` predicates).
2. **Then one range/sort column.**
3. Stop after the first range column — columns after a range predicate can't be used for further filtering in the index, only the leftmost prefix up to and including the range is useful.

```sql
-- Good for: filter by status (=), then range/sort by created_at
CREATE INDEX idx_users_status_created ON users (status, created_at DESC);

EXPLAIN ANALYZE
SELECT * FROM users
WHERE status = 'SUSPENDED' AND created_at >= now() - interval '7 days'
ORDER BY created_at DESC
LIMIT 50;
```

```
Limit
  ->  Index Scan using idx_users_status_created on users
        Index Cond: ((status = 'SUSPENDED') AND (created_at >= ...))
```

One index, filter and sort both satisfied, no separate sort step.

---

## Covering Indexes and Index-Only Scans

Normally an index tells the database *where* a row is; it then fetches the row from the heap. If the index already contains every column the query needs, it can skip the heap entirely — an **index-only scan**.

```sql
-- Include the payload column so the query never touches the table
CREATE INDEX idx_users_country_cover
    ON users (country) INCLUDE (email, status);

EXPLAIN (ANALYZE, BUFFERS)
SELECT email, status FROM users WHERE country = 'JP';
```

```
Index Only Scan using idx_users_country_cover on users
  Index Cond: (country = 'JP')
  Heap Fetches: 0
  Buffers: shared hit=...
```

`Heap Fetches: 0` is the goal. `INCLUDE` columns are stored only in leaf pages and aren't part of the sort key, so they don't bloat the tree navigation but do let the index answer the query alone.

For index-only scans to work in PostgreSQL, the table's visibility map must be current — run `VACUUM` so the planner knows the pages are all-visible. If `Heap Fetches` is high, that's usually the reason.

**MySQL note:** In InnoDB every secondary index implicitly includes the primary key, and a secondary index covers a query if it contains all referenced columns. `EXPLAIN` shows `Using index` in the Extra column for a covering scan.

---

## When to Add an Index

Add an index when:

- The column appears in a `WHERE`, `JOIN ... ON`, or `ORDER BY` and the predicate is **selective**.
- A foreign key column is used to join or filter — PostgreSQL does **not** auto-index FK columns, and missing FK indexes cause slow joins and slow cascading deletes.
- You have a frequent `ORDER BY ... LIMIT` that currently sorts on disk.
- A composite `(equality, range)` pattern shows up repeatedly.

Find missing-index candidates from actual traffic:

```sql
-- Sequential scans on large tables are suspects
SELECT relname, seq_scan, seq_tup_read, idx_scan,
       seq_tup_read / NULLIF(seq_scan, 0) AS avg_rows_per_seq_scan
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 20;
```

A table with millions of `seq_tup_read` and few `idx_scan` is telling you it's being scanned repeatedly.

---

## When Indexing Hurts

This is the half of the topic most guides skip. Indexes are not free.

### 1. Every index slows writes

An index must be updated on every `INSERT`, `UPDATE` (of an indexed column), and `DELETE`. A table with 8 indexes does roughly 9x the write work. On a write-heavy table, over-indexing tanks throughput.

```sql
-- Find indexes that are never used and only cost you writes
SELECT s.relname AS table, i.indexrelname AS index,
       s.idx_scan AS times_used,
       pg_size_pretty(pg_relation_size(i.indexrelid)) AS size
FROM pg_stat_user_indexes i
JOIN pg_stat_user_tables s ON s.relid = i.relid
WHERE s.idx_scan IS NOT NULL
ORDER BY s.idx_scan ASC, pg_relation_size(i.indexrelid) DESC;
```

An index with `times_used = 0` after weeks of production traffic is pure overhead. Drop it.

### 2. Indexes take space and memory

A B-tree on a 2M-row table can be hundreds of MB. That's disk, and worse, it competes for the buffer cache. Unused indexes evict useful pages.

### 3. Low-cardinality indexes rarely earn their cost

An index on `status` (3 distinct values) costs writes and space but is usually ignored by the planner. If you *do* need it, a **partial index** is far better (next section).

### 4. Redundant and overlapping indexes

If you have `(country, status)`, a separate index on `(country)` is redundant — the composite already serves leading-column queries. Keep the composite, drop the single-column one.

### 5. Random-UUID primary keys fragment the tree

Inserting random UUIDs as a clustered/primary key scatters writes across the whole B-tree, causing page splits and bloat. Prefer sequential keys (bigint identity, or UUIDv7 which is time-ordered) for insert-heavy tables.

---

## Partial Indexes — Index Only What You Query

If you always query a small subset, index only that subset. Smaller index, cheaper writes, and it stays in cache.

```sql
-- You only ever query suspended/deleted accounts for admin review
CREATE INDEX idx_users_flagged
    ON users (created_at)
    WHERE status IN ('SUSPENDED', 'DELETED');

EXPLAIN ANALYZE
SELECT * FROM users
WHERE status = 'SUSPENDED' AND created_at >= now() - interval '30 days';
```

```
Index Scan using idx_users_flagged on users
  Index Cond: (created_at >= ...)
  Filter: (status = 'SUSPENDED'::text)
```

The index only contains ~40% of rows (the flagged ones) and none of the 60% `ACTIVE` rows, so it's smaller and untouched by writes to active accounts. Partial indexes are one of PostgreSQL's most underused features.

**MySQL note:** MySQL does not support partial (filtered) indexes. SQL Server does, via `CREATE INDEX ... WHERE`.

---

## Beyond B-Trees — Know They Exist

- **GIN** — for `jsonb`, arrays, and full-text search. Indexes the *contents* of a composite value. Use for `WHERE tags @> ARRAY['x']` or `tsvector` search.
- **GiST** — for geometric data and ranges (e.g., PostGIS, `tstzrange` overlap).
- **BRIN** — tiny index for huge, naturally-ordered tables (append-only time-series). Stores min/max per block range instead of per row. Massive tables, minimal space.
- **Hash** — equality-only. Rarely worth it over a B-tree in PostgreSQL.

```sql
-- GIN for jsonb containment
CREATE INDEX idx_users_meta_gin ON users USING gin (metadata jsonb_path_ops);

-- BRIN for an append-only events table ordered by time
CREATE INDEX idx_events_created_brin ON events USING brin (created_at);
```

Reach for these only when the B-tree can't express the query. For scalar equality, range, and sort — B-tree.

---

## A Practical Indexing Workflow

1. **Don't guess. Measure.** Use `pg_stat_statements` to find the queries that cost the most total time.
2. **Read the plan** with `EXPLAIN (ANALYZE, BUFFERS)`. Confirm the query is doing a `Seq Scan` where a selective index would help.
3. **Design the index for the query**, not the column — equality columns first, one range column, `INCLUDE` the returned columns if you can make it index-only.
4. **Create it without downtime**: `CREATE INDEX CONCURRENTLY`.
5. **Verify the planner uses it** and measure the before/after.
6. **Audit periodically**: drop unused and redundant indexes.

```sql
-- Non-blocking index build on a live table
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);
```

Note: `CONCURRENTLY` can't run inside a transaction block and takes longer, but it doesn't lock out writes.

---

## Final Thought

An index is a sorted data structure that trades write cost and storage for read speed. That trade is worth it when a query is selective and runs often; it's a loss when the column has low cardinality, the index is never used, or the table is write-heavy and over-indexed.

Learn the B-tree, respect the leftmost-prefix rule, design indexes around your real queries, and audit for the ones that only cost you. The goal isn't more indexes — it's the *right* indexes.
