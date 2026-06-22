# 🏋️ Chapter 8.4 — SQL Query Practice — 50 Must-Solve Problems

> **"You don't learn SQL by reading about it. You learn SQL by writing it, breaking it, and fixing it — over and over again."** — Every DBA who ever lived

---

## 📌 Metadata

| Field | Value |
|-------|-------|
| **Level** | 🟢 Basic → 🔴 Advanced |
| **Time to Master** | ~8-10 hours (solve, don't just read!) |
| **Prerequisites** | SQL Mastery (Part 2A), Window Functions (2.7) |
| **Where to practice** | PostgreSQL, MySQL, SQL Server, or [db-fiddle.com](https://db-fiddle.com) |

---

## 🎯 What You'll Master

- ✅ 50 real SQL problems — the exact patterns asked in interviews
- ✅ Multiple solution approaches for each problem
- ✅ Window functions, CTEs, self-joins, gaps & islands
- ✅ Progressive difficulty (warm up → brain melters)
- ✅ Patterns that transfer to ANY SQL interview

---

## ⚠️ IMPORTANT: Don't Just Read — SOLVE First!

```
┌──────────────────────────────────────────────────────────────┐
│  THE ONLY WAY TO GET BETTER AT SQL:                         │
│                                                              │
│  1. Read the problem                                        │
│  2. TRY to solve it yourself (even if wrong!)               │
│  3. THEN check the solution                                 │
│  4. If your solution differs → understand WHY               │
│  5. Re-solve from scratch the next day                      │
│                                                              │
│  Reading solutions without trying = ZERO learning           │
└──────────────────────────────────────────────────────────────┘
```

---

## 📋 Practice Database Schema

All problems use this schema. **Set it up before you start!**

```sql
-- ============================================
-- PRACTICE DATABASE SETUP
-- ============================================

-- Employees
CREATE TABLE employees (
    emp_id      INT PRIMARY KEY,
    emp_name    VARCHAR(100),
    department  VARCHAR(50),
    salary      DECIMAL(10,2),
    manager_id  INT REFERENCES employees(emp_id),
    hire_date   DATE,
    city        VARCHAR(50)
);

INSERT INTO employees VALUES
(1,  'Alice',   'Engineering', 120000, NULL, '2019-01-15', 'New York'),
(2,  'Bob',     'Engineering', 110000, 1,    '2019-03-20', 'San Francisco'),
(3,  'Charlie', 'Marketing',   90000,  1,    '2020-06-10', 'New York'),
(4,  'Diana',   'Engineering', 130000, 1,    '2018-11-01', 'Seattle'),
(5,  'Eve',     'Marketing',   85000,  3,    '2021-02-14', 'New York'),
(6,  'Frank',   'Sales',       95000,  NULL, '2020-08-22', 'Chicago'),
(7,  'Grace',   'Sales',       88000,  6,    '2021-05-30', 'Chicago'),
(8,  'Hank',    'Engineering', 105000, 2,    '2022-01-10', 'San Francisco'),
(9,  'Ivy',     'Marketing',   92000,  3,    '2020-09-15', 'Seattle'),
(10, 'Jack',    'Sales',       78000,  6,    '2023-03-01', 'New York');

-- Orders
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    emp_id      INT REFERENCES employees(emp_id),
    product     VARCHAR(100),
    amount      DECIMAL(10,2),
    order_date  DATE,
    status      VARCHAR(20)
);

INSERT INTO orders VALUES
(101, 1, 'Laptop',      1200.00, '2024-01-15', 'completed'),
(102, 2, 'Monitor',      450.00, '2024-01-20', 'completed'),
(103, 1, 'Keyboard',      75.00, '2024-02-10', 'completed'),
(104, 3, 'Mouse',         25.00, '2024-02-15', 'cancelled'),
(105, 4, 'Laptop',      1200.00, '2024-03-01', 'completed'),
(106, 2, 'Headphones',   150.00, '2024-03-10', 'completed'),
(107, 5, 'Webcam',        80.00, '2024-03-15', 'pending'),
(108, 1, 'Desk',         500.00, '2024-04-01', 'completed'),
(109, 6, 'Chair',        350.00, '2024-04-10', 'completed'),
(110, 3, 'Monitor',      450.00, '2024-04-20', 'completed'),
(111, 7, 'Laptop',      1200.00, '2024-05-01', 'pending'),
(112, 2, 'Desk',         500.00, '2024-05-15', 'completed'),
(113, 8, 'Chair',        350.00, '2024-06-01', 'completed'),
(114, 4, 'Monitor',      450.00, '2024-06-15', 'cancelled'),
(115, 1, 'Laptop',      1200.00, '2024-07-01', 'completed');

-- Daily logins (for streak/gap problems)
CREATE TABLE daily_logins (
    user_id     INT,
    login_date  DATE,
    PRIMARY KEY (user_id, login_date)
);

INSERT INTO daily_logins VALUES
(1, '2024-01-01'), (1, '2024-01-02'), (1, '2024-01-03'),
(1, '2024-01-06'), (1, '2024-01-07'),
(1, '2024-01-10'),
(2, '2024-01-01'), (2, '2024-01-02'),
(2, '2024-01-05'), (2, '2024-01-06'), (2, '2024-01-07'), (2, '2024-01-08'),
(3, '2024-01-01'), (3, '2024-01-03'), (3, '2024-01-05');

-- Monthly revenue (for trend analysis)
CREATE TABLE monthly_revenue (
    month_date  DATE PRIMARY KEY,
    revenue     DECIMAL(12,2)
);

INSERT INTO monthly_revenue VALUES
('2024-01-01', 150000), ('2024-02-01', 165000), ('2024-03-01', 142000),
('2024-04-01', 180000), ('2024-05-01', 175000), ('2024-06-01', 195000),
('2024-07-01', 210000), ('2024-08-01', 188000), ('2024-09-01', 220000),
('2024-10-01', 205000), ('2024-11-01', 240000), ('2024-12-01', 260000);
```

---

# 🟢 WARM UP: Problems 1–10 (Basic)

> _Get your SQL muscles moving. Should take 1-3 minutes each._

---

### Problem 1: Find all employees in New York

```
DIFFICULTY: ★☆☆☆☆
TOPIC: Basic WHERE
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT * FROM employees WHERE city = 'New York';
```

</details>

---

### Problem 2: Find the highest salary in each department

```
DIFFICULTY: ★☆☆☆☆
TOPIC: GROUP BY, MAX
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT department, MAX(salary) AS max_salary
FROM employees
GROUP BY department;
```

</details>

---

### Problem 3: Find employees who have placed at least 2 orders

```
DIFFICULTY: ★★☆☆☆
TOPIC: JOIN, GROUP BY, HAVING
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT e.emp_name, COUNT(o.order_id) AS order_count
FROM employees e
JOIN orders o ON e.emp_id = o.emp_id
GROUP BY e.emp_name
HAVING COUNT(o.order_id) >= 2;
```

</details>

---

### Problem 4: Find employees who have NEVER placed an order

```
DIFFICULTY: ★★☆☆☆
TOPIC: LEFT JOIN, IS NULL (or NOT EXISTS)
```

<details>
<summary>💡 Solution</summary>

```sql
-- Method 1: LEFT JOIN
SELECT e.emp_name
FROM employees e
LEFT JOIN orders o ON e.emp_id = o.emp_id
WHERE o.order_id IS NULL;

-- Method 2: NOT EXISTS (often faster)
SELECT e.emp_name
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.emp_id = e.emp_id
);

-- Method 3: NOT IN (⚠️ careful with NULLs)
SELECT emp_name FROM employees
WHERE emp_id NOT IN (SELECT emp_id FROM orders WHERE emp_id IS NOT NULL);
```

</details>

---

### Problem 5: Find the total order amount per employee, including those with zero orders

```
DIFFICULTY: ★★☆☆☆
TOPIC: LEFT JOIN, COALESCE, SUM
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    e.emp_name,
    COALESCE(SUM(o.amount), 0) AS total_amount,
    COUNT(o.order_id) AS order_count
FROM employees e
LEFT JOIN orders o ON e.emp_id = o.emp_id
GROUP BY e.emp_name
ORDER BY total_amount DESC;
```

</details>

---

### Problem 6: Find the second highest salary

```
DIFFICULTY: ★★☆☆☆
TOPIC: Window Function or Subquery
```

<details>
<summary>💡 Solution</summary>

```sql
-- Method 1: DENSE_RANK (handles ties)
SELECT DISTINCT salary
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk = 2;

-- Method 2: Subquery
SELECT MAX(salary) FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Method 3: OFFSET (PostgreSQL/MySQL)
SELECT DISTINCT salary FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

</details>

---

### Problem 7: Find employees whose salary is above the average salary

```
DIFFICULTY: ★★☆☆☆
TOPIC: Subquery
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT emp_name, salary, department
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees)
ORDER BY salary DESC;
```

</details>

---

### Problem 8: List departments that have more than 2 employees

```
DIFFICULTY: ★☆☆☆☆
TOPIC: GROUP BY, HAVING
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT department, COUNT(*) AS emp_count
FROM employees
GROUP BY department
HAVING COUNT(*) > 2;
```

</details>

---

### Problem 9: Find the most expensive order for each product

```
DIFFICULTY: ★★☆☆☆
TOPIC: GROUP BY or Window Function
```

<details>
<summary>💡 Solution</summary>

```sql
-- Method 1: GROUP BY
SELECT product, MAX(amount) AS max_amount
FROM orders
GROUP BY product;

-- Method 2: Window function (includes full row details)
SELECT *
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY product ORDER BY amount DESC) AS rn
    FROM orders
) ranked
WHERE rn = 1;
```

</details>

---

### Problem 10: Find the employee with the longest tenure (earliest hire date)

```
DIFFICULTY: ★☆☆☆☆
TOPIC: ORDER BY, LIMIT
```

<details>
<summary>💡 Solution</summary>

```sql
-- PostgreSQL/MySQL
SELECT emp_name, hire_date 
FROM employees 
ORDER BY hire_date ASC 
LIMIT 1;

-- SQL Server
SELECT TOP 1 emp_name, hire_date 
FROM employees 
ORDER BY hire_date ASC;

-- Window function (handles ties)
SELECT emp_name, hire_date
FROM (
    SELECT *, RANK() OVER (ORDER BY hire_date ASC) AS rnk
    FROM employees
) ranked
WHERE rnk = 1;
```

</details>

---

# 🟡 LEVEL UP: Problems 11–25 (Intermediate)

> _JOINs, window functions, and conditional logic. You should spend 3-8 minutes each._

---

### Problem 11: Find employees who earn more than their manager

```
DIFFICULTY: ★★★☆☆
TOPIC: Self-JOIN
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    e.emp_name AS employee,
    e.salary AS emp_salary,
    m.emp_name AS manager,
    m.salary AS mgr_salary
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```

</details>

---

### Problem 12: Find the top earner in EACH department (with name)

```
DIFFICULTY: ★★★☆☆
TOPIC: Window Function, PARTITION BY
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT emp_name, department, salary
FROM (
    SELECT *,
        RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk = 1;
```

</details>

---

### Problem 13: Calculate a running total of order amounts by date

```
DIFFICULTY: ★★★☆☆
TOPIC: Window Function, Running Total
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    order_date,
    product,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total,
    SUM(amount) OVER (
        ORDER BY order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total_explicit
FROM orders
WHERE status = 'completed'
ORDER BY order_date;
```

</details>

---

### Problem 14: Find employees in departments with an average salary above $95,000

```
DIFFICULTY: ★★★☆☆
TOPIC: Subquery or CTE with JOIN
```

<details>
<summary>💡 Solution</summary>

```sql
-- Method 1: Subquery
SELECT emp_name, department, salary
FROM employees
WHERE department IN (
    SELECT department FROM employees
    GROUP BY department
    HAVING AVG(salary) > 95000
);

-- Method 2: CTE + JOIN
WITH high_avg_depts AS (
    SELECT department, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department
    HAVING AVG(salary) > 95000
)
SELECT e.emp_name, e.department, e.salary, d.avg_sal
FROM employees e
JOIN high_avg_depts d ON e.department = d.department;

-- Method 3: Window function
SELECT emp_name, department, salary, dept_avg
FROM (
    SELECT *, AVG(salary) OVER (PARTITION BY department) AS dept_avg
    FROM employees
) sub
WHERE dept_avg > 95000;
```

</details>

---

### Problem 15: Show each employee's salary as a percentage of their department total

```
DIFFICULTY: ★★★☆☆
TOPIC: Window Function, Percentage
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    emp_name,
    department,
    salary,
    SUM(salary) OVER (PARTITION BY department) AS dept_total,
    ROUND(salary * 100.0 / SUM(salary) OVER (PARTITION BY department), 2) AS pct_of_dept
FROM employees
ORDER BY department, pct_of_dept DESC;
```

</details>

---

### Problem 16: Find the month-over-month revenue growth rate

```
DIFFICULTY: ★★★☆☆
TOPIC: LAG, Window Function
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    month_date,
    revenue,
    LAG(revenue) OVER (ORDER BY month_date) AS prev_month,
    revenue - LAG(revenue) OVER (ORDER BY month_date) AS change,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month_date)) * 100.0 
        / LAG(revenue) OVER (ORDER BY month_date), 2
    ) AS growth_pct
FROM monthly_revenue;
```

</details>

---

### Problem 17: Find duplicate employee names (if any exist)

```
DIFFICULTY: ★★☆☆☆
TOPIC: GROUP BY, HAVING
```

<details>
<summary>💡 Solution</summary>

```sql
-- Find which names are duplicated
SELECT emp_name, COUNT(*) AS cnt
FROM employees
GROUP BY emp_name
HAVING COUNT(*) > 1;

-- Show all records that are duplicates
SELECT *
FROM employees
WHERE emp_name IN (
    SELECT emp_name FROM employees
    GROUP BY emp_name
    HAVING COUNT(*) > 1
);
```

</details>

---

### Problem 18: Pivot — Show total orders per product per status

```
DIFFICULTY: ★★★☆☆
TOPIC: Conditional Aggregation (CASE)
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    product,
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'completed' THEN 1 END) AS completed,
    COUNT(CASE WHEN status = 'pending' THEN 1 END) AS pending,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled,
    SUM(CASE WHEN status = 'completed' THEN amount ELSE 0 END) AS completed_revenue
FROM orders
GROUP BY product
ORDER BY total_orders DESC;
```

</details>

---

### Problem 19: Find the employee with the highest order total (including name)

```
DIFFICULTY: ★★★☆☆
TOPIC: JOIN, GROUP BY, ORDER BY
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT e.emp_name, SUM(o.amount) AS total_orders
FROM employees e
JOIN orders o ON e.emp_id = o.emp_id
WHERE o.status = 'completed'
GROUP BY e.emp_name
ORDER BY total_orders DESC
LIMIT 1;

-- More robust (handles ties):
WITH emp_totals AS (
    SELECT e.emp_name, SUM(o.amount) AS total_orders,
           RANK() OVER (ORDER BY SUM(o.amount) DESC) AS rnk
    FROM employees e
    JOIN orders o ON e.emp_id = o.emp_id
    WHERE o.status = 'completed'
    GROUP BY e.emp_name
)
SELECT emp_name, total_orders FROM emp_totals WHERE rnk = 1;
```

</details>

---

### Problem 20: Find the average time between consecutive orders per employee

```
DIFFICULTY: ★★★★☆
TOPIC: LEAD/LAG, Date Arithmetic
```

<details>
<summary>💡 Solution</summary>

```sql
WITH order_gaps AS (
    SELECT 
        emp_id,
        order_date,
        LEAD(order_date) OVER (PARTITION BY emp_id ORDER BY order_date) AS next_order,
        LEAD(order_date) OVER (PARTITION BY emp_id ORDER BY order_date) - order_date AS days_gap
    FROM orders
)
SELECT 
    e.emp_name,
    ROUND(AVG(og.days_gap), 1) AS avg_days_between_orders,
    MIN(og.days_gap) AS min_gap,
    MAX(og.days_gap) AS max_gap,
    COUNT(og.days_gap) AS num_gaps
FROM order_gaps og
JOIN employees e ON og.emp_id = e.emp_id
WHERE og.days_gap IS NOT NULL
GROUP BY e.emp_name
ORDER BY avg_days_between_orders;
```

</details>

---

### Problem 21: Create an employee hierarchy report (with level)

```
DIFFICULTY: ★★★★☆
TOPIC: Recursive CTE
```

<details>
<summary>💡 Solution</summary>

```sql
WITH RECURSIVE org_tree AS (
    -- Anchor: Top-level (no manager)
    SELECT emp_id, emp_name, manager_id, 1 AS level,
           CAST(emp_name AS VARCHAR(500)) AS path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: Direct reports
    SELECT e.emp_id, e.emp_name, e.manager_id, ot.level + 1,
           CAST(ot.path || ' → ' || e.emp_name AS VARCHAR(500))
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.emp_id
)
SELECT 
    REPEAT('  ', level - 1) || emp_name AS org_chart,
    level,
    path
FROM org_tree
ORDER BY path;
```

```
Expected Output:
org_chart           | level | path
Alice               | 1     | Alice
  Bob               | 2     | Alice → Bob
    Hank            | 3     | Alice → Bob → Hank
  Charlie           | 2     | Alice → Charlie
    Eve             | 3     | Alice → Charlie → Eve
    Ivy             | 3     | Alice → Charlie → Ivy
  Diana             | 2     | Alice → Diana
Frank               | 1     | Frank
  Grace             | 2     | Frank → Grace
  Jack              | 2     | Frank → Jack
```

</details>

---

### Problem 22: Rank employees within their department by salary

```
DIFFICULTY: ★★★☆☆
TOPIC: Window Functions (RANK, DENSE_RANK, ROW_NUMBER)
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    emp_name,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rnk,
    NTILE(2) OVER (PARTITION BY department ORDER BY salary DESC) AS salary_half
FROM employees
ORDER BY department, salary DESC;
```

</details>

---

### Problem 23: Find the cumulative percentage of total revenue by month

```
DIFFICULTY: ★★★☆☆
TOPIC: Window Function, Cumulative Distribution
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    month_date,
    revenue,
    SUM(revenue) OVER (ORDER BY month_date) AS cumulative_revenue,
    SUM(revenue) OVER () AS total_annual,
    ROUND(
        SUM(revenue) OVER (ORDER BY month_date) * 100.0 
        / SUM(revenue) OVER (), 2
    ) AS cumulative_pct
FROM monthly_revenue;
```

</details>

---

### Problem 24: Find employees who were hired in the same month as another employee

```
DIFFICULTY: ★★★☆☆
TOPIC: Self-JOIN, Date Functions
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT DISTINCT
    a.emp_name AS employee1,
    b.emp_name AS employee2,
    EXTRACT(YEAR FROM a.hire_date) AS hire_year,
    EXTRACT(MONTH FROM a.hire_date) AS hire_month
FROM employees a
JOIN employees b 
    ON EXTRACT(YEAR FROM a.hire_date) = EXTRACT(YEAR FROM b.hire_date)
    AND EXTRACT(MONTH FROM a.hire_date) = EXTRACT(MONTH FROM b.hire_date)
    AND a.emp_id < b.emp_id  -- Avoid self-pairing and duplicates
ORDER BY hire_year, hire_month;
```

</details>

---

### Problem 25: Calculate a 3-month moving average of revenue

```
DIFFICULTY: ★★★☆☆
TOPIC: Window Function, Frame Clause
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    month_date,
    revenue,
    ROUND(AVG(revenue) OVER (
        ORDER BY month_date 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_3m,
    ROUND(AVG(revenue) OVER (
        ORDER BY month_date 
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ), 2) AS centered_avg_3m
FROM monthly_revenue;
```

</details>

---

# 🔴 CHALLENGE: Problems 26–40 (Advanced)

> _These are the problems that separate the top 10%. Window functions, gaps & islands, and creative solutions. Budget 5-15 minutes each._

---

### Problem 26: ⭐ Gaps and Islands — Find consecutive login streaks

```
DIFFICULTY: ★★★★☆
TOPIC: Gaps and Islands (Classic Pattern!)

For each user, find their consecutive login streaks.
```

<details>
<summary>💡 Solution</summary>

```sql
WITH numbered AS (
    SELECT 
        user_id,
        login_date,
        login_date - (ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY login_date
        ))::int AS grp
    FROM daily_logins
)
SELECT 
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*) AS streak_length
FROM numbered
GROUP BY user_id, grp
HAVING COUNT(*) >= 2
ORDER BY user_id, streak_start;
```

```
How it works:
  User 1 dates:   Jan 1,  Jan 2,  Jan 3,  Jan 6,  Jan 7,  Jan 10
  ROW_NUMBER:       1,      2,      3,      4,      5,      6
  date - RN:      Dec31,  Dec31,  Dec31,  Jan2,   Jan2,   Jan4
                  ─────group 1─────  ──group 2──  ─group 3─

Result:
  User 1: Jan 1 → Jan 3 (3 days), Jan 6 → Jan 7 (2 days)
  User 2: Jan 1 → Jan 2 (2 days), Jan 5 → Jan 8 (4 days)
```

</details>

---

### Problem 27: Find the longest login streak for each user

```
DIFFICULTY: ★★★★☆
TOPIC: Gaps and Islands + MAX
```

<details>
<summary>💡 Solution</summary>

```sql
WITH streaks AS (
    SELECT 
        user_id,
        login_date,
        login_date - (ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY login_date
        ))::int AS grp
    FROM daily_logins
),
streak_lengths AS (
    SELECT 
        user_id,
        MIN(login_date) AS streak_start,
        MAX(login_date) AS streak_end,
        COUNT(*) AS streak_length
    FROM streaks
    GROUP BY user_id, grp
)
SELECT DISTINCT ON (user_id)
    user_id, streak_start, streak_end, streak_length
FROM streak_lengths
ORDER BY user_id, streak_length DESC;

-- Alternative without DISTINCT ON (all databases):
SELECT * FROM (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY streak_length DESC) AS rn
    FROM streak_lengths
) ranked WHERE rn = 1;
```

</details>

---

### Problem 28: Find gaps in login dates per user

```
DIFFICULTY: ★★★★☆
TOPIC: LEAD, Gap Detection
```

<details>
<summary>💡 Solution</summary>

```sql
WITH login_gaps AS (
    SELECT 
        user_id,
        login_date,
        LEAD(login_date) OVER (PARTITION BY user_id ORDER BY login_date) AS next_login,
        LEAD(login_date) OVER (PARTITION BY user_id ORDER BY login_date) - login_date AS gap_days
    FROM daily_logins
)
SELECT 
    user_id,
    login_date AS last_active,
    next_login AS returned_on,
    gap_days AS days_absent
FROM login_gaps
WHERE gap_days > 1
ORDER BY user_id, login_date;
```

</details>

---

### Problem 29: Find the top 2 products by revenue for each month

```
DIFFICULTY: ★★★★☆
TOPIC: Window Function, DATE_TRUNC
```

<details>
<summary>💡 Solution</summary>

```sql
WITH monthly_product AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        product,
        SUM(amount) AS total_revenue,
        COUNT(*) AS order_count,
        DENSE_RANK() OVER (
            PARTITION BY DATE_TRUNC('month', order_date) 
            ORDER BY SUM(amount) DESC
        ) AS rnk
    FROM orders
    WHERE status = 'completed'
    GROUP BY DATE_TRUNC('month', order_date), product
)
SELECT month, product, total_revenue, order_count, rnk
FROM monthly_product
WHERE rnk <= 2
ORDER BY month, rnk;
```

</details>

---

### Problem 30: Find employees whose salary is within 10% of the department average

```
DIFFICULTY: ★★★☆☆
TOPIC: Window Function, Range Comparison
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT emp_name, department, salary, dept_avg,
    ROUND((salary - dept_avg) * 100.0 / dept_avg, 2) AS pct_diff
FROM (
    SELECT *,
        AVG(salary) OVER (PARTITION BY department) AS dept_avg
    FROM employees
) sub
WHERE salary BETWEEN dept_avg * 0.9 AND dept_avg * 1.1
ORDER BY department;
```

</details>

---

### Problem 31: Assign quartile ranking to employees by salary

```
DIFFICULTY: ★★★☆☆
TOPIC: NTILE, Percentile
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    emp_name,
    salary,
    NTILE(4) OVER (ORDER BY salary) AS quartile,
    CASE NTILE(4) OVER (ORDER BY salary)
        WHEN 1 THEN 'Bottom 25%'
        WHEN 2 THEN '25-50%'
        WHEN 3 THEN '50-75%'
        WHEN 4 THEN 'Top 25%'
    END AS salary_band,
    PERCENT_RANK() OVER (ORDER BY salary) AS percentile
FROM employees
ORDER BY salary;
```

</details>

---

### Problem 32: Find the median salary per department

```
DIFFICULTY: ★★★★☆
TOPIC: PERCENTILE_CONT or ROW_NUMBER
```

<details>
<summary>💡 Solution</summary>

```sql
-- Method 1: PostgreSQL PERCENTILE_CONT
SELECT 
    department,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees
GROUP BY department;

-- Method 2: ROW_NUMBER (works on all databases)
WITH ranked AS (
    SELECT department, salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary) AS rn,
        COUNT(*) OVER (PARTITION BY department) AS cnt
    FROM employees
)
SELECT department, AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (FLOOR((cnt + 1) / 2.0), CEIL((cnt + 1) / 2.0))
GROUP BY department;
```

</details>

---

### Problem 33: Generate a date series and find days with no orders

```
DIFFICULTY: ★★★★☆
TOPIC: GENERATE_SERIES, LEFT JOIN
```

<details>
<summary>💡 Solution</summary>

```sql
-- PostgreSQL: Generate all dates, find gaps
WITH date_range AS (
    SELECT generate_series(
        '2024-01-01'::date, 
        '2024-07-31'::date, 
        '1 day'::interval
    )::date AS dt
)
SELECT dr.dt AS missing_date
FROM date_range dr
LEFT JOIN orders o ON dr.dt = o.order_date
WHERE o.order_id IS NULL
    AND EXTRACT(DOW FROM dr.dt) NOT IN (0, 6)  -- Exclude weekends
ORDER BY dr.dt;
```

</details>

---

### Problem 34: Find employees who have orders in every month of Q1 2024

```
DIFFICULTY: ★★★★☆
TOPIC: GROUP BY, HAVING COUNT(DISTINCT)
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT e.emp_name
FROM employees e
JOIN orders o ON e.emp_id = o.emp_id
WHERE o.order_date >= '2024-01-01' AND o.order_date < '2024-04-01'
GROUP BY e.emp_name
HAVING COUNT(DISTINCT EXTRACT(MONTH FROM o.order_date)) = 3;
```

</details>

---

### Problem 35: Create a salary distribution histogram

```
DIFFICULTY: ★★★★☆
TOPIC: CASE, WIDTH_BUCKET, Histogram
```

<details>
<summary>💡 Solution</summary>

```sql
-- Method 1: Manual buckets with CASE
SELECT 
    CASE 
        WHEN salary < 80000 THEN '< 80K'
        WHEN salary < 90000 THEN '80K-90K'
        WHEN salary < 100000 THEN '90K-100K'
        WHEN salary < 110000 THEN '100K-110K'
        WHEN salary < 120000 THEN '110K-120K'
        ELSE '120K+'
    END AS salary_range,
    COUNT(*) AS emp_count,
    REPEAT('█', COUNT(*)::int) AS bar_chart
FROM employees
GROUP BY 1
ORDER BY MIN(salary);

-- Method 2: PostgreSQL WIDTH_BUCKET
SELECT 
    WIDTH_BUCKET(salary, 70000, 140000, 7) AS bucket,
    MIN(salary) AS range_start,
    MAX(salary) AS range_end,
    COUNT(*) AS emp_count
FROM employees
GROUP BY 1
ORDER BY 1;
```

</details>

---

### Problem 36: Find employees with the same salary (pairs)

```
DIFFICULTY: ★★★☆☆
TOPIC: Self-JOIN
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    a.emp_name AS employee_1,
    b.emp_name AS employee_2,
    a.salary
FROM employees a
JOIN employees b ON a.salary = b.salary AND a.emp_id < b.emp_id
ORDER BY a.salary DESC;
```

</details>

---

### Problem 37: Calculate the percentage change in revenue from first month

```
DIFFICULTY: ★★★★☆
TOPIC: FIRST_VALUE, Window Functions
```

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    month_date,
    revenue,
    FIRST_VALUE(revenue) OVER (ORDER BY month_date) AS first_month_rev,
    ROUND(
        (revenue - FIRST_VALUE(revenue) OVER (ORDER BY month_date)) * 100.0
        / FIRST_VALUE(revenue) OVER (ORDER BY month_date), 2
    ) AS pct_change_from_start
FROM monthly_revenue;
```

</details>

---

### Problem 38: Find employees whose salary rank changed over departments

```
DIFFICULTY: ★★★★☆
TOPIC: Multiple Window Functions
```

<details>
<summary>💡 Solution</summary>

```sql
-- Show each employee's rank within dept vs overall
SELECT 
    emp_name,
    department,
    salary,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS overall_rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) 
    - DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_shift
FROM employees
ORDER BY overall_rank;
```

</details>

---

### Problem 39: Unpivot — Convert columns to rows

```
DIFFICULTY: ★★★★☆
TOPIC: UNPIVOT, UNION ALL, LATERAL
```

<details>
<summary>💡 Solution</summary>

```sql
-- Given a table: monthly_scores(student, jan, feb, mar)
-- Convert to rows: (student, month, score)

-- Method 1: UNION ALL (universal)
SELECT emp_name, 'salary' AS metric, salary AS value FROM employees
UNION ALL
SELECT emp_name, 'emp_id', emp_id FROM employees
ORDER BY emp_name, metric;

-- Method 2: PostgreSQL LATERAL / VALUES
SELECT e.emp_name, x.metric, x.value
FROM employees e
CROSS JOIN LATERAL (
    VALUES ('salary', e.salary), ('emp_id', e.emp_id::decimal)
) AS x(metric, value);

-- Method 3: SQL Server UNPIVOT
-- SELECT emp_name, metric, value
-- FROM employees
-- UNPIVOT (value FOR metric IN (salary, emp_id)) AS unpvt;
```

</details>

---

### Problem 40: Find the most common pair of products ordered together

```
DIFFICULTY: ★★★★★
TOPIC: Self-JOIN, Combination Logic
```

<details>
<summary>💡 Solution</summary>

```sql
-- Find products ordered by the same employee
WITH emp_products AS (
    SELECT DISTINCT emp_id, product FROM orders
)
SELECT 
    a.product AS product_1,
    b.product AS product_2,
    COUNT(DISTINCT a.emp_id) AS co_buyers
FROM emp_products a
JOIN emp_products b 
    ON a.emp_id = b.emp_id 
    AND a.product < b.product  -- Avoid duplicate pairs
GROUP BY a.product, b.product
ORDER BY co_buyers DESC;
```

</details>

---

# 🔥 EXPERT: Problems 41–50 (Brain Melters)

> _These are the questions that make senior engineers sweat. If you can solve these, you're ready for FAANG. Take 10-20 minutes each._

---

### Problem 41: ⭐ Running total with reset — cumulative sum that resets on condition

```
DIFFICULTY: ★★★★★
TOPIC: Window Functions, Conditional Reset

Calculate running total of order amounts per employee,
but RESET the running total when status = 'cancelled'.
```

<details>
<summary>💡 Solution</summary>

```sql
-- Group by segments separated by 'cancelled' status
WITH segments AS (
    SELECT *,
        SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) 
            OVER (PARTITION BY emp_id ORDER BY order_date) AS segment_id
    FROM orders
)
SELECT 
    emp_id,
    order_date,
    product,
    amount,
    status,
    SUM(CASE WHEN status != 'cancelled' THEN amount ELSE 0 END) 
        OVER (PARTITION BY emp_id, segment_id ORDER BY order_date) AS running_total
FROM segments
ORDER BY emp_id, order_date;
```

</details>

---

### Problem 42: Dense Ranking with ties — find Nth highest salary per department

```
DIFFICULTY: ★★★★★
TOPIC: Parameterized Nth value, Window Functions
```

<details>
<summary>💡 Solution</summary>

```sql
-- Find 2nd highest salary per department (handling ties)
WITH dept_ranked AS (
    SELECT 
        emp_name, department, salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS drnk
    FROM employees
)
SELECT department, emp_name, salary
FROM dept_ranked
WHERE drnk = 2;

-- Generalized: Find Nth highest
-- Change WHERE drnk = N for any N
-- If department has fewer than N distinct salaries → no result for that dept
```

</details>

---

### Problem 43: Session analysis — group events into sessions with 30-minute timeout

```
DIFFICULTY: ★★★★★
TOPIC: Gaps, LAG, Conditional grouping
```

<details>
<summary>💡 Solution</summary>

```sql
-- Concept: If gap between events > 30 min → new session
WITH events AS (
    SELECT *,
        LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_event,
        CASE 
            WHEN event_time - LAG(event_time) OVER (
                PARTITION BY user_id ORDER BY event_time
            ) > INTERVAL '30 minutes' 
            THEN 1 
            ELSE 0 
        END AS new_session
    FROM user_events
),
sessions AS (
    SELECT *,
        SUM(new_session) OVER (
            PARTITION BY user_id ORDER BY event_time
        ) AS session_id
    FROM events
)
SELECT 
    user_id,
    session_id,
    MIN(event_time) AS session_start,
    MAX(event_time) AS session_end,
    COUNT(*) AS events_in_session,
    MAX(event_time) - MIN(event_time) AS session_duration
FROM sessions
GROUP BY user_id, session_id
ORDER BY user_id, session_start;
```

</details>

---

### Problem 44: Funnel analysis — conversion rates through steps

```
DIFFICULTY: ★★★★★
TOPIC: Conditional COUNT, Funnel Logic
```

<details>
<summary>💡 Solution</summary>

```sql
-- Given: User events (page_view → add_to_cart → checkout → purchase)
-- Calculate conversion rate at each step

WITH funnel AS (
    SELECT 
        COUNT(DISTINCT CASE WHEN event = 'page_view' THEN user_id END) AS step1_views,
        COUNT(DISTINCT CASE WHEN event = 'add_to_cart' THEN user_id END) AS step2_cart,
        COUNT(DISTINCT CASE WHEN event = 'checkout' THEN user_id END) AS step3_checkout,
        COUNT(DISTINCT CASE WHEN event = 'purchase' THEN user_id END) AS step4_purchase
    FROM user_events
    WHERE event_date BETWEEN '2024-01-01' AND '2024-03-31'
)
SELECT 
    step1_views,
    step2_cart,
    ROUND(step2_cart * 100.0 / NULLIF(step1_views, 0), 2) AS view_to_cart_pct,
    step3_checkout,
    ROUND(step3_checkout * 100.0 / NULLIF(step2_cart, 0), 2) AS cart_to_checkout_pct,
    step4_purchase,
    ROUND(step4_purchase * 100.0 / NULLIF(step3_checkout, 0), 2) AS checkout_to_purchase_pct,
    ROUND(step4_purchase * 100.0 / NULLIF(step1_views, 0), 2) AS overall_conversion_pct
FROM funnel;
```

</details>

---

### Problem 45: Retained users — find users active in both current and previous month

```
DIFFICULTY: ★★★★☆
TOPIC: Self-JOIN, Date Logic, Retention
```

<details>
<summary>💡 Solution</summary>

```sql
WITH monthly_active AS (
    SELECT DISTINCT 
        user_id,
        DATE_TRUNC('month', login_date) AS active_month
    FROM daily_logins
)
SELECT 
    curr.active_month,
    COUNT(DISTINCT curr.user_id) AS active_users,
    COUNT(DISTINCT prev.user_id) AS retained_users,
    ROUND(
        COUNT(DISTINCT prev.user_id) * 100.0 
        / NULLIF(COUNT(DISTINCT curr.user_id), 0), 2
    ) AS retention_rate
FROM monthly_active curr
LEFT JOIN monthly_active prev
    ON curr.user_id = prev.user_id
    AND curr.active_month = prev.active_month + INTERVAL '1 month'
GROUP BY curr.active_month
ORDER BY curr.active_month;
```

</details>

---

### Problem 46: Identify the best performing month for each department

```
DIFFICULTY: ★★★★☆
TOPIC: Window Function, Multiple GROUP BY
```

<details>
<summary>💡 Solution</summary>

```sql
WITH dept_monthly AS (
    SELECT 
        e.department,
        DATE_TRUNC('month', o.order_date) AS month,
        SUM(o.amount) AS monthly_revenue,
        COUNT(*) AS order_count
    FROM orders o
    JOIN employees e ON o.emp_id = e.emp_id
    WHERE o.status = 'completed'
    GROUP BY e.department, DATE_TRUNC('month', o.order_date)
),
ranked AS (
    SELECT *,
        RANK() OVER (PARTITION BY department ORDER BY monthly_revenue DESC) AS rnk
    FROM dept_monthly
)
SELECT department, month AS best_month, monthly_revenue, order_count
FROM ranked
WHERE rnk = 1;
```

</details>

---

### Problem 47: Calculate a weighted moving average

```
DIFFICULTY: ★★★★★
TOPIC: Window Functions, Custom Weights
```

<details>
<summary>💡 Solution</summary>

```sql
-- 3-month weighted moving average: current (3x), prev (2x), 2 months ago (1x)
WITH weighted AS (
    SELECT 
        month_date,
        revenue,
        revenue * 3 AS w_current,
        LAG(revenue, 1) OVER (ORDER BY month_date) * 2 AS w_prev1,
        LAG(revenue, 2) OVER (ORDER BY month_date) * 1 AS w_prev2
    FROM monthly_revenue
)
SELECT 
    month_date,
    revenue,
    ROUND(
        (COALESCE(w_current, 0) + COALESCE(w_prev1, 0) + COALESCE(w_prev2, 0)) 
        / (3 + CASE WHEN w_prev1 IS NOT NULL THEN 2 ELSE 0 END 
             + CASE WHEN w_prev2 IS NOT NULL THEN 1 ELSE 0 END)
    , 2) AS weighted_avg
FROM weighted;
```

</details>

---

### Problem 48: Find employees with consistently increasing salary history

```
DIFFICULTY: ★★★★★
TOPIC: LAG, ALL comparison
```

<details>
<summary>💡 Solution</summary>

```sql
-- Using a salary history table: (emp_id, effective_date, salary)
-- Find employees whose salary ALWAYS increased (never decreased)

WITH salary_changes AS (
    SELECT 
        emp_id,
        salary,
        LAG(salary) OVER (PARTITION BY emp_id ORDER BY effective_date) AS prev_salary
    FROM salary_history
)
SELECT emp_id
FROM salary_changes
WHERE prev_salary IS NOT NULL
GROUP BY emp_id
HAVING MIN(CASE WHEN salary > prev_salary THEN 1 ELSE 0 END) = 1;
-- MIN = 1 means ALL changes were increases (no decrease ever had 0)
```

</details>

---

### Problem 49: Create a calendar report showing daily revenue with weekday names

```
DIFFICULTY: ★★★★☆
TOPIC: GENERATE_SERIES, Date Functions, LEFT JOIN
```

<details>
<summary>💡 Solution</summary>

```sql
WITH calendar AS (
    SELECT generate_series('2024-01-01'::date, '2024-01-31'::date, '1 day')::date AS dt
),
daily_rev AS (
    SELECT order_date, SUM(amount) AS revenue, COUNT(*) AS orders
    FROM orders
    WHERE status = 'completed'
    GROUP BY order_date
)
SELECT 
    c.dt AS date,
    TO_CHAR(c.dt, 'Day') AS day_name,
    CASE EXTRACT(DOW FROM c.dt) 
        WHEN 0 THEN '🟥' WHEN 6 THEN '🟥' ELSE '🟢' 
    END AS type,
    COALESCE(d.revenue, 0) AS revenue,
    COALESCE(d.orders, 0) AS orders,
    SUM(COALESCE(d.revenue, 0)) OVER (ORDER BY c.dt) AS mtd_revenue
FROM calendar c
LEFT JOIN daily_rev d ON c.dt = d.order_date
ORDER BY c.dt;
```

</details>

---

### Problem 50: ⭐ The Grand Finale — Build a complete dashboard query

```
DIFFICULTY: ★★★★★
TOPIC: CTEs, Window Functions, Aggregation, Conditional Logic

Create a single query that produces a department-level dashboard:
- Department name
- Total employees
- Total salary spend
- Average salary
- Highest earner name
- Count of orders by employees
- Total order revenue
- Avg order value
- Revenue per employee
- Department rank by revenue
```

<details>
<summary>💡 Solution</summary>

```sql
WITH dept_stats AS (
    SELECT 
        e.department,
        COUNT(DISTINCT e.emp_id) AS total_employees,
        SUM(e.salary) AS total_salary,
        ROUND(AVG(e.salary), 2) AS avg_salary,
        MAX(e.salary) AS max_salary
    FROM employees e
    GROUP BY e.department
),
top_earners AS (
    SELECT DISTINCT ON (department)
        department, emp_name AS top_earner
    FROM employees
    ORDER BY department, salary DESC
),
order_stats AS (
    SELECT 
        e.department,
        COUNT(o.order_id) AS total_orders,
        COALESCE(SUM(o.amount), 0) AS total_revenue,
        ROUND(COALESCE(AVG(o.amount), 0), 2) AS avg_order_value
    FROM employees e
    LEFT JOIN orders o ON e.emp_id = o.emp_id AND o.status = 'completed'
    GROUP BY e.department
)
SELECT 
    ds.department,
    ds.total_employees,
    ds.total_salary,
    ds.avg_salary,
    te.top_earner,
    os.total_orders,
    os.total_revenue,
    os.avg_order_value,
    ROUND(os.total_revenue / NULLIF(ds.total_employees, 0), 2) AS rev_per_employee,
    RANK() OVER (ORDER BY os.total_revenue DESC) AS revenue_rank
FROM dept_stats ds
JOIN top_earners te ON ds.department = te.department
JOIN order_stats os ON ds.department = os.department
ORDER BY revenue_rank;
```

</details>

---

## 📊 Progress Tracker

```
┌─────────────────────────────────────────────────────────────┐
│                YOUR SQL MASTERY LEVEL                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Problems 1-10 solved:  Basic SQL ✅                       │
│  Problems 11-25 solved: Intermediate SQL ✅                │
│  Problems 26-40 solved: Advanced SQL 🔥                    │
│  Problems 41-50 solved: Expert SQL 🏆                      │
│                                                             │
│  Can solve 40+:  You're interview-ready for MOST companies │
│  Can solve 50:   You're ready for FAANG/Big Tech           │
│  Can solve all from memory: You're a SQL GOD 👑            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔑 Key Patterns to Remember

```
┌──────────────────────────────────────────────────────────────┐
│                SQL PATTERN CHEAT SHEET                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Nth Highest       → DENSE_RANK() + WHERE rnk = N          │
│  Top N Per Group   → ROW_NUMBER() + PARTITION BY + WHERE rn≤N│
│  Running Total     → SUM() OVER (ORDER BY ...)              │
│  Moving Average    → AVG() OVER (ROWS BETWEEN N PREC...)   │
│  Gaps and Islands  → date - ROW_NUMBER() = group identifier │
│  Consecutive Streak→ Same as Gaps & Islands                 │
│  Year-over-Year    → LAG() for previous period              │
│  Cumulative %      → SUM() OVER () for total               │
│  Pivot             → CASE WHEN inside aggregate             │
│  Find Duplicates   → GROUP BY + HAVING COUNT(*) > 1        │
│  Delete Duplicates → ROW_NUMBER() + DELETE WHERE rn > 1    │
│  Hierarchy         → Recursive CTE                          │
│  Missing Records   → generate_series + LEFT JOIN            │
│  Self-Comparison   → Self-JOIN with a.id < b.id            │
│  Retention         → Self-JOIN on user_id + month offset    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

| Next Chapter | Link |
|-------------|------|
| Quick Cheat Sheets | [Chapter 8.5 — Database Cheat Sheets](./05-Cheat-Sheets.md) |
| Go back to SQL Interview Questions | [Chapter 8.1 — SQL Interview Questions](./01-SQL-Interview-Questions.md) |

---

> **"The SQL problems you can't solve today will be easy next week — if you practice every day."** 🏋️
