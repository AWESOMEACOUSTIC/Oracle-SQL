# Oracle PL/SQL Collections — Complete Study Guide

*A single-file exam-prep reference covering collections, storage, JSON, bulk processing, pipelined/table functions, and performance tuning.*

## Table of Contents

1. [Overview of PL/SQL Collections](#1-overview-of-plsql-collections)
2. [Associative Arrays, Nested Tables, and VARRAYs](#2-associative-arrays-nested-tables-and-varrays)
3. [Iterating and Storing Collections in Tables](#3-iterating-and-storing-collections-in-tables)
4. [The TABLE and CAST Operators](#4-the-table-and-cast-operators)
5. [The UTL_COLL Package (Collection Locators)](#5-the-utl_coll-package-collection-locators)
6. [Working with JSON Data in PL/SQL](#6-working-with-json-data-in-plsql)
7. [Exception Handling in Collections — SAVE EXCEPTIONS](#7-exception-handling-in-collections--save-exceptions)
8. [BULK COLLECT and FORALL](#8-bulk-collect-and-forall)
9. [Pipelined Table Functions](#9-pipelined-table-functions)
10. [Table Functions (General)](#10-table-functions-general)
11. [Collection Performance Tuning](#11-collection-performance-tuning)
12. [Master Comparison Cheat-Sheet](#12-master-comparison-cheat-sheet)
13. [Practice Exercises](#13-practice-exercises)
14. [Answer Key](#14-answer-key)

---

## 1. Overview of PL/SQL Collections

A **collection** is an ordered group of elements, all of the **same data type**, accessed by a subscript (index). Think of it as PL/SQL's version of an array/list, but more flexible.

### 1.1 Why collections exist
- Hold a set of values in memory instead of one at a time.
- Enable **bulk operations** (BULK COLLECT / FORALL) that drastically cut context switches between the PL/SQL and SQL engines.
- Allow a single column of a database table to store a repeating group of values (nested tables / varrays as column types).
- Act as the return type of **pipelined** and **table** functions, letting a PL/SQL function be queried like a table.

### 1.2 The three collection types (memorize this)

| Type | Analogy |
|---|---|
| **Associative Array** (a.k.a. Index-By Table / PL/SQL Table) | A hash map / dictionary |
| **Nested Table** | A resizable set/list, can be a DB column type |
| **VARRAY** (Variable-size Array) | A fixed-maximum-size ordered list |

### 1.3 Collection pseudo-columns / attributes — none exist; use **methods** instead

Collections do **not** have properties accessed with dot notation for size etc.; PL/SQL exposes **collection methods**:

| Method | Type | Purpose |
|---|---|---|
| `EXISTS(n)` | Function | TRUE if element at index *n* exists |
| `COUNT` | Function | Number of elements currently in the collection |
| `LIMIT` | Function | Maximum size (VARRAY only; NULL/none for others) |
| `FIRST` | Function | Lowest index in use (NULL if empty) |
| `LAST` | Function | Highest index in use (NULL if empty) |
| `PRIOR(n)` | Function | Index before *n* |
| `NEXT(n)` | Function | Index after *n* |
| `EXTEND` / `EXTEND(n)` / `EXTEND(n,i)` | Procedure | Adds elements (Nested Table & VARRAY only) |
| `TRIM` / `TRIM(n)` | Procedure | Removes elements from the **end** (Nested Table & VARRAY only) |
| `DELETE` / `DELETE(n)` / `DELETE(m,n)` | Procedure | Removes elements (Associative Array & Nested Table only — **not VARRAY**) |

> ⚠️ **Exam trap:** `DELETE` and `TRIM` are **not interchangeable**. `TRIM` removes from the end only and physically shrinks storage (no gaps possible). `DELETE(n)` can remove an element **from the middle**, creating a **sparse** collection with a gap at index *n*. `EXTEND`/`TRIM`/`DELETE` do **not** apply to associative arrays' `EXTEND`/`TRIM` (assoc arrays auto-grow when you assign to a new key, and don't support `TRIM`).

### 1.4 Collections are single-type and (mostly) single-level
Each collection holds elements of **one declared type** — scalar, `%TYPE`, record, object type, or (with restrictions) another collection ("multi-level" or "nested" collections — supported in SQL from 9i R2 onward but rarely tested; avoid unless the syllabus explicitly names it).

### 1.5 Quick declaration cheat
```sql
-- Associative Array (PL/SQL only, cannot be a table column)
TYPE t_assoc IS TABLE OF employees.salary%TYPE INDEX BY PLS_INTEGER;

-- Nested Table (can be a SQL/schema-level type -> table column)
TYPE t_nt IS TABLE OF VARCHAR2(30);

-- VARRAY (can be a SQL/schema-level type -> table column)
TYPE t_va IS VARRAY(10) OF NUMBER;
```

---

## 2. Associative Arrays, Nested Tables, and VARRAYs

### 2.1 Associative Arrays (Index-By Tables)

```sql
DECLARE
  TYPE t_sal_tab IS TABLE OF NUMBER INDEX BY PLS_INTEGER;
  v_sal t_sal_tab;
BEGIN
  v_sal(1)   := 50000;      -- no need to initialize/constructor
  v_sal(100) := 75000;      -- sparse: index 2..99 don't exist
  DBMS_OUTPUT.PUT_LINE(v_sal.COUNT);  -- 2
END;
/
```
- Index can be `PLS_INTEGER`/`BINARY_INTEGER` **or `VARCHAR2`** (string-indexed / "indexed by varchar2" — very common exam question).
- **Always sparse-capable** and **unbounded**.
- **No initialization** needed (default value is an *empty but usable* collection — NOT NULL like nested tables).
- **PL/SQL-only** — cannot be a column type in a database table, cannot be used directly with the `TABLE()` operator, cannot be a parameter of a function called from SQL unless it is declared in a package spec (from 12c associative arrays with `INDEX BY PLS_INTEGER` declared in a **package specification** can be passed to/from procedures callable from SQL... but this is niche — the safe exam answer is "associative arrays cannot be stored in the database and cannot be used with TABLE()/CAST()").
- Not persistent — exists only for the duration of the session/block (unless declared in a package spec, in which case it persists for the session).

### 2.2 Nested Tables

```sql
-- Schema-level (SQL) type — required if you want a table column or TABLE()/CAST()
CREATE TYPE t_names_nt AS TABLE OF VARCHAR2(30);
/

DECLARE
  v_names t_names_nt := t_names_nt('Asha','Bala','Chitra');  -- constructor required
BEGIN
  v_names.EXTEND;              -- add one NULL slot
  v_names(4) := 'Deepa';
  v_names.DELETE(2);           -- now sparse: index 2 is gone
  DBMS_OUTPUT.PUT_LINE(v_names.COUNT); -- 3
END;
/
```
- Must be **initialized** with a constructor (or set to an empty constructor `t_names_nt()`), otherwise it is **atomically NULL** and referencing an element raises `ORA-06531: Reference to uninitialized collection`.
- **Unbounded**, starts **dense** (1..n) but can become **sparse** after `DELETE`.
- **Can be a database column type** (needs a `NESTED TABLE ... STORE AS` clause).
- No inherent order guarantee when fetched back from the database (Oracle does **not** promise the retrieval order equals insertion order for a stored nested table) — a very common exam trick question.
- Supports `MULTISET` set operations (`MULTISET UNION`, `MULTISET INTERSECT`, `MULTISET EXCEPT`) and can be compared for equality with `=` / `IN` / `SUBMULTISET OF` in SQL.

### 2.3 VARRAYs (Variable-size Arrays)

```sql
CREATE TYPE t_phones_va AS VARRAY(3) OF VARCHAR2(15);
/

DECLARE
  v_phones t_phones_va := t_phones_va('99900011','99900022');
BEGIN
  v_phones.EXTEND;
  v_phones(3) := '99900033';
  -- v_phones.EXTEND;  -- would raise ORA-06532: Subscript outside limit (LIMIT=3)
  DBMS_OUTPUT.PUT_LINE(v_phones.LIMIT); -- 3
END;
/
```
- Declared with a **fixed maximum size** (`LIMIT`), e.g., `VARRAY(3)`.
- **Always dense** — you can never delete from the middle; `DELETE` is **not a legal method for VARRAYs at all** (only `TRIM`/`EXTEND` may change size, and only from the end).
- Must be **initialized** with a constructor, like nested tables.
- **Retains order** — unlike nested tables, a VARRAY **preserves element order** when stored in and retrieved from a database column. This is the #1 differentiator tested between VARRAY and Nested Table.
- Can be a database column type; storage is inline in the row if small, or as a **LOB (BLOB)** if it exceeds ~4000 bytes total.

### 2.4 Choosing the right collection — decision table

| Requirement | Use |
|---|---|
| Need a hash-map / string keys / sparse lookup, purely in PL/SQL | Associative Array |
| Need to store the collection as a table column, order doesn't matter, size is unbounded/unknown | Nested Table |
| Need to store the collection as a table column, order **matters**, and there's a sensible upper bound | VARRAY |
| Passing bulk data with `BULK COLLECT`/`FORALL` in PL/SQL only | Associative Array or Nested Table (either works; assoc array is most common) |
| Need `TABLE()`/`CAST()` in SQL | Nested Table or VARRAY (**not** Associative Array, with the narrow 12c package-level exception) |

### 2.5 Common Mistakes (Section 2)
- ❌ Forgetting the constructor for nested tables/VARRAYs → `ORA-06531 Reference to uninitialized collection`.
- ❌ Assuming a nested table preserves insertion order on retrieval from a table — it does **not** guarantee it (VARRAY does).
- ❌ Trying to `DELETE` an element from a VARRAY — not allowed; only `TRIM`/`EXTEND`.
- ❌ Believing associative arrays can be a table column type — they cannot.
- ❌ Confusing `EXTEND` (adds capacity/elements) with initializing — `EXTEND` on an **atomically null** collection still raises an error; you must initialize first.
- ❌ Forgetting that a `FOR i IN 1..v.COUNT LOOP` loop breaks on a **sparse** nested table (after `DELETE`) because `COUNT` ≠ highest index, and some indexes may not `EXIST`. Use `FIRST`/`NEXT`/`LAST` instead (see Section 3).


---

## 3. Iterating and Storing Collections in Tables

### 3.1 Safe iteration patterns

**Pattern A — Dense collection, simple counting loop:**
```sql
FOR i IN 1 .. v_names.COUNT LOOP
  DBMS_OUTPUT.PUT_LINE(v_names(i));
END LOOP;
```
Safe **only** when you are certain the collection is dense and starts at 1 (e.g., freshly filled by `BULK COLLECT`, or a VARRAY, which is always dense).

**Pattern B — Possibly sparse collection (the exam-safe, general-purpose pattern):**
```sql
DECLARE
  idx PLS_INTEGER := v_names.FIRST;
BEGIN
  WHILE idx IS NOT NULL LOOP
    DBMS_OUTPUT.PUT_LINE(v_names(idx));
    idx := v_names.NEXT(idx);
  END LOOP;
END;
```
This is the pattern to reach for whenever `DELETE(n)` might have been used, or the collection is an associative array indexed sparsely (e.g., by employee ID, by VARCHAR2 key, etc.). It also works flawlessly for dense collections, so **when in doubt, use Pattern B.**

**Pattern C — Reverse iteration:**
```sql
DECLARE
  idx PLS_INTEGER := v_names.LAST;
BEGIN
  WHILE idx IS NOT NULL LOOP
    DBMS_OUTPUT.PUT_LINE(v_names(idx));
    idx := v_names.PRIOR(idx);
  END LOOP;
END;
```

> ⚠️ **Exam trap:** `FOR i IN 1..v.COUNT LOOP ... v(i) ...` on a sparse nested table can raise `ORA-01403: no data found` (for associative arrays with non-contiguous keys) because element `i` might not `EXIST`. Always check `EXISTS(i)` inside such a loop, or switch to Pattern B.

### 3.2 Storing collections as table columns

To persist a collection in the database, the **collection type itself must be a schema-level (SQL) type** — created with `CREATE TYPE`, not declared inside a PL/SQL block/package.

**Nested table column:**
```sql
CREATE TYPE t_skill_nt AS TABLE OF VARCHAR2(30);
/

CREATE TABLE employees_skills (
  emp_id  NUMBER PRIMARY KEY,
  skills  t_skill_nt
)
NESTED TABLE skills STORE AS skills_storage_tab;  -- mandatory storage clause
```
- The `NESTED TABLE ... STORE AS <storage_table>` clause is **mandatory** for a nested table column — Oracle physically stores nested table rows in a separate storage table linked back by a hidden key.
- You may add `RETURN LOCATOR` to fetch a **locator** (pointer) instead of the whole collection (see Section 5) — improves performance when you don't need the full collection every time.

**VARRAY column** (no separate storage table needed by default):
```sql
CREATE TYPE t_phones_va AS VARRAY(5) OF VARCHAR2(15);
/

CREATE TABLE contacts (
  contact_id NUMBER PRIMARY KEY,
  phones     t_phones_va
);
```
- Small VARRAYs store **inline** with the row; if the VARRAY's maximum possible size exceeds ~4000 bytes, Oracle automatically stores it **out-of-line as a BLOB** (`STORE AS LOB`).

### 3.3 Inserting/Updating a collection column

```sql
INSERT INTO employees_skills VALUES (101, t_skill_nt('SQL','PL/SQL','Java'));

UPDATE employees_skills
SET skills = t_skill_nt('SQL','PL/SQL','Java','Python')
WHERE emp_id = 101;
```
You must supply a **whole new collection** via a constructor for `UPDATE` on the collection column as a scalar — you cannot directly `UPDATE ... SET skills(2) = 'X'` in plain SQL. To modify a *single element* you either (a) fetch into a PL/SQL variable, modify it, and write it back, or (b) use the `TABLE()` operator to treat the nested table as a set of rows and `UPDATE`/`INSERT`/`DELETE` against it directly (see Section 4).

### 3.4 Reading a collection column back into PL/SQL

```sql
DECLARE
  v_skills t_skill_nt;
BEGIN
  SELECT skills INTO v_skills FROM employees_skills WHERE emp_id = 101;
  FOR i IN 1 .. v_skills.COUNT LOOP
    DBMS_OUTPUT.PUT_LINE(v_skills(i));
  END LOOP;
END;
/
```

### 3.5 Common Mistakes (Section 3)
- ❌ Forgetting `NESTED TABLE col STORE AS ...` when creating a table with a nested table column → Oracle raises an error at `CREATE TABLE` time; it is **not optional**.
- ❌ Trying `UPDATE tab SET nt_col(1) = 'x'` directly — invalid syntax; must replace the whole collection or use `TABLE()`.
- ❌ Assuming order is preserved for a nested table read back from the DB — write code that doesn't depend on it (sort explicitly if order matters, or use VARRAY).
- ❌ Iterating a possibly-sparse collection using `1..COUNT` and indexing directly — leads to `NO_DATA_FOUND`/subscript errors.


---

## 4. The TABLE and CAST Operators

### 4.1 The `TABLE()` operator — "unpack a collection into rows"

`TABLE()` converts a collection expression into something you can `SELECT ... FROM` — i.e., it lets SQL treat each *element* of the collection as a *row*.

```sql
DECLARE
  v_skills t_skill_nt := t_skill_nt('SQL','PL/SQL','Java');
BEGIN
  FOR rec IN (SELECT COLUMN_VALUE AS skill FROM TABLE(v_skills)) LOOP
    DBMS_OUTPUT.PUT_LINE(rec.skill);
  END LOOP;
END;
/
```
- For a collection of scalars, the unpacked "table" has one pseudo-column: **`COLUMN_VALUE`**.
- For a collection of `%ROWTYPE`/object types, the unpacked table exposes each **attribute as a real column name**.
- Querying a stored nested table column directly (a very common real-world/exam pattern):
```sql
SELECT e.emp_id, s.COLUMN_VALUE AS skill
FROM employees_skills e, TABLE(e.skills) s;
```
- You can also `INSERT INTO TABLE(...)`, `UPDATE TABLE(...)`, and `DELETE FROM TABLE(...)` to manipulate individual elements of a stored nested table **without** rewriting the whole collection:
```sql
INSERT INTO TABLE (SELECT skills FROM employees_skills WHERE emp_id = 101) VALUES ('Python');

UPDATE TABLE (SELECT skills FROM employees_skills WHERE emp_id = 101) s
SET VALUE(s) = 'PLSQL-Advanced'
WHERE VALUE(s) = 'PL/SQL';

DELETE FROM TABLE (SELECT skills FROM employees_skills WHERE emp_id = 101)
WHERE COLUMN_VALUE = 'Java';
```
> Note `VALUE(alias)` — used to reference the whole row/element when it is an object type, analogous to `COLUMN_VALUE` for scalars.

### 4.2 The `CAST()` operator — "convert between collection types"

`CAST()` converts an expression of one (collection) type to another **compatible** collection type. It's most often paired with `TABLE()` when you have a **PL/SQL-only collection expression built from a `MULTISET`/subquery** that needs an explicit target SQL type:

```sql
SELECT *
FROM TABLE(
       CAST(
         MULTISET(SELECT department_id FROM departments WHERE location_id = 1700)
         AS t_id_nt
       )
     );
```
- `CAST( collection_expr AS sql_collection_type )` — the target type **must be a schema-level type**.
- Also commonly used to cast a `NULL` to a specific collection type: `CAST(NULL AS t_skill_nt)`.
- Also used to convert between two structurally compatible collection types, e.g., VARRAY ↔ Nested Table of the same element type.

### 4.3 Local (PL/SQL-declared) collection types and SQL — the 12c relaxation

Historically (pre-12c), only **schema-level** collection types could be used inside `TABLE()`. Oracle Database **12c** introduced the ability to use `TABLE()` on a collection whose type is declared **locally in a PL/SQL package/subprogram**, but **only within the *same* PL/SQL scope/statement it's declared in** (a native, non-dynamic SQL statement embedded in that same PL/SQL unit) — you still cannot pass a locally-typed collection into an arbitrary external SQL*Plus query. For exam purposes, the safe, universally true rule is:

> **To use a collection type with `TABLE()`/`CAST()` from ordinary SQL (SQL*Plus, other units, views), the type must be created at schema level with `CREATE TYPE`.**

### 4.4 Common Mistakes (Section 4)
- ❌ Using `TABLE()` on an **associative array** — not supported at all (schema-level type required).
- ❌ Forgetting `COLUMN_VALUE` when selecting scalar elements via `TABLE()` — `SELECT skill FROM TABLE(v_skills)` fails because there is no column literally named `skill`.
- ❌ Omitting `CAST` when the collection instance can't be resolved to a known SQL type by the optimizer (typical with `MULTISET(subquery)`), producing `ORA-00932: inconsistent datatypes`.
- ❌ Trying to `TABLE()` a PL/SQL-local type from a different scope/unit — only works within strict 12c same-unit rules; default to schema-level types for anything crossing unit boundaries.
- ❌ Forgetting the correlation quirk: `TABLE(e.skills)` for correlated nested-table columns needs the enclosing table (`employees_skills e`) to be listed **before** it, comma-joined, not sub-queried independently.


---

## 5. The UTL_COLL Package (Collection Locators)

> 📌 **Important naming clarification (a frequent exam/interview mix-up):** There is **no Oracle-supplied package literally named `DBMS_COLLECTION_UTL`**. The real, documented package for collection-locator utilities is **`UTL_COLL`**. If your syllabus/question paper says "DBMS_COLLECTION_UTL," it almost certainly means `UTL_COLL`. Always answer with the correct name, `UTL_COLL`, and mention this common confusion if the question is ambiguous.

### 5.1 What is a collection locator?
When a nested table column is declared with **`RETURN LOCATOR`** in its storage clause, a `SELECT` against that column does **not** fetch the entire collection into memory — instead it returns a lightweight **locator** (a pointer/handle to the collection's storage). The actual elements are fetched **lazily**, only when accessed (e.g., inside a cursor `FOR` loop with `TABLE()`).

```sql
CREATE TABLE phone_book1 OF phone_book_t
  NESTED TABLE ph STORE AS nt_ph_1 RETURN LOCATOR;
```
- **Benefit:** Avoids pulling potentially large nested tables into memory when you only need to test existence, pass them along, or process them lazily — reduces memory footprint for big/rarely-touched collections.
- **Default** (no `RETURN LOCATOR` clause) → `RETURN VALUE`, meaning the **entire collection is materialized** every time the column is fetched.

### 5.2 `UTL_COLL.IS_LOCATOR`

The package has exactly **one function**:

```sql
UTL_COLL.IS_LOCATOR(coln IN <collection>) RETURN BOOLEAN;
```
- Returns `TRUE` if the collection instance passed in is a **locator** (backed by `RETURN LOCATOR` storage) rather than a fully materialized value.
- Typical use: diagnostics/defensive coding when a function might receive either kind of nested-table instance and needs to branch its logic.

```sql
DECLARE
  plist  t_skill_nt;
  plist1 t_skill_nt;
BEGIN
  SELECT skills INTO plist  FROM employees_skills  WHERE emp_id = 101;     -- RETURN VALUE table
  SELECT ph     INTO plist1 FROM phone_book1        WHERE pno   = 1;       -- RETURN LOCATOR table

  IF UTL_COLL.IS_LOCATOR(plist)  THEN DBMS_OUTPUT.PUT_LINE('plist is a locator');
  ELSE DBMS_OUTPUT.PUT_LINE('plist is NOT a locator'); END IF;

  IF UTL_COLL.IS_LOCATOR(plist1) THEN DBMS_OUTPUT.PUT_LINE('plist1 is a locator');
  ELSE DBMS_OUTPUT.PUT_LINE('plist1 is NOT a locator'); END IF;
END;
/
```

### 5.3 Common Mistakes (Section 5)
- ❌ Writing "DBMS_COLLECTION_UTL" as the package name on an exam — the correct name is **`UTL_COLL`**.
- ❌ Thinking `RETURN LOCATOR` changes how you write PL/SQL code to access elements — it doesn't; it only changes the **fetch/materialization strategy** under the hood.
- ❌ Assuming `UTL_COLL` has CRUD-style methods for manipulating collections — it has **only** `IS_LOCATOR`; all manipulation still uses standard collection methods (`EXTEND`, `DELETE`, etc.) or `TABLE()`.
- ❌ Confusing "locator" (a nested-table storage/fetch strategy) with a LOB locator (BLOB/CLOB) — they are conceptually similar (pointer vs. materialized value) but are different Oracle features.

---

## 6. Working with JSON Data in PL/SQL

Two complementary toolsets exist, and exam questions often test whether you know **when to use which**:

1. **SQL/JSON functions** (`JSON_TABLE`, `JSON_VALUE`, `JSON_QUERY`, `JSON_OBJECT`, `JSON_ARRAY`, `JSON_ARRAYAGG`, `JSON_OBJECTAGG`) — used **inside SQL statements**.
2. **PL/SQL JSON object types** (`JSON_OBJECT_T`, `JSON_ARRAY_T`, `JSON_ELEMENT_T`, `JSON_SCALAR_T`, `JSON_KEY_LIST`) — used for **programmatic, in-memory** manipulation inside PL/SQL blocks.

### 6.1 JSON → Collection: using `JSON_TABLE` + `BULK COLLECT`

`JSON_TABLE` projects JSON data into relational rows/columns, which you can then bulk-fetch into a PL/SQL collection — this is the most exam-relevant JSON+collections pattern:

```sql
DECLARE
  TYPE t_emp_rec IS RECORD (emp_name VARCHAR2(50), salary NUMBER);
  TYPE t_emp_tab IS TABLE OF t_emp_rec;
  v_emps t_emp_tab;

  v_json CLOB := '[{"name":"Asha","sal":50000},{"name":"Bala","sal":60000}]';
BEGIN
  SELECT name, sal
  BULK COLLECT INTO v_emps
  FROM JSON_TABLE(v_json, '$[*]'
         COLUMNS (name VARCHAR2(50) PATH '$.name',
                  sal  NUMBER       PATH '$.sal'));

  FOR i IN 1 .. v_emps.COUNT LOOP
    DBMS_OUTPUT.PUT_LINE(v_emps(i).emp_name || ' - ' || v_emps(i).salary);
  END LOOP;
END;
/
```
> ⚠️ Note the mismatch trap above: the `BULK COLLECT INTO` record's field names (`emp_name`) do **not** need to match the `JSON_TABLE` column aliases (`name`) — but the **order and count** of expressions in the `SELECT` list must match the record's fields exactly, or you get `ORA-00947`/type-mismatch errors.

### 6.2 Collection → JSON: `JSON_ARRAY`, `JSON_ARRAYAGG`, `JSON_OBJECT`

`JSON_ARRAY` (SQL function) can take a **VARRAY or Nested Table instance directly** and turn it into a JSON array:
```sql
SELECT JSON_ARRAY(t_skill_nt('SQL','PL/SQL','Java')) FROM dual;
-- ["SQL","PL/SQL","Java"]
```
To turn a **query result set** into a JSON array (the far more common real-world need), use `JSON_ARRAYAGG` combined with `JSON_OBJECT` per row:
```sql
SELECT JSON_ARRAYAGG(JSON_OBJECT('name' VALUE first_name, 'sal' VALUE salary))
FROM employees
WHERE department_id = 90;
```

### 6.3 PL/SQL object types for in-memory JSON manipulation

```sql
DECLARE
  po_obj  JSON_OBJECT_T;
  li_arr  JSON_ARRAY_T;
  li_item JSON_ELEMENT_T;
  v_total NUMBER := 0;
BEGIN
  po_obj := JSON_OBJECT_T.parse('{"items":[{"qty":2,"price":10},{"qty":3,"price":5}]}');
  li_arr := po_obj.get_Array('items');

  FOR i IN 0 .. li_arr.get_size - 1 LOOP          -- ⚠️ JSON_ARRAY_T is 0-indexed!
    DECLARE
      li_obj JSON_OBJECT_T := TREAT(li_arr.get(i) AS JSON_OBJECT_T);
    BEGIN
      v_total := v_total + li_obj.get_Number('qty') * li_obj.get_Number('price');
    END;
  END LOOP;

  DBMS_OUTPUT.PUT_LINE('Total: ' || v_total);
END;
/
```
Key members to remember:

| Type | Key methods |
|---|---|
| `JSON_OBJECT_T` | `parse()`, `get(key)`, `get_String/Number/Date/Boolean/Object/Array(key)`, `put(key, val)`, `get_keys` (returns `JSON_KEY_LIST`, a VARRAY of VARCHAR2(4000)), `to_string`/`to_clob` |
| `JSON_ARRAY_T` | `parse()`, `get(i)`, `get_size`, `append(val)`, `to_string` |
| `JSON_ELEMENT_T` | Base/generic type; `is_Object`, `is_Array`, `is_Scalar`, `is_String`, `is_Number`, `is_Boolean`; use `TREAT(... AS JSON_OBJECT_T)` / `TREAT(... AS JSON_ARRAY_T)` to narrow it |
| `JSON_KEY_LIST` | Just a `VARRAY(...) OF VARCHAR2(4000)` — behaves like any other VARRAY (`COUNT`, indexing, etc.) |

### 6.4 Common Mistakes (Section 6)
- ❌ Treating `JSON_ARRAY_T` as **1-indexed** — its `get(i)` method (and internal iteration) is **0-indexed** (`0 .. get_size-1`), unlike native PL/SQL collections which are conventionally 1-indexed. This is a classic exam gotcha.
- ❌ Forgetting `TREAT(... AS JSON_OBJECT_T)` (or `JSON_ARRAY_T`) when a `JSON_ELEMENT_T` is known to actually be an object/array — calling object/array-only methods directly on a generic `JSON_ELEMENT_T` variable fails to compile/raises an error.
- ❌ Mixing up `JSON_QUERY` (returns JSON fragment — object/array, as text) with `JSON_VALUE` (returns a single **scalar**) in SQL — `JSON_VALUE` on a JSON array/object target raises `ORA-40462`/error by default.
- ❌ Forgetting that `JSON_TABLE`'s `COLUMNS` list must line up positionally with `BULK COLLECT INTO` target fields.
- ❌ Assuming plain `VARCHAR2` can hold arbitrarily large JSON — use `CLOB` for JSON documents beyond ~32K/4K depending on context.


---

## 7. Exception Handling in Collections — SAVE EXCEPTIONS

### 7.1 The problem
By default, a `FORALL` statement **stops at the first error** — if row 50 out of 1,000 in a bulk DML fails (e.g., a constraint violation), rows 51–1,000 are **never attempted**, and only row 50's error is raised (as a normal exception).

### 7.2 The fix: `SAVE EXCEPTIONS`
Adding `SAVE EXCEPTIONS` tells Oracle: *"keep going through the entire bind array, log every failure, and raise one consolidated exception (`ORA-24381`) at the very end."*

```sql
DECLARE
  TYPE t_id_tab IS TABLE OF NUMBER;
  v_ids t_id_tab := t_id_tab(1, 2, 999, 4, 999, 6);  -- assume 999 violates a FK/constraint twice
  bulk_errors EXCEPTION;
  PRAGMA EXCEPTION_INIT(bulk_errors, -24381);
BEGIN
  FORALL i IN v_ids.FIRST .. v_ids.LAST SAVE EXCEPTIONS
    DELETE FROM orders WHERE customer_id = v_ids(i);

EXCEPTION
  WHEN bulk_errors THEN
    FOR j IN 1 .. SQL%BULK_EXCEPTIONS.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE(
        'Error #' || j ||
        ' at iteration ' || SQL%BULK_EXCEPTIONS(j).ERROR_INDEX ||
        ': ' || SQLERRM(-SQL%BULK_EXCEPTIONS(j).ERROR_CODE)   -- note the leading minus sign!
      );
    END LOOP;
END;
/
```

### 7.3 Key facts to memorize

| Fact | Detail |
|---|---|
| Implicit cursor attribute for logged errors | `SQL%BULK_EXCEPTIONS` — a collection of records with `.ERROR_INDEX` and `.ERROR_CODE` |
| `ERROR_INDEX` | The **iteration number** (position in the driving collection / bind array) that failed — **not** a row ID |
| `ERROR_CODE` | A **positive** Oracle error number; must be **negated** (`-ERROR_CODE`) before passing to `SQLERRM` to get the readable message |
| Overall exception raised after the loop completes | `ORA-24381` — you must trap it (typically via `PRAGMA EXCEPTION_INIT`) to inspect `SQL%BULK_EXCEPTIONS`, otherwise the block still fails/propagates |
| Successful iterations | **Are still committed/executed** — `SAVE EXCEPTIONS` does *not* roll back the whole FORALL; only failed iterations are skipped, everything else proceeds normally (still subject to your own explicit `COMMIT`/`ROLLBACK`) |
| Max number of logged errors | Up to `SQL%BULK_EXCEPTIONS.COUNT`; historically capped around a few hundred/thousand internal limit, essentially a non-issue for exam purposes but sometimes asked as trivia — the important behavioral fact is "it keeps going and reports them all," not the numeric cap |
| Without `SAVE EXCEPTIONS` | `FORALL` behaves like a normal DML — **stops at first error**, raises the normal Oracle exception, rows before the failure ARE still applied (up to that point), rows after are never attempted |

### 7.4 Common Mistakes (Section 7)
- ❌ Forgetting to negate `ERROR_CODE` when calling `SQLERRM(-SQL%BULK_EXCEPTIONS(j).ERROR_CODE)` — passing it positive returns the wrong/generic message.
- ❌ Treating `ERROR_INDEX` as a primary key/row identifier from the table — it is the **position in the bind collection**, so you must map it back via `v_ids(SQL%BULK_EXCEPTIONS(j).ERROR_INDEX)` to find which actual value failed.
- ❌ Not trapping `ORA-24381` at all — without a handler, the FORALL block still raises an unhandled exception even though `SAVE EXCEPTIONS` collected the details; the details are lost/inaccessible from outside a handler that checks `SQL%BULK_EXCEPTIONS` **before** the exception scope ends.
- ❌ Believing `SAVE EXCEPTIONS` implies automatic commit or automatic rollback — it does **neither**; that's entirely up to your transaction control statements.
- ❌ Forgetting `PRAGMA EXCEPTION_INIT` to give `-24381` a name — you *can* catch it with `WHEN OTHERS`, but naming it is the cleaner, exam-expected style and lets you distinguish it from unrelated errors.

---

## 8. BULK COLLECT and FORALL

These two features exist to solve the same underlying performance problem: **minimizing context switches** between the PL/SQL engine and the SQL engine.

### 8.1 The context-switch problem
A row-by-row loop like:
```sql
FOR rec IN (SELECT * FROM employees) LOOP
  UPDATE salaries SET amt = amt * 1.1 WHERE emp_id = rec.emp_id;
END LOOP;
```
performs **one context switch per row** for the `UPDATE` (and, absent an explicit cursor, potentially per `SELECT` fetch too) — extremely slow for large data volumes. `BULK COLLECT` batches the **SELECT** direction; `FORALL` batches the **DML** direction.

### 8.2 BULK COLLECT — bulk fetch (SQL → PL/SQL, one round trip)

```sql
DECLARE
  TYPE t_names_tab IS TABLE OF employees.last_name%TYPE;
  v_names t_names_tab;
BEGIN
  SELECT last_name
  BULK COLLECT INTO v_names
  FROM employees
  WHERE department_id = 90;
END;
/
```
- Works with `SELECT ... BULK COLLECT INTO`, `FETCH cursor BULK COLLECT INTO`, and `RETURNING ... BULK COLLECT INTO` (for DML).
- **`LIMIT` clause** — caps how many rows are fetched per batch, essential for very large result sets to avoid exhausting PGA memory:
```sql
DECLARE
  CURSOR c_emps IS SELECT * FROM employees;
  TYPE t_emp_tab IS TABLE OF employees%ROWTYPE;
  v_emps t_emp_tab;
BEGIN
  OPEN c_emps;
  LOOP
    FETCH c_emps BULK COLLECT INTO v_emps LIMIT 500;
    EXIT WHEN v_emps.COUNT = 0;
    -- process v_emps(1..v_emps.COUNT)
  END LOOP;
  CLOSE c_emps;
END;
/
```
- **No `NO_DATA_FOUND` exception** if zero rows match — unlike a scalar `SELECT INTO`. You must check `collection.COUNT = 0` instead.
- Target collection is **automatically initialized and reset** (implicitly does the equivalent of `DELETE` first) by `BULK COLLECT` — you don't need to pre-initialize it.
- Multi-column bulk fetch into multiple collections at once is allowed:
```sql
SELECT last_name, salary BULK COLLECT INTO v_names, v_sals FROM employees;
```

### 8.3 FORALL — bulk DML (PL/SQL → SQL, one round trip)

```sql
DECLARE
  TYPE t_id_tab IS TABLE OF employees.employee_id%TYPE;
  v_ids t_id_tab := t_id_tab(100,101,102,103);
BEGIN
  FORALL i IN v_ids.FIRST .. v_ids.LAST
    UPDATE employees SET salary = salary * 1.1 WHERE employee_id = v_ids(i);

  DBMS_OUTPUT.PUT_LINE(SQL%ROWCOUNT);        -- total rows affected across all iterations
END;
/
```
- `FORALL` is **not a general-purpose loop** — the body must be **exactly one** DML statement (`INSERT`, `UPDATE`, `DELETE`, or `MERGE`), referencing the bind collection with **simple, unmodified subscripts** like `v_ids(i)`.
- The bounds `v_ids.FIRST .. v_ids.LAST` handle sparse collections incorrectly if there are gaps — use the **`INDICES OF`** clause for a sparse driving collection, or **`VALUES OF`** to drive off a secondary index collection:
```sql
FORALL i IN INDICES OF v_ids
  UPDATE employees SET salary = salary * 1.1 WHERE employee_id = v_ids(i);
```
- `SQL%ROWCOUNT` after `FORALL` = **total** rows affected by **all** iterations combined (not per-iteration). For a per-iteration breakdown, use the (less commonly tested) `SQL%BULK_ROWCOUNT(i)` collection.
- `RETURNING ... BULK COLLECT INTO` can be combined with `FORALL` to capture output values (e.g., generated IDs) from every iteration in one shot:
```sql
FORALL i IN v_names.FIRST .. v_names.LAST
  INSERT INTO departments (department_id, department_name)
  VALUES (dept_seq.NEXTVAL, v_names(i))
  RETURNING department_id BULK COLLECT INTO v_new_ids;
```

### 8.4 BULK COLLECT vs FORALL — side-by-side

| | BULK COLLECT | FORALL |
|---|---|---|
| Direction | SQL → PL/SQL (fetch) | PL/SQL → SQL (DML) |
| Used with | `SELECT`, `FETCH`, `RETURNING` | `INSERT`/`UPDATE`/`DELETE`/`MERGE` |
| Loop construct? | No (it's a clause on SELECT/FETCH) | It *is* an implicit loop construct, but only one DML statement is allowed |
| Handles sparse driving collection? | N/A (produces a fresh, dense collection) | Needs `INDICES OF` / `VALUES OF` for sparse input |
| Error behavior | Normal (no special option) | `SAVE EXCEPTIONS` available |
| Row-count attribute | N/A directly | `SQL%ROWCOUNT` (total), `SQL%BULK_ROWCOUNT(i)` (per iteration) |

### 8.5 Common Mistakes (Section 8)
- ❌ Using `FOR i IN v.FIRST..v.LAST LOOP ... FORALL ...` — **wrong**: `FORALL` is *itself* the loop; wrapping it in another loop or putting more than one DML statement inside it is a compile error.
- ❌ Forgetting `LIMIT` on `BULK COLLECT` for huge tables — can consume excessive PGA memory / cause `ORA-04030`.
- ❌ Expecting `NO_DATA_FOUND` from a zero-row `BULK COLLECT` — it silently returns an empty (zero-`COUNT`), **not** atomically-null, collection.
- ❌ Using `v.FIRST..v.LAST` in `FORALL` on a **sparse** collection — silently skips or errors on missing indices; must use `INDICES OF`.
- ❌ Assuming `SQL%ROWCOUNT` after `FORALL` gives a per-row breakdown — it's the **sum total**; use `SQL%BULK_ROWCOUNT(i)` for per-iteration counts.
- ❌ Modifying the subscript expression inside `FORALL`'s DML (e.g., `v_ids(i+1)`) — the bind expression must be a simple `collection(index_variable)` reference for the bulk bind optimization to apply (complex expressions can silently fall back to row-by-row execution or raise errors depending on version/expression).


---

## 9. Pipelined Table Functions

### 9.1 What is a pipelined function?
A **pipelined function** is a table function that returns rows **incrementally**, one at a time (or in small batches), using the `PIPE ROW` statement — instead of building the *entire* result collection in memory first and returning it at the end. The caller can start consuming rows **before the function finishes producing all of them**.

```sql
CREATE TYPE t_num_row  AS OBJECT (n NUMBER);
/
CREATE TYPE t_num_tab  AS TABLE OF t_num_row;
/

CREATE OR REPLACE FUNCTION squares_upto(p_max IN NUMBER)
  RETURN t_num_tab PIPELINED
IS
BEGIN
  FOR i IN 1 .. p_max LOOP
    PIPE ROW (t_num_row(i * i));
  END LOOP;
  RETURN;   -- mandatory, but returns NO value (bare RETURN)
END;
/

SELECT * FROM TABLE(squares_upto(5));
-- 1, 4, 9, 16, 25
```

### 9.2 Key syntax rules
- The function must be declared `RETURN <collection_type> PIPELINED`.
- Inside the body, you emit rows with `PIPE ROW(<element_of_collection_type>)`.
- The function still needs a `RETURN;` statement (bare, with **no expression**) to end execution — this is **not** the same as returning a value; it just terminates the pipelining.
- `<collection_type>` must be a **schema-level nested table (or VARRAY) type**, so callers can wrap the call in `TABLE()`.

### 9.3 Why use pipelined functions
- **Memory efficiency:** rows stream out as produced; the whole result set need not fit in PGA/session memory at once.
- **Can begin returning rows to the consumer immediately**, improving perceived latency for large result sets (rows can be consumed as soon as produced, rather than waiting for the whole function to finish).
- Can be used to **transform data on the fly** inside a `SELECT` — e.g., wrap complex procedural ETL logic and expose it as if it were a table.
- Supports **parallel execution** when combined with the `PARALLEL_ENABLE` clause and a partitioning strategy on a REF CURSOR input — an advanced, sometimes-tested feature for parallelizing custom transformation logic across slaves.

### 9.4 PARALLEL_ENABLE and streaming input (advanced but examinable)
```sql
CREATE OR REPLACE FUNCTION transform_emps(p_cur IN SYS_REFCURSOR)
  RETURN t_emp_tab PIPELINED
  PARALLEL_ENABLE (PARTITION p_cur BY ANY)
IS
  v_emp employees%ROWTYPE;
BEGIN
  LOOP
    FETCH p_cur INTO v_emp;
    EXIT WHEN p_cur%NOTFOUND;
    PIPE ROW (t_emp_row(v_emp.employee_id, UPPER(v_emp.last_name)));
  END LOOP;
  CLOSE p_cur;
  RETURN;
END;
/
```
- `PARTITION p_cur BY ANY` — rows from the input cursor can be arbitrarily distributed to parallel query slaves (no ordering guarantee needed).
- `PARTITION p_cur BY HASH(col)` / `BY RANGE(col)` — used when related rows must be processed by the **same** slave.

### 9.5 Common Mistakes (Section 9)
- ❌ Forgetting the bare `RETURN;` statement at the end — a pipelined function **must** have one, even though it returns no value there (the actual output already went out via `PIPE ROW`).
- ❌ Writing `RETURN v_collection;` (a full collection) inside a pipelined function body — illegal; you can only `PIPE ROW` individual elements, plus the final **bare** `RETURN;`.
- ❌ Declaring the return type as a **PL/SQL-only** (locally declared) collection type — must be schema-level so `TABLE()` can consume it externally.
- ❌ Assuming a pipelined function can be used in a scalar context (`v := squares_upto(5);`) — it must be queried via `SELECT ... FROM TABLE(squares_upto(5))`.
- ❌ Confusing pipelined functions with plain table functions (below) — pipelining is specifically about **row-at-a-time streaming**, not just "returns a collection queryable via TABLE()."

---

## 10. Table Functions (General)

### 10.1 Definition
A **table function** (broader category than "pipelined") is *any* PL/SQL function that **returns a collection type**, allowing it to be queried in the `FROM` clause via `TABLE()`. Pipelined functions are a special, streaming **subset** of table functions.

```sql
CREATE OR REPLACE FUNCTION get_top_earners(p_dept IN NUMBER)
  RETURN t_emp_tab                       -- schema-level nested table type
IS
  v_emps t_emp_tab;
BEGIN
  SELECT t_emp_row(employee_id, last_name, salary)
  BULK COLLECT INTO v_emps
  FROM employees
  WHERE department_id = p_dept
  ORDER BY salary DESC;

  RETURN v_emps;                          -- entire collection built, THEN returned
END;
/

SELECT * FROM TABLE(get_top_earners(90));
```

### 10.2 Non-pipelined vs pipelined table function

| | Non-pipelined table function | Pipelined table function |
|---|---|---|
| Output timing | Entire collection built in memory, returned **all at once** | Rows streamed out **as produced** via `PIPE ROW` |
| Memory footprint | Higher — whole result held before returning | Lower — no need to materialize the full set |
| Syntax keyword | Just `RETURN <type>` | `RETURN <type> PIPELINED` |
| Statement to emit rows | `RETURN v_collection;` | `PIPE ROW(...)` repeatedly + bare `RETURN;` |
| Can start consuming before function finishes? | No | Yes |
| Typical use case | Small/medium result sets, simpler logic | Large result sets, ETL-style row-by-row transformation, parallel processing |

### 10.3 Using table functions to "SQL-ify" procedural logic
The core value proposition (heavily examined conceptually): table functions let you **embed procedural PL/SQL logic inside a declarative SQL statement**, e.g., joining the output of a table function to a real table:
```sql
SELECT e.department_id, t.n
FROM departments e, TABLE(squares_upto(e.department_id)) t;
```
This is also the mechanism behind chaining multiple transformation steps into a single SQL pipeline (a "pipeline" of table functions, each consuming the previous one's output), commonly used in ETL.

### 10.4 Passing a cursor as input — the "table function chaining" pattern
```sql
CREATE OR REPLACE FUNCTION uppercase_names(p_cur IN SYS_REFCURSOR)
  RETURN t_names_nt PIPELINED
IS
  v_name VARCHAR2(100);
BEGIN
  LOOP
    FETCH p_cur INTO v_name;
    EXIT WHEN p_cur%NOTFOUND;
    PIPE ROW (UPPER(v_name));
  END LOOP;
  CLOSE p_cur;
  RETURN;
END;
/

SELECT * FROM TABLE(uppercase_names(CURSOR(SELECT last_name FROM employees)));
```
This "table function chaining" pattern (feeding a `CURSOR()` expression into one table function, and its output into the next) is a classic way to build multi-stage SQL-based ETL pipelines without staging tables.

### 10.5 Common Mistakes (Section 10)
- ❌ Believing every function returning a collection is automatically "pipelined" — pipelining requires the explicit `PIPELINED` keyword and `PIPE ROW`.
- ❌ Forgetting the return type must still be a **schema-level** collection type, exactly as with pipelined functions.
- ❌ Using a table function inside `FROM` **without** `TABLE()` — required syntax; omitting it is a compile/parse error (`SELECT * FROM get_top_earners(90)` is invalid).
- ❌ Passing a PL/SQL collection variable (not a function call) into `TABLE()` and expecting column names other than `COLUMN_VALUE` for scalar types — same rule as Section 4.


---

## 11. Collection Performance Tuning

### 11.1 The golden rule
**Minimize context switches between the PL/SQL and SQL engines.** Almost every performance tip below is a restatement of this one principle.

### 11.2 Core techniques, ranked by typical exam emphasis

1. **Use `BULK COLLECT` instead of row-by-row `FETCH ... INTO` loops.** Converts N round-trips into 1 (or `N/LIMIT`).
2. **Use `FORALL` instead of DML inside a `FOR` loop.** Same logic, applied to writes.
3. **Always use `LIMIT` on `BULK COLLECT`** for large result sets — bounds PGA memory usage and avoids `ORA-04030`; process in batches inside a loop.
4. **Prefer Associative Arrays for pure in-memory, PL/SQL-only bulk staging** — lighter weight than nested tables (no need for constructors, schema-level type overhead) when the data never needs to touch SQL's `TABLE()`.
5. **Avoid unnecessary `EXTEND` calls one at a time in a loop** — use `EXTEND(n)` to grow by a known batch size at once rather than calling `EXTEND` (1 at a time) inside a tight loop, or rely on `BULK COLLECT`'s automatic sizing instead of manual `EXTEND`.
6. **Use `RETURN LOCATOR`** for large, infrequently-accessed nested table columns to avoid materializing them on every fetch when the caller may not need the full collection (see Section 5).
7. **Use VARRAY instead of Nested Table when order matters and size is bounded** — avoids extra application-level sorting and the separate physical storage table overhead of nested tables.
8. **Use `SAVE EXCEPTIONS` instead of shrinking batch size to 1 to "isolate" bad rows** — keeps the performance benefit of bulk DML while still surfacing every failure.
9. **Bind array size tuning:** very large `LIMIT` values (e.g., 100,000) don't necessarily perform better than a moderate value (e.g., 1,000–10,000) — memory pressure and garbage collection overhead can outweigh the reduced round-trip count; the "right" `LIMIT` is a **tuning/benchmarking decision**, not a fixed number, but the exam-safe takeaway is: *"NEVER omit LIMIT entirely on unbounded data, and don't assume bigger is always better."*
10. **Avoid `PIPE ROW`-ing single elements one-by-one when the whole batch is already known** — if you have the whole collection anyway, a non-pipelined table function that does one `BULK COLLECT`/`RETURN` can be simpler and just as fast; reach for pipelining specifically when *streaming/incremental* production and memory savings genuinely matter.
11. **Avoid needless `PL/SQL ↔ SQL` type conversions** — e.g., don't `CAST` back and forth repeatedly in a hot loop; settle on schema-level types up front if data must cross the SQL boundary.
12. **Use `NOCOPY` hint for large collection `IN OUT` parameters** — passing large collections by value (Oracle's default parameter-passing semantics for IN OUT) can be very expensive to copy; `NOCOPY` passes by reference, at the (usually acceptable) cost of losing guaranteed rollback of the parameter's value if an exception is raised mid-call.
```sql
PROCEDURE process_batch(p_data IN OUT NOCOPY t_emp_tab);
```

### 11.3 Sample "before vs after" comparison (a very common exam question format)

**Slow — row-by-row:**
```sql
FOR rec IN (SELECT employee_id FROM employees WHERE department_id = 90) LOOP
  UPDATE salaries SET amt = amt * 1.1 WHERE employee_id = rec.employee_id;
END LOOP;
```

**Fast — bulk:**
```sql
DECLARE
  TYPE t_ids IS TABLE OF employees.employee_id%TYPE;
  v_ids t_ids;
BEGIN
  SELECT employee_id BULK COLLECT INTO v_ids
  FROM employees WHERE department_id = 90;

  FORALL i IN v_ids.FIRST .. v_ids.LAST
    UPDATE salaries SET amt = amt * 1.1 WHERE employee_id = v_ids(i);
END;
/
```

### 11.4 Common Mistakes (Section 11)
- ❌ Using `BULK COLLECT` without `LIMIT` on tables of unknown/large size — the #1 tuning mistake examiners test for.
- ❌ Assuming `FORALL` alone (without `SAVE EXCEPTIONS`) is "safe" for production batch jobs where a few bad rows are expected — it will abort the whole batch at the first failure.
- ❌ Passing large collections as plain `IN OUT` parameters without `NOCOPY` in performance-sensitive recursive/repeated calls.
- ❌ Believing collections are always faster than set-based SQL — if a problem can be solved with a **single SQL statement** (a plain `UPDATE...WHERE`, a `MERGE`, etc.), that is usually faster than **any** PL/SQL collection approach; bulk collections shine when procedural, row-by-row logic is *unavoidable*, not as a universal replacement for SQL.
- ❌ Over-using pipelined functions for small, one-shot result sets where the added complexity buys no real memory/latency benefit.

---

## 12. Master Comparison Cheat-Sheet

| Feature | Associative Array | Nested Table | VARRAY |
|---|---|---|---|
| Also known as | Index-by table, PL/SQL table | — | Variable-size array |
| Indexing | `PLS_INTEGER`/`BINARY_INTEGER` **or** `VARCHAR2` | Positive integers | Positive integers |
| Dense or sparse | Either (commonly sparse) | Starts dense; can become sparse | Always dense |
| Bounded size? | No | No | Yes (`LIMIT` at declaration) |
| Needs constructor/initialization? | No (usable "empty" by default) | Yes (else atomically NULL) | Yes (else atomically NULL) |
| Supports `DELETE(n)` middle-removal | Yes | Yes | **No** |
| Supports `TRIM` | **No** | Yes | Yes |
| Supports `EXTEND` | **No** (auto-grows on assignment) | Yes | Yes (up to `LIMIT`) |
| Can be a database column type | **No** | Yes (`NESTED TABLE...STORE AS`) | Yes (inline or LOB) |
| Order preserved when stored & retrieved | N/A (not storable) | **Not guaranteed** | **Guaranteed** |
| Usable with `TABLE()`/`CAST()` | No (schema-level type required) | Yes | Yes |
| Typical use | PL/SQL-only lookups, sparse/keyed data, `BULK COLLECT` staging | Unbounded, order-agnostic DB storage; SQL set operations | Bounded, order-sensitive DB storage |
| Persistence | Session (if package-level) / block only | Session (if package-level) / block; or DB row (if column) | Session / block; or DB row (if column) |

### Quick syntax reference

```sql
-- Iterate safely
idx := coll.FIRST; WHILE idx IS NOT NULL LOOP ... idx := coll.NEXT(idx); END LOOP;

-- Bulk fetch with batching
FETCH cur BULK COLLECT INTO coll LIMIT 1000;

-- Bulk DML with full error capture
FORALL i IN INDICES OF coll SAVE EXCEPTIONS
  UPDATE ... WHERE key = coll(i);
-- then: EXCEPTION WHEN bulk_errors THEN ... SQL%BULK_EXCEPTIONS(j).ERROR_INDEX/.ERROR_CODE

-- Table + Cast
SELECT * FROM TABLE(CAST(MULTISET(SELECT ... ) AS schema_level_type));

-- Pipelined function skeleton
CREATE FUNCTION f(...) RETURN schema_level_nt PIPELINED IS
BEGIN
  PIPE ROW(...);
  RETURN;   -- bare
END;

-- Locator check
UTL_COLL.IS_LOCATOR(coll)  -- TRUE/FALSE
```


---

## 13. Practice Exercises

### Part A — Conceptual / MCQ style

**A1.** Which of the three collection types can be indexed by a `VARCHAR2` value?

**A2.** True or False: A VARRAY column, when read back from the database, is guaranteed to preserve the order in which elements were inserted.

**A3.** Which collection method removes elements from the *middle* of a nested table, and why can it not be used on a VARRAY?

**A4.** What error is raised if you attempt to reference an element of a nested table variable that was declared but never assigned a constructor?

**A5.** Name the one function contained in the `UTL_COLL` package, and explain what it tests.

**A6.** What is the correct name of the Oracle-supplied package for testing whether a nested table is a "locator," and what common misnomer might a question paper use instead?

**A7.** In `JSON_ARRAY_T`, is the `get(i)` method 0-indexed or 1-indexed? How does this differ from ordinary PL/SQL collection indexing conventions?

**A8.** What SQL error code corresponds to the consolidated exception raised at the end of a `FORALL ... SAVE EXCEPTIONS` statement that encountered failures?

**A9.** Explain, in one or two sentences, why `SQL%BULK_EXCEPTIONS(j).ERROR_CODE` must be negated before being passed to `SQLERRM`.

**A10.** What is the key syntactic difference between a plain table function and a pipelined table function?

### Part B — Code reading / "what happens" questions

**B1.** What is wrong with the following block, and what error/behavior results?
```sql
DECLARE
  TYPE t_nt IS TABLE OF VARCHAR2(10);
  v_nt t_nt;
BEGIN
  v_nt(1) := 'Hello';
END;
```

**B2.** Given:
```sql
CREATE TYPE t_va AS VARRAY(3) OF NUMBER;
DECLARE
  v t_va := t_va(1,2,3);
BEGIN
  v.EXTEND;
  v(4) := 4;
END;
```
What happens when this block runs, and why?

**B3.** What does the following `FORALL` print for `SQL%ROWCOUNT`, given that `employee_id` values 10, 20, and 30 each match exactly 2, 0, and 3 rows respectively in `salaries`?
```sql
FORALL i IN v_ids.FIRST .. v_ids.LAST
  DELETE FROM salaries WHERE employee_id = v_ids(i);
DBMS_OUTPUT.PUT_LINE(SQL%ROWCOUNT);
```

**B4.** A nested table `v_nt` has had `v_nt.DELETE(3)` called on it, and originally had elements at indices 1 through 5. What will `FOR i IN 1..v_nt.COUNT LOOP DBMS_OUTPUT.PUT_LINE(v_nt(i)); END LOOP;` do?

**B5.** Why does the following query fail, and how would you fix it?
```sql
DECLARE
  v_names t_names_nt := t_names_nt('A','B','C');
BEGIN
  FOR r IN (SELECT name FROM TABLE(v_names)) LOOP
    DBMS_OUTPUT.PUT_LINE(r.name);
  END LOOP;
END;
```

### Part C — Write the code

**C1.** Write a PL/SQL block that declares an associative array indexed by `VARCHAR2(20)`, storing `NUMBER` values, inserts three key-value pairs, and prints all of them using a safe iteration pattern.

**C2.** Create a schema-level nested table type of `VARCHAR2(50)`, use it as a column in a new table `student_courses(student_id NUMBER, courses <your_type>)` with the proper storage clause, and insert one row with 3 course names.

**C3.** Write a `FORALL ... SAVE EXCEPTIONS` block that attempts to insert 6 rows into a table with a `NOT NULL` constraint on one column, where 2 of the 6 rows would violate that constraint, and prints out which **input row numbers** failed and why.

**C4.** Write a pipelined table function `fibonacci_upto(p_n IN NUMBER) RETURN <nt_type> PIPELINED` that emits the first `p_n` Fibonacci numbers one at a time via `PIPE ROW`, and show the `SELECT` you'd use to call it.

**C5.** Given a CLOB variable containing a JSON array of objects like `[{"sku":"A1","qty":5},{"sku":"A2","qty":8}]`, write PL/SQL that uses `JSON_TABLE` and `BULK COLLECT` to load this into a collection of records with fields `sku` and `qty`, then prints the total quantity.

**C6.** Rewrite the following slow procedure using `BULK COLLECT` (with a `LIMIT` of 200) and `FORALL`:
```sql
PROCEDURE give_raise(p_dept IN NUMBER, p_pct IN NUMBER) IS
BEGIN
  FOR rec IN (SELECT employee_id FROM employees WHERE department_id = p_dept) LOOP
    UPDATE employees SET salary = salary * (1 + p_pct/100) WHERE employee_id = rec.employee_id;
  END LOOP;
END;
```

### Part D — Scenario / applied questions

**D1.** You need to store a per-customer list of up to 5 favorite product IDs, where the order the customer added them matters for a "recently favorited first" display. Which collection type should back the database column, and why? What is the alternative, and why is it inferior for this specific requirement?

**D2.** A batch job processes 2 million rows nightly with a `FORALL` bulk update. Occasionally a handful of rows (fewer than 50) fail due to a check constraint, and the business wants the job to process everything it can and email a report of failures rather than abort. Describe, step by step, the PL/SQL techniques you would combine to achieve this.

**D3.** A junior developer writes `BULK COLLECT INTO v_tab FROM huge_table;` with no `LIMIT` clause on a table with 40 million rows. Explain the risk, and rewrite their approach safely.

**D4.** Explain why an associative array cannot be the return type of a function that will be queried using `SELECT * FROM TABLE(my_function(...))`, and what change is required to make such a function work.


---

## 14. Answer Key

> Try each question yourself first — these answers are intentionally terse; refer back to the numbered sections for full explanations.

### Part A
- **A1.** Associative array only (`INDEX BY VARCHAR2(n)`).
- **A2.** **False** for Nested Tables — but this statement is about VARRAY, so: **True**. VARRAYs preserve order on storage/retrieval; Nested Tables do not guarantee it.
- **A3.** `DELETE(n)`. It cannot be used on a VARRAY because VARRAYs must always remain dense (contiguous, no gaps) — removing a middle element would create a gap, which is disallowed by definition.
- **A4.** `ORA-06531: Reference to uninitialized collection`.
- **A5.** `IS_LOCATOR(coln)` — returns `TRUE`/boolean indicating whether the given nested-table instance is a locator (lazy, `RETURN LOCATOR` storage) rather than a fully materialized collection value.
- **A6.** Correct name: **`UTL_COLL`**. Common misnomer in question papers: "DBMS_COLLECTION_UTL" (this package does not actually exist).
- **A7.** **0-indexed** (`0 .. get_size-1`). This differs from native PL/SQL collections, which are conventionally 1-indexed by developer convention (associative arrays/nested tables/varrays don't enforce 1-indexing, but idiomatic code almost always starts at 1; `JSON_ARRAY_T` explicitly starts at 0).
- **A8.** `ORA-24381`.
- **A9.** `SQL%BULK_EXCEPTIONS(j).ERROR_CODE` stores the Oracle error number as a **positive** integer, but `SQLERRM` expects a **negative** error number to look up the correct message text; passing it unnegated returns an incorrect/generic message.
- **A10.** A pipelined function is declared with the `PIPELINED` keyword and emits results incrementally via repeated `PIPE ROW(...)` calls followed by a bare `RETURN;`; a plain (non-pipelined) table function builds the entire result collection first and returns it in one `RETURN v_collection;` statement.

### Part B
- **B1.** `v_nt` was declared but never initialized with a constructor, so it is atomically NULL. Assigning `v_nt(1) := 'Hello'` raises `ORA-06531: Reference to uninitialized collection`. Fix: `v_nt := t_nt();` (or `t_nt('Hello')`) before assigning by index, or use `.EXTEND` after an empty-constructor init.
- **B2.** `v.EXTEND` grows the VARRAY to size 4 — but its declared `LIMIT` is 3, so this raises `ORA-06532: Subscript outside of limit` (the extend itself, or the subsequent `v(4) := 4` assignment, fails because the VARRAY cannot exceed its declared maximum size of 3).
- **B3.** `SQL%ROWCOUNT` = **5** (2 + 0 + 3) — it is the cumulative total across **all** iterations of the FORALL, not a per-iteration figure.
- **B4.** It will process indices 1, 2, 4, 5 successfully, but when `i = 3`, `v_nt(3)` does not `EXIST` (it was deleted), causing a subscript error at that iteration (referencing a non-existent element raises `ORA-01403`-style "no data found"/subscript error for collections) — the loop is **not** automatically skipping deleted indices just because it uses `1..COUNT`. Correct fix: use `FIRST`/`NEXT` iteration (Pattern B, Section 3.1) or guard with `IF v_nt.EXISTS(i) THEN`.
- **B5.** `TABLE(v_names)` unpacks a collection of scalar `VARCHAR2` elements, so the pseudo-column is named **`COLUMN_VALUE`**, not `name`. Fix: `SELECT COLUMN_VALUE AS name FROM TABLE(v_names)`. (Also, `t_names_nt` must be a schema-level type for `TABLE()` to work at all outside the strict 12c same-unit exception.)

### Part C — Reference solutions

**C1.**
```sql
DECLARE
  TYPE t_scores IS TABLE OF NUMBER INDEX BY VARCHAR2(20);
  v_scores t_scores;
  idx VARCHAR2(20);
BEGIN
  v_scores('Math')    := 90;
  v_scores('Science') := 85;
  v_scores('English') := 78;

  idx := v_scores.FIRST;
  WHILE idx IS NOT NULL LOOP
    DBMS_OUTPUT.PUT_LINE(idx || ' -> ' || v_scores(idx));
    idx := v_scores.NEXT(idx);
  END LOOP;
END;
/
```

**C2.**
```sql
CREATE TYPE t_course_nt AS TABLE OF VARCHAR2(50);
/

CREATE TABLE student_courses (
  student_id NUMBER,
  courses    t_course_nt
)
NESTED TABLE courses STORE AS student_courses_storage;

INSERT INTO student_courses
VALUES (1, t_course_nt('Database Systems','Operating Systems','Networks'));
```

**C3.**
```sql
DECLARE
  TYPE t_rec IS RECORD (id NUMBER, note VARCHAR2(20));
  TYPE t_tab IS TABLE OF t_rec;
  v_data t_tab := t_tab(
    t_rec(1,'ok'),   t_rec(2,'ok'),  t_rec(3,NULL),  -- NULL violates NOT NULL
    t_rec(4,'ok'),   t_rec(5,NULL),  t_rec(6,'ok')   -- NULL violates NOT NULL
  );
  bulk_errors EXCEPTION;
  PRAGMA EXCEPTION_INIT(bulk_errors, -24381);
BEGIN
  FORALL i IN v_data.FIRST .. v_data.LAST SAVE EXCEPTIONS
    INSERT INTO demo_notes (id, note) VALUES (v_data(i).id, v_data(i).note);
EXCEPTION
  WHEN bulk_errors THEN
    FOR j IN 1 .. SQL%BULK_EXCEPTIONS.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE(
        'Input row ' || SQL%BULK_EXCEPTIONS(j).ERROR_INDEX ||
        ' failed: ' || SQLERRM(-SQL%BULK_EXCEPTIONS(j).ERROR_CODE)
      );
    END LOOP;
END;
/
```

**C4.**
```sql
CREATE TYPE t_fib_tab AS TABLE OF NUMBER;
/

CREATE OR REPLACE FUNCTION fibonacci_upto(p_n IN NUMBER) RETURN t_fib_tab PIPELINED IS
  v_a NUMBER := 0;
  v_b NUMBER := 1;
  v_next NUMBER;
BEGIN
  FOR i IN 1 .. p_n LOOP
    PIPE ROW (v_a);
    v_next := v_a + v_b;
    v_a := v_b;
    v_b := v_next;
  END LOOP;
  RETURN;
END;
/

SELECT * FROM TABLE(fibonacci_upto(10));
```

**C5.**
```sql
DECLARE
  TYPE t_item IS RECORD (sku VARCHAR2(10), qty NUMBER);
  TYPE t_item_tab IS TABLE OF t_item;
  v_items t_item_tab;
  v_json  CLOB := '[{"sku":"A1","qty":5},{"sku":"A2","qty":8}]';
  v_total NUMBER := 0;
BEGIN
  SELECT sku, qty
  BULK COLLECT INTO v_items
  FROM JSON_TABLE(v_json, '$[*]'
         COLUMNS (sku VARCHAR2(10) PATH '$.sku',
                  qty NUMBER       PATH '$.qty'));

  FOR i IN 1 .. v_items.COUNT LOOP
    v_total := v_total + v_items(i).qty;
  END LOOP;

  DBMS_OUTPUT.PUT_LINE('Total qty: ' || v_total);
END;
/
```

**C6.**
```sql
PROCEDURE give_raise(p_dept IN NUMBER, p_pct IN NUMBER) IS
  TYPE t_ids IS TABLE OF employees.employee_id%TYPE;
  v_ids t_ids;
  CURSOR c_emps IS SELECT employee_id FROM employees WHERE department_id = p_dept;
BEGIN
  OPEN c_emps;
  LOOP
    FETCH c_emps BULK COLLECT INTO v_ids LIMIT 200;
    EXIT WHEN v_ids.COUNT = 0;

    FORALL i IN v_ids.FIRST .. v_ids.LAST
      UPDATE employees
      SET salary = salary * (1 + p_pct/100)
      WHERE employee_id = v_ids(i);
  END LOOP;
  CLOSE c_emps;
END;
/
```

### Part D
- **D1.** Use a **VARRAY(5)**: the requirement is order-sensitive ("recently favorited first") and has a known, small upper bound (5) — VARRAY guarantees retrieval order and is a natural fit. A Nested Table is inferior here because Oracle does **not** guarantee it preserves insertion/element order when stored and re-fetched, so the "recently favorited first" ordering could not be relied upon without extra bookkeeping (e.g., an explicit sequence/timestamp column).
- **D2.** (1) Load the driving keys with `BULK COLLECT ... LIMIT` for manageable batch sizes. (2) Run the update inside `FORALL ... SAVE EXCEPTIONS`. (3) Catch `ORA-24381` (named via `PRAGMA EXCEPTION_INIT`). (4) Loop over `SQL%BULK_EXCEPTIONS`, mapping each `ERROR_INDEX` back to the originating row/key and negating `ERROR_CODE` for `SQLERRM`. (5) Collect these into a log table or in-memory collection. (6) After the FORALL/batch loop completes, send the accumulated failure list via email (e.g., `UTL_MAIL`/`UTL_SMTP`) instead of letting the exception abort the whole job. (7) Explicitly `COMMIT` successful batches (e.g., per `LIMIT` chunk) so progress isn't lost.
- **D3.** Risk: fetching 40 million rows into a single in-memory collection with no `LIMIT` can exhaust PGA memory, potentially causing `ORA-04030` (out of process memory) or severe swapping/performance degradation, and delays any processing until the entire fetch completes. Safe rewrite: open an explicit cursor and loop with `FETCH ... BULK COLLECT INTO v_tab LIMIT <reasonable_batch_size, e.g. 1000–10000>`, processing/discarding each batch before fetching the next, exiting when `v_tab.COUNT = 0`.
- **D4.** `TABLE()` requires a **schema-level (SQL) collection type** so the SQL engine can resolve row/column metadata for the query; associative arrays are inherently **PL/SQL-only** constructs (declared with `INDEX BY`) and have no schema-level counterpart, so they cannot be the return type of a function invoked via `TABLE()`. Fix: change the function's return type to a schema-level **nested table** or **VARRAY** type (created via `CREATE TYPE`), and populate/return that instead (converting from an internal associative array to the nested table before returning, if the associative array was used as scratch storage).
