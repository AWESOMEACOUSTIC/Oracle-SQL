# Module 3, Topic 8: Package Structure

---

## 1. What Is a Package?

A **package** is a schema object that **groups logically related procedures, functions, variables, constants, cursors, and exceptions into a single named unit**. It is not a sub program itself — it's a **container**.

A package always has **two separate parts**, created as two separate objects:

1. **Package Specification** (the "spec" or "header") — the **public interface**. It declares *what* is available to callers: the signatures of procedures/functions, and any public variables/constants/types — but **no implementation code**.
2. **Package Body** — the **implementation**. It contains the actual code for everything declared in the spec, and can also contain **private** elements (procedures, functions, variables) that exist only for internal use within the package, invisible to anyone outside it.

This spec/body split is the single most important structural idea in this topic — everything else builds on it.

---

## 2. Why Do Packages Exist? What Problem Do They Solve?

By the time you're writing more than a handful of procedures and functions for a business domain (say, everything related to "Employee Management": `create_employee`, `update_salary`, `get_full_name`, `is_eligible_for_bonus`, `terminate_employee`...), standalone objects create real problems:

- **Namespace clutter** — dozens of independently-named objects floating in the schema, with no grouping or clear relationship, makes the schema hard to navigate and understand.
- **No privacy/encapsulation** — every standalone procedure/function is fully public by default; there's no clean way to have "helper" logic that only your own related procedures should use internally, hidden from external callers.
- **No shared, persistent state across calls in a session** — standalone sub programs have no natural place to store session-level data that multiple related procedures need to share (e.g., a "current user context" used by several procedures in a workflow).
- **Weaker organization for large systems** — real companies often have hundreds of procedures/functions; without packages, there's no way to say "these 15 objects all belong to the Employee Management module" as a first-class database concept.

Packages solve all of this: they give you **grouping, encapsulation (public vs. private), and shared package-level state** — the same core ideas as classes/modules in general-purpose programming languages, adapted to PL/SQL.

---

## 3. Why Is It Used? (The Business Case)

- **Organizing business logic by domain**: a company might have `pkg_employee_mgmt`, `pkg_order_processing`, `pkg_billing` — each package bundling everything related to that business area, making it obvious where to find (and add) relevant logic.
- **Hiding implementation details**: internal validation helpers, internal calculation steps, or internal state don't need to be exposed to every calling application — only the intended public operations are visible, reducing the surface area for misuse and making it safer to refactor internals later without breaking callers.
- **Performance**: the first call to *any* object in a package loads the **entire package** into memory (SGA) — subsequent calls to other objects in the same package don't need to be separately loaded, which can improve performance for related, frequently-used logic.
- **Overloading** (covered as its own topic soon) is only fully practical within packages — grouping multiple versions of a same-named procedure/function together.

---

## 4. Syntax: Package Specification

```sql
CREATE [OR REPLACE] PACKAGE package_name
IS  -- (or AS, interchangeable)
    -- public constants/variables
    -- public type declarations
    -- public procedure/function signatures (declaration only, no body)

    PROCEDURE procedure_name (parameter_list);
    FUNCTION function_name (parameter_list) RETURN datatype;

END package_name;
/
```

### Syntax: Package Body

```sql
CREATE [OR REPLACE] PACKAGE BODY package_name
IS
    -- private variables/constants (not visible outside the package)
    -- private procedures/functions (helper logic, internal use only)

    PROCEDURE procedure_name (parameter_list)
    IS
    BEGIN
        -- full implementation
    END procedure_name;

    FUNCTION function_name (parameter_list) RETURN datatype
    IS
    BEGIN
        -- full implementation
        RETURN expression;
    END function_name;

    -- Optional: initialization section
BEGIN
    -- runs once, the FIRST time the package is referenced in a session
END package_name;
/
```

### Syntax Breakdown

| Element | Meaning |
|---|---|
| **Package Specification** | Declares the **public contract** — what's visible to the outside world. Every procedure/function listed here **must** have a matching, fully-implemented counterpart in the body. |
| **Package Body** | Contains the **actual code**. Can include additional private procedures/functions/variables **not declared in the spec** — these are invisible outside the package, only callable by other code *within* the same package body. |
| **Optional initialization section** (`BEGIN...END` at the very end of the body, without a name) | Runs **once per session**, the first time anything in the package is accessed — commonly used to initialize package-level variables. |
| **Calling package objects** | `package_name.procedure_name(...)` or `package_name.function_name(...)` — always qualified with the package name using dot notation. |

---

## 5. Simple Examples

### Example 1 — A basic package: spec and body

**Specification:**
```sql
CREATE OR REPLACE PACKAGE pkg_employee_mgmt
IS
    PROCEDURE give_raise (p_emp_id IN NUMBER, p_raise_pct IN NUMBER);
    FUNCTION get_full_name (p_emp_id IN NUMBER) RETURN VARCHAR2;
END pkg_employee_mgmt;
/
```

**Body:**
```sql
CREATE OR REPLACE PACKAGE BODY pkg_employee_mgmt
IS
    PROCEDURE give_raise (p_emp_id IN NUMBER, p_raise_pct IN NUMBER)
    IS
    BEGIN
        UPDATE employees
        SET salary = salary + (salary * p_raise_pct / 100)
        WHERE employee_id = p_emp_id;
    END give_raise;

    FUNCTION get_full_name (p_emp_id IN NUMBER) RETURN VARCHAR2
    IS
        v_name VARCHAR2(100);
    BEGIN
        SELECT first_name || ' ' || last_name INTO v_name
        FROM employees
        WHERE employee_id = p_emp_id;

        RETURN v_name;
    END get_full_name;
END pkg_employee_mgmt;
/
```

**Calling it:**
```sql
BEGIN
    pkg_employee_mgmt.give_raise(100, 10);
    DBMS_OUTPUT.PUT_LINE(pkg_employee_mgmt.get_full_name(100));
END;
/
```

### Example 2 — Private helper (in body only, not in spec)

```sql
CREATE OR REPLACE PACKAGE pkg_payroll
IS
    FUNCTION calculate_net_pay (p_emp_id IN NUMBER) RETURN NUMBER;
END pkg_payroll;
/

CREATE OR REPLACE PACKAGE BODY pkg_payroll
IS
    -- PRIVATE helper function — not in the spec, so callers outside this package cannot see or call it
    FUNCTION calculate_tax (p_gross NUMBER) RETURN NUMBER
    IS
    BEGIN
        RETURN p_gross * 0.2;
    END calculate_tax;

    -- PUBLIC function — declared in the spec
    FUNCTION calculate_net_pay (p_emp_id IN NUMBER) RETURN NUMBER
    IS
        v_gross NUMBER;
    BEGIN
        SELECT salary INTO v_gross FROM employees WHERE employee_id = p_emp_id;
        RETURN v_gross - calculate_tax(v_gross);  -- calling the private helper internally
    END calculate_net_pay;
END pkg_payroll;
/
```

A caller can do `pkg_payroll.calculate_net_pay(100)` — but **cannot** do `pkg_payroll.calculate_tax(...)` from outside; that raises a compile error, because `calculate_tax` was never declared in the package specification.

### Example 3 — Package-level state (public variable) and initialization section

```sql
CREATE OR REPLACE PACKAGE pkg_session_context
IS
    g_current_user VARCHAR2(50);  -- public package variable
    PROCEDURE set_user (p_user VARCHAR2);
END pkg_session_context;
/

CREATE OR REPLACE PACKAGE BODY pkg_session_context
IS
    PROCEDURE set_user (p_user VARCHAR2)
    IS
    BEGIN
        g_current_user := p_user;
    END set_user;
BEGIN
    -- initialization section: runs once per session, first time this package is touched
    g_current_user := 'UNSET';
END pkg_session_context;
/
```

Every procedure/function within the *same session* that references `pkg_session_context.g_current_user` sees the **same, shared value** — this is genuine package-level state, persisting for the life of the session (not just one call), which standalone procedures/functions have no equivalent for.

---

## 6. Detailed Explanation

- **The spec and body are compiled and stored as two separate objects** in the data dictionary, even though conceptually you think of them as "one package." You can query `USER_OBJECTS` and see both `PACKAGE` and `PACKAGE BODY` listed separately for the same name.
- **You can create/change the spec without immediately having a matching body** — Oracle allows an "invalid" body-less state temporarily during development, but nothing in the package is *usable* until the body is compiled successfully with matching signatures for everything declared in the spec.
- **Dependency and recompilation**: if you change the package **specification** (e.g., add a new public procedure, or change a parameter), any code that depends on the package may be invalidated and need recompilation. Changing only the **body** (the implementation, keeping the same public signatures) does **not** invalidate dependent objects — this is a genuinely important design benefit: you can improve/fix internal logic freely without forcing every calling application to recompile, as long as the public contract (the spec) doesn't change.
- **Overloading** (next topic) becomes natural inside a package body — you can have multiple procedures/functions with the same name but different parameter signatures, something standalone objects can't do at the schema level (two standalone objects can't share a name at all).

---

## 7. When to Use / When Not to Use a Package

**Use a package when:**
- You have **multiple related** procedures/functions that logically belong together (a business domain, a module).
- You need **private helper logic** that shouldn't be exposed/callable from outside.
- You need **package-level (session-persistent) state** shared across multiple calls within a session.
- You want the **stability benefit**: ability to change internal implementation (the body) without forcing dependent code to recompile, as long as the public interface (spec) stays the same.

**Don't force it when:**
- You genuinely have a **single, standalone, unrelated utility** function/procedure with no natural grouping — wrapping every single object in its own one-object package adds ceremony without benefit. (In practice, though, most real systems still end up grouping almost everything into packages by convention, once there's more than a trivial handful of objects.)

---

## 8. Common Mistakes & Misconceptions

1. **Mistake**: Declaring a procedure/function in the spec but forgetting (or mismatching parameters/types) to implement it correctly in the body → compile error on the body; the package becomes unusable until fixed.
2. **Misconception**: "The package body's private procedures can be seen if you know their name and try to call them directly." → No — Oracle enforces this at the object/privilege level; if it's not in the spec, it is genuinely inaccessible from outside, not just "hidden by convention."
3. **Mistake**: Forgetting the required calling syntax `package_name.object_name(...)` — omitting the package name qualifier when calling from outside the package (it's mandatory from outside; only *within* the same package body can you call other members without the qualifier).
4. **Misconception**: "Changing anything in a package always invalidates all dependent code." → Only changing the **spec** does that; changing just the body (same public signatures) does not force dependents to recompile — an important distinction many learners miss.
5. **Mistake**: Assuming package-level variables (like `g_current_user` in Example 3) are shared **across different sessions**. They are not — package state in this form is **per-session**; two different users/sessions each get their own independent copy of package-level variables (this is different from the Function Result Cache's SGA-wide sharing you learned earlier — worth noticing the contrast).

---

## 9. Edge Cases to Be Aware Of

- If the package specification has **no body-required elements** (e.g., only constants, no procedures/functions), a package body isn't strictly required. But the moment you declare even one procedure/function signature in the spec, a matching body becomes mandatory.
- The **initialization section** in the body runs **exactly once per session**, the first time *any* element of the package is referenced — not once per call. This is a common point of confusion; people sometimes expect it to run every time, which it does not.
- Package-level variables **persist for the life of the session**, not just one call — meaning state set in one procedure call can be read by a *later, separate* call within the same session, which is a powerful but also potentially dangerous capability if misused (state leaking between logically unrelated operations if you're not careful).

---

## 10. Interview-Level / Practical Notes

- Extremely common interview question: *"What's the difference between a package spec and a package body?"* — Spec = public interface (declarations only); Body = implementation, plus optional private members.
- *"Why would you choose a package over standalone procedures/functions?"* — Grouping/organization, encapsulation (public/private), shared session state, and the ability to change implementation without breaking callers (as long as the spec is stable).
- *"Does changing a package body always cause recompilation of dependent objects?"* — No, this is a nuanced and frequently-tested distinction: only spec changes typically do.
- In real companies, it is extremely rare to find large PL/SQL codebases *without* heavy package usage — standalone procedures/functions tend to be reserved for small utilities or quick scripts, while all serious business logic modules are packaged.

---

## Things You Must Remember

- A package = **specification** (public contract, declarations only) + **body** (implementation, can include private members).
- Anything declared in the spec **must** have a matching implementation in the body.
- Private procedures/functions/variables exist **only in the body**, never in the spec — genuinely inaccessible from outside.
- Call syntax from outside the package: always `package_name.member_name(...)`.
- Changing the **body only** (same public signatures) does **not** invalidate dependent objects; changing the **spec** typically does.
- The optional initialization section runs **once per session**, not once per call.
- Package-level variables are **shared across calls within a session**, but **not shared across different sessions**.

## How to Recognize This Concept

Think **package** when a requirement or situation involves:
- "**Several related** operations for [some business area]" — a strong signal to group them.
- A need for **internal helper logic** that shouldn't be exposed to callers — "this calculation step is just an implementation detail."
- A need for **state that persists across multiple calls** within the same user session — "remember something between these related operations."
- A system that's clearly growing beyond a handful of standalone, unrelated procedures/functions — organizational scale is itself a signal.

If you're dealing with a single, genuinely isolated utility with no related siblings and no need for privacy or shared state, a standalone procedure/function remains perfectly reasonable — packages aren't mandatory for everything, just strongly preferred once there's real grouping or encapsulation value.

---

## Exercises

1. **(Basic package)** Design a package `pkg_inventory` with a public function `get_stock_level(p_product_id)` returning `NUMBER`, and a public procedure `restock(p_product_id, p_quantity)`. Write both the specification and the body (implementation logic can be simple/illustrative).

2. **(Private helper)** Extend the `pkg_inventory` package body from Exercise 1 with a **private** function `is_valid_product(p_product_id)` returning `VARCHAR2('Y'/'N')`, used internally by `restock` to check validity before updating. Make sure it is *not* exposed in the spec.

3. **(Predict the error)** What's wrong with this package, and why won't it compile?
   ```sql
   CREATE OR REPLACE PACKAGE pkg_test
   IS
       PROCEDURE do_something (p_val NUMBER);
       FUNCTION get_value RETURN NUMBER;
   END pkg_test;
   /

   CREATE OR REPLACE PACKAGE BODY pkg_test
   IS
       PROCEDURE do_something (p_val NUMBER)
       IS
       BEGIN
           NULL;
       END do_something;
   END pkg_test;
   /
   ```

4. **(Spec vs. body change reasoning)** You have a package `pkg_billing` used by 10 different applications. You need to fix a bug inside `calculate_late_fee`'s internal logic — the fix does **not** change the function's parameters or return type. Do the 10 calling applications need to be recompiled or redeployed because of this change? Explain why or why not, referencing the spec/body distinction.

5. **(Package-level state)** Design a package `pkg_batch_job_tracker` with a public package-level variable `g_records_processed` (`NUMBER`), a procedure `increment_counter`, and a function `get_total_processed` returning the current count. Explain, in your own words, what would happen to `g_records_processed` if two **different sessions** both called `increment_counter` several times — would they see each other's updates? Why or why not?

6. **(Realistic business scenario)** Business requirement: *"We need to organize all our customer-facing loyalty program logic — calculating tier, awarding points, checking redemption eligibility, and a few internal helper calculations that support these — into one cohesive, well-encapsulated module that other application teams can call without needing to understand the internal calculation details."* Design the **package specification only** (no body needed) for `pkg_loyalty_program`, including at least 3 public operations, and briefly note (in comments or prose) what you'd expect to be private in the body and why.

---

*Share your attempts whenever you're ready. Next up: Overloading Procedure and Function.*
