### Case Study: The End-of-Year Review

You have reached the final stage. To truly master subqueries, you must be able to read a complex business requirement, identify the "Unknown Prerequisites," and stitch different types of subqueries together.

Let's expand our company database by adding a `Projects` table. Run this script to set the stage:

```sql
CREATE TABLE Projects (
    Project_ID INT PRIMARY KEY,
    Project_Name VARCHAR(50),
    Dept_ID INT,
    Budget DECIMAL(10, 2)
);

INSERT INTO Projects (Project_ID, Project_Name, Dept_ID, Budget) VALUES 
(1, 'Cloud Migration', 10, 150000),
(2, 'Q4 Ad Campaign', 40, 80000),
(3, 'B2B Sales Portal', 20, 200000),
(4, 'Security Audit', 10, 50000);

```

---

### The Business Requirements

The executive team has handed you two critical tasks for the end-of-year review.

#### Task 1: The Analytical Report (SELECT)

**The Request:** "We need a report showing the First Name and Salary of our employees. Next to their salary, we want to see exactly how far behind they are from the absolute highest salary in the company. However, we only want to see employees if their department is currently managing a project with a budget strictly greater than $100,000."

**The Strategic Breakdown (Logical Reasoning):**
Before writing any code, break this down into components:

1. **The Base:** I need `First_Name` and `Salary` from the `Employees` table.
2. **Unknown 1 (The Gap):** How do I calculate the distance from the highest salary? I need to find the maximum salary in the company first. This is a single, static value.
* *Strategy:* Use a **Scalar Subquery** in the `SELECT` clause to find the `MAX(Salary)`, and subtract the employee's current salary from it.


3. **Unknown 2 (The Filter):** How do I know if an employee's department has a high-budget project? I need to check the `Projects` table.
* *Strategy:* Use an `IN` operator with a **Multiple-Row Subquery** (or `EXISTS` with a Correlated Subquery) in the `WHERE` clause to find all `Dept_ID`s associated with budgets > 100,000.



**The Solution:**

```sql
SELECT 
    First_Name, 
    Salary, 
    -- Scalar Subquery doing math against the current row
    (SELECT MAX(Salary) FROM Employees) - Salary AS Distance_From_Max
FROM Employees
WHERE Dept_ID IN (
    -- Multiple-Row Subquery finding departments with big budgets
    SELECT Dept_ID 
    FROM Projects 
    WHERE Budget > 100000
);

```

#### Task 2: The Cleanup (DELETE)

**The Request:** "We are restructuring. We need to delete any department from the `Departments` table that currently has absolutely zero employees assigned to it."

**The Strategic Breakdown (Logical Reasoning):**

1. **The Base:** I am deleting from the `Departments` table.
2. **Unknown (The Filter):** Which departments are empty? I need to check if a department's ID is entirely absent from the `Employees` table.
3. *Strategy:* This is the classic "find the missing data" scenario. A **Correlated Subquery** using `NOT EXISTS` is the safest and most professional way to handle this, as it completely avoids the deadly NULL trap.

**The Solution:**

```sql
DELETE FROM Departments d
WHERE NOT EXISTS (
    -- Correlated Subquery checking for any pulse in the Employees table
    SELECT 1 
    FROM Employees e 
    WHERE e.Dept_ID = d.Dept_ID
);

```
