# 🔵 Chapter 4.5 — YugabyteDB — Open Source Distributed SQL

> **Level:** 🟡 Intermediate
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 4.1 (NewSQL Overview), PostgreSQL or Cassandra basics

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand YugabyteDB's **unique dual-API architecture** (PostgreSQL + Cassandra)
- Know how **DocDB** (the storage engine) works and why it's different
- Write real SQL using **YSQL** (PostgreSQL-compatible) and **YCQL** (Cassandra-compatible)
- Design **geo-distributed** schemas with tablespaces and geo-partitioning
- Tune YugabyteDB for **performance** — tablets, partitioning, colocation
- Know when YugabyteDB is the **right choice** vs CockroachDB, TiDB, or Spanner

---

## 🧠 The Origin Story

### From Facebook's Infrastructure to Open Source

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   The founders (Kannan Muthukkaruppan, Karthik Ranganathan,         ║
║   Mikhail Bautin) were engineers at Facebook who built:             ║
║                                                                      ║
║   • Apache HBase — Facebook's Messages backend                      ║
║   • Apache Cassandra — Facebook's Inbox Search                       ║
║                                                                      ║
║   They knew distributed NoSQL inside-out.                           ║
║   But they also saw developers STRUGGLING without SQL, ACID, JOINs. ║
║                                                                      ║
║   Their mission:                                                     ║
║   "Build a database that gives you PostgreSQL on top AND             ║
║    Cassandra on top, with Google Spanner's distributed engine       ║
║    underneath. And make it 100% open source."                       ║
║                                                                      ║
║   2016: Yugabyte founded                                            ║
║   2019: YugabyteDB 2.0 — production-ready                           ║
║   2020: 100% open source (Apache 2.0)                               ║
║   2024: YugabyteDB Managed & Aeon (serverless)                       ║
║                                                                      ║
║   The Name: "Yuga" = an era/epoch in Hindu cosmology                ║
║             "Byte" = fundamental unit of data                        ║
║             Together: "A new era of data"                           ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🏗️ Architecture — The Dual-API Design

### The Big Picture

```
╔══════════════════════════════════════════════════════════════════════╗
║                    YugabyteDB ARCHITECTURE                           ║
║                                                                      ║
║   Your Application                                                   ║
║   ├── PostgreSQL driver (JDBC, psycopg2, etc.)                      ║
║   └── Cassandra driver (DataStax, etc.)                             ║
║        │                              │                              ║
║        ▼                              ▼                              ║
║   ┌──────────────┐          ┌──────────────┐                        ║
║   │    YSQL      │          │    YCQL      │                        ║
║   │ PostgreSQL   │          │  Cassandra   │                        ║
║   │ Compatible   │          │  Compatible  │                        ║
║   │ (Port 5433)  │          │  (Port 9042) │                        ║
║   │              │          │              │                        ║
║   │ Full SQL!    │          │ CQL syntax!  │                        ║
║   │ JOINs ✅     │          │ No JOINs     │                        ║
║   │ Transactions │          │ Lightweight  │                        ║
║   │ Foreign Keys │          │ transactions │                        ║
║   └──────┬───────┘          └──────┬───────┘                        ║
║          │                         │                                 ║
║          └────────────┬────────────┘                                 ║
║                       │                                              ║
║          ┌────────────▼─────────────────────────────────┐           ║
║          │              DocDB                            │           ║
║          │     (Distributed Document Store)               │           ║
║          │                                               │           ║
║          │  ┌─────────────────────────────────────────┐ │           ║
║          │  │         Raft Consensus Layer             │ │           ║
║          │  │  (Per-tablet Raft groups)                │ │           ║
║          │  └─────────────────────────────────────────┘ │           ║
║          │                                               │           ║
║          │  ┌─────────────────────────────────────────┐ │           ║
║          │  │         RocksDB Storage                  │ │           ║
║          │  │  (Per-tablet RocksDB instance)           │ │           ║
║          │  └─────────────────────────────────────────┘ │           ║
║          │                                               │           ║
║          └───────────────────────────────────────────────┘           ║
║                                                                      ║
║   ┌──────────────────────────────────────────────────────┐          ║
║   │              YB-Master Service                        │          ║
║   │  • Tablet metadata & placement                       │          ║
║   │  • Schema changes (DDL)                              │          ║
║   │  • Load balancing decisions                          │          ║
║   │  • Runs on odd number of nodes (3, 5, 7)             │          ║
║   └──────────────────────────────────────────────────────┘          ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Why TWO APIs?

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   YSQL (PostgreSQL API — Port 5433)                                 ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │  • Full PostgreSQL wire protocol                           │    ║
║   │  • JOINs, CTEs, Window Functions, Subqueries              │    ║
║   │  • Distributed ACID transactions                          │    ║
║   │  • Foreign keys, triggers, stored procedures              │    ║
║   │  • PostgreSQL extensions (PostGIS, pg_stat_statements)    │    ║
║   │                                                            │    ║
║   │  USE WHEN: You need relational features, complex queries, │    ║
║   │  or are migrating from PostgreSQL.                        │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║   YCQL (Cassandra API — Port 9042)                                  ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │  • CQL (Cassandra Query Language) compatible              │    ║
║   │  • Partition key + clustering key model                   │    ║
║   │  • Lightweight transactions                               │    ║
║   │  • Collections (LIST, SET, MAP)                           │    ║
║   │  • TTL (auto-expire data)                                 │    ║
║   │                                                            │    ║
║   │  USE WHEN: You need ultra-fast single-key lookups,        │    ║
║   │  Cassandra-style data modeling, or are migrating from     │    ║
║   │  Cassandra.                                               │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║   💡 KEY INSIGHT: Both APIs share the SAME storage engine (DocDB)   ║
║   You can even have YSQL tables and YCQL tables in the same cluster!║
║   Use the right API for each microservice's needs.                  ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### DocDB — The Storage Engine

```
╔══════════════════════════════════════════════════════════════════════╗
║                         DocDB                                        ║
║                                                                      ║
║   DocDB = "Distributed Document Store"                              ║
║   Inspired by Google Spanner + Apache HBase                         ║
║                                                                      ║
║   Data organization:                                                 ║
║                                                                      ║
║   Table → split into TABLETS (default: ~10 per tserver)             ║
║                                                                      ║
║   ┌── Table: users ─────────────────────────────────────────────┐   ║
║   │                                                              │   ║
║   │  ┌── Tablet 1 ──┐  ┌── Tablet 2 ──┐  ┌── Tablet 3 ──┐    │   ║
║   │  │  Hash: 0-5460│  │ Hash: 5461-  │  │ Hash: 10923- │    │   ║
║   │  │              │  │        10922 │  │        16383 │    │   ║
║   │  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │    │   ║
║   │  │  │RocksDB │  │  │  │RocksDB │  │  │  │RocksDB │  │    │   ║
║   │  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │    │   ║
║   │  │              │  │              │  │              │    │   ║
║   │  │  Raft Group: │  │  Raft Group: │  │  Raft Group: │    │   ║
║   │  │  L: Node1    │  │  L: Node2    │  │  L: Node3    │    │   ║
║   │  │  F: Node2    │  │  F: Node3    │  │  F: Node1    │    │   ║
║   │  │  F: Node3    │  │  F: Node1    │  │  F: Node2    │    │   ║
║   │  └──────────────┘  └──────────────┘  └──────────────┘    │   ║
║   │                                                              │   ║
║   └──────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║   Key differences from CockroachDB:                                 ║
║   • YugabyteDB: HASH-based sharding by default (consistent hash)  ║
║     → Even distribution, but range scans need to hit all tablets   ║
║   • CockroachDB: RANGE-based sharding by default                   ║
║     → Range scans are efficient, but sequential writes can hot-spot║
║                                                                      ║
║   YugabyteDB also supports RANGE sharding when needed:             ║
║     CREATE TABLE events (...) SPLIT AT VALUES ((100), (200));      ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### YB-TServer vs YB-Master

```
┌──────────────────────────────────────────────────────────────────┐
│                    NODE ARCHITECTURE                               │
│                                                                  │
│  Each YugabyteDB node runs TWO processes:                        │
│                                                                  │
│  ┌─── YB-TServer (Tablet Server) ──────────────────────────┐   │
│  │                                                          │   │
│  │  • Hosts tablets (data)                                  │   │
│  │  • Serves YSQL and YCQL queries                         │   │
│  │  • Participates in Raft for each tablet                  │   │
│  │  • Manages RocksDB instances                             │   │
│  │                                                          │   │
│  │  Ports:                                                  │   │
│  │  5433 → YSQL (PostgreSQL)                               │   │
│  │  9042 → YCQL (Cassandra)                                │   │
│  │  9000 → TServer admin UI                                │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─── YB-Master (Master Server) ───────────────────────────┐   │
│  │                                                          │   │
│  │  • Stores metadata (table schemas, tablet locations)     │   │
│  │  • Coordinates DDL operations                            │   │
│  │  • Handles tablet splitting and load balancing           │   │
│  │  • Runs on 3 or 5 nodes (Raft for HA)                    │   │
│  │                                                          │   │
│  │  Port: 7000 → Master admin UI                           │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Installation & Setup

### Option 1: Docker (Quick Start)

```bash
# Single node (development)
docker run -d --name yugabyte \
  -p 5433:5433 \
  -p 9042:9042 \
  -p 7000:7000 \
  -p 9000:9000 \
  -p 15433:15433 \
  yugabytedb/yugabyte:latest \
  bin/yugabyted start --daemon=false

# Access YSQL (PostgreSQL)
docker exec -it yugabyte bin/ysqlsh

# Access YCQL (Cassandra)  
docker exec -it yugabyte bin/ycqlsh

# Admin UI → http://localhost:15433
```

### Option 2: 3-Node Local Cluster

```bash
# Download YugabyteDB
curl -sSL https://downloads.yugabyte.com/releases/latest/yugabyte-linux-x86_64.tar.gz | tar xz

cd yugabyte-*/

# Start 3-node cluster
bin/yugabyted start --base_dir=/tmp/yb1 --advertise_address=127.0.0.1
bin/yugabyted start --base_dir=/tmp/yb2 --advertise_address=127.0.0.2 --join=127.0.0.1
bin/yugabyted start --base_dir=/tmp/yb3 --advertise_address=127.0.0.3 --join=127.0.0.1

# Connect
bin/ysqlsh -h 127.0.0.1

# Check cluster status
bin/yugabyted status --base_dir=/tmp/yb1
```

### Option 3: YugabyteDB Managed (Cloud)

```
1. Go to cloud.yugabyte.com
2. Create a free "Sandbox" cluster
3. Choose cloud provider (AWS, GCP, Azure)
4. Get connection string
5. Connect:

   psql "postgresql://admin:password@us-east-1.aws.yugabyte.cloud:5433/yugabyte?ssl=true"
```

---

## 💻 YSQL — The PostgreSQL-Compatible API

### Creating Tables

```sql
-- YSQL is PostgreSQL! Use psql, pgAdmin, any PG tool.

-- Hash-sharded table (default — best for distributed writes)
CREATE TABLE users (
    id UUID DEFAULT gen_random_uuid(),
    email TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    age INT CHECK (age >= 0),
    city TEXT,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (id HASH)  -- HASH = distributed evenly
);

-- Range-sharded table (best for range scans)
CREATE TABLE events (
    event_date DATE,
    event_id UUID DEFAULT gen_random_uuid(),
    event_type TEXT,
    payload JSONB,
    PRIMARY KEY (event_date ASC, event_id)  -- ASC = range-sharded
);

-- Colocated tables (small tables on same tablet — reduces overhead)
CREATE DATABASE myapp WITH COLOCATION = true;

-- In a colocated database, small tables share one tablet:
CREATE TABLE countries (
    code TEXT PRIMARY KEY,
    name TEXT
);  -- Small lookup table → shares tablet with other colocated tables

CREATE TABLE currencies (
    code TEXT PRIMARY KEY,
    name TEXT,
    symbol TEXT
);  -- Also colocated → same tablet → fast JOINs!

-- Large table opts OUT of colocation:
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    amount DECIMAL
) WITH (COLOCATION = false);  -- Gets its own tablets for scale
```

### YugabyteDB-Specific Table Options

```sql
-- Control tablet splitting
CREATE TABLE logs (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    level TEXT,
    message TEXT,
    ts TIMESTAMPTZ DEFAULT now()
) SPLIT INTO 12 TABLETS;
-- Pre-split into 12 tablets for write parallelism

-- Range-split with explicit boundaries
CREATE TABLE metrics (
    region TEXT,
    ts TIMESTAMPTZ,
    value DOUBLE PRECISION,
    PRIMARY KEY (region, ts ASC)
) SPLIT AT VALUES (('eu'), ('us'));
-- 3 tablets: [min, 'eu'), ['eu', 'us'), ['us', max)

-- Tablespace-based geo-distribution
CREATE TABLESPACE us_east_tablespace WITH (
    replica_placement = '{"num_replicas": 3, "placement_blocks": [
        {"cloud": "aws", "region": "us-east-1", "zone": "us-east-1a", "min_num_replicas": 1},
        {"cloud": "aws", "region": "us-east-1", "zone": "us-east-1b", "min_num_replicas": 1},
        {"cloud": "aws", "region": "us-east-1", "zone": "us-east-1c", "min_num_replicas": 1}
    ]}'
);

CREATE TABLE us_customers (
    id UUID PRIMARY KEY,
    name TEXT,
    email TEXT
) TABLESPACE us_east_tablespace;
-- Data stays in US East — compliance + low latency for US users
```

### CRUD and Advanced Queries

```sql
-- INSERT (same as PostgreSQL)
INSERT INTO users (email, name, age, city) VALUES
    ('alice@example.com', 'Alice Johnson', 28, 'New York'),
    ('bob@example.com', 'Bob Smith', 35, 'London'),
    ('carol@example.com', 'Carol Chen', 42, 'Tokyo');

-- Distributed JOIN (works across tablets!)
SELECT u.name, u.city, COUNT(o.id) AS order_count, SUM(o.amount) AS total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.name, u.city
ORDER BY total DESC NULLS LAST;

-- Window functions (full PostgreSQL support!)
SELECT 
    name,
    city,
    age,
    RANK() OVER (PARTITION BY city ORDER BY age DESC) AS age_rank,
    PERCENT_RANK() OVER (ORDER BY age) AS percentile
FROM users;

-- CTEs with recursion
WITH RECURSIVE subordinates AS (
    SELECT id, name, manager_id, 1 AS level
    FROM employees WHERE id = 1  -- CEO
    UNION ALL
    SELECT e.id, e.name, e.manager_id, s.level + 1
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates ORDER BY level, name;

-- JSONB operations (full PostgreSQL JSONB!)
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT,
    attributes JSONB
);

INSERT INTO products (name, attributes) VALUES
    ('Laptop', '{"brand": "Dell", "specs": {"ram": 16, "storage": "512GB"}}');

SELECT name, attributes->>'brand' AS brand,
       attributes->'specs'->>'ram' AS ram_gb
FROM products
WHERE attributes @> '{"brand": "Dell"}';

-- Distributed ACID transaction
BEGIN;
    UPDATE accounts SET balance = balance - 500 WHERE id = 'alice';
    UPDATE accounts SET balance = balance + 500 WHERE id = 'bob';
    
    INSERT INTO transfer_log (from_acct, to_acct, amount, ts)
    VALUES ('alice', 'bob', 500, now());
COMMIT;
-- All 3 operations succeed or ALL fail — even across tablets!
```

---

## 💻 YCQL — The Cassandra-Compatible API

```sql
-- Connect with Cassandra tools (cqlsh, DataStax drivers)
-- ycqlsh or any CQL-compatible client

-- Create keyspace
CREATE KEYSPACE ecommerce WITH replication = {
    'class': 'SimpleStrategy', 
    'replication_factor': 3
};

USE ecommerce;

-- Create table with partition + clustering keys
CREATE TABLE user_activity (
    user_id UUID,
    activity_date DATE,
    activity_ts TIMESTAMP,
    activity_type TEXT,
    details TEXT,
    PRIMARY KEY ((user_id), activity_date, activity_ts)
) WITH CLUSTERING ORDER BY (activity_date DESC, activity_ts DESC);

-- INSERT with TTL (auto-expire after 90 days!)
INSERT INTO user_activity (user_id, activity_date, activity_ts, activity_type, details)
VALUES (uuid(), '2024-01-15', toTimestamp(now()), 'login', 'from mobile app')
USING TTL 7776000;  -- 90 days in seconds

-- Fast single-partition query
SELECT * FROM user_activity 
WHERE user_id = 550e8400-e29b-41d4-a716-446655440000
AND activity_date >= '2024-01-01'
LIMIT 20;

-- YCQL adds ACID that Cassandra doesn't have!
-- Multi-row transaction (not possible in real Cassandra)
BEGIN TRANSACTION;
    UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 'laptop-001';
    INSERT INTO orders (order_id, product_id, user_id) 
    VALUES (uuid(), 'laptop-001', 'user-42');
END TRANSACTION;

-- Indexes (secondary index)
CREATE INDEX ON user_activity (activity_type);

-- JSON support in YCQL
CREATE TABLE events (
    id UUID PRIMARY KEY,
    data JSONB
);

INSERT INTO events (id, data) VALUES 
    (uuid(), '{"type": "click", "page": "/products", "duration": 5.2}');

SELECT * FROM events WHERE data->>'type' = 'click';
```

### When to Use YSQL vs YCQL

```
╔══════════════════════════════════════════════════════════════════════╗
║              YSQL vs YCQL — Decision Guide                           ║
║                                                                      ║
║   Use YSQL (PostgreSQL) when:                                       ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │  • You need JOINs across tables                            │    ║
║   │  • Complex queries (aggregations, subqueries, CTEs)        │    ║
║   │  • Foreign key constraints                                 │    ║
║   │  • Strong schema enforcement                               │    ║
║   │  • Migrating from PostgreSQL, MySQL, or Oracle             │    ║
║   │  • Using PostgreSQL extensions (PostGIS, etc.)             │    ║
║   │  • Ad-hoc analytical queries                               │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║   Use YCQL (Cassandra) when:                                        ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │  • Ultra-fast single-key lookups (sub-2ms)                 │    ║
║   │  • Time-series data with TTL                               │    ║
║   │  • Very high write throughput (100K+ writes/sec)           │    ║
║   │  • Migrating from Cassandra                                │    ║
║   │  • Simple partition-key access patterns                    │    ║
║   │  • Data that expires automatically (TTL)                   │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║   💡 Pro Tip: In the same cluster, use YSQL for your main           ║
║   business logic and YCQL for high-throughput event streams.        ║
║   They share the same DocDB storage!                                ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🌍 Geo-Distribution — YugabyteDB's Strength

### Geo-Partitioning (Row-Level Geo-Placement)

```sql
-- Step 1: Create tablespaces for each region
CREATE TABLESPACE us_tablespace WITH (
    replica_placement = '{"num_replicas": 3, "placement_blocks": [
        {"cloud":"aws","region":"us-east-1","zone":"us-east-1a","min_num_replicas":1},
        {"cloud":"aws","region":"us-east-1","zone":"us-east-1b","min_num_replicas":1},
        {"cloud":"aws","region":"us-east-1","zone":"us-east-1c","min_num_replicas":1}
    ]}'
);

CREATE TABLESPACE eu_tablespace WITH (
    replica_placement = '{"num_replicas": 3, "placement_blocks": [
        {"cloud":"aws","region":"eu-west-1","zone":"eu-west-1a","min_num_replicas":1},
        {"cloud":"aws","region":"eu-west-1","zone":"eu-west-1b","min_num_replicas":1},
        {"cloud":"aws","region":"eu-west-1","zone":"eu-west-1c","min_num_replicas":1}
    ]}'
);

-- Step 2: Create a geo-partitioned table
CREATE TABLE users (
    id UUID DEFAULT gen_random_uuid(),
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    geo_region TEXT NOT NULL,
    data JSONB,
    PRIMARY KEY (id, geo_region)
) PARTITION BY LIST (geo_region);

-- Step 3: Create partitions pinned to regions
CREATE TABLE users_us PARTITION OF users
    FOR VALUES IN ('US')
    TABLESPACE us_tablespace;

CREATE TABLE users_eu PARTITION OF users
    FOR VALUES IN ('EU')
    TABLESPACE eu_tablespace;

-- Step 4: Insert data — automatically routed to correct region!
INSERT INTO users (email, name, geo_region) VALUES
    ('alice@example.com', 'Alice', 'US'),  -- stored in US region
    ('hans@example.com', 'Hans', 'EU');    -- stored in EU region

-- Alice's data NEVER leaves US servers → GDPR compliance!
-- Hans's data NEVER leaves EU servers → data sovereignty!

-- Queries from US users hit US tablets (low latency)
-- Queries from EU users hit EU tablets (low latency)
```

### Follower Reads (Read from Nearest Replica)

```sql
-- For reads where slight staleness is OK (dashboards, reports):
SET yb_read_from_followers = true;
SET yb_follower_read_staleness_ms = 5000;  -- Max 5 seconds stale

SELECT * FROM users WHERE city = 'Tokyo';
-- Reads from the nearest follower replica instead of the leader
-- Result: sub-millisecond reads from any region!

-- Reset for strong reads
SET yb_read_from_followers = false;
```

---

## 📊 Performance Tuning

### Understanding Tablet Distribution

```sql
-- Check tablet distribution
SELECT * FROM yb_local_tablets();

-- Detailed tablet info
SELECT table_name, tablet_id, partition_key_start, partition_key_end
FROM yb_local_tablets()
WHERE table_name = 'users';

-- Check table size and tablet count
SELECT pg_size_pretty(pg_total_relation_size('users')) AS table_size;
```

### Key Performance Strategies

```
╔══════════════════════════════════════════════════════════════════════╗
║              PERFORMANCE OPTIMIZATION GUIDE                          ║
║                                                                      ║
║   1. CHOOSE THE RIGHT SHARDING STRATEGY                            ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │  HASH sharding (default):                                  │    ║
║   │  PRIMARY KEY (id HASH)                                     │    ║
║   │  ✅ Even data distribution                                 │    ║
║   │  ✅ Fast point lookups                                     │    ║
║   │  ❌ Range scans hit ALL tablets (scatter-gather)           │    ║
║   │                                                            │    ║
║   │  RANGE sharding:                                           │    ║
║   │  PRIMARY KEY (date ASC, id)                                │    ║
║   │  ✅ Efficient range scans (date BETWEEN x AND y)          │    ║
║   │  ✅ Ordered data retrieval                                 │    ║
║   │  ❌ Sequential inserts can create hot tablets              │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║   2. COLOCATE SMALL TABLES                                          ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │  Without colocation: 10 small tables = 30 tablets = overhead│   ║
║   │  With colocation: 10 small tables = 3 tablets = efficient  │    ║
║   │                                                            │    ║
║   │  CREATE DATABASE app WITH COLOCATION = true;               │    ║
║   │  -- All tables share tablets by default                    │    ║
║   │  -- Opt out large tables: WITH (COLOCATION = false)        │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║   3. OPTIMIZE TABLET COUNT                                          ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │  Too few tablets: not enough parallelism                   │    ║
║   │  Too many tablets: overhead per tablet (~20MB RAM each)    │    ║
║   │                                                            │    ║
║   │  Rule of thumb:                                            │    ║
║   │  tablets_per_table = num_nodes * cores_per_node / 2       │    ║
║   │  Example: 3 nodes × 8 cores = 12 tablets per table        │    ║
║   │                                                            │    ║
║   │  CREATE TABLE big_table (...) SPLIT INTO 12 TABLETS;      │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
║   4. USE COVERING INDEXES                                           ║
║   ┌────────────────────────────────────────────────────────────┐    ║
║   │  -- Avoid going back to the main table:                    │    ║
║   │  CREATE INDEX idx_users_city ON users (city)               │    ║
║   │    INCLUDE (name, email);                                  │    ║
║   │  -- Query can be answered entirely from the index!        │    ║
║   └────────────────────────────────────────────────────────────┘    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Reading Execution Plans

```sql
-- EXPLAIN
EXPLAIN SELECT * FROM users WHERE city = 'New York';

-- EXPLAIN ANALYZE (with actual timings)
EXPLAIN (ANALYZE, DIST, COSTS) 
SELECT u.name, COUNT(o.id) 
FROM users u JOIN orders o ON u.id = o.user_id 
GROUP BY u.name;

-- The DIST option shows distributed query execution info:
-- • Number of tablets scanned
-- • Number of RPC calls to DocDB
-- • Rows read vs rows returned (selectivity)
```

---

## 🔄 YugabyteDB vs CockroachDB — The Big Comparison

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   Feature              │ YugabyteDB       │ CockroachDB             ║
║   ─────────────────────┼──────────────────┼──────────────           ║
║   SQL API              │ PostgreSQL (YSQL)│ PostgreSQL               ║
║   NoSQL API            │ Cassandra (YCQL) │ ❌ None                  ║
║   Sharding default     │ HASH             │ RANGE                    ║
║   Storage engine       │ RocksDB (per tab)│ Pebble (Go-native)      ║
║   Consensus            │ Raft             │ Raft                     ║
║   Isolation             │ Snapshot (SI)    │ Serializable (SSI)      ║
║   Colocation           │ ✅ Yes           │ ❌ No                    ║
║   PG Extensions        │ ⚠️ Some          │ ❌ Very limited          ║
║   License              │ Apache 2.0       │ BSL → Apache             ║
║   Geo-partitioning     │ ✅ Row-level     │ ✅ Row-level             ║
║   CDC                  │ ✅ (Debezium)    │ ✅ (Changefeeds)         ║
║   Follower reads       │ ✅ Yes           │ ✅ Yes                   ║
║   Admin UI             │ ✅ Built-in      │ ✅ Built-in              ║
║                                                                      ║
║   WHEN TO CHOOSE YugabyteDB:                                       ║
║   • Need Cassandra compatibility (migration or dual API)            ║
║   • Prefer hash-sharding for high-write workloads                  ║
║   • Want true Apache 2.0 license (no BSL restrictions)             ║
║   • Need PostgreSQL extension support                              ║
║   • Use Kubernetes (YugabyteDB has excellent K8s operator)         ║
║                                                                      ║
║   WHEN TO CHOOSE CockroachDB:                                      ║
║   • Need serializable isolation (strongest ACID)                   ║
║   • Prefer range-sharding for ordered data access                  ║
║   • Want the simplest operational experience                       ║
║   • Need built-in CDC to Kafka/S3                                  ║
║   • Want the largest NewSQL community                              ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🔧 Operations & Monitoring

### Built-in Admin UI

```
Master UI:   http://localhost:7000  — Cluster overview, tables, tablets
TServer UI:  http://localhost:9000  — Per-node metrics, tablet details
Yugabyted UI: http://localhost:15433 — Simplified dashboard

┌──────────────────────────────────────────────────────────────┐
│                YugabyteDB Admin UI                            │
│                                                              │
│  📊 Overview                                                 │
│  ├── Cluster health (nodes, tablets, replication)            │
│  ├── Throughput (reads/writes per second)                    │
│  └── Latency (P50, P99)                                     │
│                                                              │
│  📋 Tables                                                   │
│  ├── Table list with sizes                                   │
│  ├── Tablet distribution per table                           │
│  └── Schema details                                          │
│                                                              │
│  🔍 Tablet Servers                                           │
│  ├── Per-node resource usage (CPU, memory, disk)             │
│  ├── Tablet count per server                                 │
│  └── RPC metrics                                             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Operations

```bash
# Check cluster status
yugabyted status

# Add a node to the cluster
yugabyted start --base_dir=/tmp/yb4 --join=127.0.0.1

# Remove a node (graceful)
yugabyted stop --base_dir=/tmp/yb4

# Backup (using ysql_dump — PostgreSQL-compatible)
ysql_dump -h 127.0.0.1 -p 5433 -U yugabyte mydb > backup.sql

# Restore
ysqlsh -h 127.0.0.1 -p 5433 -U yugabyte mydb < backup.sql

# Distributed backup (YugabyteDB's native backup to S3/GCS)
yb_backup.py --masters 127.0.0.1:7100 \
    --remote_yb_admin_binary=/usr/local/bin/yb-admin \
    --storage_type s3 \
    --backup_location s3://my-bucket/backups \
    create
```

---

## 🌍 Real-World YugabyteDB Adoption

| Company | Scale | Use Case |
|---------|-------|----------|
| **Kroger** | Billion+ txns | Real-time pricing, inventory management |
| **Flipkart** | India-scale | Checkout, payments, order management |
| **Nutanix** | Hybrid cloud | Infrastructure management database |
| **Wells Fargo** | Banking scale | Financial transaction processing |
| **GE Digital** | Industrial IoT | Sensor data with time-series + relational |
| **Jio** (India) | 450M+ users | Telecom subscriber management |
| **Infosys** | Enterprise | Distributed microservices data layer |

---

## 🧠 Interview Quick-Fire — YugabyteDB

| Question | Perfect Answer |
|----------|----------------|
| What is YugabyteDB? | An open-source distributed SQL database with dual APIs (PostgreSQL via YSQL and Cassandra via YCQL) built on a common distributed storage engine called DocDB. |
| What is DocDB? | YugabyteDB's distributed document store — data is split into tablets, each replicated via Raft, with RocksDB as the per-tablet storage engine. |
| What's unique about YugabyteDB's sharding? | It defaults to hash-based sharding (consistent hashing) for even data distribution, unlike CockroachDB which uses range-based sharding. |
| Why does it have two APIs? | YSQL for full relational SQL (JOINs, transactions, complex queries) and YCQL for Cassandra-style workloads (fast key lookups, TTL, high write throughput) — both on the same storage. |
| What is tablet colocation? | Grouping multiple small tables into the same tablet to reduce overhead (fewer Raft groups, less memory). Essential for databases with many small tables. |
| How does geo-partitioning work? | Using tablespaces to pin specific partitions (rows) to specific regions. Combined with PostgreSQL's list partitioning, each row can be routed to its region of origin. |
| YSQL vs YCQL — when to use which? | YSQL for relational queries (JOINs, aggregations, foreign keys). YCQL for ultra-fast single-key access, time-series with TTL, or Cassandra migration. |

---

## 📚 Chapter Summary

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. YugabyteDB = open-source distributed SQL with DUAL APIs        ║
║     (PostgreSQL YSQL + Cassandra YCQL)                              ║
║  2. DocDB underneath: tablets + Raft + RocksDB per tablet           ║
║  3. Hash sharding by default (even distribution, fast lookups)      ║
║  4. Colocation groups small tables for efficiency                   ║
║  5. Geo-partitioning pins data to specific regions (GDPR!)         ║
║  6. Follower reads for low-latency reads from any region           ║
║  7. Apache 2.0 license — truly open source                         ║
║  8. Choose YSQL for relational, YCQL for Cassandra-style access    ║
║  9. Best for: dual-API needs, Cassandra migration, geo-distributed ║
║     apps, and teams wanting fully open-source distributed SQL       ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🏁 PART 4 — Complete! Where to Go Next

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   You've completed PART 4 — NewSQL & Distributed SQL! 🎉           ║
║                                                                      ║
║   You now understand:                                                ║
║   ✅ Why NewSQL exists (SQL + NoSQL best of both worlds)            ║
║   ✅ CockroachDB — PostgreSQL-compatible, survive anything          ║
║   ✅ TiDB — MySQL-compatible, HTAP with TiFlash                    ║
║   ✅ Google Spanner — TrueTime, external consistency                ║
║   ✅ YugabyteDB — Dual API (PostgreSQL + Cassandra)                ║
║                                                                      ║
║   Quick Reference:                                                   ║
║   • PostgreSQL user? → CockroachDB or YugabyteDB                   ║
║   • MySQL user? → TiDB                                              ║
║   • Need strongest consistency? → Google Spanner                    ║
║   • Need analytics + OLTP? → TiDB (HTAP)                           ║
║   • Need Cassandra + SQL? → YugabyteDB                              ║
║   • Budget-conscious? → TiDB (Apache 2.0, free forever)            ║
║                                                                      ║
║   Next: PART 5 — Data Engineering & Database Tooling →              ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

> **Back to Index:** [← INDEX](../INDEX.md)
