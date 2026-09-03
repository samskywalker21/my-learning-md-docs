# Postgres Transactions & Concurrency

Part of the [Postgres SQL Mastery Guide](./postgres-mastery-guide.md).

This is where Postgres mastery really lives: how it lets dozens of clients read and write the same rows at once without them stepping on each other, and without readers blocking writers. Everything here is meant to be run — ideally with **two psql sessions open side by side** (call them Session A and Session B) so you can watch the behavior happen instead of taking it on faith.

```bash
# terminal 1
psql -d mydb   # Session A
# terminal 2
psql -d mydb   # Session B
```

All version-specific details below were checked against the [PostgreSQL 18 documentation](https://www.postgresql.org/docs/current/) (current stable is 18.6, released August 2026). Anywhere older blog posts say something different, it's flagged.

## Table of Contents

- [Transaction Basics](#transaction-basics)
- [MVCC Internals](#mvcc-internals)
- [Isolation Levels](#isolation-levels)
- [Explicit Locking](#explicit-locking)
- [Deadlocks](#deadlocks)
- [VACUUM & Autovacuum](#vacuum--autovacuum)
- [Cheat Sheets](#cheat-sheets)

---

## Transaction Basics

<a id="transaction-basics-beginner"></a>

### Beginner

**What it is.** A transaction is a group of statements that succeed or fail *together*. Postgres actually wraps every standalone statement in an implicit transaction already — `BEGIN`/`COMMIT` just lets you group several statements into one.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

If anything goes wrong before `COMMIT`, run `ROLLBACK` instead and neither update happened — this is the "atomic" in ACID.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- oops, wrong account
ROLLBACK;
```

**Wrong vs. right — forgetting you're in a transaction:**

```sql
-- WRONG: BEGIN with no matching COMMIT/ROLLBACK leaves the session
-- "idle in transaction" — it keeps holding whatever locks/snapshot it
-- acquired, which can block vacuum and other sessions indefinitely.
BEGIN;
SELECT * FROM accounts WHERE id = 1;
-- ...session sits open for an hour while you go to lunch...
```

```sql
-- RIGHT: always pair BEGIN with an explicit end, and consider
-- idle_in_transaction_session_timeout as a safety net.
BEGIN;
SELECT * FROM accounts WHERE id = 1;
COMMIT;
```

**Try it — see a transaction in isolation:**

```sql
-- Session A
BEGIN;
INSERT INTO accounts (id, balance) VALUES (99, 500);
-- don't commit yet

-- Session B
SELECT * FROM accounts WHERE id = 99;  -- nothing — Session A hasn't committed
```

Go back to Session A and `COMMIT;`, then rerun the `SELECT` in Session B — now it's there.

[↑ back to top](#table-of-contents)

<a id="transaction-basics-working-knowledge"></a>

### Working Knowledge

**Savepoints** let you roll back part of a transaction without abandoning the whole thing — useful when one statement in a batch might fail and you want to recover instead of aborting everything.

```sql
BEGIN;
INSERT INTO orders (id, customer_id) VALUES (1, 42);

SAVEPOINT before_risky_insert;
INSERT INTO orders (id, customer_id) VALUES (1, 43);  -- duplicate key, fails
ROLLBACK TO SAVEPOINT before_risky_insert;

INSERT INTO orders (id, customer_id) VALUES (2, 43);  -- proceed normally
COMMIT;
```

Without the savepoint, that failed `INSERT` would poison the whole transaction — Postgres marks it "aborted" and rejects every subsequent statement until you `ROLLBACK` (see the [transactions docs](https://www.postgresql.org/docs/current/tutorial-transactions.html)).

**Wrong vs. right — recovering from an error:**

```sql
-- WRONG: no savepoint means one failed statement kills the entire transaction
BEGIN;
INSERT INTO orders (id, customer_id) VALUES (1, 42);
INSERT INTO orders (id, customer_id) VALUES (1, 43);  -- ERROR: duplicate key
INSERT INTO orders (id, customer_id) VALUES (2, 43);  -- ERROR: current transaction is aborted
COMMIT;  -- rolls back everything, including the first successful insert
```

```sql
-- RIGHT
BEGIN;
INSERT INTO orders (id, customer_id) VALUES (1, 42);
SAVEPOINT sp1;
INSERT INTO orders (id, customer_id) VALUES (1, 43);
ROLLBACK TO SAVEPOINT sp1;
INSERT INTO orders (id, customer_id) VALUES (2, 43);
COMMIT;  -- both good inserts survive
```

**`RELEASE SAVEPOINT`** discards a savepoint you no longer need (without undoing its work) — useful in long transactions with many nested savepoints, so you don't accumulate cleanup state.

[↑ back to top](#table-of-contents)

---

## MVCC Internals

<a id="mvcc-internals-advanced"></a>

### Advanced — why MVCC exists

**The problem it solves.** Traditional lock-based databases make readers block writers and writers block readers, because a "consistent read" is implemented by holding a shared lock on whatever you're reading. Postgres instead gives every transaction a **snapshot** — a consistent view of the database as of some point in time — so readers never wait on writers and vice versa. This is Multi-Version Concurrency Control: instead of locking rows for reads, Postgres just keeps multiple versions of each row around and lets each transaction see the version appropriate to its snapshot. See the [MVCC introduction](https://www.postgresql.org/docs/current/mvcc-intro.html).

<a id="mvcc-internals-mastery"></a>

### Mastery — tuple versions, xmin/xmax, and visibility

**How it actually works.** Every row version (Postgres calls it a "tuple") is stamped with two hidden system columns:

- `xmin` — the transaction ID (XID) that *inserted* this tuple version
- `xmax` — the transaction ID that *deleted or replaced* this version (`0`/invalid if it's still live)

An `UPDATE` in Postgres never modifies a row in place — it writes a **new tuple** with a fresh `xmin`, and sets `xmax` on the old tuple to the updating transaction's XID. A `DELETE` just sets `xmax` on the existing tuple. The old tuple isn't reclaimed immediately — it's a **dead tuple**, kept around because some other in-flight transaction's snapshot might still need to see it.

```
Row id=1, balance=500  (inserted by txn 100)

   tuple v1: xmin=100  xmax=0        <- current, visible to everyone after txn 100 commits

UPDATE balance = 400 by txn 105:

   tuple v1: xmin=100  xmax=105      <- now dead to future readers, but still
                                          visible to any snapshot taken before txn 105
   tuple v2: xmin=105  xmax=0        <- new current version

UPDATE balance = 300 by txn 110:

   tuple v1: xmin=100  xmax=105      <- dead, eligible for cleanup once no
                                          snapshot needs it
   tuple v2: xmin=105  xmax=110      <- now also dead
   tuple v3: xmin=110  xmax=0        <- current version
```

**Visibility rules.** When a transaction reads a tuple, Postgres checks that tuple's `xmin`/`xmax` against the reader's snapshot (a snapshot is essentially "list of XIDs that were still in-progress when I started, plus the boundary XID above which nothing exists yet"). A tuple is visible if its inserting transaction is known-committed and happened before the snapshot, and its deleting transaction (if any) either doesn't exist or is not yet committed as of the snapshot. This is why **Postgres doesn't have a real Read Uncommitted**: there's no code path that shows you a tuple whose inserting transaction hasn't committed — the visibility check itself prevents dirty reads structurally, it isn't a lock you can choose to skip. Requesting `READ UNCOMMITTED` in Postgres silently gives you `READ COMMITTED` instead (see [transaction-iso](https://www.postgresql.org/docs/current/transaction-iso.html)).

You can watch these hidden columns directly:

```sql
CREATE TABLE demo (id int primary key, val text);
INSERT INTO demo VALUES (1, 'first');
SELECT xmin, xmax, * FROM demo;

UPDATE demo SET val = 'second' WHERE id = 1;
SELECT xmin, xmax, * FROM demo;   -- xmin has changed to the updating XID
```

**Try it — watch dead tuples accumulate:**

```sql
-- Session A
CREATE TABLE ledger (id int primary key, balance int);
INSERT INTO ledger VALUES (1, 100);

-- update the same row 5 times
UPDATE ledger SET balance = balance + 1 WHERE id = 1;
UPDATE ledger SET balance = balance + 1 WHERE id = 1;
UPDATE ledger SET balance = balance + 1 WHERE id = 1;

SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'ledger';
-- n_dead_tup shows the old versions still sitting on disk, waiting for vacuum
```

This is also *why* MVCC needs [VACUUM](#vacuum--autovacuum) at all: readers never block, but the cost is deferred cleanup of old tuple versions.

**Why not just lock?** The alternative (two-phase locking, "read locks") is simpler to reason about but makes long-running reports block all writers on the tables they touch, and makes OLTP writers contend with OLTP readers constantly. MVCC trades that contention for background cleanup work — a trade Postgres's designers considered clearly worth it for mixed read/write workloads. See the [MVCC chapter](https://www.postgresql.org/docs/current/mvcc.html) for the canonical explanation.

[↑ back to top](#table-of-contents)

---

## Isolation Levels

<a id="isolation-levels-working-knowledge"></a>

### Working Knowledge — choosing a level day to day

Postgres supports three distinct isolation levels (a fourth, `READ UNCOMMITTED`, is accepted as syntax but behaves identically to `READ COMMITTED` — see [MVCC internals](#mvcc-internals) above for why):

| Level | Default? | What changes |
|---|---|---|
| `READ COMMITTED` | Yes | Each *statement* gets its own fresh snapshot |
| `REPEATABLE READ` | No | The whole *transaction* uses one snapshot taken at its first statement |
| `SERIALIZABLE` | No | Like Repeatable Read, plus detects and blocks any anomaly that couldn't happen in some serial ordering |

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- ... statements ...
COMMIT;

-- or change the default for the whole session:
SET default_transaction_isolation = 'repeatable read';
```

**Real Scenario — Read Committed's stale count, in two sessions:**

```sql
-- setup, either session
CREATE TABLE inventory (id int primary key, qty int);
INSERT INTO inventory VALUES (1, 10);

-- Session A
BEGIN;
SELECT qty FROM inventory WHERE id = 1;   -- 10
-- pause here, don't commit yet

-- Session B
UPDATE inventory SET qty = 3 WHERE id = 1;
COMMIT;

-- back to Session A, same transaction
SELECT qty FROM inventory WHERE id = 1;   -- 3! Read Committed re-reads fresh each statement
COMMIT;
```

Under `READ COMMITTED` (the default), that second `SELECT` in Session A sees Session B's committed change mid-transaction — each statement gets a new snapshot. Rerun the same exercise with `BEGIN ISOLATION LEVEL REPEATABLE READ;` in Session A and the second `SELECT` still shows `10` — the whole transaction is pinned to the snapshot from its first statement.

[↑ back to top](#table-of-contents)

<a id="isolation-levels-advanced"></a>

### Advanced — serialization failures in practice

Under `REPEATABLE READ` or `SERIALIZABLE`, Postgres doesn't silently let a transaction act on stale data when a conflicting write already happened — it aborts one of the transactions with a `40001` SQLSTATE ("could not serialize access...") that your application is expected to catch and retry.

**Real Scenario — a Repeatable Read write-write conflict:**

```sql
-- Session A
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT qty FROM inventory WHERE id = 1;   -- 10

-- Session B
UPDATE inventory SET qty = qty - 5 WHERE id = 1;
COMMIT;

-- Session A, still in its transaction
UPDATE inventory SET qty = qty - 3 WHERE id = 1;
-- ERROR: could not serialize access due to concurrent update
COMMIT;   -- moot; already aborted, must ROLLBACK
```

Session A's snapshot said `qty = 10`, but by the time it tried to write, the row it was targeting had already been changed by a committed transaction outside its snapshot — Postgres refuses to let Session A blindly overwrite work it never saw. See this exact error explained on [Stack Overflow](https://stackoverflow.com/questions/48000431/postgresql-concurrent-update) — the standard fix is: catch `40001` in application code and retry the transaction from `BEGIN`.

[↑ back to top](#table-of-contents)

<a id="isolation-levels-mastery"></a>

### Mastery — Serializable Snapshot Isolation (SSI) internals

`SERIALIZABLE` in Postgres is implemented as **Serializable Snapshot Isolation**: it starts from the same snapshot mechanism as Repeatable Read, then layers on **predicate locks** that track *which data a transaction's queries depended on*, even data it never wrote. Unlike ordinary locks, predicate locks don't block anything by themselves — they're bookkeeping, visible in `pg_locks` with mode `SIReadLock`. Postgres uses them to detect **rw-antidependencies**: cases where transaction A read data that transaction B later wrote (or vice versa), forming a dependency cycle that couldn't occur in any serial (one-at-a-time) execution order. When such a cycle is detected, one of the transactions is aborted with `could not serialize access due to read/write dependencies among transactions`.

The canonical example — where neither `REPEATABLE READ` alone nor per-row locking would catch the problem, because no two transactions ever touch the *same row*:

```sql
-- both transactions only read class=1 or class=2 rows, never the same one,
-- yet the combined outcome could never happen if run one at a time

-- Session A
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT sum(value) FROM mytab WHERE class = 1;   -- e.g. 30
INSERT INTO mytab (class, value) VALUES (2, 30);

-- Session B
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT sum(value) FROM mytab WHERE class = 2;   -- e.g. 300 (pre-Session-A-insert)
INSERT INTO mytab (class, value) VALUES (1, 300);

-- Session A
COMMIT;   -- succeeds

-- Session B
COMMIT;   -- ERROR: could not serialize access due to read/write dependencies among transactions
```

Neither transaction wrote a row the other read — this is why Repeatable Read's simple "did the row change" check can't catch it, but SSI's predicate-lock dependency graph can. Details in the [Serializable Isolation Level section](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE) and the original paper this is based on (referenced from that page).

A read-only transaction can be told `SERIALIZABLE READ ONLY DEFERRABLE` — it will block briefly at startup until Postgres can prove it needs no predicate locks at all, after which it runs with zero serialization-failure risk. This is the one case where `SERIALIZABLE` blocks and `REPEATABLE READ` doesn't.

[↑ back to top](#table-of-contents)

---

## Explicit Locking

<a id="explicit-locking-working-knowledge"></a>

### Working Knowledge — row-level locking

**What it is.** Sometimes you need to lock specific rows yourself — e.g. "read this row and I intend to update it, don't let anyone else change it out from under me in the meantime." That's what `SELECT ... FOR UPDATE` is for.

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- other transactions trying to UPDATE, DELETE, or also FOR UPDATE this row now wait
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

The four row-level lock strengths, weakest to strongest ([row-level locks docs](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS)):

- `FOR KEY SHARE` — weakest; only conflicts with locks that would change the row's key
- `FOR SHARE` — like a read lock; blocks concurrent updates/deletes, but multiple sessions can hold it at once
- `FOR NO KEY UPDATE` — taken implicitly by most `UPDATE`s that don't touch a primary/unique key column
- `FOR UPDATE` — strongest; exclusive intent to modify or delete, blocks essentially everything else on that row

**Wrong vs. right — the classic lost-update race:**

```sql
-- WRONG: read-then-write without locking lets two sessions both read
-- the same balance and both compute the same "new" value
-- Session A                          -- Session B
BEGIN;                                 BEGIN;
SELECT balance FROM accounts           SELECT balance FROM accounts
  WHERE id = 1;  -- 100                  WHERE id = 1;  -- 100
-- app computes 100 - 30 = 70          -- app computes 100 - 50 = 50
UPDATE accounts SET balance = 70       UPDATE accounts SET balance = 50
  WHERE id = 1;                          WHERE id = 1;  -- blocks until A commits
COMMIT;                                COMMIT;  -- final balance 50, the -30 is lost!
```

```sql
-- RIGHT: FOR UPDATE forces Session B to re-read the post-A-commit value
-- Session A                          -- Session B
BEGIN;                                 BEGIN;
SELECT balance FROM accounts           SELECT balance FROM accounts
  WHERE id = 1 FOR UPDATE;  -- 100       WHERE id = 1 FOR UPDATE;  -- blocks
UPDATE accounts SET balance = 70
  WHERE id = 1;
COMMIT;                                -- unblocks now, sees balance = 70
                                        -- app computes 70 - 50 = 20
                                        UPDATE accounts SET balance = 20 WHERE id = 1;
                                        COMMIT;  -- correct final balance
```

**Try it — feel the lock wait:**

```sql
-- Session A
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- leave open

-- Session B
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;   -- hangs — it's waiting on A

-- back in Session A
COMMIT;
-- Session B's SELECT immediately returns
```

`FOR UPDATE NOWAIT` fails immediately instead of waiting; `FOR UPDATE SKIP LOCKED` silently skips already-locked rows — the pattern behind most "grab the next job from a queue table" implementations, since multiple workers can poll the same table without piling up on each other.

[↑ back to top](#table-of-contents)

<a id="explicit-locking-advanced"></a>

### Advanced — table-level locks

Every SQL statement takes a table-level lock automatically (a plain `SELECT` takes `ACCESS SHARE`, an `UPDATE` takes `ROW EXCLUSIVE`, etc.), but you can also take one explicitly with `LOCK TABLE`. The eight modes, from weakest to strongest, are documented with their full conflict matrix in [Table 13.2](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-TABLES) — the two you'll reach for by hand most often:

```sql
-- block writers but let readers through (e.g. before a bulk maintenance job)
LOCK TABLE accounts IN SHARE MODE;

-- block absolutely everything, including other readers (e.g. before
-- a schema-changing DDL that must see no concurrent activity)
LOCK TABLE accounts IN ACCESS EXCLUSIVE MODE;
```

Most DDL (e.g. `ALTER TABLE ... ADD COLUMN` without a default, in current Postgres) takes `ACCESS EXCLUSIVE` implicitly — this is why running migrations on a busy table can cause a pile of blocked queries even though the migration itself is fast: everything queues up behind the DDL's lock acquisition.

[↑ back to top](#table-of-contents)

<a id="explicit-locking-mastery"></a>

### Mastery — advisory locks

**What they are.** Advisory locks are application-defined locks that Postgres tracks but never checks against any table or row — they mean whatever your application decides they mean. Useful for coordinating application-level critical sections (e.g. "only one instance of this cron job across the fleet") without inventing a real table to lock.

```sql
-- session-level: held until explicitly unlocked or the session ends
SELECT pg_advisory_lock(12345);
-- ... do exclusive work ...
SELECT pg_advisory_unlock(12345);

-- transaction-level: released automatically at COMMIT/ROLLBACK
BEGIN;
SELECT pg_advisory_xact_lock(12345);
-- ... do exclusive work ...
COMMIT;

-- non-blocking variant — returns true/false instead of waiting
SELECT pg_try_advisory_lock(12345);
```

See the [advisory locks docs](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS). Because they're arbitrary bigints, a common convention is hashing a string key (`SELECT pg_advisory_lock(hashtext('nightly-report-job'))`) to get a stable numeric lock ID from a human-readable name.

**Lock queue starvation.** Postgres's lock manager grants row/table locks in roughly first-come-first-served order, which sounds fair but has a sharp edge: a long-held `ACCESS SHARE` lock (an ordinary long-running `SELECT`) doesn't block other readers, but once *any* session requests a conflicting stronger lock (say, `ACCESS EXCLUSIVE` for a migration) and has to wait, **every subsequent lock request — including plain reads — queues up behind that waiting exclusive request**, even though reads and reads don't conflict with each other. This is why "just run the migration off-hours" isn't a full fix if a single long report query is still active — the migration's lock request can jam the queue for everything issued after it. Diagnose with:

```sql
SELECT pid, mode, granted, relation::regclass
FROM pg_locks
WHERE relation = 'accounts'::regclass
ORDER BY granted, pid;
```

Rows with `granted = false` are queued and waiting; if you see a long chain of `false` rows behind one blocked `ACCESS EXCLUSIVE` request, that's queue starvation in action, not a deadlock (nothing here is *cyclic* — it will resolve once the blocking transaction ends, just possibly much later than you'd expect).

[↑ back to top](#table-of-contents)

---

## Deadlocks

<a id="deadlocks-advanced"></a>

### Advanced — diagnosing and avoiding them

**What it is.** A deadlock happens when two transactions each hold a lock the other needs — a cycle with no way out. Postgres periodically checks for these cycles (governed by `deadlock_timeout`, default 1 second) and, when it finds one, kills one of the participating transactions to break the cycle, letting the other proceed.

```
Session A                     Session B
   |                              |
   | holds lock on row 1          | holds lock on row 2
   |                              |
   | wants lock on row 2  ───────▶│  (waits on B)
   |                              |
   |◀─────── wants lock on row 1  |  (waits on A)
   |                              |
        cycle: A waits on B, B waits on A → deadlock detected
```

**Wrong vs. right — inconsistent lock ordering:**

```sql
-- WRONG: Session A locks 1 then 2; Session B locks 2 then 1 — classic deadlock setup
-- Session A                          -- Session B
BEGIN;                                 BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 1;
                                        UPDATE accounts SET balance = balance - 10 WHERE id = 2;
UPDATE accounts SET balance = balance + 10 WHERE id = 2;  -- waits on B
                                        UPDATE accounts SET balance = balance + 10 WHERE id = 1;  -- waits on A
-- after deadlock_timeout, Postgres detects the cycle and aborts one:
-- ERROR: deadlock detected
-- DETAIL: Process 1234 waits for ShareLock on transaction 5678; blocked by process 5678.
--         Process 5678 waits for ShareLock on transaction 1234; blocked by process 1234.
```

```sql
-- RIGHT: always acquire locks in a consistent order (e.g. by ascending id)
-- across every code path that touches these rows
-- Session A                          -- Session B
BEGIN;                                 BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 1;
                                        -- B also touches id=1 first, waits cleanly on A
UPDATE accounts SET balance = balance + 10 WHERE id = 2;
COMMIT;                                UPDATE accounts SET balance = balance - 10 WHERE id = 1;
                                        UPDATE accounts SET balance = balance + 10 WHERE id = 2;
                                        COMMIT;
```

**Real Scenario — trigger a real deadlock yourself:**

```sql
-- setup, either session
CREATE TABLE accounts (id int primary key, balance int);
INSERT INTO accounts VALUES (1, 100), (2, 100);

-- Session A
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 1;

-- Session B
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 2;

-- Session A (will block, waiting on B's lock on id=2)
UPDATE accounts SET balance = balance + 10 WHERE id = 2;

-- Session B (completes the cycle)
UPDATE accounts SET balance = balance + 10 WHERE id = 1;
-- within ~1 second, one session gets: ERROR: deadlock detected
```

Whichever session's statement Postgres chooses to abort will show the `deadlock detected` error with a `DETAIL` line naming the exact processes and lock types involved — that detail is your diagnostic starting point in production logs. Avoidance is almost always a matter of **consistent lock ordering**: if every code path updates rows in the same order (e.g. always by ascending primary key), cycles like the one above can't form. See this pattern discussed on [Stack Overflow](https://stackoverflow.com/questions/17651670/postgresql-deadlock-detected) for real-world variations (foreign key checks and trigger-driven updates are common hidden sources of extra lock acquisitions that break an otherwise-consistent ordering).

[↑ back to top](#table-of-contents)

---

## VACUUM & Autovacuum

<a id="vacuum--autovacuum-advanced"></a>

### Advanced — why cleanup is needed, and monitoring bloat

**Why MVCC needs this.** As shown in [MVCC Internals](#mvcc-internals), `UPDATE` and `DELETE` never erase a row version immediately — they leave a dead tuple behind because some concurrent snapshot might still need it. Left alone forever, dead tuples accumulate into **bloat**: tables and indexes that are physically much larger than the live data they hold, wasting disk and slowing every scan that has to skip past dead versions. `VACUUM` is the process that reclaims this space for reuse (it does **not**, in its ordinary form, shrink the file on disk).

```sql
VACUUM VERBOSE accounts;     -- reclaim dead tuples, print stats
VACUUM ANALYZE accounts;     -- also refresh planner statistics
```

**`VACUUM` vs `VACUUM FULL`** — these are not "light" and "thorough" versions of the same operation, they behave very differently:

| | `VACUUM` | `VACUUM FULL` |
|---|---|---|
| Lock taken | `SHARE UPDATE EXCLUSIVE` — reads/writes continue | `ACCESS EXCLUSIVE` — blocks everything |
| What it does | Marks dead tuple space reusable in place | Rewrites the entire table into a new file, then drops the old one |
| Returns space to OS | Only trailing all-empty pages | Yes, fully compacted |
| Typical use | Routine maintenance (usually via autovacuum) | Rare — only after a bloat event autovacuum can't keep up with |

The [routine vacuuming docs](https://www.postgresql.org/docs/current/routine-vacuuming.html) are explicit: "administrators should strive to use standard `VACUUM` and avoid `VACUUM FULL`" — the downtime from an `ACCESS EXCLUSIVE` lock on a large table is usually far worse than the bloat it fixes. If a table needs `VACUUM FULL` regularly, that's a signal autovacuum is undertuned, not that `VACUUM FULL` should become routine.

**Monitoring bloat and progress:**

```sql
-- dead vs live tuple counts per table
SELECT relname, n_live_tup, n_dead_tup,
       round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0), 3) AS dead_ratio
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- watch a VACUUM while it runs, from another session
SELECT * FROM pg_stat_progress_vacuum;
```

`pg_stat_progress_vacuum` (available since Postgres 12, still the tool to reach for) shows `heap_blks_scanned` vs `heap_blks_total` so you can tell whether a long vacuum is almost done or barely started.

**Autovacuum tuning basics.** Autovacuum triggers a table's `VACUUM` once its dead-tuple count crosses a threshold computed from its size:

```
vacuum threshold = min(
  autovacuum_vacuum_max_threshold,
  autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * reltuples
)
```

Defaults in Postgres 18: `autovacuum_vacuum_threshold = 50`, `autovacuum_vacuum_scale_factor = 0.2` (20% of the table), capped by `autovacuum_vacuum_max_threshold = 100,000,000`. That cap is a genuinely new PG18 parameter — see the caveat below.

```sql
-- per-table overrides are usually better than changing the global default,
-- since a 10-row config table and a 500M-row events table shouldn't share tuning
ALTER TABLE events SET (autovacuum_vacuum_scale_factor = 0.02);
ALTER TABLE events SET (autovacuum_vacuum_cost_delay = 2);
```

For large, high-churn tables, a common real-world fix is lowering the *scale factor* on that table specifically (20% of a 500M-row table is 100M dead tuples before vacuum even considers running — often already painful bloat by the time it triggers).

[↑ back to top](#table-of-contents)

<a id="vacuum--autovacuum-mastery"></a>

### Mastery — transaction ID wraparound and freezing

**The problem.** Postgres XIDs are 32-bit — about 4.3 billion values — and visibility comparisons rely on being able to tell "older" from "newer" XIDs, which only works if the whole range hasn't wrapped around. Left unmanaged, a table's oldest surviving XID could wrap back around and start looking like it's *in the future* relative to current transactions, which would make old committed rows suddenly appear to not-yet-exist — silent, catastrophic data loss (not corruption Postgres can just detect and refuse — the visibility rule itself would be fooled).

**The fix: freezing.** Rows old enough that no live transaction could possibly need to distinguish "before" from "after" them get their `xmin` rewritten to a special sentinel, `FrozenTransactionId`, that always sorts as "in the past" no matter how far XIDs advance afterward.

```
XID space (32-bit, wraps at ~4.3 billion)

   ...──[frozen rows]──[normal rows, xmin still meaningful]──▶ current XID
        │
        FrozenTransactionId: permanently "in the past",
        immune to wraparound comparisons

  if the gap between the oldest un-frozen xmin and the current
  XID approaches ~2 billion, Postgres forces aggressive vacuums
  to freeze more before wraparound becomes a real risk
```

Key parameters (Postgres 18 defaults, per [runtime-config-vacuum](https://www.postgresql.org/docs/current/runtime-config-vacuum.html)):

| Parameter | Default | Role |
|---|---|---|
| `vacuum_freeze_min_age` | 50,000,000 | Minimum tuple age before a normal `VACUUM` will freeze it |
| `vacuum_freeze_table_age` | 150,000,000 | Table age that forces a full-table (non-skip-visible-pages) scan |
| `autovacuum_freeze_max_age` | 200,000,000 | Table age that forces an autovacuum run regardless of dead-tuple count |

```sql
-- check how close a table is to needing a forced freeze
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY xid_age DESC
LIMIT 10;

-- database-wide oldest frozen XID
SELECT datname, age(datfrozenxid) FROM pg_database;
```

If autovacuum is disabled or can't keep up and a database's XID age approaches the ~2-billion-transaction ceiling, Postgres starts emitting increasingly urgent warnings, then refuses new transactions entirely rather than risk wraparound corruption — at that point the only way forward is a manual `VACUUM` (not `VACUUM FULL`, which doesn't help freezing and takes a much worse lock) run in single-user mode if necessary. This is described in the [preventing wraparound section](https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND) of the docs.

**Version caveat — PG18 changed autovacuum tuning knobs.** If you're following an older tutorial or blog post, check these against [PostgreSQL 18's release notes](https://www.postgresql.org/docs/current/release-18.html) before trusting them:

- `autovacuum_vacuum_max_threshold` is a **new PG18 parameter** — a hard cap (default 100,000,000 tuples) on the computed vacuum-trigger threshold, so very large tables can no longer accumulate unbounded dead tuples before autovacuum notices.
- Worker scaling changed: `autovacuum_worker_slots` sets a startup ceiling on worker processes, while `autovacuum_max_workers` can now be adjusted without a restart (previously a restart-only setting) — see [Neon's writeup](https://neon.com/postgresql/postgresql-18/autovacuum-maintenance-configuration) and the [Microsoft Community Hub post](https://techcommunity.microsoft.com/blog/adforpostgresql/postgresql-18-vacuuming-improvements-explained/4459484) on PG18 vacuum changes for a practical walkthrough.
- Postgres 18 adds asynchronous I/O to vacuum's heap scanning and reports delay time explicitly (new `total_vacuum_time`/`total_autovacuum_time`-style columns), making `pg_stat_progress_vacuum` more informative than in older versions — don't assume a pre-18 tutorial's monitoring queries are still the full picture.

[↑ back to top](#table-of-contents)

---

## Cheat Sheets

**Isolation levels — anomalies prevented (✓) vs. allowed (✗):**

| Level | Dirty read | Nonrepeatable read | Phantom read | Serialization anomaly |
|---|---|---|---|---|
| Read Committed | ✓ prevented | ✗ allowed | ✗ allowed | ✗ allowed |
| Repeatable Read | ✓ prevented | ✓ prevented | ✓ prevented* | ✗ allowed |
| Serializable | ✓ prevented | ✓ prevented | ✓ prevented | ✓ prevented |

\* Postgres's Repeatable Read exceeds the SQL standard's minimum guarantee by also blocking phantom reads.

**Serialization failure quick reference:**

| Level | Error text | SQLSTATE | App response |
|---|---|---|---|
| Repeatable Read | `could not serialize access due to concurrent update` | `40001` | Retry transaction |
| Serializable | `could not serialize access due to read/write dependencies among transactions` | `40001` | Retry transaction |

**Row-level lock modes (weakest → strongest):**

| Mode | Typical use | Blocks |
|---|---|---|
| `FOR KEY SHARE` | Referencing FK checks | Only key-changing locks |
| `FOR SHARE` | "I read this, don't let it change" | Updates/deletes, not other `FOR SHARE` |
| `FOR NO KEY UPDATE` | Ordinary `UPDATE` not touching a key column | `FOR UPDATE`, other `FOR NO KEY UPDATE` |
| `FOR UPDATE` | "I intend to modify or delete this" | Nearly everything else on the row |

**Table-level lock compatibility (selected common modes — full matrix in the [docs](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-TABLES)):**

| Mode acquired | Conflicts with |
|---|---|
| `ACCESS SHARE` (plain `SELECT`) | `ACCESS EXCLUSIVE` only |
| `ROW EXCLUSIVE` (`UPDATE`/`DELETE`/`INSERT`) | `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE`, `ACCESS EXCLUSIVE` |
| `SHARE` (e.g. `CREATE INDEX` without `CONCURRENTLY`) | `ROW EXCLUSIVE` and stronger |
| `ACCESS EXCLUSIVE` (most DDL, `VACUUM FULL`) | everything, including plain reads |

**VACUUM vs VACUUM FULL:**

| | `VACUUM` | `VACUUM FULL` |
|---|---|---|
| Lock | `SHARE UPDATE EXCLUSIVE` | `ACCESS EXCLUSIVE` |
| Blocks reads/writes | No | Yes |
| Reclaims space to OS | Only trailing empty pages | Fully |
| Routine use | Yes (via autovacuum) | Avoid; last resort |

[↑ back to top](#table-of-contents)

---

← Back to [Postgres SQL Mastery Guide](./postgres-mastery-guide.md)
