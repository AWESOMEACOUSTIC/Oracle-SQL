# Module 1, Topic 5: Operators & Variables in PL/SQL

---

## Part A: Variables (Formalized)

### 1. What This Covers

You've been declaring and using variables informally since Module 1, Topic 1. This section formalizes the **complete syntax and rules** for variable declaration — including two features you haven't seen yet: **`%TYPE` anchoring** and **`NOT NULL` constraints on variables** — both of which are genuinely important for writing robust, maintainable PL/SQL.

### 2. Syntax: Variable Declaration

```sql
variable_name datatype [NOT NULL] [:= initial_value | DEFAULT initial_value];
```

| Element | Meaning |
|---|---|
| `variable_name` | A valid identifier (per Topic 3's rules). |
| `datatype` | Any scalar, composite, reference, or LOB type (per Topic 4). |
| `NOT NULL` | Optional constraint — this variable can **never** hold `NULL`. If declared `NOT NULL`, you **must** also provide an initial value (a `NOT NULL` variable can't start out unset). |
| `:=` or `DEFAULT` | Both assign an initial value — they are functionally **interchangeable** for variable initialization; `:=` is more commonly used by convention, but `DEFAULT` is equally valid. |

### 3. Simple Examples
```sql
DECLARE
    v_name       VARCHAR2(50);                         -- no initial value; starts as NULL
    v_salary     NUMBER := 50000;                       -- initialized using :=
    v_status     VARCHAR2(10) DEFAULT 'ACTIVE';         -- initialized using DEFAULT (same effect as :=)
    v_max_limit  NUMBER NOT NULL := 100;                -- NOT NULL requires an initial value
BEGIN
    NULL;
END;
/
```

### 4. `%TYPE` — Anchoring a Variable to Another Variable or Column's Type

**What it is:** `%TYPE` lets you declare a variable's datatype by **referencing** an existing column or another variable's type, instead of hardcoding the datatype yourself.

```sql
DECLARE
    v_salary employees.salary%TYPE;      -- automatically matches the employees.salary column's exact datatype
    v_bonus  v_salary%TYPE;              -- matches whatever datatype v_salary ends up being
BEGIN
    NULL;
END;
/
```

**Why this matters (the real motivation):** Imagine the `employees.salary` column is currently `NUMBER(8,2)`, and dozens of procedures across your codebase declare local variables as `NUMBER(8,2)` to match it. If the DBA later changes the column to `NUMBER(10,2)` to accommodate larger salaries, **every one of those hardcoded declarations is now silently out of sync** with the actual column — potentially causing truncation or precision issues. If instead every one of those variables was declared using `employees.salary%TYPE`, they **all automatically adapt** the moment the column definition changes, with zero code changes required.

This is conceptually the **exact same benefit** `%ROWTYPE` gave you in Topic 4 (automatic adaptation to underlying structure) — `%TYPE` is simply the single-column version of that same idea.

### 5. Detailed Explanation — Why `%TYPE` Is Considered Best Practice

- **Consistency**: guarantees your variable can hold *exactly* whatever the column can hold, with no manual guessing about precision/length.
- **Maintainability**: schema changes propagate automatically to dependent PL/SQL code, without hunting down every hardcoded declaration.
- **Self-documentation**: `v_salary employees.salary%TYPE` immediately tells a reader "this variable mirrors the employees table's salary column," which is more informative than a bare `NUMBER(8,2)`.

Real, professional PL/SQL codebases use `%TYPE` (and `%ROWTYPE`) **extensively** — hardcoding datatypes that are meant to mirror table structures is generally considered a code smell once you know `%TYPE` exists.

### 6. Common Mistakes & Misconceptions
1. **Mistake**: Declaring a `NOT NULL` variable without an initial value → compile error, since Oracle can't guarantee non-null without something to initialize it to.
2. **Misconception**: "`:=` and `DEFAULT` behave differently." → They don't, for simple variable initialization — purely a stylistic choice (though `DEFAULT` is more commonly seen in parameter declarations, as you saw back in Module 3).
3. **Mistake**: Hardcoding a datatype that's meant to track a table column (`v_salary NUMBER(8,2);`) instead of using `%TYPE` — this works fine until the column changes, at which point it becomes a silent maintenance risk.
4. **Misconception**: "`%TYPE` only works with table columns." → It also works anchored to **other variables** (`v_bonus v_salary%TYPE;`), which is useful for keeping a group of related variables consistent with each other.

---

## Part B: Operators

### 7. What This Covers

**Operators** are symbols/keywords that perform operations on values — arithmetic, comparisons, logical combinations, string concatenation, and a few special-purpose operators. You've used several already without a formal name; this section organizes them completely.

### 8. Arithmetic Operators

| Operator | Meaning | Example |
|---|---|---|
| `+` | Addition | `v_total := v_a + v_b;` |
| `-` | Subtraction | `v_diff := v_a - v_b;` |
| `*` | Multiplication | `v_product := v_a * v_b;` |
| `/` | Division | `v_quotient := v_a / v_b;` |
| `**` | Exponentiation | `v_squared := v_a ** 2;` |

**Note**: PL/SQL has no dedicated integer-division or modulo *operator* — for remainder/modulo, you use the built-in `MOD(v_a, v_b)` function instead.

### 9. Comparison Operators

| Operator | Meaning |
|---|---|
| `=` | Equal to |
| `!=` or `<>` | Not equal to (both are valid and equivalent) |
| `<`, `>`, `<=`, `>=` | Less than, greater than, and their "or equal to" variants |
| `BETWEEN ... AND ...` | Value falls within an inclusive range |
| `IN (...)` | Value matches any one of a listed set |
| `LIKE` | Pattern matching using `%` (any number of characters) and `_` (exactly one character) as wildcards |
| `IS NULL` / `IS NOT NULL` | Tests specifically for `NULL` — **never** use `= NULL`, which is a classic, important mistake covered below |

```sql
IF v_salary BETWEEN 40000 AND 80000 THEN ...
IF v_department_id IN (10, 20, 30) THEN ...
IF v_name LIKE 'A%' THEN ...            -- starts with 'A'
IF v_manager_id IS NULL THEN ...        -- correct way to test for NULL
```
*(These are shown here for completeness of the operator reference — full `IF` syntax and behavior is formally covered in Module 2, so don't worry yet about mastering the surrounding control-flow structure, just the operators themselves.)*

### 10. Logical Operators

| Operator | Meaning |
|---|---|
| `AND` | True only if **both** conditions are true |
| `OR` | True if **at least one** condition is true |
| `NOT` | Reverses a condition's truth value |

### 11. Concatenation Operator

`||` joins two strings (or string-convertible values) together — you've used this constantly since Module 1, Topic 1.
```sql
v_message := 'Hello, ' || v_name || '!';
```

### 12. Assignment Operator

`:=` assigns a value to a variable — this is **distinct** from `=`, which is used only for **comparison**. This is one of the most fundamental syntax distinctions in the entire language, and mixing them up is a very common beginner error.
```sql
v_x := 10;        -- assignment: v_x now holds 10
IF v_x = 10 THEN  -- comparison: is v_x equal to 10?
```

---

## 13. The NULL-Handling Rules — Critical, Frequently Misunderstood

This deserves its own dedicated attention, because it's one of the most consequential and commonly mistested areas in all of PL/SQL/SQL.

- **`NULL` means "unknown/absent," not "zero" or "empty."** Any arithmetic operation involving `NULL` produces `NULL`: `5 + NULL` is `NULL`, not `5`.
- **Any comparison involving `NULL` using `=`, `!=`, `<`, `>`, etc. evaluates to `NULL` (neither true nor false)** — not `TRUE`, not `FALSE`. This means `v_x = NULL` is **always** `NULL`, **never** `TRUE`, even if `v_x` genuinely holds `NULL` — this is exactly why you must use `IS NULL` / `IS NOT NULL` instead.
- **String concatenation treats `NULL` as if it were an empty string**: `'Hello' || NULL` produces `'Hello'`, not `NULL` — this is a special-case exception to the "NULL poisons everything" rule, specific to `||`.
- Logical operators have specific `NULL`-handling rules too: `TRUE OR NULL` is `TRUE` (short-circuits), but `FALSE OR NULL` is `NULL`; `FALSE AND NULL` is `FALSE` (short-circuits), but `TRUE AND NULL` is `NULL`.

---

## 14. Common Mistakes & Misconceptions (Operators)

1. **Mistake**: Writing `IF v_x = NULL THEN` → this **never** evaluates to `TRUE`, regardless of `v_x`'s actual value — must use `IF v_x IS NULL THEN`.
2. **Mistake**: Confusing `:=` (assignment) with `=` (comparison) — using `=` where you meant to assign a value, or vice versa, causes compile errors or, worse, silently wrong logic depending on context.
3. **Misconception**: "Division by a NULL denominator raises ZERO_DIVIDE." → It does not — `NULL` division doesn't trigger `ZERO_DIVIDE` at all; it simply produces `NULL` as the result, silently, following standard NULL-arithmetic rules. Only actual **zero** as the denominator raises `ZERO_DIVIDE` (as you learned in Module 4, Topic 2).
4. **Misconception**: "5 + NULL raises an error." → It does not raise an error; it silently evaluates to `NULL` — a genuinely important distinction, since silent `NULL` propagation, not a loud error, is often the actual source of confusing downstream bugs.
5. **Mistake**: Assuming `LIKE` pattern wildcards work the same as general "regex" wildcards — `%` and `_` are the **only** two special characters in standard `LIKE` matching; anything else is literal.

---

## 15. Edge Cases to Be Aware Of

- `NULL || NULL` produces `NULL` (not an empty string) — the "NULL treated as empty string" special case for concatenation only kicks in when at least one side has an actual, non-null value to concatenate with.
- Comparing two `NULL`s directly with `=` still yields `NULL`, not `TRUE` — even two "identical" unknowns are not considered equal to each other, by design, since both are simply "unknown."
- `BETWEEN` is always **inclusive** on both ends — `v_x BETWEEN 10 AND 20` includes both 10 and 20 themselves.

---

## 16. Interview-Level / Practical Notes

- *"Why doesn't `v_x = NULL` work to check for NULL?"* — Because any comparison involving `NULL` returns `NULL`, not `TRUE`/`FALSE` — a foundational SQL/PL-SQL three-valued-logic concept, extremely commonly tested.
- *"What's the benefit of using %TYPE instead of hardcoding a datatype?"* — Automatic adaptation to underlying column changes, consistency, and self-documentation — directly parallels the `%ROWTYPE` reasoning from Topic 4.
- *"What does string concatenation do with a NULL operand?"* — Treats it as an empty string, unlike almost every other NULL-involving operation, which is a good "gotcha" question testing genuine understanding versus surface memorization.

---

## Things You Must Remember

- `%TYPE` anchors a variable's datatype to a column (or another variable) — use it whenever a variable is meant to mirror a table column, for automatic adaptation to schema changes.
- `NOT NULL` variables **must** have an initial value at declaration.
- `:=` is assignment; `=` is comparison — never confuse them.
- **Never** use `= NULL` or `!= NULL` to test for `NULL` — always use `IS NULL` / `IS NOT NULL`.
- Arithmetic with `NULL` silently produces `NULL` (no error) — this is different from division by **zero**, which raises `ZERO_DIVIDE`.
- Concatenation (`||`) is the one major exception to NULL's "poisons everything" behavior — it treats `NULL` as an empty string.
- `BETWEEN` is inclusive on both bounds.

## How to Recognize This Concept

Reach for **`%TYPE`** whenever you're declaring a variable specifically meant to **hold a value from, or eventually be compared/assigned to, a particular table column** — this should become close to a reflex, not an occasional choice.

Watch for **NULL-related traps** whenever a requirement involves data that **might legitimately be missing** — optional fields, unset foreign keys, not-yet-calculated values — these are exactly the situations where careless `= NULL` comparisons or unguarded arithmetic silently produce wrong (but not obviously "erroring") results, which is often more dangerous in production than a loud crash.

---

## Exercises

1. **(%TYPE basic use)** Declare a variable that anchors its type to `employees.hire_date`, and another variable anchored to that first variable's type.

2. **(NOT NULL)** Declare a `NOT NULL` variable representing a company's fixed minimum wage value, and explain what would happen (and why) if you tried to declare it without providing an initial value.

3. **(NULL trap, predict output)** What does this print, and why?
   ```sql
   DECLARE
       v_bonus NUMBER;  -- starts as NULL
       v_salary NUMBER := 50000;
   BEGIN
       DBMS_OUTPUT.PUT_LINE('Total: ' || (v_salary + v_bonus));
   END;
   /
   ```

4. **(NULL comparison trap)** Explain why this block never prints anything, and rewrite it correctly:
   ```sql
   DECLARE
       v_manager_id NUMBER := NULL;
   BEGIN
       IF v_manager_id = NULL THEN
           DBMS_OUTPUT.PUT_LINE('No manager assigned.');
       END IF;
   END;
   /
   ```

5. **(Concatenation exception)** Predict the output, and explain why it differs from ordinary NULL-arithmetic behavior:
   ```sql
   DECLARE
       v_middle_name VARCHAR2(20) := NULL;
   BEGIN
       DBMS_OUTPUT.PUT_LINE('First' || v_middle_name || 'Last');
   END;
   /
   ```

6. **(Realistic business scenario)** A payroll calculation does `v_final_pay := v_base_pay + v_overtime_pay + v_bonus;`, where `v_bonus` might legitimately be `NULL` for employees who didn't qualify this month. Explain what will actually happen to `v_final_pay` for such an employee, why this might silently produce an incorrect payroll result rather than an obvious error, and how you'd fix the calculation to handle this correctly (a brief mention of the `NVL` function, even if not yet formally taught, is fine if you're aware of it — otherwise describe the fix conceptually).

---

*Share your answers whenever you're ready. Next up: Module 1, Topic 6 — Lend a Hand on PL/SQL, the final practice checkpoint for this module.*
