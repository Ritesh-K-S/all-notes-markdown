# Chapter 28: ElastiCache

---

## Table of Contents

- [Overview](#overview)
- [Part 1: ElastiCache Fundamentals](#part-1-elasticache-fundamentals)
- [Part 2: ElastiCache for Redis (Full Portal Walkthrough)](#part-2-elasticache-for-redis-full-portal-walkthrough)
- [Part 3: ElastiCache for Memcached](#part-3-elasticache-for-memcached)
- [Part 4: Cluster Mode & Replication](#part-4-cluster-mode--replication)
- [Part 5: Security & Encryption](#part-5-security--encryption)
- [Part 6: Caching Strategies](#part-6-caching-strategies)
- [Part 7: Terraform & CLI Examples](#part-7-terraform--cli-examples)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Caching? Why Do We Need ElastiCache?

**Caching is like keeping frequently used books on your desk instead of walking to the library every time.** Your database is the library — reliable but slow for repeated requests. A cache is your desk — fast, but limited space.

Without caching, every time a user loads your website, the app queries the database. If 10,000 users load the same product page, that's 10,000 identical database queries. With caching, the first query goes to the database, stores the result in the cache, and the next 9,999 users get the cached result in **under 1 millisecond** instead of 10-50ms from the database.

**Why ElastiCache instead of caching in your app's memory?**
- App memory cache is lost when the server restarts
- App memory cache can't be shared across multiple servers
- ElastiCache is a dedicated, shared, persistent cache that all your servers access

**Simple real-world examples:**
- 🛒 An e-commerce site caches product details (product info doesn't change every second)
- 🔐 A web app stores user sessions in Redis (so users stay logged in even if the server restarts)
- 📰 A news site caches article content (same article served to millions of readers)
- 🏆 A gaming leaderboard stored in Redis Sorted Sets (real-time ranking)

Amazon ElastiCache provides fully managed in-memory caching with Redis or Memcached. It delivers sub-millisecond latency for read-heavy workloads, session management, real-time analytics, and as a database cache layer.

```
What you'll learn:
├── ElastiCache Fundamentals (Redis vs Memcached)
├── Creating a Redis Cluster (Full Portal Walkthrough)
├── Memcached Configuration
├── Cluster Mode (sharding for Redis)
├── Replication & High Availability
├── Security (encryption, auth, VPC)
├── Caching Strategies (lazy loading, write-through, TTL)
├── Terraform & CLI examples
└── Real-world patterns
```

---

## Part 1: ElastiCache Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           REDIS vs MEMCACHED                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┬─────────────────────┬────────────────────────┐│
│ │ Feature          │ Redis               │ Memcached              ││
│ ├──────────────────┼─────────────────────┼────────────────────────┤│
│ │ Data structures  │ Strings, lists,     │ Simple key-value only  ││
│ │                  │ sets, hashes, sorted│                        ││
│ │                  │ sets, streams, etc. │                        ││
│ │ Persistence      │ Yes (RDB + AOF)     │ No (pure cache)        ││
│ │ Replication      │ Yes (primary+replica)│ No                    ││
│ │ Multi-AZ failover│ Yes (automatic)     │ No                     ││
│ │ Cluster mode     │ Yes (sharding)      │ Yes (auto-discovery)   ││
│ │ Pub/Sub          │ Yes                 │ No                     ││
│ │ Lua scripting    │ Yes                 │ No                     ││
│ │ Transactions     │ Yes (MULTI/EXEC)    │ No                     ││
│ │ Backup/restore   │ Yes                 │ No                     ││
│ │ Encryption       │ At rest + in transit│ In transit only         ││
│ │ Max item size    │ 512 MB              │ 1 MB                   ││
│ │ Multi-threaded   │ Single-threaded*    │ Multi-threaded          ││
│ └──────────────────┴─────────────────────┴────────────────────────┘│
│ * Redis 7+ has I/O threading for better performance              │
│                                                                       │
│ Decision:                                                            │
│ ├── ⚡ Choose Redis (90% of cases): Rich data types, persistence,│
│ │   replication, Pub/Sub, Lua, backup/restore, HA               │
│ └── Choose Memcached only if: Simple key-value, multi-threaded  │
│     needed, no persistence/replication needed                    │
│                                                                       │
│ Pricing (us-east-1, cache.m6g.large):                              │
│ ├── On-Demand: ~$0.172/hr ($124/month)                          │
│ ├── Reserved (1-year): ~$0.113/hr ($81/month) — 34% off        │
│ ├── Serverless Redis: Pay per ECPUs + storage                   │
│ └── Data transfer: Standard AWS rates                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: ElastiCache for Redis (Full Portal Walkthrough)

```
Console → ElastiCache → Redis caches → Create Redis cache

┌─────────────────────────────────────────────────────────────────┐
│           CREATE REDIS CACHE                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Deployment option ──                                        │
│ ● Design your own cache                                        │
│ ○ Easy create                                                   │
│                                                                   │
│ Creation method:                                                │
│ ● Cluster cache                                                 │
│ ○ Serverless cache                                              │
│                                                                   │
│ → Cluster: You choose node types, size, and configuration    │
│ → Serverless: Auto-scales, pay-per-use. Good for variable    │
│   workloads. No node management.                               │
│                                                                   │
│ ── Cluster info ──                                              │
│ Name: [prod-redis-01]                                          │
│ Description: [Production Redis cache]                          │
│                                                                   │
│ ── Cluster mode ──                                              │
│ ● Enabled (recommended for large datasets)                     │
│ ○ Disabled                                                      │
│                                                                   │
│ → Enabled: Data sharded across multiple node groups (shards). │
│   Each shard has a primary + replicas. Scales horizontally.   │
│   Max: 500 shards. Use for: Large datasets, high throughput. │
│ → Disabled: Single shard (1 primary + up to 5 replicas).     │
│   Max data = single node memory. Simpler to manage.           │
│                                                                   │
│ ── Location ──                                                  │
│ ● AWS Cloud  ○ On premises (Outposts)                          │
│                                                                   │
│ Multi-AZ: ☑ Enabled                                           │
│ → Automatic failover to replica in different AZ               │
│ → ⚡ Always enable for production                              │
│                                                                   │
│ Auto-failover: ☑ Enabled                                      │
│ → Promotes replica to primary if primary fails                │
│                                                                   │
│ ── Cluster settings (cluster mode enabled) ──                  │
│ Number of shards: [3]                                         │
│ → More shards = more data capacity + throughput               │
│                                                                   │
│ Replicas per shard: [2]                                       │
│ → 0-5 replicas. More = higher read throughput + HA           │
│ → ⚡ 2+ replicas for production                               │
│                                                                   │
│ Node type: [cache.m6g.large ▼]                                │
│   ├── cache.t3.micro: 0.5 GB (dev/test, free tier)          │
│   ├── cache.t3.medium: 3.09 GB (small production)           │
│   ├── cache.m6g.large: 6.38 GB (⚡ production start)        │
│   ├── cache.r6g.large: 13.07 GB (memory-intensive)          │
│   └── cache.r6g.xlarge: 26.32 GB (large datasets)           │
│                                                                   │
│ ── Connectivity ──                                              │
│ Subnet group: [redis-private-subnets ▼]                       │
│ → Must be in private subnets                                  │
│ → Create: ElastiCache → Subnet groups → Create                │
│                                                                   │
│ ── Availability zone placements ──                             │
│ ● Auto  ○ Manual (select specific AZs per node)              │
│                                                                   │
│ ── Advanced settings ──                                        │
│                                                                   │
│ Engine version: [7.1 ▼]                                       │
│ → Latest stable version recommended                           │
│                                                                   │
│ Port: [6379]                                                   │
│ → Default Redis port                                           │
│                                                                   │
│ Parameter group: [default.redis7.cluster.on ▼]                │
│ → Engine configuration (maxmemory-policy, timeout, etc.)     │
│ → Create custom group for production tuning                  │
│                                                                   │
│ ── Security ──                                                  │
│ ☑ Encryption at rest                                           │
│ ☑ Encryption in transit                                        │
│                                                                   │
│ Access control:                                                │
│   ● Redis AUTH (set a password/token)                          │
│   ○ User group access control list (ACL)                      │
│                                                                   │
│ → AUTH: Simple password protection                            │
│ → ACL: User-level access control (Redis 6+). Create users   │
│   with specific permissions. ⚡ Recommended for production.   │
│                                                                   │
│ Security groups: [sg-redis ▼]                                 │
│ → Allow TCP 6379 from application security group             │
│                                                                   │
│ ── Backup ──                                                    │
│ ☑ Enable automatic backups                                    │
│ Retention period: [7] days                                    │
│ Backup window: [03:00 - 04:00 UTC ▼]                         │
│ → Daily RDB snapshots                                         │
│                                                                   │
│ ── Maintenance ──                                               │
│ Maintenance window: [Sun 04:00 - Sun 05:00 ▼]                │
│ Auto minor version upgrade: ☑                                 │
│                                                                   │
│ ── Tags ──                                                      │
│ Key: [environment]  Value: [production]                       │
│                                                                   │
│                    [Create]                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: ElastiCache for Memcached

```
Console → ElastiCache → Memcached caches → Create Memcached cache

┌─────────────────────────────────────────────────────────────────┐
│           MEMCACHED — KEY DIFFERENCES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [prod-memcached]                                         │
│ Engine version: [1.6.22 ▼]                                    │
│ Port: [11211]                                                  │
│                                                                   │
│ Node type: [cache.m6g.large ▼]                                │
│ Number of nodes: [3]                                           │
│ → Nodes are independent (no replication!)                     │
│ → Data distributed using consistent hashing                  │
│ → Node failure = data loss for that portion                  │
│                                                                   │
│ Auto Discovery: ☑ Enabled                                     │
│ → Client auto-discovers nodes (no hardcoded endpoints)       │
│ → Uses special configuration endpoint                        │
│                                                                   │
│ ⚠️ No encryption at rest (transit encryption available)        │
│ ⚠️ No backup/restore                                           │
│ ⚠️ No replication or failover                                  │
│ ⚠️ No persistence                                              │
│                                                                   │
│ Use only when:                                                  │
│ ├── Simple key-value cache is sufficient                     │
│ ├── Multi-threaded performance needed                        │
│ └── Data loss is acceptable (pure cache, can rebuild)        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Cluster Mode & Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│           REDIS CLUSTER MODE                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cluster Mode Disabled (single shard):                               │
│ ┌───────────────────────────────────────────────────────────────┐  │
│ │                                                                │  │
│ │  ┌─────────┐    ┌──────────┐    ┌──────────┐                 │  │
│ │  │ Primary │───►│ Replica 1│    │ Replica 2│                 │  │
│ │  │ (R/W)   │───►│ (Read)   │    │ (Read)   │                 │  │
│ │  └─────────┘    └──────────┘    └──────────┘                 │  │
│ │       AZ-a          AZ-b           AZ-c                      │  │
│ │                                                                │  │
│ │  All data on one shard. Max = node memory.                    │  │
│ │  Read scaling: Add replicas. Write: Single primary only.     │  │
│ └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Cluster Mode Enabled (multiple shards):                             │
│ ┌───────────────────────────────────────────────────────────────┐  │
│ │                                                                │  │
│ │  Shard 1            Shard 2            Shard 3               │  │
│ │  (slots 0-5461)    (slots 5462-10922) (slots 10923-16383)  │  │
│ │  ┌─────────┐       ┌─────────┐       ┌─────────┐           │  │
│ │  │ Primary │       │ Primary │       │ Primary │            │  │
│ │  ├─────────┤       ├─────────┤       ├─────────┤            │  │
│ │  │Replica 1│       │Replica 1│       │Replica 1│            │  │
│ │  │Replica 2│       │Replica 2│       │Replica 2│            │  │
│ │  └─────────┘       └─────────┘       └─────────┘            │  │
│ │                                                                │  │
│ │  Data partitioned by hash slot across shards.                │  │
│ │  Total capacity = sum of all shard memories.                 │  │
│ │  Can add/remove shards (online resharding).                  │  │
│ └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Decision:                                                            │
│ ├── Data < single node memory → Cluster Mode Disabled           │
│ ├── Data > single node memory → Cluster Mode Enabled            │
│ ├── Need horizontal write scaling → Cluster Mode Enabled        │
│ └── Simple setup, small data → Cluster Mode Disabled            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Security & Encryption

```
┌─────────────────────────────────────────────────────────────────────┐
│           ELASTICACHE SECURITY                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Network:                                                             │
│ ├── Private subnets only (no public access)                      │
│ ├── Security group: Allow port 6379 from app SG                 │
│ └── ⚠️ No VPC endpoints (must be in same VPC as clients)         │
│                                                                       │
│ Authentication (Redis):                                             │
│ ├── Redis AUTH: Simple token/password                             │
│ ├── Redis ACL (6.0+): User-level access control                │
│ │   ├── Default user + custom users                             │
│ │   ├── Per-user command restrictions                            │
│ │   ├── Per-user key pattern restrictions                       │
│ │   └── ⚡ Recommended for production                           │
│ └── IAM authentication: Use IAM users/roles (Redis 7+)         │
│                                                                       │
│ Encryption at rest: AES-256 via KMS                               │
│ Encryption in transit: TLS (redis-cli --tls ...)                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Caching Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│           CACHING STRATEGIES                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Lazy Loading (Cache-Aside):                                      │
│    App reads from cache first.                                      │
│    Cache miss → read from DB → write to cache → return.           │
│    ├── ✅ Only requested data is cached (efficient)               │
│    ├── ✅ Cache failures don't break the app                      │
│    ├── ⚠️ Cache miss = 3 round trips (slow first access)         │
│    └── ⚠️ Data can become stale (set TTL to mitigate)            │
│                                                                       │
│ 2. Write-Through:                                                    │
│    Every write goes to BOTH cache and DB simultaneously.           │
│    ├── ✅ Cache is always up-to-date                               │
│    ├── ✅ Read latency is always fast (no misses)                  │
│    ├── ⚠️ Every write = 2 operations (slower writes)              │
│    └── ⚠️ Lots of data cached that may never be read              │
│                                                                       │
│ 3. Write-Behind (Write-Back):                                       │
│    Write to cache immediately, async write to DB later.            │
│    ├── ✅ Fast writes (async DB update)                            │
│    ├── ⚠️ Risk of data loss if cache fails before DB write        │
│    └── Complex to implement reliably                               │
│                                                                       │
│ 4. TTL (Time-To-Live):                                              │
│    Set expiration on cached items.                                  │
│    ├── ✅ Prevents stale data (eventual consistency)               │
│    ├── ✅ Automatic cache eviction                                  │
│    ├── Common TTLs: sessions=24h, product=1h, config=5min        │
│    └── ⚡ Always set TTL — even with write-through                │
│                                                                       │
│ ⚡ Most common: Lazy Loading + TTL                                   │
│                                                                       │
│ Eviction policies (maxmemory-policy):                               │
│ ├── allkeys-lru: Evict least recently used (⚡ most common)     │
│ ├── volatile-lru: Evict LRU keys with TTL set                   │
│ ├── allkeys-lfu: Evict least frequently used                     │
│ ├── volatile-ttl: Evict keys with shortest TTL                  │
│ └── noeviction: Return error when memory full                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6.5: ElastiCache Serverless

ElastiCache Serverless **automatically scales** capacity based on your workload. No node selection, no capacity planning — you just create a cache and use it.

### When to Use Serverless vs Standard

| Aspect | Serverless | Standard (node-based) |
|--------|-----------|----------------------|
| Capacity planning | None needed | You choose node type and count |
| Scaling | Automatic (instant) | Manual or slow auto-scaling |
| Cost model | Pay per GB stored + per ECPUs | Pay per node-hour |
| Best for | Variable/unknown workloads | Predictable, steady workloads |
| Cost at scale | More expensive | Cheaper (predictable discounts) |
| Minimum cost | ~$90/month | ~$12/month (cache.t4g.micro) |

### Console Setup

```
Console → ElastiCache → Create cache

  ┌─────────────────────────────────────────────────────────┐
  │ Deployment option                                        │
  │                                                          │
  │ ● Design your own cache                                  │
  │   Creation method:                                       │
  │   ○ Standard create    ← Node-based (you manage)        │
  │   ● Easy create                                          │
  │                                                          │
  │ OR                                                       │
  │                                                          │
  │ ● Serverless           ← No nodes, auto-scales          │
  │   Cache name: my-cache                                   │
  │   Engine: Redis | Valkey | Memcached                     │
  │   [Create]                                               │
  └─────────────────────────────────────────────────────────┘
```

> 💡 **Start with Serverless** if you're unsure about your traffic patterns. You can always migrate to node-based later for cost savings once you understand your workload.

---

## Part 7: Terraform & CLI Examples

```hcl
# Redis Replication Group (cluster mode disabled)
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "prod-redis"
  description          = "Production Redis"
  engine               = "redis"
  engine_version       = "7.1"
  node_type            = "cache.m6g.large"
  num_cache_clusters   = 3  # 1 primary + 2 replicas

  automatic_failover_enabled = true
  multi_az_enabled           = true

  subnet_group_name  = aws_elasticache_subnet_group.redis.name
  security_group_ids = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token

  snapshot_retention_limit = 7
  snapshot_window          = "03:00-04:00"
  maintenance_window       = "sun:04:00-sun:05:00"

  auto_minor_version_upgrade = true

  tags = { Environment = "prod" }
}

resource "aws_elasticache_subnet_group" "redis" {
  name       = "redis-private"
  subnet_ids = [aws_subnet.private_a.id, aws_subnet.private_b.id]
}
```

```bash
# Create Redis cluster
aws elasticache create-replication-group \
  --replication-group-id prod-redis \
  --replication-group-description "Production Redis" \
  --engine redis \
  --engine-version 7.1 \
  --cache-node-type cache.m6g.large \
  --num-cache-clusters 3 \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --cache-subnet-group-name redis-private \
  --security-group-ids sg-redis \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled

# Create snapshot
aws elasticache create-snapshot \
  --replication-group-id prod-redis \
  --snapshot-name prod-redis-backup

# Describe clusters
aws elasticache describe-replication-groups \
  --replication-group-id prod-redis
```

---

## Part 8: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Database Cache                                           │
│ App → Redis (cache-aside) → RDS/Aurora                             │
│ ├── Cache hot queries (user profiles, product details)           │
│ ├── TTL: 1-24 hours depending on data freshness needs           │
│ ├── Cache key: table:primarykey (e.g., user:123)                │
│ └── 10-100x fewer database queries                               │
│                                                                       │
│ Pattern 2: Session Store                                            │
│ ├── Store session data in Redis (fast, shared across servers)   │
│ ├── TTL: Match session expiry (e.g., 24 hours)                  │
│ ├── Multi-AZ for HA (sessions survive node failure)             │
│ └── All app servers read from same Redis → stateless servers    │
│                                                                       │
│ Pattern 3: Real-Time Leaderboard                                    │
│ ├── Redis Sorted Sets (ZADD, ZRANGEBYSCORE)                     │
│ ├── O(log N) insertion and ranking                               │
│ ├── ZREVRANGE to get top N scores                                │
│ └── Millions of entries with sub-millisecond ranking            │
│                                                                       │
│ Pattern 4: Rate Limiting                                            │
│ ├── Redis INCR + EXPIRE for sliding window                      │
│ ├── Key: ratelimit:userId:minute                                │
│ ├── INCR on each request, check against limit                   │
│ └── Atomic operations → no race conditions                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
ElastiCache Quick Reference:
├── Engines: Redis (⚡ preferred) or Memcached
├── Latency: Sub-millisecond
├── Redis: Rich data types, persistence, replication, Pub/Sub
├── Memcached: Simple K/V, multi-threaded, no persistence
├── Cluster mode: Enabled (sharded) or Disabled (single shard)
├── Replication: Up to 5 replicas per shard (Redis)
├── Multi-AZ: Automatic failover to replica
├── Auth: Redis AUTH, ACL (6+), IAM (7+)
├── Encryption: At rest (KMS) + in transit (TLS)
├── Caching: Lazy loading + TTL (most common strategy)
├── Serverless: Auto-scaling, pay-per-use option
└── ⚡ Use Redis unless you specifically need Memcached
```

---

## What's Next?

In **Chapter 29: Other AWS Databases**, we'll cover DocumentDB, Neptune, Timestream, QLDB, and Amazon Keyspaces — specialized databases for specific use cases.
