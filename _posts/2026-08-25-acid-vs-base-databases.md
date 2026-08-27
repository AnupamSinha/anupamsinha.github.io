---
title: "ACID vs BASE — What Every Backend Developer Gets Wrong"
date: 2026-08-25
categories: [SQL, System Design]
tags: [sql, postgresql, databases, acid, consistency, system-design]
description: "ACID and BASE aren't opposites — they're per-operation guarantees. A deep dive into consistency, CAP, PACELC, and tunable durability."
---

## The Misconception That Starts Every Argument

"SQL is ACID, NoSQL is BASE." You've heard it. It's wrong often enough to be dangerous.

Modern document and wide-column stores offer configurable ACID transactions. Distributed SQL databases run globally while keeping strong consistency. And a plain PostgreSQL primary with async replicas gives you ACID *on the primary* and BASE-flavored eventual consistency *on the replicas* — in the same system, at the same time.

ACID and BASE are not a binary switch on your database engine. They're **properties of specific operations** under specific configurations. The useful question is never "is my database ACID or BASE?" It's "for *this* operation, what guarantees do I actually get?"

Let's define both precisely, then show where each genuinely applies.

---

## ACID, Precisely

ACID describes guarantees for a **transaction** — a group of operations treated as one unit.

**Atomicity** — all or nothing. If any statement fails, the whole transaction rolls back.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- debit
UPDATE accounts SET balance = balance + 100 WHERE id = 2;   -- credit
-- If the second UPDATE fails, the first is undone. No money vanishes.
COMMIT;
```

**Consistency** — the transaction moves the database from one valid state to another, honoring all constraints. This is the most abused word in our field (more below).

```sql
-- A CHECK constraint enforces an invariant. A transaction that would violate
-- it is rejected, keeping the database consistent by definition.
ALTER TABLE accounts ADD CONSTRAINT non_negative CHECK (balance >= 0);

BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- balance would go -20
COMMIT;   -- ERROR: new row violates check constraint "non_negative"  -> ROLLBACK
```

**Isolation** — concurrent transactions don't step on each other; each behaves as if it ran alone (subject to the isolation level).

```sql
-- Two withdrawals racing. Without isolation, both could read balance=100
-- and each subtract 100, overdrawing. Isolation prevents that anomaly.
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT balance FROM accounts WHERE id = 1;   -- 100
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

**Durability** — once committed, it survives crashes. Postgres guarantees this by flushing the WAL to stable storage before acknowledging `COMMIT`.

```sql
-- synchronous_commit = on (default) => WAL fsynced before COMMIT returns.
-- Pull the power cord after COMMIT, and the change is still there on restart.
SHOW synchronous_commit;   -- on
```

The critical insight: **ACID is a single-node property by default.** The moment your transaction spans multiple machines, plain ACID no longer applies for free — you need distributed protocols (2PC, consensus) that cost latency.

---

## BASE, Precisely

BASE came from the observation (by Eric Brewer, of CAP fame) that large distributed systems often *can't* have strong consistency without sacrificing availability, so they choose differently.

- **Basically Available** — the system responds to requests, even if some nodes are down or partitioned. It favors returning *something* over erroring.
- **Soft state** — the system's state may change over time even without new input, because replicas are converging in the background.
- **Eventually consistent** — if writes stop, all replicas eventually agree. Until then, different replicas may return different values.

The everyday BASE system most backend developers already run isn't a NoSQL store — it's their **PostgreSQL read replica**:

```
t0  write "balance=200" commits on PRIMARY
t1  read from REPLICA_A  -> 200   (caught up)
t1  read from REPLICA_B  -> 100   (still lagging)  <- soft state / eventual
t2  read from REPLICA_B  -> 200   (converged)
```

That is textbook eventual consistency, delivered by a database everyone calls "ACID." Which is exactly why the SQL/ACID vs NoSQL/BASE framing collapses under inspection.

---

## The Word "Consistency" Means Three Different Things

Most ACID-vs-BASE confusion is really a vocabulary collision. "Consistency" refers to three unrelated ideas:

1. **ACID Consistency** — the database enforces declared constraints and invariants within a transaction. This is about *validity*, not about copies of data.
2. **CAP Consistency** (linearizability) — every read sees the most recent committed write, system-wide, as if there were a single copy. This is about *replica agreement*.
3. **Read consistency / isolation** — what concurrent transactions can observe of each other (dirty reads, phantoms, etc.).

```
ACID "C"   -> "the balance is never negative"          (constraint validity)
CAP  "C"   -> "every replica returns the latest write" (linearizability)
Isolation  -> "you won't see another txn's uncommitted change" (visibility)
```

When someone says "we need consistency," always ask *which one*. A payments team means ACID validity plus isolation. A globally distributed team debating CAP means linearizability. These have completely different costs and solutions.

---

## CAP and PACELC — Why You Can't Have Everything

**CAP:** during a network **P**artition, a distributed system must choose between **C**onsistency (linearizability) and **A**vailability. You cannot have both while partitioned.

```
Partition happens. A write lands on one side.
  CP choice: refuse reads/writes that can't guarantee latest data (stay correct, lose availability)
  AP choice: keep serving, accept some nodes return stale data (stay up, lose linearizability)
```

CAP only speaks to the partition case, which is misleadingly narrow. **PACELC** completes it: *if* **P**artition then **A** vs **C**; **E**lse (normal operation) then **L**atency vs **C**onsistency.

```
PACELC:
  during Partition:  Availability  <-> Consistency
  Else (no fault):   Latency       <-> Consistency
```

This is the honest framing. Even with a healthy network, stronger consistency costs latency (coordination round trips). Async Postgres replicas are effectively **PA/EL** — favor availability and low latency, accept staleness. A synchronous, consensus-backed setup leans **PC/EC** — pay latency for strong consistency.

---

## Where Each Genuinely Fits

Choose by the **operation**, not by dogma.

**Use ACID / strong consistency when incorrectness is unacceptable:**

```sql
-- Money movement, inventory decrement, unique-username claim, seat booking.
-- These need atomicity + isolation or they produce real-world bugs.
BEGIN;
UPDATE inventory SET qty = qty - 1 WHERE sku = 'ABC' AND qty > 0;
-- rowcount 0 => sold out; abort and tell the user
INSERT INTO orders(sku, customer_id) VALUES ('ABC', 42);
COMMIT;
```

**Use BASE / eventual consistency when availability and scale beat freshness:**

```
-- Like counts, view counters, activity feeds, product recommendations,
-- "who's online" — a few seconds of staleness is invisible and harmless.
-- Serving these from lagging replicas or a cache is the right call.
```

The strongest systems mix both in one product: the checkout transaction is strictly ACID on the primary; the "recommended for you" strip is happily eventual off a replica or cache. Same request, two consistency models, chosen per operation.

---

## Tunable Consistency Is the Real World

The binary is a myth because most serious systems expose **knobs**. Postgres lets you dial durability and replica consistency per transaction:

```sql
-- Relax durability for a high-volume, loss-tolerant write (e.g. analytics events).
-- Commit returns without waiting for fsync — faster, small crash-loss window.
SET synchronous_commit = off;
INSERT INTO click_events(user_id, url) VALUES (42, '/home');

-- Tighten it for money: wait until a standby has the data before acknowledging.
SET synchronous_commit = remote_apply;
BEGIN;
UPDATE ledger SET balance = balance - 100 WHERE account_id = 1;
COMMIT;   -- returns only after a synchronous standby applied it
```

Distributed databases go further, letting you pick consistency per query — strong reads from a leader vs bounded-staleness reads from a follower. The point stands: **consistency is a spectrum you tune, not a label you inherit.**

---

## MySQL Notes

- InnoDB is fully ACID with WAL-equivalent redo logs; `innodb_flush_log_at_trx_commit = 1` is the durable default, and lowering it (0 or 2) trades durability for throughput — the same knob as Postgres `synchronous_commit`.
- MySQL async replicas exhibit the same eventual consistency as Postgres replicas. Semi-synchronous replication (`rpl_semi_sync`) is the analog of waiting for a standby before acknowledging.
- The MyISAM engine (legacy) is **not** transactional or crash-safe — a reminder that "it's MySQL" tells you nothing about ACID until you know the storage engine.

---

## The Takeaways

- ACID and BASE describe **operation guarantees**, not database brands. One system routinely provides both.
- "NoSQL = BASE" is outdated; many document/wide-column stores now offer ACID transactions, and distributed SQL keeps strong consistency across nodes.
- The word "consistency" is overloaded — ACID validity, CAP linearizability, and isolation visibility are three different things. Always ask which one.
- CAP is only about partitions; **PACELC** captures the everyday latency-vs-consistency trade you pay even on a healthy network.
- Design per operation: strong where wrong answers cost money, eventual where staleness is invisible. Then tune the knobs your database already gives you.
