# Cache Invalidation — The Hardest Problem in CS

> **What you'll learn**: Why cache invalidation is notoriously difficult, the strategies to do it correctly (TTL, event-driven, version-based), the subtle race conditions that trip up even experienced engineers, and how production systems handle it at scale.

---

## Real-Life Analogy

Imagine you run a news website. You print 10,000 copies of today's newspaper (cache) and distribute them to newsstands across the city. At 2 PM, a massive breaking story drops.

**The problem**: Those 10,000 copies are now **wrong**. How do you fix this?

- **Option A (TTL)**: Print new editions every hour. People get stale news for up to 59 minutes.
- **Option B (Event-driven)**: Send a motorcycle courier to EVERY newsstand with the correction. Expensive but immediate.
- **Option C (Version-based)**: Print "v2" and tell stands to throw away "v1" when they get the new one.
- **Option D (On-demand)**: Don't fix it. When someone asks "is this today's paper?", check if newer version exists.

Each has costs. None is perfect. **That's** why Phil Karlton said:

> "There are only two hard things in Computer Science: cache invalidation and naming things."

---

## Core Concept Explained Step-by-Step

### Step 1: What is Cache Invalidation?

Cache invalidation = **removing or updating stale (outdated) data** from the cache so users don't see old information.

```
The Timeline of a Cache Problem:

t=0s    Cache stores: product:123 → {price: $99}
t=30s   Admin updates price in DB to $79
t=31s   User requests product:123
        Cache returns: $99 ← WRONG! Should be $79

This gap between "data changed" and "cache reflects change" 
is the fundamental problem.
```

### Step 2: Why is it SO Hard?

```
┌──────────────────────────────────────────────────────────────────┐
│              WHY CACHE INVALIDATION IS HARD                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. KNOWING WHAT TO INVALIDATE                                    │
│     Product price changed → which cached pages show this product? │
│     Category page? Search results? Recommendations? Related?      │
│                                                                    │
│  2. KNOWING WHEN TO INVALIDATE                                    │
│     Too early → lose caching benefit                              │
│     Too late → stale data served                                  │
│                                                                    │
│  3. RACE CONDITIONS                                               │
│     Two updates + caching = can end up with wrong version         │
│                                                                    │
│  4. DISTRIBUTED SYSTEMS                                           │
│     Cache on server A, cache on server B, CDN in 50 locations... │
│     How to invalidate ALL of them atomically?                     │
│                                                                    │
│  5. CASCADE EFFECTS                                               │
│     One entity change can affect hundreds of cached queries       │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## Invalidation Strategies

### Strategy 1: Time-To-Live (TTL)

The simplest approach: data **automatically expires** after a set time.

```
┌─────────────────────────────────────────────────────┐
│                TTL-BASED INVALIDATION                 │
├─────────────────────────────────────────────────────┤
│                                                       │
│  SET product:123 → {price: $99}  TTL=300s            │
│                                                       │
│  Timeline:                                            │
│  ├── 0s     Data cached (fresh)                      │
│  ├── 150s   Still fresh, served from cache           │
│  ├── 300s   EXPIRED! Next request = cache miss       │
│  └── 301s   Fetches new data from DB (price=$79)    │
│                                                       │
│  Max staleness: 300 seconds (worst case)             │
│                                                       │
└─────────────────────────────────────────────────────┘
```

| Short TTL (5-30s) | Long TTL (5-60 min) |
|-------------------|---------------------|
| More fresh data | More cache hits |
| More DB load (frequent misses) | Less DB load |
| Good for: rapidly changing data | Good for: stable reference data |

### Strategy 2: Event-Driven Invalidation

When data changes, an **event** triggers immediate cache deletion.

```
┌──────────┐     ┌─────────────┐     ┌─────────┐     ┌────────────┐
│  Writer  │────▶│  Database   │────▶│  Event  │────▶│   Cache    │
│          │     │  (UPDATE)   │     │  Bus    │     │ (DELETE key)│
└──────────┘     └─────────────┘     └─────────┘     └────────────┘
                                          │
                                          ├────▶ Cache Server 1: DELETE
                                          ├────▶ Cache Server 2: DELETE
                                          └────▶ CDN: PURGE
```

```python
# Event-driven invalidation using Redis Pub/Sub
import redis

publisher = redis.Redis()
subscriber = redis.Redis()

# PUBLISHER SIDE (after DB write)
def update_product(product_id, new_data):
    db.execute("UPDATE products SET price = %s WHERE id = %s", 
               new_data['price'], product_id)
    
    # Publish invalidation event
    publisher.publish('cache_invalidation', json.dumps({
        'type': 'product_updated',
        'keys': [f'product:{product_id}', f'category:{new_data["category"]}'],
        'timestamp': time.time()
    }))

# SUBSCRIBER SIDE (cache invalidation worker)
def invalidation_listener():
    pubsub = subscriber.pubsub()
    pubsub.subscribe('cache_invalidation')
    
    for message in pubsub.listen():
        if message['type'] == 'message':
            event = json.loads(message['data'])
            for key in event['keys']:
                redis_cache.delete(key)  # Immediate invalidation
```

### Strategy 3: Version-Based Invalidation

Instead of deleting old cache entries, **change the cache key** so old entries are naturally ignored.

```
Version 1: cache key = "product:123:v1"  → {price: $99}
Version 2: cache key = "product:123:v2"  → {price: $79}

App always reads the CURRENT version key.
Old versions expire naturally via TTL.

┌───────────────────────────────────────────────────────┐
│  Version tracking in database:                         │
│                                                         │
│  Table: products                                        │
│  ┌────┬───────┬───────┬─────────┐                     │
│  │ id │ price │ name  │ version │                     │
│  ├────┼───────┼───────┼─────────┤                     │
│  │123 │  $79  │ Widget│    2    │                     │
│  └────┴───────┴───────┴─────────┘                     │
│                                                         │
│  Cache key = f"product:{id}:v{version}"                │
│  On update: increment version → new cache key          │
│  Old key expires naturally                              │
└───────────────────────────────────────────────────────┘
```

### Strategy 4: Write-Invalidate vs Write-Update

```
WRITE-INVALIDATE (recommended):
  On write → DELETE the cache key
  Next read → cache miss → loads fresh data
  
  ✅ Simple, safe from race conditions
  ❌ First read after write is slower (miss)

WRITE-UPDATE:
  On write → UPDATE the cache key with new value
  
  ✅ No miss after write
  ❌ Race condition risk (see below)
```

---

## How It Works Internally

### The Race Condition Problem

This is the most subtle and dangerous bug in caching:

```
SCENARIO: Two processes update the same key

Timeline:
  t=1  Process A: reads product from DB (price=$99, version=1)
  t=2  Process B: updates DB (price=$79, version=2)
  t=3  Process B: updates cache with {price: $79}
  t=4  Process A: updates cache with {price: $99}  ← STALE DATA IN CACHE!

Result: Cache has $99, DB has $79 → INCONSISTENT

This is why DELETE is safer than UPDATE:
  t=2  Process B: updates DB
  t=3  Process B: DELETES cache key
  t=4  Process A: DELETES cache key (harmless double-delete)
  t=5  Next read: cache miss → reads $79 from DB ✓
```

### The Double-Delete Pattern

```
┌──────────────────────────────────────────────────────────────┐
│              DOUBLE-DELETE PATTERN                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Problem: Delete cache → Update DB → another thread reads     │
│           stale DB data during update and recaches it!         │
│                                                                │
│  Solution: Delete TWICE with a delay                           │
│                                                                │
│  Step 1: DELETE cache key                                      │
│  Step 2: UPDATE database                                       │
│  Step 3: Wait ~500ms (let in-flight reads complete)           │
│  Step 4: DELETE cache key AGAIN                                │
│                                                                │
│  Timeline:                                                     │
│  t=0    DELETE "product:123"                                   │
│  t=1    UPDATE DB (price → $79)                               │
│  t=2    [other thread reads stale $99 from DB replica lag]    │
│  t=3    [other thread caches $99]                             │
│  t=500  DELETE "product:123" AGAIN ← fixes the stale entry!  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Code — Double Delete in Practice

```python
import time
import threading

def update_product_safely(product_id, new_price):
    cache_key = f"product:{product_id}"
    
    # Step 1: Invalidate cache
    redis.delete(cache_key)
    
    # Step 2: Update database
    db.execute("UPDATE products SET price = %s WHERE id = %s", new_price, product_id)
    
    # Step 3: Delayed second invalidation (handles race condition)
    def delayed_delete():
        time.sleep(0.5)  # Wait for in-flight reads to complete
        redis.delete(cache_key)
    
    threading.Thread(target=delayed_delete, daemon=True).start()
```

---

## Code Examples

### Java — Event-Driven Invalidation with Kafka

```java
// Producer: Emit invalidation event after DB update
@Service
public class ProductService {
    
    @Autowired private KafkaTemplate<String, String> kafka;
    @Autowired private RedisTemplate<String, Object> redis;
    
    @Transactional
    public void updateProduct(Long productId, ProductUpdateDto dto) {
        // Update database
        productRepo.save(mapToEntity(dto));
        
        // Emit invalidation event
        CacheInvalidationEvent event = new CacheInvalidationEvent(
            "product_updated",
            List.of("product:" + productId, "category:" + dto.getCategory()),
            Instant.now()
        );
        kafka.send("cache-invalidation", productId.toString(), 
                   objectMapper.writeValueAsString(event));
    }
}

// Consumer: Process invalidation events
@KafkaListener(topics = "cache-invalidation")
public void handleInvalidation(String message) {
    CacheInvalidationEvent event = objectMapper.readValue(message, 
                                     CacheInvalidationEvent.class);
    
    for (String key : event.getKeys()) {
        redis.delete(key);
        log.info("Invalidated cache key: {}", key);
    }
}
```

### Python — Tag-Based Invalidation

```python
class TagBasedInvalidation:
    """
    Associate cache entries with tags.
    Invalidating a tag removes ALL entries tagged with it.
    """
    
    def __init__(self, redis_client):
        self.r = redis_client
    
    def set_with_tags(self, key, value, ttl, tags):
        """Store value and associate with tags."""
        pipe = self.r.pipeline()
        pipe.setex(key, ttl, json.dumps(value))
        for tag in tags:
            pipe.sadd(f"tag:{tag}", key)
            pipe.expire(f"tag:{tag}", ttl + 3600)
        pipe.execute()
    
    def invalidate_tag(self, tag):
        """Remove ALL cache entries associated with a tag."""
        keys = self.r.smembers(f"tag:{tag}")
        if keys:
            pipe = self.r.pipeline()
            for key in keys:
                pipe.delete(key)
            pipe.delete(f"tag:{tag}")
            pipe.execute()
        return len(keys)

# Usage
cache = TagBasedInvalidation(redis)

# Store product, tagged with its category and "all_products"
cache.set_with_tags(
    "product:123", product_data, ttl=600,
    tags=["products", "category:electronics", "brand:apple"]
)

# When ANY electronics product changes:
cache.invalidate_tag("category:electronics")
# → Removes ALL products in that category from cache
```

---

## Real-World Example

### How Facebook Handles Cache Invalidation at Scale

Facebook uses **TAO** (The Associations and Objects cache) — a distributed cache layer over MySQL. Invalidation is critical at their scale:

```
┌──────────────────────────────────────────────────────────────────┐
│          FACEBOOK TAO CACHE INVALIDATION                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Write: User updates profile picture                              │
│       │                                                            │
│       ▼                                                            │
│  ┌──────────────────────────┐                                     │
│  │  TAO Leader (write region)│                                     │
│  │  1. Write to MySQL        │                                     │
│  │  2. Invalidate local cache│                                     │
│  │  3. Send invalidation to  │                                     │
│  │     ALL other regions     │                                     │
│  └──────────────┬───────────┘                                     │
│                 │                                                   │
│     ┌───────────┼───────────────────────┐                         │
│     ▼           ▼                       ▼                         │
│  ┌────────┐  ┌────────┐            ┌────────┐                    │
│  │Region 2│  │Region 3│   ...      │Region N│                    │
│  │Follower│  │Follower│            │Follower│                    │
│  │        │  │        │            │        │                    │
│  │DELETE   │  │DELETE   │            │DELETE   │                    │
│  │from    │  │from    │            │from    │                    │
│  │local   │  │local   │            │local   │                    │
│  │cache   │  │cache   │            │cache   │                    │
│  └────────┘  └────────┘            └────────┘                    │
│                                                                    │
│  Cross-region invalidation latency: ~50-100ms                     │
│  During this window, other regions may serve stale data           │
│  (Acceptable trade-off for Facebook's use case)                   │
└──────────────────────────────────────────────────────────────────┘
```

**Key decisions**:
- **Eventual consistency** is acceptable (profile pic might be stale for 100ms in other regions)
- Invalidation via **async message passing**, not synchronous
- Reads during invalidation window get slightly stale data (users barely notice)

---

## Advanced: Change Data Capture (CDC) for Invalidation

The most reliable enterprise approach — listen to the database's own change log:

```
┌──────────────────────────────────────────────────────────────────┐
│          CDC-BASED CACHE INVALIDATION                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────┐    ┌────────────────┐    ┌──────────────────────┐ │
│  │   App    │───▶│   PostgreSQL   │    │   Debezium (CDC)      │ │
│  │  (write) │    │                 │───▶│   Reads WAL/binlog    │ │
│  └──────────┘    │  WAL / binlog   │    │                        │ │
│                   └────────────────┘    └───────────┬────────────┘ │
│                                                      │              │
│                                              ┌───────▼────────┐   │
│                                              │     Kafka      │   │
│                                              │  (change events)│   │
│                                              └───────┬────────┘   │
│                                                      │              │
│                                              ┌───────▼────────┐   │
│                                              │  Invalidation  │   │
│                                              │  Consumer      │   │
│                                              │  → DELETE keys  │   │
│                                              └────────────────┘   │
│                                                                    │
│  Advantage: Works even if app forgets to invalidate!              │
│  The database itself is the source of truth for changes.          │
└──────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| No invalidation strategy at all | Users see stale data indefinitely | Always have at least TTL as fallback |
| Updating cache instead of deleting on write | Race conditions → inconsistency | DELETE on write, let next read repopulate |
| Not invalidating related/derived caches | Product changes but category listing still stale | Use tags or map entity → all affected keys |
| Synchronous invalidation in hot path | Slows down write operations | Async invalidation (event-driven) |
| Using Redis `KEYS` pattern for invalidation | O(n) scan blocks Redis | Use `SCAN`, sets for tags, or explicit key tracking |
| No TTL as safety net | Failed invalidation = permanently stale | Always set TTL even with event-driven invalidation |

---

## When to Use / When NOT to Use Each Strategy

| Strategy | Use When | Avoid When |
|----------|----------|------------|
| **TTL only** | Data can tolerate staleness (mins); simple system | Sub-second freshness needed |
| **Event-driven** | Need near-instant invalidation; have event infrastructure | Simple app without message bus |
| **Version-based** | Immutable data with version increments; CDN caching | Many small frequent updates |
| **CDC (Debezium)** | Enterprise; multiple apps write to same DB; can't modify all writers | Simple single-app setups (overkill) |
| **Double-delete** | Read replicas with lag; paranoid about race conditions | Single DB without replication lag |

---

## Key Takeaways

1. **Cache invalidation is hard** because you must know WHAT changed, WHAT cache keys are affected, and handle race conditions.
2. **TTL is the safety net** — even with active invalidation, always set a TTL as fallback.
3. **Delete on write, don't update** — avoids subtle race conditions between concurrent writers.
4. **Event-driven invalidation** (Kafka, Redis Pub/Sub) gives near-instant freshness without coupling.
5. **Tag-based invalidation** maps "entity changed" → "all affected cache keys" — essential for derived/aggregated data.
6. **CDC (Change Data Capture)** is the most robust approach — the database itself tells you what changed.
7. At scale (like Facebook), **eventual consistency** in cache invalidation is often acceptable — optimize for the common case, not the edge case.

---

## What's Next?

Next, we'll explore **Cache Eviction Policies** — what happens when your cache is FULL and needs to make room for new data. LRU, LFU, TTL, and more — algorithms that decide which cached data to throw away.

→ [08-cache-eviction-policies.md](./08-cache-eviction-policies.md)
