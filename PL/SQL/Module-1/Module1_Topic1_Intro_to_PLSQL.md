# Module 1, Topic 1: Introduction to PL/SQL

---

## 1. What Is PL/SQL?

**SQL** (Structured Query Language) is a *declarative* language — you tell the database **what** data you want, not **how** to get it. `SELECT`, `INSERT`, `UPDATE`, `DELETE` are all SQL statements. SQL has no concept of variables, loops, conditions, or reusable logic blocks.

**PL/SQL** (Procedural Language extension to SQL) is Oracle's *procedural* extension built on top of SQL. It lets you write **programs** — with variables, loops, conditionals, error handling, and reusable units (procedures, functions, packages) — that can *contain* SQL statements inside them.

Think of it this way:
- SQL = the tool that touches the data (read/write).
- PL/SQL = the language that controls *when, how many times, under what conditions*, and *in what sequence* SQL gets executed, plus what to do when something goes wrong.

---

## 2. Why Does PL/SQL Exist? What Problem Does It Solve?

Imagine a business requirement like:

> "For every customer whose account balance is negative, deduct a penalty fee unless they are a premium member, and log any accounts where the deduction fails."

Try solving that with pure SQL. You immediately hit walls:
- SQL has no `IF/ELSE` branching logic across rows.
- SQL has no loops to process one row at a time when row-by-row logic is needed.
- SQL has no error handling — if one row fails, you don't get to "catch" it and continue gracefully.
- SQL has no reusable named blocks — you can't package this logic and call it from an application or scheduler.

PL/SQL was created to close this gap. It solves problems that require:
- **Conditional logic** across data (if/else, case).
- **Iteration** (looping row by row or a fixed number of times).
- **Reusable, callable code** (procedures, functions) that applications can invoke instead of duplicating logic everywhere.
- **Error resilience** — catching problems, logging them, deciding whether to continue or stop.
- **Performance** — PL/SQL runs *inside the Oracle database engine*, so a block containing 10 SQL statements doesn't need 10 round-trips between the application and the database. This is a big deal in real systems.

---

## 3. Why Is It Used? (The Business Case)

In a real company, PL/SQL typically gets used when:
- Business logic must live close to the data (e.g., "no employee's salary can be increased more than 20% without manager approval") and multiple applications need to enforce that same rule consistently. Putting it in the database (via a procedure or trigger) means every application automatically respects it — you don't rely on every app team remembering to implement it correctly.
- You need to process large volumes of data with row-by-row conditional logic (batch jobs, nightly reconciliation, data migrations).
- You need scheduled/automated jobs (e.g., "every night, flag overdue invoices") — these are often written as PL/SQL procedures triggered by a scheduler.
- Performance matters — reducing network round-trips between an app server and the database by pushing logic into the database itself.

---

## 4. Syntax: The Anatomy of a PL/SQL Block

Every PL/SQL program is built from a **block**. This is the single most fundamental structural idea in PL/SQL — everything else (procedures, functions, packages, even a simple ad-hoc script) is a variation of this block structure.

```sql
DECLARE
    -- declaration section (optional)
    -- variables, constants, cursors, exceptions declared here
BEGIN
    -- execution section (mandatory)
    -- your actual logic: SQL statements, IF/LOOP, assignments
EXCEPTION
    -- exception handling section (optional)
    -- what to do if something goes wrong in the BEGIN section
END;
/
```

### Syntax Breakdown

| Section | Keyword | Mandatory? | Purpose |
|---|---|---|---|
| Declaration | `DECLARE` | No | Where you declare variables, constants, and other elements you'll use below. If you have nothing to declare, you can omit this entire section. |
| Execution | `BEGIN ... END;` | **Yes** | The only mandatory part. Contains the actual logic — assignments, SQL, control statements. |
| Exception Handling | `EXCEPTION` | No | Catches runtime errors that occur in the execution section and lets you handle them instead of the program crashing. |
| Terminator | `/` | Depends on tool | Tells tools like SQL*Plus or SQLcl "this is the end of the block, execute it now." Not part of PL/SQL syntax itself — it's a client-tool convention. |

A block with **no name** (like above) is called an **anonymous block**. It runs once and is discarded — it is not stored in the database, and nothing else can call it. This is what we'll write when we're just testing logic or running a one-off script.

---

## 5. Types/Variations of PL/SQL Blocks

This is important to get right early, because it sets up everything coming later in Module 3.

| Type | Named? | Stored in DB? | Callable by other programs? | Example |
|---|---|---|---|---|
| **Anonymous Block** | No | No | No | The `DECLARE...BEGIN...END;` block above |
| **Procedure** | Yes | Yes | Yes | `CREATE PROCEDURE ...` (covered in Module 3) |
| **Function** | Yes | Yes | Yes (and returns a value) | `CREATE FUNCTION ...` (covered in Module 3) |
| **Package** | Yes | Yes | Yes (a collection of procedures/functions) | `CREATE PACKAGE ...` (covered in Module 3) |
| **Trigger** | Yes | Yes | Fires automatically on an event | *(Not in our syllabus — mentioning only for context, not as a topic we'll cover)* |

For now, in this topic, we only care about the **anonymous block** — it's the foundation everything else is built on.

---

## 6. Simple Examples

### Example 1 — Minimal valid block (does nothing observable)
```sql
BEGIN
    NULL;  -- NULL here means "do nothing" — a placeholder statement
END;
/
```
This is the most minimal legal PL/SQL block. `BEGIN...END` is mandatory; everything else is optional.

### Example 2 — Using the declaration section
```sql
DECLARE
    v_message VARCHAR2(50);  -- declaring a variable
BEGIN
    v_message := 'Hello, PL/SQL';  -- assignment
    DBMS_OUTPUT.PUT_LINE(v_message);  -- printing to console
END;
/
```

`DBMS_OUTPUT.PUT_LINE` is Oracle's built-in way to print text — the PL/SQL equivalent of `print()` or `console.log()`. You'll use this constantly while learning, to "see" what your code is doing.

### Example 3 — A block containing an actual SQL statement
```sql
DECLARE
    v_emp_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_emp_count FROM employees;
    DBMS_OUTPUT.PUT_LINE('Total employees: ' || v_emp_count);
END;
/
```

Notice: SQL's `SELECT` doesn't return data to the screen automatically like in SQL*Plus — inside PL/SQL, a `SELECT` must store its result **into** a variable using `INTO`. This is one of the very first "PL/SQL is not SQL" adjustments people have to make.

---

## 7. Detailed Explanation — What's Really Happening

When Oracle executes a PL/SQL block, here's the conceptual flow:

1. **Compilation**: Oracle parses the block, checks syntax, and checks that all variables/objects referenced actually exist and are used correctly (type-checking).
2. **Execution starts at `BEGIN`**, runs statement by statement, top to bottom (this is "sequential" execution — more on that in Module 2).
3. If any statement raises a runtime error, execution **immediately jumps** to the `EXCEPTION` section (if one exists). If there's no exception section, the block terminates with an unhandled error.
4. Once `END;` is reached (or an exception is handled), the block finishes. For an anonymous block, nothing is saved — it's gone. Run it again, and it executes fresh.

**Key mental model**: A PL/SQL block is a **request sent as a whole unit to the database engine**. All the SQL statements *inside* it are still processed by the SQL engine — PL/SQL doesn't reinvent SQL, it just orchestrates *when* and *how* SQL runs, and handles everything SQL itself cannot (branching, looping, variables, error handling).

---

## 8. When to Use / When Not to Use (at the "block" level)

**Anonymous blocks are useful when:**
- You're testing/prototyping logic quickly.
- You need a one-time script (e.g., a one-off data fix run by a DBA).
- You're inside a larger tool (like a migration script) and don't need a permanently stored, reusable object.

**Anonymous blocks are *not* appropriate when:**
- The logic needs to be **reused** by multiple applications or called repeatedly → use a **procedure** or **function** instead (Module 3).
- The logic needs to be **invoked automatically by database events** (insert/update/delete) → that's a trigger's job (outside our syllabus).
- Pure data retrieval with no procedural logic is needed → just use plain SQL; don't wrap a `SELECT` in PL/SQL unnecessarily. This is a **very common junior-developer mistake**: wrapping a simple, single `SELECT` in an anonymous PL/SQL block adds no value and only adds overhead when a plain SQL query would do.

---

## 9. Common Mistakes & Misconceptions

1. **Mistake**: Forgetting `INTO` in a `SELECT` inside PL/SQL.
   → SQL*Plus-style bare `SELECT` doesn't work inside PL/SQL; it must select into a variable.
2. **Misconception**: "PL/SQL replaces SQL."
   → False. PL/SQL *complements* SQL. You still write SQL for data access; PL/SQL adds control-flow around it.
3. **Mistake**: Thinking an anonymous block is "saved" somewhere after running it.
   → It isn't. Nothing persists. If you need to reuse logic, you need a named, stored object (procedure/function).
4. **Misconception**: Believing PL/SQL is slow because it's "extra layer."
   → Actually the opposite is often true in real systems — PL/SQL running inside the database reduces network round-trips compared to an application making multiple separate SQL calls.
5. **Mistake**: Not realizing `DECLARE` is optional.
   → Beginners often think every block *must* start with `DECLARE`. It doesn't — only include it if you actually have something to declare.

---

## 10. Edge Cases to Be Aware Of

- A block with a `DECLARE` section but nothing actually declared is legal but pointless — Oracle won't error, but it signals sloppy code.
- If an error occurs and there's **no matching exception handler**, the error propagates out of the block entirely (to whatever called it, or to the console if it's a top-level anonymous block).
- `DBMS_OUTPUT.PUT_LINE` output is **only visible if output is enabled** in your client (e.g., `SET SERVEROUTPUT ON` in SQL*Plus/SQLcl). This trips up almost everyone the first time — you run a block, nothing prints, and you assume it failed when actually output was just suppressed.

---

## 11. Interview-Level / Practical Notes

- Interviewers commonly ask: *"What is the difference between SQL and PL/SQL?"* — Be ready to say: SQL is declarative and set-based (operates on a whole result set at once); PL/SQL is procedural and can operate row-by-row, has control structures, and can bundle multiple SQL statements into a single unit executed by the engine.
- Another common question: *"Is `BEGIN...END` mandatory?"* — Yes, always. `DECLARE` and `EXCEPTION` are optional.
- Real teams almost never write anonymous blocks for production logic — they use procedures/functions/packages. Anonymous blocks are mainly for scripting, testing, and one-off DBA tasks. Knowing this distinction shows maturity in an interview.

---

## Things You Must Remember

- `BEGIN...END;` is the only mandatory part of any PL/SQL block.
- `DECLARE` and `EXCEPTION` sections are both optional.
- A bare `SELECT` inside PL/SQL **must** use `INTO` to store its result in a variable — you cannot just run a `SELECT` and have it display results like in a SQL client.
- Anonymous blocks are **not stored** in the database and **cannot be reused/called** by other code.
- `DBMS_OUTPUT.PUT_LINE` needs `SET SERVEROUTPUT ON` (or client equivalent) to actually display anything — a "silent" run doesn't necessarily mean failure.
- The `/` at the end is a client-tool convention (SQL*Plus/SQLcl) to trigger execution, not part of the PL/SQL language itself.
- PL/SQL runs inside the Oracle engine — this is what gives it its performance advantage over multiple round-trip SQL calls from an application.

## How to Recognize This Concept

You're looking at a **"do I even need PL/SQL"** decision point (i.e., recognizing when a plain SQL statement is not enough and you need a PL/SQL block) when the requirement contains language like:
- "...**and then** do X..." (sequencing of multiple actions/steps)
- "...**if** condition **then** do this, **otherwise** do that..." (conditional logic across data)
- "...**for each** record/row, check/update/process..." (row-by-row iteration)
- "...**handle** cases where data is missing/invalid..." (error handling)
- "...combine multiple steps as **one operation**..." (batching statements together, reducing round-trips)

If a requirement can be fully expressed as a single `SELECT`/`INSERT`/`UPDATE`/`DELETE` with no branching or looping, you don't need PL/SQL — plain SQL is the right (and better) tool.

---

## Easy Exercises

1. **(Block structure)** Write the most minimal valid PL/SQL block possible (it should do nothing except run without error).
2. **(Declaration + output)** Declare a variable to hold your name (as text) and print `"My name is <your name>"` using `DBMS_OUTPUT.PUT_LINE`.
3. **(Assignment)** Declare two number variables, assign them values, and print their sum.
4. **(SELECT...INTO)** Write a block that selects the current system date (`SYSDATE`) into a variable and prints it.
5. **(Concatenation)** Declare a variable holding a department name and print `"Department: <name>"` using the `||` concatenation operator.
6. **(No DECLARE)** Write a block with no declaration section that just prints a fixed message — to prove to yourself that `DECLARE` truly is optional.

## Intermediate Exercises

7. **(SELECT...INTO with a real table)** Assuming a table `employees(employee_id, first_name, salary)` exists, write a block that selects the salary of employee_id = 100 into a variable and prints it.
8. **(Aggregate into variable)** Select the total number of rows in `employees` into a variable and print a message like `"Total employees: N"`.
9. **(Multiple variables from one row)** Select both `first_name` and `salary` of a specific employee into two separate variables in a single `SELECT...INTO`, and print both.
10. **(Predict the error)** *Without running it*, identify what will go wrong with this block, and why:
    ```sql
    BEGIN
        SELECT salary FROM employees WHERE employee_id = 100;
    END;
    /
    ```
11. **(Real-world framing)** A teammate says: *"I don't need PL/SQL, I'll just run five separate SQL queries from my Java app."* In 3–4 sentences, explain a scenario where wrapping that logic in a single PL/SQL block instead would be the better engineering decision, and why.

---

*Post your answers in chat and I'll review them, correct any misconceptions, and then produce the file for Topic 2: Block Structure & Types of SQL Statements.*
