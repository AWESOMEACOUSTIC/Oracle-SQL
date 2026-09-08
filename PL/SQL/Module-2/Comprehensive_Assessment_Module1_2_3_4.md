# Comprehensive Mixed Assessment: Modules 1–4 (Full Syllabus)

> This is your **full company-level consolidation assessment** — every topic from your entire syllabus is fair game: block structure, datatypes, `%TYPE`/`%ROWTYPE`, operators/NULL-handling, selection (`IF`/`CASE`), iteration, sequential/nesting, procedures, functions, result cache, notation, packages, overloading, and all three exception-handling topics.
>
> As requested: **8 realistic, scenario-based business case studies** in Section A (the core of this assessment), followed by a **Quick-Fire round** (Section B: predict/debug) and a **Judgment & Reasoning round** (Section C) to round out your practice.
>
> Nothing is labeled by concept. Some scenarios are deliberately a little open-ended or ambiguous — real requirements aren't always perfectly specified, and part of the skill being tested is making (and stating) a reasonable assumption where needed, rather than stalling.

---

## Section A — Scenario-Based Business Case Studies

For each, a useful (not mandatory) structure to follow: **What's being asked → Key clues in the wording → Concepts you think are relevant → Design decisions (public/private, procedure/function, IF vs CASE, loop form, exception strategy) → Solution → Edge cases → What you'd explain/defend in a review.**

---

### Scenario 1 — Warehouse Restocking Alert System
*"Our warehouse system needs a module to monitor stock levels. Given a product ID, determine its restock urgency: if current stock is at or below 10% of its maximum capacity, urgency is 'CRITICAL'; between 10% and 30%, 'LOW'; above 30%, 'OK'. This calculation is used constantly across many dashboards and rarely changes for a given product on any given day, since stock levels only update a handful of times daily via batch jobs, not in real time. Separately, we need a way to actually process a restock — given a product ID and a quantity received, increase the stock and reject the operation outright (with a clear, distinct error) if the received quantity is zero or negative. Products that don't exist at all should be handled distinctly from invalid quantities."*

---

### Scenario 2 — Customer Support Ticket Routing
*"Support tickets come in with a numeric category code (1 through 7, but new categories get added occasionally before our routing logic is updated). Route each ticket to a queue name based on its category. Any code we don't yet recognize must be routed to a 'GENERAL' queue rather than causing the routing script to fail — this needs to be safe against category codes we haven't seen yet, by design, not as an afterthought."*

---

### Scenario 3 — Payroll Batch Run With Mixed Outcomes
*"Process employee IDs 3000 through 3050 for this month's payroll run (assume `employees(employee_id, base_salary, department_id, employment_status)`). For each: if `employment_status` is not 'ACTIVE', skip them entirely with a log note — this is a normal, expected situation, not an error. For active employees, calculate a department bonus: department 10 (Sales) gets 8%, department 20 (Engineering) gets 5%, all others get 2%. If an employee ID in that range doesn't exist in the table at all, that's a genuine data problem distinct from 'not active,' and should be logged differently. The batch must complete for all valid employees even if some IDs are problematic."*

---

### Scenario 4 — Multi-Currency Order Total (One-Off Executive Report)
*"For a one-time board presentation next week, we need a report listing every order from Q3 with its total converted to USD, using a simple internal calculation involving each order's currency code. This is genuinely a one-off — after this presentation, this exact calculation will likely never be needed again in this form. We don't want to leave a permanent function cluttering the schema for something this disposable."*

---

### Scenario 5 — Membership Renewal Engine
*"Design our membership renewal module: given a member ID, determine their renewal price using their membership level (lookup logic that's purely an internal implementation detail, not something other systems should call directly) and process the actual renewal (updating their expiry date and inserting a renewal record). If a member's account is flagged 'DELINQUENT', renewal must be rejected outright with an error that both our internal billing scripts and an external payment partner's integration can reliably detect and act on — they need to distinguish this specific rejection reason from any other possible failure. Also provide a way to check whether a given member's renewal is due soon, since this needs to power a notification system that queries many members' data at once."*

---

### Scenario 6 — Duplicate Prevention on Bulk Import
*"We're bulk-importing a list of new supplier records, supplier codes 'SUP-101' through 'SUP-120' (assume `suppliers(supplier_code, supplier_name)`, with `supplier_code` unique). For each code in that range: attempt to insert a placeholder supplier record. Some of these codes may already exist from a previous partial import attempt — these should be logged as 'already exists' and skipped, not treated as a fatal problem, since re-running this import safely is an expected, normal operation."*

---

### Scenario 7 — Tiered API Rate Limit Checker
*"Build a reusable check: given a customer account ID, determine their API rate limit tier and how many requests they have remaining this hour. The rate-limit-tier-to-request-cap mapping (Bronze: 100/hr, Silver: 500/hr, Gold: unlimited) is a fixed, rarely-changing lookup, but this check will be called an enormous number of times per second across our infrastructure — performance here matters enormously. Also, if the account ID doesn't exist, that must be communicated clearly and distinctly to the calling API gateway, which needs to return a proper structured error to the end client rather than crash silently."*

---

### Scenario 8 — End-of-Month Financial Close (The Big One)
*"At month end, we process closing entries for cost centers 1 through 40 (assume `cost_centers(center_id, budget, actual_spend, status)`). For each cost center still 'OPEN': if actual spend exceeds budget by more than 15%, flag it 'OVER_BUDGET_SEVERE' and route to Finance Director review; if it exceeds budget by any amount up to 15%, flag it 'OVER_BUDGET_MINOR' for standard manager review; if under or at budget, mark it 'CLOSED_CLEAN'. Cost centers with a NULL budget (not yet set up properly this cycle) must be flagged as 'BUDGET_NOT_SET' — a distinctly different, non-financial problem from actually overspending. If a cost center ID doesn't exist at all, log it and continue — this must never halt the month-end close process, which finance considers a business-critical, must-complete-every-time operation. Whatever internal calculation determines the over-budget percentage should not be exposed for other systems to call directly, since it's tightly coupled to this specific close process's internal logic."*

**For this one specifically**, also answer: what would you tell a code reviewer about why you separated (or didn't separate) the pieces of this into different objects, and what's your single biggest design risk if this requirement changes next quarter?

---

## Section B — Quick-Fire: Predict & Debug

### B1. Predict the output:
```sql
DECLARE
    v_val NUMBER := NULL;
BEGIN
    IF v_val > 0 THEN
        DBMS_OUTPUT.PUT_LINE('Positive');
    ELSIF v_val <= 0 THEN
        DBMS_OUTPUT.PUT_LINE('Non-positive');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Unknown');
    END IF;
END;
/
```

### B2. Predict the output and explain the underlying rule:
```sql
BEGIN
    DBMS_OUTPUT.PUT_LINE('A' || NULL || 'B');
END;
/
```

### B3. What happens when this runs, and why?
```sql
CREATE OR REPLACE FUNCTION get_rate (p_code VARCHAR2) RETURN NUMBER RESULT_CACHE
IS
    v_rate NUMBER;
BEGIN
    INSERT INTO rate_access_log(code, accessed_on) VALUES (p_code, SYSDATE);
    SELECT rate INTO v_rate FROM rates WHERE code = p_code;
    RETURN v_rate;
END;
/
```

### B4. Debug this — it compiles, but always returns the wrong value for `p_val = 50`:
```sql
CREATE OR REPLACE FUNCTION tier_label (p_val NUMBER) RETURN VARCHAR2
IS
BEGIN
    IF p_val > 0 THEN
        RETURN 'LOW';
    ELSIF p_val > 25 THEN
        RETURN 'MEDIUM';
    ELSIF p_val > 100 THEN
        RETURN 'HIGH';
    END IF;
END;
/
```

### B5. Why does this fail to compile?
```sql
CREATE OR REPLACE PACKAGE pkg_demo
IS
    FUNCTION get_val (p_id NUMBER) RETURN NUMBER;
    FUNCTION get_val (p_id NUMBER) RETURN VARCHAR2;
END pkg_demo;
/
```

### B6. This procedure is called from three different applications, and none of them can figure out why a "successful" call sometimes leaves their local variable unset. What's going on?
```sql
CREATE OR REPLACE PROCEDURE get_status (p_id IN NUMBER, p_status OUT VARCHAR2)
IS
BEGIN
    SELECT status INTO p_status FROM records WHERE record_id = p_id;
END;
/
```

### B7. Predict how many times "Checking..." prints:
```sql
DECLARE
    v_tries NUMBER := 1;
BEGIN
    LOOP
        DBMS_OUTPUT.PUT_LINE('Checking...');
        EXIT WHEN v_tries >= 3;
        v_tries := v_tries + 1;
    END LOOP;
END;
/
```

### B8. Fix the call so it compiles:
```sql
schedule_task(p_task_id => 501, 'HIGH', p_due_date => SYSDATE + 1);
```

---

## Section C — Judgment & Reasoning

### C1.
A junior developer proposes: *"Let's just make every single procedure and function in our new module PUBLIC in the package spec — it's simpler, and we can always tighten it up later if needed."* Give your honest response, including at least one concrete, realistic way this comes back to bite the team later.

### C2.
Explain, in your own words, the difference between how you'd handle "this record doesn't exist" versus "this record exists but violates a business rule" in your exception-handling design — and why treating them identically (e.g., both as generic `WHEN OTHERS`) is a weaker design choice.

### C3.
A performance-tuning review flags that a heavily-used function is a candidate for `RESULT_CACHE`. Before agreeing, what three questions would you want answered about that function first?

### C4.
Explain why a `CASE` statement without an `ELSE` clause is riskier in production than an `IF...ELSIF` chain without a final `ELSE`, even though both "look" like they're missing the same thing.

---

*Take your time with this — it's meant to be worked through over multiple sessions if needed, not rushed. Share your answers whenever you're ready, in whatever order or grouping works for you, and I'll review each one closely.*
