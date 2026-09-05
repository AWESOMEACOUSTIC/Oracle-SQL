# Mixed Assessment: Modules 3 & 4 (Procedures, Functions, Packages, Exceptions)

> This assessment covers **everything you've learned so far** — Module 3 (Sub Programs, Procedures, Functions, Result Cache, Notation, Packages, Overloading) and Module 4 (Exception Handling). Questions are **not labeled** with the concept being tested — part of the assessment is correctly identifying what's relevant from the requirement itself, exactly as you'd need to in real work.
>
> Question types are mixed deliberately: some ask you to design/write code, some ask you to predict behavior, some ask you to debug flawed code, and some ask for judgment/reasoning in prose. Take your time — there's no need to rush through all of them in one sitting.

---

## Section A — Design & Build

### A1.
A logistics company needs a reusable module for tracking delivery vehicles. Requirements:
- Register a new vehicle (`vehicle_id`, `capacity_kg`) into a `vehicles` table — reject registration if `capacity_kg` is zero or negative, with a clear error the calling fleet-management application can catch and display to a dispatcher.
- Given a `vehicle_id`, report its current utilization percentage (`current_load_kg / capacity_kg * 100`) — this will be used inside a live dashboard report that queries many vehicles at once.
- An internal calculation — determining whether a vehicle needs maintenance based on `total_distance_km` exceeding a threshold — should not be something other systems can call directly; it exists purely to support one of the above operations in a future extension.

Design and write the appropriate PL/SQL object(s) for this. Justify your structural choices (procedure vs. function, public vs. private, standalone vs. grouped) in a few sentences before or after your code.

---

### A2.
A retail company wants a **single, one-time analytical query** (not something that will ever be reused) that lists every order from the past 30 days alongside a computed label: `'RUSH'` if the order was placed and delivered within 24 hours, `'NORMAL'` otherwise. This is for a one-off meeting next week and will never be run again after that.

Write the query, using whatever mechanism from this module best fits a genuinely one-off need like this. Briefly justify why you didn't create a permanent, reusable object instead.

---

### A3.
A finance team has a lookup: given a `tax_jurisdiction_code`, return the current tax rate. This is called extremely often (thousands of times per hour across many reports and transactions), and tax rates for any given jurisdiction rarely change — maybe once or twice a year.

Design this as a PL/SQL function, applying any relevant performance techniques you've learned. Explain your reasoning for any techniques applied, and note one scenario where applying that same technique would have been a mistake.

---

## Section B — Predict the Behavior

### B1.
Without running it, describe **exactly** what this block prints, line by line, and why:
```sql
DECLARE
    v_result NUMBER;
BEGIN
    DBMS_OUTPUT.PUT_LINE('A');
    SELECT 100 / 0 INTO v_result FROM dual;
    DBMS_OUTPUT.PUT_LINE('B');
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('C');
    WHEN ZERO_DIVIDE THEN
        DBMS_OUTPUT.PUT_LINE('D');
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('E');
END;
/
```

### B2.
Two sessions, Session X and Session Y, both connect and interact with this package:
```sql
CREATE OR REPLACE PACKAGE pkg_counter
IS
    PROCEDURE bump;
    FUNCTION current_value RETURN NUMBER;
END pkg_counter;
/

CREATE OR REPLACE PACKAGE BODY pkg_counter
IS
    g_val NUMBER;
    PROCEDURE bump IS BEGIN g_val := g_val + 1; END;
    FUNCTION current_value RETURN NUMBER IS BEGIN RETURN g_val; END;
BEGIN
    g_val := 0;
END pkg_counter;
/
```
Session X calls `pkg_counter.bump;` five times, then calls `pkg_counter.current_value`. Session Y, a completely separate connection that has never touched this package before, then calls `pkg_counter.current_value` for the first time. What does each session see, and why?

### B3.
Given this procedure:
```sql
CREATE OR REPLACE PROCEDURE risky_update (p_id IN NUMBER, p_amount IN NUMBER)
IS
BEGIN
    UPDATE accounts SET balance = balance + p_amount WHERE account_id = p_id;
END risky_update;
/
```
It's called as `risky_update(99999, 500);` where `account_id = 99999` does not exist. Does this procedure raise `NO_DATA_FOUND`? What actually happens? Explain the underlying rule.

---

## Section C — Debug It

### C1.
This function is supposed to classify a customer's order volume. It compiles, but fails at runtime for certain valid inputs. Identify the bug and fix it.
```sql
CREATE OR REPLACE FUNCTION classify_volume (p_order_count NUMBER)
    RETURN VARCHAR2
IS
BEGIN
    IF p_order_count > 100 THEN
        RETURN 'HIGH';
    ELSIF p_order_count > 20 THEN
        RETURN 'MEDIUM';
    END IF;
END classify_volume;
/
```

### C2.
A developer wrote this package attempting to overload a validation routine:
```sql
CREATE OR REPLACE PACKAGE pkg_validate
IS
    PROCEDURE check_input (p_value IN NUMBER);
    PROCEDURE check_input (p_value OUT NUMBER);
END pkg_validate;
/
```
This fails to compile. Explain precisely why, and propose a corrected design that achieves something close to the developer's likely intent (validating either an incoming value or producing one).

### C3.
```sql
CREATE OR REPLACE PROCEDURE finalize_invoice (p_invoice_id IN NUMBER)
IS
    v_total NUMBER;
BEGIN
    SELECT total_amount INTO v_total FROM invoices WHERE invoice_id = p_invoice_id;

    IF v_total > 1000000 THEN
        RAISE_APPLICATION_ERROR(500001, 'Invoice amount unusually large — manual review required.');
    END IF;

    UPDATE invoices SET status = 'FINALIZED' WHERE invoice_id = p_invoice_id;
END finalize_invoice;
/
```
This raises an unexpected error the moment it's compiled/run, unrelated to the business logic itself. Identify the issue and fix it.

---

## Section D — Judgment & Reasoning (Short Answer)

### D1.
A teammate says: *"I always just use `WHEN OTHERS THEN NULL;` at the end of every procedure — it guarantees nothing ever crashes, so it's the safest option."* Respond to this in 4–6 sentences, addressing both what's appealing about the idea and what's genuinely risky about it.

### D2.
Explain, in your own words, the difference between what happens when a `SELECT INTO` matches zero rows versus what happens when an `UPDATE` matches zero rows. Why does this distinction matter when designing exception handling for a batch process?

### D3.
A procedure `pkg_hr.terminate_employee` currently has this signature:
```sql
PROCEDURE terminate_employee (p_emp_id IN NUMBER, p_termination_date IN DATE);
```
The company now wants to add an optional reason code, `p_reason_code IN VARCHAR2 DEFAULT NULL`, **without breaking any of the ~30 existing call sites** across the codebase (a mix of positional and named notation calls). Is this safe? Under what conditions would it *not* be safe, and for which of the 30 call sites specifically?

---

## Section E — Realistic Business Case (Multi-Part)

### E1.
A subscription-based company has the following requirement:

> "We need a module to handle subscription renewals. When a subscription is renewed: look up the customer's current plan and its monthly price. Apply a loyalty discount based on how many consecutive months they've been subscribed (this calculation is a simple internal lookup, not something other systems need directly). Calculate the final renewal amount. Insert a renewal record. If the customer's account is marked as `'SUSPENDED'`, the renewal must be rejected outright with a clear, specific error distinguishable from other failure types — this needs to be understood by both our billing team's internal PL/SQL scripts and our external payment gateway integration, which expects a proper database error it can catch. If the customer doesn't exist at all, that's a separate, distinctly different situation that should also be clearly communicated. Under high load, this renewal process runs for thousands of customers nightly, and one customer's issue must never stop the batch from continuing to the next customer (assume the batch-looping mechanism itself is handled elsewhere — focus only on this individual renewal module's design)."

**Your task:**
1. Identify every distinct PL/SQL concept from Modules 3 and 4 that's relevant here (don't just list module topics — explain *where* in the requirement each one applies).
2. Design the object(s) needed: what's public, what's private, procedure vs. function, and what exceptions (pre-defined and/or user-defined) are needed, with what distinct error codes if applicable.
3. Write the full implementation (specification + body, or standalone objects, whichever your design calls for).
4. In a short paragraph, explain how your design satisfies the "one customer's issue must never stop the batch" requirement — even though the looping mechanism itself is out of scope.

---

## A Note on This Assessment

Some of these questions (particularly in Sections C and E) are intentionally a little ambiguous or require you to make a reasonable judgment call, since real requirements aren't always perfectly specified either. Where you make an assumption, just state it briefly — that's part of good practice, not a weakness in your answer.

---

*Take your time with this. Share your answers whenever you're ready — all sections at once, or section by section, whichever is easier for you to work through. I'll review each answer, point out anything missed, and we'll calibrate before deciding whether to move on to Module 1 & 2, or increase the difficulty for a future assessment.*
