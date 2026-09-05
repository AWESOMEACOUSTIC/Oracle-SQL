# Module 1, Topic 3: PL/SQL Building Elements

> This topic covers the smallest, most fundamental building blocks of PL/SQL syntax: **identifiers**, **literals**, the **semicolon delimiter**, and **comments**. These are things you've already been using informally throughout Modules 3 and 4 — this topic makes the rules explicit and precise, so you stop relying on "it just seemed to work" and start knowing exactly *why* it works, and where the real boundaries are.

---

## 1. Identifiers

### What They Are
An **identifier** is a name you give to something in PL/SQL — a variable, a constant, a procedure, a function, a package, a parameter, an exception. Every named thing you've created so far (`v_salary`, `get_employee_salary`, `pkg_order_processing`, `e_insufficient_balance`) is an identifier.

### Why Rules Around Identifiers Exist
Oracle needs an unambiguous way to parse your code and distinguish "this is a name I declared" from "this is a keyword/reserved word" from "this is a literal value." The identifier rules exist to make that parsing reliable and consistent.

### The Rules
- Must **begin with a letter**.
- Can contain letters, digits, underscores (`_`), dollar signs (`$`), and hash/pound signs (`#`) after the first character.
- Maximum length: **128 bytes** (in modern Oracle versions — this limit was smaller, 30 bytes, in older versions; worth knowing this changed, in case you ever encounter legacy documentation citing the old limit).
- **Case-insensitive** by default — `v_Salary`, `V_SALARY`, and `v_salary` all refer to the **same** identifier unless you use double-quoted "case-sensitive" identifiers (a rare, generally discouraged practice — quoted identifiers also allow spaces/special characters, but using them is considered poor style in normal PL/SQL work).
- **Cannot be a reserved word** (`BEGIN`, `END`, `SELECT`, `IF`, etc.) unless quoted — and even then, it's poor practice to try.

### Naming Conventions (Practical, Not Enforced by Oracle, But Universally Expected)
These aren't Oracle rules — they're **industry-standard conventions** that make code readable and are expected in any real company codebase:

| Prefix | Meaning | Example |
|---|---|---|
| `v_` | Local variable | `v_salary` |
| `p_` | Parameter | `p_employee_id` |
| `g_` | Package-level (global) variable | `g_current_user` |
| `c_` | Constant | `c_max_salary` |
| `e_` | User-defined exception | `e_insufficient_balance` |
| `pkg_` | Package name | `pkg_order_processing` |

You've actually been using most of these conventions already, without them being formally named — this topic is simply making that implicit knowledge explicit.

### Common Mistakes
1. Starting an identifier with a digit or underscore (`1_salary`, `_temp`) → invalid.
2. Using a reserved word as an identifier without realizing it (e.g., naming a variable `number` or `date`) → often causes confusing compile errors, since Oracle tries to interpret it as the keyword instead.
3. Assuming identifiers are case-sensitive by default → they are not; `V_Total` and `v_total` are identical to Oracle.

---

## 2. Literals

### What They Are
A **literal** is a fixed, explicit value written directly in your code — not a variable, not a calculation, just the raw value itself: `100`, `'Hello'`, `TRUE`, `DATE '2026-01-15'`.

### Types of Literals

| Type | Examples | Notes |
|---|---|---|
| **Numeric** | `100`, `3.14`, `-25`, `1.5e3` (scientific notation) | No quotes. Can include a sign, decimal point, or exponent. |
| **Character (string)** | `'Hello'`, `'O''Brien'` | Always enclosed in **single quotes**. To include an actual single quote *inside* a string, double it up (`''`) — this is the standard escaping mechanism. |
| **Date** | `DATE '2026-01-15'` | Uses the ANSI date literal format, always `YYYY-MM-DD`, prefixed with the `DATE` keyword. |
| **Boolean** | `TRUE`, `FALSE`, `NULL` | Only meaningful for PL/SQL `BOOLEAN` variables — remember from Module 3 that SQL itself has no boolean type, which is exactly why SQL-callable functions typically avoid returning `BOOLEAN`. |

### Simple Examples
```sql
DECLARE
    v_name       VARCHAR2(50) := 'O''Brien';      -- escaped single quote
    v_price      NUMBER := 49.99;
    v_start_date DATE := DATE '2026-01-15';
    v_is_active  BOOLEAN := TRUE;
BEGIN
    DBMS_OUTPUT.PUT_LINE(v_name || ' - ' || v_price);
END;
/
```

### Common Mistakes
1. Forgetting to escape a single quote inside a string literal (`'O'Brien'` instead of `'O''Brien'`) → syntax error, since Oracle interprets the first unescaped quote as ending the string early.
2. Writing a date as a plain string (`'2026-01-15'`) and expecting PL/SQL to automatically treat it as a `DATE` — it will actually be treated as a `VARCHAR2` literal unless explicitly converted (via `TO_DATE` or the `DATE` literal keyword), which can cause subtle bugs in comparisons/calculations later.
3. Confusing `NULL` with an empty string `''` for `VARCHAR2` — in Oracle, an empty string and `NULL` are treated as equivalent for `VARCHAR2`, which is a famous, frequently-tested Oracle-specific quirk (most other databases distinguish them) — worth remembering explicitly.

---

## 3. The Semicolon Delimiter

### What It Is
The semicolon (`;`) marks the **end of a single PL/SQL statement** — a declaration, an assignment, a call, an `IF`, etc. This is distinct from the `/` you've seen at the end of full blocks (Module 1, Topic 1) — the `/` is a **client-tool instruction** telling SQL*Plus/SQLcl "execute the block now"; the `;` is **part of the PL/SQL language itself**, terminating each individual statement inside the block.

### Why This Distinction Matters
Beginners often confuse these two. To be precise:
- **Every individual statement** inside a block ends with `;` — this includes the final `END` of the block itself (`END;`).
- The `/` appears **once**, after the entire block (including the final `END;`), and is **not** part of PL/SQL syntax — it's purely a signal to the client tool.

```sql
DECLARE
    v_x NUMBER := 5;   -- statement 1, ends with ;
BEGIN
    v_x := v_x + 1;    -- statement 2, ends with ;
    DBMS_OUTPUT.PUT_LINE(v_x);  -- statement 3, ends with ;
END;                    -- this is itself a statement, so it also ends with ;
/                        -- NOT part of PL/SQL — a client-tool "run it now" signal
```

### Common Mistakes
1. Forgetting a semicolon after any statement → compile error (usually reported at the *next* line, which confuses beginners into looking in the wrong place for the mistake).
2. Adding a semicolon after `END` in a sub-block header context where it's not expected (rare, but worth being precise: `END;` always needs its semicolon; it's easy to accidentally type `END` without one when writing quickly).
3. Treating `/` as if it were meaningful PL/SQL syntax, or wondering "what if I forget it" as if it were a language rule — it's purely a tool convention; some GUI tools (like SQL Developer's "Run Script") don't require it at all in the same way.

---

## 4. Comments

### What They Are
**Comments** are text in your code that Oracle's compiler **completely ignores** — meant purely for human readers (including future-you). PL/SQL supports two styles:

| Style | Syntax | Scope |
|---|---|---|
| **Single-line** | `-- comment text` | From `--` to the end of that line only |
| **Multi-line (block)** | `/* comment text */` | Everything between `/*` and `*/`, can span multiple lines |

### Why Comments Matter (Beyond the Obvious)
- Documenting **why** something is done a certain way (not just *what* it does — the code itself already shows *what*) is the highest-value use of comments. E.g., `-- Using RESULT_CACHE here because this lookup rarely changes but is called on every transaction` is far more valuable than `-- caching the function`.
- In real teams, comments often explain **business context** that isn't visible from the code alone — e.g., `-- Per Finance policy FIN-2024-11, discounts above 50% require VP approval`.

### Simple Examples
```sql
DECLARE
    -- Maximum allowed discount percentage per company policy
    c_max_discount NUMBER := 50;
BEGIN
    /* This block validates the requested discount
       against the company-wide maximum before proceeding. */
    IF c_max_discount > 100 THEN  -- defensive check, should never actually happen
        NULL;
    END IF;
END;
/
```

### Common Mistakes
1. Trying to **nest** `/* ... */` block comments (`/* outer /* inner */ still outer */`) → this does **not** work as you might expect in PL/SQL; the first `*/` closes the comment, and everything after it (`still outer */`) is interpreted as actual code, likely causing a syntax error.
2. Over-commenting obvious code (`v_x := v_x + 1; -- increment x by 1`) — this adds noise without value; good commenting explains **intent/reasoning**, not restates syntax.
3. Leaving stale comments that no longer match the code after edits — arguably worse than no comment at all, since they actively mislead a future reader.

---

## Things You Must Remember

- Identifiers: start with a letter, up to 128 bytes, case-insensitive by default, cannot be a reserved word (unquoted).
- Standard naming conventions (`v_`, `p_`, `g_`, `c_`, `e_`, `pkg_`) aren't enforced by Oracle but are universally expected in real code.
- String literals use single quotes; escape an embedded single quote by doubling it (`''`).
- In Oracle, an **empty string (`''`) and `NULL` are treated as equivalent for VARCHAR2** — a famous, Oracle-specific quirk worth remembering explicitly.
- `;` terminates every individual PL/SQL statement (including `END;`); `/` is a **client-tool convention**, not part of the PL/SQL language.
- Block comments (`/* */`) **do not nest** — the first `*/` closes the whole comment, regardless of any `/*` seen after the outer one opened.
- Good comments explain **why**, not **what** — the code already shows what.

## How to Recognize This Concept
This topic isn't about recognizing *when* to apply it in a business requirement — it's foundational syntax you use in **every single line** of PL/SQL you write. The "recognition" skill here is really about **catching your own small mistakes quickly**: a missing semicolon, an unescaped quote, a reserved word accidentally used as a variable name, an empty string silently behaving like `NULL` in a comparison you didn't expect. Building a habit of mentally checking these when debugging saves enormous time.

---

## Exercises

1. **(Identifier validity)** For each of the following, state whether it's a valid PL/SQL identifier, and why or why not: `v_total`, `2nd_value`, `emp#1`, `select`, `V_Total`, `_temp`, `emp_id_2026`.

2. **(Literal escaping)** Write a `DECLARE` section that assigns the string `It's O'Brien's report` to a `VARCHAR2` variable, correctly escaped.

3. **(Empty string vs NULL)** Predict the output:
   ```sql
   DECLARE
       v_test VARCHAR2(10) := '';
   BEGIN
       IF v_test IS NULL THEN
           DBMS_OUTPUT.PUT_LINE('It is NULL');
       ELSE
           DBMS_OUTPUT.PUT_LINE('It is NOT NULL');
       END IF;
   END;
   /
   ```
   *(Note: this uses `IF`, formally covered in Module 2 — previewed minimally here since this exact quirk is the whole point of the exercise.)*

4. **(Semicolon vs slash)** Explain, in your own words, the difference between what `;` and `/` each actually do, using a short example block to illustrate.

5. **(Comment nesting pitfall)** Explain what's wrong with this code and how you'd fix it:
   ```sql
   DECLARE
       v_x NUMBER := 5; /* This variable /* holds a test value */ for our demo */
   BEGIN
       NULL;
   END;
   /
   ```

6. **(Naming convention judgment)** A colleague declares a package-level variable as `total_records` (no prefix) inside a package body that also has a local variable `v_total_records` inside one of its procedures. Explain, using the conventions from this topic, why this naming choice could cause confusion during a code review, and suggest a better name for the package-level variable.

---

*Share your answers whenever you're ready. Next up: Module 1, Topic 4 — PL/SQL Data Types (Scalar, Vector, Reference, LOB).*
