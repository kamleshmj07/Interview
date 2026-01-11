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
    
31.  **NEW:** Exception Handling (TRY...CATCH)
    
32.  **NEW:** What is Collation?
    
33.  **NEW:** Composite Key vs. Surrogate Key
    
34.  **NEW:** Varchar vs NVarchar
    
35.  **NEW:** COMMIT and ROLLBACK
    

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

### 1-30. (Standard Senior Questions)

_For brevity, refer to previous sections for the detailed breakdown of the original 30 senior questions, including ACID, Joins, and Indexing._

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

### 1-8. (Refer to previous sections for Ranking and Aggregation logic)

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
