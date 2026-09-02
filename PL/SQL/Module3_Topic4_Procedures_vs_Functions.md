# Module 3, Topic 4: Difference Between Procedures and Functions

> This topic is deliberately shorter than the previous two — it doesn't introduce new syntax. Its job is to **consolidate** what you've already learned into a sharp, decision-ready comparison, and to correct the subtle misconceptions that tend to surface once both concepts exist side by side in your head.

---

## 1. Why This Topic Exists on Its Own

You already know how to write both a procedure and a function. But in real work, the hard part usually isn't *"can I write this syntax correctly?"* — it's *"given this requirement, should this be a procedure or a function?"* That decision gets made in seconds, often before any code is written, and getting it wrong early tends to cause rework later (e.g., building something as a function, then discovering the report team can't call it from SQL because you gave it an `OUT` parameter).

This topic exists to make that decision instant and confident.

---

## 2. The Core Distinction (Restated Precisely)

| Dimension | Procedure | Function |
|---|---|---|
| **Purpose** | Perform an action | Compute and return a value |
| **Return value** | None directly (uses `OUT`/`IN OUT` params) | Exactly one, via mandatory `RETURN` |
| **Header syntax** | No `RETURN` clause | Must declare `RETURN datatype` |
| **Call style** | Standalone statement (`EXEC proc(...)`, or `proc(...);` inside a block) | Part of an expression (`v := func(...)`, or embedded in SQL) |
| **Usable inside SQL (`SELECT`/`WHERE`/`ORDER BY`)** | **No** | **Yes** (with purity-rule restrictions) |
| **Number of "outputs"** | Zero, one, or many (via multiple `OUT` params) | Always exactly one |
| **Typical parameter modes** | `IN`, `OUT`, `IN OUT` all common | Almost always `IN` only |
| **Side effects (DML)** | Expected and normal | Discouraged, and restricted in SQL-calling contexts |
| **Mental category** | Verb — "do this" | Noun/value — "give me this" |

---

## 3. Why These Differences Exist (Not Just What They Are)

It's easy to memorize the table above. It's more valuable to understand **why** Oracle designed it this way — because that's what lets you reason about *unfamiliar* requirements later, not just recall answers for familiar ones.

- **Why can't procedures be called from SQL?** SQL statements are built around retrieving/manipulating *data*, evaluated as expressions producing values. A procedure has no defined "value" to plug into that expression — it just performs actions. Allowing it in SQL would break the fundamental model of what a SQL statement *is*.
- **Why does a function need exactly one return value?** Because SQL expressions (a column in a `SELECT`, a condition in a `WHERE`) are inherently single-valued at each evaluation point. A `SELECT` column can't "return two things" for one row — so a function mirrors that same single-value contract.
- **Why are function side effects (DML) restricted in SQL contexts?** Because Oracle may evaluate a function an unpredictable number of times, in an unpredictable order, while optimizing a query (e.g., during parallel execution, or if it decides to evaluate a `WHERE` condition function multiple times). If that function silently updated data each time, your data could be corrupted based on the query optimizer's internal decisions — not something you want to depend on. This is why "purity rules" exist: Oracle enforces that functions called from SQL are (mostly) side-effect-free, so results stay predictable.

---

## 4. When Both *Could* Technically Work — Which to Prefer?

Sometimes a requirement is genuinely ambiguous — e.g., *"Get the employee's bonus amount."* You could:
- Write it as a **function** returning the bonus `NUMBER`, or
- Write it as a **procedure** with an `OUT` parameter for the bonus.

**How to decide:**
- Will this value ever need to be used **inside a query/report** (`SELECT`, `WHERE`)? → Function.
- Is this value just one part of a larger **action** that also updates something? → Procedure (bundle the calculation and the update together).
- Do you need **more than this one value** back? → Procedure.
- Is this purely a **read-only calculation** with no side effects, likely to be reused across many contexts including SQL? → Function — this is generally the *safer default* for pure calculations, because a function is strictly more flexible in where it can be called (procedures can't be used inside SQL, but nothing stops you from calling a function from inside another procedure).

**Real-world guideline:** When in doubt and the value is a *pure, read-only* computation, lean function — it's the more broadly reusable design. Reserve procedures for anything involving actual state changes (DML) or requiring multiple outputs.

---

## 5. A Realistic Business Case Showing Both Together

> **Requirement:** *"When an order is finalized, apply a loyalty discount based on the customer's tier, update the order's final amount, and let the caller know both the discount percentage applied and the final amount."*

**Breaking it down:**
- "Determine the discount percentage based on tier" → sounds like a pure calculation → good candidate for a **function**: `get_loyalty_discount(p_customer_id) RETURN NUMBER`.
- "Update the order's final amount" → this is an action with a side effect (DML) → needs a **procedure**.
- "Let the caller know the discount percentage AND the final amount" → two outputs → confirms procedure with `OUT` parameters.

**Resulting design:**
```sql
CREATE OR REPLACE FUNCTION get_loyalty_discount (p_customer_id NUMBER)
    RETURN NUMBER
IS
    v_tier VARCHAR2(20);
BEGIN
    SELECT tier INTO v_tier FROM customers WHERE customer_id = p_customer_id;

    -- Simplified logic (IF previewed minimally, formally covered in Module 2)
    IF v_tier = 'GOLD' THEN
        RETURN 15;
    ELSIF v_tier = 'SILVER' THEN
        RETURN 10;
    ELSE
        RETURN 0;
    END IF;
END get_loyalty_discount;
/

CREATE OR REPLACE PROCEDURE finalize_order
    (p_order_id IN NUMBER, p_customer_id IN NUMBER,
     p_discount_pct OUT NUMBER, p_final_amount OUT NUMBER)
IS
    v_original_amount NUMBER;
BEGIN
    SELECT order_amount INTO v_original_amount
    FROM orders
    WHERE order_id = p_order_id;

    p_discount_pct := get_loyalty_discount(p_customer_id);  -- function called FROM a procedure
    p_final_amount := v_original_amount - (v_original_amount * p_discount_pct / 100);

    UPDATE orders
    SET final_amount = p_final_amount
    WHERE order_id = p_order_id;
END finalize_order;
/
```

**Notice**: The procedure **calls the function internally**. This is completely normal and extremely common — procedures and functions aren't rivals, they're complementary tools you combine. The function handles the pure calculation piece; the procedure handles orchestration + the actual data update + returning multiple results.

---

## 6. Common Mistakes & Misconceptions

1. **Mistake**: Building a "calculation" as a procedure with a single `OUT` parameter, out of habit, when a function would have been more broadly usable (e.g., now it can't be used inside a report's `SELECT`).
2. **Misconception**: "A function can't update data at all." → It technically *can* (in a plain PL/SQL context, not called from SQL), but doing so is bad practice and gets restricted/blocked in SQL-calling contexts. The real rule to remember is: functions *should* be side-effect-free, not that they're always physically prevented from having side effects.
3. **Mistake**: Assuming you must pick one universally "better" tool. In real systems, well-designed code uses **both**, calling functions from within procedures (as shown above) — never the reverse conceptually, since a procedure has no value to give back into an expression.
4. **Misconception**: "Functions are just simpler procedures." → They serve genuinely different design roles, not just a syntax simplification.

---

## 7. Interview-Level / Practical Notes

- A favorite interview trap: *"Write me a procedure that returns a value."* — The correct answer is to explain that procedures don't "return" values in the function sense; they use `OUT` parameters, and clarify the distinction rather than trying to force a `RETURN` into a procedure (which is a syntax error, except for a bare `RETURN;` used only to exit early).
- *"Can a function call a procedure?"* — In modern Oracle versions, yes, under certain conditions, but it's uncommon and can violate purity rules if that procedure does DML and the function is called from SQL. As a practical default: **procedures calling functions** is the natural, safe direction; the reverse is a smell worth double-checking.
- Being able to instantly justify *why* you chose a procedure or function for a given requirement (not just which one) is one of the clearest signals of strong PL/SQL design maturity in an interview or code review.

---

## Things You Must Remember

- Procedure = **action**, no direct return value. Function = **single computed value**, callable from SQL.
- The single biggest practical differentiator: **"Will this be called from a SQL query?"** → if yes, it must be a function.
- Procedures and functions are **complementary**, not competing — procedures commonly call functions internally to reuse calculation logic.
- Default toward a function for **pure, read-only calculations**; default toward a procedure for anything involving **data changes or multiple return values**.
- Purity rules restrict (not always fully block) DML inside functions called from SQL — this exists to protect query predictability, not as an arbitrary limitation.

## How to Recognize This Concept

This "topic" itself is really a **decision skill**, so the recognition pattern is about the *decision*, not a single trigger phrase:

- If a requirement has **both** "calculate X" **and** "update Y" in it → you likely need **both**: a function for X, a procedure for Y (which may call the function internally), exactly like the loyalty discount example above.
- If you find yourself designing a function with an `OUT` parameter "just to return more stuff" → stop, and ask whether this should actually be a procedure instead.
- If you find yourself designing a procedure just to return one simple calculated number and nothing else changes → stop, and ask whether this should actually be a function instead, especially if a report might need it later.

---

## Exercises

These are intentionally comparison/judgment-focused rather than pure syntax drills, since that's what this topic is really about.

1. **(Classify it)** For each of the following, state whether it should be a procedure, a function, or both working together — and briefly justify:
   - a) "Return the number of days until an employee's work anniversary."
   - b) "Deactivate all customer accounts that haven't logged in for 2 years."
   - c) "Given a cart of items, calculate the total price including tax, and also apply a promo code discount, updating the cart's stored total."
   - d) "Check whether a given email address is already registered."

2. **(Refactor)** You're handed this procedure. Refactor it into a **function**, and explain why the refactor is an improvement:
   ```sql
   CREATE OR REPLACE PROCEDURE get_tax_amount (p_amount IN NUMBER, p_tax OUT NUMBER)
   IS
   BEGIN
       p_tax := p_amount * 0.18;
   END get_tax_amount;
   /
   ```

3. **(Design from scratch)** Business requirement: *"Given a support ticket ID, determine its priority level (based on how overdue it is) and update its `priority` column accordingly. The caller should also be told the new priority level."* Design this using both a function and a procedure together, similar to the loyalty discount example. Write out the two signatures (don't need full bodies) and explain the division of responsibility.

4. **(Defend your choice)** A teammate insists: *"Just make everything a procedure with OUT parameters — it's more flexible since it can return multiple things."* Write a short rebuttal (4–5 sentences) explaining the real cost of that approach, using the SQL-callability angle specifically.

5. **(Edge case reasoning)** Suppose a function that calculates a discount is accidentally written to also perform an `UPDATE` on a `discount_log` table every time it runs. This function is then used inside a `SELECT` report that scans 100,000 rows. What could go wrong, practically, in production? Think about both correctness and performance.

---

*Once you've worked through these, share your answers and I'll review them. Next up: Defining a Sub Program in an SQL Statement.*
