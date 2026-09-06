# Module 2, Topic 2: Selection Statements (with Lend a Hand)

---

## 1. What Is a Selection Statement?

A **selection statement** lets a program **choose between different paths of execution** based on whether a condition is `TRUE`, `FALSE`, or `NULL`. In PL/SQL, the primary selection construct is `IF`, in its several forms. (`CASE` is a related, distinct selection tool — covered fully in the next topic.)

---

## 2. Why Does This Exist? What Problem Does It Solve?

Without selection statements, a program can only do exactly one thing, unconditionally, every single time it runs — completely useless for real business logic, where almost every rule has the shape *"do X, but only when Y is true; otherwise do something else."* Selection statements are the mechanism that lets code **respond differently to different data**, which is the entire basis of business rule enforcement: eligibility checks, tiered pricing, validation, routing decisions — virtually everything.

---

## 3. The Four Forms of IF

### Form 1 — Simple IF (no ELSE)
```sql
IF condition THEN
    statement(s);
END IF;
```
Executes the statement(s) **only if** `condition` is `TRUE`. If `FALSE` or `NULL`, execution simply skips straight to whatever comes after `END IF;` — nothing happens.

### Form 2 — IF...ELSE
```sql
IF condition THEN
    statement(s);
ELSE
    statement(s);
END IF;
```
Exactly **one** of the two branches always runs. If `condition` is `TRUE`, the `IF` branch runs; if it's `FALSE` **or `NULL`**, the `ELSE` branch runs (this is a critical, often-missed detail — see below).

### Form 3 — IF...ELSIF...ELSE (multi-way branching)
```sql
IF condition1 THEN
    statement(s);
ELSIF condition2 THEN
    statement(s);
ELSIF condition3 THEN
    statement(s);
ELSE
    statement(s);
END IF;
```
Conditions are checked **top to bottom**, and **only the first `TRUE` condition's branch runs** — even if a later condition would also have been `TRUE`. The (optional) final `ELSE` catches anything not matched by any preceding condition.

**Critical spelling note**: it's `ELSIF` (no second "E") — not `ELSEIF`. This trips up virtually everyone coming from other languages at least once.

### Form 4 — Nested IF
```sql
IF condition1 THEN
    IF condition2 THEN
        statement(s);
    ELSE
        statement(s);
    END IF;
ELSE
    statement(s);
END IF;
```
An `IF` statement placed entirely inside another `IF` statement's branch — used when a decision only makes sense/needs to be evaluated **after** an outer condition is already known to be true.

---

## 4. Simple Examples

### Example 1 — Simple IF
```sql
DECLARE
    v_salary NUMBER := 45000;
BEGIN
    IF v_salary < 50000 THEN
        DBMS_OUTPUT.PUT_LINE('Eligible for a raise review.');
    END IF;
END;
/
```

### Example 2 — IF...ELSE
```sql
DECLARE
    v_stock NUMBER := 0;
BEGIN
    IF v_stock > 0 THEN
        DBMS_OUTPUT.PUT_LINE('In stock.');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Out of stock.');
    END IF;
END;
/
```

### Example 3 — IF...ELSIF...ELSE
```sql
DECLARE
    v_score NUMBER := 72;
BEGIN
    IF v_score >= 90 THEN
        DBMS_OUTPUT.PUT_LINE('Grade: A');
    ELSIF v_score >= 75 THEN
        DBMS_OUTPUT.PUT_LINE('Grade: B');
    ELSIF v_score >= 60 THEN
        DBMS_OUTPUT.PUT_LINE('Grade: C');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Grade: F');
    END IF;
END;
/
-- Prints "Grade: C" — even though v_score >= 60 might seem like a "lesser" match,
-- it's the FIRST condition (top-to-bottom) that evaluates to TRUE, so it wins.
```

### Example 4 — Nested IF
```sql
DECLARE
    v_is_employee  VARCHAR2(1) := 'Y';
    v_years_service NUMBER := 6;
BEGIN
    IF v_is_employee = 'Y' THEN
        IF v_years_service >= 5 THEN
            DBMS_OUTPUT.PUT_LINE('Eligible for long-service award.');
        ELSE
            DBMS_OUTPUT.PUT_LINE('Not yet eligible for long-service award.');
        END IF;
    ELSE
        DBMS_OUTPUT.PUT_LINE('Not an employee — no award consideration.');
    END IF;
END;
/
```

---

## 5. Detailed Explanation — The NULL Behavior of IF (Critical)

This connects directly back to Module 1, Topic 5's discussion of three-valued logic (`TRUE`/`FALSE`/`NULL`), and it is genuinely one of the most important, most frequently mis-predicted behaviors in all of PL/SQL:

**An `IF` condition that evaluates to `NULL` is treated exactly like `FALSE`** — the `IF` branch is skipped, and if there's an `ELSE`, the `ELSE` branch runs instead.

```sql
DECLARE
    v_manager_id NUMBER := NULL;
BEGIN
    IF v_manager_id > 100 THEN
        DBMS_OUTPUT.PUT_LINE('Manager ID is greater than 100.');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Manager ID is NOT greater than 100.');  -- this runs
    END IF;
END;
/
```
Here, `v_manager_id > 100` evaluates to `NULL` (since comparing `NULL` to anything yields `NULL`, as you learned in Module 1, Topic 5) — **not** `FALSE`, technically. But PL/SQL's `IF` statement treats **any non-`TRUE`** result (whether genuinely `FALSE` or `NULL`) identically: it skips to `ELSE`. This means the `ELSE` branch prints a message that says "NOT greater than 100" — which is subtly **misleading**, since the honest answer is "we don't know," not "no." This is a real, common source of silently-wrong business logic when `NULL`s aren't explicitly anticipated.

---

## 6. When to Use Which Form

- **Simple IF (no ELSE)**: when there's genuinely nothing to do in the "false" case — e.g., "flag the record only if it meets criteria X," with no alternative action needed.
- **IF...ELSE**: when there are exactly **two** mutually exclusive outcomes.
- **IF...ELSIF...ELSE**: when there are **more than two** mutually exclusive outcomes, checked in a meaningful priority order (first match wins) — very common for tiered logic (grading, discount tiers, risk levels).
- **Nested IF**: when a decision is only relevant/meaningful **after** confirming an outer condition — but be cautious: deeply nested `IF`s (3+ levels) hurt readability significantly, and often a well-chosen `ELSIF` chain or a `CASE` statement (next topic) expresses the same logic more clearly.

---

## 7. Common Mistakes & Misconceptions

1. **Mistake**: Misspelling `ELSIF` as `ELSEIF` → compile error; this is one of the single most common syntax mistakes for anyone coming from other languages (Python, JavaScript, etc. use `elif`/`else if`).
2. **Misconception**: "If none of my ELSIF conditions are true and there's no ELSE, something goes wrong." → Nothing goes wrong — execution simply skips the entire `IF` structure and continues with whatever comes after `END IF;`, exactly like a simple `IF` with no match.
3. **Mistake**: Assuming `IF v_x > 100 THEN` behaves differently when `v_x` is `NULL` versus genuinely `FALSE` — from the `IF` statement's perspective, they are **treated identically** (both skip to `ELSE`/skip entirely) — even though conceptually "unknown" and "false" are different ideas.
4. **Mistake**: Writing an overly-deep nested `IF` structure when a flatter `ELSIF` chain (or `CASE`) would express the same logic far more readably.
5. **Misconception**: "ELSIF conditions are all evaluated, and the 'best' match is chosen." → False — evaluation stops at the **first** `TRUE` condition, in top-to-bottom order; order matters enormously, and putting conditions in the wrong order can produce subtly wrong results (see Exercise 5 below).

---

## 8. Edge Cases to Be Aware Of

- If **multiple** `ELSIF` conditions would independently evaluate to `TRUE`, only the **first** one (in written order) actually executes — later ones are never even evaluated. This means condition **order is a real design decision**, not arbitrary.
- A simple `IF` (no `ELSE`) with a `NULL` condition behaves identically to one with a `FALSE` condition — silently does nothing, which can be surprising if you expected `NULL` to be specially flagged somehow.
- Nested `IF`s inside an `ELSIF` branch are completely legal and common — you're not limited to nesting only inside simple `IF`/`ELSE`.

---

## 9. Interview-Level / Practical Notes

- *"What happens if an IF condition evaluates to NULL?"* — Treated the same as `FALSE`; the `ELSE` (or nothing, if no `ELSE`) runs. This specific question is extremely commonly asked because it's so easy to get wrong.
- *"In an IF...ELSIF chain, if two conditions could both be true, which one executes?"* — Only the **first** one, in written order — the rest are never evaluated.
- Being able to spot when nested `IF`s have become too deep and should be refactored into an `ELSIF` chain or `CASE` statement is a practical code-quality signal that goes beyond just knowing the syntax.

---

## Things You Must Remember

- Four forms: simple `IF`, `IF...ELSE`, `IF...ELSIF...ELSE`, and nested `IF`.
- Spelling: `ELSIF` — **not** `ELSEIF`.
- In an `ELSIF` chain, only the **first** `TRUE` condition's branch runs, top to bottom — order matters.
- **A `NULL` condition is treated exactly like `FALSE`** by `IF` — skips to `ELSE` (or skips entirely if there's no `ELSE`) — this is one of the most important NULL-related facts in all of PL/SQL.
- Deep nesting hurts readability — prefer a flatter `ELSIF` chain or `CASE` when logic allows it.

## How to Recognize This Concept

Reach for **selection statements** whenever a requirement contains:
- "**if**... **then**... **otherwise**..." (classic two-way).
- "**depending on** whether..." with **more than two** distinct outcomes, checked in a **priority order** (tiered logic) → `ELSIF` chain.
- A decision that's only relevant **after confirming** another condition first ("if the employee is active, *then* check if they're eligible for X") → nested `IF`.

If instead the decision is based on **matching a single variable against many discrete, specific values** (rather than ranges/conditions), that's often a stronger signal for `CASE` — covered next.

---

## Lend a Hand — Applied Practice

### Practice 1 — Discount Tier
Write a block that, given an order amount, prints a discount tier: `'PLATINUM'` for orders over 100,000, `'GOLD'` for orders over 50,000, `'SILVER'` for orders over 10,000, and `'STANDARD'` otherwise. Use the appropriate `IF` form, and be deliberate about condition order.

### Practice 2 — NULL-Aware Eligibility Check
Write a block that checks if an employee (with a `commission_pct` that might be `NULL` for salaried staff) qualifies for a "commission bonus," where the rule is: qualify if `commission_pct > 5`. Demonstrate what happens when `commission_pct` is `NULL`, and then rewrite the check so it explicitly and correctly handles the `NULL` case with its own clear message (rather than silently falling into a generic `ELSE`).

### Practice 3 — Nested Decision
Business rule: *"If a customer is a returning customer, check whether their total lifetime spend exceeds 10,000 — if so, they get free express shipping; if not, they get free standard shipping. New customers never get free shipping of any kind on their first order."* Write this using a nested `IF` structure.

### Practice 4 — Spot the Bug
```sql
DECLARE
    v_age NUMBER := 17;
BEGIN
    IF v_age >= 13 THEN
        DBMS_OUTPUT.PUT_LINE('Teenager');
    ELSIF v_age >= 18 THEN
        DBMS_OUTPUT.PUT_LINE('Adult');
    ELSIF v_age < 13 THEN
        DBMS_OUTPUT.PUT_LINE('Child');
    END IF;
END;
/
```
This is meant to correctly classify ages, but has a logic ordering bug. Identify it, explain why it produces a misleading (though not crashing) result, and fix it.

### Practice 5 — Realistic Business Scenario
*"A loan approval script needs to check an applicant's credit score. Above 750: auto-approve. Between 600 and 750 (inclusive of both bounds): route to manual review. Below 600: auto-reject. If the credit score is missing entirely (not yet retrieved from the bureau), this must be treated as its own distinct case — 'pending data' — not silently lumped into auto-reject."* Write this as a properly ordered `IF...ELSIF...ELSE` structure, paying close attention to the NULL case and condition ordering.

---

*Share your attempts whenever you're ready. Next up: Module 2, Topic 3 — CASE Statement.*
