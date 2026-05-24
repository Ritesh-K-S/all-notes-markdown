# Chapter 28: Memorystore

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Memorystore Fundamentals](#part-1-memorystore-fundamentals)
- [Part 2: Redis vs Memcached — Which to Choose](#part-2-redis-vs-memcached--which-to-choose)
- [Part 3: Memorystore for Redis](#part-3-memorystore-for-redis)
- [Part 4: Creating a Redis Instance](#part-4-creating-a-redis-instance)
- [Part 5: Redis Tiers — Basic vs Standard vs Cluster](#part-5-redis-tiers--basic-vs-standard-vs-cluster)
- [Part 6: Connecting to Redis](#part-6-connecting-to-redis)
- [Part 7: Redis Configuration & Tuning](#part-7-redis-configuration--tuning)
- [Part 8: Redis Persistence & Backup](#part-8-redis-persistence--backup)
- [Part 9: Redis AUTH & Security](#part-9-redis-auth--security)
- [Part 10: Memorystore for Memcached](#part-10-memorystore-for-memcached)
- [Part 11: Creating a Memcached Instance](#part-11-creating-a-memcached-instance)
- [Part 12: Memorystore for Valkey](#part-12-memorystore-for-valkey)
- [Part 13: Monitoring & Maintenance](#part-13-monitoring--maintenance)
- [Part 14: Terraform & CLI](#part-14-terraform--cli)
- [Part 15: Real-World Patterns](#part-15-real-world-patterns)
- [Console Walkthrough: Managing & Deleting Instances](#console-walkthrough-managing--deleting-instances)
- [Cache Patterns for Beginners](#cache-patterns-for-beginners)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Memorystore is Google Cloud's fully managed in-memory data store service. It provides managed Redis, Memcached, and Valkey instances that you can spin up in minutes — no patching, no failover scripting, no replication setup. Use it for caching, session storage, leaderboards, pub/sub, queues, and any workload that needs sub-millisecond latency.

```
┌─────────────────────────────────────────────────────────────────────┐
│ WHAT YOU'LL LEARN                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ├── 1. What is Memorystore and when to use it                      │
│ ├── 2. Redis vs Memcached — decision framework                     │
│ ├── 3. Memorystore for Redis deep dive                             │
│ ├── 4. Creating a Redis instance (console walkthrough)             │
│ ├── 5. Redis tiers: Basic, Standard, Cluster mode                  │
│ ├── 6. Connecting from GCE, GKE, Cloud Run, Cloud Functions       │
│ ├── 7. Configuration parameters and tuning                         │
│ ├── 8. Persistence (RDB snapshots) and backups                     │
│ ├── 9. AUTH, in-transit encryption, and IAM                        │
│ ├── 10. Memorystore for Memcached                                  │
│ ├── 11. Creating a Memcached instance                              │
│ ├── 12. Memorystore for Valkey (open-source Redis alternative)     │
│ ├── 13. Monitoring, alerting, and maintenance windows              │
│ ├── 14. Terraform and gcloud CLI                                   │
│ └── 15. Real-world architecture patterns                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Memorystore Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT IS MEMORYSTORE?                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Memorystore = fully managed in-memory data stores on GCP.          │
│                                                                       │
│ Three engines available:                                            │
│ ├── Redis: Rich data structures, persistence, pub/sub, scripting   │
│ ├── Memcached: Simple key-value cache, multi-threaded, volatile    │
│ └── Valkey: Open-source Redis fork (post-Redis license change)     │
│                                                                       │
│ Key characteristics (all engines):                                  │
│ ├── Fully managed — no VMs, no OS patching, no cluster setup      │
│ ├── Sub-millisecond latency (in-memory)                            │
│ ├── Deployed in your VPC (private IP only, no public access)       │
│ ├── Automatic failover (Standard/HA tiers)                         │
│ ├── Monitoring via Cloud Monitoring                                │
│ └── Pay per provisioned capacity (not per request)                 │
│                                                                       │
│ When to use Memorystore:                                            │
│ ├── Caching database query results (reduce Cloud SQL / Spanner load│
│ ├── Session storage (web app sessions, shopping carts)             │
│ ├── Rate limiting (API throttling, login attempt limiting)         │
│ ├── Leaderboards and counting (sorted sets in Redis)               │
│ ├── Real-time analytics (HyperLogLog, streams)                     │
│ ├── Message queues and pub/sub (Redis Pub/Sub, Streams)            │
│ ├── Feature flags and configuration cache                          │
│ └── Distributed locks (Redis SETNX / Redlock)                      │
│                                                                       │
│ When NOT to use Memorystore:                                        │
│ ├── Primary data store (data can be evicted/lost)                  │
│ │   → Use Firestore, Cloud SQL, or Spanner                        │
│ ├── Large datasets > 300 GB → Consider Bigtable                   │
│ ├── Complex queries / SQL → Cloud SQL                               │
│ └── Persistent message queue → Pub/Sub (durable)                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

```
┌──────────────────────┬────────────────────┬──────────────┬──────────────┐
│ Feature              │ GCP                │ AWS          │ Azure        │
├──────────────────────┼────────────────────┼──────────────┼──────────────┤
│ Managed Redis        │ Memorystore Redis  │ ElastiCache  │ Azure Cache  │
│                      │                    │ for Redis    │ for Redis    │
│ Managed Memcached    │ Memorystore        │ ElastiCache  │ ❌ No         │
│                      │ Memcached          │ for Memcached│              │
│ Managed Valkey       │ Memorystore Valkey │ ElastiCache  │ ❌ No         │
│                      │                    │ for Valkey   │              │
│ Serverless option    │ ❌ No (provisioned)  │ ✅ Serverless │ ❌ No         │
│ Cluster mode         │ ✅ Redis Cluster    │ ✅ Cluster    │ ✅ Cluster    │
│ Multi-AZ replication │ ✅ Standard tier    │ ✅ Multi-AZ   │ ✅ Premium    │
│ Read replicas        │ ✅ Up to 5          │ ✅ Up to 5    │ ✅ Up to 3    │
│ Persistence          │ ✅ RDB snapshots    │ ✅ RDB + AOF  │ ✅ RDB + AOF  │
│ In-transit TLS       │ ✅ Yes              │ ✅ Yes        │ ✅ Yes        │
│ VPC-only access      │ ✅ Always           │ ✅ Default    │ ✅ Optional   │
│ Max memory           │ 300 GB (single)    │ 6.1 TB       │ 1.2 TB       │
│                      │ 5 TB+ (cluster)    │ (cluster)    │ (cluster)    │
└──────────────────────┴────────────────────┴──────────────┴──────────────┘
```

---

## Part 2: Redis vs Memcached — Which to Choose

```
┌─────────────────────────────────────────────────────────────────────┐
│           REDIS vs MEMCACHED DECISION FRAMEWORK                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌───────────────────────┬──────────────────┬──────────────────────┐│
│ │ Feature               │ Redis ✅          │ Memcached            ││
│ ├───────────────────────┼──────────────────┼──────────────────────┤│
│ │ Data structures       │ Strings, hashes, │ Strings only         ││
│ │                       │ lists, sets,     │                      ││
│ │                       │ sorted sets,     │                      ││
│ │                       │ streams, bitmaps,│                      ││
│ │                       │ HyperLogLog, geo │                      ││
│ │ Persistence           │ ✅ RDB snapshots  │ ❌ Volatile only      ││
│ │ Replication (HA)      │ ✅ Standard tier  │ ❌ No replication     ││
│ │ Pub/Sub               │ ✅ Built-in       │ ❌ No                 ││
│ │ Transactions          │ ✅ MULTI/EXEC     │ ❌ No                 ││
│ │ Lua scripting         │ ✅ Yes            │ ❌ No                 ││
│ │ TTL per key           │ ✅ Yes            │ ✅ Yes                ││
│ │ Max item size         │ 512 MB           │ 1 MB (default)       ││
│ │ Threading model       │ Single-threaded* │ Multi-threaded       ││
│ │ Memory efficiency     │ Good             │ Better (no overhead) ││
│ │ Automatic failover    │ ✅ Standard tier  │ ❌ No                 ││
│ │ Read replicas         │ ✅ Up to 5        │ ❌ No                 ││
│ │ Cluster/sharding      │ ✅ Redis Cluster  │ ✅ Client-side sharding││
│ │ Connection protocol   │ RESP             │ ASCII / binary       ││
│ └───────────────────────┴──────────────────┴──────────────────────┘│
│ * Redis 7+ uses I/O threads but command execution is single-threaded│
│                                                                       │
│ Decision tree:                                                      │
│                                                                       │
│ Need data structures beyond strings (hashes, sorted sets, lists)?  │
│ ├── YES → Redis ✅                                                 │
│ └── NO                                                             │
│     Need persistence or replication?                                │
│     ├── YES → Redis ✅                                             │
│     └── NO                                                         │
│         Need pub/sub or Lua scripting?                              │
│         ├── YES → Redis ✅                                         │
│         └── NO                                                     │
│             Pure key-value cache with simple GET/SET?               │
│             ├── YES → Memcached ✅ (simpler, multi-threaded)       │
│             └── Not sure → Redis ✅ (safer default)                │
│                                                                       │
│ ⚡ When in doubt, choose Redis. It does everything Memcached does  │
│    plus much more. Memcached only wins on simplicity and           │
│    multi-threaded efficiency for pure caching workloads.           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Memorystore for Redis

```
┌─────────────────────────────────────────────────────────────────────┐
│           MEMORYSTORE FOR REDIS — OVERVIEW                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Managed Redis 6.x and 7.x on GCP.                                  │
│                                                                       │
│ Fully compatible with open-source Redis — use any Redis client     │
│ library (redis-py, ioredis, Jedis, go-redis, etc.).                │
│                                                                       │
│ Key features:                                                       │
│ ├── Redis versions: 6.x, 7.x (configurable at creation)           │
│ ├── Instance sizes: 1 GB to 300 GB (Standard), 5 TB+ (Cluster)    │
│ ├── Tiers: Basic, Standard (HA), Cluster mode                      │
│ ├── Automatic failover: < 30 seconds (Standard tier)               │
│ ├── Read replicas: up to 5 (Standard tier)                         │
│ ├── RDB snapshots: automatic persistence to disk                   │
│ ├── AUTH: password-based authentication                            │
│ ├── TLS: in-transit encryption                                     │
│ ├── Maintenance window: configurable (no downtime for Standard)    │
│ ├── Scaling: resize online (up or down)                            │
│ └── Redis commands: nearly all supported (some admin commands      │
│     disabled — CONFIG, DEBUG, CLUSTER admin, etc.)                 │
│                                                                       │
│ Pricing (approximate, us-central1):                                │
│ ┌─────────────────────────┬──────────────────────────────────────┐ │
│ │ Tier                    │ Price per GB/hour                     │ │
│ ├─────────────────────────┼──────────────────────────────────────┤ │
│ │ Basic (M1)              │ $0.049/GB/hr (~$35/GB/month)         │ │
│ │ Standard (M1)           │ $0.068/GB/hr (~$49/GB/month)         │ │
│ │ Basic (M2 — higher perf)│ $0.074/GB/hr (~$53/GB/month)         │ │
│ │ Standard (M2)           │ $0.109/GB/hr (~$78/GB/month)         │ │
│ │ Network egress           │ Standard GCP rates                   │ │
│ └─────────────────────────┴──────────────────────────────────────┘ │
│                                                                       │
│ Examples:                                                           │
│ ├── 1 GB Basic M1:  ~$35/month                                    │
│ ├── 5 GB Standard M1: ~$245/month                                 │
│ ├── 16 GB Standard M2: ~$1,248/month                              │
│ └── No free tier (but 1 GB is the minimum size)                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Creating a Redis Instance

### Console Walkthrough

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONSOLE → Memorystore → Redis → Create Instance                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Instance Details                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Instance ID:      [my-redis-cache]                            │   │
│ │ Display name:     [Production Redis Cache]                    │   │
│ │                                                               │   │
│ │ Tier:                                                         │   │
│ │ ○ Basic — No replication, no HA, cheaper                     │   │
│ │ ○ Standard — Replica in another zone, automatic failover ✅  │   │
│ │                                                               │   │
│ │ → Standard for production, Basic for dev/test ✅              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 2: Capacity & Version                                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Capacity (GB):    [5]                                         │   │
│ │   Range: 1 GB to 300 GB                                      │   │
│ │   ⚠️ Can be resized later (online scaling)                   │   │
│ │                                                               │   │
│ │ Redis version:    [7.2]                                       │   │
│ │   Options: 6.x, 7.0, 7.2                                    │   │
│ │   → Choose latest (7.2) for new instances ✅                  │   │
│ │                                                               │   │
│ │ Machine tier:                                                 │   │
│ │ ○ M1 (shared-core) — Good for most workloads                │   │
│ │ ○ M2 (dedicated CPU) — Higher throughput, lower latency      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 3: Region & Network                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Region:            [us-central1]                              │   │
│ │ Zone:              [us-central1-a]                            │   │
│ │ Replica zone:      [us-central1-b] (Standard tier only)      │   │
│ │                                                               │   │
│ │ Authorized network: [default] (your VPC)                      │   │
│ │   ⚠️ Redis is ONLY accessible from within this VPC           │   │
│ │   ⚠️ No public IP (must use VPC or Serverless VPC Connector) │   │
│ │                                                               │   │
│ │ IP range:           [Auto-allocated] or [Manual CIDR]         │   │
│ │   Uses private service connection (/29 range)                │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 4: Security (optional)                                         │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ AUTH:                                                         │   │
│ │ □ Enable AUTH ✅                                               │   │
│ │   Generates a random password (or set your own)              │   │
│ │   Clients must send AUTH command before other commands        │   │
│ │                                                               │   │
│ │ In-transit encryption:                                        │   │
│ │ □ Enable TLS ✅                                                │   │
│ │   Encrypts data between client and Redis                     │   │
│ │   Uses server-side certificate (auto-managed)                │   │
│ │   ⚠️ Adds ~25% latency overhead                              │   │
│ │   ⚠️ Client must be configured for TLS connection            │   │
│ │                                                               │   │
│ │ CMEK:                                                         │   │
│ │ □ Customer-managed encryption key                             │   │
│ │   For at-rest encryption using your Cloud KMS key            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 5: Read Replicas (Standard tier only)                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Read replicas:     [0]                                        │   │
│ │   Range: 0 to 5                                              │   │
│ │   Each replica is in a different zone for HA                 │   │
│ │   Replicas serve read traffic (reduce primary load)          │   │
│ │   ⚠️ Adds cost (each replica = same size as primary)         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 6: Maintenance Window                                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Day:    [Sunday]                                              │   │
│ │ Hour:   [03:00 UTC]                                          │   │
│ │                                                               │   │
│ │ GCP applies patches/updates during this window.              │   │
│ │ Standard tier: zero-downtime updates (failover to replica).  │   │
│ │ Basic tier: brief downtime during maintenance.               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ [CREATE INSTANCE]                                                    │
│                                                                       │
│ ⚡ Instance creation takes ~3-5 minutes.                            │
│ ⚡ You'll get a private IP address to connect to.                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Redis Tiers — Basic vs Standard vs Cluster

```
┌─────────────────────────────────────────────────────────────────────┐
│           REDIS TIERS COMPARISON                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┬──────────────┬──────────────┬────────────────┐│
│ │ Feature          │ Basic        │ Standard     │ Cluster Mode   ││
│ ├──────────────────┼──────────────┼──────────────┼────────────────┤│
│ │ High availability│ ❌ No         │ ✅ Yes        │ ✅ Yes          ││
│ │ Replication      │ ❌ No         │ ✅ 1 replica  │ ✅ Per shard    ││
│ │ Automatic failover│ ❌ No        │ ✅ < 30 sec   │ ✅ < 30 sec    ││
│ │ Read replicas    │ ❌ No         │ ✅ 0-5        │ ✅ Per shard    ││
│ │ Persistence (RDB)│ ❌ No         │ ✅ Yes        │ ✅ Yes          ││
│ │ Max capacity     │ 300 GB       │ 300 GB       │ 5+ TB          ││
│ │ Scaling          │ ✅ Resize     │ ✅ Resize     │ ✅ Add shards   ││
│ │ SLA              │ ❌ None       │ 99.9%        │ 99.9%          ││
│ │ Cross-zone       │ ❌ Single zone│ ✅ Multi-zone │ ✅ Multi-zone   ││
│ │ Price            │ $              │ $$            │ $$$             ││
│ │ Use case         │ Dev, cache   │ Production   │ Large scale    ││
│ └──────────────────┴──────────────┴──────────────┴────────────────┘│
│                                                                       │
│ BASIC TIER:                                                         │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │  Zone A                                                       │   │
│ │  ┌──────────────┐                                             │   │
│ │  │   Primary    │                                             │   │
│ │  │   Redis      │  ← Single instance, no backup              │   │
│ │  │   5 GB       │  ← Data lost on failure                    │   │
│ │  └──────────────┘                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ STANDARD TIER:                                                      │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │  Zone A                    Zone B                             │   │
│ │  ┌──────────────┐         ┌──────────────┐                   │   │
│ │  │   Primary    │────────→│   Replica    │                   │   │
│ │  │   Redis      │  async  │   Redis      │                   │   │
│ │  │   5 GB       │  repl   │   5 GB       │                   │   │
│ │  └──────────────┘         └──────────────┘                   │   │
│ │  Reads + Writes           Failover target                    │   │
│ │                            + optional read endpoint           │   │
│ │                                                               │   │
│ │  Optional read replicas (zones C, D, E...):                  │   │
│ │  ┌──────────────┐  ┌──────────────┐                          │   │
│ │  │ Read Replica │  │ Read Replica │                          │   │
│ │  │ Zone C       │  │ Zone D       │                          │   │
│ │  └──────────────┘  └──────────────┘                          │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ CLUSTER MODE (Redis Cluster):                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Data is automatically sharded across multiple Redis nodes:   │   │
│ │                                                               │   │
│ │ ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │   │
│ │ │ Shard 0 │  │ Shard 1 │  │ Shard 2 │  │ Shard 3 │        │   │
│ │ │ Primary │  │ Primary │  │ Primary │  │ Primary │        │   │
│ │ │ +Replica│  │ +Replica│  │ +Replica│  │ +Replica│        │   │
│ │ │ Slots   │  │ Slots   │  │ Slots   │  │ Slots   │        │   │
│ │ │ 0-4095  │  │4096-8191│  │8192-12287│ │12288-16383│      │   │
│ │ └─────────┘  └─────────┘  └─────────┘  └─────────┘        │   │
│ │                                                               │   │
│ │ Advantages:                                                   │   │
│ │ ├── Horizontal scaling beyond 300 GB                         │   │
│ │ ├── Higher throughput (parallel across shards)               │   │
│ │ ├── Add/remove shards online                                 │   │
│ │ Limitations:                                                  │   │
│ │ ├── Multi-key operations must use hash tags {tag}            │   │
│ │ ├── Lua scripts limited to keys on same shard                │   │
│ │ └── Requires cluster-aware client library                    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Connecting to Redis

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONNECTING TO MEMORYSTORE REDIS                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Memorystore Redis has a PRIVATE IP only — no public access.        │
│ Your application must be in the same VPC (or use VPC connector).   │
│                                                                       │
│ After creation, Console shows:                                      │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Primary endpoint: 10.0.0.3:6379                               │   │
│ │ Read endpoint:    10.0.0.4:6379 (if read replicas exist)     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### From Compute Engine / GKE

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONNECT FROM GCE / GKE                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ VM or Pod must be in the same VPC as Memorystore.                  │
│                                                                       │
│ # Test connection from VM (install redis-cli)                       │
│ sudo apt-get install redis-tools                                    │
│ redis-cli -h 10.0.0.3 -p 6379                                      │
│                                                                       │
│ # With AUTH                                                         │
│ redis-cli -h 10.0.0.3 -p 6379 -a 'your-auth-string'               │
│                                                                       │
│ # With TLS                                                          │
│ redis-cli -h 10.0.0.3 -p 6378 --tls \                              │
│   --cacert /path/to/server-ca.pem                                   │
│                                                                       │
│ # Python example                                                    │
│ import redis                                                        │
│                                                                       │
│ r = redis.Redis(                                                    │
│     host='10.0.0.3',                                                │
│     port=6379,                                                      │
│     password='your-auth-string',  # if AUTH enabled                │
│     decode_responses=True                                           │
│ )                                                                   │
│                                                                       │
│ r.set('mykey', 'hello')                                             │
│ print(r.get('mykey'))  # 'hello'                                    │
│                                                                       │
│ # Node.js example                                                   │
│ const Redis = require('ioredis');                                    │
│ const client = new Redis({                                          │
│   host: '10.0.0.3',                                                 │
│   port: 6379,                                                       │
│   password: 'your-auth-string'                                      │
│ });                                                                 │
│                                                                       │
│ await client.set('mykey', 'hello');                                  │
│ const val = await client.get('mykey');  // 'hello'                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### From Cloud Run / Cloud Functions (Serverless)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONNECT FROM SERVERLESS (Cloud Run / Functions)             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Serverless services don't run in your VPC by default.              │
│ You need a Serverless VPC Access Connector to bridge.              │
│                                                                       │
│ ┌───────────────┐    ┌────────────────────┐    ┌──────────────┐   │
│ │ Cloud Run     │───→│ VPC Access         │───→│ Memorystore  │   │
│ │ / Functions   │    │ Connector          │    │ Redis        │   │
│ │ (serverless)  │    │ (in your VPC)      │    │ 10.0.0.3     │   │
│ └───────────────┘    └────────────────────┘    └──────────────┘   │
│                                                                       │
│ Setup:                                                              │
│ 1. Create VPC Access Connector:                                    │
│    gcloud compute networks vpc-access connectors create \           │
│      my-connector \                                                 │
│      --region=us-central1 \                                         │
│      --network=default \                                            │
│      --range=10.8.0.0/28 \                                          │
│      --min-instances=2 \                                            │
│      --max-instances=3                                               │
│                                                                       │
│ 2. Deploy Cloud Run with connector:                                │
│    gcloud run deploy my-api \                                       │
│      --image=gcr.io/my-proj/my-api \                                │
│      --vpc-connector=my-connector \                                 │
│      --set-env-vars=REDIS_HOST=10.0.0.3,REDIS_PORT=6379            │
│                                                                       │
│ 3. Deploy Cloud Function with connector:                           │
│    gcloud functions deploy my-function \                            │
│      --vpc-connector=my-connector \                                 │
│      --set-env-vars=REDIS_HOST=10.0.0.3,REDIS_PORT=6379            │
│                                                                       │
│ Cloud Run (Direct VPC — newer, no connector needed):               │
│    gcloud run deploy my-api \                                       │
│      --image=gcr.io/my-proj/my-api \                                │
│      --network=default \                                            │
│      --subnet=default \                                             │
│      --set-env-vars=REDIS_HOST=10.0.0.3                             │
│                                                                       │
│ ⚡ Direct VPC egress (Cloud Run) is faster and cheaper than         │
│    VPC Access Connectors. Use it for new deployments.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Redis Configuration & Tuning

```
┌─────────────────────────────────────────────────────────────────────┐
│           REDIS CONFIGURATION PARAMETERS                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Memorystore exposes a subset of Redis configuration parameters.    │
│ Set via Console → Instance → Configuration or gcloud.              │
│                                                                       │
│ Key configurable parameters:                                        │
│ ┌────────────────────────────┬──────────────────────────────────┐  │
│ │ Parameter                  │ Description & Recommendation       │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ maxmemory-policy           │ What happens when memory is full  │  │
│ │                            │ Default: volatile-lru             │  │
│ │                            │ Options:                          │  │
│ │                            │ • noeviction — return errors      │  │
│ │                            │ • allkeys-lru — evict any LRU ✅  │  │
│ │                            │ • volatile-lru — evict TTL keys   │  │
│ │                            │ • allkeys-random — random evict   │  │
│ │                            │ • volatile-ttl — evict soonest    │  │
│ │                            │   expiring                        │  │
│ │                            │ For cache: allkeys-lru ✅          │  │
│ │                            │ For session: noeviction            │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ notify-keyspace-events     │ Keyspace notification config      │  │
│ │                            │ Default: "" (disabled)            │  │
│ │                            │ "Ex" = expired events             │  │
│ │                            │ "Kg" = generic key events         │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ maxmemory-gb               │ Max usable memory                 │  │
│ │                            │ Default: instance size             │  │
│ │                            │ Can reduce for overhead room       │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ activedefrag               │ Active memory defragmentation     │  │
│ │                            │ Default: no                       │  │
│ │                            │ Enable if memory fragmentation    │  │
│ │                            │ ratio > 1.5                       │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ lfu-log-factor             │ LFU frequency counter factor      │  │
│ │ lfu-decay-time             │ LFU decay time in minutes         │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ lazyfree-lazy-eviction     │ Async eviction (non-blocking)     │  │
│ │ lazyfree-lazy-expire       │ Default: yes (Redis 7+)          │  │
│ │ lazyfree-lazy-server-del   │ Recommended: yes                  │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ stream-node-max-bytes      │ Max memory for stream nodes       │  │
│ │ stream-node-max-entries    │ Max entries per stream node        │  │
│ └────────────────────────────┴──────────────────────────────────┘  │
│                                                                       │
│ Update configuration:                                               │
│ gcloud redis instances update my-redis-cache \                      │
│   --region=us-central1 \                                            │
│   --update-redis-config=maxmemory-policy=allkeys-lru                │
│                                                                       │
│ ⚠️ Some config changes trigger a brief restart (Basic tier) or      │
│    failover (Standard tier). Check docs for specific parameters.   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Scaling (Online Resize)

```
┌─────────────────────────────────────────────────────────────────────┐
│           ONLINE SCALING                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ You can resize a Memorystore Redis instance without downtime        │
│ (Standard tier) or with brief downtime (Basic tier).               │
│                                                                       │
│ Scale up:                                                           │
│ gcloud redis instances update my-redis-cache \                      │
│   --region=us-central1 \                                            │
│   --size=10  # Resize from 5 GB to 10 GB                           │
│                                                                       │
│ Scale down:                                                         │
│ gcloud redis instances update my-redis-cache \                      │
│   --region=us-central1 \                                            │
│   --size=3  # Resize from 5 GB to 3 GB                             │
│                                                                       │
│ ⚠️ Scale down: current data must fit in new size                   │
│ ⚠️ Scaling takes several minutes for large instances               │
│ ⚠️ Standard tier: uses failover mechanism (near-zero downtime)     │
│ ⚠️ Basic tier: brief downtime during resize                        │
│                                                                       │
│ Tier upgrade:                                                       │
│ gcloud redis instances upgrade my-redis-cache \                     │
│   --region=us-central1                                              │
│ ⚠️ Basic → Standard upgrade: creates replica, brief downtime      │
│ ⚠️ Standard → Basic: NOT supported                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Redis Persistence & Backup

```
┌─────────────────────────────────────────────────────────────────────┐
│           REDIS PERSISTENCE (RDB SNAPSHOTS)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Standard tier Redis instances automatically persist data using     │
│ RDB (Redis Database Backup) snapshots.                              │
│                                                                       │
│ How it works:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Redis Instance                                               │   │
│ │  ┌──────────────┐                                             │   │
│ │  │ In-Memory    │                                             │   │
│ │  │ Data         │──── periodic ────→ ┌──────────┐            │   │
│ │  │              │     snapshot        │ RDB File │            │   │
│ │  │              │                     │ (on disk)│            │   │
│ │  └──────────────┘                     └──────────┘            │   │
│ │                                                               │   │
│ │  On restart/failover:                                         │   │
│ │  ┌──────────┐                     ┌──────────────┐            │   │
│ │  │ RDB File │──── load ──────────→│ In-Memory    │            │   │
│ │  │ (on disk)│                     │ Data         │            │   │
│ │  └──────────┘                     │ (restored)   │            │   │
│ │                                   └──────────────┘            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ RDB snapshot behavior:                                              │
│ ├── Automatic: Every ~12 hours (Standard tier)                     │
│ ├── On failover: RDB loaded on new primary                         │
│ ├── Data loss window: Up to ~12 hours of writes since last RDB     │
│ ├── Basic tier: NO persistence (all data lost on restart)          │
│ └── Cannot trigger manual snapshots                                │
│                                                                       │
│ ⚠️ Memorystore does NOT support AOF (Append-Only File).            │
│    If you need zero data loss, use Redis as cache only and         │
│    persist to Firestore/Cloud SQL/Spanner.                         │
│                                                                       │
│ Manual backups (Export/Import):                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Export RDB to Cloud Storage                                 │   │
│ │ gcloud redis instances export gs://my-bucket/redis-backup.rdb │   │
│ │   my-redis-cache --region=us-central1                         │   │
│ │                                                               │   │
│ │ # Import RDB from Cloud Storage                               │   │
│ │ gcloud redis instances import gs://my-bucket/redis-backup.rdb │   │
│ │   my-redis-cache --region=us-central1                         │   │
│ │                                                               │   │
│ │ ⚠️ Import overwrites ALL existing data!                      │   │
│ │ ⚠️ Instance is unavailable during import.                    │   │
│ │ ⚠️ RDB must be compatible with the Redis version.            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Redis AUTH & Security

```
┌─────────────────────────────────────────────────────────────────────┐
│           REDIS SECURITY                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. NETWORK SECURITY (first line of defense)                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ • Private IP ONLY — no public internet access                │   │
│ │ • Deployed in your VPC network                               │   │
│ │ • Firewall rules control which VMs/subnets can reach Redis   │   │
│ │ • VPC Service Controls for additional perimeter security     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. AUTH (password authentication)                                  │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ When enabled, clients must send AUTH before any command:      │   │
│ │                                                               │   │
│ │ redis-cli -h 10.0.0.3 -a 'my-auth-string'                    │   │
│ │                                                               │   │
│ │ Or in code:                                                   │   │
│ │ r = redis.Redis(host='10.0.0.3', password='my-auth-string')  │   │
│ │                                                               │   │
│ │ Enable AUTH on existing instance:                             │   │
│ │ gcloud redis instances update my-redis-cache \                │   │
│ │   --region=us-central1 \                                      │   │
│ │   --enable-auth                                               │   │
│ │                                                               │   │
│ │ Get the AUTH string:                                          │   │
│ │ gcloud redis instances get-auth-string my-redis-cache \       │   │
│ │   --region=us-central1                                        │   │
│ │                                                               │   │
│ │ ⚠️ AUTH string is auto-generated (not user-settable).        │   │
│ │ ⚠️ Rotate via: gcloud redis instances update ... --enable-auth│   │
│ │    (generates new auth string, old one still valid briefly)   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. IN-TRANSIT ENCRYPTION (TLS)                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Encrypts traffic between client and Redis.                    │   │
│ │ Uses port 6378 (not 6379).                                    │   │
│ │                                                               │   │
│ │ # Python with TLS                                             │   │
│ │ r = redis.Redis(                                              │   │
│ │     host='10.0.0.3',                                          │   │
│ │     port=6378,                                                │   │
│ │     password='auth-string',                                   │   │
│ │     ssl=True,                                                 │   │
│ │     ssl_ca_certs='/path/to/server-ca.pem'                     │   │
│ │ )                                                             │   │
│ │                                                               │   │
│ │ Download CA cert:                                             │   │
│ │ gcloud redis instances describe my-redis-cache \              │   │
│ │   --region=us-central1 \                                      │   │
│ │   --format='value(serverCaCerts[0].cert)' > server-ca.pem    │   │
│ │                                                               │   │
│ │ ⚠️ ~25% latency increase vs non-TLS.                         │   │
│ │ ⚠️ Must be enabled at instance creation (cannot add later    │   │
│ │    for some older versions).                                  │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 4. AT-REST ENCRYPTION                                              │
│ ├── Always encrypted with Google-managed key (default)             │
│ └── CMEK: Use your own Cloud KMS key (set at creation)             │
│                                                                       │
│ 5. IAM                                                              │
│ ├── roles/redis.admin — Full instance management                  │
│ ├── roles/redis.editor — Manage instances (not IAM)               │
│ └── roles/redis.viewer — Read-only instance info                  │
│ ⚠️ IAM controls who can manage the INSTANCE, not Redis data.      │
│    Redis AUTH controls data access.                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Memorystore for Memcached

```
┌─────────────────────────────────────────────────────────────────────┐
│           MEMORYSTORE FOR MEMCACHED                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Managed Memcached 1.5/1.6 on GCP.                                  │
│ Fully compatible with open-source Memcached protocol.               │
│                                                                       │
│ Key characteristics:                                                │
│ ├── Pure cache — NO persistence, NO replication                    │
│ ├── Multi-threaded — higher throughput per node than Redis         │
│ ├── Distributed across multiple nodes (client-side sharding)       │
│ ├── Simple GET/SET/DELETE operations                                │
│ ├── Max item size: 1 MB (configurable)                             │
│ ├── Automatic node discovery (clients discover all nodes)          │
│ ├── Scales by adding/removing nodes                                │
│ └── Cheaper than Redis for pure caching workloads                  │
│                                                                       │
│ Memcached architecture:                                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Memorystore Memcached Instance                               │   │
│ │  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │   │
│ │  │  Node 1  │  │  Node 2  │  │  Node 3  │                   │   │
│ │  │  1 vCPU  │  │  1 vCPU  │  │  1 vCPU  │                   │   │
│ │  │  1 GB    │  │  1 GB    │  │  1 GB    │                   │   │
│ │  └──────────┘  └──────────┘  └──────────┘                   │   │
│ │       ↑              ↑              ↑                         │   │
│ │       └──────────────┼──────────────┘                         │   │
│ │                      │                                        │   │
│ │              Client-side hashing                               │   │
│ │              (consistent hashing)                              │   │
│ │                      │                                        │   │
│ │                ┌─────────┐                                    │   │
│ │                │  Client │                                    │   │
│ │                └─────────┘                                    │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚠️ Node failure = data on that node is LOST (no replication).     │
│    Your application must handle cache misses gracefully.           │
│                                                                       │
│ Pricing (approximate):                                              │
│ ├── Per node: based on vCPU and memory chosen                      │
│ ├── Example: 3 nodes × 1 vCPU × 1 GB ≈ ~$30/month               │
│ └── Cheaper than Redis for equivalent memory                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Creating a Memcached Instance

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONSOLE → Memorystore → Memcached → Create Instance                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Instance Details                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Instance ID:      [my-memcached]                              │   │
│ │ Display name:     [API Cache Layer]                           │   │
│ │ Region:           [us-central1]                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 2: Node Configuration                                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Number of nodes:  [3] (min 1, max 20)                         │   │
│ │                                                               │   │
│ │ Per-node configuration:                                       │   │
│ │ • vCPUs:          [1] (1, 2, 4, 8, 16, 32)                  │   │
│ │ • Memory (GB):    [1] (1 to 256 per node)                    │   │
│ │                                                               │   │
│ │ Total capacity: 3 nodes × 1 GB = 3 GB total cache           │   │
│ │                                                               │   │
│ │ ⚠️ Node count and per-node config can be changed later.     │   │
│ │ ⚠️ Adding/removing nodes causes data redistribution          │   │
│ │    (some cache misses during rebalancing).                    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 3: Network                                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Authorized network: [default]                                 │   │
│ │   Same VPC requirement as Redis.                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 4: Memcached Configuration (optional)                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Memcached version:    [1.6.15]                                │   │
│ │ Max item size:        [1048576] bytes (1 MB default)         │   │
│ │ Connection limit:     [65000] per node                       │   │
│ │                                                               │   │
│ │ Custom parameters (params flag):                              │   │
│ │ • listen-backlog                                              │   │
│ │ • max-item-size                                               │   │
│ │ • slab-min-size                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ [CREATE]                                                             │
│                                                                       │
│ After creation:                                                     │
│ Discovery endpoint: 10.0.0.10:11211                                │
│ Node endpoints: 10.0.0.11:11211, 10.0.0.12:11211, 10.0.0.13:11211│
│                                                                       │
│ # Connect and test                                                  │
│ # Python                                                            │
│ from pymemcache.client.hash import HashClient                       │
│ client = HashClient([                                               │
│     ('10.0.0.11', 11211),                                           │
│     ('10.0.0.12', 11211),                                           │
│     ('10.0.0.13', 11211),                                           │
│ ])                                                                  │
│ client.set('mykey', 'hello')                                        │
│ print(client.get('mykey'))  # b'hello'                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Memorystore for Valkey

```
┌─────────────────────────────────────────────────────────────────────┐
│           MEMORYSTORE FOR VALKEY                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Valkey is an open-source fork of Redis created by the Linux        │
│ Foundation after Redis changed its license (2024).                  │
│ GCP Memorystore for Valkey is the newest offering.                 │
│                                                                       │
│ Key points:                                                         │
│ ├── API-compatible with Redis — same commands, same clients        │
│ ├── Open-source (BSD license) — no licensing concerns              │
│ ├── Managed by GCP just like Memorystore for Redis                 │
│ ├── Supports cluster mode (sharding) natively                      │
│ ├── Available in same regions as other Memorystore products        │
│ └── Recommended for new deployments if license matters             │
│                                                                       │
│ Valkey vs Redis on Memorystore:                                    │
│ ┌──────────────────────┬──────────────────┬──────────────────────┐│
│ │ Feature              │ Memorystore Redis│ Memorystore Valkey   ││
│ ├──────────────────────┼──────────────────┼──────────────────────┤│
│ │ Protocol compatible  │ Redis RESP       │ Redis RESP (same)    ││
│ │ Client libraries     │ All Redis clients│ All Redis clients    ││
│ │ License              │ Redis (SSPL)     │ BSD (open-source)    ││
│ │ Cluster mode         │ ✅ Yes            │ ✅ Yes                ││
│ │ Standard (HA) tier   │ ✅ Yes            │ ✅ Yes                ││
│ │ Persistence          │ ✅ RDB            │ ✅ RDB                ││
│ │ TLS / AUTH           │ ✅ Yes            │ ✅ Yes                ││
│ │ Community            │ Redis Inc.       │ Linux Foundation     ││
│ └──────────────────────┴──────────────────┴──────────────────────┘│
│                                                                       │
│ When to choose Valkey over Redis:                                  │
│ ├── New projects with no existing Redis dependency                 │
│ ├── Organizations concerned about Redis license (SSPL)             │
│ ├── Want to use an open-source governed project                    │
│ └── Functionally equivalent — choose based on licensing preference │
│                                                                       │
│ When to keep Redis:                                                │
│ ├── Existing Redis instances (no reason to migrate)                │
│ ├── Need specific Redis module (RediSearch, RedisJSON, etc.)       │
│ └── Prefer Redis-specific version features                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Monitoring & Maintenance

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Key metrics (Cloud Monitoring → Memorystore):                      │
│                                                                       │
│ Redis metrics:                                                      │
│ ┌────────────────────────────┬──────────────────────────────────┐  │
│ │ Metric                     │ Alert threshold                    │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ Memory usage ratio         │ > 80% → scale up or evict more  │  │
│ │ CPU seconds                │ > 80% → scale up or optimize    │  │
│ │ Connected clients          │ > 80% of max → investigate      │  │
│ │ Cache hit ratio            │ < 80% → review key patterns     │  │
│ │ Evicted keys               │ Increasing → need more memory   │  │
│ │ Keyspace misses            │ High ratio → cache warming issue │  │
│ │ Replication byte offset    │ Growing lag → network/load issue │  │
│ │ Commands processed/sec     │ Baseline awareness               │  │
│ │ Network bytes in/out       │ Bandwidth monitoring             │  │
│ │ Rejected connections       │ > 0 → connection limit reached  │  │
│ └────────────────────────────┴──────────────────────────────────┘  │
│                                                                       │
│ Memcached metrics:                                                  │
│ ┌────────────────────────────┬──────────────────────────────────┐  │
│ │ Metric                     │ Alert threshold                    │  │
│ ├────────────────────────────┼──────────────────────────────────┤  │
│ │ Memory usage ratio         │ > 80% → add nodes or evict      │  │
│ │ CPU utilization            │ > 80% → increase vCPU per node  │  │
│ │ Hit ratio                  │ < 80% → review patterns          │  │
│ │ Evictions                  │ Increasing → need more memory    │  │
│ │ Current connections        │ Near limit → scale               │  │
│ │ Commands per second        │ Baseline awareness               │  │
│ └────────────────────────────┴──────────────────────────────────┘  │
│                                                                       │
│ Recommended alerts:                                                 │
│ ├── Memory usage > 80% for 5 min                                  │
│ ├── Evicted keys > 0 per minute (critical for session stores)      │
│ ├── Cache hit ratio < 70%                                          │
│ ├── Connected clients > 90% of max                                 │
│ └── Replication lag increasing (Standard tier)                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Maintenance Windows

```
┌─────────────────────────────────────────────────────────────────────┐
│           MAINTENANCE WINDOWS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GCP applies Redis/Memcached patches during maintenance windows.    │
│                                                                       │
│ Configure: Console → Instance → Maintenance                        │
│ ├── Day of week: Sunday (recommended for low traffic)              │
│ └── Start hour: 03:00 UTC (recommended off-peak)                   │
│                                                                       │
│ Impact:                                                             │
│ ├── Standard tier Redis: Failover to replica → near-zero downtime │
│ ├── Basic tier Redis: Brief downtime (seconds to minutes)          │
│ ├── Memcached: Node-by-node rolling update                         │
│ └── Cannot skip or defer maintenance indefinitely                  │
│                                                                       │
│ Version upgrade:                                                    │
│ gcloud redis instances upgrade my-redis-cache \                     │
│   --region=us-central1 \                                            │
│   --redis-version=redis_7_2                                         │
│                                                                       │
│ ⚠️ Version upgrades are one-way — cannot downgrade.                │
│ ⚠️ Test in development instance first.                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 14: Terraform & CLI

### Terraform

```hcl
# ─────────────────────────────────────────────────────────────
# Memorystore for Redis (Standard Tier with HA)
# ─────────────────────────────────────────────────────────────

resource "google_redis_instance" "cache" {
  name           = "my-redis-cache"
  project        = "my-gcp-project"
  region         = "us-central1"
  tier           = "STANDARD_HA"  # BASIC or STANDARD_HA
  memory_size_gb = 5
  redis_version  = "REDIS_7_2"

  # Network
  authorized_network = "projects/my-gcp-project/global/networks/default"

  # Zone configuration
  location_id             = "us-central1-a"   # Primary zone
  alternative_location_id = "us-central1-b"   # Replica zone

  # AUTH
  auth_enabled = true

  # In-transit encryption
  transit_encryption_mode = "SERVER_AUTHENTICATION"

  # Configuration
  redis_configs = {
    maxmemory-policy = "allkeys-lru"
    notify-keyspace-events = "Ex"
  }

  # Read replicas
  replica_count     = 2
  read_replicas_mode = "READ_REPLICAS_ENABLED"

  # Maintenance window
  maintenance_policy {
    weekly_maintenance_window {
      day = "SUNDAY"
      start_time {
        hours   = 3
        minutes = 0
      }
    }
  }

  # Persistence (RDB)
  persistence_config {
    persistence_mode    = "RDB"
    rdb_snapshot_period = "TWELVE_HOURS"
  }

  labels = {
    environment = "production"
    team        = "backend"
  }
}

# Output the connection info
output "redis_host" {
  value = google_redis_instance.cache.host
}

output "redis_port" {
  value = google_redis_instance.cache.port
}

output "redis_auth_string" {
  value     = google_redis_instance.cache.auth_string
  sensitive = true
}

# ─────────────────────────────────────────────────────────────
# Memorystore for Redis (Basic — Development)
# ─────────────────────────────────────────────────────────────

resource "google_redis_instance" "dev_cache" {
  name           = "my-redis-dev"
  project        = "my-gcp-project"
  region         = "us-central1"
  tier           = "BASIC"
  memory_size_gb = 1
  redis_version  = "REDIS_7_2"

  authorized_network = "projects/my-gcp-project/global/networks/default"
  location_id        = "us-central1-a"

  redis_configs = {
    maxmemory-policy = "allkeys-lru"
  }

  labels = {
    environment = "development"
  }
}

# ─────────────────────────────────────────────────────────────
# Memorystore for Memcached
# ─────────────────────────────────────────────────────────────

resource "google_memcache_instance" "api_cache" {
  name   = "my-memcached"
  project = "my-gcp-project"
  region  = "us-central1"

  authorized_network = "projects/my-gcp-project/global/networks/default"

  node_config {
    cpu_count  = 1
    memory_size_mb = 1024  # 1 GB per node
  }

  node_count = 3  # 3 nodes × 1 GB = 3 GB total

  memcache_version = "MEMCACHE_1_6_15"

  memcache_parameters {
    params = {
      "listen-backlog" = "2048"
      "max-item-size"  = "2097152"  # 2 MB
    }
  }

  maintenance_policy {
    weekly_maintenance_window {
      day      = "SUNDAY"
      duration = "7200s"  # 2 hours
      start_time {
        hours   = 3
        minutes = 0
      }
    }
  }

  labels = {
    environment = "production"
  }
}

# Output Memcached connection info
output "memcached_discovery_endpoint" {
  value = google_memcache_instance.api_cache.discovery_endpoint
}

output "memcached_nodes" {
  value = google_memcache_instance.api_cache.memcache_nodes
}

# ─────────────────────────────────────────────────────────────
# VPC Access Connector (for Cloud Run / Functions)
# ─────────────────────────────────────────────────────────────

resource "google_vpc_access_connector" "redis_connector" {
  name          = "redis-connector"
  project       = "my-gcp-project"
  region        = "us-central1"
  network       = "default"
  ip_cidr_range = "10.8.0.0/28"

  min_instances = 2
  max_instances = 3
}
```

### gcloud CLI Reference

```bash
# ─────────────────────────────────────────────────────────────
# Redis Instance Management
# ─────────────────────────────────────────────────────────────

# Create Standard HA instance
gcloud redis instances create my-redis-cache \
  --region=us-central1 \
  --zone=us-central1-a \
  --alternative-zone=us-central1-b \
  --tier=STANDARD_HA \
  --size=5 \
  --redis-version=redis_7_2 \
  --network=default \
  --enable-auth \
  --transit-encryption-mode=SERVER_AUTHENTICATION \
  --redis-config=maxmemory-policy=allkeys-lru

# Create Basic instance (dev)
gcloud redis instances create my-redis-dev \
  --region=us-central1 \
  --tier=BASIC \
  --size=1 \
  --redis-version=redis_7_2

# List instances
gcloud redis instances list --region=us-central1

# Describe instance (shows IP, port, auth string)
gcloud redis instances describe my-redis-cache \
  --region=us-central1

# Get AUTH string
gcloud redis instances get-auth-string my-redis-cache \
  --region=us-central1

# Scale (resize memory)
gcloud redis instances update my-redis-cache \
  --region=us-central1 \
  --size=10

# Update configuration
gcloud redis instances update my-redis-cache \
  --region=us-central1 \
  --update-redis-config=maxmemory-policy=volatile-lru

# Add read replicas
gcloud redis instances update my-redis-cache \
  --region=us-central1 \
  --replica-count=2 \
  --read-replicas-mode=READ_REPLICAS_ENABLED

# Upgrade Redis version
gcloud redis instances upgrade my-redis-cache \
  --region=us-central1 \
  --redis-version=redis_7_2

# Failover (manual, Standard tier only)
gcloud redis instances failover my-redis-cache \
  --region=us-central1

# Export to GCS
gcloud redis instances export gs://my-bucket/redis-backup.rdb \
  my-redis-cache --region=us-central1

# Import from GCS
gcloud redis instances import gs://my-bucket/redis-backup.rdb \
  my-redis-cache --region=us-central1

# Delete instance
gcloud redis instances delete my-redis-cache \
  --region=us-central1

# ─────────────────────────────────────────────────────────────
# Memcached Instance Management
# ─────────────────────────────────────────────────────────────

# Create Memcached instance
gcloud memcache instances create my-memcached \
  --region=us-central1 \
  --node-count=3 \
  --node-cpu=1 \
  --node-memory=1024 \
  --memcached-version=1.6.15 \
  --authorized-network=default

# List instances
gcloud memcache instances list --region=us-central1

# Describe instance
gcloud memcache instances describe my-memcached \
  --region=us-central1

# Scale (add nodes)
gcloud memcache instances update my-memcached \
  --region=us-central1 \
  --node-count=5

# Update parameters
gcloud memcache instances update-parameters my-memcached \
  --region=us-central1 \
  --parameters="listen-backlog=2048,max-item-size=2097152"

# Apply software update
gcloud memcache instances apply-software-update my-memcached \
  --region=us-central1 \
  --node-ids=node-1,node-2,node-3

# Delete instance
gcloud memcache instances delete my-memcached \
  --region=us-central1
```

---

## Part 15: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 1: DATABASE QUERY CACHE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Reduce Cloud SQL load by caching frequent queries        │
│                                                                       │
│ Architecture:                                                       │
│ ┌─────────┐     ┌──────────────┐     ┌──────────────┐             │
│ │ Cloud   │────→│    Redis     │     │  Cloud SQL   │             │
│ │ Run     │     │  (5 GB, HA)  │     │  (PostgreSQL)│             │
│ │ API     │     │              │     │              │             │
│ │         │←────│  allkeys-lru │     │              │             │
│ └─────────┘     └──────────────┘     └──────────────┘             │
│      │                                      ↑                      │
│      │          Cache miss?                 │                      │
│      └──────────────────────────────────────┘                      │
│                  Query DB → store in Redis                          │
│                                                                       │
│ Code pattern (cache-aside):                                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ async function getUser(userId) {                              │   │
│ │   // 1. Check cache first                                     │   │
│ │   const cached = await redis.get(`user:${userId}`);           │   │
│ │   if (cached) return JSON.parse(cached);  // Cache HIT       │   │
│ │                                                               │   │
│ │   // 2. Cache miss — query database                          │   │
│ │   const user = await db.query('SELECT * FROM users WHERE ...'│   │
│ │   );                                                          │   │
│ │                                                               │   │
│ │   // 3. Store in cache with TTL                              │   │
│ │   await redis.set(`user:${userId}`, JSON.stringify(user),     │   │
│ │     'EX', 300);  // 5 minute TTL                             │   │
│ │                                                               │   │
│ │   return user;                                                │   │
│ │ }                                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Setup:                                                              │
│ ├── Redis Standard HA, 5 GB, allkeys-lru eviction                  │
│ ├── TTL on all cached keys (300s-3600s depending on freshness)     │
│ ├── VPC connector for Cloud Run → Redis                            │
│ ├── AUTH enabled, TLS optional (internal traffic)                   │
│ └── Monitor: hit ratio > 85% target, evictions should be rare      │
│                                                                       │
│ Cost: ~$245/month (5 GB Standard M1)                                │
│ Savings: Reduces Cloud SQL CPU by 60-80% typically                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 2: SESSION STORE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Stateless web app needs shared session storage           │
│                                                                       │
│ Architecture:                                                       │
│ ┌────────┐    ┌────────┐    ┌────────┐                             │
│ │ GKE    │    │ GKE    │    │ GKE    │    ← Stateless pods         │
│ │ Pod 1  │    │ Pod 2  │    │ Pod 3  │                             │
│ └───┬────┘    └───┬────┘    └───┬────┘                             │
│     │             │             │                                   │
│     └─────────────┼─────────────┘                                  │
│                   ▼                                                 │
│          ┌──────────────────┐                                      │
│          │ Redis (Standard) │  ← Session data, TTL = 30 min       │
│          │ 2 GB, HA         │  ← noeviction policy!               │
│          │ AUTH + TLS       │                                      │
│          └──────────────────┘                                      │
│                                                                       │
│ Key design:                                                         │
│ ├── Key: "session:{session_id}" → Hash with session fields        │
│ ├── TTL: 30 minutes (sliding — refreshed on each request)          │
│ ├── maxmemory-policy: noeviction (don't lose sessions!)            │
│ ├── AUTH + TLS enabled (sessions contain user data)                │
│ └── Standard tier for HA (user sessions must survive failover)     │
│                                                                       │
│ Redis commands used:                                                │
│ ├── HSET session:abc123 userId "user_42" role "admin"              │
│ ├── HGETALL session:abc123                                         │
│ ├── EXPIRE session:abc123 1800  (30 min sliding TTL)               │
│ └── DEL session:abc123  (logout)                                   │
│                                                                       │
│ ⚠️ Use noeviction — if memory is full, return errors rather       │
│    than silently evicting active sessions!                          │
│                                                                       │
│ Cost: ~$98/month (2 GB Standard M1)                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 3: RATE LIMITER                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: API rate limiting across distributed backend instances   │
│                                                                       │
│ Architecture:                                                       │
│ ┌─────────┐     ┌────────────┐     ┌──────────────┐               │
│ │ Client  │────→│ Cloud Run  │────→│  Redis       │               │
│ │         │     │ (API)      │     │  (1 GB, HA)  │               │
│ │         │     │            │     │              │               │
│ │ 429?    │←────│ Check rate │←────│ INCR + TTL   │               │
│ └─────────┘     └────────────┘     └──────────────┘               │
│                                                                       │
│ Implementation (sliding window counter):                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ async function checkRateLimit(clientIP, limit = 100) {        │   │
│ │   const key = `rate:${clientIP}:${currentMinute()}`;          │   │
│ │   const count = await redis.incr(key);                        │   │
│ │   if (count === 1) {                                          │   │
│ │     await redis.expire(key, 60);  // TTL = 1 minute          │   │
│ │   }                                                           │   │
│ │   return count <= limit;  // true = allowed, false = blocked │   │
│ │ }                                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Why Redis excels here:                                              │
│ ├── INCR is atomic — no race conditions across instances          │
│ ├── TTL auto-cleans old windows                                    │
│ ├── Sub-millisecond latency — no request delay                    │
│ └── Shared state across all Cloud Run instances                    │
│                                                                       │
│ Cost: ~$35/month (1 GB Basic is fine — rate limit data is          │
│        ephemeral, loss on failure is acceptable)                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 4: REAL-TIME LEADERBOARD                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Redis Sorted Sets are perfect for leaderboards.                    │
│                                                                       │
│ Commands:                                                           │
│ ├── ZADD leaderboard 1500 "player_42"    (set/update score)      │
│ ├── ZREVRANGE leaderboard 0 9 WITHSCORES (top 10 players)        │
│ ├── ZRANK leaderboard "player_42"         (player's rank)         │
│ ├── ZREVRANK leaderboard "player_42"      (rank from top)         │
│ ├── ZINCRBY leaderboard 10 "player_42"   (add to score)          │
│ └── ZCARD leaderboard                     (total players)         │
│                                                                       │
│ ⚡ All operations are O(log N) — works with millions of players.   │
│ ⚡ Real-time updates — no pre-computation needed.                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           MEMORYSTORE QUICK REFERENCE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Engines: Redis, Memcached, Valkey                                  │
│                                                                       │
│ Redis tiers:                                                        │
│ ├── Basic: Single node, no HA, no persistence, cheapest           │
│ ├── Standard HA: Replica, auto-failover, RDB persistence, SLA     │
│ └── Cluster: Sharded, horizontal scaling, 5+ TB capacity          │
│                                                                       │
│ Memcached: Pure cache, multi-threaded, no persistence, no HA       │
│                                                                       │
│ Network: Private IP ONLY (VPC). Use VPC connector for serverless.  │
│                                                                       │
│ Redis key configs:                                                  │
│ ├── maxmemory-policy: allkeys-lru (cache), noeviction (sessions)  │
│ ├── AUTH: auto-generated password (enable for security)            │
│ ├── TLS: port 6378, adds ~25% latency                             │
│ └── Persistence: RDB snapshots every 12h (Standard tier only)      │
│                                                                       │
│ Scaling:                                                            │
│ ├── Redis: Resize memory online (1-300 GB), add read replicas     │
│ ├── Memcached: Add/remove nodes (1-20), change per-node config    │
│ └── Both: Near-zero downtime on Standard/HA tiers                  │
│                                                                       │
│ Pricing (approximate):                                              │
│ ├── Redis Basic 1 GB: ~$35/month                                  │
│ ├── Redis Standard 5 GB: ~$245/month                              │
│ ├── Memcached 3 nodes × 1 GB: ~$30/month                          │
│ └── No free tier                                                   │
│                                                                       │
│ Common patterns:                                                    │
│ ├── Cache-aside (query cache)                                      │
│ ├── Session store (noeviction + TTL)                               │
│ ├── Rate limiting (INCR + EXPIRE)                                  │
│ ├── Leaderboards (Sorted Sets)                                     │
│ ├── Pub/Sub (real-time notifications)                              │
│ └── Distributed locks (SETNX)                                      │
│                                                                       │
│ CLI tools:                                                          │
│ ├── gcloud redis instances ... (Redis management)                  │
│ ├── gcloud memcache instances ... (Memcached management)           │
│ └── redis-cli (connect and test)                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Managing & Deleting Instances

```
┌─────────────────────────────────────────────────────────────────────┐
│           SCALING A REDIS INSTANCE FROM CONSOLE                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ CONSOLE → Memorystore → Redis → Click instance name                │
│                                                                       │
│ Scale Memory (resize):                                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ 1. Click "Edit" at the top of the instance details page      │   │
│ │ 2. Change "Capacity (GB)" to new value (1–300 GB)            │   │
│ │    ⚠️ You can scale UP or DOWN                               │   │
│ │ 3. Click "Save"                                              │   │
│ │                                                               │   │
│ │ What happens:                                                 │   │
│ │ ├── Standard tier: near-zero downtime (failover to replica)  │   │
│ │ ├── Basic tier: brief downtime during resize                 │   │
│ │ └── Data is preserved during scaling                         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Change Tier (Basic → Standard):                                    │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ 1. Click "Edit" on the instance details page                 │   │
│ │ 2. Change tier from Basic to Standard (or vice versa)        │   │
│ │    ⚠️ Basic → Standard: adds replica, enables HA             │   │
│ │    ⚠️ Standard → Basic: removes replica, loses HA            │   │
│ │ 3. Click "Save"                                              │   │
│ │                                                               │   │
│ │ ⚡ Tier change may cause brief downtime.                     │   │
│ │ ⚡ Plan tier changes during low-traffic windows.              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           UPDATING CONFIGURATION FROM CONSOLE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ CONSOLE → Memorystore → Redis → Click instance → "Edit"            │
│                                                                       │
│ What you can update:                                                │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ├── Display name                                              │   │
│ │ ├── Memory size (scale up or down)                            │   │
│ │ ├── Redis configuration parameters:                           │   │
│ │ │   ├── maxmemory-policy (e.g., allkeys-lru, noeviction)     │   │
│ │ │   ├── notify-keyspace-events                                │   │
│ │ │   ├── activedefrag                                          │   │
│ │ │   └── Other supported Redis configs                        │   │
│ │ ├── Maintenance window (day and hour)                         │   │
│ │ ├── Read replicas count (Standard tier, 0–5)                  │   │
│ │ └── AUTH password (regenerate or disable)                     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ What you CANNOT change after creation:                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ├── Instance ID                                               │   │
│ │ ├── Region / Zone                                             │   │
│ │ ├── Authorized network (VPC)                                  │   │
│ │ ├── Redis version (must create new instance)                  │   │
│ │ └── IP range allocation                                      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Configuration changes apply immediately.                        │
│ ⚡ Some config changes may briefly restart the Redis process.      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           DELETING A REDIS INSTANCE                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ From Console:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ 1. CONSOLE → Memorystore → Redis                             │   │
│ │ 2. Check the box next to the instance you want to delete     │   │
│ │ 3. Click "Delete" at the top of the list                     │   │
│ │ 4. Confirm by typing the instance name                       │   │
│ │ 5. Click "Delete"                                            │   │
│ │                                                               │   │
│ │ ⚠️ This action is PERMANENT — all data is lost!              │   │
│ │ ⚠️ There is no "undo" or recycle bin for Memorystore.        │   │
│ │ ⚠️ Export your data (RDB snapshot) before deleting if needed.│   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ From gcloud CLI:                                                    │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Delete a Redis instance                                    │   │
│ │ gcloud redis instances delete my-redis-cache \               │   │
│ │   --region=us-central1                                        │   │
│ │                                                               │   │
│ │ # You'll be prompted to confirm:                              │   │
│ │ # "You are about to delete instance [my-redis-cache].        │   │
│ │ #  Do you want to continue (Y/n)?"                           │   │
│ │                                                               │   │
│ │ # Skip confirmation (scripts/automation):                    │   │
│ │ gcloud redis instances delete my-redis-cache \               │   │
│ │   --region=us-central1 \                                      │   │
│ │   --quiet                                                     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Deletion takes ~1-2 minutes.                                    │
│ ⚡ You stop being billed immediately after deletion completes.     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Cache Patterns for Beginners

```
┌─────────────────────────────────────────────────────────────────────┐
│           CACHE-ASIDE PATTERN (Most Common)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Also called "Lazy Loading" — the application manages the cache.    │
│                                                                       │
│ How it works:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  App Request: "Get user #42"                                  │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  ┌─────────────┐   1. Check cache    ┌──────────────┐        │   │
│ │  │ Application │───────────────────→│    Redis     │        │   │
│ │  │             │                     │   (Cache)    │        │   │
│ │  │             │   2a. Cache HIT ✅   │              │        │   │
│ │  │             │←───────────────────│  Return data │        │   │
│ │  │             │                     └──────────────┘        │   │
│ │  │             │                                              │   │
│ │  │             │   2b. Cache MISS ❌                           │   │
│ │  │             │                     ┌──────────────┐        │   │
│ │  │             │   3. Query DB  ────→│  Cloud SQL   │        │   │
│ │  │             │←───────────────────│  (Database)  │        │   │
│ │  │             │   4. Get data        └──────────────┘        │   │
│ │  │             │                                              │   │
│ │  │             │   5. Write to cache  ┌──────────────┐        │   │
│ │  │             │───────────────────→│    Redis     │        │   │
│ │  │             │   (with TTL)        │   (Cache)    │        │   │
│ │  └─────────────┘                     └──────────────┘        │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Advantages:                                                         │
│ ├── Only requested data is cached (no wasted memory)               │
│ ├── Cache failures don't break the app (fallback to DB)            │
│ └── Simple to implement                                            │
│                                                                       │
│ Disadvantages:                                                      │
│ ├── First request is always slow (cache miss → DB query)           │
│ ├── Data can become stale (until TTL expires)                      │
│ └── "Thundering herd" — many requests for same missing key         │
│                                                                       │
│ ⚡ Best for: Read-heavy workloads, data that doesn't change often. │
│ ⚡ This is the #1 pattern you'll use with Memorystore.             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           WRITE-THROUGH PATTERN                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Write to cache AND database at the same time.                      │
│ Cache is always up-to-date — no stale data.                        │
│                                                                       │
│ How it works:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  App: "Update user #42 name to Alice"                         │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  ┌─────────────┐                                              │   │
│ │  │ Application │──┬── 1. Write to DB ───→ ┌──────────────┐   │   │
│ │  │             │  │                        │  Cloud SQL   │   │   │
│ │  │             │  │                        └──────────────┘   │   │
│ │  │             │  │                                           │   │
│ │  │             │  └── 2. Write to Cache ──→ ┌──────────────┐  │   │
│ │  │             │                            │    Redis     │  │   │
│ │  └─────────────┘                            └──────────────┘  │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Advantages:                                                         │
│ ├── Cache is always fresh (no stale data)                          │
│ ├── Reads are always fast (data is in cache)                       │
│ └── No cache miss penalty on subsequent reads                      │
│                                                                       │
│ Disadvantages:                                                      │
│ ├── Write latency increases (two writes: DB + cache)               │
│ ├── Cache fills with data that may never be read                   │
│ └── Higher complexity (must handle write failures)                 │
│                                                                       │
│ ⚡ Best for: Data that's read immediately after writing,           │
│    or when you can't tolerate stale data.                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Python Cache-Aside Example

```python
import redis
import json
import psycopg2  # or any DB library (mysql, sqlalchemy, etc.)

# ─────────────────────────────────────────────────────────────
# Connect to Memorystore Redis
# ─────────────────────────────────────────────────────────────
cache = redis.Redis(
    host='10.0.0.3',       # Memorystore private IP
    port=6379,
    password='your-auth-string',  # if AUTH enabled
    decode_responses=True
)

# ─────────────────────────────────────────────────────────────
# Connect to Cloud SQL (your primary database)
# ─────────────────────────────────────────────────────────────
db = psycopg2.connect(
    host='10.0.1.5',
    database='myapp',
    user='appuser',
    password='db-password'
)

# ─────────────────────────────────────────────────────────────
# Cache-Aside: Get user by ID
# ─────────────────────────────────────────────────────────────
def get_user(user_id):
    cache_key = f"user:{user_id}"

    # Step 1: Check cache first
    cached = cache.get(cache_key)
    if cached:
        print(f"Cache HIT for {cache_key}")
        return json.loads(cached)  # Return cached data

    # Step 2: Cache miss — query the database
    print(f"Cache MISS for {cache_key} — querying DB")
    cursor = db.cursor()
    cursor.execute("SELECT id, name, email FROM users WHERE id = %s", (user_id,))
    row = cursor.fetchone()

    if row is None:
        return None

    user = {"id": row[0], "name": row[1], "email": row[2]}

    # Step 3: Store in cache with TTL (e.g., 5 minutes)
    cache.setex(
        cache_key,
        300,                # TTL in seconds (5 minutes)
        json.dumps(user)    # Serialize to JSON string
    )

    return user

# ─────────────────────────────────────────────────────────────
# Usage
# ─────────────────────────────────────────────────────────────
user = get_user(42)       # First call: cache MISS → DB query → cache write
user = get_user(42)       # Second call: cache HIT → fast!
print(user)               # {"id": 42, "name": "Alice", "email": "alice@example.com"}
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           CACHE INVALIDATION STRATEGIES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ "There are only two hard things in computer science:               │
│  cache invalidation and naming things." — Phil Karlton             │
│                                                                       │
│ Strategy 1: TTL-Based Expiry (simplest)                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ cache.setex("user:42", 300, data)   # Expires in 5 minutes   │   │
│ │                                                               │   │
│ │ ✅ Simple — set it and forget it                              │   │
│ │ ✅ Stale data is automatically cleaned up                     │   │
│ │ ❌ Data can be stale until TTL expires                        │   │
│ │                                                               │   │
│ │ Good TTL values:                                              │   │
│ │ ├── Product catalog: 1–6 hours (changes infrequently)        │   │
│ │ ├── User profile: 5–15 minutes (moderate changes)            │   │
│ │ ├── API response: 30–60 seconds (changes often)              │   │
│ │ └── Session data: 30 minutes (matches session timeout)       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Strategy 2: Explicit Deletion (on write)                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ def update_user(user_id, new_data):                           │   │
│ │     # Update database                                        │   │
│ │     db.execute("UPDATE users SET ... WHERE id = %s",         │   │
│ │                (user_id,))                                    │   │
│ │     # Delete stale cache entry                               │   │
│ │     cache.delete(f"user:{user_id}")                          │   │
│ │     # Next read will trigger a cache miss → fresh data       │   │
│ │                                                               │   │
│ │ ✅ Cache is never stale after an update                       │   │
│ │ ✅ Combine with TTL as a safety net                           │   │
│ │ ❌ Requires delete logic everywhere you write data            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Strategy 3: Event-Driven Invalidation (advanced)                   │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │ DB Update ──→ Pub/Sub Message ──→ Cache Subscriber           │   │
│ │                                     (deletes stale keys)     │   │
│ │                                                               │   │
│ │ ✅ Decoupled — write code doesn't know about cache           │   │
│ │ ✅ Works across multiple services                             │   │
│ │ ❌ More complex to set up                                     │   │
│ │ ❌ Slight delay between DB write and cache invalidation       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Beginner recommendation:                                        │
│   Use TTL + Explicit Deletion together.                            │
│   TTL ensures stale data eventually expires (safety net).          │
│   Explicit deletion ensures immediate freshness on writes.         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 29: AlloyDB** → `29-alloydb.md`
