# 🏰 Chapter 3C.4 — Redis Cluster, Sentinel & Production Deployment

> **Level:** 🔴 Advanced | 🔥 High Demand
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 3C.1-3C.3 (all previous Redis chapters)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Design **Redis Cluster** topologies for horizontal scaling (sharding)
- Set up **Redis Sentinel** for automatic failover (high availability)
- Tune **RDB + AOF persistence** for production workloads
- Know the differences between **Cluster vs Sentinel vs Standalone**
- Handle **production incidents** — split-brain, failover, data recovery
- Deploy Redis with **battle-tested configurations** used by top companies

---

## 🧠 The Big Question

> *"I have a single Redis server. It works great. Why do I need anything more?"*

Because single-server Redis has THREE fatal weaknesses:

```
╔══════════════════════════════════════════════════════════════════╗
║  SINGLE REDIS SERVER — 3 FATAL WEAKNESSES                       ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1️⃣  SINGLE POINT OF FAILURE                                    ║
║     Server dies → ALL cached data lost → App crashes             ║
║     Solution: SENTINEL (automatic failover)                      ║
║                                                                  ║
║  2️⃣  MEMORY CEILING                                             ║
║     One server = max ~512 GB RAM                                 ║
║     Your data grows to 2 TB → won't fit!                        ║
║     Solution: CLUSTER (sharding across multiple nodes)           ║
║                                                                  ║
║  3️⃣  THROUGHPUT LIMIT                                           ║
║     One CPU core = ~100K-300K ops/sec (single-threaded)          ║
║     Need 1 million ops/sec? One server can't do it.             ║
║     Solution: CLUSTER (distribute load across nodes)             ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🛡️ PART 1: Redis Sentinel — High Availability

> **Sentinel = Automatic failover.** If your master dies, Sentinel promotes a replica to master — automatically, in seconds.

### Architecture

```
                    SENTINEL ARCHITECTURE

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  Sentinel 1  │  │  Sentinel 2  │  │  Sentinel 3  │
  │  (Monitor)   │  │  (Monitor)   │  │  (Monitor)   │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                 │
         │   Monitoring + Voting (Quorum)    │
         │                 │                 │
         ├─────────────────┼─────────────────┤
         │                 │                 │
         ▼                 ▼                 ▼
  ┌────────────────────────────────────────────────┐
  │                REDIS DATA NODES                 │
  │                                                 │
  │  ┌──────────────┐                              │
  │  │    MASTER     │ ← Writes go here             │
  │  │  Port: 6379   │                              │
  │  └──────┬───────┘                              │
  │         │ Replication (async)                   │
  │    ┌────┴────┐                                 │
  │    │         │                                  │
  │  ┌─▼────────┐  ┌───────────┐                   │
  │  │ REPLICA 1 │  │ REPLICA 2 │ ← Reads can go   │
  │  │ Port:6380 │  │ Port:6381 │   here too        │
  │  └──────────┘  └───────────┘                   │
  └────────────────────────────────────────────────┘

  Normal operation: Master handles writes, replicas sync
  Master dies: Sentinels vote → promote Replica 1 to Master!
```

### How Failover Works

```
FAILOVER SEQUENCE (Master Dies):

  Time 0:    Master is healthy ✅
  Time T:    Master crashes! 💀
  
  Time T+5s: Sentinel 1 can't reach Master
             → Marks master as SDOWN (Subjective DOWN)
             "I think master is down"
  
  Time T+5s: Sentinel 2 & 3 also can't reach Master
             → All 3 mark ODOWN (Objective DOWN)
             "We ALL agree master is down" (quorum reached)
  
  Time T+6s: Sentinel 1 elected as leader (Raft-like election)
             → Picks best replica (most data, lowest latency)
             → Runs REPLICAOF NO ONE on Replica 1
             → Replica 1 becomes NEW MASTER! 🎉
  
  Time T+7s: Sentinel reconfigures Replica 2:
             → REPLICAOF new-master-ip 6380
             → All clients notified of new master address
  
  Time T+??:  Old master comes back online
              → Sentinel demotes it to REPLICA of new master
              → It syncs from new master automatically

  Total downtime: ~5-15 seconds (configurable)
```

### Sentinel Configuration

```bash
# ═══════════════════════════════════════
#  sentinel.conf (for each sentinel instance)
# ═══════════════════════════════════════

# Monitor master with name "mymaster" at this address
# 2 = quorum (2 out of 3 sentinels must agree on failure)
sentinel monitor mymaster 192.168.1.10 6379 2

# How long before considering master DOWN (milliseconds)
sentinel down-after-milliseconds mymaster 5000    # 5 seconds

# Timeout for failover process
sentinel failover-timeout mymaster 60000          # 60 seconds

# How many replicas can sync simultaneously during failover
sentinel parallel-syncs mymaster 1

# Authentication
sentinel auth-pass mymaster YOUR_REDIS_PASSWORD

# Notification script (alert on failover):
sentinel notification-script mymaster /scripts/notify.sh

# ═══════════════════════════════════════
#  Start Sentinel
# ═══════════════════════════════════════

redis-sentinel /path/to/sentinel.conf
# or
redis-server /path/to/sentinel.conf --sentinel
```

### Connecting to Sentinel (Client-Side)

```python
# Python — Connect through Sentinel (auto-discovers master!)
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('sentinel-1.example.com', 26379),
    ('sentinel-2.example.com', 26379),
    ('sentinel-3.example.com', 26379),
], socket_timeout=0.5)

# Get master connection (auto-updates on failover!)
master = sentinel.master_for('mymaster', socket_timeout=0.5)
master.set('key', 'value')

# Get replica for reads (load balance reads!)
replica = sentinel.slave_for('mymaster', socket_timeout=0.5)
value = replica.get('key')

# If master fails → Sentinel updates client automatically!
# No code changes needed. No restarts. Just works.
```

```java
// Java (Jedis) — Sentinel connection
JedisSentinelPool pool = new JedisSentinelPool(
    "mymaster",
    new HashSet<>(Arrays.asList(
        "sentinel1:26379",
        "sentinel2:26379", 
        "sentinel3:26379"
    ))
);

try (Jedis jedis = pool.getResource()) {
    jedis.set("key", "value");
}
```

### Sentinel Best Practices

```
╔════════════════════════════════════════════════════════════════════╗
║  SENTINEL BEST PRACTICES                                          ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  1. Always use ODD number of Sentinels (3, 5, 7)                 ║
║     → Prevents split-brain during voting                          ║
║                                                                    ║
║  2. Deploy Sentinels on SEPARATE machines                         ║
║     → If they're on same machine as Redis → defeats the purpose  ║
║                                                                    ║
║  3. Set quorum = majority (3 sentinels → quorum = 2)             ║
║     → Too low = false positives → unnecessary failovers          ║
║     → Too high = slow detection                                   ║
║                                                                    ║
║  4. down-after-milliseconds: 5000ms (5s) is a good start        ║
║     → Lower = faster failover but risk of false positives        ║
║     → Higher = slower failover but more reliable detection       ║
║                                                                    ║
║  5. Always use Sentinel-aware clients                             ║
║     → Don't hardcode master IP! Let Sentinel manage it.          ║
║                                                                    ║
║  6. Monitor Sentinel itself!                                      ║
║     → SENTINEL CKQUORUM mymaster (check quorum health)           ║
║     → SENTINEL MASTER mymaster (check master status)             ║
║                                                                    ║
║  7. Test failovers regularly:                                     ║
║     SENTINEL FAILOVER mymaster (trigger manual failover)          ║
║     → Verify your app handles it gracefully                      ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## 🌐 PART 2: Redis Cluster — Horizontal Scaling

> **Redis Cluster = Sharding.** Data is automatically split across multiple master nodes. Each master handles a portion of the keyspace.

### Why Cluster?

```
Sentinel gives you:  HIGH AVAILABILITY (failover)
Cluster gives you:   HIGH AVAILABILITY + HORIZONTAL SCALING (sharding)

Sentinel:
  ┌────────────────────┐
  │      MASTER        │  ← ALL data here (limited by single node RAM)
  │   200 GB data      │
  │   100K ops/sec     │
  └────────────────────┘

Cluster:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Master 1 │  │ Master 2 │  │ Master 3 │
  │ 67 GB    │  │ 67 GB    │  │ 66 GB    │
  │ 33K ops  │  │ 33K ops  │  │ 33K ops  │
  └──────────┘  └──────────┘  └──────────┘
     Total: 200 GB data, 300K+ ops/sec (scales linearly!)
```

### How Redis Cluster Works — Hash Slots

```
Redis Cluster divides ALL keys into 16,384 HASH SLOTS:

  Slot Assignment:
  slot = CRC16(key) % 16384

  ┌─────────────────────────────────────────────────────────┐
  │                    16,384 Hash Slots                     │
  │                                                          │
  │  Node A          Node B          Node C                  │
  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │
  │  │ Slots       │ │ Slots       │ │ Slots       │       │
  │  │ 0 - 5460    │ │ 5461 - 10922│ │ 10923 -16383│       │
  │  │             │ │             │ │             │       │
  │  │ "user:1"    │ │ "user:2"    │ │ "user:3"    │       │
  │  │ CRC16 → 100│ │ CRC16→ 7000│ │ CRC16→12000 │       │
  │  │ Slot 100 ✓  │ │ Slot 7000 ✓ │ │ Slot 12000 ✓│       │
  │  └─────────────┘ └─────────────┘ └─────────────┘       │
  │                                                          │
  │  Each node has a REPLICA for high availability:          │
  │  Node A → Replica A'                                     │
  │  Node B → Replica B'                                     │
  │  Node C → Replica C'                                     │
  └─────────────────────────────────────────────────────────┘

  Key "user:1":
    CRC16("user:1") = 10778
    10778 % 16384 = 10778
    Slot 10778 → Node B (handles slots 5461-10922)
    → Command routed to Node B!
```

### Cluster Architecture

```
REDIS CLUSTER — PRODUCTION SETUP (Minimum 6 Nodes)

  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  ┌──────────┐      ┌──────────┐      ┌──────────┐           │
  │  │ Master 1 │      │ Master 2 │      │ Master 3 │           │
  │  │ Slots    │      │ Slots    │      │ Slots    │           │
  │  │ 0-5460   │      │ 5461-    │      │ 10923-   │           │
  │  │          │      │ 10922    │      │ 16383    │           │
  │  └────┬─────┘      └────┬─────┘      └────┬─────┘           │
  │       │ replication      │ replication      │ replication    │
  │  ┌────▼─────┐      ┌────▼─────┐      ┌────▼─────┐           │
  │  │Replica 1 │      │Replica 2 │      │Replica 3 │           │
  │  │(of M1)   │      │(of M2)   │      │(of M3)   │           │
  │  └──────────┘      └──────────┘      └──────────┘           │
  │                                                               │
  │  Every node knows about every other node (gossip protocol)   │
  │  Clients can connect to ANY node — auto-redirected           │
  │  If Master 2 dies → Replica 2 promoted automatically         │
  └──────────────────────────────────────────────────────────────┘
```

### Setting Up Redis Cluster

```bash
# ═══════════════════════════════════════
#  Option 1: Quick Setup (Development)
# ═══════════════════════════════════════

# Create 6 Redis instances (3 masters + 3 replicas)
mkdir cluster-test && cd cluster-test
mkdir 7000 7001 7002 7003 7004 7005

# redis.conf for each node (7000-7005):
cat > 7000/redis.conf << 'EOF'
port 7000
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 5000
appendonly yes
appendfilename "appendonly-7000.aof"
dbfilename dump-7000.rdb
EOF
# Repeat for ports 7001-7005...

# Start each node:
redis-server ./7000/redis.conf &
redis-server ./7001/redis.conf &
redis-server ./7002/redis.conf &
redis-server ./7003/redis.conf &
redis-server ./7004/redis.conf &
redis-server ./7005/redis.conf &

# Create the cluster:
redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1

# Output:
# >>> Performing hash slots allocation on 6 nodes...
# Master[0] -> Slots 0 - 5460
# Master[1] -> Slots 5461 - 10922
# Master[2] -> Slots 10923 - 16383
# >>> Can I set the above configuration? (type 'yes' to accept): yes

# ═══════════════════════════════════════
#  Option 2: Docker Compose (Recommended for Dev)
# ═══════════════════════════════════════

# docker-compose.yml
# version: '3'
# services:
#   redis-node-1:
#     image: redis:7
#     command: redis-server --port 7000 --cluster-enabled yes --cluster-config-file nodes.conf
#     ports: ["7000:7000"]
#   ... (repeat for 6 nodes)
```

### Cluster Commands

```bash
# ═══════════════════════════════════════
#  Connect to Cluster
# ═══════════════════════════════════════

redis-cli -c -p 7000              # -c = cluster mode (auto-redirect!)

# ═══════════════════════════════════════
#  Data Operations (with redirection)
# ═══════════════════════════════════════

SET user:1 "Alice"
# If user:1 maps to slot 10778 (on node 7001):
# → Redirected to 127.0.0.1:7001
# → OK

GET user:1
# → "Alice" (auto-redirected to correct node)

# ═══════════════════════════════════════
#  Cluster Management
# ═══════════════════════════════════════

CLUSTER INFO                       # Cluster state, slots, nodes
CLUSTER NODES                      # List all nodes with roles
CLUSTER SLOTS                      # Which node owns which slots
CLUSTER MYID                       # Current node's ID
CLUSTER KEYSLOT user:1             # → 10778 (which slot?)

# Check cluster health:
redis-cli --cluster check 127.0.0.1:7000

# Rebalance slots:
redis-cli --cluster rebalance 127.0.0.1:7000

# Add a new node:
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000

# Remove a node (migrate slots first!):
redis-cli --cluster reshard 127.0.0.1:7000   # Move slots away
redis-cli --cluster del-node 127.0.0.1:7000 <node-id>
```

### Hash Tags — Force Keys to Same Slot

```bash
# PROBLEM: Multi-key operations only work on SAME node!

MGET user:1 user:2 user:3
# ❌ ERROR: Keys may be on different nodes!

# SOLUTION: Hash Tags — only text inside {} is hashed

SET {user:42}:name "Alice"         # CRC16("user:42") → slot X
SET {user:42}:email "alice@dev.com"  # CRC16("user:42") → slot X (same!)
SET {user:42}:age 30               # CRC16("user:42") → slot X (same!)

MGET {user:42}:name {user:42}:email {user:42}:age
# ✅ Works! All keys in same slot because of {user:42}

# ═══════════════════════════════════════
#  Common Hash Tag Patterns
# ═══════════════════════════════════════

# User data together:
{user:1}:profile
{user:1}:sessions
{user:1}:preferences

# Order with its items:
{order:5042}:details
{order:5042}:items
{order:5042}:payment

# ⚠️ WARNING: Don't put ALL keys in same hash tag!
# {app}:user:1, {app}:user:2 → ALL on one node = hot spot!
# Use specific tags: {user:1}:xxx, {user:2}:xxx → distributed
```

### Cluster Limitations

```
╔════════════════════════════════════════════════════════════════════╗
║  REDIS CLUSTER LIMITATIONS                                        ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  1. Multi-key commands require SAME SLOT                          ║
║     MGET, MSET, SUNION, etc. → keys must be on same node         ║
║     Solution: Hash tags {user:42}:field1, {user:42}:field2       ║
║                                                                    ║
║  2. SELECT (multiple databases) NOT supported                     ║
║     Cluster only uses db0                                         ║
║                                                                    ║
║  3. Lua scripts — all keys must be on SAME node                  ║
║     Pass all keys as KEYS[] (not hardcoded)                      ║
║                                                                    ║
║  4. Transactions (MULTI/EXEC) — same slot only                   ║
║                                                                    ║
║  5. Pub/Sub — messages broadcast to ALL nodes                    ║
║     (works, but less efficient than standalone)                   ║
║                                                                    ║
║  6. Minimum 6 nodes for production                                ║
║     (3 masters + 3 replicas)                                      ║
║                                                                    ║
║  7. Node failures during resharding can cause brief downtime     ║
║                                                                    ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## 🆚 Sentinel vs Cluster — Which One?

```
╔═══════════════════════════════════════════════════════════════════════════╗
║  Feature              │  Sentinel              │  Cluster               ║
╠═══════════════════════════════════════════════════════════════════════════╣
║  Primary purpose      │  High Availability     │  HA + Scaling          ║
║  Data sharding        │  ❌ No (all on 1 node) │  ✅ Yes (hash slots)   ║
║  Max memory           │  Single node limit     │  Sum of all nodes      ║
║  Max throughput       │  Single node limit     │  Sum of all nodes      ║
║  Multi-key ops        │  ✅ All keys on 1 node │  ⚠️ Same slot only     ║
║  Complexity           │  🟡 Medium              │  🔴 Higher             ║
║  Minimum nodes        │  3 sentinel + 3 Redis  │  6 Redis (3M + 3R)    ║
║  Failover time        │  ~5-15 seconds         │  ~5-15 seconds        ║
║  SELECT (multi-db)    │  ✅ Supported           │  ❌ Only db0           ║
║  Pub/Sub efficiency   │  ✅ Good                │  🟡 Broadcasts all    ║
╠═══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║  📋 DECISION GUIDE:                                                      ║
║                                                                          ║
║  Data < 50 GB AND < 100K ops/sec:                                       ║
║    → SENTINEL (simpler, sufficient)                                      ║
║                                                                          ║
║  Data > 50 GB OR > 100K ops/sec OR need linear scaling:                 ║
║    → CLUSTER (sharding required)                                        ║
║                                                                          ║
║  Just caching (data loss acceptable):                                    ║
║    → SENTINEL (simpler) OR managed service (ElastiCache, Azure Cache)   ║
║                                                                          ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

---

## 📋 PART 3: Replication Deep Dive

### How Redis Replication Works

```
INITIAL SYNC (First Connection):

  ┌──────────┐                    ┌──────────┐
  │  MASTER  │                    │ REPLICA  │
  └────┬─────┘                    └────┬─────┘
       │   1. PSYNC ? -1               │
       │◄──────────────────────────────│  "I'm new, full sync please"
       │                               │
       │   2. FULLRESYNC <id> <offset> │
       │──────────────────────────────▶│  "OK, here's my ID"
       │                               │
       │   3. BGSAVE (create RDB)      │
       │   ...saving...                │
       │                               │
       │   4. Send RDB file            │
       │──────────────────────────────▶│  Replica loads RDB
       │                               │
       │   5. Send buffered commands   │
       │──────────────────────────────▶│  Catch up on writes during sync
       │                               │
       │   6. Continuous replication   │
       │──────────────────────────────▶│  Every write → replayed on replica
       │   (real-time, async)          │
       └──────────────────────────────┘

PARTIAL RESYNC (Reconnection after brief disconnect):

  Instead of full sync, replica says:
  "PSYNC <replication-id> <offset>"
  "I was at offset 12345, just send me what I missed"
  
  Master checks backlog buffer → sends only missing commands!
  Much faster than full resync.
```

### Replication Configuration

```bash
# ═══════════════════════════════════════
#  Master configuration (redis.conf)
# ═══════════════════════════════════════

# Backlog buffer for partial resync:
repl-backlog-size 64mb        # How much to buffer for reconnecting replicas
repl-backlog-ttl 3600         # Release backlog 1 hour after last replica disconnects

# Minimum replicas for write acceptance:
min-replicas-to-write 1       # Refuse writes if < 1 replica connected
min-replicas-max-lag 10       # Refuse writes if replica lag > 10 seconds
# This prevents data loss — won't accept writes with no replicas!

# ═══════════════════════════════════════
#  Replica configuration (redis.conf)
# ═══════════════════════════════════════

replicaof 192.168.1.10 6379   # Connect to master
replica-read-only yes          # ⭐ Replicas should be read-only!
replica-serve-stale-data yes   # Serve (possibly stale) data during sync

# Or dynamically:
redis-cli> REPLICAOF 192.168.1.10 6379    # Make this a replica
redis-cli> REPLICAOF NO ONE               # Promote to master!
```

### Read Replicas — Scale Reads

```
READ SCALING ARCHITECTURE:

  ┌──────────────┐
  │   MASTER     │ ← ALL writes go here
  │              │
  └───┬──────┬───┘
      │      │     Replication (async)
  ┌───▼──┐ ┌─▼────┐
  │Rep 1 │ │Rep 2 │ ← Reads distributed here
  └───▲──┘ └──▲───┘
      │       │
  ┌───┴───────┴───┐
  │  LOAD BALANCER │
  │  (reads only)  │
  └───────┬────────┘
          │
  ┌───────▼────────┐
  │   APPLICATION  │
  │  Writes → Master│
  │  Reads → Replicas│
  └────────────────┘

  ⚡ 1 Master handling 50K writes/sec
  ⚡ 3 Replicas handling 150K reads/sec (50K each)
  ⚡ Total: 200K ops/sec from a simple setup!
```

### ⚠️ Replication Gotchas

```
╔════════════════════════════════════════════════════════════════════╗
║  REPLICATION GOTCHAS — WHAT CAN GO WRONG                         ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  1. ASYNC REPLICATION = POSSIBLE DATA LOSS                        ║
║     Master writes → crashes → replica hasn't received the write  ║
║     → Data lost! (up to ~1 second of writes)                     ║
║     Mitigation: min-replicas-to-write + WAIT command             ║
║                                                                    ║
║  2. STALE READS FROM REPLICAS                                     ║
║     Replica is always slightly behind master                     ║
║     Read from replica → might get old data                       ║
║     Mitigation: WAIT command for critical reads, or read master  ║
║                                                                    ║
║  3. FULL RESYNC = EXPENSIVE                                       ║
║     If backlog overflows during disconnect → full resync needed  ║
║     Full resync on 50 GB = minutes of high CPU + memory          ║
║     Mitigation: Increase repl-backlog-size                       ║
║                                                                    ║
║  4. SPLIT-BRAIN (Network Partition)                               ║
║     Master can't reach sentinels → sentinels promote replica     ║
║     But old master is still accepting writes!                    ║
║     Two masters = data divergence! 💀                            ║
║     Mitigation: min-replicas-to-write 1                          ║
║     (old master stops accepting writes when replica disconnects) ║
║                                                                    ║
╚════════════════════════════════════════════════════════════════════╝
```

```bash
# WAIT command — synchronous replication for critical writes

SET critical_data "important_value"
WAIT 1 5000    # Wait for 1 replica to ACK, timeout 5 seconds
# → Returns number of replicas that confirmed
# If returns 1 → guaranteed durability on at least 1 replica!
# If returns 0 → no replica confirmed within 5 seconds (risk!)
```

---

## 💾 PART 4: Persistence Deep Dive — Production Tuning

### RDB Tuning

```bash
# ═══════════════════════════════════════
#  Production RDB Configuration
# ═══════════════════════════════════════

# Standard triggers:
save 900 1           # Snapshot if 1 change in 15 minutes
save 300 10          # Snapshot if 10 changes in 5 minutes  
save 60 10000        # Snapshot if 10000 changes in 1 minute

# RDB file settings:
dbfilename dump.rdb
dir /var/lib/redis/

# Compression (recommended):
rdbcompression yes   # LZF compression (~20-50% smaller files)
rdbchecksum yes      # CRC64 checksum for data integrity

# ═══════════════════════════════════════
#  fork() Performance Considerations
# ═══════════════════════════════════════

# RDB uses fork() for background saving:
#
# Dataset Size │ fork() Time  │ Notes
# ────────────-┼──────────────┼──────────────────────────
# 1 GB         │ ~10 ms       │ Negligible
# 10 GB        │ ~100 ms      │ Acceptable
# 50 GB        │ ~500 ms      │ Noticeable pause
# 100 GB       │ ~1-2 sec     │ ⚠️ May need tuning
#
# During fork(): Redis server pauses (can't serve requests!)
# After fork(): Copy-on-Write (COW) — minimal memory overhead
#
# Linux tuning for large datasets:
# echo 1 > /proc/sys/vm/overcommit_memory  (allow fork to succeed)
# Disable THP: echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### AOF Tuning

```bash
# ═══════════════════════════════════════
#  Production AOF Configuration
# ═══════════════════════════════════════

appendonly yes
appendfilename "appendonly.aof"

# Sync policies (choose ONE):
# appendfsync always         # Safest: fsync every write (~500 ops/sec penalty)
appendfsync everysec         # ⭐ BEST BALANCE: max 1 second data loss
# appendfsync no             # Fastest: OS decides when to flush (risky)

# AOF Rewrite (compaction — shrinks the AOF file):
auto-aof-rewrite-percentage 100    # Rewrite when AOF doubles in size
auto-aof-rewrite-min-size 64mb     # Don't rewrite until AOF > 64 MB

# Multi-Part AOF (Redis 7.0+):
aof-use-rdb-preamble yes           # ⭐ Hybrid format: RDB header + AOF tail
                                    # Faster restart + near-zero data loss

# ═══════════════════════════════════════
#  AOF Rewrite — What Happens
# ═══════════════════════════════════════

# Before rewrite (AOF file):
#   SET counter 0
#   INCR counter        # counter = 1
#   INCR counter        # counter = 2
#   INCR counter        # counter = 3
#   DEL counter
#   SET counter 0
#   INCR counter        # counter = 1
#   (7 commands for counter=1)

# After rewrite (compacted):
#   SET counter 1
#   (1 command! Same result, much smaller file)

# Rewrite happens in background (fork), no downtime!
```

### Which Persistence Strategy?

```
╔════════════════════════════════════════════════════════════════════╗
║  SCENARIO                           │  RECOMMENDED                ║
╠════════════════════════════════════════════════════════════════════╣
║  Cache-only (data loss OK)          │  RDB only (or no persist)   ║
║  Cache + some durability            │  RDB only, save 300 10      ║
║  Data store (minimal loss OK)       │  AOF everysec              ║
║  Data store (zero loss required)    │  AOF always + replicas     ║
║  Best balance (recommended!)        │  ⭐ RDB + AOF (hybrid)     ║
║  Maximum performance                │  RDB only + replicas       ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  ⭐ PRODUCTION RECOMMENDATION:                                    ║
║  appendonly yes                                                    ║
║  appendfsync everysec                                             ║
║  aof-use-rdb-preamble yes                                         ║
║  save 900 1                                                       ║
║  save 300 10                                                      ║
║  save 60 10000                                                    ║
║  + At least 1 replica with persistence enabled                    ║
║                                                                    ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## 🚀 PART 5: Production Deployment Checklist

### Infrastructure

```
╔════════════════════════════════════════════════════════════════════╗
║  REDIS PRODUCTION CHECKLIST                                       ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                    ║
║  SIZING:                                                           ║
║  □ RAM: 2x your dataset size (for fork + overhead)                ║
║  □ CPU: 2-4 cores (1 for Redis, rest for OS/persistence)         ║
║  □ Disk: SSD (NVMe preferred for AOF fsync)                     ║
║  □ Network: Low latency (< 1ms between app and Redis)            ║
║                                                                    ║
║  MEMORY:                                                           ║
║  □ Set maxmemory (don't let Redis eat all RAM!)                  ║
║  □ Choose eviction policy (allkeys-lfu for cache)                ║
║  □ Monitor: INFO memory → used_memory vs maxmemory               ║
║  □ vm.overcommit_memory = 1 (allow fork for BGSAVE)             ║
║                                                                    ║
║  PERSISTENCE:                                                      ║
║  □ Enable AOF + RDB (hybrid recommended)                         ║
║  □ Test recovery: Stop Redis → Restart → Verify data             ║
║  □ Backup RDB files to off-site storage (S3, etc.)              ║
║                                                                    ║
║  SECURITY:                                                         ║
║  □ Set strong password (requirepass)                              ║
║  □ Bind to specific interfaces (not 0.0.0.0)                    ║
║  □ Rename/disable dangerous commands (FLUSHALL, CONFIG, KEYS)    ║
║  □ Enable TLS for data in transit                                ║
║  □ Use ACLs for fine-grained access control (Redis 6.0+)        ║
║  □ Run Redis as non-root user                                     ║
║                                                                    ║
║  HIGH AVAILABILITY:                                                ║
║  □ Sentinel (3 instances minimum) OR Cluster (6 nodes minimum)   ║
║  □ Test failover before going to production!                     ║
║  □ Use Sentinel-aware / Cluster-aware client libraries           ║
║                                                                    ║
║  MONITORING:                                                       ║
║  □ INFO command metrics → Prometheus/Grafana                     ║
║  □ SLOWLOG for slow commands                                     ║
║  □ LATENCY DOCTOR for latency issues                             ║
║  □ Alert on: memory > 80%, hit rate < 90%, evictions > 0        ║
║  □ Alert on: replication lag, connected replicas dropping        ║
║                                                                    ║
║  PERFORMANCE:                                                      ║
║  □ Disable THP (Transparent Huge Pages)                          ║
║  □ Set somaxconn = 65535 (TCP backlog)                          ║
║  □ Set tcp-backlog 511 in redis.conf                            ║
║  □ Enable io-threads for Redis 6.0+ (4-8 threads)              ║
║  □ Use UNLINK instead of DEL for large keys                     ║
║  □ Use SCAN instead of KEYS                                     ║
║  □ Pipeline batch operations                                     ║
╚════════════════════════════════════════════════════════════════════╝
```

### ACL — Access Control (Redis 6.0+)

```bash
# ═══════════════════════════════════════
#  User-based access control
# ═══════════════════════════════════════

# Create a read-only user:
ACL SETUSER reader on >readpassword ~* &* +@read
# on = active, >password, ~* = all keys, +@read = read commands only

# Create an app user (read + write, specific prefix):
ACL SETUSER appuser on >apppassword ~app:* &* +@read +@write +@string +@hash
# ~app:* = only keys starting with "app:"

# Create an admin user:
ACL SETUSER admin on >adminpassword ~* &* +@all

# List users:
ACL LIST

# Who am I?
ACL WHOAMI

# Check what a user can do:
ACL GETUSER reader

# ⚠️ Disable the default user in production:
ACL SETUSER default off
```

### Monitoring & Debugging

```bash
# ═══════════════════════════════════════
#  Essential Monitoring Commands
# ═══════════════════════════════════════

# Overall health:
INFO all                      # Everything
INFO server                   # Version, uptime, config
INFO memory                   # Memory usage, fragmentation
INFO stats                    # Hit/miss rate, commands processed
INFO replication              # Master/replica status, lag
INFO clients                  # Connected clients, blocked clients

# Real-time command monitoring (⚠️ performance impact!):
MONITOR                       # Shows EVERY command in real-time
# Use only for debugging! Never in production long-term.

# Slow command log:
SLOWLOG GET 10               # Last 10 slow commands
SLOWLOG LEN                  # Number of slow entries
SLOWLOG RESET                # Clear slow log

# Latency diagnostics:
LATENCY LATEST               # Latest latency events
LATENCY HISTORY event        # History for specific event
LATENCY DOCTOR               # Human-readable latency report

# Memory analysis:
MEMORY DOCTOR                 # Memory diagnostics
MEMORY USAGE key              # Exact memory for a key
MEMORY STATS                  # Detailed memory breakdown
redis-cli --bigkeys           # Find largest keys
redis-cli --memkeys           # Memory per key pattern

# Client management:
CLIENT LIST                   # All connected clients
CLIENT KILL ID 42             # Kill specific client
CLIENT SETNAME "worker-1"     # Name your connections (for debugging)

# Debug (development only!):
DEBUG SLEEP 0                 # Don't use in production!
DEBUG OBJECT key              # Internal object details
```

### Linux Kernel Tuning for Redis

```bash
# ═══════════════════════════════════════
#  /etc/sysctl.conf additions
# ═══════════════════════════════════════

# Allow overcommit for BGSAVE fork():
vm.overcommit_memory = 1

# Increase max connections:
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# Increase file descriptors:
# /etc/security/limits.conf:
# redis soft nofile 65535
# redis hard nofile 65535

# Disable Transparent Huge Pages:
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Apply:
sysctl -p
```

---

## ☁️ Managed Redis Services — Cloud Options

```
╔════════════════════════════════════════════════════════════════════════╗
║  SERVICE              │  PROVIDER  │  BEST FOR                       ║
╠════════════════════════════════════════════════════════════════════════╣
║  AWS ElastiCache      │  AWS       │  AWS ecosystem, auto-scaling    ║
║  Azure Cache          │  Microsoft │  Azure/Microsoft ecosystem      ║
║  GCP Memorystore      │  Google    │  GCP ecosystem                  ║
║  Redis Cloud          │  Redis Ltd │  Full Redis Stack, multi-cloud  ║
║  Aiven for Redis      │  Aiven     │  Multi-cloud, managed           ║
║  Upstash              │  Upstash   │  Serverless, pay-per-request    ║
╠════════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  WHEN TO USE MANAGED:                                                 ║
║  ✅ You don't want to manage infrastructure                          ║
║  ✅ Need automatic backups, failover, monitoring                     ║
║  ✅ Production workload with SLA requirements                        ║
║  ✅ Multi-AZ / Multi-region requirements                             ║
║                                                                       ║
║  WHEN TO SELF-HOST:                                                   ║
║  ✅ Need Redis Stack modules (some managed services don't support)   ║
║  ✅ Extreme performance tuning needed                                ║
║  ✅ Cost optimization at very large scale                            ║
║  ✅ Data residency / compliance requirements                         ║
╚════════════════════════════════════════════════════════════════════════╝
```

---

## 🧪 Hands-On Lab — Production Redis Setup

```bash
# ═══════════════════════════════════════
#  Complete Production redis.conf
# ═══════════════════════════════════════

# === NETWORK ===
bind 10.0.0.5                         # Bind to private IP
port 6379
protected-mode yes
tcp-backlog 511
timeout 300
tcp-keepalive 60

# === SECURITY ===
requirepass "YOUR_VERY_STRONG_PASSWORD_HERE_MIN_32_CHARS"
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG "CONFIG_a8f3b2"   # Rename, don't disable
rename-command KEYS ""
rename-command DEBUG ""

# === MEMORY ===
maxmemory 4gb
maxmemory-policy allkeys-lfu
# For cache: allkeys-lru or allkeys-lfu
# For data store: noeviction

# === PERSISTENCE ===
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb
dir /var/lib/redis

appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-use-rdb-preamble yes

# === PERFORMANCE ===
io-threads 4
io-threads-do-reads yes
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes

# === LOGGING ===
loglevel notice
logfile /var/log/redis/redis.log
slowlog-log-slower-than 10000
slowlog-max-len 256

# === REPLICATION ===
# replicaof <master-ip> <master-port>     # Enable on replicas
repl-backlog-size 64mb
min-replicas-to-write 1
min-replicas-max-lag 10
```

---

## 📊 Complete Chapter Summary — Redis Production Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║               REDIS PRODUCTION CHEAT SHEET                           ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ARCHITECTURE OPTIONS:                                               ║
║  Standalone  → Dev/test only (single point of failure)               ║
║  Sentinel    → HA failover (< 50 GB data, < 100K ops/sec)          ║
║  Cluster     → HA + sharding (> 50 GB or > 100K ops/sec)           ║
║  Managed     → ⭐ Easiest (ElastiCache, Azure Cache, Redis Cloud)  ║
║                                                                      ║
║  REPLICATION:                                                        ║
║  Async by default → possible data loss on master failure            ║
║  WAIT command → synchronous (for critical data)                     ║
║  min-replicas-to-write 1 → prevent split-brain                     ║
║  repl-backlog-size 64mb → enable partial resync                     ║
║                                                                      ║
║  PERSISTENCE (recommended combo):                                    ║
║  appendonly yes + appendfsync everysec + save rules + hybrid AOF    ║
║  Test recovery regularly!                                            ║
║                                                                      ║
║  SECURITY:                                                           ║
║  requirepass (strong!) + bind (private IP) + ACLs                   ║
║  Rename FLUSHALL, CONFIG, KEYS, DEBUG                               ║
║  TLS for production                                                  ║
║                                                                      ║
║  MONITORING ESSENTIALS:                                              ║
║  INFO stats → hit_rate > 90%                                        ║
║  INFO memory → used < 80% maxmemory                                ║
║  SLOWLOG GET → find slow commands                                   ║
║  LATENCY DOCTOR → diagnose latency spikes                           ║
║  --bigkeys → find memory hogs                                       ║
║                                                                      ║
║  KERNEL TUNING:                                                      ║
║  vm.overcommit_memory = 1                                           ║
║  Disable THP                                                         ║
║  somaxconn = 65535                                                   ║
║  nofile = 65535                                                      ║
║                                                                      ║
║  CLUSTER TIPS:                                                       ║
║  Use hash tags {prefix} for multi-key ops                           ║
║  Min 6 nodes (3 masters + 3 replicas)                               ║
║  Test resharding before production                                   ║
║  Monitor CLUSTER INFO regularly                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🎓 Redis Mastery Complete!

```
Your Redis Journey:

  3C.1 Architecture ✅  →  You understand WHY Redis is fast
       Data Structures     and WHAT data structures to use

  3C.2 Caching ✅        →  You can design production caching
       Patterns              layers that scale

  3C.3 Advanced ✅       →  You can build real-time systems with
       Pub/Sub, Lua          Pub/Sub, Streams, and atomic Lua scripts

  3C.4 Cluster & HA ✅   →  You can deploy, monitor, and operate
       Production            Redis in production like a senior engineer

  🏆 You are now Redis-certified (in knowledge)!
```

---

## 🔗 Where to Go Next?

| Chapter | Topic |
|---|---|
| [3D → Cassandra](../11-Cassandra/01-Cassandra-Architecture.md) | Wide-column store for massive write-heavy workloads |
| [7.4 → Caching Strategies](../18-Architecture/04-Caching-Strategies.md) | System design perspective on caching architectures |
| [7.5 → System Design — DB Perspective](../18-Architecture/05-System-Design-DB.md) | Use Redis in system design interviews |

---

> 💡 **Final Thought:** Redis's power isn't in being fast — many things are fast. Redis's power is in giving you the **right data structure** for each problem, **in memory**, with **atomic operations**, at **web scale**. That combination is why it's in the stack of virtually every major tech company on Earth.

---

*Congratulations! You've completed the Redis chapter. Go build something amazing.* 🚀
