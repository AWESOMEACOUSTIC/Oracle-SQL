# Topic 2: Row Level Trigger

> **Syllabus position:** Topic 2 of 6 (Triggers syllabus)
> **Builds on:** Topic 1 (statement-level triggers, `RAISE_APPLICATION_ERROR`)
> **Core content of this topic:** the `:NEW`/`:OLD` pseudorecords, the `WHEN` clause, and the mutating-table restriction — all essential, all covered in full depth below

---

## 1. Concept

A **row-level trigger** fires **once for every row** affected by the triggering DML statement — not once per statement, the way Topic 1's statement-level triggers do. It's declared with a `FOR EACH ROW` clause, and inside its body, you get access to two special **pseudorecords**, `:OLD` and `:NEW`, representing the row's column values **before** and **after** the change.

```sql
CREATE OR REPLACE TRIGGER trg_example
BEFORE UPDATE ON employees
FOR EACH ROW
BEGIN
   -- fires once per row being updated; :OLD.salary and :NEW.salary are both available here
END;
/
```

## 2. Purpose / Why It Exists

Most real business rules operate **per row**, not per statement: "don't let a promotion decrease someone's salary," "record exactly what changed and what it changed from/to," "auto-fill this column based on other columns in the same row." None of this is possible with a statement-level trigger, which has zero visibility into any individual row's data. Row-level triggers exist specifically to give you that visibility — the ability to look at (and, in some cases, adjust) each row's own values as it's being processed.

## 3. What Real-World Problem It Solves

- Validating a specific row's new value against a rule (e.g., a promotion's new salary must exceed the old one).
- Auto-populating audit columns (`last_modified_by`, `last_modified_date`) based on that exact row's own change.
- Detailed, field-level audit trails — capturing exactly which columns changed, and their old/new values, per row.
- Row-specific validation that depends only on that row's own columns (e.g., a row's `start_date` must be before its `end_date`).
- Deriving one column's value from others in the same row before it's saved (e.g., `total := quantity * unit_price`).

## 4. When to Use / When NOT to Use

**Use a row-level trigger when:**
- The logic needs a specific row's actual column values — old, new, or both.
- The validation or derivation is inherently per-row: it depends only on that row's own data.
- You need a detailed, per-row audit trail, not just "an operation happened."

**Be careful (and often use a different tool — Topic 5) when:**
- The logic needs to query the **same table** the trigger is on, to look at *other* rows, while the triggering statement is still running. This runs directly into the **mutating table** restriction, covered in full below — one of the most important things in this entire topic.

## 5. The Core of This Topic: `:NEW` and `:OLD`

### What They Are
`:NEW` and `:OLD` are pseudorecords, available only inside a row-level trigger body, that represent the column values of the row currently being processed. `:OLD` is the row's state **before** the change; `:NEW` is its state **after**.

### Availability by Event Type

| Event | `:OLD` | `:NEW` |
|---|---|---|
| `INSERT` | Not meaningful — every field is `NULL`, since there was no "before" row. | **Available and meaningful** — the row being inserted. |
| `UPDATE` | **Available and meaningful** — the row's values before the update. | **Available and meaningful** — the row's values after the update. |
| `DELETE` | **Available and meaningful** — the row's values before deletion. | Not meaningful — every field is `NULL`, since there's no "after" row. |

This table is worth memorizing outright — referencing the "wrong side" for a given event (e.g., expecting `:OLD` to hold useful data in a pure `INSERT` trigger) is one of the most common early mistakes with row-level triggers.

### Read vs. Write Access — Tied to Timing

- In a **`BEFORE` row-level trigger**, `:NEW` fields **can be modified** — whatever you assign to `:NEW.column` inside the trigger body becomes the actual value that gets written to the row. This is exactly how "auto-populate/derive a column" logic works.
- In an **`AFTER` row-level trigger**, `:NEW` fields are **read-only**. By the time an `AFTER` row trigger fires, the actual row write has already physically happened — assigning to `:NEW.column` at that point would have no effect on the stored data, and Oracle actively **raises `ORA-04084`** if you try. This is a frequently-asked, genuinely important distinction.
- `:OLD` is **always read-only**, in both `BEFORE` and `AFTER` triggers — it represents history, which makes sense to never allow changing.

### Syntax
Always accessed with a leading colon and dot notation: `:NEW.column_name`, `:OLD.column_name`.

## 6. The `WHEN` Clause

An optional clause on `CREATE TRIGGER` that restricts a row-level trigger's body to firing only for rows meeting a specific condition — evaluated **before** the PL/SQL engine even starts running the trigger body for that row.

```sql
CREATE OR REPLACE TRIGGER trg_salary_increase_only
BEFORE UPDATE OF salary ON employees
FOR EACH ROW
WHEN (NEW.salary > OLD.salary)
BEGIN
   -- only reached for rows where the new salary is genuinely higher than the old one
END;
/
```

**Critical syntax quirk:** inside the `WHEN` clause condition, you write `NEW.column` and `OLD.column` — **without the leading colon**. Everywhere else (the trigger body, `DECLARE` section), the colon is required (`:NEW.column`). This inconsistency is genuinely easy to get wrong and is one of the most common syntax mistakes with this feature.

**Why use `WHEN` instead of an equivalent `IF` inside the body?** Efficiency: the `WHEN` condition is evaluated by Oracle *before* invoking the PL/SQL engine for that row at all. If the condition is false, the trigger body never even starts executing for that row — avoiding the overhead of entering the PL/SQL block, which matters when a trigger fires across many rows in a bulk operation. `WHEN` is only valid on **row-level** triggers — there's no individual row for a statement-level trigger to evaluate `NEW`/`OLD` against.

## 7. `UPDATE OF column_list` — A Distinction Worth Getting Exactly Right

You can restrict a row-level `UPDATE` trigger to fire only when specific column(s) are named in the statement's `SET` clause:

```sql
CREATE OR REPLACE TRIGGER trg_salary_col_trigger
BEFORE UPDATE OF salary ON employees
FOR EACH ROW
BEGIN
   ...
END;
/
```

**This is a syntactic check, not a semantic one.** `UPDATE OF salary` fires whenever `salary` appears in the `SET` clause — **even if the statement reassigns it to its exact current value**, e.g., `UPDATE employees SET salary = salary WHERE ...`. Nothing about the actual data changed, but the trigger still fires, because `salary` was named in `SET`.

If you need "only when the value **actually** changes," you need a genuine comparison — either a `WHEN (NEW.salary != OLD.salary)` clause, or an equivalent check inside the body. Confusing these two is a very common, very real bug: assuming `UPDATE OF column` alone guarantees the value changed, when it only guarantees the column was *mentioned*.

## 8. The Mutating Table Restriction — `ORA-04091`

This is one of the most important restrictions in all of trigger programming, and it's specific to row-level triggers.

**The rule:** a row-level trigger generally **cannot query or modify the same table** that the triggering statement is currently in the middle of changing.

```sql
CREATE OR REPLACE TRIGGER trg_check_avg_salary
BEFORE INSERT ON employees
FOR EACH ROW
DECLARE
   v_avg_salary employees.salary%TYPE;
BEGIN
   SELECT AVG(salary) INTO v_avg_salary FROM employees;   -- querying its OWN table mid-INSERT
   IF :NEW.salary > v_avg_salary * 3 THEN
      RAISE_APPLICATION_ERROR(-20020, 'New salary cannot exceed 3x the average.');
   END IF;
END;
/
```
This trigger **compiles successfully** — the restriction is only detected at **runtime**. The moment an `INSERT` actually fires it, Oracle raises `ORA-04091: table is mutating, trigger/function may not see it`.

**Why this restriction exists:** while a statement is actively inserting/updating/deleting rows in a table, that table's data is in a **transitional, inconsistent state** — some rows have already been changed, others haven't yet. Oracle cannot guarantee a consistent, repeatable view of that table's data to a query issued from a row-level trigger firing mid-statement; allowing it could produce different results depending on exactly which row happens to be processed at that moment, breaking data-consistency guarantees.

**Important nuances:**
- This applies specifically to **row-level** triggers referencing their **own** table. A statement-level trigger (Topic 1) is generally not subject to this for its own table — an `AFTER` statement trigger runs once the whole statement is done (the table isn't "mutating" anymore), and a `BEFORE` statement trigger runs before any changes have begun (the table isn't mutating yet either). This is exactly the "orientation point" flagged at the end of Topic 1.
- It does **not** prevent a row-level trigger from freely querying **other, unrelated** tables — only its own.
- **A common but ineffective workaround attempt:** adding a `WHERE` clause to exclude the row currently being processed (e.g., `WHERE employee_id != :NEW.employee_id`), hoping that avoids the conflict. **This does not work.** The mutating-table check applies to the table as a whole being in a mutating state during the statement — not to specific rows — so no `WHERE`-clause trick avoids it.
- **The real fix** is a **compound trigger** (Topic 5), which lets you collect what you need during the row-level firings into shared, in-memory state, and then do the actual cross-row query safely in an `AFTER STATEMENT` section, once the table is no longer mutating. This is one of the two central motivations for compound triggers — the other being the "share state across timing phases" need previewed in Topic 1, Exercise 7.

## 9. Syntax (Full)

```sql
CREATE [OR REPLACE] TRIGGER trigger_name
{BEFORE | AFTER} {INSERT | UPDATE [OF column_list] | DELETE} [OR ...]
ON table_name
FOR EACH ROW
[WHEN (condition)]
[DECLARE
   -- local declarations]
BEGIN
   -- body; can reference :NEW.col, :OLD.col
[EXCEPTION
   ...]
END;
```

## 10. Types / Variations

| Variation | Description |
|---|---|
| Row-level `BEFORE` (single or combined event) | Fires before each row's change; `:NEW` is writable. |
| Row-level `AFTER` (single or combined event) | Fires after each row's change; `:NEW` is read-only. |
| `UPDATE OF column_list` | Syntactically restricts firing to statements naming specific column(s) in `SET`. |
| With a `WHEN` clause | Semantically restricts which rows actually run the trigger body. |
| Row-level `INSTEAD OF` | Exists (covered properly in Topics 3 & 5) — mentioned here only for completeness. |

## 11. Simple Examples

### Example A — Field-level audit (uses `:OLD` and `:NEW` together)
```sql
CREATE OR REPLACE TRIGGER trg_salary_change_audit
AFTER UPDATE OF salary ON employees
FOR EACH ROW
BEGIN
   INSERT INTO salary_audit (employee_id, old_salary, new_salary, changed_on, changed_by)
   VALUES (:OLD.employee_id, :OLD.salary, :NEW.salary, SYSDATE, USER);
END;
/
```

### Example B — Deriving/auto-populating a column (`:NEW` is writable — `BEFORE`)
```sql
CREATE OR REPLACE TRIGGER trg_set_last_modified
BEFORE INSERT OR UPDATE ON employees
FOR EACH ROW
BEGIN
   :NEW.last_modified_date := SYSDATE;
   :NEW.last_modified_by   := USER;
END;
/
```
Notice this works for **both** `INSERT` and `UPDATE` in one trigger, since `:NEW` is meaningful and writable for both events in a `BEFORE` row-level trigger.

### Example C — `WHEN` clause, no colon prefix
```sql
CREATE OR REPLACE TRIGGER trg_salary_increase_only
BEFORE UPDATE OF salary ON employees
FOR EACH ROW
WHEN (NEW.salary > OLD.salary)
BEGIN
   INSERT INTO salary_audit (employee_id, old_salary, new_salary, changed_on, changed_by)
   VALUES (:OLD.employee_id, :OLD.salary, :NEW.salary, SYSDATE, USER);
END;
/
```
The `WHEN` clause uses `NEW.salary`/`OLD.salary` (no colon); the body uses `:OLD.employee_id` (with colon) — the same pseudorecords, two different syntaxes, depending on which part of the trigger you're writing.

### Example D — Row-only validation (no mutating-table risk)
```sql
CREATE OR REPLACE TRIGGER trg_validate_date_range
BEFORE INSERT OR UPDATE ON employee_assignments
FOR EACH ROW
BEGIN
   IF :NEW.end_date IS NOT NULL AND :NEW.end_date < :NEW.start_date THEN
      RAISE_APPLICATION_ERROR(-20030, 'End date cannot be before start date.');
   END IF;
END;
/
```
This is completely safe from the mutating-table restriction, because it only ever looks at `:NEW`'s own fields — it never queries the table itself.

## 12. Detailed Explanation

- **Row processing order across one statement is not guaranteed.** If a single `UPDATE` affects 1,000 rows, the row-level trigger fires 1,000 times, but the *order* in which Oracle processes those rows is generally unspecified and shouldn't be assumed to match, say, primary-key order or insertion order. Trigger logic should be self-contained per row unless deliberately using shared state (see next point).
- **`:OLD`/`:NEW` don't carry over between row firings, but package-level variables do.** Each row's `:OLD`/`:NEW` context is independent — but if a trigger body writes to a package-level variable (as previewed in Topic 1, Exercise 7), that value *does* persist across the multiple row-level firings within the same statement's execution, since it's ordinary PL/SQL state, not tied to any one row.
- **Combined events work at row level too.** `BEFORE INSERT OR UPDATE ON table FOR EACH ROW` is valid, and the same `INSERTING`/`UPDATING`/`DELETING` predicates from Topic 1 apply here identically, useful whenever the two events need genuinely different per-row logic within one shared trigger body.

## 13. Common Mistakes and Misconceptions

1. Referencing `:OLD` in a pure `INSERT`-only trigger, expecting a previous value — every `:OLD` field is `NULL` for `INSERT`.
2. Referencing `:NEW` in a pure `DELETE`-only trigger — every `:NEW` field is `NULL` for `DELETE`.
3. Assigning to `:NEW.column` inside an `AFTER` row trigger — raises `ORA-04084`; only `BEFORE` row triggers can modify `:NEW`.
4. Using a colon inside the `WHEN` clause (`:NEW.col` instead of `NEW.col`) — a syntax error specific to that clause.
5. Assuming `UPDATE OF column_list` means "the column's value actually changed" — it only means the column was named in `SET`.
6. Directly querying or modifying the trigger's own table from inside a row-level trigger body — raises `ORA-04091` at runtime.
7. Assuming a `WHERE`-clause trick can dodge the mutating-table restriction — it can't; the restriction applies to the table as a whole, not specific rows.
8. Assuming rows are processed in a predictable order within one statement — not guaranteed.

## 14. Edge Cases

- `SET salary = salary` (reassigning a column to its own value) → `UPDATE OF salary` still fires (syntactic condition met), but `WHEN (NEW.salary != OLD.salary)` correctly does **not** fire the body — a clean demonstration of the syntactic-vs-semantic distinction from Section 7.
- A `NULL`-involved comparison, like a column going from a real value to `NULL` (or vice versa) — plain `!=` comparisons involving `NULL` evaluate to `UNKNOWN`, not `TRUE`, so a naive `WHEN (NEW.col != OLD.col)` can silently **miss** a change that involves `NULL` on either side. This needs explicit handling (see Exercise 7 for the standard trick).
- A bulk `UPDATE` affecting many rows fires the row-level trigger once per row, each with its own independent `:OLD`/`:NEW`, but any package-level counters used inside it accumulate normally across all of them within that one statement.

## 15. How This Relates to Other Topics

- Builds directly on Topic 1's `CREATE TRIGGER` scaffold, adding `FOR EACH ROW`.
- **Topic 3 (Timing)** explores `BEFORE` vs. `AFTER` in more depth, directly building on the write-access rule for `:NEW` introduced here.
- **Topic 4 (Events & Execution Order)** covers the full firing sequence when statement-level and row-level triggers coexist on the same table (`BEFORE STATEMENT` → `BEFORE ROW` (each) → `AFTER ROW` (each) → `AFTER STATEMENT`).
- **Topic 5 (Compound Triggers)** directly resolves the mutating-table restriction and the state-sharing need from Topic 1's Exercise 7 — both flagged as forward-pointing pain points introduced here.

---

## Things You Must Remember

- `FOR EACH ROW` = row-level; fires once per affected row, not once per statement.
- `:NEW` meaningful for `INSERT`/`UPDATE`, writable **only** in `BEFORE` triggers. `:OLD` meaningful for `UPDATE`/`DELETE`, **always** read-only. `INSERT`'s `:OLD` and `DELETE`'s `:NEW` are all `NULL`.
- `WHEN` clause: **no colon**, evaluated before the trigger body even runs, valid only on row-level triggers.
- `UPDATE OF column_list` is **syntactic** (checks `SET`-clause membership) — not semantic (does not check if the value actually changed).
- A row-level trigger **cannot** query or modify its own (currently mutating) table — `ORA-04091` — and no `WHERE`-clause trick avoids this. The real fix is a compound trigger (Topic 5).
- Row processing order across multiple rows in one statement is not guaranteed.

## How to Recognize This Concept

Think **row-level trigger** when a requirement:
- Mentions a "before/after value," "old vs. new," or wants something derived from that row's own columns.
- Needs validation or transformation specific to each individual row.
- Wants a detailed, per-row audit trail — not just "an operation happened."
- Talks about "when column X changes to/from a specific value" (a `WHEN`-clause signal).

---

# Exercises — With Answers and Reasoning

## Easy (Syntax & Basics)

### Exercise 1
**Task:** Write a row-level trigger that automatically sets `created_date` and `created_by` whenever a new row is inserted into an `orders` table.

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_orders_set_created
BEFORE INSERT ON orders
FOR EACH ROW
BEGIN
   :NEW.created_date := SYSDATE;
   :NEW.created_by   := USER;
END;
/
```

**Reasoning:** `BEFORE INSERT` is required, not `AFTER` — the whole point is to set these values *before* the row is actually written, so the write includes them. This is only possible because `:NEW` is writable in a `BEFORE` row-level trigger; the exact same code in an `AFTER INSERT` trigger would raise `ORA-04084`.

---

### Exercise 2
**Task:** Write a row-level trigger with a `WHEN` clause that fires (and logs) only when an employee's `department_id` **actually changes** — not merely when `department_id` appears in the `SET` clause.

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_dept_change_log
AFTER UPDATE OF department_id ON employees
FOR EACH ROW
WHEN (NEW.department_id != OLD.department_id)
BEGIN
   INSERT INTO dept_change_log (employee_id, old_dept, new_dept, changed_on)
   VALUES (:OLD.employee_id, :OLD.department_id, :NEW.department_id, SYSDATE);
END;
/
```

**Reasoning:** `UPDATE OF department_id` alone would fire this trigger even for `UPDATE employees SET department_id = department_id WHERE ...` — a no-op reassignment. Adding `WHEN (NEW.department_id != OLD.department_id)` filters that out at the semantic level, ensuring the log only records genuine changes, exactly the distinction covered in Section 7.

---

### Exercise 3
**Task:** Write a row-level trigger that prevents inserting or updating an `employees` row where `hire_date` is in the future.

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_validate_hire_date
BEFORE INSERT OR UPDATE ON employees
FOR EACH ROW
BEGIN
   IF :NEW.hire_date > SYSDATE THEN
      RAISE_APPLICATION_ERROR(-20031, 'Hire date cannot be in the future.');
   END IF;
END;
/
```

**Reasoning:** This validation only ever needs `:NEW.hire_date` — no query against the table itself — so there's no mutating-table concern here at all. `BEFORE` is the correct timing since the goal is to reject the bad value before it's ever written.

---

### Exercise 4
**Task:** Will this compile? `BEFORE UPDATE ON employees FOR EACH ROW WHEN (:NEW.salary > :OLD.salary)` — explain why or why not, and fix it if needed.

**Answer:** This **fails to compile**. The `WHEN` clause has its own syntax rule: references to the pseudorecords inside it must be written **without** the leading colon (`NEW.salary`, `OLD.salary`), even though the colon is required everywhere else a trigger references `:NEW`/`:OLD` (the body, a `DECLARE` section). Using `:NEW.salary`/`:OLD.salary` inside the `WHEN` clause specifically is invalid syntax.

**Fix:**
```sql
CREATE OR REPLACE TRIGGER trg_example
BEFORE UPDATE ON employees
FOR EACH ROW
WHEN (NEW.salary > OLD.salary)
BEGIN
   NULL;
END;
/
```

---

## Intermediate (Reasoning & Edge Cases)

### Exercise 5
**Task:** A developer writes an `AFTER` row-level trigger that tries `:NEW.last_modified_date := SYSDATE;`, intending to auto-stamp the modification date. It fails. Explain the error and how to fix it — including any behavior differences the fix introduces.

**Answer:** This raises `ORA-04084: cannot change NEW values for this trigger type`. By the time an `AFTER` row-level trigger fires, the row's actual write to disk has **already happened** — there's nothing left for an assignment to `:NEW.column` to affect, so Oracle disallows the attempt outright rather than silently doing nothing.

**Fix:** change the trigger's timing from `AFTER` to `BEFORE`:
```sql
CREATE OR REPLACE TRIGGER trg_set_last_modified
BEFORE UPDATE ON employees
FOR EACH ROW
BEGIN
   :NEW.last_modified_date := SYSDATE;
END;
/
```

**Behavior difference introduced by the fix:** none that affects the *outcome* here — the assignment now happens before the row is written, so the correct value ends up stored either way once the trigger is `BEFORE` instead of `AFTER`. The meaningful difference is purely about *when* in the row's lifecycle this can happen at all: `BEFORE` is the only timing where writing to `:NEW` is even possible, so for any "derive/auto-populate a column" requirement, `BEFORE` isn't just a preference — it's the only option that works.

---

### Exercise 6
**Task:** A developer writes a `BEFORE INSERT` row-level trigger on `employees` that runs `SELECT COUNT(*) INTO v_cnt FROM employees WHERE department_id = :NEW.department_id` to enforce "no more than 50 employees per department." What happens when this fires during a bulk `INSERT` of multiple rows in one statement, and why?

**Answer:** This raises `ORA-04091: table is mutating, trigger/function may not see it` the moment the `INSERT` statement runs — for **every row**, not just some of them, since the very first row-level firing already attempts to query the `employees` table while an `INSERT` against that same table is actively in progress. This is a textbook mutating-table violation: the trigger's own table (`employees`) is being changed by the very statement that's invoking the trigger, so Oracle cannot guarantee the `COUNT(*)` reflects a consistent, valid snapshot — some of the batch's rows might already be counted, others not yet, and that count could differ depending on exactly which row triggered it. Note that this fails **regardless of whether the INSERT is a single row or a large bulk batch** — even a single-row `INSERT` hits this the same way, since the restriction is about the table being mutated, not about the number of rows involved. The genuine fix for "count how many rows this batch would add per department, before allowing it" requires either restructuring the check as a `BEFORE STATEMENT`-level check with row-level counting cooperation, or — cleanly — a compound trigger, both explored properly in later topics.

---

## Realistic Scenario

### Exercise 7
**Business Requirement:** *"The HR system must maintain a complete change history for the `employees` table: whenever any of `salary`, `department_id`, or `job_title` changes for an employee, record a row in `employee_history` capturing the employee ID, which field changed, its old value, its new value, and when/by whom the change was made. Fields that weren't touched by a given UPDATE must not generate history rows — even if the UPDATE statement's SET clause happens to reassign a column to its current value, that should not count as a change worth recording."*

**Assumed schema:** `employee_history(employee_id, field_changed, old_value, new_value, changed_on, changed_by)`, with `old_value`/`new_value` stored as `VARCHAR2`.

### What Is Being Asked
For a single `UPDATE` on one employee row, up to **three separate history rows** may need to be inserted — one per field that genuinely changed — while explicitly excluding any field that was merely reassigned to its existing value.

### Key Clues
- "whenever any of `salary`, `department_id`, or `job_title` changes" → a row-level trigger checking three columns independently, potentially inserting multiple history rows from one row-level firing.
- "**even if** the UPDATE statement's SET clause happens to reassign a column to its current value, that should not count" → this is an explicit, direct statement of the syntactic-vs-semantic distinction from Section 7 — the requirement is deliberately calling out the exact trap `UPDATE OF column_list` alone would fall into.

### Concepts to Consider
`UPDATE OF column_list` as a coarse, cheap first-pass filter (only fire this trigger at all if the statement touches at least one of the three relevant columns), combined with an explicit per-column value-change check inside the body for each of the three fields individually — and a NULL-safe way to perform that comparison, since a naive `!=` would miss a change involving `NULL` on either side.

### Solution
```sql
CREATE OR REPLACE TRIGGER trg_employee_change_history
AFTER UPDATE OF salary, department_id, job_title ON employees
FOR EACH ROW
BEGIN
   IF DECODE(:NEW.salary, :OLD.salary, 0, 1) = 1 THEN
      INSERT INTO employee_history (employee_id, field_changed, old_value, new_value, changed_on, changed_by)
      VALUES (:OLD.employee_id, 'SALARY', TO_CHAR(:OLD.salary), TO_CHAR(:NEW.salary), SYSDATE, USER);
   END IF;

   IF DECODE(:NEW.department_id, :OLD.department_id, 0, 1) = 1 THEN
      INSERT INTO employee_history (employee_id, field_changed, old_value, new_value, changed_on, changed_by)
      VALUES (:OLD.employee_id, 'DEPARTMENT_ID', TO_CHAR(:OLD.department_id), TO_CHAR(:NEW.department_id), SYSDATE, USER);
   END IF;

   IF DECODE(:NEW.job_title, :OLD.job_title, 0, 1) = 1 THEN
      INSERT INTO employee_history (employee_id, field_changed, old_value, new_value, changed_on, changed_by)
      VALUES (:OLD.employee_id, 'JOB_TITLE', :OLD.job_title, :NEW.job_title, SYSDATE, USER);
   END IF;
END;
/
```

### Detailed Explanation
`UPDATE OF salary, department_id, job_title` in the trigger header is a cheap **syntactic** pre-filter: this trigger doesn't even fire at all for an `UPDATE` that touches none of these three columns (e.g., an update that only changes an employee's phone number), avoiding unnecessary work. But inside the body, each field gets its own **semantic** check via `DECODE(:NEW.col, :OLD.col, 0, 1) = 1` — this is a standard, important practical trick: `DECODE` treats two `NULL`s as **equal**, unlike a plain `!=` comparison, where `NULL != NULL` (and any comparison involving a `NULL`) evaluates to `UNKNOWN`, not `TRUE` or `FALSE`. This means `DECODE` correctly and safely catches every real change — including a column going from a real value to `NULL`, or from `NULL` to a real value — cases a naive `:NEW.col != :OLD.col` would silently miss.

Each of the three `IF` blocks is independent, so a single `UPDATE` that changes, say, both `salary` and `job_title` in one statement correctly produces **two** history rows for that one row-level firing — exactly matching "record a row... whenever any of these fields changes," applied per field, not per statement or per row update as a whole.

### Alternative Approach
Instead of three separate `IF`/`DECODE` blocks, the same logic could be written using a small local `PROCEDURE` (declared in the trigger's own `DECLARE` section) taking the field name, old value, and new value as parameters, called three times — reducing repetition. This is a reasonable readability improvement for exactly three fields; if the table had a dozen auditable columns, this refactor would become considerably more valuable, as three near-identical `IF` blocks would become twelve.

### Edge Cases
- An `UPDATE` that reassigns `salary` to its exact current value (e.g., a batch job that blindly sets every column regardless of whether it changed) → `UPDATE OF salary` still fires the trigger (syntactic condition met), but `DECODE(:NEW.salary, :OLD.salary, 0, 1)` correctly evaluates to `0`, and no history row is inserted for salary — exactly the requirement's explicit "should not count as a change" case.
- A `department_id` changing from a real value to `NULL` (e.g., an employee being un-assigned from any department) → correctly detected as a change by `DECODE`, where a naive `!=` comparison would have silently missed it.

### Common Mistakes to Watch For
- Using `WHEN (NEW.salary != OLD.salary OR NEW.department_id != OLD.department_id OR NEW.job_title != OLD.job_title)` as the trigger's sole filter, then writing all three history inserts unconditionally inside the body — this would incorrectly insert history rows for **every** field whenever *any one* of them changed, rather than only for the field(s) that actually changed individually.
- Forgetting the `DECODE` (or equivalent NULL-safe) comparison and using plain `!=` — silently missing changes that involve a `NULL` on either side, a genuinely easy trap to fall into since it produces no error, just quietly incomplete audit data.
- Trying to combine all three field checks into a single history row instead of up to three separate rows — misreading the requirement, which explicitly wants one history row per changed field, not one row per statement summarizing all changes at once.

---

**End of Topic 2.** Next file: `10-trigger-timing-before-after-instead-of.md`.