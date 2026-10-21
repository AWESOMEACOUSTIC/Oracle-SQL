# DBMS_SQL — Concept, Usage & Interview Answer

---

## 1. The Problem in One Line

`EXECUTE IMMEDIATE` needs the **shape** of the SQL (how many bind values, how many columns) to be fixed **in the code**.
When that shape is only known **at run time**, `EXECUTE IMMEDIATE` can't express it — `DBMS_SQL` can.

---

## 2. The Analogy (Remember This)

**EXECUTE IMMEDIATE = a fixed combo-meal form**
The form has exactly 2 blanks: *Burger: ___  Drink: ___*.
You can change **what** you write in the blanks (values), but the form **always has 2 blanks**.
Want only a burger? Or burger + drink + fries + dessert? The form doesn't fit — you'd need a different printed form for every combination.

**DBMS_SQL = telling the waiter item by item**
"One burger." "Add fries." "Also a drink." "That's it."
You decide **how many items while ordering**, not in advance.

| Situation | Tool |
|---|---|
| Different burger every day (values change, shape same) | EXECUTE IMMEDIATE |
| Different **number** of items every order (shape changes) | DBMS_SQL |

### Common misunderstanding
It is **not** about "requirements changing in the future" — if requirements change next month, you just edit the code.
It is about whether the **shape changes while the program is running**, from one execution to the next.

- **EXECUTE IMMEDIATE** → values change, but the *number* of values/columns is always the same.
  e.g. always `dept = ? AND salary > ?` → today (50, 3000), tomorrow (80, 5000). Still 2 blanks.
- **DBMS_SQL** → the *number* of values/columns itself changes per run.
  e.g. user gives 1 filter now, 3 filters next time, 0 the time after.

---

## 3. Why EXECUTE IMMEDIATE Fails Here

```sql
EXECUTE IMMEDIATE v_sql INTO v_a, v_b USING p_x, p_y;
--                       ^^^^^^^^^^^^    ^^^^^^^^^^
--        These lists are written into the code — they can't grow or shrink at run time.
```

With 4 optional filters you'd need **2⁴ = 16** separate `EXECUTE IMMEDIATE` branches, one per combination of the `USING` list.

DBMS_SQL fixes this because **each bind is its own call** (`BIND_VARIABLE`), and **each column is read with its own call** (`COLUMN_VALUE`) — so you can call them 0, 1, 3 or 10 times, inside `IF`s or loops.

---

## 4. The DBMS_SQL Call Sequence

| Step | Subprogram | Purpose |
|---|---|---|
| 1 | `OPEN_CURSOR` | Get a cursor id |
| 2 | `PARSE(c, sql, DBMS_SQL.NATIVE)` | Parse the text (**DDL executes here**) |
| 3 | `BIND_VARIABLE` | Supply values — **by name** (`:dept`) |
| 4 | `DEFINE_COLUMN` | (queries) Declare a variable per column — **by position** |
| 5 | `EXECUTE` | Run it (DML returns rows processed) |
| 6 | `FETCH_ROWS` | (queries) Next row; returns 0 when done |
| 7 | `COLUMN_VALUE` | Copy a fetched column into a variable |
| 8 | `CLOSE_CURSOR` | Release the cursor |

Helpers: `IS_OPEN`, `DESCRIBE_COLUMNS`, `TO_REFCURSOR`, `TO_CURSOR_NUMBER`.

---

## 5. Where You Need DBMS_SQL

Default to EXECUTE IMMEDIATE. Use DBMS_SQL when:

1. **Unknown number of bind variables** — search screens with optional filters.
2. **Unknown select list ("Method 4")** — generic query printer, CSV exporter, user-picked report columns (`DESCRIBE_COLUMNS`).
3. **Parse once, execute many** — same statement run many times with different values.
4. **Statement text > 32 KB** — `PARSE` accepts a collection of text pieces.
5. **Build dynamically, return a REF CURSOR** — `TO_REFCURSOR`.

## 6. Why It Matters

- **Fills the gap NDS leaves** — no 16-branch code for 4 optional filters.
- **Performance** — binds keep values out of SQL text → plan reuse; parse-once saves more.
- **Security** — binds prevent SQL injection for values (identifiers still need `DBMS_ASSERT`).

---

## 7. Example Scenario 1 — HR Search Screen with Optional Filters

**Requirement:** Filters for department, job, minimum salary, name — all optional, any combination. Return results as `SYS_REFCURSOR` to the UI.

```sql
CREATE OR REPLACE FUNCTION search_employees (
  p_dept_id    IN NUMBER   DEFAULT NULL,
  p_job_id     IN VARCHAR2 DEFAULT NULL,
  p_min_salary IN NUMBER   DEFAULT NULL,
  p_name_like  IN VARCHAR2 DEFAULT NULL
) RETURN SYS_REFCURSOR
IS
  c      INTEGER;
  v_sql  VARCHAR2(4000) :=
         'SELECT employee_id, first_name, last_name, job_id, salary
            FROM employees
           WHERE 1 = 1';                 -- anchor so every filter can start with AND
  v_rows INTEGER;
BEGIN
  -- 1. Build the WHERE clause from the filters actually supplied
  IF p_dept_id    IS NOT NULL THEN v_sql := v_sql || ' AND department_id = :dept';             END IF;
  IF p_job_id     IS NOT NULL THEN v_sql := v_sql || ' AND job_id = :job';                     END IF;
  IF p_min_salary IS NOT NULL THEN v_sql := v_sql || ' AND salary >= :minsal';                 END IF;
  IF p_name_like  IS NOT NULL THEN v_sql := v_sql || ' AND UPPER(last_name) LIKE UPPER(:nm)';  END IF;
  v_sql := v_sql || ' ORDER BY last_name';

  -- 2. Open and parse
  c := DBMS_SQL.open_cursor;
  DBMS_SQL.parse(c, v_sql, DBMS_SQL.native);

  -- 3. Bind ONLY the placeholders that exist (variable number of binds)
  IF p_dept_id    IS NOT NULL THEN DBMS_SQL.bind_variable(c, ':dept',   p_dept_id);                 END IF;
  IF p_job_id     IS NOT NULL THEN DBMS_SQL.bind_variable(c, ':job',    p_job_id);                  END IF;
  IF p_min_salary IS NOT NULL THEN DBMS_SQL.bind_variable(c, ':minsal', p_min_salary);              END IF;
  IF p_name_like  IS NOT NULL THEN DBMS_SQL.bind_variable(c, ':nm',     '%' || p_name_like || '%'); END IF;

  -- 4. Execute, then hand back an ordinary REF CURSOR
  v_rows := DBMS_SQL.execute(c);
  RETURN DBMS_SQL.to_refcursor(c);       -- 'c' must not be used/closed after this

EXCEPTION
  WHEN OTHERS THEN
    IF DBMS_SQL.is_open(c) THEN DBMS_SQL.close_cursor(c); END IF;  -- prevent ORA-01000 leaks
    RAISE;
END search_employees;
/
```

**Calling it** — the caller just sees a normal cursor:

```sql
DECLARE
  rc     SYS_REFCURSOR;
  v_id   employees.employee_id%TYPE;
  v_fn   employees.first_name%TYPE;
  v_ln   employees.last_name%TYPE;
  v_job  employees.job_id%TYPE;
  v_sal  employees.salary%TYPE;
BEGIN
  rc := search_employees(p_dept_id => 50, p_min_salary => 3000);   -- only 2 filters used

  LOOP
    FETCH rc INTO v_id, v_fn, v_ln, v_job, v_sal;
    EXIT WHEN rc%NOTFOUND;
    DBMS_OUTPUT.put_line(v_id || ' ' || v_ln || ' ' || v_sal);
  END LOOP;
  CLOSE rc;
END;
/
```

**Why this is good design**
- One statement instead of 16 branches.
- All values bound → injection-safe, plans reused.
- Cursor closed in the exception handler → no leaks.
- UI layer receives a standard `SYS_REFCURSOR`.

> Alternative with NDS: `AND (:dept IS NULL OR department_id = :dept)` with a fixed `USING` list. It works, but one generic plan serves all combinations and is often poor. DBMS_SQL gives each combination its own optimal plan.

---

## 8. Example Scenario 2 — Parse Once, Execute Many

```sql
DECLARE
  TYPE t_raise IS TABLE OF NUMBER INDEX BY PLS_INTEGER;   -- dept_id -> pct
  v_raise t_raise;
  c       INTEGER;
  v_n     INTEGER;
  d       PLS_INTEGER;
BEGIN
  v_raise(50) := 1.05;  v_raise(60) := 1.08;  v_raise(80) := 1.03;

  c := DBMS_SQL.open_cursor;
  DBMS_SQL.parse(c,
    'UPDATE employees SET salary = salary * :pct WHERE department_id = :dept',
    DBMS_SQL.native);                                      -- parsed only ONCE

  d := v_raise.FIRST;
  WHILE d IS NOT NULL LOOP
    DBMS_SQL.bind_variable(c, ':pct',  v_raise(d));        -- re-bind ...
    DBMS_SQL.bind_variable(c, ':dept', d);
    v_n := DBMS_SQL.execute(c);                            -- ... re-execute
    DBMS_OUTPUT.put_line('Dept ' || d || ': ' || v_n || ' rows');
    d := v_raise.NEXT(d);
  END LOOP;

  DBMS_SQL.close_cursor(c);
EXCEPTION
  WHEN OTHERS THEN
    IF DBMS_SQL.is_open(c) THEN DBMS_SQL.close_cursor(c); END IF;
    RAISE;
END;
/
```

(For pure bulk DML, `FORALL` is often better — this shows the DBMS_SQL pattern.)

---

## 9. Quick Decision Rule

```
Do you know, while writing the code,
the number/types of select columns AND the number of binds?
   ├── YES → EXECUTE IMMEDIATE (simpler, faster)
   └── NO  → DBMS_SQL
Also DBMS_SQL if: same statement executed very many times,
statement text > 32 KB, or you need DESCRIBE_COLUMNS.
```

---

## 10. Interview Answer (~30–40 seconds)

> "Both EXECUTE IMMEDIATE and DBMS_SQL run dynamic SQL — the difference is whether the **shape** of the statement is fixed.
>
> Take an HR search screen. If the search always uses the same filters — say department and salary — only the values change from run to run. The shape is fixed, so EXECUTE IMMEDIATE is the best choice: simple and fast.
>
> But suppose the filters are optional. One user searches by department only, another by department, job and salary, another with no filter at all. Now the **number of bind variables changes every run**. EXECUTE IMMEDIATE can't handle that, because its USING clause is written in the code with a fixed number of values — I'd have to write a separate branch for every combination of filters.
>
> DBMS_SQL solves this because each bind is a **separate call**. I build the WHERE clause based on the filters provided, and call BIND_VARIABLE only for those filters. The same applies when the number of columns returned isn't known until run time — like a generic report or export tool — where I can use DESCRIBE_COLUMNS to discover them.
>
> So my rule is: **EXECUTE IMMEDIATE by default, DBMS_SQL when the number of binds or columns is only known at run time.**"

**Why this answer works**
1. Opens with a one-line definition of the difference.
2. Explains **why** EXECUTE IMMEDIATE fails (fixed USING list).
3. Explains **how** DBMS_SQL solves it (BIND_VARIABLE per filter).
4. Adds a second use case (unknown columns) to show depth.
5. Ends with a clear decision rule.

---

## 11. Points to Memorise

- **Bind by name** (`:dept`); **define/read columns by position** (1, 2, 3…).
- **DDL runs at PARSE** — no `EXECUTE` needed (and it commits implicitly).
- **Always close cursors**, including in the exception handler → otherwise **ORA-01000: maximum open cursors exceeded**.
- **`TO_REFCURSOR`** converts a DBMS_SQL cursor → REF CURSOR (cursor number unusable after); **`TO_CURSOR_NUMBER`** does the reverse.
- Since 11g, security checks reject invalid/guessed cursor numbers (e.g. **ORA-29471**).