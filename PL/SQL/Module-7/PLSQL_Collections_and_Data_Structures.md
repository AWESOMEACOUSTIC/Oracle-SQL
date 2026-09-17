# PL/SQL Collections & Data Structures — Complete Study Guide

A single-file, exam-oriented reference covering PL/SQL collections from fundamentals to advanced usage, with hands-on exercises and answers.

---

## Table of Contents

1. [Overview of PL/SQL Collections](#1-overview-of-plsql-collections)
2. [Associative Arrays (Index-By Tables)](#2-associative-arrays-index-by-tables)
3. [Nested Tables](#3-nested-tables)
4. [VARRAYs](#4-varrays)
5. [Comparing the Three Collection Types](#5-comparing-the-three-collection-types)
6. [Iterating Over Collections](#6-iterating-over-collections)
7. [Storing Collections in Database Tables](#7-storing-collections-in-database-tables)
8. [The TABLE and CAST Operators](#8-the-table-and-cast-operators)
9. [Collection Methods & the UTL_COLL Package](#9-collection-methods--the-utl_coll-package)
10. [Working with JSON Data in PL/SQL](#10-working-with-json-data-in-plsql)
11. [Hands-On Exercises with Answers](#11-hands-on-exercises-with-answers)
12. [Exam-Style Quick Quiz (MCQs) with Answers](#12-exam-style-quick-quiz-mcqs-with-answers)
13. [Cheat Sheet Summary](#13-cheat-sheet-summary)

---

## 1. Overview of PL/SQL Collections

A **collection** is an ordered group of elements, all of the same data type. Think of it as a single-dimensional array, a list, or a lookup table — all rolled into PL/SQL's type system.

Oracle PL/SQL supports **three** collection types:

| Type | Also known as | Dimension | Dense/Sparse | Bounded? |
|---|---|---|---|---|
| **Associative Array** | Index-by table / PL/SQL table | Single | Sparse | No |
| **Nested Table** | — | Single | Dense (can become sparse) | No |
| **VARRAY** | Variable-size array | Single | Always dense | Yes (fixed max size) |

### Why collections matter
- They let you hold multiple values in one PL/SQL variable.
- They are the backbone of **BULK COLLECT** and **FORALL**, which drastically improve performance by reducing context switches between the SQL and PL/SQL engines.
- Nested Tables and VARRAYs can be **schema-level object types**, which means they can be stored as actual columns in database tables.
- Collections are essential to any exam covering PL/SQL because they combine syntax, object-type concepts, bulk-processing, and now JSON processing.

### General collection declaration pattern
```sql
TYPE type_name IS TABLE OF element_type [NOT NULL];             -- Nested table
TYPE type_name IS VARRAY(n) OF element_type [NOT NULL];         -- Varray
TYPE type_name IS TABLE OF element_type INDEX BY index_type;    -- Associative array
```

Where `element_type` can be a scalar (`NUMBER`, `VARCHAR2`), a `%TYPE`, a record, or another object type.

---

## 2. Associative Arrays (Index-By Tables)

### Key Characteristics
- Declared **only inside PL/SQL** (in a package, procedure, function, or anonymous block) — **cannot** be a column type in a database table.
- Indexed by either `PLS_INTEGER`/`BINARY_INTEGER` (numeric) or `VARCHAR2` (string-indexed — very handy for lookup maps).
- **Sparse** — indexes need not be sequential; you can jump from index 1 to index 100.
- **No need to initialize** — unlike nested tables/varrays, an associative array is ready to use as soon as declared (implicitly empty, not NULL).
- Cannot be stored in the database, cannot use MULTISET operators, cannot use CAST/TABLE with SQL directly.

### Syntax
```sql
DECLARE
    TYPE num_tab_t IS TABLE OF NUMBER INDEX BY PLS_INTEGER;
    TYPE name_map_t IS TABLE OF VARCHAR2(50) INDEX BY VARCHAR2(20);

    v_nums   num_tab_t;
    v_names  name_map_t;
BEGIN
    v_nums(1) := 100;
    v_nums(5) := 500;        -- sparse: index 2,3,4 don't exist

    v_names('IN') := 'India';
    v_names('US') := 'United States';

    DBMS_OUTPUT.PUT_LINE(v_nums(5));
    DBMS_OUTPUT.PUT_LINE(v_names('IN'));
END;
/
```

### Example: Associative array of a record type
```sql
DECLARE
    TYPE emp_rec_t IS RECORD (
        ename  VARCHAR2(50),
        sal    NUMBER
    );
    TYPE emp_tab_t IS TABLE OF emp_rec_t INDEX BY PLS_INTEGER;

    v_emps emp_tab_t;
BEGIN
    v_emps(1).ename := 'Ravi';
    v_emps(1).sal   := 55000;

    DBMS_OUTPUT.PUT_LINE(v_emps(1).ename || ' - ' || v_emps(1).sal);
END;
/
```

### Common Errors
- `ORA-06531: Reference to uninitialized collection` — happens with nested tables/varrays, **not** typically associative arrays (since they auto-initialize).
- `NO_DATA_FOUND` — raised if you reference an index that was never assigned (e.g., `v_nums(99)` when it doesn't exist).

---

## 3. Nested Tables

### Key Characteristics
- Can be declared **at schema level** (`CREATE TYPE ... IS TABLE OF ...`) so it can be used as a **column type** in a table, or **inside PL/SQL** only.
- **Must be initialized** with a constructor before elements can be added using index assignment, OR you can use `EXTEND` after initializing to an empty collection.
- Initially **dense** (indexes 1..n), but can become **sparse** after using `DELETE` on a middle element.
- Supports **MULTISET** operators (`UNION`, `INTERSECT`, `MINUS`) when used as a SQL collection type.
- No upper bound on size (grows dynamically).

### Declaring inside PL/SQL
```sql
DECLARE
    TYPE num_ntt IS TABLE OF NUMBER;
    v_nums num_ntt := num_ntt(10, 20, 30);   -- constructor call initializes
BEGIN
    v_nums.EXTEND;              -- add 1 slot
    v_nums(4) := 40;
    DBMS_OUTPUT.PUT_LINE(v_nums.COUNT);      -- 4
END;
/
```

### Declaring at schema level (for DB storage)
```sql
CREATE TYPE phone_list_t AS TABLE OF VARCHAR2(15);
/
```
This creates a reusable SQL object type that can now be used:
- Inside PL/SQL blocks
- As a column data type in a relational table (see Section 7)

### Uninitialized Nested Table Trap
```sql
DECLARE
    TYPE num_ntt IS TABLE OF NUMBER;
    v_nums num_ntt;              -- NULL collection, NOT empty!
BEGIN
    v_nums(1) := 10;             -- ORA-06531: Reference to uninitialized collection
END;
/
```
**Fix:**
```sql
v_nums := num_ntt();     -- initialize to empty, then EXTEND
v_nums.EXTEND;
v_nums(1) := 10;
```

---

## 4. VARRAYs

### Key Characteristics
- **Fixed maximum size** declared upfront: `VARRAY(n)`.
- Always **dense** — you cannot delete an individual middle element (only `TRIM` from the end, or `DELETE` the entire array).
- Elements are **ordered** and accessed positionally.
- Like nested tables, can be declared at schema level for DB storage, or locally inside PL/SQL.
- Must be initialized via constructor before use.

### Syntax
```sql
DECLARE
    TYPE weekday_va IS VARRAY(7) OF VARCHAR2(10);
    v_days weekday_va := weekday_va('Mon','Tue','Wed','Thu','Fri','Sat','Sun');
BEGIN
    DBMS_OUTPUT.PUT_LINE(v_days(1));      -- Mon
    DBMS_OUTPUT.PUT_LINE(v_days.COUNT);   -- 7
    DBMS_OUTPUT.PUT_LINE(v_days.LIMIT);   -- 7 (max capacity)
END;
/
```

### Schema-level VARRAY
```sql
CREATE TYPE top3_scores_t AS VARRAY(3) OF NUMBER;
/
```

### Exceeding the limit
```sql
v_days.EXTEND;   -- ORA-06532: Subscript outside of limit, if already at max size 7
```

---

## 5. Comparing the Three Collection Types

| Feature | Associative Array | Nested Table | VARRAY |
|---|---|---|---|
| Where declared | PL/SQL only | PL/SQL or Schema level | PL/SQL or Schema level |
| Can be a DB column type | ❌ No | ✅ Yes | ✅ Yes |
| Indexing | Sparse, integer or string | Dense initially, can be sparse | Dense, fixed order |
| Needs constructor/initialization | ❌ No | ✅ Yes | ✅ Yes |
| Max size limit | None | None | Fixed at declaration |
| Supports DELETE(middle element) | ✅ Yes | ✅ Yes | ❌ No |
| Storage in DB | N/A | Separate storage table (out-of-line) | Stored in-line (like a LOB/RAW) usually |
| MULTISET operators (SQL context) | ❌ No | ✅ Yes | ✅ Yes |
| Typical use case | In-memory lookup/cache map | Unbounded list stored in DB or transient result sets | Small, fixed-size ordered list (e.g., days of week, RGB values) |

---

## 6. Iterating Over Collections

### a) Numeric FOR loop with FIRST/LAST (best for dense collections)
```sql
DECLARE
    TYPE num_ntt IS TABLE OF NUMBER;
    v_nums num_ntt := num_ntt(10, 20, 30, 40);
BEGIN
    FOR i IN v_nums.FIRST .. v_nums.LAST LOOP
        DBMS_OUTPUT.PUT_LINE(v_nums(i));
    END LOOP;
END;
/
```

### b) WHILE loop with FIRST/NEXT (safe for sparse collections)
```sql
DECLARE
    TYPE num_ntt IS TABLE OF NUMBER;
    v_nums num_ntt := num_ntt(10, 20, 30, 40);
    v_idx  PLS_INTEGER;
BEGIN
    v_nums.DELETE(2);              -- now sparse
    v_idx := v_nums.FIRST;
    WHILE v_idx IS NOT NULL LOOP
        DBMS_OUTPUT.PUT_LINE(v_nums(v_idx));
        v_idx := v_nums.NEXT(v_idx);
    END LOOP;
END;
/
```
> **Tip:** always prefer `FIRST/NEXT` (or `LAST/PRIOR`) over a numeric `FOR` loop when a collection **might be sparse** (associative arrays or nested tables after `DELETE`), otherwise you'll hit `NO_DATA_FOUND` on missing indexes.

### c) FOR loop directly over collection (Oracle 21c+ simplified iteration)
```sql
FOR val IN VALUES OF v_nums LOOP     -- newer syntax, returns element values directly
    DBMS_OUTPUT.PUT_LINE(val);
END LOOP;
```
> Older/most-common exam syntax still expects `FIRST..LAST` or `FIRST/NEXT`, so know both.

### d) BULK COLLECT — loading a collection from SQL in one shot
```sql
DECLARE
    TYPE ename_ntt IS TABLE OF employees.last_name%TYPE;
    v_names ename_ntt;
BEGIN
    SELECT last_name
    BULK COLLECT INTO v_names
    FROM employees
    WHERE department_id = 50;

    FOR i IN 1 .. v_names.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE(v_names(i));
    END LOOP;
END;
/
```

### e) FORALL — bulk DML using a collection (huge performance boost)
```sql
DECLARE
    TYPE id_ntt IS TABLE OF employees.employee_id%TYPE;
    v_ids id_ntt := id_ntt(100, 101, 102);
BEGIN
    FORALL i IN v_ids.FIRST .. v_ids.LAST
        UPDATE employees
        SET salary = salary * 1.10
        WHERE employee_id = v_ids(i);

    COMMIT;
END;
/
```
> `FORALL` sends the entire batch of DML statements to the SQL engine in one context switch instead of one-row-at-a-time, which is a very common exam topic paired with collections.

---

## 7. Storing Collections in Database Tables

Only **schema-level Nested Tables and VARRAYs** can be stored as table columns (associative arrays cannot).

### a) VARRAY column (stored inline)
```sql
CREATE TYPE phone_va AS VARRAY(3) OF VARCHAR2(15);
/

CREATE TABLE contacts (
    contact_id   NUMBER PRIMARY KEY,
    contact_name VARCHAR2(50),
    phones       phone_va
);

INSERT INTO contacts VALUES (
    1, 'Arjun', phone_va('9876543210', '9123456789')
);

SELECT * FROM contacts;
```

### b) Nested Table column (stored out-of-line, in a separate storage table)
```sql
CREATE TYPE skill_ntt AS TABLE OF VARCHAR2(30);
/

CREATE TABLE employees_skills (
    emp_id NUMBER PRIMARY KEY,
    ename  VARCHAR2(50),
    skills skill_ntt
)
NESTED TABLE skills STORE AS skills_storage_tab;   -- mandatory storage clause for nested tables

INSERT INTO employees_skills VALUES (
    1, 'Meena', skill_ntt('PL/SQL', 'SQL Tuning', 'Data Modeling')
);
```
> The `NESTED TABLE ... STORE AS ...` clause is **mandatory** — Oracle needs a physical table to store the (potentially unbounded) rows of the nested table. VARRAYs do **not** need this because they have a bounded size and are stored inline (or as a LOB if large).

### c) Querying nested rows with TABLE()
```sql
SELECT e.ename, s.COLUMN_VALUE AS skill
FROM   employees_skills e,
       TABLE(e.skills) s;
```

### d) Updating a stored collection
```sql
UPDATE employees_skills e
SET    skills = skill_ntt('PL/SQL', 'SQL Tuning', 'Data Modeling', 'AI/ML')
WHERE  emp_id = 1;
```

### e) Adding a single element to a stored nested table without replacing the whole collection
```sql
INSERT INTO TABLE(SELECT skills FROM employees_skills WHERE emp_id = 1)
VALUES ('APEX');
```

---

## 8. The TABLE and CAST Operators

### TABLE() Operator
Converts a **collection** into a **row-source (virtual table)** so it can be used inside a `FROM` clause of a SQL statement.

```sql
SELECT COLUMN_VALUE AS skill
FROM   TABLE(skill_ntt('Java', 'PL/SQL', 'Python'));
```

Used heavily to:
- Query nested-table columns (as shown above).
- Pass a PL/SQL collection into SQL, e.g., from a function that returns a collection.

```sql
CREATE OR REPLACE FUNCTION get_top_scores RETURN top3_scores_t IS
BEGIN
    RETURN top3_scores_t(95, 88, 76);
END;
/

SELECT * FROM TABLE(get_top_scores());
```

### CAST() Operator
Used to explicitly convert one collection type to another compatible collection type, or to convert the output of a subquery/`MULTISET` expression into a strongly typed collection so it matches an expected type.

```sql
SELECT CAST(MULTISET(SELECT last_name FROM employees WHERE department_id = 50) AS ename_ntt)
FROM dual;
```

### CAST + TABLE combined (very common exam pattern)
```sql
SELECT *
FROM TABLE(
        CAST(
            MULTISET(SELECT department_name FROM departments WHERE location_id = 1700)
            AS dept_name_ntt
        )
     );
```

### MULTISET Operators (bonus, frequently tested alongside TABLE/CAST)
Nested tables support set-like operations:
```sql
DECLARE
    TYPE num_ntt IS TABLE OF NUMBER;
    a num_ntt := num_ntt(1,2,3);
    b num_ntt := num_ntt(2,3,4);
    v_result num_ntt;
BEGIN
    v_result := a MULTISET UNION b;         -- (1,2,3,2,3,4)
    v_result := a MULTISET UNION DISTINCT b; -- (1,2,3,4)
    v_result := a MULTISET INTERSECT b;      -- (2,3)
    v_result := a MULTISET EXCEPT b;         -- (1)  [EXCEPT = MINUS]
END;
/
```

---

## 9. Collection Methods & the UTL_COLL Package

> **Note for exam clarity:** There is no Oracle-supplied package literally named `DBMS_COLLECTION_UTL`. What exams and textbooks usually mean under "collection utilities" is one (or both) of the following, so both are covered here:
> 1. The **built-in collection methods** (pseudo-functions/procedures called using dot notation on any collection variable).
> 2. The **`UTL_COLL`** package, which supplies **locator functions** for VARRAYs and Nested Tables used mainly in OCI/Pro*C client-side programming.

### a) Built-in Collection Methods (the ones tested most)

| Method | Type | Purpose |
|---|---|---|
| `COUNT` | Function | Number of elements currently in the collection |
| `LIMIT` | Function | Max size (VARRAY only); returns `NULL` for nested tables/assoc. arrays |
| `FIRST` | Function | Lowest index in use (`NULL` if empty) |
| `LAST` | Function | Highest index in use (`NULL` if empty) |
| `NEXT(n)` | Function | Index that follows index `n` (`NULL` if none) |
| `PRIOR(n)` | Function | Index that precedes index `n` |
| `EXISTS(n)` | Function | `TRUE` if element at index `n` exists |
| `EXTEND[(n[,i])]` | Procedure | Appends `n` elements (nested tables/varrays only) |
| `TRIM[(n)]` | Procedure | Removes `n` elements from the end (nested tables/varrays only) |
| `DELETE[(n [,m])]` | Procedure | Deletes element(s); with no args, deletes all |

```sql
DECLARE
    TYPE num_ntt IS TABLE OF NUMBER;
    v_nums num_ntt := num_ntt(10, 20, 30);
BEGIN
    DBMS_OUTPUT.PUT_LINE(v_nums.COUNT);        -- 3
    v_nums.EXTEND(2);                          -- adds 2 NULL slots => count 5
    v_nums(4) := 40;
    v_nums(5) := 50;
    v_nums.TRIM(1);                            -- removes last element => count 4
    v_nums.DELETE(2);                          -- makes it sparse
    IF v_nums.EXISTS(2) THEN
        DBMS_OUTPUT.PUT_LINE('exists');
    ELSE
        DBMS_OUTPUT.PUT_LINE('index 2 deleted');
    END IF;
END;
/
```

### b) UTL_COLL Package
Provides locator access to VARRAYs/Nested Tables that are stored in the database — mainly relevant for external client programs (Pro*C/OCI) that need a "locator" (a pointer/reference) rather than pulling the whole collection into memory.

Key member:
```sql
UTL_COLL.GETVARRAY(
    locator IN ANYDATA
) RETURN ANYDATA;
```
- Retrieves a VARRAY locator wrapped inside an `ANYDATA` value pulled from a LOB locator context, primarily used in advanced Pro*C/OCI programming — rarely hand-coded in ordinary PL/SQL application code, but good to know it exists for certification-level "package name recognition" questions.

> **Exam tip:** If you see a question naming an unfamiliar "collection utility package," check whether it's really testing the **built-in collection methods** table above — that's what 90% of certification questions actually target.

---

## 10. Working with JSON Data in PL/SQL

Modern PL/SQL (12c+) has first-class JSON support, and it interacts heavily with collections since JSON arrays map naturally to PL/SQL collections.

### a) JSON storage and basic validation
```sql
CREATE TABLE orders_json (
    id      NUMBER PRIMARY KEY,
    payload CLOB
    CONSTRAINT ensure_json CHECK (payload IS JSON)
);

INSERT INTO orders_json VALUES (
    1,
    '{"order_id":101,"customer":"Ravi","items":["Pen","Notebook","Bag"],"total":450}'
);
```
> From Oracle 21c you can also declare a native `JSON` data type column directly (`payload JSON`), which stores an optimized binary format instead of text.

### b) Querying JSON with dot notation / JSON_VALUE
```sql
SELECT o.payload.customer,
       o.payload.total
FROM   orders_json o
WHERE  o.id = 1;

SELECT JSON_VALUE(payload, '$.customer') AS customer_name
FROM   orders_json;
```

### c) JSON_TABLE — projecting a JSON array into relational rows
```sql
SELECT jt.item_name
FROM   orders_json o,
       JSON_TABLE(
           o.payload, '$.items[*]'
           COLUMNS (item_name VARCHAR2(50) PATH '$')
       ) jt
WHERE  o.id = 1;
```

### d) Using PL/SQL object types JSON_OBJECT_T / JSON_ARRAY_T
Oracle provides ready-made object types to parse and build JSON directly in PL/SQL — these methods return/accept PL/SQL collections and are commonly tested together with collections.

```sql
DECLARE
    v_json   JSON_OBJECT_T;
    v_items  JSON_ARRAY_T;
    v_item   VARCHAR2(100);
BEGIN
    v_json  := JSON_OBJECT_T.PARSE(
                  '{"order_id":101,"customer":"Ravi","items":["Pen","Notebook","Bag"]}'
               );

    DBMS_OUTPUT.PUT_LINE('Customer: ' || v_json.GET_STRING('customer'));

    v_items := v_json.GET_ARRAY('items');

    FOR i IN 0 .. v_items.GET_SIZE - 1 LOOP     -- JSON arrays are 0-indexed!
        v_item := v_items.GET_STRING(i);
        DBMS_OUTPUT.PUT_LINE('Item ' || (i+1) || ': ' || v_item);
    END LOOP;
END;
/
```

### e) Converting a JSON array into a PL/SQL nested table
```sql
DECLARE
    TYPE item_ntt IS TABLE OF VARCHAR2(100);
    v_items_arr JSON_ARRAY_T := JSON_ARRAY_T.PARSE('["Pen","Notebook","Bag"]');
    v_items     item_ntt := item_ntt();
BEGIN
    FOR i IN 0 .. v_items_arr.GET_SIZE - 1 LOOP
        v_items.EXTEND;
        v_items(v_items.COUNT) := v_items_arr.GET_STRING(i);
    END LOOP;

    FOR i IN 1 .. v_items.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE(v_items(i));
    END LOOP;
END;
/
```

### f) Building JSON from a PL/SQL collection
```sql
DECLARE
    v_arr JSON_ARRAY_T := JSON_ARRAY_T();
BEGIN
    v_arr.APPEND('Pen');
    v_arr.APPEND('Notebook');
    v_arr.APPEND('Bag');

    DBMS_OUTPUT.PUT_LINE(v_arr.TO_STRING);   -- ["Pen","Notebook","Bag"]
END;
/
```

### g) JSON_ARRAYAGG / JSON_OBJECT — SQL-level JSON generation from collections/tables
```sql
SELECT JSON_ARRAYAGG(last_name ORDER BY last_name)
FROM   employees
WHERE  department_id = 50;

SELECT JSON_OBJECT('id' VALUE employee_id, 'name' VALUE last_name)
FROM   employees
WHERE  employee_id = 100;
```

---

## 11. Hands-On Exercises with Answers

Try to solve each exercise yourself first, then check the answer.

---

### Exercise 1 — Associative Array Basics
**Task:** Declare an associative array indexed by `VARCHAR2` that maps country codes to country names for `'IN'`, `'US'`, `'UK'`. Print all three using a `FIRST/NEXT` loop.

<details>
<summary>Answer</summary>

```sql
DECLARE
    TYPE country_map_t IS TABLE OF VARCHAR2(50) INDEX BY VARCHAR2(5);
    v_countries country_map_t;
    v_key       VARCHAR2(5);
BEGIN
    v_countries('IN') := 'India';
    v_countries('US') := 'United States';
    v_countries('UK') := 'United Kingdom';

    v_key := v_countries.FIRST;
    WHILE v_key IS NOT NULL LOOP
        DBMS_OUTPUT.PUT_LINE(v_key || ' -> ' || v_countries(v_key));
        v_key := v_countries.NEXT(v_key);
    END LOOP;
END;
/
```
</details>

---

### Exercise 2 — Nested Table Initialization Trap
**Task:** Explain what is wrong with this code and fix it.
```sql
DECLARE
    TYPE names_ntt IS TABLE OF VARCHAR2(30);
    v_names names_ntt;
BEGIN
    v_names(1) := 'Anu';
END;
/
```

<details>
<summary>Answer</summary>

**Problem:** `v_names` is declared but never initialized with a constructor, so it is `NULL` (an atomically null collection), not empty. Assigning `v_names(1)` raises `ORA-06531: Reference to uninitialized collection`.

**Fix:**
```sql
DECLARE
    TYPE names_ntt IS TABLE OF VARCHAR2(30);
    v_names names_ntt := names_ntt();   -- initialize to empty collection
BEGIN
    v_names.EXTEND;
    v_names(1) := 'Anu';
    DBMS_OUTPUT.PUT_LINE(v_names(1));
END;
/
```
</details>

---

### Exercise 3 — VARRAY Limit Enforcement
**Task:** Create a `VARRAY(5)` of `NUMBER`, fill it with 5 values, then attempt to add a 6th and handle the resulting exception gracefully.

<details>
<summary>Answer</summary>

```sql
DECLARE
    TYPE score_va IS VARRAY(5) OF NUMBER;
    v_scores score_va := score_va(90, 85, 78, 92, 88);
BEGIN
    v_scores.EXTEND;              -- exceeds LIMIT of 5
    v_scores(6) := 100;
EXCEPTION
    WHEN OTHERS THEN
        IF SQLCODE = -6532 THEN
            DBMS_OUTPUT.PUT_LINE('Cannot exceed VARRAY limit of ' || v_scores.LIMIT);
        ELSE
            RAISE;
        END IF;
END;
/
```
</details>

---

### Exercise 4 — BULK COLLECT + FORALL
**Task:** Using the `employees` table, bulk-collect all `employee_id`s from department 60, then use `FORALL` to give each of them a 5% raise.

<details>
<summary>Answer</summary>

```sql
DECLARE
    TYPE id_ntt IS TABLE OF employees.employee_id%TYPE;
    v_ids id_ntt;
BEGIN
    SELECT employee_id
    BULK COLLECT INTO v_ids
    FROM employees
    WHERE department_id = 60;

    FORALL i IN v_ids.FIRST .. v_ids.LAST
        UPDATE employees
        SET    salary = salary * 1.05
        WHERE  employee_id = v_ids(i);

    DBMS_OUTPUT.PUT_LINE(SQL%ROWCOUNT || ' rows updated');
    COMMIT;
END;
/
```
</details>

---

### Exercise 5 — Storing and Querying a Nested Table Column
**Task:** Create a schema-level nested table type for hobbies, a table `students(id, name, hobbies)`, insert one row with 3 hobbies, then write a query that lists each hobby on its own row alongside the student name.

<details>
<summary>Answer</summary>

```sql
CREATE TYPE hobby_ntt AS TABLE OF VARCHAR2(30);
/

CREATE TABLE students (
    id      NUMBER PRIMARY KEY,
    name    VARCHAR2(50),
    hobbies hobby_ntt
)
NESTED TABLE hobbies STORE AS hobbies_store_tab;

INSERT INTO students VALUES (
    1, 'Divya', hobby_ntt('Reading', 'Painting', 'Chess')
);

SELECT s.name, h.COLUMN_VALUE AS hobby
FROM   students s,
       TABLE(s.hobbies) h;
```
</details>

---

### Exercise 6 — CAST + MULTISET + TABLE Combined
**Task:** Write a query that returns department names (as a row-source) for departments located at `location_id = 1700`, using `MULTISET`, `CAST`, and `TABLE` together. Assume a nested table type `dept_name_ntt AS TABLE OF VARCHAR2(50)` already exists.

<details>
<summary>Answer</summary>

```sql
CREATE TYPE dept_name_ntt AS TABLE OF VARCHAR2(50);
/

SELECT *
FROM TABLE(
        CAST(
            MULTISET(
                SELECT department_name
                FROM   departments
                WHERE  location_id = 1700
            ) AS dept_name_ntt
        )
     );
```
</details>

---

### Exercise 7 — Collection Methods Practice
**Task:** Given `v_nums` a nested table `(10, 20, 30, 40, 50)`, write code to:
1. Delete the element at index 3.
2. Check whether index 3 still exists.
3. Print the total count after deletion.
4. Loop safely and print remaining elements.

<details>
<summary>Answer</summary>

```sql
DECLARE
    TYPE num_ntt IS TABLE OF NUMBER;
    v_nums num_ntt := num_ntt(10, 20, 30, 40, 50);
    v_idx  PLS_INTEGER;
BEGIN
    v_nums.DELETE(3);

    IF v_nums.EXISTS(3) THEN
        DBMS_OUTPUT.PUT_LINE('Index 3 exists');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Index 3 was deleted');
    END IF;

    DBMS_OUTPUT.PUT_LINE('Count: ' || v_nums.COUNT);   -- 4 (COUNT ignores deleted slots)

    v_idx := v_nums.FIRST;
    WHILE v_idx IS NOT NULL LOOP
        DBMS_OUTPUT.PUT_LINE(v_nums(v_idx));
        v_idx := v_nums.NEXT(v_idx);
    END LOOP;
END;
/
```
</details>

---

### Exercise 8 — Parsing JSON into a PL/SQL Collection
**Task:** Given the JSON string
```json
{"student":"Kiran","subjects":["Maths","Science","English"]}
```
write PL/SQL to parse it and load the `subjects` array into a nested table, then print each subject.

<details>
<summary>Answer</summary>

```sql
DECLARE
    TYPE subject_ntt IS TABLE OF VARCHAR2(50);
    v_json     JSON_OBJECT_T;
    v_arr      JSON_ARRAY_T;
    v_subjects subject_ntt := subject_ntt();
BEGIN
    v_json := JSON_OBJECT_T.PARSE(
        '{"student":"Kiran","subjects":["Maths","Science","English"]}'
    );

    v_arr := v_json.GET_ARRAY('subjects');

    FOR i IN 0 .. v_arr.GET_SIZE - 1 LOOP
        v_subjects.EXTEND;
        v_subjects(v_subjects.COUNT) := v_arr.GET_STRING(i);
    END LOOP;

    DBMS_OUTPUT.PUT_LINE('Student: ' || v_json.GET_STRING('student'));
    FOR i IN 1 .. v_subjects.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE('Subject ' || i || ': ' || v_subjects(i));
    END LOOP;
END;
/
```
</details>

---

### Exercise 9 — Building JSON from Query Results
**Task:** Write a single SQL statement that returns a JSON array of employee last names for department 50, using `JSON_ARRAYAGG`.

<details>
<summary>Answer</summary>

```sql
SELECT JSON_ARRAYAGG(last_name ORDER BY last_name)
FROM   employees
WHERE  department_id = 50;
```
</details>

---

### Exercise 10 — VARRAY vs Nested Table Design Decision
**Task:** You need to store a list of up to 4 "favorite colors" per user (fixed max), and a separate, potentially unlimited list of "order history item codes" per user. Which collection type fits each requirement, and why?

<details>
<summary>Answer</summary>

- **Favorite colors (max 4, fixed size, order matters):** Use a **VARRAY(4)** — the bound matches the business rule exactly, storage is inline and compact, and dense ordered access fits the use case.
- **Order history item codes (unbounded, can grow indefinitely):** Use a **Nested Table** — no upper bound, stored out-of-line in its own storage table so it can grow arbitrarily large without impacting the base table's row size, and supports `MULTISET` operations if you ever need to compare/merge histories.
</details>

---

## 12. Exam-Style Quick Quiz (MCQs) with Answers

**Q1.** Which collection type can be indexed by a `VARCHAR2` value?
A) Nested Table  B) VARRAY  C) Associative Array  D) All of the above

<details><summary>Answer</summary>C) Associative Array — associative arrays alone allow string ("BINARY_INTEGER or VARCHAR2") indexing.</details>

---

**Q2.** Which statement about VARRAYs is FALSE?
A) Has a maximum size defined at declaration
B) Can be stored as a database column
C) Can become sparse after `DELETE(n)` on a middle element
D) Must be initialized with a constructor before use

<details><summary>Answer</summary>C — VARRAYs are always dense; you cannot delete an individual middle element. Only `TRIM` (from the end) or `DELETE` (entire collection) is allowed.</details>

---

**Q3.** What error is raised when referencing an index in an uninitialized nested table?
A) NO_DATA_FOUND  B) ORA-06531  C) ORA-06532  D) VALUE_ERROR

<details><summary>Answer</summary>B) ORA-06531: Reference to uninitialized collection.</details>

---

**Q4.** Which clause is mandatory when creating a table with a nested-table column?
A) `VARRAY(n)`  B) `NESTED TABLE col STORE AS storage_table`  C) `CONSTRAINT ensure_json`  D) None required

<details><summary>Answer</summary>B — Oracle requires a physical storage table for a nested table column.</details>

---

**Q5.** Which operator allows a PL/SQL collection to be queried as if it were a database table in a SQL `FROM` clause?
A) CAST  B) MULTISET  C) TABLE  D) BULK COLLECT

<details><summary>Answer</summary>C) TABLE()</details>

---

**Q6.** In `JSON_ARRAY_T`, array elements are indexed starting from:
A) 1  B) 0  C) -1  D) Depends on JSON content

<details><summary>Answer</summary>B) 0 — JSON arrays accessed via `JSON_ARRAY_T.GET_*` methods are zero-indexed, unlike PL/SQL nested tables which default to starting at 1.</details>

---

**Q7.** Which collection method returns `NULL` when called on a Nested Table (as opposed to a VARRAY)?
A) COUNT  B) FIRST  C) LIMIT  D) LAST

<details><summary>Answer</summary>C) LIMIT — Nested tables have no maximum size, so `LIMIT` always returns NULL for them; only VARRAYs return a meaningful bound.</details>

---

**Q8.** What is the main performance benefit of using `FORALL` instead of a simple `FOR` loop with individual DML statements?
A) It uses less memory
B) It sends the entire batch of DML to the SQL engine in one context switch
C) It automatically commits after every statement
D) It removes the need for a collection

<details><summary>Answer</summary>B — Reducing PL/SQL-to-SQL context switches is the core performance win of `FORALL` combined with `BULK COLLECT`.</details>

---

## 13. Cheat Sheet Summary

```
ASSOCIATIVE ARRAY   -> PL/SQL only, sparse, string/int index, no init needed
NESTED TABLE        -> PL/SQL or DB column, dense->sparse, needs init + STORE AS clause in DB
VARRAY               -> PL/SQL or DB column, always dense, fixed max size (LIMIT), needs init

Methods:  COUNT | LIMIT | FIRST | LAST | NEXT(n) | PRIOR(n) | EXISTS(n) | EXTEND | TRIM | DELETE

Iteration:
  FOR i IN v.FIRST..v.LAST LOOP ...        -- dense collections
  v_idx := v.FIRST; WHILE v_idx IS NOT NULL LOOP ... v_idx := v.NEXT(v_idx); END LOOP;  -- sparse-safe

Bulk SQL:
  SELECT ... BULK COLLECT INTO collection FROM table;
  FORALL i IN coll.FIRST..coll.LAST  <single DML statement using coll(i)>;

SQL <-> Collection bridge:
  TABLE(collection_expr)                     -- collection as row source
  CAST(MULTISET(subquery) AS collection_type) -- subquery as strongly-typed collection
  MULTISET UNION / INTERSECT / EXCEPT         -- set ops on nested tables

JSON:
  JSON_OBJECT_T.PARSE(str), .GET_STRING(key), .GET_ARRAY(key)
  JSON_ARRAY_T.PARSE(str), .GET_SIZE, .GET_STRING(i)  [0-indexed]
  JSON_TABLE(json_col, '$.path[*]' COLUMNS (...))     -- JSON array -> relational rows
  JSON_ARRAYAGG(expr), JSON_OBJECT(key VALUE val)      -- SQL rows -> JSON
```

---

**Good luck with your exam!** Work through Section 11 exercises without peeking at the answers first — that's the best predictor of exam readiness for this topic.
