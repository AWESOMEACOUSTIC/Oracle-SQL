# Topic 4: Cursors in FOR Loop

> **Syllabus position:** Topic 4 of 6 (+ final combined practice set)
> **Builds on:** Topic 1 (lifecycle, `%ROWTYPE`), Topic 2 (parameterized cursors), Topic 3 (decision-making)
> **Feeds into:** Topic 5 (`FOR UPDATE` / `WHERE CURRENT OF` work inside FOR loops too), Topic 6 (REF CURSORs, which — spoiler — are *not* used with this exact syntax)

---

## 1. Concept

A **cursor FOR loop** is a looping construct that automatically performs the entire cursor lifecycle from Topic 1 — `OPEN`, repeated `FETCH`, the `EXIT WHEN %NOTFOUND` check, and `CLOSE` — without you writing any of those statements yourself. It also automatically declares the loop's record variable for you, shaped like `cursor_name%ROWTYPE`, so you don't declare that either.

```sql
FOR emp_rec IN c_emp LOOP
   DBMS_OUTPUT.PUT_LINE(emp_rec.first_name);
END LOOP;
```
This one loop replaces the entire `DECLARE v_...; OPEN; LOOP FETCH; EXIT WHEN; END LOOP; CLOSE;` pattern from Topic 1 — Oracle does all of it behind the scenes.

## 2. Purpose / Why It Exists

Nearly every mistake covered in Topics 1–3 involving manual cursor management — forgetting `CLOSE`, misplacing `EXIT WHEN`, creating an infinite loop by omitting `EXIT WHEN` entirely, leaving a cursor open across loop iterations and hitting `ORA-06511` — stems from the same root cause: manual lifecycle management has several steps, and missing any one of them breaks something. The cursor FOR loop exists to remove that entire class of bugs by making the lifecycle **automatic and guaranteed**, including in situations manual code often gets wrong, like early exits or unhandled exceptions (detailed below). It's also simply less to type and less to read — which is precisely why it's the default idiom in real production PL/SQL for straightforward row-by-row processing.

## 3. What Real-World Problem It Solves

The vast majority of "iterate over a result set and do something with each row" requirements don't need fine-grained control over *when* each fetch happens — they just need "for every row, do X." The cursor FOR loop is built exactly for that overwhelming majority case, letting you write business logic without re-deriving lifecycle boilerplate every single time.

## 4. When to Use / When NOT to Use

**Use a cursor FOR loop when:**
- You need to process every row from start to end, one at a time, with no special fetch-timing requirements. This covers most real cursor use cases, including the nested master-detail pattern from Topics 2–3.
- You want the safety guarantee that the cursor will always be closed, even if you exit early or an exception occurs.

**Use manual `OPEN`/`FETCH`/`CLOSE` instead when:**
- You need non-standard fetch control — e.g., fetching two rows at once to compare "current vs. previous," or skipping fetches conditionally rather than every iteration.
- The cursor's `OPEN`, `FETCH`, and `CLOSE` genuinely need to happen in **different subprograms** or at different points in a larger flow — a FOR loop is a single, self-contained statement and can't be split apart like that.
- You need to inspect cursor attributes (like `%ROWCOUNT`) with fine control mid-processing in ways that don't fit neatly inside the loop body (rare, but possible — the standard case of checking `%ROWCOUNT` *inside* the loop body is actually still fully supported with a **named** cursor FOR loop; see Exercise 6).

**Recognition clue:** if a requirement is "go through every X and do Y" with no mention of needing to pause, resume, batch, or interleave fetches with unrelated logic across procedure boundaries — default to a cursor FOR loop. Reach for manual management only when the FOR loop's fixed structure genuinely can't express what's needed.

## 5. Syntax

There are two forms:

**Form 1 — named cursor (declared separately, with or without parameters):**
```sql
FOR record_name IN cursor_name[(parameter1, parameter2, ...)] LOOP
   -- use record_name.column_name
END LOOP;
```

**Form 2 — inline (implicit) cursor, no separate `CURSOR` declaration at all:**
```sql
FOR record_name IN (SELECT column1, column2 FROM table_name WHERE condition) LOOP
   -- use record_name.column_name
END LOOP;
```

### Syntax Breakdown

- `record_name` is **implicitly declared by the loop itself** — you do not declare it in a `DECLARE` section. Its structure matches `cursor_name%ROWTYPE` (Form 1) or the inline query's column list (Form 2).
- `record_name`'s scope is **limited to the loop body**. It does not exist before the loop starts or after `END LOOP` — referencing it outside the loop is a compile error, since the identifier is simply not declared there.
- No `OPEN`, `FETCH`, `EXIT WHEN`, or `CLOSE` statements are written — all handled automatically, and this is not optional or overridable; you cannot mix manual lifecycle statements into a cursor that's driving a FOR loop.
- The loop terminates automatically once the active set is exhausted, and the cursor is **guaranteed to be closed** by the time control leaves the loop — whether that's normal completion, an `EXIT` inside the loop, a `RETURN` (inside a function/procedure), or an exception propagating out of the loop body.

## 6. Types / Variations

| Variation | Description |
|---|---|
| **Named cursor, no parameters** | `FOR rec IN c_emp LOOP ... END LOOP;` |
| **Named cursor, with parameters** | `FOR rec IN c_emp(30) LOOP ... END LOOP;` — the standard idiom for reusable/nested cursors from Topics 2–3. |
| **Inline (implicit) cursor** | `FOR rec IN (SELECT ... FROM ... WHERE ...) LOOP ... END LOOP;` — fastest to write for a one-off, non-reused query; no `DECLARE` section needed if nothing else in the block requires one. |
| **Nested cursor FOR loops** | An outer FOR loop containing an inner FOR loop, with the inner query filtered using the outer loop's current row values — full automatic open/close at *both* levels. |

## 7. Simple Examples

### Example A — Named cursor, no parameters
```sql
DECLARE
   CURSOR c_emp IS
      SELECT employee_id, first_name, salary FROM employees WHERE department_id = 30;
BEGIN
   FOR emp_rec IN c_emp LOOP
      DBMS_OUTPUT.PUT_LINE(emp_rec.first_name || ' - ' || emp_rec.salary);
   END LOOP;
END;
/
```

### Example B — Named cursor, with a parameter
```sql
DECLARE
   CURSOR c_emp (p_dept_id employees.department_id%TYPE) IS
      SELECT first_name, salary FROM employees WHERE department_id = p_dept_id;
BEGIN
   FOR emp_rec IN c_emp(30) LOOP
      DBMS_OUTPUT.PUT_LINE(emp_rec.first_name);
   END LOOP;
END;
/
```

### Example C — Inline cursor, no `DECLARE` section at all
```sql
BEGIN
   FOR emp_rec IN (SELECT first_name, salary FROM employees WHERE department_id = 30) LOOP
      DBMS_OUTPUT.PUT_LINE(emp_rec.first_name);
   END LOOP;
END;
/
```

### Example D — Nested cursor FOR loops (master-detail, fully automatic)
```sql
BEGIN
   FOR dept_rec IN (SELECT department_id, department_name FROM departments) LOOP
      DBMS_OUTPUT.PUT_LINE('Dept: ' || dept_rec.department_name);

      FOR emp_rec IN (SELECT first_name FROM employees
                       WHERE department_id = dept_rec.department_id) LOOP
         DBMS_OUTPUT.PUT_LINE('   - ' || emp_rec.first_name);
      END LOOP;
   END LOOP;
END;
/
```
Compare this to the manual version in Topic 2/3 — every `OPEN`, `FETCH`, `EXIT WHEN`, and `CLOSE` at both levels is gone, and the `ORA-06511` risk from forgetting to close the inner cursor before the outer loop's next iteration (Topic 2, Exercise 6) is structurally impossible here, because the inner cursor is opened and closed automatically on *every* outer iteration.

## 8. Detailed Explanation

- **This is automation, not a different mechanism.** Underneath, Oracle still performs exactly the `OPEN → FETCH → check → CLOSE` sequence from Topic 1. Understanding that lifecycle is precisely what lets you reason correctly about what the FOR loop is doing for you — this topic doesn't replace Topic 1, it builds directly on it.
- **Guaranteed close, even on early exit or exceptions.** If you `EXIT` out of the loop early, `RETURN` from the enclosing subprogram, or an exception is raised inside the loop body and isn't handled locally, the cursor is still closed before control leaves. This is a genuine, practical advantage over manual management: in Topic 1, Exercise 5's corrected solution needed an explicit `EXCEPTION` block with an `%ISOPEN` check purely to guarantee cleanup on an error — a cursor FOR loop needs no such defensive code, because the guarantee is built in.
- **Named cursors still expose attributes inside the loop.** You can reference `cursor_name%ROWCOUNT`, `%FOUND`, etc. inside a named cursor's FOR loop body — useful for things like periodic progress logging (see Exercise 6). **Inline cursors cannot** — there's no name to attach an attribute to. If you need mid-loop attribute checks, use a named cursor, not an inline one.
- **You cannot manually `OPEN`, `FETCH`, or `CLOSE` a cursor while a FOR loop is driving it.** The FOR loop owns that cursor's lifecycle completely for the duration of the loop statement.
- **Inline cursors can still reference outer variables**, exactly like the "no-parameter, outer-variable" pattern from Topic 2 — the SELECT is just written directly in the loop header, so anything valid in a normal SELECT (including references to enclosing PL/SQL variables) is valid there too. What an inline cursor **cannot** do is take a formal parameter list the way a named cursor can.
- **Preview relevant to later topics:** `FOR UPDATE` and `WHERE CURRENT OF` (Topic 5) work perfectly well inside a cursor FOR loop — the loop's automation doesn't interfere with row locking or positioned updates. On the other hand, a REF CURSOR (Topic 6) is generally **not** used with this `FOR record_name IN ...` syntax at all — REF CURSORs are typically consumed with a manual `LOOP ... FETCH ... EXIT WHEN ... END LOOP;`, because they're usually opened somewhere else (passed as a parameter, or opened dynamically) rather than named directly at the point you're looping over them. More on this properly in Topic 6 — flagged here only so you don't assume this syntax generalizes to every cursor type.

## 9. Common Mistakes and Misconceptions

1. **Referencing the loop's record variable after `END LOOP`** — it's out of scope; this is a compile-time error ("identifier must be declared"), not a runtime surprise.
2. **Trying to manually `OPEN`/`FETCH`/`CLOSE` a cursor also driven by a FOR loop** — lifecycle ownership belongs entirely to the loop.
3. **Expecting an inline (implicit) cursor to expose attributes** like `%ROWCOUNT` — it has no name, so there's nothing to reference; switch to a named cursor.
4. **Assuming the FOR loop is a wholly separate feature from what Topic 1 taught** — it's automation of that exact lifecycle, not a replacement for understanding it.
5. **Forgetting that a parameterized named cursor still needs its parameters supplied at the FOR loop itself** — `FOR rec IN c_emp LOOP` will fail to compile if `c_emp` requires a parameter with no default; you'd need `FOR rec IN c_emp(30) LOOP`.

## 10. Edge Cases

- Query returns zero rows → loop body never executes, no error (identical behavior to the manual form).
- `EXIT` used inside a cursor FOR loop → the cursor still closes correctly before control leaves the loop.
- An uncaught exception inside the loop body → the cursor is closed before the exception continues propagating outward.
- **Cursor attributes cannot be read after the loop ends**, even for named cursors — the loop closes the cursor automatically when it finishes, and referencing an attribute on a closed cursor raises `ORA-01001: invalid cursor`, exactly as in Topic 1. This means you **cannot** rely on checking `cursor_name%ROWCOUNT` immediately after a FOR loop to find out "how many rows did this loop process in total" — you must track that yourself with a counter variable *inside* the loop if you need it afterward (see Exercise 7 for a related, more subtle version of this same issue).
- Nested FOR loops: each outer iteration gets a completely fresh, independently opened-and-closed inner cursor — no state leaks between outer iterations.

## 11. How This Relates to Other Topics

- **Topic 1:** the FOR loop is pure automation of the lifecycle you already know — nothing here works without that foundation.
- **Topics 2 & 3:** parameterized cursors are most commonly consumed via FOR loops, especially in nested master-detail processing — this is the "idiom" those topics were building toward.
- **Topic 5:** `FOR UPDATE`/`WHERE CURRENT OF` will be shown working inside cursor FOR loops without any special handling needed.
- **Topic 6:** REF CURSORs generally use manual `LOOP`/`FETCH` instead of this `FOR ... IN cursor_name` syntax — noted above so the two aren't conflated.

---

## Things You Must Remember

- A cursor FOR loop = automatic `OPEN` + repeated `FETCH` + `EXIT WHEN %NOTFOUND` + guaranteed `CLOSE`, including on early exit or exception.
- The loop's record variable exists **only inside the loop** — never after `END LOOP`.
- Two forms: named cursor (optionally parameterized) or an inline `SELECT` written directly in the loop header.
- You cannot mix manual `OPEN`/`FETCH`/`CLOSE` with a cursor being driven by a FOR loop.
- Only **named** cursors expose attributes (`%ROWCOUNT`, etc.) inside the loop — inline cursors don't.
- You cannot read a cursor's attributes **after** the loop ends, even for named cursors — the cursor is already closed by then.

## How to Recognize This Concept

Default to a cursor FOR loop whenever a requirement is "for every row/record, do X" with:
- No need to control fetch timing manually,
- No need to split `OPEN`/`FETCH`/`CLOSE` across different subprograms,
- No need for anything beyond straightforward one-row-per-iteration processing (including nested master-detail iteration).

Reach for manual `OPEN`/`FETCH`/`CLOSE` (Topic 1 style) only when one of those specific needs is actually present — not as a default habit.

---

# Exercises — With Answers and Reasoning

## Easy (Syntax & Basics)

### Exercise 1
**Task:** Rewrite the Topic 1, Exercise 1 solution (products in the 'Electronics' category) using a cursor FOR loop instead of manual `OPEN`/`FETCH`/`CLOSE`.

**Solution:**
```sql
DECLARE
   CURSOR c_products IS
      SELECT product_id, product_name, unit_price
      FROM products
      WHERE category = 'Electronics';
BEGIN
   FOR prod_rec IN c_products LOOP
      DBMS_OUTPUT.PUT_LINE(prod_rec.product_name || ' - Rs.' || prod_rec.unit_price);
   END LOOP;
END;
/
```

**Reasoning:** No variables need to be declared for `product_id`, `product_name`, or `unit_price` — `prod_rec` is implicitly typed as `c_products%ROWTYPE` by the loop itself. There's no `OPEN`, no `EXIT WHEN`, no `CLOSE` — all three are handled automatically, and the logic is reduced to just the part that actually matters: what to do with each row.

---

### Exercise 2
**Task:** Write an inline (implicit) cursor FOR loop, with no separate `DECLARE` section, printing all customers from a `customers` table located in `'Chennai'`.

**Solution:**
```sql
BEGIN
   FOR cust_rec IN (SELECT customer_name FROM customers WHERE city = 'Chennai') LOOP
      DBMS_OUTPUT.PUT_LINE(cust_rec.customer_name);
   END LOOP;
END;
/
```

**Reasoning:** Since this query is a one-off, not reused elsewhere in the block, and needs no parameters or attribute checks, Form 2 (inline cursor) is the simplest correct choice — no `CURSOR` declaration is needed at all, and there's nothing else in this block requiring a `DECLARE` section either.

---

### Exercise 3
**Task:** Rewrite the Topic 2/3 nested department → employee example using **only** nested inline cursor FOR loops — no named `CURSOR` declarations anywhere.

**Solution:**
```sql
BEGIN
   FOR dept_rec IN (SELECT department_id, department_name FROM departments) LOOP
      DBMS_OUTPUT.PUT_LINE('Department: ' || dept_rec.department_name);

      FOR emp_rec IN (SELECT first_name FROM employees
                       WHERE department_id = dept_rec.department_id) LOOP
         DBMS_OUTPUT.PUT_LINE('   - ' || emp_rec.first_name);
      END LOOP;
   END LOOP;
END;
/
```

**Reasoning:** The inner query references `dept_rec.department_id` directly — this is valid because `dept_rec` is in scope for the entire outer loop body, and the inner `SELECT` is just an ordinary query that happens to reference an enclosing PL/SQL value, exactly like the "outer-variable" cursor pattern from Topic 2, just written inline instead of as a separately named cursor. Since neither cursor is reused anywhere else and neither needs attribute checks, inline form is appropriate for both levels here.

---

### Exercise 4
**Task:** Take a parameterized cursor with a default value (like Topic 2, Exercise 3) and show how to consume it with a FOR loop, both using the default and overriding it.

**Solution:**
```sql
DECLARE
   CURSOR c_emp (p_dept_id employees.department_id%TYPE,
                 p_status employees.status%TYPE DEFAULT 'ACTIVE') IS
      SELECT first_name FROM employees
      WHERE department_id = p_dept_id AND status = p_status;
BEGIN
   FOR emp_rec IN c_emp(30) LOOP                  -- uses default 'ACTIVE'
      DBMS_OUTPUT.PUT_LINE(emp_rec.first_name);
   END LOOP;

   FOR emp_rec IN c_emp(30, 'INACTIVE') LOOP       -- overrides the default
      DBMS_OUTPUT.PUT_LINE(emp_rec.first_name);
   END LOOP;
END;
/
```

**Reasoning:** Parameter and default-value rules from Topic 2 are completely unchanged by using a FOR loop — the FOR loop only automates the open/fetch/close mechanics, not how parameters or defaults are resolved. Note also that `emp_rec` in the second loop is a **new, separate** variable from `emp_rec` in the first loop, even though they share the same name — each FOR loop's record variable only exists for the duration of its own loop.

---

## Intermediate (Reasoning & Edge Cases)

### Exercise 5
**Task:** Predict what happens when this compiles and runs, explain why, and fix it — assuming the genuine requirement is "print the last processed employee's name after the loop ends."
```sql
DECLARE
   CURSOR c_emp IS SELECT first_name FROM employees WHERE department_id = 10;
BEGIN
   FOR emp_rec IN c_emp LOOP
      DBMS_OUTPUT.PUT_LINE(emp_rec.first_name);
   END LOOP;
   DBMS_OUTPUT.PUT_LINE('Last employee processed: ' || emp_rec.first_name);
END;
/
```

**Answer:** This **fails to compile** with an error along the lines of `PLS-00201: identifier 'EMP_REC' must be declared`. The record variable `emp_rec` is implicitly declared by the `FOR` loop and its scope is strictly limited to the loop body — once `END LOOP` is reached, `emp_rec` no longer exists as an identifier at all. The final `DBMS_OUTPUT.PUT_LINE` line tries to reference it outside that scope, which the compiler rejects before the block ever runs.

**Fix — track the value in a separately declared variable:**
```sql
DECLARE
   CURSOR c_emp IS SELECT first_name FROM employees WHERE department_id = 10;
   v_last_name employees.first_name%TYPE;
BEGIN
   FOR emp_rec IN c_emp LOOP
      DBMS_OUTPUT.PUT_LINE(emp_rec.first_name);
      v_last_name := emp_rec.first_name;
   END LOOP;

   DBMS_OUTPUT.PUT_LINE('Last employee processed: ' || v_last_name);
END;
/
```

**Edge case worth noting:** if the cursor matches **zero rows**, the loop body never runs, `v_last_name` stays `NULL` (its uninitialized default), and the final line would print `"Last employee processed: "` with nothing after the colon — not an error, but possibly not what a reader expects. Depending on the actual requirement, you might want an explicit `IF v_last_name IS NULL THEN ... END IF;` to handle "no employees found" more clearly. This is a direct real-world consequence of the FOR loop hiding `%NOTFOUND` from you entirely — you have to think about the zero-row case yourself rather than seeing an explicit check for it in the code.

---

### Exercise 6
**Task:** A developer needs to print a progress message every 100 rows while iterating a large cursor, using an inline SELECT-based FOR loop. They try referencing `%ROWCOUNT` and get a compile error. Explain the issue and fix it.

**Answer:** An inline (implicit) cursor has **no name** — it's just a `SELECT` written directly inside the loop header — so there is no identifier to attach a `%ROWCOUNT` attribute reference to, and the compiler rejects it (`PLS-00225: subprogram or cursor reference is invalid` or a similar identifier error, depending on Oracle version). Cursor attributes are only available on **named** cursors. The fix is to declare the cursor explicitly instead of writing it inline:

```sql
DECLARE
   CURSOR c_big IS
      SELECT employee_id FROM employees;
BEGIN
   FOR rec IN c_big LOOP
      -- ... row processing here ...

      IF MOD(c_big%ROWCOUNT, 100) = 0 THEN
         DBMS_OUTPUT.PUT_LINE('Processed ' || c_big%ROWCOUNT || ' rows so far...');
      END IF;
   END LOOP;
END;
/
```

**Reasoning:** `c_big%ROWCOUNT` is fully valid **inside** the loop body for a named cursor, since the cursor is still open at that point — this doesn't contradict the earlier edge-case rule about not reading attributes *after* the loop ends; it's specifically checked *during* each iteration, while the cursor is still active.

---

## Realistic Scenario

### Exercise 7
**Business Requirement:** *"The payroll team needs a script that, for every department, lists each employee's name and calculates their annual bonus as 10% of salary, but only for employees who have been with the company more than 1 year. For departments with no qualifying employees, print a note saying so instead of printing nothing at all."*

**What Is Being Asked:**
A nested, per-department report where the inner listing is filtered by tenure, and departments with zero qualifying employees need an explicit fallback message rather than silence.

**Key Clues:**
- "for every department" + "lists each employee" → nested master-detail pattern (outer: departments, inner: employees).
- "only for employees who have been with the company more than 1 year" → a filter condition, best pushed into the inner cursor's `WHERE` clause (same "filter what you don't need at all" reasoning as Topic 1's warehouse example and Topic 2's finance example).
- "print a note saying so instead of nothing" → this is the interesting part. If the inner cursor's `WHERE` clause filters out all employees for a department, the inner FOR loop's body simply never runs for that department — by default, that would produce **silence**, not the required note. Something extra is needed to detect "the inner loop ran zero times" for a given department.

**Thought Process — Why This Needs Extra Handling:**
A plain nested FOR loop hides `%NOTFOUND` entirely — you never get to check "did this fetch return a row?" the way you could with manual `OPEN`/`FETCH`. And per the edge cases covered above, you **cannot** reliably check `cursor_name%ROWCOUNT` right after the inner loop ends either, because by then the inner cursor has already been automatically closed for that iteration, and reading an attribute on a closed cursor raises `ORA-01001`. So the only clean way to know "did any qualifying employee exist for this department?" is to **track it yourself** with a variable set inside the inner loop body — reset to `FALSE` before the inner loop starts on every outer iteration, and flipped to `TRUE` the moment the inner loop body runs even once.

**Solution:**
```sql
DECLARE
   CURSOR c_dept IS
      SELECT department_id, department_name FROM departments;

   CURSOR c_qualifying_emp (p_dept_id departments.department_id%TYPE) IS
      SELECT first_name, salary
      FROM employees
      WHERE department_id = p_dept_id
        AND hire_date < ADD_MONTHS(SYSDATE, -12);

   v_bonus     employees.salary%TYPE;
   v_found_any BOOLEAN;
BEGIN
   FOR dept_rec IN c_dept LOOP
      DBMS_OUTPUT.PUT_LINE('Department: ' || dept_rec.department_name);
      v_found_any := FALSE;

      FOR emp_rec IN c_qualifying_emp(dept_rec.department_id) LOOP
         v_found_any := TRUE;
         v_bonus := emp_rec.salary * 0.10;
         DBMS_OUTPUT.PUT_LINE('   ' || emp_rec.first_name || ' - Bonus: ' || v_bonus);
      END LOOP;

      IF NOT v_found_any THEN
         DBMS_OUTPUT.PUT_LINE('   No qualifying employees in this department.');
      END IF;
   END LOOP;
END;
/
```

**Explanation:** The tenure filter lives in the inner cursor's `WHERE` clause, since there is genuinely nothing to do with non-qualifying employees individually — they should never even reach the loop body. But because the requirement also needs to react to the *absence* of any qualifying row, a manually tracked `v_found_any` flag — reset per department, set inside the inner loop — is what bridges the gap that the FOR loop's automation otherwise hides. This is a good example of "the FOR loop removes lifecycle boilerplate, but doesn't remove the need to think about the zero-row case when your business logic actually cares about it."

**Alternative Approach (and why it's worse here):** You could run a separate `SELECT COUNT(*) INTO v_count FROM employees WHERE department_id = dept_rec.department_id AND hire_date < ADD_MONTHS(SYSDATE, -12);` before the inner loop, and branch on `v_count = 0`. This works, but it duplicates the exact same filter logic in two places (the `COUNT(*)` query and the cursor's `WHERE` clause) — any future change to the tenure rule (say, changing "1 year" to "18 months") now has to be updated in two spots instead of one, risking them drifting out of sync. The boolean-flag approach keeps the filter logic in exactly one place.

**Common Mistakes to Watch For:**
- Forgetting to reset `v_found_any := FALSE;` **inside** the outer loop, before the inner loop starts — if it's declared and set once outside the outer loop entirely, a department with qualifying employees would incorrectly mark every *later* department as "found" too, since the flag would never reset back to `FALSE`.
- Trying to check `c_qualifying_emp%ROWCOUNT` right after the inner `END LOOP` instead of using a separate flag — this raises `ORA-01001`, since the inner cursor is already closed by that point on every outer iteration.
- Using `hire_date <= ADD_MONTHS(SYSDATE, -12)` instead of the stricter `<` used in the solution above — a boundary-condition detail worth checking carefully against the exact wording. "More than 1 year" means an employee hired **exactly** one year ago today has not yet been with the company for *more than* a year — they've been there for *exactly* a year. Using `<=` would incorrectly include that employee; the strict `<` correctly excludes them. This kind of "more than" vs. "at least" wording is easy to skim past, but it changes which comparison operator is correct.

---

**End of Topic 4.** Next file: `05-for-update-and-where-current-of.md`.