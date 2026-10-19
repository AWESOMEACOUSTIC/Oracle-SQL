# Oracle Locking: Row Locks (TX), Table Locks (TM), and Lock Compatibility

> Study guide: concept in simple words, real-world use cases, the compatibility table explained, and interview-style scenario questions with answers.

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

## 10. One-Line Summary

Oracle locks **only the exact rows you change** (TX). Table locks (TM) are mostly intent "signboards" that protect whole-table operations. You add explicit locking (`FOR UPDATE`, `NOWAIT`, `WAIT n`, `SKIP LOCKED`, `LOCK TABLE`) when you need to **read-then-write safely, avoid hanging users, run parallel workers, or freeze a table for batch work**.