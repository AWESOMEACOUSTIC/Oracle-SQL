# Oracle PL/SQL — Dynamic SQL & Security Synthesis
### Exam Study Guide

Covers: `EXECUTE IMMEDIATE` · Dynamic DDL & DML · Bind Variables for Performance & Security · Invoker's Rights (in the Dynamic-SQL/Privilege context) · Privilege Management · SQL Injection Protection Patterns · Code-Based Access Control

> This guide focuses on **Dynamic SQL** and pulls together the security topics (invoker's rights, whitelisting, injection prevention) specifically in that context — some concepts here also appear in earlier guides on FGAC/DBMS_ASSERT/ACCESSIBLE BY and on Definer/Invoker rights, but this file adds the **dynamic-SQL-specific nuances** that make this topic distinct on its own.

---

## 1. `EXECUTE IMMEDIATE`

### What it is
`EXECUTE IMMEDIATE` is PL/SQL's primary **Native Dynamic SQL (NDS)** statement — it parses and immediately executes a SQL statement or PL/SQL block whose **text is built as a string at runtime**, rather than being fixed at compile time. It's used whenever the exact statement (table name, column list, `WHERE` clause shape, or even the statement type) can't be known until the program is actually running.

### Core syntax
```sql
EXECUTE IMMEDIATE dynamic_sql_string
  [ INTO define_var1 [, define_var2 ]... ]
  [ USING [IN|OUT|IN OUT] bind_arg1 [, bind_arg2 ]... ]
  [ RETURNING INTO out_var1 [, out_var2 ]... ];
```

### What it can execute
- DDL (`CREATE`, `ALTER`, `DROP`, `GRANT`, `REVOKE`, …)
- DML (`INSERT`, `UPDATE`, `DELETE`, `MERGE`)
- A **single-row** `SELECT` (via `INTO`) — a multi-row result raises `TOO_MANY_ROWS`
- Anonymous PL/SQL blocks (including calling a stored procedure/function inside one)
- Session/system control statements (e.g., `ALTER SESSION`)

### What it *cannot* directly execute
- A **multi-row** query — for that, `OPEN cursor_variable FOR dynamic_string;` (a **REF CURSOR**) is used instead, or `EXECUTE IMMEDIATE ... BULK COLLECT INTO collection` for a one-shot multi-row fetch.

### Practical examples
```sql
-- Dynamic single-row query with a bind variable and INTO
EXECUTE IMMEDIATE 'SELECT salary FROM employees WHERE employee_id = :id'
  INTO v_salary
  USING p_emp_id;

-- Dynamic DML with RETURNING INTO
EXECUTE IMMEDIATE
  'UPDATE employees SET salary = salary * 1.1 WHERE employee_id = :id RETURNING salary INTO :new_sal'
  USING p_emp_id
  RETURNING INTO v_new_salary;

-- Dynamic anonymous block calling a procedure name built at runtime
EXECUTE IMMEDIATE 'BEGIN ' || v_proc_name || '(:x); END;' USING p_value;

-- Bulk dynamic SQL: FORALL + EXECUTE IMMEDIATE (bind array), and BULK COLLECT for fetch
FORALL i IN 1 .. v_ids.COUNT
  EXECUTE IMMEDIATE 'DELETE FROM staging_tbl WHERE id = :1' USING v_ids(i);

EXECUTE IMMEDIATE 'SELECT salary FROM employees WHERE department_id = :d'
  BULK COLLECT INTO v_salaries USING p_dept_id;
```

### Important points to remember
- Every DDL statement causes an **implicit `COMMIT`** — this is true whether the DDL is static or issued via `EXECUTE IMMEDIATE`, and it silently ends whatever transaction was already in progress. A very common exam trap: mixing dynamic DDL into the middle of a multi-step DML transaction unexpectedly commits everything done so far.
- Bind placeholders (`:id`, `:1`, etc.) can only stand in for **data values**, never for identifiers (table/column names) — identifiers must be concatenated into the string (and validated — see `DBMS_ASSERT` in Section 6).
- Single quotes inside the dynamic string must be doubled (`''`), or you can use **alternative quoting** (`q'[...]'`) to improve readability when the string itself contains many embedded quotes.
- `SQL%ROWCOUNT`, `SQL%FOUND`, etc. still work correctly after a dynamic DML statement executed via `EXECUTE IMMEDIATE`, just as with static DML.
- `EXECUTE IMMEDIATE` is generally preferred over the older, more verbose **`DBMS_SQL`** package for straightforward cases; `DBMS_SQL` remains necessary for scenarios needing a fully dynamic, unknown-at-compile-time number/type of columns (e.g., building a generic query result-set browser).

### Exercise Questions

**Q1.** Why does `EXECUTE IMMEDIATE 'SELECT * FROM employees' INTO v_rec;` fail if `employees` has more than one row?
> **A:** The `INTO` clause of `EXECUTE IMMEDIATE` expects the dynamic query to return **exactly one row** — just like a static `SELECT ... INTO` — so if the query returns multiple rows, PL/SQL raises `TOO_MANY_ROWS` at runtime. For a query that may return many rows, you must use a **REF CURSOR** (`OPEN cursor_var FOR dynamic_string;` then `FETCH`) or `BULK COLLECT INTO` a collection instead.

**Q2.** A procedure runs several `INSERT` statements, then issues a dynamic `CREATE TABLE` via `EXECUTE IMMEDIATE`, then runs more `INSERT` statements, and finally issues an explicit `ROLLBACK`. What happens to the first batch of inserts?
> **A:** They are **not rolled back**. Because DDL statements (even when issued dynamically) cause an implicit `COMMIT`, the `CREATE TABLE` statement commits everything done in the transaction up to that point — including the first batch of inserts. The final `ROLLBACK` only undoes the second batch of inserts issued *after* the DDL statement.

**Q3.** What's the difference between using `USING` and `INTO`/`BULK COLLECT INTO` in an `EXECUTE IMMEDIATE` statement?
> **A:** `USING` supplies **bind values going INTO the dynamic statement** (input, or input/output for `IN OUT`/`OUT` binds) — the data the statement needs to run. `INTO`/`BULK COLLECT INTO` captures **values coming OUT of** a dynamic query's result set (a single row, or many rows respectively) into PL/SQL variables/collections. A single `EXECUTE IMMEDIATE` for a query can use both together: `USING` to pass in filter values, and `INTO`/`BULK COLLECT INTO` to receive the results.

---

## 2. Dynamic DDL & DML

### Why dynamic SQL exists for DDL — a fundamental PL/SQL restriction
**PL/SQL does not support static (compiled-in) DDL statements at all.** You cannot write `CREATE TABLE ...;` directly inside a PL/SQL block the way you can write a `SELECT` or `UPDATE`. **Any** DDL issued from within PL/SQL — `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `GRANT`, `REVOKE` — **must** go through dynamic SQL (`EXECUTE IMMEDIATE` or `DBMS_SQL`). This is one of the most fundamental, frequently tested facts about PL/SQL's relationship to DDL.

```sql
BEGIN
  CREATE TABLE temp_data (id NUMBER);   -- COMPILE ERROR: PL/SQL has no static DDL
END;
/
```
```sql
BEGIN
  EXECUTE IMMEDIATE 'CREATE TABLE temp_data (id NUMBER)';  -- correct: dynamic SQL required
END;
/
```

### Dynamic DML — when it's genuinely needed
Dynamic DML (`INSERT`/`UPDATE`/`DELETE`/`MERGE`/`SELECT` built as a string) is used when the **table name, column list, or predicate structure** must vary at runtime — e.g., a generic "process any table" utility, a configurable reporting engine, or an ETL framework that operates over a metadata-driven list of tables.

```sql
CREATE OR REPLACE PROCEDURE truncate_any_table (p_table_name IN VARCHAR2) IS
BEGIN
  EXECUTE IMMEDIATE 'TRUNCATE TABLE ' || DBMS_ASSERT.SQL_OBJECT_NAME(p_table_name);
END;
/
```
Notice `TRUNCATE` is itself DDL (like `CREATE`/`DROP`/`ALTER`), which is why it too must go through `EXECUTE IMMEDIATE`, and notice `DBMS_ASSERT.SQL_OBJECT_NAME` validating the dynamically supplied table name before it's concatenated (see Sections 6–7).

### Important points to remember
- **All DDL from PL/SQL is dynamic SQL, by definition** — there is no such thing as "static DDL in PL/SQL."
- Dynamic DML should be reserved for cases where the statement's **structure** genuinely varies at runtime — if only a **data value** varies (not the table/columns/clause shape), prefer static SQL with a bind variable; it's simpler, safer, and lets Oracle cache the execution plan without any dynamic-SQL overhead at all.
- Because dynamic DDL commits implicitly, **utility procedures that create/drop objects should not be called in the middle of a larger business transaction** that still has pending uncommitted work the caller expects to control.
- Required privileges for dynamic DDL/DML executed inside a **definer's rights** stored unit must be granted **directly** to the definer (not via a role) — covered further in Sections 4–5, since only the `PUBLIC` role remains enabled while a definer's-rights unit is executing.

### Exercise Questions

**Q1.** Can you write `DROP TABLE emp_backup;` as a plain statement directly inside a PL/SQL block?
> **A:** No. PL/SQL has **no static DDL support at all** — `DROP TABLE`, like every other DDL statement, must be issued through dynamic SQL: `EXECUTE IMMEDIATE 'DROP TABLE emp_backup';`. Attempting to write it directly causes a compile-time syntax error.

**Q2.** A developer needs a procedure that inserts a row into whichever of three possible tables the caller specifies. Should this be written as three separate static `INSERT` statements inside an `IF`/`ELSIF`, or as one dynamic `INSERT`?
> **A:** Either approach is valid, but they trade off differently: three static `INSERT` statements are simpler to secure (no identifier concatenation needed) and let Oracle use normal cached execution plans per statement, at the cost of some code duplication. A single dynamic `INSERT` avoids duplication but requires validating the dynamically supplied table name (e.g., via `DBMS_ASSERT` or an explicit whitelist check) to avoid injection risk. For a small, fixed, known set of tables (like three), the static `IF`/`ELSIF` approach is generally the safer, simpler default; dynamic SQL becomes more justified as the number of possible tables grows or is only known at runtime from configuration.

**Q3.** Why is `TRUNCATE TABLE` considered DDL rather than DML, and what practical consequence does that have inside PL/SQL?
> **A:** `TRUNCATE TABLE` is classified as DDL because, like `CREATE`/`DROP`/`ALTER`, it causes an implicit commit and cannot be rolled back the way `DELETE` (true DML) can. The practical consequence inside PL/SQL is the same restriction as any other DDL: it **cannot** be issued as a static statement and must be run via `EXECUTE IMMEDIATE`, and issuing it mid-transaction will implicitly commit any pending uncommitted work.

---

## 3. Using Bind Variables for Performance & Security

### What a bind variable is
A **bind variable** is a named or positional placeholder (`:name` or `:1`, `:2`, …) inside a SQL statement whose actual value is supplied **separately from the statement text itself**, at execution time — rather than being embedded (concatenated) directly into the SQL string.

### The performance argument — parsing costs
| Without binds (literal concatenation) | With binds |
|---|---|
| Every distinct literal value produces **different SQL text** | The SQL text stays **identical** across executions |
| Oracle cannot find a matching cached statement → **hard parse** every time (full parse, semantic check, optimization, execution plan generation) | Oracle finds the existing cached statement in the shared pool → **soft parse** (reuse the existing plan) — far cheaper |
| High-volume systems suffer **shared pool fragmentation** ("shared pool thrashing") from thousands of near-duplicate one-off statements | Shared pool stays compact and efficient; the same plan serves many executions |

```sql
-- BAD for performance (and security — see below): a new hard-parsed statement per call
EXECUTE IMMEDIATE 'SELECT salary FROM employees WHERE employee_id = ' || p_emp_id
  INTO v_salary;

-- GOOD: identical SQL text every time; only the bound VALUE changes
EXECUTE IMMEDIATE 'SELECT salary FROM employees WHERE employee_id = :id'
  INTO v_salary
  USING p_emp_id;
```

### The security argument — injection prevention
Because a bound value is passed as **pure data**, never re-parsed as part of the SQL grammar itself, it **cannot alter the structure or meaning** of the statement — this is why bind variables are the primary, foundational defense against SQL injection for literal/data values (contrast with `DBMS_ASSERT`, which protects identifiers instead — Section 6).
```sql
-- Vulnerable: if p_emp_id came from untrusted input like "1 OR 1=1", the WHERE clause's
-- logic itself is altered
EXECUTE IMMEDIATE 'SELECT salary FROM employees WHERE employee_id = ' || p_emp_id INTO v_salary;

-- Safe: whatever string value p_emp_id holds is treated strictly as a single data value
-- being compared to employee_id -- it cannot inject additional SQL logic
EXECUTE IMMEDIATE 'SELECT salary FROM employees WHERE employee_id = :id'
  INTO v_salary USING p_emp_id;
```

### Important points to remember
- Bind variables secure **data values only** — they have **no effect** on identifiers (table/column names), which must instead be validated with `DBMS_ASSERT` or an explicit whitelist if they must vary dynamically.
- The `CURSOR_SHARING` initialization parameter (`EXACT` / `FORCE` / `SIMILAR`) can make Oracle **automatically substitute literals with system-generated binds** at the database level — a stopgap for legacy applications that don't bind explicitly — but writing explicit bind variables in new code is always the correct, best-practice approach rather than relying on this parameter.
- **Bind variable "peeking"**: on the very first **hard parse** of a statement, Oracle "peeks" at the actual bound values to help choose an execution plan — this can occasionally cause a plan chosen for one set of values to be suboptimal for a very different, skewed distribution of values used later with the same cached plan. This is a more advanced point but a legitimate exam/interview topic when discussing bind variable trade-offs.
- Both **static** SQL (embedded directly in PL/SQL) and **dynamic** SQL (`EXECUTE IMMEDIATE`) should use bind variables wherever a data value varies — binding isn't a dynamic-SQL-only concept, though it's especially critical to remember in dynamic SQL since it's easy to fall back to naive string concatenation there.

### Exercise Questions

**Q1.** Why does replacing string concatenation with a bind variable typically improve performance in a high-volume OLTP application, even though the query still returns the same logical result either way?
> **A:** With concatenation, each distinct data value produces distinct SQL text, forcing Oracle to perform a full **hard parse** (parse, semantic/privilege check, plan generation) for essentially every execution — expensive and non-scalable, and it also bloats/fragments the shared pool with near-duplicate cached statements. With a bind variable, the SQL text stays identical across executions regardless of the value used, so Oracle can find and reuse (**soft parse**) the already-cached execution plan, which is dramatically cheaper at scale.

**Q2.** Does using a bind variable for the `employee_id` value in a `WHERE` clause protect against an attacker injecting a malicious table name elsewhere in the same dynamically built statement?
> **A:** No. Bind variables only protect **data values** — the placeholder mechanism has no bearing on identifiers like table or column names, which (if dynamically supplied) must instead be concatenated and separately validated, typically via `DBMS_ASSERT` functions or an explicit whitelist check, since there is no way to "bind" an identifier.

**Q3.** What is "bind variable peeking," and why is it sometimes cited as a downside of bind variables despite their general benefits?
> **A:** Bind variable peeking is Oracle's behavior of inspecting the **actual values** bound on a statement's first hard parse in order to help the optimizer choose a good execution plan for that statement text. The downside is that the plan chosen based on the *first* set of peeked values then gets reused (via soft parsing) for *all* subsequent executions with different bound values — which can be suboptimal if the data is skewed and a later execution's values would have benefited from a very different plan. This is a known, if relatively edge-case, trade-off of the bind-variable/plan-caching mechanism.

---

## 4. Invoker's Rights in the Dynamic-SQL & Privilege Context

> Builds on the Definer's Rights vs. Invoker's Rights basics (`AUTHID DEFINER`/`AUTHID CURRENT_USER`) — this section adds the specific nuances relevant to **dynamic SQL** and **privilege-checking timing**.

### The core new nuance: WHEN are privileges checked?
| Reference type | When are privileges checked? |
|---|---|
| **Direct, static PL/SQL calls** to another named unit (e.g., calling a function directly) | At **compile time**, against the unit owner's privileges (for definer's rights) |
| **Embedded SQL (DML) statements** or **dynamic SQL** referencing an external object | At **run time**, because they are effectively (re)parsed/recompiled at that moment |

This distinction directly affects **invoker's rights (`AUTHID CURRENT_USER`)** units in particular: since an invoker's rights unit's SQL/dynamic-SQL references resolve against the *current invoker* at runtime, **the invoker must actually hold the necessary privileges at the moment the statement executes** — the developer of the invoker's rights unit only needs to grant `EXECUTE` on the unit itself, not on every object it might reference internally, because that responsibility shifts to whoever calls it.

### Role behavior — a critical, frequently tested rule
- When a **definer's rights** unit is invoked, Oracle stores the caller's currently enabled roles, then switches to running with **only the `PUBLIC` role enabled** (all the caller's other roles are temporarily disabled) — for the entire duration that unit is on the call stack.
- When an **invoker's rights** unit is invoked, the currently enabled roles **do not change at all** — whatever roles were enabled for the caller's session remain enabled.
- **Practical consequence:** any privilege a definer's-rights unit needs (for either static or dynamic SQL) must be granted to the **definer directly** — granting it only via a role the definer happens to hold is **not sufficient**, since that role won't be enabled while the unit executes.

```sql
-- If hr_pkg is DEFINER'S rights and needs SELECT on some_other_schema.tbl,
-- this is INSUFFICIENT if only granted via a role:
GRANT some_role TO hr;                 -- role privilege -- NOT usable inside hr_pkg!

-- This IS sufficient -- a direct grant:
GRANT SELECT ON some_other_schema.tbl TO hr;
```

### Why this matters specifically for dynamic SQL
Because dynamic SQL statements are (re)checked for privileges **at the moment they run**, a definer's-rights unit that issues dynamic SQL referencing an object the definer only has *role-based* access to will compile just fine (no static reference to catch at compile time) but **fail at runtime with `ORA-01031: insufficient privileges`** the first time that `EXECUTE IMMEDIATE` statement actually executes — a classic, hard-to-diagnose bug because the failure only surfaces at runtime, deep inside a dynamically built string.

### Important points to remember
- **Static references in a definer's-rights unit** are validated at **compile time**; revoking the needed privilege afterward **invalidates** the compiled unit (forcing a recompile that will then fail).
- **Dynamic SQL / DML references** are validated at **run time**, every time they execute; revoking a needed privilege has **no effect on compiled state** — the object stays valid, but the *next actual execution* of that dynamic statement fails.
- Both static and dynamic references inside a definer's-rights unit still require **direct** grants (not role-based) — the "only `PUBLIC` role enabled" rule applies regardless of whether the reference is static or dynamic.
- For an **invoker's-rights** unit, its **direct, static PL/SQL calls to other named units** are still checked against the compiled owner at compile time (per Oracle's documentation), but references embedded in **DML or dynamic SQL** are checked against the **actual invoker**, at runtime — meaning the invoker's own currently enabled roles (which, unlike definer's rights, are *not* suppressed) can genuinely satisfy those runtime privilege checks.

### Exercise Questions

**Q1.** A definer's-rights package compiles successfully and its static SQL runs fine, but an `EXECUTE IMMEDIATE` statement inside it fails at runtime with `ORA-01031`, even though the DBA insists the definer "has access" to the referenced table. What is the most likely explanation?
> **A:** The definer most likely has access to that table **only through a role**, not a direct grant. Dynamic SQL privilege checks happen at **runtime**, and while a definer's-rights unit executes, only the `PUBLIC` role is enabled — any other role the definer holds (through which the table access was granted) is disabled during execution, causing the runtime privilege check on the dynamic statement to fail. The fix is to grant `SELECT` (or whatever privilege is needed) **directly** to the definer, not via the role.

**Q2.** Why does revoking a privilege used only inside a dynamic SQL statement not cause the containing package to become invalid, the way revoking a privilege used in static SQL would?
> **A:** Static SQL references are checked and locked in at **compile time** — Oracle's dependency tracking knows about them, so revoking the underlying privilege invalidates the compiled unit, forcing recompilation (which will then fail if the privilege is truly gone). A dynamic SQL statement's text isn't parsed/checked until it actually **executes at runtime** — there's no compile-time dependency to track, so revoking the privilege has no effect on the object's valid/invalid status; it simply means the **next runtime execution** of that specific dynamic statement will fail with an insufficient-privileges error.

**Q3.** For an invoker's-rights procedure, does the *developer* of that procedure need to be granted privileges on every table the procedure's dynamic SQL might reference?
> **A:** No — for an invoker's-rights unit, references embedded in dynamic SQL (and DML) are checked against the **actual invoker** at runtime, so it is the **calling user** who must hold the necessary privileges at the moment of execution, not the developer/owner of the procedure. The developer only needs to ensure the calling user is granted `EXECUTE` on the procedure itself; the responsibility for underlying object privileges shifts to whoever ends up calling it.

---

## 5. Privilege Management

### System privileges vs. object privileges
| | System privileges | Object privileges |
|---|---|---|
| Example | `CREATE TABLE`, `CREATE PROCEDURE`, `SELECT ANY TABLE`, `EXEMPT ACCESS POLICY` | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `EXECUTE`, `REFERENCES` on a *specific* object |
| Scope | Database-wide capability | A specific named schema object |
| Granted with | `GRANT <sys_priv> TO user [WITH ADMIN OPTION];` | `GRANT <obj_priv> ON object TO user [WITH GRANT OPTION];` |
| Special option | `WITH ADMIN OPTION` — grantee can grant/revoke it to/from others, and (unlike object grant-option chains) revoking it does **not** cascade to those they granted it to | `WITH GRANT OPTION` — grantee can re-grant it to others; **revoking** the object privilege from that grantee **does cascade**, revoking it from everyone they subsequently granted it to |

### `PUBLIC` grants — a security anti-pattern for sensitive objects
```sql
GRANT SELECT ON sensitive_salary_data TO PUBLIC;   -- extremely broad -- avoid for sensitive data
```
Granting to `PUBLIC` makes the privilege available to **every current and future database user** — appropriate only for genuinely universal, non-sensitive utility objects; a red flag in a security review when found on anything sensitive.

### Roles disabled in stored PL/SQL — the practical consequence for developers
As established in Section 4, **roles are not enabled inside definer's-rights PL/SQL units** (only `PUBLIC` remains enabled) — this is precisely why Oracle's own security guidelines state that **application developers should not be granted privileges only via roles** if those privileges will be exercised inside stored PL/SQL: the privileges genuinely needed by any stored unit's internal SQL (static or dynamic) must be granted **directly** to the unit's owner.

### Least-privilege practices (frequently tested principles)
- Avoid overly broad **`ANY`** system privileges (`SELECT ANY TABLE`, `DROP ANY TABLE`, etc.) except where genuinely required (e.g., true DBA roles) — they bypass normal object-level grant boundaries entirely.
- Prefer **custom, narrowly scoped roles** built for a specific job function over reusing broad built-in roles (Oracle explicitly warns that even its own predefined roles like `CONNECT`/`RESOURCE` have had their privilege sets changed across versions and shouldn't be relied on for a stable, intentional privilege set).
- **`DBMS_PRIVILEGE_CAPTURE`** (12c+) lets you **capture actual privilege usage** over a period of observation, then compare it against what's actually granted — a practical tool for identifying and revoking unused/excessive privileges as part of tightening a system toward least privilege.

### Important points to remember
- `WITH ADMIN OPTION` (system privileges/roles) vs. `WITH GRANT OPTION` (object privileges) look similar but have a **key difference in revoke behavior**: object-privilege grant chains cascade on revoke; admin-option grant chains for system privileges/roles generally do **not** automatically cascade the same way — a subtle, testable distinction.
- Because roles are suppressed inside definer's-rights units, this is the primary reason **why `DBMS_ASSERT`, dynamic SQL, and other privilege-sensitive PL/SQL code must be paired with direct grants**, tying this section directly back to Sections 2 and 4.
- Column-level object privileges are possible for a subset of privilege types (e.g., `GRANT UPDATE (salary) ON employees TO hr_clerk;`) — a finer-grained alternative to granting the privilege on the whole table.

### Exercise Questions

**Q1.** A DBA grants `SELECT` on a table to `USER_A`, `USER_A` (having received it `WITH GRANT OPTION`) grants it onward to `USER_B`. The DBA later revokes the original grant from `USER_A`. What happens to `USER_B`'s access?
> **A:** `USER_B`'s `SELECT` access is also **revoked** — object privilege grants made `WITH GRANT OPTION` form a dependency chain, and revoking the privilege from the original grantee **cascades** to everyone downstream who received it through that chain, however many levels deep.

**Q2.** Why do Oracle's security guidelines specifically advise against granting privileges to application developers **only through roles**, if those privileges will be used inside their stored PL/SQL code?
> **A:** Because roles (other than `PUBLIC`) are **disabled** while a definer's-rights PL/SQL unit executes — any privilege the developer holds solely via a role would be unavailable to their own compiled code at compile time (for static references) and at run time (for dynamic SQL/DML references), causing compile or runtime privilege errors despite the developer "having" the privilege in their own interactive session. The privileges the code itself needs must be granted **directly** to the unit's owner.

**Q3.** What practical problem does `DBMS_PRIVILEGE_CAPTURE` help solve, and why is it valuable for a least-privilege security posture?
> **A:** It lets administrators **observe actual privilege usage** by a user, role, or the whole database over a defined capture period, then compare that observed usage against everything that's actually **granted**. This surfaces privileges that are granted but never actually exercised — candidates for safe revocation — helping an organization progressively tighten access down toward only what's genuinely needed (the least-privilege principle), rather than guessing at what might be safe to revoke.

---

## 6. SQL Injection Protection Patterns (Synthesis)

> This section ties together the individual defenses covered across this study series into one integrated **defense-in-depth checklist** for any code that builds dynamic SQL.

### The layered defense model
| Layer | Technique | Protects against |
|---|---|---|
| 1 | **Bind variables** (`USING`, `:placeholder`) | Injection via **data values** |
| 2 | **`DBMS_ASSERT`** (`SQL_OBJECT_NAME`, `SIMPLE_SQL_NAME`, `ENQUOTE_NAME`, `ENQUOTE_LITERAL`) | Injection via dynamically supplied **identifiers** (table/column names) that can't be bound |
| 3 | **Explicit whitelisting** (e.g., `IF p_table IN ('EMP','DEPT') THEN ... ELSE raise_application_error ... END IF;`) | Even a *technically valid, existing* object that the caller shouldn't be allowed to target through this code path (`DBMS_ASSERT` alone only proves existence/accessibility, not *intent*) |
| 4 | **Least privilege** (direct grants scoped tightly; avoid `ANY` privileges) | Limits the *damage* even if an injection attempt partially succeeds |
| 5 | **Invoker's rights vs. definer's rights choice** | Limits *whose* privileges an exploited call actually runs with |
| 6 | **Code-Based Access Control (`ACCESSIBLE BY`)** | Limits *which callers* can even reach sensitive internal units in the first place |
| 7 | **Avoid dynamic SQL where static suffices** | Removes the injection surface entirely for anything that doesn't genuinely need to vary at runtime |

### Worked example — before and after, combining multiple layers
**Vulnerable:**
```sql
PROCEDURE update_value (p_table IN VARCHAR2, p_col IN VARCHAR2, p_id IN NUMBER, p_val IN VARCHAR2)
IS
BEGIN
  EXECUTE IMMEDIATE
    'UPDATE ' || p_table || ' SET ' || p_col || ' = ''' || p_val || ''' WHERE id = ' || p_id;
END;
```
Every one of `p_table`, `p_col`, `p_val`, and `p_id` here is concatenated raw — a textbook injection surface on all four inputs.

**Hardened (layers 1, 2, 3, 4 applied):**
```sql
PROCEDURE update_value (p_table IN VARCHAR2, p_col IN VARCHAR2, p_id IN NUMBER, p_val IN VARCHAR2)
IS
  v_table VARCHAR2(30);
  v_col   VARCHAR2(30);
BEGIN
  -- Layer 3: explicit whitelist of allowed tables (existence isn't enough -- must be INTENDED)
  IF p_table NOT IN ('EMPLOYEES', 'DEPARTMENTS') THEN
    RAISE_APPLICATION_ERROR(-20001, 'Table not permitted for this operation.');
  END IF;

  -- Layer 2: validate identifiers that must be concatenated
  v_table := DBMS_ASSERT.SQL_OBJECT_NAME(p_table);
  v_col   := DBMS_ASSERT.SIMPLE_SQL_NAME(p_col);

  -- Layer 1: bind the actual DATA values -- never concatenated
  EXECUTE IMMEDIATE
    'UPDATE ' || v_table || ' SET ' || v_col || ' = :val WHERE id = :id'
    USING p_val, p_id;
END;
-- Layer 4 (separately, at the DBA level): grant only UPDATE on these two specific
-- tables/columns directly to this procedure's owner -- no ANY privileges.
```

### Important points to remember
- **No single technique is a complete solution on its own** — bind variables don't help with identifiers; `DBMS_ASSERT` doesn't help with data values or confirm business-appropriate *intent*; whitelisting is only as good as the list itself. Real protection comes from **combining** the layers relevant to what your specific dynamic SQL actually concatenates.
- The single most important habit: **ask, for every piece of a dynamic SQL string, "is this a data value or an identifier, and where did it come from?"** — data values get bound; identifiers get validated (and ideally whitelisted); anything from an untrusted source gets extra scrutiny regardless.
- Prefer **not writing dynamic SQL at all** for anything that doesn't genuinely need to vary at runtime — the most secure dynamic SQL is the dynamic SQL you didn't need to write.
- Logging/auditing the actual dynamic SQL text generated (in a controlled way, without logging sensitive data) can help detect anomalous or unexpected statement shapes in production as an additional monitoring layer, though this is a detective control, not a preventive one.

### Exercise Questions

**Q1.** A developer uses `DBMS_ASSERT.SQL_OBJECT_NAME` to validate a dynamically supplied table name and considers the code fully secured against injection. What gap remains, and how is it typically closed?
> **A:** `SQL_OBJECT_NAME` only confirms that the supplied string names a **real, existing, accessible** object — it says nothing about whether that particular object is one this code path is actually **supposed** to operate on. A caller with legitimate access to some unrelated but sensitive table could still pass its name and have it validated successfully. The gap is closed by adding an **explicit whitelist check** (e.g., an `IF ... IN (...)` list of the specific tables this operation is meant to support) in addition to the `DBMS_ASSERT` call.

**Q2.** In the hardened example above, why are `p_val` and `p_id` handled with `USING` (bind variables) while `p_table` and `p_col` are handled with `DBMS_ASSERT` instead?
> **A:** `p_val` and `p_id` are **data values** being compared/assigned in the statement — they can be passed as bind variables, which is the strongest and simplest protection available for values. `p_table` and `p_col` are **identifiers** (a table name and a column name) — identifiers can never be bound in SQL, so they must be concatenated into the statement text, which is exactly why they require separate validation (via `DBMS_ASSERT` and/or an explicit whitelist) before being concatenated.

**Q3.** Why is "avoid dynamic SQL where static SQL suffices" listed as a legitimate defense-in-depth layer, rather than just a general coding-style tip?
> **A:** Every additional piece of dynamically constructed SQL is another potential injection surface that has to be correctly defended (bound values, validated identifiers, whitelists, etc.) — and every defense can, in principle, be misapplied or forgotten. Static SQL, by contrast, has **no injection surface at all** for its fixed structure, since the statement's shape is fixed at compile time and only data values (already protected via normal bind variables) can vary. Minimizing the amount of dynamic SQL in a codebase directly minimizes the amount of code that needs this defense-in-depth treatment in the first place.

---

## 7. Code-Based Access Control (Whitelisting) — the Dynamic-SQL Interaction

> Builds on the `ACCESSIBLE BY` basics from the earlier Security guide — this section adds a **critical, frequently misunderstood nuance**: how whitelisting interacts with dynamic SQL calls.

### The key rule: `ACCESSIBLE BY` only protects DIRECT, STATIC calls
Oracle's documentation is explicit and important here: **the `ACCESSIBLE BY` check only allows access when the call is direct** — a call made through **static SQL, `DBMS_SQL`, or dynamic SQL (`EXECUTE IMMEDIATE`) is checked and will FAIL**, even if it originates from a unit that *is* listed in the accessor whitelist.

```sql
CREATE OR REPLACE PACKAGE protected_pkg
  ACCESSIBLE BY (PACKAGE trusted_caller_pkg)
IS
  PROCEDURE sensitive_op;
END protected_pkg;
/
```
```sql
-- Inside TRUSTED_CALLER_PKG (which IS whitelisted):

-- DIRECT call: SUCCEEDS -- this is exactly what ACCESSIBLE BY is designed to allow
BEGIN
  protected_pkg.sensitive_op;
END;

-- DYNAMIC call from the SAME whitelisted package: STILL FAILS
BEGIN
  EXECUTE IMMEDIATE 'BEGIN protected_pkg.sensitive_op; END;';
  -- Fails -- ACCESSIBLE BY does not recognize calls made via dynamic SQL as "direct",
  -- REGARDLESS of which unit issued the EXECUTE IMMEDIATE.
END;
```

### Why this matters
This is a **security feature, not a limitation to work around** — it closes off a potential bypass technique: without this rule, an attacker who could get *any* code (even unrelated, untrusted code) to execute inside a whitelisted caller's context via dynamic SQL construction might otherwise reach a protected unit indirectly. By making the check fail for **any** non-direct access path, Oracle ensures whitelisting can't be routed around through dynamic SQL, `DBMS_SQL`, or even conditional-compilation-driven code paths (which are documented to fail the check as well).

### Other documented edge cases worth knowing
- **RPC (remote procedure calls)** to a protected unit **always fail** — there's no context available at either compile time or run time to validate the caller against the accessor list across a remote call.
- A call to a package's **initialization block** is checked against the **package specification's** accessor list — i.e., whatever units are allowed to trigger the package's first reference (and thus its init block) are governed by the same whitelist as the package itself.
- A unit can **always** call itself/its own members — no whitelist entry is needed for that.
- Best practice (from Oracle's own documentation) is to **explicitly specify the `unit_kind`** (`PACKAGE`, `PROCEDURE`, `FUNCTION`, `TRIGGER`, `TYPE`) in the accessor list, since two different kinds of units can share the same name, creating ambiguity otherwise.

### Important points to remember
- **If your design relies on a whitelisted unit calling a protected unit via dynamic SQL (e.g., because the target name is built at runtime), `ACCESSIBLE BY` will not permit it** — this is a design constraint to plan around, not a bug to debug around. If dynamic invocation of a protected unit is genuinely required, the whitelisting model isn't the right tool for that specific call path; you'd need a different control (e.g., an explicit privilege/role check written into the protected unit itself).
- This interacts directly with Section 6's advice to **minimize dynamic SQL**: code protected by `ACCESSIBLE BY` should be invoked directly and statically by its whitelisted callers wherever possible, both for this compatibility reason and for injection-surface reasons generally.
- `ACCESSIBLE BY` and privilege grants (`EXECUTE`) are **independent, complementary controls** — a caller might have `EXECUTE` privilege on a protected unit yet still be denied by `ACCESSIBLE BY` if they're not on the whitelist and/or if their otherwise-valid call is routed through a disallowed path (dynamic SQL, `DBMS_SQL`, RPC).

### Exercise Questions

**Q1.** `pkg_a` is listed in `protected_pkg`'s `ACCESSIBLE BY` clause. A developer inside `pkg_a` writes `EXECUTE IMMEDIATE 'BEGIN protected_pkg.sensitive_op; END;';` instead of calling it directly. Will this succeed?
> **A:** No. `ACCESSIBLE BY` only permits **direct** calls — access attempted through dynamic SQL fails the check regardless of whether the calling unit is on the whitelist. Even though `pkg_a` is a legitimate, whitelisted accessor, routing the call through `EXECUTE IMMEDIATE` causes Oracle to deny it. The developer must call `protected_pkg.sensitive_op;` directly (statically) from within `pkg_a`'s code for the whitelist to apply successfully.

**Q2.** Why does Oracle deliberately fail `ACCESSIBLE BY` checks for dynamic SQL and `DBMS_SQL` calls, rather than simply checking whether the *originating* compiled unit is on the whitelist?
> **A:** Dynamic SQL statements are built and parsed at **runtime**, and their content could, in principle, be influenced by data, configuration, or (in a compromised scenario) injected input rather than being fixed, auditable, compiled-in code — so there's no reliable compile-time or even fully trustworthy run-time context to validate "this call is genuinely coming from the whitelisted unit's own intended logic" the way there is for a direct static call. Failing the check unconditionally for these indirect paths closes off a potential avenue where an attacker who can influence a dynamically-built string might otherwise attempt to reach a protected unit through a nominally whitelisted caller.

**Q3.** A team wants to protect `internal_calc_pkg` so only `public_api_pkg` can call it, but `public_api_pkg` needs to invoke it dynamically because the exact function name varies based on a configuration value. Is `ACCESSIBLE BY` the right tool here, and if not, what should they use instead?
> **A:** `ACCESSIBLE BY` is **not** sufficient here on its own, since the dynamic invocation will fail the whitelist check even from the legitimate caller. Given the genuine need for dynamic dispatch, the team should instead (or additionally) implement an explicit **runtime authorization check inside `internal_calc_pkg` itself** — e.g., verifying `SYS_CONTEXT`/caller identity, or requiring a validated token/parameter passed only by `public_api_pkg` — combined with the other defense-in-depth layers from Section 6 (whitelisting the *set* of valid dynamically-built function names, using bind variables for any data values, and applying least-privilege grants), rather than relying on `ACCESSIBLE BY` to enforce the caller restriction for this particular access path.

---

## Quick Cross-Topic Summary Table

| Topic | Key Mechanism | Primary Purpose |
|---|---|---|
| `EXECUTE IMMEDIATE` | Native Dynamic SQL statement | Build & run SQL/PL·SQL whose text is only known at runtime |
| Dynamic DDL & DML | Mandatory for ALL DDL from PL/SQL; DML when structure varies | Enables runtime-determined schema/data operations |
| Bind Variables | `USING`, `:placeholder` | Performance (soft parse reuse) + security (data-value injection defense) |
| Invoker's Rights (dynamic-SQL angle) | `AUTHID CURRENT_USER`; runtime privilege checks; only `PUBLIC` role enabled for definer's rights | Determines whose privileges/roles govern dynamic SQL execution |
| Privilege Management | System vs. object privileges, direct grants vs. roles, `DBMS_PRIVILEGE_CAPTURE` | Least-privilege enforcement, especially for stored PL/SQL |
| SQL Injection Protection Patterns | Layered: binds + `DBMS_ASSERT` + whitelisting + least privilege + `ACCESSIBLE BY` | Defense-in-depth for any dynamic SQL |
| Code-Based Access Control | `ACCESSIBLE BY` — direct/static calls only | Restrict which specific units may call a sensitive unit |