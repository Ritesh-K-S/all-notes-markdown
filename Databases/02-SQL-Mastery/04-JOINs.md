# 2.4 — JOINs: The Heart of Relational Data 🟢⭐🔥

> **"Data lives in separate tables. JOINs bring them together. Master JOINs, master SQL."**

---

## 🧭 What You'll Master in This Chapter

JOINs are the **single most important concept** in relational databases. After this chapter, you'll combine data across tables like a wizard — and finally understand why they're called "relational" databases.

```
customers  ──────┐
                  ├──── JOIN ───→  One beautiful result set
orders     ──────┘
```

---

## 📖 Why Do We Need JOINs?

In a well-designed database, data is **split across tables** (normalization). A customer's info is in one table, their orders in another. To answer "What did Ritesh order?", you need to **combine** both tables.

```
     customers                        orders
┌────┬──────────────┐      ┌────┬─────────────┬────────┐
│ id │ name         │      │ id │ customer_id │ amount │
├────┼──────────────┤      ├────┼─────────────┼────────┤
│  1 │ Ritesh Singh │◄─────│  1 │      1      │ 15000  │
│  2 │ Sarah Connor │◄─────│  2 │      2      │  8500  │
│  3 │ Akira Tanaka │      │  3 │      1      │ 32000  │
│  4 │ Maria Garcia │◄─────│  4 │      4      │  6700  │
│  5 │ John Smith   │      │  5 │      1      │  9800  │
└────┴──────────────┘      └────┴─────────────┴────────┘
          PK          ◄────────────── FK
```

**The question:** "Show me each customer with their orders."
**The tool:** JOIN.

---

## 🔥 The JOIN Family — Visual Map

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE JOIN FAMILY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   INNER JOIN      → Only matching rows from both tables         │
│   LEFT JOIN       → All from left + matching from right         │
│   RIGHT JOIN      → All from right + matching from left         │
│   FULL OUTER JOIN → All from both + NULLs where no match        │
│   CROSS JOIN      → Every row × every row (Cartesian product)   │
│   SELF JOIN       → A table joined to itself                    │
│   NATURAL JOIN    → Auto-match on same column names (avoid!)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔥 1. INNER JOIN — The Most Common JOIN

**Returns only rows that have a match in BOTH tables.**

```
         Table A              Table B
        ┌───────┐            ┌───────┐
        │       │            │       │
        │   ┌───┼────────────┼───┐   │
        │   │ ██│████████████│██ │   │
        │   │ ██│████████████│██ │   │   ██ = INNER JOIN result
        │   └───┼────────────┼───┘   │
        │       │            │       │
        └───────┘            └───────┘
```

### Syntax

```sql
-- Explicit JOIN syntax (ANSI standard — always use this)
SELECT c.name, o.amount, o.order_date
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;

-- The word "INNER" is optional (JOIN alone = INNER JOIN)
SELECT c.name, o.amount, o.order_date
FROM customers c
JOIN orders o ON c.id = o.customer_id;

-- Old-style implicit JOIN (avoid — harder to read, error-prone)
SELECT c.name, o.amount, o.order_date
FROM customers c, orders o
WHERE c.id = o.customer_id;
```

### Result

```
+──────────────+────────+────────────+
│ name         │ amount │ order_date │
+──────────────+────────+────────────+
│ Ritesh Singh │ 15000  │ 2024-01-10 │
│ Sarah Connor │  8500  │ 2024-01-15 │
│ Ritesh Singh │ 32000  │ 2024-02-20 │
│ Maria Garcia │  6700  │ 2024-03-05 │
│ Ritesh Singh │  9800  │ 2024-03-15 │
+──────────────+────────+────────────+

Notice: Akira Tanaka and John Smith are MISSING — they have no orders!
        This is INNER JOIN behavior — no match = not included.
```

### Multi-Table INNER JOIN

```sql
-- Join 3 tables: customers + orders + order_items + products
SELECT 
    c.name AS customer,
    o.order_date,
    p.name AS product,
    oi.quantity,
    oi.unit_price,
    oi.quantity * oi.unit_price AS line_total
FROM customers c
JOIN orders o      ON c.id = o.customer_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p    ON oi.product_id = p.id
WHERE o.status = 'delivered'
ORDER BY o.order_date DESC;
```

> 💡 **Reading Multi-Table JOINs:** Read them left to right as a chain:
> `customers` → joined with `orders` → joined with `order_items` → joined with `products`

---

## 🔥 2. LEFT JOIN (LEFT OUTER JOIN) — Keep Everything from the Left

**All rows from the LEFT table, plus matching rows from the right. NULLs where there's no match.**

```
         Table A              Table B
        ┌───────┐            ┌───────┐
        │ ██████│            │       │
        │ ██┌───┼────────────┼───┐   │
        │ ██│ ██│████████████│██ │   │
        │ ██│ ██│████████████│██ │   │   ██ = LEFT JOIN result
        │ ██└───┼────────────┼───┘   │
        │ ██████│            │       │
        └───────┘            └───────┘
    ALL from A            Only matching from B
```

```sql
SELECT c.name, o.amount, o.order_date, o.status
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;
```

### Result

```
+──────────────+────────+────────────+──────────+
│ name         │ amount │ order_date │ status   │
+──────────────+────────+────────────+──────────+
│ Ritesh Singh │ 15000  │ 2024-01-10 │ shipped  │
│ Ritesh Singh │ 32000  │ 2024-02-20 │ delivered│
│ Ritesh Singh │  9800  │ 2024-03-15 │ pending  │
│ Sarah Connor │  8500  │ 2024-01-15 │ delivered│
│ Maria Garcia │  6700  │ 2024-03-05 │ cancelled│
│ Akira Tanaka │  NULL  │ NULL       │ NULL     │  ← No orders!
│ John Smith   │  NULL  │ NULL       │ NULL     │  ← No orders!
+──────────────+────────+────────────+──────────+

Akira and John appear with NULLs — LEFT JOIN keeps them.
```

### LEFT JOIN — Find Rows With NO Match (Anti-Join)

One of the most powerful patterns in SQL:

```sql
-- Find customers who have NEVER placed an order
SELECT c.name, c.email
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;    -- ← The magic: filter for NULL on the right side
```

```
+──────────────+──────────────────+
│ name         │ email            │
+──────────────+──────────────────+
│ Akira Tanaka │ akira@email.com  │
│ John Smith   │ john@email.com   │
+──────────────+──────────────────+
```

> ⭐ **This pattern is incredibly common:**
> - Customers with no orders
> - Products never sold
> - Users who haven't logged in
> - Students not enrolled in any course

---

## 🔥 3. RIGHT JOIN (RIGHT OUTER JOIN) — Keep Everything from the Right

Mirror image of LEFT JOIN. All rows from the RIGHT table, NULLs for unmatched left rows.

```
         Table A              Table B
        ┌───────┐            ┌───────┐
        │       │            │██████ │
        │   ┌───┼────────────┼───┐██ │
        │   │ ██│████████████│██ │██ │
        │   │ ██│████████████│██ │██ │   ██ = RIGHT JOIN result
        │   └───┼────────────┼───┘██ │
        │       │            │██████ │
        └───────┘            └───────┘
    Only matching from A     ALL from B
```

```sql
SELECT c.name, o.amount, o.order_date
FROM customers c
RIGHT JOIN orders o ON c.id = o.customer_id;
```

> 💡 **In practice, almost nobody uses RIGHT JOIN.** You can always rewrite it as a LEFT JOIN by swapping the table order. LEFT JOIN is more readable.
> ```sql
> -- These are equivalent:
> SELECT * FROM A RIGHT JOIN B ON A.id = B.a_id;
> SELECT * FROM B LEFT JOIN A ON A.id = B.a_id;
> ```

---

## 🔥 4. FULL OUTER JOIN — Keep Everything from Both

**All rows from both tables. NULLs on both sides where there's no match.**

```
         Table A              Table B
        ┌───────┐            ┌───────┐
        │ ██████│            │██████ │
        │ ██┌───┼────────────┼───┐██ │
        │ ██│ ██│████████████│██ │██ │
        │ ██│ ██│████████████│██ │██ │   ██ = FULL OUTER JOIN result
        │ ██└───┼────────────┼───┘██ │
        │ ██████│            │██████ │
        └───────┘            └───────┘
    ALL from A                ALL from B
```

```sql
SELECT c.name, o.amount
FROM customers c
FULL OUTER JOIN orders o ON c.id = o.customer_id;
```

> ⚠️ **MySQL does NOT support FULL OUTER JOIN!** You must simulate it:
> ```sql
> -- MySQL workaround: UNION of LEFT and RIGHT JOIN
> SELECT c.name, o.amount
> FROM customers c LEFT JOIN orders o ON c.id = o.customer_id
> UNION
> SELECT c.name, o.amount
> FROM customers c RIGHT JOIN orders o ON c.id = o.customer_id;
> ```

### When to Use FULL OUTER JOIN?

- **Data reconciliation:** Compare two datasets, find what's in one but not the other
- **Finding orphans on BOTH sides:** Unmatched customers AND unmatched orders
- **Merging data from two sources:** Combine and identify gaps

```sql
-- Data reconciliation: Compare our records vs payment gateway
SELECT 
    o.id AS our_order_id,
    p.transaction_id AS gateway_id,
    CASE 
        WHEN o.id IS NULL     THEN 'Missing in our system'
        WHEN p.id IS NULL     THEN 'Missing in gateway'
        WHEN o.amount <> p.amount THEN 'Amount mismatch'
        ELSE 'Matched'
    END AS status
FROM orders o
FULL OUTER JOIN payments p ON o.id = p.order_id;
```

---

## 🔥 5. CROSS JOIN — Cartesian Product

**Every row from A combined with every row from B.** No join condition needed.

```
If A has 3 rows and B has 4 rows → Result has 3 × 4 = 12 rows
```

```sql
-- Explicit CROSS JOIN
SELECT c.name, p.name AS product
FROM customers c
CROSS JOIN products p;

-- Implicit (old-style)
SELECT c.name, p.name
FROM customers c, products p;    -- No WHERE = CROSS JOIN
```

### When is CROSS JOIN Useful?

```sql
-- Generate all combinations (e.g., size × color for a product)
SELECT s.size, c.color
FROM sizes s CROSS JOIN colors c;
-- Result: S-Red, S-Blue, S-Green, M-Red, M-Blue, M-Green, L-Red, L-Blue, L-Green

-- Generate a calendar (months × years)
SELECT y.year, m.month
FROM (SELECT 2023 AS year UNION SELECT 2024 UNION SELECT 2025) y
CROSS JOIN (SELECT 1 AS month UNION SELECT 2 UNION SELECT 3 /* ...12 */) m
ORDER BY y.year, m.month;

-- Generate a report with ALL month-product combinations (even months with zero sales)
SELECT 
    m.month_name,
    p.name AS product,
    COALESCE(SUM(oi.quantity), 0) AS total_sold
FROM months m
CROSS JOIN products p
LEFT JOIN orders o ON EXTRACT(MONTH FROM o.order_date) = m.month_num
LEFT JOIN order_items oi ON o.id = oi.order_id AND oi.product_id = p.id
GROUP BY m.month_name, p.name;
```

> ⚠️ **DANGER:** CROSS JOIN on large tables is catastrophic.
> 1,000 rows × 1,000 rows = 1,000,000 rows.
> 100,000 × 100,000 = **10 billion rows.** Your server will cry.

---

## 🔥 6. SELF JOIN — A Table Joined to Itself

When a table has a **relationship to itself** — like employees and their managers, or categories and subcategories.

```sql
-- Employee → Manager relationship
-- employees table:
-- +----+--------+------------+
-- | id | name   | manager_id |
-- +----+--------+------------+
-- |  1 | CEO    |    NULL    |
-- |  2 | VP Eng |     1      |
-- |  3 | Dev    |     2      |
-- |  4 | VP Mkt |     1      |
-- +----+--------+------------+

-- Show each employee with their manager's name
SELECT 
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Result:
-- +──────────+─────────+
-- │ employee │ manager │
-- +──────────+─────────+
-- │ CEO      │ NULL    │  ← No manager (top of hierarchy)
-- │ VP Eng   │ CEO     │
-- │ Dev      │ VP Eng  │
-- │ VP Mkt   │ CEO     │
-- +──────────+─────────+
```

### Self JOIN — Find Duplicates

```sql
-- Find customers with the same email (duplicates)
SELECT a.id, a.name, a.email
FROM customers a
JOIN customers b ON a.email = b.email AND a.id <> b.id;
```

### Self JOIN — Find Pairs

```sql
-- Find all pairs of products in the same category
SELECT 
    p1.name AS product_1,
    p2.name AS product_2,
    p1.category
FROM products p1
JOIN products p2 ON p1.category = p2.category AND p1.id < p2.id;
-- Use < (not <>) to avoid duplicate pairs (A,B and B,A)
```

---

## 🔥 7. Advanced JOIN Techniques

### JOIN with Multiple Conditions

```sql
-- JOIN on multiple columns
SELECT *
FROM order_items oi
JOIN product_prices pp 
    ON oi.product_id = pp.product_id 
    AND oi.order_date BETWEEN pp.start_date AND pp.end_date;

-- JOIN with additional filtering in ON vs WHERE
-- These produce DIFFERENT results with LEFT JOIN!
```

### ON vs WHERE in LEFT JOIN — Critical Difference!

```sql
-- Scenario: Get all customers and their 'delivered' orders

-- Version 1: Filter in ON (CORRECT for LEFT JOIN intent)
SELECT c.name, o.amount, o.status
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id AND o.status = 'delivered';
-- Result: ALL customers shown. Unmatched get NULLs. 
-- Only 'delivered' orders are matched.

-- Version 2: Filter in WHERE (WRONG — turns LEFT JOIN into INNER JOIN)
SELECT c.name, o.amount, o.status
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'delivered';
-- Result: Only customers WITH delivered orders! 
-- NULLs are filtered out by WHERE, losing the LEFT JOIN benefit.
```

```
                    ON clause filter              WHERE clause filter
                    ─────────────────             ──────────────────
LEFT JOIN keeps     ✅ Unmatched rows survive     ❌ NULLs filtered out
                    (NULLs in right table)        (effectively INNER JOIN)

INNER JOIN          No difference                 No difference
                    (both behave the same)        (both behave the same)
```

> ⭐ **Rule:** With LEFT/RIGHT/FULL JOIN, put filters for the **outer table** in ON, not WHERE (unless you intentionally want to exclude NULLs).

### Non-Equi JOINs (JOIN Without =)

```sql
-- Range JOIN: Find which salary band each employee falls into
SELECT e.name, e.salary, b.band_name
FROM employees e
JOIN salary_bands b ON e.salary BETWEEN b.min_salary AND b.max_salary;

-- Comparison JOIN: Find products more expensive than average in their category
SELECT p1.name, p1.price, p1.category
FROM products p1
JOIN (
    SELECT category, AVG(price) AS avg_price 
    FROM products GROUP BY category
) p2 ON p1.category = p2.category AND p1.price > p2.avg_price;
```

### LATERAL JOIN / CROSS APPLY — The Power Move

```sql
-- Get top 3 orders for each customer (can't easily do with regular JOIN)

-- PostgreSQL: LATERAL
SELECT c.name, top_orders.*
FROM customers c
LEFT JOIN LATERAL (
    SELECT order_date, amount
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY amount DESC
    LIMIT 3
) top_orders ON TRUE;

-- SQL Server: CROSS APPLY / OUTER APPLY
SELECT c.name, top_orders.*
FROM customers c
OUTER APPLY (
    SELECT TOP 3 order_date, amount
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY amount DESC
) top_orders;
```

> 💡 **CROSS APPLY** = INNER JOIN behavior (skips customers with no orders)
> **OUTER APPLY** = LEFT JOIN behavior (keeps customers with no orders, NULLs)

---

## 🔥 8. NATURAL JOIN — Know It, Avoid It

```sql
-- NATURAL JOIN automatically joins on columns with the same name
SELECT * FROM customers NATURAL JOIN orders;
-- Joins on ALL columns with matching names between both tables
```

> ⚠️ **Why you should NEVER use NATURAL JOIN:**
> 1. If both tables have a column `id`, it joins on `id` instead of `customer_id` — wrong results!
> 2. Adding a column to any table can silently change the join condition
> 3. No explicit ON clause — impossible to understand at a glance
> 4. Code becomes fragile and unpredictable
> 
> **Always use explicit ON conditions.**

---

## 🔥 9. JOIN Performance — What Really Happens

### How Databases Execute JOINs

```
┌─────────────────────────────────────────────────────────────────────┐
│                  JOIN ALGORITHMS (Internal)                         │
├─────────────────────┬───────────────────────────────────────────────┤
│ Nested Loop Join    │ For each row in A, scan matching rows in B   │
│                     │ Best for: small tables, indexed joins        │
│                     │ Time: O(n × m) without index, O(n × log m)  │
│                     │        with index                            │
├─────────────────────┼───────────────────────────────────────────────┤
│ Hash Join           │ Build hash table from smaller table,         │
│                     │ probe with larger table                      │
│                     │ Best for: large tables, equi-joins, no index │
│                     │ Time: O(n + m)                               │
├─────────────────────┼───────────────────────────────────────────────┤
│ Merge Join          │ Sort both tables, merge like a zipper        │
│ (Sort-Merge)        │ Best for: pre-sorted data, range joins       │
│                     │ Time: O(n log n + m log m) for sort          │
│                     │        + O(n + m) for merge                  │
└─────────────────────┴───────────────────────────────────────────────┘
```

### JOIN Performance Tips

```sql
-- ✅ DO: Index the JOIN columns
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- ✅ DO: Filter early (reduce rows before joining)
SELECT c.name, o.amount
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.order_date >= '2024-01-01';    -- Filter reduces rows to join

-- ❌ DON'T: JOIN then filter on a function
SELECT c.name, o.amount
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE YEAR(o.order_date) = 2024;    -- Function on column = no index use!

-- ✅ DO: JOIN fewer tables when possible
-- If you only need customer name and order amount, don't join order_items too

-- ✅ DO: Use EXISTS instead of JOIN when you only need to check existence
-- Instead of:
SELECT DISTINCT c.name FROM customers c
JOIN orders o ON c.id = o.customer_id;    -- JOIN then DISTINCT = wasteful

-- Better:
SELECT c.name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

---

## 🔥 10. The Complete JOIN Cheat Sheet

### Visual Summary

```
Given:
  customers: {1, 2, 3, 4, 5}  (Ritesh, Sarah, Akira, Maria, John)
  orders:    customer_ids {1, 1, 1, 2, 4, 6}  (6 = Priya, who IS in customers)

┌─────────────────────┬────────────────────────────────────────────────┐
│ JOIN Type           │ Result                                         │
├─────────────────────┼────────────────────────────────────────────────┤
│ INNER JOIN          │ Ritesh(×3), Sarah(×1), Maria(×1), Priya(×1)   │
│                     │ = Only matching                                │
├─────────────────────┼────────────────────────────────────────────────┤
│ LEFT JOIN           │ All above + Akira(NULL), John(NULL)            │
│                     │ = All customers, matched or not                │
├─────────────────────┼────────────────────────────────────────────────┤
│ RIGHT JOIN          │ All orders + their customers                   │
│                     │ = All orders, matched or not                   │
├─────────────────────┼────────────────────────────────────────────────┤
│ FULL OUTER JOIN     │ All customers + All orders                     │
│                     │ = Everything, NULLs on both sides              │
├─────────────────────┼────────────────────────────────────────────────┤
│ CROSS JOIN          │ 5 customers × 6 orders = 30 rows              │
│                     │ = Every possible combination                   │
├─────────────────────┼────────────────────────────────────────────────┤
│ LEFT Anti-Join      │ Akira, John                                    │
│ (LEFT + WHERE NULL) │ = Customers with NO orders                     │
└─────────────────────┴────────────────────────────────────────────────┘
```

### When to Use Which JOIN?

```
┌──────────────────────────┬────────────────────────────────────────────┐
│ Use Case                 │ JOIN Type                                  │
├──────────────────────────┼────────────────────────────────────────────┤
│ Get related data         │ INNER JOIN                                 │
│ Include unmatched (left) │ LEFT JOIN                                  │
│ Find "not in" / orphans  │ LEFT JOIN + WHERE right.id IS NULL         │
│ Compare two datasets     │ FULL OUTER JOIN                            │
│ Generate combinations    │ CROSS JOIN                                 │
│ Hierarchical data        │ SELF JOIN                                  │
│ Top-N per group          │ LATERAL JOIN / CROSS APPLY                 │
│ Check existence only     │ EXISTS (not JOIN)                          │
└──────────────────────────┴────────────────────────────────────────────┘
```

---

## 🧠 Common JOIN Mistakes

| # | Mistake | Problem | Fix |
|---|---------|---------|-----|
| 1 | Missing ON condition | Accidental CROSS JOIN (millions of rows) | Always specify ON |
| 2 | Wrong ON condition | Incorrect results, more rows than expected | Verify PK-FK relationship |
| 3 | Using WHERE instead of ON with LEFT JOIN | Turns it into INNER JOIN | Put outer table filters in ON |
| 4 | Forgetting table aliases | Ambiguous column errors | Always alias tables |
| 5 | Using NATURAL JOIN | Fragile, unpredictable | Use explicit ON |
| 6 | Not indexing FK columns | Slow JOINs | `CREATE INDEX` on FK columns |
| 7 | JOINing too many tables | Exponential row explosion | Only JOIN what you need |
| 8 | DISTINCT to "fix" too many rows | Hiding a bad JOIN (usually wrong ON) | Fix the JOIN, don't mask with DISTINCT |
| 9 | Cartesian product from multiple JOINs | Joining A→B and A→C when B and C both multiply rows | Use subqueries or separate queries |

### The Row Multiplication Trap

```sql
-- ⚠️ DANGER: This query can produce WAY more rows than expected!
-- If a customer has 3 orders AND 2 addresses, you get 3×2 = 6 rows per customer
SELECT c.name, o.amount, a.city
FROM customers c
JOIN orders o ON c.id = o.customer_id
JOIN addresses a ON c.id = a.customer_id;
-- Each customer row multiplied by (orders × addresses)

-- ✅ FIX: Use subqueries or aggregate before joining
SELECT c.name, order_summary.total, a.city
FROM customers c
JOIN (
    SELECT customer_id, SUM(amount) AS total 
    FROM orders GROUP BY customer_id
) order_summary ON c.id = order_summary.customer_id
JOIN addresses a ON c.id = a.customer_id AND a.is_primary = TRUE;
```

---

## ⚔️ Quick Challenge

**Q1:** List all products that have never been ordered
```sql
SELECT p.name, p.price
FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
WHERE oi.id IS NULL;
```

**Q2:** Show every customer with their total spending (including those who spent $0)
```sql
SELECT c.name, COALESCE(SUM(o.amount), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.name
ORDER BY total_spent DESC;
```

**Q3:** Find employees who earn more than their manager
```sql
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

**Q4:** Generate a report showing every month × product combination with sales count
```sql
SELECT m.month_name, p.name, COALESCE(COUNT(oi.id), 0) AS units_sold
FROM months m
CROSS JOIN products p
LEFT JOIN orders o ON EXTRACT(MONTH FROM o.order_date) = m.month_num
LEFT JOIN order_items oi ON o.id = oi.order_id AND oi.product_id = p.id
GROUP BY m.month_name, p.name
ORDER BY m.month_num, p.name;
```

---

## 🎯 Key Takeaways

```
┌─────────────────────────────────────────────────────────────────────┐
│  ✅ INNER JOIN = only matching rows                                │
│  ✅ LEFT JOIN = all from left + matching from right                │
│  ✅ LEFT JOIN + WHERE NULL = anti-join (find orphans)              │
│  ✅ FULL OUTER JOIN = everything from both (MySQL: use UNION)      │
│  ✅ CROSS JOIN = Cartesian product (use carefully!)                │
│  ✅ SELF JOIN = table joined to itself (hierarchies, comparisons)  │
│  ✅ Put outer-table filters in ON, not WHERE                       │
│  ✅ Index FK columns for fast JOINs                                │
│  ✅ Avoid NATURAL JOIN — always use explicit ON                    │
│  ✅ Watch for row multiplication when joining multiple tables      │
│  ✅ Use EXISTS instead of JOIN when checking existence only        │
└─────────────────────────────────────────────────────────────────────┘
```

---

> **← Previous:** [2.3 DML — INSERT, UPDATE, DELETE, MERGE](./03-DML.md)
> **Next →** [2.5 Aggregations & GROUP BY](./05-Aggregations.md)
