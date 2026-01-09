# SQL Senior Developer Interview Guide

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


### **Practical Questions**

1.  The "Second Highest" Salary Trick
2.  Conditional Aggregation (Pivot Simulation)
3.  Grouping Sets (Rollup)
4.  Filtering on Aggregates (Having)
5.  Count Distinct Logic
6.  The "Missing Data" Aggregate
7.  ROW_NUMBER vs RANK vs DENSE_RANK
8.  Top N per Category (Partition By)
9.  Identifying Duplicates with Windows
10.  Running Totals (Cumulative Sum)
11.  Lead and Lag (Year-over-Year)
12.  Percentile Calculation (NTILE)
13.  First/Last Value Functions
14.  Moving Averages (Window Frame)
15.  Basic Pivot (Rows to Columns)
16.  Unpivoting (Columns to Rows)
17.  Percentage of Total (Partition By)
18.  Finding Consecutive Days (Gaps and Islands)
19.  Partition by Multiple Columns
20.  Difference between ROWS and RANGE
21.  Email Masking (String Manipulation)
22.  Finding the Last Day of Month
23.  Handling NULLs in Comparisons
24.  Recursive CTE for Dates
25.  String Splitting (Cross Apply)
26.  Finding the Mode (Common Record)
27.  The Anti-Join (Not Exists)
28.  Business Days Calculation Logic
29.  Update from Join
30.  Delete Duplicate Rows (The CTE Method)

----------

## Part 1: Theory Questions

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

### 9. Identifying Duplicates with Windows

**Question:** Find all duplicate emails in a `Users` table.

SQL

```
SELECT Email FROM (
    SELECT Email, ROW_NUMBER() OVER(PARTITION BY Email ORDER BY UserID) as cnt
    FROM Users
) t WHERE cnt > 1;

```

### 10. Running Totals (Cumulative Sum)

**Question:** Calculate a running total of revenue by date.

SQL

```
SELECT SaleDate, Revenue,
    SUM(Revenue) OVER(ORDER BY SaleDate ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as CumulativeRevenue
FROM Sales;

```

### 11. Lead and Lag (Year-over-Year)

**Question:** Show the percentage growth of sales compared to the previous month.

SQL

```
SELECT Month, Sales,
    ((Sales - LAG(Sales) OVER(ORDER BY Month)) / LAG(Sales) OVER(ORDER BY Month)) * 100 as Growth
FROM MonthlySales;

```

### 12. Percentile Calculation (NTILE)

**Question:** Divide employees into 4 salary quartiles.

SQL

```
SELECT Name, Salary,
    NTILE(4) OVER(ORDER BY Salary DESC) as Quartile
FROM Employees;

```

### 13. First/Last Value Functions

**Question:** For each product, show the current price and the price it launched with.

SQL

```
SELECT ProductName, Price,
    FIRST_VALUE(Price) OVER(PARTITION BY ProductID ORDER BY EffectiveDate) as LaunchPrice
FROM PriceHistory;

```

### 14. Moving Averages (Window Frame)

**Question:** Calculate a 7-day moving average of website traffic.

SQL

```
SELECT LogDate, Visitors,
    AVG(Visitors) OVER(ORDER BY LogDate ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as MovingAvg
FROM TrafficLogs;

```

### 15. Basic Pivot (Rows to Columns)

**Question:** Rotate `YearlyData` where rows are months and columns are the years 2023, 2024.

SQL

```
SELECT Month, [2023], [2024]
FROM (SELECT Year, Month, Revenue FROM YearlyData) AS Source
PIVOT (SUM(Revenue) FOR Year IN ([2023], [2024])) AS Pvt;

```

### 16. Unpivoting (Columns to Rows)

**Question:** Reverse a table that has columns `Phone1`, `Phone2`, `Phone3` into a single `PhoneNumber` column.

SQL

```
SELECT UserID, PhoneNum
FROM UserPhones
UNPIVOT (PhoneNum FOR PhoneType IN (Phone1, Phone2, Phone3)) AS Unpvt;

```

### 17. Percentage of Total (Partition By)

**Question:** What percentage of total company sales does each region contribute?

SQL

```
SELECT Region, Sales,
    (Sales * 100.0 / SUM(Sales) OVER()) as PctOfTotal
FROM RegionalSales;

```

### 18. Finding Consecutive Days (Gaps and Islands)

**Question:** Find users with a 3-day login streak.

SQL

```
WITH Groups AS (
    SELECT UserID, LogDate,
    DATEADD(day, -ROW_NUMBER() OVER(PARTITION BY UserID ORDER BY LogDate), LogDate) as Grp
    FROM Logins
)
SELECT UserID FROM Groups GROUP BY UserID, Grp HAVING COUNT(*) >= 3;

```

### 19. Partition by Multiple Columns

**Question:** Rank sales within each Country and then within each Category.

SQL

```
SELECT Country, Category, Amount,
    RANK() OVER(PARTITION BY Country, Category ORDER BY Amount DESC) as rnk
FROM GlobalSales;

```

### 20. Difference between ROWS and RANGE

Question: Explain the output difference.

Answer: ROWS is physical (always looks at specific row counts). RANGE is logical (includes all rows with the same value in the ORDER BY clause). If two sales have the exact same timestamp, RANGE includes both in the sum; ROWS might not.

### 21. Email Masking (String Manipulation)

**Question:** Show only the first 2 and last 2 characters of the username in an email.

SQL

```
SELECT 
    CONCAT(LEFT(Email, 2), '***', SUBSTRING(Email, CHARINDEX('@', Email)-2, 2), SUBSTRING(Email, CHARINDEX('@', Email), 50))
FROM Users;

```

### 22. Finding the Last Day of Month

**Question:** Filter for transactions that happened on the last day of the month.

SQL

```
SELECT * FROM Transactions 
WHERE TransDate = EOMONTH(TransDate);

```

### 23. Handling NULLs in Comparisons

**Question:** Select records where `Status` is not 'Active' (including NULLs).

SQL

```
SELECT * FROM Tasks 
WHERE Status <> 'Active' OR Status IS NULL;

```

### 24. Recursive CTE for Dates

**Question:** Generate all Mondays in 2024.

SQL

```
WITH Dates AS (
    SELECT CAST('2024-01-01' AS DATE) as D
    UNION ALL
    SELECT DATEADD(day, 1, D) FROM Dates WHERE D < '2024-12-31'
)
SELECT D FROM Dates WHERE DATENAME(weekday, D) = 'Monday'
OPTION (MAXRECURSION 400);

```

### 25. String Splitting (Cross Apply)

**Question:** Count items in a comma-separated string column.

SQL

```
SELECT value as Item, COUNT(*) 
FROM Orders CROSS APPLY STRING_SPLIT(ItemList, ',')
GROUP BY value;

```

### 26. Finding the Mode (Common Record)

**Question:** Which product is most frequently bought?

SQL

```
SELECT TOP 1 ProductID FROM OrderDetails
GROUP BY ProductID ORDER BY COUNT(*) DESC;

```

### 27. The Anti-Join (Not Exists)

**Question:** Find customers with no orders (Performance optimized).

SQL

```
SELECT CustomerName FROM Customers C
WHERE NOT EXISTS (SELECT 1 FROM Orders O WHERE O.CustomerID = C.CustomerID);

```

### 28. Business Days Calculation Logic

Question: Describe the query logic.

Answer: Calculate DATEDIFF(day, start, end). Subtract (DATEDIFF(week, start, end) * 2). Join with a Holidays table to subtract specific dates that fall between the range.

### 29. Update from Join

**Question:** Set all "Standard" users to "Premium" if they spent over $1000.

SQL

```
UPDATE U SET U.Membership = 'Premium'
FROM Users U
JOIN (SELECT UserID, SUM(Amt) as Total FROM Orders GROUP BY UserID) O ON U.ID = O.UserID
WHERE O.Total > 1000;

```

### 30. Delete Duplicate Rows (The CTE Method)

**Question:** Clean a table of duplicates based on Email, keeping the earliest ID.

SQL

```
WITH CTE AS (
    SELECT ID, ROW_NUMBER() OVER(PARTITION BY Email ORDER BY ID) as rnk
    FROM Users
)
DELETE FROM CTE WHERE rnk > 1;

```

----------

To round out your senior-level preparation, here is the **SQL Query Tuning & Optimization** section. This covers the "how-to" for identifying bottlenecks and the specific techniques used to resolve them in high-scale environments.

----------

## **Part 3: SQL Query Tuning (Senior Interview Edition)**

### **Theory: Tuning Concepts**

#### 21. How do you identify a "Slow Query"?

**Answer:** Beyond user complaints, you use system tools like **SQL Server Profiler**, **Slow Query Logs** (MySQL/PostgreSQL), or **DMVs** (Dynamic Management Views). You look for high CPU usage, high Logical Reads (I/O), and long "Elapsed Time."

#### 22. What is the "SARGability" of a query?

**Answer:** SARGable stands for **Search ARgumentable**. A query is SARGable if the DB engine can use an index.

-   **Non-SARGable:** `WHERE YEAR(OrderDate) = 2024` (Function on column prevents index use).
    
-   **SARGable:** `WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'`.
    

#### 23. Explain "Parameter Sniffing."

**Answer:** This occurs when the SQL engine creates an execution plan based on the specific parameters used the first time a query is compiled. If the data distribution is uneven (skewed), that plan might be terrible for different parameters. It is solved by using `OPTIMIZE FOR UNKNOWN` or re-compiling the stored procedure.

#### 24. What is the difference between Nested Loops, Hash Joins, and Merge Joins?

**Answer:**

-   **Nested Loops:** Best for small datasets; one table is iterated for every row of the other.
    
-   **Hash Join:** Used for large, unsorted datasets; the engine builds a hash table in memory.
    
-   **Merge Join:** Most efficient for large datasets; requires both inputs to be sorted by the join key.
    

#### 25. How do "Statistics" affect query performance?

**Answer:** Statistics describe the distribution of data in columns. The Query Optimizer uses them to estimate the number of rows (cardinality) a query will return. If statistics are outdated, the optimizer might choose a Table Scan instead of an Index Seek.

----------

### **Practical: Tuning & Optimization Scenarios**

#### 31. Refactoring Non-SARGable Queries

**Question:** The following query is taking 10 seconds because it scans the whole table. Refactor it to use an index on `SignupDate`.

-   **Slow:** `SELECT UserID FROM Users WHERE DATEADD(day, 7, SignupDate) > GETDATE();`
    
-   **Optimized:**
    

SQL

```
-- Move the operation away from the column to the constant/variable
SELECT UserID FROM Users 
WHERE SignupDate > DATEADD(day, -7, GETDATE());

```

#### 32. Eliminating `SELECT *`

Question: Why is SELECT * considered a performance anti-pattern in senior development?

Answer: 1. I/O Overhead: It fetches data from disk/memory that isn't needed.

2. Breaks Covering Indexes: Even if you have an index on Name, SELECT * forces a "Key Lookup" to find the other columns.

3. Network Latency: Larger result sets increase the time to move data to the app server.

#### 33. Optimizing `OR` with `UNION`

Question: A query with WHERE Status = 'A' OR Category = 'B' is slow because it causes an index scan. How can you optimize it?

Answer: Often, the optimizer struggles with OR. Using UNION allows the engine to perform two separate Index Seeks.

SQL

```
SELECT * FROM Items WHERE Status = 'A'
UNION
SELECT * FROM Items WHERE Category = 'B';

```

#### 34. Using `EXISTS` vs `IN`

Question: When is EXISTS better than IN?

Answer: IN usually collects the entire subquery result set in memory before filtering. EXISTS returns TRUE as soon as the first match is found (short-circuiting). For large subqueries, EXISTS is generally more efficient.

#### 35. Dealing with "Big Delete" Operations

Question: You need to delete 10 million rows from a 100 million row table. How do you do it without locking the table for an hour?

Answer: Perform the delete in batches. This prevents the transaction log from exploding and keeps locks short.

SQL

```
WHILE 1 = 1
BEGIN
    DELETE TOP (5000) FROM Logs WHERE LogDate < '2023-01-01';
    IF @@ROWCOUNT = 0 BREAK;
    -- Optional: WAITFOR DELAY '00:00:01' (to let other processes in)
END

```

----------

### **The Query Tuning Checklist (For Interview Talk-through)**

When an interviewer asks, "How would you optimize this slow query?", follow this hierarchy:

1.  **Examine the Execution Plan:** Look for the highest cost operators (Fat arrows vs Thin arrows).
    
2.  **Check for Missing Indexes:** Does the plan suggest a missing index?
    
3.  **Check Cardinality:** Does the "Estimated Number of Rows" match the "Actual Number of Rows"? If not, update statistics.
    
4.  **Look for Scans:** Is there a Table Scan where there should be a Seek?
    
5.  **Check SARGability:** Are there functions on columns in the `WHERE` or `JOIN` clauses?
    
6.  **Evaluate Joins:** Are we joining on columns of the same data type? (Implicit conversion kills performance).
    
7.  **Reduce I/O:** Remove unnecessary columns from `SELECT` and check for Covering Indexes.
    

----------
