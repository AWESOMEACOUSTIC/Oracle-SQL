# Module 3, Topic 7: Sub Program Parameter Using Mixed and Named Notations

---

## 1. What Is This Concept?

When you call a procedure or function, you must supply values for its parameters. Oracle gives you **three different notations** for doing this:

1. **Positional notation** — the style you've been using so far: values are matched to parameters purely by **order**.
2. **Named notation** — values are matched to parameters by **explicitly naming** which parameter each value belongs to, using `=>`.
3. **Mixed notation** — a combination: some leading arguments are positional, and the rest are named.

This topic is about understanding **why** the latter two exist and **when** each is the right choice — not just their syntax.

---

## 2. Why Does This Exist? What Problem Does It Solve?

Consider a procedure with many parameters, several of which have defaults:

```sql
CREATE OR REPLACE PROCEDURE create_employee
    (p_first_name    IN VARCHAR2,
     p_last_name     IN VARCHAR2,
     p_department_id IN NUMBER,
     p_salary        IN NUMBER DEFAULT 30000,
     p_is_manager    IN VARCHAR2 DEFAULT 'N',
     p_hire_date     IN DATE DEFAULT SYSDATE)
IS
BEGIN
    ...
END;
```

Now suppose a caller wants to create an employee with a **specific hire date**, but wants everything else (salary, is_manager) to just use the defaults. With **pure positional notation**, this is a real problem:

```sql
create_employee('Arun', 'Kumar', 10, 30000, 'N', DATE '2026-01-15');
```

To supply the 6th parameter, you're **forced to re-type the default values for parameters 4 and 5** even though you didn't want to change them — just to "hold their place" positionally. This is:
- **Error-prone** — if the default salary changes later, every caller that "re-typed" `30000` just to skip it is now silently out of sync with the new default.
- **Unreadable** — a call like `create_employee('Arun', 'Kumar', 10, 30000, 'N', DATE '2026-01-15')` gives no clue what each value *means* without going and checking the procedure's signature.
- **Fragile** — if the procedure's parameter order ever changes, every positional call site silently breaks (or worse, silently passes wrong values into wrong parameters, since positional matching doesn't complain about "wrong" values, only about count/type mismatches).

**Named notation solves all three problems at once**: you specify exactly which parameters you're supplying, by name, in any order, and skip any parameter that has a default you're happy with.

```sql
create_employee(
    p_first_name => 'Arun',
    p_last_name  => 'Kumar',
    p_department_id => 10,
    p_hire_date  => DATE '2026-01-15'
);
```

This reads clearly, doesn't force you to repeat default values, and stays correct even if the parameter order in the procedure definition changes later.

---

## 3. Why Is It Used? (The Business Case)

- **Skipping optional/default parameters cleanly** — extremely common in real procedures with many configurable options, where most calls only care about a few of them.
- **Self-documenting calls** — in a large codebase, a call like `apply_discount(p_customer_id => 501, p_discount_pct => 15)` is instantly understandable months later, versus `apply_discount(501, 15)`, which requires looking up the signature to know what `501` and `15` even mean.
- **Resilience to signature changes** — if a new parameter is added to a procedure **with a default value** (a very common, backward-compatible way to extend an existing procedure), every existing named-notation call site continues to work unchanged. Positional call sites might break or silently pass values into the wrong slot if the new parameter is inserted anywhere but the very end.

---

## 4. Syntax

### Positional Notation (baseline — for contrast)
```sql
procedure_name(value1, value2, value3);
```
Values are matched to parameters **strictly in declared order**.

### Named Notation
```sql
procedure_name(param_name1 => value1, param_name2 => value2, ...);
```
- `=>` is the "association operator" — read it as "gets the value of."
- Order **does not matter** in named notation — you can supply them in any sequence.
- You can **omit** any parameter that has a `DEFAULT` value defined.

### Mixed Notation
```sql
procedure_name(value1, value2, param_name3 => value3, ...);
```
**Critical rule**: Once you switch to named notation partway through a call, **every parameter after that point must also be named**. You cannot go back to positional after using a named argument.

```sql
-- VALID mixed notation: positional first, then named
create_employee('Arun', 'Kumar', p_department_id => 10, p_hire_date => DATE '2026-01-15');

-- INVALID: cannot go back to positional after a named argument
create_employee('Arun', p_last_name => 'Kumar', 10, ...);  -- ERROR
```

---

## 5. Simple Examples

Using the `create_employee` procedure defined above:

### Example 1 — Pure positional (all values supplied, in order)
```sql
BEGIN
    create_employee('Neha', 'Sharma', 20, 45000, 'Y', DATE '2025-06-01');
END;
/
```

### Example 2 — Pure named (skip defaults freely, any order)
```sql
BEGIN
    create_employee(
        p_department_id => 20,
        p_first_name    => 'Neha',
        p_last_name     => 'Sharma'
    );
    -- p_salary, p_is_manager, p_hire_date all use their DEFAULT values
END;
/
```

### Example 3 — Mixed notation
```sql
BEGIN
    create_employee('Neha', 'Sharma', p_department_id => 20, p_is_manager => 'Y');
    -- p_salary and p_hire_date use defaults; p_department_id and p_is_manager named explicitly
END;
/
```

### Example 4 — Using named notation to skip a middle parameter
```sql
BEGIN
    create_employee(
        p_first_name    => 'Neha',
        p_last_name     => 'Sharma',
        p_department_id => 20,
        p_hire_date     => DATE '2025-06-01'
        -- p_salary and p_is_manager skipped entirely — defaults apply
    );
END;
/
```
This specific case — needing to supply a *later* parameter while skipping *earlier optional ones* — is exactly the scenario where **positional notation alone cannot work at all**, and named notation isn't just "nicer," it's **necessary**.

---

## 6. Detailed Explanation

- **Matching rule for positional**: parameter 1 in the call maps to parameter 1 in the declaration, parameter 2 to parameter 2, and so on — purely by position, regardless of name.
- **Matching rule for named**: Oracle looks up each `param_name => value` pair against the procedure's declared parameter names and assigns accordingly — order in the call is irrelevant.
- **Mixed notation constraint, explained**: Once a named argument appears, Oracle can no longer safely infer what "the next positional slot" would even mean, since named arguments could have been supplied out of order — so the rule simply forbids reverting to positional afterward, keeping the matching logic unambiguous.
- This applies identically to **both procedures and functions** — everything in this topic works the same way regardless of which sub program type you're calling.

---

## 7. When to Use / When Not to Use Each

**Use positional notation when:**
- The sub program has **few parameters** (2–3) and their meaning is obvious from context or a very familiar, frequently-used signature.
- You're supplying **all** parameters anyway, in the natural declared order.

**Use named notation when:**
- The sub program has **several parameters**, especially with defaults.
- You want to **skip** some optional parameters entirely.
- Readability and long-term maintainability matter (which, in real production code, is almost always).
- You're calling a sub program whose signature might evolve over time, and you want your call site to be resilient to new parameters being added (as long as they're added with defaults).

**Use mixed notation when:**
- The first few parameters are unambiguous and commonly always supplied (so positional is fine for them), but later ones are optional/skippable (so named notation is needed for those).

**In real-world practice**: many teams adopt a standard of **always using named notation** for any procedure/function call beyond 1–2 parameters, purely for the long-term readability and safety benefits — even when it's not strictly *required* by the specific call. This is considered a mark of mature, professional PL/SQL code.

---

## 8. Common Mistakes & Misconceptions

1. **Mistake**: Trying to revert to positional notation after already using a named argument in the same call → compile error.
2. **Misconception**: "Named notation is just a style preference with no functional difference." → Not true in cases where you need to skip an earlier optional parameter to supply a later one — there, named notation isn't optional, it's the *only* way to express that call at all.
3. **Mistake**: Misspelling a parameter name in named notation — this raises a compile-time error (`PLS-00302: component must be declared`), which is arguably a *feature*, not just a risk: it catches typos immediately, unlike positional notation where a wrong-order value might silently pass type-checking (if types happen to coincidentally match) and produce subtly wrong behavior instead of an obvious error.
4. **Misconception**: "You can mix named and positional in any pattern, as long as all parameters are eventually covered." → False — the strict rule is: positional arguments (if any) must all come **first**, before any named ones.

---

## 9. Edge Cases to Be Aware Of

- If a parameter has **no default value** and you don't supply it (in either notation), Oracle raises an error (`PLS-00306` or similar) — defaults only apply to parameters explicitly declared with `DEFAULT`.
- Named notation works identically whether you're calling a **standalone** procedure/function or one defined **inside a package** (packages are our next major topic area) — the same rules apply.
- You *can* supply **all** parameters using named notation even when positional would have worked fine — there's no penalty for "over-using" named notation; it's always valid, just sometimes more verbose than necessary for very simple calls.

---

## 10. Interview-Level / Practical Notes

- A common practical question: *"When would you be forced to use named notation, not just prefer it?"* — Precisely when you need to supply a parameter that comes **after** one or more optional (defaulted) parameters, without wanting to explicitly restate those earlier defaults.
- *"What's the real-world benefit of named notation beyond readability?"* — Resilience to signature changes: if a procedure gets a new parameter added (with a default) in the middle of its parameter list (a design that's generally discouraged, but does happen), positional call sites can break or misbehave, while named notation call sites remain completely unaffected.
- Being fluent in mixed notation — recognizing exactly where the positional-to-named boundary must fall — is a good practical/interview signal of real hands-on experience versus purely theoretical knowledge.

---

## Things You Must Remember

- Positional: matched by **order**. Named: matched by **explicit name** using `=>`. Mixed: positional first, then named — never the reverse.
- Named notation lets you **skip** any parameter with a `DEFAULT` value, in any position — including skipping earlier ones to supply a later one, which positional notation **cannot do at all**.
- A misspelled parameter name in named notation is a **compile-time error** — this is a safety benefit, not just a cosmetic one.
- These rules apply identically to **procedures and functions**, standalone or packaged.
- Real-world teams often default to named notation for any non-trivial call, purely for long-term maintainability.

## How to Recognize This Concept

You need named (or mixed) notation, not just prefer it, when a requirement or situation implies:
- "Call this sub program but only supply **some** of its optional settings — leave the rest at their defaults."
- "The value I need to pass is for a parameter that comes **after** other optional ones I don't want to touch."
- "Make this call resilient to the procedure's signature changing over time."

If every parameter is being supplied anyway, in natural order, with a short and obvious signature — positional notation remains perfectly fine, and forcing named notation everywhere would just be unnecessary verbosity.

---

## Exercises

Using this procedure for all exercises:
```sql
CREATE OR REPLACE PROCEDURE schedule_shipment
    (p_order_id       IN NUMBER,
     p_carrier        IN VARCHAR2 DEFAULT 'STANDARD',
     p_priority       IN VARCHAR2 DEFAULT 'NORMAL',
     p_insured        IN VARCHAR2 DEFAULT 'N',
     p_ship_date      IN DATE DEFAULT SYSDATE + 1)
IS
BEGIN
    -- (implementation not relevant for these exercises)
    NULL;
END schedule_shipment;
/
```

1. **(Positional)** Call `schedule_shipment` positionally, supplying all 5 parameters with values of your choice.

2. **(Named, skip everything possible)** Call `schedule_shipment` using named notation, supplying only `p_order_id` and letting everything else default.

3. **(Forced named notation)** You need to schedule an order (`order_id = 4521`) with insurance (`p_insured = 'Y'`), but you want the carrier and priority to stay at their defaults. Write this call. Explain why positional notation **cannot** achieve this without also restating the defaults.

4. **(Mixed notation)** Write a call where `p_order_id` and `p_carrier` are supplied positionally, and `p_ship_date` is supplied by name, with everything else defaulted.

5. **(Spot the error)** What's wrong with this call, and why won't it compile?
   ```sql
   schedule_shipment(4521, p_carrier => 'EXPRESS', 'HIGH');
   ```

6. **(Realistic business scenario)** A logistics team's requirement: *"We're adding a new optional parameter `p_gift_wrap` (default 'N') to the `schedule_shipment` procedure. We have 40 existing call sites across the codebase — some positional, some named."* Explain which of those 40 call sites are at risk of breaking or behaving incorrectly because of this change, and why — and what this implies about which notation style is safer for long-lived code.

---

*Share your attempts whenever ready. Next up: Package Structure.*
