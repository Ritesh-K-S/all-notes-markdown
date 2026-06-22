# 🐘 Chapter 3G.4 — HBase — Hadoop's Database

> **Level:** 🔴 Advanced
> **Time to Master:** ~4–5 hours
> **Prerequisites:** Chapter 1.3 (DBMS Architecture), Chapter 3A.1 (NoSQL Overview), Chapter 3D.1 (Cassandra — for comparison)

> **"When your data is measured in petabytes and your reads must be instant — HBase enters the ring."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why HBase exists** and the problem Hadoop alone couldn't solve
- Know the **complete architecture** — RegionServers, HMaster, ZooKeeper, HDFS
- Grasp the **data model** — Row Keys, Column Families, Columns, Versions
- Understand **how reads and writes flow** at the storage level
- Know when HBase is the **perfect choice** and when it's the **wrong tool**
- Compare HBase with Cassandra and know exactly when to pick which
- Walk away with **real-world use cases** from Facebook, Adobe, Pinterest

---

## 1. Why HBase? — The Story Behind the Beast

### The Year is 2006...

Google published a groundbreaking paper: **"Bigtable: A Distributed Storage System for Structured Data."**

The paper described how Google stored data for Google Search, Google Maps, Google Earth, Gmail, and YouTube — **ALL in one system**.

```
The Problem Google Faced:
━━━━━━━━━━━━━━━━━━━━━━━
  → Billions of web pages to index (crawl the entire internet)
  → Each page has: URL, content, metadata, links, timestamps, versions
  → Need RANDOM reads by URL in < 10ms
  → Need SEQUENTIAL scans for batch analytics (MapReduce)
  → Need to store MULTIPLE VERSIONS of each page
  → Scale? 1000+ commodity servers, petabytes of data
  
  Traditional RDBMS? ❌ Can't scale to 1000 servers
  HDFS alone?        ❌ Great for batch, terrible for random reads
  Key-Value store?   ❌ Too simple — no column structure, no versioning
```

The Hadoop community thought: **"We need an open-source Bigtable."**

And so **HBase** was born in 2007 — **Hadoop Database**, modeled directly after Google's Bigtable paper.

```
Google's Stack        →    Hadoop Ecosystem Equivalent
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GFS (Google File Sys) →    HDFS (Hadoop Distributed File System)
MapReduce             →    Hadoop MapReduce / Spark
Bigtable              →    HBase ← YOU ARE HERE
Chubby (Lock Service) →    ZooKeeper
```

---

## 2. What Exactly IS HBase?

HBase is a **distributed, versioned, column-family-oriented** database that runs on top of HDFS.

Let's break that down:

| Property | What It Means | Why It Matters |
|----------|-------------|----------------|
| **Distributed** | Data spread across many machines automatically | Scale to petabytes on commodity hardware |
| **Versioned** | Every cell stores multiple timestamped values | Time-travel queries: "What was this value 3 days ago?" |
| **Column-Family** | Columns grouped into families, stored together on disk | Only read the column families you need — skip the rest |
| **Sorted** | Data is sorted by row key (lexicographic order) | Range scans are blazing fast |
| **Sparse** | Empty cells don't consume storage | A row can have 10 columns, another row 10,000 — no wasted space |
| **On HDFS** | Uses Hadoop's file system for storage | Automatic replication (3 copies default), fault-tolerant |

### The Key Difference: HBase vs "Regular" NoSQL

```
Redis, MongoDB, DynamoDB:
  → Self-contained databases
  → Manage their own storage, replication, everything

HBase:
  → Sits ON TOP of HDFS
  → HDFS handles storage + replication
  → HBase handles real-time read/write access TO that HDFS data
  → ZooKeeper handles coordination

  Think of it as:
  ┌────────────────────────────────┐
  │         HBase                  │  ← Real-time random read/write layer
  ├────────────────────────────────┤
  │         HDFS                   │  ← Distributed storage layer
  ├────────────────────────────────┤
  │    Commodity Hardware (100s)   │  ← Cheap servers, lots of them
  └────────────────────────────────┘
```

> 💡 **Key Insight**: HBase gives you what HDFS alone cannot — **random, real-time read/write access** to huge datasets. HDFS is optimized for sequential batch processing; HBase adds the ability to look up a single row by key in milliseconds.

---

## 3. HBase Data Model — Think Different

### 3.1 Forget Everything About Tables

In a relational database:
```
Table: Users
┌────┬──────┬─────┬────────────┐
│ id │ name │ age │ email      │  ← Fixed columns, every row has every column
├────┼──────┼─────┼────────────┤
│ 1  │ Raj  │ 28  │ r@mail.com │
│ 2  │ Amit │ 25  │ a@mail.com │
│ 3  │ Sara │ 30  │ NULL       │  ← NULL wastes space
└────┴──────┴─────┴────────────┘
```

In HBase, the **same data looks completely different**:

```
HBase Table: Users

Row Key    Column Family: info             Column Family: contact
           ─────────────────────           ──────────────────────
           info:name    info:age           contact:email      contact:phone

"amit"     "Amit"       "25"               "a@mail.com"       (empty — not stored)
           @t=1001      @t=1001            @t=1001

"raj"      "Raj"        "28"               "r@mail.com"       "+91-9876543210"
           @t=1003      @t=1003            @t=1003            @t=1005
           "Rajesh"     "27"
           @t=1000      @t=1000   ← Previous version! Still stored!

"sara"     "Sara"       "30"               (no email — zero storage used)
           @t=1002      @t=1002
```

### 3.2 The Four Coordinates

In HBase, every piece of data is located by **four coordinates**:

```
┌───────────────────────────────────────────────────────────┐
│              HBase Cell = 4-Dimensional Key               │
│                                                           │
│   (Row Key, Column Family:Qualifier, Timestamp) → Value   │
│                                                           │
│   Example:                                                │
│   ("raj", "info:name", 1003) → "Raj"                     │
│   ("raj", "info:name", 1000) → "Rajesh"  ← old version! │
│   ("raj", "contact:phone", 1005) → "+91-9876543210"      │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

| Coordinate | What It Is | Analogy |
|-----------|-----------|---------|
| **Row Key** | The primary lookup key (bytes, sorted lexicographically) | Like a dictionary word you look up |
| **Column Family** | A group of related columns, defined at table creation | Like sections of a filing cabinet |
| **Column Qualifier** | The specific column within a family (dynamic, no schema) | Like individual folders in the section |
| **Timestamp** | Version identifier (auto or manual, descending order) | Like revision history in Google Docs |

### 3.3 Column Families — The Critical Design Decision

```
                  Column Families in HBase
                  ═══════════════════════

  Table: "webpage"
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  Column Family: "content"     Column Family: "metadata"  │
  │  ┌────────────────────────┐   ┌────────────────────────┐ │
  │  │ Stored together on disk│   │ Stored together on disk│ │
  │  │                        │   │                        │ │
  │  │ content:html           │   │ metadata:author        │ │
  │  │ content:text           │   │ metadata:title         │ │
  │  │ content:images         │   │ metadata:crawl_date    │ │
  │  │                        │   │ metadata:language      │ │
  │  │ (Large, rarely read)   │   │ (Small, frequently     │ │
  │  │                        │   │  read)                 │ │
  │  └────────────────────────┘   └────────────────────────┘ │
  │                                                          │
  │  💡 Querying metadata NEVER touches content data!        │
  │     → Massively reduces I/O                              │
  └──────────────────────────────────────────────────────────┘
```

**Rules of Column Families:**

| Rule | Details |
|------|---------|
| **Defined at table creation** | `CREATE 'webpage', 'content', 'metadata'` — you decide families upfront |
| **Columns are dynamic** | Within a family, add ANY qualifier at runtime — no ALTER TABLE |
| **Stored separately** | Each family = separate set of files on HDFS |
| **Keep them few** | 2–3 families is ideal. HBase docs say: "Try to keep to 1." |
| **Group by access pattern** | Things read together → same family. Things read separately → different family. |

> ⚠️ **Critical Warning**: Having too many column families (10+) severely degrades performance. Each family = separate memstore + HFiles + flush cycle. The compaction storms will bring your cluster to its knees.

### 3.4 Row Key Design — The #1 Make-or-Break Decision

Your row key design **determines everything** in HBase:

```
Row Key Design Impact:
━━━━━━━━━━━━━━━━━━━━━
  ✅ Good row key → Data evenly distributed, fast lookups, efficient scans
  ❌ Bad row key  → Hot regions, skewed load, full table scans, misery

  HBase stores data SORTED by row key (lexicographic order).
  All queries are essentially:
    1. GET by exact row key  → O(log n) — fast!
    2. SCAN by row key range → Sequential read — fast!
    3. Everything else        → Full table scan — 💀 slow!
```

**Common Row Key Patterns:**

```
Pattern 1: REVERSE DOMAIN (for web crawling)
──────────────────────────────────────────
  ❌ Bad:  "www.google.com/search"    → All google.com pages on same region!
  ✅ Good: "com.google.www/search"    → Distributed across regions

Pattern 2: SALTED KEY (for time-series)
──────────────────────────────────────────
  ❌ Bad:  "2024-01-15T10:30:00_sensor1"  → All recent data hits ONE region!
  ✅ Good: "3_2024-01-15T10:30:00_sensor1" → Salt prefix distributes load
           (salt = hash(sensor1) % num_regions)

Pattern 3: COMPOSITE KEY (for multi-dimensional queries)
──────────────────────────────────────────
  Key: "user123_2024-01-15_order456"
       └───┬────┘└─────┬─────┘└───┬───┘
        user ID    date     order ID
  
  Scan: "user123_2024-01" → All orders for user123 in January 2024
  Scan: "user123_"        → All orders for user123 ever

Pattern 4: HASH PREFIX (for uniform distribution)
──────────────────────────────────────────
  Key: md5("user123")[0:4] + "_user123" → "a1b2_user123"
  ⚠️ Loses range scan ability, but perfect distribution
```

> 💡 **Pro Tip**: The most common mistake? Using **monotonically increasing keys** like auto-increment IDs or timestamps. This creates a **hot region** where ALL writes go to one server while others sit idle. Always salt or hash your keys!

---

## 4. HBase Architecture — The Big Picture

### 4.1 Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                      HBase Architecture                              │
│                                                                      │
│   ┌──────────┐                                                       │
│   │  Client  │  (Java API, REST, Thrift, or HBase Shell)             │
│   └────┬─────┘                                                       │
│        │                                                             │
│        │  1. "Where is row key 'user123'?"                           │
│        ▼                                                             │
│   ┌──────────────┐        ┌──────────────────────────────────┐       │
│   │  ZooKeeper   │        │  Meta Table (hbase:meta)          │       │
│   │  Ensemble    │───────►│  Maps: row key range → Region     │       │
│   │  (3 or 5)    │        │        → RegionServer location    │       │
│   └──────────────┘        └──────────────────────────────────┘       │
│        │                                                             │
│        │  2. "Row 'user123' is on RegionServer-2"                    │
│        ▼                                                             │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                       │
│   │ HMaster  │    │ HMaster  │    │          │                       │
│   │ (Active) │    │(Standby) │    │          │                       │
│   └────┬─────┘    └──────────┘    │          │                       │
│        │ Manages regions,          │          │                       │
│        │ load balancing,           │          │                       │
│        │ failover                  │          │                       │
│        ▼                           │          │                       │
│   ┌──────────────┐ ┌──────────────┐│┌──────────────┐                 │
│   │RegionServer-1│ │RegionServer-2│││RegionServer-3│                 │
│   │              │ │              │││              │                 │
│   │ ┌──────────┐ │ │ ┌──────────┐ │││ ┌──────────┐ │                 │
│   │ │ Region A │ │ │ │ Region C │ │││ │ Region E │ │                 │
│   │ │ (a-f)    │ │ │ │ (g-m)    │ │││ │ (n-s)    │ │                 │
│   │ └──────────┘ │ │ └──────────┘ │││ └──────────┘ │                 │
│   │ ┌──────────┐ │ │ ┌──────────┐ │││ ┌──────────┐ │                 │
│   │ │ Region B │ │ │ │ Region D │ │││ │ Region F │ │                 │
│   │ └──────────┘ │ │ └──────────┘ │││ │ (t-z)    │ │                 │
│   │              │ │              │││ └──────────┘ │                 │
│   └──────┬───────┘ └──────┬───────┘│└──────┬───────┘                 │
│          │                │        │       │                         │
│          ▼                ▼        │       ▼                         │
│   ┌──────────────────────────────────────────────────┐               │
│   │                    HDFS                           │               │
│   │   (Stores HFiles, WAL files — replicated 3x)     │               │
│   └──────────────────────────────────────────────────┘               │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 Component Deep-Dive

| Component | Role | What Happens If It Dies? |
|-----------|------|-------------------------|
| **HMaster** | DDL operations (create/delete tables), Region assignment, Load balancing | Standby takes over. Reads/writes continue — HMaster isn't in the data path! |
| **RegionServer** | Hosts regions, handles reads/writes, manages MemStore + HFiles | Regions reassigned to other servers. WAL replayed for recovery. |
| **ZooKeeper** | Tracks which RegionServer is alive, stores root of META table location | Without ZK, cluster can't coordinate. Run 3 or 5 ZK nodes. |
| **Region** | A contiguous range of row keys for a table. Unit of distribution. | Splits when too large (~10GB default). |
| **HDFS** | Persistent storage layer. Stores data files and WAL. | HDFS has 3x replication. Lose a DataNode? No data loss. |

> 💡 **Key Insight**: The HMaster is **NOT** in the read/write path. Clients talk directly to RegionServers. If HMaster goes down, existing reads and writes continue fine. You only need HMaster for admin operations (creating tables, balancing regions).

### 4.3 Inside a RegionServer

```
┌────────────────────────────────────────────────────────────┐
│                    RegionServer                             │
│                                                             │
│   ┌──────────────────────────────────────────────────────┐  │
│   │  WAL (Write-Ahead Log)                                │  │
│   │  ─ One per RegionServer (shared by all regions)       │  │
│   │  ─ Sequential writes to HDFS                          │  │
│   │  ─ For crash recovery                                 │  │
│   └──────────────────────────────────────────────────────┘  │
│                                                             │
│   ┌─────────────────────┐   ┌─────────────────────┐        │
│   │      Region A       │   │      Region B       │        │
│   │   (rows: aaa—fff)   │   │   (rows: ggg—mmm)   │        │
│   │                     │   │                     │        │
│   │  ┌───────────────┐  │   │  ┌───────────────┐  │        │
│   │  │   MemStore    │  │   │  │   MemStore    │  │        │
│   │  │  (In-Memory)  │  │   │  │  (In-Memory)  │  │        │
│   │  │   Write buf   │  │   │  │   Write buf   │  │        │
│   │  │  128MB default│  │   │  │  128MB default│  │        │
│   │  └───────┬───────┘  │   │  └───────┬───────┘  │        │
│   │          │ flush     │   │          │ flush     │        │
│   │          ▼           │   │          ▼           │        │
│   │  ┌───────────────┐  │   │  ┌───────────────┐  │        │
│   │  │   HFiles      │  │   │  │   HFiles      │  │        │
│   │  │  (on HDFS)    │  │   │  │  (on HDFS)    │  │        │
│   │  │  Sorted, imm- │  │   │  │  Sorted, imm- │  │        │
│   │  │  utable files │  │   │  │  utable files │  │        │
│   │  └───────────────┘  │   │  └───────────────┘  │        │
│   │                     │   │                     │        │
│   │  ┌───────────────┐  │   │  ┌───────────────┐  │        │
│   │  │  BlockCache   │  │   │  │  BlockCache   │  │        │
│   │  │  (Read Cache) │  │   │  │  (Read Cache) │  │        │
│   │  └───────────────┘  │   │  └───────────────┘  │        │
│   └─────────────────────┘   └─────────────────────┘        │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

---

## 5. Read & Write Paths — How Data Flows

### 5.1 Write Path

```
Client: PUT row="user123", cf:name="Raj"

Step 1: Client → ZooKeeper → "Which RegionServer has 'user123'?"
                              → Answer cached for future calls
        │
Step 2: Client → RegionServer
        │
Step 3: ▼ Write to WAL (on HDFS) ← Safety first!
        │  Sequential append — very fast
        │
Step 4: ▼ Write to MemStore (in RAM) ← This is where reads find fresh data
        │  Sorted by row key in-memory
        │
Step 5: ✅ Acknowledge to Client — "Write successful"
        │
        │  (Background — happens asynchronously)
        │
Step 6: MemStore reaches threshold (128MB default)
        │
Step 7: ▼ FLUSH → Creates a new HFile on HDFS
        │  MemStore data sorted → written as immutable HFile
        │  WAL entries for this region can be discarded
        │
Step 8: Over time, many small HFiles accumulate
        │
Step 9: ▼ COMPACTION merges HFiles
        │  Minor Compaction: Merge a few small HFiles → fewer larger ones
        │  Major Compaction: Merge ALL HFiles → single file, delete expired versions

Timeline:
━━━━━━━━
  Client Write → WAL (HDFS) → MemStore (RAM) → ACK (< 1ms)
                                     │
                               (async, background)
                                     │
                               Flush → HFile (HDFS)
                                     │
                               Compaction → Merged HFile
```

### 5.2 Read Path

```
Client: GET row="user123", cf:name

Step 1: Client → RegionServer (already knows which one from cache)
        │
Step 2: ▼ Check BlockCache (LRU read cache in RAM)
        │
        ├── HIT  → Return immediately ⚡
        │
        └── MISS → Continue...
        │
Step 3: ▼ Check MemStore (recent writes still in RAM)
        │  These are the freshest writes, not yet flushed
        │
Step 4: ▼ Check HFiles (on HDFS — this is disk I/O)
        │  Uses Bloom Filters to skip irrelevant HFiles!
        │  Uses Block Index to jump to the right block
        │
Step 5: ▼ MERGE results from all sources
        │  (MemStore + HFiles — pick the latest timestamp)
        │
Step 6: ✅ Return to client

Read Performance Trick — Bloom Filters:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Without Bloom Filter: Check EVERY HFile → O(files × log n)
  With Bloom Filter:    "Is 'user123' in this HFile?"
                        → "Definitely NO" → Skip it! (99.9% accurate)
                        → "Maybe YES" → Read it
  Result: Skip 90% of HFiles, read only the relevant ones!
```

> 💡 **Pro Tip**: Enable **Bloom Filters** on every column family — they're the single biggest read optimization in HBase. The memory overhead is tiny compared to the I/O savings.

---

## 6. Compaction — Keeping HBase Healthy

### Why Compaction Matters

```
Without Compaction:
  Write 1 → HFile1 (10MB)
  Write 2 → HFile2 (10MB)
  Write 3 → HFile3 (10MB)
  ...
  Write 100 → HFile100 (10MB)
  
  Read "user123" → Must check ALL 100 HFiles! 🐌💀

With Compaction:
  Minor: HFile1 + HFile2 + HFile3 → MergedHFile (30MB)
  Major: ALL HFiles → SingleHFile (1GB)
  
  Read "user123" → Check 1 file ⚡
```

| Type | What It Does | When | Impact |
|------|-------------|------|--------|
| **Minor Compaction** | Merges a few small HFiles into a larger one | Automatically, when file count exceeds threshold | Low I/O, quick |
| **Major Compaction** | Merges ALL HFiles for a region into ONE | Default: weekly (configurable) | Heavy I/O! Schedule during off-peak |

> ⚠️ **Production Warning**: Major compaction can consume massive I/O. Disable auto-major-compaction and schedule it during low-traffic windows:
> ```
> hbase.hregion.majorcompaction = 0   (disable auto)
> # Then run manually: major_compact 'table_name'
> ```

---

## 7. Region Splitting — Auto-Scaling Within a Table

```
Initially:
  Table "users" has 1 Region (all row keys: "" → "")
  ┌─────────────────────────────────────┐
  │          Region (all data)           │  on RegionServer-1
  └─────────────────────────────────────┘

Data grows... Region hits 10GB (default threshold)...

Auto-Split:
  ┌──────────────────┐  ┌──────────────────┐
  │    Region A      │  │    Region B      │
  │  (rows: "" → m)  │  │  (rows: m → "")  │
  │   RegionServer-1 │  │   RegionServer-2 │  ← HMaster may move it
  └──────────────────┘  └──────────────────┘

More growth... Each region splits again...

  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
  │Region A │ │Region B │ │Region C │ │Region D │
  │ "" → d  │ │ d → m   │ │ m → s   │ │ s → ""  │
  │ RS-1    │ │ RS-2    │ │ RS-3    │ │ RS-1    │
  └─────────┘ └─────────┘ └─────────┘ └─────────┘

  Automatic! No manual intervention needed.
  HMaster balances regions across RegionServers.
```

> 💡 **Pro Tip**: For large bulk loads, **pre-split** your table to avoid the initial single-region bottleneck:
> ```
> create 'users', 'info', {SPLITS => ['d', 'm', 's']}
> ```
> This creates 4 regions immediately, distributing writes from the start.

---

## 8. HBase Shell — Quick Reference

### Table Operations

```bash
# Create a table with column families
create 'users', 'info', 'contact'

# Create with specific configuration
create 'users', 
  {NAME => 'info', VERSIONS => 3, BLOOMFILTER => 'ROW', COMPRESSION => 'SNAPPY'},
  {NAME => 'contact', VERSIONS => 1, TTL => 86400}

# List all tables
list

# Describe table structure
describe 'users'

# Disable a table (required before drop or major alter)
disable 'users'

# Drop a table
drop 'users'

# Alter table — add a column family
alter 'users', {NAME => 'preferences', VERSIONS => 1}

# Alter table — remove a column family
alter 'users', {NAME => 'preferences', METHOD => 'delete'}
```

### CRUD Operations

```bash
# ── INSERT (PUT) ──
put 'users', 'user001', 'info:name', 'Ritesh'
put 'users', 'user001', 'info:age', '28'
put 'users', 'user001', 'contact:email', 'ritesh@example.com'
put 'users', 'user001', 'contact:phone', '+91-9876543210'

# ── READ (GET) ──
# Get entire row
get 'users', 'user001'

# Get specific column
get 'users', 'user001', 'info:name'

# Get specific column family
get 'users', 'user001', 'info'

# Get with versions
get 'users', 'user001', {COLUMN => 'info:name', VERSIONS => 3}

# ── SCAN (Range Query) ──
# Scan entire table (careful in production!)
scan 'users'

# Scan with row key range
scan 'users', {STARTROW => 'user001', STOPROW => 'user010'}

# Scan with limit
scan 'users', {LIMIT => 10}

# Scan with filter
scan 'users', {FILTER => "SingleColumnValueFilter('info', 'age', >=, 'binary:25')"}

# ── DELETE ──
# Delete a specific cell
delete 'users', 'user001', 'contact:phone'

# Delete entire row
deleteall 'users', 'user001'

# ── UPDATE (just PUT again — new timestamp wins) ──
put 'users', 'user001', 'info:name', 'Ritesh Singh'  # Overwrites (new version)
```

### Admin Commands

```bash
# Check cluster status
status

# Check table regions
list_regions 'users'

# Trigger major compaction
major_compact 'users'

# Flush memstore to HFiles
flush 'users'

# Count rows (launches MapReduce — slow for large tables!)
count 'users'

# Truncate a table (disable + drop + recreate)
truncate 'users'
```

---

## 9. HBase Java API — Real Code

```java
import org.apache.hadoop.hbase.*;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

// ── Connection Setup ──
Configuration config = HBaseConfiguration.create();
config.set("hbase.zookeeper.quorum", "zk1,zk2,zk3");
Connection connection = ConnectionFactory.createConnection(config);
Table table = connection.getTable(TableName.valueOf("users"));

// ── PUT (Insert/Update) ──
Put put = new Put(Bytes.toBytes("user001"));                    // Row key
put.addColumn(
    Bytes.toBytes("info"),                                      // Column family
    Bytes.toBytes("name"),                                      // Column qualifier
    Bytes.toBytes("Ritesh")                                     // Value
);
put.addColumn(Bytes.toBytes("info"), Bytes.toBytes("age"), Bytes.toBytes("28"));
put.addColumn(Bytes.toBytes("contact"), Bytes.toBytes("email"), Bytes.toBytes("r@x.com"));
table.put(put);

// ── GET (Read) ──
Get get = new Get(Bytes.toBytes("user001"));
get.addColumn(Bytes.toBytes("info"), Bytes.toBytes("name"));
Result result = table.get(get);
String name = Bytes.toString(
    result.getValue(Bytes.toBytes("info"), Bytes.toBytes("name"))
);
System.out.println("Name: " + name);  // → "Ritesh"

// ── SCAN (Range Query) ──
Scan scan = new Scan();
scan.withStartRow(Bytes.toBytes("user001"));
scan.withStopRow(Bytes.toBytes("user100"));
scan.addFamily(Bytes.toBytes("info"));  // Only read 'info' family

ResultScanner scanner = table.getScanner(scan);
for (Result row : scanner) {
    String rowKey = Bytes.toString(row.getRow());
    String userName = Bytes.toString(
        row.getValue(Bytes.toBytes("info"), Bytes.toBytes("name"))
    );
    System.out.println(rowKey + " → " + userName);
}
scanner.close();

// ── DELETE ──
Delete delete = new Delete(Bytes.toBytes("user001"));
delete.addColumn(Bytes.toBytes("contact"), Bytes.toBytes("phone"));
table.delete(delete);

// ── Cleanup ──
table.close();
connection.close();
```

---

## 10. HBase vs Cassandra — The Epic Showdown

Both are wide-column stores. Both handle massive scale. So when do you pick which?

```
                  HBase vs Cassandra
  ┌─────────────────────┬──────────────────────────┐
  │       HBase          │       Cassandra           │
  ├─────────────────────┼──────────────────────────┤
  │  Master-Slave        │  Peer-to-Peer (no master)│
  │  (HMaster + RS)      │  (all nodes equal)        │
  │                      │                           │
  │  Strong Consistency   │  Tunable Consistency      │
  │  (always latest)      │  (choose per query)       │
  │                      │                           │
  │  Runs on HDFS         │  Self-contained storage   │
  │  (needs Hadoop stack) │  (just Cassandra)          │
  │                      │                           │
  │  Reads optimized      │  Writes optimized          │
  │  (Bloom filters,      │  (Log-structured,          │
  │   block cache)        │   no read-before-write)    │
  │                      │                           │
  │  Single data center   │  Multi-data center         │
  │  (DC replication      │  (built-in, first-class)   │
  │   needs work)         │                           │
  │                      │                           │
  │  Hadoop integration   │  Standalone / Spark        │
  │  (MapReduce, Spark    │  integration               │
  │   native)             │                           │
  │                      │                           │
  │  Java/JVM             │  Java/JVM                  │
  └─────────────────────┴──────────────────────────┘
```

| Choose HBase When... | Choose Cassandra When... |
|---------------------|-------------------------|
| You already have a **Hadoop ecosystem** | You need a **standalone** database without Hadoop |
| You need **strong consistency** guaranteed | You need **multi-data-center** replication built-in |
| Heavy **random read** workloads | Heavy **write** workloads (IoT, time-series) |
| You need **MapReduce/Spark** directly on the data | You need **always-available** writes (AP system) |
| You can tolerate **single point of failure** with failover | You need **zero single point of failure** |
| Your team knows the **Hadoop stack** | You want **simpler operations** (no ZK, no HDFS) |

---

## 11. Real-World Use Cases — Who Uses HBase?

| Company | Use Case | Scale |
|---------|----------|-------|
| **Facebook** | Messages platform (was using HBase for Messenger storage) | 100+ PB |
| **Twitter** | Real-time analytics on tweets | Billions of rows |
| **Adobe** | Processing structured data pipelines for analytics | Petabytes |
| **Pinterest** | User activity data, content storage | 10s of billions of rows |
| **Spotify** | User recommendations and activity tracking | Massive scale |
| **Yahoo** | Serving web content and user profiles | 100s of TB |
| **Bloomberg** | Financial data analytics | High-throughput reads |

### Typical HBase Use Case: Web Crawler

```
Table: "crawled_pages"

Row Key: "com.stackoverflow.www_/questions/12345"  (reversed domain)

Column Family: "content"
  content:html    → "<html>...</html>"         (latest crawl)
  content:text    → "How do I sort a list..."  (extracted text)

Column Family: "metadata"
  metadata:title       → "How to sort a list in Python"
  metadata:crawl_date  → "2024-01-15"
  metadata:status      → "200"
  metadata:language    → "en"

Column Family: "links"
  links:outbound_1  → "com.python.docs_/sorting"
  links:outbound_2  → "com.google.www_/search?q=..."
  links:inbound_1   → "com.reddit.www_/r/python/..."

With VERSIONS=5 on "content" family:
  → You have the last 5 crawled versions of every page!
  → Time-travel: "What did this page look like 3 weeks ago?"
```

---

## 12. When NOT to Use HBase

```
❌ DON'T use HBase when:

  1. Your data is < 100GB
     → Overkill. Use PostgreSQL, MySQL, or MongoDB.
     
  2. You need complex queries (JOINs, aggregations, ad-hoc SQL)
     → HBase has NO SQL (no joins, no secondary indexes by default)
     → Use a relational database or Spark SQL on top
     
  3. You don't have a Hadoop team
     → HBase needs HDFS + ZooKeeper + HBase — 3 systems to manage
     → Cassandra is simpler to operate standalone
     
  4. You need multi-datacenter replication
     → HBase replication across DCs is an afterthought
     → Cassandra was built for this
     
  5. You need transactions across multiple rows
     → HBase only guarantees atomicity within a SINGLE row
     → Need cross-row? Use a relational database or CockroachDB
     
  6. Your access pattern is mostly random, small reads
     → Redis or DynamoDB will be much simpler and faster
```

---

## 13. HBase Configuration — Key Tuning Parameters

| Parameter | Default | Recommended | What It Does |
|-----------|---------|-------------|-------------|
| `hbase.regionserver.handler.count` | 30 | 100–200 | RPC handler threads per RegionServer |
| `hbase.hregion.memstore.flush.size` | 128MB | 128–256MB | When to flush MemStore to HFile |
| `hbase.hregion.max.filesize` | 10GB | 5–20GB | When to split a region |
| `hbase.regionserver.global.memstore.size` | 0.4 | 0.4 | % of heap for all MemStores |
| `hfile.block.cache.size` | 0.4 | 0.4 | % of heap for read block cache |
| `hbase.hstore.compactionThreshold` | 3 | 3–6 | Min HFiles to trigger minor compaction |
| `hbase.hregion.majorcompaction` | 7 days | 0 (manual) | Auto major compaction interval |
| `hbase.client.scanner.caching` | 100 | 500–1000 | Rows fetched per RPC in scan |
| `zookeeper.session.timeout` | 90000 | 60000–120000 | ZK session timeout (ms) |

> 💡 **Pro Tip**: A good starting formula for RegionServer heap:
> ```
> Heap = (MemStore) + (BlockCache) + (Overhead)
>      = (0.4 × Heap) + (0.4 × Heap) + (0.2 × Heap)
> 
> For a 32GB heap server:
>   MemStore  = 12.8GB → handles write buffers
>   BlockCache = 12.8GB → caches hot read blocks
>   Overhead  = 6.4GB  → GC, RPC, bookkeeping
> ```

---

## 14. Quick Reference — HBase Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    HBase Cheat Sheet                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DATA MODEL:                                                     │
│    (Row Key, Column Family:Qualifier, Timestamp) → Value         │
│                                                                  │
│  ARCHITECTURE:                                                   │
│    Client → ZooKeeper → RegionServer → MemStore/HFiles → HDFS   │
│    HMaster: Admin only (not in read/write path)                  │
│                                                                  │
│  WRITE PATH:                                                     │
│    WAL (HDFS) → MemStore (RAM) → ACK → Flush → HFile → Compact  │
│                                                                  │
│  READ PATH:                                                      │
│    BlockCache → MemStore → HFiles (Bloom Filter optimized)       │
│                                                                  │
│  ROW KEY RULES:                                                  │
│    ✅ Short, meaningful, evenly distributed                      │
│    ❌ No monotonically increasing (timestamps, auto-increment)   │
│    ✅ Use salting or hashing for even distribution                │
│    ✅ Design for your PRIMARY access pattern                     │
│                                                                  │
│  COLUMN FAMILY RULES:                                            │
│    ✅ Keep to 1–3 families                                       │
│    ✅ Group by access pattern                                    │
│    ❌ Don't create 10+ families                                  │
│    ✅ Enable Bloom Filters on every family                       │
│                                                                  │
│  CAP THEOREM:                                                    │
│    HBase = CP (Consistent + Partition-tolerant)                  │
│    Strong consistency guaranteed                                 │
│                                                                  │
│  SWEET SPOT:                                                     │
│    Petabyte-scale + Random reads + Hadoop ecosystem              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Key Takeaways

| # | Takeaway |
|---|----------|
| 1 | HBase is **Google Bigtable's open-source clone** — built for petabyte-scale on commodity hardware |
| 2 | It sits **on top of HDFS** — you need the Hadoop ecosystem to run it |
| 3 | Data model is **(Row Key, Column Family:Qualifier, Timestamp) → Value** — fundamentally different from relational |
| 4 | **Row key design is everything** — it determines data distribution, query speed, and hotspot avoidance |
| 5 | **Column families** are stored separately — keep them few (1–3), group by access pattern |
| 6 | Write path: **WAL → MemStore → HFile** — writes are fast (append-only) |
| 7 | Read path: **BlockCache → MemStore → HFiles** — use Bloom Filters to skip irrelevant files |
| 8 | HBase gives **strong consistency** (CP in CAP) — unlike Cassandra's tunable consistency |
| 9 | **Don't use HBase** for small data, complex queries, or if you don't have a Hadoop team |
| 10 | Real-world: Facebook Messenger, Twitter analytics, web crawling — always **massive scale** |

---

> **Next Chapter:** [3G.5 — Vector Databases — AI/ML Era (Pinecone, Milvus, Weaviate, pgvector)](./05-Vector-Databases.md) 🟡🔥
