# Postgres Administration & Operations

Part of the [Postgres SQL Mastery Guide](./postgres-mastery-guide.md).

This file treats Postgres as a **service you operate**, not just a place to send queries. It covers who can touch your data, how you get it back after something goes wrong, how a database stays available across machines, which server knobs are worth knowing about, and how you'd notice trouble before a user does.

Baseline: **PostgreSQL 18** (18.6 as of Aug 2026) per the [official docs](https://www.postgresql.org/docs/current/). Two 18-specific things worth flagging up front, since older blog posts won't mention them:

- **`pg_upgrade` got noticeably faster**, and as of 18 it [preserves planner statistics across a major-version upgrade](https://www.postgresql.org/docs/current/pgupgrade.html) instead of forcing a full re-`ANALYZE` before performance recovers (pass `--no-statistics` to opt back into the old clean-slate behavior).
- **OAuth 2.0 is now a supported authentication method** in `pg_hba.conf` alongside password and certificate auth — relevant if your org wants Postgres logins tied to an SSO provider instead of managing passwords or certs directly.

## Table of Contents

- [Roles and Permissions](#roles-and-permissions)
  - [Beginner](#roles-beginner)
  - [Working Knowledge](#roles-working-knowledge)
  - [Advanced](#roles-advanced)
- [Backup and Restore](#backup-and-restore)
  - [Beginner](#backup-beginner)
  - [Working Knowledge](#backup-working-knowledge)
  - [Advanced](#backup-advanced)
- [Replication Basics](#replication-basics)
  - [Beginner](#replication-beginner)
  - [Working Knowledge](#replication-working-knowledge)
  - [Advanced](#replication-advanced)
- [Configuration Knobs Worth Knowing](#configuration-knobs-worth-knowing)
  - [Beginner](#config-beginner)
  - [Working Knowledge](#config-working-knowledge)
  - [Advanced](#config-advanced)
- [Monitoring](#monitoring)
  - [Beginner](#monitoring-beginner)
  - [Working Knowledge](#monitoring-working-knowledge)
  - [Advanced](#monitoring-advanced)
- [Cheat Sheet](#cheat-sheet)

---

## Roles and Permissions

<a id="roles-beginner"></a>

### Beginner

**What it is.** Postgres has no separate concept of "users" vs. "groups" — everything is a **role**. A role can log in (`LOGIN` attribute) or not; a role without login is really just a named bucket of permissions other roles can inherit. This unification trips people up coming from MySQL, where users and privileges are more separate. See the [official role docs](https://www.postgresql.org/docs/current/user-manag.html).

```sql
-- Create a login-capable role (a "user" in common parlance)
CREATE ROLE app_reader WITH LOGIN PASSWORD 'change_me' NOSUPERUSER;

-- Create a non-login role to act as a permission group
CREATE ROLE reporting_team;
```

**Wrong vs. right — the lazy grant:**

```sql
-- WRONG: gives app_reader full control of the database, including DROP TABLE
GRANT ALL PRIVILEGES ON DATABASE analytics TO app_reader;
```

```sql
-- RIGHT: least-privilege — only what a read-only reporting client needs
GRANT CONNECT ON DATABASE analytics TO app_reader;
GRANT USAGE ON SCHEMA public TO app_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_reader;
```

`GRANT ALL PRIVILEGES` is the SQL equivalent of handing someone your house keys because they asked to borrow a cup of sugar — it works, right up until it doesn't. The [GRANT reference](https://www.postgresql.org/docs/current/sql-grant.html) lists exactly which privilege words exist per object type (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES`, `TRIGGER`, ...) — grant only the ones the role's job actually requires.

**Real Scenario — build and verify a restricted role:**

```sql
-- 1. Set up a tiny sandbox
CREATE TABLE secrets (id serial PRIMARY KEY, value text);
INSERT INTO secrets (value) VALUES ('do not leak me');

-- 2. Create a role that can only read
CREATE ROLE audit_bot WITH LOGIN PASSWORD 'auditpass';
GRANT CONNECT ON DATABASE postgres TO audit_bot;
GRANT USAGE ON SCHEMA public TO audit_bot;
GRANT SELECT ON secrets TO audit_bot;
```

```bash
# 3. Connect AS that role and prove the restriction holds
psql -U audit_bot -d postgres -c "SELECT * FROM secrets;"   -- works
psql -U audit_bot -d postgres -c "DELETE FROM secrets;"     -- ERROR: permission denied
```

If step 3's `DELETE` doesn't fail, the grant was too broad — go back and check whether `audit_bot` inherited privileges from a role you forgot about (see role inheritance below).

[↑ back to top](#table-of-contents)

<a id="roles-working-knowledge"></a>

### Working Knowledge

**Role inheritance.** Roles can be members of other roles, and by default a member automatically has (inherits) the privileges of the roles it belongs to — no extra `SET ROLE` needed since Postgres 16 made `INHERIT` the sane default. This is how you build permission "groups":

```sql
CREATE ROLE reporting_team;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporting_team;

CREATE ROLE alice WITH LOGIN PASSWORD 'x';
GRANT reporting_team TO alice;   -- alice now inherits every SELECT the group has
```

Add someone to the team, and they get every grant the team has — remove them, and it's revoked in one place instead of hunting down individual `GRANT` statements per person.

**Default privileges for future objects.** A common gotcha: `GRANT SELECT ON ALL TABLES IN SCHEMA public` only affects tables that exist *right now*. A table created tomorrow won't have that grant unless you also set a default:

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO reporting_team;
```

This is a [frequent Stack Overflow question](https://stackoverflow.com/questions/6884020/grant-all-privileges-on-database-postgres) — people run `GRANT ALL ... ON ALL TABLES`, add a new table next week, and can't figure out why the new table isn't readable by the role that could read everything else.

**Revoking.** `REVOKE` mirrors `GRANT` exactly:

```sql
REVOKE SELECT ON secrets FROM audit_bot;
REVOKE reporting_team FROM alice;
```

[↑ back to top](#table-of-contents)

<a id="roles-advanced"></a>

### Advanced

**Row-level security (RLS) basics.** Grants control access to whole tables. **Row-level security** filters which *rows* a role sees within a table it's already allowed to query — critical for multi-tenant schemas where every tenant shares one `orders` table. See the [official RLS docs](https://www.postgresql.org/docs/current/ddl-rowsecurity.html).

```sql
CREATE TABLE orders (id serial PRIMARY KEY, tenant text, amount numeric);
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
  USING (tenant = current_setting('app.current_tenant'));
```

```bash
# The app sets a session variable per connection/request, then every
# query against `orders` is transparently filtered by the policy above:
SET app.current_tenant = 'acme_corp';
SELECT * FROM orders;   -- only acme_corp's rows come back, even with SELECT *
```

Two things people miss: (1) **table owners and superusers bypass RLS by default** — the policy only binds ordinary roles unless you also run `ALTER TABLE ... FORCE ROW LEVEL SECURITY`; (2) enabling RLS with *no* policy defined means **no rows are visible at all**, which reads as "my data disappeared" the first time someone hits it.

*Goes deeper:* Postgres also supports label-based mandatory access control via the `sepgsql` extension, and fine-grained column-level privileges (`GRANT SELECT (col1, col2) ON table`) for cases where row filtering isn't granular enough — both are edge cases outside what most teams need day to day.

[↑ back to top](#table-of-contents)

---

## Backup and Restore

<a id="backup-beginner"></a>

### Beginner

**What it is.** A **logical backup** (`pg_dump`) captures your data as a sequence of SQL statements that reconstruct it — portable across Postgres versions and even architectures. A **physical backup** (`pg_basebackup`, filesystem snapshot) copies the actual on-disk files — faster for huge databases, but tied to the exact server version and settings it came from. See the [official backup docs](https://www.postgresql.org/docs/current/backup.html).

```bash
# Logical backup of one database, plain SQL text
pg_dump mydb > mydb_backup.sql

# Restore it into a fresh database
createdb mydb_restored
psql -d mydb_restored -f mydb_backup.sql
```

**Real Scenario — back up and restore a small local database:**

```bash
createdb pg_admin_demo
psql -d pg_admin_demo -c "CREATE TABLE widgets (id serial PRIMARY KEY, name text);"
psql -d pg_admin_demo -c "INSERT INTO widgets (name) VALUES ('gadget'), ('gizmo');"

pg_dump pg_admin_demo > pg_admin_demo.sql

dropdb pg_admin_demo                       # simulate "disaster"
createdb pg_admin_demo
psql -d pg_admin_demo -f pg_admin_demo.sql

psql -d pg_admin_demo -c "SELECT * FROM widgets;"   -- both rows are back
```

[↑ back to top](#table-of-contents)

<a id="backup-working-knowledge"></a>

### Working Knowledge

**Wrong vs. right — picking a dump format for selective restore:**

```bash
# WRONG: plain SQL text can only be replayed top-to-bottom with psql —
# there's no way to restore just one table out of the file afterward
pg_dump mydb > mydb.sql
```

```bash
# RIGHT: custom format is compressed AND lets pg_restore cherry-pick objects
pg_dump -Fc mydb > mydb.dump
pg_restore -d mydb_restored --table=widgets mydb.dump
```

The `-F c` (custom) format is `pg_dump`'s recommended default for anything beyond a quick throwaway dump — see the [pg_dump docs](https://www.postgresql.org/docs/current/app-pgdump.html) and [pg_restore docs](https://www.postgresql.org/docs/current/app-pgrestore.html). A `-Fd` (directory) format goes further and enables parallel dump/restore with `-j N`, which matters once a database is large enough that a single-threaded dump takes hours.

```bash
# Parallel dump and restore, 4 workers
pg_dump -j 4 -Fd -f mydb_dumpdir mydb
pg_restore -j 4 -d mydb_restored mydb_dumpdir
```

**Cluster-wide backups.** `pg_dump` only covers one database. Roles, tablespaces, and other cluster-level objects live outside any single database, so a full-cluster backup needs `pg_dumpall` too (typically just `--globals-only`, with per-database dumps handled by `pg_dump`):

```bash
pg_dumpall --globals-only > globals.sql
```

**Version-mismatch gotcha:** `pg_restore`/`psql` must be from a version *at or newer than* the target server — restoring an 18-era dump with a Postgres 14 client's `pg_restore` binary is a [recurring Stack Overflow support question](https://stackoverflow.com/questions/tagged/pg-restore) that usually traces back to whatever `pg_restore` happened to be on `$PATH`. `pg_dump`/`pg_restore` themselves are forward/backward compatible across server versions in the documented sense — it's the *client binary version* that needs to be current.

[↑ back to top](#table-of-contents)

<a id="backup-advanced"></a>

### Advanced

**Physical backups and point-in-time recovery (PITR).** A physical backup (`pg_basebackup`) plus continuous **WAL (Write-Ahead Log) archiving** lets you restore to *any moment*, not just the instant the backup was taken — the concept behind PITR. See [continuous archiving and PITR](https://www.postgresql.org/docs/current/continuous-archiving.html).

```
 base backup (files as of T0)
        │
        ▼
 ┌─────────────┐   WAL segments archived continuously   ┌──────────────┐
 │  Primary DB │ ───────────────────────────────────────▶│ WAL archive  │
 └─────────────┘                                          └──────────────┘
        │                                                        │
        │  restore: base backup + replay archived WAL up to      │
        │  a chosen target (timestamp / transaction / LSN) ◀─────┘
        ▼
 ┌───────────────────────┐
 │ Recovered DB, at any   │
 │ point since the base   │
 │ backup was taken       │
 └───────────────────────┘
```

The mental model: the base backup is a coarse starting point; the WAL is a replayable transaction log. Recovery = "start from the snapshot, then replay history up to the moment right before the mistake" (e.g., right before an accidental `DROP TABLE`). This is *why* production Postgres setups archive WAL continuously rather than relying on periodic `pg_dump` snapshots alone — a snapshot-only strategy can lose up to a full backup interval's worth of data.

```bash
pg_basebackup -D /var/lib/postgresql/backup -Fp -Xs -P
```

**Logical vs. physical, side by side for backup purposes:**

| | Logical (`pg_dump`) | Physical (`pg_basebackup` + WAL) |
|---|---|---|
| Portable across major versions | Yes | No — same major version only |
| Selective restore (one table) | Yes (custom/directory format) | No — whole cluster |
| Point-in-time recovery | No | Yes, with WAL archiving |
| Typical use | Migrations, small-to-medium DBs, exports | Production DR, large databases |

*Goes deeper:* the actual WAL record format, checksums, and how the redo/undo-free design guarantees crash-consistency are internals territory — the [WAL docs](https://www.postgresql.org/docs/current/wal.html) are the entry point if that's ever worth a specialty deep-dive on its own.

[↑ back to top](#table-of-contents)

---

## Replication Basics

<a id="replication-beginner"></a>

### Beginner

**What it is.** Replication keeps a second copy of the database (a **standby**/**replica**) continuously up to date with a **primary**, so you have a hot spare for failover and/or a place to offload read traffic. Postgres's built-in mechanism is **streaming replication**: the replica connects to the primary and receives a live stream of WAL records, replaying them as they arrive. See the [high availability docs](https://www.postgresql.org/docs/current/high-availability.html).

```
 ┌───────────┐   WAL stream (continuous)   ┌────────────┐
 │  Primary  │ ───────────────────────────▶│  Replica   │
 │ (read/    │                              │ (read-only, │
 │  write)   │◀─── feedback / ack ──────────│  "hot       │
 └───────────┘                              │  standby")  │
                                             └────────────┘
```

A replica running in **hot standby** mode can serve read-only queries while continuously replaying WAL from the primary — this is the basis of a **read replica**: point reporting/analytics traffic at the replica so it doesn't compete with production writes for the primary's resources.

[↑ back to top](#table-of-contents)

<a id="replication-working-knowledge"></a>

### Working Knowledge

**Setting up streaming replication (conceptual walkthrough).** The primary needs `wal_level = replica` (or higher) and a role with the `REPLICATION` attribute; the replica connects to it and enters continuous recovery:

```sql
-- On the primary: a dedicated replication role
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replpass';
```

```bash
# On the replica: seed it from the primary, then start it in standby mode
pg_basebackup -h primary_host -U replicator -D /var/lib/postgresql/data -R
```

The `-R` flag writes a `standby.signal` file and the connection info Postgres needs to keep streaming from the primary automatically on startup — see [`pg_basebackup`'s `-R` docs](https://www.postgresql.org/docs/current/app-pgbasebackup.html).

**Replication slots** solve a specific failure mode: if a replica disconnects and the primary has already recycled the WAL it needed, the replica can't catch back up. A **replication slot** tells the primary "don't discard WAL this replica hasn't consumed yet, no matter how far behind it falls" — at the cost that a permanently-disconnected replica with a slot can fill up the primary's disk with retained WAL if nobody notices. This is why "replication lag" is on the monitoring checklist below.

**Read replicas in practice.** Any hot-standby replica can serve reads; the tradeoff is **replication lag** — a replica is always slightly behind the primary, so a write followed immediately by a read against the replica can appear to "not have happened yet." Apps that route reads to replicas usually need a strategy for read-your-own-writes consistency (e.g., route the read that must see a just-made write back to the primary).

[↑ back to top](#table-of-contents)

<a id="replication-advanced"></a>

### Advanced

**Physical vs. logical replication.** Streaming replication as described above is **physical replication** — it ships WAL and replays it byte-for-byte, so the replica is an exact copy of the entire cluster. **Logical replication** instead decodes WAL into a stream of row-level changes (inserts/updates/deletes) and applies them at the SQL level, which allows replicating just *some* tables, replicating between different major Postgres versions, and even replicating into a different schema shape. See the [logical replication docs](https://www.postgresql.org/docs/current/logical-replication.html).

```sql
-- On the source database
CREATE PUBLICATION my_pub FOR TABLE orders, customers;

-- On the destination database
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=source_host dbname=mydb user=replicator password=replpass'
  PUBLICATION my_pub;
```

**Why logical replication needs a replica identity.** To replicate an `UPDATE` or `DELETE`, the source has to tell the subscriber *which* row changed — but WAL for an `UPDATE`/`DELETE` doesn't include a full old row image by default, only enough to identify it. Postgres uses the table's **replica identity** to decide what that "enough" is:

- Default: the primary key (if one exists) — sufficient and cheapest.
- `REPLICA IDENTITY FULL`: the entire old row is logged — needed if the table has no primary key and no suitable unique index, at the cost of a much larger WAL footprint per update.

```sql
-- A table with no primary key needs this before UPDATE/DELETE can replicate
ALTER TABLE audit_log REPLICA IDENTITY FULL;
```

Skip this on a PK-less table and logical replication of updates/deletes silently fails to apply (or errors, depending on setup) — a well-documented [gotcha](https://www.postgresql.org/docs/current/logical-replication-restrictions.html) worth checking before assuming replication "isn't working."

| | Physical replication | Logical replication |
|---|---|---|
| Granularity | Entire cluster | Chosen tables/publications |
| Cross-version | No | Yes |
| Replica writable | No (read-only) | Yes (independent database) |
| Typical use | HA failover, read replicas | Selective sync, zero-downtime major upgrades, multi-region fan-out |

*Goes deeper:* cascading replication (replicas replicating to other replicas), synchronous replication quorums, and multi-master/BDR-style topologies are their own specialty — the [high availability docs](https://www.postgresql.org/docs/current/high-availability.html) index them if that's ever a dedicated topic.

[↑ back to top](#table-of-contents)

---

## Configuration Knobs Worth Knowing

<a id="config-beginner"></a>

### Beginner

**What it is.** Postgres reads settings from `postgresql.conf` at startup (some settings need a full restart; others reload with `SIGHUP`/`pg_ctl reload`). You can also inspect and change settings from SQL:

```sql
SHOW work_mem;
SHOW ALL;                          -- every current setting

ALTER SYSTEM SET work_mem = '16MB'; -- writes postgresql.auto.conf
SELECT pg_reload_conf();            -- applies reloadable settings immediately
```

See the [server configuration docs](https://www.postgresql.org/docs/current/runtime-config.html). This section is conceptual — *what each knob controls and why you'd touch it* — not a tuning cookbook; exact values depend heavily on your hardware and workload.

- **`shared_buffers`** — the chunk of memory Postgres uses to cache table/index pages in its own process space (separate from the OS's own filesystem cache). Default is a conservative 128MB; a dedicated database server typically wants this set explicitly rather than left at default. It can only be changed at server restart.
- **`work_mem`** — the memory budget for a *single* sort/hash operation within a query (e.g., an `ORDER BY` or hash join) before it spills to disk. Default is 4MB, deliberately small.
- **`max_connections`** — the hard ceiling on concurrent client connections (default 100). Each connection is a full OS backend process, not a lightweight thread, so this isn't a number to raise casually.

[↑ back to top](#table-of-contents)

<a id="config-working-knowledge"></a>

### Working Knowledge

**Why `work_mem` and `max_connections` interact dangerously.** `work_mem` is *per sort/hash operation, per connection* — a single complex query can use several multiples of `work_mem` at once (one per sort/hash node in its plan), and that multiplies again across every concurrent connection running similar queries. A `work_mem` that looks generously sized in isolation can push a busy server into swapping once `max_connections` concurrent sessions are all running non-trivial queries simultaneously. This is why raising `work_mem` "to make one slow report query faster" is a common way to make the whole server slower under load — [Stack Overflow's greatest hits](https://stackoverflow.com/questions/tagged/postgresql+work-mem) on OOM-killed Postgres processes trace back to exactly this.

**Wrong vs. right — reacting to a slow query:**

```sql
-- WRONG: bump work_mem globally to "fix" one report query
ALTER SYSTEM SET work_mem = '512MB';
```

```sql
-- RIGHT: scope it to the session/query that actually needs it
SET work_mem = '512MB';   -- this session only, reverts on disconnect
-- run the heavy report query
RESET work_mem;
```

**Checkpoint tuning (conceptually).** A **checkpoint** flushes all dirty (modified) buffers from memory to disk and marks a point WAL replay could start from after a crash. Checkpoints that happen too often waste I/O on redundant writes; checkpoints spaced too far apart mean a longer crash-recovery replay and burstier disk I/O. The knobs:

- **`checkpoint_timeout`** — max time between automatic checkpoints (default 5 minutes).
- **`max_wal_size`** — a soft cap on WAL volume between checkpoints; exceeding it triggers an early checkpoint. This is the setting the docs point to when `shared_buffers` is raised: more cached dirty pages means more to flush per checkpoint, so `max_wal_size` usually needs to grow alongside it.
- **`checkpoint_completion_target`** — spreads a checkpoint's writes over a fraction of the time until the next one, smoothing out I/O spikes instead of writing everything in a burst.

See [WAL configuration docs](https://www.postgresql.org/docs/current/runtime-config-wal.html).

[↑ back to top](#table-of-contents)

<a id="config-advanced"></a>

### Advanced

**`max_connections` is usually the wrong lever.** Raising it to "fix" connection-exhaustion errors treats the symptom: each backend process carries real memory overhead regardless of how idle it is, and Postgres's scheduler doesn't scale gracefully to thousands of active backends. The standard production fix is a **connection pooler** (e.g., PgBouncer) sitting between the application and Postgres, multiplexing many client connections onto a much smaller pool of actual server connections — `max_connections` on the database itself stays modest, and the pooler absorbs connection churn.

**Reading the practical effect of a knob instead of guessing.** Rather than tuning blind, pair every change with the monitoring queries in the next section — e.g., `pg_stat_database`'s cache hit ratio to judge whether `shared_buffers` is actually large enough, or `pg_stat_bgwriter`/checkpoint-related columns to see whether checkpoints are running on the timer or being forced early by `max_wal_size`.

*Goes deeper:* the storage engine's buffer replacement policy (clock-sweep), how the free space map and visibility map interact with `autovacuum`, and the internals of WAL record structure are their own specialty beyond conceptual server tuning — the [internals section of the docs](https://www.postgresql.org/docs/current/internals.html) is the jumping-off point.

[↑ back to top](#table-of-contents)

---

## Monitoring

<a id="monitoring-beginner"></a>

### Beginner

**What it is.** Postgres exposes its own internal state as queryable system views (`pg_stat_*`), so you observe the server with the same `SELECT` you'd use for anything else — no separate agent required for the basics. See the [statistics views docs](https://www.postgresql.org/docs/current/monitoring-stats.html).

**`pg_stat_activity`** — one row per active backend (connection): what query it's running, how long it's been running, what state it's in (`active`, `idle`, `idle in transaction`, ...).

```sql
SELECT pid, usename, state, query, now() - query_start AS running_for
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY running_for DESC;
```

**Real Scenario — find and stop a runaway query:**

```sql
-- 1. In one session, start something slow on purpose
SELECT pg_sleep(300);
```

```sql
-- 2. In another session, find it
SELECT pid, state, query, now() - query_start AS running_for
FROM pg_stat_activity
WHERE query ILIKE '%pg_sleep%';

-- 3. Cancel it gracefully first, terminate only if that doesn't work
SELECT pg_cancel_backend(<pid>);     -- like Ctrl+C for a query
SELECT pg_terminate_backend(<pid>);  -- drops the whole connection
```

`pg_cancel_backend` stops the current query but leaves the connection open; `pg_terminate_backend` is the blunter tool — reach for cancel first.

[↑ back to top](#table-of-contents)

<a id="monitoring-working-knowledge"></a>

### Working Knowledge

**`pg_stat_statements`** — Postgres's built-in query-performance extension. It aggregates *every* query by its normalized shape (constants stripped out) and tracks total time, call count, rows returned — the single most useful tool for "which query is actually costing us the most." It's not on by default and needs a server restart to enable, since it needs to be preloaded:

```
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
```

```sql
CREATE EXTENSION pg_stat_statements;

-- Top 5 queries by total time spent, across all calls
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
```

See the [`pg_stat_statements` docs](https://www.postgresql.org/docs/current/pgstatstatements.html). This is the tool that turns "the app feels slow" into "these three queries account for 80% of database time" — a `total_exec_time`-sorted list is usually where a performance investigation should start, not end.

[↑ back to top](#table-of-contents)

<a id="monitoring-advanced"></a>

### Advanced

**What to watch for, beyond single queries:**

- **Cache hit ratio** (`pg_stat_database`) — the fraction of block reads served from `shared_buffers` vs. disk. A ratio that's dropped from its usual high-90s% suggests either `shared_buffers` is undersized for the working set or something changed in the query workload.

  ```sql
  SELECT datname,
         blks_hit::float / NULLIF(blks_hit + blks_read, 0) AS cache_hit_ratio
  FROM pg_stat_database;
  ```

- **Replication lag** (`pg_stat_replication`, run on the primary) — how far behind each connected replica is. Growing lag on a replica used for reads means stale results; growing lag on a replica used for failover means a bigger potential data-loss window if the primary dies right now.

  ```sql
  SELECT client_addr, state, sent_lsn, replay_lsn,
         write_lag, flush_lag, replay_lag
  FROM pg_stat_replication;
  ```

- **Long-idle transactions** (`pg_stat_activity`, state `idle in transaction`) — a transaction that's open but not doing anything holds locks and prevents autovacuum from cleaning up dead rows it might still need to see. This is one of the most common causes of unexplained table bloat, and it's an application bug (a connection that opened a transaction and forgot to commit/rollback), not a database problem to tune around.
- **Autovacuum activity** (`pg_stat_user_tables`: `n_dead_tup`, `last_autovacuum`) — a table where dead tuples keep climbing and autovacuum rarely completes is heading toward both bloat and, eventually, transaction ID wraparound risk if left unchecked long enough.

*Goes deeper:* `pg_stat_statements`' planning-time columns, `auto_explain` for capturing actual plans of slow queries in production, and external time-series monitoring (Prometheus's `postgres_exporter`, pgAdmin's dashboard) build on these same views but are their own setup outside what ships with Postgres itself.

[↑ back to top](#table-of-contents)

---

## Cheat Sheet

**Common GRANT patterns:**

| Goal | Statement |
|---|---|
| Read-only role, one schema | `GRANT USAGE ON SCHEMA public TO r; GRANT SELECT ON ALL TABLES IN SCHEMA public TO r;` |
| Read access to future tables too | `ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO r;` |
| Read/write app role | `GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO r;` |
| Group membership (inherited grants) | `GRANT team_role TO alice;` |
| Row-level filtering | `ALTER TABLE t ENABLE ROW LEVEL SECURITY; CREATE POLICY p ON t USING (...);` |
| Revoke everything from public | `REVOKE ALL ON DATABASE db FROM PUBLIC;` |

**`pg_dump` / `pg_restore` flags:**

| Flag | Effect |
|---|---|
| `-F p` (default) | Plain SQL text — restore with `psql`, no selective restore |
| `-F c` | Custom format, compressed, selective restore via `pg_restore` |
| `-F d` | Directory format — supports parallel dump/restore |
| `-j N` | Parallel jobs (with `-F d` or `-F c`) |
| `-t table` / `--table=table` | Dump (or restore) only the named table |
| `-n schema` | Dump only the named schema |
| `--globals-only` (`pg_dumpall`) | Roles and tablespaces only, no table data |
| `-d dbname` (`pg_restore`) | Target database to restore into |
| `--clean` (`pg_restore`) | Drop existing objects before recreating them |

**Monitoring quick queries:**

| Question | View |
|---|---|
| What's running right now? | `pg_stat_activity` |
| Which queries cost the most overall? | `pg_stat_statements` |
| Is a replica falling behind? | `pg_stat_replication` (on the primary) |
| Is the cache big enough? | `pg_stat_database` (`blks_hit` vs `blks_read`) |
| Is a table bloating? | `pg_stat_user_tables` (`n_dead_tup`) |

[↑ back to top](#table-of-contents)

---

← Back to [Postgres SQL Mastery Guide](./postgres-mastery-guide.md)
