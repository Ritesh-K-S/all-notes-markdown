# Cache Stampede, Thundering Herd & Cache Penetration

> **What you'll learn**: The three most dangerous cache failure patterns that can bring down your entire system, why they happen, and battle-tested solutions used by companies like Facebook, Instagram, and Discord.

---

## Real-Life Analogy

Imagine a popular restaurant with one chef (your database). There's a glass display case showing today's menu (your cache).

- **Cache Stampede (Thundering Herd)**: The display case breaks. Suddenly, ALL 200 customers rush to the kitchen to ask the chef directly. The chef collapses.
- **Cache Penetration**: Customers keep asking for a dish that doesn't exist ("Do you have unicorn soup?"). Every time, the chef checks the full menu, finds nothing, and the display case has nothing to show. The requests never stop.
- **Cache Breakdown**: The MOST popular dish's sign falls off the display. Everyone asking for it goes straight to the chef, even though other dishes are still displayed fine.

All three scenarios overwhelm the chef (database) because the cache failed to protect it.

---

## Problem 1: Cache Stampede (Thundering Herd)

### What Happens

When a popular cache entry **expires**, ALL concurrent requests for that key simultaneously hit the database.

```
Normal Operation:
  1000 requests/sec for "trending_products"
  All served from cache ✓ (DB load: 0)

The Moment TTL Expires:
  ┌────────────────────────────────────────────────────────────────┐
  │                   CACHE STAMPEDE                                 │
  ├────────────────────────────────────────────────────────────────┤
  │                                                                  │
  │  t=300s: TTL expires on "trending_products"                     │
  │                                                                  │
  │  t=300.001s: Request 1 → Cache MISS → Query DB                 │
  │  t=300.002s: Request 2 → Cache MISS → Query DB                 │
  │  t=300.003s: Request 3 → Cache MISS → Query DB                 │
  │  ...                                                             │
  │  t=300.050s: Request 1000 → Cache MISS → Query DB              │
  │                                                                  │
  │  Result: 1000 IDENTICAL queries hit DB simultaneously!          │
  │  DB response time: 5ms → 5000ms (overwhelmed)                  │
  │  Cascading failure begins...                                    │
  │                                                                  │
  └────────────────────────────────────────────────────────────────┘
```

```
Timeline diagram:

    Cache HIT          Expire!     Cache HIT again
    (smooth)           (storm)     (after reload)
       │                  │              │
───────┼──────────────────┼──────────────┼────────────▶ time
       │                  │              │
DB     │   ░░░░░░░░░░░░░  │  ████████████│░░░░░░░░░░░░░
Load   │   (near zero)    │  (SPIKE!)    │  (normal)
       │                  │              │
```

### Solutions

#### Solution 1: Locking (Mutex) — Only One Rebuilds

```python
import redis
import time

r = redis.Redis()

def get_with_lock(key, ttl=300, lock_ttl=10):
    """Only ONE request rebuilds cache on miss. Others wait."""
    
    # Try cache first
    value = r.get(key)
    if value:
        return json.loads(value)
    
    # Cache miss — try to acquire lock
    lock_key = f"lock:{key}"
    acquired = r.set(lock_key, "1", nx=True, ex=lock_ttl)
    
    if acquired:
        # I won the lock — I'll rebuild the cache
        try:
            result = expensive_database_query()
            r.setex(key, ttl, json.dumps(result))
            return result
        finally:
            r.delete(lock_key)
    else:
        # Someone else is rebuilding — wait and retry
        for _ in range(50):  # Wait up to 5 seconds
            time.sleep(0.1)
            value = r.get(key)
            if value:
                return json.loads(value)
        
        # Fallback: query DB directly (lock holder might have failed)
        return expensive_database_query()
```

#### Solution 2: Early Expiration (Probabilistic)

```python
import random
import time

def get_with_early_refresh(key, ttl=300, beta=1.0):
    """
    XFetch algorithm: probabilistically refresh cache BEFORE it expires.
    The closer to expiry, the higher the probability of refresh.
    """
    cached = r.get(key)
    if not cached:
        # True miss
        return rebuild_and_cache(key, ttl)
    
    data = json.loads(cached)
    expiry_time = float(r.ttl(key))
    
    # Probabilistic early refresh
    # As TTL approaches 0, probability of refresh increases
    delta = ttl  # Original TTL
    now = time.time()
    
    if expiry_time - delta * beta * math.log(random.random()) <= 0:
        # Refresh early! (I "volunteered" to rebuild)
        return rebuild_and_cache(key, ttl)
    
    return data

def rebuild_and_cache(key, ttl):
    result = expensive_database_query()
    r.setex(key, ttl, json.dumps(result))
    return result
```

#### Solution 3: Stale-While-Revalidate

```python
def get_with_stale_revalidate(key, ttl=300, stale_ttl=600):
    """
    Store data with TWO TTLs:
    - Soft TTL: when to start refreshing
    - Hard TTL: when data is truly gone
    
    Serve stale data while one thread refreshes.
    """
    raw = r.get(key)
    if not raw:
        # Hard miss — must rebuild
        return rebuild_with_lock(key, ttl, stale_ttl)
    
    entry = json.loads(raw)
    
    if time.time() > entry['soft_expiry']:
        # Soft-expired — serve stale but trigger background refresh
        if r.set(f"refresh:{key}", "1", nx=True, ex=30):
            # I got the refresh lock — update in background
            threading.Thread(target=background_refresh, 
                           args=(key, ttl, stale_ttl)).start()
    
    return entry['data']  # Return (possibly stale) data immediately

def background_refresh(key, ttl, stale_ttl):
    """Refresh cache in background thread."""
    result = expensive_database_query()
    entry = {
        'data': result,
        'soft_expiry': time.time() + ttl
    }
    r.setex(key, stale_ttl, json.dumps(entry))
    r.delete(f"refresh:{key}")
```

---

## Problem 2: Cache Penetration

### What Happens

Requests for data that **doesn't exist** in the database bypass the cache every time (since you can't cache a result that doesn't exist), hammering the DB with pointless queries.

```
┌────────────────────────────────────────────────────────────────┐
│                   CACHE PENETRATION                              │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Attacker or bug sends: GET /user/99999999 (doesn't exist)     │
│                                                                  │
│  Request → Cache: MISS (nothing to cache for non-existent data)│
│  Request → DB: SELECT * FROM users WHERE id = 99999999         │
│  Result: empty (no rows)                                        │
│  Nothing cached (can't cache "nothing")                         │
│                                                                  │
│  Next request → same thing → DB hit again                       │
│  ...                                                             │
│  1000 requests/sec for non-existent IDs → 1000 DB queries/sec  │
│                                                                  │
│  Common causes:                                                  │
│  - Attackers probing random IDs                                 │
│  - Buggy client code requesting invalid IDs                     │
│  - Enumeration attacks                                           │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

### Solutions

#### Solution 1: Cache Null/Empty Results

```python
def get_user(user_id):
    cache_key = f"user:{user_id}"
    
    cached = r.get(cache_key)
    if cached == "NULL_SENTINEL":
        return None  # Known non-existent, don't hit DB
    if cached:
        return json.loads(cached)
    
    # Cache miss
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    if user:
        r.setex(cache_key, 300, json.dumps(user))
    else:
        # Cache the ABSENCE of data (short TTL to allow creation)
        r.setex(cache_key, 60, "NULL_SENTINEL")
    
    return user
```

#### Solution 2: Bloom Filter — The Ultimate Shield

A Bloom Filter is a space-efficient probabilistic data structure that tells you:
- "Definitely NOT in set" → **guaranteed true**, skip DB
- "Possibly in set" → **might be false positive**, check DB

```
┌────────────────────────────────────────────────────────────────┐
│                 BLOOM FILTER PROTECTION                          │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Request: GET /user/99999999                                    │
│       │                                                          │
│       ▼                                                          │
│  ┌──────────────────────┐                                       │
│  │    Bloom Filter       │                                       │
│  │  "Does user 99999999 │                                       │
│  │   POSSIBLY exist?"   │                                       │
│  └──────────┬───────────┘                                       │
│        NO   │    MAYBE                                           │
│        │    │       │                                            │
│        ▼    │       ▼                                            │
│  Return 404 │   Check Cache → Check DB                          │
│  (FAST!)    │                                                    │
│             │                                                    │
│  Blocks 100% of requests for non-existent data                  │
│  (with tiny false positive rate ~0.1%)                           │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

```python
from pybloom_live import BloomFilter

# Initialize with all existing user IDs (e.g., at startup)
user_bloom = BloomFilter(capacity=10_000_000, error_rate=0.001)

# Populate bloom filter with existing IDs
for user_id in db.query("SELECT id FROM users"):
    user_bloom.add(str(user_id))

def get_user_safe(user_id):
    # Step 1: Bloom filter check (O(1), in-memory)
    if str(user_id) not in user_bloom:
        return None  # DEFINITELY doesn't exist. Don't even check cache/DB.
    
    # Step 2: Normal cache-aside pattern
    return get_user_from_cache_or_db(user_id)

# When new user is created, add to bloom filter
def create_user(data):
    user = db.insert_user(data)
    user_bloom.add(str(user.id))
    return user
```

#### Solution 3: Request Rate Limiting per Key Pattern

```python
from collections import defaultdict
import time

miss_counter = defaultdict(lambda: {'count': 0, 'window_start': time.time()})

def get_with_penetration_protection(key):
    cached = r.get(key)
    if cached:
        return json.loads(cached) if cached != "NULL" else None
    
    # Track misses for this key
    entry = miss_counter[key]
    if time.time() - entry['window_start'] > 60:
        entry['count'] = 0
        entry['window_start'] = time.time()
    
    entry['count'] += 1
    
    # If same key misses too many times, block it
    if entry['count'] > 10:
        return None  # Suspected attack, don't hit DB
    
    # Normal DB lookup
    result = db.query_by_key(key)
    r.setex(key, 300 if result else 60, json.dumps(result) if result else "NULL")
    return result
```

---

## Problem 3: Cache Breakdown (Hot Key Expiry)

### What Happens

A **single extremely hot key** expires, and the massive traffic for that specific key all hits the DB simultaneously. Similar to stampede but focused on ONE key.

```
┌────────────────────────────────────────────────────────────────┐
│                   CACHE BREAKDOWN                                │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Key "worldcup_final_score" — 500,000 reads/second              │
│                                                                  │
│  When it expires:                                                │
│  500,000 simultaneous DB queries for THE SAME ROW               │
│                                                                  │
│  This is even worse than general stampede because:              │
│  - The traffic is concentrated on ONE database row/query        │
│  - Row-level locks make it even slower                          │
│  - DB connection pool exhausted instantly                       │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

### Solutions

#### Solution: Never-Expire + Background Refresh

```java
@Service
public class HotKeyCache {
    
    @Autowired private RedisTemplate<String, String> redis;
    
    /**
     * For extremely hot keys: NEVER let them expire.
     * Instead, refresh them proactively in the background.
     */
    @Scheduled(fixedRate = 30_000)  // Every 30 seconds
    public void refreshHotKeys() {
        List<String> hotKeys = List.of(
            "trending_topics",
            "live_match_score",
            "homepage_feed"
        );
        
        for (String key : hotKeys) {
            try {
                String freshData = fetchFromDatabase(key);
                redis.opsForValue().set(key, freshData);
                // No TTL! Always present.
            } catch (Exception e) {
                // If DB fails, old cached data stays (better stale than nothing)
                log.warn("Failed to refresh hot key: {}", key, e);
            }
        }
    }
}
```

#### Solution: Singleflight Pattern (Go-inspired, all languages)

```python
import threading
from concurrent.futures import Future

class Singleflight:
    """
    Ensures that only ONE call is in-flight for a given key.
    All other callers wait for the same result.
    Like Go's singleflight package.
    """
    
    def __init__(self):
        self._lock = threading.Lock()
        self._in_flight = {}  # key → Future
    
    def do(self, key, fn):
        """Execute fn() for key, deduplicating concurrent calls."""
        with self._lock:
            if key in self._in_flight:
                # Someone else is already fetching — wait for their result
                future = self._in_flight[key]
            else:
                # I'm the first — create a future and start work
                future = Future()
                self._in_flight[key] = future
                
                # Start the actual work
                def worker():
                    try:
                        result = fn()
                        future.set_result(result)
                    except Exception as e:
                        future.set_exception(e)
                    finally:
                        with self._lock:
                            del self._in_flight[key]
                
                threading.Thread(target=worker).start()
        
        return future.result(timeout=30)  # All callers get same result

# Usage
sf = Singleflight()

def get_trending():
    cache_key = "trending_products"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Only ONE thread queries DB, others wait
    result = sf.do(cache_key, lambda: expensive_database_query())
    r.setex(cache_key, 300, json.dumps(result))
    return result
```

---

## All Three Problems — Summary Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│            CACHE FAILURE MODES COMPARISON                          │
├────────────────┬─────────────────┬────────────────────────────────┤
│                │                  │                                 │
│  STAMPEDE      │  PENETRATION     │  BREAKDOWN                     │
│  (Thundering   │  (Non-existent   │  (Hot key expiry)             │
│   Herd)        │   data attack)   │                                │
│                │                  │                                 │
│  ┌──────────┐ │  ┌──────────┐   │  ┌──────────┐                 │
│  │Many keys │ │  │Invalid   │   │  │ONE hot   │                 │
│  │expire at │ │  │requests  │   │  │key       │                 │
│  │same time │ │  │bypass    │   │  │expires   │                 │
│  └────┬─────┘ │  │cache     │   │  └────┬─────┘                 │
│       │        │  └────┬─────┘   │       │                        │
│       ▼        │       ▼         │       ▼                        │
│  Massive DB    │  Constant DB    │  Spike on one                  │
│  query spike   │  queries for    │  DB row/query                  │
│  (many queries)│  nothing        │  (extreme concurrency)         │
│                │                  │                                 │
│  SOLUTIONS:    │  SOLUTIONS:     │  SOLUTIONS:                    │
│  • Locking     │  • Cache nulls  │  • Never-expire + refresh     │
│  • Early       │  • Bloom filter │  • Singleflight               │
│    refresh     │  • Rate limit   │  • Mutex/lock                 │
│  • Stale-while │    per key      │  • Hot key replication        │
│    -revalidate │                  │                                │
└────────────────┴─────────────────┴────────────────────────────────┘
```

---

## Real-World Example

### How Instagram Handles Cache Stampede

Instagram's Explore page is one of the most trafficked pages on the internet. Cache stampede on explore data would be catastrophic:

```
┌──────────────────────────────────────────────────────────────────┐
│          INSTAGRAM EXPLORE PAGE CACHING                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Approach: "Lease" system (Facebook's Memcache Lease)             │
│                                                                    │
│  1. Request arrives, cache MISS                                   │
│  2. Memcache gives a LEASE TOKEN to the first requester           │
│  3. All subsequent requests see "lease active" → WAIT            │
│  4. First requester fetches from DB, stores result WITH lease     │
│  5. Waiting requests now get the cached result                    │
│                                                                    │
│  Timeline:                                                         │
│  t=0ms   Request A: MISS → gets lease token #7293                │
│  t=1ms   Request B: MISS → sees active lease → wait              │
│  t=2ms   Request C: MISS → sees active lease → wait              │
│  t=20ms  Request A: DB result ready → stores with lease #7293    │
│  t=21ms  Request B: cache HIT → returns                          │
│  t=21ms  Request C: cache HIT → returns                          │
│                                                                    │
│  Only 1 DB query instead of potentially 10,000+                  │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### How Discord Handles Hot Keys

Discord has channels with millions of users (e.g., during game launches):

```
Problem: 
  Channel message cache for #general (1M members)
  If that cache key expires → 1M requests hit DB

Solution:
  1. Hot keys identified automatically (access rate > threshold)
  2. Hot keys get INFINITE TTL (never expire)
  3. Background worker refreshes every 10 seconds
  4. Hot key replicated across multiple Redis nodes
     (key:channel:123:shard1, key:channel:123:shard2, ...)
  5. Client randomly picks a shard → load spread across nodes
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Same TTL for all keys | Mass expiration at same time → stampede | Add random jitter: `TTL = base_ttl + random(0, 60)` |
| Not caching empty results | Penetration attack on non-existent data | Cache "null" with short TTL |
| Lock without timeout | Lock holder crashes → all requests block forever | Always set lock TTL (10-30s) |
| Bloom filter never updated | New data not in filter → treated as non-existent | Update bloom filter on inserts |
| No circuit breaker on DB | Stampede + slow DB = cascading failure | Add circuit breaker (Chapter 12) |
| Singleflight without error handling | One error propagated to ALL waiters | Retry for individual callers on error |

### The TTL Jitter Trick

```python
import random

# ❌ BAD: All product keys expire at exactly the same time
def cache_product_bad(product_id, data):
    r.setex(f"product:{product_id}", 300, data)  # All expire at t+300

# ✅ GOOD: Random jitter prevents synchronized expiration
def cache_product_good(product_id, data):
    jitter = random.randint(0, 60)  # 0-60 seconds random
    r.setex(f"product:{product_id}", 300 + jitter, data)  # Expire between 300-360s
```

---

## When to Use / When NOT to Use These Solutions

| Solution | Use When | Overhead | Complexity |
|----------|----------|----------|------------|
| **Locking/Mutex** | Critical data; moderate traffic | Low (one lock per key) | Medium |
| **Singleflight** | Very hot keys; in-process dedup | Very low | Medium |
| **Stale-while-revalidate** | Slight staleness OK; high traffic | Low | Low |
| **Bloom filter** | Large keyspace; known-set queries | Memory (1.2 bytes/element) | Medium |
| **Cache null values** | Moderate penetration; simple fix | Low | Low |
| **Never-expire + refresh** | Identified hot keys; critical data | Background CPU | Low |
| **TTL jitter** | Mass-populated caches | Zero | Zero |

---

## Key Takeaways

1. **Cache Stampede** = popular key expires → all requests hit DB simultaneously. Fix: locks, singleflight, early refresh.
2. **Cache Penetration** = requests for non-existent data always bypass cache. Fix: cache nulls, Bloom filters.
3. **Cache Breakdown** = single extremely hot key expires. Fix: never-expire + background refresh, singleflight.
4. **Always add TTL jitter** (`base_ttl + random`) to prevent synchronized mass expiration.
5. **Bloom filters** are the most efficient defense against penetration — O(1) check, tiny memory footprint.
6. **Singleflight** pattern ensures only one goroutine/thread fetches data; all others wait for the same result.
7. These problems become **critical at scale** — what works at 100 req/s breaks catastrophically at 100,000 req/s.

---

## What's Next?

Next, we'll explore **Multi-Layer Caching Architecture** — how planet-scale systems combine browser, CDN, application, and distributed caches into a coordinated hierarchy that handles billions of requests while maintaining freshness.

→ [10-multi-layer-caching.md](./10-multi-layer-caching.md)
