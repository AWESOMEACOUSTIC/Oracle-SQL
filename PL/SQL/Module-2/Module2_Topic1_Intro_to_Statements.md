# Module 2, Topic 1: Introduction to PL/SQL Statements and Types of Statements

---

## 1. What This Topic Covers

Every line of executable code inside a PL/SQL block's `BEGIN...END` section is a **statement**. So far, you've mostly written statements that just execute one after another, top to bottom (assignments, `SELECT INTO`, procedure calls). This topic introduces the **formal classification of control flow** in PL/SQL — the categories of statements that let a program do something other than blindly execute line 1, then line 2, then line 3.

This topic is the **conceptual map** for the rest of Module 2 — Selection Statements, CASE, Iteration, and Sequential/Nesting are each a deeper dive into one category introduced here.

---

## 2. Why Does This Classification Exist? What Problem Does It Solve?

Recall from Module 1, Topic 1 why PL/SQL exists at all: SQL alone has no way to express "if this, then that," or "do this repeatedly," or "do these steps in this specific order with dependencies between them." **Control flow statements are exactly what gives PL/SQL this power** — and they fall into a small number of well-understood categories that appear, in some form, in essentially every procedural programming language ever designed. Understanding *why* these categories exist (not just their syntax) is what lets you recognize, from a business requirement's plain-English wording, which category you need.

---

## 3. The Three Fundamental Categories of Control Flow

Every control-flow need in a program boils down to one (or a combination) of these three categories:

| Category | Question It Answers | PL/SQL Mechanism |
|---|---|---|
| **Sequential** | "In what order should these steps run?" | Default top-to-bottom execution; explicitly controllable via `GOTO` (rare) and block nesting |
| **Selection (Conditional)** | "Given a condition, which path should execution take?" | `IF...THEN...ELSE`, `CASE` |
| **Iteration (Looping)** | "How many times, or under what repeating condition, should this run?" | `LOOP`, `WHILE LOOP`, `FOR LOOP` |

This is the entire "toolbox" of control flow. Every real-world program — no matter how complex — is built by combining these three categories, layered and nested as needed.

---

## 4. Sequential Statements (Preview — Fully Covered Later in This Module)

By **default**, PL/SQL executes statements **in the exact order they're written**, top to bottom, one at a time. This is "sequential" execution — the default behavior you've been relying on the entire time without needing to think about it.

```sql
BEGIN
    DBMS_OUTPUT.PUT_LINE('Step 1');  -- runs first
    DBMS_OUTPUT.PUT_LINE('Step 2');  -- runs second
    DBMS_OUTPUT.PUT_LINE('Step 3');  -- runs third
END;
/
```

This seems almost too obvious to call out — but it's worth naming explicitly, because **selection and iteration statements exist specifically to let you deviate from pure sequential execution** when the business logic demands it. Sequential execution is the "default state"; selection and iteration are the tools you reach for when the default isn't enough.

*(The formal "Sequential Statement and Nesting of Blocks" topic, later in this module, goes deeper into this — including the `GOTO` statement, and revisits block nesting from Module 1, Topic 2 in the context of control flow specifically.)*

---

## 5. Selection Statements (Preview)

**Selection** statements let your program choose **one path among several possible paths**, based on a condition. You've already seen small previews of this in Modules 3 and 4 (`IF v_tier = 'GOLD' THEN...`), used just enough to make certain examples realistic, but never formally taught.

```sql
IF v_salary > 100000 THEN
    DBMS_OUTPUT.PUT_LINE('High earner');
ELSE
    DBMS_OUTPUT.PUT_LINE('Standard earner');
END IF;
```

There are multiple **forms** of selection (`IF`, `IF...ELSE`, `IF...ELSIF...ELSE`, `CASE`) — each suited to different shapes of decision-making, covered in depth in the next two topics of this module.

---

## 6. Iteration Statements (Preview)

**Iteration** statements let your program **repeat** a block of logic — either a fixed number of times, or until/while some condition holds. This is essential for row-by-row processing, batch operations, and any "for each..." style business requirement.

```sql
FOR i IN 1..5 LOOP
    DBMS_OUTPUT.PUT_LINE('Iteration: ' || i);
END LOOP;
```

There are multiple **forms** of iteration (basic `LOOP`, `WHILE LOOP`, `FOR LOOP`), each suited to different scenarios — covered in depth later in this module.

---

## 7. Other Statement "Types" Worth Naming Here

Beyond the three big control-flow categories, it's worth formally naming a few other statement *types* you've already been using, so the full landscape is clear:

| Statement Type | Example | What It Does |
|---|---|---|
| **Assignment statement** | `v_x := 10;` | Assigns a value to a variable — you've used this constantly. |
| **NULL statement** | `NULL;` | A genuine, deliberate "do nothing" placeholder statement — seen back in Module 1, Topic 1's minimal block example. Useful when a branch of logic (e.g., in an `IF` or `CASE`) is *intentionally* meant to do nothing, making that intent explicit rather than just leaving a branch empty (which isn't even syntactically legal in some constructs). |
| **Procedure/function call statement** | `apply_discount(101, 15);` | Invokes a stored sub program. |
| **SQL statement** | `SELECT...INTO`, `INSERT`, `UPDATE`, `DELETE` | As covered in Module 1, Topic 2. |
| **Exception-related statement** | `RAISE exception_name;`, `RAISE_APPLICATION_ERROR(...)` | As covered throughout Module 4. |
| **Exit/loop-control statement** | `EXIT`, `EXIT WHEN`, `CONTINUE` | Used specifically within loops to break out early or skip an iteration — formally covered in the Iteration Statements topic. |

---

## 8. Detailed Explanation — How These Categories Combine in Real Programs

A real, non-trivial PL/SQL program almost never uses just one category in isolation. Consider this shape (illustrative, using syntax formally covered later in this module):

```sql
BEGIN
    FOR emp_rec IN (SELECT employee_id, salary FROM employees WHERE department_id = 10) LOOP  -- ITERATION
        IF emp_rec.salary < 40000 THEN                                                          -- SELECTION
            UPDATE employees SET salary = salary * 1.1 WHERE employee_id = emp_rec.employee_id; -- SEQUENTIAL (a DML step)
        END IF;
    END LOOP;
END;
/
```

Here, **iteration** (looping through employees) **contains** a **selection** (checking if the salary is low enough to qualify), which **contains** a **sequential** DML step (the actual update). This nesting of categories — loops containing conditionals containing simple statements — is the fundamental pattern behind almost all real business logic you'll ever write. Recognizing "this part of the requirement needs a loop, and inside that loop, a decision, and inside that decision, an action" is precisely the decomposition skill this entire module builds toward.

---

## 9. When to Use Which Category — Recognition Guide (Full Preview)

| Requirement Language | Category |
|---|---|
| "...**for each** record/row/employee..." | Iteration |
| "...**repeat** until..." | Iteration |
| "...**if** X **then** Y, **otherwise** Z..." | Selection |
| "...**depending on** the value of..., **do** different things..." | Selection (often CASE) |
| "...**first do A, then do B, then do C**..." | Sequential |
| "...**as a placeholder**, do nothing for now..." | NULL statement |

---

## 10. Common Mistakes & Misconceptions

1. **Misconception**: "Sequential execution is a 'basic' concept not worth naming formally." → It's worth naming precisely *because* selection and iteration are meaningful specifically as **deviations** from it — understanding the default makes the exceptions clearer.
2. **Mistake**: Reaching immediately for a loop when a plain sequential list of steps (no repetition needed) would do — over-engineering control flow where none is needed.
3. **Mistake**: Trying to use a bare empty branch (e.g., an empty `IF...THEN...END IF;` with genuinely nothing between `THEN` and `END IF`) — some constructs require **something** in a branch; `NULL;` exists specifically to satisfy this while making "intentionally does nothing" explicit and readable, rather than looking like an accidental omission.
4. **Misconception**: "These three categories are mutually exclusive — a piece of code is either 'iteration' or 'selection,' not both." → Real code **combines and nests** these categories constantly, as shown in Section 8 above.

---

## 11. Interview-Level / Practical Notes

- *"What are the three fundamental categories of control flow in any procedural language, PL/SQL included?"* — Sequential, Selection, Iteration — a foundational computer-science framing question that applies far beyond just PL/SQL, and demonstrates conceptual (not just syntax-level) understanding.
- *"What's the purpose of the NULL statement, and why not just leave a branch empty?"* — Some PL/SQL constructs require at least one statement in a branch; `NULL;` explicitly signals "this is intentional," aiding readability and preventing a reviewer from wondering if code was accidentally left out.
- Being able to **read a business requirement and immediately identify which category (or combination) it needs** — before writing a single line of code — is exactly the skill real interviews and real code reviews probe for, far more than syntax recall.

---

## Things You Must Remember

- All control flow reduces to three categories: **Sequential** (default order), **Selection** (conditional branching), **Iteration** (repetition).
- Sequential execution is the **default** — selection and iteration are deliberate tools you reach for specifically when default top-to-bottom execution isn't sufficient.
- Real programs **nest** these categories together constantly (a loop containing a conditional containing a sequential action is an extremely common shape).
- The `NULL;` statement is a deliberate, readable "do nothing" placeholder — not a sign of unfinished code.
- This topic is a **map**, not a deep dive — Selection, CASE, Iteration, and Sequential/Nesting each get their own full treatment next in this module.

## How to Recognize This Concept

This entire topic **is** a recognition framework — practice reading any business requirement and immediately sorting its needs into: "this part needs to repeat" (iteration), "this part needs to branch based on a condition" (selection), and "this part is just a fixed sequence of steps" (sequential). Getting fast and confident at this sorting, before writing any code, is the single biggest practical skill this topic aims to build — everything else in Module 2 is just filling in the syntax for each category.

---

## Exercises

These are intentionally **conceptual/classification exercises**, not syntax-heavy — full syntax practice comes in the next few topics.

1. **(Classify the need)** For each of the following business requirement fragments, state which control-flow category (or combination) it needs:
   - a) "For every order placed today, check if it qualifies for free shipping."
   - b) "First validate the input, then calculate the total, then save the record."
   - c) "Keep retrying the connection until it succeeds or 5 attempts have been made."
   - d) "If the customer is a VIP, apply a 20% discount; otherwise, apply the standard 5% discount."
   - e) "Depending on the order's status code (1, 2, or 3), route it to a different processing queue."

2. **(Identify the nesting)** Read this requirement: *"For each employee in the Sales department, if their year-to-date sales exceed their quota, calculate and apply a commission bonus."* Identify which category is the **outermost** structure, and which is **nested inside** it.

3. **(NULL statement judgment)** A developer writes an `IF` structure where, in one branch, absolutely nothing should happen — the business rule is "if the customer is already marked inactive, do nothing; otherwise, deactivate them." Explain why using `NULL;` for the "do nothing" branch is better practice than trying to leave the branch empty or awkwardly restructuring the logic to avoid having an empty branch at all.

4. **(Real-world decomposition)** Business requirement: *"Every night, review all pending orders. For each one, if it's been pending for more than 3 days, escalate it to a supervisor queue; if it's been pending for more than 7 days, additionally send a cancellation notice. Keep processing until all pending orders have been reviewed."* Break this down explicitly into its sequential, selection, and iteration components, without writing actual code yet — just identify the structure, similar to how Section 8's example was broken down.

---

*Share your answers whenever you're ready. Next up: Module 2, Topic 2 — Selection Statements with Lend a Hand, where IF/ELSE gets its full, formal treatment.*
