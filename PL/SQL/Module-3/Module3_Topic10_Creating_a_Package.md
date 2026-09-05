# Module 3, Topic 10: Creating a Package Using Procedures and Functions

> This topic is **applied and integrative** rather than introducing brand-new syntax. Topic 8 taught you package *structure* (spec/body). Topic 9 taught you *overloading*. This topic is about the **process** of designing and building a real, cohesive package from a business requirement — combining multiple procedures and functions, deciding what's public vs. private, and making the pieces work together correctly. This is exactly the skill real PL/SQL developers use daily.

---

## 1. What This Topic Is Really About

Up to now, you've built packages with one or two members, mostly to demonstrate isolated syntax. In real work, a package is a **cohesive module** — a set of operations that belong together because they operate on the same business entity or process, often **calling each other internally**, sharing **private helper logic**, and sometimes sharing **package-level state**.

The skill this topic builds is: **given a business domain, design the right set of package members — deciding what's public, what's private, what calls what — before writing a single line of implementation.**

---

## 2. The Design Process (Follow This Every Time)

When asked to "build a package for X," an experienced developer doesn't start typing `CREATE PACKAGE` immediately. They think through this sequence:

1. **What is the business domain/entity this package represents?** (e.g., "Employee Management," "Order Processing," "Loyalty Program")
2. **What operations does the outside world (other applications, other PL/SQL code) actually need to perform?** — These become your **public** procedures/functions, declared in the spec.
3. **What calculations or checks are internal implementation details that support those operations, but nobody outside should call directly?** — These become **private** members, living only in the body.
4. **Do any operations need to share state across a session** (e.g., a running counter, a cached lookup, a "current context")? — If so, that's a **package-level variable**.
5. **Do any operations naturally overload** (same conceptual action, different input shapes)? — Apply Topic 9's judgment here.
6. **What order do things need to be defined in the body?** — In PL/SQL, by default, a sub program can only call another sub program that's **already been defined earlier** in the same body (top-to-bottom). If two private helpers need to call each other in both directions, you need a **forward declaration** (a private declaration at the top of the body, similar in spirit to a spec declaration, but scoped to the body only) — this is a subtlety worth knowing about, even though it's uncommon in simple packages.
7. **Only now** — write the specification, then the body.

---

## 3. Worked Example: Building `pkg_order_processing` End to End

**Business requirement:** *"We need a module that handles finalizing customer orders. It should: calculate the order's discount based on customer tier, apply that discount and compute the final total, record the finalized order, and let other systems check whether a given order has already been finalized. Internally, tier-based discount rates are a simple lookup that shouldn't be exposed as a general-purpose utility — it's specific to this process."*

### Step 1 — Identify the business domain
Order finalization / order processing.

### Step 2 — Identify public operations
Reading the requirement carefully:
- "calculate the order's discount... apply that discount and compute the final total, record the finalized order" → this whole flow sounds like **one orchestrating action** → a **procedure**: `finalize_order`.
- "let other systems check whether a given order has already been finalized" → a **yes/no check** → a **function**: `is_order_finalized`.

### Step 3 — Identify private helpers
- "tier-based discount rates are a simple lookup that shouldn't be exposed" → explicitly called out as internal-only → a **private function**: `get_tier_discount`.

### Step 4 — Package-level state?
Nothing in the requirement suggests shared session state — skip this.

### Step 5 — Overloading?
Nothing suggests multiple input shapes for the same operation — skip this.

### Step 6 — Define order
`get_tier_discount` is used by `finalize_order`, so it must be defined **before** `finalize_order` in the body (or forward-declared). `is_order_finalized` is independent.

### Step 7 — Build it

**Specification (public contract only):**
```sql
CREATE OR REPLACE PACKAGE pkg_order_processing
IS
    PROCEDURE finalize_order (p_order_id IN NUMBER);
    FUNCTION is_order_finalized (p_order_id IN NUMBER) RETURN VARCHAR2;
END pkg_order_processing;
/
```

**Body (implementation, including the private helper):**
```sql
CREATE OR REPLACE PACKAGE BODY pkg_order_processing
IS
    -- PRIVATE helper: not declared in the spec, invisible to outside callers
    FUNCTION get_tier_discount (p_customer_id IN NUMBER) RETURN NUMBER
    IS
        v_tier VARCHAR2(20);
    BEGIN
        SELECT tier INTO v_tier FROM customers WHERE customer_id = p_customer_id;

        IF v_tier = 'GOLD' THEN
            RETURN 15;
        ELSIF v_tier = 'SILVER' THEN
            RETURN 10;
        ELSE
            RETURN 0;
        END IF;
    END get_tier_discount;

    -- PUBLIC procedure: declared in the spec
    PROCEDURE finalize_order (p_order_id IN NUMBER)
    IS
        v_customer_id   NUMBER;
        v_order_amount  NUMBER;
        v_discount_pct  NUMBER;
        v_final_amount  NUMBER;
    BEGIN
        SELECT customer_id, order_amount INTO v_customer_id, v_order_amount
        FROM orders
        WHERE order_id = p_order_id;

        v_discount_pct := get_tier_discount(v_customer_id);  -- calling the private helper
        v_final_amount := v_order_amount - (v_order_amount * v_discount_pct / 100);

        UPDATE orders
        SET final_amount = v_final_amount,
            status = 'FINALIZED'
        WHERE order_id = p_order_id;
    END finalize_order;

    -- PUBLIC function: declared in the spec
    FUNCTION is_order_finalized (p_order_id IN NUMBER) RETURN VARCHAR2
    IS
        v_status VARCHAR2(20);
    BEGIN
        SELECT status INTO v_status FROM orders WHERE order_id = p_order_id;

        IF v_status = 'FINALIZED' THEN
            RETURN 'Y';
        ELSE
            RETURN 'N';
        END IF;
    END is_order_finalized;

END pkg_order_processing;
/
```

**Notice the design decisions made and why:**
- `get_tier_discount` is private — exactly matching the requirement's explicit statement that it "shouldn't be exposed."
- `finalize_order` internally calls `get_tier_discount` **without any package-name qualifier**, because they're in the **same package body** — qualification is only required from **outside** the package.
- `finalize_order` is a procedure (it performs an action — updates data) even though it *contains* a calculation step; that calculation step is delegated to a function, but the overall orchestrating operation remains a procedure — exactly the "procedures calling functions internally" pattern from Topic 4.
- `is_order_finalized` is a function (single computed answer, naturally something another system might want to check inline, e.g., in a report `WHERE` clause).

---

## 4. Detailed Explanation — What Makes This "Good" Package Design

- **Minimal public surface**: only what callers genuinely need is public. Everything else is private. This isn't just tidiness — it means you can freely change `get_tier_discount`'s internal logic later (e.g., adding a new tier) without any risk of breaking external callers, because they never had access to it directly in the first place.
- **Single Responsibility per member**: `get_tier_discount` does exactly one thing (look up a rate). `finalize_order` orchestrates. `is_order_finalized` checks status. Nobody is trying to do too much in one place.
- **Reuse within the package**: private helpers exist specifically to be reused *within* the package by multiple public members, if needed — even though this example only uses `get_tier_discount` once, in a larger real package it might be called by several public procedures.

---

## 5. Common Mistakes & Misconceptions (Specific to This Applied Skill)

1. **Mistake**: Making everything public "just in case it's useful later." This defeats the purpose of encapsulation and increases the risk surface for external misuse — only expose what's genuinely needed by callers now.
2. **Mistake**: Defining a private helper **after** the public member that calls it, without a forward declaration → compile error, because PL/SQL package bodies resolve top-to-bottom by default.
3. **Misconception**: "I need to write `pkg_order_processing.get_tier_discount(...)` even when calling it from inside the same package body." → Not true — qualification with the package name is only required when calling from **outside** the package. Inside the same body, you call other members directly by name.
4. **Mistake**: Cramming an entire multi-step business process into a single giant procedure instead of breaking it into a public orchestrator plus private helper steps — this hurts readability, testability, and reuse.
5. **Misconception**: "A package must have package-level variables or it's not 'really' using packages properly." → False — many perfectly well-designed packages have no package-level state at all; state should only be added when the requirement genuinely calls for shared session-level data (as covered in Topic 8).

---

## 6. When to Combine Multiple Procedures/Functions Into One Package vs. Splitting Into Several

- **One package** when the operations all belong to the **same business entity/process** and would naturally be looked for together by a developer exploring the schema (e.g., everything about order finalization).
- **Separate packages** when operations belong to **genuinely distinct business domains**, even if they call each other occasionally (e.g., `pkg_order_processing` might call a function from a completely separate `pkg_customer_lookup` package — that's normal and expected; packages calling into other packages is a standard pattern, not something to avoid).

---

## 7. Interview-Level / Practical Notes

- A common practical/interview exercise is exactly this: *"Given this business requirement, design a package — tell me what's public, what's private, and why."* This tests judgment, not just syntax recall.
- Being able to explain **why** something is private (not just declaring it private) is a strong signal of real understanding — e.g., "this is private because it's an internal calculation step that no external system should depend on directly, since we want the freedom to change the tier logic later without a contract-breaking change."
- Real code reviews at companies frequently flag "everything is public" packages as a design smell — expect this exact scrutiny in a real job.

---

## Things You Must Remember

- Design **before** you code: identify the domain, then classify each operation as public (external contract) or private (internal detail).
- Calls **within the same package body** don't need the package-name qualifier; calls from **outside** the package always do.
- PL/SQL resolves package body members **top-to-bottom** by default — a helper must be defined before (or forward-declared before) something that calls it.
- Procedures commonly **orchestrate** and call private/public functions internally for calculations — this is the standard, expected pattern, not a special case.
- Packages calling other packages is normal — you don't need to cram unrelated business domains into one giant package.
- Minimal public surface = better long-term maintainability, since private internals can change freely without breaking external callers.

## How to Recognize This Concept

This "topic" is really about recognizing, when given **any multi-step business requirement**, how to decompose it:
- Look for **distinct actions/questions** in the requirement — each one is a candidate package member.
- Look for phrases like "**internally**," "**shouldn't be exposed**," "**just used to support**..." — these are explicit private-member signals.
- Look for a **verb-heavy orchestrating step** ("finalize," "process," "complete") combined with **smaller calculation/check steps** feeding into it — that's your public-procedure-calls-private-helpers pattern.
- Look for **"check whether..."** style requirements — usually a small public function, often independent of the main orchestrating procedure.

---

## Exercises

**Business requirement for all exercises:** *"We need a module for managing employee leave requests. It should: submit a new leave request (validating that the employee has enough leave balance remaining before inserting the request), approve a pending leave request (which should also deduct the approved days from the employee's leave balance), and provide a way for other systems to check an employee's remaining leave balance. The balance-checking logic used internally during submission should be reusable, since both submission and the balance-check operation need it — but the actual balance deduction math used only during approval is a private implementation detail specific to that step."*

1. **(Design phase — do this before writing any code)** Following the 7-step design process from this topic, identify: the business domain, the public members (name + procedure or function + rough parameters), and the private members. Write this out as a short design note before touching syntax.

2. **(Specification)** Based on your design in Exercise 1, write the full package **specification** for `pkg_leave_management`.

3. **(Body)** Write the full package **body**, implementing all public and private members with reasonable (illustrative) logic. Pay attention to definition order — make sure anything private that's called by a public member is defined before it (or explain if you'd need a forward declaration).

4. **(Design justification)** In 3–4 sentences, explain why the balance-checking logic is public/reusable while the deduction math is private, tying your reasoning directly back to the requirement's own wording.

5. **(Spot the flaw)** A colleague's version of this package makes `get_leave_balance` **private**, arguing "it's only used internally by submission and approval." But the requirement explicitly says *"provide a way for other systems to check an employee's remaining leave balance."* Explain what's wrong with the colleague's design choice.

6. **(Cross-package interaction)** Suppose `pkg_leave_management`'s approval logic needs to check whether the employee is still active, and there's already a separate, unrelated package `pkg_employee_mgmt` with a function `is_active_employee(p_emp_id)`. Should you duplicate that check inside `pkg_leave_management`, or call `pkg_employee_mgmt.is_active_employee(...)` from within your approval procedure? Justify your answer.

---

*Share your design notes and code whenever you're ready — this is a great one to actually attempt fully, since it mirrors real work closely. Next up: Packages (the final, more advanced topic in this module).*
