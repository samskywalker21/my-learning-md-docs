# MySQL Transactions & Concurrency

Part of the [MySQL Mastery Guide](./mysql-mastery-guide.md). Full Beginner → Mastery depth — the area where MySQL's behavior diverges from Postgres the most sharply, since the two default to different isolation levels and InnoDB's locking model has no direct Postgres analogue.

## Table of Contents

- [Transaction Basics](#transaction-basics)
- [MVCC Internals: Undo Logs, Not Tuple Versioning](#mvcc-internals-undo-logs-not-tuple-versioning)
- [Isolation Levels](#isolation-levels)
- [Gap Locks and Next-Key Locks](#gap-locks-and-next-key-locks)
- [Explicit Locking](#explicit-locking)
- [Deadlocks](#deadlocks)
- [Cheat Sheets](#cheat-sheets)

---

## Transaction Basics

**Beginner.**

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;   -- or ROLLBACK;
```

**Working Knowledge.** The trap that catches people coming from a client library with connection pooling or ORMs: `autocommit` is **on by default** in MySQL. Every standalone statement outside an explicit `START TRANSACTION` (or `BEGIN`) commits immediately on its own.

```sql
-- Wrong assumption: "I'll roll this back if something looks off"
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Already committed. There is nothing to roll back.

-- Right: wrap it first
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- inspect, then COMMIT or ROLLBACK
```

Savepoints work as in Postgres:

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SAVEPOINT before_bonus;
UPDATE accounts SET balance = balance + 10 WHERE id = 1;  -- a bonus that turns out to be wrong
ROLLBACK TO before_bonus;
COMMIT;
```

[Back to top](#mysql-transactions--concurrency)

---

## MVCC Internals: Undo Logs, Not Tuple Versioning

**Mastery.** Postgres MVCC keeps old row versions directly in the heap (dead tuples cleaned up by `VACUUM`). InnoDB takes a different physical approach: **the table always holds the current row version**, and old versions live separately in **undo logs**, reconstructed on demand for a transaction that needs an older snapshot.

Every InnoDB row carries two hidden fields:

- `DB_TRX_ID` — the transaction ID that last modified the row
- `DB_ROLL_PTR` — a pointer into the undo log, letting InnoDB walk backward to reconstruct the row as it looked before that transaction

```
Current row (in the table):          Undo log (rollback segment):
┌─────────────────────────┐          ┌──────────────────────────┐
│ id=1 balance=400          │          │ trx=88: balance was 500   │
│ DB_TRX_ID=91               │─roll_ptr─►│ trx=91: balance was 400  │  (older versions)
│ DB_ROLL_PTR ───────────────┘          └──────────────────────────┘
└─────────────────────────┘
A transaction whose snapshot predates trx 91 walks the undo
chain to reconstruct the value it's allowed to see (500, or 400,
depending on when its read view was taken).
```

A **read view**, taken at a point determined by the isolation level (see below), records which transaction IDs were already committed at that moment — any row version from an uncommitted or later transaction is invisible to that read view, and InnoDB walks the undo chain until it finds a version that qualifies.

The operational consequence: a long-running transaction holding an old read view forces InnoDB to retain undo log entries that would otherwise be purged, which can bloat undo tablespaces — the InnoDB analogue of Postgres's "long transaction blocking `VACUUM`" problem, just with a different physical mechanism underneath. See the official [InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html) reference.

[Back to top](#mysql-transactions--concurrency)

---

## Isolation Levels

**Advanced.** All four SQL-standard levels are supported, but **the default is `REPEATABLE READ`**, not `READ COMMITTED` — a genuine, easy-to-miss divergence from Postgres (whose default is `READ COMMITTED`) and from the SQL-standard-typical default most people assume.

```sql
SELECT @@transaction_isolation;
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**Real Scenario (try it yourself, two `mysql` sessions):**

```sql
-- Session A                              -- Session B
START TRANSACTION;                        
SELECT balance FROM accounts WHERE id=1;  -- reads 500
                                           START TRANSACTION;
                                           UPDATE accounts SET balance=600 WHERE id=1;
                                           COMMIT;
SELECT balance FROM accounts WHERE id=1;  -- still reads 500 under REPEATABLE READ
COMMIT;
SELECT balance FROM accounts WHERE id=1;  -- now reads 600, new transaction/read view
```

Under `REPEATABLE READ` (the default), session A's *first* read establishes a read view held for the whole transaction — session B's committed change is invisible until session A commits and starts fresh. Switch session A to `READ COMMITTED` and repeat: the second `SELECT` inside the same transaction *would* see 600, because `READ COMMITTED` takes a fresh read view on every statement, not once per transaction.

`REPEATABLE READ` in InnoDB also genuinely prevents phantom reads for locking reads (unlike the SQL-standard minimum, which only requires this at `SERIALIZABLE`) — because of the [gap locking](#gap-locks-and-next-key-locks) below.

[Back to top](#mysql-transactions--concurrency)

---

## Gap Locks and Next-Key Locks

**Mastery.** This is InnoDB-specific machinery with no Postgres equivalent, and it's the most common cause of "why did this INSERT just deadlock/block, I didn't even touch that row" confusion.

A **next-key lock** = a record lock on an index entry + a **gap lock** on the space immediately before it. Under `REPEATABLE READ`, InnoDB uses next-key locks for locking reads and index scans specifically to prevent phantom rows — another transaction can't insert a new row into a gap another transaction has scanned and locked.

```
Index on id, existing values: 5, 10, 15

Gaps:      (−∞,5)   (5,10)   (10,15)   (15,+∞)
                       ▲
A `SELECT ... FOR UPDATE WHERE id BETWEEN 6 AND 12`
locks the record at id=10 AND the gap (5,10) AND the gap (10,15) —
an INSERT of id=8 or id=12 from another session now blocks,
even though neither value exists yet.
```

```sql
-- Session A
START TRANSACTION;
SELECT * FROM accounts WHERE id BETWEEN 6 AND 12 FOR UPDATE;
-- Session B, concurrently:
INSERT INTO accounts (id, balance) VALUES (8, 0);
-- Blocks — id=8 falls in a gap session A locked, even though no row
-- with id=8 existed for session A to "touch".
```

This is why unique-constraint-violation races and unexpected `INSERT` blocking are common under `REPEATABLE READ` and largely disappear under `READ COMMITTED`, which disables gap locking for plain searches/scans (gap locks are still used for foreign-key checks and duplicate-key checking even there). See the official [InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html) reference — this is worth reading in full once, since the interaction between lock types is genuinely subtle.

[Back to top](#mysql-transactions--concurrency)

---

## Explicit Locking

**Working Knowledge → Advanced.**

```sql
-- Row-level exclusive lock, blocks other writers/lockers on matched rows
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- Row-level shared lock: others can read, but not FOR UPDATE/modify, until you commit
SELECT * FROM accounts WHERE id = 1 FOR SHARE;   -- (LOCK IN SHARE MODE is the older, pre-8.0 syntax)
```

`SKIP LOCKED` and `NOWAIT` (8.0+) — genuinely useful for job-queue patterns, avoiding the classic "everyone blocks on the same locked row" contention:

```sql
-- Wrong: every worker blocks waiting for whichever row a competitor already locked
SELECT * FROM job_queue WHERE status = 'pending' ORDER BY id LIMIT 1 FOR UPDATE;

-- Right: skip rows another transaction already has locked, grab the next available one
SELECT * FROM job_queue WHERE status = 'pending' ORDER BY id LIMIT 1 FOR UPDATE SKIP LOCKED;
```

`LOCK TABLES` is a coarser, mostly-legacy mechanism (table-level, needed for MyISAM which has no row-level locking) — avoid it on InnoDB tables in favor of transactions + row locks unless you have a specific reason (e.g. coordinating with a non-transactional engine).

[Back to top](#mysql-transactions--concurrency)

---

## Deadlocks

**Advanced.** Deadlock detection is on by default (`innodb_deadlock_detect`); InnoDB picks the transaction that's done less work (by rows modified) as the victim and rolls it back automatically with error 1213.

```sql
SHOW ENGINE INNODB STATUS\G
-- Look at the "LATEST DETECTED DEADLOCK" section: it shows both
-- competing transactions, the exact locks each held and each waited
-- for, and which one InnoDB chose to roll back.
```

**Real Scenario (try it yourself, two sessions):**

```sql
-- Session A                          -- Session B
START TRANSACTION;                    START TRANSACTION;
UPDATE accounts SET balance=balance-1 
  WHERE id=1;
                                       UPDATE accounts SET balance=balance-1
                                         WHERE id=2;
UPDATE accounts SET balance=balance+1 
  WHERE id=2;   -- blocks, waiting on B
                                       UPDATE accounts SET balance=balance+1
                                         WHERE id=1;   -- deadlock!
```

One session gets `ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction` — the fix at the application layer is always to catch that specific error and retry the whole transaction, never to treat it as a fatal error. The **general prevention pattern**: always acquire locks/update rows in the same order across every code path (e.g. always by ascending `id`) so two transactions can't form a cycle.

If deadlock detection is ever disabled (rare, sometimes done under extreme lock contention where detection overhead itself becomes the bottleneck), InnoDB instead relies on `innodb_lock_wait_timeout` (default 50s) to eventually time out one side — much slower and worse for user-facing latency, so leave detection on unless you have a specific, measured reason not to. See the official [Deadlock Detection](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-detection.html) reference, and [Stack Overflow: how to fix "Deadlock found when trying to get lock"](https://stackoverflow.com/questions/2688399/deadlock-found-when-trying-to-get-lock-try-restarting-transaction) for how practitioners actually structure the retry logic.

[Back to top](#mysql-transactions--concurrency)

---

## Cheat Sheets

**Isolation levels:**

| Level | Dirty reads | Non-repeatable reads | Phantom reads (locking) | MySQL default? |
|---|---|---|---|---|
| `READ UNCOMMITTED` | Possible | Possible | Possible | No |
| `READ COMMITTED` | Prevented | Possible | Possible | No (Postgres's default) |
| `REPEATABLE READ` | Prevented | Prevented | **Prevented** (via next-key locks) | **Yes** |
| `SERIALIZABLE` | Prevented | Prevented | Prevented | No |

**Locking quick reference:**

| Need | Statement |
|---|---|
| Exclusive row lock | `SELECT ... FOR UPDATE` |
| Shared row lock | `SELECT ... FOR SHARE` |
| Job-queue-safe row grab | `SELECT ... FOR UPDATE SKIP LOCKED` |
| Fail fast instead of waiting | `SELECT ... FOR UPDATE NOWAIT` |
| Inspect last deadlock | `SHOW ENGINE INNODB STATUS\G` |
| Change isolation for one session | `SET SESSION TRANSACTION ISOLATION LEVEL ...` |

[Back to top](#mysql-transactions--concurrency)
