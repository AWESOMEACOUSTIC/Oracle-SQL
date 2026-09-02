# Module 4, Topic 5: Raise Application Error

---

## 1. What Is RAISE_APPLICATION_ERROR?

`RAISE_APPLICATION_ERROR` is a **built-in Oracle procedure** that lets you raise an error with:
- A **custom numeric error code** (in a specific reserved range: **-20000 to -20999**), and
- A **custom, human-readable error message** that you write yourself.

Unlike a plain user-defined exception (Topic 3) — which, when unhandled, surfaces as a fairly generic PL/SQL error referencing just the exception's internal name — `RAISE_APPLICATION_ERROR` produces an error that looks and behaves **exactly like a "real" Oracle error** (e.g., `ORA-20001: Discount cannot exceed 50%`), complete with your own meaningful message, immediately visible to whatever called your code — a calling application, a SQL*Plus session, another PL/SQL block.

This is the tool that finally closes the gap flagged at the end of Topic 3: *"user-defined exceptions have no built-in friendly message."*

---

## 2. Why Does This Exist? What Problem Does It Solve?

Recall Topic 3's leave-balance procedure:
```sql
EXCEPTION
    WHEN e_insufficient_balance THEN
        DBMS_OUTPUT.PUT_LINE('Error: Requested days exceed available leave balance.');
```

This works fine **if** the caller is a human running a script and watching `DBMS_OUTPUT`. But in real systems, the caller of your procedure is very often **not** a human watching console output — it's:
- An **application** (Java, Python, a web service) calling your procedure through a database driver, expecting to catch a **proper database error** it can inspect programmatically (error code + message) and respond to (show a UI message, log it, retry, etc.).
- **Another PL/SQL program**, possibly written by a different team, that has no knowledge of your locally-declared exception name and needs a standard way to detect "something went wrong here, and here's why."

If your business rule violation is only ever communicated via a `DBMS_OUTPUT.PUT_LINE` inside your own handler, **that message never reaches the calling application at all** — it's just printed to a buffer nobody's application code is reading. Worse, if you don't re-raise anything, the procedure might appear to the caller to have **completed successfully**, even though the business rule was actually violated and nothing happened.

`RAISE_APPLICATION_ERROR` solves this by producing a **genuine, structured, propagating Oracle error** — with your own custom code and message — that any caller (application or PL/SQL) can detect and inspect through the **normal, standard error-handling mechanisms** they already use for every other Oracle error, no special knowledge of your internal exception names required.

---

## 3. Why Is It Used? (The Business Case)

- **Application-facing error messages**: when a business rule is violated, the calling application needs a **clear, specific, catchable error** it can translate into a meaningful message for an end user (e.g., "Your requested discount exceeds the maximum allowed") rather than a cryptic generic failure.
- **Consistent error codes across a system**: assigning specific numbers (within -20000 to -20999) to specific business rule violations lets calling systems programmatically distinguish *which* rule was violated (by checking the code), not just that *some* error occurred.
- **Stopping execution cleanly with a clear reason**: `RAISE_APPLICATION_ERROR` immediately halts execution (like any raised exception) — ensuring a violated business rule **never silently allows an operation to proceed**, and the reason is unambiguous to whoever is looking at the resulting error.
- **Professional-grade error reporting**: real production systems are expected to surface meaningful errors up the entire call chain — this procedure is the standard, idiomatic Oracle mechanism for doing exactly that.

---

## 4. Syntax

```sql
RAISE_APPLICATION_ERROR (error_number, error_message [, keep_errors]);
```

| Parameter | Meaning |
|---|---|
| `error_number` | A negative integer between **-20000 and -20999** (this specific range is **reserved by Oracle for custom application errors** — you cannot use numbers outside this range). |
| `error_message` | A `VARCHAR2` string, up to 2048 bytes — your custom, human-readable message. |
| `keep_errors` (optional, `TRUE`/`FALSE`, defaults to `FALSE`) | If `TRUE`, this error is **added** to the list of errors already raised in this call (useful in nested error scenarios); if `FALSE` (the default), it **replaces** any previous error list. In most everyday usage, you'll leave this at its default and not specify it at all. |

### Where It's Called From
`RAISE_APPLICATION_ERROR` is typically called **directly at the point of a business rule violation** (often replacing a plain `RAISE user_defined_exception;`), or from **within an exception handler**, to translate a caught exception into a clean, custom-coded error before it propagates further.

---

## 5. Simple Examples

### Example 1 — Direct use, replacing a plain RAISE
```sql
CREATE OR REPLACE PROCEDURE apply_discount (p_order_id IN NUMBER, p_discount_pct IN NUMBER)
IS
BEGIN
    IF p_discount_pct > 50 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Discount cannot exceed 50%.');
    END IF;

    UPDATE orders SET discount_pct = p_discount_pct WHERE order_id = p_order_id;
END apply_discount;
/
```

Calling this with an invalid discount:
```sql
BEGIN
    apply_discount(101, 75);
END;
/
-- Result: ORA-20001: Discount cannot exceed 50%.
-- ORA-06512: at "apply_discount", line X
```

Notice: **no `EXCEPTION` section was even needed inside `apply_discount`** for this to work — `RAISE_APPLICATION_ERROR` immediately raises a real, propagating error; the calling block above has no handler either, so it surfaces all the way up, exactly like any other unhandled Oracle error.

### Example 2 — Combined with a user-defined exception and a handler (translating a caught exception into a clean error)
```sql
CREATE OR REPLACE PROCEDURE submit_leave_request (p_emp_id IN NUMBER, p_days IN NUMBER)
IS
    e_insufficient_balance EXCEPTION;
    v_available_days       NUMBER;
BEGIN
    SELECT leave_balance INTO v_available_days FROM employees WHERE employee_id = p_emp_id;

    IF p_days > v_available_days THEN
        RAISE e_insufficient_balance;
    END IF;

    UPDATE employees SET leave_balance = leave_balance - p_days WHERE employee_id = p_emp_id;

EXCEPTION
    WHEN e_insufficient_balance THEN
        RAISE_APPLICATION_ERROR(-20002, 'Requested leave days exceed available balance.');
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20003, 'Employee not found: ' || p_emp_id);
END submit_leave_request;
/
```

This is an extremely common and important **pattern**: catch a specific exception (pre-defined or user-defined) internally with `WHEN`, and **inside that handler**, call `RAISE_APPLICATION_ERROR` to convert it into a clean, custom-coded, well-messaged error before it leaves this procedure — giving the caller a consistent, professional error interface, regardless of what internal exception mechanism actually detected the problem.

### Example 3 — Different codes for different rules (enabling programmatic distinction)
```sql
CREATE OR REPLACE PROCEDURE process_withdrawal (p_account_id IN NUMBER, p_amount IN NUMBER)
IS
    v_balance NUMBER;
BEGIN
    IF p_amount <= 0 THEN
        RAISE_APPLICATION_ERROR(-20010, 'Withdrawal amount must be positive.');
    END IF;

    SELECT balance INTO v_balance FROM accounts WHERE account_id = p_account_id;

    IF p_amount > v_balance THEN
        RAISE_APPLICATION_ERROR(-20011, 'Insufficient funds for this withdrawal.');
    END IF;

    UPDATE accounts SET balance = balance - p_amount WHERE account_id = p_account_id;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20012, 'Account not found: ' || p_account_id);
END process_withdrawal;
/
```
A calling application can now catch this error and check **specifically** whether it received `-20010`, `-20011`, or `-20012` — and respond differently to each (different UI messages, different retry logic, etc.) — something a single generic `WHEN OTHERS` message could never offer.

---

## 6. Detailed Explanation

- `RAISE_APPLICATION_ERROR` is part of the built-in `DBMS_STANDARD` package, but is available for direct use without any qualification — you just call it by name.
- Once called, it **behaves exactly like any raised exception**: it immediately halts execution at that point, skips the rest of the current `BEGIN` section, and either gets caught by a `WHEN OTHERS` handler in the *same* block (if you want to catch and further process your own raised error — unusual but possible) or propagates outward to the caller, exactly per the propagation rules from Topic 1.
- The resulting error, if unhandled anywhere up the chain, ultimately surfaces to whatever originally invoked the whole chain — an application's error-handling code, a script, a scheduler log — as a standard Oracle error with **your** code and **your** message, just like a native `ORA-xxxxx` error.
- The error number range (**-20000 to -20999**) is **reserved specifically for this purpose** by Oracle — using numbers outside this range in `RAISE_APPLICATION_ERROR` raises its own error, since it would risk colliding with Oracle's own internal error number space.

---

## 7. When to Use / When Not to Use

**Use `RAISE_APPLICATION_ERROR` when:**
- The error needs to be **meaningfully communicated to a calling application or external caller**, not just logged to console output for a human watching a script run.
- You want a **consistent, professional error code scheme** across your system's business rules, so different violations can be distinguished programmatically.
- You're inside an exception handler and want to **translate** an internal/technical exception (pre-defined or user-defined) into a clean, business-appropriate message before it propagates further — hiding internal implementation detail from external callers.

**Don't reach for it when:**
- You're writing a **quick, throwaway diagnostic script** for your own use, where `DBMS_OUTPUT.PUT_LINE` inside a handler is perfectly sufficient and no external caller is involved.
- The condition isn't actually an error at all (see Topic 3's "don't overuse exceptions for routine outcomes" guidance) — `RAISE_APPLICATION_ERROR` should represent genuine failures/rule violations, not expected, everyday branching.

---

## 8. Common Mistakes & Misconceptions

1. **Mistake**: Using an error number **outside** the -20000 to -20999 range → Oracle itself raises an error, since that range is reserved.
2. **Misconception**: "RAISE_APPLICATION_ERROR replaces the need for a DECLARE'd user-defined exception entirely." → Not necessarily — many well-designed procedures still **declare and RAISE a user-defined exception first** (for clean internal control flow and self-documenting `IF`/`RAISE` logic), then call `RAISE_APPLICATION_ERROR` **inside the corresponding handler** to produce the final, caller-facing error — this layered pattern (seen in Example 2 above) is very common and considered good practice, not redundant.
3. **Mistake**: Reusing the **same error number** for multiple, genuinely different business rule violations — this defeats the purpose of giving callers the ability to programmatically distinguish between different failure reasons.
4. **Misconception**: "Calling RAISE_APPLICATION_ERROR inside an EXCEPTION handler is the same as just re-raising the original exception." → It is meaningfully different — the original exception (e.g., `NO_DATA_FOUND`) had Oracle's own generic message and error number; `RAISE_APPLICATION_ERROR` lets you **replace** that with your own, more specific and business-appropriate code and message, which is exactly its value.
5. **Mistake**: Forgetting that once `RAISE_APPLICATION_ERROR` executes, the rest of the current block's code (including anything after it in the same handler or `BEGIN` section) does **not** run — same halting behavior as any other raised exception.

---

## 9. Edge Cases to Be Aware Of

- Calling `RAISE_APPLICATION_ERROR` from **within an EXCEPTION handler** (as in Example 2) is extremely common — and it's worth noting this **new** error becomes what propagates onward, effectively **replacing** the original exception's identity from the caller's perspective (the caller sees your custom `-20002` error, not the original `NO_DATA_FOUND`/`ORA-01403` unless you deliberately included that detail in your custom message).
- If you want to **preserve some detail from the original exception** in your custom message (e.g., include the actual employee ID that wasn't found), you must explicitly build that into your `error_message` string yourself — `RAISE_APPLICATION_ERROR` doesn't automatically carry forward information from whatever exception you originally caught.
- Consistently documenting your organization's **error number scheme** (e.g., "-20001 to -20099 reserved for HR module, -20100 to -20199 for Finance module") is a real, common practice in larger systems — worth being aware of as a design convention, even though it's not enforced by Oracle itself.

---

## 10. Interview-Level / Practical Notes

- A very common interview question: *"How do you raise a custom error message from PL/SQL that an external application can properly catch?"* — `RAISE_APPLICATION_ERROR` is the textbook, expected answer.
- *"What's the valid error number range for RAISE_APPLICATION_ERROR, and why is it restricted?"* — **-20000 to -20999**, reserved specifically to avoid collision with Oracle's own internal error number space.
- *"Why might you declare a user-defined exception AND use RAISE_APPLICATION_ERROR together, rather than just using one or the other?"* — The user-defined exception gives clean, readable internal control flow (`RAISE e_something;` inside an `IF`); `RAISE_APPLICATION_ERROR` (called from the corresponding handler) gives the **external, caller-facing** representation of that same failure, with a proper code and message. This layered design — being able to explain **why** both exist together — is a strong signal of real practical understanding.

---

## Things You Must Remember

- Syntax: `RAISE_APPLICATION_ERROR(error_number, error_message);` — error number **must** be between **-20000 and -20999**.
- It produces a **real, propagating Oracle-style error** (code + message), fully visible and catchable by external applications — unlike a plain `DBMS_OUTPUT.PUT_LINE` inside a handler, which never reaches a calling application at all.
- Common, high-quality pattern: declare + raise a **user-defined exception** for clean internal logic, then call `RAISE_APPLICATION_ERROR` **inside its handler** to produce the final, business-appropriate external error.
- Different business rules should generally get **different, consistent error numbers**, so callers can distinguish and respond to them programmatically.
- Once called, execution halts immediately — identical propagation/skip behavior to any other raised exception.
- It doesn't automatically preserve details from an originally-caught exception — build any needed detail into your own message string explicitly.

## How to Recognize This Concept

Reach for `RAISE_APPLICATION_ERROR` whenever a requirement implies the error needs to be **visible and meaningful beyond your own script** — language like:
- "...the **application** should show the user a message saying..."
- "...**return** a clear error that the calling system can handle..."
- "...different failures should be **distinguishable**..." (implying distinct codes)
- Any requirement describing a procedure that will be called by **other systems/applications**, not just run interactively by a developer.

If the context is a small, self-contained diagnostic script with no external caller depending on structured error information, a plain `DBMS_OUTPUT.PUT_LINE` inside a handler may still be perfectly adequate — `RAISE_APPLICATION_ERROR`'s value is specifically about **communicating cleanly beyond the current block**.

---

## Exercises

1. **(Basic usage)** Rewrite this plain user-defined exception example (from Topic 3) to use `RAISE_APPLICATION_ERROR` instead of a plain `DBMS_OUTPUT.PUT_LINE` in the handler, using error number `-20005`:
   ```sql
   DECLARE
       e_negative_amount EXCEPTION;
       v_amount NUMBER := -50;
   BEGIN
       IF v_amount < 0 THEN
           RAISE e_negative_amount;
       END IF;
   EXCEPTION
       WHEN e_negative_amount THEN
           DBMS_OUTPUT.PUT_LINE('Amount cannot be negative.');
   END;
   /
   ```

2. **(Direct use, no separate exception declared)** Write a procedure `set_employee_bonus_pct` that accepts a `p_bonus_pct` parameter and immediately raises an appropriate `RAISE_APPLICATION_ERROR` (choose a suitable code) if the value is negative or greater than 100 — without declaring a separate named exception first.

3. **(Layered pattern)** Write a procedure `cancel_order` that accepts an `order_id`. It should look up the order's status; if the status is already `'CANCELLED'`, raise a user-defined exception `e_already_cancelled` internally, then in the handler, convert it to a `RAISE_APPLICATION_ERROR` with a clear message and an appropriate code. Also handle `NO_DATA_FOUND` similarly, with a different code.

4. **(Distinct codes reasoning)** Explain, in 2–3 sentences, why using the **same** error number for both "order not found" and "order already cancelled" in Exercise 3 would be a poor design choice, even if both messages are worded differently.

5. **(Invalid range)** What happens if a developer writes `RAISE_APPLICATION_ERROR(-500, 'Some error');`? Why specifically does this fail?

6. **(Realistic business scenario)** Business requirement: *"Our order-processing procedure, `submit_order`, is called by three different systems: our website, our mobile app, and a partner company's integration API. All three need to reliably detect and display a proper message when a business rule is violated (e.g., quantity exceeds stock, customer account is suspended). Each distinct rule violation needs its own consistent, documented error code so all three calling systems can handle them appropriately, potentially differently from each other."* Design (signature + illustrative body) the `submit_order` procedure, using at least two distinct business rules, each with its own dedicated error number and message via `RAISE_APPLICATION_ERROR`. Briefly explain why this scenario — multiple, independent external callers — makes `RAISE_APPLICATION_ERROR` clearly the right tool, rather than just `DBMS_OUTPUT.PUT_LINE`.

---

*This completes Module 4: PL/SQL Exception, in full. Share your answers whenever you're ready. Once you're comfortable here, let me know and we can either loop back to the remaining Module 1 and Module 2 topics (as originally planned), or run a mixed assessment covering everything from Modules 3 and 4 first — your call.*
