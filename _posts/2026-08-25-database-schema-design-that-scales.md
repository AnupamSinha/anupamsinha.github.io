---
title: "How I Design Database Schemas That Scale — Normalization, Denormalization, and When to Break the Rules"
date: 2026-08-25
categories: [SQL, Architecture]
tags: [sql, postgresql, mysql, databases, schema-design, normalization]
description: "Normalize until it hurts, denormalize until it works — an opinionated, code-heavy field guide to schema design that stays an asset instead of a liability."
---

## The Only Schema Philosophy That Has Never Let Me Down

Normalize until it hurts, then denormalize until it works.

That's the whole thing. Start clean and correct, prove where it's actually slow with real data, and denormalize surgically where the numbers justify it. Everything below is the detail behind that sentence. I've watched teams skip the first half ("we'll need speed") and drown in data-integrity bugs, and I've watched teams cling to textbook third normal form while a report times out. The craft is knowing which half you're in right now.

This is PostgreSQL-first. The principles are universal; I note MySQL differences where they change a decision.

---

## Start With Normalization — And Actually Understand It

Normalization isn't academic ceremony. Each normal form eliminates a specific class of anomaly — the update, insert, and delete bugs that come from storing the same fact in two places.

### The anti-pattern: one wide table

```sql
-- DON'T: everything jammed into orders
CREATE TABLE orders (
    id             BIGINT PRIMARY KEY,
    customer_name  TEXT,      -- repeated on every order
    customer_email TEXT,      -- repeated, and now which one is right?
    product_names  TEXT,      -- 'Widget, Gadget' — CSV, ungueryable
    total          NUMERIC
);
```

Three bugs are baked in. Change a customer's email and you must update every order row (update anomaly). You can't store a customer who hasn't ordered yet (insert anomaly). Delete their last order and the customer vanishes (delete anomaly). Plus the CSV product list is unqueryable and unjoinable.

### The normalized version

```sql
CREATE TABLE customers (
    id    BIGINT PRIMARY KEY,
    name  TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE
);

CREATE TABLE products (
    id    BIGINT PRIMARY KEY,
    name  TEXT NOT NULL,
    price_cents BIGINT NOT NULL
);

CREATE TABLE orders (
    id          BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
    order_id   BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity   INT NOT NULL CHECK (quantity > 0),
    price_cents BIGINT NOT NULL,   -- price AT TIME OF ORDER (see below)
    PRIMARY KEY (order_id, product_id)
);
```

Each fact lives in exactly one place. The normal forms in one breath:

- **1NF** — atomic values, no repeating groups (no CSV columns, no `phone1/phone2/phone3`).
- **2NF** — no partial dependency on part of a composite key.
- **3NF** — no transitive dependency; non-key columns depend only on the key.

You'll spend 95% of your life targeting **3NF**. That's the sweet spot: correct, flexible, and fast enough for the overwhelming majority of workloads.

Note one deliberate exception already: `order_items.price_cents`. Price belongs to `products`, so copying it looks like a normalization violation. It isn't — it's capturing a **point-in-time fact**. The price when the order was placed is genuinely different data from the current price. When the product price changes next week, historical orders must not change. Recognizing "current value vs. value-at-event" is one of the most important modeling instincts you can develop.

---

## Get the Fundamentals Right Before Anything Clever

Scaling problems are usually fundamentals problems in disguise. Nail these and most performance issues never appear.

### Choose keys deliberately

```sql
-- BIGINT identity: compact, sequential, index-friendly
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY

-- UUID: globally unique, good for distributed/merge scenarios
-- but random UUIDv4 fragments index locality on inserts.
-- Prefer UUIDv7 (time-ordered) if you need UUIDs at scale.
id UUID PRIMARY KEY DEFAULT gen_random_uuid()
```

My default is `BIGINT` identity unless I have a concrete reason for UUIDs (client-generated IDs, multi-region merges, not leaking row counts). If you do need UUIDs at high insert volume, use a time-ordered variant (UUIDv7) so you don't shred B-tree locality with random inserts.

### Constrain your data at the schema level

The database is your last line of defense for integrity — use it. Application checks get bypassed by scripts, migrations, and the next service.

```sql
ALTER TABLE order_items ADD CONSTRAINT qty_positive CHECK (quantity > 0);
ALTER TABLE customers   ALTER COLUMN email SET NOT NULL;
ALTER TABLE orders      ADD CONSTRAINT fk_customer
    FOREIGN KEY (customer_id) REFERENCES customers(id);
```

A schema that enforces its own invariants is one you can trust at 3am. I'd rather a bad write fail loudly at the database than corrupt data silently pass.

### Pick the right types

Use `NUMERIC` for money (never `FLOAT` — binary floating point can't represent `0.10` exactly). Use `TIMESTAMPTZ`, not `TIMESTAMP`, and store UTC. Use native `BOOLEAN`, `TEXT` over arbitrary `VARCHAR(n)` limits in Postgres, and real `ENUM`/lookup tables over magic strings.

```sql
amount     NUMERIC(12,2)  -- money: exact
created_at TIMESTAMPTZ    -- always timezone-aware, store UTC
status     TEXT NOT NULL  -- backed by a CHECK or a lookup FK
```

---

## Index for the Queries You Actually Run

An index is a bet that a query will filter or sort on those columns. Place indexes to match real access patterns, not hunches.

```sql
-- Foreign keys: index them. Postgres does NOT auto-index FKs.
CREATE INDEX idx_orders_customer ON orders (customer_id);

-- Composite index matching a filter + sort. Column order matters:
-- equality column first, then the range/sort column.
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at DESC);

-- Partial index: only the rows you query, smaller and faster
CREATE INDEX idx_orders_open ON orders (created_at)
    WHERE status = 'open';
```

Two things people miss: PostgreSQL does **not** automatically index foreign keys (MySQL/InnoDB does) — unindexed FKs make joins and cascading deletes slow. And every index is a write-time tax; don't index columns you never filter on. Measure with `EXPLAIN ANALYZE`, don't guess.

---

## Now — When to Break the Rules

Here's my actual, opinionated position: **denormalize only when you have proof, and always make the redundant data self-maintaining.** Premature denormalization is how you get fast wrong answers.

### Signal 1: A join is genuinely too expensive at read time

You're computing an order total by summing `order_items` on every read, and it's a proven hot path. Cache the aggregate — but keep it honest with a trigger, not by trusting the application to remember:

```sql
ALTER TABLE orders ADD COLUMN total_cents BIGINT NOT NULL DEFAULT 0;

CREATE OR REPLACE FUNCTION refresh_order_total() RETURNS TRIGGER AS $$
BEGIN
    UPDATE orders o
    SET total_cents = (
        SELECT COALESCE(SUM(quantity * price_cents), 0)
        FROM order_items WHERE order_id = o.id
    )
    WHERE o.id = COALESCE(NEW.order_id, OLD.order_id);
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_order_total
AFTER INSERT OR UPDATE OR DELETE ON order_items
FOR EACH ROW EXECUTE FUNCTION refresh_order_total();
```

The rule that keeps denormalization safe: **redundant data must have a single, automatic source of truth for staying in sync.** A trigger, a materialized view, or a well-guarded write path — never "the app will update both." The moment two code paths can write the same fact, they will drift.

### Signal 2: Read-heavy reporting that hammers the same aggregation

Reach for a **materialized view**. It's denormalization the database manages for you.

```sql
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT date_trunc('day', created_at) AS day,
       SUM(total_cents) AS revenue_cents,
       COUNT(*)         AS order_count
FROM orders
GROUP BY 1;

CREATE UNIQUE INDEX ON daily_revenue (day);   -- enables CONCURRENTLY

-- Refresh on a schedule; CONCURRENTLY avoids locking readers
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;
```

This is my favorite denormalization tool because it's explicit, isolated, and trivially rebuildable — if it's ever wrong, you `REFRESH` and it's correct again. MySQL has no native materialized views; you emulate with a summary table plus scheduled `INSERT ... SELECT` or triggers.

### Signal 3: Counters and feeds

Follower counts, like counts, activity feeds — recomputing from scratch per request doesn't scale. Maintain a counter column updated in the same transaction as the event, or precompute the feed. This is legitimate denormalization for a specific, measured access pattern. Even here, prefer a scheduled reconciliation job that recomputes the true value periodically, so drift self-heals.

### The forms of denormalization, ranked by how much I trust them

1. **Materialized views** — most trustworthy; the DB owns correctness, rebuild anytime.
2. **Trigger-maintained aggregates/counters** — trustworthy if the trigger is the sole writer.
3. **Copied columns for point-in-time facts** — not really denormalization; just correct modeling.
4. **Hand-maintained duplicated data in application code** — the riskiest; avoid unless you've exhausted the above.

---

## Design for Change, Because Requirements Will Change

The schemas that age well leave room to grow.

### Soft-delete when history matters

```sql
ALTER TABLE customers ADD COLUMN deleted_at TIMESTAMPTZ;

-- A partial unique index that only applies to live rows
CREATE UNIQUE INDEX idx_customers_email_live
    ON customers (email) WHERE deleted_at IS NULL;
```

That partial unique index is a lovely Postgres trick: emails must be unique among *active* customers, but a deleted customer can re-register. Try expressing that cleanly without partial indexes.

### Migrations must be safe on a live table

An opinion I'll die on: **never run a migration that takes a long exclusive lock on a hot table.** Adding a nullable column is instant in modern Postgres. Adding a `NOT NULL` column with a default is fast in Postgres 11+ (metadata-only). But backfilling or adding a constraint that scans the table should be done in stages:

```sql
-- Add validating a constraint WITHOUT a long lock:
ALTER TABLE orders ADD CONSTRAINT chk_total
    CHECK (total_cents >= 0) NOT VALID;      -- fast, doesn't scan
ALTER TABLE orders VALIDATE CONSTRAINT chk_total;  -- scans, but doesn't block writes
```

Build indexes on live tables with `CREATE INDEX CONCURRENTLY`. In MySQL, lean on online DDL (`ALGORITHM=INPLACE`) or tools like `pt-online-schema-change` / `gh-ost`. Schema design includes *how you deploy the schema*, not just its final shape.

---

## When Relational Isn't the Answer

Being opinionated cuts both ways. Some data doesn't want to be normalized rows:

- **Genuinely schemaless, per-tenant custom fields** → `JSONB` column, indexed with GIN. Keep the *stable* fields as real columns; only the variable part goes in JSONB.
- **Deeply hierarchical / graph-shaped data** → recursive CTEs handle trees fine, but a true graph workload may want `ltree`, closure tables, or a graph store.
- **Massive append-only time-series** → partition by time (native Postgres declarative partitioning) or use TimescaleDB.

```sql
-- Time partitioning: keeps indexes small and enables cheap old-data drops
CREATE TABLE events (id BIGINT, created_at TIMESTAMPTZ, payload JSONB)
    PARTITION BY RANGE (created_at);
CREATE TABLE events_2026_08 PARTITION OF events
    FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
```

Partitioning is the scaling lever people forget: instead of one 500M-row table, you get monthly children with small indexes, and archiving old data is a `DROP TABLE` instead of a massive `DELETE`.

---

## My Checklist Before I Call a Schema Done

- Is it in 3NF, with each fact stored once?
- Are point-in-time facts (prices, addresses at time of event) captured, not looked up live?
- Are all foreign keys defined **and indexed**?
- Are invariants enforced by constraints (`NOT NULL`, `CHECK`, `UNIQUE`, FKs), not just app code?
- Right types — `NUMERIC` for money, `TIMESTAMPTZ` for time?
- Do indexes match the real query patterns, verified with `EXPLAIN ANALYZE`?
- Is every piece of denormalized data self-maintaining (trigger / materialized view), with a reconciliation path?
- Can the schema be migrated on a live table without a long lock?
- Have I resisted denormalizing anything I can't prove is slow?

Design the clean, normalized model first — it's easier to denormalize a correct schema than to correct a fast one. Let real workloads, not anxiety, tell you where to bend the rules. And whenever you do bend them, make the database responsible for keeping the redundancy honest. Do that, and your schema stays an asset instead of quietly becoming the thing everyone is afraid to touch.
