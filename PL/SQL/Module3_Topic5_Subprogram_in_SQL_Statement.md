# Module 3, Topic 5: Defining a Sub Program in an SQL Statement

---

## 1. What Is This Concept?

So far, every function or procedure we've written has been created as a **standalone, permanently stored database object** using `CREATE OR REPLACE`. It exists independently, has its own name in the schema, and can be called from anywhere (subject to permissions).

Oracle also allows something different: **defining a function (or procedure) directly inside a single SQL statement**, using a `WITH FUNCTION` (or `WITH PROCEDURE`) clause. This sub program:
- Exists **only for the duration of that one SQL statement**.
- Is **not stored** anywhere — it's not a database object, has no entry in the data dictionary, and cannot be called by anything outside that statement.
- Is scoped entirely to the query it's defined in.

This is sometimes called a **"WITH clause function"** or **"inline PL/SQL function in SQL."**

---

## 2. Why Does This Exist? What Problem Does It Solve?

Consider this situation: you need a small calculation used **only once**, inside **one specific report query**. For example:

> "For this one dashboard query, I need to classify each order as 'RUSH', 'STANDARD', or 'DELAYED' based on some date logic — but no other query or application will ever need this exact classification."

Without this feature, you'd have two unsatisfying choices:
1. Create a full, permanent, named function (`CREATE OR REPLACE FUNCTION classify_order...`) — even though it's only used in one report, cluttering the schema with a "throwaway" object that nobody else needs, and that now needs to be maintained, secured, and remembered as existing.
2. Try to cram the logic into a giant, unreadable nested `CASE`/`DECODE` expression directly in the `SELECT` — technically possible for simple logic, but painful or impossible for anything with real branching, loops, or multiple steps.

The `WITH FUNCTION` clause gives a **third option**: full PL/SQL power (loops, conditionals, exception handling) **scoped tightly to a single query**, with **no schema pollution** and **no separate deployment step**. This is especially valuable for:
- One-off analytical/reporting queries.
- Ad-hoc data investigation where you don't want to leave permanent objects behind.
- Cases where the logic is genuinely query-specific and would be meaningless as a general-purpose reusable function.

---

## 3. Why Is It Used? (The Business Case)

- **Avoiding unnecessary schema clutter** — DBAs and architects generally want the number of permanent stored objects to reflect genuine, reusable business logic — not one-off report tricks. This feature lets developers keep query-specific logic exactly where it's used.
- **Faster iteration for analysts/report writers** — no need to get `CREATE PROCEDURE` privileges granted, no deployment/migration process — the whole thing lives inside the query script itself.
- **Complex per-row logic in reports** — when a report needs genuine procedural logic (loops, multiple conditions, nested checks) that a plain `CASE` expression can't cleanly express, but creating a permanent function feels like overkill.

---

## 4. Syntax

```sql
WITH
    FUNCTION function_name (parameter_name datatype, ...) RETURN return_datatype
    IS
        -- declaration section (optional)
    BEGIN
        -- execution section
        RETURN expression;
    END function_name;
main_query AS (
    SELECT ... FROM ...
)
SELECT ...
FROM main_query
WHERE ... function_name(...) ...
```

More concretely, a full working shape:

```sql
WITH
    FUNCTION get_order_status (p_days_late NUMBER) RETURN VARCHAR2
    IS
    BEGIN
        IF p_days_late <= 0 THEN
            RETURN 'ON TIME';
        ELSIF p_days_late <= 3 THEN
            RETURN 'SLIGHTLY DELAYED';
        ELSE
            RETURN 'SEVERELY DELAYED';
        END IF;
    END;
SELECT order_id,
       expected_delivery_date,
       actual_delivery_date,
       get_order_status(actual_delivery_date - expected_delivery_date) AS status
FROM orders;
```

### Syntax Breakdown

| Element | Meaning |
|---|---|
| `WITH` | Same keyword used for the well-known "WITH clause" (subquery factoring, i.e., naming a subquery to reuse it) — Oracle extended this same clause to also allow function/procedure definitions. |
| `FUNCTION function_name (...) RETURN datatype IS ... END;` | A **complete, ordinary function definition** — syntactically almost identical to `CREATE FUNCTION`, just without the `CREATE [OR REPLACE]` prefix, and it must appear **before** the main query. |
| No semicolon-terminated `CREATE` | This isn't a DDL statement creating a permanent object — it's part of the single SQL statement's syntax tree. |
| Scope | The function is visible **only within that same SQL statement** — you cannot call it from a different query, another session, or any other PL/SQL block. |

> **Important placement rule:** The `WITH FUNCTION`/`WITH PROCEDURE` declarations must come **first**, immediately after `WITH`, before any named subquery factoring clauses in the same `WITH`.

---

## 5. Simple Examples

### Example 1 — Basic inline function
```sql
WITH
    FUNCTION double_it (p_num NUMBER) RETURN NUMBER
    IS
    BEGIN
        RETURN p_num * 2;
    END;
SELECT employee_id, salary, double_it(salary) AS doubled_salary
FROM employees;
```

### Example 2 — Inline function combined with a named subquery (both WITH features together)
```sql
WITH
    FUNCTION bonus_amount (p_salary NUMBER) RETURN NUMBER
    IS
    BEGIN
        RETURN p_salary * 0.1;
    END;
high_earners AS (
    SELECT employee_id, salary
    FROM employees
    WHERE salary > 80000
)
SELECT employee_id, salary, bonus_amount(salary) AS bonus
FROM high_earners;
```

### Example 3 — Using it inside a WHERE clause
```sql
WITH
    FUNCTION is_recent_hire (p_hire_date DATE) RETURN VARCHAR2
    IS
    BEGIN
        IF p_hire_date > ADD_MONTHS(SYSDATE, -6) THEN
            RETURN 'Y';
        ELSE
            RETURN 'N';
        END IF;
    END;
SELECT employee_id, hire_date
FROM employees
WHERE is_recent_hire(hire_date) = 'Y';
```

---

## 6. Detailed Explanation

- The function is **parsed and compiled as part of the SQL statement's execution plan**. It does not persist afterward in any form — run the exact same query again, and Oracle compiles the function fresh each time.
- You can define **multiple** `WITH FUNCTION`/`WITH PROCEDURE` blocks in the same `WITH` clause, and they can even call each other, as long as the calling one is defined after the one it depends on (or via forward-declaration patterns, which is a more advanced detail).
- This feature requires a reasonably modern Oracle version (introduced in Oracle Database 12c Release 1) — worth knowing in case you ever work with an older Oracle environment where this syntax isn't available.
- Despite living inside a `SELECT`, the function body itself has the **full power of PL/SQL** — loops, IF/CASE, exception handling — it is not limited to simple expressions the way a plain `CASE` in SQL is.

---

## 7. When to Use / When Not to Use

**Use `WITH FUNCTION` when:**
- The logic is needed for **one specific query** and has no general reuse case.
- You need real procedural logic (branching, possibly loops) that would be awkward or impossible as a plain SQL expression, but you don't want to create a permanent schema object for it.
- You're doing ad-hoc analysis/reporting and want to keep everything self-contained in one script.

**Don't use it when:**
- The logic **will** be reused across multiple queries, reports, or applications — that's exactly when a permanently stored function (`CREATE OR REPLACE FUNCTION`) is the right call, so it has one authoritative definition.
- The logic is simple enough to express with a plain `CASE` expression — don't reach for PL/SQL just because you can; a simple `CASE WHEN salary > 100000 THEN 'HIGH' ELSE 'LOW' END` doesn't need a `WITH FUNCTION` wrapper.
- You need the logic to be callable from outside SQL too (e.g., another PL/SQL block calling it directly) — a `WITH` function is invisible outside its one statement.

---

## 8. Common Mistakes & Misconceptions

1. **Misconception**: "This creates a real function in the database I can call later." → False. It vanishes the moment the statement finishes executing. Nothing is stored.
2. **Mistake**: Placing the `WITH FUNCTION` block *after* a named subquery factoring clause in the same `WITH` — the function declarations must come first.
3. **Misconception**: "This is basically the same as CREATE FUNCTION, just with different syntax." → The *body* syntax is nearly identical, but the **lifecycle** (temporary vs. permanent) and **scope** (one statement vs. schema-wide) are fundamentally different — this is a scoping/lifecycle decision, not just a stylistic one.
4. **Mistake**: Overusing this feature for logic that's actually reused across many reports — leading to the same formula being re-typed into every query instead of centralized once as a real function. This recreates the exact duplication problem functions were invented to solve, just inside individual queries instead of application code.

---

## 9. Edge Cases to Be Aware Of

- Since the function isn't stored, there's **no independent way to unit test it** outside the context of running the full query — this is a real trade-off versus a permanent function which can be tested and called in isolation.
- Performance-wise, this doesn't inherently behave differently from a permanent function once the query is running — the same purity-rule considerations (predictability, avoiding side effects) still apply if you try to sneak DML into a `WITH FUNCTION`.
- Requires appropriate Oracle version/support — if you're ever unsure whether a target environment supports this, it's worth checking before relying on it in a script that needs to run in older systems.

---

## 10. Interview-Level / Practical Notes

- This is a somewhat advanced/lesser-known feature — being aware of it (even at a conceptual level) tends to stand out positively in interviews, since it shows familiarity with modern Oracle capabilities beyond textbook basics.
- A reasonable interview question: *"How would you use a one-off calculation in a report without creating a permanent database function?"* — This is precisely the scenario `WITH FUNCTION` was designed for.
- Real-world usage tends to be concentrated among data analysts and report developers doing complex ad-hoc queries, rather than in core application backend code (which typically favors permanent, tested, named functions/procedures for anything beyond trivial one-off scripts).

---

## Things You Must Remember

- `WITH FUNCTION` / `WITH PROCEDURE` define a sub program that is **scoped to a single SQL statement only** — never stored, never reusable elsewhere.
- The function/procedure declaration(s) must appear **first** inside the `WITH` clause, before any named subquery factoring (`AS (...)`) clauses.
- Full PL/SQL logic (IF, loops, exception handling) is allowed inside — this is not limited like a plain `CASE` expression.
- This is the right tool for **query-specific, one-off** logic — not a substitute for genuinely reusable functions, which still belong as permanent `CREATE OR REPLACE FUNCTION` objects.
- Requires Oracle 12c or later.

## How to Recognize This Concept

Think of `WITH FUNCTION` when a requirement sounds like:
- "**Just for this report**, I need to classify/calculate/derive something with real logic (not a simple CASE)."
- "**One-time analysis**" or "**ad-hoc query**" language, combined with a need for branching/procedural logic.
- You catch yourself thinking *"this calculation is too complex for a CASE expression, but I don't want to create a permanent function just for this single report."*

If the same logic is described as something **multiple systems/reports will need**, that's your signal to step back to a **permanent function** instead (Topic 3), not a `WITH FUNCTION`.

---

## Exercises

1. **(Basic inline function)** Using a `WITH FUNCTION`, write a query against an `employees(employee_id, salary)` table that shows each employee's salary and a 5% raise amount, computed inline.

2. **(Conditional logic inline)** Write a query using `WITH FUNCTION` that classifies each product in a `products(product_id, price)` table as `'BUDGET'` (price < 500), `'MID-RANGE'` (500–2000), or `'PREMIUM'` (> 2000).

3. **(Combine with named subquery)** Using a table `orders(order_id, customer_id, order_amount)`, write a query that first filters to orders over 1000 using a named subquery (`AS (...)`), then uses a `WITH FUNCTION` to compute a 2% processing fee on the filtered results.

4. **(Judgment call)** A colleague wants to create a permanent function `calculate_late_fee` that will be used in exactly one, never-to-be-repeated data cleanup script that will run once and be deleted. Would you recommend `CREATE FUNCTION` or `WITH FUNCTION` here? Justify your answer in 2–3 sentences.

5. **(Judgment call, reversed)** Another colleague uses a `WITH FUNCTION` inside a critical nightly finance report to calculate "days overdue" — and you discover the exact same calculation is copy-pasted as a `WITH FUNCTION` into six other reports across the company. What would you recommend, and why?

6. **(Realistic business scenario)** Business requirement: *"For a one-time executive dashboard query, show each employee's name, salary, and a computed 'flight risk' flag — HIGH if salary is below the department average AND tenure is under 2 years, otherwise LOW. This is only needed for next week's board meeting and won't be reused."* Design the query using `WITH FUNCTION` (you can describe the department-average lookup in pseudocode if needed — we haven't covered cursors/aggregation-in-PLSQL patterns in depth yet). Explain why this is (or isn't) a good fit for `WITH FUNCTION` versus a permanent function.

---

*Share your attempts whenever ready. Next up: Function Result Cache.*
