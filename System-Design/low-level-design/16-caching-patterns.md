# Caching Patterns in LLD

Caching stores frequently accessed data in a fast-access layer to reduce latency and database load. Understanding caching strategies is essential for designing performant systems.

---

## 1. Caching Strategies

### Cache-Aside (Lazy Loading)

Application manages the cache explicitly. Most common pattern.

```
Read:  App → Cache → (miss) → DB → write to Cache → return
Write: App → DB → invalidate/delete Cache
```

```java
public class ProductService {
    private final Cache<String, Product> cache;
    private final ProductRepository repository;

    public Product getProduct(String id) {
        // Check cache first
        Product cached = cache.get(id);
        if (cached != null) return cached;

        // Cache miss — load from DB
        Product product = repository.findById(id)
            .orElseThrow(() -> new NotFoundException("Product not found"));

        cache.put(id, product, Duration.ofMinutes(30));
        return product;
    }

    public Product updateProduct(String id, ProductUpdateRequest request) {
        Product product = repository.findById(id).orElseThrow();
        product.update(request);
        repository.save(product);
        cache.evict(id);  // Invalidate cache
        return product;
    }
}
```

### Write-Through

Every write goes to cache AND database together.

```
Write: App → Cache + DB (synchronous)
Read:  App → Cache (always hit)
```

```java
public Product saveProduct(Product product) {
    Product saved = repository.save(product);
    cache.put(saved.getId(), saved);  // Write to cache immediately
    return saved;
}
```

### Write-Behind (Write-Back)

Write to cache first, flush to DB asynchronously.

```
Write: App → Cache → (async batch) → DB
```

- Faster writes, but risk of data loss if cache crashes
- Good for: counters, analytics, non-critical writes

### Read-Through

Cache itself loads data from DB on a miss (cache manages the logic).

```java
// The cache layer handles DB fetching transparently
LoadingCache<String, Product> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(30))
    .build(key -> repository.findById(key).orElse(null));  // Auto-load on miss

Product product = cache.get(productId);  // Handles miss automatically
```

---

## 2. Cache Eviction Policies

| Policy | Evicts | Best For |
|--------|--------|----------|
| **LRU** (Least Recently Used) | Oldest accessed item | General purpose |
| **LFU** (Least Frequently Used) | Least accessed item | Hot/cold data split |
| **FIFO** (First In First Out) | Oldest inserted item | Simple, time-based |
| **TTL** (Time To Live) | Expired items | Data with known staleness |
| **Random** | Random item | When all items are equally likely |

---

## 3. LRU Cache Implementation

Classic LLD interview question. Use **HashMap + Doubly Linked List** for O(1) get/put.

```java
public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head;  // Dummy head (most recent)
    private final Node<K, V> tail;  // Dummy tail (least recent)

    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;

        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        moveToHead(node);  // Mark as recently used
        return node.value;
    }

    public void put(K key, V value) {
        Node<K, V> existing = map.get(key);
        if (existing != null) {
            existing.value = value;
            moveToHead(existing);
        } else {
            if (map.size() >= capacity) {
                Node<K, V> lru = tail.prev;  // Least recently used
                removeNode(lru);
                map.remove(lru.key);
            }
            Node<K, V> newNode = new Node<>(key, value);
            addToHead(newNode);
            map.put(key, newNode);
        }
    }

    private void addToHead(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToHead(Node<K, V> node) {
        removeNode(node);
        addToHead(node);
    }
}
```

---

## 4. Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things."

### Strategies

| Strategy | How | Trade-off |
|----------|-----|-----------|
| **TTL (Time-based)** | Data expires after X minutes | Simple, may serve stale data |
| **Event-based** | Invalidate on write/update events | Real-time, more complex |
| **Version-based** | Cache key includes version number | Precise, extra bookkeeping |

### Event-Based Invalidation

```java
// When data changes, publish event to invalidate cache
public class ProductService {
    private final EventPublisher eventPublisher;

    public void updateProduct(String id, ProductUpdate update) {
        repository.save(update);
        eventPublisher.publish(new CacheInvalidationEvent("product", id));
    }
}

// Cache listener
@EventListener
public void onCacheInvalidation(CacheInvalidationEvent event) {
    cache.evict(event.getEntityType() + ":" + event.getEntityId());
}
```

---

## 5. Distributed Caching Concerns

### Cache Stampede (Thundering Herd)

**Problem:** When a popular cache key expires, hundreds of requests hit the DB simultaneously.

**Solution: Locking**

```java
public Product getProductWithLock(String id) {
    Product cached = cache.get(id);
    if (cached != null) return cached;

    // Only one thread fetches from DB
    String lockKey = "lock:product:" + id;
    if (distributedLock.tryLock(lockKey, Duration.ofSeconds(5))) {
        try {
            // Double-check after acquiring lock
            cached = cache.get(id);
            if (cached != null) return cached;

            Product product = repository.findById(id).orElseThrow();
            cache.put(id, product, Duration.ofMinutes(30));
            return product;
        } finally {
            distributedLock.unlock(lockKey);
        }
    }

    // Other threads wait and retry
    return retryGetFromCache(id);
}
```

### Cache Key Design

```java
// Pattern: {entity}:{id}:{version/qualifier}
"product:12345"
"user:67890:profile"
"search:electronics:page:1:size:20"

// Use hash for complex queries
String key = "search:" + DigestUtils.md5Hex(queryParams.toString());
```

---

## 6. Spring Cache Abstraction

```java
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product getProduct(String id) {
        return repository.findById(id).orElseThrow();
    }

    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return repository.save(product);
    }

    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(String id) {
        repository.deleteById(id);
    }

    @CacheEvict(value = "products", allEntries = true)
    public void clearAllProducts() { }
}
```

---

## Quick Reference

| Pattern | When to Use |
|---------|------------|
| **Cache-Aside** | Default choice, works with any DB |
| **Read-Through** | Cache library supports auto-loading |
| **Write-Through** | Must guarantee cache consistency |
| **Write-Behind** | High write throughput, eventual consistency OK |
| **LRU** | Memory-bounded cache, general access patterns |
| **TTL** | Data has predictable staleness |
| **Event Invalidation** | Real-time consistency needed |
