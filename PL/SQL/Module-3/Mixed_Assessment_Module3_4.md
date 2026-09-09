# Modules 3 & 4 — Sample Tables, Answer Queries & Expected Output

Same format as before: sample schema + seeded data, the PL/SQL solution, and the traced output. A few of these questions (Section B2, Section C2, Section D) are about structure or session behavior rather than table data — those are noted rather than padded with an irrelevant table.

---

## A1 — Delivery Vehicle Tracking Module

### Sample table
```sql
CREATE TABLE vehicles (
    vehicle_id        NUMBER PRIMARY KEY,
    capacity_kg        NUMBER,
    current_load_kg    NUMBER DEFAULT 0,
    total_distance_km  NUMBER DEFAULT 0
);

INSERT INTO vehicles VALUES (1, 5000, 3000, 12000);
INSERT INTO vehicles VALUES (2, 8000, 7900, 50000);
INSERT INTO vehicles VALUES (3, 2000, 500,  500);
COMMIT;
```

### Answer query
```sql
CREATE OR REPLACE PACKAGE pkg_vehicle_tracking
IS
    PROCEDURE register_vehicle (p_vehicle_id IN vehicles.vehicle_id%TYPE, p_capacity_kg IN NUMBER);
    FUNCTION  get_utilization_pct (p_vehicle_id IN vehicles.vehicle_id%TYPE) RETURN NUMBER;
END pkg_vehicle_tracking;
/

CREATE OR REPLACE PACKAGE BODY pkg_vehicle_tracking
IS
    -- Private: pure internal detail, exists only to support a future extension
    FUNCTION needs_maintenance (
        p_total_distance_km IN NUMBER,
        p_threshold_km      IN NUMBER DEFAULT 100000
    ) RETURN BOOLEAN
    IS
    BEGIN
        RETURN p_total_distance_km >= p_threshold_km;
    END needs_maintenance;

    PROCEDURE register_vehicle (p_vehicle_id IN vehicles.vehicle_id%TYPE, p_capacity_kg IN NUMBER)
    IS
    BEGIN
        IF p_capacity_kg <= 0 THEN
            RAISE_APPLICATION_ERROR(-20010, 'Invalid capacity: must be positive. Got ' || p_capacity_kg);
        END IF;

        INSERT INTO vehicles (vehicle_id, capacity_kg, current_load_kg, total_distance_km)
        VALUES (p_vehicle_id, p_capacity_kg, 0, 0);

        DBMS_OUTPUT.PUT_LINE('Vehicle ' || p_vehicle_id || ' registered.');
    END register_vehicle;

    FUNCTION get_utilization_pct (p_vehicle_id IN vehicles.vehicle_id%TYPE) RETURN NUMBER
    IS
        v_load NUMBER;
        v_cap  NUMBER;
    BEGIN
        SELECT current_load_kg, capacity_kg INTO v_load, v_cap
          FROM vehicles WHERE vehicle_id = p_vehicle_id;

        RETURN ROUND((v_load / v_cap) * 100, 2);
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            RAISE_APPLICATION_ERROR(-20011, 'Vehicle ' || p_vehicle_id || ' not found.');
    END get_utilization_pct;
END pkg_vehicle_tracking;
/
```
> Design notes: a **package** is required (not standalone objects) because `needs_maintenance` must be private and grouped with the public operations. `register_vehicle` is a procedure (performs DML, returns nothing but a distinct error). `get_utilization_pct` is a function (returns one computed value per call, which is what a dashboard iterating over vehicles needs). No `RESULT_CACHE` here — unlike a rarely-changing lookup, `current_load_kg` changes constantly as vehicles load/unload, so caching would show a dispatcher stale utilization.

### Test & expected output
```sql
BEGIN pkg_vehicle_tracking.register_vehicle(4, 3000); END; /   -- ok
BEGIN pkg_vehicle_tracking.register_vehicle(5, -100); END; /   -- rejected

DBMS_OUTPUT.PUT_LINE(pkg_vehicle_tracking.get_utilization_pct(1));   -- 60
DBMS_OUTPUT.PUT_LINE(pkg_vehicle_tracking.get_utilization_pct(2));   -- 98.75
DBMS_OUTPUT.PUT_LINE(pkg_vehicle_tracking.get_utilization_pct(3));   -- 25
DBMS_OUTPUT.PUT_LINE(pkg_vehicle_tracking.get_utilization_pct(999)); -- not found
```
```
Vehicle 4 registered.
ORA-20010: Invalid capacity: must be positive. Got -100
60
98.75
25
ORA-20011: Vehicle 999 not found.
```

---

## A2 — One-Off "RUSH vs NORMAL" Order Report

### Sample table
```sql
CREATE TABLE orders (
    order_id       NUMBER PRIMARY KEY,
    order_date     DATE,
    delivered_date DATE
);

-- "today" for this walkthrough is 2026-09-09, so the 30-day window is Aug 10 - Sep 9
INSERT INTO orders VALUES (9001, DATE '2026-08-20', DATE '2026-08-20' + 10/24); -- 10 hrs later
INSERT INTO orders VALUES (9002, DATE '2026-08-25', DATE '2026-08-27');          -- 2 days later
INSERT INTO orders VALUES (9003, DATE '2026-09-01', DATE '2026-09-01' + 23/24); -- 23 hrs later
INSERT INTO orders VALUES (9004, DATE '2026-09-05', NULL);                       -- not yet delivered
INSERT INTO orders VALUES (9005, DATE '2026-07-01', DATE '2026-07-01' + 1/24);   -- outside 30 days
COMMIT;
```

### Answer query
```sql
-- Deliberately plain SQL — no PL/SQL object at all. There's no procedural logic
-- here that a SELECT + CASE expression can't already handle, and creating a
-- stored function would leave a permanent object behind for a query that's
-- never meant to run again after next week's meeting.
SELECT order_id,
       order_date,
       delivered_date,
       CASE
           WHEN delivered_date IS NOT NULL
                AND (delivered_date - order_date) <= 1   -- within 24 hours
           THEN 'RUSH'
           ELSE 'NORMAL'
       END AS speed_label
  FROM orders
 WHERE order_date >= TRUNC(SYSDATE) - 30
 ORDER BY order_date;
```

### Expected output
```
ORDER_ID  ORDER_DATE   DELIVERED_DATE        SPEED_LABEL
--------  -----------  --------------------  -----------
9001      2026-08-20   2026-08-20 10:00:00   RUSH
9002      2026-08-25   2026-08-27 00:00:00   NORMAL
9003      2026-09-01   2026-09-01 23:00:00   RUSH
9004      2026-09-05   (null)                NORMAL
```
(Order 9005 is correctly excluded — it's outside the 30-day window.)

---

## A3 — Tax Jurisdiction Rate Lookup

### Sample table
```sql
CREATE TABLE tax_rates (
    tax_jurisdiction_code VARCHAR2(10) PRIMARY KEY,
    tax_rate              NUMBER
);

INSERT INTO tax_rates VALUES ('CA-ON', 0.13);
INSERT INTO tax_rates VALUES ('US-NY', 0.08);
INSERT INTO tax_rates VALUES ('US-CA', 0.0725);
COMMIT;
```

### Answer query
```sql
CREATE OR REPLACE FUNCTION get_tax_rate (
    p_code IN tax_rates.tax_jurisdiction_code%TYPE
) RETURN NUMBER
RESULT_CACHE RELIES_ON (tax_rates)
IS
    v_rate NUMBER;
BEGIN
    SELECT tax_rate INTO v_rate FROM tax_rates WHERE tax_jurisdiction_code = p_code;
    RETURN v_rate;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20020, 'Unknown tax jurisdiction: ' || p_code);
END get_tax_rate;
/
```
> **Reasoning:** thousands of calls/hour against data that changes maybe twice a year is the textbook `RESULT_CACHE` case — most calls become a cache lookup instead of a table read. `RELIES_ON (tax_rates)` means a mid-year rate correction still takes effect automatically; the cache invalidates itself rather than serving a stale rate silently.
>
> **Where this same technique would be a mistake:** any function whose result depends on something *other* than its input parameters and the referenced table — e.g. a function that also reads `SYS_CONTEXT` for the current user, checks `NLS` session settings, or otherwise varies by session. `RESULT_CACHE` is shared **across sessions**, so a value computed under one user's context could be incorrectly served to a different session. (The rate-limit "requests remaining" function from the previous assessment is the same trap in a different shape — see that answer key.)

### Test & expected output
```sql
DBMS_OUTPUT.PUT_LINE(get_tax_rate('CA-ON')); -- .13
DBMS_OUTPUT.PUT_LINE(get_tax_rate('US-NY')); -- .08
DBMS_OUTPUT.PUT_LINE(get_tax_rate('XX-99')); -- unknown jurisdiction
```
```
.13
.08
ORA-20020: Unknown tax jurisdiction: XX-99
```

---

## B1 — Predict the Behavior (ZERO_DIVIDE vs NO_DATA_FOUND)

No custom table needed — the block queries `DUAL`, which always returns exactly one row, so `NO_DATA_FOUND` can never fire here regardless of the arithmetic.

### The block (as given)
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

### Expected output
```
A
D
```
`'A'` prints first. The division `100/0` raises `ZERO_DIVIDE` *while evaluating the SELECT list*, before the row is even fetched — this has nothing to do with row count, so `NO_DATA_FOUND` is never a candidate. `'B'` never prints, because control jumps straight to the exception section the instant the error is raised, skipping the rest of the executable section. `ZERO_DIVIDE` is matched exactly by name, so `'D'` prints and `WHEN OTHERS` is never reached.

---

## B2 — Package State Across Sessions

No table needed — this is entirely about where package variables live.

### The package (as given)
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

### Expected output
```
Session X: pkg_counter.current_value -> 5
Session Y: pkg_counter.current_value -> 0
```
A package-level variable like `g_val` is **instantiated once per session**, not once globally. The package body's initialization section (`g_val := 0;`) runs the first time *any* given session references the package — so Session X gets its own private `g_val`, bumps it to 5, and sees 5. Session Y has never touched the package before, so its first reference triggers its own fresh initialization to 0, completely independent of Session X's state. Neither session can see or affect the other's copy.

---

## B3 — UPDATE Matching Zero Rows

### Sample table
```sql
CREATE TABLE accounts (
    account_id NUMBER PRIMARY KEY,
    balance    NUMBER
);

INSERT INTO accounts VALUES (1001, 5000);
COMMIT;
```

### The procedure (as given) and test
```sql
CREATE OR REPLACE PROCEDURE risky_update (p_id IN NUMBER, p_amount IN NUMBER)
IS
BEGIN
    UPDATE accounts SET balance = balance + p_amount WHERE account_id = p_id;
END risky_update;
/

BEGIN
    risky_update(99999, 500);  -- account_id 99999 does not exist
    DBMS_OUTPUT.PUT_LINE('Procedure completed with no error.');
END;
/
SELECT balance FROM accounts WHERE account_id = 1001;
```

### Expected output
```
Procedure completed with no error.

ACCOUNT_ID  BALANCE
----------  -------
1001        5000
```
No exception is raised, and account 1001's balance is unchanged. `NO_DATA_FOUND` is only raised by an *implicit cursor from a `SELECT INTO`* that returns zero rows — it does **not** apply to `UPDATE` or `DELETE`. An `UPDATE` that matches zero rows is not an error condition to Oracle at all; it simply does nothing and completes "successfully." The only way to detect this in code is checking `SQL%ROWCOUNT` explicitly right after the statement — this procedure doesn't, so the caller has no idea the update silently affected nothing.

---

## C1 — Debug: Function Returns Without a Value

### Sample test (no table needed — pure function of its input)
```sql
-- AS GIVEN (buggy)
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

BEGIN
    DBMS_OUTPUT.PUT_LINE(classify_volume(150)); -- HIGH
    DBMS_OUTPUT.PUT_LINE(classify_volume(5));   -- fails here
END;
/
```
### Expected output (before fix)
```
HIGH
ORA-06503: PL/SQL: Function returned without value
```
For any `p_order_count <= 20` (a perfectly valid input, e.g. 5), neither `IF` branch is taken, there's no `ELSE`, and control falls off the end of the function with nothing returned — which Oracle treats as a runtime error, not a silent `NULL`.

### Fix
```sql
CREATE OR REPLACE FUNCTION classify_volume (p_order_count NUMBER)
    RETURN VARCHAR2
IS
BEGIN
    IF p_order_count > 100 THEN
        RETURN 'HIGH';
    ELSIF p_order_count > 20 THEN
        RETURN 'MEDIUM';
    ELSE
        RETURN 'LOW';   -- every path now returns a value
    END IF;
END classify_volume;
/
```
### Expected output (after fix)
```
HIGH
LOW
```

---

## C2 — Debug: Illegal Overload-by-Mode-Only

No table needed — this is purely a package-structure question.

### Why it fails
```sql
CREATE OR REPLACE PACKAGE pkg_validate
IS
    PROCEDURE check_input (p_value IN NUMBER);
    PROCEDURE check_input (p_value OUT NUMBER);
END pkg_validate;
/
```
This will not compile. PL/SQL resolves overloads by the **number and datatypes** of the formal parameters — parameter **name** and parameter **mode** (`IN`/`OUT`/`IN OUT`) are not part of that signature. Both procedures here have exactly one `NUMBER` parameter; as far as overload resolution is concerned they're the same signature declared twice, so Oracle rejects the package spec as a duplicate declaration rather than a legal overload.

### Corrected design
```sql
CREATE OR REPLACE PACKAGE pkg_validate
IS
    -- Validates an incoming value
    FUNCTION  is_valid       (p_value IN NUMBER) RETURN BOOLEAN;
    -- Produces a value (the OUT case) — different name, since mode alone can't overload
    PROCEDURE generate_value (p_value OUT NUMBER);
END pkg_validate;
/
```
This gets close to the likely intent — one routine for checking an incoming value, one for producing an outgoing one — without relying on the mode difference to distinguish them. (True overloading is still available here if needed: `check_input(p_value IN NUMBER)` and `check_input(p_value IN VARCHAR2)` **would** compile, because the datatypes genuinely differ.)

---

## C3 — Debug: Invalid RAISE_APPLICATION_ERROR Number

### Sample table
```sql
CREATE TABLE invoices (
    invoice_id   NUMBER PRIMARY KEY,
    total_amount NUMBER,
    status       VARCHAR2(20)
);

INSERT INTO invoices VALUES (8001, 1500000, 'PENDING');  -- large, should trigger the check
INSERT INTO invoices VALUES (8002, 25000,   'PENDING');  -- normal
COMMIT;
```

### The procedure (as given) and test
```sql
-- AS GIVEN (buggy)
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

BEGIN finalize_invoice(8001); END;
/
```
### Expected output (before fix)
```
ORA-21000: error number argument to RAISE_APPLICATION_ERROR of 500001 is out of range
```
`RAISE_APPLICATION_ERROR` only accepts numbers from **-20000 to -20999**. `500001` is a positive number nowhere near that range, so the block compiles fine (it's a syntactically valid literal) but blows up the instant that line actually executes — which is why it looks unrelated to the business logic: the "manual review" condition is doing exactly what it should, but the error-raising mechanism itself is malformed.

### Fix
```sql
IF v_total > 1000000 THEN
    RAISE_APPLICATION_ERROR(-20030, 'Invoice amount unusually large — manual review required.');
END IF;
```
### Expected output (after fix)
```sql
BEGIN finalize_invoice(8001); END; /  -- ORA-20030: Invoice amount unusually large — manual review required.
BEGIN finalize_invoice(8002); END; /  -- completes silently; status now FINALIZED
```
```
ORA-20030: Invoice amount unusually large — manual review required.
```
```sql
SELECT status FROM invoices WHERE invoice_id = 8002;  -- FINALIZED
```

---

## Section D — Judgment & Reasoning

D1–D3 are prose/reasoning questions with no query or table to run — there's nothing to seed or execute. (D3, on default parameters and existing call sites, is worth answering carefully against your own call-site list once you get there — happy to review that reasoning directly when you share it, no sample data needed for it.)

---

## E1 — Subscription Renewal Module

### Sample tables
```sql
CREATE TABLE plans (
    plan_code     VARCHAR2(10) PRIMARY KEY,
    monthly_price NUMBER
);

CREATE TABLE customers (
    customer_id       NUMBER PRIMARY KEY,
    plan_code         VARCHAR2(10),
    account_status    VARCHAR2(20),
    consecutive_months NUMBER
);

CREATE TABLE subscription_renewals (
    renewal_id     NUMBER PRIMARY KEY,
    customer_id    NUMBER,
    renewal_date   DATE,
    amount_charged NUMBER
);

CREATE SEQUENCE subscription_renewals_seq START WITH 1;

INSERT INTO plans VALUES ('BASIC', 9.99);
INSERT INTO plans VALUES ('PRO', 19.99);
INSERT INTO plans VALUES ('PREMIUM', 29.99);

INSERT INTO customers VALUES (4001, 'PRO',     'ACTIVE',    14); -- 10% loyalty tier
INSERT INTO customers VALUES (4002, 'BASIC',   'SUSPENDED', 3);  -- blocked
INSERT INTO customers VALUES (4003, 'PREMIUM', 'ACTIVE',    2);  -- no discount yet
COMMIT;
```

### Answer query
```sql
CREATE OR REPLACE PACKAGE pkg_subscription_renewal
IS
    -- Distinct, stable codes: catchable by both internal scripts and the
    -- external payment gateway without parsing message text.
    e_suspended_account  EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_suspended_account, -20101);

    e_customer_not_found EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_customer_not_found, -20102);

    PROCEDURE renew_subscription (p_customer_id IN customers.customer_id%TYPE);
END pkg_subscription_renewal;
/

CREATE OR REPLACE PACKAGE BODY pkg_subscription_renewal
IS
    -- Private: internal lookup only, not something other systems call directly
    FUNCTION get_loyalty_discount_pct (p_months IN NUMBER) RETURN NUMBER
    IS
    BEGIN
        IF p_months >= 24 THEN
            RETURN 0.20;
        ELSIF p_months >= 12 THEN
            RETURN 0.10;
        ELSIF p_months >= 6 THEN
            RETURN 0.05;
        ELSE
            RETURN 0;
        END IF;
    END get_loyalty_discount_pct;

    PROCEDURE renew_subscription (p_customer_id IN customers.customer_id%TYPE)
    IS
        v_status       customers.account_status%TYPE;
        v_plan         customers.plan_code%TYPE;
        v_months       customers.consecutive_months%TYPE;
        v_price        plans.monthly_price%TYPE;
        v_discount     NUMBER;
        v_final_amount NUMBER;
    BEGIN
        SELECT account_status, plan_code, consecutive_months
          INTO v_status, v_plan, v_months
          FROM customers
         WHERE customer_id = p_customer_id;

        IF v_status = 'SUSPENDED' THEN
            RAISE e_suspended_account;
        END IF;

        SELECT monthly_price INTO v_price FROM plans WHERE plan_code = v_plan;

        v_discount     := get_loyalty_discount_pct(v_months);
        v_final_amount := ROUND(v_price * (1 - v_discount), 2);

        INSERT INTO subscription_renewals (renewal_id, customer_id, renewal_date, amount_charged)
        VALUES (subscription_renewals_seq.NEXTVAL, p_customer_id, SYSDATE, v_final_amount);

        DBMS_OUTPUT.PUT_LINE('Renewed customer ' || p_customer_id || ' for $' || v_final_amount);
    EXCEPTION
        WHEN e_suspended_account THEN
            RAISE_APPLICATION_ERROR(-20101,
                'Renewal rejected: customer ' || p_customer_id || ' account is SUSPENDED.');
        WHEN NO_DATA_FOUND THEN
            RAISE_APPLICATION_ERROR(-20102,
                'Renewal rejected: customer ' || p_customer_id || ' does not exist.');
    END renew_subscription;
END pkg_subscription_renewal;
/
```
> **Note on a simplification:** the `NO_DATA_FOUND` handler here treats "customer not found" and "plan code not found in `plans`" identically. That's fine given the requirement only asks to distinguish *customer*-not-found, but it's worth flagging in review — a missing `plan_code` is really a referential-integrity problem, not a "this customer doesn't exist" problem, and a stricter design would separate those two with their own error codes too.

### Test & expected output
```sql
BEGIN pkg_subscription_renewal.renew_subscription(4001); END; /  -- 14 mo -> 10% off PRO
BEGIN pkg_subscription_renewal.renew_subscription(4002); END; /  -- SUSPENDED
BEGIN pkg_subscription_renewal.renew_subscription(4003); END; /  -- 2 mo -> no discount
BEGIN pkg_subscription_renewal.renew_subscription(9999); END; /  -- doesn't exist
```
```
Renewed customer 4001 for $17.99
ORA-20101: Renewal rejected: customer 4002 account is SUSPENDED.
Renewed customer 4003 for $29.99
ORA-20102: Renewal rejected: customer 9999 does not exist.
```

### On "one customer's issue must never stop the batch"
`renew_subscription` itself never lets an exception propagate silently uncaught — both failure paths are converted into a distinct, well-formed `RAISE_APPLICATION_ERROR` inside the procedure's own exception section. That matters for the batch design even though the loop is out of scope here: whatever nightly job calls this procedure only needs to wrap **each individual call** in its own small `BEGIN...EXCEPTION WHEN OTHERS...END` block (the same per-iteration pattern used for the payroll batch and month-end close in the earlier assessment) to catch `-20101`/`-20102`, log them, and move to the next customer_id — the procedure's job is just to fail in a way that's catchable and specific, not to fail in a way that takes down the loop around it.