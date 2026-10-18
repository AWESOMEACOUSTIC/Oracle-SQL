# Package Initialization Block in PL/SQL — Simple Guide

## The Problem It Solves

Package variables **remember values for the whole session** (package state). But who **fills** those variables in the first place?

Say your package needs the company name and tax rate from a settings table, and many procedures in the package use them. You have two bad options:

1. **Query the table inside every procedure.** That repeats the same query again and again, which is slow and wasteful.
2. **Make the user call a `load_settings` procedure first.** People will forget, and then everything breaks.

**The initialization block is the third, good option.** It's a piece of code that Oracle runs **automatically, once per session**, the first time anyone touches the package. It loads the values, and every call after that just uses them.

### Analogy: opening a shop in the morning

The first time you arrive, you switch on the lights, open the cash counter, and arrange the shelves. That happens **once**. After that, you just serve customers all day.

| Shop                 | Package                                         |
|----------------------|-------------------------------------------------|
| Morning setup        | **Initialization block** (runs once per session) |
| Serving customers    | **Procedure calls** (happen many times)         |
| Closing the shop     | **Logout** (everything cleared; setup runs again next time) |

---

## How It Looks

It's a `BEGIN ... END` at the **bottom** of the package body, with no procedure name:

```sql
CREATE OR REPLACE PACKAGE BODY hr_pkg IS
  g_company_name VARCHAR2(100);   -- package variables (empty at first)
  g_tax_rate     NUMBER;

  PROCEDURE show_info IS
  BEGIN
    DBMS_OUTPUT.put_line(g_company_name || ' @ ' || g_tax_rate);
  END;

BEGIN   -- ← INITIALIZATION BLOCK: no name, sits at the very end
  SELECT company_name, tax_rate
  INTO   g_company_name, g_tax_rate
  FROM   company_settings
  WHERE  ROWNUM = 1;
END hr_pkg;
/
```

**How to tell it apart from a procedure:**
- It has **no `PROCEDURE` or `FUNCTION` header** above it.
- It is always the **last thing** in the package body, after all subprograms.
- It has **no name and no parameters**.
- It exists only in the **package body**, never in the spec.

---

## Example 1: Seeing It Run Once

> Run in SQL*Plus, SQL Developer, or Oracle Live SQL with `SET SERVEROUTPUT ON`.

### Setup

```sql
CREATE TABLE company_settings (company_name VARCHAR2(100), tax_rate NUMBER);
INSERT INTO company_settings VALUES ('Cognizant', 18);
COMMIT;

CREATE OR REPLACE PACKAGE hr_pkg IS
  PROCEDURE show_info;
END hr_pkg;
/

CREATE OR REPLACE PACKAGE BODY hr_pkg IS
  g_company_name VARCHAR2(100);
  g_tax_rate     NUMBER;

  PROCEDURE show_info IS
  BEGIN
    DBMS_OUTPUT.put_line(g_company_name || ' @ ' || g_tax_rate || '%');
  END;

BEGIN
  DBMS_OUTPUT.put_line('>> Init block running: loading settings...');
  SELECT company_name, tax_rate
  INTO   g_company_name, g_tax_rate
  FROM   company_settings
  WHERE  ROWNUM = 1;
END hr_pkg;
/
```

### Calling it several times

```sql
EXEC hr_pkg.show_info;
-- >> Init block running: loading settings...   ← runs automatically, first time only
-- Cognizant @ 18%

EXEC hr_pkg.show_info;
-- Cognizant @ 18%                              ← no init message, already loaded

EXEC hr_pkg.show_info;
-- Cognizant @ 18%
```

**Takeaway:** you never called the init block yourself. Oracle ran it on first use, and only once.

---

## Example 2: When Does It Run Again?

It runs again only when the package's memory is wiped:

```sql
EXEC DBMS_SESSION.RESET_PACKAGE;

EXEC hr_pkg.show_info;
-- >> Init block running: loading settings...   ← memory wiped, setup runs again
-- Cognizant @ 18%
```

| Situation                                           | Does init run again?                       |
|-----------------------------------------------------|--------------------------------------------|
| Calling the package again in the same session       | ❌ No                                       |
| A different user/session uses the package          | ✅ Yes, once for that session (own copy)    |
| Logout and log back in                              | ✅ Yes                                      |
| After `DBMS_SESSION.RESET_PACKAGE`                  | ✅ Yes, on next use                         |
| Package is `SERIALLY_REUSABLE`                      | ✅ Yes, on every new top-level call         |
| Package is recompiled                               | ✅ Yes, on next use (after ORA-04068 once)  |

---

## Example 3: The Tricky Part — When the Init Block Fails

Suppose the settings table is **empty**:

```sql
DELETE FROM company_settings;
COMMIT;
```

Log in again (new session) and call the procedure:

```sql
EXEC hr_pkg.show_info;
-- >> Init block running: loading settings...
-- ORA-01403: no data found
-- ORA-06512: at "HR.HR_PKG", line ...
```

**Why this confuses people:** you called `show_info`, and there's nothing wrong with `show_info`. The error came from the **init block's** `SELECT`, which found no rows. The package couldn't finish its setup, so nothing in it can be used.

On the next call, Oracle **tries the setup again**. If the table now has data, it works. If not, it fails again.

### The safer way: handle errors inside the init block

```sql
CREATE OR REPLACE PACKAGE BODY hr_pkg IS
  g_company_name VARCHAR2(100);
  g_tax_rate     NUMBER;

  PROCEDURE show_info IS
  BEGIN
    DBMS_OUTPUT.put_line(g_company_name || ' @ ' || g_tax_rate || '%');
  END;

BEGIN
  SELECT company_name, tax_rate
  INTO   g_company_name, g_tax_rate
  FROM   company_settings
  WHERE  ROWNUM = 1;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    g_company_name := 'Unknown';
    g_tax_rate     := 0;
END hr_pkg;
/
```

```sql
EXEC hr_pkg.show_info;
-- Unknown @ 0%      ← no crash, defaults used
```

> **Debugging tip:** if the *very first* call to any procedure in a package fails with an error that has nothing to do with that procedure, check the **initialization block** first.

---

## Example 4: Real-World Use — Caching Lookup Data

Load all country codes **once per session** into a collection, so procedures never query the table again:

```sql
CREATE TABLE countries_demo (code VARCHAR2(2), name VARCHAR2(50));
INSERT INTO countries_demo VALUES ('IN', 'India');
INSERT INTO countries_demo VALUES ('US', 'United States');
INSERT INTO countries_demo VALUES ('JP', 'Japan');
COMMIT;

CREATE OR REPLACE PACKAGE country_pkg IS
  FUNCTION get_name(p_code VARCHAR2) RETURN VARCHAR2;
END country_pkg;
/

CREATE OR REPLACE PACKAGE BODY country_pkg IS
  TYPE t_country_map IS TABLE OF VARCHAR2(50) INDEX BY VARCHAR2(2);
  g_countries t_country_map;   -- package-level cache

  FUNCTION get_name(p_code VARCHAR2) RETURN VARCHAR2 IS
  BEGIN
    IF g_countries.EXISTS(p_code) THEN
      RETURN g_countries(p_code);
    END IF;
    RETURN 'Not found';
  END;

BEGIN   -- init block fills the cache once
  DBMS_OUTPUT.put_line('>> Loading country cache...');
  FOR r IN (SELECT code, name FROM countries_demo) LOOP
    g_countries(r.code) := r.name;
  END LOOP;
END country_pkg;
/
```

```sql
EXEC DBMS_OUTPUT.put_line(country_pkg.get_name('IN'));
-- >> Loading country cache...
-- India

EXEC DBMS_OUTPUT.put_line(country_pkg.get_name('JP'));
-- Japan            ← served from memory, no table query

EXEC DBMS_OUTPUT.put_line(country_pkg.get_name('XX'));
-- Not found
```

---

## Where It's Used in Real Projects

- **Loading configuration:** company settings, tax rates, feature flags.
- **Caching lookup data:** country codes, currency rates, status codes loaded into a collection once.
- **Session setup:** recording who logged in or when the session started.

---

## Quick Summary

| Point | Detail |
|---|---|
| What it is | A nameless `BEGIN...END` at the bottom of a package body |
| When it runs | Automatically, **once per session**, on first use of the package |
| Who calls it | Nobody. Oracle runs it by itself |
| Purpose | Fill package variables/caches with values that are expensive to fetch |
| Runs again after | Logout, `RESET_PACKAGE`, recompile, or every call if `SERIALLY_REUSABLE` |
| If it fails | The call that triggered it fails too; the next reference retries the setup |
| Best practice | Handle exceptions inside it and set safe defaults |

> **Memory hook:** The init block is the package's **morning setup**: done once, automatically, before the first customer is served.

---

## Commonly Asked Interview Questions

### Q1. What is a package initialization block?
**Answer:** An optional, nameless `BEGIN...END` section at the end of a package body. Oracle runs it automatically, once per session, the first time the package is referenced. It is mainly used to initialize package-level variables.

### Q2. When exactly does the initialization block run?
**Answer:** Once per session, on the **first reference** to the package: calling any of its subprograms, or reading one of its public variables or constants. It does **not** run at compile time, and it does **not** run on every call.

### Q3. Can you call the initialization block explicitly?
**Answer:** **No.** It has no name, so it can't be called. Oracle runs it automatically.

### Q4. Where is the initialization block written?
**Answer:** In the **package body**, after all subprogram bodies, just before the final `END package_name;`. A package specification cannot have one.

### Q5. Can the initialization block take parameters?
**Answer:** **No.** It has no name, header, or parameters. If you need input-based setup, write a normal `init(p_param)` procedure instead.

### Q6. Two different users call the same package. How many times does the initialization block run?
**Answer:** **Twice,** once for each session, because each session has its own copy of the package state.

### Q7. Will the initialization block run again after `DBMS_SESSION.RESET_PACKAGE`?
**Answer:** **Yes.** The reset wipes package state, so the next reference re-instantiates the package and runs the init block again.

### Q8. How does `PRAGMA SERIALLY_REUSABLE` affect the initialization block?
**Answer:** The package state resets for every new top-level call, so the init block runs again at the **start of every call** that uses the package.

### Q9. What happens if the initialization block raises an unhandled exception?
**Answer:** The call that triggered it fails with that error, even if the procedure you called is fine. The package is not initialized, so the **next reference tries the initialization again**.

### Q10. Your very first call to `hr_pkg.show_info` fails with `ORA-01403: no data found`, but `show_info` contains no SELECT. What is the likely cause?
**Answer:** The **initialization block** has a `SELECT INTO` that returned no rows. The error comes from the init block, which runs automatically before `show_info`.

### Q11. How do you make an initialization block safer?
**Answer:** Add an `EXCEPTION` section inside it to handle expected errors (like `NO_DATA_FOUND`) and assign safe default values, so the package remains usable.

### Q12. Give a real-world use case of an initialization block.
**Answer:** Caching lookup data, such as loading all country codes into an associative array once per session, so that functions read from memory instead of querying the table on every call.

### Q13. What is the difference between giving a package variable a default value and setting it in the initialization block?
**Answer:** A default value (`g_rate NUMBER := 18;`) works for simple, fixed values. The initialization block is needed when setup requires **logic or queries**, such as reading from a table, looping, or handling errors.

### Q14. Is the initialization block mandatory in a package body?
**Answer:** **No.** It is optional. Most packages don't need one.

### Q15. Scenario: Config values in the settings table change while a user's session is active. Will the user see the new values?
**Answer:** **No,** not automatically. The init block already ran once and cached the old values for that session. The user sees the new values only after logging in again, after `DBMS_SESSION.RESET_PACKAGE`, or if the package provides a `refresh` procedure that reloads them.