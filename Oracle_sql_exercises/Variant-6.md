# Oracle SQL Certification Practice — Topic 6
## Views (Standard, Inline, Materialized) & Indexes (Architecture, B-Tree, Bitmap, Unique)

---

### Subtopic 1: What is a View? Advantages of Views

**Question 1**
What is a database VIEW, fundamentally?

A. A physical copy of a table's data, refreshed automatically whenever the base table changes.
B. A stored SELECT statement given a name, treated as a virtual table — querying the view re-executes its underlying query against the base table(s) rather than reading pre-stored data of its own.
C. A backup mechanism for recovering deleted rows.
D. A synonym for an index.

**Correct Answer:** B

**Detailed Explanation:**
A standard (non-materialized) view stores no data of its own — it is purely a named, reusable query definition. Every time the view is queried, Oracle transparently executes the stored SELECT statement against the underlying base table(s) to produce the result. This is the key conceptual distinction from a materialized view (Subtopic 3), which genuinely does store data physically.

**Why the Other Options Are Wrong:**
A. Describes a materialized view, not an ordinary view.
C. Views have no backup/recovery function.
D. Views and indexes are entirely different object types serving different purposes.

**Concept Tested:** The virtual (query-only, no stored data) nature of a standard view.

**Exam Trap:** Confusing "view" in general with "materialized view" specifically — this distinction underlies nearly every question in this topic.

---

**Question 2**
Which two are genuine **advantages** of using views? **(Choose two)**

A. Views can restrict which columns and/or rows a particular user or application sees, providing a form of security/data-hiding without needing separate physical copies of restricted data.
B. Views always execute faster than querying the base tables directly, regardless of the underlying query's complexity.
C. Views can simplify complex, multi-table queries by packaging them under a single, reusable name, insulating applications from the underlying query's complexity or even future structural changes to the base tables.
D. Views eliminate the need for any indexing on the base tables.

**Correct Answers:** A and C

**Detailed Explanation:**
Views are commonly used for security (e.g., a view that excludes a SALARY column entirely, or filters rows to only a user's own department) and for simplification/abstraction (hiding a complicated join or calculation behind a simple `SELECT * FROM my_view`), while also providing a layer of insulation if the underlying schema changes (the view can be redefined without necessarily breaking every application query that references it).

**Why the Other Options Are Wrong:**
B. A view has no inherent performance advantage — it still executes the same underlying query each time; performance depends entirely on that query and the base tables' indexing, not on the view wrapper itself.
D. Views have no bearing on whether the base tables need indexes — indexing needs are unchanged.

**Concept Tested:** The genuine advantages of views — security/restriction and simplification/abstraction — versus commonly assumed but incorrect "benefits" like automatic performance gains.

**Exam Trap:** Assuming a view is inherently a performance optimization, when it is purely a query-definition convenience/security layer (that distinction belongs to materialized views instead).

---

**Question 3**
What does the `WITH CHECK OPTION` clause do when creating a view?

A. It prevents any DML (INSERT/UPDATE/DELETE) from being performed through the view at all.
B. It ensures that any INSERT or UPDATE performed *through* the view cannot create or modify a row in a way that would make that row disappear from the view's own defining WHERE condition — in other words, you can't use the view to sneak in a row that the view itself wouldn't be able to subsequently show you.
C. It automatically creates a CHECK constraint on the underlying base table matching the view's WHERE clause.
D. It has no effect on DML; it only restricts SELECT queries.

**Correct Answer:** B

**Detailed Explanation:**
For example, if a view is defined as `... WHERE department_id = 50`, `WITH CHECK OPTION` blocks an INSERT or UPDATE through that view that would set `department_id` to anything other than 50 — since such a row, once committed, would immediately vanish from the view's own result set, which Oracle considers an inconsistent/disallowed operation.

**Why the Other Options Are Wrong:**
A. That's the effect of `WITH READ ONLY`, a separate, stricter option (see Question 4).
C. No base-table constraint is created; this is purely a DML-time check enforced at the view level.
D. It specifically targets INSERT/UPDATE consistency with the view's own WHERE condition — it is a DML-focused option, not a SELECT restriction.

**Concept Tested:** WITH CHECK OPTION's specific role — preventing DML from producing rows inconsistent with the view's own filtering condition.

**Exam Trap:** Confusing WITH CHECK OPTION (allows DML, but constrains its effect to stay consistent with the view's WHERE clause) with WITH READ ONLY (blocks DML entirely).

---

**Question 4**
Which option, when creating a view, prevents **any** INSERT, UPDATE, or DELETE from being performed through it?

A. `WITH CHECK OPTION`
B. `WITH READ ONLY`
C. `WITH NO DML`
D. `FORCE`

**Correct Answer:** B

**Detailed Explanation:**
`WITH READ ONLY` is the strict, blanket restriction — no DML of any kind is permitted through the view, regardless of whether it would otherwise satisfy the view's WHERE condition.

**Why the Other Options Are Wrong:**
A. Still permits DML, just with the added consistency constraint described in Question 3.
C. Not real Oracle syntax.
D. `FORCE` is used when creating a view whose base table(s) don't yet exist or whose defining query has other issues Oracle would normally reject — it's unrelated to DML restriction.

**Concept Tested:** WITH READ ONLY as the complete DML-blocking option.

**Exam Trap:** Assuming WITH CHECK OPTION and WITH READ ONLY are simply two intensities of the same restriction, rather than recognizing that CHECK OPTION still permits DML (under a constraint) while READ ONLY forbids it entirely.

---

**Question 5**
Which of these view characteristics makes it **non-updatable** (i.e., DML cannot be performed through it at all, regardless of any WITH options)?

A. The view's defining query includes a GROUP BY clause, a DISTINCT keyword, or an aggregate function.
B. The view is based on more than one table.
C. The view's defining query includes a WHERE clause.
D. The view was created with `WITH READ ONLY`.

**Correct Answer:** A

**Detailed Explanation:**
Views built on GROUP BY, DISTINCT, aggregate functions, set operators (UNION/INTERSECT/MINUS), or hierarchical queries (CONNECT BY/START WITH) are inherently non-updatable — there's no sensible way to map a DML operation on a summarized/deduplicated/combined row back to a single, specific underlying base-table row.

**Why the Other Options Are Wrong:**
B. Multi-table (join) views *can* be updatable, subject to additional "key-preserved table" rules — it isn't automatically disqualifying on its own.
C. A WHERE clause doesn't prevent updatability; it just restricts which rows the view shows (and interacts with WITH CHECK OPTION, per Question 3).
D. WITH READ ONLY is a deliberate, explicit restriction chosen at creation time — it's a different mechanism from the view's *structural* updatability, which is what this question is asking about.

**Concept Tested:** Structural (query-shape-based) reasons a view is inherently non-updatable, independent of any explicit WITH READ ONLY declaration.

**Exam Trap:** Assuming any multi-table view is automatically non-updatable, when the real disqualifying factors are specifically GROUP BY/DISTINCT/aggregates/set operators/hierarchical clauses.

---

### Subtopic 2: Inline View

**Question 1**
What is an **inline view**?

A. A named, permanent database object created with `CREATE VIEW`.
B. A subquery placed directly in a SELECT statement's FROM clause, functioning as a temporary, on-the-fly "virtual table" that exists only for the duration of that single statement — it is never stored in the data dictionary as its own object.
C. A materialized view that refreshes instantly ("inline") with every base-table change.
D. A view that can only return a single row.

**Correct Answer:** B

**Detailed Explanation:**
An inline view is simply a subquery used as a row source in the FROM clause — Oracle treats its result set as a temporary virtual table for that one query's execution, but it has no persistent existence afterward, unlike a `CREATE VIEW` object.

**Why the Other Options Are Wrong:**
A. Describes a standard, named (persistent) view instead.
C. Conflates "inline" (syntactic placement) with materialized-view refresh timing — unrelated concepts.
D. No row-count restriction is inherent to inline views.

**Concept Tested:** Inline view as a transient, statement-scoped FROM-clause subquery, as opposed to a persistent named view.

**Exam Trap:** Assuming "inline" refers to some special refresh or storage behavior, rather than simply describing where and how long the subquery exists.

---

**Question 2**
Which is a common, practical reason to use an inline view?

A. To permanently store a filtered subset of a table for other users to query later.
B. To pre-aggregate, pre-filter, or pre-rank a result set (e.g., computing per-department averages, or selecting a top-N slice via ORDER BY + ROWNUM/FETCH) *before* joining that intermediate result to another table in the same statement.
C. To bypass all constraint checking during an INSERT.
D. To avoid ever having to write a WHERE clause.

**Correct Answer:** B

**Detailed Explanation:**
A very common pattern: `SELECT e.last_name, dept_avg.avg_sal FROM employees e JOIN (SELECT department_id, AVG(salary) AS avg_sal FROM employees GROUP BY department_id) dept_avg ON e.department_id = dept_avg.department_id;` — the inline view computes per-department averages first, then that intermediate result is joined back to the detail rows.

**Why the Other Options Are Wrong:**
A. That's a use case for a persistent named view, not an inline view (which has no persistence).
C. Inline views have no effect on constraint enforcement.
D. Inline views are commonly used *alongside* WHERE clauses (both inside the inline view and in the outer query), not as a way to avoid them.

**Concept Tested:** The practical "pre-compute, then join" use case for inline views.

**Exam Trap:** Not recognizing the specific, very common "aggregate-then-join" or "rank-then-filter" pattern as the hallmark use case for inline views.

---

**Question 3**
Is an alias required for an inline view in Oracle?

A. No — inline views can be left unaliased; Oracle infers a default name automatically.
B. Yes — Oracle requires an explicit alias for a FROM-clause subquery; omitting one results in a syntax error.
C. Only if the inline view is joined to another table; a standalone inline view needs no alias.
D. Aliases are optional, but only for inline views that don't use GROUP BY.

**Correct Answer:** B

**Detailed Explanation:**
Unlike some other RDBMS platforms that tolerate unaliased derived tables, Oracle requires you to explicitly alias any subquery used as a row source in the FROM clause. Omitting this alias produces a syntax error.

**Why the Other Options Are Wrong:**
A, C, D. All wrongly claim the alias is optional in some circumstance — Oracle's requirement is unconditional for FROM-clause subqueries.

**Concept Tested:** Oracle's mandatory-alias requirement for inline views.

**Exam Trap:** Assuming alias requirements are lenient/optional, as they might be for some other database systems, and being caught off guard by the resulting syntax error in Oracle specifically.

---

**Question 4**
Can an inline view itself contain another inline view nested within its own FROM clause?

A. No — inline views can only reference base tables directly, never another inline view.
B. Yes — inline views can be nested to arbitrary practical depth, exactly like ordinary subqueries; an inline view's FROM clause is just another FROM clause, which can itself contain another subquery.
C. Yes, but only to exactly one additional level of nesting.
D. No — nesting inline views requires the WITH clause (subquery factoring) instead.

**Correct Answer:** B

**Detailed Explanation:**
There's nothing special preventing an inline view's own FROM clause from containing yet another inline view — it's simply ordinary SQL nesting, following the same general lack of a small nesting-depth limit already established for subqueries generally (Topic 5).

**Why the Other Options Are Wrong:**
A, C. Both invent restrictions that don't exist.
D. The WITH clause (subquery factoring) is a *different, alternative* technique for the same general goal (naming/reusing a subquery), not a prerequisite for nesting inline views.

**Concept Tested:** Inline views can be nested without a small artificial depth limit.

**Exam Trap:** Assuming inline views are somehow more restricted than ordinary subqueries when it comes to nesting depth.

---

**Question 5**
How does an inline view differ from a **WITH clause (subquery factoring)** definition used for the same underlying subquery logic?

A. There is no difference; they are exactly the same syntax with a different name.
B. A WITH clause names a subquery once at the top of the statement and allows that same named result to be referenced multiple times within the rest of the statement, potentially avoiding repeated (re-)execution of identical subquery logic; an inline view's subquery text must be fully repeated wherever it's needed if used more than once.
C. Inline views can only be used in SELECT statements; WITH clauses can only be used in DML statements.
D. WITH clauses cannot include a WHERE condition; inline views can.

**Correct Answer:** B

**Detailed Explanation:**
The WITH clause (subquery factoring, `WITH alias_name AS (SELECT ...) SELECT ... FROM alias_name ...`) is designed precisely for reuse — define the logic once, reference the resulting named result set as many times as needed in the main query, potentially even letting Oracle materialize/optimize it once rather than recomputing it per reference. An inline view, by contrast, is written directly at its point of use in the FROM clause and would need to be copy-pasted again if the same logic were needed a second time within the same statement.

**Why the Other Options Are Wrong:**
A. Understates a real, practically important difference (reusability).
C. Both constructs are usable within SELECT statements (and DML statements that contain a SELECT component); this option fabricates a restriction.
D. Both fully support WHERE conditions inside their subquery definitions.

**Concept Tested:** The reusability/naming advantage of WITH-clause subquery factoring over repeated inline-view text.

**Exam Trap:** Treating inline views and WITH-clause subqueries as pure syntax synonyms, missing the practical reuse distinction between them.

---

### Subtopic 3: Materialized View & Refreshing Techniques

**Question 1**
How does a **materialized view** fundamentally differ from an ordinary (non-materialized) view?

A. A materialized view physically stores its query's result set on disk, like a real table, whereas an ordinary view stores no data at all and simply re-executes its defining query on each reference.
B. A materialized view can only be based on a single table; ordinary views can join multiple tables.
C. A materialized view cannot be indexed; ordinary views can.
D. There is no real difference — "materialized view" is just another name for a regular view.

**Correct Answer:** A

**Detailed Explanation:**
This is the fundamental distinction underlying this entire subtopic: a materialized view genuinely persists its computed result set as physical storage (much like a table), which is precisely why it needs a **refresh mechanism** to stay synchronized with its underlying base tables as they change — a need that simply doesn't exist for an ordinary view, which is always automatically "current" since it recomputes from scratch every time it's queried.

**Why the Other Options Are Wrong:**
B. Materialized views fully support multi-table (join) defining queries, just like ordinary views.
C. Materialized views can absolutely be indexed, exactly like ordinary tables.
D. Directly contradicted by option A's genuine, fundamental distinction.

**Concept Tested:** The core storage distinction between materialized and ordinary views, and why only materialized views require refreshing.

**Exam Trap:** Assuming "materialized" is simply a synonym or a performance-tuning label applied to an ordinary view, rather than recognizing it as a structurally different kind of object requiring active data-synchronization.

---

**Question 2**
Which refresh method rebuilds the materialized view's entire contents from scratch, by fully re-executing its defining query?

A. FAST
B. COMPLETE
C. FORCE
D. INCREMENTAL

**Correct Answer:** B

**Detailed Explanation:**
COMPLETE refresh discards the materialized view's current contents and completely re-runs the defining query, populating it from zero — simple and always possible, but potentially expensive for large result sets.

**Why the Other Options Are Wrong:**
A. FAST applies only the *changes* since the last refresh (see Question 3), not a full rebuild.
C. FORCE is a *strategy* (try FAST first, fall back to COMPLETE if FAST isn't possible), not a distinct refresh mechanism of its own.
D. Not a real Oracle materialized-view refresh method name.

**Concept Tested:** COMPLETE refresh as the full, from-scratch rebuild method.

**Exam Trap:** Confusing FORCE (a fallback strategy) with a genuinely distinct third refresh mechanism, rather than understanding it as a wrapper around FAST/COMPLETE.

---

**Question 3**
What must exist on the base table(s) for a **FAST** refresh to be possible?

A. Nothing special — FAST refresh always works on any materialized view, regardless of base table setup.
B. A materialized view log must be created on each relevant base table — this log captures incremental changes (inserts/updates/deletes) since the last refresh, which FAST refresh then applies to the materialized view instead of recomputing everything from scratch.
C. The materialized view must be based on a single table with no joins.
D. The base table must have a bitmap index defined on every column.

**Correct Answer:** B

**Detailed Explanation:**
A materialized view log (`CREATE MATERIALIZED VIEW LOG ON base_table ...`) is a small auxiliary object that tracks row-level changes to the base table. FAST refresh reads this log to apply only the delta since the last refresh, rather than recomputing the entire result — much more efficient for large materialized views with relatively small amounts of change between refreshes.

**Why the Other Options Are Wrong:**
A. FAST refresh has real prerequisites; it does not "always work."
C. Multi-table (join) materialized views can support FAST refresh too, given appropriate materialized view logs on each participating base table (and other eligibility conditions).
D. Bitmap indexes are unrelated to FAST refresh eligibility.

**Concept Tested:** The materialized view log prerequisite for FAST refresh.

**Exam Trap:** Assuming FAST refresh is simply "the quick option" that's always available, without understanding its actual structural prerequisite (the log).

---

**Question 4**
What happens if you request a **FAST** refresh (specifically, not FORCE) on a materialized view whose base table has no materialized view log?

A. Oracle automatically creates the missing log on the fly and proceeds with a FAST refresh.
B. The refresh fails with an error, since FAST refresh has no fallback behavior — it either applies the incremental log-based method or fails outright.
C. Oracle silently performs a COMPLETE refresh instead, with no error or warning.
D. The materialized view is automatically dropped.

**Correct Answer:** B

**Detailed Explanation:**
Unlike FORCE (which gracefully falls back to COMPLETE if FAST isn't possible), explicitly requesting FAST is a strict request — if the prerequisite log doesn't exist, Oracle cannot perform the incremental method and raises an error rather than silently substituting a different refresh approach.

**Why the Other Options Are Wrong:**
A. Oracle never auto-creates a materialized view log on your behalf.
C. That graceful, no-error fallback specifically describes FORCE, not a plain FAST request.
D. Failing a refresh has no bearing on the materialized view object's existence — it remains, simply un-refreshed (still containing its prior data).

**Concept Tested:** FAST's strict (no-fallback) behavior versus FORCE's graceful fallback — reinforcing the practical difference between these two refresh choices.

**Exam Trap:** Conflating FAST and FORCE's behavior when a log is missing — only FORCE has the safety-net fallback; explicit FAST does not.

---

**Question 5**
What is the difference between `REFRESH ON COMMIT` and `REFRESH ON DEMAND` for a materialized view?

A. ON COMMIT automatically refreshes the materialized view as part of every transaction that commits changes to its base table(s); ON DEMAND requires an explicit, manually-triggered refresh (e.g., via `DBMS_MVIEW.REFRESH` or a scheduled job) and does not happen automatically.
B. ON COMMIT refreshes only once per day; ON DEMAND refreshes continuously in real time.
C. They are functionally identical; the keywords are interchangeable.
D. ON DEMAND is the only option available in Oracle; ON COMMIT does not exist.

**Correct Answer:** A

**Detailed Explanation:**
ON COMMIT ties the refresh directly to transaction boundaries on the base tables — every time a relevant transaction commits, the materialized view is kept synchronized automatically. ON DEMAND decouples refreshing entirely from base-table transactions, leaving it to be triggered explicitly whenever convenient (which is often preferable for expensive refreshes that shouldn't slow down every single base-table transaction).

**Why the Other Options Are Wrong:**
B. Neither timing description is accurate — ON COMMIT is transaction-triggered (not daily), and ON DEMAND has no "continuous real-time" behavior (it's explicitly triggered, whenever that may be).
C. They represent genuinely different, mutually exclusive refresh-triggering strategies.
D. Both are real, well-documented Oracle options.

**Concept Tested:** ON COMMIT (automatic, transaction-tied) vs. ON DEMAND (manual/scheduled) refresh triggering.

**Exam Trap:** Assuming ON COMMIT refreshes on some fixed time schedule rather than being tied specifically to transaction commit events on the base table(s).

---

### Subtopic 4: What is an Index? Index Architecture — Clustered vs. Non-Clustered

**Question 1**
What is the primary purpose of a database index?

A. To enforce referential integrity between two tables.
B. To provide a faster access path for locating rows matching a given condition, at the cost of additional storage and some overhead on INSERT/UPDATE/DELETE operations that must keep the index synchronized with the table.
C. To automatically compress table data to save disk space.
D. To prevent duplicate rows from ever being inserted into a table.

**Correct Answer:** B

**Detailed Explanation:**
Indexes trade storage space and DML overhead for faster read performance — instead of scanning every row of a table, Oracle can use the index's ordered structure to jump directly (or nearly directly) to matching rows.

**Why the Other Options Are Wrong:**
A. That's the role of FOREIGN KEY constraints (Topic 2), not indexes generally (though FK columns are often manually indexed for performance reasons, as covered in Topic 2).
C. Indexes are not a compression mechanism.
D. That's specifically the role of a UNIQUE index/constraint (Subtopic 7) — an ordinary (non-unique) index has no such enforcement role.

**Concept Tested:** The fundamental read-speed-vs-write-overhead tradeoff that defines what an index is for.

**Exam Trap:** Attributing constraint-enforcement responsibilities (referential integrity, uniqueness) to indexes in general, when those are properties of specific constraint types (some of which happen to be backed by indexes).

---

**Question 2**
True or False: In a standard Oracle heap-organized table, the physical storage order of rows on disk automatically matches the sorted order of any B-tree index defined on that table.

A. True
B. False

**Correct Answer:** B (False)

**Detailed Explanation:**
An ordinary Oracle table (heap-organized table) stores rows with no guaranteed physical ordering relative to any index. The B-tree index is a completely separate structure, containing sorted key values paired with ROWIDs that point back to wherever the actual row physically happens to reside — the table's row storage itself is untouched and unordered by the index's existence.

**Concept Tested:** The physical independence of a heap-organized table's row storage from any index built on it.

**Exam Trap:** Importing a "clustered index" mental model from other database systems (where the table data itself is physically reordered/stored according to the clustering key) and assuming Oracle's default table structure behaves the same way.

---

**Question 3**
What is Oracle's closest structural equivalent to what other database systems (e.g., SQL Server) call a "clustered index" — where the table's actual data is physically stored in index order?

A. A standard B-tree index on a heap-organized table.
B. An Index-Organized Table (IOT) — here, the table's actual row data is stored directly within the B-tree structure itself, physically ordered by the table's primary key, rather than in a separate heap table pointed to by a separate index.
C. A bitmap index.
D. Oracle has no equivalent concept whatsoever.

**Correct Answer:** B

**Detailed Explanation:**
An Index-Organized Table fuses the table and its primary-key index into a single physical structure — there's no separate heap segment at all; the "table" *is* the B-tree, sorted by primary key. This directly parallels the "data physically ordered by key" property that defines a clustered index elsewhere.

**Why the Other Options Are Wrong:**
A. Describes the standard, non-clustered-style relationship (separate heap table + separate index) — the opposite of what's being asked.
C. Bitmap indexes are an entirely different indexing strategy (Subtopic 6), unrelated to physical row-storage ordering.
D. Oracle does have a genuine structural equivalent — IOT.

**Concept Tested:** Index-Organized Tables as Oracle's answer to the "clustered index" concept from other platforms.

**Exam Trap:** Assuming Oracle simply lacks this concept entirely (since it doesn't use the term "clustered index"), rather than recognizing IOT as the terminology-translated equivalent.

---

**Question 4**
How does an Oracle **table CLUSTER** (created with `CREATE CLUSTER`) differ from the "clustered index" concept discussed in Question 3?

A. They are exactly the same thing, just with different names.
B. A table cluster physically stores rows from **multiple different tables** together in the same data blocks, grouped by a shared cluster key value (e.g., storing all EMPLOYEES rows for a given department physically near that department's own DEPARTMENTS row) — this is a distinct Oracle-specific feature for co-locating related rows across tables, not a single-table index-ordering technique like an IOT.
C. A table cluster is simply Oracle's marketing term for a bitmap index.
D. A table cluster can only be created on views, never on tables.

**Correct Answer:** B

**Detailed Explanation:**
This is an important terminology disambiguation: Oracle's own "cluster" feature is about physically co-locating related rows *across different tables* sharing a common key, to speed up joins between them — a fundamentally different goal from IOT's single-table, primary-key-ordered storage.

**Why the Other Options Are Wrong:**
A. Conflates two genuinely distinct Oracle features that happen to share a name-adjacent term ("cluster" vs. "clustered").
C. Bitmap indexes are unrelated to table clusters.
D. Table clusters apply to tables, not views.

**Concept Tested:** Distinguishing Oracle's own CREATE CLUSTER feature (multi-table physical co-location) from the "clustered index" terminology borrowed from other database systems (which maps to IOT).

**Exam Trap:** Being misled by overlapping vocabulary ("cluster," "clustered") into merging two genuinely different Oracle concepts that happen to share similar-sounding names.

---

**Question 5**
How many primary-key-ordered "index-organized" structures can a single table have, given that IOT status is a structural property of the table itself?

A. Unlimited — a table can be simultaneously index-organized around any number of different key combinations.
B. At most one — a table is either created as an ordinary heap-organized table, or as an Index-Organized Table structured specifically around its (single) primary key; you cannot have multiple, independent "clustered" physical orderings of the same table's data simultaneously.
C. Exactly two — one for the primary key, one for a designated secondary "clustering" column.
D. Zero — Oracle does not support this feature at all.

**Correct Answer:** B

**Detailed Explanation:**
Whether a table is index-organized (IOT) is decided once, at `CREATE TABLE ... ORGANIZATION INDEX` time, and there's exactly one primary-key-ordered physical arrangement possible for the table's actual row data — unlike ordinary secondary indexes, of which a table can have many, only one physical row-storage arrangement can exist at a time.

**Why the Other Options Are Wrong:**
A, C. Both invent multiplicity that doesn't exist for the table's single physical storage arrangement.
D. Directly contradicted by Question 3's confirmation that IOT is a real, supported Oracle feature.

**Concept Tested:** A table's physical storage organization (heap vs. index-organized) is a single, exclusive structural choice, unlike the many independent secondary indexes a table can otherwise have.

**Exam Trap:** Confusing "how many secondary indexes can a table have" (many, unrestricted) with "how many physical storage organizations can a table have" (exactly one).

---

### Subtopic 5: B-Tree Index

**Question 1**
What is the general structure of an Oracle B-tree index?

A. A single flat, unsorted list of ROWIDs.
B. A balanced tree with a root node, intermediate branch nodes, and leaf nodes; the leaf nodes store the indexed column value(s) alongside the corresponding ROWID(s), and are linked together (as a doubly-linked list) in sorted key order to efficiently support range scans.
C. A hash table mapping each indexed value directly to exactly one row, with no ordering at all.
D. A random binary tree with no rebalancing.

**Correct Answer:** B

**Detailed Explanation:**
This is the standard, foundational B-tree structure: balanced (so lookup cost stays predictable regardless of tree size), with leaf-level entries sorted and linked, enabling Oracle to efficiently walk sequentially through a range of key values once it locates the starting point.

**Why the Other Options Are Wrong:**
A. Lacks the ordering/structure that makes B-trees efficient for range queries.
C. Describes a hash-based structure, not a B-tree — and doesn't match Oracle's default B-tree implementation.
D. B-trees are specifically *balanced* by design and maintenance, unlike an arbitrary/unbalanced binary tree.

**Concept Tested:** The foundational root/branch/leaf, sorted-and-linked B-tree structure.

**Exam Trap:** Confusing B-tree's balanced, sorted design with a simpler (and less capable) unordered or hash-based structure.

---

**Question 2**
True or False: A standard, single-column B-tree index contains an entry for **every** row of the indexed table, including rows where the indexed column's value is NULL.

A. True
B. False

**Correct Answer:** B (False)

**Detailed Explanation:**
This is a significant, frequently-tested Oracle-specific behavior: a B-tree index does **not** create an entry for a row where the (single) indexed column's value is NULL. Since there's no value to index, Oracle simply omits that row from the index structure entirely.

**Concept Tested:** B-tree indexes exclude entries for NULL-valued (single-column) indexed rows.

**Exam Trap:** Assuming every row is always represented in every index, regardless of value — a natural but incorrect assumption that directly affects query-plan reasoning (see Question 3).

---

**Question 3**
Given the previous question's fact, what is the practical consequence for a query like `SELECT * FROM employees WHERE commission_pct IS NULL;`, assuming `commission_pct` has a single-column B-tree index and no other applicable index exists?

A. The optimizer can use the B-tree index directly and efficiently to locate exactly the NULL rows.
B. The B-tree index is generally **not useful** for this specific query, since it contains no entries at all for NULL-valued rows — Oracle typically must resort to a full table scan (or another mechanism) to find these rows instead.
C. The query fails with an error, since B-tree indexes cannot coexist with IS NULL predicates.
D. Oracle automatically converts the B-tree index to a bitmap index on the fly to handle this case.

**Correct Answer:** B

**Detailed Explanation:**
Since NULL rows have no representation in the index (per Question 2), there's nothing for the index to "point to" for an IS NULL search — the optimizer generally can't use this index to satisfy that particular predicate and instead falls back to a full table scan (unless some other structure, like a bitmap index or a function-based index specifically designed around this case, is available).

**Why the Other Options Are Wrong:**
A. Directly contradicted by the absence of NULL entries in the index.
C. No such error occurs; the query runs successfully, just without leveraging this particular index for this particular predicate.
D. No such automatic, on-the-fly index-type conversion happens.

**Concept Tested:** The practical query-performance implication of B-tree's NULL-exclusion behavior.

**Exam Trap:** Not connecting the abstract "B-tree skips NULL rows" fact to its very concrete consequence for IS NULL query performance — a classic two-step certification trap.

---

**Question 4**
For a **composite** (multi-column) B-tree index on `(col_a, col_b)`, under what condition does Oracle skip creating an index entry for a given row?

A. If either col_a or col_b (at least one) is NULL for that row.
B. Only if **both** col_a and col_b are NULL for that row simultaneously — if even one of the two columns has a non-null value, an entry is still created (with a NULL placeholder for whichever column actually is null).
C. Composite indexes always index every row, regardless of NULLs in any column.
D. Composite indexes never allow NULL values in any column to begin with.

**Correct Answer:** B

**Detailed Explanation:**
The NULL-skipping rule generalizes to composite indexes as "skip only if every column in the key is null" — not "skip if any column is null." This is a materially different (and more permissive) rule than the single-column case might suggest, and it's a frequently tested extension of the NULL-indexing concept.

**Why the Other Options Are Wrong:**
A. Overstates the skipping condition — a partially-null composite key still gets indexed.
C. Understates it — full-NULL composite rows genuinely are skipped.
D. Composite index columns can individually be NULL; there's no blanket prohibition.

**Concept Tested:** The "skip only if ALL composite key columns are NULL" rule for B-tree indexes.

**Exam Trap:** Overgeneralizing the single-column NULL-skipping rule ("any NULL means no entry") to composite indexes, where the actual rule requires *every* column to be NULL.

---

**Question 5**
B-tree indexes are generally best suited for which kind of column?

A. Low-cardinality columns with only a handful of distinct values (e.g., a GENDER column with just 'M'/'F').
B. High-cardinality columns with many distinct values (e.g., EMPLOYEE_ID or EMAIL), especially for equality and range-based lookups typical of OLTP (transactional) workloads.
C. Columns that are never queried with an equality or range condition.
D. Columns that are updated extremely frequently by many concurrent transactions, regardless of cardinality.

**Correct Answer:** B

**Detailed Explanation:**
B-tree indexes shine when there are many distinct values to distinguish between (high cardinality), letting the tree efficiently narrow down to a small number of matching rows — precisely the profile of typical OLTP lookups (find one specific employee, find orders in a date range, etc.). This sets up the direct contrast with bitmap indexes (Subtopic 6), which favor the opposite (low-cardinality) profile.

**Why the Other Options Are Wrong:**
A. Low-cardinality columns are the domain where bitmap indexes typically outperform B-tree (Subtopic 6).
C. Indexes are specifically valuable *because* of equality/range querying; a column never filtered this way gains little benefit from any index.
D. Heavy concurrent DML volume is a separate consideration from cardinality, and is actually more of a concern for bitmap indexes' locking behavior (Subtopic 6) than for B-tree.

**Concept Tested:** B-tree's suitability profile — high cardinality, OLTP-style equality/range access.

**Exam Trap:** Not yet drawing the cardinality-based contrast between B-tree and bitmap indexes, which becomes the central theme of the next subtopic.

---

### Subtopic 6: Bitmap Index

**Question 1**
For which kind of column is a **bitmap index** typically the better choice?

A. High-cardinality columns like EMPLOYEE_ID.
B. Low-cardinality columns with relatively few distinct values (e.g., GENDER, MARITAL_STATUS, a coded REGION column), especially common in read-heavy/data-warehouse environments.
C. Columns that are the target of a PRIMARY KEY constraint.
D. Columns subject to constant, high-volume single-row INSERT/UPDATE/DELETE from many concurrent OLTP transactions.

**Correct Answer:** B

**Detailed Explanation:**
Bitmap indexes store one compact bitmap per distinct value, with one bit per row indicating whether that row holds that value — this representation is extremely space-efficient and fast to combine (via bitwise operations) specifically when there are few distinct values to represent, which is exactly the opposite profile from B-tree's ideal case.

**Why the Other Options Are Wrong:**
A. High-cardinality columns are B-tree's strength, not bitmap's — a bitmap index on a column like EMPLOYEE_ID would need one bitmap per unique ID, defeating the entire compactness advantage.
C. PRIMARY KEY constraints are backed by unique B-tree indexes automatically (Topic 2) — bitmap indexes are not used for this purpose.
D. This describes an OLTP-heavy write profile — a scenario bitmap indexes handle poorly (see Question 5).

**Concept Tested:** Bitmap index's low-cardinality sweet spot, directly contrasting with B-tree's high-cardinality strength.

**Exam Trap:** Reversing the cardinality preference between B-tree and bitmap indexes — one of the most fundamental facts tested across both index types.

---

**Question 2**
True or False: Unlike a standard single-column B-tree index, a bitmap index **does** create an entry (specifically, its own bitmap) representing rows where the indexed column's value is NULL.

A. True
B. False

**Correct Answer:** A (True)

**Detailed Explanation:**
This is a genuinely important contrast with Subtopic 5's B-tree behavior: bitmap indexes treat NULL as just another distinct "value" worth its own bitmap. Consequently, unlike a plain B-tree index, a bitmap index *can* be used to efficiently satisfy an `IS NULL` predicate — directly resolving the limitation demonstrated in Subtopic 5, Question 3.

**Concept Tested:** Bitmap indexes' ability to represent (and be used for queries against) NULL values, unlike B-tree.

**Exam Trap:** Assuming NULL-handling limitations are universal across all index types, rather than recognizing this as one of bitmap indexes' specific advantages over B-tree.

---

**Question 3**
Why are bitmap indexes generally considered a **poor fit** for tables subject to heavy, concurrent, single-row OLTP-style DML (frequent individual INSERTs/UPDATEs/DELETEs by many simultaneous transactions)?

A. Bitmap indexes cannot be created on tables that receive any DML at all.
B. Each bitmap segment typically represents (and must be updated for) a *range* of many ROWIDs at once, so a single-row DML operation can end up requiring an exclusive lock across that entire range — causing severe lock contention among concurrent transactions trying to modify different rows that happen to fall within the same bitmap segment.
C. Bitmap indexes are always significantly larger in storage size than an equivalent B-tree index, causing DML slowdowns purely due to disk I/O volume.
D. Oracle physically disables all bitmap indexes automatically whenever any DML transaction begins.
 
**Correct Answer:** B

**Detailed Explanation:**
This locking-granularity mismatch is the primary reason bitmap indexes are steered toward data-warehouse/read-heavy environments rather than OLTP systems: modifying one row's underlying bitmap-indexed value can force Oracle to lock (and rewrite) a much broader bitmap segment, blocking unrelated concurrent transactions whose rows happen to share that segment — a scale of contention that simply doesn't occur with B-tree's per-row-entry granularity.

**Why the Other Options Are Wrong:**
A. Bitmap indexes can absolutely be created on DML-active tables; they're just not the *recommended* choice for heavy write concurrency.
C. Bitmap indexes are typically far more compact than B-tree for low-cardinality data — storage size isn't the actual concern here.
D. No such automatic disabling mechanism exists.

**Concept Tested:** The locking-granularity mechanism that makes bitmap indexes unsuitable for high-concurrency OLTP DML.

**Exam Trap:** Assuming the OLTP-unsuitability is about raw performance/size rather than the specific, mechanism-level locking/contention issue — understanding *why* is what's actually tested, not just the conclusion.

---

**Question 4**
What is a major performance advantage of bitmap indexes when a query filters on **multiple**, separately bitmap-indexed, low-cardinality columns simultaneously (e.g., `WHERE gender = 'F' AND marital_status = 'Married' AND region = 'West'`)?

A. Oracle must still perform three completely separate full table scans, one per condition, and then intersect the results manually.
B. Oracle can combine the separate bitmaps for each condition using fast, low-level bitwise AND/OR operations directly on the compact bitmap representations, efficiently narrowing down to the matching rows without needing to consult the actual table data until the final matching ROWIDs are identified.
C. Bitmap indexes cannot be combined across multiple columns; only one bitmap index can be used per query.
D. This scenario provides no particular advantage over B-tree indexes.

**Correct Answer:** B

**Detailed Explanation:**
This is bitmap indexing's other headline strength (alongside NULL support): combining multiple simple, compact bitmaps via bit-level AND/OR is extremely fast, making bitmap indexes especially well-suited to exactly this kind of multi-condition, low-cardinality filtering common in reporting and analytics.

**Why the Other Options Are Wrong:**
A. Understates bitmap indexing's actual mechanism, which specifically avoids this brute-force approach.
C. Multiple bitmap indexes are routinely combined together in a single query — that's precisely bitmap indexing's specialty.
D. Understates a genuine, well-documented advantage specific to this multi-condition, low-cardinality scenario.

**Concept Tested:** Bitmap index combination via bitwise operations as an efficient multi-condition filtering technique.

**Exam Trap:** Assuming index types can only ever be used one at a time per query, missing bitmap indexing's specific strength in efficient multi-index combination.

---

**Question 5**
Which scenario best fits the overall profile where a bitmap index would typically be **recommended**, based on everything established in this subtopic?

A. A high-volume OLTP order-processing table's ORDER_ID primary key column.
B. A large, mostly read-only data warehouse fact table's REGION column, which has only 6 distinct values and is frequently filtered in combination with several other similarly low-cardinality dimension columns.
C. A frequently-updated CUSTOMER_BALANCE column in a banking OLTP system.
D. A UNIQUE-constrained EMAIL column on a customer table with millions of rows.

**Correct Answer:** B

**Detailed Explanation:**
This scenario checks every box established in this subtopic: low cardinality (6 distinct values), a read-heavy/data-warehouse context (favorable for bitmap's locking tradeoffs), and multi-column combination filtering (leveraging bitmap's bitwise-AND strength).

**Why the Other Options Are Wrong:**
A. High-cardinality, OLTP write-heavy — squarely B-tree's domain instead.
C. Frequently-updated OLTP column — exactly the concurrency profile bitmap indexes handle poorly.
D. High-cardinality (millions of distinct emails) and UNIQUE-constrained — this calls for a standard unique B-tree index (Subtopic 7), not a bitmap index.

**Concept Tested:** Synthesizing cardinality, workload type (OLTP vs. warehouse), and multi-column filtering into a single "which index type fits" judgment.

**Exam Trap:** Focusing on only one favorable signal (e.g., "it's a warehouse table") while ignoring a disqualifying signal in the same scenario (e.g., high cardinality) — real exam scenarios often combine several such signals, some pointing toward bitmap and some away from it.

---

### Subtopic 7: Unique Index

**Question 1**
What does a **UNIQUE index** enforce, that an ordinary (non-unique) index does not?

A. Nothing different — "unique" and ordinary indexes behave identically for all purposes.
B. A UNIQUE index actively prevents a new row's indexed value from duplicating any existing non-null value already present — attempting such a duplicate INSERT/UPDATE is rejected. An ordinary index has no such restriction; it exists purely to speed up access, with no bearing on whether duplicate values are permitted.
C. A UNIQUE index prevents any NULL values from ever being stored in the indexed column.
D. A UNIQUE index can only be created on a PRIMARY KEY column.

**Correct Answer:** B

**Detailed Explanation:**
This is the fundamental distinction: uniqueness enforcement is an active business rule (rejecting duplicate data), while an ordinary index is purely a passive performance structure with no data-validation role whatsoever.

**Why the Other Options Are Wrong:**
A. Understates a genuine, defining behavioral difference.
C. As established for UNIQUE constraints in Topic 2 (and revisited in Question 3 below), UNIQUE indexes do still tolerate multiple NULLs — they don't forbid NULL outright.
D. UNIQUE indexes can be created standalone, entirely independent of any PRIMARY KEY (see Question 2).

**Concept Tested:** The active enforcement role of a UNIQUE index versus the passive nature of an ordinary index.

**Exam Trap:** Assuming "unique index" and "ordinary index" differ only in some internal storage detail, rather than recognizing the genuine data-integrity enforcement difference.

---

**Question 2**
Is it possible to create a UNIQUE index on a column with no corresponding UNIQUE or PRIMARY KEY constraint declared at all?

A. No — a UNIQUE index can only ever be created automatically, as a side effect of adding a UNIQUE or PRIMARY KEY constraint (Topic 2).
B. Yes — `CREATE UNIQUE INDEX` can be issued directly and independently, enforcing uniqueness at the index level without any formal constraint object existing in the data dictionary at all.
C. Yes, but only on columns that are also indexed with a separate bitmap index.
D. No — UNIQUE indexes require Oracle Enterprise Edition specifically.

**Correct Answer:** B

**Detailed Explanation:**
While it's true that adding a UNIQUE or PRIMARY KEY constraint automatically creates a backing unique index (Topic 2), the reverse dependency doesn't hold — you can create a standalone unique index directly via `CREATE UNIQUE INDEX idx_name ON table (column);` with no formal constraint involved at all. The index alone still enforces uniqueness at the storage level, even without a named constraint object documenting the "business rule" explicitly.

**Why the Other Options Are Wrong:**
A. Understates the flexibility — direct creation is fully supported.
C. No such bitmap-index dependency exists.
D. Not an edition-specific restriction.

**Concept Tested:** UNIQUE indexes can exist entirely independently of any UNIQUE/PRIMARY KEY constraint.

**Exam Trap:** Assuming the constraint-to-index relationship only flows one way (constraint creates index) without realizing an index can also be created standalone, achieving the same enforcement without a named constraint.

---

**Question 3**
Given that a B-tree index (which underlies a UNIQUE index) skips indexing fully-NULL entries (Subtopic 5), what does this imply about how many rows can hold NULL in a single-column UNIQUE-indexed field?

A. Exactly one row may hold NULL; any additional NULL is rejected as a duplicate.
B. Any number of rows may hold NULL — since NULL entries aren't placed in the B-tree at all, there's nothing for the uniqueness check to compare, so no "duplicate NULL" violation can ever be detected or triggered.
C. Zero rows may ever hold NULL in a UNIQUE-indexed column.
D. Exactly two rows may hold NULL, matching Oracle's default `NULLS FIRST`/`NULLS LAST` pairing.

**Correct Answer:** B

**Detailed Explanation:**
This directly connects two facts already established in this document: UNIQUE constraints tolerate unlimited NULLs (Topic 2, Subtopic 5), and the underlying mechanism explaining *why* is exactly the B-tree NULL-skipping behavior from this topic's Subtopic 5 — since NULL rows simply have no index entry to compare, the uniqueness check has nothing to flag as a conflict, no matter how many rows are NULL.

**Why the Other Options Are Wrong:**
A, C, D. Each invents an artificial NULL limit that contradicts the actual underlying mechanism (no entry exists at all for NULL rows).

**Concept Tested:** Tying the "why" (B-tree NULL-skipping, this topic) to the "what" (UNIQUE constraints tolerate multiple NULLs, Topic 2) into one coherent explanation.

**Exam Trap:** Knowing the Topic 2 rule ("UNIQUE allows multiple NULLs") as a memorized fact without understanding the underlying B-tree mechanism that actually causes it — this question tests the connected, mechanism-level understanding rather than rote recall.

---

**Question 4**
You attempt `DROP INDEX emp_email_uk;` where `EMP_EMAIL_UK` is the index currently enforcing an active UNIQUE constraint on EMPLOYEES.EMAIL. What happens?

A. The index is dropped successfully, and the UNIQUE constraint silently becomes unenforced going forward.
B. The statement fails — Oracle raises an error (ORA-02429-style: cannot drop index used for enforcement of a unique/primary key) because an index currently backing an active constraint cannot be dropped directly; you must instead drop or disable the constraint itself.
C. Both the index and the underlying EMAIL column are dropped together.
D. The index is dropped, but Oracle automatically and silently creates a replacement index of the same name immediately afterward.

**Correct Answer:** B

**Detailed Explanation:**
Oracle protects constraint enforcement from being silently undermined by a direct `DROP INDEX` on a constraint-backing index — you must go through the constraint itself (e.g., `ALTER TABLE ... DROP CONSTRAINT ...` or `DISABLE CONSTRAINT ...`), which then appropriately handles the associated index (dropping it too, by default, unless it was created with a `USING INDEX` clause specifying it should be kept independently).

**Why the Other Options Are Wrong:**
A. Directly contradicted — Oracle actively blocks this rather than allowing silent unenforcement.
C. Dropping an index never cascades to drop a column.
D. No such automatic recreation behavior exists.

**Concept Tested:** Oracle's protection against directly dropping an index that's actively enforcing a constraint.

**Exam Trap:** Assuming DROP INDEX is always unconditionally permitted regardless of what else in the database might depend on that index for enforcement purposes.

---

**Question 5**
Reconcile these two facts: (1) a composite B-tree index still creates an entry for a row as long as **not all** of its key columns are NULL (Subtopic 5), and (2) a composite UNIQUE constraint treats a row as automatically passing the uniqueness check if **any** of its key columns are NULL (Topic 2). Are these two rules contradictory?

A. Yes — if the row has an index entry, it must also be compared for duplicates; these two facts cannot both be true simultaneously.
B. No — they describe two different, independent aspects of the same index. "Does this row get an index entry at all?" (skipped only if *all* columns are NULL) is a separate question from "does this row's entry participate in duplicate-value comparison?" (exempted from comparison if *any* column is NULL) — a row can have a genuine index entry (containing a NULL placeholder for one column) while still being excused, by Oracle's uniqueness-comparison rule, from being flagged as a duplicate of another similar entry.
C. No — fact (1) is simply incorrect; only fact (2)'s "any NULL" rule actually governs Oracle's real behavior.
D. No — fact (2) is simply incorrect; only fact (1)'s "all NULL" rule actually governs Oracle's real behavior.

**Correct Answer:** B

**Detailed Explanation:**
This is a genuinely sophisticated but fully consistent Oracle behavior, connecting this topic's indexing mechanics with Topic 2's constraint semantics: a composite unique index entry like `('Smith', NULL)` *does* physically exist in the B-tree (since 'Smith' is non-null), but Oracle's specific rule for detecting *duplicates* among composite unique keys exempts any comparison involving a NULL component — so two rows like `('Smith', NULL)` and `('Smith', NULL)` (or even `('Smith', 10)` and `('Smith', NULL)`) are never flagged as violating uniqueness, even though both have genuine, present index entries.

**Why the Other Options Are Wrong:**
A. Assumes "having an entry" and "being compared for duplicates" must be the same question — they are documented as two distinct, separable behaviors in Oracle.
C, D. Both facts are independently accurate; neither needs to be discarded to resolve the apparent tension.

**Concept Tested:** The sophisticated distinction between "is this row indexed at all" and "does this row's entry count toward a duplicate-value violation" for composite unique keys — synthesizing this topic's B-tree mechanics with Topic 2's constraint-level NULL-tolerance rule.

**Exam Trap:** This is the most advanced synthesis question in this topic, deliberately designed to look like a contradiction between two previously-established facts, when in reality they describe two independent layers of the same underlying mechanism.