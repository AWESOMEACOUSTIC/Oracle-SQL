# Module 4, Topic 2: Pre-defined Exceptions

---

## 1. What Is a Pre-defined Exception?

A **pre-defined exception** is an exception that Oracle **already knows about, has already named, and automatically raises** for a specific set of common runtime error conditions — without you having to do anything to "create" or "trigger" it yourself. You just write a `WHEN <exception_name> THEN` clause using its standard name, and Oracle handles matching it whenever that specific error condition occurs.

Contrast this with **user-defined exceptions** (next topic), which you must explicitly declare and explicitly raise yourself for business-specific error conditions Oracle has no built-in awareness of.

---

## 2. Why Do Pre-defined Exceptions Exist? What Problem Do They Solve?

In Topic 1, you saw this pattern:
```sql
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        ...
```

Without pre-defined exceptions, you'd have to know Oracle's raw numeric error codes (`ORA-01403`, `ORA-01422`, etc.) and match against them manually — memorizing numbers, which is error-prone and unreadable. Pre-defined exceptions give **meaningful, memorable names** to the small set of error conditions that come up **extremely frequently** in normal PL/SQL work — so your code reads as `WHEN NO_DATA_FOUND` (self-explanatory) rather than `WHEN raw_error_code = -1403` (opaque).

They exist specifically because certain failure conditions are **so common** (a lookup finding no rows, a lookup finding too many rows, dividing by zero, a NULL used where a value was required) that Oracle pre-declares and pre-names them for you, saving every developer from having to redeclare the same handful of "obvious" errors from scratch in every single program.

---

## 3. The Most Important Pre-defined Exceptions

| Exception Name | Raised When | Oracle Error |
|---|---|---|
| `NO_DATA_FOUND` | A `SELECT INTO` returns **zero rows**. | ORA-01403 |
| `TOO_MANY_ROWS` | A `SELECT INTO` returns **more than one row** (when only one variable/row was expected). | ORA-01422 |
| `ZERO_DIVIDE` | An arithmetic operation attempts to **divide by zero**. | ORA-01476 |
| `VALUE_ERROR` | A conversion, truncation, or arithmetic/constraint error — e.g., trying to fit a value into a variable too small for it, or an invalid implicit type conversion. | ORA-06502 |
| `INVALID_NUMBER` | A string fails to convert to a valid number (e.g., in a SQL context). | ORA-01722 |
| `DUP_VAL_ON_INDEX` | An `INSERT`/`UPDATE` attempts to create a **duplicate value** in a column protected by a unique index/constraint. | ORA-00001 |
| `CURSOR_ALREADY_OPEN` | An attempt to open a cursor that is already open. *(Cursors aren't in our syllabus — mentioned here only for completeness of this reference list; not something you need to act on yet.)* | ORA-06511 |
| `INVALID_CURSOR` | An illegal cursor operation. *(Same note as above.)* | ORA-01001 |
| `LOGIN_DENIED` | Invalid username/password when connecting. | ORA-01017 |
| `NOT_LOGGED_ON` | Attempting a database operation without an active connection. | ORA-01012 |
| `STORAGE_ERROR` | PL/SQL runs out of memory or memory is corrupted. | ORA-06500 |
| `PROGRAM_ERROR` | An internal PL/SQL error. | ORA-06501 |
| `TIMEOUT_ON_RESOURCE` | Timeout while waiting for a resource. | ORA-00051 |

**For our syllabus and realistic day-to-day business work, the ones you will use constantly are: `NO_DATA_FOUND`, `TOO_MANY_ROWS`, `ZERO_DIVIDE`, `VALUE_ERROR`, and `DUP_VAL_ON_INDEX`.** The rest are worth recognizing by name if you encounter them, but far less frequently hand-written.

---

## 4. Syntax

```sql
BEGIN
    -- statement(s) that might raise a pre-defined exception
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        -- handle this specific case
    WHEN TOO_MANY_ROWS THEN
        -- handle this different specific case
    WHEN ZERO_DIVIDE THEN
        -- handle this different specific case
    WHEN OTHERS THEN
        -- catch-all for anything not explicitly named above
END;
/
```

### Key Syntax Rules
- You can list **multiple `WHEN` clauses**, each handling a **different** exception, in the same `EXCEPTION` section.
- A single `WHEN` clause can handle **multiple exceptions together** using `OR`:
  ```sql
  WHEN NO_DATA_FOUND OR TOO_MANY_ROWS THEN
      DBMS_OUTPUT.PUT_LINE('Lookup did not return exactly one row.');
  ```
- `WHEN OTHERS` is a special **catch-all** clause that matches **any** exception not already matched by an earlier, more specific `WHEN` clause. If present, it should always be the **last** clause (Oracle raises a compile error if you put anything after `WHEN OTHERS`, since nothing could ever be reached after a catch-all).
- Only **one** `WHEN` clause runs per exception occurrence — Oracle checks them top-to-bottom and stops at the first match.

---

## 5. Simple Examples

### Example 1 — Handling `NO_DATA_FOUND` and `TOO_MANY_ROWS` distinctly
```sql
DECLARE
    v_salary NUMBER;
BEGIN
    SELECT salary INTO v_salary FROM employees WHERE department_id = 10;  -- might match 0, 1, or many rows
    DBMS_OUTPUT.PUT_LINE('Salary: ' || v_salary);
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('No employees found in that department.');
    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('More than one employee found — query needs refining.');
END;
/
```

### Example 2 — `ZERO_DIVIDE`
```sql
DECLARE
    v_avg NUMBER;
    v_total NUMBER := 1000;
    v_count NUMBER := 0;
BEGIN
    v_avg := v_total / v_count;
    DBMS_OUTPUT.PUT_LINE('Average: ' || v_avg);
EXCEPTION
    WHEN ZERO_DIVIDE THEN
        DBMS_OUTPUT.PUT_LINE('Cannot calculate average: division by zero.');
END;
/
```

### Example 3 — `DUP_VAL_ON_INDEX`
```sql
BEGIN
    INSERT INTO departments (department_id, department_name)
    VALUES (10, 'Finance');  -- assume department_id 10 already exists (unique constraint)
EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
        DBMS_OUTPUT.PUT_LINE('A department with this ID already exists.');
END;
/
```

### Example 4 — Combining exceptions in one handler, plus a catch-all
```sql
DECLARE
    v_result NUMBER;
BEGIN
    SELECT salary / (salary - salary) INTO v_result FROM employees WHERE employee_id = 100;
EXCEPTION
    WHEN NO_DATA_FOUND OR TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('Lookup issue: not exactly one matching row.');
    WHEN ZERO_DIVIDE THEN
        DBMS_OUTPUT.PUT_LINE('Calculation issue: division by zero.');
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('An unexpected error occurred.');
END;
/
```

---

## 6. Detailed Explanation — Matching Behavior

- Oracle evaluates `WHEN` clauses **top-to-bottom**, in the order you wrote them, and executes **only the first one that matches**. Order matters — put more specific handlers before more general ones (and `WHEN OTHERS` last, always).
- If none of your named `WHEN` clauses match the actual exception raised, and there's **no `WHEN OTHERS`**, the exception is **not** considered handled by this block — it propagates outward exactly as described in Topic 1, even though *an* `EXCEPTION` section exists (having an exception section doesn't mean everything is caught — only what's actually matched).
- `WHEN OTHERS` is powerful but should be used **thoughtfully**, not as a lazy blanket "catch everything and move on" — because it also silently swallows exceptions you *didn't* anticipate, including genuine bugs, unless you handle it carefully (this becomes especially important once we reach `RAISE` and `RAISE_APPLICATION_ERROR` later in this module, which let you log the real error and/or re-raise it rather than silently suppressing it).

---

## 7. When to Use Pre-defined Exceptions (and When You'll Need More)

**Pre-defined exceptions are the right tool when:**
- The failure condition is a **generic, Oracle-recognized runtime error** — a missing row, too many rows, a math error, a constraint violation.
- You want clean, readable, self-documenting handler code instead of raw numeric error codes.

**They are NOT enough when:**
- The failure condition is a **business rule violation** that Oracle has no built-in concept of — e.g., "an employee's requested leave exceeds their available balance." Oracle doesn't know or care about your leave-balance business rule; there's no pre-defined exception for it. This is exactly the gap **User-Defined Exceptions** (next topic) exist to fill.

---

## 8. Common Mistakes & Misconceptions

1. **Mistake**: Assuming `WHEN OTHERS` alone is sufficient exception handling everywhere. → It catches *everything*, but gives you no ability to respond **differently** to different failure types — losing valuable diagnostic precision, and often masking real bugs by treating all failures identically.
2. **Mistake**: Putting `WHEN OTHERS` **before** more specific handlers. → This is not just bad style — the more specific handlers become **unreachable dead code**, since `WHEN OTHERS` would catch everything first. Oracle will actually raise a compile warning/error for unreachable handlers placed after `WHEN OTHERS`.
3. **Misconception**: "NO_DATA_FOUND only applies to SELECT INTO." → It specifically applies to implicit single-row queries (`SELECT INTO`) — it's not a general-purpose "nothing happened" signal; for example, an `UPDATE`/`DELETE` that matches zero rows does **not** raise `NO_DATA_FOUND` — it simply succeeds silently, having updated/deleted zero rows. This is a very commonly misunderstood distinction.
4. **Misconception**: "TOO_MANY_ROWS means my query is wrong." → It just means a `SELECT INTO` (which expects exactly one row into a scalar variable) received more than one — the query itself might be perfectly correct; it might just need a different retrieval approach (e.g., aggregate function, or a cursor — outside current syllabus) if multiple rows are legitimately possible.
5. **Mistake**: Forgetting that `DUP_VAL_ON_INDEX` requires an actual **unique constraint or unique index** on the relevant column(s) — inserting a "duplicate-looking" value into a column with no such constraint won't raise this exception at all; it'll just succeed.

---

## 9. Edge Cases to Be Aware Of

- **`UPDATE`/`DELETE` matching zero rows does NOT raise `NO_DATA_FOUND`.** This surprises many learners coming from the `SELECT INTO` behavior. If you need to know whether an `UPDATE`/`DELETE` affected any rows, you check `SQL%ROWCOUNT` (a system attribute) — worth knowing exists, even though implicit cursor attributes in depth sit outside our current syllabus scope.
- A `SELECT INTO` that would match **more than one row**, but you only asked for **aggregate functions** (like `COUNT(*)`, `SUM(...)`) — these never raise `TOO_MANY_ROWS`, because aggregates always collapse to a single row/value by definition, regardless of how many underlying rows they summarize.
- `VALUE_ERROR` can be raised by a surprisingly wide range of situations — numeric/string conversion issues, assigning a value too large for a variable's declared size, and more — it's a bit of a "broad" pre-defined exception compared to the very specific `NO_DATA_FOUND`/`TOO_MANY_ROWS`.

---

## 10. Interview-Level / Practical Notes

- A very common interview question: *"What's the difference between how SELECT INTO and UPDATE handle zero matching rows?"* — `SELECT INTO` raises `NO_DATA_FOUND`; `UPDATE`/`DELETE` simply affects zero rows silently, with no exception. This distinction is frequently tested because it's genuinely easy to get wrong.
- *"Why should WHEN OTHERS always be last?"* — Because it's a catch-all; anything placed after it would be unreachable, and Oracle enforces this ordering.
- *"Name three pre-defined exceptions and when they occur"* — a very standard baseline interview question; `NO_DATA_FOUND`, `TOO_MANY_ROWS`, and `ZERO_DIVIDE` are the safest, most universally expected answers.

---

## Things You Must Remember

- Pre-defined exceptions have **standard, memorable names** — no need to declare them yourself; Oracle raises them automatically for their specific conditions.
- The essentials to know cold: `NO_DATA_FOUND`, `TOO_MANY_ROWS`, `ZERO_DIVIDE`, `VALUE_ERROR`, `DUP_VAL_ON_INDEX`.
- `WHEN OTHERS` must always be the **last** clause — it's a catch-all, and anything after it is unreachable.
- `WHEN` clauses are checked **top-to-bottom**; only the **first** match runs.
- `UPDATE`/`DELETE` affecting **zero rows** does **not** raise `NO_DATA_FOUND` — that exception is specific to `SELECT INTO`.
- Aggregate functions (`COUNT`, `SUM`, etc.) never raise `TOO_MANY_ROWS`, since they always return exactly one summarized value.
- Pre-defined exceptions cover **generic runtime errors only** — business-specific rules need **user-defined exceptions** (next topic).

## How to Recognize This Concept

Reach for a **pre-defined exception handler** whenever you're writing:
- Any `SELECT ... INTO` where the row **might not exist**, or **might match more than one row** → anticipate `NO_DATA_FOUND` / `TOO_MANY_ROWS`.
- Any calculation involving **division** where the denominator could plausibly be zero → anticipate `ZERO_DIVIDE`.
- Any `INSERT`/`UPDATE` touching a column with a **unique constraint** where duplicates are a realistic possibility → anticipate `DUP_VAL_ON_INDEX`.
- Any conversion between types (string-to-number, fitting a value into a constrained variable) where the input **isn't fully trusted/validated** → anticipate `VALUE_ERROR` / `INVALID_NUMBER`.

If the failure condition you're worried about is a **business rule** rather than a generic Oracle runtime condition (e.g., "this discount can't exceed 50%"), that's your cue that pre-defined exceptions alone won't be enough — keep reading into the next topic.

---

## Exercises

1. **(Basic NO_DATA_FOUND)** Write a block that looks up a department name by `department_id` (assume the table `departments(department_id, department_name)`), handling the case where the ID doesn't exist with a clear message.

2. **(TOO_MANY_ROWS)** Write a block that attempts `SELECT salary INTO ... FROM employees WHERE department_id = 10` (assume multiple employees share this department), and handle `TOO_MANY_ROWS` appropriately.

3. **(ZERO_DIVIDE)** Write a block that calculates `total_sales / number_of_transactions` for a report, handling the case where `number_of_transactions` might be zero.

4. **(Combine exceptions in one handler)** Write a block with a `SELECT INTO` that could plausibly raise either `NO_DATA_FOUND` or `TOO_MANY_ROWS`, and handle **both together** in a single `WHEN` clause with one shared message.

5. **(Ordering judgment)** Explain what's wrong with this exception section, and reorder it correctly:
   ```sql
   EXCEPTION
       WHEN OTHERS THEN
           DBMS_OUTPUT.PUT_LINE('Something went wrong.');
       WHEN NO_DATA_FOUND THEN
           DBMS_OUTPUT.PUT_LINE('No data found specifically.');
   ```

6. **(UPDATE edge case reasoning)** A developer writes:
   ```sql
   BEGIN
       UPDATE employees SET salary = salary * 1.1 WHERE employee_id = 99999;  -- doesn't exist
   EXCEPTION
       WHEN NO_DATA_FOUND THEN
           DBMS_OUTPUT.PUT_LINE('Employee not found — raise handled.');
   END;
   /
   ```
   They expect the "Employee not found" message to print. Explain why it **won't**, based on what you learned about `UPDATE` behavior in the Edge Cases section, and what the block will actually do instead.

7. **(Realistic business scenario)** Business requirement: *"When processing a bulk employee-ID upload, look up each employee's current salary. If an employee ID doesn't exist, log it as an error and continue with the next one instead of stopping the whole batch. If somehow more than one employee matches the same ID (a data integrity issue), that should be flagged as a more serious problem requiring immediate attention, distinct from a simple 'not found' case."* Explain how you'd structure the exception handling here — which specific exceptions you'd catch, and why you'd treat them differently rather than using one single `WHEN OTHERS` for everything.

---

*Share your answers whenever you're ready. Next up: User Defined Exception — where you'll learn how to raise your own business-specific errors that Oracle has no built-in concept of.*
