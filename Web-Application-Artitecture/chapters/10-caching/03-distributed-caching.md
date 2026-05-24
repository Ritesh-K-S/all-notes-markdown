# Distributed Caching (Redis, Memcached)

> **What you'll learn**: How distributed caches like Redis and Memcached work, how they scale across multiple servers, when to use each, and how production systems handle millions of cache operations per second.

---

## Real-Life Analogy

Imagine a chain of coffee shops (your app servers). Each shop could keep its own supply of milk in a mini-fridge (local cache), but then each shop orders separately, and some run out while others have extra.

Instead, they share a **central warehouse** (distributed cache) just down the street. Every shop knows exactly what's there, can grab what they need in seconds, and when stock is updated, everyone sees the same inventory.

A **distributed cache** is this shared warehouse — a fast, centralized (or clustered) storage system that all your application servers can access together.

---

## Core Concept Explained Step-by-Step

### Step 1: The Problem with Local Caches in Multi-Instance Deployments

When you scale to multiple app server instances, local caches diverge:

```
Instance A's cache: user:123 = {name: "Alice", age: 30}
Instance B's cache: user:123 = {name: "Alice", age: 29}  ← STALE!
Instance C's cache: (no entry)                            ← MISS

User updates age to 31 → only one instance knows!
```

### Step 2: Distributed Cache — One Shared Truth

```
┌────────────┐     ┌────────────┐     ┌────────────┐
│ Instance A │     │ Instance B │     │ Instance C │
└─────┬──────┘     └─────┬──────┘     └─────┬──────┘
      │                   │                   │
      └───────────────────┼───────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │    Redis / Memcached   │
              │    (Shared Cache)      │
              │                        │
              │  user:123 = {age: 31}  │  ← Single source of truth
              │  product:456 = {...}   │
              └───────────────────────┘
```

Now **every instance** sees the same cached data.

### Step 3: How It Works — Network Access to Shared Memory

```
App Instance ──[TCP/network]──▶ Cache Server ──▶ In-Memory Storage
                                                       │
                                                  ┌────┴────┐
                                                  │ Key-Value│
                                                  │ Hash Map │
                                                  │ in RAM   │
                                                  └──────────┘
```

The trade-off: You pay **~0.5–2ms network latency** for every cache access, but gain:
- Consistency across all instances
- Persistence (Redis can write to disk)
- Much larger storage (dedicated memory servers)
- Shared state (sessions, rate limiting, locks)

---

## Redis vs Memcached — Side by Side

```
┌──────────────────────┬──────────────────────┬──────────────────────┐
│     Feature          │       Redis          │     Memcached        │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Data Structures      │ Strings, Lists, Sets,│ Strings only         │
│                      │ Hashes, Sorted Sets, │ (key → value)        │
│                      │ Streams, HyperLogLog │                      │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Persistence          │ Yes (RDB + AOF)      │ No (pure memory)     │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Replication          │ Master-Replica       │ No built-in          │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Clustering           │ Redis Cluster        │ Client-side sharding │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Pub/Sub              │ Yes                  │ No                   │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Lua Scripting        │ Yes                  │ No                   │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Multi-threaded       │ Single-threaded I/O  │ Multi-threaded       │
│                      │ (6.0+ has I/O threads│                      │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Memory Efficiency    │ More overhead        │ Better for simple KV │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Typical Latency      │ < 1ms                │ < 1ms                │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Use Case             │ General purpose,     │ Simple, high-throughput│
│                      │ sessions, queues,    │ caching only          │
│                      │ rate limiting, more  │                      │
└──────────────────────┴──────────────────────┴──────────────────────┘
```

> **Bottom Line**: Redis is the default choice for most teams today. Memcached is simpler and can be better for pure key-value caching at extreme scale (Facebook uses it).

---

## How It Works Internally

### Redis Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    REDIS SERVER                           │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Event Loop (Single-Threaded Command Processing) │    │
│  │                                                   │    │
│  │  ┌───────────┐  ┌───────────┐  ┌──────────┐    │    │
│  │  │ Read Cmd  │→ │ Execute   │→ │  Write   │    │    │
│  │  │ from      │  │ Command   │  │  Response │    │    │
│  │  │ Socket    │  │ (in-mem)  │  │  to Socket│    │    │
│  │  └───────────┘  └───────────┘  └──────────┘    │    │
│  └─────────────────────────────────────────────────┘    │
│                                                           │
│  ┌──────────────────┐  ┌─────────────────────────────┐  │
│  │  Hash Table      │  │  Persistence                 │  │
│  │  (main dict)     │  │  • RDB: Point-in-time snap   │  │
│  │                   │  │  • AOF: Append-only log      │  │
│  │  Key → Value     │  └─────────────────────────────┘  │
│  │  with TTL        │                                    │
│  └──────────────────┘                                    │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**Why single-threaded?**  
Redis processes commands sequentially on ONE thread. This avoids lock contention and context switching. Despite being single-threaded, Redis handles **100,000+ operations/second** per core because operations are pure in-memory and take microseconds each.

### Redis Cluster — Scaling Horizontally

When one Redis node isn't enough:

```
┌─────────────────────────────────────────────────────────────────┐
│                      REDIS CLUSTER                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  16,384 hash slots distributed across nodes:                     │
│                                                                   │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐       │
│  │   Node A      │  │   Node B      │  │   Node C      │       │
│  │  Slots 0-5460 │  │ Slots 5461-   │  │ Slots 10923-  │       │
│  │               │  │      10922    │  │      16383    │       │
│  │  ┌─────────┐  │  │  ┌─────────┐  │  │  ┌─────────┐  │       │
│  │  │ Replica │  │  │  │ Replica │  │  │  │ Replica │  │       │
│  │  └─────────┘  │  │  └─────────┘  │  │  └─────────┘  │       │
│  └───────────────┘  └───────────────┘  └───────────────┘       │
│                                                                   │
│  Key "user:123" → CRC16("user:123") % 16384 = slot 5649        │
│                 → Routes to Node B                               │
└─────────────────────────────────────────────────────────────────┘
```

### Memcached Architecture — Client-Side Sharding

```
┌──────────────────┐
│     Client       │
│  (your app)      │
│                  │
│  Consistent Hash │
│  Ring:           │
│  key → server    │
└───────┬──────────┘
        │
        ├──────────────▶ Memcached Server 1 (32 GB RAM)
        ├──────────────▶ Memcached Server 2 (32 GB RAM)
        └──────────────▶ Memcached Server 3 (32 GB RAM)

Total cache capacity: 96 GB
```

Unlike Redis Cluster, Memcached nodes don't know about each other. The **client** decides which server holds each key using consistent hashing.

---

## Code Examples

### Python — Redis Caching with Connection Pooling

```python
import redis
import json
from typing import Optional

# Connection pool — reuse connections across requests
pool = redis.ConnectionPool(
    host='redis-cluster.internal',
    port=6379,
    max_connections=50,
    decode_responses=True
)
r = redis.Redis(connection_pool=pool)

def get_user(user_id: str) -> dict:
    """Get user with Redis caching."""
    cache_key = f"user:{user_id}"
    
    # Try cache first
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)  # HIT (~0.5ms)
    
    # Cache miss — fetch from DB (~20ms)
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # Store in Redis with 5-minute TTL
    r.setex(cache_key, 300, json.dumps(user))
    
    return user

def invalidate_user(user_id: str):
    """Remove user from cache after update."""
    r.delete(f"user:{user_id}")
```

### Python — Redis with Hash Data Structure

```python
# Store structured data as a Redis Hash (more memory-efficient)
def cache_product(product):
    r.hset(f"product:{product['id']}", mapping={
        'name': product['name'],
        'price': str(product['price']),
        'stock': str(product['stock'])
    })
    r.expire(f"product:{product['id']}", 600)  # 10 min TTL

def get_product_price(product_id):
    # Get just one field — efficient!
    price = r.hget(f"product:{product_id}", 'price')
    return float(price) if price else None
```

### Java — Redis with Jedis Client

```java
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;
import com.fasterxml.jackson.databind.ObjectMapper;

public class UserCacheService {
    
    private final JedisPool jedisPool;
    private final ObjectMapper mapper = new ObjectMapper();

    public UserCacheService() {
        JedisPoolConfig config = new JedisPoolConfig();
        config.setMaxTotal(50);         // Max connections
        config.setMaxIdle(10);          // Idle connections to keep
        this.jedisPool = new JedisPool(config, "redis-cluster.internal", 6379);
    }

    public User getUser(String userId) {
        try (var jedis = jedisPool.getResource()) {
            // Try cache
            String cached = jedis.get("user:" + userId);
            if (cached != null) {
                return mapper.readValue(cached, User.class);  // HIT
            }
            
            // Miss — load from DB
            User user = database.findById(userId);
            
            // Cache with 5-min TTL
            jedis.setex("user:" + userId, 300, mapper.writeValueAsString(user));
            
            return user;
        }
    }
}
```

### Java — Spring Boot with Redis (Annotation-Based)

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.stereotype.Service;

@Service
public class ProductService {

    // Automatically caches return value in Redis
    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(String productId) {
        // This only runs on cache miss
        return productRepository.findById(productId);
    }

    // Removes from cache when product is updated
    @CacheEvict(value = "products", key = "#product.id")
    public void updateProduct(Product product) {
        productRepository.save(product);
    }
}
```

---

## Infrastructure Setup

### Redis with Docker Compose (Development)

```yaml
# docker-compose.yml
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data

  redis-commander:  # Web UI to inspect Redis
    image: rediscommander/redis-commander
    ports:
      - "8081:8081"
    environment:
      REDIS_HOSTS: local:redis:6379

volumes:
  redis-data:
```

### Redis Cluster (Production on AWS ElastiCache)

```
┌─────────────────────────────────────────────────────────┐
│              AWS ElastiCache Redis Cluster                │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  VPC: 10.0.0.0/16                                        │
│                                                           │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Subnet Group (Private Subnets)                   │   │
│  │                                                    │   │
│  │  AZ-1a          AZ-1b          AZ-1c             │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │   │
│  │  │Primary 1 │  │Primary 2 │  │Primary 3 │       │   │
│  │  │(Shard 1) │  │(Shard 2) │  │(Shard 3) │       │   │
│  │  └──────────┘  └──────────┘  └──────────┘       │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │   │
│  │  │Replica 1 │  │Replica 2 │  │Replica 3 │       │   │
│  │  └──────────┘  └──────────┘  └──────────┘       │   │
│  └──────────────────────────────────────────────────┘   │
│                                                           │
│  Total: 6 nodes, 3 shards, ~150 GB capacity              │
│  Throughput: ~500,000 ops/sec                            │
└─────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### How Twitter Uses Redis + Memcached

Twitter handles **600,000 requests/second** for timeline reads. They use a combination:

```
Timeline Request
     │
     ▼
┌──────────────────────────┐
│  Redis Cluster            │  ← Timeline cache (who posted what, in order)
│  (Sorted Sets per user)   │     Uses ZRANGEBYSCORE for time-ordered feeds
│  user:123:timeline →      │
│    [{tweet_id: 999, ts},  │
│     {tweet_id: 998, ts}]  │
└────────────┬─────────────┘
             │ miss (cold user)
             ▼
┌──────────────────────────┐
│  Memcached Cluster        │  ← Tweet content cache
│  tweet:999 → {text,       │     Simple key-value, very fast
│    author, media_urls}     │
└────────────┬─────────────┘
             │ miss
             ▼
┌──────────────────────────┐
│  Manhattan (Twitter DB)   │  ← Persistent storage
└──────────────────────────┘
```

**Key insight**: Redis is used for its sorted sets (timeline ordering). Memcached is used for simple key-value lookups (tweet content) where its multi-threading shines under high load.

### How Instagram Caches with Redis

- **12+ million Redis requests/second** at peak
- Uses Redis for sessions, counters (likes), feed caching
- Runs on **AWS ElastiCache** with auto-failover
- Key pattern: `{user_id}:{data_type}` for clean namespacing

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| No connection pooling | Opens new TCP connection per request (slow!) | Use connection pools (50-100 connections) |
| Storing huge values (>1MB) | Blocks Redis event loop | Break into smaller chunks or use a different store |
| No TTL on keys | Memory fills up, OOM | ALWAYS set TTL or use `maxmemory-policy` |
| Not handling cache failures | App crashes if Redis is down | Fallback to DB on cache error |
| Serializing with language-specific formats | Can't share cache between Python/Java services | Use JSON or Protobuf |
| Hot key problem | One key gets millions of requests → one node overloaded | Replicate hot keys, add local L1 cache |
| Not using pipelining | 100 keys = 100 round trips | Pipeline multiple commands in one batch |

### The Hot Key Problem

```
Problem:
  "trending_topic:worldcup" gets 1M reads/sec
  All requests go to ONE Redis node → bottleneck

Solutions:
  1. Local cache (L1) in front of Redis — absorb hot reads
  2. Read replicas — spread reads across multiple nodes
  3. Key replication — duplicate key across multiple slots
     "trending_topic:worldcup:1", "trending_topic:worldcup:2", etc.
```

---

## When to Use / When NOT to Use

### ✅ Use Redis When:
- Need **shared state** across multiple app instances
- Need **rich data structures** (lists, sets, sorted sets, streams)
- Need **persistence** — data should survive restarts
- Need **pub/sub**, **streams**, or **Lua scripting**
- Use cases: sessions, rate limiting, leaderboards, queues, real-time analytics

### ✅ Use Memcached When:
- **Pure caching** only (no need for persistence or data structures)
- Need **maximum simplicity** and multi-threading
- Caching **large blobs** (rendered HTML pages, serialized objects)
- Want **horizontal scaling** with simple consistent hashing

### ❌ Don't Use Distributed Cache When:
- Single server deployment (local cache is faster)
- Data is too large for RAM (use database or object storage)
- You need **strong consistency** (cache is eventually consistent)
- Sub-microsecond latency required (use local/application-level cache)

---

## Key Takeaways

1. **Distributed cache** = shared in-memory store accessible by all app instances over the network.
2. **Redis** is the default choice — supports rich data structures, persistence, clustering, and pub/sub.
3. **Memcached** is simpler and can outperform Redis for pure key-value caching at extreme scale.
4. Redis Cluster shards data across nodes using **16,384 hash slots** for horizontal scaling.
5. Always use **connection pooling** — creating a new TCP connection per request is expensive.
6. Handle the **hot key problem** with local caches or key replication.
7. Design for **cache failure** — your app should degrade gracefully if Redis goes down, not crash.

---

## What's Next?

Next, we'll explore **CDN Caching** — how to cache content at edge locations around the world so users get responses from a server just a few milliseconds away, instead of crossing continents.

→ [04-cdn-caching.md](./04-cdn-caching.md)
