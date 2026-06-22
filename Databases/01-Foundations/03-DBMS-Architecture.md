# 1.3 — DBMS Architecture & How Databases Work Internally 🟡

> **"You don't need to build an engine to drive a car. But the best drivers understand what's under the hood."**

---

## 📌 What You'll Learn

- The **3-level architecture** (ANSI/SPARC) — why it matters
- How a **query travels** from your application to disk and back
- **Storage engines** — the heart of every database
- **B-Trees and B+Trees** — the data structure behind 90% of databases
- **LSM Trees** — the write-optimized alternative
- **Buffer Pool / Cache** — why databases are fast
- **Write-Ahead Logging (WAL)** — how databases survive crashes
- How all of this works together in real databases

---

## 1. The Three-Level Architecture (ANSI/SPARC)

Every DBMS is designed with **three abstraction levels** that separate users from physical storage. This is the foundation of data independence.

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   👤 User A        👤 User B        👤 User C            │
│   (sees orders)    (sees products)  (sees analytics)     │
│       │                │                │                │
│       ▼                ▼                ▼                │
│   ┌────────────────────────────────────────────┐         │
│   │          EXTERNAL LEVEL (Views)             │  ← What users/apps see
│   │  View 1: Order Summary                      │    (customized per user)
│   │  View 2: Product Catalog                    │
│   │  View 3: Sales Dashboard                    │
│   └────────────────────┬───────────────────────┘         │
│                        │                                  │
│                        ▼                                  │
│   ┌────────────────────────────────────────────┐         │
│   │        CONCEPTUAL LEVEL (Logical Schema)    │  ← Complete logical structure
│   │                                              │    (tables, relationships,
│   │  Tables: customers, orders, products,        │     constraints, data types)
│   │          order_items, payments                │
│   │  Relationships: FK constraints               │
│   │  Constraints: NOT NULL, UNIQUE, CHECK        │
│   └────────────────────┬───────────────────────┘         │
│                        │                                  │
│                        ▼                                  │
│   ┌────────────────────────────────────────────┐         │
│   │         INTERNAL LEVEL (Physical)           │  ← How data is stored on disk
│   │                                              │    (files, indexes, pages,
│   │  Storage: Pages, Extents, Tablespaces        │     compression, partitions)
│   │  Indexes: B+Tree on customer_id              │
│   │  File Org: Heap, Clustered, Hash             │
│   │  Compression: LZ4 on archive tables          │
│   └────────────────────────────────────────────┘         │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Why Three Levels?

| Benefit | Explanation |
|---------|------------|
| **Logical Data Independence** | Change physical storage (add index, move to SSD) without affecting applications |
| **Physical Data Independence** | Change logical schema (add column) without breaking user views |
| **Security** | Each user sees only their view, not the full schema |
| **Simplicity** | Users don't need to know WHERE or HOW data is stored |

> 💡 **Real Example**: You can switch PostgreSQL from HDD to SSD, or add a new index — and your application's SQL queries don't change at all. That's physical data independence.

---

## 2. The Journey of a SQL Query — From Keyboard to Disk

When you type `SELECT * FROM employees WHERE salary > 50000`, here's what **actually** happens inside the DBMS:

```
Step 1           Step 2           Step 3           Step 4
YOUR APP         PARSER           OPTIMIZER        EXECUTOR
   │                │                │                │
   │  SQL String    │  Parse Tree    │  Execution     │  Results
   │  ──────────→   │  ──────────→   │  Plan          │  ──────────→  📊
   │                │                │  ──────────→   │
   │                │                │                │
   ▼                ▼                ▼                ▼

┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Step 1: CONNECTION MANAGER                                      │
│  • Authenticate user (username/password)                         │
│  • Check permissions (can this user SELECT from employees?)      │
│  • Assign a thread/process for this connection                   │
│                                                                  │
│  Step 2: PARSER (Syntax Checker)                                 │
│  • Lexical analysis: Break SQL into tokens                       │
│  • Syntax check: Is the SQL grammatically correct?               │
│  • Semantic check: Does table "employees" exist?                 │
│         Does column "salary" exist in that table?                │
│  • Output: Parse Tree (Abstract Syntax Tree)                     │
│                                                                  │
│  Step 3: QUERY OPTIMIZER (The Brain 🧠)                          │
│  • Generate multiple execution plans                             │
│  • Estimate cost of each plan (I/O, CPU, Memory)                │
│  • Consider: Which indexes to use? Scan full table?             │
│         Join order? Hash join vs Nested loop?                   │
│  • Pick the CHEAPEST plan                                        │
│  • Output: Optimized Execution Plan                              │
│                                                                  │
│  Step 4: EXECUTION ENGINE                                        │
│  • Execute the chosen plan step by step                          │
│  • Read data from Buffer Pool (memory) or Disk                  │
│  • Apply filters (WHERE salary > 50000)                         │
│  • Return result set to the client                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Detailed Flow Diagram

```
          SQL Query
              │
              ▼
     ┌────────────────┐
     │   CONNECTION    │  Auth + Session
     │    MANAGER      │
     └───────┬────────┘
              │
              ▼
     ┌────────────────┐
     │     PARSER      │  SQL → Parse Tree
     │  + VALIDATOR     │  Check syntax & semantics
     └───────┬────────┘
              │
              ▼
     ┌────────────────┐
     │    QUERY        │  Find the fastest way
     │   OPTIMIZER     │  to get results
     │  (Cost-Based)   │  Uses STATISTICS
     └───────┬────────┘
              │
              ▼
     ┌────────────────┐
     │   EXECUTION     │  Execute the plan
     │    ENGINE       │
     └───────┬────────┘
              │
         ┌────┴────┐
         │         │
         ▼         ▼
  ┌──────────┐ ┌──────────┐
  │  BUFFER   │ │   DISK   │
  │   POOL    │ │  STORAGE │
  │ (Memory)  │←│  (Files) │
  └──────────┘ └──────────┘
```

---

## 3. Storage Engine — The Heart of the Database

The **storage engine** is responsible for how data is physically stored, retrieved, and managed on disk. Different databases use different engines — and some let you choose.

### What a Storage Engine Does

```
┌─────────────────────────────────────────────────┐
│            STORAGE ENGINE RESPONSIBILITIES        │
├─────────────────────────────────────────────────┤
│  • Read/Write data to/from disk                  │
│  • Manage in-memory caches (Buffer Pool)         │
│  • Handle locking & concurrency                  │
│  • Implement transactions (ACID)                 │
│  • Manage indexes                                │
│  • Compress data                                 │
│  • Handle crash recovery                         │
│  • Manage data file layout (pages, extents)      │
└─────────────────────────────────────────────────┘
```

### Storage Engines by Database

| Database | Storage Engine(s) | Notes |
|----------|------------------|-------|
| **MySQL** | InnoDB (default), MyISAM, Memory, Archive | InnoDB = ACID; MyISAM = legacy, no transactions |
| **PostgreSQL** | Heap-based (custom) | Single engine, highly optimized MVCC |
| **MongoDB** | WiredTiger (default) | Replaced MMAPv1 in MongoDB 3.2+ |
| **Oracle** | Integrated (proprietary) | Tightly coupled, not pluggable |
| **SQL Server** | Integrated + In-Memory OLTP (Hekaton) | Hekaton for extreme OLTP performance |
| **SQLite** | B-Tree based (integrated) | Single file, self-contained |
| **Cassandra** | LSM Tree based | Optimized for writes |
| **RocksDB** | LSM Tree | Used by CockroachDB, TiDB, YugabyteDB |

### MySQL InnoDB vs MyISAM (Classic Comparison)

| Feature | InnoDB ⭐ | MyISAM |
|---------|----------|--------|
| Transactions | ✅ Full ACID | ❌ None |
| Row-level Locking | ✅ Yes | ❌ Table-level only |
| Foreign Keys | ✅ Supported | ❌ Not supported |
| Crash Recovery | ✅ Auto (redo log) | ❌ Manual repair |
| Full-Text Search | ✅ (5.6+) | ✅ |
| Count(*) Speed | Slower (counts rows) | Fast (stores count) |
| Use Case | Everything modern | Legacy read-heavy |

> 💡 **Rule**: Always use **InnoDB** for MySQL. MyISAM is effectively deprecated.

---

## 4. How Data is Stored on Disk — Pages, Extents, Tablespaces

Databases don't read/write individual rows from disk. They work in **pages** (also called blocks).

### The Storage Hierarchy

```
DATABASE
  └── TABLESPACE (logical container)
        └── SEGMENT (one per table/index)
              └── EXTENT (group of contiguous pages)
                    └── PAGE / BLOCK (basic I/O unit)
                          └── ROWS (actual data)
```

### Pages — The Fundamental Unit

```
┌──────────────────────────────────────────┐
│              PAGE (8 KB typical)          │
├──────────────────────────────────────────┤
│  PAGE HEADER                             │
│  • Page ID, Page type, Checksum          │
│  • Free space pointer, LSN (Log Seq #)   │
├──────────────────────────────────────────┤
│                                          │
│  ROW 1: {id:1, name:"Rahul", sal:80000}  │
│  ROW 2: {id:2, name:"Priya", sal:70000}  │
│  ROW 3: {id:3, name:"John",  sal:65000}  │
│  ROW 4: {id:4, name:"Sarah", sal:90000}  │
│                                          │
│  ... (free space for more rows) ...      │
│                                          │
├──────────────────────────────────────────┤
│  ROW DIRECTORY (pointers to row offsets) │
└──────────────────────────────────────────┘

Page sizes by database:
• PostgreSQL: 8 KB (fixed)
• MySQL/InnoDB: 16 KB (configurable)
• Oracle: 8 KB (configurable: 2-32 KB)
• SQL Server: 8 KB (fixed)
• SQLite: 4 KB (configurable: 512B - 64KB)
```

### Why Pages Matter

```
Scenario: You want ROW with id = 42

WITHOUT pages (read individual rows):
  → Disk seek for every single row → EXTREMELY SLOW

WITH pages (read pages):
  → Read ONE page (8 KB) → Get 50-100 rows at once
  → Row 42 is probably in the same page as rows around it
  → MUCH faster (locality of reference)

Key Insight:
  Disk I/O is measured in PAGES, not rows.
  Reading 1 row = Reading 1 page (might as well read all rows in that page!)
```

---

## 5. B-Tree and B+Tree — The Data Structure Behind Everything

### Why Not Binary Search Tree?

```
Binary Search Tree for 1 million records:
  → Height = ~20 levels
  → Finding one record = 20 disk reads
  → Each disk read ≈ 10ms
  → Total: 200ms 😞

B+Tree for 1 million records (branching factor 100):
  → Height = ~3 levels
  → Finding one record = 3 disk reads
  → Total: 30ms 😊

B+Tree for 1 BILLION records:
  → Height = ~5 levels
  → 5 disk reads = 50ms (still fast!) 🚀
```

### B-Tree vs B+Tree

```
B-TREE:
  • Data stored in ALL nodes (internal + leaf)
  • Each key appears only once in the tree
  • Good for: Random lookups

B+TREE (used by most databases):
  • Data stored ONLY in leaf nodes
  • Internal nodes = only keys (navigation/routing)
  • Leaf nodes are LINKED (doubly linked list)
  • Good for: Range scans + Random lookups
```

### B+Tree Visualized

```
                    ┌─────────────────────┐
                    │    [30]  [60]  [90]   │  ← INTERNAL NODE (keys only)
                    └──┬──────┬──────┬──┬──┘    (no data here)
                       │      │      │  │
            ┌──────────┘      │      │  └───────────┐
            │                 │      │              │
            ▼                 ▼      ▼              ▼
    ┌──────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
    │ [10][20][25] │→ │[35][42][55]│→ │[61][75][88]│→ │[91][95][99]│
    │  D   D   D   │  │ D   D   D  │  │ D   D   D  │  │ D   D   D  │
    └──────────────┘  └────────────┘  └────────────┘  └────────────┘
         ↑                                                    ↑
    LEAF NODES                                           LEAF NODES
    (actual data/pointers)                        (linked for range scans)
    ← ← ← ← ← ← ← DOUBLY LINKED LIST → → → → → → → →

    Finding key=42:
    Step 1: Root → 42 > 30, 42 < 60 → go to middle child
    Step 2: Leaf → scan → found [42]!
    Only 2 disk reads! 🚀

    Range scan WHERE id BETWEEN 35 AND 75:
    Step 1: Find 35 in leaf
    Step 2: Follow linked list → 42 → 55 → 61 → 75 → STOP
    Sequential reads (FAST!) 🚀
```

### How B+Trees Map to Database Operations

| SQL Operation | B+Tree Action |
|--------------|---------------|
| `SELECT WHERE id = 42` | Navigate tree → leaf (point lookup) |
| `SELECT WHERE id BETWEEN 10 AND 50` | Find 10, follow leaf links to 50 |
| `INSERT` | Find correct leaf, insert, possibly split |
| `DELETE` | Find in leaf, remove, possibly merge |
| `UPDATE WHERE id = 42` | Find in leaf, update in place (or delete + insert) |
| `ORDER BY id` | Just read leaf nodes left to right! Already sorted! |

---

## 6. LSM Trees — The Write-Optimized Alternative

B+Trees are great for reads, but every write must update the tree on disk (**random I/O**). LSM Trees flip the trade-off: **writes are always sequential**.

### How LSM Trees Work

```
STEP 1: Write comes in
         │
         ▼
┌─────────────────────┐
│   MEMTABLE (RAM)     │  ← In-memory sorted tree (Red-Black or Skip List)
│   [key5] [key2]      │    Fast writes! Just insert into memory.
│   [key8] [key1]      │
└────────┬────────────┘
         │  When Memtable is full...
         │
STEP 2:  ▼  FLUSH to disk (sequential write)
┌─────────────────────┐
│  SSTABLE (Level 0)   │  ← Sorted String Table (immutable file on disk)
│  [key1][key2][key5]  │    Keys are sorted within each file
│  [key8]              │
└─────────────────────┘

Over time, many SSTables accumulate...

STEP 3: COMPACTION (merge SSTables)
┌──────────┐  ┌──────────┐     ┌──────────────────────┐
│ SSTable 1 │ +│ SSTable 2 │ ──→ │  Merged SSTable       │
│ [1][3][7] │  │ [2][5][9] │     │  [1][2][3][5][7][9]  │
└──────────┘  └──────────┘     └──────────────────────┘
```

### B+Tree vs LSM Tree Comparison

| Feature | B+Tree | LSM Tree |
|---------|--------|----------|
| **Write Speed** | Slower (random I/O) | Faster (sequential I/O) |
| **Read Speed** | Faster (single location) | Slower (check multiple levels) |
| **Space Usage** | Efficient | May have duplicates during compaction |
| **Write Amplification** | Lower | Higher (rewrite during compaction) |
| **Used By** | PostgreSQL, MySQL, Oracle, SQL Server | Cassandra, RocksDB, LevelDB, HBase |
| **Best For** | Read-heavy workloads | Write-heavy workloads |

> 💡 **Rule of Thumb**: 
> - **Read-heavy** → B+Tree (most RDBMS)
> - **Write-heavy** → LSM Tree (most NoSQL)
> - Many NewSQL databases (CockroachDB, TiDB) use **RocksDB** (LSM) underneath

---

## 7. Buffer Pool — Why Databases Are Fast

### The Problem

```
Disk access: ~10 milliseconds (HDD) or ~0.1ms (SSD)
RAM access:  ~0.0001 milliseconds (100 nanoseconds)

RAM is 100,000x faster than HDD and 1,000x faster than SSD!
```

### The Solution: Buffer Pool

The **Buffer Pool** (also called **Buffer Cache** or **Shared Buffers**) is a region of RAM that caches frequently accessed disk pages.

```
┌──────────────────────────────────────────────────────────────┐
│                        APPLICATION                            │
│                     "SELECT * FROM orders WHERE id = 42"     │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                      BUFFER POOL (RAM)                        │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Page 1  │ Page 7  │ Page 42 │ Page 15 │ Page 88 │ ... │  │
│  │(orders) │(users)  │(orders) │(index)  │(orders) │     │  │
│  │         │         │ ⭐ HIT! │         │         │     │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Page 42 found in Buffer Pool!                               │
│  → Return immediately (0.0001ms) ← 100,000x faster!        │
│  → No disk I/O needed!                                       │
│                                                              │
│  If NOT found (cache miss):                                  │
│  → Read from disk → Put in Buffer Pool → Return             │
│  → Evict an old page if Buffer Pool is full (LRU/Clock)     │
└──────────────────────────────────────────────────────────────┘
                           │
                    (cache miss only)
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                     DISK STORAGE                              │
│   [Page 1] [Page 2] [Page 3] ... [Page 42] ... [Page N]     │
└──────────────────────────────────────────────────────────────┘
```

### Buffer Pool Configuration by Database

| Database | Setting | Recommended |
|----------|---------|-------------|
| PostgreSQL | `shared_buffers` | 25% of RAM |
| MySQL/InnoDB | `innodb_buffer_pool_size` | 70-80% of RAM |
| Oracle | SGA (`db_cache_size`) | 40-60% of RAM |
| SQL Server | `max server memory` | Leave 2-4 GB for OS |
| MongoDB | WiredTiger cache | 50% of RAM (auto) |

### Buffer Pool Hit Ratio — The Golden Metric

```
                    Pages found in memory
Hit Ratio =  ─────────────────────────────── × 100
                Total page requests

Target: 99%+ for OLTP workloads

If hit ratio < 95%:
  → Buffer pool too small
  → OR queries scanning too many pages (missing indexes!)
  → OR working set doesn't fit in memory

PostgreSQL: SELECT 
  round(100.0 * sum(blks_hit) / sum(blks_hit + blks_read), 2) AS hit_ratio
  FROM pg_stat_database;

MySQL: SHOW STATUS LIKE 'Innodb_buffer_pool_read%';
```

---

## 8. Write-Ahead Logging (WAL) — How Databases Survive Crashes

### The Problem

```
Scenario:
1. You run: UPDATE accounts SET balance = balance - 500 WHERE id = 1;
2. Database modifies the page IN MEMORY (Buffer Pool)
3. 💥 POWER FAILURE before the page is written to disk
4. Data is LOST! The customer's balance is wrong!

Without WAL, you'd lose every update that hasn't been flushed to disk.
```

### The Solution: Write-Ahead Log

> **Rule**: Before ANY change is applied to the actual data files, it MUST first be written to the WAL (a sequential log file on disk).

```
┌────────────────────────────────────────────────────────────┐
│                    WRITE-AHEAD LOGGING                      │
│                                                            │
│  Step 1: Write the CHANGE to WAL (sequential write = fast) │
│  Step 2: Modify the page in Buffer Pool (memory)           │
│  Step 3: At some point, flush dirty pages to disk          │
│  Step 4: WAL entries can be discarded (checkpoint)          │
│                                                            │
│  ┌──────────────────────────────────────────────────┐      │
│  │  WAL FILE (append-only, sequential)               │      │
│  │  ┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐            │      │
│  │  │LSN:1││LSN:2││LSN:3││LSN:4││LSN:5│            │      │
│  │  │UPD  ││INS  ││DEL  ││UPD  ││INS  │            │      │
│  │  │T:acc ││T:ord││T:log││T:acc││T:usr│            │      │
│  │  │id:1  ││id:99││id:50││id:2  ││id:10│           │      │
│  │  │old:  ││val: ││     ││old:  ││val: │            │      │
│  │  │1000  ││2500 ││     ││3000 ││"Ram"│            │      │
│  │  │new:  ││     ││     ││new:  ││     │            │      │
│  │  │500   ││     ││     ││2500 ││     │            │      │
│  │  └─────┘└─────┘└─────┘└─────┘└─────┘            │      │
│  └──────────────────────────────────────────────────┘      │
│                                                            │
│  💥 CRASH at LSN:5                                         │
│  → On restart: REPLAY WAL from last checkpoint             │
│  → All committed changes are recovered!                    │
│  → Uncommitted changes are rolled back!                    │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### WAL Terminology by Database

| Concept | PostgreSQL | MySQL/InnoDB | Oracle | SQL Server |
|---------|-----------|-------------|--------|------------|
| WAL File | WAL (pg_wal/) | Redo Log (ib_logfile) | Redo Log | Transaction Log (.ldf) |
| Before-Image | Undo via MVCC | Undo Log | Undo Tablespace | Version Store |
| Flush to Disk | Checkpoint | Fuzzy Checkpoint | Checkpoint | Checkpoint |
| Recovery | Crash Recovery | Crash Recovery | Instance Recovery | Crash Recovery |

### The Checkpoint Process

```
During normal operation:
  Memory (Buffer Pool)              Disk (Data Files)
  ┌──────────────┐                 ┌──────────────┐
  │ Page 5 ★DIRTY│   ──────────→  │ Page 5 (old) │
  │ Page 12 CLEAN│                │ Page 12      │
  │ Page 8 ★DIRTY│   ──────────→  │ Page 8 (old) │
  └──────────────┘                └──────────────┘

CHECKPOINT:
  1. Flush ALL dirty pages from memory to disk
  2. Write checkpoint record to WAL
  3. Old WAL entries (before checkpoint) can be recycled

  After checkpoint:
  → If crash happens, only replay WAL from LAST checkpoint
  → Much faster recovery!
```

---

## 9. Query Optimizer — The Brain of the DBMS 🧠

The optimizer decides **HOW** to execute your query. The same SQL can be executed in many different ways — the optimizer picks the cheapest one.

### Example: Multiple Ways to Execute One Query

```sql
SELECT e.name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.id
WHERE e.salary > 50000;
```

```
Plan A: Full Table Scan                     Cost: 10,000 ❌
  → Scan ALL employees
  → For each, lookup department
  → Filter salary > 50000

Plan B: Index Scan on salary                Cost: 500 ✅
  → Use index on salary to find rows > 50000
  → For those rows only, lookup department

Plan C: Index Scan + Hash Join              Cost: 300 ✅✅
  → Use index on salary to find matching employees
  → Build hash table of departments
  → Hash join (O(1) per lookup)

Optimizer picks Plan C! 🎯
```

### What the Optimizer Considers

```
┌──────────────────────────────────────────────────────────┐
│              OPTIMIZER INPUTS                              │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  📊 STATISTICS (auto-collected)                          │
│  • Number of rows in each table                         │
│  • Number of distinct values per column                 │
│  • Data distribution (histograms)                       │
│  • Index selectivity                                    │
│                                                          │
│  📐 COST MODEL                                           │
│  • Disk I/O cost (sequential vs random)                 │
│  • CPU cost (comparison, hashing)                       │
│  • Memory cost (sorting, hash tables)                   │
│  • Network cost (distributed queries)                   │
│                                                          │
│  🔧 AVAILABLE ACCESS PATHS                               │
│  • Full table scan                                      │
│  • Index scan (which indexes exist?)                    │
│  • Index-only scan (covering index)                     │
│  • Bitmap index scan                                    │
│                                                          │
│  🔗 JOIN ALGORITHMS                                      │
│  • Nested Loop Join (small tables)                      │
│  • Hash Join (equality joins, medium-large)             │
│  • Merge Join (sorted inputs, large tables)             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Join Algorithms — Quick Reference

```
NESTED LOOP JOIN (like two nested for-loops)
  for each row in table_A:
    for each row in table_B:
      if A.key == B.key: output
  Best for: Small outer table + indexed inner table
  Cost: O(N × M) worst case

HASH JOIN
  Step 1: Build hash table from smaller table
  Step 2: Probe hash table with each row of larger table
  Best for: Large tables, equality joins, no useful index
  Cost: O(N + M)

MERGE JOIN (Sort-Merge)
  Step 1: Sort both tables by join key (or use index)
  Step 2: Merge sorted results (like merge in merge-sort)
  Best for: Pre-sorted data, range joins
  Cost: O(N log N + M log M) or O(N + M) if pre-sorted
```

---

## 10. Concurrency Control — MVCC

When multiple users access data simultaneously, the database must prevent chaos. The most popular approach is **MVCC (Multi-Version Concurrency Control)**.

### The Problem Without Concurrency Control

```
Time    User A (reads balance)       User B (writes balance)
────    ─────────────────────        ──────────────────────
T1      SELECT balance → 1000
T2                                   UPDATE balance = 500
T3      SELECT balance → ???

Without control:
  T3 might see 1000 (stale) or 500 (depends on timing)
  This is called a "dirty read" or "non-repeatable read"
```

### MVCC — The Solution

Instead of locking data, MVCC keeps **multiple versions** of each row:

```
┌──────────────────────────────────────────────────────────┐
│                    MVCC IN ACTION                         │
│                                                          │
│  Row: accounts (id=1)                                    │
│  ┌─────────────────────────────────────────────────┐     │
│  │ Version 1 (created by Txn 100)                   │     │
│  │ balance = 1000, xmin=100, xmax=200              │     │
│  ├─────────────────────────────────────────────────┤     │
│  │ Version 2 (created by Txn 200)                   │     │
│  │ balance = 500,  xmin=200, xmax=∞ (current)      │     │
│  └─────────────────────────────────────────────────┘     │
│                                                          │
│  Transaction 150 (started before Txn 200):               │
│  → Sees Version 1 (balance = 1000) ← snapshot!         │
│                                                          │
│  Transaction 250 (started after Txn 200 committed):     │
│  → Sees Version 2 (balance = 500) ← latest            │
│                                                          │
│  KEY BENEFIT:                                            │
│  Readers NEVER block Writers!                            │
│  Writers NEVER block Readers!                            │
│  Each transaction sees a CONSISTENT SNAPSHOT             │
└──────────────────────────────────────────────────────────┘
```

### MVCC Implementation by Database

| Database | MVCC Approach | Old Versions Stored |
|----------|--------------|-------------------|
| PostgreSQL | In-place (heap tuple versions) | In main table (cleaned by VACUUM) |
| MySQL/InnoDB | Undo log | In undo tablespace |
| Oracle | Undo segments | In undo tablespace |
| SQL Server | tempdb version store | In tempdb (when RCSI enabled) |

---

## 11. Putting It All Together — Full Architecture

### PostgreSQL Architecture Example

```
┌────────────────────────────────────────────────────────────────┐
│                    CLIENT APPLICATIONS                          │
│              (psql, pgAdmin, Your App)                          │
└───────────────────────┬────────────────────────────────────────┘
                        │  TCP/IP Connection
                        ▼
┌────────────────────────────────────────────────────────────────┐
│                     POSTMASTER                                  │
│              (Main daemon process)                              │
│  → Spawns one backend process per client connection            │
└───────────────────────┬────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ Backend  │   │ Backend  │   │ Backend  │    ← One per connection
  │ Process 1│   │ Process 2│   │ Process 3│    ← Parser + Optimizer
  └────┬─────┘   └────┬─────┘   └────┬─────┘      + Executor
       │              │              │
       └──────────────┼──────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────────┐
│                  SHARED MEMORY                                  │
│  ┌──────────────────────────────────────────────────────┐      │
│  │              SHARED BUFFERS (Buffer Pool)              │      │
│  │   [Page][Page][Page][Page][Page][Page][Page]...       │      │
│  └──────────────────────────────────────────────────────┘      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐      │
│  │  WAL Buffers  │  │   CLOG       │  │  Lock Manager  │      │
│  │  (write cache)│  │ (commit log) │  │  (row locks)   │      │
│  └──────────────┘  └──────────────┘  └────────────────┘      │
└───────────────────────┬────────────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────────────────┐
│                 BACKGROUND PROCESSES                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐    │
│  │  Writer   │ │WAL Writer│ │Checkpointr│ │ Autovacuum     │    │
│  │ (flushes  │ │(flushes  │ │(periodic  │ │ (cleans dead   │    │
│  │ dirty     │ │WAL to    │ │full flush)│ │  row versions) │    │
│  │ pages)    │ │disk)     │ │          │ │               │    │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────┘    │
│  ┌──────────┐ ┌──────────────────┐                            │
│  │Stats      │ │Archiver (ships   │                            │
│  │Collector  │ │WAL for replication│                           │
│  └──────────┘ └──────────────────┘                            │
└───────────────────────┬────────────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────────────────┐
│                     DISK STORAGE                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐     │
│  │  Data Files   │  │  WAL Files    │  │  CLOG Files       │    │
│  │  (base/)      │  │  (pg_wal/)    │  │  (pg_xact/)       │    │
│  │  - Tables     │  │  - Redo log   │  │  - Commit status  │    │
│  │  - Indexes    │  │  - Recovery   │  │  - Txn tracking   │    │
│  │  - TOAST      │  │  - Replication│  │                    │    │
│  └──────────────┘  └──────────────┘  └──────────────────┘     │
└────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Quick Recall — Chapter Summary

| Concept | One-Line Summary |
|---------|-----------------|
| Three-Level Architecture | External (views) → Conceptual (logic) → Internal (physical) |
| Query Journey | Parse → Optimize → Execute → Buffer Pool → Disk |
| Storage Engine | Manages how data is physically stored and retrieved |
| Pages | Fixed-size blocks (4-16 KB) — the basic I/O unit |
| B+Tree | Balanced tree, data in leaves, linked for range scans — used by most RDBMS |
| LSM Tree | Write-optimized, memtable → SSTables → compaction — used by NoSQL/NewSQL |
| Buffer Pool | RAM cache for disk pages — 99%+ hit ratio is the goal |
| WAL | Write changes to log FIRST, then apply — survives crashes |
| Query Optimizer | Cost-based brain that picks the fastest execution plan |
| MVCC | Multiple row versions — readers never block writers |

---

## ❓ Self-Check Questions

1. What are the three levels of the ANSI/SPARC architecture?
2. Why do databases use pages instead of reading individual rows?
3. How is a B+Tree different from a B-Tree?
4. When would you prefer an LSM Tree over a B+Tree?
5. What is the Buffer Pool and why does it matter?
6. Explain WAL in one sentence.
7. Name three join algorithms and when to use each.
8. How does MVCC allow readers and writers to not block each other?
9. What is a checkpoint and why is it necessary?
10. What inputs does the Query Optimizer use to choose a plan?

---

> **Next Chapter** → [1.4 — Data Modeling — The Art of Designing Data](./04-Data-Modeling.md)
