# Topic 3: Hands-On Practice — Choosing Between Cursors With and Without Parameters

> **Syllabus position:** Topic 3 of 6 (+ final combined practice set)
> **Nature of this topic:** Unlike Topics 1 and 2, this topic introduces **no new syntax**. Its entire purpose is to sharpen your **decision-making** — given a real requirement, can you correctly judge whether a parameterized or non-parameterized cursor is the right call, and defend that choice? This is the "lend a hand" reinforcement block sitting between the mechanics (Topic 2) and the next new concept (Topic 4, Cursor FOR Loops).

---

## Why This Topic Exists on Its Own

Topic 2 taught you *how* to write both forms. Knowing the syntax is not the same as knowing *when* to reach for which — and this is exactly the kind of judgment call that separates "I can write a cursor" from "I know when and why to use this concept," which is the whole point of this course. In real codebases, the wrong choice here doesn't cause a compile error — it causes maintainability debt (duplicated near-identical cursors), subtle bugs (the name-collision trap), or code review pushback. This topic drills that judgment through a decision framework and a set of realistic, sometimes intentionally ambiguous scenarios.

## Decision Framework

Work through these questions, in order, whenever you're unsure which form to use:

1. **Will this cursor's filter value ever need to differ across multiple opens — in this block, in a loop, or across different callers of the subprogram it lives in?**
   - No, it's genuinely a one-time, fixed value → a literal is fine. Don't add a parameter nobody will ever use.
   - Yes → lean toward a parameter.

2. **Is this cursor going inside a reusable procedure or function that different callers will invoke with different inputs?**
   - Yes → almost always parameterize. The subprogram's own parameter should flow straight into the cursor's parameter — this is the standard idiom.

3. **Does the filter value come from another cursor's current row (a master-detail/nested pattern)?**
   - Yes → parameterize the inner cursor. This also pairs naturally with a cursor FOR loop (Topic 4) so open/close bookkeeping is automatic.

4. **Is the value the kind of thing that's realistically likely to become variable later, even if it's fixed today** (an ID like a warehouse, store, region, or customer)?
   - These identifiers change far more often than developers expect, and the cost of adding a parameter is essentially zero. When in doubt on this specific kind of value, most experienced developers parameterize by default — this isn't "speculative over-engineering," it's recognizing a very common pattern of business growth (a company that says "we only have one warehouse" today frequently doesn't say that in a year).
   - This is different from adding genuinely speculative flexibility nobody asked for (e.g., parameterizing a value that is a true, permanent business constant, like a fixed regulatory threshold) — that would be over-engineering. The distinction is: **identifiers tied to business entities** (store, customer, region, department) vs. **values that are truly constant by definition**.

5. **Am I about to write a second (or third) cursor that's nearly identical to one I already wrote, differing only in a hardcoded literal?**
   - This is a design smell (see below) — stop and refactor into one parameterized cursor.

6. **Am I relying on an outer/global variable purely so I can "reopen" the same cursor with different values?**
   - This technically works (Topic 2 showed this), but it's almost always a sign the cursor should be parameterized instead, for the readability/self-documentation reasons already covered.

## Design Smells to Watch For

- **Multiple near-identical cursors** differing only by a literal in the WHERE clause (`c_emp_dept10`, `c_emp_dept20`, `c_emp_dept30`) — should be one parameterized cursor.
- **A cursor's WHERE clause references a variable that gets reassigned several times** purely to drive repeated opens of that same cursor — should be a parameter.
- **A parameter or variable name that's textually identical to a column it's compared against** — the name-collision trap from Topic 2; always worth a specific second look in code review, since it produces no error and no warning.
- **A cursor with a parameter that's never actually varied anywhere it's called** — mild smell in the other direction; not wrong, but worth asking whether the parameter is pulling its weight, or whether the value is genuinely fixed and the parameter is unnecessary ceremony.

---

# Exercises — With Answers and Reasoning

These are deliberately mixed: some are quick judgment calls, some are refactors, one is a bug hunt. None will tell you upfront which form to use — that's the decision you're being asked to make.

### Exercise 1
**Scenario:** HR has asked for a one-off audit script to print all employees in department 20, related to a specific investigation happening this week only. It will be run once, by one person, and then discarded.

**Decision:** Non-parameterized, literal-based cursor.

**Reasoning:** Apply Question 1 of the framework: will the filter value ever need to differ across multiple opens or callers? No — this is explicitly a single-use, throwaway script tied to one specific investigation. Adding a parameter here provides zero real benefit and is pure ceremony for code that will never be reused. This is the correct case for the simplest form:
```sql
DECLARE
   CURSOR c_dept20_emp IS
      SELECT employee_id, first_name FROM employees WHERE department_id = 20;
BEGIN
   FOR rec IN c_dept20_emp LOOP
      DBMS_OUTPUT.PUT_LINE(rec.first_name);
   END LOOP;
END;
/
```

---

### Exercise 2
**Scenario:** You're writing a procedure, `list_orders_by_status`, that will be called from several different parts of the application — one screen needs pending orders, another needs shipped orders, another needs cancelled orders.

**Decision:** Parameterized cursor, fed by the procedure's own parameter.

**Reasoning:** Question 2 applies directly — this cursor lives inside a reusable procedure that different callers will invoke with different filter values. The standard idiom is to have the procedure's parameter flow straight into the cursor's parameter:
```sql
CREATE OR REPLACE PROCEDURE list_orders_by_status (p_status IN orders.status%TYPE) IS
   CURSOR c_orders (p_status_filter orders.status%TYPE) IS
      SELECT order_id, order_date FROM orders WHERE status = p_status_filter;
BEGIN
   FOR rec IN c_orders(p_status) LOOP
      DBMS_OUTPUT.PUT_LINE(rec.order_id || ' - ' || rec.order_date);
   END LOOP;
END;
/
```
Note the cursor parameter (`p_status_filter`) is named differently from both the procedure parameter (`p_status`) and the column (`status`) — this avoids any ambiguity, even though `p_status` alone wouldn't have collided with the column `status` here (they aren't textually identical). Being deliberate about this avoids ever having to think twice about whether a collision exists.

---

### Exercise 3
**Scenario:** For every store in the `stores` table, the operations team wants a list of products that went out of stock at least once this month in that store.

**Decision:** Parameterized inner cursor, nested inside an outer cursor over stores.

**Reasoning:** Question 3 applies — the filter value (store ID) comes directly from another cursor's current row. This is the master-detail pattern from Topic 2, Example E:
```sql
DECLARE
   CURSOR c_stores IS
      SELECT store_id, store_name FROM stores;

   CURSOR c_stockouts (p_store_id stores.store_id%TYPE) IS
      SELECT product_name FROM stock_events
      WHERE store_id = p_store_id
        AND event_type = 'OUT_OF_STOCK'
        AND event_date >= TRUNC(SYSDATE, 'MM');
BEGIN
   FOR store_rec IN c_stores LOOP
      DBMS_OUTPUT.PUT_LINE('Store: ' || store_rec.store_name);
      FOR stockout_rec IN c_stockouts(store_rec.store_id) LOOP
         DBMS_OUTPUT.PUT_LINE('   - ' || stockout_rec.product_name);
      END LOOP;
   END LOOP;
END;
/
```
Using cursor FOR loops for both levels means the inner cursor is automatically opened and closed on every outer iteration — no risk of the `ORA-06511` "already open" error discussed in Topic 2, Exercise 6.

---

### Exercise 4
**Scenario:** A nightly batch job always processes Warehouse ID 5 — the company currently operates exactly one warehouse. The requirement, as written, says nothing about future plans.

**Decision:** Lean parameterized, even though only one value is used today — but this one is a genuine judgment call, and a literal isn't "wrong."

**Reasoning:** This is Question 4 territory, and it's intentionally ambiguous. Two defensible positions:
- **For the literal:** the requirement doesn't mention any plan to add warehouses, and YAGNI ("you aren't gonna need it") argues against building flexibility nobody asked for.
- **For the parameter:** a warehouse ID is exactly the kind of business-entity identifier that very commonly stops being "just one value" as a company grows, and the cost of writing `CURSOR c_movements (p_warehouse_id NUMBER) IS ...` instead of hardcoding `5` is essentially zero — no extra complexity, no extra risk, just one more token in the declaration. Because the downside of guessing wrong (literal, then the company opens a second warehouse next quarter) is "go find and rewrite this script," while the downside of guessing wrong the other way (parameter, but it turns out this really was permanent) is "nothing — it still works exactly the same, just with one harmless parameter," the asymmetry favors parameterizing.

In practice, most experienced developers would parameterize this without much deliberation, specifically because IDs tied to a business entity (warehouse, store, region, customer, department) are a recognizable pattern worth defaulting to parameterized — **not** because "always parameterize everything" is a good rule (it isn't; see Exercise 1). The distinguishing signal is: *is this value tied to a business entity that could plausibly multiply, or is it a true structural constant?* A warehouse ID is the former.

```sql
CURSOR c_movements (p_warehouse_id warehouses.warehouse_id%TYPE) IS
   SELECT movement_id, item_id, quantity FROM inventory_movements
   WHERE warehouse_id = p_warehouse_id;
-- called today as: OPEN c_movements(5);
```

---

### Exercise 5
**Scenario:** You inherit this code in a review:
```sql
DECLARE
   v_filter_dept employees.department_id%TYPE;
   CURSOR c_emp IS
      SELECT first_name FROM employees WHERE department_id = v_filter_dept;
BEGIN
   v_filter_dept := 10;
   OPEN c_emp;
   -- ... fetch loop, close ...

   v_filter_dept := 20;
   OPEN c_emp;
   -- ... fetch loop, close ...

   v_filter_dept := 30;
   OPEN c_emp;
   -- ... fetch loop, close ...
END;
/
```
Should this be refactored? Why or why not?

**Decision:** Yes, refactor to a parameterized cursor.

**Reasoning:** This is Question 6's exact pattern — an outer variable is being reassigned three separate times purely to drive three separate opens of the same cursor. It works (Topic 2 confirmed the variable is re-read fresh at each `OPEN`), but it's strictly worse than a parameter here: a reader has to trace `v_filter_dept`'s reassignments through the whole block to understand what each `OPEN` actually does, and it's easy to introduce a bug by, say, accidentally reordering statements or reusing `v_filter_dept` for something else later in a larger block. Refactored:
```sql
DECLARE
   CURSOR c_emp (p_dept_id employees.department_id%TYPE) IS
      SELECT first_name FROM employees WHERE department_id = p_dept_id;
BEGIN
   OPEN c_emp(10);
   -- ... fetch loop, close ...

   OPEN c_emp(20);
   -- ... fetch loop, close ...

   OPEN c_emp(30);
   -- ... fetch loop, close ...
END;
/
```
Each `OPEN` now states its own filter value directly, with no need to track a variable's mutation history.

---

### Exercise 6
**Scenario — spot the smell:** You find this in a codebase:
```sql
CURSOR c_emp_dept10 IS SELECT * FROM employees WHERE department_id = 10;
CURSOR c_emp_dept20 IS SELECT * FROM employees WHERE department_id = 20;
CURSOR c_emp_dept30 IS SELECT * FROM employees WHERE department_id = 30;
```
What's wrong with this, and how should it be rewritten?

**Answer:** This is the clearest form of the "multiple near-identical cursors" smell from the checklist. All three cursors have exactly the same query shape and differ only in one hardcoded literal. This means: (a) any future change to the SELECT list or business logic has to be made in three places instead of one, risking them drifting out of sync; (b) it doesn't scale — a fourth department means a fourth near-duplicate cursor; and (c) it obscures the fact that there's really only one piece of logic here, not three. Rewrite as one parameterized cursor, opened with whichever department ID is needed:
```sql
CURSOR c_emp_by_dept (p_dept_id employees.department_id%TYPE) IS
   SELECT * FROM employees WHERE department_id = p_dept_id;
-- OPEN c_emp_by_dept(10);  OPEN c_emp_by_dept(20);  OPEN c_emp_by_dept(30);
```

---

### Exercise 7
**Scenario:** The reporting team wants a procedure that accepts a customer ID and a date range (start and end date) and returns all orders for that customer within that range. This procedure will be called from multiple places in the application with different customers and ranges each time. If no end date is given, it should default to today.

**Decision:** A procedure with three of its own parameters, feeding a cursor with matching parameters — including a default.

**Reasoning:** This combines several framework points: it's a reusable subprogram called with varying inputs (Question 2), it has multiple filter values that all vary together (multi-parameter cursor from Topic 2), and one of them has a sensible fallback (default value from Topic 2).

```sql
CREATE OR REPLACE PROCEDURE list_customer_orders (
   p_customer_id IN orders.customer_id%TYPE,
   p_start_date  IN DATE,
   p_end_date    IN DATE DEFAULT SYSDATE
) IS
   CURSOR c_orders (p_cust_id orders.customer_id%TYPE,
                    p_from    DATE,
                    p_to      DATE) IS
      SELECT order_id, order_date, amount
      FROM orders
      WHERE customer_id = p_cust_id
        AND order_date BETWEEN p_from AND p_to;
BEGIN
   FOR rec IN c_orders(p_customer_id, p_start_date, p_end_date) LOOP
      DBMS_OUTPUT.PUT_LINE(rec.order_id || ' - ' || rec.order_date || ' - ' || rec.amount);
   END LOOP;
END;
/
```
Note the default lives on the **procedure's** parameter (`p_end_date DEFAULT SYSDATE`), which is then always passed explicitly into the cursor. You could alternatively put the default directly on the cursor's own parameter instead — either works, but defaulting at the procedure boundary is generally clearer, since that's the interface callers actually interact with; the cursor is an internal implementation detail.

---

### Exercise 8
**Scenario — bug hunt:** A junior developer writes this for a procedure meant to process one order at a time, given an order ID from an external system:
```sql
CURSOR c_ord (order_id NUMBER) IS
   SELECT * FROM orders WHERE order_id = order_id;
```
In testing, no matter what order ID is passed in, the cursor returns **every** order in the table. What's the bug, and how would you catch this kind of issue during code review even before running it?

**Answer:** This is the name-collision trap from Topic 2 — the parameter `order_id` is textually identical to the `order_id` column being filtered on, so Oracle resolves the unqualified reference in the WHERE clause to the column, making it compare `order_id = order_id`, which is true for every row (except any with a `NULL` order_id, since `NULL = NULL` is `UNKNOWN`, not `TRUE`). The input parameter is never actually used for filtering, and no error is raised anywhere — it compiles and runs "successfully," just against the wrong logic.

**How to catch it in review, before even running it:** specifically scan any cursor's parameter list against its own WHERE clause for **textually identical names** to columns being compared — this is a fast, mechanical check, not something that requires deep analysis. As a team convention, requiring a prefix (`p_`) on every cursor and subprogram parameter removes the possibility of this collision entirely, which is why that convention exists — it's not a style preference, it's a bug-class elimination. If you're testing rather than reviewing source, the tell is: **passing two different input values produces identical output** — any time changing an input doesn't change a result at all, that's a strong signal the input isn't actually being used where you think it is.

**Fix:**
```sql
CURSOR c_ord (p_order_id orders.order_id%TYPE) IS
   SELECT * FROM orders WHERE order_id = p_order_id;
```

---

## Things You Must Remember

- This topic added no new syntax — its value is entirely in the judgment calls above. If you can look at a requirement and immediately land on the right side of Questions 1–6, this topic has done its job.
- "Always parameterize" and "never parameterize unless asked" are both wrong defaults. The real skill is recognizing **which kind of value** you're filtering on — a business-entity identifier (store, customer, warehouse, region) vs. a genuinely fixed, one-off, or truly constant value.
- Multiple near-identical cursors differing only by a literal, and repeated re-assignment of a variable purely to drive re-opens of the same cursor, are both concrete, checkable design smells — not vague style opinions.
- The name-collision trap is a silent, no-error bug. Mechanically checking parameter names against column names in the same query is a fast, specific code-review habit worth building now.

## How to Recognize Which Form Fits

By now, treat these as fast pattern-matches rather than long deliberations:

- "Run this once, for this one specific case" → literal.
- "Called from multiple places / by different callers with different values" → parameter.
- "For each [outer entity], do [inner thing]" → parameterized inner cursor + outer loop.
- "This ID probably won't always be just one value" → parameter, even if only one value exists today.
- Seeing three cursors that look like clones of each other → stop, that's a parameter that got skipped.

---

**End of Topic 3.** Next file: `04-cursor-for-loops.md`.