# Topic 2: Cursors With Parameter and Without Parameter

> **Syllabus position:** Topic 2 of 6 (+ final combined practice set)
> **Builds on:** Topic 1 (cursor lifecycle, `%TYPE`, `%ROWTYPE`)
> **Feeds into:** Topic 3 (dedicated harder decision-making drills between the two forms)

---

## 1. Concept

A cursor **without a parameter** gets its filter values either from a **hardcoded literal** in its query, or from an **outer PL/SQL variable** that's in scope wherever the cursor is declared. A cursor **with a parameter** declares its own input values explicitly, the same way a procedure or function declares a parameter list — the cursor's query then uses those parameters instead of a literal or an outer variable.

```sql
-- Without parameter (relies on an outer variable)
DECLARE
   v_dept_id employees.department_id%TYPE := 30;
   CURSOR c_emp IS
      SELECT employee_id, first_name FROM employees WHERE department_id = v_dept_id;

-- With parameter (self-contained)
DECLARE
   CURSOR c_emp (p_dept_id employees.department_id%TYPE) IS
      SELECT employee_id, first_name FROM employees WHERE department_id = p_dept_id;
```

## 2. Purpose / Why It Exists

Without parameters, if you want to run the *same* cursor logic against *different* filter values, you have two options: hardcode a literal (rigid — you'd need a separate cursor per value) or reference an outer variable (works, but the cursor's behavior now silently depends on whatever that variable happens to hold at the moment you call `OPEN`). That second option is fragile in larger blocks — anyone reading the `CURSOR ... IS` line alone can't tell what inputs it needs; they have to go hunt for where the variable gets set.

Cursor parameters fix this by making the inputs **explicit and local to the cursor**, exactly like a function signature documents what it needs. This means:
- The same cursor can be **opened multiple times with different values** in the same block, without redeclaring it.
- The cursor is self-documenting — its parameter list tells you exactly what it depends on.
- It enables clean **nested/master-detail cursor patterns** (see below), which are extremely common in real ETL and reporting code.

## 3. What Real-World Problem It Solves

Any time the *same shape* of query needs to run repeatedly against varying filter values — processing one department at a time, running the same report logic per region, or driving an inner cursor's filter from an outer cursor's current row (e.g., "for each department, list its employees"). Parameters let you write that query logic once and reuse it cleanly.

## 4. When to Use / When NOT to Use

**Use a parameterized cursor when:**
- The same cursor will be opened more than once in the block with different filter values.
- You're building a nested cursor pattern where an inner cursor's filter depends on an outer cursor's current row.
- You want the cursor's dependencies to be explicit rather than implicit (better readability/maintainability — a real code-review consideration).

**A non-parameterized cursor is fine when:**
- It's opened exactly once, with either a genuinely fixed literal or a single outer variable set once before `OPEN`.
- Introducing a parameter would add no real reuse value — e.g., a one-shot report cursor inside a simple block.

**Recognition clue:** if the requirement implies "do this for each region/department/customer the caller specifies" or "this logic needs to run multiple times with different inputs," that's a parameter signal. If it's "run this one specific report once," a plain cursor is enough — don't over-engineer.

## 5. Syntax

```sql
CURSOR cursor_name (param1 datatype [DEFAULT default_value]
                    [, param2 datatype [DEFAULT default_value] ...]) IS
   SELECT ...
   FROM ...
   WHERE some_column = param1;
```

**Opening it:**
```sql
OPEN cursor_name(value1, value2);          -- positional notation
OPEN cursor_name(param1 => value1);        -- named notation
OPEN cursor_name;                          -- only valid if ALL parameters have defaults
```

### Syntax Breakdown

- Parameters are declared in parentheses right after the cursor name, **before** `IS`.
- Each parameter has a **datatype only** — no size/precision/scale is allowed (e.g., `p_name VARCHAR2` is valid, `p_name VARCHAR2(20)` is **not**, and will raise a compile error). This is the same rule that applies to all PL/SQL subprogram parameters, not something unique to cursors.
- `%TYPE` **is** allowed and is the recommended style: `p_dept_id employees.department_id%TYPE`.
- `DEFAULT` lets a parameter be omitted at `OPEN` time, falling back to the given value.
- Cursor parameters are **IN-only** — there is no `OUT` or `IN OUT` mode for cursor parameters (unlike procedure/function parameters). They only pass values *into* the query; they can never return anything.
- Parameter scope is **local to the cursor** — you cannot reference a cursor's parameters anywhere outside that cursor's own query.
- To use a *different* parameter value on a cursor that's already open, you must `CLOSE` it first, then `OPEN` it again with the new value. Attempting to `OPEN` an already-open cursor raises `ORA-06511`, regardless of whether you're supplying new parameter values.

## 6. Types / Variations

| Form | Description |
|---|---|
| **No-parameter, literal-based** | `WHERE department_id = 30` — fixed, single-purpose. |
| **No-parameter, outer-variable-based** | `WHERE department_id = v_dept_id` — flexible in theory (the variable is re-read fresh every time you `OPEN`), but implicit and less self-documenting. |
| **Single-parameter cursor** | One filter value passed in, e.g., `(p_dept_id NUMBER)`. |
| **Multi-parameter cursor** | Several filter values, e.g., `(p_dept_id NUMBER, p_min_salary NUMBER)`. |
| **Parameter with default value** | `(p_dept_id NUMBER DEFAULT 10)` — can be opened with or without overriding. |
| **Nested/parameterized inner cursor** | An inner cursor's parameter is fed from an outer cursor's current row — the classic master-detail pattern. |

## 7. Simple Examples

### Example A — No parameter, literal
```sql
DECLARE
   CURSOR c_emp IS
      SELECT employee_id, first_name FROM employees WHERE department_id = 30;
BEGIN
   FOR rec IN c_emp LOOP
      DBMS_OUTPUT.PUT_LINE(rec.first_name);
   END LOOP;
END;
/
```
*(Note: this uses a cursor FOR loop for brevity, which is formally Topic 4 — you'll see it a lot from here on since it's the natural way to consume a cursor once you know the underlying OPEN/FETCH/CLOSE mechanics from Topic 1.)*

### Example B — No parameter, outer variable
```sql
DECLARE
   v_dept_id employees.department_id%TYPE := 30;
   CURSOR c_emp IS
      SELECT employee_id, first_name FROM employees WHERE department_id = v_dept_id;
BEGIN
   OPEN c_emp;
   -- ... fetch loop ...
   CLOSE c_emp;

   v_dept_id := 40;          -- change the variable
   OPEN c_emp;                -- re-opens using the NEW value of v_dept_id
   -- ... fetch loop ...
   CLOSE c_emp;
END;
/
```
**Important behavior:** the outer variable is re-evaluated **fresh at every `OPEN`**, not fixed once at `DECLARE` time — so this *does* technically support reuse with different values. The downside isn't that it can't be reused; it's that the dependency is invisible from the cursor declaration itself, and it's easy to forget to update the variable before the next `OPEN`, or to have some other part of the block silently change it.

### Example C — Single parameter
```sql
DECLARE
   CURSOR c_emp (p_dept_id employees.department_id%TYPE) IS
      SELECT employee_id, first_name FROM employees WHERE department_id = p_dept_id;
BEGIN
   OPEN c_emp(30);
   -- ... fetch loop ...
   CLOSE c_emp;

   OPEN c_emp(40);
   -- ... fetch loop ...
   CLOSE c_emp;
END;
/
```

### Example D — Multiple parameters with a default
```sql
DECLARE
   CURSOR c_emp (p_dept_id employees.department_id%TYPE,
                 p_min_salary employees.salary%TYPE DEFAULT 0) IS
      SELECT employee_id, first_name, salary
      FROM employees
      WHERE department_id = p_dept_id
        AND salary > p_min_salary;
BEGIN
   OPEN c_emp(30);             -- p_min_salary defaults to 0
   -- ...
   CLOSE c_emp;

   OPEN c_emp(30, 5000);       -- override the default
   -- ...
   CLOSE c_emp;
END;
/
```

### Example E — Nested (master-detail) parameterized cursor
This is the single most common real-world use of cursor parameters:

```sql
DECLARE
   CURSOR c_dept IS
      SELECT department_id, department_name FROM departments;

   CURSOR c_emp (p_dept_id departments.department_id%TYPE) IS
      SELECT first_name, salary FROM employees WHERE department_id = p_dept_id;
BEGIN
   FOR dept_rec IN c_dept LOOP
      DBMS_OUTPUT.PUT_LINE('Department: ' || dept_rec.department_name);

      FOR emp_rec IN c_emp(dept_rec.department_id) LOOP
         DBMS_OUTPUT.PUT_LINE('   - ' || emp_rec.first_name || ' (' || emp_rec.salary || ')');
      END LOOP;
   END LOOP;
END;
/
```
Here, the **outer cursor** drives department rows one at a time, and for **each** department row, the **inner cursor** is opened fresh with that department's ID as the parameter. Using cursor FOR loops (Topic 4) here automatically handles opening and closing the inner cursor on every outer iteration — no manual `OPEN`/`CLOSE` bookkeeping needed, which is precisely why parameterized cursors and FOR loops are so often used together.

## 8. Detailed Explanation

- **Parameters are evaluated once, at `OPEN` time.** Whatever value you pass in is locked in for that opening of the cursor — if some other variable it was derived from changes afterward, it has no further effect until you close and reopen with a new value.
- **Positional vs. named notation:** `OPEN c_emp(30)` matches parameters by position; `OPEN c_emp(p_dept_id => 30)` matches by name. Named notation is especially useful with multiple optional parameters, letting you skip earlier ones and only override a later one: `OPEN c_emp(p_min_salary => 5000)` would use the default for `p_dept_id` (if it has one) while overriding only the salary filter.
- **The self-reference name-collision trap:** if you name a cursor parameter identically to a column referenced in the same query, Oracle resolves the unqualified name **to the column**, not the parameter, inside that SQL statement. This means `WHERE department_id = department_id` (parameter named the same as the column) silently becomes "department_id equals itself" — always true, for every row — and your filter does nothing at all, without any error being raised. This is a well-known, genuinely dangerous gotcha, which is exactly why the convention of prefixing cursor (and all PL/SQL) parameters with `p_` exists — it isn't just a style preference, it prevents this specific bug.
- **NULL parameter values:** if a parameter is passed as `NULL` and used in a `WHERE column = p_param` comparison, it will match **zero rows** — because in SQL, `x = NULL` is never `TRUE` (three-valued logic), it evaluates to `UNKNOWN`. If a business rule genuinely needs "match everything when no filter value is given," the query needs explicit handling like `WHERE (p_param IS NULL OR column = p_param)`, not a plain equality.

## 9. Common Mistakes and Misconceptions

1. **Declaring a constrained parameter datatype** — `p_name VARCHAR2(20)` — compile error; must be unconstrained (`VARCHAR2`, `NUMBER`, or `%TYPE`).
2. **The name-collision trap** described above — parameter name identical to a column name, silently defeating the filter.
3. **Assuming you can change a parameter's effective value on an already-open cursor** without closing it first — you can't; you must `CLOSE` then `OPEN` again.
4. **Treating cursor parameters as if they support `OUT`** — they're IN-only; there's no way to get a value back out through a cursor parameter.
5. **Passing NULL and expecting an equality filter to match everything** — it matches nothing, due to SQL's three-valued logic.
6. **Forgetting that an "outer variable" cursor is re-evaluated at every `OPEN`** — some learners assume it's fixed permanently at `DECLARE` time, which isn't true, but relying on this behavior is still a maintainability weakness even though it technically "works."

## 10. Edge Cases

- All parameters have defaults → `OPEN cursor_name;` (no parentheses needed) is valid.
- Passing `NULL` explicitly for a parameter used in `=` comparison → returns zero rows (see above); if you needed "no filter," this must be handled deliberately in the WHERE clause.
- Reopening an already-open cursor (with same or different values) without closing first → `ORA-06511: PL/SQL: cursor already open`.
- Nested cursor pattern where the inner cursor is opened via a manual `OPEN`/`CLOSE` (not a FOR loop) inside an outer loop — if you forget to `CLOSE` the inner cursor before the outer loop's next iteration tries to `OPEN` it again, you get `ORA-06511` on the second outer iteration. Using a cursor FOR loop for the inner cursor avoids this entirely, since FOR loops always close automatically, including on early exits.

## 11. How This Relates to Other Topics

- **Topic 1 (lifecycle, `%TYPE`/`%ROWTYPE`):** everything here still follows OPEN → FETCH → check → CLOSE; parameters only change what values feed the query, not the lifecycle itself.
- **Topic 3 (dedicated practice):** Topic 3 will drill much harder on *choosing* between parameterized and non-parameterized forms in ambiguous, realistic situations — this topic focused on mechanics and basic recognition; Topic 3 sharpens the decision-making.
- **Topic 4 (Cursor FOR loops):** parameterized cursors are opened directly inside a FOR loop's header (`FOR rec IN c_emp(30) LOOP`), and this combination is the standard idiom for nested/master-detail processing, as shown in Example E.
- **Topic 6 (REF CURSOR):** a parameterized cursor still has a **fixed query shape** at compile time — only the filter *values* vary. A REF CURSOR generalizes further, letting even the query itself vary at runtime. Don't confuse "flexible values" (this topic) with "flexible query" (Topic 6).

---

## Things You Must Remember

- Cursor parameters are declared with **unconstrained datatypes only** — no size/precision.
- Cursor parameters are **IN-only**; there's no OUT/IN OUT.
- To reuse an open cursor with a new value, you must `CLOSE` it first — you cannot just "re-open" it while it's still open.
- **Never name a cursor parameter the same as a column it filters against** — Oracle resolves the unqualified name to the column, and your filter silently becomes a no-op self-comparison. Prefix parameters (`p_`) as a habit, not a style choice.
- A non-parameterized cursor referencing an outer variable *is* re-evaluated at each `OPEN` — it isn't "frozen," but it's still less maintainable than an explicit parameter.
- `NULL` passed into an `=` comparison matches nothing — handle that case explicitly if "no filter" needs to mean "match all."

## How to Recognize This Concept

Think **"parameterized cursor"** when a requirement says things like:
- "For each [region/department/customer/branch] the user specifies..." (explicit reusable input)
- "Run the same report/check for every [X], one at a time" (nested/master-detail signal)
- "The script should be reusable for different [values] without changing the code"

Think **"plain cursor is enough"** when the filter is genuinely fixed for the life of that one script/report run, and there's no indication it needs to run again with different values in the same execution.

---

# Exercises — With Answers and Reasoning

## Easy (Syntax & Basics)

### Exercise 1
**Task:** Write two versions of the same cursor for retrieving orders by `customer_id` from an `orders` table — one hardcoded with a literal, one using a parameter. Explain the reusability difference.

**Solution:**
```sql
-- Version 1: hardcoded literal
DECLARE
   CURSOR c_orders_hardcoded IS
      SELECT order_id, order_date, amount FROM orders WHERE customer_id = 501;
BEGIN
   NULL;
END;
/

-- Version 2: parameterized
DECLARE
   CURSOR c_orders_param (p_customer_id orders.customer_id%TYPE) IS
      SELECT order_id, order_date, amount FROM orders WHERE customer_id = p_customer_id;
BEGIN
   OPEN c_orders_param(501);
   -- ... fetch loop ...
   CLOSE c_orders_param;

   OPEN c_orders_param(502);
   -- ... fetch loop ...
   CLOSE c_orders_param;
END;
/
```

**Reasoning:** Version 1 can only ever retrieve customer 501's orders — to check a different customer you'd need a second, near-identical cursor declaration, which duplicates logic. Version 2 declares the query shape once and can be opened for *any* customer ID at runtime, including looping through a list of customer IDs and reopening it each time (with a `CLOSE` in between). This is the core reusability benefit parameters provide.

---

### Exercise 2
**Task:** Write a parameterized cursor with two parameters (`p_dept_id`, `p_min_salary`) returning employees in that department earning more than the given salary. Open it twice with different values.

**Solution:**
```sql
DECLARE
   CURSOR c_high_earners (p_dept_id employees.department_id%TYPE,
                          p_min_salary employees.salary%TYPE) IS
      SELECT first_name, salary
      FROM employees
      WHERE department_id = p_dept_id
        AND salary > p_min_salary;

   v_first_name employees.first_name%TYPE;
   v_salary     employees.salary%TYPE;
BEGIN
   OPEN c_high_earners(30, 5000);
   LOOP
      FETCH c_high_earners INTO v_first_name, v_salary;
      EXIT WHEN c_high_earners%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_first_name || ' - ' || v_salary);
   END LOOP;
   CLOSE c_high_earners;

   OPEN c_high_earners(40, 8000);
   LOOP
      FETCH c_high_earners INTO v_first_name, v_salary;
      EXIT WHEN c_high_earners%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_first_name || ' - ' || v_salary);
   END LOOP;
   CLOSE c_high_earners;
END;
/
```

**Reasoning:** Both parameters are passed positionally, matching declaration order (`p_dept_id` first, `p_min_salary` second). Each `OPEN`/`CLOSE` pair is a fully independent "run" of the cursor with its own values — the cursor must be closed after the first run before it can legally be opened again for the second.

---

### Exercise 3
**Task:** Write a parameterized cursor with a default value for one parameter. Open it once using the default and once overriding it.

**Solution:**
```sql
DECLARE
   CURSOR c_emp (p_dept_id employees.department_id%TYPE,
                 p_status employees.status%TYPE DEFAULT 'ACTIVE') IS
      SELECT first_name FROM employees
      WHERE department_id = p_dept_id AND status = p_status;

   v_first_name employees.first_name%TYPE;
BEGIN
   OPEN c_emp(30);                      -- uses default 'ACTIVE'
   LOOP
      FETCH c_emp INTO v_first_name;
      EXIT WHEN c_emp%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_first_name);
   END LOOP;
   CLOSE c_emp;

   OPEN c_emp(30, 'INACTIVE');          -- overrides the default
   LOOP
      FETCH c_emp INTO v_first_name;
      EXIT WHEN c_emp%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_first_name);
   END LOOP;
   CLOSE c_emp;
END;
/
```

**Reasoning:** The first `OPEN` supplies only `p_dept_id`, so `p_status` falls back to `'ACTIVE'`. The second `OPEN` explicitly supplies both, overriding the default. Defaults only apply to parameters you *omit* — supplying any value, even one identical to the default, counts as an explicit override (functionally the same result, but conceptually it's not "using the default," it's "passing a matching value").

---

### Exercise 4
**Task:** Take a cursor that references an outer variable (non-parameter form) and rewrite it using a parameter instead. Explain why the parameterized version is generally better practice.

**Given (non-parameter form):**
```sql
DECLARE
   v_region VARCHAR2(20) := 'SOUTH';
   CURSOR c_sales IS
      SELECT sale_id, amount FROM sales WHERE region = v_region;
BEGIN
   OPEN c_sales;
   -- ...
   CLOSE c_sales;
END;
/
```

**Rewritten (parameterized form):**
```sql
DECLARE
   CURSOR c_sales (p_region sales.region%TYPE) IS
      SELECT sale_id, amount FROM sales WHERE region = p_region;
BEGIN
   OPEN c_sales('SOUTH');
   -- ...
   CLOSE c_sales;
END;
/
```

**Reasoning:** Functionally, both versions can be reused with different values (the original can be reused by reassigning `v_region` before each `OPEN`, since it's re-evaluated fresh at `OPEN` time — see the "Detailed Explanation" section above). The real difference is **clarity and safety, not raw capability**: in the parameterized version, anyone reading the `CURSOR c_sales (...)` declaration immediately knows what input drives it, with no need to trace back through the block to find where `v_region` gets set or whether something else might change it first. It also removes any risk of forgetting to update the variable before a later `OPEN`, or of another part of a larger block accidentally modifying `v_region` between the intended uses. This is a maintainability argument, not a "one is broken" argument — both are syntactically valid PL/SQL.

---

## Intermediate (Reasoning & Edge Cases)

### Exercise 5
**Task:** Spot the bug. Given:
```sql
CURSOR c_dept (department_id NUMBER) IS
   SELECT * FROM employees WHERE department_id = department_id;
```
Explain why this cursor always returns **every** employee regardless of the value passed in, and fix it.

**Answer:** This is the **name-collision trap** described in the Detailed Explanation section. The cursor's parameter is named `department_id` — identical to the actual `department_id` **column** on the `employees` table. Inside the SQL statement embedded in the cursor, Oracle resolves an unqualified identifier to the **table column** in preference to a same-named PL/SQL parameter. As a result, `WHERE department_id = department_id` doesn't compare the column to the parameter at all — it compares the column to **itself**, which is trivially `TRUE` for every row (as long as `department_id` isn't `NULL`, since `NULL = NULL` is `UNKNOWN`, not `TRUE` — so rows with a `NULL` department would actually be excluded, an extra subtlety). The parameter value you pass in at `OPEN` is effectively ignored. No compile-time or runtime error is raised, which is exactly what makes this bug dangerous — it fails silently and would likely only be caught by noticing the output looks wrong.

**Fix:**
```sql
CURSOR c_dept (p_department_id employees.department_id%TYPE) IS
   SELECT * FROM employees WHERE department_id = p_department_id;
```
Prefixing the parameter (`p_department_id`) removes the naming collision entirely, so the comparison correctly filters the column against the value passed in.

---

### Exercise 6
**Task:** Loop through each department in `departments`, and for each one, open a parameterized cursor to list its employees (nested/master-detail pattern). Explain what would go wrong if the inner cursor weren't closed before the outer loop's next iteration tried to reopen it.

**Solution:**
```sql
DECLARE
   CURSOR c_dept IS
      SELECT department_id, department_name FROM departments;

   CURSOR c_emp (p_dept_id departments.department_id%TYPE) IS
      SELECT first_name FROM employees WHERE department_id = p_dept_id;

   v_dept_id   departments.department_id%TYPE;
   v_dept_name departments.department_name%TYPE;
   v_emp_name  employees.first_name%TYPE;
BEGIN
   OPEN c_dept;
   LOOP
      FETCH c_dept INTO v_dept_id, v_dept_name;
      EXIT WHEN c_dept%NOTFOUND;

      DBMS_OUTPUT.PUT_LINE('Department: ' || v_dept_name);

      OPEN c_emp(v_dept_id);
      LOOP
         FETCH c_emp INTO v_emp_name;
         EXIT WHEN c_emp%NOTFOUND;
         DBMS_OUTPUT.PUT_LINE('   - ' || v_emp_name);
      END LOOP;
      CLOSE c_emp;   -- <-- critical: must close before the outer loop tries to reopen it
   END LOOP;
   CLOSE c_dept;
END;
/
```

**Reasoning — what goes wrong without the `CLOSE c_emp;` line:**
On the **first** outer iteration, `OPEN c_emp(v_dept_id)` succeeds normally, since `c_emp` starts closed. But if the inner `CLOSE c_emp;` is removed, the cursor is left open when the outer loop moves to its **second** iteration and tries `OPEN c_emp(v_dept_id)` again with the new department's ID — this raises `ORA-06511: PL/SQL: cursor already open`, because you cannot open a cursor that's already open, even to pass it a different parameter value. The block would fail on the second department, not the first, which is a classic symptom worth recognizing: "works for one iteration then blows up" is a strong signal of a missing `CLOSE` inside a loop.

**Note:** this exact failure mode disappears if you use a cursor **FOR loop** for the inner cursor instead (`FOR emp_rec IN c_emp(v_dept_id) LOOP ... END LOOP;`), since FOR loops open and close the cursor automatically on every entry/exit — this is one of the strongest practical reasons FOR loops (Topic 4) and parameterized cursors are so often paired together in real code.

---

## Realistic Scenario

### Exercise 7
**Business Requirement:** *"The finance team needs a script that, for each region code they provide, lists all pending invoices in that region and flags any invoice older than 30 days as overdue. The tool should be reusable so the finance team can rerun it for different regions without any code changes — just supplying a different region code."*

**What Is Being Asked:**
Given a region code as input, retrieve pending invoices for that region, and for each one, indicate whether it's overdue (older than 30 days) or not.

**Key Clues:**
- "for each region code **they provide**" and "reusable... **without any code changes** — just supplying a different region code" → this is a direct, explicit signal for a **parameter**, not a hardcoded literal or an ambient variable. The requirement is practically describing the definition of a parameterized cursor (or, at a higher level, a procedure that takes the region as its own parameter and uses it inside a parameterized cursor).
- "flags any invoice older than 30 days" → per-row calculated condition, needs a date comparison against `SYSDATE`, handled inside the loop (or via a `CASE` expression in the query — both are valid; see Alternative Approach).

**What Data Is Required:** `invoice_id`, `invoice_date`, `amount`, `region`, `status` from an `invoices` table (assuming `status = 'PENDING'` identifies pending invoices).

**Relevant Concepts:** Explicit cursor (Topic 1) + cursor parameter (this topic) for the region filter. No nested cursor is needed here since there's only one level of iteration (invoices), not a master-detail relationship.

**Step-by-Step Approach:**
1. Declare a cursor with `p_region_code` as its parameter.
2. Filter `WHERE region = p_region_code AND status = 'PENDING'` inside the cursor query — same reasoning as the warehouse-stock exercise in Topic 1: only pending invoices in the target region are relevant at all, so filter them out at the SQL level rather than fetching everything and branching.
3. Inside the loop, compute the age of each invoice (`SYSDATE - invoice_date`) and print an "OVERDUE" or normal message accordingly.
4. Wrap the whole thing so it can be re-run for a different region simply by opening the cursor with a different argument — no code edits needed.

**Solution:**
```sql
DECLARE
   CURSOR c_pending_invoices (p_region_code invoices.region%TYPE) IS
      SELECT invoice_id, invoice_date, amount
      FROM invoices
      WHERE region = p_region_code
        AND status = 'PENDING';

   v_invoice_id   invoices.invoice_id%TYPE;
   v_invoice_date invoices.invoice_date%TYPE;
   v_amount       invoices.amount%TYPE;
   v_age_days     NUMBER;
BEGIN
   OPEN c_pending_invoices('SOUTH');   -- finance team supplies this value; only line that changes per run
   LOOP
      FETCH c_pending_invoices INTO v_invoice_id, v_invoice_date, v_amount;
      EXIT WHEN c_pending_invoices%NOTFOUND;

      v_age_days := TRUNC(SYSDATE) - TRUNC(v_invoice_date);

      IF v_age_days > 30 THEN
         DBMS_OUTPUT.PUT_LINE('OVERDUE - Invoice ' || v_invoice_id ||
            ' (' || v_amount || '), ' || v_age_days || ' days old.');
      ELSE
         DBMS_OUTPUT.PUT_LINE('Pending - Invoice ' || v_invoice_id ||
            ' (' || v_amount || '), ' || v_age_days || ' days old.');
      END IF;
   END LOOP;
   CLOSE c_pending_invoices;
END;
/
```

**Explanation:** The cursor's parameter (`p_region_code`) is the *only* thing that changes when the finance team wants a different region — exactly matching the "reusable without code changes" requirement. The `status = 'PENDING'` filter lives in the query (like the Topic 1 warehouse example) because there's genuinely nothing to report for non-pending invoices here. The overdue check, however, is **not** filtered out in the WHERE clause — both overdue and non-overdue pending invoices must be *shown*, just labeled differently, which is precisely the case where branching inside the loop (rather than filtering in SQL) is the correct choice, contrasting directly with the Topic 1 warehouse scenario where non-matching rows were skipped entirely.

**Alternative Approach:** The overdue flag could instead be computed in the SQL itself using a `CASE` expression:
```sql
SELECT invoice_id, invoice_date, amount,
       CASE WHEN SYSDATE - invoice_date > 30 THEN 'OVERDUE' ELSE 'PENDING' END AS invoice_status
FROM invoices
WHERE region = p_region_code AND status = 'PENDING';
```
This pushes the labeling logic into SQL and simplifies the PL/SQL loop to pure formatting/printing. It's arguably cleaner here since the "overdue" logic is a simple one-line comparison with no procedural branching needed elsewhere (no call to another procedure, no additional side effect per case) — a good discussion point for "when does moving logic into SQL make sense vs. keeping it in PL/SQL." As a rule of thumb: if the per-row logic is a simple expression with no side effects, SQL can often express it directly; once it needs multiple steps, exception handling, or calls to other logic, PL/SQL branching (as in the main solution) becomes necessary.

**Common Mistakes to Watch For:**
- Naming the parameter `region` (same as the column) — would trigger the name-collision trap from Exercise 5, silently breaking the region filter.
- Using `SYSDATE - v_invoice_date` without `TRUNC()` — if `invoice_date` has a non-midnight time component, the day-count comparison can be off by fractional days, causing invoices right at the 30-day boundary to be misclassified.
- Forgetting `CLOSE`, especially if this logic is later wrapped in a procedure that gets called repeatedly for multiple regions in a batch — that would reintroduce exactly the cursor-leak risk discussed in Topic 1, Exercise 6.

---

**End of Topic 2.** Next file: `03-hands-on-drills-parameter-vs-no-parameter.md`.