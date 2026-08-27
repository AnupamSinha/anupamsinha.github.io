---
title: "PostgreSQL vs MySQL in 2026 — An Honest Comparison"
date: 2026-08-25
categories: [SQL, Databases]
tags: [sql, postgresql, mysql, databases, comparison, architecture]
description: "After a decade shipping both to production — where PostgreSQL wins, where MySQL wins, where the gap closed, and my default recommendation for a new backend in 2026."
---

## My Bias, Stated Up Front

I'll save you the suspense: for a greenfield backend in 2026, I reach for PostgreSQL by default. Not because MySQL is bad — it isn't, and I'll defend it below — but because Postgres's feature surface, correctness defaults, and extension ecosystem make it the safer bet across the widest range of workloads. That said, "it depends" is doing real work in this comparison. If you're running a read-heavy web app on managed infrastructure your team already knows, MySQL will serve you perfectly well for years.

This isn't a benchmark shootout. Synthetic benchmarks lie, and both databases are fast enough that your schema, indexes, and query patterns matter a thousand times more than the engine badge. This is about the decisions that actually bite you six months in.

---

## The Honest Summary Table

| Dimension | PostgreSQL | MySQL (InnoDB) |
|---|---|---|
| Data types | Rich: arrays, JSONB, ranges, custom types, enums | Practical, fewer native types |
| JSON | JSONB with GIN indexing, rich operators | JSON type, functional but weaker indexing story |
| Extensions | Huge (PostGIS, pgvector, TimescaleDB, ...) | Limited plugin surface |
| Full-text search | Built-in `tsvector`/`tsquery` | Built-in but simpler |
| Window functions / CTEs | Complete, recursive CTEs, `MATERIALIZED` | Present since 8.0, solid |
| Replication | Logical + physical, mature | Very mature, battle-tested at scale |
| Read scaling | Good, needs tooling | Excellent, huge operational knowledge base |
| Ecosystem / hosting | Excellent | Excellent, arguably more ubiquitous |
| Default correctness | Stricter, fewer surprises | Historically lax, much improved |

Now the parts that matter, with code.

---

## Where PostgreSQL Genuinely Wins

### 1. The Type System Is a Different League

Postgres treats types as a first-class extensibility point. Arrays, ranges, JSONB, `citext`, `uuid`, enums, and fully custom composite types are native and indexable.

```sql
-- Native arrays with GIN indexing
CREATE TABLE products (
    id   BIGINT PRIMARY KEY,
    tags TEXT[]
);
CREATE INDEX ON products USING gin (tags);
SELECT * FROM products WHERE tags @> ARRAY['sale'];

-- Range types: model a booking with an exclusion constraint that
-- prevents overlapping reservations for the same room — in the schema.
CREATE EXTENSION btree_gist;
CREATE TABLE bookings (
    room_id INT,
    during  TSTZRANGE,
    EXCLUDE USING gist (room_id WITH =, during WITH &&)
);
```

That `EXCLUDE` constraint enforces "no two bookings for the same room can overlap" at the database level. Replicating that in MySQL means application-level locking and a class of race conditions you now own. This kind of thing is why I keep coming back.

### 2. JSONB Is the Best Document Story in a Relational DB

```sql
CREATE TABLE events (id BIGINT PRIMARY KEY, payload JSONB);
CREATE INDEX ON events USING gin (payload);

-- Containment query, index-accelerated
SELECT * FROM events WHERE payload @> '{"type": "signup"}';

-- Extract, filter, and index on a nested path
SELECT payload->>'email' FROM events WHERE payload->'user'->>'plan' = 'pro';
```

MySQL has a JSON type and functional indexes, but the operator ergonomics and GIN indexing in Postgres are meaningfully ahead. If part of your data is genuinely semi-structured, this alone can decide it.

### 3. Extensions Turn Postgres Into a Platform

This is the quiet superpower. One database engine, many workloads:

- **pgvector** — vector similarity search for AI/RAG, no separate vector DB needed.
- **PostGIS** — the gold standard for geospatial, full stop.
- **TimescaleDB** — time-series at scale on top of plain Postgres.
- **pg_trgm** — trigram fuzzy/substring search.

```sql
-- Semantic search living right next to your relational data
CREATE EXTENSION vector;
CREATE TABLE docs (id BIGINT PRIMARY KEY, embedding vector(1536));
SELECT id FROM docs ORDER BY embedding <=> $1 LIMIT 5;  -- cosine distance
```

Being able to add vector search or geospatial without introducing a new datastore is an architectural win that compounds.

### 4. Correctness Defaults

Postgres tends to refuse ambiguous or lossy operations rather than silently guessing. Transactional DDL is a standout: wrap schema changes in a transaction and roll the whole thing back on failure.

```sql
BEGIN;
ALTER TABLE orders ADD COLUMN discount_cents BIGINT NOT NULL DEFAULT 0;
CREATE INDEX idx_orders_status ON orders (status);
-- something fails here?
ROLLBACK;  -- schema is fully restored, no half-applied migration
```

MySQL's DDL is largely non-transactional — a failed migration can leave you in a partially-applied state you clean up by hand. If you've ever done that at 2am, you feel this one.

---

## Where MySQL Genuinely Wins

I refuse to write a one-sided piece, because in practice MySQL earns its place.

### 1. Operational Familiarity and Read Scaling at Scale

MySQL has run some of the largest web properties on earth for two decades. The operational playbook — primary/replica topologies, read replicas, failover, sharding via Vitess — is deep, documented, and battle-tested. If you're building the next high-traffic read-heavy consumer app and your ops team has scars from running MySQL, that institutional knowledge is worth more than a feature checklist.

```sql
-- Read-replica routing is a first-class, well-trodden pattern.
-- Vitess makes horizontal sharding of MySQL a solved (if involved) problem
-- at a scale most teams never reach.
```

### 2. Replication Maturity

MySQL replication is older and, for classic topologies, extremely well understood. Postgres logical replication has matured a lot and is excellent now, but the sheer volume of MySQL replication experience in the wild means fewer novel surprises when things go sideways.

### 3. Simplicity as a Feature

MySQL's smaller surface area can be an advantage. Fewer knobs, fewer types, fewer ways to shoot yourself. For a team that wants a relational database that just stores rows and serves them fast, the reduced cognitive load is real. Not every project needs range types and custom operators.

### 4. It Closed Most of the Historical Gaps

The old jokes are stale. MySQL 8.0+ has:

```sql
-- Window functions
SELECT id, RANK() OVER (PARTITION BY customer_id ORDER BY total DESC) FROM orders;

-- Recursive CTEs
WITH RECURSIVE nums AS (
    SELECT 1 AS n UNION ALL SELECT n + 1 FROM nums WHERE n < 10
) SELECT * FROM nums;

-- CHECK constraints (actually enforced now)
CREATE TABLE t (age INT CHECK (age >= 0));
```

Window functions, CTEs, enforced CHECK constraints, a real JSON type, and better defaults. If your only reason for avoiding MySQL is "it doesn't have CTEs," update your mental model.

---

## Head-to-Head on the Things People Actually Ask

### Full-Text Search

Postgres ships full-text as a native, composable feature:

```sql
CREATE INDEX idx_docs_fts ON docs USING gin (to_tsvector('english', body));
SELECT * FROM docs
WHERE to_tsvector('english', body) @@ plainto_tsquery('english', 'query terms');
```

MySQL has `FULLTEXT` indexes on InnoDB and they work fine for basic needs. For anything sophisticated, both teams often end up putting Elasticsearch/OpenSearch alongside — so treat built-in FTS as "good enough for many apps" on both, with Postgres a bit more capable out of the box.

### Concurrency and MVCC

Both use MVCC. The practical difference: Postgres keeps dead row versions in the table and reclaims them via `VACUUM` (autovacuum handles it, but you must understand it exists, and bloat/`VACUUM` tuning is a real operational topic). MySQL/InnoDB stores undo separately, so the "vacuum" concept doesn't surface the same way. Neither is strictly better; they're different failure modes you need to know.

### JSON, Again

If more than a slice of your data is document-shaped, Postgres JSONB wins clearly. If it's occasional metadata, both are fine.

### Upserts

```sql
-- PostgreSQL: expressive, with access to the conflicting row
INSERT INTO counters (key, n) VALUES ('hits', 1)
ON CONFLICT (key) DO UPDATE SET n = counters.n + EXCLUDED.n;

-- MySQL: also solid
INSERT INTO counters (key, n) VALUES ('hits', 1)
ON DUPLICATE KEY UPDATE n = n + VALUES(n);
```

Both handle upserts well. Postgres's `ON CONFLICT` is a little more expressive (you can target specific constraints and reference `EXCLUDED`), but this is a wash for most uses.

---

## The Gotchas I'd Warn a Teammate About

**PostgreSQL:**
- Connection overhead is real. Each connection is a process; hundreds of direct app connections will hurt. Put **PgBouncer** in front for connection pooling — treat this as mandatory at scale, not optional.
- Understand autovacuum and table bloat before you're debugging it in production.
- Major-version upgrades historically needed `pg_upgrade` planning (logical replication makes near-zero-downtime upgrades feasible now, but plan for it).

**MySQL:**
- Watch default collations and character sets — use `utf8mb4`, never legacy `utf8` (which is a 3-byte trap that can't store all of Unicode, including many emoji).
- Historically lax modes could truncate or coerce data silently. Run in strict SQL mode and verify it.
- DDL is largely non-transactional — a failed migration can leave partial state.

---

## My Actual Recommendation

For a **new backend in 2026**, default to **PostgreSQL**. The richer type system, JSONB, the extension ecosystem (especially pgvector if there's any chance AI features land on your roadmap), transactional DDL, and stricter correctness defaults reduce the number of problems you'll have to solve yourself. It's the choice that keeps the most doors open.

Choose **MySQL** when: your team has deep MySQL operational expertise, you're building a read-heavy workload where the mature replication/sharding playbook (including Vitess) is a direct asset, or your managed platform and tooling are already MySQL-centric and switching buys you nothing.

What I would **not** do is pick based on stale reputation. "MySQL can't do CTEs" and "Postgres is slow" are both years out of date. Pick based on your workload's actual shape, your team's actual scars, and where you want your architecture to be able to grow — and then invest in knowing whichever engine you chose deeply. The engine you understand always beats the engine that merely looks better on paper.
