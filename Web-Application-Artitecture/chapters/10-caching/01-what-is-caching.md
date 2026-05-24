# What is Caching? Why is it So Important?

> **What you'll learn**: How caching works, why it's the single most impactful optimization you can make, and where caches live in a web application.

---

## Real-Life Analogy

Imagine you're studying for an exam. Your textbook is in a library (the **database**) 20 minutes away. Every time you need to look something up, you walk there, find the book, read the page, and walk back.

Now imagine you photocopy the 10 most important pages and keep them on your desk (the **cache**). The next time you need that information, you just glance at your desk — **instant access**.

That's caching. You're keeping a copy of frequently-used data somewhere **faster and closer** so you don't have to fetch it from the slow, distant source every time.

---

## Core Concept Explained Step-by-Step

### Step 1: The Problem — Latency

Every request to a database, an API, or a disk takes time:

| Operation | Approximate Time |
|-----------|-----------------|
| L1 CPU Cache | 0.5 ns |
| L2 CPU Cache | 7 ns |
| RAM Access | 100 ns |
| SSD Read | 150,000 ns (150 μs) |
| Network (same datacenter) | 500,000 ns (0.5 ms) |
| Database query | 1–50 ms |
| Cross-continent API call | 100–300 ms |

> **Key Insight**: Accessing RAM is **1,000x faster** than reading from disk, and **10,000x faster** than a network round-trip.

### Step 2: The Solution — Keep a Copy Nearby

A **cache** is a temporary storage layer that holds a subset of data, so future requests for that data can be served faster.

```
Without Cache:
┌────────┐         ┌────────────┐
│ Client │ ──────▶ │  Database  │   (slow: 5-50ms)
└────────┘         └────────────┘

With Cache:
┌────────┐    ┌─────────┐    ┌────────────┐
│ Client │ ──▶│  Cache  │    │  Database  │
└────────┘    └─────────┘    └────────────┘
                  │                  │
              HIT? ──▶ Return       │
              MISS? ─────────────▶ Fetch ──▶ Store in Cache ──▶ Return
```

### Step 3: Cache Hit vs Cache Miss

- **Cache Hit**: The data is found in the cache. Fast. ✓
- **Cache Miss**: The data is NOT in the cache. Must fetch from the source. Slow.

```
Request arrives
     │
     ▼
┌─────────────────┐
│ Is it in cache? │
└────────┬────────┘
    YES  │   NO
     │   │    │
     ▼   │    ▼
  Return │  Fetch from DB
  data   │     │
         │     ▼
         │  Store in cache
         │     │
         │     ▼
         │  Return data
         ▼
      [Done]
```

### Step 4: Hit Rate — The Most Important Metric

**Hit Rate** = (Cache Hits) / (Cache Hits + Cache Misses) × 100%

| Hit Rate | Performance Impact |
|----------|-------------------|
| 50% | Moderate improvement |
| 80% | Significant improvement |
| 95% | Massive improvement |
| 99% | Near-instant responses |

> A 95% hit rate means only 5 out of 100 requests actually reach your database.

---

## Where Caches Live in a Web Application

Caches exist at **every layer** of a web application:

```
┌─────────────────────────────────────────────────────────────────┐
│                    CACHING LAYERS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────┐    ┌──────┐    ┌──────────┐    ┌──────────────┐ │
│  │  Browser  │    │ CDN  │    │  App     │    │   Database   │ │
│  │  Cache    │    │      │    │  Server  │    │   Query      │ │
│  │           │    │      │    │  Cache   │    │   Cache      │ │
│  └───────────┘    └──────┘    └──────────┘    └──────────────┘ │
│                                                                   │
│  Layer 1         Layer 2      Layer 3          Layer 4           │
│  (Client)        (Edge)      (Application)    (Database)        │
│                                                                   │
│  Closest ◄─────────────────────────────────────────► Farthest   │
│  to User                                              from User  │
└─────────────────────────────────────────────────────────────────┘
```

| Layer | What's Cached | Example |
|-------|--------------|---------|
| Browser | HTML, CSS, JS, images | Browser HTTP cache |
| CDN | Static assets, API responses | CloudFront, Cloudflare |
| Application | Computed results, session data | Redis, Memcached, in-memory |
| Database | Query results, query plans | MySQL Query Cache, pg_stat |

---

## How It Works Internally

### The Basic Cache Data Structure

At its core, a cache is a **key-value store** (like a hash map/dictionary):

```
Key                          Value
─────────────────────────    ─────────────────────────
"user:1234"                  {"name": "Alice", "email": "..."}
"product:5678:price"         449.99
"homepage:trending"          [list of 20 items]
"session:abc123"             {userId: 1234, role: "admin"}
```

### Cache Metadata

Each cache entry typically stores:
- **Key**: Lookup identifier
- **Value**: The cached data
- **TTL (Time To Live)**: When this entry expires
- **Created At**: When it was cached
- **Access Count**: How many times it's been accessed (for eviction)

### The Cache Lookup Process

```
1. Hash the key ──▶ O(1) lookup
2. Check if entry exists
3. Check if TTL has expired
4. If valid ──▶ return value (HIT)
5. If expired or missing ──▶ MISS
```

---

## Code Examples

### Python — Simple In-Memory Cache

```python
import time

class SimpleCache:
    """A basic cache with TTL (Time To Live) support."""
    
    def __init__(self):
        self._store = {}  # key -> (value, expiry_time)
    
    def get(self, key):
        """Get value from cache. Returns None on miss."""
        if key in self._store:
            value, expiry = self._store[key]
            if time.time() < expiry:
                return value  # Cache HIT
            else:
                del self._store[key]  # Expired, remove it
        return None  # Cache MISS
    
    def set(self, key, value, ttl_seconds=60):
        """Store value in cache with TTL."""
        expiry = time.time() + ttl_seconds
        self._store[key] = (value, expiry)

# Usage
cache = SimpleCache()
cache.set("user:1234", {"name": "Alice"}, ttl_seconds=300)

result = cache.get("user:1234")
if result is None:
    # Cache miss — fetch from database
    result = fetch_from_database("user:1234")
    cache.set("user:1234", result, ttl_seconds=300)
```

### Java — Simple Cache with HashMap and TTL

```java
import java.util.concurrent.ConcurrentHashMap;
import java.time.Instant;

public class SimpleCache<K, V> {
    
    private record CacheEntry<V>(V value, Instant expiresAt) {}
    
    private final ConcurrentHashMap<K, CacheEntry<V>> store = new ConcurrentHashMap<>();

    /** Get value from cache. Returns null on miss. */
    public V get(K key) {
        CacheEntry<V> entry = store.get(key);
        if (entry != null && Instant.now().isBefore(entry.expiresAt())) {
            return entry.value();  // Cache HIT
        }
        store.remove(key);  // Expired or missing
        return null;         // Cache MISS
    }

    /** Store value in cache with TTL in seconds. */
    public void put(K key, V value, long ttlSeconds) {
        Instant expiresAt = Instant.now().plusSeconds(ttlSeconds);
        store.put(key, new CacheEntry<>(value, expiresAt));
    }
}

// Usage
SimpleCache<String, User> cache = new SimpleCache<>();
cache.put("user:1234", user, 300);  // Cache for 5 minutes

User user = cache.get("user:1234");
if (user == null) {
    user = database.findUser("1234");  // Cache miss
    cache.put("user:1234", user, 300);
}
```

---

## Infrastructure Example — Redis as a Cache

Redis is the most popular caching solution. Here's how to use it:

```python
import redis
import json

# Connect to Redis
r = redis.Redis(host='localhost', port=6379, db=0)

def get_user(user_id):
    # Try cache first
    cached = r.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)  # HIT
    
    # Miss — fetch from database
    user = db.query(f"SELECT * FROM users WHERE id = {user_id}")
    
    # Store in Redis with 5-minute TTL
    r.setex(f"user:{user_id}", 300, json.dumps(user))
    
    return user
```

---

## Real-World Example

### How Facebook Uses Caching

Facebook handles **billions of requests per second**. Without caching, their databases would instantly collapse.

```
User Request
     │
     ▼
┌──────────────────────┐
│   Browser Cache      │ ──▶ Cached CSS/JS/Images
│   (Client-side)      │
└──────────┬───────────┘
           │ (if miss)
           ▼
┌──────────────────────┐
│   CDN (Akamai)       │ ──▶ Cached static content globally
└──────────┬───────────┘
           │ (if miss)
           ▼
┌──────────────────────┐
│   Memcached Layer    │ ──▶ Cached user profiles, feeds, sessions
│   (TAO Cache)        │     ~99% hit rate on reads
└──────────┬───────────┘
           │ (if miss, ~1% of requests)
           ▼
┌──────────────────────┐
│   MySQL/TAO DB       │ ──▶ Actual persistent storage
└──────────────────────┘
```

**Key stats from Facebook's TAO system**:
- Serves **billions** of reads per second from cache
- 99.8% cache hit rate on read operations
- Only ~0.2% of requests actually reach the database
- Uses a distributed cache spanning multiple data centers

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Caching everything | Wastes memory, stale data | Cache only hot, expensive data |
| No TTL set | Data becomes stale forever | Always set appropriate TTL |
| Cache without monitoring | Don't know hit rate | Monitor hit/miss ratio |
| Caching dynamic/personal data without care | Security risk, wrong data served | Include user ID in key |
| Assuming cache is always available | Cache can crash/restart | Handle miss gracefully |
| Not warming the cache | Cold start = slow after restart | Pre-load popular data |

---

## When to Use / When NOT to Use

### ✅ Use Caching When:
- Data is **read frequently** and **changes infrequently**
- Database queries are **expensive** (complex joins, aggregations)
- Response time is critical (user-facing pages)
- The same data is requested by **many users** (product pages, feeds)
- Computation is expensive (ML inference results, report generation)

### ❌ Don't Use Caching When:
- Data changes **on every request** (real-time stock prices that must be exact)
- Data is **highly personalized and unique** (each user sees different data, low reuse)
- **Strong consistency** is absolutely required (bank account balance during transfer)
- Data is accessed **only once** (one-time report, no benefit from caching)
- Available memory is extremely limited

---

## Key Takeaways

1. **Caching = storing a copy of data in a faster location** to avoid repeated expensive fetches.
2. **Cache Hit Rate** is the #1 metric — aim for 90%+ in production.
3. Caches exist at **every layer**: browser, CDN, application, database.
4. At its core, a cache is just a **fast key-value store** with expiration.
5. **TTL (Time To Live)** prevents stale data from being served forever.
6. Caching is often the **single biggest performance optimization** you can make — before adding more servers, add caching.
7. The trade-off is always **speed vs freshness** — cached data might be slightly stale.

---

## What's Next?

Next, we'll look at **Application-Level Caching** — how to implement caching directly inside your application using libraries like Guava and Caffeine that give you powerful in-memory caches without needing an external system like Redis.

→ [02-application-level-caching.md](./02-application-level-caching.md)
