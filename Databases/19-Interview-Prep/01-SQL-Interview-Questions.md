# 🎯 Chapter 8.1 — Top 100 SQL Interview Questions (With Solutions)

> **"I've seen candidates with 5 years of experience fail on a simple GROUP BY question. It's not about experience — it's about clarity of concepts."** — Senior Database Architect at Google

---

## 📌 Metadata

| Field | Value |
|-------|-------|
| **Level** | 🟡 Intermediate → 🔴 Advanced |
| **Time to Master** | ~6-8 hours (spread across days) |
| **Prerequisites** | SQL Mastery (Part 2A), Indexing (1.7), Transactions (1.8) |
| **Interview Coverage** | FAANG, Banks, Product Companies, Startups |

---

## 🎯 What You'll Master

- ✅ 100 real SQL interview questions — categorized from Basic → Expert
- ✅ Battle-tested answers with **explanations**, not just code
- ✅ Common **traps** interviewers set and how to avoid them
- ✅ Questions across **Oracle, SQL Server, MySQL, PostgreSQL**
- ✅ The difference between a "correct" answer and an **impressive** answer
- ✅ Real-world scenarios that top companies actually ask

---

## 🧠 How Interviewers Think

Before we dive into questions, understand **what interviewers are really evaluating**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INTERVIEW EVALUATION MATRIX                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Level 1: SYNTAX (Can you write SQL?)                              │
│  ├── SELECT, WHERE, JOIN, GROUP BY                                 │
│  └── 30% of candidates fail here                                   │
│                                                                     │
│  Level 2: LOGIC (Can you solve problems?)                          │
│  ├── Subqueries, Window Functions, CTEs                            │
│  └── 50% of candidates fail here                                   │
│                                                                     │
│  Level 3: OPTIMIZATION (Can you write EFFICIENT SQL?)              │
│  ├── Index awareness, Execution plans, SARGability                 │
│  └── 85% of candidates fail here                                   │
│                                                                     │
│  Level 4: ARCHITECTURE (Can you DESIGN with SQL?)                  │
│  ├── Schema design, Trade-offs, Scaling decisions                  │
│  └── 95% of candidates fail here — THIS is where you stand out    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

> 💡 **Pro Tip**: Don't just memorize queries. Understand the **WHY** behind each answer. Interviewers follow up with "Why not use X instead?" — and that's where most people fail.

---

## 📋 Sample Data — Used Throughout This Chapter

We'll reference these tables in most questions:

```sql
-- EMPLOYEES TABLE
CREATE TABLE employees (
    emp_id      INT PRIMARY KEY,
    emp_name    VARCHAR(100),
    department  VARCHAR(50),
    salary      DECIMAL(10,2),
    manager_id  INT,
    hire_date   DATE,
    city        VARCHAR(50)
);

-- Sample Data
INSERT INTO employees VALUES
(1, 'Alice',   'Engineering', 120000, NULL,  '2019-01-15', 'New York'),
(2, 'Bob',     'Engineering', 110000, 1,     '2019-03-20', 'San Francisco'),
(3, 'Charlie', 'Marketing',   90000, 1,     '2020-06-10', 'New York'),
(4, 'Diana',   'Engineering', 130000, 1,     '2018-11-01', 'Seattle'),
(5, 'Eve',     'Marketing',   85000, 3,     '2021-02-14', 'New York'),
(6, 'Frank',   'Sales',       95000, NULL,  '2020-08-22', 'Chicago'),
(7, 'Grace',   'Sales',       88000, 6,     '2021-05-30', 'Chicago'),
(8, 'Hank',    'Engineering', 105000, 2,    '2022-01-10', 'San Francisco'),
(9, 'Ivy',     'Marketing',   92000, 3,     '2020-09-15', 'Seattle'),
(10,'Jack',    'Sales',       78000, 6,     '2023-03-01', 'New York');

-- ORDERS TABLE
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    emp_id      INT REFERENCES employees(emp_id),
    product     VARCHAR(100),
    amount      DECIMAL(10,2),
    order_date  DATE,
    status      VARCHAR(20)
);

-- DEPARTMENTS TABLE
CREATE TABLE departments (
    dept_name   VARCHAR(50) PRIMARY KEY,
    budget      DECIMAL(12,2),
    location    VARCHAR(50)
);
```

---

# 🟢 SECTION 1: BASIC SQL (Questions 1–25)

> _If you can't ace these with your eyes closed, stop and revisit Part 2A._

---

### Q1: What is the difference between `WHERE` and `HAVING`? ⭐

**The Trap**: Many say "HAVING is used with GROUP BY." That's incomplete.

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  WHERE  → Filters ROWS before grouping                      │
│  HAVING → Filters GROUPS after aggregation                  │
│                                                              │
│  Processing Order:                                          │
│  FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```sql
-- WHERE: Filter rows BEFORE grouping
SELECT department, AVG(salary) AS avg_salary
FROM employees
WHERE city = 'New York'       -- ← Filters individual rows
GROUP BY department;

-- HAVING: Filter groups AFTER aggregation
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 100000;  -- ← Filters aggregated results
```

**🎯 Impressive Answer**: "WHERE filters rows before aggregation — it can't reference aggregate functions. HAVING filters after GROUP BY — it CAN reference aggregates. Performance-wise, always prefer WHERE when possible because it reduces the dataset BEFORE the expensive grouping operation."

---

### Q2: What is the difference between `DELETE`, `TRUNCATE`, and `DROP`? ⭐

```
┌────────────────┬──────────────────┬──────────────────┬────────────────┐
│ Feature        │ DELETE           │ TRUNCATE         │ DROP           │
├────────────────┼──────────────────┼──────────────────┼────────────────┤
│ Type           │ DML              │ DDL              │ DDL            │
│ WHERE clause   │ ✅ Yes           │ ❌ No            │ ❌ No          │
│ Rollback       │ ✅ Yes           │ ⚠️ DB-dependent  │ ⚠️ DB-dependent│
│ Triggers       │ ✅ Fires         │ ❌ No            │ ❌ No          │
│ Speed          │ 🐌 Slow (row by)│ ⚡ Fast (pages)  │ ⚡ Instant     │
│ Resets Identity│ ❌ No            │ ✅ Yes           │ N/A            │
│ Table exists   │ ✅ Yes           │ ✅ Yes           │ ❌ Gone        │
│ Logged         │ Row-by-row       │ Minimal/Bulk     │ Minimal        │
│ Space freed    │ ❌ Not immediately│ ✅ Immediately   │ ✅ Completely  │
└────────────────┴──────────────────┴──────────────────┴────────────────┘
```

> ⚠️ **Gotcha**: In PostgreSQL, TRUNCATE IS transactional (can be rolled back). In Oracle and SQL Server, it's a DDL — can't be rolled back after commit. MySQL depends on the storage engine.

---

### Q3: What are the different types of JOINs? Explain with examples. ⭐

```
     INNER JOIN              LEFT JOIN               RIGHT JOIN
    ┌────┬────┐            ┌────┬────┐            ┌────┬────┐
    │    │████│            │████│████│            │    │████│
    │  A │████│ B          │████│████│ B          │ A  │████│ B
    │    │████│            │████│████│            │    │████│
    └────┴────┘            └────┴────┘            └────┴────┘
     Only matching          All from A             All from B
                           + matching B           + matching A

     FULL OUTER JOIN        CROSS JOIN             SELF JOIN
    ┌────┬────┐            ┌──────────┐            ┌────────┐
    │████│████│            │ A × B    │            │ A ⟲ A  │
    │████│████│            │ Every    │            │ Table   │
    │████│████│            │ combo    │            │ joins   │
    └────┴────┘            └──────────┘            │ itself  │
     Everything             Cartesian              └────────┘
     from both              Product
```

```sql
-- INNER JOIN: Only matching rows
SELECT e.emp_name, d.budget
FROM employees e
INNER JOIN departments d ON e.department = d.dept_name;

-- LEFT JOIN: All employees, even without department match
SELECT e.emp_name, d.budget
FROM employees e
LEFT JOIN departments d ON e.department = d.dept_name;

-- SELF JOIN: Find each employee's manager
SELECT 
    e.emp_name AS employee,
    m.emp_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;
```

---

### Q4: What is the difference between `UNION` and `UNION ALL`? ⭐

```sql
-- UNION: Removes duplicates (slower — needs sorting/hashing)
SELECT city FROM employees
UNION
SELECT location FROM departments;

-- UNION ALL: Keeps duplicates (faster — no dedup overhead)
SELECT city FROM employees
UNION ALL
SELECT location FROM departments;
```

**🎯 Impressive Answer**: "Use UNION ALL unless you specifically need deduplication. UNION internally performs a DISTINCT operation which requires sorting or hashing — on large datasets, this can be the difference between milliseconds and minutes."

---

### Q5: What is a NULL? How does it behave in comparisons? ⭐

```
┌────────────────────────────────────────────────────────────┐
│  NULL is NOT a value. It means "UNKNOWN" or "MISSING."    │
│                                                            │
│  NULL = NULL    →  NULL  (not TRUE!)                      │
│  NULL != NULL   →  NULL  (not TRUE!)                      │
│  NULL > 5       →  NULL                                    │
│  NULL + 100     →  NULL                                    │
│  NULL AND TRUE  →  NULL                                    │
│  NULL OR TRUE   →  TRUE                                    │
│  NULL AND FALSE →  FALSE                                   │
│                                                            │
│  To check for NULL:  IS NULL / IS NOT NULL                │
│  NEVER use:  = NULL  or  != NULL                          │
└────────────────────────────────────────────────────────────┘
```

```sql
-- ❌ WRONG — This will NEVER return rows where manager_id is NULL
SELECT * FROM employees WHERE manager_id = NULL;

-- ✅ CORRECT
SELECT * FROM employees WHERE manager_id IS NULL;

-- Handle NULLs in results
SELECT emp_name, COALESCE(manager_id, 0) AS manager_id FROM employees;
-- Oracle: NVL(manager_id, 0)
-- SQL Server: ISNULL(manager_id, 0)
```

---

### Q6: Write a query to find the 2nd highest salary.

**This is THE most asked SQL interview question in history.** Here are 5 ways to solve it:

```sql
-- Method 1: Subquery (Works everywhere)
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Method 2: DENSE_RANK (Best — handles ties correctly)
SELECT salary AS second_highest
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk = 2;

-- Method 3: LIMIT/OFFSET (MySQL, PostgreSQL)
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;

-- Method 4: TOP (SQL Server)
SELECT TOP 1 salary
FROM (SELECT DISTINCT TOP 2 salary FROM employees ORDER BY salary DESC) sub
ORDER BY salary ASC;

-- Method 5: ROWNUM (Oracle)
SELECT salary FROM (
    SELECT salary, ROWNUM AS rn
    FROM (SELECT DISTINCT salary FROM employees ORDER BY salary DESC)
) WHERE rn = 2;
```

> 💡 **Pro Tip**: Always use DENSE_RANK in interviews. It handles ties (multiple people with same salary) and generalizes to "Nth highest" easily.

---

### Q7: What is the difference between `RANK()`, `DENSE_RANK()`, and `ROW_NUMBER()`? ⭐

```
Data: 100, 100, 90, 80

┌────────┬──────────────┬──────────────┬──────────────┐
│ Salary │ ROW_NUMBER() │ RANK()       │ DENSE_RANK() │
├────────┼──────────────┼──────────────┼──────────────┤
│ 100    │ 1            │ 1            │ 1            │
│ 100    │ 2            │ 1            │ 1            │
│ 90     │ 3            │ 3 ← skips 2 │ 2 ← no skip │
│ 80     │ 4            │ 4            │ 3            │
└────────┴──────────────┴──────────────┴──────────────┘
```

```
ROW_NUMBER() → Always unique (1,2,3,4) — Arbitrary tiebreak
RANK()       → Ties get same rank, then SKIPS (1,1,3,4)
DENSE_RANK() → Ties get same rank, NO skip (1,1,2,3)
```

**When to use which:**
- **ROW_NUMBER()** → Pagination, deduplication (pick 1 per group)
- **RANK()** → Sports leaderboards (skip positions after tie)
- **DENSE_RANK()** → Finding Nth highest (no gaps)

---

### Q8: What is the difference between `IN` and `EXISTS`? ⭐

```sql
-- IN: Best when subquery returns small result set
SELECT emp_name FROM employees
WHERE department IN (SELECT dept_name FROM departments WHERE budget > 500000);

-- EXISTS: Best when outer table is small, subquery table is large
SELECT emp_name FROM employees e
WHERE EXISTS (SELECT 1 FROM departments d WHERE d.dept_name = e.department AND d.budget > 500000);
```

```
┌──────────────────────────────────────────────────────────────┐
│                     IN vs EXISTS                             │
├──────────────┬───────────────────┬───────────────────────────┤
│ Feature      │ IN                │ EXISTS                    │
├──────────────┼───────────────────┼───────────────────────────┤
│ NULL handling│ ⚠️ Fails with NULL│ ✅ Handles NULLs          │
│ Optimization │ Hashes subquery   │ Short-circuits on first   │
│ Best when    │ Small subquery    │ Large subquery, indexed   │
│ Readability  │ ✅ Simpler        │ 🟡 Slightly complex       │
│ Correlated   │ ❌ No             │ ✅ Yes                    │
└──────────────┴───────────────────┴───────────────────────────┘
```

> ⚠️ **Critical Trap**: `NOT IN` with NULLs returns ZERO rows! If the subquery returns even ONE NULL, the entire NOT IN evaluates to UNKNOWN. **Always use NOT EXISTS instead of NOT IN.**

---

### Q9: What is a Correlated Subquery?

```sql
-- Regular Subquery: Runs ONCE
SELECT * FROM employees 
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Correlated Subquery: Runs ONCE PER ROW of outer query
SELECT e.emp_name, e.salary, e.department
FROM employees e
WHERE e.salary > (
    SELECT AVG(salary) 
    FROM employees 
    WHERE department = e.department  -- ← References outer query!
);
```

```
Regular Subquery:                Correlated Subquery:
┌──────────────────┐            ┌────────────────────────────┐
│ Inner query runs │            │ For EACH row in outer:     │
│ ONCE             │            │   Run inner query with     │
│ ↓                │            │   that row's values        │
│ Result cached    │            │   ↓                        │
│ ↓                │            │   Compare                  │
│ Outer uses cache │            │   ↓                        │
└──────────────────┘            │   Next row → run again     │
                                └────────────────────────────┘
```

> 💡 **Pro Tip**: Correlated subqueries can be performance killers. On a table with 1M rows, the inner query runs 1M times. Consider rewriting as a JOIN or window function.

---

### Q10: Write a query to find employees who earn more than their department average.

```sql
-- Method 1: Correlated Subquery
SELECT emp_name, salary, department
FROM employees e
WHERE salary > (
    SELECT AVG(salary) FROM employees WHERE department = e.department
);

-- Method 2: Window Function (Better Performance) ⚡
SELECT emp_name, salary, department, dept_avg
FROM (
    SELECT *, AVG(salary) OVER (PARTITION BY department) AS dept_avg
    FROM employees
) sub
WHERE salary > dept_avg;

-- Method 3: CTE + JOIN
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department
)
SELECT e.emp_name, e.salary, e.department, d.avg_sal
FROM employees e
JOIN dept_avg d ON e.department = d.department
WHERE e.salary > d.avg_sal;
```

---

### Q11: What is a PRIMARY KEY vs UNIQUE KEY?

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Feature          │ PRIMARY KEY          │ UNIQUE KEY           │
├──────────────────┼──────────────────────┼──────────────────────┤
│ NULLs allowed    │ ❌ Never             │ ✅ One NULL (varies) │
│ Per table        │ Only ONE             │ Multiple allowed     │
│ Creates index    │ Clustered (usually)  │ Non-clustered        │
│ Purpose          │ Row identifier       │ Business constraint  │
│ FK reference     │ ✅ Yes               │ ✅ Yes               │
└──────────────────┴──────────────────────┴──────────────────────┘
```

> ⚠️ **Gotcha**: PostgreSQL allows multiple NULLs in UNIQUE columns. SQL Server allows only ONE NULL. Oracle treats NULLs as distinct in UNIQUE but ignores all-NULL composite keys. Know your database!

---

### Q12: What is Normalization? Explain all Normal Forms.

```
┌──────────────────────────────────────────────────────────────────┐
│  1NF: No repeating groups, atomic values                        │
│  ↓                                                              │
│  2NF: 1NF + No partial dependencies (on part of composite key) │
│  ↓                                                              │
│  3NF: 2NF + No transitive dependencies (non-key → non-key)     │
│  ↓                                                              │
│  BCNF: Every determinant is a candidate key                     │
│  ↓                                                              │
│  4NF: No multi-valued dependencies                              │
│  ↓                                                              │
│  5NF: No join dependencies                                      │
└──────────────────────────────────────────────────────────────────┘
```

**🎯 Interview Shortcut**: In practice, most databases are normalized to 3NF or BCNF. If asked about 4NF/5NF, say: "These address multi-valued and join dependencies respectively, but are rarely needed in practice. BCNF handles 99% of real-world scenarios."

---

### Q13: What is Denormalization and when would you use it?

**Denormalization** = Intentionally adding redundancy to improve **read performance**.

```
USE WHEN:                              AVOID WHEN:
✅ Read-heavy workloads                ❌ Write-heavy workloads
✅ Reporting / analytics queries       ❌ Data integrity is critical
✅ Reducing expensive JOINs            ❌ Storage is a concern
✅ Data warehouse / OLAP               ❌ OLTP with many updates
```

```sql
-- Normalized: 3 JOINs to show order with customer and product
SELECT o.order_id, c.name, p.product_name, o.quantity
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id;

-- Denormalized: Single table scan — fast reads
SELECT order_id, customer_name, product_name, quantity
FROM order_details_denormalized;
```

---

### Q14: What is the difference between `CHAR` and `VARCHAR`?

```
┌────────────┬────────────────────────┬─────────────────────────────┐
│ Feature    │ CHAR(10)               │ VARCHAR(10)                 │
├────────────┼────────────────────────┼─────────────────────────────┤
│ Storage    │ Fixed: Always 10 bytes │ Variable: actual + 1-2 bytes│
│ Padding    │ Right-padded with ' '  │ No padding                  │
│ Speed      │ ⚡ Slightly faster      │ 🐌 Slightly slower          │
│ Use when   │ Fixed-length: codes,   │ Variable-length: names,     │
│            │ states, zip codes      │ emails, addresses           │
│ Comparison │ 'AB' = 'AB        '   │ 'AB' = 'AB' (no padding)   │
└────────────┴────────────────────────┴─────────────────────────────┘
```

---

### Q15: Write a query to find duplicate records in a table. ⭐

```sql
-- Method 1: GROUP BY + HAVING (Find duplicate values)
SELECT emp_name, COUNT(*) AS cnt
FROM employees
GROUP BY emp_name
HAVING COUNT(*) > 1;

-- Method 2: Find ALL duplicate rows with their IDs
SELECT *
FROM employees
WHERE emp_name IN (
    SELECT emp_name FROM employees GROUP BY emp_name HAVING COUNT(*) > 1
);

-- Method 3: ROW_NUMBER() — Most powerful (keep one, delete rest)
WITH duplicates AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY emp_name, department ORDER BY emp_id) AS rn
    FROM employees
)
SELECT * FROM duplicates WHERE rn > 1;  -- These are the duplicates
-- DELETE FROM duplicates WHERE rn > 1;  -- To remove them
```

---

### Q16: What is a View? When would you use it?

```sql
-- A view is a stored query — a "virtual table"
CREATE VIEW high_earners AS
SELECT emp_name, department, salary
FROM employees
WHERE salary > 100000;

-- Use it like a table
SELECT * FROM high_earners;
```

**Use Cases**: Security (hide columns), simplification (complex joins), abstraction (rename columns for an API), reusable logic.

> 💡 **Pro Tip**: A view doesn't store data — it runs the underlying query each time. For performance, use **Materialized Views** (PostgreSQL, Oracle) which cache results.

---

### Q17: What is an Index and why does it matter? ⭐

```
WITHOUT INDEX (Full Table Scan):       WITH INDEX (Index Seek):
┌─────────────────────┐               ┌──────────┐
│ Scan ALL 10M rows   │               │ B-Tree   │
│ to find "Alice"     │               │ lookup   │
│ Time: 45 seconds    │               │ → 3 hops │
│ 💀                   │               │ Time: 2ms│
└─────────────────────┘               │ ⚡        │
                                      └──────────┘
```

```sql
-- Create an index
CREATE INDEX idx_emp_name ON employees(emp_name);

-- Composite index (order matters!)
CREATE INDEX idx_dept_salary ON employees(department, salary);
```

**When NOT to index:**
- ❌ Small tables (< 1000 rows)
- ❌ Columns with low cardinality (gender, status)
- ❌ Heavily updated columns (index maintenance overhead)
- ❌ Tables with more writes than reads

---

### Q18: What are the different types of indexes?

```
┌──────────────────┬────────────────────────────────────────────┐
│ Index Type       │ Use Case                                   │
├──────────────────┼────────────────────────────────────────────┤
│ B-Tree           │ Default. Range queries, equality, sorting  │
│ Hash             │ Exact match only (=), no range             │
│ Bitmap           │ Low cardinality (gender, status) — Oracle  │
│ GIN              │ Full-text, arrays, JSONB — PostgreSQL      │
│ GiST             │ Geospatial, range types — PostgreSQL       │
│ BRIN             │ Large sequential data (timestamps)         │
│ Covering         │ Includes all columns needed by query       │
│ Partial/Filtered │ Index only subset of rows (WHERE clause)   │
│ Clustered        │ Reorders physical data — one per table     │
│ Non-Clustered    │ Pointer to data — multiple per table       │
└──────────────────┴────────────────────────────────────────────┘
```

---

### Q19: What is the difference between Clustered and Non-Clustered Index? ⭐

```
CLUSTERED INDEX:                      NON-CLUSTERED INDEX:
┌──────────────┐                     ┌──────────────┐
│ The table IS │                     │ Separate     │
│ the index.   │                     │ structure    │
│ Data sorted  │                     │ with pointers│
│ physically.  │                     │ to actual    │
│              │                     │ data rows.   │
│ Only ONE per │                     │ Many per     │
│ table.       │                     │ table.       │
│              │                     │              │
│ Like a       │                     │ Like a book's│
│ phone book   │                     │ back index   │
│ (sorted by   │                     │ (page refs)  │
│  last name)  │                     │              │
└──────────────┘                     └──────────────┘
```

---

### Q20: What is a Foreign Key? Can it be NULL?

```sql
-- Foreign Key: Enforces referential integrity
ALTER TABLE employees
ADD CONSTRAINT fk_manager
FOREIGN KEY (manager_id) REFERENCES employees(emp_id);
```

**Yes, a Foreign Key CAN be NULL.** A NULL FK means "this relationship doesn't exist yet" — like an employee without a manager (CEO). But a FK can NEVER point to a non-existent row.

---

### Q21: What are constraints in SQL? List all types.

```
┌──────────────────┬────────────────────────────────────────────┐
│ Constraint       │ Purpose                                    │
├──────────────────┼────────────────────────────────────────────┤
│ PRIMARY KEY      │ Uniquely identifies each row               │
│ FOREIGN KEY      │ Links to another table's PK/UNIQUE         │
│ UNIQUE           │ No duplicate values in column              │
│ NOT NULL         │ Column cannot be empty                     │
│ CHECK            │ Custom validation (salary > 0)             │
│ DEFAULT          │ Auto-fill if no value provided             │
│ EXCLUSION (PG)   │ No overlapping ranges (scheduling)         │
└──────────────────┴────────────────────────────────────────────┘
```

---

### Q22: What is the difference between `GROUP BY` and `PARTITION BY`?

```sql
-- GROUP BY: Collapses rows → one row per group
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;
-- Result: 3 rows (Engineering, Marketing, Sales)

-- PARTITION BY: Keeps all rows, adds calculated column
SELECT emp_name, department, salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
-- Result: 10 rows (all employees + their department average)
```

```
GROUP BY = "Give me a summary"     →  Fewer rows
PARTITION BY = "Add context"       →  Same number of rows
```

---

### Q23: What is a Self Join? Give a real example.

```sql
-- Find each employee and their manager's name
SELECT 
    e.emp_name AS employee,
    e.salary AS emp_salary,
    m.emp_name AS manager,
    m.salary AS mgr_salary
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;
```

**Other Self Join use cases:**
- Find employees in the same city
- Compare rows within the same table (year-over-year)
- Hierarchical data traversal

---

### Q24: What is the order of SQL clause execution? ⭐

```
Written Order:                  Execution Order:
SELECT   ──── 5th ────→        1. FROM / JOIN
FROM     ──── 1st ────→        2. WHERE
WHERE    ──── 2nd ────→        3. GROUP BY
GROUP BY ──── 3rd ────→        4. HAVING
HAVING   ──── 4th ────→        5. SELECT
ORDER BY ──── 6th ────→        6. ORDER BY
LIMIT    ──── 7th ────→        7. LIMIT / OFFSET
```

> 💡 **This is why you can't use column aliases in WHERE** — SELECT hasn't run yet! But you CAN use them in ORDER BY (it runs after SELECT).

---

### Q25: What is a Stored Procedure vs a Function?

```
┌────────────────┬─────────────────────┬─────────────────────────┐
│ Feature        │ Stored Procedure    │ Function                │
├────────────────┼─────────────────────┼─────────────────────────┤
│ Return value   │ Optional (OUT param)│ Must return a value     │
│ Use in SELECT  │ ❌ No               │ ✅ Yes                  │
│ DML allowed    │ ✅ INSERT/UPDATE/DEL│ ⚠️ Read-only (usually)  │
│ Transactions   │ ✅ Can manage       │ ❌ Cannot (usually)     │
│ Call syntax    │ EXEC / CALL         │ In expressions          │
│ Side effects   │ ✅ Yes              │ ❌ Should be pure       │
└────────────────┴─────────────────────┴─────────────────────────┘
```

---

# 🟡 SECTION 2: INTERMEDIATE SQL (Questions 26–55)

> _This is where interviews get real. Window functions, CTEs, and complex JOINs separate the juniors from the seniors._

---

### Q26: Write a query to find the top 3 earners in EACH department. ⭐🔥

```sql
-- DENSE_RANK: Top 3 distinct salaries per department
WITH ranked AS (
    SELECT *,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT emp_name, department, salary
FROM ranked
WHERE rnk <= 3;
```

**Why DENSE_RANK and not ROW_NUMBER?**
- If two people tie at #2, ROW_NUMBER arbitrarily picks one → unfair
- DENSE_RANK gives both rank 2, then rank 3 → correct semantics

---

### Q27: What is a CTE? How is it different from a subquery?

```sql
-- CTE (Common Table Expression) — Named temporary result
WITH expensive_depts AS (
    SELECT department, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department
    HAVING AVG(salary) > 100000
)
SELECT e.emp_name, e.salary, ed.avg_sal
FROM employees e
JOIN expensive_depts ed ON e.department = ed.department;
```

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Feature          │ CTE                  │ Subquery             │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Readability      │ ✅ Named & clear     │ 🟡 Can be nested     │
│ Reusability      │ ✅ Reference multiple │ ❌ Must repeat        │
│                  │    times in query     │                      │
│ Recursion        │ ✅ Supports recursive │ ❌ No                │
│ Performance      │ Same as subquery*    │ Same as CTE*         │
│ Scope            │ Single statement     │ Single statement     │
└──────────────────┴──────────────────────┴──────────────────────┘
* In most databases, CTEs and subqueries produce identical execution plans
```

---

### Q28: Write a Recursive CTE to display an org hierarchy.

```sql
WITH RECURSIVE org_tree AS (
    -- Anchor: Top-level employees (no manager)
    SELECT emp_id, emp_name, manager_id, 1 AS level,
           CAST(emp_name AS VARCHAR(500)) AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: Find direct reports
    SELECT e.emp_id, e.emp_name, e.manager_id, ot.level + 1,
           CAST(ot.path || ' → ' || e.emp_name AS VARCHAR(500))
    FROM employees e
    INNER JOIN org_tree ot ON e.manager_id = ot.emp_id
)
SELECT level, path, emp_name, emp_id
FROM org_tree
ORDER BY path;
```

```
Output:
Level  Path                           Employee
1      Alice                          Alice
2      Alice → Bob                    Bob
3      Alice → Bob → Hank             Hank
2      Alice → Charlie                Charlie
3      Alice → Charlie → Eve          Eve
3      Alice → Charlie → Ivy          Ivy
2      Alice → Diana                  Diana
1      Frank                          Frank
2      Frank → Grace                  Grace
2      Frank → Jack                   Jack
```

> ⚠️ **SQL Server**: Use `WITH org_tree AS` (no RECURSIVE keyword). Oracle: Use `CONNECT BY PRIOR` (or CTE with 12c+).

---

### Q29: Explain Window Functions with LAG and LEAD. ⭐

```sql
-- LAG: Previous row's value | LEAD: Next row's value
SELECT 
    emp_name,
    salary,
    LAG(salary) OVER (ORDER BY salary) AS prev_salary,
    LEAD(salary) OVER (ORDER BY salary) AS next_salary,
    salary - LAG(salary) OVER (ORDER BY salary) AS diff_from_prev
FROM employees;
```

**Real-World Use**: Month-over-month revenue comparison, stock price changes, detecting gaps in sequential data.

---

### Q30: What is COALESCE and how is it different from ISNULL/NVL?

```sql
-- COALESCE: Returns first non-NULL value (ANSI standard, all DBs)
SELECT COALESCE(phone, mobile, email, 'No Contact') AS contact FROM users;

-- ISNULL: SQL Server only, takes exactly 2 args
SELECT ISNULL(phone, 'N/A') FROM users;

-- NVL: Oracle only, takes exactly 2 args
SELECT NVL(phone, 'N/A') FROM users;

-- IFNULL: MySQL only, takes exactly 2 args
SELECT IFNULL(phone, 'N/A') FROM users;
```

> 💡 **Pro Tip**: Always use COALESCE in interviews — it's ANSI standard, portable, and takes multiple arguments.

---

### Q31: Explain the CASE expression with a real scenario.

```sql
-- Categorize employees by salary band
SELECT emp_name, salary,
    CASE
        WHEN salary >= 120000 THEN 'Senior Band'
        WHEN salary >= 100000 THEN 'Mid Band'
        WHEN salary >= 85000  THEN 'Junior Band'
        ELSE 'Entry Level'
    END AS salary_band,
    CASE department
        WHEN 'Engineering' THEN '💻'
        WHEN 'Marketing'   THEN '📢'
        WHEN 'Sales'       THEN '💰'
    END AS dept_icon
FROM employees;
```

**Two forms**: Searched CASE (with conditions) and Simple CASE (with values).

---

### Q32: Write a query to find gaps in sequential IDs.

```sql
-- Find missing IDs in a sequence (1, 2, 4, 5, 8 → gaps: 3, 6, 7)
SELECT 
    emp_id + 1 AS gap_start,
    next_id - 1 AS gap_end
FROM (
    SELECT emp_id, 
           LEAD(emp_id) OVER (ORDER BY emp_id) AS next_id
    FROM employees
) sub
WHERE next_id - emp_id > 1;
```

---

### Q33: What is a CROSS JOIN and when would you actually use it?

```sql
-- Generate all possible combinations
SELECT e.emp_name, d.dept_name
FROM employees e
CROSS JOIN departments d;  -- 10 employees × 3 departments = 30 rows
```

**Real use cases:**
- Generate a calendar (dates × time slots)
- Create a product matrix (sizes × colors)
- Pivot operations
- Generate test data

---

### Q34: Explain PIVOT and UNPIVOT with examples.

```sql
-- SQL Server PIVOT: Rows → Columns
SELECT *
FROM (
    SELECT department, city, salary FROM employees
) src
PIVOT (
    AVG(salary) FOR department IN ([Engineering], [Marketing], [Sales])
) pvt;

-- PostgreSQL equivalent (using CASE/FILTER)
SELECT 
    city,
    AVG(salary) FILTER (WHERE department = 'Engineering') AS engineering,
    AVG(salary) FILTER (WHERE department = 'Marketing') AS marketing,
    AVG(salary) FILTER (WHERE department = 'Sales') AS sales
FROM employees
GROUP BY city;

-- MySQL equivalent (using CASE)
SELECT 
    city,
    AVG(CASE WHEN department = 'Engineering' THEN salary END) AS engineering,
    AVG(CASE WHEN department = 'Marketing' THEN salary END) AS marketing,
    AVG(CASE WHEN department = 'Sales' THEN salary END) AS sales
FROM employees
GROUP BY city;
```

---

### Q35: Write a query for Running Total (Cumulative Sum). ⭐

```sql
SELECT 
    emp_name,
    salary,
    SUM(salary) OVER (ORDER BY hire_date) AS running_total,
    SUM(salary) OVER (PARTITION BY department ORDER BY hire_date) AS dept_running_total
FROM employees
ORDER BY hire_date;
```

```
Window Frame (what SUM operates on):
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  ← default for ORDER BY
    │                                         │
    └── From first row ──── to current row ───┘
```

---

### Q36: What is a Materialized View? How is it different from a regular view?

```
┌────────────────┬──────────────────────┬──────────────────────────┐
│ Feature        │ Regular View         │ Materialized View        │
├────────────────┼──────────────────────┼──────────────────────────┤
│ Data storage   │ ❌ No (virtual)      │ ✅ Yes (physical copy)   │
│ Freshness      │ ✅ Always current    │ ⚠️ Stale until refreshed│
│ Read speed     │ 🐌 Same as query     │ ⚡ Pre-computed          │
│ Write overhead │ None                 │ Refresh cost             │
│ Index support  │ ⚠️ Limited           │ ✅ Can index             │
│ Use case       │ Simplification       │ Heavy reports, caching   │
│ Supported      │ All databases        │ PG, Oracle, SQL Server*  │
└────────────────┴──────────────────────┴──────────────────────────┘
* SQL Server calls them "Indexed Views"
```

---

### Q37: How would you delete duplicate rows, keeping only one?

```sql
-- Method 1: CTE + ROW_NUMBER (Works on all modern DBs)
WITH duplicates AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY emp_name, department, salary 
            ORDER BY emp_id
        ) AS rn
    FROM employees
)
DELETE FROM duplicates WHERE rn > 1;

-- Method 2: Self-join DELETE (MySQL-friendly)
DELETE e1 FROM employees e1
INNER JOIN employees e2 
ON e1.emp_name = e2.emp_name 
   AND e1.department = e2.department
   AND e1.emp_id > e2.emp_id;

-- Method 3: NOT IN (Simple but slower)
DELETE FROM employees 
WHERE emp_id NOT IN (
    SELECT MIN(emp_id) FROM employees GROUP BY emp_name, department
);
```

---

### Q38: What is the difference between INNER JOIN and OUTER JOIN? When would you use each?

```sql
-- INNER: Only show employees WHO HAVE a matching department
SELECT e.emp_name, d.budget
FROM employees e INNER JOIN departments d ON e.department = d.dept_name;
-- Missing: Employees in departments not in departments table

-- LEFT OUTER: Show ALL employees, even without department match
SELECT e.emp_name, d.budget
FROM employees e LEFT JOIN departments d ON e.department = d.dept_name;
-- Includes everyone, budget = NULL for unmatched
```

**🎯 Rule of Thumb**: Use INNER JOIN when you only want matching data. Use LEFT JOIN when you want to keep all records from one side and identify missing relationships.

---

### Q39: Explain ACID properties with a real-world scenario.

```
SCENARIO: Online ticket booking

A = Atomicity
  → Either the seat is booked AND payment is charged, 
    or NEITHER happens. No "paid but no seat" scenario.

C = Consistency  
  → Seat count can never go negative.
    Constraints (CHECK seat_count >= 0) enforced.

I = Isolation
  → Two people booking the LAST seat simultaneously:
    Only ONE succeeds. The other gets "sold out."

D = Durability
  → Server crashes 1 second after "Booking confirmed!"
    When it restarts, your booking is STILL there.
```

---

### Q40: What are Isolation Levels? Explain each.

```
┌──────────────────────┬───────────┬──────────────┬───────────────┐
│ Isolation Level      │ Dirty     │ Non-Repeat.  │ Phantom       │
│                      │ Read      │ Read         │ Read          │
├──────────────────────┼───────────┼──────────────┼───────────────┤
│ READ UNCOMMITTED     │ ✅ Yes    │ ✅ Yes       │ ✅ Yes        │
│ READ COMMITTED       │ ❌ No     │ ✅ Yes       │ ✅ Yes        │
│ REPEATABLE READ      │ ❌ No     │ ❌ No        │ ✅ Yes        │
│ SERIALIZABLE         │ ❌ No     │ ❌ No        │ ❌ No         │
├──────────────────────┼───────────┼──────────────┼───────────────┤
│ ↑ More isolation     │           │              │               │
│ ↓ More concurrency   │           │              │               │
└──────────────────────┴───────────┴──────────────┴───────────────┘

Database Defaults:
  Oracle       → READ COMMITTED
  PostgreSQL   → READ COMMITTED  
  SQL Server   → READ COMMITTED
  MySQL        → REPEATABLE READ ← different!
```

---

### Q41: What is a Deadlock? How do you prevent it?

```
DEADLOCK:
  Transaction A holds Lock 1, waits for Lock 2
  Transaction B holds Lock 2, waits for Lock 1
  → Neither can proceed → Database kills one (victim)

  T_A:  LOCK(Row1) ────→ WAIT(Row2) ──→ 💀 KILLED
                              ↕
  T_B:  LOCK(Row2) ────→ WAIT(Row1) ──→ ✅ Proceeds
```

**Prevention strategies:**
1. Always lock resources in the **same order**
2. Keep transactions **short**
3. Use **row-level** locking, not table-level
4. Use **NOWAIT** or **SKIP LOCKED** to fail fast
5. Implement **retry logic** in application code

---

### Q42: What is the difference between OLTP and OLAP?

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Feature          │ OLTP                 │ OLAP                 │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Purpose          │ Day-to-day operations│ Analysis & Reporting │
│ Queries          │ Simple, short        │ Complex, aggregations│
│ Users            │ Many (thousands)     │ Few (analysts)       │
│ Data model       │ Normalized (3NF)     │ Denormalized (Star)  │
│ Response time    │ Milliseconds         │ Seconds to minutes   │
│ Data size        │ GB to TB             │ TB to PB             │
│ Example DB       │ PostgreSQL, MySQL    │ Redshift, BigQuery   │
│ Example query    │ "Get order #12345"   │ "Revenue by region   │
│                  │                      │  last 3 years"       │
└──────────────────┴──────────────────────┴──────────────────────┘
```

---

### Q43: What is a Trigger? Give an example.

```sql
-- PostgreSQL Trigger: Auto-update modified_at timestamp
CREATE OR REPLACE FUNCTION update_modified()
RETURNS TRIGGER AS $$
BEGIN
    NEW.modified_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_modified
BEFORE UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION update_modified();

-- SQL Server Trigger: Audit salary changes
CREATE TRIGGER trg_salary_audit
ON employees
AFTER UPDATE
AS
BEGIN
    INSERT INTO salary_audit (emp_id, old_salary, new_salary, changed_at)
    SELECT i.emp_id, d.salary, i.salary, GETDATE()
    FROM inserted i
    JOIN deleted d ON i.emp_id = d.emp_id
    WHERE i.salary != d.salary;
END;
```

---

### Q44: What is a Cursor? Why should you avoid it?

```sql
-- Cursor: Process rows ONE BY ONE (usually bad idea)
DECLARE emp_cursor CURSOR FOR SELECT emp_name, salary FROM employees;
OPEN emp_cursor;
FETCH NEXT FROM emp_cursor INTO @name, @salary;
WHILE @@FETCH_STATUS = 0
BEGIN
    -- Process each row individually
    PRINT @name + ': ' + CAST(@salary AS VARCHAR);
    FETCH NEXT FROM emp_cursor INTO @name, @salary;
END;
CLOSE emp_cursor;
DEALLOCATE emp_cursor;
```

**Why avoid cursors?**
- 🐌 Row-by-row = slow (RBAR: Row By Agonizing Row)
- 🔒 Holds locks longer
- 💾 Uses more memory
- **Alternative**: Set-based operations (JOINs, Window Functions, CTEs) are almost always better

---

### Q45: How do you optimize a slow query? ⭐🔥

```
STEP-BY-STEP QUERY OPTIMIZATION:

┌─────────────────────────────────────────────────────────────┐
│ 1. EXPLAIN / EXPLAIN ANALYZE the query                     │
│    → Look for: Full Table Scans, Nested Loops on large     │
│       tables, Sort operations                              │
│                                                            │
│ 2. CHECK INDEXES                                           │
│    → Are WHERE/JOIN columns indexed?                       │
│    → Is the index actually being USED? (check plan)        │
│                                                            │
│ 3. MAKE QUERIES SARGable                                   │
│    → ❌ WHERE YEAR(hire_date) = 2020                       │
│    → ✅ WHERE hire_date >= '2020-01-01'                    │
│                                                            │
│ 4. REDUCE DATA EARLY                                       │
│    → Filter in WHERE, not HAVING                           │
│    → Select only needed columns (not SELECT *)             │
│                                                            │
│ 5. AVOID N+1 QUERIES                                       │
│    → Replace loops with JOINs                              │
│                                                            │
│ 6. CHECK STATISTICS                                        │
│    → Outdated statistics = bad execution plans             │
│    → UPDATE STATISTICS / ANALYZE                           │
│                                                            │
│ 7. CONSIDER DENORMALIZATION                                │
│    → For read-heavy queries, add redundant data            │
│    → Materialized views for complex aggregations           │
└─────────────────────────────────────────────────────────────┘
```

---

### Q46: What is SARGable? Why does it matter?

**SARGable** = **S**earch **ARG**ument **able** → Can the query use an index?

```sql
-- ❌ NOT SARGable (function on column = can't use index)
WHERE YEAR(hire_date) = 2020
WHERE UPPER(emp_name) = 'ALICE'
WHERE salary + 1000 > 50000
WHERE emp_name LIKE '%lice'

-- ✅ SARGable (column untouched = CAN use index)
WHERE hire_date >= '2020-01-01' AND hire_date < '2021-01-01'
WHERE emp_name = 'Alice'
WHERE salary > 49000
WHERE emp_name LIKE 'Ali%'
```

> 💡 **Pro Tip**: "Never put a function on the LEFT side of a WHERE clause" is the simplest rule to remember.

---

### Q47: What is the N+1 Query Problem?

```
THE PROBLEM:
  Query 1: SELECT * FROM departments;                    -- 1 query
  For each department:
    Query 2: SELECT * FROM employees WHERE dept = ?;     -- N queries
  
  Total: 1 + N queries → 1 + 100 departments = 101 queries! 💀

THE SOLUTION:
  Single query with JOIN:
  SELECT d.*, e.* 
  FROM departments d 
  LEFT JOIN employees e ON d.dept_name = e.department;   -- 1 query ✅
```

---

### Q48: What is the difference between UNION, INTERSECT, and EXCEPT?

```sql
-- UNION: All rows from both (no duplicates)
SELECT city FROM employees
UNION
SELECT location FROM departments;

-- INTERSECT: Only rows in BOTH
SELECT city FROM employees
INTERSECT
SELECT location FROM departments;

-- EXCEPT (MINUS in Oracle): Rows in first but NOT in second
SELECT city FROM employees
EXCEPT
SELECT location FROM departments;
```

---

### Q49: Write a query to get a comma-separated list of employees per department.

```sql
-- PostgreSQL / MySQL 8+
SELECT department, STRING_AGG(emp_name, ', ' ORDER BY emp_name) AS team
FROM employees
GROUP BY department;

-- SQL Server
SELECT department, STRING_AGG(emp_name, ', ') WITHIN GROUP (ORDER BY emp_name) AS team
FROM employees
GROUP BY department;

-- MySQL (older versions)
SELECT department, GROUP_CONCAT(emp_name ORDER BY emp_name SEPARATOR ', ') AS team
FROM employees
GROUP BY department;

-- Oracle
SELECT department, LISTAGG(emp_name, ', ') WITHIN GROUP (ORDER BY emp_name) AS team
FROM employees
GROUP BY department;
```

---

### Q50: What is the difference between WHERE and ON in a JOIN?

```sql
-- Scenario: LEFT JOIN employees with a filter

-- Filter in ON: Affects how the JOIN works (keeps all left rows)
SELECT e.emp_name, d.budget
FROM employees e
LEFT JOIN departments d ON e.department = d.dept_name AND d.budget > 500000;
-- Returns ALL employees; budget = NULL if dept budget ≤ 500K

-- Filter in WHERE: Filters AFTER the join (eliminates left rows too!)
SELECT e.emp_name, d.budget
FROM employees e
LEFT JOIN departments d ON e.department = d.dept_name
WHERE d.budget > 500000;
-- Returns ONLY employees in departments with budget > 500K
-- ⚠️ This effectively becomes an INNER JOIN!
```

> ⚠️ **This is a VERY common interview trap.** For INNER JOINs, ON vs WHERE is identical. For OUTER JOINs, they're completely different!

---

### Q51: What is the difference between APPLY (CROSS/OUTER) in SQL Server and LATERAL JOIN in PostgreSQL?

```sql
-- SQL Server: CROSS APPLY (like INNER JOIN with table-valued function)
SELECT e.emp_name, t.recent_order
FROM employees e
CROSS APPLY (
    SELECT TOP 1 order_id AS recent_order
    FROM orders o
    WHERE o.emp_id = e.emp_id
    ORDER BY order_date DESC
) t;

-- PostgreSQL: LATERAL JOIN (same concept)
SELECT e.emp_name, t.recent_order
FROM employees e,
LATERAL (
    SELECT order_id AS recent_order
    FROM orders o
    WHERE o.emp_id = e.emp_id
    ORDER BY order_date DESC
    LIMIT 1
) t;

-- OUTER APPLY / LEFT JOIN LATERAL: Keep rows with no match too
```

---

### Q52: Write a query for Year-over-Year growth.

```sql
WITH yearly_revenue AS (
    SELECT 
        EXTRACT(YEAR FROM order_date) AS yr,
        SUM(amount) AS total_revenue
    FROM orders
    GROUP BY EXTRACT(YEAR FROM order_date)
)
SELECT 
    yr,
    total_revenue,
    LAG(total_revenue) OVER (ORDER BY yr) AS prev_year,
    ROUND(
        (total_revenue - LAG(total_revenue) OVER (ORDER BY yr)) * 100.0 
        / LAG(total_revenue) OVER (ORDER BY yr), 2
    ) AS yoy_growth_pct
FROM yearly_revenue;
```

---

### Q53: Explain MERGE (UPSERT) statement.

```sql
-- SQL Server / Oracle: MERGE
MERGE INTO employees AS target
USING new_employees AS source
ON target.emp_id = source.emp_id
WHEN MATCHED THEN
    UPDATE SET salary = source.salary
WHEN NOT MATCHED THEN
    INSERT (emp_id, emp_name, salary) 
    VALUES (source.emp_id, source.emp_name, source.salary);

-- PostgreSQL: INSERT ... ON CONFLICT
INSERT INTO employees (emp_id, emp_name, salary)
VALUES (1, 'Alice', 125000)
ON CONFLICT (emp_id) DO UPDATE SET salary = EXCLUDED.salary;

-- MySQL: INSERT ... ON DUPLICATE KEY UPDATE
INSERT INTO employees (emp_id, emp_name, salary)
VALUES (1, 'Alice', 125000)
ON DUPLICATE KEY UPDATE salary = VALUES(salary);
```

---

### Q54: What is a Temporary Table vs Table Variable vs CTE?

```
┌────────────────┬──────────────────┬──────────────────┬──────────────┐
│ Feature        │ Temp Table       │ Table Variable   │ CTE          │
├────────────────┼──────────────────┼──────────────────┼──────────────┤
│ Scope          │ Session/Global   │ Batch/Procedure  │ Single query │
│ Stored in      │ tempdb (disk)    │ Memory (mostly)  │ Not stored   │
│ Indexable      │ ✅ Yes           │ ⚠️ Limited       │ ❌ No        │
│ Statistics     │ ✅ Yes           │ ❌ No            │ N/A          │
│ Transactions   │ ✅ Participates  │ ❌ Independent   │ N/A          │
│ Best for       │ Large datasets   │ Small datasets   │ Readability  │
│ Reusable       │ ✅ In session    │ ✅ In batch      │ ❌ One query │
└────────────────┴──────────────────┴──────────────────┴──────────────┘
```

---

### Q55: Write a query to find employees hired in the last 30 days.

```sql
-- PostgreSQL
SELECT * FROM employees WHERE hire_date >= CURRENT_DATE - INTERVAL '30 days';

-- MySQL
SELECT * FROM employees WHERE hire_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY);

-- SQL Server
SELECT * FROM employees WHERE hire_date >= DATEADD(DAY, -30, GETDATE());

-- Oracle
SELECT * FROM employees WHERE hire_date >= SYSDATE - 30;
```

> 💡 **Pro Tip**: In interviews, always show you know the difference across databases. It demonstrates breadth.

---

# 🔴 SECTION 3: ADVANCED SQL (Questions 56–80)

> _These questions separate senior developers from everyone else. If you nail these, you're in the top 5%._

---

### Q56: Write a query to solve the "Gaps and Islands" problem. ⭐🔥

```sql
-- Given: Login dates with gaps. Find consecutive "islands" of activity.
-- Data: 2024-01-01, 01-02, 01-03, 01-06, 01-07, 01-10

WITH numbered AS (
    SELECT login_date,
        login_date - (ROW_NUMBER() OVER (ORDER BY login_date) * INTERVAL '1 day') AS grp
    FROM user_logins
)
SELECT 
    MIN(login_date) AS island_start,
    MAX(login_date) AS island_end,
    COUNT(*) AS consecutive_days
FROM numbered
GROUP BY grp
ORDER BY island_start;
```

```
Result:
island_start  | island_end  | consecutive_days
2024-01-01    | 2024-01-03  | 3
2024-01-06    | 2024-01-07  | 2
2024-01-10    | 2024-01-10  | 1
```

**How it works**: When you subtract row_number from a sequential date, consecutive dates produce the SAME group value. Gaps create different groups.

---

### Q57: Write a query for a Moving Average (last 3 months). ⭐

```sql
SELECT 
    order_month,
    monthly_revenue,
    AVG(monthly_revenue) OVER (
        ORDER BY order_month 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3m
FROM (
    SELECT 
        DATE_TRUNC('month', order_date) AS order_month,
        SUM(amount) AS monthly_revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
) monthly;
```

```
Window Frame Options:
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW     → Last 3 rows (fixed)
  RANGE BETWEEN INTERVAL '2' MONTH PRECEDING    → Calendar-based
  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT  → Running total
```

---

### Q58: What is a Covering Index and why is it powerful?

```sql
-- Query: Find employee names in Engineering
SELECT emp_name, salary FROM employees WHERE department = 'Engineering';

-- Regular index: Index lookup → then go to table for name, salary (bookmark lookup)
CREATE INDEX idx_dept ON employees(department);

-- Covering index: ALL needed columns are IN the index → no table access needed! ⚡
CREATE INDEX idx_dept_covering ON employees(department) INCLUDE (emp_name, salary);
-- PostgreSQL: INCLUDE syntax
-- SQL Server: INCLUDE syntax
-- MySQL: Add columns to index directly
```

```
Regular Index:                     Covering Index:
Index → Find row IDs              Index → Return data directly
  ↓                               (No table access needed!)
Table → Fetch actual data         ⚡ 10-100x faster for
  ↓                                  covered queries
Return result
```

---

### Q59: Explain Query Execution Plan. What should you look for?

```
EXPLAIN ANALYZE SELECT * FROM employees WHERE department = 'Engineering';

KEY THINGS TO CHECK:
┌─────────────────────────────────────────────────────────────┐
│ 🔴 DANGER SIGNS:                                           │
│  • Seq Scan / Full Table Scan on large tables              │
│  • Nested Loop on large tables (should be Hash/Merge Join) │
│  • Sort operation without index                            │
│  • High "Actual Rows" vs "Estimated Rows" (stats outdated)│
│  • Bitmap Heap Scan re-checking lots of rows               │
│                                                            │
│ ✅ GOOD SIGNS:                                              │
│  • Index Scan / Index Only Scan                            │
│  • Hash Join on large tables                               │
│  • Merge Join on sorted data                               │
│  • Low actual rows (good selectivity)                      │
│  • "Index Only Scan" = covering index working              │
└─────────────────────────────────────────────────────────────┘
```

---

### Q60: What is Parameter Sniffing? How do you fix it?

```
PROBLEM:
  A stored procedure is compiled with execution plan for parameter 'NY' 
  (10 rows → Index Seek). Then called with parameter 'ALL' 
  (10M rows → still uses Index Seek plan → SLOW!)

  sp_get_orders @city = 'NY'     → Plan: Index Seek (fast ✅)
  sp_get_orders @city = 'ALL'    → Same plan: Index Seek (slow 💀)
                                    Should use Table Scan!
```

**Fixes:**
```sql
-- SQL Server Fix 1: OPTIMIZE FOR UNKNOWN
SELECT * FROM orders WHERE city = @city OPTION (OPTIMIZE FOR UNKNOWN);

-- SQL Server Fix 2: RECOMPILE (new plan each time)
EXEC sp_get_orders @city = 'ALL' WITH RECOMPILE;

-- SQL Server Fix 3: Local variable (breaks the sniff)
DECLARE @local_city VARCHAR(50) = @city;
SELECT * FROM orders WHERE city = @local_city;

-- PostgreSQL Fix: plan_cache_mode = force_generic_plan
```

---

### Q61: Write a query to implement pagination efficiently.

```sql
-- ❌ OFFSET pagination (SLOW for deep pages — scans all previous rows)
SELECT * FROM employees ORDER BY emp_id LIMIT 20 OFFSET 10000;
-- Must scan and discard 10,000 rows first! 💀

-- ✅ Keyset/Cursor pagination (FAST — always O(1) index seek)
SELECT * FROM employees 
WHERE emp_id > 10000          -- Last seen ID
ORDER BY emp_id 
LIMIT 20;
```

```
OFFSET (Page 500):                KEYSET (Page 500):
┌────────────────────┐           ┌────────────────────┐
│ Scan 10,000 rows   │           │ Index seek to      │
│ Discard them       │           │ emp_id > 10000     │
│ Return 20          │           │ Return 20          │
│ Time: 500ms 🐌     │           │ Time: 2ms ⚡        │
└────────────────────┘           └────────────────────┘
```

---

### Q62: What is a Partial (Filtered) Index?

```sql
-- Only index active orders (90% of orders are completed → don't index those!)
CREATE INDEX idx_active_orders ON orders(order_date, amount) 
WHERE status = 'active';   -- PostgreSQL

-- SQL Server
CREATE INDEX idx_active_orders ON orders(order_date, amount)
WHERE status = 'active';   -- Filtered index
```

**Benefit**: Smaller index → fits in memory → faster lookups → less maintenance.

---

### Q63: Explain the difference between Pessimistic and Optimistic Locking.

```
PESSIMISTIC (Lock first, then work):
  BEGIN;
  SELECT * FROM products WHERE id = 1 FOR UPDATE;  -- LOCKS the row
  -- ... do calculations ...
  UPDATE products SET stock = stock - 1 WHERE id = 1;
  COMMIT;  -- Release lock
  
  ✅ Safe for high contention
  ❌ Blocks other transactions

OPTIMISTIC (Work first, check at save):
  SELECT *, version FROM products WHERE id = 1;  -- version = 5
  -- ... do calculations ...
  UPDATE products SET stock = stock - 1, version = version + 1
  WHERE id = 1 AND version = 5;  -- Check version hasn't changed!
  -- If 0 rows updated → someone else changed it → RETRY
  
  ✅ Better concurrency (no locks)
  ❌ Must handle retries
```

---

### Q64: Write a query to find the median salary.

```sql
-- PostgreSQL: Built-in
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median
FROM employees;

-- MySQL 8+ / SQL Server: Window function approach
WITH ordered AS (
    SELECT salary,
        ROW_NUMBER() OVER (ORDER BY salary) AS rn,
        COUNT(*) OVER () AS total
    FROM employees
)
SELECT AVG(salary) AS median
FROM ordered
WHERE rn IN (FLOOR((total + 1) / 2.0), CEIL((total + 1) / 2.0));

-- Oracle
SELECT MEDIAN(salary) FROM employees;
```

---

### Q65: What is MVCC (Multi-Version Concurrency Control)?

```
MVCC: Instead of locking rows, keep MULTIPLE VERSIONS of each row.
Readers don't block writers. Writers don't block readers.

Transaction Timeline:
  T1 (Reader):  BEGIN ──── SELECT(sees V1) ──── SELECT(still V1) ──── COMMIT
  T2 (Writer):  BEGIN ──── UPDATE(creates V2) ──── COMMIT
  
  T1 keeps seeing V1 (snapshot) even after T2 commits.
  No locks, no waiting, no blocking! ⚡

Implementation varies:
  PostgreSQL: Stores old versions IN the table (needs VACUUM)
  Oracle:     Stores old versions in UNDO tablespace
  MySQL:      Stores old versions in undo log (InnoDB)
  SQL Server: Stores old versions in tempdb (when RCSI enabled)
```

---

### Q66: Write a query for Conditional Aggregation. ⭐

```sql
-- Count and sum with conditions in a single query
SELECT 
    department,
    COUNT(*) AS total_employees,
    COUNT(CASE WHEN salary > 100000 THEN 1 END) AS high_earners,
    COUNT(CASE WHEN salary <= 100000 THEN 1 END) AS others,
    SUM(CASE WHEN hire_date >= '2022-01-01' THEN salary ELSE 0 END) AS new_hire_cost,
    ROUND(AVG(CASE WHEN city = 'New York' THEN salary END), 2) AS ny_avg_salary
FROM employees
GROUP BY department;

-- PostgreSQL: FILTER syntax (more readable)
SELECT 
    department,
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE salary > 100000) AS high_earners,
    AVG(salary) FILTER (WHERE city = 'New York') AS ny_avg
FROM employees
GROUP BY department;
```

---

### Q67: What is a Recursive Query? Solve "Bill of Materials" (BOM).

```sql
-- Parts hierarchy: A car contains an engine, which contains pistons...
WITH RECURSIVE bom AS (
    -- Top-level product
    SELECT part_id, part_name, parent_id, 1 AS level, quantity,
           CAST(part_name AS VARCHAR(500)) AS path
    FROM parts
    WHERE parent_id IS NULL

    UNION ALL

    -- Sub-components
    SELECT p.part_id, p.part_name, p.parent_id, b.level + 1, 
           p.quantity * b.quantity AS total_qty,
           CAST(b.path || ' > ' || p.part_name AS VARCHAR(500))
    FROM parts p
    JOIN bom b ON p.parent_id = b.part_id
)
SELECT * FROM bom ORDER BY path;
```

---

### Q68: What is Database Sharding? Explain strategies.

```
SHARDING: Split data HORIZONTALLY across multiple database instances.

┌───────────────────────────────────────────────────┐
│               SHARDING STRATEGIES                  │
├────────────────────┬──────────────────────────────┤
│ Hash-Based         │ shard = hash(user_id) % N    │
│                    │ Even distribution             │
│                    │ Hard to add shards            │
├────────────────────┼──────────────────────────────┤
│ Range-Based        │ Users 1-1M → Shard 1         │
│                    │ Users 1M-2M → Shard 2        │
│                    │ Hot spots possible            │
├────────────────────┼──────────────────────────────┤
│ Directory-Based    │ Lookup table maps key→shard   │
│                    │ Flexible, but extra hop       │
├────────────────────┼──────────────────────────────┤
│ Geo-Based          │ US users → US shard           │
│                    │ EU users → EU shard           │
│                    │ Lower latency                 │
└────────────────────┴──────────────────────────────┘
```

---

### Q69: What is the CAP Theorem? Give real examples.

```
CAP: You can only guarantee 2 of 3 in a distributed system:

              C (Consistency)
             / \
            /   \
           /     \
     CP Systems  CA Systems
     MongoDB      PostgreSQL
     HBase        MySQL
     Redis        (single-node)
          \       /
           \     /
            \   /
         AP Systems
         Cassandra
         DynamoDB
         CouchDB

              A (Availability)  ─────  P (Partition Tolerance)

In practice: Network partitions WILL happen. So you really choose between CP and AP.
```

---

### Q70: Write a query using GROUPING SETS, ROLLUP, and CUBE.

```sql
-- GROUPING SETS: Specific combinations of groupings
SELECT department, city, SUM(salary)
FROM employees
GROUP BY GROUPING SETS (
    (department, city),     -- Dept + City
    (department),           -- Dept total
    (city),                 -- City total
    ()                      -- Grand total
);

-- ROLLUP: Hierarchical subtotals (Dept → Dept+City → Grand)
SELECT department, city, SUM(salary)
FROM employees
GROUP BY ROLLUP(department, city);

-- CUBE: ALL possible combinations (2^n groupings)
SELECT department, city, SUM(salary)
FROM employees
GROUP BY CUBE(department, city);
```

```
ROLLUP(A, B):              CUBE(A, B):
(A, B)                     (A, B)
(A)                        (A)
()                         (B)     ← Extra!
                           ()
3 groupings                4 groupings
```

---

### Q71: How do you handle hierarchical data in SQL?

```
4 APPROACHES:
┌─────────────────────┬────────────────────────────────────────┐
│ Method              │ Pros/Cons                              │
├─────────────────────┼────────────────────────────────────────┤
│ Adjacency List      │ Simple. parent_id column.              │
│ (parent_id)         │ Slow for deep traversal.               │
├─────────────────────┼────────────────────────────────────────┤
│ Path Enumeration    │ path = '/1/3/7/'. Fast reads.          │
│ (materialized path) │ Expensive moves.                       │
├─────────────────────┼────────────────────────────────────────┤
│ Nested Sets         │ Fast subtree queries. Expensive writes.│
│ (left/right values) │ Complex to maintain.                   │
├─────────────────────┼────────────────────────────────────────┤
│ Closure Table       │ Separate table for all paths.          │
│                     │ Fast reads AND writes. Extra storage.  │
└─────────────────────┴────────────────────────────────────────┘
```

---

### Q72: What is the difference between CHAR, VARCHAR, NCHAR, and NVARCHAR?

```
┌──────────┬────────────┬───────────────┬────────────────────────┐
│ Type     │ Fixed/Var  │ Character Set │ Bytes per char         │
├──────────┼────────────┼───────────────┼────────────────────────┤
│ CHAR     │ Fixed      │ Non-Unicode   │ 1 byte                 │
│ VARCHAR  │ Variable   │ Non-Unicode   │ 1 byte                 │
│ NCHAR    │ Fixed      │ Unicode       │ 2 bytes (UCS-2/UTF-16) │
│ NVARCHAR │ Variable   │ Unicode       │ 2 bytes (UCS-2/UTF-16) │
└──────────┴────────────┴───────────────┴────────────────────────┘
```

> 💡 **Pro Tip**: If your application supports multiple languages (Chinese, Arabic, emoji), always use NVARCHAR. In PostgreSQL, VARCHAR already supports Unicode via UTF-8 encoding.

---

### Q73: What are Window Frame clauses? Explain ROWS vs RANGE.

```sql
SELECT emp_name, salary,
    SUM(salary) OVER (
        ORDER BY salary 
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS rows_sum,
    SUM(salary) OVER (
        ORDER BY salary 
        RANGE BETWEEN 1000 PRECEDING AND 1000 FOLLOWING
    ) AS range_sum
FROM employees;
```

```
ROWS:  Physical row count (exactly 1 row before, 1 row after)
RANGE: Logical value range (salary within ±1000 of current)

Data: Salaries = 78K, 85K, 88K, 90K, 92K

For salary 88K:
  ROWS (1 PREC, 1 FOLLOW):  85K + 88K + 90K = 263K  (3 rows)
  RANGE (2000 PREC, 2000 FOLLOW):  88K + 90K = 178K  (salary between 86K-90K)
```

---

### Q74: Write a query to detect consecutive events (streaks).

```sql
-- Find employees who logged in for 3+ consecutive days
WITH streaks AS (
    SELECT 
        user_id,
        login_date,
        login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int AS grp
    FROM daily_logins
)
SELECT 
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*) AS streak_length
FROM streaks
GROUP BY user_id, grp
HAVING COUNT(*) >= 3
ORDER BY streak_length DESC;
```

---

### Q75: Explain different JOIN algorithms used by databases internally.

```
┌──────────────────┬────────────────────────────────────────────┐
│ Algorithm        │ How It Works                               │
├──────────────────┼────────────────────────────────────────────┤
│ Nested Loop      │ For each row in A, scan B for matches.    │
│                  │ Best: Small outer table + indexed inner.   │
│                  │ Cost: O(N × M) worst case.                │
├──────────────────┼────────────────────────────────────────────┤
│ Hash Join        │ Build hash table on smaller table.         │
│                  │ Probe with larger table.                   │
│                  │ Best: Large unsorted tables, equality.     │
│                  │ Cost: O(N + M) but needs memory.          │
├──────────────────┼────────────────────────────────────────────┤
│ Merge Join       │ Sort both tables, merge like a zipper.    │
│                  │ Best: Already sorted data (index).         │
│                  │ Cost: O(N log N + M log M) if unsorted.   │
│                  │ Cost: O(N + M) if pre-sorted.             │
└──────────────────┴────────────────────────────────────────────┘
```

---

### Q76: What is a Composite Index? How does column order matter?

```sql
CREATE INDEX idx_dept_salary ON employees(department, salary);
```

```
This index is useful for:
  ✅ WHERE department = 'Engineering'                    (leftmost prefix)
  ✅ WHERE department = 'Engineering' AND salary > 100K  (both columns)
  ✅ ORDER BY department, salary                          (same order)

This index is NOT useful for:
  ❌ WHERE salary > 100000                (skips leftmost column!)
  ❌ ORDER BY salary, department           (wrong order)
  
THE RULE: "Leftmost Prefix" — index on (A, B, C) supports:
  ✅ (A)
  ✅ (A, B)
  ✅ (A, B, C)
  ❌ (B)
  ❌ (B, C)
  ❌ (C)
```

---

### Q77: Write a query for "Top N per Group" — the efficient way.

```sql
-- Get the 2 most recent orders per customer (PostgreSQL, MySQL 8+)
SELECT *
FROM (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
) sub
WHERE rn <= 2;

-- Optimized with LATERAL JOIN (PostgreSQL) — uses index better
SELECT c.customer_id, c.name, o.*
FROM customers c
CROSS JOIN LATERAL (
    SELECT * FROM orders 
    WHERE customer_id = c.customer_id 
    ORDER BY order_date DESC 
    LIMIT 2
) o;
```

---

### Q78: How do you handle Time Zones in SQL databases?

```sql
-- PostgreSQL: TIMESTAMPTZ (stores in UTC, displays in session timezone)
SELECT NOW() AT TIME ZONE 'US/Eastern' AS eastern_time;
SET timezone = 'Asia/Kolkata';

-- MySQL: CONVERT_TZ
SELECT CONVERT_TZ(NOW(), 'UTC', 'Asia/Kolkata');

-- SQL Server: AT TIME ZONE
SELECT GETUTCDATE() AT TIME ZONE 'India Standard Time';

-- Oracle: FROM_TZ + AT TIME ZONE
SELECT FROM_TZ(SYSTIMESTAMP, 'UTC') AT TIME ZONE 'Asia/Calcutta' FROM DUAL;
```

> 💡 **Pro Tip**: ALWAYS store timestamps in UTC. Convert to local timezone only at display time. This prevents countless timezone-related bugs.

---

### Q79: What is a Sequence vs Identity/Auto-Increment?

```sql
-- PostgreSQL: SERIAL / IDENTITY
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,           -- Auto-increment (legacy)
    id INT GENERATED ALWAYS AS IDENTITY  -- Modern standard
);

-- MySQL: AUTO_INCREMENT
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY
);

-- Oracle: SEQUENCE (separate object)
CREATE SEQUENCE order_seq START WITH 1 INCREMENT BY 1;
INSERT INTO orders VALUES (order_seq.NEXTVAL, 'product1');

-- SQL Server: IDENTITY
CREATE TABLE orders (
    id INT IDENTITY(1,1) PRIMARY KEY
);
```

**Sequence advantages**: Can be shared across tables, can cache values, supports CYCLE, more flexible than identity.

---

### Q80: What are the differences between SQL Server, Oracle, MySQL, and PostgreSQL?

```
┌──────────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│ Feature          │ PostgreSQL   │ MySQL        │ SQL Server   │ Oracle       │
├──────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ License          │ Open Source  │ Open Source* │ Commercial   │ Commercial   │
│ Default ISO Level│ Read Commit  │ Repeat Read  │ Read Commit  │ Read Commit  │
│ JSON Support     │ ⭐ JSONB     │ ✅ JSON      │ ✅ JSON      │ ✅ JSON      │
│ Procedural Lang  │ PL/pgSQL     │ Stored Procs │ T-SQL        │ PL/SQL       │
│ Pagination       │ LIMIT OFFSET │ LIMIT OFFSET │ OFFSET FETCH │ ROWNUM/FETCH │
│ String concat    │ ||           │ CONCAT()     │ + or CONCAT  │ ||           │
│ NULL sort default│ NULLS LAST   │ NULLS FIRST  │ NULLS FIRST  │ NULLS LAST   │
│ Boolean type     │ ✅ Native    │ TINYINT(1)   │ BIT          │ NUMBER(1)    │
│ UPSERT syntax    │ ON CONFLICT  │ ON DUPLICATE │ MERGE        │ MERGE        │
│ Window functions │ ⭐ Full      │ ✅ (8.0+)    │ ✅ Full      │ ✅ Full      │
│ Full-text search │ ✅ Built-in  │ ✅ Built-in  │ ✅ Built-in  │ Oracle Text  │
│ Geospatial       │ PostGIS ⭐   │ ✅ Basic     │ ✅ Good      │ ✅ SDO       │
└──────────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

---

# 🔥 SECTION 4: EXPERT / SYSTEM DESIGN SQL (Questions 81–100)

> _These questions are asked at FAANG, unicorn startups, and senior DBA interviews. This is where you become unstoppable._

---

### Q81: How would you design a database for a URL Shortener (like bit.ly)?

```sql
CREATE TABLE short_urls (
    id          BIGINT PRIMARY KEY,           -- Snowflake ID or sequence
    short_code  VARCHAR(10) UNIQUE NOT NULL,   -- Base62 encoded
    original_url TEXT NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    expires_at  TIMESTAMPTZ,
    click_count BIGINT DEFAULT 0,
    created_by  INT REFERENCES users(id)
);

-- Indexes
CREATE INDEX idx_short_code ON short_urls(short_code);  -- Main lookup
CREATE INDEX idx_expires ON short_urls(expires_at) WHERE expires_at IS NOT NULL;

-- Click analytics (separate table for write performance)
CREATE TABLE url_clicks (
    click_id    BIGSERIAL PRIMARY KEY,
    short_code  VARCHAR(10),
    clicked_at  TIMESTAMPTZ DEFAULT NOW(),
    ip_address  INET,
    user_agent  TEXT,
    referrer    TEXT,
    country     VARCHAR(2)
) PARTITION BY RANGE (clicked_at);  -- Partition by month!
```

**Key decisions:**
- **Base62 encoding** for short codes (a-z, A-Z, 0-9)
- **Separate analytics table** — writes won't slow reads
- **Partitioning** by time for analytics data
- **Redis cache** in front for hot short codes

---

### Q82: How would you design a database for a Chat application?

```sql
-- Users
CREATE TABLE users (
    user_id     BIGSERIAL PRIMARY KEY,
    username    VARCHAR(50) UNIQUE,
    status      VARCHAR(20) DEFAULT 'offline'
);

-- Conversations (1:1 and group chats)
CREATE TABLE conversations (
    conv_id     BIGSERIAL PRIMARY KEY,
    type        VARCHAR(10) CHECK (type IN ('direct', 'group')),
    name        VARCHAR(100),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Participants
CREATE TABLE participants (
    conv_id     BIGINT REFERENCES conversations(conv_id),
    user_id     BIGINT REFERENCES users(user_id),
    joined_at   TIMESTAMPTZ DEFAULT NOW(),
    last_read   TIMESTAMPTZ,
    PRIMARY KEY (conv_id, user_id)
);

-- Messages (partitioned by time!)
CREATE TABLE messages (
    msg_id      BIGSERIAL,
    conv_id     BIGINT NOT NULL,
    sender_id   BIGINT NOT NULL,
    content     TEXT,
    msg_type    VARCHAR(10) DEFAULT 'text',
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (msg_id, created_at)
) PARTITION BY RANGE (created_at);

-- Key indexes
CREATE INDEX idx_messages_conv ON messages(conv_id, created_at DESC);
CREATE INDEX idx_participants_user ON participants(user_id);
```

**Scaling decisions:**
- Shard by `conv_id` — all messages for a chat stay on same shard
- Partition by month — old messages can be archived
- Use **Cassandra or ScyllaDB** for message storage at scale (write-heavy)
- **Redis** for online status and unread counts

---

### Q83: Explain database connection pooling. Why is it critical?

```
WITHOUT POOLING:                    WITH POOLING:
┌──────────────────┐               ┌──────────────────┐
│ App Server       │               │ App Server       │
│ 1000 requests    │               │ 1000 requests    │
│    ↓             │               │    ↓             │
│ 1000 DB          │               │ Connection Pool  │
│ connections      │               │ (20 connections) │
│    ↓             │               │    ↓             │
│ DB: 💀 DEAD      │               │ DB: ✅ Happy     │
│ (max_connections │               │ (20 connections  │
│  usually 100-500)│               │  reused)         │
└──────────────────┘               └──────────────────┘
```

**Popular poolers**: PgBouncer (PostgreSQL), ProxySQL (MySQL), HikariCP (Java), c3p0 (Java)

**Pool sizing formula** (by PostgreSQL experts):
```
connections = (core_count * 2) + effective_spindle_count
Example: 4 cores, SSD = (4 * 2) + 1 = 9 connections optimal
```

---

### Q84: What is the Write-Ahead Log (WAL)? Why is it essential?

```
WRITE-AHEAD LOG: Write to log FIRST, then to actual data.
If crash happens → replay log to recover.

Transaction Flow:
  1. Write intent to WAL (sequential write → FAST)
  2. Return "COMMIT OK" to client
  3. Later: Write actual data pages to disk (background)
  4. If crash before step 3 → replay WAL on recovery ✅

Without WAL:
  1. Write directly to data pages (random I/O)
  2. Crash mid-write → corrupted data 💀
```

**Every serious database uses WAL**: PostgreSQL (pg_wal), MySQL (redo log), Oracle (redo log), SQL Server (transaction log).

---

### Q85: How would you migrate a database with zero downtime?

```
ZERO-DOWNTIME MIGRATION STRATEGY:

Phase 1: EXPAND
  ┌──────┐    ┌──────┐
  │ Old  │───→│ New  │  Dual-write to both
  │  DB  │    │  DB  │  OR use CDC (Change Data Capture)
  └──────┘    └──────┘

Phase 2: MIGRATE
  • Backfill historical data to new DB
  • Verify data integrity (checksums)

Phase 3: SWITCH
  • Point reads to new DB (shadow traffic first)
  • Compare results (old vs new)
  • Gradually shift traffic: 1% → 10% → 50% → 100%

Phase 4: CONTRACT
  • Stop writes to old DB
  • Keep old DB read-only for 1 week (safety net)
  • Decommission old DB

Tools: Flyway, Liquibase, gh-ost (GitHub), pt-online-schema-change
```

---

### Q86: What is Event Sourcing and how does it relate to databases?

```
TRADITIONAL:                     EVENT SOURCING:
┌──────────────────┐            ┌────────────────────────────┐
│ Current state    │            │ All events (append-only)   │
│ Balance: $500    │            │ 1. Account opened: $0      │
│                  │            │ 2. Deposited: +$1000       │
│ (overwritten     │            │ 3. Withdrew: -$200         │
│  each time)      │            │ 4. Deposited: +$300        │
│                  │            │ 5. Withdrew: -$600         │
│                  │            │ Current: $500 (computed)   │
└──────────────────┘            └────────────────────────────┘

Advantages: Full audit trail, time-travel, replay events
Database: Kafka + PostgreSQL, EventStoreDB, DynamoDB Streams
```

---

### Q87: What is CQRS and when would you use it?

```
CQRS: Command Query Responsibility Segregation

Traditional:                          CQRS:
┌──────┐                             ┌──────┐    ┌──────────┐
│ App  │──Read/Write──→ ┌────┐       │ App  │──W→│ Write DB │ (normalized)
│      │                │ DB │       │      │    └──────────┘
└──────┘                └────┘       │      │         │ (sync: events)
                                     │      │         ↓
                                     │      │──R→┌──────────┐
                                     └──────┘    │ Read DB  │ (denormalized)
                                                 └──────────┘

USE WHEN:
  ✅ Read/write patterns are VERY different
  ✅ Need different models for reads vs writes
  ✅ High read:write ratio (100:1)
  ❌ Simple CRUD apps (over-engineering)
```

---

### Q88: How would you design a Rate Limiter using SQL?

```sql
-- Sliding window rate limiter (100 requests per minute per user)
CREATE TABLE rate_limits (
    user_id     INT,
    request_at  TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_rate (user_id, request_at)
);

-- Check if user exceeded limit
SELECT COUNT(*) AS request_count
FROM rate_limits
WHERE user_id = 42 
  AND request_at > NOW() - INTERVAL '1 minute';

-- If request_count < 100, allow and insert
INSERT INTO rate_limits (user_id) VALUES (42);

-- Cleanup old records periodically  
DELETE FROM rate_limits WHERE request_at < NOW() - INTERVAL '1 hour';
```

> 💡 **Pro Tip**: In production, use Redis with INCR + EXPIRE for rate limiting. SQL is too slow for this at scale. But this question tests your SQL design skills.

---

### Q89: What are the differences between Row-Level and Statement-Level Triggers?

```sql
-- ROW-LEVEL: Fires ONCE for EACH affected row
CREATE TRIGGER trg_row
AFTER UPDATE ON employees
FOR EACH ROW         -- ← Row-level
EXECUTE FUNCTION log_change();
-- UPDATE 100 rows → trigger fires 100 times

-- STATEMENT-LEVEL: Fires ONCE per SQL statement
CREATE TRIGGER trg_stmt
AFTER UPDATE ON employees
FOR EACH STATEMENT   -- ← Statement-level
EXECUTE FUNCTION log_batch();
-- UPDATE 100 rows → trigger fires 1 time
```

---

### Q90: What is the Saga Pattern for distributed transactions?

```
SAGA: A sequence of local transactions with compensating actions.

Order Service          Payment Service        Inventory Service
     │                       │                       │
     │──Create Order────────→│                       │
     │     (T1)              │──Charge Payment──────→│
     │                       │     (T2)              │──Reserve Stock──→
     │                       │                       │     (T3)
     │                       │                       │
     │  If T3 FAILS:         │                       │
     │                       │←──Refund Payment──────│  (C2: compensate)
     │←──Cancel Order────────│                       │  (C1: compensate)
     │                       │                       │

Two styles:
  Choreography: Each service publishes events, others react
  Orchestration: Central coordinator directs the saga
```

---

### Q91: How do you handle Slowly Changing Dimensions (SCD)?

```
TYPE 1: Overwrite (lose history)
  Old: {city: 'New York'}  →  New: {city: 'Seattle'}
  
TYPE 2: Add new row (keep history) ⭐ Most common
  {id: 1, city: 'New York', valid_from: '2020-01-01', valid_to: '2024-06-01', current: false}
  {id: 1, city: 'Seattle',  valid_from: '2024-06-01', valid_to: '9999-12-31', current: true}

TYPE 3: Add new column (limited history)
  {id: 1, current_city: 'Seattle', previous_city: 'New York'}
  
TYPE 4: Mini-dimension (separate history table)
TYPE 6: Hybrid of 1+2+3
```

---

### Q92: What are Phantom Reads and how do you prevent them?

```
PHANTOM READ: A transaction re-runs a query and gets DIFFERENT ROWS
(not different values — different rows appearing/disappearing)

T1: SELECT * FROM employees WHERE salary > 100K;  → 3 rows
T2: INSERT INTO employees VALUES (11, 'New Guy', 'Eng', 150000, ...);
T2: COMMIT;
T1: SELECT * FROM employees WHERE salary > 100K;  → 4 rows! 👻

PREVENTION:
  • SERIALIZABLE isolation level
  • PostgreSQL: Uses SSI (Serializable Snapshot Isolation)
  • MySQL: Uses gap locks in REPEATABLE READ (prevents phantoms!)
  • SQL Server: Uses key-range locks in SERIALIZABLE
```

---

### Q93: Explain the difference between Horizontal and Vertical Scaling.

```
VERTICAL SCALING (Scale Up):          HORIZONTAL SCALING (Scale Out):
┌──────────────────────┐             ┌───┐ ┌───┐ ┌───┐ ┌───┐
│     BIGGER           │             │ S1│ │ S2│ │ S3│ │ S4│
│     SERVER           │             └───┘ └───┘ └───┘ └───┘
│     64 CPU           │                   (many small)
│     512 GB RAM       │
│     (one giant)      │
└──────────────────────┘

Vertical:                            Horizontal:
✅ Simple (no code changes)          ✅ Unlimited scaling
✅ Strong consistency easy           ✅ High availability
❌ Hardware limits                    ❌ Complex (sharding, routing)
❌ Single point of failure           ❌ Distributed transactions hard
❌ Expensive at top tier             ✅ Commodity hardware

SQL databases: Usually start vertical, then horizontal (read replicas, sharding)
NoSQL databases: Designed for horizontal from day 1
```

---

### Q94: What is a Bloom Filter and how do databases use it?

```
BLOOM FILTER: Probabilistic data structure that tells you:
  "Definitely NOT in set" or "PROBABLY in set"

  False Positives: Possible (says "yes" but actually "no")
  False Negatives: IMPOSSIBLE (if it says "no", it's definitely "no")

Database Use Cases:
  • PostgreSQL: Checks if a value might be in a page before loading
  • Cassandra: Checks if a partition key exists before disk access
  • HBase: Avoids unnecessary disk reads
  • LSM-Tree databases: Checks which SSTable might have the key
```

---

### Q95: Design a database schema for a Social Media News Feed.

```sql
-- Users
CREATE TABLE users (user_id BIGSERIAL PRIMARY KEY, username VARCHAR(50));

-- Posts
CREATE TABLE posts (
    post_id     BIGSERIAL PRIMARY KEY,
    author_id   BIGINT REFERENCES users(user_id),
    content     TEXT,
    media_url   TEXT[],
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Follow relationships
CREATE TABLE follows (
    follower_id BIGINT REFERENCES users(user_id),
    followee_id BIGINT REFERENCES users(user_id),
    followed_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id)
);

-- Feed: Fan-out on write (pre-computed feed per user)
CREATE TABLE user_feed (
    user_id     BIGINT,
    post_id     BIGINT,
    author_id   BIGINT,
    created_at  TIMESTAMPTZ,
    PRIMARY KEY (user_id, created_at, post_id)
) PARTITION BY RANGE (created_at);

-- Indexes
CREATE INDEX idx_posts_author ON posts(author_id, created_at DESC);
CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_feed_user ON user_feed(user_id, created_at DESC);
```

**Fan-out strategies:**
- **Fan-out on Write**: Pre-compute feed when post is created (fast reads, slow writes)
- **Fan-out on Read**: Build feed at read time from followed users (slow reads, fast writes)
- **Hybrid**: Fan-out on write for normal users, fan-out on read for celebrities (Instagram approach)

---

### Q96: What is Connection Multiplexing in database proxies?

```
WITHOUT MULTIPLEXING:                WITH MULTIPLEXING (PgBouncer):
App: 500 connections                 App: 500 connections
       │                                    │
       ↓                                    ↓
DB: 500 connections 💀               PgBouncer: 500 → 20 mapping
                                            │
                                            ↓
                                     DB: 20 connections ✅

Modes (PgBouncer):
  Session:     1 app conn → 1 DB conn (entire session) — safest
  Transaction: 1 app conn → 1 DB conn (per transaction) — most used
  Statement:   1 app conn → 1 DB conn (per query) — most efficient, most restrictions
```

---

### Q97: How do you troubleshoot a database that's suddenly slow?

```
EMERGENCY SLOWNESS CHECKLIST:

┌──────────────────────────────────────────────────────────────┐
│ 1. CHECK ACTIVE QUERIES                                     │
│    PostgreSQL: SELECT * FROM pg_stat_activity;              │
│    MySQL:      SHOW PROCESSLIST;                            │
│    SQL Server: sp_who2 / sys.dm_exec_requests               │
│    → Look for: Long-running queries, blocked queries        │
│                                                              │
│ 2. CHECK LOCKS                                               │
│    → Is something blocking everything?                      │
│    → Kill the blocking query if non-critical                │
│                                                              │
│ 3. CHECK RESOURCES                                           │
│    → CPU, Memory, Disk I/O, Network                         │
│    → Disk full? Out of memory?                              │
│                                                              │
│ 4. CHECK SLOW QUERY LOG                                      │
│    → What query suddenly got slow?                          │
│    → Did data volume increase? Statistics stale?            │
│                                                              │
│ 5. CHECK RECENT CHANGES                                      │
│    → New deployment? Schema change? Index dropped?          │
│    → New query pattern? Batch job running?                  │
│                                                              │
│ 6. CHECK CONNECTION COUNT                                    │
│    → Connection exhaustion? Pool misconfigured?             │
│                                                              │
│ 7. CHECK REPLICATION LAG                                     │
│    → Read replicas falling behind?                          │
└──────────────────────────────────────────────────────────────┘
```

---

### Q98: What is the Outbox Pattern for reliable messaging?

```
PROBLEM: How to update DB AND send a message atomically?
  
  BEGIN;
  INSERT INTO orders (...);     -- ✅ Succeeds
  COMMIT;
  send_to_kafka('order_created'); -- 💀 Fails → inconsistent!

OUTBOX PATTERN:
  BEGIN;
  INSERT INTO orders (...);                    -- Business data
  INSERT INTO outbox (event_type, payload);    -- Event in same TX!
  COMMIT;  -- Both succeed or both fail (ACID!) ✅
  
  -- Separate process (CDC/Debezium) reads outbox → sends to Kafka
  -- OR: Poll outbox table periodically
```

---

### Q99: What is a Distributed Transaction? What is Two-Phase Commit (2PC)?

```
TWO-PHASE COMMIT (2PC):

  Coordinator               Participant A        Participant B
       │                         │                     │
       │──── PREPARE ───────────→│                     │
       │──── PREPARE ────────────│────────────────────→│
       │                         │                     │
       │←─── VOTE YES ──────────│                     │
       │←─── VOTE YES ──────────│─────────────────────│
       │                         │                     │
       │  (All voted YES?)       │                     │
       │                         │                     │
       │──── COMMIT ────────────→│                     │
       │──── COMMIT ─────────────│────────────────────→│
       │                         │                     │
       │←─── ACK ────────────────│                     │
       │←─── ACK ────────────────│─────────────────────│
       
  If ANY participant votes NO → ABORT ALL

  Problems:
  • Coordinator crashes → all participants BLOCKED (holding locks)
  • Slow — requires 2 round trips
  • Alternative: Saga pattern (eventual consistency)
```

---

### Q100: If you could only give ONE piece of database advice, what would it be?

> **"Measure before you optimize. The bottleneck is never where you think it is."**

```
THE GOLDEN RULES OF DATABASES:

1. 📊 ALWAYS use EXPLAIN before optimizing
2. 🔑 Index your WHERE, JOIN, and ORDER BY columns
3. 📝 Keep transactions SHORT
4. 🚫 Never SELECT * in production
5. 💾 Store dates in UTC
6. 🔒 Parameterize queries (prevent SQL injection)
7. 📈 Monitor slow query logs
8. 🧹 Update statistics regularly
9. 🔄 Test with production-like data volumes
10. 🤔 Choose the RIGHT database for the job

REMEMBER:
  - PostgreSQL: When in doubt, start here
  - MySQL: Web apps with simple queries
  - Oracle/SQL Server: Enterprise with budget
  - MongoDB: Flexible schema, rapid development
  - Redis: Caching and real-time
  - Cassandra: Massive write throughput
  - Elasticsearch: Full-text search
```

---

## 🔑 Key Takeaways

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SQL INTERVIEW SURVIVAL KIT                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Master Window Functions — they're in 70% of SQL interviews     │
│  2. Know JOINs cold — including edge cases with NULLs              │
│  3. Understand execution plans — at least at a high level          │
│  4. Practice "Nth highest salary" variations until automatic       │
│  5. Know GROUP BY vs PARTITION BY — #1 confusion point             │
│  6. SARGability — show you care about performance                  │
│  7. Know ACID + Isolation Levels — conceptual depth                │
│  8. Practice the Gaps & Islands pattern — it's the "FizzBuzz"      │
│     of SQL interviews                                              │
│  9. Know the differences between databases (PG vs MySQL vs Oracle) │
│  10. Always think about indexes when writing any query             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

| Next Chapter | Link |
|-------------|------|
| Database Theory Interview Questions | [Chapter 8.2 — DB Theory Questions](./02-DB-Theory-Questions.md) |
| MongoDB Interview Questions | [Chapter 8.3 — MongoDB Interview](./03-MongoDB-Interview.md) |
| SQL Practice Problems | [Chapter 8.4 — 50 Must-Solve Problems](./04-SQL-Practice-Problems.md) |

---

> **"The best way to prepare for a SQL interview is to write SQL every day. Not memorize answers."** 🎯
