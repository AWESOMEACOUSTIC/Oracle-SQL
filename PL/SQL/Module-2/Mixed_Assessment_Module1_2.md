# Mixed Assessment: Modules 1 & 2 (Fundamentals, Data Types, Operators, Control Flow)

> This assessment covers **everything from Module 1** (Block Structure, Building Elements, Data Types, Operators & Variables) **and Module 2** (Selection, CASE, Iteration, Sequential/Nesting). Same format as your Module 3 & 4 assessment: questions are **not labeled** with the concept being tested, and are mixed across design/build, prediction, debugging, judgment, and a full realistic case study.
>
> Take your time — work through it all at once or section by section.

---

## Section A — Design & Build

### A1.
A retail company needs a diagnostic script that fetches one product's complete row (assume `products(product_id, product_name, price, stock_qty)`) for `product_id = 305`, and prints a line like: `"Wireless Mouse | Price: 899.5 | In Stock: 42"`. Use the most maintainable approach for capturing the row, so the script keeps working automatically even if a column is added to the table later.

---

### A2.
Design a block that classifies a customer's `loyalty_points` balance into a tier for display on their account page: `0` points is `'NEW MEMBER'`, `1–499` is `'BRONZE'`, `500–1999` is `'SILVER'`, `2000` and above is `'GOLD'`. The balance is stored as a `NUMBER` and could, in rare data-quality cases, be `NULL` for accounts migrated from an old system — this must be handled as its own distinct, clearly-labeled case, not silently grouped with `'NEW MEMBER'`.

---

### A3.
Write a script that processes a fixed batch of invoice IDs, 4001 through 4015 (assume `invoices(invoice_id, amount, due_date, paid_flag)`). For each invoice still unpaid (`paid_flag = 'N'`): if it's more than 30 days past its due date, print an "OVERDUE - ESCALATE" message; otherwise print a normal "PENDING" message. If a particular invoice ID doesn't exist in the table, this individual problem must not stop the rest of the batch from being processed.

---

## Section B — Predict the Behavior

### B1.
Without running it, predict exactly what this prints, line by line:
```sql
DECLARE
    v_score NUMBER := NULL;
BEGIN
    IF v_score > 50 THEN
        DBMS_OUTPUT.PUT_LINE('Above 50');
    ELSE
        DBMS_OUTPUT.PUT_LINE('50 or below');
    END IF;

    DBMS_OUTPUT.PUT_LINE('Total (with bonus): ' || (v_score + 10));
END;
/
```

### B2.
Predict the output, and explain precisely why, referencing the specific rule involved:
```sql
DECLARE
    v_code NUMBER := 4;
    v_label VARCHAR2(20);
BEGIN
    CASE v_code
        WHEN 1 THEN v_label := 'One';
        WHEN 2 THEN v_label := 'Two';
        WHEN 3 THEN v_label := 'Three';
    END CASE;

    DBMS_OUTPUT.PUT_LINE(v_label);
END;
/
```

### B3.
Predict how many times the loop body executes, and what (if anything) gets printed:
```sql
DECLARE
    v_remaining NUMBER := 0;
BEGIN
    WHILE v_remaining > 0 LOOP
        DBMS_OUTPUT.PUT_LINE('Processing... remaining: ' || v_remaining);
        v_remaining := v_remaining - 1;
    END LOOP;

    DBMS_OUTPUT.PUT_LINE('Loop finished.');
END;
/
```

### B4.
Predict the output of this nested-block scenario, and explain what happens to `v_inner` across iterations:
```sql
BEGIN
    FOR i IN 1..3 LOOP
        DECLARE
            v_inner NUMBER := 0;
        BEGIN
            v_inner := v_inner + i;
            DBMS_OUTPUT.PUT_LINE('i=' || i || ', v_inner=' || v_inner);
        END;
    END LOOP;
END;
/
```

---

## Section C — Debug It

### C1.
This block is meant to print `'Weekend'` for day numbers 6 and 7, and the day name otherwise, but it doesn't compile. Fix it.
```sql
DECLARE
    v_day NUMBER := 6;
    v_result VARCHAR2(10);
BEGIN
    IF v_day = 6 THEN
        v_result := 'Weekend';
    ELSEIF v_day = 7 THEN
        v_result := 'Weekend';
    ELSE
        v_result := 'Weekday';
    END IF;

    DBMS_OUTPUT.PUT_LINE(v_result);
END;
/
```

### C2.
This function is supposed to classify a batch size, but occasionally raises a runtime error your team can't explain from the logic alone. Identify the bug and fix it.
```sql
CREATE OR REPLACE FUNCTION classify_batch (p_size NUMBER)
    RETURN VARCHAR2
IS
    v_result VARCHAR2(20);
BEGIN
    CASE
        WHEN p_size > 1000 THEN v_result := 'LARGE';
        WHEN p_size > 100 THEN v_result := 'MEDIUM';
        WHEN p_size > 0 THEN v_result := 'SMALL';
    END CASE;

    RETURN v_result;
END classify_batch;
/
```

### C3.
A developer wanted a loop that counts down from 10 to 1, but it never prints anything. Identify the bug and fix it.
```sql
BEGIN
    FOR i IN REVERSE 10..1 LOOP
        DBMS_OUTPUT.PUT_LINE(i);
    END LOOP;
END;
/
```

### C4.
This block is meant to declare a variable that always matches the `employees.salary` column's exact datatype, but it doesn't compile as written. Identify and fix the issue.
```sql
DECLARE
    v_salary employees%TYPE;
BEGIN
    NULL;
END;
/
```

---

## Section D — Judgment & Reasoning (Short Answer)

### D1.
Explain, in your own words, why `IF v_x = NULL THEN` will never behave the way a beginner might expect, and what the correct alternative is. Then explain why the exact same underlying rule also affects how a `CASE` statement handles a `NULL` selector value.

### D2.
A teammate says: *"I always use a basic `LOOP` for everything, even simple counted iterations, because it's the most flexible loop form — I can make it do anything."* Respond in 4–5 sentences: what's true about their claim, and what real risk are they introducing by defaulting to this instead of `FOR` when the iteration count is actually known in advance?

### D3.
Explain the practical difference between declaring `v_total NUMBER(8,2);` versus `v_total orders.total_amount%TYPE;` for a variable meant to hold a value pulled from that column. Under what future circumstance would the difference actually matter in production?

### D4.
A nested block inside a `FOR` loop declares and initializes a local variable on every iteration. A colleague asks: *"Doesn't that waste time re-declaring the same variable over and over? Shouldn't we declare it once outside the loop instead?"* Explain what would actually change in behavior (not just performance) if that variable were moved to the outer, enclosing block instead of staying inside the loop body — is it really just a performance question?

---

## Section E — Realistic Business Case (Multi-Part)

### E1.
A telecom company has this requirement:

> "Every night, we process a batch of prepaid account top-ups, account IDs 7000 through 7025 (assume `prepaid_accounts(account_id, balance, plan_type, last_topup_date)`). For each account: determine the top-up bonus percentage based on `plan_type` — 'PREMIUM' gets 20%, 'STANDARD' gets 10%, 'BASIC' gets 5%, and any plan type we don't recognize should be clearly flagged rather than silently given zero bonus. Apply the bonus to whatever top-up amount is provided (assume a fixed test amount of 100 for every account in this batch, for simplicity). If an account's `plan_type` is missing entirely (NULL), this is a distinct, separately-flagged situation from an unrecognized plan type — one means bad/incomplete data, the other means a genuinely new plan code we haven't accounted for yet. If a specific account ID doesn't exist in the table at all, log it and continue to the next account without halting the batch. After processing, update each account's balance with the bonus-adjusted top-up amount."

**Your task:**
1. Identify every distinct concept from Modules 1 and 2 relevant here, and explain *where* in the requirement each applies (don't just list module topics — tie each one to specific wording).
2. Decide `IF`/`ELSIF` vs. `CASE` for the plan-type-to-bonus mapping, and justify your choice given the "unrecognized plan type" and "missing plan type" requirements specifically.
3. Design the datatype choices you'd use for key variables, including whether `%TYPE`/`%ROWTYPE` applies anywhere.
4. Write the full solution.
5. In a short paragraph, explain how your design ensures one bad account doesn't stop the whole batch, and how it distinguishes the two different "something's wrong with this account's plan data" cases from each other.

---

*Share your answers whenever you're ready — all sections at once or section by section. I'll review each one, and then we can talk about whether you're ready for the full syllabus-spanning company-level final challenge.*
