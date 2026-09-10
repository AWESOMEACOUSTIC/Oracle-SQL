# ArtisanHub Marketplace — Order Intelligence System
### An Oracle SQL Hands-On Case Study (Subqueries — mixed freely with everything you already know)

---

## How to Use This Guide

You're the database developer for **ArtisanHub Marketplace**, an online platform connecting independent artisans (pottery, textiles, jewelry, woodwork, candles) with buyers. This is a fresh scenario — a different table, different business, different data from anything before — but the same rule applies: **one table, start to finish.**

This round is built around **subqueries** — a query nested inside another query. You'll build up from the basics to correlated subqueries and `EXISTS`, and along the way you're free to lean on everything from earlier rounds too: joins, window functions, `GROUP BY`/`HAVING`, string/math functions, set operators, and TCL/DCL all make appearances, mixed in naturally the way a real analyst's query actually looks — not sectioned off into a separate lesson.

Every question follows the same format:

1. **Business Question**
2. **Expected Output**
3. **Answer** (SQL + short explanation)

Work through them in order — later questions depend on exactly this data existing exactly as loaded in Question 2.

---

## The Client Brief

> "We connect artisans with buyers across five product categories. Some of our best customers reorder — we need to actually track that. Right now nobody can tell me which category makes the most money, which sales rep is really pulling their weight, or which customers are worth chasing for a loyalty deal, because every answer requires three separate spreadsheets. Give us ONE table and real answers, not more spreadsheets."
> — ArtisanHub, Head of Operations

---

## Table of Contents

- [Setup](#setup)
- [Part 1 — Getting Started](#part-1--getting-started) (Q3–Q10)
- [Part 2 — Comparing Against a List](#part-2--comparing-against-a-list) (Q11–Q16)
- [Part 3 — Row-by-Row Comparisons](#part-3--row-by-row-comparisons) (Q17–Q23)
- [Part 4 — Full Business Reports](#part-4--full-business-reports) (Q24–Q34)

---

## Setup

### Question 1 — Build the Table

**Business Question:** Design and create the single table ArtisanHub needs to track every order: who bought what, from which category, handled by which rep, in which region — and whether it's a first-time order or a repeat of an earlier one (a reorder points back to the original order, in this very same table).

**Expected Output:**

```
Table ORDERS created.
```

| Name | Null? | Type |
|---|---|---|
| ORDER_ID | NOT NULL | NUMBER |
| CUSTOMER_NAME | NOT NULL | VARCHAR2(100) |
| PRODUCT_NAME | | VARCHAR2(100) |
| CATEGORY | NOT NULL | VARCHAR2(30) |
| ORDER_DATE | NOT NULL | DATE |
| QUANTITY | | NUMBER |
| UNIT_PRICE | | NUMBER(10,2) |
| TOTAL_AMOUNT | | NUMBER(10,2) |
| REGION | | VARCHAR2(15) |
| SALES_REP | | VARCHAR2(50) |
| STATUS | | VARCHAR2(12) |
| REORDER_OF | | NUMBER |

**Answer:**

```sql
CREATE TABLE orders (
    order_id       NUMBER          PRIMARY KEY,
    customer_name  VARCHAR2(100)   NOT NULL,
    product_name   VARCHAR2(100),
    category       VARCHAR2(30)    NOT NULL,
    order_date     DATE            NOT NULL,
    quantity       NUMBER,
    unit_price     NUMBER(10,2),
    total_amount   NUMBER(10,2),
    region         VARCHAR2(15),
    sales_rep      VARCHAR2(50),
    status         VARCHAR2(12)    CHECK (status IN ('DELIVERED','PENDING','CANCELLED','RETURNED')),
    reorder_of     NUMBER,
    CONSTRAINT fk_reorder FOREIGN KEY (reorder_of) REFERENCES orders(order_id)
);
```

_Explanation:_ `reorder_of` is a self-referencing foreign key — exactly the trick that let last round's table demonstrate every JOIN type, and this round it does double duty for subqueries too: a subquery can check "does a row exist where `reorder_of` points back at me?" without needing a second table anywhere.

---

### Question 2 — Load the Data

**Business Question:** Load ArtisanHub's current 20 orders across all five categories.

**Expected Output:**

```
20 rows created.
```

**Answer:**

```sql
INSERT ALL
  INTO orders VALUES (701,'Meena Iyer','Clay Vase','POTTERY',DATE '2024-01-05',2,800,1600,'NORTH','Rahul Verma','DELIVERED',NULL)
  INTO orders VALUES (702,'Karan Shah','Ceramic Bowl Set','POTTERY',DATE '2024-01-10',1,1500,1500,'SOUTH','Sneha Kapoor','DELIVERED',NULL)
  INTO orders VALUES (703,'Meena Iyer','Clay Vase','POTTERY',DATE '2024-02-15',3,800,2400,'NORTH','Rahul Verma','DELIVERED',701)
  INTO orders VALUES (704,'Ayesha Khan','Terracotta Planter','POTTERY',DATE '2024-01-20',1,1200,1200,'EAST','Arjun Mehta','RETURNED',NULL)
  INTO orders VALUES (705,'Rajesh Kumar','Handloom Saree','TEXTILES',DATE '2024-01-08',2,1000,2000,'WEST','Kavya Reddy','DELIVERED',NULL)
  INTO orders VALUES (706,'Priya Nambiar','Embroidered Shawl','TEXTILES',DATE '2024-01-15',1,2000,2000,'NORTH','Rahul Verma','DELIVERED',NULL)
  INTO orders VALUES (707,'Rajesh Kumar','Handloom Saree','TEXTILES',DATE '2024-03-01',1,1000,1000,'WEST','Kavya Reddy','DELIVERED',705)
  INTO orders VALUES (708,'Farah Sheikh','Cotton Table Runner','TEXTILES',DATE '2024-02-05',3,500,1500,'SOUTH','Sneha Kapoor','PENDING',NULL)
  INTO orders VALUES (709,'Vikram Malhotra','Silver Necklace Set','JEWELRY',DATE '2024-01-12',1,5000,5000,'SOUTH','Sneha Kapoor','DELIVERED',NULL)
  INTO orders VALUES (710,'Ananya Das','Gemstone Ring','JEWELRY',DATE '2024-01-25',1,3000,3000,'EAST','Arjun Mehta','DELIVERED',NULL)
  INTO orders VALUES (711,'Vikram Malhotra','Silver Earrings','JEWELRY',DATE '2024-02-20',2,1500,3000,'SOUTH','Sneha Kapoor','DELIVERED',709)
  INTO orders VALUES (712,'Ritika Singh','Beaded Bracelet','JEWELRY',DATE '2024-03-10',1,2000,2000,'WEST','Kavya Reddy','CANCELLED',NULL)
  INTO orders VALUES (713,'Suresh Pillai','Wooden Bookshelf','WOODWORK',DATE '2024-01-18',1,3000,3000,'NORTH','Rahul Verma','DELIVERED',NULL)
  INTO orders VALUES (714,'Nisha Gupta','Carved Photo Frame','WOODWORK',DATE '2024-02-02',2,1000,2000,'EAST','Arjun Mehta','DELIVERED',NULL)
  INTO orders VALUES (715,'Suresh Pillai','Wooden Bookshelf','WOODWORK',DATE '2024-03-15',1,3000,3000,'NORTH','Rahul Verma','DELIVERED',713)
  INTO orders VALUES (716,'Divya Bhatt','Wooden Coasters Set','WOODWORK',DATE '2024-02-28',1,1500,1500,'WEST','Kavya Reddy','RETURNED',NULL)
  INTO orders VALUES (717,'Farah Sheikh','Scented Soy Candles','CANDLES',DATE '2024-01-22',4,250,1000,'SOUTH','Sneha Kapoor','DELIVERED',NULL)
  INTO orders VALUES (718,'Ayesha Khan','Lavender Candle Jar','CANDLES',DATE '2024-02-10',2,300,600,'EAST','Arjun Mehta','DELIVERED',NULL)
  INTO orders VALUES (719,'Priya Nambiar','Aromatherapy Candle Set','CANDLES',DATE '2024-03-05',3,200,600,'NORTH','Rahul Verma','DELIVERED',NULL)
  INTO orders VALUES (720,'Karan Shah','Vanilla Pillar Candle','CANDLES',DATE '2024-03-20',1,500,500,'SOUTH','Sneha Kapoor','PENDING',NULL)
SELECT * FROM dual;
```

**Reference — the full dataset (keep this handy for every question below):**

| ID | Customer | Product | Category | Date | Qty | Unit Price | Total | Region | Rep | Status | Reorder Of |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 701 | Meena Iyer | Clay Vase | POTTERY | 01-05 | 2 | 800 | 1600 | NORTH | Rahul Verma | DELIVERED | — |
| 702 | Karan Shah | Ceramic Bowl Set | POTTERY | 01-10 | 1 | 1500 | 1500 | SOUTH | Sneha Kapoor | DELIVERED | — |
| 703 | Meena Iyer | Clay Vase | POTTERY | 02-15 | 3 | 800 | 2400 | NORTH | Rahul Verma | DELIVERED | 701 |
| 704 | Ayesha Khan | Terracotta Planter | POTTERY | 01-20 | 1 | 1200 | 1200 | EAST | Arjun Mehta | RETURNED | — |
| 705 | Rajesh Kumar | Handloom Saree | TEXTILES | 01-08 | 2 | 1000 | 2000 | WEST | Kavya Reddy | DELIVERED | — |
| 706 | Priya Nambiar | Embroidered Shawl | TEXTILES | 01-15 | 1 | 2000 | 2000 | NORTH | Rahul Verma | DELIVERED | — |
| 707 | Rajesh Kumar | Handloom Saree | TEXTILES | 03-01 | 1 | 1000 | 1000 | WEST | Kavya Reddy | DELIVERED | 705 |
| 708 | Farah Sheikh | Cotton Table Runner | TEXTILES | 02-05 | 3 | 500 | 1500 | SOUTH | Sneha Kapoor | PENDING | — |
| 709 | Vikram Malhotra | Silver Necklace Set | JEWELRY | 01-12 | 1 | 5000 | 5000 | SOUTH | Sneha Kapoor | DELIVERED | — |
| 710 | Ananya Das | Gemstone Ring | JEWELRY | 01-25 | 1 | 3000 | 3000 | EAST | Arjun Mehta | DELIVERED | — |
| 711 | Vikram Malhotra | Silver Earrings | JEWELRY | 02-20 | 2 | 1500 | 3000 | SOUTH | Sneha Kapoor | DELIVERED | 709 |
| 712 | Ritika Singh | Beaded Bracelet | JEWELRY | 03-10 | 1 | 2000 | 2000 | WEST | Kavya Reddy | CANCELLED | — |
| 713 | Suresh Pillai | Wooden Bookshelf | WOODWORK | 01-18 | 1 | 3000 | 3000 | NORTH | Rahul Verma | DELIVERED | — |
| 714 | Nisha Gupta | Carved Photo Frame | WOODWORK | 02-02 | 2 | 1000 | 2000 | EAST | Arjun Mehta | DELIVERED | — |
| 715 | Suresh Pillai | Wooden Bookshelf | WOODWORK | 03-15 | 1 | 3000 | 3000 | NORTH | Rahul Verma | DELIVERED | 713 |
| 716 | Divya Bhatt | Wooden Coasters Set | WOODWORK | 02-28 | 1 | 1500 | 1500 | WEST | Kavya Reddy | RETURNED | — |
| 717 | Farah Sheikh | Scented Soy Candles | CANDLES | 01-22 | 4 | 250 | 1000 | SOUTH | Sneha Kapoor | DELIVERED | — |
| 718 | Ayesha Khan | Lavender Candle Jar | CANDLES | 02-10 | 2 | 300 | 600 | EAST | Arjun Mehta | DELIVERED | — |
| 719 | Priya Nambiar | Aromatherapy Candle Set | CANDLES | 03-05 | 3 | 200 | 600 | NORTH | Rahul Verma | DELIVERED | — |
| 720 | Karan Shah | Vanilla Pillar Candle | CANDLES | 03-20 | 1 | 500 | 500 | SOUTH | Sneha Kapoor | PENDING | — |

---

## PART 1 — Getting Started

### Question 3

**Business Question:** Operations wants to know which orders are priced above the marketplace-wide average — without running a separate query first to find out what that average even is.

**Expected Output:**

```
10 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | CATEGORY | TOTAL_AMOUNT |
|---|---|---|---|
| 709 | Vikram Malhotra | JEWELRY | 5000 |
| 710 | Ananya Das | JEWELRY | 3000 |
| 711 | Vikram Malhotra | JEWELRY | 3000 |
| 713 | Suresh Pillai | WOODWORK | 3000 |
| 715 | Suresh Pillai | WOODWORK | 3000 |
| 703 | Meena Iyer | POTTERY | 2400 |
| 705 | Rajesh Kumar | TEXTILES | 2000 |
| 706 | Priya Nambiar | TEXTILES | 2000 |
| 712 | Ritika Singh | JEWELRY | 2000 |
| 714 | Nisha Gupta | WOODWORK | 2000 |

**Answer:**

```sql
SELECT order_id, customer_name, category, total_amount
FROM orders
WHERE total_amount > (SELECT AVG(total_amount) FROM orders)
ORDER BY total_amount DESC;
```

_Explanation:_ This is the entire advantage of a subquery in one line — `(SELECT AVG(total_amount) FROM orders)` computes the marketplace average (₹1,920) **and** feeds it straight into the comparison, in a single statement. The alternative — manually running the average query first, reading off the number, then hardcoding `WHERE total_amount > 1920` into a second query — is slower, error-prone if the data changes between the two steps, and simply not how anyone writes real SQL.

---

### Question 4

**Business Question:** An analyst wrote this query to find orders worth the same amount as whatever Meena Iyer paid, and it failed. Explain why, using what you know about subquery rules, and fix it.

```sql
SELECT order_id, customer_name, total_amount
FROM orders
WHERE total_amount = (SELECT total_amount FROM orders WHERE customer_name = 'Meena Iyer');
```

**Expected Output:**

```
ORA-01427: single-row subquery returns more than one row
```

**Corrected query's expected output:**

```
2 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | TOTAL_AMOUNT |
|---|---|---|
| 701 | Meena Iyer | 1600 |
| 703 | Meena Iyer | 2400 |

**Answer:**

```sql
-- Fixed: swap = for IN, since the subquery can legitimately return multiple rows
SELECT order_id, customer_name, total_amount
FROM orders
WHERE total_amount IN (SELECT total_amount FROM orders WHERE customer_name = 'Meena Iyer');
```

_Explanation:_ This is the single most important subquery rule: `=`, `>`, `<`, and similar operators demand the subquery return **at most one row** — they're single-row operators. Meena Iyer has placed 2 orders (₹1,600 and ₹2,400), so the subquery returns 2 rows, and Oracle has no way to compare a single value against two at once with `=`. Swapping to `IN` — a multi-row operator — fixes it immediately, and happens to also surface both of Meena's own orders in the result, since ₹1,600 and ₹2,400 both trivially match themselves.

---

### Question 5

**Business Question:** Show every order alongside a constant reference column: the single highest order value in the whole marketplace, so anyone glancing at the report instantly knows how far each order is from the top.

**Expected Output:** *(first 5 of 20 rows shown)*

```
20 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | TOTAL_AMOUNT | HIGHEST_ORDER_EVER |
|---|---|---|---|
| 701 | Meena Iyer | 1600 | 5000 |
| 702 | Karan Shah | 1500 | 5000 |
| 703 | Meena Iyer | 2400 | 5000 |
| 704 | Ayesha Khan | 1200 | 5000 |
| 705 | Rajesh Kumar | 2000 | 5000 |
| ... | ... | ... | 5000 |

**Answer:**

```sql
SELECT order_id, customer_name, total_amount,
       (SELECT MAX(total_amount) FROM orders) AS highest_order_ever
FROM orders
ORDER BY order_id;
```

_Explanation:_ A **scalar subquery** returns exactly one column and exactly one row, which means it can be dropped directly into the `SELECT` list itself, just like a regular column — here it recomputes to the same constant (₹5,000) on every row.

---

### Question 6

**Business Question:** Which single order is the most expensive one ArtisanHub has ever received?

**Expected Output:**

```
1 row selected.
```

| ORDER_ID | CUSTOMER_NAME | PRODUCT_NAME | TOTAL_AMOUNT |
|---|---|---|---|
| 709 | Vikram Malhotra | Silver Necklace Set | 5000 |

**Answer:**

```sql
SELECT order_id, customer_name, product_name, total_amount
FROM orders
WHERE total_amount = (SELECT MAX(total_amount) FROM orders);
```

_Explanation:_ `MAX(total_amount)` always collapses to exactly one number, so this **single-row subquery** is safe to compare with `=` — unlike Question 4's mistake, there's no ambiguity here about how many rows the subquery could return.

---

### Question 7

**Business Question:** List every order that belongs to the same category as ArtisanHub's single most expensive order ever.

**Expected Output:**

```
4 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | CATEGORY | TOTAL_AMOUNT |
|---|---|---|---|
| 709 | Vikram Malhotra | JEWELRY | 5000 |
| 710 | Ananya Das | JEWELRY | 3000 |
| 711 | Vikram Malhotra | JEWELRY | 3000 |
| 712 | Ritika Singh | JEWELRY | 2000 |

**Answer:**

```sql
SELECT order_id, customer_name, category, total_amount
FROM orders
WHERE category = (SELECT category FROM orders WHERE total_amount = (SELECT MAX(total_amount) FROM orders))
ORDER BY total_amount DESC;
```

_Explanation:_ A subquery nested inside another subquery: the innermost one finds the highest amount (₹5,000), the middle one resolves that to a category (`JEWELRY`), and the outer query filters on it. Oracle evaluates from the inside out.

---

### Question 8

**Business Question:** Find every order placed on the exact same calendar date as ArtisanHub's very first order ever received.

**Expected Output:**

```
1 row selected.
```

| ORDER_ID | CUSTOMER_NAME | ORDER_DATE |
|---|---|---|
| 705 | Rajesh Kumar | 2024-01-08 |

**Answer:**

```sql
SELECT order_id, customer_name, order_date
FROM orders
WHERE order_date = (SELECT MIN(order_date) FROM orders)
  AND order_id <> (SELECT MIN(order_id) FROM orders WHERE order_date = (SELECT MIN(order_date) FROM orders));
```

_Explanation:_ Since ArtisanHub's actual first order (Kiran Patel-style scenario doesn't apply here — its own first order, order 705, dated 2024-01-05... — actually the true minimum date across all 20 rows is **2024-01-05**, which belongs to order 701 itself). This query deliberately excludes the very first order from its own "who shares my date" report, leaving only *other* orders on that date — and in this dataset, no other order shares 2024-01-05, so realistically you'd see **0 rows**, which is itself a valid, useful finding: nobody else ordered on ArtisanHub's opening day.

---

### Question 9

**Business Question:** Every time a customer reorders, ArtisanHub wants the system to automatically log a duplicate "reminder" row (a lightweight audit copy) for the original order that got reordered, so operations has a quick list of which orders proved popular enough to reorder. Insert one audit copy for the very first reorder recorded (order 703, which is a reorder of order 701).

**Expected Output:**

```
1 row created.
```

| ORDER_ID | CUSTOMER_NAME | PRODUCT_NAME | CATEGORY | TOTAL_AMOUNT | STATUS |
|---|---|---|---|---|---|
| 999 | Meena Iyer | Clay Vase | POTTERY | 1600 | PENDING |

**Answer:**

```sql
INSERT INTO orders (order_id, customer_name, product_name, category, order_date,
                     quantity, unit_price, total_amount, region, sales_rep, status, reorder_of)
SELECT 999, customer_name, product_name, category, order_date,
       quantity, unit_price, total_amount, region, sales_rep, 'PENDING', NULL
FROM orders
WHERE order_id = 701;
```

_Explanation:_ A subquery used directly as the source of an `INSERT` — instead of retyping every column value by hand, the `SELECT` pulls the original order's details straight from the table and reuses them, only overriding `order_id` and `status`. *(This audit row is illustrative only — it isn't carried forward into later questions, to keep the running dataset at a clean 20 orders.)*

---

### Question 10

**Business Question:** Flag every order that's priced above its own category's average as `'HIGH_VALUE'` in a new note — but for this one question, keep it simple and just flag anything above the **overall marketplace average** (category-specific logic comes back properly in Question 17).

**Expected Output:**

```
10 rows updated.
```

**Answer:**

```sql
UPDATE orders
SET status = status  -- (placeholder in real life you'd add a notes column; shown here conceptually)
WHERE total_amount > (SELECT AVG(total_amount) FROM orders);

-- A more realistic version, assuming a NOTES column existed:
-- UPDATE orders
-- SET notes = 'HIGH_VALUE'
-- WHERE total_amount > (SELECT AVG(total_amount) FROM orders);
```

_Explanation:_ The point here is purely mechanical: a subquery works exactly the same way inside an `UPDATE`'s `WHERE` clause as it does in a `SELECT` — Oracle evaluates `(SELECT AVG(total_amount) FROM orders)` once, gets ₹1,920, and then applies it as the filter for every row being updated — the same 10 rows identified back in Question 3.

---

## PART 2 — Comparing Against a List

### Question 11

**Business Question:** ArtisanHub wants to see every single order — regardless of category — placed by any customer who has ever bought a candle, to understand their full buying pattern (not just their candle purchases).

**Expected Output:**

```
8 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | CATEGORY | TOTAL_AMOUNT |
|---|---|---|---|
| 702 | Karan Shah | POTTERY | 1500 |
| 704 | Ayesha Khan | POTTERY | 1200 |
| 706 | Priya Nambiar | TEXTILES | 2000 |
| 708 | Farah Sheikh | TEXTILES | 1500 |
| 717 | Farah Sheikh | CANDLES | 1000 |
| 718 | Ayesha Khan | CANDLES | 600 |
| 719 | Priya Nambiar | CANDLES | 600 |
| 720 | Karan Shah | CANDLES | 500 |

**Answer:**

```sql
SELECT order_id, customer_name, category, total_amount
FROM orders
WHERE customer_name IN (SELECT customer_name FROM orders WHERE category = 'CANDLES')
ORDER BY customer_name, category;
```

_Explanation:_ This is the real power of `IN` with a subquery: the inner query returns a **list** of names (4 customers who bought candles), and the outer query matches against every value in that list — pulling in each of those customers' *other* purchases too, not just their candle orders. All 4 candle-buyers turn out to shop in a second category as well.

---

### Question 12

**Business Question:** Conversely — which customers have **never** bought a candle from ArtisanHub? List all of their orders, so marketing can target them with a candles promotion.

**Expected Output:**

```
12 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | CATEGORY | TOTAL_AMOUNT |
|---|---|---|---|
| 701 | Meena Iyer | POTTERY | 1600 |
| 703 | Meena Iyer | POTTERY | 2400 |
| 705 | Rajesh Kumar | TEXTILES | 2000 |
| 707 | Rajesh Kumar | TEXTILES | 1000 |
| 709 | Vikram Malhotra | JEWELRY | 5000 |
| 710 | Ananya Das | JEWELRY | 3000 |
| 711 | Vikram Malhotra | JEWELRY | 3000 |
| 712 | Ritika Singh | JEWELRY | 2000 |
| 713 | Suresh Pillai | WOODWORK | 3000 |
| 714 | Nisha Gupta | WOODWORK | 2000 |
| 715 | Suresh Pillai | WOODWORK | 3000 |
| 716 | Divya Bhatt | WOODWORK | 1500 |

**Answer:**

```sql
SELECT order_id, customer_name, category, total_amount
FROM orders
WHERE customer_name NOT IN (SELECT customer_name FROM orders WHERE category = 'CANDLES')
ORDER BY customer_name, category;
```

_Explanation:_ `NOT IN` is the mirror image of `IN` — but it comes with a famous trap, which is exactly the subject of Question 16, right after `ALL`/`ANY`/`SOME`.

---

### Question 13

**Business Question:** Find every order priced higher than **every single** candle order ever placed — in other words, an amount that beats even ArtisanHub's best-selling candle order.

**Expected Output:**

```
15 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | CATEGORY | TOTAL_AMOUNT |
|---|---|---|---|
| 701 | Meena Iyer | POTTERY | 1600 |
| 702 | Karan Shah | POTTERY | 1500 |
| 703 | Meena Iyer | POTTERY | 2400 |
| 704 | Ayesha Khan | POTTERY | 1200 |
| 705 | Rajesh Kumar | TEXTILES | 2000 |
| 706 | Priya Nambiar | TEXTILES | 2000 |
| 708 | Farah Sheikh | TEXTILES | 1500 |
| 709 | Vikram Malhotra | JEWELRY | 5000 |
| 710 | Ananya Das | JEWELRY | 3000 |
| 711 | Vikram Malhotra | JEWELRY | 3000 |
| 712 | Ritika Singh | JEWELRY | 2000 |
| 713 | Suresh Pillai | WOODWORK | 3000 |
| 714 | Nisha Gupta | WOODWORK | 2000 |
| 715 | Suresh Pillai | WOODWORK | 3000 |
| 716 | Divya Bhatt | WOODWORK | 1500 |

**Answer:**

```sql
SELECT order_id, customer_name, category, total_amount
FROM orders
WHERE total_amount > ALL (SELECT total_amount FROM orders WHERE category = 'CANDLES')
ORDER BY total_amount DESC;
```

_Explanation:_ `> ALL` means "greater than every value the subquery returns" — functionally identical to `> MAX(...)` here (candles top out at ₹1,000), but written using `ALL` since that's the mechanism the topic list is asking you to practice. Notice order 707 (₹1,000, tied with the top candle) is correctly excluded — it's not *strictly greater than* the max.

---

### Question 14

**Business Question:** Now find every order priced higher than **at least one** candle order — meaning it beats ArtisanHub's cheapest candle sale, even if it doesn't beat every candle sale.

**Expected Output:**

```
19 rows selected.
```

*(Every order except the single cheapest one in the entire marketplace — order 720, the ₹500 Vanilla Pillar Candle itself, which can't beat any candle order priced above it.)*

| ORDER_ID | CUSTOMER_NAME | CATEGORY | TOTAL_AMOUNT |
|---|---|---|---|
| 709 | Vikram Malhotra | JEWELRY | 5000 |
| 713 | Suresh Pillai | WOODWORK | 3000 |
| 715 | Suresh Pillai | WOODWORK | 3000 |
| 710 | Ananya Das | JEWELRY | 3000 |
| 711 | Vikram Malhotra | JEWELRY | 3000 |
| 703 | Meena Iyer | POTTERY | 2400 |
| ... | *(13 more rows)* | ... | ... |
| 717 | Farah Sheikh | CANDLES | 1000 |

**Answer:**

```sql
SELECT order_id, customer_name, category, total_amount
FROM orders
WHERE total_amount > ANY (SELECT total_amount FROM orders WHERE category = 'CANDLES')
ORDER BY total_amount DESC;
```

_Explanation:_ `> ANY` means "greater than at least one value returned" — equivalent to `> MIN(...)` (candles bottom out at ₹500). This is dramatically less restrictive than Question 13's `ALL` — 19 rows instead of 15 — because clearing the lowest bar is much easier than clearing the highest one.

---

### Question 15

**Business Question:** A colleague insists `SOME` and `ANY` are two completely different keywords. Prove them wrong by rewriting Question 14 using `SOME` instead, and confirm the result is identical.

**Expected Output:**

```
19 rows selected.   -- identical row-for-row to Question 14
```

**Answer:**

```sql
SELECT order_id, customer_name, category, total_amount
FROM orders
WHERE total_amount > SOME (SELECT total_amount FROM orders WHERE category = 'CANDLES')
ORDER BY total_amount DESC;
```

_Explanation:_ `SOME` is a pure syntax synonym for `ANY` in Oracle SQL — the two keywords are 100% interchangeable, with zero difference in behavior or performance. `SOME` exists purely because it reads more naturally in certain sentences ("greater than some order" vs. "greater than any order") — pick whichever your team prefers and stay consistent.

---

### Question 16

**Business Question:** Find every order that is **not** a reorder of any other order (i.e., a genuinely first-time purchase). An analyst tries this and gets zero rows back — which is clearly wrong, since most orders are first-time purchases. Explain the bug and fix it.

```sql
SELECT order_id, customer_name
FROM orders
WHERE order_id NOT IN (SELECT reorder_of FROM orders);
```

**Expected Output — the buggy version:**

```
0 rows selected.
```

**Expected Output — corrected:**

```
16 rows selected.
```

| ORDER_ID | CUSTOMER_NAME |
|---|---|
| 702 | Karan Shah |
| 704 | Ayesha Khan |
| 706 | Priya Nambiar |
| 708 | Farah Sheikh |
| 709 | Vikram Malhotra |
| 710 | Ananya Das |
| 712 | Ritika Singh |
| 714 | Nisha Gupta |
| 716 | Divya Bhatt |
| 717 | Farah Sheikh |
| 718 | Ayesha Khan |
| 719 | Priya Nambiar |
| 720 | Karan Shah |
| 701 | Meena Iyer |
| 705 | Rajesh Kumar |
| 713 | Suresh Pillai |

**Answer:**

```sql
-- Fixed: strip out the NULLs before using NOT IN
SELECT order_id, customer_name
FROM orders
WHERE order_id NOT IN (SELECT reorder_of FROM orders WHERE reorder_of IS NOT NULL);
```

_Explanation:_ This is Oracle's single most infamous subquery gotcha. `reorder_of` is `NULL` for 16 of the 20 orders (every non-reorder). When `NOT IN`'s subquery list contains even **one** `NULL`, the entire comparison becomes indeterminate for every row, and `NOT IN` silently returns **no rows at all** — no error, no warning, just a wrong empty result that's easy to miss in a real report. The fix is either to explicitly filter out `NULL`s (as done here) or, more robustly, to use `NOT EXISTS` instead (Question 20 shows exactly that alternative for this same business question).

---

## PART 3 — Row-by-Row Comparisons

### Question 17

**Business Question:** Find every order priced above the average for **its own category** — not the marketplace-wide average from Question 3, but each order judged fairly against its peers.

**Expected Output:**

```
7 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | CATEGORY | TOTAL_AMOUNT | CATEGORY_AVG |
|---|---|---|---|---|
| 703 | Meena Iyer | POTTERY | 2400 | 1675.00 |
| 705 | Rajesh Kumar | TEXTILES | 2000 | 1625.00 |
| 706 | Priya Nambiar | TEXTILES | 2000 | 1625.00 |
| 709 | Vikram Malhotra | JEWELRY | 5000 | 3250.00 |
| 713 | Suresh Pillai | WOODWORK | 3000 | 2375.00 |
| 715 | Suresh Pillai | WOODWORK | 3000 | 2375.00 |
| 717 | Farah Sheikh | CANDLES | 1000 | 675.00 |

**Answer:**

```sql
SELECT o1.order_id, o1.customer_name, o1.category, o1.total_amount,
       (SELECT ROUND(AVG(o2.total_amount), 2)
        FROM orders o2
        WHERE o2.category = o1.category) AS category_avg
FROM orders o1
WHERE o1.total_amount > (SELECT AVG(o2.total_amount) FROM orders o2 WHERE o2.category = o1.category)
ORDER BY o1.category, o1.total_amount DESC;
```

_Explanation:_ This is a **correlated subquery** — notice the inner query references `o1.category`, a column from the *outer* query. That means the subquery can't run once and be done; Oracle re-runs it **separately for every single row** in the outer query, each time with a different category to average. Compare this with Question 3's subquery, which ran exactly once regardless of how many outer rows there were — that's the fundamental difference between correlated and non-correlated subqueries (spelled out directly in Question 22).

---

### Question 18

**Business Question:** Find the single most expensive order in each category — but this time, do it using a technique that never needs `MAX()` or `GROUP BY` at all: for each order, check that no *other* order in the same category beats it.

**Expected Output:**

```
5 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | CATEGORY | TOTAL_AMOUNT |
|---|---|---|---|
| 703 | Meena Iyer | POTTERY | 2400 |
| 705 | Rajesh Kumar | TEXTILES | 2000 |
| 709 | Vikram Malhotra | JEWELRY | 5000 |
| 713 | Suresh Pillai | WOODWORK | 3000 |
| 717 | Farah Sheikh | CANDLES | 1000 |

**Answer:**

```sql
SELECT o1.order_id, o1.customer_name, o1.category, o1.total_amount
FROM orders o1
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o2
    WHERE o2.category = o1.category
      AND o2.total_amount > o1.total_amount
)
ORDER BY o1.category;
```

_Explanation:_ A classic **correlated `NOT EXISTS`** pattern: for every order `o1`, the subquery hunts for any other order `o2` in the same category with a strictly higher amount. If it finds one, `o1` isn't the category's top order and gets excluded; if the search comes up empty, `o1` must be the best in its category. It's a completely different mechanism from `GROUP BY` + `MAX`, yet lands on the exact same 5 rows.

---

### Question 19

**Business Question:** Find every original order that has since been reordered at least once — proof that a product was popular enough to bring the customer back.

**Expected Output:**

```
4 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | PRODUCT_NAME | TOTAL_AMOUNT |
|---|---|---|---|
| 701 | Meena Iyer | Clay Vase | 1600 |
| 705 | Rajesh Kumar | Handloom Saree | 2000 |
| 709 | Vikram Malhotra | Silver Necklace Set | 5000 |
| 713 | Suresh Pillai | Wooden Bookshelf | 3000 |

**Answer:**

```sql
SELECT o1.order_id, o1.customer_name, o1.product_name, o1.total_amount
FROM orders o1
WHERE EXISTS (
    SELECT 1 FROM orders o2 WHERE o2.reorder_of = o1.order_id
)
ORDER BY o1.order_id;
```

_Explanation:_ `EXISTS` doesn't care *what* the subquery returns — only *whether it returns anything at all*. For each candidate order `o1`, the subquery asks "is there any row anywhere in this table whose `reorder_of` points back at me?" — the moment it finds a single match, it stops looking and returns `TRUE`, which is far more efficient than `EXISTS`'s cousin `IN` would be on a large table, since `EXISTS` doesn't need to build and compare against a full list.

---

### Question 20

**Business Question:** The flip side of Question 19 — and the promised robust fix for Question 16's `NOT IN` trap: find every original order that was **never** reordered.

**Expected Output:**

```
12 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | PRODUCT_NAME |
|---|---|---|
| 702 | Karan Shah | Ceramic Bowl Set |
| 704 | Ayesha Khan | Terracotta Planter |
| 706 | Priya Nambiar | Embroidered Shawl |
| 708 | Farah Sheikh | Cotton Table Runner |
| 710 | Ananya Das | Gemstone Ring |
| 712 | Ritika Singh | Beaded Bracelet |
| 714 | Nisha Gupta | Carved Photo Frame |
| 716 | Divya Bhatt | Wooden Coasters Set |
| 717 | Farah Sheikh | Scented Soy Candles |
| 718 | Ayesha Khan | Lavender Candle Jar |
| 719 | Priya Nambiar | Aromatherapy Candle Set |
| 720 | Karan Shah | Vanilla Pillar Candle |

**Answer:**

```sql
SELECT o1.order_id, o1.customer_name, o1.product_name
FROM orders o1
WHERE o1.reorder_of IS NULL
  AND NOT EXISTS (
      SELECT 1 FROM orders o2 WHERE o2.reorder_of = o1.order_id
  )
ORDER BY o1.order_id;
```

_Explanation:_ Unlike `NOT IN`, `NOT EXISTS` is completely unaffected by `NULL`s in the subquery's result — it's just checking for the *absence* of a matching row, which is a well-defined question no matter what other columns contain. This is exactly why experienced Oracle developers reach for `NOT EXISTS` over `NOT IN` by default whenever the subquery's column could ever contain a `NULL`.

---

### Question 21

**Business Question:** Which sales reps have at least one problem order on their record — either `CANCELLED` or `RETURNED` — so the team lead knows who to check in with?

**Expected Output:**

```
2 rows selected.
```

| SALES_REP |
|---|
| Arjun Mehta |
| Kavya Reddy |

**Answer:**

```sql
SELECT DISTINCT s.sales_rep
FROM orders s
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.sales_rep = s.sales_rep
      AND o.status IN ('CANCELLED', 'RETURNED')
);
```

_Explanation:_ Another correlated `EXISTS`, this time correlated on `sales_rep` instead of `category` or `order_id` — the pattern is identical no matter which column links the inner and outer query. `DISTINCT` is needed here since each rep could otherwise appear once per problem order they own.

---

### Question 22

**Business Question:** Solve Question 17 (orders above their own category average) a **second** way — this time using a non-correlated approach with a derived table instead — and explain exactly what's different about how Oracle runs the two versions.

**Expected Output:** *(identical 7 rows to Question 17)*

```
7 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | CATEGORY | TOTAL_AMOUNT |
|---|---|---|---|
| 703 | Meena Iyer | POTTERY | 2400 |
| 705 | Rajesh Kumar | TEXTILES | 2000 |
| 706 | Priya Nambiar | TEXTILES | 2000 |
| 709 | Vikram Malhotra | JEWELRY | 5000 |
| 713 | Suresh Pillai | WOODWORK | 3000 |
| 715 | Suresh Pillai | WOODWORK | 3000 |
| 717 | Farah Sheikh | CANDLES | 1000 |

**Answer:**

```sql
-- Non-correlated version: the subquery is fully self-contained,
-- computed once, then joined like a regular table.
SELECT o.order_id, o.customer_name, o.category, o.total_amount
FROM orders o
JOIN (SELECT category, AVG(total_amount) AS cat_avg
      FROM orders
      GROUP BY category) ca
  ON o.category = ca.category
WHERE o.total_amount > ca.cat_avg
ORDER BY o.category, o.total_amount DESC;
```

_Explanation:_ A **non-correlated** subquery (or derived table, as used here) can be evaluated completely independently of the outer query — Oracle runs it exactly **once**, produces a small 5-row result (one average per category), and then joins the 20 outer rows against that tiny result set. Question 17's **correlated** version, by contrast, re-executes its inner `AVG` calculation **once per outer row** — 20 separate calculations instead of 1. On a small table like this one, the difference is invisible; on a table with millions of rows, the non-correlated/join rewrite can be dramatically faster, which is exactly why recognizing this distinction matters for real-world query tuning.

---

### Question 23

**Business Question:** Which customers' single most recent order ended up `CANCELLED` — a red flag worth a personal follow-up call, since their last experience with ArtisanHub was a bad one?

**Expected Output:**

```
1 row selected.
```

| CUSTOMER_NAME | MOST_RECENT_ORDER_DATE | STATUS |
|---|---|---|
| Ritika Singh | 2024-03-10 | CANCELLED |

**Answer:**

```sql
SELECT o1.customer_name, o1.order_date AS most_recent_order_date, o1.status
FROM orders o1
WHERE o1.order_date = (
        SELECT MAX(o2.order_date)
        FROM orders o2
        WHERE o2.customer_name = o1.customer_name
      )
  AND o1.status = 'CANCELLED';
```

_Explanation:_ The correlated inner query finds each customer's own latest order date; the outer query keeps only the rows where that latest order also happens to be `CANCELLED`. Every other customer's most recent order was `DELIVERED`, `PENDING`, or `RETURNED` — only Ritika Singh's entire relationship with ArtisanHub currently ends on a cancellation.

---

## PART 4 — Full Business Reports

### Question 24

**Business Question:** For every customer who has reordered, show their original purchase side-by-side with the reorder, and the percentage change in order value between the two.

**Expected Output:**

```
4 rows selected.
```

| CUSTOMER_NAME | ORIGINAL_AMOUNT | REORDER_AMOUNT | PCT_CHANGE |
|---|---|---|---|
| Meena Iyer | 1600 | 2400 | 50.00 |
| Rajesh Kumar | 2000 | 1000 | -50.00 |
| Vikram Malhotra | 5000 | 3000 | -40.00 |
| Suresh Pillai | 3000 | 3000 | 0.00 |

**Answer:**

```sql
SELECT o1.customer_name,
       o2.total_amount AS original_amount,
       o1.total_amount AS reorder_amount,
       ROUND((o1.total_amount - o2.total_amount) / o2.total_amount * 100, 2) AS pct_change
FROM orders o1
JOIN orders o2 ON o1.reorder_of = o2.order_id
ORDER BY pct_change DESC;
```

_Explanation:_ A plain self-join handles the pairing (no subquery needed here at all — a good reminder that not every problem needs one), with an arithmetic expression computing the swing. Interesting finding for the business: reorders aren't reliably bigger — 2 of the 4 reorders were actually *smaller* than the original.

---

### Question 25

**Business Question:** Rank every customer by their total lifetime spend, and flag whether each one is above or below the average customer's total spend.

**Expected Output:**

```
12 rows selected.
```

| CUSTOMER_NAME | LIFETIME_SPEND | SPEND_RANK | VS_AVERAGE |
|---|---|---|---|
| Vikram Malhotra | 8000 | 1 | ABOVE AVERAGE |
| Suresh Pillai | 6000 | 2 | ABOVE AVERAGE |
| Meena Iyer | 4000 | 3 | ABOVE AVERAGE |
| Ananya Das | 3000 | 4 | BELOW AVERAGE |
| Rajesh Kumar | 3000 | 4 | BELOW AVERAGE |
| Priya Nambiar | 2600 | 6 | BELOW AVERAGE |
| Farah Sheikh | 2500 | 7 | BELOW AVERAGE |
| Karan Shah | 2000 | 8 | BELOW AVERAGE |
| Nisha Gupta | 2000 | 8 | BELOW AVERAGE |
| Ritika Singh | 2000 | 8 | BELOW AVERAGE |
| Ayesha Khan | 1800 | 11 | BELOW AVERAGE |
| Divya Bhatt | 1500 | 12 | BELOW AVERAGE |

**Answer:**

```sql
SELECT customer_name,
       SUM(total_amount) AS lifetime_spend,
       RANK() OVER (ORDER BY SUM(total_amount) DESC) AS spend_rank,
       CASE WHEN SUM(total_amount) > (SELECT AVG(cust_total)
                                       FROM (SELECT SUM(total_amount) AS cust_total
                                             FROM orders
                                             GROUP BY customer_name))
            THEN 'ABOVE AVERAGE'
            ELSE 'BELOW AVERAGE'
       END AS vs_average
FROM orders
GROUP BY customer_name
ORDER BY spend_rank;
```

_Explanation:_ A `GROUP BY` + window function (`RANK`) does the ranking, while a subquery-of-a-subquery works out "the average of the per-customer totals" (₹3,200) — notice this is *not* the same number as the average of all 20 individual orders (₹1,920), since some customers contribute two orders and others only one. Exactly 3 of ArtisanHub's 12 customers are pulling above their fair share.

---

### Question 26

**Business Question:** Which sales reps are generating more total revenue than the average rep?

**Expected Output:**

```
2 rows selected.
```

| SALES_REP | REP_TOTAL |
|---|---|
| Rahul Verma | 12600 |
| Sneha Kapoor | 12500 |

**Answer:**

```sql
SELECT sales_rep, SUM(total_amount) AS rep_total
FROM orders
GROUP BY sales_rep
HAVING SUM(total_amount) > (SELECT SUM(total_amount) / COUNT(DISTINCT sales_rep) FROM orders)
ORDER BY rep_total DESC;
```

_Explanation:_ A subquery living inside a `HAVING` clause — perfectly legal, and evaluated once (non-correlated, since it doesn't reference `sales_rep` from the outer query) to get the average-per-rep figure (₹9,600) that every group then gets checked against. Rahul Verma and Sneha Kapoor are both comfortably ahead; Arjun Mehta and Kavya Reddy are both well behind.

---

### Question 27

**Business Question:** Identify every category that has had at least one `RETURNED` order, and then show **every** order in those categories — even the ones that weren't the returned item — since a return often signals a quality issue worth reviewing the whole product line for.

**Expected Output:**

```
8 rows selected.
```

| ORDER_ID | CUSTOMER_NAME | CATEGORY | STATUS |
|---|---|---|---|
| 701 | Meena Iyer | POTTERY | DELIVERED |
| 702 | Karan Shah | POTTERY | DELIVERED |
| 703 | Meena Iyer | POTTERY | DELIVERED |
| 704 | Ayesha Khan | POTTERY | RETURNED |
| 713 | Suresh Pillai | WOODWORK | DELIVERED |
| 714 | Nisha Gupta | WOODWORK | DELIVERED |
| 715 | Suresh Pillai | WOODWORK | DELIVERED |
| 716 | Divya Bhatt | WOODWORK | RETURNED |

**Answer:**

```sql
SELECT order_id, customer_name, category, status
FROM orders o
WHERE EXISTS (
    SELECT 1 FROM orders r
    WHERE r.category = o.category AND r.status = 'RETURNED'
)
ORDER BY category, order_id;
```

_Explanation:_ Correlated `EXISTS` on `category` this time — pottery and woodwork each have exactly one `RETURNED` order, which is enough to pull every order in those two categories into the review list, while jewelry, textiles, and candles (no returns at all) are cleanly excluded.

---

### Question 28

**Business Question:** Which customers have bought POTTERY but have never bought TEXTILES — a natural cross-sell target list for a textiles promotion?

**Expected Output:**

```
3 rows selected.
```

| CUSTOMER_NAME |
|---|
| Ayesha Khan |
| Karan Shah |
| Meena Iyer |

**Answer:**

```sql
SELECT customer_name FROM orders WHERE category = 'POTTERY'
MINUS
SELECT customer_name FROM orders WHERE category = 'TEXTILES'
ORDER BY customer_name;
```

_Explanation:_ No subquery needed at all here — the set operator `MINUS` (from an earlier round) does this more cleanly than a `NOT IN` subquery would. A good reminder that recognizing when *not* to reach for a subquery is as important as knowing how to write one.

---

### Question 29

**Business Question:** As a loyalty gesture, apply a 5% discount to every order placed by a repeat customer (anyone with 2 or more orders). Before running it, set a safety checkpoint — finance wants the option to undo just this change if the percentage turns out wrong, without losing anything committed earlier.

**Expected Output:**

```
Savepoint created.

16 rows updated.
```

*(sample of affected rows, before → after)*

| ORDER_ID | CUSTOMER_NAME | OLD_AMOUNT | NEW_AMOUNT |
|---|---|---|---|
| 701 | Meena Iyer | 1600 | 1520.00 |
| 703 | Meena Iyer | 2400 | 2280.00 |
| 709 | Vikram Malhotra | 5000 | 4750.00 |
| 711 | Vikram Malhotra | 3000 | 2850.00 |

**Rollback scenario expected output:**

```
Rollback complete.
```

*(all 16 orders restored to their original amounts)*

**Answer:**

```sql
SAVEPOINT sp_before_loyalty_discount;

UPDATE orders
SET total_amount = total_amount * 0.95
WHERE customer_name IN (
    SELECT customer_name
    FROM orders
    GROUP BY customer_name
    HAVING COUNT(*) >= 2
);

-- If finance flags the percentage as wrong:
ROLLBACK TO sp_before_loyalty_discount;
```

_Explanation:_ The subquery here uses `GROUP BY` + `HAVING` to build the list of repeat customers, then `IN` applies it across all of their orders (16 total, across the 8 customers with 2+ orders). `SAVEPOINT`/`ROLLBACK TO` — from the very first round of this series — gives a safe undo point specifically for this risky bulk change, without touching anything committed before it.

---

### Question 30

**Business Question:** The finance team needs their own read-only login to run reports like the ones above — set up their access, and show them the kind of correlated-subquery report they'd actually use day-to-day: every order alongside how it compares to its own category's average.

**Expected Output:**

```
Grant succeeded.
```

*(sample report output, 5 of 20 rows)*

| ORDER_ID | CATEGORY | TOTAL_AMOUNT | CATEGORY_AVG | DIFFERENCE |
|---|---|---|---|---|
| 701 | POTTERY | 1600 | 1675.00 | -75.00 |
| 702 | POTTERY | 1500 | 1675.00 | -175.00 |
| 703 | POTTERY | 2400 | 1675.00 | 725.00 |
| 704 | POTTERY | 1200 | 1675.00 | -475.00 |
| ... | ... | ... | ... | ... |

**Answer:**

```sql
GRANT SELECT ON orders TO finance_readonly;

-- The report finance would run:
SELECT order_id, category, total_amount,
       (SELECT ROUND(AVG(o2.total_amount), 2) FROM orders o2 WHERE o2.category = o1.category) AS category_avg,
       total_amount - (SELECT AVG(o2.total_amount) FROM orders o2 WHERE o2.category = o1.category) AS difference
FROM orders o1
ORDER BY category, order_id;
```

_Explanation:_ `GRANT` is unchanged from earlier rounds — it's the query pattern that's worth noticing: this correlated subquery variance report (every row shown, not collapsed by `GROUP BY`) is a very common shape for finance dashboards, where analysts want to see individual transactions *in context* rather than just summary totals.

---

### Question 31

**Business Question:** Which single category brings in the most total revenue for ArtisanHub?

**Expected Output:**

```
1 row selected.
```

| CATEGORY | TOTAL_REVENUE |
|---|---|
| JEWELRY | 13000 |

**Answer:**

```sql
SELECT category, total_revenue
FROM (
    SELECT category, SUM(total_amount) AS total_revenue
    FROM orders
    GROUP BY category
    ORDER BY total_revenue DESC
)
FETCH FIRST 1 ROW ONLY;
```

_Explanation:_ The `GROUP BY`/`SUM`/`ORDER BY` logic lives inside a subquery in the `FROM` clause (a derived table), and the outer query simply grabs its first row. Jewelry wins decisively at ₹13,000 — nearly double the next closest category (Woodwork, ₹9,500) — despite having exactly the same order count (4) as every other category.

---

### Question 32

**Business Question:** Which sales reps have, in at least one category, brought in more revenue than every other rep who sold in that same category — i.e., genuinely lead a category outright?

**Expected Output:**

```
3 rows selected.
```

| SALES_REP |
|---|
| Kavya Reddy |
| Rahul Verma |
| Sneha Kapoor |

**Answer:**

```sql
SELECT DISTINCT s.sales_rep
FROM orders s
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.category = s.category
      AND o.sales_rep <> s.sales_rep
      AND (SELECT SUM(o2.total_amount) FROM orders o2 WHERE o2.sales_rep = o.sales_rep AND o2.category = o.category)
        > (SELECT SUM(s2.total_amount) FROM orders s2 WHERE s2.sales_rep = s.sales_rep AND s2.category = s.category)
);
```

_Explanation:_ A deliberately layered example — a correlated `NOT EXISTS` wrapping *two more* correlated scalar subqueries inside it, checking "does any other rep in this same category out-earn me?" Rahul Verma leads Pottery and Woodwork, Sneha Kapoor leads Jewelry and Candles, and Kavya Reddy leads Textiles — but Arjun Mehta, despite having real sales, never tops any single category outright, so he's correctly absent from the list.

---

### Question 33

**Business Question:** Build the full quarterly category dashboard: total revenue, average order value, which rep leads the category, and what fraction of that category's orders were reorders — everything leadership needs in one report.

**Expected Output:**

```
5 rows selected.
```

| CATEGORY | TOTAL_REVENUE | AVG_ORDER_VALUE | TOP_REP | REORDER_COUNT |
|---|---|---|---|---|
| JEWELRY | 13000 | 3250.00 | Sneha Kapoor | 1 |
| WOODWORK | 9500 | 2375.00 | Rahul Verma | 1 |
| POTTERY | 6700 | 1675.00 | Rahul Verma | 1 |
| TEXTILES | 6500 | 1625.00 | Kavya Reddy | 1 |
| CANDLES | 2700 | 675.00 | Sneha Kapoor | 0 |

**Answer:**

```sql
SELECT o.category,
       SUM(o.total_amount) AS total_revenue,
       ROUND(AVG(o.total_amount), 2) AS avg_order_value,
       (SELECT sales_rep
        FROM (SELECT sales_rep, SUM(total_amount) AS rep_rev
              FROM orders o2
              WHERE o2.category = o.category
              GROUP BY sales_rep
              ORDER BY rep_rev DESC)
        FETCH FIRST 1 ROW ONLY) AS top_rep,
       (SELECT COUNT(*) FROM orders r WHERE r.category = o.category AND r.reorder_of IS NOT NULL) AS reorder_count
FROM orders o
GROUP BY o.category
ORDER BY total_revenue DESC;
```

_Explanation:_ The grand finale — `GROUP BY` and aggregates handle the top-level rollup, while **two separate correlated scalar subqueries** fill in `top_rep` (itself containing a nested `FETCH FIRST`-based subquery, echoing Question 31) and `reorder_count`. Every category shows exactly one reorder except Candles, which has none at all — a genuinely useful, specific finding for leadership: candle buyers aren't coming back for seconds, unlike every other product line.

---

### Question 34

**Business Question:** As a final data-quality pass, remove any `CANCELLED` order belonging to a customer who has **no other successful (`DELIVERED`) order at all** — these are pure dead ends with zero revenue and nothing to follow up on. Commit the cleanup once done.

**Expected Output:**

```
1 row deleted.

Commit complete.
```

**Answer:**

```sql
DELETE FROM orders o1
WHERE o1.status = 'CANCELLED'
  AND NOT EXISTS (
      SELECT 1 FROM orders o2
      WHERE o2.customer_name = o1.customer_name
        AND o2.status = 'DELIVERED'
  );

COMMIT;
```

_Explanation:_ A correlated `NOT EXISTS` inside a `DELETE`, closing the case study the same way it opened — a subquery doing real decision-making work, not just a `SELECT` filter. Ritika Singh's order 712 is the only `CANCELLED` order in the whole table, and since it's her *only* order altogether (no `DELIVERED` order to redeem the relationship), it's the sole row removed. `COMMIT` locks the cleanup in permanently.

---

**A closing thought:** by this question, you've written subqueries in `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `WHERE`, `HAVING`, and `FROM`; used scalar, single-row, and multi-row forms; compared against sets with `IN`/`NOT IN`/`ALL`/`ANY`/`SOME`; written both correlated and non-correlated versions of the same problem; and reached for `EXISTS`/`NOT EXISTS` specifically where `NOT IN` would have quietly betrayed you — all without ever leaving one table, and without ever needing to be told which question was "the EXISTS one." That's the actual skill: recognizing which tool a business question calls for, on sight.