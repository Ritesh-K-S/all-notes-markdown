# Chapter 30: Azure Cache for Redis

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Redis Cache Fundamentals](#part-1-redis-cache-fundamentals)
- [Part 2: Creating Azure Cache for Redis (Portal Walkthrough)](#part-2-creating-azure-cache-for-redis-portal-walkthrough)
- [Part 3: Tiers & Sizing](#part-3-tiers--sizing)
- [Part 4: Common Caching Patterns](#part-4-common-caching-patterns)
- [Part 5: Clustering & Geo-Replication](#part-5-clustering--geo-replication)
- [Part 6: Persistence & Backup](#part-6-persistence--backup)
- [Part 7: Terraform & Bicep](#part-7-terraform--bicep)
- [Part 8: az CLI Reference](#part-8-az-cli-reference)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Cache for Redis is a fully managed in-memory data store based on the open-source Redis software. It provides sub-millisecond response times — much faster than any database. Think of it as a super-fast "lookup table" in memory that sits between your app and database, so you don't hit the database for every request.

```
What you'll learn:
├── Redis Cache Fundamentals
│   ├── What is Redis (in-memory key-value store)
│   ├── Why caching matters (speed + cost reduction)
│   ├── Common data structures (strings, hashes, sets, lists)
│   └── Redis vs Cosmos DB vs SQL
├── Creating Azure Cache for Redis (Portal)
├── Tiers (Basic, Standard, Premium, Enterprise)
├── Caching Patterns (cache-aside, write-through, etc.)
├── Clustering & Geo-Replication
├── Persistence & Backup
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: Redis Cache Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           REDIS CACHE CONCEPT                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Redis?                                                       │
│ ├── In-memory key-value data store                                 │
│ ├── Sub-millisecond read/write latency (<1ms!)                   │
│ ├── Data stored in RAM (not disk — that's why it's fast)        │
│ ├── Supports data structures: strings, hashes, lists, sets, etc.│
│ ├── Used as: cache, session store, message broker, leaderboard  │
│ └── Fully managed by Azure (patching, HA, monitoring)           │
│                                                                       │
│ Why use caching?                                                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ WITHOUT cache:                                               │  │
│ │ User → App → Database (50-100ms per query)                 │  │
│ │ 1000 users = 1000 database queries = SLOW + EXPENSIVE      │  │
│ │                                                              │  │
│ │ WITH cache:                                                  │  │
│ │ User → App → Redis (<1ms) → Database (only on cache miss)│  │
│ │ 1000 users = 950 cache hits + 50 database queries = FAST!  │  │
│ │                                                              │  │
│ │ ⚡ Cache hit ratio of 95% → 20x fewer database calls!      │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Common use cases:                                                    │
│ ├── Session storage (user sessions for web apps)                 │
│ ├── Database query caching (cache expensive query results)      │
│ ├── API response caching (cache repeated API responses)         │
│ ├── Rate limiting (count requests per user per minute)          │
│ ├── Leaderboards/rankings (sorted sets)                         │
│ ├── Real-time messaging (pub/sub)                                │
│ └── Distributed locks (prevent race conditions)                 │
│                                                                       │
│ ⚡ AWS equivalent: Amazon ElastiCache for Redis                     │
│ ⚡ GCP equivalent: Memorystore for Redis                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating Azure Cache for Redis (Portal Walkthrough)

```
Console → Azure Cache for Redis → Create

┌─────────────────────────────────────────────────────────────────────┐
│           BASICS                                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-cache-prod ▼]                                  │
│                                                                       │
│ DNS name: [redis-myapp-prod]                                       │
│ ⚡ Creates: redis-myapp-prod.redis.cache.windows.net              │
│                                                                       │
│ Location: [Central India ▼]                                        │
│                                                                       │
│ Cache SKU:                                                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ○ Basic (single node, no SLA — dev/test only)              │  │
│ │ ● Standard (replicated, 2 nodes — production)              │  │
│ │ ○ Premium (clustering, VNet, persistence, geo-replication) │  │
│ │ ○ Enterprise (Redis Enterprise modules)                    │  │
│ │ ○ Enterprise Flash (large datasets, SSD-backed)            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Cache size: [C1 Standard (1 GB) ▼]                                │
│ ┌───────┬──────────┬──────────┬──────────────────────────────┐   │
│ │ Size  │ Memory   │ Max Conn │ Monthly Cost (approx)        │   │
│ ├───────┼──────────┼──────────┼──────────────────────────────┤   │
│ │ C0    │ 250 MB   │ 256      │ ~$16 (Basic) / $42 (Std)    │   │
│ │ C1    │ 1 GB     │ 1,000    │ ~$42 (Basic) / $84 (Std)    │   │
│ │ C2    │ 2.5 GB   │ 2,000    │ ~$80 (Basic) / $160 (Std)   │   │
│ │ C3    │ 6 GB     │ 5,000    │ ~$155 (Basic) / $310 (Std)  │   │
│ │ C4    │ 13 GB    │ 10,000   │ ~$310 / $620                 │   │
│ │ C5    │ 26 GB    │ 15,000   │ ~$610 / $1220                │   │
│ │ C6    │ 53 GB    │ 20,000   │ ~$1220 / $2440               │   │
│ └───────┴──────────┴──────────┴──────────────────────────────┘   │
│                                                                       │
│ ── Networking ──                                                    │
│ ○ Public endpoint                                                  │
│ ● Private endpoint                                                 │
│ [+ Add private endpoint]                                           │
│                                                                       │
│ ── Advanced ──                                                      │
│ Redis version: [6 ▼]                                              │
│ Non-TLS port (6379): ● Disabled (always use TLS!)               │
│ Microsoft Entra Authentication: ☑ Enable                         │
│ ⚡ Use Entra + managed identity (no connection strings in code!) │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Tiers & Sizing

```
┌─────────────────────────────────────────────────────────────────────┐
│           TIERS COMPARISON                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────┬──────┬──────┬───────┬────────────┬────────────┐ │
│ │ Feature       │Basic │ Std  │Premium│ Enterprise │ Ent. Flash │ │
│ ├──────────────┼──────┼──────┼───────┼────────────┼────────────┤ │
│ │ SLA           │ None │99.9% │99.9% │ 99.99%     │ 99.99%    │ │
│ │ Replication   │ No   │ Yes  │ Yes  │ Yes        │ Yes        │ │
│ │ Clustering    │ No   │ No   │ Yes  │ Yes        │ Yes        │ │
│ │ VNet          │ No   │ No   │ Yes  │ Yes        │ Yes        │ │
│ │ Persistence   │ No   │ No   │ Yes  │ Yes        │ Yes        │ │
│ │ Geo-replication│No   │ No   │ Yes  │ Yes (active)│ Yes       │ │
│ │ Max size      │53 GB │53 GB │120 GB│ 2 TB       │ 4.5 TB    │ │
│ │ Zone redundancy│No   │ No   │ Yes  │ Yes        │ Yes        │ │
│ │ Modules       │ No   │ No   │ No   │ Yes ✅    │ Yes        │ │
│ └──────────────┴──────┴──────┴───────┴────────────┴────────────┘ │
│                                                                       │
│ Decision guide:                                                      │
│ ├── Dev/test → Basic C0/C1 (cheapest)                            │
│ ├── Production → Standard C1-C3 (replicated, SLA)              │
│ ├── Enterprise → Premium (VNet, clustering, geo-replication)   │
│ ├── Huge data → Enterprise Flash (SSD + RAM, up to 4.5 TB)    │
│ └── Redis modules (RediSearch, RedisJSON) → Enterprise         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Common Caching Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           CACHING PATTERNS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Cache-Aside (Lazy Loading) — most common!                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ App: "I need user #123"                                     │  │
│ │ 1. Check Redis → cache MISS (not found)                   │  │
│ │ 2. Query Database → get user #123                          │  │
│ │ 3. Store in Redis: SET user:123 "{name:John}" EX 300      │  │
│ │ 4. Return data to user                                      │  │
│ │                                                              │  │
│ │ Next request for user #123:                                  │  │
│ │ 1. Check Redis → cache HIT (found!)                       │  │
│ │ 2. Return data from Redis (<1ms)                           │  │
│ │ 3. Database NOT called (saved a query!)                    │  │
│ │                                                              │  │
│ │ EX 300 = expire after 300 seconds (5 minutes)              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 2. Write-Through                                                     │
│    Write to cache AND database simultaneously.                     │
│    Ensures cache is always up to date.                              │
│    Downside: Slower writes (2 writes per operation).               │
│                                                                       │
│ 3. Write-Behind (Write-Back)                                        │
│    Write to cache first, then asynchronously to database.          │
│    Fastest writes but risk of data loss if cache fails.            │
│                                                                       │
│ 4. Session Store                                                     │
│    Store user sessions in Redis (shared across app instances).     │
│    SET session:abc123 "{userId:1,cart:[...]}" EX 3600             │
│                                                                       │
│ ⚡ Cache-Aside is the most common and safest pattern!              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Clustering & Geo-Replication

```
Clustering (Premium tier):
├── Shard data across multiple nodes (up to 10 shards)
├── Each shard holds a portion of the keys
├── Total memory = shard count × node size
│   Example: 10 shards × 6 GB = 60 GB total cache
├── Higher throughput (each shard handles requests independently)
└── Portal → Redis Cache → Scale → Shard count

Geo-Replication (Premium/Enterprise):
├── Passive geo-replication (Premium):
│   Primary cache in Region A → linked to secondary in Region B
│   Secondary is read-only, async replication
│   Manual failover if primary fails
│
├── Active geo-replication (Enterprise):
│   Both caches can read AND write (active-active)
│   Conflict resolution: Last Writer Wins
│   Multi-region with low latency
│
└── Portal → Redis Cache → Geo-replication → Add link
```

---

## Part 6: Persistence & Backup

```
Data Persistence (Premium/Enterprise):
├── RDB persistence (snapshots):
│   Periodic snapshots of data saved to storage
│   Frequency: every 15min, 30min, 60min, 6h, 12h, 24h
│   Data survives node restarts
│
├── AOF persistence (append-only file):
│   Every write operation logged to storage
│   More durable but higher disk I/O
│   Better for "can't lose any data" scenarios
│
└── Enable: Portal → Redis Cache → Data persistence

Export/Import:
├── Export cache to blob storage (backup)
├── Import from blob storage (restore)
└── Useful for migration between caches
```

---

## Part 7: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_redis_cache" "main" {
  name                = "redis-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  capacity            = 1
  family              = "C"
  sku_name            = "Standard"
  minimum_tls_version = "1.2"
  enable_non_ssl_port = false

  redis_configuration {
    maxmemory_policy = "allkeys-lru"
  }
}
```

### Bicep

```bicep
// Azure Cache for Redis
resource redisCache 'Microsoft.Cache/redis@2023-08-01' = {
  name: 'redis-myapp-prod'
  location: resourceGroup().location
  properties: {
    sku: {
      name: 'Standard'
      family: 'C'
      capacity: 1
    }
    enableNonSslPort: false
    minimumTlsVersion: '1.2'
    redisConfiguration: {
      'maxmemory-policy': 'allkeys-lru'
    }
    publicNetworkAccess: 'Enabled'
  }
}

// Firewall rule
resource firewallRule 'Microsoft.Cache/redis/firewallRules@2023-08-01' = {
  parent: redisCache
  name: 'allow-app-subnet'
  properties: {
    startIP: '10.0.1.0'
    endIP: '10.0.1.255'
  }
}

// Output connection info
output hostName string = redisCache.properties.hostName
output sslPort int = redisCache.properties.sslPort
output primaryKey string = listKeys(redisCache.id, '2023-08-01').primaryKey
```
  --name redis-myapp-prod \
  --resource-group rg-cache-prod \
  --location centralindia \
  --sku Standard \
  --vm-size C1 \
  --minimum-tls-version 1.2 \
  --enable-non-ssl-port false

# Get connection keys
az redis list-keys \
  --name redis-myapp-prod \
  --resource-group rg-cache-prod

# Show cache info
az redis show --name redis-myapp-prod --resource-group rg-cache-prod

# Scale up
az redis update \
  --name redis-myapp-prod \
  --resource-group rg-cache-prod \
  --sku Standard \
  --vm-size C2

# Force reboot (node restart)
az redis force-reboot \
  --name redis-myapp-prod \
  --resource-group rg-cache-prod \
  --reboot-type PrimaryNode

# Delete
az redis delete --name redis-myapp-prod --resource-group rg-cache-prod --yes
```

---

## Part 9: Real-World Patterns

```
Pattern: Web App with Redis Cache
├── App Service (Node.js/Python/.NET)
├── Redis Cache (Standard C1, private endpoint)
├── Azure SQL Database (backend data store)
│
├── Caching strategy:
│   ├── Cache user profiles: TTL = 5 minutes
│   ├── Cache product catalog: TTL = 1 hour
│   ├── Cache API responses: TTL = 30 seconds
│   └── Session store: TTL = 30 minutes
│
├── Connection: Managed identity + Entra auth (no keys!)
└── Monitoring: Redis metrics in Azure Monitor
```

---

## Quick Reference

```
Redis = In-memory key-value store (sub-millisecond latency)
Tiers: Basic (dev) → Standard (prod) → Premium (enterprise) → Enterprise
Cache-Aside = Most common pattern (check cache → miss → query DB → store)
TTL = Time to Live (auto-expire cached data)
maxmemory-policy = What happens when cache is full (allkeys-lru recommended)

Connection: redis-myapp-prod.redis.cache.windows.net:6380 (TLS)
Always disable non-TLS port (6379)!
Use Entra auth + managed identity (avoid connection strings)
```

---

## What's Next?

Next chapter: [Chapter 31: Azure SQL Managed Instance](31-sql-managed-instance.md) — Full SQL Server instance in the cloud for migrating on-premises workloads.
