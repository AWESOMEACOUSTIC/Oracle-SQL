# Package State in PL/SQL — Simple Guide

## What Are You Learning?

A procedure's **local variables** start fresh on every call, so the procedure forgets everything from the last call:

```sql
PROCEDURE add_one IS
  v_count NUMBER := 0;   -- starts at 0 EVERY time
BEGIN
  v_count := v_count + 1;
  DBMS_OUTPUT.put_line(v_count);   -- always prints 1
END;
```

Call it 5 times and it prints `1` five times. It has no memory.

**Package state is the fix.** A variable declared at the **package level** (outside any procedure) **remembers its value** between calls. That remembered value is called **package state**.

### Analogy: calculator vs scoreboard

- **Local variable = calculator.** You switch it on, do a sum, and switch it off. Next time it starts from zero.
- **Package variable = scoreboard.** Every run gets added to the same board, which keeps the total until the match ends.

The "match" is your **session**: from the moment you log in to the database until you log out.

### Where it's used in real projects

| Use              | What package state does                                                  |
|------------------|--------------------------------------------------------------------------|
| Counters         | Count how many times something happened in this session                  |
| Caching          | Load a lookup table (e.g., country codes) once, reuse without re-querying |
| Flags            | Remember "already validated" or the current user's role across calls     |
| Paging data      | Keep a package-level cursor open and fetch in batches                    |

---

## The Four Rules

1. **Each session gets its own copy.** Package state is private to each session and ends at logout.
2. **`DBMS_SESSION.RESET_PACKAGE`** clears all package state without logging out.
3. **`PRAGMA SERIALLY_REUSABLE`** turns the memory off. State lasts for one top-level call only, which saves memory.
4. **Package-level cursors are package state too.** They stay open and positioned between calls.

> Run the examples below in SQL*Plus, SQL Developer, or Oracle Live SQL with `SET SERVEROUTPUT ON`.

### Base package (used for Rules 1 and 2)

```sql
CREATE OR REPLACE PACKAGE counter_pkg IS
  PROCEDURE increment;
  FUNCTION  get_count RETURN NUMBER;
END counter_pkg;
/

CREATE OR REPLACE PACKAGE BODY counter_pkg IS
  g_count NUMBER := 0;   -- package state

  PROCEDURE increment IS
  BEGIN
    g_count := g_count + 1;
  END;

  FUNCTION get_count RETURN NUMBER IS
  BEGIN
    RETURN g_count;
  END;
END counter_pkg;
/
```

---

## Rule 1: Each Session Gets Its Own Copy

Open **two separate windows** (two logins). Each window is a separate session.

**Window 1 (Session A):**
```sql
EXEC counter_pkg.increment;
EXEC counter_pkg.increment;
EXEC counter_pkg.increment;

EXEC DBMS_OUTPUT.put_line('Session A sees: ' || counter_pkg.get_count);
-- Output: Session A sees: 3
```

**Window 2 (Session B), at the same time:**
```sql
EXEC DBMS_OUTPUT.put_line('Session B sees: ' || counter_pkg.get_count);
-- Output: Session B sees: 0      ← A's increments are invisible here

EXEC counter_pkg.increment;
EXEC DBMS_OUTPUT.put_line('Session B sees: ' || counter_pkg.get_count);
-- Output: Session B sees: 1
```

**Back in Window 1:**
```sql
EXEC DBMS_OUTPUT.put_line('Session A sees: ' || counter_pkg.get_count);
-- Output: Session A sees: 3      ← B's increment didn't affect A
```

**Log out of Window 1 and log back in:**
```sql
EXEC DBMS_OUTPUT.put_line('Session A sees: ' || counter_pkg.get_count);
-- Output: Session A sees: 0      ← new session, fresh copy
```

| | Session A | Session B |
|---|---|---|
| After A increments 3 times | 3 | 0 |
| After B increments once    | 3 | 1 |

**Takeaway:** values survive across calls (1 → 2 → 3), but they are **private to each session** and disappear at logout. Technically, package state lives in each session's private memory, the **UGA (User Global Area)**.

---

## Rule 2: `RESET_PACKAGE` Clears Memory Without Logging Out

### Basic reset

```sql
EXEC counter_pkg.increment;
EXEC counter_pkg.increment;
EXEC DBMS_OUTPUT.put_line('Before reset: ' || counter_pkg.get_count);
-- Output: Before reset: 2

EXEC DBMS_SESSION.RESET_PACKAGE;

EXEC DBMS_OUTPUT.put_line('After reset: ' || counter_pkg.get_count);
-- Output: After reset: 0         ← same session, memory wiped
```

> **Note:** run the reset as its **own separate call**. The reset takes effect only after the current call finishes, so resetting and reading inside the same `BEGIN...END` block won't show the effect yet.
>
> `RESET_PACKAGE` resets **all** packages in the session. There is no per-package reset command (the `ALTER PACKAGE ... STATE` syntax does not exist).

### Why it matters: connection pooling

Real applications (e.g., a Java web app) keep a few database connections open and **reuse** them for different users. This is called **connection pooling**. Leftover package state can leak from one user to the next.

```sql
CREATE OR REPLACE PACKAGE app_ctx IS
  PROCEDURE set_user(p_name VARCHAR2);
  FUNCTION  get_user RETURN VARCHAR2;
END app_ctx;
/

CREATE OR REPLACE PACKAGE BODY app_ctx IS
  g_user VARCHAR2(50);   -- package state

  PROCEDURE set_user(p_name VARCHAR2) IS
  BEGIN
    g_user := p_name;
  END;

  FUNCTION get_user RETURN VARCHAR2 IS
  BEGIN
    RETURN NVL(g_user, 'nobody');
  END;
END app_ctx;
/
```

**The problem:**
```sql
-- Amit's request uses the connection
EXEC app_ctx.set_user('Amit');
EXEC DBMS_OUTPUT.put_line('Current user: ' || app_ctx.get_user);
-- Output: Current user: Amit

-- Amit finishes. The SAME connection is given to Neha.
-- Neha's code forgets to call set_user:
EXEC DBMS_OUTPUT.put_line('Current user: ' || app_ctx.get_user);
-- Output: Current user: Amit     ❌ Neha is treated as Amit!
```

**The fix:**
```sql
EXEC app_ctx.set_user('Amit');
-- Amit finishes...

EXEC DBMS_SESSION.RESET_PACKAGE;   -- clean up before reuse

-- Neha gets the connection
EXEC DBMS_OUTPUT.put_line('Current user: ' || app_ctx.get_user);
-- Output: Current user: nobody   ✅ clean start
```

---

## Rule 3: `SERIALLY_REUSABLE` Turns the Memory Off

The pragma goes in **both** the spec and the body:

```sql
CREATE OR REPLACE PACKAGE sr_counter_pkg IS
  PRAGMA SERIALLY_REUSABLE;
  PROCEDURE increment;
  FUNCTION  get_count RETURN NUMBER;
END sr_counter_pkg;
/

CREATE OR REPLACE PACKAGE BODY sr_counter_pkg IS
  PRAGMA SERIALLY_REUSABLE;
  g_count NUMBER := 0;

  PROCEDURE increment IS
  BEGIN
    g_count := g_count + 1;
  END;

  FUNCTION get_count RETURN NUMBER IS
  BEGIN
    RETURN g_count;
  END;
END sr_counter_pkg;
/
```

### Separate calls: memory is lost

```sql
EXEC sr_counter_pkg.increment;
EXEC sr_counter_pkg.increment;
EXEC DBMS_OUTPUT.put_line('Count: ' || sr_counter_pkg.get_count);
-- Output: Count: 0      ← each EXEC is a new call, so it resets every time
```

(The normal `counter_pkg` would print `2` here.)

### Inside one block: memory lasts until the block ends

```sql
BEGIN
  sr_counter_pkg.increment;
  sr_counter_pkg.increment;
  DBMS_OUTPUT.put_line('Inside block: ' || sr_counter_pkg.get_count);
END;
/
-- Output: Inside block: 2   ← same top-level call, state survives within it

EXEC DBMS_OUTPUT.put_line('Next call: ' || sr_counter_pkg.get_count);
-- Output: Next call: 0      ← new call, reset again
```

### Why would anyone want this?

**To save memory.** If 10,000 users are logged in and each calls a package only once in a while, a normal package keeps 10,000 copies of its variables in memory for every session's whole lifetime, mostly unused. With `SERIALLY_REUSABLE`, the memory is borrowed from a shared pool only while the call runs, then handed back.

**Takeaway:** with `SERIALLY_REUSABLE`, state lasts for **one top-level call** (one `EXEC` or one `BEGIN...END` block), not the whole session.

---

## Rule 4: Package Cursors Stay Open Between Calls

### Setup

```sql
CREATE TABLE emp_demo (emp_id NUMBER, emp_name VARCHAR2(30));
INSERT INTO emp_demo VALUES (1, 'Amit');
INSERT INTO emp_demo VALUES (2, 'Neha');
INSERT INTO emp_demo VALUES (3, 'Ravi');
COMMIT;
```

### Package with a package-level cursor

```sql
CREATE OR REPLACE PACKAGE emp_reader IS
  PROCEDURE open_list;
  PROCEDURE show_next;
  PROCEDURE close_list;
END emp_reader;
/

CREATE OR REPLACE PACKAGE BODY emp_reader IS
  CURSOR c_emp IS
    SELECT emp_id, emp_name FROM emp_demo ORDER BY emp_id;   -- package state

  PROCEDURE open_list IS
  BEGIN
    OPEN c_emp;
  END;

  PROCEDURE show_next IS
    v_id   emp_demo.emp_id%TYPE;
    v_name emp_demo.emp_name%TYPE;
  BEGIN
    FETCH c_emp INTO v_id, v_name;
    IF c_emp%FOUND THEN
      DBMS_OUTPUT.put_line(v_id || ' - ' || v_name);
    ELSE
      DBMS_OUTPUT.put_line('No more rows');
    END IF;
  END;

  PROCEDURE close_list IS
  BEGIN
    CLOSE c_emp;
  END;
END emp_reader;
/
```

### Each line is a separate call, yet the cursor remembers its position

```sql
EXEC emp_reader.open_list;

EXEC emp_reader.show_next;   -- Output: 1 - Amit
EXEC emp_reader.show_next;   -- Output: 2 - Neha
EXEC emp_reader.show_next;   -- Output: 3 - Ravi
EXEC emp_reader.show_next;   -- Output: No more rows

EXEC emp_reader.close_list;
```

### Proof it stays open: opening twice fails

```sql
EXEC emp_reader.open_list;
EXEC emp_reader.open_list;
-- ORA-06511: PL/SQL: cursor already open   ❌
```

The cursor stays open until you **close it**, **reset the package**, or **log out**.

---

## Quick Summary

| Rule | What the code showed |
|---|---|
| 1. Per-session copy | Session A saw 3 while Session B saw 0 at the same time |
| 2. `RESET_PACKAGE` | Count went 2 → 0 without logout, and Neha didn't inherit Amit's value |
| 3. `SERIALLY_REUSABLE` | Separate calls always showed 0. Only inside one block did it reach 2 |
| 4. Package cursors | Each `show_next` call continued from the next row |

> **Memory hook:** Package variables remember their values between calls, but only for **your session**.
> Everyone has their own copy, and it ends at logout.
> `RESET_PACKAGE` = clear memory without logging out.
> `SERIALLY_REUSABLE` = don't remember at all, to save memory.

---

## Commonly Asked Interview Questions

### Q1. What is package state?
**Answer:** The values held by variables, constants, and cursors declared at the **package level** (in the spec or body, outside any subprogram). These values persist across multiple calls within the same session.

### Q2. How is a package variable different from a local variable in a procedure?
**Answer:** A local variable is re-initialized on **every call**. A package variable is initialized **once per session** (the first time the package is used) and keeps its value across calls until the session ends or the package is reset.

### Q3. If Session A changes a package variable, does Session B see the change?
**Answer:** **No.** Each session has its own private copy of package state, stored in its UGA. Package variables are **not** a way to share data between sessions.

### Q4. Then how would you share data across sessions?
**Answer:** Store it in a **table** (or use mechanisms like global application contexts). Package variables are session-private by design.

### Q5. How long does package state last?
**Answer:** For the **lifetime of the session**, until logout, unless it is reset with `DBMS_SESSION.RESET_PACKAGE`, the package is recompiled, or the package is `SERIALLY_REUSABLE`.

### Q6. How do you reset package state without ending the session?
**Answer:** `EXEC DBMS_SESSION.RESET_PACKAGE;` It resets **all** packages in the current session. It takes effect after the current call completes.

### Q7. Why is `RESET_PACKAGE` important in connection pooling?
**Answer:** In connection pooling, one physical database session is reused by many application users. Without resetting, the next user can inherit the previous user's leftover package values (e.g., a stored username or role), which leads to wrong results or security problems.

### Q8. What does `PRAGMA SERIALLY_REUSABLE` do?
**Answer:** It makes the package **not keep its state** across top-level calls. Package variables are re-initialized at the start of each new call, and the memory is returned to a shared pool after the call instead of being held for the whole session.

### Q9. Why would you use `SERIALLY_REUSABLE`?
**Answer:** To **reduce memory usage** when a package is called occasionally by a very large number of concurrent sessions and does not need to remember anything between calls.

### Q10. Where must `PRAGMA SERIALLY_REUSABLE` be written?
**Answer:** In the **package specification**, and also in the **package body** if the package has one.

### Q11. A `SERIALLY_REUSABLE` package's procedure is called twice inside the same `BEGIN...END` block. Does the variable keep its value between those two calls?
**Answer:** **Yes.** State lasts for the duration of one **top-level call**. Both calls are inside the same block, so the value survives. It resets once the block ends.

### Q12. Are package-level cursors part of package state?
**Answer:** **Yes.** A package-level cursor stays open and keeps its position across calls. You can fetch row 1 in one call and row 2 in the next, until it is closed, the package is reset, or the session ends.

### Q13. What error do you get if you open an already-open package cursor?
**Answer:** `ORA-06511: PL/SQL: cursor already open`. A safe pattern is to check first: `IF NOT c_emp%ISOPEN THEN OPEN c_emp; END IF;`

### Q14. What happens to a session's package state if someone recompiles the package while the session is using it?
**Answer:** The session's state is discarded, and its next call to the package raises **`ORA-04068: existing state of packages has been discarded`**. The call after that works, with state re-initialized.

### Q15. Scenario: You want to cache a country-code lookup table so it is queried only once per user login. How would you do it?
**Answer:** Declare a **package-level collection** (e.g., an associative array), fill it the first time it's needed (or in the package initialization section), and read from it on later calls. Because package state persists for the session, the table is queried only once per session.

### Q16. Scenario: A counter in a package shows a value of 5 for one user and 0 for another user at the same time. Is this a bug?
**Answer:** **No.** It is expected behavior. Each session has its own copy of the package variable, so each user sees only their own count.