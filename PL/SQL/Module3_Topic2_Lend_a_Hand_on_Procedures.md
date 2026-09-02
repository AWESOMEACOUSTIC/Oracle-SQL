# Module 3, Topic 2: Lend a Hand on Procedures (Practice Checkpoint)

> This is a **hands-on consolidation session**, not a new theory topic. Its purpose is to take everything from Topic 1 (Types of Sub Programs and PL/SQL Procedure) and stretch it through realistic, business-flavored problems — before we move on to Functions. No new syntax is introduced here beyond what Topic 1 already covered.

---

## How to Use This Session

For each problem below, follow this discipline **before** writing any code:

1. What exactly is the business requirement asking for?
2. What data is involved, and where does it come from?
3. Does this need to be a **procedure**? (Revisit "How to Recognize This Concept" from Topic 1 if unsure.)
4. How many inputs does it need? What mode (`IN`) are they?
5. How many outputs does it need? What mode (`OUT` / `IN OUT`) are they?
6. What could go wrong (missing data, invalid input, no matching rows)? — Even though we haven't formally covered exception handling yet, just *identify* the risk; don't fix it yet unless you want to attempt it.
7. Only then, write the procedure.

I will **not** reveal which specific parameter mode or structure to use for most of these — that's exactly the skill this checkpoint is building.

---

## Practice Problems

### Problem 1 — Warehouse Stock Check
**Business context:** The inventory team wants a reusable routine that, given a `product_id`, tells them the current `stock_quantity` for that product from a table `inventory(product_id, product_name, stock_quantity)`.

**Task:** Design and write the procedure. Decide for yourself what inputs/outputs are needed.

---

### Problem 2 — Apply a Department-Wide Raise
**Business context:** Finance wants a routine where, given a `department_id` and a raise percentage, **every employee's salary in that department** gets increased by that percentage. No value needs to be returned — the update itself is the outcome. Assume a table `employees(employee_id, department_id, salary)`.

**Note:** You don't yet know loops (that's Module 2, coming later) — for this exercise, assume `UPDATE ... WHERE department_id = ...` as a single SQL statement handles "every employee" for you (this is standard SQL, not something new). Focus purely on the procedure wrapper: parameters, structure.

---

### Problem 3 — Transfer Funds Between Two Accounts
**Business context:** A banking system needs a routine that, given a `from_account_id`, a `to_account_id`, and an `amount`, deducts the amount from one account's balance and adds it to the other's. Assume a table `accounts(account_id, balance)`.

**Task:** Write the procedure. Think carefully: how many SQL statements does this really require? Should they be treated as one unit? (You don't need to solve "what if it fails halfway" yet — that's a Module 4 exception-handling concern. Just build the structure.)

---

### Problem 4 — Get Full Name and Grade in One Call
**Business context:** A reporting tool needs a routine where, given an `employee_id`, it gets back **both** the employee's full name (`first_name || ' ' || last_name`) **and** their `job_grade`, in a single call — without running two separate queries from the application side.

**Task:** Write the procedure. Pay attention to how many things need to flow *back* to the caller, and choose your parameter modes accordingly.

---

### Problem 5 — Conditional Bonus Assignment (identify the gap)
**Business context:** HR says: *"Given an employee ID and a performance rating (1 to 5), calculate and apply a bonus: rating 5 gets 20%, rating 4 gets 10%, rating 3 or below gets no bonus. Update the employee's salary accordingly."*

**Task:** Attempt to design the procedure's **signature** (name + parameters + modes) and write out, in plain English or pseudocode, what the body needs to do step by step. **Do not worry about actually writing the `IF` logic in real syntax** — we haven't covered selection statements yet. This exercise is about correctly identifying: "I know what data flows in, what needs to happen conceptually, and what (if anything) flows out — even though I can't fully implement the conditional part yet." This is intentional: it previews why Module 2 (Selection Statements) will matter, and shows you a real gap in your current toolkit, which is exactly how you'd feel hitting this in a real job before learning `IF`.

---

### Problem 6 — Spot the Design Flaw
**Business context:** A junior developer on your team wrote this procedure and asks for your review:

```sql
CREATE OR REPLACE PROCEDURE get_customer_status (p_customer_id IN NUMBER)
IS
    v_status VARCHAR2(20);
BEGIN
    SELECT status INTO v_status
    FROM customers
    WHERE customer_id = p_customer_id;
END get_customer_status;
/
```

**Task:** Review this critically. What is functionally wrong or incomplete about this procedure, given what it's clearly *trying* to do (return a customer's status to the caller)? Explain the issue in your own words, and rewrite it correctly.

---

## Notes Before You Attempt These

- These are intentionally **less guided** than Topic 1's exercises — no "concept tags" telling you what's being tested. That's the point of a "Lend a Hand" checkpoint: applying judgment, not just recalling syntax.
- It's completely fine if Problem 5 feels incomplete — that's a deliberate design choice to show you where your current knowledge boundary is, and to motivate why Module 2 exists.
- Take these one at a time if that's easier — you don't have to submit all six at once.

---

*Whenever you're ready, share your attempts (all at once or one by one) and I'll review them — pointing out design issues, better alternatives, and real-world considerations — before we move to the next file: Functions (with Lend a Hand).*
