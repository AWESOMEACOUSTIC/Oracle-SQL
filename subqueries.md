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

