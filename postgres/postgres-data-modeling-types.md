Part of the [Postgres SQL Mastery Guide](./postgres-mastery-guide.md).

# Data Modeling & Advanced Types

Scalar built-ins (`int`, `text`, `timestamp`, etc.) are covered in [Foundations](./postgres-foundations.md) — this file assumes you know those and picks up where structure gets interesting: constraints, when to break normal form on purpose, PostgreSQL 18's temporal constraints, generated columns, JSONB, arrays, ENUMs, and full-text search.

## Table of Contents

- [Constraints: The Rules Your Data Must Obey](#constraints-the-rules-your-data-must-obey)
- [Normalization Tradeoffs: When to Denormalize on Purpose](#normalization-tradeoffs-when-to-denormalize-on-purpose)
- [Temporal Constraints (New in PostgreSQL 18)](#temporal-constraints-new-in-postgresql-18)
- [Generated Columns: Stored vs. Virtual](#generated-columns-stored-vs-virtual)
- [JSONB: Binary JSON with Indexing](#jsonb-binary-json-with-indexing)
- [Arrays](#arrays)
- [ENUM Types and Their Gotchas](#enum-types-and-their-gotchas)
- [Full-Text Search](#full-text-search)
- [Key Extensions: pg_trgm and a PostGIS Pointer](#key-extensions-pg_trgm-and-a-postgis-pointer)
- [Cheat Sheet](#cheat-sheet)

---

## Constraints: The Rules Your Data Must Obey

**What it is.** A constraint is a rule Postgres enforces on every write, so bad data simply cannot land in the table — no application code required. The five you'll use constantly: `NOT NULL`, `UNIQUE`, `PRIMARY KEY` (NOT NULL + UNIQUE, one per table), `CHECK` (a boolean expression that must hold), and `FOREIGN KEY` (a value must exist in another table's key). See the [official constraints docs](https://www.postgresql.org/docs/current/ddl-constraints.html).

### Beginner

```sql
CREATE TABLE customers (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       text NOT NULL UNIQUE,
    signup_date date NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE orders (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id bigint NOT NULL REFERENCES customers(id),
    total_cents integer NOT NULL CHECK (total_cents >= 0)
);
```

**Try It:**

```sql
-- Violates NOT NULL
INSERT INTO customers (email) VALUES (NULL);
-- Violates CHECK
INSERT INTO orders (customer_id, total_cents) VALUES (1, -500);
-- Violates FOREIGN KEY (no customer 999)
INSERT INTO orders (customer_id, total_cents) VALUES (999, 100);
```

Each should fail with a distinct error (`null value in column`, `violates check constraint`, `violates foreign key constraint`) — read the error, it names the exact constraint.

### Working Knowledge

**Named constraints** matter once you have more than one CHECK per table — an unnamed constraint gets an auto-generated name like `orders_total_cents_check`, which is fine until you need to `ALTER TABLE ... DROP CONSTRAINT` it and have to go look the name up.

```sql
ALTER TABLE orders
  ADD CONSTRAINT total_must_be_positive CHECK (total_cents >= 0);
```

**Wrong vs. right — composite uniqueness:**

```sql
-- WRONG: allows the same SKU to appear in the same warehouse twice
CREATE TABLE inventory (
    warehouse_id int,
    sku          text,
    quantity     int NOT NULL CHECK (quantity >= 0)
);

-- RIGHT: the pair (warehouse_id, sku) is the real unique identity
CREATE TABLE inventory (
    warehouse_id int,
    sku          text,
    quantity     int NOT NULL CHECK (quantity >= 0),
    PRIMARY KEY (warehouse_id, sku)
);
```

**ON DELETE behavior** — decide deliberately, don't accept the default (`NO ACTION`, which blocks the delete):

```sql
CREATE TABLE order_items (
    order_id bigint REFERENCES orders(id) ON DELETE CASCADE,  -- delete order → delete its items
    product  text
);
```

### Advanced

`CHECK` constraints can reference multiple columns, which is how you enforce cross-column invariants without a trigger:

```sql
CREATE TABLE promotions (
    starts_on date NOT NULL,
    ends_on   date NOT NULL,
    CHECK (ends_on > starts_on)
);
```

For "at most one row matches this rule" beyond simple uniqueness — e.g., "only one *active* subscription per user" — a plain `UNIQUE` constraint can't express the condition. Two production-grade options:

```sql
-- Partial unique index: uniqueness only among active rows
CREATE UNIQUE INDEX one_active_subscription
  ON subscriptions (user_id) WHERE status = 'active';
```

```sql
-- EXCLUDE constraint: generalizes uniqueness to any operator, e.g. "no two rows
-- with the same user_id whose date ranges overlap" — the ancestor of PG18's
-- temporal constraints below, and still what you reach for pre-18 or for
-- non-range overlap conditions.
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE room_bookings (
    room_id int,
    during  tsrange,
    EXCLUDE USING gist (room_id WITH =, during WITH &&)
);
```

Real-world gotcha: adding a `FOREIGN KEY` or `CHECK` to an existing large table takes a lock and validates every existing row by default, which can stall production traffic. Use `NOT VALID` then validate separately:

```sql
ALTER TABLE orders
  ADD CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES customers(id) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT fk_customer; -- lighter lock, scans in the background
```

See [Stack Overflow: adding a NOT NULL/FK constraint without long locks](https://stackoverflow.com/questions/tagged/postgresql+locking) for the pattern in practice, and the [ALTER TABLE docs](https://www.postgresql.org/docs/current/sql-altertable.html) for exact lock levels.

[↑ back to top](#table-of-contents)

---

## Normalization Tradeoffs: When to Denormalize on Purpose

**What it is.** Normalization (splitting data so each fact lives in exactly one place) prevents update anomalies — change a customer's name once, not in 10,000 order rows. Denormalization (deliberately duplicating or pre-aggregating data) trades that safety for read speed or simplicity. Neither is "correct" in the abstract — it's a tradeoff you make with eyes open, row by row.

### Working Knowledge

Classic normalized shape:

```sql
CREATE TABLE customers (id bigint PRIMARY KEY, name text NOT NULL);
CREATE TABLE orders (
    id bigint PRIMARY KEY,
    customer_id bigint REFERENCES customers(id)
    -- customer NAME is NOT duplicated here — join to get it
);
```

A common, *deliberate* denormalization: snapshotting a value at the time of the transaction, because the "current" value would be historically wrong.

```sql
CREATE TABLE orders (
    id bigint PRIMARY KEY,
    customer_id bigint REFERENCES customers(id),
    customer_name_at_purchase text NOT NULL  -- intentionally duplicated
);
```

This isn't sloppy modeling — a customer legally changing their name shouldn't rewrite history on a five-year-old invoice. The rule of thumb: normalize for *current* facts, snapshot for *historical* facts.

### Advanced

Other legitimate reasons to denormalize:

- **Read-heavy aggregates** — a `product.review_count` and `product.avg_rating` column updated by trigger, avoiding a `COUNT()`/`AVG()` over millions of review rows on every product-page load.
- **Avoiding a hot join path** — duplicating a rarely-changing `team_name` onto a high-volume `events` table so dashboards don't join a huge table against a small one on every query.
- **JSONB for genuinely variable shape** (covered below) — when normalizing would mean a table-per-attribute explosion for data that doesn't have a fixed schema (e.g., arbitrary product attributes across categories).

The cost you're accepting: you now own keeping the duplicate in sync (trigger, application code, or scheduled reconciliation), and a bug there silently produces wrong numbers instead of a loud constraint violation. A materialized view is the middle ground — looks denormalized to readers, but is mechanically derived and refreshable:

```sql
CREATE MATERIALIZED VIEW product_ratings AS
  SELECT product_id, COUNT(*) AS review_count, AVG(rating) AS avg_rating
  FROM reviews GROUP BY product_id;

REFRESH MATERIALIZED VIEW CONCURRENTLY product_ratings; -- needs a unique index on product_id
```

**Real Scenario (try it):** Build both versions of a "top rated products" query — one joining `reviews` live, one reading a denormalized `avg_rating` column — over a few hundred thousand synthetic review rows, and compare with `EXPLAIN ANALYZE`. The denormalized version wins on read latency every time; the question worth sitting with is *how* you'd keep it correct (trigger on every insert? nightly batch? accept some staleness?) — that's the real tradeoff, not the query speed itself.

[↑ back to top](#table-of-contents)

---

## Temporal Constraints (New in PostgreSQL 18)

**What it is.** Before PostgreSQL 18, "this row's validity period must not overlap another row's" required an `EXCLUDE USING gist` constraint (shown above) — correct, but not discoverable as a keying concept, and not composable with foreign keys at all. PostgreSQL 18 adds native syntax: `WITHOUT OVERLAPS` for primary/unique keys over a range column, and `PERIOD` for foreign keys that reference them. See the [PostgreSQL 18 release notes](https://www.postgresql.org/docs/current/release-18.html) and the [CREATE TABLE docs](https://www.postgresql.org/docs/current/sql-createtable.html) for the authoritative syntax — this is genuinely new, so older tutorials will only show the `EXCLUDE` form.

### Advanced / Mastery

**`WITHOUT OVERLAPS` in a UNIQUE or PRIMARY KEY** — the range column must be a range or multirange type (e.g. `tsrange`, `daterange`); non-range columns are compared for equality, the range column for overlap. Under the hood this is enforced with a GiST index, equivalent to `EXCLUDE USING gist (code WITH =, valid_at WITH &&)`:

```sql
CREATE TABLE room_bookings (
    room_id  int,
    valid_at tsrange NOT NULL,
    UNIQUE (room_id, valid_at WITHOUT OVERLAPS)
);

-- OK: different rooms, or same room non-overlapping periods
INSERT INTO room_bookings VALUES
  (101, '[2026-09-03 09:00, 2026-09-03 10:00)'),
  (101, '[2026-09-03 10:00, 2026-09-03 11:00)');

-- FAILS: overlaps the first booking for room 101
INSERT INTO room_bookings VALUES (101, '[2026-09-03 09:30, 2026-09-03 10:30)');
```

```
room 101 timeline:
  [09:00 ────── 10:00)[10:00 ────── 11:00)     <- allowed, back-to-back, no overlap
        [09:30 ────── 10:30)                   <- rejected, straddles both
```

**`PERIOD` in a FOREIGN KEY** — checks *range containment* instead of equality: the referenced table's combined periods must fully cover the referencing row's period, not just match a single row.

```sql
CREATE TABLE companies (
    id         int,
    valid_at   tsrange NOT NULL,
    PRIMARY KEY (id, valid_at WITHOUT OVERLAPS)
);

CREATE TABLE contracts (
    id                 bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    company_id         int NOT NULL,
    contract_period    tsrange NOT NULL,
    FOREIGN KEY (company_id, PERIOD contract_period)
      REFERENCES companies (id, PERIOD valid_at)
);
```

A contract can only exist for a period fully covered by that company's recorded existence — no more "the company row got archived/date-bounded but the contract silently pointed at nothing meaningful."

**Caveats to know before reaching for this:**
- `ON DELETE`/`ON UPDATE` actions `RESTRICT`, `CASCADE`, `SET NULL`, `SET DEFAULT` are **not supported** on temporal foreign keys yet.
- Empty ranges are rejected outright.
- For non-range scalar columns to participate in the GiST-backed exclusion alongside the range, you need the `btree_gist` extension (same as the pre-18 `EXCLUDE` pattern).

**Real Scenario (try it):** Model employee-department assignments with `WITHOUT OVERLAPS` so no employee is assigned to two departments at once, then try inserting an overlapping assignment and read the error Postgres gives you — compare it to what you'd have gotten from a `CHECK` constraint trying to do the same thing procedurally (it can't, without a trigger, since it needs to see *other rows*).

[↑ back to top](#table-of-contents)

---

## Generated Columns: Stored vs. Virtual

**What it is.** A generated column's value is computed from other columns in the same row, not written directly — Postgres derives it so it can never drift out of sync with the columns it depends on. Until PostgreSQL 18, generated columns were always **`STORED`**: computed on write, occupying disk space like a normal column. PostgreSQL 18 adds **`VIRTUAL`** columns — computed on read, occupying no storage — and makes `VIRTUAL` the default when neither keyword is written. See [Generated Columns in the docs](https://www.postgresql.org/docs/current/ddl-generated-columns.html).

### Working Knowledge

```sql
CREATE TABLE measurements (
    height_cm numeric,
    height_in numeric GENERATED ALWAYS AS (height_cm / 2.54) STORED
);

INSERT INTO measurements (height_cm) VALUES (180);
-- Wrong: can't write to a generated column directly
UPDATE measurements SET height_in = 999;  -- ERROR
-- Right: update the source column, the generated one follows automatically
UPDATE measurements SET height_cm = 175;
SELECT * FROM measurements;
```

### Advanced / Mastery

**PostgreSQL 18 flips the default.** Writing `GENERATED ALWAYS AS (expr)` with no storage keyword now means `VIRTUAL`, not `STORED` — a real behavior change from every pre-18 tutorial, which only had `STORED` to write about. Be explicit in schema you intend to keep, so a future reader (or a Postgres version bump) doesn't have to infer which behavior you meant:

```sql
-- PG18+: computed fresh on every read, zero bytes on disk
height_in numeric GENERATED ALWAYS AS (height_cm / 2.54) VIRTUAL

-- explicit STORED: computed once at write time, occupies disk, reads are "free"
height_in numeric GENERATED ALWAYS AS (height_cm / 2.54) STORED
```

**The tradeoff, concretely:**

| | VIRTUAL | STORED |
|---|---|---|
| Disk space | none | full column storage |
| Write cost | none (nothing computed) | recomputed on every INSERT/UPDATE of a dependency |
| Read cost | recomputed every SELECT | none — just reads the stored value |
| Can be indexed | as of PG18, yes | yes |
| Logical replication | nothing to replicate (no physical value) | supported via `publish_generated_columns = stored` |
| Good fit | cheap expressions, rarely-read columns, saving storage on wide tables | expensive expressions, hot-read columns, columns you'll index heavily |

**Restrictions that apply to `VIRTUAL` specifically:** the expression can only use built-in functions and types — no user-defined functions/types, directly or through an operator/cast that resolves to one. Both `VIRTUAL` and `STORED` share the general restrictions: no writing to the column directly, the expression can't reference other generated columns or system columns (other than `tableoid`), no subqueries, only immutable functions, and a generated column can't have its own default, identity, or be part of a partition key.

**Real Scenario (try it):** Add a generated `full_name` column two ways on a copy of a customers table with a few hundred thousand rows — once `VIRTUAL`, once `STORED` — then compare `\d+` (storage), a bulk `UPDATE` of the source columns, and a `SELECT full_name FROM customers` timing under each. Neither answer is universally right — this is the mental model to internalize, not a bug to fix in your schema.

[↑ back to top](#table-of-contents)

---

## JSONB: Binary JSON with Indexing

**What it is.** Postgres has two JSON types: `json` (stores the exact input text verbatim, re-parses on every access) and `jsonb` (stores a decomposed binary representation — no whitespace, no duplicate keys, keys sorted). `jsonb` is slightly slower to *insert* (it does the parsing up front) but much faster to *query*, and — critically — it's the only one of the two that GIN indexes can index. Unless you have a specific reason to preserve exact input formatting, **use `jsonb`**. See [JSON Types docs](https://www.postgresql.org/docs/current/datatype-json.html).

### Beginner

```sql
CREATE TABLE products (
    id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    attrs jsonb NOT NULL DEFAULT '{}'
);

INSERT INTO products (attrs) VALUES
  ('{"color": "red", "size": "M", "tags": ["sale", "new"]}'),
  ('{"color": "blue", "size": "L"}');

SELECT attrs -> 'color' AS color_json,   -- returns jsonb: "red" (with quotes)
       attrs ->> 'color' AS color_text   -- returns text: red
FROM products;
```

**Wrong vs. right — type choice:**

```sql
-- WRONG if you ever intend to query/filter on this column at scale
attrs json

-- RIGHT: same data, indexable, faster to query
attrs jsonb
```

### Working Knowledge

**Containment (`@>`)** — "does this JSONB contain this shape" — is the workhorse operator, and it's what the default GIN index accelerates:

```sql
-- Find products where color is red
SELECT * FROM products WHERE attrs @> '{"color": "red"}';

-- Does the tags array contain "sale"? Use the array containment form
SELECT * FROM products WHERE attrs -> 'tags' @> '"sale"';
```

**Path extraction** with `#>` (returns jsonb) and `#>>` (returns text) for nested values:

```sql
SELECT attrs #>> '{shipping,weight_kg}' FROM products;
```

**Updating in place:**

```sql
UPDATE products SET attrs = attrs || '{"on_sale": true}' WHERE id = 1; -- merge
UPDATE products SET attrs = attrs - 'size' WHERE id = 2;               -- remove a key
UPDATE products SET attrs = jsonb_set(attrs, '{size}', '"XL"') WHERE id = 1;
```

### Advanced

**Indexing strategy is the part that actually bites people in production.** A plain GIN index on the whole column supports `@>`, `?`, `?&`, `?|`:

```sql
CREATE INDEX idx_products_attrs ON products USING GIN (attrs);
```

```
GIN index on jsonb (default jsonb_ops opclass)
┌──────────────┐        ┌───────────────────────────┐
│ key/value    │──────▶ │ posting list of row ctids  │
│ "color"      │        │ [row3, row7, row9, ...]    │
│ "sale" (elem)│        │ [row1, row5, ...]          │
└──────────────┘        └───────────────────────────┘
supports: @>, ?, ?|, ?&      does NOT accelerate: ->>'x' = 'y' equality lookups
```

The single most common "why isn't my index being used" gotcha: a query like `WHERE attrs ->> 'color' = 'red'` does **not** use a default GIN index — that's a text-equality comparison on an extracted value, not a containment check. Either rewrite as containment (`attrs @> '{"color":"red"}'`), or index the specific path with a B-tree expression index:

```sql
CREATE INDEX idx_products_color ON products ((attrs ->> 'color'));
SELECT * FROM products WHERE attrs ->> 'color' = 'red'; -- now uses the expression index
```

See [Stack Overflow: jsonb query not using GIN index](https://stackoverflow.com/questions/tagged/jsonb+gin) for the recurring pattern of this exact confusion, and the [GIN indexing docs](https://www.postgresql.org/docs/current/gin-builtin-opclasses.html) for the `jsonb_ops` (default, indexes everything, bigger) vs. `jsonb_path_ops` (smaller, faster, only supports `@>`) tradeoff:

```sql
CREATE INDEX idx_products_attrs_pathops ON products USING GIN (attrs jsonb_path_ops);
```

### Mastery

**SQL/JSON path language** (`jsonb_path_query` and friends) gives you a query language *inside* the value, closer to XPath/JSONPath than the `->`/`#>` operators — worth it once paths get conditional or you need to reach into arrays of objects:

```sql
SELECT jsonb_path_query(attrs, '$.tags[*] ? (@ == "sale")') FROM products;

-- Extract every product's color only if it's on sale, across many rows,
-- with a proper SQL/JSON path predicate rather than string-matching
SELECT id, jsonb_path_query(attrs, '$.color ? (exists ($.on_sale ? (@ == true)))') 
FROM products;
```

`jsonb_path_query` returns a set (usable in `SELECT`, or wrap in `jsonb_path_query_array` to get a single jsonb array back); `jsonb_path_exists` returns just a boolean, which is often what you actually want in a `WHERE` clause and is cheaper to plan. See the [SQL/JSON path language docs](https://www.postgresql.org/docs/current/functions-json.html#FUNCTIONS-SQLJSON-PATH).

**Extended statistics on correlated JSONB fields.** The planner treats each expression it sees as statistically independent by default, so `WHERE attrs @> '{"color":"red"}' AND attrs ->> 'size' = 'M'` can misestimate row counts badly if color and size are correlated in your actual data (e.g., a particular color only ever ships in one size). `CREATE STATISTICS` on the underlying expressions helps the planner get row-count estimates — hence join order and index choice — right:

```sql
CREATE STATISTICS products_color_size_stats (dependencies)
  ON (attrs ->> 'color'), (attrs ->> 'size') FROM products;
ANALYZE products;
```

See the [extended statistics docs](https://www.postgresql.org/docs/current/planner-stats.html#PLANNER-STATS-EXTENDED) — this is genuinely advanced territory, worth reaching for only once `EXPLAIN ANALYZE` shows the planner's row estimate is far off from reality on a JSONB-heavy query.

**Real Scenario (try it):** Load 100k synthetic product rows with `attrs jsonb`, add both a `jsonb_ops` GIN index and a targeted B-tree expression index on `attrs ->> 'color'`, then run `EXPLAIN ANALYZE` on a containment query vs. an equality-on-extracted-text query — watch which index each one picks (or ignores).

[↑ back to top](#table-of-contents)

---

## Arrays

**What it is.** Postgres columns can natively hold arrays of any type (`integer[]`, `text[]`, even arrays of composite types) — genuinely useful for small, unordered, rarely-individually-queried collections, and genuinely a foot-gun when overused as a substitute for a proper child table.

### Working Knowledge

```sql
CREATE TABLE articles (
    id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tags text[] NOT NULL DEFAULT '{}'
);

INSERT INTO articles (tags) VALUES ('{"postgres","sql","tutorial"}');

SELECT * FROM articles WHERE 'postgres' = ANY(tags);   -- contains element
SELECT * FROM articles WHERE tags @> ARRAY['postgres']; -- containment, same idea
SELECT unnest(tags) FROM articles;                      -- one row per element
```

**Indexing arrays** uses GIN, same operator family idea as JSONB:

```sql
CREATE INDEX idx_articles_tags ON articles USING GIN (tags);
```

### Advanced

**When arrays are the wrong tool:** the moment you need to query "which articles have exactly this tag AND that tag were added by this user on this date" — i.e., per-element metadata — you actually want a join table (`article_tags(article_id, tag_id, added_by, added_at)`), not an array. Arrays don't have foreign keys to their elements, can't enforce that every tag exists in a canonical `tags` table, and updating one element means rewriting the whole array value.

```sql
-- WRONG (in the sense of "will regret it"): tags aren't validated against
-- anything, can't be foreign-keyed, typos silently create phantom tags
tags text[]

-- RIGHT once tags need their own identity/metadata:
CREATE TABLE tags (id int PRIMARY KEY, name text UNIQUE);
CREATE TABLE article_tags (
    article_id bigint REFERENCES articles(id),
    tag_id     int REFERENCES tags(id),
    PRIMARY KEY (article_id, tag_id)
);
```

Rule of thumb: arrays are fine for genuinely small, denormalized, read-mostly lists (a handful of tags, a set of enum-like flags); reach for a join table the moment the elements need their own attributes or referential integrity.

[↑ back to top](#table-of-contents)

---

## ENUM Types and Their Gotchas

**What it is.** `CREATE TYPE ... AS ENUM` gives you a fixed, ordered set of string labels stored efficiently (as an integer internally) with compile-time-ish safety — you can't insert a value outside the set. See the [Enumerated Types docs](https://www.postgresql.org/docs/current/datatype-enum.html).

### Beginner

```sql
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'delivered', 'cancelled');

CREATE TABLE orders (
    id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status order_status NOT NULL DEFAULT 'pending'
);

INSERT INTO orders (status) VALUES ('paid');
INSERT INTO orders (status) VALUES ('lost_in_transit'); -- ERROR: invalid input value
```

### Advanced — the gotcha

**Adding** a value is cheap and safe:

```sql
ALTER TYPE order_status ADD VALUE 'refunded';
```

**Removing or renaming** a value is not straightforward at all — Postgres has no `DROP VALUE`. If a value was ever used anywhere (even in a row that's since been deleted, if it touched an index or was referenced in a view/function), the type is entangled with it. The realistic paths to "remove" a value:

```sql
-- Option A: rename it out of visibility (doesn't actually remove it from the type)
ALTER TYPE order_status RENAME VALUE 'cancelled' TO 'cancelled_deprecated';

-- Option B: the real fix — migrate to a new type entirely
CREATE TYPE order_status_v2 AS ENUM ('pending','paid','shipped','delivered','refunded');
ALTER TABLE orders ALTER COLUMN status TYPE order_status_v2
  USING status::text::order_status_v2; -- fails loudly if any row has a value not in the new set
DROP TYPE order_status;
ALTER TYPE order_status_v2 RENAME TO order_status;
```

Note also: `ALTER TYPE ... ADD VALUE` cannot run inside the same transaction block as a later use of that value pre-PG12; this has been relaxed in modern Postgres but is exactly the kind of thing an old Stack Overflow answer will warn you about incorrectly for your version — [check the current ALTER TYPE docs](https://www.postgresql.org/docs/current/sql-altertype.html) rather than trust a five-year-old answer verbatim.

**Wrong vs. right — reaching for ENUM at all:**

```sql
-- WRONG if this list changes often (e.g., driven by user/admin input, or by
-- business categories that get added weekly) — every addition is a schema migration
status order_status  -- ENUM

-- RIGHT for volatile/lookup-driven sets: a foreign key to a lookup table.
-- Adding a value is just an INSERT, no migration, no lock on the parent table's type.
CREATE TABLE order_statuses (code text PRIMARY KEY);
status text REFERENCES order_statuses(code)
```

Use `ENUM` for genuinely fixed, rarely-changing, small sets (days of week, HTTP-method-like categories) where the compactness and `ORDER BY` behavior (enums sort by declaration order, which is often exactly what you want for a status pipeline) is worth the migration friction. Reach for a lookup table the moment the set is business-data rather than schema.

**Real Scenario (try it):** Create an ENUM, insert a few rows using each value, then try to remove one of the *middle* values (not the last one you added) — notice Postgres gives you no direct way, and walk through the type-swap migration above on a scratch table to feel how much more involved it is than adding a value.

[↑ back to top](#table-of-contents)

---

## Full-Text Search

**What it is.** Full-text search matches on *words* (normalized, stemmed) rather than exact substrings — "running" matches a search for "run". Postgres represents a document as a `tsvector` (a sorted list of normalized lexemes with position info) and a search as a `tsquery` (a boolean expression over lexemes); the `@@` operator matches them. See [Full Text Search docs](https://www.postgresql.org/docs/current/textsearch.html).

### Working Knowledge

```sql
CREATE TABLE articles (
    id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title   text,
    body    text
);

INSERT INTO articles (title, body) VALUES
  ('Understanding Postgres Indexes', 'B-tree and GIN indexes speed up running queries.');

-- to_tsvector normalizes text into lexemes; to_tsquery parses a search expression
SELECT * FROM articles
WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('english', 'run & index');
```

Notice `run` matches `running` and `index` matches `indexes` — that's stemming, the whole point of `tsvector` over `LIKE '%run%'`.

### Advanced

**Don't recompute `to_tsvector` on every query** — precompute it, ideally as a generated column (PG12+, works with either `STORED` or, on PG18, `VIRTUAL` — though `STORED` is usually the right call here since you'll index it, and indexing a `VIRTUAL` column still means Postgres re-derives the value during the index build/maintenance path):

```sql
ALTER TABLE articles ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))) STORED;

CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);

SELECT * FROM articles WHERE search_vector @@ to_tsquery('english', 'run & index');
```

**Ranking results** — a plain `@@` match doesn't sort by relevance; `ts_rank` does:

```sql
SELECT title, ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'index') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

`plainto_tsquery` and `websearch_to_tsquery` are usually better than raw `to_tsquery` for user-typed search boxes — they tolerate ordinary text input (spaces, quotes) instead of requiring `&`/`|` boolean syntax:

```sql
SELECT * FROM articles
WHERE search_vector @@ websearch_to_tsquery('english', 'postgres indexes -mysql');
```

[↑ back to top](#table-of-contents)

---

## Key Extensions: pg_trgm and a PostGIS Pointer

**`pg_trgm`** adds trigram-based fuzzy matching — good for typo-tolerant search, `ILIKE '%partial%'` acceleration, and similarity scoring, which full-text search's stemming approach doesn't cover (full-text search fails on typos; trigram similarity is forgiving of them):

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX idx_articles_title_trgm ON articles USING GIN (title gin_trgm_ops);

SELECT title, similarity(title, 'Postgre Indexs') AS sim
FROM articles
ORDER BY sim DESC LIMIT 5; -- tolerates the typos, still finds "Understanding Postgres Indexes"

-- this index also accelerates ILIKE '%...%' scans that would otherwise be a full table scan
SELECT * FROM articles WHERE title ILIKE '%index%';
```

See the [pg_trgm docs](https://www.postgresql.org/docs/current/pgtrgm.html).

**PostGIS** (mentioned, not covered in depth here) is the extension for geospatial types and operations — points, polygons, distance queries, spatial indexes. If you're modeling anything with real-world coordinates or shapes, it's the tool, not a hand-rolled lat/lng-column approach — see the [PostGIS project docs](https://postgis.net/documentation/) when you get there.

[↑ back to top](#table-of-contents)

---

## Cheat Sheet

**Constraints:**

| Constraint | Enforces | Notes |
|---|---|---|
| `NOT NULL` | column always has a value | cheapest, use liberally |
| `UNIQUE` | no duplicate value(s) across listed columns | can be composite; `NULLS NOT DISTINCT` treats NULLs as duplicates too |
| `PRIMARY KEY` | NOT NULL + UNIQUE, one per table | defines row identity |
| `CHECK` | a boolean expression on the row | can reference multiple columns, not other rows |
| `FOREIGN KEY` | value exists in another table's key | `NOT VALID` + `VALIDATE CONSTRAINT` to avoid long locks on big tables |
| `EXCLUDE USING gist` | no two rows violate a given operator (e.g. `&&` overlap) | needs `btree_gist` for non-range columns |
| `UNIQUE (... WITHOUT OVERLAPS)` | PG18: temporal uniqueness on a range column | range/multirange type required, GiST-backed |
| `FOREIGN KEY (... PERIOD ...)` | PG18: referenced period(s) must cover the row's period | no CASCADE/SET NULL/SET DEFAULT support yet |

**JSONB operators:**

| Operator/Function | Purpose |
|---|---|
| `->` | get JSON value by key/index (returns jsonb) |
| `->>` | get value as text |
| `#>` / `#>>` | get nested value by path array (jsonb / text) |
| `@>` / `<@` | containment (left contains right / right contains left) — **GIN-indexable** |
| `?` / `?\|` / `?&` | key exists / any key exists / all keys exist |
| `\|\|` | concatenate/merge |
| `-` | remove key or array element |
| `jsonb_set(target, path, value)` | update a value at a path |
| `jsonb_path_query(target, path)` | SQL/JSON path language query, returns a set |
| `jsonb_path_exists(target, path)` | boolean existence check, cheap in `WHERE` |

**GIN opclasses for jsonb:** `jsonb_ops` (default — supports `@>`, `?`, `?|`, `?&`, larger index) vs. `jsonb_path_ops` (supports only `@>`, smaller and faster).

[↑ back to top](#table-of-contents)

---

← Back to [Postgres SQL Mastery Guide](./postgres-mastery-guide.md)
