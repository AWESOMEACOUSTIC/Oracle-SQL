# Topic 5: Compound Triggers, Enable and Disable Triggers, INSTEAD OF Triggers

> **Syllabus position:** Topic 5 of 6 (Triggers syllabus) — the last new-concept topic before Topic 6's combined assessment
> **Builds on:** Topics 1–4 (statement/row-level triggers, `:NEW`/`:OLD`, timing, events, execution order)
> **Three related but distinct pieces in this topic:** compound triggers (a structural upgrade to what you already know), enabling/disabling (an operational concern), and `INSTEAD OF` triggers in full depth (Topic 3 only classified them — this is where the mechanics live, per your syllabus).

---

# Part A: Compound Triggers

## 1. Concept

A **compound trigger** is a single trigger object that can hold **up to four separate timing-point sections** for the same table and DML event(s), all under one `CREATE TRIGGER` statement: `BEFORE STATEMENT`, `BEFORE EACH ROW`, `AFTER EACH ROW`, and `AFTER STATEMENT`. It also has its own **shared declaration section** at the top, visible to every section that follows — giving you natural, built-in state-sharing between timing phases, without a separate package.

## 2. Purpose — What This Directly Solves

This topic exists to resolve two specific pain points you've already hit head-on in this course:

1. **Topic 1, Exercise 7's clunky workaround.** Needing to share a running count between a `BEFORE STATEMENT` reset, a `BEFORE ROW` increment, and an `AFTER STATEMENT` log meant creating a whole separate **package** just to hold one shared variable, plus three separate trigger objects — four database objects for one logical rule.
2. **Topic 2's mutating-table restriction.** A row-level trigger cannot query its own table mid-statement (`ORA-04091`). But by the time an `AFTER STATEMENT` section runs, every row-level change for that statement is already complete — the table is no longer mutating. Compound triggers let you **collect** what you need during the row-level sections (cheaply, without querying the table) and defer the actual cross-row query to `AFTER STATEMENT`, where it's completely legal.

## 3. What Real-World Problem It Solves

- "Validate that no department's average salary exceeds policy after this batch of updates" — impossible from a plain row-level trigger (mutating table), straightforward from a compound trigger's `AFTER STATEMENT` section.
- Any per-statement aggregate (row counts, sums) computed via a shared variable, with **no separate package required**.
- Any single, coherent business rule that genuinely has both a per-row part and a per-statement part.

## 4. When to Use / When NOT to Use

**Use a compound trigger when:**
- Logic naturally spans both a per-row part and a per-statement part of the **same** rule.
- You need to safely aggregate/query over the rows just changed by the current statement.
- You'd otherwise need multiple separate trigger objects sharing state via a package, and would rather have one self-contained object.

**Don't use one when:**
- The logic has no shared state and no mutating-table concern — a plain, simple trigger is more idiomatic.
- Only **one** timing point is actually needed — there's no benefit to the four-section scaffold for something that's genuinely single-phase.

## 5. Syntax

```sql
CREATE [OR REPLACE] TRIGGER trigger_name
FOR {INSERT | UPDATE [OF column_list] | DELETE} [OR ...]
ON table_name
COMPOUND TRIGGER

   -- shared declaration section: variables, types, constants — visible to every section below

   BEFORE STATEMENT IS
   BEGIN
      ...
   END BEFORE STATEMENT;

   BEFORE EACH ROW IS
   BEGIN
      ...
   END BEFORE EACH ROW;

   AFTER EACH ROW IS
   BEGIN
      ...
   END AFTER EACH ROW;

   AFTER STATEMENT IS
   BEGIN
      ...
   END AFTER STATEMENT;

END trigger_name;
/
```

**Key syntax notes:**
- The header uses **`FOR`**, not `BEFORE`/`AFTER` — since a compound trigger can hold *both* timing categories at once, the header itself deliberately doesn't commit to one.
- **All four sections are optional** — declare only the ones you actually need.
- Each section has its own tiny `BEGIN...END`, explicitly named to match (`END BEFORE STATEMENT;`, etc.), and the whole trigger closes with `END trigger_name;`.
- **A `WHEN` clause is not supported on the compound trigger header.** Conditional filtering must be written as an `IF` inside the relevant row section's body instead.
- `:NEW`/`:OLD` are available in `BEFORE EACH ROW`/`AFTER EACH ROW`, with exactly the same write rules from Topic 2 (`:NEW` writable only in `BEFORE EACH ROW`) — but are **not** available in the `BEFORE STATEMENT`/`AFTER STATEMENT` sections, since those have no row context, exactly like ordinary statement-level triggers.
- `INSERTING`/`UPDATING`/`DELETING` work identically in any section.

## 6. A Genuine Advantage Over the Package Workaround — Automatic Per-Statement Reset

The compound trigger's shared declaration section is **instantiated fresh for every triggering statement** and does not persist between statements — unlike a package-level variable, which persists for the life of the session unless explicitly reset. This is exactly why Topic 1, Exercise 7's package-based version needed its own explicit `BEFORE STATEMENT` reset step (`g_row_count := 0;`) — because a package variable *would* otherwise carry over incorrectly from one statement to the next. A compound trigger's shared state resets automatically, by construction. You'll still often want an explicit reset for clarity (as shown below), but the *risk* of forgetting one and getting silently wrong accumulated data across statements is structurally lower with a compound trigger's own state than with a package variable.

## 7. Worked Example — Revisiting Topic 1's Exercise 7, Dramatically Simplified

Recall Topic 1, Exercise 7 needed a package plus three separate triggers just to log a per-statement row count and time-window flag. Here's the same logic as **one** compound trigger:

```sql
CREATE OR REPLACE TRIGGER trg_price_update_summary
FOR UPDATE ON products
COMPOUND TRIGGER

   g_row_count PLS_INTEGER := 0;

   BEFORE STATEMENT IS
   BEGIN
      g_row_count := 0;
   END BEFORE STATEMENT;

   BEFORE EACH ROW IS
   BEGIN
      g_row_count := g_row_count + 1;
   END BEFORE EACH ROW;

   AFTER STATEMENT IS
   BEGIN
      INSERT INTO audit_log (table_name, operation, rows_affected, in_window, action_time, action_by)
      VALUES ('PRODUCTS', 'UPDATE', g_row_count,
              CASE WHEN TO_CHAR(SYSDATE, 'HH24') BETWEEN '08' AND '20' THEN 'Y' ELSE 'N' END,
              SYSDATE, USER);
   END AFTER STATEMENT;

END trg_price_update_summary;
/
```
One object, `g_row_count` declared once, automatically shared and automatically fresh per statement — no package needed at all.

## 8. Worked Example — Cleanly Solving Topic 2's Mutating-Table Trap

Recall Topic 2's forbidden example: a `BEFORE INSERT ON employees FOR EACH ROW` trigger trying to `SELECT AVG(salary) FROM employees` mid-`INSERT`, raising `ORA-04091`. Here's the same intent, done correctly:

```sql
CREATE OR REPLACE TRIGGER trg_check_dept_avg_salary
FOR INSERT ON employees
COMPOUND TRIGGER

   TYPE t_dept_list IS TABLE OF employees.department_id%TYPE;
   g_depts_touched t_dept_list := t_dept_list();

   AFTER EACH ROW IS
   BEGIN
      g_depts_touched.EXTEND;
      g_depts_touched(g_depts_touched.LAST) := :NEW.department_id;
   END AFTER EACH ROW;

   AFTER STATEMENT IS
      v_avg_salary employees.salary%TYPE;
   BEGIN
      FOR i IN 1 .. g_depts_touched.COUNT LOOP
         SELECT AVG(salary) INTO v_avg_salary
         FROM employees
         WHERE department_id = g_depts_touched(i);

         IF v_avg_salary > 200000 THEN
            RAISE_APPLICATION_ERROR(-20070,
               'Average salary in department ' || g_depts_touched(i) || ' exceeds policy limit after this batch.');
         END IF;
      END LOOP;
   END AFTER STATEMENT;

END trg_check_dept_avg_salary;
/
```
`AFTER EACH ROW` only ever reads `:NEW.department_id` (no query against `employees` at all — cheap, and no mutating-table concern). Only in `AFTER STATEMENT` — once every row for this `INSERT` has already been written, and `employees` is no longer mutating — does the trigger run `SELECT AVG(salary)...`, which is now completely legal. If the check fails, `RAISE_APPLICATION_ERROR` still rolls back the **entire** statement, including every row just inserted — consistent with Topic 3's "`AFTER` isn't unconditionally final" lesson.

*(Minor honest note: this could insert the same department multiple times into `g_depts_touched` if several employees in the same department are inserted in one statement, causing a redundant repeat check. Not incorrect — just a small, easy refinement opportunity, e.g., checking `g_depts_touched` for the value before adding it.)*

## 9. Common Mistakes and Misconceptions

1. Trying to add a `WHEN` clause to a compound trigger's header — not supported; filter inside the relevant section's body instead.
2. Assuming shared variables persist **across** separate statements — they don't; they're fresh for each triggering statement's complete execution.
3. Forgetting each section's own named `END` (`END BEFORE STATEMENT;`, not just `END;`) — a syntax error.
4. Trying to reference `:NEW`/`:OLD` inside `BEFORE STATEMENT`/`AFTER STATEMENT` sections — not available; no row context there.
5. **Attempting a cross-row query against the trigger's own table inside `BEFORE EACH ROW`/`AFTER EACH ROW`** — still restricted by the mutating-table rule. Compound triggers don't lift this restriction for the row sections themselves; they give you a clean way to **defer** the risky query to `AFTER STATEMENT`, where it's always been legal.
6. Assuming a compound trigger changes *when* things fire relative to constraints or other triggers — it doesn't. Its four sections fire at exactly the same points in Topic 4's canonical sequence as if they were four separate ordinary triggers; it only changes *how* they're packaged and how state is shared.

## 10. Edge Cases

- A compound trigger implementing only **one** section (e.g., just `AFTER STATEMENT`) is perfectly valid.
- A zero-row statement: `BEFORE STATEMENT`/`AFTER STATEMENT` sections still fire once each; `BEFORE EACH ROW`/`AFTER EACH ROW` don't fire at all — the same zero-row rule from Topics 1 & 4, now applied specifically to a compound trigger's row sections.
- A compound trigger can participate in a `FOLLOWS`/`PRECEDES` chain (Topic 4) alongside ordinary triggers, referenced by its own trigger name like any other.

---

# Part B: Enable and Disable Triggers

## 11. Concept

Triggers are **enabled by default** the moment they're created — they fire automatically. Oracle lets you explicitly **disable** a trigger (stop it from firing, without losing its definition) and later re-**enable** it.

## 12. Purpose / Why This Exists

- **Bulk/historical data loads.** Firing per-row validation or audit triggers during a large, already-vetted migration can be enormously slow — and can be *logically wrong*, not just slow: an audit trail meant to capture genuine business events shouldn't record "we loaded 2 million historical rows during a migration" as if they were live activity, and a "set `created_by := USER`" trigger would incorrectly overwrite historical attribution a migration is specifically trying to *preserve*.
- **Diagnosing a production issue** a specific trigger might be causing, without permanently removing its logic (which `DROP`+recreate would risk, if not scripted carefully).
- **Maintenance windows** where certain automatic side effects (e.g., notification triggers) should be temporarily suspended.

## 13. Syntax

```sql
ALTER TRIGGER trigger_name DISABLE;
ALTER TRIGGER trigger_name ENABLE;
```
Table-wide (every trigger on a table at once):
```sql
ALTER TABLE table_name DISABLE ALL TRIGGERS;
ALTER TABLE table_name ENABLE ALL TRIGGERS;
```
Checking status:
```sql
SELECT trigger_name, status FROM user_triggers WHERE table_name = 'PRODUCTS';
```
(`status` is `'ENABLED'` or `'DISABLED'`.)

## 14. Important Practical Notes

- **Disabling does not drop the trigger.** Its full definition remains intact in the data dictionary; it simply stops firing until re-enabled. This is the key distinction from `DROP TRIGGER`, which permanently removes it.
- **A disabled trigger stays disabled until explicitly re-enabled** — it does **not** automatically re-enable at the end of a session, statement, or transaction. Forgetting to re-enable after a migration is a real, serious, and surprisingly common operational incident: a trigger gets switched off for a "one-time" load and silently stays off for months, with nobody noticing until the audit trail or validation it used to provide is found to have a gap.
- **`CREATE OR REPLACE TRIGGER` resets status to `ENABLED`**, even when recreating the exact same logic, regardless of its prior disabled state. A routine "let me just tweak this trigger slightly" deployment can inadvertently re-enable something that was deliberately turned off.
- `ALTER TABLE ... DISABLE ALL TRIGGERS` is a blunt, broad tool — it disables *every* trigger on the table (row, statement, compound, `INSTEAD OF` — everything) at once. `ALTER TRIGGER trigger_name DISABLE` gives finer control over exactly which one to suspend.
- **DDL and PL/SQL:** `ALTER TRIGGER` is a DDL statement, and DDL isn't part of the static SQL subset PL/SQL can run directly — from inside a PL/SQL block, it must be run via `EXECUTE IMMEDIATE` (dynamic SQL). As a standalone statement in a script (not inside a `BEGIN...END`), it can be run directly.

## 15. Example
```sql
BEGIN
   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_set_last_modified DISABLE';
   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_audit_employees_stmt DISABLE';

   -- ... bulk INSERT/UPDATE of historical records, preserving original attribution ...

   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_set_last_modified ENABLE';
   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_audit_employees_stmt ENABLE';
END;
/
```

## 16. Common Mistakes and Misconceptions

1. Forgetting to re-enable after a maintenance/migration window — a silent, potentially long-lasting loss of trigger protection.
2. Assuming `DISABLE` removes the trigger's logic — it doesn't; it's purely a status flag.
3. Assuming a disabled trigger auto-re-enables at session/transaction end — it doesn't; the status is persistent at the object level.
4. Using `ALTER TABLE ... DISABLE ALL TRIGGERS` when only one specific trigger needed to be disabled — unnecessarily broad, potentially suspending unrelated logic that should have kept running.
5. Not realizing `CREATE OR REPLACE TRIGGER` resets status to `ENABLED`.

## 17. Edge Cases

- Disabling an already-disabled trigger, or enabling an already-enabled one — no error; both are idempotent no-ops.
- Disabling one trigger within a `FOLLOWS`/`PRECEDES` chain — it simply doesn't fire; the other triggers in the chain still fire at their normal relative positions, unaffected.

---

# Part C: INSTEAD OF Triggers (Full Depth)

## 18. Concept

An `INSTEAD OF` trigger is attached to a **view** (never a table) and **completely replaces** the DML statement issued against that view. The view's underlying data is not modified "automatically" — the trigger's body **is** what executes, and it's entirely up to that code to perform whatever `INSERT`/`UPDATE`/`DELETE` against the appropriate **base table(s)** achieves the intended effect.

## 19. Why This Exists / What Problem It Solves

Some views are **not inherently updatable** by Oracle automatically — specifically ones involving joins across multiple tables, `GROUP BY`/aggregate functions, `DISTINCT`, set operators (`UNION`, `INTERSECT`, `MINUS`), or certain other constructs. A plain `INSERT`/`UPDATE`/`DELETE` issued directly against such a view fails outright, since Oracle cannot automatically and unambiguously determine what base-table change corresponds to a requested change against a derived, combined result set.

`INSTEAD OF` triggers give you a way to **explicitly define** what a DML statement against such a view should do — making the view "updatable by custom logic," even though it isn't automatically so. The common real use case: presenting a simplified, "flattened" view to an application or admin tool (e.g., a view joining a product and its current price tier into one row), while the actual schema stays properly normalized underneath. The application gets to work with something that *looks* like a single table; the `INSTEAD OF` trigger quietly translates its DML into the correct multi-table operations behind the scenes.

## 20. When to Use / When NOT to Use

**Use `INSTEAD OF` when:**
- The view isn't inherently updatable (join, aggregate, `DISTINCT`, `GROUP BY`, set operator), and DML against it needs to actually do something meaningful.
- You want to present a simplified interface to consumers while keeping the real schema normalized.

**Don't use it when:**
- The view is already simple and inherently updatable (single table, no aggregation) — as Topic 3's Example E cautioned, ordinary DML against such a view already works directly; adding an `INSTEAD OF` trigger here is unnecessary machinery.

## 21. Syntax

```sql
CREATE [OR REPLACE] TRIGGER trigger_name
INSTEAD OF {INSERT | UPDATE [OF column_list] | DELETE} [OR ...]
ON view_name
FOR EACH ROW
BEGIN
   -- explicit DML against the appropriate base table(s), using :NEW / :OLD
END;
/
```

**Restrictions and rules:**
- Valid **only** `ON` a view — never a table.
- Always row-level in effect; `FOR EACH ROW` is conventionally always written, since there's no meaningful statement-level `INSTEAD OF`.
- `:NEW`/`:OLD` reflect the **view's own column list** (as defined by the view's `SELECT`), not any one base table's columns directly — the trigger body is responsible for mapping those values onto the correct base table(s) and columns.
- Cannot combine `INSTEAD OF` with `BEFORE`/`AFTER` on the same trigger (Topic 3).
- **Compound trigger syntax cannot target `INSTEAD OF`.** Compound triggers are specifically `FOR ... ON table_name` with `BEFORE`/`AFTER` `STATEMENT`/`ROW` sections — a fundamentally different, single-body, view-only structure. The two mechanisms in this topic serve different targets (tables needing multi-phase logic vs. views needing DML redirection) and don't combine.
- A view needs its **own** `INSTEAD OF` trigger (or triggers) for **each** DML operation it should support — a view with only an `INSTEAD OF INSERT` trigger still can't be `UPDATE`d or `DELETE`d; those still fail, since no trigger exists to define what they should do.

## 22. Worked Example — The Classic Join-Based View

```sql
CREATE OR REPLACE VIEW customer_shipping_vw AS
SELECT c.customer_id, c.customer_name, c.email,
       a.address_id, a.street, a.city, a.postal_code
FROM customers c
JOIN addresses a ON a.customer_id = c.customer_id AND a.is_default = 'Y';
```
This view joins two tables — **not** inherently updatable. A direct `UPDATE customer_shipping_vw SET city = 'Chennai' WHERE customer_id = 501;` fails without an `INSTEAD OF` trigger.

### UPDATE
```sql
CREATE OR REPLACE TRIGGER trg_customer_shipping_update
INSTEAD OF UPDATE ON customer_shipping_vw
FOR EACH ROW
BEGIN
   UPDATE customers
   SET customer_name = :NEW.customer_name, email = :NEW.email
   WHERE customer_id = :OLD.customer_id;

   UPDATE addresses
   SET street = :NEW.street, city = :NEW.city, postal_code = :NEW.postal_code
   WHERE address_id = :OLD.address_id;
END;
/
```
One logical `UPDATE` against the view is translated into **two** separate `UPDATE`s against the real base tables — the application never needs to know the view is really backed by a join.

### INSERT
```sql
CREATE OR REPLACE TRIGGER trg_customer_shipping_insert
INSTEAD OF INSERT ON customer_shipping_vw
FOR EACH ROW
DECLARE
   v_new_customer_id customers.customer_id%TYPE;
BEGIN
   INSERT INTO customers (customer_id, customer_name, email)
   VALUES (customers_seq.NEXTVAL, :NEW.customer_name, :NEW.email)
   RETURNING customer_id INTO v_new_customer_id;

   INSERT INTO addresses (address_id, customer_id, street, city, postal_code, is_default)
   VALUES (addresses_seq.NEXTVAL, v_new_customer_id, :NEW.street, :NEW.city, :NEW.postal_code, 'Y');
END;
/
```
`RETURNING ... INTO` captures the newly generated `customer_id` from the first `INSERT` (via a sequence) so it can be used correctly as the foreign key in the second `INSERT` — the standard pattern for inserting related rows across two tables in one logical operation.

### DELETE — A Deliberate Business Decision
```sql
CREATE OR REPLACE TRIGGER trg_customer_shipping_delete
INSTEAD OF DELETE ON customer_shipping_vw
FOR EACH ROW
BEGIN
   DELETE FROM addresses WHERE address_id = :OLD.address_id;
   -- Deliberately NOT deleting the customer record itself: "deleting" through this
   -- view only removes the shipping-address association, not the customer.
END;
/
```
This is worth pausing on: for a simple, single-table view, what `DELETE` means is obvious. For a join-based view, it **is not automatic** — you have to explicitly decide (and document) what deleting "through" the view actually does to the real schema. Here, the decision is that removing a row from this view removes only the address association, never the customer — a business choice, not something Oracle infers for you.

## 23. Detailed Explanation

- The trigger body is **100% responsible for correctness** — there's no automatic partial fallback. If the logic is incomplete (e.g., forgets to handle one of the view's columns), the operation against the view will simply do the wrong thing, or nothing for that column, rather than raising an error — because from Oracle's perspective, the `INSTEAD OF` trigger's execution **is** the complete, successful handling of that statement.
- Since there's no separate `BEFORE` phase for `INSTEAD OF` — it's the *only* phase — validation and rejection logic belongs right there in the same body: `RAISE_APPLICATION_ERROR` works exactly as it does anywhere else, and rejects the operation against the view.
- **Constraints on the base tables still apply normally** to whatever DML the trigger's body actually issues. If the trigger's `INSERT` into `customers` violates a `NOT NULL` constraint there, that failure surfaces normally, aborting the trigger (and therefore the original view-level DML), exactly like any ordinary constraint violation.

## 24. Common Mistakes and Misconceptions

1. Creating an `INSTEAD OF` trigger on a table — invalid; views only.
2. Assuming one `INSTEAD OF` trigger covers all DML operations on a view — each operation (`INSERT`, `UPDATE`, `DELETE`) needs its own handling, whether as separate triggers or one combined-event trigger (Topic 4's combine/separate reasoning applies here too).
3. Assuming `:NEW`/`:OLD` map directly to one base table's columns — they reflect the **view's own** column list; the trigger must explicitly map them.
4. Not making an explicit, deliberate decision about what `DELETE` (or a partial `UPDATE`) actually means for a multi-table view — accidentally deleting or updating a broader scope of data than intended.
5. Adding an unnecessary `INSTEAD OF` trigger to an already-simple, inherently-updatable view.
6. Assuming compound trigger sections (`BEFORE STATEMENT`/`EACH ROW`/etc.) can be combined with `INSTEAD OF` — not possible; structurally separate mechanisms.

## 25. Edge Cases

- A view with `INSTEAD OF` triggers for `INSERT` and `UPDATE` but not `DELETE` → `DELETE` against that view still fails, since no trigger defines its behavior.
- An intentionally empty `INSTEAD OF` trigger body (`BEGIN NULL; END;`) is technically valid — the operation "succeeds" from the caller's perspective but has no effect on any base table whatsoever. Unusual, but a legitimate way to make a view's DML silently a no-op if that were ever genuinely desired.
- Constraint violations on base-table DML issued from inside the trigger propagate normally and abort the original view-level statement.

## 26. How This Relates to Other Topics

- Builds directly on Topic 3's classification-level introduction, now with full mechanics.
- Uses `:NEW`/`:OLD` exactly as taught in Topic 2, applied to a view's column list instead of a table's.
- Uses `RAISE_APPLICATION_ERROR` (Topic 1) identically for rejecting an operation.
- Structurally distinct from compound triggers (Part A) — different target, different form, and the two never combine.

---

## Things You Must Remember

- **Compound triggers:** one object, up to four optional sections, shared state resets automatically per statement, no `WHEN` clause on the header, `:NEW`/`:OLD` only in the row sections, and the mutating-table restriction still applies *within* the row sections — the fix is deferring the risky query to `AFTER STATEMENT`, not bypassing the rule.
- **Enable/disable:** `ALTER TRIGGER ... DISABLE/ENABLE` (or `ALTER TABLE ... DISABLE/ENABLE ALL TRIGGERS`); disabling never drops the definition; status persists until explicitly changed; `CREATE OR REPLACE` resets to `ENABLED`; DDL from PL/SQL needs `EXECUTE IMMEDIATE`.
- **`INSTEAD OF`:** views only, always row-level, `:NEW`/`:OLD` reflect the view's columns, replaces the DML entirely (nothing happens "underneath" automatically), each operation needs its own explicit handling, and base-table constraints still apply to whatever the trigger body actually does.

## How to Recognize These Concepts

- "one rule with both a per-row part and a per-statement part," or "validate an aggregate across the table after a batch of changes" → compound trigger.
- "suspend this trigger for a bulk load / maintenance window, then turn it back on" → enable/disable.
- "this view isn't directly editable, but users need to insert/update/delete through it anyway" → `INSTEAD OF`.

---

# Exercises — With Answers and Reasoning

## Easy (Syntax & Basics)

### Exercise 1 (Compound)
**Task:** Write a compound trigger that counts how many rows were deleted in a single `DELETE` statement on `orders` and logs the count via `AFTER STATEMENT`.

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_orders_delete_count
FOR DELETE ON orders
COMPOUND TRIGGER

   g_deleted_count PLS_INTEGER := 0;

   BEFORE STATEMENT IS
   BEGIN
      g_deleted_count := 0;
   END BEFORE STATEMENT;

   BEFORE EACH ROW IS
   BEGIN
      g_deleted_count := g_deleted_count + 1;
   END BEFORE EACH ROW;

   AFTER STATEMENT IS
   BEGIN
      INSERT INTO audit_log (table_name, operation, rows_affected, action_time, action_by)
      VALUES ('ORDERS', 'DELETE', g_deleted_count, SYSDATE, USER);
   END AFTER STATEMENT;

END trg_orders_delete_count;
/
```
**Reasoning:** Directly parallels Section 7's worked example — a shared counter, reset once, incremented once per row, and read once at the end. No package required.

---

### Exercise 2 (Compound — spot the bug)
**Task:** Find the bugs.
```sql
CREATE OR REPLACE TRIGGER trg_buggy
FOR INSERT ON orders
COMPOUND TRIGGER
   v_count PLS_INTEGER := 0;

   BEFORE STATEMENT IS
   BEGIN
      v_count := 0;
      DBMS_OUTPUT.PUT_LINE('New order for: ' || :NEW.customer_id);
   END BEFORE STATEMENT;

   AFTER STATEMENT IS
   BEGIN
      DBMS_OUTPUT.PUT_LINE('Total rows: ' || v_count);
   END AFTER STATEMENT;
END trg_buggy;
/
```

**Answer — two bugs:**
1. **`:NEW` is referenced inside `BEFORE STATEMENT`.** This section has no row context — `:NEW` simply doesn't exist there, exactly like an ordinary statement-level trigger. Compile error.
2. **`v_count` is declared but never incremented anywhere.** There's no `BEFORE EACH ROW` or `AFTER EACH ROW` section at all, so nothing ever touches `v_count` after it's reset to `0` — `AFTER STATEMENT` will always print `Total rows: 0`, regardless of how many rows were actually inserted.

**Fix:**
```sql
CREATE OR REPLACE TRIGGER trg_buggy
FOR INSERT ON orders
COMPOUND TRIGGER
   v_count PLS_INTEGER := 0;

   BEFORE STATEMENT IS
   BEGIN
      v_count := 0;
   END BEFORE STATEMENT;

   BEFORE EACH ROW IS
   BEGIN
      v_count := v_count + 1;
      DBMS_OUTPUT.PUT_LINE('New order for: ' || :NEW.customer_id);
   END BEFORE EACH ROW;

   AFTER STATEMENT IS
   BEGIN
      DBMS_OUTPUT.PUT_LINE('Total rows: ' || v_count);
   END AFTER STATEMENT;
END trg_buggy;
/
```

---

### Exercise 3 (Enable/Disable)
**Task:** Write the statements needed to disable a trigger named `trg_audit_orders`, run a bulk load, then re-enable it.

**Solution:**
```sql
ALTER TRIGGER trg_audit_orders DISABLE;

-- ... bulk load statements here ...

ALTER TRIGGER trg_audit_orders ENABLE;
```
**Reasoning:** As standalone script statements (not inside a `BEGIN...END` block), `ALTER TRIGGER` can be run directly — no `EXECUTE IMMEDIATE` needed here, unlike Exercise 8, where the same logic is wrapped in a PL/SQL block.

---

### Exercise 4 (Enable/Disable)
**Task:** Write a query to check whether a specific trigger is currently enabled or disabled.

**Solution:**
```sql
SELECT trigger_name, status
FROM user_triggers
WHERE trigger_name = 'TRG_AUDIT_ORDERS';
```
**Reasoning:** `user_triggers` (or `all_triggers`/`dba_triggers` for broader scope) exposes a `status` column showing `'ENABLED'` or `'DISABLED'` — the standard way to verify a trigger's current state without needing to inspect its source.

---

### Exercise 5 (INSTEAD OF)
**Task:** Given
```sql
CREATE OR REPLACE VIEW employee_department_vw AS
SELECT e.employee_id, e.first_name, e.salary, d.department_id, d.department_name
FROM employees e
JOIN departments d ON d.department_id = e.department_id;
```
write an `INSTEAD OF UPDATE` trigger allowing `salary` to be updated through this view, while rejecting any attempt to change `department_name` through it.

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_emp_dept_update
INSTEAD OF UPDATE ON employee_department_vw
FOR EACH ROW
BEGIN
   IF :NEW.department_name != :OLD.department_name THEN
      RAISE_APPLICATION_ERROR(-20090, 'Department name cannot be changed through this view.');
   END IF;

   UPDATE employees
   SET salary = :NEW.salary
   WHERE employee_id = :OLD.employee_id;
END;
/
```
**Reasoning:** This is exactly the "explicit business decision" theme from Section 22's `DELETE` example, applied to `UPDATE` instead: renaming a department via one employee's row in this view doesn't make sense, so the trigger explicitly rejects that case with `RAISE_APPLICATION_ERROR`, right inside the same body — there's no separate `BEFORE` phase needed for validation, since `INSTEAD OF` is the only phase.

---

### Exercise 6 (INSTEAD OF — True/False)
**Task:** *"An `INSTEAD OF` trigger can call `RAISE_APPLICATION_ERROR` to reject the DML issued against the view, just like a `BEFORE` trigger can reject DML against a table."*

**Answer: True.** Exercise 5 demonstrates this directly. Since `INSTEAD OF` is the *only* phase for a view's DML (there's no separate `BEFORE`/`AFTER` split the way there is for tables), any rejection logic simply lives in the same trigger body — `RAISE_APPLICATION_ERROR` works identically to how it works anywhere else, aborting the operation from the caller's perspective.

---

## Intermediate (Reasoning & Edge Cases)

### Exercise 7 (Compound)
**Task:** A policy requires that after any bulk `UPDATE` to `salary`, no department's **total** salary budget (summed across *all* its employees — including ones not touched by this particular `UPDATE`) may exceed its allocated budget. Explain why a plain row-level trigger can't implement this, and provide a compound trigger that can.

**Answer — why a plain row-level trigger fails:** This genuinely requires running `SELECT SUM(salary) FROM employees WHERE department_id = ...` — an actual aggregate query against `employees` itself. Since `employees` is the table currently being updated, any row-level trigger (`BEFORE ROW` or `AFTER ROW`) attempting this mid-statement hits the mutating-table restriction (`ORA-04091`) directly, exactly as in Topic 2's original example — this isn't a case that can be solved with simple in-memory counting (like Topic 1, Exercise 7's row-count problem); it genuinely needs to query the table's true, aggregated state.

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_check_dept_budget
FOR UPDATE OF salary ON employees
COMPOUND TRIGGER

   TYPE t_dept_list IS TABLE OF employees.department_id%TYPE;
   g_depts_touched t_dept_list := t_dept_list();

   AFTER EACH ROW IS
   BEGIN
      g_depts_touched.EXTEND;
      g_depts_touched(g_depts_touched.LAST) := :NEW.department_id;
   END AFTER EACH ROW;

   AFTER STATEMENT IS
      v_total_budget employees.salary%TYPE;
      v_allocated    departments.budget%TYPE;
   BEGIN
      FOR i IN 1 .. g_depts_touched.COUNT LOOP
         SELECT SUM(e.salary), d.budget
         INTO v_total_budget, v_allocated
         FROM employees e
         JOIN departments d ON d.department_id = e.department_id
         WHERE e.department_id = g_depts_touched(i)
         GROUP BY d.budget;

         IF v_total_budget > v_allocated THEN
            RAISE_APPLICATION_ERROR(-20071,
               'Department ' || g_depts_touched(i) || ' salary total exceeds its allocated budget.');
         END IF;
      END LOOP;
   END AFTER STATEMENT;

END trg_check_dept_budget;
/
```
**Reasoning:** `AFTER EACH ROW` only records *which* departments were affected (a cheap read of `:NEW.department_id`, no query against `employees`). The genuine aggregate query runs once per affected department, safely, in `AFTER STATEMENT` — after all of this statement's salary changes have already been applied, so `SUM(salary)` correctly reflects the true, post-update total, including employees this particular statement never touched.

---

### Exercise 8 (Enable/Disable)
**Task:** A DBA disables `trg_audit_orders` for a Friday-evening migration, and the migration script crashes partway through, before reaching the re-enable statement. What real risk does this create, and how should the script be written differently to avoid it?

**Answer:** The real risk is that `trg_audit_orders` stays **silently disabled indefinitely** — since disabled status persists until explicitly changed, and nothing automatically restores it after a crash. This is a genuinely common, serious real-world incident pattern: an audit or validation trigger gets switched off for a "quick" migration, something goes wrong partway through, and nobody re-enables it — potentially for weeks or months, until someone eventually notices a gap in the audit trail (by which point it's too late to reconstruct).

**Fix — guarantee re-enabling even on failure**, using an exception handler:
```sql
BEGIN
   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_audit_orders DISABLE';

   -- ... migration DML here ...

   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_audit_orders ENABLE';
EXCEPTION
   WHEN OTHERS THEN
      EXECUTE IMMEDIATE 'ALTER TRIGGER trg_audit_orders ENABLE';
      RAISE;
END;
/
```
**Reasoning:** `ALTER TRIGGER` is DDL, so from inside a PL/SQL block it must run via `EXECUTE IMMEDIATE` (dynamic SQL) — it isn't part of the static SQL PL/SQL can execute directly. The `EXCEPTION WHEN OTHERS` handler re-enables the trigger **before** re-raising the original error, guaranteeing the trigger comes back on regardless of what goes wrong in the migration logic itself, rather than depending on the script reaching its final line successfully.

---

### Exercise 9 (INSTEAD OF)
**Task:** In Section 22's `trg_customer_shipping_insert` example, what happens if the `INSERT INTO customers` violates a `NOT NULL` constraint on `email`? Trace the outcome.

**Answer:** The `INSERT INTO customers` statement inside the trigger body fails with the standard `NOT NULL` constraint violation, exactly as it would if that `INSERT` had been issued directly and outside any trigger context. Since this failure happens inside the `INSTEAD OF` trigger's body — the *only* code handling this operation — the entire trigger execution aborts at that point, and the original `INSERT` issued against `customer_shipping_vw` fails as a whole. Crucially, the **second** `INSERT` (into `addresses`) never even runs, since execution never reaches it — so there's no risk of an orphaned `addresses` row with no matching `customers` row. This directly confirms the point from Section 23: constraints on the base tables apply completely normally to whatever DML the trigger body issues; `INSTEAD OF` doesn't bypass or weaken them in any way, it just relocates *where* the DML that triggers them comes from.

---

## Realistic Scenario

### Exercise 10
**Business Requirement:** *"The company's product catalog exposes a `product_pricing_vw` view to the pricing team's admin tool, joining `products` and `price_tiers` (each product has exactly one current price-tier row). The admin tool needs to `INSERT`, `UPDATE`, and `DELETE` through this view as if it were a single table — deleting through the view should remove only the price-tier association, never the product itself. Separately, the company has a hard policy that the average price across all products in any single category must never exceed \$500 after any batch of pricing changes; this must be enforced using fully up-to-date price data after the whole batch, not per-row while changes are still in progress. Finally, once a year, the finance team runs a large one-time historical repricing correction that intentionally needs to bypass the \$500 average check entirely — this must be possible without disabling the view's DML behavior or any other trigger."*

### Requirement Analysis
Three distinct pieces, each squarely one of this topic's three mechanisms: a non-updatable join-based view needing full `INSTEAD OF` handling for all three DML operations; a cross-row aggregate policy check needing a compound trigger; and a scoped, independently-controllable suspension of just that one policy check.

### Concepts to Consider
`INSTEAD OF` triggers on `product_pricing_vw` (Part C); a compound trigger on `price_tiers` enforcing the per-category average (Part A); and `ALTER TRIGGER ... DISABLE`/`ENABLE` scoped to *just* the compound trigger (Part B) — which, if the compound trigger is kept as its own independent object (as it naturally would be), already satisfies "without disabling the view's DML behavior or any other trigger" with zero extra design work.

### Solution

**The view and its `INSTEAD OF` triggers:**
```sql
CREATE OR REPLACE VIEW product_pricing_vw AS
SELECT p.product_id, p.product_name, p.category, pt.price_tier_id, pt.current_price
FROM products p
JOIN price_tiers pt ON pt.product_id = p.product_id;
```
```sql
CREATE OR REPLACE TRIGGER trg_product_pricing_insert
INSTEAD OF INSERT ON product_pricing_vw
FOR EACH ROW
DECLARE
   v_new_product_id products.product_id%TYPE;
BEGIN
   INSERT INTO products (product_id, product_name, category)
   VALUES (products_seq.NEXTVAL, :NEW.product_name, :NEW.category)
   RETURNING product_id INTO v_new_product_id;

   INSERT INTO price_tiers (price_tier_id, product_id, current_price)
   VALUES (price_tiers_seq.NEXTVAL, v_new_product_id, :NEW.current_price);
END;
/

CREATE OR REPLACE TRIGGER trg_product_pricing_update
INSTEAD OF UPDATE ON product_pricing_vw
FOR EACH ROW
BEGIN
   UPDATE products
   SET product_name = :NEW.product_name, category = :NEW.category
   WHERE product_id = :OLD.product_id;

   UPDATE price_tiers
   SET current_price = :NEW.current_price
   WHERE price_tier_id = :OLD.price_tier_id;
END;
/

CREATE OR REPLACE TRIGGER trg_product_pricing_delete
INSTEAD OF DELETE ON product_pricing_vw
FOR EACH ROW
BEGIN
   -- deliberate decision: only the price-tier association is removed
   DELETE FROM price_tiers WHERE price_tier_id = :OLD.price_tier_id;
END;
/
```

**The compound trigger enforcing the per-category average, on `price_tiers`:**
```sql
CREATE OR REPLACE TRIGGER trg_check_category_avg_price
FOR INSERT OR UPDATE OF current_price ON price_tiers
COMPOUND TRIGGER

   TYPE t_category_list IS TABLE OF products.category%TYPE;
   g_categories_touched t_category_list := t_category_list();

   AFTER EACH ROW IS
      v_category products.category%TYPE;
   BEGIN
      -- querying PRODUCTS here is fine: it's a different table from PRICE_TIERS,
      -- which is the one actually mutating during this statement
      SELECT category INTO v_category FROM products WHERE product_id = :NEW.product_id;
      g_categories_touched.EXTEND;
      g_categories_touched(g_categories_touched.LAST) := v_category;
   END AFTER EACH ROW;

   AFTER STATEMENT IS
      v_avg_price price_tiers.current_price%TYPE;
   BEGIN
      FOR i IN 1 .. g_categories_touched.COUNT LOOP
         SELECT AVG(pt.current_price)
         INTO v_avg_price
         FROM price_tiers pt
         JOIN products p ON p.product_id = pt.product_id
         WHERE p.category = g_categories_touched(i);

         IF v_avg_price > 500 THEN
            RAISE_APPLICATION_ERROR(-20080,
               'Average price in category ' || g_categories_touched(i) || ' exceeds $500 policy limit.');
         END IF;
      END LOOP;
   END AFTER STATEMENT;

END trg_check_category_avg_price;
/
```

**The annual correction load's scoped suspension:**
```sql
BEGIN
   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_check_category_avg_price DISABLE';

   -- ... large historical repricing correction UPDATE statements here ...

   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_check_category_avg_price ENABLE';
EXCEPTION
   WHEN OTHERS THEN
      EXECUTE IMMEDIATE 'ALTER TRIGGER trg_check_category_avg_price ENABLE';
      RAISE;
END;
/
```

### Detailed Explanation
Two "already works for free" moments are worth calling out explicitly here, both consistent with a recurring theme across this course:

1. **The `AFTER EACH ROW` section's query against `products`** (to find each touched row's category) is completely legal, with no mutating-table concern at all — because the compound trigger is defined `ON price_tiers`, and `products` is a *different* table. The restriction only ever applies to a trigger's own table (Topic 2); querying a related table from a row section has never been a problem.
2. **The scoped enable/disable requirement** ("without disabling the view's DML behavior or any other trigger") is satisfied automatically, purely because the average-price policy was correctly kept as its **own**, independent trigger object rather than folded into anything else. Disabling `trg_check_category_avg_price` has zero effect on `trg_product_pricing_insert`/`update`/`delete`, since they're entirely separate objects governing an entirely different table. No special design accommodation was needed to make this "possible" — it's simply a consequence of not conflating two unrelated rules into one trigger in the first place, echoing Part A's design guidance directly.

### Alternative Approach
The category check could instead run as a nightly batch job over the whole `price_tiers` table rather than a trigger firing on every relevant `UPDATE`/`INSERT` — trading immediate enforcement (a bad pricing change is rejected the moment it's attempted) for lower per-transaction overhead (no aggregate query runs on every single pricing change, only once per night). The trigger-based approach is the right fit here specifically because the policy explicitly needs to be enforced "after any batch of pricing changes," implying real-time rejection matters — a nightly batch would let a policy-violating price sit live in the system for up to a day before being caught.

### Edge Cases
- The annual correction load's `UPDATE`s, run through direct DML against `price_tiers` rather than through `product_pricing_vw`, never invoke the `INSTEAD OF` triggers at all — those only fire for DML issued *against the view*. This is expected and correct: the correction load is a backend/finance operation working directly on base tables, not an admin-tool interaction through the catalog view.
- If the correction load's `UPDATE`s are large enough to also touch many different categories, the `AFTER STATEMENT` section's loop (when the check is re-enabled for ordinary use afterward) will still correctly evaluate each one independently the next time it's active.

### Common Mistakes to Watch For
- Trying to fold the average-price check into one of the `INSTEAD OF` triggers on `product_pricing_vw` — wrong table entirely (the view spans two tables, but the aggregate check is specifically about `price_tiers`' own data across *all* products in a category, most of which won't be part of any single view-level DML operation at all), and it would also break the clean, independent disable/enable story.
- Forgetting that `INSTEAD OF` triggers only fire for DML against the **view** — direct DML against `price_tiers` (like the correction load) bypasses them entirely, which is expected here but could be a nasty surprise if a developer assumed the view's triggers provide some kind of universal safety net over the base tables.
- Omitting the `EXCEPTION WHEN OTHERS` re-enable safeguard on the correction load's script — reintroducing exactly the "forgot to turn it back on" risk from Exercise 8.

---

**End of Topic 5 — all new concepts from the Triggers syllabus are now covered.** Next file: `13-combined-practice-assessment-triggers.md`, which combines Topics 1–5 into realistic, mixed scenarios without naming which concept(s) each one requires.