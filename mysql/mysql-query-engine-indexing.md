# MySQL Query Engine & Indexing

Part of the [MySQL Mastery Guide](./mysql-mastery-guide.md). Full Beginner → Mastery depth. Uses a bigger table so index effects are actually visible:

```sql
CREATE TABLE orders (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  customer_id BIGINT UNSIGNED NOT NULL,
  status VARCHAR(20) NOT NULL,
  total DECIMAL(10,2) NOT NULL,
  placed_at DATETIME NOT NULL
) ENGINE=InnoDB;

-- Populate with ~500k synthetic rows for any Working Knowledge+ example
-- below, e.g. via a recursive CTE insert loop or a script.
```

## Table of Contents

- [Clustered vs. Secondary Indexes](#clustered-vs-secondary-indexes-innodbs-biggest-departure-from-postgres)
- [Reading EXPLAIN and EXPLAIN ANALYZE](#reading-explain-and-explain-analyze)
- [Choosing an Index Type](#choosing-an-index-type)
- [Optimizer Statistics: ANALYZE TABLE and Histograms](#optimizer-statistics-analyze-table-and-histograms)
- [Why the Optimizer Didn't Use Your Index](#why-the-optimizer-didnt-use-your-index)
- [Covering Indexes and Index-Only Access](#covering-indexes-and-index-only-access)
- [Invisible, Descending, and Functional Indexes](#invisible-descending-and-functional-indexes)
- [Cheat Sheet: Index Types Compared](#cheat-sheet-index-types-compared)

---

## Clustered vs. Secondary Indexes: InnoDB's Biggest Departure from Postgres

**Working Knowledge — foundational, read this before anything else in this file.**

Postgres's heap-plus-indexes model has no true clustering by default (`CLUSTER` is a one-time manual reorg). InnoDB is structurally different: **every table *is* a B-tree ordered by its primary key**, and row data lives directly in the leaf nodes of that tree — this is the *clustered index*. Every other index (a "secondary index") stores only the indexed column(s) plus the primary key value, and a lookup through a secondary index does a second traversal — down the secondary index to find the PK, then down the clustered index to fetch the row. This is called a **bookmark lookup**.

```
Clustered index (= the table itself, ordered by PK):
┌─────────────────────────────────────────┐
│ id=1 [full row]  id=2 [full row]  id=3…  │
└─────────────────────────────────────────┘

Secondary index on status:
┌───────────────────────────────┐        two-step lookup:
│ status='shipped' → id=1        │──┐     1. find id in secondary index
│ status='shipped' → id=4        │  │     2. fetch full row from clustered
│ status='pending' → id=2        │  └───► index by that id ("bookmark lookup")
└───────────────────────────────┘
```

Implications that don't exist in Postgres:

- **Primary key choice affects every other index's size and every secondary-index lookup's cost**, because the PK value is duplicated into every secondary index. A wide or ever-growing-then-shrinking PK (e.g. a UUID inserted in random order) bloats every index and causes page splits throughout the whole table, not just one.
- **A monotonically increasing PK (like `AUTO_INCREMENT`) is the cheap case** — new rows append to the end of the clustered index instead of splitting pages in the middle.

```sql
-- Wrong: random-order UUID as primary key on a high-insert-rate table —
-- every insert lands in a random spot in a multi-GB B-tree, causing
-- constant page splits and fragmentation.
CREATE TABLE events (id CHAR(36) PRIMARY KEY, payload JSON) ENGINE=InnoDB;
-- id values like 'a3f1...' arrive in random order.

-- Right: AUTO_INCREMENT surrogate PK, UUID as a secondary UNIQUE column
-- if you need a public-facing opaque identifier
CREATE TABLE events (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  public_id CHAR(36) NOT NULL,
  payload JSON,
  UNIQUE KEY uq_events_public_id (public_id)
) ENGINE=InnoDB;
```

See the official [Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html) reference for the exact fallback rules when no primary key exists (InnoDB picks the first `UNIQUE NOT NULL` index, or else generates a hidden, unusable `GEN_CLUST_INDEX`).

[Back to top](#mysql-query-engine--indexing)

---

## Reading EXPLAIN and EXPLAIN ANALYZE

**Working Knowledge.**

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42 AND status = 'shipped';
```

Classic tabular `EXPLAIN` shows one row per table accessed, with `type` (access method — `const`, `eq_ref`, `ref`, `range`, `index`, `ALL` from best to worst), `key` (index actually chosen), `rows` (estimated rows examined), and `Extra` (`Using index`, `Using filesort`, `Using temporary` — the last two are common red flags on large tables).

**Advanced.** `EXPLAIN ANALYZE` (8.0.18+) actually runs the query and reports real timing and row counts per step, not just estimates — always `FORMAT=TREE`, the only format that shows hash-join usage:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42 AND status = 'shipped';
```

```
-> Filter: (orders.status = 'shipped')  (cost=... rows=... actual time=0.05..1.2 rows=8 loops=1)
    -> Index lookup on orders using idx_customer_id (customer_id=42)
       (cost=... rows=40 actual time=0.03..0.9 rows=40 loops=1)
```

Read it bottom-up: the leaf `Index lookup` step ran first and found 40 candidate rows, then the `Filter` step above it discarded all but 8. If `rows` (estimate) and `actual ... rows` diverge wildly, your table statistics are stale — see [Optimizer Statistics](#optimizer-statistics-analyze-table-and-histograms) below.

```sql
-- Wrong: trusting the estimated `rows` column alone on a query you
-- suspect is slow — estimates can be badly wrong on skewed data.
EXPLAIN SELECT * FROM orders WHERE status = 'cancelled';

-- Right: EXPLAIN ANALYZE to see what actually happened
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'cancelled';
```

**Real Scenario (try it yourself):** On the 500k-row `orders` table, run `EXPLAIN` on `SELECT * FROM orders WHERE status = 'cancelled'` before adding any index on `status`. You'll see `type: ALL` (full table scan) and a large `rows` estimate. Add `CREATE INDEX idx_status ON orders(status);`, re-run, and watch `type` change to `ref` — but if `cancelled` orders happen to be 40% of the table, the optimizer may *still* choose a full scan, because reading most of the table via a secondary index (with a bookmark lookup per row) costs more than just scanning it directly. This is expected, not a bug — see [Why the Optimizer Didn't Use Your Index](#why-the-optimizer-didnt-use-your-index).

Official reference: [EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.4/en/explain.html).

[Back to top](#mysql-query-engine--indexing)

---

## Choosing an Index Type

**Working Knowledge.**

| Index type | Storage engine support | Good for |
|---|---|---|
| `BTREE` (default) | InnoDB, MyISAM | Equality, range, prefix, sorting — the default and correct choice almost always |
| `HASH` | Memory engine only (InnoDB has an internal *adaptive hash index* you don't create directly) | Pure equality lookups on a Memory table |
| `FULLTEXT` | InnoDB, MyISAM | Natural-language / boolean text search — see [Data Modeling & Types §Full-Text Search](./mysql-data-modeling-types.md#full-text-search) |
| `SPATIAL` (`R-TREE`) | InnoDB, MyISAM | Geometry columns |

```sql
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);
```

**Column order in a composite index matters** — it's usable left-to-right, same principle as Postgres. An index on `(customer_id, status)` serves `WHERE customer_id = ?` and `WHERE customer_id = ? AND status = ?`, but not `WHERE status = ?` alone.

[Back to top](#mysql-query-engine--indexing)

---

## Optimizer Statistics: ANALYZE TABLE and Histograms

**Advanced.** InnoDB's cardinality statistics are estimated by sampling, not exact, and go stale as data changes — `ANALYZE TABLE` recomputes them on demand:

```sql
ANALYZE TABLE orders;
```

For columns without a usable index (so no per-value cardinality is otherwise tracked), MySQL 8.0+ supports **histograms** to give the optimizer a real data-distribution estimate instead of a flat guess:

```sql
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 10 BUCKETS;

SELECT * FROM information_schema.column_statistics
WHERE table_name = 'orders';
```

Histograms are computed once and stored — they don't auto-refresh on writes, so a table with a rapidly shifting value distribution (e.g. `status` values cycling from `pending` to `shipped` constantly) needs a periodic re-`ANALYZE`, not a one-time setup. This is a real operational gotcha absent from Postgres's `autovacuum`-driven, always-fresh-ish statistics model. See the official [ANALYZE TABLE Statement](https://dev.mysql.com/doc/refman/8.4/en/analyze-table.html) and [Optimizer Statistics](https://dev.mysql.com/doc/refman/8.4/en/optimizer-statistics.html) references.

[Back to top](#mysql-query-engine--indexing)

---

## Why the Optimizer Didn't Use Your Index

**Advanced.** The most common real-world causes, in the order they actually come up:

1. **The predicate reads most of the table.** As in the Real Scenario above — the optimizer correctly judges a full scan cheaper than thousands of bookmark lookups.
2. **Implicit type coercion.** Comparing a string column to a number (or vice versa) can force MySQL to convert every row's value before comparing, disabling index use:

```sql
-- Wrong: order_code is VARCHAR, comparing to a bare number forces
-- a conversion of every row and skips the index on order_code.
SELECT * FROM orders WHERE order_code = 12345;

-- Right: match the column's actual type
SELECT * FROM orders WHERE order_code = '12345';
```

3. **A function wrapped around the indexed column.** `WHERE YEAR(placed_at) = 2026` can't use a plain index on `placed_at` — rewrite as a range, or build a [functional index](#invisible-descending-and-functional-indexes):

```sql
-- Wrong: function on the column defeats a plain index on placed_at
SELECT * FROM orders WHERE YEAR(placed_at) = 2026;

-- Right: sargable range predicate
SELECT * FROM orders WHERE placed_at >= '2026-01-01' AND placed_at < '2027-01-01';
```

4. **Leading-wildcard `LIKE`.** `WHERE name LIKE '%smith'` can't use a B-tree index (no fixed prefix to seek on) — `WHERE name LIKE 'smith%'` can. A true leading-wildcard search needs `FULLTEXT` or a trigram-style approach.

Confirm which of these is happening with `EXPLAIN` — `Extra: Using where` combined with `type: ALL` on a column you expected indexed is the signal to dig in.

[Back to top](#mysql-query-engine--indexing)

---

## Covering Indexes and Index-Only Access

**Advanced.** If every column a query needs is present in a secondary index (indexed columns plus the implicitly-included primary key), InnoDB never has to do the bookmark lookup into the clustered index — `EXPLAIN` reports `Extra: Using index`.

```sql
CREATE INDEX idx_orders_covering ON orders (customer_id, status, total);

-- Covered entirely by the index above — no clustered-index lookup needed
SELECT customer_id, status, total FROM orders WHERE customer_id = 42;
```

```sql
-- Wrong: SELECT * defeats a carefully built covering index, forcing a
-- bookmark lookup for every matched row even though the WHERE and ORDER
-- BY columns are all covered.
SELECT * FROM orders WHERE customer_id = 42 ORDER BY status;

-- Right: select only what the index actually covers
SELECT customer_id, status, total FROM orders WHERE customer_id = 42 ORDER BY status;
```

[Back to top](#mysql-query-engine--indexing)

---

## Invisible, Descending, and Functional Indexes

**Mastery.** Three MySQL 8.0+ index capabilities with no Postgres equivalent in the same form:

**Invisible indexes** — mark an index invisible to the optimizer without dropping it, to safely test the impact of removal before committing:

```sql
ALTER TABLE orders ALTER INDEX idx_orders_covering INVISIBLE;
-- Run production traffic / EXPLAIN checks. If nothing regresses:
ALTER TABLE orders DROP INDEX idx_orders_covering;
-- If something did regress, flip it back instantly — no rebuild:
ALTER TABLE orders ALTER INDEX idx_orders_covering VISIBLE;
```

This is fundamentally safer than `DROP INDEX` directly: making an index invisible/visible is a fast metadata-only operation, so a mistaken removal is a one-line, instant fix instead of a full index rebuild. See the official [Invisible Indexes](https://dev.mysql.com/doc/refman/8.4/en/invisible-indexes.html) reference.

**Descending indexes** (8.0+, InnoDB/BTREE only) — store a column in actual descending physical order, so `ORDER BY a ASC, b DESC` can be satisfied by one index scan instead of a filesort:

```sql
CREATE INDEX idx_orders_sort ON orders (customer_id ASC, placed_at DESC);
```

**Functional (expression) indexes** (8.0.13+) — index the result of an expression, letting you fix the "function wrapped around the column" problem from the previous section without denormalizing:

```sql
CREATE INDEX idx_orders_year ON orders ((YEAR(placed_at)));

SELECT * FROM orders WHERE YEAR(placed_at) = 2026;  -- now indexable
```

[Back to top](#mysql-query-engine--indexing)

---

## Cheat Sheet: Index Types Compared

| Situation | What to reach for |
|---|---|
| Equality / range on one or more leading columns | Composite `BTREE` index, correct column order |
| Query only needs indexed columns + PK | Covering index (`SELECT` only those columns) |
| Testing whether an index is safe to drop | `ALTER TABLE t ALTER INDEX i INVISIBLE` first |
| `ORDER BY a, b DESC` avoiding filesort | Descending index: `(a ASC, b DESC)` |
| `WHERE fn(col) = x` | Functional index: `((fn(col)))` or rewrite as a sargable range |
| Full-text search | `FULLTEXT` index, not `LIKE '%term%'` |
| Stale-looking row-count estimates | `ANALYZE TABLE`, optionally `UPDATE HISTOGRAM ON col` |
| Confirm real (not estimated) cost | `EXPLAIN ANALYZE` (always `FORMAT=TREE`) |

[Back to top](#mysql-query-engine--indexing)
