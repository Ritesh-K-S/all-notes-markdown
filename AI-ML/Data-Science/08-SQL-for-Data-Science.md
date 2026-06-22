# Chapter 08: SQL for Data Science

## Table of Contents
- [What is SQL for Data Science?](#what-is-sql-for-data-science)
- [Why It Matters](#why-it-matters)
- [SQL Fundamentals Refresher](#sql-fundamentals-refresher)
- [Advanced Joins and Subqueries](#advanced-joins-and-subqueries)
- [Window Functions — The Power Tool](#window-functions--the-power-tool)
- [Common Table Expressions (CTEs)](#common-table-expressions-ctes)
- [Aggregations and Grouping Patterns](#aggregations-and-grouping-patterns)
- [Query Optimization](#query-optimization)
- [SQL for Analytics — Real Patterns](#sql-for-analytics--real-patterns)
- [SQL vs Pandas — When to Use What](#sql-vs-pandas--when-to-use-what)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is SQL for Data Science?

### Simple Explanation

SQL (Structured Query Language) is the language you use to talk to databases. If a database is a giant organized filing cabinet, SQL is how you ask questions like:

- "Show me all customers who spent more than $1000 last month"
- "What's the average order value by country?"
- "Which product category is growing fastest?"

For data science, SQL isn't just about retrieving data — it's about **analyzing** data directly in the database, which is often faster than pulling everything into Python.

### SQL in the Data Science Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                   DATA SCIENCE WORKFLOW                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐ │
│  │  SQL     │    │  Python/  │    │   ML     │    │  Deploy  │ │
│  │  Query   │───▶│  Pandas   │───▶│  Model   │───▶│  & Mon.  │ │
│  │  Data    │    │  Analysis │    │  Train   │    │          │ │
│  └──────────┘    └───────────┘    └──────────┘    └──────────┘ │
│       ▲                                                          │
│       │     Also used for:                                       │
│       ├── Feature engineering (in-database)                      │
│       ├── A/B test analysis                                      │
│       ├── Dashboard/report queries                               │
│       └── Data quality monitoring                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Why It Matters

### The Reality

- **80% of data science work** involves querying databases
- Most companies store their data in SQL databases or SQL-queryable warehouses
- SQL is the **universal language** — every data tool supports it
- Many interview rounds are pure SQL (especially at FAANG companies)
- Complex analysis that takes 50 lines of Pandas can be 10 lines of SQL

### Where SQL is Used in Data Science

| Task | How SQL is Used |
|------|----------------|
| Data extraction | Pull training data for ML models |
| EDA | Quick aggregations and distributions |
| Feature engineering | Create features directly in the warehouse |
| A/B testing | Compare metrics between control and treatment groups |
| Reporting | Power dashboards (Tableau, Looker, Metabase) |
| Data quality | Monitor for anomalies, nulls, duplicates |
| Pipeline transforms | dbt models, warehouse transformations |

> **Important:** A data scientist who can't write efficient SQL is like a carpenter who can't use a saw. It's not optional.

---

## SQL Fundamentals Refresher

### The Building Blocks

```sql
-- SQL query execution order (NOT the order you write it):
-- 1. FROM / JOIN     — Which tables?
-- 2. WHERE           — Filter rows
-- 3. GROUP BY        — Group rows
-- 4. HAVING          — Filter groups
-- 5. SELECT          — Choose columns
-- 6. DISTINCT        — Remove duplicates
-- 7. ORDER BY        — Sort results
-- 8. LIMIT/OFFSET    — Pagination
```

### Essential Query Structure

```sql
-- Basic SELECT with all common clauses
SELECT 
    category,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value,
    MIN(order_date) AS first_order,
    MAX(order_date) AS last_order
FROM orders
WHERE status = 'completed'           -- Filter BEFORE grouping
  AND order_date >= '2024-01-01'
GROUP BY category
HAVING COUNT(*) >= 10                -- Filter AFTER grouping
ORDER BY total_revenue DESC
LIMIT 20;
```

### Data Types You Must Know

| Type | Examples | Use Case |
|------|----------|----------|
| `INTEGER` / `BIGINT` | 1, 42, -7 | IDs, counts |
| `FLOAT` / `DECIMAL(10,2)` | 3.14, 99.99 | Money, measurements |
| `VARCHAR(n)` / `TEXT` | 'hello', 'New York' | Names, descriptions |
| `DATE` | '2024-01-15' | Calendar dates |
| `TIMESTAMP` | '2024-01-15 14:30:00' | Exact moments |
| `BOOLEAN` | TRUE, FALSE | Flags |
| `JSON` / `JSONB` | {"key": "value"} | Semi-structured data |

---

## Advanced Joins and Subqueries

### Join Types Visualized

```
INNER JOIN: Only matching rows from both tables
┌─────────┐   ┌─────────┐
│    A    │   │    B    │
│   ┌─────┼───┼─────┐   │
│   │█████│   │█████│   │     ← Only the overlap
│   └─────┼───┼─────┘   │
└─────────┘   └─────────┘

LEFT JOIN: All rows from A + matching from B
┌─────────┐   ┌─────────┐
│█████████│   │    B    │
│███┌─────┼───┼─────┐   │
│███│█████│   │█████│   │     ← All of A + matching B
│███└─────┼───┼─────┘   │
│█████████│   │         │
└─────────┘   └─────────┘

FULL OUTER JOIN: All rows from both tables
┌─────────┐   ┌─────────┐
│█████████│   │█████████│
│███┌─────┼───┼─────┐███│
│███│█████│   │█████│███│     ← Everything from both
│███└─────┼───┼─────┘███│
│█████████│   │█████████│
└─────────┘   └─────────┘

CROSS JOIN: Every row from A paired with every row from B
A has 3 rows × B has 4 rows = 12 result rows
```

### Practical Join Examples

```sql
-- Find customers who placed orders but never left a review
SELECT DISTINCT c.customer_id, c.name, c.email
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
LEFT JOIN reviews r ON c.customer_id = r.customer_id
WHERE r.review_id IS NULL;  -- No matching review exists

-- Self-join: Find employees who earn more than their manager
SELECT 
    e.name AS employee_name,
    e.salary AS employee_salary,
    m.name AS manager_name,
    m.salary AS manager_salary
FROM employees e
INNER JOIN employees m ON e.manager_id = m.employee_id
WHERE e.salary > m.salary;

-- Anti-join: Products that have NEVER been ordered
SELECT p.product_id, p.name
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE oi.order_item_id IS NULL;
```

### Subqueries — Queries Inside Queries

```sql
-- Scalar subquery: Compare each order to the average
SELECT 
    order_id,
    amount,
    (SELECT AVG(amount) FROM orders) AS avg_amount,
    amount - (SELECT AVG(amount) FROM orders) AS diff_from_avg
FROM orders;

-- Correlated subquery: Most recent order for each customer
SELECT c.customer_id, c.name,
    (SELECT MAX(order_date) 
     FROM orders o 
     WHERE o.customer_id = c.customer_id) AS last_order_date
FROM customers c;

-- EXISTS: Customers who have placed at least one order over $500
SELECT c.customer_id, c.name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id 
    AND o.amount > 500
);

-- IN with subquery: Products in the top-selling category
SELECT product_id, name, price
FROM products
WHERE category_id IN (
    SELECT category_id
    FROM order_items
    GROUP BY category_id
    ORDER BY SUM(quantity) DESC
    LIMIT 1
);
```

---

## Window Functions — The Power Tool

### What Are Window Functions?

Window functions perform calculations **across a set of rows related to the current row** — without collapsing the rows (unlike GROUP BY). They "look through a window" at surrounding rows.

**Analogy:** Imagine you're in a race. GROUP BY tells you "the average race time was 4 minutes." A window function tells you "YOUR time was 3:45, you're ranked 3rd, and you're 15 seconds faster than the person behind you" — all while keeping every runner's row visible.

### Syntax

```sql
function_name() OVER (
    [PARTITION BY column]    -- Reset for each group (optional)
    [ORDER BY column]        -- Define row ordering (optional)
    [ROWS/RANGE frame]       -- Define window size (optional)
)
```

### The Essential Window Functions

#### ROW_NUMBER, RANK, DENSE_RANK

```sql
-- Ranking customers by total spending
SELECT 
    customer_id,
    total_spent,
    ROW_NUMBER() OVER (ORDER BY total_spent DESC) AS row_num,   -- 1, 2, 3, 4, 5
    RANK() OVER (ORDER BY total_spent DESC) AS rank_num,         -- 1, 2, 2, 4, 5 (gaps after ties)
    DENSE_RANK() OVER (ORDER BY total_spent DESC) AS dense_rank  -- 1, 2, 2, 3, 4 (no gaps)
FROM customer_summary;
```

**Difference illustrated:**

| Customer | Spent | ROW_NUMBER | RANK | DENSE_RANK |
|----------|-------|-----------|------|------------|
| Alice | $500 | 1 | 1 | 1 |
| Bob | $400 | 2 | 2 | 2 |
| Charlie | $400 | 3 | 2 | 2 |
| Dave | $300 | 4 | 4 | 3 |
| Eve | $200 | 5 | 5 | 4 |

#### LAG and LEAD — Look at Previous/Next Rows

```sql
-- Month-over-month revenue growth
SELECT 
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month) AS prev_month_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY month) AS revenue_change,
    ROUND(
        (revenue - LAG(revenue, 1) OVER (ORDER BY month)) * 100.0 
        / LAG(revenue, 1) OVER (ORDER BY month), 2
    ) AS pct_growth
FROM monthly_revenue;
```

**Result:**

| month | revenue | prev_month_revenue | revenue_change | pct_growth |
|-------|---------|-------------------|----------------|------------|
| Jan | 100K | NULL | NULL | NULL |
| Feb | 120K | 100K | 20K | 20.00% |
| Mar | 115K | 120K | -5K | -4.17% |
| Apr | 140K | 115K | 25K | 21.74% |

#### Running Totals and Moving Averages

```sql
-- Cumulative sum (running total)
SELECT 
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        ORDER BY order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_revenue,
    
    -- 7-day moving average
    AVG(daily_revenue) OVER (
        ORDER BY order_date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7day,
    
    -- 30-day moving average
    AVG(daily_revenue) OVER (
        ORDER BY order_date 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS moving_avg_30day
FROM daily_sales;
```

#### PARTITION BY — Window Functions Per Group

```sql
-- Top 3 products by revenue IN EACH CATEGORY
WITH ranked_products AS (
    SELECT 
        category,
        product_name,
        revenue,
        ROW_NUMBER() OVER (
            PARTITION BY category          -- Reset numbering for each category
            ORDER BY revenue DESC          -- Rank by revenue within category
        ) AS rank_in_category
    FROM product_sales
)
SELECT * FROM ranked_products
WHERE rank_in_category <= 3;
```

#### FIRST_VALUE, LAST_VALUE, NTH_VALUE

```sql
-- Compare each employee's salary to the highest in their department
SELECT 
    department,
    employee_name,
    salary,
    FIRST_VALUE(employee_name) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS highest_paid_in_dept,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS max_salary_in_dept,
    salary - FIRST_VALUE(salary) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS gap_from_top
FROM employees;
```

#### NTILE — Divide Into Buckets

```sql
-- Divide customers into quartiles by lifetime value
SELECT 
    customer_id,
    lifetime_value,
    NTILE(4) OVER (ORDER BY lifetime_value DESC) AS quartile
    -- quartile 1 = top 25% (VIP), quartile 4 = bottom 25%
FROM customers;
```

### Window Frame Specification

```sql
-- The frame defines which rows the window function "sees"
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW    -- All rows from start to current
ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING             -- 7-row window centered on current
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING     -- Current row to end
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW             -- Last 7 rows (rolling 7-day)
```

```
Frame visualization for "ROWS BETWEEN 2 PRECEDING AND 1 FOLLOWING":

Row 1:  [■ ■]          (only current + 1 following — no preceding available)
Row 2:  [■ ■ ■]        (1 preceding + current + 1 following)
Row 3:  [■ ■ ■ ■]      (2 preceding + current + 1 following) ← full frame
Row 4:    [■ ■ ■ ■]    (2 preceding + current + 1 following)
Row 5:      [■ ■ ■ ■]  (2 preceding + current + 1 following)
Row 6:        [■ ■ ■]  (2 preceding + current — no following)
```

---

## Common Table Expressions (CTEs)

### What Are CTEs?

CTEs are temporary named result sets that make complex queries readable. Think of them as "variables" in SQL — you compute something once, give it a name, and reference it later.

### Basic CTE Syntax

```sql
-- Without CTE: Nested, hard to read
SELECT * FROM (
    SELECT * FROM (
        SELECT customer_id, SUM(amount) AS total
        FROM orders
        GROUP BY customer_id
    ) sub1
    WHERE total > 1000
) sub2
ORDER BY total DESC;

-- With CTE: Clear, readable, maintainable
WITH customer_totals AS (
    SELECT 
        customer_id, 
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY customer_id
),
high_value_customers AS (
    SELECT * 
    FROM customer_totals 
    WHERE total_spent > 1000
)
SELECT * FROM high_value_customers
ORDER BY total_spent DESC;
```

### Multi-Step Analysis with CTEs

```sql
-- Business question: "Which customers are at risk of churning?"
-- Definition: No order in last 90 days, but were active before

WITH customer_activity AS (
    -- Step 1: Calculate each customer's activity metrics
    SELECT 
        customer_id,
        COUNT(*) AS total_orders,
        MAX(order_date) AS last_order_date,
        MIN(order_date) AS first_order_date,
        AVG(amount) AS avg_order_value,
        DATEDIFF('day', MAX(order_date), CURRENT_DATE) AS days_since_last_order
    FROM orders
    GROUP BY customer_id
),
customer_segments AS (
    -- Step 2: Segment customers based on activity
    SELECT 
        *,
        CASE 
            WHEN days_since_last_order <= 30 THEN 'Active'
            WHEN days_since_last_order <= 90 THEN 'At Risk'
            WHEN days_since_last_order <= 180 THEN 'Lapsed'
            ELSE 'Lost'
        END AS segment,
        CASE
            WHEN total_orders >= 10 AND avg_order_value >= 100 THEN 'High Value'
            WHEN total_orders >= 5 THEN 'Medium Value'
            ELSE 'Low Value'
        END AS value_tier
    FROM customer_activity
    WHERE total_orders >= 2  -- Exclude one-time buyers
)
-- Step 3: Focus on high-value customers at risk
SELECT 
    segment,
    value_tier,
    COUNT(*) AS customer_count,
    AVG(avg_order_value) AS avg_order_val,
    AVG(days_since_last_order) AS avg_days_inactive
FROM customer_segments
WHERE segment IN ('At Risk', 'Lapsed')
GROUP BY segment, value_tier
ORDER BY segment, value_tier;
```

### Recursive CTEs

```sql
-- Generate a date series (useful for filling gaps)
WITH RECURSIVE date_series AS (
    -- Base case: starting date
    SELECT DATE('2024-01-01') AS date_val
    UNION ALL
    -- Recursive case: add one day
    SELECT DATE(date_val, '+1 day')
    FROM date_series
    WHERE date_val < '2024-12-31'
)
-- Join with actual data to find days with zero sales
SELECT 
    d.date_val,
    COALESCE(s.daily_revenue, 0) AS revenue
FROM date_series d
LEFT JOIN daily_sales s ON d.date_val = s.sale_date;

-- Organizational hierarchy: Find all reports under a VP
WITH RECURSIVE org_tree AS (
    -- Base: Start with the VP
    SELECT employee_id, name, manager_id, 1 AS level
    FROM employees
    WHERE name = 'Jane Smith'  -- The VP
    
    UNION ALL
    
    -- Recursive: Find all people reporting to someone already in the tree
    SELECT e.employee_id, e.name, e.manager_id, ot.level + 1
    FROM employees e
    INNER JOIN org_tree ot ON e.manager_id = ot.employee_id
)
SELECT * FROM org_tree ORDER BY level, name;
```

---

## Aggregations and Grouping Patterns

### GROUPING SETS, ROLLUP, CUBE

```sql
-- ROLLUP: Hierarchical aggregation (total → country → city)
SELECT 
    COALESCE(country, 'ALL COUNTRIES') AS country,
    COALESCE(city, 'ALL CITIES') AS city,
    SUM(revenue) AS total_revenue,
    COUNT(*) AS order_count
FROM orders
GROUP BY ROLLUP(country, city);

-- Result:
-- USA       | New York  | 50000  | 200
-- USA       | LA        | 30000  | 150
-- USA       | ALL CITIES| 80000  | 350  ← Subtotal for USA
-- UK        | London    | 40000  | 180
-- UK        | ALL CITIES| 40000  | 180  ← Subtotal for UK
-- ALL       | ALL CITIES| 120000 | 530  ← Grand total

-- CUBE: All possible combinations
SELECT 
    category,
    region,
    SUM(revenue) AS total_revenue
FROM sales
GROUP BY CUBE(category, region);
-- Gives: (category, region), (category, ALL), (ALL, region), (ALL, ALL)
```

### Conditional Aggregation (Pivot-like)

```sql
-- Pivot: Monthly revenue by category (without PIVOT keyword)
SELECT 
    category,
    SUM(CASE WHEN MONTH(order_date) = 1 THEN amount ELSE 0 END) AS jan_revenue,
    SUM(CASE WHEN MONTH(order_date) = 2 THEN amount ELSE 0 END) AS feb_revenue,
    SUM(CASE WHEN MONTH(order_date) = 3 THEN amount ELSE 0 END) AS mar_revenue,
    SUM(CASE WHEN MONTH(order_date) = 4 THEN amount ELSE 0 END) AS apr_revenue,
    COUNT(CASE WHEN amount > 100 THEN 1 END) AS high_value_orders,
    COUNT(CASE WHEN amount <= 100 THEN 1 END) AS low_value_orders,
    AVG(CASE WHEN status = 'completed' THEN amount END) AS avg_completed_amount
FROM orders
GROUP BY category;
```

### STRING_AGG / GROUP_CONCAT

```sql
-- Combine multiple values into a single string
SELECT 
    customer_id,
    STRING_AGG(product_name, ', ' ORDER BY order_date) AS products_purchased,
    COUNT(*) AS product_count
FROM orders o
JOIN products p ON o.product_id = p.product_id
GROUP BY customer_id;

-- Result: customer_123 | "Laptop, Mouse, Keyboard, Monitor" | 4
```

---

## Query Optimization

### Understanding Query Plans

```sql
-- Always use EXPLAIN to understand what the database is doing
EXPLAIN ANALYZE
SELECT c.name, SUM(o.amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.name
ORDER BY total_spent DESC
LIMIT 10;

-- Look for:
-- • Seq Scan (bad for large tables — means no index used)
-- • Index Scan (good — using an index)
-- • Hash Join vs Nested Loop (hash = better for large joins)
-- • Rows estimated vs actual (big difference = stale statistics)
```

### Indexing Strategy

```sql
-- Create indexes on columns you filter/join on frequently
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Composite index for queries that filter on multiple columns
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);
-- ↑ This helps: WHERE customer_id = X AND order_date > Y
-- ↑ Column ORDER matters! Most selective column first.

-- Covering index: includes all columns the query needs
CREATE INDEX idx_orders_covering ON orders(customer_id, order_date, amount);
-- ↑ Query can be answered entirely from the index (no table lookup)
```

### Optimization Tips

```sql
-- ❌ BAD: Function on indexed column (prevents index usage)
SELECT * FROM orders WHERE YEAR(order_date) = 2024;

-- ✓ GOOD: Range condition (uses index)
SELECT * FROM orders 
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';

-- ❌ BAD: SELECT * (reads all columns from disk)
SELECT * FROM orders WHERE customer_id = 123;

-- ✓ GOOD: Select only needed columns
SELECT order_id, amount, order_date FROM orders WHERE customer_id = 123;

-- ❌ BAD: NOT IN with subquery (can be slow)
SELECT * FROM customers 
WHERE customer_id NOT IN (SELECT customer_id FROM orders);

-- ✓ GOOD: LEFT JOIN + IS NULL (usually faster)
SELECT c.* FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;

-- ❌ BAD: DISTINCT on large result set
SELECT DISTINCT customer_id FROM orders;

-- ✓ GOOD: GROUP BY (often optimized better)
SELECT customer_id FROM orders GROUP BY customer_id;

-- ❌ BAD: OR across different columns (hard to optimize)
SELECT * FROM orders WHERE customer_id = 5 OR product_id = 10;

-- ✓ GOOD: UNION ALL (each part uses its own index)
SELECT * FROM orders WHERE customer_id = 5
UNION ALL
SELECT * FROM orders WHERE product_id = 10 AND customer_id != 5;
```

### Partitioning

```sql
-- Partition large tables by date for faster queries
CREATE TABLE orders (
    order_id BIGINT,
    order_date DATE,
    amount DECIMAL(10,2)
) PARTITION BY RANGE (order_date);

-- Create monthly partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Now queries with WHERE order_date = '2024-01-15' 
-- only scan the January partition (partition pruning)
```

---

## SQL for Analytics — Real Patterns

### Cohort Analysis

```sql
-- Monthly retention cohort analysis
WITH first_purchase AS (
    -- Step 1: Find each customer's first purchase month
    SELECT 
        customer_id,
        DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders
    GROUP BY customer_id
),
monthly_activity AS (
    -- Step 2: Find which months each customer was active
    SELECT DISTINCT
        o.customer_id,
        fp.cohort_month,
        DATE_TRUNC('month', o.order_date) AS activity_month,
        DATEDIFF('month', fp.cohort_month, DATE_TRUNC('month', o.order_date)) AS months_since_first
    FROM orders o
    JOIN first_purchase fp ON o.customer_id = fp.customer_id
)
-- Step 3: Calculate retention rates
SELECT 
    cohort_month,
    months_since_first,
    COUNT(DISTINCT customer_id) AS active_customers,
    FIRST_VALUE(COUNT(DISTINCT customer_id)) OVER (
        PARTITION BY cohort_month ORDER BY months_since_first
    ) AS cohort_size,
    ROUND(
        COUNT(DISTINCT customer_id) * 100.0 / 
        FIRST_VALUE(COUNT(DISTINCT customer_id)) OVER (
            PARTITION BY cohort_month ORDER BY months_since_first
        ), 1
    ) AS retention_pct
FROM monthly_activity
GROUP BY cohort_month, months_since_first
ORDER BY cohort_month, months_since_first;
```

### Funnel Analysis

```sql
-- E-commerce conversion funnel
WITH funnel AS (
    SELECT
        COUNT(DISTINCT CASE WHEN event = 'page_view' THEN user_id END) AS viewed,
        COUNT(DISTINCT CASE WHEN event = 'add_to_cart' THEN user_id END) AS added_to_cart,
        COUNT(DISTINCT CASE WHEN event = 'checkout_start' THEN user_id END) AS started_checkout,
        COUNT(DISTINCT CASE WHEN event = 'purchase' THEN user_id END) AS purchased
    FROM user_events
    WHERE event_date >= CURRENT_DATE - INTERVAL '30 days'
)
SELECT 
    viewed,
    added_to_cart,
    ROUND(added_to_cart * 100.0 / viewed, 1) AS view_to_cart_pct,
    started_checkout,
    ROUND(started_checkout * 100.0 / added_to_cart, 1) AS cart_to_checkout_pct,
    purchased,
    ROUND(purchased * 100.0 / started_checkout, 1) AS checkout_to_purchase_pct,
    ROUND(purchased * 100.0 / viewed, 1) AS overall_conversion_pct
FROM funnel;
```

### Sessionization

```sql
-- Group user events into sessions (30-minute inactivity gap)
WITH events_with_gap AS (
    SELECT 
        user_id,
        event_timestamp,
        event_type,
        LAG(event_timestamp) OVER (
            PARTITION BY user_id ORDER BY event_timestamp
        ) AS prev_event_time,
        DATEDIFF('minute', 
            LAG(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp),
            event_timestamp
        ) AS minutes_since_last
    FROM user_events
),
session_starts AS (
    SELECT 
        *,
        CASE 
            WHEN minutes_since_last IS NULL OR minutes_since_last > 30 
            THEN 1 ELSE 0 
        END AS is_new_session
    FROM events_with_gap
),
sessions AS (
    SELECT 
        *,
        SUM(is_new_session) OVER (
            PARTITION BY user_id ORDER BY event_timestamp
        ) AS session_id
    FROM session_starts
)
SELECT 
    user_id,
    session_id,
    MIN(event_timestamp) AS session_start,
    MAX(event_timestamp) AS session_end,
    COUNT(*) AS events_in_session,
    DATEDIFF('minute', MIN(event_timestamp), MAX(event_timestamp)) AS session_duration_min
FROM sessions
GROUP BY user_id, session_id;
```

### Year-over-Year Comparison

```sql
-- Compare this year's monthly revenue to last year
WITH monthly_revenue AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        EXTRACT(YEAR FROM order_date) AS year,
        EXTRACT(MONTH FROM order_date) AS month_num,
        SUM(amount) AS revenue
    FROM orders
    WHERE order_date >= DATE_TRUNC('year', CURRENT_DATE - INTERVAL '1 year')
    GROUP BY 1, 2, 3
)
SELECT 
    this_year.month_num,
    this_year.revenue AS revenue_2024,
    last_year.revenue AS revenue_2023,
    this_year.revenue - last_year.revenue AS absolute_change,
    ROUND((this_year.revenue - last_year.revenue) * 100.0 / last_year.revenue, 1) AS yoy_growth_pct
FROM monthly_revenue this_year
LEFT JOIN monthly_revenue last_year 
    ON this_year.month_num = last_year.month_num
    AND this_year.year = last_year.year + 1
WHERE this_year.year = EXTRACT(YEAR FROM CURRENT_DATE)
ORDER BY this_year.month_num;
```

### Customer Lifetime Value (CLV)

```sql
-- Calculate CLV components for each customer
WITH customer_metrics AS (
    SELECT 
        customer_id,
        COUNT(DISTINCT order_id) AS total_orders,
        SUM(amount) AS total_revenue,
        AVG(amount) AS avg_order_value,
        MIN(order_date) AS first_order,
        MAX(order_date) AS last_order,
        DATEDIFF('day', MIN(order_date), MAX(order_date)) AS customer_lifespan_days,
        COUNT(DISTINCT DATE_TRUNC('month', order_date)) AS active_months
    FROM orders
    GROUP BY customer_id
    HAVING COUNT(*) >= 2  -- Need at least 2 orders to calculate frequency
)
SELECT 
    customer_id,
    total_orders,
    total_revenue,
    avg_order_value,
    -- Purchase frequency: orders per month
    ROUND(total_orders * 1.0 / NULLIF(active_months, 0), 2) AS orders_per_month,
    -- Simple CLV estimate: AOV × Frequency × Lifespan
    ROUND(
        avg_order_value * (total_orders * 1.0 / NULLIF(active_months, 0)) * 
        (customer_lifespan_days / 30.0), 2
    ) AS estimated_clv,
    -- Segment
    NTILE(5) OVER (ORDER BY total_revenue) AS value_quintile
FROM customer_metrics
ORDER BY total_revenue DESC;
```

---

## SQL vs Pandas — When to Use What

| Scenario | Use SQL | Use Pandas |
|----------|---------|------------|
| Data lives in a database | ✓ | |
| Dataset fits in memory (<1GB) | | ✓ |
| Complex joins across large tables | ✓ | |
| Quick prototyping and exploration | | ✓ |
| Need to serve dashboards | ✓ | |
| Complex string manipulation | | ✓ |
| ML feature engineering (iterative) | | ✓ |
| Shared analysis (others can query) | ✓ | |
| Time-series resampling | | ✓ |
| Data > 10GB | ✓ (or Spark) | |

### Same Operation in Both

```sql
-- SQL: Average order value by customer segment
SELECT 
    segment,
    AVG(amount) AS avg_order_value,
    COUNT(*) AS order_count
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
GROUP BY segment
HAVING COUNT(*) >= 10;
```

```python
# Pandas equivalent
import pandas as pd

merged = orders.merge(customers, on='customer_id')
result = (merged
    .groupby('segment')
    .agg(avg_order_value=('amount', 'mean'), order_count=('amount', 'count'))
    .query('order_count >= 10')
    .reset_index()
)
```

---

## Common Mistakes

### 1. GROUP BY Errors

```sql
-- ❌ BAD: Selecting non-aggregated column without GROUP BY
SELECT customer_id, name, SUM(amount)
FROM orders
GROUP BY customer_id;
-- Error: 'name' not in GROUP BY (which name for this customer_id?)

-- ✓ GOOD: Include all non-aggregated columns in GROUP BY
SELECT customer_id, name, SUM(amount)
FROM orders
GROUP BY customer_id, name;
```

### 2. NULL Handling

```sql
-- ❌ BAD: NULL comparisons don't work with =
SELECT * FROM customers WHERE phone = NULL;    -- Returns nothing!
SELECT * FROM customers WHERE phone != NULL;   -- Also returns nothing!

-- ✓ GOOD: Use IS NULL / IS NOT NULL
SELECT * FROM customers WHERE phone IS NULL;
SELECT * FROM customers WHERE phone IS NOT NULL;

-- ⚠ TRAP: NULL in aggregations
-- COUNT(*) counts all rows, COUNT(column) excludes NULLs
SELECT 
    COUNT(*) AS total_rows,              -- 100
    COUNT(phone) AS rows_with_phone,     -- 85 (15 NULLs excluded)
    COUNT(DISTINCT phone) AS unique_phones -- 70

-- ⚠ TRAP: NULL in NOT IN
-- If subquery returns any NULL, NOT IN returns empty result!
SELECT * FROM a WHERE id NOT IN (SELECT id FROM b);  -- Breaks if b.id has NULLs

-- ✓ GOOD: Use NOT EXISTS instead
SELECT * FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.id = a.id);
```

### 3. Cartesian Joins (Accidental)

```sql
-- ❌ BAD: Missing join condition creates cartesian product
SELECT * FROM orders, customers;  -- 1M orders × 50K customers = 50 BILLION rows

-- ✓ GOOD: Always specify join condition
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.customer_id;
```

### 4. Division by Zero

```sql
-- ❌ BAD: Will error if total_orders is 0
SELECT revenue / total_orders AS avg_per_order FROM summary;

-- ✓ GOOD: Use NULLIF to convert 0 to NULL (returns NULL instead of error)
SELECT revenue / NULLIF(total_orders, 0) AS avg_per_order FROM summary;
```

### 5. Implicit Type Conversion

```sql
-- ❌ BAD: Comparing string to integer (prevents index usage in some DBs)
SELECT * FROM orders WHERE order_id = '12345';  -- order_id is INTEGER

-- ✓ GOOD: Match types
SELECT * FROM orders WHERE order_id = 12345;
```

---

## Interview Questions

### Easy

**Q1: What's the difference between WHERE and HAVING?**
> WHERE filters rows BEFORE grouping. HAVING filters groups AFTER aggregation. You can't use aggregate functions in WHERE.

**Q2: What's the difference between UNION and UNION ALL?**
> UNION removes duplicates (slower, does a sort). UNION ALL keeps all rows (faster). Use UNION ALL unless you specifically need deduplication.

**Q3: Write a query to find the second highest salary.**
```sql
-- Method 1: Subquery
SELECT MAX(salary) FROM employees 
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Method 2: Window function (more flexible — works for Nth)
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk = 2;
```

### Medium

**Q4: Find customers who made purchases in January 2024 but NOT in February 2024.**
```sql
SELECT DISTINCT customer_id
FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'
AND customer_id NOT IN (
    SELECT customer_id FROM orders
    WHERE order_date BETWEEN '2024-02-01' AND '2024-02-29'
);
```

**Q5: Calculate the running total of daily revenue, resetting each month.**
```sql
SELECT 
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        PARTITION BY DATE_TRUNC('month', order_date)
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS mtd_revenue
FROM daily_sales;
```

### Hard

**Q6: Find the median salary per department.**
```sql
-- Method using PERCENTILE_CONT (PostgreSQL, BigQuery)
SELECT 
    department,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees
GROUP BY department;

-- Method using window functions (works everywhere)
WITH ranked AS (
    SELECT 
        department, salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary) AS rn,
        COUNT(*) OVER (PARTITION BY department) AS cnt
    FROM employees
)
SELECT 
    department,
    AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (FLOOR((cnt + 1) / 2.0), CEIL((cnt + 1) / 2.0))
GROUP BY department;
```

**Q7: Write a query to detect gaps in sequential order IDs.**
```sql
WITH id_gaps AS (
    SELECT 
        order_id,
        LEAD(order_id) OVER (ORDER BY order_id) AS next_id,
        LEAD(order_id) OVER (ORDER BY order_id) - order_id AS gap
    FROM orders
)
SELECT 
    order_id AS gap_starts_after,
    next_id AS gap_ends_before,
    gap - 1 AS missing_count
FROM id_gaps
WHERE gap > 1;
```

**Q8: Sessionize user activity (group events into sessions with 30-min timeout).**
> See the [Sessionization](#sessionization) section above for the full solution.

---

## Quick Reference

### Window Function Cheat Sheet

| Function | Purpose | Example |
|----------|---------|---------|
| `ROW_NUMBER()` | Unique sequential number | Top-N per group |
| `RANK()` | Rank with gaps | Competition ranking |
| `DENSE_RANK()` | Rank without gaps | Percentile buckets |
| `NTILE(n)` | Divide into n buckets | Quartiles, deciles |
| `LAG(col, n)` | Previous row's value | Period-over-period |
| `LEAD(col, n)` | Next row's value | Forward-looking |
| `FIRST_VALUE()` | First in window | Compare to best/worst |
| `SUM() OVER` | Running/cumulative sum | Cumulative revenue |
| `AVG() OVER` | Moving average | Smoothed trends |
| `COUNT() OVER` | Running count | Customer order number |

### Query Optimization Checklist

| Check | Why |
|-------|-----|
| Use `EXPLAIN ANALYZE` | Understand actual execution plan |
| Index filter/join columns | Avoid full table scans |
| Avoid functions on indexed columns | `WHERE YEAR(date)=2024` → `WHERE date >= '2024-01-01'` |
| Select only needed columns | Less I/O, less memory |
| Use `EXISTS` over `IN` for subqueries | Often faster, NULL-safe |
| Prefer `UNION ALL` over `UNION` | Skip dedup sort when possible |
| Filter early | Reduce data before joins |
| Use CTEs for readability (not always perf) | Some DBs materialize CTEs |

### Common Date Functions (PostgreSQL Syntax)

```sql
CURRENT_DATE                          -- Today's date
CURRENT_TIMESTAMP                     -- Now with time
DATE_TRUNC('month', date_col)         -- First day of month
EXTRACT(YEAR FROM date_col)           -- Get year
date_col + INTERVAL '7 days'          -- Add days
AGE(date1, date2)                     -- Difference as interval
TO_CHAR(date_col, 'YYYY-MM-DD')      -- Format as string
```

---

*Next: [Chapter 09 - A/B Testing and Experimentation](09-AB-Testing-and-Experimentation.md)*
