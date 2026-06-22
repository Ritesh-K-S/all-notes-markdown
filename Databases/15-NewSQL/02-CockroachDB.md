# 🪳 Chapter 4.2 — CockroachDB — Survive Anything

> **Level:** 🟡 Intermediate | 🔥 High Demand
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 4.1 (NewSQL Overview), PostgreSQL basics

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand CockroachDB's **architecture** deeply enough to explain it in any interview
- Know how **ranges, leaseholders, and Raft groups** work internally
- Write **real SQL** on CockroachDB (it's PostgreSQL-compatible!)
- Design **multi-region** deployments that survive entire datacenter outages
- Tune CockroachDB for **performance** — indexes, query plans, hot spots
- Understand **serializable isolation** — the strongest guarantee in databases
- Know when CockroachDB is the **right** choice and when it's NOT

---

## 🧠 The Origin Story

### Why Is It Called "CockroachDB"?

> *"We named it after the cockroach because cockroaches are nearly impossible to kill."*
> — Spencer Kimball, Co-founder

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   The founders (Spencer Kimball, Peter Mattis, Ben Darnell)         ║
║   were all ex-Google engineers who worked on Google's internal      ║
║   infrastructure.                                                    ║
║                                                                      ║
║   They saw Google Spanner — global consistency, survives anything.  ║
║   But Spanner was locked inside Google. Not available to anyone.    ║
║                                                                      ║
║   Their mission: Build an OPEN SOURCE Spanner for the world.       ║
║                                                                      ║
║   2015: CockroachDB v1.0 beta                                      ║
║   2017: CockroachDB v1.0 GA                                        ║
║   2020: CockroachDB Serverless launched                             ║
║   2024: CockroachDB powers Netflix, DoorDash, Bose, and more      ║
║                                                                      ║
║   Design Goals:                                                      ║
║   ┌──────────────────────────────────────────────────────────────┐  ║
║   │  1. Survive ANYTHING — disk, node, rack, datacenter failure  │  ║
║   │  2. Scale HORIZONTALLY — add nodes, get more capacity        │  ║
║   │  3. PostgreSQL COMPATIBLE — use existing tools & ORMs       │  ║
║   │  4. SERIALIZABLE by default — strongest ACID guarantee       │  ║
║   │  5. ZERO data loss — no committed transaction is ever lost   │  ║
║   └──────────────────────────────────────────────────────────────┘  ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🏗️ Architecture Deep Dive

### The Big Picture

```
╔══════════════════════════════════════════════════════════════════════╗
║                 CockroachDB Architecture                             ║
║                                                                      ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │                     SQL LAYER                              │    ║
║   │  Parses SQL → Plans → Optimizes → Distributes execution   │    ║
║   │  (PostgreSQL wire protocol — pgwire)                      │    ║
║   └────────────────────────┬───────────────────────────────────┘    ║
║                            │                                        ║
║   ┌────────────────────────▼───────────────────────────────────┐    ║
║   │                 TRANSACTION LAYER (KV)                     │    ║
║   │  MVCC • Serializable Isolation • Distributed Transactions │    ║
║   │  Timestamp ordering • Write intents • Transaction records  │    ║
║   └────────────────────────┬───────────────────────────────────┘    ║
║                            │                                        ║
║   ┌────────────────────────▼───────────────────────────────────┐    ║
║   │                 DISTRIBUTION LAYER                         │    ║
║   │  Ranges • Raft Consensus • Leaseholders • Range Splits    │    ║
║   │  Load balancing • Replication • Zone configurations       │    ║
║   └────────────────────────┬───────────────────────────────────┘    ║
║                            │                                        ║
║   ┌────────────────────────▼───────────────────────────────────┐    ║
║   │                 STORAGE LAYER                              │    ║
║   │  Pebble (Go-native LSM engine, forked from RocksDB)      │    ║
║   │  Key-value pairs on disk • SSTables • Compaction          │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Layer 1: SQL Layer

```
Your App sends:  SELECT * FROM users WHERE id = 42;

┌──────────────────────────────────────────────────────────────┐
│                      SQL LAYER                                │
│                                                              │
│  Step 1: PARSE                                               │
│  ┌──────────────────────────────────────────────────┐       │
│  │  SQL string → Abstract Syntax Tree (AST)         │       │
│  │  "SELECT * FROM users WHERE id = 42"              │       │
│  │   → SelectNode { Table: users, Filter: id=42 }   │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  Step 2: PLAN                                                │
│  ┌──────────────────────────────────────────────────┐       │
│  │  Cost-based optimizer (same concepts as PG)       │       │
│  │  Should we do index scan? Full scan? Which index? │       │
│  │  Generated plan: IndexScan(users_pkey, id=42)     │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  Step 3: DISTRIBUTE                                          │
│  ┌──────────────────────────────────────────────────┐       │
│  │  "id=42 lives in Range 7, which is on Node 3"    │       │
│  │  Route the request to Node 3 (the leaseholder)   │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
└──────────────────────────────────────────────────────────────┘

💡 Key Insight: ANY node can accept ANY query.
   The SQL layer figures out WHERE the data lives and routes there.
   The client doesn't need to know the cluster topology.
```

### Layer 2: The Distribution Layer — Ranges & Raft

> This is the **heart** of CockroachDB. Understand this, and you understand everything.

```
╔══════════════════════════════════════════════════════════════════════╗
║                         RANGES                                       ║
║                                                                      ║
║   All data in CockroachDB is stored as sorted key-value pairs.      ║
║   These KV pairs are split into RANGES (default: 512 MB each).     ║
║                                                                      ║
║   Table: users                                                       ║
║   ┌─────────┬────────────┬─────────────────────────┐                ║
║   │ id (PK) │ name       │ KV representation        │                ║
║   ├─────────┼────────────┼─────────────────────────┤                ║
║   │ 1       │ Alice      │ /users/1 → {name:Alice}  │                ║
║   │ 2       │ Bob        │ /users/2 → {name:Bob}    │                ║
║   │ ...     │ ...        │ ...                      │                ║
║   │ 1000000 │ Zara       │ /users/1M → {name:Zara}  │                ║
║   └─────────┴────────────┴─────────────────────────┘                ║
║                                                                      ║
║   Split into Ranges by key order:                                    ║
║                                                                      ║
║   Range 1: /users/1 → /users/250000        (Node 1 = Leaseholder)  ║
║   Range 2: /users/250001 → /users/500000   (Node 3 = Leaseholder)  ║
║   Range 3: /users/500001 → /users/750000   (Node 2 = Leaseholder)  ║
║   Range 4: /users/750001 → /users/1000000  (Node 1 = Leaseholder)  ║
║                                                                      ║
║   Each Range is replicated 3x via Raft:                             ║
║   Range 1: [Node1(Leader), Node2(Follower), Node3(Follower)]        ║
║   Range 2: [Node3(Leader), Node1(Follower), Node2(Follower)]        ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### The Leaseholder — CockroachDB's Secret Weapon

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   In Raft, the LEADER handles writes.                               ║
║   In CockroachDB, there's also a LEASEHOLDER — handles reads.      ║
║                                                                      ║
║   ┌──────────┐   ┌──────────┐   ┌──────────┐                       ║
║   │  Node 1  │   │  Node 2  │   │  Node 3  │                       ║
║   │          │   │          │   │          │                       ║
║   │  LEADER  │   │ FOLLOWER │   │ FOLLOWER │                       ║
║   │    +     │   │          │   │          │                       ║
║   │ LEASE-   │   │          │   │          │                       ║
║   │ HOLDER   │   │          │   │          │                       ║
║   └──────────┘   └──────────┘   └──────────┘                       ║
║                                                                      ║
║   LEADER:      Coordinates writes (Raft consensus)                  ║
║   LEASEHOLDER: Serves reads WITHOUT Raft (fast! no consensus!)     ║
║                                                                      ║
║   Why separate them?                                                 ║
║   • Reads are usually 10-100x more frequent than writes             ║
║   • Reads from leaseholder don't need consensus → FAST             ║
║   • The leaseholder is ALWAYS up-to-date (it IS the leader)        ║
║   • In multi-region, leaseholder can be pinned to the user's       ║
║     region for low-latency reads!                                   ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Layer 3: Storage Layer — Pebble

```
┌──────────────────────────────────────────────────────────────┐
│                    PEBBLE STORAGE ENGINE                      │
│                                                              │
│  CockroachDB originally used RocksDB (C++ LSM tree engine)  │
│  In 2020, they built PEBBLE — a Go-native replacement       │
│                                                              │
│  Why switch?                                                 │
│  • RocksDB is C++ → CGo overhead in a Go application        │
│  • Pebble is pure Go → better integration, no FFI cost      │
│  • Custom optimizations for CockroachDB's access patterns   │
│                                                              │
│  How it works (LSM Tree):                                    │
│                                                              │
│  Writes:                                                     │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │ MemTable │ →  │ WAL (Write-  │ →  │ SSTable      │       │
│  │ (in RAM) │    │ Ahead Log)   │    │ (on disk)    │       │
│  └──────────┘    └──────────────┘    └──────────────┘       │
│                                                              │
│  Reads:                                                      │
│  Check MemTable → Check L0 SSTables → L1 → L2 → ... → Ln  │
│  (Uses bloom filters to skip SSTables that don't have key)  │
│                                                              │
│  Compaction:                                                 │
│  Background process merges SSTables to reduce read overhead  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Installation & Setup

### Option 1: Docker (Fastest for Learning)

```bash
# Single-node (for development)
docker run -d --name=crdb \
  -p 26257:26257 \
  -p 8080:8080 \
  cockroachdb/cockroach start-single-node --insecure

# Access the SQL shell
docker exec -it crdb cockroach sql --insecure

# Access the Admin UI → http://localhost:8080
```

### Option 2: 3-Node Local Cluster (For Real Testing)

```bash
# Start 3 nodes
cockroach start --insecure --store=node1 --listen-addr=localhost:26257 \
  --http-addr=localhost:8080 --join=localhost:26257,localhost:26258,localhost:26259 --background

cockroach start --insecure --store=node2 --listen-addr=localhost:26258 \
  --http-addr=localhost:8081 --join=localhost:26257,localhost:26258,localhost:26259 --background

cockroach start --insecure --store=node3 --listen-addr=localhost:26259 \
  --http-addr=localhost:8082 --join=localhost:26257,localhost:26258,localhost:26259 --background

# Initialize the cluster
cockroach init --insecure --host=localhost:26257

# Connect
cockroach sql --insecure --host=localhost:26257
```

### Option 3: CockroachDB Serverless (Cloud — Free Tier)

```
1. Go to cockroachlabs.com/cloud
2. Sign up for free
3. Create a "Serverless" cluster
4. Get your connection string
5. Connect with any PostgreSQL client:

   psql "postgresql://user:password@free-tier.gcp-us-central1.cockroachlabs.cloud:26257/defaultdb?sslmode=verify-full"
```

---

## 💻 SQL in CockroachDB — It's PostgreSQL!

### Creating Tables

```sql
-- CockroachDB uses PostgreSQL syntax
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email STRING UNIQUE NOT NULL,
    name STRING NOT NULL,
    age INT CHECK (age >= 0 AND age <= 150),
    city STRING,
    created_at TIMESTAMPTZ DEFAULT now(),
    
    -- CockroachDB-specific: define where data lives
    -- FAMILY groups columns that are often read together
    FAMILY f_core (id, email, name),
    FAMILY f_metadata (age, city, created_at)
);

-- Secondary index (same as PostgreSQL)
CREATE INDEX idx_users_city ON users (city);

-- Inverted index for JSONB (like GIN in PostgreSQL)
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING NOT NULL,
    attributes JSONB,
    INVERTED INDEX idx_attributes (attributes)
);
```

### CRUD Operations

```sql
-- INSERT (same as PostgreSQL)
INSERT INTO users (email, name, age, city) VALUES
    ('alice@example.com', 'Alice Johnson', 28, 'New York'),
    ('bob@example.com', 'Bob Smith', 35, 'San Francisco'),
    ('carol@example.com', 'Carol Williams', 42, 'London');

-- SELECT with JOINs (works exactly like PostgreSQL)
SELECT u.name, o.total, o.created_at
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.city = 'New York'
ORDER BY o.created_at DESC
LIMIT 10;

-- UPDATE with RETURNING (PostgreSQL feature — works!)
UPDATE users 
SET city = 'Los Angeles' 
WHERE email = 'alice@example.com'
RETURNING id, name, city;

-- UPSERT (CockroachDB's INSERT ... ON CONFLICT)
UPSERT INTO users (email, name, age, city)
VALUES ('alice@example.com', 'Alice Johnson', 29, 'Los Angeles');
-- If email exists → update; if not → insert

-- DELETE with subquery
DELETE FROM users 
WHERE id IN (
    SELECT id FROM users WHERE created_at < now() - INTERVAL '1 year'
);
```

### Distributed Transactions

```sql
-- This is the KILLER feature.
-- Multi-table, multi-range transaction — fully ACID, fully distributed.

BEGIN;

-- Deduct from Alice's account (might be on Node 1)
UPDATE accounts SET balance = balance - 100.00
WHERE user_id = (SELECT id FROM users WHERE email = 'alice@example.com');

-- Credit Bob's account (might be on Node 3)
UPDATE accounts SET balance = balance + 100.00
WHERE user_id = (SELECT id FROM users WHERE email = 'bob@example.com');

-- Create a transaction record (might be on Node 2)
INSERT INTO transactions (from_user, to_user, amount, type)
VALUES (
    (SELECT id FROM users WHERE email = 'alice@example.com'),
    (SELECT id FROM users WHERE email = 'bob@example.com'),
    100.00,
    'transfer'
);

COMMIT;

-- If ANY of these fail → ALL are rolled back.
-- Even across different nodes and different ranges.
-- This is what makes CockroachDB special.
```

### Window Functions, CTEs — All Work!

```sql
-- CTEs
WITH monthly_revenue AS (
    SELECT
        date_trunc('month', created_at) AS month,
        SUM(amount) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY 1
)
SELECT 
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month)) / 
        LAG(revenue) OVER (ORDER BY month) * 100, 
        2
    ) AS growth_pct
FROM monthly_revenue
ORDER BY month;

-- Window Functions
SELECT 
    name,
    city,
    age,
    ROW_NUMBER() OVER (PARTITION BY city ORDER BY age DESC) AS rank_in_city,
    AVG(age) OVER (PARTITION BY city) AS avg_age_in_city
FROM users;
```

---

## 🌍 Multi-Region — CockroachDB's Crown Jewel

### The Problem It Solves

```
╔══════════════════════════════════════════════════════════════════════╗
║   You have users in US, Europe, and Asia.                            ║
║   You need:                                                          ║
║   1. Low-latency reads for ALL users (< 10ms)                       ║
║   2. Data survives if an ENTIRE region goes down                    ║
║   3. Strong consistency (no stale reads)                            ║
║                                                                      ║
║   Traditional approach:                                              ║
║   ┌──────────┐        ┌──────────┐        ┌──────────┐             ║
║   │  US DB   │ ──────►│  EU DB   │ ──────►│ Asia DB  │             ║
║   │ (primary)│  async │ (replica)│  async │ (replica)│             ║
║   └──────────┘  lag!  └──────────┘  lag!  └──────────┘             ║
║                                                                      ║
║   Problems: Stale reads, complex failover, data loss risk           ║
║                                                                      ║
║   CockroachDB approach:                                              ║
║   ┌──────────┐        ┌──────────┐        ┌──────────┐             ║
║   │  US      │ ◄─────►│  EU      │ ◄─────►│  Asia    │             ║
║   │  Nodes   │  Raft  │  Nodes   │  Raft  │  Nodes   │             ║
║   └──────────┘ sync!  └──────────┘ sync!  └──────────┘             ║
║                                                                      ║
║   Every node is equal. Raft keeps all copies consistent.            ║
║   Leaseholders pinned to users' regions for fast reads.             ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Multi-Region Topologies

```sql
-- Step 1: Add regions to the database
ALTER DATABASE myapp PRIMARY REGION "us-east1";
ALTER DATABASE myapp ADD REGION "eu-west1";
ALTER DATABASE myapp ADD REGION "ap-southeast1";

-- Topology 1: REGIONAL TABLES (most common)
-- Data lives in the region where it's most accessed
ALTER TABLE users SET LOCALITY REGIONAL BY ROW;

-- Each row gets a hidden `crdb_region` column
INSERT INTO users (crdb_region, email, name, city) VALUES
    ('us-east1', 'alice@example.com', 'Alice', 'New York'),
    ('eu-west1', 'hans@example.com', 'Hans', 'Berlin'),
    ('ap-southeast1', 'yuki@example.com', 'Yuki', 'Tokyo');

-- Alice's data → leaseholder in US (fast reads for US users)
-- Hans's data → leaseholder in EU (fast reads for EU users)
-- Yuki's data → leaseholder in Asia (fast reads for Asia users)
```

```sql
-- Topology 2: GLOBAL TABLES
-- Data is read everywhere, rarely written (e.g., config, currencies)
ALTER TABLE currencies SET LOCALITY GLOBAL;

-- Reads from ANY region are fast (served locally)
-- Writes are slower (must replicate to all regions)
-- Perfect for: reference data, feature flags, currency rates
```

```sql
-- Topology 3: REGIONAL BY TABLE
-- Entire table pinned to one region
ALTER TABLE us_audit_logs SET LOCALITY REGIONAL IN "us-east1";
-- All data stays in US region — great for compliance!
```

### Multi-Region Performance Characteristics

```
╔══════════════════════════════════════════════════════════════════════╗
║              LATENCY BY TOPOLOGY                                     ║
║                                                                      ║
║   ┌───────────────────┬────────────────┬────────────────┐           ║
║   │ Topology          │ Read Latency   │ Write Latency  │           ║
║   ├───────────────────┼────────────────┼────────────────┤           ║
║   │ REGIONAL BY ROW   │ 1-5 ms (local) │ ~50ms (Raft    │           ║
║   │ (user in correct  │                │ across regions)│           ║
║   │  region)          │                │                │           ║
║   ├───────────────────┼────────────────┼────────────────┤           ║
║   │ GLOBAL            │ 1-5 ms (local) │ 100-300 ms     │           ║
║   │ (read-heavy)      │ EVERYWHERE!    │ (all regions)  │           ║
║   ├───────────────────┼────────────────┼────────────────┤           ║
║   │ REGIONAL BY TABLE │ 1-5 ms (same   │ 5-15 ms (same  │           ║
║   │ (pinned to one    │ region only)   │ region only)   │           ║
║   │  region)          │                │                │           ║
║   └───────────────────┴────────────────┴────────────────┘           ║
║                                                                      ║
║   💡 The trick: Pin LEASEHOLDERS to where users ARE.                ║
║      US users read US leaseholders = local speed.                   ║
║      EU users read EU leaseholders = local speed.                   ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🔒 Serializable Isolation — The Strongest Guarantee

> CockroachDB defaults to **Serializable** isolation — the STRONGEST level. Most databases default to Read Committed (weaker).

### What Does Serializable Mean?

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   SERIALIZABLE means: The result of concurrent transactions        ║
║   is the SAME as if they ran ONE AT A TIME, in some serial order.  ║
║                                                                      ║
║   Example: Two users buy the last item simultaneously              ║
║                                                                      ║
║   ┌──────────────────┐     ┌──────────────────┐                    ║
║   │  Transaction A   │     │  Transaction B   │                    ║
║   │  (Alice)         │     │  (Bob)           │                    ║
║   ├──────────────────┤     ├──────────────────┤                    ║
║   │ SELECT stock     │     │ SELECT stock     │                    ║
║   │ → stock = 1      │     │ → stock = 1      │                    ║
║   │                  │     │                  │                    ║
║   │ UPDATE stock = 0 │     │ UPDATE stock = 0 │                    ║
║   │ INSERT order     │     │ INSERT order     │                    ║
║   │ COMMIT ✅        │     │ COMMIT ❌ RETRY! │                    ║
║   └──────────────────┘     └──────────────────┘                    ║
║                                                                      ║
║   With READ COMMITTED (PostgreSQL default):                         ║
║   Both might succeed → OVERSOLD! 💀                                ║
║                                                                      ║
║   With SERIALIZABLE (CockroachDB default):                         ║
║   One succeeds, one gets a RETRY error → safe! ✅                  ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Handling Transaction Retries

```python
# In your application code — you MUST handle retries!
# CockroachDB may ask you to retry a transaction 
# (this is the cost of serializable isolation)

import psycopg2
import time

def run_transaction(conn, callback, max_retries=3):
    """Execute a transaction with automatic retry on serialization errors."""
    for attempt in range(max_retries):
        try:
            with conn.cursor() as cur:
                callback(cur)
                conn.commit()
                return  # Success!
        except psycopg2.errors.SerializationFailure:
            conn.rollback()
            # Exponential backoff
            sleep_time = (2 ** attempt) * 0.1  # 0.1s, 0.2s, 0.4s
            time.sleep(sleep_time)
            continue
    raise Exception("Transaction failed after max retries")

# Usage
def transfer_money(cur):
    cur.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
    cur.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2")

conn = psycopg2.connect("postgresql://root@localhost:26257/bank?sslmode=disable")
run_transaction(conn, transfer_money)
```

> 💡 **Pro Tip:** Every CockroachDB ORM driver has built-in retry logic. Use the official drivers (Go: `pgx`, Python: `sqlalchemy-cockroachdb`, Java: Hibernate with retry interceptor).

---

## 📊 Performance Tuning

### Reading Execution Plans

```sql
-- EXPLAIN (just the plan)
EXPLAIN SELECT * FROM users WHERE city = 'New York';

-- EXPLAIN ANALYZE (runs the query and shows actual timing)
EXPLAIN ANALYZE SELECT * FROM users WHERE city = 'New York';

-- Full distributed execution plan
EXPLAIN (DISTSQL) SELECT u.name, COUNT(o.id)
FROM users u JOIN orders o ON u.id = o.user_id
GROUP BY u.name;
```

```
Output example:
                                        info
─────────────────────────────────────────────────────────────
  distribution: full
  vectorized: true

  • group (hash)
  │ group by: name
  │ ordered: +name
  │
  └── • hash join
      │ equality: (id) = (user_id)
      │
      ├── • scan
      │     table: users@users_pkey
      │     spans: FULL SCAN
      │
      └── • scan
            table: orders@orders_pkey
            spans: FULL SCAN

💡 See "FULL SCAN"? That's a problem. Add an index!
```

### Avoiding Hot Spots

```
╔══════════════════════════════════════════════════════════════════════╗
║                    THE HOT SPOT PROBLEM                              ║
║                                                                      ║
║   ❌ BAD: Sequential primary keys (AUTO_INCREMENT / SERIAL)         ║
║                                                                      ║
║   CREATE TABLE orders (                                              ║
║       id SERIAL PRIMARY KEY,  -- 1, 2, 3, 4, 5...                   ║
║       ...                                                            ║
║   );                                                                 ║
║                                                                      ║
║   Problem: ALL new inserts go to the SAME range (the last one)      ║
║   ┌──────────┐  ┌──────────┐  ┌──────────┐                         ║
║   │ Range 1  │  │ Range 2  │  │ Range 3  │  ← ALL writes here! 🔥  ║
║   │ 1-100K   │  │ 100K-200K│  │ 200K+    │                         ║
║   │ idle     │  │ idle     │  │ OVERLOAD │                         ║
║   └──────────┘  └──────────┘  └──────────┘                         ║
║                                                                      ║
║   ✅ GOOD: UUID primary keys (CockroachDB best practice)            ║
║                                                                      ║
║   CREATE TABLE orders (                                              ║
║       id UUID PRIMARY KEY DEFAULT gen_random_uuid(),                 ║
║       ...                                                            ║
║   );                                                                 ║
║                                                                      ║
║   UUIDs are random → inserts spread across ALL ranges evenly        ║
║   ┌──────────┐  ┌──────────┐  ┌──────────┐                         ║
║   │ Range 1  │  │ Range 2  │  │ Range 3  │                         ║
║   │ writes ✓ │  │ writes ✓ │  │ writes ✓ │  ← Even distribution!   ║
║   └──────────┘  └──────────┘  └──────────┘                         ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

```sql
-- If you MUST use sequential IDs, use HASH-SHARDED INDEX
CREATE TABLE events (
    id INT PRIMARY KEY USING HASH WITH (bucket_count = 8),
    event_type STRING,
    data JSONB,
    created_at TIMESTAMPTZ DEFAULT now()
);
-- CockroachDB prepends a hash bucket to the key → spreads inserts!
```

### Key Performance Settings

```sql
-- Check cluster settings
SHOW ALL CLUSTER SETTINGS;

-- Important tuning knobs
SET CLUSTER SETTING kv.range_merge.queue_enabled = true;   -- auto-merge small ranges
SET CLUSTER SETTING kv.snapshot_rebalance.max_rate = '64MB'; -- faster rebalancing

-- Table-level tuning
ALTER TABLE users CONFIGURE ZONE USING
    num_replicas = 5,            -- More replicas for critical data
    gc.ttlseconds = 86400;      -- Keep MVCC versions for 24 hours (time travel!)
    
-- Check range distribution
SHOW RANGES FROM TABLE users;

-- Check for hot ranges
SELECT range_id, start_key, end_key, 
       lease_holder, replicas, 
       queries_per_second
FROM [SHOW RANGES FROM TABLE users WITH DETAILS];
```

---

## 🛡️ Survivability — Kill Anything, It Keeps Running

### The Cockroach Promise

```
╔══════════════════════════════════════════════════════════════════════╗
║                WHAT CAN COCKROACHDB SURVIVE?                         ║
║                                                                      ║
║   Failure Type              │ Impact  │ Recovery Time               ║
║   ──────────────────────────┼─────────┼────────────────             ║
║   Single disk failure       │ NONE    │ Instant                     ║
║   Single node failure       │ NONE    │ Instant (Raft failover)     ║
║   Entire rack failure       │ NONE    │ ~1-2 seconds                ║
║   Entire datacenter failure │ NONE*   │ ~5-10 seconds               ║
║   Network partition         │ PARTIAL │ Available side keeps going  ║
║                                                                      ║
║   * With 3+ regions configured and survival goal = REGION           ║
║                                                                      ║
║   ┌──────────────────────────────────────────────────────────┐      ║
║   │  3-node cluster: survives 1 node failure                 │      ║
║   │  5-node cluster: survives 2 node failures                │      ║
║   │  3-region cluster: survives 1 entire region failure      │      ║
║   │                                                          │      ║
║   │  Formula: survives ⌊(n-1)/2⌋ failures out of n replicas │      ║
║   └──────────────────────────────────────────────────────────┘      ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Testing Survivability (Chaos Engineering)

```bash
# Kill a node — watch the cluster survive
docker stop crdb-node2

# Check cluster health
cockroach node status --insecure --host=localhost:26257

# The cluster continues serving ALL queries!
# Raft automatically elected new leaders for affected ranges.

# Bring the node back
docker start crdb-node2

# CockroachDB automatically replicates any missed data
# No manual intervention needed!
```

---

## 🔧 Admin UI & Monitoring

### Built-in Admin Dashboard

```
Navigate to: http://localhost:8080

┌──────────────────────────────────────────────────────────────┐
│                CockroachDB Admin UI                           │
│                                                              │
│  📊 Overview                                                 │
│  ├── Cluster health (nodes up/down)                          │
│  ├── QPS (queries per second)                                │
│  ├── Latency (P50, P90, P99)                                │
│  └── Storage used                                            │
│                                                              │
│  📈 Metrics                                                  │
│  ├── SQL Statements (slow queries, errors)                   │
│  ├── Replication (under-replicated ranges)                   │
│  ├── Storage (compaction, disk usage)                         │
│  └── Network (inter-node traffic)                            │
│                                                              │
│  🔍 Statements                                               │
│  ├── Top queries by execution time                           │
│  ├── Execution plans                                         │
│  └── Contention analysis                                     │
│                                                              │
│  🗺️ Network                                                  │
│  └── Cluster topology visualization                          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Monitoring Queries

```sql
-- Check node status
SELECT node_id, address, is_live, ranges, leases
FROM crdb_internal.gossip_nodes;

-- Find slow queries
SELECT query, calls, mean_service_lat, max_service_lat
FROM crdb_internal.node_statement_statistics
WHERE mean_service_lat > '100ms'::INTERVAL
ORDER BY mean_service_lat DESC
LIMIT 10;

-- Check range distribution per table
SELECT table_name, range_count, approximate_disk_size
FROM [SHOW RANGES FROM DATABASE myapp]
GROUP BY table_name;

-- Find contention (lock conflicts)
SELECT * FROM crdb_internal.cluster_contention_events
ORDER BY count DESC LIMIT 10;
```

---

## 📋 CockroachDB vs PostgreSQL — What's Different?

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   ✅ What WORKS exactly like PostgreSQL:                             ║
║   • SQL syntax (SELECT, INSERT, UPDATE, DELETE, JOINs)              ║
║   • Data types (INT, STRING, JSONB, UUID, TIMESTAMP, ARRAY)         ║
║   • Indexes (B-tree, GIN/Inverted, Partial, Covering)              ║
║   • Window functions, CTEs, Subqueries                              ║
║   • pgwire protocol (use psql, pgAdmin, any PG driver)             ║
║   • ORMs (SQLAlchemy, Hibernate, Prisma, ActiveRecord)             ║
║                                                                      ║
║   ❌ What's DIFFERENT:                                               ║
║   • No LISTEN/NOTIFY (pub/sub) — use changefeeds instead           ║
║   • No custom extensions (PostGIS, pg_cron) — built-in spatial only║
║   • No TEMP tables across transactions                              ║
║   • SERIAL → use UUID (to avoid hot spots)                          ║
║   • Default isolation = Serializable (PG default = Read Committed)  ║
║   • Transaction retries needed (serializable trade-off)             ║
║   • No stored procedures (functions only, limited PL/pgSQL)        ║
║                                                                      ║
║   🆕 What CockroachDB ADDS (not in PostgreSQL):                     ║
║   • Multi-region locality (REGIONAL BY ROW, GLOBAL)                 ║
║   • Changefeeds (CDC — stream changes to Kafka, etc.)               ║
║   • Hash-sharded indexes (prevent hot spots)                        ║
║   • Built-in Admin UI with metrics                                  ║
║   • Automatic range splitting and rebalancing                       ║
║   • Follower reads (read from closest replica)                      ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🔄 Change Data Capture (Changefeeds)

> Stream every change in your database to Kafka, cloud storage, or webhooks — in real time.

```sql
-- Stream all changes to a Kafka topic
CREATE CHANGEFEED FOR TABLE orders, users
INTO 'kafka://broker:9092'
WITH updated, resolved = '10s';

-- Stream to a webhook endpoint
CREATE CHANGEFEED FOR TABLE orders
INTO 'webhook-https://api.myapp.com/events'
WITH updated;

-- Stream to cloud storage (for data lake)
CREATE CHANGEFEED FOR TABLE orders
INTO 's3://my-bucket/cdc/?AWS_ACCESS_KEY_ID=xxx&AWS_SECRET_ACCESS_KEY=xxx'
WITH updated, resolved = '1m';
```

```json
// Each change event looks like:
{
  "key": "[\"8f2e8c1a-...\"]",
  "value": {
    "after": {
      "id": "8f2e8c1a-...",
      "user_id": "alice",
      "amount": 99.99,
      "status": "shipped"
    },
    "updated": "1623456789.000000000"
  }
}
```

> 💡 **Real-World Use:** Netflix uses CockroachDB changefeeds to keep their search indexes and caches in sync with the source of truth.

---

## 🧠 Interview Quick-Fire — CockroachDB

| Question | Perfect Answer |
|----------|----------------|
| What is CockroachDB? | A distributed SQL database that's PostgreSQL wire-compatible, provides serializable ACID transactions, and survives any infrastructure failure. |
| How does it distribute data? | Data is split into 512 MB ranges. Each range is replicated via Raft across 3+ nodes. Ranges auto-split and auto-rebalance. |
| What is a leaseholder? | The node that holds the "lease" for a range — it serves reads without consensus (fast) and is also typically the Raft leader for writes. |
| Why use UUID instead of SERIAL? | SERIAL creates sequential keys → all inserts go to the last range (hot spot). UUID distributes inserts randomly across all ranges. |
| What isolation level does it use? | Serializable by default — the strongest ACID guarantee. This means concurrent transactions behave as if they ran serially. |
| How does multi-region work? | Data is annotated with regions. Leaseholders are pinned to the region closest to users. Raft still replicates across regions for durability. |
| What's the trade-off? | Higher per-query latency (Raft consensus adds ~5-15ms) in exchange for horizontal scale, zero downtime, and global consistency. |

---

## 📚 Chapter Summary

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. CockroachDB = open-source distributed SQL, inspired by Spanner  ║
║  2. PostgreSQL-compatible — use psql, ORMs, existing tools          ║
║  3. Data splits into Ranges → replicated via Raft → auto-balanced  ║
║  4. Serializable isolation by default — strongest ACID              ║
║  5. Multi-region with pinned leaseholders = local-speed reads       ║
║  6. Changefeeds for real-time CDC to Kafka/S3/webhooks              ║
║  7. Use UUIDs (not SERIAL) to avoid hot spots                       ║
║  8. Handle transaction retries in your application code             ║
║  9. Best for: Global apps, fintech, anywhere you need SQL at scale  ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

> **Next Chapter:** [TiDB — MySQL-Compatible Distributed DB →](./03-TiDB.md)
