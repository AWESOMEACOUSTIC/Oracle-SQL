# Oracle PL/SQL — Locking, Transactions & Concurrency
### Exam Study Guide

Covers: Lock Types · Deadlock Avoidance · Autonomous Transactions · Serializable vs. Read Committed · Integrated Case Study

---

## 1. Lock Types

### What it is
A **lock** is a mechanism that prevents destructive interference between concurrent transactions accessing the same resource. Oracle locks fall into two broad categories relevant to this topic: **row-level (DML) locks (TX)** and **table-level (DML) locks (TM)** — Oracle always locks at the **row** level for data changes (there is no row-lock escalation to a full table lock the way some other RDBMS platforms behave), and automatically acquires a complementary, less-restrictive table-level lock alongside it to protect against conflicting DDL/structural changes.

### Row-level locks (TX)
Acquired automatically whenever a transaction modifies (or explicitly locks) a row — `INSERT`, `UPDATE`, `DELETE`, `MERGE`, or `SELECT ... FOR UPDATE`. Exactly one transaction can hold a TX lock on a given row at a time; a second transaction attempting to modify the same row **waits** until the first commits or rolls back.

### Table-level locks (TM) — the five modes
| Mode | Name | Acquired automatically by | Restrictiveness |
|---|---|---|---|
| **RS** | Row Share | `SELECT ... FOR UPDATE`, `LOCK TABLE ... IN ROW SHARE MODE` | Least restrictive |
| **RX** | Row Exclusive | `INSERT`, `UPDATE`, `DELETE`, `MERGE` | |
| **S** | Share | `LOCK TABLE ... IN SHARE MODE` | |
| **SRX** | Share Row Exclusive | `LOCK TABLE ... IN SHARE ROW EXCLUSIVE MODE` | |
| **X** | Exclusive | `LOCK TABLE ... IN EXCLUSIVE MODE` | Most restrictive |

The core intuition: **RS/RX locks are "intent" locks** — they signal "I'm about to touch some rows," and are compatible with most other RS/RX activity from other sessions (this is exactly why ordinary row-level DML from many concurrent sessions scales well in Oracle: each transaction only ever really contends over the *specific rows* it touches, not the whole table). **S/SRX/X are much stronger**, typically used for `LOCK TABLE` statements taken deliberately before bulk operations, structural staging, or integrity-critical batch work where you must guarantee no conflicting concurrent DML.

### Manual locking
```sql
LOCK TABLE inventory IN EXCLUSIVE MODE WAIT 5;   -- wait up to 5 seconds, else ORA-00054
LOCK TABLE inventory IN ROW SHARE MODE NOWAIT;   -- fail immediately if unavailable
```
`NOWAIT` fails immediately with `ORA-00054` if the lock can't be acquired; `WAIT n` blocks for up to `n` seconds before failing; with neither, the default behavior is to **wait indefinitely**.

### `SELECT ... FOR UPDATE` — explicit row locking without a table lock
```sql
SELECT quantity INTO v_qty FROM inventory WHERE item_id = 501 FOR UPDATE;
-- Acquires: a TX row lock on that specific row, and an RS table lock (the least restrictive)
```
`FOR UPDATE [OF column_list] [NOWAIT | WAIT n | SKIP LOCKED]` is the standard way to explicitly reserve rows you intend to change later in the same transaction, preventing another session from modifying (or, depending on mode, even locking) them in the meantime. **`SKIP LOCKED`** (12c+) lets a session simply skip over rows currently locked by another transaction instead of waiting or failing — useful for building a concurrent, contention-free "work queue" pattern where multiple sessions each grab a different available row.

### Important points to remember
- Oracle's row-level locking is always **exactly at row granularity** — there is no lock escalation from row locks to a full table lock as data volume grows, which is a deliberate design difference from some other database systems and a common comparative exam point.
- A table lock's mode determines what **other** table locks are permitted concurrently on the same table — e.g., many sessions can each hold RX simultaneously (ordinary concurrent DML on different rows), but an X lock permits **no** other lock of any kind concurrently.
- `SELECT` (with no `FOR UPDATE`) takes **no lock at all** — Oracle's multiversion read consistency (Section 4) means readers never block writers and writers never block readers in Oracle, a frequently tested differentiator from lock-based-read RDBMS platforms.
- `SKIP LOCKED` is specifically valuable for implementing **queue-processing patterns** (e.g., multiple worker sessions each picking up different orders/rows to process) without contention or blocking.

### Exercise Questions

**Q1.** Two sessions each run `UPDATE employees SET salary = salary * 1.05 WHERE employee_id = 100;` at the same time, targeting the **same row**. What happens?
> **A:** The first session to reach the statement acquires a TX row lock on that specific row (plus an RX table lock). The second session's `UPDATE` **blocks**, waiting for the first transaction to `COMMIT` or `ROLLBACK` and release the row lock — it does not fail immediately (unless a `NOWAIT`/timeout mechanism was explicitly used, which plain `UPDATE` does not have).

**Q2.** Why does Oracle acquire only a lightweight **RX** table lock (rather than a full table-level exclusive lock) when a session runs a normal `UPDATE` on a handful of rows?
> **A:** Oracle's row-level locking model only needs to lock the **specific rows actually being modified** — the accompanying RX table lock exists purely to signal "this table has pending DML" (chiefly to block conflicting DDL, since you can't `ALTER`/`DROP` a table while an uncommitted transaction holds any table lock on it) and to participate in lock-mode compatibility checks with other sessions' table-level lock requests. It deliberately does **not** need to be a full exclusive table lock, because doing so would needlessly block every other session's unrelated row-level work on the same table — that would defeat Oracle's fine-grained concurrency model.

**Q3.** In a job-queue-style table where many worker sessions each want to grab and process one row without waiting on each other, which clause is designed exactly for this, and why does it help?
> **A:** **`SKIP LOCKED`**, used with `SELECT ... FOR UPDATE SKIP LOCKED`. Instead of a worker session waiting (or failing) when it encounters a row another worker has already locked for processing, it simply **skips that row** and moves on to the next available, unlocked one — letting many workers pull different rows concurrently with zero blocking or contention between them.

---

## 2. Deadlock Avoidance

### What a deadlock is
A **deadlock** occurs when two (or more) transactions are each waiting for a lock held by the other, forming a **cycle of mutual waiting** that can never resolve on its own. Classic example:
```
Session A: locks row 1, then tries to lock row 2 (held by B) -> waits
Session B: locks row 2, then tries to lock row 1 (held by A) -> waits
```
Neither session can ever proceed — without intervention, both would wait forever.

### Oracle's automatic deadlock detection
Unlike many failure conditions, Oracle **automatically detects** deadlocks (it periodically checks for wait cycles) and resolves them itself: it picks **one of the two involved sessions**, rolls back the **single statement** that caused the deadlock in that session (not necessarily the whole transaction), and returns `ORA-00060: deadlock detected while waiting for resource` to that session's application code. The other session's blocked statement then proceeds normally.

### Avoidance — this is on the *application design* side, since Oracle only detects/resolves, it doesn't prevent
| Technique | How it helps |
|---|---|
| **Consistent locking order** | If every transaction that needs to touch both Table/Row A and Table/Row B always locks them in the **same order** (e.g., always A before B), the circular-wait condition that causes deadlocks becomes structurally impossible. |
| **Keep transactions short** | The shorter the window between acquiring a lock and committing/releasing it, the smaller the chance of overlapping with another transaction's conflicting lock request. |
| **Use `NOWAIT`/`WAIT n` and handle the resulting exception with a retry strategy** | Fail fast instead of piling up long, blocking wait chains that increase deadlock likelihood under load. |
| **Lock rows explicitly and predictably with `SELECT ... FOR UPDATE`** at the **start** of a transaction, for all rows it will need, rather than acquiring locks piecemeal as the transaction progresses | Reduces the chance of a transaction discovering mid-way that it needs another lock already held by a session that is, in turn, waiting on it |
| **Minimize unnecessary locking scope** | Only lock the specific rows genuinely being modified, rather than broader locks (e.g., unnecessary `LOCK TABLE ... EXCLUSIVE`) that increase the surface area for conflicts |

### Handling `ORA-00060` in PL/SQL
```sql
BEGIN
  UPDATE accounts SET balance = balance - 100 WHERE acct_id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE acct_id = 2;
  COMMIT;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE = -60 THEN
      ROLLBACK;
      -- application-level retry logic here (often with a short backoff delay)
    ELSE
      RAISE;
    END IF;
END;
/
```

### Important points to remember
- Oracle **detects and resolves** deadlocks automatically — it does **not prevent** them from occurring in the first place; prevention is entirely an **application design responsibility** (this exact distinction — detection/resolution vs. prevention — is a very common exam question).
- The deadlock victim's **entire transaction is not necessarily lost** — only the specific statement that triggered the deadlock is rolled back; the application typically still needs to `ROLLBACK` the rest of the transaction explicitly (or handle it per its own logic) since the transaction is left in an indeterminate partial state.
- The single most effective **structural** deadlock-avoidance technique is enforcing a **consistent order of resource acquisition** across all transactions/procedures that might touch the same set of resources.
- Deadlocks in Oracle typically show up in trace files/alert logs (`ORA-00060`) with full details of the blocking chain, session IDs, and the exact SQL statements involved — useful for diagnosing which piece of application code violated the consistent-ordering discipline.

### Exercise Questions

**Q1.** Does Oracle prevent deadlocks from occurring, or only detect and resolve them after they happen?
> **A:** Oracle only **detects and resolves** deadlocks after they occur — it periodically checks for cyclic lock-wait dependencies, and once found, rolls back the statement of one of the involved sessions (raising `ORA-00060` to that session) to break the cycle. **Preventing** deadlocks from arising in the first place is the responsibility of application design (e.g., consistent lock ordering) — Oracle provides no built-in mechanism to stop the circular-wait condition from forming.

**Q2.** Two batch procedures both update the `ORDERS` and `INVENTORY` tables, but Procedure X always updates `ORDERS` first then `INVENTORY`, while Procedure Y always updates `INVENTORY` first then `ORDERS`. Why is this arrangement a classic setup for deadlocks, and what is the fix?
> **A:** If Procedure X locks a row in `ORDERS` while Procedure Y simultaneously locks a row in `INVENTORY`, then X tries to lock the same `INVENTORY` row (blocked by Y) while Y tries to lock the same `ORDERS` row (blocked by X) — a circular wait, i.e., a deadlock. The fix is to enforce a **consistent locking order** across both procedures: rewrite Procedure Y to also update `ORDERS` before `INVENTORY` (matching Procedure X's order), which structurally eliminates the possibility of this particular circular-wait pattern.

**Q3.** After catching `ORA-00060` in an exception handler, is it sufficient to simply retry the same statement immediately without a `ROLLBACK`?
> **A:** No. When Oracle resolves a deadlock, it rolls back only the **one statement** that caused the deadlock in the "victim" session — the transaction is left in a partially completed, indeterminate state (earlier statements in that transaction may still be uncommitted and holding other locks). The correct handling is to explicitly `ROLLBACK` the transaction (undoing everything done so far in it) before deciding whether/how to retry the overall unit of work, rather than assuming only the failed statement needs to be reissued.

---

## 3. Autonomous Transactions

### What it is
An **autonomous transaction** is a PL/SQL block marked with `PRAGMA AUTONOMOUS_TRANSACTION` that executes as an **independent transaction**, nested inside — but logically separate from — the transaction that called it. It gets its own private transaction context: its own commit/rollback, its own read-consistent view, completely separate from the calling ("main") transaction.

```sql
CREATE OR REPLACE PROCEDURE log_error (p_message IN VARCHAR2) IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO error_log (log_time, message) VALUES (SYSTIMESTAMP, p_message);
  COMMIT;    -- REQUIRED before returning -- see below
END;
/
```

### The defining behavior
- The autonomous transaction **cannot see the calling transaction's own uncommitted changes** (it's a genuinely separate transaction, subject to the same read-consistency rules as any other independent session's transaction would be relative to the caller's uncommitted work).
- It **must explicitly `COMMIT` or `ROLLBACK`** before control returns to the calling block — if it doesn't, Oracle raises **`ORA-06519`: active autonomous transaction detected and rolled back before returning**, automatically rolling back whatever the autonomous block did.
- Once the autonomous transaction commits, **that commit is permanent and independent** — even if the **outer, calling transaction later rolls back**, the autonomous transaction's committed work **survives** untouched.

### The classic use case — audit/error logging that must survive a rollback
```sql
CREATE OR REPLACE PROCEDURE process_order (p_order_id NUMBER) IS
BEGIN
  UPDATE orders SET status = 'PROCESSING' WHERE order_id = p_order_id;
  -- ... more business logic that might fail ...
  RAISE_APPLICATION_ERROR(-20001, 'Inventory check failed');
EXCEPTION
  WHEN OTHERS THEN
    log_error('Order ' || p_order_id || ' failed: ' || SQLERRM);  -- autonomous -- survives!
    RAISE;   -- re-raise so the caller still sees (and can roll back) the failure
END;
/
```
Even though `process_order`'s own `UPDATE` gets rolled back (by the caller, or by an unhandled exception propagating up), the **log entry written by the autonomous `log_error` call is already committed and permanent** — exactly the point of using an autonomous transaction here: you want a durable record of what happened, *especially* when the main transaction fails and rolls back.

### Important points to remember
- **Common uses:** audit trails, error/event logging, "fire and forget" side-effects that must persist independent of the outcome of the main business transaction, and some retry/queuing patterns.
- An autonomous transaction is **not** a way to peek at or influence the calling transaction's *uncommitted* data — it is fully isolated from it, exactly as if it were a completely separate session's transaction (aside from sharing the same session/process).
- Forgetting the mandatory `COMMIT`/`ROLLBACK` before the autonomous block ends is a very common bug, producing `ORA-06519` — always ensure every code path (including exception handlers *inside* the autonomous procedure itself) ends with an explicit commit or rollback.
- Overuse of autonomous transactions to "work around" a locking or transactional design problem is generally considered a **design smell** — they're a legitimate, narrow tool (logging, auditing) rather than a general mechanism to bypass normal transaction semantics; using them broadly can make application transactional behavior much harder to reason about.

### Exercise Questions

**Q1.** A procedure inserts a row, then calls an autonomous-transaction logging procedure, and afterward the outer procedure's caller issues a `ROLLBACK`. Does the log entry get rolled back too?
> **A:** No. Because the logging procedure ran as an **autonomous transaction**, its `COMMIT` created a permanent, independent transaction boundary, completely separate from the outer transaction. The outer `ROLLBACK` only undoes the outer transaction's own uncommitted work (the row insert) — it has no effect on the already-committed autonomous transaction.

**Q2.** Inside an autonomous transaction procedure, what happens if the block's logic completes without ever issuing a `COMMIT` or `ROLLBACK`?
> **A:** Oracle raises `ORA-06519: active autonomous transaction detected and rolled back before returning`, and automatically rolls back whatever the autonomous block had done. Every autonomous transaction **must** explicitly commit or roll back before control returns to its caller — there is no implicit commit, and simply falling off the end of the block (or letting an unhandled exception propagate without cleanup) triggers this error.

**Q3.** Why can't an autonomous transaction see uncommitted changes made earlier in the same calling transaction (e.g., an `UPDATE` the outer procedure ran just before calling the autonomous procedure)?
> **A:** An autonomous transaction is a genuinely **separate, independent transaction** — from a read-consistency standpoint, it behaves exactly as if it were started by an entirely different session, and per Oracle's standard multiversion read consistency, one transaction never sees another (uncommitted) transaction's changes. The fact that both transactions happen to be running within the same PL/SQL call stack/session does not create any special visibility between them.

---

## 4. Serializable vs. Read Committed

### The two isolation levels Oracle supports
| | **Read Committed** (Oracle's default) | **Serializable** |
|---|---|---|
| Consistency granularity | **Statement-level** — each individual statement sees data as it existed (committed) at the moment **that statement** began | **Transaction-level** — every statement within the transaction sees data as it existed at the moment the **transaction** began |
| Non-repeatable reads | **Possible** — re-running the same query later in the same transaction can see different data if another transaction committed changes in between | **Not possible** — the transaction's view is frozen from its own start point |
| Phantom reads | **Possible** | **Not possible** |
| Writers that conflict with concurrent commits | Simply wait for the row lock and then proceed against the now-current version | If it detects the row was changed by a **different transaction that committed after this transaction started**, raises **`ORA-08177: can't serialize access for this transaction`** — the application must catch this and retry the whole transaction |
| Setting it | Default (or `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` / `ALTER SESSION SET ISOLATION_LEVEL = READ COMMITTED;`) | `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;` / `ALTER SESSION SET ISOLATION_LEVEL = SERIALIZABLE;` |

### How Oracle achieves this — multiversion concurrency control (MVCC), not lock-based reads
Oracle's approach to isolation is fundamentally different from lock-based-read systems: **readers never block writers, and writers never block readers.** Consistency is achieved via **undo segments** — when a query needs to see data "as of" an earlier point (statement start for read committed; transaction start for serializable), Oracle reconstructs that earlier version from undo information rather than blocking anyone.

### Practical example — the difference in action
```sql
-- READ COMMITTED (default): each statement gets its own fresh committed snapshot
BEGIN
  SELECT balance INTO v_bal FROM accounts WHERE acct_id = 1;  -- sees value at this moment
  -- ...(another session commits a change to this row in the meantime)...
  SELECT balance INTO v_bal FROM accounts WHERE acct_id = 1;  -- CAN see the new value now
END;

-- SERIALIZABLE: the whole transaction is pinned to its start-time snapshot
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN
  SELECT balance INTO v_bal FROM accounts WHERE acct_id = 1;  -- snapshot as of transaction start
  -- ...(another session commits a change to this row in the meantime)...
  SELECT balance INTO v_bal FROM accounts WHERE acct_id = 1;  -- SAME value as before -- frozen
  UPDATE accounts SET balance = v_bal - 50 WHERE acct_id = 1;
  -- If the OTHER session's committed change touched THIS row, this UPDATE
  -- raises ORA-08177 instead of silently overwriting based on stale data.
END;
```

### Important points to remember
- **Read Committed is Oracle's default**, and is appropriate for the vast majority of applications; **Serializable** is a deliberate, narrower choice made when a transaction genuinely needs a fully consistent, unchanging view of the data for its *entire* duration.
- `ORA-08177` is raised **only when the serializable transaction itself tries to modify a row that another transaction has changed and committed since the serializable transaction began** — it is not raised merely for reading changed data; reads under serializable simply keep returning the transaction's original snapshot values.
- Applications using `SERIALIZABLE` **must implement retry logic** to catch `ORA-08177` and re-attempt the whole transaction from the start — this is an expected, designed-for part of using this isolation level, not an unusual error condition to guard against defensively.
- Serializable transactions typically need **more undo/rollback segment capacity** and are best suited to **shorter transactions** — long-running serializable transactions in a busy system are much more likely to encounter `ORA-08177`, since the odds of *some* concurrent transaction committing a conflicting change during a long window rise accordingly. This is exactly why Oracle's own guidance is essentially "use Read Committed unless you have a specific, well-understood need for Serializable."
- Oracle also documents a third mode, **`READ ONLY`** (not part of the SQL92 standard), giving a transaction a fixed, unchanging snapshot for its duration but disallowing any modifications at all within it.
- Both isolation levels use the **same row-level locking mechanics** underneath (Section 1) for actual write conflicts — the difference between them is entirely about **the consistency of what a query is allowed to *see*, not how writes are locked**.

### Exercise Questions

**Q1.** A transaction running under Read Committed executes the same `SELECT` twice, several seconds apart, and gets two different results because another session committed a change in between. Is this a bug?
> **A:** No — this is the **expected, defined behavior of Read Committed isolation**: consistency is guaranteed only at the **statement** level, so each individual `SELECT` is entitled to see whatever was committed as of that statement's own start time, and different statements within the same transaction can legitimately see different snapshots of the data if commits happen in between them. This is precisely the "non-repeatable read" phenomenon that Read Committed permits (and that Serializable is specifically designed to prevent).

**Q2.** A transaction is running under `SERIALIZABLE`. It reads a row's balance, performs a calculation, and then tries to `UPDATE` that same row — but another session had already updated and committed a change to that exact row after this transaction started. What happens, and why?
> **A:** The `UPDATE` raises **`ORA-08177: can't serialize access for this transaction`**. Because a serializable transaction is defined to behave as though it operates against a single, frozen snapshot taken at its own start, Oracle cannot silently let it write a change that would be based on now-stale data without violating that guarantee — rather than allowing an inconsistent write, it refuses the operation and forces the application to detect this and retry the entire transaction against current data.

**Q3.** Why does Oracle's own guidance generally favor Read Committed over Serializable for most applications, even though Serializable offers stronger consistency guarantees?
> **A:** Serializable's stronger guarantee comes at a real practical cost: transactions must retry (potentially repeatedly, under heavy concurrent write load) whenever `ORA-08177` occurs, which adds application complexity and can hurt throughput; serializable transactions also typically demand more undo capacity and are far better suited to short-lived work, since longer transactions face a proportionally higher chance of a conflicting concurrent commit occurring during their (frozen) window. For the majority of applications, Read Committed's statement-level consistency is sufficient and avoids these costs entirely — Serializable is reserved for the specific, narrower set of cases where a transaction genuinely requires a single consistent view across its whole duration (e.g., certain financial reconciliation or reporting scenarios).

---

## 5. Case Study — Putting It All Together

### Scenario
An e-commerce platform has an `order_processing` procedure that, for a given order, needs to: (1) check and decrement available stock in an `inventory` table, (2) insert a row into an `orders` table, and (3) log every attempt (success or failure) to an `audit_log` table for compliance — the audit entry must be preserved **even if the order itself fails and rolls back**. Multiple customers can attempt to order the *same* popular item simultaneously.

```sql
CREATE OR REPLACE PROCEDURE log_attempt (p_order_id NUMBER, p_message VARCHAR2) IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO audit_log (order_id, log_time, message)
  VALUES (p_order_id, SYSTIMESTAMP, p_message);
  COMMIT;
END;
/

CREATE OR REPLACE PROCEDURE process_order (p_order_id NUMBER, p_item_id NUMBER, p_qty NUMBER) IS
  v_available NUMBER;
BEGIN
  -- Explicitly lock the specific inventory row we're about to modify
  SELECT quantity INTO v_available
  FROM   inventory
  WHERE  item_id = p_item_id
  FOR UPDATE;                          -- TX row lock + RS table lock

  IF v_available < p_qty THEN
    log_attempt(p_order_id, 'FAILED -- insufficient stock');
    RAISE_APPLICATION_ERROR(-20001, 'Insufficient stock for item ' || p_item_id);
  END IF;

  UPDATE inventory SET quantity = quantity - p_qty WHERE item_id = p_item_id;
  INSERT INTO orders (order_id, item_id, qty, status) VALUES (p_order_id, p_item_id, p_qty, 'CONFIRMED');

  log_attempt(p_order_id, 'SUCCESS');
  COMMIT;
EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;                          -- undo the inventory/orders changes for THIS transaction
    RAISE;
END;
/
```

Two customers, Session A and Session B, both attempt to buy the **last unit** of the same popular item (`item_id = 501`, `quantity = 1`) at almost the same moment.

### Case Study Questions

**Q1.** Why does `process_order` use `SELECT ... FOR UPDATE` rather than a plain `SELECT` before checking `v_available`?
> **A:** A plain `SELECT` takes **no lock at all**, so both Session A and Session B could read `quantity = 1` at the same instant, both conclude stock is available, and both proceed to decrement it — resulting in an oversold item (a **lost-update** race condition). `SELECT ... FOR UPDATE` acquires a **TX row lock** on that specific inventory row immediately: whichever session gets there first holds the lock, and the second session's `FOR UPDATE` **blocks** until the first transaction commits or rolls back — at which point it re-reads the row and correctly sees the updated (now zero) quantity, preventing the overselling scenario entirely.

**Q2.** If Session A's transaction ultimately fails and issues `ROLLBACK` (as in the `EXCEPTION` handler), does Session B's earlier-logged `'FAILED -- insufficient stock'` audit entry (if it also failed) get rolled back along with it?
> **A:** No. `log_attempt` is an **autonomous transaction** — each call to it commits independently of whatever happens afterward in the calling `process_order` transaction. Even though `process_order`'s own `ROLLBACK` undoes its `UPDATE`/`INSERT` on `inventory`/`orders`, the audit log entry was already permanently committed the moment `log_attempt` ran, exactly as intended — the compliance record survives regardless of the outcome of the order itself.

**Q3.** Suppose, instead of `FOR UPDATE`, the design used `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;` at the start of `process_order`, with a plain `SELECT` (no `FOR UPDATE`) to check quantity, followed later by the `UPDATE`. Would this correctly prevent overselling? What would the failure mode look like instead?
> **A:** It would still prevent an inconsistent oversold outcome, but through a **different mechanism and a different failure mode**: both sessions could read `quantity = 1` under their own respective frozen snapshots (serializable reads don't block), and both would proceed to attempt the `UPDATE`. Whichever session **commits first** succeeds normally. The **second** session's `UPDATE`, upon detecting that the row was changed by a transaction that committed after its own transaction began, would fail with **`ORA-08177: can't serialize access for this transaction`** rather than silently overselling — but this requires the application to have **explicit retry logic** to catch `ORA-08177`, re-read the (now-current) quantity, and re-evaluate whether stock is still available, essentially re-running the whole transaction. This is a valid alternative design, but it trades the simpler "just block and proceed" behavior of `FOR UPDATE` for the added complexity of building correct retry handling around `ORA-08177`.

**Q4.** Why is a **consistent locking order** less of a concern in this specific scenario compared to, say, a procedure that updates both `inventory` and a separate `customer_credit_limit` table in varying order across different code paths?
> **A:** `process_order`, as written, only ever locks **one** row of `inventory` before doing its other work — there's no scenario within this procedure where it acquires a lock on one resource, then tries to acquire a lock on a second resource that another instance of the *same* procedure might have already locked in the opposite order (since every invocation follows the identical single-row-then-proceed sequence). Deadlock risk from inconsistent ordering specifically arises when **two or more distinct resources** can be locked in **different orders** by different transactions/code paths — a scenario like updating both `inventory` and `customer_credit_limit` would only become deadlock-prone if some code path locked them in the reverse order from another code path, which is exactly the kind of design flaw Section 2's "consistent locking order" guidance is meant to prevent.

**Q5.** A code reviewer suggests changing the `FOR UPDATE` to `FOR UPDATE NOWAIT` so that a customer trying to buy an item currently being purchased by someone else gets an immediate error instead of waiting. What is the trade-off of this change?
> **A:** With `NOWAIT`, if Session B's `FOR UPDATE` finds the row already locked by Session A, it fails **immediately** with `ORA-00054` rather than waiting for Session A to finish — this gives faster feedback to the user (avoiding a potentially long wait for a popular item under heavy contention) and requires the application to handle that specific error (e.g., "someone else is currently purchasing this item, please try again"). The trade-off is that a customer whose request arrives just moments after another's now needs to **retry themselves** (or the application needs automatic retry logic) rather than simply and transparently waiting the (usually very short) time for the first transaction to complete — a legitimate design choice depending on whether the application prioritizes immediate responsiveness or transparent, automatic serialization of concurrent requests for the same item.

---

## Quick Cross-Topic Summary Table

| Topic | Key Mechanism | Who's Responsible |
|---|---|---|
| Lock Types | TX (row), TM (table: RS/RX/S/SRX/X), `FOR UPDATE`, `SKIP LOCKED` | Oracle acquires automatically; manual via `LOCK TABLE` |
| Deadlock Avoidance | `ORA-00060` auto-detected & resolved by Oracle | **Application design** (consistent lock order, short transactions) |
| Autonomous Transactions | `PRAGMA AUTONOMOUS_TRANSACTION`; independent commit/rollback | Developer — must always commit/rollback before returning |
| Serializable vs. Read Committed | Statement-level (default) vs. transaction-level snapshot; `ORA-08177` | DBA/developer chooses per transaction; app must handle retries under Serializable |