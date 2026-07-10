# SQL Server (T-SQL) — Complete Developer Guide (Basic → Expert)
### Developer-focused: querying, procs, performance, testing — not DBA/admin topics

---

## Table of Contents

1. [Fundamentals](#1-fundamentals)
2. [Basic Querying](#2-basic-querying)
3. [Joins](#3-joins)
4. [Aggregation & Grouping](#4-aggregation--grouping)
5. [Subqueries & CTEs](#5-subqueries--ctes)
6. [Window Functions](#6-window-functions)
7. [DML — Insert, Update, Delete, Merge](#7-dml--insert-update-delete-merge)
8. [Data Types & Schema Design](#8-data-types--schema-design)
9. [Stored Procedures & Functions](#9-stored-procedures--functions)
10. [Transactions & Error Handling](#10-transactions--error-handling)
11. [Indexing (Developer Perspective)](#11-indexing-developer-perspective)
12. [Query Performance & Execution Plans](#12-query-performance--execution-plans)
13. [Advanced T-SQL](#13-advanced-t-sql)
14. [Security (Developer-Relevant)](#14-security-developer-relevant)
15. [Testing T-SQL](#15-testing-t-sql)
16. [Tricky Interview Questions & Answers](#16-tricky-interview-questions--answers)

---

## 1. Fundamentals

### T-SQL vs standard SQL

T-SQL (Transact-SQL) is Microsoft's SQL dialect — a superset of ANSI SQL with proprietary extensions: variables (`DECLARE @x`), control flow (`IF`/`WHILE`), error handling (`TRY`/`CATCH`), procedural constructs (stored procedures, functions, triggers), and SQL Server–specific functions (`ISNULL`, `GETDATE()`, etc.).

### Batches and `GO`

```sql
DECLARE @x INT = 1;
PRINT @x;
GO   -- batch separator, NOT a T-SQL keyword — it's a client-tool (SSMS/sqlcmd) convention

-- @x does NOT exist here; each batch is compiled/executed independently
```

> **Gotcha:** `GO` is not part of T-SQL itself — it's a signal to the client tool (SSMS, `sqlcmd`) to send everything before it as one batch to the server. Variables, `USE` statements, and some DDL boundaries don't carry across a `GO` — a very common source of confusion for developers coming from other database tools.

### Schemas and object naming

```sql
SELECT * FROM Sales.Orders;   -- Schema.TableName
SELECT * FROM dbo.Products;   -- 'dbo' = default schema
```

Fully qualified: `Server.Database.Schema.Object` — in application code you'll typically only ever need `Schema.Object`.

---

## 2. Basic Querying

```sql
SELECT TOP (10) OrderId, CustomerId, OrderDate
FROM Sales.Orders
WHERE OrderDate >= '2026-01-01'
ORDER BY OrderDate DESC;

SELECT DISTINCT City FROM Customers;
```

### NULL handling — the single most misunderstood area for SQL beginners

```sql
-- WRONG — this NEVER matches, no matter how many rows have NULL Email
SELECT * FROM Customers WHERE Email = NULL;

-- RIGHT
SELECT * FROM Customers WHERE Email IS NULL;
SELECT * FROM Customers WHERE Email IS NOT NULL;

-- Substituting a default when NULL
SELECT Name, ISNULL(Phone, 'N/A') AS Phone FROM Customers;      -- SQL Server-specific
SELECT Name, COALESCE(Phone, MobilePhone, 'N/A') AS Phone FROM Customers; -- ANSI standard, takes N args
```

> **Gotcha:** `NULL = NULL` evaluates to `NULL` (neither true nor false, i.e. "unknown"), not `TRUE` — this is why `WHERE Email = NULL` returns zero rows even for genuinely-NULL emails. It's a constant early-interview trap question.

### `ISNULL` vs `COALESCE` — not just a syntax preference

| | `ISNULL` | `COALESCE` |
|---|---|---|
| Standard | SQL Server–specific | ANSI SQL standard |
| Arguments | Exactly 2 | 2 or more |
| Return type | Type of the **first** argument | Type with highest precedence among all arguments |
| NULL literal handling | `ISNULL(NULL, NULL)` → typed as `int` | `COALESCE(NULL, NULL)` → error, can't determine type |

---

## 3. Joins

```sql
-- INNER JOIN — only matching rows from both sides
SELECT o.OrderId, c.Name
FROM Orders o
INNER JOIN Customers c ON o.CustomerId = c.CustomerId;

-- LEFT OUTER JOIN — all rows from Orders, matching Customer data if it exists (else NULL)
SELECT o.OrderId, c.Name
FROM Orders o
LEFT JOIN Customers c ON o.CustomerId = c.CustomerId;

-- FULL OUTER JOIN — all rows from both sides, NULLs where no match exists on either side
SELECT o.OrderId, c.Name
FROM Orders o
FULL OUTER JOIN Customers c ON o.CustomerId = c.CustomerId;

-- CROSS JOIN — cartesian product, every row of A with every row of B
SELECT s.SizeName, co.ColorName
FROM Sizes s CROSS JOIN Colors co;

-- Self-join — classic use case: employee/manager hierarchy
SELECT e.Name AS Employee, m.Name AS Manager
FROM Employees e
LEFT JOIN Employees m ON e.ManagerId = m.EmployeeId;
```

### `APPLY` — join a table to a table-valued expression per row (no equivalent in standard JOIN)

```sql
-- CROSS APPLY: like INNER JOIN, but the right side can reference columns from the left side
-- Get each customer's 3 most recent orders
SELECT c.Name, recentOrders.OrderId, recentOrders.OrderDate
FROM Customers c
CROSS APPLY (
    SELECT TOP (3) OrderId, OrderDate
    FROM Orders o
    WHERE o.CustomerId = c.CustomerId
    ORDER BY o.OrderDate DESC
) AS recentOrders;

-- OUTER APPLY: like LEFT JOIN — keeps customers even if the subquery returns zero rows
SELECT c.Name, recentOrders.OrderId
FROM Customers c
OUTER APPLY (
    SELECT TOP (3) OrderId FROM Orders o WHERE o.CustomerId = c.CustomerId
) AS recentOrders;
```

> **Gotcha:** A regular `JOIN`'s `ON` clause can't reference a correlated "top N per group" subquery the way `APPLY` can — this is the classic scenario ("get the latest N rows per group") where interviewers expect you to reach for `CROSS APPLY`/`OUTER APPLY` rather than trying to force it through a plain `JOIN`.

---

## 4. Aggregation & Grouping

```sql
SELECT CustomerId, COUNT(*) AS OrderCount, SUM(TotalAmount) AS TotalSpent
FROM Orders
GROUP BY CustomerId
HAVING COUNT(*) > 5;   -- filters on AGGREGATED results — WHERE can't do this
```

> **Gotcha:** `WHERE` filters rows *before* grouping; `HAVING` filters groups *after* aggregation. Trying to write `WHERE COUNT(*) > 5` is a compile error — a near-universal beginner mistake and common interview question.

### `GROUPING SETS`, `ROLLUP`, `CUBE`

```sql
-- Multiple grouping combinations in ONE query instead of UNION-ing several GROUP BYs
SELECT Region, ProductCategory, SUM(Sales) AS TotalSales
FROM SalesData
GROUP BY GROUPING SETS (
    (Region, ProductCategory),  -- subtotal by region+category
    (Region),                    -- subtotal by region only
    ()                            -- grand total
);

-- ROLLUP — hierarchical subtotals (Region → Region+Category → grand total)
SELECT Region, ProductCategory, SUM(Sales) AS TotalSales
FROM SalesData
GROUP BY ROLLUP (Region, ProductCategory);

-- Detecting which "level" a row represents
SELECT Region, ProductCategory, SUM(Sales) AS TotalSales,
       GROUPING(Region) AS IsRegionSubtotal,
       GROUPING(ProductCategory) AS IsCategorySubtotal
FROM SalesData
GROUP BY ROLLUP (Region, ProductCategory);
```

---

## 5. Subqueries & CTEs

### Scalar and correlated subqueries

```sql
-- Scalar subquery — returns exactly one value
SELECT Name, (SELECT AVG(TotalAmount) FROM Orders) AS AvgOrderValue
FROM Customers;

-- Correlated subquery — references the outer query, re-evaluated per outer row
SELECT c.Name
FROM Customers c
WHERE EXISTS (
    SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId AND o.TotalAmount > 1000
);
```

> **Gotcha:** `EXISTS` typically outperforms `IN` with a subquery when NULLs might be present in the subquery's result set — `NOT IN` silently returns **zero rows** if the subquery result contains even a single `NULL`, because `x NOT IN (1, NULL)` evaluates to `UNKNOWN` for every row, not `TRUE`. `NOT EXISTS` doesn't have this trap. This is a very common, very sneaky production bug.

```sql
-- DANGEROUS if CustomerId can be NULL in Orders
SELECT Name FROM Customers
WHERE CustomerId NOT IN (SELECT CustomerId FROM Orders); -- silently returns 0 rows if any Orders.CustomerId IS NULL

-- SAFE
SELECT Name FROM Customers c
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId);
```

### CTEs (Common Table Expressions)

```sql
WITH RecentOrders AS (
    SELECT CustomerId, OrderId, OrderDate,
           ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY OrderDate DESC) AS rn
    FROM Orders
)
SELECT * FROM RecentOrders WHERE rn = 1;   -- each customer's most recent order
```

CTEs are primarily a **readability** tool — they don't materialize results or get cached the way a temp table does; the optimizer typically inlines them into the surrounding query, so a CTE referenced multiple times can be recomputed multiple times (see Section 12).

### Recursive CTEs

```sql
WITH OrgChart AS (
    -- Anchor member
    SELECT EmployeeId, Name, ManagerId, 0 AS Level
    FROM Employees
    WHERE ManagerId IS NULL

    UNION ALL

    -- Recursive member — joins back to the CTE itself
    SELECT e.EmployeeId, e.Name, e.ManagerId, oc.Level + 1
    FROM Employees e
    INNER JOIN OrgChart oc ON e.ManagerId = oc.EmployeeId
)
SELECT * FROM OrgChart
OPTION (MAXRECURSION 100);  -- default cap is 100; 0 = unlimited (use with caution)
```

---

## 6. Window Functions

Window functions compute a value across a "window" of rows related to the current row, without collapsing rows the way `GROUP BY` does.

```sql
SELECT
    OrderId, CustomerId, OrderDate, TotalAmount,
    ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY OrderDate DESC) AS RowNum,
    RANK()       OVER (PARTITION BY CustomerId ORDER BY TotalAmount DESC) AS RankByAmount,
    DENSE_RANK() OVER (PARTITION BY CustomerId ORDER BY TotalAmount DESC) AS DenseRankByAmount,
    SUM(TotalAmount) OVER (PARTITION BY CustomerId ORDER BY OrderDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS RunningTotal,
    LAG(TotalAmount)  OVER (PARTITION BY CustomerId ORDER BY OrderDate) AS PrevOrderAmount,
    LEAD(TotalAmount) OVER (PARTITION BY CustomerId ORDER BY OrderDate) AS NextOrderAmount
FROM Orders;
```

### `ROW_NUMBER` vs `RANK` vs `DENSE_RANK` — the classic tie-breaking question

Given amounts `[100, 100, 90]`:

| Function | Output | Behavior on ties |
|---|---|---|
| `ROW_NUMBER()` | `1, 2, 3` | Always unique, arbitrary order among ties |
| `RANK()` | `1, 1, 3` | Ties share rank, but **skips** the next number |
| `DENSE_RANK()` | `1, 1, 2` | Ties share rank, **no gap** for the next value |

### Deleting duplicates using `ROW_NUMBER` (extremely common real-world pattern)

```sql
WITH Duplicates AS (
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY Email ORDER BY CustomerId
    ) AS rn
    FROM Customers
)
DELETE FROM Duplicates WHERE rn > 1;   -- keep the first row per Email, delete the rest
```

---

## 7. DML — Insert, Update, Delete, Merge

```sql
INSERT INTO Products (Name, Price) VALUES ('Widget', 9.99);

INSERT INTO Products (Name, Price)
OUTPUT inserted.ProductId, inserted.Name   -- capture generated identity values immediately
VALUES ('Gadget', 14.99);

UPDATE Products SET Price = Price * 1.1 WHERE CategoryId = 3;

UPDATE p
SET p.Price = p.Price * 1.1
OUTPUT deleted.Price AS OldPrice, inserted.Price AS NewPrice
FROM Products p
WHERE p.CategoryId = 3;

DELETE FROM Orders WHERE OrderDate < '2020-01-01';
```

### `MERGE` — upsert in a single statement

```sql
MERGE INTO Inventory AS target
USING (SELECT @ProductId AS ProductId, @QuantityChange AS QuantityChange) AS source
ON target.ProductId = source.ProductId
WHEN MATCHED THEN
    UPDATE SET target.Quantity = target.Quantity + source.QuantityChange
WHEN NOT MATCHED THEN
    INSERT (ProductId, Quantity) VALUES (source.ProductId, source.QuantityChange);
```

> **Gotcha:** `MERGE` has had documented edge-case bugs around race conditions under concurrent execution (two sessions both hitting `WHEN NOT MATCHED` simultaneously can both attempt an insert, causing a primary-key violation) — Microsoft's own documentation recommends wrapping `MERGE` in `SERIALIZABLE` isolation or using an explicit lock hint for high-concurrency upsert scenarios, or just using a plain `IF EXISTS(...) UPDATE ELSE INSERT` pattern instead, which is often simpler to reason about.

---

## 8. Data Types & Schema Design

### Choosing the right type

| Need | Prefer | Avoid |
|---|---|---|
| Variable-length Unicode text | `NVARCHAR(n)` | `NTEXT` (deprecated) |
| Variable-length ASCII text | `VARCHAR(n)` | oversized fixed `CHAR(n)` for variable data |
| Money/currency | `DECIMAL(p,s)` or `MONEY` (with care) | `FLOAT`/`REAL` — rounding errors on money math |
| Date only | `DATE` | `DATETIME` when time isn't needed (extra storage + confusion) |
| Date + time, high precision | `DATETIME2` | `DATETIME` (less precision, pre-2008 legacy type) |
| Boolean-like flag | `BIT` | `INT` for true/false semantics |
| Large binary data | `VARBINARY(MAX)` or filestream | storing files as base64 text |

### Constraints

```sql
CREATE TABLE Orders (
    OrderId INT IDENTITY(1,1) PRIMARY KEY,
    CustomerId INT NOT NULL REFERENCES Customers(CustomerId),
    Status VARCHAR(20) NOT NULL DEFAULT 'Pending'
        CHECK (Status IN ('Pending', 'Shipped', 'Cancelled')),
    OrderDate DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    OrderNumber VARCHAR(20) UNIQUE NOT NULL
);
```

### `IDENTITY` vs `SEQUENCE`

```sql
-- IDENTITY — tied to one specific table/column
CREATE TABLE Orders (OrderId INT IDENTITY(1,1) PRIMARY KEY, ...);

-- SEQUENCE — independent object, can be shared across multiple tables,
-- values can be fetched BEFORE the insert
CREATE SEQUENCE OrderNumberSeq START WITH 1000 INCREMENT BY 1;

DECLARE @NextOrderNumber INT = NEXT VALUE FOR OrderNumberSeq;
INSERT INTO Orders (OrderId, OrderNumber) VALUES (@NextOrderNumber, ...);
```

> **Gotcha:** `IDENTITY` values can have gaps (a rolled-back transaction still consumes the identity value — it is NOT transactional), which surprises developers who assume identity columns are always perfectly sequential with no missing numbers. Never rely on `IDENTITY` values being gap-free for business logic (e.g., invoice numbering that must be strictly sequential needs a different mechanism entirely).

### Computed columns

```sql
CREATE TABLE OrderItems (
    Quantity INT,
    UnitPrice DECIMAL(10,2),
    LineTotal AS (Quantity * UnitPrice) PERSISTED   -- PERSISTED = physically stored & indexable
);
```

---

## 9. Stored Procedures & Functions

### Stored procedures

```sql
CREATE PROCEDURE dbo.GetOrdersByCustomer
    @CustomerId INT,
    @Status VARCHAR(20) = NULL   -- optional parameter, default NULL
AS
BEGIN
    SET NOCOUNT ON;   -- suppress "N rows affected" messages — avoids overhead + client confusion

    SELECT OrderId, OrderDate, TotalAmount
    FROM Orders
    WHERE CustomerId = @CustomerId
      AND (@Status IS NULL OR Status = @Status);
END;
```

```sql
-- Output parameters
CREATE PROCEDURE dbo.CreateOrder
    @CustomerId INT,
    @TotalAmount DECIMAL(10,2),
    @NewOrderId INT OUTPUT
AS
BEGIN
    INSERT INTO Orders (CustomerId, TotalAmount) VALUES (@CustomerId, @TotalAmount);
    SET @NewOrderId = SCOPE_IDENTITY();  -- NOT @@IDENTITY — see gotcha below
END;
```

> **Gotcha:** `SCOPE_IDENTITY()` returns the last identity value generated in the **current scope** (this exact stored proc/batch); `@@IDENTITY` returns the last identity value generated in the current **session**, regardless of scope — meaning if an `INSERT` trigger on the table also inserts into a different table with its own identity column, `@@IDENTITY` returns the trigger's generated value instead of the one you actually just inserted. Always prefer `SCOPE_IDENTITY()` unless you have a very specific reason not to.

### Scalar functions vs Table-Valued Functions

```sql
-- Scalar function — returns a single value
CREATE FUNCTION dbo.GetCustomerFullName (@CustomerId INT)
RETURNS NVARCHAR(200)
AS
BEGIN
    DECLARE @Name NVARCHAR(200);
    SELECT @Name = FirstName + ' ' + LastName FROM Customers WHERE CustomerId = @CustomerId;
    RETURN @Name;
END;

-- Inline Table-Valued Function (iTVF) — a parameterized view, gets INLINED into the query plan
CREATE FUNCTION dbo.GetActiveOrdersForCustomer (@CustomerId INT)
RETURNS TABLE
AS
RETURN (
    SELECT OrderId, OrderDate FROM Orders
    WHERE CustomerId = @CustomerId AND Status <> 'Cancelled'
);

-- Multi-statement Table-Valued Function (MSTVF) — NOT inlined, materialized into a table variable
CREATE FUNCTION dbo.GetOrderSummary (@CustomerId INT)
RETURNS @Result TABLE (OrderCount INT, TotalSpent DECIMAL(10,2))
AS
BEGIN
    INSERT INTO @Result
    SELECT COUNT(*), SUM(TotalAmount) FROM Orders WHERE CustomerId = @CustomerId;
    RETURN;
END;
```

> **Gotcha (one of the most important developer-level SQL Server performance facts):** Scalar UDFs and Multi-Statement TVFs are notorious performance killers when used inside a `WHERE` clause or `SELECT` over many rows — because (in most SQL Server versions/compat levels) they execute **row-by-row** rather than being inlined into the set-based query plan, and older versions couldn't even estimate their cost correctly for the optimizer. Inline TVFs don't have this problem — they get expanded into the calling query like a parameterized view and participate fully in set-based optimization. Prefer inline TVFs or a plain `JOIN`/subquery over scalar functions/MSTVFs in performance-sensitive queries. (SQL Server 2019+ introduced "scalar UDF inlining" that automatically fixes many simple scalar-function cases under certain conditions — but don't assume it applies to every function without checking.)

---

## 10. Transactions & Error Handling

### Explicit transactions

```sql
BEGIN TRANSACTION;

UPDATE Inventory SET Quantity = Quantity - 1 WHERE ProductId = 101;
UPDATE Orders SET Status = 'Confirmed' WHERE OrderId = 555;

IF @@ERROR <> 0   -- legacy style — TRY/CATCH below is preferred
    ROLLBACK TRANSACTION;
ELSE
    COMMIT TRANSACTION;
```

### `TRY`/`CATCH` (the modern, preferred approach)

```sql
BEGIN TRY
    BEGIN TRANSACTION;

    UPDATE Inventory SET Quantity = Quantity - 1 WHERE ProductId = 101;
    IF (SELECT Quantity FROM Inventory WHERE ProductId = 101) < 0
        THROW 51000, 'Insufficient inventory.', 1;

    UPDATE Orders SET Status = 'Confirmed' WHERE OrderId = 555;

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;

    DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
    DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
    DECLARE @ErrorState INT = ERROR_STATE();

    THROW;   -- re-throws the ORIGINAL error with original line number/severity — see gotcha
END CATCH;
```

### `THROW` vs `RAISERROR`

| | `THROW` | `RAISERROR` |
|---|---|---|
| Introduced | SQL Server 2012+ | Legacy (all versions) |
| Re-throw original error in CATCH | `THROW;` (no args) preserves original error number/message/line | Cannot re-throw the original error faithfully |
| Custom error message format strings | No (must pre-format the string) | Yes (`RAISERROR('Value: %d', 16, 1, @val)`) |
| Severity | Always terminates batch at severity matching original error (or 16 if you specify one) | You control severity explicitly, can be non-terminating |

> **Gotcha:** `RAISERROR` inside a `CATCH` block does **not** preserve the original exception's line number, procedure name, or exact error number the way bare `THROW;` does — a very common "why did my error logging lose the original context" issue when teams migrate old `RAISERROR`-based error handling to modern patterns without switching to `THROW`.

### Isolation levels (developer-relevant subset)

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- SQL Server default
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;  -- "dirty reads" — avoid in business logic
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;           -- optimistic, row-versioning based, no blocking reads
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;       -- strictest, most blocking/locking
```

`READ COMMITTED SNAPSHOT` (a database-level setting, distinct from session-level `SNAPSHOT` isolation) is widely enabled in modern application databases specifically to reduce reader/writer blocking without changing application code — worth knowing exists even though enabling it is technically a DBA-level config change.

---

## 11. Indexing (Developer Perspective)

### Clustered vs Nonclustered

```sql
-- Clustered index determines the PHYSICAL order of table data — a table has AT MOST one
CREATE CLUSTERED INDEX IX_Orders_OrderDate ON Orders (OrderDate);

-- Nonclustered index is a separate structure with pointers back to the table/clustered key
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId ON Orders (CustomerId);
```

A `PRIMARY KEY` creates a clustered index **by default** unless you explicitly specify `NONCLUSTERED` — a detail worth knowing since it affects physical row ordering, not just uniqueness.

### Covering indexes with `INCLUDE`

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId_Covering
ON Orders (CustomerId)
INCLUDE (OrderDate, TotalAmount);
```

A query that only touches `CustomerId`, `OrderDate`, and `TotalAmount` can be satisfied entirely from this index without a "key lookup" back to the clustered index — this is the single biggest lever a developer has over query performance without touching application code.

### SARGable predicates (Search ARGument-able)

```sql
-- NOT SARGable — wrapping the column in a function prevents index usage on YearColumn
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2026;

-- SARGable — the column itself is untouched, index CAN be used
SELECT * FROM Orders WHERE OrderDate >= '2026-01-01' AND OrderDate < '2027-01-01';

-- NOT SARGable — leading wildcard prevents index seek, forces scan
SELECT * FROM Customers WHERE Name LIKE '%smith';

-- SARGable — leading characters known, index seek possible
SELECT * FROM Customers WHERE Name LIKE 'smith%';
```

> **Gotcha:** Applying ANY function to an indexed column in a `WHERE` clause (`YEAR()`, `UPPER()`, string concatenation, arithmetic, implicit type conversion) typically defeats index usage for that predicate, forcing a full scan — this single concept accounts for an enormous fraction of real-world "why is this simple query slow" tickets, and is one of the most frequently tested SQL Server interview concepts.

### Implicit conversion — a sneaky SARGability killer

```sql
-- Customers.Phone is VARCHAR, @phone below is accidentally NVARCHAR
DECLARE @phone NVARCHAR(20) = '555-1234';
SELECT * FROM Customers WHERE Phone = @phone; -- may silently convert Phone on EVERY row, killing index use
```

Mismatched data types between a parameter and an indexed column can silently force SQL Server to convert the *column* (not the parameter) for comparison, which defeats the index across the entire table even though the query "looks" fine.

---

## 12. Query Performance & Execution Plans

### Reading an execution plan (conceptually)

- **Estimated Execution Plan**: what the optimizer *thinks* will happen, based on statistics — doesn't actually run the query.
- **Actual Execution Plan**: includes real row counts and actual timing — always prefer this when diagnosing a real slow query, since large gaps between *estimated* and *actual* row counts are a huge red flag (usually stale statistics).

Key operators to recognize:

| Operator | Meaning | Concern level |
|---|---|---|
| Index Seek | Direct, targeted lookup using an index | Good |
| Index Scan | Reading the entire index, not a targeted lookup | Investigate — often means a missing/unusable index for this predicate |
| Table Scan | Reading the entire table (no useful index at all) | Usually bad on large tables |
| Key Lookup | Nonclustered index found the row, but needs extra columns from the clustered index | Frequent = consider a covering index (`INCLUDE`) |
| Sort | Explicit sort operation | Can sometimes be avoided by an index matching the `ORDER BY` |
| Hash Match | Hash-based join/aggregate | Fine for large unsorted sets, but can be memory-intensive |

### Statistics — why the optimizer's guess can be badly wrong

```sql
UPDATE STATISTICS Orders;   -- refresh statistics manually (normally auto-updated, but can lag)

-- Check when stats were last updated & how many rows sampled
SELECT * FROM sys.dm_db_stats_properties(OBJECT_ID('Orders'), 1);
```

Stale statistics (common after large bulk loads/deletes) make the optimizer misjudge row counts, leading it to pick a bad join strategy (e.g., a nested loop join expecting 10 rows when there are actually 10 million) — a classic cause of a query that was fast yesterday suddenly being slow today with no code change.

### `SET STATISTICS IO, TIME ON` — a developer's fastest diagnostic tool

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT * FROM Orders WHERE CustomerId = 42;

-- Output includes "logical reads" per table — a very reliable, environment-independent
-- performance signal, unlike wall-clock time which varies by server load
```

### Common anti-patterns recap

- Scalar UDFs/MSTVFs in `WHERE`/`SELECT` over large row counts (Section 9)
- Non-SARGable predicates — functions wrapping indexed columns (Section 11)
- `SELECT *` instead of only the needed columns (defeats covering indexes, pulls unnecessary data over the wire)
- Implicit conversions from mismatched parameter/column types
- Excessive use of scalar subqueries in the `SELECT` list evaluated per row instead of a `JOIN`

---

## 13. Advanced T-SQL

### Dynamic SQL — `sp_executesql` vs `EXEC`

```sql
-- EXEC — string concatenation, injection risk if input isn't controlled
DECLARE @sql NVARCHAR(MAX) = 'SELECT * FROM Orders WHERE CustomerId = ' + @CustomerId;
EXEC (@sql);

-- sp_executesql — parameterized, safe, AND enables plan reuse across calls with different values
DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM Orders WHERE CustomerId = @CustomerId';
EXEC sp_executesql @sql, N'@CustomerId INT', @CustomerId = @CustomerId;
```

> **Gotcha:** Beyond the obvious SQL-injection risk, plain `EXEC` string concatenation also hurts performance at scale — every slightly different literal value produces a textually different SQL string, so the plan cache treats each one as a brand-new query needing a fresh compile ("plan cache bloat"). `sp_executesql` with real parameters lets SQL Server reuse a single cached, parameterized plan across many calls.

### `PIVOT` / `UNPIVOT`

```sql
SELECT * FROM (
    SELECT ProductCategory, YEAR(OrderDate) AS OrderYear, TotalAmount
    FROM Orders
) AS src
PIVOT (
    SUM(TotalAmount) FOR OrderYear IN ([2024], [2025], [2026])
) AS pvt;
```

### JSON support (SQL Server 2016+)

```sql
-- FOR JSON — shape relational results as JSON text
SELECT OrderId, CustomerId, OrderDate
FROM Orders
FOR JSON AUTO;

-- OPENJSON — parse incoming JSON into a relational rowset
DECLARE @json NVARCHAR(MAX) = N'[{"ProductId":1,"Qty":2},{"ProductId":2,"Qty":5}]';

SELECT ProductId, Qty
FROM OPENJSON(@json)
WITH (ProductId INT '$.ProductId', Qty INT '$.Qty');
```

> Note: SQL Server has no dedicated native JSON data type (unlike PostgreSQL's `jsonb`) — JSON is stored/manipulated as `NVARCHAR(MAX)` text with these functions operating on it; this is a common "gotcha" for developers coming from Postgres expecting a proper binary JSON type.

### Temporal tables (system-versioned, built-in row history)

```sql
CREATE TABLE Products (
    ProductId INT PRIMARY KEY,
    Name NVARCHAR(200),
    Price DECIMAL(10,2),
    ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START,
    ValidTo DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductsHistory));

-- Query the table as it looked at a specific point in time — no app code needed to track history
SELECT * FROM Products
FOR SYSTEM_TIME AS OF '2026-01-01T00:00:00'
WHERE ProductId = 42;
```

### `TRY_CONVERT` / `TRY_CAST` / `TRY_PARSE`

```sql
SELECT TRY_CONVERT(INT, 'abc');   -- returns NULL instead of throwing an error
SELECT TRY_CAST('2026-13-40' AS DATE);  -- returns NULL for an invalid date instead of erroring
```

Useful for defensively parsing untrusted/messy data without wrapping every conversion in `TRY`/`CATCH`.

---

## 14. Security (Developer-Relevant)

### SQL injection — the classic developer-facing risk

```sql
-- VULNERABLE — never build queries via string concatenation of user input
string sql = $"SELECT * FROM Users WHERE Username = '{userInput}'";

-- Parameterized — the ONLY correct way, regardless of language/data access technology
-- (ADO.NET example, but the principle applies everywhere: EF Core, Dapper, raw ADO.NET)
var cmd = new SqlCommand("SELECT * FROM Users WHERE Username = @username", connection);
cmd.Parameters.AddWithValue("@username", userInput);
```

Parameters are sent to SQL Server separately from the query text — the database never interprets user input as part of the SQL grammar itself, which is what prevents injection regardless of how malicious the input string looks.

### Principle of least privilege at the object level

```sql
-- Grant only what's needed for a specific stored procedure/table, not blanket db_owner access
GRANT EXECUTE ON dbo.GetOrdersByCustomer TO AppUser;
GRANT SELECT, INSERT, UPDATE ON dbo.Orders TO AppUser;
DENY DELETE ON dbo.Orders TO AppUser;
```

Even as an application developer (not a DBA), understanding that your application's SQL login should have exactly the permissions its code paths require — not more — matters because it's the difference between an injection bug being merely embarrassing versus catastrophic.

### Dynamic SQL + injection double risk

Dynamic SQL built via string concatenation (Section 13) is a double risk: it can create both a SQL-injection vulnerability AND plan-cache bloat. Always prefer `sp_executesql` with real parameters over `EXEC` with concatenated strings, for both reasons simultaneously.

### Views and column-level security as a lightweight app-facing safeguard

```sql
CREATE VIEW dbo.PublicCustomerInfo AS
SELECT CustomerId, Name, City   -- deliberately excludes SSN, CreditCardToken, etc.
FROM Customers;

GRANT SELECT ON dbo.PublicCustomerInfo TO ReportingAppUser;
```

Exposing a narrowly-scoped view (rather than direct table access) to certain application roles is a simple, developer-friendly way to enforce "this code path should never even be able to see this column" at the database layer itself, as defense-in-depth beyond just careful application code.

---

## 15. Testing T-SQL

### `tSQLt` — the standard T-SQL unit testing framework

```sql
EXEC tSQLt.NewTestClass 'OrderTests';
GO

CREATE PROCEDURE OrderTests.[test CreateOrder inserts a row with Pending status]
AS
BEGIN
    -- Arrange — tSQLt.FakeTable replaces the real table with an identical, empty, unconstrained one
    EXEC tSQLt.FakeTable 'dbo.Orders';

    -- Act
    EXEC dbo.CreateOrder @CustomerId = 1, @TotalAmount = 50.00;

    -- Assert
    DECLARE @ActualStatus VARCHAR(20) = (SELECT Status FROM dbo.Orders);
    EXEC tSQLt.AssertEquals @Expected = 'Pending', @Actual = @ActualStatus;
END;
GO

EXEC tSQLt.RunAll;   -- or EXEC tSQLt.Run 'OrderTests';
```

`tSQLt.FakeTable` is the key mechanism — it swaps in an empty, constraint-free shadow table for the duration of the test so you can test a stored procedure's logic in complete isolation from real data and without needing to clean up afterward (each test typically runs inside a transaction that's rolled back automatically).

### Testing considerations specific to T-SQL

- Prefer testing stored procedures/functions at the T-SQL layer for logic that's inherently set-based or hard to meaningfully express in application-layer unit tests (complex `MERGE` logic, recursive CTEs, window-function-based calculations).
- For anything that's really just "does this query return the right rows," an integration test at the application layer (as covered in the EF Core guide, Section 11) against a real or container-based SQL Server instance often gives equivalent confidence with less specialized tooling — `tSQLt` earns its keep specifically for procedural logic living inside stored procedures/functions themselves.

---

## 16. Tricky Interview Questions & Answers

**Q1: Why does `WHERE Email = NULL` always return zero rows, even for genuinely NULL values?**
A: In SQL's three-valued logic, any comparison involving `NULL` (including `NULL = NULL`) evaluates to `UNKNOWN`, not `TRUE` — and `WHERE` only keeps rows where the condition evaluates to `TRUE`. You must use `IS NULL`/`IS NOT NULL`, which are special predicates designed specifically to test for NULL rather than relying on the `=` operator.

**Q2: Why can `NOT IN` silently return zero rows when you expect matches, while `NOT EXISTS` doesn't have this problem?**
A: If the subquery inside `NOT IN` returns even a single `NULL` value, the comparison `x NOT IN (1, 2, NULL)` evaluates to `UNKNOWN` for every row (because it's logically "x <> 1 AND x <> 2 AND x <> NULL", and that last comparison is always `UNKNOWN`, which poisons the whole `AND` chain) — so the entire query returns zero rows regardless of what `x` actually is. `NOT EXISTS` uses a correlated existence check rather than a value-list comparison, so it isn't affected by NULLs in the subquery's result set the same way.

**Q3: What's the real difference between `RANK()` and `DENSE_RANK()` when there are ties?**
A: Both assign the same rank number to tied rows, but `RANK()` then skips ahead by the number of tied rows (so `1, 1, 3` for two rows tied at rank 1), while `DENSE_RANK()` leaves no gap (`1, 1, 2`). `ROW_NUMBER()` ignores ties entirely and always assigns strictly increasing unique numbers.

**Q4: Why are scalar UDFs and multi-statement table-valued functions often disastrous for performance when used in a `WHERE` clause over a large table?**
A: In most SQL Server versions/database compatibility levels, scalar UDFs and MSTVFs are executed row-by-row rather than being inlined into the surrounding set-based query plan — turning what looks like a simple filtered `SELECT` into effectively N separate function invocations, one per candidate row, with the optimizer often unable to estimate their true cost. Inline table-valued functions don't have this problem because they get expanded/inlined like a parameterized view.

**Q5: What's the difference between `SCOPE_IDENTITY()`, `@@IDENTITY`, and `IDENT_CURRENT()`?**
A: `SCOPE_IDENTITY()` returns the last identity value generated in the current scope (the current stored procedure/batch) — the one you almost always want. `@@IDENTITY` returns the last identity value generated in the current *session*, regardless of scope, which can return a value from a trigger's insert into a completely different table instead of the one you actually intended. `IDENT_CURRENT('TableName')` returns the last identity value generated for a *specific table*, regardless of session or scope — which means it can return a value inserted by a completely different user/connection, making it the least safe of the three for typical "get the ID I just inserted" use cases.

**Q6: Why is applying a function like `YEAR(OrderDate)` in a `WHERE` clause considered a performance anti-pattern, even though it's readable and correct?**
A: Wrapping an indexed column in a function makes the predicate non-SARGable — the optimizer generally can't use an index seek on `OrderDate` anymore, because it would have to evaluate `YEAR()` for every single row to know which ones match, forcing a full scan/index scan instead. Rewriting it as a range predicate (`OrderDate >= '2026-01-01' AND OrderDate < '2027-01-01'`) leaves the column untouched and keeps it fully SARGable.

**Q7: What's the difference between a clustered and nonclustered index, and how many of each can a table have?**
A: A clustered index determines the actual physical storage order of the table's rows — there can be at most **one** per table (since data can only be physically sorted one way). A nonclustered index is a separate, independent structure that stores a copy of the indexed columns plus a pointer back to the corresponding row in the clustered index (or a row identifier if the table is a heap with no clustered index) — a table can have many nonclustered indexes.

**Q8: What is a "key lookup" in an execution plan, and how do you typically eliminate it?**
A: It occurs when a nonclustered index seek finds the matching row(s) but the query needs additional columns that aren't part of that index, forcing an extra lookup back into the clustered index (or heap) per matching row to fetch them. You typically eliminate it by adding the needed extra columns to the nonclustered index via `INCLUDE (...)`, turning it into a "covering index" that can satisfy the entire query without the extra lookup.

**Q9: Why might a query that ran fast last week suddenly be slow today with no code changes at all?**
A: The most common cause is stale or newly-invalidated statistics after significant data changes (large inserts/deletes/updates) — the query optimizer's row-count estimates become wrong, leading it to choose a different (worse) join strategy or operator than before (e.g., switching from a hash join to a nested loop join that's now inappropriate for the actual data volume). Comparing estimated vs. actual row counts in the execution plan is the standard first diagnostic step; a huge gap between them points straight at statistics.

**Q10: What's the practical difference between a CTE and a temp table for performance purposes?**
A: A CTE is primarily a syntactic/readability construct — in most cases the optimizer inlines it directly into the surrounding query as if you'd written a subquery, meaning a CTE referenced multiple times in the same query can be computed multiple times rather than cached. A temp table (`#TempTable`) actually materializes and physically stores its result set once, which can be explicitly indexed and reused efficiently across multiple subsequent statements in the same batch/procedure — the right choice when the same intermediate result set needs to be queried repeatedly or is expensive to recompute.

**Q11: Why does `THROW;` (with no arguments) inside a `CATCH` block behave differently from `RAISERROR` when re-raising an error?**
A: Bare `THROW;` re-raises the *exact original* exception — same error number, message, severity, and line number — exactly as SQL Server generated it. `RAISERROR`, even when fed values pulled from `ERROR_MESSAGE()`/`ERROR_NUMBER()`/etc., generates a *new* error and cannot fully replicate the original's identity (notably the original line number and, in older syntax forms, sometimes the original error number itself gets renumbered to a generic one) — losing fidelity that matters for debugging and centralized error logging.

**Q12: What's wrong with using `EXEC(@sql)` with string-concatenated dynamic SQL, beyond the SQL-injection risk?**
A: Even setting security aside, string-concatenated dynamic SQL defeats plan-cache reuse — every slightly different literal value embedded directly into the SQL text produces a textually distinct query string, so SQL Server treats each execution as brand-new and must compile a fresh plan every time ("plan cache bloat"), wasting CPU and cache space. `sp_executesql` with genuine parameters lets many different calls share one cached, parameterized plan.

**Q13: Why can mismatched data types between a query parameter and an indexed column silently ruin performance, even when the query returns correct results?**
A: SQL Server has type-precedence rules determining which side of a comparison gets implicitly converted when types don't match. If the *column* (not the parameter) is the one implicitly converted, that conversion effectively gets applied to every row's value during evaluation, which — just like wrapping the column in an explicit function — makes the predicate non-SARGable and defeats index seeks across the whole table, even though the query is functionally correct and returns the right answer.

**Q14: Why is `IDENTITY` alone not a safe mechanism for a business requirement like "invoice numbers must be strictly sequential with no gaps"?**
A: `IDENTITY` value generation is not transactional/rollback-safe — if a transaction that inserted a row (consuming an identity value) is later rolled back, that identity value is still permanently "used up" and never reissued, creating a gap. Any business requirement demanding strictly gapless sequential numbers needs an entirely different mechanism (e.g., a manually managed counter table with careful locking, generated and validated as part of a controlled, possibly serialized process) rather than relying on `IDENTITY`/`SEQUENCE` alone.

**Q15: What's a realistic risk with using `MERGE` for a concurrent upsert scenario, and what's a common simpler alternative?**
A: Under concurrent execution, two sessions can both evaluate `WHEN NOT MATCHED` as true for the same key at nearly the same time (since the check and the subsequent insert aren't atomic against each other without extra locking), both attempt to insert, and one fails with a primary-key/unique-constraint violation — a documented, known edge case in SQL Server's `MERGE` implementation under default isolation. A common simpler and often equally effective alternative is a straightforward `IF EXISTS (...) UPDATE ... ELSE INSERT ...` pattern, sometimes combined with an explicit `HOLDLOCK`/higher isolation level, or a `TRY/CATCH` around the insert that falls back to an update on a caught primary-key violation.

**Q16: Why would you choose an inline table-valued function over a scalar function that returns similar logic wrapped as a single value?**
A: An inline TVF's body is a single `RETURN (SELECT ...)` statement that the optimizer can inline/expand directly into the calling query, just like a parameterized view — it participates fully in set-based query optimization (index usage, join reordering, etc.). A scalar function invoked per-row in a `SELECT`/`WHERE` doesn't get this treatment in most versions/compatibility levels and instead executes as a black-box row-by-row call, which is far more expensive at scale even if the underlying logic is equivalent.

**Q17: SQL Server has no native JSON data type like PostgreSQL's `jsonb` — what does this actually mean in practice for a developer?**
A: JSON data in SQL Server is stored and manipulated as plain `NVARCHAR(MAX)` text — `FOR JSON`/`OPENJSON` provide convenient functions to convert to/from that text representation and relational rowsets, but there's no dedicated binary storage format, no native JSON-specific indexing structure, and no type-level validation that a column's text is actually well-formed JSON unless you add a `CHECK (ISJSON(column) = 1)` constraint yourself.

**Q18: Why does `SET STATISTICS IO ON` often give a more reliable performance signal than simply timing how long a query takes to run?**
A: Wall-clock execution time is heavily affected by transient factors — server load from other queries, caching state (cold vs. warm buffer pool), network latency to the client — that vary run-to-run and make before/after comparisons noisy. Logical reads (the count `STATISTICS IO` reports) measure how many pages the query actually had to touch logically, which is a stable, environment-independent number for a given query/data/plan combination, making it far more reliable for judging whether an optimization actually helped.

**Q19: If two application developers disagree about whether to put business logic in a stored procedure or in application code, what's a reasonable framework for deciding?**
A: Favor stored procedures for logic that's inherently set-based, needs to run close to the data to avoid excessive round trips (bulk operations, complex multi-table validation), or must be shared identically across multiple different application/consumers hitting the same database. Favor application-layer logic for anything benefiting from unit testing in the primary application language/tooling, business rules that change frequently and benefit from application-level deployment/versioning (rather than a separate DB migration path), or logic that needs to call out to non-database services. Many real systems land on a hybrid: simple CRUD and validation in the app layer (as in the EF Core guide), with select performance-critical or heavily-shared operations pushed into stored procedures.

**Q20: Why might `HAVING COUNT(*) > 5` work fine but `WHERE COUNT(*) > 5` fail to even compile?**
A: `WHERE` is evaluated *before* `GROUP BY` groups rows together and computes aggregates — at that point in logical query processing, there's no such thing as a per-group `COUNT(*)` yet, only individual rows, so referencing an aggregate function in `WHERE` is simply invalid syntax. `HAVING` is evaluated *after* grouping/aggregation, exactly when per-group aggregate values like `COUNT(*)` actually exist to filter on.
