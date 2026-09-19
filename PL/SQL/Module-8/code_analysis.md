# Oracle PL/SQL — Security, Code Analysis & New Features
### Exam Study Guide

Covers: DBMS_RLS (FGAC/VPD) · Application Context · DBMS_ASSERT · Code-Based Access Control (Whitelisting) & Wrapping · Conditional Compilation & Warnings · PL/Scope & UTL_CALL_STACK · Edition-Based Redefinition & Invisible Columns

---

## 1. Fine-Grained Access Control — `DBMS_RLS` (Virtual Private Database / VPD)

### What it is
Fine-Grained Access Control (FGAC), commonly called **Virtual Private Database (VPD)**, lets the *database itself* attach a security predicate (a `WHERE` clause fragment) to every SQL statement issued against a table, view, or synonym — **regardless of the tool used** (SQL*Plus, a report writer, an ad-hoc query, an application). This is its key advantage over view-based security, which can be bypassed by querying the base table directly.

The predicate is generated at runtime by a **PL/SQL policy function** you write; Oracle transparently rewrites the query to include it.

### Core package & procedures
| Procedure | Purpose |
|---|---|
| `DBMS_RLS.ADD_POLICY` | Attaches a policy (and its function) to a table/view |
| `DBMS_RLS.DROP_POLICY` | Removes a policy |
| `DBMS_RLS.ENABLE_POLICY` | Enables/disables an existing policy |
| `DBMS_RLS.REFRESH_POLICY` | Forces re-parsing of cached cursors using the policy |
| `DBMS_RLS.CREATE_POLICY_GROUP` / `ADD_GROUPED_POLICY` | Manage multiple policies grouped by application context (multi-tenant / driving-context scenarios) |

### Policy types (important exam distinction)
| Type | Behavior |
|---|---|
| `STATIC` (default) | Predicate is the same for every user/query — computed **once** and cached in SGA. Fastest. |
| `SHARED_STATIC` | Same as static, but shared across multiple objects that use an identical policy function. |
| `CONTEXT_SENSITIVE` | Predicate can change between executions **if the session context changes**; re-evaluated when `SYS_CONTEXT` values change, not on every parse. |
| `SHARED_CONTEXT_SENSITIVE` | Context-sensitive + shared across objects. |
| `DYNAMIC` | Predicate is re-evaluated on **every single execution** — most flexible, most expensive. |

### The Policy Function — required signature
```sql
CREATE OR REPLACE FUNCTION emp_sec_policy (
  p_schema  IN VARCHAR2,
  p_object  IN VARCHAR2
) RETURN VARCHAR2
IS
BEGIN
  RETURN 'department_id = SYS_CONTEXT(''USERENV'',''SESSION_USER'')'; -- illustrative
END;
/
```
It **must** take exactly these two `VARCHAR2` parameters (schema, object name) and return a `VARCHAR2` predicate string. It must be a **pure function** — no DML, no side effects — because it may be called implicitly during query parsing.

### Practical example
```sql
-- 1. Policy function: a manager sees only rows for their own department
CREATE OR REPLACE FUNCTION dept_filter (schema_p VARCHAR2, obj_p VARCHAR2)
RETURN VARCHAR2
IS
BEGIN
  IF SYS_CONTEXT('USERENV','SESSION_USER') = 'HR_ADMIN' THEN
    RETURN NULL;                     -- NULL predicate = no restriction
  ELSE
    RETURN 'dept_id = SYS_CONTEXT(''HR_CTX'',''DEPT_ID'')';
  END IF;
END;
/

-- 2. Attach the policy
BEGIN
  DBMS_RLS.ADD_POLICY (
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    policy_name     => 'EMP_DEPT_POLICY',
    function_schema => 'HR',
    policy_function => 'DEPT_FILTER',
    statement_types => 'SELECT,UPDATE,DELETE',
    policy_type     => DBMS_RLS.CONTEXT_SENSITIVE
  );
END;
/

-- 3. Now, transparently:
SELECT * FROM employees;
-- is rewritten (for a non-admin) to:
-- SELECT * FROM employees WHERE dept_id = SYS_CONTEXT('HR_CTX','DEPT_ID');
```

**Column-level VPD** — restrict the policy so it fires only when specific *sensitive* columns are referenced:
```sql
DBMS_RLS.ADD_POLICY(
  object_schema      => 'HR', object_name => 'EMPLOYEES',
  policy_name        => 'SALARY_POLICY', function_schema => 'HR',
  policy_function     => 'SALARY_FILTER',
  sec_relevant_cols   => 'SALARY',
  sec_relevant_cols_opt => DBMS_RLS.ALL_ROWS  -- return all rows but NULL the salary column,
                                                -- instead of hiding the whole row
);
```

### Important points to remember
- The system privilege **`EXEMPT ACCESS POLICY`** bypasses **all** VPD policies for a user (SYS is always exempt). Grant it very sparingly — it's an easy point in "spot the security hole" questions.
- Policy functions **cannot perform DML** and should avoid raising unhandled exceptions (an exception in the function blocks the *entire* query).
- A `NULL` return from the policy function means "no restriction added."
- `STATIC` policies are cached and reused — cheapest but least flexible; `DYNAMIC` is re-run every execution — most flexible, highest overhead. **`CONTEXT_SENSITIVE` is the usual "best of both worlds" exam answer** because it's cached but responds to context changes.
- VPD predicates apply to `SELECT`, `INSERT`, `UPDATE`, `DELETE` — controlled individually via `statement_types`.
- VPD is the base technology also used by **Oracle Label Security** and **DBMS_REDACT** for column masking.
- Works well combined with **Application Context** (Section 2) so the predicate doesn't need to re-query a lookup table each time.

### Exercise Questions

**Q1.** Why is VPD (DBMS_RLS) considered more secure than restricting access purely through views?
> **A:** Views can be bypassed if a user (or a report tool) is granted or can query the underlying base table directly, or uses a different access path. A VPD policy is enforced by the database kernel at the object level for *any* access path — SQL*Plus, JDBC, ad-hoc tools — so it cannot be routed around the way a view can.

**Q2.** A developer writes a static policy function that performs a `SELECT` against a lookup table to build the predicate string. What's wrong with this design, and what would fix it?
> **A:** Nothing prevents a policy function from querying a table for the predicate value itself — but it's inefficient, because a `STATIC` policy is cached and only re-executed rarely, so lookup changes may not be reflected without `REFRESH_POLICY`, and repeated re-parsing for a context-sensitive/dynamic policy would query the table on every execution, hurting performance. The idiomatic fix is to load the required value into a session-level **application context** attribute once (via a trusted package/logon trigger) and have the policy function read it cheaply with `SYS_CONTEXT`, rather than re-querying a table.

**Q3.** What is the effect of granting `EXEMPT ACCESS POLICY` to a user?
> **A:** That user's sessions bypass **every** VPD policy in the database (unless the grant is object-specific in later releases with fine-grained exemptions) — they see unfiltered data. It should be treated like a highly privileged grant, reserved for trusted administrative accounts only.

---

## 2. Application Context Security

### What it is
An **application context** is a secure, session-scoped (or globally-scoped) namespace of attribute/value pairs that PL/SQL code and SQL (via `SYS_CONTEXT`) can read. Contexts are the standard mechanism for storing "who is this user, really, and what are they allowed to see" information cheaply and securely — and are the standard partner technology for VPD policy functions.

### Why it's "secure"
```sql
CREATE CONTEXT hr_ctx USING hr_ctx_pkg;
```
The `USING <package>` clause means the namespace `HR_CTX` can **only be set by code inside `HR_CTX_PKG`** (via `DBMS_SESSION.SET_CONTEXT`). No other session, ad-hoc SQL, or malicious client-side code can forge or overwrite the attribute values — this is the entire point: the context is trusted because only a specific, audited, server-side PL/SQL package is allowed to populate it.

### Setting and reading values
```sql
-- Inside HR_CTX_PKG (the ONLY code allowed to set this namespace)
PROCEDURE set_dept_context IS
  v_dept employees.department_id%TYPE;
BEGIN
  SELECT department_id INTO v_dept
  FROM   employees
  WHERE  username = SYS_CONTEXT('USERENV','SESSION_USER');

  DBMS_SESSION.SET_CONTEXT('HR_CTX', 'DEPT_ID', v_dept);
END;

-- Anywhere in the session, reading is unrestricted:
SELECT SYS_CONTEXT('HR_CTX','DEPT_ID') FROM dual;
```
Typically `set_dept_context` is invoked from an **`AFTER LOGON` database trigger** so context is populated as soon as the user connects.

### Built-in `USERENV` namespace
Oracle ships a predefined, always-available context called `USERENV`, populated automatically by the database — e.g. `SYS_CONTEXT('USERENV','SESSION_USER')`, `'IP_ADDRESS'`, `'CURRENT_SCHEMA'`, `'CLIENT_IDENTIFIER'`, `'DB_NAME'`. **You cannot set `USERENV` values yourself** — they are maintained by Oracle.

### Global application context
```sql
CREATE CONTEXT app_ctx USING app_ctx_pkg ACCESSED GLOBALLY;
```
Used in **connection-pooled / multi-tier applications**, where the database session is shared by many end users through a middle tier. The middle tier calls `DBMS_SESSION.SET_IDENTIFIER('end_user_id')` to tag the current lightweight session, and the global context associates attribute values with that *client identifier* rather than the (shared) physical database session — so the correct end-user's row-level restrictions still apply even though many end users share the same physical DB connection.

### Important points to remember
- `CLIENT_IDENTIFIER` (set via `DBMS_SESSION.SET_IDENTIFIER`) is **client-supplied and not trustworthy on its own** — never use it *directly* as a security value. It's safe to use only as a *lookup key* into server-validated data (e.g., the app package looks up the real user's privileges based on the client identifier and stores the validated result in a global context).
- The context namespace and its `USING` package do **not** have to be created by the same schema that owns the protected table, but the package must exist and be a genuinely trusted piece of code — review it carefully.
- Contexts are stored in memory for the session (PGA/SGA for global contexts), making `SYS_CONTEXT` lookups **much faster** than repeatedly querying a table — a major performance argument for using them with VPD policy functions.
- A context can hold multiple named attributes (like a small in-memory associative structure): `SET_CONTEXT(namespace, attribute, value)`.
- `DBMS_SESSION.CLEAR_CONTEXT` / `CLEAR_ALL_CONTEXT` release values — important to reset in connection-pooled scenarios between different end users.

### Exercise Questions

**Q1.** Why does creating a context `USING` a package make it more secure than simply letting any session call `DBMS_SESSION.SET_CONTEXT` directly?
> **A:** Without the `USING` clause restriction, any session with execute privilege on `DBMS_SESSION` could set arbitrary values into the namespace and effectively self-grant access (e.g., set their own `DEPT_ID` to see everyone's data). Binding the namespace to a specific trusted package means the attribute values can only originate from vetted, server-side logic — typically logic that authenticates the value against a real table — closing that loophole.

**Q2.** A developer uses `SYS_CONTEXT('USERENV','CLIENT_IDENTIFIER')` directly inside a VPD policy function to restrict rows to "the current end user" in a connection-pooled web app. Is this secure? Why or why not?
> **A:** No. `CLIENT_IDENTIFIER` is set by the *client/middle-tier* via `DBMS_SESSION.SET_IDENTIFIER` and is not independently validated by the database — a malicious or buggy client could set any identifier it wants. It should only be used as a key to look up validated data (typically populated once into a trusted global application context by server-side code), not consumed directly as an authorization value.

**Q3.** What's the performance argument for application context over a lookup-table query inside a VPD policy function?
> **A:** `SYS_CONTEXT` reads a value already resident in session (or shared) memory — essentially free — whereas a table lookup inside the policy function means an extra query (and potential recursive VPD evaluation) on every parse or execution, adding real overhead, especially for `DYNAMIC` policies evaluated on every execution.

---

## 3. SQL Injection Prevention — `DBMS_ASSERT`

### What it is
`DBMS_ASSERT` is a built-in package that **validates** strings that must be concatenated into dynamic SQL when a **bind variable cannot be used** — most commonly identifiers (table names, column names, schema names) that are supplied dynamically, e.g. for generic reporting procedures where the table to query is passed as a parameter.

> **Core exam point:** Bind variables remain the *primary* defense against SQL injection for **data values**. `DBMS_ASSERT` exists for the cases bind variables cannot cover — **object/identifier names**, which can never be bound.

### Key functions
| Function | Purpose | Raises on failure |
|---|---|---|
| `ENQUOTE_LITERAL(str)` | Wraps a string as a safe SQL literal, doubling embedded quotes | — |
| `ENQUOTE_NAME(str, capitalize)` | Double-quotes an identifier, escaping embedded quotes (default capitalizes) | — |
| `NOOP(str)` | Returns input unchanged — used historically just to enforce `VARCHAR2` typing/length; provides no real validation | — |
| `SCHEMA_NAME(str)` | Confirms input is an **existing** schema name | `ORA-44001` if invalid |
| `SQL_OBJECT_NAME(str)` | Confirms input is a qualified name of an **existing** object the caller can access | `ORA-44002`/related if invalid |
| `SIMPLE_SQL_NAME(str)` | Confirms input is **lexically** a legal simple SQL identifier (does *not* check existence) | error if syntax invalid |
| `QUALIFIED_SQL_NAME(str)` | Confirms input is **lexically** a legal `schema.object`-style qualified name (does *not* check existence) | error if syntax invalid |

### Practical example — vulnerable vs. safe

**Vulnerable (classic injection via identifier):**
```sql
PROCEDURE list_rows (p_table IN VARCHAR2) IS
  v_sql VARCHAR2(200);
BEGIN
  v_sql := 'SELECT * FROM ' || p_table;   -- p_table could be
  EXECUTE IMMEDIATE v_sql;                -- 'EMP; DROP TABLE X--'
END;
```

**Safe, using `DBMS_ASSERT`:**
```sql
PROCEDURE list_rows (p_table IN VARCHAR2) IS
  v_sql VARCHAR2(200);
BEGIN
  v_sql := 'SELECT * FROM ' || DBMS_ASSERT.SQL_OBJECT_NAME(p_table);
  EXECUTE IMMEDIATE v_sql;
  -- If p_table is not a real, accessible object, SQL_OBJECT_NAME raises
  -- an exception BEFORE the string is ever executed.
END;
```

**Dynamic `ORDER BY` column, safely:**
```sql
v_sql := 'SELECT * FROM employees ORDER BY '
         || DBMS_ASSERT.SIMPLE_SQL_NAME(p_sort_column);
```

### Important points to remember
- `DBMS_ASSERT` protects **identifiers**, not data literal values — never rely on it in place of a bind variable for `WHERE` clause values.
- `SQL_OBJECT_NAME`/`SCHEMA_NAME` check *existence and accessibility*, not *intent* — a determined attacker who has legitimate access to some other object they shouldn't be querying via this code path could still pass it and get past the assertion. Combine with an explicit whitelist (e.g., checking against a known list of allowed table names) for maximum safety in high-risk code.
- All functions raise exceptions on invalid input — always call them, **don't just trust that no exception means "safe," but do let the exception propagate/abort the operation** rather than catching and ignoring it.
- This package was introduced specifically because dynamic SQL that assembles identifiers is otherwise very hard to defend, since Oracle offers no native way to "bind" a table or column name.

### Exercise Questions

**Q1.** Why can't you simply use a bind variable to prevent injection when a table name is supplied dynamically?
> **A:** Bind variables can only substitute for **literal values** in a SQL statement (things that appear where a constant would go), not for **identifiers** such as table or column names, which must be part of the parsed SQL text itself. Since the table name has to be concatenated into the statement, `DBMS_ASSERT` is used to validate that the concatenated text really is a legitimate object name before it becomes part of executable SQL.

**Q2.** What's the practical difference between `SQL_OBJECT_NAME` and `SIMPLE_SQL_NAME`?
> **A:** `SQL_OBJECT_NAME` checks that the string names an object that **actually exists** and that the calling session can access — a semantic/existence check. `SIMPLE_SQL_NAME` only checks that the string is **lexically valid** as an unquoted SQL identifier (correct characters, doesn't start with a digit, etc.) — it says nothing about whether such an object exists. Use `SQL_OBJECT_NAME` when you want to guarantee the target is real and accessible; use `SIMPLE_SQL_NAME` for things like column aliases or `ORDER BY` targets that may not correspond to a standalone database object.

**Q3.** A junior developer says: "We call `DBMS_ASSERT.SQL_OBJECT_NAME` on every input, so our dynamic SQL is now 100% safe from injection." Evaluate this claim.
> **A:** Overstated. `DBMS_ASSERT` reduces risk for identifier-based injection but is not a complete solution: (a) it only validates that the identifier is real and accessible, not that it's the *intended* table for that business operation (privilege misuse is still possible if the caller can legitimately access an unintended object); (b) any data literal values in the same dynamic SQL statement must still be handled via bind variables, not `DBMS_ASSERT`; (c) defense-in-depth (least privilege, explicit whitelists, code review) is still recommended.

---

## 4. Code-Based Access Control (Whitelisting) & Code Protection

> Exam terminology note: Oracle's official term for this 12c feature is **Code-Based Access Control (CBAC)**, implemented via the **`ACCESSIBLE BY`** clause — commonly nicknamed "whitelisting." PL/SQL does not have a cryptographic "digital code-signing" feature in the sense of signed certificates; syllabi that say "code signing and white-listing" are referring to this whitelisting mechanism, typically paired with the separate **source-wrapping** (obfuscation) utility for protecting source-code confidentiality. Both are covered below since exams often pair them.

### 4a. Whitelisting — the `ACCESSIBLE BY` clause (12.1, enhanced in 12.2)

**Problem it solves:** By default, if schema `A` owns packages `P1…Pn`, *any* of those packages can call *any other* — there's no way to say "only `P1` is allowed to call `P2`." If an attacker finds a SQL-injection flaw anywhere in the schema (or in the calling application), they can potentially reach *any* procedure in that schema. Whitelisting shrinks that blast radius.

**Syntax (package/procedure/function/type spec):**
```sql
CREATE OR REPLACE PACKAGE pay_internal
  ACCESSIBLE BY (PACKAGE pay_api)
IS
  PROCEDURE apply_raise (p_emp_id NUMBER, p_pct NUMBER);
END pay_internal;
/
```
Now `pay_internal` can **only** be invoked from inside `pay_api` (or from `pay_internal` itself). Any other caller — including a top-level anonymous block, another schema's code, or a SQL-injected call — fails with:
```
PLS-00904: insufficient privilege to access object PAY_INTERNAL
```

**12.2 enhancements:**
1. You can whitelist an **individual subprogram inside a package** (fine-tuning below the whole-package level).
2. You can specify the **unit kind** of the whitelisted caller (e.g. `PACKAGE`, `PROCEDURE`, `FUNCTION`, `TRIGGER`, `TYPE`, `VIEW`) for a more precise whitelist entry.

### Important points to remember
- The whitelisted unit is **always accessible to itself** — you don't need to list it in its own clause.
- Referencing a unit in another schema requires the **fully qualified name** in the `ACCESSIBLE BY` list, plus the normal `EXECUTE`/object grants.
- This directly mitigates SQL injection risk: even if an attacker manages to inject a call, calling a non-whitelisted internal unit from the "wrong" place will fail at the PL/SQL engine level — independent of object privileges.
- Supports the **principle of least privilege**: expose a thin, audited public API package (whitelisted caller) while hiding sensitive internal logic packages behind `ACCESSIBLE BY`.

### 4b. Source protection — Wrapping (`DBMS_DDL.WRAP` / the `wrap` utility)
```sql
-- Command-line utility (obfuscates a .sql source file):
$ wrap iname=my_package.sql oname=my_package.plb
```
or programmatically:
```sql
v_wrapped := DBMS_DDL.WRAP('CREATE OR REPLACE PACKAGE BODY ... END;');
```
Wrapping converts readable PL/SQL source into an obfuscated, non-human-readable representation before shipping it to customers/other environments — protecting intellectual property (hiding *how* the code works), **not** a control against SQL injection or a substitute for `ACCESSIBLE BY`/privilege management. It is reversible in principle (it's obfuscation, not encryption), so it should not be treated as a strong security boundary.

### Exercise Questions

**Q1.** How does `ACCESSIBLE BY` specifically reduce the impact of a SQL injection vulnerability compared to relying only on `GRANT EXECUTE`?
> **A:** `GRANT EXECUTE` controls *who* (which schema/role) may call a unit, but within the *same schema* all units are normally mutually callable by default with no way to prevent one package from invoking another. `ACCESSIBLE BY` restricts *which specific program units* — even within the same schema — may call a given unit. So even if an attacker's injected SQL executes in the context of a schema that technically owns a sensitive internal procedure, the call still fails unless it originates from the exact whitelisted caller(s), containing the damage to a much smaller surface.

**Q2.** True or False: Wrapping a package body with `DBMS_DDL.WRAP` prevents SQL injection attacks against that code.
> **A:** **False.** Wrapping only obscures the *readability* of the source code (protects intellectual property); it has no effect on how dynamic SQL inside the package is constructed or executed. A wrapped package with unsafe string concatenation of user input is exactly as vulnerable to injection as its unwrapped equivalent.

**Q3.** In 12.1, could you restrict access to a *single procedure* inside a package while leaving the rest of the package normally accessible? What changed in 12.2?
> **A:** In 12.1, `ACCESSIBLE BY` could only be applied at the level of a whole schema-level unit (package, standalone procedure/function, type). You could not whitelist just one subprogram inside a package spec while leaving others open. 12.2 added the ability to apply whitelisting to an **individual subprogram within a package**, plus the ability to specify the **unit kind** of the accessing program, giving finer-grained control.

---

## 5. Conditional Compilation & Compile-Time Warnings

### 5a. Conditional Compilation
Lets you compile **different PL/SQL code from the same source** depending on compile-time conditions — e.g., include debug code only in dev, support multiple database versions from one code base, or strip out Enterprise-Edition-only features when deploying to Standard Edition.

**Directives:**
| Directive | Purpose |
|---|---|
| `$IF ... $THEN ... $ELSIF ... $ELSE ... $END` | Conditionally include/exclude source text at compile time |
| `$ERROR '<message>' $END` | Force a **compile-time error** with a custom message if a condition is met |
| Inquiry directives: `$$PLSQL_LINE`, `$$PLSQL_UNIT`, `$$PLSQL_UNIT_OWNER`, `$$PLSQL_UNIT_TYPE` | Built-in values available at compile time |
| User-defined flags: `$$my_flag` | Custom flags you define via `PLSQL_CCFLAGS` |

**Setting custom flags:**
```sql
ALTER SESSION SET PLSQL_CCFLAGS = 'debug_mode:1, use_new_algo:0';
```
**Using them:**
```sql
CREATE OR REPLACE PROCEDURE proc1 IS
BEGIN
  $IF $$debug_mode = 1 $THEN
    DBMS_OUTPUT.put_line('DEBUG: entering proc1');
  $END

  $IF DBMS_DB_VERSION.VER_LE_11 $THEN
    -- legacy code path for 11g
  $ELSE
    -- modern code path
  $END
END;
/
```
**Forcing a compile error for an unsupported combination:**
```sql
$IF NOT DBMS_DB_VERSION.VER_LE_18 $THEN
  $ERROR 'This unit requires Oracle 18c or earlier.' $END
$END
```

**Viewing the resolved (post-processed) source** — useful for debugging what conditional compilation actually produced:
```sql
SELECT DBMS_PREPROCESSOR.print_post_processed_source('PACKAGE BODY','HR','MY_PKG')
FROM dual;
```
(`DBMS_PREPROCESSOR.GET_SOURCE` returns it as a `CLOB`/table instead of printing.)

### 5b. Compile-Time Warnings
Oracle can emit **non-fatal** advisory warnings (`PLW-nnnnn` codes) during compilation — code still compiles and is usable, but the warning flags likely bugs, performance issues, or style problems.

**Enabling:**
```sql
ALTER SESSION SET PLSQL_WARNINGS = 'ENABLE:ALL';
-- or more targeted:
ALTER SESSION SET PLSQL_WARNINGS = 'ENABLE:SEVERE','DISABLE:INFORMATIONAL';
```
**Warning categories:**
| Category | Meaning | Example |
|---|---|---|
| `SEVERE` | Likely to cause wrong results / bugs | Referencing an aggregate incorrectly |
| `PERFORMANCE` | Code compiles fine but runs sub-optimally | `PLW-07204`: implicit conversion may prevent index use |
| `INFORMATIONAL` | Style / best-practice notices | `PLW-07203`: parameter/variable declared but never used; `PLW-06002`: unreachable code |

**Managing programmatically:** `DBMS_WARNING` package — `SET_WARNING_SETTING_STRING`, `GET_WARNING_SETTING_STRING`, `ADD_WARNING_SETTING_CAT`, etc., useful for controlling warning behavior per-session or in build scripts.

### Important points to remember
- Conditional compilation is resolved **before** the compiler parses the remaining code — `$IF` branches that evaluate false are never even syntax-checked, which is *by design* (lets you write code referencing objects/packages that may not exist in a given target version).
- `$ERROR` is the mechanism to turn an unsupported configuration into a hard compile failure rather than a silent runtime problem.
- Compile-time warnings **do not stop compilation** — the object is still created and usable (unlike an actual PL/SQL compile error). This is the classic exam trap: warnings ≠ errors.
- `PLSQL_WARNINGS` and `PLSCOPE_SETTINGS` are two **separate** compile parameters — don't confuse them.
- `PLSQL_CCFLAGS` is stored **with the compiled unit** in some contexts (via `ALTER ... COMPILE PLSQL_CCFLAGS = ...`), letting you set flags per-object, not just per-session.

### Exercise Questions

**Q1.** What is the difference in effect between a `$IF...$THEN...$END` branch evaluating to false, versus a runtime `IF...THEN...END IF` branch evaluating to false?
> **A:** A false `$IF` branch is **removed from the source before compilation** — that code is never compiled, never even checked for syntax/semantic errors, and generates zero runtime overhead since it doesn't exist in the compiled unit. A false runtime `IF` branch is still fully compiled and present in the executable; it's simply skipped during execution. This is why conditional compilation can safely reference objects that might not exist under certain configurations, while a runtime `IF` referencing a nonexistent object would fail to compile.

**Q2.** Does enabling `PLSQL_WARNINGS = 'ENABLE:ALL'` cause previously-compiling code to fail to compile? Explain.
> **A:** No. Compile-time warnings are advisory; a unit that produces warnings still compiles successfully and is fully usable. Only actual PL/SQL compilation **errors** (not warnings) prevent an object from becoming valid. Warnings appear via `SHOW ERRORS` / `*_ERRORS` views with an "ERROR" attribute value that indicates they are warnings (e.g., prefixed `PLW-`), not `ORA-`/`PLS-` failures.

**Q3.** You need code that behaves differently between Standard Edition and Enterprise Edition without maintaining two separate source files, and you want compilation to actively fail with a clear message if someone tries to compile Enterprise-only code against a Standard Edition target. Which two conditional-compilation features do you combine, and how?
> **A:** Combine `$IF` (using an inquiry directive or an appropriate `DBMS_DB_VERSION`/edition-detection condition, or a custom `PLSQL_CCFLAGS` flag set to indicate the edition) with `$ERROR '<message>' $END` inside the branch representing the unsupported combination. When compiled against the unsupported edition, the `$ERROR` directive forces a hard compile-time failure with the custom explanatory message, rather than allowing a silent runtime failure later.

---

## 6. PL/Scope & `UTL_CALL_STACK` (Code Analysis)

### 6a. PL/Scope
A **compiler-integrated code-analysis tool** (first in 11g, enhanced in 12.2) that captures detailed cross-reference metadata about every identifier (and, since 12.2, every SQL statement) in your PL/SQL source — declarations, definitions, references, calls, assignments — and stores it in static data-dictionary views for querying.

**Enabling collection (a compile-time parameter):**
```sql
ALTER SESSION SET PLSCOPE_SETTINGS = 'IDENTIFIERS:ALL, STATEMENTS:ALL';
-- Then simply (re)compile the unit(s) you want analyzed.
```
| `IDENTIFIERS` value | Meaning |
|---|---|
| `NONE` (default) | No collection |
| `ALL` | Collect all PL/SQL and SQL identifier data |
| `PUBLIC` | Only publicly-visible identifiers (excludes DEFINITION usage) |
| `SQL` | Only SQL identifiers (tables, columns, views...) |
| `PLSQL` | Only PL/SQL identifiers (variables, types, subprograms...) |

**Key views:**
| View | Contains |
|---|---|
| `USER/ALL/DBA_IDENTIFIERS` | Every identifier: name, type, usage (declaration/definition/reference/call/assignment...), object, line, column |
| `USER/ALL/DBA_STATEMENTS` | Every SQL statement inside PL/SQL: statement type (SELECT/INSERT/UPDATE/DELETE/MERGE/EXECUTE IMMEDIATE/OPEN), owning object, line/column — added as part of the 12.2 enhancement |
| `USER/ALL/DBA_PLSQL_OBJECT_SETTINGS` | Shows the `PLSCOPE_SETTINGS` a given unit was compiled with |

**Typical use case — impact analysis:** "If I rename/drop this column, which program units reference it?"
```sql
SELECT owner, object_name, object_type, line, column_value.type AS usage
FROM   all_identifiers
WHERE  name = 'SALARY' AND type = 'COLUMN';
```

### 6b. `UTL_CALL_STACK`
A structured, **object-oriented replacement** for the old text-parsing approach of `DBMS_UTILITY.FORMAT_CALL_STACK` / `FORMAT_ERROR_STACK` / `FORMAT_ERROR_BACKTRACE`. Instead of getting one big formatted string you had to manually parse, you get **typed functions** returning individual pieces of information about each frame on the call stack.

**Key functions:**
| Function | Returns |
|---|---|
| `DYNAMIC_DEPTH` | Current number of frames on the call stack |
| `SUBPROGRAM(dynamic_depth)` | A collection giving the qualified name (unit, nested subprogram names) at that depth |
| `UNIT_LINE(dynamic_depth)` | Line number of the call at that depth |
| `OWNER(dynamic_depth)` | Owning schema of the unit at that depth |
| `CONCATENATE_SUBPROGRAM(name_collection)` | Turns the subprogram-name collection into a readable dotted string |
| `ERROR_DEPTH`, `ERROR_MSG(depth)`, `ERROR_NUMBER(depth)` | Equivalent structured access to the **error stack** (multiple stacked exceptions) |
| `BACKTRACE_DEPTH`, `BACKTRACE_UNIT(depth)`, `BACKTRACE_LINE(depth)` | Structured access to the **error backtrace** (where the exception actually occurred/propagated, as opposed to where it's handled) |

**Example:**
```sql
CREATE OR REPLACE PROCEDURE display_call_stack IS
  l_depth PLS_INTEGER := UTL_CALL_STACK.dynamic_depth;
BEGIN
  FOR i IN 1 .. l_depth LOOP
    DBMS_OUTPUT.put_line(
      i || ': ' ||
      UTL_CALL_STACK.concatenate_subprogram(UTL_CALL_STACK.subprogram(i)) ||
      ' (line ' || UTL_CALL_STACK.unit_line(i) || ')'
    );
  END LOOP;
END;
/
```

### Important points to remember
- PL/Scope data is generated **only for units compiled while the relevant `PLSCOPE_SETTINGS` was in effect** — recompiling a unit is required after changing the setting for it to be re-analyzed.
- PL/Scope is purely a **static, compile-time** analysis tool (like a built-in "cscope"/"ctags" for PL/SQL) — it tells you about the *code structure*, not runtime behavior.
- `UTL_CALL_STACK` is purely a **runtime** introspection tool (what's currently executing) — do not confuse the two: PL/Scope = analyze source at compile time; `UTL_CALL_STACK` = inspect the active call chain during execution (commonly used in generic logging/error-handling frameworks).
- `UTL_CALL_STACK` frames are numbered from **1 = most deeply nested/most recent call** outward, matching how `DBMS_UTILITY.FORMAT_CALL_STACK` orders its text, but now accessible as discrete, typed values instead of a string you'd have to parse with regular expressions.
- `UTL_CALL_STACK` operations can raise `UTL_CALL_STACK.BAD_DEPTH_INDICATOR` if you pass an out-of-range depth.

### Exercise Questions

**Q1.** A team wants to find every PL/SQL program unit in the database that calls a specific deprecated procedure, before removing it. Which tool is appropriate, and how would they use it?
> **A:** PL/Scope. After ensuring the calling units were compiled with `PLSCOPE_SETTINGS='IDENTIFIERS:ALL'` (or at least `PLSQL`), they can query `ALL_IDENTIFIERS` (or `DBA_IDENTIFIERS`) filtering on `NAME = '<procedure name>'` and `TYPE = 'PROCEDURE'` with a usage context of `CALL`, joining back to `OBJECT_NAME`/`OWNER` to get the list of calling units and exact line numbers — a compile-time cross-reference, not something you could get by just grepping source text reliably (which would miss synonyms, package-qualified calls, etc., and be case/whitespace-fragile).

**Q2.** Why was `UTL_CALL_STACK` introduced when `DBMS_UTILITY.FORMAT_CALL_STACK` already existed?
> **A:** `FORMAT_CALL_STACK` (and the related `FORMAT_ERROR_STACK`/`FORMAT_ERROR_BACKTRACE`) return one large pre-formatted text block that a developer has to parse manually (with string functions/regex) to extract individual pieces like a specific line number or a specific subprogram name — fragile and error-prone, especially since the format could shift or be hard to disambiguate for deeply nested/overloaded names. `UTL_CALL_STACK` exposes the same information as discrete, strongly-typed function calls (per stack depth), which is far more reliable to consume programmatically, e.g. inside generic logging/exception-handling frameworks.

**Q3.** True or False: Enabling `PLSCOPE_SETTINGS='IDENTIFIERS:ALL'` retroactively populates `ALL_IDENTIFIERS` for all packages already compiled in the schema.
> **A:** **False.** `PLSCOPE_SETTINGS` is a compile-time parameter — it only affects units compiled (or recompiled) **after** it is set. Existing compiled objects will show no PL/Scope data (or stale data from whatever setting was in effect when they were last compiled) until they are explicitly recompiled under the new setting.

---

## 7. Edition-Based Redefinition (EBR) & Invisible Columns (12c/19c)

### 7a. Edition-Based Redefinition (EBR)
EBR allows you to **upgrade PL/SQL application logic (and views) online, with zero downtime**, by letting multiple versions ("editions") of the same PL/SQL objects and views coexist in the database simultaneously. Existing sessions keep running against the old edition while new sessions can be pointed at the new edition; once everyone has moved over, the old edition is dropped.

**Key concepts:**
| Concept | Description |
|---|---|
| **Edition** | A non-schema database object (no owner) representing a "version" of editionable objects. Every database starts with a default edition `ORA$BASE`. |
| **Editioned objects** | Objects whose type is editionable (views, synonyms, and all PL/SQL object types — packages, procedures, functions, triggers, types) *and* whose owning schema is "edition-enabled." Each edition can have its own independent copy. |
| **Non-editioned objects** | **Tables are never editioned** — table structure is shared by all editions. This is why editioning *views* exist (below). |
| **Editioning view** | A special, restricted view (one table, `SELECT` only, no `FOR UPDATE`, no subquery factoring) placed **over** a base table so PL/SQL code compiles against the *view* rather than the table directly — this indirection is what lets you change a table's apparent shape per-edition without altering the shared table itself. |
| **Cross-edition trigger** | A trigger designed to fire *during the upgrade window* to keep data written by one edition consistent with the structure expected by another edition. A **forward** crossedition trigger propagates old-edition writes into new-edition columns/shape; a **reverse** crossedition trigger does the opposite (needed for "hot rollover", where old and new editions are in simultaneous live use). These are temporary — dropped once everyone is on the new edition. |

**Basic workflow to "editions-enable" an existing app** (classic exam sequence):
1. `ALTER USER <schema> ENABLE EDITIONS;`
2. Rename the base table (e.g., `EMPLOYEES` → `EMPLOYEES_TAB`) — this invalidates dependent PL/SQL (except triggers, which move with the table).
3. Create an **editioning view** with the *original* table name (`EMPLOYEES`) over the renamed table, exposing the original columns — dependent code recompiles cleanly against the view.
4. Move/recompile any triggers to fire on the editioning view instead of the base table.
5. Re-point VPD policies, grants, and synonyms at the editioning view rather than the base table.

**Creating and using a new edition:**
```sql
CREATE EDITION v2 CHILD OF ora$base;
ALTER SESSION SET EDITION = v2;
-- Now redefine PL/SQL / views only within this session's edition —
-- ora$base sessions are completely unaffected.
```

**Dictionary views:** `*_EDITIONS`, `*_OBJECTS` (current edition only) vs. `*_OBJECTS_AE` ("All Editions" — shows objects across every edition), `*_EDITIONING_VIEWS`.

### Important points to remember
- **Tables cannot be editioned** — that's the whole reason editioning views and crossedition triggers exist. This is a very common exam trap ("which of the following IS editionable: table / view / package / trigger?" → table is **not**).
- EBR's core value proposition: **online application upgrade with no downtime** and safe rollback (just switch the default edition back).
- Editions form a **parent/child hierarchy** — a child edition inherits everything from its parent except what it explicitly overrides.
- In a Multitenant (CDB) environment, an edition's scope is the **PDB**, not the whole CDB.
- Crossedition triggers are commonly written as **compound triggers** because handling the transformation can require logic across multiple timing points.

### 7b. Invisible Columns (12c+)
A column can be marked `INVISIBLE`, meaning it is **hidden from**:
- `SELECT *`
- `DESCRIBE`
- Generic tools that don't explicitly name it

...but it is **still fully usable** if explicitly referenced by name in `SELECT`, `INSERT`, `UPDATE`, `WHERE`, indexes, and constraints.

```sql
CREATE TABLE employees (
  emp_id     NUMBER,
  emp_name   VARCHAR2(50),
  ssn        VARCHAR2(11) INVISIBLE     -- hidden by default
);

ALTER TABLE employees MODIFY (ssn VISIBLE);     -- reveal it
ALTER TABLE employees MODIFY (ssn INVISIBLE);   -- hide it again
```

**Why it matters for security/exam purposes:**
- Lets you **add a new sensitive/internal column** to a widely-used table without breaking every `INSERT ... VALUES` / `SELECT *`-based piece of legacy application code that assumes the old column list — a practical online-schema-evolution tool, often used *alongside* EBR during a rolling upgrade.
- It is **not an access-control mechanism** — invisibility is not security. Any session that knows the column name and has ordinary object privileges can still query it directly (`SELECT ssn FROM employees`). For actual protection you still need VPD/column masking (`DBMS_REDACT`) or column-level privileges.

### Exercise Questions

**Q1.** Why can't tables themselves be "editioned" the way packages and views can, and how does EBR work around this?
> **A:** Table data must remain a single, shared, consistent copy for all sessions regardless of edition — you cannot have "one copy of the data per edition" without massive duplication and synchronization problems, so Oracle deliberately makes tables non-editionable. EBR works around this by never letting application code reference the table directly: it interposes an **editioning view** (which *is* editionable) with the original table name, so each edition can present its own column set/shape over the *same underlying table data*, and uses **crossedition triggers** to keep the shared data consistent in the shapes both old and new editions expect during the transition.

**Q2.** What is the difference between a forward crossedition trigger and a reverse crossedition trigger, and when would you need both?
> **A:** A **forward** crossedition trigger transforms data written by the **old edition** into the shape/columns required by the **new edition** — needed so the new edition sees consistent data even while old-edition sessions are still writing. A **reverse** crossedition trigger does the opposite: transforms data written by the **new edition** back into the shape the old edition expects. You need **both** only in a "hot rollover" scenario, where old and new editions must be simultaneously live and both actively read/write data (rather than simply migrating everyone over at once) — otherwise a forward trigger alone typically suffices.

**Q3.** A developer marks a `SALARY` column `INVISIBLE` and tells the security team "this now satisfies our requirement to protect salary data from unauthorized access." Is the security team right to accept this?
> **A:** No. `INVISIBLE` only removes the column from default/implicit results like `SELECT *` and `DESCRIBE` — it does not restrict access. Any session with `SELECT` privilege on the table can still retrieve the column by naming it explicitly (`SELECT salary FROM employees`). Genuine protection requires an actual access-control mechanism — e.g., column-level VPD (`sec_relevant_cols`), `DBMS_REDACT` data redaction, or revoking column-level privileges — not visibility alone.

---

## Quick Cross-Topic Summary Table

| Topic | Primary Package/Clause | Enforced At | Introduced |
|---|---|---|---|
| Fine-Grained Access Control | `DBMS_RLS` | Query rewrite (row/column level) | 8i, expanded through 12c |
| Application Context | `CREATE CONTEXT`, `DBMS_SESSION`, `SYS_CONTEXT` | Session/global memory | 8i, global context in 9i/10g |
| SQL Injection Prevention | `DBMS_ASSERT` | Dynamic SQL string construction | 10g R2 |
| Code-Based Access Control | `ACCESSIBLE BY` | PL/SQL compiler (caller identity) | 12.1, enhanced 12.2 |
| Source Obfuscation | `DBMS_DDL.WRAP` / `wrap` utility | Source distribution | Long-standing, still used |
| Conditional Compilation | `$IF/$THEN/$ERROR`, `PLSQL_CCFLAGS` | Compile time | 10g R2 |
| Compile-Time Warnings | `PLSQL_WARNINGS`, `DBMS_WARNING` | Compile time | 10g R1 |
| PL/Scope | `PLSCOPE_SETTINGS`, `*_IDENTIFIERS`/`*_STATEMENTS` | Compile time (static analysis) | 11g, statements added 12.2 |
| UTL_CALL_STACK | `UTL_CALL_STACK` | Runtime | 12.1 |
| Edition-Based Redefinition | `CREATE EDITION`, editioning views, crossedition triggers | Object versioning | 11g R2 |
| Invisible Columns | `INVISIBLE`/`VISIBLE` column modifier | Schema/DDL (not security) | 12.1 |