# Application-Level Caching (In-Memory: Guava, Caffeine)

> **What you'll learn**: How to implement high-performance caching directly inside your application's memory space, using libraries like Google Guava and Caffeine, without needing any external infrastructure.

---

## Real-Life Analogy

Think of a chef in a busy restaurant. The pantry (database) is in the basement — every trip takes 2 minutes. So the chef keeps the **most-used ingredients on the counter** right next to the stove (application-level cache).

Salt, pepper, olive oil, garlic — they're used in almost every dish, so they stay within arm's reach. The chef only goes to the basement when they need something rare (like saffron).

Application-level caching is exactly this: keeping frequently-used data **inside your application's own memory** — the fastest storage available.

---

## Core Concept Explained Step-by-Step

### Step 1: What is Application-Level Caching?

It's a cache that lives **inside your application process** — in the JVM heap, Python process memory, or Node.js memory. No network calls. No external servers. Just a fast in-memory data structure.

```
┌─────────────────────────────────────────────┐
│           YOUR APPLICATION PROCESS           │
│                                              │
│   ┌──────────────────────────┐              │
│   │   In-Memory Cache        │              │
│   │   (HashMap + Eviction)   │              │
│   │                          │              │
│   │   key1 → value1          │              │
│   │   key2 → value2          │              │
│   │   key3 → value3          │              │
│   └──────────────────────────┘              │
│              │                               │
│              │ miss?                         │
│              ▼                               │
│   ┌──────────────────────────┐              │
│   │   Business Logic         │              │
│   └──────────┬───────────────┘              │
│              │                               │
└──────────────┼───────────────────────────────┘
               │ network call
               ▼
        ┌──────────────┐
        │   Database   │
        └──────────────┘
```

### Step 2: Why Not Just Use a HashMap?

A plain HashMap/Dictionary has problems:

| Problem | Impact |
|---------|--------|
| No size limit | Memory grows until OOM crash |
| No expiration | Stale data forever |
| No eviction policy | Can't remove old entries intelligently |
| No stats | Can't monitor hit rate |
| Not thread-safe | Corruption in multi-threaded apps |

**Caching libraries solve all of these.**

### Step 3: When to Use Application-Level vs Distributed Cache

```
┌────────────────────────────────┬────────────────────────────────┐
│   Application-Level Cache      │   Distributed Cache (Redis)    │
├────────────────────────────────┼────────────────────────────────┤
│ • Single server / instance     │ • Multiple servers / instances │
│ • Nanosecond access (in-RAM)   │ • Sub-millisecond (network)    │
│ • No extra infrastructure      │ • Requires Redis/Memcached     │
│ • Data lost on restart         │ • Data persists across restarts│
│ • Each instance has own copy   │ • Shared across all instances  │
│ • Best for: reference data,    │ • Best for: sessions, shared   │
│   configs, computed results    │   state, rate limiting         │
└────────────────────────────────┴────────────────────────────────┘
```

---

## How It Works Internally

### The Architecture of a Modern Cache Library

```
┌─────────────────────────────────────────────────────────┐
│                 CACHE LIBRARY INTERNALS                   │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────┐ │
│  │ Hash Map │    │  Eviction    │    │  Expiration   │ │
│  │ (O(1)    │    │  Policy      │    │  Manager      │ │
│  │  lookup) │    │  (LRU/LFU/W) │    │  (TTL check)  │ │
│  └──────────┘    └──────────────┘    └───────────────┘ │
│                                                           │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────┐ │
│  │ Size     │    │  Statistics  │    │  Loader /     │ │
│  │ Limiter  │    │  Tracker     │    │  Writer       │ │
│  │          │    │  (hit/miss)  │    │  (auto-fill)  │ │
│  └──────────┘    └──────────────┘    └───────────────┘ │
│                                                           │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Concurrency Control (lock-striping / CAS)        │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Caffeine's Internal Algorithm: W-TinyLFU

Caffeine (Java's best cache library) uses a sophisticated algorithm called **Window-TinyLFU**:

```
New Entry ──▶ ┌──────────────┐     ┌─────────────────┐
              │   Window     │     │   Main Cache     │
              │   Cache      │ ──▶ │   (Segmented     │
              │   (1% size)  │     │    LRU)          │
              └──────────────┘     └─────────────────┘
                                          │
              ┌──────────────┐            │
              │  TinyLFU     │◀───────────┘
              │  Frequency   │   "Should this entry
              │  Sketch      │    be admitted?"
              └──────────────┘
```

1. **Window cache** (1% of space): Admits ALL new entries — gives them a chance
2. **TinyLFU filter**: Decides if a new entry deserves main cache space (based on frequency)
3. **Main cache** (99% of space): Segmented LRU for long-term residents

This achieves **near-optimal hit rates** — better than plain LRU or LFU alone.

---

## Code Examples

### Java — Caffeine Cache (Industry Standard)

```java
import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.LoadingCache;
import java.time.Duration;

public class UserService {
    
    // Build a cache: max 10,000 entries, expire after 5 minutes
    private final LoadingCache<String, User> userCache = Caffeine.newBuilder()
        .maximumSize(10_000)                    // Evict when full
        .expireAfterWrite(Duration.ofMinutes(5)) // TTL: 5 min
        .recordStats()                           // Track hit/miss ratio
        .build(this::loadFromDatabase);          // Auto-load on miss

    /** Called automatically on cache miss */
    private User loadFromDatabase(String userId) {
        return database.query("SELECT * FROM users WHERE id = ?", userId);
    }

    public User getUser(String userId) {
        // Returns cached value or auto-loads from DB
        return userCache.get(userId);
    }

    public void printStats() {
        var stats = userCache.stats();
        System.out.printf("Hit rate: %.2f%%, Hits: %d, Misses: %d%n",
            stats.hitRate() * 100, stats.hitCount(), stats.missCount());
    }
}
```

### Java — Google Guava Cache

```java
import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;
import java.util.concurrent.TimeUnit;

public class ProductService {
    
    private final LoadingCache<String, Product> productCache = CacheBuilder.newBuilder()
        .maximumSize(5_000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .refreshAfterWrite(5, TimeUnit.MINUTES)  // Async refresh before expiry
        .recordStats()
        .build(new CacheLoader<>() {
            @Override
            public Product load(String productId) {
                return database.findProduct(productId);
            }
        });

    public Product getProduct(String productId) {
        return productCache.getUnchecked(productId);
    }
}
```

### Python — Using `cachetools` Library

```python
from cachetools import TTLCache, LRUCache, cached
from cachetools.keys import hashkey
import threading

# Thread-safe TTL cache: max 1000 items, 300s TTL
user_cache = TTLCache(maxsize=1000, ttl=300)
cache_lock = threading.Lock()

def get_user(user_id):
    """Fetch user with caching."""
    with cache_lock:
        if user_id in user_cache:
            return user_cache[user_id]  # HIT
    
    # MISS — fetch from database
    user = database.find_user(user_id)
    
    with cache_lock:
        user_cache[user_id] = user
    
    return user

# Alternative: decorator-based caching
@cached(cache=TTLCache(maxsize=500, ttl=600))
def get_product(product_id):
    """Automatically cached for 10 minutes."""
    return database.find_product(product_id)
```

### Python — Using `functools.lru_cache` (Built-in)

```python
from functools import lru_cache

@lru_cache(maxsize=256)
def compute_expensive_result(query_hash):
    """Cache up to 256 unique results."""
    return heavy_computation(query_hash)

# Check cache stats
print(compute_expensive_result.cache_info())
# CacheInfo(hits=47, misses=12, maxsize=256, currsize=12)
```

---

## Caffeine vs Guava — Feature Comparison

| Feature | Guava Cache | Caffeine |
|---------|-------------|----------|
| Eviction Policy | LRU (size), LRU (weight) | W-TinyLFU (near-optimal) |
| Performance | Good | **2–10x faster** |
| Async Loading | Limited | Full async support |
| Refresh | refreshAfterWrite | Same + async refresh |
| Statistics | Yes | Yes (more detailed) |
| Java Version | Java 8+ | Java 11+ (latest) |
| Recommendation | Legacy projects | **New projects** |

> **Bottom line**: Use Caffeine for new Java projects. It's the successor to Guava Cache, designed by the same person (Ben Manes), with significantly better performance.

---

## Real-World Example

### How Netflix Uses Application-Level Caching

Netflix runs hundreds of microservices. Each service uses **EVCache** (their in-memory caching layer built on Memcached) but ALSO uses local application caches for ultra-hot data:

```
┌─────────────────────────────────────────────────────────────┐
│             NETFLIX MICROSERVICE INSTANCE                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│   Request ──▶ ┌───────────────────┐                         │
│               │  L1: Local Cache   │  (Caffeine, ~100μs)    │
│               │  (Hot data only)   │                         │
│               └────────┬──────────┘                         │
│                   MISS │                                      │
│                        ▼                                      │
│               ┌───────────────────┐                         │
│               │  L2: EVCache      │  (Memcached, ~1ms)      │
│               │  (Shared cluster) │                         │
│               └────────┬──────────┘                         │
│                   MISS │                                      │
│                        ▼                                      │
│               ┌───────────────────┐                         │
│               │  Database/Service │  (5-50ms)               │
│               └───────────────────┘                         │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Why local + distributed?**  
Even a 1ms network call to Redis/Memcached adds up at Netflix scale (millions of requests/second). For ultra-hot keys (like a trending movie's metadata), a local cache eliminates even that 1ms overhead.

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Cache too large | OOM errors, GC pressure | Set `maximumSize` based on object size × count |
| No size limit | Memory leak in disguise | Always configure `maximumSize` or `maximumWeight` |
| Caching mutable objects | Caller modifies cached value, corrupts cache | Cache immutable objects or deep copies |
| Same data cached in multiple places | Inconsistency between caches | Single cache per data type |
| Not monitoring hit rate | Can't tell if cache is effective | Use `.recordStats()` and expose metrics |
| Using local cache in multi-instance setup without thinking | Each instance has different data | Combine with distributed cache for consistency |

### The Mutable Object Trap

```java
// ❌ DANGEROUS: Caching a mutable object
List<String> items = cache.get("items");
items.add("new-item");  // This MODIFIES the cached list!

// ✅ SAFE: Return an unmodifiable copy
List<String> items = List.copyOf(cache.get("items"));
```

---

## When to Use / When NOT to Use

### ✅ Use Application-Level Caching When:
- Data is **read-heavy** (read:write ratio > 10:1)
- You need **sub-microsecond** access times
- Data doesn't need to be **shared** across server instances
- Data is **relatively small** (fits in a fraction of your heap)
- Examples: config values, reference data, feature flags, user permissions

### ❌ Don't Use Application-Level Caching When:
- You have **multiple instances** and need **consistent** cached data → use Redis
- Data changes **very frequently** (every few seconds)
- Cached objects are **very large** (hundreds of MB) → will cause GC issues
- You need **persistence** — local cache is lost on restart
- Strong consistency across nodes is required

---

## Sizing Your Cache — A Practical Guide

```
Formula:
  Max Cache Memory = maxSize × average_object_size

Example:
  10,000 entries × 2 KB average = 20 MB
  100,000 entries × 5 KB average = 500 MB

Rule of thumb:
  • Keep cache < 25% of your total heap
  • JVM: -Xmx4g heap → cache up to ~1 GB
  • Monitor GC impact as cache grows
```

---

## Key Takeaways

1. **Application-level cache** lives inside your process — zero network latency.
2. Use **Caffeine** (Java) or **cachetools** (Python) — never a raw HashMap.
3. Always set **maximumSize** and **TTL** to prevent memory issues.
4. Caffeine's **W-TinyLFU** algorithm provides near-optimal hit rates automatically.
5. Cache **immutable objects** to avoid corruption bugs.
6. Monitor **hit rate** — if it's below 80%, reconsider what you're caching.
7. Local caches work best as a **first layer** (L1) in front of a distributed cache (L2).

---

## What's Next?

Next, we'll explore **Distributed Caching with Redis and Memcached** — what to do when your application runs on multiple servers and needs a shared, consistent cache that all instances can access.

→ [03-distributed-caching.md](./03-distributed-caching.md)
