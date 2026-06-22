# 2.1 — SQL Basics: SELECT, WHERE, ORDER BY 🟢⭐

> **"SQL is the lingua franca of data. Learn it once, speak it everywhere."**

---

## 🧭 What You'll Master in This Chapter

By the end of this chapter, you'll write queries with the confidence of someone who's been doing it for years. Not kidding.

```
You → "Show me all customers from India who spent over ₹10,000, sorted by highest spender first"
SQL  → Done. In 3 lines.
```

---

## 📖 What is SQL?

**SQL** = **Structured Query Language** (pronounced "sequel" or "S-Q-L" — both are correct, don't let anyone tell you otherwise).

Born in **1974** at IBM. Standardized by **ANSI/ISO**. Still the **#1 most-used language** for data in 2024.

```
┌──────────────────────────────────────────────────────┐
│              SQL Language Categories                  │
├──────────────┬───────────────────────────────────────┤
│  DQL         │  SELECT (Data Query Language)         │  ← This chapter
│  DDL         │  CREATE, ALTER, DROP                  │  ← Chapter 2.2
│  DML         │  INSERT, UPDATE, DELETE, MERGE        │  ← Chapter 2.3
│  DCL         │  GRANT, REVOKE                        │
│  TCL         │  COMMIT, ROLLBACK, SAVEPOINT          │
└──────────────┴───────────────────────────────────────┘
```

> 💡 **Fun Fact:** Some textbooks say SELECT is DML. Some say it's DQL. Both camps exist. In interviews, know both perspectives.

---

## 🗄️ Sample Database — Used Throughout This Guide

We'll use a **fictional e-commerce company** called **"TechMart"**. Here are the core tables:

```sql
-- CUSTOMERS
+----+----------------+--------+----------+------------+
| id | name           | city   | country  | joined     |
+----+----------------+--------+----------+------------+
|  1 | Ritesh Singh   | Delhi  | India    | 2022-01-15 |
|  2 | Sarah Connor   | NYC    | USA      | 2021-06-20 |
|  3 | Akira Tanaka   | Tokyo  | Japan    | 2023-03-10 |
|  4 | Maria Garcia   | Madrid | Spain    | 2022-11-05 |
|  5 | John Smith     | London | UK       | 2020-09-01 |
|  6 | Priya Sharma   | Mumbai | India    | 2023-07-22 |
+----+----------------+--------+----------+------------+

-- ORDERS
+----+-------------+------------+---------+--------+
| id | customer_id | order_date | amount  | status |
+----+-------------+------------+---------+--------+
|  1 |     1       | 2024-01-10 | 15000   | shipped|
|  2 |     2       | 2024-01-15 |  8500   | delivered|
|  3 |     1       | 2024-02-20 | 32000   | delivered|
|  4 |     3       | 2024-02-25 |  4200   | pending|
|  5 |     5       | 2024-03-01 | 11000   | shipped|
|  6 |     4       | 2024-03-05 |  6700   | cancelled|
|  7 |     6       | 2024-03-10 | 21000   | delivered|
|  8 |     1       | 2024-03-15 |  9800   | pending|
+----+-------------+------------+---------+--------+

-- PRODUCTS
+----+-------------------+----------+-------+----------+
| id | name              | category | price | stock    |
+----+-------------------+----------+-------+----------+
|  1 | iPhone 15         | Phones   | 79999 |   50     |
|  2 | Samsung Galaxy S24| Phones   | 69999 |   75     |
|  3 | MacBook Pro M3    | Laptops  | 199999|   25     |
|  4 | Dell XPS 15       | Laptops  | 149999|   40     |
|  5 | AirPods Pro       | Audio    | 24999 |  200     |
|  6 | Sony WH-1000XM5   | Audio    | 29999 |  150     |
+----+-------------------+----------+-------+----------+
```

---

## 🔥 1. The SELECT Statement — Your First Superpower

### The Most Basic Query in the Universe

```sql
SELECT * FROM customers;
```

That's it. You just asked: *"Show me everything from the customers table."*

The `*` means **all columns**. Think of it as "give me everything."

### Selecting Specific Columns

```sql
SELECT name, city, country
FROM customers;
```

**Result:**
```
+----------------+--------+---------+
| name           | city   | country |
+----------------+--------+---------+
| Ritesh Singh   | Delhi  | India   |
| Sarah Connor   | NYC    | USA     |
| Akira Tanaka   | Tokyo  | Japan   |
| Maria Garcia   | Madrid | Spain   |
| John Smith     | London | UK      |
| Priya Sharma   | Mumbai | India   |
+----------------+--------+---------+
```

> 💡 **PRO TIP:** In production, **NEVER use `SELECT *`**. Always specify columns. Why?
> - Wastes bandwidth (fetches columns you don't need)
> - Breaks your app if someone adds/removes columns
> - Prevents index-only scans (performance killer)
> - Makes code harder to understand

### SQL Execution Order — THE MOST IMPORTANT CONCEPT

**What you write** is NOT how SQL **executes** it. This trips up 90% of beginners:

```
What you WRITE:          How SQL EXECUTES:
─────────────────        ─────────────────
1. SELECT               ← 5th (pick columns)
2. FROM                 ← 1st (which table?)
3. WHERE                ← 2nd (filter rows)
4. GROUP BY             ← 3rd (group rows)
5. HAVING               ← 4th (filter groups)
6. ORDER BY             ← 6th (sort result)
7. LIMIT / OFFSET       ← 7th (paginate)
```

> ⭐ **This is why you can't use a column alias in WHERE but CAN use it in ORDER BY.** The alias doesn't exist yet when WHERE runs!

```sql
-- ❌ This FAILS (alias not available in WHERE)
SELECT name, amount * 1.18 AS total_with_tax
FROM orders
WHERE total_with_tax > 10000;    -- ERROR!

-- ✅ This WORKS (alias available in ORDER BY)
SELECT name, amount * 1.18 AS total_with_tax
FROM orders
ORDER BY total_with_tax DESC;    -- WORKS!
```

> 💡 **Exception:** MySQL actually allows aliases in WHERE/GROUP BY/HAVING. But don't rely on it — it's non-standard.

---

## 🔥 2. Column Aliases — Rename Your Output

```sql
SELECT 
    name AS customer_name,
    city AS location,
    country
FROM customers;
```

- `AS` is optional (but recommended for readability)
- Use double quotes for aliases with spaces: `"Customer Name"` (ANSI standard)

```sql
-- Aliases with spaces
SELECT 
    name AS "Customer Name",       -- ANSI standard (Oracle, PostgreSQL)
    city AS [Customer City],       -- SQL Server specific
    country AS `Customer Country`  -- MySQL specific
FROM customers;
```

### Computed Columns with Aliases

```sql
SELECT 
    name,
    price,
    price * 0.18 AS gst_amount,
    price * 1.18 AS price_with_gst
FROM products;
```

---

## 🔥 3. WHERE — Filtering Rows Like a Pro

WHERE is your filter. Only rows that satisfy the condition survive.

### Comparison Operators

```sql
-- Equal
SELECT * FROM customers WHERE country = 'India';

-- Not Equal (both work, <> is ANSI standard)
SELECT * FROM customers WHERE country <> 'USA';
SELECT * FROM customers WHERE country != 'USA';    -- Also works

-- Greater than / Less than
SELECT * FROM orders WHERE amount > 10000;
SELECT * FROM orders WHERE amount <= 5000;

-- Between (inclusive on BOTH sides)
SELECT * FROM orders WHERE amount BETWEEN 5000 AND 15000;
-- Same as: WHERE amount >= 5000 AND amount <= 15000
```

### Logical Operators: AND, OR, NOT

```sql
-- AND — Both conditions must be true
SELECT * FROM customers 
WHERE country = 'India' AND city = 'Delhi';

-- OR — At least one condition must be true
SELECT * FROM customers 
WHERE country = 'India' OR country = 'Japan';

-- NOT — Negate the condition
SELECT * FROM orders 
WHERE NOT status = 'cancelled';

-- Combining with parentheses (CRITICAL for correct logic)
SELECT * FROM orders 
WHERE (status = 'shipped' OR status = 'delivered')
  AND amount > 10000;
```

> ⚠️ **CLASSIC TRAP — Operator Precedence:**
> `AND` has higher precedence than `OR`. Always use parentheses!
> ```sql
> -- Without parentheses — WRONG logic (AND binds first)
> WHERE status = 'shipped' OR status = 'delivered' AND amount > 10000
> -- Means: shipped (any amount) OR (delivered AND > 10000) ← NOT what you want!
>
> -- With parentheses — CORRECT logic
> WHERE (status = 'shipped' OR status = 'delivered') AND amount > 10000
> ```

### IN — Multiple Values (Cleaner than OR)

```sql
-- Instead of:
SELECT * FROM customers 
WHERE country = 'India' OR country = 'Japan' OR country = 'USA';

-- Use IN:
SELECT * FROM customers 
WHERE country IN ('India', 'Japan', 'USA');

-- NOT IN
SELECT * FROM customers 
WHERE country NOT IN ('India', 'USA');
```

> ⚠️ **NULL TRAP with NOT IN:** If the list contains NULL, `NOT IN` returns **no rows**!
> ```sql
> WHERE id NOT IN (1, 2, NULL)    -- Returns NOTHING! (because NULL comparison = UNKNOWN)
> ```

### LIKE — Pattern Matching

```sql
-- % = any number of characters (including zero)
-- _ = exactly one character

SELECT * FROM customers WHERE name LIKE 'S%';       -- Starts with S
SELECT * FROM customers WHERE name LIKE '%Singh';    -- Ends with Singh
SELECT * FROM customers WHERE name LIKE '%ar%';      -- Contains "ar"
SELECT * FROM customers WHERE name LIKE '_a%';       -- 2nd character is 'a'
SELECT * FROM customers WHERE city LIKE '____';      -- Exactly 4 characters
```

```
Pattern Cheat Sheet:
┌───────────┬──────────────────────────────────┐
│ Pattern   │ Matches                          │
├───────────┼──────────────────────────────────┤
│ 'A%'      │ Starts with A                    │
│ '%A'      │ Ends with A                      │
│ '%A%'     │ Contains A                       │
│ '_A%'     │ Second char is A                 │
│ 'A_%_%'   │ Starts with A, at least 3 chars  │
│ 'A%Z'     │ Starts with A, ends with Z       │
└───────────┴──────────────────────────────────┘
```

> 💡 **Performance Warning:** `LIKE '%something'` (leading wildcard) **cannot use indexes**. It forces a full table scan. Avoid in large tables!

**Escaping wildcards:**
```sql
-- If your data contains % or _, use ESCAPE
SELECT * FROM products WHERE name LIKE '%50\%%' ESCAPE '\';   -- Finds "50%"
```

### IS NULL / IS NOT NULL

```sql
-- NULL is NOT a value — it's the ABSENCE of value
-- You CANNOT use = or != with NULL

-- ❌ WRONG (always returns empty)
SELECT * FROM orders WHERE discount = NULL;

-- ✅ CORRECT
SELECT * FROM orders WHERE discount IS NULL;
SELECT * FROM orders WHERE discount IS NOT NULL;
```

> ⭐ **The Three-Valued Logic of SQL:**
> Every comparison with NULL yields **UNKNOWN** (not TRUE, not FALSE).
> `NULL = NULL` → UNKNOWN (not TRUE!)
> `NULL <> NULL` → UNKNOWN (not FALSE!)
> This is why `WHERE discount = NULL` returns nothing.

---

## 🔥 4. DISTINCT — Remove Duplicates

```sql
-- Get unique countries
SELECT DISTINCT country FROM customers;

-- DISTINCT on multiple columns (unique COMBINATIONS)
SELECT DISTINCT country, city FROM customers;

-- Count distinct values
SELECT COUNT(DISTINCT country) AS unique_countries 
FROM customers;
```

> 💡 **Performance Note:** DISTINCT sorts or hashes data internally. On large datasets, it can be expensive. Consider if you really need it.

---

## 🔥 5. ORDER BY — Sort Your Results

```sql
-- Ascending (default)
SELECT * FROM customers ORDER BY name;           -- A → Z
SELECT * FROM customers ORDER BY name ASC;       -- Same thing

-- Descending
SELECT * FROM orders ORDER BY amount DESC;       -- Highest first

-- Multiple columns (sort by first, then break ties with second)
SELECT * FROM customers 
ORDER BY country ASC, name ASC;

-- Sort by column position (NOT recommended, but works)
SELECT name, city, country FROM customers 
ORDER BY 3, 1;    -- 3rd column (country), then 1st (name)

-- Sort by expression
SELECT name, price, price * 0.18 AS tax
FROM products
ORDER BY price * 0.18 DESC;

-- Sort by alias (works because ORDER BY executes AFTER SELECT)
SELECT name, price * 1.18 AS total
FROM products
ORDER BY total DESC;
```

### NULL Sorting Behavior — Cross-Database Differences!

```
┌──────────────┬──────────────────────────────────┐
│ Database     │ NULLs sort...                    │
├──────────────┼──────────────────────────────────┤
│ Oracle       │ LAST in ASC, FIRST in DESC       │
│ PostgreSQL   │ LAST in ASC, FIRST in DESC       │
│ SQL Server   │ FIRST in ASC, LAST in DESC       │
│ MySQL        │ FIRST in ASC, LAST in DESC       │
└──────────────┴──────────────────────────────────┘
```

```sql
-- Override NULL sorting (Oracle & PostgreSQL)
SELECT * FROM orders ORDER BY discount NULLS FIRST;
SELECT * FROM orders ORDER BY discount NULLS LAST;

-- SQL Server workaround
SELECT * FROM orders ORDER BY CASE WHEN discount IS NULL THEN 1 ELSE 0 END, discount;
```

---

## 🔥 6. LIMIT / TOP / FETCH — Pagination & Row Limiting

This is where databases diverge the most. Same goal, different syntax:

### MySQL & PostgreSQL: LIMIT

```sql
-- First 5 rows
SELECT * FROM customers LIMIT 5;

-- Skip 10, get next 5 (pagination)
SELECT * FROM customers LIMIT 5 OFFSET 10;

-- MySQL shorthand (offset, count)
SELECT * FROM customers LIMIT 10, 5;    -- Skip 10, get 5
```

### SQL Server: TOP

```sql
-- First 5 rows
SELECT TOP 5 * FROM customers;

-- Top 5 by amount
SELECT TOP 5 * FROM orders ORDER BY amount DESC;

-- TOP with PERCENT
SELECT TOP 10 PERCENT * FROM orders ORDER BY amount DESC;

-- TOP WITH TIES (includes rows that tie with the last row)
SELECT TOP 5 WITH TIES * FROM orders ORDER BY amount DESC;
```

### Oracle (12c+) & ANSI SQL: FETCH FIRST

```sql
-- ANSI standard (works in Oracle 12c+, PostgreSQL, SQL Server 2012+)
SELECT * FROM customers
ORDER BY name
OFFSET 10 ROWS
FETCH FIRST 5 ROWS ONLY;

-- FETCH NEXT (same as FETCH FIRST)
FETCH NEXT 5 ROWS ONLY;

-- With ties
FETCH FIRST 5 ROWS WITH TIES;
```

### Oracle (Pre-12c): ROWNUM

```sql
-- Old Oracle style (still seen in legacy code)
SELECT * FROM (
    SELECT * FROM customers ORDER BY name
) WHERE ROWNUM <= 5;

-- ⚠️ WRONG: ROWNUM is assigned BEFORE ORDER BY
SELECT * FROM customers WHERE ROWNUM <= 5 ORDER BY name;    -- Wrong order!
```

### Cross-Database Pagination Cheat Sheet

```
┌──────────────┬─────────────────────────────────────────────┐
│ Database     │ Get rows 11-20 (page 2, 10 per page)       │
├──────────────┼─────────────────────────────────────────────┤
│ MySQL        │ LIMIT 10 OFFSET 10                         │
│ PostgreSQL   │ LIMIT 10 OFFSET 10                         │
│ SQL Server   │ OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY    │
│ Oracle 12c+  │ OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY    │
│ Oracle <12c  │ Subquery with ROWNUM                       │
└──────────────┴─────────────────────────────────────────────┘
```

> 💡 **Performance Warning:** `OFFSET` with large values is SLOW. The database still reads and discards all skipped rows. For large datasets, use **keyset pagination** (WHERE id > last_seen_id) instead.

---

## 🔥 7. Expressions & Operators in SELECT

### Arithmetic

```sql
SELECT 
    name,
    price,
    price * 0.10 AS discount,
    price - (price * 0.10) AS final_price,
    price * 1.18 AS price_with_gst
FROM products;
```

### String Concatenation — Cross-Database Differences!

```sql
-- ANSI standard (PostgreSQL, Oracle)
SELECT first_name || ' ' || last_name AS full_name FROM employees;

-- SQL Server
SELECT first_name + ' ' + last_name AS full_name FROM employees;
-- Or use CONCAT (safer with NULLs)
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM employees;

-- MySQL
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM employees;
```

> ⚠️ **NULL in concatenation:**
> - `||` with NULL → NULL (in Oracle/PostgreSQL)
> - `+` with NULL → NULL (in SQL Server)
> - `CONCAT()` treats NULL as '' (empty string) in MySQL & SQL Server

### CASE — SQL's IF-ELSE

```sql
-- Simple CASE
SELECT 
    name,
    amount,
    CASE status
        WHEN 'delivered' THEN '✅ Delivered'
        WHEN 'shipped'   THEN '📦 In Transit'
        WHEN 'pending'   THEN '⏳ Waiting'
        WHEN 'cancelled' THEN '❌ Cancelled'
        ELSE '❓ Unknown'
    END AS status_display
FROM orders;

-- Searched CASE (more flexible — allows conditions)
SELECT 
    name,
    amount,
    CASE 
        WHEN amount >= 20000 THEN '🔥 Premium Order'
        WHEN amount >= 10000 THEN '⭐ Standard Order'
        WHEN amount >= 5000  THEN '📦 Basic Order'
        ELSE '🔹 Small Order'
    END AS order_tier
FROM orders;
```

> 💡 CASE is evaluated **top to bottom**. The FIRST matching condition wins.

---

## 🔥 8. Useful Built-in Functions (Quick Reference)

### String Functions

```sql
-- These work across most databases (minor syntax differences)
SELECT 
    UPPER('hello')           AS upper_result,     -- 'HELLO'
    LOWER('HELLO')           AS lower_result,     -- 'hello'
    LENGTH('Database')       AS len,              -- 8 (LEN in SQL Server)
    TRIM('  hello  ')        AS trimmed,          -- 'hello'
    SUBSTRING('Database', 1, 4) AS sub,           -- 'Data' (SUBSTR in Oracle)
    REPLACE('2024-01-01', '-', '/') AS replaced,  -- '2024/01/01'
    LEFT('Database', 4)      AS left_part,        -- 'Data'
    RIGHT('Database', 4)     AS right_part,       -- 'base'
    REVERSE('SQL')           AS reversed;         -- 'LQS'
```

### Date Functions (Cross-Database)

```sql
-- Current date/time
-- Oracle:      SYSDATE, SYSTIMESTAMP
-- SQL Server:  GETDATE(), SYSDATETIME()
-- MySQL:       NOW(), CURDATE()
-- PostgreSQL:  NOW(), CURRENT_TIMESTAMP

-- Extract parts
-- ANSI:        EXTRACT(YEAR FROM order_date)
-- SQL Server:  DATEPART(YEAR, order_date), YEAR(order_date)
-- MySQL:       YEAR(order_date), MONTH(order_date)
```

### NULL Handling Functions

```sql
-- COALESCE (ANSI standard — works everywhere) — returns first non-NULL
SELECT COALESCE(discount, 0) AS discount FROM orders;
SELECT COALESCE(phone, email, 'No Contact') AS contact FROM customers;

-- Database-specific alternatives:
-- Oracle:      NVL(discount, 0), NVL2(discount, 'Has', 'None')
-- SQL Server:  ISNULL(discount, 0)
-- MySQL:       IFNULL(discount, 0), IF(discount IS NULL, 0, discount)

-- NULLIF — Returns NULL if two values are equal (prevents division by zero!)
SELECT amount / NULLIF(quantity, 0) AS unit_price FROM orders;
-- If quantity = 0, NULLIF returns NULL, and NULL/0 = NULL (not error!)
```

---

## 🔥 9. DUAL Table & SELECT Without FROM

```sql
-- Oracle requires FROM, so it has the DUAL table (a dummy 1-row table)
SELECT 1 + 1 FROM DUAL;                -- Oracle
SELECT SYSDATE FROM DUAL;              -- Oracle

-- Others don't need FROM for simple expressions
SELECT 1 + 1;                          -- MySQL, PostgreSQL, SQL Server
SELECT GETDATE();                      -- SQL Server
SELECT NOW();                          -- MySQL, PostgreSQL
```

---

## 🧠 10. Common Mistakes Beginners Make

| # | Mistake | Fix |
|---|---------|-----|
| 1 | Using `= NULL` instead of `IS NULL` | `WHERE col IS NULL` |
| 2 | Forgetting operator precedence (AND vs OR) | Use parentheses |
| 3 | Using `SELECT *` in production | List specific columns |
| 4 | Not knowing execution order | FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY |
| 5 | String comparison case sensitivity | Depends on collation! MySQL default is case-insensitive, PostgreSQL is case-sensitive |
| 6 | Using `LIKE '%value'` on large tables | Leading wildcards kill performance |
| 7 | `NOT IN` with NULLs in the list | Use `NOT EXISTS` instead |
| 8 | Relying on row order without ORDER BY | SQL returns rows in **no guaranteed order** without ORDER BY |

---

## ⚔️ Quick Challenge — Test Yourself!

Try writing these queries before looking at the answers:

**Q1:** Get all customers from India, sorted by name
```sql
SELECT * FROM customers 
WHERE country = 'India' 
ORDER BY name;
```

**Q2:** Find orders above ₹10,000 that are NOT cancelled
```sql
SELECT * FROM orders 
WHERE amount > 10000 
  AND status != 'cancelled';
```

**Q3:** Get the top 3 most expensive products with their GST-inclusive price
```sql
-- MySQL/PostgreSQL
SELECT name, price, price * 1.18 AS price_with_gst
FROM products
ORDER BY price DESC
LIMIT 3;

-- SQL Server
SELECT TOP 3 name, price, price * 1.18 AS price_with_gst
FROM products
ORDER BY price DESC;

-- Oracle 12c+
SELECT name, price, price * 1.18 AS price_with_gst
FROM products
ORDER BY price DESC
FETCH FIRST 3 ROWS ONLY;
```

**Q4:** Find all customers whose name contains "ar" (case-insensitive)
```sql
-- PostgreSQL (case-sensitive by default, so use ILIKE)
SELECT * FROM customers WHERE name ILIKE '%ar%';

-- Others (depends on collation)
SELECT * FROM customers WHERE UPPER(name) LIKE '%AR%';

-- MySQL (case-insensitive by default with utf8_general_ci)
SELECT * FROM customers WHERE name LIKE '%ar%';
```

---

## 🎯 Key Takeaways

```
┌─────────────────────────────────────────────────────────────────┐
│  ✅ SELECT picks columns, FROM picks the table                 │
│  ✅ WHERE filters rows BEFORE grouping                         │
│  ✅ ORDER BY sorts the final result                            │
│  ✅ LIMIT/TOP/FETCH restricts how many rows you get            │
│  ✅ NULL is not a value — it's the absence of value            │
│  ✅ SQL execution order ≠ writing order (memorize this!)       │
│  ✅ Different databases have syntax variations — know them!    │
│  ✅ NEVER use SELECT * in production code                      │
└─────────────────────────────────────────────────────────────────┘
```

---

> **Next Chapter →** [2.2 DDL — Creating the World (CREATE, ALTER, DROP)](./02-DDL.md)
