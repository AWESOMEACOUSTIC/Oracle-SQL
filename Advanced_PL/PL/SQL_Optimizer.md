# PL/SQL Optimizer, Subprogram Inlining & Context Switching

> Oracle PL/SQL Exam Prep: Study Guide
> Topics: PRAGMA basics · PLSQL_OPTIMIZE_LEVEL · PRAGMA INLINE · Context switching · BULK COLLECT / FORALL · PRAGMA UDF

---

## 1. What is a PRAGMA?

A **pragma** is an instruction to the **compiler**, not a statement that executes at runtime. Its syntax is `PRAGMA name(...)`, and it changes how the compiler handles your code. When execution reaches a pragma line, nothing happens, because its effect was already applied at compile time.

> 💡 Analogy: a pragma is like a sticky note on a recipe that says "prep this step ahead of time." The note isn't a cooking step. It changes how the cook prepares.

### Common PL/SQL pragmas

| Pragma | What it tells the compiler |
|---|---|
| `AUTONOMOUS_TRANSACTION` | This subprogram runs in its own independent transaction |
| `EXCEPTION_INIT` | Bind a user-defined exception name to an Oracle error number |
| `SERIALLY_REUSABLE` | Package state lasts for one server call only, not the whole session |
| `INLINE` | Inline (or don't inline) calls to a subprogram, which is **this topic** |
| `UDF` (12c+) | Compile a function so it is cheaper to call **from SQL** |

---

## 2. The PL/SQL Optimizer

Since **Oracle 10g**, PL/SQL compilation includes an optimizer. It can rearrange and transform your source code so it runs faster, **without changing its logical behavior**.

Examples of what it can do:
- Move a calculation whose result never changes **out of a loop** (computed once instead of N times)
- Remove computations whose results are never used
- **Inline** small subprograms (section 3)

It is controlled by the **compile-time** parameter `PLSQL_OPTIMIZE_LEVEL`.

### Levels

| Level | Behavior |
|---|---|
| **0** | No optimization at all (legacy/compatibility use only) |
| **1** | Basic optimizations, but code is **not rearranged**. This is the lowest level that still applies some optimization. |
| **2 (default)** | Full optimization, **including code rearrangement**. Subprogram inlining requires an explicit `PRAGMA INLINE(subprogram, 'YES')` before each statement you want inlined. |
| **3** | Most aggressive. The compiler **automatically seeks out subprograms to inline** without needing the pragma. The pragma can still raise or lower priority. *(Level 3 was added in 11g.)* |

### Setting and checking the level

```sql
-- Session level: affects units compiled AFTER this
ALTER SESSION SET PLSQL_OPTIMIZE_LEVEL = 3;

-- Recompile one unit at a specific level
ALTER PROCEDURE my_proc COMPILE PLSQL_OPTIMIZE_LEVEL = 3;

-- Recompile while keeping the unit's existing settings
ALTER PROCEDURE my_proc COMPILE REUSE SETTINGS;

-- See which level each unit was compiled with
SELECT name, type, plsql_optimize_level
FROM   user_plsql_object_settings;
```

> ⚠️ **Exam point:** the level is **stored with each compiled unit**. If you change the session parameter, units that are already compiled stay at their old level **until they are recompiled**.

### Why does Level 1 exist?

It is an **escape hatch**. In rare cases, the code rearrangement done at level 2 or higher can:
- cause an exception to be raised **earlier, later, or not at all** in edge cases, or
- add **excessive compile-time overhead** for very large units.

Level 1 keeps basic optimizations and **disables the rearrangement**.

---

## 3. Subprogram Inlining

A normal call has overhead. The program jumps to the subprogram, passes parameters, runs the body, returns, and resumes. For a **tiny function called millions of times**, that overhead can cost more than the function's own work.

**Inlining** replaces the call with an **actual copy of the subprogram's code at the call site**:

```sql
-- What you wrote
v := add_nums(1,2) + add_nums(3,4);

-- What the compiler effectively produces after inlining
v := (1 + 2) + (3 + 4);
```

| ✅ Benefit | ❌ Cost |
|---|---|
| Call overhead is eliminated entirely | Larger compiled code (the body is copied at each call site) |
| Opens the door to further optimizations on the combined code | Longer compile time |

> ⚠️ **Same-unit rule:** inlining is only possible when the caller and callee are in the **same compiled unit**. Examples are a local subprogram in the same block, or two subprograms in the **same package body**. A standalone function in a separate unit **cannot** be inlined.

---

## 4. PRAGMA INLINE

```sql
PRAGMA INLINE (subprogram_name, 'YES');   -- request inlining
PRAGMA INLINE (subprogram_name, 'NO');    -- forbid inlining
```

### 🔑 Placement rule (frequently tested)

The pragma affects **only the single declaration or statement immediately after it**, not the whole block.

#### ❌ Wrong: the pragma has no effect

```sql
DECLARE
  FUNCTION add_nums (a PLS_INTEGER, b PLS_INTEGER) RETURN PLS_INTEGER IS
  BEGIN
    RETURN a + b;
  END;
  PRAGMA INLINE (add_nums, 'YES');   -- applies to the next declaration...
  v NUMBER;                          -- ...which contains no call, so nothing is inlined
BEGIN
  v := add_nums(1,2) + add_nums(3,4);  -- NOT inlined at level 2
END;
/
```

#### ✅ Correct

```sql
DECLARE
  FUNCTION add_nums (a PLS_INTEGER, b PLS_INTEGER) RETURN PLS_INTEGER IS
  BEGIN
    RETURN a + b;
  END;
  v NUMBER;
BEGIN
  PRAGMA INLINE (add_nums, 'YES');
  v := add_nums(1,2) + add_nums(3,4);   -- BOTH calls in this statement are inlined

  v := add_nums(5,6);                   -- NOT covered: the pragma applied only to the previous statement
END;
/
```

### Pragma behavior by level

| Level | `'YES'` | `'NO'` |
|---|---|---|
| **0 / 1** | No inlining happens | n/a |
| **2** | Turns inlining on for that statement. Without it, **nothing** is inlined. | Prevents inlining |
| **3** | **Raises priority** (the compiler may already inline the call on its own) | Prevents inlining, overriding the compiler's own decision |

> ⚠️ If both `'YES'` and `'NO'` apply to the same subprogram in the same statement, **`'NO'` wins**.

**When to use `'NO'`:** at level 3, when profiling (e.g., the hierarchical profiler `DBMS_HPROF`) shows that automatic inlining of a specific call **hurt** performance or bloated the code.

**Checking what was inlined:** enable compiler warnings (`ALTER SESSION SET PLSQL_WARNINGS = 'ENABLE:ALL';`), and the compiler reports inlining decisions, e.g., *PLW-06005: inlining of call of procedure ... was done*.

---

## 5. Context Switching

SQL and PL/SQL run on **two separate engines**. A **context switch** happens **any time control moves between them**. One switch is cheap, but millions in a loop are slow.

### Direction 1: PL/SQL → SQL (the one people usually think of)
- DML (`INSERT`, `UPDATE`, `DELETE`, `MERGE`)
- Implicit and explicit cursors, `SELECT INTO`, `FETCH`
- `COMMIT` / `ROLLBACK`

### Direction 2: SQL → PL/SQL (⚠️ common exam trap)
- Calling a **PL/SQL function from inside a SQL statement**:

```sql
SELECT emp_id, calc_bonus(salary) FROM employees;   -- 1 million rows = ~1 million switches
```

### Reducing switches

| Problem | Tool | Side it helps |
|---|---|---|
| Row-by-row fetching | `BULK COLLECT` | **Query / fetch** side |
| Row-by-row DML in a loop | `FORALL` | **DML** side |
| PL/SQL function called from SQL | `PRAGMA UDF`, or `WITH FUNCTION` in the query (12c+) | **SQL → PL/SQL** side |

> 🧠 Memory hook: **BULK COLLECT brings data IN** (fetch); **FORALL pushes changes OUT** (DML).

---

## 6. Important Points to Remember

1. A pragma is a **compiler directive**. It does nothing at runtime.
2. The optimizer was introduced in **10g**. **Level 2 is the default.** Level 3 and `PRAGMA INLINE` came in **11g**.
3. Level 1 = optimization **without rearrangement**, used as an escape hatch for edge-case exception timing or huge compile overhead.
4. **Level 2:** inlining is **opt-in** via `PRAGMA INLINE(..., 'YES')`. **Level 3:** the compiler inlines **proactively** without the pragma. This distinction is frequently tested.
5. `PRAGMA INLINE` affects **only the next declaration or statement**.
6. `'NO'` beats `'YES'`.
7. Inlining works only within the **same compiled unit**.
8. The optimize level is **saved per unit** and changes only on recompile.
9. Context switches happen in **both directions**. SQL → PL/SQL (functions in a SELECT) counts too.
10. `BULK COLLECT` → fetch side; `FORALL` → DML side.

---

## 7. Coding Questions with Answers

### Q1. Predict the inlining (level 2)

```sql
ALTER SESSION SET PLSQL_OPTIMIZE_LEVEL = 2;

CREATE OR REPLACE PROCEDURE demo IS
  FUNCTION sq (n PLS_INTEGER) RETURN PLS_INTEGER IS
  BEGIN
    RETURN n * n;
  END;
  x PLS_INTEGER;
BEGIN
  x := sq(2);                    -- (A)
  PRAGMA INLINE (sq, 'YES');
  x := sq(3) + sq(4);            -- (B)
  x := sq(5);                    -- (C)
END;
/
```

Which calls are inlined?

<details><summary>Answer</summary>

Only the two calls in **(B)**. At level 2 nothing is inlined without the pragma, and the pragma covers only the statement right after it. (A) comes before the pragma, and (C) comes after the covered statement.
</details>

---

### Q2. Same code, level 3

If the same procedure is compiled with `PLSQL_OPTIMIZE_LEVEL = 3`, what changes?

<details><summary>Answer</summary>

The compiler may inline **(A)** and **(C)** on its own, because `sq` is tiny and a good candidate. **(B)** has its inlining priority **raised** by the pragma. The pragma isn't required at level 3, but it still affects priority.
</details>

---

### Q3. Prevent inlining of one call at level 3

Rewrite so that, at level 3, the call in **(C)** is **never** inlined while the others can be.

<details><summary>Answer</summary>

```sql
BEGIN
  x := sq(2);
  PRAGMA INLINE (sq, 'YES');
  x := sq(3) + sq(4);
  PRAGMA INLINE (sq, 'NO');
  x := sq(5);                    -- guaranteed not inlined
END;
```
</details>

---

### Q4. Find the bug

A developer says: "I added PRAGMA INLINE but nothing got faster." Find the problem.

```sql
CREATE OR REPLACE PACKAGE BODY util_pkg IS
  PROCEDURE run IS
    total NUMBER := 0;
  BEGIN
    PRAGMA INLINE (tax_calc, 'YES');
    FOR i IN 1 .. 1000000 LOOP
      total := total + tax_calc(i);
    END LOOP;
  END;
END util_pkg;
/
-- tax_calc is a STANDALONE function created separately
```

<details><summary>Answer</summary>

There are two problems:
1. `tax_calc` is a **standalone function** in a different unit. Inlining only works within the **same compiled unit**. **Fix:** move `tax_calc` into `util_pkg` as a private (body-only) function.
2. The pragma applies to the **next statement**, which is the whole `FOR` loop statement. That part is acceptable. The cleaner habit is to put it directly before the assignment inside the loop:

```sql
FOR i IN 1 .. 1000000 LOOP
  PRAGMA INLINE (tax_calc, 'YES');
  total := total + tax_calc(i);
END LOOP;
```
</details>

---

### Q5. Rewrite with BULK COLLECT + FORALL

Rewrite the following to minimize context switches:

```sql
BEGIN
  FOR r IN (SELECT emp_id, salary FROM employees WHERE dept_id = 10) LOOP
    UPDATE employees
    SET    salary = r.salary * 1.10
    WHERE  emp_id = r.emp_id;
  END LOOP;
  COMMIT;
END;
/
```

<details><summary>Answer</summary>

```sql
DECLARE
  TYPE t_ids  IS TABLE OF employees.emp_id%TYPE;
  TYPE t_sals IS TABLE OF employees.salary%TYPE;
  l_ids  t_ids;
  l_sals t_sals;
BEGIN
  SELECT emp_id, salary
  BULK COLLECT INTO l_ids, l_sals          -- one switch for the fetch
  FROM   employees
  WHERE  dept_id = 10;

  FORALL i IN 1 .. l_ids.COUNT             -- one switch for all updates
    UPDATE employees
    SET    salary = l_sals(i) * 1.10
    WHERE  emp_id = l_ids(i);

  COMMIT;
END;
/
```

The original does roughly one switch per row for the UPDATE (the cursor FOR loop already prefetches 100 rows at a time at level 2+, but the DML is still row by row). The rewrite does one switch for the fetch and one for the DML.

*(The best solution here is a single SQL statement: `UPDATE employees SET salary = salary * 1.10 WHERE dept_id = 10;`. Mention it in interviews.)*
</details>

---

### Q6. FORALL with error handling

Modify the FORALL from Q5 so one bad row doesn't stop the rest, and print which rows failed.

<details><summary>Answer</summary>

```sql
DECLARE
  -- ... same declarations as Q5 ...
  bulk_errors EXCEPTION;
  PRAGMA EXCEPTION_INIT (bulk_errors, -24381);
BEGIN
  -- ... BULK COLLECT as in Q5 ...
  BEGIN
    FORALL i IN 1 .. l_ids.COUNT SAVE EXCEPTIONS
      UPDATE employees
      SET    salary = l_sals(i) * 1.10
      WHERE  emp_id = l_ids(i);
  EXCEPTION
    WHEN bulk_errors THEN
      FOR j IN 1 .. SQL%BULK_EXCEPTIONS.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE(
          'Row ' || SQL%BULK_EXCEPTIONS(j).ERROR_INDEX ||
          ' failed: ' || SQLERRM(-SQL%BULK_EXCEPTIONS(j).ERROR_CODE));
      END LOOP;
  END;
  COMMIT;
END;
/
```

Note the use of **`PRAGMA EXCEPTION_INIT`**, another pragma, to name error ORA-24381.
</details>

---

### Q7. Reduce SQL → PL/SQL switching

This query is slow on 5 million rows. Give two 12c+ ways to reduce the cost of calling the function.

```sql
CREATE OR REPLACE FUNCTION fmt_name (p_first VARCHAR2, p_last VARCHAR2)
RETURN VARCHAR2 IS
BEGIN
  RETURN UPPER(p_last) || ', ' || INITCAP(p_first);
END;
/
SELECT fmt_name(first_name, last_name) FROM customers;
```

<details><summary>Answer</summary>

**Option 1: `PRAGMA UDF`**

```sql
CREATE OR REPLACE FUNCTION fmt_name (p_first VARCHAR2, p_last VARCHAR2)
RETURN VARCHAR2 IS
  PRAGMA UDF;
BEGIN
  RETURN UPPER(p_last) || ', ' || INITCAP(p_first);
END;
/
```

**Option 2: `WITH FUNCTION` in the query**

```sql
WITH
  FUNCTION fmt_name (p_first VARCHAR2, p_last VARCHAR2) RETURN VARCHAR2 IS
  BEGIN
    RETURN UPPER(p_last) || ', ' || INITCAP(p_first);
  END;
SELECT fmt_name(first_name, last_name) FROM customers;
/
```

**Option 3 (often the best):** no function at all, pure SQL:
`SELECT UPPER(last_name) || ', ' || INITCAP(first_name) FROM customers;`
</details>

---

### Q8. Check and change the optimize level

Write the statements to (a) find the optimize level of procedure `CALC_PAYROLL`, (b) recompile only it at level 3, and (c) recompile it later without losing that setting.

<details><summary>Answer</summary>

```sql
-- (a)
SELECT name, plsql_optimize_level
FROM   user_plsql_object_settings
WHERE  name = 'CALC_PAYROLL';

-- (b)
ALTER PROCEDURE calc_payroll COMPILE PLSQL_OPTIMIZE_LEVEL = 3;

-- (c)
ALTER PROCEDURE calc_payroll COMPILE REUSE SETTINGS;
```
</details>

---

### Q9. Count the context switches

How many SQL ↔ PL/SQL context-switch types happen here? Name them.

```sql
BEGIN
  INSERT INTO audit_log (msg) VALUES ('start');
  UPDATE orders SET status = get_status(order_id);
  COMMIT;
END;
/
```

<details><summary>Answer</summary>

- `INSERT`: PL/SQL → SQL
- `UPDATE`: PL/SQL → SQL
- `get_status(order_id)` inside the UPDATE: **SQL → PL/SQL, once per row updated** (the trap)
- `COMMIT`: PL/SQL → SQL

The per-row function calls are the most expensive part.
</details>

---

## 8. Interview Questions with Answers

**Q1. What is a pragma in PL/SQL?**
A compiler directive. It changes how the compiler processes code and does nothing at runtime. Examples: `AUTONOMOUS_TRANSACTION`, `EXCEPTION_INIT`, `SERIALLY_REUSABLE`, `INLINE`, `UDF`.

**Q2. What is `PLSQL_OPTIMIZE_LEVEL` and what is its default?**
A compile-time parameter (0–3) that controls how aggressively the PL/SQL compiler optimizes code. The default is **2**.

**Q3. What's the difference between levels 2 and 3?**
Both do full optimization with rearrangement. At **2**, inlining happens only where `PRAGMA INLINE(..., 'YES')` is used. At **3**, the compiler identifies and inlines candidates automatically, and the pragma only adjusts priority (`'YES'`) or blocks inlining (`'NO'`).

**Q4. Why would anyone use level 1?**
To keep basic optimizations while preventing code rearrangement. Rearrangement can, in rare edge cases, change when (or whether) an exception is raised, or cause heavy compile overhead on very large units.

**Q5. What is subprogram inlining? What are its pros and cons?**
Replacing a call with a copy of the subprogram's body at the call site. Pro: removes call overhead and enables further optimization. Con: larger compiled code and longer compile time.

**Q6. Can any function be inlined?**
No. The caller and callee must be in the **same compiled unit** (e.g., the same package body or the same block). Standalone subprograms in other units can't be inlined.

**Q7. What is the scope of `PRAGMA INLINE`?**
Only the immediately following declaration or statement. Every call to the named subprogram in that one statement is affected.

**Q8. If both `PRAGMA INLINE(f,'YES')` and `PRAGMA INLINE(f,'NO')` apply, what happens?**
`'NO'` wins.

**Q9. I changed `PLSQL_OPTIMIZE_LEVEL` in my session, but my package's behavior didn't change. Why?**
The level is captured at compile time and stored with the unit. You must recompile the package for the new level to take effect.

**Q10. How do you verify that inlining actually happened?**
Enable compiler warnings (`PLSQL_WARNINGS = 'ENABLE:ALL'`). The compiler reports inlining decisions (e.g., PLW-06005). You can also measure with the hierarchical profiler (`DBMS_HPROF`).

**Q11. What is a context switch?**
The transfer of control between the SQL engine and the PL/SQL engine. Each switch has overhead, so row-by-row processing in loops ("slow-by-slow") is costly.

**Q12. Name all the situations that cause a context switch.**
PL/SQL → SQL: DML, `SELECT INTO`, implicit and explicit cursors and fetches, `COMMIT`/`ROLLBACK`. SQL → PL/SQL: calling a user-defined PL/SQL function from a SQL statement (e.g., in a SELECT list or WHERE clause).

**Q13. BULK COLLECT vs FORALL: which does what?**
`BULK COLLECT` reduces switches on the **fetch/query** side by loading many rows into collections in one call. `FORALL` reduces switches on the **DML** side by sending many DML executions in one call.

**Q14. Is FORALL a loop?**
No. Although it looks like one, it is a single statement that sends the whole batch of DML to the SQL engine at once. It can contain only **one** DML statement.

**Q15. How do you limit memory when using BULK COLLECT on a huge table?**
Use an explicit cursor with `FETCH ... BULK COLLECT INTO ... LIMIT n` inside a loop, processing in chunks (e.g., 100–1000 rows).

**Q16. How do you reduce the cost of calling PL/SQL functions from SQL?**
Use `PRAGMA UDF` on the function, define it with `WITH FUNCTION` inside the query (12c+), use result caching or deterministic functions where appropriate, or best of all, replace the function with pure SQL.

**Q17. Does the optimizer ever change what my code computes?**
It is designed not to change logical behavior. The known edge case is exception timing under code rearrangement, which is why level 1 exists.

**Q18. Interviewer: "Your PL/SQL loop updates 1 million rows and is slow. Walk me through your fix."**
1. First ask whether it can be **one SQL statement**. That is usually fastest.
2. If procedural logic is needed, use **BULK COLLECT with LIMIT** plus **FORALL**, with **SAVE EXCEPTIONS** for error handling.
3. Check for **PL/SQL functions called inside the SQL** (SQL → PL/SQL switches) and apply `PRAGMA UDF` or rewrite them as SQL.
4. For hot, tiny helper subprograms in the same package, consider **inlining** (level 3, or `PRAGMA INLINE` at level 2).
5. **Measure** with the hierarchical profiler before and after.

---

## 9. Quick Self-Check

1. At level 2, with no pragma anywhere, are any calls inlined? → **No**
2. Can a standalone function `f` be inlined into procedure `p`? → **No**, they aren't in the same unit
3. The pragma is before statement A, and statement B calls the same function. Is B inlined at level 2? → **No**
4. Which bulk feature helps an UPDATE loop? → **FORALL**
5. Is `SELECT my_func(col) FROM t` a context switch? → **Yes**, SQL → PL/SQL, once per row
6. `'YES'` and `'NO'` both apply. Which wins? → **'NO'**