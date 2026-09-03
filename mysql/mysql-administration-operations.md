# MySQL Administration & Operations

Part of the [MySQL Mastery Guide](./mysql-mastery-guide.md). Beginner → Advanced — Mastery-level ops internals (custom replication topologies, deep buffer-pool/redo-log internals) are their own specialty and out of scope for a SQL-focused series; Mastery-level asides are folded into brief "goes deeper" notes.

## Table of Contents

- [Users, Privileges, and Roles](#users-privileges-and-roles)
- [Backup and Restore](#backup-and-restore)
- [Replication Basics](#replication-basics)
- [Configuration Knobs Worth Knowing](#configuration-knobs-worth-knowing)
- [Monitoring](#monitoring)
- [Cheat Sheet](#cheat-sheet)

---

## Users, Privileges, and Roles

**Working Knowledge.**

```sql
CREATE USER 'app'@'10.0.0.%' IDENTIFIED BY 'a-strong-password';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'app'@'10.0.0.%';
FLUSH PRIVILEGES;   -- rarely needed today for GRANT/REVOKE/CREATE USER (they take effect immediately);
                     -- still relevant after directly editing grant tables, which you shouldn't do anyway
```

MySQL users are identified by `'user'@'host'` — the host part is a real access-control dimension with no Postgres equivalent (Postgres relies on `pg_hba.conf` for host-based rules, separate from the role itself). `'app'@'10.0.0.%'` and `'app'@'%'` are different users that happen to share a name.

```sql
-- Wrong: granting from anywhere when the app only ever connects from
-- one subnet — needlessly widens the attack surface
CREATE USER 'app'@'%' IDENTIFIED BY 'a-strong-password';

-- Right: scope the host as tightly as the deployment allows
CREATE USER 'app'@'10.0.0.%' IDENTIFIED BY 'a-strong-password';
```

**Advanced.** Roles (8.0+) let you define a named privilege bundle once and grant/revoke it as a unit, instead of repeating the same `GRANT` list per user:

```sql
CREATE ROLE 'readonly_analyst';
GRANT SELECT ON myapp.* TO 'readonly_analyst';

CREATE USER 'jane'@'%' IDENTIFIED BY 'x';
GRANT 'readonly_analyst' TO 'jane'@'%';
SET DEFAULT ROLE 'readonly_analyst' TO 'jane'@'%';   -- so it's active on login without an extra SET ROLE
```

See the official [Using Roles](https://dev.mysql.com/doc/refman/8.4/en/roles.html) and [GRANT Statement](https://dev.mysql.com/doc/refman/8.4/en/grant.html) references.

[Back to top](#mysql-administration--operations)

---

## Backup and Restore

**Working Knowledge → Advanced.**

| Method | Type | Notes |
|---|---|---|
| `mysqldump` | Logical | Simple, portable, human-readable SQL; slow to restore on large databases, locks tables briefly (or uses `--single-transaction` for InnoDB to avoid locking) |
| `mysqlpump` | Logical | Parallel, faster dump than `mysqldump`; less universally used |
| Percona XtraBackup / MySQL Enterprise Backup | Physical | Hot, non-blocking copy of the actual data files — the production-grade choice for large databases needing fast restore and minimal downtime |
| `mysqlbinlog` + binary logs | Point-in-time | Replays binlog events since the last full backup — needed for true point-in-time recovery |

```bash
# Logical backup, consistent snapshot without locking InnoDB tables
mysqldump --single-transaction --routines --triggers myapp > myapp.sql

# Restore
mysql myapp < myapp.sql
```

```bash
# Wrong: mysqldump without --single-transaction on a live InnoDB
# database — without it, mysqldump takes table locks (or gets an
# inconsistent snapshot across tables), hurting concurrent writers.
mysqldump myapp > myapp.sql

# Right
mysqldump --single-transaction myapp > myapp.sql
```

**Point-in-time recovery** — restore the last full backup, then replay binary log events from just after that backup up to (but not including) the mistake:

```bash
mysqlbinlog --start-datetime="2026-09-01 00:00:00" --stop-datetime="2026-09-01 14:32:00" \
  binlog.000123 | mysql myapp
```

This only works if `log_bin` was enabled *before* the backup was taken — a database running with binary logging off has no path to point-in-time recovery, only whatever your last full backup captured.

[Back to top](#mysql-administration--operations)

---

## Replication Basics

**Working Knowledge → Advanced.** MySQL replication is fundamentally different from Postgres's WAL-shipping model: it's driven by the **binary log** (`binlog`), a record of data-changing events, which replicas apply.

```
Source                              Replica
┌───────────────┐  binlog events   ┌───────────────┐
│ writes to      │ ───────────────► │ I/O thread     │
│ binary log     │                  │ writes to      │
└───────────────┘                  │ relay log      │
                                    └───────┬───────┘
                                            │
                                    ┌───────▼───────┐
                                    │ SQL thread     │
                                    │ applies events │
                                    │ to data        │
                                    └───────────────┘
```

Binlog format is a real operational choice:

- **`ROW`** (the default) — logs the actual row changes, not the statement. Deterministic, safe for any statement including ones with nondeterministic functions (`UUID()`, `NOW()` used implicitly, etc.) — the modern default and the right choice almost always.
- **`STATEMENT`** — logs the SQL text itself; smaller logs, but breaks for nondeterministic statements (replica could compute a different result than the source).
- **`MIXED`** — statement-based by default, auto-switches to row-based when MySQL detects a statement that isn't safely replicated as a statement.

**GTIDs** (global transaction identifiers, on by default in modern setups) replace the old file-name+offset coordinate system for tracking replication position — every transaction gets a globally unique ID, which makes failover and multi-source replication dramatically simpler than manually tracking binlog file/position pairs.

```sql
-- Check replication status on a replica
SHOW REPLICA STATUS\G   -- SHOW SLAVE STATUS is the older, pre-8.0.22 name
```

Watch `Seconds_Behind_Source` (replication lag) and `Last_IO_Error`/`Last_SQL_Error` for the two most common health checks. See the official [Replication Formats](https://dev.mysql.com/doc/refman/8.4/en/replication-formats.html) and [Restrictions on Replication with GTIDs](https://dev.mysql.com/doc/refman/8.4/en/replication-gtids-restrictions.html) references — the GTID restrictions page is worth reading before combining GTIDs with `STATEMENT`-format temporary tables, since that combination has real, documented limitations.

*Goes deeper (Mastery):* multi-source replication, Group Replication / InnoDB Cluster for automated failover, and semi-synchronous replication tuning are each their own topic beyond this file's scope.

[Back to top](#mysql-administration--operations)

---

## Configuration Knobs Worth Knowing

**Advanced.** Conceptual tuning, not exhaustive — always benchmark against your own workload rather than copying numbers.

| Variable | What it controls | Starting point |
|---|---|---|
| `innodb_buffer_pool_size` | InnoDB's main cache — table/index data held in memory | ~70-80% of RAM on a dedicated DB server; 50-60% on a shared box, leaving headroom for the OS and connection overhead |
| `innodb_log_file_size` (or `innodb_redo_log_capacity` in 8.0.30+) | Redo log size — larger reduces checkpoint I/O churn on write-heavy workloads, at the cost of longer crash recovery | Workload-dependent; too small causes frequent checkpoint stalls under sustained writes |
| `max_connections` | Hard cap on concurrent connections | Size to your connection pool's actual max, not a guess — each connection has real memory overhead |
| `innodb_flush_log_at_trx_commit` | Durability vs. write throughput trade-off | `1` (full ACID durability, fsync every commit) for anything you can't afford to lose; `2` trades a small durability window for throughput |

```sql
SELECT @@innodb_buffer_pool_size / 1024 / 1024 / 1024 AS buffer_pool_gb;
SHOW VARIABLES LIKE 'max_connections';
```

```sql
-- Wrong: innodb_flush_log_at_trx_commit=2 (or 0) on a financial
-- ledger table because "it's faster" — a crash can lose up to ~1s of
-- committed transactions, which is unacceptable for that data.
SET GLOBAL innodb_flush_log_at_trx_commit = 2;

-- Right: keep durability=1 for data you can't afford to lose; only
-- relax it for genuinely disposable, high-throughput data (e.g. raw
-- event ingestion you can re-derive or tolerate losing seconds of)
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
```

[Back to top](#mysql-administration--operations)

---

## Monitoring

**Working Knowledge → Advanced.**

```sql
-- Buffer pool hit rate — should stay well above 99% for a healthy cache
SELECT
  (1 - Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests) * 100 AS hit_rate_pct
FROM (
  SELECT
    (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Innodb_buffer_pool_reads') AS Innodb_buffer_pool_reads,
    (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Innodb_buffer_pool_read_requests') AS Innodb_buffer_pool_read_requests
) t;

-- Currently running queries and their state
SELECT * FROM performance_schema.processlist WHERE command != 'Sleep';

-- Last deadlock, current lock waits
SHOW ENGINE INNODB STATUS\G
SELECT * FROM performance_schema.data_lock_waits;
```

`performance_schema` (on by default since 5.6.6, with low overhead in modern versions) is MySQL's equivalent of Postgres's `pg_stat_*` views — instrumented, queryable operational data. The `sys` schema (bundled by default in modern installs) wraps it in more human-readable views, e.g. `sys.schema_table_statistics`, `sys.processlist`.

*Goes deeper (Mastery):* the Performance Schema's instrument-level configuration (enabling/disabling specific instruments to control overhead, wait-event histograms) is its own deep area worth a dedicated pass once you have a specific bottleneck to chase.

[Back to top](#mysql-administration--operations)

---

## Cheat Sheet

| Task | Command |
|---|---|
| Create a scoped user | `CREATE USER 'u'@'host' IDENTIFIED BY '...';` |
| Bundle privileges | `CREATE ROLE 'r'; GRANT ... TO 'r'; GRANT 'r' TO 'u'@'host';` |
| Consistent logical backup | `mysqldump --single-transaction db > db.sql` |
| Point-in-time replay | `mysqlbinlog --start-datetime=... --stop-datetime=... binlog.NNN \| mysql db` |
| Replica health | `SHOW REPLICA STATUS\G` — check `Seconds_Behind_Source`, `Last_*_Error` |
| Buffer pool size | `SHOW VARIABLES LIKE 'innodb_buffer_pool_size';` |
| Running queries | `SELECT * FROM performance_schema.processlist WHERE command != 'Sleep';` |
| Last deadlock / current locks | `SHOW ENGINE INNODB STATUS\G` |

[Back to top](#mysql-administration--operations)
