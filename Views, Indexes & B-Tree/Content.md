# Oracle Database Objects: Views & Indexes
### A Complete Theory Guide — Concepts, Internals, Algorithms, Real-World Use, and Practice

---

## How This Guide Is Organized

This is a reference document, not a quiz — read it top to bottom the first time, then come back to specific sections when you need a refresher. It covers:

- **Views** — what they are, why they exist, the different flavors (simple, complex, inline, materialized), and exactly how Oracle refreshes a materialized view behind the scenes.
- **Indexes** — what they are, how clustered vs. non-clustered architecture actually works (and how Oracle's approach differs from SQL Server/MySQL), and a genuinely deep dive into **how a B-Tree works internally** — the algorithm, not just the definition — plus the same depth for **Bitmap indexes**.
- **A real-world case study** tying both topics together.
- **Practice exercises with answers.**

All examples use one consistent sample schema so you can follow the thread from concept to concept:

```sql
CREATE TABLE employees (
    employee_id   NUMBER PRIMARY KEY,
    first_name    VARCHAR2(50),
    last_name     VARCHAR2(50),
    department    VARCHAR2(30),   -- e.g. 'SALES', 'IT', 'HR', 'FINANCE'
    job_title     VARCHAR2(30),   -- e.g. 'MANAGER', 'ANALYST', 'CLERK'
    salary        NUMBER(10,2),
    hire_date     DATE,
    region        VARCHAR2(15),   -- e.g. 'NORTH', 'SOUTH', 'EAST', 'WEST'
    status        VARCHAR2(10)    -- 'ACTIVE' or 'INACTIVE'
);
```

For the Materialized View section, we'll also use a second, larger table — the kind of thing materialized views actually exist to solve:

```sql
CREATE TABLE sales_transactions (
    txn_id        NUMBER PRIMARY KEY,
    employee_id   NUMBER,         -- the sales rep who closed the deal
    txn_date      DATE,
    region        VARCHAR2(15),
    amount        NUMBER(12,2)
);
```

Imagine `sales_transactions` has **millions** of rows — that scale is the entire reason materialized views exist, and it's worth holding that image in your head for that section.

---

# PART 1 — VIEWS

## 1.1 What Is a View?

A **view** is a **named, stored SQL query** that behaves like a table when you select from it — but it stores **no data of its own**. Every time you query a view, Oracle runs the underlying `SELECT` statement (or, more precisely, merges it into your query) and returns the result live, from the current data in the base table(s).

Think of a view as a **saved lens**, not a saved photograph. A photograph (a table, or a materialized view) captures a moment in time. A lens (a view) shows you whatever is actually there, right now, every time you look through it.

```sql
CREATE VIEW active_it_employees AS
SELECT employee_id, first_name, last_name, salary, hire_date
FROM employees
WHERE department = 'IT' AND status = 'ACTIVE';
```

From this point on, anyone can run:

```sql
SELECT * FROM active_it_employees WHERE salary > 80000;
```

...and Oracle transparently rewrites this, roughly, into:

```sql
SELECT employee_id, first_name, last_name, salary, hire_date
FROM employees
WHERE department = 'IT' AND status = 'ACTIVE' AND salary > 80000;
```

This rewriting process is called **view merging**, and it's key to understanding views correctly: **a view is not a cache**. If a new IT employee is hired one second after you create this view, that employee shows up in `active_it_employees` immediately, with zero extra work — because the view has no memory of its own; it's just a stored question that gets re-asked against live data every single time.

### What's actually stored in the data dictionary

Only the view's **definition** (the `SELECT` text) and its **metadata** (column names, data types inferred from the query) are stored — typically in `USER_VIEWS` / `ALL_VIEWS` (you can inspect it yourself with `SELECT text FROM user_views WHERE view_name = 'ACTIVE_IT_EMPLOYEES'`). No rows, no blocks of actual data, no separate storage segment. This is the single most important fact that distinguishes an ordinary view from a materialized view, covered in section 1.6.

---

## 1.2 Advantages of Views

**1. Security — column and row-level restriction.**
You can `GRANT SELECT` on a view without ever granting access to the underlying table. A view can expose only the columns a user is allowed to see (e.g., hide `salary` entirely) and only the rows they're allowed to see (via the `WHERE` clause). This is far more granular than table-level `GRANT`.

```sql
CREATE VIEW employee_directory AS
SELECT employee_id, first_name, last_name, department, job_title
FROM employees;   -- no salary column at all

GRANT SELECT ON employee_directory TO hr_intern;
```

**2. Simplicity — hiding complexity.**
A report that normally requires a 4-table join with aggregate functions and a `HAVING` clause can be wrapped in a view once, and every future consumer just writes `SELECT * FROM quarterly_sales_summary`. The complexity is written once by someone who understands it, and reused forever by people who don't need to.

**3. Logical data independence.**
If the underlying table structure changes (a column gets renamed, a table gets split into two, data moves to a new schema), you can often adjust the *view's definition* to compensate — and every application querying the view keeps working, completely unaware anything changed underneath. This decouples the physical schema from the logical interface applications depend on.

**4. Consistency — a single source of truth.**
If five different reports each need "active employees earning above department average," and each one reimplements that logic independently, you now have five chances for someone to get the logic subtly wrong or for it to drift out of sync when the business rule changes. One view, referenced five times, guarantees they all agree — always.

**5. Simplifying complex or repetitive queries for end users.**
Business analysts and BI tools often can't write (or shouldn't have to write) a 6-table join with window functions. A well-designed view turns that into something as simple as `SELECT * FROM sales_dashboard`.

**6. Backward compatibility during migrations.**
When restructuring a schema, you can rename/split/move the real tables and then recreate the *old* table name as a view over the *new* structure — every legacy query that references the old name keeps working without modification, while the real schema underneath has already moved on.

---

## 1.3 Types of Views

### Simple Views vs. Complex Views

| | Simple View | Complex View |
|---|---|---|
| Base tables | Exactly one | One or more (joins) |
| Aggregate functions | No | Often yes (`SUM`, `AVG`, `COUNT`...) |
| `GROUP BY` / `DISTINCT` | No | Often yes |
| Set operators (`UNION`, etc.) | No | Sometimes |
| Generally updatable via DML? | Usually yes | Usually no, or restricted |

### The rules for an updatable view (Oracle)

A view is **updatable** (you can `INSERT`/`UPDATE`/`DELETE` directly against it, and it correctly affects the underlying table) only if it satisfies conditions including:

- No `DISTINCT`, `GROUP BY`, `HAVING`, aggregate functions, or set operators (`UNION`, `MINUS`, `INTERSECT`) in the defining query
- No `ROWNUM` in the query
- For a join view, only **key-preserved tables** can be updated through the view — a table is key-preserved if every row in it maps to at most one row in the joined result (roughly: its primary/unique key uniquely determines each joined row)
- Columns being modified must map to a real column in a real base table, not an expression like `salary * 1.1`

```sql
-- Updatable: simple, single table, no aggregation
CREATE VIEW it_department AS
SELECT employee_id, first_name, salary FROM employees WHERE department = 'IT';

UPDATE it_department SET salary = salary * 1.05 WHERE employee_id = 101; -- works fine

-- NOT updatable: aggregation makes individual rows meaningless to modify
CREATE VIEW dept_avg_salary AS
SELECT department, AVG(salary) AS avg_sal FROM employees GROUP BY department;

UPDATE dept_avg_salary SET avg_sal = 90000 WHERE department = 'IT'; -- ORA-01732
```

### WITH CHECK OPTION

Without it, you can `INSERT`/`UPDATE` through a view in a way that makes the row **disappear from the view's own result** the moment you commit — because the new data no longer satisfies the view's `WHERE` clause. `WITH CHECK OPTION` blocks exactly that.

```sql
CREATE VIEW it_department AS
SELECT employee_id, first_name, department, salary
FROM employees
WHERE department = 'IT'
WITH CHECK OPTION;

-- This now FAILS, because it would move the row out of the view's own WHERE clause:
UPDATE it_department SET department = 'SALES' WHERE employee_id = 101;
-- ORA-01402: view WITH CHECK OPTION where-clause violation
```

### WITH READ ONLY

Explicitly disables all DML through the view, regardless of whether it would otherwise be updatable. Good practice for any view that exists purely for reporting.

```sql
CREATE VIEW employee_directory AS
SELECT employee_id, first_name, last_name, department
FROM employees
WITH READ ONLY;
```

### FORCE vs. NOFORCE

`CREATE FORCE VIEW` lets you create a view even if the base table doesn't exist yet (or you lack privileges on it right now) — the view is created in an invalid state and becomes usable once the dependency is resolved. `NOFORCE` (the default) requires everything to already be valid.

---

## 1.4 Inline Views

An **inline view** is a subquery placed directly in the `FROM` clause of another query — it acts exactly like a view (a virtual, queryable table) but exists **only for the duration of that one query**. It is never stored in the data dictionary, has no name outside the query, and disappears the moment the query finishes.

```sql
SELECT dept_summary.department, dept_summary.avg_salary
FROM (
    SELECT department, ROUND(AVG(salary), 2) AS avg_salary
    FROM employees
    WHERE status = 'ACTIVE'
    GROUP BY department
) dept_summary
WHERE dept_summary.avg_salary > 70000;
```

The parenthesized subquery aliased as `dept_summary` **is** the inline view — Oracle materializes it (conceptually) into a virtual result set and then runs the outer query against it, exactly as if it were a real table.

### Inline View vs. Named (Stored) View — the actual difference

| | Named View | Inline View |
|---|---|---|
| Stored as a schema object? | Yes, permanently, in the data dictionary | No — exists only inside the one query that defines it |
| Reusable across queries? | Yes, by name, anywhere | No — must be rewritten (or copy-pasted) into every query that needs it |
| Needs a `CREATE` privilege? | Yes | No — it's just a subquery |
| Typical use case | A stable, reusable business definition everyone should share | A one-off intermediate step needed to solve *this specific* query (pre-aggregating before a join, pagination, ranking before filtering) |

### Common real-world uses for inline views

- **Pre-aggregating before a join** — compute a `GROUP BY` summary first, *then* join it to another table, rather than joining raw rows and aggregating afterward (often much faster, since the join now operates on far fewer rows).
- **Two-step filtering** — compute a ranked/windowed result in the inline view, then filter on that computed rank in the outer query (you cannot filter on a window function's result in the same `SELECT`'s `WHERE` clause — recall the "order of execution" rule — so the inline view is exactly the workaround).

```sql
SELECT * FROM (
    SELECT employee_id, first_name, salary,
           RANK() OVER (ORDER BY salary DESC) AS salary_rank
    FROM employees
)
WHERE salary_rank <= 5;   -- top 5 earners, made possible by the inline view
```

- **Pagination** — combined with `ROWNUM` or `FETCH FIRST`/`OFFSET`, to select a specific "page" of ordered results.

---

## 1.5 Materialized Views

### What Is a Materialized View?

A **materialized view (MV)** looks like a view syntactically, but behaves completely differently at the physical level: **it actually stores the query's result set on disk**, in its own storage segment, just like a real table. Querying it doesn't re-run the defining query against the base tables — it just reads the pre-computed rows, which is why materialized views can be dramatically faster for expensive aggregations over huge tables.

The cost of that speed: the stored data can become **stale** — it reflects the base table as of the last **refresh**, not necessarily right now. Managing that staleness is the entire subject of section 1.5.3.

### View vs. Materialized View

| | View | Materialized View |
|---|---|---|
| Stores data? | No — just the query definition | Yes — the actual result set, physically |
| Query speed | Same as running the underlying query (view merging) | Very fast — just a table read |
| Data freshness | Always 100% current (live) | Only as current as the last refresh |
| Storage cost | None (metadata only) | Full storage for the result set |
| Write overhead | None | Refresh cost, paid periodically |
| Typical use case | OLTP, security, simplifying live queries | OLAP/reporting, data warehousing, expensive aggregates, replication |

### Creating a Materialized View

```sql
CREATE MATERIALIZED VIEW mv_regional_sales_summary
BUILD IMMEDIATE
REFRESH COMPLETE ON DEMAND
AS
SELECT region,
       TRUNC(txn_date, 'MM') AS sales_month,
       COUNT(*)              AS txn_count,
       SUM(amount)           AS total_sales
FROM sales_transactions
GROUP BY region, TRUNC(txn_date, 'MM');
```

- **`BUILD IMMEDIATE`** (default) populates the MV right away with the query's current results. **`BUILD DEFERRED`** creates the object empty, to be populated by the first refresh (useful when you want to create many MVs quickly and refresh them all together later, off-peak).
- **`REFRESH ... ON DEMAND`** (default) means the MV only updates when explicitly told to (manually, or via a scheduled job). **`REFRESH ... ON COMMIT`** means Oracle automatically refreshes the MV the instant a transaction against the base table commits.

### 1.5.3 Refresh Techniques — the real depth here

This is the part of materialized views that actually matters in practice, because getting it wrong either wastes enormous amounts of database resources or silently serves stale data to a dashboard.

**COMPLETE refresh**

Oracle simply **truncates the materialized view and re-runs the entire defining query from scratch**, exactly as if you'd dropped and recreated it.

- Always works, no matter how complex the query is (joins, aggregates, `DISTINCT`, anything).
- Guaranteed correct, but potentially extremely expensive on a huge base table — you're re-aggregating everything every time, even if only 10 rows changed since the last refresh.

```sql
EXEC DBMS_MVIEW.REFRESH('MV_REGIONAL_SALES_SUMMARY', 'C');  -- 'C' = Complete
```

**FAST refresh**

Instead of recomputing everything, Oracle looks at only the rows in the base table that **changed** since the last refresh, and applies just that incremental delta to the existing materialized view data — far cheaper on a large, mostly-unchanged table.

FAST refresh has real prerequisites, and this is a very common exam/interview point:

- A **materialized view log** must exist on every base table involved, recording each row-level `INSERT`/`UPDATE`/`DELETE` since the last refresh (essentially a change-tracking table Oracle maintains automatically via a trigger-like mechanism):

```sql
CREATE MATERIALIZED VIEW LOG ON sales_transactions
WITH ROWID, SEQUENCE (employee_id, region, txn_date, amount)
INCLUDING NEW VALUES;
```

- The defining query has restrictions — for aggregate MVs specifically, it generally needs `COUNT(*)` alongside every `SUM`/`AVG` (so Oracle can correctly recompute an average incrementally, since `AVG` isn't simply additive — Oracle needs the running sum *and* count to adjust it), and every column being aggregated needs to be trackable via the log.
- Not every possible query is fast-refreshable — complex multi-table joins with certain constructs may not qualify at all, in which case Oracle raises an error at MV creation time if you request `FAST` and it isn't achievable.

```sql
CREATE MATERIALIZED VIEW mv_regional_sales_summary
BUILD IMMEDIATE
REFRESH FAST ON DEMAND
AS
SELECT region, TRUNC(txn_date,'MM') AS sales_month,
       COUNT(*) AS txn_count, SUM(amount) AS total_sales
FROM sales_transactions
GROUP BY region, TRUNC(txn_date,'MM');

EXEC DBMS_MVIEW.REFRESH('MV_REGIONAL_SALES_SUMMARY', 'F');  -- 'F' = Fast
```

**FORCE refresh**

The pragmatic default for most real systems: Oracle **attempts a FAST refresh first**; if for any reason a fast refresh isn't currently possible (say, the MV log was dropped, or too much has changed), it **transparently falls back to a COMPLETE refresh** instead of simply failing.

```sql
EXEC DBMS_MVIEW.REFRESH('MV_REGIONAL_SALES_SUMMARY', 'F', force => TRUE);
-- or equivalently, REFRESH FORCE ON DEMAND at creation time
```

**ON COMMIT vs. ON DEMAND — the timing dimension**

These control *when* a refresh happens, independent of *how* (complete/fast/force):

- **`ON DEMAND`**: nothing happens automatically. Someone (a DBA, a scheduled `DBMS_SCHEDULER` job, an ETL pipeline) must explicitly trigger a refresh — typical for nightly batch reporting, where "as of last night" freshness is perfectly acceptable.
- **`ON COMMIT`**: the moment any transaction against a base table commits, Oracle automatically refreshes the MV (incrementally, if fast-refreshable) as part of that same commit. This keeps the MV essentially real-time, but adds refresh overhead directly onto every write transaction against the base table — a real trade-off between OLTP write latency and reporting freshness.

**A realistic refresh strategy for a nightly sales report:**

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_JOB (
    job_name        => 'refresh_sales_mv_nightly',
    job_type        => 'PLSQL_BLOCK',
    job_action      => 'BEGIN DBMS_MVIEW.REFRESH(''MV_REGIONAL_SALES_SUMMARY'', ''F''); END;',
    start_date      => TRUNC(SYSDATE) + 1 + 2/24,  -- 2 AM tomorrow
    repeat_interval => 'FREQ=DAILY; BYHOUR=2',
    enabled         => TRUE
  );
END;
/
```

### Query Rewrite

A related, more advanced feature: if a materialized view is created `ENABLE QUERY REWRITE`, the Oracle optimizer can **automatically redirect** a query written against the *base tables* to use the materialized view instead — completely transparently, without the application ever knowing an MV exists — whenever it recognizes the MV already contains the answer. This lets you get materialized-view performance while your application code still queries the original tables directly.

### 1.5.4 Real-World Use Cases for Materialized Views

- **Data warehousing star-schema aggregates**: pre-computing "total sales by region by month" once, refreshed nightly, instead of scanning and re-aggregating hundreds of millions of transaction rows every single time an executive opens a dashboard.
- **Replication**: keeping a read-only copy of a table (or a subset of it) in a different database or geographic location, refreshed periodically — a lightweight alternative to full replication tooling for reporting-only replicas.
- **Reducing OLTP load**: heavy analytical queries running directly against the live transactional tables can slow down the very system trying to process real orders/payments; offloading those queries to a materialized view protects OLTP performance.
- **Pre-computed KPIs and dashboards**: any metric that's expensive to compute but doesn't need to be second-by-second accurate (most business dashboards) is a strong MV candidate.

---

## 1.6 Real-Life Case Scenarios for Views (Quick Hits)

- **Row/column-level security layer**: a multinational company gives each country's regional manager a view filtered to `WHERE region = <their region>` — one underlying table, dozens of tailored views, zero duplicated data.
- **API/application abstraction layer**: an application team builds against a stable set of views even while the DBA team refactors the underlying physical schema behind the scenes (normalizing tables, moving columns) — as long as the views' output contract doesn't change, the application never notices.
- **ETL staging**: an inline view is commonly used mid-pipeline to reshape/filter data before it lands in a final destination table, without ever creating a permanent intermediate object.
- **Legacy system bridge**: after a schema migration, the old table name is recreated as a view mapping onto the new structure, so hundreds of old reports and scripts that hardcode the old table name keep working unmodified.

# PART 2 — INDEXES

## 2.1 What Is an Index?

An index is a **separate data structure** that stores a sorted, searchable mapping from column values to the physical location of the rows that contain them — so the database can find matching rows without having to scan the entire table.

The classic analogy is a book's index: to find every page mentioning "materialized views," you don't read the whole book front to back — you flip to the index at the back, find the term alphabetically, and jump straight to the listed page numbers. A database index does exactly this, using a **ROWID** (Oracle's internal, physical row address) instead of a page number.

```sql
-- Without an index: Oracle must read every single block of the table (a "full table scan")
-- to find employee_id = 4521 among a million rows.
SELECT * FROM employees WHERE employee_id = 4521;

CREATE INDEX idx_emp_id ON employees(employee_id);
-- With an index: Oracle jumps almost directly to the matching row.
```

### The fundamental trade-off

An index is **never free**. It costs:

- **Storage** — a full copy of the indexed column's values, plus pointers, stored in its own separate structure.
- **Write overhead** — every `INSERT`, `UPDATE` (of an indexed column), or `DELETE` must also update every index defined on that table, not just the table itself. A table with 6 indexes pays that update cost 6 extra times on every write.

This trade-off — **faster reads, slower writes, more storage** — is the single most important thing to internalize about indexing. It's why indexing isn't "always create more indexes"; it's a deliberate design decision, covered fully in section 2.8.

### How the optimizer decides to use an index (or not)

Having an index doesn't guarantee Oracle will use it. The **cost-based optimizer** looks at table/index statistics (row counts, distinct value counts, data distribution) and estimates whether an index lookup or a full table scan will be cheaper for a *specific* query. For a query expected to return most of the table's rows (low **selectivity**), a full table scan can genuinely be faster than the overhead of index lookups plus the subsequent trips back to the table — this is exactly why bitmap vs. B-tree choice (section 2.4) depends so heavily on column cardinality.

---

## 2.2 Index Architecture: Clustered vs. Non-Clustered

This is general relational-database theory (the terms come up constantly in SQL Server and MySQL/InnoDB), and it's worth understanding properly — including where Oracle's actual implementation diverges from the terminology.

### The general concept

**Clustered index**: the table's actual data rows are physically stored on disk **in the sorted order of the index key itself**. The index *is*, in effect, the table — there's no separate "table storage" plus "index storage"; the leaf level of the index *contains* the full row data. Because data can only be physically sorted one way at a time, **a table can have at most one clustered index**.

**Non-clustered index**: a completely separate structure from the table. It stores (key value → row locator) pairs, sorted by key, but the actual table rows remain wherever they physically are (commonly in unsorted, "heap" order). Looking up a value means: search the index, get a pointer, then follow that pointer to fetch the actual row from the table — an extra hop compared to a clustered index. A table can have **many** non-clustered indexes.

| | Clustered Index | Non-Clustered Index |
|---|---|---|
| Data storage | IS the table, sorted by key | Separate structure with pointers back to the table |
| How many per table | At most 1 | Many |
| Lookup for indexed value | Direct — data is right there in the leaf | Two-step — index leaf, then fetch from table |
| Range queries on the key | Extremely fast (data is physically contiguous) | Fast, but with extra row-fetch overhead per result |

### Oracle's actual implementation — an important nuance

By default, an Oracle table is a **heap-organized table**: rows are stored in no particular guaranteed order, wherever there's free space. A standard `CREATE INDEX` in Oracle builds a **B-tree index whose leaf nodes store the indexed value plus a ROWID pointer back into that heap** — structurally, this is a **non-clustered index**, even though Oracle documentation doesn't typically use the word "non-clustered" at all.

Oracle offers two features that behave like a true clustered index:

**1. Index-Organized Table (IOT)** — the table itself is stored *as* a B-tree, keyed on its primary key, with the full row data living in the leaf nodes instead of separate table blocks. This is Oracle's closest direct equivalent to SQL Server's clustered index.

```sql
CREATE TABLE employees_iot (
    employee_id  NUMBER PRIMARY KEY,
    first_name   VARCHAR2(50),
    last_name    VARCHAR2(50),
    salary       NUMBER(10,2)
) ORGANIZATION INDEX;
```

Great for tables that are almost always looked up by primary key and rarely need alternate access paths — lookups by `employee_id` need no second hop to a separate table segment at all.

**2. Oracle Clusters** — a distinct schema object (not to be confused with "clustered index") that physically stores rows from **one or more related tables together on the same data blocks**, grouped by a shared cluster key. For example, storing each department's employees physically adjacent to that department's row, so fetching "a department and all its employees" touches far fewer disk blocks than if the two tables were stored independently.

**The practical takeaway**: when you see the term "clustered index" in Oracle material or interview questions, understand it conceptually (data physically sorted by key, single structure), but know that in real Oracle systems it's usually **Index-Organized Tables** doing that job, while ordinary `CREATE INDEX` in Oracle is architecturally a **non-clustered** B-tree index by default.

---

## 2.3 B-Tree Index — Deep Dive

This is the default, general-purpose index type in Oracle (a plain `CREATE INDEX ... ON ...` is a B-tree index unless you specify otherwise), so it's worth understanding not just *what* it is, but *how it actually works internally* — the algorithm, not just the name.

### 2.3.1 What "B-Tree" Actually Means

**B-Tree stands for "Balanced Tree"** — not "Binary Tree," a very common mix-up. Unlike a binary tree (where each node has at most 2 children and holds 1 key), a B-tree node can hold **many** keys and have **many** children — often hundreds, in a real database index. This high **fan-out** (branching factor) is deliberate and crucial: it means the tree stays extremely **shallow** even over millions of rows, which directly minimizes the number of disk reads needed to find anything — and disk I/O, not comparisons, is the real bottleneck in a database.

**"Balanced"** means: every leaf node sits at **exactly the same depth** from the root, no matter which path you take to reach it. This is the property that guarantees consistent, predictable performance — there's no "worst case" branch of the tree that's suddenly much deeper than the rest, which is exactly the failure mode of a naive, unbalanced binary search tree when data is inserted in sorted order (it degenerates into what's effectively a linked list, with O(n) lookups instead of O(log n)).

### 2.3.2 Structure

A B-tree index has three levels of node:

- **Root node** — the single entry point at the top of the tree.
- **Branch nodes** (internal/non-leaf nodes) — contain keys and pointers that route a search down toward the correct leaf. A tree can have zero or more levels of branch nodes depending on how much data there is (more data → more levels, but very slowly, because of the high fan-out).
- **Leaf nodes** — the bottom level. Each leaf entry holds an actual **(indexed column value, ROWID)** pair — the ROWID being the physical pointer back to the real row in the table. Leaf nodes are additionally **linked together** in a doubly-linked list, left to right in sorted key order — this is what makes range scans (`BETWEEN`, `>`, `ORDER BY`) fast: once you find the starting point, you simply walk the linked leaf chain instead of re-traversing the tree for every subsequent value.

```
                         [ Root ]
                      50        90
                    /     |       \
             [Branch]  [Branch]  [Branch]
            10  30    50  70     90  99
           /  |  \   /  |  \    /  |   \
       [Leaf][Leaf][Leaf][Leaf][Leaf][Leaf][Leaf]
        1-9  10-29 30-49 50-69 70-89 90-98 99+
         |     |     |     |     |     |     |
      (val,ROWID) pairs, leaf nodes linked left-to-right →→→→→→→→→→
```

In a real Oracle B-tree index, these correspond directly to physical **blocks**: a **root block**, one or more levels of **branch blocks**, and **leaf blocks** — and the number of levels is called the index's **BLEVEL** (you can inspect it in `INDEX_STATS`/`USER_INDEXES`). Because each block can hold hundreds of entries, even a table with **tens of millions of rows** typically produces a B-tree only **3–4 levels deep** — meaning finding any single row takes only 3–4 block reads, regardless of table size. That logarithmic depth relative to data volume is the entire point of the structure.

### 2.3.3 The Search Algorithm — Step by Step

To find a value (say, searching for the key `62`):

1. **Start at the root.** The root contains a small number of keys that partition the entire key range. Binary-search (or scan) the root's keys to determine which child pointer to follow — e.g., if the root's keys are `[50, 90]`, then `62` falls between them, so follow the middle child pointer.
2. **Descend to the branch node.** Repeat the same process: binary-search this node's keys to pick the next child pointer down. This may repeat across multiple branch levels for a very large index.
3. **Arrive at a leaf node.** Binary-search *within* the leaf for the exact key. If found, you now have the ROWID — a direct, physical pointer to the exact row.
4. **Fetch the row.** Use the ROWID to go directly to the exact data block containing the row — no scanning required.

For a **range query** (`WHERE salary BETWEEN 60000 AND 90000`), steps 1–3 locate the *first* qualifying leaf entry, and then Oracle simply **walks the linked list of leaf nodes to the right**, collecting every entry until it passes the upper bound — no need to re-traverse from the root for each subsequent match. This is precisely why B-tree indexes are excellent for both **equality** lookups and **range** lookups, unlike a hash-based structure (which only supports fast equality checks and cannot support "give me everything between X and Y" at all).

### 2.3.4 The Insert Algorithm — Node Splitting

Inserting a new key follows the same downward traversal as a search, to find the correct leaf — then:

1. **Insert the key into the leaf, in sorted position.**
2. **If the leaf now still has room** (below its maximum capacity), you're done — this is the common case, and it's fast.
3. **If the leaf is now full (overflow)**, it must **split**: the leaf's keys are divided roughly in half into two leaf nodes, and a copy of the **middle (or first) key of the new right leaf** is pushed **up** into the parent branch node, along with a pointer to the new leaf.
4. **If that push-up causes the parent branch node to overflow too**, the *same splitting process* happens one level up — recursively, all the way up if necessary.
5. **If even the root overflows and splits**, a **brand-new root** is created above it, pointing to the two halves of the old root — and this is the *only* way the tree's height ever increases, which is why it happens so rarely and why the tree stays so shallow.

**Worked example** — building a small B-tree (max 3 keys per node, for illustration) by inserting `10, 20, 30, 40, 5`:

```
Insert 10:                         [10]

Insert 20:                         [10, 20]

Insert 30:                         [10, 20, 30]     (leaf now full — at capacity)

Insert 40:  LEAF OVERFLOWS -> SPLIT
            Middle key (20) pushes up to become a new root:

                                    [20]
                                   /    \
                               [10]    [30, 40]

Insert 5:   goes to the left leaf, which has room:

                                    [20]
                                   /    \
                            [5, 10]    [30, 40]
```

Notice the tree grew from a single node to two levels **only once**, precisely when the very first split occurred — and it grew by promoting one key upward, not by copying the whole structure.

### 2.3.5 The Delete Algorithm — Textbook vs. Oracle's Real Behavior

**Textbook B-tree deletion** works roughly the reverse of insertion: remove the key from its leaf; if the leaf now has **too few** keys (underflow, below some minimum threshold), it **borrows** a key from an adjacent sibling if that sibling has one to spare, or **merges** with a sibling if not — and a merge can cascade the underflow up to the parent, potentially shrinking the tree's height if the root itself ends up with just one child.

**What Oracle actually does is more pragmatic**, and this is a genuinely important real-world distinction: Oracle B-tree indexes generally **do not eagerly rebalance or merge nodes on every delete**. Instead, a deleted leaf entry is simply **marked as deleted (logically removed)**, and the physical space isn't immediately reclaimed or the tree restructured. Why? Because immediate rebalancing on every single delete would be expensive, and in most workloads that space gets reused again soon anyway by future inserts into the same key range.

The consequence: over time, a table with **heavy delete/update activity** on an indexed column can accumulate a lot of "dead" space inside its B-tree index, making it larger and slightly less efficient than it needs to be. Oracle gives you two explicit maintenance tools for this:

- **`ALTER INDEX ... COALESCE`** — merges adjacent leaf blocks that have a lot of free space, *without* rebuilding the whole index or requiring an exclusive lock — a lightweight, online-friendly cleanup.
- **`ALTER INDEX ... REBUILD`** — completely rebuilds the index structure from scratch, fully compact and optimal, but a heavier operation (though it can be done `ONLINE` in Enterprise Edition to avoid blocking DML).

```sql
ALTER INDEX idx_emp_salary COALESCE;
-- or, for a more thorough cleanup:
ALTER INDEX idx_emp_salary REBUILD ONLINE;
```

### 2.3.6 Time Complexity

| Operation | Complexity | Why |
|---|---|---|
| Search (equality) | O(log n) | Tree height grows logarithmically with row count, thanks to high fan-out |
| Search (range) | O(log n + k) | O(log n) to find the start, then O(k) to walk k matching leaf entries |
| Insert | O(log n) amortized | Occasional splits are rare and still only cost O(log n) to propagate |
| Delete | O(log n) | Locating the key is O(log n); Oracle's lazy-delete approach keeps the actual delete itself cheap |

`n` here is the number of rows/keys in the index — and because of the logarithmic relationship, going from 1 million to 1 billion rows barely changes the tree's height at all (roughly 4-5 levels either way, given a realistic fan-out in the hundreds), which is the whole reason B-tree indexes scale so well.

### 2.3.7 When B-Tree Is the Right Choice

- **High-cardinality columns** (many distinct values — `employee_id`, `email`, `order_id`) — the default, general-purpose choice.
- **OLTP systems** — frequent, individual-row `INSERT`/`UPDATE`/`DELETE` — B-tree's per-row, localized update cost handles this far better than a bitmap index would (see section 2.4.4 for why bitmap indexes are actively dangerous here).
- **Primary keys and foreign keys** — Oracle automatically creates a unique B-tree index to enforce a primary key constraint.
- **Range queries and `ORDER BY`** — the sorted, linked-leaf structure directly serves `BETWEEN`, `>`, `<`, and can let Oracle skip a separate sort step entirely if the query's `ORDER BY` matches the index's key order.

---

## 2.4 Bitmap Index — Deep Dive

### 2.4.1 Structure

Instead of storing (value, ROWID) pairs like a B-tree, a bitmap index builds **one bit-vector (bitmap) per distinct value** in the indexed column. Each bitmap has exactly one bit for **every row in the table**: the bit is `1` if that row has this particular value, `0` otherwise.

**Worked example** — a `region` column with 4 distinct values, across 8 rows:

| ROWID (row position) | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|
| `region` value | NORTH | SOUTH | NORTH | EAST | WEST | SOUTH | NORTH | EAST |

The bitmap index stores exactly this, as four separate bit-vectors:

```
Bitmap for 'NORTH':   1  0  1  0  0  0  1  0
Bitmap for 'SOUTH':   0  1  0  0  0  1  0  0
Bitmap for 'EAST' :   0  0  0  1  0  0  0  1
Bitmap for 'WEST' :   0  0  0  0  1  0  0  0
```

### 2.4.2 How It Accelerates Queries

The real power of a bitmap index shows up when **multiple conditions on multiple bitmap-indexed columns** are combined — because combining bit-vectors is just a raw **bitwise `AND`/`OR`/`NOT`** operation, something a CPU does extraordinarily fast, on huge numbers of rows at once.

```sql
SELECT * FROM employees WHERE region = 'NORTH' AND status = 'ACTIVE';
```

If both `region` and `status` have bitmap indexes, Oracle takes the `NORTH` bitmap and the `ACTIVE` bitmap and performs a single bitwise `AND` between them — the resulting bitmap has a `1` in exactly the positions where *both* conditions are true, instantly identifying every matching row, without ever touching a B-tree traversal for each condition separately and then merging row-id lists. This is dramatically more efficient than B-tree indexes for exactly this pattern: **many low-cardinality filters combined together**, which is the bread and butter of data-warehouse "slice and dice" reporting queries (filter by region AND category AND status AND year, all at once).

### 2.4.3 Compression

Real bitmap indexes are almost never stored as raw, uncompressed bit strings — for a low-cardinality column, a bitmap is overwhelmingly long runs of `0`s punctuated by occasional `1`s, which compresses extremely well. Oracle stores bitmap indexes in a compressed form internally, which is a major part of why they stay compact even across tables with tens of millions of rows, as long as the indexed column's cardinality stays low.

### 2.4.4 When to Use — and When Absolutely Not To

**Good fit:**
- **Low-cardinality columns** — a handful of distinct values (`gender`, `status`, `region`, `yes/no` flags, `order_status`). The fewer distinct values, the fewer bitmaps needed, and the more effective each one is.
- **Read-heavy, analytical / data-warehouse / decision-support systems** — where queries constantly combine several filter conditions and the underlying data doesn't change every few seconds.
- **Static or rarely-updated (batch-loaded) data** — nightly ETL loads, not constant live transactions.

**Actively dangerous fit — high-DML OLTP systems:**

This is the single most important practical warning about bitmap indexes, and a favorite interview question: **a single-row `UPDATE`/`INSERT`/`DELETE` on a bitmap-indexed column doesn't just lock that one row's bit — it can lock the *entire bitmap segment*, which represents potentially thousands of unrelated rows.** In a busy OLTP system with many concurrent transactions, this causes severe **lock contention**: two completely unrelated transactions, updating two completely different rows, can end up blocking each other simply because both rows happen to fall within the same compressed bitmap chunk. Never put a bitmap index on a column in a table that experiences frequent concurrent single-row writes.

**Poor fit — high-cardinality columns:** if a column has (nearly) as many distinct values as there are rows (like `employee_id` or `email`), a bitmap index would need almost as many separate bitmaps as there are rows — each one mostly `0`s with a single `1` — which is both wasteful and pointless. A B-tree is the correct structure for that case instead.

### 2.4.5 B-Tree vs. Bitmap — Side by Side

| | B-Tree Index | Bitmap Index |
|---|---|---|
| Best for cardinality | High (many distinct values) | Low (few distinct values) |
| Best for workload | OLTP — frequent single-row writes | OLAP / data warehouse — read-heavy, batch-loaded |
| Combining multiple conditions | Merges row-id lists (slower) | Bitwise AND/OR (extremely fast) |
| Concurrency under DML | Row-level, safe for concurrent writes | Can lock large row ranges — dangerous under concurrent DML |
| Storage pattern | Sorted (key, ROWID) pairs in a tree | One compressed bit-vector per distinct value |
| Typical columns | Primary keys, foreign keys, IDs, dates, high-variety text | Gender, status flags, region, category, boolean-like columns |

```sql
CREATE INDEX idx_emp_id ON employees(employee_id);              -- B-tree (default)
CREATE BITMAP INDEX idx_emp_status ON employees(status);        -- Bitmap
CREATE BITMAP INDEX idx_emp_region ON employees(region);        -- Bitmap
```

---

## 2.5 Unique Index

A **unique index** guarantees that no two rows in the table have the same value in the indexed column(s) — with one well-known exception: **Oracle allows any number of rows to have `NULL`** in a unique-indexed column, because `NULL` is never considered equal to another `NULL` (or to anything else) in SQL's three-valued logic.

```sql
CREATE UNIQUE INDEX idx_emp_email ON employees(email);
```

### Unique Index vs. Unique Constraint

These are closely related but conceptually distinct:

- A **`UNIQUE` constraint** (or `PRIMARY KEY` constraint) is a **declarative, logical rule** recorded in the data dictionary — it documents *business intent* ("this column must be unique") and shows up clearly in `USER_CONSTRAINTS`, tools, and ER diagrams.
- A **unique index** is the **physical mechanism** that actually *enforces* that rule at the storage level.

When you create a `PRIMARY KEY` or `UNIQUE` constraint, **Oracle automatically creates a backing unique index for you** if one doesn't already exist — so in practice, every unique constraint has a unique index working behind the scenes. The reverse isn't automatic: you *can* create a standalone unique index directly (as above) without ever declaring a formal constraint — it will still enforce uniqueness, but it won't be as clearly documented as an explicit business rule in the schema's metadata, and tools that read constraint metadata (rather than index metadata) won't necessarily surface it the same way.

```sql
-- These two statements achieve very similar physical enforcement,
-- but only the first one also documents the rule as a named constraint:
ALTER TABLE employees ADD CONSTRAINT uq_emp_email UNIQUE (email);   -- creates a backing unique index automatically
CREATE UNIQUE INDEX idx_emp_email2 ON employees(email);             -- enforces uniqueness, but isn't a declared constraint
```

### Composite Unique Index

Uniqueness can also be enforced across the **combination** of multiple columns, even when each individual column may repeat on its own:

```sql
CREATE UNIQUE INDEX idx_emp_name_dept ON employees(first_name, last_name, department);
-- Two "John Smith"s are fine, as long as they're in different departments.
-- Two "John Smith"s in the SAME department are rejected.
```

---

## 2.6 Other Index Types Worth Knowing (Briefly)

- **Composite (concatenated) index** — a B-tree index built on multiple columns together, e.g. `(department, hire_date)`. Extremely useful when queries regularly filter on the same combination of columns together — but column *order* matters enormously: this index efficiently serves `WHERE department = 'IT'` and `WHERE department = 'IT' AND hire_date > ...`, but is much less useful for a query that filters on `hire_date` alone, since that's not the leading column.
- **Function-based index** — indexes the *result of an expression or function* applied to a column, not the raw column value: `CREATE INDEX idx_upper_email ON employees(UPPER(email))`. Necessary if your queries routinely filter using `WHERE UPPER(email) = 'X'` — a plain index on `email` can't help with that, since the stored index values wouldn't match what's being searched for.
- **Reverse key index** — a B-tree index where the bytes of the key are physically reversed before being stored. This is a niche fix for a specific problem: when a column is populated by a sequence (constantly increasing values), every new insert lands in the *same* rightmost block of the index, creating a "hot block" contention point under heavy concurrent inserts. Reversing the key bytes scatters new entries randomly across the index instead.
- **Partitioned index (local vs. global)** — on a partitioned table, a **local** index is partitioned to exactly mirror the table's own partitions (each index partition only indexes rows from its matching table partition — easy to maintain, and partition-independent). A **global** index spans across table partitions in its own independent partitioning scheme — more flexible for certain queries, but more expensive to maintain when table partitions are dropped or added.

---

## 2.7 Choosing the Right Index — A Practical Decision Guide

1. **Is this column frequently used in `WHERE`, `JOIN`, or `ORDER BY`?** If not, don't index it at all — it's pure overhead.
2. **How many distinct values does it have (cardinality)?**
   - High cardinality (IDs, emails, timestamps) → **B-tree**.
   - Low cardinality (status, gender, boolean flags, region) → consider **Bitmap** — *but only if step 3 allows it.*
3. **What's the write pattern on this table?**
   - Frequent concurrent single-row writes (OLTP) → **B-tree only**, even for a low-cardinality column; avoid Bitmap here entirely.
   - Batch-loaded, read-heavy, rarely updated (data warehouse/reporting) → **Bitmap is safe and often ideal** for low-cardinality columns.
4. **Do queries filter using an expression, not the raw column** (`UPPER(name)`, `TRUNC(date)`)? → **Function-based index**.
5. **Do queries routinely filter on the same multiple columns together?** → **Composite index**, with the most selective / most commonly-filtered-alone column listed first.
6. **Does this need to be enforced as a business rule (no duplicates)?** → **Unique index** (ideally backed by a formal constraint).

## 2.8 When NOT to Index

- **Small tables.** If a table only has a few hundred rows, a full table scan is often *faster* than the overhead of an index lookup — the entire table might fit in a handful of disk blocks anyway.
- **Low-selectivity columns for a B-tree.** An index on a column where one value covers 80% of the rows (e.g., `status = 'ACTIVE'` when almost everyone is active) rarely helps — the optimizer will likely ignore it and scan the table anyway, since following an index for that many matching rows is more expensive than just reading the table directly.
- **Columns rarely referenced in `WHERE`/`JOIN`/`ORDER BY`.** An index nobody's queries ever use is pure cost with zero benefit.
- **Write-heavy tables with many existing indexes.** Every additional index on a table makes every `INSERT`/`UPDATE`/`DELETE` against it slower. There's a real ceiling past which "just add another index" starts actively hurting overall system throughput.
- **Bitmap indexes on any column in a high-concurrency OLTP write path** — covered in depth in 2.4.4, but it bears repeating as its own rule: this is the most common real-world bitmap-index mistake.

---

## 2.9 How Views and Indexes Work Together

- An ordinary view has no indexes of its own — but when Oracle merges the view into your outer query (section 1.1), any indexes on the **underlying base tables** are still fully available to the optimizer, exactly as if you'd queried those tables directly. A well-written view doesn't cost you index usage.
- A **materialized view**, since it physically stores its own data in its own segment, **can and often should have its own indexes** — including a unique index on whatever the MV's defining query naturally treats as a key (e.g., `region + sales_month` in the earlier `mv_regional_sales_summary` example), which also happens to be a prerequisite for that MV to support `ON COMMIT` fast refresh in many cases.
- An **inline view** is typically too short-lived for indexing to be a meaningful concept at all — but if the same inline-view pattern is repeated across many queries and turns out to be a performance bottleneck, that's usually a strong signal it should be promoted into either a real materialized view or a properly indexed intermediate table.

# PART 3 — REAL-WORLD CASE STUDY

## Global Retail Inc. — A Composite Case Study

Global Retail Inc. runs an e-commerce platform. Their engineering team hits a series of real, escalating problems — and the fix for each one is exactly one of the concepts covered above.

**Problem 1 — "Every report team writes their own version of the same 5-table join, and they keep disagreeing with each other."**

Finance, Marketing, and Operations each independently join `orders`, `order_items`, `customers`, `products`, and `employees` to calculate "net revenue by region," and each team's number is slightly different because of subtly different filtering logic (one excludes returns, one doesn't; one uses order date, another uses ship date).

**Fix: a single, authoritative view.**

```sql
CREATE VIEW vw_net_revenue_by_region AS
SELECT c.region,
       o.order_date,
       SUM(oi.quantity * oi.unit_price) AS gross_amount,
       SUM(CASE WHEN o.status = 'RETURNED' THEN oi.quantity * oi.unit_price ELSE 0 END) AS returned_amount
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN customers c ON c.customer_id = o.customer_id
GROUP BY c.region, o.order_date;
```

Every team now queries `vw_net_revenue_by_region` directly. The business logic is defined once, by someone who actually understands the returns policy, and every downstream number agrees by construction.

**Problem 2 — "The executive dashboard takes 45 seconds to load because it re-aggregates 300 million order rows every time someone opens it."**

The dashboard runs a heavy `GROUP BY` over the entire `orders` history on every page load — acceptable for a one-off analysis, brutal for something 50 executives refresh throughout the day.

**Fix: a materialized view, refreshed on a schedule the business can live with.**

```sql
CREATE MATERIALIZED VIEW LOG ON orders WITH ROWID, SEQUENCE (customer_id, region, order_date, status) INCLUDING NEW VALUES;

CREATE MATERIALIZED VIEW mv_daily_revenue_by_region
BUILD IMMEDIATE
REFRESH FAST ON DEMAND
AS
SELECT region, TRUNC(order_date) AS order_day, COUNT(*) AS order_count, SUM(gross_amount) AS total_revenue
FROM vw_net_revenue_by_region
GROUP BY region, TRUNC(order_date);
```

A nightly `DBMS_SCHEDULER` job runs `DBMS_MVIEW.REFRESH('MV_DAILY_REVENUE_BY_REGION', 'F')` at 2 AM. The dashboard now reads a small, pre-aggregated table and loads in under a second — the trade-off (data as of last night, not this second) is one the business happily accepts for a daily revenue dashboard.

**Problem 3 — "Filtering products by category and in-stock status is slow, even though there's an index."**

`products.category` has 12 distinct values; `products.in_stock` is a boolean-like flag. A standard B-tree index on each barely helps, because each individual condition still matches a huge fraction of the table — the optimizer often ignores the B-tree indexes entirely and does a full scan anyway.

**Fix: bitmap indexes**, since this is a read-heavy catalog table, rarely updated by individual customers, with exactly the low-cardinality-multi-condition profile bitmap indexes are built for.

```sql
CREATE BITMAP INDEX idx_prod_category ON products(category);
CREATE BITMAP INDEX idx_prod_instock ON products(in_stock);
```

`WHERE category = 'ELECTRONICS' AND in_stock = 'Y'` now resolves via a single fast bitwise `AND` between the two bitmaps instead of a full scan.

**Problem 4 — "Looking up a single order by `order_id` on the checkout page is instant, but looking up all orders for a given `customer_id` is slow."**

`order_id` (the primary key) already has its automatic unique B-tree index. `customer_id`, a foreign key, has none.

**Fix:** a standard B-tree index on the foreign key column — the textbook use case.

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

**Problem 5 — "We just discovered two customer accounts with the exact same email address, and our password-reset flow sent the reset link to the wrong one."**

**Fix:** a formal unique constraint (backed automatically by a unique index) — this should never have been physically possible in the first place.

```sql
ALTER TABLE customers ADD CONSTRAINT uq_customer_email UNIQUE (email);
```

**Problem 6 — "Ever since the DBA team added 4 more indexes to speed up reporting queries, checkout (`INSERT INTO orders`) has gotten noticeably slower."**

This is the trade-off from section 2.1 showing up in production: every one of those reporting indexes must also be updated on every single new order insert. The fix isn't "remove all indexes" — it's recognizing that **heavily-queried reporting columns and heavily-written OLTP tables are in tension**, and the real answer, consistent with everything above, is to move that reporting workload onto a **materialized view** (Problem 2's fix) instead of indexing the live `orders` table for every possible report filter.

---

# PRACTICE EXERCISES

Work through these using the sample schema from the top of this guide. A small sample dataset is given for exercises where concrete output matters:

| employee_id | name | department | job_title | salary | hire_date | region | status |
|---|---|---|---|---|---|---|---|
| 101 | John Smith | IT | MANAGER | 95000 | 2019-03-15 | NORTH | ACTIVE |
| 102 | Priya Nair | IT | ANALYST | 72000 | 2020-07-01 | NORTH | ACTIVE |
| 103 | Carlos Ruiz | SALES | CLERK | 45000 | 2021-01-10 | SOUTH | ACTIVE |
| 104 | Aisha Bello | SALES | MANAGER | 88000 | 2018-11-20 | SOUTH | ACTIVE |
| 105 | Wei Zhang | HR | ANALYST | 61000 | 2022-02-14 | EAST | INACTIVE |
| 106 | Fatima Noor | FINANCE | CLERK | 50000 | 2021-09-05 | WEST | ACTIVE |
| 107 | David Cohen | IT | ANALYST | 76000 | 2020-12-01 | NORTH | ACTIVE |
| 108 | Sara Kim | FINANCE | MANAGER | 91000 | 2017-06-30 | WEST | ACTIVE |

---

### Exercise 1

**Task:** Create a view exposing only active employees' names, department, and hire date — no salary.

**Expected Output:**
```
View VW_ACTIVE_EMPLOYEES created.
```

**Answer:**
```sql
CREATE VIEW vw_active_employees AS
SELECT employee_id, name, department, hire_date
FROM employees
WHERE status = 'ACTIVE';
```
This satisfies the security advantage directly — anyone with access to this view can never see a salary figure, no matter what they query.

---

### Exercise 2

**Task:** Create a view of IT department employees with `WITH CHECK OPTION`, then attempt to move an employee out of IT through the view, and explain what happens.

**Expected Output:**
```
View VW_IT_EMPLOYEES created.

ORA-01402: view WITH CHECK OPTION where-clause violation
```

**Answer:**
```sql
CREATE VIEW vw_it_employees AS
SELECT employee_id, name, department, salary
FROM employees
WHERE department = 'IT'
WITH CHECK OPTION;

UPDATE vw_it_employees SET department = 'SALES' WHERE employee_id = 102;
```
The update is rejected because it would make the row vanish from the view's own defining `WHERE department = 'IT'` clause — exactly the scenario `WITH CHECK OPTION` exists to prevent.

---

### Exercise 3

**Task:** Using an inline view, find every department whose average salary exceeds ₹70,000.

**Expected Output:**
```
2 rows selected.
```

| DEPARTMENT | AVG_SALARY |
|---|---|
| IT | 81000.00 |
| FINANCE | 70500.00 |

**Answer:**
```sql
SELECT department, avg_salary
FROM (
    SELECT department, ROUND(AVG(salary), 2) AS avg_salary
    FROM employees
    GROUP BY department
)
WHERE avg_salary > 70000
ORDER BY avg_salary DESC;
```
IT averages ₹81,000 and Finance averages ₹70,500 — both clear the bar; Sales (₹66,500) and HR (₹61,000) don't.

---

### Exercise 4

**Task:** Explain, without running anything, whether the following view is updatable — and why:
```sql
CREATE VIEW vw_dept_headcount AS
SELECT department, COUNT(*) AS headcount FROM employees GROUP BY department;
```

**Expected Output:** *(a reasoning answer, not a query result)*

**Answer:** **Not updatable.** It uses both an aggregate function (`COUNT`) and `GROUP BY` — both are on the explicit list of constructs that disqualify a view from being updatable, because a single row in this view's result (e.g., "IT, 3") doesn't correspond to any single row in the base table at all; there's no sensible way to map an `UPDATE` on `headcount` back down into a change on individual employee rows.

---

### Exercise 5

**Task:** Create a materialized view storing department-wise average salary, built immediately, refreshed completely on demand. Then run a manual complete refresh.

**Expected Output:**
```
Materialized view MV_DEPT_AVG_SALARY created.

Complete refresh performed.
```

**Answer:**
```sql
CREATE MATERIALIZED VIEW mv_dept_avg_salary
BUILD IMMEDIATE
REFRESH COMPLETE ON DEMAND
AS
SELECT department, ROUND(AVG(salary), 2) AS avg_salary, COUNT(*) AS headcount
FROM employees
GROUP BY department;

EXEC DBMS_MVIEW.REFRESH('MV_DEPT_AVG_SALARY', 'C');
```

---

### Exercise 6

**Task:** The above MV needs to support a `FAST` refresh instead. What's missing, and how do you fix it?

**Expected Output:**
```
ORA-23413: table "HR"."EMPLOYEES" does not have a materialized view log
```

**Answer:** A **materialized view log** must first exist on the base table (`employees`) before any MV defined over it can be fast-refreshed:
```sql
CREATE MATERIALIZED VIEW LOG ON employees
WITH ROWID, SEQUENCE (department, salary)
INCLUDING NEW VALUES;

-- Now recreate (or the MV could have been defined this way from the start):
CREATE MATERIALIZED VIEW mv_dept_avg_salary
BUILD IMMEDIATE
REFRESH FAST ON DEMAND
AS
SELECT department, COUNT(*) AS headcount, SUM(salary) AS total_salary
FROM employees
GROUP BY department;
```
Note the rewritten query also swaps `AVG` for `SUM` + `COUNT` — a fast-refreshable aggregate MV generally needs the raw ingredients (`SUM`, `COUNT`) rather than `AVG` directly, precisely because `AVG` can't be incrementally adjusted from a single new row without also knowing the running count.

---

### Exercise 7

**Task:** A new employee, Meera Joshi, is hired into IT at ₹80,000. After the `INSERT` and a `COMMIT`, is `mv_dept_avg_salary` (from Exercise 6, `REFRESH FAST ON DEMAND`) automatically up to date?

**Expected Output:** *(reasoning answer)*

**Answer:** **No.** `ON DEMAND` means the MV only changes when someone explicitly calls `DBMS_MVIEW.REFRESH` (or a scheduled job does). The `COMMIT` on the base table has no automatic effect on this particular MV. If the requirement were "always reflect the latest hire immediately," the MV would need to be defined with `REFRESH FAST ON COMMIT` instead — which does carry the trade-off of adding refresh work directly onto every commit against `employees`.

---

### Exercise 8

**Task:** Create an appropriate index to speed up a report that frequently runs `WHERE hire_date BETWEEN :start AND :end`.

**Expected Output:**
```
Index IDX_EMP_HIRE_DATE created.
```

**Answer:**
```sql
CREATE INDEX idx_emp_hire_date ON employees(hire_date);
```
A standard B-tree index is the right choice here — `hire_date` is high-cardinality (dates rarely repeat exactly) and this is precisely the range-query pattern B-tree's linked leaf nodes are built to serve efficiently.

---

### Exercise 9 — Algorithm Trace

**Task:** Given this small B-tree (max 3 keys per leaf), trace the exact path taken to search for the key `47`:

```
                    [40]
                   /    \
            [20, 30]    [50, 60]
           /  |   \     /   |   \
       [..] [..] [..] [..] [40-49] [..]
```

**Expected Output:** *(a step-by-step trace)*

**Answer:**
1. **At the root `[40]`:** is `47` less than, equal to, or greater than `40`? Greater — follow the right child pointer.
2. **At branch `[50, 60]`:** is `47` less than `50`? Yes — follow the leftmost child pointer (everything less than `50`).
3. **Arrive at leaf `[40–49]`:** binary-search within this leaf for `47`. If present, its entry holds the ROWID; follow that ROWID directly to the actual table row.

Total cost: 3 block reads (root, one branch, one leaf) — regardless of whether the table behind this index has one thousand or one hundred million rows, as long as the tree's overall depth stays the same.

---

### Exercise 10 — Algorithm Trace (Insert with Split)

**Task:** A leaf node currently holds `[70, 75, 80]` (at its maximum capacity of 3). A new key, `72`, needs to be inserted. Walk through exactly what happens.

**Expected Output:** *(a step-by-step trace)*

**Answer:**
1. `72` belongs in this leaf, in sorted position: `[70, 72, 75, 80]` — but that's 4 keys, exceeding the max of 3. **Overflow.**
2. The leaf **splits** into two: roughly `[70, 72]` and `[75, 80]`.
3. The **first key of the new right leaf (`75`)** is copied up into the parent branch node as a new separating key, alongside a new pointer to the new right leaf.
4. If the parent branch node now also exceeds its own capacity from this insertion, the exact same split-and-push-up process repeats one level higher — and only if that cascades all the way to the root does the tree actually grow a new level.

---

### Exercise 11

**Task:** Create bitmap indexes to speed up a reporting query that filters `WHERE region = 'NORTH' AND status = 'ACTIVE'`, and explain in one sentence how Oracle answers this query once both indexes exist.

**Expected Output:**
```
Bitmap index IDX_EMP_REGION_BMP created.
Bitmap index IDX_EMP_STATUS_BMP created.
```

**Answer:**
```sql
CREATE BITMAP INDEX idx_emp_region_bmp ON employees(region);
CREATE BITMAP INDEX idx_emp_status_bmp ON employees(status);
```
Oracle takes the pre-built `NORTH` bitmap and the pre-built `ACTIVE` bitmap and performs a single bitwise `AND` between them, instantly identifying the exact set of matching rows without touching a full table scan or a B-tree traversal per condition.

---

### Exercise 12 — Reasoning Quiz

**Task:** For each column below, say whether a B-tree or a Bitmap index is the better fit, and why:

(a) `orders.order_id` (primary key, millions of unique values, heavy concurrent `INSERT`s from live checkout traffic)
(b) `orders.payment_method` (5 possible values: CARD, UPI, COD, WALLET, NETBANKING; table is read-heavy for reporting, updated in nightly batches only)
(c) `customers.date_of_birth` (high-cardinality, occasional range queries for age-based marketing segments)
(d) `support_tickets.priority` (3 values: LOW, MEDIUM, HIGH; table receives constant concurrent status updates all day from live support agents)

**Expected Output:** *(reasoning answers)*

**Answer:**
- **(a) B-tree.** High cardinality *and* heavy concurrent OLTP writes — the textbook B-tree case, and bitmap would be actively dangerous here regardless of cardinality concerns.
- **(b) Bitmap.** Low cardinality, read-heavy, batch-updated — exactly the safe, ideal bitmap scenario.
- **(c) B-tree.** High cardinality, and the workload described (range queries) is exactly what B-tree's linked leaves are built for.
- **(d) B-tree — despite the low cardinality.** This is the trap in the exercise: `priority` *looks* like a bitmap candidate on cardinality alone, but the table is under **constant concurrent single-row updates** from live agents — precisely the high-DML OLTP pattern where bitmap indexes cause severe lock contention. Cardinality alone never overrides the write-pattern rule.

---

### Exercise 13

**Task:** Explain the difference between these two statements, and which one you'd choose for a `customers.email` column that must never be duplicated:

```sql
-- Option A
ALTER TABLE customers ADD CONSTRAINT uq_email UNIQUE (email);
-- Option B
CREATE UNIQUE INDEX idx_email ON customers(email);
```

**Expected Output:** *(reasoning answer)*

**Answer:** Both physically prevent duplicate emails — Option A actually creates a backing unique index automatically under the hood, so their *enforcement* is nearly identical. The real difference is **documentation and intent**: Option A registers this as a formal, named business rule visible in `USER_CONSTRAINTS`, discoverable by ER-diagram tools, other developers, and referential-integrity-aware tooling. Option B enforces the same rule mechanically but doesn't communicate it as clearly as a declared constraint. **Option A is the better choice** for a rule that represents genuine business intent (which this clearly is) — reserve a bare unique index for narrower, more mechanical cases.

---

### Exercise 14

**Task:** A report frequently runs `WHERE department = :dept AND hire_date > :cutoff`, always filtering on both columns together, department first. Design the right index.

**Expected Output:**
```
Index IDX_EMP_DEPT_HIRE created.
```

**Answer:**
```sql
CREATE INDEX idx_emp_dept_hire ON employees(department, hire_date);
```
A composite index with `department` listed first, matching the query's filtering pattern — this single index efficiently serves both "department alone" queries and "department + hire_date range" queries, but would be far less useful if the column order were reversed, since `hire_date` alone isn't how this report ever filters.

---

### Exercise 15 — Troubleshooting

**Task:** A B-tree index exists on `employees.status`, but `EXPLAIN PLAN` shows Oracle is doing a full table scan anyway for `WHERE status = 'ACTIVE'`, even on a large table. Why might this be happening, and is this actually a bug?

**Expected Output:** *(reasoning answer)*

**Answer:** **Not a bug — the optimizer is very likely making the correct call.** `status` almost certainly has very low cardinality (`ACTIVE`/`INACTIVE`), and if the large majority of rows are `ACTIVE`, then using the B-tree index would mean: traverse the tree, retrieve a huge number of matching ROWIDs, then perform that many separate round-trips back to the table to fetch each row — which ends up costing *more* total I/O than simply reading the table sequentially once. This is precisely the scenario described in section 2.8 ("low-selectivity columns for a B-tree") — and it's also exactly the situation where a **bitmap index** (if the write pattern allows it) would genuinely help, since bitmap indexes don't suffer the same "too many matches" penalty the way B-tree row-by-row fetching does.

---

### Exercise 16 — Troubleshooting

**Task:** Since adding a bitmap index on `support_tickets.assigned_agent_id` (a foreign key with hundreds of distinct agents, in a table with constant concurrent status updates from live agents), the support team reports the whole ticketing system has become sluggish, with agents frequently stuck waiting. Diagnose the problem.

**Expected Output:** *(reasoning answer)*

**Answer:** This combines *two* bitmap-index mistakes from this guide at once: `assigned_agent_id` (hundreds of distinct values) is far too high-cardinality for an effective bitmap index in the first place, **and** this is a live, high-concurrency OLTP write path — exactly the profile flagged in section 2.4.4 as actively dangerous. Concurrent agents updating different, unrelated tickets are very likely locking each other out because their rows fall within the same compressed bitmap segment. The fix: drop the bitmap index and replace it with a standard **B-tree index** on `assigned_agent_id` — the correct structure for a high-cardinality foreign key under concurrent writes.

---

### Exercise 17

**Task:** Reports need to filter `WHERE UPPER(name) LIKE 'SMITH%'`, but a plain index on `name` doesn't help this query at all. Fix it.

**Expected Output:**
```
Index IDX_EMP_NAME_UPPER created.
```

**Answer:**
```sql
CREATE INDEX idx_emp_name_upper ON employees(UPPER(name));
```
A function-based index stores the *result of the expression* (`UPPER(name)`), not the raw column — so it can only ever help a query whose `WHERE` clause uses that exact same expression. A regular index on `name` alone is blind to a query wrapping the column in `UPPER()`, since the stored, sorted values in that index are the original mixed-case names, not the uppercased versions being searched for.

---

### Exercise 18 — Capstone

**Task:** Global Retail Inc.'s new `product_reviews` table has: `review_id` (PK), `product_id` (FK, high cardinality), `customer_id` (FK, high cardinality), `rating` (1–5, low cardinality), `is_verified_purchase` (Y/N flag), `review_text` (large free text), `review_date`. The table receives thousands of new reviews per hour from live customer traffic, and is also queried constantly by a public-facing "filter reviews by rating and verified-purchase status" widget on the product page. Design the complete indexing strategy, and justify each choice.

**Expected Output:** *(a complete design with justification)*

**Answer:**
```sql
-- Primary key: automatic unique B-tree, no action needed
-- ALTER TABLE product_reviews ADD CONSTRAINT pk_review PRIMARY KEY (review_id);

-- Foreign keys, high cardinality, frequently joined/filtered individually -> B-tree
CREATE INDEX idx_reviews_product_id  ON product_reviews(product_id);
CREATE INDEX idx_reviews_customer_id ON product_reviews(customer_id);

-- rating and is_verified_purchase are BOTH low-cardinality AND filtered together
-- by the public widget -- textbook bitmap case...
-- ...EXCEPT this table has heavy concurrent live-traffic INSERTs (new reviews),
-- which is exactly the disqualifying condition from section 2.4.4.
-- So: B-tree for both instead, despite the low cardinality, to protect write concurrency.
CREATE INDEX idx_reviews_rating   ON product_reviews(rating);
CREATE INDEX idx_reviews_verified ON product_reviews(is_verified_purchase);

-- Or, better still: a single composite B-tree index matching the widget's
-- actual combined filter pattern, avoiding the cost of maintaining two separate indexes:
-- CREATE INDEX idx_reviews_rating_verified ON product_reviews(rating, is_verified_purchase);
```

This exercise deliberately sets up the same trap as Exercise 12(d): `rating` and `is_verified_purchase` *look* like an obvious bitmap opportunity purely from their cardinality — but the stated write pattern (thousands of concurrent live inserts per hour) rules bitmap out entirely, regardless of how attractive the cardinality looks. Recognizing that the **write pattern overrides the cardinality signal** is the single most important judgment call this whole guide is building toward.

---

# QUICK REFERENCE / CHEAT SHEET

### View vs. Materialized View

| | View | Materialized View |
|---|---|---|
| Stores data | No | Yes |
| Freshness | Always live | As of last refresh |
| Speed | Same as base query | Fast (pre-computed) |
| Refresh needed | N/A | Complete / Fast / Force, On Demand / On Commit |

### B-Tree vs. Bitmap Index

| | B-Tree | Bitmap |
|---|---|---|
| Cardinality | High | Low |
| Workload | OLTP, concurrent writes | OLAP, read-heavy, batch-loaded |
| Structure | Sorted tree of (key, ROWID) | One compressed bit-vector per distinct value |
| Combining conditions | Merge row-id lists | Bitwise AND/OR — very fast |
| Danger zone | None special | High-concurrency DML → severe lock contention |

### Core Syntax At a Glance

```sql
-- Views
CREATE [OR REPLACE] [FORCE] VIEW view_name AS SELECT ... [WITH CHECK OPTION] [WITH READ ONLY];

-- Materialized Views
CREATE MATERIALIZED VIEW LOG ON table_name WITH ROWID, SEQUENCE (cols) INCLUDING NEW VALUES;
CREATE MATERIALIZED VIEW mv_name
  BUILD IMMEDIATE | DEFERRED
  REFRESH COMPLETE | FAST | FORCE
  ON DEMAND | ON COMMIT
  AS SELECT ...;
EXEC DBMS_MVIEW.REFRESH('MV_NAME', 'C' | 'F');

-- Indexes
CREATE INDEX idx_name ON table_name(col);                    -- B-tree (default)
CREATE UNIQUE INDEX idx_name ON table_name(col);              -- Unique
CREATE BITMAP INDEX idx_name ON table_name(col);               -- Bitmap
CREATE INDEX idx_name ON table_name(col1, col2);                -- Composite
CREATE INDEX idx_name ON table_name(UPPER(col));                  -- Function-based
ALTER INDEX idx_name COALESCE;                                     -- Lightweight cleanup
ALTER INDEX idx_name REBUILD [ONLINE];                               -- Full rebuild

-- Index-Organized Table (Oracle's "true clustered index")
CREATE TABLE t (id NUMBER PRIMARY KEY, ...) ORGANIZATION INDEX;
```

### The One Rule Worth Remembering Above All Others

**Cardinality tells you B-tree or Bitmap. Write pattern can override that answer. When in doubt, or under heavy concurrent DML, B-tree is always the safe default.**