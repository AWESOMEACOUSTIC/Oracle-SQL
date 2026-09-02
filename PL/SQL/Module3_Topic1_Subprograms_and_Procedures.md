# Module 3, Topic 1: Types of Sub Programs and PL/SQL Procedure

> **Prerequisite note (not part of your syllabus, used only where unavoidable):** A couple of examples below use a variable declaration and a simple `IF` condition to make the procedure examples realistic. These are previewed minimally here only so the examples make sense — they will be taught properly and in full depth when we return to Module 1 (Variables) and Module 2 (Selection Statements). Nothing here substitutes for that.

---

## 1. What Is a Sub Program?

A **sub program** is a **named, storable block of PL/SQL code** that performs a specific task and can be **called (invoked) by other programs** — as many times as needed, from different places (other PL/SQL blocks, applications, schedulers, even SQL statements).

Recall from Topic 1 (Intro to PL/SQL): an **anonymous block** runs once and disappears. A **sub program** is the opposite — it's compiled once, stored permanently in the database (as a database object), and can be executed repeatedly without rewriting the logic each time.

---

## 2. Why Do Sub Programs Exist? What Problem Do They Solve?

Imagine your company has this rule:

> "Whenever an employee's salary is updated, validate that the new salary doesn't exceed the max salary allowed for their job grade, and log the change."

Now imagine 5 different applications (HR portal, batch payroll job, admin console, integration API, migration script) all need to update salaries and all must obey this rule.

**Without sub programs:** every application team writes this validation logic themselves, in their own code, in their own language (Java, Python, etc.). Result:
- Logic gets duplicated 5 times.
- One team updates the rule; the other 4 forget to. Now you have inconsistent business rules across the company — a real and common production bug source.
- Every change requires touching multiple codebases.

**With a sub program (a stored procedure):** you write the validation + update logic **once**, inside the database, as a procedure. All 5 applications simply *call* that procedure. One place to fix, one source of truth, guaranteed consistency.

This is the single biggest reason sub programs exist: **centralizing reusable logic** so it lives in one place, close to the data it operates on.

Other reasons:
- **Modularity** — break a large problem into smaller, named, testable units.
- **Security** — you can grant a user permission to *execute* a procedure without granting them direct access to the underlying tables (this is huge in real companies — app users often don't get direct `UPDATE` rights on sensitive tables; they only get `EXECUTE` rights on a controlled procedure).
- **Performance** — sub programs are compiled once and cached; repeated calls don't need re-parsing.
- **Maintainability** — a business rule change means editing one object, not hunting across many codebases.

---

## 3. Types of Sub Programs (Overview)

PL/SQL has two fundamental categories of *named, storable* sub programs — this topic is specifically about the first one, but you should know both exist and how they differ conceptually:

| Type | Returns a value? | Called how? | Typical purpose |
|---|---|---|---|
| **Procedure** | No (but can output values via `OUT` parameters) | As a standalone statement: `EXEC procedure_name(...);` or inside another PL/SQL block | To **perform an action** — update data, run a process, orchestrate a task |
| **Function** | **Yes**, exactly one value, via `RETURN` | Used **inside an expression**, like `v_x := function_name(...);` or even inside a `SELECT` | To **compute and return a value** |

We'll cover Functions in depth as their own topic shortly. For now, focus entirely on **Procedures**.

There's also a third category — **Packages** — which is not itself a sub program, but a **container that groups related procedures and functions together**. We'll get there once procedures and functions are solid individually.

---

## 4. What Is a PL/SQL Procedure, Specifically?

A **procedure** is a named PL/SQL block designed to **perform an action** (or a sequence of actions). It does not return a value the way a function does — but it *can* communicate results back to the caller using **`OUT` or `IN OUT` parameters**.

Think of a procedure as a **verb** — `UpdateSalary`, `TransferFunds`, `ArchiveOldRecords`, `ValidateAndInsertOrder`. It *does* something.

---

## 5. Syntax: Creating a Procedure

```sql
CREATE [OR REPLACE] PROCEDURE procedure_name
    (parameter_name [IN | OUT | IN OUT] datatype [DEFAULT default_value], ...)
IS
    -- declaration section (optional): local variables, constants
BEGIN
    -- execution section (mandatory): the actual logic
EXCEPTION
    -- exception handling section (optional)
END procedure_name;
/
```

### Syntax Breakdown

| Element | Meaning |
|---|---|
| `CREATE [OR REPLACE]` | `CREATE` makes a new procedure; `OR REPLACE` lets you overwrite an existing one with the same name without dropping it first — used constantly during development. |
| `PROCEDURE procedure_name` | The name you'll use to call it. Must be a valid identifier, unique within its schema (or package). |
| Parameter list | Zero or more parameters, each with a **mode** (`IN`, `OUT`, `IN OUT`) and a **datatype**. Parentheses are omitted entirely if there are no parameters. |
| `IS` (or `AS`, they're interchangeable) | Marks the start of the procedure body — equivalent to where `DECLARE` would start in an anonymous block, except the keyword is `IS`/`AS` instead. |
| `BEGIN...END` | Same as an anonymous block — mandatory execution section. |
| `END procedure_name;` | Naming the end is optional but strongly recommended for readability, especially in long procedures. |

### Parameter Modes — Critical Concept

This is one of the most important ideas in this entire topic, so let's go deep.

| Mode | Direction | Behavior |
|---|---|---|
| `IN` (default if omitted) | Caller → Procedure | The value is passed **in**. Inside the procedure, it acts like a **read-only constant** — you cannot assign a new value to it. |
| `OUT` | Procedure → Caller | The parameter starts with **no meaningful value** (technically `NULL`) when the procedure begins. The procedure's job is to **assign** a value to it, which the caller then receives after the call. |
| `IN OUT` | Both directions | The caller passes in a value, the procedure can **read it AND overwrite it**, and the final value flows back to the caller. |

**Why this matters for real problems:** When you read a business requirement like *"the procedure should validate the salary and return whether it was successful, along with an error message if not"* — that's your cue: you need **`OUT` parameters** to hand information back to the caller (since a procedure itself doesn't `RETURN` anything).

---

## 6. Simple Examples

### Example 1 — Procedure with no parameters
```sql
CREATE OR REPLACE PROCEDURE greet_user
IS
BEGIN
    DBMS_OUTPUT.PUT_LINE('Hello from the greet_user procedure!');
END greet_user;
/
```
Calling it:
```sql
BEGIN
    greet_user;
END;
/
-- or, in SQL*Plus/SQLcl:
EXEC greet_user;
```

### Example 2 — Procedure with an `IN` parameter
```sql
CREATE OR REPLACE PROCEDURE greet_user_by_name (p_name IN VARCHAR2)
IS
BEGIN
    DBMS_OUTPUT.PUT_LINE('Hello, ' || p_name || '!');
END greet_user_by_name;
/
```
Calling it:
```sql
BEGIN
    greet_user_by_name('Priya');
END;
/
```

### Example 3 — Procedure with an `OUT` parameter
```sql
CREATE OR REPLACE PROCEDURE get_employee_salary
    (p_emp_id IN NUMBER, p_salary OUT NUMBER)
IS
BEGIN
    SELECT salary INTO p_salary
    FROM employees
    WHERE employee_id = p_emp_id;
END get_employee_salary;
/
```
Calling it (must use a variable to *catch* the OUT value):
```sql
DECLARE
    v_salary NUMBER;
BEGIN
    get_employee_salary(100, v_salary);
    DBMS_OUTPUT.PUT_LINE('Salary: ' || v_salary);
END;
/
```

### Example 4 — Procedure with `IN OUT`
```sql
CREATE OR REPLACE PROCEDURE apply_bonus (p_salary IN OUT NUMBER, p_bonus_pct IN NUMBER)
IS
BEGIN
    p_salary := p_salary + (p_salary * p_bonus_pct / 100);
END apply_bonus;
/
```
Calling it:
```sql
DECLARE
    v_salary NUMBER := 50000;
BEGIN
    apply_bonus(v_salary, 10);  -- 10% bonus
    DBMS_OUTPUT.PUT_LINE('New salary: ' || v_salary);  -- prints 55000
END;
/
```

---

## 7. Detailed Explanation — What's Really Happening

- When you `CREATE OR REPLACE PROCEDURE`, Oracle **compiles** the code and stores the compiled object in the database's data dictionary. It's not re-parsed every time you call it — this is where the performance benefit comes from.
- A procedure call is a **separate PL/SQL statement/unit** — it has its own execution, and if it fails with an unhandled exception, that exception propagates to whoever called it, unless the procedure itself has an `EXCEPTION` section that handles it internally.
- **Parameter passing mechanics**: For `IN` parameters, Oracle typically passes by reference (not a full copy) for efficiency, but semantically you should treat it as read-only. For `OUT` and `IN OUT` parameters, the actual copy-back to the caller's variable happens **only if the procedure completes successfully** (this is called "copy-out" semantics) — if an unhandled exception occurs mid-procedure, the OUT variable may remain unassigned in the caller's scope. This is a subtle but important edge case.

---

## 8. When to Use / When Not to Use a Procedure

**Use a procedure when:**
- The requirement is about **performing an action** — updating data, inserting records, orchestrating multiple steps, running a batch process.
- You need to **encapsulate multiple SQL statements** as one logical operation (e.g., "deduct from account A, credit account B" — a transfer).
- You need to return **zero, one, or multiple values** back to the caller (via `OUT` parameters) — a function can only return exactly one value.
- Security matters — you want callers to be able to *do* something without direct table access.

**Don't use a procedure when:**
- You just need to **compute and return a single value** to be used inline in an expression or a `SELECT` — that's a function's job, not a procedure's (a procedure cannot be called inside a `SELECT` or a WHERE clause the way a function can).
- The logic runs exactly once and will never be reused — an anonymous block may suffice.

---

## 9. Common Mistakes & Misconceptions

1. **Mistake**: Trying to assign a new value to an `IN` parameter inside the procedure body.
   → This raises a compile-time error (`PLS-00363: expression cannot be used as an assignment target`). `IN` parameters are read-only inside the procedure.
2. **Mistake**: Calling a procedure and expecting it to "return" a value like a function (`v_x := my_procedure(...)`).
   → Illegal. Procedures cannot be used in expressions. Use `OUT` parameters and call it as a standalone statement.
3. **Misconception**: "OUT parameters need an initial value when calling."
   → False — you pass an **uninitialized (or any) variable**; the procedure will overwrite it.
4. **Mistake**: Forgetting that if a procedure with an `OUT` parameter throws an unhandled exception, the caller's variable may never get assigned — leading to confusing "why is my variable still NULL" bugs downstream.
5. **Misconception**: "CREATE and CREATE OR REPLACE are basically the same."
   → Not quite — plain `CREATE` fails with an error if the object already exists; `CREATE OR REPLACE` silently overwrites it. In real development, `CREATE OR REPLACE` is used almost universally so scripts are re-runnable.
6. **Mistake**: Forgetting that dropping/recreating a procedure invalidates any dependent objects until they're recompiled (this matters more in packages — but worth knowing early).

---

## 10. Edge Cases to Be Aware Of

- If `SELECT ... INTO` inside a procedure returns **zero rows**, Oracle raises `NO_DATA_FOUND`. If it returns **more than one row**, Oracle raises `TOO_MANY_ROWS`. Both are runtime exceptions (we'll cover exception handling for these formally in Module 4) — a procedure like `get_employee_salary` in Example 3 above will **crash** if `p_emp_id` doesn't exist, unless you add exception handling.
- Parameters can have **default values** (`p_bonus_pct IN NUMBER DEFAULT 5`), which lets callers omit them. This is a clean way to handle "optional" business parameters.
- You can call parameters using **positional notation** (`greet_user_by_name('Priya')`) or **named notation** (`greet_user_by_name(p_name => 'Priya')`). Named notation is safer in real systems with many parameters because it removes ambiguity about order — this becomes especially relevant once we cover "mixed and named notation" as its own topic later in this module.

---

## 11. Interview-Level / Practical Notes

- A extremely common interview question: *"Can a procedure be called from inside a SQL SELECT statement?"* — **No.** Only functions can be called inside SQL expressions (with restrictions, covered in the Functions topic). This is one of the clearest procedure-vs-function distinctions.
- *"Why use OUT parameters instead of just making it a function?"* — Because a procedure can return **multiple** values via multiple `OUT` parameters, while a function can only return exactly one. If a requirement says "return the employee's name AND their manager's name AND a status flag," that's 3 outputs → strongly suggests a procedure with 3 `OUT` parameters (or a function returning a composite/record type, a more advanced approach).
- In real companies, procedures are the backbone of "API-like" access to the database for applications — an app team is often given a list of procedures they're allowed to call, rather than raw table access.

---

## Things You Must Remember

- Procedures **do not** use `RETURN` to send back a value (functions do); they use `OUT` / `IN OUT` parameters.
- Parameter mode defaults to `IN` if you don't specify one.
- `IN` parameters are **read-only** inside the procedure body.
- A procedure **cannot** be called inside a SQL expression (e.g., inside a `SELECT` column list or `WHERE` clause) — that's exclusive to functions.
- `CREATE OR REPLACE` is the standard way to develop procedures — it avoids "already exists" errors during iterative development.
- If a procedure fails with an unhandled exception mid-execution, `OUT` parameters may not get assigned back to the caller — don't assume partial success.
- `SELECT...INTO` inside a procedure is risky without exception handling — zero rows or multiple rows both raise runtime errors.

## How to Recognize This Concept

Think **procedure** when the business requirement:
- Uses **action verbs**: "update," "process," "insert," "transfer," "archive," "validate and save," "perform."
- Needs to **return more than one piece of information**, or no information at all (just "do this and confirm it happened").
- Describes a task that will be **triggered repeatedly** by different callers (applications, jobs, other PL/SQL code) — i.e., "whenever X happens, do Y."
- Talks about restricting **who can do what** to data (security/access control angle) — "the app should be able to update salaries but not directly touch the salary table."

If instead the requirement says "calculate," "determine," "compute," or "return a single value for use elsewhere" — that's a strong signal toward a **function** instead (next topic).

---

## Easy Exercises

1. **(No parameters)** Write a procedure `show_current_date` that prints the current system date.
2. **(IN parameter)** Write a procedure `print_department_name` that accepts a department name (`VARCHAR2`) and prints `"Department: <name>"`.
3. **(OUT parameter)** Write a procedure `get_employee_count` that accepts no input and returns (via `OUT`) the total number of rows in an `employees` table.
4. **(IN OUT parameter)** Write a procedure `double_value` that accepts a `NUMBER` as `IN OUT` and doubles it in place.
5. **(Default parameter value)** Write a procedure `apply_discount` that accepts a price (`IN NUMBER`) and a discount percentage (`IN NUMBER DEFAULT 10`), and prints the discounted price.
6. **(Multiple OUT parameters)** Write a procedure `get_employee_details` that accepts an `employee_id` and returns (via two separate `OUT` parameters) the employee's `first_name` and `salary`.

## Intermediate Exercises

7. **(Named notation)** Call the `apply_discount` procedure from Exercise 5 using **named notation**, explicitly skipping the default and supplying your own discount percentage.
8. **(Predict the error)** What will go wrong when this is called, and why?
   ```sql
   CREATE OR REPLACE PROCEDURE bad_example (p_value IN NUMBER)
   IS
   BEGIN
       p_value := p_value + 1;
       DBMS_OUTPUT.PUT_LINE(p_value);
   END bad_example;
   /
   ```
9. **(Procedure vs. plain SQL judgment)** A colleague asks you to "write a procedure that just runs `SELECT * FROM employees`." Should this be a procedure? Explain in 2–3 sentences why or why not, and what you'd suggest instead if the goal is just to view data.
10. **(Realistic business scenario)** Business requirement: *"Create a procedure that accepts an employee ID and a percentage raise. It should update the employee's salary and also return the employee's new salary and their name back to the caller."* Identify: (a) how many parameters you need, (b) the mode of each parameter, and (c) write the procedure.
11. **(Edge case reasoning)** In Exercise 10's procedure, what happens if the `employee_id` passed in doesn't exist in the table? Will your `OUT` parameters get a value? What do you think *should* happen in a real system, even though we haven't formally covered exception handling yet?

---

*Post your answers whenever you're ready — I'll review them and then produce the next file: Module 3, Topic 2 (Lend a Hand on Procedures — a consolidated practice checkpoint) before moving into Functions.*
