# Oracle SQL Certification Practice — Topic 5
## Subqueries: Advantages, Rules, DML Usage, Scalar/Single/Multi-Row Types, IN/ANY/ALL/SOME, Correlated Subqueries & EXISTS

---

### Subtopic 1: Advantages & Rules of Subqueries

**Question 1**
Which of the following best describes a genuine **advantage** of using a subquery instead of hardcoding a literal value?

A. Subqueries always execute faster than any equivalent join.
B. A subquery lets a query's filtering condition be computed dynamically from the current state of the data at execution time — e.g., "employees earning more than the current average salary" — without the author needing to know or hardcode that average in advance, and the condition stays correct even as the underlying data changes.
C. Subqueries eliminate the need for a WHERE clause entirely.
D. Subqueries can only be used with SELECT statements, never with DML.

**Correct Answer:** B

**Detailed Explanation:**
The core advantage of a subquery is decoupling a query's logic from a specific, known-in-advance value. Instead of a developer manually computing and pasting in "the average salary is 6500.75," the subquery computes it live, every time the query runs, automatically staying correct as data changes.

**Why the Other Options Are Wrong:**
A. Performance depends entirely on the specific query, data volume, and optimizer choices — there's no universal speed guarantee either way.
C. A subquery is typically used *within* a WHERE (or other) clause, not as a replacement for it.
D. Subqueries are fully usable in INSERT, UPDATE, and DELETE as well (see Subtopic 2).

**Concept Tested:** The core conceptual advantage of subqueries — dynamic, self-updating value computation.

**Exam Trap:** Assuming "advantage" questions are really about performance, when the certification's intended answer is almost always about correctness/dynamism/avoiding hardcoded values.

---

**Question 2**
Is the following legal? `SELECT * FROM employees WHERE department_id IN (SELECT department_id FROM departments ORDER BY department_name);`

A. Legal — ORDER BY is always permitted in any subquery, in any context.
B. Illegal as written — a subquery feeding a WHERE-clause comparison condition (IN, =, EXISTS, etc.) generally cannot include an ORDER BY clause, unless it's paired with a FETCH FIRST/OFFSET row-limiting clause. (Note: this restriction is specifically about subqueries used *as a comparison value source*; a subquery used as an inline view directly in the FROM clause, by contrast, can freely use ORDER BY — that's the basis of the classic ROWNUM/FETCH top-N pattern.)
C. Legal, but only because IN specifically is exempt from the general ORDER BY restriction that applies to other operators.
D. Illegal, and no subquery of any kind, in any clause, may ever contain an ORDER BY.

**Correct Answer:** B

**Detailed Explanation:**
This distinguishes two different subquery roles that are easy to conflate: a subquery supplying values to a WHERE-clause comparison (IN/=/EXISTS/etc.) traditionally cannot carry its own ORDER BY (since ordering doesn't affect set membership or single-value comparison anyway) — except when paired with FETCH FIRST/OFFSET to deterministically select a specific top-N row. A subquery used as an inline view (a row source in the FROM clause) is an entirely different role and freely supports ORDER BY.

**Why the Other Options Are Wrong:**
A. Overstates the permission — the restriction is real for comparison-condition subqueries.
C. IN isn't specially exempted; the same restriction applies uniformly to comparison-condition subqueries regardless of which operator introduces them.
D. Understates the permission — inline-view subqueries in FROM absolutely can and routinely do use ORDER BY.

**Concept Tested:** The ORDER BY restriction applies to comparison-condition subqueries, not to FROM-clause inline views, and has a FETCH FIRST/OFFSET exception.

**Exam Trap:** Conflating the two very different subquery roles (comparison-value source vs. inline view row source) and applying one role's rule to the other.

---

**Question 3**
Must a subquery always be written on the **right-hand side** of a comparison operator?

A. Yes — Oracle requires the subquery to always follow the operator.
B. No — a subquery can legally appear on either side of a comparison operator, e.g., `(SELECT MAX(salary) FROM employees) > 10000` is just as valid as `10000 < (SELECT MAX(salary) FROM employees)`.
C. No, but only EXISTS/NOT EXISTS subqueries can appear on the left side.
D. Yes, except when the subquery is correlated.

**Correct Answer:** B

**Detailed Explanation:**
There is no positional restriction — a subquery is simply an expression enclosed in parentheses, and expressions can appear on either side of a comparison operator. Convention (readability) often places it on the right, but this is a style choice, not a syntax rule.

**Why the Other Options Are Wrong:**
A, D. Both invent a positional restriction that doesn't exist.
C. EXISTS/NOT EXISTS aren't comparison operators with a "left side" at all — they're unary tests on the subquery itself, unrelated to this question.

**Concept Tested:** No fixed left/right positional requirement for comparison subqueries.

**Exam Trap:** Mistaking a common stylistic convention (right-hand placement) for an enforced syntax rule.

---

**Question 4**
How many levels deep can subqueries be nested (a subquery inside a subquery inside a subquery, and so on) in a WHERE or HAVING clause?

A. Exactly 1 level — a subquery can never itself contain another subquery.
B. Exactly 2 levels, matching the aggregate-function nesting limit from Topic 3.
C. Oracle supports many levels of nesting (well beyond what any realistic certification question would require you to count) — there is no small, easily-hit limit for WHERE/HAVING subqueries.
D. Unlimited nesting is supported in the FROM clause, but WHERE/HAVING subqueries cannot be nested at all.

**Correct Answer:** C

**Detailed Explanation:**
Unlike the aggregate-function 2-level nesting rule (an entirely different, unrelated restriction from Topic 3), subqueries can be nested to a substantial depth — Oracle documents support for up to 255 levels in a WHERE clause, far beyond anything tested conceptually. The practical exam takeaway is simply: nesting subqueries within subqueries is normal and well-supported, not artificially restricted to 1 or 2 levels.

**Why the Other Options Are Wrong:**
A, B, D. All invent small, incorrect nesting limits, and B specifically confuses this with the unrelated aggregate-function nesting rule.

**Concept Tested:** Subquery nesting depth is not meaningfully restricted, unlike aggregate function nesting.

**Exam Trap:** Cross-contaminating the aggregate-function "nest to depth 2" rule (Topic 3) with subquery nesting, which is a completely separate concept with a much higher practical limit.

---

**Question 5**
Which statement about subquery **rules** is TRUE?

A. A subquery used with a single-row comparison operator (like `=`) must be guaranteed, by the query's own logic, to return at most one row — Oracle does not verify this at compile time; if it unexpectedly returns multiple rows at runtime, the query fails.
B. Oracle performs static analysis at compile time to guarantee a single-row subquery will never return more than one row, rejecting the query upfront if it can't prove this.
C. A subquery can never reference a column from a table that isn't listed in its own FROM clause.
D. A subquery is only legal if it is the very first clause after WHERE — it cannot appear after AND/OR.

**Correct Answer:** A

**Detailed Explanation:**
The single-row-vs-multi-row operator distinction is a *runtime* concern, not something Oracle validates at parse/compile time — the database has no way to know in advance how many rows a subquery will produce for a given data state. If a single-row operator's subquery unexpectedly returns more than one row when executed, Oracle raises ORA-01427 at runtime, not a compile-time syntax error.

**Why the Other Options Are Wrong:**
B. Directly contradicts A — no such compile-time guarantee exists or is even possible in the general case.
C. A subquery can absolutely reference outer-query columns (that's precisely what makes it "correlated" — see Subtopic 6).
D. Subqueries can appear anywhere a value/condition is expected, including after AND/OR within a larger compound WHERE condition.

**Concept Tested:** The single-row/multi-row operator mismatch is a runtime error, not a compile-time-checked constraint.

**Exam Trap:** Assuming Oracle can somehow "know in advance" a subquery will misbehave, rather than recognizing this is only discoverable when the query actually executes against real data.

---

### Subtopic 2: Using Subqueries with SELECT, INSERT, UPDATE, DELETE

**Question 1**
Which of these correctly uses a subquery within an **UPDATE** statement?

A. `UPDATE employees SET salary = salary * 1.1 SUBQUERY (SELECT department_id FROM departments);`
B. `UPDATE employees SET department_id = (SELECT department_id FROM departments WHERE department_name = 'IT Support') WHERE last_name = 'King';`
C. `UPDATE employees SUBQUERY SET salary = salary * 1.1;`
D. `UPDATE (SELECT * FROM employees) SET salary = salary * 1.1 USING departments;`

**Correct Answer:** B

**Detailed Explanation:**
A scalar subquery can appear directly in the SET clause, supplying the new value dynamically — here, looking up IT Support's department_id rather than hardcoding it. This is a completely standard, common pattern.

**Why the Other Options Are Wrong:**
A, C. Invent nonexistent `SUBQUERY` syntax.
D. While `UPDATE (subquery) SET ...` is valid syntax for updating through an updatable view/inline view in Oracle, the fabricated `USING departments` clause here isn't valid UPDATE syntax.

**Concept Tested:** Scalar subqueries supplying dynamically-computed values in an UPDATE's SET clause.

**Exam Trap:** Being unfamiliar with the legitimate SET-clause subquery pattern and instead reaching for an invented keyword.

---

**Question 2**
Which of these correctly uses a subquery within a **DELETE** statement to remove all employees in departments located in `'Seattle'`?

A. `DELETE FROM employees WHERE department_id IN (SELECT department_id FROM departments WHERE city = 'Seattle');`
B. `DELETE FROM employees SUBQUERY department_id = (SELECT department_id FROM departments WHERE city = 'Seattle');`
C. `DELETE employees, departments WHERE employees.department_id = departments.department_id AND city = 'Seattle';`
D. Subqueries cannot be used in DELETE statements; a join must be used instead.

**Correct Answer:** A

**Detailed Explanation:**
A multi-row subquery combined with IN is the standard, fully-supported way to delete rows based on a related condition in another table, without needing any special multi-table DELETE syntax.

**Why the Other Options Are Wrong:**
B. Fabricated syntax.
C. Oracle's DELETE statement doesn't support this multi-table comma syntax the way SELECT's FROM clause does.
D. Directly contradicted — subqueries are fully standard in DELETE's WHERE clause.

**Concept Tested:** Multi-row subquery with IN inside a DELETE's WHERE clause.

**Exam Trap:** Assuming DELETE has fundamentally different subquery support than SELECT/UPDATE, when the WHERE-clause mechanics are identical across all three.

---

**Question 3**
Which of these correctly uses a subquery within an **INSERT** statement to copy all current managers into an ARCHIVE_MANAGERS table?

A. `INSERT INTO archive_managers VALUES (SELECT employee_id, last_name FROM employees WHERE employee_id IN (SELECT manager_id FROM employees));`
B. `INSERT INTO archive_managers (employee_id, last_name) SELECT employee_id, last_name FROM employees WHERE employee_id IN (SELECT DISTINCT manager_id FROM employees WHERE manager_id IS NOT NULL);`
C. `INSERT INTO archive_managers SUBQUERY (SELECT employee_id, last_name FROM employees);`
D. Both A and B are equally valid, interchangeable syntaxes.

**Correct Answer:** B

**Detailed Explanation:**
This is the "INSERT ... SELECT" form — replacing the VALUES clause entirely with a SELECT statement (which may itself contain a nested subquery, here to identify distinct manager IDs). This is the correct, standard way to bulk-insert rows derived from a query.

**Why the Other Options Are Wrong:**
A. Cannot mix VALUES with a bare SELECT this way — VALUES expects literal expressions, not an embedded query; this is invalid syntax.
C. Fabricated syntax.
D. A is invalid, so the two are not interchangeable.

**Concept Tested:** INSERT ... SELECT (replacing VALUES) as the correct pattern for subquery-driven bulk inserts.

**Exam Trap:** Trying to nest a full SELECT inside a VALUES clause, rather than recognizing that VALUES and SELECT are two mutually exclusive ways of supplying an INSERT's row source.

---

**Question 4**
A developer writes: `UPDATE employees SET salary = (SELECT salary FROM employees WHERE department_id = 50);` intending to give every employee the same salary as one representative employee in department 50 — but department 50 currently has 12 employees. What happens?

A. The UPDATE succeeds, using the first row returned arbitrarily.
B. The UPDATE fails at runtime with ORA-01427: single-row subquery returns more than one row, since the SET clause's subquery is a single-row (scalar) context but the subquery returns 12 rows.
C. The UPDATE succeeds and sets every employee's salary to the average of the 12 department-50 salaries.
D. The UPDATE succeeds and creates 12 separate versions of each row, one per possible salary value.

**Correct Answer:** B

**Detailed Explanation:**
A SET clause value slot is inherently scalar — it expects exactly one value per column, per row. Supplying a subquery that returns 12 rows violates this at runtime, just as it would for any other single-row context (WHERE with `=`, SELECT list, etc.).

**Why the Other Options Are Wrong:**
A, C, D. All invent lenient fallback behaviors that Oracle does not provide — there's no arbitrary-row selection, no implicit averaging, and no row-multiplication.

**Concept Tested:** SET-clause subqueries are scalar contexts, subject to the same single-row enforcement as any other scalar usage.

**Exam Trap:** Assuming DML contexts (UPDATE's SET, INSERT's VALUES) are somehow more forgiving of multi-row subquery results than a WHERE-clause comparison — they are equally strict.

---

**Question 5**
Which of the following is a valid use of a subquery directly in a **SELECT list** (a scalar subquery)?

A. `SELECT last_name, (SELECT department_name FROM departments d WHERE d.department_id = e.department_id) AS dept FROM employees e;`
B. `SELECT last_name, department_name FROM employees, (SELECT * FROM departments);`
C. `SELECT last_name FROM employees WHERE (SELECT department_name FROM departments);`
D. All three are equally valid and produce the same result.

**Correct Answer:** A

**Detailed Explanation:**
This is a correlated scalar subquery in the SELECT list, computing one department_name value per outer employee row by referencing that row's department_id — a completely standard, well-formed pattern (foreshadowing correlated subqueries in Subtopic 6).

**Why the Other Options Are Wrong:**
B. Describes an (unjoined) inline view in the FROM clause producing a Cartesian product — a very different construct, not a SELECT-list scalar subquery, and not equivalent to A.
C. A bare subquery placed directly as a WHERE condition (with no comparison operator or EXISTS) is not valid syntax on its own.
D. Since B and C are structurally different (and C is invalid), they cannot all be equivalent.

**Concept Tested:** Correct placement and correlation of a scalar subquery in the SELECT list.

**Exam Trap:** Confusing an inline view (subquery in FROM) with a SELECT-list scalar subquery — they look superficially similar but serve entirely different roles.

---

### Subtopic 3: Scalar Subquery

**Question 1**
By definition, what must a **scalar subquery** return in order to be valid wherever a single expression is expected?

A. Any number of rows, but exactly one column.
B. Exactly one row and exactly one column — a single, individual value.
C. Exactly one row, with any number of columns.
D. Any number of rows and any number of columns, as long as GROUP BY is used.

**Correct Answer:** B

**Detailed Explanation:**
A scalar subquery must resolve to a single value — one row, one column — because it's meant to be usable anywhere a plain literal or column reference could go (SELECT list, WHERE comparisons, CASE expressions, function arguments, SET clauses, etc.).

**Why the Other Options Are Wrong:**
A, C, D. Each allows for more than a single value in some dimension (rows or columns), which would make the "one expression, one value" substitution model impossible.

**Concept Tested:** The strict one-row, one-column definition of a scalar subquery.

**Exam Trap:** Conflating "single-row subquery" (which could still return multiple columns, e.g., for a row-comparison construct) with the stricter "scalar subquery" (single row AND single column).

---

**Question 2**
What does `SELECT last_name, (SELECT MAX(salary) FROM employees WHERE department_id = 999) AS dept999_max FROM employees;` return for `dept999_max`, assuming department_id 999 doesn't exist in the table?

A. An error, since the subquery matches zero rows.
B. NULL, for every row — a scalar subquery that finds zero matching rows returns NULL rather than raising an error, exactly like the MAX() aggregate itself returning NULL over an empty set.
C. Zero (0), treating "no rows" as a numeric default.
D. The query returns zero rows entirely.

**Correct Answer:** B

**Detailed Explanation:**
This combines two established rules: MAX() over zero rows returns NULL (rather than erroring or defaulting to 0), and a scalar subquery is simply a container for that single resulting value — so if the aggregate itself resolves to NULL, that NULL is what the scalar subquery returns, without any error.

**Why the Other Options Are Wrong:**
A. No error occurs — an empty subquery result set is a normal, valid outcome for a scalar subquery, not a violation.
C. There's no implicit NULL-to-zero substitution.
D. The outer query is entirely unaffected in terms of its own row count; only the value in that one derived column becomes NULL for every row.

**Concept Tested:** A scalar subquery matching zero rows resolves to NULL, not an error.

**Exam Trap:** Assuming "the subquery found no matching data" must mean something goes wrong, rather than recognizing NULL as the well-defined, unremarkable result.

---

**Question 3**
Contrast the previous question with this one: what happens if a scalar subquery unexpectedly returns **two or more rows** at runtime (as opposed to zero)?

A. Also resolves quietly to NULL, exactly like the zero-row case.
B. Raises ORA-01427: single-row subquery returns more than one row — unlike the "zero rows → NULL" case, "too many rows" is a genuine runtime error, since Oracle has no reasonable way to pick a single value among several.
C. Automatically returns only the first row found, silently discarding the rest.
D. Automatically returns the last row found, silently discarding the rest.

**Correct Answer:** B

**Detailed Explanation:**
Zero rows and multiple rows are handled completely differently: zero rows is a well-defined "nothing found" case (NULL), while multiple rows represents a genuine ambiguity Oracle refuses to silently resolve — it has no rule for "pick one," so it raises an error instead.

**Why the Other Options Are Wrong:**
A. Wrongly equates the two very different cases (absence of data vs. ambiguous excess of data).
C, D. Both invent a silent "pick one" behavior Oracle does not implement.

**Concept Tested:** Zero-row (NULL) vs. multi-row (error) outcomes for scalar subqueries are fundamentally different, not symmetric.

**Exam Trap:** Assuming both "too few" and "too many" rows are handled the same lenient way, when in fact only the zero-row case is quietly tolerated.

---

**Question 4**
Must every scalar subquery be **correlated** (i.e., reference a column from the outer query)?

A. Yes — by definition, a scalar subquery must always depend on the outer query's current row.
B. No — a scalar subquery can be entirely non-correlated, computing one fixed value (e.g., the company-wide average salary) that is then reused identically for every row of the outer query.
C. No, but a non-correlated scalar subquery is illegal in the SELECT list specifically (only correlated ones are allowed there).
D. Yes, but only when used in the SELECT list; scalar subqueries used in WHERE may be non-correlated.

**Correct Answer:** B

**Detailed Explanation:**
"Scalar" describes the subquery's *shape* (one row, one column) — it says nothing about whether it references the outer query. A non-correlated scalar subquery (e.g., `(SELECT AVG(salary) FROM employees)`) computes once and supplies that same single value to every outer row it's paired with in the SELECT list, or to a single comparison in WHERE.

**Why the Other Options Are Wrong:**
A, D. Both wrongly tie "scalar" to "correlated," when these are independent, orthogonal properties.
C. Non-correlated scalar subqueries are extremely common and fully legal in the SELECT list (e.g., displaying "company average" alongside each employee's own salary).

**Concept Tested:** Scalar (shape) and correlated (outer-query dependency) are independent subquery properties.

**Exam Trap:** Assuming any subquery placed in the SELECT list must automatically be correlated, since that's the more commonly demonstrated pattern.

---

**Question 5**
Can a scalar subquery be passed as an argument to another function, such as `ROUND((SELECT AVG(salary) FROM employees), 2)`?

A. No — scalar subqueries can only be compared directly to a column; they cannot be nested inside another function call.
B. Yes — since a scalar subquery resolves to a single value, it can be used anywhere a single-value expression is valid, including as a function argument.
C. Yes, but only inside single-row functions, never inside aggregate functions.
D. No — this specific example is illegal because ROUND cannot accept a subquery, only a literal or column.

**Correct Answer:** B

**Detailed Explanation:**
Once again, this follows directly from the "one row, one column = usable as any expression" definition — a scalar subquery is substitutable anywhere a literal, column, or arithmetic expression could appear, including as one of ROUND's arguments.

**Why the Other Options Are Wrong:**
A, D. Both wrongly restrict scalar subquery usage to direct comparisons only.
C. No such single-row/aggregate-function distinction exists for accepting scalar-subquery arguments — both accept any valid scalar expression, subquery included.

**Concept Tested:** Full expression-substitutability of scalar subqueries, including as function arguments.

**Exam Trap:** Underestimating how broadly a scalar subquery can be used, given its strict one-value definition.

---

### Subtopic 4: Single-Row Subquery

**Question 1**
Which operators are valid for a subquery that is expected to return exactly one row?

A. `IN`, `ANY`, `ALL` only.
B. `=`, `<`, `>`, `<=`, `>=`, `<>` (the standard single-row comparison operators).
C. Only `=` — all other comparison operators require multi-row subqueries.
D. `EXISTS` and `NOT EXISTS` only.

**Correct Answer:** B

**Detailed Explanation:**
Single-row subqueries pair with the same comparison operators used for ordinary single-value comparisons, since the subquery is expected to behave like a single value being compared against.

**Why the Other Options Are Wrong:**
A. Those are specifically the multi-row operators (Subtopic 5) — using them signals the subquery may return several rows.
C. `<`, `>`, etc. are equally valid single-row operators; `=` isn't uniquely privileged.
D. EXISTS/NOT EXISTS are an entirely separate mechanism (Subtopic 7) that doesn't compare values at all — it only tests for row existence.

**Concept Tested:** The standard comparison-operator set associated with single-row subqueries.

**Exam Trap:** Mixing up which operators are "single-row" vs. "multi-row" — a foundational distinction that governs which operator is legal with which kind of subquery.

---

**Question 2**
`SELECT * FROM employees WHERE salary > (SELECT salary FROM employees WHERE employee_id = 9999);` — employee_id 9999 does not exist. What happens?

A. Raises an error, since the subquery returns zero rows and `=`-style comparisons require exactly one.
B. The subquery returns zero rows, meaning its scalar result is effectively NULL; comparing `salary > NULL` evaluates to UNKNOWN for every row, so the outer query returns **zero rows** — no error, just an empty (and possibly surprising) result.
C. The outer query returns every row, since a "no comparison value found" condition is treated as universally true.
D. Oracle substitutes 0 for the missing subquery result, so the condition becomes `salary > 0`, returning almost all rows.

**Correct Answer:** B

**Detailed Explanation:**
This combines the scalar-subquery zero-rows-means-NULL rule (Subtopic 3) with three-valued logic (Topic 1): comparing anything to NULL yields UNKNOWN, and a WHERE clause only keeps rows where the condition is definitively TRUE — so every row is silently excluded, and the query runs successfully but returns nothing.

**Why the Other Options Are Wrong:**
A. No error occurs; this is a legal (if silently empty) outcome, not a violation.
C, D. Both invent behaviors (auto-TRUE, auto-zero-substitution) that Oracle does not implement.

**Concept Tested:** Combining "zero-row scalar subquery → NULL" with "comparison to NULL → UNKNOWN → row excluded" to explain a common silently-empty-result scenario.

**Exam Trap:** Expecting an error message that never actually appears — this specific scenario fails silently, which is often more dangerous (and more heavily tested) than an outright error.

---

**Question 3**
Is `BETWEEN` compatible with single-row subqueries as its bounds, e.g., `WHERE salary BETWEEN (SELECT MIN(salary) FROM employees WHERE department_id=10) AND (SELECT MAX(salary) FROM employees WHERE department_id=10)`?

A. No — BETWEEN can never take a subquery as either bound.
B. Yes — BETWEEN is a single-row-style operator, and each bound can independently be supplied by its own single-row (here, scalar aggregate) subquery.
C. Yes, but only if both subqueries are combined into a single multi-row subquery first.
D. No — BETWEEN only accepts literal values, never expressions of any kind.

**Correct Answer:** B

**Detailed Explanation:**
BETWEEN is shorthand for two single-value comparisons (`>=` and `<=` combined), so each bound independently accepts anything a single-value comparison would — including a scalar subquery. Here, each bound is its own independent single-row aggregate subquery, cleanly computing the department's min and max salary dynamically.

**Why the Other Options Are Wrong:**
A, D. Both wrongly deny subquery usage entirely.
C. There's no need (or mechanism) to "combine" two separate scalar subqueries into one multi-row subquery for this purpose — each bound is independently scalar.

**Concept Tested:** BETWEEN's bounds can each independently be supplied by single-row (scalar) subqueries.

**Exam Trap:** Assuming BETWEEN, being a range operator, must therefore be classified as "multi-row" — it's actually built from two single-value comparisons.

---

**Question 4**
A developer knows that `department_id = 50` currently has exactly one employee named 'Weiss', but writes the query defensively anyway. Which subquery approach is more robust against a **future** data change where department 50 might gain a second employee?

A. `WHERE salary > (SELECT salary FROM employees WHERE department_id = 50);` — using `=` and assuming exactly one row will always exist.
B. `WHERE salary > ALL (SELECT salary FROM employees WHERE department_id = 50);` — using the multi-row operator ALL, which gracefully handles one row today and multiple rows in the future without needing to be rewritten.
C. Both are equally robust; there's no meaningful difference.
D. Option A is more robust, since `=` is a stricter, safer operator than ALL.

**Correct Answer:** B

**Detailed Explanation:**
This directly applies the earlier ORA-01427 lesson practically: if the underlying data's cardinality might change over time, choosing a multi-row-tolerant operator (ALL, ANY, IN) up front avoids a query that works today but silently breaks (or errors) the moment department 50 gains a second row. `> ALL` correctly means "greater than every value returned," which degrades gracefully to a simple single-value comparison when only one row exists.

**Why the Other Options Are Wrong:**
A. Works only as long as the row-count assumption holds; the moment it doesn't, this becomes a ticking ORA-01427 time bomb.
C. Understates a real robustness difference.
D. Reverses the actual risk — `=` is the fragile choice here, not the safe one.

**Concept Tested:** Choosing multi-row-tolerant operators defensively when a subquery's row count isn't structurally guaranteed to stay at one.

**Exam Trap:** Assuming stricter-looking operators (`=`) are inherently "safer" without considering whether the underlying assumption they depend on (exactly one row) is actually guaranteed by the schema.

---

**Question 5**
True or False: If a subquery is written with a single-row operator like `=`, but a UNIQUE or PRIMARY KEY constraint on the filtered column structurally guarantees the subquery can never return more than one row, Oracle will still perform the same runtime check.

A. True
B. False

**Correct Answer:** A (True)

**Detailed Explanation:**
Oracle doesn't reason about *why* a subquery happens to be safe (e.g., filtering on a PK-constrained column) — the single-row/multi-row enforcement is a uniform runtime mechanism applied to every subquery of this kind, constraint-backed or not. In practice this particular case simply never trips the error (since the constraint genuinely guarantees uniqueness), but the mechanism itself doesn't "know" that in advance — it just checks the actual row count each time.

**Concept Tested:** The runtime single-row check applies uniformly, regardless of whether a constraint happens to structurally guarantee safety.

**Exam Trap:** Assuming Oracle's query optimizer "reasons" about constraints to skip runtime checks it doesn't actually need to skip — the checking mechanism is uniform, even if it never actually fires in a constraint-guaranteed-safe case.

---

### Subtopic 5: Multiple Row Subquery — IN, NOT IN, ALL, ANY, SOME

**Question 1**
Table B's subquery result contains (100, 200, NULL). Given `WHERE dept_id NOT IN (SELECT dept_id FROM b);` (assume the outer table's dept_id has no NULLs) — what is returned?

A. All outer rows whose dept_id isn't 100 and isn't 200.
B. Zero rows — a NULL anywhere in a NOT IN subquery's result set poisons the entire comparison chain (each internally becomes an ANDed `<>` comparison, and any single UNKNOWN in an AND chain with no FALSE present prevents the whole condition from ever being TRUE), so the outer query returns nothing, regardless of its own data.
C. All rows, since NOT IN ignores NULLs found only within a subquery's result (as opposed to literal lists).
D. Only rows where dept_id IS NULL.

**Correct Answer:** B

**Detailed Explanation:**
This is the identical NOT IN/NULL trap from Topic 1, now expressed through a subquery rather than a literal list — the underlying mechanism (NOT IN as an ANDed chain of `<>` comparisons) doesn't care whether the NULL came from a hardcoded list or a subquery's result set; the effect is the same.

**Why the Other Options Are Wrong:**
A. The intuitive but incorrect expectation.
C. Wrongly assumes subquery-sourced NULLs behave differently from literal-list NULLs — they don't.
D. Unrelated to the actual failure mode here.

**Concept Tested:** The NOT IN/NULL poisoning trap applies identically whether the NULL originates from a literal list or a subquery.

**Exam Trap:** Assuming this classic trap is somehow "fixed" or different simply because the NULL now comes from a subquery instead of a hand-typed list — the mechanism (and the danger) is exactly the same.

---

**Question 2**
Given MANAGERS.SALARY values (5000, 7000, 9000) returned by a subquery, which of these correctly finds employees earning more than **every** manager?

A. `salary > ANY (SELECT salary FROM managers)`
B. `salary > ALL (SELECT salary FROM managers)`
C. `salary > SOME (SELECT salary FROM managers)`
D. Both A and C are correct and equivalent to each other.

**Correct Answer:** B

**Detailed Explanation:**
`> ALL` requires exceeding every value in the subquery's result — i.e., exceeding the maximum. This is the strict "beats everyone" condition.

**Why the Other Options Are Wrong:**
A. `> ANY` only requires exceeding at least one value (effectively, beating the minimum) — a much weaker condition.
C, D. `SOME` is simply a synonym for `ANY` (see Question 3) — so C is exactly as weak as A, and D is wrong specifically because it claims A and C together are the *correct* (strict) answer, when both are actually the weak form.

**Concept Tested:** Reinforcing ANY (weakest/minimum) vs. ALL (strictest/maximum) in a subquery-driven context.

**Exam Trap:** Being misdirected by the unfamiliar `SOME` keyword into thinking it behaves differently from `ANY`, rather than recognizing it as a pure synonym.

---

**Question 3**
What is the relationship between `SOME` and `ANY` in Oracle SQL?

A. `SOME` is stricter than `ANY` — it requires matching a majority of the subquery's returned values.
B. `SOME` is a pure, fully interchangeable synonym for `ANY` — `= SOME (subquery)` behaves identically to `= ANY (subquery)`, with no behavioral difference whatsoever.
C. `SOME` can only be used with numeric subqueries; `ANY` works with any data type.
D. `SOME` is not valid Oracle syntax at all.

**Correct Answer:** B

**Detailed Explanation:**
Per the ANSI SQL standard (which Oracle follows here), `SOME` and `ANY` are exact synonyms — there is zero difference in behavior, evaluation, or supported data types between them. `SOME` exists purely as an alternate keyword for readability preference.

**Why the Other Options Are Wrong:**
A, C, D. Each invents a distinction that doesn't exist.

**Concept Tested:** SOME/ANY synonym relationship.

**Exam Trap:** Assuming an unfamiliar-sounding keyword (SOME) must carry some special, different behavior rather than simply being a drop-in alias.

---

**Question 4**
What is the relationship between `= ANY (subquery)` and `IN (subquery)`?

A. They are completely unrelated constructs that happen to look similar.
B. `= ANY (subquery)` is functionally equivalent to `IN (subquery)` — both test whether the left-hand value matches at least one value returned by the subquery.
C. `= ANY` is stricter than IN — it requires matching every value the subquery returns.
D. IN can only be used with literal lists, never with subqueries; `= ANY` is required for subquery-based multi-row comparisons.

**Correct Answer:** B

**Detailed Explanation:**
`IN` is essentially syntactic sugar for `= ANY` — both check "does this value equal at least one of the subquery's returned values?" They are interchangeable for this specific comparison.

**Why the Other Options Are Wrong:**
A. They are directly, provably equivalent, not merely similar-looking.
C. Describes `= ALL`'s behavior (matching every value, which is actually only satisfiable if the subquery returns a single repeated value or one row), not `= ANY`'s.
D. IN works perfectly well with subqueries — it's one of its most common uses (see Question 1 of this subtopic).

**Concept Tested:** IN ≡ = ANY equivalence.

**Exam Trap:** Assuming IN and ANY-based operators are fundamentally different mechanisms rather than recognizing IN as shorthand for the equality case of ANY.

---

**Question 5**
What is the relationship between `<> ALL (subquery)` and `NOT IN (subquery)`, versus `<> ANY (subquery)`?

A. All three are fully equivalent to each other.
B. `<> ALL (subquery)` is equivalent to `NOT IN (subquery)` (both require the value to differ from *every* returned value) — but `<> ANY (subquery)` is a much weaker condition, satisfied as soon as the value differs from at least *one* returned value, which is true almost all the time unless the subquery returns only a single repeated value equal to the tested value.
C. `<> ANY` and `NOT IN` are equivalent; `<> ALL` is the weaker one.
D. None of these three constructs are related to one another.

**Correct Answer:** B

**Detailed Explanation:**
This mirrors the `= ANY`/`= ALL` relationship, but inverted: `NOT IN` matches `<> ALL`'s "differs from everything" semantics, not `<> ANY`'s much weaker "differs from at least one thing" semantics. Confusing `<> ANY` for a NOT-IN-equivalent is a classic, high-value certification trap, since `<> ANY` is nearly always TRUE (it only fails if the subquery returns a single value, or multiple copies of the same value, matching the tested value exactly).

**Why the Other Options Are Wrong:**
A. Directly contradicted — `<> ANY` is materially different (weaker) than the other two.
C. Reverses the correct pairing.
D. Understates the genuine, well-defined equivalence between `<> ALL` and `NOT IN`.

**Concept Tested:** `<> ALL` ≡ `NOT IN` (strict, "differs from everything"); `<> ANY` is a much weaker, frequently-almost-always-true condition.

**Exam Trap:** This is arguably the single most consequential ANY/ALL trap on certification exams — reflexively assuming `<>` pairs with `ANY` the same intuitive way `=` pairs with `ANY` (for IN-equivalence), when the negated case actually pairs with `ALL` instead.

---

### Subtopic 6: Correlated Subqueries

**Question 1**
What defines a **correlated subquery**, as distinct from an ordinary (non-correlated) subquery?

A. A correlated subquery must always use the EXISTS operator.
B. A correlated subquery references one or more columns from the *outer* query, meaning it cannot be meaningfully executed as a standalone statement on its own — its result depends on which outer row is currently being evaluated.
C. A correlated subquery must return multiple rows; a non-correlated subquery must return exactly one.
D. A correlated subquery can only appear in the WHERE clause, never the SELECT list.

**Correct Answer:** B

**Detailed Explanation:**
The defining feature of correlation is the outer-query column reference — the subquery's meaning and result literally change depending on the current outer row's values, making it logically (though not necessarily physically, depending on optimizer transformations) re-evaluated once per outer row.

**Why the Other Options Are Wrong:**
A. EXISTS is commonly paired with correlated subqueries (Subtopic 7) but isn't a requirement — plenty of correlated subqueries use ordinary comparison operators instead.
C. Row-count expectations (scalar/single-row/multi-row) are an entirely separate, orthogonal classification from correlation.
D. Correlated scalar subqueries in the SELECT list are common and legal (as shown in Subtopic 2, Question 5).

**Concept Tested:** The defining characteristic of correlation — dependency on outer-query column values.

**Exam Trap:** Conflating correlation with a specific operator (EXISTS) or clause placement, rather than recognizing it as being purely about outer-column referencing.

---

**Question 2**
Which query correctly finds employees who earn more than the **average salary of their own department** (a classic correlated-subquery use case)?

A. `SELECT last_name FROM employees e WHERE salary > (SELECT AVG(salary) FROM employees);`
B. `SELECT last_name FROM employees e WHERE salary > (SELECT AVG(salary) FROM employees i WHERE i.department_id = e.department_id);`
C. `SELECT last_name FROM employees WHERE salary > (SELECT AVG(salary) FROM employees GROUP BY department_id);`
D. `SELECT last_name FROM employees e WHERE salary > (SELECT AVG(salary) FROM departments WHERE department_id = e.department_id);`

**Correct Answer:** B

**Detailed Explanation:**
This is the textbook correlated subquery: the inner query's `i.department_id = e.department_id` ties the inner AVG computation specifically to whichever department the *current outer row* belongs to, so each employee is compared against their own department's average — not the company-wide average.

**Why the Other Options Are Wrong:**
A. Non-correlated — computes one single company-wide average, applied uniformly to every employee, not a per-department comparison.
C. Attempting to add GROUP BY here without a correlation condition would return multiple rows (one average per department) — as a single-row-context subquery, this would raise ORA-01427 the moment more than one department exists, since there's no correlation tying it to a single specific department per outer row.
D. Averaging over DEPARTMENTS (which doesn't have a salary column) is nonsensical/invalid — this option confuses the base table.

**Concept Tested:** Correctly constructing the correlation condition to tie an inner aggregate to each outer row's specific group.

**Exam Trap:** Writing a superficially similar-looking subquery that's actually non-correlated (A) or that misuses GROUP BY where correlation was needed instead (C) — both are extremely common near-miss mistakes for this exact "salary vs. department average" pattern.

---

**Question 3**
Conceptually (logical execution model, regardless of internal optimizer transformations), how often is a correlated subquery evaluated?

A. Exactly once, regardless of the outer query's row count.
B. Once for each row processed by the outer query, since the subquery's outer-column reference changes with each row.
C. Exactly twice — once to validate syntax, once to execute.
D. Zero times if the outer query returns zero rows, and exactly once otherwise.

**Correct Answer:** B

**Detailed Explanation:**
Because the subquery's meaning depends on the current outer row's value(s), the conceptual (logical) model is "re-run this subquery for every outer row, plugging in that row's correlation value." (Oracle's actual optimizer may transform this into a more efficient join-like execution plan internally, but the *logical* row-by-row model is what defines correlated subquery semantics and what's tested at the SQL-language level.)

**Why the Other Options Are Wrong:**
A. Describes non-correlated subquery behavior instead (compute once, reuse for every row).
C. Not a meaningful description of subquery execution at all.
D. Wrongly assumes a flat one-or-zero execution count regardless of the outer row count.

**Concept Tested:** The logical (once-per-outer-row) execution model that defines correlated subquery semantics.

**Exam Trap:** Confusing the logical/conceptual execution model (which is what defines correctness and is testable) with actual physical optimizer execution plans (which can differ for performance reasons but shouldn't change the certification-level understanding of "how correlation works").

---

**Question 4**
In a correlated subquery, why is it necessary to use an alias for the outer table when the inner subquery also queries the same table?

A. It isn't necessary; aliases are purely optional stylistic choices even in this scenario.
B. Without distinct aliases for the outer and inner references to the same table, Oracle cannot determine which "copy" of the table a given column reference belongs to — exactly the same ambiguity problem addressed by aliasing in self joins (Topic 4).
C. Oracle automatically assumes the innermost reference always refers to the inner subquery, making aliases unnecessary.
D. Aliases are required only for performance reasons, not correctness.

**Correct Answer:** B

**Detailed Explanation:**
This directly parallels the self-join aliasing requirement from Topic 4: when a correlated subquery queries the same table as its outer query (as in the department-average example), both references need distinct aliases (`e` and `i`, for instance) so that `i.department_id` and `e.department_id` are unambiguous.

**Why the Other Options Are Wrong:**
A. Understates a genuine correctness requirement, not just a style preference, in this same-table scenario.
C. No such automatic disambiguation exists; Oracle requires explicit, distinct aliases.
D. This is fundamentally a correctness/ambiguity issue, not a performance consideration.

**Concept Tested:** Aliasing requirement for correlated subqueries referencing the same table as the outer query — directly connecting back to the self-join aliasing concept.

**Exam Trap:** Assuming aliases in this context are "just good practice" rather than an outright correctness requirement once the same table appears on both sides of the correlation.

---

**Question 5**
True or False: A correlated subquery can appear in a SELECT list, computing one value per outer row that depends on that row's own column values.

A. True
B. False

**Correct Answer:** A (True)

**Detailed Explanation:**
This was already demonstrated in Subtopic 2, Question 5 (department_name looked up per employee via `d.department_id = e.department_id`) — a correlated scalar subquery is a completely standard SELECT-list pattern, not restricted to WHERE clauses.

**Concept Tested:** Correlated subqueries are not restricted to WHERE — they're equally valid (and common) in the SELECT list.

**Exam Trap:** Assuming correlation is somehow a "WHERE-clause-only" concept, rather than a general property that applies wherever a subquery can legally appear.

---

### Subtopic 7: Usage of EXISTS, NOT EXISTS

**Question 1**
What does `EXISTS (subquery)` actually test?

A. Whether the subquery's returned values match a specific comparison value.
B. Simply whether the subquery returns **at least one row** — the actual column values or data inside that row are irrelevant to EXISTS; it only cares about row presence/absence, which is why the subquery is often written as `SELECT 1 ...` or `SELECT * ...` rather than any specific meaningful column.
C. Whether the subquery returns exactly one row, no more and no fewer.
D. Whether the subquery's table has at least one row total, ignoring any WHERE condition inside the subquery.

**Correct Answer:** B

**Detailed Explanation:**
EXISTS is fundamentally a row-presence test, not a value-comparison test. This is precisely why the conventional `SELECT 1 FROM ...` (or `SELECT 'X' FROM ...`) style is so common inside EXISTS subqueries — the selected column(s) are functionally irrelevant; only whether any row satisfies the subquery's own WHERE condition matters.

**Why the Other Options Are Wrong:**
A. That's the role of comparison operators (=, IN, ANY, ALL), not EXISTS.
C. EXISTS is satisfied by one or more rows — it doesn't care about the exact count once it's non-zero.
D. The subquery's own WHERE condition (including any correlation) absolutely matters — EXISTS tests row existence *given* that condition, not the base table's total row count.

**Concept Tested:** EXISTS as a pure existence test, indifferent to which/how many columns are selected.

**Exam Trap:** Assuming EXISTS somehow inspects or compares the selected column values, rather than purely checking for row presence.

---

**Question 2**
Why is `NOT EXISTS` generally considered a **safer** alternative to `NOT IN` when the subquery's result might contain NULLs?

A. NOT EXISTS is faster in every case, which is why it's "safer."
B. NOT EXISTS never evaluates equality against individual subquery values at all — it merely checks whether *any* row satisfies the (usually correlated) subquery condition. Since there's no value-by-value equality comparison happening, a NULL appearing in some unrelated column of the subquery's underlying data doesn't have the same poisoning effect that it does for NOT IN's ANDed chain of `<>` comparisons.
C. NOT EXISTS automatically filters out NULL rows from the subquery before evaluating, while NOT IN does not.
D. There is no real safety difference; this is a myth.

**Correct Answer:** B

**Detailed Explanation:**
This is the resolution to the NOT IN/NULL trap first introduced in Topic 1 and revisited in Subtopic 5 of this topic: NOT IN's mechanism is fundamentally value-comparison-based (and therefore vulnerable to a single NULL poisoning the whole chain), while NOT EXISTS's mechanism is fundamentally row-existence-based and doesn't perform that same kind of per-value equality chain — so a NULL in the correlated subquery's filtering column doesn't create the same all-or-nothing failure mode.

**Why the Other Options Are Wrong:**
A. Performance is a separate (and data/plan-dependent) concern, not the actual reason for the NULL-safety difference.
C. NOT EXISTS doesn't "filter out" NULLs as a preprocessing step; it simply never performs the kind of comparison that would be vulnerable to them in the first place.
D. This is a well-documented, real, and frequently-tested distinction — not a myth.

**Concept Tested:** Why NOT EXISTS avoids the NOT IN/NULL poisoning trap — connecting three-valued logic (Topic 1) to a practical rewriting technique.

**Exam Trap:** Not understanding the *mechanism* behind the safety difference, and therefore being unable to recognize or explain it when a rewritten query is presented on the exam.

---

**Question 3**
Rewrite this NOT-IN query, which is vulnerable to the NULL trap, using NOT EXISTS instead:
`SELECT * FROM departments d WHERE d.department_id NOT IN (SELECT department_id FROM employees WHERE department_id IS NOT NULL);` (assume this particular subquery is NULL-safe already via the added filter) — but suppose that filter were accidentally omitted. Which NOT EXISTS rewrite is safe regardless of NULLs in employees.department_id?

A. `SELECT * FROM departments d WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.department_id);`
B. `SELECT * FROM departments d WHERE d.department_id NOT EXISTS (SELECT department_id FROM employees);`
C. `SELECT * FROM departments d WHERE NOT EXISTS (SELECT department_id FROM employees WHERE department_id <> d.department_id);`
D. Both A and C are equally safe and correct.

**Correct Answer:** A

**Detailed Explanation:**
This is the canonical, NULL-immune "departments with no employees" pattern: a correlated NOT EXISTS checks, for each department, whether *any* employee row matches that department_id — regardless of whether other, unrelated employee rows happen to have a NULL department_id, since those NULL rows simply fail the correlation condition (`e.department_id = d.department_id`) silently and don't participate in an ANDed equality chain the way NOT IN's mechanism does.

**Why the Other Options Are Wrong:**
B. Invalid syntax — EXISTS/NOT EXISTS take a subquery as their sole operand; they aren't comparison operators placed after a column.
C. Logically wrong — this checks for the existence of *any* employee whose department differs from d's, which would be true for almost every department (satisfied by nearly any other department's employees), not what's intended.
D. C is incorrect, so they aren't equally valid.

**Concept Tested:** Correctly constructing the NULL-safe correlated NOT EXISTS rewrite of a NOT IN query.

**Exam Trap:** Getting the correlation condition subtly wrong (as in C, using `<>` instead of `=`) while still superficially "using NOT EXISTS," and assuming that alone guarantees correctness.

---

**Question 4**
Must an EXISTS subquery always be correlated?

A. Yes — EXISTS is only ever meaningful when tied to outer-query values.
B. No — while EXISTS is *most commonly* paired with a correlated condition, a non-correlated EXISTS is also legal (if less commonly useful) — e.g., simply checking whether a table has any rows at all, applied uniformly regardless of the outer row.
C. No, but a non-correlated EXISTS always evaluates to TRUE, making it pointless.
D. Yes, and Oracle raises an error if you attempt a non-correlated EXISTS.

**Correct Answer:** B

**Detailed Explanation:**
EXISTS and correlation are, once again, independent properties (mirroring the scalar/correlated independence from Subtopic 3). A non-correlated `EXISTS (SELECT 1 FROM some_table)` is syntactically and semantically valid — it just answers the same yes/no question ("does this table have any rows matching this fixed condition?") for every outer row uniformly, which is admittedly a less common, less useful pattern than the correlated form.

**Why the Other Options Are Wrong:**
A, D. Both wrongly claim correlation is mandatory for EXISTS.
C. A non-correlated EXISTS isn't guaranteed to always be TRUE — it depends entirely on whether the fixed (non-correlated) subquery condition actually matches any rows; it's a legitimate, evaluable condition, just not one that varies per outer row.

**Concept Tested:** EXISTS does not structurally require correlation, even though correlation is by far its most common and useful application.

**Exam Trap:** Assuming EXISTS and "correlated" are effectively synonymous, when in fact EXISTS is just another subquery-consuming construct that happens to pair unusually well with correlation.

---

**Question 5**
Which query correctly finds all departments that currently have **no employees**, using a correlated NOT EXISTS?

A. `SELECT department_name FROM departments d WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.department_id);`
B. `SELECT department_name FROM departments d WHERE EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.department_id);`
C. `SELECT department_name FROM departments WHERE department_id NOT IN (SELECT department_id FROM employees);` — always equally safe as A, regardless of NULLs.
D. `SELECT department_name FROM departments d, employees e WHERE d.department_id != e.department_id;`

**Correct Answer:** A

**Detailed Explanation:**
This is precisely the pattern established in Question 3: for each department, check whether *no* employee row correlates to it — meaning no employee references that department_id at all, correctly identifying empty departments.

**Why the Other Options Are Wrong:**
B. Uses EXISTS (not NOT EXISTS) — this instead finds departments that DO have at least one employee, the opposite of what's asked.
C. Directly contradicted by Subtopic 5, Question 1 — this NOT IN version is *not* equally safe; if EMPLOYEES.department_id contains even one NULL, this query silently returns **zero rows**, entirely wrong.
D. A Cartesian-style `!=` join doesn't correctly express "no employees in this department" — it would instead return department/employee pairs from *different* departments, an entirely different and much larger, largely meaningless result.

**Concept Tested:** The full correct, NULL-safe correlated NOT EXISTS pattern for finding "no matching child rows" cases.

**Exam Trap:** Mistaking EXISTS for NOT EXISTS (option B) — an easy, careless-looking mistake that completely inverts the query's meaning — and underestimating how fragile the superficially-similar NOT IN version (option C) really is.

---

### Subtopic 8: Correlated vs. Non-Correlated Subqueries — Summary Differences

**Question 1**
Which of the following is the clearest, most fundamental distinguishing test for whether a subquery is correlated?

A. Whether the subquery uses EXISTS.
B. Whether the subquery, if copy-pasted and run entirely on its own (outside the outer query), would execute successfully and produce a meaningful, complete result — a correlated subquery cannot do this, since it references an outer-query column/alias that doesn't exist in a standalone execution context; a non-correlated subquery runs standalone without issue.
C. Whether the subquery returns more than one row.
D. Whether the subquery appears in the WHERE clause versus the SELECT list.

**Correct Answer:** B

**Detailed Explanation:**
This is a genuinely useful practical diagnostic: try running just the subquery by itself. If it references an alias/column from the outer query that Oracle doesn't recognize in isolation (raising an "invalid identifier" error), the subquery is correlated. If it runs cleanly on its own, it's non-correlated.

**Why the Other Options Are Wrong:**
A. EXISTS usage is a common *association*, not a defining test (Subtopic 7, Question 4).
C. Row count (scalar/single-row/multi-row) is an orthogonal classification, unrelated to correlation.
D. Clause placement doesn't determine correlation — both WHERE and SELECT-list subqueries can be either correlated or non-correlated.

**Concept Tested:** A practical, mechanism-based test for identifying correlation ("can this subquery run standalone?").

**Exam Trap:** Reaching for surface-level heuristics (keyword or clause placement) instead of the underlying, always-reliable test of standalone executability.

---

**Question 2**
Which two of the following statements correctly summarize a core difference between correlated and non-correlated subqueries? **(Choose two)**

A. A non-correlated subquery's result can be computed once and conceptually reused across the entire outer query; a correlated subquery's result conceptually depends on, and must be re-derived for, each individual outer row.
B. Only correlated subqueries can use aggregate functions inside them.
C. A non-correlated subquery has no reference to any outer-query column or alias; a correlated subquery does.
D. Correlated subqueries can only be used in DELETE statements.

**Correct Answers:** A and C

**Detailed Explanation:**
These two statements capture the essential distinction from opposite angles: A describes the *execution-model* consequence (once vs. per-row), while C describes the *syntactic root cause* (outer-column reference) that produces that consequence.

**Why the Other Options Are Wrong:**
B. Aggregate functions are equally usable in correlated and non-correlated subqueries — there's no such restriction.
D. Correlated subqueries are fully usable across SELECT, INSERT, UPDATE, and DELETE alike (Subtopic 2) — nothing restricts them to DELETE.

**Concept Tested:** The two complementary ways of describing the correlated/non-correlated distinction — cause (outer-column reference) and effect (per-row vs. once-only evaluation).

**Exam Trap:** Only remembering one half of the distinction (e.g., just the "references outer column" part) without connecting it to the resulting execution-model implication, or vice versa.

---

**Question 3**
Can many correlated EXISTS-style subqueries be rewritten as equivalent joins?

A. Never — correlated subqueries and joins are fundamentally incompatible concepts.
B. Often, yes — many correlated EXISTS patterns (like "departments with at least one employee") have join-based equivalents, though care must be taken regarding potential duplicate row multiplication a join might introduce that the original EXISTS-based query wouldn't have (since EXISTS only tests presence, never multiplying rows for multiple matches).
C. Yes, and the join-based version is always guaranteed to return the exact same result set with zero risk of any difference.
D. Only DELETE statements can rewrite correlated subqueries as joins; SELECT cannot.

**Correct Answer:** B

**Detailed Explanation:**
This is a nuanced but important practical point: `SELECT d.* FROM departments d WHERE EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.department_id)` can often be rewritten as `SELECT DISTINCT d.* FROM departments d JOIN employees e ON e.department_id = d.department_id` — but note the necessary DISTINCT, since a plain join would return one row per *matching employee*, not one row per qualifying department, whereas EXISTS inherently returns each department at most once.

**Why the Other Options Are Wrong:**
A. Directly contradicted by the well-known rewriting technique described above.
C. Overstates the guarantee — the duplicate-row risk from joins (absent DISTINCT or similar handling) is real and must be actively managed, not automatically avoided.
D. Both SELECT and DELETE (and other DML) can potentially benefit from such rewrites where applicable; there's no such statement-type restriction.

**Concept Tested:** The practical relationship (and important caveat about row multiplication) between correlated EXISTS subqueries and equivalent join rewrites.

**Exam Trap:** Assuming a join rewrite of an EXISTS subquery is a "free," risk-free transformation, without accounting for the row-multiplication difference between "does a match exist" (EXISTS) and "return one row per match" (a plain join).

---

**Question 4**
True or False: In Oracle, a subquery placed directly in the FROM clause (an inline view) can freely reference columns from other tables listed elsewhere in that same FROM clause, making it automatically correlated, with no special syntax required.

A. True
B. False

**Correct Answer:** B (False)

**Detailed Explanation:**
This is an important, more advanced nuance: an ordinary FROM-clause subquery in Oracle is evaluated independently and, by default, **cannot** reference other tables in the same FROM list — attempting to do so raises an error. To achieve genuine correlation for a FROM-clause subquery (a "lateral" reference), Oracle requires the explicit `LATERAL` keyword, introduced specifically to permit this behavior when needed.

**Concept Tested:** FROM-clause subqueries are non-correlated by default; genuine correlation there requires the explicit LATERAL keyword.

**Exam Trap:** Assuming that because WHERE-clause and SELECT-list subqueries can be freely correlated with no special syntax, the same must automatically be true for FROM-clause subqueries — Oracle specifically restricts this case unless LATERAL is used.

---

**Question 5**
Identify whether the following subquery is correlated or non-correlated:
`SELECT last_name FROM employees WHERE department_id = (SELECT department_id FROM departments WHERE department_name = 'Finance');`

A. Correlated — it references the DEPARTMENTS table.
B. Non-correlated — the subquery's WHERE condition (`department_name = 'Finance'`) depends only on values from within the subquery's own table (DEPARTMENTS); it never references any column from the outer query (EMPLOYEES), so it computes exactly once and its single resulting department_id is then reused for the outer comparison.
C. Cannot be determined without knowing the actual data in the tables.
D. Correlated, because the outer query's WHERE clause uses the subquery's result.

**Correct Answer:** B

**Detailed Explanation:**
Referencing a *different* table isn't what makes a subquery correlated — the test (per Question 1 of this subtopic) is whether it references an outer-query *column/alias*. Here, the subquery is entirely self-contained (querying DEPARTMENTS using only DEPARTMENTS' own columns) and would run perfectly well standalone, producing 'Finance's department_id on its own.

**Why the Other Options Are Wrong:**
A. Referencing a different table is completely normal for non-correlated subqueries too — that alone says nothing about correlation.
C. Correlation is a structural/syntactic property, determinable directly from the query text — no data inspection is needed.
D. Being *used by* the outer query's WHERE clause (i.e., its result feeds into an outer comparison) is not the same as being correlated *to* the outer query — every subquery's result is "used" by whatever contains it; correlation specifically requires the reverse direction (the subquery reaching out to reference the outer row).

**Concept Tested:** Correctly applying the standalone-executability test to distinguish "references a different table" from "is correlated to the outer query."

**Exam Trap:** Conflating "subquery involves another table" or "subquery's result feeds the outer query" with the actual, narrower definition of correlation (the subquery reaching back into the outer row's own values).