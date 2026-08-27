---
title: "Database Transactions and Isolation Levels — Explained with Race Conditions"
date: 2026-08-25
categories: [SQL, System Design]
tags: [sql, postgresql, databases, transactions, isolation, system-design]
description: "Reproduce dirty reads, non-repeatable reads, phantoms, and write skew with real SQL — and learn exactly which isolation level fixes each."
---

## Isolation Is a Dial, and the Default Isn't the Safest

A transaction groups statements into one atomic unit. But when many transactions run at once, **isolation** decides how much of each other they can see. This is the setting most developers never touch — and the source of the nastiest, least-reproducible production bugs.

The SQL standard defines four isolation levels by which **anomalies** they forbid:

```
Level             Dirty Read  Non-repeatable Read  Phantom Read
READ UNCOMMITTED   possible*    possible             possible
READ COMMITTED     no           possible             possible
REPEATABLE READ    no           no                   possible (std) / no (PG)
SERIALIZABLE       no           no                   no
```

The trap: **Read Committed is the default in PostgreSQL** (and in most setups), which means the database will happily let non-repeatable reads and phantoms through unless you ask for more. Higher isolation is safer but costs concurrency and introduces retries. Let's reproduce each anomaly, then fix it.

Setup:

```sql
CREATE TABLE accounts (
    id      BIGINT PRIMARY KEY,
    owner   TEXT NOT NULL,
    balance INT  NOT NULL
);
INSERT INTO accounts VALUES (1, 'alice', 100), (2, 'alice', 100);
```

Every example below uses **two concurrent `psql` sessions**, labeled T1 and T2.

---

## Anomaly 1 — Dirty Read

A dirty read is seeing another transaction's **uncommitted** changes. If that transaction later rolls back, you acted on data that never existed.

```sql
-- T1
BEGIN;
UPDATE accounts SET balance = 500 WHERE id = 1;   -- not committed yet
-- ... pause ...

-- T2 (if it could dirty-read)
SELECT balance FROM accounts WHERE id = 1;   -- would see 500 (uncommitted!)

-- T1
ROLLBACK;   -- the 500 never happened; T2 acted on a phantom value
```

**PostgreSQL never allows dirty reads** — even at its lowest level, `READ UNCOMMITTED` behaves like `READ COMMITTED`. So T2 always sees the last *committed* value (100). This anomaly is essentially unreachable in Postgres, which is why the table above marks it `possible*` — the standard permits it, Postgres does not.

---

## Anomaly 2 — Non-Repeatable Read

You read a row, someone else commits an update to it, you read it again in the *same* transaction and get a different value. The read was not repeatable.

```sql
-- T1 (READ COMMITTED, the default)
BEGIN;
SELECT balance FROM accounts WHERE id = 1;   -- 100

-- T2
BEGIN;
UPDATE accounts SET balance = 300 WHERE id = 1;
COMMIT;

-- T1 (same transaction, reads again)
SELECT balance FROM accounts WHERE id = 1;   -- 300  ❌ changed under us!
COMMIT;
```

Under `READ COMMITTED`, each statement sees a fresh snapshot of committed data, so the second `SELECT` picks up T2's commit. That's fine for many apps but wrong when you need a stable view (e.g., a report that reads the same rows twice and must agree).

**The fix — `REPEATABLE READ`:** the transaction takes one snapshot at its first statement and reads from it for its entire life.

```sql
-- T1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;   -- 100

-- T2 commits balance = 300 (as above)

-- T1
SELECT balance FROM accounts WHERE id = 1;   -- 100  ✅ stable within the txn
COMMIT;
```

T1 keeps seeing 100 no matter what T2 does, because it reads from its frozen snapshot.

---

## Anomaly 3 — Phantom Read

A phantom is when a query's **result set** changes because another transaction inserted or deleted rows matching your predicate. It's non-repeatable reads, but for *sets* of rows rather than a single row.

```sql
-- T1 (READ COMMITTED)
BEGIN;
SELECT count(*) FROM accounts WHERE owner = 'alice';   -- 2

-- T2
BEGIN;
INSERT INTO accounts VALUES (3, 'alice', 50);
COMMIT;

-- T1
SELECT count(*) FROM accounts WHERE owner = 'alice';   -- 3  ❌ a row appeared
COMMIT;
```

A new "phantom" row matched the predicate on the second query. The standard says `REPEATABLE READ` may still allow phantoms — but **PostgreSQL's snapshot-based `REPEATABLE READ` prevents them too**, because the whole transaction reads from one snapshot. Under PG `REPEATABLE READ`, T1 sees 2 both times.

```sql
-- PostgreSQL: REPEATABLE READ already blocks phantoms via snapshot isolation.
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM accounts WHERE owner = 'alice';   -- 2
-- (T2 inserts + commits)
SELECT count(*) FROM accounts WHERE owner = 'alice';   -- 2  ✅
COMMIT;
```

This is a genuine difference from the ANSI standard and from MySQL's semantics — worth knowing when you port assumptions between engines.

---

## The Anomaly Snapshot Isolation Does NOT Fix — Write Skew

Here's the subtle one that catches senior engineers. PostgreSQL's `REPEATABLE READ` is **Snapshot Isolation (SI)**, and SI permits an anomaly the four-level table doesn't even list: **write skew**.

The classic example: a hospital on-call rule requiring **at least one** doctor on shift. Two doctors, both currently on call, each decide to go off shift at the same time.

```sql
CREATE TABLE doctors (id INT PRIMARY KEY, name TEXT, on_call BOOLEAN);
INSERT INTO doctors VALUES (1, 'Alice', true), (2, 'Bob', true);
```

```sql
-- T1 (REPEATABLE READ / Snapshot Isolation)
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM doctors WHERE on_call;   -- 2, so it's "safe" to leave
UPDATE doctors SET on_call = false WHERE id = 1;

-- T2 (REPEATABLE READ), concurrently
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM doctors WHERE on_call;   -- 2 (own snapshot), "safe" to leave
UPDATE doctors SET on_call = false WHERE id = 2;

-- both COMMIT
COMMIT;   -- T1
COMMIT;   -- T2
```

Each transaction checked the invariant against its own snapshot (saw 2 on call), updated a *different* row, and committed. Neither overwrote the other, so there's no lost update and no conflict SI can detect. Result: **zero doctors on call** — the invariant is violated even though each transaction was individually correct.

Write skew happens whenever two transactions read an overlapping set, then write to disjoint rows, and their combined effect breaks a rule neither one broke alone. SI cannot catch it because there's no direct write-write conflict.

---

## Serializable — The Only Level That Fixes Write Skew

`SERIALIZABLE` guarantees the outcome is equivalent to running the transactions **one at a time in some order**. PostgreSQL implements this as **Serializable Snapshot Isolation (SSI)**: it runs at snapshot-isolation speed but *monitors* for dangerous read-write dependency patterns and aborts a transaction that would break serializability.

```sql
-- Both T1 and T2 use SERIALIZABLE this time.
-- T1
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM doctors WHERE on_call;   -- 2
UPDATE doctors SET on_call = false WHERE id = 1;

-- T2
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM doctors WHERE on_call;   -- 2
UPDATE doctors SET on_call = false WHERE id = 2;

COMMIT;   -- T1 succeeds
COMMIT;   -- T2:
-- ERROR: could not serialize access due to read/write dependencies
--        among transactions
-- HINT: The transaction might succeed if retried.
```

T2 is aborted. On retry, its snapshot now shows only one doctor on call, so it correctly refuses to leave. The invariant holds.

The price: `SERIALIZABLE` **can abort any transaction at commit** with SQLSTATE `40001`, so every serializable transaction needs a retry loop.

```java
// Serializable transactions MUST be retry-safe.
void runSerializable(Runnable txn) {
    for (int attempt = 1; ; attempt++) {
        try { txn.run(); return; }
        catch (SQLException e) {
            if (!"40001".equals(e.getSQLState()) || attempt >= 5)
                throw new RuntimeException(e);
            sleep(20L * attempt);   // backoff before retry
        }
    }
}
```

---

## Choosing a Level (and the Cheaper Alternatives)

You don't always need `SERIALIZABLE`. Often a targeted lock fixes a specific anomaly at lower cost.

```sql
-- Option A: escalate isolation for the whole transaction.
BEGIN ISOLATION LEVEL SERIALIZABLE;  -- safest, needs retry handling

-- Option B: stay at READ COMMITTED but lock the rows you'll depend on,
--           turning a read into a guarded read (prevents write skew on those rows).
BEGIN;
SELECT count(*) FROM doctors WHERE on_call FOR UPDATE;  -- materialize + lock the set
UPDATE doctors SET on_call = false WHERE id = 1;
COMMIT;

-- Option C: express the invariant as a constraint so the DB enforces it directly.
ALTER TABLE accounts ADD CONSTRAINT non_negative CHECK (balance >= 0);
```

Practical guidance:

- **Read Committed (default)** — fine for most OLTP. Combine with `FOR UPDATE` on the specific rows a decision depends on.
- **Repeatable Read** — when a transaction reads the same data multiple times and must see a stable snapshot (reports, multi-step reads). Free phantom protection in Postgres.
- **Serializable** — when correctness depends on invariants across rows you read but don't all write (write skew), and you can't easily express it as a constraint or a lock. Always pair with retries.

---

## MySQL Notes

- InnoDB's **default is `REPEATABLE READ`** (not `READ COMMITTED` like Postgres) — a frequent cross-engine gotcha.
- InnoDB's `REPEATABLE READ` blocks phantoms via **next-key (gap) locks** rather than pure snapshot isolation, so it prevents phantoms by locking ranges, with different locking side effects than Postgres.
- MySQL `SERIALIZABLE` is implemented largely by turning plain `SELECT`s into locking reads (`LOCK IN SHARE MODE`), so it leans on blocking rather than Postgres's abort-and-retry SSI. Expect lock waits instead of `40001` serialization errors.
- Set the level with `SET TRANSACTION ISOLATION LEVEL ...;` before `BEGIN`, or `SET SESSION` for the connection.

---

## The Takeaways

- Isolation levels are defined by the anomalies they forbid: **dirty read, non-repeatable read, phantom** — plus the unlisted **write skew**.
- **Read Committed is the default**, and it permits non-repeatable reads and phantoms. Know what your default lets through.
- PostgreSQL's `REPEATABLE READ` is Snapshot Isolation: it blocks phantoms (beyond the standard) but **not write skew**.
- Only `SERIALIZABLE` prevents write skew; in Postgres it does so via SSI and can abort transactions with `40001`, so **serializable code must retry**.
- Reach for the lowest level that's correct for the operation, and prefer targeted `FOR UPDATE` locks or `CHECK` constraints when they solve the specific anomaly more cheaply.
