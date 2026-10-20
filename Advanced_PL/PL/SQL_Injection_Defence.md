# SQL Injection Protection Patterns — A Scenario Walkthrough

> **Goal:** Understand defense in depth by attacking one vulnerable procedure three different ways, then hardening it **one layer at a time** and watching each attack fail.

---

## 1. The Scenario

**TechNova** runs an internal HR portal. Employees log in on the web and can edit **their own first name and phone number**.

The database setup:

| Schema | What it owns / does |
|---|---|
| `HR` | Owns the tables: `EMPLOYEES`, `APP_USERS` |
| `HR_APP` | Owns the PL/SQL code the portal calls |
| `PORTAL_USER` | The account the web app connects as. It has **only** `EXECUTE` on `HR_APP`'s API |

```sql
-- Tables (owned by HR)
CREATE TABLE hr.employees (
  employee_id  NUMBER PRIMARY KEY,
  first_name   VARCHAR2(50),
  phone_number VARCHAR2(20),
  salary       NUMBER(10,2)
);

CREATE TABLE hr.app_users (
  employee_id NUMBER PRIMARY KEY, -- links the login to the employee
  username  VARCHAR2(30),
  role_name VARCHAR2(20)          -- 'EMPLOYEE' or 'ADMIN'
);
```

A developer, wanting "one generic procedure for every edit screen", writes this:

```sql
-- VERSION 0 — VULNERABLE
CREATE OR REPLACE PROCEDURE hr_app.update_value (
  p_table IN VARCHAR2,
  p_col   IN VARCHAR2,
  p_id    IN NUMBER,
  p_val   IN VARCHAR2
) IS
BEGIN
  EXECUTE IMMEDIATE
    'UPDATE ' || p_table || ' SET ' || p_col ||
    ' = ''' || p_val || ''' WHERE employee_id = ' || p_id;
END;
/
```

The DBA, in a hurry, gave the owner broad power:

```sql
GRANT UPDATE ANY TABLE TO hr_app;      -- lazy and dangerous
GRANT EXECUTE ON hr_app.update_value TO portal_user;
```

The web form sends `p_table`, `p_col`, `p_id`, and `p_val` as hidden or editable fields. **Anyone with browser dev tools can change all four.**

### Normal use

```sql
EXEC hr_app.update_value('HR.EMPLOYEES', 'FIRST_NAME', 101, 'Raj');
```

Generated SQL:

```sql
UPDATE HR.EMPLOYEES SET FIRST_NAME = 'Raj' WHERE employee_id = 101
```

This works as intended.

---

## 2. Three Attacks on Version 0

### Attack A — Injection through a **data value** (`p_val`)

```sql
EXEC hr_app.update_value('HR.EMPLOYEES', 'FIRST_NAME', 101,
                         'Raj'', salary = 999999 --');
```

Generated SQL:

```sql
UPDATE HR.EMPLOYEES SET FIRST_NAME = 'Raj', salary = 999999 --' WHERE employee_id = 101
```

- `'` closes the string early.
- `, salary = 999999` adds an attacker-chosen SET clause.
- `--` comments out the WHERE clause.

**Result: every employee's salary is now 999999.**

### Attack B — Injection through an **identifier** (`p_col`)

```sql
EXEC hr_app.update_value('HR.EMPLOYEES', 'salary = 0 --', 101, 'x');
```

Generated SQL:

```sql
UPDATE HR.EMPLOYEES SET salary = 0 -- = 'x' WHERE employee_id = 101
```

**Result: every salary in the company is zero.**

### Attack C — A **valid but unintended target** (privilege escalation)

```sql
EXEC hr_app.update_value('HR.APP_USERS', 'ROLE_NAME', 101, 'ADMIN');
```

Generated SQL:

```sql
UPDATE HR.APP_USERS SET ROLE_NAME = 'ADMIN' WHERE employee_id = 101
```

There are **no strange characters at all.** Every name is real and legal. The attacker simply pointed the generic procedure at a table it was never meant to touch.

**Result: employee 101 makes themselves an administrator.**

> **Key insight:** Attack C is the one people forget. It contains no "injection characters", so input-validation tools don't catch it.

---

## 3. Hardening, One Layer at a Time

### Layer 1 — Bind variables (stops Attack A)

```sql
-- VERSION 1
CREATE OR REPLACE PROCEDURE hr_app.update_value (
  p_table IN VARCHAR2, p_col IN VARCHAR2, p_id IN NUMBER, p_val IN VARCHAR2
) IS
BEGIN
  EXECUTE IMMEDIATE
    'UPDATE ' || p_table || ' SET ' || p_col ||
    ' = :val WHERE employee_id = :id'
    USING p_val, p_id;
END;
/
```

**Rerun Attack A:**

```sql
EXEC hr_app.update_value('HR.EMPLOYEES', 'FIRST_NAME', 101,
                         'Raj'', salary = 999999 --');
```

The SQL text is now always `UPDATE HR.EMPLOYEES SET FIRST_NAME = :val WHERE employee_id = :id`. The attacker's string travels **separately** as data, so employee 101's first name literally becomes `Raj', salary = 999999 --`. That's an ugly name, but no harm is done.

**Why it works:** Oracle parses the statement shape *before* it looks at the bind values, so data can never change the shape.

**Still vulnerable:** Attack B (`p_col`) and Attack C. You **cannot** bind identifiers. `UPDATE :tbl SET :col = ...` is illegal SQL.

> **Side note on `p_id`:** It's declared `NUMBER`, so concatenating it is far less dangerous, because a NUMBER can't contain SQL text. You still bind it, for consistency and so Oracle can reuse one shared cursor for all IDs instead of hard-parsing a new statement each time.

---

### Layer 2 — `DBMS_ASSERT` on identifiers (stops Attack B)

```sql
-- VERSION 2
CREATE OR REPLACE PROCEDURE hr_app.update_value (
  p_table IN VARCHAR2, p_col IN VARCHAR2, p_id IN NUMBER, p_val IN VARCHAR2
) IS
  v_table VARCHAR2(130);
  v_col   VARCHAR2(130);
BEGIN
  v_table := DBMS_ASSERT.SQL_OBJECT_NAME(p_table);   -- must be an existing object
  v_col   := DBMS_ASSERT.SIMPLE_SQL_NAME(p_col);     -- must be a legal simple name

  EXECUTE IMMEDIATE
    'UPDATE ' || v_table || ' SET ' || v_col ||
    ' = :val WHERE employee_id = :id'
    USING p_val, p_id;
END;
/
```

**Rerun Attack B:**

```sql
EXEC hr_app.update_value('HR.EMPLOYEES', 'salary = 0 --', 101, 'x');
-- ORA-44003: invalid SQL name
```

`salary = 0 --` contains spaces, `=`, and `--`, so it isn't a legal identifier, and it's rejected **before** any SQL runs.

**Rerun Attack C:**

```sql
EXEC hr_app.update_value('HR.APP_USERS', 'ROLE_NAME', 101, 'ADMIN');
-- Succeeds! The user is now ADMIN.
```

`HR.APP_USERS` is a real object and `ROLE_NAME` is a legal name, so `DBMS_ASSERT` happily approves both.

> `DBMS_ASSERT` checks **validity** ("is this a real, legal name?"), not **intent** ("is this code supposed to touch it?").

---

### Layer 3 — Whitelisting (stops Attack C)

```sql
-- VERSION 3
CREATE OR REPLACE PROCEDURE hr_app.update_value (
  p_table IN VARCHAR2, p_col IN VARCHAR2, p_id IN NUMBER, p_val IN VARCHAR2
) IS
  v_table VARCHAR2(130) := UPPER(TRIM(p_table));
  v_col   VARCHAR2(130) := UPPER(TRIM(p_col));
BEGIN
  -- Whitelist the exact table + column combinations this screen supports
  IF NOT (v_table = 'HR.EMPLOYEES' AND v_col IN ('FIRST_NAME', 'PHONE_NUMBER')) THEN
    RAISE_APPLICATION_ERROR(-20001,
      'Update of ' || v_table || '.' || v_col || ' is not permitted.');
  END IF;

  -- Still validate (cheap, and protects you if someone edits the whitelist badly)
  v_table := DBMS_ASSERT.SQL_OBJECT_NAME(v_table);
  v_col   := DBMS_ASSERT.SIMPLE_SQL_NAME(v_col);

  EXECUTE IMMEDIATE
    'UPDATE ' || v_table || ' SET ' || v_col ||
    ' = :val WHERE employee_id = :id'
    USING p_val, p_id;
END;
/
```

**Rerun Attack C:**

```sql
EXEC hr_app.update_value('HR.APP_USERS', 'ROLE_NAME', 101, 'ADMIN');
-- ORA-20001: Update of HR.APP_USERS.ROLE_NAME is not permitted.
```

**And a sneaky variant**, a legal table with a forbidden column:

```sql
EXEC hr_app.update_value('HR.EMPLOYEES', 'SALARY', 101, '999999');
-- ORA-20001: Update of HR.EMPLOYEES.SALARY is not permitted.
```

> Notice we whitelisted **columns as well as tables**. A table-only whitelist would still let users change their own salary through a screen meant for names.

**All three attacks are now blocked.** The next layers ask a different question: *what if a future developer breaks one of these layers?*

---

### Layer 4 — Least privilege (limits the damage)

Fix the DBA's lazy grant:

```sql
REVOKE UPDATE ANY TABLE FROM hr_app;

-- Direct grant (not via a role — roles are disabled in definer's-rights code!)
-- and column-level, only what the portal genuinely needs:
GRANT UPDATE (first_name, phone_number) ON hr.employees TO hr_app;
GRANT SELECT ON hr.employees TO hr_app;
```

Now imagine that six months later, a new developer "simplifies" the code and accidentally deletes the whitelist. Attack C comes back:

```sql
EXEC hr_app.update_value('HR.APP_USERS', 'ROLE_NAME', 101, 'ADMIN');
-- ORA-01031: insufficient privileges
```

The attack **got through the code**, but the owner simply doesn't have the power to do it. The salary variant fails the same way, because there's no `UPDATE` grant on the `SALARY` column.

> Least privilege is not prevention. It is **damage control** for when prevention fails.

---

### Layer 5 — Definer's vs invoker's rights (whose power does it run with?)

Here, **definer's rights (the default) is the correct choice.** `PORTAL_USER` has no table privileges at all. The procedure is a controlled gateway, like an ATM, and it runs with `HR_APP`'s now **narrow** privileges.

The contrast: if `update_value` were a **generic DBA utility** used by many schemas, you would make it invoker's rights so that each caller can only touch what they can already touch:

```sql
CREATE OR REPLACE PROCEDURE util.generic_update (...)
  AUTHID CURRENT_USER    -- runs with the CALLER's privileges
IS ...
```

> **Rule of thumb:** Use definer's rights with a narrowly privileged owner for app gateways. Use invoker's rights for shared tools.
> The dangerous combination is **definer's rights + a powerful owner + dynamic SQL**.

---

### Layer 6 — `ACCESSIBLE BY` (who can even call it?)

The generic updater is an internal helper. The portal should call a **business API** instead, such as "update my phone number", never the raw helper.

```sql
-- Internal helper: only the API package may call it (12c+)
CREATE OR REPLACE PACKAGE hr_app.emp_dml_internal
  ACCESSIBLE BY (PACKAGE hr_app.hr_portal_api)
AS
  PROCEDURE update_value (p_table IN VARCHAR2, p_col IN VARCHAR2,
                          p_id IN NUMBER, p_val IN VARCHAR2);
END;
/

-- Public API: fixed, meaningful operations
CREATE OR REPLACE PACKAGE hr_app.hr_portal_api AS
  PROCEDURE update_my_phone (p_emp_id IN NUMBER, p_phone IN VARCHAR2);
  PROCEDURE update_my_name  (p_emp_id IN NUMBER, p_name  IN VARCHAR2);
END;
/

CREATE OR REPLACE PACKAGE BODY hr_app.hr_portal_api AS
  PROCEDURE update_my_phone (p_emp_id IN NUMBER, p_phone IN VARCHAR2) IS
  BEGIN
    emp_dml_internal.update_value('HR.EMPLOYEES', 'PHONE_NUMBER', p_emp_id, p_phone);
  END;

  PROCEDURE update_my_name (p_emp_id IN NUMBER, p_name IN VARCHAR2) IS
  BEGIN
    emp_dml_internal.update_value('HR.EMPLOYEES', 'FIRST_NAME', p_emp_id, p_name);
  END;
END;
/

GRANT EXECUTE ON hr_app.hr_portal_api TO portal_user;
```

The table and column names are now **hard-coded by the API**, so the browser never sends them at all. Even if someone grants `EXECUTE` on the internal package by mistake:

```sql
-- Called directly from an anonymous block or another unit:
EXEC hr_app.emp_dml_internal.update_value('HR.APP_USERS','ROLE_NAME',101,'ADMIN');
-- PLS-00904: insufficient privilege to access object EMP_DML_INTERNAL
```

---

### Layer 7 — Do we even need dynamic SQL? (removes the surface)

Look at what we ended up with. The API only ever updates **two fixed columns on one fixed table.** Nothing actually varies at runtime except data values, so dynamic SQL isn't needed at all:

```sql
-- FINAL VERSION — no dynamic SQL, no injection surface
CREATE OR REPLACE PACKAGE BODY hr_app.hr_portal_api AS

  PROCEDURE update_my_phone (p_emp_id IN NUMBER, p_phone IN VARCHAR2) IS
  BEGIN
    UPDATE hr.employees
       SET phone_number = p_phone       -- PL/SQL variables in static SQL
     WHERE employee_id  = p_emp_id;     -- are bound automatically
  END;

  PROCEDURE update_my_name (p_emp_id IN NUMBER, p_name IN VARCHAR2) IS
  BEGIN
    UPDATE hr.employees
       SET first_name  = p_name
     WHERE employee_id = p_emp_id;
  END;

END;
/
```

Static SQL gives you several benefits:
- **No injection possible.** The statement shape is fixed at compile time.
- **Compile-time checks.** A typo in a column name fails at compile time, not in production.
- **Dependency tracking.** Oracle knows this package depends on `HR.EMPLOYEES`.

> The most secure dynamic SQL is the dynamic SQL you didn't write.
> The "generic, flexible" procedure was the root cause of all three attacks.

---

## 4. Scorecard — Which Layer Stops What?

| Layer | Attack A (data) | Attack B (identifier) | Attack C (valid but unintended) | Future developer mistake |
|---|---|---|---|---|
| 1. Bind variables | ✅ Blocked | ❌ | ❌ | — |
| 2. `DBMS_ASSERT` | ❌ | ✅ Blocked | ❌ | — |
| 3. Whitelist | — | ✅ | ✅ Blocked | — |
| 4. Least privilege | Limits damage | Limits damage | Limits damage | ✅ Contains it |
| 5. Rights model | Limits whose power | Limits whose power | Limits whose power | ✅ Contains it |
| 6. `ACCESSIBLE BY` | Reduces who can attack | Reduces who can attack | Reduces who can attack | ✅ Contains it |
| 7. Static SQL | ✅ No surface | ✅ No surface | ✅ No surface | ✅ No surface |

**No single row covers every column.** That is the whole point of defense in depth.

---

## 5. The One Habit

For every piece of a dynamic SQL string, ask:

> **"Is this a value or a name — and where did it come from?"**

| Piece | Example | Treatment |
|---|---|---|
| Data value | `'Raj'`, `101`, a date | **Bind it** — `:val` + `USING` |
| Identifier | table or column name | **Validate** (`DBMS_ASSERT`) **and whitelist** |
| SQL keyword | `UPDATE`, `SET`, `WHERE` | Hard-code it, never take it from input |
| Anything from a browser/user | — | Treat as hostile, always |

---

## 6. Important Points to Remember

- **Bind variables are the strongest defense, but only for values.** Identifiers can never be bound.
- **`DBMS_ASSERT` proves a name is legal or existing, not that it's allowed.** Always pair it with a whitelist for anything security-relevant.
- **Whitelist columns, not just tables.** A legal table with a sensitive column is still an attack.
- **Normalize before comparing** (`UPPER(TRIM(...))`). Otherwise `'hr.employees'` vs `'HR.EMPLOYEES'` either breaks valid calls or creates bypasses.
- **Privileges needed by definer's-rights code must be direct grants.** Roles are disabled inside it.
- **Definer's rights + powerful owner + dynamic SQL is the most dangerous combination.**
- **`ACCESSIBLE BY` (12c+)** restricts which PL/SQL units can call a unit. A violation raises `PLS-00904`.
- **Prefer static SQL.** If your whitelist turns out to be small and fixed, you probably didn't need dynamic SQL.
- **Logging generated SQL text** helps *detect* odd statement shapes in production. It's a **detective** control, not a preventive one.

**Error codes to recognize:**

| Error | Meaning in this context |
|---|---|
| `ORA-44002` | `DBMS_ASSERT.SQL_OBJECT_NAME`: invalid object name |
| `ORA-44003` | `DBMS_ASSERT.SIMPLE_SQL_NAME`: invalid SQL name |
| `ORA-01031` | Insufficient privileges (least privilege doing its job) |
| `PLS-00904` | Insufficient privilege to access object (`ACCESSIBLE BY` violation) |
| `ORA-20000`–`ORA-20999` | Your own `RAISE_APPLICATION_ERROR` (e.g., whitelist rejection) |

---

## 7. Scenario-Based Exercise Questions

**Q1.** A procedure builds `'SELECT * FROM orders WHERE status = ''' || p_status || ''''`. A tester passes `p_status => 'X'' OR ''1''=''1'`. What happens, and what is the fix?

> **A:** The generated SQL becomes `... WHERE status = 'X' OR '1'='1'`, which is always true, so **every order is returned**. `p_status` is a **data value**, so the fix is a bind variable:
> ```sql
> OPEN c FOR 'SELECT * FROM orders WHERE status = :s' USING p_status;
> ```
> Better still, if the table never changes, use static SQL: `SELECT * FROM orders WHERE status = p_status`.

---

**Q2.** A report procedure lets users choose a sort column: `'... ORDER BY ' || p_sort_col`. A developer tries to fix it with `ORDER BY :col USING p_sort_col`. The code runs without error, but the results are never sorted correctly. Why, and what is the right fix?

> **A:** `:col` binds a **value**, not a column name. Oracle sorts by the constant string (for example `'SALARY'`), which is identical for every row, so no real sorting happens. Column names are identifiers and **cannot be bound**. The fix is to whitelist the allowed sort columns, then validate:
> ```sql
> IF UPPER(p_sort_col) NOT IN ('LAST_NAME','HIRE_DATE','SALARY') THEN
>   RAISE_APPLICATION_ERROR(-20002, 'Invalid sort column');
> END IF;
> v_sql := v_sql || ' ORDER BY ' || DBMS_ASSERT.SIMPLE_SQL_NAME(p_sort_col);
> ```

---

**Q3.** A team's purge utility uses `DBMS_ASSERT.SQL_OBJECT_NAME(p_table)` before running `'DELETE FROM ' || p_table || ' WHERE created < SYSDATE - 90'`. An intern calls it with `p_table => 'HR.AUDIT_LOG'` and wipes 90 days of audit history. `DBMS_ASSERT` didn't complain. Why, and which two layers would have prevented or contained this?

> **A:** `HR.AUDIT_LOG` is a real, existing object, so it passes `SQL_OBJECT_NAME`, which checks validity, not intent.
> - **Layer 3 (whitelist):** list the tables the purge job is meant to clean (`STAGING_ORDERS`, `TEMP_IMPORTS`, …) and reject everything else.
> - **Layer 4 (least privilege):** the utility's owner should have `DELETE` only on those staging tables, directly granted, so a `DELETE` on `AUDIT_LOG` would fail with `ORA-01031`.
>
> Layer 5 is also worth considering: if the utility were `AUTHID CURRENT_USER`, the intern could only purge tables the intern already had `DELETE` on.

---

**Q4.** After whitelisting, a developer's generic procedure only ever updates `EMPLOYEES.FIRST_NAME` and `EMPLOYEES.PHONE_NUMBER`. A senior reviewer says, "Delete the dynamic SQL entirely." Give three concrete benefits of doing that.

> **A:**
> 1. **Zero injection surface.** The statement shape is fixed at compile time, and PL/SQL variables are bound automatically.
> 2. **Compile-time validation.** A misspelled column fails at compile time instead of at runtime in production.
> 3. **Dependency tracking and privilege checks at compile time.** Oracle records that the unit depends on `EMPLOYEES`, invalidates it if the table changes, and catches missing privileges at compile time. Dynamic SQL hides all of that until runtime (recall: dynamic SQL privileges are checked only when it executes).
>
> A further benefit: the code becomes easier to read and review.

---

**Q5.** In TechNova's setup, `HR_APP` received `UPDATE` on `HR.EMPLOYEES` through a role called `HR_WRITER_ROLE`. The hardened Version 3 procedure compiles, but every call fails. Which error do you expect, why, and what's the fix?

> **A:** You'd see `ORA-01031: insufficient privileges` **at runtime**, the first time `EXECUTE IMMEDIATE` runs. Because the SQL is dynamic, it isn't privilege-checked at compile time, which is why the compile succeeds. At runtime, `update_value` is a definer's-rights unit, so only `PUBLIC` is enabled, and `HR_WRITER_ROLE` is switched off.
> **Fix:** grant the privilege **directly**:
> ```sql
> GRANT UPDATE (first_name, phone_number) ON hr.employees TO hr_app;
> ```
> If the same `UPDATE` had been **static** SQL, the failure would instead appear at **compile time**, typically as `PL/SQL: ORA-00942: table or view does not exist` or an insufficient-privilege compile error.

---

**Q6.** A teammate argues: "We bind every value AND use `DBMS_ASSERT` on every identifier, so `ACCESSIBLE BY` and least privilege are pointless overhead." How do you respond?

> **A:** Bind variables and `DBMS_ASSERT` are **preventive** controls. They only work if every developer applies them correctly in every code path, forever. Least privilege and `ACCESSIBLE BY` are **containment** controls: they limit **what an attacker can reach** and **who can reach the code at all** when a preventive layer is skipped, misapplied, or removed in a future change (as in the Layer 4 example). Defense in depth assumes that some layer will eventually fail. The later layers exist so that one mistake doesn't become a breach.

---

## 8. One-Line Summary

**Bind the values, validate and whitelist the names, give the owner only what it needs, restrict who can call it — and if nothing really varies, don't use dynamic SQL at all.**