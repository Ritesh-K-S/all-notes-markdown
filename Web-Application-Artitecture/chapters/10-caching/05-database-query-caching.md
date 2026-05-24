# Database Query Caching

> **What you'll learn**: How to cache database query results to eliminate redundant queries, dramatically reduce database load, and serve repeated reads in microseconds instead of milliseconds.

---

## Real-Life Analogy

Imagine you're a librarian. Every day, 50 students ask "Do you have the book 'Atomic Habits'?" You walk to the shelf, check, come back and say "Yes, aisle 3, shelf 2."

After the 10th time, you'd be crazy not to just **write the answer on a sticky note** at your desk. Now when someone asks, you glance at the sticky note — instant answer, no walking to the shelf.

**Database query caching** is that sticky note. When the same query is asked repeatedly, you remember the result instead of asking the database again.

---

## Core Concept Explained Step-by-Step

### Step 1: The Problem — Repetitive Expensive Queries

In a typical web app, the same queries run **thousands of times per second**:

```
Request 1: "Show me product #5678"  → SELECT * FROM products WHERE id = 5678
Request 2: "Show me product #5678"  → SELECT * FROM products WHERE id = 5678
Request 3: "Show me product #5678"  → SELECT * FROM products WHERE id = 5678
...
Request 10,000: same query!

Each takes ~5ms × 10,000 = 50 seconds of DB time for IDENTICAL results
```

### Step 2: Cache the Query Results

```
┌────────────────┐         ┌──────────────┐         ┌──────────────┐
│  Application   │         │    Cache     │         │   Database   │
│                │──GET───▶│  (Redis)     │         │              │
│                │         │              │         │              │
│                │◀─HIT────│  Found!      │         │              │
│                │         └──────────────┘         │              │
│                │                                   │              │
│                │──GET───▶ MISS ────────GET────────▶│              │
│                │                                   │  Execute     │
│                │◀─────────────────RESULT──────────◀│  Query       │
│                │                                   │              │
│                │──SET───▶ Store result             │              │
│                │         with TTL                  │              │
└────────────────┘         └──────────────┘         └──────────────┘
```

### Step 3: Where Query Caching Can Happen

```
┌──────────────────────────────────────────────────────────────────┐
│               DATABASE QUERY CACHING LAYERS                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Layer 1: APPLICATION CODE                                        │
│  ─────────────────────────                                        │
│  Your code checks Redis before running query                      │
│  Most flexible, you control everything                            │
│                                                                    │
│  Layer 2: ORM / FRAMEWORK LEVEL                                   │
│  ─────────────────────────────                                    │
│  Hibernate L2 cache, Django cache framework                       │
│  Automatic, works with your models                                │
│                                                                    │
│  Layer 3: DATABASE PROXY                                          │
│  ────────────────────────                                         │
│  ProxySQL, PgBouncer with query caching                           │
│  Transparent to application code                                  │
│                                                                    │
│  Layer 4: DATABASE INTERNAL                                       │
│  ─────────────────────────                                        │
│  MySQL Query Cache (deprecated), PostgreSQL prepared statements   │
│  Buffer pool / shared buffers (page caching)                      │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Creating a Cache Key from a Query

The cache key must **uniquely identify** the query AND its parameters:

```
Query: SELECT * FROM products WHERE category = 'electronics' AND price < 500
Params: ['electronics', 500]

Cache Key Options:
  1. Hash-based: MD5("SELECT...electronics...500") → "query:a7f3b2c1..."
  2. Structured: "products:category=electronics:price_lt=500"
  3. Named: "product_list:electronics:under_500"

Best practice: Use structured, human-readable keys
```

### Query Cache Invalidation Flow

```
Write Operation (UPDATE/INSERT/DELETE)
        │
        ▼
┌────────────────────────────┐
│ Which cache keys are        │
│ affected by this write?     │
├────────────────────────────┤
│                              │
│ Strategy A: Tag-based        │
│ ─────────────────           │
│ Keys tagged with "products"  │
│ → Invalidate all of them     │
│                              │
│ Strategy B: Pattern-based    │
│ ─────────────────────       │
│ Delete keys matching          │
│ "products:*"                  │
│                              │
│ Strategy C: Event-driven     │
│ ─────────────────────       │
│ Product update event →       │
│ Invalidation handler         │
└────────────────────────────┘
```

### Database Internal Caching — Buffer Pool

Even without application-level caching, databases have their own internal caches:

```
┌─────────────────────────────────────────────────────┐
│             POSTGRESQL SHARED BUFFERS                 │
├─────────────────────────────────────────────────────┤
│                                                       │
│  Query: SELECT * FROM users WHERE id = 123           │
│                                                       │
│  Step 1: Check if page is in shared_buffers (RAM)    │
│          ├── HIT: Read from RAM (~0.1ms)             │
│          └── MISS: Read from disk (~2-10ms)          │
│                   then load into shared_buffers      │
│                                                       │
│  ┌─────────────────────────────────────────────┐    │
│  │  Shared Buffers (typically 25% of RAM)       │    │
│  │                                               │    │
│  │  Page 1001 [users table, rows 100-120]       │    │
│  │  Page 1002 [users table, rows 121-140]       │    │
│  │  Page 5003 [orders idx, B-tree node]         │    │
│  │  ...                                          │    │
│  └─────────────────────────────────────────────┘    │
│                                                       │
│  This is PAGE-level caching, not query-result caching │
└─────────────────────────────────────────────────────┘
```

> **Important distinction**: Database buffer pools cache **data pages**, not query results. If you run a complex JOIN, the DB still re-executes the query logic even if pages are cached. Application-level query caching skips the query entirely.

---

## Code Examples

### Python — Query Caching with Redis

```python
import redis
import json
import hashlib
from functools import wraps

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def cache_query(ttl_seconds=300, prefix="query"):
    """Decorator to cache database query results in Redis."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Build cache key from function name + arguments
            key_data = f"{func.__name__}:{args}:{sorted(kwargs.items())}"
            cache_key = f"{prefix}:{hashlib.md5(key_data.encode()).hexdigest()}"
            
            # Check cache
            cached = r.get(cache_key)
            if cached:
                return json.loads(cached)  # HIT
            
            # Miss — execute query
            result = func(*args, **kwargs)
            
            # Store in cache
            r.setex(cache_key, ttl_seconds, json.dumps(result, default=str))
            
            return result
        return wrapper
    return decorator

# Usage
@cache_query(ttl_seconds=600)
def get_products_by_category(category, max_price=None):
    """Cached for 10 minutes."""
    query = "SELECT * FROM products WHERE category = %s"
    params = [category]
    if max_price:
        query += " AND price <= %s"
        params.append(max_price)
    return db.execute(query, params).fetchall()

# Invalidation: delete related cache entries when product changes
def update_product(product_id, data):
    db.execute("UPDATE products SET ... WHERE id = %s", product_id)
    # Invalidate all product query caches (tag-based approach)
    for key in r.scan_iter("query:*"):
        r.delete(key)  # Simple but aggressive
```

### Python — Django ORM with Cache Framework

```python
from django.core.cache import cache
from .models import Product

def get_trending_products():
    """Cache expensive query with Django cache framework."""
    cache_key = "trending_products_top_20"
    
    # Try cache first (uses Redis backend)
    products = cache.get(cache_key)
    if products is not None:
        return products
    
    # Expensive query with aggregation
    products = list(
        Product.objects
        .filter(is_active=True)
        .annotate(score=F('views') * 0.3 + F('purchases') * 0.7)
        .order_by('-score')[:20]
        .values('id', 'name', 'price', 'image_url')
    )
    
    # Cache for 5 minutes
    cache.set(cache_key, products, timeout=300)
    return products

# Invalidate when a product is updated
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=Product)
def invalidate_product_cache(sender, instance, **kwargs):
    cache.delete("trending_products_top_20")
    cache.delete(f"product:{instance.id}")
```

### Java — Spring with Hibernate Second-Level Cache

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CacheEvict;

@Service
public class ProductService {
    
    @Autowired
    private ProductRepository repo;

    /**
     * Result is cached in Redis. Same parameters = same cache key.
     * Only runs the DB query on cache miss.
     */
    @Cacheable(value = "productsByCategory", 
               key = "#category + ':' + #maxPrice",
               unless = "#result.isEmpty()")
    public List<Product> getByCategory(String category, Double maxPrice) {
        return repo.findByCategoryAndPriceLessThan(category, maxPrice);
    }

    /**
     * Evicts all entries in 'productsByCategory' cache when any product changes.
     */
    @CacheEvict(value = "productsByCategory", allEntries = true)
    public Product updateProduct(Long id, ProductUpdateDto dto) {
        Product product = repo.findById(id).orElseThrow();
        product.setName(dto.getName());
        product.setPrice(dto.getPrice());
        return repo.save(product);
    }
}
```

### Java — Manual Query Caching with Redis

```java
public class OrderRepository {
    
    private final JedisPool redis;
    private final JdbcTemplate jdbc;
    private final ObjectMapper mapper;

    public List<Order> getRecentOrders(String userId, int limit) {
        String cacheKey = String.format("orders:%s:recent:%d", userId, limit);
        
        // Check cache
        try (Jedis jedis = redis.getResource()) {
            String cached = jedis.get(cacheKey);
            if (cached != null) {
                return mapper.readValue(cached, new TypeReference<>() {});
            }
        }
        
        // Cache miss — run query
        List<Order> orders = jdbc.query(
            "SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT ?",
            new Object[]{userId, limit},
            orderRowMapper
        );
        
        // Cache result for 2 minutes
        try (Jedis jedis = redis.getResource()) {
            jedis.setex(cacheKey, 120, mapper.writeValueAsString(orders));
        }
        
        return orders;
    }
    
    // Invalidate user's order cache when new order is placed
    public void invalidateOrderCache(String userId) {
        try (Jedis jedis = redis.getResource()) {
            Set<String> keys = jedis.keys("orders:" + userId + ":*");
            if (!keys.isEmpty()) {
                jedis.del(keys.toArray(new String[0]));
            }
        }
    }
}
```

---

## Infrastructure Example — ProxySQL Query Caching

ProxySQL sits between your app and MySQL, transparently caching queries:

```
┌─────────────┐      ┌──────────────┐      ┌──────────────┐
│ Application │─────▶│  ProxySQL    │─────▶│    MySQL     │
│             │      │              │      │              │
│             │      │ Query Cache: │      │              │
│             │      │ SELECT * ... │      │              │
│             │      │ → cached!    │      │              │
└─────────────┘      └──────────────┘      └──────────────┘
```

```sql
-- ProxySQL configuration for query caching
-- Cache SELECT queries matching this pattern for 5 seconds

INSERT INTO mysql_query_rules (
    rule_id, match_pattern, cache_ttl, apply
) VALUES (
    1, 
    '^SELECT .* FROM products WHERE category', 
    5000,    -- TTL in milliseconds
    1
);

-- Reload rules
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

---

## Real-World Example

### How Stack Overflow Caches Database Queries

Stack Overflow serves **1.3 billion page views/month** with remarkably few servers, largely thanks to aggressive query caching:

```
┌──────────────────────────────────────────────────────────────────┐
│            STACK OVERFLOW QUERY CACHING                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Request: GET /questions/12345                                    │
│       │                                                            │
│       ▼                                                            │
│  ┌────────────────────────────┐                                   │
│  │  L1: In-Process Cache      │  (per-server, microseconds)      │
│  │  (.NET MemoryCache)        │                                   │
│  │  "Hot questions" list      │                                   │
│  └────────────┬───────────────┘                                   │
│          MISS │                                                    │
│               ▼                                                    │
│  ┌────────────────────────────┐                                   │
│  │  L2: Redis Cache           │  (~0.5ms, shared across web tier)│
│  │  Question + answers +      │                                   │
│  │  vote counts + user info   │                                   │
│  └────────────┬───────────────┘                                   │
│          MISS │                                                    │
│               ▼                                                    │
│  ┌────────────────────────────┐                                   │
│  │  SQL Server                │  (~5-20ms)                        │
│  │  (only ~10% of reads      │                                   │
│  │   reach here)              │                                   │
│  └────────────────────────────┘                                   │
│                                                                    │
│  Cache Invalidation: When a vote/edit happens →                   │
│  Pub/Sub message → all servers invalidate that question's cache   │
└──────────────────────────────────────────────────────────────────┘
```

**Result**: 9 web servers handle the entire site. ~90% of reads never touch SQL Server.

---

## Advanced Pattern: Read-Through Cache with Tags

```python
class TaggedQueryCache:
    """Cache with tag-based invalidation for related queries."""
    
    def __init__(self, redis_client):
        self.r = redis_client
    
    def get_or_load(self, cache_key, loader_fn, ttl=300, tags=None):
        """Get from cache or load and cache with tags."""
        # Check cache
        cached = self.r.get(cache_key)
        if cached:
            return json.loads(cached)
        
        # Load from source
        result = loader_fn()
        
        # Store result
        self.r.setex(cache_key, ttl, json.dumps(result, default=str))
        
        # Associate key with tags for later invalidation
        if tags:
            for tag in tags:
                self.r.sadd(f"tag:{tag}", cache_key)
                self.r.expire(f"tag:{tag}", ttl + 60)
        
        return result
    
    def invalidate_by_tag(self, tag):
        """Invalidate ALL cache entries with this tag."""
        keys = self.r.smembers(f"tag:{tag}")
        if keys:
            self.r.delete(*keys)
        self.r.delete(f"tag:{tag}")

# Usage
cache = TaggedQueryCache(redis_client)

# Cache product queries tagged with 'products' and 'category:electronics'
products = cache.get_or_load(
    "products:electronics:under500",
    lambda: db.query("SELECT * FROM products WHERE ..."),
    ttl=600,
    tags=["products", "category:electronics"]
)

# When any product in 'electronics' changes — invalidate all related caches
cache.invalidate_by_tag("category:electronics")
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Caching queries with `NOW()` or random | Every call is unique → 0% hit rate | Remove time-dependent parts from cache key |
| Cache key doesn't include ALL parameters | Wrong results returned | Include every param that affects output |
| Caching empty results without thought | If temporarily empty, stays "empty" in cache | Cache nulls with very short TTL, or don't cache |
| No invalidation strategy | Stale data served for hours | Plan invalidation before implementing caching |
| Using `KEYS *` pattern in Redis for invalidation | Blocks Redis (O(n) scan) | Use `SCAN`, tags, or pub/sub |
| Caching too-large result sets | Bloats cache, slow serialization | Paginate and cache per page |
| Not caching at the right granularity | Cache whole page vs individual query | Cache the smallest reusable unit |

---

## When to Use / When NOT to Use

### ✅ Use Database Query Caching When:
- Same query runs **many times** with identical parameters
- Query is **expensive** (JOINs, aggregations, full-text search)
- Data doesn't change frequently (product catalog, blog posts)
- Read-to-write ratio is **high** (10:1 or more)
- Database is becoming a bottleneck

### ❌ Don't Use Query Caching When:
- Data changes **every request** (real-time counters updated per click)
- Query results are **unique per user** with low reuse
- Consistency is critical and **TTL-based staleness** is unacceptable
- Queries are already fast (<1ms) — caching adds overhead
- MySQL's built-in query cache (deprecated in 8.0 for good reason — global mutex lock destroyed performance)

---

## Key Takeaways

1. **Query caching** stores the result of database queries so identical requests skip the database entirely.
2. Cache keys must **uniquely identify** the query + all parameters.
3. **Invalidation** is the hard part — use tag-based, event-driven, or short TTL strategies.
4. Database internal caches (buffer pools) cache **pages**, not query results — they still re-execute query logic.
5. Combine with **read-through pattern** for clean code — cache transparently loads on miss.
6. Stack Overflow serves 1.3B page views/month with just 9 web servers thanks to aggressive query caching.
7. **Never** cache queries that include time-dependent functions (`NOW()`, `RANDOM()`) — they'll never hit.

---

## What's Next?

Next, we'll explore **Cache Strategies** — the different patterns for reading from and writing to a cache (Write-Through, Write-Behind, Write-Around, Read-Through) and when each makes sense.

→ [06-cache-strategies.md](./06-cache-strategies.md)
