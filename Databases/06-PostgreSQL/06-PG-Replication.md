# 2E.6 — PostgreSQL Replication & High Availability 🔴🔥

> **"The best database in the world means nothing if it's down when your users need it."**
> Build systems that survive server crashes, data center failures, and 3 AM emergencies.

---

## 🎯 What You'll Master

```
✅ Why replication matters — RPO, RTO, and the cost of downtime
✅ Streaming Replication — real-time byte-level copy
✅ Logical Replication — selective, cross-version, flexible
✅ Synchronous vs Asynchronous replication trade-offs
✅ Failover strategies — manual and automatic
✅ Patroni — automated HA for production PostgreSQL
✅ PgBouncer — connection pooling at scale
✅ HAProxy / Keepalived — load balancing replicas
✅ Multi-region architectures
✅ Real-world HA patterns used by Netflix, Instagram, GitLab
```

---

## 🧠 The Big Picture — Why Replication?

```
┌──────────────────────────────────────────────────────────────┐
│          The Three Pillars of Database Reliability            │
├──────────┬───────────────────────────────────────────────────┤
│  Backup  │ Protect against data loss (accidental DELETE,     │
│          │ corruption, ransomware). Recovery takes hours.    │
├──────────┼───────────────────────────────────────────────────┤
│ Replica- │ Protect against downtime. A standby server takes  │
│   tion   │ over in seconds/minutes. Also distributes reads.  │
├──────────┼───────────────────────────────────────────────────┤
│ Cluster- │ Protect against site failures. Multiple active     │
│   ing    │ nodes across data centers. Zero downtime.         │
└──────────┴───────────────────────────────────────────────────┘
```

### Key Concepts

| Term | Meaning | Example |
|------|---------|---------|
| **RPO** (Recovery Point Objective) | How much data can you afford to lose? | "We can't lose more than 5 seconds of transactions" |
| **RTO** (Recovery Time Objective) | How quickly must the system recover? | "We must be back online within 30 seconds" |
| **Primary** | The main server accepting all writes | Your production PostgreSQL instance |
| **Standby / Replica** | Read-only copy of the primary | A second server receiving replicated data |
| **Hot Standby** | Replica that accepts read queries while replicating | ✅ PostgreSQL supports this natively |
| **Warm Standby** | Replica that replicates but doesn't accept any queries | Used for pure failover |
| **Failover** | Promoting a standby to become the new primary | When the primary crashes |
| **Switchover** | Planned, graceful swap of primary and standby roles | During maintenance windows |

---

## 📡 Streaming Replication — The Foundation

Streaming replication is PostgreSQL's **native, built-in** replication. It copies the WAL (Write-Ahead Log) stream from primary to standby in real-time.

### How It Works

```
┌─────────────┐         WAL Stream         ┌──────────────┐
│   PRIMARY   │ ──────────────────────────→ │   STANDBY    │
│  (Read/Write)│    (continuous binary       │  (Read-Only) │
│             │     byte-by-byte copy)      │              │
│  Port 5432  │                             │  Port 5432   │
│             │  ← Feedback (LSN position)  │              │
└─────────────┘                             └──────────────┘

1. Client writes to PRIMARY
2. PRIMARY writes to WAL (Write-Ahead Log)
3. WAL records stream to STANDBY over TCP
4. STANDBY replays WAL records → identical copy
5. STANDBY sends feedback: "I've applied up to LSN X/Y"
```

> 💡 **WAL = Write-Ahead Log:** Every change PostgreSQL makes is first written to WAL before being applied to data files. This is the same log used for crash recovery. Streaming replication simply ships this log to another server.

---

### Setting Up Streaming Replication

#### Step 1: Configure the Primary

```ini
# postgresql.conf on PRIMARY

# Enable WAL shipping
wal_level = replica               # 'replica' or 'logical' (not 'minimal')
max_wal_senders = 5               # Max number of standby connections
wal_keep_size = 1GB               # Keep this much WAL for slow standbys (PG 13+)
# wal_keep_segments = 64          # (PG 12 and earlier)

# Optional: Enable replication slots (prevents WAL cleanup before standby consumes it)
max_replication_slots = 5
```

```ini
# pg_hba.conf on PRIMARY — Allow the standby to connect for replication

# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    replication     replicator      10.0.1.0/24             scram-sha-256
```

```sql
-- Create a replication user on PRIMARY
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_password_here';
```

#### Step 2: Create a Base Backup on the Standby

```bash
# On the STANDBY server:
# Stop PostgreSQL, clear the data directory, then:

pg_basebackup -h primary-host -D /var/lib/postgresql/16/main \
    -U replicator -Fp -Xs -P -R

# -h  = primary host
# -D  = data directory on standby
# -U  = replication user
# -Fp = plain format (not tar)
# -Xs = stream WAL during backup (ensures consistency)
# -P  = show progress
# -R  = create standby.signal and configure recovery  ← KEY FLAG
```

> 💡 The `-R` flag automatically creates the `standby.signal` file and adds the `primary_conninfo` to `postgresql.auto.conf`. No manual config needed!

#### Step 3: Verify the Auto-Generated Config

```ini
# postgresql.auto.conf on STANDBY (auto-created by pg_basebackup -R)
primary_conninfo = 'host=primary-host port=5432 user=replicator password=strong_password_here'
```

```bash
# A file named 'standby.signal' exists in the data directory
# This tells PostgreSQL: "I am a standby server"
ls /var/lib/postgresql/16/main/standby.signal
```

#### Step 4: Start the Standby and Verify

```bash
# Start PostgreSQL on the standby
sudo systemctl start postgresql
```

```sql
-- On PRIMARY: Check replication status
SELECT 
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes,
    sync_state
FROM pg_stat_replication;
```

```
 client_addr  |   state   |  sent_lsn  | replay_lsn | lag_bytes | sync_state
--------------+-----------+------------+------------+-----------+------------
 10.0.1.20    | streaming | 0/3000148  | 0/3000148  |         0 | async
```

```sql
-- On STANDBY: Verify it's in recovery mode
SELECT pg_is_in_recovery();
-- Returns: true  ← This is a standby

-- Check replication lag
SELECT 
    now() - pg_last_xact_replay_timestamp() AS replication_lag;
-- Returns: 00:00:00.003  ← 3 milliseconds behind. Excellent.
```

---

### Replication Slots — Prevent WAL Loss

Without replication slots, if a standby falls behind, the primary may delete WAL files the standby hasn't consumed yet. The standby then can't catch up. 💥

```sql
-- On PRIMARY: Create a physical replication slot
SELECT pg_create_physical_replication_slot('standby1_slot');

-- On STANDBY: Use the slot (postgresql.conf or postgresql.auto.conf)
-- primary_slot_name = 'standby1_slot'
```

```
Without slot:

PRIMARY WAL:  [seg1] [seg2] [seg3] [seg4] [seg5]
                 ↑ deleted          ↑ standby is here
                                    💥 Can't replay seg1-2!

With slot:

PRIMARY WAL:  [seg1] [seg2] [seg3] [seg4] [seg5]
              ↑ kept because slot says standby needs it
                                    ↑ standby is here
                                    ✅ Can replay everything
```

> ⚠️ **Danger:** If a standby goes offline for days with a replication slot, the primary keeps accumulating WAL, potentially filling up the disk. Monitor slot lag:
> ```sql
> SELECT slot_name, active, 
>        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
> FROM pg_replication_slots;
> ```

---

## ⚡ Synchronous vs Asynchronous Replication

### Asynchronous (Default)

```
Client → PRIMARY (commits immediately) → ack to client
                    ↓ (streams WAL async)
                 STANDBY (applies later, slight delay)

✅ Faster writes (client doesn't wait for standby)
❌ Risk: If primary crashes BEFORE standby receives WAL → data lost
   (typically 0-1 second of data)
```

### Synchronous

```
Client → PRIMARY (writes WAL) → waits for STANDBY confirmation → ack to client
                    ↓ (streams WAL)
                 STANDBY (applies WAL, sends confirmation back)

✅ Zero data loss guaranteed (RPO = 0)
❌ Slower writes (every commit waits for network round-trip to standby)
❌ If standby is down, PRIMARY hangs! (all writes blocked)
```

### Configuration

```ini
# On PRIMARY postgresql.conf:

# Enable synchronous replication
synchronous_standby_names = 'standby1'

# Multiple standbys — FIRST 1 = at least 1 must confirm
synchronous_standby_names = 'FIRST 1 (standby1, standby2)'

# ANY 2 of 3 must confirm (for maximum safety)
synchronous_standby_names = 'ANY 2 (standby1, standby2, standby3)'
```

### Synchronous Commit Levels

```sql
-- Per-transaction control! Not all data needs the same durability.

-- Maximum safety: wait for standby to flush WAL to disk
SET synchronous_commit = 'remote_apply';  -- Wait for standby to APPLY changes

-- High safety: wait for standby to write WAL (but not necessarily apply)
SET synchronous_commit = 'remote_write';

-- Standard: wait for standby to receive WAL
SET synchronous_commit = 'on';  -- default when sync replication is configured

-- Fastest: don't even wait for local WAL flush
SET synchronous_commit = 'off';  -- ⚠️ Risk of losing last ~600ms of commits on crash
```

```
Durability vs Performance Spectrum:

  MOST DURABLE                                    FASTEST
  ←──────────────────────────────────────────────→
  remote_apply  >  remote_write  >  on  >  local  >  off

  remote_apply: Standby has applied changes (readable immediately)
  remote_write: Standby OS has received (but might not have flushed)
  on:           Local WAL is flushed to disk
  local:        Same as 'on' (ignores synchronous standby)
  off:          Returns before local WAL flush (dangerous!)
```

> 💡 **Real-World Pattern:** Use `synchronous_commit = 'on'` for financial transactions, `'off'` for logging/analytics where losing a few milliseconds of data is acceptable.

---

## 🔀 Logical Replication — The Flexible Option

Logical replication sends **row-level changes** (INSERT/UPDATE/DELETE) rather than WAL byte streams. This gives you incredible flexibility.

### Streaming vs Logical — When to Use Which?

| Feature | Streaming Replication | Logical Replication |
|---------|----------------------|---------------------|
| **Data copied** | Entire database cluster (all DBs) | Selected tables/databases |
| **Cross-version** | ❌ Must be same major version | ✅ Different PG versions OK |
| **Cross-platform** | ❌ Same OS/architecture | ✅ Different platforms OK |
| **Write to replica** | ❌ Read-only | ✅ Can write to other tables |
| **Selective tables** | ❌ All or nothing | ✅ Pick specific tables |
| **Use for failover** | ✅ Yes (standard HA) | ❌ Not designed for failover |
| **Performance** | Lower overhead | Higher overhead (decoding WAL) |
| **Sequences** | ✅ Replicated | ❌ Not replicated |
| **DDL (CREATE TABLE)** | ✅ Replicated | ❌ Not replicated |
| **Setup complexity** | Low | Medium |

### Setting Up Logical Replication

```ini
# postgresql.conf on PUBLISHER (source)
wal_level = logical               # Must be 'logical' (not 'replica')
max_replication_slots = 5
max_wal_senders = 5
```

```sql
-- On PUBLISHER: Create a publication
CREATE PUBLICATION my_pub FOR TABLE orders, customers;

-- Publish ALL tables:
CREATE PUBLICATION my_pub FOR ALL TABLES;

-- Publish with filters (PostgreSQL 15+):
CREATE PUBLICATION my_pub FOR TABLE orders WHERE (status = 'active');

-- Publish only specific operations:
CREATE PUBLICATION my_pub FOR TABLE orders 
    WITH (publish = 'insert, update');  -- Skip deletes
```

```sql
-- On SUBSCRIBER: Create a subscription
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=publisher-host port=5432 dbname=mydb user=replicator password=secret'
    PUBLICATION my_pub;

-- ⚡ PostgreSQL automatically:
-- 1. Creates a replication slot on the publisher
-- 2. Copies existing data (initial table sync)
-- 3. Starts streaming changes in real-time
```

### Managing Logical Replication

```sql
-- Check subscription status
SELECT * FROM pg_stat_subscription;

-- Check publication tables
SELECT * FROM pg_publication_tables WHERE pubname = 'my_pub';

-- Add a table to an existing publication
ALTER PUBLICATION my_pub ADD TABLE products;

-- On subscriber: refresh to pick up new tables
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION;

-- Temporarily disable replication
ALTER SUBSCRIPTION my_sub DISABLE;

-- Re-enable
ALTER SUBSCRIPTION my_sub ENABLE;

-- Remove subscription (also drops the replication slot on publisher)
DROP SUBSCRIPTION my_sub;
```

### Logical Replication Use Cases

```
1. ZERO-DOWNTIME MAJOR VERSION UPGRADE
   PG 14 (publisher) → PG 16 (subscriber)
   Replicate data, switch traffic, done!

2. SELECTIVE DATA SHARING
   Replicate only 'orders' table from production to analytics DB.

3. DATA CONSOLIDATION
   Multiple source databases → One central reporting database.

4. MULTI-MASTER (limited)
   Server A publishes table X, subscribes to table Y from Server B.
   Server B publishes table Y, subscribes to table X from Server A.
   ⚠️ Don't publish the same table in both directions (conflict!)

5. CROSS-CLOUD REPLICATION
   AWS RDS → GCP Cloud SQL (both running PostgreSQL)
```

---

## 🤖 Patroni — Automated High Availability

Patroni is the **industry standard** for automated PostgreSQL HA. It handles failover, switchover, and cluster management automatically.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PATRONI CLUSTER                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│  │  Node 1   │  │  Node 2   │  │  Node 3   │               │
│  │ Patroni   │  │ Patroni   │  │ Patroni   │               │
│  │ PostgreSQL│  │ PostgreSQL│  │ PostgreSQL│               │
│  │ (PRIMARY) │  │ (STANDBY) │  │ (STANDBY) │               │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘               │
│        │              │              │                      │
│        └──────────────┼──────────────┘                      │
│                       │                                     │
│              ┌────────▼────────┐                            │
│              │   Distributed   │  etcd / ZooKeeper /        │
│              │   Consensus     │  Consul                    │
│              │   Store (DCS)   │                            │
│              └─────────────────┘                            │
│                                                             │
│  How it works:                                              │
│  1. Patroni on each node monitors PostgreSQL health         │
│  2. Leader election via DCS (etcd/ZK/Consul)                │
│  3. If PRIMARY dies, Patroni promotes a STANDBY             │
│  4. Other standbys automatically follow the new primary     │
│  5. Entire failover: 10-30 seconds                          │
└─────────────────────────────────────────────────────────────┘
```

### Patroni Configuration

```yaml
# patroni.yml (per node)
scope: my-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.10:8008

etcd3:
  hosts: 10.0.2.10:2379,10.0.2.11:2379,10.0.2.12:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB — don't promote if lag > this
    postgresql:
      use_pg_rewind: true              # Re-sync old primary as standby
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 5
        max_replication_slots: 5
        wal_log_hints: "on"            # Required for pg_rewind

  initdb:
    - encoding: UTF8
    - data-checksums                    # Detect silent corruption

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.10:5432
  data_dir: /var/lib/postgresql/16/main
  authentication:
    superuser:
      username: postgres
      password: supersecret
    replication:
      username: replicator
      password: replsecret
```

### Patroni Commands

```bash
# Check cluster status
patronictl -c /etc/patroni/patroni.yml list

# Output:
# +--------+--------+---------+---------+----+-----------+
# | Member | Host   | Role    | State   | TL | Lag in MB |
# +--------+--------+---------+---------+----+-----------+
# | node1  | 10.0.1.10 | Leader  | running | 5  |         0 |
# | node2  | 10.0.1.11 | Replica | running | 5  |         0 |
# | node3  | 10.0.1.12 | Replica | running | 5  |         0 |
# +--------+--------+---------+---------+----+-----------+

# Planned switchover (graceful, zero data loss)
patronictl -c /etc/patroni/patroni.yml switchover --master node1 --candidate node2

# Force failover (when primary is unreachable)
patronictl -c /etc/patroni/patroni.yml failover

# Restart PostgreSQL on a node
patronictl -c /etc/patroni/patroni.yml restart my-cluster node1

# Reinitialize a node (rebuild from scratch)
patronictl -c /etc/patroni/patroni.yml reinit my-cluster node3

# Edit cluster-wide PostgreSQL config
patronictl -c /etc/patroni/patroni.yml edit-config
```

### What Happens During Automatic Failover

```
Timeline of a Patroni Failover:

T+0s    Primary (node1) crashes 💥
T+0s    Patroni on node1 stops sending heartbeats to etcd
T+10s   etcd TTL expires → leader key released
T+11s   Patroni on node2 acquires the leader key
T+12s   Patroni promotes PostgreSQL on node2 to primary
T+13s   node2 starts accepting writes
T+14s   node3's Patroni detects new leader, reconfigures to follow node2
T+15s   Cluster is fully operational with new primary

Total downtime: ~15 seconds (depending on TTL and loop_wait settings)
```

> 💡 **pg_rewind:** When the old primary (node1) comes back online, Patroni uses `pg_rewind` to automatically resync it as a standby of the new primary (node2). No manual intervention needed!

---

## 🔄 HAProxy — Load Balancing Reads Across Replicas

```
                    ┌──────────────┐
                    │   HAProxy    │
                    │  Port 5000   │ ← Writes (route to primary)
                    │  Port 5001   │ ← Reads (distribute across replicas)
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  node1   │ │  node2   │ │  node3   │
        │ PRIMARY  │ │ REPLICA  │ │ REPLICA  │
        │ R/W      │ │ R/O      │ │ R/O      │
        └──────────┘ └──────────┘ └──────────┘
```

### HAProxy Configuration with Patroni Health Checks

```
# haproxy.cfg

global
    maxconn 1000

defaults
    mode tcp
    timeout connect 5s
    timeout client  30s
    timeout server  30s

# ─── WRITE PORT: Routes to PRIMARY only ───
listen postgresql_write
    bind *:5000
    option httpchk GET /primary      # Patroni REST API endpoint
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 10.0.1.10:5432 maxconn 100 check port 8008
    server node2 10.0.1.11:5432 maxconn 100 check port 8008
    server node3 10.0.1.12:5432 maxconn 100 check port 8008

# ─── READ PORT: Routes to REPLICAS (round-robin) ───
listen postgresql_read
    bind *:5001
    balance roundrobin
    option httpchk GET /replica      # Only healthy replicas
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 10.0.1.10:5432 maxconn 100 check port 8008
    server node2 10.0.1.11:5432 maxconn 100 check port 8008
    server node3 10.0.1.12:5432 maxconn 100 check port 8008
```

### Application Connection Pattern

```python
# Application code — separate read and write connections

# Writes → HAProxy port 5000 (always hits primary)
write_conn = psycopg2.connect(
    host='haproxy-host', port=5000, dbname='mydb'
)

# Reads → HAProxy port 5001 (distributed across replicas)
read_conn = psycopg2.connect(
    host='haproxy-host', port=5001, dbname='mydb'
)

# ⚠️ Be aware of replication lag!
# After a write, an immediate read from a replica might not see the change.
# For critical reads-after-writes, use the write connection.
```

---

## 🏗️ Production HA Architecture Patterns

### Pattern 1: Standard 3-Node HA (Most Common)

```
┌──────────────────────────────────────────────────────────────┐
│                    Single Data Center                         │
│                                                              │
│  App → PgBouncer → HAProxy ──┬── Node1 (Primary)            │
│                              ├── Node2 (Sync Standby)        │
│                              └── Node3 (Async Standby)       │
│                                                              │
│  All three nodes run Patroni + etcd                          │
│                                                              │
│  RPO: 0 (sync standby)                                       │
│  RTO: 15-30 seconds (Patroni auto-failover)                  │
│  Cost: 3 servers                                             │
└──────────────────────────────────────────────────────────────┘
```

### Pattern 2: Multi-Region HA (High Durability)

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Region A (US-East)              Region B (US-West)          │
│  ┌────────────────┐              ┌────────────────┐          │
│  │ Node1 (Primary)│─── async ──→│ Node3 (Standby) │          │
│  │ Node2 (Sync)   │             │ Node4 (Standby) │          │
│  └────────────────┘              └────────────────┘          │
│                                                              │
│  Sync replication within region (zero data loss)             │
│  Async replication across regions (minimal lag)              │
│                                                              │
│  RPO: 0 within region, ~1s cross-region                      │
│  RTO: 15-30s within region, 1-5 min cross-region             │
│  Survives: Server failure, full data center failure           │
└──────────────────────────────────────────────────────────────┘
```

### Pattern 3: Read Scaling (Heavy Read Workloads)

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Writes:  App → PgBouncer → Primary                          │
│                                                              │
│  Reads:   App → PgBouncer → HAProxy → ┬ Replica 1           │
│                                        ├ Replica 2           │
│                                        ├ Replica 3           │
│                                        ├ Replica 4           │
│                                        └ Replica 5           │
│                                                              │
│  One primary handles all writes.                             │
│  Five replicas share the read load.                          │
│  Add more replicas as traffic grows.                         │
│                                                              │
│  Used by: Instagram (12+ replicas per cluster)               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🛡️ Failover Deep Dive

### Manual Failover (Without Patroni)

```sql
-- Step 1: On the STANDBY that will become primary
-- Promote it
SELECT pg_promote();
-- Or from command line:
-- pg_ctl promote -D /var/lib/postgresql/16/main

-- Step 2: Verify it's now a primary
SELECT pg_is_in_recovery();
-- Returns: false  ← No longer in recovery = it's the primary now

-- Step 3: Point your application to the new primary
-- Update connection strings, DNS, or HAProxy config

-- Step 4: Rebuild the old primary as a standby
-- Option A: pg_rewind (fast, if WAL is available)
pg_rewind --target-pgdata=/var/lib/postgresql/16/main \
          --source-server='host=new-primary port=5432 user=postgres'

-- Option B: pg_basebackup (full copy, always works)
pg_basebackup -h new-primary -D /var/lib/postgresql/16/main -U replicator -Fp -Xs -P -R
```

### Timeline and Timeline IDs

```
Every time a standby is promoted, it starts a new TIMELINE.
This prevents the old primary from accidentally connecting and causing conflicts.

Timeline 1: Primary (node1) ──────────┬── node1 crashes here
                                      │
Timeline 2:             Standby (node2) promoted ──────── continues

When node1 comes back, it's on Timeline 1.
node2 is on Timeline 2.
pg_rewind can fast-forward node1 from TL1 to TL2.
```

---

## 📊 Monitoring Replication Health

### Essential Monitoring Queries

```sql
-- ═══ On PRIMARY ═══

-- 1. Replication status & lag
SELECT 
    client_addr,
    application_name,
    state,
    sync_state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS replay_lag_pretty
FROM pg_stat_replication;

-- 2. Replication slot status
SELECT 
    slot_name, 
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS slot_lag
FROM pg_replication_slots;

-- ═══ On STANDBY ═══

-- 3. Replication lag in seconds
SELECT 
    CASE 
        WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() THEN 0
        ELSE EXTRACT(EPOCH FROM now() - pg_last_xact_replay_timestamp())
    END AS lag_seconds;

-- 4. WAL receiver status
SELECT * FROM pg_stat_wal_receiver;
```

### Alerting Thresholds

```
 Metric                    │ Warning  │ Critical
───────────────────────────┼──────────┼──────────
 Replication lag (seconds) │  > 5s    │  > 30s
 Replication lag (bytes)   │  > 50MB  │  > 500MB
 Replication slot lag      │  > 1GB   │  > 10GB
 Inactive replication slot │  > 1hr   │  > 6hr
 WAL directory size        │  > 5GB   │  > 20GB
 pg_is_in_recovery()       │  -       │  changed unexpectedly
```

---

## 🧪 Testing Your HA Setup

```bash
# ─── Test 1: Kill the primary and verify automatic failover ───
# On primary node:
sudo systemctl stop postgresql
# Watch Patroni logs on standby nodes:
# Should see: "promoted self to leader"
# Check: patronictl list → new leader elected

# ─── Test 2: Network partition (simulate split-brain) ───
# Block traffic from primary to etcd:
sudo iptables -A OUTPUT -p tcp --dport 2379 -j DROP
# Primary should step down (can't reach DCS)
# A standby should be promoted

# ─── Test 3: Switchover (planned maintenance) ───
patronictl switchover --master node1 --candidate node2
# Verify zero data loss:
# Insert a row on primary JUST before switchover
# Verify it exists on the new primary after switchover

# ─── Test 4: Rebuild a failed node ───
patronictl reinit my-cluster node1
# Node1 should rebuild from the new primary and rejoin as standby
```

---

## 🎓 Key Takeaways

```
┌──────────────────────────────────────────────────────────────┐
│  PostgreSQL Replication & HA — The Cheat Sheet                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  📡 Streaming Replication = byte-level WAL copy (built-in)   │
│  🔀 Logical Replication = row-level, selective, cross-version│
│  ⚡ Sync replication = zero data loss, slower writes          │
│  💨 Async replication = fast writes, ~1s data loss risk       │
│  🤖 Patroni = automated failover in 15-30 seconds            │
│  🔌 PgBouncer = connection pooling (500 clients → 50 conns)  │
│  🔄 HAProxy = route writes to primary, reads to replicas     │
│  🛡️ Replication slots = prevent WAL deletion for slow replicas│
│  📊 Monitor lag, slot size, and WAL directory constantly      │
│  🧪 Test failover regularly — untested HA is not HA          │
│                                                              │
│  Production minimum: 3 nodes + Patroni + etcd + HAProxy     │
│  + PgBouncer. Everything else is an optimization.            │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [2E.7 — PostgreSQL Administration](./07-PG-Admin.md) 🔴
> *Roles, Row-Level Security, pg_dump/pg_restore, Upgrade strategies, and monitoring your PostgreSQL fleet.*
