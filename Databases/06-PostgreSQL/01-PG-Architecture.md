# 2E.1 — PostgreSQL Architecture & Internals 🟡⭐

> **"PostgreSQL is not just a database — it's a 35+ year masterpiece of software engineering."**
> Understanding its internals separates the script-kiddie from the database engineer.

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 1.3 (DBMS Architecture), Chapter 1.5 (ACID vs BASE)

---

## 🎯 What You'll Master

```
✅ PostgreSQL process architecture (multi-process model)
✅ The Postmaster — the gatekeeper of everything
✅ Memory architecture — Shared Buffers, WAL Buffers, Work Mem
✅ Storage internals — Tablespaces, Pages, Tuples, TOAST
✅ Write-Ahead Logging (WAL) — the crash recovery secret
✅ MVCC — how PostgreSQL handles concurrency WITHOUT locking reads
✅ VACUUM — the janitor you MUST understand
✅ The life of a query — from SQL string to result set
✅ Why PostgreSQL is considered the most "correct" open-source RDBMS
```

---

## 🧠 Why PostgreSQL? — The 60-Second Pitch

```
┌───────────────────────────────────────────────────────────────────────┐
│                    WHY POSTGRESQL WINS                                │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  🏆  Most Advanced Open-Source Relational Database                   │
│  📅  Born 1986 at UC Berkeley (originally "POSTGRES")                │
│  🔒  ACID-compliant by default — no shortcuts                       │
│  🧬  Extensible — create custom types, operators, index methods      │
│  📊  MVCC — readers NEVER block writers                              │
│  🌍  Runs everywhere — Linux, Windows, macOS, Docker, K8s           │
│  💰  100% free — PostgreSQL License (most liberal OSS license)       │
│  🔥  Powers: Apple, Instagram, Spotify, Reddit, Discord, NASA       │
│                                                                       │
│  MySQL says: "I'm fast and simple"                                   │
│  PostgreSQL says: "I'm correct AND fast. And I have JSONB."          │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ 1. PostgreSQL Process Model — Multi-Process (NOT Multi-Threaded)

Unlike MySQL (which uses threads), PostgreSQL uses a **multi-process** architecture. Every client connection gets its own **dedicated OS process**.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     OPERATING SYSTEM                                 │
│                                                                      │
│   ┌───────────────────────────────────────────────────────────────┐  │
│   │                    POSTMASTER (PID 1)                         │  │
│   │         The "Parent" process — manages everything            │  │
│   │           • Listens on port 5432                             │  │
│   │           • Spawns backend processes for each client         │  │
│   │           • Spawns background worker processes               │  │
│   └───────────┬──────────────┬──────────────┬────────────────────┘  │
│               │              │              │                        │
│   ┌───────────▼──┐  ┌───────▼──────┐  ┌───▼──────────────────┐    │
│   │  Backend #1  │  │  Backend #2  │  │  Backend #3          │    │
│   │ (Client A)   │  │ (Client B)   │  │ (Client C)           │    │
│   │ - Parser     │  │ - Parser     │  │ - Parser             │    │
│   │ - Planner    │  │ - Planner    │  │ - Planner            │    │
│   │ - Executor   │  │ - Executor   │  │ - Executor           │    │
│   └──────────────┘  └──────────────┘  └──────────────────────┘    │
│                                                                      │
│   ┌──────────────────────────────────────────────────────────────┐   │
│   │             BACKGROUND PROCESSES (always running)            │   │
│   │                                                              │   │
│   │   📝 WAL Writer     — flushes WAL buffers to disk           │   │
│   │   💾 Checkpointer   — writes dirty pages to data files      │   │
│   │   🧹 Autovacuum     — cleans up dead tuples                 │   │
│   │   📊 Stats Collector— gathers table/index statistics        │   │
│   │   📋 BG Writer      — writes dirty shared buffers to disk   │   │
│   │   🔄 WAL Sender     — sends WAL to replicas (replication)   │   │
│   │   📥 WAL Receiver   — receives WAL on replica               │   │
│   │   📦 Archiver       — archives WAL files for PITR           │   │
│   │   🔍 Logical Repl   — logical replication workers           │   │
│   └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Why Multi-Process Instead of Multi-Thread?

| Aspect | Multi-Process (PostgreSQL) | Multi-Thread (MySQL/SQL Server) |
|--------|---------------------------|----------------------------------|
| **Crash Isolation** | One client crashes → only that process dies ✅ | One thread crash → entire server can die ❌ |
| **Memory Safety** | Each process has protected memory space | Threads share memory — bugs can corrupt data |
| **Context Switching** | Heavier (OS process switch) | Lighter (thread switch) |
| **Max Connections** | ~hundreds to low thousands (use pooling!) | Can handle more concurrent connections |
| **Debugging** | Easier (isolate by PID) | Harder (thread races) |

> 💡 **PRO TIP:** Because of the process-per-connection model, PostgreSQL can struggle with **thousands of connections**. This is why **PgBouncer** (connection pooler) is practically mandatory in production. We cover this in Chapter 2E.6.

---

### The Postmaster — The Boss

The **Postmaster** is the first process that starts. It:

1. **Reads configuration** (`postgresql.conf`, `pg_hba.conf`)
2. **Allocates shared memory** (shared buffers, WAL buffers, etc.)
3. **Starts background processes** (autovacuum, WAL writer, etc.)
4. **Listens on TCP port 5432** (default)
5. **Forks a new backend process** for every incoming client connection

```bash
# See PostgreSQL processes on Linux
ps aux | grep postgres

# Typical output:
postgres   1234  postmaster                  ← The boss
postgres   1235  checkpointer                ← Writes dirty pages
postgres   1236  background writer           ← Flushes shared buffers
postgres   1237  walwriter                   ← Flushes WAL buffers
postgres   1238  autovacuum launcher          ← Manages vacuum workers
postgres   1239  stats collector              ← Gathers statistics
postgres   1240  logical replication launcher
postgres   1241  postgres: user1 mydb [active] ← Backend for client
postgres   1242  postgres: user2 mydb [idle]   ← Backend (waiting)
```

---

## 🧠 2. Memory Architecture — Where the Speed Lives

PostgreSQL memory is split into **Shared Memory** (shared across all processes) and **Per-Process Memory** (private to each backend).

```
┌─────────────────────────────────────────────────────────────────┐
│                    SHARED MEMORY                                 │
│           (Allocated at server startup, shared by ALL)           │
│                                                                  │
│  ┌──────────────────────────────────────┐                       │
│  │        SHARED BUFFERS                │  ← The #1 performance │
│  │   (default: 128MB, tune to 25%      │     setting to tune!  │
│  │    of total RAM)                     │                       │
│  │   • Cache for data pages (8KB each) │                       │
│  │   • Reads hit here BEFORE disk      │                       │
│  │   • Dirty pages written by BG Writer│                       │
│  └──────────────────────────────────────┘                       │
│                                                                  │
│  ┌───────────────────┐  ┌───────────────────────────────┐      │
│  │   WAL BUFFERS     │  │   CLOG (Commit Log) Buffers   │      │
│  │  (default: 16MB)  │  │   • Transaction commit status │      │
│  │  • Buffer for WAL │  │   • Shared across processes   │      │
│  │    before flush   │  └───────────────────────────────┘      │
│  └───────────────────┘                                          │
│                                                                  │
│  ┌────────────────────────────────────────────┐                 │
│  │   LOCK SPACE                               │                 │
│  │   • Lightweight locks, heavyweight locks   │                 │
│  │   • Lock table for row/table locking       │                 │
│  └────────────────────────────────────────────┘                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│               PER-PROCESS (BACKEND) MEMORY                       │
│             (Private to each client connection)                  │
│                                                                  │
│  ┌──────────────────────────────────────┐                       │
│  │   work_mem (default: 4MB)            │                       │
│  │   • Memory for sorts, hash joins,   │                       │
│  │     hash aggregations               │                       │
│  │   ⚠️ PER OPERATION, not per query!  │                       │
│  │   A query with 5 sorts uses 5×4MB!  │                       │
│  └──────────────────────────────────────┘                       │
│                                                                  │
│  ┌──────────────────────────────────────┐                       │
│  │   maintenance_work_mem (default: 64MB)│                      │
│  │   • Memory for VACUUM, CREATE INDEX, │                       │
│  │     ALTER TABLE ADD FOREIGN KEY      │                       │
│  └──────────────────────────────────────┘                       │
│                                                                  │
│  ┌──────────────────────────────────────┐                       │
│  │   temp_buffers (default: 8MB)        │                       │
│  │   • Memory for temporary tables      │                       │
│  └──────────────────────────────────────┘                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Memory Settings Cheat Sheet

| Parameter | Default | Production Recommendation | What It Does |
|-----------|---------|---------------------------|--------------|
| `shared_buffers` | 128MB | **25% of total RAM** | Cache for data pages. THE most important setting |
| `effective_cache_size` | 4GB | **50-75% of total RAM** | Hint to planner about OS cache size (doesn't allocate!) |
| `work_mem` | 4MB | **16-256MB** (careful!) | Memory per sort/hash operation |
| `maintenance_work_mem` | 64MB | **256MB - 1GB** | Memory for VACUUM, index builds |
| `wal_buffers` | -1 (auto) | **64MB** | Buffer for WAL before disk flush |
| `temp_buffers` | 8MB | 8-32MB | Per-session temp table storage |

> ⚠️ **THE CLASSIC `work_mem` TRAP:**
> ```
> If work_mem = 256MB
> And you have 100 connections
> Each running a query with 3 sorts
> 
> Potential memory usage = 256MB × 100 × 3 = 75 GB!! 💀
> ```
> **Rule of thumb:** `work_mem = Total RAM / (max_connections × 3)`

---

## 💾 3. Storage Internals — How Data Lives on Disk

### The Storage Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                    DATABASE CLUSTER                          │
│         (One PostgreSQL instance = one cluster)              │
│              $PGDATA directory (e.g., /var/lib/postgresql)    │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              TABLESPACE (pg_default)                  │   │
│   │                                                      │   │
│   │   ┌──────────────┐  ┌──────────────┐               │   │
│   │   │  DATABASE:   │  │  DATABASE:   │               │   │
│   │   │  "myapp"     │  │  "analytics" │               │   │
│   │   │              │  │              │               │   │
│   │   │  ┌────────┐  │  │  ┌────────┐  │               │   │
│   │   │  │ TABLE  │  │  │  │ TABLE  │  │               │   │
│   │   │  │"users" │  │  │  │"events"│  │               │   │
│   │   │  │        │  │  │  │        │  │               │   │
│   │   │  │ File:  │  │  │  │ File:  │  │               │   │
│   │   │  │ 16389  │  │  │  │ 24601  │  │               │   │
│   │   │  └────────┘  │  │  └────────┘  │               │   │
│   │   └──────────────┘  └──────────────┘               │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │          TABLESPACE (fast_ssd)                       │   │
│   │    → On a separate SSD mount point                   │   │
│   │    → For high-performance tables/indexes             │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Page Structure — The 8KB Building Block

**Everything in PostgreSQL is stored in 8KB pages** (also called "blocks"). Each table file is a sequence of pages.

```
┌──────────────────────── 8KB PAGE ────────────────────────┐
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  PAGE HEADER (24 bytes)                              │ │
│  │  • pd_lsn      → Last WAL position that modified    │ │
│  │  • pd_checksum  → Page checksum (if enabled)        │ │
│  │  • pd_lower     → Offset to end of line pointers    │ │
│  │  • pd_upper     → Offset to start of free space     │ │
│  │  • pd_special   → Offset to special space           │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  LINE POINTERS (Item IDs) — 4 bytes each            │ │
│  │  [1] → offset 8100   (points to Tuple 1)            │ │
│  │  [2] → offset 7980   (points to Tuple 2)            │ │
│  │  [3] → offset 7850   (points to Tuple 3)            │ │
│  │  ... grows DOWNWARD ↓                               │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                                                      │ │
│  │              FREE SPACE                               │ │
│  │     (Line pointers grow down ↓                       │ │
│  │      Tuples grow up ↑)                               │ │
│  │                                                      │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  TUPLES (Heap Tuples = actual row data)             │ │
│  │                                                      │ │
│  │  Tuple 3: [header | col1 | col2 | col3 | ...]      │ │
│  │  Tuple 2: [header | col1 | col2 | col3 | ...]      │ │
│  │  Tuple 1: [header | col1 | col2 | col3 | ...]      │ │
│  │  ... grows UPWARD ↑                                 │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  SPECIAL SPACE (for index pages — B-tree metadata)  │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### Tuple (Row) Header — The Hidden 23 Bytes

Every row in PostgreSQL has a **hidden header** of at least 23 bytes:

```
┌────────────────── TUPLE HEADER (23+ bytes) ──────────────────┐
│                                                               │
│  t_xmin (4 bytes)  → Transaction ID that INSERTED this row  │
│  t_xmax (4 bytes)  → Transaction ID that DELETED/UPDATED    │
│  t_cid  (4 bytes)  → Command ID within the transaction      │
│  t_ctid (6 bytes)  → Current Tuple ID (page, offset)        │
│  t_infomask (2 bytes)  → Status bits (committed? aborted?)  │
│  t_infomask2 (2 bytes) → More flags + number of columns     │
│  t_hoff (1 byte)   → Offset to user data                    │
│                                                               │
│  NULL BITMAP (variable) → Which columns are NULL             │
│                                                               │
│  ─────────── USER DATA STARTS HERE ───────────               │
│  col1_value | col2_value | col3_value | ...                  │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

> 💡 **Why does this matter?** 
> - A table with many small rows has significant overhead (23 bytes per row!)
> - A table with 100 million rows = ~2.3 GB just for tuple headers
> - `t_xmin` and `t_xmax` are the backbone of **MVCC** (more on this below)

---

### TOAST — The Oversized Attribute Storage Technique

When a row is too large to fit in an 8KB page, PostgreSQL uses **TOAST** (The Oversized-Attribute Storage Technique):

```
Normal row (fits in page):
┌────────────────────────────────────┐
│  id=1 | name="Ritesh" | bio="..." │  ← All in one page ✅
└────────────────────────────────────┘

TOASTed row (bio is 50KB of text):
┌────────────────────────────────────────────┐
│  id=1 | name="Ritesh" | bio=TOAST_POINTER │  ← Main table
└─────────────────────────────┬──────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  TOAST TABLE       │
                    │  (pg_toast.pg_123) │
                    │                    │
                    │  chunk 1: [data]   │
                    │  chunk 2: [data]   │
                    │  chunk 3: [data]   │
                    └────────────────────┘
```

**TOAST Strategies:**

| Strategy | Behavior | When Used |
|----------|----------|-----------|
| `PLAIN` | No TOAST — must fit in page | Fixed-length types (int, float) |
| `EXTENDED` | Compress first, then TOAST externally | Default for `text`, `jsonb`, `bytea` |
| `EXTERNAL` | TOAST externally WITHOUT compression | When you need fast substring access |
| `MAIN` | Try compression first, TOAST only if needed | Rarely used manually |

```sql
-- Check TOAST strategy for a column
SELECT attname, attstorage 
FROM pg_attribute 
WHERE attrelid = 'my_table'::regclass AND attnum > 0;

-- Change TOAST strategy
ALTER TABLE my_table ALTER COLUMN big_json SET STORAGE EXTERNAL;
```

---

## 📝 4. Write-Ahead Logging (WAL) — The Crash Recovery Hero

**WAL is THE most critical concept in PostgreSQL durability.** Without WAL, your data would be lost on every crash.

### The Core Principle

> **"Write the LOG before writing the DATA."**

```
Without WAL (dangerous!):
  1. Modify data in shared buffers
  2. Write to data file
  3. 💥 CRASH before complete write → DATA CORRUPTED

With WAL (safe!):
  1. Write change description to WAL file (sequential write — fast!)
  2. Modify data in shared buffers
  3. Eventually, data file gets updated (checkpoint)
  4. 💥 CRASH at any point → Replay WAL to recover → DATA SAFE ✅
```

### How WAL Works — Step by Step

```
┌────────────────────────────────────────────────────────────────┐
│              THE WAL LIFECYCLE                                  │
│                                                                 │
│   Client: UPDATE users SET name = 'John' WHERE id = 42;       │
│                                                                 │
│   Step 1: Backend writes WAL record to WAL Buffer              │
│   ┌──────────────────────────────────┐                         │
│   │  WAL BUFFER (in shared memory)   │                         │
│   │  [LSN: 0/3A000100]              │                         │
│   │  "Page 42, offset 3: change     │                         │
│   │   name from 'Jane' to 'John'"   │                         │
│   └──────────────┬───────────────────┘                         │
│                  │                                              │
│   Step 2: WAL Writer flushes to WAL file (every 200ms or on   │
│           COMMIT)                                               │
│                  │                                              │
│   ┌──────────────▼───────────────────┐                         │
│   │  WAL FILE (on disk)              │                         │
│   │  pg_wal/000000010000000000000003 │                         │
│   │  16MB segments, sequential write │                         │
│   └──────────────────────────────────┘                         │
│                                                                 │
│   Step 3: Data page modified in SHARED BUFFERS (dirty page)    │
│                                                                 │
│   Step 4: CHECKPOINT — dirty pages flushed to data files       │
│   (periodically, every 5 min by default)                       │
│                                                                 │
│   Step 5: Old WAL files can be recycled/archived               │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### WAL Configuration

```sql
-- Key WAL settings in postgresql.conf

wal_level = replica          -- minimal | replica | logical
                             -- 'replica' for streaming replication
                             -- 'logical' for logical replication

fsync = on                   -- NEVER turn this off in production!
                             -- Forces WAL to be physically written

synchronous_commit = on      -- Wait for WAL flush before confirming commit
                             -- 'off' = faster but risk losing last few ms

wal_compression = on         -- Compress WAL (saves I/O, uses CPU)

checkpoint_timeout = 5min    -- Max time between checkpoints
checkpoint_completion_target = 0.9  -- Spread checkpoint I/O over 90% of interval

max_wal_size = 1GB           -- Max WAL size before forced checkpoint
min_wal_size = 80MB          -- Min WAL size to retain
```

> ⚠️ **NEVER set `fsync = off` in production!** It's like removing your car's seatbelts because they're uncomfortable. One crash and you lose everything.

### WAL Levels Explained

```
┌─────────────┬──────────────────────────────────────────────────┐
│  WAL Level  │  What's Logged                                   │
├─────────────┼──────────────────────────────────────────────────┤
│  minimal    │  Only enough for crash recovery                  │
│             │  ❌ No replication, no archiving, no PITR        │
├─────────────┼──────────────────────────────────────────────────┤
│  replica    │  + Enough for streaming replication              │
│             │  ✅ Replication, archiving, PITR                 │
│             │  ⭐ This is the DEFAULT (and recommended)        │
├─────────────┼──────────────────────────────────────────────────┤
│  logical    │  + Enough for logical replication                │
│             │  ✅ All of replica + logical decoding            │
│             │  Needed for tools like Debezium, pg_logical      │
└─────────────┴──────────────────────────────────────────────────┘
```

---

## 🔄 5. MVCC — How PostgreSQL Achieves Concurrency WITHOUT Read Locks

**MVCC (Multi-Version Concurrency Control)** is PostgreSQL's secret weapon. It means:

> **Readers NEVER block writers. Writers NEVER block readers.** 🎯

### How It Works — Visually

```
Transaction 100: INSERT INTO users VALUES (1, 'Alice');

Physical row stored:
┌────────────────────────────────────────────────────┐
│  t_xmin = 100  │  t_xmax = 0  │  data: (1, Alice) │
│  (created by   │  (not deleted │                    │
│   txn 100)     │   yet)        │                    │
└────────────────────────────────────────────────────┘

This row is VISIBLE to any transaction where:
  - t_xmin (100) is committed AND
  - t_xmax (0) means no one has deleted it
```

### UPDATE = DELETE Old + INSERT New

**PostgreSQL does NOT update rows in-place.** An UPDATE creates a NEW version:

```
Transaction 200: UPDATE users SET name = 'Bob' WHERE id = 1;

BEFORE (old version — now "dead"):
┌────────────────────────────────────────────────────────────────┐
│  t_xmin = 100  │  t_xmax = 200  │  data: (1, Alice)           │
│  (created by   │  (deleted by   │  ← This is now a "dead tuple"│
│   txn 100)     │   txn 200)     │                              │
└────────────────────────────────────────────────────────────────┘

AFTER (new version — the "live" one):
┌────────────────────────────────────────────────────────────────┐
│  t_xmin = 200  │  t_xmax = 0    │  data: (1, Bob)             │
│  (created by   │  (not deleted  │  ← This is the current      │
│   txn 200)     │   yet)         │     version                  │
└────────────────────────────────────────────────────────────────┘
```

### Visibility Check — Who Sees What?

```
Timeline:
  Txn 100: INSERT (1, 'Alice')     → COMMITS at time T1
  Txn 200: UPDATE to (1, 'Bob')    → COMMITS at time T3
  Txn 150: SELECT * FROM users;    → Started at time T2 (between T1 and T3)

  ┌─────────┬─────────┬─────────┬──────────┐
  │         T1        T2        T3          │
  │    Txn 100     Txn 150    Txn 200       │
  │    commits     starts     commits       │
  │                                          │
  │  What does Txn 150 see?                 │
  │                                          │
  │  Row v1 (Alice): t_xmin=100 ✅ committed │
  │                  t_xmax=200 ❓ not yet   │
  │                  committed               │
  │  → Txn 150 sees 'Alice' ✅              │
  │                                          │
  │  Row v2 (Bob):   t_xmin=200 ❌ not yet  │
  │                  committed when 150      │
  │                  took its snapshot        │
  │  → Txn 150 does NOT see 'Bob' ❌        │
  └──────────────────────────────────────────┘

Result: Txn 150 sees (1, 'Alice') — because that's what existed
when its SNAPSHOT was taken!
```

> 💡 **This is called Snapshot Isolation.** Each transaction works with a "snapshot" of the database as it was when the transaction started. This is how PostgreSQL achieves **READ COMMITTED** and **REPEATABLE READ** isolation levels.

### The MVCC Trade-off: Dead Tuples (Table Bloat)

The problem with "UPDATE = INSERT new + mark old as dead":

```
1000 UPDATEs on the same row = 1000 dead tuples piling up!

┌──────────┐
│ Live: v1000 (current)
│ Dead: v999
│ Dead: v998
│ Dead: v997
│ ...
│ Dead: v1
│ 
│ 999 dead tuples taking up space! 💀
│ This is called "TABLE BLOAT"
└──────────┘
```

**Solution?** → **VACUUM** 🧹

---

## 🧹 6. VACUUM — PostgreSQL's Garbage Collector

VACUUM is what cleans up **dead tuples** left by MVCC. Without it, your tables grow infinitely.

### Two Types of VACUUM

```
┌──────────────────────────────────────────────────────────────┐
│                    VACUUM Types                               │
├──────────────────────┬───────────────────────────────────────┤
│                      │                                        │
│  REGULAR VACUUM      │  VACUUM FULL                          │
│  (Non-blocking)      │  (Blocking — EXCLUSIVE LOCK!)         │
│                      │                                        │
│  • Marks dead tuples │  • Rewrites entire table              │
│    as reusable       │  • Reclaims disk space to OS          │
│  • Does NOT return   │  • Locks table for duration           │
│    space to OS       │  • Creates new physical file          │
│  • Does NOT lock     │  • Very expensive for large tables    │
│    the table         │                                        │
│  • Fast and safe     │  ⚠️ Use ONLY as last resort!          │
│  • Run frequently    │  Prefer pg_repack instead             │
│                      │                                        │
└──────────────────────┴───────────────────────────────────────┘
```

### VACUUM in Action

```sql
-- Manual VACUUM (rarely needed — autovacuum handles this)
VACUUM users;                -- Regular vacuum
VACUUM FULL users;           -- Full vacuum (blocks table!)
VACUUM VERBOSE users;        -- Vacuum with detailed output
VACUUM ANALYZE users;        -- Vacuum + update statistics (best combo!)

-- Check dead tuples
SELECT 
    relname AS table_name,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    n_dead_tup::float / NULLIF(n_live_tup, 0) AS dead_ratio,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### Autovacuum — The Automatic Janitor

PostgreSQL runs autovacuum in the background. It triggers when:

```
Trigger condition:
  dead_tuples > autovacuum_vacuum_threshold 
                + autovacuum_vacuum_scale_factor × n_live_tup

Default values:
  threshold = 50 dead tuples
  scale_factor = 0.2 (20%)

Example: Table with 10,000 rows
  Trigger when dead_tuples > 50 + (0.2 × 10,000) = 2,050 dead tuples
```

### Autovacuum Configuration

```sql
-- Global settings (postgresql.conf)
autovacuum = on                          -- NEVER turn this off!
autovacuum_max_workers = 3               -- Max parallel vacuum workers
autovacuum_vacuum_threshold = 50         -- Min dead tuples before vacuum
autovacuum_vacuum_scale_factor = 0.2     -- + 20% of table size
autovacuum_analyze_threshold = 50        -- Min changes before analyze
autovacuum_analyze_scale_factor = 0.1    -- + 10% of table size
autovacuum_vacuum_cost_delay = 2ms       -- Throttle vacuum I/O

-- Per-table overrides (for hot tables)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- Vacuum at 1% dead tuples
    autovacuum_vacuum_threshold = 100,
    autovacuum_analyze_scale_factor = 0.005
);
```

> ⚠️ **THE #1 PostgreSQL PRODUCTION DISASTER:**
> Autovacuum falling behind → Transaction ID wraparound → **Database SHUTS DOWN to prevent data corruption!**
> 
> PostgreSQL transaction IDs are 32-bit (~4.2 billion). If VACUUM can't clean up old transactions fast enough, you hit the wraparound limit. PostgreSQL will REFUSE all writes.
> 
> **Monitor this:**
> ```sql
> SELECT datname, age(datfrozenxid) AS xid_age,
>        current_setting('autovacuum_freeze_max_age') AS freeze_max
> FROM pg_database
> ORDER BY age(datfrozenxid) DESC;
> -- If xid_age approaches 2 billion → DANGER ZONE 🚨
> ```

---

## 🔍 7. The Life of a Query — From SQL Text to Result

When you run `SELECT * FROM users WHERE id = 42;`, here's what happens:

```
┌────────────────────────────────────────────────────────────────┐
│                    QUERY LIFECYCLE                              │
│                                                                 │
│  1️⃣  PARSER                                                    │
│     │  "SELECT * FROM users WHERE id = 42"                     │
│     │  → Lexical analysis (tokenize)                           │
│     │  → Syntax check (is it valid SQL?)                       │
│     │  → Produces: Parse Tree                                  │
│     ▼                                                          │
│  2️⃣  ANALYZER / REWRITER                                      │
│     │  → Resolves table/column names (do they exist?)          │
│     │  → Type checking (is id an integer?)                     │
│     │  → View expansion (if querying a view)                   │
│     │  → Rule application (if any rules exist)                 │
│     │  → Produces: Query Tree                                  │
│     ▼                                                          │
│  3️⃣  PLANNER / OPTIMIZER                                      │
│     │  → Generates multiple execution plans                    │
│     │  → Estimates cost for each plan                          │
│     │  → Considers: indexes, join methods, scan types          │
│     │  → Picks the CHEAPEST plan                               │
│     │  → Produces: Execution Plan                              │
│     ▼                                                          │
│  4️⃣  EXECUTOR                                                  │
│     │  → Executes the plan step by step                        │
│     │  → Reads pages from shared buffers (or disk)             │
│     │  → Applies filters, joins, sorts                         │
│     │  → Returns result rows to client                         │
│     ▼                                                          │
│  5️⃣  RESULT → Client                                          │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Scan Types the Planner Chooses From

| Scan Type | When Used | Speed |
|-----------|-----------|-------|
| **Seq Scan** | No useful index, or small table, or fetching large % of rows | Slow for large tables |
| **Index Scan** | B-tree index exists, few rows match | Fast — reads index + heap |
| **Index Only Scan** | All needed columns are IN the index (covering index) | Fastest — skips heap entirely |
| **Bitmap Index Scan** | Multiple index conditions OR medium selectivity | Builds bitmap, then scans heap |
| **TID Scan** | Direct physical row lookup by (page, offset) | Ultra-fast but rare |

```sql
-- See the execution plan
EXPLAIN SELECT * FROM users WHERE id = 42;

-- See the ACTUAL execution with timings
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM users WHERE id = 42;
```

---

## 🗂️ 8. Key System Catalogs — PostgreSQL's Brain

PostgreSQL stores all metadata in **system catalogs** — which are just regular tables!

```sql
-- Core system catalogs you should know

-- All databases in the cluster
SELECT datname, datistemplate, encoding FROM pg_database;

-- All tables in current database
SELECT schemaname, tablename, tableowner 
FROM pg_tables 
WHERE schemaname = 'public';

-- All columns of a table
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'users';

-- All indexes
SELECT indexname, indexdef 
FROM pg_indexes 
WHERE tablename = 'users';

-- Table sizes (including indexes and TOAST)
SELECT 
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
    pg_size_pretty(pg_relation_size(oid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(oid) - pg_relation_size(oid)) AS index_size
FROM pg_class
WHERE relkind = 'r' AND relnamespace = 'public'::regnamespace
ORDER BY pg_total_relation_size(oid) DESC;

-- Active connections and their queries
SELECT pid, usename, state, query, query_start
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;
```

---

## 🆚 PostgreSQL vs MySQL — The Architecture Showdown

| Feature | PostgreSQL | MySQL (InnoDB) |
|---------|-----------|----------------|
| **Process Model** | Multi-process (fork per connection) | Multi-threaded |
| **MVCC** | Stores old versions in main table (needs VACUUM) | Stores old versions in undo tablespace (auto-cleaned) |
| **Default Isolation** | READ COMMITTED | REPEATABLE READ |
| **Full ACID** | Always ✅ | Only with InnoDB engine ✅ |
| **Data Types** | Extremely rich (arrays, JSONB, ranges, custom) | Standard types |
| **Extensibility** | Extensions, custom types, operators, index methods | Plugins, but less flexible |
| **Concurrency** | MVCC — no read locks | MVCC — no read locks |
| **Replication** | Streaming + Logical replication | Binary log replication |
| **JSON** | JSONB with indexing (GIN) 🏆 | JSON type (less powerful) |
| **Full-Text Search** | Built-in (tsvector, tsquery) | Built-in (less flexible) |

---

## 📝 Quick Reference — Essential Commands

```sql
-- PostgreSQL version
SELECT version();

-- Current database and user
SELECT current_database(), current_user, inet_server_addr();

-- Server uptime
SELECT pg_postmaster_start_time(), 
       now() - pg_postmaster_start_time() AS uptime;

-- Database sizes
SELECT datname, pg_size_pretty(pg_database_size(datname)) 
FROM pg_database ORDER BY pg_database_size(datname) DESC;

-- Current activity
SELECT count(*) AS total_connections,
       count(*) FILTER (WHERE state = 'active') AS active,
       count(*) FILTER (WHERE state = 'idle') AS idle,
       count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_txn
FROM pg_stat_activity;

-- Lock monitoring
SELECT l.pid, l.mode, l.granted, a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE NOT l.granted;
```

---

## 🧪 Chapter Summary — Key Takeaways

```
┌────────────────────────────────────────────────────────────────────┐
│  🧠 REMEMBER THESE:                                               │
│                                                                     │
│  1. PostgreSQL = Multi-Process (not threads)                       │
│     → Use connection pooling (PgBouncer) in production             │
│                                                                     │
│  2. Shared Buffers = THE #1 setting to tune (25% of RAM)           │
│                                                                     │
│  3. work_mem is PER OPERATION, not per query — be careful!         │
│                                                                     │
│  4. Everything is stored in 8KB pages                              │
│     → Large values get TOASTed                                     │
│                                                                     │
│  5. WAL = Write-Ahead Log → crash recovery guarantee               │
│     → NEVER disable fsync in production                            │
│                                                                     │
│  6. MVCC = UPDATE creates new row version, old one is "dead"       │
│     → Readers never block writers                                   │
│     → Dead tuples need VACUUM to clean up                          │
│                                                                     │
│  7. VACUUM is NOT optional — autovacuum MUST be running            │
│     → Monitor transaction ID wraparound                            │
│                                                                     │
│  8. Query lifecycle: Parse → Analyze → Plan → Execute              │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [2E.2 — PostgreSQL Installation & Configuration](./02-PG-Installation.md) 🟢
> *Set up PostgreSQL, configure it like a pro, and write your first query in under 10 minutes.*

---

[← Back to Index](../INDEX.md)
