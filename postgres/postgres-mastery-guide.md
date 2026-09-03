# Postgres SQL Mastery Guide

A curiosity-driven, hands-on reference for mastering Postgres SQL — built to be reopened and searched, not just read once. This file is the index: short summaries and links into six deep-dive files, each covering one major area at full depth.

## About This Document

- **Framing:** Curiosity-driven — no upcoming task or deadline drove this. It's structured as a guided, hands-on tutorial (runnable examples and exercises) rather than a historical/evolution narrative. Theory ("why it's designed this way") appears only where needed to explain a specific mechanism, not as standalone sections.
- **Depth tiers requested:** All four — **Beginner → Working Knowledge → Advanced → Mastery** — applied per section within each topic file, collapsing tiers that would be trivial for a given concept. The [Administration & Operations](./postgres-administration-operations.md) file deliberately collapses a full Mastery tier (deep ops internals like WAL internals or custom replication topologies are their own specialty and out of scope for a SQL-focused series); Mastery-level asides there are folded into brief "goes deeper" notes under Advanced instead.
- **Confirmed scope:** All four candidate angles, each as its own file:
  1. Core SQL + query engine (split into **Core Querying** and **Query Engine & Indexing** below, since each warranted full-depth treatment on its own)
  2. Transactions & concurrency
  3. Data modeling & advanced types
  4. Administration & operations
  - Plus a **Foundations** file (Beginner/Working Knowledge only, no Mastery tier) covering the scaffolding — psql, basic DDL/DML, core scalar types — that the other files assume.
- **File organization:** One main index (this file, with summaries + links) plus one file per major topic, per explicit request — rather than one giant document.
- **Version baseline:** [PostgreSQL 18](https://www.postgresql.org/docs/current/) (current stable, patch 18.6 as of August 2026). Each topic file checked official docs before writing anything version-specific and flagged inline wherever recent-version behavior contradicts older blog-post conventions (PostgreSQL 19 is in beta and intentionally not relied on).
- **Sourcing convention used throughout:** Official docs and Stack Overflow cited inline first; GitHub/other sites only as supplementary sources.
- **If you ask for updates later:** Match this structure — keep new content in the relevant topic file (not this index), update this index's summary/links if headers change, and preserve the tier labeling and "wrong vs. right" / "Real Scenario" / ASCII-diagram conventions used in the existing files.

## Table of Contents

- [1. Foundations](#1-foundations)
- [2. Core Querying](#2-core-querying)
- [3. Query Engine & Indexing](#3-query-engine--indexing)
- [4. Data Modeling & Advanced Types](#4-data-modeling--advanced-types)
- [5. Transactions & Concurrency](#5-transactions--concurrency)
- [6. Administration & Operations](#6-administration--operations)
- [Suggested Learning/Reference Order](#suggested-learningreference-order)
- [Quick Self-Check](#quick-self-check)

---

## 1. Foundations

📄 [`postgres-foundations.md`](./postgres-foundations.md)

The scaffolding: connecting via `psql` (and why its meta-commands use `\`), schemas and the search path, `CREATE TABLE`/`ALTER TABLE` (including identity columns and safe patterns on large tables), the core scalar types (numeric/text/boolean/date-time), and INSERT/SELECT/UPDATE/DELETE fundamentals. Beginner/Working Knowledge only — this is where you start, not where mastery lives.

**Jump to:** [Connecting with psql](./postgres-foundations.md#connecting-with-psql) · [Schemas & Search Path](./postgres-foundations.md#schemas--the-search-path) · [CREATE TABLE](./postgres-foundations.md#creating-tables-create-table) · [ALTER TABLE](./postgres-foundations.md#altering-tables-alter-table) · [Core Data Types](./postgres-foundations.md#core-data-types) · [Cheat Sheet](./postgres-foundations.md#cheat-sheet)

> **Version note:** Identity columns (`GENERATED ... AS IDENTITY`) are now the docs-recommended replacement for `serial`. Adding a column with a constant default no longer rewrites the table. `char`/`varchar`/`text` have no performance difference in current docs, despite persistent legacy advice otherwise.

[Back to top](#postgres-sql-mastery-guide)

---

## 2. Core Querying

📄 [`postgres-core-querying.md`](./postgres-core-querying.md)

The heart of the language: SELECT and filtering, every join type (including self-joins and LATERAL), subqueries (with the classic `NOT IN` + NULL trap), CTEs including recursive CTEs with `SEARCH`/`CYCLE`, window functions and frame clauses, GROUP BY/GROUPING SETS/ROLLUP/CUBE, and set operations. Built around one shared runnable schema so examples compose across the whole file, full Beginner → Mastery depth throughout.

**Jump to:** [SELECT Fundamentals](./postgres-core-querying.md#select-fundamentals--filtering) · [Joins](./postgres-core-querying.md#joins) · [Subqueries](./postgres-core-querying.md#subqueries) · [CTEs](./postgres-core-querying.md#common-table-expressions-ctes) · [Window Functions](./postgres-core-querying.md#window-functions) · [Aggregates & Grouping Sets](./postgres-core-querying.md#aggregates-group-by--grouping-sets) · [Set Operations](./postgres-core-querying.md#set-operations-union-intersect-except) · [Cheat Sheet](./postgres-core-querying.md#quick-reference-cheat-sheet)

> **Version note:** PostgreSQL 18 sped up `INTERSECT`/`EXCEPT`/window-aggregates/hash joins/GROUP BY, extended GROUP BY functional-dependency elision to any unique index (not just primary keys), added `HAVING` pushdown for `GROUPING SETS`, fixed a prior `GROUPING SETS` correctness bug, and added planner self-join elimination.

[Back to top](#postgres-sql-mastery-guide)

---

## 3. Query Engine & Indexing

📄 [`postgres-query-engine-indexing.md`](./postgres-query-engine-indexing.md)

Reading `EXPLAIN`/`EXPLAIN ANALYZE`, the planner's cost model, statistics/`ANALYZE`, index types (B-tree/GIN/GiST/BRIN/hash) and when each fits, diagnosing planner misfires (type casts, leading-wildcard `LIKE`, unindexed function calls), partial/expression/multicolumn indexes, index-only scans — up through PG18's B-tree skip scan, visibility maps, extended statistics, and the genetic query optimizer. Built around one shared 2M-row `orders` table across every tier.

**Jump to:** [Reading EXPLAIN](./postgres-query-engine-indexing.md#beginner-reading-explain-and-explain-analyze) · [Cost Model & Index Types](./postgres-query-engine-indexing.md#working-knowledge-the-cost-model-and-choosing-an-index-type) · [Statistics & Autovacuum](./postgres-query-engine-indexing.md#working-knowledge-statistics-analyze-and-autovacuum) · [Why the Planner Didn't Use Your Index](./postgres-query-engine-indexing.md#advanced-why-the-planner-didnt-use-your-index) · [Partial & Expression Indexes](./postgres-query-engine-indexing.md#advanced-partial-and-expression-indexes) · [B-Tree Skip Scan (PG18)](./postgres-query-engine-indexing.md#mastery-b-tree-skip-scan-postgresql-18) · [Cheat Sheet: Index Types Compared](./postgres-query-engine-indexing.md#cheat-sheet-index-types-compared)

> **Version note:** PG18 makes `BUFFERS` implicit whenever `ANALYZE` is used in `EXPLAIN`. B-tree skip scan lifts the old "leading column is mandatory" limitation for composite indexes, though cost still governs whether it's actually chosen.

[Back to top](#postgres-sql-mastery-guide)

---

## 4. Data Modeling & Advanced Types

📄 [`postgres-data-modeling-types.md`](./postgres-data-modeling-types.md)

Constraints and deliberate denormalization, PG18's new temporal constraints (`WITHOUT OVERLAPS`/`PERIOD`) and virtual generated columns, JSONB (GIN indexing, `jsonb_path_query`, extended statistics), arrays, ENUM migration pain, full-text search with ranking, and a pointer to `pg_trgm`/PostGIS.

**Jump to:** [Constraints](./postgres-data-modeling-types.md#constraints-the-rules-your-data-must-obey) · [Normalization Tradeoffs](./postgres-data-modeling-types.md#normalization-tradeoffs-when-to-denormalize-on-purpose) · [Temporal Constraints (PG18)](./postgres-data-modeling-types.md#temporal-constraints-new-in-postgresql-18) · [Generated Columns](./postgres-data-modeling-types.md#generated-columns-stored-vs-virtual) · [JSONB](./postgres-data-modeling-types.md#jsonb-binary-json-with-indexing) · [Arrays](./postgres-data-modeling-types.md#arrays) · [ENUM Gotchas](./postgres-data-modeling-types.md#enum-types-and-their-gotchas) · [Full-Text Search](./postgres-data-modeling-types.md#full-text-search) · [Cheat Sheet](./postgres-data-modeling-types.md#cheat-sheet)

> **Version note:** `WITHOUT OVERLAPS`/`PERIOD` temporal constraints are PG18-only (older material only has the `EXCLUDE USING gist` equivalent). PG18 flips the generated-column default from `STORED` to `VIRTUAL` when no keyword is given — every pre-18 example assumes `STORED`. Temporal FKs don't yet support `CASCADE`/`SET NULL`/`SET DEFAULT`/`RESTRICT`.

[Back to top](#postgres-sql-mastery-guide)

---

## 5. Transactions & Concurrency

📄 [`postgres-transactions-concurrency.md`](./postgres-transactions-concurrency.md)

Where genuine concurrency mastery lives: transaction basics/savepoints, MVCC internals (xmin/xmax tuple versioning), all three isolation levels with two-session runnable exercises, explicit/advisory locking, deadlock diagnosis, and VACUUM/autovacuum through the lens of transaction ID wraparound and freezing.

**Jump to:** [Transaction Basics](./postgres-transactions-concurrency.md#transaction-basics) · [MVCC Internals](./postgres-transactions-concurrency.md#mvcc-internals) · [Isolation Levels](./postgres-transactions-concurrency.md#isolation-levels) · [Explicit Locking](./postgres-transactions-concurrency.md#explicit-locking) · [Deadlocks](./postgres-transactions-concurrency.md#deadlocks) · [VACUUM & Autovacuum](./postgres-transactions-concurrency.md#vacuum--autovacuum) · [Cheat Sheets](./postgres-transactions-concurrency.md#cheat-sheets)

> **Version note:** PG18 added `autovacuum_vacuum_max_threshold` (a hard cap, default 100M tuples), made `autovacuum_worker_slots`/`autovacuum_max_workers` adjustable without a restart, added async I/O to vacuum heap scans, and enriched `pg_stat_progress_vacuum`. Postgres still has no true Read Uncommitted — it silently maps to Read Committed.

[Back to top](#postgres-sql-mastery-guide)

---

## 6. Administration & Operations

📄 [`postgres-administration-operations.md`](./postgres-administration-operations.md)

Postgres as an operated service rather than a query target: roles/permissions and row-level security, logical vs. physical backups and point-in-time recovery, streaming vs. logical replication, conceptual configuration tuning (`shared_buffers`, `work_mem`, `max_connections`, checkpoints), and monitoring via `pg_stat_activity`/`pg_stat_statements`. Beginner → Advanced only; Mastery-level asides are folded into brief "goes deeper" notes.

**Jump to:** [Roles & Permissions](./postgres-administration-operations.md#roles-and-permissions) · [Backup & Restore](./postgres-administration-operations.md#backup-and-restore) · [Replication Basics](./postgres-administration-operations.md#replication-basics) · [Configuration Knobs](./postgres-administration-operations.md#configuration-knobs-worth-knowing) · [Monitoring](./postgres-administration-operations.md#monitoring) · [Cheat Sheet](./postgres-administration-operations.md#cheat-sheet)

> **Version note:** PG18's `pg_upgrade` now preserves planner statistics across major-version upgrades (opt out via `--no-statistics`) and runs faster generally. OAuth 2.0 is now a supported `pg_hba.conf` authentication method. Role `INHERIT` has been the default since Postgres 16 — older material may assume `SET ROLE` was required.

[Back to top](#postgres-sql-mastery-guide)

---

## Suggested Learning/Reference Order

1. **[Foundations](./postgres-foundations.md)** — get comfortable with psql, tables, and basic CRUD before anything else.
2. **[Core Querying](./postgres-core-querying.md)** — this is "SQL" in the everyday sense; spend the most time here first.
3. **[Query Engine & Indexing](./postgres-query-engine-indexing.md)** — once you can write queries, learn to read what Postgres does with them.
4. **[Data Modeling & Advanced Types](./postgres-data-modeling-types.md)** — JSONB, arrays, and constraints matter more once you're designing schemas, not just querying existing ones.
5. **[Transactions & Concurrency](./postgres-transactions-concurrency.md)** — the deepest and most rewarding file; best tackled once querying and indexing feel natural, since the two-session exercises assume fluency with basic queries.
6. **[Administration & Operations](./postgres-administration-operations.md)** — last, since it treats Postgres as a service rather than a query target; useful even for pure app developers who need to reason about backups, roles, and replication.

*Query Engine & Indexing and Transactions & Concurrency can be swapped or interleaved — both assume Core Querying but not each other.*

[Back to top](#postgres-sql-mastery-guide)

## Quick Self-Check

Work through these without looking at the docs first. If one stalls you, that's your cue for which file to revisit.

1. Why does `NOT IN (SELECT ... )` silently return zero rows when the subquery produces a NULL, and how do you avoid it? *(→ [Core Querying](./postgres-core-querying.md#subqueries))*
2. What's the difference between a window function's `ROWS` and `RANGE` frame modes, and when does it change your result? *(→ [Core Querying](./postgres-core-querying.md#window-functions))*
3. Given an `EXPLAIN ANALYZE` plan, how do you tell whether the planner's row-count estimate was wrong, and what usually causes that? *(→ [Query Engine & Indexing](./postgres-query-engine-indexing.md#working-knowledge-statistics-analyze-and-autovacuum))*
4. Why won't Postgres use an index on `WHERE some_int_column = '5'` in every case, and what's the fix? *(→ [Query Engine & Indexing](./postgres-query-engine-indexing.md#advanced-why-the-planner-didnt-use-your-index))*
5. When would you reach for a `VIRTUAL` generated column instead of `STORED`, and what's the tradeoff? *(→ [Data Modeling & Advanced Types](./postgres-data-modeling-types.md#generated-columns-stored-vs-virtual))*
6. Why can a JSONB containment query (`@>`) use a GIN index while a JSONB path query might not, without the right operator class? *(→ [Data Modeling & Advanced Types](./postgres-data-modeling-types.md#jsonb-binary-json-with-indexing))*
7. Under Read Committed, two concurrent transactions both read a row, then both update it — what does each one see, and why doesn't this deadlock? *(→ [Transactions & Concurrency](./postgres-transactions-concurrency.md#isolation-levels))*
8. What's actually happening at the tuple level (xmin/xmax) when you `UPDATE` a row, and why does that make `VACUUM` necessary? *(→ [Transactions & Concurrency](./postgres-transactions-concurrency.md#mvcc-internals))*
9. Two transactions each lock a different row and then try to lock the other's row — what does Postgres do, and how would you have prevented it? *(→ [Transactions & Concurrency](./postgres-transactions-concurrency.md#deadlocks))*
10. Why does `pg_dump`'s custom format (`--format=custom`) matter for `pg_restore`, and what can't you do with a plain-SQL dump? *(→ [Administration & Operations](./postgres-administration-operations.md#backup-and-restore))*
11. What's the practical difference between streaming replication and logical replication, and why does logical replication care about replica identity? *(→ [Administration & Operations](./postgres-administration-operations.md#replication-basics))*

[Back to top](#postgres-sql-mastery-guide)
