# Section A — Sample Tables, Answer Queries & Expected Output

For each scenario: a minimal sample schema + data, the PL/SQL solution, and the expected output when run against that data. These are illustrative datasets only — small enough to trace by hand, but deliberately including the edge cases each scenario calls out (missing IDs, NULLs, boundary percentages, etc.).

---

## Scenario 1 — Warehouse Restocking Alert System

### Sample table
```sql
CREATE TABLE products (
    product_id      NUMBER PRIMARY KEY,
    product_name    VARCHAR2(100),
    current_stock   NUMBER,
    max_capacity    NUMBER
);

INSERT INTO products VALUES (101, 'Widget A', 5,  100);  -- 5%  of capacity
INSERT INTO products VALUES (102, 'Widget B', 10, 100);  -- exactly 10% (boundary)
INSERT INTO products VALUES (103, 'Widget C', 25, 100);  -- 25%
INSERT INTO products VALUES (104, 'Widget D', 30, 100);  -- exactly 30% (boundary)
INSERT INTO products VALUES (105, 'Widget E', 80, 100);  -- 80%
COMMIT;
```

### Answer query
```sql
-- Read-heavy, rarely-changing, batch-updated data -> RESULT_CACHE candidate
CREATE OR REPLACE FUNCTION get_restock_urgency (
    p_product_id IN products.product_id%TYPE
) RETURN VARCHAR2
RESULT_CACHE RELIES_ON (products)
IS
    v_stock products.current_stock%TYPE;
    v_max   products.max_capacity%TYPE;
    v_pct   NUMBER;
BEGIN
    SELECT current_stock, max_capacity
      INTO v_stock, v_max
      FROM products
     WHERE product_id = p_product_id;

    v_pct := (v_stock / v_max) * 100;

    IF v_pct <= 10 THEN
        RETURN 'CRITICAL';
    ELSIF v_pct <= 30 THEN
        RETURN 'LOW';
    ELSE
        RETURN 'OK';
    END IF;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN 'UNKNOWN_PRODUCT';
END get_restock_urgency;
/

CREATE OR REPLACE PROCEDURE process_restock (
    p_product_id   IN products.product_id%TYPE,
    p_qty_received IN NUMBER
)
IS
    e_invalid_qty EXCEPTION;
    v_dummy       products.product_id%TYPE;
BEGIN
    IF p_qty_received <= 0 THEN
        RAISE e_invalid_qty;
    END IF;

    SELECT product_id INTO v_dummy FROM products WHERE product_id = p_product_id;

    UPDATE products
       SET current_stock = current_stock + p_qty_received
     WHERE product_id = p_product_id;

    DBMS_OUTPUT.PUT_LINE('Restocked product ' || p_product_id || ' by ' || p_qty_received);
EXCEPTION
    WHEN e_invalid_qty THEN
        DBMS_OUTPUT.PUT_LINE('ERROR: quantity must be positive. Got: ' || p_qty_received);
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('ERROR: product ' || p_product_id || ' does not exist.');
END process_restock;
/
```

### Test & expected output
```sql
BEGIN
    DBMS_OUTPUT.PUT_LINE(get_restock_urgency(101)); -- 5%  -> CRITICAL
    DBMS_OUTPUT.PUT_LINE(get_restock_urgency(102)); -- 10% -> CRITICAL (boundary: "at or below")
    DBMS_OUTPUT.PUT_LINE(get_restock_urgency(103)); -- 25% -> LOW
    DBMS_OUTPUT.PUT_LINE(get_restock_urgency(104)); -- 30% -> LOW (boundary: only ABOVE 30 is OK)
    DBMS_OUTPUT.PUT_LINE(get_restock_urgency(105)); -- 80% -> OK
    DBMS_OUTPUT.PUT_LINE(get_restock_urgency(999)); -- no such product -> UNKNOWN_PRODUCT
END;
/
process_restock(101, 50);  -- Restocked product 101 by 50
process_restock(101, -5);  -- ERROR: quantity must be positive. Got: -5
process_restock(999, 10);  -- ERROR: product 999 does not exist.
```
```
CRITICAL
CRITICAL
LOW
LOW
OK
UNKNOWN_PRODUCT
Restocked product 101 by 50
ERROR: quantity must be positive. Got: -5
ERROR: product 999 does not exist.
```

---

## Scenario 2 — Customer Support Ticket Routing

### Sample table
```sql
CREATE TABLE support_tickets (
    ticket_id     NUMBER PRIMARY KEY,
    category_code NUMBER
);

INSERT INTO support_tickets VALUES (1, 1);
INSERT INTO support_tickets VALUES (2, 3);
INSERT INTO support_tickets VALUES (3, 7);
INSERT INTO support_tickets VALUES (4, 9);   -- not a recognized code yet
INSERT INTO support_tickets VALUES (5, 12);  -- not a recognized code yet
COMMIT;
```

### Answer query
```sql
CREATE OR REPLACE FUNCTION route_ticket (p_category_code IN NUMBER) RETURN VARCHAR2
IS
BEGIN
    CASE p_category_code
        WHEN 1 THEN RETURN 'BILLING';
        WHEN 2 THEN RETURN 'TECHNICAL';
        WHEN 3 THEN RETURN 'SALES';
        WHEN 4 THEN RETURN 'RETURNS';
        WHEN 5 THEN RETURN 'ACCOUNT';
        WHEN 6 THEN RETURN 'SHIPPING';
        WHEN 7 THEN RETURN 'ESCALATIONS';
        ELSE RETURN 'GENERAL';   -- catches unrecognized codes by design, not by accident
    END CASE;
END route_ticket;
/
```
> Design note: a searched `CASE` **statement** (not a standalone `CASE` expression with no `ELSE`) is used deliberately — an unmatched `CASE` expression raises `CASE_NOT_FOUND`, which would defeat the "never fail on an unknown code" requirement.

### Test & expected output
```sql
FOR t IN (SELECT ticket_id, category_code FROM support_tickets) LOOP
    DBMS_OUTPUT.PUT_LINE('Ticket ' || t.ticket_id || ' -> ' || route_ticket(t.category_code));
END LOOP;
```
```
Ticket 1 -> BILLING
Ticket 2 -> SALES
Ticket 3 -> ESCALATIONS
Ticket 4 -> GENERAL
Ticket 5 -> GENERAL
```

---

## Scenario 3 — Payroll Batch Run With Mixed Outcomes

### Sample table
```sql
CREATE TABLE employees (
    employee_id       NUMBER PRIMARY KEY,
    base_salary       NUMBER,
    department_id     NUMBER,
    employment_status VARCHAR2(20)
);

INSERT INTO employees VALUES (3001, 5000, 10, 'ACTIVE');      -- Sales     -> 8%
INSERT INTO employees VALUES (3002, 6000, 20, 'ACTIVE');      -- Eng       -> 5%
INSERT INTO employees VALUES (3003, 4500, 30, 'ACTIVE');      -- Other     -> 2%
INSERT INTO employees VALUES (3004, 5500, 10, 'TERMINATED');  -- skip, not an error
INSERT INTO employees VALUES (3005, 7000, 20, 'ON_LEAVE');    -- skip, not an error
-- IDs 3006-3050 deliberately left absent from the table to simulate genuine data gaps
COMMIT;
```

### Answer query
```sql
DECLARE
    v_status    employees.employment_status%TYPE;
    v_salary    employees.base_salary%TYPE;
    v_dept      employees.department_id%TYPE;
    v_bonus_pct NUMBER;
    v_bonus     NUMBER;
BEGIN
    FOR v_id IN 3000..3050 LOOP
        BEGIN
            SELECT employment_status, base_salary, department_id
              INTO v_status, v_salary, v_dept
              FROM employees
             WHERE employee_id = v_id;

            IF v_status <> 'ACTIVE' THEN
                DBMS_OUTPUT.PUT_LINE('SKIP (not active): ' || v_id || ' status=' || v_status);
            ELSE
                v_bonus_pct := CASE v_dept
                                   WHEN 10 THEN 0.08
                                   WHEN 20 THEN 0.05
                                   ELSE 0.02
                               END;
                v_bonus := v_salary * v_bonus_pct;
                DBMS_OUTPUT.PUT_LINE('PROCESSED: ' || v_id || ' bonus=' || v_bonus);
            END IF;
        EXCEPTION
            WHEN NO_DATA_FOUND THEN
                DBMS_OUTPUT.PUT_LINE('DATA ERROR: ' || v_id || ' does not exist.');
        END;
    END LOOP;
END;
/
```
> Design note: the exception block sits **inside** the loop, scoped to a single iteration. A `NO_DATA_FOUND` there does not stop the loop, satisfying "the batch must complete for all valid employees even if some IDs are problematic."

### Expected output (abbreviated — 3006 through 3050 all repeat the "DATA ERROR" pattern)
```
PROCESSED: 3001 bonus=400
PROCESSED: 3002 bonus=300
PROCESSED: 3003 bonus=90
SKIP (not active): 3004 status=TERMINATED
SKIP (not active): 3005 status=ON_LEAVE
DATA ERROR: 3006 does not exist.
DATA ERROR: 3007 does not exist.
...
DATA ERROR: 3050 does not exist.
```

---

## Scenario 4 — Multi-Currency Order Total (One-Off Executive Report)

### Sample table
```sql
CREATE TABLE orders (
    order_id      NUMBER PRIMARY KEY,
    order_date    DATE,
    currency_code VARCHAR2(3),
    order_amount  NUMBER
);

INSERT INTO orders VALUES (5001, DATE '2026-07-15', 'USD', 1000);
INSERT INTO orders VALUES (5002, DATE '2026-08-02', 'EUR', 850);
INSERT INTO orders VALUES (5003, DATE '2026-09-20', 'GBP', 600);
INSERT INTO orders VALUES (5004, DATE '2026-07-30', 'INR', 75000);
INSERT INTO orders VALUES (5005, DATE '2026-06-28', 'USD', 300);  -- Q2, should NOT appear
COMMIT;
```

### Answer query
```sql
-- Deliberately an anonymous block, NOT a stored function/procedure:
-- this is a one-time report and shouldn't leave a permanent object in the schema.
DECLARE
    CURSOR c_q3_orders IS
        SELECT order_id, currency_code, order_amount
          FROM orders
         WHERE order_date BETWEEN DATE '2026-07-01' AND DATE '2026-09-30';

    v_usd_total NUMBER;
BEGIN
    FOR r IN c_q3_orders LOOP
        v_usd_total := CASE r.currency_code
                           WHEN 'USD' THEN r.order_amount
                           WHEN 'EUR' THEN r.order_amount * 1.08
                           WHEN 'GBP' THEN r.order_amount * 1.27
                           WHEN 'INR' THEN r.order_amount * 0.012
                           ELSE NULL
                       END;

        DBMS_OUTPUT.PUT_LINE('Order ' || r.order_id || ' (' || r.currency_code || '): $'
                              || ROUND(v_usd_total, 2));
    END LOOP;
END;
/
```

### Expected output
```
Order 5001 (USD): $1000
Order 5002 (EUR): $918
Order 5003 (GBP): $762
Order 5004 (INR): $900
```
(Order 5005 is correctly excluded — it falls in Q2, not Q3.)

---

## Scenario 5 — Membership Renewal Engine

### Sample tables
```sql
CREATE TABLE members (
    member_id        NUMBER PRIMARY KEY,
    membership_level VARCHAR2(20),
    account_status   VARCHAR2(20),
    expiry_date      DATE
);

CREATE TABLE renewals (
    renewal_id   NUMBER PRIMARY KEY,
    member_id    NUMBER,
    renewal_date DATE,
    amount_paid  NUMBER
);

CREATE SEQUENCE renewals_seq START WITH 1;

-- "today" for this walkthrough is 2026-09-09
INSERT INTO members VALUES (201, 'GOLD',   'ACTIVE',     DATE '2026-09-15'); -- due in 6 days
INSERT INTO members VALUES (202, 'SILVER', 'DELINQUENT', DATE '2026-09-10'); -- blocked
INSERT INTO members VALUES (203, 'BRONZE', 'ACTIVE',     DATE '2026-12-01'); -- not due soon
COMMIT;
```

### Answer query
```sql
CREATE OR REPLACE PACKAGE pkg_membership
IS
    -- Named, stable error code so both internal scripts and the external
    -- payment partner can reliably detect this specific rejection reason.
    e_delinquent_account EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_delinquent_account, -20001);

    PROCEDURE process_renewal (p_member_id IN members.member_id%TYPE);

    FUNCTION is_renewal_due_soon (
        p_member_id      IN members.member_id%TYPE,
        p_days_threshold IN NUMBER DEFAULT 7
    ) RETURN BOOLEAN;
END pkg_membership;
/

CREATE OR REPLACE PACKAGE BODY pkg_membership
IS
    -- Private: pricing lookup is an internal implementation detail only
    FUNCTION get_renewal_price (p_level IN members.membership_level%TYPE) RETURN NUMBER
    IS
    BEGIN
        RETURN CASE p_level
                   WHEN 'GOLD'   THEN 199
                   WHEN 'SILVER' THEN 99
                   WHEN 'BRONZE' THEN 49
                   ELSE 49
               END;
    END get_renewal_price;

    PROCEDURE process_renewal (p_member_id IN members.member_id%TYPE)
    IS
        v_status members.account_status%TYPE;
        v_level  members.membership_level%TYPE;
        v_price  NUMBER;
    BEGIN
        SELECT account_status, membership_level
          INTO v_status, v_level
          FROM members
         WHERE member_id = p_member_id;

        IF v_status = 'DELINQUENT' THEN
            RAISE_APPLICATION_ERROR(-20001,
                'Renewal rejected: member ' || p_member_id || ' is DELINQUENT.');
        END IF;

        v_price := get_renewal_price(v_level);

        UPDATE members
           SET expiry_date = ADD_MONTHS(expiry_date, 12)
         WHERE member_id = p_member_id;

        INSERT INTO renewals (renewal_id, member_id, renewal_date, amount_paid)
        VALUES (renewals_seq.NEXTVAL, p_member_id, SYSDATE, v_price);

        DBMS_OUTPUT.PUT_LINE('Renewed member ' || p_member_id || ' for $' || v_price);
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            DBMS_OUTPUT.PUT_LINE('ERROR: member ' || p_member_id || ' not found.');
    END process_renewal;

    FUNCTION is_renewal_due_soon (
        p_member_id      IN members.member_id%TYPE,
        p_days_threshold IN NUMBER DEFAULT 7
    ) RETURN BOOLEAN
    IS
        v_expiry members.expiry_date%TYPE;
    BEGIN
        SELECT expiry_date INTO v_expiry FROM members WHERE member_id = p_member_id;
        RETURN (v_expiry - TRUNC(SYSDATE)) <= p_days_threshold;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            RETURN FALSE;
    END is_renewal_due_soon;
END pkg_membership;
/
```

### Test & expected output
```sql
BEGIN pkg_membership.process_renewal(201); END; /  -- Renewed member 201 for $199
BEGIN pkg_membership.process_renewal(202); END; /  -- ORA-20001: ... is DELINQUENT.
BEGIN pkg_membership.process_renewal(999); END; /  -- ERROR: member 999 not found.

DBMS_OUTPUT.PUT_LINE(CASE WHEN pkg_membership.is_renewal_due_soon(201) THEN 'TRUE' ELSE 'FALSE' END); -- TRUE
DBMS_OUTPUT.PUT_LINE(CASE WHEN pkg_membership.is_renewal_due_soon(203) THEN 'TRUE' ELSE 'FALSE' END); -- FALSE
```
```
Renewed member 201 for $199
ORA-20001: Renewal rejected: member 202 is DELINQUENT.
ERROR: member 999 not found.
TRUE
FALSE
```

---

## Scenario 6 — Duplicate Prevention on Bulk Import

### Sample table
```sql
CREATE TABLE suppliers (
    supplier_code VARCHAR2(10) PRIMARY KEY,
    supplier_name VARCHAR2(100)
);

-- simulates a previous partial import: some codes already loaded
INSERT INTO suppliers VALUES ('SUP-101', 'Acme Corp');
INSERT INTO suppliers VALUES ('SUP-105', 'Beta Supplies');
INSERT INTO suppliers VALUES ('SUP-110', 'Gamma Traders');
COMMIT;
```

### Answer query
```sql
DECLARE
    v_code VARCHAR2(10);
BEGIN
    FOR i IN 101..120 LOOP
        v_code := 'SUP-' || i;
        BEGIN
            INSERT INTO suppliers (supplier_code, supplier_name)
            VALUES (v_code, 'Placeholder Supplier ' || i);

            DBMS_OUTPUT.PUT_LINE('INSERTED: ' || v_code);
        EXCEPTION
            WHEN DUP_VAL_ON_INDEX THEN
                DBMS_OUTPUT.PUT_LINE('SKIPPED (already exists): ' || v_code);
        END;
    END LOOP;
END;
/
```

### Expected output (abbreviated)
```
SKIPPED (already exists): SUP-101
INSERTED: SUP-102
INSERTED: SUP-103
INSERTED: SUP-104
SKIPPED (already exists): SUP-105
INSERTED: SUP-106
...
SKIPPED (already exists): SUP-110
...
INSERTED: SUP-120
```
Re-running the whole block a second time is safe: every code now hits `DUP_VAL_ON_INDEX` and is skipped — nothing fails.

---

## Scenario 7 — Tiered API Rate Limit Checker

### Sample table
```sql
CREATE TABLE customer_accounts (
    account_id               NUMBER PRIMARY KEY,
    rate_tier                VARCHAR2(10),
    requests_used_this_hour  NUMBER
);

INSERT INTO customer_accounts VALUES (7001, 'BRONZE', 40);
INSERT INTO customer_accounts VALUES (7002, 'SILVER', 480);
INSERT INTO customer_accounts VALUES (7003, 'GOLD',   10000);
COMMIT;
```

### Answer query
```sql
-- The TIER -> CAP mapping is fixed and rarely changes -> safe to RESULT_CACHE.
CREATE OR REPLACE FUNCTION get_tier_cap (p_tier IN VARCHAR2) RETURN NUMBER
RESULT_CACHE
IS
BEGIN
    RETURN CASE p_tier
               WHEN 'BRONZE' THEN 100
               WHEN 'SILVER' THEN 500
               WHEN 'GOLD'   THEN NULL   -- NULL = unlimited
           END;
END get_tier_cap;
/

-- Requests-used changes constantly -> deliberately NOT RESULT_CACHE'd.
-- Caching this would silently serve stale "remaining" counts to the gateway.
CREATE OR REPLACE FUNCTION get_requests_remaining (
    p_account_id IN customer_accounts.account_id%TYPE
) RETURN NUMBER
IS
    v_tier customer_accounts.rate_tier%TYPE;
    v_used customer_accounts.requests_used_this_hour%TYPE;
    v_cap  NUMBER;
BEGIN
    SELECT rate_tier, requests_used_this_hour
      INTO v_tier, v_used
      FROM customer_accounts
     WHERE account_id = p_account_id;

    v_cap := get_tier_cap(v_tier);

    IF v_cap IS NULL THEN
        RETURN -1;  -- convention: -1 = unlimited
    ELSE
        RETURN GREATEST(v_cap - v_used, 0);
    END IF;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        -- Distinct, structured error the gateway can catch by code, not by
        -- parsing a message string, and turn into a proper client-facing error.
        RAISE_APPLICATION_ERROR(-20002, 'ACCOUNT_NOT_FOUND: ' || p_account_id);
END get_requests_remaining;
/
```

### Test & expected output
```sql
DBMS_OUTPUT.PUT_LINE(get_requests_remaining(7001)); -- 100 - 40  = 60
DBMS_OUTPUT.PUT_LINE(get_requests_remaining(7002)); -- 500 - 480 = 20
DBMS_OUTPUT.PUT_LINE(get_requests_remaining(7003)); -- unlimited = -1
DBMS_OUTPUT.PUT_LINE(get_requests_remaining(9999)); -- raises ORA-20002
```
```
60
20
-1
ORA-20002: ACCOUNT_NOT_FOUND: 9999
```

---

## Scenario 8 — End-of-Month Financial Close

### Sample table
```sql
CREATE TABLE cost_centers (
    center_id    NUMBER PRIMARY KEY,
    budget       NUMBER,
    actual_spend NUMBER,
    status       VARCHAR2(10)
);

INSERT INTO cost_centers VALUES (1, 100000, 95000,  'OPEN');   -- under budget
INSERT INTO cost_centers VALUES (2, 50000,  60000,  'OPEN');   -- 20%   over -> SEVERE
INSERT INTO cost_centers VALUES (3, 80000,  85000,  'OPEN');   -- 6.25% over -> MINOR
INSERT INTO cost_centers VALUES (4, NULL,   20000,  'OPEN');   -- budget never set
INSERT INTO cost_centers VALUES (5, 100000, 100000, 'OPEN');   -- exactly at budget
INSERT INTO cost_centers VALUES (6, 40000,  45000,  'CLOSED'); -- already closed, skip
-- IDs 7-40 deliberately left absent to simulate real gaps in setup
COMMIT;
```

### Answer query
```sql
CREATE OR REPLACE PACKAGE pkg_month_end_close
IS
    PROCEDURE run_close (p_start_id IN NUMBER DEFAULT 1, p_end_id IN NUMBER DEFAULT 40);
END pkg_month_end_close;
/

CREATE OR REPLACE PACKAGE BODY pkg_month_end_close
IS
    -- Private: tightly coupled to this close process, not for other systems to call
    FUNCTION get_overage_pct (p_budget IN NUMBER, p_actual IN NUMBER) RETURN NUMBER
    IS
    BEGIN
        RETURN ((p_actual - p_budget) / p_budget) * 100;
    END get_overage_pct;

    PROCEDURE run_close (p_start_id IN NUMBER DEFAULT 1, p_end_id IN NUMBER DEFAULT 40)
    IS
        v_budget NUMBER;
        v_actual NUMBER;
        v_status cost_centers.status%TYPE;
        v_pct    NUMBER;
        v_flag   VARCHAR2(30);
    BEGIN
        FOR v_id IN p_start_id..p_end_id LOOP
            BEGIN
                SELECT budget, actual_spend, status
                  INTO v_budget, v_actual, v_status
                  FROM cost_centers
                 WHERE center_id = v_id;

                IF v_status <> 'OPEN' THEN
                    DBMS_OUTPUT.PUT_LINE('SKIP (not open): Center ' || v_id);
                ELSIF v_budget IS NULL THEN
                    DBMS_OUTPUT.PUT_LINE('Center ' || v_id || ': BUDGET_NOT_SET');
                ELSE
                    v_pct := get_overage_pct(v_budget, v_actual);
                    IF v_pct > 15 THEN
                        v_flag := 'OVER_BUDGET_SEVERE';
                    ELSIF v_pct > 0 THEN
                        v_flag := 'OVER_BUDGET_MINOR';
                    ELSE
                        v_flag := 'CLOSED_CLEAN';
                    END IF;
                    DBMS_OUTPUT.PUT_LINE('Center ' || v_id || ': ' || v_flag);
                END IF;
            EXCEPTION
                WHEN NO_DATA_FOUND THEN
                    -- Must never halt the run — finance needs it to complete every time
                    DBMS_OUTPUT.PUT_LINE('DATA ERROR: Center ' || v_id || ' does not exist. Continuing.');
            END;
        END LOOP;
    END run_close;
END pkg_month_end_close;
/
```

### Expected output (abbreviated — 7 through 40 all repeat the "DATA ERROR" pattern)
```
Center 1: CLOSED_CLEAN
Center 2: OVER_BUDGET_SEVERE
Center 3: OVER_BUDGET_MINOR
Center 4: BUDGET_NOT_SET
Center 5: CLOSED_CLEAN
SKIP (not open): Center 6
DATA ERROR: Center 7 does not exist. Continuing.
DATA ERROR: Center 8 does not exist. Continuing.
...
DATA ERROR: Center 40 does not exist. Continuing.
```

---

*One thing worth double-checking as you work through these: the boundary values (Scenario 1's exactly-10%/exactly-30% products, Scenario 8's exactly-at-budget center) are in the sample data on purpose. If your `IF`/`CASE` logic uses `<` where it needs `<=` (or vice versa), these are the rows that will expose it — trace them by hand against your own code before assuming it's correct.*