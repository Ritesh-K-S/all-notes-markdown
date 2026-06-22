# 2.11 — Query Optimization & Execution Plans 🔴⭐🔥

> **"A fast query isn't written — it's crafted. The difference between 30 seconds and 30 milliseconds is understanding HOW the database thinks."**
> This chapter transforms you from someone who writes SQL to someone who writes FAST SQL.

---

## 🎯 What You'll Master

```
✅ How the query optimizer works (the database's "brain")
✅ Reading execution plans — the X-ray of your queries
✅ EXPLAIN, EXPLAIN ANALYZE, and what every operator means
✅ SARGable queries — the #1 rule of fast SQL
✅ Index usage and why the optimizer ignores your indexes
✅ Statistics and cardinality estimation
✅ Common performance killers and how to fix them
✅ Query hints — overriding the optimizer (use with caution!)
✅ Database-specific tuning tools
```

---

## 🧠 Part 1: How the Query Optimizer Works

### The Journey of a SQL Query

When you execute a query, the database doesn't just "run it." It goes through a pipeline:

```
Your SQL Query
      │
      ▼
┌──────────────┐
│   PARSER     │  Syntax check → Parse tree
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  ANALYZER    │  Semantic check (tables/columns exist?)
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│              QUERY OPTIMIZER                          │
│                                                      │
│  "Given this query, what's the FASTEST way to        │
│   get the result?"                                   │
│                                                      │
│  Considers:                                          │
│  • Which indexes to use (or not use)                 │
│  • Join order (A→B→C or C→B→A?)                      │
│  • Join algorithm (Nested Loop, Hash, Merge?)        │
│  • Scan method (Sequential, Index, Bitmap?)          │
│  • Parallelism (use multiple CPU cores?)             │
│                                                      │
│  Uses: Table statistics, index info, data volume     │
│  Output: EXECUTION PLAN (the "recipe")               │
└──────────────────┬───────────────────────────────────┘
                   │
                   ▼
          ┌──────────────┐
          │   EXECUTOR   │  Runs the plan → Returns results
          └──────────────┘
```

The optimizer generates **many possible plans** and picks the one with the **lowest estimated cost**.

---

## 📊 Part 2: Reading Execution Plans

### How to Get an Execution Plan

```sql
-- PostgreSQL
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;           -- Plan only (estimated)
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;   -- Plan + actual execution!
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;             -- Full details

-- SQL Server
SET SHOWPLAN_TEXT ON;       -- Text plan
SET STATISTICS PROFILE ON; -- Actual execution stats
-- Or in SSMS: Ctrl+L (estimated plan), Ctrl+M (actual plan)

-- MySQL
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;  -- MySQL 8.0.18+
EXPLAIN FORMAT=JSON SELECT ...;                                -- Detailed JSON plan
EXPLAIN FORMAT=TREE SELECT ...;                                -- Tree format

-- Oracle
EXPLAIN PLAN FOR SELECT * FROM orders WHERE customer_id = 42;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- Or use: SET AUTOTRACE ON
```

---

### Anatomy of a PostgreSQL Execution Plan

```sql
EXPLAIN ANALYZE 
SELECT o.order_id, c.name, o.amount
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.order_date > '2024-01-01'
  AND o.amount > 100;
```

```
Hash Join  (cost=35.50..127.42 rows=89 width=52) (actual time=0.845..2.134 rows=94 loops=1)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (cost=0.00..89.00 rows=89 width=20) (actual time=0.012..1.023 rows=94 loops=1)
        Filter: ((order_date > '2024-01-01') AND (amount > 100))
        Rows Removed by Filter: 9906
  ->  Hash  (cost=22.00..22.00 rows=1000 width=36) (actual time=0.754..0.755 rows=1000 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 72kB
        ->  Seq Scan on customers c  (cost=0.00..22.00 rows=1000 width=36) (actual time=0.004..0.345 rows=1000 loops=1)
Planning Time: 0.215 ms
Execution Time: 2.267 ms
```

Let's decode this line by line:

```
Hash Join  (cost=35.50..127.42 rows=89 width=52) (actual time=0.845..2.134 rows=94 loops=1)
│          │                    │        │          │                     │         │
│          │                    │        │          │                     │         └── loops: iterations
│          │                    │        │          │                     └── actual rows returned
│          │                    │        │          └── actual time (startup..total) in ms
│          │                    │        └── width: avg bytes per row
│          │                    └── estimated rows (optimizer's guess)
│          └── cost: startup..total (arbitrary units — lower = better)
└── Operation type
```

### Key Metrics to Watch

| Metric | What to check |
|--------|---------------|
| **Estimated vs Actual rows** | If wildly different → bad statistics! Run ANALYZE |
| **Seq Scan on large table** | Missing index? Or query not SARGable? |
| **High cost** | Which step has the highest cost? That's your bottleneck |
| **Rows Removed by Filter** | Large number? Index might help pre-filter |
| **Loops** | High number in Nested Loop? Consider Hash/Merge Join |
| **Sort** | External sort (disk) vs in-memory sort |
| **Buffers** | shared hit (cache) vs shared read (disk) |

---

### Common Execution Plan Operators

```
📋 SCAN OPERATORS (How data is read)

┌────────────────────┬──────────────────────────────────────────────────┐
│ Seq Scan           │ Full table scan — reads EVERY row                │
│                    │ 🔴 Bad for large tables with selective filters    │
│                    │ ✅ OK for small tables or when most rows needed   │
├────────────────────┼──────────────────────────────────────────────────┤
│ Index Scan         │ Uses index to find rows, then fetches from table │
│                    │ ✅ Great for selective queries (few rows)          │
├────────────────────┼──────────────────────────────────────────────────┤
│ Index Only Scan    │ All needed columns ARE in the index              │
│                    │ ⭐ Best possible — never touches the table!       │
├────────────────────┼──────────────────────────────────────────────────┤
│ Bitmap Index Scan  │ Reads index → builds bitmap → scans table       │
│                    │ ✅ Good for medium selectivity or OR conditions   │
├────────────────────┼──────────────────────────────────────────────────┤
│ Clustered Index    │ SQL Server: Data IS the index (sorted on disk)   │
│ Scan               │ Like a Seq Scan but on the clustered index       │
├────────────────────┼──────────────────────────────────────────────────┤
│ Key Lookup /       │ SQL Server/PG: Found row in non-clustered index, │
│ Table Fetch        │ but needs to go back to table for extra columns  │
│                    │ 🔴 Can be expensive — consider covering index     │
└────────────────────┴──────────────────────────────────────────────────┘
```

```
📋 JOIN OPERATORS (How tables are combined)

┌────────────────────┬──────────────────────────────────────────────────┐
│ Nested Loop Join   │ For each row in A, scan matching rows in B       │
│                    │ ✅ Best when: inner table has index, few rows     │
│                    │ 🔴 Worst when: both tables large, no index        │
│                    │ Cost: O(N × M) worst case                        │
├────────────────────┼──────────────────────────────────────────────────┤
│ Hash Join          │ Build hash table from smaller table, probe with  │
│                    │ larger table                                     │
│                    │ ✅ Best for: equality joins on large tables        │
│                    │ 🔴 Needs memory for hash table                    │
│                    │ Cost: O(N + M)                                   │
├────────────────────┼──────────────────────────────────────────────────┤
│ Merge Join /       │ Both inputs sorted, merge like zipper            │
│ Sort Merge Join    │ ✅ Best for: pre-sorted data or range joins       │
│                    │ 🔴 Needs sorted input (costly if not indexed)     │
│                    │ Cost: O(N log N + M log M) with sort             │
└────────────────────┴──────────────────────────────────────────────────┘
```

```
📋 OTHER OPERATORS

┌────────────────────┬──────────────────────────────────────────────────┐
│ Sort               │ Sorts data (ORDER BY, Merge Join, DISTINCT)     │
│                    │ 🔴 Watch for "external sort" (spilling to disk)  │
├────────────────────┼──────────────────────────────────────────────────┤
│ Hash Aggregate     │ GROUP BY using hash table                        │
│ Sort + Group       │ GROUP BY using sort                              │
├────────────────────┼──────────────────────────────────────────────────┤
│ Materialize        │ Caches subquery result in memory                 │
├────────────────────┼──────────────────────────────────────────────────┤
│ Limit              │ Stops after N rows (LIMIT/TOP)                   │
├────────────────────┼──────────────────────────────────────────────────┤
│ Gather / Gather    │ Collects results from parallel workers           │
│ Merge              │                                                  │
└────────────────────┴──────────────────────────────────────────────────┘
```

---

## 🎯 Part 3: SARGable Queries — The #1 Optimization Rule

### What is SARGable?

**SARG** = **S**earch **ARG**ument. A SARGable query is one where the database **can use an index** to filter rows.

> 💡 **The Rule:** Never apply a function or operation to the **column** in a WHERE clause. Apply it to the **value** instead.

### ❌ Non-SARGable (Index CANNOT be used)

```sql
-- ❌ Function on column — index on order_date is USELESS
WHERE YEAR(order_date) = 2024

-- ❌ Math on column
WHERE salary * 1.1 > 100000

-- ❌ Function on column
WHERE UPPER(name) = 'ALICE'

-- ❌ Leading wildcard
WHERE name LIKE '%smith'

-- ❌ Implicit conversion (string column, int comparison)
WHERE phone_number = 12345

-- ❌ OR with different columns (can't use single index)
WHERE department = 'Sales' OR hire_date > '2024-01-01'

-- ❌ NOT IN / NOT EXISTS / != (usually can't use index efficiently)
WHERE status != 'inactive'

-- ❌ Column on both sides
WHERE column_a + column_b > 100
```

### ✅ SARGable Rewrites (Index CAN be used)

```sql
-- ✅ Range instead of function
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'

-- ✅ Math on value, not column
WHERE salary > 100000 / 1.1

-- ✅ Functional index (if supported) OR application-level normalization
CREATE INDEX idx_name_upper ON employees (UPPER(name));
WHERE UPPER(name) = 'ALICE'  -- Now SARGable with functional index!

-- ✅ Trailing wildcard only
WHERE name LIKE 'smith%'

-- ✅ Correct type
WHERE phone_number = '12345'

-- ✅ UNION ALL instead of OR
SELECT * FROM t WHERE department = 'Sales'
UNION ALL
SELECT * FROM t WHERE hire_date > '2024-01-01' AND department != 'Sales'

-- ✅ Positive condition
WHERE status IN ('active', 'pending', 'processing')
```

### SARGability Visual

```
NON-SARGable:                        SARGable:

Index: [.... sorted data ....]      Index: [.... sorted data ....]

WHERE f(column) = value              WHERE column = value
      │                                    │
      ▼                                    ▼
Must scan ENTIRE index               Can SEEK directly to value!
(function changes the value,         (value maps directly to
 can't binary search)                 index position)
```

---

## 📈 Part 4: Statistics & Cardinality Estimation

### What Are Statistics?

Statistics are **metadata about your data** that the optimizer uses to make decisions:

```
Table: orders (1,000,000 rows)

Column: status
┌──────────┬──────────┬───────────┐
│ Value    │ Count    │ Frequency │
├──────────┼──────────┼───────────┤
│ pending  │ 50,000   │ 5%        │
│ active   │ 200,000  │ 20%       │
│ shipped  │ 400,000  │ 40%       │
│ complete │ 350,000  │ 35%       │
└──────────┴──────────┴───────────┘

With this, optimizer knows:
WHERE status = 'pending'  → ~50,000 rows → USE INDEX ✅
WHERE status = 'shipped'  → ~400,000 rows → FULL SCAN cheaper ✅
```

### Cardinality Estimation

**Cardinality** = number of rows the optimizer estimates a step will produce.

```sql
-- If the optimizer estimates 10 rows but actual is 100,000:
-- It might choose Nested Loop Join (good for 10 rows)
-- Instead of Hash Join (good for 100,000 rows)
-- Result: TERRIBLE performance!
```

### Keeping Statistics Fresh

```sql
-- PostgreSQL: ANALYZE (updates statistics)
ANALYZE orders;            -- One table
ANALYZE;                   -- All tables
-- PostgreSQL auto-vacuums and auto-analyzes, but you can trigger manually

-- SQL Server: UPDATE STATISTICS
UPDATE STATISTICS orders;
-- SQL Server auto-updates, but after ~20% of rows change (can be stale!)

-- MySQL: ANALYZE TABLE
ANALYZE TABLE orders;

-- Oracle: DBMS_STATS
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA', 'ORDERS');
```

> 💡 **Pro tip:** After bulk loads, large deletes, or schema changes, ALWAYS update statistics. The #1 cause of bad plans is stale statistics.

---

## 🔧 Part 5: Common Performance Killers & Fixes

### Killer 1: SELECT * (The Lazy Query)

```sql
-- ❌ BAD: Fetches ALL columns, prevents Index Only Scan
SELECT * FROM orders WHERE customer_id = 42;

-- ✅ GOOD: Only needed columns, enables covering index
SELECT order_id, amount, order_date 
FROM orders WHERE customer_id = 42;

-- Even better: Create a covering index
CREATE INDEX idx_orders_cust_covering 
ON orders (customer_id) INCLUDE (order_id, amount, order_date);
-- Now it's an Index Only Scan — never touches the table!
```

### Killer 2: N+1 Query Problem

```sql
-- ❌ Application code does this (1 + N queries!):
-- Query 1: SELECT * FROM customers;
-- For each customer:
--   Query N: SELECT * FROM orders WHERE customer_id = ?;

-- ✅ ONE query with JOIN:
SELECT c.name, o.order_id, o.amount
FROM customers c
JOIN orders o ON o.customer_id = c.id;

-- ✅ Or batch with IN:
SELECT * FROM orders WHERE customer_id IN (1, 2, 3, 4, 5);
```

### Killer 3: Implicit Conversions

```sql
-- Column is VARCHAR, but you pass INT:
-- ❌ BAD: Database converts EVERY row's column to INT (full scan!)
WHERE varchar_column = 12345

-- ✅ GOOD: Match the type
WHERE varchar_column = '12345'
```

### Killer 4: Correlated Subqueries (Running for Each Row)

```sql
-- ❌ Runs subquery once PER ROW in the outer query
SELECT name, salary,
    (SELECT AVG(salary) FROM employees e2 WHERE e2.department = e1.department) AS dept_avg
FROM employees e1;

-- ✅ JOIN with pre-computed aggregate
SELECT e.name, e.salary, d.dept_avg
FROM employees e
JOIN (
    SELECT department, AVG(salary) AS dept_avg
    FROM employees GROUP BY department
) d ON d.department = e.department;

-- ✅ Or use Window Function
SELECT name, salary, AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```

### Killer 5: OR Conditions Across Columns

```sql
-- ❌ Can't use a single index efficiently
WHERE department = 'Sales' OR salary > 100000

-- ✅ UNION ALL (each part can use its own index)
SELECT * FROM employees WHERE department = 'Sales'
UNION ALL
SELECT * FROM employees WHERE salary > 100000 AND department != 'Sales';
```

### Killer 6: Missing LIMIT on Existence Checks

```sql
-- ❌ Scans potentially millions of rows
SELECT COUNT(*) FROM orders WHERE customer_id = 42;
IF count > 0 THEN ...

-- ✅ Stops at first match!
SELECT EXISTS (SELECT 1 FROM orders WHERE customer_id = 42);

-- ✅ Or with LIMIT
SELECT 1 FROM orders WHERE customer_id = 42 LIMIT 1;
```

### Killer 7: ORDER BY Without Index Support

```sql
-- ❌ Sorts 10M rows, returns 10
SELECT * FROM huge_table ORDER BY created_at DESC LIMIT 10;

-- ✅ With index, it just reads the first 10 entries from the index!
CREATE INDEX idx_created_at ON huge_table (created_at DESC);
SELECT * FROM huge_table ORDER BY created_at DESC LIMIT 10;
-- Index scan reads 10 rows → instant!
```

### Killer 8: DISTINCT When Not Needed

```sql
-- ❌ Forces a sort/hash of all results
SELECT DISTINCT customer_id FROM orders WHERE order_date > '2024-01-01';

-- ✅ If you just need existence, use EXISTS
-- ✅ If you need unique values, ensure your query doesn't produce duplicates first
-- ✅ GROUP BY can sometimes be more efficient than DISTINCT
```

---

## 🔍 Part 6: Index Usage — Why the Optimizer Ignores Your Index

The optimizer might **choose not to use your index** even if it exists. Here's why:

```
WHEN THE OPTIMIZER SKIPS YOUR INDEX:

1. Low selectivity (>10-20% of rows match)
   → Full scan is cheaper than index + table lookup
   → Solution: Only index selective columns

2. Non-SARGable query (function on column)
   → Can't use index for seeking
   → Solution: Rewrite to be SARGable

3. Stale statistics
   → Optimizer thinks it'll return 10 rows (uses index)
   → Actually returns 500,000 rows (full scan was better)
   → Solution: UPDATE STATISTICS / ANALYZE

4. Wrong index type
   → B-tree index for LIKE '%pattern%' → useless
   → Solution: Use Full-Text index or trigram index

5. Covering not possible
   → Index has key columns but SELECT needs more
   → Each index hit requires table lookup (expensive)
   → Solution: INCLUDE columns in index (covering index)

6. Table is too small
   → Index overhead > full scan time
   → Solution: Don't index tiny tables
```

### Force Index Usage (Last Resort!)

```sql
-- PostgreSQL: Set planner costs to discourage seq scan
SET enable_seqscan = OFF;  -- For testing only!

-- MySQL: Index hints
SELECT * FROM orders FORCE INDEX (idx_customer_id) WHERE customer_id = 42;
SELECT * FROM orders USE INDEX (idx_order_date) WHERE order_date > '2024-01-01';
SELECT * FROM orders IGNORE INDEX (idx_status) WHERE status = 'active';

-- SQL Server: Index hints
SELECT * FROM orders WITH (INDEX(idx_customer_id)) WHERE customer_id = 42;
-- Or query hint:
SELECT * FROM orders WHERE customer_id = 42
OPTION (TABLE HINT(orders, INDEX(idx_customer_id)));

-- Oracle: Optimizer hints
SELECT /*+ INDEX(o idx_customer_id) */ * FROM orders o WHERE customer_id = 42;
SELECT /*+ FULL(o) */ * FROM orders o WHERE customer_id = 42;  -- Force full scan
```

> ⚠️ **Warning:** Forcing index usage is almost always a bad idea. If the optimizer avoids your index, it usually has a good reason. Fix the root cause (statistics, query structure, index design) instead.

---

## 🛠️ Part 7: Database-Specific Tuning Tools

### PostgreSQL

```sql
-- pg_stat_statements: Top resource-consuming queries
CREATE EXTENSION pg_stat_statements;

SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- pg_stat_user_tables: Table access patterns
SELECT relname, seq_scan, idx_scan, n_live_tup
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;
-- High seq_scan on large table? Needs an index!

-- pg_stat_user_indexes: Index usage
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
-- idx_scan = 0? Index is unused — consider dropping it!

-- auto_explain: Automatically log plans for slow queries
LOAD 'auto_explain';
SET auto_explain.log_min_duration = '100ms';
```

### SQL Server

```sql
-- Query Store (built-in query performance insight)
ALTER DATABASE MyDB SET QUERY_STORE = ON;

-- Find regressed queries
SELECT TOP 20 
    qt.query_sql_text,
    rs.avg_duration,
    rs.avg_cpu_time,
    rs.avg_logical_io_reads
FROM sys.query_store_query_text qt
JOIN sys.query_store_query q ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
ORDER BY rs.avg_duration DESC;

-- DMVs: Missing indexes the optimizer wishes existed
SELECT 
    migs.avg_user_impact,
    mid.statement AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_group_stats migs
JOIN sys.dm_db_missing_index_groups mig ON migs.group_handle = mig.index_group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
ORDER BY migs.avg_user_impact DESC;

-- Index usage stats
SELECT 
    OBJECT_NAME(ius.object_id) AS table_name,
    i.name AS index_name,
    ius.user_seeks, ius.user_scans, ius.user_lookups, ius.user_updates
FROM sys.dm_db_index_usage_stats ius
JOIN sys.indexes i ON ius.object_id = i.object_id AND ius.index_id = i.index_id
WHERE ius.database_id = DB_ID()
ORDER BY ius.user_seeks + ius.user_scans DESC;
```

### MySQL

```sql
-- Slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- Log queries taking > 1 second

-- Performance Schema: Top queries
SELECT 
    DIGEST_TEXT,
    COUNT_STAR AS exec_count,
    ROUND(SUM_TIMER_WAIT/1000000000, 2) AS total_time_ms,
    ROUND(AVG_TIMER_WAIT/1000000000, 2) AS avg_time_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- EXPLAIN ANALYZE (MySQL 8.0.18+)
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

### Oracle

```sql
-- AWR Report (Automatic Workload Repository)
@$ORACLE_HOME/rdbms/admin/awrrpt.sql

-- ASH (Active Session History) — what's happening RIGHT NOW
SELECT sql_id, COUNT(*) AS active_sessions
FROM v$active_session_history
WHERE sample_time > SYSDATE - INTERVAL '5' MINUTE
GROUP BY sql_id
ORDER BY active_sessions DESC;

-- SQL Tuning Advisor
DECLARE
    task_name VARCHAR2(30);
BEGIN
    task_name := DBMS_SQLTUNE.CREATE_TUNING_TASK(sql_id => 'abc123');
    DBMS_SQLTUNE.EXECUTE_TUNING_TASK(task_name);
END;
/
SELECT DBMS_SQLTUNE.REPORT_TUNING_TASK('task_name') FROM DUAL;
```

---

## 📐 Part 8: Query Optimization Checklist

```
┌──────────────────────────────────────────────────────────────────┐
│              QUERY OPTIMIZATION CHECKLIST                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  □ STEP 1: MEASURE FIRST                                         │
│    • Get the execution plan (EXPLAIN ANALYZE)                    │
│    • Identify the most expensive operator                        │
│    • Compare estimated vs actual rows                            │
│                                                                  │
│  □ STEP 2: CHECK THE BASICS                                      │
│    • Is the query SARGable? (no functions on columns)            │
│    • Are statistics up to date? (ANALYZE / UPDATE STATISTICS)    │
│    • Are appropriate indexes in place?                           │
│    • Is SELECT * being used? (limit columns)                     │
│                                                                  │
│  □ STEP 3: INDEX OPTIMIZATION                                    │
│    • Does a useful index exist for WHERE/JOIN/ORDER BY?          │
│    • Is the index being used? (check plan)                       │
│    • Would a covering index eliminate table lookups?             │
│    • Are there unused indexes wasting write performance?         │
│                                                                  │
│  □ STEP 4: QUERY RESTRUCTURING                                   │
│    • Replace correlated subquery with JOIN?                      │
│    • Replace OR with UNION ALL?                                  │
│    • Add LIMIT for existence checks?                             │
│    • Use EXISTS instead of COUNT(*)?                             │
│    • Push filters earlier (closer to FROM)?                      │
│                                                                  │
│  □ STEP 5: ARCHITECTURE                                          │
│    • Does the table need partitioning?                           │
│    • Can results be cached (materialized view)?                  │
│    • Should denormalization be considered?                        │
│    • Is pagination using OFFSET? (switch to keyset pagination)  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔥 Real-World Optimization Examples

### Example 1: Pagination — OFFSET vs Keyset

```sql
-- ❌ OFFSET pagination (gets slower on later pages)
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 100000;
-- Database must read and discard 100,000 rows! 🐌

-- ✅ Keyset pagination (constant speed regardless of page)
SELECT * FROM products WHERE id > 100000 ORDER BY id LIMIT 20;
-- Index seek to id 100000, read 20 rows. Instant! ⚡
```

### Example 2: COUNT Optimization

```sql
-- ❌ Exact count of huge table
SELECT COUNT(*) FROM orders;  -- Scans entire table 🐌

-- ✅ Approximate count (PostgreSQL) — instant!
SELECT reltuples::BIGINT AS estimate FROM pg_class WHERE relname = 'orders';

-- ✅ SQL Server — instant!
SELECT SUM(row_count) FROM sys.dm_db_partition_stats
WHERE object_id = OBJECT_ID('orders') AND index_id IN (0, 1);
```

### Example 3: JOIN Order Matters

```sql
-- The optimizer usually picks the best order, but with complex queries:
-- Small table → Large table (filter early, join less)

-- ✅ Filter BEFORE joining
SELECT o.order_id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.order_date > '2024-06-01'  -- Filters orders FIRST, then joins
  AND o.amount > 500;

-- Equivalent but explicit:
WITH filtered_orders AS (
    SELECT order_id, customer_id FROM orders
    WHERE order_date > '2024-06-01' AND amount > 500
)
SELECT fo.order_id, c.name
FROM filtered_orders fo
JOIN customers c ON c.id = fo.customer_id;
```

### Example 4: Parameter Sniffing (SQL Server)

```sql
-- SQL Server caches the first plan. If first call is atypical:
EXEC sp_get_orders @status = 'pending';   -- 100 rows → Nested Loop plan cached
EXEC sp_get_orders @status = 'completed'; -- 500,000 rows → STILL uses Nested Loop! 🐌

-- ✅ Fix 1: OPTION (RECOMPILE) for variable queries
SELECT * FROM orders WHERE status = @status
OPTION (RECOMPILE);  -- Fresh plan every time

-- ✅ Fix 2: OPTIMIZE FOR unknown
SELECT * FROM orders WHERE status = @status
OPTION (OPTIMIZE FOR UNKNOWN);  -- Uses average statistics

-- ✅ Fix 3: Plan guides (force specific plan)
```

---

## 🧪 Practice Challenges

### Challenge 1: Read This Plan
> What's wrong with this execution plan? How would you fix it?

```
Seq Scan on orders  (cost=0.00..25000.00 rows=1000000 width=100)
  Filter: (EXTRACT(YEAR FROM order_date) = 2024)
  Rows Removed by Filter: 950000
```

<details>
<summary>💡 Solution</summary>

**Problem:** Non-SARGable query (`EXTRACT` on column prevents index usage). Full table scan filtering out 95% of rows.

**Fix:**
```sql
-- Rewrite to be SARGable
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'

-- Create index
CREATE INDEX idx_orders_date ON orders (order_date);
```

New plan: `Index Scan using idx_orders_date (rows=50000)` — 20x fewer rows read!
</details>

### Challenge 2: Missing Index
> This query runs in 8 seconds. Make it run under 100ms.

```sql
SELECT c.name, COUNT(o.id) AS order_count, SUM(o.amount) AS total
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE c.country = 'US'
  AND o.order_date >= '2024-01-01'
GROUP BY c.name
ORDER BY total DESC
LIMIT 10;
```

<details>
<summary>💡 Solution</summary>

```sql
-- Index on customers for the WHERE filter
CREATE INDEX idx_customers_country ON customers (country) INCLUDE (id, name);

-- Index on orders for the JOIN and date filter
CREATE INDEX idx_orders_cust_date ON orders (customer_id, order_date) INCLUDE (amount);

-- The optimizer can now:
-- 1. Index scan customers WHERE country = 'US'
-- 2. Index scan orders for matching customer_ids after date
-- 3. No table lookups (covering indexes!)
```
</details>

---

## 🎯 Key Takeaways

```
┌──────────────────────────────────────────────────────────────────┐
│  QUERY OPTIMIZATION CHEAT SHEET                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  #1 RULE: MEASURE BEFORE OPTIMIZING                              │
│  Always EXPLAIN ANALYZE first. Don't guess.                      │
│                                                                  │
│  SARGABILITY:                                                    │
│  Never put functions on columns in WHERE                         │
│  WHERE f(col) = val ❌  →  WHERE col = f⁻¹(val) ✅               │
│                                                                  │
│  STATISTICS:                                                     │
│  Stale stats = bad plans. ANALYZE after bulk changes.            │
│                                                                  │
│  INDEXES:                                                        │
│  Right index + SARGable query = fast                             │
│  Covering index = fastest (no table lookup)                      │
│  Unused indexes = wasted write performance                       │
│                                                                  │
│  COMMON FIXES:                                                   │
│  Seq Scan → Add index + make SARGable                            │
│  Nested Loop (slow) → Likely missing index on inner table        │
│  High "Rows Removed" → Filter isn't using index                  │
│  Est. vs Actual differ → Update statistics                       │
│  OFFSET pagination → Switch to keyset pagination                 │
│                                                                  │
│  TOOLS:                                                          │
│  PostgreSQL: pg_stat_statements, auto_explain                    │
│  SQL Server: Query Store, DMVs, missing index DMV                │
│  MySQL: slow query log, Performance Schema, EXPLAIN ANALYZE      │
│  Oracle: AWR, ASH, SQL Tuning Advisor                            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

> 🚀 **Next Up:** [2.12 — Advanced SQL — Pivoting, JSON, XML, Regex](./12-Advanced-SQL.md) — Master the advanced features that make SQL a complete data transformation language.

---

*The fastest query is the one that reads the least data. Every optimization technique in this chapter serves that one principle. Internalize it, and you'll intuitively write fast SQL.* ⚡
