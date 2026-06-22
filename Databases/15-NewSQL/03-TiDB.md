# 🐯 Chapter 4.3 — TiDB — MySQL-Compatible Distributed Database

> **Level:** 🟡 Intermediate
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 4.1 (NewSQL Overview), MySQL basics

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand TiDB's **unique 3-layer architecture** (TiDB + TiKV + PD)
- Know how TiDB achieves **HTAP** — run OLTP and OLAP in ONE database
- Write **MySQL-compatible SQL** on TiDB without changing your app code
- Understand **TiFlash** — the columnar engine that gives TiDB analytical superpowers
- Design schemas that work well with TiDB's **distributed nature**
- Know when TiDB is the **best choice** vs CockroachDB, Spanner, or plain MySQL

---

## 🧠 The Origin Story

### Born at PingCAP — Inspired by Google

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   2015: Three engineers in Beijing (Ed Huang, Dylan Cui, Max Liu)   ║
║         founded PingCAP with a radical idea:                        ║
║                                                                      ║
║   "What if we combined Google Spanner's distributed SQL             ║
║    with MySQL's compatibility — so ANYONE could use it?"            ║
║                                                                      ║
║   They built two things:                                             ║
║                                                                      ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │  TiKV (2016)  — Distributed key-value store               │    ║
║   │                  Inspired by Google's Spanner storage      │    ║
║   │                  Written in Rust 🦀 (performance + safety)│    ║
║   │                  Donated to CNCF (Cloud Native Computing) │    ║
║   │                                                            │    ║
║   │  TiDB (2017)  — MySQL-compatible SQL layer on top of TiKV│    ║
║   │                  Written in Go 🐹 (productivity + concur.)│    ║
║   │                  Drop-in MySQL replacement                 │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║   The name: Ti = Titanium (strong, lightweight)                     ║
║             DB = Database                                            ║
║                                                                      ║
║   2020: TiDB 4.0 — TiFlash (real-time analytics engine)            ║
║   2022: TiDB 6.0 — Production-grade HTAP                           ║
║   2024: TiDB Serverless — fully managed, pay-per-use               ║
║                                                                      ║
║   Today: Used by 3000+ companies worldwide                          ║
║   Banks, e-commerce, gaming, fintech, telecom                       ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🏗️ Architecture — The 3-Component Design

> TiDB is NOT a single binary. It's **3 separate components** that work together. This is what makes it special.

```
╔══════════════════════════════════════════════════════════════════════╗
║                     TiDB ARCHITECTURE                                ║
║                                                                      ║
║   Your App (MySQL client/driver)                                    ║
║        │                                                             ║
║        │  MySQL Protocol (port 4000)                                ║
║        ▼                                                             ║
║   ┌─────────────────────────────────────────────────────┐           ║
║   │              TiDB Server (Stateless)                │           ║
║   │   • Parses SQL → Plans → Optimizes → Executes      │           ║
║   │   • Written in Go                                   │           ║
║   │   • Horizontally scalable (add more TiDB servers)  │           ║
║   │   • STATELESS — no data stored here!               │           ║
║   └───────────────────┬─────────────────────────────────┘           ║
║                       │                                              ║
║           ┌───────────┼───────────┐                                 ║
║           │           │           │                                  ║
║           ▼           ▼           ▼                                  ║
║   ┌───────────┐ ┌───────────┐ ┌───────────┐                       ║
║   │  TiKV     │ │  TiKV     │ │  TiKV     │   (Row Store)         ║
║   │  Node 1   │ │  Node 2   │ │  Node 3   │   OLTP Workloads      ║
║   │  (Rust)   │ │  (Rust)   │ │  (Rust)   │                       ║
║   └───────────┘ └───────────┘ └───────────┘                       ║
║        │              │              │                               ║
║        └──────────────┼──────────────┘                               ║
║                       │                                              ║
║              Raft Consensus (per Region)                            ║
║                                                                      ║
║   ┌───────────┐ ┌───────────┐ ┌───────────┐                       ║
║   │ TiFlash   │ │ TiFlash   │ │ TiFlash   │   (Columnar Store)    ║
║   │ Node 1    │ │ Node 2    │ │ Node 3    │   OLAP Workloads      ║
║   │ (C++)     │ │ (C++)     │ │ (C++)     │                       ║
║   └───────────┘ └───────────┘ └───────────┘                       ║
║                                                                      ║
║   ┌─────────────────────────────────────────────────────┐           ║
║   │              PD (Placement Driver)                   │           ║
║   │   • Cluster brain — metadata, scheduling, TSO       │           ║
║   │   • Assigns timestamps (Timestamp Oracle)           │           ║
║   │   • Balances data across TiKV/TiFlash nodes        │           ║
║   │   • Written in Go                                   │           ║
║   └─────────────────────────────────────────────────────┘           ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Component Breakdown

#### TiDB Server (The SQL Brain)

```
╔═══════════════════════════════════════════════════════╗
║                  TiDB SERVER                          ║
║                                                       ║
║   Role: Parse SQL, optimize, coordinate execution     ║
║   Language: Go                                        ║
║   Stateless: YES — scales by adding more servers      ║
║   Protocol: MySQL wire protocol (port 4000)           ║
║                                                       ║
║   ┌─── What happens when you send a query ──────┐   ║
║   │                                              │   ║
║   │  1. Parse SQL → AST (Abstract Syntax Tree)   │   ║
║   │  2. Validate (tables exist? columns valid?)   │   ║
║   │  3. Optimize (cost-based, stats-aware)       │   ║
║   │  4. Build execution plan                     │   ║
║   │  5. Route to correct TiKV regions            │   ║
║   │  6. Collect results, return to client        │   ║
║   │                                              │   ║
║   └──────────────────────────────────────────────┘   ║
║                                                       ║
║   💡 Because TiDB is stateless, you can put a        ║
║      load balancer in front of multiple TiDB servers  ║
║      for horizontal read scaling!                     ║
║                                                       ║
║   [App] → [HAProxy / F5] → [TiDB1, TiDB2, TiDB3]   ║
║                                                       ║
╚═══════════════════════════════════════════════════════╝
```

#### TiKV (The Distributed Storage Engine)

```
╔═══════════════════════════════════════════════════════════════╗
║                      TiKV                                     ║
║                                                               ║
║   Role: Store data, handle distributed transactions           ║
║   Language: Rust 🦀 (safety + bare-metal performance)        ║
║   Engine: RocksDB (LSM tree) underneath                       ║
║   Replication: Raft consensus per Region                      ║
║                                                               ║
║   Data Organization:                                          ║
║                                                               ║
║   ┌── Region (default: 96 MB) ──────────────────────────┐   ║
║   │                                                      │   ║
║   │  Key range: [table_42_row_1 ... table_42_row_50000] │   ║
║   │                                                      │   ║
║   │  Replicated across 3 TiKV nodes via Raft:           │   ║
║   │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │   ║
║   │  │ Leader   │  │ Follower │  │ Follower │          │   ║
║   │  │ TiKV-1   │  │ TiKV-2   │  │ TiKV-3   │          │   ║
║   │  └──────────┘  └──────────┘  └──────────┘          │   ║
║   │                                                      │   ║
║   │  When region grows > 96 MB → auto-split into 2      │   ║
║   │  When regions are too small → auto-merge             │   ║
║   └──────────────────────────────────────────────────────┘   ║
║                                                               ║
║   Transaction Model:                                          ║
║   • Percolator-based (Google's distributed txn protocol)     ║
║   • 2-Phase Commit with optimistic locking                   ║
║   • MVCC — multiple versions per key                         ║
║   • Snapshot Isolation (SI) by default                        ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

#### PD — Placement Driver (The Cluster Brain)

```
╔═══════════════════════════════════════════════════════════════╗
║                  PD (Placement Driver)                        ║
║                                                               ║
║   Role: Cluster metadata + scheduling + timestamp oracle      ║
║   Language: Go                                                ║
║   Deployment: 3 or 5 nodes (odd number for Raft quorum)      ║
║                                                               ║
║   Critical Functions:                                         ║
║                                                               ║
║   1. TIMESTAMP ORACLE (TSO)                                   ║
║      ┌────────────────────────────────────────────────┐      ║
║      │ Every transaction gets a globally unique        │      ║
║      │ timestamp from PD. This guarantees ordering.   │      ║
║      │                                                 │      ║
║      │ Txn1 → ts=100                                   │      ║
║      │ Txn2 → ts=101                                   │      ║
║      │ Txn3 → ts=102                                   │      ║
║      │                                                 │      ║
║      │ PD generates ~million timestamps per second     │      ║
║      └────────────────────────────────────────────────┘      ║
║                                                               ║
║   2. REGION SCHEDULER                                         ║
║      • Balances regions across TiKV nodes                    ║
║      • Moves hot regions away from overloaded nodes          ║
║      • Ensures 3 replicas are on different physical machines ║
║                                                               ║
║   3. METADATA STORE                                           ║
║      • Which TiKV node has which regions                     ║
║      • Cluster topology and health                           ║
║                                                               ║
║   ⚠️ PD is a potential bottleneck:                            ║
║   Every transaction needs a timestamp from PD.               ║
║   PD uses batching + caching to handle millions of requests. ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

#### TiFlash — The HTAP Secret Weapon

```
╔══════════════════════════════════════════════════════════════════════╗
║                        TiFlash                                       ║
║                                                                      ║
║   Role: Real-time columnar analytics alongside OLTP                 ║
║   Language: C++ (ClickHouse-derived engine)                          ║
║   Data: Replicated from TiKV via Raft Learner                       ║
║                                                                      ║
║   ┌─── The HTAP Magic ──────────────────────────────────────────┐   ║
║   │                                                              │   ║
║   │  TiKV stores data in ROW format:                             │   ║
║   │  ┌────┬───────┬─────┬─────────┐                             │   ║
║   │  │ id │ name  │ age │ balance │  ← Great for point queries  │   ║
║   │  ├────┼───────┼─────┼─────────┤     (SELECT * WHERE id=42)  │   ║
║   │  │ 1  │ Alice │ 28  │ 1000    │                             │   ║
║   │  │ 2  │ Bob   │ 35  │ 500     │                             │   ║
║   │  └────┴───────┴─────┴─────────┘                             │   ║
║   │                                                              │   ║
║   │  TiFlash stores SAME data in COLUMN format:                  │   ║
║   │  id:      [1, 2, 3, 4, 5, ...]     ← Great for analytics   │   ║
║   │  name:    [Alice, Bob, Carol, ...]     (SUM(balance),       │   ║
║   │  age:     [28, 35, 42, ...]            AVG(age), etc.)      │   ║
║   │  balance: [1000, 500, 2000, ...]                            │   ║
║   │                                                              │   ║
║   │  Changes flow from TiKV → TiFlash via Raft Learner:        │   ║
║   │  • Learner = receives Raft logs but DOESN'T vote            │   ║
║   │  • Near-real-time replication (sub-second lag)              │   ║
║   │  • Does NOT slow down TiKV writes                           │   ║
║   │                                                              │   ║
║   └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║   The optimizer AUTOMATICALLY chooses TiKV or TiFlash:              ║
║   • Point query (SELECT WHERE id=42) → TiKV (row store)            ║
║   • Analytics (SELECT SUM(balance) GROUP BY city) → TiFlash        ║
║   • You don't need to change your SQL! The optimizer decides.      ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

> 💡 **Why This Matters:** Without TiFlash, companies need separate OLTP (MySQL) + OLAP (Snowflake/BigQuery) databases + ETL pipelines to sync them. TiDB eliminates the ETL completely.

---

## ⚙️ Installation & Setup

### Option 1: TiUP — The Official Package Manager

```bash
# Install TiUP
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh

# Add TiUP to PATH
source ~/.bashrc

# Start a local playground (1 TiDB + 1 TiKV + 1 PD + 1 TiFlash)
tiup playground

# Output:
# TiDB:    127.0.0.1:4000
# PD:      127.0.0.1:2379
# TiFlash: 127.0.0.1:9000

# Connect with MySQL client!
mysql -h 127.0.0.1 -P 4000 -u root
```

### Option 2: Docker Compose (Multi-Node)

```yaml
# docker-compose.yml
version: '3.8'
services:
  pd:
    image: pingcap/pd:latest
    ports: ["2379:2379"]
    command:
      - --name=pd
      - --client-urls=http://0.0.0.0:2379

  tikv:
    image: pingcap/tikv:latest
    depends_on: [pd]
    command:
      - --pd-endpoints=pd:2379
      - --addr=0.0.0.0:20160

  tidb:
    image: pingcap/tidb:latest
    depends_on: [tikv]
    ports: ["4000:4000", "10080:10080"]
    command:
      - --store=tikv
      - --path=pd:2379
```

### Option 3: TiDB Serverless (Cloud — Free Tier)

```
1. Go to tidbcloud.com
2. Sign up → Create a "Serverless" cluster (free!)
3. Choose region (US, EU, Asia)
4. Get connection string:

   mysql -h gateway01.us-east-1.prod.aws.tidbcloud.com \
         -P 4000 -u username --ssl-mode=VERIFY_IDENTITY \
         --ssl-ca=/etc/ssl/certs/ca-certificates.crt -p
```

---

## 💻 SQL in TiDB — It's MySQL!

### Schema Design

```sql
-- TiDB supports MySQL syntax with distributed extensions

-- Standard MySQL table
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- Works! But see note below
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    age TINYINT UNSIGNED,
    city VARCHAR(100),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_city (city),
    INDEX idx_email (email)
) ENGINE=InnoDB;  -- Always InnoDB in TiDB

-- ⚠️ AUTO_INCREMENT in TiDB is NOT sequential across TiDB servers
-- Each TiDB server pre-allocates a batch of IDs (30000 by default)
-- TiDB Server 1: IDs 1-30000
-- TiDB Server 2: IDs 30001-60000
-- Result: IDs are unique but NOT contiguous

-- If you need sequential IDs, use AUTO_ID_CACHE=1 (slower)
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY AUTO_ID_CACHE=1,
    user_id BIGINT NOT NULL,
    amount DECIMAL(10,2),
    status ENUM('pending','processing','shipped','delivered'),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### TiDB-Specific Table Options

```sql
-- SHARD_ROW_ID_BITS: Scatter data across TiKV nodes for write-heavy tables
-- Prevents hot spot on the auto-increment range
CREATE TABLE events (
    id BIGINT AUTO_RANDOM PRIMARY KEY,  -- AUTO_RANDOM: random bit prefix!
    event_type VARCHAR(50),
    payload JSON,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- AUTO_RANDOM explained:
-- Instead of: 1, 2, 3, 4, 5 (all go to same region → hot spot)
-- AUTO_RANDOM: 48901, 12847, 73291, 55102 (spread across regions!)
-- The randomness is in the high bits, so it's still globally unique

-- Pre-split a table into multiple regions at creation
CREATE TABLE logs (
    id BIGINT PRIMARY KEY AUTO_RANDOM,
    level VARCHAR(10),
    message TEXT,
    ts DATETIME
) PRE_SPLIT_REGIONS=4;
-- Creates 2^4 = 16 regions immediately → instant write parallelism
```

### HTAP in Action — TiFlash

```sql
-- Step 1: Add a TiFlash replica for a table
ALTER TABLE orders SET TIFLASH REPLICA 1;

-- Step 2: Check replication status
SELECT * FROM information_schema.tiflash_replica;

-- Step 3: Run analytics — optimizer auto-selects TiFlash!

-- This query will use TiFlash (columnar = fast for aggregations)
SELECT 
    city,
    COUNT(*) AS user_count,
    AVG(age) AS avg_age,
    SUM(o.amount) AS total_revenue
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.created_at >= '2024-01-01'
GROUP BY city
ORDER BY total_revenue DESC;

-- Force the optimizer to use TiFlash (if needed)
SELECT /*+ READ_FROM_STORAGE(TIFLASH[orders]) */ 
    DATE(created_at) AS day,
    SUM(amount) AS daily_revenue
FROM orders
GROUP BY day
ORDER BY day;

-- Force the optimizer to use TiKV (for point queries)
SELECT /*+ READ_FROM_STORAGE(TIKV[users]) */
    * FROM users WHERE id = 42;
```

### Practical HTAP Example — Real-Time Dashboard

```sql
-- Scenario: E-commerce dashboard that updates in real-time
-- Traditional approach: MySQL → ETL (hourly) → Redshift → Dashboard
-- TiDB approach: One database. One query. Real-time.

-- Real-time revenue by hour (runs on TiFlash, sub-second!)
SELECT 
    DATE_FORMAT(created_at, '%Y-%m-%d %H:00:00') AS hour,
    COUNT(*) AS order_count,
    SUM(amount) AS revenue,
    AVG(amount) AS avg_order_value,
    COUNT(DISTINCT user_id) AS unique_customers
FROM orders
WHERE created_at >= NOW() - INTERVAL 24 HOUR
GROUP BY hour
ORDER BY hour DESC;

-- Meanwhile, OLTP queries run on TiKV with zero impact!
-- INSERT into orders keeps working at full speed
INSERT INTO orders (user_id, amount, status) 
VALUES (12345, 99.99, 'pending');
-- ↑ This goes to TiKV (row store) — not affected by TiFlash queries
```

---

## 🔬 Transaction Model Deep Dive

### Percolator — TiDB's Transaction Protocol

```
╔══════════════════════════════════════════════════════════════════════╗
║             TiDB's Transaction Flow (Percolator-based)              ║
║                                                                      ║
║   Inspired by Google's Percolator paper (2010)                      ║
║   Optimistic by default, Pessimistic mode available                 ║
║                                                                      ║
║   BEGIN;                                                             ║
║     UPDATE accounts SET balance = balance - 100 WHERE id = 1;      ║
║     UPDATE accounts SET balance = balance + 100 WHERE id = 2;      ║
║   COMMIT;                                                            ║
║                                                                      ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │ Phase 1: PREWRITE                                          │    ║
║   │                                                            │    ║
║   │ 1. Get start_ts from PD (e.g., ts=100)                    │    ║
║   │ 2. Pick one key as "primary" (accounts/id=1)              │    ║
║   │ 3. Write LOCK + data to each key:                          │    ║
║   │    accounts/id=1 → {lock: primary, ts:100, val:-100}      │    ║
║   │    accounts/id=2 → {lock: @primary, ts:100, val:+100}     │    ║
║   │ 4. Each write goes through Raft on the key's region       │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │ Phase 2: COMMIT                                            │    ║
║   │                                                            │    ║
║   │ 1. Get commit_ts from PD (e.g., ts=101)                   │    ║
║   │ 2. Write COMMIT record on the PRIMARY key:                 │    ║
║   │    accounts/id=1 → {committed: ts=101}                     │    ║
║   │ 3. Once primary is committed → transaction is committed!   │    ║
║   │ 4. Async: clean up secondary locks                         │    ║
║   │    accounts/id=2 → {committed: ts=101}                     │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║   KEY INSIGHT: The transaction is committed the moment the          ║
║   primary lock is committed. Secondary cleanup is async.            ║
║   If TiDB crashes after primary commit → secondaries are           ║
║   resolved by the next reader (lazy cleanup).                       ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Optimistic vs Pessimistic Transactions

```sql
-- OPTIMISTIC (TiDB default before v4.0)
-- Locks are NOT acquired during execution — only during COMMIT
-- If conflict detected at commit → transaction fails → retry

SET tidb_txn_mode = 'optimistic';
BEGIN OPTIMISTIC;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  -- No lock held yet! Other transactions can modify id=1
COMMIT;
-- At COMMIT time: if someone else modified id=1 → ERROR 9007 (Write Conflict)

-- Best for: Low-contention workloads (most writes hit different rows)
-- Worst for: High-contention (many transactions updating same rows)

-- PESSIMISTIC (TiDB default since v4.0 — like MySQL!)
-- Locks are acquired immediately when you modify a row

SET tidb_txn_mode = 'pessimistic';
BEGIN PESSIMISTIC;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  -- Lock acquired NOW! Other transactions must WAIT.
COMMIT;
-- Lock released at COMMIT.

-- Best for: High-contention workloads (banking, inventory)
-- This is what MySQL developers expect!
```

---

## 📊 Performance Tuning

### Reading Execution Plans

```sql
-- Basic EXPLAIN
EXPLAIN SELECT * FROM users WHERE city = 'New York';

-- EXPLAIN ANALYZE (executes and shows real timings)
EXPLAIN ANALYZE SELECT u.name, COUNT(o.id) as orders
FROM users u 
JOIN orders o ON u.id = o.user_id
GROUP BY u.name
HAVING COUNT(o.id) > 5;
```

```
+---------------------------+----------+------+---------------------------+
| id                        | estRows  | task | operator info             |
+---------------------------+----------+------+---------------------------+
| Selection_12              | 800.00   | root | gt(count(orders.id), 5)   |
| └─HashAgg_15              | 1000.00  | root | group by: users.name      |
|   └─HashJoin_18           | 10000.00 | root | inner join, eq: id=user_id|
|     ├─TableReader_22(B)   | 1000.00  | root |                           |
|     │ └─TableFullScan_21  | 1000.00  | cop  | table: users              |
|     └─TableReader_24(P)   | 10000.00 | root |                           |
|       └─TableFullScan_23  | 10000.00 | cop  | table: orders             |
+---------------------------+----------+------+---------------------------+

KEY TERMS:
  root = executed on TiDB server
  cop  = pushed down to TiKV (coprocessor) — this is GOOD!
  
TiDB pushes work DOWN to TiKV whenever possible:
  WHERE, GROUP BY, COUNT, SUM → executed on TiKV nodes in parallel
  Results aggregated on TiDB server
```

### TiDB-Specific Optimizations

```sql
-- 1. Check table regions (data distribution)
SHOW TABLE orders REGIONS;

-- 2. Check hot regions
SELECT * FROM information_schema.tikv_region_status
WHERE table_name = 'orders'
ORDER BY written_bytes DESC
LIMIT 10;

-- 3. Manually split a hot region
SPLIT TABLE orders BETWEEN (0) AND (1000000) REGIONS 10;

-- 4. Analyze tables (update statistics for optimizer)
ANALYZE TABLE orders;

-- 5. Use optimizer hints
-- Force index usage
SELECT /*+ USE_INDEX(orders, idx_user_id) */ 
    * FROM orders WHERE user_id = 42;

-- Force hash join
SELECT /*+ HASH_JOIN(u, o) */
    u.name, o.amount
FROM users u JOIN orders o ON u.id = o.user_id;

-- Force reading from TiFlash for analytics
SELECT /*+ READ_FROM_STORAGE(TIFLASH[orders]) */
    DATE(created_at), SUM(amount) 
FROM orders GROUP BY DATE(created_at);

-- 6. Session-level tuning
SET tidb_distsql_scan_concurrency = 30;  -- More parallel scanners
SET tidb_index_lookup_concurrency = 8;   -- More index lookups in parallel
SET tidb_hash_join_concurrency = 8;      -- More hash join workers
```

---

## 🔄 TiDB vs MySQL — Compatibility & Differences

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   ✅ COMPATIBLE with MySQL:                                          ║
║   • SQL syntax (99%+ MySQL 5.7/8.0 compatible)                     ║
║   • Data types (INT, VARCHAR, DATETIME, JSON, ENUM, etc.)           ║
║   • Indexes (B-tree, unique, composite, prefix)                     ║
║   • Stored procedures (basic support)                               ║
║   • Triggers (basic support)                                        ║
║   • Views                                                            ║
║   • MySQL protocol (port 4000 instead of 3306)                     ║
║   • MySQL drivers (JDBC, Python MySQL connector, etc.)              ║
║   • ORMs (Hibernate, SQLAlchemy, Sequelize, etc.)                   ║
║                                                                      ║
║   ⚠️ DIFFERENT from MySQL:                                           ║
║   • AUTO_INCREMENT: non-contiguous across TiDB servers              ║
║   • Foreign keys: supported since TiDB 6.6, but not recommended    ║
║     for distributed DBs (cross-region FK checks are slow)           ║
║   • SELECT ... FOR UPDATE: works, but locks are distributed        ║
║   • Temporary tables: limited support                               ║
║   • Some MySQL built-in functions differ slightly                   ║
║   • No SPATIAL indexes (as of 2024)                                 ║
║   • Charset: UTF8MB4 by default (MySQL default was latin1)          ║
║                                                                      ║
║   🆕 EXTRA features (not in MySQL):                                  ║
║   • AUTO_RANDOM (scatter writes across regions)                     ║
║   • TiFlash (real-time columnar analytics)                          ║
║   • SHOW TABLE REGIONS (see data distribution)                      ║
║   • Built-in dashboard with slow query analysis                     ║
║   • Changefeed (TiCDC — Change Data Capture)                       ║
║   • Backup & Restore with BR tool                                   ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🔧 TiDB Tooling Ecosystem

```
╔══════════════════════════════════════════════════════════════════════╗
║                    TiDB TOOL ECOSYSTEM                               ║
║                                                                      ║
║   ┌─── Deployment & Management ─────────────────────────────────┐   ║
║   │  TiUP        — Package manager & cluster deployer           │   ║
║   │  TiDB Operator — Kubernetes-native deployment               │   ║
║   │  TiDB Cloud   — Fully managed service                       │   ║
║   └─────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║   ┌─── Data Migration ──────────────────────────────────────────┐   ║
║   │  DM (Data Migration) — MySQL/MariaDB → TiDB (live sync!)   │   ║
║   │  Lightning     — Fast bulk import (TB in hours)             │   ║
║   │  Dumpling      — Logical export (like mysqldump, faster)    │   ║
║   └─────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║   ┌─── Change Data Capture ─────────────────────────────────────┐   ║
║   │  TiCDC         — Stream changes to Kafka, MySQL, S3, etc.  │   ║
║   └─────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║   ┌─── Backup & Restore ────────────────────────────────────────┐   ║
║   │  BR (Backup & Restore) — Distributed backup to S3/GCS      │   ║
║   │  PITR          — Point-In-Time Recovery                     │   ║
║   └─────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║   ┌─── Monitoring ──────────────────────────────────────────────┐   ║
║   │  TiDB Dashboard — Built-in web UI for diagnostics           │   ║
║   │  Prometheus     — Metrics collection                        │   ║
║   │  Grafana        — Pre-built dashboards                      │   ║
║   └─────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Migrating from MySQL to TiDB

```bash
# Step 1: Export from MySQL with Dumpling
dumpling -h mysql-host -P 3306 -u root -p password \
    -o /backup/mysql-dump \
    --filetype sql

# Step 2: Import to TiDB with Lightning (fast!)
tidb-lightning \
    -config lightning.toml \
    -d /backup/mysql-dump

# Step 3: Set up ongoing replication with DM
# DM reads MySQL binlog → applies to TiDB in real-time
# Your MySQL and TiDB stay in sync during migration
# When ready, switch traffic to TiDB. Done!
```

---

## 🌍 Real-World TiDB Use Cases

| Company | Scale | Use Case |
|---------|-------|----------|
| **iQIYI** (China's Netflix) | 100+ TiDB clusters | User profiles, watch history, recommendations |
| **Shopee** | 10+ PB data | E-commerce transactions, inventory |
| **PingCAP** (maker of TiDB) | Dogfooding | Their own internal systems |
| **Zhihu** (China's Quora) | Billions of rows | Posts, comments, user interactions |
| **BookMyShow** | Peak ticketing loads | Event ticketing with burst traffic |
| **Square** | Financial data | Payment processing |
| **PayPay** (Japan) | Mobile payments | Transaction processing |

---

## 🧠 Interview Quick-Fire — TiDB

| Question | Perfect Answer |
|----------|----------------|
| What is TiDB? | A MySQL-compatible distributed SQL database with real-time HTAP capability, built on a Raft-based distributed storage engine (TiKV). |
| What are TiDB's 3 components? | TiDB (stateless SQL layer, Go), TiKV (distributed KV storage, Rust), PD (cluster metadata + timestamp oracle, Go). |
| What is TiFlash? | A columnar storage engine that receives real-time replication from TiKV via Raft Learner, enabling analytics without ETL. |
| What does HTAP mean? | Hybrid Transactional/Analytical Processing — running OLTP and OLAP workloads on the same database without copying data. |
| How does TiDB handle transactions? | Percolator-based 2PC with MVCC. PD provides globally-ordered timestamps. Default is pessimistic locking (like MySQL). |
| What is AUTO_RANDOM? | A TiDB-specific feature that randomizes the high bits of auto-increment IDs to prevent write hot spots across regions. |
| When to choose TiDB over CockroachDB? | When you need MySQL compatibility, HTAP analytics, or are in the MySQL ecosystem. CockroachDB is better for PostgreSQL users and strongest isolation. |

---

## 📚 Chapter Summary

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. TiDB = MySQL-compatible distributed SQL with HTAP               ║
║  2. 3 components: TiDB (SQL) + TiKV (storage) + PD (brain)        ║
║  3. TiKV uses Raft consensus per Region (96 MB chunks)             ║
║  4. TiFlash = columnar analytics engine, synced via Raft Learner   ║
║  5. Optimizer auto-selects TiKV (OLTP) or TiFlash (OLAP)          ║
║  6. Use AUTO_RANDOM instead of AUTO_INCREMENT to avoid hot spots   ║
║  7. Pessimistic transactions by default (like MySQL)               ║
║  8. Rich tool ecosystem: DM (migration), TiCDC, Lightning, BR     ║
║  9. Best for: MySQL users who need scale + real-time analytics     ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

> **Next Chapter:** [Google Spanner & Cloud-Native SQL →](./04-Google-Spanner.md)
