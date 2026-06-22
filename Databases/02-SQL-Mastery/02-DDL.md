# 2.2 — DDL: Creating the World (CREATE, ALTER, DROP) 🟢⭐

> **"Before you can store data, you must build the house. DDL is your blueprint."**

---

## 🧭 What You'll Master in This Chapter

You'll learn to **design and build database structures** — tables, constraints, and relationships — the skeleton on which all data lives.

```
Real World → Database Design → DDL → Living, Breathing Database
     🏢              📐           🔨            🗄️
```

---

## 📖 What is DDL?

**DDL = Data Definition Language** — the SQL commands that define the **structure** (schema) of your database.

```
┌─────────────────────────────────────────────────────────────┐
│                    DDL Commands                              │
├──────────────┬──────────────────────────────────────────────┤
│  CREATE      │  Build new objects (tables, indexes, views)  │
│  ALTER       │  Modify existing objects                     │
│  DROP        │  Destroy objects permanently                 │
│  TRUNCATE    │  Remove ALL rows (keep structure)            │
│  RENAME      │  Rename objects                              │
│  COMMENT     │  Add metadata descriptions                  │
└──────────────┴──────────────────────────────────────────────┘
```

> ⚠️ **Critical:** DDL statements are **auto-committed** in Oracle. In PostgreSQL and SQL Server, they're **transactional** (you can roll them back). MySQL auto-commits DDL.

---

## 🔥 1. CREATE TABLE — Building Your First Table

### Basic Syntax

```sql
CREATE TABLE table_name (
    column_name  data_type  [constraints],
    column_name  data_type  [constraints],
    ...
    [table_level_constraints]
);
```

### Real Example — Building the TechMart Schema

```sql
CREATE TABLE customers (
    id          INT           PRIMARY KEY,
    name        VARCHAR(100)  NOT NULL,
    email       VARCHAR(255)  UNIQUE NOT NULL,
    phone       VARCHAR(20),
    city        VARCHAR(50),
    country     VARCHAR(50)   DEFAULT 'India',
    joined_date DATE          DEFAULT CURRENT_DATE,
    is_active   BOOLEAN       DEFAULT TRUE
);
```

---

## 🔥 2. Data Types — The Building Blocks

### Numeric Types

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        NUMERIC DATA TYPES                                │
├───────────────┬──────────────┬──────────────┬───────────────┬────────────┤
│ Category      │ Oracle       │ SQL Server   │ MySQL         │ PostgreSQL │
├───────────────┼──────────────┼──────────────┼───────────────┼────────────┤
│ Small Int     │ NUMBER(5)    │ SMALLINT     │ SMALLINT      │ SMALLINT   │
│ Integer       │ NUMBER(10)   │ INT          │ INT           │ INTEGER    │
│ Big Integer   │ NUMBER(19)   │ BIGINT       │ BIGINT        │ BIGINT     │
│ Decimal       │ NUMBER(10,2) │ DECIMAL(10,2)│ DECIMAL(10,2) │ NUMERIC(10,2)│
│ Float         │ BINARY_FLOAT │ FLOAT        │ FLOAT         │ REAL       │
│ Double        │ BINARY_DOUBLE│ FLOAT(53)    │ DOUBLE        │ DOUBLE PRECISION│
│ Auto-ID       │ SEQUENCE     │ IDENTITY     │ AUTO_INCREMENT│ SERIAL     │
│ Boolean       │ NUMBER(1)    │ BIT          │ BOOLEAN/TINYINT│ BOOLEAN   │
│ Money         │ NUMBER(19,4) │ MONEY        │ DECIMAL(19,4) │ MONEY      │
└───────────────┴──────────────┴──────────────┴───────────────┴────────────┘
```

> 💡 **Golden Rule:** Use `DECIMAL`/`NUMERIC` for money. **NEVER** use `FLOAT` or `DOUBLE` for money — they have rounding errors!
> ```sql
> -- ❌ WRONG: FLOAT can't represent 0.10 exactly
> CREATE TABLE prices (amount FLOAT);    -- 0.1 + 0.2 might = 0.30000000000000004
>
> -- ✅ CORRECT: DECIMAL is exact
> CREATE TABLE prices (amount DECIMAL(10, 2));    -- 0.10 + 0.20 = 0.30 always
> ```

### String Types

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        STRING DATA TYPES                                 │
├───────────────┬──────────────┬──────────────┬───────────────┬────────────┤
│ Type          │ Oracle       │ SQL Server   │ MySQL         │ PostgreSQL │
├───────────────┼──────────────┼──────────────┼───────────────┼────────────┤
│ Fixed-length  │ CHAR(n)      │ CHAR(n)      │ CHAR(n)       │ CHAR(n)    │
│ Variable      │ VARCHAR2(n)  │ VARCHAR(n)   │ VARCHAR(n)    │ VARCHAR(n) │
│ Unicode Var   │ NVARCHAR2(n) │ NVARCHAR(n)  │ VARCHAR(n)*   │ VARCHAR(n)*│
│ Large Text    │ CLOB         │ VARCHAR(MAX) │ TEXT/LONGTEXT │ TEXT       │
│ Max Length    │ 4000/32767** │ 8000/MAX     │ 65535***      │ 1 GB       │
└───────────────┴──────────────┴──────────────┴───────────────┴────────────┘
   *  PostgreSQL & MySQL handle Unicode natively with proper encoding
   ** Oracle: 4000 default, 32767 with extended MAX_STRING_SIZE
   *** MySQL: VARCHAR max depends on row size limit (65535 bytes total per row)
```

> 💡 **CHAR vs VARCHAR:**
> - `CHAR(10)` → always stores 10 bytes, padded with spaces. Use for **fixed-length** data (country codes, state codes)
> - `VARCHAR(100)` → stores only what's needed + 1-2 bytes overhead. Use for **variable-length** data (names, emails)

### Date & Time Types

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      DATE & TIME DATA TYPES                              │
├───────────────┬──────────────┬──────────────┬───────────────┬────────────┤
│ Type          │ Oracle       │ SQL Server   │ MySQL         │ PostgreSQL │
├───────────────┼──────────────┼──────────────┼───────────────┼────────────┤
│ Date only     │ DATE*        │ DATE         │ DATE          │ DATE       │
│ Time only     │ —            │ TIME         │ TIME          │ TIME       │
│ Date + Time   │ DATE*        │ DATETIME2    │ DATETIME      │ TIMESTAMP  │
│ With TZ       │ TIMESTAMP    │ DATETIMEOFFSET│ TIMESTAMP    │ TIMESTAMPTZ│
│               │ WITH TIME    │              │               │            │
│               │ ZONE         │              │               │            │
└───────────────┴──────────────┴──────────────┴───────────────┴────────────┘
   * Oracle's DATE includes time component (HH:MM:SS)
```

> ⚠️ **Oracle Trap:** Oracle's `DATE` type includes time! `SYSDATE` returns both date AND time. If you compare dates, the time portion can bite you.

### Binary / Large Object Types

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Type          │ Oracle       │ SQL Server   │ MySQL         │ PostgreSQL │
├───────────────┼──────────────┼──────────────┼───────────────┼────────────┤
│ Binary        │ RAW(n)       │ VARBINARY(n) │ VARBINARY(n)  │ BYTEA      │
│ Large Binary  │ BLOB         │ VARBINARY(MAX)│ LONGBLOB     │ BYTEA      │
│ JSON          │ CLOB/JSON*   │ NVARCHAR(MAX)│ JSON          │ JSON/JSONB │
│ XML           │ XMLTYPE      │ XML          │ —             │ XML        │
│ UUID          │ RAW(16)      │ UNIQUEIDENTIFIER│ CHAR(36)   │ UUID       │
└───────────────┴──────────────┴──────────────┴───────────────┴────────────┘
   * Oracle 21c+ has native JSON type
```

---

## 🔥 3. Constraints — The Rules That Protect Your Data

Constraints are the **guardians** of data integrity. They prevent bad data from entering your database.

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE CONSTRAINT HIERARCHY                      │
├────────────────┬────────────────────────────────────────────────┤
│ PRIMARY KEY    │ Unique + NOT NULL. One per table. The ID card. │
│ FOREIGN KEY    │ Links to another table's PK. The relationship. │
│ UNIQUE         │ No duplicates allowed. Multiple per table.     │
│ NOT NULL       │ Must have a value. No emptiness allowed.       │
│ CHECK          │ Custom validation rule.                        │
│ DEFAULT        │ Auto-fill if no value provided.                │
└────────────────┴────────────────────────────────────────────────┘
```

### PRIMARY KEY — The Identity

```sql
-- Column-level (single column PK)
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

-- Table-level (same thing, more explicit)
CREATE TABLE customers (
    id INT,
    name VARCHAR(100),
    CONSTRAINT pk_customers PRIMARY KEY (id)
);

-- Composite Primary Key (multiple columns)
CREATE TABLE order_items (
    order_id    INT,
    product_id  INT,
    quantity    INT,
    CONSTRAINT pk_order_items PRIMARY KEY (order_id, product_id)
);
```

### Auto-Incrementing Primary Keys — Cross-Database

```sql
-- MySQL
CREATE TABLE customers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100)
);

-- PostgreSQL (modern — GENERATED ALWAYS)
CREATE TABLE customers (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100)
);

-- PostgreSQL (legacy — SERIAL)
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

-- SQL Server
CREATE TABLE customers (
    id INT IDENTITY(1, 1) PRIMARY KEY,    -- Start at 1, increment by 1
    name VARCHAR(100)
);

-- Oracle (12c+ — GENERATED ALWAYS)
CREATE TABLE customers (
    id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR2(100)
);

-- Oracle (pre-12c — SEQUENCE + TRIGGER)
CREATE SEQUENCE seq_customers START WITH 1 INCREMENT BY 1;
-- Then use seq_customers.NEXTVAL in INSERT or trigger
```

### FOREIGN KEY — Building Relationships

```sql
CREATE TABLE orders (
    id          INT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date  DATE DEFAULT CURRENT_DATE,
    amount      DECIMAL(10, 2),
    
    -- This column MUST reference a valid customer
    CONSTRAINT fk_orders_customer 
        FOREIGN KEY (customer_id) 
        REFERENCES customers(id)
        ON DELETE CASCADE          -- Delete orders when customer is deleted
        ON UPDATE CASCADE          -- Update FK when customer PK changes
);
```

**Referential Actions (ON DELETE / ON UPDATE):**

```
┌───────────────┬──────────────────────────────────────────────────────┐
│ Action        │ What Happens                                         │
├───────────────┼──────────────────────────────────────────────────────┤
│ CASCADE       │ Delete/update child rows automatically               │
│ SET NULL      │ Set FK column to NULL (column must be nullable)      │
│ SET DEFAULT   │ Set FK column to its DEFAULT value                   │
│ RESTRICT      │ Prevent delete/update if child rows exist            │
│ NO ACTION     │ Same as RESTRICT (checked at statement end)          │
└───────────────┴──────────────────────────────────────────────────────┘
```

> ⭐ **Best Practice:**
> - Use `CASCADE` when children have **no meaning** without parent (order_items → order)
> - Use `RESTRICT` when children are **important records** (orders → customer — don't auto-delete order history!)
> - Use `SET NULL` when relationship is **optional** (employee → manager)

### UNIQUE Constraint

```sql
CREATE TABLE customers (
    id    INT PRIMARY KEY,
    email VARCHAR(255) UNIQUE,                    -- Column-level
    phone VARCHAR(20),
    
    CONSTRAINT uq_customer_phone UNIQUE (phone)   -- Table-level
);

-- Composite UNIQUE (combination must be unique)
CREATE TABLE enrollments (
    student_id INT,
    course_id  INT,
    CONSTRAINT uq_enrollment UNIQUE (student_id, course_id)
);
```

> 💡 **UNIQUE vs PRIMARY KEY:**
> - PK: Only ONE per table, cannot be NULL
> - UNIQUE: Multiple per table, **can be NULL** (only one NULL in SQL Server, multiple NULLs in Oracle/PostgreSQL/MySQL)

### CHECK Constraint

```sql
CREATE TABLE products (
    id       INT PRIMARY KEY,
    name     VARCHAR(100) NOT NULL,
    price    DECIMAL(10, 2) CHECK (price > 0),
    stock    INT DEFAULT 0 CHECK (stock >= 0),
    rating   DECIMAL(2, 1) CHECK (rating BETWEEN 0 AND 5),
    status   VARCHAR(20) CHECK (status IN ('active', 'discontinued', 'draft')),
    
    -- Table-level CHECK (can reference multiple columns)
    CONSTRAINT chk_price_consistency CHECK (price > cost)
);

-- Age validation
CREATE TABLE employees (
    id   INT PRIMARY KEY,
    name VARCHAR(100),
    age  INT,
    CONSTRAINT chk_age CHECK (age >= 18 AND age <= 120)
);
```

> ⚠️ **MySQL Note:** Before MySQL 8.0.16, CHECK constraints were **parsed but ignored**! They were silently accepted but never enforced. Always verify your MySQL version.

### DEFAULT Values

```sql
CREATE TABLE orders (
    id          INT PRIMARY KEY,
    order_date  DATE DEFAULT CURRENT_DATE,
    status      VARCHAR(20) DEFAULT 'pending',
    is_paid     BOOLEAN DEFAULT FALSE,
    tax_rate    DECIMAL(4, 2) DEFAULT 18.00,
    notes       TEXT DEFAULT NULL
);

-- Default with function (PostgreSQL)
CREATE TABLE logs (
    id         UUID DEFAULT gen_random_uuid(),
    created_at TIMESTAMP DEFAULT NOW(),
    message    TEXT
);

-- SQL Server — DEFAULT with NEWID()
CREATE TABLE logs (
    id         UNIQUEIDENTIFIER DEFAULT NEWID(),
    created_at DATETIME2 DEFAULT SYSDATETIME(),
    message    NVARCHAR(MAX)
);
```

### NOT NULL Constraint

```sql
CREATE TABLE customers (
    id    INT PRIMARY KEY,           -- PK implies NOT NULL
    name  VARCHAR(100) NOT NULL,     -- Cannot be empty
    email VARCHAR(255) NOT NULL,     -- Cannot be empty
    phone VARCHAR(20)                -- CAN be NULL (optional)
);
```

> 💡 **Design Philosophy:**
> Make columns NOT NULL **by default**. Only allow NULL when there's a genuine business reason for "unknown/not applicable." NULLs complicate queries, indexing, and aggregations.

---

## 🔥 4. Complete Real-World Schema Example

Let's build the entire TechMart e-commerce schema:

```sql
-- ============================================
-- TechMart E-Commerce Database Schema
-- ============================================

-- 1. CUSTOMERS
CREATE TABLE customers (
    id          INT             PRIMARY KEY AUTO_INCREMENT,  -- MySQL
    first_name  VARCHAR(50)     NOT NULL,
    last_name   VARCHAR(50)     NOT NULL,
    email       VARCHAR(255)    NOT NULL UNIQUE,
    phone       VARCHAR(20),
    city        VARCHAR(50),
    country     VARCHAR(50)     DEFAULT 'India',
    created_at  TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    is_active   BOOLEAN         DEFAULT TRUE,
    
    -- Indexes for frequently searched columns
    INDEX idx_customer_email (email),
    INDEX idx_customer_country (country)
);

-- 2. CATEGORIES
CREATE TABLE categories (
    id          INT             PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(100)    NOT NULL UNIQUE,
    parent_id   INT,
    
    CONSTRAINT fk_category_parent 
        FOREIGN KEY (parent_id) REFERENCES categories(id)
        ON DELETE SET NULL
);

-- 3. PRODUCTS
CREATE TABLE products (
    id          INT             PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(200)    NOT NULL,
    description TEXT,
    category_id INT             NOT NULL,
    price       DECIMAL(10, 2)  NOT NULL CHECK (price > 0),
    cost        DECIMAL(10, 2)  CHECK (cost > 0),
    stock       INT             DEFAULT 0 CHECK (stock >= 0),
    sku         VARCHAR(50)     UNIQUE,
    is_active   BOOLEAN         DEFAULT TRUE,
    created_at  TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT fk_product_category 
        FOREIGN KEY (category_id) REFERENCES categories(id)
        ON DELETE RESTRICT,
    CONSTRAINT chk_price_above_cost CHECK (price >= cost)
);

-- 4. ORDERS
CREATE TABLE orders (
    id          INT             PRIMARY KEY AUTO_INCREMENT,
    customer_id INT             NOT NULL,
    order_date  TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,
    status      VARCHAR(20)     DEFAULT 'pending' 
                                CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')),
    total       DECIMAL(12, 2)  DEFAULT 0,
    shipping_address TEXT,
    
    CONSTRAINT fk_order_customer 
        FOREIGN KEY (customer_id) REFERENCES customers(id)
        ON DELETE RESTRICT
);

-- 5. ORDER_ITEMS (junction/bridge table)
CREATE TABLE order_items (
    id          INT             PRIMARY KEY AUTO_INCREMENT,
    order_id    INT             NOT NULL,
    product_id  INT             NOT NULL,
    quantity    INT             NOT NULL CHECK (quantity > 0),
    unit_price  DECIMAL(10, 2)  NOT NULL,
    
    CONSTRAINT fk_item_order 
        FOREIGN KEY (order_id) REFERENCES orders(id)
        ON DELETE CASCADE,
    CONSTRAINT fk_item_product 
        FOREIGN KEY (product_id) REFERENCES products(id)
        ON DELETE RESTRICT,
    CONSTRAINT uq_order_product UNIQUE (order_id, product_id)
);
```

---

## 🔥 5. ALTER TABLE — Modifying Existing Tables

### Add a Column

```sql
-- Add single column
ALTER TABLE customers ADD loyalty_points INT DEFAULT 0;

-- Add multiple columns (MySQL)
ALTER TABLE customers 
    ADD middle_name VARCHAR(50),
    ADD birth_date DATE;

-- Add with NOT NULL (need DEFAULT for existing rows)
ALTER TABLE customers ADD membership VARCHAR(20) NOT NULL DEFAULT 'basic';

-- PostgreSQL: Add multiple
ALTER TABLE customers 
    ADD COLUMN middle_name VARCHAR(50),
    ADD COLUMN birth_date DATE;

-- Oracle
ALTER TABLE customers ADD (
    middle_name VARCHAR2(50),
    birth_date  DATE
);
```

### Drop a Column

```sql
-- Standard
ALTER TABLE customers DROP COLUMN middle_name;

-- MySQL
ALTER TABLE customers DROP middle_name;

-- Oracle (multiple columns)
ALTER TABLE customers DROP (middle_name, birth_date);

-- SQL Server — drop with constraint
ALTER TABLE customers DROP CONSTRAINT DF_customers_middle;   -- Drop default first
ALTER TABLE customers DROP COLUMN middle_name;               -- Then drop column
```

### Modify/Change a Column

```sql
-- MySQL: MODIFY (change type/constraints) vs CHANGE (also rename)
ALTER TABLE customers MODIFY phone VARCHAR(25);
ALTER TABLE customers CHANGE phone phone_number VARCHAR(25);

-- PostgreSQL
ALTER TABLE customers ALTER COLUMN phone TYPE VARCHAR(25);
ALTER TABLE customers ALTER COLUMN phone SET NOT NULL;
ALTER TABLE customers ALTER COLUMN phone DROP NOT NULL;
ALTER TABLE customers ALTER COLUMN phone SET DEFAULT '+91';

-- SQL Server
ALTER TABLE customers ALTER COLUMN phone VARCHAR(25) NOT NULL;

-- Oracle
ALTER TABLE customers MODIFY (phone VARCHAR2(25) NOT NULL);
```

### Rename

```sql
-- Rename table
ALTER TABLE customers RENAME TO clients;          -- MySQL, PostgreSQL, Oracle
EXEC sp_rename 'customers', 'clients';            -- SQL Server

-- Rename column
ALTER TABLE customers RENAME COLUMN phone TO phone_number;  -- PostgreSQL, Oracle, MySQL 8+
EXEC sp_rename 'customers.phone', 'phone_number', 'COLUMN'; -- SQL Server
```

### Add/Drop Constraints

```sql
-- Add PRIMARY KEY
ALTER TABLE customers ADD CONSTRAINT pk_customers PRIMARY KEY (id);

-- Add FOREIGN KEY
ALTER TABLE orders ADD CONSTRAINT fk_orders_customer 
    FOREIGN KEY (customer_id) REFERENCES customers(id);

-- Add UNIQUE
ALTER TABLE customers ADD CONSTRAINT uq_email UNIQUE (email);

-- Add CHECK
ALTER TABLE products ADD CONSTRAINT chk_price CHECK (price > 0);

-- Drop constraints
ALTER TABLE orders DROP CONSTRAINT fk_orders_customer;
ALTER TABLE customers DROP CONSTRAINT uq_email;

-- MySQL: Drop FOREIGN KEY specifically
ALTER TABLE orders DROP FOREIGN KEY fk_orders_customer;

-- Drop PRIMARY KEY
ALTER TABLE customers DROP PRIMARY KEY;                  -- MySQL
ALTER TABLE customers DROP CONSTRAINT pk_customers;      -- Others
```

---

## 🔥 6. DROP — Destroying Objects

```sql
-- Drop a table
DROP TABLE orders;

-- Drop only if exists (prevents errors)
DROP TABLE IF EXISTS orders;              -- MySQL, PostgreSQL, SQL Server 2016+

-- Oracle (no IF EXISTS, use PL/SQL or exception handling)
BEGIN
    EXECUTE IMMEDIATE 'DROP TABLE orders';
EXCEPTION
    WHEN OTHERS THEN NULL;
END;

-- Drop with CASCADE (also drops dependent objects)
DROP TABLE customers CASCADE;             -- PostgreSQL: drops FKs referencing this table
DROP TABLE customers CASCADE CONSTRAINTS; -- Oracle: drops FK constraints only

-- ⚠️ SQL Server: Must drop FK constraints manually before dropping referenced table!
```

### Drop Order Matters!

```
❌ WRONG ORDER (will fail due to FK constraints):
   DROP TABLE customers;     -- orders.customer_id references this!
   DROP TABLE orders;

✅ CORRECT ORDER (children first, then parents):
   DROP TABLE order_items;   -- depends on orders, products
   DROP TABLE orders;        -- depends on customers
   DROP TABLE products;      -- depends on categories
   DROP TABLE categories;
   DROP TABLE customers;
```

---

## 🔥 7. TRUNCATE vs DROP vs DELETE

```
┌───────────┬────────────┬───────────────┬──────────────┬───────────────┐
│           │ Rows       │ Structure     │ Speed        │ Rollback?     │
├───────────┼────────────┼───────────────┼──────────────┼───────────────┤
│ DELETE    │ Removes    │ Keeps         │ Slow (logged)│ ✅ Yes        │
│ TRUNCATE  │ Removes ALL│ Keeps         │ Fast         │ Depends*      │
│ DROP      │ Removes ALL│ Destroys      │ Fast         │ Depends*      │
└───────────┴────────────┴───────────────┴──────────────┴───────────────┘

* Rollback support:
  - PostgreSQL: TRUNCATE and DROP are transactional ✅
  - SQL Server: TRUNCATE is transactional ✅, DROP is transactional ✅
  - Oracle: TRUNCATE and DROP auto-commit (NOT rollbackable) ❌
  - MySQL: TRUNCATE and DROP auto-commit ❌
```

```sql
-- DELETE (slow, logged, filterable)
DELETE FROM orders WHERE status = 'cancelled';
DELETE FROM orders;          -- All rows, but still logged

-- TRUNCATE (fast, minimal logging, no WHERE)
TRUNCATE TABLE orders;       -- Resets auto-increment in MySQL
                             -- Doesn't fire DELETE triggers

-- DROP (nuclear option)
DROP TABLE orders;           -- Table structure is gone forever
```

> ⚠️ **TRUNCATE gotchas:**
> - Cannot TRUNCATE a table referenced by FK constraints (SQL Server, PostgreSQL)
> - Oracle can TRUNCATE even with FK if child table is empty
> - Resets IDENTITY/AUTO_INCREMENT in MySQL & SQL Server, NOT in PostgreSQL

---

## 🔥 8. CREATE INDEX — Speed Up Your Queries

```sql
-- Basic index
CREATE INDEX idx_customer_country ON customers(country);

-- Unique index
CREATE UNIQUE INDEX idx_customer_email ON customers(email);

-- Composite index (column order matters!)
CREATE INDEX idx_order_date_status ON orders(order_date, status);

-- Drop index
DROP INDEX idx_customer_country;                      -- PostgreSQL, Oracle
DROP INDEX idx_customer_country ON customers;          -- MySQL, SQL Server

-- Conditional/Partial index (PostgreSQL only)
CREATE INDEX idx_active_customers ON customers(email) WHERE is_active = TRUE;
```

> ⭐ **Index Design Rules:**
> 1. Index columns used in WHERE, JOIN, ORDER BY
> 2. Put high-selectivity columns first in composite indexes
> 3. Don't over-index — each index slows down INSERT/UPDATE/DELETE
> 4. Cover your queries when possible (all needed columns in the index)

---

## 🔥 9. Temporary Tables

```sql
-- MySQL
CREATE TEMPORARY TABLE temp_results (
    id INT, total DECIMAL(10,2)
);

-- SQL Server (# prefix = session temp, ## = global temp)
CREATE TABLE #temp_results (
    id INT, total DECIMAL(10,2)
);
CREATE TABLE ##global_temp (
    id INT, total DECIMAL(10,2)
);

-- PostgreSQL
CREATE TEMP TABLE temp_results (
    id INT, total DECIMAL(10,2)
);

-- Oracle (Global Temporary Table — structure persists, data doesn't)
CREATE GLOBAL TEMPORARY TABLE temp_results (
    id NUMBER, total NUMBER(10,2)
) ON COMMIT DELETE ROWS;         -- Data deleted on COMMIT
-- or ON COMMIT PRESERVE ROWS    -- Data deleted when session ends
```

---

## 🔥 10. CREATE TABLE AS SELECT (CTAS) — Clone & Transform

```sql
-- Create a table from query results (copies data AND structure)
CREATE TABLE premium_customers AS
SELECT id, name, email
FROM customers
WHERE loyalty_points > 1000;

-- SQL Server syntax
SELECT id, name, email
INTO premium_customers
FROM customers
WHERE loyalty_points > 1000;

-- Copy structure only (no data)
CREATE TABLE customers_backup AS SELECT * FROM customers WHERE 1 = 0;    -- MySQL/PostgreSQL/Oracle
SELECT * INTO customers_backup FROM customers WHERE 1 = 0;               -- SQL Server

-- PostgreSQL: LIKE (copies constraints, indexes, etc.)
CREATE TABLE customers_backup (LIKE customers INCLUDING ALL);
```

---

## 🧠 Common Mistakes in DDL

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Using FLOAT for money | Rounding errors | Use DECIMAL(10,2) |
| 2 | No foreign keys | Orphan records | Always define FKs |
| 3 | Dropping tables in wrong order | FK violation errors | Drop children first |
| 4 | VARCHAR(255) for everything | Wasted space, poor validation | Size columns appropriately |
| 5 | Forgetting NOT NULL | NULLs everywhere | Default to NOT NULL |
| 6 | Not naming constraints | Hard to modify later | Always name: `CONSTRAINT pk_table_name ...` |
| 7 | CHAR when VARCHAR is better | Wasted storage + padding issues | Use CHAR only for fixed-length data |
| 8 | No indexes on FK columns | Slow JOINs and CASCADE operations | Index all FK columns |

---

## ⚔️ Quick Challenge

**Design a schema for a Library Management System with these rules:**
- Books have ISBN (unique), title, author, price
- Members have a unique member_id, name, email, join_date
- Members can borrow books (track borrow_date, return_date, status)
- A member can't borrow the same book twice at the same time

```sql
-- Solution:
CREATE TABLE books (
    id       INT PRIMARY KEY AUTO_INCREMENT,
    isbn     VARCHAR(17) NOT NULL UNIQUE,
    title    VARCHAR(300) NOT NULL,
    author   VARCHAR(200) NOT NULL,
    price    DECIMAL(8, 2) CHECK (price >= 0),
    copies   INT DEFAULT 1 CHECK (copies >= 0)
);

CREATE TABLE members (
    id        INT PRIMARY KEY AUTO_INCREMENT,
    member_id VARCHAR(20) NOT NULL UNIQUE,
    name      VARCHAR(100) NOT NULL,
    email     VARCHAR(255) NOT NULL UNIQUE,
    join_date DATE DEFAULT (CURRENT_DATE),
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE borrowings (
    id          INT PRIMARY KEY AUTO_INCREMENT,
    member_id   INT NOT NULL,
    book_id     INT NOT NULL,
    borrow_date DATE NOT NULL DEFAULT (CURRENT_DATE),
    due_date    DATE NOT NULL,
    return_date DATE,
    status      VARCHAR(20) DEFAULT 'borrowed'
                CHECK (status IN ('borrowed', 'returned', 'overdue', 'lost')),
    
    CONSTRAINT fk_borrow_member FOREIGN KEY (member_id) REFERENCES members(id),
    CONSTRAINT fk_borrow_book   FOREIGN KEY (book_id) REFERENCES books(id),
    CONSTRAINT uq_active_borrow UNIQUE (member_id, book_id, status)
);
```

---

## 🎯 Key Takeaways

```
┌─────────────────────────────────────────────────────────────────┐
│  ✅ CREATE TABLE = blueprint. Constraints = rules.             │
│  ✅ Always name your constraints explicitly                    │
│  ✅ Use DECIMAL for money, VARCHAR for text, appropriate types │
│  ✅ DEFAULT to NOT NULL — allow NULL only when meaningful      │
│  ✅ Foreign keys maintain referential integrity                │
│  ✅ Know CASCADE vs RESTRICT — choose wisely                   │
│  ✅ ALTER TABLE for changes, DROP for destruction              │
│  ✅ TRUNCATE ≠ DELETE — know the differences                   │
│  ✅ Index FK columns and frequently searched columns           │
│  ✅ Different databases have different DDL syntax — know yours │
└─────────────────────────────────────────────────────────────────┘
```

---

> **← Previous:** [2.1 SQL Basics — SELECT, WHERE, ORDER BY](./01-SQL-Basics.md)
> **Next →** [2.3 DML — Manipulating Data](./03-DML.md)
