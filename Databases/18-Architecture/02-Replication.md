# 🔄 Chapter 7.2 — Replication Strategies Deep Dive

> **Level:** 🔴 Advanced | ⭐ Must-Know
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 1.6 (CAP Theorem), Chapter 1.8 (Transactions & Concurrency)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why** replication is non-negotiable for production systems
- Master **3 replication topologies** (Single-Leader, Multi-Leader, Leaderless)
- Know the difference between **synchronous and asynchronous** replication — and why it matters for your money
- Design **quorum reads and writes** like a distributed systems engineer
- Handle **replication lag** and **conflict resolution** in real systems
- Understand replication in **PostgreSQL, MySQL, MongoDB, Cassandra**, and more
- Make **architecture decisions** that prevent data loss at 3 AM

---

## 🧠 Why Replication Exists — The Nightmare Scenario

```
  Friday, 11:47 PM. Your database server's hard drive fails.

  ┌─────────────────────────┐
  │      Database Server    │
  │                         │
  │   💀 DISK FAILURE 💀    │
  │                         │
  │   10 years of data      │
  │   50 million users      │
  │   $2M revenue/day       │
  │                         │
  │   ALL GONE.             │
  └─────────────────────────┘

  Without replication:
  → Restore from last night's backup (8 hours old)
  → 8 hours of transactions LOST permanently
  → Users: "Where's my money?!" 😱
  → CEO: "You're fired." 🔥

  With replication:
  → Replica takes over in SECONDS
  → ZERO data loss
  → Users don't even notice
  → You sleep through the night 😴
```

> **Replication = Keeping copies of the same data on multiple machines.**

### The Three Reasons to Replicate

```
┌──────────────────────────────────────────────────────────────┐
│          WHY REPLICATE?                                       │
│                                                               │
│  1. HIGH AVAILABILITY (Fault Tolerance)                       │
│     → If one server dies, another takes over                  │
│     → No downtime for your users                              │
│                                                               │
│  2. READ SCALABILITY                                          │
│     → Spread read queries across multiple replicas            │
│     → 1 primary + 5 replicas = 6x read throughput             │
│                                                               │
│  3. GEO-LOCALITY (Latency Reduction)                          │
│     → Put replicas close to users                             │
│     → US user → US replica (5ms)                              │
│     → vs US user → EU primary (150ms)                         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🏛️ Topology 1: Single-Leader (Master-Slave) Replication

> **The most common pattern — used by PostgreSQL, MySQL, SQL Server, MongoDB, and more.**

### How It Works

```
                Single-Leader Replication
                ════════════════════════

                    ┌──────────────────┐
                    │     PRIMARY      │
                    │   (Leader/Master)│
                    │                  │
                    │  Handles ALL     │
                    │  WRITES ✍️       │
                    │                  │
                    │  Also handles    │
                    │  reads (optional)│
                    └────────┬─────────┘
                             │
                    Replication Stream
                    (WAL / Binlog / Oplog)
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ REPLICA 1│  │ REPLICA 2│  │ REPLICA 3│
        │ (Follower│  │ (Follower│  │ (Follower│
        │ / Slave) │  │ / Slave) │  │ / Slave) │
        │          │  │          │  │          │
        │ READS ✅ │  │ READS ✅ │  │ READS ✅ │
        │ WRITES ❌│  │ WRITES ❌│  │ WRITES ❌│
        └──────────┘  └──────────┘  └──────────┘
        
        ✅ Client reads can go to ANY replica
        ✍️ Client writes MUST go to the PRIMARY
```

### Synchronous vs Asynchronous Replication

```
        SYNCHRONOUS REPLICATION
        ═══════════════════════

  Client          Primary           Replica
    │                │                 │
    │── WRITE ──────►│                 │
    │                │── replicate ───►│
    │                │                 │── apply
    │                │◄── ACK ────────│
    │◄── SUCCESS ───│                 │
    │                │                 │

  ✅ ZERO data loss (replica is always up-to-date)
  ❌ SLOWER (must wait for replica to confirm)
  ❌ If replica is down → writes BLOCK or FAIL
  
  Used for: Financial systems, banking, healthcare

  ─────────────────────────────────────────────────

        ASYNCHRONOUS REPLICATION
        ════════════════════════

  Client          Primary           Replica
    │                │                 │
    │── WRITE ──────►│                 │
    │◄── SUCCESS ───│                 │
    │                │── replicate ───►│  (later)
    │                │                 │── apply (later)

  ✅ FAST (client doesn't wait for replica)
  ✅ Primary works even if replica is down
  ❌ DATA LOSS possible (replica might be behind)
  ❌ Replication lag = stale reads
  
  Used for: Most web apps, social media, content platforms

  ─────────────────────────────────────────────────

        SEMI-SYNCHRONOUS REPLICATION
        ════════════════════════════

  At least ONE replica must confirm, others can be async.

  Primary → Replica 1 (sync) → ACK → confirm to client
          → Replica 2 (async) → eventually catches up
          → Replica 3 (async) → eventually catches up

  ✅ Balance between safety and speed
  ✅ At least ONE replica has the latest data
  ❌ Slightly slower than full async
  
  Used by: MySQL (semi-sync), PostgreSQL (synchronous_commit)
```

### Replication Lag — The #1 Problem

```
  WHAT IS REPLICATION LAG?
  
  Time between a write on Primary and that write appearing on Replica.

  Timeline:
  ──────────────────────────────────────────────────────────────►
  
  T=0ms    User writes "name = John" to Primary
  T=1ms    Primary confirms write to client
  T=5ms    Primary sends WAL record to Replica
  T=20ms   Replica applies the change
  
  Replication Lag = 20ms
  
  During those 20ms, if someone reads from the Replica,
  they see the OLD value! 👻

  ┌──────────────────────────────────────────────────────────┐
  │  REAL SCENARIO: "I just updated my profile but I can't  │
  │  see the changes!"                                       │
  │                                                          │
  │  T=0:   User updates display name via PRIMARY            │
  │  T=1:   Server responds "Success!"                       │
  │  T=2:   Page refreshes → reads from REPLICA              │
  │  T=2:   Replica hasn't received the update yet!          │
  │  T=2:   User sees OLD name → "Your app is broken!" 😤   │
  └──────────────────────────────────────────────────────────┘
```

### Solutions for Replication Lag

```
┌──────────────────────────────────────────────────────────────────┐
│  SOLUTION 1: Read-Your-Own-Writes                                │
│                                                                   │
│  After a user WRITES, route their subsequent READS to Primary    │
│  for a short window (e.g., 5 seconds), then back to replicas.   │
│                                                                   │
│  Implementation:                                                  │
│  • Track "last write timestamp" per user session                 │
│  • If (now - last_write) < 5s → read from Primary               │
│  • Otherwise → read from any replica                              │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  SOLUTION 2: Monotonic Reads                                      │
│                                                                   │
│  Ensure a user always reads from the SAME replica.               │
│  They might see stale data, but they'll never see data           │
│  go BACKWARDS in time.                                            │
│                                                                   │
│  Implementation:                                                  │
│  • Hash(user_id) % num_replicas = sticky replica assignment      │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  SOLUTION 3: Causal Consistency                                   │
│                                                                   │
│  If write A happened before write B, everyone sees A before B.   │
│  Use logical clocks or vector clocks to track causality.         │
│                                                                   │
│  Implementation:                                                  │
│  • Attach a monotonic timestamp/sequence to each write           │
│  • Replicas apply writes in causal order                          │
│  • Reads wait until replica has seen required timestamp           │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  SOLUTION 4: Synchronous Replication (Nuclear Option)             │
│                                                                   │
│  Wait for replicas before confirming writes.                      │
│  Eliminates lag but kills write performance.                      │
│  Use only for critical data (financial transactions).             │
└──────────────────────────────────────────────────────────────────┘
```

### Failover — When the Primary Dies

```
                    PRIMARY FAILOVER
                    ════════════════

  Before:
        ┌──────────┐
        │ PRIMARY  │ ← All writes go here
        │   💀     │ ← CRASHED!
        └──────────┘
             │
      ┌──────┼──────┐
      ▼      ▼      ▼
    ┌────┐ ┌────┐ ┌────┐
    │ R1 │ │ R2 │ │ R3 │
    │lag:│ │lag:│ │lag:│
    │ 1s │ │ 5s │ │ 2s │
    └────┘ └────┘ └────┘

  Failover Process:
  
  Step 1: DETECT failure (heartbeat timeout)
          → Typically 10-30 seconds
  
  Step 2: ELECT new primary
          → R1 has least lag → PROMOTE R1 to Primary
  
  Step 3: RECONFIGURE replicas
          → R2 and R3 now replicate from R1
  
  Step 4: UPDATE routing
          → Application / proxy points to new Primary

  After:
        ┌──────────┐
        │ R1 (NEW  │ ← Promoted to Primary!
        │ PRIMARY) │
        └──────────┘
             │
          ┌──┴──┐
          ▼     ▼
        ┌────┐ ┌────┐
        │ R2 │ │ R3 │
        └────┘ └────┘

  ⚠️ DANGERS OF FAILOVER:

  1. DATA LOSS: R1 was 1 second behind!
     → That 1 second of transactions = GONE
     → This is why financial systems use sync replication

  2. SPLIT-BRAIN: What if Primary isn't actually dead?
     → Network partition made it UNREACHABLE
     → Old Primary comes back, thinks it's still Primary
     → TWO Primaries writing = data corruption! 💀
     → Solution: STONITH (Shoot The Other Node In The Head)
       → Forcefully power off the old Primary
```

### Single-Leader Replication in Real Databases

```sql
-- ═══════════════════════════════════════════════════════
-- PostgreSQL Streaming Replication
-- ═══════════════════════════════════════════════════════

-- On PRIMARY (postgresql.conf):
wal_level = replica
max_wal_senders = 5              -- Max replication connections
synchronous_commit = on          -- Wait for replica? (on = sync)
synchronous_standby_names = 'replica1'  -- Which replica is sync?

-- Create replication user:
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_pw';

-- On REPLICA:
-- pg_basebackup -h primary_host -D /var/lib/postgresql/data -U replicator -P

-- standby.signal file (just create empty file):
-- touch /var/lib/postgresql/data/standby.signal

-- In postgresql.conf on replica:
primary_conninfo = 'host=primary_host user=replicator password=secure_pw'

-- Check replication lag:
SELECT 
    client_addr,
    state,
    sent_lsn,
    replay_lsn,
    sent_lsn - replay_lsn AS replication_lag_bytes,
    replay_lag
FROM pg_stat_replication;
```

```sql
-- ═══════════════════════════════════════════════════════
-- MySQL Replication Setup
-- ═══════════════════════════════════════════════════════

-- On PRIMARY (my.cnf):
-- server-id = 1
-- log_bin = mysql-bin
-- binlog_format = ROW

-- Create replication user:
CREATE USER 'replicator'@'%' IDENTIFIED BY 'secure_pw';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
SHOW MASTER STATUS;  -- Note: File and Position

-- On REPLICA (my.cnf):
-- server-id = 2
-- relay_log = relay-bin
-- read_only = ON

-- Configure replication:
CHANGE MASTER TO
    MASTER_HOST='primary_host',
    MASTER_USER='replicator',
    MASTER_PASSWORD='secure_pw',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=154;

START SLAVE;
SHOW SLAVE STATUS\G

-- Key fields to monitor:
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
-- Seconds_Behind_Master: 0   ← This is replication lag!
```

```javascript
// ═══════════════════════════════════════════════════════
// MongoDB Replica Set
// ═══════════════════════════════════════════════════════

// Initialize a 3-node replica set
rs.initiate({
    _id: "myReplicaSet",
    members: [
        { _id: 0, host: "mongo1:27017", priority: 2 },  // Preferred primary
        { _id: 1, host: "mongo2:27017", priority: 1 },
        { _id: 2, host: "mongo3:27017", priority: 1 }
    ]
});

// Check replica set status
rs.status()

// Read from secondary (requires explicit opt-in)
db.getMongo().setReadPref("secondaryPreferred")

// Write concern: Wait for majority of replicas
db.orders.insertOne(
    { item: "laptop", qty: 1 },
    { writeConcern: { w: "majority", wtimeout: 5000 } }
);

// Read concern: Read only majority-committed data
db.orders.find({ item: "laptop" }).readConcern("majority")
```

---

## 🌐 Topology 2: Multi-Leader (Master-Master) Replication

> **Multiple nodes accept writes — used for multi-datacenter setups**

### How It Works

```
              Multi-Leader Replication
              ═══════════════════════

     US Data Center                 EU Data Center
     ═════════════                  ═════════════

    ┌──────────────┐              ┌──────────────┐
    │   LEADER 1   │◄────────────►│   LEADER 2   │
    │   (US-East)  │  bi-directional  │   (EU-West)  │
    │              │  replication │              │
    │  Writes ✅   │              │  Writes ✅   │
    │  Reads  ✅   │              │  Reads  ✅   │
    └──────┬───────┘              └──────┬───────┘
           │                             │
      ┌────┴────┐                   ┌────┴────┐
      ▼         ▼                   ▼         ▼
   ┌──────┐ ┌──────┐            ┌──────┐ ┌──────┐
   │ R1   │ │ R2   │            │ R3   │ │ R4   │
   │(read)│ │(read)│            │(read)│ │(read)│
   └──────┘ └──────┘            └──────┘ └──────┘

   US user writes to Leader 1 → fast (10ms)
   EU user writes to Leader 2 → fast (10ms)
   Both leaders sync with each other → async (100ms)
```

### When to Use Multi-Leader

```
┌──────────────────────────────────────────────────────────────┐
│  ✅ GOOD USE CASES:                                          │
│                                                               │
│  1. Multi-datacenter (reduce write latency per region)       │
│  2. Collaborative editing (Google Docs-style)                │
│  3. Offline-first apps (PouchDB/CouchDB sync)              │
│                                                               │
│  ❌ BAD USE CASES (avoid multi-leader here):                 │
│                                                               │
│  1. Financial transactions (conflicts = lost money)          │
│  2. Inventory management (overselling risk)                  │
│  3. Anything requiring strong consistency                     │
│                                                               │
│  ⚠️  THE BIG PROBLEM: WRITE CONFLICTS                       │
└──────────────────────────────────────────────────────────────┘
```

### The Conflict Problem — The Hardest Part

```
  CONFLICT SCENARIO:

  T=0: User profile has name = "Alice"

  T=1: Leader 1 (US): UPDATE name = "Bob"    (by Admin A)
  T=1: Leader 2 (EU): UPDATE name = "Charlie" (by Admin B)
       ^ SAME TIME, SAME ROW, DIFFERENT VALUES!

  T=2: Leaders sync...
  
  Leader 1 has: "Bob" + incoming "Charlie"
  Leader 2 has: "Charlie" + incoming "Bob"
  
  ❓ Who wins? "Bob" or "Charlie"?
  
  ┌──────────────────────────────────────────────────────────┐
  │  CONFLICT RESOLUTION STRATEGIES:                          │
  │                                                           │
  │  1. LAST WRITE WINS (LWW)                                │
  │     → Highest timestamp wins                              │
  │     → Simple but LOSES DATA                               │
  │     → Used by: Cassandra, DynamoDB                        │
  │     → If timestamps are close, it's arbitrary             │
  │                                                           │
  │  2. CUSTOM MERGE FUNCTION                                 │
  │     → Application-specific logic                          │
  │     → "Concatenate values": "Bob + Charlie" = "Bob,Charlie│
  │     → Works for CRDTs (counters, sets)                    │
  │     → Complex to implement correctly                      │
  │                                                           │
  │  3. KEEP BOTH VERSIONS (Conflict-Free)                    │
  │     → Store both versions, let user resolve               │
  │     → Used by: CouchDB, Git                              │
  │     → Best for collaborative apps                         │
  │                                                           │
  │  4. OPERATIONAL TRANSFORM (OT)                            │
  │     → Used by Google Docs                                 │
  │     → Transform concurrent ops to maintain consistency     │
  │     → Extremely complex                                   │
  │                                                           │
  │  5. CRDTs (Conflict-Free Replicated Data Types)           │
  │     → Data structures that auto-merge                     │
  │     → G-Counter, PN-Counter, OR-Set, LWW-Register        │
  │     → Mathematically proven conflict-free                  │
  │     → Used by: Redis (CRDT mode), Riak                   │
  └──────────────────────────────────────────────────────────┘
```

### CRDTs — The Elegant Solution

```
  G-Counter (Grow-Only Counter) — CRDT Example

  Three nodes counting "page views":

  Node A: {A: 5, B: 0, C: 0}   Total: 5
  Node B: {A: 0, B: 3, C: 0}   Total: 3
  Node C: {A: 0, B: 0, C: 7}   Total: 7

  Merge Rule: Take MAX of each node's counter

  Merged: {A: 5, B: 3, C: 7}   Total: 15 ✅

  No conflicts! No matter the merge order!

  Merge(A,B) = {A:5, B:3, C:0} → then Merge with C = {A:5, B:3, C:7} = 15
  Merge(B,C) = {A:0, B:3, C:7} → then Merge with A = {A:5, B:3, C:7} = 15
  Same result! ✅ This is the magic of CRDTs.
```

### Multi-Leader in Real Databases

```sql
-- ═══════════════════════════════════════════════════════
-- MySQL Group Replication (Multi-Primary Mode)
-- ═══════════════════════════════════════════════════════

-- my.cnf on each node:
-- group_replication_single_primary_mode = OFF  ← Multi-primary!
-- group_replication_group_name = "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"

-- Conflict detection: 
-- MySQL uses CERTIFICATION (optimistic concurrency)
-- If two nodes modify the same row → FIRST commit wins, 
-- second gets ROLLBACK'd

-- Check group status:
SELECT * FROM performance_schema.replication_group_members;
```

```sql
-- ═══════════════════════════════════════════════════════
-- PostgreSQL BDR (Bi-Directional Replication)
-- ═══════════════════════════════════════════════════════

-- BDR (by 2ndQuadrant/EDB) supports multi-master
-- Uses logical replication under the hood

-- Conflict handling options:
-- • last_update_wins (default)
-- • first_update_wins
-- • Custom conflict handler functions

-- Example conflict handler:
CREATE OR REPLACE FUNCTION custom_conflict_handler(
    row1 RECORD, row2 RECORD, 
    conflict_type TEXT, conflict_resolution TEXT
) RETURNS RECORD AS $$
BEGIN
    -- Custom logic: prefer the row with higher version
    IF row1.version > row2.version THEN
        RETURN row1;
    ELSE
        RETURN row2;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

---

## 🕸️ Topology 3: Leaderless Replication (Quorum-Based)

> **No leader at all — any node can accept reads AND writes. Used by Cassandra, DynamoDB, Riak.**

### How It Works

```
              Leaderless Replication
              ═════════════════════

         Client writes to MULTIPLE nodes simultaneously
         Client reads from MULTIPLE nodes simultaneously

              ┌──────────┐
              │  Client  │
              └─────┬────┘
                    │
          Write to 3 out of 5 nodes
                    │
       ┌────────────┼────────────┐──────────────┐
       │            │            │              │
       ▼            ▼            ▼              │
  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────▼┐ ┌─────────┐
  │ Node 1  │ │ Node 2  │ │ Node 3  │ │ Node 4  │ │ Node 5  │
  │         │ │         │ │         │ │         │ │         │
  │ Got the │ │ Got the │ │ Got the │ │ Didn't  │ │ Didn't  │
  │ write ✅│ │ write ✅│ │ write ✅│ │ get it ❌│ │ get it ❌│
  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘

  Write succeeds if W nodes confirm (e.g., W=3 out of N=5) ✅
  
  Later, client reads from R nodes (e.g., R=3 out of N=5):
  → Gets 3 responses, at least one has the latest value
  → Returns the latest version
```

### Quorum Formula — The Math That Matters

```
┌──────────────────────────────────────────────────────────────┐
│                    QUORUM FORMULA                             │
│                                                               │
│    N = Total number of replicas                               │
│    W = Number of nodes that must confirm a WRITE              │
│    R = Number of nodes that must respond to a READ            │
│                                                               │
│    ┌─────────────────────────────────────────────────┐        │
│    │                                                 │        │
│    │         W + R > N  →  Strong Consistency         │        │
│    │                                                 │        │
│    └─────────────────────────────────────────────────┘        │
│                                                               │
│    If W + R > N, then the read set and write set OVERLAP.    │
│    At least one node in the read set has the latest write.   │
│                                                               │
│    Example: N=5, W=3, R=3                                    │
│    W + R = 6 > 5 ✅ → guaranteed to read latest write        │
│                                                               │
│    ┌───┬───┬───┬───┬───┐                                     │
│    │ 1 │ 2 │ 3 │ 4 │ 5 │  N=5 nodes                        │
│    ├───┼───┼───┼───┼───┤                                     │
│    │ W │ W │ W │   │   │  W=3 (wrote to 1,2,3)              │
│    ├───┼───┼───┼───┼───┤                                     │
│    │   │   │ R │ R │ R │  R=3 (read from 3,4,5)             │
│    └───┴───┴───┴───┴───┘                                     │
│                  ↑                                            │
│              Node 3 has both the write AND is in read set!   │
│              → Guaranteed to see latest data ✅               │
│                                                               │
│  TUNING:                                                      │
│  • W=N, R=1: Fast reads, slow writes (read-heavy workload)  │
│  • W=1, R=N: Fast writes, slow reads (write-heavy workload) │
│  • W=⌈N/2⌉+1, R=⌈N/2⌉+1: Balanced (most common)           │
│  • W=1, R=1: FAST but NO consistency (eventual only)         │
│                                                               │
│  COMMON CONFIGS:                                              │
│  N=3, W=2, R=2 → Majority quorum (typical)                  │
│  N=5, W=3, R=3 → Tolerates 2 failures                       │
│  N=3, W=1, R=1 → Eventual consistency (fast but risky)      │
└──────────────────────────────────────────────────────────────┘
```

### Read Repair & Anti-Entropy

```
  When a read finds STALE data on some nodes, fix it!

  READ REPAIR:
  ════════════

  Client reads from 3 nodes:
  
  Node 1: { name: "Bob",    version: 5 }  ← LATEST ✅
  Node 3: { name: "Alice",  version: 3 }  ← STALE ❌
  Node 5: { name: "Bob",    version: 5 }  ← LATEST ✅
  
  Client sees version 5 is latest → returns "Bob"
  AND sends a repair write to Node 3 with version 5 data!
  
  Node 3: { name: "Bob", version: 5 }  ← REPAIRED ✅

  ─────────────────────────────────────────────────

  ANTI-ENTROPY (Background Process):
  ═══════════════════════════════════
  
  A background process constantly compares data between nodes
  and copies missing data to bring nodes into sync.
  
  Like a librarian who walks through all shelves daily,
  checking that every shelf has the same books.
  
  Merkle Trees: Efficiently compare large datasets
  → Hash each row → Hash groups → Hash of hashes
  → Compare root hash → if different, drill down to find differences
  → Only transfer the DIFFERENT rows (not everything)
```

### Sloppy Quorums & Hinted Handoff

```
  PROBLEM: What if W nodes from the designated set are DOWN?

  N=3, W=2, designated nodes for key "X" = {A, B, C}
  But B and C are down!

  Strict Quorum:  FAIL! Can't write to 2 out of {A, B, C} ❌
  Sloppy Quorum:  Write to A + D (any available node) ✅

  ┌────────────────────────────────────────────────────────────┐
  │  HINTED HANDOFF:                                           │
  │                                                            │
  │  When D receives a write meant for B:                      │
  │  → D stores the data with a "hint": "This belongs to B"   │
  │  → When B comes back online                                │
  │  → D forwards the data to B                               │
  │  → D deletes its temporary copy                            │
  │                                                            │
  │  Like a neighbor accepting a package when you're away,     │
  │  and giving it to you when you get home. 📦                │
  └────────────────────────────────────────────────────────────┘
```

### Leaderless Replication in Real Databases

```cql
-- ═══════════════════════════════════════════════════════
-- Cassandra Consistency Levels
-- ═══════════════════════════════════════════════════════

-- Write with quorum consistency (W = majority)
INSERT INTO users (user_id, name, email)
VALUES (uuid(), 'Alice', 'alice@example.com')
USING CONSISTENCY QUORUM;

-- Read with quorum consistency (R = majority)
SELECT * FROM users WHERE user_id = some_uuid
USING CONSISTENCY QUORUM;

-- Consistency Levels in Cassandra:
-- ONE         → W=1 or R=1 (fastest, least safe)
-- TWO         → W=2 or R=2
-- THREE       → W=3 or R=3
-- QUORUM      → W=⌈N/2⌉+1 (majority of replicas)
-- LOCAL_QUORUM→ Majority in LOCAL data center only
-- EACH_QUORUM → Majority in EACH data center
-- ALL         → W=N or R=N (slowest, most safe)

-- Typical production setup:
-- Writes: LOCAL_QUORUM (fast, consistent within DC)
-- Reads:  LOCAL_QUORUM
-- Together: W + R > N → strong consistency within DC
```

```python
# ═══════════════════════════════════════════════════════
# DynamoDB — Leaderless with Managed Infrastructure
# ═══════════════════════════════════════════════════════

import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

# Eventually consistent read (default — fast, might be stale)
response = table.get_item(
    Key={'user_id': '42'},
    ConsistentRead=False  # Default
)

# Strongly consistent read (slower, always latest)
response = table.get_item(
    Key={'user_id': '42'},
    ConsistentRead=True  # Routes to leader partition
)

# DynamoDB internally uses:
# → 3 replicas per partition
# → Writes acknowledged by 2 of 3 (W=2)
# → Eventually consistent reads from any 1 replica (R=1)
# → Strongly consistent reads from leader (R=1 from leader)
```

---

## ⚔️ Replication Topology Comparison

```
┌───────────────────┬──────────────────┬──────────────────┬──────────────────┐
│     Feature       │  Single-Leader   │  Multi-Leader    │   Leaderless     │
├───────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Write nodes       │ 1 (Primary only) │ Multiple leaders │ Any node         │
│ Read nodes        │ All (any replica)│ All              │ Any node         │
│ Consistency       │ Strong possible  │ Eventual (mostly)│ Tunable (quorum) │
│ Write latency     │ One hop          │ Low (local DC)   │ Multi-hop        │
│ Conflicts         │ None!            │ YES (complex)    │ YES (versioning) │
│ Failover          │ Needed (promote) │ Automatic        │ Not needed       │
│ Complexity        │ 🟢 Low           │ 🔴 High          │ 🟡 Medium        │
│ Data loss risk    │ Depends on sync  │ Conflicts can    │ Tunable (W/R)    │
│                   │ vs async         │ lose updates     │                  │
│                   │                  │                  │                  │
│ Used by           │ PostgreSQL       │ MySQL Group Rep  │ Cassandra        │
│                   │ MySQL (default)  │ CouchDB          │ DynamoDB         │
│                   │ MongoDB          │ PostgreSQL BDR   │ Riak             │
│                   │ SQL Server       │ Oracle GoldenGate│ Voldemort        │
│                   │                  │                  │                  │
│ Best for          │ Most apps        │ Multi-datacenter │ High write       │
│                   │ (default choice) │ Collaborative    │ throughput       │
│                   │                  │ Offline-first    │ Always available │
└───────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

## 📡 Physical vs Logical Replication

```
┌──────────────────────────────────────────────────────────────┐
│  PHYSICAL REPLICATION (Byte-level)                           │
│  ═══════════════════════════════                             │
│                                                              │
│  Ships raw WAL/Redo log bytes from primary to replica.      │
│  Replica applies the EXACT same disk changes.                │
│                                                              │
│  ✅ Exact copy (byte-for-byte identical)                    │
│  ✅ Supports all features (DDL, sequences, etc.)            │
│  ✅ Lower overhead                                           │
│  ❌ Must be same DB version                                  │
│  ❌ Must be same OS/architecture                             │
│  ❌ Can't replicate subset of tables                        │
│  ❌ Can't transform data during replication                  │
│                                                              │
│  Used by: PostgreSQL streaming replication                   │
│           MySQL binlog (ROW format)                          │
│           Oracle Data Guard                                   │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  LOGICAL REPLICATION (Row-level)                             │
│  ═══════════════════════════════                             │
│                                                              │
│  Ships INSERT/UPDATE/DELETE operations (logical changes).    │
│  Replica re-executes the operations.                         │
│                                                              │
│  ✅ Can replicate between different DB versions              │
│  ✅ Can replicate specific tables/schemas                   │
│  ✅ Can transform data during replication                    │
│  ✅ Can replicate to different DB engines!                   │
│  ❌ Higher overhead (parsing + executing SQL)                │
│  ❌ DDL changes not always replicated                       │
│  ❌ Sequences/large objects may not replicate               │
│                                                              │
│  Used by: PostgreSQL logical replication                     │
│           MySQL GTID replication                              │
│           Oracle GoldenGate                                   │
│           Debezium (Change Data Capture)                      │
└──────────────────────────────────────────────────────────────┘
```

### PostgreSQL: Physical vs Logical

```sql
-- ═══════════════════════════════════════════════════════
-- Physical Replication (WAL Shipping)
-- ═══════════════════════════════════════════════════════

-- Primary postgresql.conf:
wal_level = replica         -- minimum for physical replication
max_wal_senders = 5

-- Replica is an EXACT COPY of primary
-- Read-only, can't even create temporary tables
-- Perfect for HA failover

-- ═══════════════════════════════════════════════════════
-- Logical Replication (Publication/Subscription)
-- ═══════════════════════════════════════════════════════

-- On PRIMARY: Create a publication
CREATE PUBLICATION my_pub FOR TABLE orders, products;

-- Or publish everything:
CREATE PUBLICATION all_changes FOR ALL TABLES;

-- On SUBSCRIBER (can be different PG version!):
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=primary dbname=mydb user=replicator password=pw'
    PUBLICATION my_pub;

-- Now 'orders' and 'products' tables sync automatically!
-- Subscriber can have ADDITIONAL tables, indexes, etc.
-- Subscriber is WRITABLE (be careful with conflicts!)
```

---

## 🧪 Change Data Capture (CDC) — Modern Replication

```
  CDC captures every change to your database and streams it
  as events to other systems.

  ┌────────────┐     ┌──────────────┐     ┌────────────────┐
  │  Database  │────►│  CDC Tool    │────►│  Kafka / Event │
  │            │     │  (Debezium)  │     │  Stream        │
  │  INSERT ✅ │     │              │     │                │
  │  UPDATE ✅ │     │  Reads WAL / │     │  Consumers:    │
  │  DELETE ✅ │     │  Binlog      │     │  • Search Index│
  │            │     │              │     │  • Cache       │
  └────────────┘     └──────────────┘     │  • Analytics   │
                                          │  • Another DB  │
                                          │  • Audit Log   │
                                          └────────────────┘

  WHY CDC?
  → Replicate to Elasticsearch for search
  → Replicate to Redis for caching
  → Replicate to data warehouse for analytics
  → Build event-driven architectures
  → Audit trail of every change ever made
```

```yaml
# Debezium CDC Configuration (for PostgreSQL)
# Deployed as a Kafka Connect connector

{
  "name": "pg-cdc-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres-primary",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "secret",
    "database.dbname": "myapp",
    "database.server.name": "myapp-server",
    "table.include.list": "public.orders,public.users",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot"
  }
}

# Every INSERT/UPDATE/DELETE on 'orders' and 'users' tables
# produces a Kafka event like:
# Topic: myapp-server.public.orders
# Key: {"order_id": 42}
# Value: {
#   "before": {"status": "pending", "amount": 99.99},
#   "after":  {"status": "shipped", "amount": 99.99},
#   "op": "u",  // u=update, c=create, d=delete
#   "ts_ms": 1717200000000
# }
```

---

## 📊 Replication in Different Databases — Summary

```
┌────────────────┬─────────────┬─────────────┬─────────────────────────┐
│   Database     │  Default    │   Options   │   Failover              │
│                │  Topology   │             │                         │
├────────────────┼─────────────┼─────────────┼─────────────────────────┤
│ PostgreSQL     │ Single-     │ Physical,   │ Patroni, pg_auto_       │
│                │ Leader      │ Logical,    │ failover, Stolon        │
│                │             │ Sync/Async  │                         │
├────────────────┼─────────────┼─────────────┼─────────────────────────┤
│ MySQL          │ Single-     │ Async, Semi-│ MySQL Router, ProxySQL, │
│                │ Leader      │ sync, Group │ Orchestrator, MHA       │
│                │             │ Replication │                         │
├────────────────┼─────────────┼─────────────┼─────────────────────────┤
│ MongoDB        │ Single-     │ Replica Set │ Automatic (built-in     │
│                │ Leader      │ (auto       │ election via Raft)      │
│                │             │ failover)   │                         │
├────────────────┼─────────────┼─────────────┼─────────────────────────┤
│ SQL Server     │ Single-     │ Always On AG│ Automatic/Manual        │
│                │ Leader      │ Log Shipping│ failover with AG        │
│                │             │ Mirroring   │                         │
├────────────────┼─────────────┼─────────────┼─────────────────────────┤
│ Oracle         │ Single-     │ Data Guard  │ Fast-Start Failover,    │
│                │ Leader      │ (Physical/  │ Active Data Guard       │
│                │             │ Logical)    │                         │
├────────────────┼─────────────┼─────────────┼─────────────────────────┤
│ Cassandra      │ Leaderless  │ Tunable     │ No failover needed!     │
│                │             │ Consistency │ Any node handles any    │
│                │             │ (ONE→ALL)   │ request                 │
├────────────────┼─────────────┼─────────────┼─────────────────────────┤
│ DynamoDB       │ Leaderless  │ Eventually  │ Fully managed (AWS)     │
│                │ (managed)   │ or Strongly │ No failover to manage   │
│                │             │ consistent  │                         │
├────────────────┼─────────────┼─────────────┼─────────────────────────┤
│ CockroachDB    │ Leaderless  │ Serializable│ Automatic via Raft      │
│                │ (Raft-based)│ by default! │ consensus               │
└────────────────┴─────────────┴─────────────┴─────────────────────────┘
```

---

## 🎯 Replication Decision Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                REPLICATION DECISION MATRIX                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Need HA with simple setup?                                       │
│  → Single-Leader + automatic failover (PostgreSQL + Patroni)     │
│                                                                   │
│  Need read scaling?                                               │
│  → Single-Leader + read replicas                                  │
│                                                                   │
│  Need multi-region writes?                                        │
│  → Multi-Leader (but prepare for conflicts)                      │
│  → OR use CockroachDB/Spanner (distributed SQL, no conflicts)   │
│                                                                   │
│  Need maximum write throughput?                                   │
│  → Leaderless (Cassandra, DynamoDB)                              │
│                                                                   │
│  Need zero data loss?                                             │
│  → Synchronous replication (slower writes, safe data)            │
│                                                                   │
│  Need to replicate to other systems (search, cache, analytics)?  │
│  → CDC with Debezium → Kafka → consumers                        │
│                                                                   │
│  Building offline-first app?                                      │
│  → CouchDB / PouchDB with sync protocol                         │
│                                                                   │
│  GOLDEN RULE:                                                     │
│  Start with Single-Leader Async Replication.                      │
│  Upgrade ONLY when you have a specific reason to.                 │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

Now that you understand how data is **copied** across nodes, the next chapter covers how databases fit into **Microservices Architecture** — where the real complexity begins.

> **Chapter 7.3:** [Database in Microservices Architecture →](./03-DB-in-Microservices.md)

---

> _"Replication seems simple until you realize that the speed of light is finite, networks are unreliable, and clocks lie."_
> — Every distributed systems engineer
