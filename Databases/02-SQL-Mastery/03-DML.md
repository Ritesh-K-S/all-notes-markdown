# 2.3 — DML: Manipulating Data (INSERT, UPDATE, DELETE, MERGE) 🟢⭐

> **"DDL builds the house. DML fills it with life."**

---

## 🧭 What You'll Master in This Chapter

After this chapter, you'll confidently **add, modify, and remove data** across any SQL database — and know the dangerous patterns that cause production outages.

```
CREATE (table exists) → INSERT (data enters) → UPDATE (data changes) → DELETE (data leaves)
         📐                    ✅                     ✏️                     🗑️
```

---

## 📖 What is DML?

**DML = Data Manipulation Language** — the commands that **change the data** inside your tables.

```
┌──────────────────────────────────────────────────────────────────┐
│                       DML Commands                               │
├───────────┬──────────────────────────────────────────────────────┤
│ INSERT    │ Add new rows                                         │
│ UPDATE    │ Modify existing rows                                 │
│ DELETE    │ Remove rows                                          │
│ MERGE     │ Insert OR Update in one statement (UPSERT)           │
└───────────┴──────────────────────────────────────────────────────┘
```

> ⚠️ **DML is transactional.** You can ROLLBACK DML statements (if you haven't committed yet). This is your safety net.

---

## 🔥 1. INSERT — Adding Data

### Insert a Single Row (All Columns)

```sql
-- Must provide values in the EXACT column order of the table
INSERT INTO customers 
VALUES (1, 'Ritesh', 'Singh', 'ritesh@email.com', '+91-9876543210', 'Delhi', 'India', '2024-01-15', TRUE);
```

> ⚠️ **Never use this in production!** If someone adds a column to the table, your INSERT breaks. Always specify columns.

### Insert a Single Row (Specified Columns) — The Right Way

```sql
INSERT INTO customers (first_name, last_name, email, city, country)
VALUES ('Ritesh', 'Singh', 'ritesh@email.com', 'Delhi', 'India');
-- id = auto-generated, phone = NULL, created_at = default, is_active = TRUE (default)
```

### Insert Multiple Rows — Bulk Insert

```sql
-- Standard SQL (MySQL, PostgreSQL, SQL Server 2008+)
INSERT INTO customers (first_name, last_name, email, city, country)
VALUES 
    ('Sarah',  'Connor',  'sarah@email.com',  'NYC',    'USA'),
    ('Akira',  'Tanaka',  'akira@email.com',  'Tokyo',  'Japan'),
    ('Maria',  'Garcia',  'maria@email.com',  'Madrid', 'Spain'),
    ('John',   'Smith',   'john@email.com',   'London', 'UK'),
    ('Priya',  'Sharma',  'priya@email.com',  'Mumbai', 'India');

-- Oracle (pre-12c — uses INSERT ALL)
INSERT ALL
    INTO customers (first_name, last_name, email) VALUES ('Sarah', 'Connor', 'sarah@email.com')
    INTO customers (first_name, last_name, email) VALUES ('Akira', 'Tanaka', 'akira@email.com')
    INTO customers (first_name, last_name, email) VALUES ('Maria', 'Garcia', 'maria@email.com')
SELECT 1 FROM DUAL;

-- Oracle 12c+ supports multi-row VALUES like others
```

> 💡 **Performance Tip:** Multi-row INSERT is **significantly faster** than individual INSERTs. For 10,000 rows, one multi-row INSERT can be 10-50x faster than 10,000 separate INSERTs.

### INSERT ... SELECT — Copy Data from Another Table

```sql
-- Copy all Indian customers to a separate table
INSERT INTO indian_customers (name, email, city)
SELECT first_name || ' ' || last_name, email, city
FROM customers
WHERE country = 'India';

-- Archive old orders
INSERT INTO orders_archive 
SELECT * FROM orders 
WHERE order_date < '2023-01-01';
```

### INSERT with Returning (Get Auto-Generated Values)

```sql
-- PostgreSQL: RETURNING clause
INSERT INTO customers (first_name, last_name, email)
VALUES ('New', 'Customer', 'new@email.com')
RETURNING id, created_at;
-- Returns: id = 7, created_at = 2024-03-20 14:30:00

-- SQL Server: OUTPUT clause
INSERT INTO customers (first_name, last_name, email)
OUTPUT INSERTED.id, INSERTED.created_at
VALUES ('New', 'Customer', 'new@email.com');

-- MySQL: LAST_INSERT_ID()
INSERT INTO customers (first_name, last_name, email)
VALUES ('New', 'Customer', 'new@email.com');
SELECT LAST_INSERT_ID();    -- Returns the auto-incremented id

-- Oracle: RETURNING INTO (in PL/SQL)
INSERT INTO customers (first_name, last_name, email)
VALUES ('New', 'Customer', 'new@email.com')
RETURNING id INTO v_new_id;
```

---

## 🔥 2. UPDATE — Modifying Existing Data

### Basic UPDATE

```sql
-- Update a single row
UPDATE customers 
SET city = 'Bangalore', 
    country = 'India'
WHERE id = 3;

-- Update with expression
UPDATE products 
SET price = price * 1.10    -- 10% price increase
WHERE category = 'Phones';

-- Update multiple columns
UPDATE orders 
SET status = 'delivered', 
    delivered_date = CURRENT_TIMESTAMP
WHERE id = 42;
```

> ⚠️ **THE #1 PRODUCTION DISASTER — Forgetting WHERE:**
> ```sql
> -- ⚠️⚠️⚠️ THIS UPDATES EVERY ROW IN THE TABLE ⚠️⚠️⚠️
> UPDATE customers SET is_active = FALSE;    -- No WHERE clause!
> -- Congratulations, you just deactivated ALL customers. 🔥
> ```
> 
> **Always write WHERE first, then fill in SET.** Or better yet, run a SELECT with the same WHERE first to verify which rows will be affected.

### The Safe Update Pattern

```sql
-- Step 1: SEE what you'll update (verify the WHERE clause)
SELECT id, name, status FROM orders WHERE status = 'pending' AND order_date < '2024-01-01';

-- Step 2: Count the rows (sanity check)
SELECT COUNT(*) FROM orders WHERE status = 'pending' AND order_date < '2024-01-01';
-- Result: 47 rows — looks reasonable

-- Step 3: BEGIN TRANSACTION (safety net)
BEGIN;    -- or BEGIN TRANSACTION in SQL Server

-- Step 4: UPDATE
UPDATE orders 
SET status = 'cancelled' 
WHERE status = 'pending' AND order_date < '2024-01-01';
-- "47 rows affected" ← matches our count ✅

-- Step 5: Verify
SELECT * FROM orders WHERE status = 'cancelled' AND order_date < '2024-01-01';

-- Step 6: COMMIT or ROLLBACK
COMMIT;     -- If everything looks good
-- ROLLBACK; -- If something went wrong
```

> 💡 **MySQL Safe Mode:** MySQL has `sql_safe_updates` mode that prevents UPDATE/DELETE without WHERE or without using a key column:
> ```sql
> SET sql_safe_updates = 1;
> UPDATE customers SET is_active = FALSE;    -- ERROR! No WHERE clause
> ```

### UPDATE with JOIN (Update Based on Another Table)

```sql
-- MySQL: UPDATE with JOIN
UPDATE products p
JOIN categories c ON p.category_id = c.id
SET p.price = p.price * 0.90     -- 10% discount
WHERE c.name = 'Audio';

-- PostgreSQL: UPDATE with FROM
UPDATE products
SET price = price * 0.90
FROM categories
WHERE products.category_id = categories.id
  AND categories.name = 'Audio';

-- SQL Server: UPDATE with JOIN
UPDATE p
SET p.price = p.price * 0.90
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE c.name = 'Audio';

-- Oracle: UPDATE with subquery
UPDATE products
SET price = price * 0.90
WHERE category_id IN (
    SELECT id FROM categories WHERE name = 'Audio'
);

-- Oracle: MERGE (see MERGE section below for full syntax)
```

### UPDATE with Subquery

```sql
-- Set each customer's order_count based on actual orders
UPDATE customers
SET total_orders = (
    SELECT COUNT(*) 
    FROM orders 
    WHERE orders.customer_id = customers.id
);

-- Update with correlated subquery — set price to category average
UPDATE products p
SET price = (
    SELECT AVG(price) 
    FROM products p2 
    WHERE p2.category_id = p.category_id
)
WHERE price IS NULL;
```

### UPDATE with CASE — Conditional Updates

```sql
-- Different tax rates for different countries
UPDATE customers
SET tax_rate = CASE country
    WHEN 'India' THEN 18.00
    WHEN 'USA'   THEN 8.50
    WHEN 'UK'    THEN 20.00
    WHEN 'Japan' THEN 10.00
    ELSE 15.00
END;

-- Tiered pricing update
UPDATE products
SET discount = CASE 
    WHEN stock > 100 THEN 20     -- Overstocked → big discount
    WHEN stock > 50  THEN 10     -- Moderate stock → small discount
    WHEN stock > 0   THEN 5      -- Low stock → tiny discount
    ELSE 0                       -- Out of stock → no discount
END;
```

---

## 🔥 3. DELETE — Removing Data

### Basic DELETE

```sql
-- Delete specific rows
DELETE FROM orders WHERE status = 'cancelled';

-- Delete one row
DELETE FROM customers WHERE id = 42;
```

> ⚠️ **SAME WARNING AS UPDATE — Forgetting WHERE:**
> ```sql
> DELETE FROM orders;     -- ⚠️ DELETES ALL ROWS. Table structure remains but is now empty.
> ```

### DELETE with JOIN (Delete Based on Related Data)

```sql
-- MySQL: DELETE with JOIN
DELETE o FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'Test';

-- PostgreSQL: DELETE with USING
DELETE FROM orders
USING customers
WHERE orders.customer_id = customers.id
  AND customers.country = 'Test';

-- SQL Server: DELETE with JOIN
DELETE o
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'Test';

-- Standard SQL: DELETE with subquery (works everywhere)
DELETE FROM orders
WHERE customer_id IN (
    SELECT id FROM customers WHERE country = 'Test'
);
```

### DELETE with Subquery

```sql
-- Delete customers who have never placed an order
DELETE FROM customers
WHERE id NOT IN (
    SELECT DISTINCT customer_id FROM orders
);

-- Better (handles NULLs): Use NOT EXISTS
DELETE FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

### DELETE with LIMIT (Batch Deletion)

```sql
-- MySQL: Delete in batches (prevents long locks)
DELETE FROM logs 
WHERE created_at < '2023-01-01'
LIMIT 10000;
-- Run repeatedly until 0 rows affected

-- SQL Server: TOP
DELETE TOP (10000) FROM logs 
WHERE created_at < '2023-01-01';

-- PostgreSQL: ctid trick
DELETE FROM logs
WHERE ctid IN (
    SELECT ctid FROM logs
    WHERE created_at < '2023-01-01'
    LIMIT 10000
);
```

> 💡 **Why batch delete?** Deleting millions of rows in one transaction:
> - Locks the entire table for a long time
> - Fills up the transaction log
> - Can cause replication lag
> - Might time out
> **Solution:** Delete in batches of 10K-100K with small pauses between.

---

## 🔥 4. TRUNCATE vs DELETE — The Full Comparison

```
┌──────────────────┬──────────────────────┬──────────────────────────┐
│ Feature          │ DELETE               │ TRUNCATE                 │
├──────────────────┼──────────────────────┼──────────────────────────┤
│ Rows affected    │ Some or all (WHERE)  │ ALL rows only            │
│ WHERE clause     │ ✅ Yes               │ ❌ No                    │
│ Speed            │ Slow (row-by-row log)│ Fast (deallocates pages) │
│ Transaction log  │ Full logging         │ Minimal logging          │
│ Triggers fired   │ ✅ Yes (row triggers)│ ❌ No                    │
│ Auto-increment   │ Does NOT reset       │ Resets (MySQL, SQL Srv)  │
│ Rollback         │ ✅ Always            │ Depends on DB*           │
│ FK constraints   │ Works with FKs       │ Fails if FK references** │
│ Locks            │ Row locks            │ Table lock               │
│ Space reclaim    │ Doesn't free space*  │ Frees space immediately  │
└──────────────────┴──────────────────────┴──────────────────────────┘
* DELETE doesn't reclaim space in Oracle (need ALTER TABLE SHRINK/MOVE) or 
  PostgreSQL (need VACUUM). SQL Server & MySQL reclaim gradually.
** SQL Server & PostgreSQL prevent TRUNCATE with FK; Oracle allows if child is empty.
```

---

## 🔥 5. MERGE / UPSERT — The Swiss Army Knife

MERGE = **INSERT if not exists, UPDATE if exists** — in a single atomic statement.

### SQL Server / Oracle: MERGE

```sql
MERGE INTO products AS target
USING staging_products AS source
ON target.sku = source.sku
WHEN MATCHED THEN
    UPDATE SET 
        target.price = source.price,
        target.stock = source.stock,
        target.name  = source.name
WHEN NOT MATCHED THEN
    INSERT (sku, name, price, stock, category_id)
    VALUES (source.sku, source.name, source.price, source.stock, source.category_id)
WHEN NOT MATCHED BY SOURCE THEN
    DELETE;    -- Optional: delete target rows not in source (SQL Server only)
```

### PostgreSQL: INSERT ... ON CONFLICT (UPSERT)

```sql
-- Upsert: Insert or Update on conflict
INSERT INTO products (sku, name, price, stock)
VALUES ('IPHONE15', 'iPhone 15', 79999, 50)
ON CONFLICT (sku)
DO UPDATE SET 
    price = EXCLUDED.price,
    stock = EXCLUDED.stock,
    name  = EXCLUDED.name;

-- Upsert: Insert or Do Nothing on conflict
INSERT INTO products (sku, name, price, stock)
VALUES ('IPHONE15', 'iPhone 15', 79999, 50)
ON CONFLICT (sku) DO NOTHING;
```

### MySQL: INSERT ... ON DUPLICATE KEY UPDATE

```sql
INSERT INTO products (sku, name, price, stock)
VALUES ('IPHONE15', 'iPhone 15', 79999, 50)
ON DUPLICATE KEY UPDATE 
    price = VALUES(price),
    stock = VALUES(stock),
    name  = VALUES(name);

-- MySQL 8.0.19+ — use alias
INSERT INTO products (sku, name, price, stock)
VALUES ('IPHONE15', 'iPhone 15', 79999, 50) AS new_vals
ON DUPLICATE KEY UPDATE 
    price = new_vals.price,
    stock = new_vals.stock;

-- MySQL: REPLACE INTO (DELETE + INSERT — be careful!)
REPLACE INTO products (sku, name, price, stock)
VALUES ('IPHONE15', 'iPhone 15', 79999, 50);
-- ⚠️ REPLACE deletes the old row and inserts a new one — triggers ON DELETE, 
--    gets new auto-increment ID, and loses columns not specified!
```

### MERGE — Real-World Use Case: Daily Data Sync

```sql
-- Scenario: Every night, a CSV of product prices arrives.
-- If product exists → update price & stock
-- If product is new → insert it

-- SQL Server / Oracle
MERGE INTO products AS p
USING daily_price_feed AS f
ON p.sku = f.sku
WHEN MATCHED AND (p.price <> f.price OR p.stock <> f.stock) THEN
    UPDATE SET 
        p.price = f.price,
        p.stock = f.stock,
        p.last_updated = CURRENT_TIMESTAMP
WHEN NOT MATCHED THEN
    INSERT (sku, name, price, stock, category_id, last_updated)
    VALUES (f.sku, f.name, f.price, f.stock, f.category_id, CURRENT_TIMESTAMP);
```

---

## 🔥 6. Returning Modified Data

### See What You Changed

```sql
-- PostgreSQL: RETURNING (works with INSERT, UPDATE, DELETE)
UPDATE products SET price = price * 1.10 WHERE category = 'Phones'
RETURNING id, name, price AS new_price;

DELETE FROM orders WHERE status = 'cancelled'
RETURNING *;

-- SQL Server: OUTPUT
UPDATE products SET price = price * 1.10
OUTPUT DELETED.price AS old_price, INSERTED.price AS new_price, INSERTED.name
WHERE category = 'Phones';

DELETE FROM orders
OUTPUT DELETED.*
WHERE status = 'cancelled';

-- Oracle: RETURNING (in PL/SQL or single-row DML)
UPDATE products SET price = price * 1.10
WHERE id = 1
RETURNING price INTO v_new_price;
```

---

## 🔥 7. Transaction Control (TCL) — Your Safety Net

```sql
-- Basic transaction flow
BEGIN;                          -- or BEGIN TRANSACTION (SQL Server)
    INSERT INTO orders (customer_id, amount) VALUES (1, 15000);
    UPDATE products SET stock = stock - 1 WHERE id = 5;
    -- Something went wrong?
    -- ROLLBACK;               -- Undo everything
COMMIT;                        -- Save everything permanently

-- SAVEPOINT — Partial rollback
BEGIN;
    INSERT INTO orders (customer_id, amount) VALUES (1, 15000);
    SAVEPOINT after_order;
    
    UPDATE products SET stock = stock - 1 WHERE id = 5;
    -- Oops, wrong product!
    ROLLBACK TO SAVEPOINT after_order;    -- Undo only the UPDATE
    
    UPDATE products SET stock = stock - 1 WHERE id = 3;    -- Correct product
COMMIT;
```

### Auto-Commit Behavior

```
┌──────────────┬──────────────────────────────────────────────┐
│ Database     │ Auto-Commit Default                          │
├──────────────┼──────────────────────────────────────────────┤
│ MySQL        │ ON (each statement auto-commits)             │
│ PostgreSQL   │ ON (each statement auto-commits)             │
│ SQL Server   │ ON (implicit transactions off by default)    │
│ Oracle       │ OFF (must explicitly COMMIT)                 │
└──────────────┴──────────────────────────────────────────────┘

-- MySQL: Disable auto-commit for explicit transactions
SET autocommit = 0;
-- Or use START TRANSACTION / BEGIN
START TRANSACTION;
    -- your DML here
COMMIT;
```

---

## 🔥 8. Common DML Patterns & Recipes

### Pattern 1: Swap Values Between Two Rows

```sql
-- Swap the prices of products 1 and 2
-- ❌ Naive approach FAILS:
UPDATE products SET price = (SELECT price FROM products WHERE id = 2) WHERE id = 1;
UPDATE products SET price = (SELECT price FROM products WHERE id = 1) WHERE id = 2;
-- After first UPDATE, product 1 already has product 2's price!

-- ✅ Correct: Use a single UPDATE with CASE
UPDATE products
SET price = CASE id
    WHEN 1 THEN (SELECT price FROM products WHERE id = 2)
    WHEN 2 THEN (SELECT price FROM products WHERE id = 1)
END
WHERE id IN (1, 2);

-- ✅ Or use a CTE (PostgreSQL, SQL Server)
WITH swap AS (
    SELECT 
        id,
        CASE id WHEN 1 THEN 2 WHEN 2 THEN 1 END AS swap_id
    FROM products WHERE id IN (1, 2)
)
UPDATE products p
SET price = (SELECT p2.price FROM products p2 WHERE p2.id = s.swap_id)
FROM swap s
WHERE p.id = s.id;
```

### Pattern 2: Delete Duplicates (Keep One)

```sql
-- Keep the lowest id for each email, delete the rest

-- MySQL
DELETE t1 FROM customers t1
INNER JOIN customers t2 
WHERE t1.id > t2.id AND t1.email = t2.email;

-- PostgreSQL
DELETE FROM customers
WHERE id NOT IN (
    SELECT MIN(id) FROM customers GROUP BY email
);

-- SQL Server (using ROW_NUMBER)
WITH cte AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
    FROM customers
)
DELETE FROM cte WHERE rn > 1;

-- Oracle
DELETE FROM customers
WHERE ROWID NOT IN (
    SELECT MIN(ROWID) FROM customers GROUP BY email
);
```

### Pattern 3: Insert If Not Exists

```sql
-- PostgreSQL
INSERT INTO customers (email, name)
VALUES ('test@email.com', 'Test')
ON CONFLICT (email) DO NOTHING;

-- MySQL
INSERT IGNORE INTO customers (email, name)
VALUES ('test@email.com', 'Test');

-- Standard SQL (all databases)
INSERT INTO customers (email, name)
SELECT 'test@email.com', 'Test'
WHERE NOT EXISTS (
    SELECT 1 FROM customers WHERE email = 'test@email.com'
);
```

### Pattern 4: Soft Delete (Don't Actually Delete)

```sql
-- Instead of deleting, mark as inactive
-- This is the standard approach in most production systems

-- Add soft-delete columns
ALTER TABLE customers 
    ADD is_deleted BOOLEAN DEFAULT FALSE,
    ADD deleted_at TIMESTAMP;

-- "Delete" = set flag
UPDATE customers 
SET is_deleted = TRUE, deleted_at = CURRENT_TIMESTAMP 
WHERE id = 42;

-- All queries must filter out deleted records
SELECT * FROM customers WHERE is_deleted = FALSE;

-- Or create a view for convenience
CREATE VIEW active_customers AS
SELECT * FROM customers WHERE is_deleted = FALSE;
```

> 💡 **Why soft delete?**
> - Audit trail — know what was deleted and when
> - Recovery — can "undelete" rows
> - Legal compliance — some data must be retained
> - Referential integrity — no broken foreign keys

---

## 🧠 Common Mistakes with DML

| # | Mistake | Consequence | Prevention |
|---|---------|-------------|------------|
| 1 | UPDATE/DELETE without WHERE | All rows affected | Use `sql_safe_updates`, always preview with SELECT |
| 2 | Not using transactions | Can't undo mistakes | Wrap critical operations in BEGIN/COMMIT |
| 3 | INSERT without column list | Breaks when table changes | Always list columns explicitly |
| 4 | Using REPLACE instead of UPSERT in MySQL | Deletes and re-inserts (new ID!) | Use ON DUPLICATE KEY UPDATE |
| 5 | Bulk DELETE without batching | Long locks, log overflow | Delete in batches of 10K-100K |
| 6 | Not checking row count after DML | Silent failures | Always verify "N rows affected" |
| 7 | Forgetting COMMIT (Oracle) | Data only visible in your session | COMMIT after DML in Oracle |
| 8 | Cascading DELETE surprise | Accidentally deleting child rows | Review FK ON DELETE actions |

---

## ⚔️ Quick Challenge

**Scenario:** You're managing the TechMart database. Execute these operations:

**Q1:** Insert 3 new products into the products table
```sql
INSERT INTO products (name, category, price, stock)
VALUES 
    ('iPad Air', 'Tablets', 59999, 60),
    ('Galaxy Tab S9', 'Tablets', 74999, 45),
    ('Kindle Paperwhite', 'E-Readers', 13999, 200);
```

**Q2:** Give a 15% discount to all products with stock > 100
```sql
UPDATE products 
SET price = price * 0.85 
WHERE stock > 100;
```

**Q3:** Delete all cancelled orders older than 6 months
```sql
DELETE FROM orders 
WHERE status = 'cancelled' 
  AND order_date < CURRENT_DATE - INTERVAL '6 months';     -- PostgreSQL
-- AND order_date < DATE_SUB(CURDATE(), INTERVAL 6 MONTH)  -- MySQL
-- AND order_date < DATEADD(MONTH, -6, GETDATE())          -- SQL Server
-- AND order_date < ADD_MONTHS(SYSDATE, -6)                -- Oracle
```

**Q4:** Upsert — insert a product or update its price if SKU already exists
```sql
-- PostgreSQL
INSERT INTO products (sku, name, price, stock)
VALUES ('IPAD-AIR', 'iPad Air M2', 64999, 50)
ON CONFLICT (sku)
DO UPDATE SET price = EXCLUDED.price, stock = EXCLUDED.stock;
```

---

## 🎯 Key Takeaways

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ✅ Always list columns in INSERT — never rely on column order         │
│  ✅ Multi-row INSERT is much faster than multiple single INSERTs       │
│  ✅ ALWAYS write WHERE before SET in UPDATE — preview with SELECT      │
│  ✅ Use transactions for critical operations — BEGIN, then COMMIT/ROLLBACK│
│  ✅ MERGE/UPSERT = INSERT + UPDATE in one atomic statement             │
│  ✅ TRUNCATE ≠ DELETE — know when to use each                          │
│  ✅ Batch large DELETEs to prevent lock escalation                     │
│  ✅ Consider soft delete over hard delete in production                 │
│  ✅ Use RETURNING/OUTPUT to see what changed                           │
│  ✅ Know your database's auto-commit behavior                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

> **← Previous:** [2.2 DDL — CREATE, ALTER, DROP](./02-DDL.md)
> **Next →** [2.4 JOINs — The Heart of Relational Data](./04-JOINs.md)
