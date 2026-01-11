
Normalization is the process of organizing data in a database to reduce redundancy and improve data integrity. Each "Normal Form" (NF) is a set of rules you must follow, with each level building upon the previous one.

### 1NF: First Normal Form (Atomic Values)

A table is in 1NF if every cell contains a **single, indivisible (atomic) value**, and every record is unique.

-   **Rule:** No multiple values in one cell (e.g., no lists like "Red, Blue").
    
-   **Rule:** No repeating groups of similar data.
    
-   **Rule:** Each row must be uniquely identifiable via a Primary Key.
    

----------

### 2NF: Second Normal Form (Partial Dependencies)

A table is in 2NF if it is already in 1NF and **all non-key attributes are fully dependent on the entire primary key.**

-   **Rule:** This only applies to tables with **Composite Primary Keys** (keys made of two or more columns).
    
-   **The Problem:** If a column depends on only _part_ of the composite key, it’s a partial dependency.
    
-   **The Fix:** Move the partial dependency into a separate table.
    

----------

### 3NF: Third Normal Form (Transitive Dependencies)

A table is in 3NF if it is in 2NF and has **no transitive dependencies.**

-   **The Rule:** Non-key columns should not depend on other non-key columns. They must depend _only_ on the Primary Key.
    
-   **Example:** If you have `Book_ID (PK)`, `Author_ID`, and `Author_Birthdate`:
    
    -   `Author_Birthdate` depends on `Author_ID`.
        
    -   `Author_ID` depends on `Book_ID`.
        
    -   Therefore, `Author_Birthdate` depends on `Book_ID` _transitively_.
        
-   **The Fix:** Move `Author_ID` and `Author_Birthdate` to their own `Authors` table.
    

----------

### BCNF: Boyce-Codd Normal Form

BCNF is a slightly stronger version of 3NF, often called "3.5NF." It deals with anomalies that occur when a table has **multiple overlapping candidate keys.**

-   **The Rule:** For every functional dependency ($X \to Y$), **X must be a Super Key** (a candidate key).
    
-   **In Simple Terms:** In 3NF, a non-key column can't depend on another non-key column. In BCNF, even a _part_ of a composite key cannot depend on a non-key column.
    

----------

### Combined Summary: The "Key" Analogy

A famous way to remember the relationship between the forms is Bill Kent's variation of the courtroom oath. To be normalized, every non-key attribute must provide information about:

> **"The Key (1NF), the Whole Key (2NF), and Nothing But The Key (3NF), so help me Codd (BCNF)."**

|**Normal Form**|**Requirement**|**Goal**|
|--|--|--|
| **1NF** | Atomic values & Primary Key | Eliminate duplicate data in a single row. |
| **2NF** | 1NF + No partial dependencies | Ensure columns depend on the _entire_ PK. |
| **3NF** | 2NF + No transitive dependencies | Ensure columns depend _only_ on the PK. |
| **BCNF** | 3NF + Every determinant is a candidate key | Fix anomalies in tables with overlapping keys. |

----------
To make this stick, let's look at each level through a different lens.

### 1NF: The "Single Box" Rule

**Example:** Imagine a `Customer_Hobby` table. One cell says: `Hobby: "Soccer, Chess, Painting"`.

-   **The Violation:** You can't easily search for "Chess" players without using slow "contains" logic.
    
-   **The 1NF Fix:** Every hobby gets its own row. The "cell" must be atomic (indivisible).
    

### 2NF: The "All or Nothing" Rule

**Example:** A `Store_Inventory` table with a Composite Key: `{Store_ID, Product_ID}`.

-   **The Violation:** You include a column `Store_Address`.
    
-   **Why?** The `Store_Address` only depends on the `Store_ID`, not the `Product_ID`. It doesn't matter what product is being sold; the address of Store #5 is always the same.
    
-   **The 2NF Fix:** Move `Store_Address` to a separate `Stores` table.
    

### 3NF: The "No Middleman" Rule

**Example:** An `Employee` table with `Employee_ID (PK)`, `Zip_Code`, and `City`.

-   **The Violation:** `City` depends on `Zip_Code`. While both depend on the `Employee_ID`, the relationship between Zip and City exists independently of the employee.
    
-   **Why?** If you change a Zip Code, you _must_ remember to change the City, or your data becomes inconsistent.
    
-   **The 3NF Fix:** Create a `Location` table with `Zip_Code (PK)` and `City`.
    

### BCNF: The "Strict Ownership" Rule

**Example:** A `Clinic_Schedule` table with `{Student_ID, Subject}` as a composite key. Each student has one `Teacher` per subject, but a `Teacher` only teaches one `Subject`.

-   **The Violation:** `Subject` depends on `Teacher` (the determinant), but `Teacher` is not a candidate key (because one teacher can have many students).
    
-   **The BCNF Fix:** Separate `Teacher` and `Subject` into their own table so that the "Teacher" is the primary key for that relationship.

-----------
### Practice Challenge: The "Messy" Table

Below is a table for a **University Club System**. It is currently unnormalized.
#### Your Task:

Try to normalize this table down to 3NF. Here are the "hidden" rules of this data:

1.  A student can join multiple clubs.
    
2.  Each club has one fixed fee and one leader.
    
3.  Each leader has one phone number.
    

**How would you split this into 3 tables to satisfy 3NF?**

**Table: `Club_Enrollment`**
```
-- 1. Create the unnormalized table
CREATE TABLE Club_Enrollment (
    Student_ID INT,
    Student_Name VARCHAR(50),
    Club_Code VARCHAR(10),
    Club_Name VARCHAR(50),
    Club_Fees DECIMAL(10, 2),
    Activity_Leader VARCHAR(50),
    Leader_Phone VARCHAR(15)
);

-- 2. Insert the redundant data
INSERT INTO Club_Enrollment 
VALUES (1001, 'Alice', 'CHESS', 'Chess Club', 20.00, 'Bob', '555-0123'),
       (1001, 'Alice', 'ART', 'Art Society', 30.00, 'Clara', '555-9999'),
       (1002, 'David', 'CHESS', 'Chess Club', 20.00, 'Bob', '555-0123');

```
