# SQL Problem-Solving Mastery: Think Like a Query Architect

You already know the syntax. This document trains the part that's harder to teach: **how to read a question and decide what to do before you touch the keyboard.**

## How to use this

For every question, don't jump to the SQL. Cover it with your hand if you have to. Work through:
1. What is actually being asked (in plain English)?
2. What tables do I need?
3. What operations does this require — and in what order?
4. If there's a subquery, what's the "thing I need to know first"?

Then look at the incremental build and compare it to your own thinking, not just the final answer. The gap between your reasoning and the shown reasoning is where the learning happens.

24 fully-worked questions across 5 levels, 8 different business scenarios (so you're not memorizing one schema), a bonus practice set with no solutions, and a syllabus-mapped appendix at the very end covering the setup/administrative SQL (DDL, DCL, TCL, constraints, remaining join syntax, `FETCH FIRST`, `NULLIF`) that doesn't fit the "logical puzzle" format but is still worth knowing.

This version specifically adds coverage for: **set operators (UNION/INTERSECT/MINUS)**, **ANY/ALL/SOME**, **multi-column subqueries**, **FULL OUTER JOIN**, and a fully worked **DELETE with subquery** — the gaps against a standard Oracle SQL syllabus, excluding views, indexes, and equi-joins as requested.

---

# LEVEL 1 — Reading the Question Correctly

The goal here isn't difficulty. It's building the habit of translating English into a sequence of operations before writing anything.

## Question 1 — ShopEase (E-commerce)

**Tables:**
- `customers(customer_id, name, city, signup_date)`
- `orders(order_id, customer_id, order_date, status)`

**Question:** *"List every customer from Chennai who placed at least one order in 2025, along with how many orders each of them placed. Show the most active customers first."*

### A. Understand the business question
Break the sentence into pieces:
```
"customer from Chennai"        → filter on customers.city
"placed at least one order"    → needs orders table, and "at least one" means the customer must appear there
"in 2025"                      → filter on orders.order_date
"how many orders"              → count per customer
"most active first"            → sort by that count, descending
```
This is a **counting** question wearing a "list customers" disguise — the real output is customer + a computed number.

### B. Tables needed
- `customers` — for name and city.
- `orders` — the only place order counts and dates exist.
Both are required because the filter condition ("Chennai") lives on one table and the thing you're counting lives on the other.

### C–D. Logical operations and order of thought
- Do I need a JOIN? Yes — the two facts (city, order count) live in two tables.
- Do I need filtering? Yes — city and year.
- Do I need aggregation? Yes — COUNT of orders.
- Do I need GROUP BY? Yes — one count per customer, not one global count.
- Do I need HAVING? No — there's no filter *on the aggregated result itself* (we're not saying "more than 5 orders"). "At least one order" is naturally satisfied by an inner join — a customer with zero matching orders simply won't produce a row.
- Do I need a subquery? No — nothing needs to be known "first."

```
Step 1 → Join customers to orders on customer_id
Step 2 → Filter to city = 'Chennai' and order year = 2025
Step 3 → Count orders per customer
Step 4 → Sort by that count, descending
```

### G. Build incrementally
```sql
SELECT c.customer_id, c.name
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
```
Add the filter:
```sql
WHERE c.city = 'Chennai'
  AND o.order_date >= '2025-01-01' AND o.order_date < '2026-01-01'
```
Add the aggregation:
```sql
SELECT c.customer_id, c.name, COUNT(o.order_id) AS order_count
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE c.city = 'Chennai'
  AND o.order_date >= '2025-01-01' AND o.order_date < '2026-01-01'
GROUP BY c.customer_id, c.name
ORDER BY order_count DESC;
```

### H. Line-by-line tie-back
- `JOIN` → connects "who they are" to "what they ordered."
- `WHERE` → enforces "from Chennai" and "in 2025" — both are row-level facts, known before any grouping happens.
- `COUNT(o.order_id)` + `GROUP BY` → answers "how many orders each."
- `ORDER BY ... DESC` → "most active first."

### 🔁 Worth noticing
`WHERE` was correct here, not `HAVING` — because both conditions filter facts that exist on individual rows *before* grouping. A common instinct is to reach for `HAVING` any time counting is involved; `HAVING` is only needed when the filter is *on the aggregated value itself* (see Question 4).

---

## Question 2 — SkyHigh Airlines

**Tables:**
- `passengers(passenger_id, name, country)`
- `flights(flight_id, origin, destination, flight_date, capacity)`
- `bookings(booking_id, passenger_id, flight_id, booking_date, seat_class, fare)`

**Question:** *"Give me an alphabetical list of every unique passenger who has ever booked a flight to Dubai — no duplicates, even if they've flown there five times."*

### A. Understand the question
```
"booked a flight to Dubai"   → bookings joined to flights, filtered on destination
"unique... no duplicates"    → the same passenger flying 5 times should appear once
"alphabetical"               → sort by name
```

### B. Tables needed
`passengers` for names, `flights` for destination, `bookings` to connect the two (a passenger isn't directly linked to a flight — the booking is the bridge).

### C–D. Operations
- JOIN: yes, across three tables (or two, if you only need the name and don't need passenger_id elsewhere).
- Filtering: yes, `destination = 'Dubai'`.
- Aggregation: **no** — you're not counting or summing anything, just deduplicating.
- DISTINCT vs GROUP BY: this is the key decision point. Since there's no aggregate function needed, `DISTINCT` is the right, simpler tool. `GROUP BY` would work identically here but signals "I'm computing something per group," which isn't true — that's misleading to a reader of your query.

```
Step 1 → Join bookings to flights on flight_id
Step 2 → Join to passengers on passenger_id
Step 3 → Filter to destination = 'Dubai'
Step 4 → Remove duplicate passengers
Step 5 → Sort alphabetically
```

### G. Build
```sql
SELECT DISTINCT p.name
FROM passengers p
JOIN bookings b ON b.passenger_id = p.passenger_id
JOIN flights f ON f.flight_id = b.flight_id
WHERE f.destination = 'Dubai'
ORDER BY p.name;
```

### 🔁 Worth noticing
**DISTINCT vs GROUP BY**: use `GROUP BY` when you need to *compute something per group* (a count, sum, average). Use `DISTINCT` when you just need to *collapse duplicate rows* with no computation. Using `GROUP BY` without any aggregate function is a signal you actually wanted `DISTINCT`.

---

## Question 3 — CityCare Hospital

**Tables:**
- `doctors(doctor_id, name, department_id, specialization)`
- `departments(department_id, department_name)`
- `appointments(appointment_id, patient_id, doctor_id, appointment_date, status, fee)`

**Question:** *"For every doctor, show their name, their department's name, and the total number of appointments they've handled — including doctors with zero appointments. Order by busiest doctor first."*

### A. Understand the question
```
"name, department name, total appointments"   → three columns from three tables
"including doctors with zero appointments"    → an inner join would silently drop them — this phrase is the signal
"busiest first"                                → ORDER BY count DESC
```
The phrase *"including doctors with zero"* is doing a lot of work — it's telling you which JOIN type to use.

### B. Tables needed
All three: `doctors` (name), `departments` (department name), `appointments` (what you're counting).

### C–D. Operations
- JOIN type matters here: `doctors` → `departments` can be an inner join (every doctor has a department). `doctors` → `appointments` **must** be a LEFT JOIN, because an inner join would exclude doctors with no appointments — and the question explicitly asked to keep them.
- Aggregation: yes, COUNT.
- GROUP BY: yes, per doctor.
- A NULL-handling detail: `COUNT(appointment_id)` (not `COUNT(*)`) is required — with a LEFT JOIN, a doctor with zero appointments produces one row with all `appointments` columns as NULL. `COUNT(*)` would count that row as 1; `COUNT(appointment_id)` correctly counts 0, because COUNT of a column ignores NULLs.

```
Step 1 → Join doctors to departments (inner join, always exists)
Step 2 → Left join doctors to appointments (preserve doctors with none)
Step 3 → Count appointments per doctor, using COUNT(column) not COUNT(*)
Step 4 → Sort descending
```

### G. Build
```sql
SELECT d.name AS doctor_name, dep.department_name,
       COUNT(a.appointment_id) AS total_appointments
FROM doctors d
JOIN departments dep ON dep.department_id = d.department_id
LEFT JOIN appointments a ON a.doctor_id = d.doctor_id
GROUP BY d.doctor_id, d.name, dep.department_name
ORDER BY total_appointments DESC;
```

### H. Tie-back
The `LEFT JOIN` is the whole trick of this question — everything else is mechanical once you catch it. This is worth internalizing: **whenever a question says "including those with none/zero/never," that's a LEFT JOIN (or equivalent NOT EXISTS) signal, not an INNER JOIN.**

---

# LEVEL 2 — Combining Concepts

Now questions require you to hold two or three decisions in your head at once: what to filter, what to compute, what to filter *about* the computation, and how to label results.

## Question 4 — QuickBite (Food Delivery)

**Tables:**
- `restaurants(restaurant_id, name, city, cuisine_type)`
- `customers(customer_id, name, city)`
- `delivery_orders(order_id, customer_id, restaurant_id, rider_id, order_date, amount, status)`

**Question:** *"Which cities have an average delivery order value above ₹450? Only consider cities that have had more than 50 orders total — smaller cities skew the average and aren't reliable yet."*

### A. Understand the question
```
"average delivery order value above ₹450"   → AVG(amount), filtered — but on the AVERAGE, not on individual orders
"per city"                                   → GROUP BY city
"more than 50 orders total"                  → COUNT(*), also filtered on the aggregated result
"smaller cities skew..."                     → this is just justification text, not a new operation
```
Two separate conditions, and **both apply to computed, grouped values** — not to individual rows. That's the tell for `HAVING`.

### B. Tables needed
`delivery_orders` has everything you need (city could live on `customers` or `restaurants` — assume delivery city = customer's city, so join to `customers`).

### C–D. Operations
- JOIN: yes, orders to customers, to get city.
- WHERE: no row-level filter needed here — there's nothing to exclude before grouping.
- GROUP BY: yes, by city.
- HAVING: yes — **two conditions**, both on aggregates: `AVG(amount) > 450` AND `COUNT(*) > 50`.
- This is the core WHERE-vs-HAVING lesson: if a condition mentions "average," "total," "count," or any aggregate word, it almost always belongs in HAVING, because that value doesn't exist until after grouping.

```
Step 1 → Join orders to customers for city
Step 2 → Group by city
Step 3 → Compute AVG(amount) and COUNT(*) per city
Step 4 → Keep only groups where AVG > 450 AND COUNT > 50
```

### G. Build
```sql
SELECT c.city, AVG(o.amount) AS avg_order_value, COUNT(*) AS total_orders
FROM delivery_orders o
JOIN customers c ON c.customer_id = o.customer_id
GROUP BY c.city
HAVING AVG(o.amount) > 450 AND COUNT(*) > 50
ORDER BY avg_order_value DESC;
```

### 🔁 Worth noticing
Both conditions sit in the *same* `HAVING` clause, joined with `AND`. A common mistake is trying to filter the count with `WHERE` because it "feels like filtering" — but `WHERE` runs before grouping exists, so it has no access to `COUNT(*)` yet.

---

## Question 5 — Riverdale University

**Tables:**
- `students(student_id, name, major, enrollment_year)`
- `professors(professor_id, name, department)`
- `courses(course_id, course_name, department, credits)`
- `enrollments(enrollment_id, student_id, course_id, semester, grade)`

**Question:** *"For each professor, show how many distinct students they've taught across all their courses, and label them: 'High Demand' for 100+ students, 'Moderate' for 30–99, and 'Niche' for under 30."*

### A. Understand the question
```
"distinct students... across all their courses"   → a professor may teach multiple courses; don't double-count a student who took two of them
"per professor"                                    → GROUP BY professor
"label based on ranges"                             → CASE, applied to the aggregated count
```

### B. Tables needed
`professors` (name), `courses` (links professor to department — assume `courses.department` matches `professors.department`, or more realistically a `professor_id` on courses; we'll assume `courses` has a `professor_id` column for this to work cleanly), `enrollments` (the actual student-course link).

*(Correction to schema for this question: assume `courses(course_id, course_name, department, credits, professor_id)`.)*

### C–D. Operations
- JOIN: professors → courses → enrollments.
- DISTINCT inside an aggregate: `COUNT(DISTINCT student_id)` — this is different from plain `COUNT`, because a student enrolled in two of the same professor's courses should count once.
- GROUP BY: per professor.
- CASE: applied to the *result* of the aggregation, not to a raw column — so it goes in the SELECT list, referencing the same aggregate expression (or a subquery/CTE, if your SQL dialect won't let you reuse an alias directly in CASE — safest is to repeat the expression).

```
Step 1 → Join professors → courses → enrollments
Step 2 → Group by professor
Step 3 → Count DISTINCT students per professor
Step 4 → Apply CASE to bucket the count into a label
```

### G. Build
```sql
SELECT p.name,
       COUNT(DISTINCT e.student_id) AS student_count,
       CASE
           WHEN COUNT(DISTINCT e.student_id) >= 100 THEN 'High Demand'
           WHEN COUNT(DISTINCT e.student_id) >= 30  THEN 'Moderate'
           ELSE 'Niche'
       END AS demand_label
FROM professors p
JOIN courses c ON c.professor_id = p.professor_id
JOIN enrollments e ON e.course_id = c.course_id
GROUP BY p.professor_id, p.name;
```

### 🔁 Worth noticing
**COUNT(DISTINCT x) vs COUNT(x)**: use `DISTINCT` inside the aggregate whenever the same entity could legitimately appear more than once in the joined rows for reasons unrelated to what you're counting — here, one student across two courses of the same professor.

---

## Question 6 — TrustBank

**Tables:**
- `accounts(account_id, customer_id, account_type, balance)`
- `transactions(transaction_id, account_id, transaction_date, amount, transaction_type)` — `transaction_type` is `'deposit'` or `'withdrawal'`

**Question:** *"For each account type, show total amount deposited and total amount withdrawn, side by side in the same row."*

### A. Understand the question
This is the trap that makes people reach for two separate queries or a self-join. Re-read carefully: *"side by side in the same row"* per account type — one row per type, two numeric columns. That phrase is the signal for **conditional aggregation**, not filtering.

```
"total deposited" and "total withdrawn"   → two separate sums
"same row, per account type"              → both computed in one GROUP BY pass, not two filtered queries
```

### B. Tables needed
`transactions` (the amounts and types), `accounts` (the account_type to group by).

### C–D. Operations
- JOIN: transactions to accounts.
- GROUP BY: account_type.
- Conditional aggregation: `SUM(CASE WHEN transaction_type = 'deposit' THEN amount ELSE 0 END)`, and the mirror for withdrawal. This is different from filtering with WHERE, because WHERE would only let you keep *one* type per query — you need both types summed in parallel, in the same group.

```
Step 1 → Join transactions to accounts
Step 2 → Group by account_type
Step 3 → Sum amount conditionally, once per transaction_type, in the same SELECT
```

### G. Build
```sql
SELECT a.account_type,
       SUM(CASE WHEN t.transaction_type = 'deposit' THEN t.amount ELSE 0 END) AS total_deposited,
       SUM(CASE WHEN t.transaction_type = 'withdrawal' THEN t.amount ELSE 0 END) AS total_withdrawn
FROM transactions t
JOIN accounts a ON a.account_id = t.account_id
GROUP BY a.account_type;
```

### 🔁 Worth noticing
**CASE vs filtering (WHERE)**: if you need *two (or more) numbers side by side that each depend on a different condition*, that's conditional aggregation with CASE inside an aggregate function. If you only need *one* filtered subset, that's a WHERE clause. Mixing these up is one of the most common real-world SQL mistakes.

---

## Question 7 — NexaTech (Employees & Projects)

**Tables:**
- `employees(employee_id, name, department_id, salary, manager_id, hire_date)`
- `departments(department_id, department_name)`

**Question:** *"List every employee along with their manager's name, and flag with a Yes/No column whether the employee earns more than their manager."*

### A. Understand the question
```
"employee along with their manager's name"   → manager is just another row in the SAME table
"earns more than their manager"              → compare two salary values from the same table, different rows
```
The tell here: manager and employee are the *same kind of entity*, stored in the *same table*, just referenced by `manager_id`. That's a **self-join**.

### B. Tables needed
Just `employees` — twice, under two aliases, playing two different roles ("the employee" and "the employee's manager").

### C–D. Operations
- Self-join: `employees e` joined to `employees m` where `e.manager_id = m.employee_id`.
- Should this be an inner or left join? Depends on whether every employee has a manager. If the CEO has `manager_id = NULL`, an inner join would silently drop that row. Since the question says "list *every* employee," use a LEFT JOIN.
- CASE: to turn the comparison into Yes/No.
- NULL handling: if `m.salary` is NULL (no manager), the comparison `e.salary > m.salary` evaluates to NULL, not TRUE or FALSE — so the CASE needs an explicit branch for that, or it will silently fall to ELSE.

```
Step 1 → Self-join employees to employees on manager_id = employee_id (LEFT, to keep employees without a manager)
Step 2 → Compare salaries with CASE
Step 3 → Handle the NULL-manager case explicitly
```

### G. Build
```sql
SELECT e.name AS employee_name, m.name AS manager_name,
       CASE
           WHEN m.employee_id IS NULL THEN 'No Manager'
           WHEN e.salary > m.salary THEN 'Yes'
           ELSE 'No'
       END AS earns_more_than_manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```

### 🔁 Worth noticing
A **self-join** isn't a special SQL feature — it's the same JOIN syntax, applied to one table twice with two aliases. The recognition trigger is always: *"does this question compare two rows of the same kind of entity to each other?"* (employee vs. manager, product vs. product, city vs. city in the same table.)

---

## Question 8 — NexaTech (Employees & Departments)

**Tables:** same as above, plus `departments(department_id, department_name)`.

**Question:** *"Which departments have more than 3 employees earning above ₹80,000?"*

### A. Understand the question
```
"employees earning above ₹80,000"     → a row-level filter on employees
"more than 3... [of them]"            → a filter on the COUNT of the already-filtered rows
```
This is subtly different from Question 4. Here, the filter (`salary > 80000`) applies to *individual employees*, not to an aggregate — so it belongs in `WHERE`, applied *before* grouping. Only the count-of-3 belongs in `HAVING`.

### B. Tables needed
`employees` (salary, department), `departments` (name).

### C–D. Operations
- WHERE: `salary > 80000` — a row-level condition, known before any grouping.
- GROUP BY: department.
- HAVING: `COUNT(*) > 3` — a condition on the grouped result.

```
Step 1 → Filter employees to salary > 80000 (WHERE — row-level, pre-grouping)
Step 2 → Join to departments for the name
Step 3 → Group by department
Step 4 → Keep only groups with COUNT(*) > 3 (HAVING — post-grouping)
```

### G. Build
```sql
SELECT dep.department_name, COUNT(*) AS high_earners
FROM employees e
JOIN departments dep ON dep.department_id = e.department_id
WHERE e.salary > 80000
GROUP BY dep.department_name
HAVING COUNT(*) > 3;
```

### 🔁 Worth noticing
Compare this directly to Question 4. There, both conditions were about the *aggregate itself* (average, total count) → both in HAVING. Here, one condition is about *individual rows before grouping* (a specific employee's salary) and the other is about the *count of the group* → split between WHERE and HAVING. **The test is always: "does this condition exist before I group, or only after?"**

---

## Question 8B — QuickBite (Set Operators)

**Tables:**
- `restaurants(restaurant_id, name, city, cuisine_type)`
- `customers(customer_id, name, city)`

**Question:** *"Give me one combined, deduplicated list of every city QuickBite has any presence in — through a restaurant or a customer. Separately: which cities have a restaurant but currently zero customers (expansion with no demand yet)? And which have customers but zero restaurants (demand with no supply)?"*

### A. Understand the question
```
"combined, deduplicated... either"      → UNION of two city lists
"restaurant but zero customers"         → cities in restaurants, absent from customers
"customers but zero restaurants"        → the reverse — cities in customers, absent from restaurants
```
Notice there's no shared key being matched row-to-row here — you're not asking "which restaurant belongs to which customer." You're comparing **two independent lists of the same shape** (a single city column) as sets. That distinction is the whole lesson: **JOIN widens rows by matching a key across tables; set operators (UNION/INTERSECT/MINUS) combine or compare entire result sets that already have the same columns.**

### B. Tables needed
Both — but not joined. Each is queried independently into a same-shaped result (one `city` column), and the two results are combined afterward.

### C–D. Operations and order
```
Step 1 → SELECT DISTINCT city FROM restaurants
Step 2 → SELECT DISTINCT city FROM customers
Step 3 → UNION the two for "any presence"
Step 4 → restaurants' cities MINUS customers' cities → supply, no demand
Step 5 → customers' cities MINUS restaurants' cities → demand, no supply
```

### G. Build
```sql
-- Any presence at all (deduplicated)
SELECT city FROM restaurants
UNION
SELECT city FROM customers;

-- Supply without demand
SELECT city FROM restaurants
MINUS
SELECT city FROM customers;
-- (Oracle: MINUS. SQL Server / Postgres: EXCEPT — same behavior, different keyword.)

-- Demand without supply
SELECT city FROM customers
MINUS
SELECT city FROM restaurants;
```

### 🔁 Worth noticing
- `MINUS`/`EXCEPT` is **not symmetric** — `A MINUS B` and `B MINUS A` are different questions, unlike `UNION` and `INTERSECT`, which give the same result regardless of order.
- `UNION` automatically removes duplicate rows across both sides (it behaves like `DISTINCT` applied to the combined set). `UNION ALL` keeps every row from both sides, duplicates included — use it when row counts genuinely matter (e.g., combining transaction logs from two systems where a repeat is a real repeat, not noise).
- Every `SELECT` in a set operation must return the **same number of columns, in compatible data types** — this is a strict structural requirement, unlike JOIN, which has no such constraint.

---

# LEVEL 3 — Subquery Thinking

The core skill from here on: recognizing when the question secretly contains *two* questions — an inner one you must answer first, and an outer one that depends on it.

## Question 9 — StreamPlex

**Tables:**
- `users(user_id, name, country, plan_type, signup_date)`
- `content(content_id, title, genre, release_year, type)`
- `watch_history(watch_id, user_id, content_id, watch_date, minutes_watched)`

**Question:** *"Which users have watched more distinct titles than the platform average?"*

### A. Understand the question
```
"more distinct titles than the platform average"

Users
  ↓
Count distinct titles watched, per user
  ↓
Also need: the average of that count, across ALL users
  ↓
Compare each user's count to that one average number
  ↓
Return users above it
```

### E. The inner problem
> "What do I need to know FIRST, before I can answer the main question?"
→ **The platform-wide average number of distinct titles watched per user.** You cannot know if a user is "above average" without first computing what average means here.

This average is itself a two-step calculation: count distinct titles *per user*, then average *those counts*. You can't do this in one flat aggregate — averaging `watch_history` rows directly would count re-watches and overweight binge-watchers. This has to be a grouped subquery.

### F. What does the subquery return?
One number — the platform-wide average. **Scalar subquery.** That means it can be compared directly with `>`, no `IN`/`ANY`/`ALL` needed.

### C–D. Operations and order
```
Step 1 → (Inner) Group watch_history by user, count distinct content_id per user
Step 2 → (Inner) Average those per-user counts → one number
Step 3 → (Outer) Group watch_history by user again, count distinct content_id per user
Step 4 → (Outer) Compare each user's count to the inner average (HAVING)
```

### G. Build
Inner query alone, to prove it works:
```sql
SELECT AVG(title_count) FROM (
    SELECT user_id, COUNT(DISTINCT content_id) AS title_count
    FROM watch_history
    GROUP BY user_id
) per_user_counts;
```
Now the outer query, using it as a scalar subquery in HAVING:
```sql
SELECT u.name, COUNT(DISTINCT w.content_id) AS titles_watched
FROM users u
JOIN watch_history w ON w.user_id = u.user_id
GROUP BY u.user_id, u.name
HAVING COUNT(DISTINCT w.content_id) > (
    SELECT AVG(title_count) FROM (
        SELECT user_id, COUNT(DISTINCT content_id) AS title_count
        FROM watch_history
        GROUP BY user_id
    ) per_user_counts
);
```

### 🔁 Worth noticing
This is a **subquery in FROM** (a derived table) nested inside a **scalar subquery used in HAVING**. Notice the average couldn't be computed as `AVG(COUNT(DISTINCT content_id))` directly — SQL doesn't allow nesting an aggregate inside another aggregate in the same query level. That restriction is *why* the derived table exists: it's how you force the per-user count to be computed and finalized before averaging it.

---

## Question 10 — ShopEase (revisited)

**Tables:** same as Question 1.

**Question:** *"Which customers have never placed a single order?"*

### A. Understand the question
```
"never placed" → the customer must have ZERO matching rows in orders
```
This is the opposite shape from most questions — you're not looking for a match, you're looking for the *absence* of one.

### B–C. Tables and operations
Two equally valid tools exist for this, and choosing between them is the actual lesson:
1. `LEFT JOIN` + `WHERE order_id IS NULL` — join everything, then keep only the rows where the join "found nothing."
2. `NOT EXISTS` (correlated subquery) — for each customer, explicitly check "does at least one order exist for this person?" and keep the ones where the answer is no.
3. `NOT IN` — tempting, but dangerous here (see below).

```
Step 1 → For each customer, check whether ANY row exists in orders with that customer_id
Step 2 → Keep only customers where that check finds nothing
```

### G. Build — two correct approaches
**LEFT JOIN approach:**
```sql
SELECT c.customer_id, c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;
```
**NOT EXISTS approach:**
```sql
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

### 🔁 Worth noticing — the NULL trap
A third option looks equivalent but isn't safe:
```sql
-- Risky if customer_id can ever be NULL in orders:
SELECT c.customer_id, c.name
FROM customers c
WHERE c.customer_id NOT IN (SELECT customer_id FROM orders);
```
If even **one row** in `orders.customer_id` is NULL (e.g., a corrupted or guest-checkout record), `NOT IN` returns an empty result set for the *entire query* — because SQL can't prove any value is "not equal to NULL," so the whole IN-list comparison becomes unknown. `NOT EXISTS` doesn't have this problem, because it never compares against the NULL value directly — it just checks row existence. **Rule of thumb: prefer `NOT EXISTS` over `NOT IN` whenever the subquery column could contain NULLs (which, unless you've verified otherwise, you should assume is possible).**

---

## Question 11 — TrustBank (revisited)

**Tables:** same as Question 6, plus `branches(branch_id, branch_name, city)`.

**Question:** *"List every account whose balance is higher than the average balance of accounts at the same branch."*

### A. Understand the question
This is the exact pattern from the prompt's own example ("customers who spent more than the average customer in their city") — same shape, different scenario. That repetition is intentional: recognizing the *pattern*, not the *scenario*, is the actual skill.
```
Account
  ↓
Balance
  ↓
Compare to: the average balance, but only among accounts at the SAME branch
  ↓
This average is different for every account being checked
```
The phrase *"the same branch"* is the giveaway: the comparison value isn't one fixed number for the whole table — it's a *different* number depending on which row you're looking at. That's the signature of a **correlated subquery**.

### E–F. Inner problem and subquery shape
> What do I need first? → The average balance *for this specific account's branch* — recomputed for every row being checked.

This subquery returns one number, but that number depends on the outer row (`account.branch... ` via the account's owner). It's still scalar, but *correlated* — it re-executes conceptually once per outer row, using `account_id`'s branch as context. Since `accounts` doesn't have `branch_id` directly in this schema, you first need it via `customers`.

*(Schema note: assume `accounts.customer_id → customers.branch_id` gives the branch.)*

### C–D. Order of thought
```
Step 1 → For the current account's owner, find their branch_id
Step 2 → Compute the average balance across all accounts belonging to customers at that same branch
Step 3 → Compare the current account's balance to that average
Step 4 → Repeat for every account (this is what "correlated" means)
```

### G. Build
```sql
SELECT a.account_id, a.balance
FROM accounts a
JOIN customers c ON c.customer_id = a.customer_id
WHERE a.balance > (
    SELECT AVG(a2.balance)
    FROM accounts a2
    JOIN customers c2 ON c2.customer_id = a2.customer_id
    WHERE c2.branch_id = c.branch_id   -- correlation to the outer row
);
```

### 🔁 Worth noticing
The inner query references `c.branch_id` from the *outer* query — that's what makes it correlated, not non-correlated. A non-correlated subquery (like Question 9's platform-wide average) computes one fixed answer regardless of which outer row you're on. A correlated subquery recomputes *conceptually per row*. Correlated subqueries are more expensive but are the only way to express "compared to peers in the same group" without a window function.

---

## Question 12 — CityCare Hospital (revisited)

**Tables:** same as Question 3.

**Question:** *"List doctors who have zero appointments scheduled for the coming week."*

### A. Understand the question
```
"zero appointments... coming week" → absence check, scoped to a date range
```
Structurally identical to Question 10 (the "never" pattern), but now the absence is scoped to a *date window*, not "ever."

### B–D. Tables and operations
`doctors`, and a correlated existence check against `appointments` filtered by date.
```
Step 1 → For each doctor, check: does an appointment exist for them where appointment_date falls in the next 7 days?
Step 2 → Keep the doctor if that check finds nothing
```

### G. Build
```sql
SELECT d.doctor_id, d.name
FROM doctors d
WHERE NOT EXISTS (
    SELECT 1
    FROM appointments a
    WHERE a.doctor_id = d.doctor_id
      AND a.appointment_date >= CURRENT_DATE
      AND a.appointment_date < CURRENT_DATE + INTERVAL '7 days'
);
```
*(Dialect note: `CURRENT_DATE + INTERVAL '7 days'` is Postgres-style. In MySQL, use `DATE_ADD(CURRENT_DATE, INTERVAL 7 DAY)`; in SQL Server, `DATEADD(day, 7, CAST(GETDATE() AS DATE))`.)*

### 🔁 Worth noticing — EXISTS vs IN, when both look possible
You *could* write this with `doctor_id NOT IN (SELECT doctor_id FROM appointments WHERE ...)`, but it carries the same NULL risk as Question 10. `EXISTS`/`NOT EXISTS` is also often faster on large tables because the engine can stop scanning as soon as it finds one match — it doesn't need to build the full candidate list first the way `IN` conceptually does.

---

## Question 12B — NexaTech (ANY vs ALL)

**Tables:** `employees`, `departments`.

**Question:** *"Find every employee who earns more than every employee in the Sales department. Then, separately, find every employee who earns more than at least one employee in the Sales department."*

### A. Understand the question — two questions, deliberately close together
```
"more than EVERY employee in Sales"        → must beat the HIGHEST Sales salary
"more than AT LEAST ONE employee in Sales" → only needs to beat the LOWEST Sales salary
```
Both comparisons are against a **list** of salaries, not a single number — Sales has many employees — so a plain `salary > (subquery)` won't work; a scalar subquery must return exactly one value. This is precisely the situation `ALL` and `ANY` exist for.

### E–F. What the subquery returns
A **multi-row, single-column subquery** — every salary in the Sales department. The question is what the outer comparison should mean against that whole list:
- `> ALL (list)` → greater than *every* value in the list → equivalent to `> MAX(list)`.
- `> ANY (list)` (equivalently `> SOME (list)`) → greater than *at least one* value → equivalent to `> MIN(list)`.

### D. Order of thought
```
Step 1 → Subquery: all salaries where department = 'Sales'
Step 2 → Outer query: compare each employee's salary against that whole list
Step 3 → Pick ALL ("beats everyone") or ANY ("beats someone") based on the wording
```

### G. Build
```sql
-- Beats every Sales employee (equivalent to salary > MAX(Sales salaries))
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > ALL (
    SELECT e2.salary FROM employees e2
    JOIN departments d ON d.department_id = e2.department_id
    WHERE d.department_name = 'Sales'
);

-- Beats at least one Sales employee (equivalent to salary > MIN(Sales salaries))
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > ANY (
    SELECT e2.salary FROM employees e2
    JOIN departments d ON d.department_id = e2.department_id
    WHERE d.department_name = 'Sales'
);
```

### 🔁 Worth noticing
Useful equivalences worth memorizing rather than deriving each time: `= ANY (...)` behaves like `IN (...)`; `<> ALL (...)` behaves like `NOT IN (...)`. In practice, `MAX`/`MIN` or `IN`/`NOT IN` are usually clearer to a future reader than `ALL`/`ANY` — but `ALL`/`ANY` show up often enough in interviews and older Oracle codebases that recognizing them on sight matters.

---

## Question 12C — NexaTech (Multi-Column Subquery)

**Tables:** `employees`, `departments`.

**Question:** *"Find the top earner in each department, in a single query — if two employees tie for the top salary in a department, return both."*

### A. Understand the question
```
"top earner IN EACH department"    → the comparison value (max salary) is different per department
"if two tie... return both"        → rules out any approach that arbitrarily picks one row (like LIMIT 1)
```

### E–F. The inner problem and what it returns
> What do I need first? → For every department, its highest salary — but paired *with* the department, not as a separate fact.

A subquery of `SELECT department_id, MAX(salary) FROM employees GROUP BY department_id` returns **multiple rows, and each row has two columns**. To use this correctly, the outer query must match on *both columns together* — matching only on `salary` risks matching an employee in the wrong department who happens to earn the same amount; matching only on `department_id` doesn't identify the earner at all. That's a **multi-column subquery**, compared with `IN` against a pair.

### D. Order of thought
```
Step 1 → Subquery: GROUP BY department, MAX(salary) → (department_id, max_salary) pairs
Step 2 → Outer query: keep employees whose (department_id, salary) matches one of those pairs exactly
```

### G. Build
```sql
SELECT e.name, e.department_id, e.salary
FROM employees e
WHERE (e.department_id, e.salary) IN (
    SELECT department_id, MAX(salary)
    FROM employees
    GROUP BY department_id
);
```

### 🔁 Worth noticing
A correlated subquery could answer the same question — `WHERE salary = (SELECT MAX(salary) FROM employees e2 WHERE e2.department_id = e.department_id)` — and is arguably more intuitive to read. The multi-column form does the same job without correlation, in one logical pass. Both are correct; recognizing that a problem is really "match on a pair of columns at once" — rather than trying to force it into two separate single-column conditions — is the actual skill being tested here.

---

# LEVEL 4 — Complex Combinations

These questions ask for several things at once. The skill is decomposing the sentence into independent sub-goals *before* worrying about how they'll fit into one query.

## Question 13 — QuickBite (revisited)

**Tables:** same as Question 4, plus `riders(rider_id, name, city)`.

**Question:** *"For each rider, show total completed deliveries and total cancelled deliveries. Only include riders whose completed deliveries are more than double their cancelled ones, AND whose completed count is above the platform average. Label each qualifying rider 'Excellent' (150+ completed) or 'Good' (below 150), sorted by completed deliveries descending."*

### A. Understand the question — decompose into independent pieces
```
Piece 1: per rider, count completed AND cancelled (conditional aggregation)
Piece 2: keep only riders where completed > 2 × cancelled (HAVING, comparing two aggregates to each other)
Piece 3: keep only riders where completed > platform-wide average completed (HAVING, comparing to a subquery)
Piece 4: label via CASE, based on the completed count
Piece 5: sort by completed, descending
```
Five sub-goals. None of them are individually hard — the difficulty is holding all five without losing one.

### E–F. The inner problem
> What do I need first? → The platform-wide average number of *completed* deliveries per rider (piece 3). Same shape as Question 9: a scalar subquery built from a grouped derived table.

### C–D. Order of thought
```
Step 1 → Group delivery_orders by rider
Step 2 → Conditionally count completed and cancelled per rider (CASE inside COUNT/SUM)
Step 3 → Compute the platform average of completed-per-rider (subquery)
Step 4 → HAVING: completed > 2*cancelled AND completed > platform average
Step 5 → CASE label based on completed count
Step 6 → ORDER BY completed DESC
```

### G. Build
```sql
SELECT r.name,
       SUM(CASE WHEN o.status = 'completed' THEN 1 ELSE 0 END) AS completed,
       SUM(CASE WHEN o.status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled,
       CASE
           WHEN SUM(CASE WHEN o.status = 'completed' THEN 1 ELSE 0 END) >= 150 THEN 'Excellent'
           ELSE 'Good'
       END AS performance_label
FROM riders r
JOIN delivery_orders o ON o.rider_id = r.rider_id
GROUP BY r.rider_id, r.name
HAVING SUM(CASE WHEN o.status = 'completed' THEN 1 ELSE 0 END)
       > 2 * SUM(CASE WHEN o.status = 'cancelled' THEN 1 ELSE 0 END)
   AND SUM(CASE WHEN o.status = 'completed' THEN 1 ELSE 0 END) > (
        SELECT AVG(completed_count) FROM (
            SELECT rider_id, SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) AS completed_count
            FROM delivery_orders
            GROUP BY rider_id
        ) rider_totals
   )
ORDER BY completed DESC;
```

### H. Tie-back
Every clause maps to one sentence fragment from the question: `SUM(CASE...)` twice → "completed and cancelled"; first `HAVING` condition → "more than double"; the subquery condition → "above platform average"; the `CASE` in `SELECT` → the label; `ORDER BY` → "sorted by completed."

---

## Question 14 — NexaTech (revisited)

**Tables:** `employees`, `departments`, plus `projects(project_id, project_name, department_id, budget)` and `assignments(assignment_id, employee_id, project_id, hours_allocated, role)`.

**Question:** *"Show department names where at least 2 employees are assigned to more than one project each — but only for departments whose total project budget is above the company-wide average project budget."*

### A. Understand the question — two independent inner problems
```
Inner problem 1: "employees assigned to more than one project each"
   → per employee, count DISTINCT projects, keep those with count > 1
   → then per department, count how many such employees exist, keep departments with >= 2

Inner problem 2: "total project budget above company-wide average"
   → per department, SUM(budget); separately, AVG(budget) across ALL departments
```
Two unrelated calculations, both filtering the same final department list. This is a case where breaking the problem into two derived tables, evaluated independently, then combined, is far more manageable than trying to write one flat query.

### E–F. What each subquery returns
- Inner problem 1's first stage: a derived table of `(employee_id, department_id, project_count)`, one row per employee — a **multi-row, multi-column derived table** (subquery in FROM).
- Inner problem 2: a **scalar subquery** — one number, the company-wide average budget.

### C–D. Order of thought
```
Step 1 → Derived table: employees × distinct project count, keep those > 1 project
Step 2 → Group that derived table by department, count qualifying employees, keep >= 2
Step 3 → Separately: department-level total budget vs company-wide average budget (scalar subquery)
Step 4 → Join the two results together on department
```

### G. Build
```sql
WITH multi_project_employees AS (
    SELECT e.employee_id, e.department_id
    FROM employees e
    JOIN assignments a ON a.employee_id = e.employee_id
    GROUP BY e.employee_id, e.department_id
    HAVING COUNT(DISTINCT a.project_id) > 1
),
dept_multi_project_counts AS (
    SELECT department_id, COUNT(*) AS multi_project_employee_count
    FROM multi_project_employees
    GROUP BY department_id
    HAVING COUNT(*) >= 2
)
SELECT dep.department_name
FROM departments dep
JOIN dept_multi_project_counts c ON c.department_id = dep.department_id
JOIN projects p ON p.department_id = dep.department_id
GROUP BY dep.department_id, dep.department_name
HAVING SUM(p.budget) > (SELECT AVG(budget) FROM projects);
```

*(This uses `WITH` / CTEs, which are functionally subqueries in FROM given readable names — worth knowing both forms, since interview settings sometimes disallow CTEs.)*

### 🔁 Worth noticing
This question shows *why* subqueries in FROM exist: some conditions ("more than one project per employee") must be fully resolved and finalized as their own grouped result before you can ask a second question about that result ("at least 2 such employees per department"). Trying to nest both `HAVING` conditions into a single `GROUP BY employee` pass doesn't work, because they operate at *different grains* — one row per employee vs. one row per department. Changing grain mid-query is exactly what a derived table/CTE is for.

---

## Question 15 — StreamPlex (revisited)

**Tables:** same as Question 9.

**Question:** *"For each piece of content, show its title, total minutes watched, and whether it's 'Above Average' or 'Below Average' compared to the platform-wide average minutes watched per title."*

### A. Understand the question
```
"total minutes watched" per title    → SUM, GROUP BY content
"platform-wide average"              → one fixed number, computed once, same for every row
"Above/Below Average label"          → CASE comparing the two
```
Unlike Question 11 (correlated, recomputed per row), this average is the *same single number* for every title — a **non-correlated scalar subquery**, usable directly inside the SELECT list or the CASE expression.

### C–D. Order of thought
```
Step 1 → Group watch_history by content, sum minutes_watched
Step 2 → Separately compute one number: the average of those per-title sums, across all titles
Step 3 → For each title, compare its sum to that single number via CASE
```

### G. Build
```sql
SELECT c.title,
       SUM(w.minutes_watched) AS total_minutes,
       CASE
           WHEN SUM(w.minutes_watched) > (
                SELECT AVG(title_minutes) FROM (
                    SELECT content_id, SUM(minutes_watched) AS title_minutes
                    FROM watch_history
                    GROUP BY content_id
                ) per_title
           ) THEN 'Above Average'
           ELSE 'Below Average'
       END AS performance
FROM content c
JOIN watch_history w ON w.content_id = c.content_id
GROUP BY c.content_id, c.title;
```

### 🔁 Worth noticing
This is a **scalar subquery used inside a SELECT-list expression (via CASE)**, distinct from Questions 9 and 13 where the scalar subquery sat inside `HAVING`. Same subquery-writing skill, different *location* — the location depends only on whether you're **filtering rows** (HAVING/WHERE) or **producing a value to display** (SELECT).

---

## Question 15B — ShopEase (FULL OUTER JOIN)

**Tables:**
- `products(product_id, product_name, category, price)`
- `inventory_counts(count_id, product_id, counted_stock, count_date)` — this month's physical stock audit

**Question:** *"Compare this month's physical stock count against the product catalog. Show every product that appears on either side. Flag catalog products that were never physically counted, counted items that don't match any catalog product (likely data-entry errors), and items that matched normally."*

### A. Understand the question — why not a LEFT or INNER join
```
"every product that appears on EITHER side"   → keep unmatched rows from BOTH tables, not just one
"catalog products never counted"              → missing on the inventory_counts side
"counted items with no catalog match"         → missing on the products side
```
An `INNER JOIN` would silently drop both problem cases — it only keeps matches. A `LEFT JOIN` would catch "never counted" but would hide "counted but not in catalog" (those rows simply wouldn't appear at all, since they don't exist on the left/products side). You need **both directions of "what's missing" at once** — that phrase pattern ("everything from either side, tell me what's missing from each") is the signature of a `FULL OUTER JOIN`.

### B. Tables needed
Both, matched on `product_id`, with an outer join in both directions.

### C–D. Operations and order
```
Step 1 → FULL OUTER JOIN products to inventory_counts on product_id
Step 2 → For display, coalesce product_id (since a FULL join can leave either side NULL)
Step 3 → CASE, based on WHICH side is NULL, to produce the reconciliation label
```

### G. Build
```sql
SELECT COALESCE(p.product_id, ic.product_id) AS product_id,
       p.product_name,
       ic.counted_stock,
       CASE
           WHEN p.product_id IS NULL  THEN 'Data Entry Error - Not in Catalog'
           WHEN ic.product_id IS NULL THEN 'Not Counted This Month'
           ELSE 'Matched'
       END AS reconciliation_status
FROM products p
FULL OUTER JOIN inventory_counts ic ON ic.product_id = p.product_id;
```

### H. Tie-back
`COALESCE(p.product_id, ic.product_id)` exists specifically because a `FULL OUTER JOIN` can leave *either* side NULL for a given row — you need "whichever ID actually exists" to display a usable identifier at all. Each `WHEN` in the `CASE` corresponds directly to one of the three states the question asked for.

### 🔁 Worth noticing
This is the textbook real-world reason `FULL OUTER JOIN` exists: reconciling two datasets that are *supposed* to align but might not, in both directions simultaneously. It's genuinely rare in everyday reporting — most business questions favor one side of a relationship, which is why `LEFT JOIN` dominates in practice — but for audits, data-migration checks, and reconciliation reports, it's the only join that answers the question correctly in a single pass.

---

# LEVEL 5 — Interview-Style Traps

These are designed so the "obvious" first instinct is often wrong, or at least imprecise. Read the wording twice before deciding on an approach.

## Question 16 — NexaTech: the classic "every" trap

**Tables:** `employees`, `departments`.

**Question:** *"Find departments where every single employee earns more than the company-wide average salary."*

### A. Understand the question — the trap
The instinct: filter employees to `salary > company_avg`, group by department, done. **That's wrong.** That query finds departments that *contain at least one* high earner — not departments where *every* employee qualifies. "Every" is a universal condition, and SQL has no native "for all" operator — you have to flip it into its logical opposite.

```
"every employee earns more than average"
   ⇔ (logically equivalent to)
"there is NO employee in this department who earns average-or-less"
```
This flip — turning "for all X, condition holds" into "there does not exist an X where condition fails" — is the single most important trick for "every / only / all" questions in SQL.

### B–C. Tables and operations
`employees` filtered by the company-wide average (a non-correlated scalar subquery), then a `NOT EXISTS` check per department for any disqualifying employee.

### E–F. Two nested subqueries, two different jobs
- Subquery 1 (scalar, non-correlated): the one company-wide average salary number.
- Subquery 2 (correlated, existence check): "does this department contain anyone who fails the condition?"

### D. Order of thought
```
Step 1 → Compute company-wide average salary (once, scalar subquery)
Step 2 → For each department, check: does ANY employee here earn <= that average?
Step 3 → Keep departments where that check finds nobody (NOT EXISTS)
```

### G. Build
```sql
SELECT DISTINCT dep.department_name
FROM departments dep
WHERE NOT EXISTS (
    SELECT 1
    FROM employees e
    WHERE e.department_id = dep.department_id
      AND e.salary <= (SELECT AVG(salary) FROM employees)
)
AND EXISTS (   -- guard against departments with zero employees "vacuously" qualifying
    SELECT 1 FROM employees e2 WHERE e2.department_id = dep.department_id
);
```

### 🔁 Worth noticing
Two things people miss on this exact question:
1. Without the second `EXISTS` guard, a department with **zero employees** technically satisfies "every employee earns more than average" — vacuously true, since there's no employee to violate it. Whether that's "correct" depends on business intent, but an interviewer will usually want you to *notice and address it*, not just get lucky.
2. This is functionally a **division problem** ("departments where the employee set has no counterexample") — the double-negative NOT EXISTS pattern is the standard way to express it without window functions.

---

## Question 17 — SkyHigh Airlines: second highest

**Tables:** same as Question 2.

**Question:** *"For each flight, find the second-highest fare paid by any passenger."*

### A. Understand the question
```
"second highest, PER FLIGHT" → not a global second-highest; recomputed per flight
```
The "per flight" is easy to miss and changes the entire approach — a naive `ORDER BY fare DESC LIMIT 1 OFFSET 1` only works for a *single* flight, not for every flight simultaneously.

### C–D. Approaches, and choosing between them
**Approach 1 — correlated subquery, counting fares above the current one:**
For a given fare on a given flight, it is the "second highest" exactly when there is precisely **one** other fare on that same flight strictly greater than it.
```
Step 1 → For each booking, count how many OTHER bookings on the same flight have a strictly higher fare
Step 2 → Keep the booking(s) where that count = 1
```

### G. Build (Approach 1 — portable across all SQL dialects)
```sql
SELECT b1.flight_id, b1.fare AS second_highest_fare
FROM bookings b1
WHERE 1 = (
    SELECT COUNT(*)
    FROM bookings b2
    WHERE b2.flight_id = b1.flight_id
      AND b2.fare > b1.fare
);
```

### 🔁 Worth noticing — alternative approach
On dialects that support window functions this becomes a one-liner with `DENSE_RANK()` — but since this training deliberately excludes window functions, the correlated self-comparison above is the technique to master; it's also the one most likely to be asked about explicitly, precisely *because* it forces you to reason about ranking without a built-in ranking tool. Note also: this uses `DENSE`-style logic (ties for the highest fare don't break it) rather than `ROW_NUMBER`-style logic (which would need a tie-breaker); if the business meaning is "second distinct fare value," this is correct — if it means "second row when sorted," that's a different, more fragile question to clarify before writing code.

---

## Question 18 — ShopEase: INSERT with aggregation

**Tables:** same as Question 1, plus a new empty table `vip_customers(customer_id, name, total_spent)`.

**Question:** *"Populate the vip_customers table with every customer whose total spending across all their orders exceeds ₹5,000."*

### A. Understand the question
```
"populate a table"                    → INSERT
"total spending... exceeds 5000"      → this filter is on an AGGREGATE (SUM), so it needs GROUP BY + HAVING first, computed as a SELECT, then inserted
```
The mental model: **write the SELECT first as if you were just answering "who qualifies," get it fully correct and tested on its own, then wrap it in an INSERT.** Never write INSERT and the aggregation logic simultaneously — that's how mistakes happen.

*(Note: this schema's `orders` doesn't carry amount directly — assume total spend requires joining to `order_items` for real e-commerce logic. For this exercise, assume `orders` has an `amount` column for simplicity.)*

### D. Order of thought
```
Step 1 → Write and verify: SELECT customer_id, name, SUM(amount) FROM ... GROUP BY ... HAVING SUM(amount) > 5000
Step 2 → Once correct, prepend INSERT INTO vip_customers (columns) 
Step 3 → No VALUES — the SELECT itself supplies the rows
```

### G. Build
Step 1, verify alone:
```sql
SELECT c.customer_id, c.name, SUM(o.amount) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.name
HAVING SUM(o.amount) > 5000;
```
Step 2, wrap it:
```sql
INSERT INTO vip_customers (customer_id, name, total_spent)
SELECT c.customer_id, c.name, SUM(o.amount) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.name
HAVING SUM(o.amount) > 5000;
```

### 🔁 Worth noticing
`INSERT ... SELECT` doesn't introduce any new logical concept — it's a delivery mechanism wrapped around a SELECT you should already be able to write and trust on its own. The discipline of testing the SELECT in isolation first (and checking the row count / spot-checking a few rows) before wrapping it in INSERT is what prevents inserting thousands of wrong rows in one shot.

---

## Question 19 — TrustBank: UPDATE with a correlated subquery

**Tables:** `accounts`, `transactions` (from Question 6). Assume `accounts` has a `status` column.

**Question:** *"Mark as 'Dormant' every account that has had no transactions in the last 12 months."*

### A. Understand the question
```
"no transactions in the last 12 months"   → same "never/absence" shape as Questions 10 and 12, just scoped by a rolling date window
"mark as Dormant"                          → UPDATE, not SELECT
```
Same instinct as Question 18: get the identification logic right as a SELECT first, then convert to UPDATE.

### D. Order of thought
```
Step 1 → Write and verify: SELECT accounts that have NO transaction within the last 12 months (NOT EXISTS, correlated, date-filtered)
Step 2 → Once verified, convert to UPDATE ... SET status = 'Dormant' WHERE <same condition>
```

### G. Build
Step 1, verify:
```sql
SELECT a.account_id
FROM accounts a
WHERE NOT EXISTS (
    SELECT 1 FROM transactions t
    WHERE t.account_id = a.account_id
      AND t.transaction_date >= CURRENT_DATE - INTERVAL '12 months'
);
```
Step 2, convert:
```sql
UPDATE accounts a
SET status = 'Dormant'
WHERE NOT EXISTS (
    SELECT 1 FROM transactions t
    WHERE t.account_id = a.account_id
      AND t.transaction_date >= CURRENT_DATE - INTERVAL '12 months'
);
```

### 🔁 Worth noticing
Same `NOT EXISTS` vs `NOT IN` NULL trap from Question 10 applies here — if `transactions.account_id` could ever be NULL, `NOT IN` would silently update *zero* accounts (or all, depending on how you phrase it), and there'd be no error to alert you. `NOT EXISTS` sidesteps this entirely. When an UPDATE or DELETE is involved, this isn't just a style preference — a NULL-related bug here means silently corrupting real data.

---

## Question 20 — Riverdale University: DELETE with a subquery

**Tables:** `courses(course_id, course_name, department, credits)`, `enrollments(enrollment_id, student_id, course_id, semester, grade)`.

**Question:** *"Delete every course from the catalog that has never had a single student enroll in it."*

### A. Understand the question
Same absence pattern as Questions 10, 12, and 19 — only now the row that "doesn't exist" isn't just being reported, it's being **removed**. That raises the stakes on getting the logic exactly right before running anything.

### D. Order of thought
```
Step 1 → Write and verify: SELECT courses where NO row exists in enrollments for that course_id
Step 2 → Once verified against real output, convert SELECT → DELETE, same WHERE clause
```

### G. Build
```sql
-- Step 1: verify first — never skip this on a DELETE
SELECT c.course_id, c.course_name
FROM courses c
WHERE NOT EXISTS (
    SELECT 1 FROM enrollments e WHERE e.course_id = c.course_id
);

-- Step 2: convert once the above looks correct
DELETE FROM courses c
WHERE NOT EXISTS (
    SELECT 1 FROM enrollments e WHERE e.course_id = c.course_id
);
```

### 🔁 Worth noticing
The tempting shortcut — `DELETE FROM courses WHERE course_id NOT IN (SELECT course_id FROM enrollments)` — carries the same NULL risk as Questions 10 and 19. If `enrollments.course_id` ever contains a single NULL (a bad migration row, an orphaned record), this `DELETE` silently removes **nothing at all**, with no error thrown. For a destructive statement, that's the worst possible failure mode — it looks like it succeeded. Default to `NOT EXISTS` for any absence-based `DELETE` or `UPDATE` unless you've explicitly confirmed the subquery column can never be NULL.

---

# BONUS — Practice Without Solutions

Try these on your own using the framework below before checking your logic against a peer, documentation, or a fresh conversation. Hints only.

1. **Riverdale University** — *"Which students have taken more courses than the average number of courses taken by students in their own major?"* (Same shape as Question 11 — correlated by major instead of by branch. Can you spot the pattern without re-reading Q11?)

2. **StreamPlex** — *"Find users who have watched at least one title from every genre available on the platform."* (This is a true division problem — "every genre," similar spirit to Question 16's "every employee," but comparing against a *set* of genres rather than a single average. Try flipping it to "no genre exists that this user hasn't touched.")

3. **QuickBite** — *"Which restaurants have at least one rider who has never cancelled a single delivery for them?"* (Combines EXISTS with a nested NOT EXISTS — a double-negation question. Read it very slowly.)

4. **TrustBank** — *"Show every branch, whether or not it has any accounts, and every account that somehow references a branch_id no longer in the branches table (orphaned data). One query, one result set."* (A FULL OUTER JOIN data-quality question, same family as Question 15B — try it in a totally different scenario before checking your instinct against that one.)

---

# APPENDIX — Syllabus Reference (Setup & Administrative SQL)

Everything above trains *logical decomposition* — the skill your original ask was about. The items below are genuinely part of a full Oracle SQL syllabus, but they aren't "read the question, break it into steps" puzzles — they're mostly recall and syntax. They're included here for completeness against the syllabus, in quick-reference form rather than the A–H format. (Views and Indexes are skipped per your instruction; ordinary/equi-joins were already covered inline above and aren't repeated here.)

### DDL — Data Definition Language
Defines and changes structure, not data.
```sql
CREATE TABLE departments (department_id INT PRIMARY KEY, department_name VARCHAR2(50));
ALTER TABLE employees ADD (bonus NUMBER(10,2));
RENAME employees TO staff;
DROP TABLE staff;
TRUNCATE TABLE transactions;   -- removes ALL rows, resets storage; cannot be rolled back
```
**Worth knowing:** `TRUNCATE` vs `DELETE` is a common trap. `DELETE FROM transactions` removes rows one at a time, can be filtered with `WHERE`, fires triggers, and can be rolled back before COMMIT. `TRUNCATE` removes everything at once, can't be filtered, and in Oracle is a DDL statement — it auto-commits and can't be undone with `ROLLBACK`. `DROP` removes the table structure itself, not just its rows.

### DML — Data Manipulation Language
Already covered in depth above (`INSERT...SELECT` in Q18, `UPDATE` in Q19, `DELETE` in Q20) — those are the logic-heavy versions. Plain single-row forms:
```sql
INSERT INTO customers (customer_id, name, city) VALUES (101, 'Asha Rao', 'Chennai');
UPDATE customers SET city = 'Bengaluru' WHERE customer_id = 101;
DELETE FROM customers WHERE customer_id = 101;
```

### DCL — Data Control Language
Permissions, not data or structure.
```sql
GRANT SELECT, INSERT ON employees TO analyst_role;
REVOKE INSERT ON employees FROM analyst_role;
```

### TCL — Transaction Control Language
Controls when changes made by DML actually become permanent.
```sql
UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
SAVEPOINT before_credit;
UPDATE accounts SET balance = balance + 500 WHERE account_id = 2;
-- if something looks wrong:
ROLLBACK TO before_credit;
-- once satisfied:
COMMIT;
```
**Worth knowing:** this is exactly why Questions 18–20 stressed "verify the SELECT before running the write." `COMMIT` makes changes permanent for everyone; `ROLLBACK` (with no savepoint) undoes every uncommitted change in the current transaction; a `SAVEPOINT` lets you undo *part* of a transaction without discarding all of it.

### Constraints
```sql
CREATE TABLE employees (
    employee_id   INT PRIMARY KEY,
    email         VARCHAR2(100) UNIQUE NOT NULL,
    department_id INT REFERENCES departments(department_id),   -- FOREIGN KEY
    salary        NUMBER(10,2) CHECK (salary > 0)
);
CREATE SEQUENCE employee_seq START WITH 1 INCREMENT BY 1;      -- for generating IDs
ALTER TABLE employees DISABLE CONSTRAINT emp_salary_chk;
ALTER TABLE employees ENABLE CONSTRAINT emp_salary_chk;
```
**Worth knowing, and it connects directly to the questions above:** a `FOREIGN KEY` is the database *enforcing* the same relationship you've been navigating manually with `JOIN` this whole time — `orders.customer_id REFERENCES customers.customer_id` is the formal version of "this is why these two tables join cleanly." `NOT NULL` constraints are also *why* some of the NULL traps in Questions 10/19/20 don't happen in a well-constrained schema — but you should never assume that's guaranteed, especially on columns you didn't design yourself.

### Remaining JOIN syntax (beyond INNER/LEFT/SELF/FULL, already used above)
```sql
-- CROSS JOIN: every row of A with every row of B (Cartesian product) — rare on
-- purpose, e.g. generating all (product × size) combinations for a form
SELECT p.product_name, s.size_label
FROM products p
CROSS JOIN sizes s;

-- JOIN ... USING: shorthand when the join column has the SAME NAME in both tables
SELECT o.order_id, c.name
FROM orders o
JOIN customers c USING (customer_id);   -- equivalent to ON o.customer_id = c.customer_id

-- NATURAL JOIN: joins automatically on ALL identically-named columns —
-- convenient, but risky: a new column added later with a matching name on
-- either table silently changes what the query does. Prefer explicit ON.
SELECT * FROM orders NATURAL JOIN customers;

-- RIGHT OUTER JOIN: mirror of LEFT JOIN — keeps every row from the right
-- table even without a match. Rarely used in practice, since swapping table
-- order and using LEFT JOIN instead reads more naturally to most people.
SELECT c.name, o.order_id
FROM orders o
RIGHT JOIN customers c ON c.customer_id = o.customer_id;
```

### `FETCH FIRST` (Oracle's modern row-limiting clause)
```sql
SELECT name, salary FROM employees
ORDER BY salary DESC
FETCH FIRST 5 ROWS ONLY;
```
This is Oracle 12c+ ANSI-standard syntax, equivalent to MySQL/Postgres `LIMIT 5` or older Oracle's `WHERE ROWNUM <= 5` (which behaves subtly differently, since `ROWNUM` is assigned *before* `ORDER BY` conceptually — a classic Oracle gotcha).

### `NULLIF`
The inverse companion to `COALESCE`: returns NULL if two expressions are equal, otherwise returns the first one. Most common use — safely avoiding a divide-by-zero:
```sql
SELECT customer_id, total_spent / NULLIF(order_count, 0) AS avg_order_value
FROM customer_summary;
-- if order_count is 0, NULLIF returns NULL instead of 0, so the division
-- returns NULL instead of throwing a divide-by-zero error
```

### Order of execution of clauses in a SELECT statement
The order you *write* a query is not the order the database *evaluates* it — this explains several rules used throughout this document (like why `WHERE` can't see an alias defined in `SELECT`, or why `HAVING` can use aggregates but `WHERE` can't):
```
FROM  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  ORDER BY  →  FETCH FIRST
```

### Deterministic vs. non-deterministic functions
A **deterministic** function always returns the same output for the same input (`UPPER('abc')` is always `'ABC'`). A **non-deterministic** function can return different results on different calls with identical input (`SYSDATE`, `CURRENT_TIMESTAMP`, sequence `.NEXTVAL`). This matters mainly for things like function-based indexes and materialized views (out of scope here) — but it's worth knowing why a query using `SYSDATE` in a `WHERE` clause can return different rows tomorrow with no data having changed at all.

---

# THE REUSABLE MENTAL FRAMEWORK

Apply this to any unfamiliar SQL question, before touching a keyboard:

```
1.  What exactly is the final output — one row per what?
2.  Which table(s) contain each piece of information I need?
3.  Do I need to combine tables? If so, will an INNER join silently
    drop rows I actually need (watch for "including/never/zero/none")?
4.  Is there a row-level condition (true/false per individual row,
    knowable before any grouping)? → WHERE
5.  Do I need to compute something (a count, sum, average)?
    → requires GROUP BY, at the right grain (per customer? per
    department? per city? — re-read the question for this)
6.  Is there a condition on that COMPUTED value itself
    (a total, an average, a count)? → HAVING, not WHERE
7.  Do I need two or more numbers side-by-side that each depend on
    a different condition, in the same row? → conditional
    aggregation (SUM/COUNT with CASE inside)
8.  Do I need a label or bucket based on a value? → CASE in SELECT
9.  Does answering this question require knowing something else
    FIRST — a threshold, an average, a max, a list — before I can
    filter or compare? → that "something else" is a subquery.
        a. Does it return ONE value, fixed for the whole query?
           → non-correlated scalar subquery.
        b. Does it return ONE value, but a DIFFERENT one depending
           on the outer row ("compared to peers in the same X")?
           → correlated subquery.
        c. Does it return a LIST of values to check membership
           against? → IN / NOT IN (but prefer EXISTS/NOT EXISTS
           if the column could contain NULLs).
        d. Am I checking whether something EXISTS or DOESN'T EXIST
           at all, rather than comparing a value? → EXISTS / NOT
           EXISTS.
        e. Does the question say "every / all / only", implying a
           universal condition with no direct SQL operator?
           → flip it: "NOT EXISTS a counterexample."
10. Am I comparing a row to another row of the SAME KIND, in the
    SAME table (employee vs. manager, product vs. product)?
    → self-join, not a subquery.
11. Do I need to eliminate duplicate rows with NO computation
    involved? → DISTINCT. If computation IS involved, it's
    GROUP BY instead.
12. Do multiple UNRELATED conditions apply at DIFFERENT grains
    (one per-employee, one per-department)? → resolve them as
    separate derived tables / CTEs first, then combine — don't
    force one GROUP BY to answer two different-grained questions.
13. Is this a write operation (INSERT/UPDATE/DELETE)? → write and
    fully verify the SELECT that identifies the correct rows FIRST,
    in isolation. Only wrap it in INSERT/UPDATE/DELETE once you
    trust it.
14. How should the final result be ordered?
15. Read it once more: does every phrase in the original question
    map to something in my query? If a phrase has no home, I
    probably missed a requirement.
```

The habit worth building isn't memorizing this list — it's the pause between reading the question and writing `SELECT`. That pause, spent translating English into steps, is what separates "I know SQL syntax" from "I can solve SQL problems."