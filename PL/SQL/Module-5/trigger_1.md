# Topic 1: Introduction to Triggers, Statement Level Trigger

> **Syllabus position:** Topic 1 of 6 (new Triggers syllabus)
> **Essential supporting tool introduced here (not a separate syllabus item):** `RAISE_APPLICATION_ERROR`
> **Feeds into:** Topic 2 (Row Level Triggers), Topic 3 (Timing), Topic 4 (Events & Execution Order), Topic 5 (Compound Triggers)

---

## 0. Essential Supporting Tool: `RAISE_APPLICATION_ERROR`

You cannot write a trigger that actually *rejects* an operation without this, so it's covered here before anything else.

**What it is:** a built-in procedure, `RAISE_APPLICATION_ERROR(error_number, message)`, that immediately raises a custom error and aborts whatever PL/SQL is currently executing — inside a trigger, this means it aborts the **triggering DML statement itself**, rolling back any changes that statement had already made, and returns the error back to whoever issued that statement, exactly as if a database error had occurred.

**Rules:**
- `error_number` must be in the range **-20000 to -20999** — a range Oracle reserves specifically for user-defined application errors. Using a number outside this range raises an error of its own.
- `message` is any custom string you choose, shown to whoever's DML gets rejected.

```sql
RAISE_APPLICATION_ERROR(-20001, 'Deletes on ORDERS are only allowed during business hours (9am-6pm).');
```

This is the standard way a trigger says "no — this operation must not be allowed to happen at all." You'll use it constantly from this point forward.

---

## 1. Concept

A **trigger** is a stored PL/SQL block that Oracle automatically executes ("fires") in response to a specified event on a table, view, schema, or the database itself — most often a DML event (`INSERT`, `UPDATE`, `DELETE`) against a table. Unlike a procedure or function, **you never call a trigger directly** — it's invoked implicitly by the database the moment its triggering event occurs.

A **statement-level trigger** fires **exactly once per triggering statement**, no matter how many rows that statement actually affects — including a statement that matches **zero rows**. This is the key distinction from a row-level trigger (Topic 2), which fires once *per affected row*.

## 2. Purpose / Why It Exists

Triggers let you enforce business rules, maintain audit trails, or run validation **centrally, on the table itself**, rather than relying on every application, script, or ad-hoc user that might ever touch the table to remember to apply that logic themselves. Because a trigger lives on the table, it fires no matter *what* performs the DML — a full application, a one-off SQL*Plus session, a data-load script — closing a gap that logic living only in application code would leave wide open.

Statement-level triggers specifically exist for logic that only cares **"did this kind of operation happen,"** not "what changed in each individual row." Example: logging "an UPDATE occurred on this table, at this time, by this user" — one audit line, regardless of whether the `UPDATE` touched 1 row or 10,000. A row-level trigger doing the same logging would produce 10,000 duplicate audit entries for a single bulk `UPDATE` — statement-level fires exactly once, which is the right granularity for this kind of requirement.

## 3. What Real-World Problem It Solves

- Auditing "an operation of type X happened," independent of row count.
- Enforcing table-wide or statement-wide business rules unrelated to any specific row's data — e.g., "no DML against this table outside business hours," or "block this entire class of operation regardless of who issues it."
- Vetoing an entire class of operation outright (e.g., "nobody may ever `DELETE` from this table," full stop).

## 4. When to Use / When NOT to Use

**Use a statement-level trigger when:**
- The logic needs to run exactly once per statement, independent of how many rows are affected — including zero.
- You need to check something about the *statement itself* (time of day, type of operation) rather than about specific row data.

**Use a row-level trigger instead (Topic 2) when:**
- The logic needs per-row detail — the actual old/new values of the columns being changed. **A statement-level trigger has no access to `:OLD`/`:NEW` at all** — those are exclusively a row-level feature. A statement-level trigger only knows "this statement fired," never what data it touched.

**Recognition clue:** if a requirement never mentions a specific row's old or new value, and instead talks about "this kind of operation happening" or "this type of change being allowed/restricted," that's a statement-level signal.

## 5. Syntax

```sql
CREATE [OR REPLACE] TRIGGER trigger_name
{BEFORE | AFTER} {INSERT | UPDATE | DELETE} [OR {INSERT | UPDATE | DELETE} ...]
ON table_name
[DECLARE
   -- local variable declarations]
BEGIN
   -- trigger body
END;
```

```sql
CREATE OR REPLACE TRIGGER trg_log_salary_update
AFTER UPDATE ON employees
BEGIN
   INSERT INTO audit_log (action, action_time, action_by)
   VALUES ('SALARY TABLE UPDATED', SYSDATE, USER);
END;
/
```

### Syntax Breakdown

- `CREATE [OR REPLACE] TRIGGER trigger_name` — same `OR REPLACE` convention as procedures/functions; lets you redefine an existing trigger without dropping it first.
- `{BEFORE | AFTER}` — timing (full depth in Topic 3). Statement-level triggers can be `BEFORE` or `AFTER` — never `INSTEAD OF`, which is always row-level and only applies to views (Topics 3 & 5).
- `{INSERT | UPDATE | DELETE}` — the triggering event(s); multiple events can be combined on one trigger with `OR` (Topic 4 covers how to tell, inside the body, which one actually fired).
- `ON table_name` — the table the trigger is attached to.
- **No `FOR EACH ROW` clause** — this absence is the single defining syntactic marker of a statement-level trigger. Its presence (Topic 2) is what turns a trigger into a row-level one.
- The trigger body is an ordinary PL/SQL block — it can have its own `DECLARE` section and exception handling, exactly like a procedure body.

## 6. Types / Variations

| Variation | Description |
|---|---|
| **Single-event** | e.g., `AFTER INSERT ON table_name` — fires only for that one DML type. |
| **Combined-event** | e.g., `AFTER INSERT OR UPDATE OR DELETE ON table_name` — one trigger body handles multiple event types; distinguishing which one fired needs `INSERTING`/`UPDATING`/`DELETING` (previewed below, full depth in Topic 4). |
| **`BEFORE` statement-level** | Runs once, before the statement makes any changes — used for statement-wide validation or vetoing. |
| **`AFTER` statement-level** | Runs once, after the statement has completed all its changes — used for statement-wide follow-up (typically logging). |

## 7. Simple Examples

### Example A — Statement-wide time restriction (vetoing an entire operation)
```sql
CREATE OR REPLACE TRIGGER trg_restrict_order_deletes
BEFORE DELETE ON orders
BEGIN
   IF TO_CHAR(SYSDATE, 'HH24') NOT BETWEEN '09' AND '18' THEN
      RAISE_APPLICATION_ERROR(-20001,
         'Deletes on ORDERS are only allowed during business hours (9am-6pm).');
   END IF;
END;
/
```
Notice this check has nothing to do with *which* rows are being deleted — it's purely about whether the statement is allowed to run at all, which is exactly the statement-level use case.

### Example B — Combined-event audit log (a preview of `INSERTING`/`UPDATING`/`DELETING`)
```sql
CREATE OR REPLACE TRIGGER trg_audit_employees_stmt
AFTER INSERT OR UPDATE OR DELETE ON employees
BEGIN
   INSERT INTO audit_log (table_name, operation, action_time, action_by)
   VALUES ('EMPLOYEES',
           CASE
              WHEN INSERTING THEN 'INSERT'
              WHEN UPDATING  THEN 'UPDATE'
              WHEN DELETING  THEN 'DELETE'
           END,
           SYSDATE, USER);
END;
/
```
`INSERTING`, `UPDATING`, and `DELETING` are boolean conditional predicates available inside a trigger body that combines multiple events — they let one trigger body know which specific event actually fired on this occasion. Full depth on this is Topic 4; this example previews it because a combined-event trigger is barely useful without it.

### Example C — Demonstrating "once per statement, not once per row"
```sql
CREATE OR REPLACE TRIGGER trg_count_stmt_fire
AFTER INSERT ON employees
BEGIN
   DBMS_OUTPUT.PUT_LINE('Statement-level trigger fired.');
END;
/
```
If a single `INSERT INTO employees SELECT ... FROM new_hires` statement adds 500 rows at once, this trigger prints `'Statement-level trigger fired.'` **exactly once** — not 500 times.

## 8. Detailed Explanation

- **Fires once per statement, regardless of rows affected — including zero.** `DELETE FROM orders WHERE order_id = 999999` (no such row exists) still fires an `AFTER DELETE` statement-level trigger exactly once, because the *statement* executed, even though it changed nothing. This trips up a lot of learners who assume "zero rows affected" means "nothing happens at all," including no trigger firing.
- **No `:OLD`/`:NEW` access.** Since a statement-level trigger has no concept of "the row being changed," there's nothing to reference for old/new column values — that's exclusively a row-level feature (Topic 2).
- **Order among multiple statement-level triggers of the same type is not guaranteed** unless explicitly controlled. If two separate `AFTER INSERT` statement-level triggers exist on the same table, both will fire, but which one runs first is not something you should assume based on creation order — Oracle provides an explicit mechanism to control this (Topic 4's `FOLLOWS`/`PRECEDES` clause), but without it, the order is not a safe assumption to build logic on.
- **`RAISE_APPLICATION_ERROR` aborts the whole statement, not just the trigger.** Calling it inside a `BEFORE` statement trigger means the DML never executes at all. Calling it inside an `AFTER` statement trigger rolls back whatever changes that statement had already made — the entire statement is treated as failed, exactly as if the database itself had rejected it.
- **A key early orientation point** (fully explored in Topic 2's "mutating table" discussion): an `AFTER` statement-level trigger can safely query or even modify its *own* table, because by the time it fires, every row-level change for that statement has already completed. This is *not* generally safe from inside a row-level trigger — a preview of a genuinely important restriction covered properly next topic.

## 9. Common Mistakes and Misconceptions

1. Expecting a statement-level trigger to fire once per row — it fires once per **statement**.
2. Trying to reference `:NEW` or `:OLD` inside a statement-level trigger — not available; this is a compile-time error, since those identifiers simply don't exist outside a row-level trigger context.
3. Assuming a statement matching zero rows skips the trigger entirely — it doesn't; the trigger still fires once.
4. Assuming multiple triggers of the same type/timing on the same table fire in creation order — not guaranteed without an explicit ordering clause (Topic 4).
5. Using an error number outside -20000 to -20999 with `RAISE_APPLICATION_ERROR` — invalid, raises its own error.
6. Believing a `BEFORE` statement-level trigger runs "after the first row changes" — it doesn't; it runs once, before *any* row processing for that statement begins.

## 10. Edge Cases

- A statement affecting zero rows still fires the statement-level trigger exactly once (repeated here because it's the single most commonly misunderstood point in this topic).
- A statement-level trigger's body performing DML against a **different** table (e.g., inserting into an audit table) is completely normal and extremely common.
- `RAISE_APPLICATION_ERROR` in a `BEFORE` trigger prevents the DML from running at all; in an `AFTER` trigger, it rolls back changes that statement had already applied, since the statement as a whole is now treated as failed.

## 11. How This Relates to Other Topics

- **Topic 2 (Row Level Triggers):** the per-row counterpart — adds `FOR EACH ROW` and `:OLD`/`:NEW` access, which statement-level triggers structurally cannot have.
- **Topic 3 (Timing):** formalizes `BEFORE` vs. `AFTER` vs. `INSTEAD OF`, building directly on the functional BEFORE/AFTER distinction introduced here.
- **Topic 4 (Events & Execution Order):** formalizes combining multiple events on one trigger (previewed here with `INSERTING`/`UPDATING`/`DELETING`) and the full firing sequence when statement-level and row-level triggers coexist on the same table.
- **Topic 5 (Compound Triggers):** provides a single trigger object that can hold statement-level and row-level sections together, instead of several separate `CREATE TRIGGER` statements — directly motivated by a limitation you'll see clearly in Exercise 7 below.

---

## Things You Must Remember

- Statement-level = fires once per statement, no matter how many rows are affected, including zero.
- No `FOR EACH ROW` clause = statement-level. Its presence (Topic 2) makes it row-level.
- No access to `:OLD`/`:NEW` at statement level — structurally impossible, not just disallowed.
- `RAISE_APPLICATION_ERROR(-20000 to -20999, message)` is the standard way to veto/abort a triggering statement from any trigger.
- Firing order among multiple statement-level triggers of the same type/timing on the same table is not guaranteed without explicit ordering (Topic 4).

## How to Recognize This Concept

Think **statement-level trigger** when a requirement says things like:
- "log that this kind of operation happened" (not what specific data changed)
- "restrict/allow this type of operation" based on time, day, or statement pattern — unrelated to specific row values
- "block this entire class of operation" outright
- Nothing in the requirement asks about any row's old or new value.

---

# Exercises — With Answers and Reasoning

## Easy (Syntax & Basics)

### Exercise 1
**Task:** Write a statement-level trigger that logs `'PRODUCTS TABLE MODIFIED'` into an `audit_log` table any time an `INSERT`, `UPDATE`, or `DELETE` happens on a `products` table — one generic message regardless of which of the three occurred.

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_products_modified_log
AFTER INSERT OR UPDATE OR DELETE ON products
BEGIN
   INSERT INTO audit_log (table_name, operation, action_time, action_by)
   VALUES ('PRODUCTS', 'PRODUCTS TABLE MODIFIED', SYSDATE, USER);
END;
/
```

**Reasoning:** Since the requirement explicitly wants **one generic message** regardless of which of the three DML types occurred, there's no need for `INSERTING`/`UPDATING`/`DELETING` branching here at all — the trigger body doesn't care which event fired, only that *one* of the three did. This is a good contrast with Example B, where the message genuinely needed to differ by event type.

---

### Exercise 2
**Task:** Write a statement-level trigger that prevents any `DELETE` on a `configuration_settings` table altogether — nobody should ever be allowed to delete from it, under any circumstance.

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_block_config_deletes
BEFORE DELETE ON configuration_settings
BEGIN
   RAISE_APPLICATION_ERROR(-20002, 'Deleting rows from CONFIGURATION_SETTINGS is not permitted.');
END;
/
```

**Reasoning:** This is the simplest possible use of `RAISE_APPLICATION_ERROR` — there's no `IF` condition at all, because the rule is unconditional ("nobody, ever"). `BEFORE` is the right timing here: the goal is to stop the deletion from happening in the first place, not to log it after the fact and then undo it.

---

### Exercise 3
**Task:** Predict the outcome. An application runs `UPDATE employees SET salary = salary * 1.05 WHERE department_id = 999` (a department that doesn't exist — zero rows match) against a table with an `AFTER UPDATE` statement-level trigger that inserts one audit row. How many audit rows get inserted, and why?

**Answer:** **Exactly one** audit row gets inserted, despite the `UPDATE` matching **zero** actual employee rows. A statement-level trigger fires based on the **statement itself executing**, not on how many rows that statement happens to match. Oracle doesn't check "did this affect at least one row?" before deciding whether to fire an `AFTER UPDATE` statement-level trigger — the `UPDATE` statement ran (successfully, just against no matching rows), and that alone is sufficient to fire the trigger once. This is precisely the "fires even for a zero-row statement" edge case flagged repeatedly above, and it's worth internalizing early since it surprises almost everyone the first time they encounter it.

---

### Exercise 4
**Task:** Write a statement-level trigger combining `INSERT` and `UPDATE` (not `DELETE`) on an `invoices` table, logging which of the two operations occurred using `INSERTING`/`UPDATING`.

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_invoices_stmt_log
AFTER INSERT OR UPDATE ON invoices
BEGIN
   INSERT INTO audit_log (table_name, operation, action_time, action_by)
   VALUES ('INVOICES',
           CASE WHEN INSERTING THEN 'INSERT' WHEN UPDATING THEN 'UPDATE' END,
           SYSDATE, USER);
END;
/
```

**Reasoning:** Only `INSERTING` and `UPDATING` are checked here — `DELETING` isn't part of this trigger's event list at all (`DELETE` was deliberately excluded from the syllabus of events this trigger fires for), so referencing it wouldn't even make sense in this trigger's context. The `CASE` expression correctly distinguishes the two events that actually can fire this trigger.

---

## Intermediate (Reasoning & Edge Cases)

### Exercise 5
**Task:** A developer writes a statement-level trigger and, inside its body, tries to reference `:NEW.salary` to log the new salary value being set. Explain what happens and why, and describe what would need to change for that value to actually be accessible.

**Answer:** This **fails to compile**. `:NEW` (and `:OLD`) are identifiers that only exist inside a **row-level** trigger's body — they represent the new/old values of the specific row currently being processed. A statement-level trigger has no concept of "the row currently being processed" at all; it fires once for the whole statement, with no awareness of individual row data. Referencing `:NEW.salary` here is exactly as invalid as referencing an out-of-scope variable — the compiler rejects it because the identifier is never defined in this context.

**What needs to change:** the trigger would need to add a `FOR EACH ROW` clause, turning it into a row-level trigger (Topic 2) — that's the only way to gain access to `:NEW`/`:OLD` and see individual column values as each row is processed.

---

### Exercise 6
**Task:** Two statement-level `AFTER INSERT` triggers exist on the same `orders` table — one inserts an audit row, another sends a (simulated) notification. A developer assumes the audit trigger always runs first because it was created first. Explain why this is risky, and name (without full syntax) the feature that would let them guarantee this order if it actually matters.

**Answer:** This assumption is risky because **Oracle does not guarantee execution order among multiple triggers of the same type and timing on the same table based on creation order** — or any other implicit rule a developer might assume. Relying on "whichever I created first runs first" is not a documented, dependable behavior; a database upgrade, a trigger being dropped and recreated (even with identical logic), or simply differences across environments could change the observed order without any warning. If the actual business logic genuinely depends on one trigger's effects being visible before another's — for instance, if the notification trigger needs to reference something the audit trigger just wrote — that dependency needs to be made **explicit**, not left to chance.

**The feature to guarantee this** is the `FOLLOWS` (or `PRECEDES`) clause on `CREATE TRIGGER`, which explicitly states that one trigger must fire after (or before) another named trigger. Full syntax and worked examples of this are covered properly in Topic 4 (Execution Order) — flagged here only so you know such a tool exists the moment you notice you're relying on an unstated ordering assumption.

---

## Realistic Scenario

### Exercise 7
**Business Requirement:** *"Compliance requires that any bulk price update run against the `products` table be logged with exactly one entry per update statement — including the number of rows it affected and whether it happened inside or outside the company's designated 8am–8pm maintenance window. Updates attempted outside that window must be blocked entirely, not just logged."*

**What Is Being Asked:**
Two things, bundled into one requirement: (1) block any `UPDATE` on `products` outside 8am–8pm entirely, and (2) for updates that do go through, log exactly one audit entry per statement, including how many rows it touched.

**Key Clues:**
- "blocked entirely" → a `BEFORE` statement-level trigger with `RAISE_APPLICATION_ERROR`, exactly like Exercise 2 — this part is fully solvable with what this topic has covered.
- "exactly one entry per update statement... including the number of rows it affected" → this second half is where it gets genuinely interesting, and worth slowing down on.

**Thought Process — Where a Plain Statement-Level Trigger Hits a Real Limit:**
The blocking requirement is straightforward — a `BEFORE` statement-level trigger checking the time and calling `RAISE_APPLICATION_ERROR` if it's outside the window, exactly as in Exercise 2. But now think carefully about the second half: **how would a statement-level trigger know how many rows its own triggering statement affected?** This is a genuinely important limitation to notice rather than paper over. A statement-level trigger isn't code that *issues* the `UPDATE` and can then check something like `SQL%ROWCOUNT` afterward — it *is* logic invoked internally as a side effect of that `UPDATE` executing. There's no built-in "how many rows is this statement about to touch / did it touch" value directly available inside a plain statement-level trigger body.

**Getting the row count actually requires row-level cooperation** — something has to increment a counter once for every row the statement touches, and that's exactly what a *row-level* trigger (`FOR EACH ROW`, Topic 2) is for, not a statement-level one alone. Since row-level triggers haven't been formally taught yet at this point in the syllabus, what follows is a **preview** of the shape of a real solution, using a package-level variable to share a running count between separate trigger objects that all fire as part of the same statement's execution.

**Partial Solution — the part this topic can fully solve (blocking):**
```sql
CREATE OR REPLACE TRIGGER trg_restrict_price_update_window
BEFORE UPDATE ON products
BEGIN
   IF TO_CHAR(SYSDATE, 'HH24') NOT BETWEEN '08' AND '20' THEN
      RAISE_APPLICATION_ERROR(-20010,
         'Price updates on PRODUCTS are only allowed between 8am and 8pm.');
   END IF;
END;
/
```

**Preview Sketch — the row-count-and-log part (uses a row-level trigger not yet formally taught; fully explained once Topic 2 and especially Topic 5 are covered):**
```sql
-- A package purely to hold a running count, shared across the separate trigger
-- objects that all fire during the same UPDATE statement's execution.
CREATE OR REPLACE PACKAGE pkg_price_update_tracking IS
   g_row_count PLS_INTEGER := 0;
END pkg_price_update_tracking;
/

-- Reset the counter once, before this statement's row processing begins
CREATE OR REPLACE TRIGGER trg_reset_price_update_count
BEFORE UPDATE ON products
BEGIN
   pkg_price_update_tracking.g_row_count := 0;
END;
/

-- Increment the counter once per row actually touched (a row-level trigger — Topic 2 preview)
CREATE OR REPLACE TRIGGER trg_count_price_update_rows
BEFORE UPDATE ON products
FOR EACH ROW
BEGIN
   pkg_price_update_tracking.g_row_count := pkg_price_update_tracking.g_row_count + 1;
END;
/

-- Log exactly once, after all rows for this statement have been processed
CREATE OR REPLACE TRIGGER trg_log_price_update_summary
AFTER UPDATE ON products
BEGIN
   INSERT INTO audit_log (table_name, operation, rows_affected, in_window, action_time, action_by)
   VALUES ('PRODUCTS', 'UPDATE', pkg_price_update_tracking.g_row_count,
           CASE WHEN TO_CHAR(SYSDATE, 'HH24') BETWEEN '08' AND '20' THEN 'Y' ELSE 'N' END,
           SYSDATE, USER);
END;
/
```

**Explanation:** Four separate database objects (a package plus three triggers) are cooperating here to do what conceptually feels like one piece of logic: reset a counter before the statement starts, increment it once per row as rows are processed, then read the final count once after the statement finishes, in a single log entry. This works correctly, but it's genuinely clunky — four objects to create, deploy, and maintain in sync for what's really one coherent rule, and if any one of them is accidentally dropped or disabled independently of the others (Topic 5 covers enabling/disabling), the whole scheme silently breaks.

**This exact pain point is the direct motivation for compound triggers**, covered fully in Topic 5 — a compound trigger lets you write a `BEFORE STATEMENT` section, a row-level section, and an `AFTER STATEMENT` section **inside one single trigger object**, sharing state between them naturally without needing a separate package just to pass a counter around. Once you reach Topic 5, come back to this exercise and notice how much cleaner the same logic becomes.

**Common Mistakes to Watch For (in the preview sketch above):**
- Forgetting the `BEFORE STATEMENT`-level reset (`trg_reset_price_update_count`) — without resetting `g_row_count` to zero at the start of each statement, counts would keep accumulating across separate `UPDATE` statements issued later in the same session, producing an incorrect, ever-growing total instead of "rows affected by *this* statement."
- Assuming the row-count trigger alone (without the separate reset trigger) is sufficient — it isn't; incrementing a counter that's never reset just keeps growing indefinitely.
- Not yet knowing about (or not yet needing to know about) the "mutating table" restriction that governs what a row-level trigger like `trg_count_price_update_rows` is and isn't allowed to query from its own table — this exact restriction is the centerpiece of Topic 2, immediately next.

---

**End of Topic 1.** Next file: `09-row-level-triggers.md`.