# 2.5 — Aggregations & GROUP BY 🟢⭐

> **"Individual rows are data. Aggregated rows are insight."**

---

## 🧭 What You'll Master in This Chapter

Aggregation = turning **thousands of rows into meaningful summaries**. This chapter takes you from basic COUNT to advanced ROLLUP, CUBE, and GROUPING SETS — the stuff that makes your reports sing.

```
1000 rows of orders  ──→  "Total revenue: ₹45,00,000 across 6 countries"
     Raw Data              Aggregated Insight
```

---

## 🔥 1. Aggregate Functions — The Big Five

These functions collapse multiple rows into a single value:

```sql
SELECT 
    COUNT(*)          AS total_orders,      -- Count all rows
    COUNT(discount)   AS orders_with_disc,  -- Count non-NULL values only
    SUM(amount)       AS total_revenue,     -- Sum of all amounts
    AVG(amount)       AS average_order,     -- Average amount
    MIN(amount)       AS smallest_order,    -- Minimum amount
    MAX(amount)       AS largest_order      -- Maximum amount
FROM orders;
```

```
+──────────────+──────────────────+───────────────+───────────────+────────────────+───────────────+
│ total_orders │ orders_with_disc │ total_revenue │ average_order │ smallest_order │ largest_order │
+──────────────+──────────────────+───────────────+───────────────+────────────────+───────────────+
│      8       │        3         │    109000     │   13625.00    │     4200       │    32000      │
+──────────────+──────────────────+───────────────+───────────────+────────────────+───────────────+
```

### COUNT — The Three Flavors

```sql
-- COUNT(*) — counts ALL rows, including NULLs
SELECT COUNT(*) FROM customers;                    -- 6

-- COUNT(column) — counts non-NULL values only
SELECT COUNT(phone) FROM customers;                -- 4 (2 have NULL phone)

-- COUNT(DISTINCT column) — counts unique non-NULL values
SELECT COUNT(DISTINCT country) FROM customers;     -- 5
SELECT COUNT(DISTINCT status) FROM orders;         -- 4 (pending, shipped, delivered, cancelled)
```

> ⭐ **Critical Difference:**
> `COUNT(*)` counts **rows**.
> `COUNT(column)` counts **non-NULL values**.
> If 100 rows have 20 NULLs in `phone`, `COUNT(*)` = 100, `COUNT(phone)` = 80.

### SUM & AVG — Numeric Only

```sql
-- SUM ignores NULLs (doesn't treat as 0)
SELECT SUM(amount) FROM orders;             -- 109000
SELECT SUM(discount) FROM orders;           -- Sums only non-NULL discounts

-- AVG also ignores NULLs (this matters!)
-- If amounts are: 100, 200, NULL, 400
SELECT AVG(amount) FROM orders;             -- (100+200+400)/3 = 233.33
-- NOT (100+200+0+400)/4 = 175  ← NULLs are excluded from both sum AND count!

-- If you WANT NULLs treated as 0:
SELECT AVG(COALESCE(discount, 0)) FROM orders;    -- Now NULLs count as 0
```

> ⚠️ **The AVG + NULL Trap:** This is a classic interview question!
> ```
> Values: 10, 20, NULL, 30
> AVG(amount)                 = (10+20+30) / 3 = 20.00  ← NULL excluded
> AVG(COALESCE(amount, 0))    = (10+20+0+30) / 4 = 15.00  ← NULL as 0
> SUM(amount) / COUNT(*)      = 60 / 4 = 15.00  ← Another way to include NULLs
> ```

### MIN & MAX — Work With Any Ordered Type

```sql
-- Numbers
SELECT MIN(price), MAX(price) FROM products;

-- Dates (earliest, latest)
SELECT MIN(order_date) AS first_order, MAX(order_date) AS last_order FROM orders;

-- Strings (alphabetical — A is MIN, Z is MAX)
SELECT MIN(name), MAX(name) FROM customers;        -- Akira Tanaka ... Sarah Connor

-- Combined with other functions
SELECT 
    MAX(amount) - MIN(amount) AS amount_range,
    MAX(order_date) - MIN(order_date) AS date_span    -- PostgreSQL (returns interval)
FROM orders;
```

---

## 🔥 2. GROUP BY — Aggregating by Category

GROUP BY splits your data into groups, then applies aggregate functions to each group.

```
Before GROUP BY:                    After GROUP BY country:
┌──────────────┬─────────┐          ┌─────────┬───────┬──────────┐
│ customer     │ country │          │ country │ count │ total    │
├──────────────┼─────────┤          ├─────────┼───────┼──────────┤
│ Ritesh       │ India   │          │ India   │   2   │   56800  │
│ Sarah        │ USA     │  ──→     │ USA     │   1   │    8500  │
│ Akira        │ Japan   │          │ Japan   │   1   │    4200  │
│ Maria        │ Spain   │          │ Spain   │   1   │    6700  │
│ John         │ UK      │          │ UK      │   1   │   11000  │
│ Priya        │ India   │          └─────────┴───────┴──────────┘
└──────────────┴─────────┘
```

### Basic GROUP BY

```sql
-- Orders per customer
SELECT customer_id, COUNT(*) AS order_count, SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id;

-- Revenue by status
SELECT status, COUNT(*) AS count, SUM(amount) AS total
FROM orders
GROUP BY status;

-- Revenue by month
SELECT 
    EXTRACT(YEAR FROM order_date) AS year,
    EXTRACT(MONTH FROM order_date) AS month,
    COUNT(*) AS orders,
    SUM(amount) AS revenue
FROM orders
GROUP BY EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date)
ORDER BY year, month;
```

### The Golden Rule of GROUP BY

> ⭐ **Every column in SELECT must be either:**
> 1. **In the GROUP BY clause**, OR
> 2. **Inside an aggregate function** (COUNT, SUM, AVG, etc.)

```sql
-- ❌ WRONG: 'name' is not in GROUP BY or an aggregate
SELECT name, country, COUNT(*)
FROM customers
GROUP BY country;
-- ERROR: "name" must appear in GROUP BY or be in an aggregate function

-- ✅ CORRECT: name is in GROUP BY
SELECT name, country, COUNT(*) 
FROM customers 
GROUP BY name, country;

-- ✅ CORRECT: name is in an aggregate
SELECT country, COUNT(*) AS customer_count, STRING_AGG(name, ', ') AS names
FROM customers
GROUP BY country;
```

> 💡 **MySQL Exception:** MySQL (with default settings) allows non-aggregated columns not in GROUP BY. It just picks an **arbitrary value** from the group. This is dangerous and non-standard! Enable `ONLY_FULL_GROUP_BY` mode to prevent this.

### GROUP BY with Multiple Columns

```sql
-- Sales by country AND status
SELECT 
    c.country,
    o.status,
    COUNT(*) AS orders,
    SUM(o.amount) AS total
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.country, o.status
ORDER BY c.country, total DESC;
```

```
+─────────+───────────+────────+────────+
│ country │ status    │ orders │ total  │
+─────────+───────────+────────+────────+
│ India   │ delivered │   2    │ 53000  │
│ India   │ shipped   │   1    │ 15000  │
│ India   │ pending   │   1    │  9800  │
│ Spain   │ cancelled │   1    │  6700  │
│ USA     │ delivered │   1    │  8500  │
│ ...     │ ...       │  ...   │  ...   │
+─────────+───────────+────────+────────+
```

### GROUP BY with Expressions

```sql
-- Group by computed value
SELECT 
    CASE 
        WHEN amount >= 20000 THEN 'Premium'
        WHEN amount >= 10000 THEN 'Standard'
        ELSE 'Basic'
    END AS tier,
    COUNT(*) AS order_count,
    AVG(amount) AS avg_amount
FROM orders
GROUP BY 
    CASE 
        WHEN amount >= 20000 THEN 'Premium'
        WHEN amount >= 10000 THEN 'Standard'
        ELSE 'Basic'
    END;

-- Group by date part
SELECT 
    DATE_TRUNC('month', order_date) AS month,       -- PostgreSQL
    -- FORMAT(order_date, 'yyyy-MM') AS month,      -- SQL Server
    -- DATE_FORMAT(order_date, '%Y-%m') AS month,    -- MySQL
    COUNT(*) AS orders,
    SUM(amount) AS revenue
FROM orders
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;
```

---

## 🔥 3. HAVING — Filter on Aggregated Results

WHERE filters **individual rows** (before grouping).
HAVING filters **groups** (after grouping).

```
Execution order reminder:
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY

WHERE  = filter rows     (happens BEFORE GROUP BY)
HAVING = filter groups   (happens AFTER GROUP BY)
```

```sql
-- Customers who placed more than 2 orders
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 2;

-- Categories with average price above ₹50,000
SELECT category, AVG(price) AS avg_price
FROM products
GROUP BY category
HAVING AVG(price) > 50000;

-- Countries with total spending above ₹20,000
SELECT c.country, SUM(o.amount) AS total
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.country
HAVING SUM(o.amount) > 20000
ORDER BY total DESC;
```

### WHERE vs HAVING — Know the Difference!

```sql
-- ❌ WRONG: Can't use aggregate in WHERE
SELECT status, COUNT(*) 
FROM orders 
WHERE COUNT(*) > 1       -- ERROR! Aggregates not allowed in WHERE
GROUP BY status;

-- ✅ CORRECT: Use HAVING for aggregate conditions
SELECT status, COUNT(*) 
FROM orders 
GROUP BY status
HAVING COUNT(*) > 1;

-- Both WHERE and HAVING in same query
SELECT c.country, COUNT(*) AS order_count, SUM(o.amount) AS total
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.status != 'cancelled'        -- WHERE: filter ROWS before grouping
GROUP BY c.country
HAVING SUM(o.amount) > 10000         -- HAVING: filter GROUPS after grouping
ORDER BY total DESC;
```

> 💡 **Performance Tip:** Use WHERE to filter as many rows as possible BEFORE grouping. Don't use HAVING when WHERE would work — HAVING runs later and processes more data.
> ```sql
> -- ❌ Slower: HAVING filters after grouping all statuses
> SELECT status, SUM(amount) FROM orders 
> GROUP BY status HAVING status != 'cancelled';
> 
> -- ✅ Faster: WHERE filters before grouping
> SELECT status, SUM(amount) FROM orders 
> WHERE status != 'cancelled' GROUP BY status;
> ```

---

## 🔥 4. DISTINCT vs GROUP BY

```sql
-- These produce the SAME result:
SELECT DISTINCT country FROM customers;
SELECT country FROM customers GROUP BY country;

-- But GROUP BY can do MORE (add aggregates):
SELECT country, COUNT(*) FROM customers GROUP BY country;
-- DISTINCT can't do this!
```

> 💡 Performance-wise, they're usually identical (same execution plan). Use DISTINCT for "unique values" and GROUP BY for "unique values + aggregation."

---

## 🔥 5. String Aggregation — Combine Values into One String

```sql
-- PostgreSQL 9.0+: STRING_AGG
SELECT country, STRING_AGG(name, ', ' ORDER BY name) AS customers
FROM customers
GROUP BY country;
-- India → "Priya Sharma, Ritesh Singh"

-- MySQL: GROUP_CONCAT
SELECT country, GROUP_CONCAT(name ORDER BY name SEPARATOR ', ') AS customers
FROM customers
GROUP BY country;

-- SQL Server 2017+: STRING_AGG
SELECT country, STRING_AGG(name, ', ') WITHIN GROUP (ORDER BY name) AS customers
FROM customers
GROUP BY country;

-- Oracle: LISTAGG
SELECT country, LISTAGG(name, ', ') WITHIN GROUP (ORDER BY name) AS customers
FROM customers
GROUP BY country;
```

---

## 🔥 6. Advanced Grouping — ROLLUP, CUBE, GROUPING SETS

These generate **subtotals and grand totals** in a single query. Incredibly powerful for reports.

### ROLLUP — Hierarchical Subtotals

ROLLUP creates subtotals from right to left, plus a grand total.

```sql
SELECT 
    country,
    city,
    COUNT(*) AS customers,
    SUM(total_spent) AS revenue
FROM customer_summary
GROUP BY ROLLUP (country, city);
```

```
+─────────+────────+───────────+──────────+
│ country │ city   │ customers │ revenue  │
+─────────+────────+───────────+──────────+
│ India   │ Delhi  │     1     │  56800   │  ← Delhi detail
│ India   │ Mumbai │     1     │  21000   │  ← Mumbai detail
│ India   │ NULL   │     2     │  77800   │  ← India subtotal (ROLLUP)
│ USA     │ NYC    │     1     │   8500   │  ← NYC detail
│ USA     │ NULL   │     1     │   8500   │  ← USA subtotal (ROLLUP)
│ Japan   │ Tokyo  │     1     │   4200   │
│ Japan   │ NULL   │     1     │   4200   │  ← Japan subtotal
│ NULL    │ NULL   │     6     │ 109000   │  ← GRAND TOTAL (ROLLUP)
+─────────+────────+───────────+──────────+
```

> The NULLs represent "all values" — the subtotal/grand total rows.

### CUBE — All Possible Combinations

CUBE generates subtotals for **every possible combination** of the grouped columns.

```sql
SELECT 
    country,
    status,
    COUNT(*) AS orders,
    SUM(amount) AS total
FROM orders o JOIN customers c ON o.customer_id = c.id
GROUP BY CUBE (country, status);
```

```
CUBE generates:
✅ (country, status)  — each combination
✅ (country, NULL)    — subtotal per country
✅ (NULL, status)     — subtotal per status
✅ (NULL, NULL)       — grand total
```

> 💡 **ROLLUP vs CUBE:**
> - ROLLUP(A, B) = 3 grouping levels: (A,B), (A), ()
> - CUBE(A, B) = 4 grouping levels: (A,B), (A), (B), ()
> - ROLLUP is for **hierarchies** (Year > Quarter > Month)
> - CUBE is for **all dimensions** (Country × Status)

### GROUPING SETS — Custom Grouping

Define exactly which groupings you want — full control.

```sql
SELECT 
    country,
    status,
    COUNT(*) AS orders,
    SUM(amount) AS total
FROM orders o JOIN customers c ON o.customer_id = c.id
GROUP BY GROUPING SETS (
    (country, status),    -- Detail level
    (country),            -- Country subtotal
    (status),             -- Status subtotal
    ()                    -- Grand total
);
```

### GROUPING() Function — Identify Subtotal Rows

How do you distinguish a NULL that means "subtotal" from a real NULL in the data?

```sql
SELECT 
    CASE WHEN GROUPING(country) = 1 THEN '** ALL COUNTRIES **' ELSE country END AS country,
    CASE WHEN GROUPING(status) = 1 THEN '** ALL STATUSES **' ELSE status END AS status,
    COUNT(*) AS orders,
    SUM(amount) AS total
FROM orders o JOIN customers c ON o.customer_id = c.id
GROUP BY ROLLUP (country, status);
```

```
+────────────────────+───────────────────+────────+────────+
│ country            │ status            │ orders │ total  │
+────────────────────+───────────────────+────────+────────+
│ India              │ delivered         │   2    │ 53000  │
│ India              │ shipped           │   1    │ 15000  │
│ India              │ ** ALL STATUSES **│   3    │ 68000  │  ← India subtotal
│ USA                │ delivered         │   1    │  8500  │
│ USA                │ ** ALL STATUSES **│   1    │  8500  │  ← USA subtotal
│ ** ALL COUNTRIES **│ ** ALL STATUSES **│   8    │109000  │  ← Grand total
+────────────────────+───────────────────+────────+────────+
```

### Cross-Database Support

```
┌───────────────┬──────────┬──────────┬──────────────┬────────────┐
│ Feature       │ Oracle   │ SQL Srv  │ MySQL        │ PostgreSQL │
├───────────────┼──────────┼──────────┼──────────────┼────────────┤
│ ROLLUP        │ ✅       │ ✅       │ ✅ (8.0+)    │ ✅         │
│ CUBE          │ ✅       │ ✅       │ ❌           │ ✅         │
│ GROUPING SETS │ ✅       │ ✅       │ ❌           │ ✅         │
│ GROUPING()    │ ✅       │ ✅       │ ✅ (8.0+)    │ ✅         │
└───────────────┴──────────┴──────────┴──────────────┴────────────┘
```

---

## 🔥 7. Conditional Aggregation — Pivot Without PIVOT

Use CASE inside aggregate functions to create cross-tab reports:

```sql
-- Revenue breakdown by status in a single row
SELECT 
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS delivered,
    COUNT(CASE WHEN status = 'shipped' THEN 1 END)   AS shipped,
    COUNT(CASE WHEN status = 'pending' THEN 1 END)   AS pending,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled
FROM orders;

-- Monthly revenue as columns (manual pivot)
SELECT 
    c.country,
    SUM(CASE WHEN EXTRACT(MONTH FROM o.order_date) = 1 THEN o.amount ELSE 0 END) AS jan,
    SUM(CASE WHEN EXTRACT(MONTH FROM o.order_date) = 2 THEN o.amount ELSE 0 END) AS feb,
    SUM(CASE WHEN EXTRACT(MONTH FROM o.order_date) = 3 THEN o.amount ELSE 0 END) AS mar,
    SUM(o.amount) AS total
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.country;
```

```
+─────────+────────+────────+────────+────────+
│ country │ jan    │ feb    │ mar    │ total  │
+─────────+────────+────────+────────+────────+
│ India   │ 15000  │ 32000  │ 30800  │ 77800  │
│ USA     │  8500  │     0  │     0  │  8500  │
│ Japan   │     0  │  4200  │     0  │  4200  │
│ Spain   │     0  │     0  │  6700  │  6700  │
│ UK      │     0  │     0  │ 11000  │ 11000  │
+─────────+────────+────────+────────+────────+
```

> 💡 This is called **conditional aggregation** or **manual pivot**. It works on ALL databases, unlike PIVOT which is vendor-specific.

### FILTER Clause (PostgreSQL only — cleaner syntax)

```sql
-- PostgreSQL 9.4+: FILTER instead of CASE
SELECT 
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE status = 'delivered') AS delivered,
    COUNT(*) FILTER (WHERE status = 'pending') AS pending,
    SUM(amount) FILTER (WHERE status != 'cancelled') AS active_revenue
FROM orders;
```

---

## 🔥 8. Aggregate Function Extras

### Statistical Functions

```sql
-- Standard deviation & variance
SELECT 
    STDDEV(price) AS std_dev,           -- Sample standard deviation
    STDDEV_POP(price) AS pop_std_dev,   -- Population standard deviation
    VARIANCE(price) AS variance,        -- Sample variance
    VAR_POP(price) AS pop_variance      -- Population variance
FROM products;

-- Percentile (PostgreSQL, Oracle)
SELECT 
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) AS median,     -- Median
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY amount) AS p95        -- 95th percentile
FROM orders;
```

### BOOL Aggregates (PostgreSQL)

```sql
-- Check if ANY or ALL values match
SELECT 
    BOOL_OR(is_active) AS any_active,     -- TRUE if at least one is active
    BOOL_AND(is_active) AS all_active,    -- TRUE only if ALL are active
    EVERY(is_active) AS every_active      -- Same as BOOL_AND
FROM customers;
```

### Array/JSON Aggregates

```sql
-- PostgreSQL: Collect into array
SELECT country, ARRAY_AGG(name ORDER BY name) AS customers
FROM customers
GROUP BY country;
-- India → {Priya Sharma, Ritesh Singh}

-- PostgreSQL: Collect into JSON
SELECT JSON_AGG(ROW_TO_JSON(c)) FROM customers c WHERE country = 'India';

-- MySQL: JSON_ARRAYAGG
SELECT country, JSON_ARRAYAGG(name) AS customers
FROM customers
GROUP BY country;
```

---

## 🔥 9. Execution Order Deep Dive — How Aggregation Actually Works

```
Step 1: FROM orders o JOIN customers c ON ...     ← Combine tables
Step 2: WHERE o.status != 'cancelled'             ← Filter individual rows
Step 3: GROUP BY c.country                        ← Create groups
Step 4: HAVING SUM(o.amount) > 10000              ← Filter groups
Step 5: SELECT c.country, SUM(o.amount) AS total  ← Compute final columns
Step 6: ORDER BY total DESC                       ← Sort results
Step 7: LIMIT 5                                   ← Paginate
```

```
Original rows (after WHERE):
┌──────────┬─────────┬────────┐
│ customer │ country │ amount │
├──────────┼─────────┼────────┤
│ Ritesh   │ India   │ 15000  │    ──┐
│ Ritesh   │ India   │ 32000  │    ──┤ GROUP: India
│ Priya    │ India   │ 21000  │    ──┘ SUM = 68000 ✅ (>10000)
│ Sarah    │ USA     │  8500  │    ──  GROUP: USA
│                                      SUM = 8500  ❌ (<10000, filtered by HAVING)
│ Akira    │ Japan   │  4200  │    ──  GROUP: Japan
│                                      SUM = 4200  ❌
└──────────┴─────────┴────────┘

After HAVING SUM > 10000:
┌─────────┬────────┐
│ country │ total  │
├─────────┼────────┤
│ India   │ 68000  │
└─────────┴────────┘
```

---

## 🧠 Common Mistakes with Aggregation

| # | Mistake | Problem | Fix |
|---|---------|---------|-----|
| 1 | Non-aggregated column not in GROUP BY | Error (except MySQL) | Add to GROUP BY or wrap in aggregate |
| 2 | Using WHERE instead of HAVING for aggregates | Error: "aggregate not allowed in WHERE" | Use HAVING for aggregate conditions |
| 3 | AVG ignoring NULLs silently | Wrong average | Use `AVG(COALESCE(col, 0))` if NULLs should be 0 |
| 4 | COUNT(*) vs COUNT(col) confusion | Different results with NULLs | Know the difference |
| 5 | HAVING when WHERE would work | Slower (processes all groups first) | Use WHERE for row-level filters |
| 6 | SUM with JOINs causing double-counting | Inflated totals (row multiplication) | Aggregate in subquery before JOIN |
| 7 | GROUP BY with wrong granularity | Missing or duplicate groups | Verify which columns define a unique group |

### The Double-Counting Trap

```sql
-- ❌ WRONG: If a customer has 3 orders and 2 addresses, 
-- each order is counted 2x (once per address)
SELECT c.id, SUM(o.amount) AS total
FROM customers c
JOIN orders o ON c.id = o.customer_id
JOIN addresses a ON c.id = a.customer_id
GROUP BY c.id;
-- Ritesh's total shows 56800 × 2 = 113600 instead of 56800!

-- ✅ FIX: Aggregate before joining
SELECT c.id, order_totals.total
FROM customers c
JOIN (
    SELECT customer_id, SUM(amount) AS total
    FROM orders GROUP BY customer_id
) order_totals ON c.id = order_totals.customer_id;
```

---

## ⚔️ Quick Challenge

**Q1:** How many orders does each status have? Show only statuses with more than 1 order.
```sql
SELECT status, COUNT(*) AS count
FROM orders
GROUP BY status
HAVING COUNT(*) > 1;
```

**Q2:** Find the top 3 customers by total spending
```sql
SELECT c.name, SUM(o.amount) AS total_spent
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.name
ORDER BY total_spent DESC
LIMIT 3;
```

**Q3:** Show monthly revenue with a running note for months above ₹50,000
```sql
SELECT 
    EXTRACT(MONTH FROM order_date) AS month,
    SUM(amount) AS revenue,
    CASE WHEN SUM(amount) > 50000 THEN 'Above Target' ELSE 'Below Target' END AS performance
FROM orders
GROUP BY EXTRACT(MONTH FROM order_date)
ORDER BY month;
```

**Q4:** Create a cross-tab report: countries as rows, statuses as columns, with order counts
```sql
SELECT 
    c.country,
    COUNT(CASE WHEN o.status = 'delivered' THEN 1 END) AS delivered,
    COUNT(CASE WHEN o.status = 'shipped' THEN 1 END) AS shipped,
    COUNT(CASE WHEN o.status = 'pending' THEN 1 END) AS pending,
    COUNT(CASE WHEN o.status = 'cancelled' THEN 1 END) AS cancelled,
    COUNT(*) AS total
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.country
ORDER BY total DESC;
```

---

## 🎯 Key Takeaways

```
┌──────────────────────────────────────────────────────────────────────┐
│  ✅ COUNT(*) counts rows, COUNT(col) counts non-NULLs              │
│  ✅ AVG ignores NULLs — use COALESCE if NULLs should be 0         │
│  ✅ Every non-aggregated column must be in GROUP BY                │
│  ✅ WHERE filters rows, HAVING filters groups — use WHERE first    │
│  ✅ ROLLUP = hierarchical subtotals (right to left)                │
│  ✅ CUBE = all dimension combinations                              │
│  ✅ GROUPING SETS = custom grouping levels                         │
│  ✅ Conditional aggregation (CASE in SUM/COUNT) = manual pivot     │
│  ✅ Watch for double-counting when JOINing before aggregating      │
│  ✅ Use GROUPING() to distinguish real NULLs from subtotal NULLs   │
└──────────────────────────────────────────────────────────────────────┘
```

---

> **← Previous:** [2.4 JOINs — The Heart of Relational Data](./04-JOINs.md)
> **Next →** [2.6 Subqueries & Derived Tables](./06-Subqueries.md)
