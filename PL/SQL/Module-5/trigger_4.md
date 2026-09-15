# Topic 4: Trigger Events (INSERT, UPDATE, DELETE) and Execution Order of Triggers

> **Syllabus position:** Topic 4 of 6 (Triggers syllabus)
> **Builds on:** Topics 1–3 (statement/row-level triggers, `:NEW`/`:OLD`, timing)
> **Two pillars in this topic:** (A) how trigger *events* work and when to combine vs. separate them, (B) the full, formal *execution order* — including the `FOLLOWS`/`PRECEDES` tool you've seen named (but not yet defined) in Topics 1 and 3.

---

# Part A: Trigger Events

## 1. Concept

The **event** portion of a trigger's definition (`INSERT`, `UPDATE`, `DELETE`) specifies which DML operation(s) cause it to fire. A single trigger can be tied to one event, or multiple combined with `OR` — you've seen this throughout Topics 1–3; this topic formalizes it as its own subject, including the genuine design trade-off between combining events into one trigger versus keeping them as separate triggers.

## 2. Purpose — Why Combine Multiple Events Into One Trigger?

- **Shared logic across events.** If the *same* body genuinely applies regardless of which event fired (e.g., "log that this table changed"), combining avoids duplicating that logic across three separate trigger objects.
- **A single point of maintenance** for logic that's conceptually *one rule* spanning multiple event types.
- **Fewer trigger objects on a table**, which can make Part B's execution-order reasoning simpler, since there's less to keep track of.

## 3. `INSERTING` / `UPDATING` / `DELETING` — Full Treatment

These are boolean conditional predicates, available inside **any** trigger body (row-level or statement-level), that report `TRUE` if the *current firing* was caused by that specific event. They're used like plain boolean expressions — no parentheses, no arguments:

```sql
IF INSERTING THEN
   ...
ELSIF UPDATING THEN
   ...
ELSIF DELETING THEN
   ...
END IF;
```

**Important nuances:**
- They're not *restricted* to combined-event triggers — technically, a plain `BEFORE INSERT` trigger could check `IF INSERTING THEN`, and it would always be `TRUE` (just pointlessly, since there's nothing else it could be). They're most *usefully* applied in combined-event triggers, where they're genuinely needed to distinguish what's happening.
- For an ordinary single-row `INSERT`, `UPDATE`, or `DELETE`, **exactly one** of these is ever `TRUE` for a given row's firing — never more than one, never zero.
- **One real exception worth knowing about:** a `MERGE` statement can cause **both** `INSERT`-triggering and `UPDATE`-triggering row events within the **same overall statement** — some rows matched (triggering `UPDATE`-style firings), others unmatched (triggering `INSERT`-style firings). For any *individual* row's firing, still exactly one of `INSERTING`/`UPDATING` is `TRUE` — never both simultaneously for the same row — but a single `MERGE` statement can produce a mix of both kinds of firings across its different rows. `MERGE` itself isn't part of this syllabus in depth, but this is a genuinely important practical fact to be aware of if you ever see triggers behave unexpectedly around `MERGE` statements.

## 4. The Real Trade-Off: Combine vs. Keep Separate

This is worth thinking through deliberately, not defaulting to one or the other.

**Combining tends to work well when:**
- The logic is genuinely the same shape across events (shared audit logging, as above).

**Keeping events as separate triggers tends to work better when:**
- The events need **different `WHEN` clauses or different `UPDATE OF column_list` restrictions.** Both of these apply to the *whole* trigger, not to individual branches inside it — so you can't declaratively say "only fire the `UPDATE` branch when column X changes" while leaving the `INSERT` branch unconditional, within one combined trigger. You'd have to move that filtering into an `IF` inside the body instead, losing the efficiency benefit of `WHEN`/`UPDATE OF` pre-filtering for the events that didn't need it.
- The event-specific logic is substantially unrelated in *purpose*, not just mechanically different — cramming unrelated concerns into one trigger via `INSERTING`/`UPDATING`/`DELETING` branches just to "save an object" tends to produce harder-to-read, harder-to-maintain code, recreating the kind of branching complexity that clean, single-purpose triggers were meant to avoid.

### Worked Comparison

**Good use of combining** (truly shared logic):
```sql
CREATE OR REPLACE TRIGGER trg_audit_products
AFTER INSERT OR UPDATE OR DELETE ON products
FOR EACH ROW
BEGIN
   IF INSERTING THEN
      INSERT INTO products_audit (product_id, action, new_price, action_time)
      VALUES (:NEW.product_id, 'INSERT', :NEW.price, SYSDATE);
   ELSIF UPDATING THEN
      INSERT INTO products_audit (product_id, action, old_price, new_price, action_time)
      VALUES (:NEW.product_id, 'UPDATE', :OLD.price, :NEW.price, SYSDATE);
   ELSIF DELETING THEN
      INSERT INTO products_audit (product_id, action, old_price, action_time)
      VALUES (:OLD.product_id, 'DELETE', :OLD.price, SYSDATE);
   END IF;
END;
/
```
Notice `DELETING` must use `:OLD` (since `:NEW` is `NULL` for `DELETE`), `INSERTING` must use `:NEW` (since `:OLD` is `NULL` for `INSERT`), and `UPDATING` uses both — a direct application of Topic 2's availability table.

**Better kept separate** (genuinely different needs per event):
```sql
-- INSERT-specific: default a column
CREATE OR REPLACE TRIGGER trg_set_default_status
BEFORE INSERT ON tickets
FOR EACH ROW
BEGIN
   IF :NEW.status IS NULL THEN
      :NEW.status := 'OPEN';
   END IF;
END;
/

-- UPDATE-specific: only cares about one column genuinely changing, needs its own WHEN
CREATE OR REPLACE TRIGGER trg_priority_change_alert
AFTER UPDATE OF priority ON tickets
FOR EACH ROW
WHEN (NEW.priority = 'CRITICAL' AND NVL(OLD.priority, 'X') != 'CRITICAL')
BEGIN
   INSERT INTO alerts (ticket_id, alert_type, alert_time)
   VALUES (:NEW.ticket_id, 'CRITICAL_PRIORITY', SYSDATE);
END;
/
```
If these were merged into one `BEFORE INSERT OR UPDATE OF priority` trigger, the `WHEN` clause would need to apply to *both* firings — but `NEW.priority = 'CRITICAL' AND OLD.priority != 'CRITICAL'` doesn't even make clean sense for an `INSERT` (there's no meaningful "old" priority yet), and would need extra `IF` branching inside the body to sort out, losing the clarity and efficiency of a clean `WHEN` clause. Keeping them separate avoids that mess entirely.

---

# Part B: Execution Order of Triggers

## 5. The Canonical Firing Sequence

For a single DML statement against one table (assuming one trigger per timing/level, for now):

1. **`BEFORE STATEMENT`** triggers fire once, before any row processing begins.
2. **For each affected row** (in an order not guaranteed):
   a. **`BEFORE ROW`** triggers fire for that row.
   b. The row's actual DML change is applied.
   c. **Constraint checks** are validated for that row (`NOT NULL`, `CHECK`, foreign key, unique).
   d. **`AFTER ROW`** triggers fire for that row.
3. **`AFTER STATEMENT`** triggers fire once, after every row has been fully processed.

**Critical clarification — this is per-row, not batched by phase.** Step 2 (a → b → c → d) runs to completion for **one row at a time** — it is *not* "run `BEFORE ROW` for every row first, then apply every row's change, then run `AFTER ROW` for every row." Row 1 completes its entire `a → b → c → d` sequence before row 2's sequence begins. This is a genuinely common point of confusion — some learners assume all `BEFORE ROW` firings happen as one batch before any `AFTER ROW` firing starts, which is incorrect.

### Example — Tracing the Full Sequence
```sql
CREATE OR REPLACE TRIGGER trg_a_before_stmt
BEFORE UPDATE ON accounts
BEGIN
   DBMS_OUTPUT.PUT_LINE('1. BEFORE STATEMENT');
END;
/
CREATE OR REPLACE TRIGGER trg_b_before_row
BEFORE UPDATE ON accounts
FOR EACH ROW
BEGIN
   DBMS_OUTPUT.PUT_LINE('2. BEFORE ROW for account ' || :NEW.account_id);
END;
/
CREATE OR REPLACE TRIGGER trg_c_after_row
AFTER UPDATE ON accounts
FOR EACH ROW
BEGIN
   DBMS_OUTPUT.PUT_LINE('3. AFTER ROW for account ' || :OLD.account_id);
END;
/
CREATE OR REPLACE TRIGGER trg_d_after_stmt
AFTER UPDATE ON accounts
BEGIN
   DBMS_OUTPUT.PUT_LINE('4. AFTER STATEMENT');
END;
/
```
Running `UPDATE accounts SET balance = balance * 1.01 WHERE branch_id = 5;` against 3 matching rows (101, 102, 103) prints:
```
1. BEFORE STATEMENT
2. BEFORE ROW for account 101
3. AFTER ROW for account 101
2. BEFORE ROW for account 102
3. AFTER ROW for account 102
2. BEFORE ROW for account 103
3. AFTER ROW for account 103
4. AFTER STATEMENT
```
(The specific *order* among accounts 101/102/103 isn't guaranteed — but the **interleaving pattern** — `BEFORE ROW` immediately followed by `AFTER ROW` for the *same* row before moving to the next row — is guaranteed.)

## 6. Order Among Multiple Triggers of the Same Type

When more than one trigger shares the exact same timing + level + overlapping event on the same table (e.g., two separate `BEFORE ROW` triggers both firing for an `INSERT`), Oracle fires **all** of them — but by default, the order **between** them is **unspecified**, exactly the point flagged repeatedly in Topics 1 and 3. Topic 3, Exercise 5 showed a concrete case where this genuinely changes the outcome.

## 7. `FOLLOWS` / `PRECEDES` — Full Syntax

```sql
CREATE [OR REPLACE] TRIGGER trigger_name
{BEFORE | AFTER} ... ON table_name
[FOR EACH ROW]
FOLLOWS other_trigger_name [, another_trigger_name ...]
BEGIN
   ...
END;
```
or `PRECEDES` in place of `FOLLOWS`.

- **`FOLLOWS other_trigger`** — this trigger fires *after* the named trigger (of the same timing + level) has fired for that row/statement.
- **`PRECEDES other_trigger`** — this trigger fires *before* the named trigger.
- **Only meaningful between triggers of the same timing + level combination.** It doesn't make sense to order a `BEFORE ROW` trigger relative to an `AFTER STATEMENT` trigger using `FOLLOWS`/`PRECEDES` — their relative order is already fixed by the canonical sequence in Section 5, regardless of creation order or anything else.
- You can name multiple triggers in one clause.
- **The referenced trigger must already exist** at the time this trigger is created — if it doesn't, creation fails. This is a real, practical deployment-order consideration for migration scripts: if Trigger B says `FOLLOWS Trigger A`, Trigger A must be deployed first.

### Example — Removing the Ambiguity from Topic 3's Exercise 5
```sql
CREATE OR REPLACE TRIGGER trg_validate_first
BEFORE INSERT ON orders
FOR EACH ROW
BEGIN
   IF :NEW.quantity <= 0 THEN
      RAISE_APPLICATION_ERROR(-20060, 'Quantity must be positive.');
   END IF;
END;
/

CREATE OR REPLACE TRIGGER trg_calc_total_second
BEFORE INSERT ON orders
FOR EACH ROW
FOLLOWS trg_validate_first
BEGIN
   :NEW.total_amount := :NEW.quantity * :NEW.unit_price;
END;
/
```
`trg_calc_total_second` explicitly `FOLLOWS trg_validate_first`, guaranteeing validation always runs first — rejecting a bad quantity before any calculation is even attempted, removing exactly the kind of ordering ambiguity Topic 3's Exercise 5 demonstrated going wrong.

## 8. Cascading Triggers Across Tables

If a trigger's body performs DML against **another** table (extremely common — audit inserts, updating a summary table), and *that* table has its own triggers, those fire too — nested within the original trigger's execution, forming a genuine trigger cascade. This is completely legitimate and common, but carries real risks worth naming explicitly:

- **Infinite loop risk.** If Table A's trigger updates Table B, and Table B's trigger updates Table A again, and Table A's trigger fires again… this can spiral into recursive triggering until Oracle hits a hard limit and raises an error. This is a genuinely serious risk with cross-table trigger design — be very cautious whenever two tables' triggers each modify the other, even indirectly through a longer chain.
- **Debugging difficulty.** A cascade spanning several tables and triggers can be genuinely hard to trace when something goes wrong, since the "call stack" isn't as visible as ordinary nested procedure calls would be.
- **Cumulative overhead.** Each additional hop in a cascade adds real cost to what might look, from the outside, like a single simple `INSERT`.

## 9. Detailed Explanation — Additional Notes

- `FOLLOWS`/`PRECEDES` chains can be transitive: if C `FOLLOWS` B, and B `FOLLOWS` A, Oracle resolves the full chain correctly (A, then B, then C).
- A **circular** `FOLLOWS` relationship (A `FOLLOWS` B, while B `FOLLOWS` A) is not permitted — Oracle detects this circular dependency and raises an error at creation time, refusing to create the trigger that would complete the cycle.

## 10. Common Mistakes and Misconceptions

1. Assuming `BEFORE ROW` fires for *every* row before `AFTER ROW` starts for *any* of them — incorrect; it's fully interleaved, per row.
2. Assuming firing order among same-type, same-timing triggers is deterministic (e.g., by creation order) without an explicit `FOLLOWS`/`PRECEDES`.
3. Creating a trigger with `FOLLOWS` a trigger that doesn't exist yet — fails; deployment order matters.
4. Trying to use `FOLLOWS`/`PRECEDES` to order triggers across *different* timing/level categories — not meaningful; that order is already fixed.
5. Cramming unrelated per-event logic into one combined trigger purely to reduce object count, losing the ability to use `WHEN`/`UPDATE OF` cleanly per event.
6. Not considering cascading-trigger risk when a trigger's DML touches another table that has its own triggers, especially any that could loop back.
7. Assuming `INSERTING`/`UPDATING`/`DELETING` could be `TRUE` together for one row's firing under ordinary `INSERT`/`UPDATE`/`DELETE` — never true (the `MERGE` nuance is about different rows within one statement, not one row being two things at once).

## 11. Edge Cases

- A **zero-row statement** still fires `BEFORE STATEMENT` and `AFTER STATEMENT` once each; **no** `BEFORE ROW`/`AFTER ROW` firings occur at all, since there are no rows to iterate over (ties back directly to Topic 1's zero-row edge case, now precisely distinguished from the row-level triggers, which genuinely don't fire in this case).
- A statement affecting exactly one row still runs the full canonical sequence — just with a single row iteration.
- A `FOLLOWS` chain of three or more triggers resolves correctly and transitively.
- A circular `FOLLOWS` relationship is rejected at creation time.

## 12. How This Relates to Other Topics

- Formalizes the `BEFORE`/`AFTER` distinction from Topic 3 into a complete, precise sequence.
- Gives proper syntax to the `FOLLOWS`/`PRECEDES` tool named (but not defined) in Topic 1 and directly needed by Topic 3's Exercise 5.
- **Topic 5 (Compound Triggers)** offers an alternative to juggling multiple separate trigger objects and `FOLLOWS`/`PRECEDES` chains — a compound trigger keeps the statement-level and row-level sections of one logical rule together in a single object, sidestepping some of this topic's ordering concerns entirely for logic that's really "one rule."

---

## Things You Must Remember

- Combining events into one trigger suits genuinely shared logic; separate triggers suit event-specific `WHEN`/`UPDATE OF` needs or unrelated concerns.
- `INSERTING`/`UPDATING`/`DELETING` — exactly one is `TRUE` per row's firing under ordinary DML (a `MERGE` statement can mix both across *different* rows in one statement, but never both for the *same* row).
- Canonical order: `BEFORE STATEMENT` → (`BEFORE ROW` → apply → constraints → `AFTER ROW`) *per row, one row at a time* → `AFTER STATEMENT`.
- Order among same-type, same-timing triggers is unspecified unless controlled with `FOLLOWS`/`PRECEDES`.
- `FOLLOWS`/`PRECEDES` only matters *within* the same timing + level category; the referenced trigger must already exist.
- Cross-table trigger cascades are legitimate but carry real infinite-loop and debugging risk — be especially careful with any two tables whose triggers could modify each other.

## How to Recognize This Concept

- "log/react the same way regardless of which operation happened" → combine events into one trigger.
- "this rule only cares about one specific column, or only one specific event" → keep it as its own, separate trigger.
- "trigger X must always run before/after trigger Y" (same timing/level) → `FOLLOWS`/`PRECEDES`.
- "trigger X must always run before/after trigger Y" where X and Y are naturally `BEFORE` vs. `AFTER` → often **already guaranteed for free** by the canonical sequence, no extra ordering clause needed (see Exercise 7).

---

# Exercises — With Answers and Reasoning

## Easy (Syntax & Basics)

### Exercise 1
**Task:** Write a combined-event trigger for a `feedback` table that logs into `feedback_audit` whenever a row is inserted **or** deleted (not updated).

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_feedback_audit
AFTER INSERT OR DELETE ON feedback
FOR EACH ROW
BEGIN
   INSERT INTO feedback_audit (feedback_id, operation, action_time)
   VALUES (
      NVL(:NEW.feedback_id, :OLD.feedback_id),
      CASE WHEN INSERTING THEN 'INSERT' WHEN DELETING THEN 'DELETE' END,
      SYSDATE
   );
END;
/
```

**Reasoning:** `UPDATE` is deliberately excluded from the event list, so `UPDATING` would never be `TRUE` here and isn't checked. `NVL(:NEW.feedback_id, :OLD.feedback_id)` correctly picks up the ID regardless of which of the two possible events fired, since exactly one side is always populated.

---

### Exercise 2
**Task:** Predict the exact printed sequence for a **single-row** `UPDATE` against a table with `BEFORE STATEMENT`, `BEFORE ROW`, `AFTER ROW`, and `AFTER STATEMENT` triggers, each printing a distinct label.

**Answer:** For one row, the canonical sequence collapses to a single pass through the per-row loop:
```
BEFORE STATEMENT
BEFORE ROW
AFTER ROW
AFTER STATEMENT
```
**Reasoning:** With only one row affected, there's no interleaving ambiguity to worry about — the sequence is just `BEFORE STATEMENT` once, then one full `BEFORE ROW → apply → constraints → AFTER ROW` pass for that single row, then `AFTER STATEMENT` once. This is the simplest possible case of the canonical sequence from Section 5.

---

### Exercise 3
**Task:** Two `BEFORE ROW` triggers exist on `accounts` for `UPDATE`: `trg_x` and `trg_y`. Write `trg_y` so it's guaranteed to fire **before** `trg_x`.

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_y
BEFORE UPDATE ON accounts
FOR EACH ROW
PRECEDES trg_x
BEGIN
   NULL;   -- trg_y's actual logic here
END;
/
```

**Reasoning:** `PRECEDES trg_x` directly states "this trigger must fire before `trg_x`." An equally valid alternative would be adding `FOLLOWS trg_y` to `trg_x`'s own definition instead — either direction works, as long as exactly one of the two triggers declares the relationship.

---

### Exercise 4
**Task — True or False, with reasoning:** *"If an UPDATE statement affects zero rows, the BEFORE ROW and AFTER ROW triggers on that table still each fire once, just with :NEW and :OLD both NULL."*

**Answer: False.** `BEFORE ROW` and `AFTER ROW` triggers don't fire **at all** when zero rows are affected — there's no row for the per-row loop (Section 5, step 2) to iterate over, so that entire phase is simply skipped. Only the `BEFORE STATEMENT` and `AFTER STATEMENT` triggers fire in this case (each exactly once), consistent with Topic 1's original zero-row edge case — this exercise specifically sharpens that point by contrasting it against row-level triggers, which genuinely do not fire at all for a zero-row statement, rather than firing with `NULL` values as the (incorrect) statement suggests.

---

## Intermediate (Reasoning & Edge Cases)

### Exercise 5
**Task:** An `AFTER INSERT` row-level trigger on `orders` inserts a summary row into `order_totals_by_customer`. That table has its own `AFTER UPDATE` trigger that recalculates a `customer_tier` value on `customers`. Walk through what happens when a single new order is inserted, and identify one realistic risk of this design.

**Answer — the cascade, step by step:**
1. `INSERT INTO orders (...)` fires `orders`'s `AFTER INSERT` row-level trigger.
2. That trigger's body performs an `UPDATE` (or `INSERT`) against `order_totals_by_customer`.
3. This, in turn, fires `order_totals_by_customer`'s own `AFTER UPDATE` trigger.
4. That trigger performs an `UPDATE` against `customers.customer_tier`.

So a single `INSERT` into `orders` ends up triggering a chain across **three tables**, entirely automatically, nested within the original statement's execution.

**Realistic risk:** if `customers` itself had any trigger that, directly or indirectly, ended up modifying `orders` or `order_totals_by_customer` again (perhaps added later by a different developer, unaware of this existing chain), the cascade would loop back on itself — `orders` → `order_totals_by_customer` → `customers` → back to `orders` → … — spiraling into recursive triggering until Oracle hits a hard limit and fails. Even without full circularity, a three-table chain triggered by a single `INSERT` is harder to trace when debugging (a developer looking only at the `orders` trigger wouldn't immediately realize `customers.customer_tier` also changes as a side effect), and adds real, easy-to-underestimate cumulative overhead to what looks like one simple row insert.

---

### Exercise 6
**Task:** A developer's deployment script creates `trg_B` (with `FOLLOWS trg_A`) *before* creating `trg_A`. What happens, and what does this imply about how deployment scripts using `FOLLOWS`/`PRECEDES` must be ordered?

**Answer:** Creating `trg_B` **fails** — Oracle cannot resolve a `FOLLOWS` reference to a trigger that doesn't exist yet at the moment `trg_B` itself is being created. This implies that any deployment or migration script using `FOLLOWS`/`PRECEDES` must create triggers in **dependency order**: the referenced trigger first, then the one that follows (or precedes) it. This is conceptually the same kind of ordering constraint as a foreign key that can't reference a table that doesn't exist yet — a real, practical concern for anyone writing DDL scripts that use this feature, not just an abstract syntax rule.

---

## Realistic Scenario

### Exercise 7
**Business Requirement:** *"The order fulfillment system has a `shipments` table. Whenever a shipment row is inserted, updated, or deleted, an entry must be recorded in `shipments_audit` describing which operation occurred and the shipment's key details. Separately, whenever a shipment's `status` column specifically changes to `'CANCELLED'`, a `cancellation_fee` must be calculated and written onto that same row before it's saved — and this calculation must always happen before the general audit entry is written, since the audit entry needs to reflect the final, already-calculated cancellation fee, not a stale value."*

**What Is Being Asked:**
Two related pieces of trigger logic on the same table — a general, combined-event audit log, and a specific calculation tied to one particular status transition — with an explicit ordering constraint tying the two together.

**Key Clues:**
- "recorded... describing which operation occurred" for `INSERT`/`UPDATE`/`DELETE` → a combined-event, `AFTER` audit trigger (Part A's "good use of combining" case, and Topic 3's "log what genuinely happened" reasoning for choosing `AFTER`).
- "calculated and written onto that same row before it's saved" → must write `:NEW.cancellation_fee`, which is only possible `BEFORE` (Topic 2).
- **"this calculation must always happen before the general audit entry is written"** → this phrasing is the heart of the exercise, examined below.

**Thought Process — The Ordering Requirement Is a Trap, in a Good Way**
A first instinct on reading "must always happen before" might be to reach immediately for `FOLLOWS`/`PRECEDES`, since that's this topic's tool for controlling order between triggers. But pause and check: **are these two triggers even in the same timing + level category?** The cancellation-fee logic needs to be `BEFORE` (it writes `:NEW`). The audit logic needs to be `AFTER` (it logs the genuinely final state, following the same reasoning Topic 3 established for preferring `AFTER` for this kind of logging). `BEFORE ROW` and `AFTER ROW` triggers on the same table are **already guaranteed, by the canonical sequence itself**, to run in that relative order for the same row — `BEFORE ROW` always completes before `AFTER ROW` even begins, with no `FOLLOWS`/`PRECEDES` clause required, because they're different timing categories, not two triggers competing within the same one. The requirement's ordering constraint is satisfied **for free**, simply by making the correct `BEFORE` vs. `AFTER` choice for each piece of logic — exactly the kind of "don't add machinery a design decision already handles" recognition this course has emphasized before.

**Solution:**
```sql
-- Cancellation fee: BEFORE, because it must write :NEW before the row is saved
CREATE OR REPLACE TRIGGER trg_shipments_cancellation_fee
BEFORE UPDATE OF status ON shipments
FOR EACH ROW
WHEN (NEW.status = 'CANCELLED' AND NVL(OLD.status, 'X') != 'CANCELLED')
BEGIN
   :NEW.cancellation_fee := :NEW.shipping_cost * 0.15;
END;
/

-- General audit: AFTER, because it logs the genuinely final state — including the now-final cancellation_fee
CREATE OR REPLACE TRIGGER trg_shipments_audit
AFTER INSERT OR UPDATE OR DELETE ON shipments
FOR EACH ROW
BEGIN
   INSERT INTO shipments_audit (shipment_id, operation, status, cancellation_fee, action_time)
   VALUES (
      NVL(:NEW.shipment_id, :OLD.shipment_id),
      CASE WHEN INSERTING THEN 'INSERT' WHEN UPDATING THEN 'UPDATE' WHEN DELETING THEN 'DELETE' END,
      NVL(:NEW.status, :OLD.status),
      NVL(:NEW.cancellation_fee, :OLD.cancellation_fee),
      SYSDATE
   );
END;
/
```

**Explanation:** No `FOLLOWS`/`PRECEDES` clause appears anywhere in this solution, and none is needed. For any given `UPDATE` that changes `status` to `'CANCELLED'`, the canonical sequence guarantees `trg_shipments_cancellation_fee` (`BEFORE ROW`) completes — including its `:NEW.cancellation_fee` assignment — before `trg_shipments_audit` (`AFTER ROW`) even begins for that same row. The audit trigger's `NVL(:NEW.cancellation_fee, :OLD.cancellation_fee)` then correctly reads that already-calculated value (falling back to `:OLD.cancellation_fee` for `DELETE`, where `:NEW` is meaningless) — exactly satisfying "the audit entry needs to reflect the final, already-calculated cancellation fee."

**Common Mistakes to Watch For:**
- Reflexively adding a `FOLLOWS`/`PRECEDES` clause here — unnecessary, and arguably a sign of not recognizing that `BEFORE` vs. `AFTER` already provides the guarantee. `FOLLOWS`/`PRECEDES` is the right tool specifically when two triggers share the *same* timing + level category (as in Exercise 3 above) — not when the ordering is already structurally fixed by choosing different categories correctly.
- Forgetting the `NVL` wrapping on `:NEW`/`:OLD` fields in the combined-event audit trigger — without it, a `DELETE` firing (where every `:NEW` field is `NULL`) would produce an audit row missing its shipment ID, status, and fee entirely, since those would all try to read from the `NULL` `:NEW` side.
- Merging the two pieces of logic into a single trigger — since one genuinely needs `BEFORE` and the other genuinely needs `AFTER`, and a trigger has exactly one timing category (Topic 3), these must remain two separate trigger objects; there's no single timing choice that serves both correctly.

---

**End of Topic 4.** Next file: `12-compound-triggers-enable-disable-instead-of.md`.