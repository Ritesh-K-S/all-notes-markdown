# 2D.3 — MySQL SQL Dialect & Features 🟡

> **"Every SQL database speaks SQL — but each has its own accent. Master MySQL's dialect, and you'll write queries that sing."**

---

## 📌 What You'll Master in This Chapter

- **AUTO_INCREMENT** — MySQL's identity column (and its quirks)
- **LIMIT & OFFSET** — pagination the MySQL way
- **GROUP_CONCAT** — turn rows into comma-separated strings
- **ON DUPLICATE KEY UPDATE** — MySQL's UPSERT superpower
- **MySQL-specific functions** — string, date, JSON, and more
- **MySQL SQL modes** — strict vs permissive (and why it matters)
- How MySQL differs from Oracle, SQL Server, and PostgreSQL

---

## 🔥 1. AUTO_INCREMENT — Automatic ID Generation

Every table needs a primary key. In MySQL, `AUTO_INCREMENT` generates sequential IDs automatically.

### Basic Usage

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- INSERT without specifying id — MySQL generates it!
INSERT INTO users (name, email) VALUES ('Ritesh', 'ritesh@example.com');   -- id = 1
INSERT INTO users (name, email) VALUES ('Priya', 'priya@example.com');     -- id = 2
INSERT INTO users (name, email) VALUES ('John', 'john@example.com');       -- id = 3

-- Get the last generated ID
SELECT LAST_INSERT_ID();    -- Returns 3
```

### AUTO_INCREMENT Behavior & Gotchas

```
┌──────────────────────────────────────────────────────────────┐
│  AUTO_INCREMENT — Things You MUST Know                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. GAPS ARE NORMAL (and expected!)                            │
│     • DELETE id=2 → next insert is id=4, NOT id=2             │
│     • Failed INSERT → counter still increments                 │
│     • ROLLBACK → counter still increments                      │
│     • Never rely on AUTO_INCREMENT being gapless!              │
│                                                                │
│  2. ONE PER TABLE                                              │
│     • Only ONE column can be AUTO_INCREMENT                    │
│     • It MUST be indexed (usually PRIMARY KEY)                 │
│     • Must be an integer type (INT, BIGINT, etc.)              │
│                                                                │
│  3. COUNTER IS IN MEMORY (pre-8.0)                             │
│     • MySQL < 8.0: counter = MAX(id) + 1 on restart!          │
│       → IDs can be REUSED after delete + restart 😱            │
│     • MySQL 8.0+: counter persisted in redo log ✅             │
│       → No more ID reuse after restart                         │
│                                                                │
│  4. CONCURRENCY SAFE                                           │
│     • AUTO_INCREMENT uses a lightweight lock                   │
│     • No two connections get the same ID                       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Advanced AUTO_INCREMENT Operations

```sql
-- Set starting value
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    total DECIMAL(10,2)
) AUTO_INCREMENT = 1000;     -- First id will be 1000

-- Reset AUTO_INCREMENT on existing table
ALTER TABLE orders AUTO_INCREMENT = 5000;

-- Check current AUTO_INCREMENT value
SELECT AUTO_INCREMENT 
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'techmart' AND TABLE_NAME = 'orders';

-- Insert with explicit ID (skips auto-increment)
INSERT INTO users (id, name, email) VALUES (100, 'Admin', 'admin@x.com');
-- Next auto-increment will be 101

-- Set increment step (for multi-server setups)
SET @@auto_increment_increment = 2;     -- Increment by 2
SET @@auto_increment_offset = 1;        -- Start from 1
-- Server 1: 1, 3, 5, 7, 9...
-- Server 2: 2, 4, 6, 8, 10...   (offset=2 on server 2)
```

### INT vs BIGINT — Choose Wisely

```
┌─────────────┬──────────────────────┬──────────────────────────┐
│ Type        │ Max Unsigned Value   │ When to Use              │
├─────────────┼──────────────────────┼──────────────────────────┤
│ TINYINT     │ 255                  │ Lookup tables, enums     │
│ SMALLINT    │ 65,535               │ Small reference tables   │
│ MEDIUMINT   │ 16,777,215           │ Medium-sized tables      │
│ INT         │ 4,294,967,295 (~4B)  │ Most tables              │
│ BIGINT      │ 18.4 quintillion     │ High-volume tables       │
└─────────────┴──────────────────────┴──────────────────────────┘

-- ALWAYS use UNSIGNED for AUTO_INCREMENT (no negative IDs!)
CREATE TABLE events (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    event_type VARCHAR(50)
);
```

> 💡 **Pro Tip:** Use `BIGINT UNSIGNED` for any table that might grow large (users, orders, events, logs). INT UNSIGNED maxes out at ~4 billion — which sounds like a lot until you realize Facebook generates billions of events per day. The extra 4 bytes per row is a tiny price for never hitting the limit.

---

## 🔥 2. LIMIT & OFFSET — Pagination MySQL-Style

MySQL's `LIMIT` clause is the simplest pagination syntax in the SQL world.

### Basic Syntax

```sql
-- Get first 10 rows
SELECT * FROM products LIMIT 10;

-- Get 10 rows, starting from row 21 (page 3, 10 per page)
SELECT * FROM products LIMIT 10 OFFSET 20;

-- MySQL shorthand: LIMIT offset, count
SELECT * FROM products LIMIT 20, 10;    -- Same as above
-- ⚠️ Confusing: first number is OFFSET, second is COUNT
-- LIMIT 20, 10 = skip 20, get 10 (NOT "get 20 through 10")
```

### Pagination Pattern

```sql
-- Page 1 (rows 1-10)
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 0;

-- Page 2 (rows 11-20)
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 10;

-- Page 3 (rows 21-30)
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 20;

-- General formula:
-- OFFSET = (page_number - 1) * page_size
-- LIMIT  = page_size
```

### The OFFSET Performance Problem (And the Fix)

```
THE PROBLEM:
─────────────
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 1000000;

MySQL reads 1,000,010 rows, discards 1,000,000, returns 10. 😱
The deeper you paginate, the slower it gets.

Page 1:      reads 10 rows      → ⚡ fast
Page 100:    reads 1,000 rows   → still okay
Page 10000:  reads 100,000 rows → 🐢 slow
Page 100000: reads 1,000,000    → 🐌 painfully slow
```

```sql
-- ❌ SLOW: Deep offset pagination
SELECT * FROM products 
ORDER BY id 
LIMIT 10 OFFSET 1000000;    -- Reads 1M+ rows

-- ✅ FAST: Keyset pagination (seek method)
-- Remember the last ID from previous page
SELECT * FROM products 
WHERE id > 1000000           -- Jump directly to the right spot!
ORDER BY id 
LIMIT 10;                    -- Only reads 10 rows ⚡

-- ✅ FAST: Deferred join (if you need OFFSET)
SELECT p.* FROM products p
INNER JOIN (
    SELECT id FROM products ORDER BY id LIMIT 10 OFFSET 1000000
) AS sub ON p.id = sub.id;
-- The subquery only reads IDs (index-only scan), then joins to get full rows
```

> ⭐ **Rule of Thumb:** Use OFFSET for small datasets or shallow pages. Use **keyset pagination** (WHERE id > last_id) for large datasets or infinite scroll. Every major tech company does this.

### LIMIT with Other Statements

```sql
-- LIMIT with UPDATE (MySQL-specific!)
UPDATE products SET price = price * 0.9 
WHERE category = 'Electronics' 
ORDER BY price DESC 
LIMIT 5;    -- Only update top 5 most expensive

-- LIMIT with DELETE (MySQL-specific!)
DELETE FROM logs 
WHERE created_at < '2024-01-01' 
ORDER BY created_at 
LIMIT 10000;    -- Delete in batches to avoid long locks

-- ⚠️ This does NOT work in Oracle, SQL Server, or PostgreSQL!
-- They don't support LIMIT/ORDER BY in UPDATE/DELETE
```

---

## 🔥 3. GROUP_CONCAT — Rows Into Strings

One of MySQL's most beloved functions. It takes multiple row values and concatenates them into a single string.

### Basic Usage

```sql
-- Without GROUP_CONCAT (multiple rows per customer)
SELECT customer_id, product_name
FROM order_items
WHERE customer_id = 1;
/*
+────────────+──────────────────+
| customer_id| product_name     |
+────────────+──────────────────+
| 1          | iPhone 15        |
| 1          | AirPods Pro      |
| 1          | MacBook Pro M3   |
+────────────+──────────────────+
*/

-- WITH GROUP_CONCAT (one row per customer!)
SELECT 
    customer_id,
    GROUP_CONCAT(product_name) AS products
FROM order_items
GROUP BY customer_id;
/*
+────────────+─────────────────────────────────────+
| customer_id| products                             |
+────────────+─────────────────────────────────────+
| 1          | iPhone 15,AirPods Pro,MacBook Pro M3 |
| 2          | Samsung Galaxy S24,Sony WH-1000XM5   |
+────────────+─────────────────────────────────────+
*/
```

### Full Power of GROUP_CONCAT

```sql
-- Custom separator
SELECT 
    department,
    GROUP_CONCAT(name SEPARATOR ' | ') AS employees
FROM staff
GROUP BY department;
-- Output: "Ritesh | Priya | John"

-- With ORDER BY inside GROUP_CONCAT
SELECT 
    department,
    GROUP_CONCAT(name ORDER BY name ASC SEPARATOR ', ') AS employees
FROM staff
GROUP BY department;
-- Output: "John, Priya, Ritesh" (alphabetical!)

-- Remove duplicates with DISTINCT
SELECT 
    customer_id,
    GROUP_CONCAT(DISTINCT category ORDER BY category SEPARATOR ', ') AS categories
FROM order_items
GROUP BY customer_id;
-- Output: "Audio, Laptops, Phones" (unique categories only)

-- Combine multiple columns
SELECT 
    department,
    GROUP_CONCAT(name, ' (', salary, ')' ORDER BY salary DESC SEPARATOR '\n') AS details
FROM staff
GROUP BY department;
-- Output: "Ritesh (80000)\nPriya (60000)\nJohn (50000)"
```

### GROUP_CONCAT Limit — The Hidden Trap!

```sql
-- Default max length is only 1024 bytes!
-- If your concatenated string is longer, IT GETS SILENTLY TRUNCATED! 😱

-- Check current limit
SHOW VARIABLES LIKE 'group_concat_max_len';    -- Default: 1024

-- Increase it (session or global)
SET SESSION group_concat_max_len = 1000000;     -- ~1MB
SET GLOBAL group_concat_max_len = 1000000;      -- For all connections

-- Or in my.cnf:
-- group_concat_max_len = 1000000
```

> ⭐ **Equivalent in Other Databases:**
> - **PostgreSQL:** `STRING_AGG(name, ', ')` ← closest equivalent
> - **SQL Server:** `STRING_AGG(name, ', ')` (2017+) or `FOR XML PATH`
> - **Oracle:** `LISTAGG(name, ', ') WITHIN GROUP (ORDER BY name)`

---

## 🔥 4. ON DUPLICATE KEY UPDATE — MySQL's UPSERT

The classic problem: "Insert this row, but if it already exists, update it instead."

### The Problem Without UPSERT

```sql
-- Approach 1: Check first, then INSERT or UPDATE
-- ❌ Race condition! Another thread could INSERT between your SELECT and INSERT
SELECT COUNT(*) FROM products WHERE sku = 'IPH15';
-- If 0: INSERT INTO products ...
-- If 1: UPDATE products SET ...

-- Approach 2: Use REPLACE (MySQL-specific)
-- ❌ DELETE + INSERT internally — loses auto-increment, fires DELETE triggers
REPLACE INTO products (sku, name, price) VALUES ('IPH15', 'iPhone 15', 79999);
```

### The Solution: ON DUPLICATE KEY UPDATE

```sql
-- INSERT ... ON DUPLICATE KEY UPDATE
-- If PK or UNIQUE key conflict → UPDATE instead of error

INSERT INTO products (sku, name, price, stock)
VALUES ('IPH15', 'iPhone 15', 79999, 50)
ON DUPLICATE KEY UPDATE
    price = VALUES(price),          -- Update with the new value
    stock = stock + VALUES(stock);  -- Add to existing stock

-- What happens:
-- Case 1: SKU 'IPH15' doesn't exist → INSERT new row
-- Case 2: SKU 'IPH15' exists → UPDATE price and add stock
-- All in ONE atomic operation! No race conditions! ✅
```

### VALUES() vs New Alias Syntax (MySQL 8.0.19+)

```sql
-- OLD syntax (VALUES() function — deprecated in 8.0.20)
INSERT INTO products (sku, name, price)
VALUES ('IPH15', 'iPhone 15', 79999)
ON DUPLICATE KEY UPDATE
    name = VALUES(name),       -- VALUES() refers to the INSERT values
    price = VALUES(price);

-- NEW syntax (MySQL 8.0.19+ — use row alias)
INSERT INTO products (sku, name, price)
VALUES ('IPH15', 'iPhone 15', 79999) AS new_product
ON DUPLICATE KEY UPDATE
    name = new_product.name,           -- Clearer!
    price = new_product.price;

-- NEW syntax with column aliases too
INSERT INTO products (sku, name, price)
VALUES ('IPH15', 'iPhone 15', 79999) AS new(s, n, p)
ON DUPLICATE KEY UPDATE
    name = n,
    price = p;
```

### Bulk UPSERT

```sql
-- Bulk insert/update in a single statement
INSERT INTO products (sku, name, price, stock)
VALUES 
    ('IPH15', 'iPhone 15', 79999, 50),
    ('SGS24', 'Samsung Galaxy S24', 69999, 75),
    ('MBP3', 'MacBook Pro M3', 199999, 25)
ON DUPLICATE KEY UPDATE
    price = VALUES(price),
    stock = VALUES(stock);

-- Affected rows:
-- 0 = row existed and no values changed
-- 1 = new row inserted
-- 2 = existing row updated (MySQL reports 2 for updates, confusingly)
```

### Conditional UPSERT

```sql
-- Only update price if new price is higher
INSERT INTO products (sku, name, price)
VALUES ('IPH15', 'iPhone 15', 84999)
ON DUPLICATE KEY UPDATE
    price = GREATEST(price, VALUES(price));    -- Keep the higher price

-- Only update if last_updated is newer
INSERT INTO inventory (sku, quantity, last_updated)
VALUES ('IPH15', 100, '2024-06-01')
ON DUPLICATE KEY UPDATE
    quantity = IF(VALUES(last_updated) > last_updated, VALUES(quantity), quantity),
    last_updated = IF(VALUES(last_updated) > last_updated, VALUES(last_updated), last_updated);
```

> ⭐ **UPSERT in Other Databases:**
> - **PostgreSQL:** `INSERT ... ON CONFLICT (col) DO UPDATE SET ...` ← most flexible
> - **SQL Server:** `MERGE ... WHEN MATCHED THEN UPDATE WHEN NOT MATCHED THEN INSERT`
> - **Oracle:** `MERGE INTO ... USING ... ON (...) WHEN MATCHED/NOT MATCHED`
> - **SQLite:** `INSERT ... ON CONFLICT DO UPDATE SET ...`

---

## 🔥 5. MySQL-Specific Functions — The Power Toolkit

### 5.1 String Functions

```sql
-- CONCAT — concatenate strings (NULL-safe!)
SELECT CONCAT('Hello', ' ', 'World');          -- 'Hello World'
SELECT CONCAT('Hello', NULL, 'World');          -- NULL! (any NULL → result is NULL)
SELECT CONCAT_WS(', ', 'a', 'b', NULL, 'c');   -- 'a, b, c' (WS = With Separator, skips NULL!)

-- LENGTH vs CHAR_LENGTH
SELECT LENGTH('Hello');                -- 5 (bytes)
SELECT CHAR_LENGTH('Hello');           -- 5 (characters)
SELECT LENGTH('🚀');                   -- 4 (bytes — emoji is 4 bytes in utf8mb4)
SELECT CHAR_LENGTH('🚀');              -- 1 (character)
-- ⚠️ Always use CHAR_LENGTH for character count in utf8mb4!

-- SUBSTRING / SUBSTR
SELECT SUBSTRING('Hello World', 7);           -- 'World' (from position 7)
SELECT SUBSTRING('Hello World', 1, 5);        -- 'Hello' (position 1, length 5)
SELECT SUBSTRING('Hello World', -5);          -- 'World' (last 5 chars — MySQL-specific!)

-- LEFT / RIGHT
SELECT LEFT('Hello World', 5);                -- 'Hello'
SELECT RIGHT('Hello World', 5);               -- 'World'

-- REPLACE
SELECT REPLACE('Hello World', 'World', 'MySQL');  -- 'Hello MySQL'

-- REVERSE
SELECT REVERSE('Hello');                       -- 'olleH'

-- UPPER / LOWER / LCASE / UCASE
SELECT UPPER('hello');                         -- 'HELLO'
SELECT LOWER('HELLO');                         -- 'hello'

-- TRIM / LTRIM / RTRIM
SELECT TRIM('  hello  ');                      -- 'hello'
SELECT TRIM(LEADING '0' FROM '00042');         -- '42'
SELECT TRIM(BOTH 'x' FROM 'xxxhelloxxx');      -- 'hello'

-- LPAD / RPAD (padding)
SELECT LPAD(42, 5, '0');                       -- '00042'
SELECT RPAD('Hi', 10, '.');                    -- 'Hi........'

-- LOCATE / INSTR / POSITION (find substring position)
SELECT LOCATE('World', 'Hello World');         -- 7
SELECT INSTR('Hello World', 'World');          -- 7

-- FORMAT (number formatting with locale)
SELECT FORMAT(1234567.89, 2);                  -- '1,234,567.89'
SELECT FORMAT(1234567.89, 2, 'de_DE');         -- '1.234.567,89' (German locale)

-- REPEAT
SELECT REPEAT('Ha', 3);                        -- 'HaHaHa'

-- FIELD (find position in a list)
SELECT FIELD('b', 'a', 'b', 'c');              -- 2 (position of 'b' in list)
-- Useful for custom sorting:
SELECT * FROM products ORDER BY FIELD(category, 'Phones', 'Laptops', 'Audio');
```

### 5.2 Date & Time Functions

```sql
-- CURRENT DATE/TIME
SELECT NOW();                     -- '2024-06-02 14:30:00' (datetime)
SELECT CURDATE();                 -- '2024-06-02' (date only)
SELECT CURTIME();                 -- '14:30:00' (time only)
SELECT CURRENT_TIMESTAMP();       -- same as NOW()
SELECT UNIX_TIMESTAMP();          -- 1717331400 (epoch seconds)

-- DATE ARITHMETIC
SELECT DATE_ADD('2024-01-15', INTERVAL 30 DAY);     -- '2024-02-14'
SELECT DATE_ADD('2024-01-15', INTERVAL 3 MONTH);     -- '2024-04-15'
SELECT DATE_SUB(NOW(), INTERVAL 1 YEAR);              -- 1 year ago
SELECT '2024-06-02' + INTERVAL 7 DAY;                 -- '2024-06-09' (shorthand)

-- DATEDIFF — difference in DAYS
SELECT DATEDIFF('2024-12-31', '2024-01-01');          -- 365

-- TIMESTAMPDIFF — difference in any unit
SELECT TIMESTAMPDIFF(YEAR, '1995-03-15', NOW());      -- Age in years
SELECT TIMESTAMPDIFF(HOUR, '2024-06-01 08:00:00', NOW());  -- Hours difference
SELECT TIMESTAMPDIFF(MINUTE, order_time, delivery_time) AS delivery_mins FROM orders;

-- EXTRACT parts
SELECT YEAR('2024-06-02');        -- 2024
SELECT MONTH('2024-06-02');       -- 6
SELECT DAY('2024-06-02');         -- 2
SELECT HOUR(NOW());               -- 14
SELECT DAYNAME('2024-06-02');     -- 'Sunday'
SELECT MONTHNAME('2024-06-02');   -- 'June'
SELECT DAYOFWEEK('2024-06-02');   -- 1 (Sunday=1 in MySQL!)
SELECT DAYOFYEAR('2024-06-02');   -- 154
SELECT WEEK('2024-06-02');        -- 22
SELECT QUARTER('2024-06-02');     -- 2

-- DATE_FORMAT — format dates (MySQL-specific format codes)
SELECT DATE_FORMAT(NOW(), '%d-%m-%Y');               -- '02-06-2024'
SELECT DATE_FORMAT(NOW(), '%W, %M %d, %Y');          -- 'Sunday, June 02, 2024'
SELECT DATE_FORMAT(NOW(), '%H:%i:%s');               -- '14:30:00'
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');      -- '2024-06-02 14:30:00'
SELECT DATE_FORMAT(NOW(), '%d %b %Y %h:%i %p');      -- '02 Jun 2024 02:30 PM'

-- STR_TO_DATE — parse string to date
SELECT STR_TO_DATE('02-06-2024', '%d-%m-%Y');        -- '2024-06-02'
SELECT STR_TO_DATE('June 2, 2024', '%M %d, %Y');     -- '2024-06-02'

-- LAST_DAY — last day of the month
SELECT LAST_DAY('2024-02-15');                        -- '2024-02-29' (leap year!)
SELECT LAST_DAY(NOW());                               -- Last day of current month
```

```
┌──────────────────────────────────────────────────────────────┐
│  MySQL DATE_FORMAT Codes (Most Used)                         │
├───────┬──────────────────────────────────────────────────────┤
│  %Y   │  4-digit year (2024)                                │
│  %y   │  2-digit year (24)                                  │
│  %m   │  Month 01-12                                        │
│  %d   │  Day 01-31                                          │
│  %H   │  Hour 00-23 (24-hour)                               │
│  %h   │  Hour 01-12 (12-hour)                               │
│  %i   │  Minutes 00-59                                      │
│  %s   │  Seconds 00-59                                      │
│  %p   │  AM or PM                                           │
│  %W   │  Full weekday name (Sunday)                         │
│  %M   │  Full month name (June)                             │
│  %b   │  Abbreviated month (Jun)                            │
│  %a   │  Abbreviated weekday (Sun)                          │
│  %j   │  Day of year 001-366                                │
└───────┴──────────────────────────────────────────────────────┘
```

### 5.3 Numeric Functions

```sql
-- ROUNDING
SELECT ROUND(3.14159, 2);        -- 3.14
SELECT CEIL(3.2);                -- 4 (round up)
SELECT FLOOR(3.9);               -- 3 (round down)
SELECT TRUNCATE(3.14159, 2);     -- 3.14 (chop, don't round)

-- MATH
SELECT ABS(-42);                 -- 42
SELECT MOD(10, 3);               -- 1 (remainder: 10 % 3)
SELECT POWER(2, 10);             -- 1024
SELECT SQRT(144);                -- 12
SELECT LOG(100);                 -- 4.60517 (natural log)
SELECT LOG10(100);               -- 2
SELECT LOG2(1024);               -- 10

-- RANDOM
SELECT RAND();                          -- 0.7321... (0 to 1)
SELECT FLOOR(RAND() * 100) + 1;        -- Random 1-100
SELECT * FROM products ORDER BY RAND() LIMIT 5;  -- 5 random rows
-- ⚠️ ORDER BY RAND() scans ENTIRE table — terrible for large tables!

-- CONVERSION
SELECT CAST('42' AS SIGNED);           -- 42 (string to int)
SELECT CAST(3.14 AS DECIMAL(5,2));     -- 3.14
SELECT CONV(255, 10, 16);              -- 'FF' (decimal to hex)
SELECT CONV('FF', 16, 10);             -- '255' (hex to decimal)
SELECT BIN(255);                       -- '11111111' (to binary)
SELECT OCT(255);                       -- '377' (to octal)
SELECT HEX(255);                       -- 'FF' (to hexadecimal)
```

### 5.4 Control Flow Functions

```sql
-- IF (inline if — like ternary operator)
SELECT name, price,
    IF(price > 50000, 'Premium', 'Standard') AS tier
FROM products;

-- IFNULL (replace NULL with default)
SELECT name, IFNULL(discount, 0) AS discount FROM products;
-- PostgreSQL equivalent: COALESCE(discount, 0)
-- Oracle equivalent: NVL(discount, 0)

-- NULLIF (return NULL if two values are equal)
SELECT NULLIF(10, 10);       -- NULL
SELECT NULLIF(10, 20);       -- 10
-- Useful to prevent division by zero:
SELECT total / NULLIF(count, 0) AS average;  -- Returns NULL instead of error

-- COALESCE (first non-NULL value — ANSI standard)
SELECT COALESCE(phone, email, 'No contact') AS contact FROM users;

-- CASE (the universal swiss-army knife)
SELECT name, price,
    CASE 
        WHEN price >= 100000 THEN 'Premium'
        WHEN price >= 50000  THEN 'Mid-Range'
        WHEN price >= 10000  THEN 'Budget'
        ELSE 'Economy'
    END AS price_tier
FROM products;

-- CASE with simple comparison
SELECT name,
    CASE category
        WHEN 'Phones'  THEN '📱'
        WHEN 'Laptops' THEN '💻'
        WHEN 'Audio'   THEN '🎧'
        ELSE '📦'
    END AS icon
FROM products;

-- ELT and FIELD (MySQL-specific — lookup by position)
SELECT ELT(2, 'Small', 'Medium', 'Large');     -- 'Medium' (2nd element)
SELECT FIELD('Medium', 'Small', 'Medium', 'Large');  -- 2 (position of 'Medium')
```

### 5.5 JSON Functions (MySQL 5.7+)

MySQL has native JSON support — store, query, and manipulate JSON data directly in the database.

```sql
-- Create table with JSON column
CREATE TABLE events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event_type VARCHAR(50),
    payload JSON,                          -- Native JSON type!
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert JSON data
INSERT INTO events (event_type, payload) VALUES 
    ('purchase', '{"user_id": 1, "items": ["iPhone", "AirPods"], "total": 104998}'),
    ('signup', '{"user_id": 2, "source": "google", "referral": "promo2024"}'),
    ('purchase', '{"user_id": 1, "items": ["MacBook"], "total": 199999}');

-- EXTRACT values with -> and ->>
SELECT 
    event_type,
    payload->'$.user_id' AS user_id,              -- Returns JSON value: 1
    payload->>'$.user_id' AS user_id_text,         -- Returns text: "1" (unquoted)
    payload->'$.items' AS items,                   -- Returns: ["iPhone", "AirPods"]
    payload->'$.items[0]' AS first_item            -- Returns: "iPhone"
FROM events;

-- JSON_EXTRACT (same as ->)
SELECT JSON_EXTRACT(payload, '$.total') AS total FROM events;

-- JSON_UNQUOTE (same as ->>)
SELECT JSON_UNQUOTE(JSON_EXTRACT(payload, '$.source')) AS source FROM events;

-- FILTER by JSON values
SELECT * FROM events 
WHERE payload->>'$.user_id' = '1';

-- JSON_CONTAINS
SELECT * FROM events
WHERE JSON_CONTAINS(payload->'$.items', '"iPhone"');  -- Has "iPhone" in items array?

-- JSON_ARRAY_LENGTH
SELECT JSON_LENGTH(payload->'$.items') AS item_count FROM events;

-- CREATE JSON
SELECT JSON_OBJECT('name', 'Ritesh', 'age', 28);     -- {"name": "Ritesh", "age": 28}
SELECT JSON_ARRAY('a', 'b', 'c');                      -- ["a", "b", "c"]

-- MODIFY JSON
UPDATE events 
SET payload = JSON_SET(payload, '$.status', 'completed')     -- Add/update key
WHERE id = 1;

UPDATE events
SET payload = JSON_REMOVE(payload, '$.referral')             -- Remove key
WHERE id = 2;

UPDATE events
SET payload = JSON_ARRAY_APPEND(payload, '$.items', 'Case')  -- Add to array
WHERE id = 1;

-- JSON_TABLE (MySQL 8.0) — convert JSON array to rows!
SELECT e.id, jt.*
FROM events e,
JSON_TABLE(e.payload, '$.items[*]' COLUMNS (
    item_name VARCHAR(100) PATH '$'
)) AS jt;
/*
+----+──────────────+
| id | item_name    |
+----+──────────────+
|  1 | iPhone       |
|  1 | AirPods      |
|  3 | MacBook      |
+----+──────────────+
*/
```

### JSON Indexing (Multi-Valued Index — MySQL 8.0.17+)

```sql
-- You CAN'T directly index a JSON column
-- But you CAN index generated columns from JSON:

-- Method 1: Generated Column + Index
ALTER TABLE events 
ADD COLUMN user_id INT GENERATED ALWAYS AS (payload->>'$.user_id') STORED,
ADD INDEX idx_user_id (user_id);

-- Method 2: Multi-Valued Index (for JSON arrays — MySQL 8.0.17+)
CREATE INDEX idx_items ON events ((CAST(payload->'$.items' AS CHAR(100) ARRAY)));

-- Now you can query with MEMBER OF
SELECT * FROM events 
WHERE 'iPhone' MEMBER OF (payload->'$.items');
```

> 💡 **When to use JSON in MySQL:**
> - Semi-structured data (user preferences, API payloads, metadata)
> - Schema flexibility (columns vary per row)
> - Avoiding excessive joins for nested data
> 
> **When NOT to use JSON:**
> - Data you frequently filter/join/aggregate on → use proper columns
> - Core business entities → use normalized tables
> - JSON is for flexibility, not laziness

---

## 🔥 6. MySQL-Specific SQL Features

### 6.1 INSERT IGNORE — Skip Duplicates Silently

```sql
-- Normal INSERT (throws error on duplicate)
INSERT INTO users (id, name, email) VALUES (1, 'Ritesh', 'r@x.com');
-- ERROR 1062: Duplicate entry '1' for key 'PRIMARY'

-- INSERT IGNORE (silently skips duplicates)
INSERT IGNORE INTO users (id, name, email) VALUES (1, 'Ritesh', 'r@x.com');
-- Query OK, 0 rows affected, 1 warning (silently ignored!)

-- Bulk insert — skip only the duplicates, insert the rest
INSERT IGNORE INTO users (id, name, email) VALUES 
    (1, 'Ritesh', 'r@x.com'),    -- Duplicate → skipped
    (4, 'New User', 'new@x.com'); -- New → inserted
```

> ⚠️ **Caution:** INSERT IGNORE also silences OTHER errors (data truncation, invalid values). Use ON DUPLICATE KEY UPDATE if you want more control.

### 6.2 REPLACE INTO — Delete + Insert

```sql
-- If row exists (by PK/UNIQUE): DELETE old + INSERT new
-- If row doesn't exist: just INSERT
REPLACE INTO products (sku, name, price) 
VALUES ('IPH15', 'iPhone 15 Pro', 89999);

-- ⚠️ PROBLEMS with REPLACE:
-- • Changes AUTO_INCREMENT (new ID generated!)
-- • Fires DELETE + INSERT triggers (not UPDATE trigger)
-- • Foreign key cascades may fire
-- • Prefer ON DUPLICATE KEY UPDATE instead
```

### 6.3 Multiple Table UPDATE & DELETE

```sql
-- UPDATE with JOIN (MySQL syntax — different from ANSI!)
UPDATE orders o
INNER JOIN customers c ON o.customer_id = c.id
SET o.discount = 10
WHERE c.country = 'India';

-- DELETE with JOIN
DELETE o FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'India' AND o.status = 'cancelled';

-- Multi-table DELETE (delete from multiple tables at once!)
DELETE o, oi 
FROM orders o
INNER JOIN order_items oi ON o.id = oi.order_id
WHERE o.status = 'cancelled' AND o.order_date < '2023-01-01';
```

### 6.4 SHOW Commands — MySQL's Swiss Army Knife

```sql
-- SHOW commands are MySQL-specific (not standard SQL!)
SHOW DATABASES;
SHOW TABLES;
SHOW COLUMNS FROM users;               -- or: DESCRIBE users;
SHOW CREATE TABLE users;               -- Full DDL!
SHOW INDEX FROM users;
SHOW TABLE STATUS FROM techmart;
SHOW PROCESSLIST;                       -- Running queries
SHOW FULL PROCESSLIST;                  -- With full query text
SHOW VARIABLES LIKE 'innodb%';         -- Server variables
SHOW STATUS LIKE 'Com_select';          -- Server counters
SHOW WARNINGS;                          -- After a query with warnings
SHOW ERRORS;                            -- After a query with errors
SHOW GRANTS FOR 'appuser'@'localhost';  -- User privileges
SHOW ENGINE INNODB STATUS\G             -- InnoDB diagnostic info
SHOW BINARY LOGS;                       -- Binary log files
SHOW MASTER STATUS;                     -- Replication position
SHOW SLAVE STATUS\G                     -- Replication status
```

### 6.5 Useful MySQL System Variables & Functions

```sql
-- Connection & Session Info
SELECT CONNECTION_ID();                 -- Current connection ID
SELECT CURRENT_USER();                  -- Authenticated user
SELECT DATABASE();                      -- Current database
SELECT VERSION();                       -- MySQL version
SELECT @@hostname;                      -- Server hostname
SELECT @@port;                          -- Server port
SELECT @@datadir;                       -- Data directory path
SELECT @@basedir;                       -- MySQL installation path

-- UUID Generation
SELECT UUID();                          -- 'a1b2c3d4-e5f6-7890-abcd-ef1234567890'
SELECT UUID_SHORT();                    -- 24637540957081601 (64-bit integer, faster)
SELECT BIN_TO_UUID(UUID_TO_BIN(UUID(), 1));  -- Ordered UUID (better for InnoDB!)

-- Misc
SELECT FOUND_ROWS();                    -- Total rows without LIMIT (after SQL_CALC_FOUND_ROWS)
SELECT ROW_COUNT();                     -- Affected rows from last statement
SELECT BENCHMARK(1000000, SHA2('test', 256));  -- Benchmark an expression
SELECT SLEEP(2);                        -- Sleep 2 seconds (for testing)
```

---

## 🔥 7. SQL Mode — MySQL's Personality Switch

SQL mode controls how strictly MySQL validates data. This is **critically important** and a common source of bugs.

```sql
-- Check current SQL mode
SELECT @@sql_mode;

-- MySQL 8.0 default:
-- ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,
-- NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

### Key SQL Modes Explained

```
┌───────────────────────────┬────────────────────────────────────────────┐
│ Mode                      │ What It Does                               │
├───────────────────────────┼────────────────────────────────────────────┤
│                           │                                            │
│ STRICT_TRANS_TABLES ⭐    │ Rejects invalid data with ERROR            │
│                           │ Without it: MySQL silently truncates       │
│                           │ or converts bad data (very dangerous!)     │
│                           │                                            │
│ ONLY_FULL_GROUP_BY ⭐     │ Non-aggregated columns in SELECT must be   │
│                           │ in GROUP BY (standard SQL behavior).       │
│                           │ Without it: MySQL picks random values! 😱  │
│                           │                                            │
│ NO_ZERO_DATE              │ Rejects '0000-00-00' as a date             │
│ NO_ZERO_IN_DATE           │ Rejects '2024-00-15' (zero month/day)      │
│                           │                                            │
│ ERROR_FOR_DIVISION_BY_ZERO│ Division by zero returns error/warning     │
│                           │ Without it: returns NULL silently           │
│                           │                                            │
│ NO_ENGINE_SUBSTITUTION    │ Error if requested engine unavailable       │
│                           │ Without it: silently uses default engine    │
│                           │                                            │
│ ANSI_QUOTES               │ Treat double-quotes as identifier quotes   │
│                           │ (like PostgreSQL/Oracle) instead of strings │
│                           │                                            │
│ PAD_CHAR_TO_FULL_LENGTH   │ CHAR columns return full padded length     │
│                           │ Without it: trailing spaces are trimmed     │
│                           │                                            │
│ NO_AUTO_VALUE_ON_ZERO     │ Don't generate auto-inc for INSERT of 0    │
│                           │ Without it: INSERT id=0 triggers auto-inc  │
│                           │                                            │
└───────────────────────────┴────────────────────────────────────────────┘
```

### The ONLY_FULL_GROUP_BY Horror Story

```sql
-- ❌ WITHOUT ONLY_FULL_GROUP_BY (MySQL picks RANDOM values!)
SET SESSION sql_mode = '';

SELECT customer_id, name, SUM(amount)    -- Which name? RANDOM! 😱
FROM orders 
JOIN customers ON orders.customer_id = customers.id
GROUP BY customer_id;

-- ✅ WITH ONLY_FULL_GROUP_BY (correct, standard SQL)
SET SESSION sql_mode = 'ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES';

SELECT customer_id, name, SUM(amount)    -- ERROR! 'name' not in GROUP BY
FROM orders 
JOIN customers ON orders.customer_id = customers.id
GROUP BY customer_id;
-- FIX: Add 'name' to GROUP BY or use ANY_VALUE(name)
```

### The STRICT_TRANS_TABLES Horror Story

```sql
-- ❌ WITHOUT STRICT MODE:
CREATE TABLE test (name VARCHAR(5));
INSERT INTO test VALUES ('Hello World');    -- String is 11 chars, column is VARCHAR(5)
-- Result: Silently truncated to 'Hello' with just a WARNING! 😱
-- No error! Your data is corrupted and you don't even know!

-- ✅ WITH STRICT MODE:
INSERT INTO test VALUES ('Hello World');    
-- ERROR 1406: Data too long for column 'name' at row 1
-- Now you KNOW there's a problem! ✅
```

> ⭐ **ALWAYS use STRICT_TRANS_TABLES in production.** Non-strict mode is a legacy foot-gun from MySQL's early "be permissive" philosophy. Modern MySQL defaults to strict — don't disable it!

---

## 🔥 8. MySQL vs Others — Quick Syntax Comparison

### Common Syntax Differences

```
┌────────────────────┬─────────────────┬─────────────────┬──────────────────┬─────────────────┐
│ Operation          │ MySQL           │ PostgreSQL      │ SQL Server       │ Oracle           │
├────────────────────┼─────────────────┼─────────────────┼──────────────────┼─────────────────┤
│ Auto-increment     │ AUTO_INCREMENT  │ SERIAL/IDENTITY │ IDENTITY(1,1)    │ SEQUENCE +       │
│                    │                 │                 │                  │ trigger or 12c+  │
│                    │                 │                 │                  │ IDENTITY         │
│                    │                 │                 │                  │                  │
│ String concat      │ CONCAT(a, b)    │ a || b          │ a + b or         │ a || b           │
│                    │                 │                 │ CONCAT(a, b)     │                  │
│                    │                 │                 │                  │                  │
│ Current datetime   │ NOW()           │ NOW()           │ GETDATE()        │ SYSDATE          │
│                    │                 │                 │                  │                  │
│ LIMIT rows         │ LIMIT n         │ LIMIT n         │ TOP n or         │ FETCH FIRST n    │
│                    │                 │                 │ FETCH FIRST n    │ ROWS ONLY (12c+) │
│                    │                 │                 │                  │ or ROWNUM        │
│                    │                 │                 │                  │                  │
│ UPSERT             │ ON DUPLICATE    │ ON CONFLICT     │ MERGE            │ MERGE            │
│                    │ KEY UPDATE      │ DO UPDATE       │                  │                  │
│                    │                 │                 │                  │                  │
│ Group concat       │ GROUP_CONCAT()  │ STRING_AGG()    │ STRING_AGG()     │ LISTAGG()        │
│                    │                 │                 │ (2017+)          │                  │
│                    │                 │                 │                  │                  │
│ Identifier quote   │ `backtick`      │ "double_quote"  │ [brackets]       │ "double_quote"   │
│                    │                 │                 │                  │                  │
│ ISNULL check       │ IFNULL(a, b)    │ COALESCE(a, b)  │ ISNULL(a, b)     │ NVL(a, b)        │
│                    │                 │                 │                  │                  │
│ Boolean type       │ TINYINT(1)      │ BOOLEAN         │ BIT              │ NUMBER(1) or     │
│                    │ (no real bool)  │ (true bool)     │                  │ CHAR(1)          │
│                    │                 │                 │                  │                  │
│ EXPLAIN plan       │ EXPLAIN         │ EXPLAIN ANALYZE │ SET SHOWPLAN_ALL │ EXPLAIN PLAN FOR │
│                    │                 │                 │ or actual plan   │                  │
└────────────────────┴─────────────────┴─────────────────┴──────────────────┴─────────────────┘
```

### MySQL-Only Features

```
┌──────────────────────────────────────────────────────────────┐
│  Things That ONLY MySQL Does (or does differently)           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ✅ LIMIT in UPDATE/DELETE                                     │
│  ✅ INSERT IGNORE                                              │
│  ✅ REPLACE INTO                                               │
│  ✅ ON DUPLICATE KEY UPDATE                                    │
│  ✅ Backtick identifier quoting (`table`)                      │
│  ✅ GROUP_CONCAT() with ORDER BY inside                        │
│  ✅ SHOW commands (SHOW TABLES, SHOW PROCESSLIST, etc.)        │
│  ✅ USE database_name (switch database)                        │
│  ✅ Engine-per-table (InnoDB, MyISAM, Memory)                  │
│  ✅ @user_variables (session variables without DECLARE)        │
│  ✅ \G for vertical output in CLI                              │
│  ✅ HANDLER statement (direct table access, bypasses optimizer)│
│  ✅ LOAD DATA INFILE (ultra-fast bulk loading)                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔥 9. User-Defined Variables — MySQL's Quick Hack

```sql
-- Set a variable (@ prefix)
SET @tax_rate = 0.18;
SET @min_price = 10000;

-- Use in queries
SELECT name, price, price * @tax_rate AS tax 
FROM products 
WHERE price > @min_price;

-- Set from a query result
SELECT @max_price := MAX(price) FROM products;
SELECT * FROM products WHERE price = @max_price;

-- Use in UPDATE
SET @counter = 0;
UPDATE products SET display_order = (@counter := @counter + 1)
ORDER BY name;
-- Assigns sequential display_order: 1, 2, 3, ...

-- ⚠️ WARNING: @variable behavior in complex queries is UNDEFINED!
-- MySQL docs: "the order of evaluation for expressions involving 
-- user variables is undefined." Don't use them in WHERE + SELECT 
-- of the same query. Use window functions (ROW_NUMBER) instead.
```

---

## 🔥 10. LOAD DATA INFILE — Blazing Fast Bulk Import

```sql
-- Load a CSV file into a table (10-100x faster than INSERT!)
LOAD DATA INFILE '/var/lib/mysql-files/products.csv'
INTO TABLE products
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS                    -- Skip header row
(sku, name, @price, @stock)      -- Map columns
SET price = @price * 100,        -- Transform during load!
    stock = IFNULL(@stock, 0),
    created_at = NOW();

-- LOCAL variant (load from client machine)
LOAD DATA LOCAL INFILE 'C:/data/products.csv'
INTO TABLE products
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\r\n'       -- Windows line endings
IGNORE 1 ROWS;

-- Export to CSV
SELECT * FROM products
INTO OUTFILE '/var/lib/mysql-files/export.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n';
```

```
Speed comparison for 1 million rows:
┌─────────────────────────────┬──────────────┐
│ Method                      │ Time         │
├─────────────────────────────┼──────────────┤
│ Individual INSERTs          │ ~15 minutes  │
│ Batch INSERT (1000 per stmt)│ ~30 seconds  │
│ LOAD DATA INFILE            │ ~3 seconds ⚡│
└─────────────────────────────┴──────────────┘
```

> 💡 **Security Note:** `LOAD DATA LOCAL INFILE` is disabled by default in MySQL 8.0 (security risk — client can read any file the MySQL process can access). Enable only if needed: `local_infile = ON` in my.cnf.

---

## 🧠 Quick Recall — Chapter Summary

| Feature | One-Line Summary |
|---------|-----------------|
| AUTO_INCREMENT | Auto-generate sequential IDs — use BIGINT UNSIGNED for large tables |
| LIMIT/OFFSET | MySQL pagination — use keyset pagination for deep pages |
| GROUP_CONCAT | Aggregate rows into comma-separated string — watch the 1024 byte limit! |
| ON DUPLICATE KEY UPDATE | Atomic UPSERT — insert or update in one statement |
| INSERT IGNORE | Skip duplicate rows silently (suppresses other errors too!) |
| REPLACE INTO | Delete + Insert — avoid, use ON DUPLICATE KEY UPDATE instead |
| JSON functions | Native JSON: store, query (`->`, `->>`), modify (`JSON_SET`), index |
| DATE_FORMAT | Format dates with `%Y-%m-%d` codes — MySQL-specific format strings |
| IFNULL / COALESCE | Replace NULL with default value |
| SQL Mode | STRICT_TRANS_TABLES = strict validation — ALWAYS enable in production |
| ONLY_FULL_GROUP_BY | Forces correct GROUP BY — prevents random value selection |
| LOAD DATA INFILE | Fastest bulk import — 10-100x faster than INSERT statements |
| User Variables (@var) | Quick session variables — but behavior is undefined in complex queries |

---

## ❓ Self-Check Questions

1. What happens to AUTO_INCREMENT IDs after DELETE + server restart in MySQL 5.7 vs 8.0?
2. Why is `LIMIT 10 OFFSET 1000000` slow? What's the alternative?
3. What is the default `group_concat_max_len` and what happens when you exceed it?
4. Write an UPSERT statement that inserts a product or updates its price if it already exists.
5. What does STRICT_TRANS_TABLES prevent? What happens without it?
6. What does ONLY_FULL_GROUP_BY enforce? What's the danger of disabling it?
7. How do you extract a value from a JSON column? What's the difference between `->` and `->>`?
8. Compare MySQL's UPSERT syntax with PostgreSQL's and SQL Server's.
9. When would you use LOAD DATA INFILE instead of INSERT statements?
10. Why should you use `utf8mb4` instead of `utf8` in MySQL?

---

## 🎯 Hands-On Lab

```sql
-- ============================================
-- LAB: MySQL Dialect Features in Action
-- ============================================

-- Setup
CREATE DATABASE IF NOT EXISTS mysql_lab;
USE mysql_lab;

-- 1. AUTO_INCREMENT
CREATE TABLE products (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sku VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    category VARCHAR(50),
    price DECIMAL(10,2),
    stock INT DEFAULT 0,
    metadata JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. Bulk INSERT
INSERT INTO products (sku, name, category, price, stock, metadata) VALUES
    ('IPH15', 'iPhone 15', 'Phones', 79999.00, 50, '{"color": "blue", "storage": "256GB"}'),
    ('SGS24', 'Samsung Galaxy S24', 'Phones', 69999.00, 75, '{"color": "black", "storage": "128GB"}'),
    ('MBP3', 'MacBook Pro M3', 'Laptops', 199999.00, 25, '{"ram": "16GB", "storage": "512GB"}'),
    ('DXP15', 'Dell XPS 15', 'Laptops', 149999.00, 40, '{"ram": "32GB", "storage": "1TB"}'),
    ('APP2', 'AirPods Pro 2', 'Audio', 24999.00, 200, '{"color": "white", "anc": true}'),
    ('SNY5', 'Sony WH-1000XM5', 'Audio', 29999.00, 150, '{"color": "silver", "anc": true}');

-- 3. GROUP_CONCAT
SELECT 
    category,
    GROUP_CONCAT(name ORDER BY price DESC SEPARATOR ' | ') AS products,
    COUNT(*) AS count,
    FORMAT(AVG(price), 2) AS avg_price
FROM products
GROUP BY category;

-- 4. ON DUPLICATE KEY UPDATE
INSERT INTO products (sku, name, category, price, stock)
VALUES ('IPH15', 'iPhone 15 Pro', 'Phones', 84999.00, 30)
ON DUPLICATE KEY UPDATE
    name = VALUES(name),
    price = VALUES(price),
    stock = stock + VALUES(stock);    -- Add to existing stock

-- Verify
SELECT * FROM products WHERE sku = 'IPH15';

-- 5. JSON queries
SELECT name, 
    metadata->>'$.color' AS color,
    metadata->>'$.storage' AS storage
FROM products 
WHERE metadata->>'$.storage' IS NOT NULL;

-- 6. Date functions
SELECT name, 
    created_at,
    DATE_FORMAT(created_at, '%d %b %Y %h:%i %p') AS formatted_date,
    TIMESTAMPDIFF(HOUR, created_at, NOW()) AS hours_ago
FROM products;

-- 7. LIMIT with UPDATE
UPDATE products SET price = price * 0.9 
WHERE category = 'Audio'
ORDER BY price DESC
LIMIT 1;    -- Only discount the most expensive audio product

-- 8. User variables
SET @row_num = 0;
SELECT 
    (@row_num := @row_num + 1) AS rank,
    name, price
FROM products
ORDER BY price DESC;

-- Cleanup
-- DROP DATABASE mysql_lab;
```

---

> **Next Chapter** → [2D.4 — MySQL Performance Tuning](./04-MySQL-Performance.md)
