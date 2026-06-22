# 🔶 Chapter 2B.3 — Oracle SQL — Dialect Specifics

> **Level:** 🟡 Intermediate
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 2A (SQL Language Mastery), Chapter 2B.2 (Oracle Installation)

> **"SQL is the language. But every database speaks its own dialect. Oracle's dialect is rich, powerful, and sometimes quirky. Master it, and you'll write SQL that others can only dream of."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand Oracle-specific SQL syntax that doesn't exist in other databases
- Use **ROWNUM**, **ROWID**, and **ROW_NUMBER()** correctly (and know the difference)
- Write **hierarchical queries** with `CONNECT BY` (Oracle's superpower)
- Master Oracle's unique functions: `DECODE`, `NVL`, `NVL2`, `NULLIF`, `COALESCE`
- Work with **Sequences**, **Synonyms**, **Dual table**, and Oracle-specific data types
- Convert between Oracle-specific and ANSI-standard SQL

---

## 1. The DUAL Table — Oracle's Magic Helper

In most databases, you can do `SELECT 1+1`. In Oracle, every `SELECT` needs a `FROM` clause. Enter `DUAL`:

```sql
-- DUAL is a special 1-row, 1-column table that always exists

-- Basic arithmetic
SELECT 1 + 1 FROM dual;           -- 2
SELECT 7 * 6 FROM dual;           -- 42

-- Current date/time
SELECT SYSDATE FROM dual;         -- 02-JUN-26
SELECT SYSTIMESTAMP FROM dual;    -- 02-JUN-26 09.30.15.123456 AM +05:30
SELECT CURRENT_TIMESTAMP FROM dual;

-- String operations
SELECT 'Hello' || ' ' || 'World' FROM dual;   -- Hello World
-- Note: Oracle uses || for concatenation (not + like SQL Server)

-- User/session info
SELECT USER FROM dual;            -- STUDENT
SELECT SYS_CONTEXT('USERENV', 'SESSION_USER') FROM dual;
SELECT SYS_CONTEXT('USERENV', 'DB_NAME') FROM dual;

-- Sequences
SELECT my_seq.NEXTVAL FROM dual;
SELECT my_seq.CURRVAL FROM dual;

-- Type conversion
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS') FROM dual;
SELECT TO_NUMBER('42') FROM dual;
SELECT TO_DATE('2026-06-02', 'YYYY-MM-DD') FROM dual;
```

> 💡 **Fun Fact**: Starting from Oracle 23c, you can finally do `SELECT 1+1;` without `FROM dual`! But for versions up to 21c (which most companies use), you still need `DUAL`.

---

## 2. ROWNUM vs ROWID vs ROW_NUMBER() — The Holy Trinity

This is one of the most confusing areas in Oracle. Let's clear it up forever.

### 2.1 ROWNUM — The Row Counter (Pseudo-column)

```
ROWNUM is assigned BEFORE sorting, BEFORE grouping.
It's assigned as rows are FETCHED from the query result.

┌─────────────────────────────────────────────────────────────┐
│  Query Processing Order:                                    │
│                                                              │
│  1. FROM → 2. WHERE → 3. SELECT → ROWNUM assigned here!    │
│  4. ORDER BY (happens AFTER ROWNUM!)                         │
│                                                              │
│  This means ORDER BY + ROWNUM don't mix well directly!      │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- ✅ Get first 5 rows (works correctly)
SELECT * FROM employees WHERE ROWNUM <= 5;

-- ❌ WRONG: Get top 5 highest salaries (ROWNUM assigned BEFORE ORDER BY!)
SELECT * FROM employees 
WHERE ROWNUM <= 5 
ORDER BY salary DESC;
-- This gets 5 random rows, THEN sorts them. NOT what you want!

-- ✅ CORRECT: Use a subquery to sort first, then apply ROWNUM
SELECT * FROM (
    SELECT * FROM employees ORDER BY salary DESC
) WHERE ROWNUM <= 5;

-- ❌ WRONG: ROWNUM > 5 never returns rows!
SELECT * FROM employees WHERE ROWNUM > 5;
-- Why? ROWNUM starts at 1. The first row is ROWNUM=1, fails > 5 check,
-- so it's excluded. The NEXT row becomes ROWNUM=1, fails again. Forever.

-- ✅ CORRECT: Pagination using nested subquery
SELECT * FROM (
    SELECT e.*, ROWNUM AS rn
    FROM (SELECT * FROM employees ORDER BY salary DESC) e
    WHERE ROWNUM <= 20   -- Upper bound
) WHERE rn > 10;         -- Lower bound
-- Returns rows 11-20 (page 2 with 10 rows per page)
```

### 2.2 ROWID — The Physical Address

```
ROWID is the PHYSICAL address of a row on disk. It's the fastest way to access a row.

Format: OOOOOOFFFBBBBBBRRR
  OOOOOO = Data object number
  FFF    = Relative file number  
  BBBBBB = Block number
  RRR    = Row number within block
```

```sql
-- See ROWIDs
SELECT ROWID, employee_id, first_name FROM employees WHERE ROWNUM <= 3;
-- ROWID                 EMPLOYEE_ID  FIRST_NAME
-- AAAECQAAEAAAADnAAA    100          Steven
-- AAAECQAAEAAAADnAAB    101          Neena
-- AAAECQAAEAAAADnAAC    102          Lex

-- ROWID is the FASTEST way to access a specific row
SELECT * FROM employees WHERE ROWID = 'AAAECQAAEAAAADnAAA';
-- This is even faster than primary key lookup!

-- Delete duplicate rows (classic Oracle interview question!)
DELETE FROM employees
WHERE ROWID NOT IN (
    SELECT MIN(ROWID) 
    FROM employees 
    GROUP BY employee_id, first_name, last_name
);
-- Keeps one copy, deletes duplicates based on ROWID
```

### 2.3 ROW_NUMBER() — The Modern Way (ANSI Standard)

```sql
-- ROW_NUMBER() is a window function — assigned AFTER sorting ✅
-- This is the PREFERRED method for pagination in modern Oracle

-- Top 5 highest-paid employees
SELECT * FROM (
    SELECT employee_id, first_name, salary,
           ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
    FROM employees
) WHERE rn <= 5;

-- Pagination (Page 2, 10 rows per page) — Clean and reliable
SELECT * FROM (
    SELECT employee_id, first_name, salary,
           ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
    FROM employees
) WHERE rn BETWEEN 11 AND 20;

-- Oracle 12c+ : OFFSET-FETCH (finally, like SQL Server and PostgreSQL!)
SELECT employee_id, first_name, salary
FROM employees
ORDER BY salary DESC
OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY;
-- ⭐ Use this in Oracle 12c and later — cleanest syntax!
```

### Comparison Table

| Feature | ROWNUM | ROWID | ROW_NUMBER() |
|---------|--------|-------|-------------|
| **What is it?** | Pseudo-column (counter) | Physical row address | Window function |
| **Assigned when?** | During query execution (before ORDER BY) | When row is stored | After ORDER BY (in window) |
| **Persistent?** | No (changes per query) | Yes (until row moves) | No (computed per query) |
| **Use for pagination?** | Yes (with subquery tricks) | No | ✅ Yes (preferred) |
| **Use for "TOP N"?** | Yes | No | ✅ Yes (preferred) |
| **ANSI Standard?** | ❌ Oracle only | ❌ Oracle only | ✅ Standard SQL |
| **Performance** | Fast | Fastest row access | Slightly more overhead |

---

## 3. Oracle Data Types — What's Different

### Oracle-Specific Data Types

```sql
-- ═══════════════════════════════════════════════════
-- STRING TYPES
-- ═══════════════════════════════════════════════════

-- VARCHAR2 (Oracle's version of VARCHAR — use this, not VARCHAR!)
CREATE TABLE demo (
    name VARCHAR2(100),          -- Variable-length, max 4000 bytes (SQL) / 32767 bytes (PL/SQL)
    code CHAR(5),                -- Fixed-length, padded with spaces
    description CLOB,            -- Character Large Object — up to 128 TB
    notes NVARCHAR2(200),        -- Unicode variable-length
    nchar_col NCHAR(10)          -- Unicode fixed-length
);

-- ⚠️ VARCHAR vs VARCHAR2: In Oracle, ALWAYS use VARCHAR2
-- VARCHAR is reserved for future use and may behave differently!

-- ═══════════════════════════════════════════════════
-- NUMBER TYPES
-- ═══════════════════════════════════════════════════

CREATE TABLE numbers_demo (
    id NUMBER,                   -- Up to 38 digits precision
    price NUMBER(10,2),          -- 10 digits total, 2 decimal → 99999999.99
    quantity NUMBER(8,0),        -- Integer (8 digits, no decimals)
    rate BINARY_FLOAT,           -- 32-bit IEEE floating point
    precise_rate BINARY_DOUBLE,  -- 64-bit IEEE floating point
    flag NUMBER(1)               -- Poor man's boolean (0 or 1)
);
-- Note: Oracle has NO BOOLEAN type in SQL! (Only in PL/SQL)
-- Use NUMBER(1) with CHECK(flag IN (0,1)) or CHAR(1) 'Y'/'N'

-- ═══════════════════════════════════════════════════
-- DATE & TIME TYPES
-- ═══════════════════════════════════════════════════

CREATE TABLE dates_demo (
    created_date DATE,                         -- Date + Time (to seconds)
    precise_time TIMESTAMP,                    -- Date + Time (fractional seconds)
    with_tz TIMESTAMP WITH TIME ZONE,          -- With timezone info
    local_tz TIMESTAMP WITH LOCAL TIME ZONE,   -- Converts to session timezone
    duration INTERVAL YEAR TO MONTH,           -- e.g., '3-6' (3 years 6 months)
    exact_dur INTERVAL DAY TO SECOND           -- e.g., '5 04:30:00' (5d 4h 30m)
);

-- ⚠️ Oracle DATE includes TIME! This surprises people from MySQL/SQL Server
-- Oracle DATE = DATE + TIME (to seconds)
-- If you need fractional seconds → use TIMESTAMP

-- ═══════════════════════════════════════════════════
-- LARGE OBJECT TYPES
-- ═══════════════════════════════════════════════════

CREATE TABLE lob_demo (
    text_data CLOB,         -- Character LOB (text documents, XML, JSON)
    binary_data BLOB,       -- Binary LOB (images, PDFs, videos)
    file_pointer BFILE      -- Pointer to external OS file (read-only)
);
```

### Oracle vs Other Databases — Data Type Mapping

| Concept | Oracle | SQL Server | MySQL | PostgreSQL |
|---------|--------|------------|-------|------------|
| Variable string | `VARCHAR2(n)` | `VARCHAR(n)` | `VARCHAR(n)` | `VARCHAR(n)` |
| Integer | `NUMBER(10)` | `INT` | `INT` | `INTEGER` |
| Decimal | `NUMBER(10,2)` | `DECIMAL(10,2)` | `DECIMAL(10,2)` | `NUMERIC(10,2)` |
| Date only | `DATE` (includes time!) | `DATE` | `DATE` | `DATE` |
| Date+Time | `DATE` or `TIMESTAMP` | `DATETIME` | `DATETIME` | `TIMESTAMP` |
| Boolean | `NUMBER(1)` ❌ | `BIT` | `BOOLEAN` / `TINYINT(1)` | `BOOLEAN` |
| Auto-increment | `SEQUENCE` + trigger / `IDENTITY` (12c+) | `IDENTITY` | `AUTO_INCREMENT` | `SERIAL` / `IDENTITY` |
| Text LOB | `CLOB` | `NVARCHAR(MAX)` | `LONGTEXT` | `TEXT` |
| Binary LOB | `BLOB` | `VARBINARY(MAX)` | `LONGBLOB` | `BYTEA` |

---

## 4. Oracle-Specific Functions — The Power Tools

### 4.1 NVL, NVL2, COALESCE, NULLIF — Null Handling

```sql
-- ═══════════════════════════════════════════════════
-- NVL(expr, replacement) — If expr is NULL, return replacement
-- ═══════════════════════════════════════════════════
SELECT first_name, 
       NVL(commission_pct, 0) AS commission  -- NULL → 0
FROM employees;

SELECT NVL(NULL, 'default') FROM dual;    -- 'default'
SELECT NVL('hello', 'default') FROM dual; -- 'hello'


-- ═══════════════════════════════════════════════════
-- NVL2(expr, if_not_null, if_null) — Oracle-specific!
-- ═══════════════════════════════════════════════════
SELECT first_name,
       NVL2(commission_pct, 'Has Commission', 'No Commission') AS status
FROM employees;

-- NVL2 is like a shorthand IF-ELSE for NULL checking
SELECT NVL2(NULL, 'NOT NULL', 'IS NULL') FROM dual;  -- 'IS NULL'
SELECT NVL2(42, 'NOT NULL', 'IS NULL') FROM dual;    -- 'NOT NULL'


-- ═══════════════════════════════════════════════════
-- COALESCE(expr1, expr2, expr3, ...) — First non-NULL (ANSI standard)
-- ═══════════════════════════════════════════════════
SELECT COALESCE(commission_pct, bonus, 0) AS pay_extra
FROM employees;
-- Returns commission_pct if not null, else bonus if not null, else 0

SELECT COALESCE(NULL, NULL, NULL, 'found!') FROM dual; -- 'found!'


-- ═══════════════════════════════════════════════════
-- NULLIF(expr1, expr2) — Returns NULL if expr1 = expr2
-- ═══════════════════════════════════════════════════
SELECT NULLIF(10, 10) FROM dual;  -- NULL (they're equal)
SELECT NULLIF(10, 20) FROM dual;  -- 10 (not equal)

-- Useful to avoid divide-by-zero:
SELECT salary / NULLIF(commission_pct, 0) FROM employees;
-- If commission_pct = 0, NULLIF returns NULL, and salary/NULL = NULL (no error!)
```

### 4.2 DECODE — Oracle's Inline CASE (Before CASE Existed)

```sql
-- DECODE(expression, search1, result1, search2, result2, ..., default)
-- It's like a compact CASE statement

-- Simple mapping
SELECT employee_id, first_name,
       DECODE(department_id,
              10, 'Administration',
              20, 'Marketing',
              30, 'Purchasing',
              40, 'Human Resources',
              50, 'Shipping',
              'Other Department') AS dept_name
FROM employees;

-- Equivalent ANSI CASE (preferred for new code):
SELECT employee_id, first_name,
       CASE department_id
           WHEN 10 THEN 'Administration'
           WHEN 20 THEN 'Marketing'
           WHEN 30 THEN 'Purchasing'
           WHEN 40 THEN 'Human Resources'
           WHEN 50 THEN 'Shipping'
           ELSE 'Other Department'
       END AS dept_name
FROM employees;

-- DECODE handles NULLs (CASE doesn't with = comparison)
SELECT DECODE(NULL, NULL, 'Match!', 'No Match') FROM dual;  -- 'Match!'
-- CASE version: CASE WHEN NULL = NULL → FALSE (NULL = NULL is unknown!)
-- So DECODE actually compares NULLs, while = in CASE doesn't
```

> 💡 **When to Use What**: Use `CASE` for new code (ANSI standard, works everywhere). Use `DECODE` only when maintaining legacy Oracle code. The one edge case: DECODE can compare NULLs, which CASE `WHEN expr = val` cannot.

### 4.3 Date Functions — Oracle Style

```sql
-- ═══════════════════════════════════════════════════
-- SYSDATE & SYSTIMESTAMP
-- ═══════════════════════════════════════════════════
SELECT SYSDATE FROM dual;              -- Database server time
SELECT SYSTIMESTAMP FROM dual;         -- Server time with fractional seconds + timezone
SELECT CURRENT_TIMESTAMP FROM dual;    -- Session timezone time

-- ═══════════════════════════════════════════════════
-- Date Arithmetic (Oracle treats dates as numbers!)
-- ═══════════════════════════════════════════════════
SELECT SYSDATE + 1 FROM dual;         -- Tomorrow
SELECT SYSDATE - 1 FROM dual;         -- Yesterday
SELECT SYSDATE + 7 FROM dual;         -- Next week
SELECT SYSDATE + 1/24 FROM dual;      -- 1 hour from now
SELECT SYSDATE + 1/1440 FROM dual;    -- 1 minute from now
SELECT SYSDATE + 1/86400 FROM dual;   -- 1 second from now

-- Difference between dates = number of DAYS
SELECT SYSDATE - hire_date AS days_employed FROM employees;

-- ═══════════════════════════════════════════════════
-- ADD_MONTHS, MONTHS_BETWEEN
-- ═══════════════════════════════════════════════════
SELECT ADD_MONTHS(SYSDATE, 3) FROM dual;     -- 3 months from now
SELECT ADD_MONTHS(SYSDATE, -6) FROM dual;    -- 6 months ago

SELECT MONTHS_BETWEEN(SYSDATE, hire_date) AS months_employed
FROM employees;

-- ═══════════════════════════════════════════════════
-- NEXT_DAY, LAST_DAY, ROUND, TRUNC
-- ═══════════════════════════════════════════════════
SELECT NEXT_DAY(SYSDATE, 'MONDAY') FROM dual;   -- Next Monday
SELECT LAST_DAY(SYSDATE) FROM dual;              -- Last day of current month

SELECT TRUNC(SYSDATE) FROM dual;                 -- Today at midnight (strips time)
SELECT TRUNC(SYSDATE, 'MM') FROM dual;           -- First day of current month
SELECT TRUNC(SYSDATE, 'YYYY') FROM dual;         -- First day of current year
SELECT TRUNC(SYSDATE, 'Q') FROM dual;            -- First day of current quarter

SELECT ROUND(SYSDATE) FROM dual;                 -- Rounds to nearest day
SELECT ROUND(SYSDATE, 'MM') FROM dual;           -- Rounds to nearest month

-- ═══════════════════════════════════════════════════
-- EXTRACT — Pull parts from a date
-- ═══════════════════════════════════════════════════
SELECT EXTRACT(YEAR FROM SYSDATE) FROM dual;     -- 2026
SELECT EXTRACT(MONTH FROM SYSDATE) FROM dual;    -- 6
SELECT EXTRACT(DAY FROM SYSDATE) FROM dual;      -- 2
SELECT EXTRACT(HOUR FROM SYSTIMESTAMP) FROM dual; -- Current hour
```

### 4.4 TO_CHAR, TO_DATE, TO_NUMBER — Type Conversion

```sql
-- ═══════════════════════════════════════════════════
-- TO_CHAR — Convert to string (dates and numbers)
-- ═══════════════════════════════════════════════════

-- Date formatting
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD') FROM dual;           -- 2026-06-02
SELECT TO_CHAR(SYSDATE, 'DD-Mon-YYYY HH24:MI:SS') FROM dual; -- 02-Jun-2026 14:30:45
SELECT TO_CHAR(SYSDATE, 'Day, Month DD, YYYY') FROM dual;  -- Monday, June 02, 2026
SELECT TO_CHAR(SYSDATE, 'DY') FROM dual;                    -- MON
SELECT TO_CHAR(SYSDATE, 'Q') FROM dual;                     -- 2 (quarter)
SELECT TO_CHAR(SYSDATE, 'WW') FROM dual;                    -- Week of year

-- Date Format Elements
-- YYYY=Year, MM=Month(01-12), MON=Month(Jan), MONTH=Month(January)
-- DD=Day(01-31), DY=Day(Mon), DAY=Day(Monday)
-- HH24=Hour(0-23), HH=Hour(1-12), MI=Minutes, SS=Seconds
-- AM/PM, FF=Fractional seconds, TZR=Timezone region

-- Number formatting
SELECT TO_CHAR(1234567.89, '9,999,999.99') FROM dual;      -- 1,234,567.89
SELECT TO_CHAR(1234567.89, '$9,999,999.99') FROM dual;     -- $1,234,567.89
SELECT TO_CHAR(0.75, '990.00') FROM dual;                   --   0.75
SELECT TO_CHAR(42, '0000') FROM dual;                        -- 0042 (leading zeros)

-- ═══════════════════════════════════════════════════
-- TO_DATE — Convert string to date
-- ═══════════════════════════════════════════════════
SELECT TO_DATE('2026-06-02', 'YYYY-MM-DD') FROM dual;
SELECT TO_DATE('02/06/2026 14:30', 'DD/MM/YYYY HH24:MI') FROM dual;
SELECT TO_DATE('June 02, 2026', 'Month DD, YYYY') FROM dual;

-- ⚠️ COMMON MISTAKE: Comparing dates without TO_DATE
-- ❌ WHERE hire_date = '02-JUN-2026'     (depends on NLS_DATE_FORMAT!)
-- ✅ WHERE hire_date = TO_DATE('2026-06-02', 'YYYY-MM-DD')
-- ✅ WHERE TRUNC(hire_date) = DATE '2026-06-02'  (ANSI date literal)

-- ═══════════════════════════════════════════════════
-- TO_NUMBER — Convert string to number
-- ═══════════════════════════════════════════════════
SELECT TO_NUMBER('1,234.56', '9,999.99') FROM dual;   -- 1234.56
SELECT TO_NUMBER('$99.99', '$99.99') FROM dual;       -- 99.99
```

---

## 5. CONNECT BY — Hierarchical Queries (Oracle's Superpower)

This is something Oracle does that **no other SQL database** does as elegantly. Hierarchical queries traverse tree-structured data.

### The Employee-Manager Hierarchy Example

```sql
-- Sample data: Employee → Manager relationship
-- KING (no manager)
--   ├── JONES
--   │     ├── SCOTT
--   │     │     └── ADAMS
--   │     └── FORD
--   │           └── SMITH
--   ├── BLAKE
--   │     ├── ALLEN
--   │     ├── WARD
--   │     └── MARTIN
--   └── CLARK
--         └── MILLER

-- ═══════════════════════════════════════════════════
-- Basic Hierarchical Query
-- ═══════════════════════════════════════════════════
SELECT employee_id, first_name, manager_id, 
       LEVEL,                              -- Depth in hierarchy (root = 1)
       LPAD(' ', (LEVEL-1)*4) || first_name AS org_tree  -- Visual indentation
FROM employees
START WITH manager_id IS NULL              -- Start from the CEO (no manager)
CONNECT BY PRIOR employee_id = manager_id  -- Child's manager_id = Parent's employee_id
ORDER SIBLINGS BY first_name;              -- Sort within each level

-- Output:
-- LEVEL  ORG_TREE
-- 1      Steven
-- 2          Lex
-- 3              Alexander
-- 4                  Amit
-- 4                  Bruce
-- 2          Neena
-- 3              John
-- 3              Nancy
```

### CONNECT BY Keywords

```sql
-- ═══════════════════════════════════════════════════
-- KEY CONCEPTS
-- ═══════════════════════════════════════════════════

-- LEVEL — Pseudo-column showing depth (root = 1)
SELECT LEVEL, employee_id, first_name
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id = manager_id;

-- SYS_CONNECT_BY_PATH — Full path from root to current node
SELECT employee_id, 
       SYS_CONNECT_BY_PATH(first_name, ' → ') AS path
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id = manager_id;
-- Output: → Steven → Neena → Nancy

-- CONNECT_BY_ROOT — Access root row's column
SELECT employee_id, first_name,
       CONNECT_BY_ROOT first_name AS ceo_name,
       CONNECT_BY_ROOT employee_id AS ceo_id
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id = manager_id;

-- CONNECT_BY_ISLEAF — Is this a leaf node (no children)?
SELECT employee_id, first_name,
       CONNECT_BY_ISLEAF AS is_leaf  -- 1 = leaf (no subordinates)
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id = manager_id;

-- ORDER SIBLINGS BY — Sort within same level (preserving hierarchy)
-- ... ORDER SIBLINGS BY first_name;
```

### Practical Use: Generate a Series of Numbers/Dates

```sql
-- Generate numbers 1 to 100 (Oracle's version of generate_series)
SELECT LEVEL AS n
FROM dual
CONNECT BY LEVEL <= 100;

-- Generate dates for the next 30 days
SELECT TRUNC(SYSDATE) + LEVEL - 1 AS calendar_date,
       TO_CHAR(TRUNC(SYSDATE) + LEVEL - 1, 'DY') AS day_name
FROM dual
CONNECT BY LEVEL <= 30;

-- Generate months for a year
SELECT ADD_MONTHS(TRUNC(SYSDATE, 'YYYY'), LEVEL - 1) AS month_start,
       TO_CHAR(ADD_MONTHS(TRUNC(SYSDATE, 'YYYY'), LEVEL - 1), 'Month YYYY') AS month_name
FROM dual
CONNECT BY LEVEL <= 12;

-- Split comma-separated values into rows
SELECT REGEXP_SUBSTR('Apple,Banana,Cherry,Date', '[^,]+', 1, LEVEL) AS fruit
FROM dual
CONNECT BY REGEXP_SUBSTR('Apple,Banana,Cherry,Date', '[^,]+', 1, LEVEL) IS NOT NULL;
-- Apple
-- Banana
-- Cherry
-- Date
```

> 💡 **Pro Tip**: `CONNECT BY LEVEL <= N` with `DUAL` is Oracle's equivalent of PostgreSQL's `generate_series()` or SQL Server's recursive CTEs for number generation. Use it to create calendar tables, fill gaps, and generate test data.

---

## 6. Sequences — Oracle's Auto-Increment

Unlike MySQL's `AUTO_INCREMENT` or SQL Server's `IDENTITY`, Oracle traditionally uses **Sequences** — independent objects that generate unique numbers.

```sql
-- ═══════════════════════════════════════════════════
-- CREATE SEQUENCE
-- ═══════════════════════════════════════════════════
CREATE SEQUENCE emp_seq
  START WITH 1000        -- First value
  INCREMENT BY 1         -- Step
  MINVALUE 1             -- Minimum value
  MAXVALUE 9999999       -- Maximum value
  CACHE 20               -- Pre-allocate 20 values in memory (performance!)
  NOCYCLE                -- Don't restart after reaching MAXVALUE
  ORDER;                 -- Guarantee order (needed for RAC; NOORDER is faster)

-- ═══════════════════════════════════════════════════
-- USE SEQUENCE
-- ═══════════════════════════════════════════════════

-- Get next value (advances the counter)
SELECT emp_seq.NEXTVAL FROM dual;   -- 1000
SELECT emp_seq.NEXTVAL FROM dual;   -- 1001
SELECT emp_seq.NEXTVAL FROM dual;   -- 1002

-- Get current value (doesn't advance; must call NEXTVAL first in session)
SELECT emp_seq.CURRVAL FROM dual;   -- 1002

-- Use in INSERT
INSERT INTO employees (employee_id, first_name, last_name)
VALUES (emp_seq.NEXTVAL, 'Ritesh', 'Singh');

-- ═══════════════════════════════════════════════════
-- ORACLE 12c+ : DEFAULT with SEQUENCE (finally!)
-- ═══════════════════════════════════════════════════
CREATE TABLE orders (
    order_id NUMBER DEFAULT emp_seq.NEXTVAL PRIMARY KEY,
    order_date DATE DEFAULT SYSDATE,
    customer_id NUMBER NOT NULL
);

-- Insert without specifying order_id — auto-populated!
INSERT INTO orders (customer_id) VALUES (42);

-- ═══════════════════════════════════════════════════
-- ORACLE 12c+ : IDENTITY COLUMNS (like SQL Server!)
-- ═══════════════════════════════════════════════════
CREATE TABLE products (
    product_id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    -- GENERATED ALWAYS  → Cannot manually insert ID
    -- GENERATED BY DEFAULT → Can manually insert, auto-gen if omitted
    -- GENERATED BY DEFAULT ON NULL → Auto-gen when NULL is inserted
    
    product_name VARCHAR2(100) NOT NULL,
    price NUMBER(10,2) NOT NULL
);

INSERT INTO products (product_name, price) VALUES ('Laptop', 999.99);
-- product_id auto-generated: 1

-- Modify sequence
ALTER SEQUENCE emp_seq INCREMENT BY 10;
ALTER SEQUENCE emp_seq CACHE 50;

-- Drop sequence
DROP SEQUENCE emp_seq;
```

### Sequence vs Identity — When to Use What

| Feature | Sequence | Identity Column (12c+) |
|---------|----------|----------------------|
| **Shared across tables?** | ✅ Yes (one seq, many tables) | ❌ No (tied to one table) |
| **Need separate DDL?** | Yes (CREATE SEQUENCE) | No (part of CREATE TABLE) |
| **Control over values?** | Full (START, INCREMENT, CACHE) | Limited |
| **Oracle version** | All versions | 12c+ only |
| **Industry standard** | Oracle-specific | ANSI standard |
| **Use case** | When ID is used across multiple tables | Simple auto-increment |

---

## 7. Synonyms — Aliases for Database Objects

Synonyms create **alternative names** for tables, views, sequences, procedures, etc.

```sql
-- ═══════════════════════════════════════════════════
-- PRIVATE SYNONYM (visible only to the creating user)
-- ═══════════════════════════════════════════════════
CREATE SYNONYM emp FOR hr.employees;
-- Now: SELECT * FROM emp;  (instead of hr.employees)

CREATE SYNONYM dept FOR hr.departments;
SELECT * FROM dept;  -- Same as: SELECT * FROM hr.departments

-- ═══════════════════════════════════════════════════
-- PUBLIC SYNONYM (visible to ALL users)
-- ═══════════════════════════════════════════════════
CREATE PUBLIC SYNONYM all_employees FOR hr.employees;
-- Any user can now: SELECT * FROM all_employees;

-- Drop synonyms
DROP SYNONYM emp;
DROP PUBLIC SYNONYM all_employees;

-- View your synonyms
SELECT synonym_name, table_owner, table_name
FROM user_synonyms;
```

### When to Use Synonyms

```
┌─────────────────────────────────────────────────────────────┐
│  USE SYNONYMS WHEN:                                          │
│                                                              │
│  ✅ Simplifying cross-schema access                         │
│     (hr.employees → emp)                                     │
│                                                              │
│  ✅ Hiding object ownership from app code                   │
│     App always uses "orders" regardless of which schema      │
│                                                              │
│  ✅ Creating a DB link abstraction                          │
│     CREATE SYNONYM remote_emp FOR employees@remote_db;       │
│     App code doesn't know it's accessing a remote database!  │
│                                                              │
│  ✅ Schema migration (old name → new name)                  │
│     Rename table, create synonym with old name               │
│     No app code changes needed!                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. String Functions — Oracle Specifics

```sql
-- ═══════════════════════════════════════════════════
-- CONCATENATION
-- ═══════════════════════════════════════════════════
-- Oracle uses || (double pipe) — NOT + like SQL Server
SELECT first_name || ' ' || last_name AS full_name FROM employees;
SELECT CONCAT('Hello', ' World') FROM dual;  -- Only takes 2 args!
-- For 3+, nest: CONCAT(CONCAT('a','b'),'c') or just use ||

-- ═══════════════════════════════════════════════════
-- SUBSTR (not SUBSTRING!)
-- ═══════════════════════════════════════════════════
SELECT SUBSTR('Oracle Database', 1, 6) FROM dual;   -- Oracle (start, length)
SELECT SUBSTR('Oracle Database', 8) FROM dual;       -- Database (from pos 8 to end)
SELECT SUBSTR('Oracle Database', -8) FROM dual;      -- Database (8 chars from end)

-- ═══════════════════════════════════════════════════
-- INSTR — Find position of substring
-- ═══════════════════════════════════════════════════
SELECT INSTR('Hello World', 'World') FROM dual;      -- 7
SELECT INSTR('Hello World', 'o') FROM dual;           -- 5 (first occurrence)
SELECT INSTR('Hello World', 'o', 1, 2) FROM dual;    -- 8 (second occurrence)
-- INSTR(string, search, start_pos, occurrence)

-- ═══════════════════════════════════════════════════
-- LENGTH, TRIM, PAD
-- ═══════════════════════════════════════════════════
SELECT LENGTH('Oracle') FROM dual;                     -- 6
SELECT LENGTHB('Oracle') FROM dual;                    -- 6 (in bytes)

SELECT TRIM('  hello  ') FROM dual;                    -- 'hello'
SELECT LTRIM('***hello***', '*') FROM dual;            -- 'hello***'
SELECT RTRIM('***hello***', '*') FROM dual;            -- '***hello'
SELECT TRIM(BOTH '*' FROM '***hello***') FROM dual;    -- 'hello'

SELECT LPAD('42', 8, '0') FROM dual;                  -- '00000042'
SELECT RPAD('hello', 10, '.') FROM dual;               -- 'hello.....'

-- ═══════════════════════════════════════════════════
-- REPLACE, TRANSLATE, REGEXP
-- ═══════════════════════════════════════════════════
SELECT REPLACE('Hello World', 'World', 'Oracle') FROM dual;  -- Hello Oracle

-- TRANSLATE: Character-by-character replacement
SELECT TRANSLATE('Hello 123', '123', 'ABC') FROM dual;       -- Hello ABC
SELECT TRANSLATE('abc123', 'abc', 'xyz') FROM dual;           -- xyz123

-- Remove all digits
SELECT TRANSLATE('Hello123World456', '0123456789', ' ') FROM dual;

-- REGEXP_REPLACE, REGEXP_SUBSTR, REGEXP_LIKE, REGEXP_INSTR
SELECT REGEXP_REPLACE('Hello  World  Oracle', '\s+', ' ') FROM dual;  -- Single spaces
SELECT REGEXP_SUBSTR('abc123def456', '\d+', 1, 1) FROM dual;          -- 123 (first number)
SELECT * FROM employees WHERE REGEXP_LIKE(email, '^[A-Z]+@');         -- Regex filter

-- ═══════════════════════════════════════════════════
-- INITCAP, UPPER, LOWER
-- ═══════════════════════════════════════════════════
SELECT INITCAP('hello world oracle') FROM dual;  -- Hello World Oracle
SELECT UPPER('hello') FROM dual;                  -- HELLO
SELECT LOWER('HELLO') FROM dual;                  -- hello
```

---

## 9. Analytical/Window Functions — Oracle Invented These!

Oracle was the first database to implement window functions (called "analytical functions" in Oracle). They're now ANSI standard.

```sql
-- ═══════════════════════════════════════════════════
-- RANKING FUNCTIONS
-- ═══════════════════════════════════════════════════
SELECT employee_id, first_name, department_id, salary,
       ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
       RANK()       OVER (ORDER BY salary DESC) AS rank,      -- Gaps after ties
       DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank -- No gaps
FROM employees;

-- Per-department ranking
SELECT employee_id, first_name, department_id, salary,
       RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank
FROM employees;

-- ═══════════════════════════════════════════════════
-- LAG / LEAD — Access previous/next row
-- ═══════════════════════════════════════════════════
SELECT employee_id, first_name, hire_date, salary,
       LAG(salary, 1, 0)  OVER (ORDER BY hire_date) AS prev_salary,
       LEAD(salary, 1, 0) OVER (ORDER BY hire_date) AS next_salary,
       salary - LAG(salary, 1) OVER (ORDER BY hire_date) AS salary_diff
FROM employees;

-- ═══════════════════════════════════════════════════
-- FIRST_VALUE, LAST_VALUE, NTH_VALUE
-- ═══════════════════════════════════════════════════
SELECT employee_id, department_id, salary,
       FIRST_VALUE(salary) OVER (
           PARTITION BY department_id ORDER BY salary DESC
       ) AS highest_in_dept,
       LAST_VALUE(salary) OVER (
           PARTITION BY department_id ORDER BY salary DESC
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS lowest_in_dept
FROM employees;

-- ═══════════════════════════════════════════════════
-- Running Totals, Moving Averages
-- ═══════════════════════════════════════════════════
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date) AS running_total,
       AVG(amount) OVER (
           ORDER BY order_date 
           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ) AS moving_avg_7day
FROM orders;

-- ═══════════════════════════════════════════════════
-- LISTAGG — Concatenate values from multiple rows (Oracle 11g+)
-- ═══════════════════════════════════════════════════
SELECT department_id,
       LISTAGG(first_name, ', ') WITHIN GROUP (ORDER BY first_name) AS employees
FROM employees
GROUP BY department_id;
-- 10 | Jennifer, Michael, Shelley
-- 20 | Michael, Pat

-- LISTAGG with overflow handling (Oracle 12c R2+):
SELECT department_id,
       LISTAGG(first_name, ', ' ON OVERFLOW TRUNCATE '...' WITH COUNT)
       WITHIN GROUP (ORDER BY first_name) AS employees
FROM employees
GROUP BY department_id;
```

---

## 10. MERGE — Oracle's UPSERT Statement

```sql
-- MERGE = INSERT if not exists, UPDATE if exists (all in one statement)
-- Also known as "UPSERT" in other databases

MERGE INTO target_table t
USING source_table s
ON (t.id = s.id)
WHEN MATCHED THEN
    UPDATE SET 
        t.name = s.name,
        t.salary = s.salary,
        t.updated_at = SYSDATE
    DELETE WHERE s.is_deleted = 'Y'  -- Optional: delete during merge!
WHEN NOT MATCHED THEN
    INSERT (id, name, salary, created_at)
    VALUES (s.id, s.name, s.salary, SYSDATE);

-- Practical example: Sync daily sales
MERGE INTO sales_summary dest
USING daily_sales src
ON (dest.product_id = src.product_id AND dest.sale_date = src.sale_date)
WHEN MATCHED THEN
    UPDATE SET dest.total_amount = dest.total_amount + src.amount,
               dest.total_qty = dest.total_qty + src.qty
WHEN NOT MATCHED THEN
    INSERT (product_id, sale_date, total_amount, total_qty)
    VALUES (src.product_id, src.sale_date, src.amount, src.qty);
```

---

## 11. Oracle-Specific Joins & Set Operations

```sql
-- ═══════════════════════════════════════════════════
-- OLD-STYLE ORACLE JOINS (you'll see in legacy code)
-- ═══════════════════════════════════════════════════

-- Inner Join (Oracle old syntax)
SELECT e.first_name, d.department_name
FROM employees e, departments d
WHERE e.department_id = d.department_id;

-- Left Outer Join (Oracle old syntax — the (+) operator)
SELECT e.first_name, d.department_name
FROM employees e, departments d
WHERE e.department_id = d.department_id(+);
-- The (+) goes on the DEFICIENT side (the side that might be NULL)

-- Right Outer Join
SELECT e.first_name, d.department_name
FROM employees e, departments d
WHERE e.department_id(+) = d.department_id;

-- ✅ PREFERRED: ANSI JOIN syntax (use this in new code!)
SELECT e.first_name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;

-- ⚠️ Oracle's (+) CANNOT do FULL OUTER JOIN — use ANSI syntax
SELECT e.first_name, d.department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.department_id;
```

---

## 12. Flashback — Oracle's Time Machine

```sql
-- ═══════════════════════════════════════════════════
-- FLASHBACK QUERY — See data as it was in the past
-- ═══════════════════════════════════════════════════

-- Query data as of 1 hour ago
SELECT * FROM employees
AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' HOUR)
WHERE employee_id = 100;

-- Query data as of a specific SCN
SELECT * FROM employees
AS OF SCN 12345678;

-- Compare current vs past
SELECT curr.salary AS current_salary, past.salary AS old_salary
FROM employees curr
JOIN employees AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' DAY) past
  ON curr.employee_id = past.employee_id
WHERE curr.employee_id = 100;

-- ═══════════════════════════════════════════════════
-- FLASHBACK VERSION QUERY — See ALL changes to a row
-- ═══════════════════════════════════════════════════
SELECT employee_id, salary,
       VERSIONS_STARTTIME, VERSIONS_ENDTIME,
       VERSIONS_OPERATION  -- I=Insert, U=Update, D=Delete
FROM employees
VERSIONS BETWEEN TIMESTAMP 
    SYSTIMESTAMP - INTERVAL '24' HOUR AND SYSTIMESTAMP
WHERE employee_id = 100;

-- ═══════════════════════════════════════════════════
-- FLASHBACK TABLE — Undo a table to a point in time
-- ═══════════════════════════════════════════════════
-- Oops, someone deleted all data!
-- Don't panic:
ALTER TABLE employees ENABLE ROW MOVEMENT;
FLASHBACK TABLE employees TO TIMESTAMP 
    (SYSTIMESTAMP - INTERVAL '30' MINUTE);
```

> 🔥 **Real-World Savior**: Flashback has saved countless DBAs from disasters. Accidentally deleted production data? Flashback. Need to audit who changed what? Flashback Versions. This is one of Oracle's killer features that competitors struggle to match.

---

## 🧠 Chapter Summary — Oracle SQL Cheat Sheet

```
┌──────────────────────────────────────────────────────────────┐
│         ORACLE SQL DIALECT — QUICK REFERENCE                  │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  DUAL TABLE:   SELECT expr FROM dual;                        │
│  CONCAT:       || (NOT +)                                    │
│  SUBSTRING:    SUBSTR(str, pos, len)                         │
│  NULL HANDLE:  NVL, NVL2, COALESCE, NULLIF                  │
│  INLINE IF:    DECODE(expr, val1, res1, ..., default)        │
│  STRING TYPE:  VARCHAR2 (NOT VARCHAR!)                       │
│  DATE TYPE:    DATE includes time! Use TIMESTAMP for frac.   │
│  BOOLEAN:      Doesn't exist in SQL! Use NUMBER(1)          │
│  AUTO-ID:      SEQUENCE (all versions) / IDENTITY (12c+)    │
│                                                               │
│  PAGINATION:                                                  │
│    Legacy:    ROWNUM (with subquery tricks)                   │
│    Modern:    ROW_NUMBER() OVER (...)                         │
│    12c+:      OFFSET n ROWS FETCH NEXT m ROWS ONLY          │
│                                                               │
│  HIERARCHY:   CONNECT BY PRIOR col = col                     │
│               START WITH condition                            │
│               LEVEL, SYS_CONNECT_BY_PATH, CONNECT_BY_ROOT   │
│                                                               │
│  UPSERT:      MERGE INTO ... USING ... ON ...                │
│               WHEN MATCHED THEN UPDATE                        │
│               WHEN NOT MATCHED THEN INSERT                    │
│                                                               │
│  FLASHBACK:   SELECT ... AS OF TIMESTAMP/SCN                 │
│               VERSIONS BETWEEN ... AND ...                    │
│               FLASHBACK TABLE ... TO TIMESTAMP                │
│                                                               │
│  OLD JOINS:   WHERE a.col = b.col(+)  →  LEFT JOIN          │
│  LISTAGG:     Concat rows into comma-separated string        │
│                                                               │
│  REMEMBER:                                                    │
│    • NULL = NULL is unknown (except in DECODE!)              │
│    • ROWNUM before ORDER BY, ROW_NUMBER() after              │
│    • Always use VARCHAR2 (not VARCHAR)                       │
│    • Always use TO_DATE() for date comparisons               │
│    • Date arithmetic: SYSDATE + 1 = tomorrow                │
└──────────────────────────────────────────────────────────────┘
```

---

## ❓ Interview Questions You WILL Be Asked

| # | Question | Key Answer |
|---|----------|-----------|
| 1 | ROWNUM vs ROW_NUMBER()? | ROWNUM assigned before ORDER BY (tricky), ROW_NUMBER() after (reliable). Use ROW_NUMBER() for pagination. |
| 2 | What is DUAL? | Special 1-row, 1-column table. Used when SELECT needs FROM but you just want to compute. |
| 3 | DECODE vs CASE? | DECODE is Oracle-specific, handles NULL equality. CASE is ANSI standard. Use CASE for new code. |
| 4 | NVL vs COALESCE? | NVL takes 2 args, always evaluates both. COALESCE takes N args, short-circuits (stops at first non-NULL). |
| 5 | Sequence vs Identity? | Sequence: separate object, shared across tables. Identity (12c+): column property, ANSI standard. |
| 6 | DELETE vs TRUNCATE? | DELETE: DML, logged, can rollback, fires triggers. TRUNCATE: DDL, minimal logging, no rollback, no triggers. |
| 7 | What is Flashback? | Oracle's time-travel: query past data (AS OF), see change history (VERSIONS), undo mistakes (FLASHBACK TABLE). |
| 8 | VARCHAR vs VARCHAR2? | Always use VARCHAR2 in Oracle. VARCHAR is reserved and may behave differently in future versions. |
| 9 | How to do pagination in Oracle? | ROWNUM (legacy), ROW_NUMBER() OVER (standard), OFFSET-FETCH (12c+, cleanest). |
| 10 | What is CONNECT BY? | Oracle's hierarchical query syntax. Traverses parent-child trees. Uses LEVEL, SYS_CONNECT_BY_PATH. |

---

> **Next Chapter**: [2B.4 — PL/SQL — Oracle's Procedural Language →](./04-PLSQL.md)
