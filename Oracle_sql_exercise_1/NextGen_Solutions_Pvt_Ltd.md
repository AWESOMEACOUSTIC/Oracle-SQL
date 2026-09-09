# NextGen Solutions Pvt Ltd — Employee Workforce Management System
### An Oracle SQL Hands-On Case Study (Data Types → DDL/DML/DQL/DCL/TCL → Data Integrity → Operators)

---

## How to Use This Guide

You are the **database developer** on a project for **NextGen Solutions Pvt Ltd**, a mid-size IT/HR consulting firm. The client's HR team has been tracking employees in spreadsheets and wants a proper Oracle database table to manage their workforce.

Everything in this guide happens on **one single table**. It starts empty, gets designed, gets built, gets populated, gets fixed, gets constrained, and finally gets queried and reported on — exactly the way a real client engagement unfolds. Concepts are introduced **topic by topic first** (so you can revise each in isolation), and the final section deliberately **mixes everything together** the way real business questions do.

Each of the 30 questions follows this format, as requested:

1. **Business Question** — the scenario, phrased the way a client or PM would ask it
2. **Expected Output** — what you should see if you run the query correctly
3. **Answer** — the SQL statement plus a short explanation

Run these in Oracle (Live SQL, SQL*Plus, SQL Developer, or an Autonomous DB) in order — the data at Question 20 depends on everything that happened from Question 9 onward, so skipping around will give you different numbers than shown here.

---

## Table of Contents

- [The Client Brief](#the-client-brief)
- [Part 1 — SQL Data Types](#part-1--sql-data-types-designing-the-table) (Q1–Q2)
- [Part 2 — Data Definition Language (DDL)](#part-2--data-definition-language-ddl) (Q3–Q8)
- [Part 3 — Data Manipulation Language (DML)](#part-3--data-manipulation-language-dml) (Q9–Q13)
- [Part 4 — Data Query Language (DQL)](#part-4--data-query-language-dql) (Q14–Q17)
- [Part 5 — Data Control Language (DCL)](#part-5--data-control-language-dcl) (Q18–Q19)
- [Part 6 — Transaction Control Language (TCL)](#part-6--transaction-control-language-tcl) (Q20–Q22)
- [Part 7 — Data Integrity & Constraints](#part-7--data-integrity--constraints) (Q23–Q27)
- [Part 8 — SQL Operators (mixed, capstone-style)](#part-8--sql-operators) (Q28–Q30)
- [Final Schema Snapshot & Answer Key](#final-schema-snapshot--answer-key)

---

## The Client Brief

> "We're a 100-person consulting firm and we're done managing people in Excel. We need a proper employee table — nothing fancy, no multi-table setup yet, just **one solid table** that tracks who works for us, what they earn, who they report to, and their status. We'll want to run reports on department costs, give raises, manage access for our HR software, and make sure bad data can't sneak in. Build it right, and show us it works."
> — NextGen HR Director

Requirements gathered from the client kickoff call:

- Every employee needs a unique ID, full name, email, phone, and joining date
- Employees belong to a department and have a job role (Manager, Developer, Analyst, Sales Rep, HR Executive, Intern, Director)
- Salary is monthly, and sales reps additionally earn a commission percentage
- Employees report to a manager, who is *also* an employee in the same table (no separate manager table — this is a small firm)
- Status must be tracked: `ACTIVE`, `ON_LEAVE`, or `INACTIVE`
- The HR software team will need controlled database access
- Bad data (duplicate emails, negative salaries, missing names) must be impossible once the system is live

This single set of requirements is what every question below builds toward.

---

## PART 1 — SQL Data Types (Designing the Table)

Before writing a single `CREATE TABLE`, you need to pick the right Oracle data type for every column. Here's a quick reference of the Oracle types relevant to this project:

| Oracle Type | Stores | Typical Use |
|---|---|---|
| `NUMBER(p,s)` | Numeric, precision `p`, scale `s` decimals | IDs, salaries, percentages |
| `VARCHAR2(n)` | Variable-length text, up to `n` chars | Names, emails, codes |
| `CHAR(n)` | Fixed-length text, always padded to `n` chars | Rarely ideal — true fixed codes only |
| `DATE` | Date + time (to the second) | Hire dates, DOBs |
| `TIMESTAMP` | Date + time with fractional seconds | Precise audit/event logging |
| `CLOB` | Large text (up to 4GB) | Long free-text notes/remarks |

### Question 1 — Designing the Employee Table

**Business Question:** Based on the client brief above, propose the Oracle data type for each of these attributes, with a one-line justification: unique employee ID, full name, email, phone number, date of joining, job role, department, monthly salary (which must support paise/cents), commission percentage (0–100, up to 2 decimals), reporting manager's ID, and employment status.

**Expected Output:** *(This is a design deliverable, not a query — the "output" is the finalized mapping below.)*

| Column | Data Type | Why |
|---|---|---|
| `emp_id` | `NUMBER` | Whole-number surrogate key, no decimals needed |
| `emp_name` | `VARCHAR2(50)` | Names vary in length — no reason to pad |
| `email` | `VARCHAR2(100)` | Variable length, needs room for long domains |
| `phone_number` | `VARCHAR2(15)` | Stored as text, not numeric — no arithmetic is ever done on a phone number, and leading zeros/`+` must survive |
| `hire_date` | `DATE` | Native date type gives you date arithmetic and comparisons for free |
| `job_role` | `VARCHAR2(20)` | Short codes like `DEVELOPER`, `MANAGER` |
| `department` | `VARCHAR2(20)` | Variable length department names |
| `salary` | `NUMBER(10,2)` | Up to 8 digits before the decimal, 2 after (paise-safe) |
| `commission_pct` | `NUMBER(4,2)` | Percentages like `10.00`, nullable for non-sales roles |
| `manager_id` | `NUMBER` | Same type as `emp_id`, since it will reference it |
| `status` | `VARCHAR2(10)` | Fits `ACTIVE`, `INACTIVE`, `ON_LEAVE` |

**Answer / Explanation:** The key reasoning: `NUMBER` for anything you'll do arithmetic on or that behaves like a true ID (`emp_id`, `manager_id`, `salary`, `commission_pct`), `DATE` for anything you'll do date math or range filtering on (`hire_date`), and `VARCHAR2` — never `CHAR` — for text, because text length genuinely varies here. `phone_number` is deliberately `VARCHAR2`, not `NUMBER`, since phone numbers aren't quantities.

---

### Question 2 — CHAR vs VARCHAR2

**Business Question:** A junior developer on the team suggests using `CHAR(20)` instead of `VARCHAR2(20)` for the `department` column, arguing "it's more standard." The HR lead wants to know if this will cause any problems before go-live. What do you tell them?

**Expected Output:**

```
Storing 'IT' in a CHAR(20) column:
  Actual stored value: 'IT                  '   (padded with 18 trailing spaces)

Comparison behavior:
  WHERE department = 'IT'    -->  TRUE  (Oracle blank-pads the literal to compare)
  WHERE department = 'IT '   -->  TRUE  (still matches — blank-padded comparison)
  LENGTH(department)         -->  20    (not 2, as most developers expect)
```

**Answer:** Use `VARCHAR2(20)`, not `CHAR(20)`.

```sql
-- Recommended
department VARCHAR2(20)

-- Not recommended for this column
department CHAR(20)
```

_Explanation:_ `CHAR` always pads to its declared length with trailing spaces and wastes storage on every row for short values like `'IT'` or `'HR'`. It also silently breaks assumptions elsewhere in application code that calls `LENGTH()` or concatenates values, because the padding is real stored data. `CHAR` is only appropriate for genuinely fixed-length values (e.g., a 2-character ISO country code). `VARCHAR2` is the correct default for almost all text in Oracle, which is exactly what we used in Question 1.

---

## PART 2 — Data Definition Language (DDL)

DDL is how the table's *structure* is built and changed. The client wants to move fast, so the first version of the table goes live without formal constraints — those get added later once the structure is stable (Part 7 will circle back to that technical debt, just like it would on a real project).

### Question 3 — CREATE the Table

**Business Question:** Build the initial version of the employee table using the data types agreed on in Question 1, so the HR team can start onboarding data next sprint.

**Expected Output:**

```
Table EMPLOYEES created.
```

| Name | Null? | Type |
|---|---|---|
| EMP_ID | | NUMBER |
| EMP_NAME | | VARCHAR2(50) |
| EMAIL | | VARCHAR2(100) |
| PHONE_NUMBER | | VARCHAR2(15) |
| HIRE_DATE | | DATE |
| JOB_ROLE | | VARCHAR2(20) |
| DEPARTMENT | | VARCHAR2(20) |
| SALARY | | NUMBER(10,2) |
| COMMISSION_PCT | | NUMBER(4,2) |
| MANAGER_ID | | NUMBER |
| STATUS | | VARCHAR2(10) |

**Answer:**

```sql
CREATE TABLE employees (
    emp_id          NUMBER,
    emp_name        VARCHAR2(50),
    email           VARCHAR2(100),
    phone_number    VARCHAR2(15),
    hire_date       DATE,
    job_role        VARCHAR2(20),
    department      VARCHAR2(20),
    salary          NUMBER(10,2),
    commission_pct  NUMBER(4,2),
    manager_id      NUMBER,
    status          VARCHAR2(10)
);
```

_Explanation:_ `CREATE TABLE` is DDL — it defines structure, not data. No constraints yet on purpose; we'll add those formally in Part 7 once we cover Data Integrity.

---

### Question 4 — ALTER: Add a Column

**Business Question:** Mid-sprint, HR asks for one more field: a free-text notes area where managers can jot down remarks about an employee (training completed, warnings, achievements). This could get long, so plan for it.

**Expected Output:**

```
Table altered.
```

| Name | Null? | Type |
|---|---|---|
| ... (all previous columns) | | |
| REMARKS | | CLOB |

**Answer:**

```sql
ALTER TABLE employees ADD (remarks CLOB);
```

_Explanation:_ `CLOB` is used because remarks are unbounded free text — this is exactly the scenario `CLOB` exists for, tying back to Part 1.

---

### Question 5 — ALTER: Modify a Column

**Business Question:** During testing, a few employees with long names (including middle names) got truncated when inserted — `emp_name VARCHAR2(50)` isn't enough. Widen it before real data loading starts.

**Expected Output:**

```
Table altered.
```

| Name | Null? | Type |
|---|---|---|
| EMP_NAME | | VARCHAR2(100) |

**Answer:**

```sql
ALTER TABLE employees MODIFY (emp_name VARCHAR2(100));
```

_Explanation:_ `MODIFY` changes an existing column's definition. Widening a `VARCHAR2` is always safe; shrinking it would fail if existing data is already longer than the new size.

---

### Question 6 — RENAME the Table

**Business Question:** As part of a company-wide naming standard, the client wants every core HR table prefixed with `_MASTER`. Rename the table accordingly. From this point forward in the project, the table is called `employee_master`.

**Expected Output:**

```
Table renamed.
```

**Answer:**

```sql
RENAME employees TO employee_master;
```

_Explanation:_ `RENAME` is DDL that changes the object's identity without touching its data or structure. **Every query from here on uses `employee_master`.**

---

### Question 7 — TRUNCATE: Clear Test Data

**Business Question:** During UAT, the QA team loaded 4 dummy rows to validate the table works. Testing is complete and go-live is tomorrow — clear all test data but keep the table structure intact for the real data load.

**Expected Output — before truncation (QA's dummy rows):**

| EMP_ID | EMP_NAME | DEPARTMENT | STATUS |
|---|---|---|---|
| 1 | TEST ONE | QA | ACTIVE |
| 2 | TEST TWO | QA | ACTIVE |
| 3 | TEST THREE | QA | ACTIVE |
| 4 | TEST FOUR | QA | ACTIVE |

**Expected Output — after truncation:**

```
Table truncated.
```
```
0 rows selected.
```

**Answer:**

```sql
TRUNCATE TABLE employee_master;
```

_Explanation:_ `TRUNCATE` (DDL) instantly removes **all** rows and resets storage, but keeps the table structure. Unlike `DELETE` (a DML command covered in Part 3), it can't be rolled back, doesn't fire row-level triggers, and is much faster for wiping an entire table — the right tool here since QA wants *everything* gone, not selective rows.

---

### Question 8 — ALTER: Drop a Column

**Business Question:** Before the real data load, product decides the free-text `remarks` field is unnecessary — HR will track notes in a separate system instead. Clean it up now, before go-live.

**Expected Output:**

```
Table altered.
```

| Name | Null? | Type |
|---|---|---|
| EMP_ID | | NUMBER |
| EMP_NAME | | VARCHAR2(100) |
| EMAIL | | VARCHAR2(100) |
| PHONE_NUMBER | | VARCHAR2(15) |
| HIRE_DATE | | DATE |
| JOB_ROLE | | VARCHAR2(20) |
| DEPARTMENT | | VARCHAR2(20) |
| SALARY | | NUMBER(10,2) |
| COMMISSION_PCT | | NUMBER(4,2) |
| MANAGER_ID | | NUMBER |
| STATUS | | VARCHAR2(10) |

*(`REMARKS` is gone — this is the final column list for the rest of the project.)*

**Answer:**

```sql
ALTER TABLE employee_master DROP COLUMN remarks;
```

_Explanation:_ `DROP COLUMN` permanently removes a column and its data. This closes out the DDL phase — the table structure is now locked in for the go-live data load.

---

## PART 3 — Data Manipulation Language (DML)

`employee_master` is now empty and ready. The client's legacy-system export team has already migrated 7 employees (`EMP_ID` 100–106) as part of go-live. This is your starting dataset for everything that follows — treat it as already loaded, no question needed:

**Reference — data already migrated (context only, not a graded question):**

| EMP_ID | EMP_NAME | JOB_ROLE | DEPARTMENT | SALARY | COMMISSION_PCT | MANAGER_ID | HIRE_DATE | STATUS |
|---|---|---|---|---|---|---|---|---|
| 100 | Ananya Sharma | DIRECTOR | EXEC | 185000 | NULL | NULL | 2015-01-10 | ACTIVE |
| 101 | Rohan Mehta | MANAGER | IT | 125000 | NULL | 100 | 2017-03-15 | ACTIVE |
| 102 | Priya Nair | MANAGER | SALES | 118000 | NULL | 100 | 2018-06-01 | ACTIVE |
| 103 | Karthik Iyer | DEVELOPER | IT | 78000 | NULL | 101 | 2019-02-20 | ACTIVE |
| 104 | Sneha Reddy | DEVELOPER | IT | 82000 | NULL | 101 | 2019-11-05 | ACTIVE |
| 105 | Arjun Verma | SALES_REP | SALES | 55000 | 8 | 102 | 2020-04-18 | ACTIVE |
| 106 | Divya Krishnan | SALES_REP | SALES | 58000 | 10 | 102 | 2020-09-09 | ON_LEAVE |

### Question 9 — INSERT a Single Row

**Business Question:** A new Finance Analyst, Vikram Singh, just joined and reports directly to the Director (Ananya Sharma, ID 100). Add him to the system.

**Expected Output:**

```
1 row created.
```

| EMP_ID | EMP_NAME | JOB_ROLE | DEPARTMENT | SALARY | MANAGER_ID | STATUS |
|---|---|---|---|---|---|---|
| 107 | Vikram Singh | ANALYST | FINANCE | 72000 | 100 | ACTIVE |

**Answer:**

```sql
INSERT INTO employee_master
    (emp_id, emp_name, email, phone_number, hire_date, job_role,
     department, salary, commission_pct, manager_id, status)
VALUES
    (107, 'Vikram Singh', 'vikram.singh@nextgen.com', '9840011129',
     DATE '2021-01-25', 'ANALYST', 'FINANCE', 72000, NULL, 100, 'ACTIVE');
```

_Explanation:_ Standard single-row `INSERT`. `commission_pct` is `NULL` because Analysts don't earn commission.

---

### Question 10 — INSERT Multiple Rows in One Statement

**Business Question:** HR just finished a hiring drive at a campus job fair and has 4 new offers to onboard in one batch: an HR Executive, a Developer, a Sales Rep, and an Intern. Load all 4 in a single statement.

**Expected Output:**

```
4 rows created.
```

| EMP_ID | EMP_NAME | JOB_ROLE | DEPARTMENT | SALARY | MANAGER_ID | STATUS |
|---|---|---|---|---|---|---|
| 108 | Meera Pillai | HR_EXEC | HR | 65000 | 100 | ACTIVE |
| 109 | Suresh Kumar | DEVELOPER | IT | 76000 | 101 | ACTIVE |
| 110 | Rahul Nanda | SALES_REP | SALES | 52000 | 102 | ACTIVE |
| 111 | Anjali Desai | INTERN | IT | 25000 | 101 | ACTIVE |

**Answer:**

```sql
INSERT ALL
  INTO employee_master (emp_id, emp_name, email, phone_number, hire_date,
       job_role, department, salary, commission_pct, manager_id, status)
       VALUES (108, 'Meera Pillai', 'meera.pillai@nextgen.com', '9840011130',
       DATE '2023-06-01', 'HR_EXEC', 'HR', 65000, NULL, 100, 'ACTIVE')
  INTO employee_master (emp_id, emp_name, email, phone_number, hire_date,
       job_role, department, salary, commission_pct, manager_id, status)
       VALUES (109, 'Suresh Kumar', 'suresh.kumar@nextgen.com', '9840011131',
       DATE '2023-06-01', 'DEVELOPER', 'IT', 76000, NULL, 101, 'ACTIVE')
  INTO employee_master (emp_id, emp_name, email, phone_number, hire_date,
       job_role, department, salary, commission_pct, manager_id, status)
       VALUES (110, 'Rahul Nanda', 'rahul.nanda@nextgen.com', '9840011132',
       DATE '2023-06-01', 'SALES_REP', 'SALES', 52000, NULL, 102, 'ACTIVE')
  INTO employee_master (emp_id, emp_name, email, phone_number, hire_date,
       job_role, department, salary, commission_pct, manager_id, status)
       VALUES (111, 'Anjali Desai', 'anjali.desai@nextgen.com', '9840011133',
       DATE '2023-06-01', 'INTERN', 'IT', 25000, NULL, 101, 'ACTIVE')
SELECT * FROM dual;
```

_Explanation:_ Oracle doesn't support MySQL-style `VALUES (...), (...), (...)` multi-row inserts. The Oracle idiom is `INSERT ALL ... SELECT * FROM dual`, which fires each `INTO` block once against the single dummy row from `dual`. Note Rahul Nanda (110) has `commission_pct = NULL` — he's on probation and commission hasn't been finalized yet, which sets up Question 12.

---

### Question 11 — UPDATE: Department-Wide Raise

**Business Question:** The IT department just hit a major project milestone. Leadership approves a 10% raise for every **active, non-intern** IT employee.

**Expected Output:**

```
4 rows updated.
```

| EMP_ID | EMP_NAME | OLD SALARY | NEW SALARY |
|---|---|---|---|
| 101 | Rohan Mehta | 125000 | 137500 |
| 103 | Karthik Iyer | 78000 | 85800 |
| 104 | Sneha Reddy | 82000 | 90200 |
| 109 | Suresh Kumar | 76000 | 83600 |

**Answer:**

```sql
UPDATE employee_master
SET salary = salary * 1.10
WHERE department = 'IT'
  AND status = 'ACTIVE'
  AND job_role <> 'INTERN';
```

_Explanation:_ The intern (Anjali Desai, 111) is deliberately excluded via `job_role <> 'INTERN'` — interns aren't part of the milestone bonus policy. This is `salary * 1.10`, an arithmetic expression inside an `UPDATE`, previewing Part 8.

---

### Question 12 — UPDATE: Fix a Missing Value

**Business Question:** Rahul Nanda (Sales Rep, ID 110) has completed his probation. Sales leadership confirms his commission rate at 7%.

**Expected Output:**

```
1 row updated.
```

| EMP_ID | EMP_NAME | COMMISSION_PCT |
|---|---|---|
| 110 | Rahul Nanda | 7 |

**Answer:**

```sql
UPDATE employee_master
SET commission_pct = 7
WHERE emp_id = 110;
```

_Explanation:_ A simple, precisely-targeted `UPDATE` using the primary identifier — the safest way to update a single row.

---

### Question 13 — DELETE: Remove a Record

**Business Question:** Anjali Desai's 3-month internship has ended and she did not convert to full-time. Per data-retention policy, her record should be removed from the active employee table.

**Expected Output:**

```
1 row deleted.
```

*(EMP_ID 111 no longer appears in any subsequent query.)*

**Answer:**

```sql
DELETE FROM employee_master
WHERE emp_id = 111;
```

_Explanation:_ `DELETE` removes specific rows (unlike `TRUNCATE`, which we already used in Question 7 to remove everything). It's transactional — it can be rolled back if it hasn't been committed yet, which becomes very relevant in Part 6.

---

## PART 4 — Data Query Language (DQL)

The working dataset now stands at 11 employees (100–110). Here it is for reference before we start querying:

| EMP_ID | EMP_NAME | JOB_ROLE | DEPARTMENT | SALARY | COMMISSION_PCT | MANAGER_ID | HIRE_DATE | STATUS |
|---|---|---|---|---|---|---|---|---|
| 100 | Ananya Sharma | DIRECTOR | EXEC | 185000 | NULL | NULL | 2015-01-10 | ACTIVE |
| 101 | Rohan Mehta | MANAGER | IT | 137500 | NULL | 100 | 2017-03-15 | ACTIVE |
| 102 | Priya Nair | MANAGER | SALES | 118000 | NULL | 100 | 2018-06-01 | ACTIVE |
| 103 | Karthik Iyer | DEVELOPER | IT | 85800 | NULL | 101 | 2019-02-20 | ACTIVE |
| 104 | Sneha Reddy | DEVELOPER | IT | 90200 | NULL | 101 | 2019-11-05 | ACTIVE |
| 105 | Arjun Verma | SALES_REP | SALES | 55000 | 8 | 102 | 2020-04-18 | ACTIVE |
| 106 | Divya Krishnan | SALES_REP | SALES | 58000 | 10 | 102 | 2020-09-09 | ON_LEAVE |
| 107 | Vikram Singh | ANALYST | FINANCE | 72000 | NULL | 100 | 2021-01-25 | ACTIVE |
| 108 | Meera Pillai | HR_EXEC | HR | 65000 | NULL | 100 | 2023-06-01 | ACTIVE |
| 109 | Suresh Kumar | DEVELOPER | IT | 83600 | NULL | 101 | 2023-06-01 | ACTIVE |
| 110 | Rahul Nanda | SALES_REP | SALES | 52000 | 7 | 102 | 2023-06-01 | ACTIVE |

### Question 14 — SELECT with WHERE and ORDER BY

**Business Question:** The CTO wants a quick look at everyone in the IT department, ranked from highest to lowest paid, to review compensation before budget planning.

**Expected Output:**

```
4 rows selected.
```

| EMP_NAME | JOB_ROLE | SALARY |
|---|---|---|
| Rohan Mehta | MANAGER | 137500 |
| Sneha Reddy | DEVELOPER | 90200 |
| Karthik Iyer | DEVELOPER | 85800 |
| Suresh Kumar | DEVELOPER | 83600 |

**Answer:**

```sql
SELECT emp_name, job_role, salary
FROM employee_master
WHERE department = 'IT'
ORDER BY salary DESC;
```

---

### Question 15 — FETCH FIRST: Top-N Report

**Business Question:** The board wants a one-page summary of the company's 3 highest-paid employees, company-wide.

**Expected Output:**

```
3 rows selected.
```

| EMP_NAME | DEPARTMENT | SALARY |
|---|---|---|
| Ananya Sharma | EXEC | 185000 |
| Rohan Mehta | IT | 137500 |
| Priya Nair | SALES | 118000 |

**Answer:**

```sql
SELECT emp_name, department, salary
FROM employee_master
ORDER BY salary DESC
FETCH FIRST 3 ROWS ONLY;
```

_Explanation:_ `FETCH FIRST n ROWS ONLY` is the modern (12c+) Oracle syntax for top-N queries — cleaner than the older `ROWNUM` trick.

---

### Question 16 — Department Summary Report

**Business Question:** Finance wants a department-wise headcount and average salary report for the quarterly budget review.

**Expected Output:**

```
5 rows selected.
```

| DEPARTMENT | HEADCOUNT | AVG_SALARY |
|---|---|---|
| EXEC | 1 | 185000.00 |
| IT | 4 | 99275.00 |
| FINANCE | 1 | 72000.00 |
| HR | 1 | 65000.00 |
| SALES | 3 | 55000.00 |

**Answer:**

```sql
SELECT department,
       COUNT(*)          AS headcount,
       ROUND(AVG(salary), 2) AS avg_salary
FROM employee_master
GROUP BY department
ORDER BY avg_salary DESC;
```

_Explanation:_ `GROUP BY` with an aggregate is a natural extension of `SELECT` for reporting — every real HR system needs exactly this kind of rollup.

---

### Question 17 — FETCH FIRST with OFFSET: Pagination

**Business Question:** The HR dashboard shows the salary leaderboard 3 employees at a time. Get **page 2** (skip the top 3, show the next 3).

**Expected Output:**

```
3 rows selected.
```

| EMP_NAME | SALARY |
|---|---|
| Sneha Reddy | 90200 |
| Karthik Iyer | 85800 |
| Suresh Kumar | 83600 |

**Answer:**

```sql
SELECT emp_name, salary
FROM employee_master
ORDER BY salary DESC
OFFSET 3 ROWS FETCH NEXT 3 ROWS ONLY;
```

_Explanation:_ `OFFSET ... FETCH NEXT ... ROWS ONLY` is Oracle's row-limiting pagination syntax — `FETCH FIRST` (Question 15) is really just this with an implicit offset of 0.

---

## PART 5 — Data Control Language (DCL)

### Question 18 — GRANT: Onboard the HR Software's Database User

**Business Question:** The HR self-service portal uses a dedicated database account, `hr_exec`, to let HR staff view and update employee records through the app. Give it the access it needs.

**Expected Output:**

```
Grant succeeded.
```

| GRANTEE | PRIVILEGE | ON |
|---|---|---|
| HR_EXEC | SELECT | EMPLOYEE_MASTER |
| HR_EXEC | UPDATE | EMPLOYEE_MASTER |

**Answer:**

```sql
GRANT SELECT, UPDATE ON employee_master TO hr_exec;
```

_Explanation:_ `GRANT` is DCL — it controls *who can do what*, as opposed to DML which controls *what happens to the data*. `hr_exec` gets exactly the two privileges the portal needs: read and update. It deliberately does **not** get `DELETE` or `INSERT`, since those flows go through a different admin tool.

---

### Question 19 — REVOKE: Tighten Access After an Audit

**Business Question:** A security audit flags that the HR portal account shouldn't be able to modify salary data directly — only the payroll system should. Remove the `UPDATE` privilege from `hr_exec` but leave read access intact.

**Expected Output:**

```
Revoke succeeded.
```

| GRANTEE | PRIVILEGE | ON |
|---|---|---|
| HR_EXEC | SELECT | EMPLOYEE_MASTER |

*(`UPDATE` no longer listed.)*

**Answer:**

```sql
REVOKE UPDATE ON employee_master FROM hr_exec;
```

_Explanation:_ `REVOKE` removes a previously granted privilege without touching any other privileges the user has — `hr_exec` can still `SELECT`, it just lost `UPDATE`.

---

## PART 6 — Transaction Control Language (TCL)

### Question 20 — COMMIT: Lock In a Verified Batch

**Business Question:** HR has spot-checked every insert, update, and delete made so far (Questions 9–13) against the source spreadsheets and confirmed everything is correct. Make it permanent.

**Expected Output:**

```
Commit complete.
```

**Answer:**

```sql
COMMIT;
```

_Explanation:_ Until this point, all the DML from Part 3 was only visible within the current session and could still be undone. `COMMIT` makes it permanent and visible to every other session/user.

---

### Question 21 — SAVEPOINT: Checkpoint Before a Risky Change

**Business Question:** Sales leadership wants a 15% raise applied to the entire SALES department — but this number came verbally and hasn't been signed off in writing yet. Create a safety checkpoint before applying it, in case it needs to be undone.

**Expected Output:**

```
Savepoint created.

3 rows updated.
```

| EMP_ID | EMP_NAME | OLD SALARY | NEW SALARY (pending) |
|---|---|---|---|
| 105 | Arjun Verma | 55000 | 63250 |
| 106 | Divya Krishnan | 58000 | 66700 |
| 110 | Rahul Nanda | 52000 | 59800 |

**Answer:**

```sql
SAVEPOINT sp_before_sales_raise;

UPDATE employee_master
SET salary = salary * 1.15
WHERE department = 'SALES';
```

_Explanation:_ `SAVEPOINT` marks a named point *within* the current uncommitted transaction that you can roll back to later, without undoing everything since the last `COMMIT`.

---

### Question 22 — ROLLBACK TO SAVEPOINT: Undo Just the Risky Change

**Business Question:** As suspected — Finance calls back and says the real approved figure is 5%, not 15%, and someone misheard on the call. Undo the 15% raise **without** losing the fact that Questions 9–13 are already safely committed.

**Expected Output:**

```
Rollback complete.
```

| EMP_ID | EMP_NAME | SALARY (restored) |
|---|---|---|
| 105 | Arjun Verma | 55000 |
| 106 | Divya Krishnan | 58000 |
| 110 | Rahul Nanda | 52000 |

**Answer:**

```sql
ROLLBACK TO sp_before_sales_raise;
```

_Explanation:_ `ROLLBACK TO <savepoint>` undoes only the work done *after* that savepoint — the 15% update — while everything committed in Question 20 stays exactly as it was. This is the safe, surgical alternative to a full `ROLLBACK`, which would have undone everything back to the last commit. (The correct 5% raise can now be reapplied as a fresh, separate `UPDATE` + `COMMIT` — not shown here, since it's identical mechanically to Question 11.)

---

## PART 7 — Data Integrity & Constraints

The table has been live for a few weeks with no formal constraints — exactly the kind of technical debt a real project accumulates under deadline pressure. Now it's time to lock the data down properly.

### Question 23 — PRIMARY KEY Constraint + a Sequence Generator

**Business Question:** Nothing currently stops two employees from accidentally getting the same `emp_id`. Enforce uniqueness formally, and set up auto-numbering so nobody has to manually track "what's the next free ID" ever again. Then onboard the next new hire, Kavya Menon (IT Intern, reports to Rohan Mehta), using the new auto-numbering.

**Expected Output:**

```
Table altered.

Sequence EMP_ID_SEQ created.

1 row created.
```

| EMP_ID | EMP_NAME | JOB_ROLE | DEPARTMENT | MANAGER_ID |
|---|---|---|---|---|
| 111 | Kavya Menon | INTERN | IT | 101 |

**Answer:**

```sql
-- Entity integrity: no duplicate or missing IDs allowed
ALTER TABLE employee_master
ADD CONSTRAINT pk_employee_master PRIMARY KEY (emp_id);

-- Sequence generator, picking up right after the highest existing ID
CREATE SEQUENCE emp_id_seq
    START WITH 111
    INCREMENT BY 1
    NOCACHE
    NOCYCLE;

INSERT INTO employee_master
    (emp_id, emp_name, email, phone_number, hire_date, job_role,
     department, salary, commission_pct, manager_id, status)
VALUES
    (emp_id_seq.NEXTVAL, 'Kavya Menon', 'kavya.menon@nextgen.com', '9840011134',
     DATE '2024-01-15', 'INTERN', 'IT', 27000, NULL, 101, 'ACTIVE');
```

_Explanation:_ A `PRIMARY KEY` constraint enforces **entity integrity** — every row must be uniquely and reliably identifiable, and `emp_id` can never be `NULL` from this point on. The **sequence generator** removes manual ID management: `emp_id_seq.NEXTVAL` always returns the next free number. It was deliberately started at 111 (Anjali Desai's old, now-deleted ID) since 110 is the current highest ID in use.

---

### Question 24 — FOREIGN KEY Constraint (Self-Referencing)

**Business Question:** Right now, `manager_id` is just a plain number — nothing stops someone from typing in `manager_id = 9999`, a manager who doesn't exist. Since every manager is *also* an employee in this same table, enforce that every `manager_id` must point to a real, existing `emp_id`.

**Expected Output:**

```
Table altered.
```

```
-- Test: attempting to insert an employee with a fake manager afterward now fails:
INSERT INTO employee_master (emp_id, emp_name, ..., manager_id, status)
VALUES (emp_id_seq.NEXTVAL, 'Test Person', ..., 9999, 'ACTIVE');

ORA-02291: integrity constraint (NEXTGEN.FK_MANAGER) violated - parent key not found
```

**Answer:**

```sql
ALTER TABLE employee_master
ADD CONSTRAINT fk_manager FOREIGN KEY (manager_id)
REFERENCES employee_master (emp_id);
```

_Explanation:_ This is **referential integrity**, and it's a **self-referencing foreign key** — `employee_master.manager_id` references `employee_master.emp_id`, the same table. This is exactly how "an employee reports to another employee" is modeled without a separate managers table, which matches the client's original one-table requirement. `manager_id` can still be `NULL` (for Ananya Sharma, the Director, who has no manager) — a foreign key only restricts *non-null* values to existing parent rows.

---

### Question 25 — Domain Integrity: NOT NULL, UNIQUE, CHECK

**Business Question:** Before the system is considered "production-grade," the client's data governance policy requires: every employee must have a name and a hire date on file; no two employees can share an email address; salaries must never be zero or negative; and status must always be one of the three valid values (no typos like `'ACTVE'` slipping in).

**Expected Output:**

```
Table altered.

Table altered.

Table altered.

Table altered.

Table altered.
```

| Constraint Name | Type | Column(s) |
|---|---|---|
| SYS_C0012345 (system-named, from MODIFY) | NOT NULL | EMP_NAME |
| SYS_C0012346 (system-named, from MODIFY) | NOT NULL | HIRE_DATE |
| UQ_EMAIL | UNIQUE | EMAIL |
| CHK_SALARY | CHECK (`salary > 0`) | SALARY |
| CHK_STATUS | CHECK (`status IN ('ACTIVE','INACTIVE','ON_LEAVE')`) | STATUS |

**Answer:**

```sql
-- Domain integrity: mandatory fields
ALTER TABLE employee_master MODIFY (emp_name VARCHAR2(100) NOT NULL);
ALTER TABLE employee_master MODIFY (hire_date NOT NULL);

-- No two employees may share an email
ALTER TABLE employee_master ADD CONSTRAINT uq_email UNIQUE (email);

-- Business rule: salary must be a positive amount
ALTER TABLE employee_master ADD CONSTRAINT chk_salary CHECK (salary > 0);

-- Business rule: status is restricted to 3 known values
ALTER TABLE employee_master
ADD CONSTRAINT chk_status CHECK (status IN ('ACTIVE','INACTIVE','ON_LEAVE'));
```

_Explanation:_ `NOT NULL` and `CHECK` enforce **domain integrity** — the *content* of a column, not just its uniqueness or its relationship to another table. `UNIQUE` here is also domain-level (email format/business rule), distinct from the `PRIMARY KEY` in Question 23, which is about row identity. Naming constraints explicitly (`uq_email`, `chk_salary`, `chk_status`) is good practice — it gives you a readable name in error messages instead of an auto-generated `SYS_C00...` name.

---

### Question 26 — Testing the Constraints (User-Defined Integrity in Action)

**Business Question:** A new HR intern is onboarding a fresh hire, Sanjana Rao, and accidentally reuses Meera Pillai's email address by copy-paste error. Show what happens, and how it gets corrected.

**Expected Output — the mistake:**

```sql
INSERT INTO employee_master
    (emp_id, emp_name, email, phone_number, hire_date, job_role,
     department, salary, commission_pct, manager_id, status)
VALUES
    (emp_id_seq.NEXTVAL, 'Sanjana Rao', 'meera.pillai@nextgen.com', '9840011135',
     DATE '2024-02-01', 'DEVELOPER', 'IT', 74000, NULL, 101, 'ACTIVE');
```
```
ORA-00001: unique constraint (NEXTGEN.UQ_EMAIL) violated
```

**Expected Output — corrected:**

```
1 row created.
```

**Answer:**

```sql
-- Corrected: use Sanjana's own email
INSERT INTO employee_master
    (emp_id, emp_name, email, phone_number, hire_date, job_role,
     department, salary, commission_pct, manager_id, status)
VALUES
    (emp_id_seq.NEXTVAL, 'Sanjana Rao', 'sanjana.rao@nextgen.com', '9840011135',
     DATE '2024-02-01', 'DEVELOPER', 'IT', 74000, NULL, 101, 'ACTIVE');
```

_Explanation:_ This is the entire point of the `UQ_EMAIL` constraint from Question 25 — it makes bad data **impossible**, not just discouraged, exactly as the client's original brief demanded. *(This record is used only to illustrate the fix in action — the running case-study dataset used in Part 8 continues with the 12 employees established through Question 23, so the numbers stay consistent.)*

---

### Question 27 — ENABLE / DISABLE Constraints

**Business Question:** The client wants to bulk-migrate 200 historical records of resigned employees from an old legacy system. Some of those old rows use `0` as a placeholder salary for incomplete records, which would violate the `CHK_SALARY` constraint. The DBA needs to load the raw data first and clean it up afterward, rather than blocking the whole migration.

**Expected Output:**

```
Table altered.        -- constraint disabled

-- (bulk historical load happens here — 200 legacy rows inserted, some with salary = 0)

Table altered.         -- data cleaned up, constraint re-enabled successfully
```

**Answer:**

```sql
-- Step 1: temporarily disable the check so the raw legacy load doesn't fail
ALTER TABLE employee_master DISABLE CONSTRAINT chk_salary;

-- Step 2: bulk-load the 200 historical rows (some with placeholder salary = 0)
-- ... INSERT / SQL*Loader / external table process happens here ...

-- Step 3: clean up the placeholder values before re-enabling
UPDATE employee_master
SET salary = 1
WHERE salary = 0;

-- Step 4: re-enable the constraint now that data is valid
ALTER TABLE employee_master ENABLE CONSTRAINT chk_salary;
```

_Explanation:_ `DISABLE CONSTRAINT` temporarily turns off enforcement without dropping the rule — useful for bulk loads, migrations, or emergency fixes. Re-running `ENABLE CONSTRAINT` **re-validates every existing row**; if any row still violates the rule (e.g., a leftover `salary = 0`), Oracle throws `ORA-02293` and refuses to re-enable until the data is fixed. This is exactly why Step 3 (the cleanup) has to happen *before* Step 4.

---

## PART 8 — SQL Operators

This is the mixed, capstone section — every question here deliberately combines operators with everything learned in Parts 2–7 (filtering, arithmetic on real columns, and multi-set business logic), the way a real analytics request from a client actually reads.

### Question 28 — Arithmetic Operators: Annual CTC Report

**Business Question:** Finance needs an annual Cost-To-Company (CTC) report for all Sales Reps: their annual base salary, their annual commission earnings, and the combined total.

**Expected Output:**

```
3 rows selected.
```

| EMP_NAME | MONTHLY_SALARY | ANNUAL_SALARY | ANNUAL_COMMISSION | TOTAL_ANNUAL_CTC |
|---|---|---|---|---|
| Arjun Verma | 55000 | 660000 | 52800 | 712800 |
| Divya Krishnan | 58000 | 696000 | 69600 | 765600 |
| Rahul Nanda | 52000 | 624000 | 43680 | 667680 |

**Answer:**

```sql
SELECT emp_name,
       salary                                                AS monthly_salary,
       salary * 12                                           AS annual_salary,
       ROUND(salary * 12 * (commission_pct / 100), 2)        AS annual_commission,
       ROUND(salary * 12 + salary * 12 * (commission_pct/100), 2) AS total_annual_ctc
FROM employee_master
WHERE job_role = 'SALES_REP';
```

_Explanation:_ `*` and `+` are **arithmetic operators**, applied directly to the `salary` and `commission_pct` columns designed all the way back in Part 1. `/100` converts the stored whole-number percentage into a decimal multiplier.

---

### Question 29 — Comparison + Logical Operators: Targeted Talent Segment

**Business Question:** Leadership is planning a "core technical/finance talent" retention bonus, targeted at active, mid-salary-band individual contributors (not managers) in IT or Finance. Find everyone earning between ₹60,000–₹120,000, in IT or Finance, currently active, who is not a Manager.

**Expected Output:**

```
4 rows selected.
```

| EMP_NAME | DEPARTMENT | JOB_ROLE | SALARY | STATUS |
|---|---|---|---|---|
| Karthik Iyer | IT | DEVELOPER | 85800 | ACTIVE |
| Sneha Reddy | IT | DEVELOPER | 90200 | ACTIVE |
| Suresh Kumar | IT | DEVELOPER | 83600 | ACTIVE |
| Vikram Singh | FINANCE | ANALYST | 72000 | ACTIVE |

**Answer:**

```sql
SELECT emp_name, department, job_role, salary, status
FROM employee_master
WHERE status = 'ACTIVE'
  AND salary BETWEEN 60000 AND 120000
  AND department IN ('IT', 'FINANCE')
  AND job_role <> 'MANAGER';
```

_Explanation:_ This combines **comparison operators** (`BETWEEN`, `IN`, `<>`) with the **logical operator** `AND` to chain four independent conditions together. Rohan Mehta is correctly excluded (he's a Manager), and Kavya Menon is correctly excluded (₹27,000 is below the band) — both edge cases prove the filter is working as intended.

---

### Question 30 — Set Operators: UNION, INTERSECT, and MINUS (Capstone)

**Business Question:** The client's strategy team needs three different talent-analytics cuts from the same table for an offsite planning session:

**(a)** A combined list of "IT department OR long-tenured (hired before 2020)" employees, for a retention-risk review — no duplicates.
**(b)** Sales employees who *also* earn more than 7% commission, to identify top sales performers, using set logic rather than a single `AND`.
**(c)** Active employees who are *not* in the lower salary band (below ₹70,000), to build the retention-bonus-eligible list.

**Expected Output (a) — UNION:**

```
7 rows selected.
```

| EMP_ID | EMP_NAME | DEPARTMENT | HIRE_DATE |
|---|---|---|---|
| 100 | Ananya Sharma | EXEC | 2015-01-10 |
| 101 | Rohan Mehta | IT | 2017-03-15 |
| 102 | Priya Nair | SALES | 2018-06-01 |
| 103 | Karthik Iyer | IT | 2019-02-20 |
| 104 | Sneha Reddy | IT | 2019-11-05 |
| 109 | Suresh Kumar | IT | 2023-06-01 |
| 111 | Kavya Menon | IT | 2024-01-15 |

**Expected Output (b) — INTERSECT:**

```
2 rows selected.
```

| EMP_ID | EMP_NAME | DEPARTMENT | COMMISSION_PCT |
|---|---|---|---|
| 105 | Arjun Verma | SALES | 8 |
| 106 | Divya Krishnan | SALES | 10 |

**Expected Output (c) — MINUS:**

```
7 rows selected.
```

| EMP_ID | EMP_NAME | SALARY | STATUS |
|---|---|---|---|
| 100 | Ananya Sharma | 185000 | ACTIVE |
| 101 | Rohan Mehta | 137500 | ACTIVE |
| 102 | Priya Nair | 118000 | ACTIVE |
| 103 | Karthik Iyer | 85800 | ACTIVE |
| 104 | Sneha Reddy | 90200 | ACTIVE |
| 107 | Vikram Singh | 72000 | ACTIVE |
| 109 | Suresh Kumar | 83600 | ACTIVE |

**Answer:**

```sql
-- (a) UNION: everyone matching either condition, duplicates removed automatically
SELECT emp_id, emp_name, department, hire_date
FROM employee_master
WHERE department = 'IT'
UNION
SELECT emp_id, emp_name, department, hire_date
FROM employee_master
WHERE hire_date < DATE '2020-01-01'
ORDER BY emp_id;

-- (b) INTERSECT: only rows present in BOTH result sets
SELECT emp_id, emp_name, department, commission_pct
FROM employee_master
WHERE department = 'SALES'
INTERSECT
SELECT emp_id, emp_name, department, commission_pct
FROM employee_master
WHERE commission_pct > 7
ORDER BY emp_id;

-- (c) MINUS: everything in the first set that is NOT in the second set
SELECT emp_id, emp_name, salary, status
FROM employee_master
WHERE status = 'ACTIVE'
MINUS
SELECT emp_id, emp_name, salary, status
FROM employee_master
WHERE salary < 70000
ORDER BY emp_id;
```

_Explanation:_ All three are **set operators**, and each behaves differently:
- `UNION` combines two result sets and removes duplicate *rows* (not duplicate values in one column) — Rohan Mehta (101) satisfies both conditions but appears only once.
- `INTERSECT` keeps only rows that exist in *both* queries — Rahul Nanda (110, exactly 7% commission) is correctly excluded since the condition is strictly `> 7`.
- `MINUS` (Oracle's version of `EXCEPT`) subtracts the second set from the first — Divya Krishnan (106) is excluded from the start since she's `ON_LEAVE`, not `ACTIVE`, so she was never in the first set to begin with.

For all three operators, **column count, order, and compatible data types must match exactly** between the two `SELECT` statements — that's a common first mistake.

---

## Final Schema Snapshot & Answer Key

By Question 30, `employee_master` has traveled through every concept in the brief:

```
EMPLOYEE_MASTER
├── emp_id          NUMBER          PRIMARY KEY (pk_employee_master)
├── emp_name        VARCHAR2(100)   NOT NULL
├── email           VARCHAR2(100)   UNIQUE (uq_email)
├── phone_number    VARCHAR2(15)
├── hire_date       DATE            NOT NULL
├── job_role        VARCHAR2(20)
├── department      VARCHAR2(20)
├── salary          NUMBER(10,2)    CHECK > 0 (chk_salary)
├── commission_pct  NUMBER(4,2)
├── manager_id      NUMBER          FOREIGN KEY → emp_id (fk_manager, self-referencing)
└── status          VARCHAR2(10)    CHECK IN ('ACTIVE','INACTIVE','ON_LEAVE') (chk_status)

Supporting object: EMP_ID_SEQ (sequence, next value 112)
Access granted:    HR_EXEC → SELECT only (UPDATE was revoked)
```

| # | Topic | Command(s) Practiced |
|---|---|---|
| 1–2 | Data Types | Type selection & justification, CHAR vs VARCHAR2 |
| 3–8 | DDL | CREATE, ALTER (ADD/MODIFY), RENAME, TRUNCATE, ALTER (DROP) |
| 9–13 | DML | INSERT (single + INSERT ALL), UPDATE ×2, DELETE |
| 14–17 | DQL | SELECT + WHERE + ORDER BY, FETCH FIRST, GROUP BY, OFFSET/FETCH |
| 18–19 | DCL | GRANT, REVOKE |
| 20–22 | TCL | COMMIT, SAVEPOINT, ROLLBACK TO |
| 23–27 | Data Integrity | PRIMARY KEY + sequence, FOREIGN KEY (self-ref), NOT NULL/UNIQUE/CHECK, constraint violation, ENABLE/DISABLE |
| 28–30 | Operators | Arithmetic, Comparison + Logical, Set (UNION/INTERSECT/MINUS) |

### Where to go next

Once this feels solid, natural extensions (for a future round) would be: splitting `department` and `job_role` into proper lookup tables and practicing **JOINs**, adding a `payroll_history` table to practice multi-table integrity, and moving from single expressions into **PL/SQL** procedures for the raise/onboarding logic you wrote by hand above.