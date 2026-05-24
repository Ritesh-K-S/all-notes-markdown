# Polyglot Persistence — Using Multiple Databases Together

> **What you'll learn**: Why one database can't optimally serve all use cases, how to choose the right database for each data type, patterns for keeping multiple databases in sync, and how companies like Netflix, Uber, and LinkedIn architect multi-database systems.

---

## Real-Life Analogy

Think of a **kitchen** in a professional restaurant:
- **Refrigerator** (PostgreSQL) — main storage, everything organized, reliable, ACID-safe
- **Spice rack** (Redis) — instant access to frequently needed items
- **Deep freezer** (S3/Glacier) — archival, rarely accessed, massive capacity
- **Prep station whiteboard** (Elasticsearch) — quick lookups, finding things fast
- **Recipe binder** (MongoDB) — flexible format, each recipe is different structure
- **Relationship chart** (Neo4j) — "this ingredient pairs with that one"

No chef uses ONLY the refrigerator for everything. Each tool is optimized for a specific job. **Polyglot persistence** means choosing the right storage for each type of data.

---

## Core Concept Explained Step-by-Step

### The Problem with One Database for Everything

```
ANTI-PATTERN: Single database doing everything poorly
──────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────┐
│              PostgreSQL (doing EVERYTHING)               │
│                                                         │
│  Users table          → OK ✓                           │
│  Orders table         → OK ✓                           │
│  Session cache        → SLOW ✗ (should be in-memory)  │
│  Full-text search     → MEDIOCRE ✗ (not specialized)  │
│  Social graph         → PAINFUL ✗ (recursive joins!)  │
│  Time-series metrics  → BLOATED ✗ (wrong data model) │
│  Event log (append)   → INEFFICIENT ✗ (write-heavy)  │
│  File metadata        → AWKWARD ✗ (schema too rigid)  │
└────────────────────────────────────────────────────────┘

Result: Mediocre at everything, optimal at nothing.
```

### Polyglot Persistence Architecture

```
CORRECT: Each database does what it's BEST at
──────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                        APPLICATION LAYER                              │
│                                                                       │
│   User Service │ Order Service │ Search │ Analytics │ Social         │
└───────┬────────┴──────┬────────┴───┬────┴─────┬─────┴────┬──────────┘
        │               │            │          │           │
        ▼               ▼            ▼          ▼           ▼
┌─────────────┐  ┌──────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│ PostgreSQL  │  │  Redis   │  │Elastic │  │ClickHse│  │ Neo4j  │
│             │  │          │  │search  │  │        │  │        │
│ • Users     │  │ • Sessions│  │• Product│ │• Events│  │• Follow │
│ • Orders    │  │ • Cart    │  │  search │ │• Metrics│ │  graph  │
│ • Payments  │  │ • Rate    │  │• Auto-  │ │• Aggre-│  │• Recomm-│
│ • Inventory │  │   limits  │  │  complete│ │  gations│ │  endations│
│             │  │ • Leaderb │  │• Logs   │ │• Funnel│  │         │
│ ACID ✓      │  │ <1ms ✓   │  │ FTS ✓   │ │ OLAP ✓ │  │Graph ✓  │
└─────────────┘  └──────────┘  └────────┘  └────────┘  └────────┘
```

### Choosing the Right Database

```
┌──────────────────────────────────────────────────────────────────────┐
│              DATABASE SELECTION DECISION MATRIX                        │
├──────────────────────┬──────────────────────┬────────────────────────┤
│ Data Type / Need     │ Best Database        │ Why                    │
├──────────────────────┼──────────────────────┼────────────────────────┤
│ Transactions, ACID   │ PostgreSQL, MySQL    │ Strong consistency     │
│ User profiles, orders│                      │ JOINS, constraints     │
├──────────────────────┼──────────────────────┼────────────────────────┤
│ Caching, sessions    │ Redis, Memcached     │ Sub-millisecond reads  │
│ Rate limiting        │                      │ In-memory, TTL         │
├──────────────────────┼──────────────────────┼────────────────────────┤
│ Full-text search     │ Elasticsearch,       │ Inverted indexes       │
│ Autocomplete, logs   │ OpenSearch           │ BM25 ranking, facets   │
├──────────────────────┼──────────────────────┼────────────────────────┤
│ Flexible documents   │ MongoDB, CouchDB     │ Schema-free, nested    │
│ CMS, product catalog │                      │ documents, fast writes │
├──────────────────────┼──────────────────────┼────────────────────────┤
│ Relationships, graph │ Neo4j, Neptune       │ Traversal queries      │
│ Social, fraud detect │                      │ Relationship-first     │
├──────────────────────┼──────────────────────┼────────────────────────┤
│ Time-series, metrics │ InfluxDB, TimescaleDB│ Time-based compression │
│ IoT, monitoring      │                      │ Retention policies     │
├──────────────────────┼──────────────────────┼────────────────────────┤
│ Analytics, OLAP      │ ClickHouse, BigQuery │ Columnar storage       │
│ Dashboards, reports  │ Redshift             │ Aggregation speed      │
├──────────────────────┼──────────────────────┼────────────────────────┤
│ Message queue/stream │ Kafka, Redis Streams │ High-throughput append │
│ Event sourcing       │                      │ Ordering guarantees    │
├──────────────────────┼──────────────────────┼────────────────────────┤
│ File/blob storage    │ S3, MinIO, GCS       │ Unlimited capacity     │
│ Images, videos       │                      │ CDN integration        │
└──────────────────────┴──────────────────────┴────────────────────────┘
```

---

## How It Works Internally

### Data Synchronization Patterns

```
PATTERN 1: CHANGE DATA CAPTURE (CDC) — Most Reliable
────────────────────────────────────────────────────────────────────────

┌────────────┐    WAL/Binlog     ┌──────────┐     ┌──────────────┐
│ PostgreSQL │───────────────────▶│ Debezium │────▶│ Kafka Topic  │
│ (source    │    (captures ALL   │ (CDC)    │     │              │
│  of truth) │     changes)       └──────────┘     └──────┬───────┘
└────────────┘                                            │
                                                          │
         ┌────────────────────────────────────────────────┼────┐
         │                                                │    │
         ▼                                                ▼    ▼
┌──────────────┐                               ┌────────┐ ┌────────┐
│Elasticsearch │                               │ Redis  │ │ClickHse│
│(search index)│                               │(cache) │ │(OLAP)  │
└──────────────┘                               └────────┘ └────────┘

Advantages:
  ✅ Source DB unaware of consumers (no code changes)
  ✅ Guaranteed delivery (Kafka durability)
  ✅ Multiple consumers from single stream
  ✅ Can replay events (rebuild any downstream DB)


PATTERN 2: DUAL WRITE — Simple but Dangerous
────────────────────────────────────────────────────────────────────────

Application code:
  1. Write to PostgreSQL  ✓
  2. Write to Redis       ✓  ← What if this fails?
  3. Write to Elasticsearch  ✗  ← DATA INCONSISTENCY!

Problems:
  ❌ Partial failures = inconsistent state
  ❌ No atomicity across different databases
  ❌ Retry logic is complex and error-prone


PATTERN 3: OUTBOX PATTERN — Best of Both Worlds
────────────────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL (single ACID transaction):                        │
│                                                              │
│   BEGIN;                                                     │
│     INSERT INTO orders (...) VALUES (...);                   │
│     INSERT INTO outbox (event_type, payload) VALUES          │
│       ('ORDER_CREATED', '{"id": 123, "amount": 99.99}');   │
│   COMMIT;   ← Both in SAME transaction = atomic!           │
│                                                              │
└────────────────────────────────────────────────┬────────────┘
                                                  │
  Outbox Poller (background process):            │
  ─────────────────────────────────              │
    SELECT * FROM outbox WHERE processed = false │
    → Publish to Kafka                           │
    → Mark as processed                          │
    → Consumers update Elasticsearch, Redis, etc.│
```

### Consistency Challenges

```
THE FUNDAMENTAL PROBLEM:
─────────────────────────
If data lives in multiple databases, they WILL be temporarily inconsistent.
You must decide: how much inconsistency is acceptable?

Timeline of a write:
───────────────────────────────────────────────────────────────────────
t=0ms:   Write to PostgreSQL (source of truth) ✓
t=5ms:   CDC captures change from WAL
t=15ms:  Published to Kafka topic
t=50ms:  Elasticsearch consumer processes event
t=55ms:  Search index updated ← 55ms BEHIND PostgreSQL!

During those 55ms:
  - PostgreSQL shows: New order exists ✓
  - Elasticsearch: Order NOT yet searchable ✗
  - Redis cache: Still has stale data ✗

This is EXPECTED. Design your UI to handle it:
  - Show "Order created!" from DB response (not from search)
  - Eventual consistency for non-critical reads (search, analytics)
  - Strong consistency only for critical reads (balance, inventory)
```

---

## Code Examples

### Python — Multi-Database Service

```python
import psycopg2
import redis
from elasticsearch import Elasticsearch
from pymongo import MongoClient
import json

class OrderService:
    """Service that uses multiple databases, each for its strength"""
    
    def __init__(self):
        # PostgreSQL: Source of truth for orders (ACID)
        self.pg = psycopg2.connect(
            "host=localhost dbname=shop user=admin password=secret"
        )
        # Redis: Caching hot data (speed)
        self.redis = redis.Redis(host='localhost', port=6379, db=0)
        # Elasticsearch: Full-text search (search capability)
        self.es = Elasticsearch(['http://localhost:9200'])
        # MongoDB: Product catalog (flexible schema)
        self.mongo = MongoClient('localhost', 27017).shop
    
    def create_order(self, user_id, items, total):
        """Write path: PostgreSQL (truth) + Outbox pattern"""
        cursor = self.pg.cursor()
        try:
            cursor.execute("BEGIN")
            
            # 1. Insert order in PostgreSQL (ACID, source of truth)
            cursor.execute("""
                INSERT INTO orders (user_id, items, total, status, created_at)
                VALUES (%s, %s, %s, 'pending', NOW())
                RETURNING id
            """, (user_id, json.dumps(items), total))
            order_id = cursor.fetchone()[0]
            
            # 2. Write to outbox table (same transaction!)
            cursor.execute("""
                INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
                VALUES ('Order', %s, 'OrderCreated', %s)
            """, (order_id, json.dumps({
                'order_id': order_id,
                'user_id': user_id,
                'items': items,
                'total': float(total),
                'status': 'pending'
            })))
            
            self.pg.commit()
            
            # 3. Eagerly update cache (best-effort, not critical)
            try:
                self.redis.setex(
                    f"order:{order_id}",
                    3600,  # 1 hour TTL
                    json.dumps({'id': order_id, 'status': 'pending', 'total': float(total)})
                )
            except redis.RedisError:
                pass  # Cache miss is OK, will be populated on next read
            
            return order_id
            
        except Exception:
            self.pg.rollback()
            raise
    
    def search_orders(self, query, user_id=None):
        """Read path: Elasticsearch for full-text search"""
        body = {
            "query": {
                "bool": {
                    "must": [{"match": {"items.name": query}}],
                    "filter": [{"term": {"user_id": user_id}}] if user_id else []
                }
            },
            "sort": [{"created_at": "desc"}],
            "size": 20
        }
        result = self.es.search(index="orders", body=body)
        return [hit['_source'] for hit in result['hits']['hits']]
    
    def get_order(self, order_id):
        """Read path: Redis (cache) → PostgreSQL (fallback)"""
        # Try cache first
        cached = self.redis.get(f"order:{order_id}")
        if cached:
            return json.loads(cached)
        
        # Cache miss: read from PostgreSQL
        cursor = self.pg.cursor()
        cursor.execute("SELECT id, user_id, total, status FROM orders WHERE id = %s",
                      (order_id,))
        row = cursor.fetchone()
        if row:
            order = {'id': row[0], 'user_id': row[1], 'total': float(row[2]), 'status': row[3]}
            # Populate cache for next time
            self.redis.setex(f"order:{order_id}", 3600, json.dumps(order))
            return order
        return None
    
    def get_product(self, product_id):
        """Read path: MongoDB for flexible product catalog"""
        return self.mongo.products.find_one(
            {"_id": product_id},
            {"_id": 0, "name": 1, "price": 1, "variants": 1, "attributes": 1}
        )


# Outbox processor (runs as background service)
class OutboxProcessor:
    """Reads outbox and syncs to other databases"""
    
    def __init__(self, pg, es, redis_client):
        self.pg = pg
        self.es = es
        self.redis = redis_client
    
    def process_pending_events(self):
        cursor = self.pg.cursor()
        cursor.execute("""
            SELECT id, event_type, payload FROM outbox 
            WHERE processed = false 
            ORDER BY created_at 
            LIMIT 100 FOR UPDATE SKIP LOCKED
        """)
        
        for event_id, event_type, payload in cursor.fetchall():
            data = json.loads(payload)
            
            if event_type == 'OrderCreated':
                # Sync to Elasticsearch
                self.es.index(index='orders', id=data['order_id'], document=data)
            
            elif event_type == 'OrderStatusChanged':
                # Update Elasticsearch
                self.es.update(index='orders', id=data['order_id'],
                             doc={'status': data['new_status']})
                # Invalidate cache
                self.redis.delete(f"order:{data['order_id']}")
            
            # Mark as processed
            cursor.execute("UPDATE outbox SET processed = true WHERE id = %s",
                         (event_id,))
        
        self.pg.commit()
```

### Java — Multi-Database Architecture with Spring

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ProductService {
    
    private final JdbcTemplate postgres;      // Inventory (ACID)
    private final RedisTemplate<String, String> redis;  // Cache
    private final ElasticsearchClient elastic; // Search
    private final MongoTemplate mongo;         // Catalog
    
    // Read: Cache → Database (Cache-Aside Pattern)
    public Product getProduct(String productId) {
        // 1. Check Redis cache first
        String cached = redis.opsForValue().get("product:" + productId);
        if (cached != null) {
            return objectMapper.readValue(cached, Product.class);
        }
        
        // 2. Cache miss — fetch from MongoDB (product catalog)
        Product product = mongo.findById(productId, Product.class);
        if (product != null) {
            // 3. Add stock info from PostgreSQL (accurate inventory)
            Integer stock = postgres.queryForObject(
                "SELECT quantity FROM inventory WHERE product_id = ?",
                Integer.class, productId);
            product.setStockQuantity(stock);
            
            // 4. Populate cache
            redis.opsForValue().set(
                "product:" + productId,
                objectMapper.writeValueAsString(product),
                Duration.ofMinutes(15)
            );
        }
        return product;
    }
    
    // Write: Source of truth + Outbox for sync
    @Transactional
    public void purchaseProduct(String productId, int quantity, String userId) {
        // 1. Deduct inventory (PostgreSQL, ACID, source of truth)
        int updated = postgres.update(
            "UPDATE inventory SET quantity = quantity - ? " +
            "WHERE product_id = ? AND quantity >= ?",
            quantity, productId, quantity);
        
        if (updated == 0) {
            throw new OutOfStockException(productId);
        }
        
        // 2. Write to outbox (same transaction as inventory update!)
        postgres.update(
            "INSERT INTO outbox (event_type, aggregate_id, payload) " +
            "VALUES (?, ?, ?::jsonb)",
            "InventoryDeducted", productId,
            String.format("{\"productId\":\"%s\",\"quantity\":%d,\"userId\":\"%s\"}",
                productId, quantity, userId));
        
        // 3. Invalidate cache (best-effort)
        redis.delete("product:" + productId);
    }
    
    // Search: Elasticsearch for full-text product search
    public List<Product> searchProducts(String query, String category) {
        SearchResponse<Product> response = elastic.search(s -> s
            .index("products")
            .query(q -> q
                .bool(b -> b
                    .must(m -> m.multiMatch(mm -> mm
                        .query(query)
                        .fields("name^3", "description", "tags")
                    ))
                    .filter(f -> category != null ? 
                        f.term(t -> t.field("category").value(category)) : 
                        f.matchAll(ma -> ma))
                )
            )
            .size(20),
            Product.class
        );
        
        return response.hits().hits().stream()
            .map(Hit::source)
            .collect(Collectors.toList());
    }
}
```

---

## Infrastructure Example

### Docker Compose — Complete Polyglot Stack

```yaml
version: '3.8'
services:
  # Source of truth: Transactional data
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: shop
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports: ["5432:5432"]

  # Cache: Sub-millisecond reads
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
    ports: ["6379:6379"]

  # Search: Full-text & faceted
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ports: ["9200:9200"]

  # Flexible catalog: Schema-free documents
  mongodb:
    image: mongo:7
    ports: ["27017:27017"]

  # CDC: Sync PostgreSQL → Kafka → Others
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    ports: ["9092:9092"]

  # Debezium: CDC from PostgreSQL
  debezium:
    image: debezium/connect:2.4
    depends_on: [kafka, postgres]
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: debezium-group
    ports: ["8083:8083"]

volumes:
  pgdata:
```

---

## Real-World Example

### Netflix — Polyglot Persistence at Scale

```
┌─────────────────────────────────────────────────────────────────┐
│              Netflix Multi-Database Architecture                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Cassandra (primary data store)                           │    │
│  │ • Viewing history (billions of records)                  │    │
│  │ • User preferences                                      │    │
│  │ • Why: Massive write throughput, multi-region            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ MySQL/PostgreSQL (via CockroachDB)                       │    │
│  │ • Billing & subscriptions                                │    │
│  │ • Content licensing                                      │    │
│  │ • Why: ACID for financial data                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Elasticsearch                                            │    │
│  │ • Content search ("find movies about...")                │    │
│  │ • Logs (100+ PB of operational data)                    │    │
│  │ • Why: Full-text search, log aggregation                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Redis / EVCache (custom memcached)                       │    │
│  │ • Session data                                           │    │
│  │ • API response caching                                   │    │
│  │ • Real-time recommendations                              │    │
│  │ • Why: Sub-millisecond latency                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Apache Kafka                                             │    │
│  │ • Event streaming between all services                   │    │
│  │ • 700 billion events/day                                 │    │
│  │ • Sync between databases (CDC)                           │    │
│  │ • Why: Decouples producers from consumers                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Key Principle:                                                   │
│  "Use the right tool for the job. Kafka connects them all."     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using 5 databases from day one | Operational nightmare, team can't manage | Start with PostgreSQL + Redis, add others when needed |
| Dual writes without outbox | Partial failures = inconsistent data | Use Outbox pattern or CDC (Debezium) |
| No clear source of truth | Which database has the "correct" value? | Designate ONE database as source of truth per entity |
| Synchronous cross-DB operations | One slow DB blocks everything | Async sync via events/CDC |
| Ignoring operational cost | Each DB needs monitoring, backups, patching | Factor in team expertise and on-call burden |
| Tight coupling between services and DBs | Changing one DB breaks multiple services | Each service owns its database(s) |
| Not handling eventual consistency in UI | User creates record but search doesn't show it | After write, read from source of truth (not cache/search) |

---

## When to Use / When NOT to Use

### ✅ Use Polyglot Persistence When:
- **Different access patterns**: Some data is read-heavy (cache), some write-heavy (append logs)
- **Performance requirements differ**: Sub-ms cache + complex analytics in same app
- **Data structures vary**: Relational orders + graph relationships + documents
- **Scale is uneven**: 1000 writes/sec to one type, 100K reads/sec to another
- **Team has expertise**: You can actually operate multiple databases

### ❌ Do NOT Use (Stay Single DB) When:
- **Small team** — operating multiple databases is complex
- **Simple app** — PostgreSQL handles 90% of use cases well enough
- **Early stage** — premature optimization, add complexity later when you NEED it
- **No clear performance bottleneck** — adding databases without measuring is waste
- **Consistency is critical everywhere** — multi-DB consistency is hard

### Migration Path:

```
Stage 1 (Start): PostgreSQL only
Stage 2 (Cache): PostgreSQL + Redis (cache hot queries)
Stage 3 (Search): + Elasticsearch (full-text search)
Stage 4 (Analytics): + ClickHouse (reporting/dashboards)
Stage 5 (Scale): + Cassandra or DynamoDB (massive write throughput)
```

---

## Key Takeaways

1. **Polyglot persistence** = using the best database for each specific data access pattern, not forcing one DB to do everything.
2. **PostgreSQL is your starting point** — it handles most use cases. Add specialized databases only when you hit real limitations.
3. **One source of truth per entity** — always know which database has the "correct" data. Others are derived/cached copies.
4. **CDC (Change Data Capture) is the safest sync pattern** — captures all changes from the DB's write-ahead log, no code changes needed.
5. **Dual writes are dangerous** — if you write to two databases in application code, partial failures cause inconsistency. Use the Outbox pattern instead.
6. **Each database you add costs operational burden** — monitoring, backups, upgrades, on-call expertise. Don't add databases you can't support.
7. **Embrace eventual consistency for non-critical paths** — search results, analytics, and caches can be seconds behind. Only source-of-truth reads need strong consistency.

---

## What's Next?

Next, we'll explore **NewSQL Databases (CockroachDB, Google Spanner, TiDB)** (Chapter 9.16), where you'll learn about databases that combine the scalability of NoSQL with the ACID guarantees of traditional SQL — the best of both worlds at planet scale.
