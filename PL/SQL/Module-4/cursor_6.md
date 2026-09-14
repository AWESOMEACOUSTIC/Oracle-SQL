# Topic 6: REF Cursor

> **Syllabus position:** Topic 6 of 6 (+ final combined practice set)
> **Builds on:** Topics 1–5 (lifecycle, parameters, FOR loops, FOR UPDATE/WHERE CURRENT OF)
> **This is the last new-concept topic** — Topic 7 combines everything from Topics 1–6 into realistic, mixed case-study problems.

---

## 1. Concept

A **REF CURSOR** (also called a **cursor variable**) is a PL/SQL data type whose value is a pointer to a query's result set — but unlike everything in Topics 1–5, its query is **not fixed at declaration time**. You declare a REF CURSOR *variable*, and only decide what query it points to when you `OPEN ... FOR <query>`. Because it's a genuine variable (not a compile-time-bound named object like a static cursor), it can be **passed as a parameter** into and out of procedures and functions — something no cursor from Topics 1–5 can do.

```sql
DECLARE
   v_cursor SYS_REFCURSOR;
BEGIN
   OPEN v_cursor FOR SELECT first_name FROM employees WHERE department_id = 30;
   -- ... fetch manually, close ...
END;
/
```

## 2. Purpose / Why It Exists

Every cursor type covered so far — with or without parameters, consumed manually or via a FOR loop — has its query shape **fixed at compile time** and is **scoped to the block or package it's declared in**. That creates three real limitations REF CURSOR was specifically built to remove:

1. **You can't hand a result set to another subprogram.** A helper procedure can't `OPEN` a cursor and pass it to a different procedure to `FETCH` from — with a static cursor, there's nothing to "pass"; it isn't a variable.
2. **You can't return a result set to an external client application.** A Java, .NET, or reporting-tool caller invoking an Oracle stored procedure needs *some* way to receive "a table of rows" back — PL/SQL functions/procedures don't otherwise have a return mechanism client drivers can consume as a native result set.
3. **You can't vary the query itself at runtime.** A static cursor's SELECT text is fixed when the PL/SQL unit is compiled; if the *shape* of the query genuinely needs to change based on runtime conditions (e.g., an optional multi-filter search screen), a fixed cursor can't express that.

REF CURSOR solves all three: it's a real variable (so it can be a parameter or a function's return value), and its query is supplied at `OPEN` time (so the query itself can vary, including being built dynamically as a string).

## 3. What Real-World Problem It Solves

The single most common real-world use is exactly problem #2 above: a stored procedure that a Java/.NET/BI/reporting-tool front end calls to retrieve data, returning the result set through an `OUT` parameter that the client driver consumes natively. Beyond that: building reusable "generic fetch" utility procedures, and handling optional-filter search screens where the actual WHERE clause needed depends on which filters a user chose to fill in.

## 4. When to Use / When NOT to Use

**Use a REF CURSOR when:**
- A result set needs to be returned from a procedure/function to a caller — especially an external client application.
- A cursor needs to be passed between subprograms.
- The query's shape genuinely needs to vary at runtime (dynamic SQL).

**Use a plain static cursor (Topics 1–5) instead when:**
- The query is fixed, known at compile time, and never needs to leave the block/subprogram where it's declared.
- You want to use a cursor FOR loop for convenience — **REF CURSOR variables cannot be used directly with `FOR rec IN cursor_variable LOOP`** (more on this below); if you don't need any of REF CURSOR's specific benefits, a static cursor stays simpler and requires less code.

**Recognition clue:** "return the results to the calling application/report," "the query should be built based on which filters the user provides," "a reusable procedure other parts of the system can call for different queries," or any mention of an external client (Java, .NET, BI tool) consuming query results — all point straight at REF CURSOR.

## 5. Syntax

### Declaring
```sql
-- Strongly typed (restricted): return shape is fixed and checked
TYPE emp_cursor_type IS REF CURSOR RETURN employees%ROWTYPE;
v_emp_cursor emp_cursor_type;

-- Weakly typed (unrestricted): any query shape allowed
TYPE generic_cursor_type IS REF CURSOR;
v_generic_cursor generic_cursor_type;

-- Built-in weak type — no custom TYPE declaration needed at all
v_cursor SYS_REFCURSOR;
```

### Opening
```sql
OPEN v_emp_cursor FOR SELECT * FROM employees WHERE department_id = 30;

-- Dynamic SQL variant — query text built as a string, values supplied via bind variables
OPEN v_cursor FOR 'SELECT * FROM employees WHERE department_id = :1' USING p_dept_id;
```

### Fetching and closing (identical mechanics to Topic 1 — always manual, never a FOR loop)
```sql
LOOP
   FETCH v_emp_cursor INTO v_emp_rec;
   EXIT WHEN v_emp_cursor%NOTFOUND;
   -- process v_emp_rec
END LOOP;
CLOSE v_emp_cursor;
```

### The defining pattern — passing a REF CURSOR as an OUT parameter
```sql
CREATE OR REPLACE PROCEDURE get_employees_by_dept (
   p_dept_id IN  employees.department_id%TYPE,
   p_result  OUT SYS_REFCURSOR
) IS
BEGIN
   OPEN p_result FOR
      SELECT employee_id, first_name, salary
      FROM employees
      WHERE department_id = p_dept_id;
END;
/
```
Called from another PL/SQL block:
```sql
DECLARE
   v_cursor SYS_REFCURSOR;
   v_id     employees.employee_id%TYPE;
   v_name   employees.first_name%TYPE;
   v_sal    employees.salary%TYPE;
BEGIN
   get_employees_by_dept(30, v_cursor);
   LOOP
      FETCH v_cursor INTO v_id, v_name, v_sal;
      EXIT WHEN v_cursor%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_name || ' - ' || v_sal);
   END LOOP;
   CLOSE v_cursor;
END;
/
```
An external client (Java/.NET/reporting tool) calling `get_employees_by_dept` would bind its own result-set object to `p_result` and fetch rows using its own driver's native mechanism — it wouldn't write a manual PL/SQL `FETCH` loop itself; that loop above is specifically what a *PL/SQL* caller would do.

### Syntax Breakdown

- `REF CURSOR RETURN return_type` — **strong typing**: `return_type` must be a record type or `%ROWTYPE`; every query later opened against a variable of this type must structurally match (same number of columns, compatible datatypes, in order).
- `REF CURSOR` with no `RETURN` — **weak typing**: any query shape is allowed.
- `SYS_REFCURSOR` — Oracle's predefined weak type; needs no custom `TYPE` declaration, which is exactly why it's the standard, default choice in real code — any caller, PL/SQL or external, can use it without needing access to a custom type defined in your package.
- `OPEN cursor_variable FOR select_statement;` — note the difference from static cursors: there's no separately declared query to "open" — the SELECT is supplied right at the `OPEN` statement. This also means the **same variable** can be opened for a **different query** on separate occasions (freely, if weakly typed; only if shape-compatible, if strongly typed).
- `OPEN ... FOR 'sql_string' USING bind_values;` — the native dynamic SQL form; lets the query text itself be built at runtime, with bind variables supplied via `USING`.
- Fetching/closing uses the **exact same** `FETCH`/`%NOTFOUND`/`%ROWCOUNT`/`%ISOPEN`/`CLOSE` mechanics as any explicit cursor from Topic 1 — nothing new to learn there.
- REF CURSOR variables can be passed with **any** parameter mode — `IN`, `OUT`, or `IN OUT` — unlike static cursor parameters (Topic 2), which are always IN-only and belong to one fixed query shape.

## 6. Types / Variations

| Variation | Description |
|---|---|
| **Strongly typed (restricted)** | `TYPE t IS REF CURSOR RETURN record_type;` — structurally checked against the declared return shape. |
| **Weakly typed (unrestricted)** | `TYPE t IS REF CURSOR;` — any query shape allowed, custom type name. |
| **`SYS_REFCURSOR`** | Built-in weak type — the standard, most commonly used form in practice. |
| **Opened against a static query** | Query text is fixed in the source code, but the cursor itself can still be passed/returned as a variable. |
| **Opened against dynamic SQL** | Query text is built as a string at runtime — full flexibility of both passing the cursor *and* varying its shape. |
| **As an OUT parameter** | The defining, most common real-world pattern — returning a result set from a procedure to its caller. |
| **As a function return type** | `RETURN SYS_REFCURSOR;` — a functionally similar alternative to the OUT-parameter style. |

## 7. Simple Examples

### Example A — Strong REF CURSOR
```sql
DECLARE
   TYPE emp_cursor_type IS REF CURSOR RETURN employees%ROWTYPE;
   v_emp_cursor emp_cursor_type;
   v_emp_rec    employees%ROWTYPE;
BEGIN
   OPEN v_emp_cursor FOR SELECT * FROM employees WHERE department_id = 30;
   LOOP
      FETCH v_emp_cursor INTO v_emp_rec;
      EXIT WHEN v_emp_cursor%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_emp_rec.first_name);
   END LOOP;
   CLOSE v_emp_cursor;
END;
/
```

### Example B — `SYS_REFCURSOR`, same variable opened for two entirely different queries
```sql
DECLARE
   v_cursor SYS_REFCURSOR;
   v_text   VARCHAR2(100);
BEGIN
   OPEN v_cursor FOR SELECT first_name FROM employees WHERE department_id = 30;
   LOOP
      FETCH v_cursor INTO v_text;
      EXIT WHEN v_cursor%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE('Employee: ' || v_text);
   END LOOP;
   CLOSE v_cursor;

   OPEN v_cursor FOR SELECT department_name FROM departments WHERE department_id = 30;  -- totally different shape
   LOOP
      FETCH v_cursor INTO v_text;
      EXIT WHEN v_cursor%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE('Department: ' || v_text);
   END LOOP;
   CLOSE v_cursor;
END;
/
```
This is the clearest demonstration of the structural difference from every earlier topic: the same variable holds a completely different query shape on its second `OPEN`, which is only possible because it's weakly typed.

### Example C — Dynamic SQL for optional search filters (naive version — see caution below)
```sql
CREATE OR REPLACE PROCEDURE search_employees (
   p_dept_id IN employees.department_id%TYPE DEFAULT NULL,
   p_min_sal IN employees.salary%TYPE DEFAULT NULL,
   p_result  OUT SYS_REFCURSOR
) IS
   v_sql VARCHAR2(500);
BEGIN
   v_sql := 'SELECT employee_id, first_name, salary FROM employees WHERE 1=1';
   IF p_dept_id IS NOT NULL THEN
      v_sql := v_sql || ' AND department_id = ' || p_dept_id;
   END IF;
   IF p_min_sal IS NOT NULL THEN
      v_sql := v_sql || ' AND salary >= ' || p_min_sal;
   END IF;
   OPEN p_result FOR v_sql;
END;
/
```
**Important caution:** concatenating values directly into a dynamic SQL string like this is a real SQL-injection and shared-pool efficiency concern (detailed below). Example D shows the safer, bind-variable version — treat Example C as illustrating the *problem*, not the recommended pattern.

### Example D — The safer version, using bind variables
```sql
CREATE OR REPLACE PROCEDURE search_employees (
   p_dept_id IN employees.department_id%TYPE DEFAULT NULL,
   p_min_sal IN employees.salary%TYPE DEFAULT NULL,
   p_result  OUT SYS_REFCURSOR
) IS
   v_sql VARCHAR2(500);
BEGIN
   v_sql := 'SELECT employee_id, first_name, salary FROM employees
             WHERE (department_id = :dept_id OR :dept_id IS NULL)
               AND (salary >= :min_sal OR :min_sal IS NULL)';

   OPEN p_result FOR v_sql USING p_dept_id, p_dept_id, p_min_sal, p_min_sal;
END;
/
```
Here, the SQL **text** is identical no matter which filters the caller actually supplies — only the bound values change — which matters for both security and performance (see Detailed Explanation).

## 8. Detailed Explanation

- **Strong typing checks structure, not meaning.** A strongly typed REF CURSOR verifies that whatever query you `OPEN` it for returns a result set matching its declared `RETURN` type in column count and compatible datatypes. It does **not** verify that the query is *semantically* the right query — you could open it against a completely different, unrelated table that happens to have the same column shape, and Oracle won't stop you. Type safety here has real limits worth knowing, especially for interview-level discussion.
- **The defining capability is crossing subprogram boundaries.** Every cursor in Topics 1–5 is a compile-time-bound object scoped to the block or package where it's declared — you cannot `OPEN` one in Procedure A and `FETCH` from it inside Procedure B by passing it as an argument. A REF CURSOR variable, being a genuine variable of a specific type, can be passed, assigned, and returned exactly like any other PL/SQL variable.
- **No FOR loop shortcut.** Recall the Topic 4 preview: the cursor FOR loop syntax `FOR record_name IN {cursor_name | (SELECT ...)}` only accepts a **static** cursor's name or an inline `SELECT` — never a cursor *variable*. Consuming a REF CURSOR therefore always requires the manual `LOOP ... FETCH ... EXIT WHEN ... CLOSE` pattern from Topic 1. This is a real, small trade-off you accept in exchange for REF CURSOR's flexibility.
- **Always prefer bind variables over string concatenation in dynamic SQL.** Concatenating raw values into a dynamic SQL string (Example C) has two real problems: it's a SQL-injection risk whenever any part of the concatenated value could be influenced by external/user input, and it hurts performance at scale, because the exact SQL *text* differs slightly for every distinct value concatenated in, preventing Oracle from reusing a single cached, parsed version of the statement from the shared pool. Using `USING` (Example D) keeps the SQL text identical across calls, differing only in bound values, letting Oracle reuse the parsed statement.
- **`FOR UPDATE`/`WHERE CURRENT OF` (Topic 5) work identically** with a REF CURSOR, as long as the query opened against it includes `FOR UPDATE` — nothing about locking changes based on whether the cursor is static or a REF CURSOR.
- **A REF CURSOR that goes out of scope without an explicit `CLOSE`** is automatically closed by PL/SQL when its enclosing block ends — similar in spirit to any local variable ceasing to exist. Relying on this isn't good practice, though, especially in long-running or loop-heavy code; explicit `CLOSE` remains the right habit.
- **REF CURSOR vs. returning a PL/SQL collection:** both are ways to "return multiple rows" from a function, but REF CURSOR is the standard choice specifically when the consumer might be an **external client application** — virtually every client driver (Java's JDBC, .NET's ODP.NET, reporting/BI tools) understands how to consume an `OUT` REF CURSOR natively as a result set. PL/SQL collections, by contrast, require a PL/SQL-aware consumer and are typically used when the caller is itself PL/SQL. (Collections aren't part of this syllabus — mentioned only as context for *why* REF CURSOR is the standard answer to "return rows to an external caller.")

## 9. Common Mistakes and Misconceptions

1. **Trying to use a cursor FOR loop directly on a REF CURSOR variable** (`FOR rec IN v_cursor LOOP`) — not supported; use the manual fetch loop.
2. **Concatenating values into dynamic SQL instead of using bind variables** — SQL-injection risk and worse shared-pool reuse.
3. **Assuming a weak REF CURSOR (or `SYS_REFCURSOR`) performs any structural checking** — it performs none; a mismatch between the query's actual columns and what you `FETCH INTO` only surfaces as a runtime error.
4. **Forgetting `CLOSE` is still required** — REF CURSOR's flexibility doesn't remove lifecycle responsibility; whichever block finishes using the data is generally responsible for closing it (unless it's being passed further along to another layer that will close it instead).
5. **Assuming strong typing catches everything** — it only checks structural shape, not whether the query is the *semantically correct* one.
6. **Mixing up parameter modes** — a procedure that needs to `OPEN` a cursor itself and hand results back to its caller needs an `OUT` (or `IN OUT`) parameter, not `IN`. This is a common early stumbling block when first writing this pattern.

## 10. Edge Cases

- Opening a REF CURSOR variable that's already open, without closing first → the same `ORA-06511: cursor already open` error as static cursors — this is the same underlying resource management under the hood.
- Passing a REF CURSOR `OUT` parameter through multiple layers of procedures (A calls B, which opens and returns it; A passes it straight through, untouched, to its own caller) — perfectly valid; only whichever layer actually needs the data must fetch from it.
- A strongly typed REF CURSOR opened against a query that's shape-compatible but pulls from an entirely different, unrelated table — still works, since the check is structural only (see Detailed Explanation).
- Zero-row queries, `NOWAIT`/`WAIT`/`SKIP LOCKED`, `FOR UPDATE`/`WHERE CURRENT OF` — all behave identically to static cursors; nothing about being a REF CURSOR changes these mechanics.

## 11. How This Relates to Other Topics

- Topic 1's lifecycle mechanics (`FETCH`, attributes, `CLOSE`) apply completely unchanged.
- Topic 2's idea that "parameters make a cursor reusable for different *values*" is generalized further here: a REF CURSOR makes a cursor reusable for entirely different *queries*, not just different filter values on one fixed shape.
- Topic 4's cursor FOR loop syntax explicitly does **not** extend to REF CURSOR variables — an important negative fact to hold onto.
- Topic 5's `FOR UPDATE`/`WHERE CURRENT OF` apply identically once a REF CURSOR is opened against a query that includes `FOR UPDATE`.
- Topic 3's decision framework gains one more question specific to this topic: *"Does this result set need to leave this subprogram (returned to a caller, or passed as a parameter), or does the query's shape need to vary at runtime? If yes → REF CURSOR. If no → a static cursor (Topics 1–3) is simpler and sufficient."*

---

## Things You Must Remember

- Strong REF CURSOR: fixed, structurally-checked return type. Weak REF CURSOR / `SYS_REFCURSOR`: no restriction — the standard real-world choice.
- The query is supplied at `OPEN ... FOR ...`, never at declaration — this is the core syntactic difference from every earlier topic.
- REF CURSOR variables can be passed as `IN`/`OUT`/`IN OUT` parameters and returned from functions — this is the single defining reason to reach for one.
- Cannot be used directly with `FOR rec IN cursor_variable LOOP` — always a manual fetch loop.
- Always prefer bind variables (`USING`) over string concatenation when opening against dynamic SQL.
- Still must be explicitly `CLOSE`d — REF CURSOR changes how the query attaches to the cursor, not whether lifecycle discipline still applies.
- The standard, most common real-world pattern: a procedure/function with a `SYS_REFCURSOR` `OUT` parameter, handing a result set back to an external client application or another PL/SQL caller.

## How to Recognize This Concept

Think **REF CURSOR** when a requirement says things like:
- "return the results to the calling application / front end / report"
- "the query should be built dynamically based on which filters the user provides"
- "a reusable procedure that different parts of the system can call to get different result sets"
- "pass the result set from this procedure to that one"
- Any mention of an external client (Java, .NET, reporting tool, BI tool) needing to consume query results from a stored procedure.

---

# Exercises — With Answers and Reasoning

## Easy (Syntax & Basics)

### Exercise 1
**Task:** Declare a `SYS_REFCURSOR`, open it for a `SELECT` on a `customers` table filtered by a given city, and fetch/print results manually (no FOR loop).

**Solution:**
```sql
DECLARE
   v_cursor      SYS_REFCURSOR;
   v_customer_nm customers.customer_name%TYPE;
BEGIN
   OPEN v_cursor FOR SELECT customer_name FROM customers WHERE city = 'Chennai';
   LOOP
      FETCH v_cursor INTO v_customer_nm;
      EXIT WHEN v_cursor%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_customer_nm);
   END LOOP;
   CLOSE v_cursor;
END;
/
```

**Reasoning:** No `CURSOR ... IS` declaration exists here at all — `v_cursor` is a plain variable of type `SYS_REFCURSOR`, and the query is attached only at the `OPEN ... FOR` statement. Fetching, checking `%NOTFOUND`, and closing follow the exact same mechanics as Topic 1's manual pattern — REF CURSOR changes *how the query gets attached*, not how you consume it once it's open.

---

### Exercise 2
**Task:** Write a strongly typed REF CURSOR (`RETURN employees%ROWTYPE`), demonstrate a valid `OPEN`, and explain why opening it against a structurally incompatible query would fail.

**Solution (valid use):**
```sql
DECLARE
   TYPE emp_cursor_type IS REF CURSOR RETURN employees%ROWTYPE;
   v_emp_cursor emp_cursor_type;
   v_emp_rec    employees%ROWTYPE;
BEGIN
   OPEN v_emp_cursor FOR SELECT * FROM employees WHERE department_id = 30;
   -- ... fetch, close ...
   CLOSE v_emp_cursor;
END;
/
```

**Reasoning — why an incompatible query would fail:** `emp_cursor_type` is declared to `RETURN employees%ROWTYPE`, meaning every query opened against a variable of this type must return a result set with the same number of columns, in the same order, with compatible datatypes, as the full `employees` table structure. If you tried, for example, `OPEN v_emp_cursor FOR SELECT department_id, department_name FROM departments;` — a two-column result that doesn't match the many-column `employees%ROWTYPE` shape — Oracle would reject this, because strong typing exists precisely to catch this kind of shape mismatch before you get to the point of trying to `FETCH` into a mismatched record and encountering a confusing runtime error instead.

---

### Exercise 3
**Task:** Write a procedure `get_department_employees` taking a department ID as an `IN` parameter and returning a `SYS_REFCURSOR` `OUT` parameter with employee details. Then write a calling block that fetches and prints from it.

**Solution:**
```sql
CREATE OR REPLACE PROCEDURE get_department_employees (
   p_dept_id IN  employees.department_id%TYPE,
   p_result  OUT SYS_REFCURSOR
) IS
BEGIN
   OPEN p_result FOR
      SELECT first_name, salary FROM employees WHERE department_id = p_dept_id;
END;
/

DECLARE
   v_cursor SYS_REFCURSOR;
   v_name   employees.first_name%TYPE;
   v_sal    employees.salary%TYPE;
BEGIN
   get_department_employees(30, v_cursor);
   LOOP
      FETCH v_cursor INTO v_name, v_sal;
      EXIT WHEN v_cursor%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_name || ' - ' || v_sal);
   END LOOP;
   CLOSE v_cursor;
END;
/
```

**Reasoning:** This is the defining REF CURSOR pattern — the procedure `OPEN`s the cursor against its own internal query and hands it back through `p_result`; the caller never sees the SQL text at all, only receives an already-open cursor ready to `FETCH` from. Note that the caller, not the procedure, is responsible for `CLOSE`-ing it — whichever side actually finishes consuming the data owns the closing responsibility.

---

### Exercise 4
**Task:** Modify Exercise 3 so that passing `NULL` for the department ID returns **all** employees instead of none.

**Solution:**
```sql
CREATE OR REPLACE PROCEDURE get_department_employees (
   p_dept_id IN  employees.department_id%TYPE DEFAULT NULL,
   p_result  OUT SYS_REFCURSOR
) IS
BEGIN
   OPEN p_result FOR
      SELECT first_name, salary FROM employees
      WHERE department_id = p_dept_id OR p_dept_id IS NULL;
END;
/
```

**Reasoning:** A plain `WHERE department_id = p_dept_id` would match **zero rows** when `p_dept_id` is `NULL`, because SQL's three-valued logic means `column = NULL` is never `TRUE` — a direct callback to the same NULL-comparison trap covered in Topic 2. Adding `OR p_dept_id IS NULL` explicitly handles the "no filter chosen" case, making `NULL` mean "match everything" rather than "match nothing." This is the same pattern used more extensively in Exercise 7's multi-filter scenario below.

---

## Intermediate (Reasoning & Edge Cases)

### Exercise 5
**Task:** A developer writes `FOR emp_rec IN v_ref_cursor LOOP ... END LOOP;` where `v_ref_cursor` is a `SYS_REFCURSOR` already opened for a query. What happens, and how should it be fixed?

**Answer:** This **fails to compile**. The cursor FOR loop syntax only accepts a **static, named cursor** (declared with `CURSOR name IS ...`) or an **inline `SELECT`** written directly in the loop header — it does not accept a cursor *variable*, regardless of whether that variable happens to already be open. This is exactly the negative fact flagged back in Topic 4's preview and restated in this topic's Detailed Explanation: REF CURSOR variables are always consumed manually.

**Fix:**
```sql
DECLARE
   v_ref_cursor SYS_REFCURSOR;
   v_name       employees.first_name%TYPE;
BEGIN
   OPEN v_ref_cursor FOR SELECT first_name FROM employees WHERE department_id = 30;
   LOOP
      FETCH v_ref_cursor INTO v_name;
      EXIT WHEN v_ref_cursor%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_name);
   END LOOP;
   CLOSE v_ref_cursor;
END;
/
```

---

### Exercise 6
**Task — spot the risk:** A procedure builds a dynamic SQL string by concatenating a user-supplied `p_last_name` parameter directly into the WHERE clause:
```sql
v_sql := 'SELECT * FROM employees WHERE last_name = ''' || p_last_name || '''';
OPEN p_result FOR v_sql;
```
What's wrong, and how should it be rewritten safely?

**Answer:** This concatenates a value that may originate from user input **directly into the SQL text**, which is a classic SQL-injection vulnerability. If `p_last_name` were something like `X' OR '1'='1`, the resulting SQL text would become `... WHERE last_name = 'X' OR '1'='1'` — a condition that's always true, returning every row in the table regardless of the intended filter, and in more elaborate injection attempts, potentially allowing far more damaging manipulation of the query. Beyond the security issue, this approach also hurts performance: since the literal value is baked directly into the SQL text, the text differs on every distinct call, preventing Oracle from reusing a single cached, parsed version of the statement from the shared pool.

**Fix — use a bind variable:**
```sql
v_sql := 'SELECT * FROM employees WHERE last_name = :lname';
OPEN p_result FOR v_sql USING p_last_name;
```
Now the SQL text is always identical regardless of what value is searched for; only the *bound value* changes per call, eliminating the injection risk entirely (the value is never interpreted as part of the SQL syntax) and allowing the parsed statement to be reused.

---

## Realistic Scenario

### Exercise 7
**Business Requirement:** *"The company's new web reporting dashboard needs a single stored procedure that fulfills employee search requests: users may optionally filter by department, minimum salary, and hire date range, in any combination — all fields are optional, and the dashboard passes NULL for anything not chosen. The procedure must return the matching employee records back to the web application's reporting layer, which knows how to consume a standard result set."*

**What Is Being Asked:**
A single procedure, callable with any combination of four optional filters (department, minimum salary, hire-date-from, hire-date-to), that returns a result set an external client can consume directly.

**Key Clues:**
- "return the matching employee records back to the web application's reporting layer" → direct REF CURSOR signal — an external client needs to consume the result set.
- "optionally filter... in any combination... passes NULL for anything not chosen" → the NULL-means-"no filter" pattern from Exercise 4, but now across **four** independent optional filters that can combine in any way — this needs a query shape robust to every possible combination, not just one optional value.

**Relevant Concepts:** `SYS_REFCURSOR` as an `OUT` parameter (the return-to-client mechanism) + the `(column = :bind OR :bind IS NULL)` pattern generalized across all four filters, using bind variables throughout for both safety and shared-pool efficiency.

**Step-by-Step Approach:**
1. Declare the procedure with four optional `IN` parameters (all defaulting to `NULL`) and one `OUT SYS_REFCURSOR`.
2. Build a single SQL string whose text never changes regardless of which filters are actually supplied — each filter expressed as `(condition OR :bind IS NULL)`.
3. Open the REF CURSOR against that string, passing every value through `USING`.

**Solution:**
```sql
CREATE OR REPLACE PROCEDURE search_employees_dashboard (
   p_dept_id        IN employees.department_id%TYPE DEFAULT NULL,
   p_min_salary     IN employees.salary%TYPE DEFAULT NULL,
   p_hire_date_from IN DATE DEFAULT NULL,
   p_hire_date_to   IN DATE DEFAULT NULL,
   p_result         OUT SYS_REFCURSOR
) IS
   v_sql VARCHAR2(1000);
BEGIN
   v_sql := 'SELECT employee_id, first_name, last_name, department_id, salary, hire_date
             FROM employees
             WHERE (department_id = :dept_id      OR :dept_id      IS NULL)
               AND (salary       >= :min_salary   OR :min_salary   IS NULL)
               AND (hire_date    >= :hire_from     OR :hire_from     IS NULL)
               AND (hire_date    <= :hire_to       OR :hire_to       IS NULL)';

   OPEN p_result FOR v_sql USING p_dept_id, p_dept_id,
                                   p_min_salary, p_min_salary,
                                   p_hire_date_from, p_hire_date_from,
                                   p_hire_date_to, p_hire_date_to;
END;
/
```

**Explanation:** Each of the four filters is independently "switched off" by its own `OR :bind IS NULL` clause, so any combination of chosen/unchosen filters is handled correctly by one single query — the dashboard can call this procedure the exact same way regardless of which fields the user actually filled in. Critically, the **SQL text itself never changes** between calls, no matter which filters are NULL or populated — only the bound values differ — which lets Oracle reuse one cached, parsed execution plan across every call, rather than hard-parsing a new statement for every distinct filter combination a naive conditional-concatenation approach (like Example C earlier in this topic) would produce.

**Alternative Approach — and a genuinely important performance trade-off:** an alternative is conditional string concatenation, building only the `AND` clauses actually needed for the filters that were supplied (still using bind placeholders for the values themselves, never raw concatenation of the values):
```sql
v_sql := 'SELECT employee_id, first_name, last_name, department_id, salary, hire_date FROM employees WHERE 1=1';
IF p_dept_id IS NOT NULL THEN
   v_sql := v_sql || ' AND department_id = :dept_id';
END IF;
-- ... similarly for the other three filters, each appending only if supplied ...
```
This produces a **different SQL text per distinct combination of supplied filters**, meaning Oracle ends up caching several different parsed statements instead of one — worse for shared-pool reuse than the single fixed-text version above. However, it has a real countervailing advantage: the `(column = :bind OR :bind IS NULL)` pattern used in the main solution is a well-known case where Oracle's query optimizer can struggle to use indexes efficiently, since it must plan for both branches of the `OR` regardless of the actual bound value at execution time — on a very large `employees` table with tight performance requirements, this could mean the "always identical SQL" version scales worse per individual query than the conditionally-built version, even though the latter costs more in parse-caching variety. There is no universally "correct" choice here — it's a genuine trade-off between **shared-pool efficiency** (favors the fixed-text version) and **per-query execution-plan efficiency** (favors the conditionally-built version), and the right call depends on data volume, how often this procedure is called, and how selective each filter actually is in practice. Recognizing that this is a real, debated trade-off — not a simple right-or-wrong pick — is itself a company-level, senior-developer-level insight worth having.

**Common Mistakes to Watch For:**
- Concatenating any of the filter *values* (not just conditional clause fragments) directly into the SQL string — reintroducing the SQL-injection risk from Exercise 6, even if the clause structure itself is built conditionally.
- Forgetting that a plain `WHERE department_id = p_dept_id` (without the `OR ... IS NULL` companion) silently excludes every row whenever that particular filter is left `NULL`, due to SQL's three-valued logic — a direct repeat of the Exercise 4 / Topic 2 NULL-comparison trap, easy to reintroduce when a fourth filter is added carelessly without the matching `OR :bind IS NULL`.
- Trying to consume `p_result` with a cursor FOR loop inside any PL/SQL caller — must use the manual fetch loop, per Exercise 5.

---

**End of Topic 6 — all new concepts from the syllabus are now covered.** Next file: `07-final-combined-case-studies.md`, which combines Topics 1–6 into realistic, mixed business scenarios without naming which concept(s) each one requires — exactly as originally requested for the final practice set.