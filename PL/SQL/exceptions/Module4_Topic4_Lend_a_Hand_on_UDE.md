# Module 4, Topic 4: Lend a Hand on User Defined Exception (Practice Checkpoint)

> This is a **hands-on consolidation session**, not a new theory topic — same format as the "Lend a Hand on Procedures" checkpoint in Module 3. It stretches everything from Topics 1–3 (exception mechanism, pre-defined exceptions, user-defined exceptions) through realistic, less-guided business problems, before we move into Raise Application Error.

---

## How to Use This Session

For each problem, work through this discipline before writing code:

1. What is the business requirement actually asking for?
2. What could realistically go wrong — both **generic runtime issues** (missing data, bad math, duplicates) and **business rule violations** (limits, policies, invalid states)?
3. For each thing that could go wrong: is it something **Oracle already recognizes** (→ pre-defined exception) or something **only your business logic knows about** (→ user-defined exception)?
4. Where exactly does the `RAISE` belong for any user-defined exception — i.e., what's the precise condition that triggers it?
5. How should each distinct problem be **handled differently** — do they all deserve the same generic response, or does the business context call for different messages/actions?
6. Only then, write the code.

As with the procedures checkpoint, I will **not** tell you upfront which specific exceptions (pre-defined vs. user-defined) apply to each problem — identifying that is the core skill being tested here.

---

## Practice Problems

### Problem 1 — Employee Lookup with Validation
**Business context:** HR wants a procedure `get_employee_grade` that, given an `employee_id`, returns (via `OUT` parameter) the employee's job grade from a table `employees(employee_id, job_grade)`. The `employee_id` passed in must also be a positive number — negative or zero IDs should never be looked up at all, since they're clearly invalid input, and this should be treated as a distinctly different situation from "employee not found."

---

### Problem 2 — Inventory Deduction
**Business context:** A warehouse system needs a procedure `deduct_stock` that, given a `product_id` and a `quantity_requested`, reduces the stock in a table `inventory(product_id, stock_quantity)`. The deduction must **not** be allowed to bring stock below zero — this is a hard company policy, not something the database schema itself enforces (no CHECK constraint exists). Also consider: what should happen if the `product_id` doesn't exist at all?

---

### Problem 3 — Percentage-Based Calculation
**Business context:** Finance wants a function `calculate_growth_rate` that, given a `previous_value` and a `current_value`, returns the percentage growth: `((current_value - previous_value) / previous_value) * 100`. Think carefully about what happens if `previous_value` is zero — is this a business rule issue, or something else entirely?

---

### Problem 4 — Duplicate Registration Check
**Business context:** A customer-facing system needs a procedure `register_customer` that inserts a new row into `customers(customer_id, email)`, where `email` has a unique constraint. The business wants a clear, specific response when someone tries to register with an email that's already in use — distinctly different from any other kind of failure.

---

### Problem 5 — Multi-Rule Approval Workflow
**Business context:** A procedure `approve_expense_claim` accepts a `claim_id` and an `approver_id`. Business rules:
- The claim must exist (assume table `expense_claims(claim_id, amount, status)`).
- The claim's current status must be `'PENDING'` — approving an already-approved or already-rejected claim should be explicitly rejected with a clear message different from "claim not found."
- The claim amount cannot exceed 10,000 without a **senior approver** — assume a table `approvers(approver_id, is_senior)` where `is_senior` is `'Y'`/`'N'`; if the amount exceeds 10,000 and the approver isn't senior, this should be its own distinctly-handled situation.

**Task:** This one has multiple genuinely different failure conditions. Identify all of them first, in writing, before attempting any code — decide which are user-defined exceptions and design clear, distinct names for each.

---

### Problem 6 — Spot the Design Flaw
**Business context:** A junior developer wrote this procedure and wants your review:

```sql
CREATE OR REPLACE PROCEDURE process_refund (p_order_id IN NUMBER, p_refund_amount IN NUMBER)
IS
    v_order_total NUMBER;
BEGIN
    SELECT order_total INTO v_order_total FROM orders WHERE order_id = p_order_id;

    IF p_refund_amount > v_order_total THEN
        DBMS_OUTPUT.PUT_LINE('Refund amount cannot exceed order total.');
    ELSE
        UPDATE orders SET refunded_amount = p_refund_amount WHERE order_id = p_order_id;
    END IF;

EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('An error occurred.');
END process_refund;
/
```

**Task:** Critically review this. There are at least **two** meaningful design problems here related directly to what you've learned in this module (Topics 1–3) — one involving how the "refund exceeds total" rule is communicated, and one involving how `WHEN OTHERS` is being used. Identify both and rewrite the procedure properly.

---

## Notes Before You Attempt These

- As with the procedures checkpoint, these are intentionally less guided — no concept tags telling you "this is NO_DATA_FOUND" or "this is user-defined." Reading the business language and making that call yourself is the entire point.
- Problem 5 is the most involved — it's meant to mirror a genuinely realistic multi-rule approval workflow you might actually be handed at a job. Take your time with the design step before coding.
- It's fine to work through these one at a time.

---

*Share your attempts whenever you're ready — all at once or one by one — and I'll review them, covering both correctness and design judgment (like exception naming and separation of concerns). After that, we'll move to the final topic in this module: Raise Application Error.*
