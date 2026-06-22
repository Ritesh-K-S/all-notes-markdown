# 2.9 — Views, Materialized Views & Temp Tables 🟡

> **"Views are the windows into your data. Temp tables are the workbenches. Know when to use each."**
> A senior developer doesn't just write queries — they architect layers of data access.

---

## 🎯 What You'll Master

```
✅ Views — virtual tables, security layer, abstraction
✅ Updatable views — when you can INSERT/UPDATE/DELETE through a view
✅ Indexed/Materialized Views — pre-computed, cached results
✅ Temporary Tables — session/transaction-scoped storage
✅ Table Variables (SQL Server) — lightweight temp storage
✅ CTEs vs Temp Tables vs Views — the decision framework
✅ Real-world patterns and when to use what
```

---

## 🪟 Part 1: Views — Virtual Tables

### What is a View?

A view is a **saved query** that behaves like a table. It doesn't store data — it runs the query every time you access it.

```sql
-- Create a view
CREATE VIEW active_customers AS
SELECT 
    c.id, c.name, c.email, c.tier,
    COUNT(o.id) AS order_count,
    SUM(o.amount) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE c.status = 'active'
GROUP BY c.id, c.name, c.email, c.tier;
```

Now use it like a table:

```sql
-- Query the view
SELECT * FROM active_customers WHERE tier = 'Platinum';

-- Join with other tables
SELECT ac.name, p.product_name
FROM active_customers ac
JOIN purchases p ON p.customer_id = ac.id;
```

```
How a view works:

┌────────────────────┐
│  Your Query        │     ┌──────────────────────┐
│  SELECT *          │ ──→ │  View Definition     │ ──→ Database runs the
│  FROM view_name    │     │  (stored SQL query)  │     combined query
│  WHERE tier = 'X'  │     └──────────────────────┘
└────────────────────┘

The database REPLACES the view name with its definition,
then optimizes and runs the combined query.
```

---

### Why Use Views?

| Benefit | Example |
|---------|---------|
| **Simplification** | Hide complex 10-table JOINs behind a simple view name |
| **Security** | Expose only specific columns (hide salary, SSN, etc.) |
| **Abstraction** | Change underlying tables without breaking app queries |
| **Consistency** | Same business logic used everywhere (e.g., "active customer" definition) |
| **Row-Level Security** | Filter rows based on user context |

### Security Example

```sql
-- HR can see everything
CREATE VIEW hr_employee_view AS
SELECT * FROM employees;

-- Managers see their department only
CREATE VIEW manager_employee_view AS
SELECT emp_id, name, title, department, hire_date
FROM employees
WHERE department = CURRENT_USER_DEPARTMENT();
-- Salary hidden! Department filtered!

-- Public directory
CREATE VIEW employee_directory AS
SELECT name, title, department, email
FROM employees WHERE status = 'active';
-- Salary, SSN, address all hidden!
```

---

### View Syntax (All Databases)

```sql
-- CREATE
CREATE VIEW view_name [(column_aliases)] AS
SELECT ...
[WITH CHECK OPTION];  -- Prevents INSERT/UPDATE that would disappear from view

-- CREATE OR REPLACE (update without dropping)
CREATE OR REPLACE VIEW view_name AS
SELECT ...;

-- ALTER (some databases)
ALTER VIEW view_name AS
SELECT ...;

-- DROP
DROP VIEW view_name;
DROP VIEW IF EXISTS view_name;  -- Safe drop

-- See view definition
-- PostgreSQL:
SELECT definition FROM pg_views WHERE viewname = 'view_name';

-- SQL Server:
EXEC sp_helptext 'view_name';

-- MySQL:
SHOW CREATE VIEW view_name;
```

---

### Updatable Views — INSERT/UPDATE/DELETE Through Views

Not all views support modifications. The rules:

```
✅ UPDATABLE (generally) if the view:
   • Selects from a SINGLE base table
   • Includes the PRIMARY KEY
   • No GROUP BY, HAVING, DISTINCT, UNION
   • No aggregate functions (SUM, COUNT, etc.)
   • No subqueries in SELECT list

❌ NOT UPDATABLE if the view:
   • Has JOINs (some databases allow with restrictions)
   • Uses GROUP BY / DISTINCT / UNION
   • Has aggregate functions
   • Contains derived/computed columns
```

```sql
-- Simple updatable view
CREATE VIEW california_customers AS
SELECT id, name, email, state
FROM customers
WHERE state = 'CA';

-- These work!
INSERT INTO california_customers (id, name, email, state)
VALUES (100, 'Alice', 'alice@mail.com', 'CA');

UPDATE california_customers SET email = 'new@mail.com' WHERE id = 100;

DELETE FROM california_customers WHERE id = 100;
```

### WITH CHECK OPTION — The Safety Net

```sql
CREATE VIEW california_customers AS
SELECT id, name, email, state
FROM customers
WHERE state = 'CA'
WITH CHECK OPTION;

-- This FAILS! The row would have state = 'NY', which violates the view's WHERE clause
INSERT INTO california_customers (id, name, email, state)
VALUES (101, 'Bob', 'bob@mail.com', 'NY');
-- ERROR: new row violates check option for view "california_customers"
```

> 💡 `WITH CHECK OPTION` prevents inserting/updating rows that would "disappear" from the view. It enforces the view's filter as a constraint.

---

### INSTEAD OF Triggers (SQL Server, PostgreSQL, Oracle)

Make **any** view updatable — even complex multi-table views:

```sql
-- SQL Server / PostgreSQL
CREATE TRIGGER trg_update_customer_orders
INSTEAD OF UPDATE ON customer_order_view
FOR EACH ROW
BEGIN
    UPDATE customers SET name = NEW.name WHERE id = NEW.customer_id;
    UPDATE orders SET amount = NEW.amount WHERE id = NEW.order_id;
END;
```

---

## 📦 Part 2: Materialized Views — Pre-Computed & Cached

### The Problem with Regular Views

A regular view runs its query **every single time** you access it. For expensive queries (complex JOINs, aggregations over millions of rows), this is slow.

### The Solution: Materialize It

A **Materialized View** (MV) runs the query once and **stores the results physically**. Like a cache.

```
Regular View:              Materialized View:

Query → Run SQL → Result   Query → Read from cache → Result (FAST!)
Query → Run SQL → Result   Query → Read from cache → Result (FAST!)
Query → Run SQL → Result   ⚡ REFRESH → Re-run SQL → Update cache
Query → Run SQL → Result   Query → Read from cache → Result (FAST!)
```

---

### PostgreSQL Materialized Views

```sql
-- Create
CREATE MATERIALIZED VIEW mv_monthly_revenue AS
SELECT 
    DATE_TRUNC('month', order_date) AS month,
    SUM(amount) AS revenue,
    COUNT(*) AS order_count,
    AVG(amount) AS avg_order_value
FROM orders
GROUP BY DATE_TRUNC('month', order_date)
WITH DATA;  -- Populate immediately (WITH NO DATA to create empty)

-- Query it (instant results!)
SELECT * FROM mv_monthly_revenue WHERE month >= '2024-01-01';

-- Refresh (re-compute from source data)
REFRESH MATERIALIZED VIEW mv_monthly_revenue;

-- Refresh concurrently (doesn't lock reads during refresh — needs UNIQUE index!)
CREATE UNIQUE INDEX ON mv_monthly_revenue(month);
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_revenue;

-- Drop
DROP MATERIALIZED VIEW mv_monthly_revenue;
```

### Oracle Materialized Views

```sql
-- Oracle: More powerful — supports automatic refresh!
CREATE MATERIALIZED VIEW mv_monthly_revenue
BUILD IMMEDIATE                           -- Populate now
REFRESH FAST ON COMMIT                    -- Auto-refresh when source changes!
ENABLE QUERY REWRITE                      -- Optimizer can auto-use this MV
AS
SELECT 
    TRUNC(order_date, 'MONTH') AS month,
    SUM(amount) AS revenue,
    COUNT(*) AS order_count
FROM orders
GROUP BY TRUNC(order_date, 'MONTH');
```

Oracle Refresh Options:

| Option | Meaning |
|--------|---------|
| `REFRESH COMPLETE` | Full re-computation |
| `REFRESH FAST` | Incremental (using materialized view logs) |
| `ON COMMIT` | Refresh automatically when source data changes |
| `ON DEMAND` | Refresh only when you ask |
| `START WITH ... NEXT ...` | Scheduled refresh (e.g., every hour) |

### SQL Server — Indexed Views

SQL Server doesn't call them "Materialized Views" — they're **Indexed Views**:

```sql
-- SQL Server: Create an indexed (materialized) view
CREATE VIEW dbo.vw_monthly_revenue
WITH SCHEMABINDING  -- Required for indexed views!
AS
SELECT 
    DATEADD(MONTH, DATEDIFF(MONTH, 0, order_date), 0) AS month,
    SUM(amount) AS revenue,
    COUNT_BIG(*) AS order_count  -- Must use COUNT_BIG, not COUNT
FROM dbo.orders
GROUP BY DATEADD(MONTH, DATEDIFF(MONTH, 0, order_date), 0);
GO

-- Materialize it by creating a unique clustered index
CREATE UNIQUE CLUSTERED INDEX IX_vw_monthly_revenue
ON dbo.vw_monthly_revenue(month);
```

> 💡 SQL Server indexed views **auto-refresh** when source data changes! No manual refresh needed. But they have strict requirements (SCHEMABINDING, no OUTER JOINs, etc.).

### MySQL — No Native MVs

MySQL doesn't have materialized views. Workarounds:

```sql
-- Option 1: Manual table + scheduled refresh
CREATE TABLE mv_monthly_revenue AS
SELECT ... FROM orders GROUP BY ...;

-- Refresh with event scheduler
CREATE EVENT refresh_monthly_revenue
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    TRUNCATE TABLE mv_monthly_revenue;
    INSERT INTO mv_monthly_revenue SELECT ... FROM orders GROUP BY ...;
END;

-- Option 2: Use Flexviews (third-party tool for incremental MVs)
```

---

### When to Use Materialized Views

```
✅ USE MATERIALIZED VIEWS WHEN:
   • Query is expensive and runs frequently
   • Data doesn't change often (or staleness is acceptable)
   • Dashboard/reporting queries
   • Aggregations over large tables
   • Cross-database or complex JOIN results

❌ DON'T USE WHEN:
   • Data must be real-time (unless ON COMMIT refresh)
   • Source data changes very frequently (refresh cost > query cost)
   • Simple queries that are already fast
   • Storage is a major concern
```

---

## 🗂️ Part 3: Temporary Tables — Your Session Workbench

### What Are Temporary Tables?

Temporary tables are real tables that exist **only for the duration of your session or transaction**. They store data on disk (or in memory), can be indexed, and are automatically cleaned up.

### When to Use Them

```
✅ Multi-step data processing (ETL pipelines)
✅ Staging data before final INSERT
✅ Breaking a complex query into debuggable steps
✅ Storing intermediate results referenced multiple times
✅ When CTEs are too slow (no indexes on CTEs!)
✅ When you need data to persist across multiple queries in a session
```

---

### PostgreSQL Temporary Tables

```sql
-- Create temp table (dropped at end of session)
CREATE TEMPORARY TABLE temp_active_orders AS
SELECT * FROM orders WHERE status = 'active';

-- Create temp table (dropped at end of transaction)
CREATE TEMPORARY TABLE temp_calculations (
    id INT,
    result DECIMAL(10,2)
) ON COMMIT DROP;

-- ON COMMIT options:
--   PRESERVE ROWS (default) — keep data after COMMIT
--   DELETE ROWS — truncate after COMMIT
--   DROP — drop entire table after COMMIT

-- Temp tables are in a special schema
SELECT * FROM pg_temp.temp_active_orders;  -- or just temp_active_orders
```

### SQL Server Temporary Tables

```sql
-- LOCAL temp table (# prefix) — visible only in current session
CREATE TABLE #temp_orders (
    order_id INT,
    amount DECIMAL(10,2),
    status VARCHAR(20)
);

INSERT INTO #temp_orders
SELECT order_id, amount, status FROM orders WHERE order_date > '2024-01-01';

-- Add an index (yes, you can index temp tables!)
CREATE INDEX IX_temp_orders_status ON #temp_orders(status);

-- Use it
SELECT * FROM #temp_orders WHERE status = 'pending';

-- It drops automatically when session ends, or manually:
DROP TABLE #temp_orders;


-- GLOBAL temp table (## prefix) — visible to ALL sessions
CREATE TABLE ##global_temp (
    config_key VARCHAR(100),
    config_value VARCHAR(500)
);
-- Drops when the LAST session using it disconnects
```

### MySQL Temporary Tables

```sql
-- MySQL temporary table
CREATE TEMPORARY TABLE temp_summary AS
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;

-- Limitation: Can't reference the same temp table twice in one query!
-- This FAILS in MySQL:
SELECT a.* FROM temp_summary a JOIN temp_summary b ON a.department != b.department;
-- ERROR: Can't reopen table 'temp_summary'
```

### Oracle Temporary Tables

```sql
-- Oracle: Global Temporary Tables (structure is permanent, data is temporary!)
CREATE GLOBAL TEMPORARY TABLE gtt_calculations (
    emp_id NUMBER,
    bonus  NUMBER(10,2)
) ON COMMIT PRESERVE ROWS;  -- or ON COMMIT DELETE ROWS

-- In Oracle 18c+: Private Temporary Tables (like other databases)
CREATE PRIVATE TEMPORARY TABLE ora$ptt_temp AS
SELECT * FROM employees WHERE department = 'Sales';
```

---

### SQL Server Table Variables — Lightweight Alternative

```sql
-- Table variable (lives in memory, scoped to batch)
DECLARE @recent_orders TABLE (
    order_id INT,
    customer_id INT,
    amount DECIMAL(10,2)
);

INSERT INTO @recent_orders
SELECT order_id, customer_id, amount
FROM orders WHERE order_date > '2024-01-01';

SELECT * FROM @recent_orders WHERE amount > 1000;
```

### Temp Table vs Table Variable (SQL Server)

| Feature | #Temp Table | @Table Variable |
|---------|-------------|-----------------|
| **Scope** | Session | Batch/Procedure |
| **Statistics** | ✅ Yes | ❌ No (assumes 1 row!) |
| **Indexes** | ✅ Any | Limited (PK, UNIQUE) |
| **Transaction rollback** | ✅ Participates | ❌ Not rolled back |
| **Best for** | Large datasets (>100 rows) | Small datasets (<100 rows) |
| **Parallelism** | ✅ Supported | ❌ Not supported |
| **Schema changes** | ALTER TABLE works | Cannot ALTER |

> 💡 **Rule of thumb:** Use temp tables for > 100 rows. Use table variables for small lookups.

---

## ⚔️ The Ultimate Comparison

### CTE vs Temp Table vs View vs Materialized View

```
                    ┌─────────────────────────────────────┐
                    │        DECISION FLOWCHART            │
                    └─────────────────────────────────────┘

Need data across multiple queries?
├── YES → Need it permanently?
│         ├── YES → Need real-time data? 
│         │         ├── YES → VIEW
│         │         └── NO  → MATERIALIZED VIEW
│         └── NO  → TEMP TABLE
└── NO  → Need recursion?
          ├── YES → CTE (WITH RECURSIVE)
          └── NO  → Need to reference result multiple times?
                    ├── YES and small result → CTE
                    ├── YES and large result → TEMP TABLE
                    └── NO  → SUBQUERY or CTE
```

| Criteria | CTE | Temp Table | View | Materialized View |
|----------|-----|------------|------|-------------------|
| **Lifetime** | Single query | Session/txn | Permanent | Permanent |
| **Stores data?** | No* | Yes | No | Yes |
| **Indexable?** | No | Yes | Base table indexes | Yes |
| **Updatable?** | N/A | Yes | Sometimes | No (refresh only) |
| **Recursive?** | Yes | No | No | No |
| **Shared across queries?** | No | Yes (session) | Yes (all users) | Yes (all users) |
| **Performance for repeated access** | Re-evaluates | Good (indexed) | Re-evaluates | Best (cached) |
| **Use case** | Readability, recursion | ETL, staging | Abstraction, security | Dashboards, reports |

---

## 🔥 Real-World Patterns

### Pattern 1: Security Layer with Views

```sql
-- Create role-based views
CREATE VIEW v_customer_public AS
SELECT name, city, state FROM customers;         -- For support staff

CREATE VIEW v_customer_finance AS
SELECT name, credit_limit, balance, payment_status 
FROM customers;                                   -- For finance team

-- Grant access per role
GRANT SELECT ON v_customer_public TO support_role;
GRANT SELECT ON v_customer_finance TO finance_role;
-- No one gets direct table access!
```

### Pattern 2: API Versioning with Views

```sql
-- v1: Original API expects this shape
CREATE VIEW api_v1_products AS
SELECT id, name, price, category FROM products;

-- v2: New API adds fields, renames columns
CREATE VIEW api_v2_products AS
SELECT id, name AS product_name, price, category, 
       created_at, sku, weight
FROM products;

-- Old apps use v1, new apps use v2. Same underlying table!
```

### Pattern 3: ETL Staging with Temp Tables

```sql
-- Step 1: Load raw data into temp table
CREATE TEMPORARY TABLE stg_daily_import AS
SELECT * FROM external_source_via_csv;

-- Step 2: Clean and validate
DELETE FROM stg_daily_import WHERE email IS NULL;
UPDATE stg_daily_import SET phone = REGEXP_REPLACE(phone, '[^0-9]', '');

-- Step 3: Merge into production table
INSERT INTO customers (name, email, phone)
SELECT name, email, phone FROM stg_daily_import
ON CONFLICT (email) DO UPDATE SET phone = EXCLUDED.phone;

-- Temp table auto-drops at session end
```

### Pattern 4: Dashboard with Materialized Views

```sql
-- Pre-compute expensive dashboard data
CREATE MATERIALIZED VIEW mv_dashboard_metrics AS
SELECT 
    DATE_TRUNC('day', created_at) AS day,
    COUNT(*) AS signups,
    COUNT(*) FILTER (WHERE plan = 'pro') AS pro_signups,
    SUM(amount) AS revenue
FROM users u
LEFT JOIN payments p ON p.user_id = u.id
GROUP BY DATE_TRUNC('day', created_at);

-- Refresh every hour via cron job / pg_cron
SELECT cron.schedule('0 * * * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_dashboard_metrics');
```

---

## 🧪 Practice Challenges

### Challenge 1: Security View
> Create a view that shows employee names, departments, and years of service — but hides salary and personal details.

<details>
<summary>💡 Solution</summary>

```sql
CREATE VIEW employee_public_directory AS
SELECT 
    name, 
    department, 
    title,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date)) AS years_of_service
FROM employees
WHERE status = 'active';
```
</details>

### Challenge 2: Materialized View Refresh
> Create a materialized view of monthly sales totals and set up concurrent refresh.

<details>
<summary>💡 Solution</summary>

```sql
CREATE MATERIALIZED VIEW mv_monthly_sales AS
SELECT 
    DATE_TRUNC('month', sale_date) AS month,
    product_category,
    SUM(amount) AS total_sales,
    COUNT(*) AS transaction_count
FROM sales
GROUP BY DATE_TRUNC('month', sale_date), product_category;

CREATE UNIQUE INDEX idx_mv_monthly_sales 
ON mv_monthly_sales(month, product_category);

-- Refresh without locking reads
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_sales;
```
</details>

### Challenge 3: Temp Table Pipeline
> Use temp tables to find customers who bought in Q1 but not Q2, and categorize them by spend.

<details>
<summary>💡 Solution</summary>

```sql
CREATE TEMPORARY TABLE q1_buyers AS
SELECT customer_id, SUM(amount) AS q1_spend
FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31'
GROUP BY customer_id;

CREATE TEMPORARY TABLE q2_buyers AS
SELECT DISTINCT customer_id
FROM orders
WHERE order_date BETWEEN '2024-04-01' AND '2024-06-30';

-- Customers who bought in Q1 but churned in Q2
SELECT 
    c.name, q1.q1_spend,
    CASE 
        WHEN q1.q1_spend >= 5000 THEN 'High Value Churn'
        WHEN q1.q1_spend >= 1000 THEN 'Medium Value Churn'
        ELSE 'Low Value Churn'
    END AS churn_category
FROM q1_buyers q1
JOIN customers c ON c.id = q1.customer_id
LEFT JOIN q2_buyers q2 ON q2.customer_id = q1.customer_id
WHERE q2.customer_id IS NULL
ORDER BY q1.q1_spend DESC;
```
</details>

---

## 🎯 Key Takeaways

```
┌─────────────────────────────────────────────────────────────────┐
│  VIEWS & TEMP TABLES CHEAT SHEET                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VIEW:                                                          │
│  • Saved query, acts like a table                               │
│  • No data storage — re-runs every time                         │
│  • Great for: security, abstraction, simplification             │
│  • WITH CHECK OPTION prevents "disappearing" rows               │
│                                                                 │
│  MATERIALIZED VIEW:                                             │
│  • Saved query + cached results                                 │
│  • Must REFRESH to get updated data                             │
│  • Great for: dashboards, reports, expensive aggregations       │
│  • PostgreSQL: REFRESH CONCURRENTLY (needs unique index)        │
│  • Oracle: REFRESH FAST ON COMMIT (auto-update!)                │
│  • SQL Server: Indexed Views (auto-maintained)                  │
│                                                                 │
│  TEMP TABLE:                                                    │
│  • Real table, session-scoped, auto-dropped                     │
│  • Can be indexed for performance                               │
│  • Great for: ETL, staging, multi-step processing               │
│  • SQL Server: #local, ##global, @table_variable                │
│                                                                 │
│  QUICK DECISION:                                                │
│  • One query, readability?  → CTE                               │
│  • Multiple queries, indexed? → Temp Table                      │
│  • Shared, permanent, real-time? → View                         │
│  • Shared, permanent, cached? → Materialized View               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

> 🚀 **Next Up:** [2.10 — Stored Procedures, Functions & Triggers](./10-Procedures-Functions-Triggers.md) — Add procedural logic, automate reactions, and build reusable database programs.

---

*Views, materialized views, and temp tables are the building blocks of a well-architected data layer. Master these, and you'll never again write the same complex query twice.* 🏗️
