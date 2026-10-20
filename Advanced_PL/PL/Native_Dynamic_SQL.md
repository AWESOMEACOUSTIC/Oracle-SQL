# Dynamic SQL in PL/SQL — Easy Guide
*EXECUTE IMMEDIATE · Dynamic DDL & DML · The Two Hidden Traps*

---

## 1. The Big Idea

Normally, the SQL you write inside PL/SQL is **fixed**: you type it, Oracle checks it when the procedure compiles, and it never changes.

`EXECUTE IMMEDIATE` lets you **build the SQL as a text string while the program runs**, and then tells Oracle to run that string.

- **Static SQL** is a printed form. Every field is decided in advance.
- **Dynamic SQL** is writing a note on the spot. You decide what to say when the moment comes.

### Why this is in your syllabus

So far in PL/SQL you've written code where everything is known upfront: cursors, procedures, exception handling. **Dynamic SQL is the chapter where your code becomes flexible.** It adapts to tables, filters, and statements that only exist at runtime.

This guide covers:
1. **How** `EXECUTE IMMEDIATE` works
2. **When** to use it (and when not to)
3. **How to stay safe** (validating names)
4. **Two runtime surprises** that catch almost everyone

---

## 2. Why Do We Need It?

Static SQL has two limits that you will run into in real projects.

### Limit 1: Oracle must know every table and column name at compile time

```sql
-- Goal: delete old rows from WHICHEVER table the caller asks for
PROCEDURE purge(p_table VARCHAR2) IS
BEGIN
  DELETE FROM p_table WHERE created < SYSDATE - 90;   -- ❌ Won't compile
END;
```

Oracle reads `p_table` as the literal name of a table called "P_TABLE", which doesn't exist. Static SQL cannot use a variable as a table name.

```sql
  EXECUTE IMMEDIATE 'DELETE FROM ' || p_table || ' WHERE created < SYSDATE - 90';  -- ✅
```

Now the table name is just part of a string, joined in at runtime.

### Limit 2: PL/SQL has no static DDL at all

PL/SQL was designed to work with **data**, not to reshape the database itself. When a block compiles, Oracle checks every table and column it mentions. If the block could create or drop tables halfway through, Oracle couldn't safely check anything ahead of time. So PL/SQL simply refuses DDL as normal code.

```sql
BEGIN
  DROP TABLE emp_backup;                       -- ❌ Compile error
  TRUNCATE TABLE staging_tbl;                  -- ❌ Compile error
  EXECUTE IMMEDIATE 'DROP TABLE emp_backup';   -- ✅ The only way
END;
```

Think of it as a **locked door with one key**: any DDL (CREATE, ALTER, DROP, TRUNCATE, GRANT, REVOKE) inside PL/SQL can only get through via dynamic SQL (`EXECUTE IMMEDIATE` or `DBMS_SQL`).

> **Summary:** `EXECUTE IMMEDIATE` exists because some SQL **can't be known in advance**, and some SQL **isn't allowed as static code**.

---

## 3. Where You'd Actually Use It

| Situation | Why static SQL fails |
|---|---|
| Monthly tables like `sales_2026_09`, `sales_2026_10` | The table name changes every month |
| Nightly ETL job that does `TRUNCATE` on a staging table before loading | TRUNCATE is DDL |
| Search screen where the user may filter by name, city, salary, or any mix | The WHERE clause changes shape each time |
| A generic "archive/purge" utility used for many tables | The table name comes from a parameter |
| Calling a procedure whose name is stored in a config table | The procedure name is only known at runtime |
| `ALTER SESSION SET NLS_DATE_FORMAT = ...` inside a procedure | Session control statements aren't static PL/SQL |

The "truncate staging table, then reload" pattern alone makes this a daily-use statement in batch and ETL work.

---

## 4. How EXECUTE IMMEDIATE Works

### Syntax

```sql
EXECUTE IMMEDIATE dynamic_sql_string
  [ INTO define_var1 [, define_var2 ]... ]
  [ USING [IN|OUT|IN OUT] bind_arg1 [, bind_arg2 ]... ]
  [ RETURNING INTO out_var1 [, out_var2 ]... ];
```

### Think of it as a machine with an input slot and an output slot

```
            USING  (values going IN)
                 │
                 ▼
   ┌──────────────────────────────┐
   │  'SQL text built at runtime' │
   └──────────────────────────────┘
                 │
                 ▼
   INTO / BULK COLLECT INTO / RETURNING INTO  (values coming OUT)
```

| Clause | Job |
|---|---|
| `USING` | Fill in the `:placeholders` with values (an employee ID, a department number) |
| `INTO` | Catch **one row** a SELECT returns |
| `BULK COLLECT INTO` | Catch **many rows** into a collection |
| `RETURNING INTO` | Catch values from an UPDATE/INSERT/DELETE (e.g., the new salary after a raise) |

### What it can and can't run

| ✅ Can run | ❌ Can't run directly |
|---|---|
| DDL (CREATE, ALTER, DROP, TRUNCATE, GRANT…) | A multi-row query with plain `INTO` → raises `TOO_MANY_ROWS` |
| DML (INSERT, UPDATE, DELETE, MERGE) | For many rows use a **REF CURSOR** (`OPEN cur FOR string`) or **BULK COLLECT INTO** |
| A single-row SELECT (via `INTO`) | |
| Anonymous PL/SQL blocks (including calls to procedures) | |
| Session statements (`ALTER SESSION`) | |

### Examples

```sql
-- Single-row query: USING = the question (IN), INTO = the answer (OUT)
EXECUTE IMMEDIATE 'SELECT salary FROM employees WHERE employee_id = :id'
  INTO  v_salary
  USING p_emp_id;

-- DML with RETURNING INTO
EXECUTE IMMEDIATE
  'UPDATE employees SET salary = salary * 1.1 WHERE employee_id = :id RETURNING salary INTO :new_sal'
  USING p_emp_id
  RETURNING INTO v_new_salary;

-- Calling a procedure whose name is only known at runtime
EXECUTE IMMEDIATE 'BEGIN ' || v_proc_name || '(:x); END;' USING p_value;

-- Bulk: FORALL with dynamic DELETE, and BULK COLLECT for a multi-row fetch
FORALL i IN 1 .. v_ids.COUNT
  EXECUTE IMMEDIATE 'DELETE FROM staging_tbl WHERE id = :1' USING v_ids(i);

EXECUTE IMMEDIATE 'SELECT salary FROM employees WHERE department_id = :d'
  BULK COLLECT INTO v_salaries USING p_dept_id;
```

### Small but useful facts

- **Quotes inside the string** must be doubled (`''`), or use alternative quoting `q'[ ... ]'` for readability.
- **`SQL%ROWCOUNT`, `SQL%FOUND`** still work after dynamic DML, just like static DML.
- **`EXECUTE IMMEDIATE` vs `DBMS_SQL`:** prefer `EXECUTE IMMEDIATE` for normal cases. `DBMS_SQL` is the older, more verbose package, and it's only needed when even the **number or types of columns** are unknown (e.g., a generic query-result browser).

---

## 5. Binds vs Concatenation (Values vs Names)

### The rule: placeholders are for values, never for names

```sql
'SELECT * FROM :tbl'            -- ❌ Can't bind a table name
'SELECT * FROM ' || v_tbl       -- ✅ Names must be glued in
'... WHERE id = :id'            -- ✅ Values should be bound
```

### Why bind values instead of gluing them in? SQL injection.

If you glue user input straight into the string:

```sql
'... WHERE name = ''' || p_name || ''''
```

a malicious user could type `x' OR '1'='1` and change what your query means. **Binds treat input purely as data**, so they can't alter the query.

### Names must be glued in, so validate them with DBMS_ASSERT

```sql
-- Caller passes:  'emp; DROP TABLE salaries'   😱
EXECUTE IMMEDIATE 'TRUNCATE TABLE ' || p_table_name;
```

`DBMS_ASSERT.SQL_OBJECT_NAME` checks that the text really is the name of an existing object. If it isn't, it raises an error **before** anything runs. Think of it as **checking someone's ID before letting them in**.

```sql
CREATE OR REPLACE PROCEDURE truncate_any_table (p_table_name IN VARCHAR2) IS
BEGIN
  EXECUTE IMMEDIATE 'TRUNCATE TABLE ' || DBMS_ASSERT.SQL_OBJECT_NAME(p_table_name);
END;
/
```

---

## 6. When to Use Dynamic SQL (and When Not To)

### DDL → always dynamic (no choice)

### DML → it's a judgment call. Ask:

> **Is only a *value* changing, or is the *structure* changing?**

**Only a value changes → use static SQL with a variable**

```sql
-- Employee ID differs each call, but the query shape never does
SELECT salary INTO v_sal FROM employees WHERE employee_id = p_id;   -- ✅ Static
```

**Structure changes (table name, columns, WHERE shape) → use dynamic SQL**

```sql
-- The TABLE itself differs each call
EXECUTE IMMEDIATE 'DELETE FROM ' || v_table || ' WHERE created < SYSDATE - 90';
```

### Why prefer static whenever you can

- Oracle checks it at **compile time**, so typos are caught early instead of crashing at runtime
- No injection risk
- Simpler to read and maintain
- Oracle reuses the execution plan with no dynamic-SQL overhead

Dynamic SQL is a powerful tool, but use it only when static SQL can't do the job.

### Example: "insert into one of three tables"

If the caller can pick from only **3** tables:

```sql
IF    p_tbl = 'A' THEN INSERT INTO table_a ...;
ELSIF p_tbl = 'B' THEN INSERT INTO table_b ...;
ELSIF p_tbl = 'C' THEN INSERT INTO table_c ...;
END IF;
```

This is a bit repetitive, but it's safe and simple, so it's the better default.
If there are **50** tables, or the list lives in a config table, writing 50 branches is silly. Then dynamic SQL wins, **as long as you validate the name** (DBMS_ASSERT or a whitelist).

### TRUNCATE vs DELETE: why TRUNCATE counts as DDL

| | DELETE | TRUNCATE |
|---|---|---|
| Type | DML | DDL |
| Can you ROLLBACK? | Yes | No |
| Auto-commits? | No | Yes |
| Speed on big tables | Slower (row by row) | Very fast (resets the table) |
| Static in PL/SQL? | Yes | No, needs `EXECUTE IMMEDIATE` |

---

## 7. Trap 1: DDL Secretly Commits

### First, a quick refresher on transactions

When you run INSERT, UPDATE, or DELETE, the changes are **pending** at first. They're like items in a shopping cart, not yet paid for.

- `COMMIT` means "pay for the cart." The changes become permanent.
- `ROLLBACK` means "empty the cart." The changes disappear.

### The hidden rule

**Every DDL statement presses COMMIT for you automatically,** and it does so *before* it runs. You don't see it happen, and you can't stop it. This is true whether the DDL is written statically (in plain SQL) or issued through `EXECUTE IMMEDIATE`.

### The story

Priya writes a procedure that transfers money:

```sql
-- Step 1
UPDATE accounts SET balance = balance - 5000 WHERE id = 1;   -- take money from A

-- Step 2: she calls a "helpful" logging utility someone else wrote
create_log_table;   -- inside it: EXECUTE IMMEDIATE 'CREATE TABLE log_tbl ...'

-- Step 3
UPDATE accounts SET balance = balance + 5000 WHERE id = 2;   -- give money to B
-- 💥 Step 3 fails (say, a constraint error)

ROLLBACK;   -- Priya expects everything to be undone
```

Here's what actually happens:

| Moment | What Priya thinks | What really happened |
|---|---|---|
| After Step 1 | ₹5000 removed (pending) | ₹5000 removed (pending) |
| During Step 2 | Just creating a table | **Hidden COMMIT. Step 1 is now permanent.** |
| Step 3 fails | — | — |
| ROLLBACK | Everything undone | Only things *after* the commit are undone. Step 1 stays. |

**Result:** ₹5000 left account A and never reached account B. The money vanished.

### The lesson

> Don't put DDL (CREATE, DROP, TRUNCATE, ALTER) in the middle of work that must succeed or fail as a whole.

Priya didn't even write the DDL herself. It was hidden inside someone else's procedure. That's why people who write **utility procedures** that create or drop objects have to be especially careful.

---

## 8. Trap 2: Privileges Through Roles

We'll build this up in four small steps.

### Step 1: What is a privilege?

A privilege is **permission to do something**. Examples:
- `CREATE TABLE`: you're allowed to create tables.
- `SELECT ON hr.employees`: you're allowed to read HR's employees table.

### Step 2: Two ways to get a privilege

**Way A: Direct grant.** The DBA gives it to you personally.

```sql
GRANT CREATE TABLE TO aditya;
```

**Way B: Through a role.** A role is a **bundle of privileges** with a name. The DBA puts privileges into the role, then gives the role to people.

```sql
CREATE ROLE dev_role;
GRANT CREATE TABLE TO dev_role;   -- privilege goes into the bundle
GRANT dev_role TO aditya;         -- bundle goes to Aditya
```

Companies use roles because it's easier. When 50 new joiners arrive, the DBA hands each one `dev_role` instead of granting 20 privileges one by one.

**Analogy:**
- Direct grant = **your personal key**, cut just for you.
- Role = **a key on a shared team keyring** that you've been handed.

### Step 3: The surprise

Aditya got `CREATE TABLE` only through `dev_role` (the team keyring).

**Typed directly in SQL Developer:**

```sql
CREATE TABLE test1 (id NUMBER);   -- ✅ Works
```

**Inside a stored procedure:**

```sql
CREATE OR REPLACE PROCEDURE make_table IS
BEGIN
  EXECUTE IMMEDIATE 'CREATE TABLE test2 (id NUMBER)';
END;
/
-- Compiles fine ✅

EXEC make_table;
-- ❌ ORA-01031: insufficient privileges
```

Same user, same statement. It works in one place and fails in the other.

### Step 4: Why does Oracle do this?

**Inside a normal stored procedure, Oracle ignores your roles** (only the `PUBLIC` role stays active). Only your *personal keys* (direct grants) count.

The reason: roles are **unstable**. In any session, a user can switch roles on or off (`SET ROLE`), and the DBA can change what's in a role at any time. A stored procedure, on the other hand, is a permanent object. It is saved once, and other people may run it for months. Oracle wants that procedure to always behave the same way, no matter which roles happen to be switched on in whatever session runs it. So it decides:

> "For stored procedures, I'll only trust permissions granted directly to the owner."

**Analogy:** the team keyring works when you're walking around the office yourself. But when you leave instructions for a robot to do a task for you while you're away (the stored procedure), the building only lets the robot use keys registered in **your name personally**.

### What "definer's rights" means

By default, a procedure runs **with the permissions of the person who created it** (the *definer*), not the person calling it. That's the normal mode, and it's the one where roles get switched off.

### How to fix it

**Fix 1 (most common):** ask the DBA for a direct grant.

```sql
GRANT CREATE TABLE TO aditya;
```

Now the procedure works.

**Fix 2:** make the procedure run with the *caller's* rights instead.

```sql
CREATE OR REPLACE PROCEDURE make_table
  AUTHID CURRENT_USER   -- "invoker's rights": run with the caller's privileges
IS
BEGIN
  EXECUTE IMMEDIATE 'CREATE TABLE test2 (id NUMBER)';
END;
/
```

In this mode, the caller's roles *are* active. (Covered in more depth in later sections, so it's enough to know it exists for now.)

### Why did it compile but fail only at runtime?

When Oracle compiles the procedure, `'CREATE TABLE test2 ...'` is **just a piece of text** to it. Oracle doesn't look inside the string, so it can't check permissions yet. The check happens only when the string actually runs, and that's when it fails.

(Compare: if your procedure had a *static* `SELECT * FROM hr.employees` and you could read that table only through a role, it would fail **at compile time**, because static SQL is checked during compilation.)

---

## 9. Both Traps at a Glance

| | Trap 1: Implicit commit | Trap 2: Roles disabled |
|---|---|---|
| What happens | DDL silently commits pending work | Privileges from roles don't count inside a procedure |
| Symptom | ROLLBACK doesn't undo everything | "Insufficient privileges," even though it works outside the procedure |
| When you notice | After data is already wrong | At runtime |
| Fix | Keep DDL out of the middle of transactions | Direct grant, or `AUTHID CURRENT_USER` |

---

## 10. Bind Variables: Performance & Security

A **bind variable** is a blank slot in your SQL (`:id`, `:1`) that you fill in *separately* when the SQL runs, instead of pasting the value straight into the SQL text.

```sql
-- ❌ Concatenation: the value becomes part of the text
EXECUTE IMMEDIATE 'SELECT salary FROM employees WHERE employee_id = ' || p_emp_id
  INTO v_salary;

-- ✅ Bind: the text stays identical every time; only the value changes
EXECUTE IMMEDIATE 'SELECT salary FROM employees WHERE employee_id = :id'
  INTO v_salary
  USING p_emp_id;
```

The query returns the same result either way. Binds change **how much work Oracle does**, and **how safe** the query is.

### Benefit 1: Performance (hard parse vs soft parse)

Before running any SQL, Oracle must **parse** it: check the grammar, check the objects and privileges, and work out the fastest route to the data (the **execution plan**). Prepared plans are cached in the **shared pool**, and Oracle reuses one only when the **SQL text matches exactly**.

- **Hard parse** = full preparation from scratch (slow, expensive)
- **Soft parse** = "I've seen this exact text before, reuse the plan" (fast, cheap)

| Without binds | With binds |
|---|---|
| `... = 101`, `... = 102`, `... = 103` → every value makes **different text** | `... = :id` → **one text** for every value |
| Hard parse on every call | 1 hard parse, then soft parses |
| Shared pool fills with near-duplicate plans (**fragmentation / thrashing**) | Shared pool stays clean |

**Analogy:** without binds, a chef treats "Dosa for table 1" and "Dosa for table 2" as two brand-new recipes and works each one out from scratch. With binds, every order is "Dosa for table `:t`," so the chef recognises the recipe instantly and only changes the table number.

### Benefit 2: Security

A bound value is passed purely as **data** and never read as SQL. If an attacker sends `'1 OR 1=1'`, it's compared to `employee_id` as one value, and it can't add logic to the query (see Section 5).

| Thing that varies | Protection |
|---|---|
| **Values** (IDs, names, dates) | Bind variables |
| **Names** (tables, columns) | `DBMS_ASSERT` or a whitelist; binds can't help |

### Important points

- **Static SQL binds automatically.** In `WHERE employee_id = p_id`, PL/SQL turns `p_id` into a bind behind the scenes. The real risk is in **dynamic SQL**, where it's tempting to `||` values into the string.
- **`CURSOR_SHARING = FORCE`** makes Oracle auto-replace literals with system-generated binds. It's a **stopgap for legacy apps**, not a replacement for writing explicit binds in new code. (Values: `EXACT` is the default, `FORCE`, and `SIMILAR`, which is deprecated.)

### Bind Variable Peeking (the one downside)

**The same query can need different plans for different values.**
- **Index scan:** best when **few rows** match (like using a book's index)
- **Full table scan:** best when **most rows** match (like reading the whole book)

**Example: an `orders` table with 1,000,000 rows (skewed data)**

| status | How many rows | Best strategy |
|---|---|---|
| `'CANCELLED'` | 10,000 (1%) | Index scan |
| `'DELIVERED'` | 990,000 (99%) | Full table scan |

```sql
SELECT * FROM orders WHERE status = :s
```

**What peeking is:** on the **first hard parse only**, Oracle looks at the actual value inside `:s` and chooses the best plan for *that* value. After that, every run soft-parses and reuses the same plan. Oracle **doesn't peek again**.

**Where it goes wrong:**

| Run | Value | What Oracle does | Result |
|---|---|---|---|
| 1st | `'CANCELLED'` | Hard parse → peeks → "only 1%" → saves **index plan** | ✅ Fast |
| 2nd | `'DELIVERED'` | Soft parse → reuses **index plan** (no peek) | ❌ Very slow for 990,000 rows |
| Every later run | Any value | Index plan again | Depends on luck |

> **The first value decided the plan for everyone, and Oracle never looked again.**

It works in the opposite direction too: if `'DELIVERED'` ran first, every `'CANCELLED'` search would needlessly read all 1,000,000 rows. So performance depends on **which value happened to run first**. That's why the same query can suddenly become slow after the shared pool is flushed or the database restarts.

- **Only a problem when** the column's data is **skewed** *and* the query runs with both rare and common values. For evenly spread data (like `employee_id`), every value wants the same plan, so peeking does no harm.
- **Still not a reason to avoid binds.** It's an edge-case trade-off.
- **The fix:** **Adaptive Cursor Sharing** (Oracle 11g+). When Oracle notices a reused plan performing badly for certain values, it creates an additional plan for them.

---

## 11. Key Points for the Exam

1. **All DDL from PL/SQL is dynamic.** There is no "static DDL in PL/SQL."
2. **`INTO` expects exactly one row.** Multiple rows raise `TOO_MANY_ROWS`; use a REF CURSOR or `BULK COLLECT INTO` instead.
3. **Binds are for values only.** Table and column names must be concatenated, then validated (`DBMS_ASSERT`).
4. **`USING` = data in; `INTO` / `BULK COLLECT INTO` / `RETURNING INTO` = data out.** A query can use both together.
5. **Use dynamic DML only when the structure varies.** If only a value varies, static SQL with a variable is better.
6. **DDL causes an implicit COMMIT**, even when issued dynamically.
7. **Role privileges are disabled inside definer's-rights units.** Dynamic SQL needing them fails at runtime, not compile time.
8. **TRUNCATE is DDL**: it can't be rolled back, it commits, and it needs `EXECUTE IMMEDIATE`.
9. **Binds improve performance**: identical text → soft parse and plan reuse, instead of a hard parse every time.
10. **Binds protect values only**, never identifiers.
11. **`CURSOR_SHARING = FORCE`** is a legacy stopgap; explicit binds are best practice.
12. **Bind peeking**: the plan is chosen from the first value's peek and reused, which is a risk with skewed data. Adaptive Cursor Sharing (11g+) reduces it.

---

## 12. Exercise Questions

**Q1. Why does `EXECUTE IMMEDIATE 'SELECT * FROM employees' INTO v_rec;` fail if `employees` has more than one row?**

> `INTO` expects exactly one row, just like a static `SELECT ... INTO`. Multiple rows raise `TOO_MANY_ROWS` at runtime. For many rows, use a REF CURSOR (`OPEN cur FOR string;` then `FETCH`) or `BULK COLLECT INTO` a collection.

**Q2. A procedure runs several INSERTs, then a dynamic `CREATE TABLE`, then more INSERTs, then `ROLLBACK`. What happens to the first batch of inserts?**

> They are **not** rolled back. The `CREATE TABLE` implicitly commits everything before it. The `ROLLBACK` only undoes the second batch.

**Q3. What's the difference between `USING` and `INTO` / `BULK COLLECT INTO`?**

> `USING` supplies bind values going **into** the statement (IN, or OUT / IN OUT). `INTO` / `BULK COLLECT INTO` captures results coming **out** of a query (one row, or many rows). A single query can use both.

**Q4. Can you write `DROP TABLE emp_backup;` directly inside a PL/SQL block?**

> No. PL/SQL has no static DDL, so this is a compile-time error. Use `EXECUTE IMMEDIATE 'DROP TABLE emp_backup';`.

**Q5. A procedure must insert into one of three tables the caller specifies. Use three static INSERTs in IF/ELSIF, or one dynamic INSERT?**

> Both work. For a small, fixed set like three, **static IF/ELSIF** is the safer, simpler default: no identifier concatenation, and normal cached plans. Dynamic SQL becomes more justified as the number of tables grows or comes from configuration, but then the table name must be validated (DBMS_ASSERT or a whitelist).

**Q6. Why is `TRUNCATE TABLE` DDL rather than DML, and what's the practical consequence in PL/SQL?**

> Like CREATE/DROP/ALTER, it implicitly commits and cannot be rolled back (unlike DELETE). In PL/SQL it must run via `EXECUTE IMMEDIATE`, and running it mid-transaction commits any pending work.

**Q7. Why does `EXECUTE IMMEDIATE 'DROP TABLE :t' USING 'OLD_DATA';` fail?**

> `:t` is trying to bind a table name, and binds can only hold values. The name must be concatenated into the string.

**Q8. A user has `CREATE TABLE` only through a role. Their procedure runs `EXECUTE IMMEDIATE 'CREATE TABLE x (id NUMBER)'`. It compiles. What happens at runtime, and why didn't it fail at compile time?**

> It fails at runtime with `ORA-01031: insufficient privileges`, because roles are disabled inside a definer's-rights procedure. It compiled fine because Oracle doesn't look inside the string at compile time, so the privilege check only happens when the statement runs.

**Q9. Aditya can run `SELECT * FROM hr.employees` in SQL Developer, but `EXECUTE IMMEDIATE 'SELECT COUNT(*) FROM hr.employees' INTO v_cnt;` fails inside a normal procedure. Most likely reason?**

> His SELECT permission on that table comes through a role, and roles are switched off inside a definer's-rights procedure. Fix: a direct `GRANT SELECT ON hr.employees TO aditya;`, or `AUTHID CURRENT_USER`.

**Q10. Why does replacing concatenation with a bind variable improve performance in a high-volume application, even though the result is the same?**

> With concatenation, each distinct value makes distinct SQL text, so Oracle does a full hard parse almost every time and fills the shared pool with near-duplicate statements. With a bind, the text stays identical, so Oracle soft-parses and reuses the cached plan, which is far cheaper at scale.

**Q11. Does binding `employee_id` protect against a malicious table name injected elsewhere in the same dynamic statement?**

> No. Binds protect data values only. A dynamically supplied table name has to be concatenated and validated separately (DBMS_ASSERT or a whitelist).

**Q12. What is bind variable peeking, and why is it sometimes called a downside of binds?**

> On a statement's first hard parse, Oracle peeks at the actual bound values to choose an execution plan. That plan is then reused for all later executions with different values. With skewed data, it can be a poor fit for those later values.

**Q13. `WHERE country = :c` runs fast all morning. At noon the shared pool is flushed, and the query suddenly becomes slow for most users. What probably happened?**

> Before noon, the first run used a common value (e.g., `'INDIA'`), so Oracle chose a full scan, which suits most searches. After the flush, the first run happened to use a rare value (e.g., `'ICELAND'`). Oracle peeked, chose an index plan, and now reuses it for everyone, including common countries where it's slow.