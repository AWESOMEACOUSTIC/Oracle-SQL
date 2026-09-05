# Module 3, Topic 3: Functions (with Lend a Hand)

---

## 1. What Is a Function?

A **function** is a named, storable PL/SQL sub program — just like a procedure — but with one defining difference: **a function must compute and `RETURN` exactly one value.**

Where a procedure is a **verb** that *does* something, a function is closer to a **noun/value producer** — it answers a question and hands back a single answer: `CalculateTax(amount)`, `GetFullName(emp_id)`, `IsEligibleForBonus(emp_id)`, `TotalOrderValue(order_id)`.

---

## 2. Why Do Functions Exist? What Problem Do They Solve?

Consider this requirement:

> "Multiple reports need to calculate an employee's annual bonus using the same formula: 10% of salary if tenure > 5 years, else 5%."

If every report writer re-implements this formula independently:
- Formula drifts over time as people copy-paste and tweak it slightly.
- A business rule change (say, bonus tiers change) means hunting down every place the formula was duplicated.

A **function** solves this by centralizing the *computation* itself as a single, reusable, callable unit — and critically, because it returns a single value, it can be **used directly inside SQL** (`SELECT`, `WHERE`, `ORDER BY`), which a procedure cannot do. This is huge: it means the calculation logic can live in reports, dashboards, ad-hoc queries, and other PL/SQL code — all pulling from the same single source of truth.

Functions also solve:
- **Composability** — a function's result can be immediately used in an expression, nested inside another function call, or passed as an argument elsewhere, without needing an intermediate variable declaration+call+read cycle like a procedure would (via OUT parameter) require.
- **Read-time consistency** — application queries and reports needing a computed value (like `GetDiscountedPrice`) can call the function inline rather than duplicating math everywhere.

---

## 3. Why Is It Used? (The Business Case)

Real business use cases for functions:
- **Derived/calculated values**: tax amounts, discounted prices, formatted names, age from date of birth, eligibility flags.
- **Validation checks that return true/false**: "is this customer a VIP?", "is this invoice overdue?"
- **Reusable lookups used inside reports/queries**: e.g., `SELECT employee_id, get_full_name(employee_id) FROM employees;` — this lets a formula be used *as if it were a column*.

---

## 4. Syntax: Creating a Function

```sql
CREATE [OR REPLACE] FUNCTION function_name
    (parameter_name [IN | OUT | IN OUT] datatype [DEFAULT default_value], ...)
    RETURN return_datatype
IS
    -- declaration section (optional)
BEGIN
    -- execution section (mandatory)
    -- must contain at least one RETURN statement that returns return_datatype
EXCEPTION
    -- exception handling section (optional)
END function_name;
/
```

### Syntax Breakdown

| Element | Meaning |
|---|---|
| `RETURN return_datatype` | **Mandatory** in the function header — declares the datatype of the single value this function will hand back. This is the core syntactic difference from a procedure. |
| `RETURN expression;` (inside the body) | The actual statement that sends a value back to the caller **and immediately exits the function** — any code after an executed `RETURN` in that path won't run. |
| Parameter modes | Technically `IN`, `OUT`, and `IN OUT` are all *allowed* in a function's parameter list — but using `OUT`/`IN OUT` in a function is considered poor practice (see below) and blocks the function from being called inside SQL. In real-world practice, function parameters are almost always `IN`. |

---

## 5. Simple Examples

### Example 1 — Basic function
```sql
CREATE OR REPLACE FUNCTION get_full_name (p_first VARCHAR2, p_last VARCHAR2)
    RETURN VARCHAR2
IS
BEGIN
    RETURN p_first || ' ' || p_last;
END get_full_name;
/
```
Calling it:
```sql
DECLARE
    v_name VARCHAR2(100);
BEGIN
    v_name := get_full_name('Arun', 'Kumar');
    DBMS_OUTPUT.PUT_LINE(v_name);
END;
/
```

### Example 2 — Function using data from a table
```sql
CREATE OR REPLACE FUNCTION get_employee_salary (p_emp_id NUMBER)
    RETURN NUMBER
IS
    v_salary NUMBER;
BEGIN
    SELECT salary INTO v_salary
    FROM employees
    WHERE employee_id = p_emp_id;

    RETURN v_salary;
END get_employee_salary;
/
```

### Example 3 — Calling a function directly inside SQL
```sql
SELECT employee_id, get_employee_salary(employee_id) AS salary
FROM employees;
```
This is the single most important practical capability a function has that a procedure does not — using it as if it were a computed column.

### Example 4 — Function returning a boolean-style flag
```sql
CREATE OR REPLACE FUNCTION is_high_earner (p_salary NUMBER)
    RETURN VARCHAR2
IS
BEGIN
    IF p_salary > 100000 THEN
        RETURN 'YES';
    ELSE
        RETURN 'NO';
    END IF;
END is_high_earner;
/
```
> **Prerequisite note:** This example uses `IF...THEN...ELSE`, which is formally a Module 2 topic (Selection Statements) we haven't covered yet. It's shown here minimally because "eligibility check" functions are extremely common and hard to demonstrate meaningfully otherwise. We'll return to `IF` logic properly and in depth in Module 2. Also note: PL/SQL does have a native `BOOLEAN` type, but SQL itself has no boolean type, so functions meant to be called from SQL typically return `VARCHAR2('YES'/'NO')` or `NUMBER (1/0)` instead of `BOOLEAN` — this is a real, common design pattern, not a shortcut.

---

## 6. Detailed Explanation — What's Really Happening

- A function **must** return a value on **every possible execution path**. If there's a code path where control reaches `END` without hitting a `RETURN`, Oracle raises a runtime error (`ORA-06503: PL/SQL: Function returned without value`). This is a real and common bug — especially once `IF/ELSE` branches are involved and one branch forgets a `RETURN`.
- `RETURN` in a function **terminates execution immediately** — it's not just "set the return value," it's "exit now." (Contrast: in a procedure, a bare `RETURN;` with no expression can also be used to exit early, but it doesn't carry a value.)
- When called from SQL, functions run in a slightly restricted environment (this is called "purity rules" in Oracle) — for example, a function called from a `SELECT` generally should not perform DML (`INSERT`/`UPDATE`/`DELETE`) or depend on/modify package-level state, depending on the SQL context. This restriction exists to keep SQL queries predictable and side-effect-free.

---

## 7. When to Use / When Not to Use a Function

**Use a function when:**
- The requirement is phrased around **computing/deriving/returning a single value**: "calculate," "determine," "check whether," "format," "convert."
- The result needs to be used **inside a SQL statement** — a `SELECT` column, a `WHERE` condition, an `ORDER BY` expression.
- You want composability — nesting the result into another expression or function call directly.

**Don't use a function when:**
- You need to return **more than one distinct value** — that pushes you back toward a procedure with multiple `OUT` parameters (or, in advanced cases, a function returning a composite/record type — outside our current syllabus).
- The task is fundamentally about **performing an action with side effects** (updating data, sending something, orchestrating multiple steps) — that's a procedure's job. Using a function purely for its side effects (and ignoring its return value) is considered bad practice, because it hides the fact that data is being modified inside something that looks like a harmless calculation.

---

## 8. Function vs. Procedure — Direct Comparison

*(This directly sets up the next topic, "Difference between Procedures and Functions," but it's worth anchoring here while it's fresh.)*

| Aspect | Procedure | Function |
|---|---|---|
| Returns a value? | No (uses OUT params instead) | Yes, exactly one, via `RETURN` |
| Callable from SQL (`SELECT`/`WHERE`)? | No | Yes (with restrictions) |
| Called as | Standalone statement | Part of an expression |
| Best for | Performing actions | Computing/deriving values |
| Typical parameter modes | IN, OUT, IN OUT | IN (almost always) |

---

## 9. Common Mistakes & Misconceptions

1. **Mistake**: Missing a `RETURN` on some code path (commonly inside an `IF` where one branch forgets it) → runtime error `ORA-06503`.
2. **Misconception**: "Functions can't have OUT parameters." → They technically *can*, but it's discouraged, and a function with `OUT`/`IN OUT` parameters **cannot** be called from a SQL statement (only PL/SQL). This defeats much of the point of using a function in the first place.
3. **Mistake**: Using a function purely to perform an `UPDATE`/`INSERT` and discarding its return value, just to "trick" SQL into running DML from a query. This is a serious anti-pattern — DML inside SQL-context functions can be blocked outright by Oracle in many contexts, and even where allowed, it's considered dangerous, unpredictable design.
4. **Misconception**: "A function is just a procedure that returns something." → Directionally true, but the SQL-callability and single-value-return constraint carry real design implications, not just syntax differences.
5. **Mistake**: Forgetting that `RETURN` exits immediately — writing code after a `RETURN` inside the same branch expecting it to still execute.

---

## 10. Edge Cases to Be Aware Of

- If a function is called from SQL and internally hits an unhandled exception (e.g., `NO_DATA_FOUND` from a `SELECT INTO`), the entire SQL statement calling it fails — a single row's bad function call can abort a whole query. This is an important production risk to be aware of.
- Functions can be **overloaded** (same name, different parameter signatures) — this is coming up as its own topic later in this module, but keep it in mind as a capability functions and procedures both share.
- A function's return datatype **cannot** specify a size constraint in the header — e.g., you write `RETURN VARCHAR2`, not `RETURN VARCHAR2(50)`. The size is only relevant for the variable that eventually stores the returned value.

---

## 11. Interview-Level / Practical Notes

- A very common interview question: *"Can you call a procedure inside a SELECT statement?"* — No. *"Can you call a function inside a SELECT statement?"* — Yes, with some restrictions around side effects (purity rules).
- *"Why might a function called from SQL not be able to perform an UPDATE?"* — Because SQL statements should be predictable/side-effect-free (especially since Oracle may evaluate a function multiple times or in unpredictable order across rows) — this is exactly why purity rules exist.
- Real teams build large libraries of small, focused functions like `get_full_name`, `format_currency`, `is_active_customer` — this becomes especially powerful once wrapped into **packages** (coming later in this module).

---

## Things You Must Remember

- Every function **must** declare a `RETURN datatype` in its header and execute a `RETURN expression;` on every possible path.
- `RETURN` **exits the function immediately** — nothing after it (in that execution path) runs.
- Functions are the **only** sub program type callable directly inside SQL statements (`SELECT`, `WHERE`, `ORDER BY`).
- Function parameters are, by convention, almost always `IN` — avoid `OUT`/`IN OUT` in functions unless you have a very specific reason, since it blocks SQL-context usage.
- Using a function purely for DML side effects (and ignoring its return value) is a design anti-pattern — use a procedure instead.
- `ORA-06503` (function returned without value) is a very common real bug — always verify every branch returns something.

## How to Recognize This Concept

Think **function** when the requirement uses language like:
- "**Calculate** the ...", "**Determine** whether...", "**Return** the ...", "**Compute** the ...", "**Format**/**Convert** the ..."
- The result needs to be **displayed as part of a report or query** — "show me a column with X computed for each row."
- The result is a **single, well-defined answer** — a number, a flag, a formatted string.
- You picture yourself wanting to write `SELECT ..., my_logic(...) FROM ...` — that's the clearest possible signal for "this should be a function."

If the requirement instead needs to **change data**, or needs to hand back **multiple pieces of information**, that's your cue to go back to a **procedure**.

---

## Easy Exercises

1. **(Basic function)** Write a function `celsius_to_fahrenheit` that accepts a temperature in Celsius (`NUMBER`) and returns the Fahrenheit equivalent.
2. **(String function)** Write a function `get_initials` that accepts a first name and last name and returns a string like `"A.K."` (first letters of each, each followed by a period).
3. **(Function using a table)** Write a function `get_department_name` that accepts a `department_id` and returns the department's name from a table `departments(department_id, department_name)`.
4. **(Numeric calculation)** Write a function `calculate_tax` that accepts an amount (`NUMBER`) and a tax rate percentage (`NUMBER`) and returns the tax amount.
5. **(Default parameter)** Write a function `apply_markup` that accepts a cost (`NUMBER`) and a markup percentage (`NUMBER DEFAULT 15`) and returns the marked-up price.
6. **(Call inside SQL)** Using the `get_department_name` function from Exercise 3, write a `SELECT` statement against an `employees` table that shows `employee_id`, `first_name`, and the department name using your function.

## Intermediate Exercises

7. **(Predict the error)** What's wrong with this function, and why will it fail at runtime for certain inputs?
   ```sql
   CREATE OR REPLACE FUNCTION classify_amount (p_amount NUMBER)
       RETURN VARCHAR2
   IS
   BEGIN
       IF p_amount > 1000 THEN
           RETURN 'HIGH';
       END IF;
   END classify_amount;
   /
   ```
   *(Yes, this uses `IF`, previewed minimally as noted earlier — focus your answer on the function-specific problem, not the IF syntax itself.)*

8. **(Function vs. Procedure judgment)** A requirement says: *"Given an order ID, return the total order value AND whether the order qualifies for free shipping (yes/no)."* Should this be one function, two functions, or a procedure? Justify your choice in 3–4 sentences.

9. **(Realistic business scenario)** Business requirement: *"Reports need a reusable way to check if a customer is a 'Gold' tier customer — defined as someone whose total lifetime purchase amount exceeds 50,000, based on a table `customers(customer_id, lifetime_purchase_amount)`."* Design and write this as a function. Decide the return type yourself and justify it (e.g., why `VARCHAR2` vs `NUMBER` vs `BOOLEAN`).

10. **(SQL-context restriction reasoning)** Explain in your own words why Oracle restricts functions called from SQL statements from freely performing `INSERT`/`UPDATE`/`DELETE`. What real-world bug could happen if this restriction didn't exist and a report-writer's function silently updated data every time a dashboard query ran?

11. **(Edge case)** In the `get_employee_salary` function from Example 2 above, what happens if `p_emp_id` doesn't exist in the table? What is the practical business impact if this function is called from inside a `SELECT` that's scanning thousands of rows and one row's `employee_id` doesn't have a match?

---

## Lend a Hand — Applied Practice (No Concept Tags)

These follow the "Lend a Hand" pattern — realistic, workplace-style requirements. Work through the same discipline: read the requirement, decide function vs. procedure, identify the single return value and its type, then write it.

**A.** *"Sales wants a reusable check: given an `order_id`, is it overdue? An order is overdue if its `expected_delivery_date` has passed and its `status` is not 'DELIVERED'. Table: `orders(order_id, expected_delivery_date, status)`."*

**B.** *"Payroll needs a formula: given an employee's `years_of_service`, return their leave entitlement in days — 15 days if under 5 years, 20 days if 5–10 years, 25 days if over 10 years."* (Note: You can describe the logic in pseudocode/plain English if `IF/ELSIF` syntax isn't familiar yet — focus on getting the function signature and structure right.)

**C.** *"The finance dashboard needs each invoice row annotated with a formatted display string like `'INV-00234 ($1,200.00)'`, combining the invoice number and amount, for use directly in a report query."*

---

*Whenever you're ready, share your attempts — all at once or in batches — and I'll review them. After that, we'll move to the next file: Difference between Procedures and Functions.*
