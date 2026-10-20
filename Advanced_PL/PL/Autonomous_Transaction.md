# Autonomous Transactions (Oracle PL/SQL)

## What it is

An autonomous transaction is a PL/SQL block marked with `PRAGMA AUTONOMOUS_TRANSACTION` that runs as an **independent transaction**, nested inside (but logically separate from) the transaction that called it. It gets its own private transaction context: its own commit/rollback and its own read-consistent view, completely separate from the calling ("main") transaction.

**In one line:** it's a separate transaction that runs in the middle of your current one. It commits or rolls back on its own, and nothing the main transaction does afterward can undo it.

```sql
CREATE OR REPLACE PROCEDURE log_error (p_message IN VARCHAR2) IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO error_log (log_time, message) VALUES (SYSTIMESTAMP, p_message);
  COMMIT;    -- REQUIRED before returning
END;
/
```

### Analogy

You're filling out a long form at a bank counter. That's your main transaction, and nothing is final until you sign at the end. Halfway through, you step over to a different counter and drop a complaint slip in a box. That slip is the autonomous transaction.

- If you later tear up the main form (ROLLBACK), the complaint slip is still in the box.
- The person at the complaint counter can't see what you wrote on your unsigned form. To them it doesn't exist yet.
- You must actually drop the slip in the box (COMMIT) or throw it away (ROLLBACK) before walking back. Walking back with it still in your hand is the ORA-06519 error.

---

## The defining behavior

- **Isolated from the caller's uncommitted changes.** It's a genuinely separate transaction, subject to the same read-consistency rules as any other session's transaction.
- **Must explicitly COMMIT or ROLLBACK before returning.** Otherwise Oracle raises `ORA-06519: active autonomous transaction detected and rolled back` and rolls back whatever the autonomous block did.
- **Its commit is permanent and independent.** Even if the outer transaction later rolls back, the autonomous transaction's committed work survives.

### The three rules to remember

1. **Isolated in both directions.** The autonomous transaction can't see the main transaction's uncommitted changes, and the main transaction's rollback can't touch the autonomous commit. Oracle treats them as if they came from two different sessions, even though they share one session.
2. **It must end explicitly.** Every code path inside the autonomous block, including its own exception handlers, must hit COMMIT or ROLLBACK.
3. **Its commit doesn't commit the parent.** The COMMIT inside the autonomous block only commits the autonomous transaction. The caller's work is still pending afterward.

---

## The classic use case: logging that must survive a rollback

```sql
CREATE OR REPLACE PROCEDURE process_order (p_order_id NUMBER) IS
BEGIN
  UPDATE orders SET status = 'PROCESSING' WHERE order_id = p_order_id;
  -- ... more business logic that might fail ...
  RAISE_APPLICATION_ERROR(-20001, 'Inventory check failed');
EXCEPTION
  WHEN OTHERS THEN
    log_error('Order ' || p_order_id || ' failed: ' || SQLERRM);  -- autonomous: survives!
    RAISE;   -- re-raise so the caller still sees (and can roll back) the failure
END;
/
```

Even though `process_order`'s UPDATE gets rolled back, the log entry written by `log_error` is already committed. That's the point: you want a durable record of what happened, especially when the main transaction fails.

### Timeline

```
Main transaction (T1)                 Autonomous transaction (T2)
─────────────────────                 ───────────────────────────
UPDATE orders SET status='PROCESSING'   (uncommitted, only T1 sees it)
error raised → jump to EXCEPTION
call log_error(...)  ──────────────►  T1 is PAUSED, T2 begins
                                      INSERT INTO error_log ...
                                      COMMIT   ← permanent now
                     ◄──────────────  T2 ends, T1 resumes
RAISE
caller does ROLLBACK
  → UPDATE on orders is undone
  → error_log row is STILL THERE ✅
```

---

## Important points to remember

- **Common uses:** audit trails, error/event logging, "fire and forget" side effects that must persist regardless of the main transaction's outcome, and some retry/queuing patterns.
- **Not a way to peek at the caller's data.** It's fully isolated from the caller's uncommitted work, exactly as if it were another session.
- **Forgetting COMMIT/ROLLBACK is a very common bug** (ORA-06519). Make sure every code path, including exception handlers, ends with one.
- **Self-deadlock risk.** Because the autonomous transaction behaves like a separate session, it waits on the caller's locks. If it tries to modify a row the caller already locked, you get `ORA-00060: deadlock detected`.
- **Triggers:** a normal trigger can't COMMIT (ORA-04092). An autonomous trigger can, which is how audit triggers log even rejected changes.
- **Functions called from SQL:** a function used in a SELECT normally can't do DML. Making it autonomous allows it, though that's usually a design smell.
- **Design smell when overused.** Using autonomous transactions to work around locking or transaction design problems makes behavior hard to reason about. Treat them as a narrow tool for logging and auditing.

---

## Scenario-Based Examples

All scenarios use these tables:

```sql
CREATE TABLE orders    (order_id NUMBER PRIMARY KEY, status VARCHAR2(20));
CREATE TABLE error_log (log_time TIMESTAMP, message VARCHAR2(200));

INSERT INTO orders VALUES (101, 'NEW');
COMMIT;
```

### Scenario 1: The log that survives

```sql
CREATE OR REPLACE PROCEDURE log_error (p_msg VARCHAR2) IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO error_log VALUES (SYSTIMESTAMP, p_msg);
  COMMIT;
END;
/

BEGIN
  UPDATE orders SET status = 'SHIPPED' WHERE order_id = 101;
  log_error('Tried to ship order 101');
  ROLLBACK;
END;
/

SELECT status FROM orders WHERE order_id = 101;
SELECT COUNT(*) FROM error_log;
```

**Question:** What do the two SELECTs return?

> **Answer:** `status = 'NEW'` and `COUNT = 1`.
> The ROLLBACK undoes the UPDATE because it belongs to the main transaction. The log row was already committed by its own transaction, so the ROLLBACK can't reach it.

### Scenario 2: The forgotten COMMIT

```sql
CREATE OR REPLACE PROCEDURE log_error (p_msg VARCHAR2) IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO error_log VALUES (SYSTIMESTAMP, p_msg);
  -- no COMMIT
END;
/

BEGIN
  log_error('hello');
END;
/
```

**Question:** What happens?

> **Answer:** `ORA-06519: active autonomous transaction detected and rolled back`.
> The INSERT started a transaction that was still open when control returned to the caller. Oracle rolls it back and raises the error, so no row is logged.

### Scenario 3: Can it see my work?

```sql
CREATE OR REPLACE FUNCTION count_orders RETURN NUMBER IS
  PRAGMA AUTONOMOUS_TRANSACTION;
  v_cnt NUMBER;
BEGIN
  SELECT COUNT(*) INTO v_cnt FROM orders;
  RETURN v_cnt;
END;
/

DECLARE
  v_main_cnt NUMBER;
BEGIN
  INSERT INTO orders VALUES (102, 'NEW');   -- not committed
  SELECT COUNT(*) INTO v_main_cnt FROM orders;
  DBMS_OUTPUT.PUT_LINE('Inside caller: '   || v_main_cnt);
  DBMS_OUTPUT.PUT_LINE('Autonomous sees: ' || count_orders);
END;
/
```

**Question:** What do the two lines print?

> **Answer:** The caller sees `2`. The autonomous function sees `1`.
> Order 102 is uncommitted, so only the main transaction can see it. The autonomous transaction only sees committed data.
>
> **Bonus:** this function has no COMMIT, yet there's no ORA-06519. A plain SELECT doesn't start a transaction, so nothing is left open. ORA-06519 only appears after DML (or `SELECT ... FOR UPDATE`).

### Scenario 4: Deadlocking yourself

```sql
CREATE OR REPLACE PROCEDURE add_note (p_id NUMBER) IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  UPDATE orders SET status = 'NOTED' WHERE order_id = p_id;
  COMMIT;
END;
/

BEGIN
  UPDATE orders SET status = 'PROCESSING' WHERE order_id = 101;  -- locks row 101
  add_note(101);
  COMMIT;
END;
/
```

**Question:** What happens?

> **Answer:** `ORA-00060: deadlock detected while waiting for resource`.
> The main transaction holds the lock on row 101 and is paused while the autonomous procedure runs. The autonomous transaction needs that same lock and waits for it. Neither can continue, and Oracle detects the deadlock.
> **Takeaway:** an autonomous transaction should never modify rows the caller has already locked.

### Scenario 5: The audit trigger that records rejected changes

```sql
CREATE TABLE employees    (emp_id NUMBER, salary NUMBER);
CREATE TABLE salary_audit (emp_id NUMBER, old_sal NUMBER, new_sal NUMBER, changed_at TIMESTAMP);
INSERT INTO employees VALUES (1, 50000);
COMMIT;

CREATE OR REPLACE TRIGGER trg_salary_audit
BEFORE UPDATE OF salary ON employees
FOR EACH ROW
DECLARE
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO salary_audit VALUES (:OLD.emp_id, :OLD.salary, :NEW.salary, SYSTIMESTAMP);
  COMMIT;

  IF :NEW.salary > :OLD.salary * 1.5 THEN
    RAISE_APPLICATION_ERROR(-20010, 'Raise above 50% not allowed');
  END IF;
END;
/

UPDATE employees SET salary = 100000 WHERE emp_id = 1;
```

**Question:** Does the UPDATE succeed? Is there an audit row?

> **Answer:** The UPDATE fails with ORA-20010, and the salary stays at 50000. The audit row does exist, because it was committed before the error was raised.
> This is a real-world pattern: you keep a record of who *attempted* a suspicious change, not just the changes that went through.
> A normal trigger can't COMMIT (ORA-04092). The autonomous pragma is what makes the COMMIT legal here.

### Scenario 6: The exception handler that leaves the transaction open

```sql
CREATE OR REPLACE PROCEDURE safe_log (p_msg VARCHAR2) IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO error_log VALUES (SYSTIMESTAMP, p_msg);
  INSERT INTO error_log VALUES (SYSTIMESTAMP, RPAD('x', 500, 'x'));  -- too long, fails
  COMMIT;
EXCEPTION
  WHEN OTHERS THEN
    NULL;   -- "swallow the error"
END;
/

BEGIN
  safe_log('test');
END;
/
```

**Question:** What happens?

> **Answer:** `ORA-06519`.
> The first INSERT succeeded, so a transaction is open. The second INSERT fails and jumps to the handler, skipping the COMMIT. The handler does nothing, so the block ends with the transaction still open.
> **Fix:** put `ROLLBACK;` (or `COMMIT;`, depending on what you want to keep) inside the handler. Every exit path needs one.

### Scenario 7: Does the child's COMMIT commit the parent?

```sql
BEGIN
  UPDATE orders SET status = 'CANCELLED' WHERE order_id = 101;
  log_error('Cancelling 101');   -- the procedure from Scenario 1, which COMMITs
END;
/

-- Now, in the same session:
ROLLBACK;
SELECT status FROM orders WHERE order_id = 101;
```

**Question:** What's the status?

> **Answer:** `'NEW'`.
> The COMMIT inside `log_error` only committed the autonomous transaction. The parent's UPDATE stayed pending, and the ROLLBACK undid it.
> If `log_error` were a **normal** procedure, its COMMIT would have committed the UPDATE too, and the status would be `'CANCELLED'`. This is the most commonly tested difference.

### Scenario summary

| Scenario | Rule being tested |
|---|---|
| 1, 7 | Main ROLLBACK can't undo autonomous COMMIT, and autonomous COMMIT doesn't commit main |
| 2, 6 | Every exit path must COMMIT or ROLLBACK, or you get ORA-06519 |
| 3 | Autonomous can't see the caller's uncommitted data |
| 4 | Autonomous waits on the caller's locks, which can cause ORA-00060 |
| 5 | Autonomous triggers can COMMIT, and are used to audit rejected changes |

---

## Exercise Questions

**Q1.** A procedure inserts a row, then calls an autonomous logging procedure, and afterward the caller issues a ROLLBACK. Does the log entry get rolled back too?

> **A:** No. The logging procedure ran as an autonomous transaction, so its COMMIT created a permanent, independent boundary. The outer ROLLBACK only undoes the outer transaction's own uncommitted work (the row insert). It has no effect on the already-committed autonomous transaction.

**Q2.** Inside an autonomous procedure, what happens if the block completes without a COMMIT or ROLLBACK?

> **A:** Oracle raises `ORA-06519: active autonomous transaction detected and rolled back` and rolls back whatever the block did. There is no implicit commit. Falling off the end of the block, or letting an unhandled exception propagate without cleanup, triggers this error.

**Q3.** Why can't an autonomous transaction see the calling transaction's uncommitted changes?

> **A:** The autonomous transaction is fully independent. Oracle's read consistency only shows a transaction its own changes plus data other transactions have committed. The caller's uncommitted changes belong to a different transaction, so they're invisible to it, just as they would be to another user's session.