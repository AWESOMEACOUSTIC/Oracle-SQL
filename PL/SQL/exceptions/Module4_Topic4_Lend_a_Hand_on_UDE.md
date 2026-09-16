# Module 4, Topic 4: Lend a Hand on User Defined Exceptions
### Practice Checkpoint

This is a **hands-on consolidation session**, not a new theory topic — same format as the "Lend a Hand on Procedures" checkpoint in Module 3. It stretches everything from Topics 1–3 (exception mechanism, pre-defined exceptions, user-defined exceptions) through realistic, less-guided business problems, before we move into `RAISE_APPLICATION_ERROR`.

> **Updated edition:** all six problems now share **one central set of tables**. Run the schema once, and every problem below plugs straight into it — no more spinning up a fresh throwaway table per scenario. An Answer Key with fully worked solutions and design rationale is included at the end so you can self-check after attempting each problem.

---

## Table of Contents

1. [How to Use This Session](#how-to-use-this-session)
2. [Central Schema — Run This Once](#central-schema--run-this-once)
3. [Practice Problems](#practice-problems)
   - [Problem 1 — Employee Lookup with Validation](#problem-1--employee-lookup-with-validation)
   - [Problem 2 — Inventory Deduction](#problem-2--inventory-deduction)
   - [Problem 3 — Percentage-Based Calculation](#problem-3--percentage-based-calculation)
   - [Problem 4 — Duplicate Registration Check](#problem-4--duplicate-registration-check)
   - [Problem 5 — Multi-Rule Approval Workflow](#problem-5--multi-rule-approval-workflow)
   - [Problem 6 — Spot the Design Flaw](#problem-6--spot-the-design-flaw)
4. [Notes Before You Attempt These](#notes-before-you-attempt-these)
5. [✅ Answer Key](#-answer-key)
6. [A Note on What's Not Here Yet](#a-note-on-whats-not-here-yet)

---

## How to Use This Session

For each problem, work through this discipline **before writing code**:

1. What is the business requirement actually asking for?
2. What could realistically go wrong — both generic runtime issues (missing data, bad math, duplicates) and business rule violations (limits, policies, invalid states)?
3. For each thing that could go wrong: is it something Oracle already recognizes (→ **pre-defined exception**) or something only your business logic knows about (→ **user-defined exception**)?
4. Where exactly does the `RAISE` belong for any user-defined exception — i.e., what's the precise condition that triggers it?
5. How should each distinct problem be handled differently — do they all deserve the same generic response, or does the business context call for different messages/actions?
6. **Only then**, write the code.

> As with the procedures checkpoint, I will not tell you upfront which specific exceptions (pre-defined vs. user-defined) apply to each problem — identifying that is the core skill being tested here.

---

## Central Schema — Run This Once

Every problem in this checkpoint reuses these same five tables. Create and seed them once, and you're set for the whole session.

```sql
-- ============================================================
--  CORE SCHEMA — shared across every problem in this checkpoint
-- ============================================================

CREATE TABLE employees (
    employee_id         NUMBER PRIMARY KEY,
    employee_name        VARCHAR2(100),
    job_grade            VARCHAR2(10),
    is_senior_approver   CHAR(1) DEFAULT 'N' CHECK (is_senior_approver IN ('Y','N'))
);

CREATE TABLE products (
    product_id      NUMBER PRIMARY KEY,
    product_name    VARCHAR2(100),
    stock_quantity  NUMBER NOT NULL
);

CREATE TABLE customers (
    customer_id     NUMBER PRIMARY KEY,
    customer_name   VARCHAR2(100),
    email           VARCHAR2(100) UNIQUE
);

CREATE TABLE orders (
    order_id          NUMBER PRIMARY KEY,
    customer_id       NUMBER REFERENCES customers(customer_id),
    order_total       NUMBER NOT NULL,
    refunded_amount   NUMBER DEFAULT 0
);

CREATE TABLE expense_claims (
    claim_id      NUMBER PRIMARY KEY,
    employee_id   NUMBER REFERENCES employees(employee_id),
    amount        NUMBER NOT NULL,
    status        VARCHAR2(20) DEFAULT 'PENDING'
);

-- ============================================================
--  SEED DATA
-- ============================================================

INSERT INTO employees VALUES (101, 'Asha Rao',    'G5', 'N');
INSERT INTO employees VALUES (102, 'Vikram Shah', 'G7', 'Y');
INSERT INTO employees VALUES (103, 'Neha Iyer',   'G4', 'N');

INSERT INTO products VALUES (201, 'Wireless Mouse',      50);
INSERT INTO products VALUES (202, 'Mechanical Keyboard',  5);
INSERT INTO products VALUES (203, 'USB-C Hub',            0);

INSERT INTO customers VALUES (301, 'Ramesh Kumar', 'ramesh.kumar@example.com');
INSERT INTO customers VALUES (302, 'Priya Menon',  'priya.menon@example.com');

INSERT INTO orders VALUES (401, 301, 5000,  0);
INSERT INTO orders VALUES (402, 302, 12000, 0);
INSERT INTO orders VALUES (403, 301, 3000,  500);

INSERT INTO expense_claims VALUES (501, 101, 4500,  'PENDING');
INSERT INTO expense_claims VALUES (502, 102, 15000, 'PENDING');
INSERT INTO expense_claims VALUES (503, 103, 2000,  'APPROVED');

COMMIT;
```

### Which Problem Uses Which Table(s)

| Table | Used in Problem(s) |
|---|---|
| `employees` | 1, 5 *(also doubles as the approvers list — `is_senior_approver` replaces the separate approvers table)* |
| `products` | 2 |
| `customers` | 4 |
| `orders` | 6 |
| `expense_claims` | 5 |

> Problem 3 is intentionally left out of this table — see the note in that problem for why.

---

## Practice Problems

### Problem 1 — Employee Lookup with Validation

**Business context:** HR wants a procedure `get_employee_grade` that, given an `employee_id`, returns (via `OUT` parameter) the employee's job grade from the central `employees` table. The `employee_id` passed in must also be a **positive number** — negative or zero IDs should never be looked up at all, since they're clearly invalid input, and this should be treated as a **distinctly different situation** from "employee not found."

---

### Problem 2 — Inventory Deduction

**Business context:** A warehouse system needs a procedure `deduct_stock` that, given a `product_id` and a `quantity_requested`, reduces the stock in the central `products` table (`stock_quantity` column). The deduction must **not** be allowed to bring stock below zero — this is a hard company policy, not something the database schema itself enforces (no `CHECK` constraint exists on this column). Also consider: what should happen if the `product_id` doesn't exist at all?

---

### Problem 3 — Percentage-Based Calculation

**Business context:** Finance wants a function `calculate_growth_rate` that, given a `previous_value` and a `current_value`, returns the percentage growth:

$$\left(\frac{current\_value - previous\_value}{previous\_value}\right) \times 100$$

Think carefully about what happens if `previous_value` is zero — is this a business rule issue, or something else entirely?

> **Why no central table here:** this one is a pure calculation utility — it doesn't look anything up, it just operates on the two numbers it's handed. There's genuinely nothing to consolidate; forcing it onto `orders` or any other table would just bolt on an unrelated lookup step and dilute the actual lesson (which is about spotting a pre-defined arithmetic exception). It's left as-is on purpose.

---

### Problem 4 — Duplicate Registration Check

**Business context:** A customer-facing system needs a procedure `register_customer` that inserts a new row into the central `customers` table (`email` has a unique constraint). The business wants a clear, specific response when someone tries to register with an email that's already in use — distinctly different from any other kind of failure.

---

### Problem 5 — Multi-Rule Approval Workflow

**Business context:** A procedure `approve_expense_claim` accepts a `claim_id` and an `approver_id`. Business rules:

- The claim must **exist** (in the central `expense_claims` table).
- The claim's current status must be `'PENDING'` — approving an already-approved or already-rejected claim should be explicitly rejected with a clear message different from "claim not found."
- The claim amount **cannot exceed 10,000** without a senior approver. There's no separate approvers table anymore — the central `employees` table's `is_senior_approver` column (`'Y'`/`'N'`) tells you this; the `approver_id` you're given is an `employee_id`. If the amount exceeds 10,000 and the approver isn't senior, this should be its own distinctly-handled situation.

**Task:** This one has multiple genuinely different failure conditions. Identify all of them first, in writing, before attempting any code — decide which are user-defined exceptions and design clear, distinct names for each.

> 💡 **Hint:** think about what happens if the `approver_id` itself doesn't correspond to any real employee — is that the same failure as "claim not found"?

---

### Problem 6 — Spot the Design Flaw

**Business context:** A junior developer wrote this procedure and wants your review. It uses the central `orders` table (`order_total`, `refunded_amount`):

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

**Task:** Critically review this. There are at least **two meaningful design problems** here related directly to what you've learned in this module (Topics 1–3) — one involving how the "refund exceeds total" rule is communicated, and one involving how `WHEN OTHERS` is being used. Identify both and rewrite the procedure properly.

---

## Notes Before You Attempt These

- As with the procedures checkpoint, these are intentionally less guided — no concept tags telling you "this is `NO_DATA_FOUND`" or "this is user-defined." Reading the business language and making that call yourself is the entire point.
- Problem 5 is the most involved — it's meant to mirror a genuinely realistic multi-rule approval workflow you might actually be handed at a job. Take your time with the design step before coding.
- It's fine to work through these one at a time.
- Don't peek at the Answer Key below until you've written your own attempt — that's the only way this checkpoint actually does its job.

Share your attempts whenever you're ready — all at once or one by one — and I'll review them, covering both correctness and design judgment (like exception naming and separation of concerns). After that, we'll move to the final topic in this module: **Raise Application Error**.

---

## ✅ Answer Key

> Work through each problem yourself first. Solutions and the reasoning behind each design choice are below, in order.

### Problem 1 — Answer

```sql
CREATE OR REPLACE PROCEDURE get_employee_grade (
    p_employee_id   IN  employees.employee_id%TYPE,
    o_job_grade     OUT employees.job_grade%TYPE
)
IS
    e_invalid_employee_id EXCEPTION;
BEGIN
    IF p_employee_id <= 0 THEN
        RAISE e_invalid_employee_id;
    END IF;

    SELECT job_grade
      INTO o_job_grade
      FROM employees
     WHERE employee_id = p_employee_id;

EXCEPTION
    WHEN e_invalid_employee_id THEN
        DBMS_OUTPUT.PUT_LINE('Error: employee_id must be a positive number. Received: ' || p_employee_id);
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Error: No employee found with ID ' || p_employee_id);
END get_employee_grade;
/
```

**Why:**

| Failure | Exception Type | Reasoning |
|---|---|---|
| Invalid ID (`<= 0`) | User-defined (`e_invalid_employee_id`) | Oracle has no idea what a "valid employee ID" means to this business; only this procedure's rules define that. Note the check happens **before** the query, so a bad ID never even reaches the database, matching "should never be looked up at all." |
| Employee not found | Pre-defined `NO_DATA_FOUND` | A `SELECT ... INTO` returning zero rows is something Oracle already recognizes and raises automatically; no need to reinvent it. |

---

### Problem 2 — Answer

```sql
CREATE OR REPLACE PROCEDURE deduct_stock (
    p_product_id         IN products.product_id%TYPE,
    p_quantity_requested IN NUMBER
)
IS
    v_current_stock       products.stock_quantity%TYPE;
    e_insufficient_stock  EXCEPTION;
BEGIN
    SELECT stock_quantity
      INTO v_current_stock
      FROM products
     WHERE product_id = p_product_id;

    IF v_current_stock - p_quantity_requested < 0 THEN
        RAISE e_insufficient_stock;
    END IF;

    UPDATE products
       SET stock_quantity = stock_quantity - p_quantity_requested
     WHERE product_id = p_product_id;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Error: No product found with ID ' || p_product_id);
    WHEN e_insufficient_stock THEN
        DBMS_OUTPUT.PUT_LINE('Error: Cannot deduct ' || p_quantity_requested ||
                              ' units — only ' || v_current_stock || ' in stock for product ' || p_product_id);
END deduct_stock;
/
```

**Why:**

| Failure | Exception Type | Reasoning |
|---|---|---|
| Product doesn't exist | Pre-defined `NO_DATA_FOUND` | Same reasoning as Problem 1. |
| Deduction would push stock negative | User-defined (`e_insufficient_stock`) | The problem statement is explicit that this is a company policy with no schema-level enforcement (no `CHECK` constraint) — that's exactly the signal that it's business logic Oracle has no built-in awareness of. |

---

### Problem 3 — Answer

```sql
CREATE OR REPLACE FUNCTION calculate_growth_rate (
    p_previous_value IN NUMBER,
    p_current_value  IN NUMBER
) RETURN NUMBER
IS
BEGIN
    RETURN ((p_current_value - p_previous_value) / p_previous_value) * 100;

EXCEPTION
    WHEN ZERO_DIVIDE THEN
        DBMS_OUTPUT.PUT_LINE('Error: previous_value is zero — growth rate is undefined.');
        RETURN NULL;
END calculate_growth_rate;
/
```

**Why:** `previous_value = 0` is **not** a business rule violation at all — it's a mathematical impossibility, and division by zero is something Oracle already recognizes and raises as the pre-defined `ZERO_DIVIDE` exception.

> ⚠️ **The trap in this problem:** it's tempting to reach for a user-defined exception here (since it "feels" like validation), but the moment something is a generic runtime condition Oracle itself detects, it doesn't need a user-defined wrapper.

---

### Problem 4 — Answer

```sql
CREATE OR REPLACE PROCEDURE register_customer (
    p_customer_id   IN customers.customer_id%TYPE,
    p_customer_name IN customers.customer_name%TYPE,
    p_email         IN customers.email%TYPE
)
IS
BEGIN
    INSERT INTO customers (customer_id, customer_name, email)
    VALUES (p_customer_id, p_customer_name, p_email);

EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
        DBMS_OUTPUT.PUT_LINE('Error: The email "' || p_email || '" is already registered.');
END register_customer;
/
```

**Why:** the `email` column already has a unique constraint, so a duplicate insert is something Oracle detects on its own — that's the pre-defined `DUP_VAL_ON_INDEX` exception. No user-defined exception is needed; the "clear, specific response" the business wants is achieved just by giving this pre-defined exception its own handler with a tailored message, rather than lumping it into `WHEN OTHERS`.

---

### Problem 5 — Answer

```sql
CREATE OR REPLACE PROCEDURE approve_expense_claim (
    p_claim_id    IN expense_claims.claim_id%TYPE,
    p_approver_id IN employees.employee_id%TYPE
)
IS
    v_claim_amount    expense_claims.amount%TYPE;
    v_claim_status    expense_claims.status%TYPE;
    v_is_senior       employees.is_senior_approver%TYPE;
    v_claim_count     NUMBER;
    v_approver_count  NUMBER;

    e_claim_not_found          EXCEPTION;
    e_approver_not_found       EXCEPTION;
    e_claim_not_pending        EXCEPTION;
    e_senior_approval_required EXCEPTION;
BEGIN
    -- 1. Claim must exist
    SELECT COUNT(*) INTO v_claim_count FROM expense_claims WHERE claim_id = p_claim_id;
    IF v_claim_count = 0 THEN
        RAISE e_claim_not_found;
    END IF;

    SELECT amount, status
      INTO v_claim_amount, v_claim_status
      FROM expense_claims
     WHERE claim_id = p_claim_id;

    -- 2. Claim must be PENDING
    IF v_claim_status != 'PENDING' THEN
        RAISE e_claim_not_pending;
    END IF;

    -- 3. Approver must exist
    SELECT COUNT(*) INTO v_approver_count FROM employees WHERE employee_id = p_approver_id;
    IF v_approver_count = 0 THEN
        RAISE e_approver_not_found;
    END IF;

    SELECT is_senior_approver INTO v_is_senior FROM employees WHERE employee_id = p_approver_id;

    -- 4. Large claims need a senior approver
    IF v_claim_amount > 10000 AND v_is_senior != 'Y' THEN
        RAISE e_senior_approval_required;
    END IF;

    UPDATE expense_claims SET status = 'APPROVED' WHERE claim_id = p_claim_id;

EXCEPTION
    WHEN e_claim_not_found THEN
        DBMS_OUTPUT.PUT_LINE('Error: No expense claim found with ID ' || p_claim_id);
    WHEN e_claim_not_pending THEN
        DBMS_OUTPUT.PUT_LINE('Error: Claim ' || p_claim_id || ' is already ' || v_claim_status || ' — cannot approve again.');
    WHEN e_approver_not_found THEN
        DBMS_OUTPUT.PUT_LINE('Error: No approver found with ID ' || p_approver_id);
    WHEN e_senior_approval_required THEN
        DBMS_OUTPUT.PUT_LINE('Error: Claim amount ' || v_claim_amount || ' exceeds 10,000 and requires a senior approver.');
END approve_expense_claim;
/
```

**Why — four distinct failure conditions:**

| # | Failure | Exception Type | Reasoning |
|---|---|---|---|
| 1 | Claim not found | User-defined (see design note below) | Genuinely a "not found" case, but implemented as user-defined here rather than relying on `NO_DATA_FOUND`. |
| 2 | Claim not `PENDING` | User-defined | Pure business rule. Oracle has no concept of a claim "lifecycle"; only this procedure knows `'APPROVED'`/`'REJECTED'` claims shouldn't be re-approved. |
| 3 | Approver not found | User-defined (see design note below) | Same reasoning as #1. |
| 4 | Amount > 10,000 without a senior approver | User-defined | Pure business rule; nothing about this is a generic database condition. |

> **Design note — why `COUNT(*)` + user-defined exceptions instead of `NO_DATA_FOUND` for #1 and #3:**
> This procedure has two separate lookups that could each fail to find a row (the claim, and the approver). If both simply let `NO_DATA_FOUND` propagate, the caller would get the exact same exception for two completely different problems — there'd be no way to tell "bad claim ID" apart from "bad approver ID" from the exception itself. Checking with `SELECT COUNT(*)` first and raising a named exception for each case is what makes those two failures distinctly handleable, which is exactly what the business asked for.
>
> **General pattern to remember:** reach for a pre-defined exception when there's only one place it can come from; convert to an explicit check + user-defined exception once ambiguity like this shows up.

---

### Problem 6 — Answer

**The two design flaws:**

1. **The "refund exceeds total" rule isn't actually an exception.** It's just a `DBMS_OUTPUT.PUT_LINE` inside a plain `IF`/`ELSE`. That means the caller of this procedure has no reliable, structured way to detect that the operation failed — no exception was raised, so from the caller's perspective the procedure "succeeded" even though nothing happened. This should be a proper user-defined exception, raised and handled like any other business rule violation.
2. **`WHEN OTHERS` is being used as a silent catch-all.** It swallows everything — including `NO_DATA_FOUND` if `p_order_id` doesn't exist at all — behind one generic, useless message ("An error occurred."), with no distinction between failure types and no diagnostic information (no `SQLCODE`/`SQLERRM`, nothing re-raised). This hides real problems instead of surfacing them, and it means a completely different failure (invalid order) gets the same message as any other failure.

**Corrected procedure:**

```sql
CREATE OR REPLACE PROCEDURE process_refund (
    p_order_id      IN orders.order_id%TYPE,
    p_refund_amount IN NUMBER
)
IS
    v_order_total           orders.order_total%TYPE;
    e_refund_exceeds_total  EXCEPTION;
BEGIN
    SELECT order_total
      INTO v_order_total
      FROM orders
     WHERE order_id = p_order_id;

    IF p_refund_amount > v_order_total THEN
        RAISE e_refund_exceeds_total;
    END IF;

    UPDATE orders
       SET refunded_amount = p_refund_amount
     WHERE order_id = p_order_id;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Error: No order found with ID ' || p_order_id);
    WHEN e_refund_exceeds_total THEN
        DBMS_OUTPUT.PUT_LINE('Error: Refund amount ' || p_refund_amount ||
                              ' cannot exceed order total ' || v_order_total || ' for order ' || p_order_id);
END process_refund;
/
```

**Why:** order not found is pre-defined `NO_DATA_FOUND` (now explicitly handled instead of being buried in `WHEN OTHERS`); refund exceeding the order total is a user-defined exception (`e_refund_exceeds_total`) since it's a business rule Oracle has no awareness of. Both now have their own named handler with a specific, useful message — and there's no `WHEN OTHERS` left to accidentally mask either of them.

---

## A Note on What's Not Here Yet

None of these answers use `RAISE_APPLICATION_ERROR` — that's deliberate. This checkpoint only covers Topics 1–3 (the exception mechanism, pre-defined exceptions, and user-defined exceptions), and `RAISE_APPLICATION_ERROR` is the next topic in this module. Once we get there, you'll revisit these same six procedures and see how it changes (and improves) the way these errors get communicated back to a caller.