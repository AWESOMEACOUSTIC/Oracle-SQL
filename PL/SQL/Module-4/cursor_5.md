# Topic 5: FOR UPDATE & WHERE CURRENT OF Clause

> **Syllabus position:** Topic 5 of 6 (+ final combined practice set)
> **Builds on:** Topics 1–4 (lifecycle, parameters, decision-making, FOR loops)
> **Feeds into:** Topic 6 (REF CURSOR) — this topic's locking behavior applies equally whether you consume the cursor manually, via a FOR loop, or eventually via a REF CURSOR

---

## 1. Concept

- **`FOR UPDATE`** is a clause added to a cursor's `SELECT` statement that **locks** every row in the active set — preventing other sessions from updating, deleting, or also locking those same rows — from the moment the cursor is opened until your transaction ends (`COMMIT` or `ROLLBACK`).
- **`WHERE CURRENT OF cursor_name`** is used inside an `UPDATE` or `DELETE` statement to target **exactly the row the cursor is currently positioned on** — the row from the most recent `FETCH` — without repeating any key or business-logic conditions.

```sql
CURSOR c_orders IS
   SELECT order_id, status FROM orders WHERE status = 'PENDING' FOR UPDATE;
...
UPDATE orders SET status = 'APPROVED' WHERE CURRENT OF c_orders;
```

## 2. Purpose / Why It Exists

Consider a "read a row, decide something based on it, then write a new value back to that same row" pattern — extremely common in batch processing. Without locking, there's a window between the `FETCH` and the eventual `UPDATE` during which **another session could modify or delete that very row**. If your `UPDATE` then runs against stale assumptions (e.g., you approved an order based on stock levels that have since changed), you get a **lost update** — a classic concurrency bug. `FOR UPDATE` closes that window by locking the row the instant it's part of the cursor's active set, guaranteeing nothing else can change it while you're deciding what to do with it.

`WHERE CURRENT OF` solves a smaller, related problem: once you've fetched a row, how do you update *exactly that row* afterward? You could re-select its primary key and write a normal `UPDATE ... WHERE id = v_id`, but that means selecting an extra column purely to re-identify the row, and re-matching by key value rather than by the row's actual physical position. `WHERE CURRENT OF` avoids both — it references the cursor's current row directly (internally, via `ROWID`), so it's simpler to write and doesn't depend on the row having a convenient, unique key value to filter on.

## 3. What Real-World Problem It Solves

Any batch or interactive process that follows the pattern **read → decide → write, on the same rows, in a system with concurrent users or processes** — nightly jobs approving/rejecting pending records, worker processes pulling tasks off a shared queue table, or any "process each row and update its status" workflow where more than one process could plausibly touch the same data at the same time.

## 4. When to Use / When NOT to Use

**Use `FOR UPDATE` + `WHERE CURRENT OF` when:**
- You will update or delete the same rows you're iterating, based on decisions made per row.
- You need a guarantee that no other session changes those rows between your read and your write.
- The write is a targeted change to "the row I just fetched" — not a broader, unrelated update.

**Do NOT use `FOR UPDATE` when:**
- You're only reading the data, with no subsequent write to those same rows — locking would needlessly block other users for no benefit.
- The write your business logic needs isn't confined to the row just fetched (e.g., incrementing a summary counter in a completely different table) — `WHERE CURRENT OF` only ever applies to the table the cursor locked, at the cursor's current position; unrelated writes still need ordinary `WHERE` conditions.
- You're processing a very large number of rows in a high-concurrency, always-on OLTP system — locking a huge active set for the duration of slow, row-by-row processing can create real blocking/contention for other users. This is a genuine production trade-off to weigh, not just a syntax choice (see Exercise 6 for the specific trap this leads to).

**Recognition clue:** phrases like "make sure no other process changes this while we're deciding," "must not be modified concurrently," "lock the record while it's being processed," or "prevent two people from updating the same row at once" are direct signals for this topic.

## 5. Syntax

```sql
CURSOR cursor_name IS
   SELECT column1, column2, ...
   FROM table_name
   WHERE condition
   FOR UPDATE [OF column_list] [NOWAIT | WAIT n | SKIP LOCKED];
```

Then, inside the loop, exactly one of:
```sql
UPDATE table_name SET column = new_value WHERE CURRENT OF cursor_name;
DELETE FROM table_name WHERE CURRENT OF cursor_name;
```

### Syntax Breakdown

- **`FOR UPDATE`** — with no other options, Oracle attempts to lock **every row matching the query's WHERE clause, all at once, at `OPEN` time** — not incrementally as each row is later fetched. If the query would return 10,000 rows, `OPEN` attempts to lock all 10,000 immediately. If any row is already locked by another session, `OPEN` **waits indefinitely** by default until it becomes available.
- **`OF column_list`** — optional; mainly relevant when the cursor's query **joins multiple tables**. Naming a column here tells Oracle to lock only the rows of the table that column belongs to. Without `OF` in a multi-table join, **all** tables referenced in the query have their matching rows locked — often more locking than you actually need.
- **`NOWAIT`** — if any qualifying row is already locked by another session, `OPEN` fails **immediately** with `ORA-00054: resource busy and acquire with NOWAIT specified`, instead of waiting.
- **`WAIT n`** — waits up to `n` seconds for the lock; raises the same error if still unavailable after that.
- **`SKIP LOCKED`** — rows already locked by other sessions are **silently excluded** from the active set entirely; only currently-unlocked matching rows are returned and locked. No error, no waiting. This is the standard pattern for concurrent "worker" processes pulling unclaimed work off a shared queue table.
- **`WHERE CURRENT OF cursor_name`** — valid only inside an `UPDATE` or `DELETE`, and only when the referenced cursor was declared `FOR UPDATE`. It always targets the single row most recently returned by that cursor's last `FETCH`.

### Important Restriction

A cursor's query **cannot** use `FOR UPDATE` together with `DISTINCT`, `GROUP BY`, aggregate functions, set operators (`UNION`, `UNION ALL`, `INTERSECT`, `MINUS`), or `CONNECT BY`/`START WITH`. This makes sense once you think about *why*: `FOR UPDATE` locks actual, identifiable base-table rows, and any of these constructs can produce a result set that no longer maps one-to-one, unambiguously, back to single physical rows in a single table.

## 6. Types / Variations

| Variation | Behavior |
|---|---|
| `FOR UPDATE` (default) | Locks the whole active set at `OPEN`; waits indefinitely for any conflicting locks. |
| `FOR UPDATE NOWAIT` | Fails immediately (`ORA-00054`) if **any** row in the active set is already locked. |
| `FOR UPDATE WAIT n` | Waits up to `n` seconds, then fails the same way if still unavailable. |
| `FOR UPDATE SKIP LOCKED` | Silently excludes already-locked rows from the active set; no error. |
| `FOR UPDATE OF column_list` | Restricts locking to a specific table's rows in a multi-table join. |
| `WHERE CURRENT OF` + `UPDATE` | Modifies the cursor's current row. |
| `WHERE CURRENT OF` + `DELETE` | Removes the cursor's current row. |

## 7. Simple Examples

### Example A — Basic pattern
```sql
DECLARE
   CURSOR c_orders IS
      SELECT order_id, status FROM orders WHERE status = 'PENDING' FOR UPDATE;
BEGIN
   FOR order_rec IN c_orders LOOP
      IF order_rec.order_id MOD 2 = 0 THEN   -- placeholder business condition
         UPDATE orders SET status = 'APPROVED' WHERE CURRENT OF c_orders;
      ELSE
         UPDATE orders SET status = 'REJECTED' WHERE CURRENT OF c_orders;
      END IF;
   END LOOP;
   COMMIT;
END;
/
```

### Example B — NOWAIT
```sql
CURSOR c_accounts IS
   SELECT account_id FROM accounts WHERE balance < 0 FOR UPDATE NOWAIT;
```
If even one qualifying account row is already locked by another session, this cursor's `OPEN` fails immediately with `ORA-00054` — nothing gets locked or processed in this attempt.

### Example C — SKIP LOCKED (job-queue pattern)
```sql
CURSOR c_jobs IS
   SELECT job_id FROM job_queue WHERE status = 'READY' FOR UPDATE SKIP LOCKED;
```
Multiple worker sessions can run identical code concurrently; each grabs only the currently-unlocked ready jobs, naturally partitioning the work without collisions or errors.

### Example D — `FOR UPDATE OF` in a join
```sql
CURSOR c_order_items IS
   SELECT o.order_id, oi.item_id, oi.quantity
   FROM orders o
   JOIN order_items oi ON oi.order_id = o.order_id
   WHERE o.status = 'PENDING'
   FOR UPDATE OF oi.quantity;   -- locks only order_items rows, not orders rows
```

### Example E — DELETE with `WHERE CURRENT OF`
```sql
DECLARE
   CURSOR c_expired IS
      SELECT session_id FROM sessions WHERE last_active < SYSDATE - 1 FOR UPDATE;
BEGIN
   FOR sess_rec IN c_expired LOOP
      DELETE FROM sessions WHERE CURRENT OF c_expired;
   END LOOP;
   COMMIT;
END;
/
```

## 8. Detailed Explanation

- **Locking happens for the whole active set at `OPEN`, not per fetch.** This is one of the most commonly misunderstood points — many learners assume rows get locked one at a time as they're fetched. They don't; Oracle locks everything the query matches, immediately, as part of `OPEN`.
- **Closing the cursor does NOT release the locks.** Only `COMMIT` or `ROLLBACK` releases row locks. You can `CLOSE` a `FOR UPDATE` cursor and still be holding every one of its locks minutes later if you haven't ended the transaction — a common source of "why is this row still locked?" confusion in production.
- **`WHERE CURRENT OF` is implemented via `ROWID` internally**, which is why it's both simpler to write and slightly more robust than re-matching by a business key — it doesn't care whether the row's other column values are unique, only that it's "the physical row this cursor is currently on."
- **`FOR UPDATE OF` and `WHERE CURRENT OF` together in a join:** if `FOR UPDATE OF` names a column from only one table in the join, `WHERE CURRENT OF` can only be used to update/delete **that** table — not any other table referenced in the join, even though it appeared in the SELECT.
- **Other sessions can still read locked rows normally.** `FOR UPDATE` blocks other sessions from updating, deleting, or also `SELECT ... FOR UPDATE`-ing the same rows, but an ordinary `SELECT` (no `FOR UPDATE`) from another session is generally **not** blocked, thanks to Oracle's read-consistency model — they simply see the last committed version of the row.
- **`OPEN` with `NOWAIT`/`WAIT` fails or blocks for the entire active set, not row-by-row.** If even a single row out of thousands is already locked by someone else, the *whole* `OPEN` fails (with `NOWAIT`) or blocks (with `WAIT`) — none of the other, unlocked rows get processed either, in that attempt. This is a critical distinction from `SKIP LOCKED`, which is the only option of the three that lets processing continue on the rows that *are* available. Confusing `NOWAIT` with "skip the locked ones and continue" is a very natural but incorrect assumption — see Exercise 7's Alternative Approach for a worked comparison.

## 9. Common Mistakes and Misconceptions

1. Assuming rows are locked one at a time as fetched, rather than all at once at `OPEN`.
2. Assuming `CLOSE` releases locks — only `COMMIT`/`ROLLBACK` does.
3. Manually tracking a primary key just to write a separate `UPDATE ... WHERE id = v_id` instead of using `WHERE CURRENT OF` — extra, unnecessary work and a missed simplification.
4. Using `WHERE CURRENT OF` against a cursor that wasn't declared `FOR UPDATE` — this is not allowed and Oracle raises a runtime error; `WHERE CURRENT OF` has a hard dependency on the cursor being opened for update.
5. Using `FOR UPDATE OF` in a join, then trying `WHERE CURRENT OF` against a table that wasn't the one named in `OF` — not permitted; only the locked table can be targeted this way.
6. Confusing `NOWAIT` with "skip locked rows and keep going" — `NOWAIT` fails the **entire** `OPEN` if even one row is locked; only `SKIP LOCKED` provides the "continue with what's available" behavior.
7. **Forgetting the final `COMMIT`** after processing — locks remain held for as long as the session's transaction stays open, silently blocking other users or processes indefinitely. This is a real, frequently-seen production issue ("the batch job finished ages ago but nobody can update these records") almost always traced back to a missing `COMMIT`.
8. Committing partway through a loop that's still fetching from an open `FOR UPDATE` cursor (see Exercise 6) — a subtle but serious mistake, not just a style issue.

## 10. Edge Cases

- Zero matching rows → `OPEN` succeeds (nothing to lock), loop body never runs — same as any ordinary cursor.
- `NOWAIT`/`WAIT` failure happens **at `OPEN`**, not inside the fetch loop — any exception handling for a lock conflict needs to wrap the `OPEN` statement, not the loop body.
- `FOR UPDATE` combines fine with cursor FOR loops (Topic 4) — `WHERE CURRENT OF` still correctly refers to the loop's current row; none of this topic's behavior changes based on how you consume the cursor.
- Committing while a `FOR UPDATE` cursor remains open and you intend to keep fetching from it is a documented risk — it can raise `ORA-01002: fetch out of sequence`, and even where it doesn't immediately error, it breaks the very locking guarantee `FOR UPDATE` was providing for the rows not yet processed (see Exercise 6).

## 11. How This Relates to Other Topics

- Builds directly on the lifecycle from Topic 1 — `FOR UPDATE` only changes what happens at `OPEN`; everything else about `FETCH`/`EXIT WHEN`/`CLOSE` is unchanged.
- Combines naturally with cursor FOR loops (Topic 4), as shown throughout this topic's examples.
- Parameters (Topic 2) work identically on a `FOR UPDATE` cursor — nothing about locking changes how parameters are declared or passed.
- **Topic 6 (REF CURSOR):** locking behavior is unaffected by whether a cursor is static or a REF CURSOR — the same `FOR UPDATE`/`WHERE CURRENT OF` rules apply regardless of how the cursor is ultimately declared or passed around.

---

## Things You Must Remember

- `FOR UPDATE` locks the **entire active set at `OPEN`**, not incrementally per fetch.
- Locks are released only by `COMMIT`/`ROLLBACK` — **never** by `CLOSE`.
- `WHERE CURRENT OF` requires the cursor to have been declared `FOR UPDATE`; it targets exactly the row from the most recent `FETCH`, via `ROWID`.
- `NOWAIT`/`WAIT` failures affect the **whole** active set if even one row is locked; only `SKIP LOCKED` lets you continue processing the rows that are actually available.
- `FOR UPDATE` cannot be combined with `DISTINCT`, `GROUP BY`, aggregates, set operators, or `CONNECT BY`/`START WITH`.
- In a join, `FOR UPDATE OF` restricts both the locking and the valid target of `WHERE CURRENT OF` to one specific table.
- Never commit in the middle of a loop that's still fetching from an open `FOR UPDATE` cursor.

## How to Recognize This Concept

Think **`FOR UPDATE` + `WHERE CURRENT OF`** when a requirement includes language like:
- "must not be modified by another process while this is running"
- "lock the record while it's being reviewed/processed"
- "prevent two people/processes from updating the same row at the same time"
- "read, decide, then update the very same rows" in a system with concurrent users
- A queue/worker pattern: "each worker should grab only unclaimed rows" → `FOR UPDATE SKIP LOCKED` specifically.

---

# Exercises — With Answers and Reasoning

## Easy (Syntax & Basics)

### Exercise 1
**Task:** Extend the Topic 4 products cursor (Electronics category) so that rows are locked `FOR UPDATE`, and if `unit_price` is below a minimum threshold, update it to that minimum using `WHERE CURRENT OF`.

**Solution:**
```sql
DECLARE
   CURSOR c_products IS
      SELECT product_id, product_name, unit_price
      FROM products
      WHERE category = 'Electronics'
      FOR UPDATE;

   v_min_price CONSTANT products.unit_price%TYPE := 500;
BEGIN
   FOR prod_rec IN c_products LOOP
      IF prod_rec.unit_price < v_min_price THEN
         UPDATE products
         SET unit_price = v_min_price
         WHERE CURRENT OF c_products;
      END IF;
   END LOOP;
   COMMIT;
END;
/
```

**Reasoning:** `FOR UPDATE` locks every matching Electronics row the moment the cursor opens, so no other session can change `unit_price` while this loop is deciding which rows need adjusting. `WHERE CURRENT OF c_products` updates exactly the row just evaluated — no need to separately track or re-match on `product_id`. `COMMIT` at the end releases all locks acquired during the run.

---

### Exercise 2
**Task:** Write a cursor with `FOR UPDATE NOWAIT` on an `accounts` table for accounts with a negative balance, and explain what happens if another session already holds a lock on one of those rows.

**Solution:**
```sql
DECLARE
   CURSOR c_neg_accounts IS
      SELECT account_id, balance FROM accounts WHERE balance < 0 FOR UPDATE NOWAIT;
BEGIN
   OPEN c_neg_accounts;
   -- ... fetch loop ...
   CLOSE c_neg_accounts;
EXCEPTION
   WHEN OTHERS THEN
      IF SQLCODE = -54 THEN
         DBMS_OUTPUT.PUT_LINE('Could not lock all accounts - one or more rows are already locked by another session.');
      ELSE
         RAISE;
      END IF;
END;
/
```

**Reasoning:** With `NOWAIT`, if **any** negative-balance account row is already locked by another session, the `OPEN` statement itself fails immediately with `ORA-00054: resource busy and acquire with NOWAIT specified`, before a single row has been fetched. Note this failure is on the **whole attempt** — even accounts that aren't locked by anyone don't get processed in this run, since the entire `OPEN` failed. `SQLCODE = -54` corresponds to `ORA-00054`, and checking it here lets the block distinguish this specific, expected lock-conflict condition from any other unexpected error.

---

### Exercise 3
**Task:** Simulate a job-queue "worker" pattern: grab rows from a `tasks` table where `status = 'NEW'`, locking them but skipping any already locked by other workers, then mark each grabbed row as `'IN_PROGRESS'`.

**Solution:**
```sql
DECLARE
   CURSOR c_tasks IS
      SELECT task_id FROM tasks WHERE status = 'NEW' FOR UPDATE SKIP LOCKED;
BEGIN
   FOR task_rec IN c_tasks LOOP
      UPDATE tasks SET status = 'IN_PROGRESS' WHERE CURRENT OF c_tasks;
   END LOOP;
   COMMIT;
END;
/
```

**Reasoning:** With `SKIP LOCKED`, any `'NEW'` task row already locked by a different, concurrently-running worker session is simply excluded from this cursor's active set — no error, no waiting. Multiple copies of this exact script can run at the same time (e.g., several worker processes), and each invocation naturally ends up claiming only the tasks that aren't already claimed elsewhere, safely partitioning the work without any collision-handling logic needed.

---

### Exercise 4
**Task:** Write a `FOR UPDATE` cursor that joins `orders` and `customers`, but correctly restricts locking to only the `orders` rows.

**Solution:**
```sql
CURSOR c_order_customer IS
   SELECT o.order_id, o.status, c.customer_name
   FROM orders o
   JOIN customers c ON c.customer_id = o.customer_id
   WHERE o.status = 'PENDING'
   FOR UPDATE OF o.status;
```

**Reasoning:** Naming `o.status` in the `OF` clause tells Oracle to lock only the `orders` table's matching rows; the joined `customers` rows are read normally, unlocked. If `OF o.status` were omitted here, both `orders` and `customers` rows involved in the join would be locked — unnecessary and potentially disruptive, since this business logic (per the requirement) only ever intends to modify orders, not customers. Any later `WHERE CURRENT OF c_order_customer` in an `UPDATE`/`DELETE` must target `orders`, since that's the table actually locked.

---

## Intermediate (Reasoning & Edge Cases)

### Exercise 5
**Task — True or False, with reasoning:** *"Closing a cursor that was opened `FOR UPDATE` releases the row locks it acquired."*

**Answer: False.** Closing a cursor releases the cursor's own private SQL area (its internal resources) — it has **no effect whatsoever** on row locks. Row locks are a **transaction-level** resource, controlled exclusively by `COMMIT` or `ROLLBACK`. This means it is entirely possible to `CLOSE` a `FOR UPDATE` cursor and still be holding every lock it acquired, indefinitely, until the transaction is explicitly ended. This is a genuinely common source of confusion and a real production issue: a developer sees the cursor is closed and assumes the locks are gone, while other users remain blocked because the session simply never issued a final `COMMIT`.

---

### Exercise 6
**Task:** A developer writes a loop processing a `FOR UPDATE` cursor over a very large table, and calls `COMMIT` inside the loop after every 500th row, intending to avoid holding locks for too long. What's wrong with this, and what should be done instead?

**Answer:** This is a genuinely dangerous pattern, not just a style concern. Once `COMMIT` executes, **all** locks held by the transaction are released immediately — including the locks on every row in the cursor's active set that **hasn't been fetched yet**. From that moment, any other session is free to modify those still-unprocessed rows, which completely defeats the purpose of having declared the cursor `FOR UPDATE` in the first place — the very guarantee it was providing (nothing changes underneath you while you decide) is broken for the remainder of the loop. On top of that, continuing to `FETCH` from a cursor after an intervening `COMMIT` is documented to risk `ORA-01002: fetch out of sequence` in various circumstances, since Oracle's read-consistency model for an open cursor is tied to the transaction it was opened in.

**What to do instead:** don't keep a single `FOR UPDATE` cursor open across multiple commits. For very large batches that genuinely need periodic commits (to avoid holding an enormous number of locks for too long), the standard approach is **explicit batching**: process a bounded chunk of rows (e.g., using a status flag, or a bounded row range), `COMMIT` that chunk, then open a **fresh** cursor for the next chunk of still-unprocessed rows, repeating until none remain. This means closing and reopening the cursor between commits rather than holding one cursor open across them — a more advanced batch-processing pattern beyond this syllabus's cursor topics in its full detail, but the core rule belongs squarely here: **never commit while a `FOR UPDATE` cursor you still intend to fetch from remains open.**

---

## Realistic Scenario

### Exercise 7
**Business Requirement:** *"The order fulfillment team needs a script that reviews every pending order, checks current stock for each item, and either marks the order as 'READY_TO_SHIP' if all items are in stock, or 'BACKORDERED' otherwise. Since multiple fulfillment staff may run overlapping checks or place manual holds on orders during business hours, the update must guarantee no other process changes an order between when this script reads it and when it writes the new status."*

**What Is Being Asked:**
For every pending order, check item availability and write back a new status — with an explicit, stated guarantee against concurrent modification during that read-decide-write window.

**Key Clues:**
- "for every pending order... checks... and either marks..." → nested-style processing (order → its items), same shape as earlier master-detail patterns, but here condensed into a per-order aggregate check rather than a full nested loop.
- "must guarantee no other process changes an order between when this script reads it and when it writes" → this is a direct, textbook description of the lost-update problem `FOR UPDATE` exists to prevent. This is the strongest possible signal for this topic.

**What Data Is Required:** `order_id`, `status` from `orders`; item requirements from `order_items` joined against current stock in `inventory`.

**Step-by-Step Approach:**
1. Declare a cursor over pending orders with `FOR UPDATE`, locking every pending order row up front.
2. For each order, check whether any of its items are short of stock (a per-order aggregate check).
3. Use `WHERE CURRENT OF` to write the new status directly onto the row already locked and fetched.
4. `COMMIT` once all decisions are made, releasing every lock at once.

**Solution:**
```sql
DECLARE
   CURSOR c_pending_orders IS
      SELECT order_id FROM orders WHERE status = 'PENDING' FOR UPDATE;

   v_out_of_stock_count NUMBER;
BEGIN
   FOR order_rec IN c_pending_orders LOOP

      SELECT COUNT(*)
      INTO v_out_of_stock_count
      FROM order_items oi
      JOIN inventory inv ON inv.item_id = oi.item_id
      WHERE oi.order_id = order_rec.order_id
        AND inv.quantity_on_hand < oi.quantity_requested;

      IF v_out_of_stock_count = 0 THEN
         UPDATE orders SET status = 'READY_TO_SHIP' WHERE CURRENT OF c_pending_orders;
      ELSE
         UPDATE orders SET status = 'BACKORDERED' WHERE CURRENT OF c_pending_orders;
      END IF;

   END LOOP;

   COMMIT;
END;
/
```

**Explanation:** `FOR UPDATE` on the outer `orders` cursor locks every pending order the instant the cursor opens, satisfying the "guarantee no other process changes an order" requirement directly — any other session trying to update or delete one of these specific order rows would now block (or fail/skip, depending on how *they* opened their own cursor) until this transaction commits. The stock check runs as a small, targeted aggregate query per order rather than a full nested loop over items, since the business logic only needs a yes/no answer ("is anything short?"), not to process each item individually. `WHERE CURRENT OF` writes the decision straight onto the already-locked, already-fetched row.

**Alternative Approach — and a genuinely important comparison:** the fulfillment team might instead want the script to simply **skip** any order a staff member currently has under manual review, rather than waiting behind them. A natural but **incorrect** first instinct would be to add `NOWAIT`:
```sql
CURSOR c_pending_orders IS
   SELECT order_id FROM orders WHERE status = 'PENDING' FOR UPDATE NOWAIT;
```
This does **not** achieve "skip the ones under review, process the rest." Because `NOWAIT` failure applies to the **entire `OPEN`**, if even a single pending order happens to be locked by a staff member's manual hold, the *whole cursor fails to open* — none of the other, perfectly available pending orders get processed either, in that run. The correct tool for "skip what's locked, keep processing everything else" is `SKIP LOCKED`:
```sql
CURSOR c_pending_orders IS
   SELECT order_id FROM orders WHERE status = 'PENDING' FOR UPDATE SKIP LOCKED;
```
With this version, any order currently held by manual staff review is simply excluded from this run's active set, and every other pending order is processed normally — exactly matching the "don't wait behind staff, but still get through everything else" intent, in a way `NOWAIT` fundamentally cannot provide.

**Common Mistakes to Watch For:**
- Reaching for `NOWAIT` when `SKIP LOCKED` was actually needed — both relate to "handling already-locked rows," but they behave in opposite ways (fail everything vs. quietly exclude just the locked rows), and it's an easy mix-up.
- Forgetting the final `COMMIT` — the locks acquired on every pending order would persist for the rest of the session, potentially blocking fulfillment staff's own later actions on those exact orders, a confusing and hard-to-diagnose issue to track down after the fact.
- Running the per-order stock check as a row-by-row query inside the loop rather than considering a bulk pre-check for very large order volumes — a reasonable performance discussion for high-volume systems, though acceptable here given that locking is already being done order-by-order regardless.

---

**End of Topic 5.** Next file: `06-ref-cursor.md`.