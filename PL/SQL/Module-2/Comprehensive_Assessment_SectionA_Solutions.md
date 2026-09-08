# Section A — Worked Solutions with Expected Output & Explanations

> This is a companion answer key to **Section A** of your Comprehensive Mixed Assessment (Modules 1–4). For each scenario: a brief restatement, the **assumed sample data** (since no real tables exist, I've defined small, realistic sample rows so the expected output is concrete and verifiable), the **solution code**, the **expected output**, and an **explanation** tying the result back to the concepts involved.
>
> Try each scenario yourself first if you haven't already — this is most useful as a way to check your reasoning, not as a substitute for attempting it.

---

## Scenario 1 — Warehouse Restocking Alert System

### Assumed Sample Data
`products(product_id, current_stock, max_capacity)`
| product_id | current_stock | max_capacity |
|---|---|---|
| 100 | 5 | 100 |
| 101 | 40 | 100 |
| 102 | 80 | 100 |

### Solution
```sql
CREATE OR REPLACE FUNCTION get_restock_urgency (p_product_id IN NUMBER)
    RETURN VARCHAR2
    RESULT_CACHE
IS
    v_current NUMBER;
    v_max     NUMBER;
    v_pct     NUMBER;
BEGIN
    SELECT current_stock, max_capacity INTO v_current, v_max
    FROM products WHERE product_id = p_product_id;

    v_pct := (v_current / v_max) * 100;

    IF v_pct <= 10 THEN
        RETURN 'CRITICAL';
    ELSIF v_pct <= 30 THEN
        RETURN 'LOW';
    ELSE
        RETURN 'OK';
    END IF;
END get_restock_urgency;
/

CREATE OR REPLACE PROCEDURE process_restock (p_product_id IN NUMBER, p_quantity IN NUMBER)
IS
    e_invalid_quantity EXCEPTION;
BEGIN
    IF p_quantity <= 0 THEN
        RAISE e_invalid_quantity;
    END IF;

    UPDATE products
    SET current_stock = current_stock + p_quantity
    WHERE product_id = p_product_id;

    IF SQL%ROWCOUNT = 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Product not found: ' || p_product_id);
    END IF;

    DBMS_OUTPUT.PUT_LINE('Restock processed for product ' || p_product_id);

EXCEPTION
    WHEN e_invalid_quantity THEN
        RAISE_APPLICATION_ERROR(-20002, 'Received quantity must be positive.');
END process_restock;
/

-- Demo
BEGIN
    DBMS_OUTPUT.PUT_LINE('Product 100 urgency: ' || get_restock_urgency(100));
    DBMS_OUTPUT.PUT_LINE('Product 101 urgency: ' || get_restock_urgency(101));
    process_restock(100, 20);
    process_restock(999, 10);  -- doesn't exist
END;
/
```

### Expected Output
```
Product 100 urgency: CRITICAL
Product 101 urgency: OK
Restock processed for product 100
ORA-20001: Product not found: 999
ORA-06512: at "PROCESS_RESTOCK", line X
```
*(The last two lines represent the unhandled `RAISE_APPLICATION_ERROR` propagating out of the anonymous block, since the demo block itself has no `EXCEPTION` section — execution stops there.)*

### Explanation
- `get_restock_urgency` is `RESULT_CACHE`-tagged because it's called constantly across dashboards, and stock levels only change via infrequent batch updates — a textbook good-fit case from Module 3's Function Result Cache topic. Product 100 (5/100 = 5%) → `CRITICAL`; product 101 (40/100 = 40%) → `OK`.
- `process_restock` uses `UPDATE`, which — per Module 4, Topic 2's edge cases — does **not** raise `NO_DATA_FOUND` on zero matching rows. That's exactly why `SQL%ROWCOUNT` is checked explicitly afterward to detect a missing product, translated into a clean, distinct `RAISE_APPLICATION_ERROR(-20001, ...)`.
- Invalid quantity (≤ 0) is a genuine **business rule violation**, not a missing-row problem — handled via a **user-defined exception** (`e_invalid_quantity`), then translated into its own distinct error code (`-20002`) so calling systems can tell the two failure types apart.

---

## Scenario 2 — Customer Support Ticket Routing

### Solution
```sql
CREATE OR REPLACE FUNCTION route_ticket (p_category IN NUMBER)
    RETURN VARCHAR2
IS
BEGIN
    RETURN CASE p_category
        WHEN 1 THEN 'BILLING'
        WHEN 2 THEN 'TECHNICAL'
        WHEN 3 THEN 'SALES'
        WHEN 4 THEN 'RETURNS'
        WHEN 5 THEN 'ACCOUNT_MGMT'
        WHEN 6 THEN 'COMPLAINTS'
        WHEN 7 THEN 'VIP'
        ELSE 'GENERAL'
    END;
END route_ticket;
/

BEGIN
    DBMS_OUTPUT.PUT_LINE('Category 3 -> ' || route_ticket(3));
    DBMS_OUTPUT.PUT_LINE('Category 9 -> ' || route_ticket(9));  -- unrecognized
END;
/
```

### Expected Output
```
Category 3 -> SALES
Category 9 -> GENERAL
```

### Explanation
This uses a **CASE expression** (not a CASE statement) so it can be directly `RETURN`ed as a single value — matching Module 2, Topic 3's guidance that value-producing logic belongs in an expression. The **`ELSE 'GENERAL'`** clause is what makes this safe against unseen codes — a Simple CASE with no `ELSE` would instead raise `CASE_NOT_FOUND` the moment an unrecognized code (like `9`) appeared, exactly the trap that topic warned about. Including `ELSE` turns "new category we haven't mapped yet" from a production crash into a graceful, correct fallback.

---

## Scenario 3 — Payroll Batch Run With Mixed Outcomes

### Assumed Sample Data
`employees(employee_id, base_salary, department_id, employment_status)`
| employee_id | base_salary | department_id | employment_status |
|---|---|---|---|
| 3000 | 50000 | 10 | ACTIVE |
| 3001 | 60000 | 20 | ACTIVE |
| 3002 | 45000 | 30 | INACTIVE |
| 3003 | 70000 | 10 | ACTIVE |
| *(3004 does not exist)* | | | |

### Solution
```sql
BEGIN
    FOR emp_id IN 3000..3004 LOOP
        DECLARE
            v_salary employees.base_salary%TYPE;
            v_dept   employees.department_id%TYPE;
            v_status employees.employment_status%TYPE;
            v_bonus_pct NUMBER;
        BEGIN
            SELECT base_salary, department_id, employment_status
            INTO v_salary, v_dept, v_status
            FROM employees
            WHERE employee_id = emp_id;

            IF v_status != 'ACTIVE' THEN
                DBMS_OUTPUT.PUT_LINE('Employee ' || emp_id || ': skipped (not active).');
            ELSE
                v_bonus_pct := CASE v_dept
                                   WHEN 10 THEN 8
                                   WHEN 20 THEN 5
                                   ELSE 2
                               END;

                UPDATE employees
                SET base_salary = base_salary + (base_salary * v_bonus_pct / 100)
                WHERE employee_id = emp_id;

                DBMS_OUTPUT.PUT_LINE('Employee ' || emp_id || ': bonus ' || v_bonus_pct || '% applied.');
            END IF;

        EXCEPTION
            WHEN NO_DATA_FOUND THEN
                DBMS_OUTPUT.PUT_LINE('Employee ' || emp_id || ': NOT FOUND - data issue, logged.');
        END;
    END LOOP;
END;
/
```

### Expected Output
```
Employee 3000: bonus 8% applied.
Employee 3001: bonus 5% applied.
Employee 3002: skipped (not active).
Employee 3003: bonus 8% applied.
Employee 3004: NOT FOUND - data issue, logged.
```

### Explanation
- The `FOR` loop is correct here since the range (3000–3004) is fixed and known in advance (Module 2, Topic 4).
- Each iteration gets its **own nested block** with local variables and its **own exception handler** — this is the exact "one bad record shouldn't stop the batch" pattern from Module 2, Topic 5 / Module 4: employee 3004's missing-row `NO_DATA_FOUND` is caught **locally**, and the loop continues to completion regardless.
- "Not active" (3002) is deliberately **not** an exception at all — it's normal, expected branching handled with a plain `IF`, exactly per Module 4, Topic 3's guidance not to overuse exceptions for routine outcomes. This is what distinguishes it clearly from 3004's genuine data problem.
- `%TYPE` anchors each variable to its source column, so this code keeps working automatically if the underlying column definitions ever change.

---

## Scenario 4 — Multi-Currency Order Total (One-Off Executive Report)

### Assumed Sample Data
`orders(order_id, currency_code, order_amount, order_date)` — all within Q3 2026:
| order_id | currency_code | order_amount |
|---|---|---|
| 9001 | EUR | 1000 |
| 9002 | USD | 500 |
| 9003 | GBP | 200 |
| 9004 | INR | 10000 |

### Solution
```sql
WITH
    FUNCTION to_usd (p_amount NUMBER, p_currency VARCHAR2) RETURN NUMBER
    IS
    BEGIN
        RETURN CASE p_currency
                   WHEN 'USD' THEN p_amount
                   WHEN 'EUR' THEN p_amount * 1.08
                   WHEN 'GBP' THEN p_amount * 1.27
                   WHEN 'INR' THEN p_amount * 0.012
                   ELSE NULL
               END;
    END;
SELECT order_id, currency_code, order_amount,
       to_usd(order_amount, currency_code) AS amount_usd
FROM orders
WHERE order_date BETWEEN DATE '2026-07-01' AND DATE '2026-09-30';
```

### Expected Output
| order_id | currency_code | order_amount | amount_usd |
|---|---|---|---|
| 9001 | EUR | 1000 | 1080 |
| 9002 | USD | 500 | 500 |
| 9003 | GBP | 200 | 254 |
| 9004 | INR | 10000 | 120 |

### Explanation
This is Module 3's `WITH FUNCTION` — chosen specifically because the requirement states the calculation is genuinely disposable ("likely never needed again in this form"). Creating a permanent `CREATE FUNCTION` object here would leave unnecessary schema clutter for a one-time need, exactly the anti-pattern that topic warned against. The function is scoped to this single query, uses full CASE logic, and disappears the moment the query finishes.

---

## Scenario 5 — Membership Renewal Engine

### Assumed Sample Data
`members(member_id, membership_level, status, expiry_date)`
| member_id | membership_level | status | expiry_date |
|---|---|---|---|
| 501 | GOLD | ACTIVE | SYSDATE + 10 |
| 502 | SILVER | DELINQUENT | SYSDATE + 5 |
| 503 | BASIC | ACTIVE | SYSDATE + 90 |

### Solution
```sql
CREATE OR REPLACE PACKAGE pkg_membership
IS
    PROCEDURE renew_membership (p_member_id IN NUMBER);
    FUNCTION is_renewal_due_soon (p_member_id IN NUMBER) RETURN VARCHAR2;
END pkg_membership;
/

CREATE OR REPLACE PACKAGE BODY pkg_membership
IS
    -- private: internal pricing lookup, not exposed
    FUNCTION get_renewal_price (p_level VARCHAR2) RETURN NUMBER
    IS
    BEGIN
        RETURN CASE p_level
                   WHEN 'GOLD'   THEN 100
                   WHEN 'SILVER' THEN 70
                   WHEN 'BASIC'  THEN 40
                   ELSE 50
               END;
    END get_renewal_price;

    PROCEDURE renew_membership (p_member_id IN NUMBER)
    IS
        v_level  members.membership_level%TYPE;
        v_status members.status%TYPE;
        v_price  NUMBER;
    BEGIN
        SELECT membership_level, status INTO v_level, v_status
        FROM members WHERE member_id = p_member_id;

        IF v_status = 'DELINQUENT' THEN
            RAISE_APPLICATION_ERROR(-20010, 'Renewal rejected: account is delinquent.');
        END IF;

        v_price := get_renewal_price(v_level);

        UPDATE members
        SET expiry_date = expiry_date + 365
        WHERE member_id = p_member_id;

        INSERT INTO renewals (member_id, amount_charged, renewed_on)
        VALUES (p_member_id, v_price, SYSDATE);

        DBMS_OUTPUT.PUT_LINE('Member ' || p_member_id || ' renewed at price ' || v_price);

    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            RAISE_APPLICATION_ERROR(-20011, 'Member not found: ' || p_member_id);
    END renew_membership;

    FUNCTION is_renewal_due_soon (p_member_id IN NUMBER) RETURN VARCHAR2
    IS
        v_expiry members.expiry_date%TYPE;
    BEGIN
        SELECT expiry_date INTO v_expiry FROM members WHERE member_id = p_member_id;

        IF v_expiry - SYSDATE <= 30 THEN
            RETURN 'Y';
        ELSE
            RETURN 'N';
        END IF;
    END is_renewal_due_soon;

END pkg_membership;
/

-- Demo
BEGIN
    pkg_membership.renew_membership(501);
    DBMS_OUTPUT.PUT_LINE('501 due soon? ' || pkg_membership.is_renewal_due_soon(501));
    DBMS_OUTPUT.PUT_LINE('503 due soon? ' || pkg_membership.is_renewal_due_soon(503));
    pkg_membership.renew_membership(502);  -- delinquent
END;
/
```

### Expected Output
```
Member 501 renewed at price 100
501 due soon? Y
503 due soon? N
ORA-20010: Renewal rejected: account is delinquent.
ORA-06512: at "PKG_MEMBERSHIP", line X
```

### Explanation
- `get_renewal_price` is **private** (body-only, not in the spec) — exactly matching the requirement's explicit "internal implementation detail" language from Module 3, Topics 8/10.
- The `DELINQUENT` check is deliberately checked and rejected via `RAISE_APPLICATION_ERROR(-20010, ...)` **before** any pricing/update logic runs — giving both internal scripts and the external payment partner a distinct, structured, catchable error code, per the requirement's explicit dual-consumer need (Module 4, Topic 5).
- `is_renewal_due_soon` is a separate **public function**, since the requirement explicitly says other systems need to query it independently — it has no side effects, making it a clean, safe, reusable check.
- Member 501 (10 days left) → `'Y'`; member 503 (90 days left) → `'N'` — straightforward threshold logic via `IF`.

---

## Scenario 6 — Duplicate Prevention on Bulk Import

### Assumed Sample Data
`suppliers(supplier_code, supplier_name)` — `supplier_code` has a **unique constraint**. Pre-existing row: `'SUP-102'` already exists from a prior partial import.

### Solution
```sql
BEGIN
    FOR i IN 101..105 LOOP
        DECLARE
            v_code VARCHAR2(10) := 'SUP-' || i;
        BEGIN
            INSERT INTO suppliers (supplier_code, supplier_name)
            VALUES (v_code, 'Supplier ' || i);

            DBMS_OUTPUT.PUT_LINE(v_code || ': imported.');

        EXCEPTION
            WHEN DUP_VAL_ON_INDEX THEN
                DBMS_OUTPUT.PUT_LINE(v_code || ': already exists - skipped.');
        END;
    END LOOP;
END;
/
```

### Expected Output
```
SUP-101: imported.
SUP-102: already exists - skipped.
SUP-103: imported.
SUP-104: imported.
SUP-105: imported.
```

### Explanation
This is a direct application of Module 4, Topic 2's `DUP_VAL_ON_INDEX` pre-defined exception — raised automatically by the unique constraint violation on `SUP-102`. Because it's caught **locally**, inside a nested block within the loop, the batch continues cleanly to `SUP-103` onward, exactly matching the requirement's explicit "safe to re-run" expectation — a duplicate is treated as a normal, anticipated outcome of re-running an import, not a fatal error.

---

## Scenario 7 — Tiered API Rate Limit Checker

### Assumed Sample Data
`api_accounts(account_id, tier, requests_used_this_hour)`
| account_id | tier | requests_used_this_hour |
|---|---|---|
| 1 | GOLD | 5000 |
| 2 | BRONZE | 80 |
| *(3 does not exist)* | | |

### Solution
```sql
-- Cacheable: the tier -> cap mapping is fixed and rarely changes
CREATE OR REPLACE FUNCTION get_tier_cap (p_tier IN VARCHAR2)
    RETURN VARCHAR2
    RESULT_CACHE
IS
BEGIN
    RETURN CASE p_tier
               WHEN 'BRONZE' THEN '100'
               WHEN 'SILVER' THEN '500'
               WHEN 'GOLD'   THEN 'UNLIMITED'
               ELSE 'UNKNOWN'
           END;
END get_tier_cap;
/

-- NOT cached: usage figures change constantly, every single request
CREATE OR REPLACE FUNCTION get_account_rate_status (p_account_id IN NUMBER)
    RETURN VARCHAR2
IS
    v_tier VARCHAR2(20);
    v_used NUMBER;
    v_cap  VARCHAR2(20);
BEGIN
    SELECT tier, requests_used_this_hour INTO v_tier, v_used
    FROM api_accounts WHERE account_id = p_account_id;

    v_cap := get_tier_cap(v_tier);

    RETURN 'Tier: ' || v_tier || ', Used: ' || v_used || ', Cap: ' || v_cap;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20020, 'Account not found: ' || p_account_id);
END get_account_rate_status;
/

BEGIN
    DBMS_OUTPUT.PUT_LINE(get_account_rate_status(1));
    DBMS_OUTPUT.PUT_LINE(get_account_rate_status(2));
    DBMS_OUTPUT.PUT_LINE(get_account_rate_status(3));
END;
/
```

### Expected Output
```
Tier: GOLD, Used: 5000, Cap: UNLIMITED
Tier: BRONZE, Used: 80, Cap: 100
ORA-20020: Account not found: 3
ORA-06512: at "GET_ACCOUNT_RATE_STATUS", line X
```

### Explanation
This scenario tests a **judgment split**, not just mechanical `RESULT_CACHE` application: the **tier-to-cap mapping** (`get_tier_cap`) is a perfect caching candidate — fixed, rarely changing, called constantly (Module 3, Topic 6). But **per-account usage counts** change every single request; wrapping `get_account_rate_status` itself in `RESULT_CACHE` would be a serious mistake — it would silently serve **stale usage numbers**, defeating the entire purpose of a rate limiter. Only caching the genuinely static piece, while leaving the dynamic per-account lookup uncached, is the correct design. The missing account (`3`) is cleanly translated into a structured `RAISE_APPLICATION_ERROR`, exactly as the requirement demands for the API gateway to handle properly.

---

## Scenario 8 — End-of-Month Financial Close (The Big One)

### Assumed Sample Data
`cost_centers(center_id, budget, actual_spend, status)`
| center_id | budget | actual_spend | status |
|---|---|---|---|
| 1 | 10000 | 9000 | OPEN |
| 2 | 10000 | 11200 | OPEN |
| 3 | 10000 | 13000 | OPEN |
| 4 | NULL | 5000 | OPEN |
| 5 | 8000 | 8000 | CLOSED *(already closed from a prior run)* |
| *(6 does not exist)* | | | |

### Solution
```sql
CREATE OR REPLACE PACKAGE pkg_month_end_close
IS
    PROCEDURE run_close (p_start_id IN NUMBER, p_end_id IN NUMBER);
END pkg_month_end_close;
/

CREATE OR REPLACE PACKAGE BODY pkg_month_end_close
IS
    -- private: tightly coupled to this close process, not for general use
    FUNCTION calc_over_budget_pct (p_budget NUMBER, p_actual NUMBER) RETURN NUMBER
    IS
    BEGIN
        RETURN ((p_actual - p_budget) / p_budget) * 100;
    END calc_over_budget_pct;

    PROCEDURE run_close (p_start_id IN NUMBER, p_end_id IN NUMBER)
    IS
    BEGIN
        FOR center_id IN p_start_id..p_end_id LOOP
            DECLARE
                v_budget     cost_centers.budget%TYPE;
                v_actual     cost_centers.actual_spend%TYPE;
                v_status     cost_centers.status%TYPE;
                v_pct        NUMBER;
                v_new_status VARCHAR2(30);
            BEGIN
                SELECT budget, actual_spend, status
                INTO v_budget, v_actual, v_status
                FROM cost_centers WHERE center_id = center_id;

                IF v_status != 'OPEN' THEN
                    DBMS_OUTPUT.PUT_LINE('Cost Center ' || center_id || ': already closed - skipped.');
                    CONTINUE;
                END IF;

                IF v_budget IS NULL THEN
                    v_new_status := 'BUDGET_NOT_SET';
                ELSE
                    v_pct := calc_over_budget_pct(v_budget, v_actual);
                    IF v_pct > 15 THEN
                        v_new_status := 'OVER_BUDGET_SEVERE';
                    ELSIF v_pct > 0 THEN
                        v_new_status := 'OVER_BUDGET_MINOR';
                    ELSE
                        v_new_status := 'CLOSED_CLEAN';
                    END IF;
                END IF;

                UPDATE cost_centers SET status = v_new_status WHERE center_id = center_id;
                DBMS_OUTPUT.PUT_LINE('Cost Center ' || center_id || ': ' || v_new_status);

            EXCEPTION
                WHEN NO_DATA_FOUND THEN
                    DBMS_OUTPUT.PUT_LINE('Cost Center ' || center_id || ': NOT FOUND - logged, continuing.');
            END;
        END LOOP;
    END run_close;

END pkg_month_end_close;
/

BEGIN
    pkg_month_end_close.run_close(1, 6);
END;
/
```

### Expected Output
```
Cost Center 1: CLOSED_CLEAN
Cost Center 2: OVER_BUDGET_MINOR
Cost Center 3: OVER_BUDGET_SEVERE
Cost Center 4: BUDGET_NOT_SET
Cost Center 5: already closed - skipped.
Cost Center 6: NOT FOUND - logged, continuing.
```

### Explanation
Walking through each center against the requirement's exact rules:
- **Center 1**: (9000−10000)/10000 = **−10%** → not over budget at all → `CLOSED_CLEAN`.
- **Center 2**: (11200−10000)/10000 = **12%** → over budget, but ≤15% → `OVER_BUDGET_MINOR`.
- **Center 3**: (13000−10000)/10000 = **30%** → over budget, >15% → `OVER_BUDGET_SEVERE`.
- **Center 4**: `budget IS NULL` → checked **before** attempting the percentage calculation (which would otherwise silently produce `NULL` or, if budget were literally `0` instead of `NULL`, raise `ZERO_DIVIDE`) → `BUDGET_NOT_SET`, a distinctly different, non-financial flag, exactly as required.
- **Center 5**: already `CLOSED` → the `CONTINUE` statement (Module 2, Topic 4) skips straight to the next iteration — this is normal, expected branching (Module 4, Topic 3's "don't overuse exceptions for routine outcomes"), not an error.
- **Center 6**: doesn't exist → caught locally by the nested block's `NO_DATA_FOUND` handler → logged, batch continues, never halts — fulfilling the "must-complete-every-time" business-critical requirement.
- `calc_over_budget_pct` is **private** — it's described as "tightly coupled to this specific close process," matching Module 3's public/private design judgment exactly.

**On the reviewer questions**: the calculation was kept private and paired tightly with `run_close` because it has no meaning outside this specific process (unlike Scenario 1's restock urgency, which genuinely is reused elsewhere — a good contrast to notice). The single biggest design risk if requirements change next quarter is the **hardcoded 15% severity threshold** — if finance changes this tiering rule, it's a straightforward one-line change inside a private function, precisely because it was isolated rather than duplicated inline throughout the loop — a direct payoff of the encapsulation principles from Module 3.

---

*That's the full worked answer key for Section A. If any of your own attempts diverged — especially in the exception-handling structure or the public/private boundaries — it's worth comparing your reasoning against the explanations above rather than just the code itself, since the reasoning is what transfers to new, unfamiliar requirements.*
