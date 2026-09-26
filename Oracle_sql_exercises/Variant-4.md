# Oracle SQL Certification Practice — Topic 4
## HAVING, ORDER BY, Clause Execution Order & Joins (Theta, ANSI, CROSS, INNER, EQUI, NATURAL, OUTER, SELF)

---

### Subtopic 1: HAVING Clause

**Question 1**
Which statement about HAVING is TRUE?

A. HAVING can only reference columns or expressions that are either listed in the GROUP BY clause or wrapped in an aggregate function.
B. HAVING can filter on any column in the FROM clause's tables, aggregated or not, with no restriction.
C. HAVING is functionally identical to WHERE and the two are fully interchangeable.
D. HAVING can only be used together with an explicit GROUP BY clause — it cannot appear alone.

**Correct Answer:** A

**Detailed Explanation:**
HAVING filters *groups*, not individual rows, so any bare column reference in HAVING must correspond to something that is single-valued per group — either a GROUP BY column or an aggregate function's result. Referencing a non-grouped, non-aggregated column raises **ORA-00979: not a GROUP BY expression**.

**Why the Other Options Are Wrong:**
B. Overstates HAVING's flexibility — the GROUP BY/aggregate restriction is real and enforced.
C. WHERE filters rows before grouping; HAVING filters groups after grouping — they operate at different stages and aren't interchangeable.
D. HAVING can legally appear without GROUP BY, treating the entire table as a single implicit group (see Question 2).

**Concept Tested:** HAVING's restriction to grouped/aggregated expressions.

**Exam Trap:** Assuming HAVING is simply "WHERE, but for groups" without recognizing its stricter column-reference rule.

---

**Question 2**
Is `SELECT AVG(salary) FROM employees HAVING AVG(salary) > 5000;` (no GROUP BY) legal?

A. No — HAVING requires an explicit GROUP BY clause to precede it.
B. Yes — without GROUP BY, the entire table is treated as a single group, and HAVING filters whether that one group's aggregate satisfies the condition (returning either one row or zero rows).
C. Yes, but the result is always identical to omitting HAVING entirely.
D. No — this raises ORA-00937: not a single-group group function.

**Correct Answer:** B

**Detailed Explanation:**
HAVING doesn't strictly require GROUP BY. Without one, the whole table is the (only) group, and HAVING acts as a gate on whether that single aggregate result is displayed at all — the query returns exactly one row if the condition is true, or zero rows if false.

**Why the Other Options Are Wrong:**
A, D. Both wrongly claim GROUP BY is mandatory for HAVING.
C. The result differs meaningfully: with HAVING, an unsatisfying condition suppresses the single output row entirely, rather than always displaying it.

**Concept Tested:** HAVING without GROUP BY, applied to the whole table as one implicit group.

**Exam Trap:** Assuming any use of HAVING must be paired with GROUP BY, and mislabeling a perfectly legal query as a syntax error.

---

**Question 3**
Given `SELECT department_id, COUNT(*) AS emp_count FROM employees GROUP BY department_id HAVING emp_count > 5;` — what is wrong with this query?

A. Nothing — this is perfectly valid and returns departments with more than 5 employees.
B. HAVING cannot reference `emp_count`, a column alias defined in the SELECT list — aliases are not yet resolved at the logical point where HAVING is evaluated, so this raises an "invalid identifier" error. The condition must instead repeat the full expression: `HAVING COUNT(*) > 5`.
C. GROUP BY cannot be combined with HAVING in the same query.
D. COUNT(*) cannot be aliased.

**Correct Answer:** B

**Detailed Explanation:**
In Oracle's logical processing order, HAVING is evaluated *before* the SELECT list's aliases are established — so, much like WHERE, HAVING cannot reference a SELECT-list alias. The condition must spell out the actual expression again: `HAVING COUNT(*) > 5`.

**Why the Other Options Are Wrong:**
A. Overlooks the alias-reference problem.
C. GROUP BY and HAVING are routinely combined — that's their most common use.
D. COUNT(*) can absolutely be aliased for display purposes; the alias just isn't usable inside HAVING itself.

**Concept Tested:** SELECT-list aliases are not visible to HAVING (or WHERE), only to ORDER BY.

**Exam Trap:** Assuming any alias defined anywhere in the query is immediately usable everywhere else in that same query.

---

**Question 4**
Which two conditions could legally appear in a HAVING clause for the query `SELECT department_id, AVG(salary) FROM employees GROUP BY department_id HAVING ...`? **(Choose two)**

A. `department_id IN (10, 20, 30)`
B. `AVG(salary) > 5000`
C. `salary > 3000`
D. `MAX(hire_date) > DATE '2020-01-01'`

**Correct Answers:** A and B (and D would also be legal, but only two are required)

**Detailed Explanation:**
`department_id` is the GROUP BY column, so it's legal in HAVING. `AVG(salary)` is an aggregate expression, also legal. `salary` alone (C) is neither grouped nor aggregated — illegal (ORA-00979). `MAX(hire_date)...` (D) is also a legal aggregate-based condition, illustrating that HAVING can carry multiple independent aggregate conditions, though only two options were required here.

**Why C Is Wrong:**
`salary` is a raw, non-aggregated, non-grouped column — HAVING cannot evaluate it per-group without an aggregate wrapper.

**Concept Tested:** Distinguishing legal (grouped/aggregated) vs. illegal (raw, ungrouped) column references in HAVING.

**Exam Trap:** Assuming that once a query has *any* GROUP BY, every column from the base tables becomes fair game in HAVING.

---

**Question 5**
True or False: A single HAVING clause can combine multiple conditions with AND/OR, including conditions on different aggregate functions.

A. True
B. False

**Correct Answer:** A (True)

**Detailed Explanation:**
HAVING supports full boolean expressions, just like WHERE — e.g., `HAVING AVG(salary) > 5000 AND COUNT(*) > 3` is completely standard and commonly tested in combination-heavy certification questions.

**Concept Tested:** HAVING supports compound boolean logic across multiple aggregate conditions.

**Exam Trap:** Assuming HAVING is limited to a single simple condition per query.

---

### Subtopic 2: ORDER BY Clause

**Question 1**
Which of the following is a valid use of ORDER BY that is specifically **not** legal in a WHERE or GROUP BY clause?

A. `ORDER BY department_id`
B. `ORDER BY salary_display` — where `salary_display` is a column alias defined in the SELECT list.
C. `ORDER BY 1`
D. `ORDER BY salary DESC`

**Correct Answer:** B

**Detailed Explanation:**
ORDER BY is the one major clause that *can* reference a SELECT-list column alias, because logically it executes after the SELECT list has been resolved — the opposite situation from HAVING/WHERE (Subtopic 1, Question 3). A, C, and D are all legal in ORDER BY, but none of them demonstrate this specific alias-referencing capability that's unique to ORDER BY.

**Concept Tested:** ORDER BY is the only major clause that can reference SELECT-list aliases.

**Exam Trap:** After learning that WHERE/HAVING can't use aliases, wrongly assuming *no* clause can — ORDER BY is the documented exception.

---

**Question 2**
SALARY values include several NULLs. What is the default position of NULL rows in `ORDER BY salary ASC` versus `ORDER BY salary DESC`?

A. NULLs always sort first, regardless of ASC or DESC.
B. NULLs always sort last, regardless of ASC or DESC.
C. In ASC order, NULLs sort last by default; in DESC order, NULLs sort first by default — Oracle treats NULL as the "largest" possible value for default sorting purposes.
D. NULLs are excluded from the result entirely by ORDER BY.

**Correct Answer:** C

**Detailed Explanation:**
This is an Oracle-specific default (differing from some other RDBMS defaults): NULL is conceptually treated as larger than any non-null value. So ascending order pushes NULLs to the very end, while descending order — being the reverse — puts them at the very beginning. This default can be overridden explicitly with `NULLS FIRST` or `NULLS LAST`.

**Why the Other Options Are Wrong:**
A, B. Both wrongly claim a single fixed position regardless of sort direction.
D. ORDER BY never removes rows; it only affects display sequence.

**Concept Tested:** Oracle's default NULL placement in ASC vs. DESC sorts.

**Exam Trap:** Assuming NULLs always sort to one fixed position (typically "last") regardless of sort direction — the direction genuinely flips the default.

---

**Question 3**
Which clause explicitly overrides Oracle's default NULL-ordering behavior?

A. `ORDER BY salary ASC NULLS FIRST`
B. `ORDER BY NULLS(salary) ASC`
C. `WHERE salary IS NOT NULL ORDER BY salary`
D. There is no way to override the default; it is fixed per sort direction.

**Correct Answer:** A

**Detailed Explanation:**
Oracle's `NULLS FIRST` / `NULLS LAST` modifiers, appended after the sort direction, let you explicitly control NULL placement independent of the ASC/DESC default — e.g., forcing NULLs to appear first even in an otherwise-ascending sort.

**Why the Other Options Are Wrong:**
B. Invented, invalid syntax.
C. This filters NULLs out entirely rather than repositioning them within the result — a fundamentally different (and often undesired) effect.
D. Directly contradicted by the existence of NULLS FIRST/LAST.

**Concept Tested:** NULLS FIRST / NULLS LAST syntax for explicit override.

**Exam Trap:** Confusing "excluding NULLs via WHERE" with "repositioning NULLs via ORDER BY ... NULLS FIRST/LAST" — these solve different problems.

---

**Question 4**
`SELECT last_name, department_id, salary FROM employees ORDER BY department_id, salary DESC;` — how are rows sorted?

A. Both department_id and salary are sorted descending.
B. department_id is sorted ascending (the default when unspecified), and *within* each department_id group, salary is sorted descending.
C. Only salary determines the sort order; department_id is ignored once a second ORDER BY column is present.
D. This raises a syntax error — you cannot mix implied ASC and explicit DESC in the same ORDER BY list.

**Correct Answer:** B

**Detailed Explanation:**
Each column in an ORDER BY list carries its own independent sort direction; omitting a direction defaults to ASC for that column only. Sorting proceeds primarily by the first column, with the second column breaking ties within each value of the first — here, ascending department_id as the primary sort key, descending salary as the secondary/tie-breaking key.

**Why the Other Options Are Wrong:**
A. Wrongly applies DESC to both columns.
C. The first-listed column is never "ignored" — it remains the primary sort key.
D. Mixed directions per column are completely standard, legal syntax.

**Concept Tested:** Multi-column ORDER BY with independent per-column sort directions and precedence.

**Exam Trap:** Assuming a single DESC/ASC keyword applies to the entire ORDER BY list rather than only to the column it's attached to.

---

**Question 5**
True or False: `ORDER BY 2, 1` is valid syntax that sorts by the second column in the SELECT list first, then the first column as a tiebreaker.

A. True
B. False

**Correct Answer:** A (True)

**Detailed Explanation:**
ORDER BY supports positional references to the SELECT list's column positions (1-indexed), and multiple positions can be combined with the same primary/secondary precedence rules as named columns.

**Concept Tested:** Positional ORDER BY syntax and its combination with multi-column tie-breaking.

**Exam Trap:** Assuming positional ORDER BY only supports a single column reference, not a combination.

---

### Subtopic 3: Order of Execution of Clauses in a SELECT Statement

**Question 1**
What is the correct logical order of execution for the major clauses of a SELECT statement?

A. SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY
B. FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
C. FROM → SELECT → WHERE → GROUP BY → HAVING → ORDER BY
D. WHERE → FROM → GROUP BY → HAVING → SELECT → ORDER BY

**Correct Answer:** B

**Detailed Explanation:**
Even though we *write* SELECT first, Oracle logically processes: identify the source rows (FROM), filter individual rows (WHERE), collapse into groups (GROUP BY), filter those groups (HAVING), compute and name the final output columns (SELECT, including aliases), and finally sort the result (ORDER BY). This explains every alias/aggregate rule covered so far: WHERE and HAVING run before SELECT resolves aliases (so they can't use them), and ORDER BY runs after SELECT (so it can).

**Why the Other Options Are Wrong:**
A, C, D. Each misplaces SELECT relative to WHERE/GROUP BY/HAVING, which would make the established alias-visibility rules impossible to explain.

**Concept Tested:** The canonical logical execution order underlying every alias/aggregate-placement rule tested elsewhere in this topic.

**Exam Trap:** Confusing written (syntactic) order with logical (execution) order — this single confusion is the root cause of most WHERE/HAVING/ORDER BY alias mistakes.

---

**Question 2**
Why does `WHERE AVG(salary) > 5000` raise an error, while `HAVING AVG(salary) > 5000` does not?

A. WHERE simply doesn't support the `>` operator with aggregate results; HAVING has special operator support.
B. At the logical point WHERE is evaluated, grouping/aggregation hasn't happened yet — there's no "group" for AVG to summarize — whereas HAVING runs after GROUP BY has already produced group-level aggregates to filter on.
C. WHERE and HAVING are identical; this is simply a case-sensitivity issue with the keyword AVG.
D. WHERE actually does support this; the error described doesn't really occur.

**Correct Answer:** B

**Detailed Explanation:**
This question directly connects the execution-order concept to the WHERE-vs-HAVING restriction: aggregate functions require a defined group to summarize, and at WHERE's execution point, no grouping has occurred yet.

**Why the Other Options Are Wrong:**
A. Misattributes the restriction to operator syntax rather than logical timing.
C. Irrelevant/fabricated.
D. The error (ORA-00934: group function is not allowed here) is real and well-documented.

**Concept Tested:** Root cause of the WHERE-cannot-use-aggregates rule, tied back to execution order.

**Exam Trap:** Memorizing "WHERE can't use aggregates" as an arbitrary rule rather than understanding *why*, which makes it easy to misapply to edge cases.

---

**Question 3**
Where does a row-limiting clause like `FETCH FIRST n ROWS ONLY` fit into the logical execution order relative to ORDER BY?

A. It is evaluated before ORDER BY, so the "first n rows" are chosen before any sorting occurs.
B. It is evaluated after ORDER BY — the full result set is sorted first, and only then are the first n rows of that sorted output selected.
C. It replaces ORDER BY entirely; the two cannot be used together.
D. Its position relative to ORDER BY is undefined and varies by query.

**Correct Answer:** B

**Detailed Explanation:**
This is precisely why `ORDER BY salary DESC FETCH FIRST 3 ROWS ONLY` reliably returns the top 3 by salary (as established in Topic 1): the entire ordering is logically completed first, and the row-limiting clause then trims the already-sorted sequence.

**Why the Other Options Are Wrong:**
A. Would produce arbitrary, unordered "first n" rows rather than the top n — the opposite of FETCH FIRST's guarantee.
C. FETCH FIRST is commonly and correctly combined with ORDER BY.
D. The order is fixed and well-defined, not variable.

**Concept Tested:** FETCH FIRST's logical position after ORDER BY in execution order.

**Exam Trap:** Reintroducing the same misconception from the Topic 1 ROWNUM trap, now framed around FETCH FIRST's relationship to ORDER BY instead.

---

**Question 4**
Given:
```sql
SELECT department_id, COUNT(*) AS cnt
FROM employees
WHERE salary > 3000
GROUP BY department_id
HAVING COUNT(*) > 2
ORDER BY cnt DESC;
```
Which clause filters **individual employee rows**, and which filters **entire departments**?

A. WHERE filters individual rows (salary > 3000); HAVING filters entire groups/departments (more than 2 qualifying employees).
B. HAVING filters individual rows; WHERE filters entire departments.
C. Both WHERE and HAVING filter individual rows; only GROUP BY filters departments.
D. GROUP BY filters individual rows; WHERE and HAVING both filter departments.

**Correct Answer:** A

**Detailed Explanation:**
WHERE excludes individual employees earning ≤ 3000 *before* any grouping happens. GROUP BY then collapses the *remaining* employees into per-department groups. HAVING then excludes entire departments whose surviving employee count doesn't exceed 2. This is the clean, textbook separation of row-level vs. group-level filtering.

**Concept Tested:** Correctly attributing "what gets filtered" to the right clause in a combined WHERE + GROUP BY + HAVING query.

**Exam Trap:** Swapping WHERE's and HAVING's roles, especially in longer queries where it's easy to lose track of which clause operates at which granularity.

---

**Question 5**
True or False: In the logical execution order, DISTINCT is applied before ORDER BY.

A. True
B. False

**Correct Answer:** A (True)

**Detailed Explanation:**
DISTINCT (part of the SELECT clause's processing) removes duplicate result rows *before* the final ORDER BY sorts the remaining, de-duplicated set — otherwise, ORDER BY would have no stable, finalized row set to sort in the first place.

**Concept Tested:** DISTINCT's place in the execution order, just before ORDER BY.

**Exam Trap:** Assuming ORDER BY (since it's written last) might act on the "raw," pre-DISTINCT rows, rather than the already-deduplicated final set.

---

### Subtopic 4: Theta-Style Joins & Oracle-Style JOIN...ON / JOIN...USING

**Question 1**
What defines a "theta join"?

A. A join that can only use the `=` operator.
B. A join whose condition may use **any** comparison operator (`=`, `<`, `>`, `<=`, `>=`, `<>`, or `BETWEEN`) between columns of the joined tables — the general category that equijoins (using only `=`) fall under as a special case.
C. A join that always produces a Cartesian product.
D. A join type exclusive to Oracle, unsupported in the ANSI SQL standard.

**Correct Answer:** B

**Detailed Explanation:**
"Theta join" is the umbrella relational-algebra term for any join using a comparison operator (θ) in its condition. An equijoin is simply the theta join whose operator happens to be `=`. Non-equi theta joins (e.g., matching a salary into a grade range using BETWEEN) are equally valid theta joins.

**Why the Other Options Are Wrong:**
A. Describes only the equijoin subset, not theta joins generally.
C. Describes a CROSS JOIN, an entirely different concept (no condition at all).
D. Theta joins are a general relational concept, not Oracle-specific.

**Concept Tested:** Theta join as the general category encompassing both equi- and non-equijoins.

**Exam Trap:** Conflating "theta join" with "equijoin" specifically, rather than recognizing equijoin as one particular case of the broader theta-join category.

---

**Question 2**
Which of these is a classic example of a **non-equi** theta join?

A. `SELECT * FROM employees e, departments d WHERE e.department_id = d.department_id;`
B. `SELECT e.last_name, s.grade FROM employees e, salgrade s WHERE e.salary BETWEEN s.losal AND s.hisal;`
C. `SELECT * FROM employees NATURAL JOIN departments;`
D. `SELECT * FROM employees e JOIN departments d USING (department_id);`

**Correct Answer:** B

**Detailed Explanation:**
BETWEEN is shorthand for `>=` and `<=` combined — a genuine non-equi comparison. This range-matching pattern (matching a salary into the correct grade bracket) is the textbook non-equijoin example.

**Why the Other Options Are Wrong:**
A, C, D. All join on exact equality of `department_id` — each is an equijoin, just expressed in different syntaxes (old-style WHERE, NATURAL JOIN, and USING, respectively).

**Concept Tested:** Recognizing non-equijoins (range/comparison-based) versus equijoins expressed in various syntaxes.

**Exam Trap:** Assuming any join phrased with old-style comma+WHERE syntax must be non-equi, or any modern ANSI JOIN syntax must be equi — the syntax style and the comparison-operator type are independent axes.

---

**Question 3**
What is the key restriction on `JOIN ... USING (column)` compared to `JOIN ... ON`?

A. USING can only join exactly two tables; ON can join any number.
B. USING requires the join column to have the **same name** in both tables, and that shared column must **not** be qualified with a table prefix anywhere else in the statement (raising ORA-25154 if you do); ON has no such name-matching or qualification restriction.
C. USING cannot be combined with a WHERE clause; ON can.
D. USING always performs an outer join; ON always performs an inner join.

**Correct Answer:** B

**Detailed Explanation:**
USING is essentially a more constrained, explicit-column version of NATURAL JOIN: you name the shared column, but Oracle still forbids qualifying that specific column with a table alias elsewhere (e.g., `e.department_id`) — you simply write `department_id` unqualified. `ON`, by contrast, freely allows differently-named columns on each side and always allows full table-qualification.

**Why the Other Options Are Wrong:**
A. Both USING and ON can be used across multi-way joins.
C. Both can be freely combined with additional WHERE conditions.
D. Neither USING nor ON dictates inner vs. outer join type by itself — that's determined by which JOIN keyword (INNER/LEFT/RIGHT/FULL) precedes them.

**Concept Tested:** USING's same-name-required, unqualified-reference restriction vs. ON's full flexibility.

**Exam Trap:** Assuming USING and ON are simply stylistic alternatives with identical capabilities, rather than USING carrying real additional restrictions.

---

**Question 4**
Can you add extra filtering conditions to a query that uses `JOIN ... USING (department_id)`?

A. No — once USING is specified, no further conditions of any kind may be added to the query.
B. Yes — but only by adding them directly inside the USING clause with AND, e.g., `USING (department_id AND salary > 3000)`.
C. Yes — by adding a separate WHERE clause after the USING clause, e.g., `... USING (department_id) WHERE salary > 3000`.
D. Yes, but only if the additional condition also references department_id.

**Correct Answer:** C

**Detailed Explanation:**
The USING clause's parentheses may only contain a list of shared column names — it is not a general boolean-expression container. Any additional filtering belongs in a normal WHERE clause following the join, exactly as it would with any other join syntax.

**Why the Other Options Are Wrong:**
A. Additional filtering is completely normal and expected.
B. USING's syntax doesn't support embedding arbitrary conditions with AND inside it — only column names.
D. The additional WHERE condition can reference any column, unrelated to the join column.

**Concept Tested:** USING's syntax is restricted to column-name lists; general filtering still requires WHERE.

**Exam Trap:** Assuming USING behaves like a mini-WHERE clause that can carry arbitrary AND-ed conditions.

---

**Question 5**
Which of the following correctly rewrites `SELECT * FROM employees e, departments d WHERE e.department_id = d.department_id AND e.salary > 3000;` into ANSI join syntax?

A. `SELECT * FROM employees e JOIN departments d ON e.department_id = d.department_id WHERE e.salary > 3000;`
B. `SELECT * FROM employees e JOIN departments d ON e.department_id = d.department_id AND e.salary > 3000;`
C. Both A and B are valid and produce identical results.
D. Neither is valid; old-style theta joins cannot be rewritten using ANSI syntax.

**Correct Answer:** C

**Detailed Explanation:**
The join condition (equating department_id) can be placed in the ON clause, while the additional filter (salary > 3000) can either stay in a separate WHERE clause (A) or be folded directly into the ON clause alongside the join condition using AND (B) — for an INNER JOIN, both placements produce the same final row set, since there's no NULL-extension behavior to worry about (that distinction becomes important only for OUTER JOINs, covered later in this topic).

**Why D Is Wrong:**
Old-style theta/equijoins can always be re-expressed using ANSI JOIN syntax — the underlying relational logic doesn't change, only the syntax.

**Concept Tested:** Old-style comma+WHERE syntax is fully convertible to ANSI JOIN...ON syntax, with filter conditions flexibly placed in ON or WHERE for inner joins.

**Exam Trap:** Assuming there's only one "correct" ANSI translation, when for an INNER JOIN, ON and WHERE placement of a non-join filter condition are functionally interchangeable.

---

### Subtopic 5: CROSS JOIN

**Question 1**
Table A has 6 rows; Table B has 4 rows. How many rows does `SELECT * FROM a CROSS JOIN b;` return?

A. 10
B. 24
C. 6
D. 4

**Correct Answer:** B

**Detailed Explanation:**
CROSS JOIN produces the full Cartesian product: every row of A paired with every row of B, with no filtering condition — 6 × 4 = 24 rows.

**Concept Tested:** CROSS JOIN's row-count arithmetic (m × n).

**Exam Trap:** Confusing Cartesian multiplication with simple addition of row counts.

---

**Question 2**
Can `CROSS JOIN` be given an `ON` clause, e.g., `SELECT * FROM a CROSS JOIN b ON a.id = b.id;`?

A. Yes — this is standard, and simply restricts the Cartesian product to matching rows.
B. No — CROSS JOIN, by definition, carries no join condition; adding ON is a syntax error. To get matching rows, you'd use INNER JOIN...ON instead.
C. Yes, but only when combined with USING instead of ON.
D. Yes, and it silently ignores the ON clause, still returning the full Cartesian product.

**Correct Answer:** B

**Detailed Explanation:**
CROSS JOIN is specifically the *conditionless* join — its entire definition is "no filtering, full Cartesian product." Attaching ON or USING contradicts that definition and is rejected as invalid syntax; the correct keyword for a conditioned join is INNER JOIN (or LEFT/RIGHT/FULL for outer variants).

**Why the Other Options Are Wrong:**
A, C, D. All wrongly assume CROSS JOIN can somehow coexist with a join condition.

**Concept Tested:** CROSS JOIN cannot take ON or USING — it is condition-free by design.

**Exam Trap:** Assuming every JOIN keyword uniformly supports ON/USING, rather than recognizing CROSS JOIN as the deliberate exception.

---

**Question 3**
Which old-style (pre-ANSI) syntax is equivalent to a CROSS JOIN?

A. `SELECT * FROM a, b;` with no WHERE clause joining the two tables at all.
B. `SELECT * FROM a, b WHERE a.id = b.id;`
C. `SELECT * FROM a NATURAL JOIN b;`
D. There is no old-style equivalent; CROSS JOIN is an ANSI-only innovation.

**Correct Answer:** A

**Detailed Explanation:**
Simply listing multiple tables in the old comma-separated FROM clause, with no corresponding WHERE condition linking them, silently produces the full Cartesian product — the classic (and classically dangerous) old-style equivalent of CROSS JOIN.

**Why the Other Options Are Wrong:**
B. Includes a join condition — this is an equijoin, not a Cartesian join.
C. NATURAL JOIN actively looks for matching columns — the opposite of condition-free.
D. The old-style comma syntax predates ANSI JOIN keywords and can absolutely produce this same Cartesian behavior.

**Concept Tested:** Old-style comma-separated FROM clauses default to Cartesian product absent a WHERE join condition.

**Exam Trap:** Not recognizing that an *accidentally* omitted WHERE join condition in old-style syntax produces the exact same (often unintended) result as an explicit CROSS JOIN — this is one of the most common real-world SQL bugs, and a frequently tested concept.

---

**Question 4**
True or False: Forgetting the WHERE join condition in an old-style, comma-separated multi-table query causes Oracle to raise a syntax error, alerting you to the missing join.

A. True
B. False

**Correct Answer:** B (False)

**Detailed Explanation:**
This is precisely the danger: Oracle does **not** error out. The query executes successfully and silently returns a full Cartesian product — syntactically valid, but almost certainly not what was intended, and easy to overlook, especially with small test datasets where the resulting row count might look plausible.

**Concept Tested:** No syntax-level protection exists against accidental Cartesian products in old-style join syntax.

**Exam Trap:** Assuming Oracle would proactively flag a "missing join condition" as an error — it has no way to know whether a Cartesian product was intended or accidental.

---

**Question 5**
Which of these row counts is only possible via CROSS JOIN (or its Cartesian equivalent), not via any INNER/OUTER JOIN with a genuine matching condition, given Table A = 5 rows and Table B = 3 rows (with no duplicate keys in either table)?

A. 3
B. 5
C. 8
D. 15

**Correct Answer:** D

**Detailed Explanation:**
With unique keys in each table and no duplicated join-key values, any genuine equijoin's maximum possible row count is bounded by the smaller table's row count for an inner join (at most 3, if every B row finds an A match) or by the outer table's row count for an outer join (at most 5). A row count of 15 (5 × 3) can only arise from an unconditional Cartesian product.

**Why the Other Options Are Wrong:**
A, B, C. All fall within the range achievable by ordinary INNER/OUTER JOINs under these uniqueness assumptions.

**Concept Tested:** Recognizing when an observed row count is only explainable by a Cartesian product.

**Exam Trap:** Not using row-count arithmetic as a diagnostic tool for identifying an accidental Cartesian join in a "what went wrong with this query" scenario.

---

### Subtopic 6: INNER JOIN & EQUI-JOIN

**Question 1**
Which statement correctly describes the relationship between "equijoin" and "INNER JOIN"?

A. They are entirely different, unrelated concepts.
B. Every equijoin is an inner join, but not every inner join is an equijoin — INNER JOIN is the general ANSI keyword for "matched-rows-only" joins, regardless of which comparison operator the ON condition uses, while "equijoin" specifically refers to a join whose condition uses `=`.
C. INNER JOIN can only ever use the `=` operator; a non-equi condition automatically becomes an OUTER JOIN.
D. "Equijoin" is Oracle-specific terminology; INNER JOIN is ANSI-specific — they refer to identical joins but the term used depends on which syntax style you write.

**Correct Answer:** B

**Detailed Explanation:**
INNER JOIN describes the *behavior* (only rows with a match in both tables survive) and permits any comparison operator in its ON condition — including `<`, `>`, or BETWEEN, producing a non-equi inner join. Equijoin specifically describes joins using `=`. So equijoin is a proper subset of inner join, not a synonym for it.

**Why the Other Options Are Wrong:**
A. They are closely related, not unrelated.
C. A non-equi condition in an ON clause is still an INNER JOIN (assuming no OUTER keyword is used) — operator choice doesn't change inner/outer status.
D. Conflates a terminology distinction (operator type) with a syntax-style distinction (old vs. ANSI) — these are independent axes, as shown in Subtopic 4.

**Concept Tested:** Equijoin is a specific case of inner join; the two terms are not interchangeable.

**Exam Trap:** Treating "equijoin" and "inner join" as synonyms, rather than recognizing the subset relationship.

---

**Question 2**
Is writing `SELECT * FROM a JOIN b ON a.id = b.id;` (bare `JOIN`, no INNER keyword) different from `SELECT * FROM a INNER JOIN b ON a.id = b.id;`?

A. Yes — bare JOIN defaults to a CROSS JOIN unless INNER is specified.
B. No — INNER is the implicit default for the bare `JOIN` keyword; the two statements are functionally identical.
C. Yes — bare JOIN defaults to a NATURAL JOIN.
D. Yes — bare JOIN defaults to a LEFT OUTER JOIN.

**Correct Answer:** B

**Detailed Explanation:**
In ANSI join syntax, `JOIN` with no qualifying keyword is always shorthand for `INNER JOIN`. This is purely a stylistic/verbosity choice — many style guides prefer writing INNER explicitly for clarity, but it changes nothing about the query's behavior.

**Why the Other Options Are Wrong:**
A, C, D. All wrongly invent a different default join type for bare JOIN.

**Concept Tested:** Bare `JOIN` = `INNER JOIN` by default.

**Exam Trap:** Assuming an unqualified JOIN keyword must default to the "broadest" or "safest" join type (like CROSS or an OUTER variant), rather than INNER.

---

**Question 3**
Table EMPLOYEES has 20 rows; 5 of them have `department_id = NULL`. Table DEPARTMENTS has 8 rows. How many rows does `SELECT * FROM employees e JOIN departments d ON e.department_id = d.department_id;` return, at most?

A. Exactly 20, always.
B. At most 15 — the 5 employees with a NULL department_id can never satisfy the equality condition against any department_id (NULL is never equal to anything), so those rows are necessarily excluded from an INNER JOIN's results.
C. Exactly 8, always.
D. Exactly 28 (20 + 8).

**Correct Answer:** B

**Detailed Explanation:**
An equality-based INNER JOIN condition can never be satisfied by a NULL operand (three-valued logic: NULL = anything is UNKNOWN, not TRUE). So the 5 employees with NULL department_id are guaranteed to be dropped from the inner-join result, capping the maximum possible row count at 15 (and it could be fewer still, if some non-null department_id values don't match any row in DEPARTMENTS).

**Why the Other Options Are Wrong:**
A, C, D. None correctly account for NULL's effect on equijoin matching.

**Concept Tested:** NULL values in the join column are always excluded from INNER JOIN results (revisiting three-valued logic from Topic 1, now applied to joins).

**Exam Trap:** Forgetting that NULL-valued join keys silently vanish from inner-join output — this connects directly back to the NOT IN/NULL trap from earlier topics.

---

**Question 4**
Which join type is being used in `SELECT e.last_name, d.department_name FROM employees e, departments d WHERE e.department_id = d.department_id;`?

A. CROSS JOIN
B. NATURAL JOIN
C. Old-style (theta) equijoin, functionally equivalent to an ANSI INNER JOIN...ON with an equality condition.
D. LEFT OUTER JOIN

**Correct Answer:** C

**Detailed Explanation:**
This is the pre-ANSI, comma-separated syntax expressing an equijoin via WHERE. It behaves identically to `employees e INNER JOIN departments d ON e.department_id = d.department_id` — same rows, same matching logic, just older syntax.

**Why the Other Options Are Wrong:**
A. A genuine join condition is present — this is not condition-free.
B. NATURAL JOIN has its own specific keyword-based syntax; this query doesn't use it (and NATURAL JOIN auto-detects columns rather than specifying them explicitly in a WHERE clause).
D. Nothing here signals outer-join (NULL-preserving) behavior — unmatched rows from either side are excluded, as in any inner/equijoin.

**Concept Tested:** Recognizing an old-style equijoin and its ANSI equivalent.

**Exam Trap:** Not recognizing older syntax as functionally equivalent to modern ANSI keywords, and second-guessing into an incorrect, more exotic join classification.

---

**Question 5**
Which of these is a valid, legal **non-equi inner join** using ANSI syntax?

A. `SELECT * FROM employees e JOIN salgrade s ON e.salary BETWEEN s.losal AND s.hisal;`
B. `SELECT * FROM employees e NATURAL JOIN salgrade s;`
C. `SELECT * FROM employees e JOIN salgrade s USING (salary);`
D. All three are equally valid ways to express this relationship.

**Correct Answer:** A

**Detailed Explanation:**
ANSI's `JOIN...ON` fully supports non-equi conditions like BETWEEN — this is the modern-syntax equivalent of the classic salary/grade range-matching theta join from Subtopic 4.

**Why the Other Options Are Wrong:**
B. NATURAL JOIN can only match on equally-named columns using `=` — it has no mechanism for range-based matching.
C. USING likewise only supports equality matching on a shared column name — no range logic possible.
D. Only A actually accomplishes range-based, non-equi matching.

**Concept Tested:** ON is the only ANSI join clause flexible enough to express non-equi conditions; NATURAL and USING are equality-only mechanisms.

**Exam Trap:** Assuming NATURAL JOIN or USING are just alternative syntaxes for "any kind of join," rather than recognizing they're both hard-wired to equality matching specifically.

---

### Subtopic 7: NATURAL JOIN

**Question 1**
`EMPLOYEES` and `DEPARTMENTS` both happen to have columns named `DEPARTMENT_ID` and `MANAGER_ID`. What does `SELECT * FROM employees NATURAL JOIN departments;` actually join on?

A. Only DEPARTMENT_ID, since that's the "obvious" intended join column.
B. **Both** DEPARTMENT_ID and MANAGER_ID simultaneously — NATURAL JOIN automatically joins on every column that shares the same name (and compatible data type) in both tables, whether or not that matches the query author's actual intent.
C. Neither — NATURAL JOIN requires you to explicitly confirm which shared columns to use.
D. Whichever shared column comes first alphabetically.

**Correct Answer:** B

**Detailed Explanation:**
NATURAL JOIN is entirely automatic and column-name-driven: it joins on *every* identically-named, type-compatible column pair it finds — with no way to exclude one of them. If EMPLOYEES.MANAGER_ID and DEPARTMENTS.MANAGER_ID happen to share a name but represent unrelated concepts, NATURAL JOIN will incorrectly fold that into the join condition too, likely producing fewer (or entirely wrong) rows than intended.

**Why the Other Options Are Wrong:**
A, D. Both wrongly assume NATURAL JOIN exercises some judgment about "which" shared column is the real intended key — it has none.
C. NATURAL JOIN never asks for confirmation; it acts automatically and silently.

**Concept Tested:** NATURAL JOIN's automatic, all-shared-columns behavior — and why it's considered a risky feature in real schemas with generic, reused column names.

**Exam Trap:** Assuming NATURAL JOIN is "smart enough" to pick only the sensible foreign-key relationship, rather than mechanically joining on every same-named column without exception.

---

**Question 2**
What happens if you try `SELECT e.department_id, e.last_name FROM employees e NATURAL JOIN departments d;`?

A. It succeeds and returns the department_id column qualified with the "e" alias.
B. It fails with an error (ORA-25155-style) — a column that participates in a NATURAL JOIN cannot be qualified with a table alias/prefix anywhere in the statement, since Oracle can't determine which of the two (identically-valued) source columns you mean to prefix.
C. It succeeds, but silently ignores the "e." prefix.
D. It succeeds and returns NULL for department_id, since the prefix conflicts with the join.

**Correct Answer:** B

**Detailed Explanation:**
Just like USING (Subtopic 4), NATURAL JOIN's shared/common columns must be referenced *unqualified* everywhere in the SQL statement — including the SELECT list. Attempting to qualify `department_id` with `e.` raises an error, since NATURAL JOIN has already merged that column into a single, shared identity — qualifying it artificially reintroduces an ambiguity Oracle deliberately disallows.

**Why the Other Options Are Wrong:**
A, C, D. All wrongly assume the query executes successfully in some form.

**Concept Tested:** NATURAL JOIN forbids qualifying its automatically-matched columns, mirroring the USING restriction.

**Exam Trap:** Assuming table-qualifying a column is always harmless/optional syntax — for NATURAL JOIN's (and USING's) shared columns specifically, it's actively illegal.

---

**Question 3**
Can you combine `NATURAL JOIN` with an explicit `ON` or `USING` clause in the same join, e.g., `... NATURAL JOIN departments ON (e.department_id = d.department_id)`?

A. Yes — this simply adds an extra explicit condition on top of NATURAL JOIN's automatic matching.
B. No — combining NATURAL JOIN with ON or USING is a syntax error; NATURAL JOIN's entire premise is that no explicit join columns/conditions are specified.
C. Yes, but only with USING, never with ON.
D. Yes, but the ON/USING clause is simply ignored at runtime.

**Correct Answer:** B

**Detailed Explanation:**
NATURAL JOIN is, by definition, condition-free (in the sense that you never *specify* the join columns — Oracle infers them). Pairing it with an explicit ON or USING clause is contradictory and rejected outright as invalid syntax.

**Why the Other Options Are Wrong:**
A, C, D. All incorrectly assume some form of coexistence is possible.

**Concept Tested:** NATURAL JOIN is syntactically incompatible with ON/USING.

**Exam Trap:** Assuming you can "layer" NATURAL JOIN with more precise join conditions to fix its all-shared-columns behavior — the correct fix is actually to abandon NATURAL JOIN in favor of explicit USING or ON.

---

**Question 4**
True or False: NATURAL JOIN requires the matching columns in both tables to have not just the same name, but also compatible (implicitly convertible) data types.

A. True
B. False

**Correct Answer:** A (True)

**Detailed Explanation:**
Same name alone isn't sufficient — NATURAL JOIN also requires the columns to be of compatible data types for the equality comparison to make sense; a same-named column pair with incompatible types would not be used to join (or would raise an error, depending on the specific type mismatch).

**Concept Tested:** NATURAL JOIN's dual requirement — name match AND type compatibility.

**Exam Trap:** Assuming name-matching alone fully determines NATURAL JOIN's behavior, overlooking the type-compatibility requirement.

---

**Question 5**
Given the risks illustrated in Question 1, which alternative is generally recommended over NATURAL JOIN for production code, while still avoiding fully-qualified ON conditions for simple single-column joins?

A. CROSS JOIN
B. `JOIN ... USING (department_id)` — explicitly naming only the intended shared column, avoiding NATURAL JOIN's "join on every same-named column" risk while still keeping the concise unqualified-column style.
C. There is no safer alternative; NATURAL JOIN is the only concise option.
D. A theta join using `<>`.

**Correct Answer:** B

**Detailed Explanation:**
USING gives you NATURAL JOIN's terseness (no repeated `table.column = table.column` boilerplate) while requiring you to explicitly name exactly which column(s) should drive the join — sidestepping the danger of unintended extra matches from other same-named columns.

**Why the Other Options Are Wrong:**
A. Solves a completely different problem (no filtering at all).
C. USING is exactly this safer alternative.
D. `<>` produces a "not equal" condition — nonsensical as a primary join key match.

**Concept Tested:** USING as the safer, explicit middle ground between NATURAL JOIN's automatic (risky) behavior and ON's full verbosity.

**Exam Trap:** Assuming NATURAL JOIN's conciseness has no viable substitute, when USING offers the same brevity with explicit column control.

---

### Subtopic 8: OUTER JOIN — LEFT, RIGHT, FULL

**Question 1**
What does `SELECT e.last_name, d.department_name FROM employees e LEFT OUTER JOIN departments d ON e.department_id = d.department_id;` guarantee?

A. Every row from DEPARTMENTS appears at least once, regardless of whether it has any matching employees.
B. Every row from EMPLOYEES appears at least once; for employees with no matching department (e.g., a NULL or orphaned department_id), department_name displays as NULL rather than the row being dropped.
C. Only employees who have a matching department appear — functionally identical to an INNER JOIN.
D. Every possible combination of employees and departments appears, matched or not (a full Cartesian product).

**Correct Answer:** B

**Detailed Explanation:**
LEFT OUTER JOIN preserves every row from the "left" table (the one listed first / before the JOIN keyword) unconditionally. Where no matching right-side (DEPARTMENTS) row exists, the right-side columns are NULL-extended rather than causing the row to disappear — this is the entire point of an outer join versus an inner join.

**Why the Other Options Are Wrong:**
A. Describes RIGHT OUTER JOIN's guarantee instead, applied to the wrong table.
C. Describes plain INNER JOIN behavior — the opposite of what makes this an outer join.
D. Describes CROSS JOIN — no condition-based matching at all.

**Concept Tested:** LEFT OUTER JOIN's row-preservation guarantee for the left-side table.

**Exam Trap:** Mixing up which side ("left" = first-listed table) is the preserved, guaranteed side.

---

**Question 2**
Which two are TRUE about `RIGHT OUTER JOIN`? **(Choose two)**

A. It preserves every row from the table listed **after** the JOIN keyword (the "right" table), NULL-extending unmatched left-side columns.
B. `a RIGHT OUTER JOIN b` produces the exact same result as `b LEFT OUTER JOIN a` (with the ON condition otherwise unchanged) — the two keywords let you express the same relationship by swapping which table you name as "preserved" and which order you list them.
C. RIGHT OUTER JOIN cannot be used with the ANSI JOIN syntax; it requires the old `(+)` operator.
D. RIGHT OUTER JOIN always returns more rows than LEFT OUTER JOIN on the same two tables.

**Correct Answers:** A and B

**Detailed Explanation:**
RIGHT OUTER JOIN is the mirror image of LEFT OUTER JOIN — it guarantees every row of the second-named table survives, NULL-extending the first table's columns where there's no match. Because "left" and "right" are purely about table listing order, any RIGHT OUTER JOIN can be equivalently rewritten as a LEFT OUTER JOIN by swapping the table order.

**Why the Other Options Are Wrong:**
C. RIGHT OUTER JOIN is fully supported in ANSI syntax — no need for the legacy `(+)` operator.
D. Which side returns more rows depends entirely on the actual data (how many unmatched rows exist on each side) — there's no universal guarantee either way.

**Concept Tested:** RIGHT OUTER JOIN as LEFT OUTER JOIN's mirror image, differing only in table order/naming.

**Exam Trap:** Assuming RIGHT and LEFT OUTER JOIN are fundamentally different operations rather than the same operation viewed from opposite table orderings.

---

**Question 3**
What does `FULL OUTER JOIN` guarantee, that neither LEFT nor RIGHT OUTER JOIN alone guarantees?

A. Every row from *both* tables is preserved — unmatched rows from the left table appear with NULLs on the right side, AND unmatched rows from the right table appear with NULLs on the left side, in the same single result set.
B. Every possible row combination between the two tables (a Cartesian product), with no NULL-extension at all.
C. Only rows that exist in both tables, exactly like an INNER JOIN.
D. FULL OUTER JOIN is simply another name for CROSS JOIN.

**Correct Answer:** A

**Detailed Explanation:**
FULL OUTER JOIN is effectively the union of LEFT OUTER JOIN and RIGHT OUTER JOIN's results: every row from either table is guaranteed to appear at least once, with NULLs filling in for whichever side lacks a match for that particular row.

**Why the Other Options Are Wrong:**
B. Describes CROSS JOIN — a completely different, condition-free operation.
C. Describes INNER JOIN — the opposite guarantee (only matched rows).
D. FULL OUTER JOIN and CROSS JOIN are unrelated concepts.

**Concept Tested:** FULL OUTER JOIN as the union of LEFT and RIGHT OUTER JOIN guarantees.

**Exam Trap:** Assuming "FULL" implies "every combination" (Cartesian-style) rather than "every row from both sides, each appearing at least once."

---

**Question 4**
Using the legacy `(+)` operator, which statement is TRUE?

A. `(+)` can be used on both sides of the same join condition simultaneously to express a FULL OUTER JOIN.
B. `(+)` can only be applied to one side of a given join condition; you cannot mark both operands with `(+)` in the same condition, which is precisely why `(+)` alone cannot express a FULL OUTER JOIN (Oracle raises ORA-01468) — you'd need ANSI's `FULL OUTER JOIN` keyword instead, or a UNION of a LEFT and a RIGHT outer join.
C. `(+)` is purely cosmetic and has no effect on query results.
D. `(+)` must be placed on the same side as the "preserved" (guaranteed-to-appear) table.

**Correct Answer:** B

**Detailed Explanation:**
The `(+)` marker is placed on the side of the join condition belonging to the table that may be **missing** a match (the "optional" side) — the opposite table is the one whose rows are unconditionally preserved. Because `(+)` can only decorate one side of a given equality condition, expressing "preserve both sides" (a full outer join) isn't achievable with `(+)` alone; Oracle explicitly rejects the attempt to place it on both operands.

**Why the Other Options Are Wrong:**
A. Directly contradicted — this specific attempt raises an error.
C. `(+)` fundamentally changes which rows are preserved/NULL-extended — far from cosmetic.
D. Reverses the actual placement rule — `(+)` goes on the optional/"might be missing" side, not the guaranteed side.

**Concept Tested:** The `(+)` operator's one-side-only restriction and its inability to directly express FULL OUTER JOIN.

**Exam Trap:** Assuming `(+)` is a fully general substitute for ANSI OUTER JOIN keywords in every scenario, including FULL OUTER JOIN — it specifically cannot express that one case unaided.

---

**Question 5**
You write:
```sql
SELECT e.last_name, d.department_name
FROM employees e LEFT OUTER JOIN departments d
  ON e.department_id = d.department_id
WHERE d.location_id = 1700;
```
What is the likely, often-unintended consequence of adding this WHERE condition?

A. None — LEFT OUTER JOIN behavior is fully preserved regardless of any additional WHERE conditions.
B. The WHERE condition filters out every row where `d.location_id` is NULL — which includes all the NULL-extended rows the LEFT OUTER JOIN was specifically preserving (employees with no matching department) — effectively collapsing the query's behavior back down to that of an INNER JOIN.
C. It raises a syntax error, since WHERE cannot reference a column from the "optional" side of an outer join.
D. It has no effect, since d.location_id is automatically treated as matching for NULL-extended rows.

**Correct Answer:** B

**Detailed Explanation:**
This is one of the most common real-world outer-join bugs: an outer-joined row with no matching department gets `d.location_id = NULL`. The condition `d.location_id = 1700` then evaluates to UNKNOWN for that row (three-valued logic again), and WHERE excludes any row that isn't definitively TRUE — silently discarding exactly the NULL-extended rows the LEFT OUTER JOIN was meant to preserve. The fix is typically to move such a condition into the ON clause instead of WHERE, or to explicitly account for the NULL case (e.g., `d.location_id = 1700 OR d.location_id IS NULL`).

**Why the Other Options Are Wrong:**
A, D. Both wrongly assume outer-join NULL-extension is somehow immune to subsequent WHERE filtering.
C. No such syntax restriction exists — the query is syntactically valid; the problem is purely semantic/logical.

**Concept Tested:** How a post-join WHERE filter on the "optional" side's column can silently negate an outer join's entire purpose.

**Exam Trap:** This is arguably the single most consequential OUTER JOIN trap on certification exams — recognizing that placing a filter condition on the nullable side in WHERE (rather than ON) reintroduces inner-join-like row loss.

---

### Subtopic 9: SELF JOIN

**Question 1**
What is a "self join," and how is it typically written?

A. A join where a table is compared against itself using two different aliases, since SQL requires distinguishable names to reference "two copies" of the same table within one query.
B. A special dedicated keyword, `SELF JOIN`, distinct from INNER/OUTER/CROSS JOIN.
C. A join that can only be performed using the old comma+WHERE syntax, never ANSI JOIN syntax.
D. A join that always returns zero rows, since a table can never match itself.

**Correct Answer:** A

**Detailed Explanation:**
A self join treats one physical table as if it were two separate logical tables via aliasing — e.g., `employees e1 JOIN employees e2 ON e1.manager_id = e2.employee_id` (finding each employee's manager, who is also stored as a row in the same EMPLOYEES table). This is achieved with ordinary join syntax and aliases; there's no special "SELF JOIN" keyword.

**Why the Other Options Are Wrong:**
B. No such dedicated keyword exists — self joins are expressed using standard INNER/OUTER JOIN or old-style syntax with aliases.
C. ANSI JOIN syntax handles self joins perfectly well, exactly like any other join, provided aliases are used.
D. Self joins routinely return meaningful, non-empty results (e.g., the classic employee/manager example).

**Concept Tested:** Self joins are an aliasing technique applied to ordinary join syntax, not a distinct join type/keyword.

**Exam Trap:** Assuming "self join" implies special syntax or restricted capabilities, rather than recognizing it as a straightforward application of table aliasing.

---

**Question 2**
Which query correctly retrieves each employee's name alongside their manager's name, using a self join?

A. `SELECT e.last_name, m.last_name AS manager_name FROM employees e JOIN employees m ON e.manager_id = m.employee_id;`
B. `SELECT e.last_name, m.last_name AS manager_name FROM employees e, employees m WHERE e.manager_id = e.employee_id;`
C. `SELECT last_name, manager_id FROM employees NATURAL JOIN employees;`
D. `SELECT e.last_name FROM employees e JOIN employees e ON e.manager_id = e.employee_id;`

**Correct Answer:** A

**Detailed Explanation:**
Two distinct aliases (`e` for the "employee" role, `m` for the "manager" role) let the same underlying table be referenced twice with unambiguous column resolution — `e.manager_id` looks up the manager's employee_id in the "m" copy of the table.

**Why the Other Options Are Wrong:**
B. The WHERE condition mistakenly compares `e.manager_id` to `e.employee_id` (the same alias twice) rather than to `m.employee_id` — this would only match rows where an employee is literally their own manager, not the intended relationship.
C. NATURAL JOIN requires two *different* table references to alias against each other, and joining a table to itself this way with no aliasing is invalid; NATURAL JOIN would also be a poor/dangerous choice here since virtually every column name would match itself.
D. Reuses the same alias `e` for both references — Oracle requires distinct aliases to distinguish the two logical copies of the table; this raises an error.

**Concept Tested:** Correct alias usage as the mechanism that makes self joins possible and unambiguous.

**Exam Trap:** Subtly reusing the same alias for both "copies" of the table, or miswriting the ON/WHERE condition to compare a column against itself under the same alias rather than across the two distinct aliases.

---

**Question 3**
Why is `NATURAL JOIN` generally a poor choice for a self join, such as matching employees to their managers?

A. NATURAL JOIN cannot be used with any table that has a PRIMARY KEY.
B. Since both "copies" of the table share literally every column name (they're the same table), NATURAL JOIN would attempt to match on *all* identically-named columns simultaneously (employee_id, last_name, salary, hire_date, etc.) — not just the intended manager_id-to-employee_id relationship — producing a nonsensical or empty result.
C. NATURAL JOIN requires at least one column with a different name between the two table references.
D. NATURAL JOIN works correctly here and is actually the recommended approach.

**Correct Answer:** B

**Detailed Explanation:**
This extends the Subtopic 7 danger to its most extreme case: since a self join's two "sides" are the exact same table structure, NATURAL JOIN's blanket "match every same-named column" behavior becomes maximally unhelpful — it isn't looking for the one meaningful relationship (manager_id ↔ employee_id, which don't even share a name), but instead tries to match every column against its identical counterpart, which is not the intended logic at all.

**Why the Other Options Are Wrong:**
A. Unrelated restriction that doesn't exist.
C. NATURAL JOIN has no such requirement — the problem here is the *opposite*, too much column-name overlap.
D. Directly contradicts the explanation above; NATURAL JOIN is a poor fit here.

**Concept Tested:** Extending the "NATURAL JOIN matches every shared column name" danger to the extreme case of a self join.

**Exam Trap:** Not recognizing that self joins are exactly the scenario where NATURAL JOIN's automatic behavior is least appropriate, since the two "tables" are maximally similar by definition.

---

**Question 4**
Which query finds employees who currently have **no manager** (manager_id is NULL), using a self join?

A. `SELECT e.last_name FROM employees e JOIN employees m ON e.manager_id = m.employee_id WHERE m.employee_id IS NULL;`
B. `SELECT e.last_name FROM employees e LEFT OUTER JOIN employees m ON e.manager_id = m.employee_id WHERE m.employee_id IS NULL;`
C. `SELECT e.last_name FROM employees e WHERE e.manager_id IS NULL;` — this alone, with no join at all, is simpler and would work too, but the question specifically asks for a self-join formulation, and B correctly provides one.
D. Both B and C would correctly identify these employees, but only B demonstrates the self-join technique using an outer join to surface unmatched rows.

**Correct Answer:** D

**Detailed Explanation:**
B is the canonical self-join pattern for finding "rows with no match": a LEFT OUTER JOIN from employees (e) to employees (m) on the manager relationship, followed by filtering for `m.employee_id IS NULL` — meaning no manager row was found, since manager_id itself was NULL (or referenced a non-existent employee). Option A uses an INNER JOIN, which would (per Subtopic 6, Question 3) exclude exactly those NULL manager_id rows before the WHERE clause even runs, producing zero results instead of the intended answer.

**Why the Other Options Are Wrong:**
A. INNER JOIN drops NULL-valued manager_id rows entirely (no match possible against NULL) — so `m.employee_id IS NULL` afterward would find nothing, since no unmatched rows survive to be checked.
C. Correctly identifies the same rows more simply, but doesn't use a self join, which the question specifically asked for.

**Concept Tested:** Using a self LEFT OUTER JOIN plus an IS NULL check on the joined side to find "no match" rows — a direct combination of Subtopic 8's outer-join concepts with self-join aliasing.

**Exam Trap:** Using INNER JOIN instead of LEFT OUTER JOIN for a self join intended to find *unmatched* rows — since INNER JOIN would have already discarded the very rows the query is trying to find.

---

**Question 5**
True or False: A self join can only compare a table to itself using `=` (an equi self join); non-equi self joins are not possible.

A. True
B. False

**Correct Answer:** B (False)

**Detailed Explanation:**
Self joins follow the exact same rules as joins between two different tables — any comparison operator is valid in the ON/WHERE condition. A non-equi self join is entirely possible, e.g., comparing every employee's salary against every other employee's salary within the same department to find pairs where one earns more than another (`e1.salary > e2.salary AND e1.department_id = e2.department_id`).

**Concept Tested:** Self joins support the full range of theta-join operators, not just equality.

**Exam Trap:** Assuming self joins are a fundamentally restricted or special-cased join type with narrower operator support than ordinary two-table joins, rather than simply "the same join mechanics, applied to one table referenced twice."