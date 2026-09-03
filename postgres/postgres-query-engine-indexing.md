Part of the [Postgres SQL Mastery Guide](./postgres-mastery-guide.md).

## Table of Contents

- [Setup: A Table Worth Explaining](#setup-a-table-worth-explaining)
- [Beginner: Reading EXPLAIN and EXPLAIN ANALYZE](#beginner-reading-explain-and-explain-analyze)
- [Working Knowledge: The Cost Model and Choosing an Index Type](#working-knowledge-the-cost-model-and-choosing-an-index-type)
- [Working Knowledge: Statistics, ANALYZE, and Autovacuum](#working-knowledge-statistics-analyze-and-autovacuum)
- [Advanced: Why the Planner Didn't Use Your Index](#advanced-why-the-planner-didnt-use-your-index)
- [Advanced: Partial and Expression Indexes](#advanced-partial-and-expression-indexes)
- [Advanced: Multicolumn Indexes and Index-Only Scans](#advanced-multicolumn-indexes-and-index-only-scans)
- [Mastery: B-Tree Skip Scan (PostgreSQL 18)](#mastery-b-tree-skip-scan-postgresql-18)
- [Mastery: Visibility Maps and the Mechanics of Index-Only Scans](#mastery-visibility-maps-and-the-mechanics-of-index-only-scans)
- [Mastery: Extended Statistics for Correlated Columns](#mastery-extended-statistics-for-correlated-columns)
- [Mastery: The Genetic Query Optimizer](#mastery-the-genetic-query-optimizer)
- [Cheat Sheet: Index Types Compared](#cheat-sheet-index-types-compared)

---

## Setup: A Table Worth Explaining

Everything below is meant to be typed into `psql` against a throwaway database. Small tables don't produce interesting plans — Postgres will seq scan a 100-row table no matter what indexes you build, because a seq scan on 100 rows is genuinely cheaper than the overhead of an index probe. So the first exercise is to build something big enough that the planner's cost model has to make a real choice.

```sql
CREATE TABLE orders (
    id          bigserial PRIMARY KEY,
    customer_id integer NOT NULL,
    status      text NOT NULL,
    amount      numeric(10,2) NOT NULL,
    notes       text,
    created_at  timestamptz NOT NULL DEFAULT now()
);

-- 2 million rows, skewed so 'status' has a few common values
-- and customer_id has high cardinality.
INSERT INTO orders (customer_id, status, amount, notes, created_at)
SELECT
    (random() * 50000)::int,
    (ARRAY['pending','shipped','delivered','cancelled'])[1 + floor(random()*4)],
    (random() * 500)::numeric(10,2),
    'order note ' || g,
    now() - (random() * interval '365 days')
FROM generate_series(1, 2_000_000) AS g;

ANALYZE orders;
```

Keep this table around — every "Real Scenario" exercise in this file reuses it.

**[Back to top](#table-of-contents)**

---

## Beginner: Reading EXPLAIN and EXPLAIN ANALYZE

`EXPLAIN` shows you the plan Postgres *intends* to run, with estimated costs. `EXPLAIN ANALYZE` actually runs the query and shows real timings and row counts alongside the estimates. The difference matters: a plan can look reasonable and still be wrong, and you only find out by comparing estimated vs. actual rows.

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 12345;
```

```
Seq Scan on orders  (cost=0.00..38334.00 rows=40 width=64)
  Filter: (customer_id = 12345)
```

Read this right to left, inside out: Postgres estimates a `Seq Scan` will cost `38334.00` "cost units" (an arbitrary unit calibrated to `seq_page_cost = 1.0`), and expects to return about 40 rows. No query ran yet — these are estimates from table statistics.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 12345;
```

```
Seq Scan on orders (cost=0.00..38334.00 rows=40 width=64)
                    (actual time=0.021..245.918 rows=38 loops=1)
  Filter: (customer_id = 12345)
  Rows Removed by Filter: 1999962
  Buffers: shared hit=1245 read=16941
Planning Time: 0.112 ms
Execution Time: 246.012 ms
```

Now you get `actual time` (startup..total, in milliseconds), `actual rows`, and — as of PostgreSQL 18 — `Buffers` is reported automatically whenever `ANALYZE` is used, without needing to ask for it separately, since `ANALYZE` now implicitly enables `BUFFERS`. On older versions you must add `BUFFERS` explicitly. `Rows Removed by Filter` tells you the seq scan read every row and threw away all but 38 of them — a strong hint an index would help here. See the official [EXPLAIN reference](https://www.postgresql.org/docs/current/sql-explain.html) and the narrative walkthrough in [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html).

**Exercise:** Run the query above on your `orders` table before creating any index, and confirm you get a seq scan with a `Rows Removed by Filter` line in the millions.

**[Back to top](#table-of-contents)**

---

## Working Knowledge: The Cost Model and Choosing an Index Type

Postgres is a **cost-based** optimizer, not a rule-based one — it doesn't have a hardcoded rule like "always use an index if one exists." Instead it estimates a cost for every plan it can construct (seq scan, index scan, bitmap heap scan, various join orders) using configurable per-operation costs (`seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, etc.) and picks the cheapest. This is why the *same query* can get different plans on different tables, or after a data-distribution shift — the estimates changed, not the rules.

Create an index and rerun the query:

```sql
CREATE INDEX idx_orders_customer_id ON orders (customer_id);

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 12345;
```

```
Index Scan using idx_orders_customer_id on orders
  (cost=0.42..8.85 rows=40 width=64)
  (actual time=0.031..0.058 rows=38 loops=1)
  Index Cond: (customer_id = 12345)
  Buffers: shared hit=6 read=2
```

The planner flipped from seq scan to index scan because the *estimated* cost dropped from ~38000 to ~9 — not because "an index exists." An **Index Scan** walks the index and fetches each matching heap tuple individually (good for small result sets); a **Bitmap Heap Scan** instead builds an in-memory bitmap of matching heap pages from the index, sorts it by physical page order, then reads those pages sequentially — worthwhile when moderately many rows match, because it avoids random I/O per row:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'cancelled';
```

```
Bitmap Heap Scan on orders  (cost=... rows=500000 ...)
  Recheck Cond: (status = 'cancelled')
  ->  Bitmap Index Scan on idx_orders_status  (cost=... rows=500000 ...)
        Index Cond: (status = 'cancelled')
```

```
             Bitmap Index Scan              Bitmap Heap Scan
          (builds page bitmap)          (reads pages in physical order)
    idx: [pg3][pg7][pg7][pg9][pg3]  ->   heap: pg3, pg3, pg7, pg7, pg9
                  |                              |
          collapses to a compact         one sequential-ish pass,
          set of matching pages          not N random lookups
```

When two indexes each match part of a `WHERE`, Postgres can combine their bitmaps with `BitmapAnd`/`BitmapOr` before touching the heap — this is the mechanism, not a special "multi-index scan" node.

Choosing an index type is a separate decision from *whether* to index:

- **B-tree** (the default): equality and range comparisons (`=, <, >, BETWEEN, IN`, sorting, `LIKE 'prefix%'`). Reach for this unless you have a specific reason not to.
- **Hash**: only equality (`=`). Since Postgres 10, hash indexes are WAL-logged and crash-safe, but they're rarely a win over B-tree since B-tree handles equality fine and also supports ranges/sorting — see the [Index Types docs](https://www.postgresql.org/docs/current/indexes-types.html).
- **GIN** (Generalized Inverted Index): composite/multi-valued columns — `jsonb`, arrays, full-text `tsvector`. Fast lookups, slower/costlier writes (each value inside the composite item gets its own index entry).
- **GiST**: geometric types, full-text search, range types, and nearest-neighbor (`<->`) queries — anything needing a "lossy," extensible notion of overlap or distance rather than strict equality.
- **BRIN** (Block Range Index): tiny index for huge, naturally-ordered tables (e.g., a `created_at` column that correlates with physical insert order). Stores only min/max per block range, so it's minuscule but only useful when physical and logical order correlate.

**Real Scenario:** On `orders`, compare a B-tree on `created_at` against a BRIN, given that rows were inserted in roughly increasing time order:

```sql
CREATE INDEX idx_orders_created_btree ON orders (created_at);
CREATE INDEX idx_orders_created_brin   ON orders USING brin (created_at);

SELECT pg_size_pretty(pg_relation_size('idx_orders_created_btree')),
       pg_size_pretty(pg_relation_size('idx_orders_created_brin'));
```

You should see the BRIN index is a tiny fraction of the B-tree's size, at the cost of being less precise (it can only narrow a scan down to block ranges, then relies on a `Recheck`).

**[Back to top](#table-of-contents)**

---

## Working Knowledge: Statistics, ANALYZE, and Autovacuum

Every cost estimate above depends on table statistics — row count, most-common values, histogram of value distribution, correlation between physical and logical order. These live in `pg_statistic` (`pg_stats` is the readable view) and are only as fresh as the last `ANALYZE`.

```sql
SELECT attname, n_distinct, most_common_vals, correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

`autovacuum` is the background process that runs `ANALYZE` (and `VACUUM`) automatically once enough rows have changed (governed by `autovacuum_analyze_scale_factor` / `autovacuum_analyze_threshold`). If you bulk-load or bulk-delete data and then immediately query, you can hit stale statistics before autovacuum catches up — this is one of the most common causes of a surprising plan right after a big data load. Deep coverage of `VACUUM`/bloat/MVCC internals lives in [postgres-transactions-concurrency.md](./postgres-transactions-concurrency.md); here, just know that stale stats → bad row estimates → bad plan choice, and running `ANALYZE` manually after a bulk operation is cheap insurance. See [Planner Statistics](https://www.postgresql.org/docs/current/planner-stats.html) and [Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html).

**Real Scenario:** Skew the table further and watch the plan react to fresh vs. stale stats.

```sql
UPDATE orders SET status = 'delivered' WHERE id % 10 = 0;  -- big change, no ANALYZE yet
EXPLAIN SELECT * FROM orders WHERE status = 'delivered';   -- stats still reflect old distribution
ANALYZE orders;
EXPLAIN SELECT * FROM orders WHERE status = 'delivered';   -- estimate should shift
```

**[Back to top](#table-of-contents)**

---

## Advanced: Why the Planner Didn't Use Your Index

An index existing is not sufficient — the query has to be written so the planner can *prove* the index applies. The three classic misfires:

**1. Implicit type cast.** Comparing a `text` column to something Postgres has to cast changes the operator class involved, and the index may not match:

```sql
-- customer_id is integer; comparing to a text literal forces a cast
-- WRONG: can silently prevent index use depending on column types involved
EXPLAIN SELECT * FROM orders WHERE customer_id = '12345';  -- often fine for int, but...

-- classic real failure: indexing a numeric column, then comparing
-- against a differently-typed parameter from an ORM, e.g. bigint vs numeric
```

The safer habit: keep literal/parameter types matching the column type exactly, and check `\d orders` for the real column type before assuming a mismatch isn't happening.

**2. Leading wildcard `LIKE`.** A standard B-tree index supports `LIKE 'prefix%'` but cannot support `LIKE '%suffix'`, because a B-tree is ordered and a leading wildcard gives it no prefix to seek on:

```sql
CREATE INDEX idx_orders_notes ON orders (notes);

-- WRONG: leading wildcard can't use a plain B-tree index
EXPLAIN SELECT * FROM orders WHERE notes LIKE '%999';

-- RIGHT: trailing wildcard can use the B-tree
EXPLAIN SELECT * FROM orders WHERE notes LIKE 'order note 999%';
```

For genuine substring/suffix search, use a trigram index instead: `CREATE EXTENSION pg_trgm;` then `CREATE INDEX ... USING gin (notes gin_trgm_ops);` — see the [pg_trgm docs](https://www.postgresql.org/docs/current/pgtrgm.html) and the well-worn Stack Overflow thread [PostgreSQL LIKE query performance variations](https://stackoverflow.com/questions/1566717/postgresql-like-query-performance-variations).

**3. Function call on an indexed column.** Wrapping the column in a function hides it from a plain index on that column, because the index stores raw column values, not `lower(column)`:

```sql
-- WRONG: lower(status) on every row means the plain index on status can't be used
EXPLAIN SELECT * FROM orders WHERE lower(status) = 'cancelled';

-- RIGHT: index the expression itself (see next section)
CREATE INDEX idx_orders_status_lower ON orders (lower(status));
EXPLAIN SELECT * FROM orders WHERE lower(status) = 'cancelled';
```

This general class of gotcha — "why isn't Postgres using my index" — is one of the most-asked Postgres questions on Stack Overflow; see [Why does PostgreSQL not use an index?](https://stackoverflow.com/questions/tagged/query-optimization+postgresql) for a running catalog of cases, and always check `EXPLAIN` output for `Filter:` (post-fetch, index not helping with this condition) vs. `Index Cond:` (the index actually narrowed the scan).

**[Back to top](#table-of-contents)**

---

## Advanced: Partial and Expression Indexes

A **partial index** only indexes rows matching a `WHERE` clause — smaller, cheaper to maintain, and often a better fit than a full index when queries only ever care about a slice of the table (e.g., "active" rows, or a rare status):

```sql
-- Most queries only care about non-cancelled, non-delivered orders
CREATE INDEX idx_orders_open ON orders (customer_id)
WHERE status IN ('pending', 'shipped');

EXPLAIN SELECT * FROM orders
WHERE customer_id = 999 AND status IN ('pending', 'shipped');
```

An **expression index** indexes the *result* of a function or expression rather than a raw column, which is exactly the fix for the `lower(status)` misfire above, or for computed values like `amount * 1.1`:

```sql
CREATE INDEX idx_orders_amount_with_tax ON orders ((amount * 1.1));

EXPLAIN SELECT * FROM orders WHERE (amount * 1.1) > 100;
```

The expression in the query must match the indexed expression exactly (modulo some normalization) for the planner to recognize it — see [Indexes on Expressions](https://www.postgresql.org/docs/current/indexes-expressional.html) and [Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html).

A note on index bloat: partial and expression indexes still bloat under heavy update/delete churn like any B-tree, since dead index entries aren't reclaimed until vacuumed — the mechanics of that belong to [postgres-transactions-concurrency.md](./postgres-transactions-concurrency.md), but `REINDEX` (or `REINDEX CONCURRENTLY` to avoid locking writes) is the practical fix once `pg_stat_user_indexes`/`pgstattuple` shows an index has grown much larger than its live data would justify.

**[Back to top](#table-of-contents)**

---

## Advanced: Multicolumn Indexes and Index-Only Scans

A multicolumn B-tree index is ordered by its first column, then its second within each value of the first, and so on — like a phone book sorted by last name, then first name. This means, prior to PostgreSQL 18's skip scan (next section), a condition on only the *second* column of a two-column index generally couldn't use that index efficiently, because there's no way to seek to "all rows where col2 = X" without scanning every distinct col1 value.

```sql
CREATE INDEX idx_orders_cust_status ON orders (customer_id, status);

-- Uses the index efficiently: constrains the leading column
EXPLAIN SELECT * FROM orders WHERE customer_id = 12345 AND status = 'shipped';

-- Historically much weaker: no constraint on the leading column
EXPLAIN SELECT * FROM orders WHERE status = 'shipped';
```

An **index-only scan** answers a query entirely from the index, without touching the heap at all, when every column the query needs is present in the index:

```sql
-- covers customer_id and status entirely -> can be index-only
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id FROM orders WHERE customer_id = 12345 AND status = 'shipped';
```

Look for `Index Only Scan` in the plan (vs. `Index Scan`), and note the `Heap Fetches` counter in the `ANALYZE` output — a high `Heap Fetches` count means the visibility map (see Mastery below) isn't sparing you heap trips as much as expected. You can widen coverage with `INCLUDE` columns, which ride along in the index for output purposes without participating in the index's sort order or search predicates:

```sql
CREATE INDEX idx_orders_cust_status_incl ON orders (customer_id, status) INCLUDE (amount);
```

See [Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html) and [Index-Only Scans](https://www.postgresql.org/docs/current/indexes-index-only-scans.html).

**[Back to top](#table-of-contents)**

---

## Mastery: B-Tree Skip Scan (PostgreSQL 18)

PostgreSQL 18 introduces **skip scan** for multicolumn B-tree indexes, directly addressing the leading-column limitation described above. Before 18, a query constraining only a non-leading column of a composite index (e.g., `status` in `(customer_id, status)`) generally fell back to a seq scan or a full index scan of the whole index. As of 18, the planner can have the executor probe the index once per distinct value of the skipped leading column(s), effectively treating the index as if it were a union of narrower per-value indexes — cutting the portion of the index that has to be read even without an `=` condition on the prefix column ([PostgreSQL 18 release notes](https://www.postgresql.org/docs/release/18.0/); overview in [Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)).

```sql
-- On PG18+, this can now use idx_orders_cust_status via skip scan,
-- where on PG17 and earlier it would typically seq scan or index-scan the whole thing.
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'cancelled';
```

Look for `Index Cond` mentioning the skipped column together with an internal marker of repeated re-seeks (implementation detail: Postgres runs the scan as a series of targeted searches, one per distinct leading-column value it needs to visit). Whether skip scan actually beats a seq scan still depends on `customer_id`'s cardinality — with 50,000 distinct `customer_id` values, 50,000 re-seeks may cost more than one sequential pass, so this is a genuine cost-model decision, not an automatic win. This is a good moment to re-emphasize why Postgres is cost-based: skip scan is a new *plan shape* the planner can now consider, but it still only gets chosen when its estimated cost wins. See the community discussion and worked benchmarks in [PostgreSQL 18 B-tree Skip Scan for Better Queries](https://neon.com/postgresql/18/skip-scan-btree) and [Postgres 18: Skip Scan — Breaking Free from the Left-Most Index Limitation](https://www.pgedge.com/blog/postgres-18-skip-scan-breaking-free-from-the-left-most-index-limitation). If you've read older advice ("always put your most-selective equality column first, and never expect a composite index to help a query that skips it") — that advice is now only partially true on 18+: column order still matters for cost, but the hard "impossible without the leading column" limitation is gone.

**Exercise:** Drop and recreate `idx_orders_cust_status`, run both queries above, and compare the `Buffers` line between an equality-on-leading-column query and a skip-scan-eligible query to see the actual page-read difference on your data.

**[Back to top](#table-of-contents)**

---

## Mastery: Visibility Maps and the Mechanics of Index-Only Scans

Index-only scans work only when Postgres can prove a heap page's rows are all visible to every transaction, without checking row-level visibility info that lives in the heap (MVCC details covered in [postgres-transactions-concurrency.md](./postgres-transactions-concurrency.md)). That proof comes from the **visibility map** (`VM`), a compact per-relation bitmap tracking which heap pages are "all-visible." `VACUUM` is what sets these bits; heavy write activity without enough vacuuming means the VM stays stale, and index-only scans fall back to real heap fetches (visible as a nonzero `Heap Fetches` in `EXPLAIN ANALYZE` output) even though the query shape looks index-only-eligible.

```sql
SELECT relname, pg_relation_size(oid, 'vm') AS vm_bytes
FROM pg_class WHERE relname = 'orders';
```

**Real Scenario:** Force some churn, then compare `Heap Fetches` before and after a `VACUUM`:

```sql
UPDATE orders SET amount = amount + 1 WHERE customer_id = 12345;
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id FROM orders WHERE customer_id = 12345 AND status = 'shipped';
-- note Heap Fetches

VACUUM orders;
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id FROM orders WHERE customer_id = 12345 AND status = 'shipped';
-- Heap Fetches should drop
```

See [Index-Only Scans and Visibility Map](https://www.postgresql.org/docs/current/indexes-index-only-scans.html) and [Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html).

**[Back to top](#table-of-contents)**

---

## Mastery: Extended Statistics for Correlated Columns

By default, Postgres's planner assumes columns are statistically independent — it estimates the selectivity of `WHERE a = 1 AND b = 2` by multiplying the individual selectivities of each condition. When columns are correlated (e.g., `city` and `zip_code`, or in our table, if `status` correlated with some derived value of `customer_id`), this systematically mis-estimates row counts, sometimes by orders of magnitude, and drives the planner toward a worse join order or scan type.

`CREATE STATISTICS` lets you tell Postgres explicitly to track dependencies between columns:

```sql
CREATE STATISTICS orders_cust_status_stats (dependencies, ndistinct)
ON customer_id, status FROM orders;

ANALYZE orders;

EXPLAIN SELECT * FROM orders WHERE customer_id = 12345 AND status = 'shipped';
```

Compare the estimated `rows=` before and after creating the extended statistics object — with correlated columns, the post-stats estimate should move closer to the actual row count you'd see under `EXPLAIN ANALYZE`. See [CREATE STATISTICS](https://www.postgresql.org/docs/current/sql-createstatistics.html) and [Extended Statistics](https://www.postgresql.org/docs/current/planner-stats-extended.html) for the `mcv` (most-common-values combinations) kind as well, useful for skewed *combinations* of values rather than just pairwise dependence.

**[Back to top](#table-of-contents)**

---

## Mastery: The Genetic Query Optimizer

Join planning is combinatorial: for `n` tables, the number of possible join orders grows roughly factorially. For small joins, Postgres exhaustively evaluates the search space via dynamic programming. Past `geqo_threshold` (default 12 tables in a single query), it switches to the **Genetic Query Optimizer (GEQO)** — a randomized, heuristic search that doesn't guarantee the optimal plan but finds a good one in tractable time.

You won't hit this with the two-table `orders` example, but it's worth knowing the switchover exists: if you ever see a large multi-join query's plan quality vary slightly between runs, or fluctuate after unrelated schema changes, GEQO's randomized search is a likely explanation, and `geqo_seed`/`geqo_threshold` are the relevant knobs. See [Genetic Query Optimizer](https://www.postgresql.org/docs/current/geqo.html).

```sql
SHOW geqo_threshold;
SHOW geqo;
```

**[Back to top](#table-of-contents)**

---

## Cheat Sheet: Index Types Compared

| Index Type | Best For | Write Cost | Supports |
|---|---|---|---|
| B-tree | Equality, ranges, sorting, `LIKE 'prefix%'` | Low–moderate | `=`, `<`, `>`, `BETWEEN`, `IN`, `ORDER BY`, multicolumn, skip scan (PG18+) |
| Hash | Pure equality only | Low | `=` only; rarely beats B-tree in practice |
| GIN | `jsonb`, arrays, full-text (`tsvector`), multi-valued columns | High (many entries per row) | Containment (`@>`), full-text search, `pg_trgm` substring search |
| GiST | Geometric types, ranges, nearest-neighbor, full-text | Moderate | Overlap (`&&`), distance (`<->`), extensible operator classes |
| BRIN | Huge tables where column correlates with physical row order (e.g., timestamps) | Very low | Range queries on naturally-ordered columns; tiny footprint |

**[Back to top](#table-of-contents)**

---

← Back to [Postgres SQL Mastery Guide](./postgres-mastery-guide.md)
