# 2.8 — CTEs & Recursive Queries 🟡⭐

> **"CTEs are to SQL what functions are to programming — they make your code readable, reusable, and elegant."**
> And recursive CTEs? They let SQL do things most people think require application code.

---

## 🎯 What You'll Master

```
✅ What CTEs are and why they exist
✅ Single and multiple CTEs (chaining WITH clauses)
✅ CTEs vs Subqueries vs Temp Tables — when to use which
✅ Recursive CTEs — the concept that changes everything
✅ Hierarchy traversal (org charts, folder trees, categories)
✅ Bill of Materials (BOM) — manufacturing classic
✅ Generating series/sequences with recursion
✅ Graph walking and path finding
✅ Performance implications and gotchas
```

---

## 🧠 Part 1: Common Table Expressions (CTEs)

### The Problem: Subquery Hell

Ever written a query like this?

```sql
-- 😵 Three levels of nesting — good luck debugging this
SELECT department, avg_salary
FROM (
    SELECT department, AVG(salary) AS avg_salary
    FROM (
        SELECT * FROM employees 
        WHERE hire_date > '2020-01-01'
    ) AS recent_hires
    GROUP BY department
) AS dept_averages
WHERE avg_salary > 80000;
```

Your brain reads **inside-out**. That's unnatural. Enter CTEs.

### The CTE Solution: Read Top-to-Bottom

```sql
-- 😎 Same query, but readable like a story
WITH recent_hires AS (
    SELECT * FROM employees 
    WHERE hire_date > '2020-01-01'
),
dept_averages AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM recent_hires
    GROUP BY department
)
SELECT department, avg_salary
FROM dept_averages
WHERE avg_salary > 80000;
```

> 💡 **A CTE is a temporary named result set** that exists only for the duration of a single query. Think of it as a "query-scoped view."

---

### 📐 CTE Syntax

```sql
WITH cte_name [(column_list)] AS (
    -- The CTE query (called the "anchor")
    SELECT ...
)
-- The main query that uses the CTE
SELECT ... FROM cte_name ...;
```

```
┌──────────────────────────────────────────────────┐
│  WITH   keyword_starts_the_CTE                   │
│  ├── cte_name  AS (                              │
│  │       SELECT ... FROM ...                     │
│  │   )                                           │
│  │                                               │
│  └── Main Query                                  │
│         SELECT ... FROM cte_name                 │
└──────────────────────────────────────────────────┘
```

---

### 🔗 Chaining Multiple CTEs

CTEs can reference each other (earlier ones only — no forward references):

```sql
WITH 
-- Step 1: Get all orders from this year
current_year_orders AS (
    SELECT * FROM orders 
    WHERE EXTRACT(YEAR FROM order_date) = 2024
),

-- Step 2: Aggregate by customer (uses Step 1)
customer_totals AS (
    SELECT 
        customer_id, 
        COUNT(*) AS order_count,
        SUM(amount) AS total_spent
    FROM current_year_orders
    GROUP BY customer_id
),

-- Step 3: Classify customers (uses Step 2)
customer_tiers AS (
    SELECT *,
        CASE 
            WHEN total_spent >= 10000 THEN 'Platinum'
            WHEN total_spent >= 5000  THEN 'Gold'
            WHEN total_spent >= 1000  THEN 'Silver'
            ELSE 'Bronze'
        END AS tier
    FROM customer_totals
)

-- Final: Join with customer names (uses Step 3)
SELECT c.name, c.email, ct.order_count, ct.total_spent, ct.tier
FROM customer_tiers ct
JOIN customers c ON c.id = ct.customer_id
ORDER BY ct.total_spent DESC;
```

```
Flow visualization:

current_year_orders ──→ customer_totals ──→ customer_tiers ──→ Final SELECT
       (filter)            (aggregate)        (classify)        (present)
```

> 💡 Each CTE is like a **pipeline stage**. Data flows through transformations step by step. Much easier to debug — you can SELECT from any intermediate CTE to inspect it.

---

### 🔄 CTE with INSERT, UPDATE, DELETE

CTEs aren't just for SELECT. They work with DML too:

```sql
-- Delete duplicate emails, keeping the earliest record
WITH duplicates AS (
    SELECT id, email,
        ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
    FROM users
)
DELETE FROM users 
WHERE id IN (SELECT id FROM duplicates WHERE rn > 1);
```

```sql
-- Update based on complex calculations
WITH revenue_stats AS (
    SELECT 
        salesperson_id,
        SUM(amount) AS total_revenue,
        RANK() OVER (ORDER BY SUM(amount) DESC) AS rnk
    FROM sales
    GROUP BY salesperson_id
)
UPDATE employees 
SET bonus = CASE 
    WHEN rs.rnk = 1 THEN 50000
    WHEN rs.rnk <= 5 THEN 25000
    ELSE 5000
END
FROM revenue_stats rs
WHERE employees.id = rs.salesperson_id;
```

---

### ⚔️ CTE vs Subquery vs Temp Table vs View

| Feature | CTE | Subquery | Temp Table | View |
|---------|-----|----------|------------|------|
| **Scope** | Single statement | Single statement | Session/block | Permanent |
| **Reusable in query?** | ✅ Multiple references | ❌ Must repeat | ✅ | ✅ |
| **Readability** | ⭐⭐⭐ Excellent | ⭐ Poor (nested) | ⭐⭐ Good | ⭐⭐⭐ |
| **Can be recursive?** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Indexable?** | ❌ No | ❌ No | ✅ Yes | Depends |
| **Materialized?** | Usually no* | No | Yes (on disk) | No |
| **Performance** | Inline expansion | Inline expansion | Disk I/O | Inline expansion |

> *PostgreSQL can materialize CTEs with `MATERIALIZED` hint. SQL Server always inlines.

**When to use what:**

```
Need it once, simple?           → Subquery
Need it referenced multiple times? → CTE
Need recursion?                 → CTE (only option)
Need indexes or large dataset?  → Temp Table
Need it across multiple queries? → View or Temp Table
```

---

## 🔁 Part 2: Recursive CTEs — Where the Magic Happens

### What is Recursion in SQL?

A recursive CTE references **itself**. It has two parts:

```sql
WITH RECURSIVE cte_name AS (
    -- 1. ANCHOR MEMBER: The starting point (non-recursive)
    SELECT ...
    
    UNION ALL
    
    -- 2. RECURSIVE MEMBER: References the CTE itself
    SELECT ... FROM cte_name WHERE ...
)
SELECT * FROM cte_name;
```

```
🔄 HOW IT WORKS:

Step 1: Execute ANCHOR → produces initial rows
Step 2: Execute RECURSIVE using Step 1's results → produces new rows
Step 3: Execute RECURSIVE using Step 2's results → produces new rows
Step 4: Execute RECURSIVE using Step 3's results → produces new rows
...
Step N: RECURSIVE produces no new rows → STOP!

Final result = UNION ALL of all steps
```

> ⚠️ **PostgreSQL, MySQL, SQLite** use `WITH RECURSIVE`. **SQL Server and Oracle** just use `WITH` (they auto-detect recursion).

---

### 🔢 Example 1: Generate a Number Sequence

```sql
-- Generate numbers 1 to 10
WITH RECURSIVE numbers AS (
    -- Anchor: start at 1
    SELECT 1 AS n
    
    UNION ALL
    
    -- Recursive: add 1 each time, stop at 10
    SELECT n + 1 
    FROM numbers 
    WHERE n < 10
)
SELECT n FROM numbers;
```

| n |
|---|
| 1 |
| 2 |
| 3 |
| ... |
| 10 |

```
Execution trace:

Iteration 0 (Anchor):  n = 1
Iteration 1:           n = 1 + 1 = 2     (1 < 10 ✅)
Iteration 2:           n = 2 + 1 = 3     (2 < 10 ✅)
...
Iteration 9:           n = 9 + 1 = 10    (9 < 10 ✅)
Iteration 10:          10 < 10 ❌ → STOP!
```

---

### 📅 Example 2: Generate a Date Series

```sql
-- Generate all dates in January 2024
WITH RECURSIVE dates AS (
    SELECT DATE '2024-01-01' AS dt
    UNION ALL
    SELECT dt + INTERVAL '1 day'
    FROM dates
    WHERE dt < DATE '2024-01-31'
)
SELECT dt, TO_CHAR(dt, 'Day') AS day_name 
FROM dates;
```

> 💡 **Real-world use:** Generate a calendar table, fill in missing dates in a time series, create reporting date ranges.

---

### 🏢 Example 3: Organizational Hierarchy (The Classic!)

```sql
-- The org chart table
CREATE TABLE org_chart (
    emp_id    INT PRIMARY KEY,
    name      VARCHAR(50),
    title     VARCHAR(100),
    manager_id INT REFERENCES org_chart(emp_id)
);

INSERT INTO org_chart VALUES
(1, 'Sarah',  'CEO',              NULL),
(2, 'Mike',   'VP Engineering',   1),
(3, 'Lisa',   'VP Sales',         1),
(4, 'Tom',    'Senior Dev',       2),
(5, 'Anna',   'Junior Dev',       4),
(6, 'Jake',   'DevOps Lead',      2),
(7, 'Priya',  'Sales Manager',    3),
(8, 'Carlos', 'Sales Rep',        7);
```

```
Organization Tree:

Sarah (CEO)
├── Mike (VP Engineering)
│   ├── Tom (Senior Dev)
│   │   └── Anna (Junior Dev)
│   └── Jake (DevOps Lead)
└── Lisa (VP Sales)
    └── Priya (Sales Manager)
        └── Carlos (Sales Rep)
```

#### Find All Subordinates of a Manager

```sql
WITH RECURSIVE subordinates AS (
    -- Anchor: Start with the manager
    SELECT emp_id, name, title, manager_id, 0 AS depth
    FROM org_chart
    WHERE name = 'Sarah'  -- Starting point
    
    UNION ALL
    
    -- Recursive: Find direct reports of current level
    SELECT e.emp_id, e.name, e.title, e.manager_id, s.depth + 1
    FROM org_chart e
    INNER JOIN subordinates s ON e.manager_id = s.emp_id
)
SELECT 
    REPEAT('  ', depth) || name AS org_tree,
    title,
    depth
FROM subordinates
ORDER BY depth, name;
```

| org_tree | title | depth |
|----------|-------|-------|
| Sarah | CEO | 0 |
|   Lisa | VP Sales | 1 |
|   Mike | VP Engineering | 1 |
|     Jake | DevOps Lead | 2 |
|     Priya | Sales Manager | 2 |
|     Tom | Senior Dev | 2 |
|       Anna | Junior Dev | 3 |
|       Carlos | Sales Rep | 3 |

```
🔄 EXECUTION TRACE:

Iteration 0 (Anchor): Sarah
Iteration 1: Who reports to Sarah? → Mike, Lisa
Iteration 2: Who reports to Mike or Lisa? → Tom, Jake, Priya
Iteration 3: Who reports to Tom, Jake, or Priya? → Anna, Carlos
Iteration 4: Who reports to Anna or Carlos? → Nobody → STOP!
```

---

#### Build the Full Path from Root to Each Employee

```sql
WITH RECURSIVE org_path AS (
    SELECT 
        emp_id, name, title, manager_id,
        name::TEXT AS full_path,
        0 AS depth
    FROM org_chart
    WHERE manager_id IS NULL  -- Start from CEO
    
    UNION ALL
    
    SELECT 
        e.emp_id, e.name, e.title, e.manager_id,
        op.full_path || ' → ' || e.name,
        op.depth + 1
    FROM org_chart e
    INNER JOIN org_path op ON e.manager_id = op.emp_id
)
SELECT full_path, title 
FROM org_path
ORDER BY full_path;
```

| full_path | title |
|-----------|-------|
| Sarah | CEO |
| Sarah → Lisa | VP Sales |
| Sarah → Lisa → Priya | Sales Manager |
| Sarah → Lisa → Priya → Carlos | Sales Rep |
| Sarah → Mike | VP Engineering |
| Sarah → Mike → Jake | DevOps Lead |
| Sarah → Mike → Tom | Senior Dev |
| Sarah → Mike → Tom → Anna | Junior Dev |

---

### 🏭 Example 4: Bill of Materials (BOM)

A classic manufacturing problem: "What parts make up this product, and what are their total costs?"

```sql
CREATE TABLE parts (
    part_id     INT PRIMARY KEY,
    part_name   VARCHAR(50),
    unit_cost   DECIMAL(10,2)
);

CREATE TABLE bom (
    parent_id   INT REFERENCES parts(part_id),
    child_id    INT REFERENCES parts(part_id),
    quantity    INT,
    PRIMARY KEY (parent_id, child_id)
);

-- A bicycle and its components
INSERT INTO parts VALUES
(1, 'Bicycle',      0),
(2, 'Frame',        150),
(3, 'Wheel',        45),
(4, 'Tire',         20),
(5, 'Rim',          15),
(6, 'Spoke',        0.50),
(7, 'Handlebar',    35);

INSERT INTO bom VALUES
(1, 2, 1),   -- Bicycle needs 1 Frame
(1, 3, 2),   -- Bicycle needs 2 Wheels
(1, 7, 1),   -- Bicycle needs 1 Handlebar
(3, 4, 1),   -- Wheel needs 1 Tire
(3, 5, 1),   -- Wheel needs 1 Rim
(3, 6, 36);  -- Wheel needs 36 Spokes
```

```
BICYCLE
├── Frame (1x) ............ $150.00
├── Wheel (2x)
│   ├── Tire (1x) ......... $20.00
│   ├── Rim (1x) .......... $15.00
│   └── Spoke (36x) ....... $0.50 each
└── Handlebar (1x) ........ $35.00
```

```sql
-- Explode the full BOM with costs
WITH RECURSIVE bill AS (
    -- Anchor: Top-level product
    SELECT 
        p.part_id, p.part_name, p.unit_cost,
        1 AS quantity,
        0 AS depth,
        p.part_name::TEXT AS path
    FROM parts p
    WHERE p.part_name = 'Bicycle'
    
    UNION ALL
    
    -- Recursive: Find sub-components
    SELECT 
        p.part_id, p.part_name, p.unit_cost,
        b.quantity * bill.quantity AS quantity,
        bill.depth + 1,
        bill.path || ' > ' || p.part_name
    FROM bom b
    JOIN parts p ON p.part_id = b.child_id
    JOIN bill ON bill.part_id = b.parent_id
)
SELECT 
    REPEAT('  ', depth) || part_name AS component,
    quantity,
    unit_cost,
    quantity * unit_cost AS total_cost
FROM bill
ORDER BY path;
```

| component | quantity | unit_cost | total_cost |
|-----------|----------|-----------|------------|
| Bicycle | 1 | 0.00 | 0.00 |
|   Frame | 1 | 150.00 | 150.00 |
|   Handlebar | 1 | 35.00 | 35.00 |
|   Wheel | 2 | 45.00 | 90.00 |
|     Rim | 2 | 15.00 | 30.00 |
|     Spoke | 72 | 0.50 | 36.00 |
|     Tire | 2 | 20.00 | 40.00 |

**Total bicycle cost = $381.00** (sum of leaf-node total_cost + wheel unit cost if applicable)

---

### 📂 Example 5: File System / Category Tree

```sql
CREATE TABLE categories (
    id        INT PRIMARY KEY,
    name      VARCHAR(100),
    parent_id INT REFERENCES categories(id)
);

INSERT INTO categories VALUES
(1, 'Electronics', NULL),
(2, 'Computers', 1),
(3, 'Laptops', 2),
(4, 'Gaming Laptops', 3),
(5, 'Business Laptops', 3),
(6, 'Desktops', 2),
(7, 'Phones', 1),
(8, 'Smartphones', 7),
(9, 'Feature Phones', 7);

-- Get full category breadcrumb for each category
WITH RECURSIVE breadcrumb AS (
    SELECT id, name, parent_id, 
           name::TEXT AS full_path,
           0 AS depth
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT c.id, c.name, c.parent_id,
           b.full_path || ' > ' || c.name,
           b.depth + 1
    FROM categories c
    JOIN breadcrumb b ON c.parent_id = b.id
)
SELECT full_path FROM breadcrumb ORDER BY full_path;
```

| full_path |
|-----------|
| Electronics |
| Electronics > Computers |
| Electronics > Computers > Desktops |
| Electronics > Computers > Laptops |
| Electronics > Computers > Laptops > Business Laptops |
| Electronics > Computers > Laptops > Gaming Laptops |
| Electronics > Phones |
| Electronics > Phones > Feature Phones |
| Electronics > Phones > Smartphones |

---

### 🌐 Example 6: Graph Walking (Find All Connected Nodes)

```sql
-- Social network: find all people connected to Alice (friends of friends)
CREATE TABLE friendships (
    person1 VARCHAR(50),
    person2 VARCHAR(50)
);

INSERT INTO friendships VALUES
('Alice', 'Bob'), ('Bob', 'Carol'), ('Carol', 'Dave'),
('Dave', 'Eve'), ('Alice', 'Frank'), ('Frank', 'Grace');

WITH RECURSIVE network AS (
    -- Start with Alice's direct friends
    SELECT person2 AS person, 1 AS degree, 
           ARRAY[person1, person2] AS path  -- Track visited to prevent cycles
    FROM friendships
    WHERE person1 = 'Alice'
    
    UNION ALL
    
    SELECT f.person2, n.degree + 1,
           n.path || f.person2
    FROM friendships f
    JOIN network n ON f.person1 = n.person
    WHERE f.person2 != ALL(n.path)  -- Prevent cycles!
      AND n.degree < 4              -- Max depth
)
SELECT person, degree AS degrees_of_separation
FROM network
ORDER BY degree, person;
```

---

## ⚠️ Recursive CTE Safety — Preventing Infinite Loops

### 🛡️ Defense 1: WHERE Clause with Termination Condition

```sql
-- Always include a termination condition
WHERE n < 100          -- For number generation
WHERE depth < 10       -- For tree traversal
WHERE NOT visited      -- For graph walking
```

### 🛡️ Defense 2: MAXRECURSION (SQL Server Only)

```sql
-- SQL Server: limit recursion depth
WITH hierarchy AS (...)
SELECT * FROM hierarchy
OPTION (MAXRECURSION 100);  -- Default is 100, max is 32767, 0 = unlimited
```

### 🛡️ Defense 3: Cycle Detection (PostgreSQL 14+)

```sql
-- PostgreSQL 14+: Built-in cycle detection
WITH RECURSIVE tree AS (
    SELECT id, parent_id, name
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.parent_id, c.name
    FROM categories c JOIN tree t ON c.parent_id = t.id
)
CYCLE id SET is_cycle USING path;  -- Automatic cycle detection!
```

### 🛡️ Defense 4: Manual Cycle Detection (Portable)

```sql
WITH RECURSIVE tree AS (
    SELECT id, parent_id, ARRAY[id] AS visited
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.parent_id, t.visited || c.id
    FROM categories c 
    JOIN tree t ON c.parent_id = t.id
    WHERE c.id != ALL(t.visited)  -- Skip if already visited
)
SELECT * FROM tree;
```

---

## ⚡ Performance Considerations

### When CTEs Help Performance

```
✅ Replacing deeply nested subqueries (optimizer can work better)
✅ When the same subquery is referenced multiple times
✅ Recursive problems (no alternative)
✅ Code clarity leads to better optimization decisions
```

### When CTEs Hurt Performance

```
❌ PostgreSQL < 12: CTEs were optimization barriers (materialized by default)
❌ Very large intermediate results (no indexes on CTE output)
❌ Using CTE where a simple JOIN would suffice
```

### PostgreSQL Materialization Control (v12+)

```sql
-- Force materialization (compute once, store in memory)
WITH cached_data AS MATERIALIZED (
    SELECT * FROM expensive_view
)
SELECT * FROM cached_data cd1
JOIN cached_data cd2 ON cd1.related_id = cd2.id;

-- Force inlining (merge into main query for optimizer)
WITH inline_data AS NOT MATERIALIZED (
    SELECT * FROM simple_table WHERE status = 'active'
)
SELECT * FROM inline_data WHERE amount > 100;
```

---

## 🗄️ Database-Specific Notes

| Feature | PostgreSQL | MySQL 8+ | SQL Server | Oracle | SQLite |
|---------|-----------|----------|------------|--------|--------|
| Basic CTE | ✅ | ✅ | ✅ | ✅ | ✅ |
| Multiple CTEs | ✅ | ✅ | ✅ | ✅ | ✅ |
| Recursive CTE | ✅ `WITH RECURSIVE` | ✅ `WITH RECURSIVE` | ✅ `WITH` | ✅ `WITH` | ✅ `WITH RECURSIVE` |
| CTE in DML | ✅ | ✅ (limited) | ✅ | ✅ | ✅ |
| MAXRECURSION | ❌ | cte_max_recursion_depth | ✅ OPTION() | ❌ | ❌ |
| Cycle detection | ✅ (v14+ CYCLE) | ❌ (manual) | ❌ (manual) | ✅ (CYCLE) | ❌ (manual) |
| Materialization | ✅ (v12+ hints) | Optimizer decides | Always inlined | Optimizer decides | Optimizer decides |
| Default recursion limit | None (caution!) | 1000 | 100 | None | 1000000000 |

---

## 🧪 Practice Challenges

### Challenge 1: Fibonacci Sequence
> Generate the first 20 Fibonacci numbers using a recursive CTE.

<details>
<summary>💡 Solution</summary>

```sql
WITH RECURSIVE fib AS (
    SELECT 1 AS n, 0::BIGINT AS fib_n, 1::BIGINT AS fib_next
    UNION ALL
    SELECT n + 1, fib_next, fib_n + fib_next
    FROM fib
    WHERE n < 20
)
SELECT n, fib_n AS fibonacci_number FROM fib;
```
</details>

### Challenge 2: Employee Chain of Command
> For a given employee, show their entire management chain up to the CEO.

<details>
<summary>💡 Solution</summary>

```sql
WITH RECURSIVE chain AS (
    -- Start from the employee
    SELECT emp_id, name, title, manager_id, 0 AS level
    FROM org_chart
    WHERE name = 'Anna'
    
    UNION ALL
    
    -- Walk UP to their manager
    SELECT o.emp_id, o.name, o.title, o.manager_id, c.level + 1
    FROM org_chart o
    JOIN chain c ON o.emp_id = c.manager_id
)
SELECT name, title, level
FROM chain
ORDER BY level;
```

| name | title | level |
|------|-------|-------|
| Anna | Junior Dev | 0 |
| Tom | Senior Dev | 1 |
| Mike | VP Engineering | 2 |
| Sarah | CEO | 3 |
</details>

### Challenge 3: Expand Comma-Separated Values
> Split the string 'apple,banana,cherry,date' into separate rows using a recursive CTE.

<details>
<summary>💡 Solution</summary>

```sql
WITH RECURSIVE split AS (
    SELECT 
        'apple,banana,cherry,date' AS remaining,
        '' AS item,
        0 AS pos
    UNION ALL
    SELECT
        CASE 
            WHEN POSITION(',' IN remaining) > 0 
            THEN SUBSTRING(remaining FROM POSITION(',' IN remaining) + 1)
            ELSE ''
        END,
        CASE 
            WHEN POSITION(',' IN remaining) > 0 
            THEN SUBSTRING(remaining FROM 1 FOR POSITION(',' IN remaining) - 1)
            ELSE remaining
        END,
        pos + 1
    FROM split
    WHERE remaining != ''
)
SELECT item FROM split WHERE pos > 0;
```
</details>

### Challenge 4: Calculate Depth of Each Category
> For the categories table, calculate how deep each category is and show its full breadcrumb path.

<details>
<summary>💡 Solution</summary>

```sql
WITH RECURSIVE cat_tree AS (
    SELECT id, name, parent_id, 0 AS depth, name::TEXT AS breadcrumb
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id, ct.depth + 1, ct.breadcrumb || ' > ' || c.name
    FROM categories c
    JOIN cat_tree ct ON c.parent_id = ct.id
)
SELECT name, depth, breadcrumb FROM cat_tree ORDER BY breadcrumb;
```
</details>

---

## 🎯 Key Takeaways

```
┌──────────────────────────────────────────────────────────────────┐
│  CTE CHEAT SHEET                                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  WITH name AS (query)                                            │
│  SELECT ... FROM name;                                           │
│                                                                  │
│  RECURSIVE = Anchor + UNION ALL + Recursive Member               │
│                                                                  │
│  USE CTEs FOR:                                                   │
│  • Readability (top-to-bottom flow)                              │
│  • Reuse (reference same CTE multiple times)                     │
│  • Recursion (hierarchies, trees, graphs, sequences)             │
│  • Complex DML (delete dupes, calculated updates)                │
│                                                                  │
│  ⚠️ ALWAYS add termination condition to recursive CTEs           │
│  ⚠️ Watch for cycles in graph/tree data                          │
│  ⚠️ PostgreSQL < 12: CTEs are optimization barriers              │
│  ⚠️ SQL Server default max recursion = 100                       │
│                                                                  │
│  COMMON PATTERNS:                                                │
│  • Org chart traversal (manager → reports)                       │
│  • Bill of Materials (parent → children)                         │
│  • Category breadcrumbs (root → leaf)                            │
│  • Date/number sequence generation                               │
│  • Graph walking (friend-of-friend)                              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

> 🚀 **Next Up:** [2.9 — Views, Materialized Views & Temp Tables](./09-Views.md) — Create virtual tables, cache expensive queries, and manage temporary data like a pro.

---

*Recursive CTEs are the closest SQL gets to "thinking in loops." Once you internalize the Anchor + Recursive pattern, you'll see hierarchies everywhere — and you'll know exactly how to query them.* 🔁
