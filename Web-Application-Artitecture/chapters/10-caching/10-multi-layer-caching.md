# Multi-Layer Caching Architecture

> **What you'll learn**: How planet-scale systems orchestrate multiple caching layers (browser → CDN → application → distributed → database) into a unified hierarchy that handles billions of requests while maintaining data freshness and consistency.

---

## Real-Life Analogy

Think of how a country's water system works:

1. **Your glass** (Browser cache) — water you already poured, instantly available
2. **Kitchen tap** (Application cache) — local reservoir, instant flow
3. **Neighborhood water tower** (Distributed cache) — shared among houses, very close
4. **Regional treatment plant** (CDN/Edge) — serves the whole area
5. **River/Aquifer** (Database) — the ultimate source, but expensive to tap directly

Each layer **absorbs demand** so the layers below don't get overwhelmed. If everyone tapped the river directly, it would run dry.

Multi-layer caching works the same way — **each layer shields the next**, and only a tiny fraction of requests ever reach the origin database.

---

## Core Concept Explained Step-by-Step

### Step 1: The Cache Hierarchy

```
┌──────────────────────────────────────────────────────────────────────┐
│                   MULTI-LAYER CACHE ARCHITECTURE                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Layer 0: CLIENT/BROWSER CACHE                                        │
│  ┌────────────────────────────────────┐                              │
│  │ • HTTP Cache (Cache-Control)        │  Latency: 0ms (local)       │
│  │ • Service Worker cache              │  Hit rate target: 60-80%    │
│  │ • LocalStorage/IndexedDB            │                              │
│  └───────────────────┬────────────────┘                              │
│                  MISS │                                                │
│                       ▼                                                │
│  Layer 1: CDN / EDGE CACHE                                           │
│  ┌────────────────────────────────────┐                              │
│  │ • CloudFront, Cloudflare, Akamai   │  Latency: 1-10ms            │
│  │ • Geographically distributed       │  Hit rate target: 80-95%    │
│  │ • Static + some dynamic content    │                              │
│  └───────────────────┬────────────────┘                              │
│                  MISS │                                                │
│                       ▼                                                │
│  Layer 2: APPLICATION LOCAL CACHE (L1)                                │
│  ┌────────────────────────────────────┐                              │
│  │ • Caffeine, Guava, in-process      │  Latency: ~1μs              │
│  │ • Per-instance, not shared         │  Hit rate target: 50-80%    │
│  │ • Hot data only (small)            │                              │
│  └───────────────────┬────────────────┘                              │
│                  MISS │                                                │
│                       ▼                                                │
│  Layer 3: DISTRIBUTED CACHE (L2)                                      │
│  ┌────────────────────────────────────┐                              │
│  │ • Redis, Memcached, EVCache        │  Latency: 0.5-2ms           │
│  │ • Shared across all instances      │  Hit rate target: 90-99%    │
│  │ • Large capacity (100s of GB)      │                              │
│  └───────────────────┬────────────────┘                              │
│                  MISS │                                                │
│                       ▼                                                │
│  Layer 4: DATABASE + STORAGE                                          │
│  ┌────────────────────────────────────┐                              │
│  │ • PostgreSQL, MongoDB, DynamoDB    │  Latency: 5-50ms            │
│  │ • Source of truth                   │  Goal: <5% of reads reach  │
│  │ • Buffer pool (internal cache)      │  here                       │
│  └────────────────────────────────────┘                              │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Step 2: How Traffic Reduces at Each Layer

```
Incoming requests: 1,000,000 / second

Layer 0 (Browser):     60% hit → 400,000 pass through
Layer 1 (CDN):         85% hit → 60,000 pass through
Layer 2 (Local L1):    70% hit → 18,000 pass through
Layer 3 (Redis L2):    95% hit → 900 pass through
Layer 4 (Database):    receives only 900 queries/sec (0.09% of total!)

Without caching: 1,000,000 DB queries/second → impossible
With caching: 900 DB queries/second → easy!
```

### Step 3: What Lives in Each Layer

| Layer | What's Cached | Size | TTL |
|-------|---------------|------|-----|
| Browser | Static assets, API responses user saw | 50-500 MB | Hours to years |
| CDN | Static assets, public API responses | TB scale across PoPs | Minutes to days |
| Local (L1) | Ultra-hot data, config, feature flags | 10-500 MB per instance | Seconds to minutes |
| Distributed (L2) | Session data, query results, computed data | 10-500 GB cluster | Minutes to hours |
| DB Internal | Data pages, query plans | Configurable (shared_buffers) | Until evicted |

---

## How It Works Internally

### The Request Flow Through All Layers

```
User clicks "View Product #5678"
│
├─[Browser Cache]── Has /api/products/5678 cached? (Cache-Control: max-age=60)
│   ├── YES → Serve from browser (0ms) ✓ DONE
│   └── NO → Continue...
│
├─[CDN]── Edge server has /api/products/5678?
│   ├── YES → Serve from edge (5ms) ✓ DONE
│   └── NO → Forward to origin
│
├─[Load Balancer → App Server Instance]
│   │
│   ├─[L1 Local Cache]── Caffeine has "product:5678"?
│   │   ├── YES → Return from memory (0.001ms) ✓ DONE
│   │   └── NO → Continue...
│   │
│   ├─[L2 Redis]── Redis has "product:5678"?
│   │   ├── YES → Return from Redis (0.5ms), store in L1 ✓ DONE
│   │   └── NO → Continue...
│   │
│   └─[Database]── SELECT * FROM products WHERE id = 5678
│       │   (5ms)
│       │   Store result in Redis (L2) with TTL=600s
│       │   Store result in L1 with TTL=30s
│       └── Return to user ✓ DONE
│
└── Response sent with Cache-Control headers for browser + CDN
```

### Consistency Across Layers — The Invalidation Cascade

When data changes, invalidation must flow **outward** through all layers:

```
Product price updated: $99 → $79
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│                INVALIDATION CASCADE                             │
├───────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: Update Database                                       │
│       UPDATE products SET price = 79 WHERE id = 5678           │
│                                                                 │
│  Step 2: Invalidate L2 (Redis)                                 │
│       DEL product:5678                                         │
│       PUBLISH cache_invalidation "product:5678"                │
│                                                                 │
│  Step 3: Invalidate L1 (All instances via pub/sub)             │
│       All app instances subscribe to invalidation channel      │
│       Each instance: localCache.invalidate("product:5678")     │
│                                                                 │
│  Step 4: Invalidate CDN                                        │
│       POST /purge to CDN API for /api/products/5678            │
│       Or: wait for CDN TTL to expire (simpler)                 │
│                                                                 │
│  Step 5: Browser cache                                         │
│       Can't directly invalidate! Must wait for max-age expiry  │
│       Or: use ETag/Last-Modified for conditional requests      │
│                                                                 │
└───────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Complete Multi-Layer Cache Implementation

```python
import redis
import json
import time
from cachetools import TTLCache
import threading

class MultiLayerCache:
    """
    L1: In-process TTLCache (fastest, per-instance)
    L2: Redis (shared across instances)
    L3: Database (source of truth)
    """
    
    def __init__(self, redis_client, db_client, l1_size=1000, l1_ttl=30, l2_ttl=300):
        self.l1 = TTLCache(maxsize=l1_size, ttl=l1_ttl)
        self.l1_lock = threading.Lock()
        self.l2 = redis_client
        self.db = db_client
        self.l2_ttl = l2_ttl
        
        # Subscribe to invalidation events
        self._start_invalidation_listener()
    
    def get(self, key, loader_fn):
        """
        Try L1 → L2 → Database.
        On miss, populate all higher layers.
        """
        # L1: In-process cache (microseconds)
        with self.l1_lock:
            if key in self.l1:
                return self.l1[key]  # L1 HIT
        
        # L2: Redis (sub-millisecond)
        cached = self.l2.get(key)
        if cached:
            value = json.loads(cached)
            with self.l1_lock:
                self.l1[key] = value  # Promote to L1
            return value  # L2 HIT
        
        # L3: Database (milliseconds)
        value = loader_fn()
        
        # Populate L2
        self.l2.setex(key, self.l2_ttl, json.dumps(value, default=str))
        
        # Populate L1
        with self.l1_lock:
            self.l1[key] = value
        
        return value
    
    def invalidate(self, key):
        """Invalidate across all layers."""
        # Remove from L1
        with self.l1_lock:
            self.l1.pop(key, None)
        
        # Remove from L2
        self.l2.delete(key)
        
        # Notify other instances to invalidate their L1
        self.l2.publish("cache_invalidation", key)
    
    def _start_invalidation_listener(self):
        """Listen for invalidation events from other instances."""
        def listener():
            pubsub = self.l2.pubsub()
            pubsub.subscribe("cache_invalidation")
            for msg in pubsub.listen():
                if msg['type'] == 'message':
                    key = msg['data'].decode()
                    with self.l1_lock:
                        self.l1.pop(key, None)
        
        thread = threading.Thread(target=listener, daemon=True)
        thread.start()

# Usage
cache = MultiLayerCache(
    redis_client=redis.Redis(host='redis-cluster', port=6379),
    db_client=database,
    l1_size=5000,    # 5000 items in local cache
    l1_ttl=30,       # Local cache: 30s TTL
    l2_ttl=300       # Redis: 5min TTL
)

def get_product(product_id):
    return cache.get(
        f"product:{product_id}",
        loader_fn=lambda: db.query("SELECT * FROM products WHERE id = %s", product_id)
    )

def update_product(product_id, new_data):
    db.execute("UPDATE products SET ... WHERE id = %s", product_id)
    cache.invalidate(f"product:{product_id}")
```

### Java — Spring Boot Multi-Layer Cache

```java
import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.Cache;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.listener.ChannelTopic;

@Service
public class MultiLayerCacheService {
    
    // L1: Local Caffeine cache (per instance)
    private final Cache<String, Object> l1Cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofSeconds(30))
        .recordStats()
        .build();
    
    // L2: Redis (shared)
    @Autowired private RedisTemplate<String, String> redis;
    @Autowired private ObjectMapper mapper;
    
    public <T> T get(String key, Class<T> type, Supplier<T> dbLoader) {
        // L1 check
        Object l1Value = l1Cache.getIfPresent(key);
        if (l1Value != null) {
            return type.cast(l1Value);
        }
        
        // L2 check (Redis)
        String l2Value = redis.opsForValue().get(key);
        if (l2Value != null) {
            T value = mapper.readValue(l2Value, type);
            l1Cache.put(key, value);  // Promote to L1
            return value;
        }
        
        // Database
        T value = dbLoader.get();
        
        // Populate L2 (5 min TTL)
        redis.opsForValue().set(key, mapper.writeValueAsString(value), 
                                Duration.ofMinutes(5));
        // Populate L1
        l1Cache.put(key, value);
        
        return value;
    }
    
    public void invalidate(String key) {
        l1Cache.invalidate(key);
        redis.delete(key);
        // Notify other instances
        redis.convertAndSend("cache-invalidation", key);
    }
    
    // Listen for invalidation from other instances
    @RedisListener(topic = "cache-invalidation")
    public void onInvalidation(String key) {
        l1Cache.invalidate(key);
    }
}

// Usage in Controller
@GetMapping("/products/{id}")
public Product getProduct(@PathVariable String id) {
    return cacheService.get(
        "product:" + id,
        Product.class,
        () -> productRepository.findById(id)
    );
}
```

---

## Infrastructure — Production Multi-Layer Setup

### Docker Compose for Development

```yaml
version: '3.8'
services:
  app:
    build: .
    environment:
      - REDIS_URL=redis://redis:6379
      - DB_URL=postgresql://postgres:5432/myapp
      - L1_CACHE_SIZE=5000
      - L1_CACHE_TTL=30
      - L2_CACHE_TTL=300
    deploy:
      replicas: 3  # Multiple instances share Redis L2
  
  redis:
    image: redis:7-alpine
    command: >
      redis-server 
      --maxmemory 2gb 
      --maxmemory-policy allkeys-lru
      --save 60 1000
    ports:
      - "6379:6379"
  
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
    command: >
      postgres 
      -c shared_buffers=1GB
      -c effective_cache_size=3GB
```

### AWS Production Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                 AWS MULTI-LAYER CACHING PRODUCTION                     │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌────────────────────────────────────────┐                          │
│  │         CloudFront (CDN)                │                          │
│  │  • 450+ edge locations worldwide       │                          │
│  │  • Cache-Control based caching          │                          │
│  │  • Lambda@Edge for dynamic logic        │                          │
│  └───────────────────┬────────────────────┘                          │
│                       │ Miss                                           │
│                       ▼                                                │
│  ┌────────────────────────────────────────┐                          │
│  │         ALB (Application Load Balancer) │                          │
│  └───────────────────┬────────────────────┘                          │
│                       │                                                │
│         ┌─────────────┼─────────────┐                                │
│         ▼             ▼             ▼                                 │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                       │
│  │  ECS/EKS   │ │  ECS/EKS   │ │  ECS/EKS   │  (App instances)     │
│  │  + Caffeine │ │  + Caffeine │ │  + Caffeine │  L1: per-instance  │
│  └──────┬─────┘ └──────┬─────┘ └──────┬─────┘                       │
│         │               │               │                             │
│         └───────────────┼───────────────┘                             │
│                         ▼                                              │
│  ┌────────────────────────────────────────┐                          │
│  │    ElastiCache Redis Cluster            │  L2: shared             │
│  │    • 3 shards, 6 nodes (3 primary +    │                          │
│  │      3 replica)                         │                          │
│  │    • 150 GB total memory               │                          │
│  │    • Multi-AZ with auto-failover       │                          │
│  └───────────────────┬────────────────────┘                          │
│                       │ Miss (~1-5% of requests)                      │
│                       ▼                                                │
│  ┌────────────────────────────────────────┐                          │
│  │    RDS PostgreSQL / Aurora             │                          │
│  │    • Multi-AZ                          │                          │
│  │    • Read replicas for read scaling    │                          │
│  └────────────────────────────────────────┘                          │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### How Netflix Uses Multi-Layer Caching

Netflix serves 200+ million subscribers with a sophisticated multi-layer architecture:

```
┌──────────────────────────────────────────────────────────────────────┐
│                NETFLIX MULTI-LAYER CACHE ARCHITECTURE                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  USER DEVICE                                                          │
│  └── Client-side cache (app stores viewed content metadata)           │
│                                                                        │
│  OPEN CONNECT CDN (Video Delivery)                                    │
│  └── 17,000+ servers inside ISPs worldwide                           │
│      95% of video bytes served from local ISP cache                   │
│                                                                        │
│  EDGE SERVICES (API Gateway Layer - AWS)                              │
│  └── Zuul edge cache: API response caching                           │
│      Reduces repetitive API calls from millions of devices            │
│                                                                        │
│  MICROSERVICES LAYER                                                  │
│  ├── L1: Guava/Caffeine in each JVM (~10K entries/instance)         │
│  │   Hot data: user preferences, feature flags, A/B test configs     │
│  │                                                                    │
│  ├── L2: EVCache (Memcached-based distributed cache)                 │
│  │   • Dozens of clusters, thousands of nodes                        │
│  │   • Tens of millions of operations/second                        │
│  │   • Cross-region replication for global consistency               │
│  │   • Zone-aware routing (prefer same-AZ cache)                    │
│  │                                                                    │
│  └── L3: Cassandra / MySQL (persistent stores)                       │
│      Only ~1-2% of reads reach here                                  │
│                                                                        │
│  INVALIDATION FLOW:                                                   │
│  Content change → Kafka event → EVCache invalidation                 │
│                               → Local cache invalidation via         │
│                                  discovery service broadcast          │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Key design decisions**:
- EVCache has **zone awareness** — prefers cache in the same availability zone (lower latency)
- **Replication factor** — hot data cached in multiple zones for resilience
- **Fallback chain** — if EVCache is down, service degrades to direct DB access (not crashes)
- **Warming** — new cache nodes get pre-populated from existing nodes before receiving traffic

---

## Advanced Patterns

### Pattern 1: Cache Warming on Deploy

```python
class CacheWarmer:
    """Pre-populate caches after deployment to avoid cold-start misses."""
    
    def __init__(self, cache, db):
        self.cache = cache
        self.db = db
    
    def warm(self):
        """Load most-accessed data into cache proactively."""
        # Warm top 1000 products (most viewed)
        popular_products = self.db.query(
            "SELECT id FROM products ORDER BY view_count DESC LIMIT 1000"
        )
        for product_id in popular_products:
            product = self.db.get_product(product_id)
            self.cache.l2.setex(
                f"product:{product_id}", 
                600, 
                json.dumps(product)
            )
        
        # Warm configuration data
        configs = self.db.query("SELECT * FROM app_config")
        for config in configs:
            self.cache.l2.setex(f"config:{config['key']}", 3600, config['value'])
        
        print(f"Warmed {len(popular_products)} products + {len(configs)} configs")

# Run on application startup
@app.on_event("startup")
async def startup():
    warmer = CacheWarmer(cache, db)
    warmer.warm()
```

### Pattern 2: Adaptive TTL Based on Access Patterns

```python
class AdaptiveTTLCache:
    """Automatically adjusts TTL based on how frequently data is accessed."""
    
    def __init__(self, redis_client, min_ttl=60, max_ttl=3600):
        self.r = redis_client
        self.min_ttl = min_ttl
        self.max_ttl = max_ttl
    
    def get(self, key, loader_fn):
        cached = self.r.get(key)
        if cached:
            # Track access frequency
            self.r.incr(f"freq:{key}")
            return json.loads(cached)
        
        # Miss — load and cache with adaptive TTL
        value = loader_fn()
        ttl = self._calculate_ttl(key)
        self.r.setex(key, ttl, json.dumps(value, default=str))
        self.r.incr(f"freq:{key}")
        return value
    
    def _calculate_ttl(self, key):
        """More frequently accessed = longer TTL (more valuable to cache)."""
        freq = int(self.r.get(f"freq:{key}") or 0)
        
        if freq > 1000:
            return self.max_ttl      # Very hot: cache for 1 hour
        elif freq > 100:
            return self.max_ttl // 2  # Hot: 30 min
        elif freq > 10:
            return self.max_ttl // 4  # Warm: 15 min
        else:
            return self.min_ttl       # Cold: minimum TTL
```

### Pattern 3: Read-Your-Own-Writes Consistency

```python
class ReadYourWritesCache:
    """
    After a user writes data, ensure THEY always see fresh data,
    while other users can see slightly stale cache.
    """
    
    def __init__(self, cache, db):
        self.cache = cache
        self.db = db
        self.write_timestamps = {}  # user_id → {key → timestamp}
    
    def get(self, key, user_id=None, loader_fn=None):
        # Check if this user recently wrote to this key
        if user_id and self._user_wrote_recently(user_id, key):
            # Bypass cache — read directly from DB for this user
            return loader_fn() if loader_fn else self.db.get(key)
        
        # Normal multi-layer cache read
        return self.cache.get(key, loader_fn)
    
    def write(self, key, value, user_id):
        self.db.write(key, value)
        self.cache.invalidate(key)
        
        # Track that this user just wrote (expires after 5 seconds)
        if user_id not in self.write_timestamps:
            self.write_timestamps[user_id] = {}
        self.write_timestamps[user_id][key] = time.time()
    
    def _user_wrote_recently(self, user_id, key):
        if user_id in self.write_timestamps:
            ts = self.write_timestamps[user_id].get(key, 0)
            if time.time() - ts < 5:  # Within 5 seconds of their write
                return True
        return False
```

---

## Monitoring Multi-Layer Caches

```
Key Metrics to Track Per Layer:
┌─────────────────────────────────────────────────────────────┐
│  Metric              │ L1 (Local) │ L2 (Redis) │ Database   │
├──────────────────────┼────────────┼────────────┼────────────┤
│ Hit Rate             │   >70%     │   >90%     │    N/A     │
│ Latency (p99)       │   <1ms     │   <5ms     │   <50ms    │
│ Eviction Rate       │   Monitor  │   Low      │    N/A     │
│ Memory Usage        │   <25% heap│   <80% max │   Buffer   │
│ Requests/sec        │   All      │   L1 miss  │   L2 miss  │
└──────────────────────┴────────────┴────────────┴────────────┘

Alert Thresholds:
  • L2 hit rate drops below 85% → investigate
  • L2 latency p99 > 10ms → check Redis cluster health
  • DB queries/sec spikes → cache layer may be failing
  • Memory usage > 90% → add capacity or review TTLs
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| L1 TTL > L2 TTL | L1 serves stale data AFTER L2 has been invalidated | L1 TTL should always be < L2 TTL |
| No invalidation broadcast for L1 | Instance A invalidates, instance B still serves stale | Use pub/sub to broadcast L1 invalidation |
| Over-caching in L1 | Large local cache → GC pressure → latency spikes (Java) | Keep L1 small (thousands, not millions of entries) |
| Same TTL across layers | Synchronized expiration → stampede | Use decreasing TTLs: DB→L2(5min)→L1(30s)→CDN(1min)→Browser(30s) |
| No fallback on cache layer failure | If Redis dies, all requests pound DB | Circuit breaker: degrade gracefully |
| Not warming after deploy | Fresh instances have cold cache → temporary slow | Pre-warm hot data on startup |

### The TTL Ordering Rule

```
CORRECT TTL ordering (inner layers shorter):
  Browser: 30s → CDN: 60s → L1 Local: 30s → L2 Redis: 300s → DB: source

WHY?
  • L1 expires faster → checks L2 frequently → picks up invalidations
  • If L2 is invalidated but L1 still has old data, it's only stale for 30s max
  • Browser TTL short → users get fresh data on page refresh

❌ WRONG: L1 TTL = 10 min, L2 TTL = 5 min
  L2 expires and gets fresh data. But L1 still serves 10-min-old data!
```

---

## When to Use / When NOT to Use

### ✅ Use Multi-Layer Caching When:
- Traffic is **massive** (thousands+ requests/second)
- You have **multiple app instances** behind a load balancer
- Data has **varying hotness** (some items ultra-popular, others rarely accessed)
- You need **both speed** (local) and **consistency** (shared)
- Read-to-write ratio is **very high** (50:1 or more)

### ❌ Don't Use Multi-Layer Caching When:
- Single-instance deployment (L1 + DB is enough)
- Traffic is low (<100 req/s) — complexity not worth it
- Data changes **every request** — caching provides no benefit
- Team doesn't have **operational capacity** to monitor multiple cache layers
- Storage/memory budget is extremely limited

---

## Key Takeaways

1. **Multi-layer caching** stacks caches at Browser → CDN → Local (L1) → Distributed (L2) → Database, with each layer shielding the next.
2. **Traffic reduction** is multiplicative: 1M requests might become 900 DB queries (99.91% absorbed by caches).
3. **L1 (local) caches should be small and short-TTL** — they're for ultra-hot data that can tolerate per-instance inconsistency.
4. **L2 (distributed) caches are the workhorse** — shared, consistent, large capacity.
5. **Invalidation must cascade** outward through all layers — use pub/sub for L1 cross-instance invalidation.
6. **TTL ordering matters**: inner layers (L1) should have shorter TTLs than outer layers (L2).
7. **Monitor each layer independently** — a failing layer can cascade into catastrophic database overload.

---

## What's Next?

Congratulations! You've completed **Part 10: Caching**. You now understand everything from basic cache concepts to planet-scale multi-layer architectures.

Next up is **Part 11: Message Queues & Event Streaming** — how systems communicate asynchronously using RabbitMQ, Kafka, and event-driven patterns to handle massive workloads without blocking.

→ [../11-message-queues/01-sync-vs-async.md](../11-message-queues/01-sync-vs-async.md)
