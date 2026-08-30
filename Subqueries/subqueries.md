### Core Strategy & Rules

A subquery (inner query) executes to provide a value or set of values to the main (outer) query.

* **Strategic Mapping:** When reading a prompt, ask: *"What data do I need before I can find the final answer?"* If asked, "Who earns more than the company average?", you cannot find the "who" without first finding the "average." The average calculation becomes your subquery.
* **Syntax Rules:** Subqueries must be enclosed in parentheses, typically placed on the right side of the comparison operator, and should generally not include an `ORDER BY` clause.
* **Advantages:** They allow for step-by-step logical breakdown, often replacing complex joins and making queries significantly easier to read and debug.

### Foundation Tables for Practice

Run this setup to practice the scenarios below:

```sql
CREATE TABLE Dept (DeptID INT, DeptName VARCHAR(20));
CREATE TABLE Emp (EmpID INT, Name VARCHAR(20), Salary INT, DeptID INT);
INSERT INTO Dept VALUES (10, 'IT'), (20, 'HR'), (30, 'Sales');
INSERT INTO Emp VALUES (1, 'Alice', 5000, 10), (2, 'Bob', 7000, 10), (3, 'Charlie', 4000, 20);

```

### Non-Correlated Subqueries (Independent)

These subqueries are self-contained. They run exactly once, independently of the outer query, and pass their result outward.

* **Scalar / Single-Row:** Returns exactly one value. Use single-value operators (`=, >, <`).
* *Scenario:* Find employees earning more than Alice.
* *Query:* `SELECT Name FROM Emp WHERE Salary > (SELECT Salary FROM Emp WHERE Name = 'Alice');`


* **Multiple-Row:** Returns a column of multiple values. You cannot use `=`, so you must use `IN`, `NOT IN`, `ANY`, or `ALL`.
* *Scenario:* Find employees working in departments that exist in the `Dept` table.
* *Query:* `SELECT Name FROM Emp WHERE DeptID IN (SELECT DeptID FROM Dept);`



### Correlated Subqueries (Dependent)

Unlike non-correlated queries, these execute row-by-row. The inner query references a column from the outer query, meaning it cannot run independently.

* **The Logic:** For every row the outer query evaluates, the inner query recalculates using that specific row's data.
* **EXISTS / NOT EXISTS:** Often used with correlated subqueries. `EXISTS` evaluates to true if the subquery returns at least one row, making it highly efficient for checking conditions without retrieving heavy data.
* *Scenario:* Find employees earning more than the average salary of their *own* specific department.
* *Query:* `SELECT Name, Salary FROM Emp e1 WHERE Salary > (SELECT AVG(Salary) FROM Emp e2 WHERE e1.DeptID = e2.DeptID);`

### Using Subqueries With SELECT, INSERT, UPDATE, DELETE

**The Strategic Mapping:** Whenever you need to move, change, or remove data, but you think, *"Wait, I need to look up a piece of information before I know exactly what to change,"* you need a subquery.

Let's create a temporary archive table to practice these operations without ruining our main `Employees` table:

```sql
CREATE TABLE Emp_Archive (
    Emp_ID INT,
    First_Name VARCHAR(50),
    Salary DECIMAL(10, 2)
);

```

#### A. Subqueries with INSERT

**When to apply:** You want to copy a batch of data from one table to another based on a specific condition. Instead of writing out `VALUES (1, 'Alice', ...)` manually for 100 rows, you let a query do the heavy lifting.

* **The Logic:** "Insert records into my Archive table, but instead of providing explicit values, go run this `SELECT` query and insert its results."
* **Scenario:** Archive all employees who earn more than 70,000.
* **Query:**

```sql
INSERT INTO Emp_Archive (Emp_ID, First_Name, Salary)
SELECT Emp_ID, First_Name, Salary 
FROM Employees 
WHERE Salary > 70000;

```

*(Notice we don't use the `VALUES` keyword here. The subquery entirely replaces it).*

#### B. Subqueries with UPDATE

**When to apply:** You need to modify existing records, but the new value—or the rule deciding *who* gets updated—lives in another table. You can use a subquery in the `SET` clause (to calculate the new value) or the `WHERE` clause (to filter who gets updated).

* **The Logic (WHERE clause):** "Give a 10% raise to everyone in the 'Sales' department." You cannot update the `Employees` table directly with 'Sales' because the `Employees` table only has `Dept_ID`. Your "Unknown Prerequisite" is the `Dept_ID` for Sales.
* **Scenario:** Increase the salary of employees in the Sales department by 10%.
* **Query:**

```sql
UPDATE Employees
SET Salary = Salary * 1.10
WHERE Dept_ID = (
    SELECT Dept_ID 
    FROM Departments 
    WHERE Dept_Name = 'Sales'
);

```

#### C. Subqueries with DELETE

**When to apply:** You need to delete records, but the criteria for deletion depends on data stored elsewhere.

* **The Logic:** "Delete employees who work in a specific department, but I only know the department's name, not its ID."
* **Scenario:** Delete any employee who works in the 'Marketing' department.
* **Query:**

```sql
DELETE FROM Employees
WHERE Dept_ID = (
    SELECT Dept_ID 
    FROM Departments 
    WHERE Dept_Name = 'Marketing'
);

```

#### D. Subqueries within SELECT (A deeper look)

While we have used subqueries in the `WHERE` clause of a `SELECT` statement, you can also place them directly in the `SELECT` clause (this is called a **Scalar Subquery**).

* **The Logic:** "For every employee I list, I also want to display a single, calculated value right next to their name, like the company's average salary."
* **Scenario:** Show every employee's name, their salary, and the overall company average salary next to it for direct comparison.
* **Query:**

```sql
SELECT 
    First_Name, 
    Salary, 
    (SELECT AVG(Salary) FROM Employees) AS Company_Avg
FROM Employees;

```

### Single Row Subquery

**What is it?**
A single-row subquery is a query that returns exactly **one row** (or zero rows) to the outer query.

**Wait, isn't that just a Scalar Subquery?**
They are very closely related, but there is a subtle and powerful distinction in Oracle SQL:

* A **Scalar Subquery** strictly returns 1 row *and* 1 column (a single isolated value, like just a salary).
* A **Single-Row Subquery** strictly returns 1 row, but it can return **multiple columns** at once for that specific row.

**Strategic Mapping (How to think about it):**
Use this when your "Unknown Prerequisite" revolves around a single specific entity (like finding the specific traits of *one* employee, *one* department, or *one* exact date). Because it guarantees only one row is returned, you **must use single-row operators** (`=`, `>`, `<`, `>=`, `<=`, `<>`).

**The Infamous Error (ORA-01427):**
If you use a single-row operator (like `=`) but your inner query accidentally finds two rows (e.g., you searched by name, but there are two people named 'Alice' in the company), Oracle will immediately throw an error: `ORA-01427: single-row subquery returns more than one row`.

#### Scenario A: Single Column, Single Row

**The Logic:** "I need to find everyone who works in the exact same department as Alice. My unknown is Alice's department."
**The Query:**

```sql
SELECT First_Name, Salary 
FROM Employees 
WHERE Dept_ID = (
    -- This inner query returns 1 row and 1 column (Scalar/Single-Row)
    SELECT Dept_ID 
    FROM Employees 
    WHERE First_Name = 'Alice'
) 
AND First_Name <> 'Alice'; -- (To exclude Alice herself from the results)

```

#### Scenario B: Multiple Columns, Single Row (Oracle Specific Power-Move)

**The Logic:** "I need to find employees who have the *exact same* Salary AND Department as Eve." Instead of writing two separate subqueries (one for salary, one for department), Oracle lets you compare multiple columns simultaneously.
**The Query:**

```sql
SELECT First_Name, Salary, Dept_ID
FROM Employees
WHERE (Salary, Dept_ID) = (
    -- This inner query returns 1 row, but 2 columns
    SELECT Salary, Dept_ID 
    FROM Employees 
    WHERE First_Name = 'Eve'
)
AND First_Name <> 'Eve';

```

*(Notice how the outer `WHERE` clause brackets `(Salary, Dept_ID)` match perfectly with the two columns returned by the inner `SELECT`).*

### Multiple Row Subquery

**What is it?**
A multiple-row subquery is exactly what it sounds like: a query that returns a list (one or more rows) back to the outer query.

**Strategic Mapping (How to think about it):**
Use this when your "Unknown Prerequisite" yields a *set of possibilities* rather than a single, isolated fact.

* *Single-Row thought:* "What is the ID for the HR department?" (One answer).
* *Multiple-Row thought:* "What are the IDs for all departments that currently have active employees?" (Many answers).

**The Danger Zone (Why we need new operators):**
If your inner query returns a list (e.g., `10, 20, 30`), you cannot logically ask SQL if a department ID *equals* `10, 20, 30`. An ID can only equal one thing at a time. If you try to use an `=` sign here, Oracle will throw the classic error we mentioned earlier: `ORA-01427: single-row subquery returns more than one row`.

To fix this, you must abandon single-row operators (`=`, `>`, `<`) and upgrade to **Multiple-Row Operators**. The most common one is `IN`.

**Scenario:**
You need a list of employees, but only those who work in the 'Engineering' or 'Sales' departments.

* *The Unknown:* You know the names of the departments, but the `Employees` table only stores the `Dept_ID`. You need to look up those IDs first.

**The Query:**

```sql
SELECT First_Name, Salary, Dept_ID
FROM Employees
WHERE Dept_ID IN (
    -- Inner query returns a list: 10, 20
    SELECT Dept_ID 
    FROM Departments 
    WHERE Dept_Name IN ('Engineering', 'Sales')
);

```

**Logical Breakdown:**

1. SQL runs the inner query. It finds that Engineering is 10 and Sales is 20.
2. The inner query essentially transforms into a list: `(10, 20)`.
3. The outer query runs: `SELECT ... WHERE Dept_ID IN (10, 20);`

### Usage of IN, NOT IN, ALL, ANY, and SOME

When your subquery returns a list of values (a Multiple-Row Subquery), you must use these specific operators to bridge the outer query and the inner query.

**Strategic Mapping (How to think about it):**
Imagine your inner query returns a bucket of numbers: `[50000, 60000, 75000]`. How do you want to compare a specific employee's salary to that bucket?

#### 1. IN and NOT IN

* **`IN` Logic:** "Does my value exist anywhere inside this bucket?" (Equivalent to `= ANY`).
* **`NOT IN` Logic:** "Is my value completely absent from this bucket?"

**Scenario (NOT IN):** Find employees who do *not* work in the 'HR' or 'Marketing' departments.

```sql
SELECT First_Name, Dept_ID 
FROM Employees 
WHERE Dept_ID NOT IN (
    SELECT Dept_ID 
    FROM Departments 
    WHERE Dept_Name IN ('HR', 'Marketing')
);

```

> **The Deadly NULL Trap (Crucial for Interviews):**
> If your subquery returns a list that contains even a single `NULL` value, a `NOT IN` condition will evaluate to FALSE for every row, returning zero results. Always ensure your inner query filters out NULLs (e.g., `WHERE column IS NOT NULL`) if you are using `NOT IN`.

#### 2. ANY (and SOME)

* **Logic:** "Is my condition true compared to *at least one* value in the bucket?"
* **Note:** `SOME` is simply a grammatical synonym for `ANY` in Oracle. They do exactly the same thing.

**Strategic Breakdown of `ANY`:**
If your subquery returns salaries `[50000, 60000, 75000]`:

* `> ANY` means "Greater than the lowest value." (If you are greater than 50000, you are greater than *any* of them).
* `< ANY` means "Less than the highest value." (If you are less than 75000, you are less than *any* of them).

**Scenario (`< ANY`):** Find employees whose salary is less than the maximum salary in the Engineering department (Dept 10), but exclude Engineering employees from the result.

```sql
SELECT First_Name, Salary 
FROM Employees 
WHERE Dept_ID <> 10
AND Salary < ANY (
    SELECT Salary 
    FROM Employees 
    WHERE Dept_ID = 10
);

```

*(Engineering salaries are 90k, 75k, 95k. The query looks for non-engineers earning less than 95k).*

#### 3. ALL

* **Logic:** "Is my condition true compared to *every single* value in the bucket?"

**Strategic Breakdown of `ALL`:**
If your subquery returns salaries `[50000, 60000, 75000]`:

* `> ALL` means "Greater than the highest value." (You must beat the top number to beat them *all*).
* `< ALL` means "Less than the lowest value." (You must be under the bottom number to be less than them *all*).

**Scenario (`> ALL`):** Find employees who earn more than *every single* employee in the Sales department (Dept 20).

```sql
SELECT First_Name, Salary 
FROM Employees 
WHERE Salary > ALL (
    SELECT Salary 
    FROM Employees 
    WHERE Dept_ID = 20
);

```

*(Sales salaries are 60k and 80k. The query will only return employees earning more than 80,000).*

### Correlated Subqueries

This is where SQL transitions from simple data retrieval into dynamic, row-by-row programming logic. Understanding this concept is a major milestone in mastering SQL.

**What is it?**
A correlated subquery is a subquery that **depends on the outer query** for its values. Unlike the independent subqueries we just covered (which run once, get a result, and hand it off), a correlated subquery executes repeatedly—once for *every single row* evaluated by the outer query.

**Strategic Mapping (How to think about it):**
Think of it exactly like a `FOR EACH` loop in programming.

* *Independent Subquery thought:* "Find the company's maximum salary, then compare everyone to it." (Runs once).
* *Correlated Subquery thought:* "Look at Employee A. Take their specific Department ID. Go find the average salary for *that specific* department. Compare Employee A's salary to it. Now move to Employee B and repeat the whole process." (Runs for every row).

**The Golden Rule of Syntax:**
To make a subquery correlated, the inner query **must reference a column from the outer query**. To do this without confusing the database, you *must* use table aliases (e.g., `e1`, `e2`).

**Scenario:**
Find all employees who earn more than the average salary of their *own* specific department.

* *The Unknown:* The average salary isn't a single static number; it changes depending on which employee the outer query is currently looking at.

**The Query:**

```sql
SELECT e1.First_Name, e1.Salary, e1.Dept_ID
FROM Employees e1 -- (Outer query table aliased as e1)
WHERE e1.Salary > (
    -- Inner query calculates the average for the CURRENT row's department
    SELECT AVG(e2.Salary) 
    FROM Employees e2 
    WHERE e2.Dept_ID = e1.Dept_ID -- THIS is the correlation!
);

```

**Logical Breakdown (How the engine processes it):**

1. **Row 1 (Alice):** The outer query looks at Alice (Salary: 90000, Dept: 10).
2. **Inner Query Execution:** The inner query takes Alice's Dept 10 and runs: `SELECT AVG(Salary) FROM Employees WHERE Dept_ID = 10`. The result is 86,666.
3. **Comparison:** Is Alice's 90,000 > 86,666? Yes. Alice is kept in the final results.
4. **Row 2 (Bob):** The outer query moves to Bob (Salary: 60000, Dept: 20).
5. **Inner Query Execution:** The inner query takes Bob's Dept 20 and runs: `SELECT AVG(Salary) FROM Employees WHERE Dept_ID = 20`. The result is 70,000.
6. **Comparison:** Is Bob's 60,000 > 70,000? No. Bob is filtered out.
7. *(This process repeats for every single employee).*

### Usage of EXISTS, NOT EXISTS

The `EXISTS` and `NOT EXISTS` operators are almost always used in conjunction with correlated subqueries. They are considered some of the most powerful and highly optimized tools in an SQL developer's arsenal.

**What is it?**
Instead of returning actual data (like a list of IDs or a specific salary), `EXISTS` simply returns a boolean value: **TRUE or FALSE**. It checks whether the inner query returns *at least one row*.

**Strategic Mapping (How to think about it):**
Think of `EXISTS` as a radar sweep that is only looking for a "Yes" or "No".

* *The `IN` thought process:* "Go get the IDs of every single person in the Engineering department, build a list of them, and then check if my current employee's ID is in that list."
* *The `EXISTS` thought process:* "Knock on the door of the Engineering department. The moment anyone answers, report back TRUE and stop looking."

**The Performance Superpower (Short-Circuiting):**
Because `EXISTS` only cares *if* a row exists, the database engine stops searching the inner query the absolute millisecond it finds the first match. It doesn't waste time reading the rest of the table.

**The `SELECT 1` Convention:**
Because the outer query doesn't care *what* data the inner query finds—only *if* it finds something—it is a global best practice to write `SELECT 1` inside the `EXISTS` subquery instead of a column name.

#### Scenario A: EXISTS

**The Logic:** Find the details of all departments that currently have at least one employee earning more than 85,000.

* *The Unknown:* Which departments meet this criteria?

**The Query:**

```sql
SELECT Dept_Name 
FROM Departments d
WHERE EXISTS (
    -- The radar sweep: Look at the current department 'd'. 
    -- Does anyone here make > 85000?
    SELECT 1 
    FROM Employees e 
    WHERE e.Dept_ID = d.Dept_ID  -- The correlation
    AND e.Salary > 85000
);

```

*(As soon as it finds Alice making 90,000 in Dept 10, it immediately marks Dept 10 as TRUE and stops checking the other engineers).*

#### Scenario B: NOT EXISTS

**The Logic:** Find the "orphan" records or empty categories. For example, find any department that currently has absolutely zero employees.

* *The Unknown:* Which department IDs do not appear in the employee table?

**The Query:**

```sql
SELECT Dept_Name 
FROM Departments d
WHERE NOT EXISTS (
    SELECT 1 
    FROM Employees e 
    WHERE e.Dept_ID = d.Dept_ID
);

```

*(In our setup data, this will return 'Marketing', because Dept_ID 40 does not exist anywhere in the Employees table).*

### Why use EXISTS instead of IN?

While you could solve the above scenarios using `IN` or `NOT IN`, `EXISTS` is safer and often faster. Remember the "Deadly NULL Trap" we discussed in step 9? `NOT IN` will fail completely if it encounters a `NULL` value in the subquery. `NOT EXISTS` is immune to the NULL trap, making it the industry standard for finding missing data between two tables.

### Difference between Correlated & Non-Correlated Subquery

You have already learned how to write both types, but taking a moment to clearly contrast them is crucial for both job interviews and deciding which one to use when you are staring at a blank screen.

Here is the ultimate cheat sheet to distinguish between the two:

| Feature | Non-Correlated Subquery | Correlated Subquery |
| --- | --- | --- |
| **Dependency** | **Independent:** The inner query can be highlighted, run on its own, and it will return a valid result. | **Dependent:** The inner query will throw an error if run on its own because it is missing data from the outer query. |
| **Execution Frequency** | Executes exactly **once** before the main query starts. | Executes **repeatedly**—once for every single row the outer query evaluates. |
| **Execution Order** | **Inner first, then Outer.** (Calculates the answer, hands it off). | **Outer first, then Inner.** (Picks a row, calculates the answer, repeats). |
| **Performance** | Generally faster. The database can run it once, cache the result in memory, and reuse it instantly. | Can be slower on massive datasets due to the row-by-row looping, though modern SQL optimizers handle `EXISTS` very efficiently. |
| **Syntax Marker** | Does not reference any tables from the outer query. | **Must** reference a column from the outer query (usually using table aliases like `e1.Dept_ID`). |

**The Strategic Rule of Thumb:**

* If your requirement sounds like it needs a **global or static metric** (e.g., "the company average," "the highest earner overall," "a specific ID list"), use a **Non-Correlated Subquery**.
* If your requirement sounds like it needs a **dynamic, tailored metric** (e.g., "their *own* department's average," "has *any* matching records in another table"), use a **Correlated Subquery**.

---

