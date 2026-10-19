# Oracle Locking: Row Locks (TX), Table Locks (TM), Lock Compatibility, and Deadlocks

> Study guide: concept in simple words, real-world use cases, the compatibility table explained, deadlocks, and interview-style scenario questions with answers.

---

## 1. The Big Idea

When many people use the same data at the same time, you need rules so they don't overwrite each other's work. **Locks are those rules.**

Oracle locks at two levels:

| Lock | Meaning in plain words |
|---|---|
| **TX (row lock)** | "This specific row is mine until I commit." |
| **TM (table lock)** | A *signboard* on the table saying what kind of work is happening inside, so nobody does something incompatible to the whole table. |

### Hotel analogy

Think of a **table as a hotel** and **rows as rooms**.

- **TX lock** is the key to one room. Only one guest holds it at a time. Anyone else wanting that room waits in the lobby until the guest checks out (`COMMIT` or `ROLLBACK`).
- **TM lock** is the board at reception that says "Guests are staying here." It stops someone from demolishing or renovating the whole hotel (`DROP`, `ALTER`, an exclusive bulk job) while guests are inside. The renovator doesn't knock on every door. They just check the board.

That's why **RS and RX are called *intent* locks**. They don't block other guests. They only announce "I'm using some rooms," so whole-table operations know to wait.

---

## 2. The Problem Locks Solve: the "Lost Update"

**Scenario:** an e-commerce site has **1 iPhone left** in stock.

| Time | Customer A (Session 1) | Customer B (Session 2) |
|---|---|---|
| T1 | reads qty = 1 | |
| T2 | | reads qty = 1 |
| T3 | sets qty = 0, places order | |
| T4 | | sets qty = 0, places order |

Both bought the last phone, so the item is **oversold**. B's decision was based on stale data.

### Fix with `SELECT ... FOR UPDATE`

```sql
SELECT quantity INTO v_qty
FROM   inventory
WHERE  item_id = 501
FOR UPDATE;          -- "I'm reading this row BECAUSE I plan to change it"

IF v_qty > 0 THEN
   UPDATE inventory SET quantity = quantity - 1 WHERE item_id = 501;
END IF;
COMMIT;
```

Now B's `SELECT ... FOR UPDATE` **waits** at T2. After A commits, B reads `0` and gets "Out of stock." The sale is correct.

> **Rule of thumb:** If you **read a value, decide based on it, then write it back**, use `FOR UPDATE`.

---

## 3. Row-Level Locks (TX)

- Acquired automatically by `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `SELECT ... FOR UPDATE`.
- **Exactly one transaction** can hold a TX lock on a given row at a time.
- A second transaction trying to modify that row **waits** until the first commits or rolls back.
- Released **only** at `COMMIT` / `ROLLBACK`. There is no "UNLOCK" statement.

---

## 4. Table-Level Locks (TM): The Five Modes

| Mode | Name | Plain meaning | Acquired by |
|---|---|---|---|
| **RS** | Row Share | "I've reserved a few rows." | `LOCK TABLE ... IN ROW SHARE MODE` (classically also `SELECT ... FOR UPDATE`, see note) |
| **RX** | Row Exclusive | "I'm changing a few rows." | `INSERT`, `UPDATE`, `DELETE`, `MERGE` |
| **S** | Share | "Nobody change anything. Others may join this freeze." | `LOCK TABLE ... IN SHARE MODE` |
| **SRX** | Share Row Exclusive | "Nobody change anything, and only *I* may freeze it." | `LOCK TABLE ... IN SHARE ROW EXCLUSIVE MODE` |
| **X** | Exclusive | "The table is entirely mine." | `LOCK TABLE ... IN EXCLUSIVE MODE` |

Restrictiveness ladder (mild to strong):

```
RS  →  RX  →  S  →  SRX  →  X
```

> **Version note (good interview bonus):** Older documentation lists `SELECT ... FOR UPDATE` as taking **RS**. From **Oracle 11g onward**, it takes **RX** (row exclusive) on the table. Either way, the row itself gets a **TX** lock. If asked, say: "Classically RS; in modern versions Oracle uses RX."

> Even under an **X lock, plain `SELECT` still works**, because Oracle readers take no locks.

---

## 5. The Compatibility Table, Explained

### What it answers

> If **Session 1 already holds lock A**, can **Session 2 get lock B** on the same table, or must it wait?

|  Already held ↓ / Requested → | RS | RX | S | SRX | X |
|---|---|---|---|---|---|
| **RS**  | ✓ | ✓ | ✓ | ✓ | ✗ |
| **RX**  | ✓ | ✓ | ✗ | ✗ | ✗ |
| **S**   | ✓ | ✗ | ✓ | ✗ | ✗ |
| **SRX** | ✓ | ✗ | ✗ | ✗ | ✗ |
| **X**   | ✗ | ✗ | ✗ | ✗ | ✗ |

### How to read it

- **Row** = the lock Session 1 **already has**
- **Column** = the lock Session 2 is **asking for**
- **✓** = granted immediately; both work together
- **✗** = Session 2 **waits** (or gets `ORA-00054` with `NOWAIT` / after `WAIT n`)

The table is **symmetric**, so it doesn't matter which session came first.

### Row by row

**RS held** (rows reserved): very mild, so everything is allowed **except X**. X means "nobody else is inside," which can't be true while Session 1 is inside.

**RX held** (normal UPDATE running):
- RS ✓, RX ✓: others can reserve or update **other rows**. This is everyday concurrency.
- S, SRX, X ✗: they all mean "nobody may change the table," but Session 1 *is* changing it.

**S held** (shared freeze):
- RS ✓: reserving rows is fine.
- S ✓: two sessions saying "nobody change anything" don't contradict each other.
- RX ✗: an UPDATE would break the freeze.
- SRX, X ✗: they demand more ownership than a shared freeze allows.

**SRX held** (freeze, and I'm the only freezer): only RS is allowed. This is the key difference from S: **S allows other S holders, SRX doesn't.**

**X held**: everything ✗.

### Story example

1. Ravi: `UPDATE inventory SET qty = 5 WHERE item_id = 1;` (uncommitted). He holds **RX**.
2. Priya: `UPDATE inventory SET qty = 9 WHERE item_id = 2;` She needs **RX**. Row RX, column RX: **✓**. She proceeds, because the rows are different.
3. Admin: `LOCK TABLE inventory IN EXCLUSIVE MODE;` He needs **X**. Row RX, column X: **✗**. He **waits** until both commit.
4. Arjun: `SELECT * FROM inventory;` No lock needed, so it runs instantly and sees the last committed data.

> **The one cell to remember: RX + RX = ✓.** That's why many users can update the same table simultaneously in Oracle without blocking each other, as long as they touch different rows.

---

## 6. Manual Locking Syntax

```sql
LOCK TABLE inventory IN EXCLUSIVE MODE WAIT 5;   -- wait up to 5 sec, else ORA-00054
LOCK TABLE inventory IN ROW SHARE MODE NOWAIT;   -- fail immediately if unavailable
LOCK TABLE inventory IN SHARE MODE;              -- wait indefinitely (default)
```

`SELECT ... FOR UPDATE` options:

```sql
SELECT ... FOR UPDATE;                 -- wait forever
SELECT ... FOR UPDATE NOWAIT;          -- fail immediately with ORA-00054
SELECT ... FOR UPDATE WAIT 5;          -- wait up to 5 seconds, then ORA-30006
SELECT ... FOR UPDATE SKIP LOCKED;     -- skip rows locked by others
SELECT ... FOR UPDATE OF e.salary;     -- in a join, lock only rows of the table owning e.salary
```

---

## 7. When Do You Need Each Feature? (Use Cases)

### a) `FOR UPDATE`: read-then-modify safely
Use it for inventory, seat booking, wallet balance, or a **bank transfer** (lock both accounts before moving money).

### b) `NOWAIT` / `WAIT n`: don't hang the user
In a movie-ticket app, if someone is mid-booking seat A12, the second user shouldn't freeze forever.

```sql
SELECT * FROM seats WHERE seat_no = 'A12' FOR UPDATE NOWAIT;
-- ORA-00054 → show "Seat is being booked, try another"
```

### c) `SKIP LOCKED`: many workers, one queue
A food-delivery `orders` table has 5 background workers.

- Plain `FOR UPDATE`: all 5 try the same first row, and 4 of them wait.
- `SKIP LOCKED`: each worker skips taken rows and grabs the next free one.

```sql
SELECT order_id
FROM   orders
WHERE  status = 'PENDING'
FOR UPDATE SKIP LOCKED;
```

Worker 1 gets 101, Worker 2 gets 102, and so on, with no waiting and no duplicate processing.

### d) `LOCK TABLE ... EXCLUSIVE`: critical batch work
At month end, a bank runs interest calculation on all accounts. A deposit sneaking in midway would break the totals.

```sql
LOCK TABLE accounts IN EXCLUSIVE MODE WAIT 10;
-- run interest calculation on all rows
COMMIT;   -- lock released
```

Also used for archival, bulk price revisions, and reconciliation. It's rare in everyday application code.

---

## 8. Key Points for Interviews

1. **No lock escalation.** Some databases (e.g., SQL Server) convert many row locks into a table lock. Oracle never does, because lock information is stored inside the data block itself rather than in a central lock list. Locking 1 row or 1 million rows stays row-level.
2. **Readers never block writers; writers never block readers.** A plain `SELECT` sees the last **committed** version, rebuilt from undo (multiversion read consistency). Only writers of the *same row* block each other.
3. **Locks are released only at COMMIT/ROLLBACK.** Long uncommitted transactions are the #1 cause of blocking in production.
4. **DDL needs an exclusive lock.** `ALTER TABLE` while someone has uncommitted DML gives `ORA-00054: resource busy`. By default DDL doesn't wait; `DDL_LOCK_TIMEOUT` can make it wait.
5. **Deadlocks** (`ORA-00060`) are detected automatically. Oracle rolls back the **statement** (not the whole transaction) of one session, which must then commit or roll back. Prevent them by always locking rows **in a consistent order**.
6. **`SKIP LOCKED`** is the go-to pattern for concurrent queue processing.

---

## 9. Scenario Practice Questions (with Answers)

Try each one before reading the answer.

---

### Q1. Basic row conflict
Session A runs `UPDATE emp SET sal = 5000 WHERE empno = 7369;` and does not commit. Session B runs `UPDATE emp SET sal = 6000 WHERE empno = 7369;`. What happens?

<details><summary>Answer</summary>

Session B **waits (hangs)**. Session A holds a TX lock on row 7369. When A commits or rolls back, B proceeds and its update is applied on top. If A commits, the final salary is 6000 after B commits.
</details>

---

### Q2. Different rows, same table
Session A updates `empno = 7369`, and Session B updates `empno = 7499`, both uncommitted. Does B wait?

<details><summary>Answer</summary>

**No.** Both hold **RX** on the table, and RX + RX = ✓. Their TX locks are on different rows, so there is no conflict. This is why Oracle DML scales well.
</details>

---

### Q3. Reader vs writer
Session A updates `empno = 7369` from sal 800 to 5000 and doesn't commit. Session B runs `SELECT sal FROM emp WHERE empno = 7369;`. Does B wait? What value does it see?

<details><summary>Answer</summary>

B **does not wait** and sees **800**, the last committed value, rebuilt from undo. Readers never block writers, and writers never block readers.
</details>

---

### Q4. Lost update prevention
Two clerks run this PL/SQL simultaneously for the last seat on a flight:
```sql
SELECT seats_left INTO v FROM flights WHERE flight_id = 10;
IF v > 0 THEN UPDATE flights SET seats_left = seats_left - 1 WHERE flight_id = 10; END IF;
```
What can go wrong, and how do you fix it?

<details><summary>Answer</summary>

Both can read `seats_left = 1` before either updates, so both book the seat and the flight is overbooked. This is a lost update. **Fix:** add `FOR UPDATE` to the SELECT so the second session waits, then re-reads `0` after the first commits. An alternative is a single atomic statement: `UPDATE flights SET seats_left = seats_left - 1 WHERE flight_id = 10 AND seats_left > 0;` and then check `SQL%ROWCOUNT`.
</details>

---

### Q5. Don't make the user wait
In a booking UI, you don't want the user's screen to freeze if another user is booking the same seat. What would you use?

<details><summary>Answer</summary>

`SELECT ... FOR UPDATE NOWAIT`. If the row is locked, it raises **ORA-00054** immediately. Catch it in an exception handler and show "Seat is being booked, try again." Use `FOR UPDATE WAIT n` if a short wait is acceptable; that raises **ORA-30006** on timeout.
</details>

---

### Q6. Worker queue
You have 10 worker processes that each pick one `PENDING` job from a `jobs` table, process it, and mark it `DONE`. With plain `FOR UPDATE`, workers keep waiting on each other. How do you fix it?

<details><summary>Answer</summary>

Use `FOR UPDATE SKIP LOCKED`. Each worker skips rows already locked by another worker and locks the next available one, so there's no blocking and no job is processed twice.
```sql
SELECT job_id FROM jobs WHERE status = 'PENDING'
FOR UPDATE SKIP LOCKED;
```
(Typically fetch only 1 row or a small batch from a cursor.)
</details>

---

### Q7. Compatibility table: S vs RX
Session A runs `LOCK TABLE products IN SHARE MODE;`. Session B runs `UPDATE products SET price = 10 WHERE id = 1;`. What happens?

<details><summary>Answer</summary>

B **waits**. UPDATE needs **RX**, and row S, column RX in the table is ✗. A shared freeze doesn't allow changes. B proceeds after A commits or rolls back.
</details>

---

### Q8. S vs S, and SRX vs SRX
(a) Session A holds SHARE mode on `orders`. Can Session B also get SHARE mode?
(b) Session A holds SHARE ROW EXCLUSIVE mode. Can Session B get SHARE ROW EXCLUSIVE mode?

<details><summary>Answer</summary>

(a) **Yes.** S + S = ✓. Multiple sessions can hold a shared freeze together.
(b) **No.** SRX + SRX = ✗. Only one session can hold SRX. That is the core difference between S and SRX.
</details>

---

### Q9. Exclusive lock and queries
Session A runs `LOCK TABLE accounts IN EXCLUSIVE MODE;`. Session B runs `SELECT * FROM accounts;`. Session C runs `SELECT * FROM accounts FOR UPDATE;`. What happens to B and C?

<details><summary>Answer</summary>

- **B succeeds immediately.** A plain SELECT takes no lock, and Oracle's read consistency allows it even under X.
- **C waits.** `FOR UPDATE` needs a table lock (RS/RX), which is incompatible with X.
</details>

---

### Q10. DDL during active DML
A developer runs `ALTER TABLE emp ADD (bonus NUMBER);` while another session has an uncommitted update on `emp`. What happens?

<details><summary>Answer</summary>

The ALTER fails with **ORA-00054: resource busy and acquire with NOWAIT specified or timeout expired**. DDL needs an exclusive lock and doesn't wait by default. You can set `ALTER SESSION SET DDL_LOCK_TIMEOUT = 30;` to make DDL wait up to 30 seconds.
</details>

---

### Q11. Deadlock
Session A updates account 1, then tries to update account 2. Session B, at the same time, updates account 2, then tries to update account 1. What happens, and how do you prevent it?

<details><summary>Answer</summary>

Each session waits for the other's row, which is a **deadlock**. Oracle detects it automatically and raises **ORA-00060** in one session, rolling back only that **statement**. That session must then roll back (or commit) to release its locks. **Prevention:** always lock rows in a **consistent order** (e.g., lowest account_id first), or lock both rows up front with one `SELECT ... WHERE id IN (1,2) FOR UPDATE`.
</details>

---

### Q12. Lock escalation question
An interviewer asks: "If I update 5 million rows in one transaction, will Oracle convert the row locks into a table lock?"

<details><summary>Answer</summary>

**No.** Oracle never escalates locks. Row lock information is stored in the data block header (the ITL, Interested Transaction List), not in a central lock manager, so there's no memory pressure forcing escalation. The table only gets an RX lock, and other sessions can still update rows you haven't touched. This is a key difference from SQL Server and DB2.
</details>

---

### Q13. Duplicate primary key insert
Session A runs `INSERT INTO emp (empno) VALUES (9999);` without committing. Session B runs the same insert with `empno = 9999`. What happens?

<details><summary>Answer</summary>

Session B **waits**, because it can't yet know whether A's row will exist.
- If A **commits**, B gets **ORA-00001: unique constraint violated**.
- If A **rolls back**, B's insert **succeeds**.
</details>

---

### Q14. When are locks released?
A procedure locks rows with `FOR UPDATE`, then raises an unhandled exception and returns to the caller, who doesn't commit or roll back. Are the locks released?

<details><summary>Answer</summary>

**Not necessarily.** Locks are released only by `COMMIT` or `ROLLBACK`. An unhandled exception rolls back the failed statement (statement-level rollback), but earlier work and locks in the transaction remain until the caller (or the client tool) ends the transaction. Always end transactions explicitly. This is a common cause of "mysterious" blocking.
</details>

---

### Q15. COMMIT inside a FOR UPDATE cursor loop
```sql
CURSOR c IS SELECT * FROM emp FOR UPDATE;
...
FOR r IN c LOOP
   UPDATE emp SET sal = sal * 1.1 WHERE CURRENT OF c;
   COMMIT;
END LOOP;
```
What's the problem?

<details><summary>Answer</summary>

The `COMMIT` releases **all** locks taken by the `FOR UPDATE` cursor. The next fetch raises **ORA-01002: fetch out of sequence**. Commit **after** the loop. Also note that a `FOR UPDATE` cursor locks **all** qualifying rows when it is **OPENed**, not as each row is fetched.
</details>

---

### Q16. FOR UPDATE OF in a join
```sql
SELECT e.ename, d.dname
FROM   emp e JOIN dept d ON e.deptno = d.deptno
WHERE  d.deptno = 10
FOR UPDATE OF e.sal;
```
Which rows are locked?

<details><summary>Answer</summary>

Only the **emp** rows (from the table owning `sal`) for department 10. The `dept` row is **not** locked. Without `OF`, rows from **both** tables would be locked. Use `OF` to avoid locking more than you need.
</details>

---

### Q17. Choose the right lock (design question)
For each requirement, pick a locking approach:
1. Nightly job recalculates every customer's loyalty points; no new orders may be inserted midway.
2. Validate totals across a table; nobody may modify it during validation, but other validation jobs may run alongside.
3. Transfer ₹500 from account A to account B.
4. 20 microservice instances process payment events from one table.

<details><summary>Answer</summary>

1. `LOCK TABLE orders IN EXCLUSIVE MODE` (or SHARE mode if only preventing changes is needed), then run the job and commit.
2. `LOCK TABLE ... IN SHARE MODE`. S + S is compatible, so other validators can join, while all DML is blocked.
3. `SELECT ... FOR UPDATE` on both account rows (in a consistent order), update both, then commit.
4. `SELECT ... FOR UPDATE SKIP LOCKED` to build a contention-free work queue.
</details>

---

### Q18. Rapid-fire interview one-liners

| Question | Answer |
|---|---|
| Does a plain SELECT lock anything? | No. |
| Error for NOWAIT failure? | ORA-00054 |
| Error for `FOR UPDATE WAIT n` timeout? | ORA-30006 |
| Deadlock error? | ORA-00060 |
| Least restrictive table lock? | RS (Row Share) |
| Most restrictive table lock? | X (Exclusive) |
| Lock taken on the table by UPDATE? | RX (Row Exclusive) |
| Can two sessions hold RX on the same table? | Yes |
| Can two sessions hold SRX on the same table? | No |
| Does Oracle escalate row locks? | Never |
| Only ways to release a lock? | COMMIT or ROLLBACK |
| Pattern for parallel queue workers? | `FOR UPDATE SKIP LOCKED` (officially documented from 11g) |

---

## 10. Deadlocks

### 10.1 What a deadlock is

**Analogy:** two cars meet on a one-lane bridge from opposite sides. Car A won't reverse until B moves, and B won't reverse until A moves. Neither is wrong alone, but together they're stuck forever. Someone must step in and make one back up.

In Oracle, the cars are **sessions** and the bridge is the **rows they lock**.

### 10.2 The classic timeline

Two opposite transfers happen at the same moment:
- Session A: ₹100 from account 1 → account 2
- Session B: ₹50 from account 2 → account 1

| Time | Session A | Session B |
|---|---|---|
| T1 | `UPDATE ... WHERE acct_id = 1` → locks row 1 ✅ | |
| T2 | | `UPDATE ... WHERE acct_id = 2` → locks row 2 ✅ |
| T3 | `UPDATE ... WHERE acct_id = 2` → held by B, **waits** | |
| T4 | | `UPDATE ... WHERE acct_id = 1` → held by A, **waits** |

A waits for B, and B waits for A. That's a **cycle**, and it can never resolve on its own.

### 10.3 Deadlock vs normal blocking

| | Normal blocking | Deadlock |
|---|---|---|
| Shape | A chain: B waits for A, and A waits for nobody | A cycle: A waits for B, and B waits for A |
| Resolves by itself? | **Yes**, when A commits | **Never** |
| Oracle intervenes? | No, it just waits | Yes, with ORA-00060 |

### 10.4 What Oracle does

1. It detects the cycle automatically, usually within a few seconds.
2. It picks **one** session as the victim (say A).
3. It rolls back **only the stuck statement**, A's update on row 2.
4. It returns **ORA-00060: deadlock detected while waiting for resource** to A.

**The critical detail:** A's earlier update on row 1 is **not** rolled back. A **still holds row 1's lock**, and B is still waiting for row 1. **B proceeds only after A runs `ROLLBACK` (or `COMMIT`).** If A's code just logs the error and sits idle, B stays blocked.

> Oracle **cuts the knot** (rolls back one statement). Your code must **untie it** (`ROLLBACK` the transaction to release the remaining locks).

**Why not roll back the whole transaction?** Oracle only knows that this one statement can't proceed. It doesn't know your business logic, so it does the minimum and hands control back to the application.

### 10.5 Detection vs prevention (exam favourite)

- Oracle **detects and resolves** deadlocks.
- Oracle does **not prevent** them. **Prevention is the application's responsibility.**

### 10.6 Prevention techniques

| Technique | Why it works |
|---|---|
| **Consistent locking order** (most important) | If everyone locks account 1 before account 2, nobody can ever hold "the second" while waiting for "the first," so a cycle is impossible. |
| **Lock everything up front** with one `SELECT ... FOR UPDATE` | You either get all the rows or wait while holding none. You never discover mid-way that you need a lock someone else holds. |
| **Keep transactions short** | A smaller lock window means fewer overlaps. Never call external APIs or wait for user input while holding locks. |
| **NOWAIT / WAIT n plus retry** | Fail fast instead of building long wait chains under load. |
| **Minimize lock scope** | Lock only the rows you'll modify. Avoid unnecessary `LOCK TABLE ... EXCLUSIVE` or unfiltered `FOR UPDATE`. |
| **Index foreign key columns** | An unindexed FK makes parent DELETE/UPDATE take a table-level lock on the child, a hidden source of blocking and deadlocks. |

**Consistent order in code:**

```sql
-- Always lock the lower acct_id first, whichever direction the transfer goes
SELECT balance FROM accounts
WHERE  acct_id IN (LEAST(:from_id, :to_id), GREATEST(:from_id, :to_id))
ORDER  BY acct_id
FOR UPDATE;
```

### 10.7 Handling ORA-00060 in PL/SQL (with retry)

```sql
DECLARE
  e_deadlock EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_deadlock, -60);   -- name ORA-00060
  v_attempts NUMBER := 0;
BEGIN
  LOOP
    BEGIN
      v_attempts := v_attempts + 1;

      UPDATE accounts SET balance = balance - 100 WHERE acct_id = 1;
      UPDATE accounts SET balance = balance + 100 WHERE acct_id = 2;
      COMMIT;
      EXIT;                                 -- success

    EXCEPTION
      WHEN e_deadlock THEN
        ROLLBACK;                           -- release ALL locks
        IF v_attempts >= 3 THEN
          RAISE;                            -- give up after 3 tries
        END IF;
        DBMS_SESSION.SLEEP(1);              -- backoff (18c+; older: DBMS_LOCK.SLEEP)
    END;
  END LOOP;
END;
/
```

- `PRAGMA EXCEPTION_INIT` is cleaner than `WHEN OTHERS THEN IF SQLCODE = -60`.
- Other errors aren't caught, so they propagate normally.
- `ROLLBACK` before retrying is mandatory.

### 10.8 Diagnosing deadlocks

On ORA-00060, Oracle writes a **trace file** and an entry in the **alert log** showing both sessions, the rows involved, and the exact SQL. In practice, you find the two code paths that lock the same resources in **opposite order** and make one follow the common order.

---

## 11. Deadlock Scenario Questions (with Answers)

### D1. Blocking or deadlock?
Session A updates row 10 (uncommitted). Session B tries to update row 10 and waits. Session A is idle. Is this a deadlock?

<details><summary>Answer</summary>

**No, it's normal blocking.** B waits for A, but A isn't waiting for anyone, so there's no cycle. Oracle won't raise ORA-00060, and B simply waits until A commits or rolls back. (If A never ends its transaction, a DBA may need to find and kill the blocking session.)
</details>

---

### D2. Spot the deadlock
```
T1  A: UPDATE emp SET sal = sal + 100 WHERE empno = 7369;
T2  B: UPDATE emp SET sal = sal + 200 WHERE empno = 7499;
T3  A: UPDATE emp SET sal = sal + 100 WHERE empno = 7499;
T4  B: UPDATE emp SET sal = sal + 200 WHERE empno = 7369;
```
What happens at T3 and T4?

<details><summary>Answer</summary>

- **T3:** A waits for B (row 7499). This is ordinary blocking so far.
- **T4:** B waits for A (row 7369), which completes a **cycle**, so it's a deadlock.
- Within seconds, Oracle raises **ORA-00060** in one session and rolls back only that session's stuck statement.
</details>

---

### D3. What's left after ORA-00060?
Continuing D2, assume Session A receives ORA-00060. Which of A's changes are rolled back? Does B continue immediately?

<details><summary>Answer</summary>

- Only A's **stuck statement** (the T3 update on 7499) is rolled back.
- A's **T1 update on 7369 remains**, and A **still holds that row lock**.
- B is waiting for 7369, so **B does NOT continue yet**. It proceeds only after A issues `ROLLBACK` (or `COMMIT`).

This is why the application must `ROLLBACK` on ORA-00060.
</details>

---

### D4. "Oracle prevents deadlocks." True or false?

<details><summary>Answer</summary>

**False.** Oracle **detects and resolves** deadlocks after they occur by rolling back one statement and raising ORA-00060. **Prevention is the application's responsibility**, mainly through a consistent locking order.
</details>

---

### D5. Fix the code
Procedure `transfer(p_from, p_to, p_amt)` updates `p_from` first, then `p_to`. Users report occasional ORA-00060 when two people transfer money between the same two accounts in opposite directions. How do you fix it **structurally**?

<details><summary>Answer</summary>

Enforce a **consistent lock order** regardless of transfer direction, for example always locking the **lower account_id first**:
```sql
SELECT balance FROM accounts
WHERE  acct_id IN (LEAST(p_from, p_to), GREATEST(p_from, p_to))
ORDER  BY acct_id
FOR UPDATE;
-- then do both UPDATEs and COMMIT
```
Now both sessions compete for the same first row, so the second one waits while holding nothing, and a cycle is impossible. Retry logic alone treats the symptom; consistent ordering removes the cause.
</details>

---

### D6. Three-session deadlock
A holds row 1 and wants row 2. B holds row 2 and wants row 3. C holds row 3 and wants row 1. Is this a deadlock?

<details><summary>Answer</summary>

**Yes.** A → B → C → A forms a cycle. Deadlocks aren't limited to two sessions. Oracle detects cycles of any length, and one session gets ORA-00060.
</details>

---

### D7. Can NOWAIT eliminate deadlocks?
If every lock in the application is acquired with `SELECT ... FOR UPDATE NOWAIT` before modifying, can a deadlock still occur?

<details><summary>Answer</summary>

**Effectively no.** A deadlock requires sessions to **wait** on each other, and NOWAIT never waits. Instead you'll get **ORA-00054** immediately, which you handle with rollback and retry. The trade-off is more failed attempts under contention, so the application must handle ORA-00054 gracefully. (Any later plain UPDATE on rows *not* already locked could still wait, so lock all needed rows up front.)
</details>

---

### D8. The missing ROLLBACK bug
```sql
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE = -60 THEN
      log_error('deadlock');      -- no ROLLBACK here
    ELSE
      RAISE;
    END IF;
```
What's the problem?

<details><summary>Answer</summary>

After ORA-00060, the session **still holds its earlier locks**, and its transaction is **partially done** (some updates applied, one rolled back). Without `ROLLBACK`:
- The other session **stays blocked**.
- A later `COMMIT` could save an **inconsistent half-transfer** (money debited but not credited).

Always `ROLLBACK` (and optionally retry) on ORA-00060. Also note that if `log_error` uses an autonomous transaction, it's fine; otherwise its insert becomes part of the broken transaction.
</details>

---

### D9. Self-deadlock with an autonomous transaction
```sql
UPDATE emp SET sal = 9000 WHERE empno = 7369;   -- main transaction, uncommitted
-- then calls:
PROCEDURE audit_and_fix IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  UPDATE emp SET comm = 100 WHERE empno = 7369;
  COMMIT;
END;
```
What happens?

<details><summary>Answer</summary>

**ORA-00060 (deadlock) within a single session.** The autonomous transaction is a **separate transaction**, so it waits for the row lock held by the main transaction. But the main transaction is suspended until the autonomous one finishes, which is a cycle. Oracle raises ORA-00060 in the autonomous transaction. **Lesson:** never have an autonomous transaction modify rows the calling transaction has locked.
</details>

---

### D10. Unindexed foreign key
`orders.customer_id` references `customers.customer_id`, but `orders.customer_id` has **no index**. Users report blocking and occasional deadlocks when deleting customers while orders are being inserted. Why, and what's the fix?

<details><summary>Answer</summary>

When you DELETE a parent row (or update its key), Oracle must make sure no child rows reference it. Without an index on the FK column, it takes a **table-level lock on the child table** (`orders`) during the operation. That blocks concurrent DML on `orders` and, combined with row locks, can create deadlock cycles. **Fix:** create an index on `orders.customer_id`. Then Oracle uses row-level checks only. This is a classic real-world and interview topic.
</details>

---

### D11. Bitmap index deadlocks
An OLTP table with heavy concurrent inserts and updates has a **bitmap index** on `status`. Deadlocks keep appearing even though sessions update different rows. Why?

<details><summary>Answer</summary>

A single bitmap index entry covers **many rows** (a range of rowids for one key value). Updating one row's `status` locks the whole bitmap entry, which effectively locks many rows at once. Sessions modifying *different* rows can end up waiting on each other's bitmap entries and deadlock. **Fix:** use B-tree indexes on OLTP tables with concurrent DML. Bitmap indexes suit read-mostly data warehouse tables.
</details>

---

### D12. Which session becomes the victim?
In a two-session deadlock, can you control which session receives ORA-00060?

<details><summary>Answer</summary>

**No, not reliably.** Oracle chooses, typically the session whose wait detects the cycle. Because either side might be the victim, **every code path involved must handle ORA-00060** (roll back, then retry or report).
</details>

---

### D13. Where do you look?
Production logs show ORA-00060 a few times a day. How do you find the root cause?

<details><summary>Answer</summary>

Check the **alert log** for the ORA-00060 entry, which points to a **trace file**. The trace contains the **deadlock graph**: the sessions, the locks held and waited for, and the **exact SQL statements** involved. Identify the two code paths locking the same resources in **opposite order**, then fix them to follow one consistent order. Also look for unindexed foreign keys or bitmap indexes on OLTP tables.
</details>

---

### D14. Rapid-fire deadlock one-liners

| Question | Answer |
|---|---|
| Deadlock error code? | ORA-00060 |
| Does Oracle detect deadlocks automatically? | Yes |
| Does Oracle prevent deadlocks? | No; prevention is the application's job |
| What does Oracle roll back? | Only the victim's stuck **statement** |
| Does the victim still hold earlier locks? | Yes, until it commits or rolls back |
| What must the application do? | `ROLLBACK`, then retry or report |
| #1 prevention technique? | Consistent locking order |
| Can deadlocks involve 3+ sessions? | Yes, any cycle |
| Can one session deadlock itself? | Yes, via an autonomous transaction |
| Hidden deadlock causes? | Unindexed FKs, bitmap indexes in OLTP |
| Where are details logged? | Alert log + trace file (deadlock graph) |
| Cleaner way to catch it in PL/SQL? | `PRAGMA EXCEPTION_INIT(e, -60)` |

---

## 12. One-Line Summary

Oracle locks **only the exact rows you change** (TX). Table locks (TM) are mostly intent "signboards" that protect whole-table operations. You add explicit locking (`FOR UPDATE`, `NOWAIT`, `WAIT n`, `SKIP LOCKED`, `LOCK TABLE`) when you need to **read-then-write safely, avoid hanging users, run parallel workers, or freeze a table for batch work**.

A **deadlock** is a cycle of sessions each holding what the other needs. Oracle **detects** it and rolls back one statement (ORA-00060). Your code must **ROLLBACK and retry**, and you **prevent** deadlocks by **always locking resources in the same order**.