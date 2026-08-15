  # SQL — Lead .NET Interview Prep

## Table of Contents

1. [Indexes](#1-indexes)
2. [Execution Plans](#2-execution-plans)
3. [Deadlocks](#3-deadlocks)
4. [Isolation Levels](#4-isolation-levels)
5. [CTEs (Common Table Expressions)](#5-ctes)
6. [Window Functions](#6-window-functions)
7. [PARTITION BY](#7-partition-by)
8. [Pagination](#8-pagination)
9. [Stored Procedures vs Inline Queries](#9-stored-procedures-vs-inline-queries)
10. [Performance Tuning](#10-performance-tuning)
11. [Coding Problems](#11-coding-problems)
    - [Problem 1: Delete Duplicate Rows](#problem-1-delete-duplicate-rows)
    - [Problem 2: Nth Highest Salary](#problem-2-nth-highest-salary)
    - [Problem 3: Running Total](#problem-3-running-total)
    - [Problem 4: Recursive Employee Hierarchy](#problem-4-recursive-employee-hierarchy)
    - [Problem 5: Find Gaps in Sequence](#problem-5-find-gaps-in-sequence)
    - [Problem 6: Pivot / Cross-Tab](#problem-6-pivot--cross-tab)

---

## 1. Indexes

### Clustered Index

A clustered index defines the **physical order** of data rows on disk. The leaf nodes of a clustered index **are** the actual data pages.

- Only **one** clustered index per table (because data can only be sorted one way)
- By default, a primary key creates a clustered index
- Lookups by clustered key are fastest — no extra hop required
- Range scans are efficient because data is physically ordered

```sql
-- Create a table with a clustered index on OrderID (via PRIMARY KEY)
CREATE TABLE Orders (
    OrderID   INT           NOT NULL PRIMARY KEY,  -- Clustered by default
    CustomerID INT          NOT NULL,
    OrderDate  DATETIME     NOT NULL,
    Total      DECIMAL(10,2)NOT NULL
);
```

### Non-Clustered Index

A non-clustered index is a **separate B-tree structure** whose leaf nodes contain the index key plus a **row locator** (the clustered key or a RID for heap tables). SQL Server follows the pointer to fetch the actual row.

- Up to **999** non-clustered indexes per table
- Adds overhead on INSERT/UPDATE/DELETE
- Best for columns frequently used in WHERE, JOIN ON, or ORDER BY

```sql
-- Non-clustered index on CustomerID for fast customer lookups
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID
    ON Orders (CustomerID);
```

### Covering Index

A covering index includes all columns the query needs, so SQL Server **never has to go back to the base table**. Extra columns are added via the `INCLUDE` clause (not in the key).

```sql
-- Query: SELECT OrderDate, Total FROM Orders WHERE CustomerID = 42
-- Without covering index: Index Seek on CustomerID + Key Lookup for OrderDate, Total
-- With covering index: Index Seek only — zero key lookups

CREATE NONCLUSTERED INDEX IX_Orders_CustomerID_Covering
    ON Orders (CustomerID)
    INCLUDE (OrderDate, Total);
```

> **Interview Q: What is the INCLUDE clause and why use it instead of adding columns to the key?**
>
> Including columns in the key changes the sort order and increases the key size (which propagates through all index levels). INCLUDE columns only live in the leaf level, covering the query without affecting the key structure. This keeps the index narrower and more efficient.

### Composite Index

A composite index spans multiple columns. The **leftmost prefix rule** applies: the index is only usable if the query filters on the leading column(s) in order.

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerDate
    ON Orders (CustomerID, OrderDate);

-- USES the index (leading column present)
SELECT * FROM Orders WHERE CustomerID = 5;
SELECT * FROM Orders WHERE CustomerID = 5 AND OrderDate > '2024-01-01';

-- Does NOT use the index efficiently (skips leading column)
SELECT * FROM Orders WHERE OrderDate > '2024-01-01';
```

| Column Order | Query | Uses Index? |
|---|---|---|
| (CustomerID, OrderDate) | WHERE CustomerID = 5 | Yes |
| (CustomerID, OrderDate) | WHERE CustomerID = 5 AND OrderDate = '2024-01-01' | Yes (both cols) |
| (CustomerID, OrderDate) | WHERE OrderDate = '2024-01-01' | No (leading column missing) |
| (OrderDate, CustomerID) | WHERE OrderDate = '2024-01-01' | Yes |

### When NOT to Index

| Situation | Reason |
|---|---|
| Low-cardinality columns (e.g., IsActive BIT) | Too many matching rows — table scan is cheaper |
| Frequently updated columns | Every update must maintain the index |
| Very small tables | Table scan is fast enough |
| Columns never used in WHERE/JOIN/ORDER BY | Index is never consulted |

### Real Example: Before/After Index

```sql
-- Slow query: no index on CustomerID
SELECT OrderDate, Total
FROM   Orders
WHERE  CustomerID = 1001
ORDER  BY OrderDate DESC;
-- Execution plan: TABLE SCAN — reads every row

-- Fix: add covering index
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID_Cover
    ON Orders (CustomerID, OrderDate DESC)
    INCLUDE (Total);

-- After: INDEX SEEK on CustomerID + ordered by OrderDate — zero key lookups
```

> **Interview Q: What is the difference between a clustered and non-clustered index?**
>
> A clustered index defines the physical storage order of rows — the leaf level IS the data. A table can have only one. A non-clustered index is a separate structure whose leaf nodes hold the key plus a pointer back to the data row. Multiple non-clustered indexes are allowed. When a non-clustered index does not cover the query, SQL Server performs a Key Lookup to fetch remaining columns from the clustered index, which is expensive at scale.

---

## 2. Execution Plans

### How to Read an Execution Plan

Open in SSMS: `SET STATISTICS IO, TIME ON` and Ctrl+M for actual plan.

| Operator | Meaning | When You See It |
|---|---|---|
| **Table Scan** | Full table read, no index used | Missing index or low selectivity |
| **Index Scan** | Full index read | Non-selective predicate on indexed col |
| **Index Seek** | Jump directly to matching rows | Good — selective WHERE clause |
| **Key Lookup** | Extra trip to clustered index for missing cols | Non-covering index; fix with INCLUDE |
| **Nested Loop** | For each outer row, scan inner | Good for small outer sets |
| **Hash Join** | Build hash table for one side | Large unsorted sets, no index |
| **Merge Join** | Merge two sorted inputs | Both inputs already ordered |

### Key Lookup Problem

```sql
-- Index exists on CustomerID only
CREATE NONCLUSTERED INDEX IX_Orders_Cust ON Orders (CustomerID);

-- This query triggers a Key Lookup for every row
SELECT CustomerID, OrderDate, Total
FROM   Orders
WHERE  CustomerID = 42;
-- Plan: Index Seek (CustomerID) → Key Lookup (OrderDate, Total) × N rows

-- Fix: add INCLUDE to eliminate key lookups
CREATE NONCLUSTERED INDEX IX_Orders_Cust_Cover
    ON Orders (CustomerID)
    INCLUDE (OrderDate, Total);
```

### Statistics and Why They Matter

SQL Server's query optimizer uses **statistics** (histograms of column value distribution) to estimate row counts and choose join strategies. Stale statistics cause bad plans.

```sql
-- Update statistics manually
UPDATE STATISTICS Orders;

-- Or update all
EXEC sp_updatestats;

-- View statistics for a table
DBCC SHOW_STATISTICS ('Orders', 'IX_Orders_CustomerID');
```

### Real Example: Troubleshoot a Slow JOIN

```sql
-- Slow query
SELECT c.Name, COUNT(o.OrderID) AS OrderCount
FROM   Customers c
JOIN   Orders o ON c.CustomerID = o.CustomerID
WHERE  c.Region = 'West'
GROUP  BY c.Name;

-- Diagnosis steps:
-- 1. Check execution plan for Table Scans
-- 2. Look for Hash Joins (may indicate missing index)
-- 3. Check estimated vs actual rows — big diff = stale statistics

-- Fix 1: Index on the join column that lacks one
CREATE NONCLUSTERED INDEX IX_Customers_Region
    ON Customers (Region)
    INCLUDE (Name, CustomerID);

-- Fix 2: Update statistics
UPDATE STATISTICS Customers;
UPDATE STATISTICS Orders;
```

> **Tip:** A **Hash Join** is not always bad — it is optimal for large unsorted inputs. The problem is when you see it on small tables (bad cardinality estimates).

---

## 3. Deadlocks

### What Causes Deadlocks

A deadlock occurs when two (or more) transactions each hold a lock the other needs, creating a **circular lock dependency**. Neither can proceed.

```
Transaction A locks Row 1, then tries to lock Row 2 → waits for B
Transaction B locks Row 2, then tries to lock Row 1 → waits for A
= Deadlock
```

### How SQL Server Handles Deadlocks

SQL Server's lock monitor detects cycles every ~5 seconds and selects the **deadlock victim** — typically the transaction with the lowest cost to roll back (configurable via `DEADLOCK_PRIORITY`). The victim receives error 1205 and must retry.

```sql
-- Capture deadlocks with Extended Events (in production)
-- Or read the deadlock graph from the system health session:
SELECT xdr.value('@timestamp', 'datetime2') AS DeadlockTime,
       xdr.query('.') AS DeadlockGraph
FROM (
    SELECT CAST(target_data AS XML) AS target_data
    FROM   sys.dm_xe_session_targets t
    JOIN   sys.dm_xe_sessions s ON t.event_session_address = s.address
    WHERE  s.name = 'system_health'
      AND  t.target_name = 'ring_buffer'
) AS data
CROSS APPLY target_data.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS xdr_data(xdr);
```

### Prevention Strategies

1. **Consistent lock order** — always access tables/rows in the same order across transactions
2. **Keep transactions short** — minimize time holding locks
3. **Use row-level locking** — avoid table-level locks
4. **Add appropriate indexes** — fewer rows locked per operation
5. **Use SNAPSHOT isolation** — readers don't block writers (MVCC)
6. **Retry logic in application** — catch error 1205, retry

### Real Example: Deadlock Scenario and Fix

```sql
-- SETUP
CREATE TABLE Accounts (
    AccountID INT PRIMARY KEY,
    Balance   DECIMAL(10,2)
);
INSERT INTO Accounts VALUES (1, 1000.00), (2, 500.00);

-- Transaction A (Session 1)
BEGIN TRAN;
    UPDATE Accounts SET Balance = Balance - 100 WHERE AccountID = 1; -- Locks row 1
    WAITFOR DELAY '00:00:05';
    UPDATE Accounts SET Balance = Balance + 100 WHERE AccountID = 2; -- Waits for row 2

-- Transaction B (Session 2) — run concurrently
BEGIN TRAN;
    UPDATE Accounts SET Balance = Balance - 50  WHERE AccountID = 2; -- Locks row 2
    UPDATE Accounts SET Balance = Balance + 50  WHERE AccountID = 1; -- Waits for row 1 → DEADLOCK

-- FIX: Always update rows in ascending AccountID order in both transactions
-- Both transactions then contend on row 1 first — one waits, no circular dependency
BEGIN TRAN;
    UPDATE Accounts SET Balance = Balance - 100 WHERE AccountID = 1;
    UPDATE Accounts SET Balance = Balance + 100 WHERE AccountID = 2;
COMMIT;

BEGIN TRAN;
    UPDATE Accounts SET Balance = Balance + 50  WHERE AccountID = 1; -- Same order
    UPDATE Accounts SET Balance = Balance - 50  WHERE AccountID = 2;
COMMIT;
```

---

## 4. Isolation Levels

Isolation levels control how transactions interact with concurrent data changes.

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Blocking |
|---|:---:|:---:|:---:|---|
| READ UNCOMMITTED | Yes | Yes | Yes | None |
| READ COMMITTED | No | Yes | Yes | Readers block writers |
| REPEATABLE READ | No | No | Yes | Higher |
| SERIALIZABLE | No | No | No | Highest |
| SNAPSHOT | No | No | No | Readers never block |

### READ UNCOMMITTED

Reads data that hasn't been committed — "dirty reads." Fastest but dangerous.

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM Orders;  -- May see uncommitted rows that get rolled back
```

### READ COMMITTED (SQL Server Default)

Only reads committed data. Releases shared locks immediately after read. Non-repeatable reads can occur (re-reading the same row may give different values if another transaction commits in between).

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;  -- Default, no need to set
```

### REPEATABLE READ

Holds shared locks until end of transaction. Guarantees the same rows return the same values if re-read. Phantom reads still possible (new rows can be inserted by another transaction).

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### SERIALIZABLE

Fully serializes transactions. Range locks prevent phantom reads. Maximum consistency, maximum blocking.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

### SNAPSHOT (MVCC)

Uses row versioning in TempDB. Readers see a consistent snapshot from when the transaction started. Readers never block writers; writers never block readers. Best for read-heavy workloads.

```sql
-- Must be enabled at database level first:
ALTER DATABASE YourDatabase SET ALLOW_SNAPSHOT_ISOLATION ON;

-- Then use in session:
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
```

### Real Example: Banking Scenario

**Scenario:** Transfer $100 from Account A to Account B. The balance must be read and written atomically with no other transaction seeing a partial state.

```sql
-- Which isolation level?
-- READ COMMITTED: not enough — a concurrent read could see A debited but B not yet credited
-- REPEATABLE READ: better, but phantom reads possible
-- SERIALIZABLE: correct but high contention on busy systems
-- SNAPSHOT: best choice for banking reads — consistent view without blocking

-- Implementation with SNAPSHOT isolation:
ALTER DATABASE BankDB SET ALLOW_SNAPSHOT_ISOLATION ON;

SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRAN;
    DECLARE @BalanceA DECIMAL(10,2);
    SELECT @BalanceA = Balance FROM Accounts WHERE AccountID = 1;

    IF @BalanceA >= 100
    BEGIN
        UPDATE Accounts SET Balance = Balance - 100 WHERE AccountID = 1;
        UPDATE Accounts SET Balance = Balance + 100 WHERE AccountID = 2;
        COMMIT;
    END
    ELSE
    BEGIN
        ROLLBACK;
        RAISERROR('Insufficient funds', 16, 1);
    END
```

> **Interview Q: What is the difference between SNAPSHOT isolation and READ COMMITTED SNAPSHOT (RCSI)?**
>
> SNAPSHOT isolation is transaction-scoped — your snapshot is fixed at the start of the transaction. RCSI (Read Committed Snapshot Isolation) is statement-scoped — each statement sees committed data as of the start of that statement. RCSI is enabled with `ALTER DATABASE ... SET READ_COMMITTED_SNAPSHOT ON` and changes the behavior of the default READ COMMITTED level without requiring code changes.

---

## 5. CTEs

### Regular CTE

```sql
-- CTE for readability and reuse within a single query
WITH RecentOrders AS (
    SELECT CustomerID,
           OrderID,
           OrderDate,
           Total,
           ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY OrderDate DESC) AS rn
    FROM   Orders
    WHERE  OrderDate >= DATEADD(YEAR, -1, GETDATE())
)
SELECT CustomerID, OrderID, OrderDate, Total
FROM   RecentOrders
WHERE  rn = 1;  -- Latest order per customer
```

### Recursive CTE

A recursive CTE has two parts joined by `UNION ALL`:
1. **Anchor member** — base case (returns root rows)
2. **Recursive member** — references the CTE itself to traverse levels

```sql
-- Employee hierarchy traversal
WITH EmployeeHierarchy AS (
    -- Anchor: top-level employees (no manager)
    SELECT EmployeeID,
           Name,
           ManagerID,
           0 AS Level,
           CAST(Name AS VARCHAR(500)) AS Path
    FROM   Employees
    WHERE  ManagerID IS NULL

    UNION ALL

    -- Recursive: each employee's direct reports
    SELECT e.EmployeeID,
           e.Name,
           e.ManagerID,
           eh.Level + 1,
           CAST(eh.Path + ' > ' + e.Name AS VARCHAR(500))
    FROM   Employees e
    INNER JOIN EmployeeHierarchy eh ON e.ManagerID = eh.EmployeeID
)
SELECT Level,
       REPLICATE('  ', Level) + Name AS IndentedName,
       Path
FROM   EmployeeHierarchy
ORDER  BY Path;
```

### CTE for Pagination

```sql
WITH PagedOrders AS (
    SELECT OrderID,
           CustomerID,
           OrderDate,
           Total,
           ROW_NUMBER() OVER (ORDER BY OrderDate DESC) AS RowNum
    FROM   Orders
)
SELECT OrderID, CustomerID, OrderDate, Total
FROM   PagedOrders
WHERE  RowNum BETWEEN 21 AND 30;  -- Page 3, 10 rows per page
```

> **Pitfall:** A CTE is not materialized by default — SQL Server may evaluate it multiple times if referenced multiple times in the query. Use a temp table or `OPTION (MAXRECURSION n)` for recursive CTEs that might run deep (default limit is 100).

---

## 6. Window Functions

Window functions perform calculations **across a set of rows related to the current row** without collapsing them into a single group (unlike `GROUP BY`).

### Ranking Functions

```sql
SELECT EmployeeID,
       DepartmentID,
       Salary,
       ROW_NUMBER()  OVER (PARTITION BY DepartmentID ORDER BY Salary DESC) AS RowNum,
       RANK()        OVER (PARTITION BY DepartmentID ORDER BY Salary DESC) AS Rnk,
       DENSE_RANK()  OVER (PARTITION BY DepartmentID ORDER BY Salary DESC) AS DenseRnk,
       NTILE(4)      OVER (PARTITION BY DepartmentID ORDER BY Salary DESC) AS Quartile
FROM   Employees;

-- Given salaries: 100k, 90k, 90k, 80k
-- ROW_NUMBER:  1, 2, 3, 4   (always unique, arbitrary tiebreak)
-- RANK:        1, 2, 2, 4   (gaps after ties)
-- DENSE_RANK:  1, 2, 2, 3   (no gaps after ties)
-- NTILE(4):    1, 2, 3, 4   (divides into 4 equal buckets)
```

### LAG and LEAD

```sql
SELECT OrderDate,
       Total,
       LAG(Total, 1, 0)  OVER (ORDER BY OrderDate) AS PreviousTotal,
       LEAD(Total, 1, 0) OVER (ORDER BY OrderDate) AS NextTotal,
       Total - LAG(Total, 1, 0) OVER (ORDER BY OrderDate) AS DeltaFromPrevious
FROM   Orders
WHERE  CustomerID = 42;
```

### Aggregate Window Functions

```sql
-- Running total and moving average per customer
SELECT CustomerID,
       OrderDate,
       Total,
       SUM(Total)  OVER (PARTITION BY CustomerID ORDER BY OrderDate
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS RunningTotal,
       AVG(Total)  OVER (PARTITION BY CustomerID ORDER BY OrderDate
                          ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)         AS MovingAvg3
FROM   Orders;
```

### PARTITION BY vs GROUP BY

| Feature | GROUP BY | PARTITION BY (Window) |
|---|---|---|
| Collapses rows | Yes — one row per group | No — all rows preserved |
| Can mix aggregated and non-aggregated | No (without tricks) | Yes |
| Use case | Summary reports | Per-row running totals, ranks, comparisons |

```sql
-- GROUP BY: one row per customer
SELECT CustomerID, SUM(Total) AS TotalSpend
FROM   Orders
GROUP  BY CustomerID;

-- PARTITION BY: all order rows retained, with customer total beside each
SELECT CustomerID, OrderID, Total,
       SUM(Total) OVER (PARTITION BY CustomerID) AS CustomerTotal
FROM   Orders;
```

### Real Example: Month-over-Month Growth

```sql
WITH MonthlySales AS (
    SELECT YEAR(OrderDate)  AS Yr,
           MONTH(OrderDate) AS Mo,
           SUM(Total)       AS Revenue
    FROM   Orders
    GROUP  BY YEAR(OrderDate), MONTH(OrderDate)
)
SELECT Yr, Mo, Revenue,
       LAG(Revenue) OVER (ORDER BY Yr, Mo) AS PrevRevenue,
       ROUND(
           (Revenue - LAG(Revenue) OVER (ORDER BY Yr, Mo))
           / NULLIF(LAG(Revenue) OVER (ORDER BY Yr, Mo), 0) * 100,
           2
       ) AS MoMGrowthPct
FROM   MonthlySales
ORDER  BY Yr, Mo;
```

---

## 7. PARTITION BY

`PARTITION BY` divides the result set into partitions for the window function to operate on independently.

### Latest Order per Customer (Deduplication Pattern)

```sql
-- Pattern: keep only the most recent row per group
WITH RankedOrders AS (
    SELECT CustomerID,
           OrderID,
           OrderDate,
           Total,
           ROW_NUMBER() OVER (
               PARTITION BY CustomerID
               ORDER BY OrderDate DESC, OrderID DESC  -- tiebreak by OrderID
           ) AS rn
    FROM   Orders
)
SELECT CustomerID, OrderID, OrderDate, Total
FROM   RankedOrders
WHERE  rn = 1;
```

### First Purchase Date per Customer

```sql
SELECT DISTINCT
       CustomerID,
       MIN(OrderDate) OVER (PARTITION BY CustomerID) AS FirstPurchaseDate,
       MAX(OrderDate) OVER (PARTITION BY CustomerID) AS LastPurchaseDate,
       COUNT(*)       OVER (PARTITION BY CustomerID) AS TotalOrders
FROM   Orders;
```

> **Tip:** When you only need `MIN`/`MAX` per group without other window calculations, `GROUP BY` is simpler. Use `PARTITION BY` when you need the per-group aggregate alongside individual row data.

---

## 8. Pagination

### OFFSET / FETCH (SQL Standard, SQL Server 2012+)

```sql
-- Page N (0-indexed), PageSize rows
DECLARE @Page     INT = 2;   -- 0-based page number
DECLARE @PageSize INT = 10;

SELECT OrderID, CustomerID, OrderDate, Total
FROM   Orders
ORDER  BY OrderDate DESC, OrderID DESC   -- Must have ORDER BY
OFFSET  @Page * @PageSize ROWS
FETCH NEXT @PageSize ROWS ONLY;
```

**Limitation:** Performance degrades on large offsets. `OFFSET 1000000 ROWS` forces SQL Server to read and discard 1 million rows before returning results.

### Keyset Pagination (Cursor-Based)

Instead of skipping rows by count, remember the **last value seen** and filter from there.

```sql
-- First page
DECLARE @PageSize INT = 10;

SELECT TOP (@PageSize)
       OrderID, OrderDate, Total
FROM   Orders
ORDER  BY OrderDate DESC, OrderID DESC;

-- Subsequent pages: pass the last OrderDate and OrderID from previous page
DECLARE @LastOrderDate DATETIME = '2024-03-15 09:22:11';
DECLARE @LastOrderID   INT      = 50231;

SELECT TOP (@PageSize)
       OrderID, OrderDate, Total
FROM   Orders
WHERE  OrderDate < @LastOrderDate
   OR (OrderDate = @LastOrderDate AND OrderID < @LastOrderID)
ORDER  BY OrderDate DESC, OrderID DESC;
```

### Comparison

| Aspect | OFFSET/FETCH | Keyset Pagination |
|---|---|---|
| Performance on large offsets | Degrades — reads all preceding rows | Constant — seeks directly |
| Random page access | Yes (jump to page 50) | No (must go page by page) |
| Stable results | No (new rows shift pages) | Yes (anchor is a value, not a position) |
| Complexity | Simple | Requires unique sortable key |
| Best for | Small-medium datasets, UI paging | Infinite scroll, large datasets (10M+) |

> **Interview Q: Why is keyset pagination faster for large datasets?**
>
> OFFSET forces the database to read and count all preceding rows before returning results — it cannot skip them. An index seek on a WHERE predicate (`OrderDate < @lastDate`) jumps directly to the starting position. On a 10-million-row table, page 1000 with OFFSET reads 10,000 rows to discard; keyset reads exactly the next 10.

---

## 9. Stored Procedures vs Inline Queries

### Stored Procedures — Pros and Cons

| Pros | Cons |
|---|---|
| Execution plan cached and reused | Parameter sniffing can cause bad plans |
| Reduced network traffic (one call) | Schema coupling — harder to refactor |
| Encapsulates business logic in DB | Logic spread across DB and application |
| Permissions granted on proc, not tables | Harder to test and version control |
| Can use transactions cleanly | ORM tools work better with inline SQL |

### Parameter Sniffing

SQL Server compiles a stored procedure's plan using the **first set of parameter values** passed. If those values are not representative, the cached plan may be terrible for subsequent calls with different parameters.

```sql
-- Proc compiled for CustomerID=1 (1 order) — bad plan for CustomerID=100 (10,000 orders)
CREATE PROCEDURE GetOrdersByCustomer
    @CustomerID INT
AS
BEGIN
    SELECT OrderID, OrderDate, Total
    FROM   Orders
    WHERE  CustomerID = @CustomerID;
END

-- Proof: run with low-volume customer first
EXEC GetOrdersByCustomer @CustomerID = 1;     -- Compiles plan assuming 1 row
EXEC GetOrdersByCustomer @CustomerID = 100;   -- Uses same (bad) plan
```

### Fixes for Parameter Sniffing

```sql
-- Option 1: OPTION (RECOMPILE) — recompile every time (use for infrequent, variable queries)
CREATE PROCEDURE GetOrdersByCustomer
    @CustomerID INT
AS
BEGIN
    SELECT OrderID, OrderDate, Total
    FROM   Orders
    WHERE  CustomerID = @CustomerID
    OPTION (RECOMPILE);
END

-- Option 2: Local variable trick — breaks sniffing (optimizer estimates avg, not sniffed value)
CREATE PROCEDURE GetOrdersByCustomer
    @CustomerID INT
AS
BEGIN
    DECLARE @LocalID INT = @CustomerID;
    SELECT OrderID, OrderDate, Total
    FROM   Orders
    WHERE  CustomerID = @LocalID;
END

-- Option 3: OPTIMIZE FOR — give the optimizer a specific or UNKNOWN value
SELECT OrderID, OrderDate, Total
FROM   Orders
WHERE  CustomerID = @CustomerID
OPTION (OPTIMIZE FOR (@CustomerID UNKNOWN));
```

### When to Use What

| Scenario | Recommendation |
|---|---|
| Complex multi-step logic with transactions | Stored procedure |
| Simple CRUD from ORM (EF Core) | Inline / parameterized query |
| Ad-hoc reporting with variable parameters | Stored proc with OPTION(RECOMPILE) |
| Microservices — logic in application | Inline parameterized queries |
| Batch operations | Stored procedure or TVPs |

---

## 10. Performance Tuning

### Avoid SELECT *

```sql
-- Bad: fetches all columns, prevents covering index use
SELECT * FROM Orders WHERE CustomerID = 42;

-- Good: only fetch needed columns, enables covering indexes
SELECT OrderID, OrderDate, Total FROM Orders WHERE CustomerID = 42;
```

### SARGable vs Non-SARGable Predicates

**SARGable** (Search ARGument able) means the predicate allows the optimizer to use an index seek.

| Non-SARGable (Bad) | SARGable Fix |
|---|---|
| `WHERE YEAR(OrderDate) = 2024` | `WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'` |
| `WHERE UPPER(Name) = 'SMITH'` | `WHERE Name = 'Smith'` (with case-insensitive collation) |
| `WHERE LEN(Description) > 100` | N/A — avoid or use computed column |
| `WHERE CustomerID + 1 = 42` | `WHERE CustomerID = 41` |

```sql
-- Non-SARGable: function on column forces table scan
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2024;

-- SARGable: range predicate uses index
SELECT * FROM Orders
WHERE OrderDate >= '2024-01-01'
  AND OrderDate <  '2025-01-01';
```

### Avoid Implicit Conversions

```sql
-- Column is INT, parameter is VARCHAR — implicit conversion, index ignored
DECLARE @ID VARCHAR(10) = '42';
SELECT * FROM Orders WHERE CustomerID = @ID;  -- Bad

-- Fix: use matching types
DECLARE @ID INT = 42;
SELECT * FROM Orders WHERE CustomerID = @ID;  -- Good
```

### TempDB Usage

Expensive operations that spill to TempDB:
- Hash joins with insufficient memory grants
- Sort operations on large datasets
- `#temp` tables and table variables for large datasets

```sql
-- Check for spills in execution plan (look for warnings)
-- Or monitor in DMVs:
SELECT * FROM sys.dm_exec_query_stats
WHERE total_spills > 0
ORDER BY total_spills DESC;
```

### Query Hints (Use Sparingly)

```sql
-- Force index use (when optimizer makes wrong choice)
SELECT * FROM Orders WITH (INDEX(IX_Orders_CustomerID))
WHERE CustomerID = 42;

-- Prevent locking (read-heavy reporting)
SELECT * FROM Orders WITH (NOLOCK) WHERE OrderDate >= '2024-01-01';

-- Force join type
SELECT * FROM Orders o
JOIN Customers c ON o.CustomerID = c.CustomerID
OPTION (LOOP JOIN);  -- Or HASH JOIN, MERGE JOIN
```

> **Pitfall:** Query hints hard-code optimizer decisions. A future data change (new index, updated statistics) might make the hint harmful. Use only as a last resort and document why.

---

## 11. Coding Problems

---

### Problem 1: Delete Duplicate Rows

**Scenario:** The `Employees` table has duplicate rows (same Name, Department, Salary). Keep one row, delete the rest.

```sql
-- Setup
CREATE TABLE Employees (
    EmployeeID  INT IDENTITY(1,1) PRIMARY KEY,
    Name        NVARCHAR(100)     NOT NULL,
    Department  NVARCHAR(100)     NOT NULL,
    Salary      DECIMAL(10,2)     NOT NULL
);

INSERT INTO Employees (Name, Department, Salary) VALUES
    ('Alice',   'Engineering', 95000),
    ('Alice',   'Engineering', 95000),  -- duplicate
    ('Alice',   'Engineering', 95000),  -- duplicate
    ('Bob',     'Marketing',   72000),
    ('Bob',     'Marketing',   72000),  -- duplicate
    ('Charlie', 'HR',          65000);

-- Verify duplicates
SELECT Name, Department, Salary, COUNT(*) AS Cnt
FROM   Employees
GROUP  BY Name, Department, Salary
HAVING COUNT(*) > 1;
```

**Approach 1: CTE with ROW_NUMBER (Preferred)**

```sql
WITH Dupes AS (
    SELECT EmployeeID,
           ROW_NUMBER() OVER (
               PARTITION BY Name, Department, Salary
               ORDER BY EmployeeID   -- keep the lowest ID
           ) AS rn
    FROM   Employees
)
DELETE FROM Dupes WHERE rn > 1;

-- Verify: should have 3 rows remaining
SELECT * FROM Employees;
```

**Approach 2: Self-Join**

```sql
DELETE e1
FROM   Employees e1
JOIN   Employees e2
    ON  e1.Name       = e2.Name
   AND  e1.Department = e2.Department
   AND  e1.Salary     = e2.Salary
   AND  e1.EmployeeID > e2.EmployeeID;  -- keep the row with lower ID
```

> **Interview Q: Which approach is better and why?**
>
> The CTE with ROW_NUMBER is generally preferred because it is more readable, handles three or more duplicates cleanly in a single pass, and lets you verify what will be deleted by doing a SELECT first (just change DELETE to SELECT). The self-join can become a cross-product on large tables.

---

### Problem 2: Nth Highest Salary

**Scenario:** Find the 3rd highest distinct salary from the Employees table.

```sql
-- Setup (reuse Employees table from above, or fresh)
CREATE TABLE SalaryTest (
    EmpID  INT IDENTITY PRIMARY KEY,
    Name   NVARCHAR(100),
    Salary DECIMAL(10,2)
);

INSERT INTO SalaryTest (Name, Salary) VALUES
    ('Alice',   120000),
    ('Bob',     110000),
    ('Charlie', 110000),  -- same salary as Bob
    ('Diana',    95000),
    ('Eve',      85000),
    ('Frank',    80000);
```

**Approach 1: DENSE_RANK (Handles Ties Correctly)**

```sql
DECLARE @N INT = 3;

WITH RankedSalaries AS (
    SELECT Salary,
           DENSE_RANK() OVER (ORDER BY Salary DESC) AS Rnk
    FROM   SalaryTest
)
SELECT DISTINCT Salary
FROM   RankedSalaries
WHERE  Rnk = @N;
-- Result: 95000 (1st=120k, 2nd=110k, 3rd=95k — ties don't skip ranks)
```

**Approach 2: OFFSET/FETCH**

```sql
DECLARE @N INT = 3;

SELECT DISTINCT Salary
FROM   SalaryTest
ORDER  BY Salary DESC
OFFSET  (@N - 1) ROWS
FETCH NEXT 1 ROW ONLY;
-- Result: 95000
```

> **Pitfall:** Using `RANK()` instead of `DENSE_RANK()` would skip ranks on ties. If two employees both have the 2nd highest salary, `RANK()` returns 1, 2, 2, 4 — there is no 3rd rank. `DENSE_RANK()` returns 1, 2, 2, 3 — correct.

---

### Problem 3: Running Total

**Scenario:** Calculate a running total of sales by date.

```sql
-- Setup
CREATE TABLE DailySales (
    SaleDate DATE          NOT NULL,
    Amount   DECIMAL(10,2) NOT NULL
);

INSERT INTO DailySales VALUES
    ('2024-01-01',  500.00),
    ('2024-01-02',  750.00),
    ('2024-01-03',  300.00),
    ('2024-01-04', 1200.00),
    ('2024-01-05',  450.00);
```

**Solution: SUM() OVER with ROWS**

```sql
SELECT SaleDate,
       Amount,
       SUM(Amount) OVER (
           ORDER BY SaleDate
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS RunningTotal,
       AVG(Amount) OVER (
           ORDER BY SaleDate
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS RunningAvg
FROM   DailySales
ORDER  BY SaleDate;

-- Output:
-- SaleDate    Amount    RunningTotal  RunningAvg
-- 2024-01-01  500.00    500.00        500.00
-- 2024-01-02  750.00   1250.00        625.00
-- 2024-01-03  300.00   1550.00        516.67
-- 2024-01-04 1200.00   2750.00        687.50
-- 2024-01-05  450.00   3200.00        640.00
```

> **Note:** `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is explicit and correct. Without the frame clause, `SUM() OVER (ORDER BY ...)` defaults to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which groups ties — both give the same result unless there are duplicate dates.

---

### Problem 4: Recursive Employee Hierarchy

**Scenario:** Given an employee table with a self-referencing ManagerID, display the full org chart with levels and paths.

```sql
-- Setup
CREATE TABLE OrgChart (
    EmployeeID  INT          NOT NULL PRIMARY KEY,
    Name        NVARCHAR(100)NOT NULL,
    ManagerID   INT          NULL REFERENCES OrgChart(EmployeeID),
    Title       NVARCHAR(100)NOT NULL
);

INSERT INTO OrgChart VALUES
    (1, 'Sarah Chen',    NULL, 'CEO'),
    (2, 'Mark Torres',   1,    'VP Engineering'),
    (3, 'Lisa Park',     1,    'VP Marketing'),
    (4, 'James Liu',     2,    'Senior Engineer'),
    (5, 'Emma Davis',    2,    'Senior Engineer'),
    (6, 'Ryan Kim',      4,    'Engineer'),
    (7, 'Anna Smith',    3,    'Marketing Manager'),
    (8, 'Tom Brown',     3,    'Marketing Manager'),
    (9, 'Zoe Wilson',    7,    'Marketing Coordinator');

-- Solution: Recursive CTE
WITH OrgHierarchy AS (
    -- Anchor: root node (CEO, no manager)
    SELECT EmployeeID,
           Name,
           Title,
           ManagerID,
           0                          AS Level,
           CAST(Name AS NVARCHAR(MAX)) AS OrgPath
    FROM   OrgChart
    WHERE  ManagerID IS NULL

    UNION ALL

    -- Recursive: each report one level deeper
    SELECT e.EmployeeID,
           e.Name,
           e.Title,
           e.ManagerID,
           h.Level + 1,
           h.OrgPath + N' → ' + e.Name
    FROM   OrgChart e
    INNER JOIN OrgHierarchy h ON e.ManagerID = h.EmployeeID
)
SELECT Level,
       REPLICATE(N'    ', Level) + Name AS IndentedName,
       Title,
       OrgPath
FROM   OrgHierarchy
ORDER  BY OrgPath
OPTION (MAXRECURSION 50);  -- Safety limit (default 100)

-- Output (indented):
-- Level 0: Sarah Chen (CEO)
-- Level 1:     Mark Torres (VP Engineering)
-- Level 2:         James Liu (Senior Engineer)
-- Level 3:             Ryan Kim (Engineer)
-- Level 2:         Emma Davis (Senior Engineer)
-- Level 1:     Lisa Park (VP Marketing)
-- Level 2:         Anna Smith (Marketing Manager)
-- Level 3:             Zoe Wilson (Marketing Coordinator)
-- Level 2:         Tom Brown (Marketing Manager)
```

**Find all reports under a specific manager:**

```sql
DECLARE @ManagerID INT = 2;  -- Mark Torres

WITH Reports AS (
    SELECT EmployeeID, Name, ManagerID, 0 AS Depth
    FROM   OrgChart
    WHERE  EmployeeID = @ManagerID

    UNION ALL

    SELECT e.EmployeeID, e.Name, e.ManagerID, r.Depth + 1
    FROM   OrgChart e
    INNER JOIN Reports r ON e.ManagerID = r.EmployeeID
)
SELECT EmployeeID, Name, Depth
FROM   Reports
WHERE  EmployeeID <> @ManagerID;  -- Exclude the manager themselves
```

---

### Problem 5: Find Gaps in Sequence

**Scenario:** Find missing IDs in an Orders table (IDs should be consecutive).

```sql
-- Setup
CREATE TABLE OrderSeq (
    OrderID INT NOT NULL PRIMARY KEY
);

INSERT INTO OrderSeq VALUES (1),(2),(3),(5),(6),(9),(10),(11),(15);
-- Missing: 4, 7, 8, 12, 13, 14
```

**Approach 1: LAG() to Find Start of Each Gap**

```sql
WITH Gaps AS (
    SELECT OrderID,
           LAG(OrderID) OVER (ORDER BY OrderID) AS PrevID
    FROM   OrderSeq
)
SELECT PrevID + 1    AS GapStart,
       OrderID - 1   AS GapEnd,
       OrderID - PrevID - 1 AS MissingCount
FROM   Gaps
WHERE  OrderID - PrevID > 1;

-- Output:
-- GapStart  GapEnd  MissingCount
-- 4         4       1
-- 7         8       2
-- 12        14      3
```

**Approach 2: Numbers Table (Generate All Expected IDs)**

```sql
-- Generate a sequence using a tally CTE, then LEFT JOIN
DECLARE @MinID INT = (SELECT MIN(OrderID) FROM OrderSeq);
DECLARE @MaxID INT = (SELECT MAX(OrderID) FROM OrderSeq);

WITH Numbers AS (
    SELECT @MinID AS n
    UNION ALL
    SELECT n + 1 FROM Numbers WHERE n < @MaxID
)
SELECT n AS MissingOrderID
FROM   Numbers
WHERE  n NOT IN (SELECT OrderID FROM OrderSeq)
OPTION (MAXRECURSION 10000);

-- Output: 4, 7, 8, 12, 13, 14
```

**Approach 3: Find Missing Dates in a Date Range**

```sql
CREATE TABLE DailyRevenue (
    RevenueDate DATE          NOT NULL PRIMARY KEY,
    Revenue     DECIMAL(10,2) NOT NULL
);

INSERT INTO DailyRevenue VALUES
    ('2024-01-01', 1000), ('2024-01-02', 1500),
    ('2024-01-04', 800),  ('2024-01-07', 2000);
-- Missing: 2024-01-03, 2024-01-05, 2024-01-06

WITH DateRange AS (
    SELECT CAST('2024-01-01' AS DATE) AS d
    UNION ALL
    SELECT DATEADD(DAY, 1, d)
    FROM   DateRange
    WHERE  d < '2024-01-07'
)
SELECT d AS MissingDate
FROM   DateRange
WHERE  d NOT IN (SELECT RevenueDate FROM DailyRevenue)
OPTION (MAXRECURSION 400);
```

---

### Problem 6: Pivot / Cross-Tab

**Scenario:** Transform monthly sales data from rows into columns (one column per month).

```sql
-- Setup: rows of (Year, Month, Category, Revenue)
CREATE TABLE MonthlyCategorySales (
    SaleYear    INT            NOT NULL,
    SaleMonth   INT            NOT NULL,  -- 1-12
    Category    NVARCHAR(50)   NOT NULL,
    Revenue     DECIMAL(10,2)  NOT NULL,
    PRIMARY KEY (SaleYear, SaleMonth, Category)
);

INSERT INTO MonthlyCategorySales VALUES
    (2024, 1, 'Electronics', 50000),
    (2024, 2, 'Electronics', 62000),
    (2024, 3, 'Electronics', 58000),
    (2024, 1, 'Clothing',    30000),
    (2024, 2, 'Clothing',    28000),
    (2024, 3, 'Clothing',    35000),
    (2024, 1, 'Books',       12000),
    (2024, 2, 'Books',       14000),
    (2024, 3, 'Books',       11000);
```

**Approach 1: CASE WHEN (Flexible, Works Everywhere)**

```sql
SELECT Category,
       SUM(CASE WHEN SaleMonth = 1 THEN Revenue ELSE 0 END) AS Jan,
       SUM(CASE WHEN SaleMonth = 2 THEN Revenue ELSE 0 END) AS Feb,
       SUM(CASE WHEN SaleMonth = 3 THEN Revenue ELSE 0 END) AS Mar
FROM   MonthlyCategorySales
WHERE  SaleYear = 2024
GROUP  BY Category
ORDER  BY Category;

-- Output:
-- Category     Jan     Feb     Mar
-- Books      12000   14000   11000
-- Clothing   30000   28000   35000
-- Electronics 50000  62000   58000
```

**Approach 2: PIVOT Operator (SQL Server)**

```sql
SELECT Category, [1] AS Jan, [2] AS Feb, [3] AS Mar
FROM (
    SELECT Category, SaleMonth, Revenue
    FROM   MonthlyCategorySales
    WHERE  SaleYear = 2024
) AS src
PIVOT (
    SUM(Revenue)
    FOR SaleMonth IN ([1], [2], [3])
) AS pvt
ORDER BY Category;
```

**Dynamic PIVOT (for unknown number of columns):**

```sql
DECLARE @Cols   NVARCHAR(MAX);
DECLARE @SQL    NVARCHAR(MAX);
DECLARE @Year   INT = 2024;

-- Build column list dynamically
SELECT @Cols = STRING_AGG(QUOTENAME(SaleMonth), ', ')
FROM (
    SELECT DISTINCT SaleMonth
    FROM   MonthlyCategorySales
    WHERE  SaleYear = @Year
) AS months;

SET @SQL = N'
SELECT Category, ' + @Cols + N'
FROM (
    SELECT Category, SaleMonth, Revenue
    FROM   MonthlyCategorySales
    WHERE  SaleYear = ' + CAST(@Year AS NVARCHAR) + N'
) AS src
PIVOT (
    SUM(Revenue)
    FOR SaleMonth IN (' + @Cols + N')
) AS pvt
ORDER BY Category;';

EXEC sp_executesql @SQL;
```

> **Interview Q: When would you use CASE WHEN vs PIVOT?**
>
> Use CASE WHEN when the columns are known at query-write time, when you need conditions beyond simple aggregation, or for cross-database compatibility. Use PIVOT when you want cleaner syntax for straightforward pivots. Use dynamic PIVOT (with `sp_executesql`) when columns are determined at runtime — but be aware of SQL injection risk; always use `QUOTENAME()` to sanitize column names.

---

## Quick Reference: Interview Q&A

> **Q: What happens when you update a clustered index key column?**
>
> SQL Server performs a delete of the old row and an insert of the new row (a "row move") because the physical ordering must be maintained. This also updates all non-clustered indexes that hold the clustering key as a row locator.

> **Q: Why can a query be slow even with the right indexes?**
>
> Stale statistics causing bad cardinality estimates; non-SARGable predicates preventing seek; implicit type conversions; index fragmentation causing excessive I/O; parameter sniffing giving a plan optimized for different data; blocking or lock waits.

> **Q: What is the difference between DELETE, TRUNCATE, and DROP?**
>
> DELETE is logged row-by-row, fires triggers, can have WHERE clause, can be rolled back. TRUNCATE is minimally logged, removes all rows, resets identity, cannot have WHERE or fire row triggers, can be rolled back if inside an explicit transaction. DROP removes the table schema entirely.

> **Q: What is a covering index and when should you use it?**
>
> A covering index includes all columns a query needs so the engine never has to go back to the clustered index (no Key Lookup). Use it when a query has frequent Key Lookups visible in the execution plan — add the missing columns to INCLUDE.