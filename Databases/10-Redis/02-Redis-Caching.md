# 🏎️ Chapter 3C.2 — Redis as Cache — Patterns & Strategies

> **Level:** 🟡 Intermediate | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~2-3 hours
> **Prerequisites:** Chapter 3C.1 (Redis Architecture & Data Structures)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why caching is the #1 performance optimization** in any system
- Master **all caching patterns** — Cache-Aside, Read-Through, Write-Through, Write-Behind
- Design **TTL strategies** that prevent stale data without killing performance
- Solve the hardest problem in CS — **Cache Invalidation** — with proven patterns
- Prevent **cache stampede, thundering herd**, and other production disasters
- Build **real-world caching layers** like senior architects do

---

## 🧠 Why Caching? — The $100 Billion Question

> *"There are only two hard things in Computer Science: cache invalidation and naming things."*
> — Phil Karlton

### The Speed Problem Without Cache

```
Without Cache (Every Request Hits Database):

  User → API Server → Database (5-50ms) → API Server → User
  User → API Server → Database (5-50ms) → API Server → User
  User → API Server → Database (5-50ms) → API Server → User
  × 10,000 requests/second = Database is CRUSHED 💀

With Cache (Most Requests Hit Redis):

  User → API Server → Redis (0.5ms) → API Server → User     ⚡ cache HIT
  User → API Server → Redis (0.5ms) → API Server → User     ⚡ cache HIT
  User → API Server → Redis (miss) → DB (15ms) → Redis → User  📝 cache MISS
  × 10,000 requests/second = Database handles only ~500 requests 😌

  Cache Hit Rate: 95% = Database load reduced by 95%!
```

### The Numbers Don't Lie

```
╔═══════════════════════════════════════════════════════════════╗
║              WITHOUT CACHE    vs    WITH CACHE (95% hit)     ║
╠═══════════════════════════════════════════════════════════════╣
║  Avg Response Time │  50ms          │  3ms            🚀     ║
║  DB Queries/sec    │  10,000        │  500            📉     ║
║  DB CPU Usage      │  85%           │  8%             📉     ║
║  Server Costs      │  $5,000/mo     │  $800/mo        💰     ║
║  User Experience   │  "It's slow"   │  "It's instant" 😊     ║
║  Can Handle 10x    │  ❌ Need more  │  ✅ Easily      🔥     ║
║  Traffic Spike?    │     servers    │                         ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## 📦 Caching Pattern 1: Cache-Aside (Lazy Loading)

> **The most common pattern.** Application manages both cache and database. 95% of Redis caching uses this.

### How It Works

```
READ FLOW:
                                    ┌──────────────┐
                               ┌───▶│    REDIS      │
                               │    │   (Cache)     │
                          1. Check  └──────┬───────┘
                          cache     2. HIT? │
                               │    Return  │
  ┌──────────┐  Request   ┌────┴────┐ data ┌▼────────┐
  │  Client  │───────────▶│   App   │◄─────┤         │
  │          │◄───────────│  Server │      │         │
  └──────────┘  Response  └────┬────┘      └─────────┘
                               │
                          3. MISS?│
                          Query   │    ┌──────────────┐
                          DB ─────┴───▶│  DATABASE    │
                                       │  (Source)    │
                          4. Store     └──────┬───────┘
                          in cache ◄──────────┘
                          5. Return
```

### Code Example

```python
# Python Example (Cache-Aside Pattern)
import redis
import json
import psycopg2

r = redis.Redis(host='localhost', port=6379, db=0)

def get_user(user_id):
    # Step 1: Check cache
    cache_key = f"user:{user_id}"
    cached = r.get(cache_key)
    
    if cached:
        print("CACHE HIT ⚡")
        return json.loads(cached)
    
    # Step 2: Cache miss → Query database
    print("CACHE MISS → Querying DB 🐌")
    db = psycopg2.connect("dbname=myapp")
    cursor = db.cursor()
    cursor.execute("SELECT id, name, email FROM users WHERE id = %s", (user_id,))
    row = cursor.fetchone()
    
    if row:
        user = {"id": row[0], "name": row[1], "email": row[2]}
        
        # Step 3: Store in cache with TTL
        r.setex(cache_key, 3600, json.dumps(user))  # Cache for 1 hour
        
        return user
    return None

def update_user(user_id, name, email):
    # Update database
    db = psycopg2.connect("dbname=myapp")
    cursor = db.cursor()
    cursor.execute(
        "UPDATE users SET name = %s, email = %s WHERE id = %s",
        (name, email, user_id)
    )
    db.commit()
    
    # Invalidate cache (delete, don't update!)
    r.delete(f"user:{user_id}")
```

```bash
# Redis commands behind the scenes:

# Read (cache hit):
GET user:42                    # → '{"id":42,"name":"Alice",...}'

# Read (cache miss):
GET user:42                    # → (nil)  — cache miss!
# ... query database ...
SETEX user:42 3600 '{"id":42,"name":"Alice","email":"alice@dev.com"}'

# Write (invalidate):
DEL user:42                    # Force next read to hit DB
```

### Pros & Cons

```
╔════════════════════════════════════╦════════════════════════════════════╗
║  ✅ PROS                          ║  ❌ CONS                           ║
╠════════════════════════════════════╬════════════════════════════════════╣
║  Simple to implement              ║  First request is always slow      ║
║  Cache only what's accessed       ║  (cold cache / cache miss)         ║
║  Cache failure ≠ app failure      ║                                    ║
║  (graceful degradation)           ║  Stale data possible between       ║
║  Works with any database          ║  DB update and cache expiry        ║
║  Most flexible pattern            ║                                    ║
║  Industry standard                ║  Application manages complexity    ║
╚════════════════════════════════════╩════════════════════════════════════╝
```

---

## 📦 Caching Pattern 2: Read-Through

> **Cache acts as the middleman.** Application only talks to cache. Cache loads from database on miss.

```
┌──────────┐         ┌──────────────┐         ┌──────────────┐
│  Client  │────────▶│    CACHE     │────────▶│   DATABASE   │
│          │◄────────│  (Redis +    │◄────────│              │
└──────────┘         │   loader)    │         └──────────────┘
                     └──────────────┘
                     
  App ONLY talks to cache.
  Cache auto-loads from DB on miss.
  Cache library handles everything.
```

```python
# Read-Through Pseudocode
# (Typically implemented by cache libraries like Spring Cache, Caffeine)

class ReadThroughCache:
    def __init__(self, redis_client, data_loader):
        self.redis = redis_client
        self.loader = data_loader  # Function to load from DB
    
    def get(self, key):
        # Check cache
        value = self.redis.get(key)
        if value:
            return json.loads(value)
        
        # Auto-load from DB (transparent to caller)
        value = self.loader(key)
        if value:
            self.redis.setex(key, 3600, json.dumps(value))
        return value

# Usage — caller doesn't know about the database!
cache = ReadThroughCache(redis_client, lambda key: db.query(key))
user = cache.get("user:42")  # Automatically handles miss → DB → cache
```

```
╔════════════════════════════════════╦════════════════════════════════════╗
║  ✅ PROS                          ║  ❌ CONS                           ║
╠════════════════════════════════════╬════════════════════════════════════╣
║  Application code is cleaner      ║  Cache library complexity          ║
║  Cache loading logic centralized  ║  First request still slow          ║
║  Consistent loading behavior      ║  Less flexibility in loading       ║
║  Great for frameworks (Spring)    ║  Harder to debug cache issues      ║
╚════════════════════════════════════╩════════════════════════════════════╝
```

---

## 📦 Caching Pattern 3: Write-Through

> **Every write goes through the cache to the database.** Cache is always consistent with DB.

```
WRITE FLOW:
                     ┌──────────────┐         ┌──────────────┐
  ┌──────────┐  1.   │    CACHE     │  2.     │   DATABASE   │
  │  Client  │──────▶│   (Redis)    │────────▶│              │
  │          │◄──────│  Write here  │◄────────│  Write here  │
  └──────────┘  4.   │   FIRST     │  3.     │   SECOND     │
                     └──────────────┘         └──────────────┘
                     
  1. App writes to cache
  2. Cache writes to database (synchronously)
  3. DB confirms write
  4. Cache confirms to app

  ⚡ Reads are always fast (data is in cache)
  🐌 Writes are slower (write to TWO places)
```

```python
# Write-Through Pattern
class WriteThroughCache:
    def set(self, key, value, ttl=3600):
        # Step 1: Write to database FIRST
        self.db.execute(
            "INSERT INTO cache_data (key, value) VALUES (%s, %s) "
            "ON CONFLICT (key) DO UPDATE SET value = %s",
            (key, json.dumps(value), json.dumps(value))
        )
        
        # Step 2: Write to cache
        self.redis.setex(key, ttl, json.dumps(value))
        
        # Both are now in sync!
```

```
╔════════════════════════════════════╦════════════════════════════════════╗
║  ✅ PROS                          ║  ❌ CONS                           ║
╠════════════════════════════════════╬════════════════════════════════════╣
║  Cache never stale                ║  Write latency increases           ║
║  (always matches DB)              ║  (write to 2 places)               ║
║  Reads are always cache hits      ║  Cache fills with data that        ║
║  Great for read-heavy workloads   ║  may never be read                 ║
║  Simple consistency model         ║  Two points of failure on write    ║
╚════════════════════════════════════╩════════════════════════════════════╝
```

---

## 📦 Caching Pattern 4: Write-Behind (Write-Back)

> **Writes go to cache immediately.** Cache asynchronously syncs to database later. **Fastest writes possible.**

```
WRITE FLOW:
                     ┌──────────────┐    (async, later)   ┌──────────────┐
  ┌──────────┐  1.   │    CACHE     │  ─ ─ ─ ─ ─ ─ ─ ─ ▶│   DATABASE   │
  │  Client  │──────▶│   (Redis)    │  Batch/delayed      │              │
  │          │◄──────│  Write here  │  write               │              │
  └──────────┘  2.   │  DONE! ⚡   │                      └──────────────┘
                     └──────────────┘
                     
  1. App writes to cache (sub-ms)
  2. Returns immediately to client! ⚡
  3. Background worker batch-writes to DB (seconds/minutes later)

  ⚡ Writes are blazing fast
  ⚠️ Risk: Data loss if Redis crashes before DB sync
```

```python
# Write-Behind Pattern (with background worker)
import threading
import time

class WriteBehindCache:
    def __init__(self):
        self.dirty_keys = []
        self.worker = threading.Thread(target=self._sync_worker, daemon=True)
        self.worker.start()
    
    def set(self, key, value):
        # Immediate write to Redis (fast!)
        self.redis.set(key, json.dumps(value))
        self.dirty_keys.append(key)
        # Return immediately — don't wait for DB!
    
    def _sync_worker(self):
        """Background worker: flush dirty keys to DB every 5 seconds"""
        while True:
            time.sleep(5)
            keys_to_sync = self.dirty_keys.copy()
            self.dirty_keys.clear()
            
            for key in keys_to_sync:
                value = self.redis.get(key)
                if value:
                    self.db.upsert(key, json.loads(value))
            
            print(f"Synced {len(keys_to_sync)} keys to DB")
```

```
╔════════════════════════════════════╦════════════════════════════════════╗
║  ✅ PROS                          ║  ❌ CONS                           ║
╠════════════════════════════════════╬════════════════════════════════════╣
║  Fastest write performance        ║  Data loss risk (cache → DB lag)   ║
║  Batch writes reduce DB load      ║  Complex implementation            ║
║  Great for high-write workloads   ║  Eventual consistency only         ║
║  DB can go down temporarily       ║  Harder to debug                   ║
╚════════════════════════════════════╩════════════════════════════════════╝
```

---

## 🆚 Pattern Comparison — Which One Should I Use?

```
╔════════════════════════════════════════════════════════════════════════════╗
║  Pattern           │ Read Perf │ Write Perf │ Consistency │ Complexity   ║
╠════════════════════════════════════════════════════════════════════════════╣
║  Cache-Aside       │ ⚡ Fast*  │ 🏃 Normal  │ 🟡 Eventual │ 🟢 Simple   ║
║  Read-Through      │ ⚡ Fast*  │ 🏃 Normal  │ 🟡 Eventual │ 🟡 Medium   ║
║  Write-Through     │ ⚡⚡ Always│ 🐌 Slower  │ 🟢 Strong   │ 🟡 Medium   ║
║  Write-Behind      │ ⚡⚡ Always│ ⚡⚡ Fastest│ 🔴 Eventual │ 🔴 Complex  ║
╠════════════════════════════════════════════════════════════════════════════╣
║  * = Fast after first miss (cold start is slow)                          ║
╠════════════════════════════════════════════════════════════════════════════╣
║  🔥 MOST COMMON IN INDUSTRY:                                            ║
║  1. Cache-Aside (90% of projects — start here!)                         ║
║  2. Write-Through (when consistency matters)                             ║
║  3. Write-Behind (high-write systems like analytics, IoT)               ║
╚════════════════════════════════════════════════════════════════════════════╝
```

---

## ⏰ TTL Strategies — The Art of Expiration

### Strategy 1: Fixed TTL

```bash
# Every cached item expires after a fixed time
SETEX user:42 3600 '{"name":"Alice"}'          # 1 hour
SETEX product:100 86400 '{"name":"Laptop"}'    # 24 hours
SETEX config:app 300 '{"theme":"dark"}'        # 5 minutes

# ═══════════════════════════════════════
#  TTL Guidelines by Data Type
# ═══════════════════════════════════════

# Highly dynamic data (changes often):
#   Session data       → 30 min - 2 hours
#   User status        → 1-5 minutes
#   Real-time prices   → 10-60 seconds
#   API rate limits    → 1 minute windows

# Moderately dynamic data:
#   User profiles      → 1-4 hours
#   Product details    → 1-24 hours
#   Search results     → 5-30 minutes

# Rarely changing data:
#   App configuration  → 1-24 hours
#   Country/city lists → 24-168 hours (1 week)
#   Static content     → 24-720 hours (1 month)
```

### Strategy 2: Adaptive TTL

```python
# Adjust TTL based on how often data is accessed
def get_with_adaptive_ttl(key, base_ttl=3600):
    # Track access frequency
    access_count = r.incr(f"access_count:{key}")
    
    if access_count > 100:
        ttl = base_ttl * 4      # Hot data → longer TTL (4 hours)
    elif access_count > 10:
        ttl = base_ttl * 2      # Warm data → medium TTL (2 hours)
    else:
        ttl = base_ttl          # Cold data → short TTL (1 hour)
    
    value = r.get(key)
    if value:
        r.expire(key, ttl)      # Refresh TTL on access
        return json.loads(value)
    
    # ... load from DB and cache with calculated TTL ...
```

### Strategy 3: Staggered TTL (Prevent Thundering Herd!)

```python
import random

# ❌ BAD: All cache entries expire at the SAME time
r.setex("product:1", 3600, data1)    # All expire at T+3600
r.setex("product:2", 3600, data2)    # All expire at T+3600
r.setex("product:3", 3600, data3)    # All expire at T+3600
# At T+3600: ALL 3 hit the database simultaneously! 💀

# ✅ GOOD: Add random jitter
def cache_with_jitter(key, value, base_ttl=3600):
    jitter = random.randint(0, 300)  # ±5 minutes randomness
    ttl = base_ttl + jitter
    r.setex(key, ttl, json.dumps(value))

cache_with_jitter("product:1", data1)  # TTL: 3600-3900 (random)
cache_with_jitter("product:2", data2)  # TTL: 3600-3900 (random)
cache_with_jitter("product:3", data3)  # TTL: 3600-3900 (random)
# Entries expire at DIFFERENT times → gradual DB load 😌
```

---

## 🧹 Cache Invalidation — The Hardest Problem

> *"The two hardest things in CS: cache invalidation, naming things, and off-by-one errors."*

### Strategy 1: TTL-Based Expiry (Passive)

```bash
# Let it expire naturally
SETEX user:42 3600 '{"name":"Alice"}'
# After 1 hour → key disappears → next read loads fresh data from DB

# Pros: Simple, no extra logic
# Cons: Data can be stale for up to TTL duration
```

### Strategy 2: Event-Driven Invalidation (Active)

```python
# When data changes → immediately invalidate cache

def update_user(user_id, name, email):
    # 1. Update database
    db.execute("UPDATE users SET name=%s, email=%s WHERE id=%s",
               (name, email, user_id))
    
    # 2. Invalidate cache immediately
    r.delete(f"user:{user_id}")           # Delete cached version
    
    # OR: Update cache (less common, more complex)
    # r.setex(f"user:{user_id}", 3600, json.dumps(updated_user))

# ⚠️ DELETE vs UPDATE in cache:
# DELETE (recommended): Next read gets fresh data from DB
#   → Simpler, safer, handles edge cases better
# UPDATE: Write new value to cache immediately  
#   → Faster reads, but risk of cache-DB inconsistency
```

### Strategy 3: Pub/Sub Invalidation (Distributed)

```python
# When you have MULTIPLE app servers sharing a cache

# Server 1 updates data:
def update_product(product_id, price):
    db.execute("UPDATE products SET price=%s WHERE id=%s", (price, product_id))
    
    # Publish invalidation event
    r.publish("cache_invalidation", json.dumps({
        "action": "invalidate",
        "key": f"product:{product_id}"
    }))

# ALL servers listen for invalidation:
def cache_listener():
    pubsub = r.pubsub()
    pubsub.subscribe("cache_invalidation")
    
    for message in pubsub.listen():
        if message["type"] == "message":
            data = json.loads(message["data"])
            if data["action"] == "invalidate":
                r.delete(data["key"])           # Each server clears its local cache
                print(f"Invalidated: {data['key']}")
```

### Strategy 4: Version-Based Invalidation

```python
# Use a version number to invalidate entire categories

# Set version:
r.set("users:version", 1)

def get_user_cached(user_id):
    version = r.get("users:version")
    cache_key = f"user:{user_id}:v{version}"
    
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Load from DB and cache with version
    user = db.get_user(user_id)
    r.setex(cache_key, 3600, json.dumps(user))
    return user

def invalidate_all_users():
    r.incr("users:version")  # v1 → v2
    # All old keys (v1) become orphaned and expire via TTL
    # New requests create v2 keys with fresh data
    # No need to find and delete every user key!
```

---

## 💀 Cache Problems & Solutions

### Problem 1: Cache Stampede (Thundering Herd)

```
SCENARIO: A popular cache key expires. 1000 requests arrive simultaneously.

  T=3600: Cache key expires
  T=3601: 1000 requests check cache → ALL MISS → ALL query database!
  
  ┌────────┐
  │ Req 1  │──┐
  │ Req 2  │──┤                    ┌──────────┐
  │ Req 3  │──┼── ALL cache miss──▶│ DATABASE │ 💀
  │  ...   │──┤                    │ 1000     │
  │ Req 999│──┤                    │ queries! │
  │ Req1000│──┘                    └──────────┘
  └────────┘
```

#### Solution A: Mutex Lock (Single Flight)

```python
import time

def get_with_mutex(key, ttl=3600):
    value = r.get(key)
    if value:
        return json.loads(value)
    
    # Try to acquire lock
    lock_key = f"lock:{key}"
    if r.set(lock_key, "1", nx=True, ex=30):  # Lock for 30 seconds
        try:
            # Only ONE request queries the database
            value = db.query(key)
            r.setex(key, ttl, json.dumps(value))
            return value
        finally:
            r.delete(lock_key)
    else:
        # Other requests wait and retry
        time.sleep(0.1)
        return get_with_mutex(key, ttl)  # Retry (cache should be populated)
```

```
With Mutex:
  T=3601: 1000 requests check cache → ALL MISS
  T=3601: Request 1 acquires lock → queries database
  T=3601: Requests 2-1000 see lock → wait 100ms → retry
  T=3602: Request 1 populates cache
  T=3602: Requests 2-1000 retry → CACHE HIT! ⚡
  
  Database received: 1 query (instead of 1000) ✅
```

#### Solution B: Early Refresh (Probabilistic)

```python
import random

def get_with_early_refresh(key, ttl=3600, early_window=300):
    value = r.get(key)
    remaining_ttl = r.ttl(key)
    
    if value:
        # Probabilistic early refresh
        # As TTL approaches 0, probability of refresh increases
        if remaining_ttl < early_window:
            refresh_probability = 1 - (remaining_ttl / early_window)
            if random.random() < refresh_probability:
                # Refresh in background (non-blocking)
                refresh_cache_async(key, ttl)
        return json.loads(value)
    
    # Cache miss — load from DB
    return load_and_cache(key, ttl)
```

#### Solution C: Never-Expire + Background Refresh

```python
# Cache never expires! Background worker refreshes periodically.

# Worker (runs every 5 minutes):
def cache_refresh_worker():
    hot_keys = ["product:1", "product:2", "homepage:data"]
    for key in hot_keys:
        fresh_data = db.query(key)
        r.set(key, json.dumps(fresh_data))  # No TTL — never expires!

# Pros: Zero cache misses ever!
# Cons: More complex, needs to know which keys to refresh
```

---

### Problem 2: Cache Penetration

```
SCENARIO: Queries for data that DOESN'T EXIST (not in cache OR database)

  Request: GET /user/99999999  (user doesn't exist)
  
  1. Check cache → MISS (doesn't exist in cache)
  2. Query database → NO RESULTS (doesn't exist in DB either)
  3. Don't cache (nothing to cache!)
  4. Next request → same thing → DB hit again!
  
  Attackers can exploit this to overwhelm your database! 💀
```

#### Solution A: Cache Null Values

```python
def get_user_safe(user_id):
    cache_key = f"user:{user_id}"
    cached = r.get(cache_key)
    
    if cached == "__NULL__":
        return None  # We know it doesn't exist — don't hit DB
    
    if cached:
        return json.loads(cached)
    
    user = db.get_user(user_id)
    
    if user:
        r.setex(cache_key, 3600, json.dumps(user))
    else:
        # Cache the "not found" result with SHORT TTL
        r.setex(cache_key, 300, "__NULL__")  # 5 min — prevents DB hammering
    
    return user
```

#### Solution B: Bloom Filter (Pre-check)

```python
# Bloom filter: "Does this ID POSSIBLY exist?"
# If bloom filter says NO → definitely doesn't exist → skip DB!
# If bloom filter says YES → might exist → check cache/DB

# One-time setup: Load all existing user IDs into bloom filter
bloom_filter = redis_bloom.BFReserve("user_ids_bloom", 0.01, 10000000)
for user_id in db.get_all_user_ids():
    redis_bloom.BFAdd("user_ids_bloom", str(user_id))

def get_user_with_bloom(user_id):
    # Pre-check with Bloom filter
    if not redis_bloom.BFExists("user_ids_bloom", str(user_id)):
        return None  # Definitely doesn't exist — no DB query needed!
    
    # Might exist — proceed with normal cache-aside pattern
    return get_user_cached(user_id)
```

---

### Problem 3: Cache Avalanche

```
SCENARIO: Many cache keys expire at the SAME TIME
          (e.g., after a cache restart, or all set with same TTL)

  T=0:     Cache populated with 10,000 keys, all TTL=3600
  T=3600:  ALL 10,000 keys expire simultaneously!
  T=3601:  10,000 cache misses → 10,000 DB queries → DB dies 💀
```

#### Solutions

```python
# Solution 1: Staggered TTL (already covered above)
def cache_with_jitter(key, value, base_ttl=3600):
    ttl = base_ttl + random.randint(0, 600)  # ±10 minutes
    r.setex(key, ttl, json.dumps(value))

# Solution 2: Cache Warming on Deploy/Restart
def warm_cache():
    """Pre-populate cache after restart"""
    hot_data = db.get_frequently_accessed_data()
    pipe = r.pipeline()
    for item in hot_data:
        ttl = 3600 + random.randint(0, 600)
        pipe.setex(f"item:{item['id']}", ttl, json.dumps(item))
    pipe.execute()  # Batch write — very fast!

# Solution 3: Multi-layer Cache
#   L1: Application memory (Caffeine, Guava) — microseconds
#   L2: Redis — sub-millisecond
#   L3: Database — milliseconds
#   If L2 dies, L1 still serves most requests
```

---

### Problem 4: Hot Key

```
SCENARIO: One key gets 100x more traffic than others

  "product:iphone15" → 50,000 reads/sec (during launch event)
  All other keys → 100 reads/sec each
  
  Problem: Single Redis node bottleneck for that one key!
```

#### Solutions

```python
# Solution 1: Local Cache (L1) for hot keys
from cachetools import TTLCache

local_cache = TTLCache(maxsize=1000, ttl=5)  # 5-second local cache

def get_hot_product(product_id):
    key = f"product:{product_id}"
    
    # Check local (in-process) cache first
    if key in local_cache:
        return local_cache[key]
    
    # Check Redis
    value = r.get(key)
    if value:
        data = json.loads(value)
        local_cache[key] = data  # Cache locally for 5 seconds
        return data
    
    return load_from_db(key)

# Solution 2: Key Replication (spread across multiple keys)
def set_hot_key(key, value, replicas=5):
    for i in range(replicas):
        r.setex(f"{key}:replica:{i}", 3600, json.dumps(value))

def get_hot_key(key, replicas=5):
    # Random replica selection — distributes load!
    replica = random.randint(0, replicas - 1)
    return r.get(f"{key}:replica:{replica}")
```

---

## 🏗️ Real-World Caching Architecture

```
Production Caching Architecture:

  ┌────────────────────────────────────────────────────────────┐
  │                      CLIENT TIER                           │
  │  Browser Cache (HTTP headers: Cache-Control, ETag)         │
  │  CDN (CloudFlare, CloudFront) — static assets              │
  └────────────────────────┬───────────────────────────────────┘
                           │
  ┌────────────────────────▼───────────────────────────────────┐
  │                   APPLICATION TIER                         │
  │  ┌─────────────────────────────────────────────────────┐  │
  │  │  L1 Cache: In-Process (Caffeine/Guava/functools)    │  │
  │  │  TTL: 5-30 seconds | Size: 100-10,000 entries       │  │
  │  │  Latency: ~1 μs | No network hop!                   │  │
  │  └──────────────────────┬──────────────────────────────┘  │
  │                         │ miss                             │
  │  ┌──────────────────────▼──────────────────────────────┐  │
  │  │  L2 Cache: Redis                                     │  │
  │  │  TTL: 5 min - 24 hours | Size: millions of entries   │  │
  │  │  Latency: ~0.5ms | Shared across all app servers     │  │
  │  └──────────────────────┬──────────────────────────────┘  │
  │                         │ miss                             │
  └─────────────────────────┼──────────────────────────────────┘
                            │
  ┌─────────────────────────▼──────────────────────────────────┐
  │                    DATABASE TIER                            │
  │  PostgreSQL / MySQL / MongoDB                              │
  │  Latency: 5-50ms | Source of truth                         │
  └────────────────────────────────────────────────────────────┘
```

---

## 🔧 Redis Caching Commands — Advanced

### Pipelining — Batch Commands for Speed

```bash
# Without pipelining: 100 commands = 100 round trips
# 100 × 0.5ms network = 50ms total

# With pipelining: 100 commands = 1 round trip!
# 1 × 0.5ms network + processing = ~1ms total

# Python:
pipe = r.pipeline()
for i in range(100):
    pipe.get(f"user:{i}")
results = pipe.execute()  # All 100 results in one batch!
```

```bash
# Redis CLI pipelining:
redis-cli --pipe < commands.txt

# Or using MULTI for atomic batch:
MULTI
SET user:1 "Alice"
SET user:2 "Bob"
INCR counter
EXEC
# All 3 commands execute atomically
```

### Memory-Efficient Caching with Compression

```python
import zlib

def cache_compressed(key, value, ttl=3600):
    """Compress before caching — saves 60-80% memory for JSON data"""
    json_bytes = json.dumps(value).encode('utf-8')
    compressed = zlib.compress(json_bytes)
    r.setex(key, ttl, compressed)
    
    # Example: 10KB JSON → 2KB compressed (80% savings!)

def get_compressed(key):
    compressed = r.get(key)
    if compressed:
        json_bytes = zlib.decompress(compressed)
        return json.loads(json_bytes.decode('utf-8'))
    return None
```

---

## 📊 Cache Metrics — What to Monitor

```
╔═══════════════════════════════════════════════════════════════════╗
║  METRIC              │ FORMULA                   │ HEALTHY       ║
╠═══════════════════════════════════════════════════════════════════╣
║  Hit Rate            │ hits / (hits + misses)    │ > 90%         ║
║  Miss Rate           │ misses / (hits + misses)  │ < 10%         ║
║  Memory Usage        │ used_memory / maxmemory   │ < 80%         ║
║  Eviction Rate       │ evicted_keys/sec          │ < 100/sec     ║
║  Latency (p99)       │ 99th percentile           │ < 1ms         ║
║  Connection Count    │ connected_clients         │ < max_clients ║
║  Fragmentation Ratio │ mem_frag_ratio            │ 1.0 - 1.5    ║
╚═══════════════════════════════════════════════════════════════════╝
```

```bash
# Redis commands to check metrics:
INFO stats
# keyspace_hits:1234567
# keyspace_misses:12345
# → Hit rate = 1234567 / (1234567 + 12345) = 99.0% ✅

INFO memory
# used_memory_human:2.5G
# maxmemory_human:4G
# mem_fragmentation_ratio:1.08

redis-cli --latency              # Real-time latency monitoring
redis-cli --bigkeys              # Find memory-hungry keys
redis-cli --memkeys              # Memory usage per key pattern
```

---

## 📊 Chapter Summary — Quick Reference Card

```
╔══════════════════════════════════════════════════════════════════════╗
║                 REDIS CACHING CHEAT SHEET                            ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  CACHING PATTERNS:                                                   ║
║  Cache-Aside   → App checks cache, loads DB on miss (90% of cases)  ║
║  Read-Through  → Cache auto-loads from DB (framework-managed)        ║
║  Write-Through → Write to cache + DB synchronously (consistent)      ║
║  Write-Behind  → Write to cache, async DB sync (fastest writes)      ║
║                                                                      ║
║  INVALIDATION:                                                       ║
║  TTL Expiry    → Simple, eventual consistency                        ║
║  Event-Driven  → DELETE on write (recommended)                       ║
║  Pub/Sub       → Distributed invalidation across servers             ║
║  Versioning    → Increment version, old keys auto-expire             ║
║                                                                      ║
║  CACHE PROBLEMS & FIXES:                                             ║
║  Stampede      → Mutex lock / Early refresh / Never-expire           ║
║  Penetration   → Cache NULL values / Bloom filter                    ║
║  Avalanche     → Staggered TTL (jitter) / Cache warming              ║
║  Hot Key       → Local cache (L1) / Key replication                  ║
║                                                                      ║
║  TTL GUIDELINES:                                                     ║
║  Dynamic data   → 1-5 min  │ User profiles → 1-4 hours              ║
║  API responses  → 5-30 min │ Static config → 24+ hours              ║
║  Always add JITTER to prevent thundering herd!                       ║
║                                                                      ║
║  GOLDEN RULES:                                                       ║
║  1. DELETE cache on write (don't update)                             ║
║  2. Always use TTL (never cache forever without reason)              ║
║  3. Add jitter to TTLs                                               ║
║  4. Monitor hit rate (target > 90%)                                  ║
║  5. Use PIPELINE for batch operations                                ║
║  6. Compress large values (zlib/gzip)                                ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 🔗 What's Next?

| Next Chapter | Topic |
|---|---|
| [3C.3 → Redis Advanced](./03-Redis-Advanced.md) | Pub/Sub, Streams deep dive, Lua scripting, Pipelining, Transactions, Redis Modules |
| [3C.4 → Redis Cluster & HA](./04-Redis-Cluster.md) | Redis Cluster, Sentinel, persistence tuning, production deployment |

---

> 💡 **Remember:** A well-designed cache doesn't just make your app faster — it fundamentally changes what's architecturally possible. The difference between "we need 50 database servers" and "we need 3" is often just a Redis caching layer.

---

*Next up: Let's unlock Redis's advanced features →* [Chapter 3C.3 — Redis Advanced](./03-Redis-Advanced.md)
