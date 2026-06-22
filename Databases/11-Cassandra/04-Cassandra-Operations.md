# 3D.4 — Cassandra Operations & Tuning 🔴

> **"Building a Cassandra cluster is easy. Keeping it healthy at scale while your CEO sleeps soundly? That's the real skill."**

---

## 📌 What You'll Master in This Chapter

- **nodetool** — the Swiss Army knife of Cassandra administration
- **Cluster Operations** — adding/removing nodes, decommissioning, bootstrapping
- **Repair** — why it's mandatory and how to do it right
- **Compaction Tuning** — choosing and configuring strategies
- **Consistency Level Selection** — production trade-offs
- **Monitoring & Alerting** — what metrics to watch and what they mean
- **Performance Tuning** — JVM, OS, disk, and CQL-level optimizations
- **Backup & Restore** — snapshot strategies for disaster recovery
- **Common Production Issues** — and how to diagnose them
- **Security** — authentication, authorization, encryption

---

## 🔥 1. nodetool — Your Primary Ops Tool

`nodetool` is the command-line admin tool that ships with Cassandra. It communicates with the JMX interface of a running Cassandra node.

### Cluster Status & Health

```bash
# THE command you'll run 100 times a day
nodetool status
# Output:
# Datacenter: us-east
# ===============
# Status=Up/Down  State=Normal/Leaving/Joining/Moving
# --  Address        Load       Tokens  Owns    Host ID                               Rack
# UN  10.0.1.1       256.5 GiB  256     33.3%   a1b2c3d4-...                          rack1
# UN  10.0.1.2       248.1 GiB  256     33.2%   e5f6g7h8-...                          rack2
# DN  10.0.1.3       252.3 GiB  256     33.5%   i9j0k1l2-...                          rack3
#  ↑                                              
#  U = Up, D = Down                               
#  N = Normal, L = Leaving, J = Joining, M = Moving

# Quick status flags:
# UN = Up Normal ✅ (healthy)
# DN = Down Normal ❌ (node is down!)
# UJ = Up Joining (new node bootstrapping)
# UL = Up Leaving (node decommissioning)

# Cluster info
nodetool info                    # Node-specific info (load, uptime, heap)
nodetool describecluster         # Cluster name, snitch, schema versions
nodetool ring                    # Token ring details
nodetool gossipinfo              # Gossip state of all nodes
```

### Data & Table Operations

```bash
# Table statistics (critical for performance analysis)
nodetool tablestats <keyspace>.<table>
# Shows:
# - Read/write latency (p50, p99)
# - SSTable count
# - Partition size (min, max, avg)
# - Tombstone count
# - Bloom filter false positive ratio
# - Compaction pending

# Specific stats
nodetool tablehistograms <keyspace>.<table>    # Read/write latency distribution
nodetool toppartitions <keyspace>.<table> 1000  # Largest partitions (hot partitions!)

# Flush memtables to SSTables
nodetool flush <keyspace>                       # Flush all tables in keyspace
nodetool flush <keyspace> <table>               # Flush specific table

# Compact SSTables
nodetool compact <keyspace> <table>             # Force compaction
nodetool compactionstats                        # View running compactions

# Cleanup (remove data no longer owned after topology change)
nodetool cleanup <keyspace>                     # Run after adding/removing nodes

# Scrub (fix corrupt SSTables)
nodetool scrub <keyspace> <table>               # Re-validates SSTables
```

### Repair Operations

```bash
# Full repair (mandatory for data consistency!)
nodetool repair <keyspace>                      # Full repair
nodetool repair <keyspace> <table>              # Repair specific table
nodetool repair -pr <keyspace>                  # Primary range only (preferred!)
nodetool repair --full <keyspace>               # Full (non-incremental) repair

# Incremental repair (default in newer versions)
nodetool repair <keyspace>                      # Incremental by default in 4.x

# Check repair status
nodetool netstats                               # Active streams/repairs
nodetool compactionstats                        # Repair uses compaction
```

### Node Lifecycle

```bash
# Decommission a node (graceful removal)
nodetool decommission
# → Streams data to remaining nodes
# → Safe! No data loss
# → Wait for completion before shutting down

# Remove a dead node (node is already dead, can't decommission)
nodetool removenode <host-id>
# → Tells cluster to forget this node
# → Redistributes data ownership
# → Use ONLY when node cannot be brought back

# Drain (prepare for shutdown)
nodetool drain
# → Flushes memtables
# → Stops accepting new connections
# → Run BEFORE shutting down Cassandra

# Move (change token assignment — rarely used with vnodes)
nodetool move <new-token>

# Assassinate (forcefully remove a completely dead node)
nodetool assassinate <ip-address>
# → Nuclear option — use only when removenode hangs
```

### nodetool Cheat Sheet

```
┌──────────────────────────────────────────────────────────────┐
│  NODETOOL COMMAND CATEGORIES                                  │
│                                                                │
│  CLUSTER HEALTH:                                              │
│  status | info | ring | describecluster | gossipinfo          │
│                                                                │
│  TABLE DIAGNOSTICS:                                           │
│  tablestats | tablehistograms | toppartitions | cfstats       │
│                                                                │
│  DATA MANAGEMENT:                                             │
│  flush | compact | scrub | cleanup | upgradesstables         │
│                                                                │
│  REPAIR & CONSISTENCY:                                        │
│  repair | verify | statusbackup                               │
│                                                                │
│  NODE LIFECYCLE:                                              │
│  decommission | removenode | drain | assassinate | bootstrap  │
│                                                                │
│  PERFORMANCE:                                                  │
│  tpstats | proxyhistograms | gcstats | gettimeout            │
│                                                                │
│  THREAD POOL STATS:                                           │
│  tpstats    ← Shows pending/blocked tasks (very important!)  │
│                                                                │
│  STREAMING:                                                    │
│  netstats    ← Active data streaming (repair, bootstrap)     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔥 2. Cluster Operations — Adding and Removing Nodes

### Adding a New Node (Bootstrapping)

```
┌─────────────────────────────────────────────────────────────────┐
│                ADDING A NEW NODE                                 │
│                                                                   │
│  1. Install Cassandra on new machine                            │
│  2. Configure cassandra.yaml:                                   │
│     - cluster_name: same as existing cluster                    │
│     - seeds: 2-3 existing nodes (NOT the new node itself)       │
│     - listen_address: new node's IP                             │
│     - endpoint_snitch: same as cluster                          │
│  3. Start Cassandra → node auto-joins!                          │
│                                                                   │
│  What happens:                                                   │
│  ┌──────────────────────────────────────────┐                   │
│  │ 1. New node contacts seed nodes          │                   │
│  │ 2. Gossip: learns cluster topology       │                   │
│  │ 3. Gets assigned token ranges (vnodes)   │                   │
│  │ 4. Streams data for its token ranges     │  ← Can take hours!│
│  │ 5. Status: UJ (Up Joining)               │                   │
│  │ 6. Once done: UN (Up Normal)             │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                   │
│  After bootstrap completes:                                     │
│  → Run nodetool cleanup on ALL other nodes                      │
│  → This removes data they no longer own                         │
│  → Run cleanup one node at a time to avoid overload             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Removing a Node

```
GRACEFUL REMOVAL (node is alive):
─────────────────────────────────
1. nodetool decommission          ← Streams data away, then leaves
2. Wait for completion
3. Shut down Cassandra
4. Remove from seed list

FORCED REMOVAL (node is dead/unrecoverable):
───────────────────────────────────────────
1. nodetool removenode <host-id>  ← Tells cluster to reassign data
2. Wait for data rebuild
3. Run repair on affected keyspaces

⚠️ NEVER just shut down a node and walk away!
   The cluster will hold hints for 3 hours, then data is lost
   on that node's replicas.
```

### Scaling Guidelines

```
┌─────────────────────────────────────────────────────────────────┐
│  SCALING BEST PRACTICES:                                         │
│                                                                   │
│  ✅ Add nodes in increments that maintain balance                │
│     (e.g., 3 → 6 → 9, not 3 → 4 → 5)                          │
│  ✅ Double capacity by doubling nodes                            │
│  ✅ Keep same number of nodes per rack/DC                       │
│  ✅ Run cleanup after adding nodes                              │
│  ✅ Monitor during bootstrap — streaming is I/O intensive!      │
│  ❌ Never add more than one node at a time                      │
│  ❌ Never add a node while repair or compaction is running      │
│                                                                   │
│  CAPACITY PLANNING:                                              │
│  Rule of thumb:                                                  │
│  Data per node = Total data × RF / Number of nodes              │
│  Keep disk utilization < 50% (compaction needs temp space)       │
│  Target: < 1 TB of data per node                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔥 3. Repair — The Most Important Operational Task

Repair is **mandatory** in Cassandra. Without regular repair, replicas drift apart, deleted data resurrects, and chaos ensues.

### Why Repair is Mandatory

```
┌─────────────────────────────────────────────────────────────────┐
│                WHY REPAIR IS NON-NEGOTIABLE                      │
│                                                                   │
│  1. HINTED HANDOFF has a window (default: 3 hours)              │
│     If a node is down longer → writes are lost on that replica  │
│     → Repair syncs the missing data                             │
│                                                                   │
│  2. TOMBSTONE GARBAGE COLLECTION (gc_grace_seconds = 10 days)   │
│     If you don't repair within gc_grace_seconds:                │
│     → Old tombstones get removed on some replicas               │
│     → But the data they "deleted" still exists on other replicas│
│     → Result: DELETED DATA COMES BACK (zombie data!) 🧟        │
│                                                                   │
│  3. READ REPAIR only fixes data you actually read               │
│     → Unread data stays inconsistent forever                    │
│     → Repair fixes ALL data, even unread                        │
│                                                                   │
│  RULE: Repair frequency < gc_grace_seconds (10 days default)    │
│  RECOMMENDATION: Repair every 7 days                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Types of Repair

| Type | Command | Description | When to Use |
|------|---------|-------------|-------------|
| **Full repair** | `nodetool repair --full` | Compares ALL data between replicas | After major incidents, initial setup |
| **Incremental repair** | `nodetool repair` | Only repairs data changed since last repair | Regular maintenance (default in 4.x) |
| **Primary range** | `nodetool repair -pr` | Only repairs ranges this node is primary for | Run on EACH node (avoids redundant work) |
| **Subrange repair** | `nodetool repair -st <start> -et <end>` | Repairs a specific token range | When you need fine-grained control |

### Repair Best Practices

```bash
# ✅ Recommended repair approach:
# Run primary-range repair on EACH node, one at a time

# On node 1:
nodetool repair -pr my_keyspace

# Wait for completion, then on node 2:
nodetool repair -pr my_keyspace

# ... and so on for each node

# ✅ Better: Use Reaper (automated repair scheduling)
# https://cassandra-reaper.io/
# Reaper manages repair across the cluster, handles scheduling,
# and prevents overloading nodes with concurrent repairs.
```

### Reaper — Automated Repair Management

```
┌─────────────────────────────────────────────────────────────────┐
│  CASSANDRA REAPER (recommended for production)                   │
│                                                                   │
│  Instead of running nodetool repair manually:                   │
│                                                                   │
│  1. Deploy Reaper alongside your cluster                        │
│  2. Register your cluster with Reaper                           │
│  3. Create a repair schedule:                                   │
│     - Keyspace: my_keyspace                                     │
│     - Intensity: 0.5 (use 50% of available resources)           │
│     - Schedule: every 7 days                                    │
│     - Segment count: auto                                       │
│                                                                   │
│  Reaper handles:                                                 │
│  ✅ Scheduling repairs across nodes                             │
│  ✅ Parallelism control (don't overload the cluster)           │
│  ✅ Resume after failure                                        │
│  ✅ Web UI for monitoring                                       │
│  ✅ REST API for automation                                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔥 4. Compaction Tuning

### Diagnosing Compaction Issues

```bash
# Check compaction status
nodetool compactionstats
# Output:
# pending tasks: 42              ← Should be low (< 50)
# id     compaction type    keyspace  table   completed   total    unit    progress
# abc    COMPACTION         app       msgs    1.5 GiB     3.0 GiB bytes   50.00%

# Check SSTable count per table
nodetool tablestats my_keyspace.my_table | grep "SSTable count"
# SSTable count: 35              ← Watch this number

# If SSTable count keeps growing → compaction can't keep up!
```

### Choosing the Right Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│  DECISION TREE FOR COMPACTION STRATEGY:                          │
│                                                                   │
│  Is your data TIME-SERIES with TTL?                              │
│  ├── YES → TimeWindowCompactionStrategy (TWCS) ✅               │
│  └── NO ↓                                                        │
│                                                                   │
│  Is your workload mostly READS with frequent updates?            │
│  ├── YES → LeveledCompactionStrategy (LCS) ✅                  │
│  └── NO ↓                                                        │
│                                                                   │
│  Is your workload mostly WRITES with few reads?                  │
│  ├── YES → SizeTieredCompactionStrategy (STCS) ✅              │
│  └── NO ↓                                                        │
│                                                                   │
│  Mixed workload?                                                 │
│  └── Try UnifiedCompactionStrategy (UCS, Cassandra 5.0+)       │
│      or start with STCS and monitor                             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Configuring Compaction

```sql
-- STCS (Size-Tiered) — default
ALTER TABLE my_table WITH compaction = {
    'class': 'SizeTieredCompactionStrategy',
    'min_threshold': 4,           -- Min SSTables to trigger compaction
    'max_threshold': 32,          -- Max SSTables to compact at once
    'min_sstable_size': 52428800  -- 50MB — ignore smaller SSTables
};

-- LCS (Leveled) — for read-heavy workloads
ALTER TABLE my_table WITH compaction = {
    'class': 'LeveledCompactionStrategy',
    'sstable_size_in_mb': 160     -- Target SSTable size per level
};

-- TWCS (Time Window) — for time-series data
ALTER TABLE my_table WITH compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_size': 1,
    'compaction_window_unit': 'DAYS'
};
-- Compacts SSTables within the same day together
-- NEVER compacts across windows
-- Perfect when combined with TTL
```

---

## 🔥 5. Performance Tuning

### JVM Tuning (cassandra-env.sh)

```bash
# Heap Size — THE most critical JVM setting
# Rule of thumb: MAX_HEAP = min(50% RAM, 8GB)
# More heap is NOT always better! (GC pauses!)

# For machines with 32GB RAM:
-Xms8G          # Initial heap
-Xmx8G          # Max heap (same as initial — avoid resizing)
-Xmn2G          # Young generation (25% of heap)

# GC Settings (Cassandra 4.x uses G1GC by default)
-XX:+UseG1GC
-XX:MaxGCPauseMillis=500
-XX:G1RSetUpdatingPauseTimePercent=5
-XX:InitiatingHeapOccupancyPercent=70

# Key GC rules:
# ✅ Keep heap ≤ 8GB (avoids long GC pauses)
# ✅ Use G1GC (default in modern Cassandra)
# ✅ Monitor GC pauses — p99 should be < 500ms
# ✅ Use off-heap for caches and memtables
```

### cassandra.yaml — Key Settings

```yaml
# ═══════════════════════════════════════════════
# MEMORY SETTINGS
# ═══════════════════════════════════════════════

# Memtable allocation type (off-heap reduces GC pressure)
memtable_allocation_type: offheap_objects     # or: heap_buffers, offheap_buffers

# How much memory for memtables (fraction of heap)
memtable_heap_space_in_mb: 2048
memtable_offheap_space_in_mb: 2048

# When to flush memtables (as fraction of total memtable space)
memtable_cleanup_threshold: 0.11               # Flush when 11% of space used

# ═══════════════════════════════════════════════
# DISK SETTINGS
# ═══════════════════════════════════════════════

# Commit log location (separate disk = huge performance win!)
commitlog_directory: /var/lib/cassandra/commitlog

# Data directory (multiple dirs = Cassandra stripes across them)
data_file_directories:
    - /data1/cassandra/data
    - /data2/cassandra/data          # JBOD (Just a Bunch Of Disks)

# Commit log sync (batch = fastest, periodic = more durable)
commitlog_sync: periodic
commitlog_sync_period_in_ms: 10000   # Flush every 10 seconds

# ═══════════════════════════════════════════════
# CONCURRENCY & THROUGHPUT
# ═══════════════════════════════════════════════

# Concurrent reads/writes
concurrent_reads: 32                  # Default: 32 (2× CPU cores)
concurrent_writes: 32                 # Default: 32
concurrent_compactors: 2              # Compaction threads (1 per spinning disk)

# Batch size warning thresholds
batch_size_warn_threshold_in_kb: 5    # Warn if batch > 5KB
batch_size_fail_threshold_in_kb: 50   # Fail if batch > 50KB

# ═══════════════════════════════════════════════
# CACHING
# ═══════════════════════════════════════════════

# Key cache: partition key → SSTable offset (almost always helpful)
key_cache_size_in_mb: 100             # Or leave empty for auto (5% of heap)
key_cache_save_period: 14400          # Save every 4 hours

# Row cache: full rows in memory (use sparingly!)
row_cache_size_in_mb: 0               # Disabled by default (usually leave it)
# Row cache is rarely beneficial — OS page cache is usually better

# ═══════════════════════════════════════════════
# TIMEOUTS
# ═══════════════════════════════════════════════

read_request_timeout_in_ms: 5000      # 5 seconds
write_request_timeout_in_ms: 2000     # 2 seconds
range_request_timeout_in_ms: 10000    # 10 seconds (for range scans)
cas_contention_timeout_in_ms: 1000    # LWT contention timeout
```

### OS-Level Tuning

```bash
# ═══════════════════════════════════════════════
# LINUX OS TUNING FOR CASSANDRA
# ═══════════════════════════════════════════════

# 1. Disable swap (CRITICAL!)
sudo swapoff -a
# Or set vm.swappiness = 1
echo "vm.swappiness = 1" >> /etc/sysctl.conf

# 2. Set maximum file descriptors
# /etc/security/limits.conf:
cassandra  -  nofile  100000
cassandra  -  nproc   32768
cassandra  -  memlock unlimited

# 3. Disable zone reclaim for NUMA
echo 0 > /proc/sys/vm/zone_reclaim_mode

# 4. Set read-ahead to 8KB for SSDs
blockdev --setra 8 /dev/sda

# 5. Use deadline or noop I/O scheduler for SSDs
echo noop > /sys/block/sda/queue/scheduler

# 6. Disable transparent huge pages (THP)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 7. Network tuning
echo "net.core.rmem_max = 16777216" >> /etc/sysctl.conf
echo "net.core.wmem_max = 16777216" >> /etc/sysctl.conf
echo "net.ipv4.tcp_rmem = 4096 87380 16777216" >> /etc/sysctl.conf
echo "net.ipv4.tcp_wmem = 4096 65536 16777216" >> /etc/sysctl.conf
```

### Hardware Recommendations

```
┌─────────────────────────────────────────────────────────────────┐
│  HARDWARE GUIDELINES PER NODE:                                   │
│                                                                   │
│  CPU:     8-16 cores (more for compaction-heavy workloads)      │
│  RAM:     32 GB (16 GB minimum, 64 GB for large datasets)      │
│  Heap:    8 GB max (regardless of RAM!)                         │
│  Disk:    SSDs strongly recommended (NVMe ideal)                │
│           1-5 TB per node                                        │
│           Keep utilization < 50% (compaction headroom!)         │
│  Network: 10 Gbps (1 Gbps minimum)                             │
│                                                                   │
│  DISK LAYOUT:                                                    │
│  ┌─────────────┬────────────────────────────┐                   │
│  │ Device      │ Use                         │                   │
│  ├─────────────┼────────────────────────────┤                   │
│  │ SSD/NVMe 1  │ Commit log (dedicated!)    │                   │
│  │ SSD/NVMe 2  │ Data directory             │                   │
│  │ SSD/NVMe 3  │ Data directory (JBOD)      │                   │
│  └─────────────┴────────────────────────────┘                   │
│                                                                   │
│  ⚠️ NEVER use RAID for Cassandra data directories!             │
│  Cassandra handles replication itself. RAID adds overhead.      │
│  Use JBOD (Just a Bunch Of Disks) instead.                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔥 6. Monitoring & Alerting

### Key Metrics to Monitor

```
┌──────────────────┬─────────────────────────┬──────────────────────┐
│ Metric           │ What It Tells You       │ Alert Threshold      │
├──────────────────┼─────────────────────────┼──────────────────────┤
│ Read Latency p99 │ Worst-case read speed   │ > 100ms              │
│ Write Latency p99│ Worst-case write speed  │ > 50ms               │
│ SSTable Count    │ Compaction keeping up?   │ > 30 per table       │
│ Pending Compact. │ Compaction backlog       │ > 50 tasks           │
│ Tombstone Count  │ Delete overhead          │ > 10K per read       │
│ GC Pause Time    │ JVM garbage collection  │ > 500ms              │
│ Heap Usage       │ Memory pressure          │ > 75% of max         │
│ Disk Usage       │ Space remaining          │ > 50% capacity       │
│ Dropped Messages │ Overloaded node          │ > 0 (any drops!)     │
│ Thread Pool Pend.│ Blocked operations       │ > 0 for key pools    │
│ Bloom Filter FP  │ Index efficiency         │ > 3%                 │
│ Key Cache Hit    │ Cache effectiveness      │ < 85%                │
│ Connection Count │ Client pressure          │ > 80% max_connections│
│ Hints Pending    │ Down nodes missing writes│ > 0 for > 3 hours    │
│ Repair Status    │ Data consistency         │ > 10 days since last │
└──────────────────┴─────────────────────────┴──────────────────────┘
```

### Monitoring Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                RECOMMENDED MONITORING STACK                      │
│                                                                   │
│  Option 1: Prometheus + Grafana (Most popular)                  │
│  ┌──────────┐    ┌────────────┐    ┌─────────┐                 │
│  │Cassandra │───▶│ Prometheus │───▶│ Grafana  │                 │
│  │(JMX/     │    │  (scrape)  │    │(dashbrd) │                 │
│  │ Metrics) │    └────────────┘    └─────────┘                 │
│  └──────────┘                                                    │
│  Use: metrics-collector for Cassandra (JMX → Prometheus)        │
│  Dashboard: cassandra-exporter or mcac (Metrics Collector)      │
│                                                                   │
│  Option 2: Datadog / New Relic (SaaS)                           │
│  ┌──────────┐    ┌────────────┐                                 │
│  │Cassandra │───▶│  Datadog   │  ← Zero-config with agent     │
│  └──────────┘    └────────────┘                                 │
│                                                                   │
│  Option 3: DataStax MCAC (Metrics Collector for Apache Cassandra)│
│  Purpose-built for Cassandra, integrates with Prometheus/Grafana│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Using nodetool for Quick Diagnostics

```bash
# Thread pool stats — see if anything is blocked
nodetool tpstats
# Look for:
# - "Pending" > 0 in MutationStage, ReadStage → overloaded!
# - "Blocked" > 0 anywhere → serious problem!
# - "All time blocked" increasing → capacity issue

# Proxy histograms — client-facing latency
nodetool proxyhistograms
# Shows read/write/range latency percentiles
# p99 read > 100ms → investigate

# GC stats
nodetool gcstats
# Max GC pause > 500ms → tune JVM or reduce heap

# Check for dropped messages
nodetool tpstats | grep -i dropped
# ANY dropped messages = overloaded node
# MUTATION dropped = writes failed!
# READ dropped = reads timed out!
```

---

## 🔥 7. Backup & Restore

### Snapshot-Based Backups

```bash
# Create a snapshot (instantaneous — uses hard links)
nodetool snapshot -t backup_2024_01_15 my_keyspace
# Creates: /data/cassandra/data/my_keyspace/my_table-<id>/snapshots/backup_2024_01_15/

# List snapshots
nodetool listsnapshots

# Clear a snapshot (delete backup files)
nodetool clearsnapshot -t backup_2024_01_15 my_keyspace

# Clear ALL snapshots
nodetool clearsnapshot --all
```

### Full Backup Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│  BACKUP STRATEGY FOR PRODUCTION:                                 │
│                                                                   │
│  1. SNAPSHOT (point-in-time, instant)                           │
│     nodetool snapshot on each node                              │
│     Copy snapshot files to remote storage (S3, GCS)             │
│     Frequency: Daily                                            │
│                                                                   │
│  2. INCREMENTAL BACKUP (continuous)                             │
│     Set incremental_backups: true in cassandra.yaml             │
│     Each flushed SSTable is hard-linked to /backups/ dir        │
│     Ship these to remote storage continuously                   │
│     Frequency: Continuous                                       │
│                                                                   │
│  3. COMMIT LOG ARCHIVING (for point-in-time recovery)          │
│     Configure commitlog_archiving.properties                    │
│     Archive commit logs to remote storage                       │
│     Replay on restore for up-to-the-second recovery            │
│                                                                   │
│  RESTORE PROCESS:                                                │
│  1. Stop Cassandra                                              │
│  2. Clear data directories                                      │
│  3. Copy snapshot SSTables to data directories                  │
│  4. Start Cassandra                                             │
│  5. Run nodetool repair                                         │
│                                                                   │
│  💡 Tools: Medusa (open-source Cassandra backup tool)           │
│     Handles backup to S3/GCS, orchestrates multi-node backups   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔥 8. Security

### Authentication & Authorization

```yaml
# cassandra.yaml — enable authentication
authenticator: PasswordAuthenticator        # Default: AllowAllAuthenticator
authorizer: CassandraAuthorizer             # Default: AllowAllAuthorizer
role_manager: CassandraRoleManager

# Default superuser: cassandra/cassandra
# ⚠️ CHANGE THIS IMMEDIATELY in production!
```

```sql
-- Create a new superuser
CREATE ROLE admin WITH PASSWORD = 'StrongP@ssw0rd!'
AND SUPERUSER = true AND LOGIN = true;

-- Create application role
CREATE ROLE app_user WITH PASSWORD = 'AppP@ss123!'
AND LOGIN = true;

-- Grant permissions
GRANT SELECT ON KEYSPACE my_app TO app_user;
GRANT MODIFY ON KEYSPACE my_app TO app_user;

-- Granular permissions
GRANT SELECT ON TABLE my_app.users TO app_user;
GRANT MODIFY ON TABLE my_app.users TO app_user;

-- Revoke permissions
REVOKE MODIFY ON TABLE my_app.users FROM app_user;

-- List permissions
LIST ALL PERMISSIONS OF app_user;

-- Permission types:
-- ALL | ALTER | AUTHORIZE | CREATE | DESCRIBE | DROP |
-- EXECUTE | MODIFY | SELECT
```

### Encryption

```yaml
# ═══════════════════════════════════════════════
# ENCRYPTION IN TRANSIT (TLS/SSL)
# ═══════════════════════════════════════════════

# Node-to-node encryption (internode)
server_encryption_options:
    internode_encryption: all            # none | rack | dc | all
    keystore: /etc/cassandra/keystore.jks
    keystore_password: keystorepass
    truststore: /etc/cassandra/truststore.jks
    truststore_password: truststorepass
    protocol: TLS
    cipher_suites: [TLS_RSA_WITH_AES_256_CBC_SHA]

# Client-to-node encryption
client_encryption_options:
    enabled: true
    optional: false                       # Force encryption
    keystore: /etc/cassandra/keystore.jks
    keystore_password: keystorepass
    truststore: /etc/cassandra/truststore.jks
    truststore_password: truststorepass

# ═══════════════════════════════════════════════
# ENCRYPTION AT REST (Transparent Data Encryption)
# ═══════════════════════════════════════════════

# Available in DataStax Enterprise (DSE) and newer Apache Cassandra
# Encrypts SSTables, commit logs on disk
transparent_data_encryption_options:
    enabled: true
    cipher: AES/CBC/PKCS5Padding
    key_alias: my_key
    key_provider:
        - class_name: org.apache.cassandra.security.JKSKeyProvider
          parameters:
              - keystore: /etc/cassandra/encryption-keystore.jks
                keystore_password: keystorepass
```

---

## 🔥 9. Common Production Issues & Solutions

### Issue 1: High Read Latency

```
SYMPTOMS: p99 read latency > 100ms, users complaining about slow responses

DIAGNOSIS STEPS:
┌─────────────────────────────────────────────────────────────────┐
│ 1. Check SSTable count:                                         │
│    nodetool tablestats keyspace.table | grep "SSTable count"    │
│    → High count? Compaction can't keep up.                      │
│    → FIX: Tune compaction, switch strategy, or add nodes        │
│                                                                   │
│ 2. Check partition size:                                         │
│    nodetool tablehistograms keyspace.table                       │
│    → Large partitions? > 100MB                                  │
│    → FIX: Redesign data model with bucketing                    │
│                                                                   │
│ 3. Check tombstones:                                            │
│    nodetool tablestats keyspace.table | grep "tombstone"        │
│    → High tombstone count per read?                             │
│    → FIX: Reduce deletes, use TTL, adjust gc_grace_seconds      │
│                                                                   │
│ 4. Check GC pauses:                                             │
│    nodetool gcstats                                             │
│    → GC pauses > 500ms?                                         │
│    → FIX: Reduce heap, use off-heap memtables, tune G1GC       │
│                                                                   │
│ 5. Check bloom filter false positives:                          │
│    nodetool tablestats keyspace.table | grep "Bloom"            │
│    → FP rate > 3%?                                              │
│    → FIX: ALTER TABLE ... WITH bloom_filter_fp_chance = 0.01;  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Issue 2: Disk Full

```
SYMPTOMS: Cassandra crashes or rejects writes

IMMEDIATE ACTIONS:
1. Remove old snapshots: nodetool clearsnapshot --all
2. Remove old backup hardlinks from /backups/ directory
3. Force compaction (temporarily uses MORE space!)
4. Lower gc_grace_seconds to reclaim tombstone space faster
   (⚠️ only if repair is up-to-date!)

PREVENTION:
→ Alert at 50% disk usage (compaction needs headroom!)
→ Set disk_failure_policy: stop (in cassandra.yaml)
→ Never let a node exceed 70% disk usage
```

### Issue 3: Dropped Messages

```
SYMPTOMS: nodetool tpstats shows dropped mutations/reads

ROOT CAUSES:
→ Node is overloaded (too much traffic for hardware)
→ Compaction backlog consuming I/O
→ GC pauses causing timeouts
→ Network issues between nodes

FIXES:
1. Add capacity (more nodes)
2. Increase timeout thresholds (temporary)
3. Reduce query volume or add caching
4. Tune compaction to reduce I/O impact
5. Increase concurrent_reads/writes (if CPU allows)
```

### Issue 4: Zombie Data (Resurrected Deletes)

```
SYMPTOMS: Data you deleted reappears after some time

CAUSE: Repair not running frequently enough
→ Tombstone was GC'd on some replicas (past gc_grace_seconds)
→ But the original data still exists on another replica
→ Read repair or anti-entropy copies the old data back!

FIX:
1. Run full repair on the affected keyspace
2. Schedule repairs more frequently (< gc_grace_seconds)
3. Consider increasing gc_grace_seconds if needed
4. Use Reaper for automated repair scheduling
```

---

## 🔥 10. Production Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│           CASSANDRA PRODUCTION READINESS CHECKLIST               │
│                                                                   │
│  INFRASTRUCTURE:                                                 │
│  □ SSDs for data + separate SSD for commit log                  │
│  □ min 3 nodes per DC (for RF=3 + QUORUM)                      │
│  □ Nodes sized: 8+ cores, 32GB RAM, 8GB heap max              │
│  □ Disk < 50% utilized                                         │
│  □ 10Gbps network between nodes                                │
│                                                                   │
│  CONFIGURATION:                                                  │
│  □ NetworkTopologyStrategy (never SimpleStrategy)               │
│  □ GossipingPropertyFileSnitch (or cloud-specific snitch)       │
│  □ Swap disabled (vm.swappiness = 1)                            │
│  □ Transparent Huge Pages disabled                              │
│  □ File descriptor limit raised (100K+)                         │
│  □ Commit log on separate disk                                  │
│  □ incremental_backups: true                                    │
│                                                                   │
│  SECURITY:                                                       │
│  □ Authentication enabled (PasswordAuthenticator)               │
│  □ Default cassandra/cassandra superuser changed                │
│  □ Client-to-node TLS enabled                                   │
│  □ Node-to-node TLS enabled                                     │
│  □ Least-privilege roles for applications                       │
│  □ JMX authentication enabled                                   │
│                                                                   │
│  OPERATIONS:                                                     │
│  □ Repair scheduled (every 7 days, using Reaper)                │
│  □ Monitoring in place (Prometheus + Grafana)                   │
│  □ Alerts configured for key metrics                            │
│  □ Backup strategy tested (snapshot + offsite)                  │
│  □ Restore procedure tested and documented                     │
│  □ Runbook for common issues                                    │
│                                                                   │
│  DATA MODEL:                                                     │
│  □ Partition sizes validated (< 100MB)                          │
│  □ No ALLOW FILTERING in production queries                     │
│  □ No secondary indexes on high-cardinality columns             │
│  □ Compaction strategy matches workload                         │
│  □ TTL set for time-bounded data                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📋 Cassandra Operations Cheat Sheet

```
┌────────────────────────────────────────────────────────────────┐
│                DAILY OPERATIONS                                 │
│                                                                  │
│  Check health:     nodetool status                              │
│  Thread pools:     nodetool tpstats                             │
│  Table stats:      nodetool tablestats ks.table                 │
│  Compaction:       nodetool compactionstats                     │
│  GC health:        nodetool gcstats                             │
│  Latency:          nodetool proxyhistograms                     │
│  Hot partitions:   nodetool toppartitions ks.table 1000         │
│                                                                  │
│                WEEKLY OPERATIONS                                 │
│                                                                  │
│  Repair:           nodetool repair -pr <keyspace> (per node)    │
│  Snapshot:         nodetool snapshot -t <tag> <keyspace>        │
│  Check disk:       nodetool status (Load column)                │
│                                                                  │
│                EMERGENCY OPERATIONS                              │
│                                                                  │
│  Drain before stop: nodetool drain                              │
│  Force compaction:  nodetool compact ks table                   │
│  Remove dead node:  nodetool removenode <host-id>               │
│  Clear snapshots:   nodetool clearsnapshot --all                │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Chapter Summary — What You Now Know

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  ✅ nodetool is your primary admin tool (status, repair, flush) │
│  ✅ Adding nodes: configure & start → auto-bootstraps           │
│  ✅ Removing nodes: decommission (alive) or removenode (dead)   │
│  ✅ Repair is MANDATORY — run within gc_grace_seconds           │
│  ✅ Use Reaper for automated repair scheduling                  │
│  ✅ Compaction strategy must match workload (STCS/LCS/TWCS)    │
│  ✅ JVM heap: max 8GB, use G1GC, off-heap memtables            │
│  ✅ OS: disable swap, disable THP, raise file descriptors      │
│  ✅ Hardware: SSDs, separate commit log disk, JBOD (no RAID)   │
│  ✅ Monitor: latency, SSTables, tombstones, GC, dropped msgs   │
│  ✅ Backup: snapshots + incremental + commit log archiving      │
│  ✅ Security: auth + TLS + role-based access                    │
│  ✅ Disk: alert at 50%, never exceed 70%                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🗺️ Where You Are in the Cassandra Journey

```
  ✅ 3D.1 — Architecture & Ring Topology        (COMPLETED)
  ✅ 3D.2 — CQL: Cassandra Query Language        (COMPLETED)
  ✅ 3D.3 — Data Modeling (Query-First Design)   (COMPLETED)
  ✅ 3D.4 — Operations & Tuning                  (YOU ARE HERE)

  🎉 CONGRATULATIONS — You've completed the full Cassandra chapter!
  
  You now know enough to:
  → Design Cassandra data models for real-world applications
  → Write efficient CQL queries
  → Deploy, monitor, and operate a production cluster
  → Troubleshoot common performance issues
  → Implement security and backup strategies
```

---

> **"Operating Cassandra well isn't about knowing every setting — it's about knowing which 20 settings matter, monitoring the right 10 metrics, and running repair before you forget."** 🔧
