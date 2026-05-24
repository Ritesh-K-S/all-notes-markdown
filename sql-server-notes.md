# SQL Server — Complete Notes

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Databases](#2-databases)
3. [Data Types](#3-data-types)
4. [Tables](#4-tables)
5. [Constraints](#5-constraints)
6. [Insert, Update, Delete](#6-insert-update-delete)
7. [Select & Querying](#7-select--querying)
8. [Joins](#8-joins)
9. [Aggregate Functions & Group By](#9-aggregate-functions--group-by)
10. [Subqueries](#10-subqueries)
11. [Common Table Expressions (CTE)](#11-common-table-expressions-cte)
12. [Window Functions](#12-window-functions)
13. [Views](#13-views)
14. [Indexes](#14-indexes)
15. [Stored Procedures](#15-stored-procedures)
16. [Functions](#16-functions)
17. [Triggers](#17-triggers)
18. [Transactions](#18-transactions)
19. [Error Handling](#19-error-handling)
20. [String Functions](#20-string-functions)
21. [Date & Time Functions](#21-date--time-functions)
22. [Mathematical Functions](#22-mathematical-functions)
23. [Conversion Functions](#23-conversion-functions)
24. [Case Expression](#24-case-expression)
25. [Set Operators](#25-set-operators)
26. [Pivot & Unpivot](#26-pivot--unpivot)
27. [Dynamic SQL](#27-dynamic-sql)
28. [Cursors](#28-cursors)
29. [Variables & Control Flow](#29-variables--control-flow)
30. [JSON Support (SQL Server 2016+)](#30-json-support-sql-server-2016)
31. [XML Support](#31-xml-support)
32. [Schemas](#32-schemas)
33. [User Management & Security](#33-user-management--security)
34. [Backup & Restore](#34-backup--restore)
35. [System Views & DMVs](#35-system-views--dmvs)
36. [Execution Plans & Performance](#36-execution-plans--performance)
37. [Temporal Tables (SQL Server 2016+)](#37-temporal-tables-sql-server-2016)
38. [Sequences](#38-sequences)
39. [Apply Operator (Cross Apply / Outer Apply)](#39-apply-operator-cross-apply--outer-apply)
40. [Common Table Patterns](#40-common-table-patterns)
41. [SQL Server Agent & Jobs](#41-sql-server-agent--jobs)
42. [Useful System Stored Procedures](#42-useful-system-stored-procedures)
43. [DBCC Commands](#43-dbcc-commands)
44. [Common T-SQL Patterns & Tips](#44-common-t-sql-patterns--tips)
45. [Quick Reference Cheat Sheet](#45-quick-reference-cheat-sheet)

---

## 1. INTRODUCTION

### What is SQL Server?
- **Relational Database Management System (RDBMS)** developed by **Microsoft**.
- First released in 1989 (partnership with Sybase), fully Microsoft-owned since SQL Server 2000.
- Uses **T-SQL (Transact-SQL)** — Microsoft's extension of standard SQL.
- Used for enterprise applications, data warehousing, BI, reporting, cloud databases.

### Editions
| Edition | Use Case |
|---------|----------|
| **Enterprise** | Full features, unlimited compute, mission-critical workloads |
| **Standard** | Mid-tier apps, limited memory/cores |
| **Express** | Free, 10 GB database limit, lightweight apps |
| **Developer** | Free, full Enterprise features — for dev/test only |
| **Azure SQL** | Cloud-hosted (PaaS) — managed by Microsoft |

### Key Components
| Component | Description |
|-----------|-------------|
| **Database Engine** | Core service — stores, processes, and secures data |
| **SSMS** | SQL Server Management Studio — GUI tool |
| **Azure Data Studio** | Cross-platform lightweight editor |
| **SQL Server Agent** | Job scheduling and automation |
| **SSIS** | SQL Server Integration Services — ETL tool |
| **SSRS** | SQL Server Reporting Services — reports |
| **SSAS** | SQL Server Analysis Services — OLAP / data mining |
| **sqlcmd** | Command-line tool for SQL Server |

### Connecting to SQL Server
```sql
-- sqlcmd from command line
sqlcmd -S localhost -U sa -P YourPassword123

-- or with Windows Authentication
sqlcmd -S localhost -E

-- Azure Data Studio / SSMS — connect via GUI using:
-- Server: localhost (or hostname\instance)
-- Authentication: Windows Authentication or SQL Server Authentication
```

---

## 2. DATABASES

### Creating a Database
```sql
CREATE DATABASE MyAppDB;

-- With options
CREATE DATABASE MyAppDB
ON PRIMARY (
    NAME = 'MyAppDB_Data',
    FILENAME = 'C:\SQLData\MyAppDB.mdf',
    SIZE = 100MB,
    MAXSIZE = 10GB,
    FILEGROWTH = 50MB
)
LOG ON (
    NAME = 'MyAppDB_Log',
    FILENAME = 'C:\SQLData\MyAppDB.ldf',
    SIZE = 50MB,
    MAXSIZE = 5GB,
    FILEGROWTH = 25MB
);
```

### Switching / Using a Database
```sql
USE MyAppDB;
```

### Modifying a Database
```sql
-- Rename
ALTER DATABASE MyAppDB MODIFY NAME = NewAppDB;

-- Add a filegroup
ALTER DATABASE MyAppDB ADD FILEGROUP SecondaryFG;

-- Change recovery model
ALTER DATABASE MyAppDB SET RECOVERY FULL;      -- FULL | SIMPLE | BULK_LOGGED

-- Set single-user mode (for maintenance)
ALTER DATABASE MyAppDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;

-- Back to multi-user
ALTER DATABASE MyAppDB SET MULTI_USER;
```

### Dropping a Database
```sql
-- Check before dropping
IF DB_ID('MyAppDB') IS NOT NULL
    DROP DATABASE MyAppDB;

-- SQL Server 2016+
DROP DATABASE IF EXISTS MyAppDB;
```

### Viewing Databases
```sql
SELECT name, database_id, create_date, state_desc
FROM sys.databases;

-- System databases: master, msdb, model, tempdb
-- master  → server-level config, logins, linked servers
-- msdb    → SQL Agent jobs, alerts, backups
-- model   → template for new databases
-- tempdb  → temporary tables, intermediate results (recreated on restart)
```

---

## 3. DATA TYPES

### Numeric Types
| Type | Size | Range / Description |
|------|------|---------------------|
| `BIT` | 1 bit | 0, 1, or NULL (boolean) |
| `TINYINT` | 1 byte | 0 to 255 |
| `SMALLINT` | 2 bytes | -32,768 to 32,767 |
| `INT` | 4 bytes | -2³¹ to 2³¹-1 |
| `BIGINT` | 8 bytes | -2⁶³ to 2⁶³-1 |
| `DECIMAL(p,s)` | 5-17 bytes | Exact numeric — p = precision, s = scale |
| `NUMERIC(p,s)` | 5-17 bytes | Same as DECIMAL |
| `FLOAT(n)` | 4 or 8 bytes | Approximate — n= 1-24 (4B), 25-53 (8B) |
| `REAL` | 4 bytes | Same as FLOAT(24) |
| `MONEY` | 8 bytes | -922 trillion to +922 trillion (4 decimals) |
| `SMALLMONEY` | 4 bytes | -214,748.3648 to +214,748.3647 |

### String Types
| Type | Description |
|------|-------------|
| `CHAR(n)` | Fixed-length non-Unicode — n up to 8,000 |
| `VARCHAR(n)` | Variable-length non-Unicode — n up to 8,000 |
| `VARCHAR(MAX)` | Variable-length non-Unicode — up to 2 GB |
| `NCHAR(n)` | Fixed-length Unicode — n up to 4,000 |
| `NVARCHAR(n)` | Variable-length Unicode — n up to 4,000 |
| `NVARCHAR(MAX)` | Variable-length Unicode — up to 2 GB |
| `TEXT` | Deprecated — use VARCHAR(MAX) |
| `NTEXT` | Deprecated — use NVARCHAR(MAX) |

### Date & Time Types
| Type | Format | Range |
|------|--------|-------|
| `DATE` | YYYY-MM-DD | 0001-01-01 to 9999-12-31 |
| `TIME` | HH:MM:SS.nnnnnnn | 00:00:00.0000000 to 23:59:59.9999999 |
| `DATETIME` | YYYY-MM-DD HH:MM:SS | 1753-01-01 to 9999-12-31 (3.33ms accuracy) |
| `DATETIME2` | YYYY-MM-DD HH:MM:SS.nnnnnnn | 0001-01-01 to 9999-12-31 (100ns accuracy) |
| `SMALLDATETIME` | YYYY-MM-DD HH:MM | 1900-01-01 to 2079-06-06 (1 min accuracy) |
| `DATETIMEOFFSET` | YYYY-MM-DD HH:MM:SS +HH:MM | With timezone offset |

### Binary Types
| Type | Description |
|------|-------------|
| `BINARY(n)` | Fixed-length — n up to 8,000 bytes |
| `VARBINARY(n)` | Variable-length — n up to 8,000 bytes |
| `VARBINARY(MAX)` | Up to 2 GB (replaces IMAGE) |
| `IMAGE` | Deprecated — use VARBINARY(MAX) |

### Other Types
| Type | Description |
|------|-------------|
| `UNIQUEIDENTIFIER` | 16-byte GUID — `NEWID()` generates one |
| `XML` | Stores XML documents |
| `JSON` | No native type — stored as NVARCHAR, queried via JSON functions |
| `SQL_VARIANT` | Stores values of various data types |
| `TABLE` | Special type for table variables |
| `CURSOR` | Reference to a cursor |
| `HIERARCHYID` | Represents position in a tree hierarchy |
| `GEOGRAPHY` | Spatial data — Earth coordinates |
| `GEOMETRY` | Spatial data — plane coordinates |

---

## 4. TABLES

### Creating Tables
```sql
CREATE TABLE Employees (
    EmployeeId   INT           IDENTITY(1,1) PRIMARY KEY,
    FirstName    NVARCHAR(50)  NOT NULL,
    LastName     NVARCHAR(50)  NOT NULL,
    Email        NVARCHAR(100) UNIQUE,
    Salary       DECIMAL(10,2) DEFAULT 0.00,
    DepartmentId INT           NULL,
    HireDate     DATE          DEFAULT GETDATE(),
    IsActive     BIT           DEFAULT 1,
    CreatedAt    DATETIME2     DEFAULT SYSDATETIME()
);
```

### IDENTITY Column
```sql
-- IDENTITY(seed, increment) — auto-incrementing
EmployeeId INT IDENTITY(1,1)     -- starts at 1, increments by 1

-- Get last inserted identity
SELECT SCOPE_IDENTITY();          -- current scope (recommended)
SELECT @@IDENTITY;                -- any scope (avoid — triggers can affect)
SELECT IDENT_CURRENT('Employees'); -- any session, specific table

-- Reset identity
DBCC CHECKIDENT('Employees', RESEED, 0);

-- Insert explicit identity value
SET IDENTITY_INSERT Employees ON;
INSERT INTO Employees (EmployeeId, FirstName, LastName)
VALUES (100, 'John', 'Doe');
SET IDENTITY_INSERT Employees OFF;
```

### Altering Tables
```sql
-- Add column
ALTER TABLE Employees ADD PhoneNumber NVARCHAR(15);

-- Drop column
ALTER TABLE Employees DROP COLUMN PhoneNumber;

-- Alter column type
ALTER TABLE Employees ALTER COLUMN Email NVARCHAR(200);

-- Add constraint
ALTER TABLE Employees ADD CONSTRAINT CK_Salary CHECK (Salary >= 0);

-- Drop constraint
ALTER TABLE Employees DROP CONSTRAINT CK_Salary;

-- Rename column
EXEC sp_rename 'Employees.Email', 'EmailAddress', 'COLUMN';

-- Rename table
EXEC sp_rename 'Employees', 'Staff';
```

### Dropping Tables
```sql
DROP TABLE IF EXISTS Employees;
```

### Temporary Tables
```sql
-- Local temp table (visible in current session only)
CREATE TABLE #TempEmployees (
    Id   INT,
    Name NVARCHAR(50)
);
-- Auto-dropped when session ends

-- Global temp table (visible to all sessions)
CREATE TABLE ##GlobalTemp (
    Id   INT,
    Name NVARCHAR(50)
);
-- Dropped when creating session ends and no other sessions reference it

-- Table variable (scoped to batch/procedure)
DECLARE @TempTable TABLE (
    Id   INT,
    Name NVARCHAR(50)
);
```

### Viewing Table Info
```sql
-- List all tables
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE';

-- Table structure
EXEC sp_help 'Employees';
EXEC sp_columns 'Employees';

-- Detailed column info
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE, CHARACTER_MAXIMUM_LENGTH
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'Employees';
```

---

## 5. CONSTRAINTS

### Types of Constraints
| Constraint | Description |
|------------|-------------|
| `PRIMARY KEY` | Uniquely identifies each row — NOT NULL + UNIQUE |
| `FOREIGN KEY` | References primary key of another table |
| `UNIQUE` | Ensures all values are distinct (allows one NULL) |
| `NOT NULL` | Column cannot contain NULL |
| `CHECK` | Validates data against a condition |
| `DEFAULT` | Sets a default value if none provided |

### Primary Key
```sql
-- Inline
CREATE TABLE Products (
    ProductId INT PRIMARY KEY,
    Name NVARCHAR(100)
);

-- Named constraint
CREATE TABLE Products (
    ProductId INT,
    Name NVARCHAR(100),
    CONSTRAINT PK_Products PRIMARY KEY (ProductId)
);

-- Composite primary key
CREATE TABLE OrderItems (
    OrderId   INT,
    ProductId INT,
    Quantity  INT,
    CONSTRAINT PK_OrderItems PRIMARY KEY (OrderId, ProductId)
);
```

### Foreign Key
```sql
CREATE TABLE Orders (
    OrderId    INT IDENTITY(1,1) PRIMARY KEY,
    CustomerId INT NOT NULL,
    OrderDate  DATE DEFAULT GETDATE(),
    CONSTRAINT FK_Orders_Customers
        FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId)
        ON DELETE CASCADE        -- delete orders when customer deleted
        ON UPDATE CASCADE        -- update if customer ID changes
);

-- Referential actions:
-- ON DELETE CASCADE       — delete child rows
-- ON DELETE SET NULL      — set FK to NULL
-- ON DELETE SET DEFAULT   — set FK to default value
-- ON DELETE NO ACTION     — prevent delete (default)
-- Same options for ON UPDATE
```

### CHECK Constraint
```sql
ALTER TABLE Employees
ADD CONSTRAINT CK_Employees_Salary CHECK (Salary >= 0 AND Salary <= 1000000);

ALTER TABLE Employees
ADD CONSTRAINT CK_Employees_Email CHECK (Email LIKE '%@%.%');
```

### UNIQUE Constraint
```sql
ALTER TABLE Employees
ADD CONSTRAINT UQ_Employees_Email UNIQUE (Email);

-- Composite unique
ALTER TABLE Employees
ADD CONSTRAINT UQ_Employees_Name UNIQUE (FirstName, LastName);
```

### DEFAULT Constraint
```sql
ALTER TABLE Employees
ADD CONSTRAINT DF_Employees_IsActive DEFAULT 1 FOR IsActive;
```

---

## 6. INSERT, UPDATE, DELETE

### INSERT
```sql
-- Single row
INSERT INTO Employees (FirstName, LastName, Email, Salary)
VALUES ('Ritesh', 'Singh', 'ritesh@example.com', 75000.00);

-- Multiple rows
INSERT INTO Employees (FirstName, LastName, Email, Salary)
VALUES
    ('Alice', 'Johnson', 'alice@example.com', 80000.00),
    ('Bob', 'Smith', 'bob@example.com', 72000.00),
    ('Carol', 'Williams', 'carol@example.com', 90000.00);

-- Insert from SELECT
INSERT INTO ArchivedEmployees (FirstName, LastName, Email)
SELECT FirstName, LastName, Email
FROM Employees
WHERE IsActive = 0;

-- Insert with OUTPUT (return inserted data)
INSERT INTO Employees (FirstName, LastName, Email)
OUTPUT INSERTED.EmployeeId, INSERTED.FirstName
VALUES ('Dave', 'Brown', 'dave@example.com');

-- SELECT INTO — create new table from query
SELECT FirstName, LastName, Salary
INTO HighEarners
FROM Employees
WHERE Salary > 100000;
```

### UPDATE
```sql
-- Single column
UPDATE Employees
SET Salary = 85000.00
WHERE EmployeeId = 1;

-- Multiple columns
UPDATE Employees
SET Salary = Salary * 1.10,
    IsActive = 1
WHERE DepartmentId = 5;

-- Update with JOIN
UPDATE e
SET e.Salary = e.Salary * 1.05
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.DepartmentId
WHERE d.Name = 'Engineering';

-- Update with OUTPUT
UPDATE Employees
SET Salary = Salary * 1.10
OUTPUT DELETED.Salary AS OldSalary, INSERTED.Salary AS NewSalary
WHERE DepartmentId = 3;

-- ⚠ Always use WHERE — without it, ALL rows are updated
```

### DELETE
```sql
-- Delete specific rows
DELETE FROM Employees
WHERE EmployeeId = 10;

-- Delete with JOIN
DELETE e
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.DepartmentId
WHERE d.Name = 'Defunct';

-- Delete with OUTPUT
DELETE FROM Employees
OUTPUT DELETED.EmployeeId, DELETED.FirstName
WHERE IsActive = 0;

-- ⚠ Always use WHERE — without it, ALL rows are deleted
```

### TRUNCATE vs DELETE
```sql
TRUNCATE TABLE Employees;
-- Removes ALL rows (no WHERE clause)
-- Faster than DELETE — minimal logging
-- Resets IDENTITY counter
-- Cannot be used with FK references
-- Cannot use OUTPUT

DELETE FROM Employees;
-- Removes ALL rows (or use WHERE for specific)
-- Fully logged — slower
-- Does NOT reset IDENTITY
-- Fires triggers
-- Can use OUTPUT
```

### MERGE (Upsert)
```sql
MERGE INTO Employees AS target
USING StagingEmployees AS source
ON target.Email = source.Email
WHEN MATCHED THEN
    UPDATE SET
        target.FirstName = source.FirstName,
        target.Salary = source.Salary
WHEN NOT MATCHED BY TARGET THEN
    INSERT (FirstName, LastName, Email, Salary)
    VALUES (source.FirstName, source.LastName, source.Email, source.Salary)
WHEN NOT MATCHED BY SOURCE THEN
    DELETE;
-- Must end with semicolon
```

---

## 7. SELECT & QUERYING

### Basic SELECT
```sql
-- All columns
SELECT * FROM Employees;

-- Specific columns
SELECT FirstName, LastName, Salary FROM Employees;

-- Aliases
SELECT
    FirstName AS [First Name],
    LastName AS [Last Name],
    Salary * 12 AS AnnualSalary
FROM Employees;

-- Distinct values
SELECT DISTINCT DepartmentId FROM Employees;

-- Top N rows
SELECT TOP 10 * FROM Employees ORDER BY Salary DESC;
SELECT TOP 10 PERCENT * FROM Employees ORDER BY Salary DESC;
SELECT TOP 5 WITH TIES * FROM Employees ORDER BY Salary DESC;  -- includes ties
```

### WHERE Clause
```sql
-- Comparison operators: =, <>, !=, <, >, <=, >=
SELECT * FROM Employees WHERE Salary > 50000;
SELECT * FROM Employees WHERE DepartmentId <> 5;

-- Logical operators: AND, OR, NOT
SELECT * FROM Employees WHERE Salary > 50000 AND IsActive = 1;
SELECT * FROM Employees WHERE DepartmentId = 1 OR DepartmentId = 2;
SELECT * FROM Employees WHERE NOT IsActive = 0;

-- BETWEEN (inclusive)
SELECT * FROM Employees WHERE Salary BETWEEN 50000 AND 100000;

-- IN
SELECT * FROM Employees WHERE DepartmentId IN (1, 2, 3);
SELECT * FROM Employees WHERE DepartmentId NOT IN (4, 5);

-- LIKE (pattern matching)
SELECT * FROM Employees WHERE FirstName LIKE 'R%';       -- starts with R
SELECT * FROM Employees WHERE LastName LIKE '%son';       -- ends with son
SELECT * FROM Employees WHERE Email LIKE '%@gmail%';      -- contains @gmail
SELECT * FROM Employees WHERE FirstName LIKE '_o%';       -- second char is 'o'
SELECT * FROM Employees WHERE FirstName LIKE '[A-C]%';    -- starts with A, B, or C
SELECT * FROM Employees WHERE FirstName LIKE '[^A-C]%';   -- NOT starting with A-C

-- NULL checks
SELECT * FROM Employees WHERE DepartmentId IS NULL;
SELECT * FROM Employees WHERE DepartmentId IS NOT NULL;

-- EXISTS
SELECT * FROM Departments d
WHERE EXISTS (SELECT 1 FROM Employees e WHERE e.DepartmentId = d.DepartmentId);
```

### ORDER BY
```sql
SELECT * FROM Employees ORDER BY LastName ASC;                  -- ascending (default)
SELECT * FROM Employees ORDER BY Salary DESC;                   -- descending
SELECT * FROM Employees ORDER BY DepartmentId ASC, Salary DESC; -- multi-column
SELECT * FROM Employees ORDER BY 3 DESC;                        -- by column position (avoid)
```

### OFFSET-FETCH (Pagination — SQL Server 2012+)
```sql
SELECT * FROM Employees
ORDER BY EmployeeId
OFFSET 20 ROWS                 -- skip first 20
FETCH NEXT 10 ROWS ONLY;       -- return next 10

-- Page 3, page size 10:
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

---

## 8. JOINS

### Types of Joins
```
INNER JOIN       — matching rows from both tables
LEFT JOIN        — all from left + matching from right (NULL if no match)
RIGHT JOIN       — all from right + matching from left (NULL if no match)
FULL OUTER JOIN  — all rows from both tables (NULL where no match)
CROSS JOIN       — cartesian product (every row × every row)
SELF JOIN        — table joined with itself
```

### Syntax & Examples
```sql
-- INNER JOIN
SELECT e.FirstName, e.LastName, d.Name AS Department
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.DepartmentId;

-- LEFT JOIN (LEFT OUTER JOIN)
SELECT e.FirstName, d.Name AS Department
FROM Employees e
LEFT JOIN Departments d ON e.DepartmentId = d.DepartmentId;
-- Employees without a department → Department = NULL

-- RIGHT JOIN (RIGHT OUTER JOIN)
SELECT e.FirstName, d.Name AS Department
FROM Employees e
RIGHT JOIN Departments d ON e.DepartmentId = d.DepartmentId;
-- Departments with no employees → FirstName = NULL

-- FULL OUTER JOIN
SELECT e.FirstName, d.Name AS Department
FROM Employees e
FULL OUTER JOIN Departments d ON e.DepartmentId = d.DepartmentId;

-- CROSS JOIN (cartesian product)
SELECT e.FirstName, p.ProjectName
FROM Employees e
CROSS JOIN Projects p;
-- 10 employees × 5 projects = 50 rows

-- SELF JOIN (e.g., manager-employee relationship)
SELECT
    emp.FirstName AS Employee,
    mgr.FirstName AS Manager
FROM Employees emp
LEFT JOIN Employees mgr ON emp.ManagerId = mgr.EmployeeId;

-- Multiple joins
SELECT e.FirstName, d.Name AS Department, p.ProjectName
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.DepartmentId
INNER JOIN ProjectAssignments pa ON e.EmployeeId = pa.EmployeeId
INNER JOIN Projects p ON pa.ProjectId = p.ProjectId;
```

### Join with Additional Conditions
```sql
SELECT e.FirstName, d.Name
FROM Employees e
INNER JOIN Departments d
    ON e.DepartmentId = d.DepartmentId
    AND d.IsActive = 1;          -- filter in JOIN (before WHERE)
```

---

## 9. AGGREGATE FUNCTIONS & GROUP BY

### Aggregate Functions
```sql
SELECT
    COUNT(*)              AS TotalRows,       -- count all rows
    COUNT(DepartmentId)   AS NonNullDepts,     -- count non-null values
    COUNT(DISTINCT DepartmentId) AS UniqueDepts,
    SUM(Salary)           AS TotalSalary,
    AVG(Salary)           AS AvgSalary,
    MIN(Salary)           AS MinSalary,
    MAX(Salary)           AS MaxSalary,
    STDEV(Salary)         AS StdDevSalary,
    VAR(Salary)           AS VarianceSalary,
    STRING_AGG(FirstName, ', ') AS AllNames    -- SQL Server 2017+
FROM Employees;
```

### GROUP BY
```sql
-- Group and aggregate
SELECT DepartmentId, COUNT(*) AS EmpCount, AVG(Salary) AS AvgSalary
FROM Employees
GROUP BY DepartmentId;

-- Multiple grouping columns
SELECT DepartmentId, IsActive, COUNT(*) AS EmpCount
FROM Employees
GROUP BY DepartmentId, IsActive;

-- ⚠ Every non-aggregated column in SELECT must be in GROUP BY
```

### HAVING (filter groups)
```sql
-- HAVING filters AFTER grouping (WHERE filters BEFORE)
SELECT DepartmentId, AVG(Salary) AS AvgSalary
FROM Employees
GROUP BY DepartmentId
HAVING AVG(Salary) > 60000;

-- Combine WHERE and HAVING
SELECT DepartmentId, COUNT(*) AS ActiveCount
FROM Employees
WHERE IsActive = 1              -- filter rows first
GROUP BY DepartmentId
HAVING COUNT(*) > 5;            -- then filter groups
```

### GROUPING SETS, ROLLUP, CUBE
```sql
-- GROUPING SETS — custom grouping combinations
SELECT DepartmentId, IsActive, COUNT(*) AS Cnt
FROM Employees
GROUP BY GROUPING SETS (
    (DepartmentId, IsActive),
    (DepartmentId),
    ()                          -- grand total
);

-- ROLLUP — hierarchical subtotals
SELECT DepartmentId, IsActive, COUNT(*) AS Cnt
FROM Employees
GROUP BY ROLLUP (DepartmentId, IsActive);
-- Groups: (Dept, Active), (Dept), ()

-- CUBE — all possible combinations
SELECT DepartmentId, IsActive, COUNT(*) AS Cnt
FROM Employees
GROUP BY CUBE (DepartmentId, IsActive);
-- Groups: (Dept, Active), (Dept), (Active), ()
```

---

## 10. SUBQUERIES

### Types of Subqueries
```sql
-- Scalar subquery (returns single value)
SELECT FirstName, Salary,
    (SELECT AVG(Salary) FROM Employees) AS CompanyAvg
FROM Employees;

-- Subquery in WHERE
SELECT * FROM Employees
WHERE Salary > (SELECT AVG(Salary) FROM Employees);

-- Subquery with IN
SELECT * FROM Employees
WHERE DepartmentId IN (
    SELECT DepartmentId FROM Departments WHERE Location = 'New York'
);

-- Subquery with EXISTS
SELECT * FROM Departments d
WHERE EXISTS (
    SELECT 1 FROM Employees e WHERE e.DepartmentId = d.DepartmentId
);

-- Subquery in FROM (derived table)
SELECT dept_stats.DepartmentId, dept_stats.AvgSalary
FROM (
    SELECT DepartmentId, AVG(Salary) AS AvgSalary
    FROM Employees
    GROUP BY DepartmentId
) AS dept_stats
WHERE dept_stats.AvgSalary > 70000;

-- Correlated subquery (references outer query)
SELECT e.FirstName, e.Salary
FROM Employees e
WHERE e.Salary > (
    SELECT AVG(e2.Salary) FROM Employees e2
    WHERE e2.DepartmentId = e.DepartmentId    -- references outer e
);
```

---

## 11. COMMON TABLE EXPRESSIONS (CTE)

### Basic CTE
```sql
WITH HighEarners AS (
    SELECT EmployeeId, FirstName, LastName, Salary
    FROM Employees
    WHERE Salary > 80000
)
SELECT * FROM HighEarners ORDER BY Salary DESC;
```

### Multiple CTEs
```sql
WITH
DeptStats AS (
    SELECT DepartmentId, AVG(Salary) AS AvgSalary, COUNT(*) AS EmpCount
    FROM Employees
    GROUP BY DepartmentId
),
LargeDepts AS (
    SELECT * FROM DeptStats WHERE EmpCount > 10
)
SELECT d.Name, ld.AvgSalary, ld.EmpCount
FROM LargeDepts ld
INNER JOIN Departments d ON ld.DepartmentId = d.DepartmentId;
```

### Recursive CTE
```sql
-- Employee hierarchy (org chart)
WITH OrgChart AS (
    -- Anchor: top-level managers (no manager)
    SELECT EmployeeId, FirstName, ManagerId, 0 AS Level
    FROM Employees
    WHERE ManagerId IS NULL

    UNION ALL

    -- Recursive: employees under each manager
    SELECT e.EmployeeId, e.FirstName, e.ManagerId, oc.Level + 1
    FROM Employees e
    INNER JOIN OrgChart oc ON e.ManagerId = oc.EmployeeId
)
SELECT * FROM OrgChart
ORDER BY Level, FirstName
OPTION (MAXRECURSION 100);   -- default is 100, 0 = unlimited (careful!)

-- Number sequence
WITH Numbers AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM Numbers WHERE n < 100
)
SELECT n FROM Numbers OPTION (MAXRECURSION 100);
```

---

## 12. WINDOW FUNCTIONS

### ROW_NUMBER, RANK, DENSE_RANK, NTILE
```sql
SELECT
    FirstName,
    Salary,
    DepartmentId,
    ROW_NUMBER() OVER (ORDER BY Salary DESC)                        AS RowNum,
    RANK()       OVER (ORDER BY Salary DESC)                        AS SalaryRank,
    DENSE_RANK() OVER (ORDER BY Salary DESC)                        AS DenseRank,
    NTILE(4)     OVER (ORDER BY Salary DESC)                        AS Quartile
FROM Employees;

-- ROW_NUMBER: 1, 2, 3, 4, 5  (no ties)
-- RANK:       1, 2, 2, 4, 5  (skips after tie)
-- DENSE_RANK: 1, 2, 2, 3, 4  (no skip after tie)
-- NTILE(4):   divides into 4 roughly equal groups
```

### PARTITION BY
```sql
-- Rank within each department
SELECT
    FirstName,
    DepartmentId,
    Salary,
    ROW_NUMBER() OVER (PARTITION BY DepartmentId ORDER BY Salary DESC) AS DeptRank
FROM Employees;

-- Top earner per department
WITH Ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY DepartmentId ORDER BY Salary DESC) AS rn
    FROM Employees
)
SELECT * FROM Ranked WHERE rn = 1;
```

### Aggregate Window Functions
```sql
SELECT
    FirstName,
    Salary,
    DepartmentId,
    SUM(Salary)   OVER (PARTITION BY DepartmentId)                     AS DeptTotal,
    AVG(Salary)   OVER (PARTITION BY DepartmentId)                     AS DeptAvg,
    COUNT(*)      OVER (PARTITION BY DepartmentId)                     AS DeptCount,
    SUM(Salary)   OVER (ORDER BY HireDate ROWS BETWEEN UNBOUNDED PRECEDING
                        AND CURRENT ROW)                               AS RunningTotal
FROM Employees;
```

### LAG, LEAD, FIRST_VALUE, LAST_VALUE
```sql
SELECT
    FirstName,
    Salary,
    HireDate,
    LAG(Salary, 1, 0)    OVER (ORDER BY HireDate)   AS PrevSalary,
    LEAD(Salary, 1, 0)   OVER (ORDER BY HireDate)   AS NextSalary,
    FIRST_VALUE(Salary)   OVER (PARTITION BY DepartmentId
                                ORDER BY HireDate)   AS FirstHireSalary,
    LAST_VALUE(Salary)    OVER (PARTITION BY DepartmentId
                                ORDER BY HireDate
                                ROWS BETWEEN UNBOUNDED PRECEDING
                                AND UNBOUNDED FOLLOWING)  AS LastHireSalary
FROM Employees;

-- LAG(column, offset, default)  — value from N rows BEFORE
-- LEAD(column, offset, default) — value from N rows AFTER
```

### Window Frame Clauses
```sql
-- Frame syntax:
-- ROWS BETWEEN <start> AND <end>
-- RANGE BETWEEN <start> AND <end>

-- Options for <start> and <end>:
-- UNBOUNDED PRECEDING    — from first row of partition
-- N PRECEDING            — N rows before current
-- CURRENT ROW            — current row
-- N FOLLOWING            — N rows after current
-- UNBOUNDED FOLLOWING    — to last row of partition

-- Moving average (3-row window)
SELECT
    OrderDate,
    Amount,
    AVG(Amount) OVER (ORDER BY OrderDate
                      ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS MovingAvg
FROM Orders;
```

---

## 13. VIEWS

### Creating Views
```sql
CREATE VIEW vw_ActiveEmployees AS
SELECT EmployeeId, FirstName, LastName, Email, Salary, DepartmentId
FROM Employees
WHERE IsActive = 1;

-- Use like a table
SELECT * FROM vw_ActiveEmployees WHERE Salary > 50000;
```

### Modifying & Dropping Views
```sql
-- Alter
ALTER VIEW vw_ActiveEmployees AS
SELECT EmployeeId, FirstName, LastName, Email, Salary
FROM Employees
WHERE IsActive = 1 AND Salary > 0;

-- Drop
DROP VIEW IF EXISTS vw_ActiveEmployees;

-- Create or replace
CREATE OR ALTER VIEW vw_ActiveEmployees AS
SELECT * FROM Employees WHERE IsActive = 1;
```

### Schema Binding
```sql
-- Prevents altering/dropping underlying tables
CREATE VIEW vw_EmpSummary
WITH SCHEMABINDING
AS
SELECT e.EmployeeId, e.FirstName, d.Name AS Department
FROM dbo.Employees e                -- must use schema-qualified names
INNER JOIN dbo.Departments d ON e.DepartmentId = d.DepartmentId;
```

### Indexed View (Materialized)
```sql
-- Must have SCHEMABINDING
CREATE VIEW vw_DeptSalary
WITH SCHEMABINDING
AS
SELECT DepartmentId, COUNT_BIG(*) AS EmpCount, SUM(Salary) AS TotalSalary
FROM dbo.Employees
GROUP BY DepartmentId;

-- Create clustered index on the view (materializes it)
CREATE UNIQUE CLUSTERED INDEX IX_vw_DeptSalary
ON vw_DeptSalary (DepartmentId);
-- Now the view results are physically stored and auto-updated
```

---

## 14. INDEXES

### What is an Index?
- Data structure that improves query performance (like a book's index).
- **Clustered Index**: Determines physical row order — only ONE per table.
- **Non-Clustered Index**: Separate structure pointing to data — up to 999 per table.

### Creating Indexes
```sql
-- Non-clustered index (most common)
CREATE INDEX IX_Employees_LastName
ON Employees (LastName);

-- Unique index
CREATE UNIQUE INDEX IX_Employees_Email
ON Employees (Email);

-- Composite index
CREATE INDEX IX_Employees_Dept_Salary
ON Employees (DepartmentId, Salary DESC);

-- Clustered index (if no PK or want custom)
CREATE CLUSTERED INDEX IX_Employees_HireDate
ON Employees (HireDate);

-- Include columns (covering index — stored in leaf but not key)
CREATE INDEX IX_Employees_Dept
ON Employees (DepartmentId)
INCLUDE (FirstName, LastName, Salary);

-- Filtered index (partial index)
CREATE INDEX IX_Employees_Active
ON Employees (LastName)
WHERE IsActive = 1;

-- Columnstore index (for analytics/data warehouse)
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_CS_Employees
ON Employees (DepartmentId, Salary, HireDate);
```

### Managing Indexes
```sql
-- View indexes on a table
EXEC sp_helpindex 'Employees';
SELECT * FROM sys.indexes WHERE object_id = OBJECT_ID('Employees');

-- Drop index
DROP INDEX IX_Employees_LastName ON Employees;

-- Rebuild (defrag)
ALTER INDEX IX_Employees_LastName ON Employees REBUILD;

-- Reorganize (lightweight defrag)
ALTER INDEX IX_Employees_LastName ON Employees REORGANIZE;

-- Rebuild all indexes on a table
ALTER INDEX ALL ON Employees REBUILD;

-- Disable index (prevents use, keeps metadata)
ALTER INDEX IX_Employees_LastName ON Employees DISABLE;

-- Index fragmentation
SELECT
    i.name AS IndexName,
    s.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('Employees'), NULL, NULL, 'LIMITED') s
JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id;
-- < 10% = fine, 10-30% = reorganize, > 30% = rebuild
```

### Index Guidelines
| Do Index | Avoid Indexing |
|----------|---------------|
| WHERE clause columns | Frequently updated columns |
| JOIN keys | Low-cardinality columns (e.g., BIT) |
| ORDER BY columns | Small tables (< few hundred rows) |
| Frequently searched columns | Wide columns (VARCHAR(MAX)) |

---

## 15. STORED PROCEDURES

### Creating Stored Procedures
```sql
CREATE PROCEDURE usp_GetEmployeeById
    @EmployeeId INT
AS
BEGIN
    SET NOCOUNT ON;        -- suppress "N rows affected" messages
    SELECT EmployeeId, FirstName, LastName, Email, Salary
    FROM Employees
    WHERE EmployeeId = @EmployeeId;
END;

-- Execute
EXEC usp_GetEmployeeById @EmployeeId = 1;
EXEC usp_GetEmployeeById 1;               -- positional
```

### Parameters (Input, Output, Default)
```sql
CREATE OR ALTER PROCEDURE usp_GetEmployees
    @DepartmentId INT = NULL,               -- optional (default NULL)
    @MinSalary DECIMAL(10,2) = 0,           -- default value
    @TotalCount INT OUTPUT                   -- output parameter
AS
BEGIN
    SET NOCOUNT ON;

    SELECT * FROM Employees
    WHERE (@DepartmentId IS NULL OR DepartmentId = @DepartmentId)
      AND Salary >= @MinSalary;

    SET @TotalCount = @@ROWCOUNT;
END;

-- Execute with output
DECLARE @Count INT;
EXEC usp_GetEmployees @DepartmentId = 5, @MinSalary = 50000, @TotalCount = @Count OUTPUT;
PRINT @Count;
```

### Return Values
```sql
CREATE PROCEDURE usp_CheckEmployeeExists
    @Email NVARCHAR(100)
AS
BEGIN
    IF EXISTS (SELECT 1 FROM Employees WHERE Email = @Email)
        RETURN 1;    -- exists
    RETURN 0;        -- not found
END;

-- Capture return value
DECLARE @Result INT;
EXEC @Result = usp_CheckEmployeeExists @Email = 'ritesh@example.com';
PRINT @Result;
```

### Error Handling in Procedures
```sql
CREATE PROCEDURE usp_TransferEmployee
    @EmployeeId INT,
    @NewDeptId INT
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        UPDATE Employees SET DepartmentId = @NewDeptId WHERE EmployeeId = @EmployeeId;

        IF @@ROWCOUNT = 0
            THROW 50001, 'Employee not found.', 1;

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        THROW;    -- re-throw the error
    END CATCH
END;
```

### Managing Procedures
```sql
-- View all procedures
SELECT name FROM sys.procedures;

-- View procedure definition
EXEC sp_helptext 'usp_GetEmployeeById';

-- Drop
DROP PROCEDURE IF EXISTS usp_GetEmployeeById;

-- Create or alter (SQL Server 2016 SP1+)
CREATE OR ALTER PROCEDURE usp_GetEmployeeById ...
```

---

## 16. FUNCTIONS

### Scalar Functions (return single value)
```sql
CREATE FUNCTION dbo.fn_GetFullName (
    @FirstName NVARCHAR(50),
    @LastName  NVARCHAR(50)
)
RETURNS NVARCHAR(101)
AS
BEGIN
    RETURN @FirstName + ' ' + @LastName;
END;

-- Usage
SELECT dbo.fn_GetFullName(FirstName, LastName) AS FullName FROM Employees;
-- ⚠ Must use schema prefix (dbo.) when calling scalar functions
```

### Inline Table-Valued Functions (iTVF)
```sql
CREATE FUNCTION dbo.fn_GetEmployeesByDept (
    @DeptId INT
)
RETURNS TABLE
AS
RETURN (
    SELECT EmployeeId, FirstName, LastName, Salary
    FROM Employees
    WHERE DepartmentId = @DeptId
);

-- Usage (like a parameterized view)
SELECT * FROM dbo.fn_GetEmployeesByDept(5);

-- CROSS APPLY / OUTER APPLY with iTVF
SELECT d.Name, e.*
FROM Departments d
CROSS APPLY dbo.fn_GetEmployeesByDept(d.DepartmentId) e;
```

### Multi-Statement Table-Valued Functions (MSTVF)
```sql
CREATE FUNCTION dbo.fn_GetEmpStats()
RETURNS @Stats TABLE (
    DepartmentId INT,
    EmpCount INT,
    AvgSalary DECIMAL(10,2)
)
AS
BEGIN
    INSERT INTO @Stats
    SELECT DepartmentId, COUNT(*), AVG(Salary)
    FROM Employees
    GROUP BY DepartmentId;

    RETURN;
END;

-- Usage
SELECT * FROM dbo.fn_GetEmpStats();
-- ⚠ MSTVF can have poor performance — prefer iTVF when possible
```

### Functions vs Stored Procedures
| Feature | Function | Stored Procedure |
|---------|----------|-----------------|
| Return value | Must return a value | Optional (via OUTPUT/RETURN) |
| Use in SELECT | Yes | No |
| DML (INSERT/UPDATE/DELETE) | Not allowed | Allowed |
| TRY-CATCH | Not allowed | Allowed |
| Transactions | Not allowed | Allowed |
| Temp tables | Not allowed | Allowed |
| Side effects | No side effects | Can have side effects |

---

## 17. TRIGGERS

### DML Triggers (INSERT, UPDATE, DELETE)
```sql
-- AFTER trigger (fires after the DML operation)
CREATE TRIGGER trg_Employees_Audit
ON Employees
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    -- INSERTED table: new rows (INSERT/UPDATE)
    -- DELETED table: old rows (DELETE/UPDATE)

    INSERT INTO AuditLog (TableName, Action, OldData, NewData, ModifiedAt)
    SELECT
        'Employees',
        CASE
            WHEN EXISTS (SELECT 1 FROM INSERTED) AND EXISTS (SELECT 1 FROM DELETED) THEN 'UPDATE'
            WHEN EXISTS (SELECT 1 FROM INSERTED) THEN 'INSERT'
            ELSE 'DELETE'
        END,
        (SELECT * FROM DELETED FOR JSON AUTO),
        (SELECT * FROM INSERTED FOR JSON AUTO),
        SYSDATETIME();
END;
```

### INSTEAD OF Trigger
```sql
-- Replaces the original DML action
CREATE TRIGGER trg_vw_ActiveEmployees_Insert
ON vw_ActiveEmployees
INSTEAD OF INSERT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO Employees (FirstName, LastName, Email, Salary, IsActive)
    SELECT FirstName, LastName, Email, Salary, 1
    FROM INSERTED;
END;
```

### DDL Triggers
```sql
-- Database-level trigger
CREATE TRIGGER trg_PreventTableDrop
ON DATABASE
FOR DROP_TABLE, ALTER_TABLE
AS
BEGIN
    PRINT 'Table modifications are not allowed!';
    ROLLBACK;
END;

-- Server-level trigger
CREATE TRIGGER trg_AuditLogin
ON ALL SERVER
FOR LOGON
AS
BEGIN
    IF ORIGINAL_LOGIN() = 'suspicious_user'
        ROLLBACK;    -- deny login
END;
```

### Managing Triggers
```sql
-- Disable
DISABLE TRIGGER trg_Employees_Audit ON Employees;

-- Enable
ENABLE TRIGGER trg_Employees_Audit ON Employees;

-- Drop
DROP TRIGGER IF EXISTS trg_Employees_Audit;

-- View all triggers
SELECT name, parent_id, type_desc FROM sys.triggers;
```

---

## 18. TRANSACTIONS

### Basic Transaction
```sql
BEGIN TRANSACTION;

    UPDATE Accounts SET Balance = Balance - 500 WHERE AccountId = 1;
    UPDATE Accounts SET Balance = Balance + 500 WHERE AccountId = 2;

COMMIT TRANSACTION;

-- Or rollback
ROLLBACK TRANSACTION;
```

### Transaction with Error Handling
```sql
BEGIN TRY
    BEGIN TRANSACTION;

    UPDATE Accounts SET Balance = Balance - 500 WHERE AccountId = 1;
    UPDATE Accounts SET Balance = Balance + 500 WHERE AccountId = 2;

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;

    -- Error details
    SELECT
        ERROR_NUMBER()    AS ErrorNumber,
        ERROR_SEVERITY()  AS Severity,
        ERROR_STATE()     AS State,
        ERROR_PROCEDURE() AS ProcedureName,
        ERROR_LINE()      AS ErrorLine,
        ERROR_MESSAGE()   AS ErrorMessage;
END CATCH
```

### Savepoints
```sql
BEGIN TRANSACTION;

    INSERT INTO Orders (...) VALUES (...);
    SAVE TRANSACTION SavePoint1;

    INSERT INTO OrderItems (...) VALUES (...);
    -- If something goes wrong with items:
    ROLLBACK TRANSACTION SavePoint1;   -- only rolls back to savepoint

COMMIT TRANSACTION;
```

### Transaction Isolation Levels
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;   -- dirty reads allowed
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;      -- default — no dirty reads
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;     -- no non-repeatable reads
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;        -- strictest — range locks
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;             -- row versioning

-- Table-level hint (in query)
SELECT * FROM Employees WITH (NOLOCK);               -- same as READ UNCOMMITTED
SELECT * FROM Employees WITH (READCOMMITTEDLOCK);
SELECT * FROM Employees WITH (HOLDLOCK);             -- same as SERIALIZABLE
```

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|-----------|-------------------|-------------|-------------|
| READ UNCOMMITTED | Yes | Yes | Yes | Fastest |
| READ COMMITTED | No | Yes | Yes | Default |
| REPEATABLE READ | No | No | Yes | Slower |
| SERIALIZABLE | No | No | No | Slowest |
| SNAPSHOT | No | No | No | Row versioning |

---

## 19. ERROR HANDLING

### TRY-CATCH
```sql
BEGIN TRY
    -- code that might fail
    SELECT 1 / 0;    -- division by zero
END TRY
BEGIN CATCH
    SELECT
        ERROR_NUMBER()    AS ErrorNumber,
        ERROR_SEVERITY()  AS Severity,
        ERROR_STATE()     AS State,
        ERROR_MESSAGE()   AS Message,
        ERROR_LINE()      AS Line,
        ERROR_PROCEDURE() AS Procedure;
END CATCH
```

### THROW (SQL Server 2012+)
```sql
-- Raise custom error
THROW 50001, 'Custom error message.', 1;

-- Re-throw in CATCH (preserves original error)
BEGIN CATCH
    THROW;
END CATCH

-- Parameterized error message
DECLARE @msg NVARCHAR(200) = 'Employee ID ' + CAST(@Id AS NVARCHAR) + ' not found.';
THROW 50001, @msg, 1;
```

### RAISERROR (Legacy — still works)
```sql
-- Syntax: RAISERROR (message, severity, state)
RAISERROR ('Something went wrong', 16, 1);

-- With formatting
RAISERROR ('Error in %s: %d records failed', 16, 1, 'Import', 5);

-- Severity levels:
-- 0-10:  Informational (not caught by CATCH)
-- 11-16: User errors (caught by CATCH)
-- 17-19: Resource/software errors
-- 20-25: Fatal errors (terminate connection)
```

### @@ERROR
```sql
-- Legacy error check (before TRY-CATCH)
INSERT INTO Employees (...) VALUES (...);
IF @@ERROR <> 0
    PRINT 'Insert failed';

-- ⚠ @@ERROR resets after EVERY statement — check immediately
```

---

## 20. STRING FUNCTIONS

```sql
-- Length
LEN('Hello')                    -- 5 (excludes trailing spaces)
DATALENGTH('Hello')             -- 5 (bytes — 10 for NVARCHAR)

-- Case
UPPER('hello')                  -- 'HELLO'
LOWER('HELLO')                  -- 'hello'

-- Trim
TRIM('  hello  ')               -- 'hello'         (SQL Server 2017+)
LTRIM('  hello')                -- 'hello'
RTRIM('hello  ')                -- 'hello'

-- Substring
SUBSTRING('Hello World', 7, 5) -- 'World'          (1-based)
LEFT('Hello', 3)                -- 'Hel'
RIGHT('Hello', 3)               -- 'llo'

-- Search
CHARINDEX('World', 'Hello World')     -- 7          (1-based, 0 if not found)
PATINDEX('%[0-9]%', 'abc123')         -- 4          (pattern search)

-- Replace
REPLACE('Hello World', 'World', 'SQL')  -- 'Hello SQL'
STUFF('Hello World', 7, 5, 'SQL')       -- 'Hello SQL'   (position-based replace)
TRANSLATE('2*[3+4]/{7-2}', '[]{}', '()()') -- '2*(3+4)/(7-2)'  (SQL Server 2017+)

-- Concatenation
CONCAT('Hello', ' ', 'World')          -- 'Hello World'  (NULL-safe)
CONCAT_WS(', ', 'Alice', 'Bob', NULL, 'Carol')  -- 'Alice, Bob, Carol'  (SQL Server 2017+)
'Hello' + ' ' + 'World'                -- 'Hello World'  (NULL propagates)

-- Replicate & Padding
REPLICATE('ab', 3)              -- 'ababab'
SPACE(5)                        -- '     '
FORMAT(1234, '0000000')         -- '0001234'         (left-pad with zeros)

-- Reverse & Unicode
REVERSE('Hello')                -- 'olleH'
UNICODE('A')                    -- 65
CHAR(65)                        -- 'A'
NCHAR(65)                       -- N'A'
ASCII('A')                      -- 65

-- STRING_AGG (SQL Server 2017+)
SELECT STRING_AGG(FirstName, ', ') WITHIN GROUP (ORDER BY FirstName)
FROM Employees;

-- STRING_SPLIT (SQL Server 2016+)
SELECT value FROM STRING_SPLIT('a,b,c,d', ',');
-- Returns table: a | b | c | d
```

---

## 21. DATE & TIME FUNCTIONS

```sql
-- Current date/time
GETDATE()                       -- current datetime (DATETIME)
SYSDATETIME()                   -- current datetime (DATETIME2 — more precision)
GETUTCDATE()                    -- current UTC datetime
SYSUTCDATETIME()                -- current UTC DATETIME2
CURRENT_TIMESTAMP               -- same as GETDATE() (ANSI standard)

-- Extract parts
YEAR('2026-04-14')              -- 2026
MONTH('2026-04-14')             -- 4
DAY('2026-04-14')               -- 14
DATEPART(WEEKDAY, '2026-04-14') -- day of week (1=Sun in US)
DATEPART(QUARTER, '2026-04-14') -- 2
DATENAME(MONTH, '2026-04-14')   -- 'April'
DATENAME(WEEKDAY, '2026-04-14') -- 'Tuesday'

-- Date arithmetic
DATEADD(DAY, 7, '2026-04-14')     -- '2026-04-21'    (add 7 days)
DATEADD(MONTH, -3, '2026-04-14')  -- '2026-01-14'    (subtract 3 months)
DATEADD(YEAR, 1, '2026-04-14')    -- '2027-04-14'

DATEDIFF(DAY, '2026-01-01', '2026-04-14')    -- 103 days
DATEDIFF(MONTH, '2025-01-01', '2026-04-14')  -- 15 months
DATEDIFF(YEAR, '2000-01-01', '2026-04-14')   -- 26 years

DATEDIFF_BIG(SECOND, '2000-01-01', SYSDATETIME())  -- large intervals

-- Construct dates
DATEFROMPARTS(2026, 4, 14)                    -- DATE: 2026-04-14
DATETIME2FROMPARTS(2026, 4, 14, 10, 30, 0, 0, 7)  -- DATETIME2
DATETIMEFROMPARTS(2026, 4, 14, 10, 30, 0, 0)       -- DATETIME

-- Convert & format
CAST('2026-04-14' AS DATE)
CONVERT(VARCHAR, GETDATE(), 120)              -- '2026-04-14 10:30:00'   (ISO)
CONVERT(VARCHAR, GETDATE(), 103)              -- '14/04/2026'            (dd/mm/yyyy)
CONVERT(VARCHAR, GETDATE(), 101)              -- '04/14/2026'            (mm/dd/yyyy)
FORMAT(GETDATE(), 'yyyy-MM-dd')               -- '2026-04-14'
FORMAT(GETDATE(), 'dd MMM yyyy HH:mm')        -- '14 Apr 2026 10:30'

-- End of month
EOMONTH('2026-04-14')           -- '2026-04-30'
EOMONTH('2026-04-14', 2)        -- '2026-06-30'     (2 months ahead)

-- Check valid date
ISDATE('2026-02-30')            -- 0 (invalid)
ISDATE('2026-04-14')            -- 1 (valid)

-- Common CONVERT style codes:
-- 101 = mm/dd/yyyy    103 = dd/mm/yyyy    104 = dd.mm.yyyy
-- 108 = HH:mm:ss      112 = yyyymmdd      120 = yyyy-mm-dd HH:mm:ss
-- 126 = ISO 8601       121 = yyyy-mm-dd HH:mm:ss.mmm
```

---

## 22. MATHEMATICAL FUNCTIONS

```sql
ABS(-42)                        -- 42
CEILING(4.2)                    -- 5
FLOOR(4.8)                      -- 4
ROUND(3.14159, 2)               -- 3.14
ROUND(3.14159, 2, 1)            -- 3.14  (truncate mode)
POWER(2, 10)                    -- 1024
SQRT(144)                       -- 12
SQUARE(7)                       -- 49
SIGN(-42)                       -- -1  (returns -1, 0, or 1)
LOG(100)                        -- natural log
LOG10(100)                      -- 2
EXP(1)                          -- 2.718... (e^1)
PI()                            -- 3.14159...
RAND()                          -- random float 0 to 1
RAND(42)                        -- seeded random

-- Random integer between 1 and 100
FLOOR(RAND() * 100) + 1
```

---

## 23. CONVERSION FUNCTIONS

```sql
-- CAST (ANSI standard)
CAST(42 AS VARCHAR(10))           -- '42'
CAST('2026-04-14' AS DATE)        -- date value
CAST(3.14 AS INT)                 -- 3 (truncated)

-- CONVERT (SQL Server specific — supports styles)
CONVERT(VARCHAR(10), 42)          -- '42'
CONVERT(VARCHAR(20), GETDATE(), 120)  -- '2026-04-14 10:30:00'

-- TRY_CAST / TRY_CONVERT (return NULL instead of error)
TRY_CAST('abc' AS INT)           -- NULL (no error)
TRY_CONVERT(INT, 'abc')          -- NULL

-- PARSE / TRY_PARSE (culture-aware — slower)
PARSE('14/04/2026' AS DATE USING 'en-GB')
TRY_PARSE('invalid' AS DATE)     -- NULL

-- IIF (inline if)
IIF(Salary > 80000, 'High', 'Low')

-- CHOOSE
CHOOSE(2, 'Red', 'Green', 'Blue')  -- 'Green' (1-based index)

-- COALESCE (first non-NULL)
COALESCE(MiddleName, FirstName, 'Unknown')

-- ISNULL (SQL Server specific — 2 args only)
ISNULL(MiddleName, 'N/A')

-- NULLIF (returns NULL if equal)
NULLIF(Salary, 0)                -- NULL if salary is 0 (prevents divide-by-zero)
```

---

## 24. CASE EXPRESSION

```sql
-- Simple CASE
SELECT FirstName,
    CASE DepartmentId
        WHEN 1 THEN 'Engineering'
        WHEN 2 THEN 'Marketing'
        WHEN 3 THEN 'Sales'
        ELSE 'Other'
    END AS Department
FROM Employees;

-- Searched CASE
SELECT FirstName, Salary,
    CASE
        WHEN Salary >= 100000 THEN 'Senior'
        WHEN Salary >= 60000  THEN 'Mid'
        WHEN Salary >= 30000  THEN 'Junior'
        ELSE 'Intern'
    END AS Level
FROM Employees;

-- CASE in ORDER BY
SELECT * FROM Employees
ORDER BY
    CASE WHEN IsActive = 1 THEN 0 ELSE 1 END,   -- active first
    LastName;

-- CASE in UPDATE
UPDATE Employees
SET Salary = CASE
    WHEN DepartmentId = 1 THEN Salary * 1.10
    WHEN DepartmentId = 2 THEN Salary * 1.05
    ELSE Salary * 1.03
END;

-- CASE in aggregate
SELECT
    COUNT(CASE WHEN IsActive = 1 THEN 1 END) AS ActiveCount,
    COUNT(CASE WHEN IsActive = 0 THEN 1 END) AS InactiveCount
FROM Employees;
```

---

## 25. SET OPERATORS

```sql
-- UNION (combines, removes duplicates)
SELECT FirstName, LastName FROM Employees
UNION
SELECT FirstName, LastName FROM Contractors;

-- UNION ALL (combines, keeps duplicates — faster)
SELECT FirstName FROM Employees
UNION ALL
SELECT FirstName FROM Contractors;

-- INTERSECT (rows in both)
SELECT Email FROM Employees
INTERSECT
SELECT Email FROM Newsletter_Subscribers;

-- EXCEPT (rows in first but not second)
SELECT Email FROM Employees
EXCEPT
SELECT Email FROM Blacklisted_Emails;

-- Rules:
-- Same number of columns
-- Compatible data types
-- Column names come from first query
-- ORDER BY applies to final result only
```

---

## 26. PIVOT & UNPIVOT

### PIVOT (rows → columns)
```sql
-- Sales by quarter
SELECT *
FROM (
    SELECT SalesPersonId, Quarter, Amount
    FROM Sales
) AS SourceTable
PIVOT (
    SUM(Amount)
    FOR Quarter IN ([Q1], [Q2], [Q3], [Q4])
) AS PivotTable;

-- Dynamic PIVOT
DECLARE @columns NVARCHAR(MAX), @sql NVARCHAR(MAX);

SELECT @columns = STRING_AGG(QUOTENAME(Quarter), ', ')
FROM (SELECT DISTINCT Quarter FROM Sales) AS q;

SET @sql = N'
SELECT *
FROM (SELECT SalesPersonId, Quarter, Amount FROM Sales) AS src
PIVOT (SUM(Amount) FOR Quarter IN (' + @columns + N')) AS pvt';

EXEC sp_executesql @sql;
```

### UNPIVOT (columns → rows)
```sql
SELECT SalesPersonId, Quarter, Amount
FROM (
    SELECT SalesPersonId, Q1, Q2, Q3, Q4
    FROM QuarterlySales
) AS src
UNPIVOT (
    Amount FOR Quarter IN (Q1, Q2, Q3, Q4)
) AS unpvt;
```

---

## 27. DYNAMIC SQL

### sp_executesql (Preferred — parameterized)
```sql
DECLARE @sql NVARCHAR(MAX);
DECLARE @deptId INT = 5;

-- Parameterized (safe from SQL injection)
SET @sql = N'SELECT * FROM Employees WHERE DepartmentId = @DeptId';
EXEC sp_executesql @sql, N'@DeptId INT', @DeptId = @deptId;

-- Dynamic table/column names (cannot be parameterized — use QUOTENAME)
DECLARE @tableName NVARCHAR(128) = N'Employees';

SET @sql = N'SELECT * FROM ' + QUOTENAME(@tableName);
EXEC sp_executesql @sql;
```

### EXEC (Legacy — no parameter support)
```sql
-- ⚠ Susceptible to SQL injection if user input is concatenated
DECLARE @sql NVARCHAR(MAX);
SET @sql = N'SELECT * FROM Employees WHERE DepartmentId = 5';
EXEC (@sql);

-- ⚠ Always prefer sp_executesql over EXEC for dynamic SQL
```

### QUOTENAME — Protecting Identifiers
```sql
-- Wraps identifier with brackets to prevent injection
QUOTENAME('Employees')          -- [Employees]
QUOTENAME('drop table x; --')  -- [drop table x; --]   (safely quoted)
```

---

## 28. CURSORS

### Basic Cursor
```sql
-- ⚠ Cursors are SLOW — prefer set-based operations when possible

DECLARE @FirstName NVARCHAR(50), @Salary DECIMAL(10,2);

DECLARE emp_cursor CURSOR LOCAL FAST_FORWARD FOR
    SELECT FirstName, Salary FROM Employees WHERE IsActive = 1;

OPEN emp_cursor;
FETCH NEXT FROM emp_cursor INTO @FirstName, @Salary;

WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT @FirstName + ': ' + CAST(@Salary AS NVARCHAR);
    FETCH NEXT FROM emp_cursor INTO @FirstName, @Salary;
END

CLOSE emp_cursor;
DEALLOCATE emp_cursor;
```

### Cursor Types
| Type | Description |
|------|-------------|
| `STATIC` | Snapshot — doesn't see changes |
| `DYNAMIC` | Sees all changes — slower |
| `KEYSET` | Sees updates, not inserts/deletes |
| `FAST_FORWARD` | Read-only, forward-only — fastest |
| `LOCAL` | Scoped to batch/procedure |
| `GLOBAL` | Visible to all batches in connection |

---

## 29. VARIABLES & CONTROL FLOW

### Variables
```sql
-- Declare variables
DECLARE @Name NVARCHAR(50) = 'Ritesh';
DECLARE @Age INT;
SET @Age = 25;

-- Multiple declarations
DECLARE @x INT = 1, @y INT = 2, @z INT;

-- Assign from query (single row)
SELECT @Name = FirstName FROM Employees WHERE EmployeeId = 1;

-- Set from query
SET @Name = (SELECT FirstName FROM Employees WHERE EmployeeId = 1);
-- ⚠ SET errors if query returns multiple rows, SELECT takes last value
```

### IF-ELSE
```sql
IF @Age >= 18
BEGIN
    PRINT 'Adult';
END
ELSE IF @Age >= 13
BEGIN
    PRINT 'Teenager';
END
ELSE
BEGIN
    PRINT 'Child';
END
```

### WHILE Loop
```sql
DECLARE @i INT = 1;
WHILE @i <= 10
BEGIN
    PRINT @i;
    SET @i = @i + 1;

    IF @i = 5 CONTINUE;     -- skip rest, next iteration
    IF @i = 8 BREAK;        -- exit loop
END
```

### WAITFOR
```sql
-- Wait for duration
WAITFOR DELAY '00:00:05';      -- wait 5 seconds

-- Wait until specific time
WAITFOR TIME '14:30:00';       -- wait until 2:30 PM
```

### GOTO
```sql
-- Avoid in production — creates spaghetti code
GOTO ErrorHandler;

ErrorHandler:
    PRINT 'Error occurred';
```

---

## 30. JSON SUPPORT (SQL Server 2016+)

### Formatting Results as JSON
```sql
-- FOR JSON AUTO (auto-detects structure from JOINs)
SELECT e.FirstName, e.LastName, d.Name AS Department
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.DepartmentId
FOR JSON AUTO;
-- [{"FirstName":"Ritesh","LastName":"Singh","Department":{"Name":"Engineering"}}]

-- FOR JSON PATH (explicit structure)
SELECT
    EmployeeId AS 'id',
    FirstName  AS 'name.first',
    LastName   AS 'name.last',
    Email      AS 'contact.email'
FROM Employees
FOR JSON PATH, ROOT('employees');
-- {"employees":[{"id":1,"name":{"first":"Ritesh","last":"Singh"},"contact":{"email":"..."}}]}

-- Include NULL values
FOR JSON PATH, INCLUDE_NULL_VALUES;

-- Single object (not array)
FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
```

### Parsing JSON
```sql
DECLARE @json NVARCHAR(MAX) = N'{
    "name": "Ritesh",
    "age": 25,
    "skills": ["Java", "SQL", "Python"],
    "address": {"city": "Delhi", "pin": "110001"}
}';

-- Extract values
SELECT
    JSON_VALUE(@json, '$.name')            AS Name,         -- scalar value
    JSON_VALUE(@json, '$.age')             AS Age,
    JSON_VALUE(@json, '$.address.city')    AS City,
    JSON_VALUE(@json, '$.skills[0]')       AS FirstSkill,
    JSON_QUERY(@json, '$.skills')          AS Skills,       -- array/object
    JSON_QUERY(@json, '$.address')         AS Address;

-- Check valid JSON
SELECT ISJSON(@json);     -- 1 = valid, 0 = invalid
```

### OPENJSON — Parse JSON into Rows
```sql
DECLARE @json NVARCHAR(MAX) = N'[
    {"id": 1, "name": "Ritesh", "salary": 75000},
    {"id": 2, "name": "Alice", "salary": 80000}
]';

-- Default schema (key, value, type)
SELECT * FROM OPENJSON(@json);

-- Explicit schema
SELECT * FROM OPENJSON(@json)
WITH (
    Id     INT           '$.id',
    Name   NVARCHAR(50)  '$.name',
    Salary DECIMAL(10,2) '$.salary'
);

-- Insert JSON data into table
INSERT INTO Employees (EmployeeId, FirstName, Salary)
SELECT Id, Name, Salary FROM OPENJSON(@json)
WITH (
    Id     INT           '$.id',
    Name   NVARCHAR(50)  '$.name',
    Salary DECIMAL(10,2) '$.salary'
);
```

### JSON_MODIFY (SQL Server 2016+)
```sql
DECLARE @json NVARCHAR(MAX) = N'{"name": "Ritesh", "age": 25}';

-- Update value
SET @json = JSON_MODIFY(@json, '$.age', 26);

-- Add new property
SET @json = JSON_MODIFY(@json, '$.city', 'Delhi');

-- Delete property
SET @json = JSON_MODIFY(@json, '$.age', NULL);

-- Append to array
SET @json = JSON_MODIFY(@json, 'append $.skills', 'SQL');
```

---

## 31. XML SUPPORT

### FOR XML
```sql
-- FOR XML RAW
SELECT EmployeeId, FirstName FROM Employees
FOR XML RAW;
-- <row EmployeeId="1" FirstName="Ritesh" />

-- FOR XML RAW with element names
SELECT EmployeeId, FirstName FROM Employees
FOR XML RAW('Employee'), ROOT('Employees'), ELEMENTS;
-- <Employees><Employee><EmployeeId>1</EmployeeId><FirstName>Ritesh</FirstName></Employee></Employees>

-- FOR XML PATH (most flexible)
SELECT
    EmployeeId AS '@Id',
    FirstName  AS 'Name/First',
    LastName   AS 'Name/Last'
FROM Employees
FOR XML PATH('Employee'), ROOT('Employees');

-- FOR XML AUTO
SELECT e.FirstName, d.Name
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.DepartmentId
FOR XML AUTO;
```

### Querying XML
```sql
DECLARE @xml XML = '<employees>
    <employee id="1"><name>Ritesh</name></employee>
    <employee id="2"><name>Alice</name></employee>
</employees>';

-- XQuery methods
SELECT @xml.value('(/employees/employee/name)[1]', 'NVARCHAR(50)');    -- 'Ritesh'
SELECT @xml.query('/employees/employee[@id=2]');                        -- <employee id="2"><name>Alice</name></employee>
SELECT @xml.exist('/employees/employee[@id=3]');                        -- 0 (false)

-- Shred XML into rows
SELECT
    t.c.value('@id', 'INT')           AS Id,
    t.c.value('name[1]', 'NVARCHAR(50)') AS Name
FROM @xml.nodes('/employees/employee') AS t(c);

-- XML MODIFY
SET @xml.modify('replace value of (/employees/employee/name)[1] with "Raj"');
SET @xml.modify('insert <employee id="3"><name>Bob</name></employee>
                 into (/employees)[1]');
SET @xml.modify('delete /employees/employee[@id=2]');
```

---

## 32. SCHEMAS

```sql
-- Create schema
CREATE SCHEMA hr AUTHORIZATION dbo;

-- Create table in schema
CREATE TABLE hr.Employees (
    EmployeeId INT PRIMARY KEY,
    Name NVARCHAR(100)
);

-- Move table to different schema
ALTER SCHEMA hr TRANSFER dbo.Employees;

-- Drop schema (must be empty)
DROP SCHEMA hr;

-- Query across schemas
SELECT * FROM hr.Employees;
SELECT * FROM sales.Orders;

-- List all schemas
SELECT name FROM sys.schemas;
```

---

## 33. USER MANAGEMENT & SECURITY

### Logins (Server Level)
```sql
-- SQL Server authentication login
CREATE LOGIN AppUser WITH PASSWORD = 'Str0ng!Pass#2026';

-- Windows authentication login
CREATE LOGIN [DOMAIN\UserName] FROM WINDOWS;

-- Alter login
ALTER LOGIN AppUser WITH PASSWORD = 'NewStr0ng!Pass#2026';
ALTER LOGIN AppUser DISABLE;
ALTER LOGIN AppUser ENABLE;

-- Drop login
DROP LOGIN AppUser;
```

### Users (Database Level)
```sql
-- Create user mapped to login
USE MyAppDB;
CREATE USER AppUser FOR LOGIN AppUser;

-- User with default schema
CREATE USER AppUser FOR LOGIN AppUser WITH DEFAULT_SCHEMA = hr;

-- Drop user
DROP USER AppUser;
```

### Roles
```sql
-- Fixed server roles: sysadmin, serveradmin, securityadmin, dbcreator, etc.
ALTER SERVER ROLE sysadmin ADD MEMBER AppUser;

-- Fixed database roles:
-- db_owner, db_datareader, db_datawriter, db_ddladmin, db_securityadmin, etc.
ALTER ROLE db_datareader ADD MEMBER AppUser;
ALTER ROLE db_datawriter ADD MEMBER AppUser;

-- Custom database role
CREATE ROLE hr_readonly;
GRANT SELECT ON SCHEMA::hr TO hr_readonly;
ALTER ROLE hr_readonly ADD MEMBER AppUser;

-- Remove from role
ALTER ROLE db_datareader DROP MEMBER AppUser;
```

### Permissions
```sql
-- Grant
GRANT SELECT, INSERT ON Employees TO AppUser;
GRANT EXECUTE ON usp_GetEmployees TO AppUser;
GRANT SELECT ON SCHEMA::hr TO AppUser;

-- Deny (overrides grant)
DENY DELETE ON Employees TO AppUser;

-- Revoke (removes grant or deny)
REVOKE INSERT ON Employees FROM AppUser;

-- View permissions
SELECT * FROM fn_my_permissions('Employees', 'OBJECT');
SELECT * FROM sys.database_permissions WHERE grantee_principal_id = USER_ID('AppUser');
```

### Row-Level Security (RLS)
```sql
-- Filter predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@DeptId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS result
WHERE @DeptId = (SELECT DepartmentId FROM dbo.Employees WHERE Email = USER_NAME());

-- Security policy
CREATE SECURITY POLICY DeptFilter
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(DepartmentId) ON dbo.Employees
WITH (STATE = ON);
```

### Dynamic Data Masking
```sql
-- Mask sensitive data from non-privileged users
ALTER TABLE Employees ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');
ALTER TABLE Employees ALTER COLUMN Salary ADD MASKED WITH (FUNCTION = 'default()');
ALTER TABLE Employees ALTER COLUMN PhoneNumber ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XXX-",4)');

-- Grant unmask permission
GRANT UNMASK TO AppUser;

-- Masking functions:
-- default()              — XXXX for strings, 0 for numbers
-- email()                — aXXX@XXXX.com
-- partial(prefix, pad, suffix)  — first N + padding + last N
-- random(start, end)     — random number in range
```

---

## 34. BACKUP & RESTORE

### Backup Types
```sql
-- Full backup
BACKUP DATABASE MyAppDB
TO DISK = 'C:\Backups\MyAppDB_Full.bak'
WITH FORMAT, INIT, COMPRESSION,
     NAME = 'MyAppDB Full Backup';

-- Differential backup (changes since last full)
BACKUP DATABASE MyAppDB
TO DISK = 'C:\Backups\MyAppDB_Diff.bak'
WITH DIFFERENTIAL, COMPRESSION;

-- Transaction log backup (requires FULL recovery model)
BACKUP LOG MyAppDB
TO DISK = 'C:\Backups\MyAppDB_Log.trn'
WITH COMPRESSION;

-- Copy-only backup (doesn't affect backup chain)
BACKUP DATABASE MyAppDB
TO DISK = 'C:\Backups\MyAppDB_CopyOnly.bak'
WITH COPY_ONLY;
```

### Restore
```sql
-- Restore full backup
RESTORE DATABASE MyAppDB
FROM DISK = 'C:\Backups\MyAppDB_Full.bak'
WITH REPLACE, RECOVERY;    -- RECOVERY = bring online

-- Restore with NORECOVERY (for applying diff/log)
RESTORE DATABASE MyAppDB
FROM DISK = 'C:\Backups\MyAppDB_Full.bak'
WITH NORECOVERY;

RESTORE DATABASE MyAppDB
FROM DISK = 'C:\Backups\MyAppDB_Diff.bak'
WITH NORECOVERY;

RESTORE LOG MyAppDB
FROM DISK = 'C:\Backups\MyAppDB_Log.trn'
WITH RECOVERY;             -- bring online after last restore

-- Restore to new database
RESTORE DATABASE MyAppDB_Test
FROM DISK = 'C:\Backups\MyAppDB_Full.bak'
WITH MOVE 'MyAppDB_Data' TO 'C:\SQLData\MyAppDB_Test.mdf',
     MOVE 'MyAppDB_Log'  TO 'C:\SQLData\MyAppDB_Test.ldf',
     REPLACE;

-- View backup contents
RESTORE HEADERONLY FROM DISK = 'C:\Backups\MyAppDB_Full.bak';
RESTORE FILELISTONLY FROM DISK = 'C:\Backups\MyAppDB_Full.bak';
```

---

## 35. SYSTEM VIEWS & DMVs

### Catalog Views (sys.*)
```sql
-- Databases
SELECT name, database_id, create_date, state_desc FROM sys.databases;

-- Tables
SELECT name, object_id, type_desc FROM sys.tables;

-- Columns
SELECT c.name, t.name AS type, c.max_length, c.is_nullable
FROM sys.columns c
JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE object_id = OBJECT_ID('Employees');

-- Indexes
SELECT name, type_desc, is_unique FROM sys.indexes
WHERE object_id = OBJECT_ID('Employees');

-- Foreign keys
SELECT name, parent_object_id, referenced_object_id FROM sys.foreign_keys;

-- Stored procedures
SELECT name, create_date, modify_date FROM sys.procedures;

-- Views
SELECT name, create_date FROM sys.views;
```

### INFORMATION_SCHEMA
```sql
SELECT * FROM INFORMATION_SCHEMA.TABLES;
SELECT * FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = 'Employees';
SELECT * FROM INFORMATION_SCHEMA.ROUTINES WHERE ROUTINE_TYPE = 'PROCEDURE';
SELECT * FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS WHERE TABLE_NAME = 'Employees';
SELECT * FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE;
```

### Dynamic Management Views (DMVs)
```sql
-- Currently running queries
SELECT r.session_id, r.status, r.command, t.text AS query_text,
       r.cpu_time, r.total_elapsed_time, r.reads, r.writes
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t;

-- Index usage stats
SELECT OBJECT_NAME(s.object_id) AS TableName,
       i.name AS IndexName,
       s.user_seeks, s.user_scans, s.user_lookups, s.user_updates
FROM sys.dm_db_index_usage_stats s
JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.database_id = DB_ID();

-- Missing indexes
SELECT d.statement AS TableName,
       d.equality_columns, d.inequality_columns, d.included_columns,
       s.avg_user_impact, s.user_seeks
FROM sys.dm_db_missing_index_details d
JOIN sys.dm_db_missing_index_groups g ON d.index_handle = g.index_handle
JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
ORDER BY s.avg_user_impact * s.user_seeks DESC;

-- Session info
SELECT session_id, login_name, host_name, program_name, status
FROM sys.dm_exec_sessions WHERE is_user_process = 1;

-- Wait stats (performance bottlenecks)
SELECT TOP 10 wait_type, wait_time_ms, waiting_tasks_count
FROM sys.dm_os_wait_stats
WHERE wait_type NOT LIKE '%SLEEP%'
ORDER BY wait_time_ms DESC;

-- Memory usage
SELECT * FROM sys.dm_os_memory_clerks ORDER BY pages_kb DESC;
```

---

## 36. EXECUTION PLANS & PERFORMANCE

### Viewing Execution Plans
```sql
-- Estimated plan (doesn't execute)
SET SHOWPLAN_TEXT ON;
GO
SELECT * FROM Employees WHERE LastName = 'Singh';
GO
SET SHOWPLAN_TEXT OFF;
GO

-- Actual plan (executes query)
SET STATISTICS PROFILE ON;
GO
SELECT * FROM Employees WHERE LastName = 'Singh';
GO
SET STATISTICS PROFILE OFF;
GO

-- In SSMS: Ctrl+L (estimated plan), Ctrl+M (include actual plan)

-- Query stats
SET STATISTICS IO ON;          -- logical/physical reads
SET STATISTICS TIME ON;        -- CPU and elapsed time
```

### Query Hints
```sql
-- Table hints
SELECT * FROM Employees WITH (NOLOCK);         -- read uncommitted
SELECT * FROM Employees WITH (INDEX(IX_LastName));  -- force index

-- Query hints
SELECT * FROM Employees
WHERE LastName = 'Singh'
OPTION (MAXDOP 4);              -- max parallelism

-- Join hints
SELECT * FROM Employees e
INNER HASH JOIN Departments d ON e.DepartmentId = d.DepartmentId;
-- LOOP JOIN, HASH JOIN, MERGE JOIN

-- Force recompile (fresh plan)
EXEC usp_MyProc WITH RECOMPILE;
-- or in query
SELECT * FROM Employees OPTION (RECOMPILE);
```

### Common Performance Tips
```sql
-- 1. Use SET NOCOUNT ON in procedures
SET NOCOUNT ON;

-- 2. Avoid SELECT * — select only needed columns
SELECT FirstName, LastName FROM Employees;

-- 3. Use EXISTS instead of COUNT for existence checks
IF EXISTS (SELECT 1 FROM Employees WHERE Email = @Email) ...

-- 4. Avoid functions on indexed columns in WHERE
-- ❌ WHERE YEAR(HireDate) = 2026
-- ✅ WHERE HireDate >= '2026-01-01' AND HireDate < '2027-01-01'

-- 5. Use parameterized queries (avoid plan cache bloat)
-- ❌ EXEC ('SELECT * FROM Employees WHERE Id = ' + @id)
-- ✅ EXEC sp_executesql N'SELECT * FROM Employees WHERE Id = @Id', N'@Id INT', @Id = @id

-- 6. Avoid cursors — use set-based operations

-- 7. Update statistics when query plans are suboptimal
UPDATE STATISTICS Employees;

-- 8. Check for parameter sniffing issues
-- Use OPTION (RECOMPILE) or OPTIMIZE FOR hints
SELECT * FROM Employees WHERE DepartmentId = @DeptId
OPTION (OPTIMIZE FOR (@DeptId UNKNOWN));
```

---

## 37. TEMPORAL TABLES (SQL Server 2016+)

### System-Versioned Temporal Tables
```sql
-- Create temporal table
CREATE TABLE Employees (
    EmployeeId    INT PRIMARY KEY,
    FirstName     NVARCHAR(50),
    LastName      NVARCHAR(50),
    Salary        DECIMAL(10,2),
    SysStartTime  DATETIME2 GENERATED ALWAYS AS ROW START NOT NULL,
    SysEndTime    DATETIME2 GENERATED ALWAYS AS ROW END   NOT NULL,
    PERIOD FOR SYSTEM_TIME (SysStartTime, SysEndTime)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.EmployeesHistory));
```

### Querying Historical Data
```sql
-- Current data
SELECT * FROM Employees;

-- Data at a specific point in time
SELECT * FROM Employees
FOR SYSTEM_TIME AS OF '2025-06-15 12:00:00';

-- Data within a range
SELECT * FROM Employees
FOR SYSTEM_TIME BETWEEN '2025-01-01' AND '2025-12-31';

-- All versions (current + history)
SELECT * FROM Employees
FOR SYSTEM_TIME ALL
WHERE EmployeeId = 1
ORDER BY SysStartTime;

-- FOR SYSTEM_TIME options:
-- AS OF <datetime>                             — snapshot at point
-- FROM <start> TO <end>                        — start < SysStart AND SysEnd > end  
-- BETWEEN <start> AND <end>                    — start <= SysStart AND SysEnd > end
-- CONTAINED IN (<start>, <end>)                — both start/end within range
-- ALL                                          — all rows ever
```

### Managing Temporal Tables
```sql
-- Turn off versioning (for maintenance)
ALTER TABLE Employees SET (SYSTEM_VERSIONING = OFF);

-- Turn back on
ALTER TABLE Employees SET (SYSTEM_VERSIONING = ON
    (HISTORY_TABLE = dbo.EmployeesHistory));

-- Clean history
DELETE FROM EmployeesHistory WHERE SysEndTime < '2020-01-01';
```

---

## 38. SEQUENCES

```sql
-- Create sequence
CREATE SEQUENCE dbo.OrderNumberSeq
    AS INT
    START WITH 1000
    INCREMENT BY 1
    MINVALUE 1000
    MAXVALUE 999999
    CYCLE                      -- restart from MINVALUE after MAX (NO CYCLE = error)
    CACHE 50;                  -- pre-allocate for performance

-- Get next value
SELECT NEXT VALUE FOR dbo.OrderNumberSeq;

-- Use in INSERT
INSERT INTO Orders (OrderNumber, CustomerId)
VALUES (NEXT VALUE FOR dbo.OrderNumberSeq, 1);

-- Use as default
ALTER TABLE Orders
ADD CONSTRAINT DF_OrderNumber DEFAULT (NEXT VALUE FOR dbo.OrderNumberSeq) FOR OrderNumber;

-- Reset sequence
ALTER SEQUENCE dbo.OrderNumberSeq RESTART WITH 1000;

-- View sequence info
SELECT * FROM sys.sequences;
```

---

## 39. APPLY OPERATOR (CROSS APPLY / OUTER APPLY)

```sql
-- CROSS APPLY — like INNER JOIN with a table-valued function
-- Returns only rows where the function returns results
SELECT d.Name, e.FirstName, e.Salary
FROM Departments d
CROSS APPLY (
    SELECT TOP 3 FirstName, Salary
    FROM Employees
    WHERE DepartmentId = d.DepartmentId
    ORDER BY Salary DESC
) e;

-- OUTER APPLY — like LEFT JOIN with a table-valued function
-- Returns all left rows (NULL if function returns nothing)
SELECT d.Name, e.FirstName, e.Salary
FROM Departments d
OUTER APPLY (
    SELECT TOP 3 FirstName, Salary
    FROM Employees
    WHERE DepartmentId = d.DepartmentId
    ORDER BY Salary DESC
) e;

-- With table-valued function
SELECT d.Name, e.*
FROM Departments d
CROSS APPLY dbo.fn_GetEmployeesByDept(d.DepartmentId) e;

-- Unpacking JSON/XML per row
SELECT o.OrderId, items.*
FROM Orders o
CROSS APPLY OPENJSON(o.ItemsJson)
WITH (ProductId INT, Quantity INT) AS items;
```

---

## 40. COMMON TABLE PATTERNS

### Pagination
```sql
-- OFFSET-FETCH (SQL Server 2012+)
DECLARE @PageNumber INT = 3, @PageSize INT = 10;

SELECT * FROM Employees
ORDER BY EmployeeId
OFFSET (@PageNumber - 1) * @PageSize ROWS
FETCH NEXT @PageSize ROWS ONLY;

-- With total count
SELECT *, COUNT(*) OVER () AS TotalCount
FROM Employees
ORDER BY EmployeeId
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

### Running Total
```sql
SELECT
    OrderDate,
    Amount,
    SUM(Amount) OVER (ORDER BY OrderDate) AS RunningTotal
FROM Orders;
```

### De-duplication
```sql
-- Keep latest row per email
WITH Ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY Email ORDER BY CreatedAt DESC) AS rn
    FROM Employees
)
DELETE FROM Ranked WHERE rn > 1;
```

### Gaps and Islands
```sql
-- Find gaps in a sequence
WITH Nums AS (
    SELECT EmployeeId,
           LEAD(EmployeeId) OVER (ORDER BY EmployeeId) AS NextId
    FROM Employees
)
SELECT EmployeeId + 1 AS GapStart, NextId - 1 AS GapEnd
FROM Nums
WHERE NextId - EmployeeId > 1;
```

### Recursive Hierarchy
```sql
-- Bill of materials / org chart
WITH Hierarchy AS (
    SELECT EmployeeId, FirstName, ManagerId, 0 AS Level,
           CAST(FirstName AS NVARCHAR(MAX)) AS Path
    FROM Employees WHERE ManagerId IS NULL

    UNION ALL

    SELECT e.EmployeeId, e.FirstName, e.ManagerId, h.Level + 1,
           h.Path + ' > ' + e.FirstName
    FROM Employees e
    INNER JOIN Hierarchy h ON e.ManagerId = h.EmployeeId
)
SELECT * FROM Hierarchy ORDER BY Path;
```

---

## 41. SQL SERVER AGENT & JOBS

### Creating a Job (via T-SQL)
```sql
-- Requires msdb database
USE msdb;

-- Create job
EXEC sp_add_job
    @job_name = N'Daily_Backup',
    @enabled = 1,
    @description = N'Daily full backup of MyAppDB';

-- Add job step
EXEC sp_add_jobstep
    @job_name = N'Daily_Backup',
    @step_name = N'Backup_Step',
    @subsystem = N'TSQL',
    @command = N'BACKUP DATABASE MyAppDB TO DISK = ''C:\Backups\MyAppDB.bak'' WITH INIT, COMPRESSION',
    @database_name = N'MyAppDB';

-- Add schedule (daily at 2 AM)
EXEC sp_add_schedule
    @schedule_name = N'DailyAt2AM',
    @freq_type = 4,               -- daily
    @freq_interval = 1,
    @active_start_time = 020000;  -- 02:00:00

EXEC sp_attach_schedule
    @job_name = N'Daily_Backup',
    @schedule_name = N'DailyAt2AM';

-- Assign to server
EXEC sp_add_jobserver
    @job_name = N'Daily_Backup',
    @server_name = N'(LOCAL)';
```

### Managing Jobs
```sql
-- View all jobs
SELECT job_id, name, enabled, date_created FROM msdb.dbo.sysjobs;

-- Job history
SELECT j.name, h.step_name, h.run_date, h.run_time, h.run_status
FROM msdb.dbo.sysjobhistory h
JOIN msdb.dbo.sysjobs j ON h.job_id = j.job_id
ORDER BY h.run_date DESC, h.run_time DESC;

-- Run job manually
EXEC sp_start_job @job_name = N'Daily_Backup';

-- Disable job
EXEC sp_update_job @job_name = N'Daily_Backup', @enabled = 0;

-- Delete job
EXEC sp_delete_job @job_name = N'Daily_Backup';
```

---

## 42. USEFUL SYSTEM STORED PROCEDURES

```sql
-- Database info
EXEC sp_helpdb;                          -- all databases
EXEC sp_helpdb 'MyAppDB';               -- specific database

-- Table info
EXEC sp_help 'Employees';               -- structure, indexes, constraints
EXEC sp_helpindex 'Employees';          -- indexes only
EXEC sp_helptext 'usp_GetEmployees';    -- procedure/view/function/trigger definition
EXEC sp_columns 'Employees';            -- column details

-- Rename
EXEC sp_rename 'OldTableName', 'NewTableName';
EXEC sp_rename 'Employees.OldCol', 'NewCol', 'COLUMN';

-- Dependencies
EXEC sp_depends 'Employees';            -- objects that depend on this

-- Space used
EXEC sp_spaceused 'Employees';          -- rows, data size, index size

-- Who's connected
EXEC sp_who;
EXEC sp_who2;

-- Active locks
EXEC sp_lock;

-- Server configuration
EXEC sp_configure;
EXEC sp_configure 'max degree of parallelism', 4;
RECONFIGURE;

-- Send email (Database Mail must be configured)
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'DefaultProfile',
    @recipients = 'admin@example.com',
    @subject = 'Alert',
    @body = 'Something happened.';
```

---

## 43. DBCC COMMANDS

```sql
-- Check database integrity
DBCC CHECKDB ('MyAppDB');
DBCC CHECKTABLE ('Employees');

-- Show table/index space
DBCC SHOWCONTIG ('Employees');           -- fragmentation info

-- Shrink database (use sparingly — causes fragmentation)
DBCC SHRINKDATABASE ('MyAppDB', 10);     -- shrink to 10% free

-- Shrink specific file
DBCC SHRINKFILE ('MyAppDB_Data', 100);   -- shrink to 100 MB

-- Free memory
DBCC FREEPROCCACHE;                      -- clear plan cache
DBCC DROPCLEANBUFFERS;                   -- clear buffer pool (dev/test only)

-- Update usage stats
DBCC UPDATEUSAGE ('MyAppDB');

-- Identity column
DBCC CHECKIDENT ('Employees', RESEED, 0);     -- reset identity

-- Trace flags
DBCC TRACEON (1222, -1);                 -- enable deadlock logging
DBCC TRACEOFF (1222, -1);
DBCC TRACESTATUS;
```

---

## 44. COMMON T-SQL PATTERNS & TIPS

### Conditional Insert/Update (UPSERT without MERGE)
```sql
IF EXISTS (SELECT 1 FROM Employees WHERE Email = @Email)
    UPDATE Employees SET Salary = @Salary WHERE Email = @Email;
ELSE
    INSERT INTO Employees (FirstName, Email, Salary) VALUES (@Name, @Email, @Salary);
```

### Handling NULLs
```sql
ISNULL(value, replacement)              -- 2 args, SQL Server specific
COALESCE(val1, val2, val3)              -- first non-NULL, ANSI standard
NULLIF(a, b)                            -- NULL if a = b

-- NULL-safe comparison
WHERE ISNULL(DepartmentId, -1) = ISNULL(@DeptId, -1)
```

### Generating Number / Date Series
```sql
-- Numbers 1-1000 using CTE
WITH Numbers AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM Numbers WHERE n < 1000
)
SELECT n FROM Numbers OPTION (MAXRECURSION 1000);

-- Date series
WITH Dates AS (
    SELECT CAST('2026-01-01' AS DATE) AS dt
    UNION ALL
    SELECT DATEADD(DAY, 1, dt) FROM Dates WHERE dt < '2026-12-31'
)
SELECT dt FROM Dates OPTION (MAXRECURSION 366);
```

### String Splitting & Aggregation
```sql
-- Split (SQL Server 2016+)
SELECT value FROM STRING_SPLIT('a,b,c,d', ',');

-- Aggregate (SQL Server 2017+)
SELECT DepartmentId,
       STRING_AGG(FirstName, ', ') WITHIN GROUP (ORDER BY FirstName) AS Employees
FROM Employees
GROUP BY DepartmentId;

-- Pre-2017: FOR XML PATH trick
SELECT DepartmentId,
    STUFF((
        SELECT ', ' + FirstName
        FROM Employees e2
        WHERE e2.DepartmentId = e1.DepartmentId
        FOR XML PATH('')
    ), 1, 2, '') AS Employees
FROM Employees e1
GROUP BY DepartmentId;
```

### TRY_* Safe Conversions
```sql
TRY_CAST('abc' AS INT)           -- NULL instead of error
TRY_CONVERT(DATE, 'not-a-date')  -- NULL
TRY_PARSE('invalid' AS INT)      -- NULL
```

---

## 45. QUICK REFERENCE CHEAT SHEET

### Query Execution Order
```
1. FROM / JOINs        — source tables
2. WHERE               — row filter
3. GROUP BY            — grouping
4. HAVING              — group filter
5. SELECT              — columns / expressions
6. DISTINCT            — remove duplicates
7. ORDER BY            — sorting
8. OFFSET / FETCH      — pagination
```

### Common Date Formats (CONVERT styles)
| Style | Format | Example |
|-------|--------|---------|
| 101 | mm/dd/yyyy | 04/14/2026 |
| 103 | dd/mm/yyyy | 14/04/2026 |
| 104 | dd.mm.yyyy | 14.04.2026 |
| 108 | HH:mm:ss | 10:30:00 |
| 112 | yyyymmdd | 20260414 |
| 120 | yyyy-mm-dd HH:mm:ss | 2026-04-14 10:30:00 |
| 126 | ISO 8601 | 2026-04-14T10:30:00 |

### Wildcard Characters (LIKE)
| Pattern | Description |
|---------|-------------|
| `%` | Zero or more characters |
| `_` | Exactly one character |
| `[abc]` | Any one char in set |
| `[a-z]` | Any one char in range |
| `[^abc]` | Any char NOT in set |

### Useful System Functions
```sql
@@VERSION               -- SQL Server version
@@SERVERNAME            -- server name
@@SPID                  -- current session ID
DB_ID()                 -- current database ID
DB_NAME()               -- current database name
OBJECT_ID('table')      -- object ID of a table/view
OBJECT_NAME(id)         -- name from object ID
USER_NAME()             -- current user
SUSER_SNAME()           -- current login
HOST_NAME()             -- client machine name
APP_NAME()              -- client application name
@@ROWCOUNT              -- rows affected by last statement
@@TRANCOUNT             -- active transaction count
@@IDENTITY              -- last identity value (any scope)
SCOPE_IDENTITY()        -- last identity value (current scope)
NEWID()                 -- generate new GUID
NEWSEQUENTIALID()       -- sequential GUID (for clustered index)
```

### Comparison: DELETE vs TRUNCATE vs DROP
| Feature | DELETE | TRUNCATE | DROP |
|---------|--------|----------|------|
| WHERE clause | Yes | No | N/A |
| Logging | Full | Minimal | Minimal |
| Identity reset | No | Yes | N/A |
| Triggers fired | Yes | No | No |
| Rollback | Yes | Yes | Yes |
| Removes structure | No | No | Yes |
| FK constraint | Works | Fails if FK | Fails if FK |

---

*Complete SQL Server Reference — All 45 Topics*
*Last updated: April 2026*
