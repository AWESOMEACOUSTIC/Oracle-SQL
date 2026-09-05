# Module 1, Topic 4: PL/SQL Data Types (Scalar, Vector, Reference, LOB)

---

## 1. What This Topic Covers

Every variable you declare needs a **datatype** — it tells Oracle how much memory to allocate, what kind of values are valid, and what operations make sense. You've already been using datatypes informally (`NUMBER`, `VARCHAR2`, `DATE`) throughout Modules 3 and 4. This topic formalizes the **full landscape** of PL/SQL datatypes, organized into four families: **Scalar**, **Vector** (composite), **Reference**, and **LOB** (Large Object).

---

## 2. Scalar Types

### What They Are
A **scalar** type holds a **single value** with no internal components — a number, a string, a date, a boolean. These are the datatypes you've been using constantly already.

### The Main Scalar Families

| Family | Common Types | Notes |
|---|---|---|
| **Numeric** | `NUMBER`, `NUMBER(p,s)`, `PLS_INTEGER`, `BINARY_INTEGER`, `BINARY_FLOAT`, `BINARY_DOUBLE` | `NUMBER` is the general-purpose, most commonly used numeric type; `PLS_INTEGER` is PL/SQL-only, faster for integer arithmetic since it uses native machine arithmetic instead of Oracle's general NUMBER representation. |
| **Character** | `VARCHAR2(n)`, `CHAR(n)`, `LONG` | `VARCHAR2` is variable-length (only uses as much space as the actual string needs, up to the declared max); `CHAR` is fixed-length (always pads with spaces to the declared size). `LONG` is a legacy type, rarely used in modern code. |
| **Date/Time** | `DATE`, `TIMESTAMP`, `TIMESTAMP WITH TIME ZONE` | `DATE` stores date + time down to the second; `TIMESTAMP` adds fractional seconds; the time-zone variant also stores zone offset information. |
| **Boolean** | `BOOLEAN` | `TRUE`, `FALSE`, or `NULL`. **PL/SQL-only** — as established in Module 3, SQL itself has no boolean type, so a `BOOLEAN` variable can never be used directly in a SQL statement (a `SELECT`, `WHERE`, or table column). |

### VARCHAR2 vs. CHAR — Why the Distinction Matters
```sql
DECLARE
    v_char    CHAR(10)     := 'Hi';       -- stored as 'Hi        ' (padded to 10 chars)
    v_varchar VARCHAR2(10) := 'Hi';       -- stored as 'Hi' (only 2 characters used)
BEGIN
    IF v_char = 'Hi' THEN                  -- comparison behavior can surprise beginners due to padding
        DBMS_OUTPUT.PUT_LINE('CHAR matched (Oracle blank-pads for comparison)');
    END IF;
END;
/
```
In real business systems, `VARCHAR2` is used **overwhelmingly more often** than `CHAR` — reserving `CHAR` for genuinely fixed-length codes (e.g., a 2-character country code) where padding behavior is actually desired or harmless.

### NUMBER Precision and Scale
`NUMBER(p, s)` — `p` is total significant digits (precision), `s` is digits after the decimal point (scale).
```sql
v_price NUMBER(8,2);   -- up to 8 total digits, 2 after the decimal — e.g., 123456.78
v_count NUMBER(5);     -- integer only, up to 5 digits, no decimal portion
v_amount NUMBER;       -- no precision/scale specified — maximum range/precision, most flexible but least "self-documenting"
```

### PLS_INTEGER vs. NUMBER
`PLS_INTEGER` is a PL/SQL-specific integer type that maps directly to native hardware integer arithmetic — meaning loops and counters using `PLS_INTEGER` run **noticeably faster** than the equivalent using `NUMBER`, especially in high-iteration scenarios (relevant once we reach Module 2's iteration statements). The trade-off: `PLS_INTEGER` has a fixed range (roughly -2,147,483,647 to 2,147,483,647) and will raise an overflow error if exceeded, whereas `NUMBER` has enormous range/precision but is comparatively slower for pure integer arithmetic.

---

## 3. Vector (Composite) Types

### What They Are
A **composite type** (sometimes called a "vector" type in this context, meaning it holds multiple values as one logical unit) groups **multiple individual values together** under a single variable. Unlike a scalar, a composite variable has **internal structure** you can access piece by piece.

### The Main Composite Types

| Type | What It Is | Example Use |
|---|---|---|
| **RECORD** | A structure grouping multiple **named fields**, each with its own datatype — like a single "row" shape you define yourself, or that mirrors a table's row. | Holding an employee's `id`, `name`, and `salary` together as one variable. |
| **%ROWTYPE** | A special shortcut that automatically creates a record matching a table's (or a `SELECT`'s) column structure, **without you manually declaring each field**. | `v_emp employees%ROWTYPE;` |
| **Collections** (`VARRAY`, Nested Table, Associative Array) | Structures holding **multiple values of the same type**, indexed/accessed by position or key — conceptually similar to arrays/lists in general-purpose languages. | Holding a list of employee IDs to process. *(Full collection mechanics are a deep topic of their own, generally covered in more advanced PL/SQL coursework — not part of our current syllabus in depth; mentioned here only so you recognize the category exists within the "composite" family.)* |

### Example: RECORD
```sql
DECLARE
    TYPE emp_record_type IS RECORD (
        emp_id   NUMBER,
        emp_name VARCHAR2(50),
        salary   NUMBER
    );
    v_emp emp_record_type;
BEGIN
    v_emp.emp_id   := 100;
    v_emp.emp_name := 'Arun Kumar';
    v_emp.salary   := 75000;

    DBMS_OUTPUT.PUT_LINE(v_emp.emp_name || ' earns ' || v_emp.salary);
END;
/
```

### Example: %ROWTYPE (the far more commonly used approach in real code)
```sql
DECLARE
    v_emp employees%ROWTYPE;  -- automatically matches employees table's column structure
BEGIN
    SELECT * INTO v_emp FROM employees WHERE employee_id = 100;

    DBMS_OUTPUT.PUT_LINE(v_emp.first_name || ' earns ' || v_emp.salary);
END;
/
```

**Why `%ROWTYPE` matters practically**: if the `employees` table later gets a new column added, `%ROWTYPE` **automatically adapts** — you don't need to edit your PL/SQL declaration at all. A manually-declared `RECORD` would need to be updated by hand to match. This is a genuinely important real-world maintenance benefit, and one of the clearest reasons `%ROWTYPE` is preferred over manual `RECORD` declarations whenever your structure is meant to mirror an actual table.

---

## 4. Reference Types

### What They Are
A **reference type** stores a **pointer/reference to a value or structure**, rather than the value itself directly being copied and held. In PL/SQL, the practical reference-type mechanism you're most likely to encounter is `REF CURSOR` — a reference to a **result set**, letting you pass a query's results around dynamically (e.g., between a PL/SQL procedure and an external application) without knowing the exact result structure at compile time.

### Where This Fits (and What's Out of Scope)
Full cursor mechanics — including `REF CURSOR`, explicit cursors, cursor `FOR` loops — are **not part of our current syllabus's named topics**, so we won't build detailed exercises around them here. This topic simply makes sure you know the **category exists** conceptually (Scalar / Vector / Reference / LOB is the standard four-way classification of PL/SQL datatypes, and skipping "Reference" entirely would leave a gap in your map of the landscape) — if a genuine requirement later needs dynamic result-set passing, that would be a clearly-flagged prerequisite concept beyond this syllabus, exactly per your original instructions about handling non-syllabus prerequisites.

### A Minimal Illustrative Mention
```sql
-- Illustrative only — REF CURSOR / cursor mechanics are outside our current syllabus scope
DECLARE
    TYPE emp_cursor_type IS REF CURSOR;
    v_emp_cursor emp_cursor_type;
BEGIN
    NULL; -- full usage requires OPEN/FETCH/CLOSE mechanics, not covered here
END;
/
```

---

## 5. LOB (Large Object) Types

### What They Are
**LOB types** store **large amounts of unstructured data** — far more than a normal `VARCHAR2` (which caps out at 32767 bytes in PL/SQL, 4000 bytes typically in a table column unless extended) can hold. Common in real systems for storing documents, images, or large text bodies.

| Type | Stores | Example Use |
|---|---|---|
| `CLOB` | Character Large Object — large text data | Storing a long contract document's full text, a large JSON payload |
| `BLOB` | Binary Large Object — large binary data | Storing an image, a PDF, a scanned document |
| `NCLOB` | Like CLOB, but for national character set (Unicode) data | Storing large multilingual text |
| `BFILE` | A reference to a large binary file **stored outside the database**, on the OS file system | Referencing an externally-stored file without importing its bytes into the database itself |

### Why LOBs Exist as a Separate Category
Ordinary scalar character/binary types have **hard size limits** that make them unsuitable for genuinely large content. LOBs use a different internal storage and access mechanism (often involving "locators" — reference-like handles to the actual large data, which is why some categorizations of Oracle datatypes discuss LOBs alongside reference-like behavior) specifically designed to handle megabytes-to-gigabytes of content efficiently, without forcing the entire value to be loaded into memory as a single, ordinary variable the way a `VARCHAR2` would be.

### A Minimal Illustrative Example
```sql
DECLARE
    v_document CLOB;
BEGIN
    SELECT contract_text INTO v_document FROM contracts WHERE contract_id = 500;
    DBMS_OUTPUT.PUT_LINE('Document length: ' || DBMS_LOB.GETLENGTH(v_document));
END;
/
```
*(Detailed `DBMS_LOB` package operations for reading/writing LOB content in depth are, like cursors, a more specialized area beyond our current syllabus's named topics — this example is illustrative of the category's existence and basic declaration, not a deep dive.)*

---

## 6. Detailed Explanation — Why This Four-Way Classification Matters

Understanding **which family** a datatype belongs to helps you reason about:
- **What operations make sense**: you can do arithmetic on scalars, but not meaningfully on a whole `RECORD`; you access a LOB's content through specialized functions (`DBMS_LOB`), not simple string operators.
- **What can (and can't) go into a SQL statement directly**: scalar types map cleanly to table columns; a PL/SQL-only `RECORD` type generally cannot be a single column value in a table (though its **individual fields**, once expanded, correspond to actual columns); `BOOLEAN` (a scalar, but PL/SQL-only) famously **cannot** appear in SQL at all, as covered in Module 3.
- **Memory and performance implications**: scalars are lightweight; LOBs involve specialized, more expensive access patterns; collections (vector/composite) can hold large numbers of elements and have their own performance characteristics.

---

## 7. Common Mistakes & Misconceptions

1. **Mistake**: Declaring a `RECORD` manually to mirror a table, then the table changes, and the manual `RECORD` silently falls out of sync → prefer `%ROWTYPE` whenever the structure should track an actual table.
2. **Misconception**: "CHAR and VARCHAR2 are basically interchangeable." → They differ in padding behavior and storage; using `CHAR` where `VARCHAR2` was intended can cause subtle string-comparison and storage-size surprises.
3. **Mistake**: Using `NUMBER` for a simple loop counter where `PLS_INTEGER` would be more appropriate and meaningfully faster in high-iteration code (relevant once you reach Module 2's loops).
4. **Misconception**: "A BOOLEAN variable can be stored in a table column or returned from a SQL-callable function directly." → Not possible; SQL has no boolean type at all — this connects directly back to the Module 3 discussion of function return types.
5. **Mistake**: Trying to treat a LOB exactly like a normal `VARCHAR2` (e.g., simple `||` concatenation without considering size) without being aware that LOBs often require specialized handling (`DBMS_LOB` functions) for anything beyond trivial operations.

---

## 8. Edge Cases to Be Aware Of

- `VARCHAR2` inside PL/SQL can hold up to **32,767 bytes** — notably larger than the traditional 4,000-byte limit for a `VARCHAR2` **table column** (extended data types in modern Oracle can raise the column limit too, but this distinction — PL/SQL variable limit vs. traditional column limit — is a classic point of confusion).
- `%ROWTYPE` based on `SELECT *` requires the variable's fields to align with **all** selected columns, in order — if you later change the `SELECT` to fetch a subset of columns without adjusting how you're populating the record, you can get a column-count mismatch error.
- `PLS_INTEGER` arithmetic that **overflows** its range raises an error (`ORA-01426` or similar) rather than silently wrapping around or extending precision the way `NUMBER` would handle a very large value — worth remembering if working with fields that could plausibly grow very large.

---

## 9. Interview-Level / Practical Notes

- *"What's the difference between CHAR and VARCHAR2, and which should you generally prefer?"* — `CHAR` is fixed-length (space-padded), `VARCHAR2` is variable-length; `VARCHAR2` is preferred in the vast majority of real-world cases.
- *"Why would you use %ROWTYPE instead of manually declaring a RECORD?"* — Automatic adaptation to underlying table structure changes, reducing maintenance burden — a very standard, expected answer.
- *"Why can't a function called from SQL return a BOOLEAN?"* — Because SQL itself has no boolean type; this directly connects two modules' worth of knowledge (Module 1 datatypes + Module 3 functions) and is a great way to demonstrate integrated understanding in an interview.
- *"Name the four broad categories of PL/SQL datatypes."* — Scalar, Composite (Vector), Reference, LOB — a clean, expected framework answer.

---

## Things You Must Remember

- **Scalar** = single value (`NUMBER`, `VARCHAR2`, `DATE`, `BOOLEAN`, etc.). `BOOLEAN` is PL/SQL-only — never usable in SQL directly.
- **Vector/Composite** = multiple values grouped together (`RECORD`, `%ROWTYPE`, collections). Prefer `%ROWTYPE` over manually-declared `RECORD` whenever mirroring an actual table, for automatic structural adaptation.
- **Reference** = a pointer/reference to something else (e.g., `REF CURSOR`) — full mechanics sit outside our current syllabus, but the category is worth knowing exists.
- **LOB** = large, often unstructured data (`CLOB`, `BLOB`, `NCLOB`, `BFILE`) — requires specialized handling (e.g., `DBMS_LOB`) beyond ordinary scalar operations for anything non-trivial.
- `VARCHAR2` in PL/SQL can hold far more (32,767 bytes) than the traditional table-column limit (4,000 bytes) — these are genuinely different limits, easy to conflate.
- `PLS_INTEGER` is faster than `NUMBER` for pure integer arithmetic (especially loop counters) but has a fixed, smaller range and raises an overflow error if exceeded.

## How to Recognize This Concept

- Reach for **%ROWTYPE** whenever a requirement involves fetching or holding **"a full row"** or **"all the details"** of a table record as one unit.
- Reach for a manually-declared **RECORD** when you need a structure that **doesn't** correspond to any single table (e.g., combining fields from multiple tables, or purely computed fields) as one logical unit.
- Reach for **CLOB/BLOB** whenever a requirement mentions storing/handling **"documents," "large text," "images," "attachments,"** or anything clearly exceeding normal short-text limits.
- Reach for **PLS_INTEGER** specifically when a requirement or performance concern involves **heavy iteration/counting** (a strong forward-link to Module 2's iteration statements).

---

## Exercises

1. **(Scalar type selection)** For each of the following pieces of data, choose the most appropriate scalar datatype and briefly justify: a customer's age, a product's exact price (e.g., 19.99), a "yes/no" internal PL/SQL flag used only within a procedure, a 2-letter country code, a timestamp needing fractional-second precision.

2. **(CHAR vs VARCHAR2)** Predict what gets stored, and explain the practical difference, for:
   ```sql
   v_code CHAR(5) := 'AB';
   v_name VARCHAR2(5) := 'AB';
   ```

3. **(RECORD vs %ROWTYPE)** Given a table `products(product_id, product_name, price, stock_qty)`, write a block using `%ROWTYPE` to fetch and print one product's full details. Then, write an equivalent manually-declared `RECORD` version. Briefly explain, in your own words, one concrete situation where the `%ROWTYPE` version would continue working without changes while the manual `RECORD` version would need to be updated.

4. **(LOB recognition)** A requirement says: *"Store each employee's scanned signature image alongside their record."* Which datatype family and specific type would you use, and why would a normal `VARCHAR2` be unsuitable here?

5. **(PLS_INTEGER judgment)** A developer is writing a loop that will run up to 1 million iterations as a simple counter, purely for internal loop control (not stored anywhere, not used in any SQL). Would `NUMBER` or `PLS_INTEGER` be the better choice here, and why? What's the trade-off/risk to be aware of?

6. **(Boolean and SQL, connecting modules)** Explain, referencing what you learned in Module 3 about functions callable from SQL, why a function meant to be used inside a `SELECT` statement's column list should avoid returning `BOOLEAN`, and what it should return instead.

---

*Share your answers whenever you're ready. Next up: Module 1, Topic 5 — Operators & Variables in PL/SQL.*
