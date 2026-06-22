# 2D.4 — MySQL Performance Tuning 🔴⭐🔥

> **"A slow query is a broken query. A fast database isn't luck — it's engineering."**
> This chapter turns you from someone who writes SQL into someone who makes SQL **fly**.

---

## 🎯 What You'll Master

```
✅ EXPLAIN & EXPLAIN ANALYZE — reading MySQL's execution plan like a pro
✅ Slow Query Log — catching every bottleneck before users complain
✅ InnoDB Buffer Pool — the #1 memory lever for MySQL performance
✅ Index optimization — creating the right indexes, killing the wrong ones
✅ Query optimization patterns — rewriting queries for 100x speed
✅ Server variables that matter — the knobs that actually move the needle
✅ Profiling & diagnostics — Performance Schema, sys schema, and beyond
✅ Real-world tuning war stories & checklists
```

---

## 🧠 The Tuning Mindset — Before You Touch Anything

```
┌──────────────────────────────────────────────────────────────┐
│           THE PERFORMANCE TUNING PYRAMID                     │
│                                                              │
│                      ▲                                       │
│                     / \        Hardware (last resort)         │
│                    /   \       CPU, RAM, SSD                  │
│                   /─────\                                     │
│                  /       \     Server Config                  │
│                 /         \    my.cnf / my.ini tuning         │
│                /───────────\                                  │
│               /             \   Schema & Indexing             │
│              /               \  Right indexes, proper types   │
│             /─────────────────\                               │
│            /                   \  Query Optimization           │
│           /                     \ Fix the SQL first!           │
│          /───────────────────────\                             │
│         /    APPLICATION DESIGN    \  Caching, connection      │
│        /    (Most Impact Lives Here) \ pooling, pagination     │
│       /───────────────────────────────\                        │
│                                                              │
│  🔥 RULE: Always optimize TOP → BOTTOM (queries first!)      │
└──────────────────────────────────────────────────────────────┘
```

> 💡 **Golden Rule:** 90% of MySQL performance problems are **bad queries** or **missing indexes**. Fix those before touching server config.

---

## 🔥 1. EXPLAIN — X-Ray Vision for Your Queries

`EXPLAIN` is your **#1 weapon**. It shows you MySQL's execution plan — how it plans to find your data.

### Basic Usage

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
```

**Output:**

```
+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | orders | ALL  | NULL          | NULL | NULL    | NULL | 9823 | Using where |
+----+-------------+--------+------+---------------+------+---------+------+------+-------------+
```

**Translation:** "I'm going to scan ALL 9,823 rows to find what you want." 💀

### EXPLAIN Columns Decoded — The Complete Reference

| Column | What It Tells You | What to Look For |
|--------|-------------------|------------------|
| `id` | Query step number | Higher = executes first (subqueries) |
| `select_type` | Type of SELECT | SIMPLE = no subquery, SUBQUERY, DERIVED, UNION |
| `table` | Which table | The table being accessed |
| `type` | **⭐ Access type** | **THE MOST IMPORTANT COLUMN** (see below) |
| `possible_keys` | Indexes MySQL *could* use | If NULL → no usable indexes exist |
| `key` | Index MySQL *actually* chose | If NULL → full table scan 💀 |
| `key_len` | Bytes of index used | Longer = more of composite index used ✅ |
| `ref` | What's compared to the index | const, column name, or func |
| `rows` | Estimated rows to examine | Lower is better. Compare with actual table size |
| `filtered` | % of rows after WHERE filter | 100% = all rows match, 1% = very selective ✅ |
| `Extra` | Additional execution info | Watch for dangerous values (see below) |

### The `type` Column — Access Type Ranking (Best → Worst)

```
⭐ MEMORIZE THIS — It's the single most important thing in EXPLAIN

BEST ──────────────────────────────────────────────────── WORST

system → const → eq_ref → ref → range → index → ALL

  │        │        │       │      │        │       │
  │        │        │       │      │        │       └── 💀 FULL TABLE SCAN
  │        │        │       │      │        └────────── 😰 Full index scan
  │        │        │       │      └─────────────────── 🟡 Range scan (ok)
  │        │        │       └────────────────────────── 🟢 Non-unique index
  │        │        └────────────────────────────────── 🟢 Unique index (JOIN)
  │        └─────────────────────────────────────────── 🟢 Single row (PK/UNIQUE)
  └──────────────────────────────────────────────────── 🟢 System table (1 row)
```

**Detailed Breakdown:**

| Type | Meaning | Example | Performance |
|------|---------|---------|-------------|
| `system` | Table has exactly 1 row | System tables | 🟢 Instant |
| `const` | Matched 1 row via PK/UNIQUE | `WHERE id = 5` | 🟢 Instant |
| `eq_ref` | 1 row per join via PK/UNIQUE | PK join between tables | 🟢 Excellent |
| `ref` | Multiple rows via non-unique index | `WHERE status = 'active'` | 🟢 Good |
| `fulltext` | FULLTEXT index search | `MATCH(...) AGAINST(...)` | 🟢 Good |
| `ref_or_null` | Like `ref` + NULL check | `WHERE col = val OR col IS NULL` | 🟡 OK |
| `range` | Index range scan | `WHERE id > 100 AND id < 200` | 🟡 OK |
| `index` | Full index scan (reads every entry) | Scanning entire index | 😰 Bad |
| `ALL` | **Full table scan** | No usable index | 💀 Terrible |

### The `Extra` Column — Red Flags & Green Flags

```
🟢 GREEN FLAGS (Good):
   Using index          → Covering index! No table lookup needed
   Using index condition → Index Condition Pushdown (ICP)
   Using where; Using index → Filtered at storage engine level

🔴 RED FLAGS (Bad — Fix these!):
   Using filesort       → MySQL can't use index for ORDER BY (sorts in memory/disk)
   Using temporary      → MySQL created a temp table (GROUP BY / DISTINCT / UNION)
   Using where          → Filtering AFTER reading rows (no index for WHERE)
   Using join buffer    → No index for JOIN condition
   Select tables optimized away → Entire query resolved from index metadata
```

### EXPLAIN FORMAT=JSON — The Deep Dive

```sql
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE customer_id = 42\G
```

Gives you cost estimates, actual filtering percentages, and optimizer decisions in JSON format.

### EXPLAIN ANALYZE (MySQL 8.0.18+) — The Real Deal

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

**This actually RUNS the query** and shows:
- **Estimated cost** vs **Actual time**
- **Estimated rows** vs **Actual rows**
- Number of loops

```
-> Filter: (orders.customer_id = 42)  (cost=1012 rows=983)
     (actual time=0.123..45.678 rows=47 loops=1)
   -> Table scan on orders  (cost=1012 rows=9823)
        (actual time=0.098..38.234 rows=9823 loops=1)
```

**Translation:** "I estimated 983 rows but only found 47. I scanned all 9,823 rows to find them. This took 45ms."

> 💡 **PRO TIP:** `EXPLAIN` estimates. `EXPLAIN ANALYZE` measures. Always verify with ANALYZE when optimizing critical queries.

---

## 🔥 2. Slow Query Log — Your Performance Detective

The Slow Query Log captures every query that takes longer than a threshold. It's your **24/7 performance watchdog**.

### Enable It (Dynamically — No Restart)

```sql
-- Turn it on
SET GLOBAL slow_query_log = 'ON';

-- Queries slower than 1 second (default is 10 — way too high)
SET GLOBAL long_query_time = 1;

-- Also log queries that don't use indexes
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- Where to write the log
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- Check current settings
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
```

### Make It Permanent (my.cnf / my.ini)

```ini
[mysqld]
slow_query_log          = 1
slow_query_log_file     = /var/log/mysql/slow.log
long_query_time         = 1
log_queries_not_using_indexes = 1
min_examined_row_limit  = 100     # Skip trivial queries
```

### Reading the Slow Query Log

```
# Time: 2024-03-15T14:23:45.678901Z
# User@Host: app_user[app_user] @ webapp01 [10.0.1.50]
# Query_time: 12.456789  Lock_time: 0.000234  Rows_sent: 47  Rows_examined: 9823456
SET timestamp=1710512625;
SELECT o.*, c.name, c.email 
FROM orders o 
JOIN customers c ON o.customer_id = c.id 
WHERE o.created_at > '2024-01-01' 
ORDER BY o.amount DESC;
```

**Key Insight:** `Rows_sent: 47` but `Rows_examined: 9,823,456` — MySQL read **~10 million rows** to return 47. That's the problem.

### mysqldumpslow — Analyze the Log

```bash
# Top 10 slowest queries (sorted by total time)
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# Top queries by count (most frequent slow queries)
mysqldumpslow -s c -t 10 /var/log/mysql/slow.log

# Top queries by rows examined
mysqldumpslow -s r -t 10 /var/log/mysql/slow.log
```

**Output:**
```
Count: 1547  Time=3.42s (5290s)  Lock=0.00s (0s)  Rows=23.4 (36198), app_user[app_user]@webapp01
  SELECT * FROM orders WHERE status = 'S' AND created_at > 'S' ORDER BY amount DESC LIMIT N
```

**Translation:** This query ran 1,547 times, averaging 3.42 seconds each. Total time: 5,290 seconds (~88 minutes of database time).

> 💡 **PRO TIP:** Use **pt-query-digest** (from Percona Toolkit) for production-grade slow query analysis. It's 100x better than mysqldumpslow.

```bash
pt-query-digest /var/log/mysql/slow.log > report.txt
```

---

## 🔥 3. InnoDB Buffer Pool — The Memory That Makes or Breaks MySQL

The Buffer Pool is InnoDB's **in-memory cache** for data and indexes. It's the **single most impactful tuning parameter** in MySQL.

### How It Works

```
┌──────────────────────────────────────────────────────────┐
│                    APPLICATION                            │
│                       │                                   │
│                       ▼                                   │
│              ┌─────────────────┐                          │
│              │  MySQL Server   │                          │
│              │  (Query Parser, │                          │
│              │   Optimizer)    │                          │
│              └────────┬────────┘                          │
│                       ▼                                   │
│    ┌──────────────────────────────────────┐               │
│    │      InnoDB BUFFER POOL (RAM)        │  ← 🔥 HERE   │
│    │                                      │               │
│    │  ┌────────┐ ┌────────┐ ┌────────┐   │               │
│    │  │  Data  │ │ Index  │ │ Change │   │               │
│    │  │ Pages  │ │ Pages  │ │ Buffer │   │               │
│    │  └────────┘ └────────┘ └────────┘   │               │
│    │  ┌────────┐ ┌────────┐              │               │
│    │  │Adaptive│ │  Lock  │              │               │
│    │  │Hash Idx│ │  Info  │              │               │
│    │  └────────┘ └────────┘              │               │
│    └──────────────────┬───────────────────┘               │
│                       ▼                                   │
│              ┌─────────────────┐                          │
│              │    DISK (SSD)   │  ← Slow (avoid this!)    │
│              │  .ibd files     │                          │
│              └─────────────────┘                          │
└──────────────────────────────────────────────────────────┘

HIT  = Data found in buffer pool → microseconds ⚡
MISS = Must read from disk → milliseconds 🐢 (1000x slower)
```

### Sizing the Buffer Pool — The Golden Rule

```ini
# my.cnf — THE most important MySQL setting

# Dedicated database server:
innodb_buffer_pool_size = 12G    # ~70-80% of total RAM

# Shared server (app + DB):
innodb_buffer_pool_size = 4G     # ~50% of available RAM

# Development machine:
innodb_buffer_pool_size = 512M   # Enough for dev work
```

**Formula:**
```
Dedicated DB Server: buffer_pool = Total RAM × 0.75
Shared Server:       buffer_pool = Available RAM × 0.50
```

> ⚠️ **NEVER set it to 100% of RAM.** MySQL needs memory for connections, sort buffers, temp tables, OS, and other processes.

### Check Your Buffer Pool Hit Rate

```sql
-- Buffer pool hit ratio (should be > 99%)
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';

-- Calculate hit ratio
SELECT 
    (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100 
    AS buffer_pool_hit_ratio
FROM (
    SELECT 
        VARIABLE_VALUE AS Innodb_buffer_pool_reads 
    FROM performance_schema.global_status 
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
) reads,
(
    SELECT 
        VARIABLE_VALUE AS Innodb_buffer_pool_read_requests 
    FROM performance_schema.global_status 
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
) requests;
```

**Interpretation:**
| Hit Ratio | Verdict | Action |
|-----------|---------|--------|
| > 99.9% | 🟢 Excellent | Buffer pool is perfect |
| 99% - 99.9% | 🟡 Good | Consider increasing if RAM allows |
| 95% - 99% | 🔴 Poor | Increase buffer pool size immediately |
| < 95% | 💀 Critical | Your DB is mostly reading from disk |

### Buffer Pool Instances (Concurrency)

```ini
# Multiple buffer pool instances reduce contention on multi-core systems
# Rule: 1 instance per 1-2 GB of buffer pool
innodb_buffer_pool_instances = 8    # For 8-16 GB buffer pool
```

### Buffer Pool Dump/Load (Warm Restart)

```ini
# Dump buffer pool state on shutdown, reload on startup
# Eliminates the "cold start" problem after restart
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup  = ON
innodb_buffer_pool_dump_pct         = 75    # Dump 75% of most recently used pages
```

> 💡 **Real-World Impact:** Without this, after a MySQL restart, your buffer pool is EMPTY. Every query hits disk. Response times spike 10-100x for minutes/hours until the cache warms up. With dump/load, you're back to normal in seconds.

---

## 🔥 4. Index Optimization — The Art of the Right Index

> Indexes are covered in detail in Chapter 1.7. Here we focus on **MySQL-specific index strategies**.

### The Composite Index Rule — Left-to-Right Prefix

```sql
-- Creating a composite index
CREATE INDEX idx_orders_status_date_amount 
ON orders (status, created_at, amount);
```

**This index can be used for:**
```sql
-- ✅ Uses full index (all 3 columns)
WHERE status = 'shipped' AND created_at > '2024-01-01' AND amount > 1000

-- ✅ Uses first 2 columns
WHERE status = 'shipped' AND created_at > '2024-01-01'

-- ✅ Uses first column only
WHERE status = 'shipped'

-- ❌ CANNOT use index (skipped first column!)
WHERE created_at > '2024-01-01'

-- ❌ CANNOT use index (skipped first two columns!)
WHERE amount > 1000
```

> 💡 **The Leftmost Prefix Rule:** A composite index `(A, B, C)` can satisfy queries on `(A)`, `(A, B)`, or `(A, B, C)` — but NOT `(B)`, `(C)`, `(B, C)`, or `(A, C)` for range scans on C.

### Covering Indexes — Zero Disk Access

```sql
-- If your query only needs columns IN the index, MySQL never touches the table
CREATE INDEX idx_covering ON orders (customer_id, status, amount);

-- This query is "covered" — all data comes from the index
SELECT customer_id, status, amount 
FROM orders 
WHERE customer_id = 42 AND status = 'shipped';

-- EXPLAIN shows: "Using index" ← That's the magic!
```

### Index Selectivity — Picking the Best Columns

```sql
-- Check selectivity (higher = better for indexing)
SELECT 
    COUNT(DISTINCT status) / COUNT(*) AS status_selectivity,
    COUNT(DISTINCT customer_id) / COUNT(*) AS customer_selectivity,
    COUNT(DISTINCT email) / COUNT(*) AS email_selectivity
FROM orders;
```

**Result:**
```
+----------------------+-------------------------+---------------------+
| status_selectivity   | customer_selectivity    | email_selectivity   |
+----------------------+-------------------------+---------------------+
|               0.0004 |                  0.1250 |              0.9800 |
+----------------------+-------------------------+---------------------+
```

**Interpretation:**
- `email` selectivity = 0.98 → Almost unique → **Excellent index candidate** ✅
- `customer_id` selectivity = 0.125 → Moderate → Good for WHERE/JOIN ✅
- `status` selectivity = 0.0004 → Very low (only 4 values) → **Poor standalone index** ❌

### Finding Unused Indexes (Dead Weight)

```sql
-- Indexes that exist but are NEVER used (MySQL 8.0+)
SELECT 
    s.TABLE_SCHEMA,
    s.TABLE_NAME,
    s.INDEX_NAME,
    s.COLUMN_NAME,
    t.TABLE_ROWS
FROM information_schema.STATISTICS s
LEFT JOIN performance_schema.table_io_waits_summary_by_index_usage u
    ON s.TABLE_SCHEMA = u.OBJECT_SCHEMA
    AND s.TABLE_NAME = u.OBJECT_NAME
    AND s.INDEX_NAME = u.INDEX_NAME
JOIN information_schema.TABLES t
    ON s.TABLE_SCHEMA = t.TABLE_SCHEMA
    AND s.TABLE_NAME = t.TABLE_NAME
WHERE u.COUNT_STAR = 0
    AND s.INDEX_NAME != 'PRIMARY'
    AND s.TABLE_SCHEMA NOT IN ('mysql', 'sys', 'performance_schema', 'information_schema')
ORDER BY t.TABLE_ROWS DESC;
```

> 💡 **Why remove unused indexes?**
> - Each index slows down INSERT/UPDATE/DELETE (must update index too)
> - Wastes disk space and buffer pool memory
> - Confuses the optimizer with too many choices

### Finding Duplicate Indexes

```sql
-- Duplicate and redundant indexes
SELECT 
    TABLE_SCHEMA, TABLE_NAME,
    GROUP_CONCAT(INDEX_NAME) AS duplicate_indexes,
    GROUP_CONCAT(COLUMN_NAME ORDER BY SEQ_IN_INDEX) AS indexed_columns
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA NOT IN ('mysql', 'sys', 'performance_schema', 'information_schema')
GROUP BY TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME
HAVING COUNT(*) > 1;
```

> 💡 **Use Percona's `pt-duplicate-key-checker` for a thorough duplicate index analysis.**

### Invisible Indexes (MySQL 8.0+) — Safe Testing

```sql
-- Make an index invisible (optimizer ignores it, but it still exists)
ALTER TABLE orders ALTER INDEX idx_status INVISIBLE;

-- Test your queries — if nothing breaks, the index is safe to drop
-- ...

-- Make it visible again (if you changed your mind)
ALTER TABLE orders ALTER INDEX idx_status VISIBLE;

-- Actually drop it
DROP INDEX idx_status ON orders;
```

> 💡 **This is a GAME CHANGER.** Before MySQL 8.0, dropping an index was scary — if you were wrong, recreating it on a 100M-row table takes hours. Invisible indexes let you test safely.

---

## 🔥 5. Query Optimization Patterns — The Cookbook

### Pattern 1: Avoid SELECT * — Always

```sql
-- ❌ BAD: Fetches all columns, can't use covering index
SELECT * FROM orders WHERE customer_id = 42;

-- ✅ GOOD: Only fetches what you need
SELECT id, amount, status, created_at 
FROM orders WHERE customer_id = 42;
```

### Pattern 2: Avoid Functions on Indexed Columns

```sql
-- ❌ BAD: Function on column destroys index usage (NOT SARGable)
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
SELECT * FROM users WHERE UPPER(email) = 'JOHN@EXAMPLE.COM';

-- ✅ GOOD: Rewrite to keep column naked
SELECT * FROM orders 
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- ✅ For case-insensitive search, use proper collation or generated column
ALTER TABLE users ADD email_lower VARCHAR(255) 
    GENERATED ALWAYS AS (LOWER(email)) STORED;
CREATE INDEX idx_email_lower ON users(email_lower);
```

> ⭐ **SARGable** = **S**earch **ARG**ument **ABLE**. A WHERE clause is SARGable if the optimizer can use an index to speed it up. Functions on columns kill SARGability.

### Pattern 3: Pagination — Don't Use OFFSET on Large Tables

```sql
-- ❌ TERRIBLE at scale: OFFSET 1000000 still reads 1,000,000 rows then skips them
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 1000000;

-- ✅ KEYSET PAGINATION: Jump directly using WHERE
SELECT * FROM products 
WHERE id > 1000000           -- last seen id from previous page
ORDER BY id 
LIMIT 20;
```

**Performance Comparison:**
| Method | Page 1 | Page 1,000 | Page 100,000 |
|--------|--------|------------|--------------|
| OFFSET | 2ms | 200ms | 15,000ms 💀 |
| Keyset | 2ms | 2ms | 2ms ✅ |

### Pattern 4: EXISTS vs IN for Subqueries

```sql
-- For LARGE outer table + SMALL subquery result → IN is fine
SELECT * FROM orders 
WHERE customer_id IN (SELECT id FROM customers WHERE country = 'India');

-- For SMALL outer table + LARGE subquery result → EXISTS is better
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.id AND o.amount > 50000
);
```

> 💡 **Modern MySQL (8.0+) is smart** — the optimizer often rewrites IN as EXISTS internally. But for older versions, this matters a lot.

### Pattern 5: JOIN Order & Type Matters

```sql
-- ❌ Joining a huge table to a huge table without proper conditions
SELECT * FROM orders o 
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.created_at > '2024-01-01';

-- ✅ Filter early, join less data
SELECT o.id, o.amount, p.name 
FROM orders o 
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.created_at > '2024-01-01'
  AND o.status = 'delivered';
```

### Pattern 6: Batch Operations for Large Deletes/Updates

```sql
-- ❌ DANGEROUS: Locks millions of rows, fills undo log, may crash replication
DELETE FROM logs WHERE created_at < '2023-01-01';

-- ✅ SAFE: Delete in batches
DELIMITER $$
CREATE PROCEDURE batch_delete_old_logs()
BEGIN
    DECLARE rows_affected INT DEFAULT 1;
    WHILE rows_affected > 0 DO
        DELETE FROM logs 
        WHERE created_at < '2023-01-01' 
        LIMIT 10000;
        SET rows_affected = ROW_COUNT();
        -- Small sleep to let replicas catch up
        DO SLEEP(0.5);
    END WHILE;
END$$
DELIMITER ;

CALL batch_delete_old_logs();
```

---

## 🔥 6. Server Variables That Matter — The Tuning Knobs

### The Top 20 Variables You Must Know

```ini
# ============================================================
# my.cnf — Production-Ready Configuration Template
# ============================================================

[mysqld]

# === INNODB (The Engine) ===
innodb_buffer_pool_size        = 12G       # 70-80% of RAM (dedicated server)
innodb_buffer_pool_instances   = 8         # 1 per 1-2GB of buffer pool
innodb_log_file_size           = 2G        # Redo log size (larger = better write perf)
innodb_log_buffer_size         = 64M       # Redo log buffer in memory
innodb_flush_log_at_trx_commit = 1         # 1=ACID safe, 2=faster but risks 1sec of data
innodb_flush_method            = O_DIRECT  # Avoid double-buffering with OS cache
innodb_file_per_table          = ON        # Each table gets its own .ibd file
innodb_io_capacity             = 2000      # IOPS for background tasks (SSD: 2000+, HDD: 200)
innodb_io_capacity_max         = 4000      # Max IOPS for urgent flushing
innodb_read_io_threads         = 8         # Parallel read threads
innodb_write_io_threads        = 8         # Parallel write threads

# === CONNECTIONS ===
max_connections                = 500       # Match app pool size + overhead
wait_timeout                   = 300       # Kill idle connections after 5 min
interactive_timeout            = 300       # Same for interactive sessions

# === MEMORY PER-CONNECTION ===
sort_buffer_size               = 4M        # Per-connection sort memory
join_buffer_size               = 4M        # Per-connection join buffer (no index)
read_buffer_size               = 2M        # Sequential scan buffer
read_rnd_buffer_size           = 4M        # Random read buffer
tmp_table_size                 = 64M       # Max in-memory temp table
max_heap_table_size            = 64M       # Must match tmp_table_size

# === QUERY CACHE (Removed in MySQL 8.0!) ===
# query_cache_type = 0         # DISABLED — it was a bottleneck, not a feature
# Use ProxySQL or application-level caching instead

# === LOGGING ===
slow_query_log                 = 1
long_query_time                = 1
log_queries_not_using_indexes  = 1

# === BINARY LOG (for replication) ===
binlog_format                  = ROW       # ROW-based replication (safest)
expire_logs_days               = 7         # Auto-purge old binlogs
sync_binlog                    = 1         # Flush binlog on every commit (ACID safe)
```

### The `innodb_flush_log_at_trx_commit` Deep Dive

This single variable controls the **durability vs speed** trade-off:

```
┌────────────────────────────────────────────────────────────────┐
│  innodb_flush_log_at_trx_commit — The ACID Knob               │
│                                                                │
│  Value = 1 (DEFAULT — Full ACID)                               │
│  ├── Flush redo log to disk on EVERY commit                    │
│  ├── Zero data loss on crash                                   │
│  └── Slowest (disk I/O on every transaction)                   │
│                                                                │
│  Value = 2 (Compromise)                                        │
│  ├── Write to OS buffer on every commit, flush every 1 second  │
│  ├── Lose up to 1 second of data on OS crash                   │
│  └── Good balance (safe from MySQL crash, not OS crash)        │
│                                                                │
│  Value = 0 (Fastest — Dangerous!)                              │
│  ├── Write + flush every 1 second                              │
│  ├── Lose up to 1 second of data on ANY crash                  │
│  └── Only for non-critical data (logs, analytics, dev)         │
│                                                                │
│  🔥 Production: ALWAYS use 1 (or 2 if you accept the risk)    │
└────────────────────────────────────────────────────────────────┘
```

---

## 🔥 7. Performance Schema & sys Schema — MySQL's Built-in APM

### Performance Schema — The Data Source

MySQL's `performance_schema` database captures detailed performance metrics at runtime.

```sql
-- Is it enabled?
SHOW VARIABLES LIKE 'performance_schema';

-- Top 10 queries by total execution time
SELECT 
    DIGEST_TEXT AS query,
    COUNT_STAR AS exec_count,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_time_sec,
    ROUND(AVG_TIMER_WAIT / 1000000000000, 4) AS avg_time_sec,
    SUM_ROWS_EXAMINED AS rows_examined,
    SUM_ROWS_SENT AS rows_sent
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10\G
```

### sys Schema — The Human-Readable Layer (MySQL 5.7+)

The `sys` schema wraps `performance_schema` in easy-to-read views.

```sql
-- 🔥 Top queries consuming the most time
SELECT * FROM sys.statement_analysis LIMIT 10\G

-- 🔥 Statements with full table scans
SELECT * FROM sys.statements_with_full_table_scans LIMIT 10\G

-- 🔥 Tables with the most I/O
SELECT * FROM sys.io_global_by_file_by_bytes LIMIT 10;

-- 🔥 Unused indexes (candidates for removal)
SELECT * FROM sys.schema_unused_indexes;

-- 🔥 Redundant indexes
SELECT * FROM sys.schema_redundant_indexes;

-- 🔥 Tables without primary keys (bad for InnoDB!)
SELECT * FROM sys.schema_tables_with_full_table_scans;

-- 🔥 Who's connected and what they're doing
SELECT * FROM sys.processlist;

-- 🔥 Memory usage breakdown
SELECT * FROM sys.memory_global_total;
SELECT * FROM sys.memory_global_by_current_bytes LIMIT 10;

-- 🔥 Wait event analysis (what's MySQL waiting on?)
SELECT * FROM sys.waits_global_by_latency LIMIT 10;
```

> 💡 **PRO TIP:** The `sys` schema is your **first stop** for any performance investigation. Before diving into EXPLAIN, check `sys.statement_analysis` to find the worst queries.

---

## 🔥 8. InnoDB Internals — Understanding What Happens Under the Hood

### The Write Path

```
Application → INSERT INTO orders (...) VALUES (...)
                         │
                         ▼
              ┌──────────────────┐
              │   Buffer Pool    │  1. Write to buffer pool (memory)
              │   (dirty page)   │
              └────────┬─────────┘
                       │
              ┌────────▼─────────┐
              │   Redo Log       │  2. Write to redo log (WAL) — SEQUENTIAL I/O ⚡
              │   (ib_logfile0)  │     This is what makes crash recovery possible
              └────────┬─────────┘
                       │
                       │  (Background thread — later)
                       ▼
              ┌──────────────────┐
              │   Data File      │  3. Eventually flush dirty page to .ibd file
              │   (.ibd)         │     Called "checkpointing"
              └──────────────────┘

💡 KEY INSIGHT: MySQL NEVER writes data directly to the table file on commit.
   It writes to the redo log (sequential, fast) and flushes to disk later.
   This is called Write-Ahead Logging (WAL).
```

### Change Buffer (Insert Buffer)

```
When you INSERT into a table with secondary indexes:

WITHOUT Change Buffer:
  Write data page → Write EACH secondary index page → RANDOM I/O 💀

WITH Change Buffer:
  Write data page → Buffer index changes in memory → Merge later → SEQUENTIAL I/O ✅

# Control what operations use the change buffer
innodb_change_buffer_max_size = 25    # % of buffer pool (default 25%)
innodb_change_buffering = all          # all, none, inserts, deletes, changes, purges
```

### Adaptive Hash Index

```sql
-- InnoDB automatically builds hash indexes for frequently accessed pages
-- Usually helpful, but can cause contention on high-concurrency workloads

-- Check if it's helping or hurting
SHOW ENGINE INNODB STATUS\G
-- Look for "Hash table size" and "hash searches/s" vs "non-hash searches/s"

-- Disable if causing contention (benchmark first!)
SET GLOBAL innodb_adaptive_hash_index = OFF;
```

---

## 🔥 9. Real-World Tuning Checklist

### Before You Ship to Production

```
┌──────────────────────────────────────────────────────────────┐
│             MYSQL PERFORMANCE CHECKLIST                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  QUERIES                                                     │
│  □ All critical queries tested with EXPLAIN ANALYZE          │
│  □ No full table scans on large tables (type != ALL)         │
│  □ No "Using filesort" on large result sets                  │
│  □ No "Using temporary" on frequently-run queries            │
│  □ No SELECT * in application code                           │
│  □ Pagination uses keyset, not OFFSET                        │
│  □ No functions on indexed columns in WHERE clauses          │
│  □ Batch operations for large DELETE/UPDATE                  │
│                                                              │
│  INDEXES                                                     │
│  □ Every table has a PRIMARY KEY (InnoDB requirement!)       │
│  □ All foreign keys are indexed                              │
│  □ Composite indexes follow query access patterns            │
│  □ Covering indexes for hot queries                          │
│  □ No duplicate or redundant indexes                         │
│  □ No unused indexes (check sys.schema_unused_indexes)       │
│                                                              │
│  SERVER CONFIG                                               │
│  □ innodb_buffer_pool_size = 70-80% of RAM                   │
│  □ innodb_log_file_size ≥ 1-2 GB                             │
│  □ innodb_flush_log_at_trx_commit = 1 (or justified 2)      │
│  □ innodb_flush_method = O_DIRECT                            │
│  □ sync_binlog = 1 (if replication enabled)                  │
│  □ Slow query log enabled (long_query_time ≤ 1)             │
│  □ Buffer pool dump/load enabled for warm restarts           │
│                                                              │
│  MONITORING                                                  │
│  □ Buffer pool hit ratio > 99%                               │
│  □ Threads_connected vs max_connections monitored            │
│  □ Performance Schema enabled                                │
│  □ sys schema queries in monitoring dashboards               │
│  □ Alerting on slow query count spikes                       │
│                                                              │
│  SCHEMA                                                      │
│  □ Use smallest appropriate data types                       │
│  □ VARCHAR over CHAR for variable-length strings             │
│  □ INT UNSIGNED for IDs (or BIGINT if > 4 billion rows)      │
│  □ DATETIME(3) or TIMESTAMP(3) for millisecond precision     │
│  □ Avoid TEXT/BLOB in frequently-queried tables              │
│  □ Use ENUM only for truly fixed sets (2-5 values)           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 🧪 10. Hands-On Lab — Find & Fix the Slow Query

### The Setup

```sql
-- Create a large test table (1 million rows)
CREATE TABLE test_orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    product_name VARCHAR(200),
    amount DECIMAL(10,2),
    status ENUM('pending', 'shipped', 'delivered', 'cancelled'),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    notes TEXT
);

-- Insert 1 million rows (use a procedure or LOAD DATA)
-- ... (insert procedure here)

-- NO indexes yet (except PRIMARY KEY)
```

### The Problem Query

```sql
-- "Show me all delivered orders over $500 from the last 90 days, sorted by amount"
EXPLAIN ANALYZE
SELECT customer_id, product_name, amount, created_at
FROM test_orders
WHERE status = 'delivered'
  AND amount > 500
  AND created_at > DATE_SUB(NOW(), INTERVAL 90 DAY)
ORDER BY amount DESC
LIMIT 50;
```

**Before optimization:** Full table scan, 1M rows, Using filesort, ~8 seconds

### The Fix — Step by Step

```sql
-- Step 1: Add a composite index following the query pattern
-- Equality columns first → Range columns next → ORDER BY columns last
CREATE INDEX idx_status_created_amount 
ON test_orders (status, created_at, amount);

-- Step 2: Rewrite to SELECT only needed columns (covering index if possible)
-- Step 3: EXPLAIN ANALYZE again
EXPLAIN ANALYZE
SELECT customer_id, product_name, amount, created_at
FROM test_orders
WHERE status = 'delivered'
  AND created_at > DATE_SUB(NOW(), INTERVAL 90 DAY)
  AND amount > 500
ORDER BY amount DESC
LIMIT 50;
```

**After optimization:** Index range scan, ~2,000 rows examined, ~15ms ✅

> 💡 **500x improvement.** This is typical when you add the right index. That's why indexing is Chapter 1 of performance tuning.

---

## 🧠 Quick Reference — EXPLAIN Output Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│  EXPLAIN CHEAT SHEET                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  type = const/eq_ref   →  Perfect. Single row lookup.          │
│  type = ref            →  Good. Multiple rows via index.        │
│  type = range          →  OK. Scanning an index range.          │
│  type = index          →  Bad. Scanning ENTIRE index.           │
│  type = ALL            →  Terrible. Full table scan.            │
│                                                                 │
│  key = NULL            →  No index used! Fix this.              │
│  rows = (huge number)  →  Too many rows examined. Add index.    │
│                                                                 │
│  Extra: Using index    →  Covering index. Best case!            │
│  Extra: Using filesort →  Can't sort via index. Check ORDER BY. │
│  Extra: Using temporary→  Temp table needed. Expensive.         │
│                                                                 │
│  filtered < 20%        →  Index is not selective enough.        │
│  rows ≫ actual rows    →  Statistics are stale. Run ANALYZE.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Key Takeaways

```
1. EXPLAIN everything before it hits production
2. Slow query log is your 24/7 detective — enable it from day 1
3. Buffer pool is THE #1 knob — size it right
4. Right index > More indexes — quality over quantity
5. SARGable queries only — keep columns naked in WHERE
6. Keyset pagination — forget OFFSET exists for large tables
7. sys schema is your monitoring Swiss Army Knife
8. Batch large operations — never do 1M row deletes in one shot
9. Invisible indexes (8.0+) — test before you drop
10. Measure, don't guess — EXPLAIN ANALYZE > gut feeling
```

---

> **Next:** [2D.5 — MySQL Replication & Clustering](./05-MySQL-Replication.md) — How to scale MySQL horizontally and build fault-tolerant architectures.
