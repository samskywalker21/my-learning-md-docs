# MySQL Data Modeling & Types

Part of the [MySQL Mastery Guide](./mysql-mastery-guide.md). Full Beginner → Mastery depth.

## Table of Contents

- [Storage Engines: InnoDB vs. MyISAM vs. the Rest](#storage-engines-innodb-vs-myisam-vs-the-rest)
- [Constraints](#constraints)
- [JSON](#json)
- [Generated Columns](#generated-columns-stored-vs-virtual)
- [ENUM and SET: Convenient, Then a Trap](#enum-and-set-convenient-then-a-trap)
- [Temporal Types Revisited](#temporal-types-revisited)
- [Full-Text Search](#full-text-search)
- [Cheat Sheet](#cheat-sheet)

---

## Storage Engines: InnoDB vs. MyISAM vs. the Rest

**Working Knowledge.** [Foundations](./mysql-foundations.md#storage-engines-pick-one-before-you-do-anything-else) introduced the concept; here's the actual comparison.

| Engine | Transactions | Row/table locking | Foreign keys | Crash recovery | When to use it |
|---|---|---|---|---|---|
| **InnoDB** | Yes | Row-level | Yes | Yes (redo/undo logs) | Default choice, essentially always |
| **MyISAM** | No | Table-level | No | No (can corrupt on crash) | Legacy only; occasionally for read-only, full-text-heavy workloads pre-8.0 |
| **Memory** | No | Table-level | No | No (data lost on restart) | Session/cache tables, explicitly ephemeral data |
| **CSV** | No | — | No | No | Interchange with external tools, not application storage |

```sql
-- Wrong: choosing MyISAM today because an old tutorial claims it's
-- "faster for reads" — that advice predates InnoDB's buffer pool and
-- MVCC maturity by over a decade, and MyISAM's table-level locking makes
-- it far worse under any concurrent write load.
CREATE TABLE product_views (product_id BIGINT, viewed_at DATETIME) ENGINE=MyISAM;

-- Right: InnoDB unless you have a specific documented reason otherwise
CREATE TABLE product_views (product_id BIGINT, viewed_at DATETIME) ENGINE=InnoDB;
```

**Real Scenario (try it yourself):** Create the same table twice, once per engine, and open two `mysql` sessions. In session A, start a transaction and run a slow `UPDATE` against one row of the InnoDB table; in session B, try to `UPDATE` a *different* row — it proceeds immediately (row-level locking). Repeat with the MyISAM table — session B blocks until session A's statement completes, even though the rows don't overlap, because MyISAM locks the whole table for writes.

[Back to top](#mysql-data-modeling--types)

---

## Constraints

**Beginner → Working Knowledge.** Standard `PRIMARY KEY`, `UNIQUE`, `NOT NULL`, `CHECK` (enforced since 8.0.16 — before that, `CHECK` was silently parsed and ignored, a real and dangerous gotcha in anything pre-8.0), and `FOREIGN KEY` (InnoDB only — MyISAM accepts the syntax but never enforces it).

```sql
CREATE TABLE orders (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  customer_id BIGINT UNSIGNED NOT NULL,
  total DECIMAL(10,2) NOT NULL CHECK (total >= 0),
  FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT
) ENGINE=InnoDB;
```

```sql
-- Wrong: defining a CHECK constraint expecting it to be enforced,
-- without confirming the storage engine and version actually enforce it
CREATE TABLE orders (total DECIMAL(10,2) CHECK (total >= 0)) ENGINE=MyISAM;
-- MyISAM: parsed, never enforced, at any version.

-- Right: InnoDB + 8.0.16 or later, and verify
CREATE TABLE orders (total DECIMAL(10,2) CHECK (total >= 0)) ENGINE=InnoDB;
INSERT INTO orders (total) VALUES (-5);  -- should now error
```

[Back to top](#mysql-data-modeling--types)

---

## JSON

**Working Knowledge → Advanced.** `JSON` is a binary-stored (not plain text), validated-on-write type, comparable to Postgres's `JSONB`. Like `JSONB`, a raw `JSON` column can't be indexed directly — you index a [generated column](#generated-columns-stored-vs-virtual) extracted from it, or (8.0.21+) a functional index using `JSON_VALUE()`.

```sql
CREATE TABLE events (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  payload JSON NOT NULL
);

INSERT INTO events (payload) VALUES ('{"type": "click", "user_id": 42}');

SELECT payload->>'$.type' AS event_type FROM events;   -- ->> unquotes the result
SELECT payload->'$.user_id' FROM events;                -- -> keeps JSON quoting
```

```sql
-- Wrong: filtering directly on an unindexed JSON path — forces a full
-- scan plus a JSON parse of every row.
SELECT * FROM events WHERE payload->>'$.type' = 'click';

-- Right: generated column + index for a hot filter path
ALTER TABLE events
  ADD COLUMN event_type VARCHAR(50)
    GENERATED ALWAYS AS (payload->>'$.type') STORED,
  ADD INDEX idx_events_type (event_type);

SELECT * FROM events WHERE event_type = 'click';  -- now indexable
```

**Advanced.** `JSON_TABLE()` (8.0.4+) shreds a JSON array into relational rows for use in a `FROM` clause — genuinely useful when you need to `JOIN` against values buried in a JSON array without writing an app-side loop:

```sql
SELECT jt.tag
FROM events, JSON_TABLE(payload, '$.tags[*]' COLUMNS (tag VARCHAR(50) PATH '$')) AS jt
WHERE events.id = 1;
```

See the official [JSON Data Type](https://dev.mysql.com/doc/refman/8.4/en/json.html) and [JSON Table Functions](https://dev.mysql.com/doc/refman/8.4/en/json-table-functions.html) references.

[Back to top](#mysql-data-modeling--types)

---

## Generated Columns: Stored vs. Virtual

**Working Knowledge.** Both `STORED` (computed and written to disk on every insert/update) and `VIRTUAL` (computed on read, not stored) have existed since 5.7 — unlike Postgres 18, MySQL has never defaulted to `VIRTUAL` when the keyword is omitted; **`VIRTUAL` is MySQL's default** if you don't specify either.

```sql
ALTER TABLE customers
  ADD COLUMN email_domain VARCHAR(255)
    GENERATED ALWAYS AS (SUBSTRING_INDEX(email, '@', -1)) VIRTUAL;
```

Use `STORED` when you need to index the generated column *and* avoid recomputing it on every read of a heavy expression, or when the column feeds a `FULLTEXT`/`SPATIAL` index (those require `STORED`, not `VIRTUAL`, in InnoDB). Otherwise `VIRTUAL` avoids the storage and write-amplification cost.

[Back to top](#mysql-data-modeling--types)

---

## ENUM and SET: Convenient, Then a Trap

**Advanced.** `ENUM` stores a fixed list of string values internally as small integers — compact and fast to compare, but the value list is baked into the column definition itself.

```sql
CREATE TABLE orders (status ENUM('pending', 'shipped', 'cancelled') NOT NULL);
```

```sql
-- Wrong: adding a new enum value with a plain ALTER TABLE on a large,
-- busy table — historically this could mean a full table rewrite
-- (ALGORITHM=COPY) unless you're appending to the end of the list AND
-- not changing the storage size class (1 vs 2 bytes).
ALTER TABLE orders MODIFY status ENUM('pending', 'shipped', 'cancelled', 'refunded');

-- Right: append-only changes that fit within the same storage size
-- (≤255 values stays 1 byte) can use ALGORITHM=INSTANT in recent 8.0/8.4 —
-- but always test with EXPLAIN/pt-online-schema-change on a large table
-- rather than assuming, since exact eligibility depends on your version
-- and the specific change. When in doubt, prefer a lookup table + FK
-- for values that will actually change over the table's lifetime.
ALTER TABLE orders MODIFY status ENUM('pending', 'shipped', 'cancelled', 'refunded'), ALGORITHM=INSTANT;
```

`SET` is the same idea for a column that can hold *multiple* values from a fixed list, bitmask-encoded — rarely the right modeling choice; a join table is almost always clearer and more flexible. Both types share the same "value list is schema, not data" problem: reordering or removing values silently changes what existing stored integers mean.

[Back to top](#mysql-data-modeling--types)

---

## Temporal Types Revisited

**Working Knowledge.** [Foundations](./mysql-foundations.md#core-data-types) covered `DATETIME` vs `TIMESTAMP`; the piece worth adding here is fractional seconds and automatic update behavior:

```sql
CREATE TABLE audit_log (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)
    ON UPDATE CURRENT_TIMESTAMP(3)
);
```

`DATETIME(3)` gives millisecond precision (up to `(6)` for microseconds) — plain `DATETIME` truncates to whole seconds, which is a common source of "why do these two nearly-simultaneous events look identical" confusion in logs. `ON UPDATE CURRENT_TIMESTAMP` is MySQL-specific sugar with no Postgres equivalent (Postgres needs a trigger for the same effect).

[Back to top](#mysql-data-modeling--types)

---

## Full-Text Search

**Advanced.** A `FULLTEXT` index works on InnoDB (since 5.6) or MyISAM, over `CHAR`/`VARCHAR`/`TEXT` columns:

```sql
ALTER TABLE articles ADD FULLTEXT INDEX idx_articles_fts (title, body);

-- Natural language mode: ranked relevance search
SELECT title, MATCH(title, body) AGAINST ('database indexing' IN NATURAL LANGUAGE MODE) AS score
FROM articles
WHERE MATCH(title, body) AGAINST ('database indexing' IN NATURAL LANGUAGE MODE)
ORDER BY score DESC;

-- Boolean mode: explicit required/excluded/wildcard terms
SELECT title FROM articles
WHERE MATCH(title, body) AGAINST ('+database -mongodb*' IN BOOLEAN MODE);
```

```sql
-- Wrong: LIKE '%term%' for "search" functionality — no index can serve
-- a leading wildcard, so this is a full table scan on every query.
SELECT * FROM articles WHERE body LIKE '%indexing%';

-- Right: FULLTEXT + MATCH ... AGAINST
SELECT * FROM articles WHERE MATCH(body) AGAINST ('indexing' IN NATURAL LANGUAGE MODE);
```

**Gotcha:** InnoDB full-text boolean mode doesn't support the `@` symbol the way MyISAM does (it's reserved for the `@distance` proximity operator), and InnoDB search doesn't apply MyISAM's old 50%-relevance-threshold exclusion. If you're reading MyISAM-era full-text advice, some of it doesn't transfer directly. See the official [Full-Text Search Functions](https://dev.mysql.com/doc/refman/8.4/en/fulltext-search.html) reference. For anything beyond basic relevance search (fuzzy matching, faceting, language-aware stemming across many languages), most production systems reach for Elasticsearch/OpenSearch/Meilisearch rather than pushing MySQL `FULLTEXT` further — know where the ceiling is.

[Back to top](#mysql-data-modeling--types)

---

## Cheat Sheet

| Need | Reach for |
|---|---|
| Default table engine | `ENGINE=InnoDB` — always, unless proven otherwise |
| Enforced row-level constraint | `CHECK (...)` — InnoDB + 8.0.16 or later only |
| Indexable JSON field | Generated column (`STORED`) extracted from the `JSON` column, then index that |
| Shred a JSON array into rows | `JSON_TABLE(col, '$.path[*]' COLUMNS (...))` |
| Fixed small value set, rarely changing | `ENUM` (compact) — but prefer a lookup table if values will evolve |
| Millisecond-precision timestamps | `DATETIME(3)` / `DATETIME(6)` |
| Auto-updating "last modified" column | `... ON UPDATE CURRENT_TIMESTAMP` |
| Full-text search | `FULLTEXT` index + `MATCH() AGAINST()`, never `LIKE '%term%'` |

[Back to top](#mysql-data-modeling--types)
