# SQL 50 — Oracle SQL Study Guide

> A note on how this guide was built: LeetCode's exact problem statements, table
> schemas, and sample data are their copyrighted content, so this isn't a copy
> of their pages. This guide covers the same 50 problem **titles/topics** from
> the Top SQL 50 study plan, but every task description, example dataset,
> thought-process breakdown, and solution below is written from scratch,
> targeting **Oracle SQL** (not the MySQL dialect LeetCode uses by default).
> Use this to learn the *patterns* — then go solve the real problems on
> LeetCode to check your work against their judge.
>
> Oracle-specific things you'll see a lot of below: `NVL`/`NVL2` instead of
> `IFNULL`, `DECODE`/`CASE`, `FETCH FIRST n ROWS ONLY` instead of `LIMIT`,
> `LISTAGG` instead of `GROUP_CONCAT`, analytic functions (`RANK`,
> `DENSE_RANK`, `LEAD`, `LAG`, `SUM() OVER`), `REGEXP_LIKE` for regex, and
> `TO_CHAR`/date arithmetic for date handling.

## Table of Contents

1. [Recyclable and Low Fat Products](#1-recyclable-and-low-fat-products)
2. [Find Customer Referee](#2-find-customer-referee)
3. [Big Countries](#3-big-countries)
4. [Article Views I](#4-article-views-i)
5. [Invalid Tweets](#5-invalid-tweets)
6. [Replace Employee ID With The Unique Identifier](#6-replace-employee-id-with-the-unique-identifier)
7. [Product Sales Analysis I](#7-product-sales-analysis-i)
8. [Customer Who Visited but Did Not Make Any Transactions](#8-customer-who-visited-but-did-not-make-any-transactions)
9. [Rising Temperature](#9-rising-temperature)
10. [Average Time of Process per Machine](#10-average-time-of-process-per-machine)
11. [Employee Bonus](#11-employee-bonus)
12. [Students and Examinations](#12-students-and-examinations)
13. [Managers with at Least 5 Direct Reports](#13-managers-with-at-least-5-direct-reports)
14. [Confirmation Rate](#14-confirmation-rate)
15. [Not Boring Movies](#15-not-boring-movies)
16. [Average Selling Price](#16-average-selling-price)
17. [Project Employees I](#17-project-employees-i)
18. [Percentage of Users Attended a Contest](#18-percentage-of-users-attended-a-contest)
19. [Queries Quality and Percentage](#19-queries-quality-and-percentage)
20. [Monthly Transactions I](#20-monthly-transactions-i)
21. [Immediate Food Delivery II](#21-immediate-food-delivery-ii)
22. [Game Play Analysis IV](#22-game-play-analysis-iv)
23. [Number of Unique Subjects Taught by Each Teacher](#23-number-of-unique-subjects-taught-by-each-teacher)
24. [User Activity for the Past 30 Days I](#24-user-activity-for-the-past-30-days-i)
25. [Product Sales Analysis III](#25-product-sales-analysis-iii)
26. [Classes More Than 5 Students](#26-classes-more-than-5-students)
27. [Find Followers Count](#27-find-followers-count)
28. [Biggest Single Number](#28-biggest-single-number)
29. [Customers Who Bought All Products](#29-customers-who-bought-all-products)
30. [The Number of Employees Which Report to Each Employee](#30-the-number-of-employees-which-report-to-each-employee)
31. [Primary Department for Each Employee](#31-primary-department-for-each-employee)
32. [Triangle Judgement](#32-triangle-judgement)
33. [Consecutive Numbers](#33-consecutive-numbers)
34. [Product Price at a Given Date](#34-product-price-at-a-given-date)
35. [Last Person to Fit in the Bus](#35-last-person-to-fit-in-the-bus)
36. [Count Salary Categories](#36-count-salary-categories)
37. [Employees Whose Manager Left the Company](#37-employees-whose-manager-left-the-company)
38. [Exchange Seats](#38-exchange-seats)
39. [Movie Rating](#39-movie-rating)
40. [Restaurant Growth](#40-restaurant-growth)
41. [Friend Requests II: Who Has the Most Friends](#41-friend-requests-ii-who-has-the-most-friends)
42. [Investments in 2016](#42-investments-in-2016)
43. [Department Top Three Salaries](#43-department-top-three-salaries)
44. [Fix Names in a Table](#44-fix-names-in-a-table)
45. [Patients With a Condition](#45-patients-with-a-condition)
46. [Delete Duplicate Emails](#46-delete-duplicate-emails)
47. [Second Highest Salary](#47-second-highest-salary)
48. [Group Sold Products By The Date](#48-group-sold-products-by-the-date)
49. [List the Products Ordered in a Period](#49-list-the-products-ordered-in-a-period)
50. [Find Users With Valid E-Mails](#50-find-users-with-valid-e-mails)

---

## 1. Recyclable and Low Fat Products
**Topic:** Basic `WHERE` filtering

**Table:** `Products(product_id NUMBER, low_fats CHAR(1), recyclable CHAR(1))` — the flag columns hold `'Y'` or `'N'`.

**Task:** Return the `product_id` of every product that is *both* low fat and recyclable.

**Example**

| product_id | low_fats | recyclable |
|---|---|---|
| 1 | Y | N |
| 2 | Y | Y |
| 3 | N | Y |

Expected output: `product_id = 2`.

**Thought process**
- Two independent boolean-ish flags → this is a pure filter problem, no joins or grouping needed.
- Just `AND` the two conditions in the `WHERE` clause.

**Oracle Solution**
```sql
SELECT product_id
FROM Products
WHERE low_fats = 'Y'
  AND recyclable = 'Y';
```

---

## 2. Find Customer Referee
**Topic:** `NULL` handling in `WHERE`

**Table:** `Customer(id NUMBER, name VARCHAR2(50), referee_id NUMBER)` — `referee_id` is nullable.

**Task:** Return the names of customers who were **not** referred by the customer with id `2`.

**Example**

| id | name | referee_id |
|---|---|---|
| 1 | Ann | NULL |
| 2 | Bob | NULL |
| 3 | Cid | 2 |
| 4 | Dan | NULL |

Expected output: `Ann, Bob, Dan` (Cid is excluded because referee_id = 2).

**Thought process**
- The trap: `referee_id != 2` alone silently drops every row where `referee_id IS NULL`, because `NULL != 2` evaluates to `NULL`, not `TRUE`.
- So you must explicitly allow `NULL` back in with `OR referee_id IS NULL`.

**Oracle Solution**
```sql
SELECT name
FROM Customer
WHERE referee_id IS NULL
   OR referee_id <> 2;
```

---

## 3. Big Countries
**Topic:** Basic `WHERE` with `OR`

**Table:** `World(name VARCHAR2(50), continent VARCHAR2(50), area NUMBER, population NUMBER, gdp NUMBER)`

**Task:** A country is "big" if its area is at least 3,000,000 **or** its population is at least 25,000,000. Return `name`, `population`, `area` for big countries.

**Example**

| name | area | population |
|---|---|---|
| Algeria | 2,381,741 | 44,700,000 |
| Fiji | 18,272 | 900,000 |

Expected output: Algeria only (population qualifies even though area doesn't).

**Thought process**
- It's an "or" condition on two independent columns — resist the urge to write `AND`.
- Only project the three requested columns, not `SELECT *`.

**Oracle Solution**
```sql
SELECT name, population, area
FROM World
WHERE area >= 3000000
   OR population >= 25000000;
```

---

## 4. Article Views I
**Topic:** Self-comparison + `DISTINCT`

**Table:** `Views(article_id NUMBER, author_id NUMBER, viewer_id NUMBER, view_date DATE)`

**Task:** Find every id of a person who viewed at least one of their own articles (`author_id = viewer_id`), sorted ascending, no duplicates.

**Example**

| article_id | author_id | viewer_id |
|---|---|---|
| 1 | 3 | 5 |
| 2 | 7 | 7 |
| 3 | 7 | 6 |
| 4 | 7 | 7 |

Expected output: `7` (only once, even though it appears twice).

**Thought process**
- Same person can self-view many articles, so `DISTINCT` is essential or you'll get duplicate ids.
- The alias in the output is conventionally `id`, not `author_id`.

**Oracle Solution**
```sql
SELECT DISTINCT author_id AS id
FROM Views
WHERE author_id = viewer_id
ORDER BY id;
```

---

## 5. Invalid Tweets
**Topic:** String length filter

**Table:** `Tweets(tweet_id NUMBER, content VARCHAR2(200))`

**Task:** Return the `tweet_id` of tweets whose content is longer than 15 characters.

**Example**

| tweet_id | content |
|---|---|
| 1 | "Let us Code" |
| 2 | "More than fifteen chars are here!" |

Expected output: `2`.

**Thought process**
- Straightforward — just wrap the column in `LENGTH()` and compare.
- In Oracle, watch out for trailing spaces if content was padded with `CHAR` instead of `VARCHAR2` (not an issue here since it's `VARCHAR2`).

**Oracle Solution**
```sql
SELECT tweet_id
FROM Tweets
WHERE LENGTH(content) > 15;
```

---

## 6. Replace Employee ID With The Unique Identifier
**Topic:** `LEFT JOIN` to preserve unmatched rows

**Tables:** `Employees(id NUMBER, name VARCHAR2(50))`, `EmployeeUNI(id NUMBER, unique_id NUMBER)`

**Task:** For every employee, show their `unique_id` (or `NULL` if they don't have one) alongside their name.

**Example**

Employees: `(1, 'Alice')`, `(2, 'Bob')`
EmployeeUNI: `(1, 100)`

Expected output: `(100, 'Alice')`, `(NULL, 'Bob')`.

**Thought process**
- "Every employee" is the key phrase — that means the `Employees` table drives the result, and unmatched employees must still appear.
- That's the textbook signal for a `LEFT JOIN` from `Employees` to `EmployeeUNI`, not an `INNER JOIN`.

**Oracle Solution**
```sql
SELECT eu.unique_id, e.name
FROM Employees e
LEFT JOIN EmployeeUNI eu ON e.id = eu.id;
```

---

## 7. Product Sales Analysis I
**Topic:** Basic `INNER JOIN`

**Tables:** `Sales(sale_id NUMBER, product_id NUMBER, year NUMBER, quantity NUMBER, price NUMBER)`, `Product(product_id NUMBER, product_name VARCHAR2(50))`

**Task:** Report `product_name`, `year`, and `price` for every sale.

**Thought process**
- Every sale references exactly one product, so an inner join on `product_id` is sufficient — there's no need to worry about unmatched rows here.

**Oracle Solution**
```sql
SELECT p.product_name, s.year, s.price
FROM Sales s
JOIN Product p ON s.product_id = p.product_id;
```

---

## 8. Customer Who Visited but Did Not Make Any Transactions
**Topic:** `LEFT JOIN` + "anti-join" pattern

**Tables:** `Visits(visit_id NUMBER, customer_id NUMBER)`, `Transactions(transaction_id NUMBER, visit_id NUMBER, amount NUMBER)`

**Task:** For each customer, count how many of their visits resulted in **zero** transactions.

**Thought process**
- "Visited but made no transaction" is a classic anti-join: `LEFT JOIN Visits → Transactions`, then keep only rows where the transaction side is `NULL`.
- After filtering, group by customer and count.

**Oracle Solution**
```sql
SELECT v.customer_id AS customer_id, COUNT(*) AS count_no_trans
FROM Visits v
LEFT JOIN Transactions t ON v.visit_id = t.visit_id
WHERE t.transaction_id IS NULL
GROUP BY v.customer_id;
```

---

## 9. Rising Temperature
**Topic:** Self-join on a date offset

**Table:** `Weather(id NUMBER, recorddate DATE, temperature NUMBER)`

**Task:** Return the `id`s of days where the temperature was strictly higher than the temperature recorded the day before (calendar day, not row order).

**Thought process**
- Because "the day before" is a date relationship, not a row-order relationship, a `LAG()` on row order would be wrong if a day is missing from the data.
- Safer approach: self-join the table to itself, matching `today.recorddate = yesterday.recorddate + 1`, then compare temperatures.

**Oracle Solution**
```sql
SELECT w2.id
FROM Weather w1
JOIN Weather w2 ON w2.recorddate = w1.recorddate + 1
WHERE w2.temperature > w1.temperature;
```

---

## 10. Average Time of Process per Machine
**Topic:** Pivoting rows with `CASE` + aggregation

**Table:** `Activity(machine_id NUMBER, process_id NUMBER, activity_type VARCHAR2(10), timestamp_val NUMBER)` — `activity_type` is `'start'` or `'end'`, `timestamp_val` is elapsed seconds.

**Task:** For each machine, compute the average processing time (end − start) across all its processes, rounded to 3 decimals.

**Thought process**
- Each `(machine_id, process_id)` pair has exactly two rows: a start and an end. You need those two rows side by side to subtract them.
- `MAX(CASE WHEN activity_type = 'start' THEN timestamp_val END)` pulls the start value into one column per group; do the same for `'end'`.
- Group by `(machine_id, process_id)` first to get one duration per process, then average those durations per machine in an outer query.

**Oracle Solution**
```sql
SELECT machine_id, ROUND(AVG(duration), 3) AS processing_time
FROM (
    SELECT machine_id, process_id,
           MAX(CASE WHEN activity_type = 'end'   THEN timestamp_val END)
         - MAX(CASE WHEN activity_type = 'start' THEN timestamp_val END) AS duration
    FROM Activity
    GROUP BY machine_id, process_id
)
GROUP BY machine_id;
```

---

## 11. Employee Bonus
**Topic:** `LEFT JOIN` + `NULL`-inclusive filter

**Tables:** `Employee(empId NUMBER, name VARCHAR2(50), supervisor NUMBER, salary NUMBER)`, `Bonus(empId NUMBER, bonus NUMBER)`

**Task:** Show the name and bonus of every employee whose bonus is less than 1000, **including** employees with no bonus row at all.

**Thought process**
- "Including employees with no row" is the same `NULL` trap as problem #2 — you need `LEFT JOIN` (not inner join) plus `OR bonus IS NULL`.

**Oracle Solution**
```sql
SELECT e.name, b.bonus
FROM Employee e
LEFT JOIN Bonus b ON e.empId = b.empId
WHERE b.bonus < 1000 OR b.bonus IS NULL;
```

---

## 12. Students and Examinations
**Topic:** `CROSS JOIN` to build a full grid, then `LEFT JOIN` to fill it in

**Tables:** `Students(student_id NUMBER, student_name VARCHAR2(50))`, `Subjects(subject_name VARCHAR2(50))`, `Examinations(student_id NUMBER, subject_name VARCHAR2(50))` — one row per exam attended.

**Task:** For every student and every subject (even combinations never attended), report how many times that student took that subject's exam.

**Thought process**
- You need a row for *every possible* student × subject combination, even when the count is zero — that rules out simply grouping the `Examinations` table, because missing combinations wouldn't appear at all.
- Build the full grid first with `CROSS JOIN (Students, Subjects)`, then `LEFT JOIN` the actual `Examinations` log onto it and count non-null matches.

**Oracle Solution**
```sql
SELECT s.student_id, s.student_name, sub.subject_name,
       COUNT(e.subject_name) AS attended_exams
FROM Students s
CROSS JOIN Subjects sub
LEFT JOIN Examinations e
       ON e.student_id = s.student_id
      AND e.subject_name = sub.subject_name
GROUP BY s.student_id, s.student_name, sub.subject_name
ORDER BY s.student_id, sub.subject_name;
```

---

## 13. Managers with at Least 5 Direct Reports
**Topic:** Self-referencing `GROUP BY … HAVING`

**Table:** `Employee(id NUMBER, name VARCHAR2(50), department VARCHAR2(50), managerId NUMBER)`

**Task:** Return the names of employees who manage 5 or more people.

**Thought process**
- "Manages 5+ people" means: count how often each `id` shows up as *someone else's* `managerId`.
- Group the table by `managerId`, filter groups with `HAVING COUNT(*) >= 5`, then look up those ids' names.

**Oracle Solution**
```sql
SELECT name
FROM Employee
WHERE id IN (
    SELECT managerId
    FROM Employee
    WHERE managerId IS NOT NULL
    GROUP BY managerId
    HAVING COUNT(*) >= 5
);
```

---

## 14. Confirmation Rate
**Topic:** Conditional aggregation with safe division

**Tables:** `Signups(user_id NUMBER, time_stamp DATE)`, `Confirmations(user_id NUMBER, time_stamp DATE, action VARCHAR2(10))` — action is `'confirmed'` or `'timeout'`.

**Task:** For every signed-up user, compute confirmed-requests ÷ total-requests, rounded to 2 decimals; users with no confirmation requests get 0.

**Thought process**
- Start from `Signups` (every signed-up user must appear) and `LEFT JOIN` `Confirmations`.
- Count confirmed vs. total with `CASE`/`COUNT`, and guard the division with `NULLIF` so a user with zero requests doesn't throw a divide-by-zero error — instead it returns `NULL`, which `NVL` then turns into 0.

**Oracle Solution**
```sql
SELECT s.user_id,
       NVL(
         ROUND(
           SUM(CASE WHEN c.action = 'confirmed' THEN 1 ELSE 0 END) * 1.0
           / NULLIF(COUNT(c.action), 0)
         , 2)
       , 0) AS confirmation_rate
FROM Signups s
LEFT JOIN Confirmations c ON s.user_id = c.user_id
GROUP BY s.user_id;
```

---

## 15. Not Boring Movies
**Topic:** Filter + sort

**Table:** `Cinema(id NUMBER, movie VARCHAR2(50), description VARCHAR2(50), rating NUMBER)`

**Task:** Return all columns for movies with an odd `id` and a description that is not `'boring'`, ordered by rating descending.

**Thought process**
- "Odd id" → `MOD(id, 2) = 1`.
- Two independent filters combined with `AND`, then a simple `ORDER BY`.

**Oracle Solution**
```sql
SELECT *
FROM Cinema
WHERE MOD(id, 2) = 1
  AND description <> 'boring'
ORDER BY rating DESC;
```

---

## 16. Average Selling Price
**Topic:** Date-range join + weighted average

**Tables:** `Prices(product_id NUMBER, start_date DATE, end_date DATE, price NUMBER)`, `UnitsSold(product_id NUMBER, purchase_date DATE, units NUMBER)`

**Task:** For each product, compute `SUM(price × units) / SUM(units)`, rounded to 2 decimals; products with no sales get 0.

**Thought process**
- This isn't a simple average of the `price` column — it's a *weighted* average, weighted by units sold. That's why you sum the products of price and units, then divide by total units, rather than just `AVG(price)`.
- The join condition isn't on equal keys alone — a sale only matches the price row whose date range contains its `purchase_date`.
- `LEFT JOIN` keeps products with zero sales in the result so you can default them to 0 with `NVL`.

**Oracle Solution**
```sql
SELECT p.product_id,
       NVL(ROUND(SUM(p.price * u.units) / NULLIF(SUM(u.units), 0), 2), 0) AS average_price
FROM Prices p
LEFT JOIN UnitsSold u
       ON p.product_id = u.product_id
      AND u.purchase_date BETWEEN p.start_date AND p.end_date
GROUP BY p.product_id;
```

---

## 17. Project Employees I
**Topic:** Join + `AVG`

**Tables:** `Project(project_id NUMBER, employee_id NUMBER)`, `Employee(employee_id NUMBER, name VARCHAR2(50), experience_years NUMBER)`

**Task:** For each project, report the average experience of its employees, rounded to 2 decimals.

**Oracle Solution**
```sql
SELECT pr.project_id, ROUND(AVG(e.experience_years), 2) AS average_years
FROM Project pr
JOIN Employee e ON pr.employee_id = e.employee_id
GROUP BY pr.project_id;
```

---

## 18. Percentage of Users Attended a Contest
**Topic:** Ratio against a global total (scalar subquery)

**Tables:** `Users(user_id NUMBER, user_name VARCHAR2(50))`, `Register(contest_id NUMBER, user_id NUMBER)`

**Task:** For each contest, compute the percentage of all users who registered, rounded to 2 decimals, sorted by percentage descending then contest_id ascending.

**Thought process**
- The denominator (total user count) is the *same number* for every contest — that's a strong hint to compute it once as a scalar subquery, rather than joining the full `Users` table in.
- Sorting needs two keys: percentage desc as primary, contest_id asc as the tiebreaker.

**Oracle Solution**
```sql
SELECT r.contest_id,
       ROUND(COUNT(DISTINCT r.user_id) * 100.0 / (SELECT COUNT(*) FROM Users), 2) AS percentage
FROM Register r
GROUP BY r.contest_id
ORDER BY percentage DESC, r.contest_id ASC;
```

---

## 19. Queries Quality and Percentage
**Topic:** Two independent conditional aggregates in one group

**Table:** `Queries(query_name VARCHAR2(50), result VARCHAR2(50), position NUMBER, rating NUMBER)`

**Task:** Per `query_name`, compute:
- `quality` = average of `rating / position`, rounded to 2 decimals
- `poor_query_percentage` = percent of rows with `rating < 3`, rounded to 2 decimals

**Thought process**
- These are two unrelated metrics computed over the *same* grouped rows — you can compute both in a single `GROUP BY query_name` pass rather than two separate queries.

**Oracle Solution**
```sql
SELECT query_name,
       ROUND(AVG(rating * 1.0 / position), 2) AS quality,
       ROUND(SUM(CASE WHEN rating < 3 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS poor_query_percentage
FROM Queries
GROUP BY query_name;
```

---

## 20. Monthly Transactions I
**Topic:** Grouping by a derived date part + multiple conditional sums

**Table:** `Transactions(id NUMBER, country VARCHAR2(10), state VARCHAR2(10), amount NUMBER, trans_date DATE)` — `state` is `'approved'` or `'declined'`.

**Task:** Per month and country, report: total transaction count, approved count, total amount, and approved amount.

**Thought process**
- "Per month" means you first need to collapse `trans_date` down to a `YYYY-MM` bucket with `TO_CHAR`, then group by that derived value together with `country`.
- Four separate conditional aggregates can all be computed in the same `GROUP BY` pass.

**Oracle Solution**
```sql
SELECT TO_CHAR(trans_date, 'YYYY-MM') AS month,
       country,
       COUNT(*) AS trans_count,
       SUM(CASE WHEN state = 'approved' THEN 1 ELSE 0 END) AS approved_count,
       SUM(amount) AS trans_total_amount,
       SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END) AS approved_total_amount
FROM Transactions
GROUP BY TO_CHAR(trans_date, 'YYYY-MM'), country;
```

---

## 21. Immediate Food Delivery II
**Topic:** "First row per group" + ratio

**Table:** `Delivery(delivery_id NUMBER, customer_id NUMBER, order_date DATE, customer_pref_delivery_date DATE)`

**Task:** Among each customer's *first* order only, what percentage were immediate (order_date = preferred delivery date)? Round to 2 decimals.

**Thought process**
- Two steps hiding in one problem: (1) identify each customer's first order, (2) compute a percentage over just that subset.
- Use a correlated/derived subquery to find `MIN(order_date)` per customer, filter the main table down to just those rows, then aggregate.

**Oracle Solution**
```sql
SELECT ROUND(
         SUM(CASE WHEN order_date = customer_pref_delivery_date THEN 1 ELSE 0 END) * 100.0
         / COUNT(*)
       , 2) AS immediate_percentage
FROM Delivery
WHERE (customer_id, order_date) IN (
    SELECT customer_id, MIN(order_date)
    FROM Delivery
    GROUP BY customer_id
);
```

---

## 22. Game Play Analysis IV
**Topic:** "Day after first event" retention pattern

**Table:** `Activity(player_id NUMBER, device_id NUMBER, event_date DATE, games_played NUMBER)`

**Task:** Of all players, what fraction logged in again exactly one day after their very first login? Round to 2 decimals.

**Thought process**
- Classic "Day-1 retention" question: first compute each player's install date (`MIN(event_date)`), then check whether a row exists for `install_date + 1`.
- The denominator is the *total number of players*, not the number who came back — easy place to make an off-by-logic mistake.

**Oracle Solution**
```sql
WITH first_login AS (
    SELECT player_id, MIN(event_date) AS install_date
    FROM Activity
    GROUP BY player_id
)
SELECT ROUND(
         COUNT(DISTINCT a.player_id) * 1.0 / (SELECT COUNT(*) FROM first_login)
       , 2) AS fraction
FROM first_login f
JOIN Activity a
     ON a.player_id = f.player_id
    AND a.event_date = f.install_date + 1;
```

---

## 23. Number of Unique Subjects Taught by Each Teacher
**Topic:** `COUNT(DISTINCT …)`

**Table:** `Teacher(teacher_id NUMBER, subject_id NUMBER, dept_id NUMBER)` — a teacher can teach the same subject in multiple departments, producing duplicate `(teacher_id, subject_id)` pairs.

**Task:** For each teacher, count the number of *distinct* subjects they teach.

**Thought process**
- Because the same subject can repeat across departments, a plain `COUNT(subject_id)` would overcount — you need `COUNT(DISTINCT subject_id)`.

**Oracle Solution**
```sql
SELECT teacher_id, COUNT(DISTINCT subject_id) AS cnt
FROM Teacher
GROUP BY teacher_id;
```

---

## 24. User Activity for the Past 30 Days I
**Topic:** Fixed date-window filter + daily grouping

**Table:** `Activity(user_id NUMBER, session_id NUMBER, activity_date DATE, activity_type VARCHAR2(20))`

**Task:** For the 30-day window ending 2019-07-27 (inclusive), report the number of distinct active users per day.

**Thought process**
- "30 days ending on X" means the window is `[X - 29, X]`, i.e. `> X - 30 AND <= X`.
- Group by the raw date — no need to bucket by month here, each day is its own group.

**Oracle Solution**
```sql
SELECT activity_date AS day, COUNT(DISTINCT user_id) AS active_users
FROM Activity
WHERE activity_date > DATE '2019-07-27' - 30
  AND activity_date <= DATE '2019-07-27'
GROUP BY activity_date;
```

---

## 25. Product Sales Analysis III
**Topic:** "First occurrence per group" via correlated subquery

**Table:** `Sales(sale_id NUMBER, product_id NUMBER, year NUMBER, quantity NUMBER, price NUMBER)`

**Task:** For every product, return only the row(s) from its very first year of sale.

**Thought process**
- This is the "first row per group" pattern again: correlate the outer row's `product_id` against the minimum year for that same product.

**Oracle Solution**
```sql
SELECT product_id, year AS first_year, quantity, price
FROM Sales s
WHERE year = (
    SELECT MIN(year) FROM Sales s2 WHERE s2.product_id = s.product_id
);
```

---

## 26. Classes More Than 5 Students
**Topic:** `GROUP BY … HAVING` with distinct counting

**Table:** `Courses(student VARCHAR2(50), class VARCHAR2(50))`

**Task:** Return classes with 5 or more distinct students enrolled.

**Thought process**
- A student could theoretically appear twice for the same class in messy data, so count `DISTINCT student` to be safe rather than just `COUNT(*)`.

**Oracle Solution**
```sql
SELECT class
FROM Courses
GROUP BY class
HAVING COUNT(DISTINCT student) >= 5;
```

---

## 27. Find Followers Count
**Topic:** Simple `GROUP BY` + sort

**Table:** `Followers(user_id NUMBER, follower_id NUMBER)`

**Task:** Count followers per user, ordered by `user_id`.

**Oracle Solution**
```sql
SELECT user_id, COUNT(follower_id) AS followers_count
FROM Followers
GROUP BY user_id
ORDER BY user_id;
```

---

## 28. Biggest Single Number
**Topic:** `HAVING COUNT = 1` + `MAX` over a filtered set

**Table:** `MyNumbers(num NUMBER)`

**Task:** Return the largest value that appears **exactly once**; if no such value exists, return `NULL`.

**Thought process**
- First isolate the numbers that occur exactly once with `GROUP BY num HAVING COUNT(*) = 1`, then take the max of that filtered set.
- `MAX()` over an empty result set naturally returns `NULL` in Oracle, which conveniently matches the "no such value" requirement without extra logic.

**Oracle Solution**
```sql
SELECT MAX(num) AS num
FROM (
    SELECT num
    FROM MyNumbers
    GROUP BY num
    HAVING COUNT(*) = 1
);
```

---

## 29. Customers Who Bought All Products
**Topic:** "Contains all elements of a set" pattern

**Tables:** `Customer(customer_id NUMBER, product_key NUMBER)`, `Product(product_key NUMBER)`

**Task:** Find customers whose set of purchased products covers the *entire* product catalog.

**Thought process**
- This is a classic "division" query: a customer qualifies only if the count of distinct products they bought equals the total count of products that exist.
- Group by customer, count distinct product_key, and compare against a scalar subquery of the catalog size.

**Oracle Solution**
```sql
SELECT customer_id
FROM Customer
GROUP BY customer_id
HAVING COUNT(DISTINCT product_key) = (SELECT COUNT(*) FROM Product);
```

---

## 30. The Number of Employees Which Report to Each Employee
**Topic:** Self-join + aggregation, rounding an average to an integer

**Table:** `Employees(employee_id NUMBER, name VARCHAR2(50), reports_to NUMBER, age NUMBER)`

**Task:** For every employee who has at least one direct report, show how many people report to them and the average age of those reports (rounded to the nearest whole number).

**Thought process**
- Self-join the table to itself: one copy plays "the manager," the other plays "the reports," linked by `reports_to = manager.employee_id`.
- Grouping by the manager's id/name naturally excludes employees with zero reports, since an inner self-join drops them.

**Oracle Solution**
```sql
SELECT e.employee_id, e.name,
       COUNT(r.employee_id) AS reports_count,
       ROUND(AVG(r.age)) AS average_age
FROM Employees e
JOIN Employees r ON r.reports_to = e.employee_id
GROUP BY e.employee_id, e.name
ORDER BY e.employee_id;
```

---

## 31. Primary Department for Each Employee
**Topic:** `UNION` to merge two disjoint cases

**Table:** `Employee(employee_id NUMBER, department_id NUMBER, primary_flag CHAR(1))` — `'Y'`/`'N'`, and an employee with only one department row may have `primary_flag = 'N'`.

**Task:** For each employee, return exactly one department: the one flagged primary if they belong to multiple, or their only department if they belong to just one.

**Thought process**
- Two separate cases that never overlap: (a) rows explicitly flagged `'Y'`, and (b) employees who only ever have one department row (regardless of flag).
- `UNION` cleanly combines both result sets into one, and naturally de-duplicates.

**Oracle Solution**
```sql
SELECT employee_id, department_id
FROM Employee
WHERE primary_flag = 'Y'
UNION
SELECT employee_id, department_id
FROM Employee
WHERE employee_id IN (
    SELECT employee_id FROM Employee GROUP BY employee_id HAVING COUNT(*) = 1
);
```

---

## 32. Triangle Judgement
**Topic:** Row-wise `CASE` logic

**Table:** `Triangle(x NUMBER, y NUMBER, z NUMBER)`

**Task:** For each row, output `'Yes'` if the three side lengths can form a valid triangle, else `'No'`.

**Thought process**
- The triangle inequality requires *all three* pairwise sums to exceed the third side — checking only one pair isn't sufficient.

**Oracle Solution**
```sql
SELECT x, y, z,
       CASE WHEN x + y > z AND x + z > y AND y + z > x THEN 'Yes' ELSE 'No' END AS triangle
FROM Triangle;
```

---

## 33. Consecutive Numbers
**Topic:** `LEAD()` window function for "look ahead" comparisons

**Table:** `Logs(id NUMBER, num NUMBER)`

**Task:** Find every number that appears at least 3 times in a row (consecutively, ordered by `id`).

**Thought process**
- "Consecutive" means you need to compare each row to the *next two* rows in id order — a self-join would work but gets messy; `LEAD()` is cleaner.
- Pull the next value and the value-after-next into the same row with two `LEAD()` calls, then check if all three match.
- Wrap in `DISTINCT` since the same qualifying number could span more than one 3-in-a-row streak.

**Oracle Solution**
```sql
SELECT DISTINCT num AS ConsecutiveNums
FROM (
    SELECT num,
           LEAD(num, 1) OVER (ORDER BY id) AS next1,
           LEAD(num, 2) OVER (ORDER BY id) AS next2
    FROM Logs
)
WHERE num = next1 AND num = next2;
```

---

## 34. Product Price at a Given Date
**Topic:** Correlated subquery to find "latest row as of a date"

**Table:** `Products(product_id NUMBER, new_price NUMBER, change_date DATE)` — every price change is logged as a new row.

**Task:** For each product, find its price as of 2019-08-16 (a product not yet changed by then defaults to 10).

**Thought process**
- "Price as of a date" = the most recent `change_date` that is still `<=` that date. Correlated subquery: for each product, find `MAX(change_date)` among rows on or before the target date, then join back to get that row's price.
- Start from the distinct list of all product ids (not just ones with a qualifying price change) so products with no changes yet still appear, defaulting via `NVL`.

**Oracle Solution**
```sql
SELECT p.product_id, NVL(t.new_price, 10) AS price
FROM (SELECT DISTINCT product_id FROM Products) p
LEFT JOIN (
    SELECT pr.product_id, pr.new_price
    FROM Products pr
    WHERE pr.change_date = (
        SELECT MAX(pr2.change_date)
        FROM Products pr2
        WHERE pr2.product_id = pr.product_id
          AND pr2.change_date <= DATE '2019-08-16'
    )
) t ON p.product_id = t.product_id;
```

---

## 35. Last Person to Fit in the Bus
**Topic:** Running total with a window function

**Table:** `Queue(person_id NUMBER, person_name VARCHAR2(50), weight NUMBER, turn NUMBER)`

**Task:** People board in `turn` order. Find the name of the *last* person who can board without the cumulative weight exceeding 1000 kg.

**Thought process**
- "Cumulative weight so far" is a running total — exactly what `SUM(...) OVER (ORDER BY turn)` computes.
- After computing the running total for every person, keep only the ones still under the 1000 kg limit, and take the one with the largest running total (i.e., boarded last among the valid ones).

**Oracle Solution**
```sql
SELECT person_name
FROM (
    SELECT person_name,
           SUM(weight) OVER (ORDER BY turn) AS running_weight
    FROM Queue
)
WHERE running_weight <= 1000
ORDER BY running_weight DESC
FETCH FIRST 1 ROW ONLY;
```

---

## 36. Count Salary Categories
**Topic:** `UNION ALL` to force fixed output categories (including zero counts)

**Table:** `Accounts(account_id NUMBER, income NUMBER)`

**Task:** Report how many accounts fall into three fixed income bands: Low (< 20000), Average (20000–50000 inclusive), High (> 50000) — even bands with zero matching accounts should appear.

**Thought process**
- A single `GROUP BY` on a derived `CASE` label would silently *omit* a category if no rows matched it — but the problem wants all three labels to always appear.
- `UNION ALL` of three independent `COUNT(*) ... WHERE` queries guarantees all three rows exist regardless of data.

**Oracle Solution**
```sql
SELECT 'Low Salary' AS category, COUNT(*) AS accounts_count FROM Accounts WHERE income < 20000
UNION ALL
SELECT 'Average Salary', COUNT(*) FROM Accounts WHERE income BETWEEN 20000 AND 50000
UNION ALL
SELECT 'High Salary', COUNT(*) FROM Accounts WHERE income > 50000;
```

---

## 37. Employees Whose Manager Left the Company
**Topic:** `NOT IN` anti-pattern (careful with `NULL`)

**Table:** `Employees(employee_id NUMBER, name VARCHAR2(50), manager_id NUMBER, salary NUMBER)`

**Task:** Find employees earning under 30000 whose manager_id points to an id that no longer exists in the table, ordered by employee_id.

**Thought process**
- The manager "left" means their id is referenced in `manager_id` but doesn't exist as an `employee_id` anymore — a `NOT IN` subquery against all employee ids.
- Guard explicitly with `manager_id IS NOT NULL`, since `NOT IN` against a subquery behaves unpredictably if the subquery list itself could contain `NULL`s, and rows with no manager at all shouldn't qualify anyway.

**Oracle Solution**
```sql
SELECT employee_id
FROM Employees
WHERE salary < 30000
  AND manager_id IS NOT NULL
  AND manager_id NOT IN (SELECT employee_id FROM Employees)
ORDER BY employee_id;
```

---

## 38. Exchange Seats
**Topic:** Row-wise `CASE` with an edge case for odd counts

**Table:** `Seat(id NUMBER, student VARCHAR2(50))` — ids are consecutive integers starting at 1.

**Task:** Swap the student names between every pair of adjacent seats (1↔2, 3↔4, …). If the total number of seats is odd, the last seat keeps its own student.

**Thought process**
- Odd ids move to `id+1`, even ids move to `id-1` — except the very last id, if the total count is odd, which stays put.
- Compute the swapped id with `CASE`, but special-case "odd id AND id equals the max id" first.

**Oracle Solution**
```sql
SELECT
  CASE
    WHEN MOD(id, 2) = 1 AND id = (SELECT MAX(id) FROM Seat) THEN id
    WHEN MOD(id, 2) = 1 THEN id + 1
    ELSE id - 1
  END AS id,
  student
FROM Seat
ORDER BY id;
```

---

## 39. Movie Rating
**Topic:** Two unrelated sub-answers combined with `UNION ALL`, tie-breaking with `ORDER BY` + row limiting

**Tables:** `Movies(movie_id NUMBER, title VARCHAR2(50))`, `Users(user_id NUMBER, name VARCHAR2(50))`, `MovieRating(movie_id NUMBER, user_id NUMBER, rating NUMBER, created_at DATE)`

**Task:** Return two results stacked into one column: (1) the name of the user who rated the most movies (ties broken by lexicographically smallest name), and (2) the title of the movie with the highest average rating in February 2020 (ties broken the same way).

**Thought process**
- These are two independent "top-1 with a tiebreak" queries; solve each on its own, then stack them with `UNION ALL` (not plain `UNION`, since we want to keep both rows even if the text happens to match).
- Each sub-query needs to sort by its primary metric descending, then by name/title ascending as the tiebreaker, then take just the top row.

**Oracle Solution**
```sql
SELECT results FROM (
    SELECT u.name AS results
    FROM MovieRating mr
    JOIN Users u ON mr.user_id = u.user_id
    GROUP BY u.name
    ORDER BY COUNT(*) DESC, u.name ASC
) WHERE ROWNUM = 1
UNION ALL
SELECT results FROM (
    SELECT m.title AS results
    FROM MovieRating mr
    JOIN Movies m ON mr.movie_id = m.movie_id
    WHERE mr.created_at >= DATE '2020-02-01' AND mr.created_at < DATE '2020-03-01'
    GROUP BY m.title
    ORDER BY AVG(mr.rating) DESC, m.title ASC
) WHERE ROWNUM = 1;
```

---

## 40. Restaurant Growth
**Topic:** Rolling window sum over a date range (self-join style)

**Table:** `Customer(customer_id NUMBER, name VARCHAR2(50), visited_on DATE, amount NUMBER)` — multiple customers can visit the same date; combine same-date amounts first.

**Task:** Starting from the 7th distinct date onward, report for each `visited_on` date the total and average amount spent over the trailing 7-day window (that date and the 6 before it).

**Thought process**
- "Trailing 7-day window" is a rolling aggregate — for each anchor date, sum amounts from every row whose date falls within `[anchor - 6, anchor]`.
- A self-join on a `BETWEEN` range does this cleanly: join the distinct dates to the raw transactions within range, then group by the anchor date.
- The "starting from the 7th day" requirement just filters out anchor dates that don't yet have a full 7-day history behind them.

**Oracle Solution**
```sql
SELECT c1.visited_on,
       SUM(c2.amount) AS amount,
       ROUND(SUM(c2.amount) / 7, 2) AS average_amount
FROM (SELECT DISTINCT visited_on FROM Customer) c1
JOIN Customer c2 ON c2.visited_on BETWEEN c1.visited_on - 6 AND c1.visited_on
WHERE c1.visited_on >= (SELECT MIN(visited_on) FROM Customer) + 6
GROUP BY c1.visited_on
ORDER BY c1.visited_on;
```

---

## 41. Friend Requests II: Who Has the Most Friends
**Topic:** `UNION ALL` to merge two "sides" of a relationship before counting

**Table:** `RequestAccepted(requester_id NUMBER, accepter_id NUMBER, accept_date DATE)`

**Task:** Find the person with the most friends and how many friends they have (friendship is symmetric — being the requester or the accepter both count).

**Thought process**
- Each accepted request creates a friendship for *two* people, not one — so you need to count each id whether it shows up as `requester_id` or `accepter_id`.
- Stack both columns into a single id column with `UNION ALL` (not `UNION`, since we want to keep every occurrence for counting), then group and count.

**Oracle Solution**
```sql
SELECT id, COUNT(*) AS num
FROM (
    SELECT requester_id AS id FROM RequestAccepted
    UNION ALL
    SELECT accepter_id AS id FROM RequestAccepted
)
GROUP BY id
ORDER BY num DESC
FETCH FIRST 1 ROW ONLY;
```

---

## 42. Investments in 2016
**Topic:** Two independent `HAVING`-filtered subqueries combined with `IN`

**Table:** `Insurance(pid NUMBER, tiv_2015 NUMBER, tiv_2016 NUMBER, lat NUMBER, lon NUMBER)`

**Task:** Sum `tiv_2016` for policyholders who (a) had the same `tiv_2015` value as at least one other policyholder, **and** (b) have a unique `(lat, lon)` location that no other policyholder shares.

**Thought process**
- Two unrelated grouping conditions on the *same* table: one groups by `tiv_2015` looking for duplicates (`COUNT > 1`), the other groups by `(lat, lon)` looking for uniqueness (`COUNT = 1`).
- Filter the main table against both conditions at once (both must hold), then sum.

**Oracle Solution**
```sql
SELECT ROUND(SUM(tiv_2016), 2) AS tiv_2016
FROM Insurance i
WHERE tiv_2015 IN (
    SELECT tiv_2015 FROM Insurance GROUP BY tiv_2015 HAVING COUNT(*) > 1
)
AND (lat, lon) IN (
    SELECT lat, lon FROM Insurance GROUP BY lat, lon HAVING COUNT(*) = 1
);
```

---

## 43. Department Top Three Salaries
**Topic:** `DENSE_RANK()` for "top N per group, ties count once"

**Tables:** `Employee(id NUMBER, name VARCHAR2(50), salary NUMBER, departmentId NUMBER)`, `Department(id NUMBER, name VARCHAR2(50))`

**Task:** For each department, list every employee whose salary is among the top 3 *distinct* salary values in that department (ties share a rank, and don't push out a 4th distinct value).

**Thought process**
- "Top 3 distinct salaries, ties included" is exactly what `DENSE_RANK()` gives you (unlike `RANK()`, which would skip numbers after a tie, or `ROW_NUMBER()`, which would arbitrarily cut ties).
- Partition the ranking by department, order by salary descending, then keep rank <= 3.

**Oracle Solution**
```sql
SELECT d.name AS department, e.name AS employee, e.salary
FROM (
    SELECT id, name, salary, departmentId,
           DENSE_RANK() OVER (PARTITION BY departmentId ORDER BY salary DESC) AS rnk
    FROM Employee
) e
JOIN Department d ON e.departmentId = d.id
WHERE e.rnk <= 3;
```

---

## 44. Fix Names in a Table
**Topic:** String manipulation — case normalization

**Table:** `Users(user_id NUMBER, name VARCHAR2(50))` — names are stored with inconsistent casing.

**Task:** Capitalize only the first letter of each name and lowercase the rest, ordered by `user_id`.

**Thought process**
- Split the string into "first character" and "everything else," transform each piece with `UPPER`/`LOWER` independently, then concatenate.

**Oracle Solution**
```sql
SELECT user_id,
       UPPER(SUBSTR(name, 1, 1)) || LOWER(SUBSTR(name, 2)) AS name
FROM Users
ORDER BY user_id;
```

---

## 45. Patients With a Condition
**Topic:** Pattern matching for a "whole word" prefix inside a space-separated list

**Table:** `Patients(patient_id NUMBER, patient_name VARCHAR2(50), conditions VARCHAR2(200))` — `conditions` holds space-separated codes, e.g. `"DIAB1 HYPER2"`.

**Task:** Find patients who have a condition code starting with `DIAB1`, as a whole code (not just any substring match — `"HDIAB100"` shouldn't count).

**Thought process**
- A plain `LIKE '%DIAB1%'` would incorrectly match a code that merely *contains* `DIAB1` in the middle of another word.
- Because it must be a standalone code, it either sits at the very start of the string (`'DIAB1%'`) or appears right after a space (`'% DIAB1%'`) — cover both cases with `OR`.

**Oracle Solution**
```sql
SELECT *
FROM Patients
WHERE conditions LIKE 'DIAB1%'
   OR conditions LIKE '% DIAB1%';
```

---

## 46. Delete Duplicate Emails
**Topic:** `DELETE` with a correlated `EXISTS` subquery

**Table:** `Person(id NUMBER, email VARCHAR2(50))`

**Task:** Delete duplicate email rows, keeping only the one with the smallest `id` for each email.

**Thought process**
- This is a mutation, not a `SELECT` — think in terms of "which rows should be removed," not "which rows should remain."
- A row should be deleted if some *other* row with the same email has a smaller id — that's naturally expressed as a correlated `EXISTS`.

**Oracle Solution**
```sql
DELETE FROM Person p
WHERE EXISTS (
    SELECT 1 FROM Person p2
    WHERE p2.email = p.email
      AND p2.id < p.id
);
```

---

## 47. Second Highest Salary
**Topic:** Scalar subquery to exclude the maximum

**Table:** `Employee(id NUMBER, salary NUMBER)`

**Task:** Return the second-highest distinct salary, or `NULL` if it doesn't exist.

**Thought process**
- "Second highest" = the highest salary *strictly less than* the maximum. Filtering out the max first, then taking `MAX()` of what's left, handles duplicates of the top salary correctly (unlike an `OFFSET`-based approach, which could return a duplicate of the max).
- If there's only one distinct salary, the filtered subquery is empty, and `MAX()` over no rows returns `NULL` automatically — exactly the required fallback.

**Oracle Solution**
```sql
SELECT MAX(salary) AS SecondHighestSalary
FROM Employee
WHERE salary < (SELECT MAX(salary) FROM Employee);
```

---

## 48. Group Sold Products By The Date
**Topic:** `LISTAGG` to concatenate grouped values into a string

**Table:** `Activities(sell_date DATE, product VARCHAR2(50))`

**Task:** Per `sell_date`, report the count of distinct products sold and a comma-separated, alphabetically sorted list of those distinct product names.

**Thought process**
- De-duplicate `(sell_date, product)` pairs *before* aggregating, otherwise a product sold twice on the same day would appear twice in the concatenated list.
- `LISTAGG(... ) WITHIN GROUP (ORDER BY ...)` is Oracle's tool for turning grouped rows into one delimited string, with the ordering controlling how the pieces appear inside that string.

**Oracle Solution**
```sql
SELECT sell_date,
       COUNT(*) AS num_sold,
       LISTAGG(product, ',') WITHIN GROUP (ORDER BY product) AS products
FROM (SELECT DISTINCT sell_date, product FROM Activities)
GROUP BY sell_date
ORDER BY sell_date;
```

---

## 49. List the Products Ordered in a Period
**Topic:** Join + date-range filter + `HAVING` on a sum

**Tables:** `Products(product_id NUMBER, product_name VARCHAR2(50), product_category VARCHAR2(50))`, `Orders(product_id NUMBER, order_date DATE, unit NUMBER)`

**Task:** Return the name and total units ordered for products with at least 100 total units ordered during February 2020.

**Thought process**
- Filter to the date range first (in the `WHERE` clause, since it's a row-level filter, not a group-level one), then aggregate per product, then filter the aggregated totals with `HAVING`.

**Oracle Solution**
```sql
SELECT p.product_name, SUM(o.unit) AS unit
FROM Orders o
JOIN Products p ON o.product_id = p.product_id
WHERE o.order_date >= DATE '2020-02-01' AND o.order_date < DATE '2020-03-01'
GROUP BY p.product_name
HAVING SUM(o.unit) >= 100;
```

---

## 50. Find Users With Valid E-Mails
**Topic:** `REGEXP_LIKE` for format validation

**Table:** `Users(user_id NUMBER, name VARCHAR2(50), mail VARCHAR2(100))`

**Task:** Return users whose email: starts with a letter, is followed by any number of letters/digits/underscore/period/dash, and ends with exactly `@leetcode.com`.

**Thought process**
- Break the requirement into regex pieces: `^[[:alpha:]]` (must start with a letter), `[[:alnum:]_.-]*` (allowed characters before the domain), then a literal, anchored `@leetcode\.com$` (the dot must be escaped, since unescaped `.` matches *any* character in regex).
- `REGEXP_LIKE` is Oracle's regex predicate — it slots into `WHERE` just like `LIKE` does.

**Oracle Solution**
```sql
SELECT *
FROM Users
WHERE REGEXP_LIKE(mail, '^[[:alpha:]][[:alnum:]_.-]*@leetcode\.com$');
```

---

## How to use this guide
1. Read the **Task** and **Example** for a problem, and try writing the query yourself before looking at the **Thought process**.
2. If you get stuck, read only the thought process (not the solution) and try again.
3. Once you've got a working query, compare it against the Oracle solution here for style/technique, then go solve the *actual* problem on LeetCode to validate against their test cases and see the real schema.
4. If your LeetCode account is set to MySQL by default, you can usually switch the language to **Oracle** (or **PL/SQL**) in the code editor's language dropdown before submitting, so these patterns translate directly.