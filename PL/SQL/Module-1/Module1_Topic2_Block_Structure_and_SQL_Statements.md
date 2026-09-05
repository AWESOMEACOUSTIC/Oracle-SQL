# Module 1, Topic 2: Block Structure & Types of SQL Statements

---

## Part A: Block Structure (Formalized)

### 1. What This Topic Covers

In Topic 1, you saw the basic shape of a PL/SQL block (`DECLARE...BEGIN...EXCEPTION...END;`) just enough to write your first programs. This topic formalizes **block structure** properly — including a concept we glossed over: **blocks can be nested inside other blocks**, and understanding *why* and *how* is foundational to everything more advanced you'll build later (including how procedures/functions/packages, which you already learned in Module 3, are themselves fundamentally built from this same block structure).

### 2. Why Block Structure Matters as a Formal Concept

PL/SQL is called a **block-structured language**. This isn't just a description of syntax — it reflects a genuine design philosophy: **every unit of PL/SQL logic, no matter how large or small, is built from the same fundamental shape.** An anonymous block, a procedure body, a function body, a package body's individual member — all of them are fundamentally `DECLARE (optional) → BEGIN → EXCEPTION (optional) → END` at their core. Once you deeply understand this one shape, you understand the skeleton of *everything* in PL/SQL. This is why Topic 1 introduced it first, before anything else.

**Nesting** exists because real logic often has **sub-tasks within a task** — a natural way to isolate a smaller piece of logic (with its own local variables, its own error handling) inside a larger one, without that smaller piece "leaking" its details into the larger surrounding logic.

### 3. Syntax: Nested Blocks

```sql
DECLARE
    v_outer_var NUMBER := 10;
BEGIN
    DBMS_OUTPUT.PUT_LINE('Outer block start. v_outer_var = ' || v_outer_var);

    DECLARE
        v_inner_var NUMBER := 20;
    BEGIN
        DBMS_OUTPUT.PUT_LINE('Inner block. v_outer_var = ' || v_outer_var || ', v_inner_var = ' || v_inner_var);
    EXCEPTION
        WHEN OTHERS THEN
            DBMS_OUTPUT.PUT_LINE('Inner block error handled locally.');
    END;

    DBMS_OUTPUT.PUT_LINE('Back in outer block. v_outer_var = ' || v_outer_var);
    -- v_inner_var is NOT accessible here — it no longer exists once the inner block ended
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Outer block error handled.');
END;
/
```

### 4. Key Rules of Nesting (Scope)

- An **inner block can see and use variables declared in any enclosing outer block** — this is called **lexical scoping**. In the example above, the inner block freely reads `v_outer_var`.
- An **outer block cannot see variables declared inside an inner block**. Once the inner block's `END;` is reached, `v_inner_var` ceases to exist entirely — trying to reference it from the outer block afterward would be a compile error.
- Each nested block has its **own, independent `EXCEPTION` section**. An error inside the inner block, if handled by the inner block's own handler, is considered fully resolved **at that inner level** — the outer block's code continues normally afterward, completely unaware anything went wrong, *unless* the inner block's exception is left unhandled (or deliberately re-raised), in which case it propagates out to the outer block exactly as described in Module 4, Topic 1.
- If an **inner variable has the same name as an outer variable**, the inner one **takes precedence** within the inner block (this is called "shadowing") — referencing that name inside the inner block refers to the inner declaration, not the outer one. This is legal but can be genuinely confusing to read, and is generally best avoided in real code for clarity.

### 5. Why This Connects to What You Already Know

Recall from Module 3: a **procedure body**, once you strip away the `CREATE OR REPLACE PROCEDURE name (...) IS` header, is *exactly* this same `BEGIN...EXCEPTION...END;` shape. When a procedure **calls another procedure**, that's conceptually very similar to nesting — the called procedure runs its own independent block, with its own local variables and its own exception handling, isolated from the caller's scope, exactly like a nested anonymous block is isolated from its outer block. Understanding nesting here gives you the mental model for understanding call isolation everywhere else in PL/SQL.

---

## Part B: Types of SQL Statements (Usable Inside PL/SQL)

### 6. What This Covers

PL/SQL isn't a replacement for SQL — it's a wrapper *around* SQL, as established in Topic 1. This section formalizes exactly **which categories of SQL statements** you can use inside a PL/SQL block, and how each category behaves differently once inside PL/SQL versus how you might be used to running them standalone.

### 7. The Categories

| Category | Full Name | Examples | Purpose |
|---|---|---|---|
| **DML** | Data Manipulation Language | `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `SELECT` | Reading and modifying **data** (rows) |
| **TCL** | Transaction Control Language | `COMMIT`, `ROLLBACK`, `SAVEPOINT` | Controlling whether DML changes are made permanent or undone |
| **DDL** | Data Definition Language | `CREATE`, `ALTER`, `DROP` | Defining/changing **database objects** (tables, procedures, etc.) |
| **DCL** | Data Control Language | `GRANT`, `REVOKE` | Controlling **permissions/access** |

### 8. How Each Behaves Inside PL/SQL

- **DML** (`INSERT`/`UPDATE`/`DELETE`/`MERGE`) can be used **directly and freely** inside any PL/SQL block, exactly as you'd expect — this is one of PL/SQL's core purposes, as established in Topic 1.
- **`SELECT`** can be used, but (as you learned in Topic 1) **only** in the `SELECT ... INTO variable` form when used as a standalone statement inside a block — a bare `SELECT` with no `INTO` is not valid PL/SQL syntax on its own (it's only valid in specific contexts like a cursor definition — outside our syllabus — or nested inside another SQL statement).
- **TCL** (`COMMIT`/`ROLLBACK`/`SAVEPOINT`) can be used directly inside PL/SQL blocks to control the fate of DML changes made within that block or session. *(Full transaction control mechanics are a deep topic on their own and sit outside our current syllabus scope — what matters here is simply knowing these statements are valid inside PL/SQL, not their complete behavior.)*
- **DDL** (`CREATE`/`ALTER`/`DROP`) **cannot** be used directly inside a PL/SQL block the way DML can. Attempting `CREATE TABLE ...;` directly inside a `BEGIN...END` block raises a compile error (`PLS-00103` or similar). To execute DDL from within PL/SQL, you must use **Native Dynamic SQL** (`EXECUTE IMMEDIATE`) — a genuinely more advanced technique that sits **outside our current syllabus**, so we won't build it here, but it's important to know *why* plain DDL doesn't just work inline: DDL statements have fundamentally different compilation and transaction behavior (they perform an implicit commit, for one) that doesn't fit PL/SQL's normal static-SQL compilation model the way DML does.
- **DCL** (`GRANT`/`REVOKE`) has the same restriction as DDL — not usable directly inline; also requires dynamic SQL if truly needed from within PL/SQL.

### 9. Detailed Explanation — Why the DML/DDL Distinction Exists

PL/SQL **pre-compiles and validates** the SQL statements inside a block at compile time — checking that referenced tables/columns exist, that datatypes are compatible, and so on, exactly like it validates your PL/SQL syntax itself. This is called **static SQL**. DML statements fit naturally into this model, because they operate on a fixed, known table/column structure that PL/SQL can check ahead of time.

DDL statements, by contrast, are fundamentally about **changing that very structure** — and DDL statements also **implicitly commit** any pending transaction the moment they run, which is a significant, different behavior from DML. Allowing arbitrary DDL to be typed directly and statically inside a compiled PL/SQL block would undermine the predictability that static SQL compilation is designed to provide. This is why DDL requires the explicit, deliberate mechanism of dynamic SQL instead — it's a genuine design boundary, not an arbitrary limitation.

### 10. Common Mistakes & Misconceptions

1. **Mistake**: Writing a bare `SELECT column FROM table;` (no `INTO`) directly inside a PL/SQL block, expecting it to behave like it would in a SQL client → compile error.
2. **Mistake**: Trying to write `CREATE TABLE temp_results (...)` directly inside a `BEGIN...END` block → compile error; this needs dynamic SQL (a topic outside our current syllabus).
3. **Misconception**: "Nested blocks share all their variables both ways." → False — only **outer-to-inner** visibility exists; inner declarations are invisible once you're back in the outer scope.
4. **Misconception**: "An error in a nested block always crashes the whole outer block." → False — if the **inner block has its own EXCEPTION handler** that catches it, the outer block continues completely normally afterward, unaffected.
5. **Mistake**: Deliberately naming an inner variable the same as an outer one "to save time thinking of a new name," creating confusing shadowing — legal, but a real readability hazard in production code.

### 11. Edge Cases to Be Aware Of

- A nested block, if it has **no** `EXCEPTION` section of its own, behaves exactly like Module 4, Topic 1 describes for any block: an unhandled exception inside it simply **propagates outward** to the next enclosing block (or further, if that one also lacks a handler) — nesting doesn't change the fundamental propagation rule, it just adds more "levels" the exception might travel through before being caught.
- Since procedures/functions are themselves block-structured, calling a procedure from inside a block is *architecturally* similar to nesting, but with one key difference: a called procedure is a **separate, independently compiled and stored object** with its own name, not an anonymous block written inline — the isolation principle (separate scope, separate exception handling) is shared, but the packaging/reusability is not.

### 12. Interview-Level / Practical Notes

- *"Can you run a CREATE TABLE statement inside a PL/SQL block?"* — Not directly; this requires dynamic SQL (`EXECUTE IMMEDIATE`), and being able to explain *why* (implicit commit, static SQL compilation model) demonstrates real understanding, not just memorized rules.
- *"What happens to an outer block's execution if a nested block's exception is fully handled inside that nested block?"* — The outer block continues completely normally, as if nothing happened, immediately after the nested block's `END;`.
- *"Can an outer block reference a variable declared in a nested inner block after that inner block ends?"* — No — this is a very standard scoping question, testing whether you genuinely understand lexical scoping versus just having memorized "nesting is allowed."

---

## Things You Must Remember

- PL/SQL blocks can be **nested**: inner blocks see outer variables (lexical scoping), but outer blocks never see inner variables, and inner variables cease to exist once the inner block ends.
- Each nested block can have its **own independent EXCEPTION section** — an error handled at the inner level does not disturb the outer block at all.
- **DML** (`INSERT`/`UPDATE`/`DELETE`/`MERGE`) and `SELECT...INTO` work directly inside PL/SQL. A bare `SELECT` without `INTO` does not.
- **TCL** (`COMMIT`/`ROLLBACK`/`SAVEPOINT`) works directly inside PL/SQL.
- **DDL** (`CREATE`/`ALTER`/`DROP`) and **DCL** (`GRANT`/`REVOKE`) **cannot** be used directly inline — they require dynamic SQL (`EXECUTE IMMEDIATE`), a topic outside our current syllabus, due to their implicit-commit behavior and incompatibility with PL/SQL's static SQL compilation model.
- Variable name shadowing (same name in inner and outer blocks) is legal but a real readability hazard — avoid it in practice.

## How to Recognize This Concept

Think about **nested blocks** when a requirement describes a task that has a clearly **separable sub-step** — especially one where you'd want **localized error handling** that doesn't disturb the rest of the surrounding logic ("if this specific small piece fails, handle it right there and keep going with everything else").

Think about the **DML vs. DDL distinction** whenever a requirement mentions **changing database structure** (creating a table, altering a column) as part of a PL/SQL process — that's your signal this isn't a simple inline statement and would need dynamic SQL, which is worth flagging as a prerequisite concept beyond our current syllabus if it ever comes up in a real exercise.

---

## Exercises

1. **(Basic nesting)** Write an outer block that declares a variable holding a company name, and a nested inner block that declares its own variable holding a department name, printing both together inside the inner block.

2. **(Scope violation, predicted)** Explain why this code fails to compile, referencing scoping rules directly:
   ```sql
   BEGIN
       DECLARE
           v_temp NUMBER := 5;
       BEGIN
           NULL;
       END;
       DBMS_OUTPUT.PUT_LINE(v_temp);  -- attempted here, outside the inner block
   END;
   /
   ```

3. **(Localized error handling)** Write an outer block that processes two independent "steps" — each step wrapped in its own nested block with its own exception handling, such that if Step 1 fails (simulate this with a deliberate division by zero), Step 2 still runs normally afterward. Print clear messages showing this happened.

4. **(Shadowing)** Write a block demonstrating variable shadowing: declare a variable `v_status` in an outer block set to `'OUTER'`, then declare another `v_status` in a nested inner block set to `'INNER'`, and print `v_status` from inside the inner block, then again from the outer block after the inner block ends. Explain the output.

5. **(DML vs DDL judgment)** For each of the following, state whether it can be written directly inline inside a PL/SQL block, or would require dynamic SQL, and why:
   - a) `UPDATE employees SET salary = salary * 1.05;`
   - b) `CREATE INDEX idx_emp_name ON employees(last_name);`
   - c) `DELETE FROM audit_log WHERE log_date < SYSDATE - 365;`
   - d) `ALTER TABLE orders ADD (priority_flag VARCHAR2(1));`

6. **(Realistic business scenario)** A requirement states: *"When processing a monthly report, first calculate departmental totals (a self-contained calculation step), then separately calculate a company-wide bonus pool figure (a second, independent calculation step). If the departmental totals step encounters a problem, it should be logged and the bonus pool calculation should still proceed regardless."* Describe (in prose or code) how nested blocks with independent exception handling would be an appropriate structural choice here, and why a single flat block with one shared exception handler would not achieve the same outcome.

---

*Share your answers whenever you're ready. Next up: Module 1, Topic 3 — PL/SQL Building Elements (Identifiers, Literals, Semicolon Delimiter, Comments).*
