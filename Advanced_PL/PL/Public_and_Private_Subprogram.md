# Private vs. Public Subprograms in PL/SQL — Simple Guide

## The Idea

A package has two parts:

- **Spec** = the **front counter** of a shop. Whatever is listed here, outsiders can use.
- **Body** = the **kitchen**. The actual work happens here, and outsiders can't walk in.

So:

- **Public** = declared in the **spec**. Anyone (with permission) can call it from outside.
- **Private** = written **only in the body**, not listed in the spec. Only other code inside the same package can call it.

> There is **no `PUBLIC` or `PRIVATE` keyword**. Being in the spec or not is the only thing that decides it.

### Analogy: a restaurant

You order **"Butter Chicken"** from the menu (public). Inside the kitchen, the chef uses helper steps like "make gravy" and "marinate chicken" (private).

- You can **order** Butter Chicken. ✅
- You **can't** ask for "just the gravy step." ❌ That's not on the menu.
- The chef can change how the gravy is made tomorrow, and you won't notice.

### Public vs private at a glance

|                                   | Public                                   | Private                         |
|-----------------------------------|------------------------------------------|---------------------------------|
| Declared in                       | Package **spec** (implemented in body)   | **Only** the package body       |
| Callable from outside?            | ✅ `package_name.subprogram(...)`         | ❌ Error                         |
| Callable from inside the body?    | ✅                                        | ✅                               |
| Does body order matter?           | ❌ No (the spec already declares it)      | ✅ Yes                           |
| Purpose                           | The package's **API** for consumers      | **Internal helper** logic       |

---

## Example 1: Public and Private Functions

```sql
-- SPEC: the menu (public)
CREATE OR REPLACE PACKAGE payroll_pkg IS
  FUNCTION calculate_net_pay (p_emp_id NUMBER) RETURN NUMBER;
END payroll_pkg;
/

-- BODY: the kitchen
CREATE OR REPLACE PACKAGE BODY payroll_pkg IS

  -- PRIVATE: not in the spec, so only usable inside this body
  FUNCTION calculate_tax (p_gross NUMBER) RETURN NUMBER IS
  BEGIN
    RETURN p_gross * 0.20;
  END;

  -- PUBLIC: listed in the spec
  FUNCTION calculate_net_pay (p_emp_id NUMBER) RETURN NUMBER IS
    v_gross NUMBER;
  BEGIN
    SELECT salary INTO v_gross FROM employees WHERE employee_id = p_emp_id;
    RETURN v_gross - calculate_tax(v_gross);   -- calling the private helper ✅
  END;

END payroll_pkg;
/
```

From outside:

```sql
SELECT payroll_pkg.calculate_net_pay(100) FROM dual;
-- ✅ Works, it's public

SELECT payroll_pkg.calculate_tax(5000) FROM dual;
-- ❌ Error: calculate_tax isn't visible from outside
```

> The exact error depends on where you call it from:
> - From a PL/SQL block: `PLS-00302: component 'CALCULATE_TAX' must be declared`
> - From a plain SQL `SELECT`: `ORA-00904: invalid identifier`

---

## Why Have Private Subprograms?

1. **Hide the messy details (encapsulation).** Outsiders only need "give me net pay," not how tax is calculated.
2. **Change freely without breaking anyone.** If tax becomes slab-based instead of a flat 20%, you rewrite `calculate_tax` and nobody outside notices, because nobody outside ever called it.
3. **Security.** Something that can't be called from outside can't be misused. Fewer public doors means fewer ways in. (Even an `EXECUTE` grant on the package can't expose a private subprogram.)

---

## Example 2: The Ordering Rule and Forward Declarations

Oracle reads the package body **top to bottom**. A **private** subprogram must appear **before** the code that uses it.

### ❌ This fails

```sql
CREATE OR REPLACE PACKAGE BODY payroll_pkg IS

  FUNCTION calculate_net_pay (p_emp_id NUMBER) RETURN NUMBER IS
    v_gross NUMBER;
  BEGIN
    SELECT salary INTO v_gross FROM employees WHERE employee_id = p_emp_id;
    RETURN v_gross - calculate_tax(v_gross);   -- ❌ Oracle hasn't seen calculate_tax yet
  END;

  FUNCTION calculate_tax (p_gross NUMBER) RETURN NUMBER IS   -- defined too late
  BEGIN
    RETURN p_gross * 0.20;
  END;

END payroll_pkg;
/
-- PLS-00313: 'CALCULATE_TAX' not declared in this scope
```

### ✅ Fix 1: Move the private function above its first use (as in Example 1)

### ✅ Fix 2: Add a forward declaration

Write only the function's **header line, ending with a semicolon**, at the top. It tells Oracle: "this exists, the full body comes later."

```sql
CREATE OR REPLACE PACKAGE BODY payroll_pkg IS

  FUNCTION calculate_tax (p_gross NUMBER) RETURN NUMBER;   -- forward declaration

  FUNCTION calculate_net_pay (p_emp_id NUMBER) RETURN NUMBER IS
    v_gross NUMBER;
  BEGIN
    SELECT salary INTO v_gross FROM employees WHERE employee_id = p_emp_id;
    RETURN v_gross - calculate_tax(v_gross);   -- ✅ Oracle now knows it exists
  END;

  FUNCTION calculate_tax (p_gross NUMBER) RETURN NUMBER IS   -- full body, later
  BEGIN
    RETURN p_gross * 0.20;
  END;

END payroll_pkg;
/
```

**Why don't public functions need this?** They are already declared in the **spec**, so Oracle knows about them before it reads the body. The spec acts as their forward declaration.

---

## Example 3: Private Variables (Getters and Setters)

The same public/private rule applies to variables, cursors, types, and constants.

### ❌ Bad: public variable

```sql
CREATE OR REPLACE PACKAGE bank_pkg IS
  g_balance NUMBER := 1000;   -- PUBLIC variable
END bank_pkg;
/

BEGIN
  bank_pkg.g_balance := 99999999;   -- anyone can change it, no checks!
END;
/
```

### ✅ Good: private variable + controlled public functions

```sql
CREATE OR REPLACE PACKAGE bank_pkg IS
  FUNCTION  get_balance RETURN NUMBER;
  PROCEDURE deposit(p_amount NUMBER);
END bank_pkg;
/

CREATE OR REPLACE PACKAGE BODY bank_pkg IS
  g_balance NUMBER := 1000;   -- PRIVATE now

  FUNCTION get_balance RETURN NUMBER IS
  BEGIN
    RETURN g_balance;
  END;

  PROCEDURE deposit(p_amount NUMBER) IS
  BEGIN
    IF p_amount <= 0 THEN
      RAISE_APPLICATION_ERROR(-20001, 'Deposit must be positive');
    END IF;
    g_balance := g_balance + p_amount;
  END;
END bank_pkg;
/
```

```sql
EXEC bank_pkg.deposit(500);                        -- ✅ allowed, with checks
EXEC DBMS_OUTPUT.put_line(bank_pkg.get_balance);   -- 1500
EXEC bank_pkg.deposit(-100);                       -- ❌ Deposit must be positive
BEGIN bank_pkg.g_balance := 99999999; END;         -- ❌ not visible from outside
```

This is the same idea as **getters and setters** in Java: keep the data private and expose only methods.

---

## Quick Summary

> **In the spec = public.** Anyone can call it from outside.
> **Only in the body = private.** Only the package's own code can call it.
> Private items hide details, let you change them safely, and reduce misuse.
> A private subprogram must appear **before** it's used in the body, or you add a **forward declaration**.
> Keep variables **private** and expose them through functions (getters/setters).

---

## Commonly Asked Interview Questions

### Q1. What is the difference between a public and a private subprogram in a package?
**Answer:** A public subprogram is declared in the **package specification** and can be called from outside the package. A private subprogram is defined **only in the package body** and can be called only by other code inside that body.

### Q2. Is there a `PRIVATE` keyword in PL/SQL?
**Answer:** **No.** Anything in the body that isn't declared in the spec is automatically private.

### Q3. What error do you get when calling a private subprogram from outside?
**Answer:** From PL/SQL: `PLS-00302: component '...' must be declared`. From a SQL statement: `ORA-00904: invalid identifier`.

### Q4. Can a private subprogram call a public one, and vice versa?
**Answer:** **Yes,** both ways, as long as they're in the same package body (and ordering rules are followed for private ones).

### Q5. What is a forward declaration?
**Answer:** A subprogram's header followed by a semicolon (no body), placed earlier in the package body. It lets code above the actual body call that subprogram.

### Q6. Why do private subprograms need forward declarations but public ones don't?
**Answer:** PL/SQL resolves names top to bottom. Public subprograms are already declared in the spec, so they're known throughout the body. Private ones are known only from the point where they're declared in the body.

### Q7. What error do you get if you call a private function before it is declared?
**Answer:** A compile error like `PLS-00313: '...' not declared in this scope`.

### Q8. What are the benefits of private subprograms?
**Answer:** Encapsulation (hiding implementation details), safe refactoring (changing internals without affecting callers), and a smaller security surface (they can't be called from outside).

### Q9. Can you grant EXECUTE on only one private procedure of a package?
**Answer:** **No.** Grants are on the whole package, and private subprograms have no external entry point at all.

### Q10. Why is a public package variable considered bad practice?
**Answer:** Anyone can change it directly with no validation. It's better to keep it private and expose public getter/setter subprograms that enforce rules.

### Q11. Do public/private rules apply only to subprograms?
**Answer:** **No.** Variables, constants, cursors, types, and exceptions follow the same rule: in the spec = public, only in the body = private.

### Q12. If you change the code inside a private function, do callers of the package need recompiling?
**Answer:** **No.** Changing only the package body doesn't invalidate the spec, so dependent objects stay valid. Changing the **spec** can invalidate dependents.

### Q13. Can a private subprogram and a public subprogram have the same name?
**Answer:** They can share a name only if they are valid **overloads** (different parameter lists). Otherwise it's a duplicate-declaration error.

---

# Practice Coding Questions

> Try each question yourself first. **Answers are at the end of this file.**
> Run with `SET SERVEROUTPUT ON`.

---

## Practice 1: Student Grade Package (basics)

Create a package called **`grade_pkg`**.

**Public (put in the spec):**
- `FUNCTION get_grade(p_marks NUMBER) RETURN VARCHAR2`

**Private (only in the body):**
- `PROCEDURE validate_marks(p_marks NUMBER)`: if marks are `NULL`, below 0, or above 100, raise error `-20010` with the message `'Marks must be between 0 and 100'`.

**Rules for `get_grade`:**
1. First call `validate_marks`.
2. Return `'A'` for 90+, `'B'` for 75–89, `'C'` for 50–74, and `'F'` below 50.

**Order rule:** define the private procedure **above** `get_grade` (no forward declaration needed).

**Test with:**
```sql
EXEC DBMS_OUTPUT.put_line(grade_pkg.get_grade(95));
EXEC DBMS_OUTPUT.put_line(grade_pkg.get_grade(60));
EXEC DBMS_OUTPUT.put_line(grade_pkg.get_grade(150));
EXEC grade_pkg.validate_marks(50);
```

**Expected:**
```
A
C
ORA-20010: Marks must be between 0 and 100
PLS-00302: component 'VALIDATE_MARKS' must be declared
```

---

## Practice 2: Fix It With Forward Declarations

The package below **does not compile**. Fix it.

```sql
CREATE OR REPLACE PACKAGE msg_pkg IS
  PROCEDURE print_salary(p_name VARCHAR2, p_amount NUMBER);
END msg_pkg;
/

CREATE OR REPLACE PACKAGE BODY msg_pkg IS

  PROCEDURE print_salary(p_name VARCHAR2, p_amount NUMBER) IS
  BEGIN
    DBMS_OUTPUT.put_line(build_message(p_name, p_amount));
  END;

  FUNCTION build_message(p_name VARCHAR2, p_amount NUMBER) RETURN VARCHAR2 IS
  BEGIN
    RETURN p_name || ' earns ' || format_amount(p_amount);
  END;

  FUNCTION format_amount(p_amount NUMBER) RETURN VARCHAR2 IS
  BEGIN
    RETURN 'Rs. ' || TO_CHAR(p_amount, 'FM999,999,999');
  END;

END msg_pkg;
/
```

**Your tasks:**
1. Identify which subprograms are **public** and which are **private**.
2. Explain **why** it doesn't compile.
3. Fix it **without moving any existing function**. Only add lines.

**Test with:**
```sql
EXEC msg_pkg.print_salary('Amit', 50000);
```

**Expected:**
```
Amit earns Rs. 50,000
```

---

## Practice 3: Shopping Cart (private state + private helpers)

Create a package called **`cart_pkg`**.

**Public (spec):**
- `PROCEDURE add_item(p_price NUMBER, p_qty NUMBER)`
- `FUNCTION get_total RETURN NUMBER`: returns the total **after discount**
- `FUNCTION get_item_count RETURN NUMBER`: total quantity of items added
- `PROCEDURE clear_cart`: resets total and count to 0

**Private (body only):**
- Variable `g_total NUMBER := 0`
- Variable `g_items NUMBER := 0`
- `PROCEDURE validate_item(p_price NUMBER, p_qty NUMBER)`: if price ≤ 0 or qty ≤ 0, raise `-20020` with `'Price and quantity must be positive'`
- `FUNCTION apply_discount(p_amount NUMBER) RETURN NUMBER`: if the amount is **more than 5000**, return 90% of it (10% off); otherwise return it unchanged

**Rules:**
- `add_item` must call `validate_item`, then add `price × qty` to `g_total` and `qty` to `g_items`.
- `get_total` must return `apply_discount(g_total)`.

**Test with:**
```sql
EXEC cart_pkg.clear_cart;
EXEC cart_pkg.add_item(2000, 2);
EXEC DBMS_OUTPUT.put_line('Total: ' || cart_pkg.get_total);
EXEC cart_pkg.add_item(1500, 1);
EXEC DBMS_OUTPUT.put_line('Total: ' || cart_pkg.get_total);
EXEC DBMS_OUTPUT.put_line('Items: ' || cart_pkg.get_item_count);
EXEC cart_pkg.add_item(-5, 1);
BEGIN cart_pkg.g_total := 0; END;
/
```

**Expected:**
```
Total: 4000
Total: 4950
Items: 3
ORA-20020: Price and quantity must be positive
PLS-00302: component 'G_TOTAL' must be declared
```

---

## Practice 4: Spot the Problems and Rewrite

A teammate wrote this loan package. It has **design and code problems**.

```sql
CREATE OR REPLACE PACKAGE loan_pkg IS
  g_interest_rate NUMBER := 10;
  FUNCTION total_payable(p_principal NUMBER, p_years NUMBER) RETURN NUMBER;
END loan_pkg;
/

CREATE OR REPLACE PACKAGE BODY loan_pkg IS

  FUNCTION total_payable(p_principal NUMBER, p_years NUMBER) RETURN NUMBER IS
  BEGIN
    RETURN p_principal + calc_interest(p_principal, p_years);
  END;

  FUNCTION calc_interest(p_principal NUMBER, p_years NUMBER) RETURN NUMBER IS
  BEGIN
    RETURN p_principal * g_interest_rate / 100 * p_years;
  END;

END loan_pkg;
/

-- A caller elsewhere:
SELECT loan_pkg.calc_interest(100000, 2) FROM dual;
```

**Your tasks:**
1. List **all three problems** (one compile problem, one design problem, one caller problem).
2. Rewrite the package so that:
   - `g_interest_rate` is **private**.
   - There is a public `FUNCTION get_rate RETURN NUMBER`.
   - There is a public `PROCEDURE set_rate(p_rate NUMBER)` that accepts only rates **1 to 20**, otherwise raises `-20040` with `'Rate must be between 1 and 20'`.
   - `calc_interest` stays **private**.
   - `total_payable` stays the **first** function in the body (so use a forward declaration).
3. Rewrite the caller's query so it works using only public members.

**Test with:**
```sql
EXEC DBMS_OUTPUT.put_line(loan_pkg.total_payable(100000, 2));
EXEC loan_pkg.set_rate(12);
EXEC DBMS_OUTPUT.put_line(loan_pkg.get_rate);
EXEC DBMS_OUTPUT.put_line(loan_pkg.total_payable(100000, 2));
EXEC loan_pkg.set_rate(50);
```

**Expected:**
```
120000
12
124000
ORA-20040: Rate must be between 1 and 20
```

---

## Practice 5: Employee Bonus Package (everything together)

### Setup

```sql
CREATE TABLE emp_bonus_demo (
  emp_id   NUMBER,
  emp_name VARCHAR2(30),
  salary   NUMBER,
  dept     VARCHAR2(10)
);
INSERT INTO emp_bonus_demo VALUES (101, 'Amit', 60000, 'IT');
INSERT INTO emp_bonus_demo VALUES (102, 'Neha', 50000, 'HR');
INSERT INTO emp_bonus_demo VALUES (103, 'Ravi', 40000, 'SALES');
COMMIT;
```

Create a package called **`bonus_pkg`**.

**Public (spec):**
- `PROCEDURE show_bonus(p_emp_id NUMBER)`: prints `'<name> gets bonus: <amount>'`
- `FUNCTION total_bonus_shown RETURN NUMBER`: sum of all bonuses shown so far in this session

**Private (body only):**
- Variable `g_total_bonus NUMBER := 0`
- `PROCEDURE get_emp_details(p_emp_id IN NUMBER, p_name OUT VARCHAR2, p_salary OUT NUMBER, p_dept OUT VARCHAR2)`: fetches from the table. If the employee doesn't exist, raise `-20030` with `'Employee <id> not found'`.
- `FUNCTION calc_bonus(p_salary NUMBER, p_dept VARCHAR2) RETURN NUMBER`: IT gets **15%**, HR gets **10%**, everyone else gets **5%**.

**Rules:**
- `show_bonus` must be written **first** in the body, before the private subprograms, so you must use **forward declarations**.
- `show_bonus` adds each bonus to `g_total_bonus`.

**Test with:**
```sql
EXEC bonus_pkg.show_bonus(101);
EXEC bonus_pkg.show_bonus(102);
EXEC bonus_pkg.show_bonus(103);
EXEC DBMS_OUTPUT.put_line('Total shown: ' || bonus_pkg.total_bonus_shown);
EXEC bonus_pkg.show_bonus(999);
EXEC DBMS_OUTPUT.put_line(bonus_pkg.calc_bonus(10000, 'IT'));
```

**Expected:**
```
Amit gets bonus: 9000
Neha gets bonus: 5000
Ravi gets bonus: 2000
Total shown: 16000
ORA-20030: Employee 999 not found
PLS-00302: component 'CALC_BONUS' must be declared
```

---
---

# Answers

> Check your solution against these. Your code doesn't have to match line for line. It just needs to follow the public/private rules and produce the expected output.

---

## Answer 1: Student Grade Package

```sql
CREATE OR REPLACE PACKAGE grade_pkg IS
  FUNCTION get_grade(p_marks NUMBER) RETURN VARCHAR2;   -- PUBLIC
END grade_pkg;
/

CREATE OR REPLACE PACKAGE BODY grade_pkg IS

  -- PRIVATE: not in spec, defined before its use
  PROCEDURE validate_marks(p_marks NUMBER) IS
  BEGIN
    IF p_marks IS NULL OR p_marks < 0 OR p_marks > 100 THEN
      RAISE_APPLICATION_ERROR(-20010, 'Marks must be between 0 and 100');
    END IF;
  END;

  -- PUBLIC implementation
  FUNCTION get_grade(p_marks NUMBER) RETURN VARCHAR2 IS
  BEGIN
    validate_marks(p_marks);

    IF p_marks >= 90 THEN
      RETURN 'A';
    ELSIF p_marks >= 75 THEN
      RETURN 'B';
    ELSIF p_marks >= 50 THEN
      RETURN 'C';
    ELSE
      RETURN 'F';
    END IF;
  END;

END grade_pkg;
/
```

**Key point:** `validate_marks` is private because it's not in the spec, so calling it from outside gives `PLS-00302`.

---

## Answer 2: Fix With Forward Declarations

**1. Public vs private:**
- Public: `print_salary` (in the spec)
- Private: `build_message`, `format_amount` (only in the body)

**2. Why it doesn't compile:**
`print_salary` calls `build_message` before `build_message` is declared, and `build_message` calls `format_amount` before `format_amount` is declared. Private subprograms are only known from the point where they're declared, so you get `PLS-00313: not declared in this scope`.

**3. Fixed code (only two lines added at the top):**

```sql
CREATE OR REPLACE PACKAGE BODY msg_pkg IS

  -- Forward declarations
  FUNCTION build_message(p_name VARCHAR2, p_amount NUMBER) RETURN VARCHAR2;
  FUNCTION format_amount(p_amount NUMBER) RETURN VARCHAR2;

  PROCEDURE print_salary(p_name VARCHAR2, p_amount NUMBER) IS
  BEGIN
    DBMS_OUTPUT.put_line(build_message(p_name, p_amount));
  END;

  FUNCTION build_message(p_name VARCHAR2, p_amount NUMBER) RETURN VARCHAR2 IS
  BEGIN
    RETURN p_name || ' earns ' || format_amount(p_amount);
  END;

  FUNCTION format_amount(p_amount NUMBER) RETURN VARCHAR2 IS
  BEGIN
    RETURN 'Rs. ' || TO_CHAR(p_amount, 'FM999,999,999');
  END;

END msg_pkg;
/
```

**Key point:** a forward declaration is just the header plus `;`. The parameter names and types must match the real body exactly.

---

## Answer 3: Shopping Cart

```sql
CREATE OR REPLACE PACKAGE cart_pkg IS
  PROCEDURE add_item(p_price NUMBER, p_qty NUMBER);
  FUNCTION  get_total RETURN NUMBER;
  FUNCTION  get_item_count RETURN NUMBER;
  PROCEDURE clear_cart;
END cart_pkg;
/

CREATE OR REPLACE PACKAGE BODY cart_pkg IS

  -- PRIVATE state
  g_total NUMBER := 0;
  g_items NUMBER := 0;

  -- PRIVATE helpers (defined before use)
  PROCEDURE validate_item(p_price NUMBER, p_qty NUMBER) IS
  BEGIN
    IF p_price <= 0 OR p_qty <= 0 THEN
      RAISE_APPLICATION_ERROR(-20020, 'Price and quantity must be positive');
    END IF;
  END;

  FUNCTION apply_discount(p_amount NUMBER) RETURN NUMBER IS
  BEGIN
    IF p_amount > 5000 THEN
      RETURN p_amount * 0.9;
    END IF;
    RETURN p_amount;
  END;

  -- PUBLIC implementations
  PROCEDURE add_item(p_price NUMBER, p_qty NUMBER) IS
  BEGIN
    validate_item(p_price, p_qty);
    g_total := g_total + (p_price * p_qty);
    g_items := g_items + p_qty;
  END;

  FUNCTION get_total RETURN NUMBER IS
  BEGIN
    RETURN apply_discount(g_total);
  END;

  FUNCTION get_item_count RETURN NUMBER IS
  BEGIN
    RETURN g_items;
  END;

  PROCEDURE clear_cart IS
  BEGIN
    g_total := 0;
    g_items := 0;
  END;

END cart_pkg;
/
```

**How the expected output comes:**
- `add_item(2000, 2)` → total 4000 → not above 5000 → **4000**
- `add_item(1500, 1)` → total 5500 → above 5000 → 5500 × 0.9 = **4950**
- Items = 2 + 1 = **3**

**Key point:** `g_total` and `g_items` are private, so outsiders can change them only through `add_item` and `clear_cart`, with validation.

---

## Answer 4: Spot the Problems and Rewrite

**1. The three problems:**

| # | Type | Problem |
|---|------|---------|
| 1 | Compile | `total_payable` calls private `calc_interest` before it's declared → `PLS-00313`. Needs a forward declaration (or reordering). |
| 2 | Design | `g_interest_rate` is in the **spec**, so it's public. Anyone can set it to any value (even negative) with no checks. |
| 3 | Caller | `calc_interest` is **private** (not in the spec), so `loan_pkg.calc_interest(...)` fails from outside (`ORA-00904` in SQL). |

**2. Rewritten package:**

```sql
CREATE OR REPLACE PACKAGE loan_pkg IS
  FUNCTION  total_payable(p_principal NUMBER, p_years NUMBER) RETURN NUMBER;
  FUNCTION  get_rate RETURN NUMBER;
  PROCEDURE set_rate(p_rate NUMBER);
END loan_pkg;
/

CREATE OR REPLACE PACKAGE BODY loan_pkg IS

  -- PRIVATE state
  g_interest_rate NUMBER := 10;

  -- Forward declaration of PRIVATE function
  FUNCTION calc_interest(p_principal NUMBER, p_years NUMBER) RETURN NUMBER;

  -- PUBLIC (kept first, as required)
  FUNCTION total_payable(p_principal NUMBER, p_years NUMBER) RETURN NUMBER IS
  BEGIN
    RETURN p_principal + calc_interest(p_principal, p_years);
  END;

  -- PRIVATE body
  FUNCTION calc_interest(p_principal NUMBER, p_years NUMBER) RETURN NUMBER IS
  BEGIN
    RETURN p_principal * g_interest_rate / 100 * p_years;
  END;

  -- PUBLIC getter
  FUNCTION get_rate RETURN NUMBER IS
  BEGIN
    RETURN g_interest_rate;
  END;

  -- PUBLIC setter with validation
  PROCEDURE set_rate(p_rate NUMBER) IS
  BEGIN
    IF p_rate IS NULL OR p_rate < 1 OR p_rate > 20 THEN
      RAISE_APPLICATION_ERROR(-20040, 'Rate must be between 1 and 20');
    END IF;
    g_interest_rate := p_rate;
  END;

END loan_pkg;
/
```

**3. Fixed caller (interest = total payable − principal):**

```sql
SELECT loan_pkg.total_payable(100000, 2) - 100000 AS interest FROM dual;
-- 20000
```

**How the expected output comes:**
- Rate 10%: 100000 × 10/100 × 2 = 20000 → total **120000**
- `set_rate(12)` → `get_rate` = **12**
- Rate 12%: 100000 × 12/100 × 2 = 24000 → total **124000**
- `set_rate(50)` → above 20 → **ORA-20040**

---

## Answer 5: Employee Bonus Package

```sql
CREATE OR REPLACE PACKAGE bonus_pkg IS
  PROCEDURE show_bonus(p_emp_id NUMBER);
  FUNCTION  total_bonus_shown RETURN NUMBER;
END bonus_pkg;
/

CREATE OR REPLACE PACKAGE BODY bonus_pkg IS

  -- PRIVATE state
  g_total_bonus NUMBER := 0;

  -- Forward declarations of PRIVATE subprograms
  PROCEDURE get_emp_details(p_emp_id IN  NUMBER,
                            p_name   OUT VARCHAR2,
                            p_salary OUT NUMBER,
                            p_dept   OUT VARCHAR2);
  FUNCTION calc_bonus(p_salary NUMBER, p_dept VARCHAR2) RETURN NUMBER;

  -- PUBLIC (written first, as required)
  PROCEDURE show_bonus(p_emp_id NUMBER) IS
    v_name   emp_bonus_demo.emp_name%TYPE;
    v_salary emp_bonus_demo.salary%TYPE;
    v_dept   emp_bonus_demo.dept%TYPE;
    v_bonus  NUMBER;
  BEGIN
    get_emp_details(p_emp_id, v_name, v_salary, v_dept);
    v_bonus := calc_bonus(v_salary, v_dept);
    g_total_bonus := g_total_bonus + v_bonus;
    DBMS_OUTPUT.put_line(v_name || ' gets bonus: ' || v_bonus);
  END;

  FUNCTION total_bonus_shown RETURN NUMBER IS
  BEGIN
    RETURN g_total_bonus;
  END;

  -- PRIVATE bodies
  PROCEDURE get_emp_details(p_emp_id IN  NUMBER,
                            p_name   OUT VARCHAR2,
                            p_salary OUT NUMBER,
                            p_dept   OUT VARCHAR2) IS
  BEGIN
    SELECT emp_name, salary, dept
    INTO   p_name, p_salary, p_dept
    FROM   emp_bonus_demo
    WHERE  emp_id = p_emp_id;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      RAISE_APPLICATION_ERROR(-20030, 'Employee ' || p_emp_id || ' not found');
  END;

  FUNCTION calc_bonus(p_salary NUMBER, p_dept VARCHAR2) RETURN NUMBER IS
  BEGIN
    IF p_dept = 'IT' THEN
      RETURN p_salary * 0.15;
    ELSIF p_dept = 'HR' THEN
      RETURN p_salary * 0.10;
    ELSE
      RETURN p_salary * 0.05;
    END IF;
  END;

END bonus_pkg;
/
```

**How the expected output comes:**
- Amit (IT): 60000 × 15% = **9000**
- Neha (HR): 50000 × 10% = **5000**
- Ravi (SALES): 40000 × 5% = **2000**
- Total = 9000 + 5000 + 2000 = **16000** (kept in private package state for this session)
- Employee 999 → `NO_DATA_FOUND` → converted to **ORA-20030**
- `calc_bonus` is private → **PLS-00302** from outside

**Key points combined here:**
- Public API: `show_bonus`, `total_bonus_shown`
- Private helpers: `get_emp_details`, `calc_bonus`
- Private state: `g_total_bonus` (ties back to **package state**)
- Forward declarations, because the public procedure is written before the private ones it calls