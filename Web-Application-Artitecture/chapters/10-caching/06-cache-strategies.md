# Cache Strategies (Write-Through, Write-Behind, Write-Around, Read-Through)

> **What you'll learn**: The four fundamental patterns for coordinating reads and writes between your cache and database, when each pattern shines, and the trade-offs that determine which strategy to use.

---

## Real-Life Analogy

Think of a library with a card catalog (cache) and the actual bookshelves (database):

- **Read-Through**: "Find this book" → check card catalog first → if not there, go to shelf, then update the catalog.
- **Write-Through**: When a new book arrives → update BOTH the catalog AND put it on the shelf at the same time.
- **Write-Behind**: When a new book arrives → update the catalog immediately, put it on the shelf later (when you have time).
- **Write-Around**: When a new book arrives → put it directly on the shelf, DON'T update the catalog (it'll be added when someone asks for it).

Each strategy has different trade-offs between speed, consistency, and complexity.

---

## The Four Strategies at a Glance

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CACHE STRATEGIES OVERVIEW                                  │
├─────────────────┬───────────────────┬────────────────┬──────────────────────┤
│                 │  Write Latency    │  Read Latency  │  Consistency         │
├─────────────────┼───────────────────┼────────────────┼──────────────────────┤
│ Write-Through   │  Slow (both)      │  Fast (hit)    │  Strong              │
│ Write-Behind    │  Fast (cache only)│  Fast (hit)    │  Eventual            │
│ Write-Around    │  Fast (DB only)   │  Slow (miss)   │  Eventual            │
│ Read-Through    │  N/A (read only)  │  Fast (hit)    │  Depends on write    │
├─────────────────┼───────────────────┼────────────────┼──────────────────────┤
│ Cache-Aside     │  Medium           │  Fast (hit)    │  Eventual            │
│ (Lazy Loading)  │                   │                │                      │
└─────────────────┴───────────────────┴────────────────┴──────────────────────┘
```

---

## Strategy 1: Cache-Aside (Lazy Loading)

The most common pattern. The **application** manages the cache explicitly.

### How It Works

```
READ PATH:
┌──────────┐      ┌─────────┐      ┌──────────┐
│   App    │─(1)─▶│  Cache  │      │    DB    │
│          │◀(2)──│  HIT?   │      │          │
│          │      └─────────┘      │          │
│          │                        │          │
│          │─(3)── MISS ──────────▶│          │
│          │◀(4)───────────────────│  Result  │
│          │─(5)─▶ Store in cache  │          │
└──────────┘      └─────────┘      └──────────┘

WRITE PATH:
┌──────────┐      ┌─────────┐      ┌──────────┐
│   App    │─(1)─────────────────▶│    DB    │
│          │─(2)─▶│  DELETE  │      │  Write   │
│          │      │  (key)   │      │          │
└──────────┘      └─────────┘      └──────────┘
```

**Steps**:
1. App checks cache
2. If hit → return cached data
3. If miss → query database
4. Get result from database
5. Store result in cache for next time

**On write**: Update database, then **delete** from cache (not update).

### Code — Python

```python
def get_user(user_id):
    """Cache-Aside: app manages cache manually."""
    cache_key = f"user:{user_id}"
    
    # Step 1: Check cache
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)  # HIT
    
    # Step 2: Miss — load from DB
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # Step 3: Populate cache
    redis.setex(cache_key, 300, json.dumps(user))
    
    return user

def update_user(user_id, data):
    """On write: update DB, then DELETE from cache."""
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # Invalidate, don't update
```

### Why DELETE Instead of UPDATE on Write?

```
❌ Updating cache on write (race condition):
  Thread A: Read user from DB (age=30)
  Thread B: Update user in DB (age=31) 
  Thread B: Update cache (age=31)
  Thread A: Update cache (age=30)  ← STALE! Overwrites Thread B's update

✅ Deleting cache on write (safe):
  Thread A: Update DB
  Thread A: Delete cache
  Next read: Cache miss → loads fresh data from DB
```

---

## Strategy 2: Read-Through Cache

The **cache itself** is responsible for loading data on a miss. The application only talks to the cache.

### How It Works

```
┌──────────┐      ┌─────────────────────────┐      ┌──────────┐
│   App    │─(1)─▶│       CACHE              │      │    DB    │
│          │      │                           │      │          │
│          │      │  Has data? ── YES ── (2) ─────▶ │          │
│          │◀─────│  Return to app            │      │          │
│          │      │                           │      │          │
│          │      │  Has data? ── NO          │      │          │
│          │      │    │                      │      │          │
│          │      │    ▼                      │      │          │
│          │      │  Load from DB ──(3)───────────▶ │          │
│          │      │  Store in self  ◀──(4)─────────│  Result  │
│          │      │  Return to app ──(5)──────────▶ │          │
│          │◀─────│                           │      │          │
└──────────┘      └─────────────────────────┘      └──────────┘
```

The app calls `cache.get(key)` — it NEVER calls the database directly. The cache handles loading.

### Code — Java with Caffeine (Read-Through)

```java
// The LoadingCache IS a read-through cache
LoadingCache<String, User> userCache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .build(userId -> {
        // This loader runs automatically on cache miss
        return userRepository.findById(userId);
    });

// App code is beautifully simple:
User user = userCache.get("user:123");
// No need to check for miss — cache handles it!
```

### Difference from Cache-Aside

| Aspect | Cache-Aside | Read-Through |
|--------|-------------|--------------|
| Who loads on miss? | Application code | Cache library |
| App talks to DB? | Yes (on miss) | No — only talks to cache |
| Code complexity | More (manual get/set) | Less (cache handles loading) |
| Flexibility | Maximum | Less (must fit cache API) |

---

## Strategy 3: Write-Through Cache

Every write goes to **both** the cache AND the database **synchronously**. The write isn't considered complete until both succeed.

### How It Works

```
WRITE PATH:
┌──────────┐      ┌─────────────────────────┐      ┌──────────┐
│   App    │─(1)─▶│       CACHE              │      │    DB    │
│          │      │                           │      │          │
│          │      │  Write to self (memory)   │      │          │
│          │      │         │                 │      │          │
│          │      │         ▼                 │      │          │
│          │      │  Write to DB ──(2)────────────▶ │          │
│          │      │                           │      │  Write   │
│          │      │  Both done? ◀──(3)────────────│  Success │
│          │◀(4)──│  Confirm to app           │      │          │
└──────────┘      └─────────────────────────┘      └──────────┘

READ PATH (always hits cache — data is always there):
┌──────────┐      ┌─────────┐
│   App    │─────▶│  Cache  │─── Always HIT (data was written here)
│          │◀─────│         │
└──────────┘      └─────────┘
```

### Code — Python

```python
class WriteThroughCache:
    def __init__(self, redis_client, db_client):
        self.cache = redis_client
        self.db = db_client
    
    def write(self, key, value, ttl=3600):
        """Write to BOTH cache and database atomically."""
        # Write to database first (source of truth)
        self.db.execute(
            "INSERT INTO kv_store (key, value) VALUES (%s, %s) "
            "ON CONFLICT (key) DO UPDATE SET value = %s",
            key, json.dumps(value), json.dumps(value)
        )
        
        # Then write to cache
        self.cache.setex(key, ttl, json.dumps(value))
        
        # Both writes complete before returning
    
    def read(self, key):
        """Read from cache — it's always populated."""
        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)
        
        # Fallback (cold start or eviction)
        result = self.db.execute("SELECT value FROM kv_store WHERE key = %s", key)
        if result:
            self.cache.setex(key, 3600, result['value'])
            return json.loads(result['value'])
        return None
```

### Trade-offs

| Pro | Con |
|-----|-----|
| Cache always has latest data | Write latency is higher (2 writes) |
| Reads are always fast (always hit) | Write throughput is limited |
| Strong consistency | Wasted space if data is write-heavy but rarely read |

---

## Strategy 4: Write-Behind (Write-Back) Cache

Writes go to the cache immediately, and the cache **asynchronously** flushes to the database later (in batches or after a delay).

### How It Works

```
WRITE PATH:
┌──────────┐      ┌────────────────────────────┐      ┌──────────┐
│   App    │─(1)─▶│         CACHE               │      │    DB    │
│          │◀(2)──│  Write to memory             │      │          │
│          │      │  Ack immediately!            │      │          │
│  (FAST!) │      │                              │      │          │
└──────────┘      │  [Background Worker]         │      │          │
                  │    │                          │      │          │
                  │    ▼  (after delay or batch)  │      │          │
                  │  Flush to DB ──(3)────────────────▶ │          │
                  │                              │      │  Write   │
                  └────────────────────────────┘      └──────────┘

Timeline:
  t=0ms    App writes to cache ──▶ Ack (DONE from app's perspective)
  t=0-5s   Cache buffers writes
  t=5s     Cache flushes batch to DB (background)
```

### Code — Java Conceptual Write-Behind

```java
public class WriteBehindCache {
    
    private final ConcurrentHashMap<String, Object> cache = new ConcurrentHashMap<>();
    private final BlockingQueue<WriteOperation> writeQueue = new LinkedBlockingQueue<>();
    
    public WriteBehindCache(Database db) {
        // Background thread flushes writes to DB every 5 seconds
        ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
        executor.scheduleAtFixedRate(() -> flushToDatabase(db), 5, 5, TimeUnit.SECONDS);
    }
    
    /** Write is instant — just updates in-memory cache */
    public void put(String key, Object value) {
        cache.put(key, value);
        writeQueue.add(new WriteOperation(key, value));  // Queue for async flush
    }
    
    /** Read is instant — from memory */
    public Object get(String key) {
        return cache.get(key);
    }
    
    /** Background: batch flush all pending writes to DB */
    private void flushToDatabase(Database db) {
        List<WriteOperation> batch = new ArrayList<>();
        writeQueue.drainTo(batch, 1000);  // Drain up to 1000 ops
        
        if (!batch.isEmpty()) {
            db.batchUpsert(batch);  // Single batch DB write
        }
    }
}
```

### Trade-offs

| Pro | Con |
|-----|-----|
| Extremely fast writes | Data loss risk if cache crashes before flush |
| Batches reduce DB load | Eventual consistency (reads from DB see stale) |
| Great for write-heavy workloads | Complex failure handling |

> **Use case**: Analytics counters, page view counts, IoT sensor data — where losing a few writes is acceptable but speed matters.

---

## Strategy 5: Write-Around Cache

Writes go **directly to the database**, bypassing the cache entirely. The cache only gets populated on subsequent reads (via Cache-Aside or Read-Through).

### How It Works

```
WRITE PATH (bypasses cache):
┌──────────┐                              ┌──────────┐
│   App    │──────── Write directly ─────▶│    DB    │
│          │                               │          │
│          │      ┌─────────┐              │          │
│          │      │  Cache  │ (not touched)│          │
│          │      └─────────┘              │          │
└──────────┘                               └──────────┘

SUBSEQUENT READ (cache gets populated on miss):
┌──────────┐      ┌─────────┐      ┌──────────┐
│   App    │─────▶│  Cache  │      │    DB    │
│          │      │  MISS!  │─────▶│  Read    │
│          │◀─────│ (load)  │◀─────│  Result  │
└──────────┘      └─────────┘      └──────────┘
```

### When Write-Around Makes Sense

```
Scenario: E-commerce order system

Writes: Customer places order → Write to DB (rarely read immediately)
Reads: Customer views order history → First read = cache miss, subsequent = hits

Why write-around?
  - Newly written data is NOT immediately read in most cases
  - Caching on write would waste memory for data nobody reads
  - Only cache data that's actually requested (demand-driven)
```

### Code — Python

```python
def create_order(user_id, items):
    """Write directly to DB — don't populate cache."""
    order = db.execute(
        "INSERT INTO orders (user_id, items, status) VALUES (%s, %s, 'pending') RETURNING *",
        user_id, json.dumps(items)
    )
    # NOTE: We do NOT write to cache here
    # It'll get cached when someone reads it
    return order

def get_order(order_id):
    """Read with cache-aside — populates cache on first read."""
    cache_key = f"order:{order_id}"
    
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    order = db.query("SELECT * FROM orders WHERE id = %s", order_id)
    redis.setex(cache_key, 600, json.dumps(order))
    return order
```

---

## Comparison Matrix — All Strategies

```
┌───────────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│                   │ Cache-Aside  │ Read-Through │Write-Through │ Write-Behind │
├───────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Read performance  │ HIT: fast    │ HIT: fast    │ Always fast  │ Always fast  │
│                   │ MISS: slow   │ MISS: auto   │ (in cache)   │ (in cache)   │
├───────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Write performance │ Medium       │ N/A          │ Slow(2 writes│ Very fast    │
│                   │ (DB + delete)│ (read only)  │ synchronous) │ (cache only) │
├───────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Consistency       │ Eventual     │ Eventual     │ Strong       │ Eventual     │
├───────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Data loss risk    │ None         │ None         │ None         │ YES (crash)  │
├───────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Complexity        │ Simple       │ Medium       │ Medium       │ High         │
├───────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Best for          │ General      │ Read-heavy   │ Read-heavy + │ Write-heavy  │
│                   │ purpose      │ apps         │ consistency  │ apps         │
└───────────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

---

## Real-World Example

### How Amazon DynamoDB Accelerator (DAX) Uses Read/Write-Through

DAX is Amazon's managed cache for DynamoDB, implementing both read-through and write-through:

```
┌─────────────────────────────────────────────────────────────┐
│                      DAX (Write-Through + Read-Through)       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Application                                                  │
│      │                                                        │
│      ▼                                                        │
│  ┌──────────────────────────────────┐                        │
│  │          DAX Cluster              │                        │
│  │                                    │                        │
│  │  READ:                             │                        │
│  │  get_item("user:123")             │                        │
│  │    → Check cache → HIT: return    │                        │
│  │    → MISS: read from DynamoDB     │                        │
│  │      → cache result → return      │                        │
│  │                                    │                        │
│  │  WRITE:                            │                        │
│  │  put_item("user:123", {...})      │                        │
│  │    → Write to DynamoDB            │                        │
│  │    → Write to cache               │                        │
│  │    → Return success               │                        │
│  └──────────────────┬───────────────┘                        │
│                     │                                          │
│                     ▼                                          │
│  ┌──────────────────────────────────┐                        │
│  │          DynamoDB Table            │                        │
│  └──────────────────────────────────┘                        │
│                                                               │
│  Result: 10x read performance, microsecond latency            │
└─────────────────────────────────────────────────────────────┘
```

### How Facebook Uses Write-Behind for Counters

```
Like button clicked → Increment counter in cache (instant)
                   → Background job writes to MySQL every few seconds

Why? 100,000 likes/second on a popular post
  - Write-through: 100K DB writes/sec → DB melts
  - Write-behind: 100K cache writes + 1 DB write every 5s with final count
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Write-through for write-heavy data | Doubles write latency for no benefit | Use write-behind for high-write scenarios |
| Write-behind without durability | Data loss on cache crash | Use Redis AOF or periodic checkpoints |
| Read-through with no TTL | Data lives forever in cache | Always configure expiration |
| Cache-aside with "set on write" | Race conditions between readers and writers | Delete on write, populate on read |
| Mixing strategies inconsistently | Some paths update cache, others don't → stale data | Choose ONE strategy per data type |
| Write-around for frequently-read-after-write data | First read after write is always slow | Use write-through for read-after-write patterns |

---

## Decision Guide: Which Strategy to Choose?

```
Start here:
     │
     ▼
Is data read-heavy or write-heavy?
     │
     ├── READ-HEAVY (10:1+ read:write)
     │       │
     │       ├── Need strong consistency? → Write-Through + Read-Through
     │       │
     │       ├── OK with eventual consistency? → Cache-Aside (simplest)
     │       │
     │       └── Need auto-loading? → Read-Through
     │
     └── WRITE-HEAVY (many writes, fewer reads)
             │
             ├── Can tolerate data loss? → Write-Behind (fastest writes)
             │
             ├── Data rarely read after write? → Write-Around
             │
             └── Need durability? → Write-Through (slower but safe)
```

---

## Key Takeaways

1. **Cache-Aside** (Lazy Loading) is the most common and simplest — app checks cache, loads on miss, deletes on write.
2. **Read-Through** — the cache itself loads data on miss; app never talks to DB directly for reads.
3. **Write-Through** — every write updates both cache AND DB synchronously; strong consistency but slower writes.
4. **Write-Behind** — writes go to cache only, flushed to DB asynchronously; fastest writes but risk of data loss.
5. **Write-Around** — writes go to DB only, cache populated on demand; best when newly written data isn't immediately read.
6. On write, **DELETE from cache** (not update) to avoid race conditions in cache-aside pattern.
7. You can **combine strategies** — e.g., read-through + write-behind for different workloads.

---

## What's Next?

Next, we'll tackle the **hardest problem in computer science** — **Cache Invalidation**. How do you know when to remove or update cached data? Get it wrong, and users see stale data. Get it too aggressive, and you lose all caching benefit.

→ [07-cache-invalidation.md](./07-cache-invalidation.md)
