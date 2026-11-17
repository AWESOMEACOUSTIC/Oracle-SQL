# PL/SQL Scenario-Based Questions

**Topics covered:** Procedures · Functions · Packages · Composite Data Types · BULK COLLECT & FORALL · Triggers · Cursor FOR UPDATE

| # | Scenario | Procedure | Function | Package | Composite Types | BULK COLLECT / FORALL | Trigger | Cursor FOR UPDATE |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| 1 | Department Salary Revision | ✅ | ✅ | ✅ | | | ✅ | ✅ |
| 2 | Bulk Order Archival | ✅ | | | ✅ | ✅ | | |
| 3 | Customer Tier Refresh | ✅ | ✅ | ✅ | ✅ | ✅ | | |
| 4 | Inventory Auto-Reorder | | | | ✅ | ✅ | ✅ | |
| 5 | Year-End Bonus Payout | ✅ | ✅ | | ✅ | | | ✅ |
| 6 | Department Restructuring | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | |

---

## Setup: Tables and Sample Data

Run this once before attempting the scenarios. Each scenario changes data, so **reload the sample data before each scenario** if your results look different from the expected output.

```sql
CREATE TABLE departments (
  dept_id    NUMBER PRIMARY KEY,
  dept_name  VARCHAR2(50),
  budget     NUMBER(12,2)
);

CREATE TABLE employees (
  emp_id     NUMBER PRIMARY KEY,
  emp_name   VARCHAR2(50),
  dept_id    NUMBER REFERENCES departments(dept_id),
  salary     NUMBER(10,2),
  hire_date  DATE,
  status     VARCHAR2(10) DEFAULT 'ACTIVE'
);

CREATE TABLE salary_audit (
  audit_id   NUMBER GENERATED ALWAYS AS IDENTITY,
  emp_id     NUMBER,
  old_salary NUMBER(10,2),
  new_salary NUMBER(10,2),
  changed_by VARCHAR2(30),
  changed_on DATE
);

CREATE TABLE customers (
  cust_id    NUMBER PRIMARY KEY,
  cust_name  VARCHAR2(50),
  tier       VARCHAR2(10)
);

CREATE TABLE orders (
  order_id   NUMBER PRIMARY KEY,
  cust_id    NUMBER REFERENCES customers(cust_id),
  order_date DATE,
  amount     NUMBER(10,2),
  status     VARCHAR2(15)
);

CREATE TABLE orders_archive (
  order_id   NUMBER PRIMARY KEY,
  cust_id    NUMBER,
  order_date DATE,
  amount     NUMBER(10,2),
  status     VARCHAR2(15)
);

CREATE TABLE products (
  product_id    NUMBER PRIMARY KEY,
  product_name  VARCHAR2(50),
  qty_on_hand   NUMBER,
  reorder_level NUMBER
);

CREATE TABLE reorder_requests (
  req_id     NUMBER GENERATED ALWAYS AS IDENTITY,
  product_id NUMBER,
  qty_needed NUMBER,
  req_date   DATE
);

CREATE TABLE error_log (
  log_id     NUMBER GENERATED ALWAYS AS IDENTITY,
  source     VARCHAR2(50),
  ref_id     NUMBER,
  err_code   NUMBER,
  err_msg    VARCHAR2(4000),
  logged_on  DATE DEFAULT SYSDATE
);
```

```sql
-- Departments
INSERT INTO departments VALUES (10, 'Engineering', 500000);
INSERT INTO departments VALUES (20, 'Sales',       200000);
INSERT INTO departments VALUES (30, 'Support',     150000);

-- Employees
INSERT INTO employees VALUES (101, 'Ravi',  10, 90000, DATE '2017-04-10', 'ACTIVE');
INSERT INTO employees VALUES (102, 'Anu',   10, 75000, DATE '2021-08-01', 'ACTIVE');
INSERT INTO employees VALUES (103, 'Kiran', 10, 60000, SYSDATE - 60,      'ACTIVE');   -- on probation
INSERT INTO employees VALUES (104, 'Meena', 20, 55000, DATE '2019-01-15', 'ACTIVE');
INSERT INTO employees VALUES (105, 'Arjun', 20, 48000, DATE '2023-03-20', 'ACTIVE');
INSERT INTO employees VALUES (106, 'Divya', 30, 40000, DATE '2015-06-30', 'INACTIVE');

-- Customers and orders
INSERT INTO customers VALUES (1, 'Acme Corp', NULL);
INSERT INTO customers VALUES (2, 'Globex',    NULL);
INSERT INTO customers VALUES (3, 'Initech',   NULL);

INSERT INTO orders VALUES (1001, 1, DATE '2024-01-10', 60000, 'DELIVERED');
INSERT INTO orders VALUES (1002, 1, DATE '2024-05-22', 45000, 'DELIVERED');
INSERT INTO orders VALUES (1003, 2, DATE '2024-02-14', 12000, 'DELIVERED');
INSERT INTO orders VALUES (1004, 2, DATE '2025-11-01',  8000, 'SHIPPED');
INSERT INTO orders VALUES (1005, 3, DATE '2023-12-31',  3000, 'DELIVERED');

-- Order 1005 is already archived, so archiving it again causes a duplicate key error
INSERT INTO orders_archive VALUES (1005, 3, DATE '2023-12-31', 3000, 'DELIVERED');

-- Products
INSERT INTO products VALUES (1, 'Keyboard', 50, 20);
INSERT INTO products VALUES (2, 'Mouse',    40, 25);
INSERT INTO products VALUES (3, 'Monitor',  15, 10);
INSERT INTO products VALUES (4, 'Webcam',   30, 10);

COMMIT;
SET SERVEROUTPUT ON;
```

---

# PART A: QUESTIONS

## Scenario 1: Department Salary Revision
**Topics:** Package · Procedure · Function · Cursor FOR UPDATE · Trigger · Savepoint

The HR team wants a package called `HR_SALARY_PKG` to run department-wide salary revisions safely. The package must contain a function `GET_DEPT_PAYROLL(p_dept_id)` that returns the total salary of all **ACTIVE** employees in a department (it should return 0, not NULL, when a department has no active employees). The package must also contain a procedure `APPLY_RAISE(p_dept_id, p_pct)`. This procedure first validates that the percentage is positive and that the department exists, raising meaningful application errors otherwise. It then locks every active employee of that department using an explicit cursor declared `FOR UPDATE OF salary`, so that no other session can change those salaries while the revision is running. It increases each salary by `p_pct` percent using `WHERE CURRENT OF`. After all raises are applied, it calls `GET_DEPT_PAYROLL` to check whether the department's new payroll exceeds its budget in `DEPARTMENTS`. If the budget is exceeded, only the raises must be undone (use a savepoint) and the procedure must raise error **-20001**, which the package spec exposes as a named exception `E_BUDGET_EXCEEDED` so callers can handle it by name. Otherwise the changes are committed. Separately, the company wants every salary change on `EMPLOYEES`, from this package or from any other program or ad-hoc SQL, recorded in `SALARY_AUDIT` with the old salary, new salary, the user who made the change and the date. Updates that don't actually change the salary should not be recorded. Write the trigger, the package spec and body, and a test block. Then explain **what happens to the audit rows when the raise is cancelled because of the budget check**, and **why `FOR UPDATE` is needed here**.

---

## Scenario 2: Bulk Order Archival
**Topics:** Procedure (IN/OUT params) · Collection of `%ROWTYPE` · User-defined record collection · BULK COLLECT LIMIT · FORALL SAVE EXCEPTIONS · FORALL INDICES OF

The `ORDERS` table has grown to millions of rows and is slowing down reports. Write a standalone procedure `ARCHIVE_ORDERS(p_cutoff_date IN DATE, p_rows_archived OUT NUMBER)` that moves every order with status `'DELIVERED'` and an `order_date` earlier than `p_cutoff_date` from `ORDERS` into `ORDERS_ARCHIVE`, and then deletes those orders from `ORDERS`. Because of the volume, the procedure must not load everything into memory at once. It must fetch the orders in **batches of 1000** with `BULK COLLECT ... LIMIT` into a nested table of `orders%ROWTYPE`, and insert each batch into the archive with a single `FORALL` statement. Some orders may already exist in `ORDERS_ARCHIVE` from a previous partial run, and inserting them again raises a primary key violation. Such a failure must not stop the rest of the batch: use `SAVE EXCEPTIONS`, and log every failed order into `ERROR_LOG` with its order id, error code and error message. Crucially, an order that **failed** to archive must **not** be deleted from `ORDERS`, so the delete step must also be a bulk operation that skips the failed elements. The procedure should commit after each batch, return the total number of successfully archived (and deleted) orders through the OUT parameter, and roll back the current batch and re-raise on any unexpected error. Write the procedure and a test block that archives orders before `01-JAN-2025`, and state the expected result for the sample data.

---

## Scenario 3: Customer Tier Refresh
**Topics:** Package (public type, private function) · User-defined record with defaults · Function returning a record · BULK COLLECT · FORALL · Associative array indexed by VARCHAR2

The marketing team wants to classify customers into loyalty tiers based on their delivered orders. Create a package `CUSTOMER_TIER_PKG` whose **specification** declares a public user-defined record type `T_CUST_STATS` with the fields `cust_id`, `order_count`, `total_spent`, `avg_order` and `tier`. Each numeric field defaults to 0 and `tier` defaults to `'BRONZE'`. Anchor the fields to table columns wherever possible. The package exposes a function `GET_CUST_STATS(p_cust_id)` that returns a `T_CUST_STATS` record with the number of delivered orders, the total and average delivered amount, and the tier: **PLATINUM** for a total of 100000 or more, **GOLD** for 50000 or more, **SILVER** for 10000 or more, and **BRONZE** otherwise. The tier logic must live in a **private** function that callers outside the package cannot see. A customer with no delivered orders must get a BRONZE record with zero values, without any exception being raised. The package also exposes a procedure `REFRESH_ALL_TIERS` that loads all customer ids with a single `BULK COLLECT`, calls `GET_CUST_STATS` for each customer, and then updates `CUSTOMERS.TIER` for every customer with **one `FORALL` statement** (not one UPDATE per loop iteration). Finally, it prints a summary of how many customers fell into each tier, using an associative array indexed by the tier name. Write the package spec and body and a test call. Then explain **why `GET_CUST_STATS` will never raise `NO_DATA_FOUND`**, and **why a record type is the right return type for this function**.

---

## Scenario 4: Inventory Auto-Reorder with Mutating-Table Protection
**Topics:** Compound trigger · Collection of user-defined records · FORALL · Statement-level validation

The warehouse system updates `PRODUCTS.QTY_ON_HAND` whenever stock moves, often many products in a single UPDATE statement. Two rules must be enforced automatically at the database level. First, when a product's quantity **drops from at or above its reorder level to below it** during an update, a row must be added to `REORDER_REQUESTS` for `(reorder_level × 2 − new qty_on_hand)` units. Products that were already below the reorder level must not generate duplicate requests. Second, as a safety rule, no single UPDATE statement may leave **more than half of all products** in the `PRODUCTS` table below their reorder level. If an update would do that, the entire UPDATE statement must be rejected with error **-20010** and no reorder requests may be created. A developer first tried to put a `SELECT COUNT(*) FROM products` inside a normal `FOR EACH ROW` trigger and got an error. Explain which error occurs and why, then write a **single compound trigger** that solves both rules. It should collect the reorder requests into a collection of user-defined records at the row level, run the safety check at the statement level, and insert all requests with one `FORALL` statement. Finally, using the sample data, show one UPDATE that succeeds and creates a reorder request, and one UPDATE that is rejected. Explain what happens to the rejected statement's row changes.

---

## Scenario 5: Year-End Bonus Payout
**Topics:** Function with `%ROWTYPE` parameter · Procedure with OUT params · Cursor FOR UPDATE NOWAIT · `WHERE CURRENT OF` · `PRAGMA EXCEPTION_INIT` · Autonomous transaction

Finance wants to pay year-end bonuses department by department. Write a function `CALC_BONUS(p_emp IN employees%ROWTYPE) RETURN NUMBER` that takes a **whole employee record** as its parameter and calculates the bonus from completed years of service: under 2 years gets no bonus, 2 to 5 years gets 5% of salary, and more than 5 years gets 10% of salary, with the bonus capped at 50000. It should safely return 0 if the hire date or salary is NULL. Then write a procedure `PAY_BONUS(p_dept_id IN NUMBER, p_paid_count OUT PLS_INTEGER, p_total_bonus OUT NUMBER)` that opens a cursor on the department's active employees with `FOR UPDATE OF salary NOWAIT`. For each employee it passes the cursor row to `CALC_BONUS` and adds any non-zero bonus to the salary using `WHERE CURRENT OF`. It returns the number of employees paid and the total bonus amount, and commits at the end. Payroll often runs while HR staff are editing employee records. If any of the department's rows are **already locked by another session**, the procedure must not wait. It must catch that specific Oracle error by giving it a name with `PRAGMA EXCEPTION_INIT`, roll back, set both OUT parameters to 0, and write an entry to `ERROR_LOG`. That log entry must **survive the rollback**, so the logging must happen in a separate procedure `LOG_ERROR` that runs as an **autonomous transaction**. Write `LOG_ERROR`, `CALC_BONUS`, `PAY_BONUS` and a test block. Then explain: **which Oracle error number `NOWAIT` raises and at what point**, **how `SKIP LOCKED` would change the behaviour**, and, if the Scenario 1 audit trigger exists, **what happens to `SALARY_AUDIT` rows when the bonus run succeeds versus when it fails**.

---

## Scenario 6: Department Restructuring
**Topics:** Package (public collection type, private variable, private procedure, initialization block) · Overloaded procedures · Function returning a collection · BULK COLLECT · FORALL with RETURNING BULK COLLECT INTO · Row-level trigger · Savepoint

The company is restructuring departments and needs a package `DEPT_RESTRUCTURE_PKG`. Its spec declares a public collection type `T_EMP_TAB` as a nested table of `employees%ROWTYPE`, and a function `GET_DEPT_EMPLOYEES(p_dept_id)` that returns all active employees of a department, ordered by salary descending, using `BULK COLLECT`. The package must offer an **overloaded** procedure `TRANSFER`. The first version, `TRANSFER(p_emp_id, p_new_dept)`, moves a single employee and raises an error if the employee does not exist. The second version, `TRANSFER(p_from_dept, p_to_dept, p_min_salary)`, reuses `GET_DEPT_EMPLOYEES` to pick every active employee of the source department earning at least `p_min_salary`. It moves them all with **one `FORALL` UPDATE**, uses `RETURNING ... BULK COLLECT INTO` to capture the names of the moved employees, and prints them. Both versions must check that the target department exists, using a **private** helper procedure, and both must count the number of employees transferred in the current session in a **private package variable**. That variable is initialized in the package's **initialization section** and can be read only through a public function `GET_SESSION_TRANSFERS`. Company policy says employees hired within the **last 6 months are on probation and cannot change department**, and this must be enforced by a row-level trigger on `EMPLOYEES` no matter which program attempts the change. The bulk transfer must be **all-or-nothing**: if even one employee in the batch is on probation, nobody in that batch is moved. Write the trigger, the package spec and body, and a test block. Then explain exactly **what a `FORALL` statement without `SAVE EXCEPTIONS` does when one iteration fails midway**, and **why the session transfer counter would be wrong if it were incremented before the `FORALL`**.

---
---

# PART B: ANSWERS

## Answer 1: Department Salary Revision

### Trigger: audit every salary change
```sql
CREATE OR REPLACE TRIGGER trg_salary_audit
AFTER UPDATE OF salary ON employees
FOR EACH ROW
WHEN (NVL(OLD.salary, -1) <> NVL(NEW.salary, -1))   -- no colons inside WHEN
BEGIN
  INSERT INTO salary_audit (emp_id, old_salary, new_salary, changed_by, changed_on)
  VALUES (:OLD.emp_id, :OLD.salary, :NEW.salary, USER, SYSDATE);
END;
/
```

### Package specification
```sql
CREATE OR REPLACE PACKAGE hr_salary_pkg AS
  -- Named exception so callers can write: WHEN hr_salary_pkg.e_budget_exceeded
  e_budget_exceeded EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_budget_exceeded, -20001);

  FUNCTION get_dept_payroll (p_dept_id IN departments.dept_id%TYPE) RETURN NUMBER;

  PROCEDURE apply_raise (p_dept_id IN departments.dept_id%TYPE,
                         p_pct     IN NUMBER);
END hr_salary_pkg;
/
```

### Package body
```sql
CREATE OR REPLACE PACKAGE BODY hr_salary_pkg AS

  FUNCTION get_dept_payroll (p_dept_id IN departments.dept_id%TYPE) RETURN NUMBER IS
    l_total NUMBER;
  BEGIN
    SELECT NVL(SUM(salary), 0)
    INTO   l_total
    FROM   employees
    WHERE  dept_id = p_dept_id
      AND  status  = 'ACTIVE';

    RETURN l_total;
  END get_dept_payroll;


  PROCEDURE apply_raise (p_dept_id IN departments.dept_id%TYPE,
                         p_pct     IN NUMBER) IS

    CURSOR c_emp IS
      SELECT emp_id, salary
      FROM   employees
      WHERE  dept_id = p_dept_id
        AND  status  = 'ACTIVE'
      FOR UPDATE OF salary;               -- locks the rows when the cursor opens

    l_budget departments.budget%TYPE;
    l_count  PLS_INTEGER := 0;
  BEGIN
    IF p_pct IS NULL OR p_pct <= 0 THEN
      RAISE_APPLICATION_ERROR(-20002, 'Raise percentage must be positive');
    END IF;

    BEGIN
      SELECT budget INTO l_budget
      FROM   departments
      WHERE  dept_id = p_dept_id;
    EXCEPTION
      WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20003, 'Department ' || p_dept_id || ' does not exist');
    END;

    SAVEPOINT before_raise;

    FOR r IN c_emp LOOP
      UPDATE employees
      SET    salary = salary * (1 + p_pct / 100)
      WHERE  CURRENT OF c_emp;            -- updates exactly the row just fetched

      l_count := l_count + 1;
    END LOOP;

    -- The function sees this session's uncommitted changes
    IF get_dept_payroll(p_dept_id) > l_budget THEN
      ROLLBACK TO before_raise;           -- undoes the raises AND the audit rows
      RAISE_APPLICATION_ERROR(-20001,
        'Raise cancelled: payroll would exceed the budget of ' || l_budget);
    END IF;

    COMMIT;                               -- releases the FOR UPDATE locks
    DBMS_OUTPUT.PUT_LINE(l_count || ' employee(s) updated in department ' || p_dept_id);
  END apply_raise;

END hr_salary_pkg;
/
```

### Test block
```sql
-- Succeeds: Engineering payroll 225000 -> 236250 (budget 500000)
EXEC hr_salary_pkg.apply_raise(10, 5);

-- Fails: 236250 * 2.5 = 590625 > 500000
BEGIN
  hr_salary_pkg.apply_raise(10, 150);
EXCEPTION
  WHEN hr_salary_pkg.e_budget_exceeded THEN
    DBMS_OUTPUT.PUT_LINE('Handled: ' || SQLERRM);
END;
/

SELECT * FROM salary_audit;   -- only the 3 rows from the successful 5% raise
```

### Explanation
- **Audit rows when the raise is cancelled:** The trigger's INSERT into `SALARY_AUDIT` runs in the **same transaction** as the UPDATE that fired it. `ROLLBACK TO before_raise` undoes everything after the savepoint, so the salary updates and the audit rows it generated are all removed. No orphan audit rows are left for raises that never happened.
- **Why `FOR UPDATE`:** Without it, another session could change an employee's salary between our read and our update. One of the two changes would then be silently lost (a "lost update"), and the budget check would be based on stale numbers. `FOR UPDATE` locks the rows when the cursor opens and holds the locks until COMMIT or ROLLBACK.
- **`WHERE CURRENT OF`** works only with a cursor declared `FOR UPDATE`. It targets the row's physical location, so it needs no extra lookup on the primary key.
- `ROLLBACK TO before_raise` also **releases the locks** taken after the savepoint, which includes the cursor's locks because the cursor opened after it.
- The savepoint is set **after** validation, so the `ROLLBACK TO` can never point to a savepoint that doesn't exist.

---

## Answer 2: Bulk Order Archival

```sql
CREATE OR REPLACE PROCEDURE archive_orders (
  p_cutoff_date   IN  DATE,
  p_rows_archived OUT NUMBER
) IS
  CURSOR c_old_orders IS
    SELECT *
    FROM   orders
    WHERE  status     = 'DELIVERED'
      AND  order_date < p_cutoff_date;

  -- Composite types
  TYPE t_order_tab IS TABLE OF orders%ROWTYPE;          -- nested table of rows
  TYPE t_id_tab    IS TABLE OF orders.order_id%TYPE;    -- nested table of ids

  TYPE t_err_rec IS RECORD (                            -- user-defined record
    idx   PLS_INTEGER,
    code  PLS_INTEGER
  );
  TYPE t_err_tab IS TABLE OF t_err_rec INDEX BY PLS_INTEGER;

  l_orders     t_order_tab;
  l_ids        t_id_tab;
  l_errors     t_err_tab;
  l_err_count  PLS_INTEGER;
  l_err_msg    VARCHAR2(4000);

  e_bulk_errors EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_bulk_errors, -24381);         -- raised by SAVE EXCEPTIONS

  c_batch_size CONSTANT PLS_INTEGER := 1000;
BEGIN
  p_rows_archived := 0;

  OPEN c_old_orders;
  LOOP
    FETCH c_old_orders BULK COLLECT INTO l_orders LIMIT c_batch_size;
    EXIT WHEN l_orders.COUNT = 0;          -- NOT %NOTFOUND (would skip the last partial batch)

    -- Copy the ids into a separate collection for the delete step
    l_ids := t_id_tab();
    l_ids.EXTEND(l_orders.COUNT);
    FOR i IN 1 .. l_orders.COUNT LOOP
      l_ids(i) := l_orders(i).order_id;
    END LOOP;

    -- Step 1: bulk insert, continuing past individual failures
    BEGIN
      FORALL i IN 1 .. l_orders.COUNT SAVE EXCEPTIONS
        INSERT INTO orders_archive VALUES l_orders(i);   -- record-based insert
    EXCEPTION
      WHEN e_bulk_errors THEN
        -- Copy the error details first, before running any other SQL
        l_errors.DELETE;
        l_err_count := SQL%BULK_EXCEPTIONS.COUNT;
        FOR j IN 1 .. l_err_count LOOP
          l_errors(j).idx  := SQL%BULK_EXCEPTIONS(j).ERROR_INDEX;
          l_errors(j).code := SQL%BULK_EXCEPTIONS(j).ERROR_CODE;
        END LOOP;

        FOR j IN 1 .. l_err_count LOOP
          l_err_msg := SQLERRM(-l_errors(j).code);       -- SQLERRM can't be used inside SQL
          INSERT INTO error_log (source, ref_id, err_code, err_msg)
          VALUES ('ARCHIVE_ORDERS', l_orders(l_errors(j).idx).order_id,
                  l_errors(j).code, l_err_msg);

          l_ids.DELETE(l_errors(j).idx);                 -- makes l_ids sparse
        END LOOP;
    END;

    -- Step 2: delete only the successfully archived orders
    IF l_ids.COUNT > 0 THEN
      FORALL i IN INDICES OF l_ids                       -- skips the deleted (failed) elements
        DELETE FROM orders WHERE order_id = l_ids(i);

      p_rows_archived := p_rows_archived + SQL%ROWCOUNT;
    END IF;

    COMMIT;                                              -- commit per batch
  END LOOP;
  CLOSE c_old_orders;

EXCEPTION
  WHEN OTHERS THEN
    IF c_old_orders%ISOPEN THEN
      CLOSE c_old_orders;
    END IF;
    ROLLBACK;            -- undoes only the current, uncommitted batch
    RAISE;
END archive_orders;
/
```

### Test block
```sql
DECLARE
  l_count NUMBER;
BEGIN
  archive_orders(DATE '2025-01-01', l_count);
  DBMS_OUTPUT.PUT_LINE('Archived: ' || l_count);
END;
/

SELECT * FROM error_log;   -- order 1005, ORA-00001 unique constraint violated
SELECT * FROM orders;      -- 1004 (SHIPPED) and 1005 (failed) remain
```

**Expected:** Orders 1001, 1002, 1003 and 1005 qualify. Order 1005 already exists in the archive, so it fails, gets logged and is **not** deleted. Output: `Archived: 3`.

### Explanation
- **`LIMIT`** keeps memory use bounded: only 1000 rows are in the PGA at any time.
- **`EXIT WHEN l_orders.COUNT = 0`** is the correct exit test. `%NOTFOUND` becomes TRUE as soon as a fetch returns fewer rows than the LIMIT, so testing it right after the fetch would skip the last partial batch.
- **`SAVE EXCEPTIONS`** lets the FORALL finish all iterations and then raise ORA-24381 once. `SQL%BULK_EXCEPTIONS(j).ERROR_INDEX` gives the **position in the collection** of each failed element.
- **`INDICES OF`** is needed because `l_ids` becomes sparse after `.DELETE(idx)`. `FORALL i IN 1..l_ids.COUNT` would raise an error when it reached a deleted element.
- Committing while a cursor is still open ("fetch across commit") is allowed. On very long runs it can cause **ORA-01555 (snapshot too old)** if undo is overwritten, so the undo retention needs to be big enough.

---

## Answer 3: Customer Tier Refresh

### Package specification
```sql
CREATE OR REPLACE PACKAGE customer_tier_pkg AS

  -- Public user-defined record type with default values
  TYPE t_cust_stats IS RECORD (
    cust_id      customers.cust_id%TYPE,
    order_count  PLS_INTEGER          := 0,
    total_spent  NUMBER(12,2)         := 0,
    avg_order    NUMBER(12,2)         := 0,
    tier         customers.tier%TYPE  := 'BRONZE'
  );

  FUNCTION  get_cust_stats (p_cust_id IN customers.cust_id%TYPE) RETURN t_cust_stats;
  PROCEDURE refresh_all_tiers;

END customer_tier_pkg;
/
```

### Package body
```sql
CREATE OR REPLACE PACKAGE BODY customer_tier_pkg AS

  -- PRIVATE: not declared in the spec, so callers outside the package can't use it
  FUNCTION derive_tier (p_total IN NUMBER) RETURN VARCHAR2 IS
  BEGIN
    RETURN CASE
             WHEN p_total >= 100000 THEN 'PLATINUM'
             WHEN p_total >=  50000 THEN 'GOLD'
             WHEN p_total >=  10000 THEN 'SILVER'
             ELSE                        'BRONZE'
           END;
  END derive_tier;


  FUNCTION get_cust_stats (p_cust_id IN customers.cust_id%TYPE) RETURN t_cust_stats IS
    l_stats t_cust_stats;            -- defaults already applied: 0, 0, 0, 'BRONZE'
  BEGIN
    l_stats.cust_id := p_cust_id;

    SELECT COUNT(*), NVL(SUM(amount), 0), NVL(AVG(amount), 0)
    INTO   l_stats.order_count, l_stats.total_spent, l_stats.avg_order
    FROM   orders
    WHERE  cust_id = p_cust_id
      AND  status  = 'DELIVERED';

    l_stats.tier := derive_tier(l_stats.total_spent);
    RETURN l_stats;
  END get_cust_stats;


  PROCEDURE refresh_all_tiers IS
    TYPE t_id_tab     IS TABLE OF customers.cust_id%TYPE;
    TYPE t_tier_tab   IS TABLE OF customers.tier%TYPE;
    TYPE t_tier_count IS TABLE OF PLS_INTEGER INDEX BY VARCHAR2(10);   -- associative array

    l_ids     t_id_tab;
    l_tiers   t_tier_tab := t_tier_tab();
    l_counts  t_tier_count;
    l_stats   t_cust_stats;
    l_key     VARCHAR2(10);
  BEGIN
    SELECT cust_id BULK COLLECT INTO l_ids FROM customers;

    l_tiers.EXTEND(l_ids.COUNT);

    FOR i IN 1 .. l_ids.COUNT LOOP
      l_stats    := get_cust_stats(l_ids(i));
      l_tiers(i) := l_stats.tier;

      IF l_counts.EXISTS(l_stats.tier) THEN
        l_counts(l_stats.tier) := l_counts(l_stats.tier) + 1;
      ELSE
        l_counts(l_stats.tier) := 1;
      END IF;
    END LOOP;

    -- ONE bulk update instead of N separate updates
    FORALL i IN 1 .. l_ids.COUNT
      UPDATE customers
      SET    tier    = l_tiers(i)
      WHERE  cust_id = l_ids(i);

    DBMS_OUTPUT.PUT_LINE(SQL%ROWCOUNT || ' customer(s) updated');
    COMMIT;

    -- Walk the VARCHAR2-indexed array (keys come back in sorted order)
    l_key := l_counts.FIRST;
    WHILE l_key IS NOT NULL LOOP
      DBMS_OUTPUT.PUT_LINE(RPAD(l_key, 10) || ': ' || l_counts(l_key));
      l_key := l_counts.NEXT(l_key);
    END LOOP;
  END refresh_all_tiers;

END customer_tier_pkg;
/
```

### Test block
```sql
EXEC customer_tier_pkg.refresh_all_tiers;

-- Use the record directly
DECLARE
  l_s customer_tier_pkg.t_cust_stats;
BEGIN
  l_s := customer_tier_pkg.get_cust_stats(1);
  DBMS_OUTPUT.PUT_LINE(l_s.cust_id || ' | ' || l_s.order_count || ' orders | '
                       || l_s.total_spent || ' | ' || l_s.tier);
END;
/
```

**Expected (fresh data):** Acme = PLATINUM (105000), Globex = SILVER (12000; the SHIPPED order is ignored), Initech = BRONZE (3000).
```
3 customer(s) updated
BRONZE    : 1
PLATINUM  : 1
SILVER    : 1
```

### Explanation
- **No `NO_DATA_FOUND`:** An aggregate query (`COUNT`, `SUM`, `AVG`) **without GROUP BY always returns exactly one row**, even when no rows match. `COUNT(*)` returns 0, and `SUM` and `AVG` return NULL, which is why `NVL` is needed.
- **Why a record return type:** A function returns a single value. The requirement needs **five related values** (count, total, average, tier, id) returned together, and some of them are calculated rather than stored in any table. Neither `%ROWTYPE` nor a cursor fits that structure, so a **user-defined record** is the right choice. Declaring it in the **spec** makes it a public "contract" that callers can declare variables of.
- **Defaults** (`:= 0`, `:= 'BRONZE'`) are only possible on user-defined record fields, not on `%ROWTYPE`.
- **Parallel scalar collections** (`l_ids`, `l_tiers`) are used in the FORALL. Referencing record fields such as `l_recs(i).field` inside FORALL is allowed only from 11g onward (before that it raised PLS-00436), so this pattern is the safe classic form.
- **Performance note:** Calling `get_cust_stats` once per customer runs N queries. In a real system with many customers, a single `GROUP BY` query would be faster. The requirement asks to reuse the function, which is a common trade-off between reusability and performance.

---

## Answer 4: Inventory Auto-Reorder with Mutating-Table Protection

### Why the row-level trigger fails
A `FOR EACH ROW` trigger on `PRODUCTS` that runs `SELECT ... FROM products` raises **ORA-04091: table PRODUCTS is mutating, trigger/function may not see it**. While an UPDATE statement is in progress, the table is in an intermediate state (some rows changed, some not yet), so Oracle doesn't let a row-level trigger read it. A **statement-level** trigger section runs after all rows are processed, when the table is consistent again, so querying it there is allowed. A **compound trigger** gives us both levels in one object, with shared state between them.

### Compound trigger
```sql
CREATE OR REPLACE TRIGGER trg_products_reorder
FOR UPDATE OF qty_on_hand ON products
COMPOUND TRIGGER

  -- Shared state across all timing points (reset automatically for every statement)
  TYPE t_req_rec IS RECORD (
    product_id  products.product_id%TYPE,
    qty_needed  NUMBER
  );
  TYPE t_req_tab IS TABLE OF t_req_rec INDEX BY PLS_INTEGER;

  g_requests  t_req_tab;

  -- Row level: only collect, never query PRODUCTS here
  AFTER EACH ROW IS
    l_idx PLS_INTEGER;
  BEGIN
    IF :NEW.qty_on_hand <  :NEW.reorder_level            -- now below
       AND :OLD.qty_on_hand >= :OLD.reorder_level THEN   -- was at or above, so it just crossed
      l_idx := g_requests.COUNT + 1;
      g_requests(l_idx).product_id := :NEW.product_id;
      g_requests(l_idx).qty_needed := (:NEW.reorder_level * 2) - :NEW.qty_on_hand;
    END IF;
  END AFTER EACH ROW;

  -- Statement level: the table is stable, so querying it is allowed
  AFTER STATEMENT IS
    l_total  PLS_INTEGER;
    l_below  PLS_INTEGER;
  BEGIN
    SELECT COUNT(*),
           COUNT(CASE WHEN qty_on_hand < reorder_level THEN 1 END)
    INTO   l_total, l_below
    FROM   products;

    IF l_total > 0 AND l_below > l_total / 2 THEN
      RAISE_APPLICATION_ERROR(-20010,
        'Rejected: ' || l_below || ' of ' || l_total ||
        ' products would be below reorder level');
    END IF;

    FORALL i IN 1 .. g_requests.COUNT
      INSERT INTO reorder_requests (product_id, qty_needed, req_date)
      VALUES (g_requests(i).product_id, g_requests(i).qty_needed, SYSDATE);
  END AFTER STATEMENT;

END trg_products_reorder;
/
```

### Test
```sql
-- 1) SUCCEEDS: Keyboard 50->20 (not below 20), Mouse 40->10 (crosses below 25)
UPDATE products SET qty_on_hand = qty_on_hand - 30 WHERE product_id IN (1, 2);
SELECT * FROM reorder_requests;    -- one request: product 2, qty 25*2-10 = 40
COMMIT;

-- 2) REJECTED: Monitor 15->5 and Webcam 30->5 put 3 of 4 products below level
UPDATE products SET qty_on_hand = 5 WHERE product_id IN (3, 4);
-- ORA-20010: Rejected: 3 of 4 products would be below reorder level

SELECT product_id, qty_on_hand FROM products WHERE product_id IN (3, 4);  -- still 15 and 30
```

### Explanation
- **Rejected statement:** An error raised in any trigger section makes the **whole triggering statement fail**. Oracle performs a **statement-level rollback**: both product rows go back to their old values, and any DML the trigger did for that statement is undone too. Here the FORALL never ran, because the error is raised before it. Earlier work in the same transaction (for example, an earlier successful UPDATE that wasn't committed yet) is **not** affected; only the failing statement is undone.
- The `:OLD >= :OLD.reorder_level` condition stops duplicate requests for products that were already below their level.
- Compound trigger state (`g_requests`) is **re-initialized for every firing statement**, so a BEFORE STATEMENT section that clears it isn't needed.
- Compound triggers are available from Oracle 11g. Before that, the workaround was a package variable plus separate row-level and statement-level triggers.

---

## Answer 5: Year-End Bonus Payout

### Autonomous logging procedure
```sql
CREATE OR REPLACE PROCEDURE log_error (
  p_source IN VARCHAR2,
  p_ref_id IN NUMBER,
  p_code   IN NUMBER,
  p_msg    IN VARCHAR2
) IS
  PRAGMA AUTONOMOUS_TRANSACTION;          -- runs in its own independent transaction
BEGIN
  INSERT INTO error_log (source, ref_id, err_code, err_msg)
  VALUES (p_source, p_ref_id, p_code, SUBSTR(p_msg, 1, 4000));
  COMMIT;                                 -- required: an autonomous unit must commit or roll back
END log_error;
/
```

### Function with a `%ROWTYPE` parameter
```sql
CREATE OR REPLACE FUNCTION calc_bonus (p_emp IN employees%ROWTYPE) RETURN NUMBER IS
  c_max_bonus CONSTANT NUMBER := 50000;
  l_years     PLS_INTEGER;
  l_bonus     NUMBER;
BEGIN
  IF p_emp.hire_date IS NULL OR p_emp.salary IS NULL THEN
    RETURN 0;                             -- without this, NULL years would fall into ELSE (10%)
  END IF;

  l_years := TRUNC(MONTHS_BETWEEN(SYSDATE, p_emp.hire_date) / 12);

  l_bonus := CASE
               WHEN l_years < 2  THEN 0
               WHEN l_years <= 5 THEN p_emp.salary * 0.05
               ELSE                   p_emp.salary * 0.10
             END;

  RETURN LEAST(l_bonus, c_max_bonus);
END calc_bonus;
/
```

### Procedure with cursor FOR UPDATE NOWAIT
```sql
CREATE OR REPLACE PROCEDURE pay_bonus (
  p_dept_id     IN  employees.dept_id%TYPE,
  p_paid_count  OUT PLS_INTEGER,
  p_total_bonus OUT NUMBER
) IS
  CURSOR c_emp IS
    SELECT *                                -- same column order as employees%ROWTYPE
    FROM   employees
    WHERE  dept_id = p_dept_id
      AND  status  = 'ACTIVE'
    FOR UPDATE OF salary NOWAIT;

  l_bonus NUMBER;

  e_row_locked EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_row_locked, -54); -- ORA-00054: resource busy ... NOWAIT
BEGIN
  p_paid_count  := 0;
  p_total_bonus := 0;

  FOR r_emp IN c_emp LOOP                   -- locks are taken when the loop opens the cursor
    l_bonus := calc_bonus(r_emp);           -- the whole row is passed as one record

    IF l_bonus > 0 THEN
      UPDATE employees
      SET    salary = salary + l_bonus
      WHERE  CURRENT OF c_emp;

      p_paid_count  := p_paid_count + 1;
      p_total_bonus := p_total_bonus + l_bonus;
    END IF;
  END LOOP;

  COMMIT;

EXCEPTION
  WHEN e_row_locked THEN
    ROLLBACK;
    p_paid_count  := 0;
    p_total_bonus := 0;
    log_error('PAY_BONUS', p_dept_id, SQLCODE, SQLERRM);   -- survives the rollback

  WHEN OTHERS THEN
    ROLLBACK;
    log_error('PAY_BONUS', p_dept_id, SQLCODE, SQLERRM);
    RAISE;
END pay_bonus;
/
```

### Test block
```sql
DECLARE
  l_cnt   PLS_INTEGER;
  l_total NUMBER;
BEGIN
  pay_bonus(10, l_cnt, l_total);
  DBMS_OUTPUT.PUT_LINE('Paid ' || l_cnt || ' employee(s), total bonus = ' || l_total);
END;
/
-- Fresh data: Ravi (9+ yrs, 10% = 9000), Anu (5 yrs, 5% = 3750), Kiran (probation, 0)
-- Output: Paid 2 employee(s), total bonus = 12750

-- To test the lock path: in SESSION 2 run
--   UPDATE employees SET salary = salary WHERE emp_id = 101;   (don't commit)
-- then run the block again in SESSION 1 and check:
SELECT * FROM error_log WHERE source = 'PAY_BONUS';
```

### Explanation
- **NOWAIT error and timing:** `NOWAIT` raises **ORA-00054 (resource busy and acquire with NOWAIT specified)**. It is raised when the **cursor is opened**, which is the moment the FOR loop starts, because that is when Oracle tries to lock **all** the rows in the result set. So either every row is locked or none are; nothing has been updated yet when the error occurs.
- **`SKIP LOCKED` instead:** `FOR UPDATE SKIP LOCKED` would not raise an error. It silently **skips the rows locked by other sessions** and processes the rest. That suits queue-style processing, but for payroll it would mean some employees silently miss their bonus, so NOWAIT plus logging is the safer choice here. `FOR UPDATE WAIT n` is the middle option: it waits up to n seconds, then raises **ORA-30006**.
- **Why autonomous logging:** The handler calls `ROLLBACK`. A normal INSERT into `ERROR_LOG` would be part of the same transaction and would be wiped out, either by that rollback or by a later one from the caller. The autonomous transaction commits independently, so the log entry always persists.
- **Passing a cursor row to a `%ROWTYPE` parameter:** This works because the cursor selects `*` from `employees`, so the record has the same fields, in the same order, with compatible types.
- **Audit trigger interaction (Scenario 1):** Each bonus UPDATE fires `trg_salary_audit`. **On success**, the audit rows are committed together with the salary changes. **On failure**, the `ROLLBACK` removes any salary changes and their audit rows (with NOWAIT there are none, since the error comes before any update). Only the **autonomous** `ERROR_LOG` row remains.

---

## Answer 6: Department Restructuring

### Probation trigger
```sql
CREATE OR REPLACE TRIGGER trg_emp_transfer_check
BEFORE UPDATE OF dept_id ON employees
FOR EACH ROW
WHEN (NEW.dept_id <> OLD.dept_id)          -- fire only on an actual department change
BEGIN
  IF :OLD.hire_date > ADD_MONTHS(SYSDATE, -6) THEN
    RAISE_APPLICATION_ERROR(-20020,
      'Employee ' || :OLD.emp_id || ' (' || :OLD.emp_name ||
      ') is on probation and cannot be transferred');
  END IF;
END;
/
```

### Package specification
```sql
CREATE OR REPLACE PACKAGE dept_restructure_pkg AS

  TYPE t_emp_tab IS TABLE OF employees%ROWTYPE;   -- public collection type

  FUNCTION get_dept_employees (p_dept_id IN employees.dept_id%TYPE) RETURN t_emp_tab;

  -- Overload 1: single employee
  PROCEDURE transfer (p_emp_id   IN employees.emp_id%TYPE,
                      p_new_dept IN employees.dept_id%TYPE);

  -- Overload 2: bulk transfer by salary threshold
  PROCEDURE transfer (p_from_dept  IN employees.dept_id%TYPE,
                      p_to_dept    IN employees.dept_id%TYPE,
                      p_min_salary IN employees.salary%TYPE);

  FUNCTION get_session_transfers RETURN PLS_INTEGER;

END dept_restructure_pkg;
/
```

### Package body
```sql
CREATE OR REPLACE PACKAGE BODY dept_restructure_pkg AS

  -- PRIVATE package variable: keeps its value for the whole session
  g_transfer_count PLS_INTEGER;

  -- PRIVATE helper procedure
  PROCEDURE validate_dept (p_dept_id IN departments.dept_id%TYPE) IS
    l_dummy PLS_INTEGER;
  BEGIN
    SELECT 1 INTO l_dummy FROM departments WHERE dept_id = p_dept_id;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      RAISE_APPLICATION_ERROR(-20021, 'Department ' || p_dept_id || ' does not exist');
  END validate_dept;


  FUNCTION get_dept_employees (p_dept_id IN employees.dept_id%TYPE) RETURN t_emp_tab IS
    l_emps t_emp_tab;
  BEGIN
    SELECT *
    BULK COLLECT INTO l_emps
    FROM   employees
    WHERE  dept_id = p_dept_id
      AND  status  = 'ACTIVE'
    ORDER  BY salary DESC;

    RETURN l_emps;      -- BULK COLLECT never raises NO_DATA_FOUND; it returns an empty collection
  END get_dept_employees;


  -- Overload 1
  PROCEDURE transfer (p_emp_id   IN employees.emp_id%TYPE,
                      p_new_dept IN employees.dept_id%TYPE) IS
  BEGIN
    validate_dept(p_new_dept);

    UPDATE employees
    SET    dept_id = p_new_dept
    WHERE  emp_id  = p_emp_id;            -- the probation trigger fires here

    IF SQL%ROWCOUNT = 0 THEN
      RAISE_APPLICATION_ERROR(-20022, 'Employee ' || p_emp_id || ' not found');
    END IF;

    g_transfer_count := g_transfer_count + 1;
    COMMIT;
  END transfer;


  -- Overload 2
  PROCEDURE transfer (p_from_dept  IN employees.dept_id%TYPE,
                      p_to_dept    IN employees.dept_id%TYPE,
                      p_min_salary IN employees.salary%TYPE) IS

    TYPE t_id_tab   IS TABLE OF employees.emp_id%TYPE;
    TYPE t_name_tab IS TABLE OF employees.emp_name%TYPE;

    l_emps   t_emp_tab;
    l_ids    t_id_tab := t_id_tab();
    l_names  t_name_tab;
  BEGIN
    SAVEPOINT before_bulk_transfer;       -- first statement, so it always exists for the handler

    validate_dept(p_to_dept);

    l_emps := get_dept_employees(p_from_dept);

    FOR i IN 1 .. l_emps.COUNT LOOP
      IF l_emps(i).salary >= p_min_salary THEN
        l_ids.EXTEND;
        l_ids(l_ids.LAST) := l_emps(i).emp_id;
      END IF;
    END LOOP;

    IF l_ids.COUNT = 0 THEN
      DBMS_OUTPUT.PUT_LINE('No employees matched the criteria');
      RETURN;
    END IF;

    FORALL i IN 1 .. l_ids.COUNT
      UPDATE employees
      SET    dept_id = p_to_dept
      WHERE  emp_id  = l_ids(i)
      RETURNING emp_name BULK COLLECT INTO l_names;

    g_transfer_count := g_transfer_count + SQL%ROWCOUNT;   -- only after success
    COMMIT;

    FOR i IN 1 .. l_names.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE('Moved: ' || l_names(i));
    END LOOP;

  EXCEPTION
    WHEN OTHERS THEN
      ROLLBACK TO before_bulk_transfer;   -- all-or-nothing
      RAISE;
  END transfer;


  FUNCTION get_session_transfers RETURN PLS_INTEGER IS
  BEGIN
    RETURN g_transfer_count;
  END get_session_transfers;

-- INITIALIZATION SECTION: runs once per session, the first time the package is referenced
BEGIN
  g_transfer_count := 0;
END dept_restructure_pkg;
/
```

### Test block
```sql
-- 1) FAILS: Ravi (90000), Anu (75000), Kiran (60000, on probation) all qualify
BEGIN
  dept_restructure_pkg.transfer(10, 20, 50000);
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/
-- Error: ORA-20020: Employee 103 (Kiran) is on probation ...
SELECT emp_id, dept_id FROM employees WHERE emp_id IN (101, 102, 103);  -- all still in 10

-- 2) SUCCEEDS: only Ravi and Anu qualify
EXEC dept_restructure_pkg.transfer(10, 20, 70000);
-- Moved: Ravi
-- Moved: Anu

-- 3) Single-employee overload
EXEC dept_restructure_pkg.transfer(104, 30);

-- 4) Session counter
EXEC DBMS_OUTPUT.PUT_LINE('Transfers this session: ' || dept_restructure_pkg.get_session_transfers);
-- Transfers this session: 3

-- 5) Using the public function and type from outside the package
DECLARE
  l_emps dept_restructure_pkg.t_emp_tab;
BEGIN
  l_emps := dept_restructure_pkg.get_dept_employees(20);
  FOR i IN 1 .. l_emps.COUNT LOOP
    DBMS_OUTPUT.PUT_LINE(l_emps(i).emp_name || ' - ' || l_emps(i).salary);
  END LOOP;
END;
/
```

### Explanation
- **FORALL without `SAVE EXCEPTIONS` when one iteration fails:** FORALL sends one DML statement per element, in order. The employees are sorted by salary descending, so Ravi (iteration 1) and Anu (iteration 2) are updated successfully. At Kiran (iteration 3) the trigger raises -20020. Then:
  1. Only the **failing iteration's** statement is rolled back (statement-level rollback).
  2. **Iterations 1 and 2 are NOT rolled back.** Their changes stay pending in the transaction.
  3. FORALL **stops immediately**; any later iterations never run.
  4. The exception propagates to the caller.

  This is why the procedure uses a **savepoint plus `ROLLBACK TO` in the handler**: without it, Ravi and Anu would stay moved, which breaks the all-or-nothing rule. With `SAVE EXCEPTIONS`, FORALL would instead run every iteration and raise ORA-24381 at the end, which is the right choice when partial success is acceptable (as in Scenario 2).
- **Why the counter must be incremented after FORALL:** Package variables are **memory, not database data, so ROLLBACK never resets them**. If `g_transfer_count` were increased before the FORALL and the transfer then failed, `ROLLBACK TO before_bulk_transfer` would undo the table changes but the counter would keep the inflated value, reporting transfers that never happened. Using `SQL%ROWCOUNT` after a successful FORALL also counts the **rows actually updated**.
- **Overloading:** Both versions are named `TRANSFER`, and Oracle picks the right one by the **number and types of the arguments** (2 vs 3 parameters). Overloading is possible only for procedures and functions inside packages (or nested inside other blocks), not for standalone ones.
- **Private vs public:** `g_transfer_count` and `validate_dept` exist only in the body, so callers cannot read or change the counter directly. They can only read it through `get_session_transfers`, which is encapsulation.
- **Initialization section:** The `BEGIN ... END` at the end of the body runs **once per session**, the first time any element of the package is referenced. Each session has its own copy of the package state.
- **`RETURNING ... BULK COLLECT INTO`** with FORALL collects the returned value from **every** iteration into one collection, so there's no need for a second SELECT to find out who was moved.
- **Probation enforced in a trigger** (not just in the package) means the rule also applies to ad-hoc UPDATEs and other programs. It can't be bypassed.

---