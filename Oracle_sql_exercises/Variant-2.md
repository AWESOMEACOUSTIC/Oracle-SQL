# Oracle SQL Certification Practice — Topic 2
## Integrity Constraints

---

### Subtopic 1: Entity Integrity — PRIMARY KEY Constraint

**Question 1**
You execute:
```sql
ALTER TABLE employees ADD CONSTRAINT emp_id_pk PRIMARY KEY (employee_id);
ALTER TABLE employees ADD CONSTRAINT emp_id_pk2 PRIMARY KEY (email);
```
What happens on the second statement?

A. It succeeds — Oracle allows multiple PRIMARY KEY constraints per table as long as each is on a different column.
B. It fails with ORA-02260: table can have only one primary key.
C. It succeeds, but silently replaces the first primary key.
D. It fails only if EMAIL already contains duplicate values.

**Correct Answer:** B

**Detailed Explanation:**
A table may have exactly one PRIMARY KEY constraint, whether single-column or composite. Attempting to add a second one — regardless of which column(s) it targets or whether that column has clean data — is rejected immediately with ORA-02260, without even checking the data.

**Why the Other Options Are Wrong:**
A. Directly contradicts the one-PK-per-table rule.
C. Oracle never silently replaces an existing constraint.
D. The error fires before Oracle even validates the data — it's a structural rule, not a data-quality one.

**Concept Tested:** One PRIMARY KEY per table.

**Exam Trap:** Assuming "primary key" just means "a strongly unique+not-null column" and that you can define several — UNIQUE constraints can be multiple, PRIMARY KEY cannot.

---

**Question 2**
The ORDERS table is defined with:
```sql
CREATE TABLE orders (
  order_id NUMBER,
  order_line NUMBER,
  qty NUMBER,
  CONSTRAINT orders_pk PRIMARY KEY (order_id, order_line)
);
```
You execute: `INSERT INTO orders VALUES (100, NULL, 5);`
What happens?

A. Succeeds — only the combination of (order_id, order_line) needs to be non-null and unique, and a partial NULL is fine as long as the pair is unique.
B. Fails — every column that is part of a composite primary key is implicitly NOT NULL, individually, so ORDER_LINE cannot be NULL even though it's only part of the key.
C. Succeeds — NULL is treated as a unique placeholder value in composite keys.
D. Fails only if another row already has order_id = 100.

**Correct Answer:** B

**Detailed Explanation:**
Every column participating in a PRIMARY KEY — composite or not — is implicitly given a NOT NULL constraint. This is stricter than a plain composite UNIQUE constraint, where individual columns may still be nullable. Here, ORDER_LINE being NULL violates its implicit NOT NULL, and Oracle rejects the row before uniqueness is even considered.

**Why the Other Options Are Wrong:**
A, C. Both wrongly assume PK columns tolerate NULLs the way UNIQUE composite columns do.
D. Irrelevant — the failure is due to NULL, not a duplicate combination.

**Concept Tested:** Implicit NOT NULL on every column of a composite primary key.

**Exam Trap:** Confusing composite PRIMARY KEY rules with composite UNIQUE KEY rules — this is the single most tested PK-vs-UNIQUE distinction.

---

**Question 3**
Which two statements about adding a PRIMARY KEY constraint to an existing populated table are true? **(Choose two)**

A. If the target column(s) already contain duplicate values, the ALTER TABLE ... ADD CONSTRAINT statement fails.
B. If the target column already contains NULLs, the statement still succeeds, and existing NULLs are automatically converted to 0.
C. Oracle automatically creates a unique index to support the constraint, if no suitable index already exists.
D. Oracle requires you to manually create a unique index first — CREATE TABLE ... ADD CONSTRAINT PRIMARY KEY never creates one implicitly.

**Correct Answers:** A and C

**Detailed Explanation:**
Adding a PK constraint validates all existing rows: duplicates or NULLs in the target column(s) cause the ALTER TABLE to fail outright. If validation passes, Oracle automatically builds a unique index (unless one already exists that can be reused) to enforce the constraint efficiently going forward.

**Why the Other Options Are Wrong:**
B. Oracle never silently converts data to satisfy a constraint — the statement simply fails if NULLs exist.
D. The opposite is true; automatic index creation is one of Oracle's defining PK behaviors, contrasted with FOREIGN KEY, which does *not* get this treatment (see Subtopic 3).

**Concept Tested:** Constraint validation against existing data; automatic supporting index creation.

**Exam Trap:** Assuming Oracle "fixes" bad data to accommodate a new constraint, rather than rejecting the DDL.

---

**Question 4**
True or False: A PRIMARY KEY constraint can be defined only at the table level (out-of-line), never inline at the column level.

A. True
B. False

**Correct Answer:** B (False)

**Detailed Explanation:**
A single-column PK can be defined inline at the column level (`employee_id NUMBER PRIMARY KEY`) or out-of-line at the table level. The table-level (out-of-line) syntax is *mandatory* only when the key is composite (spans multiple columns), since column-level syntax has no way to reference more than one column.

**Concept Tested:** Column-level vs. table-level constraint syntax.

**Exam Trap:** Overgeneralizing the "composite keys need table-level syntax" rule to *all* primary keys, including single-column ones.

---

**Question 5**
You run `ALTER TABLE employees DROP PRIMARY KEY;` on a table whose primary key is referenced by a FOREIGN KEY in another table. What happens?

A. It succeeds unconditionally; foreign keys never block dropping a referenced primary key.
B. It fails with ORA-02273 (or similar dependency error) unless CASCADE is specified.
C. It succeeds, and the dependent FOREIGN KEY constraint is automatically converted to a CHECK constraint.
D. It succeeds, and the dependent child table is automatically dropped.

**Correct Answer:** B

**Detailed Explanation:**
Oracle protects referential integrity: you cannot drop a PRIMARY KEY (or UNIQUE constraint) that a FOREIGN KEY still depends on, unless you explicitly add `CASCADE` to the DROP statement, which also drops the dependent foreign key constraint(s).

**Why the Other Options Are Wrong:**
A. Directly contradicts referential-integrity protection.
C. No such automatic conversion exists.
D. CASCADE drops the *dependent constraint*, never the table itself.

**Concept Tested:** CASCADE requirement when dropping a referenced key.

**Exam Trap:** Forgetting that dependency protection applies to DROP, not just runtime DML — many candidates only think of FK enforcement in terms of INSERT/DELETE on data rows.

---

### Subtopic 2: Sequence Generators

**Question 1**
```sql
CREATE SEQUENCE order_seq START WITH 100 INCREMENT BY 10 NOCACHE;
```
In a fresh session, you run:
```sql
SELECT order_seq.CURRVAL FROM dual;
```
What happens?

A. Returns 100, the starting value.
B. Returns NULL, since CURRVAL hasn't been initialized.
C. Raises ORA-08002: sequence ORDER_SEQ.CURRVAL is not yet defined in this session.
D. Returns 0 by default.

**Correct Answer:** C

**Detailed Explanation:**
CURRVAL returns "the value NEXTVAL most recently returned in this session." If NEXTVAL has never been called in the current session, there is no such value yet — Oracle raises ORA-08002 rather than assuming or defaulting to the sequence's START WITH value.

**Why the Other Options Are Wrong:**
A. START WITH only affects the *first* value NEXTVAL will produce — it is never implicitly "already current."
B, D. Oracle raises a hard error here, not a soft NULL/0 fallback.

**Concept Tested:** CURRVAL requires a prior NEXTVAL call within the same session.

**Exam Trap:** Assuming CURRVAL can be queried standalone to "peek" at the sequence's state without first calling NEXTVAL.

---

**Question 2**
A table has a trigger populating its PK from `order_seq.NEXTVAL` on every insert. A session performs:
```sql
INSERT INTO orders (order_id, ...) VALUES (order_seq.NEXTVAL, ...);
ROLLBACK;
INSERT INTO orders (order_id, ...) VALUES (order_seq.NEXTVAL, ...);
COMMIT;
```
Relative to a hypothetical scenario with no ROLLBACK, what is true about the sequence values used?

A. The rolled-back INSERT's sequence value is returned to the pool and reused by the second INSERT.
B. The second INSERT consumes a strictly higher value than the first — the rolled-back value is permanently lost, creating a gap.
C. ROLLBACK has no effect on sequences at all; both inserts use consecutive values regardless of order.
D. The sequence resets to START WITH after any ROLLBACK.

**Correct Answer:** B

**Detailed Explanation:**
NEXTVAL increments and returns a value independently of transaction control. Sequence generation is **not transactional** — it is not undone by ROLLBACK. Once a value is dispensed, it is gone, whether or not the surrounding DML commits. This is precisely why sequence-generated keys are *unique* but not guaranteed *gap-free*.

**Why the Other Options Are Wrong:**
A. Sequence values are never "returned" once issued.
C. Understates the actual effect — a gap genuinely results (option B is more precise than "no effect").
D. Only dropping/recreating (or specific ALTER SEQUENCE / RESTART operations) resets a sequence — ROLLBACK never does.

**Concept Tested:** Sequences are non-transactional and can produce gaps.

**Exam Trap:** Assuming ROLLBACK "undoes" everything touched in the transaction, including sequence state — one of the most common sequence-related misconceptions.

---

**Question 3**
Which two are valid uses of `order_seq.NEXTVAL`? **(Choose two)**

A. `INSERT INTO orders (order_id) VALUES (order_seq.NEXTVAL);`
B. `SELECT order_id FROM orders WHERE order_id = order_seq.NEXTVAL;`
C. `UPDATE orders SET order_id = order_seq.NEXTVAL WHERE order_id = 100;`
D. `CREATE VIEW v AS SELECT order_seq.NEXTVAL FROM dual;`

**Correct Answers:** A and C

**Detailed Explanation:**
NEXTVAL/CURRVAL are permitted in the VALUES clause of INSERT and the SET clause of UPDATE. They are explicitly disallowed in a view's defining query, and (per Oracle's documented restrictions) in the WHERE clause of a SELECT statement.

**Why the Other Options Are Wrong:**
B. Using NEXTVAL inside a WHERE clause is one of Oracle's explicitly documented restrictions.
D. A view's query definition cannot reference NEXTVAL/CURRVAL.

**Concept Tested:** Documented restrictions on where CURRVAL/NEXTVAL may legally appear.

**Exam Trap:** Assuming NEXTVAL is a generic expression usable anywhere a number is valid — several specific clauses (WHERE, subqueries, DISTINCT/GROUP BY/ORDER BY queries, views, CHECK constraints, column DEFAULTs) explicitly forbid it.

---

**Question 4**
You need to reset a sequence's next value back down to 1 (e.g., for a test environment refresh). Which approach is correct?

A. `ALTER SEQUENCE order_seq START WITH 1;`
B. You must DROP and re-CREATE the sequence, since ALTER SEQUENCE cannot change START WITH.
C. `ALTER SEQUENCE order_seq RESET TO 1;`
D. `UPDATE order_seq SET nextval = 1;`

**Correct Answer:** B

**Detailed Explanation:**
ALTER SEQUENCE can change INCREMENT BY, MAXVALUE/MINVALUE, CYCLE/NOCYCLE, and CACHE/NOCACHE — but **not** START WITH. START WITH only has meaning at creation time. To truly "restart" a sequence, you drop and recreate it (some Oracle versions also expose a RESTART clause on ALTER SEQUENCE, but the historically correct, universally safe cert answer is drop-and-recreate).

**Why the Other Options Are Wrong:**
A. Not valid — START WITH is not a legal clause of ALTER SEQUENCE.
C. RESET TO is not valid legacy ALTER SEQUENCE syntax tested at this level.
D. Sequences are not tables; you cannot UPDATE a sequence with DML.

**Concept Tested:** What ALTER SEQUENCE can and cannot modify.

**Exam Trap:** Assuming every CREATE SEQUENCE clause is also freely alterable — START WITH is the standout exception.

---

**Question 5**
Which statement about sequence CACHE is TRUE?

A. CACHE pre-allocates and holds a set of sequence values in memory for fast access, but any cached-and-unused values are lost if the instance shuts down abnormally, creating gaps.
B. NOCACHE guarantees no gaps will ever occur in the sequence.
C. CACHE values, once generated, are always used in the exact order requested across all sessions.
D. CACHE has no effect on performance; it only affects storage.

**Correct Answer:** A

**Detailed Explanation:**
CACHE (default 20) pre-generates and holds a block of sequence values in the SGA for fast dispensing without hitting the data dictionary on every call. If the instance crashes or shuts down without using all cached values, those values are permanently lost — a classic, well-documented source of sequence gaps.

**Why the Other Options Are Wrong:**
B. NOCACHE reduces (does not eliminate) one gap *source*; gaps can still occur from rollbacks or multiple concurrent sessions.
C. Under RAC or high concurrency, cached ranges can be dispensed to different sessions in ways that don't guarantee strict global ordering.
D. CACHE is fundamentally a performance feature (fewer dictionary updates), which is exactly the tradeoff against gap risk.

**Concept Tested:** CACHE/NOCACHE tradeoffs and gap sources.

**Exam Trap:** Believing any sequence configuration can *guarantee* zero gaps — Oracle sequences are explicitly documented as not gap-free under any setting.

---

### Subtopic 3: Referential Integrity — FOREIGN KEY Constraint

**Question 1**
The EMPLOYEES table has `DEPARTMENT_ID` as a FOREIGN KEY referencing `DEPARTMENTS(DEPARTMENT_ID)`. Which two statements are true? **(Choose two)**

A. You can insert an employee row with `department_id = NULL`, even though the FK constraint exists.
B. Multiple employee rows can each have `department_id = NULL` without violating the FK constraint.
C. A FOREIGN KEY constraint implicitly makes its column NOT NULL, just like a PRIMARY KEY does.
D. Every DEPARTMENT_ID value inserted into EMPLOYEES must already exist in DEPARTMENTS, with no exceptions.

**Correct Answers:** A and B

**Detailed Explanation:**
A FOREIGN KEY constraint, by itself, only restricts *non-null* values to those existing in the referenced key — it says nothing about nullability. NULL is a valid FK value (meaning "this row has no associated parent yet"), and unlike UNIQUE/PK columns, there's no ambiguity here: any number of child rows can independently hold NULL.

**Why the Other Options Are Wrong:**
C. FK does not imply NOT NULL — that's purely a PRIMARY KEY behavior, and must be added separately via a distinct NOT NULL constraint if desired.
D. False as stated — NULL is the explicit exception; the rule only applies to non-null values.

**Concept Tested:** FOREIGN KEY does not enforce NOT NULL; NULL FK values are always permitted absent an explicit separate NOT NULL constraint.

**Exam Trap:** Assuming FK and PK share the same nullability rules, when in fact this is one of their key differences.

---

**Question 2**
DEPARTMENTS(DEPARTMENT_ID=10) currently has employees assigned to it. You run:
`DELETE FROM departments WHERE department_id = 10;`
with no ON DELETE clause specified on the child's FK constraint. What happens?

A. The DELETE succeeds, and all matching EMPLOYEES rows have their department_id set to NULL.
B. The DELETE succeeds, and all matching EMPLOYEES rows are deleted too.
C. The DELETE fails with ORA-02292: integrity constraint violated — child record found.
D. The DELETE succeeds silently, leaving orphaned EMPLOYEES rows pointing to a nonexistent department.

**Correct Answer:** C

**Detailed Explanation:**
The default FK behavior (no ON DELETE clause) is effectively "restrict": Oracle refuses to delete a parent row that still has dependent child rows, raising ORA-02292. Cascading behaviors (A: ON DELETE SET NULL, B: ON DELETE CASCADE) must be explicitly declared at constraint-creation time.

**Why the Other Options Are Wrong:**
A, B. Both describe legitimate *optional* FK behaviors, but neither is the default.
D. Oracle never silently permits orphaned FK references — the entire point of the constraint is to prevent this.

**Concept Tested:** Default (restrict) FK delete behavior vs. explicit ON DELETE CASCADE / SET NULL.

**Exam Trap:** Assuming "referential integrity" automatically means cascading cleanup — by default it means the opposite: blocking the operation.

---

**Question 3**
Which statement about indexing and FOREIGN KEY constraints is TRUE?

A. Oracle automatically creates an index on the foreign key column(s), exactly as it does for PRIMARY KEY.
B. Oracle does NOT automatically create an index on foreign key columns; DBAs typically add one manually to avoid table-level locking issues on the child table during certain parent DML.
C. Foreign key columns cannot be indexed at all.
D. An index on a foreign key column is required by Oracle before the constraint can even be created.

**Correct Answer:** B

**Detailed Explanation:**
Unlike PRIMARY KEY and UNIQUE constraints, Oracle does **not** automatically build a supporting index for a FOREIGN KEY. Without one, certain operations on the parent table (e.g., deleting or updating the referenced key) can escalate to a full table lock on the child table to guard against inconsistent reads — a well-known reason DBAs are advised to manually index FK columns.

**Why the Other Options Are Wrong:**
A. This is precisely the PRIMARY KEY behavior that FOREIGN KEY does *not* share.
C. FK columns can absolutely be indexed manually — it's just not automatic.
D. No such prerequisite exists; the constraint can be created with zero supporting indexes.

**Concept Tested:** Asymmetry between PK/UNIQUE (auto-indexed) and FK (never auto-indexed).

**Exam Trap:** Assuming all key-type constraints get the same automatic-index treatment as PRIMARY KEY.

---

**Question 4**
Table EMPLOYEES has a self-referencing FK: `manager_id` references `employees(employee_id)`. Which statement is TRUE?

A. Self-referencing foreign keys are illegal in Oracle.
B. Self-referencing foreign keys are legal and commonly used to model hierarchical relationships (e.g., an employee's manager, also an employee).
C. A self-referencing FK forces `manager_id` to differ from `employee_id` on every row automatically.
D. Self-referencing FKs can only be created via a database trigger, never via a native constraint.

**Correct Answer:** B

**Detailed Explanation:**
A FOREIGN KEY may reference a PRIMARY KEY/UNIQUE key within the *same* table. This is a standard technique for hierarchical data (org charts, category trees) and is fully supported as a native declarative constraint.

**Why the Other Options Are Wrong:**
A, D. Both incorrectly claim self-referencing FKs need special/non-native handling.
C. Nothing prevents `manager_id = employee_id` (a self-managing row) unless a separate CHECK constraint is added to forbid it.

**Concept Tested:** Self-referencing FOREIGN KEY constraints.

**Exam Trap:** Assuming FK relationships must always span two distinct tables.

---

**Question 5**
A composite FOREIGN KEY is defined as `FOREIGN KEY (col_a, col_b) REFERENCES parent(pk_a, pk_b)`. A child row is inserted with `col_a = 5, col_b = NULL`. What happens?

A. The insert fails — at least one FK column is NULL, so the row cannot be validated and is rejected outright.
B. The insert succeeds — if *any* column of a composite FK is NULL, Oracle does not attempt to match the row against the parent table at all (by default, "MATCH NONE"-style behavior), so the constraint is trivially satisfied.
C. The insert succeeds only if a parent row exists with `pk_a = 5` and `pk_b IS NULL`.
D. The insert fails unless ALL columns of the composite FK are NULL.

**Correct Answer:** B

**Detailed Explanation:**
Oracle's default (and only supported) match style for composite foreign keys is effectively "MATCH NONE" behavior: if *any* one of the FK's columns is NULL, the entire FK check is skipped for that row — Oracle does not require the remaining non-null column(s) to match a parent row. This is a subtler variant of the "FK doesn't require NOT NULL" rule from Question 1, extended to composite keys.

**Why the Other Options Are Wrong:**
A. Contradicts documented behavior — a partial NULL does not block the insert.
C. This describes a stricter "MATCH PARTIAL" standard-SQL behavior that Oracle does not implement for FKs.
D. Overstates the NULL requirement — only one column needs to be NULL to skip the check entirely.

**Concept Tested:** Composite FOREIGN KEY NULL-handling ("MATCH NONE" semantics).

**Exam Trap:** Assuming partial NULLs in a composite FK are treated the same way as, e.g., a composite CHECK constraint — Oracle skips the entire check rather than partially validating.

---

### Subtopic 4: Domain Integrity — NOT NULL Constraint

**Question 1**
Which statement about NOT NULL constraints is TRUE?

A. NOT NULL can be defined at either the column level or the table level, identically to PRIMARY KEY.
B. NOT NULL can only be defined at the column level — there is no table-level (out-of-line) syntax for it.
C. NOT NULL is the default for every column unless explicitly overridden with NULL.
D. NOT NULL constraints cannot be named.

**Correct Answer:** B

**Detailed Explanation:**
Unlike PRIMARY KEY, UNIQUE, FOREIGN KEY, and CHECK — which all support both column-level and table-level (out-of-line) definitions — NOT NULL is exclusively a column-level constraint in Oracle's syntax; there's no `CONSTRAINT ... NOT NULL (col)` table-level form.

**Why the Other Options Are Wrong:**
A. Overstates the syntax flexibility NOT NULL actually has.
C. Reversed — columns are nullable by default; NOT NULL must be explicitly declared.
D. NOT NULL constraints can absolutely be explicitly named (`CONSTRAINT emp_lname_nn NOT NULL`), just like any other constraint.

**Concept Tested:** Column-level-only syntax restriction unique to NOT NULL.

**Exam Trap:** Assuming every constraint type shares the same column-level/table-level flexibility.

---

**Question 2**
A column `PHONE` currently contains some NULL values. You run:
`ALTER TABLE employees MODIFY phone NOT NULL;`
What happens?

A. Succeeds — all existing NULLs are automatically replaced with a default placeholder.
B. Fails, because existing rows already violate the constraint being added — you must first UPDATE those rows to non-null values.
C. Succeeds, but only future inserts are checked; existing NULL rows remain untouched and compliant.
D. Fails only if PHONE is also part of a PRIMARY KEY.

**Correct Answer:** B

**Detailed Explanation:**
Like every other constraint, adding NOT NULL to an existing populated column validates all current rows immediately. If NULLs already exist, the ALTER TABLE fails until those rows are corrected (typically via UPDATE ... SET phone = <value> WHERE phone IS NULL) beforehand.

**Why the Other Options Are Wrong:**
A. Oracle never auto-populates data to satisfy a new constraint.
C. Describes an ENABLE NOVALIDATE-style state, which is not what a normal ALTER TABLE ... MODIFY does by default (see Subtopic 8).
D. The PK status of the column is irrelevant to this particular check.

**Concept Tested:** Existing-data validation when adding NOT NULL.

**Exam Trap:** Assuming a "MODIFY" operation is somehow more lenient with existing bad data than an "ADD CONSTRAINT" operation — both validate identically.

---

**Question 3**
Which of the following can cause a NOT NULL constraint to be violated at runtime?

A. `INSERT INTO t (col1) VALUES (NULL);` where col1 has NOT NULL.
B. `UPDATE t SET col1 = NULL WHERE ...;` where col1 has NOT NULL.
C. Omitting col1 entirely from an INSERT's column list, when col1 has NOT NULL and no DEFAULT.
D. All of the above.

**Correct Answer:** D

**Detailed Explanation:**
All three paths ultimately attempt to store a NULL in a NOT NULL column: an explicit NULL literal, an UPDATE that clears the value, and an omitted column with no DEFAULT (which implicitly attempts NULL). Each independently raises ORA-01400 (cannot insert NULL).

**Concept Tested:** All the different DML paths that can trigger a NOT NULL violation.

**Exam Trap:** Focusing only on explicit `NULL` literals and forgetting that *omission* (with no DEFAULT) is functionally equivalent to inserting NULL.

---

**Question 4**
A column is defined as `hire_date DATE DEFAULT SYSDATE NOT NULL`. You execute:
`INSERT INTO employees (employee_id) VALUES (500);`
(HIRE_DATE omitted from the column list.) What happens?

A. Fails — HIRE_DATE was omitted and is NOT NULL, so it's treated as an attempted NULL insert.
B. Succeeds — since HIRE_DATE has a DEFAULT, the omitted column is populated with SYSDATE, which satisfies NOT NULL.
C. Fails, because DEFAULT and NOT NULL cannot be combined on the same column.
D. Succeeds, but HIRE_DATE is stored as NULL until the next UPDATE.

**Correct Answer:** B

**Detailed Explanation:**
When a column is omitted from an INSERT's column list and has a DEFAULT clause, Oracle substitutes the default value instead of attempting NULL. Since SYSDATE is never NULL, the NOT NULL constraint is satisfied transparently.

**Why the Other Options Are Wrong:**
A. Only true if there were *no* DEFAULT — this is exactly why DEFAULT changes the outcome.
C. DEFAULT and NOT NULL are a completely standard, common combination.
D. There is no such deferred-population behavior.

**Concept Tested:** Interaction between DEFAULT and NOT NULL on omitted columns.

**Exam Trap:** Reflexively applying the "omitted column = NULL = NOT NULL violation" rule from the previous question without checking whether a DEFAULT clause changes the outcome.

---

### Subtopic 5: Domain Integrity — UNIQUE KEY Constraint

**Question 1**
A column `EMAIL` has a UNIQUE constraint (not PRIMARY KEY). Which statement is TRUE?

A. Only one row in the table may have `email IS NULL`.
B. Any number of rows may have `email IS NULL` without violating the UNIQUE constraint, because NULL is never considered equal to another NULL for uniqueness purposes.
C. NULL is disallowed entirely in a UNIQUE column, identical to PRIMARY KEY.
D. The first row with a NULL email is accepted; every subsequent NULL is rejected as a duplicate.

**Correct Answer:** B

**Detailed Explanation:**
This is the single biggest UNIQUE-vs-PRIMARY-KEY distinction: UNIQUE does **not** imply NOT NULL. Since `NULL = NULL` is UNKNOWN rather than TRUE, Oracle never flags two NULLs as duplicates of each other — so an unlimited number of rows may hold NULL in a UNIQUE column.

**Why the Other Options Are Wrong:**
A, D. Both wrongly impose a "one NULL only" limit that doesn't exist for UNIQUE.
C. Directly contradicts the defining difference between UNIQUE and PRIMARY KEY.

**Concept Tested:** UNIQUE constraint's tolerance for multiple NULLs.

**Exam Trap:** Treating UNIQUE and PRIMARY KEY as functionally interchangeable — this is the most heavily tested single fact in this whole topic.

---

**Question 2**
A composite UNIQUE constraint spans `(last_name, department_id)`. Existing row: `('Smith', 10)`. You attempt to insert `('Smith', NULL)`. What happens?

A. Fails — 'Smith' already exists, so the combination is treated as a duplicate regardless of the NULL.
B. Succeeds — since one of the composite key's columns is NULL, Oracle does not check this row for duplication against any other row at all, even though LAST_NAME matches an existing value.
C. Succeeds only if no other row already has `department_id IS NULL`.
D. Fails, because NULL is never permitted in any column that is part of a UNIQUE constraint.

**Correct Answer:** B

**Detailed Explanation:**
For a composite UNIQUE (or PK-adjacent domain) constraint, if *any* column in the key is NULL for a given row, Oracle treats uniqueness as automatically satisfied for that row — it is not compared against other rows at all, even ones sharing the same non-null values elsewhere in the key.

**Why the Other Options Are Wrong:**
A. Wrongly assumes partial matching still triggers a duplicate check.
C. Introduces a "one NULL per column" restriction that doesn't apply to UNIQUE at the composite level either.
D. Overstates NULL prohibition — only PRIMARY KEY imposes blanket NOT NULL; UNIQUE does not.

**Concept Tested:** Composite UNIQUE constraint NULL-handling.

**Exam Trap:** Assuming a composite key partially "reuses" the non-null portion for comparison — Oracle's rule is all-or-nothing: any NULL exempts the row from comparison entirely.

---

**Question 3**
Which statement about the index backing a UNIQUE constraint is TRUE?

A. Oracle never creates an index to support a UNIQUE constraint; uniqueness is checked via a full table scan on each DML.
B. Oracle automatically creates a unique index to enforce the constraint, exactly as it does for PRIMARY KEY, unless a suitable index already exists.
C. A UNIQUE constraint can only be enforced if the DBA manually pre-creates the index before adding the constraint.
D. UNIQUE constraints share a single global index across the entire database.

**Correct Answer:** B

**Detailed Explanation:**
Just like PRIMARY KEY, adding a UNIQUE constraint causes Oracle to automatically create a supporting unique index (or reuse an existing compatible one) — this is one place where UNIQUE and PRIMARY KEY behave identically, in contrast to FOREIGN KEY, which gets no automatic index at all.

**Why the Other Options Are Wrong:**
A. Full table scans for every DML would be prohibitively expensive; Oracle uses an index specifically to avoid this.
C. Manual pre-creation is optional (and can be done to control index name/tablespace), not mandatory.
D. Each UNIQUE constraint gets its own index scoped to that table/column(s), never a shared global structure.

**Concept Tested:** Automatic index creation shared by PRIMARY KEY and UNIQUE (contrasted with FOREIGN KEY in Subtopic 3).

**Exam Trap:** Extending the "FK gets no automatic index" rule incorrectly to UNIQUE constraints as well.

---

**Question 4**
Can a single column have both a UNIQUE constraint and be part of a separate composite PRIMARY KEY?

A. No — a column can only participate in one constraint of any kind.
B. Yes — a column may simultaneously be constrained by multiple, independent constraints (e.g., part of a composite PK and separately declared UNIQUE on its own).
C. Yes, but only if the UNIQUE constraint is defined before the PRIMARY KEY.
D. No — UNIQUE and PRIMARY KEY are mutually exclusive constraint types system-wide.

**Correct Answer:** B

**Detailed Explanation:**
Constraints are independent metadata objects; a column can be governed by several simultaneously (e.g., NOT NULL + CHECK + participating in a composite PK + also individually UNIQUE), as long as each constraint's own rule is logically satisfiable together.

**Why the Other Options Are Wrong:**
A, D. Both invent restrictions that don't exist — Oracle allows multiple constraints stacking on the same column.
C. Constraint declaration order has no bearing on whether they can coexist.

**Concept Tested:** Multiple, independent constraints can apply to the same column.

**Exam Trap:** Assuming constraints are mutually exclusive "slots" rather than independently stackable rules.

---

### Subtopic 6: Domain Integrity — CHECK Constraint

**Question 1**
Which of the following is a valid CHECK constraint in Oracle?

A. `CONSTRAINT chk_sal CHECK (salary > (SELECT AVG(salary) FROM employees))`
B. `CONSTRAINT chk_hire CHECK (hire_date <= SYSDATE)`
C. `CONSTRAINT chk_valid CHECK (created_by = USER)`
D. `CONSTRAINT chk_id CHECK (employee_id = employees.employee_id)` referencing another row

**Correct Answer:** None of A–D is actually legal — see explanation.

**Detailed Explanation:**
Oracle explicitly disallows CHECK constraints from referencing: subqueries (rules out A), the pseudocolumns/functions SYSDATE, UID, USER, USERENV, and CURRVAL/NEXTVAL (rules out B and C), and any other row or table (rules out D — a CHECK constraint may only examine columns *within the same row* it's validating). The realistic certification-correct takeaway: the only safe CHECK constraints reference only the row's own column values and constant literals/expressions (e.g., `CHECK (salary > 0)`, `CHECK (commission_pct BETWEEN 0 AND 1)`).

*(This question is deliberately built so every option fails a different documented CHECK restriction — subquery, non-deterministic date function, pseudocolumn, and cross-row reference, respectively — to drill all four restrictions in one place.)*

**Concept Tested:** The full set of CHECK constraint restrictions: no subqueries, no SYSDATE/UID/USER/USERENV, no NEXTVAL/CURRVAL, no cross-row or cross-table references.

**Exam Trap:** Assuming at least one "obviously reasonable-looking" business rule (like a hire-date-not-in-future check using SYSDATE) is automatically fair game — several very natural-seeming CHECK ideas are specifically the ones Oracle forbids.

---

**Question 2**
A column has `CONSTRAINT chk_comm CHECK (commission_pct > 0)`. You attempt:
`INSERT INTO employees (employee_id, commission_pct) VALUES (501, NULL);`
What happens?

A. Fails — NULL is not greater than 0, so the CHECK condition is FALSE.
B. Succeeds — the CHECK condition evaluates to UNKNOWN (not FALSE) for a NULL operand, and Oracle accepts any row where the CHECK condition is TRUE or UNKNOWN, rejecting only rows that evaluate to FALSE.
C. Fails, because CHECK constraints implicitly forbid NULL in the columns they reference.
D. Succeeds, but only if a separate NOT NULL constraint is absent — otherwise it's ambiguous which constraint fires.

**Correct Answer:** B

**Detailed Explanation:**
This mirrors the three-valued-logic theme from Topic 1: `NULL > 0` is UNKNOWN, not FALSE. Oracle's rule for CHECK constraints is that a row is rejected *only* when the condition evaluates to FALSE — UNKNOWN is treated as passing. If you want to also forbid NULL, you must add an explicit separate NOT NULL constraint.

**Why the Other Options Are Wrong:**
A. Conflates UNKNOWN with FALSE — exactly the trap being tested.
C. No such implicit NOT NULL behavior exists on CHECK.
D. There's no "ambiguity" — CHECK and NOT NULL are independent constraints that can coexist and are each evaluated on their own terms.

**Concept Tested:** Three-valued logic applied specifically to CHECK constraint evaluation.

**Exam Trap:** Forgetting that CHECK constraints follow the same UNKNOWN ≠ FALSE rule as WHERE clauses — a NULL silently "passes" a CHECK unless NOT NULL is separately enforced.

---

**Question 3**
Can a single column have more than one CHECK constraint applied to it?

A. No — only one CHECK constraint is allowed per column.
B. Yes — multiple CHECK constraints can be defined on the same column (or even the same table), and all of them must independently evaluate to TRUE or UNKNOWN for a row to be accepted.
C. Yes, but only if they are combined into a single constraint using AND.
D. No — additional CHECK conditions must be added by modifying the original constraint's definition.

**Correct Answer:** B

**Detailed Explanation:**
There's no limit on the number of CHECK constraints per column or table; Oracle evaluates every applicable CHECK independently on each DML operation, and a row must pass all of them.

**Why the Other Options Are Wrong:**
A, D. Both invent a one-CHECK-per-column limitation that doesn't exist.
C. Combining into one constraint via AND is *possible* but not *required* — multiple independent CHECK constraints work identically.

**Concept Tested:** Multiple CHECK constraints coexisting on the same column.

**Exam Trap:** Assuming CHECK behaves like NOT NULL/PRIMARY KEY (implicitly singular) rather than allowing arbitrary stacking.

---

**Question 4**
Which CHECK constraint definition will Oracle reject at creation time?

A. `CHECK (salary BETWEEN 1000 AND 50000)`
B. `CHECK (job_id IN ('IT_PROG', 'SA_REP', 'ST_CLERK'))`
C. `CHECK (department_id = (SELECT department_id FROM departments WHERE location_id = 1700))`
D. `CHECK (salary * 12 <= 600000)`

**Correct Answer:** C

**Detailed Explanation:**
CHECK constraints cannot contain a subquery of any kind — the condition must be evaluable using only the current row's own column values and constants/expressions. Option C fails immediately at DDL time (not at DML time) because of the embedded SELECT.

**Why the Other Options Are Wrong:**
A, B, D. All reference only the row's own columns combined with literal values/expressions — fully legal CHECK conditions.

**Concept Tested:** No-subqueries rule for CHECK constraints.

**Exam Trap:** Assuming a subquery-based "business rule" (referencing another table for validation) is simply enforced by CHECK the way a business analyst might expect — this specific case requires a trigger instead (see Subtopic 7).

---

**Question 5**
True or False: A CHECK constraint can reference multiple columns of the same row, such as `CHECK (list_price > cost)`.

A. True
B. False

**Correct Answer:** A (True)

**Detailed Explanation:**
The "same-row-only" restriction refers to not referencing *other rows or other tables* — referencing multiple columns *within* the row being validated is completely standard and one of CHECK's most common uses (validating relationships like list_price exceeding cost, or start_date preceding end_date).

**Concept Tested:** Multi-column, same-row CHECK constraints are fully legal.

**Exam Trap:** Overextending the cross-row restriction to also (incorrectly) forbid cross-column references within the same row.

---

### Subtopic 7: User-Defined Integrity

**Question 1**
A business rule states: "An employee's salary can never be decreased." Which mechanism correctly enforces this in Oracle?

A. A CHECK constraint comparing the new salary to the old salary.
B. A database trigger (e.g., a BEFORE UPDATE row-level trigger) that compares `:NEW.salary` to `:OLD.salary` and raises an application error if the new value is lower.
C. A UNIQUE constraint on the SALARY column.
D. A FOREIGN KEY constraint referencing the employee's own prior salary history table.

**Correct Answer:** B

**Detailed Explanation:**
"User-defined integrity" refers to business rules that fall outside what declarative constraints (PK, FK, UNIQUE, NOT NULL, CHECK) can express — in this case, comparing a row's *new* value against its *own prior* value, which requires access to both OLD and NEW row images. That is exactly what a trigger provides; no declarative constraint type has this capability.

**Why the Other Options Are Wrong:**
A. CHECK constraints only ever see the current row being validated — they have no built-in concept of "the previous value of this same row" (there is no `:OLD` in a CHECK's expression).
C. UNIQUE has nothing to do with directional value comparisons over time.
D. FK enforces existence in another table's key, not a business rule about value trends.

**Concept Tested:** Declarative constraints vs. procedural (trigger-based) enforcement of complex business rules.

**Exam Trap:** Trying to force an inherently procedural, "compare to history" rule into a purely declarative constraint — this is exactly the boundary "user-defined integrity" is meant to test.

---

**Question 2**
Which of these business rules **can** be enforced with a standard declarative constraint (no trigger needed)?

A. "An order's ship_date must be on or after its order_date."
B. "A product's price cannot be changed more than once per calendar month."
C. "A customer's total unpaid invoices cannot exceed their credit limit."
D. "An employee cannot be assigned to a department located in a country currently under sanctions" (looked up from an external reference table).

**Correct Answer:** A

**Detailed Explanation:**
A is a same-row, multi-column comparison — a textbook legal CHECK constraint (`CHECK (ship_date >= order_date)`). The others all require information *outside the current row* (time-based history, cross-row aggregation, or an external/other-table lookup), which only procedural code (triggers, packages, or application logic) can evaluate.

**Why the Other Options Are Wrong:**
B. Requires tracking change history over time — impossible for a stateless CHECK.
C. Requires aggregating across multiple other rows (all of a customer's invoices) — CHECK cannot reference other rows.
D. Requires a lookup against another table's current data — off-limits to CHECK (no subqueries).

**Concept Tested:** Recognizing which rules are same-row/stateless (constraint-eligible) vs. which require history, aggregation, or external lookups (trigger/procedural-eligible).

**Exam Trap:** Assuming any rule that "sounds like a simple validation" is automatically implementable as a CHECK constraint.

---

**Question 3**
True or False: "User-defined integrity" is a distinct, named Oracle constraint TYPE (like PRIMARY KEY or CHECK) that you declare with a specific keyword.

A. True
B. False

**Correct Answer:** B (False)

**Detailed Explanation:**
Unlike entity/referential/domain integrity — each tied to specific, named constraint types (PK, FK, NOT NULL, UNIQUE, CHECK) — "user-defined integrity" is a conceptual category describing *any business rule too specific or complex for those built-in types*, typically implemented via triggers, stored procedures, or application-tier logic rather than a single dedicated SQL keyword.

**Concept Tested:** Terminology — user-defined integrity is a conceptual category, not a syntactic constraint keyword.

**Exam Trap:** Searching for a nonexistent `CREATE CONSTRAINT ... USER_DEFINED` syntax rather than recognizing this is about *where* the rule is implemented (triggers/procedures), not a new declarative construct.

---

### Subtopic 8: Enabling and Disabling Constraints

**Question 1**
`ALTER TABLE employees DISABLE CONSTRAINT emp_email_uk;`
What is the immediate effect on existing data and future DML?

A. Existing rows are checked and made compliant immediately; only future DML is unaffected.
B. Existing rows are left entirely as-is (even if some now violate the rule), and — crucially — the constraint is no longer enforced for any *future* DML either, so new violations can be freely introduced.
C. The constraint remains fully enforced for all future DML; DISABLE only affects historical/existing rows.
D. DISABLE CONSTRAINT immediately drops the constraint's definition permanently — it cannot be re-enabled.

**Correct Answer:** B

**Detailed Explanation:**
Plain `DISABLE CONSTRAINT` (equivalent to `DISABLE NOVALIDATE`) turns off enforcement completely: existing rows aren't re-validated (any that already comply remain compliant, but nothing is checked), and — unlike what many candidates expect — subsequent INSERT/UPDATE statements are also no longer checked against that rule until it's re-enabled.

**Why the Other Options Are Wrong:**
A. Reverses the actual effect — disabling never re-validates or "fixes" data.
C. Also reversed — disabling turns off enforcement for all DML, not just historical data.
D. The constraint's metadata definition persists; it can be re-enabled later with `ENABLE CONSTRAINT`.

**Concept Tested:** DISABLE CONSTRAINT (NOVALIDATE) suspends enforcement entirely, for both existing and future data.

**Exam Trap:** Assuming "disable" only means "stop checking old data" while still protecting new data — it actually means the rule is off, full stop, until re-enabled.

---

**Question 2**
Which two statements about `ENABLE CONSTRAINT` (the default `ENABLE VALIDATE` form) are true? **(Choose two)**

A. Re-enabling validates all existing rows against the constraint; if any row currently violates it, the ENABLE statement fails.
B. Re-enabling only affects rows inserted/updated after the ENABLE statement — pre-existing data is never checked.
C. If existing data already violates the constraint, you must fix or remove the offending rows before the ENABLE will succeed.
D. ENABLE CONSTRAINT can never fail — it always succeeds, silently ignoring any violating rows.

**Correct Answers:** A and C

**Detailed Explanation:**
`ENABLE VALIDATE` (the default when you just say `ENABLE CONSTRAINT`) checks *all* current data as part of re-enabling. If violations exist, the statement fails outright, and the offending rows must be corrected first — exactly mirroring how adding a brand-new constraint behaves.

**Why the Other Options Are Wrong:**
B. Describes `ENABLE NOVALIDATE` instead — a distinct, less common variant that enforces the rule going forward *without* checking existing data.
D. Directly contradicts documented behavior — ENABLE VALIDATE absolutely can and does fail on bad existing data.

**Concept Tested:** ENABLE VALIDATE (default) vs. ENABLE NOVALIDATE.

**Exam Trap:** Assuming there's only one "flavor" of enabling a constraint — the VALIDATE/NOVALIDATE distinction is frequently tested precisely because most people only know the default behavior exists.

---

**Question 3**
You need to disable a PRIMARY KEY constraint that has a dependent FOREIGN KEY in another table. Which statement correctly does this in one step?

A. `ALTER TABLE departments DISABLE CONSTRAINT dept_pk;`
B. `ALTER TABLE departments DISABLE CONSTRAINT dept_pk CASCADE;`
C. `ALTER TABLE departments DROP CONSTRAINT dept_pk CASCADE;`
D. `ALTER TABLE employees DISABLE CONSTRAINT emp_dept_fk;` (disabling the child instead)

**Correct Answer:** B

**Detailed Explanation:**
Just as DROP requires CASCADE to remove a referenced key with dependents, DISABLE also requires CASCADE to disable a parent constraint that a FOREIGN KEY currently depends on — without it, Oracle blocks the DISABLE to protect referential integrity, the same protective logic as Subtopic 1, Question 5, just applied to DISABLE instead of DROP.

**Why the Other Options Are Wrong:**
A. Without CASCADE, this fails while the dependent FK still exists.
C. DROP CASCADE removes the constraint permanently (and the dependent FK with it) — a much more destructive action than what was asked ("disable," not "drop").
D. Disabling the FK does address the dependency, but it doesn't answer what was actually asked (disabling the PK) and isn't "one step" toward that specific goal.

**Concept Tested:** CASCADE requirement extends to DISABLE, not just DROP.

**Exam Trap:** Remembering CASCADE for DROP but forgetting the identical dependency rule applies to DISABLE.

---

**Question 4**
Which statement about `DISABLE VALIDATE` (as opposed to plain `DISABLE` / `DISABLE NOVALIDATE`) is TRUE?

A. They are just two different keywords for exactly the same behavior.
B. DISABLE VALIDATE guarantees existing data satisfies the constraint (validated at the moment of disabling) but then blocks any further INSERT/UPDATE to the constrained column(s) altogether, since there's no active enforcement mechanism left to check new values.
C. DISABLE VALIDATE re-enables the constraint automatically after a fixed time period.
D. DISABLE VALIDATE only applies to CHECK constraints, never to PK/UNIQUE/FK.

**Correct Answer:** B

**Detailed Explanation:**
`DISABLE VALIDATE` is a specialized state: Oracle validates that current data satisfies the rule, then drops the enforcement index/mechanism for performance reasons (e.g., ahead of a bulk direct-path load or a partition exchange) — but to guarantee the now-unenforced constraint can't be silently violated, it locks the constrained column(s) against further modification entirely until the constraint is fully re-enabled or disabled differently.

**Why the Other Options Are Wrong:**
A. `DISABLE NOVALIDATE` (plain DISABLE) does not validate anything and does not block future DML — a materially different behavior from DISABLE VALIDATE.
C. No automatic re-enabling/timeout mechanism exists.
D. DISABLE VALIDATE is a general option available across PK, UNIQUE, FK, and CHECK constraints, not CHECK-exclusive.

**Concept Tested:** The four-way ENABLE/DISABLE × VALIDATE/NOVALIDATE matrix, focusing on the more advanced DISABLE VALIDATE case.

**Exam Trap:** Treating "VALIDATE" in the option name as simply meaning "stronger enforcement" rather than understanding its specific, counterintuitive practical effect (blocking DML on the column entirely).

---

**Question 5**
Which of the following correctly lists all four legal states in Oracle's ENABLE/DISABLE × VALIDATE/NOVALIDATE model?

A. ENABLE VALIDATE, ENABLE NOVALIDATE, DISABLE VALIDATE, DISABLE NOVALIDATE
B. ENABLE STRICT, ENABLE LOOSE, DISABLE STRICT, DISABLE LOOSE
C. ENABLE VALIDATE, DISABLE VALIDATE only (NOVALIDATE is not a real Oracle keyword)
D. VALIDATE, NOVALIDATE, ENABLE, DISABLE (used independently, never combined)

**Correct Answer:** A

**Detailed Explanation:**
Oracle's full constraint-state model is a 2×2 matrix: whether the constraint is enforced going forward (ENABLE/DISABLE) crossed with whether existing data was checked (VALIDATE/NOVALIDATE). All four combinations are legal, named exactly as in option A, and each has a distinct practical meaning (as explored across this subtopic's questions).

**Why the Other Options Are Wrong:**
B. Invents nonexistent keyword names.
C. NOVALIDATE is a real, documented keyword — this option wrongly denies it.
D. The keywords are always combined in practice (e.g., `ENABLE VALIDATE`), not used standalone.

**Concept Tested:** The complete, correctly-named ENABLE/DISABLE × VALIDATE/NOVALIDATE state matrix.

**Exam Trap:** Only memorizing plain ENABLE/DISABLE and being caught off guard by exam questions that specifically test the VALIDATE/NOVALIDATE qualifiers.