# Definer's Rights vs. Invoker's Rights (AUTHID) — Practical Guide

## The Core Idea

### Definer's rights
It's the **default**. You own the table, write a procedure, and give others permission to run it. They can do **only what your procedure does**, nothing more. Your table stays protected.

### Invoker's rights (`AUTHID CURRENT_USER`)
It does **not** mean "anyone can do anything." It means the code runs with the **caller's own permissions**, on the **caller's own tables**. So:

- Ravi can only do what Ravi was **already allowed** to do.
- Your table is **not exposed** to Ravi through this code. If Ravi has no permission on your table, the procedure can't touch it for him either.
- The only thing you're sharing is the **logic**, not your data or your power.

So invoker's rights is actually the **safer** one, because it never lends your permissions to anyone. You use it when you want others to reuse your code on their own data.

### In one line each

> **Definer's rights:** I share access to **my data**, but only through the limited actions I wrote in the procedure.
>
> **Invoker's rights:** I share **my code**. Each person runs it on **their own data**, with **their own permissions**. They can't do anything they couldn't already do.

---

## The Setup

There are two users in the database, **HR** and **RAVI**, and both happen to have a table called `employees`:

| Schema | Table       | Rows                            |
|--------|-------------|---------------------------------|
| HR     | `employees` | 5 rows (the real company data)  |
| RAVI   | `employees` | 2 rows (Ravi's own test data)   |

Ravi has **no permission** on HR's table.

---

## Example 1: Definer's Rights (sharing data safely)

HR writes this procedure. There's no `AUTHID`, so it uses definer's rights by default:

```sql
-- Logged in as HR
CREATE OR REPLACE PROCEDURE count_emp IS
  v_count NUMBER;
BEGIN
  SELECT COUNT(*) INTO v_count FROM employees;
  DBMS_OUTPUT.put_line('Count: ' || v_count);
END;
/

GRANT EXECUTE ON count_emp TO ravi;
```

Now Ravi tries three things:

```sql
-- Logged in as RAVI

EXEC hr.count_emp;
-- Output: Count: 5        ✅ Works, and reads HR's table

SELECT * FROM hr.employees;
-- ORA-00942: table or view does not exist   ❌ No direct access

DELETE FROM hr.employees;
-- ORA-00942                                  ❌ Can't touch it
```

**What happened:** Ravi got the count of HR's employees, but he can't see or change HR's table directly. The procedure is the only door, and it only counts.

---

## Example 2: Invoker's Rights (sharing code)

HR makes one change and adds `AUTHID CURRENT_USER`:

```sql
-- Logged in as HR
CREATE OR REPLACE PROCEDURE count_emp
AUTHID CURRENT_USER IS
  v_count NUMBER;
BEGIN
  SELECT COUNT(*) INTO v_count FROM employees;
  DBMS_OUTPUT.put_line('Count: ' || v_count);
END;
/
```

Ravi runs the same command:

```sql
-- Logged in as RAVI
EXEC hr.count_emp;
-- Output: Count: 2        ← Ravi's OWN table, not HR's
```

**What happened:** the same code ran, but as Ravi, so `employees` meant **Ravi's** table.

And if a third user, Priya, who has no `employees` table, runs it:

```sql
-- Logged in as PRIYA
EXEC hr.count_emp;
-- ORA-00942: table or view does not exist  ❌
```

The code didn't lend her HR's access. She gets only what she already has.

---

## Example 3: A Real Workplace Case for Definer's Rights

HR wants managers to give raises, but **never more than 10%**. If managers could update the table directly, they could set any salary. So HR writes this:

```sql
-- Logged in as HR
CREATE OR REPLACE PROCEDURE give_raise(p_emp_id NUMBER, p_pct NUMBER) IS
BEGIN
  IF p_pct > 10 THEN
    RAISE_APPLICATION_ERROR(-20001, 'Raise cannot exceed 10%');
  END IF;

  UPDATE employees
  SET salary = salary * (1 + p_pct/100)
  WHERE emp_id = p_emp_id;
END;
/

GRANT EXECUTE ON give_raise TO ravi;
```

```sql
-- Logged in as RAVI (a manager)
EXEC hr.give_raise(101, 5);    -- ✅ 5% raise applied
EXEC hr.give_raise(101, 50);   -- ❌ Error: Raise cannot exceed 10%
UPDATE hr.employees SET salary = 999999;   -- ❌ No direct access
```

**This is the main reason definer's rights exists:** people can change your data, but only by following **your rules**.

---

## Example 4: A Real Workplace Case for Invoker's Rights

A central DBA team writes a cleanup utility. Every application team has its own `app_logs` table and wants to delete logs older than 30 days:

```sql
-- Logged in as DBA_TOOLS
CREATE OR REPLACE PROCEDURE purge_old_logs
AUTHID CURRENT_USER IS
BEGIN
  DELETE FROM app_logs WHERE log_date < SYSDATE - 30;
  DBMS_OUTPUT.put_line(SQL%ROWCOUNT || ' rows deleted');
END;
/

GRANT EXECUTE ON purge_old_logs TO PUBLIC;
```

| Who runs it   | What gets cleaned        |
|---------------|--------------------------|
| `SALES_APP`   | `SALES_APP.app_logs`     |
| `BILLING_APP` | `BILLING_APP.app_logs`   |
| `HR_APP`      | `HR_APP.app_logs`        |

The code is written **once** and works for everyone on **their own** logs. Nobody can accidentally delete another team's logs, and the DBA team's own logs are never touched by others.

> **Small detail:** for this to compile, `DBA_TOOLS` also needs an `app_logs` table of its own. It acts as a template.

---

## Quick Summary

|                       | Definer's Rights (Examples 1 & 3)            | Invoker's Rights (Examples 2 & 4)        |
|-----------------------|----------------------------------------------|------------------------------------------|
| Keyword               | Default (or `AUTHID DEFINER`)                | `AUTHID CURRENT_USER`                    |
| Whose table?          | The **owner's** (HR)                         | The **caller's** (Ravi's)                |
| Whose permissions?    | The owner's                                  | The caller's                             |
| What's being shared?  | **Data**, through controlled code            | **Code**, used on your own data          |
| Real use              | Salary raises, banking, locked-down data     | Reusable utilities, cleanup, reports     |

> **Memory hook:** Definer = "use *my stuff*, through my code." Invoker = "use *my code*, on your stuff."

---

## Commonly Asked Interview Questions

### Q1. What is the difference between definer's rights and invoker's rights?
**Answer:** Definer's rights runs the program with the **owner's** privileges and resolves table names in the **owner's** schema. Invoker's rights runs it with the **caller's** privileges and resolves table names in the **caller's** schema.

### Q2. Which one is the default?
**Answer:** **Definer's rights.** If you don't write any `AUTHID` clause, Oracle treats it as `AUTHID DEFINER`.

### Q3. How do you make a procedure use invoker's rights?
**Answer:** Add `AUTHID CURRENT_USER` after the procedure header, before `IS`/`AS`:
```sql
CREATE OR REPLACE PROCEDURE my_proc
AUTHID CURRENT_USER IS
BEGIN
  NULL;
END;
/
```

### Q4. User Ravi has EXECUTE on `hr.count_emp` (definer's rights) but no SELECT on `hr.employees`. Can he run the procedure successfully?
**Answer:** **Yes.** With definer's rights, the procedure uses HR's privileges. Ravi only needs `EXECUTE` on the procedure, not access to the table.

### Q5. Same question, but the procedure is invoker's rights. What happens?
**Answer:** The query looks for `employees` in **Ravi's** schema using **Ravi's** privileges. If Ravi has his own `employees` table, it counts that. If not, he gets `ORA-00942: table or view does not exist`.

### Q6. Give a real-world use case for each.
**Answer:**
- **Definer's rights:** a `give_raise` procedure that lets managers update salaries but enforces a maximum limit. Users can't touch the salary table directly.
- **Invoker's rights:** a central utility like `purge_old_logs` that every application schema calls to clean up **its own** log table.

### Q7. Which one is more secure, and why?
**Answer:** It depends on the goal, but **invoker's rights is safer from a privilege point of view** because it never lends the owner's privileges to the caller. Definer's rights is safer for **protecting data**, because users can only access it through controlled code.

### Q8. Why is SQL injection more dangerous in definer's rights code?
**Answer:** Because the injected SQL runs with the **owner's** privileges, which are often much more powerful than the caller's. An attacker could do things through the procedure that they could never do on their own. In invoker's rights code, injected SQL can only do what the caller was already allowed to do.

### Q9. Can you use definer's rights for some procedures and invoker's rights for others inside the same package?
**Answer:** **No.** `AUTHID` is set once in the **package specification** and applies to the entire package (spec and body).

### Q10. Can you use `AUTHID CURRENT_USER` on a trigger?
**Answer:** **No.** Triggers always run with definer's rights. `AUTHID` can be used on standalone procedures, functions, packages, and object types.

### Q11. In a definer's rights procedure, can the owner rely on privileges granted through a role?
**Answer:** **No.** Roles are disabled inside definer's rights units. The owner must have the privilege granted **directly** (e.g., `GRANT SELECT ON x TO hr`, not through a role). In invoker's rights units, the caller's **enabled roles do count**.

### Q12. For an invoker's rights procedure, why does the owner's schema still need the table?
**Answer:** The procedure must **compile** in the owner's schema first. At compile time, Oracle checks that `employees` exists and has the right columns. At run time, it uses the caller's table instead. The owner's table acts as a **template**.

### Q13. How can you check whether an existing procedure uses definer's or invoker's rights?
**Answer:** Query the `AUTHID` column of the data dictionary:
```sql
SELECT object_name, authid
FROM   user_procedures
WHERE  object_name = 'COUNT_EMP';
-- AUTHID column shows DEFINER or CURRENT_USER
```

### Q14. Scenario: You're building a reporting package that 20 different teams will use on their own tables. Which model will you choose?
**Answer:** **Invoker's rights**, so the same code works on each team's own data with their own permissions, and no team can reach another team's tables through it.

### Q15. Scenario: You want users to insert orders only after the procedure validates stock and price, never directly. Which model?
**Answer:** **Definer's rights.** Grant only `EXECUTE` on the procedure. Users can insert orders only through your validation logic, and the table itself stays locked.