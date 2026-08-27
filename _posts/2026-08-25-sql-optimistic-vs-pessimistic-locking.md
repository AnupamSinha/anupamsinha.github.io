---
title: "Optimistic vs Pessimistic Locking in SQL (With Real Deadlock Examples)"
date: 2026-08-25
categories: [SQL, System Design]
tags: [sql, postgresql, databases, locking, concurrency, system-design]
description: "How pessimistic and optimistic locking prevent lost updates in SQL — with a reproducible deadlock, lock ordering, and retry loops."
---

## The Problem: The Lost Update

Two users load the same record, both edit it, both save. Whoever saves last silently wipes out the other's change. That's a **lost update**, and it's the canonical concurrency bug locking exists to prevent.

```sql
CREATE TABLE products (
    id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name     TEXT   NOT NULL,
    stock    INT    NOT NULL,
    version  INT    NOT NULL DEFAULT 0   -- used later for optimistic locking
);

INSERT INTO products (name, stock) VALUES ('Widget', 100);
```

The naive read-modify-write:

```
Session A: SELECT stock FROM products WHERE id=1;   -- reads 100
Session B: SELECT stock FROM products WHERE id=1;   -- reads 100
Session A: UPDATE products SET stock = 100 - 30 WHERE id=1;  -- writes 70
Session B: UPDATE products SET stock = 100 - 50 WHERE id=1;  -- writes 50  ❌
-- A's decrement is gone. Final stock is 50, should be 20.
```

There are two schools of thought for fixing this: **pessimistic** (assume conflict, lock first) and **optimistic** (assume no conflict, verify at write). Both are correct; they suit different workloads.

---

## Pessimistic Locking — Lock First, Ask Questions Never

Pessimistic locking takes a lock when you read, so nobody else can modify the row until you're done. In SQL this is `SELECT ... FOR UPDATE`.

```sql
BEGIN;
-- Acquires a row-level exclusive lock. Any other txn doing FOR UPDATE
-- on this row must WAIT until we COMMIT or ROLLBACK.
SELECT stock FROM products WHERE id = 1 FOR UPDATE;   -- reads 100, row locked

UPDATE products SET stock = stock - 30 WHERE id = 1;
COMMIT;   -- lock released; other waiters proceed with fresh data
```

Now the race is safe:

```
Session A: BEGIN; SELECT ... FOR UPDATE;   -- locks row, reads 100
Session B: BEGIN; SELECT ... FOR UPDATE;   -- BLOCKS, waiting for A
Session A: UPDATE stock = 70; COMMIT;      -- releases lock
Session B: (unblocks) reads 70; UPDATE stock = 20; COMMIT;   ✅ correct
```

### FOR UPDATE Variants You Should Know

```sql
-- Block until the lock is free (default).
SELECT * FROM products WHERE id = 1 FOR UPDATE;

-- Don't wait — fail immediately if the row is locked. Good for user-facing
-- flows where you'd rather tell the user "try again" than hang.
SELECT * FROM products WHERE id = 1 FOR UPDATE NOWAIT;

-- Skip locked rows entirely. THE pattern for queue/job tables: each worker
-- grabs the next available row without contending on ones others hold.
SELECT * FROM jobs
WHERE status = 'PENDING'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

`FOR UPDATE SKIP LOCKED` is the single best reason to know this feature — it turns a plain table into a concurrent work queue with no extra infrastructure.

**Pessimistic locking is right when:** conflicts are frequent, the operation is short, and retrying is expensive or the work is non-idempotent (you can't just replay it). Downsides: it holds locks (reduces concurrency), and it's how you create **deadlocks**.

---

## A Real Deadlock — Reproduced Step by Step

A deadlock is a cycle: A holds a lock B wants, and B holds a lock A wants. Neither can proceed. Here's how to reproduce one deliberately with two `psql` sessions.

```sql
-- Setup
INSERT INTO products (name, stock) VALUES ('A', 10), ('B', 10);  -- ids 2, 3
```

**Session 1:**
```sql
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 2;   -- locks row 2
-- ... pause here, do NOT commit yet ...
```

**Session 2:**
```sql
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 3;   -- locks row 3
UPDATE products SET stock = stock - 1 WHERE id = 2;   -- WAITS for Session 1's lock on row 2
```

**Back in Session 1:**
```sql
UPDATE products SET stock = stock - 1 WHERE id = 3;   -- WAITS for Session 2's lock on row 3
-- CYCLE! S1 waits on S2, S2 waits on S1.
```

PostgreSQL's deadlock detector notices the cycle (after `deadlock_timeout`, default 1s) and kills one transaction:

```
ERROR:  deadlock detected
DETAIL:  Process 12345 waits for ShareLock on transaction 890; blocked by process 12346.
         Process 12346 waits for ShareLock on transaction 889; blocked by process 12345.
HINT:  See server log for query details.
```

The victim's transaction is rolled back; the survivor completes. Your application **must** be ready to catch this and retry.

### The Fix: Consistent Lock Ordering

The deadlock happened only because the two sessions locked rows in *opposite* orders (S1: 2 then 3; S2: 3 then 2). If every transaction locks rows in the **same order**, a cycle is impossible.

```sql
-- Always lock the lower id first, everywhere in the codebase.
BEGIN;
SELECT * FROM products WHERE id IN (2, 3) ORDER BY id FOR UPDATE;  -- deterministic order
UPDATE products SET stock = stock - 1 WHERE id = 2;
UPDATE products SET stock = stock - 1 WHERE id = 3;
COMMIT;
```

Consistent lock ordering is the number-one structural defense against deadlocks. Locking a whole set in one `ORDER BY ... FOR UPDATE` statement enforces it for you.

---

## Optimistic Locking — Assume the Best, Verify at Write

Optimistic locking takes **no locks** while you think. Instead it adds a `version` column, and the `UPDATE` only succeeds if the version hasn't changed since you read it.

```sql
-- 1) Read the row and remember its version.
SELECT id, stock, version FROM products WHERE id = 1;   -- version = 0

-- 2) Later, write conditionally: bump version, but only if it's still 0.
UPDATE products
SET stock   = stock - 30,
    version = version + 1
WHERE id = 1
  AND version = 0;        -- the guard

-- 3) Check affected rows. 1 => success. 0 => someone else changed it first.
```

The magic is in the row count. If another transaction already incremented `version`, the `WHERE version = 0` matches nothing, zero rows update, and you know you lost the race — so you re-read and retry.

```
Session A: reads version=0
Session B: reads version=0
Session A: UPDATE ... WHERE version=0  -> 1 row, version now 1  ✅
Session B: UPDATE ... WHERE version=0  -> 0 rows  ❌ (conflict detected)
Session B: re-read (version=1), recompute, retry
```

### In Application Code (JPA does this for you)

```java
@Entity
class Product {
    @Id Long id;
    int stock;

    @Version                 // JPA manages the version column automatically
    int version;
}

// On flush, JPA emits: UPDATE ... WHERE id=? AND version=?
// If 0 rows update, it throws OptimisticLockException.
@Transactional
void decrementStock(Long id, int qty) {
    Product p = repo.findById(id).orElseThrow();
    p.setStock(p.getStock() - qty);
    // commit -> version check -> OptimisticLockException on conflict
}
```

Wrap it with retry:

```java
@Retryable(retryFor = OptimisticLockException.class,
           maxAttempts = 3, backoff = @Backoff(delay = 50, multiplier = 2))
@Transactional
void decrementStockWithRetry(Long id, int qty) {
    decrementStock(id, qty);
}
```

**Optimistic locking is right when:** conflicts are rare, transactions are long or interactive (a user editing a form for minutes — you'd never hold a DB lock that long), and the operation is safe to retry. Downside: under high contention, retries pile up and waste work.

---

## Choosing Between Them

| Situation | Pick |
|---|---|
| High contention on the same rows | Pessimistic (`FOR UPDATE`) |
| Low contention, mostly reads | Optimistic (`version`) |
| Long-lived / user-interactive edits | Optimistic — never hold a lock across a think-time |
| Short server-side critical section | Pessimistic |
| Work queue / job dispatch | Pessimistic `FOR UPDATE SKIP LOCKED` |
| Retry is cheap and idempotent | Optimistic |
| Retry is expensive / non-idempotent | Pessimistic |

A useful heuristic: **optimistic for user-facing edits over HTTP, pessimistic for tight server-side money/stock critical sections.** And you can combine them — optimistic for the common path, with a pessimistic fallback for hot rows.

---

## The Retry Loop Is Not Optional

Both approaches can fail under concurrency — optimistic returns 0 rows, pessimistic can deadlock. Production code needs a bounded retry with backoff for the serialization/deadlock error codes.

```java
// Retry on Postgres serialization (40001) and deadlock (40P01) failures.
void withRetry(Runnable txn) {
    int attempts = 0;
    while (true) {
        try { txn.run(); return; }
        catch (SQLException e) {
            String state = e.getSQLState();
            boolean retryable = "40001".equals(state) || "40P01".equals(state);
            if (!retryable || ++attempts >= 3) throw new RuntimeException(e);
            sleep(50L * attempts);   // simple linear backoff + jitter in practice
        }
    }
}
```

If you take one thing away: **any code that uses locking or high isolation must be retry-safe.** The database is allowed to abort your transaction to break a deadlock or resolve a serialization conflict, and it expects you to try again.

---

## MySQL Notes

- InnoDB supports `SELECT ... FOR UPDATE`, `FOR SHARE`, `NOWAIT`, and `SKIP LOCKED` — the same toolkit as Postgres.
- InnoDB also detects deadlocks automatically and rolls back the transaction that did the least work; inspect `SHOW ENGINE INNODB STATUS` for the latest deadlock.
- Watch out: InnoDB uses **next-key locks** (gap locks) under `REPEATABLE READ` (its default), so `FOR UPDATE` can lock ranges/gaps, not just matched rows — a common source of surprising lock waits absent in Postgres.
- Optimistic locking via a `version` column works identically; check `ROW_COUNT()` (or the driver's affected-rows) after the guarded `UPDATE`.

---

## The Takeaways

- The enemy is the **lost update**. Pessimistic and optimistic locking are two correct cures.
- **Pessimistic** (`FOR UPDATE`) prevents conflict by blocking — great for high contention and short critical sections, but it creates deadlocks.
- **Deadlocks are cycles**; kill them structurally with **consistent lock ordering**, and handle survivors with retries.
- **Optimistic** (`version` guard) detects conflict at write time with zero locks — great for low contention and long, interactive edits.
- Whichever you choose, wrap it in a bounded, backing-off **retry loop** keyed on the serialization/deadlock SQL states. Non-retry-safe concurrency code is a bug waiting for load.
