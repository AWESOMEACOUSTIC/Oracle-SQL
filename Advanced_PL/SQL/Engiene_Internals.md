# Oracle PL/SQL — Engine Internals, Security Rights & Package Architecture
### Exam Study Guide

Covers: Key Components (PL/SQL Engine, Context Switching, PL/SQL Optimizer) · Definer vs. Invoker Rights (`AUTHID CURRENT_USER`) · Package State · Package Initialization Block · Private vs. Public Subprograms

---

## 1. Key Components — PL/SQL Engine, Context Switching, PL/SQL Optimizer

### The PL/SQL Engine
Oracle's PL/SQL engine is **built into the database** and is what compiles and executes PL/SQL blocks. Internally it separates work between two cooperating executors:

| Component | Handles |
|---|---|
| **PL/SQL statement executor** | Procedural constructs: `IF`, loops, variable assignment, exception handling — anything that isn't SQL |
| **SQL statement executor (SQL engine)** | Any embedded SQL statement (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `COMMIT`, etc.) found inside the PL/SQL block |

The PL/SQL engine executes procedural statements directly itself, and whenever it encounters an embedded SQL statement, it **hands that statement off to the SQL engine**, waits for the result, and resumes procedural execution. This tight, in-process cooperation is why PL/SQL is described as "tightly integrated with SQL," but it is not free of overhead — which brings us to context switching.

### Context Switching
**Context switching** is the transfer of control between the PL/SQL engine and the SQL engine (in either direction) to execute an embedded SQL statement and return its result. Each switch has real overhead: Oracle must save/restore process state as control passes between the two engines.

```sql
-- Naive row-by-row loop: ONE context switch PER ROW (10,000 rows = 10,000 switches)
BEGIN
  FOR r IN (SELECT employee_id FROM employees WHERE department_id = 50) LOOP
    UPDATE employees SET salary = salary * 1.10 WHERE employee_id = r.employee_id;
  END LOOP;
END;
/
```
This row-by-row pattern (nicknamed "**slow-by-slow**" processing) is a classic performance anti-pattern precisely because of the accumulated context-switch overhead.

**Reducing context switches — the standard fix:**
```sql
-- Pure SQL: a single context switch total, regardless of row count
UPDATE employees SET salary = salary * 1.10 WHERE department_id = 50;
```
```sql
-- When row-by-row PL/SQL logic really is needed: BULK COLLECT + FORALL
-- batch many rows into ONE round trip instead of one-per-row
DECLARE
  TYPE id_tab IS TABLE OF employees.employee_id%TYPE;
  v_ids id_tab;
BEGIN
  SELECT employee_id BULK COLLECT INTO v_ids
  FROM   employees WHERE department_id = 50;

  FORALL i IN 1 .. v_ids.COUNT
    UPDATE employees SET salary = salary * 1.10 WHERE employee_id = v_ids(i);
END;
/
```
`BULK COLLECT` batches multiple **fetch** results into one switch; `FORALL` batches multiple **DML** executions into one switch — both dramatically cut context-switch count compared to an equivalent row-by-row loop.

### The PL/SQL Optimizer
Since Oracle 10g, PL/SQL **compilation** includes an optimizer that can automatically rearrange/transform your source code to run faster, without changing its logical behavior — controlled by the compile-time parameter **`PLSQL_OPTIMIZE_LEVEL`**.

| Level | Behavior |
|---|---|
| `0` | No optimization at all (legacy/compatibility use only) |
| `1` | Basic optimizations, but code is **not rearranged** — lowest level that still applies some optimization |
| `2` (**default**) | Full optimization, including code rearrangement; **subprogram inlining requires an explicit `PRAGMA INLINE(subprogram, 'YES')`** on each call site you want inlined |
| `3` | Most aggressive: the compiler **automatically seeks out** subprograms to inline on its own, without needing the pragma (the pragma can still be used to raise/lower priority) |

**Subprogram inlining** replaces a call to a subprogram with an actual copy of that subprogram's code at the call site (only possible when caller and callee are in the same compiled unit) — eliminating call overhead entirely, at the cost of larger compiled code.
```sql
DECLARE
  FUNCTION add_nums (a PLS_INTEGER, b PLS_INTEGER) RETURN PLS_INTEGER IS
  BEGIN
    RETURN a + b;
  END;
  PRAGMA INLINE (add_nums, 'YES');
  v NUMBER;
BEGIN
  v := add_nums(1,2) + add_nums(3,4);  -- with LEVEL=2, only THESE calls are inlined
END;
/
```

### Important points to remember
- Context switching happens **any time** control moves between the two engines — this includes not just DML, but also implicit/explicit cursors, `COMMIT`/`ROLLBACK`, and (importantly) calling a **PL/SQL function from within a SQL statement** (e.g., a user-defined function in a `SELECT` list) — that direction (SQL → PL/SQL) also counts as a context switch, and is a very common exam trap since people usually only think of the PL/SQL → SQL direction.
- `BULK COLLECT` reduces switches on the **fetch/query side**; `FORALL` reduces switches on the **DML side** — know which one applies to which direction.
- `PLSQL_OPTIMIZE_LEVEL=1` exists specifically as an escape hatch: in rare cases, code rearrangement performed at level 2+ could make an exception raise earlier/later or not at all in edge cases, or excessive compile-time overhead on huge units — level 1 disables the rearrangement.
- At `PLSQL_OPTIMIZE_LEVEL=2` (the default), inlining is **opt-in per call site** via `PRAGMA INLINE`; at `LEVEL=3`, the compiler inlines proactively **without needing the pragma at all** — this level distinction is a frequent exam question.
- `PRAGMA INLINE(subprogram, 'NO')` can also be used to explicitly **prevent** inlining of a specific call (useful if inlining is found to hurt performance in a specific case, e.g., via the hierarchical profiler).

### Exercise Questions

**Q1.** A developer writes a `SELECT` statement that calls a PL/SQL function in its select list to compute a bonus for every row. Does this cause context switching, and if so, in which direction?
> **A:** Yes. Even though the outer statement is pure SQL, invoking a PL/SQL function from within it requires the **SQL engine to switch control to the PL/SQL engine** to execute the function's logic for each row, then switch back to continue the query — a SQL → PL/SQL → SQL context switch. This is easy to overlook since people tend to associate context switching only with PL/SQL code calling out to SQL, not the reverse.

**Q2.** What is the specific difference in how subprogram inlining is controlled between `PLSQL_OPTIMIZE_LEVEL=2` and `PLSQL_OPTIMIZE_LEVEL=3`?
> **A:** At level 2 (the default), the compiler will **only** inline a subprogram call if that specific call site is explicitly marked with `PRAGMA INLINE(subprogram, 'YES')` — nothing is inlined automatically. At level 3, the compiler **actively looks for** opportunities to inline subprograms on its own, without requiring the pragma; the pragma can still be used at level 3 to raise a call's inlining priority or to explicitly disable inlining for a particular call (`'NO'`).

**Q3.** Why is a `FOR` loop that issues one `UPDATE` per row inside the loop considered inefficient, and what are the two standard remedies?
> **A:** Each loop iteration's `UPDATE` requires a separate context switch from the PL/SQL engine to the SQL engine and back — for 10,000 rows, that's 10,000 switches, each carrying real overhead (this is the "slow-by-slow" anti-pattern). The two standard remedies are: (1) rewrite the logic as a **single pure-SQL statement** (e.g., one `UPDATE ... WHERE department_id = 50`) if the logic permits it entirely in SQL, or (2) if row-by-row PL/SQL logic is genuinely required, use **`BULK COLLECT`** to fetch many rows in one switch and **`FORALL`** to execute the DML for all of them in one switch, instead of one-at-a-time.

---

## 2. Definer Rights vs. Invoker Rights (`AUTHID`)

### What it is
Every PL/SQL stored program unit (procedure, function, package, trigger) runs with one of two **privilege models**, controlled by the `AUTHID` clause:

| Model | Clause | Runs with whose privileges? | Which schema's objects does unqualified SQL resolve against? |
|---|---|---|---|
| **Definer's rights** (default) | `AUTHID DEFINER` (or simply omitted) | The **owner/creator** of the program unit | The **owner's** schema |
| **Invoker's rights** | `AUTHID CURRENT_USER` | The **user who is calling** the program unit | The **caller's** schema |

### Definer's rights (the default)
```sql
CREATE OR REPLACE PROCEDURE hr.show_dept_count
IS
  v_count NUMBER;
BEGIN
  SELECT COUNT(*) INTO v_count FROM departments;  -- always HR.DEPARTMENTS
  DBMS_OUTPUT.put_line(v_count);
END;
/
```
No matter which user calls `hr.show_dept_count`, the unqualified reference to `departments` **always** resolves to `HR.DEPARTMENTS`, and the procedure runs with `HR`'s privileges on that table — the caller doesn't even need direct privileges on `departments` themselves, only `EXECUTE` on the procedure.

### Invoker's rights
```sql
CREATE OR REPLACE PROCEDURE hr.show_dept_count
AUTHID CURRENT_USER
IS
  v_count NUMBER;
BEGIN
  SELECT COUNT(*) INTO v_count FROM departments;  -- resolves against the CALLER's schema!
  DBMS_OUTPUT.put_line(v_count);
END;
/
```
Now, if user `SALES_APP` calls this same procedure, `departments` resolves to `SALES_APP.DEPARTMENTS` (if such a table exists in that caller's own schema) — **not** `HR.DEPARTMENTS` — and it executes with the **caller's** own privileges, not `HR`'s.

### Practical use case for invoker's rights
A common real-world pattern: a **generic, reusable utility package** (e.g., a reporting or audit-logging package) shipped/installed once in a central schema, but intended to operate against **each calling user's own data** — invoker's rights let the same compiled code correctly resolve table references per-caller, rather than always hitting the owning schema's tables.

### Important points to remember
- **Default is definer's rights** — you must explicitly write `AUTHID CURRENT_USER` to get invoker's rights; there is no equivalent explicit keyword required for definer's rights (though `AUTHID DEFINER` can be written for clarity).
- With invoker's rights, **name resolution of schema objects happens at runtime, based on the invoker**, whereas with definer's rights all name resolution is fixed to the owner's schema regardless of caller.
- Invoker's rights units still require the **caller to hold the necessary object privileges directly** (or via a role, subject to the usual PL/SQL role restrictions) on whatever they end up referencing — the code doesn't inherit the definer's privileges the way definer's-rights code does.
- **`AUTHID` applies at the package/standalone-unit level** — for a package, it governs the whole package (both spec and body); you cannot mix definer's rights on some subprograms and invoker's rights on others within the same package.
- Invoker's rights is an important **defense-in-depth consideration**: definer's-rights code effectively grants the caller a "loan" of the definer's privileges for the duration of that call — this is exactly why SQL-injection-style attacks against definer's-rights PL/SQL are so dangerous (the injected SQL runs with the *definer's*, often more powerful, privileges).
- Views, by contrast, always use definer's rights semantics by default in terms of privilege checking, but that's a separate (though related) topic from PL/SQL's `AUTHID`.

### Exercise Questions

**Q1.** Why is definer's rights PL/SQL considered a bigger security risk in the context of SQL injection than invoker's rights code?
> **A:** Definer's-rights code executes with the **privileges of whoever created/owns it**, regardless of who calls it — so if an attacker can manipulate a definer's-rights procedure into executing unintended SQL (e.g., via dynamic SQL injection), that unintended SQL runs with the **definer's** privileges, which are often significantly broader than the calling user's own. Invoker's-rights code executes with the **caller's own** privileges, so even if injection succeeds, the attacker is limited to what the calling session itself could already do — a much smaller blast radius.

**Q2.** A reusable utility package is installed once in a central `UTIL` schema but needs to query a table named `AUDIT_LOG` that exists separately, with the same structure, in every calling application's own schema. Which `AUTHID` setting is required, and why?
> **A:** `AUTHID CURRENT_USER` (invoker's rights). With the default definer's rights, an unqualified reference to `AUDIT_LOG` inside the package would always resolve to `UTIL.AUDIT_LOG` no matter who calls it. Invoker's rights makes name resolution happen **at runtime relative to the calling user**, so each caller's own `AUDIT_LOG` table is correctly targeted without needing a separate copy of the package per schema.

**Q3.** True or False: You can declare one procedure in a package body with `AUTHID CURRENT_USER` while another procedure in the same package body keeps the default definer's rights behavior.
> **A:** **False.** `AUTHID` is specified once, at the package level (in the package specification), and applies uniformly to **every** subprogram in that package — it cannot be mixed within a single package. To get different rights models for different pieces of logic, you would need to split them into separate packages/standalone program units with different `AUTHID` settings.

---

## 3. Package State

### What it is
**Package state** refers to the values held by **package-level variables, cursors, and other data structures** declared in a package's specification or body (not local to any one subprogram) — this data **persists across multiple calls to the package's subprograms within the same session**, unlike a subprogram's own local variables, which are re-initialized on every call.

```sql
CREATE OR REPLACE PACKAGE counter_pkg IS
  PROCEDURE increment;
  FUNCTION get_count RETURN NUMBER;
END counter_pkg;
/
CREATE OR REPLACE PACKAGE BODY counter_pkg IS
  g_count NUMBER := 0;   -- PACKAGE STATE: persists across calls, for this session

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
```sql
BEGIN
  counter_pkg.increment;  -- g_count becomes 1
  counter_pkg.increment;  -- g_count becomes 2
  DBMS_OUTPUT.put_line(counter_pkg.get_count);  -- prints 2
END;
/
```

### Scope of persistence — session-specific by default
Package state is stored in the **UGA (User Global Area)**, part of each session's own private memory — meaning **each session gets its own independent copy** of package-level variables. Session A incrementing `g_count` has **no effect** on Session B's view of `g_count`; each session's package state lasts for the **duration of that session** (until logoff, or explicit reset — see below), not forever and not shared globally.

### Resetting package state
```sql
EXECUTE DBMS_SESSION.RESET_PACKAGE;         -- resets state of ALL packages in the session
-- or, per-package:
ALTER PACKAGE counter_pkg STATE = ... ;      -- (no direct SQL reset per-package; typically done via reset_package or ending the session)
```
`DBMS_SESSION.RESET_PACKAGE` reinitializes all package state for the current session back to its starting values (as if freshly instantiated) — useful in connection-pooled environments to avoid one logical "user" inheriting leftover state from a previous one sharing the same physical session.

### `SERIALLY_REUSABLE` — the exception to session-persistence
```sql
CREATE OR REPLACE PACKAGE work_pkg IS
  PRAGMA SERIALLY_REUSABLE;
  PROCEDURE do_work;
END work_pkg;
/
```
A package marked `SERIALLY_REUSABLE` does **not** retain its state across separate calls within the session — its package-level variables are reset to their initial values **each time a new "work cycle" begins** (each top-level call from outside the package), and its state storage is released back to a shared pool rather than being held in each session's own UGA for the whole session. This trades away persistence for a **smaller per-session memory footprint** — valuable for packages called occasionally by a very large number of concurrent sessions, where each session holding its own persistent copy of unused state would waste significant memory.

### Important points to remember
- Package state is **per-session**, not global/shared across all sessions — a very common exam trap ("does incrementing a package variable in one session affect another session's view of it?" → **no**).
- Package state persists for the **life of the session** by default — it is *not* reset between individual subprogram calls, which is precisely what makes it useful for things like counters, caches, or flags that need to survive across multiple calls.
- `DBMS_SESSION.RESET_PACKAGE` is the standard tool to explicitly clear all package state for a session without ending the session itself — important in **connection-pooled** architectures.
- `PRAGMA SERIALLY_REUSABLE` is the mechanism to **opt out** of session-long persistence, trading it for reduced memory overhead in high-session-count scenarios where persistent state isn't actually needed.
- Cursors declared at the package level are also part of package state — an explicit package-level cursor stays open/positioned across calls unless explicitly closed, following the same session-scoped lifetime rules as package variables.

### Exercise Questions

**Q1.** Two different users are simultaneously connected to the database, each calling `counter_pkg.increment` and `counter_pkg.get_count` (from Section 3's example) in their own separate sessions. Does one user's calls to `increment` affect the value the other user sees from `get_count`?
> **A:** No. Package state is stored per-session in each session's own UGA — each session has its **own independent copy** of `g_count`. One session's increments have no effect whatsoever on another session's view of the package's state.

**Q2.** In a connection-pooled web application where many logical end-users share the same small set of physical database sessions over time, what risk does package state create, and what's the standard mitigation?
> **A:** Package state persists for the life of the **physical session**, not the logical end-user — so if session state (like a cached "current user's" data in a package variable) isn't cleared between different end-users sharing that same pooled connection, a later end-user could inadvertently see or be affected by stale state left over from an earlier end-user. The standard mitigation is calling `DBMS_SESSION.RESET_PACKAGE` when a pooled session is handed off to a new logical user, resetting all package-level state to its initial values.

**Q3.** What is the tradeoff introduced by marking a package `PRAGMA SERIALLY_REUSABLE`?
> **A:** The package **gives up cross-call persistence of its state within a session** — its package-level variables reset each time a new top-level call cycle begins, rather than surviving for the whole session — in exchange for **reduced memory overhead**, since its state is drawn from and returned to a shared work area rather than being permanently held in every session's own UGA for the session's full lifetime. This is beneficial specifically when a package is invoked occasionally by very many concurrent sessions, most of which don't actually need to retain that state between calls.

---

## 4. Package Initialization Block

### What it is
A package body may include an **initialization section** — an executable block placed **after all the subprogram bodies**, with no separate name/header (it looks like a mini top-level `BEGIN...END` at the very end of the package body). This block runs **exactly once per session**, automatically, the **first time** that package is referenced in that session (whether by calling a subprogram, referencing a package variable/constant, etc.) — you never call it explicitly.

```sql
CREATE OR REPLACE PACKAGE BODY hr_pkg IS
  g_company_name VARCHAR2(100);
  g_tax_rate     NUMBER;

  PROCEDURE show_info IS
  BEGIN
    DBMS_OUTPUT.put_line(g_company_name || ' @ ' || g_tax_rate);
  END;

BEGIN
  -- INITIALIZATION BLOCK: runs once, automatically, on first reference this session
  SELECT company_name, tax_rate INTO g_company_name, g_tax_rate
  FROM   company_settings WHERE ROWNUM = 1;
END hr_pkg;
/
```
```sql
BEGIN
  hr_pkg.show_info;   -- FIRST reference this session:
                       --   1. initialization block runs automatically (loads settings)
                       --   2. THEN show_info executes and prints them
  hr_pkg.show_info;    -- subsequent call: initialization block does NOT run again
END;
/
```

### Important points to remember
- The initialization block runs **once per session, on first use** — not once per call, not once per database, and not at compile time.
- It is typically used to **populate package-level variables/caches** with values that are expensive to compute or fetch (e.g., configuration lookups, reference-data caching) so subsequent calls in the same session can use the already-loaded value instead of re-fetching it every time.
- Because it ties into **package state** (Section 3), the same session-scoping rules apply: if `DBMS_SESSION.RESET_PACKAGE` is called, or the package is `SERIALLY_REUSABLE` and a new work cycle begins, the initialization block will run again the next time the package is referenced.
- If the initialization block raises an unhandled exception, the **entire package becomes unusable for that session** (referencing it again typically re-attempts initialization, but the immediate reference that triggered it fails) — a common source of confusing "why did my very first call fail" bugs when the real problem is in the init block's logic, not the subprogram you actually called.
- The initialization block has **no name and no parameters** — it's simply the trailing `BEGIN ... END;` in the package body, distinguished from subprogram bodies only by position (it must come after all subprogram declarations/bodies) and the fact that it isn't attached to a `PROCEDURE`/`FUNCTION` header.

### Exercise Questions

**Q1.** A package's initialization block performs a `SELECT` to populate a package-level constant with a configuration value. If a session calls three different subprograms from this package in sequence, how many times does the initialization block execute?
> **A:** **Once** — specifically on the **first** reference to the package in that session (whichever of the three calls happens first). The other two calls do not re-trigger it; the loaded configuration value simply remains available as package state for the rest of the session (unless explicitly reset).

**Q2.** If a package's initialization block raises an unhandled exception, what happens when a subprogram from that package is called for the first time in a session?
> **A:** The reference fails — the exception from the initialization block propagates out, and the subprogram call that triggered initialization does not complete successfully. In practice, this makes debugging tricky because the visible error appears to originate from the subprogram call itself, when the actual root cause is in the package's initialization logic; identifying that the failure is coming from the init section (rather than the called subprogram's own body) is a key troubleshooting skill.

**Q3.** How does a package's initialization block interact with `PRAGMA SERIALLY_REUSABLE`?
> **A:** For a normal (non-serially-reusable) package, the initialization block runs once per **session**. For a `SERIALLY_REUSABLE` package, since its state is reset at the start of each new "work cycle" (rather than persisting for the whole session), the initialization block correspondingly re-runs at the start of **each new work cycle**, not just once per session — consistent with the fact that serially-reusable package state doesn't survive between cycles the way ordinary package state survives across a session.

---

## 5. Private vs. Public Subprograms

### What it is
Inside a **package**, subprograms (and other elements — types, constants, cursors) can be **public** or **private**, controlling their visibility to code *outside* the package:

| | Public | Private |
|---|---|---|
| Declared in | The **package specification** (and implemented in the body) | Only the **package body** (never in the spec) |
| Callable from outside the package? | **Yes** — via `package_name.subprogram_name(...)` | **No** — `PLS-00302`-class "component must be declared" error if attempted from outside |
| Callable from other subprograms within the same package body? | Yes | Yes |
| Purpose | The package's **public API** — the intended interface for consumers | **Internal helper logic** — implementation details hidden from consumers |

### Example
```sql
CREATE OR REPLACE PACKAGE payroll_pkg IS
  -- PUBLIC: declared in the spec, this IS the package's API
  FUNCTION calculate_net_pay (p_emp_id NUMBER) RETURN NUMBER;
END payroll_pkg;
/

CREATE OR REPLACE PACKAGE BODY payroll_pkg IS

  -- PRIVATE: only declared here in the body, never in the spec
  FUNCTION calculate_tax (p_gross NUMBER) RETURN NUMBER IS
  BEGIN
    RETURN p_gross * 0.20;
  END;

  -- PUBLIC implementation
  FUNCTION calculate_net_pay (p_emp_id NUMBER) RETURN NUMBER IS
    v_gross NUMBER;
  BEGIN
    SELECT salary INTO v_gross FROM employees WHERE employee_id = p_emp_id;
    RETURN v_gross - calculate_tax(v_gross);   -- private helper called internally
  END;

END payroll_pkg;
/
```
```sql
SELECT payroll_pkg.calculate_net_pay(100) FROM dual;   -- OK — public
SELECT payroll_pkg.calculate_tax(5000) FROM dual;       -- ERROR — calculate_tax is private
```

### An important ordering rule for private subprograms
A private subprogram must be **declared/defined before it is first referenced within the package body**, reading top-to-bottom — unlike public subprograms (whose specification in the package spec makes them visible everywhere in the body regardless of body ordering). If a private helper needs to be called by another private (or public) subprogram *defined earlier* in the body, you must add a **forward declaration**:
```sql
CREATE OR REPLACE PACKAGE BODY payroll_pkg IS

  FUNCTION calculate_tax (p_gross NUMBER) RETURN NUMBER;  -- forward declaration

  FUNCTION calculate_net_pay (p_emp_id NUMBER) RETURN NUMBER IS
    v_gross NUMBER;
  BEGIN
    SELECT salary INTO v_gross FROM employees WHERE employee_id = p_emp_id;
    RETURN v_gross - calculate_tax(v_gross);  -- forward-declared, so this compiles fine
  END;

  FUNCTION calculate_tax (p_gross NUMBER) RETURN NUMBER IS  -- actual body, later
  BEGIN
    RETURN p_gross * 0.20;
  END;

END payroll_pkg;
/
```

### Important points to remember
- **Only the package specification defines the public interface** — anything implemented in the body but *not* declared in the spec is automatically private, with no separate `PRIVATE` keyword needed.
- Private subprograms directly support **encapsulation/information hiding**: implementation details (helper logic, internal calculations) can be freely refactored without breaking any external caller, since external code never referenced them in the first place.
- Private subprograms are also relevant to **security surface reduction** — since they cannot be called from outside the package at all (not even via `EXECUTE` grants, since there's no public entry point to grant on), they naturally limit what an external caller (or an injected/malicious call) can directly reach, complementing techniques like `ACCESSIBLE BY` whitelisting at the package level.
- The **forward declaration** requirement (spec-only for the subprogram, ahead of its use, inside the body) is a distinctly PL/SQL-specific rule and a common source of "identifier must be declared" compile errors for developers unfamiliar with it — remember it applies specifically to *private* subprograms referenced *before* their body appears in the package body's top-to-bottom order.
- Public package-level **variables and cursors** follow the same public/private split as subprograms based on whether they're declared in the spec — though exposing mutable public package variables directly is generally considered poor practice (prefer public **functions** to read/modify state, keeping the actual variable private) since it protects package state from uncontrolled external modification.

### Exercise Questions

**Q1.** A package body defines a helper function `calc_tax` that is never declared in the package specification. Can code in another schema's procedure call `pkg_name.calc_tax(...)`?
> **A:** No. Since `calc_tax` was never declared in the package **specification**, it is private by definition — it exists only within the package body's internal implementation and cannot be referenced from outside the package at all, regardless of what privileges the caller holds on the package itself (`EXECUTE` on the package only grants access to whatever's actually declared public in the spec).

**Q2.** Within a package body, `proc_a` (defined first, near the top of the body) calls `proc_b` (a private helper defined later, near the bottom of the body). Will this compile without any special handling? If not, what's needed?
> **A:** No, it will not compile as-is — PL/SQL requires private subprograms to be **known before use**, reading the package body top-to-bottom, and `proc_b`'s body appears after `proc_a`'s. The fix is to add a **forward declaration** for `proc_b` (its signature only, ending in a semicolon, with no body) placed before `proc_a`'s definition, so that by the time `proc_a`'s body is compiled, `proc_b` is already a known identifier; `proc_b`'s actual implementation can still physically appear later in the body.

**Q3.** Why might a package designer choose to expose a public **function** to read a package variable's value, rather than declaring the variable itself as public in the package specification?
> **A:** Keeping the actual variable **private** (declared only in the body) and exposing a public **read function** (and, if needed, a controlled "setter" procedure) preserves encapsulation: external callers can only interact with the state through the sanctioned interface, which can enforce validation, logging, or business rules on any read/write. A public variable declared directly in the spec could be modified by any caller with no such control, and any internal change to how that state is represented would risk breaking external code that referenced the variable directly — exactly the kind of coupling private helpers and controlled accessors are meant to avoid.

---

## Quick Cross-Topic Summary Table

| Topic | Key Mechanism | Scope/Lifetime |
|---|---|---|
| PL/SQL Engine | PL/SQL statement executor + SQL statement executor | Per statement execution |
| Context Switching | Transfer of control between the two executors | Per SQL call from PL/SQL (or vice versa) |
| PL/SQL Optimizer | `PLSQL_OPTIMIZE_LEVEL` (0–3), `PRAGMA INLINE` | Compile time |
| Definer/Invoker Rights | `AUTHID DEFINER` (default) / `AUTHID CURRENT_USER` | Per program unit (package/procedure/function) |
| Package State | Package-level variables/cursors | Per session (unless `SERIALLY_REUSABLE`) |
| Package Initialization Block | Trailing unnamed `BEGIN...END` in package body | Once per session, on first reference |
| Private vs. Public Subprograms | Declared in spec (public) vs. body-only (private) | Compile-time visibility rule |