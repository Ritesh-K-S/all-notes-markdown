# ⚡ Chapter 2C.3 — T-SQL — SQL Server's SQL Dialect

> **"SQL is the universal language. T-SQL is SQL Server speaking it with a New York accent — faster, bolder, with extra slang."**

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 2A (SQL Language Mastery — at least 2.1 through 2.6)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Know every **T-SQL-specific** feature that doesn't exist in standard SQL
- Use `TOP`, `CROSS APPLY`, `OUTER APPLY` — SQL Server's secret weapons
- Write bulletproof error handling with `TRY...CATCH`
- Master `OFFSET-FETCH` for modern pagination
- Use `STRING_AGG`, `STRING_SPLIT`, `FORMAT`, `IIF`, `CHOOSE`
- Write **table-valued parameters**, **MERGE**, and **OUTPUT** clause
- Handle **temp tables vs table variables vs CTEs** like a pro
- Know what's T-SQL-only vs standard SQL (critical for interviews)

---

## 🗺️ T-SQL vs Standard SQL — What's Different?

```
╔══════════════════════════════════════════════════════════════════╗
║              STANDARD SQL vs T-SQL (Transact-SQL)                ║
║                                                                  ║
║   Standard SQL (ISO/ANSI)         T-SQL (Microsoft Extensions)  ║
║   ─────────────────────          ──────────────────────────────  ║
║   SELECT, FROM, WHERE            Same ✅                        ║
║   JOIN, GROUP BY, HAVING         Same ✅                        ║
║   CREATE, ALTER, DROP            Same ✅ (+ extras)             ║
║   INSERT, UPDATE, DELETE         Same ✅ (+ OUTPUT clause)      ║
║                                                                  ║
║   ❌ No TOP                      ✅ TOP n / TOP n PERCENT      ║
║   ❌ No CROSS/OUTER APPLY        ✅ APPLY operator             ║
║   ❌ No TRY...CATCH              ✅ Structured error handling   ║
║   ❌ No IIF() / CHOOSE()         ✅ Shorthand conditionals      ║
║   ❌ No FORMAT()                  ✅ .NET-style formatting       ║
║   ❌ No STRING_AGG (until 2017)   ✅ Since SQL Server 2017      ║
║   LIMIT n (MySQL/PG)             ❌ Use TOP or OFFSET-FETCH    ║
║   || for concat (PG/Oracle)      + for concat (T-SQL)          ║
║   NVL (Oracle)                   ISNULL / COALESCE (T-SQL)     ║
║   SEQUENCE (Standard)            IDENTITY + SEQUENCE (both!)   ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔝 1. TOP — Limiting Results (SQL Server Style)

In MySQL/PostgreSQL you use `LIMIT`. In SQL Server, you use `TOP`.

```sql
-- Basic TOP
SELECT TOP 10 * FROM Products ORDER BY Price DESC;

-- TOP with PERCENT
SELECT TOP 10 PERCENT * FROM Products ORDER BY Price DESC;

-- TOP with TIES (include rows with same value as last row)
SELECT TOP 5 WITH TIES *
FROM Products
ORDER BY Price DESC;
-- If 5th and 6th products have same price, BOTH are included!

-- TOP with variable (dynamic TOP)
DECLARE @n INT = 10;
SELECT TOP (@n) * FROM Products ORDER BY Price DESC;

-- TOP in UPDATE/DELETE (process in batches!)
DELETE TOP (1000) FROM Logs WHERE LogDate < '2025-01-01';
-- Deletes only 1000 rows at a time — prevents lock escalation!

-- TOP 1 vs SET ROWCOUNT (deprecated — don't use SET ROWCOUNT)
SELECT TOP 1 * FROM Products ORDER BY Price DESC;  -- ✅ Modern way
```

### TOP vs OFFSET-FETCH

```sql
-- 🟢 Modern Pagination: OFFSET-FETCH (SQL Server 2012+)
-- "Give me page 3, with 20 items per page"

SELECT ProductID, ProductName, Price
FROM Products
ORDER BY ProductID              -- ⚠️ ORDER BY is REQUIRED!
OFFSET 40 ROWS                 -- Skip first 40 rows (pages 1-2)
FETCH NEXT 20 ROWS ONLY;       -- Return next 20 rows (page 3)

-- Dynamic pagination
DECLARE @PageNumber INT = 3;
DECLARE @PageSize INT = 20;

SELECT ProductID, ProductName, Price
FROM Products
ORDER BY ProductID
OFFSET (@PageNumber - 1) * @PageSize ROWS
FETCH NEXT @PageSize ROWS ONLY;
```

### Comparison: How Different DBs Limit Rows

| Database | Syntax | Example |
|----------|--------|---------|
| **SQL Server** | `TOP n` or `OFFSET-FETCH` | `SELECT TOP 10 * FROM t` |
| **MySQL** | `LIMIT n` | `SELECT * FROM t LIMIT 10` |
| **PostgreSQL** | `LIMIT n` | `SELECT * FROM t LIMIT 10` |
| **Oracle** | `FETCH FIRST n ROWS ONLY` (12c+) | `SELECT * FROM t FETCH FIRST 10 ROWS ONLY` |

---

## 🔀 2. CROSS APPLY & OUTER APPLY — SQL Server's Secret Weapon

`APPLY` is one of SQL Server's most **powerful** and **unique** features. Think of it as a **"for each row, call this function"** operator.

### The Concept

```
╔══════════════════════════════════════════════════════════════════╗
║                    APPLY OPERATOR                                ║
║                                                                  ║
║   Regular JOIN:                                                  ║
║     "Match rows between two INDEPENDENT tables"                 ║
║                                                                  ║
║   APPLY:                                                         ║
║     "For EACH row in the left table, evaluate the right          ║
║      expression (which CAN reference the left row!)"            ║
║                                                                  ║
║   ┌──────────┐    CROSS APPLY    ┌──────────────────┐           ║
║   │ Left     │ ──────────────→   │ Right expression │           ║
║   │ Table    │   (for each row)  │ (can reference   │           ║
║   │          │                   │  left table!)    │           ║
║   └──────────┘                   └──────────────────┘           ║
║                                                                  ║
║   CROSS APPLY = Like INNER JOIN (skip if right returns nothing) ║
║   OUTER APPLY = Like LEFT JOIN (keep row, NULLs if nothing)    ║
╚══════════════════════════════════════════════════════════════════╝
```

### Example 1: Top N per Group (Classic Use Case)

```sql
-- "For each department, get the top 3 highest-paid employees"

-- ❌ Without APPLY (painful with subqueries)
-- ✅ With CROSS APPLY (elegant!)

SELECT d.DepartmentName, e.EmployeeName, e.Salary
FROM Departments d
CROSS APPLY (
    SELECT TOP 3 EmployeeName, Salary
    FROM Employees e
    WHERE e.DepartmentID = d.DepartmentID  -- ← References outer table!
    ORDER BY Salary DESC
) e;

-- Result:
-- Engineering  │ Alice   │ 150,000
-- Engineering  │ Bob     │ 140,000
-- Engineering  │ Charlie │ 135,000
-- Marketing    │ Diana   │ 120,000
-- Marketing    │ Eve     │ 115,000
-- Marketing    │ Frank   │ 110,000
```

### Example 2: CROSS APPLY with Table-Valued Function

```sql
-- Create a function that returns recent orders for a customer
CREATE FUNCTION dbo.GetRecentOrders(@CustomerID INT, @Count INT)
RETURNS TABLE
AS
RETURN (
    SELECT TOP (@Count) OrderID, OrderDate, TotalAmount
    FROM Orders
    WHERE CustomerID = @CustomerID
    ORDER BY OrderDate DESC
);
GO

-- Use CROSS APPLY to call it for EACH customer
SELECT c.CustomerName, o.OrderID, o.OrderDate, o.TotalAmount
FROM Customers c
CROSS APPLY dbo.GetRecentOrders(c.CustomerID, 3) o;
-- Returns last 3 orders for each customer!
```

### Example 3: OUTER APPLY (Keep Rows with No Match)

```sql
-- Show ALL customers, even those with no orders
SELECT c.CustomerName, o.OrderID, o.TotalAmount
FROM Customers c
OUTER APPLY (
    SELECT TOP 1 OrderID, TotalAmount
    FROM Orders
    WHERE CustomerID = c.CustomerID
    ORDER BY OrderDate DESC
) o;

-- Result:
-- Alice   │ 10248  │ 440.00     ← Has orders
-- Bob     │ 10249  │ 1863.40    ← Has orders
-- Charlie │ NULL   │ NULL       ← No orders (still shown!)
```

### CROSS APPLY vs JOIN — When to Use Which?

| Scenario | Use JOIN | Use APPLY |
|----------|----------|-----------|
| Simple table-to-table matching | ✅ | Overkill |
| Top N per group | Complex | ✅ Easy |
| Call table-valued function per row | ❌ Can't | ✅ Built for this |
| Unpivot columns to rows | Possible | ✅ Cleaner |
| Correlated subquery returning multiple rows | ❌ | ✅ |
| Performance with small right-side results | Similar | ✅ Often faster |

---

## 🛡️ 3. TRY...CATCH — Structured Error Handling

Standard SQL has no structured error handling. T-SQL gives you `TRY...CATCH`:

```sql
-- 💡 The Basic Pattern
BEGIN TRY
    -- Your risky code here
    BEGIN TRANSACTION;
        
        UPDATE Accounts SET Balance = Balance - 500 WHERE AccountID = 1;
        UPDATE Accounts SET Balance = Balance + 500 WHERE AccountID = 2;
    
    COMMIT TRANSACTION;
    PRINT 'Transfer successful! ✅';
END TRY
BEGIN CATCH
    -- If ANYTHING goes wrong, we end up here
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
    
    -- Get error details
    SELECT
        ERROR_NUMBER()    AS ErrorNumber,     -- e.g., 547
        ERROR_SEVERITY()  AS ErrorSeverity,   -- e.g., 16
        ERROR_STATE()     AS ErrorState,      -- e.g., 0
        ERROR_PROCEDURE() AS ErrorProcedure,  -- SP name (if in one)
        ERROR_LINE()      AS ErrorLine,       -- Line number
        ERROR_MESSAGE()   AS ErrorMessage;    -- Human-readable message
    
    -- Re-throw the error (SQL Server 2012+)
    THROW;  -- ← Preserves original error info!
END CATCH;
```

### Error Functions Available in CATCH Block

| Function | Returns | Example Value |
|----------|---------|---------------|
| `ERROR_NUMBER()` | Error code | `547` (FK violation) |
| `ERROR_MESSAGE()` | Human-readable message | `"The INSERT statement conflicted with..."` |
| `ERROR_SEVERITY()` | Severity level (0-25) | `16` (user error) |
| `ERROR_STATE()` | Error state | `0` |
| `ERROR_LINE()` | Line number where error occurred | `14` |
| `ERROR_PROCEDURE()` | Stored procedure name | `usp_TransferFunds` |

### THROW vs RAISERROR

```sql
-- 🟢 THROW (SQL Server 2012+) — Modern, preferred
BEGIN CATCH
    THROW;  -- Re-throws the original error AS-IS
END CATCH;

-- Custom error with THROW
THROW 50001, 'Custom error: Insufficient funds!', 1;

-- 🟡 RAISERROR — Legacy but still useful for custom severity
RAISERROR('Insufficient funds for account %d', 16, 1, @AccountID);

-- Key Differences:
-- THROW: Always severity 16, simpler syntax, re-throws original error
-- RAISERROR: Custom severity, printf-style formatting, doesn't re-throw
```

### Nested TRY...CATCH with Transaction Patterns

```sql
-- 💡 Production-Grade Error Handling Pattern
CREATE PROCEDURE dbo.usp_TransferFunds
    @FromAccount INT,
    @ToAccount INT,
    @Amount DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;  -- ← Auto-rollback on ANY error!
    
    BEGIN TRY
        BEGIN TRANSACTION;
            
            -- Validate
            IF @Amount <= 0
                THROW 50001, 'Amount must be positive', 1;
            
            IF NOT EXISTS (SELECT 1 FROM Accounts WHERE AccountID = @FromAccount)
                THROW 50002, 'Source account not found', 1;
            
            -- Check balance
            DECLARE @CurrentBalance DECIMAL(18,2);
            SELECT @CurrentBalance = Balance 
            FROM Accounts WHERE AccountID = @FromAccount;
            
            IF @CurrentBalance < @Amount
                THROW 50003, 'Insufficient funds', 1;
            
            -- Transfer
            UPDATE Accounts SET Balance = Balance - @Amount 
            WHERE AccountID = @FromAccount;
            
            UPDATE Accounts SET Balance = Balance + @Amount 
            WHERE AccountID = @ToAccount;
            
            -- Log
            INSERT INTO TransactionLog (FromAccount, ToAccount, Amount, TransDate)
            VALUES (@FromAccount, @ToAccount, @Amount, SYSDATETIME());
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        -- Log the error
        INSERT INTO ErrorLog (ErrorNumber, ErrorMessage, ErrorLine, ErrorProc, ErrorDate)
        VALUES (ERROR_NUMBER(), ERROR_MESSAGE(), ERROR_LINE(), ERROR_PROCEDURE(), SYSDATETIME());
        
        THROW;  -- Re-throw to caller
    END CATCH;
END;
```

> 💡 **Pro Tip:** Always set `SET XACT_ABORT ON` in stored procedures. Without it, some errors DON'T trigger the CATCH block and leave transactions open — a disaster.

---

## 📝 4. T-SQL Variables, Control Flow & Batches

### Variables

```sql
-- Declare and use variables
DECLARE @Name NVARCHAR(100) = 'Ritesh';
DECLARE @Age INT;
DECLARE @Today DATE = GETDATE();

SET @Age = 28;  -- SET assigns one variable at a time

-- SELECT can assign from a query (and assign multiple)
DECLARE @MaxPrice DECIMAL(10,2), @MinPrice DECIMAL(10,2);
SELECT @MaxPrice = MAX(Price), @MinPrice = MIN(Price) FROM Products;

PRINT 'Max: ' + CAST(@MaxPrice AS VARCHAR(20));
PRINT 'Min: ' + CAST(@MinPrice AS VARCHAR(20));
```

### IF...ELSE

```sql
DECLARE @Score INT = 85;

IF @Score >= 90
    PRINT 'Grade: A';
ELSE IF @Score >= 80
    PRINT 'Grade: B';
ELSE IF @Score >= 70
    PRINT 'Grade: C';
ELSE
    PRINT 'Grade: F';

-- Multi-statement blocks need BEGIN...END
IF @Score >= 60
BEGIN
    PRINT 'Passed!';
    UPDATE Students SET Status = 'Passed' WHERE StudentID = 1;
END
ELSE
BEGIN
    PRINT 'Failed!';
    UPDATE Students SET Status = 'Failed' WHERE StudentID = 1;
END;
```

### WHILE Loop

```sql
-- Print numbers 1 to 10
DECLARE @i INT = 1;
WHILE @i <= 10
BEGIN
    PRINT @i;
    SET @i = @i + 1;
END;

-- 💡 Real-world: Delete in batches (avoid lock escalation!)
DECLARE @RowsDeleted INT = 1;
WHILE @RowsDeleted > 0
BEGIN
    DELETE TOP (10000) FROM Logs WHERE LogDate < '2025-01-01';
    SET @RowsDeleted = @@ROWCOUNT;
    
    -- Optional: small delay to let other queries breathe
    WAITFOR DELAY '00:00:01';
END;
```

### IIF and CHOOSE — Inline Conditionals

```sql
-- IIF(condition, true_value, false_value) — like ternary operator
SELECT 
    ProductName,
    Price,
    IIF(Price > 100, 'Premium', 'Budget') AS Category
FROM Products;

-- CHOOSE(index, val1, val2, val3, ...) — 1-based index selection
SELECT 
    OrderID,
    CHOOSE(DATEPART(WEEKDAY, OrderDate),
        'Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'
    ) AS DayName
FROM Orders;
```

### GO — Batch Separator

```sql
-- GO is NOT a T-SQL statement — it's a batch separator recognized by tools

CREATE TABLE Products (ID INT, Name NVARCHAR(100));
GO  -- ← End of batch 1. SQL Server compiles and runs everything above.

INSERT INTO Products VALUES (1, 'Laptop');
GO  -- ← End of batch 2.

-- GO n — Execute the batch n times!
INSERT INTO TestTable VALUES (NEWID(), SYSDATETIME());
GO 1000  -- Runs the INSERT 1000 times! Great for generating test data.
```

---

## 📊 5. OUTPUT Clause — See What You Changed

One of T-SQL's most unique features — capture the **before** and **after** state of modified rows:

```sql
-- 💡 See what was inserted
INSERT INTO Products (ProductName, Price)
OUTPUT INSERTED.ProductID, INSERTED.ProductName, INSERTED.Price
VALUES ('New Widget', 29.99);

-- Output:
-- ProductID │ ProductName │ Price
-- 1042      │ New Widget  │ 29.99

-- 💡 See what was deleted
DELETE FROM Products
OUTPUT DELETED.ProductID, DELETED.ProductName, DELETED.Price
WHERE Price < 5.00;

-- 💡 See before AND after in UPDATE
UPDATE Products
SET Price = Price * 1.10  -- 10% price increase
OUTPUT 
    DELETED.ProductID,
    DELETED.ProductName,
    DELETED.Price AS OldPrice,
    INSERTED.Price AS NewPrice
WHERE CategoryID = 5;

-- Output:
-- ProductID │ ProductName │ OldPrice │ NewPrice
-- 101       │ Widget A    │ 10.00    │ 11.00
-- 102       │ Widget B    │ 20.00    │ 22.00

-- 💡 Capture OUTPUT into a table (for auditing!)
DECLARE @AuditLog TABLE (
    Action NVARCHAR(10), ProductID INT, ProductName NVARCHAR(100), 
    OldPrice DECIMAL(10,2), NewPrice DECIMAL(10,2)
);

UPDATE Products
SET Price = Price * 1.10
OUTPUT 
    'UPDATE', DELETED.ProductID, DELETED.ProductName,
    DELETED.Price, INSERTED.Price
INTO @AuditLog
WHERE CategoryID = 5;

SELECT * FROM @AuditLog;  -- Review all changes!
```

---

## 🔗 6. MERGE — The Upsert Power Tool

`MERGE` combines `INSERT`, `UPDATE`, and `DELETE` in a **single atomic statement**:

```sql
-- Sync a staging table into the main table
MERGE INTO Products AS Target
USING StagingProducts AS Source
    ON Target.ProductID = Source.ProductID

-- If match found → UPDATE
WHEN MATCHED AND Target.Price <> Source.Price THEN
    UPDATE SET 
        Target.Price = Source.Price,
        Target.ModifiedDate = SYSDATETIME()

-- If source has row but target doesn't → INSERT
WHEN NOT MATCHED BY TARGET THEN
    INSERT (ProductID, ProductName, Price, CreatedDate)
    VALUES (Source.ProductID, Source.ProductName, Source.Price, SYSDATETIME())

-- If target has row but source doesn't → DELETE (optional)
WHEN NOT MATCHED BY SOURCE THEN
    DELETE

-- Capture what happened
OUTPUT 
    $action AS MergeAction,  -- 'INSERT', 'UPDATE', or 'DELETE'
    COALESCE(INSERTED.ProductID, DELETED.ProductID) AS ProductID,
    INSERTED.ProductName,
    DELETED.Price AS OldPrice,
    INSERTED.Price AS NewPrice;
-- ⚠️ Don't forget the semicolon! MERGE requires it.
```

### MERGE Visualized

```
╔══════════════════════════════════════════════════════════════╗
║                    MERGE OPERATION                            ║
║                                                              ║
║   Source (Staging)              Target (Production)          ║
║   ┌─────┬──────┬──────┐       ┌─────┬──────┬──────┐        ║
║   │ ID  │ Name │Price │       │ ID  │ Name │Price │        ║
║   ├─────┼──────┼──────┤       ├─────┼──────┼──────┤        ║
║   │  1  │ AAA  │ 15   │       │  1  │ AAA  │ 10   │ → UPDATE║
║   │  2  │ BBB  │ 20   │       │  2  │ BBB  │ 20   │ → SKIP  ║
║   │  3  │ CCC  │ 30   │       │     │      │      │ → INSERT║
║   │     │      │      │       │  4  │ DDD  │ 40   │ → DELETE║
║   └─────┴──────┴──────┘       └─────┴──────┴──────┘        ║
║                                                              ║
║   After MERGE:                                               ║
║   Target                                                     ║
║   ┌─────┬──────┬──────┐                                     ║
║   │  1  │ AAA  │ 15   │  ← Updated                         ║
║   │  2  │ BBB  │ 20   │  ← Unchanged                       ║
║   │  3  │ CCC  │ 30   │  ← Inserted                        ║
║   └─────┴──────┴──────┘  ← ID 4 deleted                    ║
╚══════════════════════════════════════════════════════════════╝
```

> ⚠️ **Caution:** MERGE has known bugs in older SQL Server versions around concurrent operations. Always test thoroughly and consider using separate INSERT/UPDATE/DELETE for high-concurrency scenarios.

---

## 📦 7. Temp Tables, Table Variables & CTEs — The Trio

### Temp Tables (`#temp` and `##temp`)

```sql
-- LOCAL temp table (visible only to current session)
CREATE TABLE #TopProducts (
    ProductID INT,
    ProductName NVARCHAR(100),
    Price DECIMAL(10,2)
);

INSERT INTO #TopProducts
SELECT TOP 100 ProductID, ProductName, Price
FROM Products
ORDER BY Price DESC;

SELECT * FROM #TopProducts;  -- Works in same session
DROP TABLE #TopProducts;     -- Auto-dropped when session ends anyway

-- GLOBAL temp table (visible to ALL sessions — rare use)
CREATE TABLE ##SharedData (ID INT, Value NVARCHAR(100));
-- Exists until the creating session disconnects AND no other session is using it
```

### Table Variables (`@table`)

```sql
DECLARE @Results TABLE (
    ProductID INT,
    ProductName NVARCHAR(100),
    Price DECIMAL(10,2)
);

INSERT INTO @Results
SELECT TOP 10 ProductID, ProductName, Price
FROM Products
ORDER BY Price DESC;

SELECT * FROM @Results;
-- Auto-destroyed when the batch ends (no need to drop)
```

### The Big Comparison

```
╔══════════════════════════════════════════════════════════════════════╗
║           TEMP TABLE vs TABLE VARIABLE vs CTE                        ║
╠══════════════════════╦═══════════════════╦═══════════════════════════╣
║ Feature              ║ #TempTable        ║ @TableVariable            ║
╠══════════════════════╬═══════════════════╬═══════════════════════════╣
║ Created in           ║ tempdb            ║ tempdb (yes, really!)     ║
║ Statistics           ║ ✅ YES            ║ ❌ NO (except 2019+)     ║
║ Indexes              ║ ✅ All types      ║ PK/Unique only (inline)  ║
║ Scope                ║ Session           ║ Batch / SP               ║
║ Transaction logging  ║ ✅ Full           ║ Minimal                  ║
║ Parallelism          ║ ✅ YES            ║ ❌ NO (pre-2019)        ║
║ Row estimates        ║ Accurate          ║ Always 1 row! ⚠️         ║
║ Can ALTER?           ║ ✅ YES            ║ ❌ NO                    ║
║ Good for             ║ Large datasets    ║ Small datasets (<1000)   ║
║                      ║ Complex logic     ║ Simple accumulation      ║
╠══════════════════════╬═══════════════════╬═══════════════════════════╣
║ Feature              ║ CTE (WITH clause) ║                           ║
╠══════════════════════╬═══════════════════╬═══════════════════════════╣
║ Created in           ║ Inline (no store) ║                           ║
║ Persisted?           ║ ❌ NO             ║                           ║
║ Scope                ║ Single statement  ║                           ║
║ Reusable?            ║ ❌ One query only ║                           ║
║ Good for             ║ Readability,      ║                           ║
║                      ║ recursion         ║                           ║
╚══════════════════════╩═══════════════════╩═══════════════════════════╝
```

> 💡 **Rule of Thumb:**
> - **< 100 rows** → Table Variable
> - **100 - 100,000 rows** → Temp Table  
> - **Just for readability** → CTE
> - **Recursive query** → CTE (only option)
> - **Reused multiple times in a batch** → Temp Table

---

## 🔤 8. String Functions — T-SQL Specials

### STRING_AGG — Concatenate Rows into One String (SQL 2017+)

```sql
-- "Give me a comma-separated list of products per category"
SELECT 
    c.CategoryName,
    STRING_AGG(p.ProductName, ', ') AS Products
FROM Products p
JOIN Categories c ON p.CategoryID = c.CategoryID
GROUP BY c.CategoryName;

-- Result:
-- Beverages  │ Chai, Chang, Guaraná, Steeleye Stout
-- Seafood    │ Ikura, Konbu, Carnarvon Tigers

-- With ordering inside the aggregation!
SELECT 
    c.CategoryName,
    STRING_AGG(p.ProductName, ', ') WITHIN GROUP (ORDER BY p.ProductName) AS Products
FROM Products p
JOIN Categories c ON p.CategoryID = c.CategoryID
GROUP BY c.CategoryName;
```

### STRING_SPLIT — Split a String into Rows (SQL 2016+)

```sql
-- Split a comma-separated list into rows
SELECT value FROM STRING_SPLIT('apple,banana,cherry,date', ',');

-- Result:
-- value
-- -----
-- apple
-- banana
-- cherry
-- date

-- 💡 Real-world: Pass comma-separated IDs from app to SQL
CREATE PROCEDURE dbo.GetProductsByIDs
    @IDs NVARCHAR(MAX)  -- '1,5,10,42'
AS
BEGIN
    SELECT * FROM Products
    WHERE ProductID IN (
        SELECT CAST(value AS INT) FROM STRING_SPLIT(@IDs, ',')
    );
END;

-- SQL Server 2022+: STRING_SPLIT now returns ordinal!
SELECT value, ordinal 
FROM STRING_SPLIT('apple,banana,cherry', ',', 1);  -- 1 enables ordinal
-- value   │ ordinal
-- apple   │ 1
-- banana  │ 2
-- cherry  │ 3
```

### FORMAT — .NET-Style Formatting (SQL 2012+)

```sql
-- Format numbers
SELECT FORMAT(1234567.89, 'N2');        -- 1,234,567.89
SELECT FORMAT(1234567.89, 'C', 'en-US');-- $1,234,567.89
SELECT FORMAT(1234567.89, 'C', 'en-IN');-- ₹12,34,567.89

-- Format dates
SELECT FORMAT(GETDATE(), 'dd-MMM-yyyy');        -- 02-Jun-2026
SELECT FORMAT(GETDATE(), 'yyyy-MM-dd HH:mm:ss');-- 2026-06-02 14:30:00
SELECT FORMAT(GETDATE(), 'dddd, MMMM dd, yyyy');-- Tuesday, June 02, 2026

-- ⚠️ Performance warning: FORMAT uses .NET CLR internally — SLOW!
-- For high-volume queries, use CONVERT with style codes instead:
SELECT CONVERT(VARCHAR(10), GETDATE(), 105);  -- 02-06-2026 (faster!)
```

### Other Key String Functions

```sql
-- CONCAT — NULL-safe concatenation (never returns NULL)
SELECT CONCAT('Hello', ' ', 'World', NULL, '!');  -- Hello World!
-- vs + operator: 'Hello' + NULL = NULL ← dangerous!

-- CONCAT_WS — Concat With Separator (SQL 2017+)
SELECT CONCAT_WS(', ', 'Mumbai', 'Maharashtra', 'India');
-- Mumbai, Maharashtra, India
-- Skips NULLs automatically!

-- TRIM / LTRIM / RTRIM
SELECT TRIM('   Hello   ');        -- 'Hello'  (SQL 2017+)
SELECT TRIM('x' FROM 'xxxHelloxxx'); -- 'Hello' (custom char!)

-- TRANSLATE — Replace multiple characters at once (SQL 2017+)
SELECT TRANSLATE('2*[3+4]/{5-6}', '[]{}', '()()');
-- Result: 2*(3+4)/(5-6)

-- LEFT / RIGHT / SUBSTRING
SELECT LEFT('Hello World', 5);        -- Hello
SELECT RIGHT('Hello World', 5);       -- World
SELECT SUBSTRING('Hello World', 7, 5);-- World

-- CHARINDEX — Find position of substring
SELECT CHARINDEX('World', 'Hello World');  -- 7

-- REPLACE / STUFF
SELECT REPLACE('Hello World', 'World', 'T-SQL');  -- Hello T-SQL
SELECT STUFF('Hello World', 7, 5, 'T-SQL');       -- Hello T-SQL
-- STUFF(string, start, length, replacement)

-- REPLICATE — Repeat a string
SELECT REPLICATE('Ha', 3);  -- HaHaHa

-- REVERSE
SELECT REVERSE('Hello');  -- olleH
```

---

## 📅 9. Date & Time Functions — T-SQL Specials

```sql
-- Current date/time functions
SELECT GETDATE();            -- 2026-06-02 14:30:45.123  (datetime)
SELECT SYSDATETIME();        -- 2026-06-02 14:30:45.1234567  (datetime2 — more precise!)
SELECT GETUTCDATE();         -- UTC time
SELECT SYSUTCDATETIME();     -- UTC with high precision

-- DATEADD — Add interval to a date
SELECT DATEADD(DAY, 7, GETDATE());      -- 7 days from now
SELECT DATEADD(MONTH, -3, GETDATE());   -- 3 months ago
SELECT DATEADD(YEAR, 1, '2026-01-01');  -- 2027-01-01

-- DATEDIFF — Difference between two dates
SELECT DATEDIFF(DAY, '2026-01-01', '2026-06-02');    -- 152 days
SELECT DATEDIFF(MONTH, '2025-01-01', '2026-06-02');  -- 17 months
SELECT DATEDIFF(YEAR, '1998-05-15', GETDATE());      -- Age in years

-- DATEDIFF_BIG — For large differences (SQL 2016+)
SELECT DATEDIFF_BIG(MILLISECOND, '2000-01-01', GETDATE());  -- Huge number!

-- DATEPART — Extract parts of a date
SELECT DATEPART(YEAR, GETDATE());       -- 2026
SELECT DATEPART(MONTH, GETDATE());      -- 6
SELECT DATEPART(WEEKDAY, GETDATE());    -- 3 (Tuesday)
SELECT DATEPART(QUARTER, GETDATE());    -- 2

-- DATENAME — Get part name as string
SELECT DATENAME(MONTH, GETDATE());      -- June
SELECT DATENAME(WEEKDAY, GETDATE());    -- Tuesday

-- YEAR(), MONTH(), DAY() — shortcuts
SELECT YEAR(GETDATE()), MONTH(GETDATE()), DAY(GETDATE());
-- 2026, 6, 2

-- EOMONTH — End of month (SQL 2012+)
SELECT EOMONTH(GETDATE());             -- 2026-06-30
SELECT EOMONTH(GETDATE(), 2);          -- 2 months later end of month

-- DATEFROMPARTS — Build a date from parts (SQL 2012+)
SELECT DATEFROMPARTS(2026, 6, 15);     -- 2026-06-15
SELECT DATETIME2FROMPARTS(2026, 6, 15, 14, 30, 0, 0, 7);

-- DATE_BUCKET — Round dates to intervals (SQL 2022+)
SELECT DATE_BUCKET(WEEK, 1, GETDATE());  -- Start of current week
```

---

## 🔧 10. Other T-SQL Power Features

### COALESCE & ISNULL — NULL Handling

```sql
-- ISNULL(value, replacement) — T-SQL specific, fast
SELECT ISNULL(MiddleName, 'N/A') FROM Employees;

-- COALESCE(v1, v2, v3, ...) — Standard SQL, more flexible
SELECT COALESCE(Phone, Mobile, Email, 'No Contact') FROM Customers;

-- Key difference:
-- ISNULL: Takes 2 args, returns data type of FIRST arg
-- COALESCE: Takes N args, returns highest precedence data type
```

### SEQUENCE Objects (SQL 2012+)

```sql
-- Create a sequence (independent of any table)
CREATE SEQUENCE dbo.OrderNumberSeq
    AS INT
    START WITH 10000
    INCREMENT BY 1
    MINVALUE 10000
    MAXVALUE 99999
    CYCLE;  -- Restart from MINVALUE when MAXVALUE reached

-- Use it
SELECT NEXT VALUE FOR dbo.OrderNumberSeq;  -- 10000
SELECT NEXT VALUE FOR dbo.OrderNumberSeq;  -- 10001

-- Use in INSERT
INSERT INTO Orders (OrderNumber, CustomerID)
VALUES (NEXT VALUE FOR dbo.OrderNumberSeq, 42);
```

### JSON Support (SQL 2016+)

```sql
-- Parse JSON
DECLARE @json NVARCHAR(MAX) = N'{
    "name": "Ritesh",
    "age": 28,
    "skills": ["SQL", "Python", "C#"],
    "address": {
        "city": "Mumbai",
        "country": "India"
    }
}';

-- Extract values
SELECT JSON_VALUE(@json, '$.name');           -- Ritesh
SELECT JSON_VALUE(@json, '$.age');            -- 28
SELECT JSON_VALUE(@json, '$.address.city');   -- Mumbai
SELECT JSON_QUERY(@json, '$.skills');         -- ["SQL","Python","C#"]

-- Generate JSON from table
SELECT ProductID, ProductName, Price
FROM Products
FOR JSON PATH;
-- [{"ProductID":1,"ProductName":"Chai","Price":18.00}, ...]

-- FOR JSON AUTO — auto-generates nesting from JOINs
SELECT c.CategoryName, p.ProductName, p.Price
FROM Categories c
JOIN Products p ON c.CategoryID = p.CategoryID
FOR JSON AUTO;

-- OPENJSON — Parse JSON array into rows
SELECT * FROM OPENJSON(@json, '$.skills');
-- key │ value  │ type
-- 0   │ SQL    │ 1
-- 1   │ Python │ 1
-- 2   │ C#     │ 1

-- JSON_MODIFY (SQL 2016+)
SET @json = JSON_MODIFY(@json, '$.age', 29);
SET @json = JSON_MODIFY(@json, '$.address.city', 'Delhi');
```

### PIVOT & UNPIVOT

```sql
-- PIVOT: Rows to Columns
-- Monthly sales by product → columns for each month
SELECT ProductName, [Jan], [Feb], [Mar]
FROM (
    SELECT ProductName, MonthName, SalesAmount
    FROM MonthlySales
) AS SourceTable
PIVOT (
    SUM(SalesAmount)
    FOR MonthName IN ([Jan], [Feb], [Mar])
) AS PivotTable;

-- Result:
-- ProductName │ Jan    │ Feb    │ Mar
-- Laptop      │ 50000  │ 62000  │ 45000
-- Phone       │ 30000  │ 28000  │ 35000

-- UNPIVOT: Columns to Rows (reverse)
SELECT ProductName, MonthName, SalesAmount
FROM PivotedSales
UNPIVOT (
    SalesAmount FOR MonthName IN ([Jan], [Feb], [Mar])
) AS UnpivotTable;
```

### Dynamic SQL with sp_executesql

```sql
-- ⚠️ NEVER build SQL by string concatenation (SQL injection risk!)
-- ❌ DANGEROUS:
-- SET @sql = 'SELECT * FROM Products WHERE Name = ''' + @userInput + '''';

-- ✅ SAFE: Use sp_executesql with parameters
DECLARE @sql NVARCHAR(MAX);
DECLARE @params NVARCHAR(MAX);

SET @sql = N'SELECT * FROM Products WHERE CategoryID = @catID AND Price > @minPrice';
SET @params = N'@catID INT, @minPrice DECIMAL(10,2)';

EXEC sp_executesql @sql, @params, @catID = 5, @minPrice = 10.00;
-- Parameterized → safe from injection, plan cache friendly!
```

---

## 🧠 Quick Recall — Chapter Summary

| T-SQL Feature | One-Line Summary |
|---------------|-----------------|
| `TOP` | Limit rows returned (SQL Server alternative to LIMIT) |
| `OFFSET-FETCH` | Modern pagination with skip/take semantics |
| `CROSS APPLY` | For each left row, evaluate right expression (like correlated subquery returning set) |
| `OUTER APPLY` | Same as CROSS APPLY but keeps left rows even with no match (like LEFT JOIN) |
| `TRY...CATCH` | Structured error handling with error detail functions |
| `THROW` | Raise or re-throw errors (modern replacement for RAISERROR) |
| `OUTPUT` | Capture inserted/deleted/updated rows in DML operations |
| `MERGE` | Atomic INSERT + UPDATE + DELETE based on match conditions |
| `IIF` / `CHOOSE` | Inline conditional expressions |
| `STRING_AGG` | Concatenate values from multiple rows with a separator |
| `STRING_SPLIT` | Split a delimited string into rows |
| `FORMAT` | .NET-style formatting for dates and numbers (slow!) |
| `JSON_VALUE` / `OPENJSON` | Parse and generate JSON data |
| `PIVOT` / `UNPIVOT` | Transform rows↔columns |
| `sp_executesql` | Safe parameterized dynamic SQL execution |
| `SET XACT_ABORT ON` | Auto-rollback on any error — always use in stored procedures |
| `GO` | Batch separator (not a T-SQL statement) |
| Temp Tables `#` | Session-scoped, has statistics, good for large datasets |
| Table Variables `@` | Batch-scoped, no stats (pre-2019), good for small datasets |

---

## ❓ Self-Check Questions

1. What's the difference between `TOP` and `OFFSET-FETCH`?
2. Explain `CROSS APPLY` vs `INNER JOIN` — when is APPLY better?
3. What does `OUTER APPLY` do differently from `CROSS APPLY`?
4. Write a `TRY...CATCH` block that transfers money between two accounts.
5. What does the `OUTPUT` clause do? Show an example with `UPDATE`.
6. When should you use `MERGE` and what are its risks?
7. Temp Table vs Table Variable — when to use which?
8. How does `STRING_AGG` work? Write a query to list all employees per department as comma-separated.
9. Why is `FORMAT()` slow? What should you use instead in high-volume queries?
10. How do you prevent SQL injection when using dynamic SQL in T-SQL?

---

## 🎯 Hands-On Challenge

```sql
-- 1. Write a query that paginates products (page 5, 25 items per page)

-- 2. Use CROSS APPLY to get the last 3 orders per customer

-- 3. Write a MERGE statement that syncs two tables

-- 4. Use STRING_AGG to list all products per category, ordered alphabetically

-- 5. Write a TRY...CATCH block that:
--    a. Starts a transaction
--    b. Inserts a row
--    c. Intentionally causes an error (divide by zero)
--    d. Catches the error, rolls back, and prints error details

-- 6. Use the OUTPUT clause to capture all rows deleted from a cleanup operation

-- 7. Parse this JSON and extract all skills:
--    '{"name":"Alex","skills":["SQL","Python","Java"]}'

-- 8. Create a PIVOT query that shows monthly sales by product
```

---

> **Next Chapter** → [2C.4 — SQL Server Performance Tuning](./04-SQLServer-Performance.md)

---

> *"Standard SQL is a bicycle. T-SQL is a motorcycle — same road, but you get there faster, louder, and with more style."*
