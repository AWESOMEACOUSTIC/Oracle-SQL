# Module 2, Topic 3: CASE Statement

---

## 1. What Is the CASE Statement?

`CASE` is a **selection construct**, like `IF`, but built around **matching one expression against a set of discrete possible values** (or a set of independent conditions), rather than chaining `ELSIF` conditions one after another. It comes in two distinct forms:

1. **Simple CASE** — matches a single expression against specific literal values.
2. **Searched CASE** — evaluates a list of independent boolean conditions, similar in spirit to an `ELSIF` chain, but with different syntax and some genuinely different capabilities.

---

## 2. Why Does CASE Exist? What Problem Does It Solve?

Consider this `ELSIF` chain:
```sql
IF v_status_code = 1 THEN
    v_status_desc := 'Pending';
ELSIF v_status_code = 2 THEN
    v_status_desc := 'Approved';
ELSIF v_status_code = 3 THEN
    v_status_desc := 'Rejected';
ELSIF v_status_code = 4 THEN
    v_status_desc := 'Cancelled';
ELSE
    v_status_desc := 'Unknown';
END IF;
```
Notice: **every single condition** is testing the **same variable** (`v_status_code`) against a **different specific value**. This is an extremely common pattern — mapping a code to a description, translating a category into a label, routing based on a discrete status. Repeating `v_status_code = ` five times is verbose and slightly obscures the real intent: *"look up this one value in this list."*

`CASE` (specifically, the **Simple CASE** form) exists precisely for this pattern — **matching one single expression against several possible discrete values** — expressing the exact same logic far more concisely and clearly:
```sql
CASE v_status_code
    WHEN 1 THEN v_status_desc := 'Pending';
    WHEN 2 THEN v_status_desc := 'Approved';
    WHEN 3 THEN v_status_desc := 'Rejected';
    WHEN 4 THEN v_status_desc := 'Cancelled';
    ELSE        v_status_desc := 'Unknown';
END CASE;
```

The **Searched CASE** form solves a related but different problem: sometimes your conditions **aren't** all testing the same variable against different values — they're genuinely independent boolean conditions (like an `ELSIF` chain), but you still want CASE's syntax style, or you need CASE specifically because you're using it as an **expression** (covered below) rather than a statement.

---

## 3. Syntax: Simple CASE Statement

```sql
CASE selector_expression
    WHEN value1 THEN statement(s);
    WHEN value2 THEN statement(s);
    ...
    ELSE statement(s);  -- optional
END CASE;
```

- `selector_expression` is evaluated **once**.
- Its result is compared against each `WHEN value` in order, top to bottom.
- The **first** match's statement(s) execute; remaining `WHEN`s are not evaluated.
- `ELSE` (optional) runs if **no** `WHEN` matches.

---

## 4. Syntax: Searched CASE Statement

```sql
CASE
    WHEN condition1 THEN statement(s);
    WHEN condition2 THEN statement(s);
    ...
    ELSE statement(s);  -- optional
END CASE;
```

- No single selector expression — each `WHEN` has its **own independent boolean condition**.
- Conditions checked top to bottom; first `TRUE` one wins, exactly like an `ELSIF` chain.
- Functionally very close to `IF...ELSIF...ELSE`, just different syntax — useful when the conditions genuinely are independent (not testing one variable against different values), or for consistency/style within a codebase.

```sql
CASE
    WHEN v_score >= 90 THEN v_grade := 'A';
    WHEN v_score >= 75 THEN v_grade := 'B';
    WHEN v_score >= 60 THEN v_grade := 'C';
    ELSE v_grade := 'F';
END CASE;
```

---

## 5. Critical: CASE Statement vs. CASE Expression

This is the single most important distinction in this entire topic, and a very common source of confusion.

- The syntax you've seen so far (`CASE ... WHEN ... THEN statement(s); ... END CASE;`) is the **CASE statement** — it appears in the `BEGIN` section of a block as its own standalone control-flow statement, just like `IF`.
- There's a **separate, related construct** called a **CASE expression**, which **returns a value** and can be used **inline**, anywhere a value/expression is expected — including directly inside SQL, inside an assignment, or as a function argument:
```sql
v_grade := CASE
               WHEN v_score >= 90 THEN 'A'
               WHEN v_score >= 75 THEN 'B'
               WHEN v_score >= 60 THEN 'C'
               ELSE 'F'
           END;  -- note: no "CASE" repeated before END here — that's specific to the statement form
```
Or directly in SQL:
```sql
SELECT employee_id,
       CASE
           WHEN salary >= 100000 THEN 'High'
           WHEN salary >= 50000  THEN 'Medium'
           ELSE 'Low'
       END AS salary_band
FROM employees;
```

**Key syntactic differences to notice:**
- The **statement** form ends with `END CASE;` and each `WHEN` leads to one or more **statements** (ending in `;`).
- The **expression** form ends with just `END` (no `CASE` repeated, no trailing `;` inside an assignment context) and each `WHEN` leads to a **single value**, not a statement.
- The **expression** form can be used **directly inside SQL** — this is extremely common in real reporting queries — while the **statement** form cannot (it's PL/SQL control flow, not a SQL-embeddable value).

This connects directly back to Module 3's Function topic: a `CASE` **expression** is a value-producing construct, conceptually similar in spirit to why functions are SQL-callable while procedures aren't — expressions produce values usable inline; statements perform control flow and cannot be embedded inside a SQL value context.

---

## 6. Simple Examples

### Example 1 — Simple CASE statement
```sql
DECLARE
    v_day_num NUMBER := 3;
    v_day_name VARCHAR2(10);
BEGIN
    CASE v_day_num
        WHEN 1 THEN v_day_name := 'Monday';
        WHEN 2 THEN v_day_name := 'Tuesday';
        WHEN 3 THEN v_day_name := 'Wednesday';
        WHEN 4 THEN v_day_name := 'Thursday';
        WHEN 5 THEN v_day_name := 'Friday';
        ELSE v_day_name := 'Weekend';
    END CASE;

    DBMS_OUTPUT.PUT_LINE(v_day_name);
END;
/
```

### Example 2 — Searched CASE statement
```sql
DECLARE
    v_temperature NUMBER := 38;
    v_alert VARCHAR2(20);
BEGIN
    CASE
        WHEN v_temperature >= 40 THEN v_alert := 'CRITICAL';
        WHEN v_temperature >= 37 THEN v_alert := 'FEVER';
        WHEN v_temperature >= 36 THEN v_alert := 'NORMAL';
        ELSE v_alert := 'LOW';
    END CASE;

    DBMS_OUTPUT.PUT_LINE(v_alert);
END;
/
```

### Example 3 — CASE expression used in an assignment
```sql
DECLARE
    v_tier VARCHAR2(10) := 'GOLD';
    v_discount NUMBER;
BEGIN
    v_discount := CASE v_tier
                      WHEN 'GOLD'   THEN 15
                      WHEN 'SILVER' THEN 10
                      ELSE 0
                  END;

    DBMS_OUTPUT.PUT_LINE('Discount: ' || v_discount);
END;
/
```

### Example 4 — CASE with no matching WHEN and no ELSE (an important edge case)
```sql
DECLARE
    v_code NUMBER := 99;
    v_desc VARCHAR2(20);
BEGIN
    CASE v_code
        WHEN 1 THEN v_desc := 'Type A';
        WHEN 2 THEN v_desc := 'Type B';
    END CASE;  -- no ELSE, and v_code = 99 matches nothing

    DBMS_OUTPUT.PUT_LINE(v_desc);
END;
/
-- Result: raises CASE_NOT_FOUND exception! (see Detailed Explanation below)
```

---

## 7. Detailed Explanation — The CASE_NOT_FOUND Exception (Important)

This is a **critical behavioral difference** from `IF`, and one of the most commonly tested/misunderstood facts about `CASE`:

**If a CASE statement has no `ELSE` clause, and none of the `WHEN` conditions match, PL/SQL raises the pre-defined exception `CASE_NOT_FOUND`.**

Contrast this directly with `IF`: an `IF...ELSIF` chain with no matching condition and no `ELSE` simply **does nothing** and moves on — no error at all (as you learned in Topic 2). `CASE` behaves **completely differently** — it's an **error condition** if nothing matches and there's no `ELSE`. This is a real, practical trap: code that "worked fine" during testing (because test data happened to always match a `WHEN`) can suddenly crash in production the first time an unanticipated value shows up.

**Practical implication**: it is considered **very strong practice** to **always include an `ELSE` clause** in a `CASE` statement, even if you believe every possible value is already covered by your `WHEN` clauses — as a defensive safety net against exactly this kind of surprise crash.

---

## 8. When to Use CASE vs. IF

**Prefer CASE (Simple form) when:**
- You're matching **one single expression/variable** against **several specific discrete values** — this is CASE's ideal use case, and reads far more clearly than an equivalent `ELSIF` chain repeating the same variable name over and over.

**Prefer CASE (Searched form) when:**
- Your conditions are independent (not one variable against different values), but you specifically need a **value-producing expression** usable inline (in an assignment, in SQL) — reach for the CASE **expression** form here, not `IF` (which cannot produce an inline value at all).

**Prefer IF when:**
- Conditions are independent, you're performing genuine **control flow** (not producing a value), and/or you want the safer default behavior of silently doing nothing when nothing matches (no `CASE_NOT_FOUND` risk).

---

## 9. Common Mistakes & Misconceptions

1. **Mistake**: Omitting `ELSE` in a `CASE` statement, assuming it behaves like `IF` and simply does nothing if no match is found → raises `CASE_NOT_FOUND` instead, an unhandled exception if you haven't planned for it.
2. **Misconception**: "CASE statement and CASE expression are just two names for the same thing." → They have genuinely different syntax (`END CASE;` vs. plain `END`), different capabilities (expression form is SQL-embeddable; statement form is not), and different typical usage contexts.
3. **Mistake**: Trying to use a **CASE statement** directly inside a SQL `SELECT` — this doesn't work; only the **CASE expression** form can appear inside SQL.
4. **Misconception**: "Simple CASE can test range conditions like `> 100`." → It cannot — Simple CASE only tests **equality** against literal values. For range-based conditions, you need the **Searched CASE** form (or `IF`/`ELSIF`).
5. **Mistake**: Forgetting that, just like `ELSIF`, only the **first** matching `WHEN` in a Searched CASE executes — order still matters, exactly as it does with `IF...ELSIF`.

---

## 10. Edge Cases to Be Aware Of

- `CASE_NOT_FOUND` is a genuine pre-defined exception (from Module 4, Topic 2's family) — you can catch it explicitly with `WHEN CASE_NOT_FOUND THEN ...` in an `EXCEPTION` section, exactly like `NO_DATA_FOUND` or `ZERO_DIVIDE`, if you specifically want to handle this scenario gracefully rather than just always including an `ELSE`.
- If the `selector_expression` in a Simple CASE evaluates to `NULL`, and one of your `WHEN` values is also somehow meant to represent "unknown" — be careful: `CASE` uses ordinary equality comparison internally, and `NULL = NULL` is still `NULL` (not `TRUE`, per Module 1, Topic 5's rules) — so a `NULL` selector will **never** match a specific `WHEN` value; it will always fall through to `ELSE` (or raise `CASE_NOT_FOUND` if there's no `ELSE`).
- A CASE expression, unlike a CASE statement, **must** produce a value on every reachable path — omitting `ELSE` in a CASE **expression** used in a context where a value is strictly required and no match occurs will similarly raise `CASE_NOT_FOUND` at runtime.

---

## 11. Interview-Level / Practical Notes

- *"What happens if no WHEN clause matches in a CASE statement with no ELSE?"* — Raises `CASE_NOT_FOUND` — a very frequently tested, easy-to-get-wrong fact (people instinctively assume it behaves like `IF`).
- *"What's the difference between a CASE statement and a CASE expression?"* — Statement performs control flow (`END CASE;`, statements in each branch); expression produces a value (`END` only, single value per branch, usable inline/in SQL).
- *"When would Simple CASE not be sufficient, requiring Searched CASE or IF instead?"* — When you need to test **ranges or independent conditions**, not just equality against one variable.

---

## Things You Must Remember

- **Simple CASE**: one selector expression, matched against discrete `WHEN value` options (equality only).
- **Searched CASE**: independent `WHEN condition` clauses, similar to `ELSIF`, first match wins.
- **CASE statement** (`END CASE;`) performs control flow; **CASE expression** (`END` only) produces a value usable inline, including inside SQL.
- **Always include `ELSE`** in a CASE statement/expression — omitting it risks a runtime `CASE_NOT_FOUND` exception if nothing matches, unlike `IF`, which silently does nothing.
- A `NULL` selector value will never match any specific `WHEN` value in a Simple CASE — it always falls to `ELSE` (or raises `CASE_NOT_FOUND`).

## How to Recognize This Concept

Reach for **Simple CASE** when a requirement describes **mapping one variable's specific values to specific outcomes** — "status code 1 means X, code 2 means Y, code 3 means Z."

Reach for **Searched CASE** or a **CASE expression** when you need **CASE's value-producing capability inline** — especially if the result needs to go directly into a `SELECT` column, an assignment, or a function argument — rather than pure control flow.

Whenever you write **any** CASE, make it a reflex to ask: *"have I included an ELSE, or am I certain every possible value is covered — and even then, should I include one anyway as a safety net?"*

---

## Exercises

1. **(Simple CASE)** Write a Simple CASE statement mapping a `v_priority_code` (1, 2, or 3) to `'HIGH'`, `'MEDIUM'`, `'LOW'` respectively, with `'UNKNOWN'` as a safety-net `ELSE`.

2. **(Searched CASE)** Rewrite Module 2, Topic 2's "discount tier" exercise (Platinum/Gold/Silver/Standard based on order amount ranges) using a Searched CASE statement instead of `IF...ELSIF`.

3. **(CASE expression in SQL)** Write a `SELECT` statement against `employees(employee_id, salary)` that shows each employee's `salary` alongside a computed `salary_band` column (`'High'`/`'Medium'`/`'Low'`) using a CASE **expression** directly inside the query.

4. **(CASE_NOT_FOUND, predicted)** Predict what happens when this runs, and explain why:
   ```sql
   DECLARE
       v_type NUMBER := 5;
       v_label VARCHAR2(20);
   BEGIN
       CASE v_type
           WHEN 1 THEN v_label := 'One';
           WHEN 2 THEN v_label := 'Two';
       END CASE;
       DBMS_OUTPUT.PUT_LINE(v_label);
   END;
   /
   ```

5. **(NULL selector trap)** Predict the output, and explain why, referencing what you know about NULL comparisons:
   ```sql
   DECLARE
       v_status VARCHAR2(10) := NULL;
       v_result VARCHAR2(20);
   BEGIN
       CASE v_status
           WHEN 'ACTIVE' THEN v_result := 'Is active';
           WHEN NULL THEN v_result := 'Status unknown';  -- does this ever match?
           ELSE v_result := 'Other';
       END CASE;
       DBMS_OUTPUT.PUT_LINE(v_result);
   END;
   /
   ```

6. **(Statement vs expression judgment)** A developer needs to compute a `shipping_cost` value directly inside a `SELECT` query based on an `order_weight` column's ranges (light/medium/heavy). Should they use a CASE statement or a CASE expression? Explain why the other option genuinely wouldn't work here, not just why your choice is "nicer."

7. **(Realistic business scenario)** Business requirement: *"Map a customer's `subscription_type_code` (values 1 through 5, but new codes may be added in the future without our script being updated immediately) to a display name for the billing report. Any code we don't recognize should be clearly flagged as 'Unrecognized Plan' rather than causing the report to fail."* Write this using the appropriate CASE form, and explain specifically how your design choice avoids the `CASE_NOT_FOUND` risk described in this topic.

---

*Share your answers whenever you're ready. Next up: Module 2, Topic 4 — Iteration Statement.*
