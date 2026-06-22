# 2D.1 — MySQL Architecture & Storage Engines 🟡⭐

> **"MySQL didn't become the world's most popular open-source database by accident — its architecture is elegantly modular, letting you swap entire storage engines like LEGO blocks."**

---

## 📌 What You'll Master in This Chapter

- MySQL's **layered architecture** — how a query travels from your app to disk and back
- **InnoDB vs MyISAM** — the legendary battle (and why InnoDB won)
- **Buffer Pool** — MySQL's secret weapon for speed
- **Redo Log, Undo Log, Doublewrite Buffer** — the holy trinity of crash safety
- How MySQL processes a query **step by step** (execution flow)
- Other storage engines you should know about

---

## 🌍 MySQL — The Backstory (30 Seconds)

```
Born:        1995 (by Michael "Monty" Widenius)
Named After: His daughter "My" (yes, really)
Acquired:    Sun Microsystems (2008) → Oracle (2010)
Fork:        MariaDB (created by Monty after Oracle acquisition)
Powers:      Facebook, Twitter/X, Uber, Airbnb, Netflix, YouTube, WordPress
License:     Dual — GPL (Community) + Commercial (Enterprise)
```

> 💡 **Fun Fact:** Facebook runs one of the largest MySQL deployments in the world — **billions** of queries per second across thousands of servers.

---

## 🔥 1. MySQL Architecture — The Big Picture

MySQL has a **modular, layered architecture**. This is its superpower — each layer does one job, and you can even swap the storage engine without changing your SQL.

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
│   mysql CLI │ MySQL Workbench │ JDBC/ODBC │ PHP │ Python │ App  │
└──────────────────────────┬──────────────────────────────────────┘
                           │  TCP/IP, Unix Socket, Named Pipe
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     CONNECTION LAYER                              │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐     │
│  │ Thread Pool / │  │Authentication│  │ Connection Manager │     │
│  │ Thread-per-   │  │  & Security  │  │ (max_connections)  │     │
│  │ Connection    │  │              │  │                    │     │
│  └──────────────┘  └──────────────┘  └────────────────────┘     │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      SQL LAYER (Server Layer)                    │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐     │
│  │    Parser     │  │  Optimizer   │  │   Query Cache      │     │
│  │ (Syntax Check)│  │ (Cost-Based) │  │  (Removed in 8.0!) │     │
│  └──────┬───────┘  └──────┬───────┘  └────────────────────┘     │
│         ▼                 ▼                                       │
│  ┌──────────────────────────────┐                                │
│  │       Execution Engine       │                                │
│  │   (executes the plan)        │                                │
│  └──────────────┬───────────────┘                                │
└─────────────────┼───────────────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                   STORAGE ENGINE LAYER                            │
│           (Pluggable — this is the LEGO part!)                   │
│                                                                   │
│  ┌─────────┐ ┌─────────┐ ┌────────┐ ┌────────┐ ┌────────────┐ │
│  │ InnoDB  │ │ MyISAM  │ │Memory  │ │Archive │ │ NDB Cluster│ │
│  │(DEFAULT)│ │(Legacy) │ │(RAM)   │ │(Logs)  │ │(Distributed│ │
│  └────┬────┘ └────┬────┘ └────┬───┘ └────┬───┘ └────────────┘ │
│       ▼           ▼           ▼          ▼                       │
│  ┌─────────────────────────────────────────────┐                 │
│  │              FILE SYSTEM / DISK              │                 │
│  │  .ibd  .frm  .MYD  .MYI  tablespace files   │                 │
│  └─────────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
```

### Each Layer Explained

| Layer | What It Does | Key Components |
|-------|-------------|----------------|
| **Client Layer** | Connects to MySQL | mysql CLI, Workbench, JDBC, Python connectors |
| **Connection Layer** | Authenticates, manages connections | Thread handling, SSL, `max_connections` |
| **SQL Layer** | Parses, optimizes, executes SQL | Parser → Optimizer → Executor |
| **Storage Engine Layer** | Actually reads/writes data | InnoDB (default), MyISAM, Memory, etc. |
| **File System** | Physical storage on disk | Data files, log files, tablespace files |

> ⭐ **Key Insight:** The SQL layer is **engine-agnostic** — it doesn't care HOW data is stored. It just tells the storage engine "give me rows matching this condition." This separation is why MySQL is so flexible.

---

## 🔥 2. Life of a MySQL Query — Step by Step

When you run `SELECT * FROM users WHERE age > 25`, here's what happens behind the scenes:

```
┌─────────────────────────────────────────────────────────────────┐
│                   QUERY EXECUTION FLOW                           │
│                                                                   │
│  Step 1 → CONNECTION                                             │
│           Client connects (TCP 3306 or Unix socket)              │
│           Authentication (username/password/SSL)                 │
│           Thread assigned (or pulled from thread pool)           │
│                           │                                       │
│  Step 2 → PARSER          ▼                                      │
│           Lexical analysis (tokenize SQL)                        │
│           Syntax validation (is SQL valid?)                      │
│           Creates parse tree (AST)                               │
│                           │                                       │
│  Step 3 → PREPROCESSOR    ▼                                      │
│           Semantic check (does table exist?)                     │
│           Privilege check (can user access this?)                │
│           Resolve names, verify column types                     │
│                           │                                       │
│  Step 4 → OPTIMIZER       ▼                                      │
│           Cost-based optimizer (CBO)                             │
│           Chooses JOIN order, index usage                        │
│           Generates execution plan                               │
│           (EXPLAIN shows you THIS step's output!)                │
│                           │                                       │
│  Step 5 → EXECUTOR        ▼                                      │
│           Calls storage engine API                               │
│           "Open table, read rows, apply filter"                  │
│           Sends results back to client                           │
│                           │                                       │
│  Step 6 → STORAGE ENGINE  ▼                                      │
│           InnoDB reads from buffer pool (RAM)                    │
│           If not in RAM → reads from disk → caches in buffer pool│
│           Returns rows to executor                               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

> 💡 **The Query Cache (RIP):** MySQL 5.x had a query cache that stored exact query→result mappings. It was **removed in MySQL 8.0** because it caused more problems than it solved — any write to a table invalidated ALL cached queries for that table, causing massive lock contention on busy systems. Use **ProxySQL** or **Redis** for application-level caching instead.

---

## 🔥 3. InnoDB — The King of Storage Engines

InnoDB has been the **default storage engine since MySQL 5.5** (2010). If you're using MySQL, you're using InnoDB.

### Why InnoDB Won

```
┌────────────────────────────────────────────────────────────┐
│  InnoDB's Power Features                                    │
├────────────────────────────────────────────────────────────┤
│  ✅ ACID Transactions (full support)                        │
│  ✅ Row-Level Locking (not table-level!)                    │
│  ✅ MVCC (Multi-Version Concurrency Control)                │
│  ✅ Foreign Key Constraints                                 │
│  ✅ Crash Recovery (automatic, using redo logs)             │
│  ✅ Clustered Index (data stored IN the PK index)           │
│  ✅ Buffer Pool (intelligent caching)                       │
│  ✅ Doublewrite Buffer (prevents partial page writes)       │
│  ✅ Change Buffer (optimizes secondary index writes)        │
│  ✅ Adaptive Hash Index (auto-creates hash indexes)         │
└────────────────────────────────────────────────────────────┘
```

### InnoDB Architecture Deep Dive

```
┌─────────────────────────────────────────────────────────────────┐
│                    InnoDB IN-MEMORY STRUCTURES                   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │              BUFFER POOL (innodb_buffer_pool_size)       │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │     │
│  │  │  Data Pages   │  │ Index Pages  │  │ Change Buffer│  │     │
│  │  │  (cached rows)│  │(cached idx)  │  │ (sec. index  │  │     │
│  │  │              │  │              │  │  changes)    │  │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │     │
│  │  ┌──────────────┐  ┌──────────────┐                    │     │
│  │  │ Undo Pages   │  │Adaptive Hash │                    │     │
│  │  │              │  │   Index      │                    │     │
│  │  └──────────────┘  └──────────────┘                    │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐                      │
│  │   Log Buffer     │  │ Additional Memory │                      │
│  │ (redo log cache) │  │  Pools            │                      │
│  └──────────────────┘  └──────────────────┘                      │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                    InnoDB ON-DISK STRUCTURES                      │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  System      │  │  File-Per-   │  │  General             │   │
│  │  Tablespace  │  │  Table       │  │  Tablespace          │   │
│  │ (ibdata1)    │  │  (.ibd files)│  │  (MySQL 8.0+)        │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  Redo Logs   │  │  Undo        │  │  Doublewrite         │   │
│  │ (ib_logfile0 │  │  Tablespace  │  │  Buffer              │   │
│  │  ib_logfile1)│  │  (undo_001)  │  │  (doublewrite file)  │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
│                                                                   │
│  ┌──────────────┐                                                │
│  │  Temp        │                                                │
│  │  Tablespace  │                                                │
│  │  (ibtmp1)    │                                                │
│  └──────────────┘                                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔥 4. The Buffer Pool — MySQL's RAM Powerhouse

The buffer pool is the **most important performance component** in InnoDB. It's a chunk of RAM where InnoDB caches table data and index pages.

### How It Works

```
Client Query: SELECT * FROM users WHERE id = 42
                    │
                    ▼
         ┌─────────────────┐
         │   Buffer Pool   │
         │   (check RAM)   │
         └────────┬────────┘
                  │
          ┌───────┴───────┐
          │               │
     Page Found?     Page NOT Found?
     (Cache HIT)     (Cache MISS)
          │               │
          ▼               ▼
     Return data     Read from DISK
     immediately     Load into Buffer Pool
     ⚡ (~0.1ms)     Evict old page (LRU)
                     Then return data
                     🐢 (~5-10ms)
```

### Buffer Pool Key Concepts

| Concept | What It Is | Why It Matters |
|---------|-----------|----------------|
| **Page** | 16KB block (smallest I/O unit) | InnoDB reads/writes in 16KB chunks, not individual rows |
| **LRU List** | Least Recently Used eviction | Old pages get kicked out when buffer pool is full |
| **Midpoint Insertion** | New pages enter at 3/8th from head | Prevents full table scans from flushing hot data |
| **Buffer Pool Instances** | Multiple pools for parallelism | Reduces lock contention (`innodb_buffer_pool_instances`) |
| **Dirty Pages** | Modified pages not yet written to disk | Flushed periodically or at checkpoint |

### Buffer Pool Sizing — The Golden Rule

```sql
-- Check current buffer pool size
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- Check buffer pool usage
SHOW STATUS LIKE 'Innodb_buffer_pool%';
```

```
┌──────────────────────────────────────────────────────────────┐
│  GOLDEN RULE: Set buffer pool to 70-80% of total RAM         │
│  (on a dedicated MySQL server)                                │
│                                                                │
│  Server RAM:  16 GB  →  Buffer Pool: 11-12 GB                 │
│  Server RAM:  64 GB  →  Buffer Pool: 48-50 GB                 │
│  Server RAM: 256 GB  →  Buffer Pool: 192-200 GB               │
│                                                                │
│  -- Set in my.cnf:                                             │
│  innodb_buffer_pool_size = 12G                                 │
│                                                                │
│  -- For sizes > 1GB, use multiple instances:                   │
│  innodb_buffer_pool_instances = 8                              │
│  (each instance = buffer_pool_size / instances)                │
└──────────────────────────────────────────────────────────────┘
```

### Monitoring Buffer Pool Efficiency

```sql
-- Buffer pool hit rate (should be > 99%)
SELECT 
    (1 - (
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status 
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    )) * 100 AS buffer_pool_hit_rate;
```

> ⭐ **Target:** Buffer pool hit rate should be **99%+**. If it's below 95%, your buffer pool is too small or your working set is too large.

> 💡 **Pro Tip:** In MySQL 8.0, the buffer pool state is automatically saved on shutdown and restored on startup (`innodb_buffer_pool_dump_at_shutdown = ON`). This means your server starts "warm" — no more cold-start performance dips!

---

## 🔥 5. Redo Log — The Crash Recovery Guardian

The redo log is MySQL's **Write-Ahead Log (WAL)**. It guarantees that committed transactions survive a crash, even if the data pages haven't been flushed to disk yet.

### Why Redo Logs Exist

```
THE PROBLEM WITHOUT REDO LOGS:
──────────────────────────────
1. You INSERT a row → data goes to buffer pool (RAM)
2. Transaction COMMITTED ✓
3. 💥 CRASH before dirty page flushed to disk
4. Data is LOST forever! 😱

THE SOLUTION WITH REDO LOGS:
────────────────────────────
1. You INSERT a row → data goes to buffer pool (RAM)
2. Change also written to REDO LOG on disk (sequential write — FAST!)
3. Transaction COMMITTED ✓
4. 💥 CRASH before dirty page flushed to disk
5. MySQL restarts → reads redo log → replays changes → DATA RECOVERED! ✅
```

### How Redo Logs Work

```
┌──────────────────────────────────────────────────────────────┐
│                    REDO LOG FLOW                              │
│                                                                │
│  Transaction                                                   │
│      │                                                         │
│      ▼                                                         │
│  ┌───────────┐     ┌──────────────┐     ┌─────────────┐      │
│  │  Modify   │────▶│  Log Buffer  │────▶│  Redo Log    │      │
│  │  Buffer   │     │  (RAM)       │     │  Files       │      │
│  │  Pool     │     │              │ fsync│  (on Disk)   │      │
│  │  (RAM)    │     │              │ ────▶│  ib_logfile0 │      │
│  └───────────┘     └──────────────┘     │  ib_logfile1 │      │
│                                          └─────────────┘      │
│                                                ▲               │
│  COMMIT fires fsync() ─────────────────────────┘               │
│  (guarantees redo is on disk before COMMIT returns)            │
│                                                                │
│  Later... checkpoint flushes dirty pages from buffer pool      │
│  to the actual data files (.ibd). Redo log space is reclaimed. │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Redo Log Configuration

```ini
# In my.cnf — MySQL 8.0.30+
innodb_redo_log_capacity = 4G      # Total redo log size (new unified setting)

# Pre-8.0.30 (legacy — you'll still see these in older setups)
innodb_log_file_size = 1G          # Size of each redo log file
innodb_log_files_in_group = 2      # Number of redo log files (circular)
```

### Redo Log Flush Behavior — `innodb_flush_log_at_trx_commit`

This is one of the **most critical settings** in MySQL:

```
┌───────┬──────────────────────────────────────────────────────────┐
│ Value │ Behavior                                                  │
├───────┼──────────────────────────────────────────────────────────┤
│   1   │ Flush to disk on EVERY commit (SAFEST — default)         │
│       │ → Zero data loss, but slower                              │
│       │ → Required for ACID compliance                            │
│       │                                                           │
│   2   │ Write to OS cache on commit, flush every 1 second         │
│       │ → Up to 1 second of data loss on OS crash                 │
│       │ → Faster than 1, safe if only MySQL crashes               │
│       │                                                           │
│   0   │ Write to log buffer, flush every 1 second                 │
│       │ → Up to 1 second of data loss on ANY crash                │
│       │ → FASTEST, but least safe                                 │
│       │ → Only for non-critical data (logs, analytics)            │
└───────┴──────────────────────────────────────────────────────────┘
```

> ⭐ **Interview Favorite:** "What does `innodb_flush_log_at_trx_commit = 1` do?" → It calls `fsync()` on every commit, forcing the redo log to be written to physical disk, guaranteeing durability. This is the **D** in **ACID**.

---

## 🔥 6. Undo Log — The Time Machine

The undo log stores the **previous versions of modified rows**. It serves two critical purposes:

### Purpose 1: Transaction Rollback

```
BEGIN;
UPDATE accounts SET balance = 500 WHERE id = 1;   -- Was 1000
-- Undo log saves: {id=1, balance=1000} (the OLD value)

ROLLBACK;
-- InnoDB reads undo log → restores balance to 1000 ✅
```

### Purpose 2: MVCC (Multi-Version Concurrency Control)

```
┌──────────────────────────────────────────────────────────────┐
│                    MVCC IN ACTION                             │
│                                                                │
│  Transaction A (started at T1):                                │
│    SELECT balance FROM accounts WHERE id = 1;                  │
│    → Sees balance = 1000 (snapshot from T1)                    │
│                                                                │
│  Transaction B (started at T2, T2 > T1):                       │
│    UPDATE accounts SET balance = 500 WHERE id = 1;             │
│    COMMIT;                                                     │
│    → balance is now 500 on disk                                │
│                                                                │
│  Transaction A (still running, reads again):                   │
│    SELECT balance FROM accounts WHERE id = 1;                  │
│    → STILL sees 1000! (reads OLD version from undo log)        │
│    → This is REPEATABLE READ isolation (MySQL default)         │
│                                                                │
│  Without MVCC: Transaction A would need to WAIT for B's lock  │
│  With MVCC:    Readers NEVER block writers. Writers NEVER      │
│                block readers. Everyone is happy. 🎉             │
└──────────────────────────────────────────────────────────────┘
```

### Undo Log Storage

```sql
-- MySQL 8.0: Undo logs stored in separate undo tablespaces
SHOW VARIABLES LIKE 'innodb_undo%';

-- Key settings:
-- innodb_undo_tablespaces = 2      (default, minimum 2)
-- innodb_undo_directory = /path    (location of undo files)
-- innodb_max_undo_log_size = 1G    (triggers auto-truncation)
-- innodb_undo_log_truncate = ON    (auto-reclaim space)
```

> 💡 **Watch Out:** Long-running transactions prevent undo log cleanup (called **purge**). An hour-long SELECT can cause the undo log to grow to gigabytes because InnoDB must keep old row versions for that transaction's MVCC snapshot. Always keep transactions short!

---

## 🔥 7. Doublewrite Buffer — Preventing Torn Pages

This is one of InnoDB's most clever safety mechanisms.

### The Problem: Torn (Partial) Page Writes

```
InnoDB page = 16 KB
OS page     = 4 KB (typically)

When InnoDB writes a 16KB page to disk, the OS does it as 4 separate 4KB writes.

What if a crash happens after writing only 2 of the 4 OS pages?

┌────────────────────┐
│  16KB InnoDB Page   │
├─────┬─────┬────┬───┤
│ 4KB │ 4KB │4KB │4KB│
│ NEW │ NEW │OLD │OLD│  ← TORN PAGE! Half new, half old.
└─────┴─────┴────┴───┘

The page on disk is now CORRUPTED.
You can't use the old version (partially overwritten).
You can't use the new version (incomplete).
Even the redo log can't fix this — it stores CHANGES, not full pages.
```

### The Solution: Doublewrite Buffer

```
┌──────────────────────────────────────────────────────────────┐
│                 DOUBLEWRITE BUFFER FLOW                       │
│                                                                │
│  Step 1: Write dirty pages to DOUBLEWRITE BUFFER on disk      │
│          (sequential write — fast!)                            │
│          ┌──────────────────────────────┐                     │
│          │    Doublewrite Buffer File    │                     │
│          │   [Page A] [Page B] [Page C]  │                     │
│          └──────────────────────────────┘                     │
│                         │                                      │
│  Step 2: fsync() — ensure doublewrite is safely on disk        │
│                         │                                      │
│  Step 3: Write pages to their ACTUAL locations (.ibd files)   │
│                         │                                      │
│  CRASH during Step 3?                                          │
│  → On recovery, InnoDB finds the torn page                     │
│  → Reads the GOOD COPY from doublewrite buffer                 │
│  → Overwrites the torn page                                    │
│  → Then applies redo log                                       │
│  → Data is SAFE! ✅                                            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```ini
# Doublewrite is ON by default (don't turn it off unless you know what you're doing)
innodb_doublewrite = ON

# MySQL 8.0.20+: Separate doublewrite files
innodb_doublewrite_dir = /fast_ssd/    # Put on fast storage
innodb_doublewrite_pages = 64          # Pages per batch
```

> ⭐ **"But doesn't writing everything twice make it slow?"** Not really. The doublewrite buffer writes are **sequential** (fast on both HDD and SSD), and the overhead is typically only 5-10%. The safety is worth it. Some filesystems (like ZFS) have their own checksumming, so you can disable doublewrite there — but only if you're sure.

---

## 🔥 8. InnoDB vs MyISAM — The Complete Showdown

MyISAM was the default engine before MySQL 5.5. You'll still encounter it in legacy systems and interviews.

```
┌─────────────────────────┬──────────────────────┬──────────────────────┐
│       Feature           │      InnoDB          │      MyISAM          │
├─────────────────────────┼──────────────────────┼──────────────────────┤
│ ACID Transactions       │ ✅ Full support       │ ❌ No transactions    │
│ Locking Granularity     │ Row-level locks      │ Table-level locks    │
│ Foreign Keys            │ ✅ Supported          │ ❌ Not supported      │
│ Crash Recovery          │ ✅ Automatic (redo)   │ ❌ Manual repair      │
│ MVCC                    │ ✅ Yes                │ ❌ No                 │
│ Full-Text Search        │ ✅ (since 5.6)        │ ✅ Yes                │
│ Clustered Index         │ ✅ PK = clustered     │ ❌ Heap-organized     │
│ Data Caching            │ Buffer pool          │ OS cache only        │
│ Index Caching           │ Buffer pool          │ Key buffer           │
│ Compression             │ ✅ Page compression   │ ✅ Row compression    │
│ Geospatial Support      │ ✅ (since 5.7)        │ ✅ Yes                │
│ Storage Files           │ .ibd (data + index)  │ .MYD (data)          │
│                         │                      │ .MYI (index)         │
│ Max Table Size          │ 64 TB                │ 256 TB               │
│ COUNT(*) Performance    │ Slow (scans rows)    │ ⚡ Instant (stored)   │
│ INSERT Speed (bulk)     │ Moderate             │ Fast                 │
│ SELECT Speed (simple)   │ Fast (from cache)    │ Fast (from OS cache) │
│ Concurrent Reads/Writes │ ✅ Excellent          │ ❌ Terrible           │
│ Default Since           │ MySQL 5.5 (2010)     │ Before MySQL 5.5     │
│ Verdict                 │ ✅ USE THIS           │ ❌ Avoid (legacy)     │
└─────────────────────────┴──────────────────────┴──────────────────────┘
```

### When Would You Still Use MyISAM?

Almost never. But here are the rare exceptions:

```
✅ Read-only or read-heavy tables with NO concurrent writes
✅ Tables where you constantly need COUNT(*) without WHERE (MyISAM is instant)
✅ Legacy applications that haven't migrated yet
✅ Temporary tables for intermediate processing

❌ Everything else → USE InnoDB
```

### How InnoDB's Clustered Index Works (vs MyISAM)

```
InnoDB (Clustered Index):
─────────────────────────
The PRIMARY KEY IS the table. Data rows are stored INSIDE the PK B+Tree.

                     [PK B+Tree]
                    /     |      \
                [10]    [20]    [30]
                 │       │       │
            ┌────┘  ┌────┘  ┌────┘
            ▼       ▼       ▼
         [Full]  [Full]  [Full]     ← Leaf nodes contain ENTIRE ROWS
         [Row ]  [Row ]  [Row ]

Secondary indexes point to the PK value (not to a physical address):
    Secondary Index → PK value → Clustered Index lookup → Row data
    (This is called a "bookmark lookup" or "double lookup")


MyISAM (Heap + Separate Index):
────────────────────────────────
Data is stored in insertion order (heap). Indexes point to file offsets.

         .MYD (data file)            .MYI (index file)
    ┌─────────────────────┐     ┌─────────────────────┐
    │ Row at offset 0x000 │◀────│ PK Index: id=10     │
    │ Row at offset 0x100 │◀────│ PK Index: id=20     │
    │ Row at offset 0x200 │◀────│ PK Index: id=30     │
    └─────────────────────┘     └─────────────────────┘
    
    ALL indexes (including PK) point to physical file offset.
    No "double lookup" needed, but no clustering benefit either.
```

> 💡 **Pro Tip:** Because InnoDB's secondary indexes store the PK value, **choose short primary keys**. A `BIGINT` PK (8 bytes) is ideal. A `UUID` PK (36 bytes as CHAR, or 16 bytes as BINARY) makes every secondary index ~2-4x larger! If you must use UUIDs, use `BINARY(16)` with ordered UUIDs (e.g., `UUID_TO_BIN(UUID(), 1)` in MySQL 8.0).

---

## 🔥 9. Other Storage Engines Worth Knowing

| Engine | Use Case | Key Feature |
|--------|----------|-------------|
| **MEMORY (HEAP)** | Temp tables, session data, lookup caches | All data in RAM — blazing fast, but lost on restart |
| **ARCHIVE** | Log data, audit trails | Write-once, compressed, no indexes (except PK). Very compact |
| **CSV** | Data interchange | Stores data as CSV files — readable by any spreadsheet |
| **BLACKHOLE** | Replication relay, testing | Accepts writes, discards them. Like `/dev/null` for databases |
| **NDB (Cluster)** | Distributed, real-time | MySQL Cluster (NDB). In-memory, auto-sharding, 99.999% uptime |
| **FEDERATED** | Remote table access | Query tables on remote MySQL servers as if they were local |
| **TokuDB** (3rd party) | Write-heavy workloads | Fractal Tree indexes, excellent compression, high insert rate |
| **RocksDB** (MyRocks) | Write-heavy, SSD-optimized | LSM-tree based, used by Facebook's MySQL fork |

```sql
-- See which engines are available
SHOW ENGINES;

-- Check which engine a table uses
SHOW TABLE STATUS LIKE 'users'\G

-- Change engine (CAUTION: locks table and copies ALL data)
ALTER TABLE users ENGINE = InnoDB;

-- Create table with specific engine
CREATE TABLE logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE = ARCHIVE;
```

---

## 🔥 10. InnoDB Tablespace Architecture

### File-Per-Table vs System Tablespace

```
┌──────────────────────────────────────────────────────────────┐
│  FILE-PER-TABLE (innodb_file_per_table = ON — default)       │
│                                                                │
│  Each table gets its own .ibd file:                            │
│  /var/lib/mysql/mydb/users.ibd        ← users table data     │
│  /var/lib/mysql/mydb/orders.ibd       ← orders table data    │
│  /var/lib/mysql/mydb/products.ibd     ← products table data  │
│                                                                │
│  ✅ Advantages:                                                │
│  • DROP TABLE / TRUNCATE reclaims disk space immediately       │
│  • Can place tables on different disks (symlinks)              │
│  • Easier to manage individual tables                          │
│  • Table-level compression possible                            │
│                                                                │
├──────────────────────────────────────────────────────────────┤
│  SYSTEM TABLESPACE (ibdata1) — legacy approach                 │
│                                                                │
│  Everything in one giant file:                                  │
│  /var/lib/mysql/ibdata1        ← ALL table data + metadata     │
│                                                                │
│  ❌ Problem: ibdata1 only GROWS, never SHRINKS.                │
│     Even after DROP TABLE, the space is NOT freed.             │
│     Only way to reclaim: dump → delete → rebuild. Painful.     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔥 11. MySQL Threading Model

```
┌──────────────────────────────────────────────────────────────┐
│  MySQL Community Edition: Thread-per-Connection               │
│                                                                │
│  Client 1 ──→ Thread 1                                        │
│  Client 2 ──→ Thread 2                                        │
│  Client 3 ──→ Thread 3                                        │
│  ...                                                           │
│  Client N ──→ Thread N                                        │
│                                                                │
│  Problem: 10,000 connections = 10,000 threads                  │
│           Each thread uses ~256KB-1MB of stack                  │
│           Context switching becomes expensive                   │
│                                                                │
│  Solution: Connection Pooling (ProxySQL, HikariCP, etc.)       │
│            OR MySQL Enterprise Thread Pool                      │
│                                                                │
├──────────────────────────────────────────────────────────────┤
│  MySQL Enterprise Edition: Thread Pool                         │
│                                                                │
│  Client 1 ──┐                                                  │
│  Client 2 ──┤                                                  │
│  Client 3 ──┼──→ Worker Thread Pool (limited threads)          │
│  ...        │    Threads are reused, not dedicated              │
│  Client N ──┘                                                  │
│                                                                │
│  Much better for high-concurrency workloads.                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```sql
-- Check current connections
SHOW STATUS LIKE 'Threads%';
-- Threads_connected: current active connections
-- Threads_running: currently executing queries
-- Threads_cached: threads in cache for reuse
-- Threads_created: total threads created (high = bad caching)

-- Key connection settings
SHOW VARIABLES LIKE 'max_connections';        -- Default: 151
SHOW VARIABLES LIKE 'thread_cache_size';      -- Cache idle threads for reuse
```

---

## 🧠 Quick Recall — Chapter Summary

| Concept | One-Line Summary |
|---------|-----------------|
| MySQL Architecture | Layered: Client → Connection → SQL → Storage Engine → Disk |
| Pluggable Engines | MySQL's unique feature — swap storage engines per table |
| InnoDB | Default engine — ACID, row-locking, MVCC, crash recovery |
| MyISAM | Legacy engine — table-locking, no transactions, fast COUNT(*) |
| Buffer Pool | InnoDB's RAM cache — set to 70-80% of total RAM |
| Redo Log | Write-Ahead Log for crash recovery (guarantees Durability) |
| Undo Log | Stores old row versions for rollback & MVCC |
| Doublewrite Buffer | Prevents torn/partial page writes during crash |
| `innodb_flush_log_at_trx_commit` | 1 = safest (fsync per commit), 0 = fastest, 2 = compromise |
| Clustered Index | InnoDB stores rows inside the PK B+Tree |
| File-Per-Table | Each table gets its own .ibd file (default since 5.6) |
| Threading Model | Thread-per-connection (Community) or Thread Pool (Enterprise) |

---

## ❓ Self-Check Questions

1. Draw MySQL's architecture layers from memory. What does each layer do?
2. Why was the Query Cache removed in MySQL 8.0?
3. List 5 advantages of InnoDB over MyISAM.
4. What is the buffer pool and what is the golden rule for sizing it?
5. Explain the difference between redo log and undo log.
6. What problem does the doublewrite buffer solve? How?
7. What does `innodb_flush_log_at_trx_commit = 1` do? What are the other options?
8. Why should you avoid UUID primary keys with InnoDB? What's the workaround?
9. What is MVCC and how does InnoDB implement it using undo logs?
10. When (if ever) would you choose MyISAM over InnoDB?

---

## 🎯 Hands-On Lab

```sql
-- 1. Check your MySQL version and default engine
SELECT VERSION();
SHOW ENGINES;

-- 2. Check buffer pool configuration
SHOW VARIABLES LIKE 'innodb_buffer_pool%';

-- 3. Check redo log configuration
SHOW VARIABLES LIKE 'innodb_redo_log_capacity';    -- MySQL 8.0.30+
SHOW VARIABLES LIKE 'innodb_log_file%';            -- Pre-8.0.30

-- 4. Check flush behavior
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';

-- 5. Create tables with different engines and compare
CREATE TABLE test_innodb (id INT PRIMARY KEY, data VARCHAR(100)) ENGINE=InnoDB;
CREATE TABLE test_myisam (id INT PRIMARY KEY, data VARCHAR(100)) ENGINE=MyISAM;
CREATE TABLE test_memory (id INT PRIMARY KEY, data VARCHAR(100)) ENGINE=MEMORY;

-- 6. Check table status
SHOW TABLE STATUS WHERE Name LIKE 'test_%'\G

-- 7. Check buffer pool hit rate
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';

-- 8. Clean up
DROP TABLE test_innodb, test_myisam, test_memory;
```

---

> **Next Chapter** → [2D.2 — MySQL Installation & Configuration](./02-MySQL-Installation.md)
