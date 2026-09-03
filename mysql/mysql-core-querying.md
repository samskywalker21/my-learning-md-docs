# MySQL Core Querying

Part of the [MySQL Mastery Guide](./mysql-mastery-guide.md). Full Beginner → Mastery depth. Every example below runs against this shared schema — create it once and reuse it for the whole file:

```sql
CREATE TABLE customers (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  country VARCHAR(2) NOT NULL
) ENGINE=InnoDB;

CREATE TABLE orders (
  id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  customer_id BIGINT UNSIGNED NOT NULL,
  total DECIMAL(10,2) NOT NULL,
  status VARCHAR(20) NOT NULL,
  placed_at DATETIME NOT NULL,
  FOREIGN KEY (customer_id) REFERENCES customers(id)
) ENGINE=InnoDB;

INSERT INTO customers (name, country) VALUES
  ('Ada', 'US'), ('Grace', 'UK'), ('Alan', 'UK');

INSERT INTO orders (customer_id, total, status, placed_at) VALUES
  (1, 50.00, 'shipped', '2026-01-05 10:00:00'),
  (1, 20.00, 'shipped', '2026-02-01 09:00:00'),
  (2, 75.00, 'pending', '2026-02-10 14:30:00'),
  (3, 10.00, 'cancelled', '2026-02-11 08:00:00');
```

## Table of Contents

- [SELECT & Filtering](#select--filtering)
- [Joins](#joins)
- [Subqueries](#subqueries)
- [Common Table Expressions (CTEs)](#common-table-expressions-ctes)
- [Window Functions](#window-functions)
- [GROUP BY and ONLY_FULL_GROUP_BY](#group-by-and-only_full_group_by)
- [Set Operations](#set-operations)
- [Cheat Sheet](#cheat-sheet)

---

## SELECT & Filtering

**Beginner.**

```sql
SELECT name, country FROM customers WHERE country = 'UK' ORDER BY name LIMIT 10;
```

**Working Knowledge.** `LIMIT offset, count` is MySQL's own syntax (also available as standard `LIMIT count OFFSET offset`):

```sql
SELECT * FROM orders ORDER BY placed_at DESC LIMIT 20, 10;   -- skip 20, take 10
SELECT * FROM orders ORDER BY placed_at DESC LIMIT 10 OFFSET 20;  -- same thing, SQL-standard form
```

```sql
-- Wrong: LIMIT with no ORDER BY when you expect a stable "top N" — row
-- order without ORDER BY is not guaranteed, even if it looks stable today.
SELECT * FROM orders LIMIT 5;

-- Right: always pair LIMIT with ORDER BY when order matters
SELECT * FROM orders ORDER BY placed_at DESC LIMIT 5;
```

[Back to top](#mysql-core-querying)

---

## Joins

**Working Knowledge.** MySQL supports `INNER JOIN`, `LEFT JOIN`, and `CROSS JOIN` natively. There is **no `RIGHT JOIN`... wait, actually there is** — MySQL supports `RIGHT JOIN` — but it has **no `FULL OUTER JOIN`**, unlike Postgres. Emulate it with a `UNION` of a `LEFT JOIN` and a `RIGHT JOIN` (or a `LEFT JOIN` plus an anti-join `LEFT JOIN ... WHERE right.id IS NULL` unioned with the mirror).

```sql
SELECT c.name, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'shipped';
```

```sql
-- FULL OUTER JOIN emulation: customers with orders, plus customers with
-- none, plus (hypothetically) orders with no matching customer.
SELECT c.name, o.total FROM customers c LEFT JOIN orders o ON o.customer_id = c.id
UNION
SELECT c.name, o.total FROM customers c RIGHT JOIN orders o ON o.customer_id = c.id;
```

**Advanced.** MySQL's optimizer historically only supported **nested-loop joins** — no native hash join for arbitrary predicates until 8.0.18 added a **hash join** fallback for equi-joins that can't use an index. Confirm which strategy actually ran with `EXPLAIN FORMAT=TREE` (see [Query Engine & Indexing](./mysql-query-engine-indexing.md#reading-explain-and-explain-analyze)) rather than assuming — an unindexed join column on a large table used to mean a slow nested loop; today it may pick hash join, but only if no usable index exists at all.

**Real Scenario (try it yourself):** Run the `FULL OUTER JOIN` emulation above, then add a customer with no orders and an order with a `customer_id` that doesn't exist (temporarily drop the foreign key to allow it). Watch how the two halves of the `UNION` each contribute the row the other side misses — and notice `UNION` (not `UNION ALL`) also silently drops the row that both `LEFT` and `RIGHT` versions produce identically, which is exactly what you want here but would be a bug if the query had duplicate legitimate rows.

[Back to top](#mysql-core-querying)

---

## Subqueries

**Working Knowledge.**

```sql
SELECT name FROM customers
WHERE id IN (SELECT customer_id FROM orders WHERE status = 'cancelled');
```

**The classic `NOT IN` + `NULL` trap — identical to Postgres, and just as sharp here:**

```sql
-- Wrong: if any customer_id in the subquery is NULL, NOT IN returns
-- an empty result set for every row, because `x <> NULL` is UNKNOWN,
-- not TRUE, and ALL comparisons in the NOT IN must be TRUE.
SELECT name FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders WHERE status = 'cancelled');

-- Right: NOT EXISTS sidesteps NULL semantics entirely
SELECT c.name FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'cancelled'
);
```

[Back to top](#mysql-core-querying)

---

## Common Table Expressions (CTEs)

**Working Knowledge → Advanced.** Introduced in MySQL 8.0 — no CTE support at all in 5.7 and earlier, so if you're reading older material or migrating an old codebase, this is new territory, not a rename of an existing feature.

```sql
WITH shipped AS (
  SELECT customer_id, SUM(total) AS spent
  FROM orders WHERE status = 'shipped'
  GROUP BY customer_id
)
SELECT c.name, s.spent FROM customers c JOIN shipped s ON s.customer_id = c.id;
```

**Advanced.** Recursive CTEs, also 8.0+ (specifically 8.0.1):

```sql
WITH RECURSIVE counter (n) AS (
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM counter WHERE n < 5
)
SELECT n FROM counter;
```

MySQL's recursive CTEs don't have Postgres's `SEARCH`/`CYCLE` clauses — if you need cycle detection in a recursive traversal (e.g. a graph with potential loops), you have to track visited nodes manually, typically by accumulating a path string or JSON array and checking membership before recursing:

```sql
WITH RECURSIVE reports_to (id, manager_id, path) AS (
  SELECT id, manager_id, CAST(id AS CHAR(200)) FROM employees WHERE id = 1
  UNION ALL
  SELECT e.id, e.manager_id, CONCAT(r.path, ',', e.id)
  FROM employees e
  JOIN reports_to r ON e.manager_id = r.id
  WHERE FIND_IN_SET(e.id, r.path) = 0   -- manual cycle guard
)
SELECT * FROM reports_to;
```

`cte_max_recursion_depth` (default 1000) caps recursion — a runaway recursive CTE errors out rather than hanging forever, but it's worth knowing the knob exists if you legitimately need deeper recursion. See the official [WITH (Common Table Expressions)](https://dev.mysql.com/doc/refman/8.4/en/with.html) reference.

[Back to top](#mysql-core-querying)

---

## Window Functions

**Working Knowledge → Advanced.** Also new in MySQL 8.0 — a major gap versus Postgres (which has had them since 8.4, over a decade earlier). Syntax is standard SQL and close to identical to Postgres's:

```sql
SELECT
  customer_id,
  total,
  SUM(total) OVER (PARTITION BY customer_id ORDER BY placed_at) AS running_total,
  LAG(total) OVER (PARTITION BY customer_id ORDER BY placed_at) AS prev_order_total
FROM orders;
```

**Gotcha:** `LAG()`/`LEAD()` and the other pure "offset" functions (`NTILE`, `NTH_VALUE`, `RANK`, etc.) **ignore any explicit frame clause** — they always operate over the whole partition, per the [official window function reference](https://dev.mysql.com/doc/refman/8.4/en/window-function-descriptions.html). Writing `LAG(total) OVER (PARTITION BY customer_id ORDER BY placed_at ROWS BETWEEN 1 PRECEDING AND CURRENT ROW)` isn't wrong, exactly — it's just silently ignored, which reads as intentional to a reviewer and can hide a misunderstanding.

[Back to top](#mysql-core-querying)

---

## GROUP BY and ONLY_FULL_GROUP_BY

**Working Knowledge.** `ONLY_FULL_GROUP_BY` is on by default (since 5.7.5) and is the single most common source of "this worked on my old MySQL install" confusion. It rejects any non-aggregated, non-grouped column in the `SELECT` list *unless* MySQL can prove it's functionally dependent on the `GROUP BY` columns (e.g. it's the primary key, or a unique NOT NULL column, or tied by an equality join).

```sql
-- Wrong under ONLY_FULL_GROUP_BY (the default): country isn't grouped
-- or aggregated, and isn't functionally dependent on customer_id.
SELECT customer_id, country, SUM(total)
FROM orders JOIN customers ON customers.id = orders.customer_id
GROUP BY customer_id;
-- ERROR 1055: 'mydb.customers.country' isn't in GROUP BY

-- Right: group by the column too, or aggregate it, or rely on a proven
-- functional dependency (grouping by a table's own primary key makes
-- every other column in that same table selectable)
SELECT customer_id, country, SUM(total)
FROM orders JOIN customers ON customers.id = orders.customer_id
GROUP BY customer_id, country;
```

If you deliberately want "any value from the group, I don't care which," use `ANY_VALUE()` rather than disabling the mode globally — disabling `ONLY_FULL_GROUP_BY` silently reintroduces nondeterministic column values across your whole application, not just the one query where you meant it:

```sql
SELECT customer_id, ANY_VALUE(country), SUM(total)
FROM orders JOIN customers ON customers.id = orders.customer_id
GROUP BY customer_id;
```

**Advanced.** `WITH ROLLUP` is MySQL's subtotal/grand-total mechanism — the syntax and power set differ from Postgres's `GROUPING SETS`/`CUBE`/`ROLLUP` trio (MySQL 8.4 only has `ROLLUP`, not the other two):

```sql
SELECT customer_id, status, SUM(total)
FROM orders
GROUP BY customer_id, status WITH ROLLUP;
-- Adds a subtotal row per customer_id (status = NULL) and a grand total
-- row (customer_id = NULL, status = NULL).
```

Use `GROUPING()` to distinguish a real `NULL` from a rollup-generated summary row when a grouped column can itself legitimately be `NULL`.

[Back to top](#mysql-core-querying)

---

## Set Operations

**Working Knowledge.** `UNION` and `UNION ALL` have always existed; `INTERSECT` and `EXCEPT` are new as of **MySQL 8.0.31** — if you're on an earlier 8.0 patch release, they don't exist yet and you'll need the classic `WHERE ... IN (subquery)` / `WHERE NOT EXISTS` rewrites instead.

```sql
-- Customers who have both a shipped and a pending order
SELECT customer_id FROM orders WHERE status = 'shipped'
INTERSECT
SELECT customer_id FROM orders WHERE status = 'pending';

-- Customers who have a shipped order but never a cancelled one
SELECT customer_id FROM orders WHERE status = 'shipped'
EXCEPT
SELECT customer_id FROM orders WHERE status = 'cancelled';
```

```sql
-- Pre-8.0.31 equivalent, still worth knowing since it also works on
-- current versions and is sometimes clearer to a reader:
SELECT DISTINCT customer_id FROM orders o1
WHERE status = 'shipped'
  AND NOT EXISTS (SELECT 1 FROM orders o2 WHERE o2.customer_id = o1.customer_id AND o2.status = 'cancelled');
```

[Back to top](#mysql-core-querying)

---

## Cheat Sheet

| Need | Syntax |
|---|---|
| Top N with offset | `ORDER BY col LIMIT offset, count` or `LIMIT count OFFSET offset` |
| Missing-row-safe exclusion | `WHERE NOT EXISTS (...)`, never bare `NOT IN` on a nullable subquery |
| Full outer join | `LEFT JOIN ... UNION RIGHT JOIN ...` (no native `FULL OUTER JOIN`) |
| Recursive query | `WITH RECURSIVE name (...) AS (...)` — 8.0+, no `SEARCH`/`CYCLE`, guard cycles manually |
| Running total | `SUM(x) OVER (PARTITION BY ... ORDER BY ...)` — 8.0+ |
| Non-grouped column that's safe to pick any value from | `ANY_VALUE(col)` |
| Subtotals + grand total | `GROUP BY a, b WITH ROLLUP` |
| Set intersection/difference | `INTERSECT` / `EXCEPT` — 8.0.31+ only |

[Back to top](#mysql-core-querying)
