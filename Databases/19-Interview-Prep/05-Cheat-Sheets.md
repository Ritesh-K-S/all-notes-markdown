# 📋 Chapter 8.5 — Database Cheat Sheets — Quick Reference Cards

> **"The best cheat sheet is the one you never need — but always have."** — Every developer during a production incident at 3 AM

---

## 📌 Metadata

| Field | Value |
|-------|-------|
| **Level** | 🟢 All Levels |
| **Purpose** | Quick reference — NOT for learning, but for recalling |
| **Best Used** | Print it, bookmark it, pin it to your wall |

---

## 📋 What's Inside

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CHEAT SHEETS INDEX                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SHEET 1:  SQL Syntax (DDL + DML + Queries)                        │
│  SHEET 2:  SQL Window Functions                                    │
│  SHEET 3:  PostgreSQL Quick Reference                               │
│  SHEET 4:  MySQL Quick Reference                                    │
│  SHEET 5:  SQL Server Quick Reference                               │
│  SHEET 6:  Oracle Quick Reference                                   │
│  SHEET 7:  MongoDB Quick Reference                                  │
│  SHEET 8:  Redis Quick Reference                                    │
│  SHEET 9:  Database Concepts (ACID, CAP, Indexes)                  │
│  SHEET 10: SQL vs NoSQL Decision Matrix                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

# 📄 SHEET 1: SQL Syntax Cheat Sheet

---

## DDL (Data Definition Language)

```sql
-- CREATE TABLE
CREATE TABLE users (
    id          INT PRIMARY KEY AUTO_INCREMENT,  -- MySQL
    id          SERIAL PRIMARY KEY,              -- PostgreSQL
    id          INT IDENTITY(1,1) PRIMARY KEY,   -- SQL Server
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(255) UNIQUE,
    age         INT CHECK (age >= 0 AND age <= 150),
    salary      DECIMAL(10,2) DEFAULT 0.00,
    dept_id     INT REFERENCES departments(id),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ALTER TABLE
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE users DROP COLUMN phone;
ALTER TABLE users ALTER COLUMN name TYPE VARCHAR(200);     -- PostgreSQL
ALTER TABLE users MODIFY COLUMN name VARCHAR(200);          -- MySQL
ALTER TABLE users ALTER COLUMN name VARCHAR(200);           -- SQL Server
ALTER TABLE users RENAME COLUMN name TO full_name;
ALTER TABLE users ADD CONSTRAINT uk_email UNIQUE (email);
ALTER TABLE users DROP CONSTRAINT uk_email;

-- DROP / TRUNCATE
DROP TABLE users;                    -- Delete table structure + data
DROP TABLE IF EXISTS users;          -- No error if missing
TRUNCATE TABLE users;                -- Delete all data, keep structure (fast!)
```

---

## DML (Data Manipulation Language)

```sql
-- INSERT
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
INSERT INTO users (name, email) VALUES 
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com');
INSERT INTO target_table SELECT * FROM source_table WHERE condition;

-- UPDATE
UPDATE users SET salary = salary * 1.10 WHERE department = 'Engineering';
UPDATE users SET salary = 50000, dept_id = 2 WHERE id = 1;

-- DELETE
DELETE FROM users WHERE last_login < '2023-01-01';
DELETE FROM users;                   -- Delete all rows (logged, slow)

-- UPSERT
-- PostgreSQL:
INSERT INTO users (id, name) VALUES (1, 'Alice')
ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name;
-- MySQL:
INSERT INTO users (id, name) VALUES (1, 'Alice')
ON DUPLICATE KEY UPDATE name = VALUES(name);
-- SQL Server:
MERGE INTO users AS target
USING (VALUES (1, 'Alice')) AS source(id, name)
ON target.id = source.id
WHEN MATCHED THEN UPDATE SET name = source.name
WHEN NOT MATCHED THEN INSERT (id, name) VALUES (source.id, source.name);
```

---

## Queries (SELECT)

```sql
-- FULL SELECT SYNTAX (execution order numbered)
SELECT DISTINCT column1, column2, AGG(col3)  -- 5. Select
FROM table1                                    -- 1. From
JOIN table2 ON table1.id = table2.fk           -- 2. Join
WHERE condition                                -- 3. Where
GROUP BY column1, column2                      -- 4. Group
HAVING AGG(col3) > value                       -- 6. Having
ORDER BY column1 DESC                          -- 7. Order
LIMIT 10 OFFSET 20;                            -- 8. Limit

-- JOINs
INNER JOIN      -- Only matching rows from both tables
LEFT JOIN       -- All from left + matching from right (NULL if no match)
RIGHT JOIN      -- All from right + matching from left
FULL OUTER JOIN -- All from both (NULL where no match)
CROSS JOIN      -- Cartesian product (every row × every row)
SELF JOIN       -- Table joined with itself

-- SUBQUERIES
WHERE col IN (SELECT col FROM other_table)
WHERE col > (SELECT AVG(col) FROM table)
WHERE EXISTS (SELECT 1 FROM table WHERE condition)
FROM (SELECT ... FROM table) AS subquery

-- SET OPERATIONS
SELECT ... UNION SELECT ...        -- Combine + deduplicate
SELECT ... UNION ALL SELECT ...    -- Combine (keep duplicates, faster)
SELECT ... INTERSECT SELECT ...    -- Only in both
SELECT ... EXCEPT SELECT ...       -- In first but not second
```

---

## Common Functions

```sql
-- STRING
CONCAT(str1, str2)          LENGTH(str)           UPPER(str) / LOWER(str)
TRIM(str)                   SUBSTRING(str, pos, len)  REPLACE(str, old, new)
LEFT(str, n)                RIGHT(str, n)         REVERSE(str)
LPAD(str, len, pad)         SPLIT_PART(str, delim, n)  -- PostgreSQL

-- NUMERIC
ROUND(num, decimals)        CEIL(num) / FLOOR(num)    ABS(num)
MOD(num, divisor)           POWER(base, exp)           SQRT(num)
GREATEST(a, b, c)           LEAST(a, b, c)

-- DATE/TIME
CURRENT_DATE                CURRENT_TIMESTAMP          NOW()
DATE_TRUNC('month', dt)     -- PostgreSQL
EXTRACT(YEAR FROM dt)       DATE_ADD(dt, INTERVAL 1 DAY) -- MySQL
DATEADD(day, 1, dt)         -- SQL Server
DATEDIFF(day, dt1, dt2)     -- SQL Server / MySQL
AGE(dt1, dt2)               -- PostgreSQL

-- NULL HANDLING
COALESCE(a, b, c)           -- First non-null value
NULLIF(a, b)                -- Returns NULL if a = b
IFNULL(a, b)                -- MySQL: if a is null, return b
ISNULL(a, b)                -- SQL Server: if a is null, return b
NVL(a, b)                   -- Oracle: if a is null, return b

-- CONDITIONAL
CASE WHEN cond1 THEN val1 WHEN cond2 THEN val2 ELSE default END
IF(condition, true_val, false_val)  -- MySQL
IIF(condition, true_val, false_val) -- SQL Server
```

---

## Aggregate Functions

```sql
COUNT(*)          -- Count all rows
COUNT(column)     -- Count non-NULL values
COUNT(DISTINCT c) -- Count unique non-NULL values
SUM(column)       -- Total
AVG(column)       -- Average
MIN(column)       -- Minimum
MAX(column)       -- Maximum
STRING_AGG(col, ', ')   -- PostgreSQL: concatenate strings
GROUP_CONCAT(col)       -- MySQL: concatenate strings
ARRAY_AGG(col)          -- PostgreSQL: collect into array
```

---

# 📄 SHEET 2: SQL Window Functions Cheat Sheet

```
SYNTAX:
  function_name() OVER (
      [PARTITION BY col1, col2]
      [ORDER BY col3 ASC|DESC]
      [frame_clause]
  )

FRAME CLAUSE:
  ROWS BETWEEN start AND end
  RANGE BETWEEN start AND end
  
  start/end options:
    UNBOUNDED PRECEDING    -- From first row
    N PRECEDING            -- N rows before current
    CURRENT ROW            -- Current row
    N FOLLOWING            -- N rows after current
    UNBOUNDED FOLLOWING    -- To last row
```

```
┌────────────────────┬───────────────────────────────────────────────────────────┐
│ Function           │ Description & Example                                    │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ ROW_NUMBER()       │ Unique sequential number (1,2,3 — no ties)              │
│                    │ ROW_NUMBER() OVER (ORDER BY salary DESC)                │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ RANK()             │ Rank with gaps (1,2,2,4 — ties get same rank)          │
│                    │ RANK() OVER (PARTITION BY dept ORDER BY salary DESC)    │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ DENSE_RANK()       │ Rank without gaps (1,2,2,3 — no gap after tie)         │
│                    │ DENSE_RANK() OVER (ORDER BY salary DESC)                │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ NTILE(n)           │ Divide into n buckets (quartiles: NTILE(4))            │
│                    │ NTILE(4) OVER (ORDER BY salary)                         │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ LAG(col, n)        │ Value from n rows BEFORE (default n=1)                 │
│                    │ LAG(salary, 1) OVER (ORDER BY hire_date)               │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ LEAD(col, n)       │ Value from n rows AFTER                                │
│                    │ LEAD(salary, 1) OVER (ORDER BY hire_date)              │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ FIRST_VALUE(col)   │ First value in the window frame                        │
│                    │ FIRST_VALUE(name) OVER (ORDER BY salary DESC)          │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ LAST_VALUE(col)    │ Last value in frame (⚠️ needs ROWS BETWEEN)            │
│                    │ LAST_VALUE(name) OVER (ORDER BY sal ROWS BETWEEN       │
│                    │   UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)          │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ NTH_VALUE(col, n)  │ Nth value in the frame                                 │
│                    │ NTH_VALUE(name, 2) OVER (ORDER BY salary DESC)         │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ PERCENT_RANK()     │ Relative rank: (rank-1)/(rows-1) → 0 to 1             │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ CUME_DIST()        │ Cumulative distribution: rows_up_to/total_rows         │
├────────────────────┼───────────────────────────────────────────────────────────┤
│ SUM/AVG/MIN/MAX    │ Aggregate as window function (running totals, etc.)    │
│                    │ SUM(amount) OVER (ORDER BY date)                       │
└────────────────────┴───────────────────────────────────────────────────────────┘
```

### Common Patterns

```sql
-- Running Total
SUM(amount) OVER (ORDER BY date)

-- Moving Average (3 rows)
AVG(amount) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

-- Percentage of Total
amount * 100.0 / SUM(amount) OVER ()

-- Top N per Group
ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) <= 3

-- Year-over-Year
amount - LAG(amount, 12) OVER (ORDER BY month)

-- Gaps and Islands
date - ROW_NUMBER() OVER (ORDER BY date) AS grp
```

---

# 📄 SHEET 3: PostgreSQL Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│                  POSTGRESQL CHEAT SHEET                             │
├─────────────────────────────────────────────────────────────────────┤
│ Default Port: 5432          │ Config: postgresql.conf              │
│ Max Connections: 100        │ Auth: pg_hba.conf                    │
│ Data Dir: /var/lib/pgsql    │ Logs: pg_log/                       │
└─────────────────────────────┴──────────────────────────────────────┘
```

### Connection & psql Commands

```bash
# Connect
psql -U postgres -d mydb -h localhost -p 5432
PGPASSWORD=secret psql -U user -d db

# psql meta-commands
\l                  -- List databases
\c dbname           -- Connect to database
\dt                 -- List tables
\dt+                -- List tables with sizes
\d table_name       -- Describe table structure
\di                 -- List indexes
\du                 -- List roles/users
\df                 -- List functions
\dv                 -- List views
\x                  -- Toggle expanded display
\timing             -- Toggle query timing
\i file.sql         -- Execute SQL file
\copy               -- Client-side COPY
\q                  -- Quit
```

### PostgreSQL-Specific Features

```sql
-- SERIAL / IDENTITY
CREATE TABLE users (
    id SERIAL PRIMARY KEY,              -- Legacy
    id INT GENERATED ALWAYS AS IDENTITY  -- Modern (SQL standard)
);

-- ARRAY TYPE
SELECT ARRAY[1, 2, 3];
CREATE TABLE t (tags TEXT[]);
SELECT * FROM t WHERE 'sql' = ANY(tags);
SELECT * FROM t WHERE tags @> ARRAY['sql'];

-- JSON/JSONB
CREATE TABLE events (data JSONB);
INSERT INTO events VALUES ('{"name": "click", "page": "/home"}');
SELECT data->>'name' FROM events;           -- Text extraction
SELECT data->'meta'->>'browser' FROM events; -- Nested
SELECT * FROM events WHERE data @> '{"name": "click"}';
CREATE INDEX ON events USING GIN (data);    -- Index JSONB

-- CTE (WITH)
WITH RECURSIVE tree AS (
    SELECT id, parent_id, name, 1 AS depth FROM org WHERE parent_id IS NULL
    UNION ALL
    SELECT o.id, o.parent_id, o.name, t.depth + 1
    FROM org o JOIN tree t ON o.parent_id = t.id
)
SELECT * FROM tree;

-- LATERAL JOIN
SELECT u.name, recent.*
FROM users u
CROSS JOIN LATERAL (
    SELECT * FROM orders WHERE user_id = u.id ORDER BY created_at DESC LIMIT 3
) recent;

-- GENERATE_SERIES
SELECT generate_series('2024-01-01'::date, '2024-12-31'::date, '1 month');
SELECT generate_series(1, 100);

-- DISTINCT ON
SELECT DISTINCT ON (department) emp_name, department, salary
FROM employees ORDER BY department, salary DESC;

-- RETURNING
INSERT INTO users (name) VALUES ('Alice') RETURNING id;
UPDATE users SET age = 30 WHERE id = 1 RETURNING *;
DELETE FROM users WHERE id = 1 RETURNING *;

-- EXPLAIN
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;

-- VACUUM
VACUUM ANALYZE table_name;     -- Update statistics + reclaim space
VACUUM FULL table_name;        -- Rewrite table (locks!) 

-- PARTITIONING
CREATE TABLE logs (id INT, created_at DATE, msg TEXT)
PARTITION BY RANGE (created_at);
CREATE TABLE logs_2024_q1 PARTITION OF logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

-- LISTEN/NOTIFY (pub/sub)
LISTEN channel_name;
NOTIFY channel_name, 'payload';

-- FULL TEXT SEARCH
SELECT * FROM articles
WHERE to_tsvector('english', body) @@ to_tsquery('postgres & performance');
```

### Key pg_catalog Views

```sql
pg_stat_activity        -- Active connections / running queries
pg_stat_user_tables     -- Table statistics (seq scans, index usage)
pg_stat_user_indexes    -- Index usage stats
pg_locks                -- Current locks
pg_stat_replication     -- Replication status
pg_settings             -- Configuration parameters
```

---

# 📄 SHEET 4: MySQL Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│                  MYSQL CHEAT SHEET                                  │
├─────────────────────────────────────────────────────────────────────┤
│ Default Port: 3306          │ Config: my.cnf / my.ini              │
│ Storage Engines: InnoDB*    │ Max Connections: 151 (default)       │
│ Data Dir: /var/lib/mysql    │ * = default since MySQL 5.5          │
└─────────────────────────────┴──────────────────────────────────────┘
```

### Connection & Admin

```bash
# Connect
mysql -u root -p
mysql -u user -p -h localhost -P 3306 dbname

# Admin commands
SHOW DATABASES;
SHOW TABLES;
SHOW CREATE TABLE table_name;
DESCRIBE table_name;         -- or DESC
SHOW PROCESSLIST;
SHOW VARIABLES LIKE 'innodb%';
SHOW STATUS LIKE 'Threads%';
SHOW INDEX FROM table_name;
SHOW GRANTS FOR 'user'@'host';
```

### MySQL-Specific Features

```sql
-- AUTO_INCREMENT
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100)
) ENGINE=InnoDB;

-- ON DUPLICATE KEY UPDATE (Upsert)
INSERT INTO users (id, name, score) VALUES (1, 'Alice', 100)
ON DUPLICATE KEY UPDATE score = VALUES(score);

-- GROUP_CONCAT
SELECT dept, GROUP_CONCAT(name ORDER BY name SEPARATOR ', ')
FROM employees GROUP BY dept;

-- LIMIT with OFFSET
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 20;

-- IF / IFNULL
SELECT IF(age > 18, 'Adult', 'Minor') FROM users;
SELECT IFNULL(phone, 'N/A') FROM users;

-- JSON (MySQL 5.7+)
CREATE TABLE events (data JSON);
SELECT JSON_EXTRACT(data, '$.name') FROM events;
SELECT data->>"$.name" FROM events;  -- Shorthand

-- EXPLAIN
EXPLAIN SELECT ...;
EXPLAIN ANALYZE SELECT ...;  -- MySQL 8.0.18+

-- USER / GRANTS
CREATE USER 'app'@'%' IDENTIFIED BY 'password';
GRANT SELECT, INSERT ON mydb.* TO 'app'@'%';
FLUSH PRIVILEGES;

-- COMMON TABLE EXPRESSIONS (MySQL 8.0+)
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn <= 5;

-- WINDOW FUNCTIONS (MySQL 8.0+)
SELECT name, salary,
    ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn
FROM employees;
```

### Storage Engines Comparison

```
┌──────────┬──────────────┬──────────────┬──────────────┐
│ Feature  │ InnoDB       │ MyISAM       │ MEMORY       │
├──────────┼──────────────┼──────────────┼──────────────┤
│ ACID     │ ✅ Yes        │ ❌ No        │ ❌ No        │
│ Locking  │ Row-level    │ Table-level  │ Table-level  │
│ FK       │ ✅ Yes        │ ❌ No        │ ❌ No        │
│ Full-text│ ✅ (5.6+)     │ ✅ Yes       │ ❌ No        │
│ Speed    │ Good         │ Fast reads   │ Fastest      │
│ Crash    │ Recovery ✅   │ ❌ Corrupt   │ Lost on restart│
│ Use case │ Default      │ Read-heavy   │ Cache/temp   │
└──────────┴──────────────┴──────────────┴──────────────┘
```

---

# 📄 SHEET 5: SQL Server Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│                  SQL SERVER CHEAT SHEET                             │
├─────────────────────────────────────────────────────────────────────┤
│ Default Port: 1433          │ Config: SSMS / sp_configure          │
│ Editions: Express/Standard/ │ Max DB Size (Express): 10 GB         │
│   Enterprise/Developer     │ Auth: Windows + SQL Server            │
└─────────────────────────────┴──────────────────────────────────────┘
```

### SQL Server-Specific Syntax

```sql
-- TOP (instead of LIMIT)
SELECT TOP 10 * FROM users ORDER BY id;
SELECT TOP 10 PERCENT * FROM users;
SELECT TOP 10 WITH TIES * FROM users ORDER BY salary DESC;

-- IDENTITY
CREATE TABLE users (
    id INT IDENTITY(1,1) PRIMARY KEY,
    name NVARCHAR(100)
);

-- STRING_AGG (2017+)
SELECT dept, STRING_AGG(name, ', ') WITHIN GROUP (ORDER BY name)
FROM employees GROUP BY dept;

-- OFFSET FETCH (2012+ — standard pagination)
SELECT * FROM users ORDER BY id
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- TRY_CAST / TRY_CONVERT
SELECT TRY_CAST('abc' AS INT);  -- Returns NULL instead of error

-- IIF (inline IF)
SELECT IIF(salary > 100000, 'High', 'Normal') FROM employees;

-- FORMAT
SELECT FORMAT(GETDATE(), 'yyyy-MM-dd HH:mm:ss');
SELECT FORMAT(1234567.89, 'C', 'en-US');  -- $1,234,567.89

-- CROSS APPLY / OUTER APPLY (like LATERAL JOIN)
SELECT u.name, recent.*
FROM users u
CROSS APPLY (
    SELECT TOP 3 * FROM orders WHERE user_id = u.id ORDER BY date DESC
) recent;

-- MERGE (Upsert)
MERGE INTO target AS t
USING source AS s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET t.name = s.name
WHEN NOT MATCHED THEN INSERT (id, name) VALUES (s.id, s.name)
WHEN NOT MATCHED BY SOURCE THEN DELETE;

-- TEMP TABLES
CREATE TABLE #temp (id INT, name VARCHAR(50));  -- Session-scoped
CREATE TABLE ##global_temp (id INT);             -- Global temp
SELECT * INTO #backup FROM employees;            -- Create + populate

-- TABLE VARIABLES
DECLARE @results TABLE (id INT, name VARCHAR(50));
INSERT INTO @results SELECT id, name FROM employees;

-- STORED PROCEDURE
CREATE PROCEDURE GetEmployees @dept VARCHAR(50)
AS
BEGIN
    SELECT * FROM employees WHERE department = @dept;
END;
EXEC GetEmployees @dept = 'Engineering';

-- TRY/CATCH
BEGIN TRY
    INSERT INTO users (id, name) VALUES (1, 'Alice');
END TRY
BEGIN CATCH
    SELECT ERROR_NUMBER(), ERROR_MESSAGE(), ERROR_SEVERITY();
END CATCH;

-- EXECUTION PLAN
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
-- Include Actual Execution Plan (Ctrl+M in SSMS)
```

### Key DMVs (Dynamic Management Views)

```sql
sys.dm_exec_requests          -- Currently executing queries
sys.dm_exec_sessions          -- Active sessions
sys.dm_exec_query_stats       -- Query performance stats
sys.dm_os_wait_stats          -- Wait statistics
sys.dm_db_index_usage_stats   -- Index usage
sys.dm_db_missing_index_details -- Missing index suggestions
sys.dm_tran_locks             -- Current locks
```

---

# 📄 SHEET 6: Oracle Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│                  ORACLE CHEAT SHEET                                 │
├─────────────────────────────────────────────────────────────────────┤
│ Default Port: 1521          │ Listener: lsnrctl                    │
│ Config: init.ora/spfile     │ Data Dict: DBA_/ALL_/USER_ views    │
│ SGA + PGA memory model     │ RAC for clustering                   │
└─────────────────────────────┴──────────────────────────────────────┘
```

### Oracle-Specific Syntax

```sql
-- DUAL (dummy table)
SELECT SYSDATE FROM DUAL;
SELECT 1 + 1 FROM DUAL;

-- ROWNUM (legacy pagination)
SELECT * FROM (SELECT t.*, ROWNUM rn FROM employees t WHERE ROWNUM <= 30)
WHERE rn > 20;

-- FETCH FIRST (12c+)
SELECT * FROM employees ORDER BY salary DESC
FETCH FIRST 10 ROWS ONLY;
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- NVL / NVL2
SELECT NVL(phone, 'N/A') FROM users;            -- If null, return default
SELECT NVL2(phone, 'Has phone', 'No phone');     -- If not null → val1, else val2

-- DECODE (old-style CASE)
SELECT DECODE(status, 'A', 'Active', 'I', 'Inactive', 'Unknown') FROM users;

-- SEQUENCES
CREATE SEQUENCE emp_seq START WITH 1 INCREMENT BY 1;
INSERT INTO employees (id, name) VALUES (emp_seq.NEXTVAL, 'Alice');
SELECT emp_seq.CURRVAL FROM DUAL;

-- SYNONYMS
CREATE SYNONYM emps FOR hr.employees;

-- CONNECT BY (hierarchical)
SELECT LEVEL, emp_name, CONNECT_BY_ROOT emp_name AS boss,
       SYS_CONNECT_BY_PATH(emp_name, '/') AS path
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR emp_id = manager_id;

-- LISTAGG (string aggregation)
SELECT dept, LISTAGG(name, ', ') WITHIN GROUP (ORDER BY name)
FROM employees GROUP BY dept;

-- MERGE
MERGE INTO target t USING source s ON (t.id = s.id)
WHEN MATCHED THEN UPDATE SET t.name = s.name
WHEN NOT MATCHED THEN INSERT (id, name) VALUES (s.id, s.name);

-- FLASHBACK
SELECT * FROM employees AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' HOUR);

-- EXPLAIN PLAN
EXPLAIN PLAN FOR SELECT * FROM employees WHERE dept = 'IT';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Key Dictionary Views

```sql
DBA_TABLES              -- All tables (DBA privilege)
ALL_TABLES              -- Tables accessible to current user
USER_TABLES             -- Current user's tables
DBA_TAB_COLUMNS         -- Column info
DBA_INDEXES             -- Index info
DBA_CONSTRAINTS         -- Constraints
V$SESSION               -- Active sessions
V$SQL                   -- SQL in shared pool
V$LOCK                  -- Current locks
V$PARAMETER             -- Instance parameters
```

---

# 📄 SHEET 7: MongoDB Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│                  MONGODB CHEAT SHEET                                │
├─────────────────────────────────────────────────────────────────────┤
│ Default Port: 27017         │ Shell: mongosh                       │
│ Storage: WiredTiger         │ Protocol: Wire Protocol (binary)     │
│ Max Doc Size: 16MB          │ Max Nesting: 100 levels              │
└─────────────────────────────┴──────────────────────────────────────┘
```

### CRUD Operations

```javascript
// CREATE
db.users.insertOne({ name: "Alice", age: 30 })
db.users.insertMany([{ name: "Bob" }, { name: "Charlie" }])

// READ
db.users.findOne({ name: "Alice" })
db.users.find({ age: { $gte: 25 } })
db.users.find({ age: { $gte: 25 } }, { name: 1, _id: 0 })   // Projection
db.users.find().sort({ age: -1 }).limit(10).skip(20)
db.users.countDocuments({ status: "active" })

// UPDATE
db.users.updateOne({ _id: id }, { $set: { age: 31 } })
db.users.updateMany({ dept: "IT" }, { $inc: { salary: 5000 } })
db.users.replaceOne({ _id: id }, { name: "Alice", age: 31 })
db.users.findOneAndUpdate({ _id: id }, { $set: { age: 31 } }, { returnDocument: "after" })

// DELETE
db.users.deleteOne({ _id: id })
db.users.deleteMany({ status: "inactive" })
db.users.drop()                         // Drop entire collection
```

### Query Operators

```javascript
// Comparison
$eq, $ne, $gt, $gte, $lt, $lte, $in, $nin

// Logical
$and, $or, $not, $nor

// Element
$exists, $type

// Array
$all, $size, $elemMatch

// String
$regex, $text (needs text index)

// Example:
db.users.find({
    $and: [
        { age: { $gte: 25, $lte: 40 } },
        { tags: { $in: ["developer", "engineer"] } },
        { email: { $exists: true } }
    ]
})
```

### Update Operators

```javascript
$set       // Set field value
$unset     // Remove field
$inc       // Increment
$mul       // Multiply
$rename    // Rename field
$min/$max  // Set if less/greater than current
$push      // Add to array
$pull      // Remove from array
$addToSet  // Add to array (unique only)
$pop       // Remove first (-1) or last (1) from array
$each      // With $push/$addToSet for multiple values
$slice     // With $push to limit array size
$sort      // With $push to sort array
```

### Aggregation Pipeline

```javascript
db.collection.aggregate([
    { $match:    { status: "active" } },           // WHERE
    { $group:    { _id: "$dept", total: { $sum: "$sal" } } }, // GROUP BY
    { $sort:     { total: -1 } },                  // ORDER BY
    { $limit:    10 },                             // LIMIT
    { $skip:     5 },                              // OFFSET
    { $project:  { name: 1, _id: 0 } },           // SELECT
    { $unwind:   "$tags" },                        // Flatten array
    { $lookup:   { from: "orders", localField: "_id",  // JOIN
                   foreignField: "user_id", as: "orders" } },
    { $addFields:{ fullName: { $concat: ["$first", " ", "$last"] } } },
    { $count:    "total" },                        // COUNT
    { $facet:    { results: [...], count: [...] } }, // Multiple pipelines
    { $merge:    { into: "output_collection" } }   // Write results
])

// Aggregation Operators:
$sum, $avg, $min, $max, $first, $last, $push, $addToSet
$cond, $ifNull, $switch
$concat, $substr, $toLower, $toUpper
$dateToString, $year, $month, $dayOfMonth
$arrayElemAt, $filter, $map, $reduce
```

### Indexes

```javascript
db.coll.createIndex({ field: 1 })                  // Ascending
db.coll.createIndex({ field: -1 })                 // Descending
db.coll.createIndex({ a: 1, b: -1 })              // Compound
db.coll.createIndex({ field: "text" })             // Text search
db.coll.createIndex({ location: "2dsphere" })      // Geospatial
db.coll.createIndex({ field: "hashed" })           // Hashed
db.coll.createIndex({ ts: 1 }, { expireAfterSeconds: 3600 })  // TTL
db.coll.createIndex({ email: 1 }, { unique: true })           // Unique
db.coll.createIndex({ phone: 1 }, { sparse: true })           // Sparse
db.coll.createIndex({ "$**": 1 })                  // Wildcard

db.coll.getIndexes()                               // List indexes
db.coll.dropIndex("index_name")                    // Drop index
db.coll.find({}).explain("executionStats")         // Query plan
```

### Admin

```javascript
show dbs                    // List databases
use mydb                    // Switch database
show collections            // List collections
db.stats()                  // Database stats
db.coll.stats()             // Collection stats
db.currentOp()              // Running operations
db.serverStatus()           // Server info
rs.status()                 // Replica set status
sh.status()                 // Sharding status
db.setProfilingLevel(1, { slowms: 100 })  // Profile slow queries
```

---

# 📄 SHEET 8: Redis Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│                  REDIS CHEAT SHEET                                  │
├─────────────────────────────────────────────────────────────────────┤
│ Default Port: 6379          │ Config: redis.conf                   │
│ Single-threaded (I/O)       │ In-memory data structure store       │
│ Persistence: RDB + AOF     │ Max key/value: 512MB                 │
└─────────────────────────────┴──────────────────────────────────────┘
```

### Data Types & Commands

```redis
# ── STRINGS ──────────────────────────────────────────────
SET key value                      # Set string value
SET key value EX 60                # Set with TTL (60 seconds)
SET key value NX                   # Set only if NOT exists (lock!)
GET key                            # Get value
MSET k1 v1 k2 v2                  # Set multiple
MGET k1 k2                        # Get multiple
INCR counter                      # Increment by 1
INCRBY counter 5                   # Increment by 5
DECR counter                      # Decrement by 1
APPEND key " world"               # Append to string
STRLEN key                        # String length
SETNX key value                   # Set if Not eXists (atomic!)

# ── HASHES (objects) ─────────────────────────────────────
HSET user:1 name "Alice" age 30   # Set hash fields
HGET user:1 name                  # Get one field
HGETALL user:1                    # Get all fields + values
HMGET user:1 name age             # Get multiple fields
HINCRBY user:1 age 1              # Increment hash field
HDEL user:1 age                   # Delete hash field
HEXISTS user:1 name               # Check field exists
HKEYS user:1                      # Get all field names
HVALS user:1                      # Get all values

# ── LISTS (ordered, duplicates OK) ───────────────────────
LPUSH queue task1 task2            # Push to left (head)
RPUSH queue task3                  # Push to right (tail)
LPOP queue                        # Pop from left
RPOP queue                        # Pop from right
BLPOP queue 30                    # Blocking pop (30s timeout)
LRANGE queue 0 -1                 # Get all elements
LLEN queue                        # List length
LINDEX queue 0                    # Get element at index
LREM queue 2 "value"              # Remove 2 occurrences of "value"

# ── SETS (unique, unordered) ─────────────────────────────
SADD myset a b c                   # Add members
SMEMBERS myset                    # Get all members
SISMEMBER myset "a"               # Check membership
SCARD myset                       # Set cardinality (size)
SREM myset "a"                    # Remove member
SINTER set1 set2                  # Intersection
SUNION set1 set2                  # Union
SDIFF set1 set2                   # Difference
SRANDMEMBER myset 3               # Get 3 random members

# ── SORTED SETS (unique, scored, ordered) ────────────────
ZADD leaderboard 100 "Alice"      # Add with score
ZADD leaderboard 95 "Bob" 110 "Charlie"
ZRANK leaderboard "Alice"         # Rank (0-based, ascending)
ZREVRANK leaderboard "Alice"      # Reverse rank (descending)
ZRANGE leaderboard 0 -1 WITHSCORES  # Get all (ascending)
ZREVRANGE leaderboard 0 2         # Top 3 (descending)
ZRANGEBYSCORE lb 90 100           # Members with score 90-100
ZSCORE leaderboard "Alice"        # Get score
ZINCRBY leaderboard 5 "Alice"     # Increment score
ZCARD leaderboard                 # Count members
ZREM leaderboard "Bob"            # Remove member

# ── STREAMS (append-only log, like Kafka) ────────────────
XADD mystream * field1 value1     # Add entry (* = auto ID)
XLEN mystream                     # Stream length
XRANGE mystream - +               # Read all entries
XREAD COUNT 10 STREAMS mystream 0 # Read from beginning
XREAD BLOCK 5000 STREAMS mystream $ # Block for new entries
```

### Key Management

```redis
DEL key                           # Delete key
EXISTS key                        # Check if exists (returns 0/1)
EXPIRE key 60                     # Set TTL (60 seconds)
TTL key                           # Get remaining TTL (-1=no expiry, -2=gone)
PERSIST key                       # Remove TTL (make permanent)
TYPE key                          # Get data type
KEYS pattern*                     # Find keys (⚠️ blocks! Use SCAN)
SCAN 0 MATCH user:* COUNT 100    # Iterate keys safely
RENAME key newkey                 # Rename key
DBSIZE                            # Total key count
FLUSHDB                           # Delete all keys in current DB ⚠️
FLUSHALL                          # Delete ALL keys in ALL DBs ⚠️⚠️
```

### Common Patterns

```redis
# CACHING
SET cache:user:1 "{json}" EX 3600       # Cache with 1h TTL

# DISTRIBUTED LOCK
SET lock:resource owner NX EX 30        # Acquire lock (30s timeout)
# ... do work ...
DEL lock:resource                        # Release lock

# RATE LIMITING (sliding window)
MULTI
ZADD ratelimit:user:1 timestamp timestamp
ZREMRANGEBYSCORE ratelimit:user:1 0 (now - window)
ZCARD ratelimit:user:1
EXPIRE ratelimit:user:1 window
EXEC

# SESSION STORE
HSET session:abc123 user_id 1 name "Alice" role "admin"
EXPIRE session:abc123 1800               # 30 min TTL

# PUB/SUB
SUBSCRIBE channel                        # Subscribe
PUBLISH channel "message"                # Publish
```

### Persistence

```
RDB (Snapshotting):
  save 900 1          # Save if 1+ keys changed in 900s
  save 300 10         # Save if 10+ keys changed in 300s
  save 60 10000       # Save if 10000+ keys changed in 60s
  ✅ Compact, fast restart
  ❌ Data loss between snapshots

AOF (Append-Only File):
  appendonly yes
  appendfsync everysec  # Sync every second (good balance)
  appendfsync always    # Sync every write (slow but safest)
  appendfsync no        # OS decides (fastest, risky)
  ✅ Minimal data loss
  ❌ Larger files, slower restart

BEST PRACTICE: Use BOTH RDB + AOF
```

---

# 📄 SHEET 9: Database Concepts Quick Reference

---

## ACID Properties

```
┌──────────────────────────────────────────────────────────────┐
│  A — Atomicity:     All or nothing. If one part fails,      │
│                     the entire transaction is rolled back.   │
│  C — Consistency:   DB moves from one valid state to another.│
│                     Constraints are never violated.          │
│  I — Isolation:     Concurrent transactions don't interfere. │
│                     Each sees a consistent snapshot.         │
│  D — Durability:    Once committed, data survives crashes.   │
│                     Written to disk / WAL before confirming. │
└──────────────────────────────────────────────────────────────┘
```

## BASE Properties (NoSQL)

```
┌──────────────────────────────────────────────────────────────┐
│  BA — Basically Available: System guarantees availability    │
│  S  — Soft state:         State may change without input     │
│                           (eventual consistency)             │
│  E  — Eventually Consistent: Given time, all replicas agree │
└──────────────────────────────────────────────────────────────┘
```

## CAP Theorem

```
  You can only guarantee 2 of 3:
  
       Consistency
          /\
         /  \
        /    \
       / CP   \  CA
      /________\
  Partition    Availability
  Tolerance

  CP: MongoDB, Redis, HBase        (sacrifice availability during partition)
  AP: Cassandra, DynamoDB, CouchDB  (sacrifice consistency during partition)
  CA: Traditional RDBMS (PostgreSQL, MySQL) (no partition tolerance — single node)
  
  ⚠️ In distributed systems, P is mandatory → real choice is CP vs AP
```

## Isolation Levels

```
┌─────────────────────┬────────────┬──────────────┬────────────────┐
│ Level               │ Dirty Read │ Non-Repeatable│ Phantom Read  │
├─────────────────────┼────────────┼──────────────┼────────────────┤
│ Read Uncommitted    │ ✅ Possible │ ✅ Possible  │ ✅ Possible    │
│ Read Committed      │ ❌ No      │ ✅ Possible  │ ✅ Possible    │
│ Repeatable Read     │ ❌ No      │ ❌ No        │ ✅ Possible*   │
│ Serializable        │ ❌ No      │ ❌ No        │ ❌ No          │
└─────────────────────┴────────────┴──────────────┴────────────────┘
* PostgreSQL's RR prevents phantoms too (MVCC implementation)

Defaults: PostgreSQL=Read Committed, MySQL=Repeatable Read, 
          SQL Server=Read Committed, Oracle=Read Committed
```

## Normalization

```
1NF: Atomic values (no arrays, no repeating groups)
2NF: 1NF + no partial dependency (all non-key columns depend on FULL PK)
3NF: 2NF + no transitive dependency (non-key doesn't depend on non-key)
BCNF: Every determinant is a candidate key
4NF: No multi-valued dependencies
5NF: No join dependencies

RULE OF THUMB:
  OLTP → Normalize (3NF minimum) → Less redundancy, better integrity
  OLAP → Denormalize (Star/Snowflake) → Faster queries, simpler joins
```

## Index Types

```
┌──────────────────┬──────────────────────────────────────────────┐
│ Type             │ Best For                                     │
├──────────────────┼──────────────────────────────────────────────┤
│ B-Tree           │ Range queries, equality, sorting (DEFAULT)   │
│ Hash             │ Exact equality lookups ONLY                  │
│ GIN              │ Full-text search, JSONB, arrays (PostgreSQL) │
│ GiST             │ Geospatial, range types (PostgreSQL)         │
│ BRIN             │ Very large tables with natural ordering      │
│ Bitmap           │ Low-cardinality columns (Oracle, PostgreSQL) │
│ Clustered        │ One per table — reorders physical data       │
│ Non-Clustered    │ Separate structure pointing to data          │
│ Covering         │ Includes all query columns → no table lookup │
│ Partial/Filtered │ Index subset of rows (WHERE condition)       │
│ Composite        │ Multiple columns (order matters!)            │
└──────────────────┴──────────────────────────────────────────────┘
```

---

# 📄 SHEET 10: SQL vs NoSQL Decision Matrix

```
┌──────────────────────┬──────────────────┬────────────────────┐
│ Criteria             │ SQL (RDBMS)      │ NoSQL              │
├──────────────────────┼──────────────────┼────────────────────┤
│ Schema               │ Fixed, predefined│ Flexible, dynamic  │
│ Data relationships   │ Complex JOINs    │ Embedded / denorm  │
│ Transactions         │ Full ACID        │ Eventually consist.│
│ Scaling              │ Vertical (up)    │ Horizontal (out)   │
│ Query language       │ Standardized SQL │ Vendor-specific    │
│ Data model           │ Tables & rows    │ Document/Key-Value/│
│                      │                  │ Column/Graph       │
│ Best for             │ Structured data  │ Unstructured/semi  │
│ Consistency          │ Strong           │ Configurable       │
│ Community/Tooling    │ Mature, vast     │ Growing fast       │
└──────────────────────┴──────────────────┴────────────────────┘

WHEN TO USE SQL:
  ✅ Complex relationships between entities
  ✅ ACID transactions are critical (banking, inventory)
  ✅ Data structure is well-defined and stable
  ✅ Complex queries with JOINs, aggregations
  ✅ Reporting and analytics (OLAP)

WHEN TO USE NoSQL:
  ✅ Rapidly changing schema / agile development
  ✅ Massive scale (millions of writes/sec)
  ✅ Geographically distributed data
  ✅ Real-time big data (IoT sensors, clickstreams)
  ✅ Caching and session management (Redis)
  ✅ Content management / catalogs (MongoDB)
  ✅ Social networks / recommendations (Neo4j/Graph)
```

### Which Database to Pick?

```
┌─────────────────────────────────────────────────────────────────────┐
│                  DATABASE SELECTION GUIDE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  "I need transactions + JOINs"        → PostgreSQL / SQL Server    │
│  "I need simple relational + OSS"     → MySQL / MariaDB            │
│  "I need enterprise + Oracle shops"   → Oracle                     │
│  "I need flexible schema + JSON"      → MongoDB                    │
│  "I need blazing-fast cache"          → Redis / Memcached          │
│  "I need time-series data"            → TimescaleDB / InfluxDB     │
│  "I need graph relationships"         → Neo4j / Amazon Neptune     │
│  "I need wide-column at scale"        → Cassandra / ScyllaDB       │
│  "I need full-text search"            → Elasticsearch / OpenSearch │
│  "I need embedded (no server)"        → SQLite                     │
│  "I need event streaming"             → Kafka (+ Redis Streams)    │
│  "I need vector/AI embeddings"        → Pinecone / pgvector / Milvus│
│                                                                     │
│  MOST COMMON COMBO (Polyglot):                                     │
│  PostgreSQL (primary) + Redis (cache) + Elasticsearch (search)     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔑 Interview Quick-Fire Answers

```
Q: "ACID?"          → Atomicity, Consistency, Isolation, Durability
Q: "CAP?"           → Consistency, Availability, Partition Tolerance (pick 2)
Q: "MVCC?"          → Multi-Version Concurrency Control — readers don't block writers
Q: "WAL?"           → Write-Ahead Log — write to log before data file (crash recovery)
Q: "B-Tree?"        → Balanced tree index — O(log n) search, range-friendly
Q: "Sharding?"      → Horizontal partitioning data across multiple servers
Q: "Replication?"   → Copying data to multiple nodes for HA/reads
Q: "Master-Slave?"  → One writer (primary), many readers (replicas)
Q: "Connection pool?"→ Pre-opened connections reused across requests (pgbouncer, HikariCP)
Q: "N+1 Problem?"   → 1 query for parent + N queries for children → use JOIN or eager load
Q: "Denormalization?"→ Adding redundancy for read performance (trade write complexity)
Q: "Clustered idx?"  → Index that determines physical row order (one per table)
Q: "Covering idx?"   → Index containing all columns needed → no table lookup
Q: "Deadlock?"       → Two transactions waiting for each other's locks → one killed
Q: "Optimistic lock?" → Check version/timestamp at commit time (no locks held)
Q: "Pessimistic lock?"→ Acquire lock before reading (SELECT FOR UPDATE)
```

---

## 🔗 Navigation

| Chapter | Link |
|---------|------|
| SQL Interview Questions | [Chapter 8.1](./01-SQL-Interview-Questions.md) |
| DB Theory Questions | [Chapter 8.2](./02-DB-Theory-Questions.md) |
| MongoDB Interview | [Chapter 8.3](./03-MongoDB-Interview.md) |
| SQL Practice Problems | [Chapter 8.4](./04-SQL-Practice-Problems.md) |
| Back to Index | [INDEX](../INDEX.md) |

---

> **"A great engineer doesn't memorize everything — they know where to find it in under 10 seconds."** 📋
