# Oracle PL/SQL — Records & Their Types
### Exam Study Guide

Covers: Introduction to Records & its Types · Table-Based Records · Custom (User-Defined) Records · RECORD vs. OBJECT Types · Cursor-Based Records

---

## 1. Introduction to Records & Their Types

### What it is
A **PL/SQL record** is a **composite variable** — a single logical unit made up of multiple related *fields*, each with its own name and data type (similar in spirit to a `struct` in C or a row of a table). Records let you treat a whole row's worth of related data as one variable instead of declaring a separate scalar variable per column.

```sql
DECLARE
  v_emp_name  VARCHAR2(50);
  v_emp_sal   NUMBER;
  v_emp_dept  NUMBER;
BEGIN
  NULL; -- three separate variables, no logical grouping
END;
```
vs.
```sql
DECLARE
  TYPE emp_rec_t IS RECORD (name VARCHAR2(50), sal NUMBER, dept NUMBER);
  v_emp emp_rec_t;  -- ONE variable holding all three related fields
BEGIN
  NULL;
END;
```

### The three kinds of records (core exam classification)
| Type | Structure comes from | Declared with |
|---|---|---|
| **Table-based record** | The full column list of a table/view | `variable table_or_view%ROWTYPE` |
| **Cursor-based record** | The `SELECT` list of a specific cursor | `variable cursor_name%ROWTYPE` |
| **Custom (user-defined / programmer-defined) record** | Whatever fields you explicitly declare | `TYPE rec_t IS RECORD (...); variable rec_t;` |

All three are accessed the same way — **dot notation**: `record_name.field_name`.

### Important points to remember
- A record is a **PL/SQL-only** construct: it exists purely at compile/runtime inside PL/SQL blocks — it is **not** a database object and cannot be a column datatype in `CREATE TABLE` (contrast with **object types**, Section 4).
- Fields of a record do **not** automatically inherit column-level constraints (`NOT NULL`, `CHECK`) or default values from the source table/view/cursor — only the **datatype** is inherited.
- You **cannot compare two whole records with `=` or `!=`** directly in PL/SQL — there's no built-in record equality operator; you must compare field-by-field (this is a favorite exam trap).
- Whole-record **assignment** (`rec1 := rec2;`) *is* allowed, but only when both records are of the exact same declared type (structurally identical types declared separately are still considered different types).
- Records can be passed as parameters to procedures/functions and returned from functions.

### Exercise Questions

**Q1.** What is the fundamental difference between declaring three separate scalar variables for `name`, `salary`, `dept_id` versus declaring one record with those three fields?
> **A:** Functionally, both let you store and manipulate the same data, but a record groups logically related fields into a **single named unit**, letting you pass/return them as one parameter, assign them as a block via `:=`, and populate them in one `SELECT ... INTO record` rather than one `INTO` target per column. It also makes code more maintainable and self-documenting — related data travels together.

**Q2.** Which of the three record types (table-based, cursor-based, custom) would you use if you need a field that doesn't correspond to any table column at all (e.g., a computed running total)?
> **A:** A **custom (user-defined) record** — since both table-based and cursor-based records derive their field list from an existing table/view or a cursor's `SELECT` list, neither can include an arbitrary field that has no corresponding column/expression. Only an explicit `TYPE ... IS RECORD (...)` declaration lets you add fields freely.

**Q3.** True or False: `rec1 = rec2` is valid PL/SQL if `rec1` and `rec2` are the same record type.
> **A:** **False.** PL/SQL provides no built-in equality/inequality operator for whole records. Attempting `IF rec1 = rec2 THEN` raises a compilation error (`PLS-00306`-class "wrong number or types of arguments" style error, since `=` isn't defined for record types). You must write an explicit field-by-field comparison, e.g. `IF rec1.f1 = rec2.f1 AND rec1.f2 = rec2.f2 THEN`.

---

## 2. Table-Based Records

### What it is
A record declared using **`%ROWTYPE`** against a **table or view name**. Its fields automatically mirror **every column** of that table/view, in the same order, with the same datatypes.

```sql
DECLARE
  v_emp employees%ROWTYPE;   -- one field per column of EMPLOYEES
BEGIN
  SELECT * INTO v_emp
  FROM   employees
  WHERE  employee_id = 100;

  DBMS_OUTPUT.put_line(v_emp.first_name || ' ' || v_emp.last_name);
  DBMS_OUTPUT.put_line('Salary: ' || v_emp.salary);
END;
/
```

### Why it's preferred over listing scalar variables
```sql
-- Fragile: breaks (wrong number of INTO targets) the moment a column is added/removed
SELECT first_name, last_name, salary INTO v_fname, v_lname, v_sal
FROM employees WHERE employee_id = 100;

-- Robust: %ROWTYPE automatically tracks the table's current column list
SELECT * INTO v_emp FROM employees WHERE employee_id = 100;
```

### Practical example — using a table-based record for a full row update
```sql
DECLARE
  v_emp employees%ROWTYPE;
BEGIN
  SELECT * INTO v_emp FROM employees WHERE employee_id = 100;

  v_emp.salary := v_emp.salary * 1.10;   -- give a 10% raise in memory

  UPDATE employees
  SET    ROW = v_emp          -- 12c+ shorthand: update the whole row from the record
  WHERE  employee_id = 100;
END;
/
```
*(Note: the `SET ROW = record` syntax is a 12c convenience for whole-row updates from a `%ROWTYPE` record; many courses still teach field-by-field `SET col1 = rec.col1, col2 = rec.col2, ...` as the traditional/portable approach.)*

### Important points to remember
- `%ROWTYPE` field structure is resolved **at compile time** based on the table/view definition — if the table's structure changes later, dependent PL/SQL units become **invalid** and must be recompiled to pick up the new structure (they recompile automatically on next reference, assuming no errors).
- **Invisible columns are excluded** from a table-based `%ROWTYPE` record, exactly like they're excluded from `SELECT *` and `DESCRIBE` — an important cross-link to the *Invisible Columns* feature (12c+).
- Record fields do **not** inherit `NOT NULL` or `CHECK` constraints, or column default values — only the datatype.
- `%ROWTYPE` can be applied to a **view** just as easily as a table.
- Using `SELECT * INTO record_var` is idiomatic and safe specifically *because* the record's field count/order is guaranteed to match the table's current column list — unlike a hand-written list of scalar `INTO` variables, which silently becomes wrong if the table structure changes and isn't kept in sync.

### Exercise Questions

**Q1.** A developer adds a new column to the `EMPLOYEES` table after a package using `employees%ROWTYPE` was compiled. What happens to the package, and why?
> **A:** The package becomes **invalid** because its dependency (the table) changed. Oracle's dependency-tracking mechanism marks the package invalid; on its next invocation, the database automatically attempts to recompile it, and since `%ROWTYPE` is resolved from the table's current definition, the record picks up the new column automatically (assuming no other compile errors) — no source code change needed, in contrast to a hard-coded list of scalar variables which would need to be manually edited to include the new column.

**Q2.** If the `EMPLOYEES` table has an `INVISIBLE` column called `SSN`, does `v_emp employees%ROWTYPE` contain a `ssn` field? What about `SELECT * INTO v_emp FROM employees`?
> **A:** No to both. Invisible columns are excluded from `%ROWTYPE` declarations exactly as they are excluded from `SELECT *` — so neither the record's structure nor the `SELECT *` query includes the invisible column. To include it, the column must be explicitly referenced by name in both the record's design (which isn't possible with a pure `%ROWTYPE`, since that always mirrors the visible column set) and the query — meaning a **custom record** would be needed to explicitly add an `ssn` field.

**Q3.** Does a table-based `%ROWTYPE` record automatically enforce that its `salary` field can never be `NULL`, if the underlying table column has a `NOT NULL` constraint?
> **A:** No. `%ROWTYPE` fields inherit only the **datatype** of the corresponding column — constraints such as `NOT NULL` and `CHECK`, as well as column default values, are **not** inherited into the record. You could freely assign `v_emp.salary := NULL;` in PL/SQL with no error; the `NOT NULL` constraint would only be enforced later, at the point you actually try to `INSERT`/`UPDATE` the real table with that value.

---

## 3. Custom (User-Defined) Records

### What it is
A record whose structure you **explicitly declare** field-by-field using `TYPE ... IS RECORD`, rather than deriving it from a table, view, or cursor. Gives full control over field names, types, defaults, and nullability.

```sql
DECLARE
  TYPE emp_summary_t IS RECORD (
    emp_id     employees.employee_id%TYPE,
    full_name  VARCHAR2(100),
    bonus_pct  NUMBER(4,2)      DEFAULT 0.05,
    is_active  BOOLEAN          NOT NULL := TRUE   -- NOT NULL requires a default
  );

  v_summary emp_summary_t;
BEGIN
  v_summary.emp_id    := 100;
  v_summary.full_name := 'Steven King';
  v_summary.bonus_pct := 0.10;

  DBMS_OUTPUT.put_line(v_summary.full_name || ' bonus: ' || v_summary.bonus_pct);
END;
/
```

### Key syntax rules
- `TYPE type_name IS RECORD (field1 datatype [[NOT NULL] {:= | DEFAULT} expr], field2 datatype, ...);`
- A field marked `NOT NULL` **must** be given a default value in the declaration (otherwise it would start `NULL`, violating its own constraint immediately) — this is a classic exam gotcha.
- Field datatypes can themselves use `%TYPE` (anchor to a column) or `%ROWTYPE`, and can even be **another record type** (nested records).

### Nested records example
```sql
DECLARE
  TYPE address_t IS RECORD (city VARCHAR2(30), zip VARCHAR2(10));
  TYPE person_t  IS RECORD (name VARCHAR2(50), addr address_t);

  v_person person_t;
BEGIN
  v_person.name      := 'Alice';
  v_person.addr.city := 'Chennai';
  v_person.addr.zip  := '600001';

  DBMS_OUTPUT.put_line(v_person.name || ' lives in ' || v_person.addr.city);
END;
/
```

### Passing records to subprograms
```sql
PROCEDURE give_raise (p_emp IN OUT emp_summary_t, p_pct IN NUMBER) IS
BEGIN
  p_emp.bonus_pct := p_emp.bonus_pct + p_pct;
END;
```

### Important points to remember
- Custom record types declared inside a PL/SQL block/procedure are **local** to that block; to share a record type across multiple program units, declare the `TYPE` in a **package specification** so it becomes a visible, reusable type.
- Whole-record assignment (`v1 := v2;`) works only if `v1` and `v2` share the **identical declared record type** — two independently declared record types with identical field lists are still *not* assignment-compatible with each other (structural typing does **not** apply; it's nominal/name-based typing).
- Records cannot be directly used in `SELECT ... INTO` unless every field maps positionally to a query's select list (field count, order, and compatible types must line up) — this behaves much like a cursor-based record's implicit contract (see Section 5).
- Like table-based records, custom records still **cannot be compared with `=`**, still **cannot be stored directly in a database column**, and still exist only within PL/SQL scope.

### Exercise Questions

**Q1.** Why does `TYPE emp_t IS RECORD (id NUMBER NOT NULL);` (with no default) fail to compile?
> **A:** A field declared `NOT NULL` must be given an initial value at declaration time via `DEFAULT` or `:=`, because a record field with no explicit value defaults to `NULL` — and a `NULL` value would immediately violate its own `NOT NULL` constraint the moment the record variable is created. PL/SQL catches this as a compile-time error (`PLS-00218`-class) rather than letting it fail at runtime.

**Q2.** Two developers each independently declare `TYPE point_t IS RECORD (x NUMBER, y NUMBER);` in two different packages. Can a variable of one package's `point_t` be assigned directly to a variable of the other package's `point_t`?
> **A:** No. PL/SQL record types use **nominal typing** — even though the two `point_t` declarations are structurally identical, they are distinct, unrelated types from the compiler's point of view (each is scoped to/qualified by its own package). Direct assignment between them would raise a `PLS-00382`-class "expression is of wrong type" error; you would need to assign field-by-field instead.

**Q3.** When would you declare a custom record type in a **package specification** rather than inside a single procedure's declaration section?
> **A:** When the same record structure needs to be **shared across multiple program units** — e.g., a procedure that builds the record and a different procedure/function elsewhere that consumes it as a parameter. A record type declared inside one procedure's `DECLARE` section is local to that procedure and invisible elsewhere; putting it in a package spec makes it a reusable, independently referenceable type (`pkg_name.record_type_name`) for any code with visibility to that package.

---

## 4. RECORD Types vs. OBJECT Types

> This is one of the most commonly tested conceptual contrasts — know the table below cold.

### Core distinction
| Aspect | PL/SQL **RECORD** | SQL **OBJECT TYPE** |
|---|---|---|
| Created with | `TYPE ... IS RECORD (...)` inside a PL/SQL declaration section or package | `CREATE [OR REPLACE] TYPE type_name AS OBJECT (...)` — a schema-level DDL statement |
| Persistence | Exists only in PL/SQL memory during block execution; **not a database object** | A genuine, persistent **schema object**, visible in the data dictionary, can be granted privileges on |
| Can be a table column's datatype? | **No** | **Yes** — a table column, or even a whole "object table," can be of an object type |
| Methods (behavior) | **None** — pure data, no functions/procedures attached | Supports **member functions/procedures**, constructors, and comparison methods (`MAP`/`ORDER` member functions) |
| Inheritance | Not supported | Supported — object types can be declared `NOT FINAL` and extended via subtypes (`UNDER`) |
| Equality comparison | Not supported natively (`=` invalid) | Supported if a `MAP` or `ORDER` member function is defined |
| Scope of use | PL/SQL blocks only | Both SQL (tables, views, columns) and PL/SQL |

### Object type example
```sql
CREATE OR REPLACE TYPE address_obj AS OBJECT (
  city  VARCHAR2(30),
  zip   VARCHAR2(10),
  MEMBER FUNCTION full_address RETURN VARCHAR2
);
/

CREATE OR REPLACE TYPE BODY address_obj AS
  MEMBER FUNCTION full_address RETURN VARCHAR2 IS
  BEGIN
    RETURN city || ' - ' || zip;
  END;
END;
/

-- Can be used as a genuine column type, unlike a PL/SQL record:
CREATE TABLE customers (
  cust_id NUMBER,
  addr    address_obj
);

INSERT INTO customers VALUES (1, address_obj('Chennai','600001'));

SELECT c.addr.full_address() FROM customers c;   -- calling the member function in SQL
```

### Important points to remember
- A PL/SQL **record** is the right tool when you just need a **temporary, in-memory grouping of data** for use within PL/SQL logic (e.g., holding one row fetched from a cursor).
- An **object type** is the right tool when you need the structure to be **persisted in the database itself**, participate in SQL queries as a first-class column type, carry **behavior** (methods) alongside its data, or support **inheritance/polymorphism**.
- Because object types are schema objects, they require `CREATE TYPE`/`CREATE TYPE BODY` privileges and appropriate object grants (`EXECUTE`) for other users to reference them — just like a package.
- Object types support **constructors** (an implicit default constructor matching the attribute list, or explicitly defined ones) — records have no such concept; you just assign field values directly.

### Exercise Questions

**Q1.** Can a PL/SQL record be used as the datatype of a table column? Why or why not?
> **A:** No. A PL/SQL record is purely a PL/SQL-runtime construct with no persistent existence in the data dictionary — it cannot be stored as data. To store structured, multi-attribute data in a table column, you need a **SQL object type** (`CREATE TYPE ... AS OBJECT`), which *is* a genuine schema object usable as a column datatype.

**Q2.** A requirement states: "this data structure must support a method that returns a formatted string, and must be storable directly in a table column." Which construct satisfies both requirements, and which does not?
> **A:** A SQL **object type** satisfies both — it supports member functions/procedures (methods) *and* can be used directly as a column datatype in `CREATE TABLE`. A PL/SQL **record** satisfies neither: it has no method/behavior support and cannot be stored in a table column at all.

**Q3.** Why can two object type instances be compared with `=` (if a `MAP` or `ORDER` member function is defined) while two PL/SQL records of the same type cannot be compared at all?
> **A:** Object types are a richer, SQL-integrated construct that explicitly supports defining comparison semantics via `MAP` (maps an object to a scalar value for ordering/comparison) or `ORDER` (defines a pairwise comparison method) member functions — Oracle's object-relational model was designed with comparability, ordering, and use in `ORDER BY`/`DISTINCT` in mind. PL/SQL records were designed as a lightweight, structural grouping mechanism only, with no such comparison protocol ever defined for them — hence `=` is simply undefined for record types.

---

## 5. Cursor-Based Records

### What it is
A record declared using **`%ROWTYPE`** against a **cursor name** (rather than a table/view name). Its field list matches the cursor's **`SELECT` list** exactly — not necessarily the full underlying table structure.

```sql
DECLARE
  CURSOR emp_cur IS
    SELECT employee_id, first_name, salary
    FROM   employees
    WHERE  department_id = 90;

  v_emp emp_cur%ROWTYPE;   -- has EXACTLY 3 fields: employee_id, first_name, salary
BEGIN
  OPEN emp_cur;
  LOOP
    FETCH emp_cur INTO v_emp;
    EXIT WHEN emp_cur%NOTFOUND;
    DBMS_OUTPUT.put_line(v_emp.first_name || ': ' || v_emp.salary);
  END LOOP;
  CLOSE emp_cur;
END;
/
```

### Key distinguishing point vs. table-based `%ROWTYPE`
| | Table-based `%ROWTYPE` | Cursor-based `%ROWTYPE` |
|---|---|---|
| Fields come from | **Every column** of the table/view | **Only the columns/expressions in the cursor's `SELECT` list** |
| Changes if... | The table's column list changes | The cursor's `SELECT` statement changes |

### Cursor `FOR` loop — implicit cursor-based record
The most common real-world use: you don't even need to declare the record yourself — the `FOR` loop implicitly creates one for you, scoped to the loop.
```sql
BEGIN
  FOR r_emp IN (SELECT employee_id, first_name, salary
                FROM employees WHERE department_id = 90)
  LOOP
    DBMS_OUTPUT.put_line(r_emp.first_name || ': ' || r_emp.salary);
  END LOOP;
  -- r_emp is not visible/usable outside the loop
END;
/
```

### Handling expressions in the select list
If the cursor's select list contains an unaliased expression, the record has no clean field name to reference it by — always **alias computed columns**:
```sql
CURSOR c1 IS
  SELECT salary, salary * 12 AS annual_salary   -- alias required
  FROM   employees;

v_rec c1%ROWTYPE;
...
FETCH c1 INTO v_rec;
DBMS_OUTPUT.put_line(v_rec.annual_salary);        -- works because of the alias
```

### Important points to remember
- Cursor-based `%ROWTYPE` field structure is derived from the cursor's **projection** (select list), which may be a subset of columns, a join across multiple tables, or contain computed/aliased expressions — it is **not** tied to any single table's full structure.
- Just like table-based records, fields **do not inherit** `NOT NULL`/`CHECK` constraints or defaults — only datatypes.
- A cursor `FOR LOOP`'s loop-index record is **implicitly declared** — you never write `%ROWTYPE` yourself, but conceptually it behaves exactly like an explicit cursor-based record scoped to the loop body.
- Always give **explicit aliases** to any expression/function call in a cursor's select list; otherwise, the corresponding record field either has an unusable/unpredictable name or (for certain constructs) causes a compile error when referenced.
- `FETCH cursor INTO record` requires the record's field **count and order** to positionally match the cursor's select list — a mismatch causes `ORA-01007`-class errors at runtime (in explicit `INTO` lists) or, more commonly with `%ROWTYPE`, simply avoids the whole problem since the record is generated from the same cursor definition.

### Exercise Questions

**Q1.** A cursor is defined as `CURSOR c1 IS SELECT employee_id, first_name FROM employees;` and you declare `v_rec c1%ROWTYPE;`. Does `v_rec` have a `salary` field?
> **A:** No. Cursor-based `%ROWTYPE` derives its fields strictly from the cursor's `SELECT` list — since `salary` was never included in that list, it simply doesn't exist as a field in `v_rec`, even though `salary` is a real column of the underlying `employees` table. This is the defining difference from a table-based `%ROWTYPE`, which would include `salary` automatically.

**Q2.** Why does `CURSOR c1 IS SELECT salary, salary*12 FROM employees; v_rec c1%ROWTYPE;` followed by `v_rec.???` cause a problem when trying to reference the annual-salary expression?
> **A:** An unaliased expression in a cursor's select list produces a column with either no usable name or an implementation-generated name you can't reliably reference by a clean identifier — so there is no clean field like `v_rec.annual_salary` to use. The fix is to alias the expression in the cursor definition (`salary*12 AS annual_salary`), which gives the corresponding record field an explicit, referenceable name.

**Q3.** In a cursor `FOR` loop such as `FOR r_emp IN (SELECT ... FROM employees) LOOP ... END LOOP;`, do you need to explicitly declare `r_emp`'s type, and can you reference `r_emp` after the loop ends?
> **A:** No explicit declaration is needed — `r_emp` is **implicitly declared** by the `FOR` loop as a cursor-based record matching the query's select list. However, its scope is limited to the loop body: **`r_emp` is not accessible after the loop terminates** (referencing it outside the loop causes a "identifier must be declared" compile error), since it only exists for the duration of the loop construct.

---

## Quick Cross-Topic Summary Table

| Record Type | Declared With | Field List Source | Can Be Stored in a Table Column? | Supports Methods? |
|---|---|---|---|---|
| Table-based | `table_or_view%ROWTYPE` | All visible columns of the table/view | No | No |
| Cursor-based | `cursor_name%ROWTYPE` | The cursor's `SELECT` list | No | No |
| Custom (user-defined) | `TYPE ... IS RECORD (...)` | Explicitly listed fields | No | No |
| Object type *(contrast)* | `CREATE TYPE ... AS OBJECT (...)` | Explicitly listed attributes | **Yes** | **Yes** |