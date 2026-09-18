# Oracle SQL Subqueries — Client Case Study Practice Set

You already know the basics of subqueries. This guide is built to sharpen your **reasoning** — each question is framed as a real request from a stakeholder at a retail/e-commerce client, and solving it requires you to decide *which kind* of subquery fits (correlated vs non-correlated, scalar vs multi-row, `WHERE` vs `FROM` vs `HAVING` vs `SELECT`), often combined with joins, `GROUP BY`, aggregates, and `CASE`. Window functions are intentionally excluded — some "ranking-style" answers below are solved with correlated subqueries instead, which is itself a useful skill.

All 17 questions run against **one shared schema** (a small retail company), so relationships carry over between questions the way they would in a real database. A couple of questions add a few extra rows partway through — that's noted explicitly when it happens.

---

## Section 0 — Client Database Schema

**The client:** a mid-size retail company selling Electronics, Furniture, and Grocery items online.

### Create Tables

```sql
CREATE TABLE departments (
    department_id   NUMBER(4)      PRIMARY KEY,
    department_name VARCHAR2(30)   NOT NULL,
    location        VARCHAR2(30)
);

CREATE TABLE employees (
    employee_id     NUMBER(6)      PRIMARY KEY,
    employee_name   VARCHAR2(50)   NOT NULL,
    department_id   NUMBER(4)      REFERENCES departments(department_id),
    salary          NUMBER(10,2),
    hire_date       DATE,
    manager_id      NUMBER(6)      REFERENCES employees(employee_id),
    job_title       VARCHAR2(30)
);

CREATE TABLE customers (
    customer_id     NUMBER(6)      PRIMARY KEY,
    customer_name   VARCHAR2(50)   NOT NULL,
    city            VARCHAR2(30),
    country         VARCHAR2(30),
    customer_since  DATE
);

CREATE TABLE suppliers (
    supplier_id     NUMBER(6)      PRIMARY KEY,
    supplier_name   VARCHAR2(50)   NOT NULL,
    country         VARCHAR2(30)
);

CREATE TABLE products (
    product_id      NUMBER(6)      PRIMARY KEY,
    product_name    VARCHAR2(50)   NOT NULL,
    category        VARCHAR2(30),
    unit_price      NUMBER(10,2),
    supplier_id     NUMBER(6)      REFERENCES suppliers(supplier_id)
);

CREATE TABLE orders (
    order_id        NUMBER(6)      PRIMARY KEY,
    customer_id     NUMBER(6)      REFERENCES customers(customer_id),
    order_date      DATE,
    employee_id     NUMBER(6)      REFERENCES employees(employee_id),
    status          VARCHAR2(20)
);

CREATE TABLE order_items (
    order_item_id   NUMBER(6)      PRIMARY KEY,
    order_id        NUMBER(6)      REFERENCES orders(order_id),
    product_id      NUMBER(6)      REFERENCES products(product_id),
    quantity        NUMBER(5),
    unit_price      NUMBER(10,2)
);

CREATE TABLE payments (
    payment_id      NUMBER(6)      PRIMARY KEY,
    order_id        NUMBER(6)      REFERENCES orders(order_id),
    payment_date    DATE,
    amount          NUMBER(10,2),
    payment_method  VARCHAR2(20)
);

CREATE TABLE reviews (
    review_id       NUMBER(6)      PRIMARY KEY,
    product_id      NUMBER(6)      REFERENCES products(product_id),
    customer_id     NUMBER(6)      REFERENCES customers(customer_id),
    rating          NUMBER(1),
    review_date     DATE
);

CREATE TABLE returns (
    return_id       NUMBER(6)      PRIMARY KEY,
    order_item_id   NUMBER(6)      REFERENCES order_items(order_item_id),
    return_date     DATE,
    reason          VARCHAR2(100)
);
```

### Insert Sample Data

```sql
-- DEPARTMENTS
INSERT INTO departments VALUES (10,'Sales','Chicago');
INSERT INTO departments VALUES (20,'IT','Dallas');
INSERT INTO departments VALUES (30,'HR','Chicago');
INSERT INTO departments VALUES (40,'Finance','New York');

-- EMPLOYEES
INSERT INTO employees VALUES (101,'Alice Monroe',10,65000,DATE'2019-03-15',NULL,'Sales Manager');
INSERT INTO employees VALUES (102,'Ben Carter',10,48000,DATE'2020-06-01',101,'Sales Rep');
INSERT INTO employees VALUES (103,'Clara Diaz',10,52000,DATE'2021-01-10',101,'Sales Rep');
INSERT INTO employees VALUES (104,'David Lin',20,90000,DATE'2018-11-20',NULL,'IT Manager');
INSERT INTO employees VALUES (105,'Ella Fischer',20,72000,DATE'2020-02-14',104,'Developer');
INSERT INTO employees VALUES (106,'Frank Ito',20,68000,DATE'2021-07-19',104,'Developer');
INSERT INTO employees VALUES (107,'Grace Kim',30,55000,DATE'2019-09-05',NULL,'HR Manager');
INSERT INTO employees VALUES (108,'Henry Ross',30,42000,DATE'2022-04-11',107,'HR Associate');
INSERT INTO employees VALUES (109,'Isla Novak',40,85000,DATE'2017-05-23',NULL,'Finance Manager');
INSERT INTO employees VALUES (110,'Jack Owens',40,60000,DATE'2020-10-02',109,'Financial Analyst');

-- CUSTOMERS
INSERT INTO customers VALUES (201,'Nathan Brooks','Chicago','USA',DATE'2019-01-10');
INSERT INTO customers VALUES (202,'Olivia Chen','Dallas','USA',DATE'2020-03-22');
INSERT INTO customers VALUES (203,'Priya Sharma','Mumbai','India',DATE'2018-07-14');
INSERT INTO customers VALUES (204,'Quinn Walsh','London','UK',DATE'2021-05-30');
INSERT INTO customers VALUES (205,'Rosa Martinez','Madrid','Spain',DATE'2019-11-02');
INSERT INTO customers VALUES (206,'Sam Okafor','Lagos','Nigeria',DATE'2022-02-18');
INSERT INTO customers VALUES (207,'Tara Lindgren','Chicago','USA',DATE'2020-08-09');
INSERT INTO customers VALUES (208,'Uma Patel','Mumbai','India',DATE'2022-09-01');

-- SUPPLIERS
INSERT INTO suppliers VALUES (301,'TechSource Ltd','USA');
INSERT INTO suppliers VALUES (302,'Global Gadgets','China');
INSERT INTO suppliers VALUES (303,'HomeEssentials Co','Germany');
INSERT INTO suppliers VALUES (304,'FreshGoods Inc','USA');

-- PRODUCTS
INSERT INTO products VALUES (401,'Laptop Pro 15','Electronics',1200,301);
INSERT INTO products VALUES (402,'Wireless Mouse','Electronics',25,301);
INSERT INTO products VALUES (403,'4K Monitor','Electronics',350,302);
INSERT INTO products VALUES (404,'Bluetooth Speaker','Electronics',80,302);
INSERT INTO products VALUES (405,'Office Chair','Furniture',150,303);
INSERT INTO products VALUES (406,'Standing Desk','Furniture',400,303);
INSERT INTO products VALUES (407,'Desk Lamp','Furniture',30,303);
INSERT INTO products VALUES (408,'Organic Coffee Beans 1kg','Grocery',18,304);
INSERT INTO products VALUES (409,'Green Tea Pack','Grocery',12,304);
INSERT INTO products VALUES (410,'Noise Cancelling Headphones','Electronics',250,301);

-- ORDERS  (order 5013 has a NULL customer_id — a guest-checkout data anomaly, used deliberately later)
INSERT INTO orders VALUES (5001,201,DATE'2024-01-05',102,'Completed');
INSERT INTO orders VALUES (5002,202,DATE'2024-01-18',103,'Completed');
INSERT INTO orders VALUES (5003,203,DATE'2024-02-02',102,'Completed');
INSERT INTO orders VALUES (5004,201,DATE'2024-02-20',103,'Completed');
INSERT INTO orders VALUES (5005,204,DATE'2024-03-11',102,'Completed');
INSERT INTO orders VALUES (5006,205,DATE'2024-03-25',103,'Cancelled');
INSERT INTO orders VALUES (5007,206,DATE'2024-04-07',102,'Completed');
INSERT INTO orders VALUES (5008,202,DATE'2024-04-19',103,'Completed');
INSERT INTO orders VALUES (5009,207,DATE'2024-05-03',102,'Completed');
INSERT INTO orders VALUES (5010,201,DATE'2024-05-14',103,'Completed');
INSERT INTO orders VALUES (5011,203,DATE'2024-06-01',102,'Completed');
INSERT INTO orders VALUES (5012,205,DATE'2024-06-21',103,'Completed');
INSERT INTO orders VALUES (5013,NULL,DATE'2024-06-25',102,'Pending');

-- ORDER_ITEMS
INSERT INTO order_items VALUES (6001,5001,401,1,1200);
INSERT INTO order_items VALUES (6002,5001,402,2,25);
INSERT INTO order_items VALUES (6003,5002,403,1,350);
INSERT INTO order_items VALUES (6004,5003,404,1,80);
INSERT INTO order_items VALUES (6005,5003,408,3,18);
INSERT INTO order_items VALUES (6006,5004,405,2,150);
INSERT INTO order_items VALUES (6007,5005,406,1,400);
INSERT INTO order_items VALUES (6008,5006,407,1,30);
INSERT INTO order_items VALUES (6009,5007,409,5,12);
INSERT INTO order_items VALUES (6010,5008,410,1,250);
INSERT INTO order_items VALUES (6011,5009,401,1,1200);
INSERT INTO order_items VALUES (6012,5010,403,2,350);
INSERT INTO order_items VALUES (6013,5011,402,4,25);
INSERT INTO order_items VALUES (6014,5012,406,1,400);
INSERT INTO order_items VALUES (6015,5013,402,1,25);

-- PAYMENTS  (no payment for the Cancelled or Pending orders)
INSERT INTO payments VALUES (7001,5001,DATE'2024-01-06',1250,'Credit Card');
INSERT INTO payments VALUES (7002,5002,DATE'2024-01-19',350,'PayPal');
INSERT INTO payments VALUES (7003,5003,DATE'2024-02-03',134,'Credit Card');
INSERT INTO payments VALUES (7004,5004,DATE'2024-02-21',300,'Debit Card');
INSERT INTO payments VALUES (7005,5005,DATE'2024-03-12',400,'Credit Card');
INSERT INTO payments VALUES (7006,5007,DATE'2024-04-08',60,'PayPal');
INSERT INTO payments VALUES (7007,5008,DATE'2024-04-20',250,'Credit Card');
INSERT INTO payments VALUES (7008,5009,DATE'2024-05-04',1200,'Credit Card');
INSERT INTO payments VALUES (7009,5010,DATE'2024-05-15',700,'Debit Card');
INSERT INTO payments VALUES (7010,5011,DATE'2024-06-02',100,'PayPal');
INSERT INTO payments VALUES (7011,5012,DATE'2024-06-22',400,'Credit Card');

-- REVIEWS
INSERT INTO reviews VALUES (8001,401,201,5,DATE'2024-01-10');
INSERT INTO reviews VALUES (8002,402,201,4,DATE'2024-01-10');
INSERT INTO reviews VALUES (8003,403,202,4,DATE'2024-01-25');
INSERT INTO reviews VALUES (8004,404,203,3,DATE'2024-02-05');
INSERT INTO reviews VALUES (8005,408,203,5,DATE'2024-02-06');
INSERT INTO reviews VALUES (8006,405,201,2,DATE'2024-02-25');
INSERT INTO reviews VALUES (8007,406,204,4,DATE'2024-03-15');
INSERT INTO reviews VALUES (8008,409,206,5,DATE'2024-04-10');
INSERT INTO reviews VALUES (8009,410,202,5,DATE'2024-04-22');
INSERT INTO reviews VALUES (8010,401,207,4,DATE'2024-05-06');
INSERT INTO reviews VALUES (8011,403,201,3,DATE'2024-05-18');

-- RETURNS
INSERT INTO returns VALUES (9001,6006,DATE'2024-02-25','Defective - wobbly base');
INSERT INTO returns VALUES (9002,6010,DATE'2024-04-25','Changed mind');

COMMIT;
```

---

## Part 1 — Case Study Questions

---

### Question 1 — "Which customers have we never actually sold anything to?"
**Concept focus:** `NOT EXISTS` vs the `NOT IN` NULL trap (correlated subquery)

**Client scenario:** Marketing wants a list of registered customers who have **zero orders**, so they can be added to a "welcome back" email campaign. A junior analyst wrote a query using `NOT IN` and got back an empty result set — even though everyone agrees at least one such customer exists. You're asked to find out why, and produce the correct list.

**Thought process:**
1. First reproduce the junior analyst's logic: `customer_id NOT IN (SELECT customer_id FROM orders)`.
2. Recall that `NOT IN` evaluates as a series of `<> ALL` comparisons. If the subquery's result set contains even one `NULL`, every comparison becomes `UNKNOWN`, and the whole `WHERE` clause silently returns zero rows for every candidate row.
3. Check the `orders` table for a `NULL` in `customer_id` — order `5013` has one (a guest-checkout anomaly). That explains the empty result.
4. Switch to a **correlated `NOT EXISTS`** subquery instead: for each customer, check whether a matching order row exists. This approach is immune to `NULL`s in the subquery because it never does a value-vs-list comparison — it just asks "does a row exist?"

**Answer:**
```sql
-- Step 1: reproduce the bug (returns 0 rows because of the NULL in orders.customer_id)
SELECT customer_name
FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM orders);

-- Step 2: correct version using a correlated NOT EXISTS
SELECT c.customer_id, c.customer_name, c.city
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```
**Result:** Uma Patel (208) — the only customer with no orders at all.

---

### Question 2 — "Which employees are outperforming their own department's average pay?"
**Concept focus:** Correlated scalar subquery in `WHERE`, self-referencing aggregate

**Client scenario:** HR is reviewing pay equity and wants to flag every employee earning **more than the average salary within their own department** (not the company-wide average — departments have very different pay bands).

**Thought process:**
1. "Their own department's average" means the comparison value changes per row → this must be a **correlated** subquery, not a single fixed number.
2. For each employee row, the inner query needs to filter `employees` by the *same* `department_id` as the outer row, then average the salaries.
3. Compare the outer employee's salary to that inner scalar result with `>`.

**Answer:**
```sql
SELECT e.employee_name, e.department_id, e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id
)
ORDER BY e.department_id;
```
**Result:** Alice Monroe, David Lin, Grace Kim, Isla Novak — interestingly, exactly the four department managers.

---

### Question 3 — "Who are our proven big spenders?"
**Concept focus:** `EXISTS` with a correlated join-based subquery

**Client scenario:** The loyalty program team wants to identify customers who have placed **at least one order worth more than $500** — these customers qualify for a "VIP" tag regardless of their overall order history.

**Thought process:**
1. This is an existence check, not a count — the customer either has *at least one* qualifying order or doesn't. `EXISTS` is the natural fit (and stops scanning as soon as one match is found, unlike a `COUNT`-based approach).
2. The subquery must correlate on `customer_id` and needs to reach into `payments` (via `orders`) to check the order amount.
3. Write the correlated subquery, joining `orders` to `payments` inside it, filtered on `amount > 500`.

**Answer:**
```sql
SELECT c.customer_id, c.customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    JOIN payments p ON p.order_id = o.order_id
    WHERE o.customer_id = c.customer_id
    AND p.amount > 500
);
```
**Result:** Nathan Brooks and Tara Lindgren.

---

### Question 4 — "Flag each order as above or below our typical order value."
**Concept focus:** Non-correlated scalar subquery inside `SELECT`, combined with `CASE`

**Client scenario:** The finance team wants a report listing every paid order alongside the company's overall average order value, with a plain-English flag ("Above Average" / "Average" / "Below Average") next to each one — useful for spotting unusually large or small transactions at a glance.

**Thought process:**
1. The "overall average order value" is a **single fixed number** — it doesn't change per row, so this is a non-correlated scalar subquery.
2. It's needed twice: once to display it, once inside `CASE` to compare against. A scalar subquery can be placed directly in the `SELECT` list, and reused inside `CASE WHEN` conditions.
3. Join `orders` to `payments` to get the amount per order, then wrap the comparison in `CASE`.

**Answer:**
```sql
SELECT o.order_id,
       p.amount AS order_total,
       (SELECT ROUND(AVG(amount), 2) FROM payments) AS company_avg_order,
       CASE
           WHEN p.amount > (SELECT AVG(amount) FROM payments) THEN 'Above Average'
           WHEN p.amount = (SELECT AVG(amount) FROM payments) THEN 'Average'
           ELSE 'Below Average'
       END AS performance_flag
FROM orders o
JOIN payments p ON p.order_id = o.order_id
ORDER BY o.order_id;
```
**Result:** Orders 5001 (1250), 5009 (1200), and 5010 (700) come back "Above Average" (company average ≈ 467.6); the rest are "Below Average."

---

### Question 5 — "Is our flagship laptop overpriced compared to everything in Furniture?"
**Concept focus:** `ALL` vs `ANY` operators with a multi-row subquery

**Client scenario:** A pricing analyst is benchmarking categories against each other ahead of a promotion. They want two things: (1) which products are priced higher than **every single** Furniture item, and (2) which products are priced higher than **at least one** Furniture item. They keep confusing the two.

**Thought process:**
1. "Higher than every item in Furniture" means the product's price must beat the **maximum** furniture price — that's exactly what `> ALL (...)` means: greater than every value the subquery returns.
2. "Higher than at least one item in Furniture" only requires beating the **minimum** furniture price — that's what `> ANY (...)` means: greater than at least one value returned.
3. Write both queries side by side so the difference is visible in the result sets.

**Answer:**
```sql
-- Higher than ALL Furniture products (must beat the max furniture price)
SELECT product_name, category, unit_price
FROM products
WHERE unit_price > ALL (
    SELECT unit_price FROM products WHERE category = 'Furniture'
);

-- Higher than ANY Furniture product (only needs to beat the cheapest one)
SELECT product_name, category, unit_price
FROM products
WHERE unit_price > ANY (
    SELECT unit_price FROM products WHERE category = 'Furniture'
);
```
**Result:** `> ALL` returns only **Laptop Pro 15** (1200 beats Furniture's max of 400). `> ANY` returns everything priced above Furniture's cheapest item (30) — i.e., everything except Green Tea Pack (12), Organic Coffee Beans (18), and Desk Lamp (30 itself, not strictly greater).

---

### Question 6 — "Which product category is actually driving our revenue?"
**Concept focus:** Subquery in `FROM` (inline view) combined with `HAVING` and a nested aggregate subquery

**Client scenario:** The category manager wants a single row identifying the top revenue-generating category, based only on completed orders — not just a sorted list, but a query that programmatically isolates the winner.

**Thought process:**
1. Revenue isn't a stored column — it has to be derived as `quantity * unit_price`, summed per category. That aggregation needs a `GROUP BY`, which means building an inline view (subquery in the `FROM` clause) that computes category totals first.
2. To isolate just the *top* category, compare each group's total to the **maximum** of all the group totals — which itself requires re-aggregating the same grouped data, i.e., a subquery over a subquery.
3. Use `HAVING` for the group-level filter since the comparison involves an aggregate (`SUM`).

**Answer:**
```sql
SELECT p.category, SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN orders o ON o.order_id = oi.order_id
WHERE o.status = 'Completed'
GROUP BY p.category
HAVING SUM(oi.quantity * oi.unit_price) = (
    SELECT MAX(cat_total)
    FROM (
        SELECT SUM(oi2.quantity * oi2.unit_price) AS cat_total
        FROM order_items oi2
        JOIN products p2 ON p2.product_id = oi2.product_id
        JOIN orders o2 ON o2.order_id = oi2.order_id
        WHERE o2.status = 'Completed'
        GROUP BY p2.category
    )
);
```
**Result:** Electronics, at 3,930 in completed-order revenue — well ahead of Furniture (≈1,100) and Grocery (≈114).

---

### Question 7 — "Which departments are running over the company's typical budget?"
**Concept focus:** `HAVING` with a subquery built from a grouped inline view

**Client scenario:** The CFO wants to know which departments have a **total salary cost above the company's average departmental salary cost** — i.e., compare each department's total payroll to the average of all four departments' totals, not to individual salaries.

**Thought process:**
1. "Average departmental salary cost" means: first sum salaries *per department*, then average those four sums. That's a two-stage aggregation — a job for a subquery in `FROM` (or, as done here, directly inside a scalar subquery used by `HAVING`).
2. The outer query groups employees by department and sums their salaries; `HAVING` then filters those group sums against the derived average.

**Answer:**
```sql
SELECT d.department_name, SUM(e.salary) AS dept_total_salary
FROM employees e
JOIN departments d ON d.department_id = e.department_id
GROUP BY d.department_name
HAVING SUM(e.salary) > (
    SELECT AVG(dept_sum)
    FROM (
        SELECT SUM(e2.salary) AS dept_sum
        FROM employees e2
        GROUP BY e2.department_id
    )
);
```
**Result:** Sales (165,000) and IT (230,000) exceed the average departmental total (159,250); HR and Finance fall below it.

---

### Question 8 — "Is this product priced above or below the norm for its own category?"
**Concept focus:** Correlated scalar subquery combined with `CASE` (category-relative classification)

**Client scenario:** The catalog team wants every product labeled relative to **its own category's average price** — not the store-wide average — so a $400 desk and a $400 laptop aren't judged by the same yardstick.

**Thought process:**
1. Like Question 2, "its own category's average" changes per row, so the subquery must correlate on `category`.
2. The comparison then needs three branches (`>`, `=`, `<`), which is a natural fit for `CASE`.
3. Since the same correlated subquery is needed three times (once for display, twice inside `CASE`), it's worth double-checking all three copies filter on the same correlated column.

**Answer:**
```sql
SELECT p.product_name,
       p.category,
       p.unit_price,
       (SELECT ROUND(AVG(p2.unit_price), 2) FROM products p2 WHERE p2.category = p.category) AS category_avg_price,
       CASE
           WHEN p.unit_price > (SELECT AVG(p3.unit_price) FROM products p3 WHERE p3.category = p.category) THEN 'Above Average'
           WHEN p.unit_price = (SELECT AVG(p3.unit_price) FROM products p3 WHERE p3.category = p.category) THEN 'Average'
           ELSE 'Below Average'
       END AS price_position
FROM products p
ORDER BY p.category, p.unit_price DESC;
```
**Result:** Within Electronics (avg ≈381), only Laptop Pro 15 is "Above Average." Within Furniture (avg ≈193), Standing Desk is "Above Average." Within Grocery (avg =15), Coffee Beans is "Above Average." Everything else in each category is "Below Average."

---

### Question 9 — "Who is our single most frequent customer?"
**Concept focus:** Nested subquery (subquery of a subquery) to isolate a maximum count

**Client scenario:** Customer success wants to identify the customer(s) with the **highest number of orders placed** — to personally reach out and thank them.

**Thought process:**
1. First need order counts *per customer* — an aggregation, so build an inline view grouping `orders` by `customer_id`. (Exclude the `NULL` customer from order 5013, since it isn't a real customer.)
2. Then need the single highest count among those — a `MAX` over the grouped results, which requires wrapping that same grouped query inside another subquery.
3. Finally, match customers whose count equals that maximum. Using `=` (not `ORDER BY ... FETCH FIRST`) correctly handles ties automatically, since it's a filter condition rather than a row-limit.

**Answer:**
```sql
SELECT c.customer_name, oc.order_count
FROM customers c
JOIN (
    SELECT customer_id, COUNT(*) AS order_count
    FROM orders
    WHERE customer_id IS NOT NULL
    GROUP BY customer_id
) oc ON oc.customer_id = c.customer_id
WHERE oc.order_count = (
    SELECT MAX(cnt)
    FROM (
        SELECT COUNT(*) AS cnt
        FROM orders
        WHERE customer_id IS NOT NULL
        GROUP BY customer_id
    )
);
```
**Result:** Nathan Brooks, with 3 orders — more than anyone else.

---

### Question 10 — "Which product is our second-most expensive?"
**Concept focus:** Correlated subquery used as a ranking substitute (no window functions)

**Client scenario:** The pricing team wants the product that sits at **exactly the second-highest price point** across the entire catalog — useful for setting a "premium tier" threshold — but window functions like `RANK()` aren't allowed in this environment (legacy reporting tool limitation).

**Thought process:**
1. Without `RANK()`/`DENSE_RANK()`, "second highest" can be reframed as: *a product for which exactly one other distinct price is greater than its own*.
2. This is a classic **correlated counting subquery**: for each product `p1`, count how many *distinct* prices in the whole table are greater than `p1`'s price. If that count is exactly `1`, `p1` holds the second-highest price.
3. Using `COUNT(DISTINCT ...)` (rather than plain `COUNT`) matters if multiple products could tie at the top price — it keeps the logic based on distinct price *levels*, not row counts.

**Answer:**
```sql
SELECT product_name, unit_price
FROM products p1
WHERE 1 = (
    SELECT COUNT(DISTINCT p2.unit_price)
    FROM products p2
    WHERE p2.unit_price > p1.unit_price
);
```
**Result:** Standing Desk at 400 (only the Laptop Pro 15 at 1200 is priced higher).

---

### Question 11 — "Which customers bought the entire starter bundle?"
**Concept focus:** Relational division via double `NOT EXISTS`

**Client scenario:** Marketing is running a loyalty offer for anyone who has purchased **both** items in the "starter bundle" — the Laptop Pro 15 *and* the Wireless Mouse (in any orders, not necessarily the same one). This is a classic "find X that has ALL of Y" problem.

**Thought process:**
1. This is a relational division problem: for each customer, there must be **no** required product that they *haven't* purchased. Two negatives ("no missing product") is the standard `NOT EXISTS`-inside-`NOT EXISTS` pattern.
2. Outer query: candidate customers.
3. Middle subquery: the required product list (Laptop Pro 15, Wireless Mouse).
4. Inner subquery: for a given required product, check whether this customer has ever ordered it (across any of their orders/order_items).
5. If the middle subquery finds **any** required product with no matching inner purchase, that customer is excluded.

**Answer:**
```sql
SELECT c.customer_id, c.customer_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM products p
    WHERE p.product_name IN ('Laptop Pro 15', 'Wireless Mouse')
    AND NOT EXISTS (
        SELECT 1
        FROM orders o
        JOIN order_items oi ON oi.order_id = o.order_id
        WHERE o.customer_id = c.customer_id
        AND oi.product_id = p.product_id
    )
);
```
**Result:** Nathan Brooks only — his order 5001 contains both the Laptop Pro 15 and the Wireless Mouse.

---

### Question 12 — "Which sales rep is outperforming their department's typical order handling?"
**Concept focus:** `JOIN` + `GROUP BY` feeding a correlated subquery over an inline view

**Client scenario:** Sales leadership wants to know which rep is handling orders worth **more, on average, than their department's typical rep** — recognizing that department can matter (even though today only Sales reps process orders, the query should be written to generalize).

**Thought process:**
1. First compute each employee's total processed order value — requires joining `employees` → `orders` → `payments`, then grouping by employee. Build this as an inline view.
2. Then, for each employee in that inline view, compare their total to the **average total among employees in the same department** — another aggregation over the same shape of data, correlated on `department_id`.
3. Because the comparison value depends on the outer row's department, the inner subquery must reference the outer inline view's alias — a correlated subquery over a derived table, which is a step up in complexity from a correlated subquery over a base table.

**Answer:**
```sql
SELECT et.employee_name, et.total_order_value
FROM (
    SELECT e.employee_id, e.employee_name, e.department_id, SUM(p.amount) AS total_order_value
    FROM employees e
    JOIN orders o ON o.employee_id = e.employee_id
    JOIN payments p ON p.order_id = o.order_id
    GROUP BY e.employee_id, e.employee_name, e.department_id
) et
WHERE et.total_order_value > (
    SELECT AVG(det.total_order_value)
    FROM (
        SELECT e2.employee_id, SUM(p2.amount) AS total_order_value
        FROM employees e2
        JOIN orders o2 ON o2.employee_id = e2.employee_id
        JOIN payments p2 ON p2.order_id = o2.order_id
        WHERE e2.department_id = et.department_id
        GROUP BY e2.employee_id
    ) det
);
```
**Result:** Ben Carter (3,144 in processed order value) beats the Sales department's rep average (2,572); Clara Diaz (2,000) does not.

---

### Question 13 — "Which suppliers have a clean quality record?"
**Concept focus:** `NOT EXISTS` combined with a join and a condition on an aggregated attribute (rating)

**Client scenario:** Procurement wants to renew contracts only with suppliers whose products have **never received a rating below 3** — a single bad review anywhere in their product line disqualifies them for this round.

**Thought process:**
1. This is another existence-style question, but phrased as a negative: "no product from this supplier has ever scored below 3."
2. That's `NOT EXISTS`: for each supplier, check whether *any* row exists in `products` joined to `reviews` where the rating is below 3. If such a row exists, the supplier fails.
3. Products with no reviews at all don't cause a failure — they simply never produce a matching row in the inner query, which is the correct behavior here.

**Answer:**
```sql
SELECT s.supplier_name
FROM suppliers s
WHERE NOT EXISTS (
    SELECT 1
    FROM products p
    JOIN reviews r ON r.product_id = p.product_id
    WHERE p.supplier_id = s.supplier_id
    AND r.rating < 3
);
```
**Result:** TechSource Ltd, Global Gadgets, and FreshGoods Inc pass. HomeEssentials Co fails — its Office Chair received a rating of 2.

---

### Question 14 — "What share of total revenue does each customer represent?"
**Concept focus:** Non-correlated scalar subquery as a division denominator, with `LEFT JOIN` to keep zero-spend customers visible

**Client scenario:** The finance team wants a revenue concentration report: each customer's total spend and what percentage of **total company revenue** that represents — including customers who've spent nothing, shown as 0%.

**Thought process:**
1. "Total company revenue" is one fixed number, computed once — a non-correlated scalar subquery, usable directly inside a `SELECT`-level arithmetic expression.
2. Each customer's own total spend requires aggregating `payments` through `orders`, grouped by customer — another inline view.
3. Using an inner join to that inline view would silently drop customers with no orders (like Uma Patel from Question 1) — so this needs a `LEFT JOIN`, with `NVL` to turn missing spend into `0` rather than `NULL`.

**Answer:**
```sql
SELECT c.customer_name,
       NVL(ct.total_spent, 0) AS total_spent,
       ROUND(NVL(ct.total_spent, 0) / (SELECT SUM(amount) FROM payments) * 100, 2) AS pct_of_total_revenue
FROM customers c
LEFT JOIN (
    SELECT o.customer_id, SUM(p.amount) AS total_spent
    FROM orders o
    JOIN payments p ON p.order_id = o.order_id
    GROUP BY o.customer_id
) ct ON ct.customer_id = c.customer_id
ORDER BY pct_of_total_revenue DESC;
```
**Result:** Nathan Brooks leads at ≈43.7% of total revenue; Uma Patel correctly shows 0% rather than being dropped from the report.

---

### Question 15 — "Flag any other purchases that match the same product and price as a returned item."
**Concept focus:** Multi-column (paired-value) subquery with `IN`

> **Before running this query, add the following rows** (a new order came in after the earlier data was captured):
> ```sql
> INSERT INTO orders VALUES (5014, 206, DATE'2024-07-02', 103, 'Completed');
> INSERT INTO order_items VALUES (6016, 5014, 405, 1, 150);
> INSERT INTO payments VALUES (7012, 5014, DATE'2024-07-03', 150, 'Credit Card');
> COMMIT;
> ```

**Client scenario:** A quality team is investigating a defective batch. They know *which* order items were returned, and want to flag **every other order item that shares the exact same product and price point** as a returned item — those customers may be sitting on the same defect without having returned it yet.

**Thought process:**
1. The condition being matched is a *pair* of values together — `(product_id, unit_price)` — not each column independently. Matching them separately with two `IN` clauses could pair a product with the wrong price from a different returned item.
2. Oracle supports multi-column subqueries: `(col1, col2) IN (SELECT col1, col2 FROM ...)` compares the pair as a unit.
3. Build the list of returned `(product_id, unit_price)` pairs by joining `returns` to `order_items`, then use that pair-list as the multi-column subquery.

**Answer:**
```sql
SELECT oi.order_item_id, o.order_id, oi.product_id, oi.quantity, oi.unit_price
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
WHERE (oi.product_id, oi.unit_price) IN (
    SELECT oir.product_id, oir.unit_price
    FROM returns r
    JOIN order_items oir ON oir.order_item_id = r.order_item_id
);
```
**Result:** Three rows come back — the two originally-returned items themselves (6006, 6010), plus item **6016**: Sam Okafor's Office Chair, bought at the same $150 price point as Nathan's returned chair. That's the new match worth investigating.

---

### Question 16 — "Which manager's team is genuinely overpaid relative to the company?"
**Concept focus:** Self-join concept expressed through a correlated aggregate subquery, layered with a second non-correlated subquery

**Client scenario:** The CFO wants to find managers whose **direct reports' average salary** exceeds the **company-wide average salary** — a signal that a team may be over-resourced relative to the rest of the org (managers who have no direct reports, or those with underpaid teams, should be excluded).

**Thought process:**
1. `employees` is self-referencing via `manager_id`, so "this manager's team" means a correlated subquery filtering `employees` where `manager_id` equals the outer employee's `employee_id`.
2. `EXISTS` first filters out anyone who isn't actually a manager (no rows where `manager_id = m.employee_id`), avoiding a meaningless `NULL` average for individual contributors.
3. The company-wide average is a separate, non-correlated scalar subquery, computed once and compared against the correlated team average.

**Answer:**
```sql
SELECT m.employee_name AS manager_name,
       (SELECT ROUND(AVG(e.salary), 2) FROM employees e WHERE e.manager_id = m.employee_id) AS avg_team_salary
FROM employees m
WHERE EXISTS (SELECT 1 FROM employees e2 WHERE e2.manager_id = m.employee_id)
AND (SELECT AVG(e3.salary) FROM employees e3 WHERE e3.manager_id = m.employee_id)
    > (SELECT AVG(salary) FROM employees);
```
**Result:** David Lin only — his team (Ella Fischer, Frank Ito) averages 70,000, above the company-wide average of 63,700. Alice Monroe's, Grace Kim's, and Isla Novak's teams all average below it.

---

### Question 17 — "Which months outperformed our typical monthly sales?"
**Concept focus:** `TO_CHAR`-based grouping inside a `FROM` subquery, compared against a second aggregated subquery

**Client scenario:** Leadership wants a quick list of calendar months where **total revenue beat the average month** — useful for spotting seasonal spikes worth digging into further.

**Thought process:**
1. Months aren't a stored column — they have to be derived from `order_date` using `TO_CHAR(order_date, 'YYYY-MM')`, then used as the `GROUP BY` key inside an inline view that sums payment amounts per month.
2. "The average month" requires re-aggregating that same monthly breakdown — average of the per-month totals — which means nesting a second nearly-identical grouped subquery inside a scalar comparison, similar in spirit to Question 7 but grouped by a derived date expression instead of a foreign key.
3. Filter the outer monthly totals against that average using a plain `WHERE`, since the outer query is already working with pre-aggregated rows (not raw rows needing `HAVING`).

**Answer:**
```sql
SELECT m.sales_month, m.monthly_total
FROM (
    SELECT TO_CHAR(o.order_date, 'YYYY-MM') AS sales_month, SUM(p.amount) AS monthly_total
    FROM orders o
    JOIN payments p ON p.order_id = o.order_id
    GROUP BY TO_CHAR(o.order_date, 'YYYY-MM')
) m
WHERE m.monthly_total > (
    SELECT AVG(m2.monthly_total)
    FROM (
        SELECT TO_CHAR(o2.order_date, 'YYYY-MM') AS sales_month, SUM(p2.amount) AS monthly_total
        FROM orders o2
        JOIN payments p2 ON p2.order_id = o2.order_id
        GROUP BY TO_CHAR(o2.order_date, 'YYYY-MM')
    ) m2
)
ORDER BY m.sales_month;
```
**Result:** January (1,600) and May (1,900) both beat the average month (≈756 across Jan–Jul); every other month falls below it.

---

## Quick-Reference: Subquery Patterns Covered

| # | Pattern | Question |
|---|---------|----------|
| 1 | Correlated `NOT EXISTS` (and why `NOT IN` breaks with NULLs) | Q1 |
| 2 | Correlated scalar subquery in `WHERE` | Q2 |
| 3 | `EXISTS` with a joined correlated subquery | Q3 |
| 4 | Non-correlated scalar subquery in `SELECT` + `CASE` | Q4 |
| 5 | `ALL` and `ANY` operators | Q5 |
| 6 | Subquery in `FROM` (inline view) nested in `HAVING` | Q6 |
| 7 | `HAVING` with a subquery over grouped data | Q7 |
| 8 | Correlated scalar subquery + `CASE` (category-relative) | Q8 |
| 9 | Nested subquery (subquery of a subquery) for MAX | Q9 |
| 10 | Correlated counting subquery as a ranking substitute | Q10 |
| 11 | Relational division — double `NOT EXISTS` | Q11 |
| 12 | Correlated subquery over a derived table (inline view) | Q12 |
| 13 | `NOT EXISTS` with an aggregated/joined condition | Q13 |
| 14 | Scalar subquery as a division denominator + `LEFT JOIN` | Q14 |
| 15 | Multi-column (paired-value) `IN` subquery | Q15 |
| 16 | Self-referencing correlated aggregate subquery | Q16 |
| 17 | Derived-date grouping in `FROM`, compared via nested aggregate | Q17 |

Seventeen questions cover every major subquery shape you'll encounter in real Oracle SQL work — correlated and non-correlated, scalar and multi-row, single-column and multi-column, and subqueries in `WHERE`, `FROM`, `HAVING`, and `SELECT`. That's a solid foundation; further practice from here is mostly about combining these patterns in new business contexts rather than learning new mechanics.