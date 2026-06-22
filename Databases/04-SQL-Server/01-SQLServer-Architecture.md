# 🏗️ Chapter 2C.1 — SQL Server Architecture (In-Memory OLTP, Buffer Pool)

> **"SQL Server isn't just a database — it's an entire engine room. Understanding what's under the hood turns you from a user into a mechanic."**

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 1.3 (DBMS Architecture), Chapter 1.1 (What is a Database?)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand SQL Server's **layered architecture** like an engineer, not a tourist
- Know exactly what happens when you execute `SELECT * FROM Orders`
- Explain **Buffer Pool**, **Query Processor**, **Storage Engine** in an interview
- Understand **In-Memory OLTP (Hekaton)** — Microsoft's game-changing engine
- Know how SQL Server manages **memory, disk, threads, and locks**
- Draw the architecture diagram from memory

---

## 🧠 The Big Picture — What IS SQL Server?

SQL Server is **not** a single program. It's a collection of tightly integrated components working together:

```
╔══════════════════════════════════════════════════════════════════════╗
║                    MICROSOFT SQL SERVER                              ║
║                                                                      ║
║   ┌─────────────────────────────────────────────────────────────┐   ║
║   │                   PROTOCOL LAYER                             │   ║
║   │   (TDS - Tabular Data Stream Protocol)                       │   ║
║   │   TCP/IP │ Named Pipes │ Shared Memory                      │   ║
║   └──────────────────────────┬──────────────────────────────────┘   ║
║                              │                                       ║
║   ┌──────────────────────────▼──────────────────────────────────┐   ║
║   │                  RELATIONAL ENGINE                            │   ║
║   │            (a.k.a. Query Processor)                          │   ║
║   │                                                              │   ║
║   │   ┌──────────┐  ┌───────────┐  ┌──────────────────────┐    │   ║
║   │   │  Parser  │→ │ Algebrizer│→ │  Query Optimizer     │    │   ║
║   │   │          │  │           │  │  (Cost-Based)        │    │   ║
║   │   └──────────┘  └───────────┘  └──────────┬───────────┘    │   ║
║   │                                            │                │   ║
║   │                              ┌─────────────▼────────────┐   │   ║
║   │                              │   Query Executor         │   │   ║
║   │                              └─────────────┬────────────┘   │   ║
║   └────────────────────────────────────────────┼────────────────┘   ║
║                                                │                     ║
║   ┌────────────────────────────────────────────▼────────────────┐   ║
║   │                   STORAGE ENGINE                             │   ║
║   │                                                              │   ║
║   │   ┌────────────┐  ┌──────────────┐  ┌────────────────┐     │   ║
║   │   │ Access     │  │ Buffer       │  │ Transaction    │     │   ║
║   │   │ Methods    │  │ Manager      │  │ Manager        │     │   ║
║   │   └────────────┘  └──────────────┘  └────────────────┘     │   ║
║   │                                                              │   ║
║   │   ┌────────────┐  ┌──────────────┐  ┌────────────────┐     │   ║
║   │   │ Lock       │  │ Log          │  │ File           │     │   ║
║   │   │ Manager    │  │ Manager      │  │ Manager        │     │   ║
║   │   └────────────┘  └──────────────┘  └────────────────┘     │   ║
║   └──────────────────────────────────────────────────────────────┘   ║
║                              │                                       ║
║                    ┌─────────▼──────────┐                           ║
║                    │    DISK STORAGE     │                           ║
║                    │  .mdf │ .ndf │ .ldf │                           ║
║                    └────────────────────┘                            ║
╚══════════════════════════════════════════════════════════════════════╝
```

> 💡 **Real-World Analogy:** Think of SQL Server as a **5-star restaurant**:
> - **Protocol Layer** = The host who greets you at the door
> - **Query Processor** = The chef who reads your order & plans the recipe
> - **Storage Engine** = The kitchen staff who fetches ingredients & cooks
> - **Disk** = The pantry/warehouse where raw ingredients are stored
> - **Buffer Pool** = The prep counter — ingredients pulled out of storage for quick access

---

## 📡 1. The Protocol Layer — How Clients Connect

Before SQL Server processes a single query, it needs to **hear** it. That's the Protocol Layer's job.

### TDS (Tabular Data Stream)

SQL Server communicates using **TDS** — a Microsoft-designed protocol that wraps all client-server communication.

```
Client (SSMS, App, etc.)
       │
       │  TDS Packets (encrypted or plain)
       │
       ▼
┌────────────────────────────────────────────┐
│          SQL Server Network Interface       │
│               (SNI Layer)                   │
├────────────────────────────────────────────┤
│                                            │
│   ┌────────────┐  ┌──────────┐  ┌───────┐│
│   │  TCP/IP    │  │  Named   │  │Shared ││
│   │  (Default) │  │  Pipes   │  │Memory ││
│   │  Port 1433 │  │          │  │       ││
│   └────────────┘  └──────────┘  └───────┘│
│                                            │
└────────────────────────────────────────────┘
```

### Which Protocol & When?

| Protocol | Use Case | Speed | Network |
|----------|----------|-------|---------|
| **Shared Memory** | Client & Server on SAME machine | ⚡ Fastest | Local only |
| **TCP/IP** | Remote connections over network | 🏃 Fast | LAN/WAN/Internet |
| **Named Pipes** | Legacy apps on local network | 🐌 Slower | LAN only |

> 💡 **Pro Tip:** For local development, SQL Server defaults to **Shared Memory**. For production, always use **TCP/IP** on port **1433** (the default — change it for security!).

---

## 🧠 2. The Relational Engine (Query Processor) — The Brain

This is where SQL Server **thinks**. Every query you write passes through 4 stages here:

### Stage 1: Parser

```
Your Query: SELECT name, salary FROM employees WHERE dept = 'IT'
                                    │
                                    ▼
                            ┌──────────────┐
                            │   PARSER     │
                            │              │
                            │ ✅ Valid SQL? │
                            │ ✅ Syntax OK?│
                            │              │
                            │ Output:      │
                            │ Parse Tree   │
                            └──────────────┘
```

**What it does:**
- Checks SQL syntax (catches typos like `SELEC` instead of `SELECT`)
- Produces a **parse tree** — a structured representation of your query
- Does **NOT** check if table/column names exist (that's next)

### Stage 2: Algebrizer (Binding)

```
                Parse Tree
                    │
                    ▼
            ┌──────────────┐
            │  ALGEBRIZER  │
            │              │
            │ • Does table │
            │   "employees"│
            │   exist?     │
            │ • Does column│
            │   "dept"     │
            │   exist?     │
            │ • Data types │
            │   compatible?│
            │              │
            │ Output:      │
            │ Query Tree   │
            │ (with        │
            │  metadata)   │
            └──────────────┘
```

**What it does:**
- **Name resolution** — verifies tables, columns, schemas exist
- **Type checking** — ensures `WHERE salary > 'hello'` gets caught
- Produces a **Query Processor Tree** with all metadata attached

### Stage 3: Query Optimizer — The Genius

This is the **crown jewel** of SQL Server. Microsoft has invested decades into this component.

```
╔═══════════════════════════════════════════════════════════════╗
║                   QUERY OPTIMIZER                             ║
║                   (Cost-Based)                                ║
║                                                               ║
║   Input: "Get employees in IT dept"                          ║
║                                                               ║
║   ┌─────────────────────────────────────────────────────┐    ║
║   │  Option A:  Full Table Scan                          │    ║
║   │  Cost: 💰💰💰💰💰 (Read ALL 10 million rows)        │    ║
║   │                                                      │    ║
║   │  Option B:  Index Seek on dept_idx                   │    ║
║   │  Cost: 💰 (Jump directly to 'IT' rows)              │    ║
║   │                                                      │    ║
║   │  Option C:  Index Scan + Key Lookup                  │    ║
║   │  Cost: 💰💰 (Scan index, then fetch full rows)      │    ║
║   │                                                      │    ║
║   │  Winner: Option B! ✅                                │    ║
║   └─────────────────────────────────────────────────────┘    ║
║                                                               ║
║   Output: Execution Plan                                     ║
╚═══════════════════════════════════════════════════════════════╝
```

**What it does:**
- Generates **multiple candidate plans** (sometimes thousands!)
- Estimates the **cost** of each plan using **statistics**
- Picks the plan with the **lowest estimated cost**
- For simple queries, it uses a **Trivial Plan** (skip optimization)
- For complex queries, it explores **phases** (0, 1, 2) — going deeper only if needed

> 💡 **Pro Tip:** The optimizer doesn't always find the BEST plan — it finds a **good enough** plan quickly. Spending 10 minutes optimizing a plan for a 1-second query makes no sense.

### Stage 4: Query Executor

```
                Execution Plan
                    │
                    ▼
            ┌──────────────┐
            │   EXECUTOR   │
            │              │
            │ Follows the  │
            │ plan step by │
            │ step.        │
            │              │
            │ Calls the    │
            │ Storage      │
            │ Engine for   │
            │ actual data. │
            └──────────────┘
```

**What it does:**
- Executes the plan produced by the optimizer
- Calls the **Storage Engine** to fetch/modify data
- Streams results back to the client

### The Complete Query Lifecycle

```
                    "SELECT * FROM Orders WHERE total > 1000"
                                    │
                                    ▼
                            ┌──────────────┐
                     ┌──────│   PARSER     │
                     │      └──────┬───────┘
                     │             │ Parse Tree
                  Syntax     ┌────▼────────┐
                  Error! ✗   │ ALGEBRIZER  │
                             └────┬────────┘
                                  │ Query Tree
                             ┌────▼────────────────┐
                             │ PLAN CACHE           │
                             │ Plan exists? ────Yes──→ Reuse! ⚡
                             │ No? ↓                │
                             └────┬────────────────┘
                                  │
                             ┌────▼────────────┐
                             │ QUERY OPTIMIZER  │
                             │ Generate plan    │
                             │ Store in cache   │
                             └────┬────────────┘
                                  │ Execution Plan
                             ┌────▼────────────┐
                             │ QUERY EXECUTOR   │
                             │ Execute plan     │
                             └────┬────────────┘
                                  │
                                  ▼
                          Storage Engine
                          (fetch data from
                           Buffer Pool or Disk)
                                  │
                                  ▼
                            Results → Client
```

> 💡 **Plan Cache** is huge for performance. SQL Server caches compiled execution plans so the next time you run the same query, it skips optimization entirely!

---

## 💾 3. The Storage Engine — The Muscle

The Storage Engine is SQL Server's **workhorse**. It physically reads and writes data.

### 3.1 Access Methods

These are the algorithms SQL Server uses to find your data:

```
┌───────────────────────────────────────────────────┐
│               ACCESS METHODS                       │
├───────────────────────────────────────────────────┤
│                                                    │
│   HEAP SCAN         → Read every page (no index)  │
│   Clustered Index   → Data IS the index (B-Tree)  │
│   Index Seek        → Jump to exact key value     │
│   Index Scan        → Read entire index in order  │
│   Key Lookup        → From index → go to heap/    │
│                       clustered index for full row│
│   Bookmark Lookup   → Same as Key Lookup (legacy) │
│                                                    │
└───────────────────────────────────────────────────┘
```

### 3.2 Buffer Manager & Buffer Pool — The Memory King 👑

This is **THE most critical** component for SQL Server performance.

```
╔══════════════════════════════════════════════════════════════╗
║                     BUFFER POOL                              ║
║              (SQL Server's Main Memory Cache)                ║
║                                                              ║
║   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐        ║
║   │Page │ │Page │ │Page │ │Page │ │Page │ │Page │  ...     ║
║   │ 101 │ │ 205 │ │ 42  │ │ 999 │ │ 67  │ │ 301 │        ║
║   │dirty│ │clean│ │dirty│ │clean│ │clean│ │dirty│        ║
║   └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘        ║
║                                                              ║
║   Total Size: Up to 90%+ of server RAM                      ║
║   Page Size:  8 KB each (fixed)                             ║
║   Goal:       Keep HOT data in memory, avoid disk I/O       ║
║                                                              ║
║   dirty = modified but not yet written to disk              ║
║   clean = same as on disk (can be evicted freely)           ║
╚══════════════════════════════════════════════════════════════╝
```

#### How Buffer Pool Works (Step by Step)

```
Query: SELECT * FROM Products WHERE ProductID = 42

Step 1: Executor asks Buffer Manager for Page containing ProductID 42

Step 2: Buffer Manager checks Buffer Pool
        ┌─────────────────────────────────┐
        │ Is the page in Buffer Pool?      │
        │                                  │
        │   YES → Buffer Pool HIT ⚡      │
        │          Return page from memory │
        │          (microseconds)          │
        │                                  │
        │   NO  → Buffer Pool MISS 💾     │
        │          Read page from disk     │
        │          Place it in Buffer Pool │
        │          Then return it          │
        │          (milliseconds — 100x    │
        │           slower!)               │
        └─────────────────────────────────┘

Step 3: If Buffer Pool is FULL, evict a clean page (LRU algorithm)
        If all pages are dirty, trigger a CHECKPOINT first

Step 4: Return data to Query Executor
```

> 💡 **The Golden Rule:** SQL Server's #1 performance goal = **maximize Buffer Pool hit ratio**. If 99% of reads come from memory, your database flies. If 50% hit disk, it crawls.

#### Check Your Buffer Pool Hit Ratio

```sql
-- 🔍 Buffer Pool Hit Ratio (should be > 99% for OLTP)
SELECT 
    (a.cntr_value * 1.0 / b.cntr_value) * 100 AS BufferCacheHitRatio
FROM sys.dm_os_performance_counters a
JOIN sys.dm_os_performance_counters b 
    ON a.object_name = b.object_name
WHERE a.counter_name = 'Buffer cache hit ratio'
  AND b.counter_name = 'Buffer cache hit ratio base';

-- Result: 99.87%  ← Excellent! 🎉
-- Result: 85.00%  ← Problem! Need more RAM or better queries 🚨
```

### 3.3 Transaction Manager

Ensures **ACID** compliance for every operation:

```
BEGIN TRANSACTION
    UPDATE accounts SET balance = balance - 500 WHERE id = 1   -- Debit
    UPDATE accounts SET balance = balance + 500 WHERE id = 2   -- Credit
COMMIT TRANSACTION

What Transaction Manager does:
┌────────────────────────────────────────────────────────┐
│  1. Write BOTH operations to Transaction Log (.ldf)    │
│     BEFORE modifying data pages (Write-Ahead Logging)  │
│                                                        │
│  2. Acquire LOCKS on both rows                         │
│     (Nobody else can touch them during this)           │
│                                                        │
│  3. Modify data pages in Buffer Pool                   │
│     (Pages become "dirty")                             │
│                                                        │
│  4. On COMMIT: mark log records as committed           │
│                                                        │
│  5. On ROLLBACK: undo all changes using log records    │
│                                                        │
│  6. CHECKPOINT: periodically flush dirty pages to disk │
└────────────────────────────────────────────────────────┘
```

### 3.4 Lock Manager

Controls **concurrent access** so multiple users don't corrupt each other's data:

```
┌─────────────────────────────────────────────────────────────┐
│                    LOCK HIERARCHY                            │
│                                                              │
│   DATABASE                                                   │
│      │                                                       │
│      ├── TABLE         (coarsest — locks entire table)      │
│      │     │                                                 │
│      │     ├── PAGE    (locks 8KB page — ~100 rows)         │
│      │     │     │                                           │
│      │     │     ├── ROW / KEY  (finest — single row)       │
│      │     │     │                                           │
│      │     │     └── Most common for OLTP workloads ⭐     │
│      │     │                                                 │
│      │     └── Used when many rows affected                 │
│      │                                                       │
│      └── Used for DDL operations (ALTER TABLE, etc.)        │
│                                                              │
│   Lock Escalation: If too many row locks → escalate to      │
│   table lock (saves memory but reduces concurrency)          │
└─────────────────────────────────────────────────────────────┘
```

#### Lock Types

| Lock | Symbol | Meaning | When Used |
|------|--------|---------|-----------|
| **Shared (S)** | `S` | Read-only, multiple users can hold | `SELECT` queries |
| **Exclusive (X)** | `X` | Full write lock, no one else can access | `INSERT`, `UPDATE`, `DELETE` |
| **Update (U)** | `U` | Intending to modify, prevents deadlocks | `UPDATE` (seek phase) |
| **Intent Shared (IS)** | `IS` | "A row below me has S lock" | Table/Page level hint |
| **Intent Exclusive (IX)** | `IX` | "A row below me has X lock" | Table/Page level hint |
| **Schema (Sch)** | `Sch` | Protects table structure | `ALTER TABLE`, `DROP TABLE` |

### 3.5 Log Manager (Write-Ahead Logging — WAL)

```
╔═══════════════════════════════════════════════════════════╗
║              WRITE-AHEAD LOGGING (WAL)                    ║
║                                                           ║
║   RULE: Write to LOG first, THEN modify data              ║
║                                                           ║
║   Why? If server crashes AFTER log write but BEFORE       ║
║   data write → SQL Server replays the log on restart!    ║
║                                                           ║
║   ┌────────────┐                                         ║
║   │ Transaction │                                         ║
║   │  Log (.ldf) │   ← Sequential writes (FAST! ⚡)       ║
║   │             │   ← All changes recorded here FIRST    ║
║   └──────┬──────┘                                        ║
║          │                                                ║
║          │  Lazy Writer / Checkpoint                      ║
║          ▼                                                ║
║   ┌────────────┐                                         ║
║   │ Data Files  │   ← Random writes (slower)             ║
║   │  (.mdf/.ndf)│   ← Updated lazily (async)            ║
║   └────────────┘                                         ║
║                                                           ║
║   Recovery Process:                                       ║
║   1. REDO  → Replay committed transactions not on disk   ║
║   2. UNDO  → Rollback uncommitted transactions           ║
║                                                           ║
╚═══════════════════════════════════════════════════════════╝
```

---

## 📁 4. SQL Server File Structure — What's on Disk?

Every SQL Server database has **at minimum** two files:

```
┌────────────────────────────────────────────────────────────────┐
│                    DATABASE FILES                               │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│   📄 PRIMARY DATA FILE (.mdf)                                  │
│      • One per database (mandatory)                            │
│      • Contains system tables, metadata                        │
│      • Contains user data (tables, indexes)                    │
│      • Default location: C:\...\MSSQL\DATA\                   │
│                                                                 │
│   📄 SECONDARY DATA FILE(S) (.ndf)                             │
│      • Optional — for spreading data across disks              │
│      • Used in FILEGROUP strategies                            │
│      • Can have multiple .ndf files                            │
│                                                                 │
│   📄 TRANSACTION LOG FILE (.ldf)                               │
│      • One per database (mandatory)                            │
│      • Write-Ahead Log — records all changes                   │
│      • Used for recovery, replication, backup                  │
│      • ⚠️ Can grow HUGE if not managed properly               │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Pages & Extents — SQL Server's Storage Units

```
╔══════════════════════════════════════════════════════════════╗
║   SMALLEST UNIT: PAGE (8 KB)                                 ║
║                                                              ║
║   ┌──────────────────────────────────────┐                  ║
║   │  Page Header (96 bytes)               │                  ║
║   │  ┌─────────────────────────────────┐ │                  ║
║   │  │  Row 1                          │ │                  ║
║   │  │  Row 2                          │ │  ← Data rows    ║
║   │  │  Row 3                          │ │     stored here  ║
║   │  │  ...                            │ │                  ║
║   │  └─────────────────────────────────┘ │                  ║
║   │  Row Offset Array (bottom of page)   │                  ║
║   └──────────────────────────────────────┘                  ║
║                                                              ║
║   Total: 8,192 bytes per page                                ║
║   Max row size: 8,060 bytes (header + offset overhead)       ║
║                                                              ║
║   EXTENT = 8 contiguous pages = 64 KB                        ║
║                                                              ║
║   ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐       ║
║   │ P1  │ P2  │ P3  │ P4  │ P5  │ P6  │ P7  │ P8  │       ║
║   └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘       ║
║                                                              ║
║   Uniform Extent  = All 8 pages belong to SAME object       ║
║   Mixed Extent    = Pages from DIFFERENT objects (small      ║
║                     tables sharing space)                     ║
╚══════════════════════════════════════════════════════════════╝
```

### Page Types

| Page Type | Contents |
|-----------|----------|
| **Data** | Actual table rows (heap or clustered index leaf) |
| **Index** | Index rows (B-Tree internal/leaf nodes) |
| **Text/Image** | LOB data (varchar(max), varbinary(max), XML) |
| **GAM** | Global Allocation Map — tracks extent allocation |
| **SGAM** | Shared GAM — tracks mixed extents |
| **PFS** | Page Free Space — tracks space usage per page |
| **IAM** | Index Allocation Map — maps pages to an object |
| **BCM** | Bulk Changed Map — for bulk operations |
| **DCM** | Differential Changed Map — for differential backups |

---

## ⚡ 5. In-Memory OLTP (Hekaton) — The Speed Demon

Introduced in SQL Server 2014, **In-Memory OLTP** (codenamed **Hekaton** — Greek for "hundred" → 100x faster) is a completely different engine **inside** SQL Server.

### Why In-Memory OLTP?

```
Traditional Engine:                    In-Memory OLTP:
┌──────────────────────┐              ┌──────────────────────┐
│                      │              │                      │
│  Data lives on DISK  │              │  Data lives in RAM   │
│  Loaded into Buffer  │              │  No Buffer Pool!     │
│  Pool on demand      │              │  No latching!        │
│                      │              │  No locking!         │
│  Latches on pages    │              │  Lock-free data      │
│  Locks on rows       │              │  structures          │
│  Log writes for      │              │  Optimistic MVCC     │
│  every change        │              │  concurrency         │
│                      │              │                      │
│  Speed: 🏃           │              │  Speed: 🚀🚀🚀       │
│  (milliseconds)      │              │  (microseconds!)     │
└──────────────────────┘              └──────────────────────┘
```

### Architecture of In-Memory OLTP

```
╔══════════════════════════════════════════════════════════════════╗
║                   IN-MEMORY OLTP ENGINE                          ║
║                                                                  ║
║   ┌──────────────────────────────────────────────────────┐      ║
║   │              MEMORY-OPTIMIZED TABLES                  │      ║
║   │                                                       │      ║
║   │   ┌───────────┐    ┌──────────────┐                  │      ║
║   │   │  Hash     │    │  Range       │                  │      ║
║   │   │  Index    │    │  Index       │                  │      ║
║   │   │  (O(1))   │    │  (Bw-Tree)   │                  │      ║
║   │   └───────────┘    └──────────────┘                  │      ║
║   │                                                       │      ║
║   │   Data Format: NOT pages — individual rows linked    │      ║
║   │   Concurrency: Optimistic MVCC (no locks!)           │      ║
║   │   Durability: SCHEMA_AND_DATA or SCHEMA_ONLY         │      ║
║   └──────────────────────────────────────────────────────┘      ║
║                                                                  ║
║   ┌──────────────────────────────────────────────────────┐      ║
║   │          NATIVELY COMPILED STORED PROCS               │      ║
║   │                                                       │      ║
║   │   T-SQL → C Code → Machine Code (DLL)               │      ║
║   │   No interpretation at runtime!                       │      ║
║   │   Runs as native CPU instructions ⚡                 │      ║
║   └──────────────────────────────────────────────────────┘      ║
║                                                                  ║
║   ┌──────────────────────────────────────────────────────┐      ║
║   │              CHECKPOINT & RECOVERY                    │      ║
║   │                                                       │      ║
║   │   Data files + Delta files (not traditional pages)   │      ║
║   │   On restart: reload ALL data into memory            │      ║
║   └──────────────────────────────────────────────────────┘      ║
╚══════════════════════════════════════════════════════════════════╝
```

### Creating a Memory-Optimized Table

```sql
-- Step 1: Add a memory-optimized filegroup to the database
ALTER DATABASE AdventureWorks
ADD FILEGROUP InMemory_FG CONTAINS MEMORY_OPTIMIZED_DATA;

ALTER DATABASE AdventureWorks
ADD FILE (
    NAME = 'InMemory_Data',
    FILENAME = 'C:\Data\InMemory_Data'
) TO FILEGROUP InMemory_FG;

-- Step 2: Create a memory-optimized table
CREATE TABLE dbo.ShoppingCart
(
    CartID      INT          NOT NULL PRIMARY KEY NONCLUSTERED 
                             HASH WITH (BUCKET_COUNT = 1000000),
    UserID      INT          NOT NULL,
    ProductID   INT          NOT NULL,
    Quantity    INT          NOT NULL,
    AddedDate   DATETIME2    NOT NULL DEFAULT(SYSDATETIME()),

    INDEX ix_user HASH (UserID) WITH (BUCKET_COUNT = 500000)
)
WITH (
    MEMORY_OPTIMIZED = ON,          -- Lives in RAM! 🚀
    DURABILITY = SCHEMA_AND_DATA    -- Survives restart (logged)
    -- DURABILITY = SCHEMA_ONLY     -- Temp data only (even faster)
);
```

### Traditional vs In-Memory OLTP Comparison

| Feature | Traditional Engine | In-Memory OLTP |
|---------|-------------------|----------------|
| **Data Location** | Disk (cached in Buffer Pool) | RAM (always in memory) |
| **Page-Based** | Yes (8 KB pages) | No (row-based) |
| **Locking** | Pessimistic (locks & latches) | Optimistic (MVCC, lock-free) |
| **Concurrency** | Lock contention under load | Massive parallelism |
| **Row Size Limit** | 8,060 bytes | 8,060 bytes |
| **Index Types** | B-Tree, Hash, Columnstore | Hash, Bw-Tree (range) |
| **Stored Procs** | Interpreted T-SQL | Natively compiled to DLL |
| **Best For** | General workloads | High-throughput OLTP, session state, IoT |
| **Speed Boost** | Baseline | **10x to 100x** faster for eligible workloads |

> ⚠️ **Limitations:** Not all T-SQL features work with In-Memory OLTP. No `ALTER TABLE`, limited data types, no cross-database queries on memory-optimized tables.

---

## 🏢 6. SQL Server Services & Components

SQL Server isn't just the database engine. It's an ecosystem:

```
┌───────────────────────────────────────────────────────────────┐
│                  SQL SERVER SERVICES                           │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  🔵 SQL Server Database Engine (MSSQLSERVER)                  │
│     • The core — processes queries, stores data               │
│     • This is what "SQL Server" usually means                 │
│                                                                │
│  🟢 SQL Server Agent (SQLSERVERAGENT)                         │
│     • Job scheduler — automates tasks                         │
│     • Alerts, notifications, maintenance plans                │
│                                                                │
│  🟡 SQL Server Browser                                        │
│     • Helps clients find SQL Server instances on network      │
│     • Maps instance names to port numbers                     │
│                                                                │
│  🔴 SQL Server Integration Services (SSIS)                    │
│     • ETL tool — Extract, Transform, Load data                │
│                                                                │
│  🟣 SQL Server Reporting Services (SSRS)                      │
│     • Report generation and delivery                          │
│                                                                │
│  🟠 SQL Server Analysis Services (SSAS)                       │
│     • OLAP cubes, data mining, tabular models                 │
│                                                                │
│  ⚪ Full-Text Search                                           │
│     • Fast text search across large text columns              │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

---

## 🧪 7. SQL Server Memory Architecture — Deep Dive

```
╔══════════════════════════════════════════════════════════════════╗
║               SQL SERVER MEMORY LAYOUT                           ║
║                                                                  ║
║   ┌──────────────────────────────────────────────────────┐      ║
║   │                    BUFFER POOL                        │      ║
║   │            (Largest consumer — up to TBs)             │      ║
║   │                                                       │      ║
║   │   ┌──────────────┐  ┌────────────────┐               │      ║
║   │   │  Data Cache  │  │   Plan Cache   │               │      ║
║   │   │  (data pages)│  │   (exec plans) │               │      ║
║   │   └──────────────┘  └────────────────┘               │      ║
║   │                                                       │      ║
║   │   ┌──────────────┐  ┌────────────────┐               │      ║
║   │   │  Free Pages  │  │  Stolen Pages  │               │      ║
║   │   │              │  │ (internal use) │               │      ║
║   │   └──────────────┘  └────────────────┘               │      ║
║   └──────────────────────────────────────────────────────┘      ║
║                                                                  ║
║   ┌──────────────────────────────────────────────────────┐      ║
║   │              NON-BUFFER POOL MEMORY                   │      ║
║   │                                                       │      ║
║   │   • Thread stacks (each connection gets one)         │      ║
║   │   • CLR (.NET) memory                                │      ║
║   │   • Extended stored procedures                       │      ║
║   │   • Linked server providers                          │      ║
║   │   • In-Memory OLTP memory (separate!)                │      ║
║   └──────────────────────────────────────────────────────┘      ║
║                                                                  ║
║   Key Settings:                                                  ║
║   • max server memory = Cap on Buffer Pool (SET THIS! ⭐)       ║
║   • min server memory = Minimum reserved                        ║
║                                                                  ║
║   ⚠️ ALWAYS set max server memory! Leave 10-20% for OS.        ║
║      Example: 64 GB RAM → set max to ~52 GB                    ║
╚══════════════════════════════════════════════════════════════════╝
```

### Essential Memory DMVs

```sql
-- 💡 How much memory is SQL Server using?
SELECT 
    physical_memory_in_use_kb / 1024 AS PhysicalMemory_MB,
    virtual_address_space_committed_kb / 1024 AS VirtualCommitted_MB,
    page_fault_count
FROM sys.dm_os_process_memory;

-- 💡 What's in the Buffer Pool? (Which databases are hogging RAM?)
SELECT 
    DB_NAME(database_id) AS DatabaseName,
    COUNT(*) * 8 / 1024 AS BufferPool_MB
FROM sys.dm_os_buffer_descriptors
GROUP BY database_id
ORDER BY BufferPool_MB DESC;
```

---

## 🔄 8. SQL Server Instance Architecture

```
╔══════════════════════════════════════════════════════════════════╗
║                   SQL SERVER INSTANCE                            ║
║                                                                  ║
║   One physical server can run MULTIPLE instances:               ║
║                                                                  ║
║   ┌──────────────────┐  ┌──────────────────┐                   ║
║   │  Default Instance │  │  Named Instance  │                   ║
║   │  (MSSQLSERVER)    │  │  (MSSQLSERVER\   │                   ║
║   │                   │  │   DEV2026)       │                   ║
║   │  Port: 1433       │  │  Port: Dynamic   │                   ║
║   │                   │  │  (via SQL Browser)│                   ║
║   │  ┌─────────────┐ │  │  ┌─────────────┐ │                   ║
║   │  │ master      │ │  │  │ master      │ │                   ║
║   │  │ tempdb      │ │  │  │ tempdb      │ │                   ║
║   │  │ model       │ │  │  │ model       │ │                   ║
║   │  │ msdb        │ │  │  │ msdb        │ │                   ║
║   │  │ UserDB_1    │ │  │  │ UserDB_A    │ │                   ║
║   │  │ UserDB_2    │ │  │  │ UserDB_B    │ │                   ║
║   │  └─────────────┘ │  │  └─────────────┘ │                   ║
║   └──────────────────┘  └──────────────────┘                   ║
║                                                                  ║
║   Each instance has its OWN:                                    ║
║   • System databases (master, tempdb, model, msdb)              ║
║   • Memory allocation                                            ║
║   • Service account                                              ║
║   • TCP port                                                     ║
║   • Security configuration                                       ║
╚══════════════════════════════════════════════════════════════════╝
```

### System Databases — The Core Four

| Database | Purpose | Can You Delete It? |
|----------|---------|-------------------|
| **master** | Server config, logins, linked servers, all database metadata | ❌ NEVER (SQL Server won't start) |
| **tempdb** | Temp tables, sorting, joins, row versioning — rebuilt every restart | ❌ Rebuilt on every startup |
| **model** | Template for new databases — every `CREATE DATABASE` copies this | ❌ Required |
| **msdb** | SQL Agent jobs, alerts, backup history, SSIS packages | ❌ Required for Agent |
| **Resource** | Hidden, read-only — contains system objects (sys.* views) | ❌ Hidden — you can't even see it normally |

---

## 🧠 Quick Recall — Chapter Summary

| Component | One-Line Summary |
|-----------|-----------------|
| Protocol Layer | Client-server communication via TDS over TCP/IP, Named Pipes, or Shared Memory |
| Parser | Checks SQL syntax, produces parse tree |
| Algebrizer | Validates names/types, resolves metadata |
| Optimizer | Cost-based engine — picks the cheapest execution plan |
| Executor | Runs the plan, coordinates with Storage Engine |
| Buffer Pool | Giant memory cache for data pages — the #1 performance factor |
| Buffer Manager | Reads pages from disk into Buffer Pool (or serves from cache) |
| Lock Manager | Controls concurrent access via shared/exclusive/update locks |
| Log Manager | Write-Ahead Logging — ensures crash recovery |
| Transaction Manager | Enforces ACID — commit, rollback, checkpoint |
| In-Memory OLTP | Lock-free, latch-free in-memory engine — 10-100x faster for OLTP |
| Data Files (.mdf/.ndf) | Store actual data & indexes on disk |
| Log Files (.ldf) | Sequential transaction log for recovery |
| System Databases | master, tempdb, model, msdb — SQL Server's brain |

---

## ❓ Self-Check Questions

1. What are the 3 layers of SQL Server architecture?
2. What happens when you execute a `SELECT` query — trace the full path from client to data and back.
3. What is the Buffer Pool and why is it critical for performance?
4. Explain the difference between a **dirty page** and a **clean page**.
5. What is Write-Ahead Logging (WAL) and why does SQL Server use it?
6. What are the 4 system databases and what does each do?
7. Why is `tempdb` rebuilt on every restart?
8. What is In-Memory OLTP and when should you use it?
9. What lock types does SQL Server use and when?
10. How does SQL Server's Query Optimizer decide which plan to use?

---

## 🎯 Hands-On Challenge

```sql
-- Run these on your SQL Server to SEE the architecture in action:

-- 1. Check SQL Server version and edition
SELECT @@VERSION;

-- 2. See all databases on your instance
SELECT name, state_desc, recovery_model_desc 
FROM sys.databases;

-- 3. Check Buffer Pool usage per database
SELECT DB_NAME(database_id) AS DB, 
       COUNT(*) * 8 / 1024 AS BufferPool_MB
FROM sys.dm_os_buffer_descriptors
GROUP BY database_id
ORDER BY BufferPool_MB DESC;

-- 4. See what's in the Plan Cache
SELECT TOP 10 
    cp.objtype, cp.cacheobjtype, cp.size_in_bytes / 1024 AS SizeKB,
    cp.usecounts, st.text AS QueryText
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
ORDER BY cp.usecounts DESC;

-- 5. Check if In-Memory OLTP is supported
SELECT SERVERPROPERTY('IsXTPSupported') AS InMemoryOLTP_Supported;
```

---

> **Next Chapter** → [2C.2 — SQL Server Installation & Tools (SSMS, Azure Data Studio)](./02-SQLServer-Installation.md)

---

> *"Understanding SQL Server's architecture is like understanding a car engine — you don't need it to drive, but you need it to win races."*
