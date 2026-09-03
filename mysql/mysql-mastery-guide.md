# MySQL Mastery Guide

A curiosity-driven, hands-on reference for mastering MySQL — built to be reopened and searched, not just read once. This file is the index: short summaries and links into six deep-dive files, each covering one major area at full depth.

## About This Document

- **Framing:** Curiosity-driven — no upcoming task or deadline drove this. It's structured as a guided, hands-on tutorial (runnable examples and exercises) rather than a historical/evolution narrative. Theory ("why it's designed this way") appears only where needed to explain a specific mechanism, not as standalone sections.
- **Depth tiers requested:** All four — **Beginner → Working Knowledge → Advanced → Mastery** — applied per section within each topic file, collapsing tiers that would be trivial for a given concept. The [Administration & Operations](./mysql-administration-operations.md) file deliberately collapses a full Mastery tier (deep ops internals like custom replication topologies or Performance Schema instrument-level tuning are their own specialty and out of scope for a SQL-focused series); Mastery-level asides there are folded into brief "goes deeper" notes under Advanced instead.
- **Confirmed scope:** Full standalone treatment (not assumed-familiarity-with-Postgres shorthand), as its own six-file series:
  1. Foundations — connecting, storage engines at a glance, DDL, core types, CRUD
  2. Core Querying — SELECT, joins, subqueries, CTEs, window functions, GROUP BY, set operations
  3. Query Engine & Indexing — EXPLAIN/EXPLAIN ANALYZE, clustered vs. secondary indexes, index types, optimizer statistics
  4. Data Modeling & Types — storage engines in depth, constraints, JSON, generated columns, ENUM/SET, temporal types, full-text search
  5. Transactions & Concurrency — MVCC internals, isolation levels, gap/next-key locks, explicit locking, deadlocks
  6. Administration & Operations — users/roles, backup/restore, replication, configuration, monitoring
- **File organization:** One main index (this file, with summaries + links) plus one file per major topic, per explicit request — rather than one giant document. Mirrors the structure of the companion [Postgres SQL Mastery Guide](../postgres/postgres-mastery-guide.md) in this repo, adapted to how MySQL itself actually divides (e.g. storage engines and InnoDB's clustered-index model get dedicated treatment since they have no Postgres equivalent).
- **Version baseline:** [MySQL 8.4 LTS](https://dev.mysql.com/doc/refman/8.4/en/) (the long-term-support release, chosen over the shorter-lived 9.x Innovation train since it's what most production deployments run). Notable 9.x-only features are flagged inline where relevant rather than assumed as baseline. Each topic file was checked against official docs before writing anything version-specific, and flags inline wherever current-version behavior contradicts older blog-post conventions (a real risk with MySQL, given how much 5.7-era advice — silent truncation, no CHECK enforcement, no CTEs/window functions — still circulates and no longer applies to 8.0+).
- **Sourcing convention used throughout:** Official docs (`dev.mysql.com/doc/refman`) and Stack Overflow cited inline first; other sites only as supplementary sources.
- **If you ask for updates later:** Match this structure — keep new content in the relevant topic file (not this index), update this index's summary/links if headers change, and preserve the tier labeling and "wrong vs. right" / "Real Scenario" / ASCII-diagram conventions used in the existing files.

## Table of Contents

- [1. Foundations](#1-foundations)
- [2. Core Querying](#2-core-querying)
- [3. Query Engine & Indexing](#3-query-engine--indexing)
- [4. Data Modeling & Types](#4-data-modeling--types)
- [5. Transactions & Concurrency](#5-transactions--concurrency)
- [6. Administration & Operations](#6-administration--operations)
- [Suggested Learning/Reference Order](#suggested-learningreference-order)
- [Quick Self-Check](#quick-self-check)

---

## 1. Foundations

📄 [`mysql-foundations.md`](./mysql-foundations.md)

The scaffolding: connecting via the `mysql` client, choosing a storage engine at `CREATE TABLE` time (and why that choice exists at all), `CREATE TABLE`/`ALTER TABLE` (including online DDL algorithms), core data types (with the `DATETIME` vs. `TIMESTAMP` gotcha), `SQL_MODE`, and INSERT/SELECT/UPDATE/DELETE fundamentals including `ON DUPLICATE KEY UPDATE`. Beginner/Working Knowledge only — this is where you start, not where mastery lives.

**Jump to:** [Connecting](./mysql-foundations.md#connecting-with-the-mysql-client) · [Storage Engines](./mysql-foundations.md#storage-engines-pick-one-before-you-do-anything-else) · [CREATE TABLE](./mysql-foundations.md#creating-tables-create-table) · [ALTER TABLE](./mysql-foundations.md#altering-tables-alter-table) · [Core Data Types](./mysql-foundations.md#core-data-types) · [SQL_MODE](./mysql-foundations.md#sql_mode-mysqls-strictness-dial) · [Cheat Sheet](./mysql-foundations.md#cheat-sheet)

> **Version note:** `STRICT_TRANS_TABLES` (part of the default `sql_mode` since 5.7) rejects invalid data instead of silently truncating/coercing it — a lot of older MySQL folklore about "silent" bad-data behavior no longer applies. `ALGORITHM=INSTANT` (8.0.12+) makes many common `ALTER TABLE` operations metadata-only.

[Back to top](#mysql-mastery-guide)

---

## 2. Core Querying

📄 [`mysql-core-querying.md`](./mysql-core-querying.md)

The heart of the language: SELECT/filtering, joins (and MySQL's lack of `FULL OUTER JOIN`), subqueries (the same `NOT IN`/NULL trap as Postgres), CTEs including recursive CTEs, window functions, `GROUP BY`/`ONLY_FULL_GROUP_BY`/`WITH ROLLUP`, and set operations. Built around one shared runnable schema, full Beginner → Mastery depth throughout.

**Jump to:** [SELECT & Filtering](./mysql-core-querying.md#select--filtering) · [Joins](./mysql-core-querying.md#joins) · [Subqueries](./mysql-core-querying.md#subqueries) · [CTEs](./mysql-core-querying.md#common-table-expressions-ctes) · [Window Functions](./mysql-core-querying.md#window-functions) · [GROUP BY & ONLY_FULL_GROUP_BY](./mysql-core-querying.md#group-by-and-only_full_group_by) · [Set Operations](./mysql-core-querying.md#set-operations) · [Cheat Sheet](./mysql-core-querying.md#cheat-sheet)

> **Version note:** CTEs and window functions didn't exist before MySQL 8.0 at all — this is new territory if you last used MySQL 5.7 or earlier, not a rename of something familiar. `INTERSECT`/`EXCEPT` only arrived in 8.0.31. There is still no `FULL OUTER JOIN` keyword; it has to be emulated with a `UNION`.

[Back to top](#mysql-mastery-guide)

---

## 3. Query Engine & Indexing

📄 [`mysql-query-engine-indexing.md`](./mysql-query-engine-indexing.md)

InnoDB's clustered-index model (the single biggest structural departure from Postgres — the table *is* a B-tree ordered by primary key), reading `EXPLAIN`/`EXPLAIN ANALYZE`, choosing an index type, optimizer statistics and histograms, diagnosing why the optimizer skipped an index, covering indexes, and MySQL 8.0's invisible/descending/functional indexes.

**Jump to:** [Clustered vs. Secondary Indexes](./mysql-query-engine-indexing.md#clustered-vs-secondary-indexes-innodbs-biggest-departure-from-postgres) · [Reading EXPLAIN](./mysql-query-engine-indexing.md#reading-explain-and-explain-analyze) · [Choosing an Index Type](./mysql-query-engine-indexing.md#choosing-an-index-type) · [Optimizer Statistics](./mysql-query-engine-indexing.md#optimizer-statistics-analyze-table-and-histograms) · [Why the Optimizer Skipped Your Index](./mysql-query-engine-indexing.md#why-the-optimizer-didnt-use-your-index) · [Covering Indexes](./mysql-query-engine-indexing.md#covering-indexes-and-index-only-access) · [Invisible/Descending/Functional Indexes](./mysql-query-engine-indexing.md#invisible-descending-and-functional-indexes) · [Cheat Sheet](./mysql-query-engine-indexing.md#cheat-sheet-index-types-compared)

> **Version note:** `EXPLAIN ANALYZE` (real timing, not estimates) arrived in 8.0.18 and always uses `FORMAT=TREE`. Invisible indexes, descending indexes, and functional (expression) indexes are all 8.0+ — none exist pre-8.0.

[Back to top](#mysql-mastery-guide)

---

## 4. Data Modeling & Types

📄 [`mysql-data-modeling-types.md`](./mysql-data-modeling-types.md)

Storage engines compared in depth (InnoDB vs. MyISAM vs. Memory vs. CSV), constraints (including `CHECK`'s late and easy-to-miss real enforcement date), the `JSON` type and `JSON_TABLE()`, generated columns (`STORED` vs. `VIRTUAL`), `ENUM`/`SET` and their schema-as-data trap, temporal type precision, and `FULLTEXT` search.

**Jump to:** [Storage Engines Compared](./mysql-data-modeling-types.md#storage-engines-innodb-vs-myisam-vs-the-rest) · [Constraints](./mysql-data-modeling-types.md#constraints) · [JSON](./mysql-data-modeling-types.md#json) · [Generated Columns](./mysql-data-modeling-types.md#generated-columns-stored-vs-virtual) · [ENUM & SET](./mysql-data-modeling-types.md#enum-and-set-convenient-then-a-trap) · [Temporal Types](./mysql-data-modeling-types.md#temporal-types-revisited) · [Full-Text Search](./mysql-data-modeling-types.md#full-text-search) · [Cheat Sheet](./mysql-data-modeling-types.md#cheat-sheet)

> **Version note:** `CHECK` constraints were parsed but silently **not enforced** before 8.0.16 — a genuinely dangerous trap for anyone reading pre-8.0.16 examples. `JSON_TABLE()` is 8.0.4+; indexing via `JSON_VALUE()` functional indexes is 8.0.21+.

[Back to top](#mysql-mastery-guide)

---

## 5. Transactions & Concurrency

📄 [`mysql-transactions-concurrency.md`](./mysql-transactions-concurrency.md)

Where MySQL diverges from Postgres most sharply: autocommit-by-default transaction basics, InnoDB's undo-log-based MVCC (versus Postgres's in-heap tuple versioning), all four isolation levels with two-session runnable exercises (MySQL defaults to `REPEATABLE READ`, not `READ COMMITTED`), InnoDB-specific gap locks and next-key locks, explicit locking including `SKIP LOCKED`, and deadlock diagnosis via `SHOW ENGINE INNODB STATUS`.

**Jump to:** [Transaction Basics](./mysql-transactions-concurrency.md#transaction-basics) · [MVCC Internals](./mysql-transactions-concurrency.md#mvcc-internals-undo-logs-not-tuple-versioning) · [Isolation Levels](./mysql-transactions-concurrency.md#isolation-levels) · [Gap & Next-Key Locks](./mysql-transactions-concurrency.md#gap-locks-and-next-key-locks) · [Explicit Locking](./mysql-transactions-concurrency.md#explicit-locking) · [Deadlocks](./mysql-transactions-concurrency.md#deadlocks) · [Cheat Sheets](./mysql-transactions-concurrency.md#cheat-sheets)

> **Version note:** None of this is version-gated in the same way as other files — InnoDB's MVCC and locking model has been stable in its fundamentals for a long time. The one thing worth double-checking against your exact version: `SKIP LOCKED`/`NOWAIT` and `SHOW REPLICA STATUS` (renamed from `SHOW SLAVE STATUS` in 8.0.22) are both 8.0+.

[Back to top](#mysql-mastery-guide)

---

## 6. Administration & Operations

📄 [`mysql-administration-operations.md`](./mysql-administration-operations.md)

MySQL as an operated service: users/privileges and 8.0+ roles, logical (`mysqldump`) vs. physical backup and point-in-time recovery via `mysqlbinlog`, binlog-based replication (`ROW`/`STATEMENT`/`MIXED` format, GTIDs), conceptual configuration tuning (`innodb_buffer_pool_size`, `innodb_flush_log_at_trx_commit`, `max_connections`), and monitoring via `performance_schema`. Beginner → Advanced only; Mastery-level ops internals are folded into brief "goes deeper" notes.

**Jump to:** [Users & Roles](./mysql-administration-operations.md#users-privileges-and-roles) · [Backup & Restore](./mysql-administration-operations.md#backup-and-restore) · [Replication Basics](./mysql-administration-operations.md#replication-basics) · [Configuration Knobs](./mysql-administration-operations.md#configuration-knobs-worth-knowing) · [Monitoring](./mysql-administration-operations.md#monitoring) · [Cheat Sheet](./mysql-administration-operations.md#cheat-sheet)

> **Version note:** `SHOW REPLICA STATUS` replaced `SHOW SLAVE STATUS` as the primary name in 8.0.22 (the old form still works but is deprecated terminology). Roles are 8.0+; pre-8.0 MySQL had no privilege-bundling mechanism beyond repeating `GRANT` per user.

[Back to top](#mysql-mastery-guide)

---

## Suggested Learning/Reference Order

1. **[Foundations](./mysql-foundations.md)** — get comfortable with the client, storage engine choice, and basic CRUD before anything else.
2. **[Core Querying](./mysql-core-querying.md)** — this is "SQL" in the everyday sense; spend the most time here first.
3. **[Transactions & Concurrency](./mysql-transactions-concurrency.md)** — read this before you write any concurrent application code; MySQL's default isolation level and gap-lock behavior will surprise you otherwise, and it's cheaper to learn from a runnable exercise than a production incident.
4. **[Query Engine & Indexing](./mysql-query-engine-indexing.md)** — once you're comfortable writing queries, learn why they're fast or slow, starting with the clustered-index model since it reframes everything else.
5. **[Data Modeling & Types](./mysql-data-modeling-types.md)** — deepen your schema design once you know how indexing and storage engines actually behave.
6. **[Administration & Operations](./mysql-administration-operations.md)** — come back to this once you're running MySQL somewhere real, not just querying it.

[Back to top](#mysql-mastery-guide)

---

## Quick Self-Check

- Why does `CREATE TABLE ... ENGINE=MyISAM` behave completely differently under concurrent writes than the same table with `ENGINE=InnoDB`?
- What actually happens, physically, when you look up a row by a non-primary-key indexed column in InnoDB? Why does primary key choice affect the size of *every other index* on the table?
- Why might `EXPLAIN` show `type: ALL` (full scan) on a column that has an index, and the optimizer be *right* to skip it?
- What's the default transaction isolation level in MySQL, and how does it differ from Postgres's default? What concrete behavior changes because of it?
- What's a gap lock, and why can an `INSERT` for a row that doesn't exist yet block on a `SELECT ... FOR UPDATE` that never touched that exact row?
- Why is `ONLY_FULL_GROUP_BY` on by default, and when is `ANY_VALUE()` the right tool versus just adding a column to `GROUP BY`?
- What's the difference between `STORED` and `VIRTUAL` generated columns, and which do you need if you want to index the result?
- Why does `mysqldump` need `--single-transaction` on an InnoDB database, and what would happen without it?
- What's the difference between `ROW`, `STATEMENT`, and `MIXED` binlog format, and why is `ROW` the safer default?
- If `autocommit` is on by default, what does that mean for a lone `UPDATE` statement run outside an explicit `START TRANSACTION`?

[Back to top](#mysql-mastery-guide)
