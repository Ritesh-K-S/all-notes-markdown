# Stage 4: Caching + CDN — 10K to 100K Users

> **What you'll learn**: How caching (storing frequently-accessed data in fast memory) and CDNs (serving static content from locations near users) can reduce database load by 80-90% and make your app feel instant.

---

## Real-Life Analogy

Your chai shop is now serving 500 customers/hour. You notice a pattern: **80% of customers order the same 3 drinks** — masala chai, ginger chai, and plain chai.

Instead of making each cup from scratch every time (boiling water, adding spices, brewing...), you now:
1. **Keep large pre-made batches** of the top 3 drinks in heated dispensers (that's **caching**)
2. **Open small satellite counters** at nearby street corners so customers don't have to walk all the way to the main shop (that's a **CDN**)

Result: 80% of customers get served in 5 seconds instead of 5 minutes, and the main kitchen is now free to handle complex custom orders.

---

## Core Concept Explained Step-by-Step

### The Architecture at Stage 4

```
                         Internet Users
                    (Delhi, Mumbai, Bangalore, US, EU)
                              │
               ┌──────────────┼──────────────────┐
               │              │                  │
               ▼              ▼                  ▼
        ┌───────────┐  ┌───────────┐      ┌───────────┐
        │ CDN Edge  │  │ CDN Edge  │      │ CDN Edge  │
        │ Mumbai    │  │ Delhi     │      │ US-East   │
        │           │  │           │      │           │
        │ Static:   │  │ Static:   │      │ Static:   │
        │ JS,CSS,   │  │ JS,CSS,   │      │ JS,CSS,   │
        │ Images    │  │ Images    │      │ Images    │
        └─────┬─────┘  └─────┬─────┘      └─────┬─────┘
              │               │                  │
              └───────────────┼──────────────────┘
                              │ (only API calls reach origin)
                              ▼
                     ┌─────────────────┐
                     │  LOAD BALANCER  │
                     └────────┬────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │  App Server  │ │  App Server  │ │  App Server  │
     │  Instance 1  │ │  Instance 2  │ │  Instance 3  │
     └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
            │                │                │
            └───────┬────────┘                │
                    ▼                         │
           ┌──────────────┐                   │
           │    REDIS     │◀──────────────────┘
           │   (Cache)    │
           │              │
           │ Hot data in  │
           │ memory       │
           └──────┬───────┘
                  │ (cache miss)
                  ▼
           ┌──────────────┐
           │  PostgreSQL  │
           │  (Database)  │
           └──────────────┘
```

### Two Types of Caching

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  TYPE 1: APPLICATION CACHE (Redis/Memcached)                     │
│  ─────────────────────────────────────────                       │
│  What: Store database query results in RAM                       │
│  Where: Redis server (separate from app and DB)                  │
│  Speed: ~0.5ms (vs ~5-50ms for database query)                   │
│  Example: User profile, product details, feed posts              │
│                                                                  │
│  ┌─────────┐     ┌─────────┐     ┌──────────┐                   │
│  │   App   │────▶│  Redis  │     │ Database │                   │
│  │         │◀────│ (cache) │     │          │                   │
│  │         │     └─────────┘     │          │                   │
│  │         │──── cache miss ────▶│          │                   │
│  │         │◀─── fetch + store ──│          │                   │
│  └─────────┘                     └──────────┘                   │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TYPE 2: CDN (Content Delivery Network)                          │
│  ──────────────────────────────────────                          │
│  What: Serve static files from servers NEAR the user             │
│  Where: Edge servers in 100+ cities worldwide                    │
│  Speed: ~20ms (vs ~200ms if served from origin in US)            │
│  Example: JavaScript, CSS, images, videos, fonts                 │
│                                                                  │
│  User in Mumbai ──▶ CDN Edge Mumbai ──▶ Cached! (20ms)           │
│                     (not cached?) ──▶ Origin Server ──▶ Cache it │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### The Cache-Aside Pattern (Most Common)

```
Request: "Get user profile for user_id=42"

Step 1: Check cache first
┌─────┐          ┌───────┐
│ App │──GET────▶│ Redis │
│     │          │       │
│     │◀─────────│ Found?│
└─────┘          └───────┘
   │
   │ YES (cache hit) → Return cached data (0.5ms) ✓
   │
   │ NO (cache miss) ↓
   │
Step 2: Fetch from database
┌─────┐          ┌──────────┐
│ App │──SQL────▶│ Database │
│     │◀─────────│          │ (5-50ms)
└─────┘          └──────────┘
   │
Step 3: Store in cache for next time
┌─────┐          ┌───────┐
│ App │──SET────▶│ Redis │  (with TTL = 5 minutes)
└─────┘          └───────┘
   │
Step 4: Return data to user
```

### Cache Hit Ratio: The Magic Number

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  Cache Hit Ratio = Requests served from cache / Total requests │
│                                                                │
│  Scenario: 100 requests                                        │
│                                                                │
│  ████████████████████████████████████████████░░░░░ 90% hit     │
│  ▲ 90 served from cache (0.5ms each)                           │
│                          ▲ 10 go to database (20ms each)       │
│                                                                │
│  Average response time:                                        │
│  Without cache: 100 × 20ms = 2000ms total → 20ms/request      │
│  With 90% cache: (90×0.5) + (10×20) = 245ms → 2.45ms/request  │
│                                                                │
│  Result: 8x FASTER! Database handles 10x FEWER queries!        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### CDN: How Static Files Are Served Globally

```
WITHOUT CDN:
User in India ───────── 200ms ────────── Server in US
                    (crosses ocean)

WITH CDN:
User in India ── 20ms ── CDN Edge Mumbai (has cached copy)
                         (if not cached: fetches from US, caches for next user)

CDN Edge Locations (Cloudflare example):
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  🌐 300+ cities worldwide                                    │
│                                                              │
│  Mumbai ● ── Delhi ● ── Chennai ● ── Bangalore ●            │
│  Singapore ● ── Tokyo ● ── Sydney ●                          │
│  London ● ── Frankfurt ● ── Paris ●                          │
│  New York ● ── San Francisco ● ── São Paulo ●               │
│                                                              │
│  Each ● has a server with YOUR static files cached           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Redis Cache Internals

Redis stores data in **RAM** using efficient data structures:

```
Redis Memory Layout:
┌──────────────────────────────────────────────────────┐
│  Key                     │ Value           │ TTL     │
├──────────────────────────┼─────────────────┼─────────┤
│  user:42:profile         │ {json blob}     │ 300s    │
│  product:123:details     │ {json blob}     │ 600s    │
│  feed:user:42:page:1     │ [post1,post2..] │ 60s     │
│  api:rate:192.168.1.1    │ 47              │ 60s     │
│  session:abc123def       │ {user data}     │ 3600s   │
└──────────────────────────┴─────────────────┴─────────┘

Speed comparison:
┌──────────────┬──────────────┬──────────────────────────────┐
│ Operation    │ Redis (RAM)  │ PostgreSQL (Disk/Buffer)      │
├──────────────┼──────────────┼──────────────────────────────┤
│ Simple GET   │ 0.1 - 0.5ms  │ 1 - 5ms                      │
│ Complex query│ N/A          │ 10 - 500ms                   │
│ Throughput   │ 100K ops/sec │ 5K-20K queries/sec           │
└──────────────┴──────────────┴──────────────────────────────┘
```

### Cache Invalidation Strategies

```
┌──────────────────────────────────────────────────────────────────────┐
│ STRATEGY        │ HOW                         │ WHEN TO USE          │
├─────────────────┼─────────────────────────────┼──────────────────────┤
│ TTL (Time to    │ Cache expires after N        │ Data that's OK to    │
│ Live)           │ seconds automatically        │ be slightly stale    │
│                 │                             │ (user feeds, stats)  │
├─────────────────┼─────────────────────────────┼──────────────────────┤
│ Write-Through   │ Update cache whenever DB    │ Data that must be    │
│                 │ is updated                  │ always fresh         │
│                 │                             │ (user profile)       │
├─────────────────┼─────────────────────────────┼──────────────────────┤
│ Cache-Aside     │ App checks cache first,     │ Read-heavy data      │
│ (Lazy Loading)  │ fills on miss               │ (most common)        │
├─────────────────┼─────────────────────────────┼──────────────────────┤
│ Write-Behind    │ Write to cache first, async │ Write-heavy + need   │
│                 │ write to DB later           │ fast writes          │
└─────────────────┴─────────────────────────────┴──────────────────────┘
```

### CDN Internals: Cache Headers

```
How CDN knows what to cache and for how long:

Browser ──▶ CDN Edge ──▶ Origin Server
                              │
                              │ Response Headers:
                              │ Cache-Control: public, max-age=31536000
                              │ (cache for 1 year!)
                              │
                    CDN stores this file locally
                              │
Next request for same file:
Browser ──▶ CDN Edge (HIT! Serves from local cache)
            Never reaches origin server!

Cache-Control values:
┌────────────────────────────────────────────────────────────────┐
│ Header                          │ Meaning                      │
├─────────────────────────────────┼──────────────────────────────┤
│ Cache-Control: public, max-age= │ CDN can cache for N seconds  │
│   31536000                      │ (1 year for static assets)   │
│ Cache-Control: private, no-cache│ DON'T cache (user-specific)  │
│ Cache-Control: s-maxage=60      │ CDN caches 60s, browser may  │
│                                 │ have different TTL           │
└─────────────────────────────────┴──────────────────────────────┘
```

---

## Code Examples

### Python: Redis Caching with Cache-Aside Pattern

```python
# cache.py — Redis caching layer
import redis
import json
from functools import wraps

# Connect to Redis server
cache = redis.Redis(host="10.0.3.1", port=6379, decode_responses=True)

def cached(key_pattern, ttl=300):
    """Decorator: cache function results in Redis"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Build cache key from pattern and arguments
            cache_key = key_pattern.format(*args, **kwargs)

            # Step 1: Try cache first
            cached_data = cache.get(cache_key)
            if cached_data:
                return json.loads(cached_data)  # Cache HIT! (< 1ms)

            # Step 2: Cache miss — call actual function (hits DB)
            result = func(*args, **kwargs)

            # Step 3: Store in cache for next time
            cache.setex(cache_key, ttl, json.dumps(result))

            return result
        return wrapper
    return decorator

# Usage in your API:
@app.route("/api/users/<int:user_id>")
@cached("user:{0}:profile", ttl=300)  # Cache for 5 minutes
def get_user(user_id):
    """This DB query only runs on cache miss"""
    conn = db_pool.getconn()
    cur = conn.cursor()
    cur.execute("SELECT id, name, email, bio FROM users WHERE id = %s", (user_id,))
    row = cur.fetchone()
    db_pool.putconn(conn)
    return {"id": row[0], "name": row[1], "email": row[2], "bio": row[3]}

@app.route("/api/users/<int:user_id>", methods=["PUT"])
def update_user(user_id):
    """When data changes, invalidate the cache"""
    data = request.get_json()
    # Update database
    conn = db_pool.getconn()
    cur = conn.cursor()
    cur.execute("UPDATE users SET name=%s WHERE id=%s", (data["name"], user_id))
    conn.commit()
    db_pool.putconn(conn)

    # INVALIDATE cache (force fresh data on next read)
    cache.delete(f"user:{user_id}:profile")

    return jsonify({"message": "Updated"})
```

### Java: Spring Boot with Redis Cache

```java
// CacheConfig.java — Enable Redis caching
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5))     // Default TTL: 5 minutes
            .serializeValuesWith(
                SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())
            );

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("users",
                config.entryTtl(Duration.ofMinutes(10)))  // Users: 10 min
            .withCacheConfiguration("products",
                config.entryTtl(Duration.ofHours(1)))     // Products: 1 hour
            .build();
    }
}

// UserService.java — Automatic caching with annotations
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Cacheable(value = "users", key = "#userId")  // Auto-cache!
    public User getUser(Long userId) {
        // This method body ONLY runs on cache miss
        // On cache hit, Spring returns cached result directly
        return userRepository.findById(userId).orElseThrow();
    }

    @CacheEvict(value = "users", key = "#userId")  // Invalidate on update
    public User updateUser(Long userId, UserUpdateRequest request) {
        User user = userRepository.findById(userId).orElseThrow();
        user.setName(request.getName());
        return userRepository.save(user);
    }

    @CacheEvict(value = "users", allEntries = true)  // Clear all user cache
    public void clearUserCache() {
        // Called during bulk updates or data migrations
    }
}
```

---

## Infrastructure Example

### CDN Configuration (Cloudflare)

```nginx
# Nginx origin server — Set proper cache headers for CDN

server {
    listen 80;
    server_name myapp.com;

    # Static assets: cache aggressively (1 year)
    # Use versioned filenames: app.abc123.js
    location /static/ {
        root /var/www/myapp;
        expires 1y;
        add_header Cache-Control "public, immutable";
        add_header CDN-Cache-Control "max-age=31536000";
    }

    # Images: cache for 30 days
    location /images/ {
        root /var/www/myapp;
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
    }

    # API responses: no CDN caching (dynamic, user-specific)
    location /api/ {
        proxy_pass http://app_servers;
        add_header Cache-Control "private, no-store";
    }

    # HTML pages: short cache (5 minutes) for CDN
    location / {
        proxy_pass http://app_servers;
        add_header Cache-Control "public, s-maxage=300, max-age=0";
    }
}
```

### Redis Cluster Setup (Docker Compose)

```yaml
# docker-compose.yml — Adding Redis cache layer
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: >
      redis-server
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru
      --save 60 1000
    volumes:
      - redis-data:/data
    deploy:
      resources:
        limits:
          memory: 2.5G

  redis-insight:
    image: redislabs/redisinsight:latest
    ports:
      - "8001:8001"  # Redis monitoring UI

volumes:
  redis-data:
```

### Redis Eviction Policies

```
When Redis runs out of memory, which keys get deleted?

┌─────────────────────┬──────────────────────────────────────────┐
│ Policy              │ Behavior                                 │
├─────────────────────┼──────────────────────────────────────────┤
│ allkeys-lru         │ Remove Least Recently Used key           │
│ (RECOMMENDED)       │ (best for caching)                       │
├─────────────────────┼──────────────────────────────────────────┤
│ volatile-lru        │ Remove LRU key that has a TTL set        │
├─────────────────────┼──────────────────────────────────────────┤
│ allkeys-random      │ Remove a random key                      │
├─────────────────────┼──────────────────────────────────────────┤
│ noeviction          │ Return error when memory is full         │
│                     │ (DON'T use for caching!)                 │
└─────────────────────┴──────────────────────────────────────────┘
```

---

## Real-World Example

### Instagram's Caching Strategy

Instagram serves **2 billion+ daily active users** with aggressive caching:
- **Memcached** for user profiles, follower counts, feed data
- **CDN (Facebook's own)** for images and videos
- Cache hit ratio: **~99%** for popular content
- Without caching, they'd need 100x more database servers

### Netflix's CDN (Open Connect)

Netflix built their OWN CDN called **Open Connect**:
- Physical servers placed inside ISPs worldwide
- When you watch a movie, it streams from a server inside your ISP's data center
- This means Netflix video traffic barely touches the public internet
- Result: 4K streaming without buffering, even during peak hours

### Amazon's Caching

- **ElastiCache (Redis/Memcached)** for session data, product catalogs
- **CloudFront CDN** for static assets and even dynamic API responses
- During Prime Day: Cache hit ratios of 95%+ keep the database alive
- Product pages are cached and only invalidated when prices change

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Caching too aggressively** | Users see stale data (old prices, wrong info) | Set appropriate TTLs; invalidate on writes |
| **No cache invalidation** | Edited data never updates for users | Delete/update cache when data changes |
| **Cache stampede** | 1000 requests all hit DB when cache expires simultaneously | Use lock/mutex or staggered TTLs |
| **Caching user-specific data in CDN** | User A sees User B's private data! | Only CDN-cache public data; use `Cache-Control: private` for user data |
| **Not monitoring cache hit ratio** | You think caching helps but hit ratio is 20% | Track metrics; aim for > 80% hit ratio |
| **Caching errors** | An error response gets cached and served to everyone | Don't cache 4xx/5xx responses |
| **Forgetting cache warming** | After deploy/restart, everything is a cache miss | Pre-warm popular keys on startup |

### The Cache Stampede Problem (and Solution)

```
PROBLEM: Cache expires → 100 requests hit DB simultaneously

Time T=0:  Cache has "product:123" (valid)
Time T=300: Cache EXPIRES
Time T=300.001: 100 requests arrive for product:123
               ALL 100 go to database! 💥 (thundering herd)

SOLUTION: Mutex/Lock pattern

Request 1: Cache miss → Acquire lock → Query DB → Fill cache → Release lock
Request 2-100: Cache miss → See lock exists → Wait 50ms → Retry → Cache HIT ✓

Only ONE request hits the database!
```

---

## When to Use / When NOT to Use

### ✅ Add Caching When:
- Database queries are repeated (same data requested many times)
- Read-to-write ratio is high (80%+ reads)
- Response time needs to be < 100ms
- Database CPU is high even with proper indexing
- You're serving 10K+ users and growing

### ✅ Add CDN When:
- You have static assets (JS, CSS, images, videos)
- Users are geographically distributed
- Page load time matters for SEO/UX
- You want to reduce bandwidth costs from your origin server

### ❌ Don't Cache:
- Data that changes every second (real-time stock prices — use WebSockets instead)
- User-specific data that's rarely re-read
- Write-heavy workloads where data is always changing
- When you can't handle stale data even for a few seconds

---

## Key Takeaways

1. **Caching reduces database load by 80-90%** — the single most impactful performance optimization
2. **Cache-Aside is the most common pattern** — check cache first, fill on miss, invalidate on write
3. **CDN makes static content feel instant** — files served from a server 20ms away vs 200ms
4. **Cache invalidation is the hardest problem** — "There are only two hard things in CS: cache invalidation and naming things"
5. **Always set a TTL** — no TTL = stale data forever if invalidation fails
6. **Monitor your cache hit ratio** — below 80% means your caching strategy needs work
7. **Redis is the industry standard** for application caching — fast, versatile, battle-tested

---

## What's Next?

Even with caching, your single database is handling all writes and the 10-20% of reads that miss the cache. As you grow toward 100K-1M users, the database itself becomes the bottleneck again. The solution? **Database Replication with Read Replicas** — Stage 5.

Next: [05-stage5-db-replication.md](./05-stage5-db-replication.md)
