# Modules 1 & 2 — Sample Tables, Answer Queries & Expected Output

Same format as the previous two answer keys. One thing worth flagging up front: this assessment is scoped to **Modules 1 & 2 only** — formal exception handling (`EXCEPTION WHEN ...`) is a Module 4 topic. So where earlier answer keys used `WHEN NO_DATA_FOUND` to handle a missing ID, the solutions below instead use an explicit `SELECT COUNT(*)` existence check before fetching — this gets the same "don't halt the batch" robustness using only tools this assessment has actually covered (`SELECT INTO`, `IF`, loops). It's called out explicitly below wherever it applies.

---

## A1 — Diagnostic Script Using `%ROWTYPE`

### Sample table
```sql
CREATE TABLE products (
    product_id   NUMBER PRIMARY KEY,
    product_name VARCHAR2(100),
    price        NUMBER,
    stock_qty    NUMBER
);

INSERT INTO products VALUES (305, 'Wireless Mouse', 899.5, 42);
INSERT INTO products VALUES (306, 'USB Cable', 199.0, 120);
COMMIT;
```

### Answer query
```sql
DECLARE
    v_product products%ROWTYPE;
BEGIN
    SELECT * INTO v_product FROM products WHERE product_id = 305;

    DBMS_OUTPUT.PUT_LINE(v_product.product_name || ' | Price: ' || v_product.price ||
                          ' | In Stock: ' || v_product.stock_qty);
END;
/
```
> **Why `%ROWTYPE` over listing individual columns:** with `SELECT * INTO v_product` and a `%ROWTYPE` declaration, adding a new column to `products` later requires **zero changes** to this block's declaration or `SELECT` — it keeps compiling and running exactly as before. If this had instead used named scalar variables (`v_name, v_price, v_qty`) with an explicit column list, that's still fine until someone adds a column and forgets this script exists; `%ROWTYPE` removes that failure mode entirely.

### Expected output
```
Wireless Mouse | Price: 899.5 | In Stock: 42
```

---

## A2 — Loyalty Tier Classification With Explicit NULL Handling

### Sample table
```sql
CREATE TABLE customer_loyalty (
    customer_id    NUMBER PRIMARY KEY,
    loyalty_points NUMBER
);

INSERT INTO customer_loyalty VALUES (1, 0);     -- NEW MEMBER
INSERT INTO customer_loyalty VALUES (2, 250);   -- BRONZE
INSERT INTO customer_loyalty VALUES (3, 500);   -- SILVER (lower boundary)
INSERT INTO customer_loyalty VALUES (4, 1999);  -- SILVER (upper boundary)
INSERT INTO customer_loyalty VALUES (5, 2000);  -- GOLD (boundary)
INSERT INTO customer_loyalty VALUES (6, NULL);  -- migrated, unknown balance
COMMIT;
```

### Answer query
```sql
DECLARE
    v_tier VARCHAR2(20);
BEGIN
    FOR c IN (SELECT customer_id, loyalty_points FROM customer_loyalty) LOOP
        IF c.loyalty_points IS NULL THEN
            v_tier := 'DATA MISSING';
        ELSIF c.loyalty_points = 0 THEN
            v_tier := 'NEW MEMBER';
        ELSIF c.loyalty_points BETWEEN 1 AND 499 THEN
            v_tier := 'BRONZE';
        ELSIF c.loyalty_points BETWEEN 500 AND 1999 THEN
            v_tier := 'SILVER';
        ELSE
            v_tier := 'GOLD';
        END IF;

        DBMS_OUTPUT.PUT_LINE('Customer ' || c.customer_id || ': ' || v_tier);
    END LOOP;
END;
/
```
> **Why the `IS NULL` check has to come first, explicitly:** any direct comparison against a `NULL` operand (`= 0`, `BETWEEN 1 AND 499`, etc.) evaluates to `NULL` — neither `TRUE` nor `FALSE` — so every `ELSIF` would silently skip a `NULL` balance and it would fall all the way through to the final `ELSE`, landing quietly in `'GOLD'`. That's exactly the bug the requirement is warning about; only an explicit `IS NULL` test, checked before anything else, catches it correctly.

### Expected output
```
Customer 1: NEW MEMBER
Customer 2: BRONZE
Customer 3: SILVER
Customer 4: SILVER
Customer 5: GOLD
Customer 6: DATA MISSING
```

---

## A3 — Invoice Batch (4001–4015)

### Sample table
```sql
CREATE TABLE invoices (
    invoice_id NUMBER PRIMARY KEY,
    amount     NUMBER,
    due_date   DATE,
    paid_flag  VARCHAR2(1)
);

-- "today" for this walkthrough is 2026-09-09
INSERT INTO invoices VALUES (4001, 500, DATE '2026-07-01', 'N'); -- unpaid, ~70 days overdue
INSERT INTO invoices VALUES (4002, 300, DATE '2026-09-01', 'N'); -- unpaid, within 30 days
INSERT INTO invoices VALUES (4003, 200, DATE '2026-08-01', 'Y'); -- already paid
INSERT INTO invoices VALUES (4004, 750, DATE '2026-06-15', 'N'); -- unpaid, overdue
INSERT INTO invoices VALUES (4010, 100, DATE '2026-09-05', 'N'); -- unpaid, within 30 days
-- 4005-4009 and 4011-4015 deliberately absent to test the "doesn't exist" path
COMMIT;
```

### Answer query
```sql
-- Existence checked via COUNT(*), not SELECT INTO + exception handling —
-- COUNT(*) always returns exactly one row (0 or more), so it can never itself
-- raise NO_DATA_FOUND, which keeps this solution within Module 1 & 2 tools.
DECLARE
    v_count     NUMBER;
    v_due_date  invoices.due_date%TYPE;
    v_paid_flag invoices.paid_flag%TYPE;
BEGIN
    FOR v_id IN 4001..4015 LOOP
        SELECT COUNT(*) INTO v_count FROM invoices WHERE invoice_id = v_id;

        IF v_count = 0 THEN
            DBMS_OUTPUT.PUT_LINE('Invoice ' || v_id || ': NOT FOUND - skipping.');
        ELSE
            SELECT due_date, paid_flag INTO v_due_date, v_paid_flag
              FROM invoices WHERE invoice_id = v_id;

            IF v_paid_flag = 'N' THEN
                IF (TRUNC(SYSDATE) - v_due_date) > 30 THEN
                    DBMS_OUTPUT.PUT_LINE('Invoice ' || v_id || ': OVERDUE - ESCALATE');
                ELSE
                    DBMS_OUTPUT.PUT_LINE('Invoice ' || v_id || ': PENDING');
                END IF;
            END IF;
            -- paid invoices intentionally produce no line, per "still unpaid"
        END IF;
    END LOOP;
END;
/
```

### Expected output (abbreviated — 4005–4009 and 4011–4015 all repeat "NOT FOUND")
```
Invoice 4001: OVERDUE - ESCALATE
Invoice 4002: PENDING
Invoice 4004: OVERDUE - ESCALATE
Invoice 4005: NOT FOUND - skipping.
Invoice 4006: NOT FOUND - skipping.
Invoice 4007: NOT FOUND - skipping.
Invoice 4008: NOT FOUND - skipping.
Invoice 4009: NOT FOUND - skipping.
Invoice 4010: PENDING
Invoice 4011: NOT FOUND - skipping.
...
Invoice 4015: NOT FOUND - skipping.
```
(Invoice 4003 correctly produces no line at all — it's paid.)

---

## B1 — Predict: NULL in a Two-Branch IF, and NULL in Concatenation

No table needed — pure variable behavior.

### The block (as given)
```sql
DECLARE
    v_score NUMBER := NULL;
BEGIN
    IF v_score > 50 THEN
        DBMS_OUTPUT.PUT_LINE('Above 50');
    ELSE
        DBMS_OUTPUT.PUT_LINE('50 or below');
    END IF;

    DBMS_OUTPUT.PUT_LINE('Total (with bonus): ' || (v_score + 10));
END;
/
```

### Expected output
```
50 or below
Total (with bonus): 
```
`v_score > 50` evaluates to `NULL` (neither `TRUE` nor `FALSE`), and an `IF` only runs its `THEN` branch on a genuine `TRUE` — anything else, including `NULL`, falls to `ELSE`. That's why `'50 or below'` prints even though `v_score` isn't actually known to be 50 or below; there's no third branch here to say "unknown." Second line: `v_score + 10` is *arithmetic*, so `NULL + 10 = NULL`. But concatenation (`||`) treats a `NULL` operand as an empty string rather than propagating `NULL` to the whole expression — so the line prints with nothing after the colon, not an error and not the literal word "null."

---

## B2 — Predict: `CASE` With No Matching `WHEN` and No `ELSE`

No table needed.

### The block (as given)
```sql
DECLARE
    v_code NUMBER := 4;
    v_label VARCHAR2(20);
BEGIN
    CASE v_code
        WHEN 1 THEN v_label := 'One';
        WHEN 2 THEN v_label := 'Two';
        WHEN 3 THEN v_label := 'Three';
    END CASE;

    DBMS_OUTPUT.PUT_LINE(v_label);
END;
/
```

### Expected output
```
ORA-06592: CASE not found while executing CASE statement
```
`v_code = 4` matches none of the three `WHEN` clauses, and there's no `ELSE`. Unlike an `IF` chain (which just does nothing if nothing matches), a `CASE` **statement** with no matching branch and no `ELSE` raises `CASE_NOT_FOUND` at runtime. Since this block has no `EXCEPTION` section, the error propagates and terminates the block — `DBMS_OUTPUT.PUT_LINE(v_label)` never executes, and `v_label` is never even assigned.

---

## B3 — Predict: `WHILE` Loop That Never Runs

No table needed.

### The block (as given)
```sql
DECLARE
    v_remaining NUMBER := 0;
BEGIN
    WHILE v_remaining > 0 LOOP
        DBMS_OUTPUT.PUT_LINE('Processing... remaining: ' || v_remaining);
        v_remaining := v_remaining - 1;
    END LOOP;

    DBMS_OUTPUT.PUT_LINE('Loop finished.');
END;
/
```

### Expected output
```
Loop finished.
```
A `WHILE` loop tests its condition **before** the first iteration. `0 > 0` is `FALSE` immediately, so the body runs **zero** times — no "Processing..." lines at all — and control falls straight through to `'Loop finished.'`.

---

## B4 — Predict: Nested Block Re-declared Inside a `FOR` Loop

No table needed.

### The block (as given)
```sql
BEGIN
    FOR i IN 1..3 LOOP
        DECLARE
            v_inner NUMBER := 0;
        BEGIN
            v_inner := v_inner + i;
            DBMS_OUTPUT.PUT_LINE('i=' || i || ', v_inner=' || v_inner);
        END;
    END LOOP;
END;
/
```

### Expected output
```
i=1, v_inner=1
i=2, v_inner=2
i=3, v_inner=3
```
Because `v_inner` is declared **inside** the loop body, it's a brand-new variable each iteration — allocated fresh, initialized to `0`, used, and then it goes out of scope entirely when that inner block ends. There is no accumulation across iterations; each `v_inner` only ever "remembers" the current `i`, which is why the output tracks `i` exactly rather than growing (1, then 1+2=3, then 1+2+3=6, etc.).

---

## C1 — Debug: Invalid Keyword `ELSEIF`

### The block (as given) and why it fails
```sql
-- AS GIVEN (does not compile)
DECLARE
    v_day NUMBER := 6;
    v_result VARCHAR2(10);
BEGIN
    IF v_day = 6 THEN
        v_result := 'Weekend';
    ELSEIF v_day = 7 THEN     -- PL/SQL has no ELSEIF keyword
        v_result := 'Weekend';
    ELSE
        v_result := 'Weekday';
    END IF;

    DBMS_OUTPUT.PUT_LINE(v_result);
END;
/
```
```
PLS-00103: Encountered the symbol "V_DAY" ...
```
PL/SQL's keyword is `ELSIF` — one word, no space, no "E" on the end of "else." `ELSEIF` (and `ELSE IF`) are not recognized, so the parser fails right at that line.

### Fix
```sql
DECLARE
    v_day NUMBER := 6;
    v_result VARCHAR2(10);
BEGIN
    IF v_day = 6 THEN
        v_result := 'Weekend';
    ELSIF v_day = 7 THEN
        v_result := 'Weekend';
    ELSE
        v_result := 'Weekday';
    END IF;

    DBMS_OUTPUT.PUT_LINE(v_result);
END;
/
```

### Expected output
```
Weekend
```
(with `v_day := 3`, output would be `Weekday`; with `v_day := 7`, output would be `Weekend`.)

---

## C2 — Debug: Simple `CASE` Missing `ELSE`

### The function (as given) and test
```sql
-- AS GIVEN (buggy)
CREATE OR REPLACE FUNCTION classify_batch (p_size NUMBER)
    RETURN VARCHAR2
IS
    v_result VARCHAR2(20);
BEGIN
    CASE
        WHEN p_size > 1000 THEN v_result := 'LARGE';
        WHEN p_size > 100 THEN v_result := 'MEDIUM';
        WHEN p_size > 0 THEN v_result := 'SMALL';
    END CASE;

    RETURN v_result;
END classify_batch;
/

BEGIN
    DBMS_OUTPUT.PUT_LINE(classify_batch(2000)); -- LARGE
    DBMS_OUTPUT.PUT_LINE(classify_batch(50));   -- SMALL
    DBMS_OUTPUT.PUT_LINE(classify_batch(0));    -- fails here
END;
/
```
### Expected output (before fix)
```
LARGE
SMALL
ORA-06592: CASE not found while executing CASE statement
```
`p_size = 0` (or any negative value) satisfies none of the three `WHEN` conditions, and there's no `ELSE` — same underlying rule as B2, just inside a function instead of an anonymous block. This is exactly the kind of bug that "occasionally" shows up in production: it only fires for inputs nobody thought to test.

### Fix
```sql
CASE
    WHEN p_size > 1000 THEN v_result := 'LARGE';
    WHEN p_size > 100 THEN v_result := 'MEDIUM';
    WHEN p_size > 0 THEN v_result := 'SMALL';
    ELSE v_result := 'INVALID';
END CASE;
```
### Expected output (after fix)
```
LARGE
SMALL
INVALID
```

---

## C3 — Debug: `REVERSE` Doesn't Swap the Bounds

### The block (as given) and why nothing prints
```sql
-- AS GIVEN (compiles, but prints nothing)
BEGIN
    FOR i IN REVERSE 10..1 LOOP
        DBMS_OUTPUT.PUT_LINE(i);
    END LOOP;
END;
/
```
### Expected output (before fix)
```
(no output at all — not an error, the loop simply never executes)
```
`REVERSE` only changes the **direction** the loop counts in; the bounds must still be written `low..high`. `10..1` is a range where the low bound is greater than the high bound, which PL/SQL treats as an empty range regardless of `REVERSE` — so the loop body runs zero times, silently.

### Fix
```sql
BEGIN
    FOR i IN REVERSE 1..10 LOOP
        DBMS_OUTPUT.PUT_LINE(i);
    END LOOP;
END;
/
```
### Expected output (after fix)
```
10
9
8
7
6
5
4
3
2
1
```

---

## C4 — Debug: `%TYPE` Needs a Column, Not a Table

### Sample table (for context)
```sql
CREATE TABLE employees (
    employee_id NUMBER PRIMARY KEY,
    salary      NUMBER(10,2)
);
```

### The block (as given) and why it fails
```sql
-- AS GIVEN (does not compile)
DECLARE
    v_salary employees%TYPE;   -- %TYPE needs a column (or a scalar), not a whole table
BEGIN
    NULL;
END;
/
```
```
PLS-00320: the declaration of the type of this expression is incomplete or malformed
```
`%TYPE` anchors a variable to the datatype of **one specific column** (or another scalar variable/constant) — it has no meaning applied to an entire table. Anchoring to a whole row's shape is what `%ROWTYPE` is for, but that returns a record, not a scalar `NUMBER`-like variable, which isn't what was intended here anyway ("always matches `employees.salary`'s exact datatype").

### Fix
```sql
DECLARE
    v_salary employees.salary%TYPE;
BEGIN
    NULL;
END;
/
```
### Expected output
```
(compiles cleanly — no output, since the block only declares v_salary)
```
`v_salary` now has exactly `employees.salary`'s datatype and precision (`NUMBER(10,2)`), and will stay in sync automatically if that column's definition ever changes, without this line needing an edit.

---

## Section D — Judgment & Reasoning

D1–D4 are prose/reasoning questions with no query or table to run — nothing here to seed or execute. Happy to review your written answers to these directly when you share them.

---

## E1 — Prepaid Top-Up Batch (Telecom)

### Sample table
```sql
CREATE TABLE prepaid_accounts (
    account_id      NUMBER PRIMARY KEY,
    balance         NUMBER,
    plan_type       VARCHAR2(20),
    last_topup_date DATE
);

INSERT INTO prepaid_accounts VALUES (7001, 50.00, 'PREMIUM',    DATE '2026-08-01');
INSERT INTO prepaid_accounts VALUES (7002, 20.00, 'STANDARD',   DATE '2026-08-15');
INSERT INTO prepaid_accounts VALUES (7003, 10.00, 'BASIC',      DATE '2026-08-20');
INSERT INTO prepaid_accounts VALUES (7004, 5.00,  'ENTERPRISE', DATE '2026-07-01'); -- unrecognized plan
INSERT INTO prepaid_accounts VALUES (7005, 15.00, NULL,         DATE '2026-08-25'); -- plan data missing
INSERT INTO prepaid_accounts VALUES (7020, 30.00, 'STANDARD',   DATE '2026-09-01');
-- 7006-7019 and 7021-7025 deliberately absent to test the "doesn't exist" path
COMMIT;
```

### Answer query
```sql
-- IF/ELSIF chosen over CASE deliberately: a simple CASE selector can never
-- match a NULL value via WHEN (equality with NULL is never true), so a plain
-- CASE has no clean way to tell "missing plan data" apart from "plan value
-- present but unrecognized" -- both would just fall to the same ELSE with no
-- way to label them differently. An explicit IS NULL check up front, ahead of
-- an ELSIF chain, can distinguish the two cases on purpose.
--
-- Existence is checked via COUNT(*) rather than an exception handler, since
-- exception handling is Module 4 -- this stays within Module 1 & 2 tools.
DECLARE
    v_count     NUMBER;
    v_plan_type prepaid_accounts.plan_type%TYPE;
    v_bonus_pct NUMBER;
    v_topup_amt CONSTANT NUMBER := 100;
BEGIN
    FOR v_id IN 7000..7025 LOOP
        SELECT COUNT(*) INTO v_count FROM prepaid_accounts WHERE account_id = v_id;

        IF v_count = 0 THEN
            DBMS_OUTPUT.PUT_LINE('Account ' || v_id || ': NOT FOUND - skipping.');
            CONTINUE;
        END IF;

        SELECT plan_type INTO v_plan_type FROM prepaid_accounts WHERE account_id = v_id;

        IF v_plan_type IS NULL THEN
            DBMS_OUTPUT.PUT_LINE('Account ' || v_id || ': PLAN_DATA_MISSING - skipping bonus calc.');
            CONTINUE;
        ELSIF v_plan_type = 'PREMIUM' THEN
            v_bonus_pct := 0.20;
        ELSIF v_plan_type = 'STANDARD' THEN
            v_bonus_pct := 0.10;
        ELSIF v_plan_type = 'BASIC' THEN
            v_bonus_pct := 0.05;
        ELSE
            DBMS_OUTPUT.PUT_LINE('Account ' || v_id || ': UNRECOGNIZED_PLAN (' || v_plan_type ||
                                  ') - flagging, no bonus applied.');
            CONTINUE;
        END IF;

        UPDATE prepaid_accounts
           SET balance = balance + (v_topup_amt * (1 + v_bonus_pct)),
               last_topup_date = SYSDATE
         WHERE account_id = v_id;

        DBMS_OUTPUT.PUT_LINE('Account ' || v_id || ': topped up with bonus ' || (v_bonus_pct * 100) ||
                              '% -> credited ' || (v_topup_amt * (1 + v_bonus_pct)));
    END LOOP;
END;
/
```

### Expected output (abbreviated — 7006–7019 and 7021–7025 all repeat "NOT FOUND")
```
Account 7001: topped up with bonus 20% -> credited 120
Account 7002: topped up with bonus 10% -> credited 110
Account 7003: topped up with bonus 5% -> credited 105
Account 7004: UNRECOGNIZED_PLAN (ENTERPRISE) - flagging, no bonus applied.
Account 7005: PLAN_DATA_MISSING - skipping bonus calc.
Account 7006: NOT FOUND - skipping.
...
Account 7020: topped up with bonus 10% -> credited 110
Account 7021: NOT FOUND - skipping.
...
Account 7025: NOT FOUND - skipping.
```

### Design write-up (part 5)
The batch never halts on a bad account because every per-account outcome — not found, missing plan data, unrecognized plan, or a normal top-up — is resolved with `CONTINUE` or falls through to the loop's next iteration on its own; nothing here can raise an unhandled error that would abort the whole `FOR` loop, since the existence check happens before any fetch that could fail. The two "something's wrong with plan data" cases are kept distinct on purpose: `PLAN_DATA_MISSING` (caught by the `IS NULL` check, which must run first since equality checks can't detect `NULL`) means *this specific record is incomplete* — a data-entry problem to fix on that one account. `UNRECOGNIZED_PLAN` (caught by the final `ELSE`, only reachable once `NULL` has already been ruled out) means *the value itself is valid-looking but this batch's logic hasn't been taught about it yet* — a signal to update the routing/bonus table, not to fix a specific customer's record. Collapsing both into one message would hide which of those two very different follow-up actions is actually needed.

**Concepts from Modules 1 & 2 in play:** block structure (the loop body plus the flow of control via `CONTINUE`); datatypes and `%TYPE` (`v_plan_type` anchored to the column); a `CONSTANT` for the fixed top-up amount; comparison and NULL-aware operators (`IS NULL`, `=`); `IF`/`ELSIF` selection chosen over `CASE` for the reason above; numeric `FOR` iteration over a fixed ID range; and `CONTINUE` as iteration control to skip the remaining loop body for a given account without exiting the loop entirely.