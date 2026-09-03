# Postgres Foundations

Part of the [Postgres SQL Mastery Guide](./postgres-mastery-guide.md).

This file covers the ground floor: connecting with `psql`, the core DDL you'll write in nearly every project (`CREATE TABLE`, `ALTER TABLE`, schemas), the core DML (`INSERT`/`UPDATE`/`DELETE`/`SELECT`), and a tour of the built-in scalar data types. Advanced types (JSONB, arrays, ranges, composite types) live in [Data Modeling & Advanced Types](./postgres-data-modeling-types.md) — this file only tells you they exist and points you there.

All examples assume PostgreSQL 18 (18.6, current stable as of Aug 2026); anywhere behavior changed from earlier versions, it's called out explicitly.

## Table of Contents

- [Connecting with psql](#connecting-with-psql)
  - [Beginner](#connecting-beginner)
  - [Working Knowledge](#connecting-working-knowledge)
- [Schemas & the Search Path](#schemas--the-search-path)
  - [Beginner](#schemas-beginner)
  - [Working Knowledge](#schemas-working-knowledge)
  - [Advanced](#schemas-advanced)
- [Creating Tables (CREATE TABLE)](#creating-tables-create-table)
  - [Beginner](#create-table-beginner)
  - [Working Knowledge](#create-table-working-knowledge)
- [Altering Tables (ALTER TABLE)](#altering-tables-alter-table)
  - [Beginner](#alter-table-beginner)
  - [Working Knowledge](#alter-table-working-knowledge)
  - [Advanced](#alter-table-advanced)
- [Core Data Types](#core-data-types)
  - [Beginner](#data-types-beginner)
  - [Working Knowledge](#data-types-working-knowledge)
- [Inserting Data (INSERT)](#inserting-data-insert)
  - [Beginner](#insert-beginner)
  - [Working Knowledge](#insert-working-knowledge)
- [Reading Data (SELECT basics)](#reading-data-select-basics)
  - [Beginner](#select-beginner)
  - [Working Knowledge](#select-working-knowledge)
- [Updating Data (UPDATE)](#updating-data-update)
  - [Beginner](#update-beginner)
  - [Working Knowledge](#update-working-knowledge)
- [Deleting Data (DELETE)](#deleting-data-delete)
  - [Beginner](#delete-beginner)
  - [Working Knowledge](#delete-working-knowledge)
- [Cheat Sheet](#cheat-sheet)

---

## Connecting with psql

<a id="connecting-beginner"></a>

### Beginner

**What it is.** `psql` is Postgres's official interactive command-line client. You type SQL, it sends that SQL to a running Postgres **server** process, and prints the results back.

```
  you type SQL/meta-commands
        │
        ▼
  psql (client)  ──libpq over TCP/socket──▶  postgres (server, port 5432)
        ▲                                          │
        └──────────── result rows ─────────────────┘
```

Connect to a local server:

```bash
psql -U myuser -d mydb -h localhost -p 5432
# or, if a matching OS role/db exists:
psql mydb
```

Once connected, your prompt becomes `mydb=>` (or `mydb=#` if you're a superuser). A `-` instead of `=` (`mydb->`) means psql is still waiting for more input — usually a missing semicolon.

**Try It:**

```sql
SELECT version();
\conninfo
```

`\conninfo` prints the database, user, host, port, and whether the connection is encrypted — the first thing to check when "it works on my machine" turns out to mean a different server entirely.

[↑ back to top](#table-of-contents)

<a id="connecting-working-knowledge"></a>

### Working Knowledge

**Why the backslash?** Commands like `\dt` or `\c` are **meta-commands** — instructions to *psql itself*, not SQL sent to the server. Postgres's SQL grammar has no room for a "list my tables" statement (that's a client convenience, not a relational operation), so psql reserves a separate `\`-prefixed syntax for things that operate on the client session or query the server's catalogs for you and format the result nicely. This is also why meta-commands don't end in a semicolon — they're not SQL statements, they're on-the-spot delegated to psql's own parser and end at the newline.

```sql
-- SQL: ends with ;
SELECT * FROM pg_tables WHERE schemaname = 'public';

-- meta-command equivalent: ends at newline, formatted for humans
\dt
```

**Wrong vs. right — losing your query to a stray quote:**

```sql
-- wrong: an unclosed quote leaves you stuck in a "mydb'>" continuation prompt
SELECT * FROM users WHERE name = 'Sam;

-- right: close the string, or type a single quote then Enter to close it,
-- or Ctrl+C to abort the buffer and start over
SELECT * FROM users WHERE name = 'Sam';
```

**Real Scenario:** Try this in psql — connect, then run a query and time it:

```sql
\timing on
SELECT count(*) FROM pg_class;
\x auto
SELECT * FROM pg_stat_activity LIMIT 1;
```

`\timing on` shows wall-clock time per statement — useful before you've learned `EXPLAIN`. `\x auto` switches to one-column-per-line ("expanded") display automatically whenever a row is too wide for your terminal, which is exactly what you want for wide catalog views like `pg_stat_activity`.

See the [official psql docs](https://www.postgresql.org/docs/current/app-psql.html) for the full meta-command list.

[↑ back to top](#table-of-contents)

---

## Schemas & the Search Path

<a id="schemas-beginner"></a>

### Beginner

**What it is.** A **database** contains **schemas**, and a schema contains tables (and other objects). Think of a schema as a namespace/folder inside the database — every fresh database starts with one schema, `public`. A fully-qualified table name is `schema.table`, e.g. `public.users`.

```sql
CREATE SCHEMA reporting;
CREATE TABLE reporting.monthly_totals (id integer, total numeric);
SELECT * FROM reporting.monthly_totals;
```

**Try It:**

```sql
\dn            -- list schemas in the current database
SELECT current_schema();
```

[↑ back to top](#table-of-contents)

<a id="schemas-working-knowledge"></a>

### Working Knowledge

**The search_path.** When you write `SELECT * FROM users` with no schema prefix, Postgres looks through the schemas listed in the `search_path` setting, in order, and uses the first match.

```sql
SHOW search_path;                 -- typically: "$user", public
SET search_path TO reporting, public;
```

`"$user"` means "a schema named after the current role, if one exists" — handy for giving each user their own private working schema without changing application code.

**Real Scenario:** Try creating two same-named tables in different schemas and watch the search path decide which one wins:

```sql
CREATE SCHEMA staging;
CREATE TABLE public.widgets (id int, note text default 'prod');
CREATE TABLE staging.widgets (id int, note text default 'staging');

SET search_path TO staging, public;
INSERT INTO widgets (id) VALUES (1);
SELECT * FROM widgets;            -- hits staging.widgets

SET search_path TO public;
SELECT * FROM widgets;            -- now hits public.widgets
```

[↑ back to top](#table-of-contents)

<a id="schemas-advanced"></a>

### Advanced

An unqualified name is resolved *at parse time using the current `search_path`*, not fixed to whichever schema existed when you wrote the query. A `SECURITY DEFINER` function or a shared script that relies on an unqualified name is trusting whoever calls it not to have manipulated `search_path` first — this is a known privilege-escalation vector, which is why the [official docs](https://www.postgresql.org/docs/current/ddl-schemas.html#DDL-SCHEMAS-PATH) recommend schema-qualifying object names inside functions and procedures, or setting a fixed `search_path` on the function itself (`SET search_path = pg_catalog, public`). For anything security-sensitive or reused across sessions, qualify the name — don't rely on the caller's path.

[↑ back to top](#table-of-contents)

---

## Creating Tables (CREATE TABLE)

<a id="create-table-beginner"></a>

### Beginner

```sql
CREATE TABLE books (
    id          integer PRIMARY KEY,
    title       text NOT NULL,
    price       numeric(8,2),
    published   date,
    in_stock    boolean DEFAULT true
);
```

Each line is a column: `name type [constraints]`. `PRIMARY KEY` uniquely identifies each row and implies `NOT NULL` + a unique index.

**Wrong vs. right — re-running a setup script:**

```sql
-- wrong: fails with "relation "books" already exists" if it's already there
CREATE TABLE books (id integer PRIMARY KEY);

-- right: safe to re-run, does nothing if it already exists
CREATE TABLE IF NOT EXISTS books (id integer PRIMARY KEY);
```

**Try It:**

```sql
CREATE TABLE IF NOT EXISTS books (
    id      integer PRIMARY KEY,
    title   text NOT NULL
);
\d books        -- inspect the table you just made
```

[↑ back to top](#table-of-contents)

<a id="create-table-working-knowledge"></a>

### Working Knowledge

**Auto-incrementing keys.** The classic pattern was `serial`, a shorthand that creates a sequence and wires it up as the column default. The [SQL-standard, docs-recommended](https://www.postgresql.org/docs/current/ddl-identity-columns.html) replacement is an **identity column**:

```sql
CREATE TABLE authors (
    id      integer GENERATED ALWAYS AS IDENTITY,
    name    text NOT NULL
);

INSERT INTO authors (name) VALUES ('Ada Lovelace');   -- id assigned automatically
```

`GENERATED ALWAYS AS IDENTITY` rejects an explicit `id` value unless the insert uses `OVERRIDING SYSTEM VALUE`; `GENERATED BY DEFAULT AS IDENTITY` lets a caller supply one (useful when migrating rows with existing IDs). Prefer identity columns over `serial` in new schemas — `serial` still works but has odd ownership/permission quirks around its underlying sequence that identity columns avoid.

**Foreign keys and constraints:**

```sql
CREATE TABLE books (
    id          integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    author_id   integer REFERENCES authors(id),
    title       text NOT NULL,
    price       numeric(8,2) CHECK (price >= 0)
);
```

`REFERENCES authors(id)` enforces that every `author_id` in `books` actually exists in `authors` — Postgres rejects the insert/update otherwise. `CHECK` enforces an arbitrary boolean condition per row.

**Real Scenario:** Build a two-table relationship and prove the foreign key is enforced:

```sql
CREATE TABLE authors (id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY, name text);
CREATE TABLE books (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    author_id integer REFERENCES authors(id),
    title text
);

INSERT INTO authors (name) VALUES ('Ada Lovelace');
INSERT INTO books (author_id, title) VALUES (1, 'Notes on the Analytical Engine'); -- works
INSERT INTO books (author_id, title) VALUES (999, 'Ghost Book');                   -- fails: no such author
```

See [`CREATE TABLE`](https://www.postgresql.org/docs/current/sql-createtable.html) for the full grammar.

[↑ back to top](#table-of-contents)

---

## Altering Tables (ALTER TABLE)

<a id="alter-table-beginner"></a>

### Beginner

```sql
ALTER TABLE books ADD COLUMN pages integer;
ALTER TABLE books RENAME COLUMN pages TO page_count;
ALTER TABLE books DROP COLUMN page_count;
```

**Try It:**

```sql
ALTER TABLE books ADD COLUMN IF NOT EXISTS notes text;
\d books
ALTER TABLE books DROP COLUMN IF EXISTS notes;
```

`IF NOT EXISTS` / `IF EXISTS` on `ALTER TABLE ... ADD/DROP COLUMN` make the statement idempotent, the same way `IF NOT EXISTS` does on `CREATE TABLE`.

[↑ back to top](#table-of-contents)

<a id="alter-table-working-knowledge"></a>

### Working Knowledge

**Adding a NOT NULL column with a default — order matters:**

```sql
-- wrong instinct on old habits: forgetting NOT NULL leaves existing rows unconstrained
ALTER TABLE books ADD COLUMN currency text DEFAULT 'USD';

-- right, when the column must never be null:
ALTER TABLE books ADD COLUMN currency text NOT NULL DEFAULT 'USD';
```

**Changing a column's type:**

```sql
ALTER TABLE books ALTER COLUMN price TYPE numeric(10,2);
```

If the conversion isn't implicit (e.g. `text` to `integer`), add `USING`:

```sql
ALTER TABLE books ALTER COLUMN page_count TYPE integer USING page_count::integer;
```

**Real Scenario:** Add a required column to a table that already has rows, and watch Postgres complain until you supply a default:

```sql
CREATE TABLE books (id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY, title text);
INSERT INTO books (title) VALUES ('Existing Book');

ALTER TABLE books ADD COLUMN currency text NOT NULL; -- fails: no default for existing row
ALTER TABLE books ADD COLUMN currency text NOT NULL DEFAULT 'USD'; -- works
```

[↑ back to top](#table-of-contents)

<a id="alter-table-advanced"></a>

### Advanced

**Adding a column to a large table without locking it for long.** Per the [official docs](https://www.postgresql.org/docs/current/sql-altertable.html), adding a column with a **constant** default (like `DEFAULT 'USD'` above) does *not* rewrite the table — Postgres just records the default in the catalog and back-fills the value lazily as rows are read, so it's fast even on huge tables. This has been true for a long time and is worth knowing because a lot of older blog posts still recommend the three-step "add nullable column → backfill in batches → add NOT NULL constraint" dance to avoid a rewrite that, for a constant default, no longer happens.

That dance is still the right call when:
- The default is **volatile** (`clock_timestamp()`, `random()`, a sequence) — those *do* require rewriting every row immediately, because each row needs its own computed value rather than one shared constant.
- You're adding a `NOT NULL` constraint to an *existing* column — that requires a full table scan to verify no existing row violates it, and holds an `ACCESS EXCLUSIVE` lock for the duration on older versions. Prefer `ALTER TABLE ... ADD CONSTRAINT ... NOT NULL NOT VALID` followed by `VALIDATE CONSTRAINT`, which validates without blocking concurrent reads/writes for as long.

```sql
-- safer pattern for large, already-populated tables
ALTER TABLE books ADD CONSTRAINT books_title_not_null CHECK (title IS NOT NULL) NOT VALID;
ALTER TABLE books VALIDATE CONSTRAINT books_title_not_null;
```

[↑ back to top](#table-of-contents)

---

## Core Data Types

<a id="data-types-beginner"></a>

### Beginner

| Category | Type | Notes |
|---|---|---|
| Integer | `smallint`, `integer`, `bigint` | 2/4/8 bytes. `integer` is the default choice. |
| Exact decimal | `numeric(p,s)` | Exact precision — use for money, never `real`/`double precision`. |
| Floating point | `real`, `double precision` | Approximate (IEEE 754) — fine for scientific data, wrong for currency. |
| Text | `text` | Unlimited-length string. The type to reach for by default. |
| Boolean | `boolean` | `true`/`false`/`null` — also accepts `'t'`, `'yes'`, `'1'` on input. |
| Date/time | `date`, `time`, `timestamp`, `timestamptz` | See below — `timestamptz` is almost always what you want. |

```sql
CREATE TABLE example (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    label       text,
    amount      numeric(10,2),
    is_active   boolean DEFAULT true,
    created_at  timestamptz DEFAULT now()
);
```

**Try It:**

```sql
SELECT 0.1::real + 0.2::real = 0.3::real;      -- false: floating-point rounding
SELECT 0.1::numeric + 0.2::numeric = 0.3::numeric; -- true: exact decimal arithmetic
```

[↑ back to top](#table-of-contents)

<a id="data-types-working-knowledge"></a>

### Working Knowledge

**`char(n)` vs `varchar(n)` vs `text`.** Per the [official docs](https://www.postgresql.org/docs/current/datatype-character.html), all three store data the same way internally, and there is **no performance difference** — this contradicts a persistent old-school convention (carried over from other databases) that `varchar` is faster than `text`. Use `text` unless you have an actual business rule capping length, in which case `varchar(n)` documents that rule for you; avoid `char(n)`, which blank-pads values to a fixed width and rarely does what beginners expect.

```sql
SELECT 'abc'::char(5) = 'abc  '::char(5);  -- true: char(n) pads with trailing spaces
SELECT 'abc'::text    = 'abc  '::text;     -- false: text stores exactly what you gave it
```

**`timestamp` vs `timestamptz`.** `timestamp` (a.k.a. `timestamp without time zone`) stores a naive date+time with no zone info. `timestamptz` stores a point in time — internally UTC — and converts to/from the session's `TimeZone` setting on input/output. Nearly every real application wants `timestamptz`, because "what time was it" is meaningless without knowing whose clock; `timestamp` is easy to reach for by habit and then produces bugs the moment two servers disagree on local time zone.

```sql
SET TIME ZONE 'UTC';
CREATE TABLE events (id int, happened_at timestamptz);
INSERT INTO events VALUES (1, '2026-09-03 09:00:00-04:00'); -- stored as instant, converted to UTC

SET TIME ZONE 'America/New_York';
SELECT happened_at FROM events;   -- same instant, displayed in New York local time
```

**Real Scenario:** Try storing a price as `real` and watch it silently drift, then fix it with `numeric`:

```sql
CREATE TABLE prices_bad (id int, amount real);
INSERT INTO prices_bad VALUES (1, 19.99), (2, 0.10), (3, 0.20);
SELECT sum(amount) FROM prices_bad;   -- may not equal 20.29 exactly

CREATE TABLE prices_good (id int, amount numeric(10,2));
INSERT INTO prices_good VALUES (1, 19.99), (2, 0.10), (3, 0.20);
SELECT sum(amount) FROM prices_good;  -- exact: 20.29
```

For structured/semi-structured data — JSON documents, arrays, ranges, composite types, `UUID` as a modeling choice — see [Data Modeling & Advanced Types](./postgres-data-modeling-types.md); this file deliberately stops at the scalar built-ins.

[↑ back to top](#table-of-contents)

---

## Inserting Data (INSERT)

<a id="insert-beginner"></a>

### Beginner

```sql
INSERT INTO books (title, price) VALUES ('Dune', 9.99);
INSERT INTO books (title, price) VALUES
    ('Dune Messiah', 8.99),
    ('Children of Dune', 8.99);
```

Columns you omit get their default (or `NULL` if there's no default and no `NOT NULL` constraint).

[↑ back to top](#table-of-contents)

<a id="insert-working-knowledge"></a>

### Working Knowledge

**Getting values back out with `RETURNING`:**

```sql
INSERT INTO books (title, price) VALUES ('Dune', 9.99) RETURNING id, title;
```

`RETURNING` avoids a round-trip `SELECT` to find the ID an identity column just generated — genuinely useful, not just convenient, since without it there's a race between your insert and any concurrent inserts if you tried to guess the new ID yourself.

**Upserts with `ON CONFLICT`:**

```sql
INSERT INTO books (id, title, price) VALUES (1, 'Dune', 12.99)
ON CONFLICT (id) DO UPDATE SET price = EXCLUDED.price;
```

`EXCLUDED` refers to the row that *would* have been inserted — this is how you say "update with the new value" inside the same statement.

**Real Scenario:** Try an insert that violates a unique constraint, then fix it with `ON CONFLICT`:

```sql
CREATE TABLE books (id integer PRIMARY KEY, title text, price numeric(8,2));
INSERT INTO books VALUES (1, 'Dune', 9.99);
INSERT INTO books VALUES (1, 'Dune (reprint)', 11.99); -- fails: duplicate key

INSERT INTO books VALUES (1, 'Dune (reprint)', 11.99)
ON CONFLICT (id) DO UPDATE SET title = EXCLUDED.title, price = EXCLUDED.price;
SELECT * FROM books; -- row 1 now shows the reprint's title and price
```

[↑ back to top](#table-of-contents)

---

## Reading Data (SELECT basics)

<a id="select-beginner"></a>

### Beginner

```sql
SELECT * FROM books;
SELECT title, price FROM books WHERE price < 10;
SELECT title FROM books ORDER BY price DESC LIMIT 5;
```

[↑ back to top](#table-of-contents)

<a id="select-working-knowledge"></a>

### Working Knowledge

**`NULL` breaks ordinary equality.** `NULL` means "unknown," so `= NULL` never matches anything, even another `NULL`.

```sql
-- wrong: matches zero rows even where price genuinely is unset
SELECT * FROM books WHERE price = NULL;

-- right
SELECT * FROM books WHERE price IS NULL;
```

**Aggregates and `GROUP BY`:**

```sql
SELECT author_id, count(*), avg(price)
FROM books
GROUP BY author_id
HAVING count(*) > 1;
```

`WHERE` filters rows before grouping; `HAVING` filters groups after aggregation — that's the whole reason both exist.

**Real Scenario:** Try to find authors with more than one book, first the wrong way, then correctly:

```sql
-- wrong: WHERE can't see the aggregate result yet
SELECT author_id, count(*) FROM books WHERE count(*) > 1 GROUP BY author_id;

-- right
SELECT author_id, count(*) FROM books GROUP BY author_id HAVING count(*) > 1;
```

[↑ back to top](#table-of-contents)

---

## Updating Data (UPDATE)

<a id="update-beginner"></a>

### Beginner

```sql
UPDATE books SET price = 7.99 WHERE title = 'Dune';
```

**Wrong vs. right — the classic missing-WHERE:**

```sql
-- wrong: sets EVERY row's price to 7.99
UPDATE books SET price = 7.99;

-- right: scope it
UPDATE books SET price = 7.99 WHERE title = 'Dune';
```

[↑ back to top](#table-of-contents)

<a id="update-working-knowledge"></a>

### Working Knowledge

**Confirm the blast radius before committing.** Wrap risky updates in a transaction and check `RETURNING`/row count before you `COMMIT`:

```sql
BEGIN;
UPDATE books SET price = price * 1.10 WHERE price < 10 RETURNING id, title, price;
-- inspect the output; if it looks right:
COMMIT;
-- if not:
ROLLBACK;
```

**Real Scenario:** Try an update that looks safe but touches more than you think:

```sql
CREATE TABLE books (id int, title text, price numeric(8,2));
INSERT INTO books VALUES (1, 'Dune', 9.99), (2, 'Dune Messiah', NULL);

BEGIN;
UPDATE books SET price = 8.99 WHERE price < 10 RETURNING id, title; -- NULL row untouched, as expected
ROLLBACK; -- undo, since this was just a check
```

[↑ back to top](#table-of-contents)

---

## Deleting Data (DELETE)

<a id="delete-beginner"></a>

### Beginner

```sql
DELETE FROM books WHERE title = 'Dune';
```

Same missing-`WHERE` danger as `UPDATE` — `DELETE FROM books;` with no `WHERE` empties the whole table.

[↑ back to top](#table-of-contents)

<a id="delete-working-knowledge"></a>

### Working Knowledge

**`DELETE` vs `TRUNCATE`.** `DELETE FROM books;` removes rows one by one (logged, triggerable, slower on large tables); `TRUNCATE books;` deallocates the table's storage instantly but cannot be filtered with `WHERE` and, by default, cannot cross a `REFERENCES` relationship without `CASCADE`.

```sql
TRUNCATE books;                 -- fails if another table has a live FK referencing books
TRUNCATE books CASCADE;         -- also truncates dependent tables — use with real caution
```

**Real Scenario:** Try deleting with a subquery to remove books by an author who no longer exists:

```sql
DELETE FROM books
WHERE author_id NOT IN (SELECT id FROM authors);
```

Run the equivalent `SELECT` first to see what would be deleted before running the `DELETE` — a habit worth keeping for any filtered delete:

```sql
SELECT * FROM books WHERE author_id NOT IN (SELECT id FROM authors);
```

[↑ back to top](#table-of-contents)

---

## Cheat Sheet

**psql meta-commands**

| Command | Purpose |
|---|---|
| `\c dbname [user] [host] [port]` | Connect to a different database |
| `\conninfo` | Show current connection details |
| `\l` | List databases |
| `\dn` | List schemas |
| `\dt` | List tables in the search path |
| `\d tablename` | Describe a table (columns, indexes, constraints) |
| `\d+ tablename` | Same, with extra detail (storage, comments) |
| `\x` / `\x auto` | Toggle expanded (one-column-per-line) output |
| `\timing on` | Show execution time per statement |
| `\i file.sql` | Run SQL from a file |
| `\q` | Quit |

**DDL patterns**

| Task | Statement |
|---|---|
| Create a table safely | `CREATE TABLE IF NOT EXISTS t (...)` |
| Add auto-incrementing key | `id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY` |
| Add a column fast (large table) | `ALTER TABLE t ADD COLUMN c type DEFAULT <constant>` |
| Add NOT NULL without a long lock | `ADD CONSTRAINT ... CHECK (...) NOT VALID` then `VALIDATE CONSTRAINT` |
| Change a schema search order | `SET search_path TO schema1, schema2` |

**DML patterns**

| Task | Statement |
|---|---|
| Insert and get the new row back | `INSERT INTO t (...) VALUES (...) RETURNING *` |
| Upsert | `INSERT ... ON CONFLICT (key) DO UPDATE SET col = EXCLUDED.col` |
| Filter for missing values | `WHERE col IS NULL` (never `= NULL`) |
| Filter groups after aggregation | `GROUP BY ... HAVING ...` |
| Preview a risky UPDATE/DELETE | Wrap in `BEGIN; ... RETURNING ...;` then `ROLLBACK;` or `COMMIT;` |

[↑ back to top](#table-of-contents)

---

← Back to [Postgres SQL Mastery Guide](./postgres-mastery-guide.md)
