# Module 1, Topic 6: Lend a Hand on PL/SQL (Practice Checkpoint)

> This is a **hands-on consolidation session** for all of Module 1: Block Structure, Building Elements, Data Types, and Operators & Variables. No new theory here — just applied practice, following the same "Lend a Hand" format you've seen in Modules 3 and 4. As before, exercises are **not concept-tagged** — identifying what applies is part of the exercise.

---

## How to Use This Session

Before writing any code for each problem, briefly work through:
1. What datatype(s) does this data naturally call for — and should any of them use `%TYPE`/`%ROWTYPE` instead of a hardcoded type?
2. Does this need nested blocks, or is a single flat block sufficient?
3. Are there any NULL-handling risks lurking in this requirement?
4. Are you using `:=` vs `=` correctly, and literals correctly (quotes, date literals, escaping)?

---

## Practice Problems

### Problem 1 — Employee Snapshot
**Business context:** You need a quick diagnostic block that fetches one employee's complete row (assume `employees(employee_id, first_name, last_name, salary, hire_date)`) for `employee_id = 150`, and prints a formatted line like: `"Arun Kumar | Hired: 2020-03-15 | Salary: 75000"`.

**Task:** Write this using the most maintainable approach for holding the fetched row — think carefully about which datatype tool from Topic 4 avoids hardcoding column types manually.

---

### Problem 2 — Safe Bonus Calculation
**Business context:** A block calculates `v_total_comp := v_salary + v_bonus;`, where `v_bonus` is fetched from a table and may legitimately be `NULL` for employees who don't have a bonus this cycle. The business wants the final total to correctly treat a missing bonus as zero, not silently produce a missing total.

**Task:** Write a block demonstrating the problem *would* occur without any fix, then fix it so the total always comes out correct regardless of whether `v_bonus` is `NULL`. (You are free to research or recall a suitable built-in function for this if you know one, or handle it via a plain conditional check — both are acceptable.)

---

### Problem 3 — Anchored Variable Set
**Business context:** A reporting script needs three variables: one to hold a value matching `products.price`'s exact datatype, one to hold a "discounted price" that should always match the first variable's type exactly (so they're guaranteed compatible for later arithmetic), and one `NOT NULL` constant representing a fixed shipping fee of `9.99` that should never accidentally be left unset.

**Task:** Declare all three variables/constants correctly, using the appropriate tools from Topics 4 and 5.

---

### Problem 4 — Isolated Sub-Calculation
**Business context:** A block needs to calculate a "risk score" as a self-contained sub-step (using its own local working variable that has no meaning outside that calculation), before continuing on to use the final risk score value in the rest of the block's logic. The business wants this sub-calculation's own internal working variable to be completely inaccessible/non-existent once the sub-calculation is done, to avoid any accidental reuse elsewhere in the block.

**Task:** Structure this using the appropriate block-structure concept from Topic 2. Use placeholder/illustrative logic for the actual "risk score" math itself.

---

### Problem 5 — Spot the Flaws
**Business context:** A colleague wrote this diagnostic block and it's not behaving as expected — it never prints the "high value order" message, even for orders that clearly qualify:

```sql
DECLARE
    v_order_total NUMBER := 150000;
    v_customer_note VARCHAR2(50) = 'VIP account';  -- note this line
BEGIN
    IF v_order_total = NULL THEN
        DBMS_OUTPUT.PUT_LINE('Order total is missing.');
    END IF;

    IF v_order_total > 100000 THEN
        DBMS_OUTPUT.PUT_LINE('High value order: ' || v_customer_note);
    END IF;
END;
/
```

**Task:** This block actually has **two separate bugs** — one is a straightforward syntax error that would stop it from compiling at all, and one is a logic issue related to something you learned about NULL handling (even though, in this specific case, it doesn't cause the described symptom — explain why not, too). Find both, explain each clearly, and provide the corrected version.

---

### Problem 6 — Realistic End-to-End Scenario
**Business context:** *"Write a diagnostic script for the finance team that looks up a specific invoice (assume `invoices(invoice_id, customer_name, invoice_amount, due_date, notes)`, where `notes` can be a lengthy free-text field potentially containing paragraphs of context and should not be limited to a short VARCHAR2). The script should print the invoice's core details, and separately, as an isolated internal step, calculate a 'days overdue' figure only if the due date has passed — treating this calculation as self-contained and not needing to affect the rest of the script if something about the calculation itself goes wrong unexpectedly."*

**Task:** This problem deliberately requires you to pull together **several** concepts from across this entire module at once. Identify the right datatype choice for `notes`, the right approach for capturing the invoice's row, and the right block-structuring choice for the isolated calculation step — then write the full script (illustrative logic is fine for the actual overdue-day math, since date arithmetic details aren't the focus here).

---

## Notes Before You Attempt These

- Problem 6 is intentionally the most integrative — it's meant to feel like a small, realistic "here's a script, go build it" request, pulling together datatypes, %ROWTYPE/%TYPE thinking, nesting, and NULL-awareness all at once.
- If you're unsure about any built-in function names (like NULL-handling functions) but understand the *concept* you need, it's completely fine to describe the approach in plain English — we haven't formally named every built-in function in this syllabus, and the reasoning matters more than perfect recall of a function name at this stage.

---

*Share your attempts whenever you're ready — one at a time or all together. Once we're through this, we'll move into Module 2: PL/SQL Statements, starting with "Intro to PL/SQL Statements and Types of Statements."*
