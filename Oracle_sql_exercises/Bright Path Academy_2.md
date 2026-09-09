# Bright Path Academy — Student Enrollment & Performance Tracking System
### An Oracle SQL Hands-On Case Study (SQL Functions → Clauses → JOINs, all mixed at the end)

---

## How to Use This Guide

You're the database developer for **Bright Path Academy**, a professional upskilling institute running cohort-based courses in Data Science, Web Development, Cloud Computing, and Digital Marketing. This is a **different scenario from any previous case study** — but the same hands-on approach: everything happens on **one single table**, built once and never abandoned.

The twist that makes a single table work for **every JOIN type** (including LEFT/RIGHT/FULL OUTER and NATURAL JOIN) is that the table tracks a **referral program** — students who joined because another student referred them. Since a referrer is *also just a student in the same table*, this one table can join to itself in every way a real two-table relationship would, which is exactly what the JOIN section exploits.

Every question follows the format you asked for:

1. **Business Question** — the scenario, phrased the way a client or PM would ask it
2. **Expected Output** — what you should see if you run the query correctly
3. **Answer** — the SQL statement plus a short explanation

Run these in order — later questions (especially the JOIN and capstone sections) depend on exactly this data existing exactly as loaded in Question 2.

---

## Table of Contents

- [The Client Brief](#the-client-brief)
- [Setup — Building & Loading the Table](#setup--building--loading-the-table) (Q1–Q2)
- [Part 1 — SQL Functions Classification](#part-1--sql-functions-classification) (Q3–Q14)
- [Part 2 — Clauses](#part-2--clauses) (Q15–Q20)
- [Part 3 — JOINs & JOIN Styles](#part-3--joins--join-styles) (Q21–Q31)
- [Part 4 — Capstone: Everything Mixed Together](#part-4--capstone-everything-mixed-together) (Q32–Q35)
- [Final Answer Key & Concept Map](#final-answer-key--concept-map)

---

## The Client Brief

> "We run four cohort-based courses and most of our new students come from word-of-mouth — one student tells a friend, that friend enrolls, and so on. We've never been able to properly track who's referring whom, which courses are performing well academically, or get clean reports out of our data because everyone entered names differently. We need ONE proper table — student details, fees, exam scores, and who referred them — that we can actually run real reports against."
> — Bright Path Academy, Program Director

Requirements gathered from the kickoff call:

- One record per student: contact info, course, batch timing, fees (total/paid/discount), exam score, city, and status (`ACTIVE`, `COMPLETED`, `DROPPED`)
- Every student who joined via referral should have that traceable — **to another student in the very same table**, since referrers are themselves students, not a separate category of person
- Reports need to compare students within their course (rankings, averages), track revenue, and identify top referrers
- Some students haven't taken their exam yet — reports must handle that gracefully, not break
- Front-desk data entry has been inconsistent (casing, formatting) — the system needs to clean this up, not just store it as-is

---

## Setup — Building & Loading the Table

### Question 1 — Design and Create the Table

**Business Question:** Build the single table Bright Path Academy needs, based on the brief above. Since a referrer is just another student, `referred_by` must point back into the same table.

**Expected Output:**

```
Table STUDENT_ENROLLMENT created.
```

| Name | Null? | Type |
|---|---|---|
| STUDENT_ID | NOT NULL | NUMBER |
| STUDENT_NAME | NOT NULL | VARCHAR2(100) |
| EMAIL | | VARCHAR2(100) |
| PHONE_NUMBER | | VARCHAR2(15) |
| COURSE | NOT NULL | VARCHAR2(30) |
| BATCH | | VARCHAR2(15) |
| ENROLLMENT_DATE | NOT NULL | DATE |
| FEES_TOTAL | | NUMBER(10,2) |
| FEES_PAID | | NUMBER(10,2) |
| DISCOUNT_PCT | | NUMBER(4,2) |
| EXAM_SCORE | | NUMBER(5,2) |
| REFERRED_BY | | NUMBER |
| CITY | | VARCHAR2(30) |
| STATUS | | VARCHAR2(12) |

**Answer:**

```sql
CREATE TABLE student_enrollment (
    student_id       NUMBER          PRIMARY KEY,
    student_name     VARCHAR2(100)   NOT NULL,
    email            VARCHAR2(100)   UNIQUE,
    phone_number     VARCHAR2(15),
    course           VARCHAR2(30)    NOT NULL,
    batch            VARCHAR2(15),
    enrollment_date  DATE            NOT NULL,
    fees_total       NUMBER(10,2)    CHECK (fees_total > 0),
    fees_paid        NUMBER(10,2)    DEFAULT 0,
    discount_pct     NUMBER(4,2),
    exam_score       NUMBER(5,2),
    referred_by      NUMBER,
    city             VARCHAR2(30),
    status           VARCHAR2(12)    CHECK (status IN ('ACTIVE','COMPLETED','DROPPED')),
    CONSTRAINT fk_referrer FOREIGN KEY (referred_by) REFERENCES student_enrollment(student_id)
);
```

_Explanation:_ `fk_referrer` is a **self-referencing foreign key** — `referred_by` points back at `student_id` in this exact same table. This single design decision is what lets one table demonstrate every JOIN type later, because "another row in this table" can play the role of a related record. Constraints are included upfront this time since Data Integrity was covered in an earlier case study — the focus here is Functions, Clauses, and JOINs.

---

### Question 2 — Load the Initial Dataset

**Business Question:** Load Bright Path's current 16 enrolled students in one batch. Front-desk staff enter names inconsistently (some in ALL CAPS, some lowercase) — load it exactly as received; cleaning it up is a job for SQL functions, which is coming up next.

**Expected Output:**

```
16 rows created.
```

**Answer:**

```sql
INSERT ALL
  INTO student_enrollment VALUES (501,'AARAV KRISHNAN','aarav.krishnan@brightpath.com','9880011201','DATA SCIENCE','MORNING',DATE '2024-01-10',60000,60000,10,88,NULL,'Chennai','COMPLETED')
  INTO student_enrollment VALUES (502,'divya menon','divya.menon@brightpath.com','9880011202','DATA SCIENCE','MORNING',DATE '2024-01-12',60000,45000,NULL,76,501,'Chennai','ACTIVE')
  INTO student_enrollment VALUES (503,'Rohit Nair','rohit.nair@brightpath.com','9880011203','DATA SCIENCE','EVENING',DATE '2024-02-01',60000,60000,5,92,501,'Coimbatore','COMPLETED')
  INTO student_enrollment VALUES (504,'SANYA KAPOOR','sanya.kapoor@brightpath.com','9880011204','DATA SCIENCE','EVENING',DATE '2024-02-15',60000,30000,NULL,NULL,NULL,'Bengaluru','ACTIVE')
  INTO student_enrollment VALUES (505,'kiran patel','kiran.patel@brightpath.com','9880011205','WEB DEVELOPMENT','MORNING',DATE '2024-01-05',45000,45000,15,81,NULL,'Chennai','COMPLETED')
  INTO student_enrollment VALUES (506,'Meera Iyer','meera.iyer@brightpath.com','9880011206','WEB DEVELOPMENT','MORNING',DATE '2024-01-20',45000,45000,NULL,69,505,'Chennai','COMPLETED')
  INTO student_enrollment VALUES (507,'FARHAN SHEIKH','farhan.sheikh@brightpath.com','9880011207','WEB DEVELOPMENT','WEEKEND',DATE '2024-03-01',45000,20000,NULL,NULL,505,'Hyderabad','ACTIVE')
  INTO student_enrollment VALUES (508,'priyanka das','priyanka.das@brightpath.com','9880011208','WEB DEVELOPMENT','WEEKEND',DATE '2024-03-10',45000,45000,10,74,NULL,'Kolkata','ACTIVE')
  INTO student_enrollment VALUES (509,'Vikas Shetty','vikas.shetty@brightpath.com','9880011209','CLOUD COMPUTING','EVENING',DATE '2024-01-18',70000,70000,NULL,90,NULL,'Chennai','COMPLETED')
  INTO student_enrollment VALUES (510,'ANUSHKA RAO','anushka.rao@brightpath.com','9880011210','CLOUD COMPUTING','EVENING',DATE '2024-02-05',70000,35000,5,58,509,'Pune','DROPPED')
  INTO student_enrollment VALUES (511,'devansh gupta','devansh.gupta@brightpath.com','9880011211','CLOUD COMPUTING','MORNING',DATE '2024-02-20',70000,70000,NULL,85,NULL,'Chennai','ACTIVE')
  INTO student_enrollment VALUES (512,'Ishita Bose','ishita.bose@brightpath.com','9880011212','CLOUD COMPUTING','MORNING',DATE '2024-03-05',70000,50000,NULL,NULL,509,'Chennai','ACTIVE')
  INTO student_enrollment VALUES (513,'NIKHIL MENON','nikhil.menon@brightpath.com','9880011213','DIGITAL MARKETING','WEEKEND',DATE '2024-01-25',35000,35000,20,79,NULL,'Chennai','COMPLETED')
  INTO student_enrollment VALUES (514,'pooja varma','pooja.varma@brightpath.com','9880011214','DIGITAL MARKETING','WEEKEND',DATE '2024-02-10',35000,35000,NULL,79,513,'Chennai','COMPLETED')
  INTO student_enrollment VALUES (515,'Aditya Ghosh','aditya.ghosh@brightpath.com','9880011215','DIGITAL MARKETING','EVENING',DATE '2024-03-15',35000,17500,NULL,NULL,513,'Kolkata','ACTIVE')
  INTO student_enrollment VALUES (516,'NEHA JOSHI','neha.joshi@brightpath.com','9880011216','DIGITAL MARKETING','EVENING',DATE '2024-03-20',35000,35000,10,65,NULL,'Chennai','ACTIVE')
SELECT * FROM dual;
```

**Reference — raw data as loaded (note inconsistent name casing; this is intentional and gets fixed in Question 4):**

| ID | Name (raw) | Course | Batch | Fees Total | Fees Paid | Disc% | Score | Referred By | Status |
|---|---|---|---|---|---|---|---|---|---|
| 501 | AARAV KRISHNAN | DATA SCIENCE | MORNING | 60000 | 60000 | 10 | 88 | — | COMPLETED |
| 502 | divya menon | DATA SCIENCE | MORNING | 60000 | 45000 | — | 76 | 501 | ACTIVE |
| 503 | Rohit Nair | DATA SCIENCE | EVENING | 60000 | 60000 | 5 | 92 | 501 | COMPLETED |
| 504 | SANYA KAPOOR | DATA SCIENCE | EVENING | 60000 | 30000 | — | — | — | ACTIVE |
| 505 | kiran patel | WEB DEVELOPMENT | MORNING | 45000 | 45000 | 15 | 81 | — | COMPLETED |
| 506 | Meera Iyer | WEB DEVELOPMENT | MORNING | 45000 | 45000 | — | 69 | 505 | COMPLETED |
| 507 | FARHAN SHEIKH | WEB DEVELOPMENT | WEEKEND | 45000 | 20000 | — | — | 505 | ACTIVE |
| 508 | priyanka das | WEB DEVELOPMENT | WEEKEND | 45000 | 45000 | 10 | 74 | — | ACTIVE |
| 509 | Vikas Shetty | CLOUD COMPUTING | EVENING | 70000 | 70000 | — | 90 | — | COMPLETED |
| 510 | ANUSHKA RAO | CLOUD COMPUTING | EVENING | 70000 | 35000 | 5 | 58 | 509 | DROPPED |
| 511 | devansh gupta | CLOUD COMPUTING | MORNING | 70000 | 70000 | — | 85 | — | ACTIVE |
| 512 | Ishita Bose | CLOUD COMPUTING | MORNING | 70000 | 50000 | — | — | 509 | ACTIVE |
| 513 | NIKHIL MENON | DIGITAL MARKETING | WEEKEND | 35000 | 35000 | 20 | 79 | — | COMPLETED |
| 514 | pooja varma | DIGITAL MARKETING | WEEKEND | 35000 | 35000 | — | 79 | 513 | COMPLETED |
| 515 | Aditya Ghosh | DIGITAL MARKETING | EVENING | 35000 | 17500 | — | — | 513 | ACTIVE |
| 516 | NEHA JOSHI | DIGITAL MARKETING | EVENING | 35000 | 35000 | 10 | 65 | — | ACTIVE |

_Explanation:_ Oracle's multi-row insert idiom is `INSERT ALL ... SELECT * FROM dual` — each `INTO` fires once. This table is the backbone of **every single question below**, so keep this reference handy.

---

## PART 1 — SQL Functions Classification

### Question 3 — Deterministic vs Non-Deterministic Functions

**Business Question:** Build a live "days enrolled" report for all currently `ACTIVE` students, refreshed fresh every time it's viewed on the dashboard.

**Expected Output:** *(Illustrative — assumes the query is run on 2024-04-01. Run it yourself today and you'll get different numbers — that's the entire point.)*

```
8 rows selected.
```

| STUDENT_NAME | ENROLLMENT_DATE | DAYS_ENROLLED |
|---|---|---|
| Divya Menon | 2024-01-12 | 80 |
| Sanya Kapoor | 2024-02-15 | 46 |
| Devansh Gupta | 2024-02-20 | 41 |
| Farhan Sheikh | 2024-03-01 | 31 |
| Ishita Bose | 2024-03-05 | 27 |
| Priyanka Das | 2024-03-10 | 22 |
| Aditya Ghosh | 2024-03-15 | 17 |
| Neha Joshi | 2024-03-20 | 12 |

**Answer:**

```sql
SELECT student_name, enrollment_date,
       TRUNC(SYSDATE) - enrollment_date AS days_enrolled
FROM student_enrollment
WHERE status = 'ACTIVE'
ORDER BY days_enrolled DESC;
```

_Explanation:_ `SYSDATE` is **non-deterministic** — it returns a different value every time it's called, so this query's output silently changes day to day even though nothing in the table changed. `TRUNC()`, by contrast, is **deterministic** — given the same input, it always returns the same output, no matter when or how many times you call it. This distinction matters most for indexing and function-based views: Oracle can't cache or index a non-deterministic function's result the way it can a deterministic one.

---

### Question 4 — Scalar Functions: Fixing Inconsistent Data

**Business Question:** The front office is embarrassed by inconsistent name casing on printed ID cards. Show every student's name properly capitalized (Title Case), and course name in uppercase for visual consistency.

**Expected Output:**

```
16 rows selected.
```

| STUDENT_NAME (cleaned) | COURSE (cleaned) |
|---|---|
| Aarav Krishnan | DATA SCIENCE |
| Divya Menon | DATA SCIENCE |
| Rohit Nair | DATA SCIENCE |
| Sanya Kapoor | DATA SCIENCE |
| Kiran Patel | WEB DEVELOPMENT |
| Meera Iyer | WEB DEVELOPMENT |
| Farhan Sheikh | WEB DEVELOPMENT |
| Priyanka Das | WEB DEVELOPMENT |
| Vikas Shetty | CLOUD COMPUTING |
| Anushka Rao | CLOUD COMPUTING |
| Devansh Gupta | CLOUD COMPUTING |
| Ishita Bose | CLOUD COMPUTING |
| Nikhil Menon | DIGITAL MARKETING |
| Pooja Varma | DIGITAL MARKETING |
| Aditya Ghosh | DIGITAL MARKETING |
| Neha Joshi | DIGITAL MARKETING |

**Answer:**

```sql
SELECT INITCAP(student_name) AS student_name, UPPER(course) AS course
FROM student_enrollment
ORDER BY course, student_name;

-- The front office liked this so much they asked to fix it permanently:
UPDATE student_enrollment
SET student_name = INITCAP(student_name);
```

_Explanation:_ `INITCAP` and `UPPER` are **scalar functions** — they operate on and return a value for *each individual row*, unlike an aggregate function which collapses many rows into one. The follow-up `UPDATE` is a nice real-world touch: once a report proves a transformation is useful, it's common to bake it into the stored data instead of recalculating it every time. **From this point on in the case study, `student_name` is clean Title Case in every table shown.**

---

### Question 5 — Scalar Functions: Payment Completion Check

**Business Question:** Finance wants to know exactly who still owes money, and what percentage of their fees they've paid so far.

**Expected Output:**

```
6 rows selected.
```

| STUDENT_NAME | FEES_PAID | FEES_TOTAL | PAYMENT_PCT |
|---|---|---|---|
| Farhan Sheikh | 20000 | 45000 | 44.44 |
| Sanya Kapoor | 30000 | 60000 | 50.00 |
| Anushka Rao | 35000 | 70000 | 50.00 |
| Aditya Ghosh | 17500 | 35000 | 50.00 |
| Ishita Bose | 50000 | 70000 | 71.43 |
| Divya Menon | 45000 | 60000 | 75.00 |

**Answer:**

```sql
SELECT student_name, fees_paid, fees_total,
       ROUND(fees_paid / fees_total * 100, 2) AS payment_pct
FROM student_enrollment
WHERE fees_paid < fees_total
ORDER BY payment_pct ASC, student_name;
```

_Explanation:_ `ROUND()` is a scalar **mathematical** function applied row-by-row — every row gets its own independently calculated percentage.

---

### Question 6 — Aggregate Functions: Institute Snapshot

**Business Question:** Leadership wants a one-line dashboard snapshot: total students enrolled, total fees collected institute-wide, and the average, highest, and lowest exam score among students who've actually been graded.

**Expected Output:**

```
1 row selected.
```

| TOTAL_STUDENTS | TOTAL_FEES_COLLECTED | AVG_SCORE | HIGHEST_SCORE | LOWEST_SCORE |
|---|---|---|---|---|
| 16 | 697500 | 78.00 | 92 | 58 |

**Answer:**

```sql
SELECT COUNT(*)              AS total_students,
       SUM(fees_paid)        AS total_fees_collected,
       ROUND(AVG(exam_score), 2) AS avg_score,
       MAX(exam_score)       AS highest_score,
       MIN(exam_score)       AS lowest_score
FROM student_enrollment;
```

_Explanation:_ These are **aggregate functions** — they collapse all 16 rows into a single summary row. Notice `COUNT(*)` returns `16` (every row), but `AVG`, `MAX`, and `MIN` silently **ignore NULLs** — only the 12 students with an actual `exam_score` contribute to those three. If you ran `COUNT(exam_score)` instead of `COUNT(*)`, you'd get `12`, not `16` — a very common source of confusion.

---

### Question 7 — String Functions: Personalized Greetings

**Business Question:** Marketing wants to send a personalized reminder email to every currently active student: "Dear `<First Name>`,".

**Expected Output:**

```
8 rows selected.
```

| GREETING |
|---|
| Dear Divya, |
| Dear Sanya, |
| Dear Farhan, |
| Dear Priyanka, |
| Dear Devansh, |
| Dear Ishita, |
| Dear Aditya, |
| Dear Neha, |

**Answer:**

```sql
SELECT 'Dear ' || SUBSTR(student_name, 1, INSTR(student_name, ' ') - 1) || ',' AS greeting
FROM student_enrollment
WHERE status = 'ACTIVE'
ORDER BY student_name;
```

_Explanation:_ `INSTR(student_name, ' ')` locates the position of the first space; `SUBSTR` then extracts everything before it — the first name. `||` is Oracle's string concatenation operator.

---

### Question 8 — String Functions: Privacy-Masked Contact Export

**Business Question:** A third-party analytics vendor needs student contact data, but compliance requires masking phone numbers (show only the last 4 digits) and stripping emails down to just the username portion — no full email or phone number should leave the building.

**Expected Output:** *(shown for the Data Science and Web Development cohorts — same pattern applies to all 16)*

```
8 rows selected.
```

| STUDENT_NAME | EMAIL_USERNAME | MASKED_PHONE |
|---|---|---|
| Aarav Krishnan | aarav.krishnan | XXXXXX1201 |
| Divya Menon | divya.menon | XXXXXX1202 |
| Rohit Nair | rohit.nair | XXXXXX1203 |
| Sanya Kapoor | sanya.kapoor | XXXXXX1204 |
| Kiran Patel | kiran.patel | XXXXXX1205 |
| Meera Iyer | meera.iyer | XXXXXX1206 |
| Farhan Sheikh | farhan.sheikh | XXXXXX1207 |
| Priyanka Das | priyanka.das | XXXXXX1208 |

**Answer:**

```sql
SELECT student_name,
       SUBSTR(email, 1, INSTR(email, '@') - 1) AS email_username,
       'XXXXXX' || SUBSTR(phone_number, -4)     AS masked_phone
FROM student_enrollment
WHERE course IN ('DATA SCIENCE', 'WEB DEVELOPMENT')
ORDER BY student_id;
```

_Explanation:_ `SUBSTR(phone_number, -4)` uses a **negative start position** — a lesser-known but very handy `SUBSTR` trick that counts from the *end* of the string, grabbing the last 4 characters regardless of the string's total length.

---

### Question 9 — Mathematical Functions: Installment Planning

**Business Question:** For every student with an outstanding balance, split what they owe into 3 equal monthly installments. Since you can't collect a fraction of a rupee, round each installment **up**, and show what tiny remainder (if any) is left over after 3 rounded-up installments — finance wants to fold that remainder into the final installment.

**Expected Output:**

```
6 rows selected.
```

| STUDENT_NAME | BALANCE_DUE | INSTALLMENT (rounded up) | REMAINDER |
|---|---|---|---|
| Divya Menon | 15000 | 5000 | 0 |
| Sanya Kapoor | 30000 | 10000 | 0 |
| Farhan Sheikh | 25000 | 8334 | 1 |
| Anushka Rao | 35000 | 11667 | 2 |
| Ishita Bose | 20000 | 6667 | 2 |
| Aditya Ghosh | 17500 | 5834 | 1 |

**Answer:**

```sql
SELECT student_name,
       (fees_total - fees_paid)                     AS balance_due,
       CEIL((fees_total - fees_paid) / 3)            AS installment,
       MOD((fees_total - fees_paid), 3)              AS remainder
FROM student_enrollment
WHERE fees_paid < fees_total
ORDER BY balance_due;
```

_Explanation:_ `CEIL` rounds up to the nearest whole rupee (never round installments down — the institute would collect less than what's owed). `MOD` returns the remainder of integer division, showing exactly how many rupees would be "lost" if you multiplied the rounded installment by 3 flat — useful for finance to adjust the final payment.

---

### Question 10 — Miscellaneous Functions: COALESCE for Safe Defaults

**Business Question:** Generate a report card for every student that never shows a blank cell — ungraded students should show "Not Graded Yet" instead of `NULL`, and students with no discount should show `0` instead of blank.

**Expected Output:**

```
16 rows selected.
```

| STUDENT_NAME | SCORE_DISPLAY | DISCOUNT_PCT (defaulted) |
|---|---|---|
| Aarav Krishnan | 88 | 10 |
| Divya Menon | 76 | 0 |
| Rohit Nair | 92 | 5 |
| Sanya Kapoor | Not Graded Yet | 0 |
| Kiran Patel | 81 | 15 |
| Meera Iyer | 69 | 0 |
| Farhan Sheikh | Not Graded Yet | 0 |
| Priyanka Das | 74 | 10 |
| Vikas Shetty | 90 | 0 |
| Anushka Rao | 58 | 5 |
| Devansh Gupta | 85 | 0 |
| Ishita Bose | Not Graded Yet | 0 |
| Nikhil Menon | 79 | 20 |
| Pooja Varma | 79 | 0 |
| Aditya Ghosh | Not Graded Yet | 0 |
| Neha Joshi | 65 | 10 |

**Answer:**

```sql
SELECT student_name,
       COALESCE(TO_CHAR(exam_score), 'Not Graded Yet') AS score_display,
       COALESCE(discount_pct, 0)                       AS discount_pct
FROM student_enrollment
ORDER BY student_id;
```

_Explanation:_ `COALESCE` returns the first non-`NULL` value from its argument list — here just 2 arguments, but it accepts any number, checked left to right, making it a generalized (and ANSI-standard) replacement for Oracle's older `NVL`.

---

### Question 11 — Miscellaneous Functions: NULLIF for Data Quality Flags

**Business Question:** QA suspects a data-entry shortcut bug: for some **currently active** (not yet completed) students, `fees_paid` was accidentally set exactly equal to `fees_total` even though they haven't actually finished paying. Flag any such case as "needs verification" instead of trusting the number blindly.

**Expected Output:**

```
8 rows selected.
```

| STUDENT_NAME | STATUS | FEES_PAID | FEES_TOTAL | VERIFIED_FEES_PAID |
|---|---|---|---|---|
| Divya Menon | ACTIVE | 45000 | 60000 | 45000 |
| Sanya Kapoor | ACTIVE | 30000 | 60000 | 30000 |
| Farhan Sheikh | ACTIVE | 20000 | 45000 | 20000 |
| Priyanka Das | ACTIVE | 45000 | 45000 | *(NULL — needs verification)* |
| Devansh Gupta | ACTIVE | 70000 | 70000 | *(NULL — needs verification)* |
| Ishita Bose | ACTIVE | 50000 | 70000 | 50000 |
| Aditya Ghosh | ACTIVE | 17500 | 35000 | 17500 |
| Neha Joshi | ACTIVE | 35000 | 35000 | *(NULL — needs verification)* |

**Answer:**

```sql
SELECT student_name, status, fees_paid, fees_total,
       NULLIF(fees_paid, fees_total) AS verified_fees_paid
FROM student_enrollment
WHERE status = 'ACTIVE'
ORDER BY student_id;
```

_Explanation:_ `NULLIF(a, b)` returns `NULL` if `a = b`, otherwise it returns `a`. Here, exactly the pattern QA suspected — a *currently active* student whose paid amount exactly equals the total — becomes `NULL`, visually flagging Priyanka Das, Devansh Gupta, and Neha Joshi for manual review, while everyone else's real balance passes through untouched.

---

### Question 12 — Analytical Functions: Course Merit Ranking

**Business Question:** Build a merit list ranking every student within their own course by exam score, highest first. Students who haven't been graded yet must appear at the bottom of their course's list, not break the ranking.

**Expected Output:**

```
16 rows selected.
```

| COURSE | STUDENT_NAME | EXAM_SCORE | COURSE_RANK |
|---|---|---|---|
| CLOUD COMPUTING | Vikas Shetty | 90 | 1 |
| CLOUD COMPUTING | Devansh Gupta | 85 | 2 |
| CLOUD COMPUTING | Anushka Rao | 58 | 3 |
| CLOUD COMPUTING | Ishita Bose | *(null)* | 4 |
| DATA SCIENCE | Rohit Nair | 92 | 1 |
| DATA SCIENCE | Aarav Krishnan | 88 | 2 |
| DATA SCIENCE | Divya Menon | 76 | 3 |
| DATA SCIENCE | Sanya Kapoor | *(null)* | 4 |
| DIGITAL MARKETING | Nikhil Menon | 79 | 1 |
| DIGITAL MARKETING | Pooja Varma | 79 | 1 |
| DIGITAL MARKETING | Neha Joshi | 65 | 3 |
| DIGITAL MARKETING | Aditya Ghosh | *(null)* | 4 |
| WEB DEVELOPMENT | Kiran Patel | 81 | 1 |
| WEB DEVELOPMENT | Priyanka Das | 74 | 2 |
| WEB DEVELOPMENT | Meera Iyer | 69 | 3 |
| WEB DEVELOPMENT | Farhan Sheikh | *(null)* | 4 |

**Answer:**

```sql
SELECT course, student_name, exam_score,
       RANK() OVER (PARTITION BY course ORDER BY exam_score DESC NULLS LAST) AS course_rank
FROM student_enrollment
ORDER BY course, course_rank;
```

_Explanation:_ This is a **window (analytical) function** — unlike `GROUP BY`, it doesn't collapse rows; every student still gets their own row, just with an extra rank number computed *within their `PARTITION BY course` window*. Notice Digital Marketing: Nikhil Menon and Pooja Varma are **tied at 79**, so both get `RANK = 1`, and the next student (Neha Joshi) jumps straight to `RANK = 3` — `RANK()` leaves a gap equal to the number of ties. If you used `DENSE_RANK()` instead, Neha Joshi would get `2`, with no gap. `ROW_NUMBER()` would instead force the tie apart arbitrarily into `1` and `2`. All three behave identically wherever there's no tie.

---

### Question 13 — Analytical Functions: Running Revenue Total

**Business Question:** Finance wants to see how fee revenue accumulated over time, in enrollment order, as a running (cumulative) total — a simple cash-flow curve for the term.

**Expected Output:**

```
16 rows selected.
```

| ENROLLMENT_DATE | STUDENT_NAME | FEES_PAID | RUNNING_TOTAL |
|---|---|---|---|
| 2024-01-05 | Kiran Patel | 45000 | 45000 |
| 2024-01-10 | Aarav Krishnan | 60000 | 105000 |
| 2024-01-12 | Divya Menon | 45000 | 150000 |
| 2024-01-18 | Vikas Shetty | 70000 | 220000 |
| 2024-01-20 | Meera Iyer | 45000 | 265000 |
| 2024-01-25 | Nikhil Menon | 35000 | 300000 |
| 2024-02-01 | Rohit Nair | 60000 | 360000 |
| 2024-02-05 | Anushka Rao | 35000 | 395000 |
| 2024-02-10 | Pooja Varma | 35000 | 430000 |
| 2024-02-15 | Sanya Kapoor | 30000 | 460000 |
| 2024-02-20 | Devansh Gupta | 70000 | 530000 |
| 2024-03-01 | Farhan Sheikh | 20000 | 550000 |
| 2024-03-05 | Ishita Bose | 50000 | 600000 |
| 2024-03-10 | Priyanka Das | 45000 | 645000 |
| 2024-03-15 | Aditya Ghosh | 17500 | 662500 |
| 2024-03-20 | Neha Joshi | 35000 | 697500 |

**Answer:**

```sql
SELECT enrollment_date, student_name, fees_paid,
       SUM(fees_paid) OVER (ORDER BY enrollment_date) AS running_total
FROM student_enrollment
ORDER BY enrollment_date;
```

_Explanation:_ Adding `ORDER BY` **inside** an `OVER()` clause (with no `PARTITION BY`) turns a plain aggregate into a **running/cumulative** calculation — each row's total includes every row before it. Notice the final `RUNNING_TOTAL` (697500) exactly matches `TOTAL_FEES_COLLECTED` from Question 6 — a good sign your numbers are internally consistent.

---

### Question 14 — Nesting Functions: Formal Name Generator

**Business Question:** The certificate-printing vendor needs names in a compact formal style: first-initial + a period + surname — e.g., "A. Krishnan" — generated automatically from the full name.

**Expected Output:**

```
16 rows selected.
```

| STUDENT_NAME | FORMAL_NAME |
|---|---|
| Aarav Krishnan | A. Krishnan |
| Divya Menon | D. Menon |
| Rohit Nair | R. Nair |
| Sanya Kapoor | S. Kapoor |
| Kiran Patel | K. Patel |
| Meera Iyer | M. Iyer |
| Farhan Sheikh | F. Sheikh |
| Priyanka Das | P. Das |
| Vikas Shetty | V. Shetty |
| Anushka Rao | A. Rao |
| Devansh Gupta | D. Gupta |
| Ishita Bose | I. Bose |
| Nikhil Menon | N. Menon |
| Pooja Varma | P. Varma |
| Aditya Ghosh | A. Ghosh |
| Neha Joshi | N. Joshi |

**Answer:**

```sql
SELECT student_name,
       UPPER(SUBSTR(student_name, 1, 1)) || '. ' ||
       SUBSTR(student_name, INSTR(student_name, ' ') + 1) AS formal_name
FROM student_enrollment
ORDER BY student_id;
```

_Explanation:_ This **nests four function calls** inside one expression: the innermost `INSTR` finds the space, the outer `SUBSTR` calls extract the surname and the first letter, and `UPPER` guarantees the initial is capitalized even if the underlying data drifts. Oracle evaluates from the innermost expression outward — reading nested calls "inside-out" is the trick to untangling any complex expression like this.

---

## PART 2 — Clauses

### Question 15 — GROUP BY: Course-Wise Revenue Report

**Business Question:** Give finance a course-wise breakdown: headcount and total fees collected per course, ranked by revenue.

**Expected Output:**

```
4 rows selected.
```

| COURSE | HEADCOUNT | TOTAL_COLLECTED |
|---|---|---|
| CLOUD COMPUTING | 4 | 225000 |
| DATA SCIENCE | 4 | 195000 |
| WEB DEVELOPMENT | 4 | 155000 |
| DIGITAL MARKETING | 4 | 122500 |

**Answer:**

```sql
SELECT course, COUNT(*) AS headcount, SUM(fees_paid) AS total_collected
FROM student_enrollment
GROUP BY course
ORDER BY total_collected DESC;
```

_Explanation:_ `GROUP BY` collapses all rows sharing the same `course` value into one summary row per course — the fundamental difference from the analytical functions in Questions 12–13, which kept every row visible.

---

### Question 16 — HAVING: Filtering Grouped Results

**Business Question:** The academic team wants to know which courses are performing **above a 75-average** benchmark (among graded students only) for a quality review.

**Expected Output:**

```
2 rows selected.
```

| COURSE | AVG_SCORE |
|---|---|
| DATA SCIENCE | 85.33 |
| CLOUD COMPUTING | 77.67 |

**Answer:**

```sql
SELECT course, ROUND(AVG(exam_score), 2) AS avg_score
FROM student_enrollment
GROUP BY course
HAVING AVG(exam_score) > 75
ORDER BY avg_score DESC;
```

_Explanation:_ `WHERE` filters individual rows **before** grouping; `HAVING` filters **groups**, after aggregation. You cannot write `WHERE AVG(exam_score) > 75` — `WHERE` runs too early in execution to know what an aggregate value even is yet (this exact mistake is the subject of Question 18). Web Development (74.67) and Digital Marketing (74.33) narrowly miss the cutoff.

---

### Question 17 — ORDER BY: Multi-Column Sort

**Business Question:** Print-ready merit book: every student, grouped by course (alphabetically), and within each course sorted by score from highest to lowest — ungraded students last.

**Expected Output:**

```
16 rows selected.
```

| COURSE | STUDENT_NAME | EXAM_SCORE |
|---|---|---|
| CLOUD COMPUTING | Vikas Shetty | 90 |
| CLOUD COMPUTING | Devansh Gupta | 85 |
| CLOUD COMPUTING | Anushka Rao | 58 |
| CLOUD COMPUTING | Ishita Bose | *(null)* |
| DATA SCIENCE | Rohit Nair | 92 |
| DATA SCIENCE | Aarav Krishnan | 88 |
| DATA SCIENCE | Divya Menon | 76 |
| DATA SCIENCE | Sanya Kapoor | *(null)* |
| DIGITAL MARKETING | Nikhil Menon | 79 |
| DIGITAL MARKETING | Pooja Varma | 79 |
| DIGITAL MARKETING | Neha Joshi | 65 |
| DIGITAL MARKETING | Aditya Ghosh | *(null)* |
| WEB DEVELOPMENT | Kiran Patel | 81 |
| WEB DEVELOPMENT | Priyanka Das | 74 |
| WEB DEVELOPMENT | Meera Iyer | 69 |
| WEB DEVELOPMENT | Farhan Sheikh | *(null)* |

**Answer:**

```sql
SELECT course, student_name, exam_score
FROM student_enrollment
ORDER BY course ASC, exam_score DESC NULLS LAST, student_id;
```

_Explanation:_ Same row shape as Question 12, but notice there's **no rank number** — `ORDER BY` only controls display sequence, it doesn't compute or label anything. That labeling is exactly what `RANK()` added back in Question 12. `student_id` is added as a third sort key purely to make the Nikhil/Pooja tie deterministic.

---

### Question 18 — Order of Execution: Debugging a Broken Query

**Business Question:** An analyst wrote this query and it fails. Explain why using the logical order clauses execute in, and fix it:

```sql
SELECT course, AVG(exam_score) AS avg_score
FROM student_enrollment
WHERE avg_score > 75
GROUP BY course;
```

**Expected Output:**

```
ORA-00904: "AVG_SCORE": invalid identifier
```

**Corrected query's expected output:**

```
2 rows selected.
```

| COURSE | AVG_SCORE |
|---|---|
| DATA SCIENCE | 85.33 |
| CLOUD COMPUTING | 77.67 |

**Answer:**

```sql
-- Fixed: use HAVING, not WHERE, for a condition on an aggregate
SELECT course, AVG(exam_score) AS avg_score
FROM student_enrollment
GROUP BY course
HAVING AVG(exam_score) > 75
ORDER BY avg_score DESC;
```

_Explanation:_ Oracle's actual logical execution order for a `SELECT` is:

```
1. FROM        (identify the source table)
2. WHERE       (filter individual rows)
3. GROUP BY    (collapse into groups)
4. HAVING      (filter groups)
5. SELECT      (compute the final column list, including aliases)
6. ORDER BY    (sort the final result)
```

`WHERE` runs at step 2 — **before** the `SELECT` list (step 5) has even been evaluated, so the alias `avg_score` doesn't exist yet as far as `WHERE` is concerned, and it certainly doesn't know about an aggregate that hasn't been computed. `HAVING` runs at step 4, safely after grouping, which is why it — and only it — can reference `AVG(exam_score)` directly.

---

### Question 19 — GROUP BY on Multiple Columns

**Business Question:** Operations needs finer-grained capacity planning: headcount and average fees paid, broken down by **course AND batch timing** together, not just by course.

**Expected Output:**

```
8 rows selected.
```

| COURSE | BATCH | HEADCOUNT | AVG_FEES_PAID |
|---|---|---|---|
| CLOUD COMPUTING | EVENING | 2 | 52500.00 |
| CLOUD COMPUTING | MORNING | 2 | 60000.00 |
| DATA SCIENCE | EVENING | 2 | 45000.00 |
| DATA SCIENCE | MORNING | 2 | 52500.00 |
| DIGITAL MARKETING | EVENING | 2 | 26250.00 |
| DIGITAL MARKETING | WEEKEND | 2 | 35000.00 |
| WEB DEVELOPMENT | MORNING | 2 | 45000.00 |
| WEB DEVELOPMENT | WEEKEND | 2 | 32500.00 |

**Answer:**

```sql
SELECT course, batch, COUNT(*) AS headcount,
       ROUND(AVG(fees_paid), 2) AS avg_fees_paid
FROM student_enrollment
GROUP BY course, batch
ORDER BY course, batch;
```

_Explanation:_ `GROUP BY course, batch` creates one group per **unique combination** of both columns — notice every course currently runs exactly 2 of its 3 possible batch timings, each with exactly 2 students. (Which batch timing is *missing* per course becomes relevant again in Question 25's `CROSS JOIN`.)

---

### Question 20 — WHERE + GROUP BY + HAVING Together

**Business Question:** Excluding students who dropped out entirely, identify which courses have **more than one** student who has fully `COMPLETED` — a sign of a genuinely strong course track.

**Expected Output:**

```
3 rows selected.
```

| COURSE | COMPLETED_COUNT |
|---|---|
| DATA SCIENCE | 2 |
| WEB DEVELOPMENT | 2 |
| DIGITAL MARKETING | 2 |

**Answer:**

```sql
SELECT course, COUNT(*) AS completed_count
FROM student_enrollment
WHERE status = 'COMPLETED'
GROUP BY course
HAVING COUNT(*) > 1
ORDER BY completed_count DESC, course;
```

_Explanation:_ All three clauses working together, in their actual execution order: `WHERE` first strips out everyone except `COMPLETED` students, `GROUP BY` then buckets what's left by course, and `HAVING` finally keeps only the buckets with more than 1 row. Cloud Computing silently drops out of this result — it only has 1 `COMPLETED` student (Vikas Shetty; Anushka Rao dropped out, and the other two are still active).

---

## PART 3 — JOINs & JOIN Styles

Every join below is a **self-join** — `student_enrollment` joined to itself, using two aliases, most commonly to connect a referred student to their referrer. A short refresher before diving in: an **OUTER JOIN** (as opposed to an INNER JOIN) keeps rows from one or both sides even when there's no match, filling the missing side with `NULL` — and that's exactly what LEFT, RIGHT, and FULL OUTER JOIN each do differently, covered in Questions 28–30.

### Question 21 — Theta Style JOIN (old syntax, non-equality condition)

**Business Question:** Set up a peer-mentoring pilot: pair students in the **same course** where one student's score is **more than 10 points higher** than another's, so the stronger student can mentor the weaker one.

**Expected Output:**

```
7 rows selected.
```

| COURSE | MENTOR | MENTOR_SCORE | MENTEE | MENTEE_SCORE |
|---|---|---|---|---|
| CLOUD COMPUTING | Devansh Gupta | 85 | Anushka Rao | 58 |
| CLOUD COMPUTING | Vikas Shetty | 90 | Anushka Rao | 58 |
| DATA SCIENCE | Aarav Krishnan | 88 | Divya Menon | 76 |
| DATA SCIENCE | Rohit Nair | 92 | Divya Menon | 76 |
| DIGITAL MARKETING | Nikhil Menon | 79 | Neha Joshi | 65 |
| DIGITAL MARKETING | Pooja Varma | 79 | Neha Joshi | 65 |
| WEB DEVELOPMENT | Kiran Patel | 81 | Meera Iyer | 69 |

**Answer:**

```sql
SELECT s1.course, s1.student_name AS mentor, s1.exam_score AS mentor_score,
       s2.student_name AS mentee, s2.exam_score AS mentee_score
FROM student_enrollment s1, student_enrollment s2
WHERE s1.course = s2.course
  AND s1.exam_score > s2.exam_score + 10
ORDER BY s1.course, mentor;
```

_Explanation:_ This is the **old-style, comma-separated FROM** syntax, with the join condition living entirely in `WHERE` — the syntax style historically called a "Theta join." A **theta join** is any join whose condition uses a comparison operator (θ) other than pure equality — here, `>`. Note it needs *two* conditions in `WHERE`: one to keep the pairing within the same course, and one to express the actual "more than 10 points higher" business rule.

---

### Question 22 — Oracle/ANSI Style: JOIN ... ON

**Business Question:** Rewrite the referral relationship using modern ANSI syntax: every student who joined via referral, alongside who referred them.

**Expected Output:**

```
8 rows selected.
```

| REFERRED_BY_NAME | REFERRED_STUDENT |
|---|---|
| Aarav Krishnan | Divya Menon |
| Aarav Krishnan | Rohit Nair |
| Kiran Patel | Farhan Sheikh |
| Kiran Patel | Meera Iyer |
| Nikhil Menon | Aditya Ghosh |
| Nikhil Menon | Pooja Varma |
| Vikas Shetty | Anushka Rao |
| Vikas Shetty | Ishita Bose |

**Answer:**

```sql
SELECT s2.student_name AS referred_by_name, s1.student_name AS referred_student
FROM student_enrollment s1
JOIN student_enrollment s2 ON s1.referred_by = s2.student_id
ORDER BY referred_by_name, referred_student;
```

_Explanation:_ Functionally identical logic to a Theta-style equality join, but the `JOIN ... ON` syntax cleanly separates the join condition from any filtering condition (there isn't one here) — the standard Oracle/ANSI style used in virtually all modern SQL.

---

### Question 23 — Oracle/ANSI Style: JOIN ... USING

**Business Question:** Attach each graded student's own score alongside their course's average score, so they can see exactly how they compare.

**Expected Output:**

```
12 rows selected.
```

| COURSE | STUDENT_NAME | EXAM_SCORE | COURSE_AVG_SCORE |
|---|---|---|---|
| CLOUD COMPUTING | Vikas Shetty | 90 | 77.67 |
| CLOUD COMPUTING | Devansh Gupta | 85 | 77.67 |
| CLOUD COMPUTING | Anushka Rao | 58 | 77.67 |
| DATA SCIENCE | Rohit Nair | 92 | 85.33 |
| DATA SCIENCE | Aarav Krishnan | 88 | 85.33 |
| DATA SCIENCE | Divya Menon | 76 | 85.33 |
| DIGITAL MARKETING | Nikhil Menon | 79 | 74.33 |
| DIGITAL MARKETING | Pooja Varma | 79 | 74.33 |
| DIGITAL MARKETING | Neha Joshi | 65 | 74.33 |
| WEB DEVELOPMENT | Kiran Patel | 81 | 74.67 |
| WEB DEVELOPMENT | Priyanka Das | 74 | 74.67 |
| WEB DEVELOPMENT | Meera Iyer | 69 | 74.67 |

**Answer:**

```sql
SELECT s.course, s.student_name, s.exam_score, cs.course_avg_score
FROM student_enrollment s
JOIN (SELECT course, ROUND(AVG(exam_score), 2) AS course_avg_score
      FROM student_enrollment
      GROUP BY course) cs
USING (course)
WHERE s.exam_score IS NOT NULL
ORDER BY s.course, s.exam_score DESC;
```

_Explanation:_ `USING (course)` is a shorthand for `ON s.course = cs.course` — it's only legal when **both sides share a column with the exact same name**, which is true here since the derived subquery's column was deliberately named `course` to match. Unlike `ON`, a `USING`-joined column appears only **once** in the result set (you can't prefix it with a table alias).

---

### Question 24 — EQUI-JOIN

**Business Question:** Using the traditional comma-separated syntax again, but this time with a pure equality condition, check whether referrals tend to stay within the same course or cross over into a different one — show both courses side by side.

**Expected Output:**

```
8 rows selected.
```

| REFERRER_NAME | REFERRER_COURSE | REFERRED_STUDENT | REFERRED_COURSE |
|---|---|---|---|
| Aarav Krishnan | DATA SCIENCE | Divya Menon | DATA SCIENCE |
| Aarav Krishnan | DATA SCIENCE | Rohit Nair | DATA SCIENCE |
| Kiran Patel | WEB DEVELOPMENT | Farhan Sheikh | WEB DEVELOPMENT |
| Kiran Patel | WEB DEVELOPMENT | Meera Iyer | WEB DEVELOPMENT |
| Nikhil Menon | DIGITAL MARKETING | Aditya Ghosh | DIGITAL MARKETING |
| Nikhil Menon | DIGITAL MARKETING | Pooja Varma | DIGITAL MARKETING |
| Vikas Shetty | CLOUD COMPUTING | Anushka Rao | CLOUD COMPUTING |
| Vikas Shetty | CLOUD COMPUTING | Ishita Bose | CLOUD COMPUTING |

**Answer:**

```sql
SELECT s2.student_name AS referrer_name, s2.course AS referrer_course,
       s1.student_name AS referred_student, s1.course AS referred_course
FROM student_enrollment s1, student_enrollment s2
WHERE s1.referred_by = s2.student_id
ORDER BY referrer_name, referred_student;
```

_Explanation:_ An **equi-join** is simply a theta join whose comparison operator is specifically `=` — the most common join type by far. The business finding here is genuinely useful: **every single referral stayed within the same course** — nobody referred a friend into a different course than their own.

---

### Question 25 — CROSS JOIN

**Business Question:** Operations wants a full scheduling grid of every course crossed with every possible batch timing, so they can spot which combinations have zero students currently enrolled (open capacity).

**Expected Output:**

```
12 rows selected.
```

| COURSE | BATCH |
|---|---|
| CLOUD COMPUTING | EVENING |
| CLOUD COMPUTING | MORNING |
| CLOUD COMPUTING | WEEKEND |
| DATA SCIENCE | EVENING |
| DATA SCIENCE | MORNING |
| DATA SCIENCE | WEEKEND |
| DIGITAL MARKETING | EVENING |
| DIGITAL MARKETING | MORNING |
| DIGITAL MARKETING | WEEKEND |
| WEB DEVELOPMENT | EVENING |
| WEB DEVELOPMENT | MORNING |
| WEB DEVELOPMENT | WEEKEND |

**Answer:**

```sql
SELECT c.course, b.batch
FROM (SELECT DISTINCT course FROM student_enrollment) c
CROSS JOIN (SELECT DISTINCT batch FROM student_enrollment) b
ORDER BY c.course, b.batch;
```

_Explanation:_ A `CROSS JOIN` produces the full **Cartesian product** — every row from the first set paired with every row from the second, with no condition at all (4 courses × 3 batches = 12 rows). Comparing this grid against Question 19's actual enrollment shows exactly 4 empty cells: Data Science has no WEEKEND batch, Web Development has no EVENING batch, Cloud Computing has no WEEKEND batch, and Digital Marketing has no MORNING batch — each course is missing exactly one timing slot.

---

### Question 26 — INNER JOIN (explicit keyword, multi-condition)

**Business Question:** Set up a "buddy program" pairing referred students with their referrer — but **only** when they also share the exact same batch timing, so they can realistically meet in person.

**Expected Output:**

```
4 rows selected.
```

| REFERRER | REFERRER_BATCH | STUDENT | STUDENT_BATCH |
|---|---|---|---|
| Aarav Krishnan | MORNING | Divya Menon | MORNING |
| Kiran Patel | MORNING | Meera Iyer | MORNING |
| Nikhil Menon | WEEKEND | Pooja Varma | WEEKEND |
| Vikas Shetty | EVENING | Anushka Rao | EVENING |

**Answer:**

```sql
SELECT s2.student_name AS referrer, s2.batch AS referrer_batch,
       s1.student_name AS student, s1.batch AS student_batch
FROM student_enrollment s1
INNER JOIN student_enrollment s2
        ON s1.referred_by = s2.student_id
       AND s1.batch = s2.batch
ORDER BY referrer;
```

_Explanation:_ The explicit `INNER JOIN` keyword (functionally identical to a plain `JOIN`) only returns rows where **both** conditions in the `ON` clause are true. Of the 8 total referral pairs, only 4 share a batch timing — the other 4 (like Rohit Nair, referred by Aarav Krishnan but enrolled in the EVENING batch while Aarav is MORNING) are correctly excluded.

---

### Question 27 — NATURAL JOIN

**Business Question:** Attach each student's course-wide average fees collected to their own row — let Oracle automatically detect the shared `course` column rather than specifying it manually.

**Expected Output:**

```
16 rows selected.
```

| STUDENT_NAME | COURSE | FEES_PAID | AVG_COURSE_FEES |
|---|---|---|---|
| Anushka Rao | CLOUD COMPUTING | 35000 | 56250.00 |
| Devansh Gupta | CLOUD COMPUTING | 70000 | 56250.00 |
| Ishita Bose | CLOUD COMPUTING | 50000 | 56250.00 |
| Vikas Shetty | CLOUD COMPUTING | 70000 | 56250.00 |
| Aarav Krishnan | DATA SCIENCE | 60000 | 48750.00 |
| Divya Menon | DATA SCIENCE | 45000 | 48750.00 |
| Rohit Nair | DATA SCIENCE | 60000 | 48750.00 |
| Sanya Kapoor | DATA SCIENCE | 30000 | 48750.00 |
| Aditya Ghosh | DIGITAL MARKETING | 17500 | 30625.00 |
| Neha Joshi | DIGITAL MARKETING | 35000 | 30625.00 |
| Nikhil Menon | DIGITAL MARKETING | 35000 | 30625.00 |
| Pooja Varma | DIGITAL MARKETING | 35000 | 30625.00 |
| Farhan Sheikh | WEB DEVELOPMENT | 20000 | 38750.00 |
| Kiran Patel | WEB DEVELOPMENT | 45000 | 38750.00 |
| Meera Iyer | WEB DEVELOPMENT | 45000 | 38750.00 |
| Priyanka Das | WEB DEVELOPMENT | 45000 | 38750.00 |

**Answer:**

```sql
SELECT s.student_name, s.course, s.fees_paid, cs.avg_course_fees
FROM student_enrollment s
NATURAL JOIN (SELECT course, ROUND(AVG(fees_paid), 2) AS avg_course_fees
              FROM student_enrollment
              GROUP BY course) cs
ORDER BY s.course, s.student_name;
```

_Explanation:_ `NATURAL JOIN` automatically joins on **every column that shares the same name** on both sides — here just `course`. It's convenient but risky in real systems: if the derived table accidentally had *another* identically-named column (say, both had a `status` column with different meanings), Oracle would silently join on that too, which is exactly why many teams avoid `NATURAL JOIN` in production and prefer explicit `ON` or `USING`.

---

### Question 28 — LEFT OUTER JOIN

**Business Question:** Give management a **complete** roster of every single student with their referrer's name where one exists — nobody should be missing from the report, even students who joined without any referral.

**Expected Output:**

```
16 rows selected.
```

| STUDENT | REFERRED_BY_NAME |
|---|---|
| Aarav Krishnan | *(NULL)* |
| Divya Menon | Aarav Krishnan |
| Rohit Nair | Aarav Krishnan |
| Sanya Kapoor | *(NULL)* |
| Kiran Patel | *(NULL)* |
| Meera Iyer | Kiran Patel |
| Farhan Sheikh | Kiran Patel |
| Priyanka Das | *(NULL)* |
| Vikas Shetty | *(NULL)* |
| Anushka Rao | Vikas Shetty |
| Devansh Gupta | *(NULL)* |
| Ishita Bose | Vikas Shetty |
| Nikhil Menon | *(NULL)* |
| Pooja Varma | Nikhil Menon |
| Aditya Ghosh | Nikhil Menon |
| Neha Joshi | *(NULL)* |

**Answer:**

```sql
SELECT s1.student_name AS student, s2.student_name AS referred_by_name
FROM student_enrollment s1
LEFT OUTER JOIN student_enrollment s2 ON s1.referred_by = s2.student_id
ORDER BY s1.student_id;
```

_Explanation:_ `LEFT OUTER JOIN` keeps **every row from the left table** (`s1`, all 16 students) regardless of whether a match exists on the right — when there's no matching referrer, `s2`'s columns simply come back `NULL`. Compare this to Question 22's plain `INNER JOIN`, which silently dropped the 8 non-referred students entirely.

---

### Question 29 — RIGHT OUTER JOIN

**Business Question:** Flip the perspective: for every single student, how many people did *they* successfully refer — including the students who referred zero people, since they should still show up with a count of 0.

**Expected Output:**

```
16 rows selected.
```

| POTENTIAL_REFERRER | STUDENTS_REFERRED |
|---|---|
| Aarav Krishnan | 2 |
| Kiran Patel | 2 |
| Nikhil Menon | 2 |
| Vikas Shetty | 2 |
| Aditya Ghosh | 0 |
| Anushka Rao | 0 |
| Devansh Gupta | 0 |
| Divya Menon | 0 |
| Farhan Sheikh | 0 |
| Ishita Bose | 0 |
| Meera Iyer | 0 |
| Neha Joshi | 0 |
| Pooja Varma | 0 |
| Priyanka Das | 0 |
| Rohit Nair | 0 |
| Sanya Kapoor | 0 |

**Answer:**

```sql
SELECT s2.student_name AS potential_referrer,
       COUNT(s1.student_id) AS students_referred
FROM student_enrollment s1
RIGHT OUTER JOIN student_enrollment s2 ON s1.referred_by = s2.student_id
GROUP BY s2.student_name
ORDER BY students_referred DESC, potential_referrer;
```

_Explanation:_ `RIGHT OUTER JOIN` is the mirror image of `LEFT OUTER JOIN` — it keeps **every row from the right table** (`s2`, all 16 students in their "potential referrer" role) even when nothing on the left matches. `COUNT(s1.student_id)` (not `COUNT(*)`) is deliberate: `COUNT` of a specific column ignores `NULL`s, correctly returning `0` rather than `1` for students who referred nobody. Any `LEFT OUTER JOIN` can be rewritten as a `RIGHT OUTER JOIN` by simply swapping which table is listed first — they're the same concept from opposite ends.

---

### Question 30 — FULL OUTER JOIN

**Business Question:** Combine the previous two reports into one unified referral audit: show every gap on **both** sides at once — students with no referrer, *and* students who referred nobody — in a single result set.

**Expected Output:** *(28 total rows, shown here in 3 labeled groups for clarity)*

```
28 rows selected.
```

**Group 1 — matched (8 rows: real referral relationships):**

| STUDENT | REFERRED_BY_NAME |
|---|---|
| Divya Menon | Aarav Krishnan |
| Rohit Nair | Aarav Krishnan |
| Meera Iyer | Kiran Patel |
| Farhan Sheikh | Kiran Patel |
| Anushka Rao | Vikas Shetty |
| Ishita Bose | Vikas Shetty |
| Pooja Varma | Nikhil Menon |
| Aditya Ghosh | Nikhil Menon |

**Group 2 — left side unmatched (8 rows: students with no referrer):**

| STUDENT | REFERRED_BY_NAME |
|---|---|
| Aarav Krishnan | *(NULL)* |
| Sanya Kapoor | *(NULL)* |
| Kiran Patel | *(NULL)* |
| Priyanka Das | *(NULL)* |
| Vikas Shetty | *(NULL)* |
| Devansh Gupta | *(NULL)* |
| Nikhil Menon | *(NULL)* |
| Neha Joshi | *(NULL)* |

**Group 3 — right side unmatched (12 rows: students who never appear as a `student_id` target of anyone's `referred_by`):**

| STUDENT | REFERRED_BY_NAME |
|---|---|
| *(NULL)* | Divya Menon |
| *(NULL)* | Rohit Nair |
| *(NULL)* | Sanya Kapoor |
| *(NULL)* | Meera Iyer |
| *(NULL)* | Farhan Sheikh |
| *(NULL)* | Priyanka Das |
| *(NULL)* | Anushka Rao |
| *(NULL)* | Devansh Gupta |
| *(NULL)* | Ishita Bose |
| *(NULL)* | Pooja Varma |
| *(NULL)* | Aditya Ghosh |
| *(NULL)* | Neha Joshi |

**Answer:**

```sql
SELECT s1.student_name AS student, s2.student_name AS referred_by_name
FROM student_enrollment s1
FULL OUTER JOIN student_enrollment s2 ON s1.referred_by = s2.student_id
ORDER BY student NULLS LAST, referred_by_name NULLS LAST;
```

_Explanation:_ `FULL OUTER JOIN` is simply `LEFT OUTER JOIN` **plus** everything a `RIGHT OUTER JOIN` would add that isn't already there — every matched pair, every unmatched left row, and every unmatched right row, all in one result. Group 3 looks large (12 rows) because 12 of the 16 students never referred anyone at all — the same 12 that showed `0` in Question 29's `RIGHT OUTER JOIN`.

---

### Question 31 — SELF JOIN Capstone: Study Buddy Matching

**Business Question:** Set up study groups: find every pair of **different** students who share both the same course and the same batch timing — each pair should appear exactly once (not twice, and never paired with themselves).

**Expected Output:**

```
8 rows selected.
```

| COURSE | BATCH | STUDENT_A | STUDENT_B |
|---|---|---|---|
| CLOUD COMPUTING | EVENING | Vikas Shetty | Anushka Rao |
| CLOUD COMPUTING | MORNING | Devansh Gupta | Ishita Bose |
| DATA SCIENCE | EVENING | Rohit Nair | Sanya Kapoor |
| DATA SCIENCE | MORNING | Aarav Krishnan | Divya Menon |
| DIGITAL MARKETING | EVENING | Aditya Ghosh | Neha Joshi |
| DIGITAL MARKETING | WEEKEND | Nikhil Menon | Pooja Varma |
| WEB DEVELOPMENT | MORNING | Kiran Patel | Meera Iyer |
| WEB DEVELOPMENT | WEEKEND | Farhan Sheikh | Priyanka Das |

**Answer:**

```sql
SELECT s1.course, s1.batch,
       s1.student_name AS student_a, s2.student_name AS student_b
FROM student_enrollment s1
JOIN student_enrollment s2
  ON s1.course = s2.course
 AND s1.batch = s2.batch
 AND s1.student_id < s2.student_id
ORDER BY s1.course, s1.batch;
```

_Explanation:_ This is the textbook **self-join pattern for unique pairs**: `s1.student_id < s2.student_id` is the key trick — without it, you'd get every pair **twice** (A-B and B-A) plus every student paired with themselves. The `<` operator guarantees each real-world pair shows up exactly once, with the lower ID always listed first.

---

## PART 4 — Capstone: Everything Mixed Together

### Question 32 — Referral Effectiveness Leaderboard

**Business Question:** Marketing wants to identify the most effective referrers — anyone who referred 2 or more students — along with the total fees those referred students have paid so far, to calculate referral-bonus payouts.

**Expected Output:**

```
4 rows selected.
```

| REFERRER | STUDENTS_REFERRED | TOTAL_FEES_FROM_REFERRALS |
|---|---|---|
| Aarav Krishnan | 2 | 105000 |
| Vikas Shetty | 2 | 85000 |
| Kiran Patel | 2 | 65000 |
| Nikhil Menon | 2 | 52500 |

**Answer:**

```sql
SELECT s2.student_name AS referrer,
       COUNT(s1.student_id) AS students_referred,
       SUM(s1.fees_paid) AS total_fees_from_referrals
FROM student_enrollment s1
JOIN student_enrollment s2 ON s1.referred_by = s2.student_id
GROUP BY s2.student_name
HAVING COUNT(s1.student_id) >= 2
ORDER BY total_fees_from_referrals DESC;
```

_Explanation:_ Self-**JOIN** (Part 3) + **aggregate functions** `COUNT`/`SUM` (Part 1) + **GROUP BY**/**HAVING** (Part 2), all in one query — this is precisely the kind of question a real analytics request looks like.

---

### Question 33 — Course Leaderboard with Referral Source

**Business Question:** Build the official merit leaderboard: rank every graded student within their course, and show whether they joined directly or via referral — with the referrer's name properly capitalized.

**Expected Output:**

```
12 rows selected.
```

| COURSE | STUDENT_NAME | EXAM_SCORE | COURSE_RANK | JOINED_VIA |
|---|---|---|---|---|
| CLOUD COMPUTING | Vikas Shetty | 90 | 1 | Direct Enrollment |
| CLOUD COMPUTING | Devansh Gupta | 85 | 2 | Direct Enrollment |
| CLOUD COMPUTING | Anushka Rao | 58 | 3 | Vikas Shetty |
| DATA SCIENCE | Rohit Nair | 92 | 1 | Aarav Krishnan |
| DATA SCIENCE | Aarav Krishnan | 88 | 2 | Direct Enrollment |
| DATA SCIENCE | Divya Menon | 76 | 3 | Aarav Krishnan |
| DIGITAL MARKETING | Nikhil Menon | 79 | 1 | Direct Enrollment |
| DIGITAL MARKETING | Pooja Varma | 79 | 1 | Nikhil Menon |
| DIGITAL MARKETING | Neha Joshi | 65 | 3 | Direct Enrollment |
| WEB DEVELOPMENT | Kiran Patel | 81 | 1 | Direct Enrollment |
| WEB DEVELOPMENT | Priyanka Das | 74 | 2 | Direct Enrollment |
| WEB DEVELOPMENT | Meera Iyer | 69 | 3 | Kiran Patel |

**Answer:**

```sql
SELECT s1.course, s1.student_name, s1.exam_score,
       RANK() OVER (PARTITION BY s1.course ORDER BY s1.exam_score DESC) AS course_rank,
       COALESCE(INITCAP(s2.student_name), 'Direct Enrollment') AS joined_via
FROM student_enrollment s1
LEFT JOIN student_enrollment s2 ON s1.referred_by = s2.student_id
WHERE s1.exam_score IS NOT NULL
ORDER BY s1.course, course_rank;
```

_Explanation:_ **Window function** `RANK()` + **LEFT JOIN** (self-join) + **misc function** `COALESCE` + **string function** `INITCAP` + a `WHERE` filter — five separate concepts from three different parts of this guide, combined into one report exactly the way a real dashboard query would look.

---

### Question 34 — Referral Value Multiplier

**Business Question:** Calculate a "loyalty multiplier" for each referrer: what percentage of their *own* fees paid is matched by the total fees their referred students have collectively contributed? A multiplier over 100% means their referrals brought in more revenue than they personally paid.

**Expected Output:**

```
4 rows selected.
```

| REFERRER | OWN_FEES_PAID | TOTAL_FEES_FROM_REFERRALS | REFERRAL_VALUE_MULTIPLIER_PCT |
|---|---|---|---|
| Aarav Krishnan | 60000 | 105000 | 175.00 |
| Nikhil Menon | 35000 | 52500 | 150.00 |
| Kiran Patel | 45000 | 65000 | 144.44 |
| Vikas Shetty | 70000 | 85000 | 121.43 |

**Answer:**

```sql
SELECT s2.student_name AS referrer,
       s2.fees_paid AS own_fees_paid,
       SUM(s1.fees_paid) AS total_fees_from_referrals,
       ROUND(SUM(s1.fees_paid) / s2.fees_paid * 100, 2) AS referral_value_multiplier_pct
FROM student_enrollment s1
JOIN student_enrollment s2 ON s1.referred_by = s2.student_id
GROUP BY s2.student_name, s2.fees_paid
ORDER BY referral_value_multiplier_pct DESC;
```

_Explanation:_ Self-join, `GROUP BY` on two columns, an aggregate `SUM`, and a nested mathematical expression (`ROUND` wrapping a division and multiplication) — every referrer here brought in **more** revenue through referrals than they personally paid, a genuinely persuasive number for justifying a referral-bonus budget.

---

### Question 35 — Grand Finale: The Term-End Report Card

**Business Question:** Build the single consolidated term-end report for every currently enrolled student (exclude dropouts): their score (or "Pending"), their rank within their course, how far above or below their course average they scored, and their referral source — sorted by course, then rank.

**Expected Output:**

```
15 rows selected.
```

| COURSE | STUDENT_NAME | SCORE_DISPLAY | COURSE_RANK | GAP_VS_COURSE_AVG | JOINED_VIA |
|---|---|---|---|---|---|
| CLOUD COMPUTING | Vikas Shetty | 90 | 1 | 12.33 | Direct Enrollment |
| CLOUD COMPUTING | Devansh Gupta | 85 | 2 | 7.33 | Direct Enrollment |
| CLOUD COMPUTING | Ishita Bose | Pending | 3 | N/A | Vikas Shetty |
| DATA SCIENCE | Rohit Nair | 92 | 1 | 6.67 | Aarav Krishnan |
| DATA SCIENCE | Aarav Krishnan | 88 | 2 | 2.67 | Direct Enrollment |
| DATA SCIENCE | Divya Menon | 76 | 3 | -9.33 | Aarav Krishnan |
| DATA SCIENCE | Sanya Kapoor | Pending | 4 | N/A | Direct Enrollment |
| DIGITAL MARKETING | Nikhil Menon | 79 | 1 | 4.67 | Direct Enrollment |
| DIGITAL MARKETING | Pooja Varma | 79 | 1 | 4.67 | Nikhil Menon |
| DIGITAL MARKETING | Neha Joshi | 65 | 3 | -9.33 | Direct Enrollment |
| DIGITAL MARKETING | Aditya Ghosh | Pending | 4 | N/A | Nikhil Menon |
| WEB DEVELOPMENT | Kiran Patel | 81 | 1 | 6.33 | Direct Enrollment |
| WEB DEVELOPMENT | Priyanka Das | 74 | 2 | -0.67 | Direct Enrollment |
| WEB DEVELOPMENT | Meera Iyer | 69 | 3 | -5.67 | Kiran Patel |
| WEB DEVELOPMENT | Farhan Sheikh | Pending | 4 | N/A | Kiran Patel |

*(Note: Anushka Rao, `CLOUD COMPUTING`, is correctly excluded — she `DROPPED`.)*

**Answer:**

```sql
WITH course_avg AS (
    SELECT course, ROUND(AVG(exam_score), 2) AS avg_score
    FROM student_enrollment
    GROUP BY course
)
SELECT s1.course,
       s1.student_name,
       COALESCE(TO_CHAR(s1.exam_score), 'Pending') AS score_display,
       RANK() OVER (PARTITION BY s1.course ORDER BY s1.exam_score DESC NULLS LAST) AS course_rank,
       CASE WHEN s1.exam_score IS NULL THEN 'N/A'
            ELSE TO_CHAR(ROUND(s1.exam_score - ca.avg_score, 2))
       END AS gap_vs_course_avg,
       COALESCE(s2.student_name, 'Direct Enrollment') AS joined_via
FROM student_enrollment s1
JOIN course_avg ca ON s1.course = ca.course
LEFT JOIN student_enrollment s2 ON s1.referred_by = s2.student_id
WHERE s1.status <> 'DROPPED'
ORDER BY s1.course, course_rank;
```

_Explanation:_ The complete stack, in one query: a `WITH` clause (subquery-based **equi-join** to `course_avg`), a **window function** (`RANK`), a **self-join** (`LEFT JOIN` for referral source), **miscellaneous functions** (`COALESCE`), a **CASE expression**, **nested math functions** (`ROUND` of a subtraction), and a plain `WHERE` filter — every single topic from this entire guide, working together on the one table you built in Question 1.

---

## Final Answer Key & Concept Map

| # | Topic | Concept(s) Practiced |
|---|---|---|
| 1–2 | Setup | Table creation with a self-referencing FK; bulk `INSERT ALL` |
| 3 | Functions | Deterministic vs non-deterministic (`SYSDATE` vs `TRUNC`) |
| 4–5 | Functions | Scalar functions (`INITCAP`, `UPPER`, `ROUND`) |
| 6 | Functions | Aggregate functions (`COUNT`, `SUM`, `AVG`, `MAX`, `MIN`) |
| 7–8 | Functions | String functions (`SUBSTR`, `INSTR`, concatenation) |
| 9 | Functions | Mathematical functions (`CEIL`, `MOD`) |
| 10–11 | Functions | Miscellaneous functions (`COALESCE`, `NULLIF`) |
| 12–13 | Functions | Analytical/window functions (`RANK`, running `SUM() OVER`) |
| 14 | Functions | Nesting functions & expressions |
| 15 | Clauses | `GROUP BY` |
| 16 | Clauses | `HAVING` |
| 17 | Clauses | `ORDER BY` (multi-column) |
| 18 | Clauses | Order of execution of `SELECT` clauses |
| 19–20 | Clauses | `GROUP BY` (multi-column), `WHERE`+`GROUP BY`+`HAVING` combined |
| 21 | Joins | Theta style (old syntax, non-equality) |
| 22 | Joins | Oracle/ANSI style — `JOIN ... ON` |
| 23 | Joins | Oracle/ANSI style — `JOIN ... USING` |
| 24 | Joins | EQUI-JOIN |
| 25 | Joins | `CROSS JOIN` |
| 26 | Joins | `INNER JOIN` (explicit, multi-condition) |
| 27 | Joins | `NATURAL JOIN` |
| 28 | Joins | `LEFT OUTER JOIN` |
| 29 | Joins | `RIGHT OUTER JOIN` |
| 30 | Joins | `FULL OUTER JOIN` |
| 31 | Joins | `SELF JOIN` (unique-pair pattern) |
| 32–35 | Capstone | Every topic above, combined into real multi-concept business queries |

### Where to go next

Natural extensions for a future round: subqueries and correlated subqueries (a lot of what's above already hints at these), `WITH` clause / CTEs in more depth, `PIVOT`/`UNPIVOT` for turning the course-wise reports into a cross-tab, and moving from ad-hoc queries into PL/SQL stored procedures for the recurring reports (Questions 15, 32, and 35 are all excellent procedure candidates).