# 2E.5 — PostgreSQL Performance Tuning 🔴⭐🔥

> **"A slow query isn't a database problem — it's a conversation you haven't had with the query planner yet."**
> Master PostgreSQL performance and you'll turn 30-second queries into 30-millisecond ones.

---

## 🎯 What You'll Master

```
✅ EXPLAIN & EXPLAIN ANALYZE — reading execution plans like a pro
✅ pg_stat_statements — finding your worst queries automatically
✅ Index types deep dive: B-tree, Hash, GIN, GiST, SP-GiST, BRIN
✅ Partial, Covering, Expression indexes — surgical precision
✅ Table Partitioning — handling billions of rows
✅ VACUUM & Autovacuum — PostgreSQL's garbage collector
✅ postgresql.conf tuning — memory, workers, WAL settings
✅ Connection pooling with PgBouncer
✅ Real-world performance patterns from production systems
```

---

## 🧠 The Performance Mindset

Before touching a single setting, understand this:

```
┌──────────────────────────────────────────────────────────────┐
│           PostgreSQL Performance Pyramid                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│                    ▲  Hardware                                │
│                   ▲▲▲  (Last resort)                         │
│                  ▲▲▲▲▲                                       │
│                 ▲▲▲▲▲▲▲  Configuration                      │
│                ▲▲▲▲▲▲▲▲▲  (postgresql.conf)                 │
│               ▲▲▲▲▲▲▲▲▲▲▲                                   │
│              ▲▲▲▲▲▲▲▲▲▲▲▲▲  Schema Design                  │
│             ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲  (Tables, Data Types)          │
│            ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲                               │
│           ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲  Indexes                    │
│          ▲▲▲▲▲▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅  (Biggest Impact)         │
│         ▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅                         │
│        ▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅  QUERY DESIGN          │
│       ▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅  (Start Here!)       │
│                                                              │
│  FIX FROM BOTTOM → UP. 80% of problems are bad queries.     │
└──────────────────────────────────────────────────────────────┘
```

> 💡 **The #1 Rule:** Don't throw hardware at a bad query. A properly indexed query on a $20/month server will beat an unindexed query on a $2000/month server. Every. Single. Time.

---

## 🔍 EXPLAIN & EXPLAIN ANALYZE — Your X-Ray Machine

### The Basics

```sql
-- Show the PLAN (doesn't execute the query)
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Show the plan AND execute it (real timings)
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;

-- The ultimate diagnostic combo
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 42;
```

| Command | Executes? | Shows Timing? | Shows I/O? | Use When |
|---------|-----------|---------------|------------|----------|
| `EXPLAIN` | ❌ No | ❌ Estimated only | ❌ | Quick plan check |
| `EXPLAIN ANALYZE` | ✅ Yes | ✅ Actual timing | ❌ | Performance debugging |
| `EXPLAIN (ANALYZE, BUFFERS)` | ✅ Yes | ✅ Actual timing | ✅ Buffer hits/reads | Deep investigation |

> ⚠️ **Warning:** `EXPLAIN ANALYZE` **actually runs the query**. If it's a `DELETE FROM users`, it will delete rows! Wrap dangerous queries:
> ```sql
> BEGIN;
> EXPLAIN ANALYZE DELETE FROM users WHERE id < 100;
> ROLLBACK;  -- Nothing actually deleted
> ```

---

### Reading an Execution Plan

```sql
EXPLAIN ANALYZE
SELECT o.id, c.name, o.amount
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.amount > 10000
ORDER BY o.amount DESC;
```

```
Sort  (cost=245.71..247.14 rows=571 width=52) (actual time=3.124..3.201 rows=583 loops=1)
  Sort Key: o.amount DESC
  Sort Method: quicksort  Memory: 72kB
  ->  Hash Join  (cost=22.50..219.41 rows=571 width=52) (actual time=0.521..2.435 rows=583 loops=1)
        Hash Cond: (o.customer_id = c.id)
        ->  Seq Scan on orders o  (cost=0.00..180.00 rows=571 width=16) (actual time=0.015..1.234 rows=583 loops=1)
              Filter: (amount > 10000)
              Rows Removed by Filter: 9417
        ->  Hash  (cost=15.00..15.00 rows=600 width=40) (actual time=0.432..0.433 rows=600 loops=1)
              Buckets: 1024  Batches: 1  Memory Usage: 42kB
              ->  Seq Scan on customers c  (cost=0.00..15.00 rows=600 width=40) (actual time=0.005..0.198 rows=600 loops=1)
Planning Time: 0.285 ms
Execution Time: 3.312 ms
```

### Decoding Each Line

```
┌─────────────────────────────────────────────────────────────────────┐
│  Node Type    (cost=startup..total  rows=estimated  width=bytes)   │
│               (actual time=start..end  rows=actual  loops=N)       │
└─────────────────────────────────────────────────────────────────────┘

cost       → Planner's estimated cost (arbitrary units, NOT milliseconds)
rows       → Estimated vs actual row count
width      → Average row size in bytes
actual time → Real wall-clock time in milliseconds
loops      → How many times this node was executed
```

### The Nodes You MUST Know

| Node Type | What It Does | Good or Bad? |
|-----------|-------------|--------------|
| **Seq Scan** | Reads every row in the table | 🔴 Bad on large tables (means no useful index) |
| **Index Scan** | Uses index to find rows, then fetches from table | 🟢 Great for selective queries |
| **Index Only Scan** | Answers entirely from index (no table access) | 🟢🟢 The best — a "covering index" |
| **Bitmap Index Scan** | Builds a bitmap of matching rows from index | 🟡 Good for moderate selectivity |
| **Bitmap Heap Scan** | Fetches rows identified by bitmap | 🟡 Paired with Bitmap Index Scan |
| **Nested Loop** | For each outer row, scan inner table | 🟢 Great for small datasets or indexed inner |
| **Hash Join** | Build hash table from one side, probe with other | 🟢 Great for equi-joins on larger sets |
| **Merge Join** | Merge two sorted inputs | 🟢 Great when both sides are pre-sorted |
| **Sort** | Sort rows (quicksort/external sort) | 🟡 Needed for ORDER BY, check memory usage |
| **Aggregate** | GROUP BY / COUNT / SUM etc. | Neutral — check input size |

---

### 🚨 Red Flags in Execution Plans

```
🚨 RED FLAG #1: Seq Scan on a large table with a WHERE clause
   → Missing index. Create one.

🚨 RED FLAG #2: Estimated rows ≠ Actual rows (off by 10x+)
   → Stale statistics. Run ANALYZE.

🚨 RED FLAG #3: "Sort Method: external merge  Disk: 45MB"
   → work_mem too low. Increase it or reduce result set.

🚨 RED FLAG #4: Nested Loop with Seq Scan on inner table
   → Missing index on the join column of inner table.

🚨 RED FLAG #5: "Rows Removed by Filter: 9,999,000" (out of 10M)
   → You're scanning 10M rows to get 1000. Need a better index.

🚨 RED FLAG #6: loops=10000 on an inner node
   → That node runs 10,000 times. Even 1ms × 10,000 = 10 seconds.
```

---

### EXPLAIN Formats

```sql
-- Text (default) — best for quick reading
EXPLAIN (FORMAT TEXT) SELECT ...;

-- JSON — best for programmatic analysis & tools
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;

-- YAML — human-readable structured format
EXPLAIN (ANALYZE, FORMAT YAML) SELECT ...;
```

> 💡 **Pro Tip:** Paste JSON plans into [explain.dalibo.com](https://explain.dalibo.com/) or [explain.depesz.com](https://explain.depesz.com/) for gorgeous visual analysis.

---

## 📊 pg_stat_statements — Find Your Worst Queries

This is the **#1 most important extension** in PostgreSQL performance tuning. It tracks every query and tells you which ones consume the most time, I/O, and resources.

### Setup

```sql
-- Step 1: Enable the extension (postgresql.conf)
-- shared_preload_libraries = 'pg_stat_statements'
-- Then restart PostgreSQL

-- Step 2: Create the extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### Finding Your Top Offenders

```sql
-- 🔥 Top 10 Slowest Queries by Total Time
SELECT 
    round(total_exec_time::numeric, 2) AS total_time_ms,
    calls,
    round(mean_exec_time::numeric, 2) AS avg_time_ms,
    round((100 * total_exec_time / 
        SUM(total_exec_time) OVER ())::numeric, 2) AS pct_of_total,
    LEFT(query, 80) AS query_preview
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

```
 total_time_ms | calls  | avg_time_ms | pct_of_total | query_preview
---------------+--------+-------------+--------------+------------------------------------
   892341.23   | 145200 |    6.15     |    34.21     | SELECT o.*, c.name FROM orders o ...
   456123.89   |  89400 |    5.10     |    17.49     | UPDATE inventory SET stock = stock...
   234567.12   | 312000 |    0.75     |     8.99     | SELECT * FROM products WHERE categ...
```

> 💡 **The query using 34.21% of your total database time is your #1 optimization target.** Fix that one query and you improve your entire system by 34%.

### Other Killer Queries

```sql
-- 🔥 Most Frequently Called Queries
SELECT calls, query 
FROM pg_stat_statements 
ORDER BY calls DESC LIMIT 10;

-- 🔥 Queries with Worst I/O (buffer reads from disk)
SELECT 
    shared_blks_read AS disk_reads,
    shared_blks_hit AS cache_hits,
    round(100.0 * shared_blks_hit / 
        NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS cache_hit_ratio,
    query
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 10;

-- 🔥 Reset statistics (do this after fixing queries to get fresh data)
SELECT pg_stat_statements_reset();
```

---

## 🗂️ Index Types — Choose Your Weapon

PostgreSQL has the **richest index ecosystem** of any relational database. Choosing the right index type is an art.

### The Index Arsenal

```
┌────────────────────────────────────────────────────────────────┐
│                 PostgreSQL Index Types                          │
├──────────┬────────────────────────────────────────────────────┤
│ B-tree   │ Default. Equality & range queries. 95% of cases.  │
│ Hash     │ Equality only. Slightly faster for = comparisons. │
│ GIN      │ Generalized Inverted Index. Arrays, JSONB, FTS.   │
│ GiST     │ Generalized Search Tree. Geometry, ranges, FTS.   │
│ SP-GiST  │ Space-Partitioned GiST. Points, phone numbers.   │
│ BRIN     │ Block Range Index. Huge tables, naturally ordered. │
└──────────┴────────────────────────────────────────────────────┘
```

---

### 🌳 B-tree — The Workhorse (Default)

```sql
-- Created by default
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Supports these operators:
-- <   <=   =   >=   >   BETWEEN   IN   IS NULL   IS NOT NULL
```

```
How B-tree works (simplified):

                    [50]
                   /    \
              [20,35]   [70,85]
              / | \      / | \
         [10] [25] [40] [60] [75] [90]
          ↓    ↓    ↓    ↓    ↓    ↓
        rows  rows rows rows rows rows

→ Looking for customer_id = 75?
  50 → go right → 70,85 → between → leaf [75] → found!
  Only 3 nodes checked instead of scanning millions of rows.
```

**When to use:** Almost always. This is your default choice.

**Multi-column B-tree:**
```sql
CREATE INDEX idx_orders_cust_date ON orders(customer_id, order_date);

-- ✅ Works for: WHERE customer_id = 5
-- ✅ Works for: WHERE customer_id = 5 AND order_date > '2024-01-01'
-- ❌ Does NOT work for: WHERE order_date > '2024-01-01' (skips first column)
```

> 💡 **The Left-Prefix Rule:** A multi-column index on `(A, B, C)` can serve queries on `(A)`, `(A, B)`, or `(A, B, C)` — but NOT `(B)`, `(C)`, or `(B, C)`. Think of it like a phone book sorted by Last Name, First Name — you can't search by First Name alone.

---

### #️⃣ Hash Index

```sql
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- Only supports: =
-- Does NOT support: <, >, BETWEEN, ORDER BY, IS NULL
```

**When to use:** Only when you ALWAYS do exact equality lookups and never need range queries. Rare in practice. B-tree handles `=` almost as fast.

> 💡 **History:** Before PostgreSQL 10, Hash indexes weren't WAL-logged (crash-unsafe!). Since PG 10, they're safe. But B-tree is still preferred in 99% of cases.

---

### 📦 GIN — Generalized Inverted Index

GIN is PostgreSQL's **secret weapon** for complex data types.

```sql
-- For JSONB queries
CREATE INDEX idx_products_attrs ON products USING gin(attributes);

-- For array containment
CREATE INDEX idx_posts_tags ON posts USING gin(tags);

-- For full-text search
CREATE INDEX idx_articles_fts ON articles USING gin(to_tsvector('english', body));
```

```
How GIN works (inverted index):

Document 1: "PostgreSQL is fast"
Document 2: "PostgreSQL is powerful"  
Document 3: "MySQL is fast"

GIN Index:
  "fast"       → {Doc 1, Doc 3}
  "is"         → {Doc 1, Doc 2, Doc 3}
  "mysql"      → {Doc 3}
  "postgresql" → {Doc 1, Doc 2}
  "powerful"   → {Doc 2}

Query: WHERE body @@ 'postgresql & fast'
→ Intersect {Doc 1, Doc 2} ∩ {Doc 1, Doc 3} = {Doc 1}  ✅
```

**Supports operators:**
```sql
@>    -- Contains (JSONB, arrays)
<@    -- Is contained by
?     -- Key exists (JSONB)
?|    -- Any key exists
?&    -- All keys exist
@@    -- Full-text search match
@@@   -- Full-text search match (older syntax)
```

**JSONB + GIN Examples:**
```sql
-- Table with JSONB column
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    attributes JSONB  -- {"color": "red", "size": "L", "tags": ["sale", "new"]}
);

CREATE INDEX idx_prod_attrs ON products USING gin(attributes);

-- These queries now use the GIN index:
SELECT * FROM products WHERE attributes @> '{"color": "red"}';
SELECT * FROM products WHERE attributes ? 'size';
SELECT * FROM products WHERE attributes @> '{"tags": ["sale"]}';
```

> 💡 **GIN vs GiST for Full-Text Search:**
> - **GIN**: Faster reads, slower writes, larger index. **Best for read-heavy workloads.**
> - **GiST**: Slower reads, faster writes, smaller index. **Best for write-heavy workloads.**

---

### 🌍 GiST — Generalized Search Tree

```sql
-- For geometric/spatial data (PostGIS)
CREATE INDEX idx_locations_geom ON locations USING gist(geom);

-- For range types
CREATE INDEX idx_bookings_period ON bookings USING gist(date_range);

-- For full-text search (alternative to GIN)
CREATE INDEX idx_articles_fts ON articles USING gist(to_tsvector('english', body));

-- For inet/cidr (IP address ranges)
CREATE INDEX idx_access_ip ON access_log USING gist(client_ip inet_ops);
```

**Supports operators:**
```sql
<<    -- Strictly left of
>>    -- Strictly right of
&&    -- Overlaps
@>    -- Contains
<@    -- Contained by
~=    -- Same as
<<|   -- Strictly below
|>>   -- Strictly above
```

**PostGIS Example:**
```sql
-- "Find all restaurants within 5km of me"
SELECT name, ST_Distance(geom, ST_MakePoint(77.209, 28.613)::geography) AS distance_m
FROM restaurants
WHERE ST_DWithin(
    geom, 
    ST_MakePoint(77.209, 28.613)::geography, 
    5000  -- 5km radius
)
ORDER BY distance_m;
-- With GiST index → milliseconds. Without → scans every restaurant on Earth.
```

---

### 🧱 BRIN — Block Range Index (Big Data's Best Friend)

BRIN stores **min/max values per block of table pages**. Tiny index, huge tables.

```sql
-- Perfect for time-series data that's inserted in chronological order
CREATE INDEX idx_logs_created ON logs USING brin(created_at);
```

```
How BRIN works:

Table pages (each page = 8KB, holds ~100 rows):

Page 1-128:    created_at range [2024-01-01, 2024-01-15]
Page 129-256:  created_at range [2024-01-15, 2024-02-01]
Page 257-384:  created_at range [2024-02-01, 2024-02-14]
...

Query: WHERE created_at = '2024-01-20'
→ Skip pages 1-128 (max is Jan 15, too early)
→ Check pages 129-256 (range includes Jan 20) ✅
→ Skip pages 257+ (min is Feb 1, too late)

B-tree index on 1 billion rows: ~20 GB
BRIN index on 1 billion rows:   ~1 MB  ← 20,000x smaller!
```

**When to use:**
- ✅ Very large tables (100M+ rows)
- ✅ Data is physically ordered by the indexed column (e.g., timestamps with sequential inserts)
- ✅ You need small index size
- ❌ NOT for randomly inserted data (BRIN ranges would overlap, defeating the purpose)
- ❌ NOT when you need exact lookups on few rows

> 💡 **Pro Tip:** BRIN + Partitioning = The ultimate combo for time-series tables with billions of rows.

---

### 🎯 Special Index Techniques

#### Partial Index — Index Only What Matters

```sql
-- Only index active orders (ignore the 95% that are completed)
CREATE INDEX idx_orders_active ON orders(customer_id)
WHERE status IN ('pending', 'processing', 'shipped');

-- Only index where the column is NOT NULL
CREATE INDEX idx_users_phone ON users(phone)
WHERE phone IS NOT NULL;
```

> 💡 **Real Impact:** If 95% of orders are 'completed', a partial index is 20x smaller and 20x faster than a full index.

#### Covering Index (INCLUDE) — Index-Only Scans

```sql
-- PostgreSQL 11+
-- The query needs customer_id (for lookup) AND name, email (for output)
CREATE INDEX idx_customers_lookup ON customers(customer_id)
INCLUDE (name, email);

-- Now this query is answered entirely from the index (no table access!):
SELECT name, email FROM customers WHERE customer_id = 42;
-- Plan shows: "Index Only Scan" ← The gold standard
```

#### Expression Index — Index Computed Values

```sql
-- Index on lowercase email for case-insensitive lookups
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Now this uses the index:
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';

-- Index on extracted JSONB field
CREATE INDEX idx_products_color ON products((attributes->>'color'));

-- Index on date part
CREATE INDEX idx_orders_year ON orders(EXTRACT(YEAR FROM order_date));
```

#### Concurrent Index Creation — Zero Downtime

```sql
-- Normal CREATE INDEX locks the table for writes. On a 500M row table = minutes of downtime.

-- CONCURRENTLY builds the index without blocking writes:
CREATE INDEX CONCURRENTLY idx_orders_amount ON orders(amount);

-- ⚠️ Takes longer, uses more resources, but zero downtime.
-- ⚠️ Cannot be run inside a transaction block.
-- ⚠️ If interrupted, leaves an INVALID index — check and drop it:
SELECT indexname, indexdef FROM pg_indexes 
WHERE tablename = 'orders' AND indexname LIKE '%amount%';
```

---

### 📊 Index Selection Decision Tree

```
What type of query?
│
├── Equality (=) or Range (<, >, BETWEEN)?
│   ├── Small/Medium table → B-tree ✅
│   ├── Very large table, data naturally ordered → BRIN ✅
│   └── Only equality, never range → Hash (rare) ✅
│
├── JSONB containment (@>, ?, ?|)?
│   └── GIN ✅
│
├── Array operations (@>, &&)?
│   └── GIN ✅
│
├── Full-Text Search (@@)?
│   ├── Read-heavy → GIN ✅
│   └── Write-heavy → GiST ✅
│
├── Geometric/Spatial (PostGIS)?
│   └── GiST ✅
│
├── Range types (&&, @>, <@)?
│   └── GiST ✅
│
└── IP addresses, phone numbers (prefix matching)?
    └── SP-GiST ✅
```

---

## 🧹 VACUUM — PostgreSQL's Garbage Collector

### Why VACUUM Exists

PostgreSQL uses **MVCC** (Multi-Version Concurrency Control). When you UPDATE or DELETE a row, the old version isn't removed — it's marked as "dead" so active transactions can still see it.

```
What happens during an UPDATE:

Before: Row v1 → [id=1, name='Alice', salary=90000]  (visible)

UPDATE employees SET salary = 95000 WHERE id = 1;

After:  Row v1 → [id=1, name='Alice', salary=90000]  (dead tuple — invisible)
        Row v2 → [id=1, name='Alice', salary=95000]  (live tuple — visible)

The dead tuple wastes space until VACUUM cleans it up.
```

### Types of VACUUM

```sql
-- 1. Regular VACUUM — marks dead space as reusable (doesn't shrink file)
VACUUM orders;

-- 2. VACUUM FULL — rewrites entire table, reclaims disk space
-- ⚠️ LOCKS THE TABLE! Requires exclusive lock. Use only when desperate.
VACUUM FULL orders;

-- 3. VACUUM ANALYZE — vacuum + update statistics
VACUUM ANALYZE orders;

-- 4. ANALYZE only — update statistics without vacuuming
ANALYZE orders;
```

| Type | Locks Table? | Reclaims Disk? | Updates Stats? | Use When |
|------|-------------|----------------|----------------|----------|
| `VACUUM` | ❌ No | Reuses space only | ❌ | Routine maintenance |
| `VACUUM FULL` | ✅ Yes (exclusive) | ✅ Shrinks file | ❌ | Table is 50%+ dead tuples |
| `VACUUM ANALYZE` | ❌ No | Reuses space only | ✅ | After bulk operations |
| `ANALYZE` | ❌ No | ❌ | ✅ | After data changes |

### Autovacuum — Let PostgreSQL Handle It

Autovacuum is **enabled by default** and handles VACUUM automatically. But you MUST tune it for busy tables.

```sql
-- Check autovacuum status for your tables
SELECT 
    schemaname, relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

### Tuning Autovacuum for Hot Tables

```sql
-- Default: autovacuum triggers when dead tuples > 20% of table + 50 rows
-- For a 10M row table: triggers at 2,000,050 dead rows — WAY too late!

-- Make autovacuum more aggressive for busy tables:
ALTER TABLE orders SET (
    autovacuum_vacuum_threshold = 1000,          -- trigger after 1000 dead tuples
    autovacuum_vacuum_scale_factor = 0.01,        -- + 1% of table size (vs default 20%)
    autovacuum_analyze_threshold = 500,
    autovacuum_analyze_scale_factor = 0.005
);

-- For extremely write-heavy tables:
ALTER TABLE events SET (
    autovacuum_vacuum_cost_delay = 2,   -- Less delay between vacuum cycles (default 2ms)
    autovacuum_vacuum_cost_limit = 1000  -- More work per cycle (default 200)
);
```

### The Transaction ID Wraparound Problem

```
PostgreSQL uses 32-bit transaction IDs (XIDs). That's ~4.2 billion transactions.
When XIDs wrap around, PostgreSQL MUST freeze old tuples or data becomes invisible.

If autovacuum can't keep up:

  ┌──────────────────────────────────────────────────────┐
  │  WARNING: database "mydb" must be vacuumed within    │
  │  10000000 transactions                               │
  │  HINT: To avoid a database shutdown, execute a       │
  │  database-wide VACUUM.                               │
  └──────────────────────────────────────────────────────┘

  If you ignore this → PostgreSQL SHUTS DOWN to prevent data loss.
  This is not a drill. It has killed production systems.
```

> 💡 **Prevention:** Monitor `age(datfrozenxid)` and ensure autovacuum is keeping up:
> ```sql
> SELECT datname, age(datfrozenxid) AS xid_age,
>        current_setting('autovacuum_freeze_max_age') AS freeze_max
> FROM pg_database
> ORDER BY age(datfrozenxid) DESC;
> ```

---

## 📐 Table Partitioning — Divide and Conquer

When tables grow to hundreds of millions of rows, even good indexes struggle. Partitioning splits one logical table into many physical pieces.

### Declarative Partitioning (PostgreSQL 10+)

```sql
-- 1. RANGE Partitioning (most common — perfect for time-series)
CREATE TABLE orders (
    id          BIGSERIAL,
    customer_id INT,
    order_date  DATE NOT NULL,
    amount      NUMERIC(12,2),
    status      TEXT
) PARTITION BY RANGE (order_date);

-- Create partitions
CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Future-proof: create a default partition for unexpected dates
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

```
What happens when you query:

SELECT * FROM orders WHERE order_date = '2024-06-15';

Without partitioning:  Scan 500M rows  → 😰
With partitioning:     Scan only orders_2024 (~170M rows)  → 😊
                       Other partitions are SKIPPED entirely (partition pruning)

EXPLAIN shows: "Partitions pruned: 4 of 5" ← Only 1 partition scanned
```

### Other Partitioning Strategies

```sql
-- 2. LIST Partitioning (by category/status/region)
CREATE TABLE customers (
    id      SERIAL,
    name    TEXT,
    country TEXT NOT NULL
) PARTITION BY LIST (country);

CREATE TABLE customers_india PARTITION OF customers FOR VALUES IN ('India');
CREATE TABLE customers_usa PARTITION OF customers FOR VALUES IN ('USA', 'Canada');
CREATE TABLE customers_eu PARTITION OF customers FOR VALUES IN ('UK', 'Germany', 'France');
CREATE TABLE customers_other PARTITION OF customers DEFAULT;


-- 3. HASH Partitioning (even distribution across N partitions)
CREATE TABLE sessions (
    id      UUID NOT NULL,
    user_id INT,
    data    JSONB
) PARTITION BY HASH (id);

CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Partition Management

```sql
-- Add a new partition (zero downtime)
CREATE TABLE orders_2026 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- Detach an old partition (for archiving)
ALTER TABLE orders DETACH PARTITION orders_2020;
-- Now orders_2020 is a standalone table — archive it, drop it, whatever.

-- Attach an existing table as a partition
ALTER TABLE orders ATTACH PARTITION orders_archive
    FOR VALUES FROM ('2019-01-01') TO ('2020-01-01');
```

### Partitioning Best Practices

```
✅ DO:
  • Always include the partition key in WHERE clauses (enables pruning)
  • Create indexes on each partition (they're inherited)
  • Use RANGE for time-based data (by month or year)
  • Automate partition creation (pg_partman extension)
  • Monitor with: SELECT * FROM pg_partition_tree('orders');

❌ DON'T:
  • Partition tables with < 10M rows (overhead outweighs benefit)
  • Create too many partitions (1000+ is problematic; aim for <200)
  • Forget the DEFAULT partition (INSERTs fail for unmatched values)
  • Use HASH partitioning for time-series (can't prune by date)
```

---

## ⚙️ postgresql.conf — Tuning the Engine

### Memory Settings

```ini
# ─── Shared Buffers (Most Important Setting) ───
# How much RAM PostgreSQL uses for caching data pages.
# Rule of thumb: 25% of total system RAM
# Example: 16 GB RAM → shared_buffers = 4GB
shared_buffers = 4GB

# ─── Work Memory ───
# RAM per sort/hash operation PER QUERY.
# ⚠️ A complex query with 5 sorts uses 5 × work_mem!
# Start conservative: 64MB. Increase for analytics workloads.
work_mem = 64MB

# ─── Maintenance Work Memory ───
# RAM for VACUUM, CREATE INDEX, ALTER TABLE.
# Can be generous: 512MB–1GB
maintenance_work_mem = 1GB

# ─── Effective Cache Size ───
# NOT actual allocation! Just tells planner how much OS cache is available.
# Rule of thumb: 75% of total system RAM
effective_cache_size = 12GB
```

```
Memory Allocation Visual (16 GB Server):

┌──────────────────────────────────────────┐
│ Total RAM: 16 GB                         │
├──────────┬───────────────────────────────┤
│  4 GB    │ shared_buffers (PG cache)     │
├──────────┼───────────────────────────────┤
│  ~2 GB   │ OS + other processes          │
├──────────┼───────────────────────────────┤
│  ~10 GB  │ OS page cache (filesystem)    │
│          │ ← effective_cache_size = 12GB │
│          │   (includes shared_buffers)   │
└──────────┴───────────────────────────────┘
```

### WAL & Checkpoint Settings

```ini
# ─── WAL Level ───
# 'replica' for streaming replication (most common)
# 'logical' for logical replication
wal_level = replica

# ─── Checkpoint Timing ───
# How often dirty pages are flushed to disk.
# Longer intervals = better write performance, longer recovery.
checkpoint_timeout = 10min           # default 5min
max_wal_size = 4GB                   # default 1GB
min_wal_size = 1GB                   # default 80MB

# ─── Checkpoint Completion Target ───
# Spread checkpoint writes over this fraction of the interval.
# 0.9 = spread over 90% of the interval (less I/O spikes)
checkpoint_completion_target = 0.9
```

### Parallelism Settings

```ini
# ─── Parallel Workers ───
# PostgreSQL can use multiple CPU cores for a single query!
max_parallel_workers_per_gather = 4    # Workers per query node
max_parallel_workers = 8               # Total parallel workers
max_parallel_maintenance_workers = 4   # For CREATE INDEX, VACUUM

# ─── When to Parallelize ───
# Minimum table size to consider parallel scan
min_parallel_table_scan_size = 8MB
min_parallel_index_scan_size = 512kB
```

### Connection Settings

```ini
# ─── Maximum Connections ───
# ⚠️ Each connection uses ~10MB of RAM!
# 100 connections = ~1GB just for connection overhead.
# Use PgBouncer instead of raising this blindly.
max_connections = 200
```

### Quick Tuning Cheat Sheet by Server Size

| Setting | 4 GB RAM | 16 GB RAM | 64 GB RAM | 256 GB RAM |
|---------|----------|-----------|-----------|------------|
| `shared_buffers` | 1 GB | 4 GB | 16 GB | 64 GB |
| `effective_cache_size` | 3 GB | 12 GB | 48 GB | 192 GB |
| `work_mem` | 16 MB | 64 MB | 256 MB | 512 MB |
| `maintenance_work_mem` | 256 MB | 1 GB | 2 GB | 4 GB |
| `max_connections` | 100 | 200 | 300 | 500 |
| `max_parallel_workers` | 2 | 4 | 8 | 16 |

> 💡 **Pro Tip:** Use [PGTune](https://pgtune.leopard.in.ua/) to generate optimal settings for your hardware. It's free and accurate.

---

## 🔌 Connection Pooling with PgBouncer

### The Problem

```
Without pooling:
  500 concurrent users → 500 database connections
  Each connection = ~10MB RAM = 5GB just for connections!
  PostgreSQL handles context switching for 500 processes = SLOW

With PgBouncer:
  500 concurrent users → PgBouncer → 50 database connections
  Only 50 × 10MB = 500MB. 10× less RAM. WAY faster.
```

### PgBouncer Modes

| Mode | How It Works | Best For |
|------|-------------|----------|
| **Session** | 1 client = 1 server conn for entire session | Apps that use session-level features (TEMP tables, SET) |
| **Transaction** | Client gets a conn only during a transaction | ✅ Most web apps (recommended) |
| **Statement** | Client gets a conn only for a single statement | Simple queries, no multi-statement transactions |

```ini
# pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction          # ← Most common
max_client_conn = 1000           # Max connections FROM clients
default_pool_size = 50           # Actual connections TO PostgreSQL
reserve_pool_size = 10           # Extra connections for bursts
```

---

## 📈 Essential Monitoring Queries

### Cache Hit Ratio (Should be > 99%)

```sql
SELECT 
    sum(heap_blks_read) AS disk_reads,
    sum(heap_blks_hit) AS cache_hits,
    round(100.0 * sum(heap_blks_hit) / 
        NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) AS cache_hit_ratio
FROM pg_statio_user_tables;

-- If cache_hit_ratio < 99% → increase shared_buffers or add RAM
```

### Index Usage Ratio (Should be > 95%)

```sql
SELECT 
    relname AS table,
    seq_scan,
    idx_scan,
    round(100.0 * idx_scan / NULLIF(seq_scan + idx_scan, 0), 2) AS idx_usage_pct,
    n_live_tup AS row_count
FROM pg_stat_user_tables
WHERE n_live_tup > 10000
ORDER BY idx_usage_pct ASC;

-- Tables with low idx_usage_pct likely need indexes!
```

### Unused Indexes (Wasting disk & slowing writes)

```sql
SELECT 
    schemaname, tablename, indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS times_used
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (SELECT conindid FROM pg_constraint)  -- Exclude PKs/UNIQUEs
ORDER BY pg_relation_size(indexrelid) DESC;

-- If an index has 0 scans after weeks of operation → DROP IT.
-- Unused indexes waste disk space AND slow down every INSERT/UPDATE/DELETE.
```

### Table Bloat Check

```sql
SELECT 
    schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS bloat_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;

-- bloat_pct > 20% → autovacuum isn't keeping up. Tune it.
-- bloat_pct > 50% → consider VACUUM FULL (with downtime) or pg_repack (online)
```

### Long-Running Queries

```sql
SELECT 
    pid,
    now() - pg_stat_activity.query_start AS duration,
    state,
    LEFT(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY duration DESC;
```

### Lock Monitoring

```sql
-- Find blocked queries and what's blocking them
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.query AS blocked_query,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

---

## 🔥 Real-World Performance Patterns

### Pattern 1: The Slow Dashboard Query

```sql
-- ❌ SLOW: 12 seconds
SELECT 
    date_trunc('day', created_at) AS day,
    COUNT(*) AS orders,
    SUM(amount) AS revenue
FROM orders
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY 1
ORDER BY 1;

-- Fix 1: Add a BRIN index (data is time-ordered)
CREATE INDEX idx_orders_created_brin ON orders USING brin(created_at);

-- Fix 2: If you need it faster, create a materialized view
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT 
    date_trunc('day', created_at) AS day,
    COUNT(*) AS orders,
    SUM(amount) AS revenue
FROM orders
GROUP BY 1;

CREATE UNIQUE INDEX idx_daily_rev ON daily_revenue(day);

-- Refresh periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;
-- CONCURRENTLY = no downtime during refresh (requires unique index)
```

### Pattern 2: N+1 Query Problem

```sql
-- ❌ Application makes 1001 queries:
-- Query 1: SELECT * FROM users LIMIT 1000;
-- Query 2-1001: SELECT * FROM orders WHERE user_id = ? (×1000)

-- ✅ Single query with JOIN:
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.id IN (SELECT id FROM users LIMIT 1000);
```

### Pattern 3: Pagination Done Right

```sql
-- ❌ SLOW on page 10000: OFFSET 100000 still scans 100000 rows
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 100000;

-- ✅ Keyset pagination (cursor-based) — constant speed regardless of page:
SELECT * FROM products 
WHERE id > 100000      -- Last seen ID from previous page
ORDER BY id 
LIMIT 10;
```

---

## 🧪 Performance Tuning Checklist

```
Before you start tuning, go through this checklist:

□ 1. Run EXPLAIN ANALYZE on the slow query
□ 2. Check for Seq Scans on large tables → add indexes
□ 3. Check estimated vs actual row counts → run ANALYZE
□ 4. Check for missing indexes on JOIN columns
□ 5. Check pg_stat_statements for top time-consuming queries
□ 6. Check cache hit ratio (> 99%?)
□ 7. Check index usage ratio (> 95%?)
□ 8. Check for unused indexes → drop them
□ 9. Check table bloat → tune autovacuum
□ 10. Check long-running queries and locks
□ 11. Review postgresql.conf settings (shared_buffers, work_mem)
□ 12. Consider partitioning for tables > 100M rows
□ 13. Consider materialized views for complex analytics
□ 14. Consider connection pooling (PgBouncer)
□ 15. Use pg_repack for zero-downtime table/index rebuild
```

---

## 🎓 Key Takeaways

```
┌──────────────────────────────────────────────────────────────┐
│  PostgreSQL Performance — The Cheat Sheet                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  🔍 EXPLAIN (ANALYZE, BUFFERS) is your best friend           │
│  📊 pg_stat_statements finds your worst queries              │
│  🌳 B-tree for 95% of indexes                               │
│  📦 GIN for JSONB, arrays, full-text search                  │
│  🌍 GiST for geometry, ranges, IP addresses                  │
│  🧱 BRIN for huge time-series tables (tiny index!)           │
│  🎯 Partial + Covering indexes = surgical precision          │
│  🧹 Tune autovacuum — don't let bloat kill you               │
│  📐 Partition tables > 100M rows                             │
│  ⚙️ shared_buffers = 25% of RAM                              │
│  🔌 PgBouncer for connection pooling                         │
│  📈 Monitor cache hit ratio, index usage, bloat              │
│                                                              │
│  Remember: Fix the QUERY first, then the INDEX,              │
│  then the CONFIG, then the HARDWARE. In that order.          │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [2E.6 — PostgreSQL Replication & High Availability](./06-PG-Replication.md) 🔴🔥
> *Streaming replication, Logical replication, Patroni, PgBouncer, and building systems that never go down.*
