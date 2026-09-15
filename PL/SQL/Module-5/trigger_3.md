# Topic 3: Trigger Timing (BEFORE, AFTER, INSTEAD OF)

> **Syllabus position:** Topic 3 of 6 (Triggers syllabus)
> **Builds on:** Topics 1 & 2 (statement-level and row-level triggers, `:NEW`/`:OLD`)
> **Scope note:** `INSTEAD OF` is introduced here only at the classification level — what it is and how it's structurally different from `BEFORE`/`AFTER` — since your syllabus places its full mechanical depth (why it's needed, view restrictions, detailed examples) in Topic 5, alongside compound triggers.

---

## 1. Concept

**Timing** determines *when*, relative to the triggering DML actually taking effect, a trigger's body runs. There are exactly three timing categories, and every trigger has **one and only one** of them:

- **`BEFORE`** — fires before the change takes effect (before the statement, or before each row's change, depending on level).
- **`AFTER`** — fires after the change has taken effect.
- **`INSTEAD OF`** — fires *in place of* the triggering DML ever executing directly at all; valid only on views, always effectively row-level. Full depth in Topic 5.

You've already been using `BEFORE` and `AFTER` throughout Topics 1 and 2 — this topic makes the distinction itself the subject, and looks much more closely at the practical consequences of choosing one over the other.

## 2. Purpose / Why It Exists

Different business needs require intervening at different points in a row's or statement's lifecycle:
- **Validate or transform data before it's committed** — reject bad data outright, or auto-populate/derive a column value. This is only possible **before** the write happens.
- **React to a change that has genuinely, finally taken effect** — audit logging that reflects the true final state, or triggering downstream effects that should only happen once you're certain the change actually went through as intended, not some intermediate value that might still be altered by other logic.
- **Redirect DML issued against something not directly writable** — certain views. Deferred to Topic 5.

## 3. What Real-World Problem It Solves

- **`BEFORE`:** data validation and vetoing, auto-derivation of column values, preventing invalid data from ever being written in the first place (avoiding the wasted work of writing then having to undo something).
- **`AFTER`:** audit trails that reflect what was genuinely, finally saved; triggering downstream effects (notifications, summary-table updates) that should only fire based on a row's truly final values, not a value that might still be overwritten by other trigger logic running later in the same processing chain.
- **`INSTEAD OF`:** enabling DML against complex views that wouldn't otherwise be updatable at all (Topic 5).

## 4. When to Use / When NOT to Use

**Use `BEFORE` when:**
- You need to inspect and potentially reject data before it's saved.
- You need to modify `:NEW` — deriving or defaulting a column value. This is **only possible in `BEFORE` row-level triggers** — not a stylistic preference, a hard structural requirement (Topic 2).
- You want to avoid wasted work: rejecting invalid data before it's ever written is cheaper than writing it and then rolling it back.

**Use `AFTER` when:**
- You need the guarantee that the row's (or statement's) change has genuinely, finally taken effect before acting on it.
- You're logging or reacting to a row's values, and there's a risk that another `BEFORE` trigger running later in the same chain could still overwrite the value you'd otherwise be checking (explored in depth below and in Exercise 7).
- You never need to modify the data itself — `AFTER` triggers cannot write to `:NEW` at all.

**Use `INSTEAD OF` when:**
- The DML target is a view that isn't inherently updatable, and users need `INSERT`/`UPDATE`/`DELETE` against it to actually do something meaningful to the underlying base tables. Full depth in Topic 5.

## 5. Syntax

```sql
CREATE [OR REPLACE] TRIGGER trigger_name
{BEFORE | AFTER | INSTEAD OF} {INSERT | UPDATE [OF column_list] | DELETE} [OR ...]
ON {table_name | view_name}
[FOR EACH ROW]
[WHEN (condition)]
BEGIN
   ...
END;
```

**Key syntax rules:**
- `BEFORE`, `AFTER`, and `INSTEAD OF` are **mutually exclusive** — a single `CREATE TRIGGER` statement picks exactly one; you cannot combine or list more than one timing category on the same trigger.
- `INSTEAD OF` is valid **only** `ON` a view — specifying it against a table raises a creation error.
- `INSTEAD OF` triggers are always, in effect, row-level. Oracle's conventional examples always write `FOR EACH ROW` on them, and there's no meaningful "statement-level `INSTEAD OF`" to contrast against the way there is for `BEFORE`/`AFTER`.

## 6. Types / Variations

| Variation | Description |
|---|---|
| `BEFORE STATEMENT` | Fires once, before the whole statement's row processing begins (Topic 1). |
| `BEFORE ROW` | Fires once per row, before that row's change is applied; `:NEW` is writable (Topic 2). |
| `AFTER STATEMENT` | Fires once, after the whole statement's row processing has completed (Topic 1). |
| `AFTER ROW` | Fires once per row, after that row's change is applied; `:NEW` is read-only (Topic 2). |
| `INSTEAD OF` | Replaces the DML entirely; valid only on views (classification here, full depth Topic 5). |

## 7. Simple Examples

### Example A — `BEFORE ROW`: veto + derive, combined
```sql
CREATE OR REPLACE TRIGGER trg_orders_before_row
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW
BEGIN
   IF :NEW.quantity <= 0 THEN
      RAISE_APPLICATION_ERROR(-20040, 'Order quantity must be positive.');
   END IF;

   :NEW.total_amount := :NEW.quantity * :NEW.unit_price;
END;
/
```

### Example B — `AFTER ROW`: audit reflecting the genuinely final change
```sql
CREATE OR REPLACE TRIGGER trg_orders_after_row
AFTER UPDATE OF status ON orders
FOR EACH ROW
BEGIN
   INSERT INTO order_status_history (order_id, old_status, new_status, changed_on)
   VALUES (:OLD.order_id, :OLD.status, :NEW.status, SYSDATE);
END;
/
```

### Example C — `BEFORE STATEMENT`: vetoing an entire operation (Topic 1 callback)
```sql
CREATE OR REPLACE TRIGGER trg_block_offhours_deletes
BEFORE DELETE ON orders
BEGIN
   IF TO_CHAR(SYSDATE, 'HH24') NOT BETWEEN '09' AND '18' THEN
      RAISE_APPLICATION_ERROR(-20041, 'Deletes only allowed 9am-6pm.');
   END IF;
END;
/
```

### Example D — `AFTER STATEMENT`: a downstream summary refresh
```sql
CREATE OR REPLACE TRIGGER trg_refresh_order_summary
AFTER INSERT OR UPDATE OR DELETE ON orders
BEGIN
   UPDATE order_summary SET last_refreshed = SYSDATE WHERE summary_id = 1;
END;
/
```
This runs once, after the whole statement's changes — across however many rows it touched — are all finalized.

### Example E — `INSTEAD OF`, classification-level preview only
```sql
CREATE OR REPLACE VIEW active_customers_vw AS
SELECT customer_id, customer_name, email FROM customers WHERE status = 'ACTIVE';

CREATE OR REPLACE TRIGGER trg_active_customers_insert
INSTEAD OF INSERT ON active_customers_vw
FOR EACH ROW
BEGIN
   INSERT INTO customers (customer_id, customer_name, email, status)
   VALUES (:NEW.customer_id, :NEW.customer_name, :NEW.email, 'ACTIVE');
END;
/
```
**Important caveat about this example:** `active_customers_vw` is actually a simple, single-table view with just a `WHERE` clause — the kind of view that's **already inherently updatable** without any `INSTEAD OF` trigger at all. This example exists purely to show the syntax; it isn't representative of when `INSTEAD OF` is actually *necessary*. That happens once a view involves joins, aggregates, `DISTINCT`, or `GROUP BY` — exactly Topic 5's focus.

## 8. Detailed Explanation

- **`BEFORE ROW` fires before Oracle's own constraint checking** (`NOT NULL`, `CHECK`, foreign key, unique) for that row; **`AFTER ROW` fires after** constraint checking has already passed for that row. This means a `BEFORE` trigger can still set a value that ends up violating a constraint once checking actually happens — the trigger itself doesn't get to "see" whether its own computed value will pass. By the time an `AFTER` trigger runs, you know that row's data was valid enough to satisfy every constraint. (The full firing sequence, including how this interacts with multiple triggers, is Topic 4's subject — this is the timing-specific piece of it.)

- **Multiple `BEFORE ROW` triggers on the same event can see — and overwrite — each other's `:NEW` changes.** If trigger A fires before trigger B (for the same row), and A modifies `:NEW.column`, then B sees A's *modified* value, not the original pre-trigger one. This means firing order can genuinely change the outcome whenever more than one `BEFORE` trigger touches the same column — see Exercise 5 for a worked example of exactly how this can go wrong.

- **`AFTER` does not mean "permanent no matter what."** If a later trigger or a constraint check fails for the *same statement* — even for a completely different row, or a different column on the same row — Oracle rolls back the **entire statement**, including any effects an `AFTER` trigger already had (e.g., an audit row already inserted for an earlier row in a multi-row statement gets rolled back too). "`AFTER`" is relative to that specific row's (or statement's) own completion — it is not an unconditional guarantee of permanence until the surrounding transaction actually commits. Exercise 4 tests this directly.

- **`INSTEAD OF` fundamentally replaces the DML rather than wrapping around it.** With `BEFORE`/`AFTER`, the "real" DML you issued still happens on its own — the trigger just runs at a specific point relative to it. With `INSTEAD OF`, the DML you issued against the view **never inherently happens by itself at all** — whatever actual data changes need to occur must be explicitly coded inside the trigger body (as in Example E's explicit `INSERT` into the base table). This is a structurally different relationship between trigger and triggering statement, not just a third timing option on the same spectrum as `BEFORE`/`AFTER`.

## 9. Common Mistakes and Misconceptions

1. Trying to modify `:NEW` in an `AFTER` trigger — raises `ORA-04084` (restated here specifically as a direct consequence of the timing choice, not just a row-level detail).
2. Assuming `BEFORE ROW` triggers run *after* constraint checks — they run before; a `BEFORE` trigger's computed value can still fail a constraint check that happens afterward.
3. Assuming an `AFTER ROW` trigger's effects (like an audit insert) are permanent the instant they happen — they're only permanent once the enclosing transaction actually commits; a later failure in the same statement undoes them too.
4. Combining `INSTEAD OF` with `BEFORE` or `AFTER` on one trigger — invalid; exactly one timing category per trigger.
5. Attempting to create an `INSTEAD OF` trigger directly on a table — only valid on views.
6. Assuming every view needs an `INSTEAD OF` trigger, or conversely that a simple view never could — the real driver is whether the view is inherently updatable, not the mere fact of it being a view (Topic 5).

## 10. Edge Cases

- Multiple `BEFORE ROW` triggers modifying the same `:NEW` column → whichever one fires **last** wins, since each is applied sequentially against the same `:NEW` record for that row, in whatever order they actually fire.
- A `BEFORE STATEMENT` trigger raising an error → **no row-level triggers for that statement ever fire at all**, since the statement is aborted before row processing even begins.
- An `AFTER STATEMENT` trigger raising an error → all of that statement's row changes (already applied) *and* any earlier `AFTER ROW` trigger effects for that same statement get rolled back too.
- An `INSTEAD OF` trigger raising an error → behaves like any other trigger error: the DML issued against the view is aborted, from the caller's perspective.

## 11. How This Relates to Other Topics

- Directly re-contextualizes Topics 1 & 2's `BEFORE`/`AFTER` usage under a unified timing lens, with the practical consequences (constraint ordering, `:NEW` mutability, rollback behavior) made explicit.
- Sets up **Topic 4**, which formalizes the complete firing sequence (`BEFORE STATEMENT` → `BEFORE ROW` → constraints → `AFTER ROW` → `AFTER STATEMENT`) and multi-trigger ordering via `FOLLOWS`/`PRECEDES`.
- `INSTEAD OF` here is classification-only; **Topic 5** provides its full mechanical depth.

---

## Things You Must Remember

- Three timing categories — `BEFORE`, `AFTER`, `INSTEAD OF` — a trigger has exactly one, never a combination.
- `BEFORE` (row) fires **before** constraint checking; `:NEW` is writable. `AFTER` (row) fires **after** constraint checking; `:NEW` is read-only.
- `BEFORE` is for validation/vetoing/deriving values before the write. `AFTER` is for logging/downstream actions once a change has genuinely, finally taken effect for that row/statement.
- Multiple `BEFORE ROW` triggers on the same event can see and overwrite each other's `:NEW` changes — firing order can matter.
- `AFTER` trigger effects are not unconditionally final — a later failure in the same statement or transaction still rolls them back.
- `INSTEAD OF` only applies to views, replaces the DML entirely, and can't combine with `BEFORE`/`AFTER`. Full depth: Topic 5.

## How to Recognize This Concept

- "before it's saved / reject it before allowing it / derive this value automatically" → `BEFORE`.
- "once it's actually happened / log what was truly saved / trigger a downstream update based on the final value" → `AFTER`.
- "this view isn't directly editable, but users need to insert/update/delete through it anyway" → `INSTEAD OF` (full depth Topic 5).

---

# Exercises — With Answers and Reasoning

## Easy (Syntax & Basics)

### Exercise 1
**Task:** Choose `BEFORE` or `AFTER` (with reasoning) for a trigger that must reject an order if its `total_amount` exceeds a customer's credit limit, then implement it.

**Answer & Reasoning:** `BEFORE`. Rejecting the order means the row must never be written at all — that's only achievable with a `BEFORE` trigger, since by the time an `AFTER` trigger runs, the row has already been saved (even if a later rollback could still undo it, doing the rejection `BEFORE` avoids ever writing invalid data in the first place, which is the cheaper and more idiomatic choice for pure validation).

```sql
CREATE OR REPLACE TRIGGER trg_check_credit_limit
BEFORE INSERT ON orders
FOR EACH ROW
DECLARE
   v_credit_limit customers.credit_limit%TYPE;
BEGIN
   SELECT credit_limit INTO v_credit_limit FROM customers WHERE customer_id = :NEW.customer_id;

   IF :NEW.total_amount > v_credit_limit THEN
      RAISE_APPLICATION_ERROR(-20051, 'Order total exceeds customer credit limit.');
   END IF;
END;
/
```
*(Note: this queries `customers`, a different table from `orders` — no mutating-table conflict, since the restriction only applies to a trigger's own table.)*

---

### Exercise 2
**Task:** Choose `BEFORE` or `AFTER` for a trigger that sends a (simulated) notification once a `shipment` row's status successfully changes to `'DELIVERED'`, then implement it.

**Answer & Reasoning:** `AFTER`. A notification should only fire once you're certain the status change has genuinely, finally taken effect — not a value that might still be altered by some other `BEFORE` trigger running later in the same chain. Nothing here needs to modify data, either, which rules out any reason to prefer `BEFORE`.

```sql
CREATE OR REPLACE TRIGGER trg_notify_delivered
AFTER UPDATE OF status ON shipment
FOR EACH ROW
WHEN (NEW.status = 'DELIVERED' AND NVL(OLD.status, 'X') != 'DELIVERED')
BEGIN
   INSERT INTO notifications (message, created_on)
   VALUES ('Shipment ' || :OLD.shipment_id || ' has been delivered.', SYSDATE);
END;
/
```

---

### Exercise 3
**Task:** On a `products` table with `price` and `discounted_price` columns, write a `BEFORE` row-level trigger that auto-calculates `discounted_price` as 90% of `price` whenever a row is inserted or `price` is updated.

**Solution:**
```sql
CREATE OR REPLACE TRIGGER trg_set_discounted_price
BEFORE INSERT OR UPDATE OF price ON products
FOR EACH ROW
BEGIN
   :NEW.discounted_price := :NEW.price * 0.90;
END;
/
```

**Reasoning:** `BEFORE` is the only option that can actually write to `:NEW.discounted_price` — this isn't a stylistic choice, it's dictated entirely by the need to derive a column value (Topic 2, restated here as a direct consequence of timing).

---

### Exercise 4
**Task — True or False, with reasoning:** *"An `AFTER ROW` trigger can still prevent the row's change from being saved by calling `RAISE_APPLICATION_ERROR`."*

**Answer: True.** This is a genuinely easy point to get backwards. An `AFTER ROW` trigger absolutely **can** call `RAISE_APPLICATION_ERROR`, and doing so aborts and rolls back the **entire triggering statement** — including that row's just-applied change, and any other rows already processed within the same statement. "`AFTER`" only means "this specific row's change has already happened, and its constraint checks have already passed" — it does **not** mean the statement can no longer be stopped. The row's change is *provisionally* applied at that point, still fully subject to rollback if anything — including the `AFTER` trigger itself — decides the statement should fail. This is the same underlying idea as the "`AFTER` isn't unconditionally final" point from the Detailed Explanation, now applied to a case where the trigger *itself* is the thing causing the failure, rather than something else failing later.

---

## Intermediate (Reasoning & Edge Cases)

### Exercise 5
**Task:** Two `BEFORE ROW` triggers exist on `orders` for `INSERT`. Trigger A: `:NEW.total_amount := :NEW.quantity * :NEW.unit_price;`. Trigger B: `:NEW.total_amount := :NEW.total_amount * 0.95;` (a standing 5% discount). If A fires before B, what's the final `total_amount`? What if B fires before A? What determines which happens, and how could a developer force a specific order?

**Answer:**

**If A fires first, then B:** A computes `total_amount = quantity * unit_price` first. B then multiplies *that* value by `0.95`. Final result: `(quantity * unit_price) * 0.95` — the intended, correct discounted total.

**If B fires first, then A:** B multiplies `:NEW.total_amount` by `0.95` — but at this point, A hasn't run yet, so `:NEW.total_amount` is still whatever it was *before* either trigger touched it (its default, likely `NULL` or `0`, since nothing has computed it yet). B's multiplication either produces `NULL` (if starting from `NULL`) or `0`. Then A runs and **completely overwrites** `:NEW.total_amount` with `quantity * unit_price` — discarding B's discount entirely, since A's assignment doesn't reference the prior value at all, it just replaces it outright. Final result: the **full, undiscounted** total — B's logic had no lasting effect whatsoever.

**What determines the order:** absent any explicit control, the firing order between two `BEFORE ROW` triggers of the same event on the same table is **not guaranteed** — exactly the point flagged in Topic 1 and repeated in this topic's Detailed Explanation.

**How to force a specific order:** Oracle provides a `FOLLOWS` (or `PRECEDES`) clause on `CREATE TRIGGER` specifically to declare that one trigger must fire after (or before) a named one — full syntax is Topic 4's subject, named here only because this exercise is exactly the situation where it becomes necessary. **A more robust alternative**, worth considering before reaching for ordering control at all: since A and B are genuinely *interdependent* (B's correctness depends entirely on A having already run), the more resilient fix is often to **not split this logic across two separate triggers in the first place** — combine both calculations into a single `BEFORE ROW` trigger, removing the ordering dependency entirely rather than just pinning it down. Ordering control (`FOLLOWS`) is the right tool when two triggers are logically independent but happen to need a specific sequence for unrelated reasons; when they're this tightly coupled, consolidating them removes the fragility altogether.

---

### Exercise 6
**Task:** A `BEFORE UPDATE ON orders FOR EACH ROW` trigger successfully computes and sets `:NEW.total_amount`. Immediately after, a `NOT NULL` constraint check on a *different* column (left `NULL` by the application) fails for that same row. What happens to the `total_amount` value the trigger computed?

**Answer:** It's discarded entirely, along with the rest of the statement. Since the constraint failure happens for that row *after* the `BEFORE` trigger already ran and set `:NEW.total_amount`, the constraint violation aborts the **whole `UPDATE` statement** — constraint failures are all-or-nothing at the statement level (and further, at the transaction level, since an explicit `ROLLBACK` would undo even more). The row is never actually written with that carefully computed `total_amount` at all; the trigger's work for that row simply never materializes into stored data. This is the direct, practical consequence of "`BEFORE ROW` runs before constraint checking" from the Detailed Explanation: setting a perfectly valid computed value for *one* column offers no protection against a constraint problem on a completely different, untouched column in the same row.

---

## Realistic Scenario

### Exercise 7
**Business Requirement:** *"The finance team needs two related but distinct trigger behaviors on the `invoices` table: (1) Before any invoice is saved (whether newly created or updated), automatically calculate its `tax_amount` as 8% of `subtotal`, and reject the save entirely if `subtotal` is negative. (2) Once an invoice's `status` has genuinely changed to `'PAID'` (not just been re-saved with the same status), send a confirmation entry to a `payment_confirmations` table capturing the invoice ID and the exact moment it became PAID — and this confirmation must never end up reflecting a status that wasn't truly, finally saved as PAID."*

**What Is Being Asked:**
Two independent trigger behaviors on the same table, each requiring a **different** timing choice for a **specific, defensible reason** — not just "BEFORE for validation, AFTER for logging" as a rule of thumb, but reasons tied directly to what this topic has actually covered.

**Key Clues:**
- Part (1): "automatically calculate... reject the save" → needs to write `:NEW.tax_amount` (only possible `BEFORE`) and veto invalid data before it's written.
- Part (2): "genuinely changed to PAID... must never end up reflecting a status that wasn't truly, finally saved" → this phrasing is doing real work, and is the interesting part of this exercise.

### Part 1: Why `BEFORE` Is Not Optional Here
```sql
CREATE OR REPLACE TRIGGER trg_invoices_before_row
BEFORE INSERT OR UPDATE ON invoices
FOR EACH ROW
BEGIN
   IF :NEW.subtotal < 0 THEN
      RAISE_APPLICATION_ERROR(-20050, 'Invoice subtotal cannot be negative.');
   END IF;

   :NEW.tax_amount := :NEW.subtotal * 0.08;
END;
/
```
This isn't really a "choice" between `BEFORE` and `AFTER` at all — the requirement to *write* `:NEW.tax_amount` dictates `BEFORE` outright, since `AFTER` structurally cannot modify `:NEW` (Topic 2). The validation is naturally bundled into the same trigger since it's cheap to check before doing the calculation.

### Part 2: Why `AFTER` Is the Genuinely Correct Choice — Not Just the Conventional One
A first-pass justification might be "use `AFTER` because if the statement fails later, the confirmation insert gets rolled back too." That's **true**, but it's actually **not a real distinguishing reason** — *any* DML performed inside a transaction, whether from a `BEFORE` or an `AFTER` trigger, rolls back if the statement or transaction ultimately fails. That protection isn't unique to `AFTER` at all, so it can't be the reason to prefer it here.

**The real, distinguishing reason** is the one buried in the requirement's phrasing: "must never end up reflecting a status that wasn't truly, finally saved." Recall from this topic's Detailed Explanation: multiple `BEFORE ROW` triggers on the same table/event can fire in an unguaranteed order, and a *later*-firing `BEFORE` trigger can still overwrite `:NEW.status` after an *earlier* one has already looked at it. If this confirmation logic were written as a `BEFORE` trigger instead, and some other `BEFORE` trigger on `invoices` (present now, or added later by someone else) happened to also touch `status` and fired *after* this one, the confirmation could end up being generated based on a `status` value that gets changed again before the row is actually written — recording a "became PAID" confirmation for a status that, in the row actually saved, might not even be `'PAID'` anymore. An `AFTER` trigger has no such risk: by the time it runs, `:NEW.status` is the row's **true, final, no-longer-changeable** value for that row, because nothing can modify `:NEW` once you're past the `BEFORE` phase. This is the genuine reason `AFTER` is correct here — not rollback safety (which both timings share equally), but the guarantee that what you're checking can't be silently overwritten by something else afterward.

```sql
CREATE OR REPLACE TRIGGER trg_invoices_paid_confirmation
AFTER UPDATE OF status ON invoices
FOR EACH ROW
WHEN (NEW.status = 'PAID' AND NVL(OLD.status, 'X') != 'PAID')
BEGIN
   INSERT INTO payment_confirmations (invoice_id, confirmed_on)
   VALUES (:OLD.invoice_id, SYSDATE);
END;
/
```
`NVL(OLD.status, 'X')` guards against `OLD.status` being `NULL` (e.g., a brand-new invoice column that hadn't been populated before) — without it, `OLD.status != 'PAID'` would evaluate to `UNKNOWN`, not `TRUE`, for a `NULL` old status, per the same NULL-comparison caution covered repeatedly in earlier topics.

### Common Mistakes to Watch For
- Assuming "roll back on failure" is what distinguishes `AFTER` from `BEFORE` for the confirmation logic — as explained above, that protection applies equally to both; the real reason to prefer `AFTER` is about guaranteeing the *final* value, not about transactional safety.
- Trying to compute `tax_amount` in an `AFTER` trigger — structurally impossible; `:NEW` is read-only there.
- Omitting the `NVL` guard on `OLD.status` and silently missing a "became PAID" event for an invoice whose prior status happened to be `NULL`.
- Merging both behaviors into a single trigger with mixed timing needs — since Part 1 *must* be `BEFORE` and Part 2 *should* be `AFTER` for the reasons given above, these genuinely need to remain two separate triggers; there's no single timing choice that correctly serves both requirements at once.

---

**End of Topic 3.** Next file: `11-trigger-events-and-execution-order.md`.