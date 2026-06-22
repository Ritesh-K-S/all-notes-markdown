# 2.6 — Subqueries & Derived Tables 🟡⭐

> **"A query inside a query. It's SQL inception — and once you get it, you'll use it everywhere."**

---

## 🧭 What You'll Master in This Chapter

Subqueries let you **nest queries inside other queries** — answering complex questions that a single query can't handle alone. This is where SQL starts feeling like a superpower.

```
"Show me customers who spend MORE than the average order amount"

    Main Query: SELECT customers WHERE amount > ???
                                                  ↑
    Subquery:                         (SELECT AVG(amount) FROM orders)

One query feeds the answer into another. That's a subquery.
```

---

## 📖 What is a Subquery?

A **subquery** (also called inner query, nested query) is a SELECT statement embedded inside another SQL statement.

```sql
-- The subquery is in parentheses
SELECT name, amount
FROM orders
WHERE amount > (SELECT AVG(amount) FROM orders);
--              └──────────── subquery ────────┘
```

### Where Can Subqueries Appear?

```
┌──────────────────────────────────────────────────────────────┐
│              SUBQUERY LOCATIONS IN SQL                        │
├──────────────┬───────────────────────────────────────────────┤
│ In WHERE     │ Filter based on another query's result        │
│ In FROM      │ Use query result as a "virtual table"         │
│ In SELECT    │ Compute a value for each row                  │
│ In HAVING    │ Filter groups based on another query          │
│ In INSERT    │ INSERT INTO ... SELECT (subquery)             │
│ In UPDATE    │ SET col = (subquery)                          │
│ In DELETE    │ DELETE WHERE col IN (subquery)                │
└──────────────┴───────────────────────────────────────────────┘
```

---

## 🔥 1. Types of Subqueries — The Complete Map

```
┌──────────────────────────────────────────────────────────────────────┐
│                    SUBQUERY CLASSIFICATION                           │
├────────────────────┬─────────────────────────────────────────────────┤
│ By Return Type     │                                                 │
│   Scalar           │ Returns ONE value (single row, single column)   │
│   Row              │ Returns ONE row (multiple columns)              │
│   Table            │ Returns multiple rows & columns                 │
├────────────────────┼─────────────────────────────────────────────────┤
│ By Dependency      │                                                 │
│   Non-correlated   │ Independent — runs once, result reused          │
│   Correlated       │ Dependent — runs once PER ROW of outer query    │
├────────────────────┼─────────────────────────────────────────────────┤
│ By Location        │                                                 │
│   WHERE subquery   │ Filtering condition                             │
│   FROM subquery    │ Derived table / inline view                     │
│   SELECT subquery  │ Scalar calculation per row                      │
└────────────────────┴─────────────────────────────────────────────────┘
```

---

## 🔥 2. Scalar Subqueries — Return ONE Value

A scalar subquery returns exactly **one row, one column** — a single value. You can use it anywhere a value is expected.

### In WHERE Clause

```sql
-- Orders above average amount
SELECT id, customer_id, amount
FROM orders
WHERE amount > (SELECT AVG(amount) FROM orders);
--               └── scalar: returns 13625.00 ──┘

-- The most expensive product
SELECT * FROM products 
WHERE price = (SELECT MAX(price) FROM products);

-- Customer who placed the latest order
SELECT * FROM customers 
WHERE id = (
    SELECT customer_id FROM orders 
    ORDER BY order_date DESC 
    LIMIT 1
);
```

### In SELECT Clause

```sql
-- Show each order with the overall average for comparison
SELECT 
    id,
    amount,
    (SELECT AVG(amount) FROM orders) AS overall_avg,
    amount - (SELECT AVG(amount) FROM orders) AS diff_from_avg
FROM orders;
```

```
+────+────────+─────────────+───────────────+
│ id │ amount │ overall_avg │ diff_from_avg │
+────+────────+─────────────+───────────────+
│  1 │ 15000  │   13625.00  │    1375.00    │
│  2 │  8500  │   13625.00  │   -5125.00    │
│  3 │ 32000  │   13625.00  │   18375.00    │
│  4 │  4200  │   13625.00  │   -9425.00    │
│  ...                                      │
+────+────────+─────────────+───────────────+
```

### In UPDATE/DELETE

```sql
-- Set discount for products priced above average
UPDATE products
SET discount = 10
WHERE price > (SELECT AVG(price) FROM products);

-- Delete the oldest order
DELETE FROM orders
WHERE order_date = (SELECT MIN(order_date) FROM orders);
```

> ⚠️ **Scalar subquery must return exactly ONE value.** If it returns multiple rows, you get an error!
> ```sql
> -- ❌ ERROR: Subquery returns more than 1 row
> SELECT * FROM orders WHERE customer_id = (SELECT id FROM customers);
>
> -- ✅ Fix: Use IN instead of =
> SELECT * FROM orders WHERE customer_id IN (SELECT id FROM customers);
> ```

---

## 🔥 3. Table Subqueries — Return Multiple Rows

### IN — Check Membership in a List

```sql
-- Customers who have placed at least one order
SELECT * FROM customers
WHERE id IN (SELECT customer_id FROM orders);

-- Products in the 'Phones' or 'Laptops' categories
SELECT * FROM products
WHERE category_id IN (
    SELECT id FROM categories WHERE name IN ('Phones', 'Laptops')
);
```

### NOT IN — Exclusion (Watch for NULLs!)

```sql
-- Customers who have NEVER ordered
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
```

> ⚠️ **THE NOT IN + NULL TRAP — This is critical!**
> ```sql
> -- If the subquery returns ANY NULL, NOT IN returns NOTHING!
> SELECT * FROM customers
> WHERE id NOT IN (1, 2, NULL);
> -- Returns EMPTY RESULT SET! Because:
> -- id NOT IN (1, 2, NULL)
> -- = id != 1 AND id != 2 AND id != NULL
> -- = TRUE AND TRUE AND UNKNOWN
> -- = UNKNOWN (not TRUE, so row excluded)
> 
> -- ✅ FIX: Use NOT EXISTS instead (NULL-safe)
> SELECT * FROM customers c
> WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
>
> -- ✅ Or filter NULLs explicitly
> SELECT * FROM customers
> WHERE id NOT IN (SELECT customer_id FROM orders WHERE customer_id IS NOT NULL);
> ```

### ANY / SOME & ALL — Compare Against a Set

```sql
-- ANY/SOME: TRUE if comparison matches AT LEAST ONE value
-- "Orders with amount greater than ANY order from customer 2"
SELECT * FROM orders
WHERE amount > ANY (SELECT amount FROM orders WHERE customer_id = 2);
-- Same as: WHERE amount > MIN(amounts from customer 2)

-- ALL: TRUE if comparison matches EVERY value
-- "Orders with amount greater than ALL orders from customer 2"  
SELECT * FROM orders
WHERE amount > ALL (SELECT amount FROM orders WHERE customer_id = 2);
-- Same as: WHERE amount > MAX(amounts from customer 2)
```

```
┌────────────────────────────────────────────────────────────────┐
│  > ANY (subquery)  ═  > MIN(subquery)  ═  "greater than some" │
│  > ALL (subquery)  ═  > MAX(subquery)  ═  "greater than all"  │
│  < ANY (subquery)  ═  < MAX(subquery)  ═  "less than some"    │
│  < ALL (subquery)  ═  < MIN(subquery)  ═  "less than all"     │
│  = ANY (subquery)  ═  IN (subquery)    ═  "equal to any"      │
└────────────────────────────────────────────────────────────────┘
```

---

## 🔥 4. EXISTS & NOT EXISTS — The Most Powerful Pattern

EXISTS doesn't care about WHAT the subquery returns — only WHETHER it returns any rows.

```sql
-- Customers who have at least one order (EXISTS = TRUE if subquery returns ≥1 row)
SELECT c.name, c.email
FROM customers c
WHERE EXISTS (
    SELECT 1                            -- "1" is convention; could be *, 'x', anything
    FROM orders o
    WHERE o.customer_id = c.id          -- Correlated: references outer query's c.id
);

-- Customers who have NEVER ordered
SELECT c.name, c.email
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

### EXISTS vs IN — When to Use Which?

```
┌────────────────────┬───────────────────────┬───────────────────────┐
│                    │ IN                    │ EXISTS                │
├────────────────────┼───────────────────────┼───────────────────────┤
│ NULL safety        │ ⚠️ NOT IN fails with  │ ✅ NULL-safe          │
│                    │ NULLs in subquery     │                       │
├────────────────────┼───────────────────────┼───────────────────────┤
│ Performance        │ Better when subquery  │ Better when outer     │
│                    │ returns FEW rows      │ table is SMALL and    │
│                    │                       │ subquery table is BIG │
├────────────────────┼───────────────────────┼───────────────────────┤
│ Short-circuits     │ ❌ No (evaluates all) │ ✅ Yes (stops at      │
│                    │                       │ first match)          │
├────────────────────┼───────────────────────┼───────────────────────┤
│ Readability        │ Simpler for lists     │ Better for correlated │
│                    │                       │ conditions            │
└────────────────────┴───────────────────────┴───────────────────────┘
```

> ⭐ **Rule of Thumb:**
> - Use `IN` for simple, small result sets
> - Use `EXISTS` for correlated conditions and when NULL safety matters
> - Use `NOT EXISTS` instead of `NOT IN` (always safer)
> - Modern query optimizers often make them equivalent in performance

---

## 🔥 5. Correlated Subqueries — The Powerful (and Tricky) Ones

A **correlated subquery** references a column from the outer query. It runs **once for each row** in the outer query.

```
Non-Correlated:                      Correlated:
┌──────────┐                         ┌──────────┐
│ Outer    │   runs once             │ Outer    │   Row 1 → run subquery
│ Query    │ ←─────────── Subquery   │ Query    │   Row 2 → run subquery
│          │   result reused         │          │   Row 3 → run subquery
└──────────┘                         └──────────┘   ...N → run subquery
                                                    (N executions!)
```

### Examples of Correlated Subqueries

```sql
-- For each customer, show their most recent order date
SELECT 
    c.name,
    (SELECT MAX(o.order_date) 
     FROM orders o 
     WHERE o.customer_id = c.id          -- ← References outer query!
    ) AS last_order_date
FROM customers c;

-- Employees earning above their DEPARTMENT average
SELECT e.name, e.salary, e.department
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary) 
    FROM employees e2 
    WHERE e2.department = e.department    -- ← Correlated to outer row
);

-- Products more expensive than the average of their category
SELECT p.name, p.price, p.category
FROM products p
WHERE p.price > (
    SELECT AVG(p2.price)
    FROM products p2
    WHERE p2.category = p.category       -- ← Same category as current row
);
```

### Correlated Subquery in UPDATE

```sql
-- Set each customer's total_orders to their actual order count
UPDATE customers c
SET total_orders = (
    SELECT COUNT(*) 
    FROM orders o 
    WHERE o.customer_id = c.id
);

-- Set product rank within category
UPDATE products p
SET category_rank = (
    SELECT COUNT(*) + 1
    FROM products p2
    WHERE p2.category = p.category AND p2.price > p.price
);
```

> ⚠️ **Performance Warning:** Correlated subqueries can be **very slow** on large tables because they execute the inner query for EVERY row of the outer query. N rows = N subquery executions. Consider rewriting as a JOIN when possible.

### Rewriting Correlated Subqueries as JOINs

```sql
-- Correlated subquery (potentially slow)
SELECT c.name,
    (SELECT SUM(o.amount) FROM orders o WHERE o.customer_id = c.id) AS total
FROM customers c;

-- Equivalent JOIN (usually faster)
SELECT c.name, COALESCE(SUM(o.amount), 0) AS total
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;

-- Equivalent JOIN with derived table (often fastest)
SELECT c.name, COALESCE(ot.total, 0) AS total
FROM customers c
LEFT JOIN (
    SELECT customer_id, SUM(amount) AS total
    FROM orders
    GROUP BY customer_id
) ot ON c.id = ot.customer_id;
```

---

## 🔥 6. Derived Tables (Subqueries in FROM) — Virtual Tables

A subquery in the FROM clause creates a **temporary, unnamed table** that exists only for the duration of the query.

```sql
-- Derived table: Get top spending customers, then join with their details
SELECT c.name, c.email, spending.total_spent, spending.order_count
FROM customers c
JOIN (
    -- This subquery IS the derived table
    SELECT 
        customer_id,
        COUNT(*) AS order_count,
        SUM(amount) AS total_spent
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY customer_id
) spending ON c.id = spending.customer_id     -- Must alias the derived table!
WHERE spending.total_spent > 20000
ORDER BY spending.total_spent DESC;
```

> ⭐ **Derived tables MUST have an alias** in most databases. Oracle is the exception (alias is optional but recommended).

### When to Use Derived Tables

```sql
-- 1. Pre-aggregate before joining (prevent row multiplication)
SELECT c.name, order_summary.total
FROM customers c
JOIN (
    SELECT customer_id, SUM(amount) AS total 
    FROM orders GROUP BY customer_id
) order_summary ON c.id = order_summary.customer_id;

-- 2. Filter on aggregated values without HAVING
SELECT * FROM (
    SELECT country, COUNT(*) AS customer_count
    FROM customers
    GROUP BY country
) country_counts
WHERE customer_count > 1;

-- 3. Use window functions then filter (can't filter window functions in WHERE)
SELECT * FROM (
    SELECT name, amount, 
           ROW_NUMBER() OVER (ORDER BY amount DESC) AS rank
    FROM orders
) ranked
WHERE rank <= 5;

-- 4. Multi-level aggregation
SELECT category, AVG(product_revenue) AS avg_product_revenue
FROM (
    SELECT p.category, p.name, SUM(oi.quantity * oi.unit_price) AS product_revenue
    FROM products p
    JOIN order_items oi ON p.id = oi.product_id
    GROUP BY p.category, p.name
) product_revenues
GROUP BY category;
```

---

## 🔥 7. Inline Views vs CTEs — Same Goal, Different Style

Both derived tables and CTEs create "virtual tables." CTEs (covered in depth in Chapter 2.8) are generally more readable:

```sql
-- DERIVED TABLE (subquery in FROM)
SELECT c.name, ot.total
FROM customers c
JOIN (
    SELECT customer_id, SUM(amount) AS total
    FROM orders GROUP BY customer_id
) ot ON c.id = ot.customer_id;

-- EQUIVALENT CTE (Common Table Expression) — often more readable
WITH order_totals AS (
    SELECT customer_id, SUM(amount) AS total
    FROM orders GROUP BY customer_id
)
SELECT c.name, ot.total
FROM customers c
JOIN order_totals ot ON c.id = ot.customer_id;
```

> 💡 **When to choose which?**
> - CTE: Readable, can be referenced multiple times, supports recursion
> - Derived table: Fine for simple, one-time subqueries
> - Performance: Usually identical (optimizer treats them the same)

---

## 🔥 8. Real-World Subquery Patterns

### Pattern 1: Top-N Per Group (Before Window Functions)

```sql
-- Top 2 most expensive products per category (without window functions)
SELECT p.*
FROM products p
WHERE (
    SELECT COUNT(*) 
    FROM products p2 
    WHERE p2.category = p.category AND p2.price > p.price
) < 2;
-- For each product, count how many in the same category cost more.
-- If fewer than 2 cost more → this product is in the top 2.
```

### Pattern 2: Running Comparison

```sql
-- Orders where the amount is above the average of all PREVIOUS orders
SELECT o1.id, o1.order_date, o1.amount,
    (SELECT AVG(o2.amount) 
     FROM orders o2 
     WHERE o2.order_date < o1.order_date
    ) AS avg_before
FROM orders o1
WHERE o1.amount > (
    SELECT AVG(o2.amount) 
    FROM orders o2 
    WHERE o2.order_date < o1.order_date
);
```

### Pattern 3: Finding Gaps

```sql
-- Find missing IDs in a sequence
SELECT t1.id + 1 AS gap_start
FROM orders t1
WHERE NOT EXISTS (
    SELECT 1 FROM orders t2 WHERE t2.id = t1.id + 1
) AND t1.id < (SELECT MAX(id) FROM orders);
```

### Pattern 4: Division — "All" Queries

```sql
-- Customers who have ordered EVERY product in the 'Phones' category
-- (Relational division — one of the hardest SQL patterns)
SELECT c.name
FROM customers c
WHERE NOT EXISTS (
    -- Find a Phone product that this customer HASN'T ordered
    SELECT p.id
    FROM products p
    WHERE p.category = 'Phones'
      AND NOT EXISTS (
          SELECT 1
          FROM orders o
          JOIN order_items oi ON o.id = oi.order_id
          WHERE o.customer_id = c.id AND oi.product_id = p.id
      )
);
-- Logic: "There does NOT EXIST a Phone that this customer has NOT ordered"
-- = Customer has ordered ALL phones
```

### Pattern 5: Row-by-Row Percentage

```sql
-- Each order as a percentage of total revenue
SELECT 
    id,
    amount,
    ROUND(amount * 100.0 / (SELECT SUM(amount) FROM orders), 2) AS pct_of_total
FROM orders
ORDER BY pct_of_total DESC;
```

```
+────+────────+──────────────+
│ id │ amount │ pct_of_total │
+────+────────+──────────────+
│  3 │ 32000  │    29.36     │
│  7 │ 21000  │    19.27     │
│  1 │ 15000  │    13.76     │
│  5 │ 11000  │    10.09     │
│  8 │  9800  │     8.99     │
│  2 │  8500  │     7.80     │
│  6 │  6700  │     6.15     │
│  4 │  4200  │     3.85     │
└────┴────────┴──────────────┘
```

---

## 🔥 9. Subquery Performance — The Truth

### Non-Correlated Subqueries

```
┌─────────────────────────────────────────────────────────────────┐
│ Non-correlated subqueries run ONCE.                             │
│ The optimizer evaluates them first, caches the result,          │
│ and reuses it for every row of the outer query.                 │
│                                                                 │
│ Performance: Usually GOOD.                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Correlated Subqueries

```
┌─────────────────────────────────────────────────────────────────┐
│ Correlated subqueries run ONCE PER ROW of the outer query.      │
│                                                                 │
│ 100 outer rows = 100 subquery executions                        │
│ 1,000,000 outer rows = 1,000,000 executions 💀                  │
│                                                                 │
│ Performance: Can be TERRIBLE on large tables.                   │
│                                                                 │
│ Fix: Rewrite as JOIN or use window functions.                   │
│ Exception: EXISTS is often optimized well by the engine.        │
└─────────────────────────────────────────────────────────────────┘
```

### Optimization Cheat Sheet

```
┌────────────────────────┬───────────────────────────────────────────┐
│ Subquery Pattern       │ Better Alternative                        │
├────────────────────────┼───────────────────────────────────────────┤
│ WHERE IN (subquery)    │ JOIN (if subquery returns many rows)      │
│ WHERE NOT IN (subq)    │ NOT EXISTS (always prefer — NULL safe)    │
│ SELECT (correlated)    │ JOIN + aggregate or Window Function       │
│ FROM (derived table)   │ CTE for readability, same performance    │
│ Correlated + COUNT     │ Window function (ROW_NUMBER, RANK)       │
│ Scalar subquery in SET │ UPDATE with JOIN                          │
└────────────────────────┴───────────────────────────────────────────┘
```

> 💡 **Modern optimizers (PostgreSQL 12+, SQL Server 2019+, Oracle 12c+, MySQL 8.0+)** are very good at automatically transforming subqueries into JOINs. But don't rely on it — write efficient queries yourself.

---

## 🔥 10. Subquery vs JOIN vs EXISTS — Decision Matrix

```
"Show me customers who have orders"

-- Method 1: JOIN (may return duplicates if customer has multiple orders)
SELECT DISTINCT c.name FROM customers c 
JOIN orders o ON c.id = o.customer_id;

-- Method 2: IN
SELECT c.name FROM customers c
WHERE c.id IN (SELECT customer_id FROM orders);

-- Method 3: EXISTS (usually best for this pattern)
SELECT c.name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- All three return the same result. Which is best?
```

```
┌──────────────────┬──────────────────────────────────────────┐
│ Use              │ When                                      │
├──────────────────┼──────────────────────────────────────────┤
│ JOIN             │ You need columns from BOTH tables         │
│ IN               │ Simple list of values, small subquery     │
│ EXISTS           │ Just checking existence, NULL-safe        │
│ NOT EXISTS       │ Finding "not in" (ALWAYS prefer over NOT IN) │
│ Derived Table    │ Need to pre-aggregate or transform data  │
│ Scalar Subquery  │ Need ONE computed value per row           │
└──────────────────┴──────────────────────────────────────────┘
```

---

## 🧠 Common Mistakes with Subqueries

| # | Mistake | Problem | Fix |
|---|---------|---------|-----|
| 1 | NOT IN with NULLs | Returns empty result | Use NOT EXISTS |
| 2 | Scalar subquery returning multiple rows | Error | Add LIMIT 1, or use IN |
| 3 | Correlated subquery on large tables | Extremely slow | Rewrite as JOIN |
| 4 | Forgetting derived table alias | Syntax error | Always alias: `(...) AS alias` |
| 5 | Multiple scalar subqueries for same data | Redundant work | Use JOIN or CTE |
| 6 | Using subquery when JOIN suffices | Harder to read, sometimes slower | Prefer JOIN for multi-table data |
| 7 | Deeply nested subqueries (3+ levels) | Unreadable, hard to debug | Refactor with CTEs |

---

## ⚔️ Quick Challenge

**Q1:** Find products more expensive than the average product price
```sql
SELECT name, price 
FROM products 
WHERE price > (SELECT AVG(price) FROM products);
```

**Q2:** Find customers who have placed more orders than the average customer
```sql
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.name
HAVING COUNT(o.id) > (
    SELECT AVG(order_count) FROM (
        SELECT COUNT(*) AS order_count 
        FROM orders 
        GROUP BY customer_id
    ) sub
);
```

**Q3:** For each product, show its price and how it compares to the category average
```sql
SELECT 
    p.name,
    p.category,
    p.price,
    cat_avg.avg_price,
    p.price - cat_avg.avg_price AS diff_from_avg,
    CASE WHEN p.price > cat_avg.avg_price THEN 'Above' ELSE 'Below' END AS position
FROM products p
JOIN (
    SELECT category, AVG(price) AS avg_price
    FROM products
    GROUP BY category
) cat_avg ON p.category = cat_avg.category
ORDER BY p.category, diff_from_avg DESC;
```

**Q4:** Find customers who have ordered ALL products in the 'Audio' category
```sql
SELECT c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT p.id FROM products p
    WHERE p.category = 'Audio'
      AND NOT EXISTS (
          SELECT 1 FROM order_items oi
          JOIN orders o ON oi.order_id = o.id
          WHERE o.customer_id = c.id AND oi.product_id = p.id
      )
);
```

---

## 🎯 Key Takeaways

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ✅ Scalar subquery = one value. Use with =, >, <, etc.               │
│  ✅ Table subquery = multiple rows. Use with IN, ANY, ALL, EXISTS     │
│  ✅ Correlated subquery = references outer query. Runs per row.       │
│  ✅ Non-correlated = independent. Runs once. Usually faster.          │
│  ✅ NEVER use NOT IN when NULLs are possible — use NOT EXISTS         │
│  ✅ Derived tables = subquery in FROM. Must have an alias.            │
│  ✅ EXISTS short-circuits (stops at first match) — efficient          │
│  ✅ Rewrite correlated subqueries as JOINs for performance           │
│  ✅ CTEs are often more readable than nested subqueries              │
│  ✅ Modern optimizers handle subqueries well, but write clean SQL    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

> **← Previous:** [2.5 Aggregations & GROUP BY](./05-Aggregations.md)
> **Next →** [2.7 Window Functions — SQL Superpower](./07-Window-Functions.md)
