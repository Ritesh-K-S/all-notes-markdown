# 🏛️ Chapter 2B.1 — Oracle Architecture & Memory (SGA, PGA, Processes)

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 1.3 (DBMS Architecture)

> **"Oracle doesn't just store data — it runs an entire city inside your server. Understanding its architecture is like getting the blueprint of that city."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand the difference between an Oracle **Instance** and a **Database** (most people confuse this)
- Know every component of the **SGA** (System Global Area) and why each matters
- Understand the **PGA** (Program Global Area) and its role
- Know all the **background processes** Oracle runs (and what happens if one dies)
- Understand **Redo Logs**, **Undo Segments**, and **Datafiles** — the holy trinity of Oracle storage
- Draw the full Oracle architecture diagram from memory (yes, you'll be able to)

---

## 1. The #1 Thing Everyone Gets Wrong: Instance vs Database

This is **THE** most fundamental concept in Oracle, and the #1 interview question.

### Simple Analogy

```
Think of a RESTAURANT:

🏪 The BUILDING (kitchen, tables, equipment)  = DATABASE (files on disk)
👨‍🍳 The STAFF (chefs, waiters, manager)         = INSTANCE (memory + processes)

• The building exists even when the restaurant is CLOSED (database files exist on disk)
• The staff only works when the restaurant is OPEN (instance starts/stops)
• You can have the building without staff (database mounted but not open)
• You CANNOT have staff without a building (instance needs a database)
• In Oracle RAC: ONE building, MULTIPLE teams of staff (multiple instances, one database)
```

### Technical Definition

```
╔══════════════════════════════════════════════════════════════════╗
║                    ORACLE = INSTANCE + DATABASE                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  ┌──────────────────────────────────────┐                       ║
║  │          INSTANCE (Memory + CPU)      │   ← Lives in RAM     ║
║  │                                       │   ← Created at       ║
║  │   ┌─────────────────────────────┐    │      STARTUP          ║
║  │   │   SGA (Shared Memory)       │    │   ← Destroyed at     ║
║  │   │   • Buffer Cache            │    │      SHUTDOWN         ║
║  │   │   • Shared Pool             │    │                       ║
║  │   │   • Redo Log Buffer         │    │                       ║
║  │   │   • Large Pool              │    │                       ║
║  │   │   • Java Pool               │    │                       ║
║  │   │   • Streams Pool            │    │                       ║
║  │   └─────────────────────────────┘    │                       ║
║  │                                       │                       ║
║  │   ┌─────────────────────────────┐    │                       ║
║  │   │   Background Processes       │    │                       ║
║  │   │   DBWn, LGWR, CKPT, SMON,  │    │                       ║
║  │   │   PMON, ARCn, MMON, RECO    │    │                       ║
║  │   └─────────────────────────────┘    │                       ║
║  └──────────────────────────────────────┘                       ║
║                        │                                         ║
║                        ▼ reads/writes                            ║
║  ┌──────────────────────────────────────┐                       ║
║  │          DATABASE (Files on Disk)     │   ← Lives on DISK    ║
║  │                                       │   ← Survives         ║
║  │   • Data Files (.dbf)                │      SHUTDOWN         ║
║  │   • Redo Log Files (.log)            │   ← Persists          ║
║  │   • Control Files (.ctl)             │      forever          ║
║  │   • Temp Files                        │                       ║
║  │   • Archive Log Files                 │                       ║
║  │   • Parameter File (spfile/pfile)     │                       ║
║  │   • Password File                     │                       ║
║  └──────────────────────────────────────┘                       ║
╚══════════════════════════════════════════════════════════════════╝
```

### Key Facts to Remember

| Aspect | Instance | Database |
|--------|----------|----------|
| **Lives in** | RAM (Memory) | Disk (Storage) |
| **Created at** | STARTUP | CREATE DATABASE |
| **Destroyed at** | SHUTDOWN | DROP/delete files |
| **Identified by** | SID / Instance Name | DB_NAME |
| **Contains** | Memory structures + Processes | Physical files |
| **Can exist alone?** | No (needs database files) | Yes (files persist without instance) |
| **Relationship** | 1 instance → 1 database (standalone) | 1 database ← many instances (RAC) |

> 💡 **Interview Gold**: "An Oracle Instance is the combination of memory structures (SGA) and background processes. The Database is the set of physical files on disk. Together, they form the Oracle Database Server."

---

## 2. SGA — The System Global Area (Oracle's Brain)

The SGA is a **shared memory region** allocated when the instance starts. It's shared by ALL users connected to the database. Think of it as the **communal workspace** where Oracle does the heavy lifting.

```
┌──────────────────────────────────────────────────────────────────┐
│                         SGA (System Global Area)                  │
│                                                                   │
│  ┌─────────────────────┐  ┌──────────────────────────────────┐  │
│  │                     │  │                                  │  │
│  │   DATABASE BUFFER   │  │         SHARED POOL              │  │
│  │       CACHE         │  │                                  │  │
│  │                     │  │  ┌─────────────┐ ┌───────────┐  │  │
│  │  Cached data blocks │  │  │ Library     │ │ Data      │  │  │
│  │  from disk          │  │  │ Cache       │ │ Dictionary│  │  │
│  │                     │  │  │ (SQL cache) │ │ Cache     │  │  │
│  │  THE #1 performance │  │  └─────────────┘ └───────────┘  │  │
│  │  component          │  │                                  │  │
│  │                     │  │  Parsed SQL, execution plans,   │  │
│  └─────────────────────┘  │  metadata                       │  │
│                            └──────────────────────────────────┘  │
│                                                                   │
│  ┌─────────────────────┐  ┌──────────────────┐                   │
│  │  REDO LOG BUFFER    │  │   LARGE POOL     │                   │
│  │                     │  │                  │                   │
│  │  Change records     │  │  RMAN backup     │                   │
│  │  before writing     │  │  Shared servers  │                   │
│  │  to redo log files  │  │  Parallel exec   │                   │
│  └─────────────────────┘  └──────────────────┘                   │
│                                                                   │
│  ┌─────────────────────┐  ┌──────────────────┐                   │
│  │   JAVA POOL         │  │  STREAMS POOL    │                   │
│  │                     │  │                  │                   │
│  │  Java stored procs  │  │  Oracle Streams  │                   │
│  │  JVM memory         │  │  / GoldenGate    │                   │
│  └─────────────────────┘  └──────────────────┘                   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  RESULT CACHE (11g+) — Stores query results for reuse      │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

---

### 2.1 Database Buffer Cache — The Speed Engine 🚀

This is the **single most important** SGA component for performance.

#### What It Does

Oracle **never reads data directly from disk to give to your query**. Instead:

```
Step-by-step: What happens when you SELECT * FROM employees WHERE id = 100

1. Oracle checks: "Is this data block already in the Buffer Cache?"
   
   ├─ YES (Cache Hit) ──→ Return data instantly from memory ⚡
   │                       (microseconds — 1000x faster than disk)
   │
   └─ NO (Cache Miss) ──→ Read block from disk into Buffer Cache
                           Then return from memory
                           Block stays cached for future queries
```

#### How It Works — The LRU Algorithm

```
Buffer Cache uses a modified LRU (Least Recently Used) algorithm:

┌──────────────────────────────────────────────────────────────┐
│                    BUFFER CACHE                               │
│                                                               │
│  HOT END (MRU)                          COLD END (LRU)       │
│  ◄── frequently accessed ──────── rarely accessed ──►        │
│                                                               │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐   │
│  │BLK-A│BLK-F│BLK-C│BLK-H│BLK-D│BLK-G│BLK-B│BLK-E│BLK-I│   │
│  │ HOT │ HOT │WARM │WARM │WARM │COOL │COOL │COLD │COLD │   │
│  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘   │
│       ▲                                           │           │
│       │         New blocks enter here ────────────┘           │
│       │         and move left as they get accessed            │
│       │                                                       │
│       └── Blocks accessed again move back to HOT end          │
│                                                               │
│  When cache is full: Evict blocks from COLD end               │
│  "Dirty" blocks (modified data) → Written to disk first       │
└──────────────────────────────────────────────────────────────┘
```

#### Buffer States

| State | Meaning | What Happens |
|-------|---------|--------------|
| **Free** | Empty buffer, never used | Can be used immediately |
| **Clean (Pinned)** | Has data, currently being read | Cannot be evicted |
| **Clean (Unpinned)** | Has data, not in use | Can be evicted (but would prefer not to) |
| **Dirty** | Modified but not written to disk | Must be written to disk before eviction |

#### Key Parameters

```sql
-- Check buffer cache size
SHOW PARAMETER db_cache_size;

-- Check buffer cache hit ratio (should be > 95%)
SELECT 1 - (physical_reads / (db_block_gets + consistent_gets)) AS hit_ratio
FROM v$sysstat
WHERE name IN ('physical reads', 'db block gets', 'consistent gets');
```

> 💡 **Pro Tip**: A buffer cache hit ratio below 90% is a red flag. Either your cache is too small, or your queries are scanning too many blocks (full table scans on large tables).

---

### 2.2 Shared Pool — The SQL Brain

The Shared Pool is where Oracle's **intelligence** lives. It caches SQL statements, execution plans, and metadata.

#### Two Main Components

```
┌──────────────────────────────────────────────────┐
│                  SHARED POOL                      │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │            LIBRARY CACHE                     │ │
│  │                                              │ │
│  │  ┌───────────────────┐ ┌──────────────────┐ │ │
│  │  │  Shared SQL Area  │ │  Private SQL     │ │ │
│  │  │                   │ │  Area (Cursors)  │ │ │
│  │  │  • Parsed SQL     │ │                  │ │ │
│  │  │  • Execution Plan │ │  • Bind values   │ │ │
│  │  │  • Parse Tree     │ │  • Session state │ │ │
│  │  └───────────────────┘ └──────────────────┘ │ │
│  │                                              │ │
│  │  Also: PL/SQL code, Java classes, Locks     │ │
│  └─────────────────────────────────────────────┘ │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │         DATA DICTIONARY CACHE                │ │
│  │         (Row Cache)                          │ │
│  │                                              │ │
│  │  Cached metadata:                            │ │
│  │  • Table definitions                         │ │
│  │  • Column info (names, types)                │ │
│  │  • User privileges                           │ │
│  │  • Constraint definitions                    │ │
│  │  • Object dependencies                       │ │
│  │                                              │ │
│  │  Without this, Oracle would hit disk for     │ │
│  │  EVERY metadata lookup — incredibly slow!    │ │
│  └─────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

#### Why the Library Cache Matters — Hard Parse vs Soft Parse

This is one of the **most critical performance concepts** in Oracle:

```
SQL Query Arrives: SELECT * FROM employees WHERE dept_id = 10

Step 1: Hash the SQL text → Generate hash value
Step 2: Search Library Cache for matching hash

├── FOUND (Soft Parse) ──→ Reuse existing execution plan ⚡
│                           Cost: ~microseconds
│                           CPU: minimal
│
└── NOT FOUND (Hard Parse) ──→ Full compilation:
                                1. Syntax check
                                2. Semantic check (do tables exist?)
                                3. Check privileges
                                4. Optimize (generate execution plans)
                                5. Choose best plan
                                6. Store in Library Cache
                                
                                Cost: ~milliseconds (100x+ slower)
                                CPU: HEAVY (locks latches)
```

#### The Bind Variable Lesson (CRITICAL!)

```sql
-- ❌ BAD: Different SQL text each time → Hard parse EVERY time
SELECT * FROM employees WHERE id = 100;   -- Hard parse #1
SELECT * FROM employees WHERE id = 200;   -- Hard parse #2 (different text!)
SELECT * FROM employees WHERE id = 300;   -- Hard parse #3
-- Oracle treats these as 3 DIFFERENT SQL statements!

-- ✅ GOOD: Same SQL text with bind variables → Hard parse ONCE
SELECT * FROM employees WHERE id = :emp_id;
-- Parse once, execute thousands of times with different values
-- This is WHY bind variables are MANDATORY in production
```

> 🔥 **Real-World Horror Story**: A developer hardcoded customer IDs in SQL strings. The shared pool filled with millions of unique SQL statements. The database crashed under `ORA-04031: unable to allocate shared memory`. Fix: Use bind variables. Always.

#### Key Parameters

```sql
-- Check shared pool size
SHOW PARAMETER shared_pool_size;

-- Check library cache hit ratio
SELECT SUM(pins) "Executions",
       SUM(reloads) "Cache Misses",
       1 - SUM(reloads)/SUM(pins) "Hit Ratio"
FROM v$librarycache;

-- See cached SQL statements
SELECT sql_text, executions, parse_calls, buffer_gets
FROM v$sql
ORDER BY executions DESC
FETCH FIRST 10 ROWS ONLY;
```

---

### 2.3 Redo Log Buffer — The Transaction Safety Net

```
Every single change Oracle makes to data is first recorded here.

┌──────────────────────────────────────────────────────┐
│                REDO LOG BUFFER                        │
│                                                       │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐ │
│  │Redo │Redo │Redo │Redo │Redo │     │     │     │ │
│  │Entry│Entry│Entry│Entry│Entry│FREE │FREE │FREE │ │
│  │ #1  │ #2  │ #3  │ #4  │ #5  │     │     │     │ │
│  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘ │
│      │                                                │
│      │  LGWR (Log Writer) flushes to disk when:      │
│      │  • COMMIT issued                               │
│      │  • Buffer 1/3 full                             │
│      │  • Every 3 seconds                             │
│      │  • Before DBWn writes dirty buffers            │
│      ▼                                                │
│  ┌──────────────────────────────────────────────┐    │
│  │  ONLINE REDO LOG FILES (on disk)              │    │
│  │  Group 1 ←→ Group 2 ←→ Group 3 (circular)    │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

#### What's in a Redo Entry?

Each redo entry contains the **change vector** — a description of the change:
- Which block was changed
- What was the old value
- What is the new value
- Timestamp and SCN (System Change Number)

> 💡 **Why It Matters**: If your server crashes at 3:00 AM, Oracle uses redo logs to **replay** all committed transactions that weren't yet written to data files. This is how Oracle guarantees **Durability** (the D in ACID).

#### Key Parameter

```sql
-- Typically 10-50 MB (small but critical)
SHOW PARAMETER log_buffer;
```

---

### 2.4 Large Pool, Java Pool, Streams Pool

| Pool | Purpose | When It's Used |
|------|---------|---------------|
| **Large Pool** | Optional area for memory-intensive operations | RMAN backup/restore, Shared Server connections, Parallel Query slaves |
| **Java Pool** | Memory for Oracle JVM | Java Stored Procedures, JDBC internal operations |
| **Streams Pool** | Oracle Streams / Advanced Queuing | GoldenGate, Change Data Capture, AQ messaging |
| **Result Cache** | Stores query results | Repeated identical queries return cached results instantly |

```sql
-- View SGA component sizes
SELECT component, current_size/1024/1024 AS size_mb
FROM v$sga_dynamic_components
ORDER BY current_size DESC;

-- View total SGA
SELECT * FROM v$sga;
```

---

## 3. PGA — The Program Global Area (Private Memory)

While the SGA is **shared** among all sessions, the PGA is **private** to each session.

```
                    ┌───────────────────────────────────────┐
                    │              SGA (Shared)              │
                    │   Shared Pool │ Buffer Cache │ Redo    │
                    └───────────────────────────────────────┘
                           ▲              ▲             ▲
                           │              │             │
                    ┌──────┴──────┐ ┌─────┴──────┐ ┌───┴─────────┐
                    │  Session 1  │ │  Session 2  │ │  Session 3  │
                    │  ┌───────┐  │ │  ┌───────┐  │ │  ┌───────┐  │
                    │  │ PGA 1 │  │ │  │ PGA 2 │  │ │  │ PGA 3 │  │
                    │  │       │  │ │  │       │  │ │  │       │  │
                    │  │•Sort  │  │ │  │•Sort  │  │ │  │•Sort  │  │
                    │  │ Area  │  │ │  │ Area  │  │ │  │ Area  │  │
                    │  │•Hash  │  │ │  │•Hash  │  │ │  │•Hash  │  │
                    │  │ Area  │  │ │  │ Area  │  │ │  │ Area  │  │
                    │  │•Session│ │ │  │•Session│ │ │  │•Session│ │
                    │  │ Data  │  │ │  │ Data  │  │ │  │ Data  │  │
                    │  │•Cursor│  │ │  │•Cursor│  │ │  │•Cursor│  │
                    │  │ State │  │ │  │ State │  │ │  │ State │  │
                    │  └───────┘  │ │  └───────┘  │ │  └───────┘  │
                    └─────────────┘ └─────────────┘ └─────────────┘
```

### What's Inside the PGA?

| Component | Purpose | Why It Matters |
|-----------|---------|---------------|
| **Sort Area** | Sorting data (ORDER BY, GROUP BY, DISTINCT) | If too small → spills to disk (TEMP tablespace) → SLOW |
| **Hash Area** | Hash joins, hash aggregations | Same — disk spill = performance killer |
| **Bitmap Merge Area** | Merging bitmap index results | Used for bitmap index operations |
| **Session Memory** | Session variables, login info, NLS settings | Fixed overhead per connection |
| **Cursor State** | Current execution state of SQL cursors | Bind variable values, fetch state |

### Key Parameters

```sql
-- Automatic PGA management (recommended)
SHOW PARAMETER pga_aggregate_target;  -- Total PGA for all sessions
SHOW PARAMETER pga_aggregate_limit;   -- Hard upper limit

-- Check actual PGA usage
SELECT name, value/1024/1024 AS mb
FROM v$pgastat
WHERE name IN ('total PGA allocated', 'total PGA inuse', 
               'maximum PGA allocated');
```

> 💡 **Pro Tip**: With `PGA_AGGREGATE_TARGET` set, Oracle automatically distributes PGA memory among sessions. A good starting point is **20% of total available RAM** for PGA, **60-70%** for SGA.

---

## 4. Background Processes — Oracle's Workforce

These are the **workers** that keep Oracle running 24/7. They start when the instance starts and run continuously.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   ORACLE BACKGROUND PROCESSES                        │
│                                                                      │
│  ┌──────────────────────── MANDATORY ────────────────────────────┐  │
│  │                                                                │  │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐          │  │
│  │  │ DBWn │  │ LGWR │  │ CKPT │  │ SMON │  │ PMON │          │  │
│  │  │      │  │      │  │      │  │      │  │      │          │  │
│  │  │Write │  │Write │  │Check-│  │System│  │Process│          │  │
│  │  │dirty │  │redo  │  │point │  │Moni- │  │Moni-  │          │  │
│  │  │blocks│  │to    │  │signal│  │tor   │  │tor    │          │  │
│  │  │to    │  │disk  │  │      │  │      │  │       │          │  │
│  │  │disk  │  │      │  │      │  │      │  │       │          │  │
│  │  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘          │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────── OPTIONAL / CONDITIONAL ───────────────────┐  │
│  │                                                                │  │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐          │  │
│  │  │ ARCn │  │ MMON │  │ MMNL │  │ RECO │  │ CJQ0 │          │  │
│  │  │      │  │      │  │      │  │      │  │      │          │  │
│  │  │Archi-│  │Manage│  │Manage│  │Recov-│  │Job   │          │  │
│  │  │ver   │  │ability│  │ability│  │erer │  │Queue │          │  │
│  │  │(copy │  │Moni- │  │Moni- │  │(dist-│  │Coord │          │  │
│  │  │redo  │  │tor   │  │tor   │  │ ribut│  │      │          │  │
│  │  │logs) │  │      │  │Light │  │ed TX)│  │      │          │  │
│  │  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘          │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.1 The Big Five — Mandatory Processes

#### DBWn (Database Writer)

```
Purpose: Write dirty (modified) buffers from Buffer Cache to data files on disk

    Buffer Cache                              Data Files
    ┌─────────────────┐                      ┌──────────────┐
    │ Clean │ Dirty!  │   ──── DBWn ────►    │  employees   │
    │ Block │ Block   │   writes dirty       │  .dbf        │
    │       │ (mod-   │   blocks to disk     │              │
    │       │ ified)  │                      │  orders      │
    └─────────────────┘                      │  .dbf        │
                                             └──────────────┘

When does DBWn write?
  • Checkpoint signal from CKPT
  • Dirty buffer threshold reached
  • No free buffers available
  • Timeout (every 3 seconds)
  • Tablespace set to READ ONLY or OFFLINE
```

> ⚠️ **Critical**: DBWn does NOT write on COMMIT! That's LGWR's job. DBWn writes lazily in the background. This is called **write-ahead logging** — redo is always written before data.

#### LGWR (Log Writer)

```
Purpose: Write redo entries from Redo Log Buffer to Online Redo Log files

    Redo Log Buffer                     Online Redo Logs
    ┌─────────────────┐                ┌──────────────────┐
    │ Redo │ Redo │    │  ── LGWR ──► │  redo01.log      │
    │ Ent  │ Ent  │    │  writes to   │  redo02.log      │
    │ #1   │ #2   │    │  disk        │  redo03.log      │
    └─────────────────┘                └──────────────────┘

When does LGWR write?
  ★ User issues COMMIT  ← Most important! (synchronous)
  • Redo Log Buffer is 1/3 full
  • Every 3 seconds
  • Before DBWn writes dirty buffers (Write-Ahead rule)
```

> 🔥 **The COMMIT Guarantee**: When you `COMMIT`, Oracle does NOT write your data to disk. It writes the REDO to disk. That's enough to guarantee recovery. The data gets written later by DBWn. This is why COMMITs are fast!

```
What happens at COMMIT:

1. User: COMMIT;
2. LGWR: Flush redo log buffer to redo log file on disk ← SYNCHRONOUS
3. Oracle: "COMMIT complete" returned to user
4. (Later, lazily) DBWn: Write actual dirty data blocks to data files

Why this works:
  Even if server crashes AFTER commit but BEFORE DBWn writes:
  → Oracle replays redo logs at startup → Data is recovered ✓
```

#### CKPT (Checkpoint Process)

```
Purpose: Signals DBWn to write dirty buffers and updates control file/data file headers

  CKPT tells Oracle: "Everything up to THIS point is safely on disk"

  Timeline of changes:
  ───────────────────────────────────────────────►
  SCN 100    SCN 200    SCN 300    SCN 400
              ▲                      ▲
              │                      │
         Last CKPT              Current CKPT
         
  At recovery: Oracle only needs to replay redo from
  the LAST checkpoint forward (not from the beginning of time)
```

#### SMON (System Monitor)

```
Purpose: The "janitor" — cleans up and recovers after crashes

What SMON does:
  ✅ Instance Recovery at startup (replay redo, rollback uncommitted)
  ✅ Coalesces free extents in dictionary-managed tablespaces
  ✅ Cleans up temporary segments no longer in use
  ✅ Recovers dead transactions in RAC
```

#### PMON (Process Monitor)

```
Purpose: The "nurse" — cleans up after failed user processes

What PMON does:
  ✅ Detects dead user processes (e.g., user killed SQL*Plus)
  ✅ Rolls back uncommitted transactions of dead processes
  ✅ Releases locks held by dead processes
  ✅ Frees PGA memory of dead processes
  ✅ Registers instance with Oracle Listener
```

### 4.2 Important Optional Processes

| Process | Full Name | What It Does |
|---------|-----------|-------------|
| **ARCn** | Archiver | Copies filled online redo logs to archive destination. Required for ARCHIVELOG mode (production). Without it, redo logs get overwritten = no point-in-time recovery! |
| **MMON** | Manageability Monitor | Collects AWR snapshots, checks metric thresholds, generates alerts. The performance monitoring backbone. |
| **MMNL** | Manageability Monitor Light | Flushes ASH (Active Session History) data from SGA to disk. Lightweight companion to MMON. |
| **RECO** | Recoverer | Resolves in-doubt distributed transactions (2PC). Important in distributed database setups. |
| **CJQ0** | Job Queue Coordinator | Manages DBMS_SCHEDULER and DBMS_JOB — Oracle's built-in cron. |
| **DBRM** | Database Resource Manager | Enforces resource plans (CPU limits, I/O limits per consumer group). |

---

## 5. Oracle Physical Storage — The Files That Matter

Every Oracle database ultimately lives as **files on disk**. Understanding these files is crucial.

```
┌──────────────────────────────────────────────────────────────────┐
│                    ORACLE DATABASE FILES                          │
│                                                                   │
│  ┌──────────────────── ESSENTIAL FILES ──────────────────────┐   │
│  │                                                            │   │
│  │  📁 DATA FILES (.dbf)                                     │   │
│  │     └─ Actual data: tables, indexes, LOBs                 │   │
│  │     └─ Organized in Tablespaces                           │   │
│  │     └─ Can be HUGE (terabytes each)                       │   │
│  │                                                            │   │
│  │  📁 REDO LOG FILES (.log)                                 │   │
│  │     └─ Record ALL changes (for crash recovery)            │   │
│  │     └─ Circular: Group 1 → Group 2 → Group 3 → Group 1   │   │
│  │     └─ MUST be multiplexed (at least 2 members/group)     │   │
│  │                                                            │   │
│  │  📁 CONTROL FILES (.ctl)                                  │   │
│  │     └─ Database metadata: name, creation time, redo info  │   │
│  │     └─ VERY small but VERY critical                       │   │
│  │     └─ MUST be multiplexed (3+ copies on different disks) │   │
│  │                                                            │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────── OTHER FILES ──────────────────────────┐   │
│  │                                                            │   │
│  │  📁 TEMP FILES         — Sort/hash operations spill here  │   │
│  │  📁 ARCHIVE LOGS       — Archived redo (for recovery)     │   │
│  │  📁 PARAMETER FILE     — spfile (binary) / pfile (text)   │   │
│  │  📁 PASSWORD FILE      — Remote SYSDBA authentication     │   │
│  │  📁 ALERT LOG          — Database diary (errors, events)  │   │
│  │  📁 TRACE FILES        — Diagnostic dumps                 │   │
│  │                                                            │   │
│  └────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### The Tablespace → Segment → Extent → Block Hierarchy

```
TABLESPACE (Logical container)
    │
    ├── SEGMENT (One per table/index/etc.)
    │       │
    │       ├── EXTENT (Contiguous blocks allocated together)
    │       │       │
    │       │       ├── BLOCK (Smallest unit — typically 8KB)
    │       │       ├── BLOCK
    │       │       ├── BLOCK
    │       │       └── BLOCK
    │       │
    │       ├── EXTENT
    │       │       ├── BLOCK
    │       │       └── BLOCK
    │       │
    │       └── EXTENT
    │
    └── SEGMENT (Another table or index)


Real-World Analogy:
  TABLESPACE = A building
  SEGMENT    = An apartment (one per tenant/table)
  EXTENT     = A room (group of blocks allocated at once)
  BLOCK      = A tile on the floor (smallest unit)
```

### Default Tablespaces Every Oracle DB Has

| Tablespace | Purpose | Can Be Dropped? |
|------------|---------|-----------------|
| **SYSTEM** | Data dictionary, core metadata | ❌ Never |
| **SYSAUX** | Auxiliary system data (AWR, ADDM, etc.) | ❌ Never |
| **UNDOTBS1** | Undo data (for rollback & read consistency) | ❌ (but can create new one) |
| **TEMP** | Temporary data (sorts, hashes) | ❌ (but can create new one) |
| **USERS** | Default tablespace for user data | ✅ Yes (but not recommended) |

```sql
-- View all tablespaces
SELECT tablespace_name, status, contents, 
       ROUND(SUM(bytes)/1024/1024) AS size_mb
FROM dba_data_files
GROUP BY tablespace_name, status, contents
ORDER BY size_mb DESC;

-- View data files
SELECT file_name, tablespace_name, 
       ROUND(bytes/1024/1024) AS size_mb, autoextensible
FROM dba_data_files;
```

---

## 6. Redo & Undo — The Twin Guardians of Data Integrity

These two mechanisms together make Oracle's ACID guarantees possible.

### Online Redo Logs — For Recovery (Roll Forward)

```
Redo answers: "WHAT CHANGED?"

Used for: CRASH RECOVERY (replaying committed transactions)

┌─────────────────────────────────────────────────────────────┐
│                 ONLINE REDO LOG GROUPS                        │
│                                                              │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐           │
│   │ Group 1  │────►│ Group 2  │────►│ Group 3  │──┐        │
│   │ CURRENT  │     │ ACTIVE   │     │ INACTIVE │  │        │
│   │          │     │          │     │          │  │        │
│   │ redo01a  │     │ redo02a  │     │ redo03a  │  │        │
│   │ redo01b  │     │ redo02b  │     │ redo03b  │  │        │
│   │(members) │     │(members) │     │(members) │  │        │
│   └──────────┘     └──────────┘     └──────────┘  │        │
│        ▲                                           │        │
│        └───────────────── circular ────────────────┘        │
│                                                              │
│   CURRENT = Being written to now                            │
│   ACTIVE  = Needed for crash recovery (checkpoint not done) │
│   INACTIVE = Not needed for crash recovery (safe to reuse)  │
│                                                              │
│   LOG SWITCH: When current group fills → move to next group │
│   CHECKPOINT: When CKPT tells DBWn to write dirty buffers   │
│               After checkpoint → group becomes INACTIVE      │
└─────────────────────────────────────────────────────────────┘
```

### Undo Segments — For Rollback (Roll Back)

```
Undo answers: "WHAT WAS THE OLD VALUE?"

Used for:
  1. ROLLBACK — Reverse uncommitted changes
  2. READ CONSISTENCY — Show other sessions the "before" image
  3. FLASHBACK — Look at data as it was in the past

┌──────────────────────────────────────────────────────────────┐
│  Session A: UPDATE employees SET salary = 90000              │
│             WHERE emp_id = 100;                              │
│             -- Old salary was 80000                          │
│                                                              │
│  ┌─────────────────────┐    ┌──────────────────────┐        │
│  │   UNDO SEGMENT      │    │   DATA BLOCK         │        │
│  │                      │    │                      │        │
│  │  Before Image:       │    │  Current Value:      │        │
│  │  emp_id=100          │    │  emp_id=100          │        │
│  │  salary=80000 (OLD)  │    │  salary=90000 (NEW)  │        │
│  └─────────────────────┘    └──────────────────────┘        │
│                                                              │
│  If Session A does ROLLBACK:                                │
│  → Oracle reads undo segment → Restores salary to 80000     │
│                                                              │
│  If Session B reads DURING Session A's uncommitted update:   │
│  → Oracle sees the block is modified                         │
│  → Goes to undo segment → Returns 80000 (read consistency!) │
│                                                              │
│  This is MVCC (Multi-Version Concurrency Control)           │
│  Readers NEVER block writers. Writers NEVER block readers.   │
└──────────────────────────────────────────────────────────────┘
```

### Redo vs Undo Summary

| Aspect | Redo | Undo |
|--------|------|------|
| **Records** | New values (what changed TO) | Old values (what changed FROM) |
| **Purpose** | Recovery (replay changes after crash) | Rollback + Read consistency |
| **Stored in** | Online Redo Log Files | Undo Tablespace (data files) |
| **Buffer** | Redo Log Buffer (SGA) | Managed in Buffer Cache |
| **Recovery action** | Roll Forward (redo committed changes) | Roll Back (undo uncommitted changes) |
| **When used** | Instance recovery, media recovery | ROLLBACK, consistent reads, Flashback |

---

## 7. Oracle Startup & Shutdown — The Lifecycle

### Startup Phases

```
                 SHUTDOWN → NOMOUNT → MOUNT → OPEN
                    │          │         │        │
                    │          │         │        │
                    ▼          ▼         ▼        ▼
                  Nothing    Instance  Control   Database
                  running    started   file      files
                             SGA       read      opened
                             allocated            Users can
                             Processes            connect!
                             started

STARTUP NOMOUNT:
  ✅ Read parameter file (spfile/pfile)
  ✅ Allocate SGA memory
  ✅ Start background processes
  ❌ No database files accessed yet
  📌 Use case: CREATE DATABASE, recreate control file

STARTUP MOUNT:
  ✅ Everything in NOMOUNT +
  ✅ Open and read control file
  ✅ Locate data files and redo log files (but don't open them)
  ❌ Users cannot access data yet
  📌 Use case: Rename data files, enable ARCHIVELOG, perform recovery

STARTUP OPEN (or just STARTUP):
  ✅ Everything in MOUNT +
  ✅ Open all data files and redo log files
  ✅ Perform instance recovery if needed (SMON)
  ✅ Database is ready for users!
  📌 Use case: Normal operation
```

### Startup Commands

```sql
-- Normal startup (goes through all phases)
STARTUP;

-- Startup in specific phase
STARTUP NOMOUNT;
ALTER DATABASE MOUNT;
ALTER DATABASE OPEN;

-- Force startup (abort any existing instance)
STARTUP FORCE;

-- Startup in restricted mode (only DBA can connect)
STARTUP RESTRICT;

-- Startup in read-only mode
STARTUP OPEN READ ONLY;
```

### Shutdown Modes

```
┌───────────────────────────────────────────────────────────────────┐
│                    SHUTDOWN MODES                                  │
│                                                                    │
│  Mode          │ New      │ Wait for  │ Wait for  │ Checkpoint  │
│                │ Connects │ Current   │ Current   │ Before      │
│                │ Allowed? │ Sessions  │ Txns to   │ Shutdown?   │
│                │          │ to End?   │ Complete? │             │
│  ──────────────┼──────────┼───────────┼───────────┼─────────────│
│  NORMAL        │  ❌ No   │  ✅ Yes   │  ✅ Yes   │  ✅ Yes     │
│  (default)     │          │ (waits!)  │           │             │
│                │          │           │           │             │
│  TRANSACTIONAL │  ❌ No   │  ❌ No    │  ✅ Yes   │  ✅ Yes     │
│                │          │(after txn)│           │             │
│                │          │           │           │             │
│  IMMEDIATE     │  ❌ No   │  ❌ No    │  ❌ No    │  ✅ Yes     │
│  ⭐ Most used  │          │           │ (rollback)│             │
│                │          │           │           │             │
│  ABORT         │  ❌ No   │  ❌ No    │  ❌ No    │  ❌ No!     │
│  ⚠️ Emergency  │          │           │           │ (recovery   │
│                │          │           │           │  needed!)   │
└───────────────────────────────────────────────────────────────────┘
```

```sql
-- Normal (waits for everyone to disconnect — could wait forever!)
SHUTDOWN NORMAL;

-- Transactional (waits for active transactions, then disconnects)
SHUTDOWN TRANSACTIONAL;

-- Immediate (rollback active transactions, disconnect, checkpoint)
SHUTDOWN IMMEDIATE;  -- ⭐ Use this in production

-- Abort (instant kill — like pulling the power plug)
SHUTDOWN ABORT;  -- ⚠️ Only in emergencies! Requires recovery at next startup
```

---

## 8. Oracle Memory Architecture — The Complete Picture

Here's how **everything** fits together:

```
╔══════════════════════════════════════════════════════════════════════════╗
║                        COMPLETE ORACLE ARCHITECTURE                      ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║   ┌────────────────────────────────────────────────────────────────┐    ║
║   │                     USER PROCESSES                              │    ║
║   │    App1        App2        SQL*Plus      JDBC Client            │    ║
║   └────┬───────────┬───────────┬─────────────┬─────────────────────┘    ║
║        │           │           │             │                          ║
║        │      Oracle Net (TNS/Listener)      │                          ║
║        │           │           │             │                          ║
║   ┌────▼───────────▼───────────▼─────────────▼─────────────────────┐    ║
║   │                   SERVER PROCESSES                               │    ║
║   │    Dedicated     Dedicated     Shared Server                     │    ║
║   │    Server 1      Server 2      Dispatcher                        │    ║
║   │    ┌─PGA─┐       ┌─PGA─┐       ┌─PGA─┐                        │    ║
║   │    └─────┘       └─────┘       └─────┘                        │    ║
║   └────────────────────────┬───────────────────────────────────────┘    ║
║                            │                                            ║
║                            ▼                                            ║
║   ┌──────────────────────────────────────────────────────────────────┐  ║
║   │                    SGA (System Global Area)                       │  ║
║   │                                                                   │  ║
║   │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐ │  ║
║   │  │Buffer Cache  │ │ Shared Pool  │ │ Redo Log Buffer         │ │  ║
║   │  │              │ │              │ │                          │ │  ║
║   │  │ Data blocks  │ │ SQL cache    │ │ Change vectors          │ │  ║
║   │  │ cached from  │ │ Dict cache   │ │ before writing to       │ │  ║
║   │  │ data files   │ │ Parse trees  │ │ redo log files          │ │  ║
║   │  └──────────────┘ └──────────────┘ └──────────────────────────┘ │  ║
║   │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐ │  ║
║   │  │ Large Pool   │ │ Java Pool    │ │ Streams Pool             │ │  ║
║   │  └──────────────┘ └──────────────┘ └──────────────────────────┘ │  ║
║   └──────────────────────────┬───────────────────────────────────────┘  ║
║                              │                                          ║
║   ┌──────────────────────────▼───────────────────────────────────────┐  ║
║   │                  BACKGROUND PROCESSES                             │  ║
║   │  DBWn  LGWR  CKPT  SMON  PMON  ARCn  MMON  MMNL  RECO  CJQ0   │  ║
║   └──────┬────────┬────────────────────────┬─────────────────────────┘  ║
║          │        │                        │                            ║
║          ▼        ▼                        ▼                            ║
║   ┌──────────────────────────────────────────────────────────────────┐  ║
║   │                    DATABASE (Physical Files)                      │  ║
║   │                                                                   │  ║
║   │   ┌────────────┐  ┌────────────┐  ┌────────────┐                │  ║
║   │   │ Data Files │  │ Redo Logs  │  │ Control    │                │  ║
║   │   │ (.dbf)     │  │ (.log)     │  │ Files(.ctl)│                │  ║
║   │   └────────────┘  └────────────┘  └────────────┘                │  ║
║   │   ┌────────────┐  ┌────────────┐  ┌────────────┐                │  ║
║   │   │ Temp Files │  │ Archive    │  │ Parameter  │                │  ║
║   │   │            │  │ Logs       │  │ File(spfile│                │  ║
║   │   └────────────┘  └────────────┘  └────────────┘                │  ║
║   └──────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════════╝
```

---

## 9. Quick Diagnostic Queries — Know Your Instance

```sql
-- 🔍 Instance information
SELECT instance_name, host_name, version_full, status, 
       startup_time, database_status
FROM v$instance;

-- 🔍 Database information
SELECT name, db_unique_name, open_mode, log_mode, 
       platform_name, created
FROM v$database;

-- 🔍 SGA breakdown
SELECT * FROM v$sga;

-- 🔍 SGA dynamic components (detailed)
SELECT component, current_size/1024/1024 AS current_mb,
       min_size/1024/1024 AS min_mb,
       max_size/1024/1024 AS max_mb
FROM v$sga_dynamic_components
WHERE current_size > 0
ORDER BY current_size DESC;

-- 🔍 PGA statistics
SELECT name, ROUND(value/1024/1024, 2) AS mb
FROM v$pgastat
WHERE name LIKE '%PGA%';

-- 🔍 Background processes running
SELECT name, description
FROM v$bgprocess
WHERE paddr != '00';

-- 🔍 Data files
SELECT file_name, tablespace_name, 
       ROUND(bytes/1024/1024) AS size_mb,
       autoextensible, status
FROM dba_data_files
ORDER BY tablespace_name;

-- 🔍 Redo log groups and members
SELECT l.group#, l.sequence#, l.bytes/1024/1024 AS size_mb,
       l.status, lf.member
FROM v$log l
JOIN v$logfile lf ON l.group# = lf.group#
ORDER BY l.group#;
```

---

## 🧠 Chapter Summary — Architecture Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│           ORACLE ARCHITECTURE — ONE-PAGE SUMMARY                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  INSTANCE = SGA (shared memory) + Background Processes           │
│  DATABASE = Physical files on disk                               │
│  ORACLE SERVER = Instance + Database                             │
│                                                                   │
│  SGA Components:                                                  │
│    Buffer Cache    → Cached data blocks (performance #1)         │
│    Shared Pool     → SQL cache + Data dictionary cache           │
│    Redo Log Buffer → Change records before disk write            │
│    Large/Java/Streams Pool → Specialized memory                  │
│                                                                   │
│  PGA = Private memory per session (sort, hash, cursor state)     │
│                                                                   │
│  Key Processes:                                                   │
│    DBWn → Writes dirty blocks to data files                      │
│    LGWR → Writes redo to disk (COMMIT guarantee!)                │
│    CKPT → Signals checkpoint (sync memory to disk)               │
│    SMON → Instance recovery + cleanup                            │
│    PMON → Process cleanup + listener registration                │
│    ARCn → Archives redo logs (for point-in-time recovery)        │
│                                                                   │
│  Physical Files:                                                  │
│    Data Files   → Your actual data                               │
│    Redo Logs    → Change history (for crash recovery)            │
│    Control File → Database metadata (multiplexed!)               │
│    Archive Logs → Copied redo logs (for full recovery)           │
│                                                                   │
│  Storage Hierarchy:                                               │
│    Tablespace → Segment → Extent → Block (8KB default)          │
│                                                                   │
│  Startup: SHUTDOWN → NOMOUNT → MOUNT → OPEN                     │
│  Shutdown: NORMAL / TRANSACTIONAL / IMMEDIATE⭐ / ABORT⚠️       │
│                                                                   │
│  Golden Rules:                                                    │
│    • LGWR writes at COMMIT (not DBWn!)                           │
│    • Write-Ahead Logging: Redo ALWAYS before data                │
│    • Use bind variables (avoid hard parses!)                     │
│    • Multiplex control files and redo logs!                       │
│    • Readers never block writers (MVCC via Undo)                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## ❓ Interview Questions You WILL Be Asked

| # | Question | Key Points |
|---|----------|-----------|
| 1 | What is the difference between an Oracle Instance and Database? | Instance = SGA + processes (memory). Database = files on disk. Instance is created/destroyed; database persists. |
| 2 | What are the main components of SGA? | Buffer Cache, Shared Pool (Library + Dict Cache), Redo Log Buffer, Large/Java/Streams Pool |
| 3 | What happens when a user issues COMMIT? | LGWR writes redo log buffer to disk. NOT DBWn. Commit returns after redo is safely on disk. |
| 4 | What is the difference between Redo and Undo? | Redo = new values, for recovery (roll forward). Undo = old values, for rollback + read consistency. |
| 5 | What are the mandatory background processes? | DBWn, LGWR, CKPT, SMON, PMON |
| 6 | What is a hard parse vs soft parse? | Hard = full SQL compilation (expensive). Soft = reuse cached plan (fast). Use bind variables! |
| 7 | What are the shutdown modes? | NORMAL, TRANSACTIONAL, IMMEDIATE (most used), ABORT (emergency) |
| 8 | What is the startup sequence? | NOMOUNT (read params, start SGA) → MOUNT (read control file) → OPEN (open data files) |
| 9 | What is Write-Ahead Logging? | Redo must be written to disk BEFORE data blocks are written. Guarantees recoverability. |
| 10 | What is MVCC in Oracle? | Uses undo segments to provide read-consistent views without blocking writers. |

---

> **Next Chapter**: [2B.2 — Oracle Installation & Configuration →](./02-Oracle-Installation.md)
