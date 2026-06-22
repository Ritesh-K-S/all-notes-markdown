# ⚡ Chapter 7.4 — Caching Strategies with Databases

> **Level:** 🟡 Intermediate | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 1.7 (Indexing), Chapter 7.1 (Sharding)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why** caching is the single biggest performance lever
- Master **5 caching patterns** (Cache-Aside, Read-Through, Write-Through, Write-Behind, Write-Around)
- Know how to handle the **#1 hardest problem in CS** — cache invalidation
- Prevent **thundering herd**, **cache stampede**, and **hot key** disasters
- Design **multi-level caching** (L1 → L2 → L3 → Database)
- Use **Redis and Memcached** effectively with real code examples
- Choose the **right caching strategy** for your specific use case

---

## 🧠 Why Caching Exists — The Speed of Data

```
  How long it takes to access data from different sources:

  ┌──────────────────────────────────────────────────────────────┐
  │  SOURCE              │ LATENCY        │ RELATIVE SPEED       │
  ├──────────────────────┼────────────────┼──────────────────────┤
  │  CPU L1 Cache        │ 0.5 ns         │ ⚡⚡⚡⚡⚡               │
  │  CPU L2 Cache        │ 7 ns           │ ⚡⚡⚡⚡                 │
  │  RAM (Memory)        │ 100 ns         │ ⚡⚡⚡                  │
  │  Redis / Memcached   │ 0.1 - 1 ms     │ ⚡⚡                   │
  │  SSD Read            │ 0.1 - 1 ms     │ ⚡⚡                   │
  │  Database Query      │ 1 - 100 ms     │ ⚡                    │
  │  Cross-DC Network    │ 50 - 150 ms    │ 🐌                   │
  │  HDD Read            │ 5 - 20 ms      │ 🐌                   │
  │  Internet API Call   │ 100 - 1000 ms  │ 🐌🐌                  │
  └──────────────────────┴────────────────┴──────────────────────┘

  Database query:    ~10 ms (with index)
  Redis cache:       ~0.5 ms
  
  That's 20x FASTER! 🚀

  For a page that makes 10 DB queries:
  Without cache: 10 × 10ms = 100ms
  With cache:    10 × 0.5ms = 5ms

  At 100,000 requests/second:
  Without cache: 100K DB queries/sec (database melts 🔥)
  With cache:    95K cache hits + 5K DB queries (database breathes 😌)
```

### When to Cache

```
┌──────────────────────────────────────────────────────────────────┐
│                 SHOULD YOU CACHE THIS DATA?                       │
│                                                                   │
│  ✅ CACHE when:                                                   │
│  • Data is READ frequently but WRITTEN rarely                    │
│  • Computing/fetching the data is EXPENSIVE                      │
│  • Slight staleness is ACCEPTABLE                                │
│  • Same data is requested by MANY users                          │
│                                                                   │
│  Examples: Product catalog, user profiles, config settings,      │
│  API responses, session data, HTML fragments, search results     │
│                                                                   │
│  ❌ DON'T CACHE when:                                             │
│  • Data changes on EVERY request                                 │
│  • Each request needs UNIQUE data                                │
│  • Staleness is UNACCEPTABLE (real-time stock prices)           │
│  • Data is too LARGE to fit in cache                             │
│  • Cache hit ratio would be < 80%                                │
│                                                                   │
│  Examples: Real-time analytics, constantly changing feeds,       │
│  unique per-user calculations, large blob storage                │
│                                                                   │
│  💡 THE 80/20 RULE:                                               │
│  If 80% of requests access 20% of data,                         │
│  caching that 20% gives you massive wins.                        │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Caching Pattern 1: Cache-Aside (Lazy Loading)

> **The most common pattern. Application manages the cache explicitly.**

### How It Works

```
               CACHE-ASIDE PATTERN
               ═══════════════════

    READ (Cache Hit ✅):
    
    ┌──────┐  1. GET key  ┌───────┐
    │ App  │─────────────►│ Cache │──► Found! Return data ✅
    │      │◄─────────────│(Redis)│
    └──────┘  2. Return   └───────┘
                                    (Database never touched)

    READ (Cache Miss ❌):

    ┌──────┐  1. GET key  ┌───────┐
    │ App  │─────────────►│ Cache │──► NOT found! ❌
    │      │              └───────┘
    │      │                  ▲
    │      │  2. Query DB     │ 4. SET key (populate cache)
    │      │──────────┐       │
    │      │          ▼       │
    │      │     ┌────────┐   │
    │      │◄────│Database│───┘
    │      │     └────────┘
    └──────┘  3. Return data

    WRITE:
    
    ┌──────┐  1. Write to DB  ┌────────┐
    │ App  │─────────────────►│Database│ ✅
    │      │                  └────────┘
    │      │  2. Invalidate   ┌───────┐
    │      │  cache (DELETE)  │ Cache │ ← Remove stale entry
    │      │─────────────────►│(Redis)│
    └──────┘                  └───────┘
```

### Implementation

```python
# ═══════════════════════════════════════════════════════
# Cache-Aside Pattern — Python + Redis
# ═══════════════════════════════════════════════════════

import redis
import json
import psycopg2

cache = redis.Redis(host='localhost', port=6379, decode_responses=True)
TTL = 3600  # 1 hour

def get_user(user_id: str) -> dict:
    cache_key = f"user:{user_id}"
    
    # Step 1: Check cache
    cached = cache.get(cache_key)
    if cached:
        return json.loads(cached)  # Cache HIT ✅
    
    # Step 2: Cache MISS — query database
    conn = psycopg2.connect("dbname=myapp")
    cur = conn.cursor()
    cur.execute("SELECT id, name, email FROM users WHERE id = %s", (user_id,))
    row = cur.fetchone()
    conn.close()
    
    if not row:
        return None
    
    user = {"id": row[0], "name": row[1], "email": row[2]}
    
    # Step 3: Populate cache for next time
    cache.setex(cache_key, TTL, json.dumps(user))
    
    return user

def update_user(user_id: str, name: str, email: str):
    # Step 1: Update database
    conn = psycopg2.connect("dbname=myapp")
    cur = conn.cursor()
    cur.execute(
        "UPDATE users SET name = %s, email = %s WHERE id = %s",
        (name, email, user_id)
    )
    conn.commit()
    conn.close()
    
    # Step 2: Invalidate cache (DELETE, don't UPDATE!)
    cache.delete(f"user:{user_id}")
    # Next read will fetch from DB and repopulate cache
```

```javascript
// ═══════════════════════════════════════════════════════
// Cache-Aside — Node.js + Redis + MongoDB
// ═══════════════════════════════════════════════════════

const Redis = require('ioredis');
const redis = new Redis();
const TTL = 3600;

async function getProduct(productId) {
    const cacheKey = `product:${productId}`;
    
    // Step 1: Check cache
    const cached = await redis.get(cacheKey);
    if (cached) {
        console.log('Cache HIT');
        return JSON.parse(cached);
    }
    
    // Step 2: Cache MISS — query MongoDB
    console.log('Cache MISS — querying DB');
    const product = await db.collection('products').findOne({ _id: productId });
    
    if (product) {
        // Step 3: Populate cache
        await redis.setex(cacheKey, TTL, JSON.stringify(product));
    }
    
    return product;
}

async function updateProduct(productId, updates) {
    // Step 1: Update MongoDB
    await db.collection('products').updateOne(
        { _id: productId },
        { $set: updates }
    );
    
    // Step 2: Invalidate cache
    await redis.del(`product:${productId}`);
}
```

### Pros & Cons

```
  ✅ PROS:
  • Simple to implement and understand
  • Cache only contains data that's actually requested (no wasted space)
  • Application has full control over cache logic
  • Works with any database
  • Resilient: if cache goes down, app falls back to DB

  ❌ CONS:
  • First request is always a cache miss (cold start)
  • Possible stale data between write and cache invalidation
  • Application code is cluttered with caching logic
  • Cache stampede possible (many concurrent cache misses)
```

---

## 🏗️ Caching Pattern 2: Read-Through

> **Cache sits in front of the database. Application only talks to cache.**

```
              READ-THROUGH PATTERN
              ════════════════════

    ┌──────┐         ┌───────┐         ┌────────┐
    │ App  │────────►│ Cache │────────►│Database│
    │      │◄────────│       │◄────────│        │
    └──────┘         └───────┘         └────────┘
    
    App ONLY talks to Cache.
    Cache is responsible for loading data from DB on miss.

    Cache Hit:
    App → Cache → "I have it!" → Return ✅

    Cache Miss:
    App → Cache → "Don't have it" → Cache queries DB → 
    Cache stores result → Cache returns to App ✅
    
    DIFFERENCE FROM CACHE-ASIDE:
    ┌──────────────────────────────────────────────────────┐
    │  Cache-Aside:  App manages cache + DB separately    │
    │  Read-Through: Cache manages DB loading itself       │
    │                App doesn't know about DB at all!     │
    └──────────────────────────────────────────────────────┘
```

---

## 🏗️ Caching Pattern 3: Write-Through

> **Every write goes through the cache to the database. Cache is always consistent.**

```
              WRITE-THROUGH PATTERN
              ═════════════════════

    WRITE:
    ┌──────┐  1. Write  ┌───────┐  2. Write  ┌────────┐
    │ App  │──────────►│ Cache │──────────►│Database│
    │      │           │       │           │        │
    │      │◄──────────│       │◄──────────│        │
    └──────┘  4. ACK    └───────┘  3. ACK    └────────┘
    
    Both cache AND database are updated on EVERY write.
    Write returns ONLY after both are written.

    READ:
    ┌──────┐  1. Read   ┌───────┐
    │ App  │──────────►│ Cache │──► Always has fresh data ✅
    │      │◄──────────│       │
    └──────┘  2. Return └───────┘
                                   (Database rarely queried)

  ✅ Cache is ALWAYS consistent with DB
  ✅ Read-through + Write-through = Complete caching layer
  ❌ Write latency is HIGHER (must write to cache + DB)
  ❌ Cache contains data that might never be read (waste)
  
  💡 Best combined with TTL to expire unused data
```

---

## 🏗️ Caching Pattern 4: Write-Behind (Write-Back)

> **Write to cache immediately, asynchronously sync to database later. Maximum write speed.**

```
              WRITE-BEHIND PATTERN
              ════════════════════

    WRITE:
    ┌──────┐  1. Write  ┌───────┐
    │ App  │──────────►│ Cache │──► Return immediately! ⚡
    │      │◄──────────│       │
    └──────┘  2. ACK    └───┬───┘
              (instant!)     │
                             │ 3. Async batch write
                             │    (every 5 seconds or
                             │     every 100 changes)
                             ▼
                        ┌────────┐
                        │Database│
                        │        │
                        │ Written│
                        │ later  │
                        └────────┘

  ✅ BLAZING FAST writes (only cache, no DB wait)
  ✅ Batch writes reduce DB load dramatically
  ✅ Great for write-heavy workloads (IoT, logging, metrics)
  
  ❌ DATA LOSS RISK! If cache crashes before sync → data gone!
  ❌ Complexity (need reliable async sync mechanism)
  ❌ Read-your-writes from DB might be stale
  
  Used by: CPU write-back caches, some CDNs, 
           gaming leaderboards, IoT sensor data
```

---

## 🏗️ Caching Pattern 5: Write-Around

> **Write directly to database, bypassing cache. Cache loads data on read.**

```
              WRITE-AROUND PATTERN
              ═════════════════════

    WRITE (bypasses cache):
    ┌──────┐                          ┌────────┐
    │ App  │─────────────────────────►│Database│ ✅
    │      │                          └────────┘
    └──────┘         ┌───────┐
                     │ Cache │ ← NOT updated (stale until TTL)
                     └───────┘

    READ (cache-aside on read):
    ┌──────┐         ┌───────┐
    │ App  │────────►│ Cache │──► Miss → Load from DB → Cache it
    └──────┘         └───────┘

  ✅ Prevents cache being flooded with write-heavy data
     that might never be read
  ✅ Good for write-once-read-rarely data
  
  ❌ First read after write is always a cache miss
  ❌ Higher read latency for recently written data
  
  Used for: Log data, write-heavy workloads where
  most writes are never immediately read
```

---

## ⚔️ Caching Patterns Comparison

```
┌──────────────────┬──────────────┬──────────────┬──────────────┬───────────────┐
│    Pattern       │ Read Speed   │ Write Speed  │ Consistency  │ Data Loss Risk│
├──────────────────┼──────────────┼──────────────┼──────────────┼───────────────┤
│ Cache-Aside      │ ⚡ Fast       │ Normal       │ ⚠️ Possible   │ 🟢 None       │
│                  │ (after warm) │ (DB + del)   │ stale reads  │               │
│                  │              │              │              │               │
│ Read-Through     │ ⚡ Fast       │ Normal       │ ⚠️ Stale      │ 🟢 None       │
│                  │ (after warm) │              │ possible     │               │
│                  │              │              │              │               │
│ Write-Through    │ ⚡ Fast       │ 🐌 Slower    │ ✅ Strong    │ 🟢 None       │
│                  │ (always hit) │ (cache + DB) │              │               │
│                  │              │              │              │               │
│ Write-Behind     │ ⚡ Fast       │ ⚡⚡ Fastest  │ ⚠️ Eventual  │ 🔴 HIGH!      │
│                  │ (always hit) │ (cache only) │              │ (cache crash) │
│                  │              │              │              │               │
│ Write-Around     │ ⚠️ Miss on   │ ⚡ Fast       │ ⚠️ Stale     │ 🟢 None       │
│                  │ new writes   │ (DB only)    │ until read   │               │
└──────────────────┴──────────────┴──────────────┴──────────────┴───────────────┘

  💡 MOST COMMON COMBINATION:
  Cache-Aside + TTL = 90% of real-world use cases
```

---

## 💀 Cache Invalidation — The Hardest Problem in CS

> _"There are only two hard things in Computer Science: cache invalidation and naming things."_ — Phil Karlton

### Invalidation Strategies

```
┌──────────────────────────────────────────────────────────────────┐
│  STRATEGY 1: TTL (Time-To-Live) — The Simplest                  │
│  ═══════════════════════════════════════════                     │
│                                                                   │
│  cache.setex("user:42", 3600, data)  ← Expires after 1 hour    │
│                                                                   │
│  ✅ Simple, automatic cleanup                                    │
│  ❌ Data can be stale for up to TTL seconds                     │
│  ❌ Too short TTL = cache is useless, too long = stale data     │
│                                                                   │
│  TTL GUIDELINES:                                                  │
│  • Product catalog: 1 hour (changes rarely)                      │
│  • User profile: 5-15 minutes                                    │
│  • Config/settings: 5-60 minutes                                 │
│  • Session data: 30 minutes                                      │
│  • Real-time data: 10-60 seconds                                 │
│  • Static assets: 24+ hours                                      │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  STRATEGY 2: Explicit Invalidation (DELETE on Write)             │
│  ════════════════════════════════════════════════                │
│                                                                   │
│  When data changes → DELETE the cache key                        │
│  Next read will cache miss → load from DB → re-cache            │
│                                                                   │
│  // After updating user in DB:                                   │
│  cache.del("user:42")                                           │
│                                                                   │
│  ✅ Stale data cleared immediately                               │
│  ❌ Must know ALL cache keys affected by the write               │
│  ❌ Forgot to invalidate? = stale data forever!                  │
│                                                                   │
│  💡 WHY DELETE instead of UPDATE?                                │
│  • Deleting is SIMPLER (no race conditions)                     │
│  • If you update cache AND DB, order matters:                    │
│    Thread A: Update cache → DB (thread B reads stale DB)        │
│    Thread B: Update DB → cache (thread A reads stale cache)     │
│  • DELETE + re-cache on next read avoids this!                   │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  STRATEGY 3: Event-Based Invalidation (CDC/Pub-Sub)             │
│  ═══════════════════════════════════════════════                 │
│                                                                   │
│  Database change → Event published → Cache invalidated          │
│                                                                   │
│  ┌──────────┐   CDC    ┌───────┐   Event   ┌──────────┐         │
│  │ Database │────────►│ Kafka │──────────►│ Cache    │         │
│  │ (write)  │         │       │           │ Invalidator│         │
│  └──────────┘         └───────┘           └──────────┘         │
│                                                                   │
│  ✅ Decoupled: write code doesn't need to know about cache      │
│  ✅ Catches ALL changes (even direct DB updates, migrations)     │
│  ❌ Small delay (event processing time)                          │
│  ❌ More infrastructure                                          │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  STRATEGY 4: Version-Based (Cache Busting)                       │
│  ═════════════════════════════════════════                       │
│                                                                   │
│  Include a version number in the cache key.                      │
│  When data changes, increment version → old key auto-expires.   │
│                                                                   │
│  cache.set("product:42:v5", data)                                │
│  // After update: look for "product:42:v6" → cache miss         │
│  // → fetch from DB → cache with v6                              │
│                                                                   │
│  ✅ Old and new data can coexist                                 │
│  ❌ Orphaned old versions waste memory (TTL helps)               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🌪️ Cache Failure Scenarios — And How to Survive Them

### Scenario 1: Cache Stampede (Thundering Herd)

```
  PROBLEM:
  A popular cache key expires.
  1000 requests arrive at the same time.
  All see cache MISS → All query the database simultaneously!

  ┌────────────────────────────────────────────────────────────┐
  │                                                            │
  │   1000 requests                                            │
  │       │                                                    │
  │       ▼                                                    │
  │   ┌───────┐                                                │
  │   │ Cache │  "Key expired!" ❌                             │
  │   └───┬───┘                                                │
  │       │                                                    │
  │       ▼ × 1000 simultaneous DB queries!                    │
  │   ┌────────┐                                               │
  │   │Database│  💀 CRUSHED! Connection pool exhausted!       │
  │   └────────┘                                               │
  │                                                            │
  └────────────────────────────────────────────────────────────┘

  SOLUTIONS:

  ═══════════════════════════════════════════════════════
  SOLUTION 1: Mutex Lock (only one thread queries DB)
  ═══════════════════════════════════════════════════════
  
  Thread 1: Cache miss → Acquire lock → Query DB → Populate cache → Release lock
  Threads 2-1000: Cache miss → Can't acquire lock → Wait → Read from cache
```

```python
# Mutex Lock Pattern — Prevent Cache Stampede

import redis
import time
import json

cache = redis.Redis()

def get_with_lock(key: str, fetch_fn, ttl=3600, lock_timeout=5):
    """Get from cache with mutex lock to prevent stampede."""
    
    # Try cache first
    cached = cache.get(key)
    if cached:
        return json.loads(cached)
    
    # Cache miss — try to acquire lock
    lock_key = f"lock:{key}"
    lock_acquired = cache.set(lock_key, "1", nx=True, ex=lock_timeout)
    
    if lock_acquired:
        try:
            # I won the lock — I'll query DB
            data = fetch_fn()
            cache.setex(key, ttl, json.dumps(data))
            return data
        finally:
            cache.delete(lock_key)
    else:
        # Someone else is fetching — wait and retry
        for _ in range(50):  # Wait up to 5 seconds
            time.sleep(0.1)
            cached = cache.get(key)
            if cached:
                return json.loads(cached)
        
        # Timeout — fetch from DB as fallback
        return fetch_fn()

# Usage
user = get_with_lock(
    "user:42", 
    lambda: db.query("SELECT * FROM users WHERE id = 42")
)
```

```
  ═══════════════════════════════════════════════════════
  SOLUTION 2: Stale-While-Revalidate
  ═══════════════════════════════════════════════════════
  
  Keep serving STALE data while refreshing in background.
  
  Cache stores: { data: {...}, expires_at: T1, stale_at: T2 }
  
  T < T1:  Serve from cache (fresh) ✅
  T1 < T < T2: Serve STALE data + trigger background refresh
  T > T2:  Cache miss, must wait for DB query
  
  ═══════════════════════════════════════════════════════
  SOLUTION 3: Pre-emptive Refresh (Proactive)
  ═══════════════════════════════════════════════════════
  
  Refresh cache BEFORE it expires.
  
  TTL = 1 hour
  Refresh at = 45 minutes (25% before expiry)
  
  A background job checks keys that will expire soon
  and refreshes them proactively.
  
  ✅ No user ever sees a cache miss!
  ❌ Wastes resources refreshing data nobody reads
```

### Scenario 2: Hot Key Problem

```
  PROBLEM: One key gets WAY more traffic than others.

  Example: Taylor Swift announces concert → "taylor_swift_tickets" key
  gets 500,000 requests/second → ONE Redis node handles ALL of them → 💀

  ┌────────────────────────────────────────────────────────────┐
  │  SOLUTIONS:                                                │
  │                                                            │
  │  1. LOCAL CACHE (L1 + L2 architecture)                    │
  │     → Each app server caches hot keys in memory            │
  │     → Reduces Redis calls by 90%+                          │
  │     → Short TTL (10-30 seconds) for local cache            │
  │                                                            │
  │  2. KEY REPLICATION                                         │
  │     → Split one key into multiple: key:1, key:2, key:3    │
  │     → Randomly read from key:{random 1-3}                 │
  │     → Spreads load across Redis cluster slots              │
  │                                                            │
  │  3. READ REPLICAS                                           │
  │     → Redis read replicas handle read traffic              │
  │     → Primary handles writes                               │
  │                                                            │
  │  4. RATE LIMITING                                           │
  │     → If key access rate > threshold, serve from local     │
  │     → Prevents cache server overload                       │
  └────────────────────────────────────────────────────────────┘
```

### Scenario 3: Cache Penetration

```
  PROBLEM: Requests for data that DOESN'T EXIST.

  "GET user:99999999" → Cache miss → DB query → Not found → 
  NOT cached (because there's nothing to cache!) →
  Next request: same thing → DB hit again → forever!

  Attacker sends millions of requests for non-existent IDs
  → Every request bypasses cache → DB is hammered! 💀

  ═══════════════════════════════════════════════════════
  SOLUTION 1: Cache NULL Results
  ═══════════════════════════════════════════════════════
  
  If DB returns nothing, cache a NULL marker with short TTL:
  
  cache.setex("user:99999999", 300, "NULL")  ← 5 min TTL
  
  Next request: cache hit → return "not found" → DB safe ✅

  ═══════════════════════════════════════════════════════
  SOLUTION 2: Bloom Filter
  ═══════════════════════════════════════════════════════

  Before checking cache, check a Bloom Filter:
  "Does this user ID possibly exist?"
  
  NO  → Return "not found" immediately (skip cache AND DB)
  YES → Check cache → Check DB (might be false positive)

  ┌────────┐     ┌──────────────┐     ┌───────┐     ┌────────┐
  │Request │────►│ Bloom Filter │────►│ Cache │────►│Database│
  │        │     │              │     │       │     │        │
  │user:42 │     │ "Maybe exists│     │       │     │        │
  │        │     │  check cache"│     │       │     │        │
  └────────┘     └──────────────┘     └───────┘     └────────┘
  
  │user:999│     │ "Definitely  │
  │99999   │     │  NOT exists" │──► Return 404 immediately!
                 │  Skip cache  │    Database never queried ✅
                 │  Skip DB"    │
```

```python
# Bloom Filter Example — Redis

import redis

cache = redis.Redis()

# Add all existing user IDs to bloom filter (on startup/periodically)
# Redis has built-in Bloom Filter (RedisBloom module)

# Add existing IDs
cache.execute_command('BF.ADD', 'users_bloom', 'user:1')
cache.execute_command('BF.ADD', 'users_bloom', 'user:42')
cache.execute_command('BF.ADD', 'users_bloom', 'user:100')

def get_user(user_id):
    key = f"user:{user_id}"
    
    # Step 0: Check Bloom Filter first!
    might_exist = cache.execute_command('BF.EXISTS', 'users_bloom', key)
    if not might_exist:
        return None  # Definitely doesn't exist — skip everything!
    
    # Step 1: Check cache (normal cache-aside)
    cached = cache.get(key)
    if cached:
        if cached == "NULL":
            return None
        return json.loads(cached)
    
    # Step 2: Query DB
    user = db.query(f"SELECT * FROM users WHERE id = %s", user_id)
    
    if user:
        cache.setex(key, 3600, json.dumps(user))
    else:
        cache.setex(key, 300, "NULL")  # Cache the miss!
    
    return user
```

### Scenario 4: Cache Avalanche

```
  PROBLEM: Many cache keys expire AT THE SAME TIME.
  → Mass cache miss → DB flooded with queries → DB down → App down

  WHY IT HAPPENS:
  • All keys set with same TTL (e.g., all cached at startup with TTL=3600)
  • Cache server restarts (all keys lost at once)

  ┌────────────────────────────────────────────────────────────┐
  │  SOLUTIONS:                                                │
  │                                                            │
  │  1. JITTER: Add random offset to TTL                      │
  │     ttl = 3600 + random.randint(-300, 300)                │
  │     Keys expire at random times, not all at once!          │
  │                                                            │
  │  2. MULTI-LEVEL CACHING:                                   │
  │     L1 (app memory, 30s) → L2 (Redis, 1h) → DB           │
  │     If L2 dies, L1 still serves traffic briefly            │
  │                                                            │
  │  3. CIRCUIT BREAKER:                                       │
  │     If DB error rate > threshold → stop sending queries   │
  │     Return cached data (even stale) or default values      │
  │                                                            │
  │  4. WARM-UP: Pre-load critical cache keys on deployment   │
  │     Don't wait for first request to populate cache         │
  └────────────────────────────────────────────────────────────┘
```

```python
# TTL Jitter — Simple but Effective

import random

def cache_with_jitter(key, data, base_ttl=3600):
    """Add random jitter to TTL to prevent avalanche."""
    jitter = random.randint(-300, 300)  # ±5 minutes
    actual_ttl = base_ttl + jitter
    cache.setex(key, actual_ttl, json.dumps(data))

# Without jitter: All 10,000 keys expire at T+3600 → 💥 BOOM
# With jitter:    Keys expire between T+3300 and T+3900 → Smooth 😌
```

---

## 🏗️ Multi-Level Caching Architecture

```
                MULTI-LEVEL CACHING
                ═══════════════════

    ┌──────────────────────────────────────────────────────┐
    │                                                       │
    │  L1: APPLICATION MEMORY (HashMap / Caffeine / Guava) │
    │  → 10-30 second TTL                                   │
    │  → Per-instance (not shared)                          │
    │  → Fastest: < 1μs access time                        │
    │  → Limited size (100MB - 1GB)                        │
    │                                                       │
    │              │ MISS                                    │
    │              ▼                                         │
    │  L2: DISTRIBUTED CACHE (Redis / Memcached)            │
    │  → 1 minute - 24 hour TTL                             │
    │  → Shared across all instances                        │
    │  → Fast: ~0.5ms access time                           │
    │  → Large size (10GB - 1TB)                            │
    │                                                       │
    │              │ MISS                                    │
    │              ▼                                         │
    │  L3: CDN CACHE (Cloudflare, CloudFront)               │
    │  → For HTTP responses / API responses                 │
    │  → Edge servers worldwide                              │
    │  → Reduces origin server load                         │
    │                                                       │
    │              │ MISS                                    │
    │              ▼                                         │
    │  DATABASE (Source of Truth)                             │
    │  → PostgreSQL / MongoDB / etc.                         │
    │  → Always consistent                                   │
    │  → Slowest but authoritative                           │
    │                                                       │
    └──────────────────────────────────────────────────────┘
```

```java
// Multi-Level Cache — Java with Caffeine (L1) + Redis (L2)

import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.Cache;

public class MultiLevelCache {
    
    // L1: In-memory cache (per JVM instance)
    private final Cache<String, String> l1Cache = Caffeine.newBuilder()
        .maximumSize(10_000)           // Max 10K entries
        .expireAfterWrite(30, TimeUnit.SECONDS)  // 30s TTL
        .build();
    
    // L2: Redis (shared across instances)
    private final RedisTemplate<String, String> redis;
    
    public User getUser(String userId) {
        String key = "user:" + userId;
        
        // L1: Check local memory
        String cached = l1Cache.getIfPresent(key);
        if (cached != null) {
            log.debug("L1 HIT: {}", key);
            return deserialize(cached);
        }
        
        // L2: Check Redis
        cached = redis.opsForValue().get(key);
        if (cached != null) {
            log.debug("L2 HIT: {}", key);
            l1Cache.put(key, cached);  // Promote to L1
            return deserialize(cached);
        }
        
        // L3: Query database
        log.debug("CACHE MISS: {}", key);
        User user = userRepository.findById(userId);
        
        if (user != null) {
            String serialized = serialize(user);
            redis.opsForValue().set(key, serialized, 1, TimeUnit.HOURS);
            l1Cache.put(key, serialized);
        }
        
        return user;
    }
}
```

---

## 🔧 Redis vs Memcached — Which Cache to Choose?

```
┌────────────────────┬───────────────────────┬──────────────────────┐
│   Feature          │      Redis            │    Memcached         │
├────────────────────┼───────────────────────┼──────────────────────┤
│ Data Structures    │ Strings, Lists, Sets, │ Strings only         │
│                    │ Hashes, Sorted Sets,  │                      │
│                    │ Streams, HyperLogLog  │                      │
│                    │                       │                      │
│ Persistence        │ ✅ RDB + AOF          │ ❌ Memory only       │
│                    │                       │                      │
│ Replication        │ ✅ Master-Replica     │ ❌ No replication    │
│                    │                       │                      │
│ Clustering         │ ✅ Redis Cluster      │ ✅ Client-side       │
│                    │                       │    sharding          │
│                    │                       │                      │
│ Pub/Sub            │ ✅ Built-in           │ ❌ No                │
│                    │                       │                      │
│ Lua Scripting      │ ✅ Yes                │ ❌ No                │
│                    │                       │                      │
│ Multi-threaded     │ ✅ (I/O threads, 6.0+)│ ✅ (fully MT)       │
│                    │                       │                      │
│ Memory Efficiency  │ 🟡 Higher overhead    │ ✅ Lower overhead   │
│                    │    per key             │    (simple slab)     │
│                    │                       │                      │
│ Max Value Size     │ 512 MB               │ 1 MB (default)       │
│                    │                       │                      │
│ Use Case           │ Caching + data store  │ Pure caching only   │
│                    │ + queues + pub/sub     │                      │
│                    │                       │                      │
│ When to Choose     │ Need data structures, │ Simple key-value     │
│                    │ persistence, or more  │ caching, multi-      │
│                    │ than just caching     │ threaded perf         │
└────────────────────┴───────────────────────┴──────────────────────┘

💡 VERDICT: Use Redis unless you have a specific reason for Memcached.
   Redis can do everything Memcached can + much more.
```

---

## 📊 Cache Eviction Policies

```
  When cache is FULL, which entries do you remove?

  ┌──────────────────────────────────────────────────────────────┐
  │  EVICTION POLICIES:                                          │
  │                                                              │
  │  LRU (Least Recently Used) ← MOST COMMON                   │
  │  → Remove the entry that hasn't been accessed longest        │
  │  → "If you haven't been read lately, you're out"            │
  │  → Redis default: allkeys-lru                               │
  │                                                              │
  │  LFU (Least Frequently Used)                                │
  │  → Remove the entry accessed the fewest times               │
  │  → Better for skewed workloads (some keys always hot)       │
  │  → Redis: allkeys-lfu (since Redis 4.0)                    │
  │                                                              │
  │  FIFO (First In, First Out)                                 │
  │  → Remove the oldest entry                                  │
  │  → Simple but not optimal                                   │
  │                                                              │
  │  Random                                                      │
  │  → Remove a random entry                                    │
  │  → Surprisingly decent! O(1) implementation                 │
  │  → Redis: allkeys-random                                    │
  │                                                              │
  │  TTL-based                                                   │
  │  → Remove entries closest to expiring                       │
  │  → Redis: volatile-ttl (only keys with TTL set)            │
  │                                                              │
  │  💡 RECOMMENDATION:                                         │
  │  → Start with LRU (simple, works for 90% of cases)         │
  │  → Switch to LFU if you have hot keys that get evicted     │
  │     during traffic spikes                                    │
  └──────────────────────────────────────────────────────────────┘
```

```bash
# Redis Eviction Configuration

# Set max memory (e.g., 2GB)
# redis.conf:
maxmemory 2gb

# Set eviction policy
maxmemory-policy allkeys-lru    # LRU across ALL keys (recommended)
# maxmemory-policy allkeys-lfu  # LFU across ALL keys
# maxmemory-policy volatile-lru # LRU only on keys with TTL set
# maxmemory-policy noeviction   # Return errors when memory full

# Check current config
redis-cli CONFIG GET maxmemory
redis-cli CONFIG GET maxmemory-policy

# Monitor cache hit rate
redis-cli INFO stats | grep keyspace
# keyspace_hits:1234567
# keyspace_misses:12345
# Hit rate = hits / (hits + misses) = 99%  ← Good! ✅
```

---

## 🎯 Caching Decision Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CACHING CHEAT SHEET                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PATTERN SELECTION:                                               │
│  ┌────────────────────────────────────────────────────┐          │
│  │ Simple app?           → Cache-Aside + TTL          │          │
│  │ Read-heavy?           → Read-Through + TTL         │          │
│  │ Need always-fresh?    → Write-Through              │          │
│  │ Write-heavy (IoT)?   → Write-Behind               │          │
│  │ Write-once, read-rare?→ Write-Around               │          │
│  └────────────────────────────────────────────────────┘          │
│                                                                   │
│  TTL GUIDELINES:                                                  │
│  ┌────────────────────────────────────────────────────┐          │
│  │ Static content     → 24 hours+                     │          │
│  │ Product catalog    → 1-6 hours                     │          │
│  │ User profile       → 5-30 minutes                  │          │
│  │ Session data       → 15-60 minutes                 │          │
│  │ Real-time feed     → 10-60 seconds                 │          │
│  │ Always add jitter! → ±10% of TTL                   │          │
│  └────────────────────────────────────────────────────┘          │
│                                                                   │
│  FAILURE PREVENTION:                                              │
│  ┌────────────────────────────────────────────────────┐          │
│  │ Stampede → Mutex lock + stale-while-revalidate    │          │
│  │ Hot key  → Local cache (L1) + key replication     │          │
│  │ Penetration → Bloom filter + cache NULL results   │          │
│  │ Avalanche → TTL jitter + multi-level caching     │          │
│  └────────────────────────────────────────────────────┘          │
│                                                                   │
│  MONITORING:                                                      │
│  ┌────────────────────────────────────────────────────┐          │
│  │ Hit rate > 95% → Excellent ✅                      │          │
│  │ Hit rate 80-95% → Good, but can improve            │          │
│  │ Hit rate < 80% → Something's wrong, investigate!   │          │
│  │ Memory usage > 80% → Time to scale or evict more  │          │
│  │ Eviction rate rising → Need more memory            │          │
│  └────────────────────────────────────────────────────┘          │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

Now that you can **speed up** your database with caching, the next chapter brings it all together with **System Design** — designing real systems from the database perspective.

> **Chapter 7.5:** [Database System Design — Interview Prep →](./05-System-Design-DB.md)

---

> _"Caching is like lying — it's easy to start, hard to stop, and eventually someone gets hurt by stale data."_
> — Anonymous Senior Engineer
