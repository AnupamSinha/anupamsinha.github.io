---
title: "JSON in SQL — Using PostgreSQL JSONB Like a Pro"
date: 2026-08-25
categories: [SQL, PostgreSQL]
tags: [sql, postgresql, mysql, databases, jsonb, json]
description: "Store, query, index, and mutate semi-structured data inside PostgreSQL — operators, GIN indexes, JSONPath, mutation, and when not to use JSONB."
---

## Why JSONB Exists

Sometimes your data is genuinely relational, and sometimes it's a bag of attributes whose shape you don't fully control — a webhook payload, a feature-flag blob, user preferences, an event envelope. Historically you either forced it into rigid columns or reached for a separate document database. PostgreSQL's `JSONB` type lets you keep that semi-structured data *inside* your relational database, query it with real operators, and index it for speed.

The key distinction up front: PostgreSQL has two JSON types.

- **`json`** stores the exact text you gave it, whitespace and duplicate keys included. It reparses on every access. Use it only when you must preserve the original document byte-for-byte.
- **`jsonb`** stores a decomposed binary form. It's slightly slower to insert, much faster to query, deduplicates keys, doesn't preserve key order or whitespace, and — crucially — supports GIN indexing.

For essentially all real work, use `jsonb`. This whole guide is about `jsonb`.

---

## Sample Data

```sql
CREATE TABLE events (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    payload    JSONB NOT NULL
);

INSERT INTO events (payload) VALUES
('{"type": "signup", "user": {"id": 1, "plan": "pro", "email": "a@x.com"}, "tags": ["web", "eu"]}'),
('{"type": "purchase", "user": {"id": 2, "plan": "free", "email": "b@x.com"}, "amount": 4200, "tags": ["ios"]}'),
('{"type": "signup", "user": {"id": 3, "plan": "pro", "email": "c@x.com"}, "tags": ["web", "us"]}'),
('{"type": "purchase", "user": {"id": 1, "plan": "pro", "email": "a@x.com"}, "amount": 900, "tags": ["web"]}');
```

---

## The Access Operators: `->`, `->>`, `#>`, `#>>`

This is the first thing to burn into memory, because mixing them up is the most common beginner mistake.

```sql
SELECT
    payload -> 'user'            AS user_json,     -- returns JSONB
    payload -> 'user' ->> 'plan' AS plan_text,     -- returns TEXT
    payload ->> 'type'           AS type_text      -- returns TEXT
FROM events;
```

- `->` returns a **JSONB** value (an object, array, or scalar-as-json). Use it to keep drilling deeper.
- `->>` returns **TEXT** (the value extracted as a string). Use it at the leaf when you want a plain value to compare, display, or cast.

The rule: **chain with `->`, finish with `->>`.** `payload -> 'user' ->> 'email'` gives you the email as text.

For deep paths, the `#>` (returns JSONB) and `#>>` (returns TEXT) operators take a path array so you don't chain arrows:

```sql
SELECT
    payload #>  '{user, plan}'  AS plan_json,   -- JSONB
    payload #>> '{user, email}' AS email_text   -- TEXT
FROM events;
```

Array elements are addressed by integer index (0-based); negative indexes count from the end:

```sql
SELECT payload #>> '{tags, 0}'  AS first_tag,
       payload #>> '{tags, -1}' AS last_tag
FROM events;
```

---

## Filtering: Getting Types Right

Because `->>` yields text, you must cast when you want numeric or boolean comparisons:

```sql
-- WRONG: compares text to integer, or does string comparison
SELECT * FROM events WHERE payload ->> 'amount' > 1000;   -- string compare!

-- RIGHT: cast to a real number first
SELECT * FROM events WHERE (payload ->> 'amount')::numeric > 1000;
```

That string-comparison bug is subtle: `'900' > '1000'` is *true* lexicographically because `'9' > '1'`. Always cast before comparing non-text values.

Filtering on a nested text value:

```sql
SELECT id, payload ->> 'type' AS type
FROM events
WHERE payload -> 'user' ->> 'plan' = 'pro';
```

---

## The Containment Operator `@>` (And Why It's the Star)

`@>` asks "does the left JSONB contain the right JSONB?" It's the workhorse for filtering, and — critically — it can use a GIN index.

```sql
-- All events for a pro user
SELECT * FROM events WHERE payload @> '{"user": {"plan": "pro"}}';

-- All signup events
SELECT * FROM events WHERE payload @> '{"type": "signup"}';

-- Events tagged 'web' (array containment)
SELECT * FROM events WHERE payload @> '{"tags": ["web"]}';
```

Containment matches structure: `{"user": {"plan": "pro"}}` matches any row whose `payload` has a `user` object with `plan` equal to `"pro"`, regardless of the other keys. This is the idiomatic, index-friendly way to filter JSONB — prefer it over `->>` equality when you can.

Related operators worth knowing:

```sql
SELECT payload ? 'amount'            FROM events;  -- top-level key exists?
SELECT payload ?| ARRAY['amount','x'] FROM events; -- any of these keys?
SELECT payload ?& ARRAY['type','user'] FROM events; -- all of these keys?
```

---

## Indexing JSONB: This Is the Whole Point

Without an index, every JSONB query is a sequential scan. Two GIN strategies:

### 1. Default GIN — indexes everything

```sql
CREATE INDEX idx_events_payload ON events USING gin (payload);
```

Supports `@>`, `?`, `?|`, `?&`. Indexes every key and value, so it's flexible but larger.

### 2. `jsonb_path_ops` GIN — smaller and faster for containment

```sql
CREATE INDEX idx_events_payload_path
    ON events USING gin (payload jsonb_path_ops);
```

Only supports `@>` (containment), but produces a smaller index and faster containment lookups. If containment is all you need — and it usually is — prefer this.

### 3. Expression B-tree index — for a specific hot field

When you constantly filter or sort on one extracted value, a plain B-tree on the expression beats a GIN index:

```sql
CREATE INDEX idx_events_type ON events ((payload ->> 'type'));

-- Now this uses the B-tree, supports range/sort, equality
SELECT * FROM events WHERE payload ->> 'type' = 'purchase';

-- For numeric filtering/sorting, index the casted expression
CREATE INDEX idx_events_amount ON events (((payload ->> 'amount')::numeric));
```

Rule of thumb: **GIN (`jsonb_path_ops`) for flexible containment across the whole document; expression B-tree for one specific field you hit constantly.** Verify with `EXPLAIN ANALYZE` that your index is actually chosen.

```sql
EXPLAIN ANALYZE
SELECT * FROM events WHERE payload @> '{"type": "purchase"}';
-- look for "Bitmap Index Scan on idx_events_payload_path"
```

---

## JSONPath: SQL/JSON Path Expressions

PostgreSQL 12+ supports the SQL standard JSONPath language via `jsonb_path_query`, `@?`, and `@@`. This is where JSONB gets genuinely powerful for complex filters.

```sql
-- Extract every value at a path
SELECT jsonb_path_query(payload, '$.user.email') FROM events;

-- Does any element match a predicate? (@? = path exists / matches)
SELECT * FROM events
WHERE payload @? '$.tags[*] ? (@ == "web")';

-- Predicate over a numeric field with filter syntax
SELECT * FROM events
WHERE payload @@ '$.amount > 1000';
```

The `?( ... )` filter syntax inside a path lets you express conditions that would be awkward with plain operators. `jsonb_path_query_array` collects matches into a JSONB array; `jsonb_path_exists` returns a boolean.

```sql
-- All tags across matching events, as arrays
SELECT id, jsonb_path_query_array(payload, '$.tags[*]') AS tags
FROM events;
```

The `@?` and `@@` operators can use a default GIN index, though complex path predicates may still fall back to filtering — check the plan.

---

## Expanding JSON Into Rows

To join JSONB against relational data or aggregate over array elements, unnest with the set-returning functions.

```sql
-- One row per (event, tag)
SELECT e.id, tag
FROM events e,
     jsonb_array_elements_text(e.payload -> 'tags') AS tag;

-- Count events per tag
SELECT tag, COUNT(*)
FROM events e,
     jsonb_array_elements_text(e.payload -> 'tags') AS tag
GROUP BY tag
ORDER BY COUNT(*) DESC;
```

- `jsonb_array_elements(jsonb)` — elements as JSONB.
- `jsonb_array_elements_text(jsonb)` — elements as text.
- `jsonb_each(jsonb)` / `jsonb_each_text(jsonb)` — expand an object into `(key, value)` rows.
- `jsonb_object_keys(jsonb)` — just the keys.

Prefer the `LATERAL` form or the comma-join above; both work, `LATERAL` reads more clearly when the function depends on the row.

---

## Mutating JSONB

JSONB values are immutable, so "updating" means producing a new JSONB and writing it back.

```sql
-- Set / add a nested value (creates the path if missing with the true flag)
UPDATE events
SET payload = jsonb_set(payload, '{user, plan}', '"enterprise"', true)
WHERE id = 1;

-- Merge objects with || (right side wins on key conflicts)
UPDATE events
SET payload = payload || '{"reviewed": true}'
WHERE id = 2;

-- Remove a key with -
UPDATE events SET payload = payload - 'reviewed' WHERE id = 2;

-- Remove a nested path with #-
UPDATE events SET payload = payload #- '{user, email}' WHERE id = 3;

-- Append to an array
UPDATE events
SET payload = jsonb_set(payload, '{tags}', (payload -> 'tags') || '"new"'::jsonb)
WHERE id = 1;
```

`jsonb_set`'s third argument must itself be JSONB — note `'"enterprise"'` (a JSON string) not `'enterprise'`. The fourth argument (`create_missing`) controls whether a non-existent path is created.

To build JSONB from relational columns:

```sql
SELECT jsonb_build_object(
    'id', id,
    'type', payload ->> 'type',
    'ts', created_at
) FROM events;

-- Aggregate rows into a JSON array / object
SELECT jsonb_agg(payload) FROM events;
SELECT jsonb_object_agg(id, payload -> 'type') FROM events;
```

---

## Constraints on JSONB

You don't have to give up validation just because the column is flexible. `CHECK` constraints can enforce structure:

```sql
ALTER TABLE events ADD CONSTRAINT payload_has_type
    CHECK (payload ? 'type');

ALTER TABLE events ADD CONSTRAINT payload_type_valid
    CHECK (payload ->> 'type' IN ('signup', 'purchase', 'login'));

-- Enforce a numeric amount when present
ALTER TABLE events ADD CONSTRAINT amount_non_negative
    CHECK (NOT (payload ? 'amount') OR (payload ->> 'amount')::numeric >= 0);
```

A little validation at the boundary keeps a "schemaless" column from turning into a garbage dump.

---

## When NOT to Use JSONB

JSONB is a tool, not a lifestyle. Reach for real columns when:

- **The field is queried, filtered, or joined constantly.** A first-class column with a normal index is simpler and faster than extracting from JSONB every time.
- **The data has a stable, known shape.** If every row has the same ten fields, those are columns. Don't hide your schema inside a blob to avoid writing a migration.
- **You need foreign keys or strong referential integrity.** JSONB can't reference other tables.
- **Multiple consumers need to agree on the shape.** A JSONB grab-bag becomes an undocumented contract that drifts.

The sweet spot is genuinely variable, sparse, or evolving data: event payloads, per-tenant custom fields, third-party API responses, feature configs. Use JSONB for the parts that are actually flexible, and promote fields to real columns the moment they stabilize into something you query all the time.

---

## MySQL Note

MySQL has a `JSON` type with functions like `JSON_EXTRACT`, the `->` and `->>` shorthand, and functional indexes on generated columns. It's serviceable. But PostgreSQL's JSONB has a richer operator set, the powerful `@>` containment with GIN indexing, and native JSONPath predicates. If document-shaped data is central to your app, this is one of the clearer points in Postgres's favor.

---

## Quick Reference

**`->` / `->>`** — access returning JSONB / TEXT. Chain with `->`, finish with `->>`.

**`#>` / `#>>`** — access by path array, JSONB / TEXT.

**`@>`** — containment. The index-friendly filter. Prefer it.

**`?` / `?|` / `?&`** — key existence checks.

**GIN `jsonb_path_ops`** — small, fast index for containment. Your default.

**Expression B-tree** — for one hot extracted field, especially for ranges/sorts.

**`jsonb_set` / `||` / `-` / `#-`** — mutate by producing new JSONB.

**`jsonb_array_elements(_text)` / `jsonb_each`** — expand into rows.

**`CHECK` constraints** — keep the flexible column honest.

Used deliberately, JSONB gives you the flexibility of a document store without leaving Postgres — one database, one transaction, one backup. Used carelessly, it becomes a schema you can't see. The skill is knowing which fields deserve which treatment.
