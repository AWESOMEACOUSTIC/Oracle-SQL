# Module 4, Topic 1: Introduction to Exception

---

## 1. What Is an Exception?

An **exception** is a **runtime error condition** that disrupts the normal flow of a PL/SQL block. When something goes wrong while a block is executing — a query finds no rows, a calculation divides by zero, a constraint is violated — Oracle **raises an exception**, which immediately stops normal execution and transfers control to a special part of the block: the **`EXCEPTION` section**.

You already saw the `EXCEPTION` section's *place* in a block's structure back in Module 1, Topic 1 — it was mentioned as optional, but left unexplained. This entire module is about understanding what actually happens there, and how to control it deliberately instead of letting your program crash.

---

## 2. Why Does Exception Handling Exist? What Problem Does It Solve?

Consider this simple block:

```sql
DECLARE
    v_salary NUMBER;
BEGIN
    SELECT salary INTO v_salary FROM employees WHERE employee_id = 9999;  -- doesn't exist
    DBMS_OUTPUT.PUT_LINE('Salary: ' || v_salary);
END;
/
```

If `employee_id = 9999` doesn't exist, the `SELECT INTO` returns **zero rows**. Without any exception handling, this block simply **crashes** — Oracle raises `ORA-01403: no data found`, execution stops immediately, and nothing after the failed line runs (not even the `DBMS_OUTPUT.PUT_LINE`).

Now think about what this means in a **real business system**:

> "For every employee in the department, calculate and apply their bonus."

If this logic runs as a loop (Module 2, coming later) processing 500 employees, and employee #217 has some data issue (missing salary, invalid grade, whatever), an **unhandled exception on employee #217 would crash the entire batch job** — meaning employees #1 through #216 might get processed, but #217 through #500 never do, and depending on how the transaction is structured, you could even lose the work already done for #1–216.

This is precisely the problem exception handling exists to solve: **real-world data is messy, real-world systems fail in expected and unexpected ways, and a robust program must decide deliberately how to respond** — log the problem and continue, roll back cleanly, notify someone, retry — rather than simply crashing uncontrolled and leaving things in an unknown, inconsistent state.

---

## 3. Why Is It Used? (The Business Case)

- **Resilience**: batch jobs, data migrations, and nightly processes need to keep going (or fail gracefully and cleanly) even when individual records have problems — a single bad row shouldn't necessarily take down an entire multi-hour job.
- **Data integrity**: when something goes wrong partway through a multi-step operation (like the fund transfer example from Module 3), you need to decide deliberately whether to roll back, and ensure the database isn't left in a half-completed, inconsistent state.
- **Meaningful error communication**: instead of a cryptic Oracle error code reaching an end user or an application log, exception handling lets you translate technical failures into **business-meaningful messages** ("Employee not found" instead of `ORA-01403`) — this becomes especially powerful once we cover Raise Application Error later in this module.
- **Debuggability and auditability**: exception handlers are often where problems get **logged** (to an error table, for instance) so support teams can investigate issues after the fact, rather than errors vanishing silently or crashing with no trace.

---

## 4. How Exceptions Fit Into Block Structure (Formal Recap)

```sql
DECLARE
    -- declarations
BEGIN
    -- normal execution logic
    -- if ANY statement here raises an exception, execution jumps immediately
    -- to the EXCEPTION section below (skipping the rest of the BEGIN section entirely)
EXCEPTION
    WHEN exception_name THEN
        -- handling logic for this specific exception
    WHEN OTHER_exception_name THEN
        -- handling logic for a different specific exception
    WHEN OTHERS THEN
        -- catch-all handler for anything not explicitly matched above
END;
/
```

We'll cover the actual exception *names* you can `WHEN` against (like `NO_DATA_FOUND`) in the next topic, **Pre-defined Exceptions**. This topic is purely about the **mechanism** — how control flow behaves once an exception occurs.

---

## 5. Simple Examples

### Example 1 — Without exception handling (crashes)
```sql
DECLARE
    v_salary NUMBER;
BEGIN
    SELECT salary INTO v_salary FROM employees WHERE employee_id = 9999;
    DBMS_OUTPUT.PUT_LINE('Salary: ' || v_salary);  -- never reached
END;
/
-- Result: ORA-01403: no data found. Block terminates with an unhandled error.
```

### Example 2 — With exception handling (recovers gracefully)
```sql
DECLARE
    v_salary NUMBER;
BEGIN
    SELECT salary INTO v_salary FROM employees WHERE employee_id = 9999;
    DBMS_OUTPUT.PUT_LINE('Salary: ' || v_salary);
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('No employee found with that ID.');
END;
/
-- Result: prints "No employee found with that ID." — block completes normally, no crash.
```

### Example 3 — Execution jumps immediately; later statements in BEGIN are skipped
```sql
DECLARE
    v_salary NUMBER;
BEGIN
    DBMS_OUTPUT.PUT_LINE('Step 1: Starting lookup...');
    SELECT salary INTO v_salary FROM employees WHERE employee_id = 9999;
    DBMS_OUTPUT.PUT_LINE('Step 2: Lookup succeeded.');  -- SKIPPED if Step above fails
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Step 2 alternative: Handling missing employee.');
END;
/
-- Output:
-- Step 1: Starting lookup...
-- Step 2 alternative: Handling missing employee.
```
Notice: "Step 2" (the success-path message) is **never printed** — the moment the exception is raised, control jumps straight to the `EXCEPTION` section, entirely bypassing whatever remained in the `BEGIN` section.

---

## 6. Detailed Explanation — What's Really Happening

- **An exception is raised, not "checked."** PL/SQL doesn't ask "did the last statement succeed?" after every line — instead, when a statement fails, Oracle **raises** an exception object internally, and the runtime engine actively searches for a handler.
- **The search for a handler happens in the current block first.** If the current block's `EXCEPTION` section has a matching handler, that handler runs, and then **the block is considered to have completed normally** (execution continues *after* the block, not back inside the `BEGIN` section — you cannot "resume" where the error occurred).
- **If no matching handler exists in the current block**, the exception **propagates outward** — to whatever called this block (e.g., if this was a procedure, the exception propagates to the caller of that procedure). This chain continues until either a handler is found somewhere up the call chain, or it reaches the outermost level unhandled, causing a visible error to whatever ultimately invoked the whole chain (an application, a script, a scheduler).
- **Once inside the EXCEPTION section, you cannot go back.** There's no "retry the failed statement" built into basic exception handling — if you want retry behavior, you have to design it explicitly (e.g., wrapping logic in a loop with its own exception handling — a more advanced pattern).

---

## 7. When Exception Handling Is Necessary vs. Optional

**You genuinely need exception handling when:**
- A `SELECT...INTO` might return zero or multiple rows for a given input (very common — almost every lookup by a non-unique or possibly-missing key needs this).
- You're processing external/user-supplied data that could be invalid, missing, or malformed.
- The operation is part of a larger batch/loop process where one bad record shouldn't be allowed to halt everything.
- The business requirement explicitly describes failure scenarios ("...handle cases where the employee doesn't exist...", "...if the account is not found...").

**It's reasonable to skip explicit handling when:**
- The block is a **quick, one-off diagnostic script** where a crash-and-see-the-error behavior is actually fine (e.g., you're debugging and want to see the raw Oracle error).
- You have strong, verified guarantees the failure condition cannot occur (rare in real systems, and often a risky assumption over time as data and code evolve).

---

## 8. Common Mistakes & Misconceptions

1. **Misconception**: "Adding an EXCEPTION section fixes the underlying problem." → No — it only **controls how your program responds** to the problem; it doesn't prevent the problem from happening (e.g., the row genuinely doesn't exist) or fix bad data.
2. **Mistake**: Assuming execution "resumes" after the failed statement once handled. → It does not — the failed statement and everything remaining in the `BEGIN` section after it are **skipped entirely**; execution goes straight to the matching handler, then the block ends.
3. **Mistake**: Wrapping an entire large, multi-step block in one broad exception handler without thinking about **which specific statement** might fail and why — this makes it hard to give a precise, useful response to different failure types (this is exactly why later topics cover specific named exceptions rather than one generic catch-all).
4. **Misconception**: "If I don't write an EXCEPTION section, nothing bad happens as long as my SQL is 'basically correct'." → Real production data is unpredictable; missing rows, duplicate matches, and invalid inputs happen even in "correct" code, especially at scale and over time.
5. **Mistake**: Not realizing an exception in a **called procedure**, if unhandled there, will propagate to the **caller** — meaning the caller's own exception handling (if any) is what ultimately catches it, not necessarily anything inside the failing procedure itself.

---

## 9. Edge Cases to Be Aware Of

- If an exception occurs **inside the DECLARE section itself** (e.g., an invalid initialization expression), it **cannot** be caught by that same block's own `EXCEPTION` section — it propagates immediately to the caller/outer context, since the block's exception-handling machinery isn't considered "active" until execution reaches `BEGIN`.
- If an exception occurs **inside an EXCEPTION handler itself** (yes, this can happen — your handling code can itself fail), that new exception is **not** caught by the same handler section; it propagates outward from that point, just like any other unhandled exception.
- Multiple statements in the `BEGIN` section, and only one fails: everything **before** the failing statement has already executed (and, importantly, any data changes made by prior statements are **not automatically undone** just because a later statement failed — this is a genuinely important point connecting to transaction control, worth remembering even though COMMIT/ROLLBACK specifics sit outside our current syllabus).

---

## 10. Interview-Level / Practical Notes

- A very common interview question: *"What happens to code after the statement that raised an exception?"* — It is **skipped entirely**; control transfers immediately to the exception handler (if one exists) or propagates outward (if not).
- *"Does an EXCEPTION section let you resume execution at the point of failure?"* — No, and this is a frequently-tested misconception; once in the handler, the block is on its way to a controlled, alternate ending, not a "resume from where it broke" flow.
- *"What happens if an exception occurs in a procedure with no EXCEPTION section?"* — It propagates to whoever called that procedure; this is the basis for designing **layered** exception handling strategies in larger systems, which we'll build toward across this module.

---

## Things You Must Remember

- An exception **immediately halts** normal execution in the `BEGIN` section and transfers control to the `EXCEPTION` section (if present).
- Code **after** the failing statement, within the same `BEGIN` section, is **never executed** once an exception is raised.
- If no handler matches in the current block, the exception **propagates outward** to the caller — all the way up the chain until handled or until it reaches the top unhandled.
- Handling an exception does **not** undo already-executed statements automatically — that's a separate transaction-control concern.
- Exceptions in the `DECLARE` section, or inside an exception handler itself, are **not** caught by that same block's own handler.
- Exception handling doesn't fix bad data or root causes — it controls your program's **response** to problems.

## How to Recognize This Concept

You should be thinking "this needs exception handling" whenever a requirement contains language like:
- "...**handle cases where** the data is missing/invalid..."
- "...**if the record doesn't exist**, do X instead..."
- "...**continue processing** the rest even if one record fails..."
- "...**log** any errors encountered..."
- Any `SELECT...INTO` against a table where the lookup key **isn't guaranteed** to exist or be unique.

If a requirement is silent about failure scenarios, that's often **not** a sign exception handling is unnecessary — in real systems, it's usually a sign the requirement is incomplete, and part of your job as a developer is to proactively ask "what should happen if this doesn't exist / fails?"

---

## Exercises

1. **(Predict behavior)** Without running it, describe exactly what happens when this block executes, line by line, assuming `employee_id = 500` does not exist:
   ```sql
   DECLARE
       v_name VARCHAR2(100);
   BEGIN
       DBMS_OUTPUT.PUT_LINE('Looking up employee...');
       SELECT first_name INTO v_name FROM employees WHERE employee_id = 500;
       DBMS_OUTPUT.PUT_LINE('Found: ' || v_name);
   EXCEPTION
       WHEN NO_DATA_FOUND THEN
           DBMS_OUTPUT.PUT_LINE('Employee not found.');
   END;
   /
   ```

2. **(Add handling)** Take this crash-prone block and add an appropriate exception handler so it prints a friendly message instead of crashing:
   ```sql
   DECLARE
       v_dept_name VARCHAR2(50);
   BEGIN
       SELECT department_name INTO v_dept_name FROM departments WHERE department_id = 9999;
       DBMS_OUTPUT.PUT_LINE(v_dept_name);
   END;
   /
   ```

3. **(Propagation reasoning)** A procedure `get_employee_salary` (no exception handling inside it) is called from an anonymous block that **does** have an `EXCEPTION` section. If the procedure's internal `SELECT INTO` fails, where is the exception actually caught — inside the procedure, or in the calling anonymous block? Explain why.

4. **(Multiple statements, partial execution)** Consider:
   ```sql
   BEGIN
       INSERT INTO audit_log (message) VALUES ('Process started');
       UPDATE employees SET salary = salary * 1.1 WHERE employee_id = 9999;  -- succeeds even if 0 rows match, no exception here
       SELECT salary INTO v_salary FROM employees WHERE employee_id = 9999;  -- this one fails, NO_DATA_FOUND
       INSERT INTO audit_log (message) VALUES ('Process completed');
   EXCEPTION
       WHEN NO_DATA_FOUND THEN
           DBMS_OUTPUT.PUT_LINE('Handled the missing employee.');
   END;
   ```
   Which statements actually executed before the exception occurred? Which never ran? What does this imply about the state of the `audit_log` table afterward?

5. **(Realistic business scenario)** Business requirement: *"When processing a customer's refund request, look up their order by order ID. If the order can't be found, log a message indicating the refund couldn't be processed and stop gracefully — don't let the whole batch job crash."* Write a block (or describe the structure, if the full lookup/logging syntax isn't familiar yet) demonstrating how exception handling addresses this requirement, and explain in your own words why an unhandled version would be unacceptable in production.

6. **(Handler-in-handler edge case)** Suppose your exception handler itself contains a statement that could fail (e.g., an `INSERT` into a logging table that happens to violate a constraint). What happens if *that* statement fails? Is it caught by the same `EXCEPTION` section? Explain your reasoning based on what you learned in the Edge Cases section above.

---

*Share your answers whenever you're ready. Next up: Pre-defined Exceptions — where you'll learn the actual named exceptions (like `NO_DATA_FOUND`) available for you to catch specifically.*
