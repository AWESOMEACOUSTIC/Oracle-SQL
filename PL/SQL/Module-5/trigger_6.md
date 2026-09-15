# Topic 6: Combined Practice Assessment on Triggers

> **Syllabus position:** Final item (6 of 6) — the combined practice set
> **Covers:** Every concept from Topics 1–5, mixed freely across realistic, company-style requirements
> **Important:** As with the earlier PL/SQL cursors course's final assessment, none of the six case studies below name which concept(s) they need up front. Each answer reveals what's actually relevant only inside its own "Concepts to Consider" section — after you've had the chance to work out the requirement yourself.

One of these six is also deliberately underspecified, the way real requirements sometimes are. Where that happens, the answer states an explicit assumption rather than silently picking one, and flags it as something you'd confirm with the business before shipping.

---

## Case Study 1: HR Policy Enforcement and Audit Trail

**Business Requirement:**
*"HR needs three protections on the `employees` table: (1) Nobody may delete more than one employee record at a time outside of a formal termination batch window (10pm–2am) — attempting a mass delete outside that window must be blocked entirely, regardless of how many rows would be affected. (2) Whenever an update touches an employee's `salary`, `department_id`, or `job_title`, the new salary must never be lower than the old one — any such update attempting to lower it must be rejected. (3) A complete field-level history must be kept of every genuine change to `salary`, `department_id`, or `job_title` — recording the old and new values — but reassigning a column to its existing value during a routine data-cleanup script must not generate history noise."*

### Requirement Analysis
Three independent rules on the same table, touching different events (`DELETE` vs. `UPDATE`), each needing a different mechanism.

### Important Clues
- (1): "more than one... at a time," "outside of... window," "regardless of how many rows" → a per-statement row count combined with a time check — this needs to know how many rows *this statement* is deleting, which isn't available to a plain statement-level trigger alone.
- (2): "must never be lower than the old one" → an old-vs-new comparison, restricted to specific columns.
- (3): "genuine change... must not generate history noise" → the same syntactic-vs-semantic trap from Topic 2, explicitly called out again.

### Concepts to Consider
A **compound trigger** (Topic 5) for rule (1), since it needs a per-statement counter with no separate package; a `WHEN` clause combined with `UPDATE OF` (Topic 2) for rule (2); and the `DECODE`-based NULL-safe field history pattern (Topic 2) for rule (3) — plus a quiet confirmation that rules (2) and (3), being `BEFORE` and `AFTER` respectively on the same columns, need no `FOLLOWS` clause at all, since their relative order is already guaranteed.

### Solution

**Rule (1) — compound trigger, no separate package needed:**
```sql
CREATE OR REPLACE TRIGGER trg_restrict_mass_delete
FOR DELETE ON employees
COMPOUND TRIGGER

   g_delete_count PLS_INTEGER := 0;

   BEFORE STATEMENT IS
   BEGIN
      g_delete_count := 0;
   END BEFORE STATEMENT;

   BEFORE EACH ROW IS
      v_hour VARCHAR2(2) := TO_CHAR(SYSDATE, 'HH24');
   BEGIN
      g_delete_count := g_delete_count + 1;

      IF g_delete_count > 1 AND NOT (v_hour >= '22' OR v_hour <= '02') THEN
         RAISE_APPLICATION_ERROR(-20100,
            'Deleting more than one employee at a time is only allowed during the 10pm-2am termination batch window.');
      END IF;
   END BEFORE EACH ROW;

END trg_restrict_mass_delete;
/
```

**Rule (2) — `WHEN` + `UPDATE OF`, rejecting immediately:**
```sql
CREATE OR REPLACE TRIGGER trg_no_salary_decrease
BEFORE UPDATE OF salary, department_id, job_title ON employees
FOR EACH ROW
WHEN (NEW.salary < OLD.salary)
BEGIN
   RAISE_APPLICATION_ERROR(-20101, 'Salary cannot decrease as part of this type of update.');
END;
/
```

**Rule (3) — field-level history, `DECODE`-safe:**
```sql
CREATE OR REPLACE TRIGGER trg_employee_field_history
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
Rule (1) uses a compound trigger purely for its automatic per-statement counter — no genuine cross-row *query* against `employees` is needed here (unlike Topic 5's average-salary example), just a simple running count, rejected as soon as it's violated rather than waiting for `AFTER STATEMENT`. This is deliberately a lighter use of the compound-trigger mechanism than Topic 5's canonical example, showing it's just as legitimate for simple per-statement counting as for genuine aggregate validation.

Rules (2) and (3) never need a `FOLLOWS` clause between them: rule (2) is `BEFORE`, rule (3) is `AFTER`, on overlapping columns of the same table — the canonical sequence (Topic 4) already guarantees rule (2) either rejects the update outright or completes first, before rule (3)'s `AFTER` trigger ever sees the row, for every affected row.

### Alternative Approach
Rule (1) could instead be built the "old way" — a plain `BEFORE ROW` trigger plus a package-level counter variable, exactly like Topic 1 Exercise 7's original workaround. It would work identically, but would need its own explicit reset logic and a separate package object purely to hold one integer — the compound trigger version is simpler and self-contained for exactly the reasons established in Topic 5.

### Edge Cases
- Deleting exactly one employee, any time of day, is always allowed — the check only triggers once `g_delete_count` exceeds `1`.
- A `salary` value going from a real number to `NULL` (or vice versa) via one of these UPDATE columns is correctly caught by rule (3)'s `DECODE` check, where a plain `!=` would have missed it.

### Common Mistakes to Watch For
- Using `ALTER TABLE employees DISABLE ALL TRIGGERS` for any future maintenance touching this table — would disable all three of these independent rules at once, when likely only one needs to be suspended for any given reason (a direct callback to Topic 5's enable/disable guidance).
- Checking `salary != old.salary` for rule (3) using a plain comparison instead of `DECODE`, silently missing NULL-involved changes.

---

## Case Study 2: E-commerce Order Processing Pipeline

**Business Requirement:**
*"For the `orders` table: (1) Whenever an order is inserted or its `quantity`/`unit_price` is updated, `total_amount` must be automatically recalculated as `quantity * unit_price`, and a standing 3% loyalty discount must then be applied to that total for any customer flagged as a loyalty member — these two calculations must always happen in a specific, guaranteed order (raw total first, discount second), regardless of deployment order. (2) Once an order's `total_amount` has been finalized and saved, if it exceeds \$10,000, a high-value order notification must be created — but only based on the order's true, final saved amount, never an intermediate value that might still change. (3) No single customer may have more than 5 orders in `'PENDING'` status at the same time; an insert that would push a customer over this limit must be rejected, reflecting the customer's complete, genuine pending-order count across the whole table — not just what this particular batch insert is adding."*

### Requirement Analysis
Three rules, each needing a distinct tool from this course, plus one explicit ordering guarantee between two triggers that share the same timing category.

### Important Clues
- (1): "must always happen in a specific, guaranteed order... regardless of deployment order" → this is a direct, explicit signal that the ordering must be *forced*, not left to chance — and since both calculations are naturally `BEFORE ROW` (both must write `:NEW.total_amount`), their relative order is genuinely **not** automatically guaranteed, unlike some other cases in this course.
- (2): "true, final saved amount, never an intermediate value" → the same reasoning Topic 3's Exercise 7 established for choosing `AFTER`.
- (3): "complete, genuine... count across the whole table" → a real cross-row aggregate query, needing the mutating-table workaround.

### Concepts to Consider
`FOLLOWS` (Topic 4) — genuinely needed here, in contrast to some other case studies in this set where it isn't; the `AFTER`-guarantees-final-value reasoning (Topic 3) for the notification; and a compound trigger (Topic 5) for the pending-order-limit check.

### Solution

**Part 1 — two dependent `BEFORE ROW` triggers, explicitly ordered:**
```sql
CREATE OR REPLACE TRIGGER trg_calc_total
BEFORE INSERT OR UPDATE OF quantity, unit_price ON orders
FOR EACH ROW
BEGIN
   :NEW.total_amount := :NEW.quantity * :NEW.unit_price;
END;
/

CREATE OR REPLACE TRIGGER trg_apply_loyalty_discount
BEFORE INSERT OR UPDATE OF quantity, unit_price ON orders
FOR EACH ROW
FOLLOWS trg_calc_total
DECLARE
   v_is_loyalty customers.is_loyalty_member%TYPE;
BEGIN
   SELECT is_loyalty_member INTO v_is_loyalty FROM customers WHERE customer_id = :NEW.customer_id;
   IF v_is_loyalty = 'Y' THEN
      :NEW.total_amount := :NEW.total_amount * 0.97;
   END IF;
END;
/
```

**Part 2 — notification based on the guaranteed-final value:**
```sql
CREATE OR REPLACE TRIGGER trg_high_value_notification
AFTER INSERT OR UPDATE OF quantity, unit_price ON orders
FOR EACH ROW
WHEN (NEW.total_amount > 10000)
BEGIN
   INSERT INTO notifications (message, created_on)
   VALUES ('High-value order ' || :NEW.order_id || ': ' || :NEW.total_amount, SYSDATE);
END;
/
```

**Part 3 — compound trigger for the genuine cross-row aggregate:**
```sql
CREATE OR REPLACE TRIGGER trg_check_pending_limit
FOR INSERT ON orders
COMPOUND TRIGGER

   TYPE t_customer_list IS TABLE OF orders.customer_id%TYPE;
   g_customers_touched t_customer_list := t_customer_list();

   AFTER EACH ROW IS
   BEGIN
      IF :NEW.status = 'PENDING' THEN
         g_customers_touched.EXTEND;
         g_customers_touched(g_customers_touched.LAST) := :NEW.customer_id;
      END IF;
   END AFTER EACH ROW;

   AFTER STATEMENT IS
      v_pending_count NUMBER;
   BEGIN
      FOR i IN 1 .. g_customers_touched.COUNT LOOP
         SELECT COUNT(*) INTO v_pending_count
         FROM orders
         WHERE customer_id = g_customers_touched(i) AND status = 'PENDING';

         IF v_pending_count > 5 THEN
            RAISE_APPLICATION_ERROR(-20110,
               'Customer ' || g_customers_touched(i) || ' would exceed the 5-pending-order limit.');
         END IF;
      END LOOP;
   END AFTER STATEMENT;

END trg_check_pending_limit;
/
```

### Detailed Explanation
Part 1 is the direct counterexample to the "ordering is often already free" lesson taught elsewhere in this course: `trg_calc_total` and `trg_apply_loyalty_discount` are **both `BEFORE ROW`** — the *same* timing/level category — so nothing structurally guarantees `trg_calc_total` runs first. `FOLLOWS` is genuinely required here, exactly the situation Topic 4's `FOLLOWS` syntax exists for.

Part 2's `WHEN (NEW.total_amount > 10000)` correctly evaluates against the **fully computed, discount-applied** total, because by the time this `AFTER ROW` trigger runs, both `BEFORE ROW` triggers from Part 1 have already completed for that row — `:NEW.total_amount` is genuinely final at this point.

**A subtlety worth checking rather than assuming:** `trg_high_value_notification` (an ordinary `AFTER ROW` trigger) and `trg_check_pending_limit`'s `AFTER EACH ROW` section (part of a compound trigger) are *also* both "`AFTER ROW`-level" pieces on the same table/event, with their relative order technically unspecified. Does this matter here? Checking: `trg_high_value_notification` only reads the already-final `:NEW.total_amount`, and `trg_check_pending_limit`'s row section only reads `:NEW.customer_id`/`:NEW.status` — neither depends on anything the other produces. So, unlike Part 1, no `FOLLOWS` is needed between them, even though they share a timing category. This is worth explicitly verifying case by case, rather than reflexively adding `FOLLOWS` everywhere two same-level triggers coexist.

### Alternative Approach
Part 3's compound trigger rejects the **entire statement** if even one customer, among possibly many in a batch insert, exceeds the limit — meaning other customers' otherwise-fine orders in the same batch get rolled back too. This is a natural, correct consequence of statement-level all-or-nothing semantics (Topic 3), not a flaw, but it's worth explicitly confirming with the business whether "reject the whole batch" or "reject only the offending customer's rows" is actually wanted — the latter would require a fundamentally different design (e.g., processing customers independently rather than as one atomic statement).

### Performance / Practical Considerations
Part 1's `SELECT is_loyalty_member INTO v_is_loyalty FROM customers ...` runs once per row — for very high-volume order inserts, this per-row lookup could be a genuine cost worth monitoring, though it's a different table from `orders`, so there's no mutating-table concern with it.

### Edge Cases
- A non-loyalty customer's order simply skips the discount multiplication in `trg_apply_loyalty_discount`, leaving `total_amount` as `trg_calc_total` set it — correct, since the `IF v_is_loyalty = 'Y'` guard only applies the discount when appropriate.
- A batch insert where no customer exceeds the pending limit produces an empty `g_customers_touched` loop with no rejections — the check silently passes, as expected.

### Common Mistakes to Watch For
- Omitting `FOLLOWS trg_calc_total` from `trg_apply_loyalty_discount` — reintroducing exactly the ordering bug traced through in Topic 3, Exercise 5 and Topic 4's `FOLLOWS` example.
- Trying to check the high-value threshold in a `BEFORE` trigger instead of `AFTER` — would risk seeing an intermediate, not-yet-discounted total, depending on trigger firing order among the `BEFORE` triggers.

---

## Case Study 3: Non-Updatable Reporting View for the Support Team

**Business Requirement:**
*"The support team's ticketing tool works against a `ticket_details_vw` view that joins `tickets` with `customers` to show the customer's name and email alongside each ticket. Support staff need to update ticket status and priority through this view directly, and occasionally close out (delete) a ticket through it — deleting through this view should remove only the ticket, obviously never the underlying customer record. Additionally, support staff must never be able to edit the customer's name or email through this view — attempts to do so should be rejected with a clear message directing them to the customer management system instead."*

### Requirement Analysis
A join-based, non-inherently-updatable view needing explicit `INSTEAD OF` handling, with a protected-columns rejection rule and a deliberate scope decision for `DELETE` — plus one notable gap in the requirement's own scope.

### Important Clues
- "joins `tickets` with `customers`" → not inherently updatable; needs `INSTEAD OF`.
- "must never be able to edit the customer's name or email" → a rejection rule, checked against genuine value changes.
- The requirement discusses `UPDATE` and `DELETE` in detail — but says **nothing at all about `INSERT`** through this view. This omission is worth noticing rather than filling in silently.

### Possible Interpretation / Assumption
Since the requirement is entirely silent on creating new tickets *through this view*, and a reporting-style view joined for display purposes is a plausible candidate for being read/update-only (new tickets more likely originate through a separate intake form or process), the assumption made here is: **`INSERT` is not supported through `ticket_details_vw`**, and no `INSTEAD OF INSERT` trigger is created for it. Attempting an `INSERT` against the view will therefore fail with Oracle's standard "not legal on this view" error. This should be explicitly confirmed with the support team before shipping — if they do need to create tickets through this same view, an `INSTEAD OF INSERT` trigger (following the same two-table-insert pattern from Topic 5, Section 22) would need to be added.

### Concepts to Consider
`INSTEAD OF UPDATE` and `INSTEAD OF DELETE` (Topic 5, Part C), a `DECODE`-based NULL-safe protected-column check (Topic 2, reapplied here), and an explicit, stated business decision about what `DELETE` means for this view (Topic 5's central `INSTEAD OF` design lesson).

### Solution
```sql
CREATE OR REPLACE VIEW ticket_details_vw AS
SELECT t.ticket_id, t.status, t.priority, t.subject,
       c.customer_id, c.customer_name, c.email
FROM tickets t
JOIN customers c ON c.customer_id = t.customer_id;
```
```sql
CREATE OR REPLACE TRIGGER trg_ticket_details_update
INSTEAD OF UPDATE ON ticket_details_vw
FOR EACH ROW
BEGIN
   IF DECODE(:NEW.customer_name, :OLD.customer_name, 0, 1) = 1
      OR DECODE(:NEW.email, :OLD.email, 0, 1) = 1 THEN
      RAISE_APPLICATION_ERROR(-20120,
         'Customer name and email cannot be edited here. Use the customer management system instead.');
   END IF;

   UPDATE tickets
   SET status = :NEW.status, priority = :NEW.priority
   WHERE ticket_id = :OLD.ticket_id;
END;
/

CREATE OR REPLACE TRIGGER trg_ticket_details_delete
INSTEAD OF DELETE ON ticket_details_vw
FOR EACH ROW
BEGIN
   DELETE FROM tickets WHERE ticket_id = :OLD.ticket_id;
   -- Deliberately NOT deleting the customer record.
END;
/
```

### Detailed Explanation
The `DECODE` checks on `customer_name`/`email` reuse the exact NULL-safe pattern from Topic 2 — a plain `!=` comparison would risk missing a rejection if either field happened to be `NULL`. The `UPDATE` trigger's rejection check runs *before* the actual `tickets` update in the same body — since `INSTEAD OF` has no separate `BEFORE` phase, both the validation and the real work naturally live in one place, exactly as covered in Topic 5.

### Alternative Approach
Rather than rejecting an attempted name/email edit outright, the trigger could instead **silently ignore** those two fields (simply never writing them anywhere, with no error) while still applying the `status`/`priority` changes. This is a real design choice with a trade-off: silently ignoring is more forgiving of a UI that naively submits the whole row including unchanged name/email fields, but it also means a *genuine* attempt to change them fails silently rather than with a clear message — the explicit-rejection version chosen above better matches the requirement's own wording ("rejected with a clear message"), which is a strong signal favoring the reject-and-explain approach over silent tolerance.

### Edge Cases
- An `UPDATE` that changes `status`/`priority` **and simultaneously and unintentionally** re-submits the exact same (unchanged) `customer_name`/`email` values — correctly passes, since `DECODE` treats "same value resubmitted" as no change at all, not a rejected edit.
- Attempting `INSERT INTO ticket_details_vw (...)` under the stated assumption → fails with the standard non-updatable-view error, since no `INSTEAD OF INSERT` trigger exists.

### Common Mistakes to Watch For
- Using plain `!=` instead of `DECODE` for the protected-column check, risking a missed rejection if either field is `NULL` for some tickets.
- Deleting the `customers` row instead of (or in addition to) the `tickets` row in the `DELETE` trigger — exactly the kind of scope mistake Topic 5 warned `INSTEAD OF` design requires deliberate care to avoid.
- Silently assuming `INSERT` should also be supported (or shouldn't) without flagging the requirement's gap explicitly — the professional move is stating the assumption, not guessing invisibly either way.

---

## Case Study 4: Nightly Historical Data Migration

**Business Requirement:**
*"The finance team is migrating five years of historical, already-reconciled invoice records from a legacy system into the `invoices` table. During this one-time load: the row-level trigger that auto-stamps `created_by`/`created_date` (which would otherwise overwrite the correct historical values coming from the legacy system) must not fire, and the row-level trigger that sends new-invoice notifications to the finance team's inbox (which would otherwise flood it with thousands of irrelevant notifications for records from years ago) must not fire either. However, the trigger that validates `invoice_total` is never negative must continue to fire throughout the load, since data-quality validation is still required even for historical data. The load must guarantee both suspended triggers are always restored afterward, even if the load itself fails partway through."*

### Requirement Analysis
A migration needing two *specific* triggers suspended, one kept running, with a guaranteed restoration even on failure.

### Important Clues
- "the row-level trigger that... must not fire" (×2), naming specific triggers → targeted, not table-wide, suspension.
- "must continue to fire throughout the load" → explicitly rules out the blunt table-wide disable tool.
- "must guarantee... restored afterward, even if the load itself fails" → the defensive exception-handler pattern from Topic 5.

### Concepts to Consider
`ALTER TRIGGER ... DISABLE`/`ENABLE`, specifically **not** `ALTER TABLE ... DISABLE ALL TRIGGERS` (Topic 5, Part B); `EXECUTE IMMEDIATE` for DDL inside a PL/SQL block; and a guaranteed-restoration pattern using `EXCEPTION WHEN OTHERS`.

### Solution
```sql
BEGIN
   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_set_created_by DISABLE';
   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_notify_new_invoice DISABLE';

   -- ... bulk INSERT of ~5 years of historical invoice records here ...

   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_set_created_by ENABLE';
   EXECUTE IMMEDIATE 'ALTER TRIGGER trg_notify_new_invoice ENABLE';
EXCEPTION
   WHEN OTHERS THEN
      EXECUTE IMMEDIATE 'ALTER TRIGGER trg_set_created_by ENABLE';
      EXECUTE IMMEDIATE 'ALTER TRIGGER trg_notify_new_invoice ENABLE';
      RAISE;
END;
/
```
`trg_validate_invoice_total` (the data-quality trigger) is never touched at all — it simply keeps running throughout the load, exactly as required, because it was never disabled in the first place.

### Detailed Explanation
Using `ALTER TABLE invoices DISABLE ALL TRIGGERS` here would have been a direct violation of the requirement — it would have also suspended `trg_validate_invoice_total`, when the requirement explicitly says data-quality validation must continue. This is precisely the scoped-vs-broad distinction Topic 5 raised: reach for the narrow `ALTER TRIGGER` tool whenever some, but not all, triggers on a table need to be affected.

A quick check on trigger interdependence: does `trg_set_created_by` (presumably `BEFORE ROW`, stamping audit columns) have any genuine ordering dependency on `trg_validate_invoice_total` (presumably also `BEFORE ROW`, rejecting negative totals)? No — neither reads anything the other writes, so no `FOLLOWS` relationship exists or is needed between them, and disabling one has no bearing on how the other behaves.

### Alternative Approach
Rather than disabling `trg_notify_new_invoice` for the whole load, the notification trigger could instead be modified with a `WHEN` clause checking the invoice's age (e.g., `WHEN (NEW.invoice_date >= SYSDATE - 30)`), so it naturally never fires for old historical records regardless of whether it's enabled. This avoids needing to disable/re-enable it at all — but it's a more invasive, permanent change to the trigger's own logic for the sake of a one-time load, and couples the trigger's design to "how old is too old to notify about," a business rule that may not actually belong there. The disable/re-enable approach is more surgical for a genuinely one-time event.

### Performance / Practical Considerations
None of this affects `trg_validate_invoice_total`'s performance cost during the load — it still fires on every one of potentially millions of historical rows, which is by design (data quality still matters), but worth being aware of as a real cost consideration for a load of this scale, separate from the two disabled triggers' now-avoided cost.

### Edge Cases
- If the migration script needs to be re-run after a partial failure (having already correctly re-enabled both triggers via the exception handler), re-running the whole block from the top is safe — `ALTER TRIGGER ... DISABLE` on an already-enabled trigger is a normal, non-erroring operation.

### Common Mistakes to Watch For
- Using the table-wide `DISABLE ALL TRIGGERS` form, accidentally suspending the validation trigger too.
- If, mid-migration, a developer needed to `CREATE OR REPLACE` either `trg_set_created_by` or `trg_notify_new_invoice` for some unrelated reason (say, fixing an unrelated bug discovered during the load), this would **silently re-enable** it immediately — directly reintroducing the exact problem the whole disable step was meant to prevent, per Topic 5's `CREATE OR REPLACE` warning.
- Forgetting the `EXCEPTION WHEN OTHERS` block entirely — leaving both triggers disabled indefinitely if the load fails partway through.

---

## Case Study 5: Cross-Table Cascading Business Rule with Aggregate Validation

**Business Requirement:**
*"Whenever a `payments` row is inserted marking an invoice as paid, the corresponding `invoices` row's `status` must be automatically updated to `'PAID'`. Whenever an invoice's status changes to `'PAID'` this way, the customer's `account_balance` on the `customers` table must be automatically reduced by the invoice's amount. Separately, finance has a rule that a customer's `account_balance` may never go negative as a result of this chain — if a payment would cause that, the entire payment must be rejected. Because this rule depends on the customer's complete, final `account_balance` after all of this transaction's effects (not a value from partway through the cascade), design this carefully, and be explicit about where in the chain the check actually needs to happen and why."*

### Requirement Analysis
A three-table cascading trigger chain (`payments` → `invoices` → `customers`), with a validation rule that must see the truly final state — and the requirement explicitly asks for reasoning about *where* that check belongs, not just what it does.

### Important Clues
- "Whenever X... the corresponding Y must be automatically updated" (×2) → a deliberate, two-hop cascade (Topic 4).
- "the entire payment must be rejected" → the rejection must propagate all the way back through the whole cascade.
- "be explicit about where in the chain the check actually needs to happen and why" → this is the exercise's real center of gravity.

### Concepts to Consider
Cascading trigger execution and error propagation (Topic 4); the `WHEN`-clause NULL-safety pattern (Topics 2/3) for detecting the genuine `PAID` transition; and — critically — recognizing that the final validation needs **no compound trigger and no cross-row query at all**, because the value it needs to check is being directly *set*, not aggregated.

### Thought Process — Where Does the Check Actually Belong?
There are three tables in this chain; the check could naively be placed on any of them. But the requirement's own framing is the clue: it needs the customer's "complete, final `account_balance` after all of this transaction's effects." The **only** point in this entire chain where `account_balance`'s new value is actually being computed and written is the `UPDATE` against `customers` itself, inside the second trigger. Placing the check anywhere else — on `payments` or `invoices` — would mean either duplicating the balance-reduction arithmetic redundantly (risking it drifting out of sync with the real calculation happening elsewhere) or querying `customers` from a table that isn't where the value is actually set, adding unnecessary coupling. The natural, correct place for this check is a simple row-level trigger directly on `customers`, seeing the exact value about to be written — **no aggregate query, no mutating-table concern, and no compound trigger required at all.**

### Solution
```sql
-- Hop 1: payment marks the invoice paid
CREATE OR REPLACE TRIGGER trg_payment_marks_invoice_paid
AFTER INSERT ON payments
FOR EACH ROW
BEGIN
   UPDATE invoices
   SET status = 'PAID'
   WHERE invoice_id = :NEW.invoice_id;
END;
/

-- Hop 2: invoice becoming genuinely PAID reduces the customer's balance
CREATE OR REPLACE TRIGGER trg_invoice_paid_reduces_balance
AFTER UPDATE OF status ON invoices
FOR EACH ROW
WHEN (NEW.status = 'PAID' AND NVL(OLD.status, 'X') != 'PAID')
BEGIN
   UPDATE customers
   SET account_balance = account_balance - :NEW.invoice_amount
   WHERE customer_id = :NEW.customer_id;
END;
/

-- The check itself: lives exactly where the final value is set
CREATE OR REPLACE TRIGGER trg_no_negative_balance
BEFORE UPDATE OF account_balance ON customers
FOR EACH ROW
BEGIN
   IF :NEW.account_balance < 0 THEN
      RAISE_APPLICATION_ERROR(-20130, 'This payment would take the customer''s account balance negative.');
   END IF;
END;
/
```

### Detailed Explanation
Trace what happens when `trg_no_negative_balance` rejects: the error is raised **inside** the `customers` `UPDATE`, which is itself running **inside** `trg_invoice_paid_reduces_balance`'s body (fired by the `invoices` `UPDATE`), which is itself running **inside** `trg_payment_marks_invoice_paid`'s body (fired by the original `INSERT INTO payments`). The error propagates all the way back up through this entire nested cascade, failing the **original `INSERT INTO payments` statement as a whole** — which means the `invoices.status` change and the (attempted) `customers.account_balance` change are **both** rolled back too, exactly matching "the entire payment must be rejected." This is cascading-trigger error propagation (Topic 4) working exactly as it should: all-or-nothing across the whole chain, however many tables deep it goes.

### Alternative Approach
The negative-balance check could instead live inside `trg_invoice_paid_reduces_balance` itself, computing the prospective new balance and checking it there before issuing the `UPDATE`. This works too, but duplicates the subtraction logic in two places (once to *check* what the balance would become, once in the actual `UPDATE` statement that sets it) — a real risk if the calculation ever needs to change and only one copy gets updated. Keeping the check on `customers`, where the value is *actually* set, avoids that duplication entirely.

### Performance / Practical Considerations
This chain fires three trigger executions (one per hop) for every single payment — worth being aware of at scale, though each hop's work is a single targeted-key `UPDATE`, not a table scan, so the per-payment cost stays modest even under volume.

### Edge Cases
- A payment for an invoice that's already `'PAID'` (a duplicate payment, perhaps) → `trg_invoice_paid_reduces_balance`'s `WHEN` clause correctly does **not** fire again, since `NVL(OLD.status, 'X') != 'PAID'` is false when the invoice was already `'PAID'` — the customer's balance isn't double-reduced.
- **Design hygiene worth flagging explicitly:** if `customers` ever gained its *own* trigger that, directly or indirectly, updated `invoices` or `payments`, this three-table chain would risk looping back on itself — exactly the infinite-recursion risk Topic 4 warned about. Anyone extending this schema later should be aware this chain already exists before adding new cross-table triggers anywhere in it.

### Common Mistakes to Watch For
- Placing the negative-balance check on `payments` or `invoices` instead of `customers` — technically possible with a cross-table query, but duplicates logic and adds unnecessary coupling to a table that isn't where the value is actually determined.
- Reaching for a compound trigger here out of habit, since the requirement "sounds" aggregate-flavored ("complete, final account_balance") — but this specific check needs no cross-row query at all, since the value is directly computed and set, not aggregated from many rows. Recognizing *when a compound trigger genuinely isn't needed* is just as important a skill as knowing when it is (contrast this directly with Case Study 2's Part 3, which genuinely does need one).
- Omitting the `NVL` guard on `OLD.status`, risking a missed (or duplicated) balance reduction for an invoice whose prior status was `NULL`.

---

## Case Study 6: Ambiguous Requirement — Change Tracking for a Legacy Product Catalog

**Business Requirement:**
*"The product catalog team wants 'basic change tracking' added to the `products` table so they can see what's been modified over time. They haven't specified exactly which columns matter or what should happen for brand-new products being added to the catalog for the first time."*

### Requirement Analysis
This is deliberately sparse. Two genuine ambiguities sit right in the requirement's own wording, and a professional response has to notice and handle both rather than guess silently.

### Important Ambiguities
1. **Which columns count as "tracked"?** The request never names specific columns.
2. **Do brand-new products (INSERTs) generate a history entry, or only later UPDATEs?** "See what's been modified over time" could reasonably be read either way — does a product's *creation* count as the first thing that's "happened" to it, or does "modified" specifically exclude the act of creation?

### Possible Interpretation / Assumptions (Stated Before Coding)
1. **Tracked columns:** assuming `price`, `category`, and `stock_status` — the three columns most typically relevant to "what changed" for a product catalog team (pricing history, recategorization, availability). This is a reasonable default, but genuinely a guess dressed up as a plausible one — it should be confirmed with the team before this ships, since they may care about other columns (e.g., `product_name`) just as much, or not care about one of these three at all.
2. **New products DO generate a baseline history entry**, recording the initial values as if changing "from nothing." Reasoning: a team asking to "see what's been modified over time" plausibly wants a *complete* picture, including when and with what values a product first appeared — not a history that only starts once the first edit happens. This is stated explicitly as an assumption, not a certainty; the alternative (INSERT doesn't count) is equally defensible, and the solution below notes exactly how small the change would be if that assumption turns out to be wrong.

### Concepts to Consider
A combined-event trigger (Topic 4) for `INSERT` and `UPDATE OF` the three assumed columns; `INSERTING`/`UPDATING` branching (Topics 1/4); the `DECODE` NULL-safe comparison pattern (Topic 2) for the `UPDATE` branch; `AFTER` timing (Topic 3), since this is pure logging with no need to alter the row.

### Solution
```sql
CREATE OR REPLACE TRIGGER trg_products_change_history
AFTER INSERT OR UPDATE OF price, category, stock_status ON products
FOR EACH ROW
BEGIN
   IF INSERTING THEN
      -- Assumption: a new product's initial values are recorded as a baseline entry
      INSERT INTO product_history (product_id, field_changed, old_value, new_value, changed_on, changed_by)
      VALUES (:NEW.product_id, 'PRICE', NULL, TO_CHAR(:NEW.price), SYSDATE, USER);

      INSERT INTO product_history (product_id, field_changed, old_value, new_value, changed_on, changed_by)
      VALUES (:NEW.product_id, 'CATEGORY', NULL, :NEW.category, SYSDATE, USER);

      INSERT INTO product_history (product_id, field_changed, old_value, new_value, changed_on, changed_by)
      VALUES (:NEW.product_id, 'STOCK_STATUS', NULL, :NEW.stock_status, SYSDATE, USER);

   ELSIF UPDATING THEN
      IF DECODE(:NEW.price, :OLD.price, 0, 1) = 1 THEN
         INSERT INTO product_history (product_id, field_changed, old_value, new_value, changed_on, changed_by)
         VALUES (:OLD.product_id, 'PRICE', TO_CHAR(:OLD.price), TO_CHAR(:NEW.price), SYSDATE, USER);
      END IF;

      IF DECODE(:NEW.category, :OLD.category, 0, 1) = 1 THEN
         INSERT INTO product_history (product_id, field_changed, old_value, new_value, changed_on, changed_by)
         VALUES (:OLD.product_id, 'CATEGORY', :OLD.category, :NEW.category, SYSDATE, USER);
      END IF;

      IF DECODE(:NEW.stock_status, :OLD.stock_status, 0, 1) = 1 THEN
         INSERT INTO product_history (product_id, field_changed, old_value, new_value, changed_on, changed_by)
         VALUES (:OLD.product_id, 'STOCK_STATUS', :OLD.stock_status, :NEW.stock_status, SYSDATE, USER);
      END IF;
   END IF;
END;
/
```

### Detailed Explanation
The `INSERTING` branch records all three columns unconditionally, since — under the stated assumption — every new product's starting values are worth a baseline record. The `UPDATING` branch reuses the exact `DECODE` NULL-safe technique from Topic 2 (and reused again in Case Studies 1 and 3 above) to only record columns that *genuinely* changed, not ones merely reassigned to their existing value.

**If the "new products don't count" assumption turns out to be the correct one instead:** the fix is small and localized — simply delete the entire `IF INSERTING THEN ... END IF;` block, and change the trigger header from `AFTER INSERT OR UPDATE OF ...` to just `AFTER UPDATE OF price, category, stock_status ON products`. Flagging this now, before deployment, means that if the assumption is wrong, correcting it is a five-minute change — not a redesign.

### Alternative Approach
Instead of guessing at three specific columns, the trigger could track **every** column on `products` generically, using a more advanced technique (comparing `:OLD`/`:NEW` records as a whole, or building the column list dynamically) — but this adds real complexity for a request that said "basic change tracking," and risks generating noisy history for columns the team never actually cares about (e.g., an internal `last_synced_timestamp` column changing on every sync job). Given the deliberately vague request, starting narrow with an explicit, stated assumption — and being ready to extend it once the team confirms what they actually want — is the more defensible choice than guessing broad.

### Edge Cases
- A product inserted with a `NULL` `price` (perhaps pending a pricing decision) → the baseline history entry correctly records `NULL` as the "new" value (`TO_CHAR(NULL)` is simply `NULL`), which is accurate — there genuinely was no price at creation.
- A later `UPDATE` that sets that same `price` for the first time → correctly detected as a change by `DECODE`, going from `NULL` to a real value.

### Common Mistakes to Watch For
- Silently picking a set of tracked columns (or a stance on whether INSERTs count) without ever stating the assumption anywhere — exactly the trap this case study is built to catch. A real deliverable should document these choices, not bury them invisibly inside the trigger's column list.
- Using plain `!=` instead of `DECODE` in the `UPDATING` branch, reintroducing the NULL-comparison trap one more time.
- Treating "basic change tracking" as license to skip stating any assumptions at all, on the theory that "basic" means "whatever's easiest to build" — the vagueness of the request is exactly why stating assumptions explicitly matters more here, not less.

---

# Course Wrap-Up

Across these six case studies, every syllabus topic from this Triggers course has appeared multiple times, in combination, without being labeled up front:

| Topic | Where it appeared |
|---|---|
| Statement-level triggers, `RAISE_APPLICATION_ERROR` (Topic 1) | Every case study — the foundation none of the others work without. |
| Row-level triggers, `:NEW`/`:OLD`, `WHEN`, `UPDATE OF`, mutating table (Topic 2) | Case Studies 1, 2, 3, 5, 6 — including the `DECODE` NULL-safe pattern reused deliberately in three different case studies. |
| Trigger timing — `BEFORE`/`AFTER`/`INSTEAD OF` (Topic 3) | Case Studies 2 and 5's "check the truly final value" reasoning; Case Study 3's `INSTEAD OF` design. |
| Trigger events and execution order, `FOLLOWS`/`PRECEDES`, cascading triggers (Topic 4) | Case Study 2's genuine need for `FOLLOWS` (contrasted against cases where it's unnecessary); Case Study 5's full cascade trace and error-propagation reasoning. |
| Compound triggers, enable/disable, `INSTEAD OF` in depth (Topic 5) | Case Studies 1 and 2 (compound triggers — one light use, one canonical aggregate use); Case Study 4 (scoped enable/disable with guaranteed restoration); Case Study 3 (full `INSTEAD OF` mechanics). |

Two recurring judgment calls were deliberately tested more than once, in both directions:
- **When `FOLLOWS` is genuinely needed** (Case Study 2, Part 1) **vs. when ordering is already free** by choosing `BEFORE`/`AFTER` correctly (implicit throughout, explicit in Case Study 2's own internal contrast and Case Study 1).
- **When a compound trigger is genuinely needed** for a real cross-row aggregate (Case Study 2, Part 3) **vs. when it isn't**, even though a requirement's wording might sound aggregate-flavored (Case Study 5's deliberate non-example).

If you were able to work through these without needing the mechanism named for you — or recognized which combination of tools a requirement called for before reading each "Concepts to Consider" section — that's the actual goal of this course achieved: not memorizing five pieces of trigger syntax, but recognizing, from a business requirement alone, which combination of tools the problem actually needs.