# Module 4, Topic 3: User Defined Exception

---

## 1. What Is a User-Defined Exception?

A **user-defined exception** is an exception **you create yourself** to represent a **business rule violation** — a condition Oracle has no built-in awareness of, because it's specific to your company's logic, not a generic database/runtime error.

Unlike pre-defined exceptions (Topic 2), which Oracle recognizes and raises **automatically**, a user-defined exception must be:
1. **Declared** — given a name, in the `DECLARE` section.
2. **Raised explicitly** — using the `RAISE` statement, at the exact point in your code where you've determined the business rule was violated.
3. **Handled** — caught with a `WHEN` clause, exactly like a pre-defined exception, using the name you gave it.

---

## 2. Why Do User-Defined Exceptions Exist? What Problem Do They Solve?

Recall the gap identified at the end of Topic 2:

> "An employee's requested leave exceeds their available balance."

Oracle has **no concept** of "leave balance" — that's pure business logic, unique to your company's rules. There is no `ORA-xxxxx` error code for this. If you tried to rely only on pre-defined exceptions, you'd have no clean way to detect and respond to this condition at all — you'd have to bury the check inside an `IF` statement and just... handle it inline, without the structural benefits of exception handling (separation of "normal logic" from "error response," consistent propagation behavior, ability to be caught by callers, etc.).

**User-defined exceptions solve this by letting you extend PL/SQL's exception mechanism to cover your own business rules** — giving business logic violations the same first-class treatment as genuine runtime errors: a clear name, a dedicated `RAISE` point, a dedicated `WHEN` handler, and the same propagation behavior up the call stack if unhandled locally.

This is important architecturally: it means a **procedure deep in your business logic can raise a meaningful, named business exception**, and a **completely different, higher-level caller** (an application, a different procedure) can catch and respond to that *specific* named condition — without needing to know or care about the underlying implementation details of how that rule was checked.

---

## 3. Why Is It Used? (The Business Case)

- **Validation logic**: "salary can't exceed the job grade's maximum," "order quantity can't exceed available stock," "discount can't exceed 50%" — none of these are things Oracle enforces natively; they're your company's rules, and user-defined exceptions let your code express and enforce them cleanly.
- **Clear, intentional failure signaling**: instead of a procedure silently doing nothing, or returning some ambiguous flag value that a caller might forget to check, raising a named exception makes a rule violation **impossible to silently ignore** — it forces the caller to either handle it or let it propagate visibly.
- **Consistency with the exception-handling model already used for runtime errors**: your codebase gets **one unified way** of communicating "something went wrong" — whether it's a genuine Oracle-level error or a business rule violation — rather than mixing exception handling for one kind of failure and ad-hoc return-flag checking for another.

---

## 4. Syntax

### Step 1 — Declare
```sql
DECLARE
    exception_name EXCEPTION;
BEGIN
    ...
```
Declared in the `DECLARE` section, just like a variable — but with the `EXCEPTION` keyword instead of a datatype. By convention, exception names are often written in `UPPER_SNAKE_CASE`, similar to pre-defined exception names, to visually distinguish them as exceptions.

### Step 2 — Raise
```sql
IF <business condition is violated> THEN
    RAISE exception_name;
END IF;
```
The `RAISE` statement is what actually **triggers** the exception at the point in your code where you've detected the problem — nothing happens automatically; you decide exactly when and where.

### Step 3 — Handle
```sql
EXCEPTION
    WHEN exception_name THEN
        -- response logic
```
Exactly the same handling syntax as pre-defined exceptions — `WHEN` treats your custom exception identically to a built-in one once it's declared.

### Full Example
```sql
DECLARE
    e_salary_too_high EXCEPTION;
    v_max_salary NUMBER := 100000;
    v_new_salary NUMBER := 150000;
BEGIN
    IF v_new_salary > v_max_salary THEN
        RAISE e_salary_too_high;
    END IF;

    DBMS_OUTPUT.PUT_LINE('Salary update approved.');
EXCEPTION
    WHEN e_salary_too_high THEN
        DBMS_OUTPUT.PUT_LINE('Error: Proposed salary exceeds the maximum allowed.');
END;
/
```

---

## 5. Simple Examples

### Example 1 — Basic validation with a user-defined exception
```sql
DECLARE
    e_invalid_quantity EXCEPTION;
    v_order_qty NUMBER := -5;
BEGIN
    IF v_order_qty <= 0 THEN
        RAISE e_invalid_quantity;
    END IF;

    DBMS_OUTPUT.PUT_LINE('Order quantity accepted: ' || v_order_qty);
EXCEPTION
    WHEN e_invalid_quantity THEN
        DBMS_OUTPUT.PUT_LINE('Error: Order quantity must be positive.');
END;
/
```

### Example 2 — User-defined exception inside a procedure (the realistic pattern)
```sql
CREATE OR REPLACE PROCEDURE apply_discount (p_order_id IN NUMBER, p_discount_pct IN NUMBER)
IS
    e_discount_too_high EXCEPTION;
BEGIN
    IF p_discount_pct > 50 THEN
        RAISE e_discount_too_high;
    END IF;

    UPDATE orders
    SET discount_pct = p_discount_pct
    WHERE order_id = p_order_id;

EXCEPTION
    WHEN e_discount_too_high THEN
        DBMS_OUTPUT.PUT_LINE('Error: Discount cannot exceed 50%.');
END apply_discount;
/
```

### Example 3 — Multiple business rules, multiple user-defined exceptions
```sql
CREATE OR REPLACE PROCEDURE submit_leave_request (p_emp_id IN NUMBER, p_days IN NUMBER)
IS
    e_insufficient_balance EXCEPTION;
    e_invalid_days         EXCEPTION;
    v_available_days       NUMBER;
BEGIN
    IF p_days <= 0 THEN
        RAISE e_invalid_days;
    END IF;

    SELECT leave_balance INTO v_available_days
    FROM employees
    WHERE employee_id = p_emp_id;

    IF p_days > v_available_days THEN
        RAISE e_insufficient_balance;
    END IF;

    UPDATE employees
    SET leave_balance = leave_balance - p_days
    WHERE employee_id = p_emp_id;

    DBMS_OUTPUT.PUT_LINE('Leave request approved.');

EXCEPTION
    WHEN e_invalid_days THEN
        DBMS_OUTPUT.PUT_LINE('Error: Number of days must be positive.');
    WHEN e_insufficient_balance THEN
        DBMS_OUTPUT.PUT_LINE('Error: Requested days exceed available leave balance.');
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Error: Employee not found.');
END submit_leave_request;
/
```

Notice how naturally **user-defined exceptions and pre-defined exceptions coexist in the same `EXCEPTION` section** — `NO_DATA_FOUND` (pre-defined, generic runtime) sits alongside `e_invalid_days` and `e_insufficient_balance` (user-defined, business-specific), each handled with equal clarity.

---

## 6. Detailed Explanation

- **Declaring** an exception doesn't do anything by itself — it just gives PL/SQL a name to recognize. Nothing is "checked" automatically; **you** are responsible for writing the `IF` condition and the `RAISE` statement at the correct point in your logic.
- **`RAISE exception_name;`** immediately transfers control to the `EXCEPTION` section, exactly like a pre-defined exception being triggered by Oracle — from that point forward, the behavior (skipping the rest of `BEGIN`, searching for a matching handler, propagating if unmatched) is **identical** to everything you learned in Topic 1.
- **Scope matters**: an exception declared inside a specific block/procedure is only *visible by that name* within that same block. If you need the **same** user-defined exception concept reused across multiple procedures, declaring it separately each time works, but a cleaner, more scalable approach (especially once we consider packages, revisiting Module 3 concepts) is to declare it **once, publicly, in a package specification** — so multiple procedures across the package can `RAISE` and reference the exact same named exception consistently. *(This is a natural, valuable connection back to packages — worth remembering as your business-rule library grows.)*

---

## 7. When to Use / When Not to Use

**Use a user-defined exception when:**
- The failure condition is a **business rule**, not a generic Oracle runtime error — something Oracle has no native way of detecting.
- You want the failure to be **explicitly named and unmissable**, forcing callers to consciously handle or propagate it, rather than silently returning some ambiguous status value.
- The same business rule might need to be checked (and responded to) in **multiple places** — having a consistently-named exception makes the intent clear everywhere it's used.

**Don't use one when:**
- The condition is actually a standard **Oracle runtime error** already covered by a pre-defined exception (Topic 2) — don't reinvent `NO_DATA_FOUND` as a custom exception; use the built-in one.
- The situation is genuinely just a **normal, expected outcome**, not an "error" at all — e.g., "the search returned zero results" in a general product search feature is often just a normal UI state, not something that should be modeled as an exception. Overusing exceptions for routine, expected branching (instead of plain `IF` logic) is a design smell — exceptions should represent genuinely **exceptional**, rule-violating conditions, not everyday control flow.

---

## 8. Common Mistakes & Misconceptions

1. **Mistake**: Declaring an exception but forgetting to actually `RAISE` it anywhere — the `IF` condition exists, but nothing triggers the exception, so the "error" is silently never detected.
2. **Misconception**: "User-defined exceptions are somehow weaker or less 'real' than pre-defined ones." → Once raised, they behave **identically** in every structural respect (propagation, `WHEN` matching, skipping remaining `BEGIN` code) — the only difference is *who* decides when they're raised (you, vs. Oracle automatically).
3. **Mistake**: Using exceptions for routine, expected outcomes rather than genuine rule violations — e.g., raising a custom exception every time a search returns no matches, when that's often just normal application behavior that should be handled with a plain `IF`, not treated as an error.
4. **Mistake**: Declaring the same conceptual exception with different names/scopes across multiple unrelated procedures, leading to inconsistent handling and duplicated logic — this is exactly the kind of duplication a shared package-level declaration (mentioned above) is meant to prevent.
5. **Misconception**: "The RAISE statement needs an explicit condition attached to it." → `RAISE` itself is unconditional — it always fires the moment it's executed. The **conditional logic lives in the surrounding `IF`**, not in the `RAISE` statement itself.

---

## 9. Edge Cases to Be Aware Of

- A user-defined exception carries **no default error message** of its own the way pre-defined exceptions effectively map to a specific Oracle error message — if unhandled, it will surface as a fairly generic PL/SQL error referencing the exception's declared name, not a friendly business message. Producing a **proper, customer/support-friendly error message** for a user-defined exception is exactly what the upcoming topic, **Raise Application Error**, is designed to solve — this is a strong preview of why that topic exists.
- You can `RAISE` a user-defined exception **without any specific message attached at all**, relying entirely on your `WHEN` handler's own logic (e.g., `DBMS_OUTPUT.PUT_LINE`) to communicate what happened — this works for simple internal scripts, but doesn't scale well for real application-facing error communication (again, motivating the next topic).
- If a user-defined exception is declared **locally inside a procedure** (not in a shared package spec) and that procedure doesn't handle it itself, it still propagates to the caller structurally — but the **caller cannot reference it by name** in their own `WHEN` clause (since the name only exists within that procedure's local scope) — the caller would only be able to catch it generically via `WHEN OTHERS`. This is a strong, practical argument for declaring shared business exceptions at the package level when multiple callers need to respond to them specifically by name.

---

## 10. Interview-Level / Practical Notes

- A common interview question: *"How do you handle a business rule violation that Oracle has no built-in exception for?"* — Declare, raise, and handle a user-defined exception; this is the textbook expected answer.
- *"What are the three steps involved in using a user-defined exception?"* — Declare (in DECLARE section), Raise (explicitly, at the point of violation), Handle (in EXCEPTION section) — expect this exact three-step framing to come up.
- *"Why might you declare a user-defined exception at the package level instead of locally inside one procedure?"* — So multiple procedures (and their external callers) can reference and catch that exact same named exception consistently, rather than each procedure inventing its own local, differently-scoped version of conceptually the same business rule.

---

## Things You Must Remember

- Three-step pattern: **Declare** (in `DECLARE`, as `EXCEPTION` type) → **Raise** (explicitly, via `RAISE exception_name;`, inside an `IF`) → **Handle** (via `WHEN exception_name THEN`, same as pre-defined).
- Nothing happens automatically — **you** control exactly when a user-defined exception fires, unlike pre-defined exceptions which Oracle raises on its own.
- Once raised, a user-defined exception behaves **identically** to a pre-defined one in terms of control flow and propagation.
- User-defined exceptions and pre-defined exceptions can (and routinely do) coexist in the same `EXCEPTION` section.
- Exceptions declared locally are only referenceable by name **within that scope** — shared business rules across multiple procedures are better declared once, at the package level.
- Don't overuse exceptions for routine/expected outcomes — reserve them for genuine rule violations.
- User-defined exceptions have no built-in friendly message — that's what `RAISE_APPLICATION_ERROR` (coming up) is for.

## How to Recognize This Concept

Reach for a **user-defined exception** when a business requirement describes a **rule Oracle has no native concept of**, phrased like:
- "...**cannot exceed**...", "...**must not be**...", "...**is not allowed to**..." (a business-specific constraint/limit).
- "...**validate** that... before proceeding..."
- "...if this violates [some company policy/rule], **reject/stop** the operation..."

If the condition you're describing already maps cleanly to a pre-defined exception (a missing row, a math error, a duplicate key) — use that instead; don't reinvent it. And if the condition is genuinely a normal, expected, non-error outcome — reach for a plain `IF`, not an exception at all.

---

## Exercises

1. **(Basic declare-raise-handle)** Write a block with a user-defined exception `e_negative_amount`, raised if a variable `v_amount` is less than zero, handled with an appropriate message.

2. **(Inside a procedure)** Write a procedure `update_product_price` that accepts a `product_id` and a `new_price`. Raise a user-defined exception `e_invalid_price` if `new_price` is less than or equal to zero, and handle it with a clear message. Otherwise, perform the update (you can write the `UPDATE` statement illustratively).

3. **(Multiple business rules)** Write a procedure `process_withdrawal` that accepts an `account_id` and a `withdrawal_amount`. It should raise `e_invalid_amount` if the amount is zero or negative, and `e_insufficient_funds` if the amount exceeds the account's balance (assume a table `accounts(account_id, balance)`). Handle both distinctly, plus handle `NO_DATA_FOUND` for a non-existent account.

4. **(Judgment: exception vs. plain IF)** A requirement says: *"When searching for products by category, if no products match, just show an empty results list — this is a completely normal, expected outcome for the search feature."* Should "no products found" be modeled as a user-defined exception here? Justify your answer in 2–3 sentences, referencing the "when not to use" guidance above.

5. **(Scope reasoning)** Explain, in your own words, what would go wrong (or what limitation you'd hit) if three completely separate, unrelated procedures each independently declared their own **locally-scoped** version of an exception meant to represent the same business rule ("discount exceeds maximum allowed"), rather than sharing one declaration. What would you recommend instead, referencing what you learned about packages in Module 3?

6. **(Realistic business scenario)** Business requirement: *"When finalizing a purchase order, the system must reject the operation if the total order value exceeds the requesting department's remaining budget for the quarter. This is a company policy, not something the database enforces natively."* Design and write a procedure `finalize_purchase_order` implementing this rule using a properly named user-defined exception, and explain in a sentence or two why this is a textbook case for this concept (versus a pre-defined exception).

---

*Share your answers whenever you're ready. Next up: Lend a Hand on User Defined Exception — an applied practice checkpoint.*
