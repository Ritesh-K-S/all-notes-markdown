# 2.7 — Window Functions — SQL Superpower 🟡⭐🔥

> **"If JOINs are the heart of SQL, Window Functions are its brain."**
> Once you master these, you'll solve in 1 line what used to take 20 lines of self-joins and subqueries.

---

## 🎯 What You'll Master

```
✅ What window functions are (and why they're different from GROUP BY)
✅ The OVER() clause — the key to everything
✅ PARTITION BY vs GROUP BY — the critical difference
✅ Ranking: ROW_NUMBER, RANK, DENSE_RANK, NTILE
✅ Navigation: LAG, LEAD, FIRST_VALUE, LAST_VALUE, NTH_VALUE
✅ Aggregation as windows: SUM, AVG, COUNT over windows
✅ Frame specification: ROWS BETWEEN, RANGE BETWEEN
✅ Running totals, moving averages, cumulative distributions
✅ Real-world patterns used in FAANG-level SQL interviews
```

---

## 🧠 The "Aha!" Moment — What is a Window Function?

### The Problem with GROUP BY

Imagine you have a `sales` table:

```sql
SELECT department, SUM(amount) 
FROM sales 
GROUP BY department;
```

| department | SUM(amount) |
|------------|-------------|
| Engineering | 500000 |
| Sales | 750000 |
| Marketing | 300000 |

**You got the totals... but you LOST every individual row.** ❌

What if you want to see **each employee's sale ALONGSIDE the department total**?

### Enter Window Functions 🪟

```sql
SELECT 
    employee_name,
    department,
    amount,
    SUM(amount) OVER (PARTITION BY department) AS dept_total
FROM sales;
```

| employee_name | department | amount | dept_total |
|---------------|------------|--------|------------|
| Alice | Engineering | 200000 | 500000 |
| Bob | Engineering | 300000 | 500000 |
| Carol | Sales | 400000 | 750000 |
| Dave | Sales | 350000 | 750000 |

**Every row is preserved. The aggregate rides alongside.** ✅

> 💡 **Think of it like this:**
> - `GROUP BY` = "Crush rows into groups, show one result per group"
> - `Window Function` = "Look through a *window* at related rows, compute something, but keep every row intact"

---

## 🏗️ Anatomy of a Window Function

```
┌─────────────────────────────────────────────────────────────────┐
│  function_name(expression)                                      │
│                                                                 │
│  OVER (                                                         │
│      [PARTITION BY column1, column2, ...]    ← Define groups    │
│      [ORDER BY column3 [ASC|DESC], ...]      ← Define order    │
│      [frame_clause]                          ← Define range     │
│  )                                                              │
└─────────────────────────────────────────────────────────────────┘
```

### The Three Components:

| Component | What it does | Required? |
|-----------|-------------|-----------|
| `PARTITION BY` | Divides rows into groups (like GROUP BY, but keeps rows) | Optional — omit = entire table is one window |
| `ORDER BY` | Orders rows within each partition | Required for ranking/navigation functions |
| `Frame Clause` | Defines which rows in the partition to include | Optional — has smart defaults |

---

## 🏆 Category 1: Ranking Functions

### The Sample Data We'll Use Throughout

```sql
CREATE TABLE employees (
    emp_id     INT PRIMARY KEY,
    name       VARCHAR(50),
    department VARCHAR(50),
    salary     DECIMAL(10,2),
    hire_date  DATE
);

INSERT INTO employees VALUES
(1, 'Alice',   'Engineering', 95000,  '2020-01-15'),
(2, 'Bob',     'Engineering', 95000,  '2019-06-01'),
(3, 'Carol',   'Engineering', 110000, '2018-03-20'),
(4, 'Dave',    'Sales',       80000,  '2021-02-10'),
(5, 'Eve',     'Sales',       85000,  '2020-07-15'),
(6, 'Frank',   'Sales',       85000,  '2019-11-01'),
(7, 'Grace',   'Marketing',   72000,  '2022-01-05'),
(8, 'Heidi',   'Marketing',   78000,  '2020-09-12');
```

---

### 🔢 ROW_NUMBER() — Unique Sequential Number

Assigns a unique integer to each row. **No ties. Ever.**

```sql
SELECT 
    name, department, salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num
FROM employees;
```

| name | department | salary | row_num |
|------|-----------|--------|---------|
| Carol | Engineering | 110000 | 1 |
| Alice | Engineering | 95000 | **2** |
| Bob | Engineering | 95000 | **3** |
| Eve | Sales | 85000 | 4 |
| Frank | Sales | 85000 | 5 |
| Dave | Sales | 80000 | 6 |
| Heidi | Marketing | 78000 | 7 |
| Grace | Marketing | 72000 | 8 |

> ⚠️ Alice and Bob have the same salary, but ROW_NUMBER gives different numbers. The tiebreaker is **non-deterministic** (random) unless you add more ORDER BY columns!

**Fix the non-determinism:**
```sql
ROW_NUMBER() OVER (ORDER BY salary DESC, emp_id ASC) AS row_num
-- Now ties are broken by emp_id — deterministic!
```

---

### 🏅 RANK() — Olympic-Style Ranking (Gaps After Ties)

```sql
SELECT 
    name, department, salary,
    RANK() OVER (ORDER BY salary DESC) AS rnk
FROM employees;
```

| name | department | salary | rnk |
|------|-----------|--------|-----|
| Carol | Engineering | 110000 | 1 |
| Alice | Engineering | 95000 | **2** |
| Bob | Engineering | 95000 | **2** |
| Eve | Sales | 85000 | **4** ← Skipped 3! |
| Frank | Sales | 85000 | **4** |
| Dave | Sales | 80000 | **6** ← Skipped 5! |

> 💡 **Think Olympics:** Two people tie for Gold → they both get Gold (rank 1). The next person gets Bronze (rank 3). There is no Silver.

---

### 🎯 DENSE_RANK() — Compact Ranking (No Gaps)

```sql
SELECT 
    name, department, salary,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rnk
FROM employees;
```

| name | department | salary | dense_rnk |
|------|-----------|--------|-----------|
| Carol | Engineering | 110000 | 1 |
| Alice | Engineering | 95000 | **2** |
| Bob | Engineering | 95000 | **2** |
| Eve | Sales | 85000 | **3** ← No gap! |
| Frank | Sales | 85000 | **3** |
| Dave | Sales | 80000 | **4** ← No gap! |

---

### 🔥 The Three Rankings — Side by Side

```sql
SELECT 
    name, salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    RANK()       OVER (ORDER BY salary DESC) AS rnk,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rnk
FROM employees;
```

| name | salary | row_num | rnk | dense_rnk |
|------|--------|---------|-----|-----------|
| Carol | 110000 | 1 | 1 | 1 |
| Alice | 95000 | 2 | 2 | 2 |
| Bob | 95000 | 3 | 2 | 2 |
| Eve | 85000 | 4 | 4 | 3 |
| Frank | 85000 | 5 | 4 | 3 |
| Dave | 80000 | 6 | 6 | 4 |

```
📊 VISUAL CHEAT SHEET:

ROW_NUMBER:  1  2  3  4  5  6     ← Always unique, no gaps
RANK:        1  2  2  4  4  6     ← Ties get same rank, gaps after
DENSE_RANK:  1  2  2  3  3  4     ← Ties get same rank, NO gaps
```

---

### 🧩 NTILE(n) — Divide Into N Equal Buckets

Splits rows into `n` approximately equal groups.

```sql
SELECT 
    name, salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS quartile
FROM employees;
```

| name | salary | quartile |
|------|--------|----------|
| Carol | 110000 | 1 |
| Alice | 95000 | 1 |
| Bob | 95000 | 2 |
| Eve | 85000 | 2 |
| Frank | 85000 | 3 |
| Dave | 80000 | 3 |
| Heidi | 78000 | 4 |
| Grace | 72000 | 4 |

> 💡 **Real-world use:** "Show me the top 25% of earners" → `WHERE quartile = 1`

---

### 🎯 PARTITION BY + Ranking = Per-Group Rankings

**"Rank employees within each department":**

```sql
SELECT 
    name, department, salary,
    ROW_NUMBER() OVER (
        PARTITION BY department 
        ORDER BY salary DESC
    ) AS dept_rank
FROM employees;
```

| name | department | salary | dept_rank |
|------|-----------|--------|-----------|
| Carol | Engineering | 110000 | 1 |
| Alice | Engineering | 95000 | 2 |
| Bob | Engineering | 95000 | 3 |
| Heidi | Marketing | 78000 | 1 |
| Grace | Marketing | 72000 | 2 |
| Eve | Sales | 85000 | 1 |
| Frank | Sales | 85000 | 2 |
| Dave | Sales | 80000 | 3 |

> 🔥 **Classic Interview Question:** "Find the top-N per group"

```sql
-- Top 2 earners per department
WITH ranked AS (
    SELECT *, 
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT name, department, salary
FROM ranked
WHERE rn <= 2;
```

---

## 🧭 Category 2: Navigation (Offset) Functions

### ⬅️ LAG() — Look at the Previous Row

```sql
SELECT 
    name, department, salary,
    LAG(salary) OVER (ORDER BY salary) AS prev_salary,
    salary - LAG(salary) OVER (ORDER BY salary) AS diff_from_prev
FROM employees;
```

| name | salary | prev_salary | diff_from_prev |
|------|--------|-------------|----------------|
| Grace | 72000 | NULL | NULL |
| Heidi | 78000 | 72000 | 6000 |
| Dave | 80000 | 78000 | 2000 |
| Eve | 85000 | 80000 | 5000 |
| Frank | 85000 | 85000 | 0 |
| Alice | 95000 | 85000 | 10000 |

**LAG Syntax:**
```sql
LAG(column, offset, default_value) OVER (...)
-- offset: how many rows back (default = 1)
-- default_value: what to return if no previous row (default = NULL)
```

```sql
-- Look 2 rows back, use 0 as default
LAG(salary, 2, 0) OVER (ORDER BY salary)
```

---

### ➡️ LEAD() — Look at the Next Row

```sql
SELECT 
    name, salary,
    LEAD(salary) OVER (ORDER BY salary) AS next_salary,
    LEAD(salary) OVER (ORDER BY salary) - salary AS gap_to_next
FROM employees;
```

| name | salary | next_salary | gap_to_next |
|------|--------|-------------|-------------|
| Grace | 72000 | 78000 | 6000 |
| Heidi | 78000 | 80000 | 2000 |
| Dave | 80000 | 85000 | 5000 |
| ... | ... | ... | ... |
| Carol | 110000 | NULL | NULL |

> 💡 **Real-world:** "Compare each month's revenue with the previous month" → Use LAG  
> "Show what next month's target should be" → Use LEAD

---

### 🏁 FIRST_VALUE() / LAST_VALUE() / NTH_VALUE()

```sql
SELECT 
    name, department, salary,
    FIRST_VALUE(name) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS top_earner,
    LAST_VALUE(name) OVER (
        PARTITION BY department ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_earner
FROM employees;
```

> ⚠️ **Gotcha Alert!** `LAST_VALUE` without a proper frame clause gives unexpected results!  
> The default frame is `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which means LAST_VALUE just returns the current row.  
> **Always specify `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`** for LAST_VALUE.

```sql
-- Get the 2nd highest salary in each department
NTH_VALUE(salary, 2) OVER (
    PARTITION BY department ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
) AS second_highest
```

---

## 📊 Category 3: Aggregate Window Functions

Any aggregate function can become a window function by adding `OVER()`.

### Running Total (Cumulative Sum)

```sql
SELECT 
    order_date, 
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;
```

| order_date | amount | running_total |
|------------|--------|---------------|
| 2024-01-01 | 100 | 100 |
| 2024-01-02 | 250 | 350 |
| 2024-01-03 | 175 | 525 |
| 2024-01-04 | 300 | 825 |
| 2024-01-05 | 200 | 1025 |

```
📊 VISUAL — How Running Total Accumulates:

Day 1:  ████ 100                        → 100
Day 2:  ████ 100 + █████████ 250        → 350
Day 3:  ████ 100 + █████████ 250 + ██████ 175  → 525
Day 4:  (all above) + ████████████ 300  → 825
Day 5:  (all above) + ████████ 200      → 1025
```

### Running Average

```sql
SELECT 
    order_date, amount,
    AVG(amount) OVER (ORDER BY order_date) AS running_avg,
    COUNT(*) OVER (ORDER BY order_date) AS running_count,
    MIN(amount) OVER (ORDER BY order_date) AS running_min,
    MAX(amount) OVER (ORDER BY order_date) AS running_max
FROM orders;
```

### Percentage of Total

```sql
SELECT 
    department, 
    salary,
    SUM(salary) OVER () AS total_salary,
    ROUND(salary * 100.0 / SUM(salary) OVER (), 2) AS pct_of_total
FROM employees;
```

### Percentage Within Group

```sql
SELECT 
    name, department, salary,
    SUM(salary) OVER (PARTITION BY department) AS dept_total,
    ROUND(salary * 100.0 / SUM(salary) OVER (PARTITION BY department), 2) AS pct_in_dept
FROM employees;
```

---

## 🖼️ Category 4: Frame Clause — The Fine Control

The frame clause determines **exactly which rows** within the partition contribute to the calculation.

### Frame Syntax

```sql
{ ROWS | RANGE | GROUPS } BETWEEN
    { UNBOUNDED PRECEDING | N PRECEDING | CURRENT ROW }
AND
    { CURRENT ROW | N FOLLOWING | UNBOUNDED FOLLOWING }
```

### Visual Guide to Frames

```
Given 7 rows ordered, and the current row is row 4:

Row 1  ──┐
Row 2    │── UNBOUNDED PRECEDING
Row 3    │── 2 PRECEDING
Row 4  ──┤── CURRENT ROW
Row 5    │── 1 FOLLOWING
Row 6    │── 3 FOLLOWING
Row 7  ──┘── UNBOUNDED FOLLOWING
```

### Common Frame Patterns

```sql
-- 1. DEFAULT (when ORDER BY is present)
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
-- Includes all rows from start to current → Running totals

-- 2. Entire partition
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
-- All rows in the partition → Same as no ORDER BY

-- 3. Moving average (last 3 rows)
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW

-- 4. Centered moving average (±1 row)
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING

-- 5. Everything after current row
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
```

### Moving Average Example (3-Day Moving Average)

```sql
SELECT 
    order_date,
    amount,
    AVG(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3day
FROM orders;
```

| order_date | amount | moving_avg_3day |
|------------|--------|-----------------|
| 2024-01-01 | 100 | 100.00 |
| 2024-01-02 | 250 | 175.00 |
| 2024-01-03 | 175 | **175.00** |
| 2024-01-04 | 300 | **241.67** |
| 2024-01-05 | 200 | **225.00** |

```
Day 3 avg: (100 + 250 + 175) / 3 = 175.00  ← looks at 2 preceding + current
Day 4 avg: (250 + 175 + 300) / 3 = 241.67  ← window slides forward
Day 5 avg: (175 + 300 + 200) / 3 = 225.00  ← window slides forward
```

### ROWS vs RANGE vs GROUPS

| Frame Type | How it works | Use when |
|-----------|-------------|----------|
| `ROWS` | Physical row count | Most common — exact N rows |
| `RANGE` | Logical value range | When ties should be grouped together |
| `GROUPS` | Groups of tied values | SQL:2011 — peer-group based (PostgreSQL 11+) |

```sql
-- ROWS: exactly 2 rows before current
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW

-- RANGE: all rows whose value is within 2 units of current
RANGE BETWEEN 2 PRECEDING AND CURRENT ROW
-- If current salary is 85000, includes all rows with salary 83000-85000

-- GROUPS: 2 peer-groups before current
GROUPS BETWEEN 2 PRECEDING AND CURRENT ROW
```

---

## 🔥 Real-World Patterns (Interview Favorites)

### Pattern 1: Year-over-Year Growth

```sql
SELECT 
    year,
    revenue,
    LAG(revenue) OVER (ORDER BY year) AS prev_year_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY year)) * 100.0 
        / LAG(revenue) OVER (ORDER BY year), 2
    ) AS yoy_growth_pct
FROM annual_revenue;
```

| year | revenue | prev_year_revenue | yoy_growth_pct |
|------|---------|-------------------|----------------|
| 2021 | 1000000 | NULL | NULL |
| 2022 | 1200000 | 1000000 | 20.00 |
| 2023 | 1500000 | 1200000 | 25.00 |
| 2024 | 1350000 | 1500000 | -10.00 |

---

### Pattern 2: Find Consecutive Sequences (Gaps & Islands)

```sql
-- Find consecutive login days for each user
WITH numbered AS (
    SELECT 
        user_id, login_date,
        login_date - ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY login_date
        ) * INTERVAL '1 day' AS grp
    FROM logins
)
SELECT 
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*) AS streak_length
FROM numbered
GROUP BY user_id, grp
ORDER BY streak_length DESC;
```

---

### Pattern 3: Running Total with Reset

```sql
-- Running total that resets each month
SELECT 
    order_date, amount,
    SUM(amount) OVER (
        PARTITION BY DATE_TRUNC('month', order_date)
        ORDER BY order_date
    ) AS monthly_running_total
FROM orders;
```

---

### Pattern 4: Median Using PERCENTILE_CONT

```sql
-- Median salary per department
SELECT DISTINCT
    department,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) 
        OVER (PARTITION BY department) AS median_salary
FROM employees;
```

---

### Pattern 5: Cumulative Distribution

```sql
SELECT 
    name, salary,
    CUME_DIST() OVER (ORDER BY salary) AS cumulative_dist,
    PERCENT_RANK() OVER (ORDER BY salary) AS percent_rank
FROM employees;
```

| name | salary | cumulative_dist | percent_rank |
|------|--------|----------------|--------------|
| Grace | 72000 | 0.125 | 0.000 |
| Heidi | 78000 | 0.250 | 0.143 |
| Dave | 80000 | 0.375 | 0.286 |
| Eve | 85000 | 0.625 | 0.429 |
| Frank | 85000 | 0.625 | 0.429 |
| Alice | 95000 | 0.875 | 0.714 |
| Bob | 95000 | 0.875 | 0.714 |
| Carol | 110000 | 1.000 | 1.000 |

> - `CUME_DIST()` = What % of rows have a value ≤ this row's value  
> - `PERCENT_RANK()` = Relative position: (rank - 1) / (total_rows - 1)

---

### Pattern 6: Deduplicate Rows (Keep Latest)

```sql
-- Remove duplicate customer records, keeping the most recent
WITH ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY email 
            ORDER BY updated_at DESC
        ) AS rn
    FROM customers
)
DELETE FROM customers 
WHERE id IN (SELECT id FROM ranked WHERE rn > 1);
```

---

## 📋 Named Windows (DRY — Don't Repeat Yourself)

When you use the same window definition multiple times, name it:

```sql
SELECT 
    name, department, salary,
    ROW_NUMBER() OVER w AS row_num,
    RANK()       OVER w AS rnk,
    DENSE_RANK() OVER w AS dense_rnk,
    SUM(salary)  OVER w AS running_salary
FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);
```

> 💡 Supported in **PostgreSQL, MySQL 8+, SQLite 3.28+**. Not in SQL Server or Oracle (use inline OVER instead).

---

## ⚡ Performance Tips

| Tip | Why |
|-----|-----|
| **Index the ORDER BY columns** | Window functions sort data — indexes prevent expensive sorts |
| **Minimize distinct OVER clauses** | Each unique OVER() may require a separate sort pass |
| **Use named windows** | Helps optimizer recognize shared sort requirements |
| **ROWS is faster than RANGE** | ROWS is a simple offset; RANGE must handle ties |
| **Filter before windowing** | Use WHERE to reduce rows BEFORE the window function runs |
| **Can't use in WHERE** | Window functions execute AFTER WHERE → use a CTE/subquery to filter |

```sql
-- ❌ WRONG: Can't filter by window function in WHERE
SELECT name, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
FROM employees
WHERE rn <= 5;  -- ERROR!

-- ✅ CORRECT: Wrap in CTE or subquery
WITH ranked AS (
    SELECT name, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn <= 5;
```

---

## 🗺️ SQL Execution Order (Where Window Functions Fit)

```
1. FROM / JOIN          ← Get all the tables
2. WHERE                ← Filter rows
3. GROUP BY             ← Aggregate groups
4. HAVING               ← Filter groups
5. SELECT               ← Choose columns
   ├── Window Functions ← ⭐ COMPUTED HERE (after SELECT expressions)
6. DISTINCT             ← Remove duplicates
7. ORDER BY             ← Sort final result
8. LIMIT / OFFSET       ← Paginate
```

> This is why you can't use window functions in WHERE or HAVING — they haven't been computed yet!

---

## 🗄️ Database Support Matrix

| Function | PostgreSQL | MySQL 8+ | SQL Server | Oracle | SQLite |
|----------|-----------|----------|------------|--------|--------|
| ROW_NUMBER | ✅ | ✅ | ✅ | ✅ | ✅ |
| RANK | ✅ | ✅ | ✅ | ✅ | ✅ |
| DENSE_RANK | ✅ | ✅ | ✅ | ✅ | ✅ |
| NTILE | ✅ | ✅ | ✅ | ✅ | ✅ |
| LAG / LEAD | ✅ | ✅ | ✅ | ✅ | ✅ |
| FIRST_VALUE | ✅ | ✅ | ✅ | ✅ | ✅ |
| LAST_VALUE | ✅ | ✅ | ✅ | ✅ | ✅ |
| NTH_VALUE | ✅ | ✅ | ❌ | ✅ | ✅ |
| CUME_DIST | ✅ | ✅ | ✅ | ✅ | ✅ |
| PERCENT_RANK | ✅ | ✅ | ✅ | ✅ | ✅ |
| PERCENTILE_CONT | ✅ | ❌ | ✅ | ✅ | ❌ |
| Named WINDOW | ✅ | ✅ | ❌ | ❌ | ✅ |
| GROUPS frame | ✅ (11+) | ❌ | ❌ | ❌ | ✅ |

---

## 🧪 Practice Challenges

### Challenge 1: Department Salary Ranking
> Write a query that ranks employees by salary within each department. Show employees where their salary is in the top 3 for their department.

<details>
<summary>💡 Solution</summary>

```sql
WITH ranked AS (
    SELECT 
        name, department, salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT * FROM ranked WHERE rnk <= 3;
```
</details>

### Challenge 2: Month-over-Month Revenue Change
> For each month, show the revenue, previous month's revenue, and the percentage change.

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month)) * 100.0 
        / NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 2
    ) AS pct_change
FROM monthly_revenue;
```
</details>

### Challenge 3: 7-Day Moving Average
> Calculate a 7-day moving average of daily sales.

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    sale_date, daily_amount,
    ROUND(AVG(daily_amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_7day
FROM daily_sales;
```
</details>

### Challenge 4: Find the Salary Gap
> For each employee, show the difference between their salary and the next higher salary in the company.

<details>
<summary>💡 Solution</summary>

```sql
SELECT 
    name, salary,
    LEAD(salary) OVER (ORDER BY salary) AS next_salary,
    LEAD(salary) OVER (ORDER BY salary) - salary AS gap
FROM employees;
```
</details>

---

## 🎯 Key Takeaways

```
┌────────────────────────────────────────────────────────────────┐
│  WINDOW FUNCTIONS CHEAT SHEET                                  │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  RANKING:     ROW_NUMBER  RANK  DENSE_RANK  NTILE             │
│  NAVIGATION:  LAG  LEAD  FIRST_VALUE  LAST_VALUE  NTH_VALUE   │
│  AGGREGATE:   SUM  AVG  COUNT  MIN  MAX  (with OVER)          │
│  STATISTICAL: CUME_DIST  PERCENT_RANK  PERCENTILE_CONT        │
│                                                                │
│  PARTITION BY  = group rows (keeps all rows)                   │
│  ORDER BY     = sequence within partition                      │
│  Frame        = which rows participate in calculation          │
│                                                                │
│  ⚠️ Can't use in WHERE → wrap in CTE / subquery               │
│  ⚠️ LAST_VALUE needs explicit frame specification              │
│  ⚠️ ROW_NUMBER ties are non-deterministic without tiebreaker  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

> 🚀 **Next Up:** [2.8 — CTEs & Recursive Queries](./08-CTEs-and-Recursive.md) — Build readable, reusable, and recursive SQL with the WITH clause.

---

*Window functions are the dividing line between "I know SQL" and "I'm dangerous with SQL." Welcome to the danger zone.* 🔥
