# Serializable vs. Read Committed — Study Guide

**Contents**
1. Concept: the two isolation levels
2. How Oracle does it (MVCC and undo)
3. Worked example and timeline
4. When to use which
5. Key points and exam traps
6. Exercise questions
7. Scenario practice (Levels 1–5)
8. Quick self-check table

---

# Part 1 — Concept

## The core question: when is your snapshot taken?

Both levels answer one question differently: when you run a query, which moment's committed data does Oracle show you?

- **Read Committed:** each **statement** takes its own snapshot at the moment that statement starts.
- **Serializable:** the whole **transaction** uses one snapshot, taken when the transaction starts.

That single difference explains everything below.

### Analogy

Think of a live cricket scoreboard.

- **Read Committed** is like glancing at the live scoreboard. Each time you look, you see the current score. If you look twice, you may see two different scores, because runs were scored in between.
- **Serializable** is like taking a photo of the scoreboard when you walk in. For the rest of your visit you only look at that photo. It never changes, even while the real match continues.

## The two isolation levels Oracle supports

| | Read Committed (Oracle's default) | Serializable |
|---|---|---|
| **Consistency granularity** | Statement-level: each statement sees data as committed at the moment that statement began | Transaction-level: every statement sees data as committed at the moment the transaction began |
| **Non-repeatable reads** | Possible: re-running the same query later in the transaction can see different data if another transaction committed in between | Not possible: the transaction's view is frozen from its own start |
| **Phantom reads** | Possible | Not possible |
| **Writers that conflict with concurrent commits** | Wait for the row lock, then proceed against the now-current version | If the row was changed by another transaction that committed after this one started, raises **ORA-08177: can't serialize access for this transaction**; the application must catch it and retry the whole transaction |
| **Setting it** | Default, or `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` / `ALTER SESSION SET ISOLATION_LEVEL = READ COMMITTED;` | `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;` / `ALTER SESSION SET ISOLATION_LEVEL = SERIALIZABLE;` |

### The two "anomalies" in the table

- **Non-repeatable read:** you read the same row twice and get different values.
- **Phantom read:** you run the same query with a condition twice, and new rows appear or disappear. For example, `SELECT COUNT(*) FROM accounts WHERE branch = 'CHN'` returns 10, someone inserts and commits a new account, and the second run returns 11.

Under Serializable neither can happen, because every read comes from the same frozen snapshot.

---

# Part 2 — How Oracle does it: MVCC, not lock-based reads

Oracle uses **multiversion concurrency control (MVCC)**. It doesn't stop other people from changing data in order to give you a consistent view. Instead:

1. When a session updates a row, Oracle writes the old version into **undo segments**.
2. When your query needs an older version of that row (as of statement start for Read Committed, or transaction start for Serializable), Oracle rebuilds it from undo.

As a result:
- **Readers never block writers.**
- **Writers never block readers.**
- Only **writer vs. writer on the same row** involves locks, and this works the same way at both isolation levels.

This is also why Serializable needs more undo space. A long serializable transaction may need versions from far back, and if that undo has been overwritten you get **ORA-01555 (snapshot too old)**.

---

# Part 3 — Worked example

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

## Timeline view

Account 1 starts with a balance of 1000.

```
Time   Session A (you)                     Session B (other user)
----   ---------------------------------   ---------------------------
T1     Transaction starts
T2     SELECT balance  → 1000
T3                                         UPDATE balance = 800; COMMIT;
T4     SELECT balance  → ???
T5     UPDATE balance = balance - 50 ...
```

**Under Read Committed:**
- At T4, A sees **800**. The new statement takes a fresh snapshot, and B's commit is now visible.
- At T5, the UPDATE applies to the current value. The result is **750**, with no error.

**Under Serializable:**
- At T4, A still sees **1000**, frozen at the T1 snapshot.
- At T5, Oracle notices that someone else changed and committed this row after A's transaction began, so it raises **ORA-08177**. A must roll back and retry the whole transaction.

---

# Part 4 — When to use which

The stronger guarantee of Serializable has costs:
- You must write **retry logic** for ORA-08177. This is normal and expected with Serializable, not an edge case.
- The longer a transaction runs, the more likely a conflicting commit happens, so retries increase.
- It needs more undo space.

So Oracle's guidance is to **use Read Committed by default**. Choose Serializable only when a transaction needs one consistent view across several statements, such as end-of-day reconciliation reports or multi-step calculations that must not see changes partway through. If you only need a frozen view and won't modify anything, `SET TRANSACTION READ ONLY;` gives you the snapshot without the ORA-08177 risk.

---

# Part 5 — Key points and exam traps

## Important points to remember

- Read Committed is Oracle's default and suits the vast majority of applications. Serializable is a deliberate, narrower choice for transactions that need a fully consistent, unchanging view of the data for their whole duration.
- ORA-08177 is raised only when the serializable transaction itself tries to **modify** a row that another transaction changed and committed after it began. It is not raised for **reading** changed data; reads simply keep returning the original snapshot values.
- Applications using SERIALIZABLE must implement retry logic that catches ORA-08177 and re-runs the whole transaction from the start. This is an expected part of using this level.
- Serializable transactions typically need more undo capacity and are best suited to short transactions. Long serializable transactions on a busy system are much more likely to hit ORA-08177.
- Oracle also offers a third mode, **READ ONLY** (not part of the SQL92 standard). It gives a fixed snapshot for the transaction's duration but allows no modifications.
- Both isolation levels use the same row-level locking for actual write conflicts. The difference is only about which consistent view a query sees, not how writes are locked.

## Exam traps

1. **ORA-08177 is raised only on a write**, such as an UPDATE, DELETE, or SELECT FOR UPDATE on a row that another transaction changed and committed after yours began.
2. **`SET TRANSACTION` must be the first statement of the transaction** (otherwise ORA-01453). Use `ALTER SESSION` to set the level for all later transactions in the session.
3. **Oracle supports only these two levels plus READ ONLY.** There is no READ UNCOMMITTED, so dirty reads never happen in Oracle.
4. **Serializable does not mean transactions run one after another.** They still run concurrently. Strictly speaking, Oracle's version is snapshot isolation, which can still allow **write skew** (see S15).

**One-line summary:** Read Committed gives each statement a fresh view of committed data, while Serializable gives the whole transaction one frozen view and raises ORA-08177 if you try to change a row someone else changed after your transaction began.

---

# Part 6 — Exercise questions

**Q1. A transaction running under Read Committed executes the same SELECT twice, several seconds apart, and gets two different results because another session committed a change in between. Is this a bug?**

**A:** No. This is the defined behavior of Read Committed. Consistency is guaranteed only at the statement level, so each SELECT sees whatever was committed as of that statement's own start time. Different statements in the same transaction can legitimately see different snapshots if commits happen in between. This is the **non-repeatable read** that Read Committed permits and Serializable prevents.

**Q2. A transaction running under SERIALIZABLE reads a row's balance, performs a calculation, and then tries to UPDATE that same row. Another session had already updated and committed a change to that row after this transaction started. What happens, and why?**

**A:** The UPDATE raises **ORA-08177: can't serialize access for this transaction**. A serializable transaction behaves as though it works against a single frozen snapshot taken at its start. Oracle cannot let it write a change based on now-stale data without breaking that guarantee, so it refuses the operation. The application must detect this and retry the whole transaction against current data.

**Q3. Why does Oracle's guidance generally favor Read Committed over Serializable, even though Serializable offers stronger consistency?**

**A:** Serializable's guarantee has practical costs. Transactions must retry, possibly repeatedly under heavy write load, whenever ORA-08177 occurs, which adds application complexity and can hurt throughput. Serializable transactions also need more undo capacity and suit short work best, since longer transactions face a higher chance of a conflicting commit during their frozen window. For most applications, statement-level consistency is enough and avoids these costs. Serializable is kept for cases that genuinely need one consistent view across the whole transaction, such as certain financial reconciliation or reporting jobs.

---

# Part 7 — Scenario practice

Every scenario uses this table unless stated otherwise:

```sql
CREATE TABLE accounts (acct_id NUMBER PRIMARY KEY, branch VARCHAR2(10), balance NUMBER);
-- Row 1: acct_id = 1, branch = 'CHN', balance = 1000
-- Row 2: acct_id = 2, branch = 'CHN', balance = 500
```

**Session A** = you. **Session B** = another user working at the same time.

---

## Level 1 — Basics

### S1. Two reads, one transaction
Session A (READ COMMITTED) runs:
```
T1  A: SELECT balance FROM accounts WHERE acct_id = 1;
T2  B: UPDATE accounts SET balance = 800 WHERE acct_id = 1; COMMIT;
T3  A: SELECT balance FROM accounts WHERE acct_id = 1;
```
What does A see at T1 and T3?

**Answer:** 1000 at T1 and **800** at T3. Under Read Committed, each statement takes a fresh snapshot. B committed before A's second SELECT started, so A sees the new value. This is a **non-repeatable read**. It is allowed under Read Committed and is not a bug.

---

### S2. Same thing, but Serializable
Same timeline, but A first ran `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;`.
What does A see at T3? Is any error raised?

**Answer:** A sees **1000** again, and **no error** is raised. The transaction is frozen at its start-time snapshot, so Oracle rebuilds the old version from undo. Only reading changed data never raises ORA-08177; the error comes only when A tries to *modify* a row that someone else changed.

---

### S3. Reading an uncommitted change
```
T1  B: UPDATE accounts SET balance = 0 WHERE acct_id = 1;   -- not committed
T2  A: SELECT balance FROM accounts WHERE acct_id = 1;
```
Does A wait? What value does A see? Does the isolation level matter?

**Answer:** A does **not** wait, and A sees **1000**. Readers never block writers, and writers never block readers. Oracle gives A the last committed version from undo. This is the same at both isolation levels, because Oracle never allows **dirty reads**. (Oracle has no READ UNCOMMITTED level.)

---

## Level 2 — Writes and conflicts

### S4. Serializable update on a changed row
```
T1  A: SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
T2  A: SELECT balance FROM accounts WHERE acct_id = 1;          -- 1000
T3  B: UPDATE accounts SET balance = 800 WHERE acct_id = 1; COMMIT;
T4  A: UPDATE accounts SET balance = balance - 50 WHERE acct_id = 1;
```
What happens at T4?

**Answer:** **ORA-08177: can't serialize access for this transaction.** The row was changed and committed by another transaction *after* A's transaction began. If A were allowed to write, it would act on a version it cannot see, which breaks the "frozen snapshot" guarantee. A must roll back and retry the whole transaction.

---

### S5. Same scenario, Read Committed
Same as S4, but A is in READ COMMITTED (no SET TRANSACTION). What happens at T4, and what is the final balance?

**Answer:** No error. The UPDATE applies to the **current** committed value: 800 − 50 = **750**. Under Read Committed, a writer simply works against the latest committed version of the row.

---

### S6. Waiting on an uncommitted writer (Serializable)
```
T1  A: SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
T2  B: UPDATE accounts SET balance = 800 WHERE acct_id = 1;   -- NOT committed yet
T3  A: UPDATE accounts SET balance = balance - 50 WHERE acct_id = 1;
```
What does A do at T3? What happens if B then **commits**? What if B **rolls back** instead?

**Answer:**
- At T3, A **waits** (blocks) on B's row lock. Writer-vs-writer on the same row always uses locks, at both levels.
- If B **commits**, A gets **ORA-08177**, because the row was changed by a transaction that committed after A started.
- If B **rolls back**, the row was never really changed, so A's update **succeeds** (balance becomes 950).

---

### S7. A different row
```
T1  A: SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
T2  B: UPDATE accounts SET balance = 0 WHERE acct_id = 2; COMMIT;
T3  A: UPDATE accounts SET balance = balance - 50 WHERE acct_id = 1;
```
Error or not?

**Answer:** **No error.** ORA-08177 is about A modifying a row that someone else changed. B touched row 2, and A is updating row 1, so there is no conflict. Locking and conflict detection work at the row level.

*(Rare edge case: Oracle can occasionally raise ORA-08177 because of block-level space settings such as INITRANS on very busy blocks. That is a tuning detail, not part of the concept.)*

---

### S8. The lost update trap (Read Committed)
```sql
-- Session A, READ COMMITTED, PL/SQL
SELECT balance INTO v_bal FROM accounts WHERE acct_id = 1;   -- 1000
-- B: UPDATE accounts SET balance = balance + 200 WHERE acct_id = 1; COMMIT;  (now 1200)
UPDATE accounts SET balance = v_bal - 50 WHERE acct_id = 1;
COMMIT;
```
What is the final balance? What went wrong, and how do you fix it?

**Answer:** The final balance is **950**, and B's +200 is **lost**. A calculated from a stale value held in a PL/SQL variable and overwrote B's change. Read Committed does not protect you from this because A's UPDATE statement itself didn't conflict with anything.

Fixes:
1. Do the math in SQL: `UPDATE accounts SET balance = balance - 50 ...` (uses the current value).
2. Lock the row when reading: `SELECT balance INTO v_bal FROM accounts WHERE acct_id = 1 FOR UPDATE;` (B then has to wait for A).
3. Use SERIALIZABLE. A would get ORA-08177 instead of silently losing B's update, and then retry.

---

## Level 3 — Multi-statement consistency

### S9. Phantom rows
Session A runs this twice in one transaction:
```sql
SELECT COUNT(*) FROM accounts WHERE branch = 'CHN';
```
Between the two runs, B inserts a new 'CHN' account and commits. What does A see under each level?

**Answer:**
- **Read Committed:** 2, then **3**. The new row "appears." This is a **phantom read**.
- **Serializable:** 2, then **2**. The new row isn't in A's start-time snapshot.

---

### S10. The end-of-day report
A report runs three queries one after another:
1. Total balance of all accounts
2. Total of today's withdrawals
3. Total of today's deposits

Transfers keep happening while it runs, and the numbers don't reconcile. Why? Which setting fixes it, and which is the **best** choice?

**Answer:** Under Read Committed, each query sees a different moment in time, so a transfer committed between query 1 and query 3 is counted in some totals but not others. Running all three against **one** snapshot fixes it.
- `SERIALIZABLE` works.
- **`SET TRANSACTION READ ONLY;`** is the better choice. It gives the same frozen snapshot, and because the report never writes, it can never hit ORA-08177. It also prevents the report from changing data by accident.

---

### S11. Seeing your own changes
```
T1  A: SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
T2  A: UPDATE accounts SET balance = 2000 WHERE acct_id = 1;
T3  A: SELECT balance FROM accounts WHERE acct_id = 1;
```
Does A see 1000 (the frozen snapshot) or 2000?

**Answer:** **2000.** The snapshot freezes *other transactions'* commits. A transaction always sees its own changes.

---

## Level 4 — Application design

### S12. Writing the retry logic
Write a PL/SQL block that withdraws 50 from account 1 under SERIALIZABLE and retries up to 3 times on ORA-08177.

**Answer:**
```sql
DECLARE
  e_cant_serialize EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_cant_serialize, -8177);
  v_bal NUMBER;
BEGIN
  FOR attempt IN 1 .. 3 LOOP
    BEGIN
      SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
      SELECT balance INTO v_bal FROM accounts WHERE acct_id = 1;
      UPDATE accounts SET balance = v_bal - 50 WHERE acct_id = 1;
      COMMIT;
      EXIT;                                -- success: leave the loop
    EXCEPTION
      WHEN e_cant_serialize THEN
        ROLLBACK;                          -- discard and start fresh
        IF attempt = 3 THEN RAISE; END IF; -- give up after 3 tries
    END;
  END LOOP;
END;
/
```
Key points:
- `PRAGMA EXCEPTION_INIT` gives ORA-08177 a name you can catch.
- Retry the **whole transaction**, including the reads, not just the failed UPDATE. The rollback ends the old snapshot, and `SET TRANSACTION` starts a new one with current data.
- Cap the retries so a constant conflict can't loop forever.

---

### S13. SET TRANSACTION in the wrong place
```sql
UPDATE accounts SET balance = 900 WHERE acct_id = 2;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```
What happens?

**Answer:** Error **ORA-01453: SET TRANSACTION must be first statement of transaction.** The UPDATE already started a transaction. Either COMMIT or ROLLBACK first, or use `ALTER SESSION SET ISOLATION_LEVEL = SERIALIZABLE;` to make it the default for all later transactions in the session.

---

### S14. The long-running serializable job
A batch job runs SERIALIZABLE for 2 hours on a busy OLTP system. It keeps failing with ORA-08177, and sometimes with ORA-01555 (snapshot too old). Explain both, and suggest a better design.

**Answer:**
- **ORA-08177:** over 2 hours, it's very likely that some other session commits a change to a row the job later updates. The longer the window, the more conflicts.
- **ORA-01555:** to rebuild the 2-hour-old snapshot, Oracle needs old undo data. On a busy system, that undo is overwritten before the job reads it.
- **Better design:** break the job into short transactions under Read Committed, with row-level locking (`SELECT ... FOR UPDATE`) where needed. If it only reads, use READ ONLY and make sure undo retention is large enough.

---

## Level 5 — Advanced (beyond most exams)

### S15. Write skew
Rule: at least one doctor must stay on call. Both are on call now.
```
A (SERIALIZABLE): SELECT COUNT(*) FROM doctors WHERE on_call = 'Y';   -- 2, OK to leave
B (SERIALIZABLE): SELECT COUNT(*) FROM doctors WHERE on_call = 'Y';   -- 2, OK to leave
A: UPDATE doctors SET on_call = 'N' WHERE name = 'Asha';  COMMIT;
B: UPDATE doctors SET on_call = 'N' WHERE name = 'Ravi';  COMMIT;
```
Does Oracle stop this? How many doctors are on call afterward?

**Answer:** Oracle does **not** stop it, and **zero** doctors are on call. Each transaction updated a **different** row, so neither one wrote to a row the other changed, and no ORA-08177 is raised. This is **write skew**. Oracle's "serializable" is really **snapshot isolation**, which does not catch conflicts between rows that are only *read*.

Fix: lock the rows you're making a decision on, for example `SELECT ... FROM doctors WHERE on_call = 'Y' FOR UPDATE;`, so the second transaction has to wait and then sees the updated state.

---

# Part 8 — Quick self-check

| Situation | Read Committed | Serializable |
|---|---|---|
| Re-read a row another session changed and committed | New value | Old value, no error |
| Update a row another session changed and committed after you started | Works on current value | ORA-08177 |
| Read a row another session changed but hasn't committed | Old committed value, no wait | Same |
| Update a row another session is changing (uncommitted) | Wait, then proceed | Wait; ORA-08177 if they commit |
| New matching rows inserted by others | Appear (phantom) | Don't appear |
| Your own uncommitted changes | Visible | Visible |
| Dirty reads | Never | Never |