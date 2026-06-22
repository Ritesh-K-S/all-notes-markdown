# 🚀 Chapter 2C.4 — SQL Server Performance Tuning

> **"A slow query isn't just a technical problem — it's a user staring at a loading spinner, a customer abandoning their cart, a business losing money. Every millisecond matters."**

> **Level:** 🔴 Advanced | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~6-8 hours (with practice)
> **Prerequisites:** Chapter 2C.1 (Architecture), Chapter 2C.3 (T-SQL), Chapter 1.7 (Indexing)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- **Read execution plans** like a DBA reads the Matrix
- Use **Query Store** — SQL Server's built-in performance time machine
- Master **DMVs** (Dynamic Management Views) — your X-ray into the engine
- Identify and fix **index problems** (missing, unused, duplicate)
- Understand **statistics** and why the optimizer makes bad choices
- Defeat **parameter sniffing** — the most misunderstood SQL Server issue
- Diagnose **wait statistics** — what is SQL Server actually waiting for?
- Apply the **performance tuning methodology** used by top DBAs

---

## 🧠 The Performance Tuning Mindset

Before touching a single query, understand this framework:

```
╔══════════════════════════════════════════════════════════════════════╗
║              THE PERFORMANCE TUNING PYRAMID                          ║
║                                                                      ║
║                        ┌──────┐                                     ║
║                        │ App  │  ← Fix the app first               ║
║                        │ Code │    (N+1 queries, no batching)      ║
║                       ┌┴──────┴┐                                    ║
║                       │ Query  │  ← Rewrite bad queries            ║
║                       │ Tuning │    (SARGable, JOINs, subqueries) ║
║                      ┌┴────────┴┐                                   ║
║                      │ Indexing │  ← Add/fix indexes               ║
║                      │          │    (The biggest bang for buck!)   ║
║                     ┌┴──────────┴┐                                  ║
║                     │ Statistics │  ← Update stale stats           ║
║                     │            │    (Optimizer needs good data)  ║
║                    ┌┴────────────┴┐                                 ║
║                    │ Server Config│  ← Memory, MAXDOP, tempdb     ║
║                    │              │                                 ║
║                   ┌┴──────────────┴┐                                ║
║                   │   Hardware     │  ← More RAM, faster disk,    ║
║                   │                │    more CPU (last resort!)    ║
║                   └────────────────┘                                ║
║                                                                      ║
║   💡 Always start from the TOP. Hardware is the LAST resort.        ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 📊 1. Execution Plans — Your #1 Diagnostic Tool

Every performance investigation starts here. The execution plan shows you **exactly** what SQL Server is doing with your query.

### How to Get an Execution Plan

```sql
-- Method 1: Estimated Plan (doesn't run the query)
-- Shortcut: Ctrl + L in SSMS
SET SHOWPLAN_XML ON;
GO
SELECT * FROM Products WHERE Price > 100;
GO
SET SHOWPLAN_XML OFF;

-- Method 2: Actual Plan (runs the query — shows real row counts)
-- Shortcut: Ctrl + M (toggle on), then F5
SET STATISTICS XML ON;
GO
SELECT * FROM Products WHERE Price > 100;
GO
SET STATISTICS XML OFF;

-- Method 3: Text plan (quick and dirty)
SET STATISTICS PROFILE ON;
GO
SELECT * FROM Products WHERE Price > 100;
GO
SET STATISTICS PROFILE OFF;

-- Method 4: I/O and Time statistics (always use these!)
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO
SELECT * FROM Products WHERE Price > 100;
GO
-- Output shows: logical reads, physical reads, CPU time, elapsed time
```

### Reading Execution Plans — The Visual Guide

```
╔════════════════════════════════════════════════════════════════════════╗
║                EXECUTION PLAN — READ RIGHT TO LEFT!                    ║
║                                                                        ║
║   ← Data flows from RIGHT to LEFT                                    ║
║   ← Execution starts from RIGHT side                                 ║
║                                                                        ║
║   ┌────────┐    ┌────────────┐    ┌──────────────┐    ┌──────────┐  ║
║   │ SELECT │ ←─ │  Nested    │ ←─ │ Index Seek   │ ←─ │  Index   │  ║
║   │        │    │  Loops     │    │ (Products    │    │  Seek    │  ║
║   │  Cost: │    │  Join      │    │  PK_Products)│    │ (ix_Cat) │  ║
║   │   0%   │    │  Cost: 10% │    │  Cost: 60%   │    │ Cost: 30%│  ║
║   └────────┘    └────────────┘    └──────────────┘    └──────────┘  ║
║                                                                        ║
║   📏 Arrow Thickness = Row count (thicker = more rows)               ║
║   💰 Cost % = Relative cost of each operation                        ║
║   ⚠️ Warnings = Yellow triangle = problem detected!                  ║
║                                                                        ║
╚════════════════════════════════════════════════════════════════════════╝
```

### The Most Important Operators

```
┌──────────────────────────────────────────────────────────────────────┐
│                  EXECUTION PLAN OPERATORS                             │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  🟢 GOOD OPERATORS (what you WANT to see)                           │
│  ─────────────────────────────────────                                │
│  Index Seek          → Jump directly to data (B-Tree traversal)     │
│  Clustered Index Seek → Best case for single-row lookups            │
│  Nested Loops Join   → Great for small datasets + indexed lookups   │
│  Stream Aggregate    → Efficient grouped aggregation                │
│  Constant Scan       → Reading a constant value (trivial)           │
│                                                                       │
│  🟡 DEPENDS-ON-CONTEXT OPERATORS                                    │
│  ────────────────────────────────                                     │
│  Index Scan          → Reading entire index (OK for small tables)   │
│  Merge Join          → Good for large sorted datasets               │
│  Hash Match Join     → Good for large unsorted datasets             │
│  Sort                → Needed for ORDER BY (expensive if large)     │
│  Hash Match Aggregate → Fine for large GROUP BY                     │
│                                                                       │
│  🔴 BAD OPERATORS (red flags!)                                       │
│  ──────────────────────────────                                       │
│  Table Scan          → No usable index — reading ENTIRE table       │
│  Clustered Index Scan → Scanning ALL rows of clustered index        │
│  Key Lookup          → Going back to table for missing columns      │
│  RID Lookup          → Same as key lookup but for heaps             │
│  Sort (with spill)   → Not enough memory → spilling to tempdb     │
│  Eager Spool         → Materializing data to tempdb (often bad)     │
│  Table Spool (lazy)  → Re-reading same data multiple times          │
│                                                                       │
│  ⚠️ WARNING SIGNS IN PLANS                                           │
│  ──────────────────────────────                                       │
│  ⚠️ Yellow triangle          → Implicit conversion, missing index  │
│  ⚠️ Thick arrows             → More rows than expected             │
│  ⚠️ EstimatedRows ≠ Actual   → Stale statistics!                   │
│  ⚠️ "Missing Index" hint     → SQL Server tells you what to create │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Key Lookup — The Hidden Performance Killer

```
╔══════════════════════════════════════════════════════════════════╗
║                   KEY LOOKUP PROBLEM                             ║
║                                                                  ║
║   Query: SELECT Name, Price, Description                        ║
║          FROM Products WHERE CategoryID = 5                     ║
║                                                                  ║
║   Index: ix_CategoryID (CategoryID)  ← doesn't include          ║
║                                        Name, Price, Description ║
║                                                                  ║
║   What happens:                                                  ║
║   ┌──────────────┐        ┌──────────────────┐                  ║
║   │ Index Seek   │ ──→──→ │ Key Lookup       │                  ║
║   │ ix_CategoryID│  For   │ (Clustered Index) │                  ║
║   │              │  EACH  │                   │                  ║
║   │ Finds 500    │  row!  │ Goes back to get  │                  ║
║   │ matching rows│        │ Name, Price, Desc │                  ║
║   └──────────────┘        └──────────────────┘                  ║
║                                                                  ║
║   500 rows × 1 key lookup each = 500 random I/Os! 💀            ║
║                                                                  ║
║   FIX: CREATE COVERING INDEX                                    ║
║   CREATE INDEX ix_Cat_Cover ON Products (CategoryID)            ║
║   INCLUDE (Name, Price, Description);                           ║
║                                                                  ║
║   Now: Index Seek ONLY → 0 key lookups → FAST! ⚡             ║
╚══════════════════════════════════════════════════════════════════╝
```

### Practical: Reading STATISTICS IO Output

```sql
SET STATISTICS IO ON;
SELECT * FROM Orders WHERE CustomerID = 42;
SET STATISTICS IO OFF;

-- Output:
-- Table 'Orders'. Scan count 1, logical reads 847,
-- physical reads 0, read-ahead reads 0, lob logical reads 0.

-- What it means:
-- logical reads = 847    → 847 pages read from Buffer Pool (memory)
--                         → 847 × 8KB = ~6.6 MB of data read
-- physical reads = 0     → Nothing from disk (all cached) ✅
-- scan count = 1         → One scan/seek operation

-- 💡 YOUR GOAL: Minimize logical reads!
-- Before tuning: logical reads = 847
-- After adding index: logical reads = 3  ← 280x improvement! 🚀
```

---

## 🏪 2. Query Store — The Performance Time Machine

Query Store (SQL Server 2016+) automatically captures and retains **query plans and performance data** over time. It's your history book for performance.

```
╔══════════════════════════════════════════════════════════════════╗
║                      QUERY STORE                                 ║
║           "Flight Data Recorder for SQL Server"                  ║
║                                                                  ║
║   ┌──────────────────────────────────────────────────────────┐  ║
║   │                                                           │  ║
║   │   Every query that runs gets:                            │  ║
║   │   • Query text stored                                     │  ║
║   │   • Execution plan(s) stored                             │  ║
║   │   • Runtime stats stored (CPU, I/O, duration, rows)      │  ║
║   │   • Historical data retained (configurable days)         │  ║
║   │                                                           │  ║
║   │   You can:                                                │  ║
║   │   ✅ See how query performance changed over time         │  ║
║   │   ✅ Compare plans (before vs after index change)        │  ║
║   │   ✅ Find regressed queries (was fast, now slow)         │  ║
║   │   ✅ FORCE a good plan (pin it!)                         │  ║
║   │   ✅ Identify top resource consumers                      │  ║
║   │                                                           │  ║
║   └──────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Enable & Configure Query Store

```sql
-- Enable Query Store (on by default in SQL 2022+)
ALTER DATABASE [AdventureWorks] SET QUERY_STORE = ON;

-- Configure it
ALTER DATABASE [AdventureWorks] SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,             -- Active collection
    DATA_FLUSH_INTERVAL_SECONDS = 900,       -- Flush to disk every 15 min
    INTERVAL_LENGTH_MINUTES = 60,            -- Aggregate stats per hour
    MAX_STORAGE_SIZE_MB = 1024,              -- 1 GB max storage
    STALE_QUERY_THRESHOLD_DAYS = 30,         -- Keep 30 days of history
    QUERY_CAPTURE_MODE = AUTO,               -- Only capture significant queries
    SIZE_BASED_CLEANUP_MODE = AUTO,          -- Auto-cleanup when full
    MAX_PLANS_PER_QUERY = 200,              -- Max plans per query
    WAIT_STATS_CAPTURE_MODE = ON             -- Capture wait stats too!
);
```

### Query Store DMVs — Find Problems Programmatically

```sql
-- 🔍 Top 10 queries by TOTAL CPU time
SELECT TOP 10
    q.query_id,
    qt.query_sql_text,
    SUM(rs.avg_cpu_time * rs.count_executions) AS total_cpu_time,
    SUM(rs.count_executions) AS total_executions,
    AVG(rs.avg_cpu_time) AS avg_cpu_time,
    AVG(rs.avg_logical_io_reads) AS avg_logical_reads
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
GROUP BY q.query_id, qt.query_sql_text
ORDER BY total_cpu_time DESC;

-- 🔍 Regressed queries (was fast, now slow)
SELECT 
    q.query_id,
    qt.query_sql_text,
    rs1.avg_duration AS duration_before,
    rs2.avg_duration AS duration_after,
    (rs2.avg_duration - rs1.avg_duration) / rs1.avg_duration * 100 AS pct_regression
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs1 ON p.plan_id = rs1.plan_id
JOIN sys.query_store_runtime_stats rs2 ON p.plan_id = rs2.plan_id
JOIN sys.query_store_runtime_stats_interval i1 ON rs1.runtime_stats_interval_id = i1.runtime_stats_interval_id
JOIN sys.query_store_runtime_stats_interval i2 ON rs2.runtime_stats_interval_id = i2.runtime_stats_interval_id
WHERE i1.start_time < DATEADD(DAY, -7, GETDATE())  -- Before: 7+ days ago
  AND i2.start_time > DATEADD(DAY, -1, GETDATE())  -- After: last 24 hours
  AND rs2.avg_duration > rs1.avg_duration * 2       -- 2x slower
ORDER BY pct_regression DESC;

-- 🔧 Force a specific plan for a query
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 17;

-- 🔓 Unforce a plan
EXEC sp_query_store_unforce_plan @query_id = 42, @plan_id = 17;
```

### Query Store in SSMS (Visual)

```
In SSMS → Database → Query Store (folder) →
├── Regressed Queries          ← Queries that got worse
├── Overall Resource Consumption ← Top consumers
├── Top Resource Consuming Queries ← Drilldown
├── Queries With Forced Plans   ← Plans you pinned
├── Queries With High Variation ← Inconsistent performance
└── Tracked Queries             ← Queries you're monitoring
```

---

## 🔍 3. Dynamic Management Views (DMVs) — X-Ray Vision

DMVs are **views into SQL Server's internal state**. They expose real-time performance data that's normally invisible.

### The Essential DMVs Every DBA Must Know

```
╔════════════════════════════════════════════════════════════════════╗
║                    DMV CATEGORIES                                  ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  📊 EXECUTION STATS                                               ║
║     sys.dm_exec_query_stats         → Query-level stats           ║
║     sys.dm_exec_procedure_stats     → SP-level stats              ║
║     sys.dm_exec_requests            → Currently running queries   ║
║     sys.dm_exec_sessions            → Active sessions             ║
║     sys.dm_exec_cached_plans        → Plan cache contents         ║
║                                                                    ║
║  📦 INDEX STATS                                                   ║
║     sys.dm_db_index_usage_stats     → How indexes are used        ║
║     sys.dm_db_missing_index_details → Indexes SQL Server wants    ║
║     sys.dm_db_index_physical_stats  → Fragmentation levels        ║
║                                                                    ║
║  🧠 MEMORY                                                        ║
║     sys.dm_os_buffer_descriptors    → What's in buffer pool       ║
║     sys.dm_os_memory_clerks         → Memory allocation breakdown ║
║     sys.dm_os_process_memory        → Overall memory usage        ║
║                                                                    ║
║  ⏳ WAIT STATS                                                    ║
║     sys.dm_os_wait_stats            → What is SQL Server waiting  ║
║     sys.dm_exec_session_wait_stats  → Per-session waits           ║
║                                                                    ║
║  🔒 LOCKS                                                         ║
║     sys.dm_tran_locks               → Current locks held          ║
║     sys.dm_os_waiting_tasks         → Tasks waiting for resources ║
║                                                                    ║
║  📁 I/O                                                           ║
║     sys.dm_io_virtual_file_stats    → I/O per database file       ║
║                                                                    ║
╚════════════════════════════════════════════════════════════════════╝
```

### Top Queries by CPU, Reads, Duration

```sql
-- 🔥 Top 20 most expensive queries by CPU
SELECT TOP 20
    qs.total_worker_time / qs.execution_count AS avg_cpu_us,
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    qs.execution_count,
    qs.total_worker_time AS total_cpu_us,
    SUBSTRING(st.text, 
        (qs.statement_start_offset / 2) + 1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset) / 2) + 1
    ) AS query_text,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_worker_time DESC;

-- 🔍 What's running RIGHT NOW?
SELECT 
    r.session_id,
    r.status,
    r.command,
    r.cpu_time,
    r.total_elapsed_time / 1000 AS elapsed_sec,
    r.logical_reads,
    r.wait_type,
    r.wait_time,
    r.blocking_session_id,
    t.text AS query_text,
    p.query_plan
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) p
WHERE r.session_id > 50  -- Exclude system sessions
ORDER BY r.total_elapsed_time DESC;
```

---

## 📏 4. Index Tuning — The Biggest Performance Lever

### Finding Missing Indexes

```sql
-- 🔍 SQL Server tells you what indexes it WISHES it had!
SELECT TOP 20
    ROUND(s.avg_total_user_cost * s.avg_user_impact * (s.user_seeks + s.user_scans), 0) 
        AS improvement_measure,
    'CREATE INDEX [IX_' + OBJECT_NAME(d.object_id, d.database_id) + '_' 
        + REPLACE(REPLACE(REPLACE(
            ISNULL(d.equality_columns, '') + ISNULL(', ' + d.inequality_columns, ''),
            '[', ''), ']', ''), ', ', '_')
        + '] ON ' + d.statement
        + ' (' + ISNULL(d.equality_columns, '')
        + CASE WHEN d.equality_columns IS NOT NULL AND d.inequality_columns IS NOT NULL 
               THEN ', ' ELSE '' END
        + ISNULL(d.inequality_columns, '')
        + ')'
        + ISNULL(' INCLUDE (' + d.included_columns + ')', '')
    AS create_index_statement,
    s.user_seeks,
    s.user_scans,
    d.equality_columns,
    d.inequality_columns,
    d.included_columns
FROM sys.dm_db_missing_index_details d
JOIN sys.dm_db_missing_index_groups g ON d.index_handle = g.index_handle
JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
WHERE d.database_id = DB_ID()
ORDER BY improvement_measure DESC;
```

### Finding Unused Indexes (Wasting Space & Slowing Writes)

```sql
-- 🗑️ Indexes that exist but NOBODY uses
SELECT 
    OBJECT_NAME(i.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc,
    u.user_seeks,
    u.user_scans,
    u.user_lookups,
    u.user_updates,  -- Writes are paying the cost for this index!
    (SELECT SUM(p.rows) FROM sys.partitions p 
     WHERE p.index_id = i.index_id AND p.object_id = i.object_id) AS row_count
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats u 
    ON i.object_id = u.object_id AND i.index_id = u.index_id
    AND u.database_id = DB_ID()
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
  AND i.type_desc <> 'HEAP'
  AND i.is_primary_key = 0
  AND i.is_unique = 0
  AND ISNULL(u.user_seeks, 0) + ISNULL(u.user_scans, 0) + ISNULL(u.user_lookups, 0) = 0
  AND ISNULL(u.user_updates, 0) > 0  -- Being updated but never read!
ORDER BY u.user_updates DESC;

-- ⚠️ CAUTION: dm_db_index_usage_stats resets on restart!
-- Check after the server has been up for a full business cycle (week/month)
```

### Index Fragmentation — When to Rebuild vs Reorganize

```sql
-- 📊 Check fragmentation
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.avg_page_space_used_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 5
  AND ips.page_count > 1000  -- Ignore tiny indexes
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

```
╔═══════════════════════════════════════════════════════════════╗
║           INDEX MAINTENANCE DECISION TREE                     ║
║                                                               ║
║   Fragmentation < 5%        → Do NOTHING ✅                  ║
║   Fragmentation 5% - 30%   → REORGANIZE (online, lightweight)║
║   Fragmentation > 30%      → REBUILD (heavier, better result)║
║                                                               ║
║   -- Reorganize (online — no blocking)                       ║
║   ALTER INDEX ix_name ON TableName REORGANIZE;               ║
║                                                               ║
║   -- Rebuild (offline by default — blocks queries)           ║
║   ALTER INDEX ix_name ON TableName REBUILD;                  ║
║                                                               ║
║   -- Rebuild ONLINE (Enterprise only — no blocking!)         ║
║   ALTER INDEX ix_name ON TableName                           ║
║       REBUILD WITH (ONLINE = ON);                            ║
║                                                               ║
║   -- Rebuild ALL indexes on a table                          ║
║   ALTER INDEX ALL ON TableName REBUILD;                      ║
║                                                               ║
║   💡 Use Ola Hallengren's maintenance scripts in production  ║
║      (industry standard — free & open source)                ║
╚═══════════════════════════════════════════════════════════════╝
```

### SARGable Queries — Can the Optimizer USE Your Index?

```sql
-- SARG = Search ARGument — can SQL Server "seek" using the index?

-- ✅ SARGable (index CAN be used)
SELECT * FROM Orders WHERE OrderDate >= '2026-01-01';
SELECT * FROM Products WHERE ProductName LIKE 'Cha%';    -- Leading wildcard OK
SELECT * FROM Employees WHERE DepartmentID = 5;

-- ❌ NOT SARGable (index CANNOT be used — full scan!)
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2026;        -- Function on column!
SELECT * FROM Products WHERE ProductName LIKE '%chai%';   -- Leading % wildcard
SELECT * FROM Employees WHERE Salary + 1000 > 50000;     -- Arithmetic on column!
SELECT * FROM Orders WHERE CAST(OrderDate AS DATE) = '2026-01-01';  -- CAST on column!

-- 🔧 FIX: Move the function to the OTHER side
-- ❌ WHERE YEAR(OrderDate) = 2026
-- ✅ WHERE OrderDate >= '2026-01-01' AND OrderDate < '2027-01-01'

-- ❌ WHERE Salary + 1000 > 50000
-- ✅ WHERE Salary > 49000

-- ❌ WHERE CAST(Price AS INT) = 10
-- ✅ WHERE Price >= 10 AND Price < 11
```

> 💡 **The Golden Rule:** NEVER put a function on the column side of a WHERE clause. Always manipulate the VALUE side instead.

---

## 📈 5. Statistics — Why the Optimizer Makes Bad Choices

Statistics are SQL Server's **knowledge about data distribution**. When statistics are wrong, the optimizer picks bad plans.

```
╔══════════════════════════════════════════════════════════════════╗
║                   HOW STATISTICS WORK                            ║
║                                                                  ║
║   Question: "How many Orders have CustomerID = 42?"             ║
║                                                                  ║
║   SQL Server looks at statistics (histogram):                   ║
║                                                                  ║
║   ┌─────────────┬──────────┬───────────────┐                   ║
║   │ RANGE_HI_KEY│ EQ_ROWS  │ RANGE_ROWS    │                   ║
║   ├─────────────┼──────────┼───────────────┤                   ║
║   │ 10          │ 50       │ 0             │                   ║
║   │ 20          │ 30       │ 150           │                   ║
║   │ 40          │ 25       │ 200           │                   ║
║   │ 50          │ 45       │ 180           │  ← CustomerID 42  ║
║   │ 60          │ 20       │ 100           │     falls here    ║
║   └─────────────┴──────────┴───────────────┘                   ║
║                                                                  ║
║   Estimate for CustomerID = 42:                                 ║
║   → It falls in range (40, 50), RANGE_ROWS = 180              ║
║   → 180 rows / (50-40-1 distinct values) ≈ 20 rows estimated  ║
║                                                                  ║
║   If actual rows = 5,000 → WRONG estimate → BAD plan!         ║
║                                                                  ║
║   Solution: UPDATE STATISTICS                                   ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Managing Statistics

```sql
-- View statistics for a table
DBCC SHOW_STATISTICS ('Products', 'ix_CategoryID');

-- Check when statistics were last updated
SELECT 
    OBJECT_NAME(object_id) AS TableName,
    name AS StatsName,
    STATS_DATE(object_id, stats_id) AS LastUpdated,
    auto_created,
    user_created,
    filter_definition
FROM sys.stats
WHERE object_id = OBJECT_ID('Products')
ORDER BY LastUpdated;

-- Update statistics for one table
UPDATE STATISTICS Products;

-- Update with full scan (most accurate, but slow)
UPDATE STATISTICS Products WITH FULLSCAN;

-- Update all statistics in the database
EXEC sp_updatestats;

-- 💡 Enable auto-update with better sampling (SQL 2016+)
ALTER DATABASE [AdventureWorks]
SET AUTO_UPDATE_STATISTICS ON;

-- For very large tables, enable async stats update
ALTER DATABASE [AdventureWorks]
SET AUTO_UPDATE_STATISTICS_ASYNC ON;
```

### When Statistics Go Wrong — The Symptoms

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Estimated rows ≠ Actual rows (10x+) | Stale statistics | `UPDATE STATISTICS` with FULLSCAN |
| Good plan yesterday, bad today | Auto-update threshold not met | Manual update or lower threshold |
| Hash join on small table | Overestimated row count | Update stats, check histogram |
| Nested loops on huge table | Underestimated row count | Update stats, check histogram |
| Parameter sniffing | Plan optimized for wrong value | See Section 6 below |

---

## 🎭 6. Parameter Sniffing — The Most Misunderstood Problem

### What Is Parameter Sniffing?

```
╔══════════════════════════════════════════════════════════════════╗
║                   PARAMETER SNIFFING                             ║
║                                                                  ║
║   SQL Server CACHES the execution plan from the FIRST call.     ║
║   Every subsequent call REUSES that same plan.                  ║
║                                                                  ║
║   Problem: What if the first call had UNUSUAL parameter values? ║
║                                                                  ║
║   Example: GetOrders(@CustomerID)                               ║
║                                                                  ║
║   First call:  @CustomerID = 42    (has 5 orders)               ║
║   Plan chosen: Index Seek + Nested Loops (great for 5 rows!)   ║
║   Plan cached: ✅                                                ║
║                                                                  ║
║   Second call: @CustomerID = 1     (has 500,000 orders!)        ║
║   Plan REUSED: Index Seek + Nested Loops (TERRIBLE for 500K!)  ║
║   Result: 500,000 key lookups → query runs for 5 minutes! 💀  ║
║                                                                  ║
║   The SAME stored procedure is either fast or slow depending    ║
║   on who called it FIRST. That's parameter sniffing.           ║
╚══════════════════════════════════════════════════════════════════╝
```

### Solutions for Parameter Sniffing

```sql
-- Solution 1: OPTION (RECOMPILE) — Compile fresh plan every time
-- Best for: Infrequently called queries, wildly varying parameters
CREATE PROCEDURE dbo.GetOrders @CustomerID INT
AS
BEGIN
    SELECT * FROM Orders WHERE CustomerID = @CustomerID
    OPTION (RECOMPILE);  -- ← New plan every time
END;
-- ⚠️ Cost: Extra CPU for compilation each call

-- Solution 2: OPTIMIZE FOR UNKNOWN — Use average statistics
-- Best for: When you don't know a "typical" value
CREATE PROCEDURE dbo.GetOrders @CustomerID INT
AS
BEGIN
    SELECT * FROM Orders WHERE CustomerID = @CustomerID
    OPTION (OPTIMIZE FOR UNKNOWN);  -- ← Use avg distribution
END;

-- Solution 3: OPTIMIZE FOR specific value
-- Best for: When you know the "typical" parameter value
CREATE PROCEDURE dbo.GetOrders @CustomerID INT
AS
BEGIN
    SELECT * FROM Orders WHERE CustomerID = @CustomerID
    OPTION (OPTIMIZE FOR (@CustomerID = 100));  -- ← Optimize for "typical" customer
END;

-- Solution 4: Local variable assignment (blunt instrument)
CREATE PROCEDURE dbo.GetOrders @CustomerID INT
AS
BEGIN
    DECLARE @CustID INT = @CustomerID;  -- ← Breaks the sniffing!
    SELECT * FROM Orders WHERE CustomerID = @CustID;
    -- Optimizer can't see the original parameter → uses average stats
END;

-- Solution 5: Plan Guides (DBA-level — force a plan without code changes)
EXEC sp_create_plan_guide 
    @name = N'Guide_GetOrders',
    @stmt = N'SELECT * FROM Orders WHERE CustomerID = @CustomerID',
    @type = N'OBJECT',
    @module_or_batch = N'dbo.GetOrders',
    @hints = N'OPTION (RECOMPILE)';

-- Solution 6: Query Store forced plans (SQL 2016+) — best modern approach!
-- Find the good plan_id in Query Store, then:
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 17;
```

---

## ⏳ 7. Wait Statistics — What Is SQL Server Waiting For?

Every time SQL Server can't do useful work, it records a **wait**. Wait statistics tell you WHERE the bottleneck is.

```
╔══════════════════════════════════════════════════════════════════╗
║                 TOP WAIT TYPES & WHAT THEY MEAN                  ║
╠═══════════════════╦══════════════════════════════════════════════╣
║ Wait Type         ║ Meaning & Action                             ║
╠═══════════════════╬══════════════════════════════════════════════╣
║ CXPACKET          ║ Parallelism waits — adjust MAXDOP/cost      ║
║                   ║ threshold. Not always bad!                   ║
╠═══════════════════╬══════════════════════════════════════════════╣
║ PAGEIOLATCH_SH    ║ Reading pages from DISK → need more RAM     ║
║                   ║ or better I/O subsystem                      ║
╠═══════════════════╬══════════════════════════════════════════════╣
║ WRITELOG          ║ Writing to transaction log → slow log disk   ║
║                   ║ Move log to faster SSD                       ║
╠═══════════════════╬══════════════════════════════════════════════╣
║ LCK_M_X / LCK_M_S║ Lock contention — queries blocking each     ║
║                   ║ other. Fix: shorter transactions, indexing   ║
╠═══════════════════╬══════════════════════════════════════════════╣
║ SOS_SCHEDULER_YIELD║ CPU pressure — need more CPU or better     ║
║                   ║ queries                                      ║
╠═══════════════════╬══════════════════════════════════════════════╣
║ ASYNC_NETWORK_IO  ║ Client is slow consuming results. App       ║
║                   ║ problem, not SQL Server.                     ║
╠═══════════════════╬══════════════════════════════════════════════╣
║ RESOURCE_SEMAPHORE║ Not enough memory for query grants →        ║
║                   ║ queries queueing for memory                  ║
╠═══════════════════╬══════════════════════════════════════════════╣
║ LATCH_EX          ║ Internal memory structure contention        ║
║                   ║ (often tempdb)                               ║
╠═══════════════════╬══════════════════════════════════════════════╣
║ IO_COMPLETION     ║ General I/O wait → disk bottleneck          ║
╚═══════════════════╩══════════════════════════════════════════════╝
```

### Query Wait Stats

```sql
-- 📊 Top waits (excluding benign/idle waits)
SELECT TOP 20
    wait_type,
    wait_time_ms / 1000.0 AS wait_time_sec,
    signal_wait_time_ms / 1000.0 AS signal_wait_sec,
    (wait_time_ms - signal_wait_time_ms) / 1000.0 AS resource_wait_sec,
    waiting_tasks_count,
    ROUND(100.0 * wait_time_ms / SUM(wait_time_ms) OVER(), 2) AS pct_of_total
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    -- Exclude benign/idle waits
    'SLEEP_TASK', 'BROKER_TASK_STOP', 'BROKER_EVENTHANDLER',
    'CLR_AUTO_EVENT', 'CLR_MANUAL_EVENT', 'LAZYWRITER_SLEEP',
    'SQLTRACE_BUFFER_FLUSH', 'WAITFOR', 'XE_TIMER_EVENT',
    'XE_DISPATCHER_WAIT', 'FT_IFTS_SCHEDULER_IDLE_WAIT',
    'BROKER_TO_FLUSH', 'SQLTRACE_INCREMENTAL_FLUSH_SLEEP',
    'SP_SERVER_DIAGNOSTICS_SLEEP', 'HADR_FILESTREAM_IOMGR_IOCOMPLETION',
    'DIRTY_PAGE_POLL', 'CHECKPOINT_QUEUE', 'REQUEST_FOR_DEADLOCK_SEARCH',
    'LOGMGR_QUEUE', 'ONDEMAND_TASK_QUEUE', 'QDS_PERSIST_TASK_MAIN_LOOP_SLEEP',
    'QDS_CLEANUP_STALE_QUERIES_TASK_MAIN_LOOP_SLEEP'
)
AND wait_time_ms > 0
ORDER BY wait_time_ms DESC;
```

### The Wait Stats Decision Tree

```
Your top wait is...

PAGEIOLATCH_SH / PAGEIOLATCH_EX
├── Is Buffer Pool Hit Ratio < 99%?
│   ├── YES → Add more RAM (data doesn't fit in memory)
│   └── NO  → Queries reading too much data → fix queries/indexes
│
CXPACKET / CXCONSUMER
├── Is MAXDOP set properly?
│   ├── NO → Set to # cores per NUMA node (max 8)
│   └── YES → Raise cost threshold for parallelism (50+)
│
LCK_M_X / LCK_M_S
├── Long-running transactions holding locks?
│   ├── YES → Shorten transactions, use RCSI
│   └── NO  → Missing indexes → table/page locks instead of row
│
SOS_SCHEDULER_YIELD
├── CPU bottleneck
│   ├── Fix expensive queries first (top CPU consumers)
│   └── Then consider more CPUs
│
WRITELOG
├── Transaction log on slow disk?
│   ├── YES → Move to SSD/NVMe
│   └── NO  → Too many small transactions → batch them
```

---

## 🔧 8. Essential Server Configuration for Performance

```sql
-- 📋 THE BIG 5 SERVER SETTINGS FOR PERFORMANCE

-- 1️⃣ MAX SERVER MEMORY (ALWAYS set this!)
-- Rule: Total RAM - 4GB (for OS) - RAM for other apps
-- 64 GB server → 64 - 4 = 60 GB for SQL Server
EXEC sp_configure 'max server memory (MB)', 61440;  -- 60 GB
RECONFIGURE;

-- 2️⃣ MAX DEGREE OF PARALLELISM (MAXDOP)
-- Rule: # of cores per NUMA node, max 8
-- 16-core server, 1 NUMA node → MAXDOP = 8
EXEC sp_configure 'max degree of parallelism', 8;
RECONFIGURE;

-- 3️⃣ COST THRESHOLD FOR PARALLELISM
-- Default is 5 (WAY too low — almost everything goes parallel)
-- Recommended: 50 (only expensive queries go parallel)
EXEC sp_configure 'cost threshold for parallelism', 50;
RECONFIGURE;

-- 4️⃣ OPTIMIZE FOR AD HOC WORKLOADS
-- Prevents single-use plans from bloating the plan cache
EXEC sp_configure 'optimize for ad hoc workloads', 1;
RECONFIGURE;

-- 5️⃣ TEMPDB CONFIGURATION
-- Create multiple tempdb data files = # of CPU cores (max 8)
-- All files SAME SIZE and SAME autogrowth
ALTER DATABASE tempdb 
MODIFY FILE (NAME = tempdev, SIZE = 8192MB, FILEGROWTH = 1024MB);

ALTER DATABASE tempdb 
ADD FILE (NAME = tempdev2, FILENAME = 'T:\tempdb\tempdev2.ndf', 
          SIZE = 8192MB, FILEGROWTH = 1024MB);
-- Repeat for tempdev3, tempdev4, etc.
```

### Read Committed Snapshot Isolation (RCSI) — Reduce Blocking

```sql
-- Enable RCSI — readers don't block writers and vice versa!
ALTER DATABASE [AdventureWorks] 
SET READ_COMMITTED_SNAPSHOT ON;
-- ← Uses row versioning in tempdb (like PostgreSQL's MVCC)
-- ← Dramatically reduces lock contention
-- ← Default behavior for Azure SQL Database!

-- Check if RCSI is enabled
SELECT name, is_read_committed_snapshot_on
FROM sys.databases
WHERE name = 'AdventureWorks';
```

---

## 🏥 9. The Performance Tuning Checklist

```
╔════════════════════════════════════════════════════════════════════════╗
║             SQL SERVER PERFORMANCE TUNING CHECKLIST                    ║
╠════════════════════════════════════════════════════════════════════════╣
║                                                                        ║
║  STEP 1: IDENTIFY THE PROBLEM                                        ║
║  □ Check wait statistics → What is SQL Server waiting for?           ║
║  □ Check top queries → Query Store / dm_exec_query_stats             ║
║  □ Check Activity Monitor → Any blocking?                            ║
║  □ Check error log → Any resource warnings?                          ║
║                                                                        ║
║  STEP 2: ANALYZE THE QUERY                                           ║
║  □ Get the actual execution plan (Ctrl+M, then F5)                   ║
║  □ Check STATISTICS IO → How many logical reads?                     ║
║  □ Compare estimated vs actual rows → Stale statistics?              ║
║  □ Look for warnings (⚠️) in the plan                               ║
║  □ Look for Key Lookups, Table Scans, Sort spills                    ║
║                                                                        ║
║  STEP 3: FIX (in order of impact)                                    ║
║  □ Add missing indexes (check dm_db_missing_index_details)           ║
║  □ Create covering indexes (INCLUDE columns to eliminate lookups)    ║
║  □ Rewrite non-SARGable predicates                                   ║
║  □ Update statistics (WITH FULLSCAN for critical tables)             ║
║  □ Address parameter sniffing (RECOMPILE, OPTIMIZE FOR, Query Store) ║
║  □ Reduce data volume (filter earlier, SELECT only needed columns)   ║
║                                                                        ║
║  STEP 4: VERIFY                                                      ║
║  □ Re-check execution plan → Did it improve?                        ║
║  □ Re-check STATISTICS IO → Fewer logical reads?                     ║
║  □ Monitor Query Store → Performance sustained?                      ║
║  □ Test under load → Still fast with concurrent users?              ║
║                                                                        ║
╚════════════════════════════════════════════════════════════════════════╝
```

---

## 🧪 10. Real-World Performance Tuning Case Study

### The Problem

```
"Our Order History page takes 45 seconds to load!"

Stored Procedure: dbo.GetOrderHistory @CustomerID INT, @StartDate DATE
Execution Count: 5,000/day
Average Duration: 45 seconds 💀
```

### The Investigation

```sql
-- Step 1: Get the actual execution plan
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

EXEC dbo.GetOrderHistory @CustomerID = 42, @StartDate = '2025-01-01';

-- STATISTICS IO output:
-- Table 'Orders'.    Logical reads: 425,847  ← HUGE!
-- Table 'OrderItems'. Logical reads: 892,156 ← MASSIVE!
-- CPU time = 38,429 ms.  Elapsed time = 45,102 ms.

-- Step 2: Execution plan reveals:
-- ✗ Clustered Index SCAN on Orders (no useful index)
-- ✗ Key Lookup for each row (missing INCLUDE columns)
-- ✗ Estimated rows: 10, Actual rows: 150,000 (stale stats!)
-- ✗ Sort spill to tempdb (not enough memory grant)
```

### The Fix

```sql
-- Fix 1: Create a proper index
CREATE NONCLUSTERED INDEX IX_Orders_CustomerDate
ON Orders (CustomerID, OrderDate)
INCLUDE (OrderTotal, Status, ShippingAddress);

-- Fix 2: Update statistics
UPDATE STATISTICS Orders WITH FULLSCAN;

-- Fix 3: Rewrite the procedure (was using non-SARGable WHERE)
-- BEFORE (bad):
-- WHERE YEAR(OrderDate) = YEAR(@StartDate)
-- AFTER (good):
ALTER PROCEDURE dbo.GetOrderHistory
    @CustomerID INT, @StartDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT o.OrderID, o.OrderDate, o.OrderTotal, o.Status,
           oi.ProductName, oi.Quantity, oi.UnitPrice
    FROM Orders o
    JOIN OrderItems oi ON o.OrderID = oi.OrderID
    WHERE o.CustomerID = @CustomerID
      AND o.OrderDate >= @StartDate             -- ✅ SARGable!
      AND o.OrderDate < DATEADD(YEAR, 1, @StartDate)
    ORDER BY o.OrderDate DESC
    OPTION (RECOMPILE);  -- Fix parameter sniffing
END;
```

### The Result

```
BEFORE TUNING                    AFTER TUNING
─────────────────               ─────────────────
Duration:  45 sec               Duration:  0.2 sec     ← 225x faster!
Logical reads: 1.3M            Logical reads: 847      ← 1,500x fewer!
CPU time: 38 sec                CPU time: 50 ms         ← 760x less CPU!
Plan: Clustered Scan            Plan: Index Seek         
      + Key Lookups                   (covering!)
      + Sort Spill                    No spills
```

> 💡 **Key Lesson:** Most performance problems are solved by **proper indexing + SARGable queries + fresh statistics**. You rarely need hardware upgrades.

---

## 🧠 Quick Recall — Chapter Summary

| Topic | One-Line Summary |
|-------|-----------------|
| Execution Plans | Read right-to-left; look for scans, key lookups, warnings, row estimate mismatches |
| STATISTICS IO | `logical reads` is your key metric — minimize it |
| Query Store | Built-in performance history — find regressions, force good plans |
| DMVs | Real-time views into SQL Server internals (waits, indexes, memory, I/O) |
| Missing Indexes | `dm_db_missing_index_details` — SQL Server tells you what it needs |
| Unused Indexes | `dm_db_index_usage_stats` — find and remove waste |
| Covering Indexes | Add INCLUDE columns to eliminate Key Lookups |
| SARGable | Never put functions on columns in WHERE — kills index usage |
| Statistics | Stale stats = wrong row estimates = bad plans. Update regularly |
| Parameter Sniffing | Cached plan from first call may be terrible for subsequent calls |
| Wait Statistics | Tells you WHERE the bottleneck is (I/O, CPU, locks, memory) |
| MAXDOP | Set to cores per NUMA node (max 8), cost threshold 50+ |
| RCSI | Read Committed Snapshot Isolation — eliminates reader-writer blocking |

---

## ❓ Self-Check Questions

1. How do you read an execution plan — left-to-right or right-to-left?
2. What does a **Key Lookup** operator mean and how do you fix it?
3. What's the difference between **estimated** and **actual** execution plans?
4. What is Query Store and why is it called a "time machine"?
5. Name 3 DMVs you'd use to find the most expensive queries.
6. What makes a query **non-SARGable**? Give 3 examples.
7. What is **parameter sniffing** and list 3 ways to fix it.
8. Your top wait type is `PAGEIOLATCH_SH` — what does it mean and what do you do?
9. Why should `max server memory` always be set? What's the rule of thumb?
10. What is RCSI and why does Azure SQL Database use it by default?

---

## 🎯 Hands-On Challenge

```sql
-- 1. Enable STATISTICS IO and TIME, run a slow query, read the output

-- 2. Get the actual execution plan for your slowest query
--    Identify: Scans? Key Lookups? Sort spills? Row estimate mismatches?

-- 3. Check dm_db_missing_index_details — any suggestions?
--    Create the top suggested index and re-run the query

-- 4. Enable Query Store on your database
--    Run a query 100 times, then find it in Query Store reports

-- 5. Find your server's top 5 wait types
--    What do they mean? What action should you take?

-- 6. Check for unused indexes in your database
--    How many are there? How much space do they waste?

-- 7. Create a deliberate parameter sniffing problem:
--    a. Create a skewed table (one value with 90% of rows)
--    b. Create a stored procedure that queries it
--    c. Call with the rare value first, then the common value
--    d. Observe the performance difference
--    e. Fix it with OPTION (RECOMPILE)

-- 8. Run the post-install checklist:
--    Is max server memory set? MAXDOP? Cost threshold?
--    Is optimize for ad hoc workloads ON?
```

---

> **Next Chapter** → [2C.5 — SSIS, SSRS, SSAS — BI Stack](./05-SQLServer-BI-Stack.md)

---

> *"Performance tuning is not about making things faster — it's about eliminating the reasons they're slow."* — Brent Ozar
