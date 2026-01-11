# SQL Senior Developer Master Interview Guide

## Questions Covered in this guide

### **Theory Questions**

1.  DBMS vs RDBMS vs NoSQL

2.  SQL Order of Execution

3.  Normalization (1NF to BCNF) & Trade-offs

4.  ACID vs BASE Properties

5.  Clustered vs Non-Clustered Indexes

6.  Composite Indexes & Left-Prefix Rule

7.  Covering Indexes

8.  Index Scans vs Index Seeks

9.  Database Sharding vs Partitioning

10.  Materialized Views vs Standard Views

11.  Stored Procedures vs Functions

12.  Deadlocks and Prevention

13.  Isolation Levels (Read Uncommitted to Serializable)

14.  Optimistic vs Pessimistic Locking

15.  Triggers: Pros and Cons

16.  CTEs (Recursive vs Non-recursive)

17.  Window Functions vs Group By

18.  Cursor Performance Issues

19.  SQL Injection & Prevention

20.  Explain Plan Analysis

21.  N+1 Query Problem

22.  Temporary Tables vs Table Variables vs CTEs

23.  Cross Apply vs Outer Apply

24.  Data Warehousing: Star vs Snowflake Schema

25.  OLTP vs OLAP

26.  Constraints and Business Logic

27.  Horizontal vs Vertical Scaling

28.  Soft Deletes vs Hard Deletes

29.  Database Migrations & Versioning

30.  CAP Theorem in Databases

31.  Exception Handling (TRY...CATCH)

32.  What is Collation?
    
33.  Composite Key vs. Surrogate Key
    
34.  Varchar vs NVarchar
    
35.  COMMIT and ROLLBACK
    

### **Practical Questions**

1.  Finding the Second Highest Salary

2.  Conditional Aggregation (Manual Pivot)

3.  Grouping Sets and ROLLUP

4.  Filtering on Aggregates (HAVING)

5.  Count Distinct Logic

6.  Identifying Missing Data (Anti-Joins)

7.  ROW_NUMBER vs RANK vs DENSE_RANK

8.  Top N Records per Category

9.  Delete Duplicate Rows (CTE vs Group By)

10.  Running Totals (Cumulative Sums)

11.  Lead and Lag (Time-series)

12.  NTILE for Percentile Distribution

13.  FIRST_VALUE and LAST_VALUE

14.  Moving Averages (Sliding Windows)

15.  Pivot: Occupations Problem

16.  Unpivoting Data

17.  Percentage of Total (Ratio Analysis)
    
18.  Gaps and Islands (Consecutive Days)

19.  Partitioning by Multiple Columns

20.  Window Frames: ROWS vs RANGE

21.  String Masking for Security

22.  Last Day of Month Calculations

23.  NULL Logic in Comparisons

24.  Recursive CTE for Date Generation

25.  Handling Delimited Strings (String Splitting)

26.  Finding the Statistical Mode

27.  Performance: EXISTS vs IN

28.  Business Day Logic (Excluding Weekends)

29.  Update from Join

30.  Query Refactoring (SARGability)


----------

## Part 1: SQL Theory

### 1. What is the difference between DBMS, RDBMS, and NoSQL?

**Answer:** DBMS is a general system for data storage. RDBMS (Relational) uses structured tables with fixed schemas and SQL (e.g., PostgreSQL, SQL Server). NoSQL (Non-relational) offers flexible schemas (Document, Key-Value, Graph) and scales horizontally more easily (e.g., MongoDB, Cassandra).

### 2. What is the SQL Order of Execution?

**Answer:** The logical order is different from the written order:

1.  `FROM/JOIN` 2. `WHERE` 3. `GROUP BY` 4. `HAVING` 5. `SELECT` 6. `DISTINCT` 7. `ORDER BY` 8. `LIMIT/OFFSET`. Knowing this helps in troubleshooting why an alias created in `SELECT` cannot be used in a `WHERE` clause.
    

### 3. Normalization (1NF to BCNF) & Trade-offs

**Answer:** Normalization reduces redundancy. 1NF (Atomicity), 2NF (No partial dependencies), 3NF (No transitive dependencies), and BCNF (Stronger 3NF). **Senior Note:** Over-normalization can lead to "Join Hell." Strategic Denormalization is often used in read-heavy systems to improve performance.

### 4. ACID vs BASE Properties

**Answer:** **ACID** (Atomicity, Consistency, Isolation, Durability) ensures strict reliability, common in RDBMS. **BASE** (Basically Available, Soft state, Eventual consistency) is used in NoSQL systems like Cassandra to achieve massive high availability.

### 5. Clustered vs Non-Clustered Indexes

**Answer:** A **Clustered Index** defines the physical storage order of data (only one per table). A **Non-Clustered Index** is a separate structure (like a book index) containing pointers to the data (multiple allowed).

### 6. Composite Indexes & Left-Prefix Rule

**Answer:** A composite index is on multiple columns. The "Left-Prefix Rule" means the index is only effective if the query filters include the _first_ column in the index. An index on `(Lastname, Firstname)` helps queries on `Lastname` but not on `Firstname` alone.

### 7. What is a Covering Index?

**Answer:** An index that contains all the columns required by a query (in the `SELECT`, `JOIN`, and `WHERE` clauses). This allows the engine to return results solely from the index without a "Bookmarked Lookup" to the actual table.

### 8. Index Scans vs Index Seeks

**Answer:** An **Index Seek** is efficient; the engine navigates the B-tree to find specific rows. An **Index Scan** is less efficient; the engine reads the entire index structure because it cannot pinpoint the exact location.

### 9. Database Sharding vs Partitioning

**Answer:** **Partitioning** (Vertical/Horizontal) happens within a single database instance (e.g., splitting a table by Year). **Sharding** distributes data across multiple independent physical machines.

### 10. Materialized Views vs Standard Views

**Answer:** A Standard View is just a saved query. A **Materialized View** physically stores the result on disk and must be refreshed. It is used in OLAP to speed up massive aggregations.

### 11. Stored Procedures vs Functions

**Answer:** Procedures can perform DML (Insert/Update) and don't need to return a value. Functions must return a value, cannot perform DML on permanent tables, and can be used directly in `SELECT` statements.

### 12. Deadlocks and Prevention

**Answer:** A deadlock occurs when two transactions hold locks and wait for each other. Prevention includes accessing tables in the same order across the app, keeping transactions short, and using the `NOLOCK` hint (carefully).

### 13. Isolation Levels (Read Uncommitted to Serializable)

**Answer:**

-   **Read Uncommitted:** Allows dirty reads.
    
-   **Read Committed:** Prevents dirty reads (Default).
    
-   **Repeatable Read:** Prevents non-repeatable reads.
    
-   **Serializable:** Prevents phantom reads (highest locking).
    

### 14. Optimistic vs Pessimistic Locking

**Answer:** **Pessimistic** locks data the moment it's read. **Optimistic** checks for changes only at commit time (usually via a `Version` or `RowVersion` column). Optimistic is better for high-concurrency web apps.

### 15. Triggers: Pros and Cons

**Answer:** **Pros:** Automated auditing, complex integrity. **Cons:** They are "hidden" logic that can cause performance overhead and side effects that are difficult to debug in a large codebase.

### 16. CTEs (Recursive vs Non-recursive)

**Answer:** CTEs improve readability. Recursive CTEs are essential for querying hierarchical data, such as an Organization Chart or a folder structure.

### 17. Window Functions vs Group By

**Answer:** `GROUP BY` collapses rows into a single summary. Window functions (`OVER`) perform calculations across a set of rows while keeping the individual row data intact.

### 18. Cursor Performance Issues

**Answer:** Cursors are "row-by-row" (procedural). SQL is optimized for "set-based" operations. Cursors use significant memory and should be the last resort after trying JOINs or CTEs.

### 19. SQL Injection & Prevention

**Answer:** It involves injecting malicious SQL code via user input. Prevention: Use **Parameterized Queries/Prepared Statements**, and never use string concatenation for building queries.

### 20. Explain Plan Analysis

**Answer:** It is a roadmap showing how the DB engine executes a query. Seniors look for "Table Scans" (bad) versus "Index Seeks" (good) and high-cost "Hash Joins."

### 21. N+1 Query Problem

**Answer:** Occurs when an ORM (like Hibernate or Entity Framework) fetches a list of 100 items and then executes 100 separate queries to get details for each. Solved by using `JOIN` or Eager Loading.

### 22. Temporary Tables vs Table Variables vs CTEs

**Answer:** * **Temp Tables (#T):** In `tempdb`, support indexes, best for large data.

-   **Table Variables (@T):** In-memory, no indexes, best for very small sets.
    
-   **CTEs:** Non-persistent, purely for query structure.
    

### 23. Cross Apply vs Outer Apply

**Answer:** These are used to join a table to a table-valued function or subquery that requires a column from the outer table as an input. `CROSS APPLY` is like `INNER JOIN`; `OUTER APPLY` is like `LEFT JOIN`.

### 24. Data Warehousing: Star vs Snowflake Schema

**Answer:** **Star:** A central Fact table with denormalized Dimension tables (optimized for speed). **Snowflake:** Dimensions are normalized (optimized for storage).

![Image of Star Schema vs Snowflake Schema](https://encrypted-tbn1.gstatic.com/licensed-image?q=tbn:ANd9GcQfvMUbmzjGXf9sp-QxMApYCblMnpQd1yjSOcXxBX-MUX30P18whvXkI3Qts8RNk74Ko0YsuvgtSksTj5Js3oJ1-7WI2wt_BEJaZOfuvvr27B9UWyA)

### 25. OLTP vs OLAP

**Answer:** **OLTP** is for real-time transactions (Insert/Update). **OLAP** is for historical data analysis (Complex SELECTs).

### 26. Constraints and Business Logic

**Answer:** Senior developers argue for placing critical integrity logic (`FK`, `CHECK`, `NOT NULL`) in the DB to ensure data safety, even if the application code has bugs.

### 27. Horizontal vs Vertical Scaling

**Answer:** **Vertical:** Adding more RAM/CPU to one machine. **Horizontal:** Adding more machines (Read Replicas or Sharding).

### 28. Soft Deletes vs Hard Deletes

**Answer:** **Hard Delete:** `DELETE FROM...`. **Soft Delete:** `UPDATE SET IsDeleted = 1`. Soft deletes are safer for auditing but require all queries to include `WHERE IsDeleted = 0`.

### 29. Database Migrations & Versioning

**Answer:** Using tools like Flyway or Liquibase to treat database schema changes like code, allowing for version control and automated deployments.

### 30. CAP Theorem in Databases

**Answer:** A distributed system can only provide two of three: **Consistency, Availability, and Partition Tolerance**. Traditional RDBMS usually sacrifice Availability in the face of a network partition to ensure strict Consistency.

![Image of SQL Joins Venn Diagram](https://encrypted-tbn2.gstatic.com/licensed-image?q=tbn:ANd9GcRXiakg1OfcQE8843MV8D0UYIef_xThubMMzjzF9lRfEJqgINn0VuE43TDX3sSt9KEMnpXm3jzdpI4qncZI44RMyjMTD9vp4L3MHBf1g9T2WUSMV98)

Shutterstock

Explore

### 31. How is Exception Handling performed in SQL Server?

**Answer:** SQL Server uses the `BEGIN TRY...END TRY` and `BEGIN CATCH...END CATCH` blocks.

-   **TRY block:** Contains the code that might fail.
    
-   **CATCH block:** Contains the logic to handle the error (e.g., logging or rolling back a transaction). **Senior Note:** Within the CATCH block, functions like `ERROR_NUMBER()`, `ERROR_MESSAGE()`, and `ERROR_SEVERITY()` are used to diagnose the issue.
    

### 32. What is Collation?

**Answer:** Collation refers to a set of rules that determine how data is sorted and compared. This includes rules for case sensitivity (`_CS` vs `_CI`) and accent sensitivity (`_AS` vs `_AI`). **Senior Note:** Incorrect collation settings can lead to "Collation Mismatch" errors when joining tables from different databases.

### 33. Composite Key vs. Surrogate Key

**Answer:** * **Composite Key:** A primary key made up of two or more columns to ensure uniqueness (e.g., `OrderID` + `ProductID`).

-   **Surrogate Key:** An artificially generated unique identifier (like an `IDENTITY` or `UUID`) that has no business meaning but serves as a primary key.
    

### 34. What is the difference between NVarchar and Varchar?

**Answer:** `Varchar` stores ASCII (1 byte per char); `NVarchar` stores Unicode (2 bytes per char). Use `NVarchar` for internationalization support.

### 35. Can we update a View?

**Answer:** Yes, if the view references only one base table and contains no aggregates, `DISTINCT`, or `GROUP BY` clauses.

----------

## Part 2: Practical Questions

### 1. The "Second Highest" Salary Trick

**Question:** Find the second-highest salary without using `TOP` or `LIMIT`.

SQL

```
SELECT MAX(Salary) FROM Employees 
WHERE Salary < (SELECT MAX(Salary) FROM Employees);

```

### 2. Conditional Aggregation (Pivot Simulation)

**Question:** Show total sales for 2023 and 2024 as separate columns for each product.

SQL

```
SELECT ProductID, 
    SUM(CASE WHEN YEAR(SaleDate) = 2023 THEN Amount ELSE 0 END) AS Sales_2023,
    SUM(CASE WHEN YEAR(SaleDate) = 2024 THEN Amount ELSE 0 END) AS Sales_2024
FROM Sales GROUP BY ProductID;

```

### 3. Grouping Sets (Rollup)

**Question:** Show sales by Region, City, and a Grand Total in one result set.

SQL

```
SELECT Region, City, SUM(Sales)
FROM SalesData
GROUP BY ROLLUP(Region, City);

```

### 4. Filtering on Aggregates (Having)

**Question:** Find departments where the average salary is above 80k, but only for departments with more than 5 employees.

SQL

```
SELECT DeptID FROM Employees
GROUP BY DeptID
HAVING AVG(Salary) > 80000 AND COUNT(*) > 5;

```

### 5. Count Distinct Logic

**Question:** How many unique customers made a purchase in the last 7 days?

SQL

```
SELECT COUNT(DISTINCT CustomerID) 
FROM Orders 
WHERE OrderDate >= DATEADD(day, -7, GETDATE());

```

### 6. The "Missing Data" Aggregate

**Question:** List categories that have no products.

SQL

```
SELECT C.CategoryName
FROM Categories C
LEFT JOIN Products P ON C.ID = P.CategoryID
GROUP BY C.CategoryName
HAVING COUNT(P.ID) = 0;

```

### 7. ROW_NUMBER vs RANK vs DENSE_RANK

**Question:** Demonstrate the difference using a list of scores.

SQL

```
SELECT Score,
    ROW_NUMBER() OVER(ORDER BY Score DESC) as 'Row',
    RANK() OVER(ORDER BY Score DESC) as 'Rank',
    DENSE_RANK() OVER(ORDER BY Score DESC) as 'Dense'
FROM ExamScores;
-- Rank skips numbers (1, 2, 2, 4); Dense Rank doesn't (1, 2, 2, 3).

```

### 8. Top N per Category (Partition By)

**Question:** Find the most recent order for every customer.

SQL

```
SELECT * FROM (
    SELECT CustomerID, OrderID, OrderDate,
    ROW_NUMBER() OVER(PARTITION BY CustomerID ORDER BY OrderDate DESC) as rnk
    FROM Orders
) t WHERE rnk = 1;

```

### 9. How to Delete Duplicate Rows? (Method Comparison)

**Task:** Remove duplicates while keeping the original. **CTE Method (Best Practice):**

SQL

```
WITH DuplicateCTE AS (
    SELECT Email, ROW_NUMBER() OVER(PARTITION BY Email ORDER BY ID) as RowNum
    FROM Users
)
DELETE FROM DuplicateCTE WHERE RowNum > 1;

```

### 15. Pivot: Occupations Problem (Dynamic Rows)

**Task:** Sort names alphabetically under their Occupation.

SQL

```
SELECT 
    MAX(CASE WHEN Occupation = 'Doctor' THEN Name END) AS Doctor,
    MAX(CASE WHEN Occupation = 'Professor' THEN Name END) AS Professor,
    MAX(CASE WHEN Occupation = 'Singer' THEN Name END) AS Singer,
    MAX(CASE WHEN Occupation = 'Actor' THEN Name END) AS Actor
FROM (
    SELECT Name, Occupation, 
           ROW_NUMBER() OVER(PARTITION BY Occupation ORDER BY Name) as row_num
    FROM OCCUPATIONS
) AS temp
GROUP BY row_num;

```

### 23. Handling NULLs in Comparisons

**Task:** Select all products where the color is NOT 'Blue'.

SQL

```
-- TRICK: Standard <> will ignore NULLs.
SELECT * FROM Products 
WHERE Color <> 'Blue' OR Color IS NULL;

```

### 25. Handling Delimited Strings (String Splitting)

**Task:** Use `STRING_SPLIT` to count tag occurrences.

SQL

```
SELECT value as Tag, COUNT(*) 
FROM Articles 
CROSS APPLY STRING_SPLIT(Tags, ',')
GROUP BY value;

```

### 29. Update from Join

**Task:** Increase salary for employees in the 'Engineering' department.

SQL

```
UPDATE E
SET E.Salary = E.Salary * 1.15
FROM Employees E
JOIN Departments D ON E.DeptID = D.DeptID
WHERE D.DeptName = 'Engineering';

```

----------

## Part 3: SQL Tuning & Performance

### The Query Tuning Checklist

1.  **Check Execution Plan:** Identify "Scans" vs "Seeks."
    
2.  **Cardinality:** Verify if statistics are up to date.
    
3.  **SARGability:** Ensure indexed columns aren't wrapped in functions (e.g., `WHERE YEAR(date) = 2024`).
    
4.  **Key Lookups:** Ensure indexes are "Covering" by using the `INCLUDE` clause.
