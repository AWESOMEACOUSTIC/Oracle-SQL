# Topic 1: Types of Cursors and Steps to Create Cursors

> **Syllabus position:** Topic 1 of 6 (+ final combined practice set)
> **Prerequisite covered in this file (not a separate syllabus topic):** `%TYPE` and `%ROWTYPE`

---

## 0. Prerequisite: `%TYPE` and `%ROWTYPE`

You can't work with cursors properly without these two attributes, so they're covered here in full before the cursor content — not as a new syllabus topic, but as required support for everything below.

### Concept

- **`%TYPE`** — lets a variable inherit the datatype of a specific table column (or another variable) instead of you hardcoding a datatype.
- **`%ROWTYPE`** — lets a single record variable inherit the structure (all columns, in order, with matching datatypes) of a table, view, or cursor.

### Why They Exist / What Problem They Solve

If you hardcode `v_salary NUMBER(8,2)` and the DBA later changes the `salary` column to `NUMBER(10,2)`, your code is now silently out of sync and may truncate data or fail. `%TYPE` and `%ROWTYPE` remove this fragility — your variable's structure is always guaranteed to match the data source it came from, because it's derived, not duplicated.

They also reduce typing: instead of declaring one variable per column with the exact right datatype and precision, you declare one record variable (`%ROWTYPE`) or anchor each variable to its source (`%TYPE`).

### Syntax

```sql
-- %TYPE: anchor a scalar variable to a column's datatype
v_emp_id     employees.employee_id%TYPE;
v_salary     employees.salary%TYPE;

-- %TYPE can also anchor to another variable
v_bonus      v_salary%TYPE;

-- %ROWTYPE: anchor a record variable to a table's full structure
v_emp_rec    employees%ROWTYPE;

-- %ROWTYPE anchored to a cursor's structure (only the columns in the cursor's SELECT list)
CURSOR c_emp IS SELECT employee_id, first_name FROM employees;
v_emp_rec2   c_emp%ROWTYPE;
```

### Syntax Breakdown

- `table.column%TYPE` — datatype only, not the value. You still need to populate it (via `SELECT INTO`, `FETCH INTO`, or assignment).
- `table%ROWTYPE` — one field per **table column**, in the same order and datatype as the table, regardless of what any query selects.
- `cursor_name%ROWTYPE` — one field per **column in the cursor's SELECT list** only. If the cursor selects 3 of a table's 10 columns, the record has exactly 3 fields — not 10.
- Access record fields with dot notation: `v_emp_rec.salary`, `v_emp_rec.first_name`.

### Types / Variations

| Form | Anchored to | Use case |
|---|---|---|
| `column%TYPE` | A specific table/view column | Single-value variables that must always match a column's datatype |
| `variable%TYPE` | Another PL/SQL variable | Keeping two related variables (e.g., `v_salary`, `v_bonus`) always in sync |
| `table%ROWTYPE` | Full table/view structure | When you need a variable representing "one full row" of a table, independent of any specific query |
| `cursor%ROWTYPE` | A cursor's SELECT list | When fetching from a cursor that doesn't select all columns — most common with cursors |

### Simple Examples

```sql
-- %TYPE example
DECLARE
   v_salary employees.salary%TYPE;
BEGIN
   SELECT salary INTO v_salary FROM employees WHERE employee_id = 100;
   DBMS_OUTPUT.PUT_LINE(v_salary);
END;
/

-- %ROWTYPE example (table-based)
DECLARE
   v_emp employees%ROWTYPE;
BEGIN
   SELECT * INTO v_emp FROM employees WHERE employee_id = 100;
   DBMS_OUTPUT.PUT_LINE(v_emp.first_name || ' ' || v_emp.last_name);
END;
/
```

### Important Rules and Restrictions

- `%TYPE`/`%ROWTYPE` are resolved at **compile time** based on the current structure of the referenced object — if the table structure changes, you must recompile the PL/SQL unit for it to pick up the change (it's not dynamic at runtime).
- With `table%ROWTYPE`, if you do `SELECT * INTO v_rec`, the column order in `SELECT *` must match the table's actual column order — this works automatically since both come from the same table definition.
- With `cursor%ROWTYPE`, the record's field names come from the cursor's SELECT list (or column aliases, if used) — if you `SELECT salary AS sal`, the field becomes `v_rec.sal`, not `v_rec.salary`.
- You **cannot** assign a `table%ROWTYPE` record directly into a cursor's FETCH if the cursor selects a different set/order of columns than the full table — the structures must match.

### Common Mistakes

1. Using `table%ROWTYPE` when the cursor selects only some columns — this causes a field-count mismatch error at `FETCH`. Use `cursor%ROWTYPE` instead in that case.
2. Forgetting that `%ROWTYPE` fields are accessed with dot notation (`v_rec.column_name`), not as standalone variables.
3. Assuming `%TYPE`/`%ROWTYPE` re-check the table structure every time the block runs — they don't; it's fixed at compile time.

### How to Recognize When to Use Which

- Need **one value** with a guaranteed-correct datatype tied to a column → `%TYPE`.
- Need to **fetch/hold a full row** and the query is `SELECT *` or matches all table columns → `table%ROWTYPE`.
- Need to **fetch/hold a row from a cursor that selects specific columns** → `cursor%ROWTYPE` (this is the most common pairing with explicit cursors, and the one you'll use constantly in this course).

---

## 1. Concept

A **cursor** is a pointer/handle to a private memory area (the "context area" or "private SQL area") that Oracle uses to execute a SQL statement and hold its result set. Cursors are how you move from "SQL returns a set of rows all at once" to "PL/SQL can walk through those rows one at a time and apply procedural logic to each."

## 2. Purpose / Why It Exists

SQL is **set-based** — a `SELECT` returns an entire result set at once, and an `UPDATE`/`DELETE` applies the *same* logic to every matching row. Real business logic is frequently **not uniform per row**. Example: "for every pending order, if the customer is VIP apply a 10% discount, else check stock and either confirm or flag as backordered." That's a different decision *per row*, possibly calling different procedures or writing to different tables depending on conditions. Plain SQL cannot express that branching row-by-row control flow — PL/SQL can, and a cursor is the bridge that lets PL/SQL "see" a SQL result set one row at a time.

## 3. What Real-World Problem It Solves

Any requirement needing **row-level decision-making, row-level side effects, or row-level calls to other logic** that a single SQL statement can't express. Typical cases: nightly batch jobs, data migration with per-record validation, generating per-row audit/log entries, orchestrating calls to other procedures for each qualifying record.

## 4. When to Use / When NOT to Use

**Use a cursor when:**
- Conditional logic differs per row, not a uniform `WHERE`/`SET`.
- You need to call other PL/SQL procedures/functions per row.
- You need row-level exception handling that one DML statement can't provide.

**Do NOT use a cursor when:**
- A single SQL statement (`UPDATE ... SET ... WHERE ...`, `INSERT ... SELECT`, `MERGE`) achieves the same result. The most common beginner mistake is writing a cursor loop for something like "give everyone in dept 10 a 5% raise" when one `UPDATE` does it faster and simpler.
- Performance matters on large volumes — row-by-row processing (informally "RBAR": row-by-row = slow-by-slow) is far slower than set-based SQL, since each `FETCH` is a context switch between the SQL engine and PL/SQL engine. (`BULK COLLECT`/`FORALL` exist to address this — outside this syllabus, but worth knowing they exist.)

**Recognition clue:** If a requirement can be phrased as "update/insert/delete rows matching a condition" with no per-row branching — that's plain SQL. If it says "for each such record, check X and then do Y or Z" — that's a cursor/loop signal.

## 5. Types of Cursors

| Type | What it is | Declared by you? |
|---|---|---|
| **Implicit Cursor** | Auto-created by Oracle for every DML statement and single-row `SELECT INTO`. | No — Oracle manages it silently. |
| **Explicit Cursor** | A named, user-defined cursor for a `SELECT` that may return zero, one, or many rows. Full lifecycle control. | Yes |
| **Cursor Variable (REF CURSOR)** | A pointer to a result set not tied to one fixed query at compile time — can be passed as a parameter, opened for different queries at runtime. | Yes (Topic 6) |

### 5.1 Implicit Cursors

Every DML statement or single-row `SELECT INTO` silently uses a cursor referenced through the pseudo-name `SQL`. You never declare/manage it — only inspect it afterward via attributes:

- `SQL%FOUND` — TRUE if the last DML affected at least one row (or SELECT INTO found a row)
- `SQL%NOTFOUND` — TRUE if it affected/found none
- `SQL%ROWCOUNT` — number of rows affected
- `SQL%ISOPEN` — always FALSE when checked (Oracle closes implicit cursors immediately after execution)

```sql
BEGIN
   UPDATE employees SET salary = salary * 1.1 WHERE department_id = 30;
   IF SQL%FOUND THEN
      DBMS_OUTPUT.PUT_LINE(SQL%ROWCOUNT || ' rows updated.');
   ELSE
      DBMS_OUTPUT.PUT_LINE('No matching employees found.');
   END IF;
END;
/
```

**Important trap:** `SELECT ... INTO` requires **exactly one row**. Zero rows raises `NO_DATA_FOUND`; more than one raises `TOO_MANY_ROWS`. It does **not** behave like DML's found/not-found — you must handle these exceptions explicitly. This is one of the most common real-world bugs: developers assume `SELECT INTO` degrades gracefully like a normal query.

### 5.2 Explicit Cursors

Declared by you, for queries that can return any number of rows (zero, one, or many). You manage the full lifecycle manually.

## 6. Steps to Create and Use an Explicit Cursor (the Lifecycle)

1. **DECLARE** — define the cursor name and its query (query doesn't run yet).
2. **OPEN** — executes the query, fixes the "active set," positions the pointer just before the first row.
3. **FETCH** — retrieves one row at a time into variables/record, advancing the pointer.
4. **Loop + exit condition** — repeat FETCH until no more rows.
5. **CLOSE** — releases resources. Mandatory — skipping this is a resource leak.

## 7. Syntax

```sql
DECLARE
   CURSOR emp_cursor IS
      SELECT employee_id, first_name, salary
      FROM employees
      WHERE department_id = 30;

   v_emp_id     employees.employee_id%TYPE;
   v_first_name employees.first_name%TYPE;
   v_salary     employees.salary%TYPE;
BEGIN
   OPEN emp_cursor;
   LOOP
      FETCH emp_cursor INTO v_emp_id, v_first_name, v_salary;
      EXIT WHEN emp_cursor%NOTFOUND;

      DBMS_OUTPUT.PUT_LINE(v_first_name || ' earns ' || v_salary);
   END LOOP;
   CLOSE emp_cursor;
END;
/
```

### Syntax Breakdown

- `CURSOR emp_cursor IS <select statement>` — declares the cursor; the query is **not executed** here, only defined.
- `OPEN emp_cursor` — runs the query now; the active set is fixed at this moment based on data visible to your session (read-consistency).
- `FETCH emp_cursor INTO ...` — pulls the next row's values into your variables/record, in the same order as the SELECT list.
- `EXIT WHEN emp_cursor%NOTFOUND` — must come **immediately after** FETCH and **before** processing the fetched values. Placing processing logic before this check means on the final failed fetch you'd process stale/duplicate data from the previous successful fetch.
- `CLOSE emp_cursor` — releases resources. Forgetting this across repeated opens can eventually raise `ORA-01000: maximum open cursors exceeded`.

### Using `%ROWTYPE` with a cursor (preferred style for multi-column fetches)

```sql
DECLARE
   CURSOR emp_cursor IS
      SELECT employee_id, first_name, salary FROM employees WHERE department_id = 30;
   v_emp_rec emp_cursor%ROWTYPE;
BEGIN
   OPEN emp_cursor;
   LOOP
      FETCH emp_cursor INTO v_emp_rec;
      EXIT WHEN emp_cursor%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_emp_rec.first_name || ' earns ' || v_emp_rec.salary);
   END LOOP;
   CLOSE emp_cursor;
END;
/
```

## 8. Cursor Attributes (Explicit)

| Attribute | Meaning |
|---|---|
| `%FOUND` | TRUE if the last FETCH returned a row |
| `%NOTFOUND` | TRUE if the last FETCH did NOT return a row (used to exit loops) |
| `%ISOPEN` | TRUE if the cursor is currently open |
| `%ROWCOUNT` | Number of rows fetched so far in this cursor's lifecycle |

## 9. Detailed Explanation — What's Actually Happening

- At `OPEN`, Oracle parses/executes the query and determines the active set. Rows inserted by *other* sessions after your `OPEN` generally won't appear in your fetches — this read-consistency matters in long-running batch cursors.
- Each `FETCH` is a round trip between the SQL engine and PL/SQL engine — the performance argument against overusing cursors on large data.
- A cursor attribute can only be evaluated on an **opened** cursor; referencing one before `OPEN` raises an error.
- If a cursor's query matches **zero rows**, `OPEN` still succeeds — the first `FETCH` immediately sets `%NOTFOUND` to TRUE and the loop body never executes. No error is raised. This trips people up because they expect a failure, but there isn't one.
- `%ROWCOUNT` reflects only **successful** fetches — the fetch attempt that triggers `%NOTFOUND` does not increment it.

## 10. Common Mistakes and Misconceptions

1. **Forgetting `CLOSE`** — resource leak; repeated over many executions leads to `ORA-01000`.
2. **Checking `%NOTFOUND` after processing instead of before** — always check immediately after `FETCH`, before using fetched values.
3. **Fetching without opening** — raises `ORA-01001: invalid cursor`.
4. **Opening an already-open cursor** — raises `ORA-06511: cursor already open`. Common when a cursor is opened inside a loop that runs more than once without a matching close first.
5. **Using a cursor loop where one DML statement would do the job** — a design smell flagged in real code review, not a syntax error.
6. **Confusing `SQL%ROWCOUNT` (implicit) with `cursor_name%ROWCOUNT` (explicit)** — tracked completely independently.
7. **Missing `EXIT WHEN`** — causes an infinite loop (see Exercise 5 below for full reasoning).

## 11. Edge Cases

- Query returns 0 rows → loop body never runs, no error.
- Query returns exactly 1 row → behaves like any normal fetch loop (don't confuse with `SELECT INTO`, a separate mechanism for exactly-one-row cases).
- Cursor declared but never opened → referencing its attributes raises an error.
- Very large result sets → correctness is fine, but performance suffers; flag as a candidate for `BULK COLLECT` in production (outside current syllabus).

## 12. How This Relates to Upcoming Topics

- **Cursor FOR loops (Topic 4)** automate steps 2–5 (open, fetch, check, close) — same lifecycle, less boilerplate.
- **Cursor parameters (Topic 2)** let you reuse one cursor definition for different filter values instead of hardcoding a `WHERE` value.
- **FOR UPDATE (Topic 5)** modifies step 2 (`OPEN`) to also lock the selected rows.
- **REF CURSOR (Topic 6)** generalizes this idea so the query isn't fixed at declare-time.

---

## Things You Must Remember

- Implicit cursor → `SQL%ATTRIBUTE`; Explicit cursor → `cursor_name%ATTRIBUTE`.
- `SELECT INTO` with zero or multiple rows raises exceptions (`NO_DATA_FOUND` / `TOO_MANY_ROWS`) — it does **not** set `%NOTFOUND`.
- Lifecycle: OPEN → FETCH → check `%NOTFOUND` → process → loop → CLOSE. Never skip CLOSE.
- The active set is fixed at `OPEN` time, not at `DECLARE` time.
- A cursor returning zero rows is not an error — the loop body simply never runs.
- `%ROWTYPE` on a cursor mirrors only the cursor's SELECT list; `%ROWTYPE` on a table mirrors the full table.
- Don't reach for a cursor when a single SQL statement solves the problem.

## How to Recognize This Concept

Think **"explicit cursor"** when a requirement says things like:
- "For every record matching X, check condition Y and then do Z" (per-row branching)
- "Process each pending/eligible record and call [other logic] for it"
- "Generate a report/log line for each qualifying row"
- Anything implying you must **iterate** and make a **decision per row**.

Think **"this doesn't need a cursor"** when the requirement is just "update/delete/insert rows matching a condition" with no per-row branching — that's plain SQL.

---

# Exercises — With Answers and Reasoning

## Easy (Syntax & Basics)

### Exercise 1
**Task:** Write a cursor that selects `product_id, product_name, unit_price` from a `products` table where `category = 'Electronics'`, and print each product's name and price.

**Solution:**
```sql
DECLARE
   CURSOR c_products IS
      SELECT product_id, product_name, unit_price
      FROM products
      WHERE category = 'Electronics';

   v_product_id   products.product_id%TYPE;
   v_product_name products.product_name%TYPE;
   v_unit_price   products.unit_price%TYPE;
BEGIN
   OPEN c_products;
   LOOP
      FETCH c_products INTO v_product_id, v_product_name, v_unit_price;
      EXIT WHEN c_products%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_product_name || ' - Rs.' || v_unit_price);
   END LOOP;
   CLOSE c_products;
END;
/
```

**Reasoning:** This is the plain lifecycle: DECLARE the cursor with the exact filter needed (`WHERE category = 'Electronics'` is pushed into the query, not checked later in an `IF`), OPEN it, FETCH into variables anchored with `%TYPE` (so the code stays correct even if `unit_price`'s precision changes later), check `%NOTFOUND` immediately after FETCH, and CLOSE at the end. Nothing here needs branching per row — it's a template exercise to get the lifecycle into muscle memory.

---

### Exercise 2
**Task:** Modify a DML statement (`UPDATE` on `customers`) to report, using implicit cursor attributes, how many rows were updated — and print a message if zero rows matched.

**Solution:**
```sql
BEGIN
   UPDATE customers
   SET status = 'INACTIVE'
   WHERE last_purchase_date < ADD_MONTHS(SYSDATE, -12);

   IF SQL%FOUND THEN
      DBMS_OUTPUT.PUT_LINE(SQL%ROWCOUNT || ' customers marked inactive.');
   ELSE
      DBMS_OUTPUT.PUT_LINE('No customers matched the inactivity condition.');
   END IF;
END;
/
```

**Reasoning:** No `CURSOR` declaration, `OPEN`, or `CLOSE` appears here — that's the whole point of an **implicit** cursor: Oracle manages it automatically for the `UPDATE`, and you only *inspect* it afterward via the `SQL` pseudo-name. `SQL%FOUND`/`SQL%ROWCOUNT` must be checked **immediately after** the DML statement — if another DML statement ran in between, `SQL%ROWCOUNT` would reflect that later statement instead, since there's only ever one "current" implicit cursor state per session.

---

### Exercise 3
**Task:** Count how many rows a cursor fetched from `orders` where `status = 'PENDING'`, using `%ROWCOUNT`, without a separate counter variable.

**Solution:**
```sql
DECLARE
   CURSOR c_pending IS
      SELECT order_id FROM orders WHERE status = 'PENDING';
   v_order_id orders.order_id%TYPE;
BEGIN
   OPEN c_pending;
   LOOP
      FETCH c_pending INTO v_order_id;
      EXIT WHEN c_pending%NOTFOUND;
   END LOOP;
   DBMS_OUTPUT.PUT_LINE('Total pending orders: ' || c_pending%ROWCOUNT);
   CLOSE c_pending;
END;
/
```

**Reasoning:** The key subtlety this exercise is testing: `%ROWCOUNT` must be read **before** `CLOSE`. Once a cursor is closed, all its attributes become invalid and referencing them raises `ORA-01001: invalid cursor`. Also note that `%ROWCOUNT` at the point the loop exits already holds the correct final count — the failed fetch that triggered `%NOTFOUND` does **not** increment it, so there's no off-by-one error to worry about.

---

### Exercise 4
**Task:** Declare a cursor using `%ROWTYPE` instead of individual variables, for a `students` table, and print each student's name and grade.

**Solution:**
```sql
DECLARE
   CURSOR c_students IS
      SELECT student_id, student_name, grade
      FROM students;
   v_student_rec c_students%ROWTYPE;
BEGIN
   OPEN c_students;
   LOOP
      FETCH c_students INTO v_student_rec;
      EXIT WHEN c_students%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE(v_student_rec.student_name || ' - Grade: ' || v_student_rec.grade);
   END LOOP;
   CLOSE c_students;
END;
/
```

**Reasoning:** `v_student_rec` is anchored to `c_students%ROWTYPE`, not `students%ROWTYPE`. This matters because the cursor selects only 3 of the table's columns — if you'd anchored to `students%ROWTYPE` instead, the record would have a field for *every* table column, and `FETCH ... INTO v_student_rec` would fail with a column-count mismatch, since the cursor only returns 3 values but the record expects as many fields as the full table has columns. Always match `%ROWTYPE` to the actual source of the values you're fetching.

---

## Intermediate (Reasoning & Edge Cases)

### Exercise 5
**Task:** A cursor is declared for `SELECT * FROM invoices WHERE amount > 100000`. Predict: (a) what happens if zero invoices match, and (b) what happens if `EXIT WHEN` is omitted entirely. Then write the block correctly.

**Answer — Prediction (a), zero rows match:**
`OPEN` succeeds without error — opening a cursor only executes the query and fixes the active set; it never requires the active set to be non-empty. The very first `FETCH` will set `c_invoices%NOTFOUND` to `TRUE` immediately, the fetched variables remain in whatever state they were before (uninitialized/NULL if this is the first fetch), and the loop body never executes even once. No exception is raised. This is different from `SELECT INTO`, which *would* raise `NO_DATA_FOUND` in the same situation — a common point of confusion.

**Answer — Prediction (b), missing `EXIT WHEN`:**
This causes an **infinite loop**. Once the active set is exhausted, each further `FETCH` simply fails to bring back a new row — it does not raise an exception, it just leaves the target variables holding their previous values and sets `%NOTFOUND` to `TRUE`. Since there is no check terminating the loop, execution keeps looping forever, repeatedly "fetching nothing" and re-running the loop body with stale, duplicate data from the last successful fetch. In practice this either hangs the session, gets killed by a DBA/monitoring tool, or the loop body's side effects (like repeated `DBMS_OUTPUT.PUT_LINE` calls or writes) pile up until some other resource limit is hit. This is a real, easy-to-introduce production bug — always double-check `EXIT WHEN` is present and placed correctly.

**Corrected solution:**
```sql
DECLARE
   CURSOR c_invoices IS
      SELECT invoice_id, amount FROM invoices WHERE amount > 100000;
   v_invoice_id invoices.invoice_id%TYPE;
   v_amount     invoices.amount%TYPE;
BEGIN
   OPEN c_invoices;
   LOOP
      FETCH c_invoices INTO v_invoice_id, v_amount;
      EXIT WHEN c_invoices%NOTFOUND;
      DBMS_OUTPUT.PUT_LINE('Invoice ' || v_invoice_id || ': ' || v_amount);
   END LOOP;
   CLOSE c_invoices;
EXCEPTION
   WHEN OTHERS THEN
      IF c_invoices%ISOPEN THEN
         CLOSE c_invoices;
      END IF;
      RAISE;
END;
/
```

> **Note beyond current syllabus scope, flagged separately as instructed:** the `EXCEPTION` block here is a defensive pattern — if something inside the loop raises an unexpected error, the cursor would otherwise stay open. Checking `%ISOPEN` before closing (so you don't try to close an already-closed cursor) is a standard production habit. Full exception handling isn't part of your syllabus, so treat this as a preview, not something to master right now.

---

### Exercise 6
**Task:** A colleague's nightly job intermittently fails with `ORA-01000: maximum open cursors exceeded`. Based on the cursor lifecycle, what's your hypothesis before seeing the code?

**Answer:** The most likely cause is a **cursor leak** — a cursor (or several) being **opened repeatedly without a matching `CLOSE`**. Typical patterns that cause this:

- A cursor declared and opened *inside* a loop (e.g., once per row of an outer cursor, or once per call inside a procedure invoked many times) without ever being closed before the next iteration opens it again — each iteration adds one more open cursor to the session until Oracle's `OPEN_CURSORS` session limit is hit.
- A subprogram that opens a cursor and returns/raises early under some condition, skipping over its own `CLOSE` statement — the cursor stays open every time that particular code path is taken.

The fact that the failure is **intermittent** and happens on a **nightly batch** (i.e., varying, often larger data volumes) supports this: with a cursor leak, the number of open cursors is proportional to how many times the leaking code path executes. On nights with more qualifying rows/records, the leak accumulates faster and crosses the `OPEN_CURSORS` limit; on lighter nights it might stay just under the limit and succeed. Before looking at the code, I'd specifically check: is there an `OPEN` anywhere that isn't matched 1:1 with a `CLOSE` on every possible exit path (including exception paths)?

---

## Realistic Scenario

### Exercise 7
**Business Requirement:** *"The inventory team wants a script that goes through the `warehouse_stock` table and, for every item where quantity is below the reorder threshold, prints an alert message with the item name and how far below threshold it is. Items at or above threshold should be skipped silently."*

**What Is Being Asked:**
Iterate over warehouse stock records and, for the subset that are under-threshold, emit an alert containing the item name and the numeric shortfall. Items meeting or exceeding threshold produce no output at all.

**Key Clues:**
- "for every item where..." → filtering condition exists.
- "skipped silently" → no cursor/loop action needed at all for non-matching rows, not even a "no alert" message.
- "how far below threshold" → requires a calculation (`threshold - quantity`), not just a flag.

**What Data Is Required:** `item_name`, `quantity`, `reorder_threshold` from `warehouse_stock`.

**Key Design Decision — where should the filter live?**
There are two ways to implement "items below threshold get an alert, others are skipped":

1. Put `WHERE quantity < reorder_threshold` directly in the cursor's query, so the active set *only* contains items that need an alert.
2. Select all rows unfiltered, and put an `IF quantity < reorder_threshold THEN ... END IF;` inside the loop.

**Decision:** Option 1 (filter in the cursor's `WHERE` clause) is the right approach here, and thinking through *why* is exactly the reasoning skill this exercise is testing:
- The requirement has **no need to do anything at all** with rows that meet/exceed threshold — there's no "else" action, not even a log line. When there's truly nothing to do for the non-matching rows, filtering them out at the SQL level means they never even become part of the active set, so PL/SQL never spends a `FETCH` round-trip on them.
- The database engine is optimized for set-based filtering (and can use an index on `quantity`/`reorder_threshold` if one exists), which is far more efficient than pulling every row into PL/SQL and discarding most of them there — especially relevant since a warehouse table could be large and most items are typically *above* threshold, not below it.
- It also simplifies the code: no `IF` branch is needed at all inside the loop, because by the time a row is fetched, it's already guaranteed to qualify.

(Option 2 would only become the right choice if the requirement also needed to *do something* with the rows that don't qualify — e.g., "log a separate 'OK' message for items above threshold too." That's not the case here.)

**Step-by-Step Approach:**
1. Declare a cursor selecting `item_name, quantity, reorder_threshold` with `WHERE quantity < reorder_threshold` already applied.
2. Anchor variables with `%TYPE` to the source columns.
3. Open → loop → fetch → check `%NOTFOUND` → compute shortfall → print alert → close.

**Solution:**
```sql
DECLARE
   CURSOR c_low_stock IS
      SELECT item_name, quantity, reorder_threshold
      FROM warehouse_stock
      WHERE quantity < reorder_threshold;

   v_item_name warehouse_stock.item_name%TYPE;
   v_quantity  warehouse_stock.quantity%TYPE;
   v_threshold warehouse_stock.reorder_threshold%TYPE;
   v_shortfall warehouse_stock.reorder_threshold%TYPE;
BEGIN
   OPEN c_low_stock;
   LOOP
      FETCH c_low_stock INTO v_item_name, v_quantity, v_threshold;
      EXIT WHEN c_low_stock%NOTFOUND;

      v_shortfall := v_threshold - v_quantity;
      DBMS_OUTPUT.PUT_LINE('ALERT: ' || v_item_name ||
         ' is ' || v_shortfall || ' units below reorder threshold.');
   END LOOP;
   CLOSE c_low_stock;
END;
/
```

**Explanation:** By the time any row reaches the loop body, it is already guaranteed to be under-threshold, so the loop body has exactly one job — compute and print the shortfall. There is no `IF` needed and no risk of accidentally printing an alert for a compliant item.

**Alternative Approach (and why it's worse here):**
```sql
-- Alternative: fetch everything, branch inside the loop
DECLARE
   CURSOR c_all_stock IS
      SELECT item_name, quantity, reorder_threshold FROM warehouse_stock;
   v_item_name warehouse_stock.item_name%TYPE;
   v_quantity  warehouse_stock.quantity%TYPE;
   v_threshold warehouse_stock.reorder_threshold%TYPE;
BEGIN
   OPEN c_all_stock;
   LOOP
      FETCH c_all_stock INTO v_item_name, v_quantity, v_threshold;
      EXIT WHEN c_all_stock%NOTFOUND;
      IF v_quantity < v_threshold THEN
         DBMS_OUTPUT.PUT_LINE('ALERT: ' || v_item_name ||
            ' is ' || (v_threshold - v_quantity) || ' units below reorder threshold.');
      END IF;
   END LOOP;
   CLOSE c_all_stock;
END;
/
```
This produces **identical output**, so it isn't "wrong" — but it's a worse design here: it fetches every single row in the table (potentially most of a large warehouse's inventory) purely to discard the majority of them with an `IF`, doing extra round-trips and extra work for no benefit. It would only become the *better* choice if you also needed an action for the rows that don't qualify (turning the `IF` into an `IF...ELSE` that does two different things).

**Common Mistakes to Watch For:**
- Computing shortfall as `v_quantity - v_threshold` instead of `v_threshold - v_quantity` — sign gets flipped, producing negative numbers instead of a meaningful shortfall.
- Forgetting `%TYPE` anchoring, risking datatype mismatches if the table's column definitions change later.
- Forgetting `CLOSE` at the end.
- Filtering with `<=` instead of `<` — the requirement explicitly says items "at or above threshold" are skipped, meaning `quantity = reorder_threshold` should **not** trigger an alert, so the boundary condition (`<` vs `<=`) must be read carefully from the wording.

---

**End of Topic 1.** Next file: `02-cursors-with-and-without-parameters.md`.