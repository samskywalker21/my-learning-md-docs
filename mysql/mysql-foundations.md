# MySQL Foundations

Part of the [MySQL Mastery Guide](./mysql-mastery-guide.md). Beginner → Working Knowledge only — this is the scaffolding the other files assume: connecting, storage engines at a glance, DDL, core types, and CRUD.

## Table of Contents

- [Connecting with the `mysql` Client](#connecting-with-the-mysql-client)
- [Storage Engines: Pick One Before You Do Anything Else](#storage-engines-pick-one-before-you-do-anything-else)
- [Creating Tables: `CREATE TABLE`](#creating-tables-create-table)
- [Altering Tables: `ALTER TABLE`](#altering-tables-alter-table)
- [Core Data Types](#core-data-types)
- [SQL_MODE: MySQL's Strictness Dial](#sql_mode-mysqls-strictness-dial)
- [CRUD Fundamentals](#crud-fundamentals)
- [Cheat Sheet](#cheat-sheet)

---

## Connecting with the `mysql` Client

**Beginner.** The `mysql` CLI is analogous to `psql`, but its meta-commands don't use a backslash prefix the same way for everything — most administrative work is just SQL (`SHOW DATABASES;`, `SHOW TABLES;`), with a handful of client-only shortcuts (`\G`, `\c`, `source file.sql`).

```bash
mysql -u root -p -h 127.0.0.1 -P 3306 mydatabase
```

```sql
SHOW DATABASES;
USE mydatabase;
SHOW TABLES;
SELECT * FROM orders LIMIT 5\G   -- \G prints one column per line, useful for wide rows
```

> **Gotcha:** `mysql -u root -p` with a space before the password prompt is normal, but `-ppassword` (no space) works too and will leak your password in `ps`/shell history. Always let it prompt, or use `--password` with a config file (`~/.my.cnf`) for scripts. See [Stack Overflow: MySQL warning "Using a password on the command line interface can be insecure"](https://stackoverflow.com/questions/8751050/mysql-warning-using-a-password-on-the-command-line-interface-can-be-insecure) for the accepted alternatives.

[Back to top](#mysql-foundations)

---

## Storage Engines: Pick One Before You Do Anything Else

**Working Knowledge.** This is the single biggest conceptual thing Postgres users have to unlearn: in MySQL, a table isn't just "a table" — it's backed by a pluggable **storage engine**, chosen (or defaulted) at `CREATE TABLE` time, and different tables in the same database can use different engines.

```sql
CREATE TABLE orders (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  total DECIMAL(10,2) NOT NULL
) ENGINE=InnoDB;
```

**InnoDB is the default and the only engine you should reach for by default** — it's the one with transactions, foreign keys, row-level locking, and crash recovery. Full engine comparison lives in [Data Modeling & Types §Storage Engines](./mysql-data-modeling-types.md#storage-engines-innodb-vs-myisam-vs-the-rest); this file just needs you to know `ENGINE=InnoDB` exists and why you write it (or rely on the default) on every `CREATE TABLE`.

```sql
-- Wrong: relying on an old dump or tutorial that still specifies MyISAM
CREATE TABLE sessions (id INT PRIMARY KEY, data TEXT) ENGINE=MyISAM;
-- No transactions, table-level locking under write load, no crash-safe recovery.

-- Right: InnoDB unless you have a specific, documented reason not to
CREATE TABLE sessions (id INT PRIMARY KEY, data TEXT) ENGINE=InnoDB;
```

Confirm the default and check what any given table is actually using:

```sql
SELECT @@default_storage_engine;
SELECT table_name, engine FROM information_schema.tables WHERE table_schema = 'mydatabase';
```

[Back to top](#mysql-foundations)

---

## Creating Tables: `CREATE TABLE`

**Beginner → Working Knowledge.**

```sql
CREATE TABLE customers (
  id           BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  email        VARCHAR(255) NOT NULL,
  created_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uq_customers_email (email)
) ENGINE=InnoDB;
```

Two things every MySQL table should have, and why:

- **A primary key, always.** InnoDB physically clusters row data around the primary key (see [Query Engine & Indexing §Clustered vs Secondary Indexes](./mysql-query-engine-indexing.md#clustered-vs-secondary-indexes-innodbs-biggest-departure-from-postgres)). A table with no primary key and no suitable `UNIQUE NOT NULL` index gets an invisible, unusable synthetic row ID (`GEN_CLUST_INDEX`) — you lose the ability to reference rows efficiently and replication/tooling gets worse.
- **`AUTO_INCREMENT` for surrogate keys**, MySQL's equivalent of Postgres's identity columns — no separate sequence object, it's a table-level counter.

```sql
-- Wrong: no primary key at all
CREATE TABLE events (payload JSON, created_at DATETIME) ENGINE=InnoDB;

-- Right
CREATE TABLE events (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  payload JSON,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

**Real Scenario (try it yourself):** Create both versions of a `session_data` table — one with `id INT PRIMARY KEY AUTO_INCREMENT`, one with no primary key — insert a few thousand rows into each with a loop, then run `EXPLAIN SELECT * FROM session_data WHERE id = 500` against both. The no-PK table has no usable index for that lookup at all, since `id` isn't even a real column there; you'd have to add one.

[Back to top](#mysql-foundations)

---

## Altering Tables: `ALTER TABLE`

**Working Knowledge.** MySQL's `ALTER TABLE` historically rewrote the whole table for most operations; since 5.6+ (and refined through 8.0/8.4) many common alterations run as **online DDL** — in place, without a full table copy, with configurable locking.

```sql
ALTER TABLE customers ADD COLUMN phone VARCHAR(20) NULL, ALGORITHM=INSTANT;
```

- `ALGORITHM=INSTANT` — metadata-only change, no table rebuild, no long lock (adding/dropping columns in many cases, since 8.0.12).
- `ALGORITHM=INPLACE` — rebuilds the table but allows concurrent DML for most operations.
- `ALGORITHM=COPY` — the old, expensive path: full table copy, blocks writes for the duration. MySQL will fall back to this if your desired algorithm isn't supported for that specific change — always specify the algorithm explicitly on a large production table so you get an error instead of an unplanned full-table lock.

```sql
-- Wrong on a large, hot table: no ALGORITHM specified, MySQL may silently pick COPY
ALTER TABLE orders ADD COLUMN discount_code VARCHAR(20);

-- Right: force INSTANT (or INPLACE) and fail loudly if that's not possible
ALTER TABLE orders ADD COLUMN discount_code VARCHAR(20), ALGORITHM=INSTANT, LOCK=NONE;
```

See the official [ALTER TABLE and Generated Columns / Online DDL reference](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html) for exactly which operations support which algorithm — it's a table worth bookmarking, not memorizing.

[Back to top](#mysql-foundations)

---

## Core Data Types

**Beginner.**

| Category | Types | Notes |
|---|---|---|
| Integer | `TINYINT`, `SMALLINT`, `INT`, `BIGINT` | Add `UNSIGNED` for non-negative counters/IDs to double the positive range |
| Fixed decimal | `DECIMAL(p,s)` | Exact — use for money, never `FLOAT`/`DOUBLE` |
| Floating point | `FLOAT`, `DOUBLE` | Approximate — fine for scientific data, wrong for currency |
| String | `CHAR(n)`, `VARCHAR(n)`, `TEXT` | Unlike Postgres, `CHAR`/`VARCHAR` have a hard length cap tied to row size; see below |
| Date/time | `DATE`, `DATETIME`, `TIMESTAMP`, `TIME` | `DATETIME` vs `TIMESTAMP` is a real gotcha — see next |
| Boolean | `BOOLEAN` | Syntactic sugar for `TINYINT(1)` — MySQL has no real boolean type |
| JSON | `JSON` | Binary-stored, validated on write; see [Data Modeling & Types](./mysql-data-modeling-types.md#json) |

**`DATETIME` vs `TIMESTAMP` — the gotcha that bites everyone once:**

```sql
-- TIMESTAMP is stored as UTC internally and converted to the connection's
-- time zone on read/write. Change @@session.time_zone and the same row
-- appears to have a different value.
CREATE TABLE events (happened_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP);

-- DATETIME stores exactly what you put in, no time zone conversion, ever.
CREATE TABLE events (happened_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP);
```

`TIMESTAMP` also has a range limited to 1970–2038 (the classic Year 2038 problem), because it's stored as a 4-byte Unix-epoch-like value; `DATETIME` has no such ceiling. Default to `DATETIME` unless you specifically want automatic UTC storage with per-connection display conversion. See the official [Date and Time Type Overview](https://dev.mysql.com/doc/refman/8.4/en/date-and-time-type-overview.html).

[Back to top](#mysql-foundations)

---

## SQL_MODE: MySQL's Strictness Dial

**Working Knowledge.** `sql_mode` is a session/global setting with no real Postgres equivalent — it controls how forgiving MySQL is about invalid data, ambiguous `GROUP BY`, and nonstandard syntax. The [MySQL 8.4 default](https://dev.mysql.com/doc/refman/8.4/en/sql-mode.html) includes `ONLY_FULL_GROUP_BY`, `STRICT_TRANS_TABLES`, `NO_ZERO_IN_DATE`, `NO_ZERO_DATE`, `ERROR_FOR_DIVISION_BY_ZERO`, and `NO_ENGINE_SUBSTITUTION`.

```sql
SELECT @@GLOBAL.sql_mode;
SELECT @@SESSION.sql_mode;
```

`STRICT_TRANS_TABLES` matters most day-to-day: without it (pre-5.7 default), inserting a too-long string silently truncated it; with it, you get an error instead. If you ever see old blog advice about MySQL "silently truncating" or "silently converting invalid dates to zero-dates," that's describing pre-strict-mode behavior — current defaults reject it.

```sql
-- With strict mode (default today): this errors, as it should
INSERT INTO customers (email) VALUES (REPEAT('a', 300));
-- ERROR 1406 (22001): Data too long for column 'email' at row 1
```

`ONLY_FULL_GROUP_BY` is covered in depth in [Core Querying §GROUP BY and ONLY_FULL_GROUP_BY](./mysql-core-querying.md#group-by-and-only_full_group_by) since it changes how you're allowed to write aggregate queries.

[Back to top](#mysql-foundations)

---

## CRUD Fundamentals

**Beginner.**

```sql
INSERT INTO customers (email) VALUES ('a@example.com'), ('b@example.com');

SELECT id, email FROM customers WHERE email LIKE '%@example.com';

UPDATE customers SET email = 'new@example.com' WHERE id = 1;

DELETE FROM customers WHERE id = 2;
```

MySQL-specific extensions worth knowing early:

```sql
-- INSERT ... ON DUPLICATE KEY UPDATE: MySQL's upsert, no Postgres-style
-- ON CONFLICT clause exists — this is the idiomatic equivalent.
INSERT INTO customers (id, email) VALUES (1, 'a@example.com')
ON DUPLICATE KEY UPDATE email = VALUES(email);

-- REPLACE INTO: deletes then re-inserts on a key conflict — resets
-- AUTO_INCREMENT-independent columns and fires DELETE triggers. Prefer
-- ON DUPLICATE KEY UPDATE unless you specifically want replace semantics.
```

```sql
-- Wrong: REPLACE INTO when you only meant to update one column —
-- it silently resets every other column to its default, and any
-- foreign-key-referencing child rows tied to the old row identity break.
REPLACE INTO customers (id, email) VALUES (1, 'new@example.com');

-- Right: say what you mean
INSERT INTO customers (id, email) VALUES (1, 'new@example.com')
ON DUPLICATE KEY UPDATE email = VALUES(email);
```

[Back to top](#mysql-foundations)

---

## Cheat Sheet

| Task | Command |
|---|---|
| Connect | `mysql -u user -p -h host -P port db` |
| List databases / tables | `SHOW DATABASES;` / `SHOW TABLES;` |
| Describe a table | `DESCRIBE table_name;` or `SHOW CREATE TABLE table_name;` |
| Check storage engine | `SELECT table_name, engine FROM information_schema.tables WHERE table_schema='db';` |
| Instant column add | `ALTER TABLE t ADD COLUMN c INT, ALGORITHM=INSTANT;` |
| Upsert | `INSERT ... ON DUPLICATE KEY UPDATE col = VALUES(col);` |
| Check sql_mode | `SELECT @@SESSION.sql_mode;` |
| Wide-row friendly output | append `\G` instead of `;` |

[Back to top](#mysql-foundations)
