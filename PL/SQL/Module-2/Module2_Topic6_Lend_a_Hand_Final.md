# Module 2, Topic 6: Lend a Hand on PL/SQL (Practice Checkpoint)

> This is the closing hands-on checkpoint for Module 2 — covering Selection (`IF`/`CASE`), Iteration (`LOOP`/`WHILE`/`FOR`), and Sequential/Nesting concepts. As with every "Lend a Hand" session, problems are **not concept-tagged**, and several problems deliberately pull in concepts from earlier modules (datatypes, exceptions, procedures) — because that's exactly how real requirements arrive: as one messy paragraph, not a neatly labeled topic list.
>
> **This also closes out your entire original syllabus.** Once you've worked through this, every topic from your provided syllabus (Modules 1–4) has been covered in full.

---

## How to Use This Session

Before coding each problem:
1. Sort the requirement into its sequential / selection / iteration components (per Module 2, Topic 1's framework).
2. Decide `IF` vs. `CASE` where a decision is involved, and justify it.
3. Decide which loop form fits, if iteration is needed.
4. Identify any NULL-handling risks (Module 1, Topic 5) or exception-handling needs (Module 4) hiding in the requirement.
5. Decide whether any part of the logic deserves its own nested block for scoping/isolation.
6. Only then, write the code.

---

## Practice Problems

### Problem 1 — Shipping Cost Calculator
**Business context:** Calculate shipping cost based on order weight (in kg): under 1kg is 50, 1kg–5kg (inclusive) is 120, 5kg–20kg (inclusive) is 300, anything above is 500. Weight is never expected to be missing, but write defensively anyway.

---

### Problem 2 — Status Code Translator for a Report
**Business context:** A legacy system stores order status as a numeric code (1–6). The reporting team needs each code translated to a display label for a dashboard query — this translation will be embedded directly inside a `SELECT` statement, not a standalone script. New codes might occasionally appear before the mapping table is updated, and these should show clearly as `'Unmapped Status'` rather than breaking the report.

---

### Problem 3 — Batch Salary Adjustment With Resilience
**Business context:** Process employee IDs 200 through 220. For each one, give a 5% raise — but only if their current salary is below 60,000 (employees at or above that threshold are frozen this cycle per company policy). Some employee IDs in that range may not exist in the table at all; this must not stop the batch from completing the rest.

---

### Problem 4 — Retry-Style Processing
**Business context:** A block needs to simulate checking a "batch job status" flag up to a maximum of 5 attempts, printing the attempt number each time, stopping early the moment the status shows `'COMPLETE'` (simulate this by having the status become `'COMPLETE'` on the 3rd attempt), and printing a final timeout message if it never completes within 5 attempts.

---

### Problem 5 — Multi-Level Discount Engine
**Business context:** Design a function `calculate_final_price` for an e-commerce checkout process:
- Base discount by customer tier: GOLD 15%, SILVER 10%, everyone else 0%.
- **Additionally**, if the order amount exceeds 5,000 (before the tier discount is applied), add a further flat 5% on top, regardless of tier.
- The function should refuse to process (raise an appropriately communicated error) if the order amount is zero or negative — this is a genuine business rule violation, not just a missing/unknown value.

**Task:** This problem intentionally requires you to combine a function design decision, a selection strategy (and justify IF vs. CASE here specifically), and proper error communication for external callers — think through all of it before writing code.

---

### Problem 6 — Spot the Flaws (Multiple Bugs)
**Business context:** A colleague wrote this and it's misbehaving in ways they can't explain:

```sql
DECLARE
    v_attempts NUMBER := 0;
    v_status VARCHAR2(20) := NULL;
BEGIN
    WHILE v_status != 'DONE' LOOP
        v_attempts := v_attempts + 1;
        DBMS_OUTPUT.PUT_LINE('Attempt: ' || v_attempts);

        CASE v_attempts
            WHEN 3 THEN v_status := 'DONE';
        END CASE;

        EXIT WHEN v_attempts > 10;
    END LOOP;
END;
/
```

**Task:** There are **at least two distinct, genuine bugs** here — one related to how the loop's condition interacts with a NULL value at the very start, and one related to what happens when `v_attempts` isn't exactly 3 on a given pass through the `CASE`. Identify both precisely (referencing the specific rules from this module that explain each), and provide a corrected version.

---

### Problem 7 — Full Realistic Case Study
**Business context:** *"We're processing a nightly file of loan applications, application IDs 1000 through 1020 (assume a table `loan_applications(app_id, credit_score, requested_amount, status)`). For each application still marked 'PENDING': if the credit score is 750 or above, auto-approve regardless of amount; if between 600 and 749 (inclusive), auto-approve only if the requested amount is 25,000 or less, otherwise flag for manual review; below 600, auto-reject. If an application's credit score is missing entirely, this must be flagged as 'DATA_INCOMPLETE' — a distinctly different outcome from a low-score rejection. If a particular application ID doesn't exist in the table at all, log this and continue to the next ID without stopping the batch. Update each processed application's status accordingly."*

**Task:** This is the most integrative problem in this checkpoint — pulling together iteration, nested selection logic, NULL-awareness, and per-record exception resilience, tying together concepts from every module you've studied. Design your approach first (in writing, briefly), then write the full solution.

---

## A Note on Finishing the Syllabus

Once you've worked through these (even a few, if you'd rather sample this checkpoint lightly), **you will have completed every topic from your original syllabus**, across all four modules. From here, a natural next step — matching the "Level 5: Company-Level Problem Solving" and "Final Company-Level Challenge" goals from your original learning plan — would be a comprehensive, ambiguous, multi-concept final assessment spanning the entire syllabus at once, without any module boundaries to hint at what's relevant. Just let me know when you're ready for that.

---

*Share your attempts whenever you're ready — one at a time or all together.*
