# 2D.5 — MySQL Replication & Clustering 🔴🔥

> **"One database is a single point of failure. Two is a plan. A cluster is an architecture."**
> This chapter takes you from a single MySQL server to a resilient, horizontally-scaled distributed system.

---

## 🎯 What You'll Master

```
✅ Why replication matters — availability, scalability, disaster recovery
✅ Binary Log (binlog) — the engine behind all MySQL replication
✅ Master-Slave (Source-Replica) replication — async, semi-sync, fully sync
✅ GTID-based replication — the modern, reliable approach
✅ Multi-Source replication — many sources, one replica
✅ Group Replication — MySQL's built-in Paxos-based consensus
✅ InnoDB Cluster — the full HA stack (Group Replication + MySQL Router + MySQL Shell)
✅ ProxySQL — intelligent query routing and connection pooling
✅ Read/Write splitting patterns & failover strategies
✅ Real-world topologies used at scale
```

---

## 🧠 Why Replicate? — The Three Pillars

```
┌────────────────────────────────────────────────────────────────┐
│              WHY MYSQL REPLICATION?                             │
│                                                                │
│  ┌──────────────────┐                                          │
│  │  HIGH AVAILABILITY│  If the primary dies, a replica takes   │
│  │  (HA)            │  over. Your app stays alive.             │
│  └──────────────────┘                                          │
│                                                                │
│  ┌──────────────────┐                                          │
│  │  READ SCALING    │  Send reads to replicas, writes to       │
│  │                  │  primary. Handle 10x more traffic.       │
│  └──────────────────┘                                          │
│                                                                │
│  ┌──────────────────┐                                          │
│  │  DISASTER        │  Replicate to a different data center.   │
│  │  RECOVERY (DR)   │  Survive entire site failures.           │
│  └──────────────────┘                                          │
│                                                                │
│  BONUS USES:                                                   │
│  • Backups from replica (no impact on production)              │
│  • Analytics/reporting on dedicated replica                    │
│  • Rolling upgrades (upgrade replica → promote → upgrade old)  │
│  • Geographic distribution (replicas closer to users)          │
└────────────────────────────────────────────────────────────────┘
```

---

## 🔥 1. Binary Log (Binlog) — The Foundation of Everything

Every replication method in MySQL is built on the **Binary Log**. Understanding it is non-negotiable.

### What Is the Binlog?

```
┌──────────────────────────────────────────────────────────┐
│  CLIENT: INSERT INTO orders VALUES (...)                  │
│              │                                            │
│              ▼                                            │
│  ┌────────────────────┐                                   │
│  │   InnoDB Engine     │  1. Write to buffer pool + redo  │
│  └─────────┬──────────┘                                   │
│            │                                              │
│            ▼                                              │
│  ┌────────────────────┐                                   │
│  │   Binary Log        │  2. Write event to binlog         │
│  │   (binlog.000001)   │     BEFORE commit is confirmed   │
│  └─────────┬──────────┘                                   │
│            │                                              │
│            ▼                                              │
│  ┌────────────────────┐                                   │
│  │   COMMIT confirmed  │  3. Transaction is durable        │
│  │   to client         │                                   │
│  └─────────┬──────────┘                                   │
│            │                                              │
│            │   (Replica pulls binlog)                     │
│            ▼                                              │
│  ┌────────────────────┐                                   │
│  │   REPLICA SERVER    │  4. Replay events on replica      │
│  └────────────────────┘                                   │
└──────────────────────────────────────────────────────────┘
```

### Binlog Formats — Which One to Use?

```sql
-- Check current format
SHOW VARIABLES LIKE 'binlog_format';
```

| Format | How It Works | Pros | Cons |
|--------|-------------|------|------|
| `STATEMENT` | Logs the actual SQL statement | Small binlog size | Non-deterministic functions (UUID(), NOW()) may differ on replica 💀 |
| `ROW` | Logs the **actual row changes** (before/after) | **100% safe.** Every row change is exact | Larger binlog (especially for bulk updates) |
| `MIXED` | STATEMENT by default, switches to ROW when needed | Compromise | Unpredictable when format switches |

```ini
# my.cnf — USE ROW FORMAT IN PRODUCTION. Always.
[mysqld]
binlog_format = ROW
```

> ⭐ **Industry Standard:** `ROW` format. Period. The disk space trade-off is worth the guaranteed consistency. Facebook, GitHub, Shopify — everyone uses ROW.

### Binlog Management

```sql
-- List all binary logs
SHOW BINARY LOGS;

-- Show current binlog position
SHOW MASTER STATUS;
-- or (MySQL 8.0.22+)
SHOW BINARY LOG STATUS;

-- View events in a specific binlog
SHOW BINLOG EVENTS IN 'binlog.000042' LIMIT 20;

-- Purge old binlogs
PURGE BINARY LOGS BEFORE '2024-01-01 00:00:00';
PURGE BINARY LOGS TO 'binlog.000042';

-- Auto-purge (keep 7 days)
SET GLOBAL expire_logs_days = 7;
-- MySQL 8.0+
SET GLOBAL binlog_expire_logs_seconds = 604800;  -- 7 days in seconds
```

---

## 🔥 2. Traditional Replication (Source → Replica)

### The Architecture

```
┌────────────────┐          ┌────────────────┐
│   SOURCE       │          │   REPLICA       │
│  (Primary)     │          │  (Secondary)    │
│                │          │                 │
│  ┌──────────┐  │  binlog  │  ┌──────────┐  │
│  │  Binlog  │──┼─────────►│  │ Relay Log│  │
│  │  Dump    │  │  stream  │  │          │  │
│  │  Thread  │  │          │  └─────┬────┘  │
│  └──────────┘  │          │        │       │
│                │          │  ┌─────▼────┐  │
│                │          │  │ SQL      │  │
│                │          │  │ Thread   │  │
│                │          │  │(Applier) │  │
│                │          │  └──────────┘  │
└────────────────┘          └────────────────┘

Three threads make replication work:
1. Binlog Dump Thread (on Source) — reads binlog, sends to replica
2. I/O Thread (on Replica) — receives binlog events, writes to relay log
3. SQL Thread (on Replica) — reads relay log, replays events on replica
```

### Step-by-Step Setup (Traditional Position-Based)

#### On the SOURCE (Primary):

```ini
# my.cnf
[mysqld]
server-id              = 1          # Unique ID (must be different for each server)
log_bin                = binlog     # Enable binary logging
binlog_format          = ROW        # Row-based replication
sync_binlog            = 1          # Flush binlog on every commit
innodb_flush_log_at_trx_commit = 1  # Full durability
```

```sql
-- Create a replication user
CREATE USER 'repl_user'@'10.0.1.%' IDENTIFIED BY 'StrongP@ss!2024';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'10.0.1.%';
FLUSH PRIVILEGES;

-- Get the current binlog position
SHOW MASTER STATUS;
-- +------------------+----------+--------------+------------------+
-- | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
-- +------------------+----------+--------------+------------------+
-- | binlog.000003    |      785 |              |                  |
-- +------------------+----------+--------------+------------------+
```

#### On the REPLICA (Secondary):

```ini
# my.cnf
[mysqld]
server-id              = 2          # Different from source!
relay_log              = relay-bin  # Relay log name
read_only              = ON         # Prevent accidental writes
super_read_only        = ON         # Even SUPER user can't write (MySQL 5.7+)
```

```sql
-- Point replica to source
CHANGE MASTER TO
    MASTER_HOST = '10.0.1.10',
    MASTER_USER = 'repl_user',
    MASTER_PASSWORD = 'StrongP@ss!2024',
    MASTER_LOG_FILE = 'binlog.000003',
    MASTER_LOG_POS = 785;

-- MySQL 8.0.23+ uses new syntax:
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = '10.0.1.10',
    SOURCE_USER = 'repl_user',
    SOURCE_PASSWORD = 'StrongP@ss!2024',
    SOURCE_LOG_FILE = 'binlog.000003',
    SOURCE_LOG_POS = 785;

-- Start replication
START SLAVE;
-- or (MySQL 8.0.22+)
START REPLICA;

-- Check status
SHOW SLAVE STATUS\G
-- or
SHOW REPLICA STATUS\G
```

### Reading SHOW REPLICA STATUS — What Matters

```sql
SHOW REPLICA STATUS\G
```

| Field | What to Check | Healthy Value |
|-------|---------------|---------------|
| `Replica_IO_Running` | Is the I/O thread connected to source? | `Yes` |
| `Replica_SQL_Running` | Is the SQL thread applying events? | `Yes` |
| `Seconds_Behind_Source` | Replication lag in seconds | `0` (or close) |
| `Last_IO_Error` | I/O thread error | (empty) |
| `Last_SQL_Error` | SQL thread error | (empty) |
| `Retrieved_Gtid_Set` | GTIDs received from source | Should grow |
| `Executed_Gtid_Set` | GTIDs applied on replica | Should match Retrieved |

> 🔴 **RED FLAG:** If `Seconds_Behind_Source` keeps growing, the replica can't keep up with the write load. Common causes: single-threaded SQL applier, slow disk on replica, heavy queries on replica.

---

## 🔥 3. GTID Replication — The Modern Way

**GTID** (Global Transaction Identifier) = a unique ID for every transaction across all servers.

### Why GTIDs are Better

```
TRADITIONAL (Position-Based):
  "Start from binlog.000003, position 785"
   ↓
  Problem: Which binlog file? Position might differ after failover.
  Problem: Failover requires manual position finding. Error-prone. 💀

GTID-Based:
  "Start from GTID 3e11fa47-71ca-11e1-9e33-c80aa9429562:1-42"
   ↓
  Every transaction has a globally unique ID.
  Any replica can become source. Automatic position finding. ✅
```

### GTID Format

```
GTID = source_uuid:transaction_id

Example: 3e11fa47-71ca-11e1-9e33-c80aa9429562:1-150
         ├────────── server UUID ───────────┤ ├─ transactions 1 through 150
```

### Enable GTID

```ini
# my.cnf (BOTH source and replica)
[mysqld]
gtid_mode                = ON
enforce_gtid_consistency = ON
log_bin                  = binlog
binlog_format            = ROW
```

### GTID Replication Setup

```sql
-- On REPLICA:
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = '10.0.1.10',
    SOURCE_USER = 'repl_user',
    SOURCE_PASSWORD = 'StrongP@ss!2024',
    SOURCE_AUTO_POSITION = 1;   -- ← This is the magic. No file/position needed!

START REPLICA;
```

> 💡 **`SOURCE_AUTO_POSITION = 1`** tells the replica: "Figure out where to start automatically using GTIDs." No more hunting for binlog files and positions!

### GTID Constraints — What You Can't Do

```sql
-- ❌ These are NOT allowed with enforce_gtid_consistency = ON:
CREATE TABLE ... SELECT ...          -- Use CREATE TABLE then INSERT ... SELECT
CREATE TEMPORARY TABLE in tx         -- Temp tables inside transactions
Mixed InnoDB + non-InnoDB in same tx -- Don't mix engines in one transaction

-- These restrictions exist because they can't be represented as a single atomic GTID
```

---

## 🔥 4. Asynchronous vs Semi-Synchronous Replication

### Asynchronous (Default)

```
Source: COMMIT → Write to binlog → Return OK to client → Send to replica (eventually)

Timeline:
  Client ──COMMIT──► Source ──OK──► Client  (replica gets it... later)
                                     │
                                     │ (lag gap)
                                     ▼
                                   Replica receives & applies

⚠️ RISK: If source crashes AFTER commit but BEFORE replica receives it,
         that transaction is LOST on failover.
```

### Semi-Synchronous Replication

```
Source: COMMIT → Write to binlog → Wait for AT LEAST 1 replica ACK → Return OK

Timeline:
  Client ──COMMIT──► Source ──────────────────────────────► Source ──OK──► Client
                       │                                       ▲
                       └── Send to replica ── ACK received ────┘
                              (waits for this!)

✅ GUARANTEE: If source says "committed," at least 1 replica has the data.
⚠️ TRADE-OFF: Slightly slower commits (network round-trip to replica).
```

### Enable Semi-Sync

```sql
-- ON SOURCE:
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
-- MySQL 8.0.26+:
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';

SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 5000;  -- 5 seconds; fallback to async if no ACK

-- ON REPLICA:
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
-- MySQL 8.0.26+:
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';

SET GLOBAL rpl_semi_sync_slave_enabled = 1;

-- Restart I/O thread to activate
STOP REPLICA IO_THREAD;
START REPLICA IO_THREAD;
```

### Comparison Matrix

```
┌──────────────────────────────────────────────────────────────┐
│           REPLICATION MODE COMPARISON                        │
├────────────────┬───────────────┬──────────────┬──────────────┤
│                │  Asynchronous │ Semi-Sync    │ Group Repl.  │
│                │  (Default)    │              │ (see §5)     │
├────────────────┼───────────────┼──────────────┼──────────────┤
│ Data Loss Risk │  Possible     │ Minimal      │ None*        │
│ Commit Latency │  Fastest      │ +1 RTT       │ +consensus   │
│ Failover       │  Manual       │ Manual       │ Automatic    │
│ Complexity     │  Low          │ Medium       │ High         │
│ Use Case       │  Read scaling │ Finance/HA   │ Full HA      │
│ Min. Servers   │  2            │ 2            │ 3            │
└────────────────┴───────────────┴──────────────┴──────────────┘
* With majority consensus
```

---

## 🔥 5. Parallel Replication — Making Replicas Faster

By default, the SQL applier thread is **single-threaded** — it applies transactions one by one. This is often the bottleneck for replication lag.

### Multi-Threaded Applier (MTA)

```ini
# my.cnf on REPLICA
[mysqld]
# MySQL 5.7+
slave_parallel_type        = LOGICAL_CLOCK   # Parallel by commit order (best)
slave_parallel_workers     = 8               # Number of applier threads
slave_preserve_commit_order = ON             # Maintain commit order

# MySQL 8.0.27+  (new variable names)
replica_parallel_type        = LOGICAL_CLOCK
replica_parallel_workers     = 8
replica_preserve_commit_order = ON
```

**How It Works:**
```
                       SINGLE-THREADED APPLIER (default):
                       tx1 → tx2 → tx3 → tx4 → tx5 → tx6 → tx7 → tx8
                       ═══════════════════════════════════════════════►
                       Time: ████████████████████████████████████████

                       MULTI-THREADED APPLIER (8 workers):
                       Thread 1: tx1 → tx5
                       Thread 2: tx2 → tx6
                       Thread 3: tx3 → tx7
                       Thread 4: tx4 → tx8
                       ═══════════════►
                       Time: ██████████  (4x faster!)
```

> 💡 **`LOGICAL_CLOCK`** uses the prepare timestamp from the source to determine which transactions can be safely applied in parallel. Transactions that were prepared concurrently on the source can be applied concurrently on the replica.

---

## 🔥 6. Multi-Source Replication — Many Sources, One Replica

```
┌──────────┐
│ Source A  │───┐
│ (US East) │   │
└──────────┘   │    ┌───────────────┐
                ├───►│   REPLICA      │
┌──────────┐   │    │ (Aggregator)  │
│ Source B  │───┤    │               │
│ (US West) │   │    │ Has ALL data  │
└──────────┘   │    │ from A + B + C│
                ├───►│               │
┌──────────┐   │    └───────────────┘
│ Source C  │───┘
│ (Europe)  │
└──────────┘

USE CASE: Centralized reporting/analytics across multiple shards or regions
```

### Setup Multi-Source

```sql
-- Each source gets a named replication CHANNEL
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = '10.0.1.10',
    SOURCE_USER = 'repl_user',
    SOURCE_PASSWORD = 'StrongP@ss!2024',
    SOURCE_AUTO_POSITION = 1
    FOR CHANNEL 'source_us_east';

CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = '10.0.2.10',
    SOURCE_USER = 'repl_user',
    SOURCE_PASSWORD = 'StrongP@ss!2024',
    SOURCE_AUTO_POSITION = 1
    FOR CHANNEL 'source_us_west';

-- Start all channels
START REPLICA FOR CHANNEL 'source_us_east';
START REPLICA FOR CHANNEL 'source_us_west';

-- or start all at once
START REPLICA;

-- Check status per channel
SHOW REPLICA STATUS FOR CHANNEL 'source_us_east'\G
```

> ⚠️ **Multi-Source requires GTID replication** and careful schema management. Avoid auto-increment collisions between sources!

---

## 🔥 7. MySQL Group Replication — Built-in Consensus

Group Replication is MySQL's **native high-availability** solution using the **Paxos consensus protocol** (similar to what Google Spanner uses).

### How It Works

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Member 1 │◄───►│ Member 2 │◄───►│ Member 3 │
│ (PRIMARY)│     │ (PRIMARY │     │ (PRIMARY │
│          │     │   or     │     │   or     │
│  R + W   │     │ SECONDARY│     │ SECONDARY│
└──────────┘     └──────────┘     └──────────┘
      ▲                ▲                ▲
      └────────────────┼────────────────┘
                       │
              GROUP COMMUNICATION
              (Paxos Consensus)

1. Transaction arrives at any member
2. Member proposes it to the group
3. MAJORITY must agree (certify) → committed
4. If conflict detected → rejected (optimistic concurrency)
5. All members apply in the same order → CONSISTENCY ✅
```

### Single-Primary vs Multi-Primary Mode

```
┌────────────────────────────────────────────────────────────┐
│  SINGLE-PRIMARY MODE (Recommended for most use cases)      │
│                                                            │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│  │ PRIMARY  │   │SECONDARY │   │SECONDARY │               │
│  │  R + W   │   │   R only │   │   R only │               │
│  └──────────┘   └──────────┘   └──────────┘               │
│                                                            │
│  • One member handles all writes                           │
│  • Others are read-only                                    │
│  • If primary fails → automatic election of new primary    │
│  • Simpler. No write conflicts. Recommended for most apps. │
├────────────────────────────────────────────────────────────┤
│  MULTI-PRIMARY MODE (Advanced)                             │
│                                                            │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│  │ PRIMARY  │   │ PRIMARY  │   │ PRIMARY  │               │
│  │  R + W   │   │  R + W   │   │  R + W   │               │
│  └──────────┘   └──────────┘   └──────────┘               │
│                                                            │
│  • ALL members accept writes                               │
│  • Conflicting transactions are detected and rolled back   │
│  • Higher write throughput (distributed writes)            │
│  • More complex. Risk of certification conflicts.          │
└────────────────────────────────────────────────────────────┘
```

### Group Replication Setup

```ini
# my.cnf — All members need similar config
[mysqld]
server_id                          = 1                    # Unique per member
gtid_mode                          = ON
enforce_gtid_consistency           = ON
binlog_format                      = ROW
log_bin                            = binlog
binlog_checksum                    = NONE                 # Required for GR
relay_log                          = relay-bin

# Group Replication Settings
plugin_load_add                    = group_replication.so
group_replication_group_name       = "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"  # Same UUID for all members
group_replication_start_on_boot    = OFF
group_replication_local_address    = "10.0.1.10:33061"    # Internal GR communication port
group_replication_group_seeds      = "10.0.1.10:33061,10.0.1.11:33061,10.0.1.12:33061"
group_replication_single_primary_mode = ON                # Single-Primary mode
```

```sql
-- On the FIRST member (bootstrap the group):
SET GLOBAL group_replication_bootstrap_group = ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group = OFF;

-- On subsequent members:
START GROUP_REPLICATION;

-- Check group membership
SELECT * FROM performance_schema.replication_group_members;
-- +--------------------------------------+-----------+-------------+-----------+--------------+
-- | MEMBER_ID                            | MEMBER_HOST| MEMBER_PORT| MEMBER_STATE| MEMBER_ROLE |
-- +--------------------------------------+-----------+-------------+-----------+--------------+
-- | 3e11fa47-71ca-11e1-9e33-c80aa9429562 | node1     | 3306        | ONLINE     | PRIMARY     |
-- | 4e22fb58-82db-22f2-af44-d91bb5530673 | node2     | 3306        | ONLINE     | SECONDARY   |
-- | 5f33gc69-93ec-33g3-bg55-ea2cc6641784 | node3     | 3306        | ONLINE     | SECONDARY   |
-- +--------------------------------------+-----------+-------------+-----------+--------------+

-- Check who is the primary
SELECT MEMBER_HOST, MEMBER_PORT, MEMBER_STATE, MEMBER_ROLE 
FROM performance_schema.replication_group_members 
WHERE MEMBER_ROLE = 'PRIMARY';
```

### Group Replication Requirements & Limitations

```
REQUIREMENTS:
  ✅ InnoDB only (no MyISAM, no MEMORY)
  ✅ Every table MUST have a PRIMARY KEY
  ✅ GTID mode = ON
  ✅ ROW-based binlog format
  ✅ Binlog checksum = NONE
  ✅ Minimum 3 members (odd number recommended for quorum)
  ✅ Low-latency network between members (< 5ms ideal)

LIMITATIONS:
  ❌ Max 9 members per group
  ❌ No SERIALIZABLE isolation level
  ❌ No table locks (LOCK TABLES)
  ❌ Large transactions may time out during certification
  ❌ Cross-datacenter: possible but latency hurts performance
```

---

## 🔥 8. InnoDB Cluster — The Complete HA Solution

InnoDB Cluster is MySQL's **official, integrated high-availability stack**:

```
┌──────────────────────────────────────────────────────────────┐
│                    InnoDB Cluster Stack                       │
│                                                              │
│   APPLICATION                                                │
│       │                                                      │
│       ▼                                                      │
│   ┌──────────────┐    Intelligent routing                    │
│   │ MySQL Router │    • R/W → Primary                        │
│   │   (Proxy)    │    • R/O → any Secondary                  │
│   └──────┬───────┘    • Automatic failover redirection       │
│          │                                                   │
│   ┌──────▼───────────────────────────────────────┐           │
│   │          GROUP REPLICATION                    │           │
│   │                                               │           │
│   │   ┌──────┐    ┌──────┐    ┌──────┐           │           │
│   │   │Node 1│◄──►│Node 2│◄──►│Node 3│           │           │
│   │   │ (P)  │    │ (S)  │    │ (S)  │           │           │
│   │   └──────┘    └──────┘    └──────┘           │           │
│   │                                               │           │
│   └───────────────────────────────────────────────┘           │
│                                                              │
│   ┌──────────────┐                                           │
│   │  MySQL Shell  │    Management & administration           │
│   │  (AdminAPI)   │    • Create/manage cluster               │
│   └──────────────┘    • Add/remove members                   │
│                       • Status monitoring                    │
└──────────────────────────────────────────────────────────────┘
```

### Create an InnoDB Cluster with MySQL Shell

```javascript
// MySQL Shell — JavaScript mode
// Connect to the first node
mysqlsh> \connect root@node1:3306

// Configure the instance for InnoDB Cluster
mysqlsh> dba.configureInstance('root@node1:3306')
mysqlsh> dba.configureInstance('root@node2:3306')
mysqlsh> dba.configureInstance('root@node3:3306')

// Create the cluster
mysqlsh> var cluster = dba.createCluster('myCluster')

// Add members
mysqlsh> cluster.addInstance('root@node2:3306')
mysqlsh> cluster.addInstance('root@node3:3306')

// Check cluster status
mysqlsh> cluster.status()
// {
//     "clusterName": "myCluster",
//     "defaultReplicaSet": {
//         "name": "default",
//         "primary": "node1:3306",
//         "status": "OK",
//         "topology": {
//             "node1:3306": { "status": "ONLINE", "mode": "R/W" },
//             "node2:3306": { "status": "ONLINE", "mode": "R/O" },
//             "node3:3306": { "status": "ONLINE", "mode": "R/O" }
//         }
//     }
// }
```

### MySQL Router — The Traffic Director

```bash
# Bootstrap router against the cluster (auto-generates config)
mysqlrouter --bootstrap root@node1:3306 --user=mysqlrouter

# Router creates these ports by default:
# Port 6446 — Read/Write (routes to PRIMARY)
# Port 6447 — Read-Only (round-robins across SECONDARIES)
# Port 64460 — R/W (X Protocol)
# Port 64470 — R/O (X Protocol)
```

**Application Connection:**
```python
# Application code — connect through MySQL Router
import mysql.connector

# Writes → port 6446 (always goes to PRIMARY)
write_conn = mysql.connector.connect(
    host='mysql-router.internal',
    port=6446,
    user='app_user',
    password='secret'
)

# Reads → port 6447 (distributed across SECONDARIES)
read_conn = mysql.connector.connect(
    host='mysql-router.internal',
    port=6447,
    user='app_user',
    password='secret'
)
```

### Failover Scenario — What Happens When Primary Dies

```
BEFORE FAILURE:
  App → Router:6446 → Node1 (PRIMARY)  ← writes
  App → Router:6447 → Node2/Node3      ← reads

NODE 1 CRASHES! 💥

AUTOMATIC RECOVERY (< 30 seconds):
  1. Group Replication detects Node1 is unreachable
  2. Remaining members (Node2, Node3) form quorum (2/3 = majority ✅)
  3. Group Replication elects new PRIMARY (e.g., Node2)
  4. MySQL Router detects topology change
  5. Router redirects port 6446 to Node2

AFTER FAILOVER:
  App → Router:6446 → Node2 (NEW PRIMARY)  ← writes
  App → Router:6447 → Node3                ← reads
  App didn't need to change ANYTHING. ✅

WHEN NODE 1 COMES BACK:
  Node1 automatically rejoins as SECONDARY
  Catches up using GTIDs
```

---

## 🔥 9. ProxySQL — The Power Tool for MySQL Routing

ProxySQL is a **high-performance MySQL proxy** used by companies like Wikipedia, Percona, and many large-scale MySQL deployments.

### Why ProxySQL Over MySQL Router?

| Feature | MySQL Router | ProxySQL |
|---------|-------------|----------|
| Query routing | R/W split only | R/W split + query rules |
| Query caching | ❌ | ✅ Built-in |
| Connection pooling | Basic | Advanced multiplexing |
| Query rewriting | ❌ | ✅ Regex-based |
| Query mirror/firewall | ❌ | ✅ |
| Admin interface | Config file | Runtime SQL admin |
| Monitoring | Basic | Detailed statistics |

### ProxySQL Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        ProxySQL                               │
│                                                              │
│  ┌──────────────────────────────────────────┐                 │
│  │          QUERY PROCESSOR                  │                 │
│  │                                          │                 │
│  │  1. Connection Pool (multiplexing)       │                 │
│  │  2. Query Rules (routing, rewriting)     │                 │
│  │  3. Query Cache                          │                 │
│  │  4. Query Firewall                       │                 │
│  └─────────────────┬────────────────────────┘                 │
│                    │                                          │
│         ┌──────────┼──────────┐                               │
│         ▼          ▼          ▼                               │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐                     │
│   │ HG 10    │ │ HG 20    │ │ HG 30    │  ← Hostgroups       │
│   │ (Writer) │ │ (Reader) │ │(Analytics│                     │
│   │ Primary  │ │ Replicas │ │ Replica) │                     │
│   └──────────┘ └──────────┘ └──────────┘                     │
└──────────────────────────────────────────────────────────────┘
```

### ProxySQL Configuration (via Admin Interface)

```sql
-- Connect to ProxySQL admin (port 6032)
-- mysql -u admin -padmin -h 127.0.0.1 -P 6032

-- Add MySQL servers
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight) VALUES
    (10, 'node1', 3306, 1000),    -- Writer hostgroup
    (20, 'node2', 3306, 1000),    -- Reader hostgroup
    (20, 'node3', 3306, 1000);    -- Reader hostgroup

-- Add query rules for R/W splitting
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply) VALUES
    (1, 1, '^SELECT .* FOR UPDATE', 10, 1),    -- SELECT FOR UPDATE → writer
    (2, 1, '^SELECT',               20, 1),    -- All other SELECTs → readers
    (3, 1, '.*',                    10, 1);    -- Everything else → writer

-- Add MySQL users
INSERT INTO mysql_users (username, password, default_hostgroup) VALUES
    ('app_user', 'password', 10);

-- Apply and save
LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
SAVE MYSQL QUERY RULES TO DISK;
SAVE MYSQL USERS TO DISK;
```

---

## 🔥 10. Replication Topologies — Real-World Architectures

### Topology 1: Simple Primary + Replicas

```
                    ┌──────────┐
            ┌──────►│ Replica 1│  (Reads)
            │       └──────────┘
┌────────┐  │       ┌──────────┐
│ Primary│──┼──────►│ Replica 2│  (Reads)
│  (R+W) │  │       └──────────┘
└────────┘  │       ┌──────────┐
            └──────►│ Replica 3│  (Analytics / Backup)
                    └──────────┘

USE CASE: Read-heavy workloads (80% reads, 20% writes)
SCALE: Up to 5-10 replicas (beyond that, binlog dump overhead)
```

### Topology 2: Chain Replication (Cascading)

```
┌────────┐     ┌──────────┐     ┌──────────┐
│ Primary│────►│ Relay     │────►│ Replica 2│
│        │     │ (Replica 1│     │          │
└────────┘     └──────────┘     └──────────┘

Relay replica re-distributes binlog.
Reduces load on primary (only 1 binlog dump thread instead of N).
USE CASE: Large number of replicas, cross-datacenter replication.
```

### Topology 3: Dual-Primary (Active-Passive)

```
┌────────────┐              ┌────────────┐
│  Primary A │◄────────────►│  Primary B │
│  (ACTIVE)  │  circular    │ (STANDBY)  │
│   R + W    │  replication │   R only   │
└────────────┘              └────────────┘

Only ONE primary accepts writes at a time.
The other is a hot standby for instant failover.
⚠️ NEVER write to both simultaneously without conflict resolution!
```

---

## 🧠 Replication Monitoring Queries

```sql
-- Current replication lag
SHOW REPLICA STATUS\G
-- Look at: Seconds_Behind_Source

-- Performance Schema replication tables (MySQL 8.0+)
SELECT * FROM performance_schema.replication_connection_status\G
SELECT * FROM performance_schema.replication_applier_status\G
SELECT * FROM performance_schema.replication_applier_status_by_worker\G

-- Group Replication specific
SELECT * FROM performance_schema.replication_group_members;
SELECT * FROM performance_schema.replication_group_member_stats\G

-- Binlog position on source vs replica
-- On Source:
SHOW BINARY LOG STATUS;
-- On Replica:
SHOW REPLICA STATUS\G
-- Compare Exec_Source_Log_Pos with source's Position
```

---

## 🎯 Key Takeaways

```
1.  Binlog is the foundation — understand ROW format, always use it
2.  GTID replication is the modern standard — no more file/position tracking
3.  Semi-sync for critical data — balance durability vs latency
4.  Parallel replication — essential for high-write workloads
5.  Group Replication — built-in consensus for automatic failover
6.  InnoDB Cluster = Group Replication + MySQL Router + MySQL Shell
7.  ProxySQL for advanced routing — query rules, caching, multiplexing
8.  read_only + super_read_only on replicas — prevent accidental writes
9.  Monitor Seconds_Behind_Source — replication lag is your #1 metric
10. Test failover regularly — an untested failover plan is no plan at all
```

---

> **Next:** [2D.6 — MySQL Administration & Security](./06-MySQL-Admin.md) — User management, privileges, encryption, backups, and operational best practices.
