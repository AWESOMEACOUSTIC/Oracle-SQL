# Oracle SQL Certification Practice — Topic 1
## SQL Fundamentals: Data Types, Language Categories & Core Operators

---

### Subtopic 1: SQL Data Types

**Question 1**
The COMMENT column is defined as CHAR(20). You insert the value 'OK' into it. Which statement is true regarding how Oracle stores and compares this value?

A. Oracle stores exactly 2 characters ('OK'), and any WHERE clause comparison against 'OK' will match regardless of trailing spaces, just like VARCHAR2.
B. Oracle right-pads the stored value with spaces up to 20 characters, but `WHERE comment = 'OK'` still evaluates to TRUE because Oracle blank-pads the shorter operand when comparing two CHAR values.
C. Oracle right-pads the value to 20 characters, and `WHERE comment = 'OK'` will return FALSE unless you explicitly pad the literal with 18 trailing spaces.
D. Oracle raises an error because 'OK' does not fill the fixed length of 20 characters.

**Correct Answer:** B

**Detailed Explanation:**
CHAR is a fixed-length datatype. Oracle right-pads stored values with trailing blanks up to the declared length. When comparing two CHAR values (or a CHAR to a character literal), Oracle applies **blank-padding comparison semantics**: the shorter operand is conceptually padded with spaces before comparison, so `'OK'` compares equal to `'OK' || RPAD(' ',18)`. This is the key contrast with VARCHAR2, which uses **nonpadded comparison semantics** — trailing spaces are never added and do matter in comparisons.

**Why the Other Options Are Wrong:**
A. Describes VARCHAR2 behavior (no automatic padding) incorrectly applied to CHAR.
C. Inverts the rule — you never need to manually pad the literal; Oracle does the padding internally for the comparison.
D. CHAR(n) silently accepts any string ≤ n characters; it doesn't error for shorter input.

**Concept Tested:** CHAR fixed-length blank-padding vs. VARCHAR2 nonpadded comparison.

**Exam Trap:** Assuming CHAR and VARCHAR2 always compare identically — the blank-padding rule is CHAR-specific.

---

**Question 2**
A column is defined as `NUMBER(5,2)`. You execute:
`INSERT INTO t (amt) VALUES (123.456);`
What happens?

A. The insert fails with ORA-01438 because 123.456 requires 6 significant digits.
B. The insert succeeds, and Oracle rounds the value to 123.46 (5 total significant digits: 3 before, 2 after the decimal).
C. The insert succeeds, and Oracle truncates the value to 123.45.
D. The insert succeeds, and the value is stored as 123.456, because NUMBER(5,2) only restricts display formatting, not storage.

**Correct Answer:** B

**Detailed Explanation:**
For `NUMBER(p,s)`, p is the total number of significant digits and s is the number of digits to the right of the decimal point; the maximum integer digits allowed is p−s = 3. 123.456 rounds to the declared scale first (123.46), which uses exactly 3 integer digits + 2 decimal digits = 5 total — within precision. Oracle **rounds** to fit the scale before checking precision.

**Why the Other Options Are Wrong:**
A. ORA-01438 would trigger only if the *integer* portion needed more than p−s digits (e.g., inserting 1234.56 here) — 123 fits fine.
C. Oracle rounds, it does not truncate, when storing NUMBER values that exceed the declared scale.
D. NUMBER(p,s) constrains actual stored precision/scale, not merely display.

**Concept Tested:** NUMBER(p,s) precision/scale rules and rounding-on-storage.

**Exam Trap:** Assuming truncation instead of rounding, or miscounting how p and s split between integer and fractional digits.

---

**Question 3**
Assuming `EMPLOYEE_ID` is `NUMBER(6)`, which statement will raise **ORA-01722: invalid number** at runtime?

A. `SELECT * FROM employees WHERE employee_id = '100';`
B. `SELECT * FROM employees WHERE employee_id = '100A';`
C. `SELECT * FROM employees WHERE TO_CHAR(employee_id) = '100';`
D. `SELECT employee_id + '50' FROM employees;`

**Correct Answer:** B

**Detailed Explanation:**
Oracle implicitly converts a character literal to NUMBER when it's compared or combined with a NUMBER value — but only when the string is a valid numeric representation. `'100'` and `'50'` convert cleanly. `'100A'` cannot be converted and raises ORA-01722.

**Why the Other Options Are Wrong:**
A. `'100'` converts implicitly and cleanly to 100.
C. Converts `employee_id` to a string via `TO_CHAR` instead, sidestepping any numeric-conversion issue.
D. `'50'` is a valid numeric string; implicit conversion for the arithmetic succeeds.

**Concept Tested:** Implicit VARCHAR2 → NUMBER conversion and its limits.

**Exam Trap:** Assuming all character-to-number implicit conversions are equally safe — the trap is spotting exactly which literal breaks the rule.

---

**Question 4**
Which statement is TRUE about Oracle's treatment of an empty string (`''`) assigned to a VARCHAR2 column?

A. Oracle stores it as a zero-length string distinct from NULL; `column = ''` returns the row.
B. Oracle stores it as NULL; `column = ''` will never return that row, but `column IS NULL` will.
C. Oracle raises an error, since `''` is not a valid VARCHAR2 literal.
D. Oracle stores it as a zero-length string, but treats it as NULL only inside arithmetic expressions.

**Correct Answer:** B

**Detailed Explanation:**
This is a well-known **Oracle-specific** deviation from the ANSI SQL standard: Oracle currently treats a zero-length character string as NULL. Inserting `''` stores an actual NULL, so any `= ''` comparison follows three-valued logic and evaluates to UNKNOWN (returns no rows), while `IS NULL` correctly finds it.

**Why the Other Options Are Wrong:**
A. Describes ANSI-standard/other-RDBMS behavior, not Oracle's.
C. No error is raised — the value is silently converted to NULL.
D. The NULL treatment isn't context-limited; it's stored as a genuine NULL everywhere.

**Concept Tested:** Oracle's empty-string-equals-NULL behavior.

**Exam Trap:** Assuming standard SQL empty-string semantics apply universally across RDBMS platforms.

---

**Question 5**
Which two statements correctly differentiate DATE and TIMESTAMP in Oracle? **(Choose two)**

A. DATE stores century, year, month, day, hour, minute, and second, but no fractional seconds.
B. TIMESTAMP can store fractional seconds, with configurable precision up to 9 digits.
C. DATE always requires exactly 11 bytes of storage.
D. TIMESTAMP cannot be used in date-arithmetic expressions with a NUMBER.

**Correct Answers:** A and B

**Detailed Explanation:**
DATE is a 7-byte field (century, year, month, day, hour, minute, second) with no sub-second component. TIMESTAMP adds a fractional-seconds component with precision from 0–9 digits (default 6). This distinction — "TIMESTAMP is not just DATE with a fancier name" — is heavily tested.

**Why the Other Options Are Wrong:**
C. DATE storage is variable up to 7 bytes, not a fixed 11 — this figure is a distractor testing storage-trivia confusion.
D. TIMESTAMP still supports `+`/`-` arithmetic with NUMBER (adding/subtracting days), just as DATE does.

**Concept Tested:** DATE vs. TIMESTAMP storage/precision differences.

**Exam Trap:** Treating DATE and TIMESTAMP as functionally interchangeable.

---

### Subtopic 2: Data Manipulation Language (INSERT, UPDATE, DELETE)

**Question 1**
The DEPARTMENTS table has columns `(DEPARTMENT_ID, DEPARTMENT_NAME, MANAGER_ID, LOCATION_ID)` in that order. You execute:
`INSERT INTO departments VALUES (280, 'Recruiting', NULL);`

A. The statement succeeds; LOCATION_ID defaults to NULL.
B. The statement fails with ORA-00947 ("not enough values") because only 3 values were supplied for a 4-column table with no column list specified.
C. The statement succeeds because Oracle allows partial column lists as long as trailing columns are nullable.
D. The statement fails because MANAGER_ID cannot be explicitly set to NULL.

**Correct Answer:** B

**Detailed Explanation:**
Omitting the column list means a value must be supplied for **every** column, in table-definition order. Only 3 values were given for 4 columns, so Oracle raises ORA-00947. To insert with an implied NULL for LOCATION_ID, you'd either add a 4th explicit NULL, or name the columns you're supplying explicitly.

**Why the Other Options Are Wrong:**
A. Oracle does not auto-fill missing trailing values with NULL when the column list is omitted.
C. No such "partial list without naming columns" tolerance exists.
D. MANAGER_ID can absolutely be set to NULL explicitly — that's not the actual problem here.

**Concept Tested:** INSERT syntax — implicit column ordering and required value count.

**Exam Trap:** Assuming missing trailing values are silently NULL-filled.

---

**Question 2**
True or False: `UPDATE employees SET salary = salary * 1.1;` (no WHERE clause) raises an error, because a WHERE clause is mandatory in an UPDATE statement.

A. True
B. False

**Correct Answer:** B (False)

**Detailed Explanation:**
WHERE is optional on both UPDATE and DELETE. Omitting it means every row in the table is affected — syntactically valid, though often unintended.

**Concept Tested:** UPDATE/DELETE syntax — optionality of WHERE.

**Exam Trap:** Confusing "dangerous practice" with "illegal syntax."

---

**Question 3**
Which two statements correctly distinguish DELETE from TRUNCATE in Oracle? **(Choose two)**

A. DELETE is DML and can be rolled back before COMMIT; TRUNCATE is DDL and cannot be rolled back once issued.
B. TRUNCATE fires any row-level DELETE triggers defined on the table, while DELETE does not.
C. TRUNCATE resets the table's high-water mark; a DELETE of all rows does not.
D. Both DELETE and TRUNCATE require a WHERE clause to restrict which rows are removed.

**Correct Answers:** A and C

**Detailed Explanation:**
DELETE generates undo/redo and can be rolled back until COMMIT. TRUNCATE is DDL — it deallocates storage, resets the high-water mark, and (because DDL triggers an implicit COMMIT) cannot be rolled back. This DML-vs-DDL distinction is one of the most heavily tested crossover concepts.

**Why the Other Options Are Wrong:**
B. Reversed — TRUNCATE bypasses row-level processing entirely and does **not** fire triggers; DELETE does.
D. TRUNCATE cannot take a WHERE clause at all (it always removes every row); DELETE's WHERE is optional, not required.

**Concept Tested:** DELETE (DML) vs. TRUNCATE (DDL) behavior.

**Exam Trap:** Treating TRUNCATE as "just a faster DELETE" and forgetting the implicit-commit/no-rollback and no-trigger consequences.

---

**Question 4**
Which clause allows a single INSERT to conditionally target different tables, stopping at the **first** matching WHEN condition per source row?

A. INSERT ALL
B. INSERT FIRST
C. MERGE
D. INSERT ANY

**Correct Answer:** B

**Detailed Explanation:**
Multitable INSERT supports INSERT ALL (every WHEN condition is evaluated independently — a row could satisfy and insert into multiple branches) and INSERT FIRST (short-circuits at the first true WHEN, like IF/ELSIF).

**Why the Other Options Are Wrong:**
A. Evaluates all conditions independently rather than stopping at the first match.
C. MERGE conditionally inserts/updates a single target based on a join match — not multitable branching.
D. Not valid Oracle syntax.

**Concept Tested:** Multitable INSERT — INSERT ALL vs. INSERT FIRST.

**Exam Trap:** Confusing INSERT ALL's independent-evaluation semantics with INSERT FIRST's short-circuit behavior.

---

**Question 5**
Evaluate:
```sql
MERGE INTO target t
USING source s ON (t.id = s.id)
WHEN MATCHED THEN UPDATE SET t.val = s.val
WHEN NOT MATCHED THEN INSERT (id, val) VALUES (s.id, s.val);
```
If a SOURCE row has no matching ID in TARGET, what happens to it?

A. It is silently ignored.
B. It is inserted into TARGET via the INSERT clause.
C. The entire MERGE statement fails.
D. It is updated in TARGET with NULL values.

**Correct Answer:** B

**Detailed Explanation:**
Each source row is evaluated against the ON condition: matched rows follow WHEN MATCHED (here, UPDATE); unmatched rows follow WHEN NOT MATCHED (here, INSERT). This "upsert" branching is the entire purpose of MERGE.

**Why the Other Options Are Wrong:**
A. Unmatched rows are handled explicitly by WHEN NOT MATCHED, not dropped.
C. This is precisely the scenario MERGE is built to handle without error.
D. Confuses the UPDATE and INSERT branches.

**Concept Tested:** MERGE statement branching logic.

**Exam Trap:** Assuming unmatched rows are dropped, or that both branches fire for the same row.

---

### Subtopic 3: Data Query Language (SELECT, FETCH FIRST)

**Question 1**
Which two approaches reliably return the 3 employees with the highest salary? **(Choose two)**

A. `SELECT * FROM employees WHERE ROWNUM <= 3 ORDER BY salary DESC;`
B. `SELECT * FROM employees ORDER BY salary DESC FETCH FIRST 3 ROWS ONLY;`
C. `SELECT * FROM employees FETCH FIRST 3 ROWS ONLY ORDER BY salary DESC;`
D. `SELECT * FROM (SELECT * FROM employees ORDER BY salary DESC) WHERE ROWNUM <= 3;`

**Correct Answers:** B and D

**Detailed Explanation:**
FETCH FIRST is applied *after* ORDER BY within the same SELECT (B), so it correctly returns the top 3. ROWNUM, by contrast, is assigned to rows as they're fetched — *before* that query block's own ORDER BY logically takes effect — so filtering `WHERE ROWNUM <= 3` in the same block (A) grabs an arbitrary 3 rows and only then sorts them. Wrapping the ordered result in an inline view and filtering ROWNUM in the *outer* query (D) is the classic correct workaround, because ordering is fully resolved before ROWNUM applies outside.

**Why the Other Options Are Wrong:**
A. Filters before the sort takes effect in that same block — not guaranteed top 3.
C. Placing FETCH FIRST before ORDER BY is invalid syntax; the FETCH clause must follow ORDER BY when both are present.

**Concept Tested:** Row-limiting — FETCH FIRST vs. ROWNUM-with-ORDER BY pitfalls.

**Exam Trap:** Arguably the single most common Oracle-cert ROWNUM trap — assuming ROWNUM is assigned after ORDER BY within the same query block.

---

**Question 2**
What does `WITH TIES` do when used with `FETCH FIRST n ROWS`?

A. Returns exactly n rows, discarding any ties.
B. Returns n rows, plus any further rows whose ORDER BY key value ties with the nth row's value.
C. Is valid only when no ORDER BY clause is present.
D. Returns duplicate rows even without ties.

**Correct Answer:** B

**Detailed Explanation:**
`FETCH FIRST n ROWS WITH TIES` returns the first n rows per the ORDER BY, then additionally includes any rows whose ordering-column value ties with the nth row — a "top N including ties" pattern.

**Why the Other Options Are Wrong:**
A. The opposite of the actual behavior.
C. Backwards — ORDER BY is required; "tie" is undefined without it.
D. Unrelated to general duplicate rows; only concerns ties in the ordering value.

**Concept Tested:** FETCH FIRST ... WITH TIES.

**Exam Trap:** Forgetting WITH TIES requires an ORDER BY, and confusing it with generic de-duplication.

---

**Question 3**
A view is created as `CREATE VIEW v AS SELECT * FROM employees;`. Later, a new column `PHONE_NUMBER2` is added to EMPLOYEES via ALTER TABLE. Does `SELECT * FROM v;` now include PHONE_NUMBER2?

A. Yes — SELECT * views automatically reflect newly added base-table columns.
B. No — the view's column list is captured at creation time and does not auto-update; the view must be recreated (or CREATE OR REPLACE'd).
C. It depends on whether the table was truncated afterward.
D. Yes, but only after re-executing the view with EXECUTE IMMEDIATE.

**Correct Answer:** B

**Detailed Explanation:**
Even though the view uses `SELECT *`, Oracle expands and stores the column list in the view's metadata at creation time. New base-table columns are invisible to the view until it's dropped/recreated or replaced.

**Why the Other Options Are Wrong:**
A. This is exactly the misconception being tested.
C. TRUNCATE is unrelated to view column metadata.
D. Not a real mechanism.

**Concept Tested:** Views with SELECT * and schema-evolution behavior.

**Exam Trap:** Assuming SELECT * views are "live" with respect to new base-table columns.

---

**Question 4**
Which clause retrieves the smallest number of rows that make up at least the top 10 **percent** of a result set (rather than a fixed row count)?

A. `FETCH FIRST 10 ROWS ONLY`
B. `FETCH FIRST 10 PERCENT ROWS ONLY`
C. `WHERE ROWNUM <= 10 PERCENT`
D. `FETCH 10 PERCENT ONLY`

**Correct Answer:** B

**Detailed Explanation:**
The row-limiting clause supports both a row-count form and a percentage form (`FETCH FIRST n PERCENT ROWS ONLY`), which returns the smallest number of rows constituting at least that percentage of the total result, based on the ORDER BY.

**Why the Other Options Are Wrong:**
A. Fixed count, not a percentage.
C. ROWNUM has no PERCENT syntax.
D. Invalid keyword sequence.

**Concept Tested:** FETCH FIRST n PERCENT syntax.

**Exam Trap:** Assuming ROWNUM supports percentage filtering, and misremembering required keyword order.

---

### Subtopic 4: Data Control Language (GRANT, REVOKE)

**Question 1**
Which two statements about GRANT/REVOKE are correct in Oracle? **(Choose two)**

A. `WITH GRANT OPTION` applies to object privileges and lets the grantee re-grant that same privilege to others.
B. `WITH ADMIN OPTION` applies to system privileges/roles and lets the grantee grant/revoke that privilege to others.
C. Revoking a system privilege granted `WITH ADMIN OPTION` automatically cascades to revoke it from everyone who received it downstream through the grantee.
D. GRANT and REVOKE are DML statements because they modify rows in the data dictionary.

**Correct Answers:** A and B

**Detailed Explanation:**
`WITH GRANT OPTION` is specific to object privileges; `WITH ADMIN OPTION` is its counterpart for system privileges and roles. This object-vs-system pairing is a classic DCL distinction.

**Why the Other Options Are Wrong:**
C. Reversed: revoking a **system** privilege does NOT cascade to further grantees (no dependency tracking); revoking an **object** privilege granted WITH GRANT OPTION DOES cascade. Mixing these up is exactly the trap.
D. GRANT/REVOKE are DCL, not DML — they control access, not table data.

**Concept Tested:** WITH GRANT OPTION vs. WITH ADMIN OPTION; cascading REVOKE behavior.

**Exam Trap:** Reversing which privilege type cascades on revoke — one of the most commonly missed DCL facts.

---

**Question 2**
What is the effect of `GRANT SELECT ON employees TO PUBLIC;`?

A. Grants SELECT to every current and future user/role in the database.
B. Grants SELECT only to users currently belonging to the PUBLIC role.
C. Has no effect — PUBLIC is not a valid grantee.
D. Grants SELECT to the table owner only.

**Correct Answer:** A

**Detailed Explanation:**
PUBLIC is an implicit group encompassing every user in the database — current *and* future. GRANT ... TO PUBLIC is a common source of unintentionally broad access.

**Why the Other Options Are Wrong:**
B. PUBLIC isn't something users "join" — it implicitly covers everyone, including users created later.
C. PUBLIC is a valid, commonly used keyword grantee.
D. Table owners already have full privileges on their own objects by default; irrelevant here.

**Concept Tested:** PUBLIC as an implicit grantee.

**Exam Trap:** Underestimating PUBLIC's scope — it isn't limited to existing users at grant time.

---

**Question 3**
Which statement correctly revokes SELECT on EMPLOYEES from user SCOTT?

A. `REVOKE SELECT ON employees FROM scott;`
B. `REVOKE SELECT FROM employees TO scott;`
C. `DENY SELECT ON employees TO scott;`
D. `REMOVE SELECT ON employees FROM scott;`

**Correct Answer:** A

**Detailed Explanation:**
Correct Oracle syntax is `REVOKE <privilege> ON <object> FROM <grantee>;`.

**Why the Other Options Are Wrong:**
B. Reverses/misuses the FROM/TO clauses.
C. DENY is SQL Server (T-SQL) syntax, not valid in Oracle.
D. REMOVE is not a DCL keyword.

**Concept Tested:** REVOKE syntax.

**Exam Trap:** Importing syntax from other RDBMS platforms that doesn't exist in Oracle.

---

### Subtopic 5: Transaction Control Language (COMMIT, SAVEPOINT, ROLLBACK)

**Question 1**
With autocommit off, you run:
```sql
INSERT INTO employees (...) VALUES (...);
CREATE INDEX idx_emp ON employees(last_name);
ROLLBACK;
```
What happens to the INSERT?

A. It's rolled back along with any effects of CREATE INDEX.
B. It's permanently committed and cannot be undone, because CREATE INDEX (a DDL statement) issues an implicit COMMIT before it runs.
C. It remains uncommitted — CREATE INDEX doesn't affect transaction state.
D. CREATE INDEX fails because of the pending uncommitted INSERT.

**Correct Answer:** B

**Detailed Explanation:**
Every DDL statement in Oracle is preceded (and followed) by an implicit COMMIT. By the time CREATE INDEX runs, the INSERT is already permanently committed — the later ROLLBACK has nothing left to undo for it.

**Why the Other Options Are Wrong:**
A. The INSERT was already committed before ROLLBACK ran.
C. DDL absolutely affects transaction state via implicit commit.
D. Oracle doesn't block DDL merely because of a pending same-session DML transaction.

**Concept Tested:** DDL causes an implicit COMMIT.

**Exam Trap:** Assuming ROLLBACK always undoes everything issued earlier in the session, ignoring intervening DDL.

---

**Question 2**
Evaluate:
```sql
UPDATE t SET col = 1;
SAVEPOINT sp1;
UPDATE t SET col = 2;
SAVEPOINT sp2;
UPDATE t SET col = 3;
ROLLBACK TO sp1;
```
What is col's value after `ROLLBACK TO sp1`?

A. 3
B. 2
C. 1
D. NULL, because ROLLBACK TO always clears the whole transaction

**Correct Answer:** C

**Detailed Explanation:**
`ROLLBACK TO SAVEPOINT` undoes everything after that savepoint, while preserving changes made before it and keeping the transaction open. sp1 was created right after col was set to 1, so both later updates (2 and 3) are undone.

**Why the Other Options Are Wrong:**
A. That's the value before the rollback.
B. That value was set at/after sp1 too, so it's undone along with the last update.
D. `ROLLBACK TO SAVEPOINT` doesn't end or clear the transaction — a plain ROLLBACK (no savepoint) does that.

**Concept Tested:** SAVEPOINT / ROLLBACK TO semantics.

**Exam Trap:** Confusing partial rollback (transaction stays open) with a full ROLLBACK (transaction ends).

---

**Question 3**
Session A runs `UPDATE employees SET salary = salary * 2 WHERE employee_id = 100;` but hasn't committed. Session B runs `SELECT salary FROM employees WHERE employee_id = 100;`. What does Session B see?

A. The doubled salary — Oracle makes changes visible to all sessions immediately.
B. The original salary — Oracle's read-consistency model doesn't expose uncommitted changes to other sessions.
C. An error — Session B is blocked from reading a row being updated.
D. NULL, because the row is locked.

**Correct Answer:** B

**Detailed Explanation:**
Oracle uses multi-version read consistency: readers are never blocked by writers, and uncommitted changes stay invisible to other sessions until COMMIT.

**Why the Other Options Are Wrong:**
A. Describes a dirty-read model, which Oracle explicitly avoids.
C. Oracle readers are never blocked by writers — a core concurrency feature.
D. A row lock from an uncommitted UPDATE doesn't block SELECT and doesn't produce NULL.

**Concept Tested:** Read consistency and its relationship to COMMIT.

**Exam Trap:** Assuming readers are blocked by writers, or that uncommitted data is visible cross-session.

---

**Question 4**
True or False: Issuing COMMIT releases all locks held by the current transaction.

A. True
B. False

**Correct Answer:** A (True)

**Detailed Explanation:**
COMMIT ends the transaction, makes its changes permanent and visible, and releases every row/table lock it was holding — part of why long uncommitted transactions cause contention.

**Concept Tested:** COMMIT's full set of effects (not just "saving data").

**Exam Trap:** Forgetting lock release is a defined effect of COMMIT.

---

**Question 5**
A user disconnects normally from SQL*Plus (types EXIT) after several uncommitted DML statements, with no explicit COMMIT/ROLLBACK. What happens to the pending changes?

A. Automatically rolled back — only explicit COMMIT makes changes permanent.
B. Automatically committed — a normal exit implicitly commits before disconnecting.
C. The session hangs until a DBA resolves it.
D. Oracle raises an error, preventing disconnection.

**Correct Answer:** B

**Detailed Explanation:**
A graceful, normal disconnection implicitly commits any pending transaction. An abnormal termination (client crash, killed session) instead triggers an implicit ROLLBACK — the opposite behavior.

**Why the Other Options Are Wrong:**
A. That's the abnormal-termination behavior, reversed.
C, D. Not real Oracle behavior.

**Concept Tested:** Implicit COMMIT vs. ROLLBACK triggers.

**Exam Trap:** Mixing up which disconnection type (normal vs. abnormal) triggers which outcome.

---

### Subtopic 6: Arithmetic Operators

**Question 1**
What does `SELECT 10 + 2 * 3 FROM dual;` return?

A. 36
B. 16
C. 32
D. Error — parentheses are required with mixed operators

**Correct Answer:** B

**Detailed Explanation:**
Multiplication/division are evaluated before addition/subtraction absent parentheses: `2*3=6`, then `10+6=16`.

**Why the Other Options Are Wrong:**
A. Results from wrongly evaluating strictly left-to-right ((10+2)*3).
D. Precedence applies automatically; parentheses aren't required.

**Concept Tested:** Arithmetic operator precedence.

**Exam Trap:** Evaluating left-to-right instead of applying standard precedence (the same trap behind the "Calculated Profit" question in your sample document).

---

**Question 2**
Which two expressions are valid in Oracle and produce a numeric (not date) result? **(Choose two)**

A. `hire_date - sysdate`
B. `hire_date + 30`
C. `hire_date + sysdate`
D. `sysdate - hire_date`

**Correct Answers:** A and D

**Detailed Explanation:**
date − date returns a number (the difference in days). Both A and D are date-minus-date expressions.

**Why the Other Options Are Wrong:**
B. date + number is valid, but the *result* is a DATE (shifted forward), not a number.
C. date + date is illegal in Oracle — raises ORA-00975.

**Concept Tested:** Date-arithmetic result types (date ± number = date; date − date = number; date + date = illegal).

**Exam Trap:** Mirrors the "find the odd one out" pattern in your sample set — remembering that date+date specifically is the disallowed combination.

---

**Question 3**
For a row where `COMMISSION_PCT IS NULL`, what is the result of `salary + salary * commission_pct`?

A. Equal to salary, since NULL is treated as 0 in arithmetic.
B. NULL — any arithmetic operation involving a NULL operand yields NULL.
C. An error — you can't do arithmetic with NULL.
D. Equal to salary * 2, since NULL defaults to 1 in multiplication.

**Correct Answer:** B

**Detailed Explanation:**
NULL represents an unknown value and propagates through arithmetic — any operation involving a NULL operand produces NULL. This is exactly why `NVL(commission_pct, 0)` is used before doing arithmetic on a nullable column.

**Why the Other Options Are Wrong:**
A, D. There is no implicit "NULL = 0" or "NULL = 1" substitution rule in arithmetic.
C. No error is raised; the expression cleanly evaluates to NULL.

**Concept Tested:** NULL propagation through arithmetic expressions.

**Exam Trap:** Assuming NULL behaves like zero (addition) or one (multiplication) — it does neither.

---

**Question 4**
What happens when you execute `SELECT 10 / 0 FROM dual;`?

A. Returns NULL.
B. Returns Infinity.
C. Raises ORA-01476: divisor is equal to zero.
D. Returns 0.

**Correct Answer:** C

**Detailed Explanation:**
Unlike NULL propagation, division by the literal 0 is a genuine error condition in Oracle SQL, not a silent NULL or infinite result.

**Why the Other Options Are Wrong:**
A. Confuses this with NULL-operand arithmetic rules — a separate case.
B. Oracle NUMBER arithmetic doesn't implement IEEE-style infinity.
D. The statement errors; it doesn't quietly return 0.

**Concept Tested:** Division-by-zero error vs. NULL-operand arithmetic.

**Exam Trap:** Conflating "NULL propagates silently" with "division by zero also fails silently" — these are two distinct rules often tested together.

---

### Subtopic 7: Comparison Operators

**Question 1**
`T2.DEPT_ID` contains (10, 20, NULL). Given:
`SELECT * FROM t1 WHERE dept_id NOT IN (SELECT dept_id FROM t2);`
(assume T1.DEPT_ID has no NULLs) — what is returned?

A. All T1 rows whose dept_id is not 10 and not 20.
B. No rows at all, regardless of T1's data.
C. All rows in T1 — NOT IN ignores NULLs in the subquery.
D. Only rows where dept_id IS NULL.

**Correct Answer:** B

**Detailed Explanation:**
`NOT IN` is internally a chain of `<>` comparisons ANDed together. Comparing any value to NULL yields UNKNOWN. Since the subquery's result set contains a NULL, at least one ANDed comparison is UNKNOWN for every candidate row, and an AND chain containing UNKNOWN (with no FALSE present) can never resolve to TRUE — so the whole condition never evaluates TRUE for any row, and the outer query returns **zero rows**, regardless of T1's contents. `IN` (an OR chain) doesn't have this problem, since OR with even one TRUE resolves the whole chain to TRUE.

**Why the Other Options Are Wrong:**
A. The intuitive but incorrect expectation — this ignores the NULL-poisoning effect.
C. The opposite is true: a NULL in the NOT IN list poisons the entire result rather than being ignored.
D. IS NULL isn't relevant here at all.

**Concept Tested:** Three-valued logic and NOT IN with a NULL in the comparison set.

**Exam Trap:** Arguably THE classic SQL certification trap — most candidates confidently pick option A without realizing a single NULL collapses the whole result to empty.

---

**Question 2**
SALARY values present: 2999, 3000, 4500, 5000, 5001. How many rows satisfy `WHERE salary BETWEEN 3000 AND 5000`?

A. 2
B. 3
C. 4
D. 5

**Correct Answer:** B

**Detailed Explanation:**
`BETWEEN x AND y` is inclusive of both boundaries — shorthand for `salary >= x AND salary <= y`. So 3000, 4500, and 5000 all qualify.

**Why the Other Options Are Wrong:**
A. Would result from wrongly treating BETWEEN as an exclusive (open) range.
C, D. Overcount by incorrectly including 2999 and/or 5001.

**Concept Tested:** BETWEEN...AND inclusivity.

**Exam Trap:** Treating BETWEEN like a mathematical open interval instead of a closed/inclusive one.

---

**Question 3**
You need to find PRODUCT_CODE values containing a **literal** underscore character, not the wildcard meaning "any single character." Which WHERE clause is correct?

A. `WHERE product_code LIKE '%_%'`
B. `WHERE product_code LIKE '%\_%' ESCAPE '\'`
C. `WHERE product_code LIKE '%_%' ESCAPE '_'`
D. `WHERE product_code = '_'`

**Correct Answer:** B

**Detailed Explanation:**
By default, `_` and `%` are wildcards. To match a literal underscore, you must designate an escape character (e.g., `\`) that precedes the underscore, telling Oracle to treat it literally: `LIKE '%\_%' ESCAPE '\'`.

**Why the Other Options Are Wrong:**
A. Here `_` still acts as a wildcard, not a literal character.
C. Defines `_` itself as the escape character, which would try to escape the following `%` instead — not the intended construction.
D. Only matches a code that is exactly a single underscore, not "contains an underscore."

**Concept Tested:** LIKE wildcard semantics and the ESCAPE clause.

**Exam Trap:** Forgetting that `_` and `%` need explicit escaping to be treated as literal characters.

---

**Question 4**
MANAGERS.SALARY values are (5000, 7000, 9000). Which correctly returns employees earning more than **every** manager (i.e., more than the highest)?

A. `WHERE salary > ANY (SELECT salary FROM managers)`
B. `WHERE salary > ALL (SELECT salary FROM managers)`
C. `WHERE salary > (SELECT MAX(salary) FROM managers)`
D. Both B and C are correct and logically equivalent.

**Correct Answer:** D

**Detailed Explanation:**
`> ALL (subquery)` requires exceeding every returned value — i.e., exceeding the maximum — which is logically identical to comparing against `MAX()` in a single-row subquery.

**Why the Other Options Are Wrong:**
A. `> ANY` only requires exceeding at least one value (effectively, exceeding the *minimum*) — a much weaker condition.

**Concept Tested:** ANY vs. ALL with subqueries.

**Exam Trap:** Confusing `> ANY` (weaker, beats the minimum) with `> ALL` (stronger, beats the maximum) — one of the most reliably tested subquery-comparison traps.

---

**Question 5**
Which of the following is **NOT** a valid "not equal to" operator in Oracle SQL?

A. `!=`
B. `<>`
C. `^=`
D. `=!`

**Correct Answer:** D

**Detailed Explanation:**
Oracle recognizes three interchangeable not-equal operators: `!=`, `<>`, and `^=`. `=!` is not valid syntax.

**Concept Tested:** Valid not-equal operator variants.

**Exam Trap:** Assuming operator symbols can be freely reordered — this tests precise memorization of the recognized forms.

---

### Subtopic 8: Logical Operators

**Question 1**
`SELECT * FROM employees WHERE job_id = 'SA_REP' OR job_id = 'ST_CLERK' AND salary > 3000;` — which rows are returned?

A. All SA_REP rows, plus ST_CLERK rows with salary > 3000.
B. All SA_REP rows with salary > 3000, plus all ST_CLERK rows.
C. Only rows that are SA_REP or ST_CLERK, all of which must also have salary > 3000.
D. A syntax error — AND and OR can't be mixed without parentheses.

**Correct Answer:** A

**Detailed Explanation:**
AND binds tighter than OR by default, so the condition is logically: `job_id = 'SA_REP' OR (job_id = 'ST_CLERK' AND salary > 3000)`. Every SA_REP row qualifies regardless of salary; ST_CLERK rows additionally need salary > 3000.

**Why the Other Options Are Wrong:**
B. Reverses which job_id the salary condition applies to.
C. Wrongly assumes salary > 3000 applies uniformly to both branches.
D. Mixing AND/OR without parentheses is syntactically legal — default precedence applies instead of erroring.

**Concept Tested:** Logical operator precedence (NOT > AND > OR).

**Exam Trap:** A "silent" precedence trap — the query is valid but easy to misread, and misjudging which condition AND binds to is very commonly tested.

---

**Question 2**
A row has `SALARY = NULL`. Does this row satisfy `NOT (salary > 3000)`?

A. Yes — NOT reverses the comparison, and NULL "isn't greater than," so it satisfies NOT.
B. No — `salary > 3000` evaluates to UNKNOWN when salary is NULL, and NOT UNKNOWN is still UNKNOWN, which is never TRUE — the row is excluded.
C. Yes — NULL is treated as 0, so 0 > 3000 is FALSE, and NOT FALSE is TRUE.
D. It raises an error — NOT cannot be applied to an UNKNOWN result.

**Correct Answer:** B

**Detailed Explanation:**
This tests three-valued logic. `salary > 3000` with NULL salary is UNKNOWN, not FALSE. NOT applied to UNKNOWN stays UNKNOWN. A WHERE clause only includes rows where the condition is definitively TRUE, so this row is silently excluded.

**Why the Other Options Are Wrong:**
A. NULL doesn't resolve to a definite result under NOT.
C. No implicit NULL-to-0 substitution exists in comparisons.
D. No error — the row is just excluded, silently.

**Concept Tested:** Three-valued logic — NOT applied to an UNKNOWN result.

**Exam Trap:** Assuming NOT always cleanly flips TRUE/FALSE, forgetting UNKNOWN is a fixed point under NOT (NOT UNKNOWN = UNKNOWN, never TRUE).

---

**Question 3**
You want **all** SA_REP or ST_CLERK employees to additionally require salary > 3000. Which WHERE clause correctly expresses this?

A. `WHERE job_id = 'SA_REP' OR job_id = 'ST_CLERK' AND salary > 3000`
B. `WHERE (job_id = 'SA_REP' OR job_id = 'ST_CLERK') AND salary > 3000`
C. `WHERE job_id = 'SA_REP' AND job_id = 'ST_CLERK' OR salary > 3000`
D. `WHERE job_id IN ('SA_REP', 'ST_CLERK') OR salary > 3000`

**Correct Answer:** B

**Detailed Explanation:**
To force the OR to be evaluated as a single unit before ANDing with the salary condition, explicit parentheses are required.

**Why the Other Options Are Wrong:**
A. Default precedence binds AND only to the ST_CLERK branch (as in Question 1 above).
C. No row can satisfy `job_id = 'SA_REP' AND job_id = 'ST_CLERK'` simultaneously, and `OR salary > 3000` is a completely different, much broader condition.
D. IN correctly replaces the OR chain, but `salary > 3000` is still OR'd rather than AND'd — changing the meaning entirely.

**Concept Tested:** Using parentheses to override default AND/OR precedence.

**Exam Trap:** Recognizing that parentheses are the *only* reliable way to force an OR condition to be evaluated as a whole before ANDing.

---

**Question 4**
Complete the three-valued logic: `TRUE AND UNKNOWN = ?`, `FALSE OR UNKNOWN = ?`

A. UNKNOWN, UNKNOWN
B. TRUE, FALSE
C. UNKNOWN, FALSE
D. FALSE, UNKNOWN

**Correct Answer:** A

**Detailed Explanation:**
AND returns the "weaker" operand (FALSE dominates); TRUE doesn't override UNKNOWN, so `TRUE AND UNKNOWN = UNKNOWN`. OR returns the "stronger" operand (TRUE dominates); FALSE doesn't override UNKNOWN, so `FALSE OR UNKNOWN = UNKNOWN`.

**Why the Other Options Are Wrong:**
B, C, D. Each assigns at least one incorrect definite result where UNKNOWN is actually correct.

**Concept Tested:** Full three-valued-logic truth tables for AND/OR with UNKNOWN.

**Exam Trap:** Assuming a definite operand always "resolves" an UNKNOWN expression — it only resolves when that definite value is the dominating one for that specific operator (FALSE for AND, TRUE for OR).

---

### Subtopic 9: Set Operators

**Question 1**
Table A: (1, 2, 2, 3). Table B: (2, 3, 4). What does `SELECT val FROM a UNION SELECT val FROM b;` return, vs. UNION ALL?

A. UNION returns (1,2,3,4) — 4 rows, duplicates eliminated across both sets; UNION ALL returns all 7 rows combined, no de-duplication.
B. Both return the same 7 rows; UNION ALL just sorts them.
C. UNION returns (1,2,2,3,3,4) — duplicates removed only within each table, not across both; UNION ALL returns 7 rows.
D. UNION returns 4 rows; UNION ALL also returns 4 rows, without sorting.

**Correct Answer:** A

**Detailed Explanation:**
UNION removes ALL duplicates across the entire combined output (distinct set = {1,2,3,4}) and Oracle applies an implicit sort. UNION ALL keeps every row from every source with zero de-duplication: 4 + 3 = 7 rows.

**Why the Other Options Are Wrong:**
B. UNION genuinely de-duplicates; it's not merely a sorted UNION ALL.
C. UNION eliminates duplicates across the whole combined set, not per-table.
D. UNION ALL doesn't de-duplicate at all, so it can't return only 4 rows here.

**Concept Tested:** UNION (implicit DISTINCT + sort) vs. UNION ALL (no de-duplication).

**Exam Trap:** Assuming UNION removes duplicates only "within" one query's result rather than across the combined output.

---

**Question 2**
Table A has one row with value NULL; Table B has one row with value NULL. What does `SELECT val FROM a INTERSECT SELECT val FROM b;` return?

A. Zero rows — NULL is never equal to NULL, even under INTERSECT.
B. One row containing NULL — set operators treat NULLs as matching for row-comparison purposes, unlike ordinary equality.
C. An error — INTERSECT cannot process NULLs.
D. Two rows, one NULL from each table.

**Correct Answer:** B

**Detailed Explanation:**
While `NULL = NULL` evaluates to UNKNOWN in an ordinary comparison, Oracle's set operators (and DISTINCT/GROUP BY) use a different rule for identifying matching/duplicate rows — they treat two NULLs as equivalent. So INTERSECT matches the NULL rows and returns it once.

**Why the Other Options Are Wrong:**
A. Correctly describes ordinary equality, but wrongly applies it to set-operator row-matching, which uses the NULL-equals-NULL rule instead.
C. No error — NULLs are fully supported.
D. INTERSECT returns the matching value once, not once per source table.

**Concept Tested:** NULL handling differences — comparison operators vs. set-operator/DISTINCT row-matching.

**Exam Trap:** Over-applying "NULL is never equal to NULL" to contexts (set operators, GROUP BY, DISTINCT) where Oracle actually treats NULLs as matching.

---

**Question 3**
Table A: (1,2,3,4). Table B: (3,4,5). What does `SELECT val FROM a MINUS SELECT val FROM b;` return?

A. (1,2)
B. (5)
C. (1,2,3,4,5)
D. (3,4)

**Correct Answer:** A

**Detailed Explanation:**
MINUS returns rows from the first query that don't appear anywhere in the second query's result. Removing (3,4,5) from (1,2,3,4) leaves (1,2).

**Why the Other Options Are Wrong:**
B. That's `B MINUS A` — the reversed order. MINUS is **not commutative**, unlike INTERSECT/UNION.
C. Describes UNION, not MINUS.
D. Describes INTERSECT (the common rows), not MINUS.

**Concept Tested:** MINUS and its non-commutativity.

**Exam Trap:** Forgetting that swapping the two queries in a MINUS produces a genuinely different result, not just a reordering.

---

**Question 4**
Which of these will cause an error when combined with UNION?

A. `SELECT employee_id, salary FROM employees UNION SELECT department_id, NULL FROM departments;`
B. `SELECT employee_id, salary FROM employees UNION SELECT department_id, budget FROM departments;`
C. `SELECT employee_id, TO_CHAR(salary) FROM employees UNION SELECT department_id, department_name FROM departments;`
D. `SELECT employee_id, hire_date FROM employees UNION SELECT department_id, salary FROM departments;`

**Correct Answer:** D

**Detailed Explanation:**
Set operators require the same column count and compatible data types in each corresponding position. In D, the second query's second column (salary, NUMBER) doesn't match the first query's second column (hire_date, DATE) — DATE and NUMBER aren't implicitly convertible here, causing a type-mismatch error.

**Why the Other Options Are Wrong:**
A. NULL is compatible with any data type in this position — no conflict.
B. Both second columns (salary/budget) are NUMBER — valid.
C. Both second columns (`TO_CHAR(salary)` and department_name) are character-compatible — valid.

**Concept Tested:** Set-operator column-count and positional data-type compatibility.

**Exam Trap:** Overlooking that matching column *count* isn't sufficient — positional data types must also be compatible, and a DATE-vs-NUMBER mismatch triggers an error even with equal column counts.

---

**Question 5**
Which statement about ORDER BY in a compound query using set operators is TRUE?

A. Each component SELECT can have its own ORDER BY.
B. ORDER BY can appear only once, at the very end of the compound statement, and refers to the column names/positions of the **first** SELECT.
C. ORDER BY is not permitted anywhere in a compound query using set operators.
D. ORDER BY must be repeated identically after each component SELECT.

**Correct Answer:** B

**Detailed Explanation:**
ORDER BY may appear only once, after the last SELECT, and applies to the whole combined result. Displayed column names/aliases come from the first SELECT, so ORDER BY references must correspond to that first query's column list.

**Why the Other Options Are Wrong:**
A. Individual component SELECTs cannot each carry their own ORDER BY.
C. ORDER BY is fully permitted — just restricted to a single occurrence.
D. No such repetition requirement exists (and would itself be a syntax error).

**Concept Tested:** ORDER BY placement and column-reference rules in compound (set-operator) queries.

**Exam Trap:** Trying to reference a column alias from the second (or later) query in the final ORDER BY — since output names come only from the first query, this fails.