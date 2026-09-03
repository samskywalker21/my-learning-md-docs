# Postgres Core Querying — SELECT, Joins, Subqueries, CTEs & Window Functions

Part of the [Postgres SQL Mastery Guide](./postgres-mastery-guide.md).

This file is the heart of the series: everything about *reading* data from Postgres — from a plain `SELECT` up through recursive CTEs and window function frame clauses. It does not cover `EXPLAIN`, the query planner, or indexing — see [Query Engine & Indexing](./postgres-query-engine-indexing.md) for why a particular plan gets chosen, or why a query that reads fine here runs slowly at scale.

All examples target **PostgreSQL 18** (18.6 as of Aug 2026). Anywhere behavior changed recently or a common blog convention is now outdated, it's flagged explicitly.

## Table of Contents

- [Setup: Sample Schema](#setup-sample-schema)
- [SELECT Fundamentals & Filtering](#select-fundamentals--filtering)
  - [Beginner](#select-beginner)
  - [Working Knowledge](#select-working-knowledge)
  - [Advanced](#select-advanced)
  - [Mastery](#select-mastery)
- [Joins](#joins)
  - [Beginner](#joins-beginner)
  - [Working Knowledge](#joins-working-knowledge)
  - [Advanced](#joins-advanced)
  - [Mastery](#joins-mastery)
- [Subqueries](#subqueries)
  - [Beginner](#subqueries-beginner)
  - [Working Knowledge](#subqueries-working-knowledge)
  - [Advanced](#subqueries-advanced)
  - [Mastery](#subqueries-mastery)
- [Common Table Expressions (CTEs)](#common-table-expressions-ctes)
  - [Beginner](#ctes-beginner)
  - [Working Knowledge](#ctes-working-knowledge)
  - [Advanced](#ctes-advanced)
  - [Mastery](#ctes-mastery)
- [Window Functions](#window-functions)
  - [Beginner](#window-beginner)
  - [Working Knowledge](#window-working-knowledge)
  - [Advanced](#window-advanced)
  - [Mastery](#window-mastery)
- [Aggregates, GROUP BY & Grouping Sets](#aggregates-group-by--grouping-sets)
  - [Beginner](#agg-beginner)
  - [Working Knowledge](#agg-working-knowledge)
  - [Advanced](#agg-advanced)
  - [Mastery](#agg-mastery)
- [Set Operations (UNION, INTERSECT, EXCEPT)](#set-operations-union-intersect-except)
  - [Beginner](#set-beginner)
  - [Working Knowledge](#set-working-knowledge)
  - [Advanced](#set-advanced)
  - [Mastery](#set-mastery)
- [Quick-Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Setup: Sample Schema

Every example below reuses one small e-commerce-plus-org-chart schema, so patterns build on each other instead of resetting per section. Run this once in a scratch database (`createdb pg_core_querying`) and follow along.

```sql
CREATE TABLE customers (
    customer_id   serial PRIMARY KEY,
    name          text NOT NULL,
    country       text NOT NULL,
    signup_date   date NOT NULL
);

CREATE TABLE products (
    product_id    serial PRIMARY KEY,
    name          text NOT NULL,
    category      text NOT NULL,
    price         numeric(10,2) NOT NULL
);

CREATE TABLE orders (
    order_id      serial PRIMARY KEY,
    customer_id   integer REFERENCES customers(customer_id),
    order_date    date NOT NULL,
    status        text NOT NULL DEFAULT 'pending',
    total_amount  numeric(10,2)
);

CREATE TABLE order_items (
    order_item_id serial PRIMARY KEY,
    order_id      integer REFERENCES orders(order_id),
    product_id    integer REFERENCES products(product_id),
    quantity      integer NOT NULL,
    unit_price    numeric(10,2) NOT NULL
);

-- For self-joins and recursive CTEs later
CREATE TABLE employees (
    employee_id   serial PRIMARY KEY,
    name          text NOT NULL,
    manager_id    integer REFERENCES employees(employee_id),
    department    text NOT NULL,
    salary        numeric(10,2) NOT NULL,
    hire_date     date NOT NULL
);

INSERT INTO customers (name, country, signup_date) VALUES
    ('Amara Okafor',   'NG', '2023-01-15'),
    ('Ben Sato',       'JP', '2023-03-02'),
    ('Carla Diaz',     'MX', '2023-03-20'),
    ('Deepa Rao',      'IN', '2024-01-10'),
    ('Erik Lindqvist', 'SE', '2024-02-01');

INSERT INTO products (name, category, price) VALUES
    ('Wireless Mouse',  'Electronics', 25.00),
    ('Mechanical KB',   'Electronics', 89.00),
    ('Standing Desk',   'Furniture',   349.00),
    ('Desk Lamp',       'Furniture',   42.00),
    ('Notebook Set',    'Office',      12.00);

INSERT INTO orders (customer_id, order_date, status, total_amount) VALUES
    (1, '2024-05-01', 'completed', 114.00),
    (1, '2024-06-14', 'completed', 349.00),
    (2, '2024-05-20', 'completed', 42.00),
    (3, '2024-06-01', 'cancelled', 89.00),
    (4, '2024-06-10', 'completed', 12.00),
    (NULL, '2024-06-11', 'completed', 25.00); -- guest checkout, no customer row

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1, 25.00), (1, 2, 1, 89.00),
    (2, 3, 1, 349.00),
    (3, 4, 1, 42.00),
    (4, 2, 1, 89.00),
    (5, 5, 1, 12.00),
    (6, 1, 1, 25.00);

INSERT INTO employees (name, manager_id, department, salary, hire_date) VALUES
    ('Grace Lin',   NULL, 'Executive', 220000, '2015-01-01'),
    ('Hassan Ali',  1,    'Engineering', 175000, '2017-03-01'),
    ('Ivy Chen',    1,    'Sales',       150000, '2018-06-15'),
    ('Jamal Idris', 2,    'Engineering', 135000, '2019-09-01'),
    ('Kira Novak',  2,    'Engineering', 128000, '2021-02-10'),
    ('Liam O''Brien', 3, 'Sales',       98000,  '2022-05-20');
```

That last "guest checkout" row (`customer_id IS NULL`) and the cancelled order are deliberate — several gotchas below only show up once NULLs and non-`completed` rows exist in the data.

[↑ back to top](#table-of-contents)

---

## SELECT Fundamentals & Filtering

<a id="select-beginner"></a>

### Beginner

**What it is.** `SELECT` reads rows; `WHERE` filters them before they're returned. Postgres does not guarantee row order unless you add `ORDER BY` — this trips up almost everyone coming from a tool that always looks ordered by insertion.

```sql
SELECT name, country FROM customers;

SELECT name, country
FROM customers
WHERE country = 'JP';

SELECT name, country, signup_date
FROM customers
ORDER BY signup_date DESC
LIMIT 3;
```

**Try It:**

```sql
-- Only customers who signed up in 2024, newest first
SELECT name, signup_date
FROM customers
WHERE signup_date >= '2024-01-01'
ORDER BY signup_date DESC;
```

[↑ back to top](#table-of-contents)

<a id="select-working-knowledge"></a>

### Working Knowledge

**Logical query processing order.** Postgres doesn't execute a `SELECT` in the order you type it — it evaluates clauses in a fixed logical sequence. This is why you can't reference a `SELECT`-list alias in `WHERE`, but you *can* in `ORDER BY`:

```
FROM  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT (+ window fns)  →  DISTINCT  →  ORDER BY  →  LIMIT/OFFSET
```

```sql
-- WRONG — "recent" doesn't exist yet at the point WHERE is evaluated
SELECT order_date, (order_date > '2024-06-01') AS recent
FROM orders
WHERE recent;  -- ERROR: column "recent" does not exist

-- RIGHT — repeat the expression, or move it to a subquery/CTE
SELECT order_date, (order_date > '2024-06-01') AS recent
FROM orders
WHERE order_date > '2024-06-01';

-- RIGHT — ORDER BY runs AFTER the SELECT list, so aliases work there
SELECT order_date, (order_date > '2024-06-01') AS recent
FROM orders
ORDER BY recent DESC;
```

**Filtering patterns:**

```sql
SELECT name FROM products WHERE category IN ('Electronics', 'Furniture');
SELECT name FROM products WHERE price BETWEEN 20 AND 100;
SELECT name FROM customers WHERE name LIKE 'C%';       -- case-sensitive
SELECT name FROM customers WHERE name ILIKE 'c%';      -- case-insensitive
SELECT * FROM orders WHERE customer_id IS NULL;        -- NULL needs IS, not =
```

**Real Scenario — the `= NULL` trap:** try `SELECT * FROM orders WHERE customer_id = NULL;` and compare it to `SELECT * FROM orders WHERE customer_id IS NULL;`. The first returns **zero rows**, silently, no error — `NULL = NULL` evaluates to `NULL` (unknown), not `true`, and `WHERE` only keeps rows where the condition is `true`. This single fact — NULL never equals anything, including itself — is the root cause of most NULL-related bugs in this file, including the `NOT IN` trap in [Subqueries](#subqueries-advanced).

[↑ back to top](#table-of-contents)

<a id="select-advanced"></a>

### Advanced

**`DISTINCT` and `DISTINCT ON`.** `DISTINCT` dedupes whole rows. Postgres's `DISTINCT ON (...)` — a non-standard extension — keeps the *first* row per group, according to `ORDER BY`:

```sql
-- One row per customer: their most recent order
SELECT DISTINCT ON (customer_id) customer_id, order_date, total_amount
FROM orders
WHERE customer_id IS NOT NULL
ORDER BY customer_id, order_date DESC;
```

`DISTINCT ON` **requires** the `ORDER BY` to start with the same expression(s) it lists, because "first row per group" only means something once you've defined an order. Compare this to the window-function equivalent (`ROW_NUMBER() OVER (PARTITION BY ...)`) in [Window Functions](#window-mastery) — same result, portable syntax, and composable with more filtering afterward.

**`FILTER` clause for conditional aggregates** (standard SQL, cleaner than `CASE WHEN ... THEN x END` inside `COUNT`):

```sql
SELECT
    count(*) FILTER (WHERE status = 'completed') AS completed_orders,
    count(*) FILTER (WHERE status = 'cancelled') AS cancelled_orders,
    count(*) AS total_orders
FROM orders;
```

**`COALESCE`, `NULLIF`, and short-circuiting that isn't:**

```sql
SELECT name, COALESCE(country, 'unknown') AS country FROM customers;

-- NULLIF(a, b) returns NULL if a = b, else a — handy for avoiding div-by-zero
SELECT total_amount / NULLIF(0, 0) AS would_be_error; -- returns NULL, not an error
```

Note: Postgres does **not** guarantee left-to-right short-circuit evaluation of `AND`/`OR` the way a general-purpose language does — the planner may reorder or reconsider expressions. Don't rely on `WHERE x <> 0 AND y / x > 1` to protect against division by zero; use `CASE` or `NULLIF` instead. See the [official docs on expression evaluation](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-EXPRESS-EVAL) — this is one of the most common "worked in my other database" surprises.

[↑ back to top](#table-of-contents)

<a id="select-mastery"></a>

### Mastery

**`SELECT DISTINCT` reordering (Postgres 18).** As of Postgres 18, the planner is allowed to internally reorder the columns in `SELECT DISTINCT` to avoid an explicit sort step when a cheaper path exists (e.g., a matching index already provides one of the orderings). This is purely an execution-plan optimization — result rows are unaffected — and can be disabled via [`enable_distinct_reordering`](https://www.postgresql.org/docs/current/runtime-config-query.html#GUC-ENABLE-DISTINCT-REORDERING) if you ever need to rule it out while debugging a plan. See [Query Engine & Indexing](./postgres-query-engine-indexing.md) for how to read the resulting plan.

**Row and array constructors / `IS DISTINCT FROM`:**

```sql
-- IS DISTINCT FROM treats NULL as a comparable value: NULL IS DISTINCT FROM NULL → false
SELECT * FROM customers c
WHERE c.country IS DISTINCT FROM 'JP';   -- includes rows where country IS NULL, unlike <>

-- Row constructor comparison — compares element-wise, left to right
SELECT * FROM orders
WHERE (order_date, total_amount) > ('2024-06-01', 0);
```

`IS [NOT] DISTINCT FROM` is the "NULL-safe" equality operator — reach for it whenever a comparison needs to treat two NULLs as equal to each other, which plain `=` and `<>` never do. It's also exactly what powers `DISTINCT`/`UNION`'s own duplicate-row logic, covered in [Set Operations](#set-advanced).

[↑ back to top](#table-of-contents)

---

## Joins

<a id="joins-beginner"></a>

### Beginner

**What it is.** A join combines rows from two tables based on a matching condition. `INNER JOIN` (the default kind of `JOIN`) keeps only rows with a match on both sides.

```sql
SELECT o.order_id, c.name, o.total_amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
```

Notice this returns **5** rows, not 6 — the guest-checkout order (`customer_id IS NULL`) has no matching customer, so an inner join silently drops it. That's expected inner-join behavior, and exactly the case `LEFT JOIN` exists for.

[↑ back to top](#table-of-contents)

<a id="joins-working-knowledge"></a>

### Working Knowledge

**LEFT / RIGHT / FULL JOIN** — keep unmatched rows from one or both sides, filling the other side's columns with `NULL`:

```sql
-- Every order, with customer info where it exists (NULL for the guest order)
SELECT o.order_id, c.name, o.total_amount
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id;

-- Every customer, even those with zero orders (none here, but the pattern matters)
SELECT c.name, o.order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id;

-- RIGHT JOIN is just LEFT JOIN with sides swapped — rarely used because
-- swapping FROM/JOIN order and using LEFT JOIN reads more naturally
SELECT o.order_id, c.name
FROM customers c
RIGHT JOIN orders o ON o.customer_id = c.customer_id;

-- FULL JOIN: keep unmatched rows from BOTH sides
SELECT c.name, o.order_id
FROM customers c
FULL JOIN orders o ON o.customer_id = c.customer_id;
```

**Wrong vs. right — filtering a LEFT JOIN in `WHERE` silently turns it into an INNER JOIN:**

```sql
-- WRONG — intent: "all customers, with completed order totals where they exist"
-- but WHERE runs AFTER the join and discards the unmatched (NULL) rows,
-- because o.status = 'completed' is never true when o.* is all NULL
SELECT c.name, o.total_amount
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.status = 'completed';

-- RIGHT — move the condition into the JOIN so it filters BEFORE the
-- outer join fills in NULLs, preserving unmatched left-side rows
SELECT c.name, o.total_amount
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id AND o.status = 'completed';
```

This is the single most common `LEFT JOIN` bug: a condition on the right-hand table that belongs in `ON`, written in `WHERE` instead. See this exact gotcha discussed on [Stack Overflow: "SQL left join vs multiple tables"](https://stackoverflow.com/questions/38549/difference-between-inner-and-outer-join) and the [official JOIN docs](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN).

[↑ back to top](#table-of-contents)

<a id="joins-advanced"></a>

### Advanced

**CROSS JOIN — every row paired with every row.** Useful deliberately (generating combinations); a bug almost everywhere else:

```sql
-- Deliberate: every product × every "size" label, to seed a variants table
SELECT p.name, s.size_label
FROM products p
CROSS JOIN (VALUES ('S'), ('M'), ('L')) AS s(size_label);

-- ACCIDENTAL cross join — a comma-join with a forgotten WHERE condition
-- produces a Cartesian product: 5 customers × 6 orders = 30 rows, most garbage
SELECT c.name, o.order_id
FROM customers c, orders o;   -- no join condition at all!
```

The old comma-join syntax (`FROM a, b WHERE a.id = b.a_id`) is still legal SQL, but it's exactly one missing `WHERE` clause away from an accidental Cartesian product with no error raised. Prefer explicit `JOIN ... ON` everywhere — the condition is required syntactically, so you cannot forget it silently.

**Self-joins** — joining a table to itself, typically to compare rows or walk one level of a hierarchy:

```sql
-- Each employee next to their manager's name
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id
ORDER BY e.employee_id;
```

Table aliases (`e`, `m`) are **required** here — without them, `employee_id` is ambiguous between the two references to the same table.

**Real Scenario:** find every pair of products in the same category, without pairing a product with itself or listing the same pair twice (A-B and B-A):

```sql
SELECT p1.name AS product_a, p2.name AS product_b, p1.category
FROM products p1
JOIN products p2 ON p1.category = p2.category AND p1.product_id < p2.product_id
ORDER BY p1.category, product_a;
```

The `<` instead of `<>` in the join condition is what eliminates both self-pairs and duplicate reverse pairs in one move — a pattern worth memorizing.

[↑ back to top](#table-of-contents)

<a id="joins-mastery"></a>

### Mastery

**LATERAL joins** — the right-hand side of the join can reference columns from the left-hand side, row by row. This is what makes "top-N per group" queries efficient and readable:

```sql
-- The 2 most recent orders per customer — impossible with a plain JOIN
-- (a plain JOIN + LIMIT only limits the WHOLE result, not per group)
SELECT c.name, o.order_id, o.order_date, o.total_amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT order_id, order_date, total_amount
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    LIMIT 2
) o ON true
ORDER BY c.name, o.order_date DESC;
```

Without `LATERAL`, the subquery in the join couldn't see `c.customer_id` at all — it would be evaluated once, independent of the outer row, and Postgres would reject the reference. `LATERAL` says "re-evaluate this subquery for every row from the left side," turning it into something closer to a correlated function call than a static join. `CROSS JOIN LATERAL` (or comma) drops rows where the lateral subquery is empty; `LEFT JOIN LATERAL ... ON true` keeps them with NULLs, mirroring the plain-join LEFT/INNER distinction.

**Why `LATERAL` beats a window function here:** `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC)` (see [Window Functions](#window-mastery)) computes over the *entire* table before filtering, and needs an outer `WHERE rn <= 2`. `LATERAL` with `LIMIT` inside lets the planner push the limit down per group — often the faster plan for a large table with an index on `(customer_id, order_date)`, because it can stop scanning each customer's orders early instead of ranking all of them. Confirm which plan actually runs faster for your data with `EXPLAIN ANALYZE` — see [Query Engine & Indexing](./postgres-query-engine-indexing.md).

**Self-join elimination (Postgres 18):** the planner can now automatically detect and remove some redundant self-joins (e.g., a table joined to itself on its primary key with no other purpose) without changing results. This is a planner optimization, not a syntax change — it means some previously-necessary-looking self-joins may now be free. Source: [PostgreSQL 18 Release Notes](https://www.postgresql.org/docs/release/18.0/).

[↑ back to top](#table-of-contents)

---

## Subqueries

<a id="subqueries-beginner"></a>

### Beginner

**Scalar subquery** — returns exactly one row, one column, usable anywhere a single value is expected:

```sql
SELECT name, price
FROM products
WHERE price > (SELECT avg(price) FROM products);
```

**Subquery in `FROM`** (a "derived table") — must have an alias:

```sql
SELECT category, avg_price
FROM (
    SELECT category, avg(price) AS avg_price
    FROM products
    GROUP BY category
) AS category_stats
WHERE avg_price > 30;
```

[↑ back to top](#table-of-contents)

<a id="subqueries-working-knowledge"></a>

### Working Knowledge

**`IN` and `EXISTS`** — both test membership, but structurally different: `IN` compares a value against a list; `EXISTS` only asks "does at least one matching row exist," ignoring the subquery's column values entirely.

```sql
-- IN: customers who have placed at least one order
SELECT name FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders WHERE customer_id IS NOT NULL);

-- EXISTS: same result, expressed as a correlated existence check
SELECT name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```

**Correlated subquery** — references a column from the outer query, so it conceptually re-runs once per outer row:

```sql
-- Each customer's most recent order date, computed inline
SELECT c.name,
       (SELECT max(o.order_date) FROM orders o WHERE o.customer_id = c.customer_id) AS last_order
FROM customers c;
```

[↑ back to top](#table-of-contents)

<a id="subqueries-advanced"></a>

### Advanced

**The `NOT IN` + NULL trap — the single most-cited Postgres subquery gotcha.**

```sql
-- WRONG — intent: "customers who never placed an order"
-- Because one order row has customer_id = NULL, the subquery's result set
-- is {1, 2, 3, 4, NULL}. "x NOT IN (1,2,3,4,NULL)" evaluates x <> NULL for
-- the last element, which is NULL (unknown) — not false — so the WHOLE
-- NOT IN expression becomes NULL for every row, and WHERE keeps NOTHING.
SELECT name FROM customers c
WHERE c.customer_id NOT IN (SELECT customer_id FROM orders);
-- returns ZERO rows, silently — no error, no warning

-- RIGHT — option 1: filter the NULL out of the subquery explicitly
SELECT name FROM customers c
WHERE c.customer_id NOT IN (
    SELECT customer_id FROM orders WHERE customer_id IS NOT NULL
);

-- RIGHT — option 2: use NOT EXISTS, which has no such trap because it
-- never compares values against the subquery's rows — it only asks
-- "is there a matching row," and NULL customer_id just never matches
SELECT name FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

**Rule of thumb:** prefer `NOT EXISTS` over `NOT IN` for "anti-join" queries whenever the subquery's column can contain a `NULL` — which, in practice, means almost always, unless the column has a `NOT NULL` constraint you've personally verified. This exact behavior is documented in the [Postgres `IN`/`NOT IN` operator docs](https://www.postgresql.org/docs/current/functions-subquery.html#FUNCTIONS-SUBQUERY-IN) and is one of the most frequently re-asked questions on [Stack Overflow](https://stackoverflow.com/questions/129077/how-to-write-not-equal-condition-in-sql-query-for-null-values) — the docs are unambiguous, but the "just use `NOT IN`" habit from other tools persists because plenty of teams simply never happen to have NULLs in that column yet.

**Correlated subquery performance trap.** A correlated subquery in the `SELECT` list, like the "last order" example above, is conceptually — and often literally — executed once per outer row. On a small `customers` table this is invisible; on a large one it can turn an O(n) scan into an O(n × m) nested loop. The planner *can* sometimes rewrite a correlated subquery into a join internally, but it isn't guaranteed to for every shape of query. Two ways out:

```sql
-- Rewritten as a LEFT JOIN to a pre-aggregated subquery — computed once, joined once
SELECT c.name, o.last_order
FROM customers c
LEFT JOIN (
    SELECT customer_id, max(order_date) AS last_order
    FROM orders
    GROUP BY customer_id
) o ON o.customer_id = c.customer_id;

-- Or, for "top-N per row" shapes a plain aggregate join can't express, use LATERAL
-- (see Joins → Mastery) — same correlation, but the planner treats it as a proper
-- join operator instead of an opaque per-row subquery.
```

Always confirm with `EXPLAIN ANALYZE` rather than assuming — see [Query Engine & Indexing](./postgres-query-engine-indexing.md) for reading the resulting plan (nested loop vs. hash join) and what index would help either form.

[↑ back to top](#table-of-contents)

<a id="subqueries-mastery"></a>

### Mastery

**`ANY`/`ALL` with subqueries** — generalize comparison operators over a subquery's result set, and don't share `NOT IN`'s NULL trap in the same way because you choose the operator explicitly:

```sql
-- Products priced higher than every "Office" category product
SELECT name, price FROM products
WHERE price > ALL (SELECT price FROM products WHERE category = 'Office');

-- Equivalent to IN, but reads naturally next to ALL
SELECT name FROM products
WHERE price = ANY (SELECT price FROM products WHERE category = 'Furniture');
```

**Subqueries vs. `LATERAL` vs. CTEs — when to reach for which:**

- A **scalar/`EXISTS` subquery** is right when you need one value or one true/false per outer row and the logic is simple enough to stay readable inline.
- A **`LATERAL` join** is right when the "subquery" needs to return multiple rows/columns per outer row (top-N per group, "most recent N of X").
- A **CTE** (next section) is right when the same derived result is referenced more than once, or when the logic is recursive.

**Subquery materialization note:** unlike some other databases, Postgres does not treat a subquery in `FROM` as an opaque, materialized black box by default — the planner can "flatten" (inline) many subqueries and push outer conditions down into them, which is usually a performance win and occasionally a source of surprising plans when a subquery relies on evaluation order. If you specifically need a subquery's result computed once and not re-planned inline, `MATERIALIZED` CTEs (next section) are the more explicit, documented tool for that — see [Query Engine & Indexing](./postgres-query-engine-indexing.md) for how to tell from a plan whether flattening happened.

[↑ back to top](#table-of-contents)

---

## Common Table Expressions (CTEs)

<a id="ctes-beginner"></a>

### Beginner

**What it is.** A CTE (`WITH name AS (...)`) names a subquery so it can be referenced like a table, mainly for readability — breaking a complex query into named, sequential steps.

```sql
WITH high_value_orders AS (
    SELECT order_id, customer_id, total_amount
    FROM orders
    WHERE total_amount > 50
)
SELECT c.name, h.total_amount
FROM high_value_orders h
JOIN customers c ON c.customer_id = h.customer_id;
```

[↑ back to top](#table-of-contents)

<a id="ctes-working-knowledge"></a>

### Working Knowledge

**Multiple CTEs, referencing earlier ones:**

```sql
WITH order_totals AS (
    SELECT customer_id, sum(total_amount) AS lifetime_value
    FROM orders
    WHERE customer_id IS NOT NULL
    GROUP BY customer_id
),
ranked_customers AS (
    SELECT customer_id, lifetime_value,
           rank() OVER (ORDER BY lifetime_value DESC) AS value_rank
    FROM order_totals
)
SELECT c.name, r.lifetime_value, r.value_rank
FROM ranked_customers r
JOIN customers c ON c.customer_id = r.customer_id
ORDER BY r.value_rank;
```

Each CTE is a step in a pipeline; later ones can reference any earlier one (but not later ones, and not themselves — that requires `RECURSIVE`, below).

[↑ back to top](#table-of-contents)

<a id="ctes-advanced"></a>

### Advanced

**Recursive CTEs** — the one case a CTE can reference itself, used for hierarchies and graphs. Structure: a non-recursive "anchor" term, `UNION [ALL]`, then a recursive term referencing the CTE's own name; Postgres stops once the recursive term returns zero new rows.

```sql
-- Full management chain for every employee (org chart, root to leaf)
WITH RECURSIVE org_chart AS (
    -- Anchor: start at the top (no manager)
    SELECT employee_id, name, manager_id, 1 AS depth,
           name::text AS chain
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive term: join the CTE's own output to the next level down
    SELECT e.employee_id, e.name, e.manager_id, oc.depth + 1,
           oc.chain || ' > ' || e.name
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT * FROM org_chart ORDER BY depth, name;
```

**Wrong vs. right — `UNION` vs. `UNION ALL` in a recursive CTE:**

```sql
-- WRONG (usually) — plain UNION deduplicates every intermediate row against
-- ALL previous rows on every iteration, which is expensive and, worse,
-- can silently under-count when two legitimately distinct paths produce
-- an identical row (e.g., same name/depth combination from different branches)
WITH RECURSIVE org_chart AS (
    SELECT employee_id, name, manager_id, 1 AS depth FROM employees WHERE manager_id IS NULL
    UNION
    SELECT e.employee_id, e.name, e.manager_id, oc.depth + 1
    FROM employees e JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT * FROM org_chart;

-- RIGHT — UNION ALL unless you specifically need row-level dedup, which is
-- both faster (no per-iteration comparison against the whole accumulated set)
-- and the form used in virtually every official example
```

**Real Scenario:** total headcount and salary cost under each manager, rolling up through the whole tree (not just direct reports):

```sql
WITH RECURSIVE reports AS (
    SELECT employee_id AS manager_id, employee_id AS report_id, salary
    FROM employees
    UNION ALL
    SELECT r.manager_id, e.employee_id, e.salary
    FROM reports r
    JOIN employees e ON e.manager_id = r.report_id
)
SELECT m.name AS manager, count(*) - 1 AS indirect_and_direct_reports, sum(r.salary) - m.salary AS team_salary_cost
FROM reports r
JOIN employees m ON m.employee_id = r.manager_id
GROUP BY m.employee_id, m.name, m.salary
ORDER BY team_salary_cost DESC;
```

Try running just the `reports` CTE alone first (`SELECT * FROM reports ORDER BY manager_id;`) to see the row-per-descendant shape before the aggregate collapses it — recursive CTEs are much easier to reason about once you've looked at their unaggregated output.

[↑ back to top](#table-of-contents)

<a id="ctes-mastery"></a>

### Mastery

**`SEARCH` and `CYCLE` clauses (standard SQL, supported since Postgres 14)** — recursive CTEs over a genuine graph (not a strict tree) can loop forever if there's a cycle (e.g., corrupted data where an employee is their own indirect manager). The standard `CYCLE` clause detects this declaratively instead of you hand-rolling a "visited" array:

```sql
WITH RECURSIVE org_chart AS (
    SELECT employee_id, name, manager_id
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.name, e.manager_id
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SEARCH DEPTH FIRST BY employee_id SET ordercol
CYCLE employee_id SET is_cycle USING path
SELECT * FROM org_chart ORDER BY ordercol;
```

`SEARCH ... SET ordercol` adds a column that gives a deterministic depth-first (or `BREADTH FIRST`) traversal order without you hand-coding the `chain` string trick from the Advanced example. `CYCLE ... SET is_cycle USING path` tracks visited rows internally and flags `is_cycle = true` instead of looping forever, the way a hand-rolled `id = ANY(visited_ids)` check would but without you maintaining the array yourself. See the [official recursive query docs](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE) for the full grammar — this is genuinely underused relative to how often people hand-roll a worse version of it.

**`MATERIALIZED` / `NOT MATERIALIZED` (Postgres 12+).** Before Postgres 12, a CTE was always an optimization fence — computed once, never inlined into the outer query, regardless of cost. Since 12, the planner may inline a non-recursive CTE referenced exactly once, *unless* you force one behavior explicitly:

```sql
WITH regional_stats AS MATERIALIZED (
    SELECT country, count(*) AS customer_count FROM customers GROUP BY country
)
SELECT * FROM regional_stats WHERE customer_count > 0;
```

Force `MATERIALIZED` when a CTE has a side-effecting statement (`INSERT`/`UPDATE`/`DELETE ... RETURNING`) that must run exactly once regardless of how many times it's referenced, or when you've confirmed via `EXPLAIN` that inlining produces a worse plan (e.g., an expensive CTE referenced inside a loop-like structure). Force `NOT MATERIALIZED` if you want to guarantee planner flattening even for something the heuristics might otherwise materialize. This is a case where an older "CTEs are always optimization fences in Postgres" blog claim is now **outdated** — check the [official `WITH` docs](https://www.postgresql.org/docs/current/queries-with.html) for the current default heuristic rather than trusting a pre-2019 post.

[↑ back to top](#table-of-contents)

---

## Window Functions

<a id="window-beginner"></a>

### Beginner

**What it is.** A window function computes a value across a *set of related rows* ("window") without collapsing them into one output row the way `GROUP BY` does — every input row survives, each annotated with a computed value.

```sql
SELECT name, price, category,
       avg(price) OVER (PARTITION BY category) AS category_avg
FROM products;
```

Every row keeps its own identity; `category_avg` is just an extra column computed per partition. This is the core difference from `GROUP BY`: grouping reduces row count, windowing does not.

[↑ back to top](#table-of-contents)

<a id="window-working-knowledge"></a>

### Working Knowledge

**Why window functions run *after* `GROUP BY`/`HAVING` but before `ORDER BY`.** Revisiting the [logical processing order](#select-working-knowledge):

```
FROM → WHERE → GROUP BY → HAVING → [window functions live here, inside SELECT] → DISTINCT → ORDER BY → LIMIT
```

This ordering is deliberate: window functions frequently need to rank or compare rows *within already-aggregated groups* (e.g., "top product per category by total revenue"), so they must see `GROUP BY`'s output, not raw rows. It also explains why a window function can appear in `SELECT`/`ORDER BY` but never in `WHERE` or `GROUP BY` directly — those clauses are evaluated before window functions exist yet.

**Ranking functions:**

```sql
SELECT name, price, category,
       row_number() OVER (PARTITION BY category ORDER BY price DESC) AS rn,
       rank()       OVER (PARTITION BY category ORDER BY price DESC) AS rnk,
       dense_rank() OVER (PARTITION BY category ORDER BY price DESC) AS drnk
FROM products;
```

- `row_number()` — unique, sequential, no ties (1, 2, 3, 4...).
- `rank()` — ties share a rank, and the *next* rank skips accordingly (1, 2, 2, 4).
- `dense_rank()` — ties share a rank, but no gap follows (1, 2, 2, 3).

**Wrong vs. right — filtering a window function's result:**

```sql
-- WRONG — window functions can't be filtered in WHERE; they don't exist yet
-- at the point WHERE is evaluated
SELECT name, category, row_number() OVER (PARTITION BY category ORDER BY price DESC) AS rn
FROM products
WHERE rn = 1;  -- ERROR: column "rn" does not exist

-- RIGHT — wrap it in a subquery or CTE, then filter the outer query
SELECT * FROM (
    SELECT name, category,
           row_number() OVER (PARTITION BY category ORDER BY price DESC) AS rn
    FROM products
) ranked
WHERE rn = 1;
```

[↑ back to top](#table-of-contents)

<a id="window-advanced"></a>

### Advanced

**Frame clauses** — `PARTITION BY` picks the group; the **frame** (`ROWS`/`RANGE`/`GROUPS ... BETWEEN ... AND ...`) picks which rows *within* that partition, relative to the current row, participate in the calculation. This is what running totals and moving averages are built from.

```sql
-- Running total of order amounts per customer, oldest to newest
SELECT customer_id, order_date, total_amount,
       sum(total_amount) OVER (
           PARTITION BY customer_id
           ORDER BY order_date
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total
FROM orders
WHERE customer_id IS NOT NULL;

-- 3-row moving average (current row + 2 preceding), per customer
SELECT customer_id, order_date, total_amount,
       avg(total_amount) OVER (
           PARTITION BY customer_id
           ORDER BY order_date
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) AS moving_avg_3
FROM orders
WHERE customer_id IS NOT NULL;
```

**`ROWS` vs. `RANGE` — a genuinely common source of subtly wrong numbers.** `ROWS` counts physical rows; `RANGE` groups by the *value* in `ORDER BY`, treating peer rows (equal values) as one unit:

```sql
-- Two orders on the SAME order_date for the same customer, with ROWS vs RANGE:
-- ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW gives each row its own
-- running total, incrementing one row at a time.
-- RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW (the default frame when
-- you write ORDER BY with no explicit frame!) treats same-date rows as PEERS
-- and gives them BOTH the combined total, not a running per-row total.
SELECT customer_id, order_date, total_amount,
       sum(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS default_is_range
FROM orders
WHERE customer_id IS NOT NULL;
```

If you want a strict row-by-row running total and there's any chance of duplicate `ORDER BY` values, specify `ROWS` explicitly — don't rely on the default frame. This exact "why is my running total wrong on tied dates" question is a recurring [Stack Overflow](https://stackoverflow.com/questions/tagged/window-functions+postgresql) theme, and the [official window function docs](https://www.postgresql.org/docs/current/functions-window.html) spell out the default (`RANGE UNBOUNDED PRECEDING`) explicitly — it's easy to miss because most tutorials only ever show `ROWS`.

**Postgres 18 performance note:** window aggregate processing was sped up in 18 (alongside `INTERSECT`/`EXCEPT`) — a plan-level improvement, not a syntax or semantics change. Source: [PostgreSQL 18 Release Notes](https://www.postgresql.org/docs/release/18.0/).

[↑ back to top](#table-of-contents)

<a id="window-mastery"></a>

### Mastery

**`LAG`/`LEAD` and `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE`** — read a neighboring or fixed-position row's value into the current row, without a self-join:

```sql
-- Change in order amount vs. this customer's previous order
SELECT customer_id, order_date, total_amount,
       total_amount - lag(total_amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS delta_vs_prev
FROM orders
WHERE customer_id IS NOT NULL;
```

**`LAST_VALUE` needs an explicit frame — the default frame silently breaks it.** Because the default frame ends at `CURRENT ROW`, `LAST_VALUE` under the default frame just returns the current row's own value, not the partition's actual last row:

```sql
-- WRONG — relies on the default frame (RANGE UNBOUNDED PRECEDING AND CURRENT ROW),
-- so last_price is just each row's own price, not the category's true last value
SELECT name, category, price,
       last_value(price) OVER (PARTITION BY category ORDER BY price) AS last_price
FROM products;

-- RIGHT — extend the frame to the full partition
SELECT name, category, price,
       last_value(price) OVER (
           PARTITION BY category ORDER BY price
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS last_price
FROM products;
```

**Frame exclusion (`EXCLUDE`)** — a standard SQL clause for excluding the current row (or its peers) from an otherwise-inclusive frame, useful for "compare this row to everyone else in its group" calculations:

```sql
-- Each order's amount vs. the average of every OTHER order from the same
-- customer (self excluded) — without EXCLUDE, you'd need a correlated
-- subquery or a manual subtract-and-divide correction
SELECT customer_id, order_id, total_amount,
       avg(total_amount) OVER (
           PARTITION BY customer_id
           ORDER BY order_date
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
           EXCLUDE CURRENT ROW
       ) AS avg_of_other_orders
FROM orders
WHERE customer_id IS NOT NULL;
```

`EXCLUDE CURRENT ROW` / `EXCLUDE GROUP` / `EXCLUDE TIES` are documented under [window function calls](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS) but rarely used in practice — most people reach for a manual `(sum(x) - x) / (count(*) - 1)` workaround without realizing `EXCLUDE` does exactly this, declaratively, and handles the "what if there's only one row" edge case (`NULL` division) the same way the built-in aggregate would.

**Named windows** — reduce repetition when several window functions in one query share the same `PARTITION BY`/`ORDER BY`:

```sql
SELECT customer_id, order_date, total_amount,
       sum(total_amount) OVER w AS running_total,
       avg(total_amount) OVER w AS running_avg,
       row_number()      OVER w AS order_sequence
FROM orders
WHERE customer_id IS NOT NULL
WINDOW w AS (PARTITION BY customer_id ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW);
```

[↑ back to top](#table-of-contents)

---

## Aggregates, GROUP BY & Grouping Sets

<a id="agg-beginner"></a>

### Beginner

**What it is.** `GROUP BY` collapses rows sharing the same value(s) into one row per group; aggregate functions (`count`, `sum`, `avg`, `min`, `max`) summarize each group.

```sql
SELECT category, count(*) AS product_count, avg(price) AS avg_price
FROM products
GROUP BY category;
```

Every column in the `SELECT` list must be either an aggregate or one of the `GROUP BY` columns — this is enforced, not a style preference:

```sql
-- WRONG — name isn't in GROUP BY and isn't aggregated: which product's
-- name would it even show for a category with 3 rows?
SELECT category, name, count(*) FROM products GROUP BY category;
-- ERROR: column "products.name" must appear in the GROUP BY clause
```

[↑ back to top](#table-of-contents)

<a id="agg-working-knowledge"></a>

### Working Knowledge

**`HAVING` filters groups; `WHERE` filters rows — and they run at different stages.** Per the [logical processing order](#select-working-knowledge), `WHERE` runs before grouping, `HAVING` runs after:

```sql
SELECT category, avg(price) AS avg_price
FROM products
WHERE price > 0                -- row-level filter, before grouping
GROUP BY category
HAVING avg(price) > 30;        -- group-level filter, after grouping
```

**Wrong vs. right — trying to filter on an aggregate in `WHERE`:**

```sql
-- WRONG — avg(price) doesn't exist yet when WHERE runs
SELECT category, avg(price) FROM products WHERE avg(price) > 30 GROUP BY category;
-- ERROR: aggregate functions are not allowed in WHERE

-- RIGHT — aggregate-level conditions belong in HAVING
SELECT category, avg(price) FROM products GROUP BY category HAVING avg(price) > 30;
```

**Aggregates quietly ignore NULLs (except `count(*)`):**

```sql
-- If a product had a NULL price, avg(price) and sum(price) would skip it
-- entirely rather than treating it as 0 — count(price) would also skip it,
-- but count(*) counts the row regardless
SELECT count(*), count(price), sum(price), avg(price) FROM products;
```

This NULL-skipping is documented behavior for every standard aggregate (`sum`, `avg`, `min`, `max`, `count(expr)`), per the [aggregate functions docs](https://www.postgresql.org/docs/current/functions-aggregate.html) — worth double-checking when a `sum()` or `avg()` looks lower than expected and the underlying column turns out to have NULLs mixed in.

[↑ back to top](#table-of-contents)

<a id="agg-advanced"></a>

### Advanced

**`GROUPING SETS`, `ROLLUP`, `CUBE`** — compute multiple grouping levels in one query instead of `UNION`-ing several separate `GROUP BY` queries:

```sql
-- Revenue by category, by status, AND the grand total — three grouping
-- levels in one pass, one result set
SELECT category, status, sum(total_amount) AS revenue
FROM orders o
JOIN order_items oi USING (order_id)
JOIN products p USING (product_id)
GROUP BY GROUPING SETS ((category, status), (category), ())
ORDER BY category NULLS LAST, status NULLS LAST;
```

`ROLLUP(a, b)` is shorthand for the grouping sets `(a,b), (a), ()` — hierarchical subtotals building up to a grand total. `CUBE(a, b)` is shorthand for *every* combination: `(a,b), (a), (b), ()`.

```sql
SELECT category, status, sum(total_amount) AS revenue
FROM orders o
JOIN order_items oi USING (order_id)
JOIN products p USING (product_id)
GROUP BY ROLLUP (category, status);
```

**Telling a real NULL from a "this row is a subtotal" NULL — `GROUPING()`:**

```sql
-- WRONG-ish — a NULL in the "status" column after ROLLUP could mean either
-- "an order with a genuinely NULL status" (doesn't exist in our sample data,
-- but could) or "this is the subtotal row for the whole category" — you
-- can't tell which from the value alone
SELECT category, status, sum(total_amount) AS revenue
FROM orders o JOIN order_items oi USING (order_id) JOIN products p USING (product_id)
GROUP BY ROLLUP (category, status);

-- RIGHT — GROUPING() returns 1 when a column was "rolled up away" for that
-- row (i.e., it's a subtotal row), 0 when the value is a real group key
SELECT category, status, sum(total_amount) AS revenue,
       GROUPING(status) AS is_subtotal_row
FROM orders o JOIN order_items oi USING (order_id) JOIN products p USING (product_id)
GROUP BY ROLLUP (category, status);
```

[↑ back to top](#table-of-contents)

<a id="agg-mastery"></a>

### Mastery

**Postgres 18: `GROUP BY` functional-dependency elision, extended.** Postgres has long allowed omitting non-key columns from `GROUP BY` when they're functionally dependent on the primary key already grouped by (e.g., `GROUP BY customers.customer_id` lets you also `SELECT customers.name` ungrouped, since one `customer_id` implies exactly one `name`). Postgres 18 extends this: **any unique index's columns** — not only the primary key — now qualify, so other columns of the same table become droppable from an otherwise-redundant `GROUP BY`. Source: [PostgreSQL 18 Release Notes](https://www.postgresql.org/docs/release/18.0/) ("Ignore GROUP BY columns that are functionally dependent on other columns"). This is an optimizer/planning change, not new syntax — queries that already relied on the primary-key version of this behavior are unaffected; some queries grouped by a `UNIQUE`-constrained column may now run a cheaper plan without any query change.

**Postgres 18: `HAVING` pushdown for `GROUPING SETS`, and a correctness fix.** Postgres 18 allows some `HAVING` conditions on `GROUPING SETS` queries to be evaluated earlier, as a `WHERE`-equivalent row filter, when the condition doesn't depend on which grouping set produced the row — letting rows be discarded before the (potentially expensive) grouping machinery runs. The same release also fixed some `GROUPING SETS` queries that had been **returning incorrect results** in earlier versions. If you have a `GROUPING SETS`/`ROLLUP`/`CUBE` query written against an older Postgres version and it looks fine on 18, that's expected — but it's worth re-verifying it on your target minor version if you were pinned below 18. Source: [PostgreSQL 18 Release Notes](https://www.postgresql.org/docs/release/18.0/).

**Custom aggregates and `FILTER` combined with window frames:** aggregates aren't limited to the built-ins — `sum(x) FILTER (WHERE ...) OVER (PARTITION BY ...)` composes a conditional aggregate *and* a window frame in one expression, useful for "running total of only completed orders, per customer":

```sql
SELECT customer_id, order_date, status,
       sum(total_amount) FILTER (WHERE status = 'completed') OVER (
           PARTITION BY customer_id ORDER BY order_date
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_completed_total
FROM orders
WHERE customer_id IS NOT NULL;
```

[↑ back to top](#table-of-contents)

---

## Set Operations (UNION, INTERSECT, EXCEPT)

<a id="set-beginner"></a>

### Beginner

**What it is.** Set operations combine the results of two `SELECT`s with the *same number and compatible types* of columns — unlike a join, which combines rows side by side, a set operation stacks or compares whole result sets.

```sql
-- All distinct locations we either have a customer OR a product category name matching (contrived, illustrates syntax)
SELECT country AS label FROM customers
UNION
SELECT category AS label FROM products;
```

`UNION` removes duplicate rows by default; `UNION ALL` keeps every row, including duplicates, and is cheaper because it skips the dedup step entirely.

[↑ back to top](#table-of-contents)

<a id="set-working-knowledge"></a>

### Working Knowledge

**`INTERSECT`** (rows in both) **and `EXCEPT`** (rows in the first, not the second):

```sql
-- Customers who exist in BOTH a "high value" list and a "2024 signups" list
SELECT customer_id FROM customers WHERE signup_date >= '2024-01-01'
INTERSECT
SELECT customer_id FROM orders WHERE total_amount > 100;

-- Customers who signed up but never placed an order (anti-join via EXCEPT)
SELECT customer_id FROM customers
EXCEPT
SELECT customer_id FROM orders WHERE customer_id IS NOT NULL;
```

`EXCEPT` here is a third way to express the same "customers with no orders" query from [Subqueries → Advanced](#subqueries-advanced) — alongside `NOT EXISTS` and a filtered `NOT IN`. For this shape, `NOT EXISTS` is usually still preferred in production because it doesn't need every column in both queries to line up, but `EXCEPT` reads cleanly when you're already comparing two same-shaped result sets.

[↑ back to top](#table-of-contents)

<a id="set-advanced"></a>

### Advanced

**Set operations treat `NULL` as equal to `NULL` for deduplication — the opposite of ordinary `=`.** This is the flip side of the `WHERE customer_id = NULL` trap from [SELECT → Working Knowledge](#select-working-knowledge): `UNION`/`INTERSECT`/`EXCEPT` (and plain `DISTINCT`) compare rows for duplicate elimination as if `NULL IS NOT DISTINCT FROM NULL` were true, even though `NULL = NULL` is not:

```sql
-- Two NULLs collapse into one row under UNION's dedup logic, even though
-- "NULL = NULL" is never true in an ordinary WHERE comparison
SELECT NULL::text AS x
UNION
SELECT NULL::text AS x;
-- returns exactly ONE row, not zero and not two
```

This is documented behavior — see the [`SELECT` docs' notes on `DISTINCT`](https://www.postgresql.org/docs/current/sql-select.html#SQL-DISTINCT) — but it surprises people precisely because it contradicts the "`NULL` never equals anything" rule everywhere else in SQL.

**Operator precedence without parentheses.** `INTERSECT` binds more tightly than `UNION`/`EXCEPT`, and `UNION`/`EXCEPT` associate left to right:

```sql
-- query1 UNION query2 INTERSECT query3
-- is evaluated as:  query1 UNION (query2 INTERSECT query3)  ← INTERSECT wins
--
-- query1 UNION query2 EXCEPT query3
-- is evaluated as:  (query1 UNION query2) EXCEPT query3     ← left to right
```

Don't rely on implicit precedence when mixing all three in one statement — add explicit parentheses so the query reads the same to the next person as it executes. Source: [Combining Queries — official docs](https://www.postgresql.org/docs/current/queries-union.html).

[↑ back to top](#table-of-contents)

<a id="set-mastery"></a>

### Mastery

**`UNION ALL` inside a recursive CTE is not just an optimization — it's frequently required for correctness**, as shown in [CTEs → Advanced](#ctes-advanced): plain `UNION`'s per-iteration dedup against the whole accumulated result changes both performance and, in edge cases, which rows survive.

**Set operations and column naming.** The output column names come from the **first** `SELECT` in the chain — naming or aliasing columns in later branches has no effect on the result's column names:

```sql
SELECT customer_id AS id FROM customers
UNION ALL
SELECT order_id AS totally_different_name FROM orders;
-- output column is named "id" — the second alias is silently ignored
```

**Postgres 18 performance note:** `INTERSECT`/`EXCEPT` (and hash-based set operations generally) got faster and lower-memory in 18, alongside hash joins and `GROUP BY` — again a plan-level change with no syntax or semantics difference. If you have an older captured `EXPLAIN` plan for a query using these operators, it's worth re-capturing rather than assuming the shape is unchanged. Source: [PostgreSQL 18 Release Notes](https://www.postgresql.org/docs/release/18.0/); see [Query Engine & Indexing](./postgres-query-engine-indexing.md) for how to read the plan itself.

[↑ back to top](#table-of-contents)

---

## Quick-Reference Cheat Sheet

### Join types

| Join | Keeps unmatched left rows? | Keeps unmatched right rows? | Typical use |
| --- | --- | --- | --- |
| `INNER JOIN` (`JOIN`) | No | No | Only rows with a match on both sides |
| `LEFT JOIN` | Yes (NULLs on right) | No | "All of A, with B where it exists" |
| `RIGHT JOIN` | No | Yes (NULLs on left) | Rarely used — equivalent to swapping table order + `LEFT JOIN` |
| `FULL JOIN` | Yes | Yes | Union of both sides' matched + unmatched rows |
| `CROSS JOIN` | n/a (no condition) | n/a | Every row × every row — deliberate combinations only |
| Self-join | depends on join type used | depends on join type used | Comparing rows of one table to each other (needs aliases) |
| `LATERAL` join | depends on `LEFT`/`CROSS` used | n/a | Right side can reference left side's columns per row (top-N per group) |

### Window frame clauses

| Frame unit | Groups by | Common use |
| --- | --- | --- |
| `ROWS BETWEEN x AND y` | Physical row position | Row-exact running totals/moving averages — use when ties in `ORDER BY` must NOT be merged |
| `RANGE BETWEEN x AND y` | Value in `ORDER BY` (peers merged) | Default frame when omitted — be explicit if you don't want peer rows merged |
| `GROUPS BETWEEN x AND y` | Distinct peer groups, counted as units | Rare; like `ROWS` but counting peer *groups* instead of individual rows |
| `EXCLUDE CURRENT ROW` | — | "Everyone but me" comparisons within a frame |
| `UNBOUNDED PRECEDING` / `CURRENT ROW` / `UNBOUNDED FOLLOWING` | — | Frame boundary keywords, combined with the units above |

### Ranking functions

| Function | Ties | Gaps after ties |
| --- | --- | --- |
| `row_number()` | Never ties (arbitrary tiebreak) | n/a |
| `rank()` | Shares rank | Yes (1, 2, 2, 4) |
| `dense_rank()` | Shares rank | No (1, 2, 2, 3) |

### Set operations

| Operator | Result | Duplicates | NULL row equality for dedup |
| --- | --- | --- | --- |
| `UNION` | Rows in either query | Removed (unless `ALL`) | Two NULLs treated as equal |
| `INTERSECT` | Rows in both queries | Removed (unless `ALL`) | Two NULLs treated as equal |
| `EXCEPT` | Rows in first, not second | Removed (unless `ALL`) | Two NULLs treated as equal |

### NULL-safety quick reference

| Situation | Trap | Safe alternative |
| --- | --- | --- |
| `WHERE col = NULL` | Always false, zero rows, no error | `WHERE col IS NULL` |
| `WHERE col NOT IN (subquery)` | Zero rows for ALL inputs if subquery yields any NULL | `NOT EXISTS (...)` or filter the subquery's NULLs explicitly |
| Comparing two possibly-NULL values | `=`/`<>` never true when either side is NULL | `IS [NOT] DISTINCT FROM` |
| `UNION`/`DISTINCT` dedup | Two NULLs ARE treated as duplicates of each other | Expected behavior — documented, not a bug |

[↑ back to top](#table-of-contents)

---

← Back to [Postgres SQL Mastery Guide](./postgres-mastery-guide.md)
