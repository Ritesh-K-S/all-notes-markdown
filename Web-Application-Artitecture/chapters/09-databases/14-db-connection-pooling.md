# Database Connection Pooling & Query Optimization

> **What you'll learn**: Why opening a new database connection per request is catastrophic at scale, how connection pooling works internally, how to size your pool correctly, and essential query optimization techniques that turn 30-second queries into 30-millisecond ones.

---

## Real-Life Analogy

### Connection Pooling = Taxi Stand
Without pooling: Every passenger calls a brand new taxi from the factory, rides to their destination, then the taxi is **destroyed**. Building a taxi takes 30 seconds (TCP handshake, TLS, authentication, memory allocation). Imagine doing this for 1000 passengers per second!

With pooling: A **taxi stand** keeps 20 taxis waiting. Passengers grab an available taxi, ride, then the taxi **returns to the stand** for the next passenger. No building/destroying — just reuse!

### Query Optimization = GPS Route Planning
Without optimization: You drive every possible route and pick the fastest one after trying them all (full table scan).
With optimization: GPS (query planner) analyzes road conditions (indexes, statistics) and picks the best route BEFORE you start driving.

---

## Core Concept Explained Step-by-Step

### Why Connection Pooling is Essential

```
WITHOUT CONNECTION POOLING:
──────────────────────────────────────────────────────────────────
Request 1 ──→ [Create Connection: 30ms] → [Query: 5ms] → [Close: 5ms]
Request 2 ──→ [Create Connection: 30ms] → [Query: 5ms] → [Close: 5ms]
Request 3 ──→ [Create Connection: 30ms] → [Query: 5ms] → [Close: 5ms]
...
Request 1000 → [Create Connection: 30ms] → [Query: 5ms] → [Close: 5ms]

Total overhead: 1000 × 30ms = 30 SECONDS just creating connections!
Each connection uses ~5-10MB RAM on the DB server
1000 connections = 5-10 GB RAM just for connections!

WITH CONNECTION POOLING:
──────────────────────────────────────────────────────────────────
Pool initialized: [Create 20 connections: 600ms one-time cost]

Request 1 ──→ [Get from pool: 0.1ms] → [Query: 5ms] → [Return to pool: 0.1ms]
Request 2 ──→ [Get from pool: 0.1ms] → [Query: 5ms] → [Return to pool: 0.1ms]
...
Request 1000 → [Get from pool: 0.1ms] → [Query: 5ms] → [Return to pool: 0.1ms]

Total overhead: ~100ms (vs 30 seconds!)
Memory: 20 connections × 10MB = 200MB (vs 10GB!)
```

### Connection Pool Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    APPLICATION SERVER                              │
│                                                                    │
│  Thread 1 ──┐                                                    │
│  Thread 2 ──┤                                                    │
│  Thread 3 ──┤     ┌─────────────────────────────┐               │
│  Thread 4 ──┼────▶│     CONNECTION POOL          │               │
│  Thread 5 ──┤     │                             │               │
│  Thread 6 ──┤     │  [Conn1: BUSY] ────────────────────┐       │
│  Thread 7 ──┤     │  [Conn2: BUSY] ───────────────────┐│       │
│  Thread 8 ──┘     │  [Conn3: IDLE] ──────────────────┐││       │
│                    │  [Conn4: IDLE]                   │││       │
│  Waiting Queue:    │  [Conn5: IDLE]                   │││       │
│  [T9, T10, T11]   │                                  │││       │
│  (wait for IDLE)   │  min=5, max=20, timeout=30s     │││       │
│                    └─────────────────────────────────┘│││       │
│                                                       │││       │
└───────────────────────────────────────────────────────┼┼┼───────┘
                                                        │││
                    NETWORK                             │││
                                                        │││
┌───────────────────────────────────────────────────────┼┼┼───────┐
│                    DATABASE SERVER                     │││       │
│                                                       │││       │
│  Backend Process 1 ◀─────────────────────────────────┘││       │
│  Backend Process 2 ◀──────────────────────────────────┘│       │
│  Backend Process 3 ◀───────────────────────────────────┘       │
│  Backend Process 4 (idle - waiting for work)                    │
│  Backend Process 5 (idle - waiting for work)                    │
│                                                                  │
│  max_connections = 100 (shared across ALL app instances!)       │
└──────────────────────────────────────────────────────────────────┘
```

### Pool Sizing — The Critical Formula

```
OPTIMAL POOL SIZE (PostgreSQL recommendation):

  connections = (core_count * 2) + effective_spindle_count

  Where:
    core_count = number of CPU cores on DB server
    effective_spindle_count = number of disks (1 for SSD)

  Example: 8-core server with SSD:
    connections = (8 * 2) + 1 = 17 connections

  "But we have 10 app instances!"
  → Total connections to DB = pool_size × app_instances
  → If pool_size=20, instances=10 → 200 connections to DB!
  → DB max_connections=100 → PROBLEM!

  Solution: External connection pooler (PgBouncer, ProxySQL)

┌──────────────────────────────────────────────────────────────┐
│                POOL SIZING GUIDE                               │
├────────────────┬─────────────────────────────────────────────┤
│ Scenario       │ Recommended Pool Size                        │
├────────────────┼─────────────────────────────────────────────┤
│ Web app        │ 10-20 connections per instance               │
│ Microservice   │ 5-10 connections per instance                │
│ Background job │ 2-5 connections (long queries OK)            │
│ Read-heavy     │ More connections (reads are fast)            │
│ Write-heavy    │ Fewer connections (writes need disk I/O)     │
└────────────────┴─────────────────────────────────────────────┘

COUNTER-INTUITIVE TRUTH:
  More connections ≠ More throughput!
  
  At some point, connections COMPETE for:
  → CPU time (context switching)
  → Disk I/O bandwidth
  → Lock acquisition
  
  Adding more connections REDUCES throughput due to contention.
  The optimal point is often surprisingly low (10-30).
```

### External Connection Pooler (PgBouncer)

```
WITHOUT PgBouncer (each app connects directly):
─────────────────────────────────────────────────────────────
App Instance 1 ──20 conns──┐
App Instance 2 ──20 conns──┤
App Instance 3 ──20 conns──┼──→ PostgreSQL (60 connections!)
App Instance 4 ──20 conns──┤     max_connections = 100
App Instance 5 ──20 conns──┘     80% utilized!

WITH PgBouncer (multiplexing):
─────────────────────────────────────────────────────────────
App Instance 1 ──20 conns──┐
App Instance 2 ──20 conns──┤     ┌──────────┐
App Instance 3 ──20 conns──┼──→  │PgBouncer │ ──15 conns──→ PostgreSQL
App Instance 4 ──20 conns──┤     │(pooler)  │              max = 100
App Instance 5 ──20 conns──┘     └──────────┘              Only 15% used!
                100 "virtual"         │
                connections          Multiplexes 100 app connections
                                     into only 15 real DB connections!

PgBouncer Pooling Modes:
━━━━━━━━━━━━━━━━━━━━━━━━
Session:      Conn held for entire client session (least efficient)
Transaction:  Conn returned after each transaction (best for web apps!)
Statement:    Conn returned after each statement (most efficient but
              no multi-statement transactions)
```

---

## Query Optimization

### EXPLAIN ANALYZE — Your Diagnostic Tool

```
BEFORE optimization: "Why is this query taking 12 seconds?"

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.amount, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at > '2024-01-01'
  AND o.status = 'completed';

OUTPUT (the query plan):
─────────────────────────────────────────────────────────────
Hash Join  (cost=1234..56789 rows=50000)
           (actual time=45.123..12340.456 rows=48532 loops=1)
  → Seq Scan on orders  ← PROBLEM! Sequential scan = reads ALL rows
      Filter: (status = 'completed' AND created_at > '2024-01-01')
      Rows Removed by Filter: 9951468  ← Scanned 10M, kept 50K!
      Buffers: shared read=234567   ← Read 1.8GB from disk!
  → Hash
    → Seq Scan on users
        Buffers: shared hit=1234

Planning Time: 0.5ms
Execution Time: 12340ms  ← 12 SECONDS!

AFTER adding index:
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

Index Scan using idx_orders_status_date on orders
  (actual time=0.045..23.456 rows=48532 loops=1)
  Index Cond: (status = 'completed' AND created_at > '2024-01-01')
  Buffers: shared hit=456  ← Read only 3.6MB (vs 1.8GB!)

Execution Time: 32ms  ← 385x FASTER!
```

### Common Query Optimization Patterns

```
┌──────────────────────────────────────────────────────────────────┐
│          QUERY OPTIMIZATION CHECKLIST                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. ADD MISSING INDEXES                                           │
│     Look for "Seq Scan" in EXPLAIN on large tables               │
│     Add index on columns used in WHERE, JOIN, ORDER BY           │
│                                                                    │
│  2. USE COVERING INDEXES                                          │
│     CREATE INDEX idx ON orders(status, created_at)                │
│     INCLUDE (amount, user_id);                                    │
│     → All needed columns in index → no table lookup!             │
│                                                                    │
│  3. AVOID SELECT *                                                │
│     SELECT * loads ALL columns (including BLOBs!)                │
│     SELECT id, name, email → only loads 3 columns                │
│                                                                    │
│  4. USE LIMIT FOR PAGINATION                                      │
│     BAD: SELECT * FROM orders ORDER BY id OFFSET 10000           │
│     GOOD: SELECT * FROM orders WHERE id > :last_id LIMIT 20     │
│     (Keyset/Cursor pagination — O(1) vs O(n))                   │
│                                                                    │
│  5. BATCH OPERATIONS                                              │
│     BAD: 1000 individual INSERT statements                       │
│     GOOD: INSERT INTO orders VALUES (...), (...), (...) × 1000  │
│                                                                    │
│  6. AVOID N+1 QUERIES                                             │
│     BAD: for user in users: query(user.orders)  → 1000 queries!│
│     GOOD: SELECT * FROM orders WHERE user_id IN (1,2,3...1000)  │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Connection Pooling with SQLAlchemy

```python
from sqlalchemy import create_engine, text
from sqlalchemy.pool import QueuePool

# Create engine with connection pool
engine = create_engine(
    "postgresql://admin:secret@localhost:5432/myapp",
    
    # Pool configuration
    poolclass=QueuePool,
    pool_size=10,          # Number of persistent connections
    max_overflow=20,       # Extra connections when pool is full (temporary)
    pool_timeout=30,       # Seconds to wait for available connection
    pool_recycle=1800,     # Recycle connections after 30 minutes
    pool_pre_ping=True,    # Test connection health before using
    
    # Connection arguments
    connect_args={
        "connect_timeout": 5,
        "options": "-c statement_timeout=30000"  # 30s query timeout
    }
)

# Using the pool (connection automatically returned after 'with' block)
def get_user_orders(user_id):
    with engine.connect() as conn:  # Borrows from pool
        result = conn.execute(
            text("""
                SELECT o.id, o.amount, o.created_at
                FROM orders o
                WHERE o.user_id = :uid
                  AND o.created_at > NOW() - INTERVAL '30 days'
                ORDER BY o.created_at DESC
                LIMIT 50
            """),
            {"uid": user_id}
        )
        return result.fetchall()
    # Connection automatically returned to pool here!

# Monitor pool health
def check_pool_stats():
    pool = engine.pool
    print(f"Pool size: {pool.size()}")
    print(f"Checked out: {pool.checkedout()}")
    print(f"Overflow: {pool.overflow()}")
    print(f"Checked in (available): {pool.checkedin()}")
```

### Python — Query Optimization Examples

```python
from sqlalchemy import text

# BAD: N+1 Query Problem
def get_users_with_orders_bad():
    with engine.connect() as conn:
        users = conn.execute(text("SELECT * FROM users LIMIT 100")).fetchall()
        result = []
        for user in users:
            # 100 separate queries! One per user!
            orders = conn.execute(
                text("SELECT * FROM orders WHERE user_id = :uid"),
                {"uid": user.id}
            ).fetchall()
            result.append({"user": user, "orders": orders})
    return result  # Total: 101 queries!

# GOOD: Single JOIN query
def get_users_with_orders_good():
    with engine.connect() as conn:
        result = conn.execute(text("""
            SELECT u.id, u.name, u.email,
                   o.id as order_id, o.amount, o.created_at
            FROM users u
            LEFT JOIN orders o ON u.id = o.user_id
            WHERE u.id IN (SELECT id FROM users LIMIT 100)
            ORDER BY u.id, o.created_at DESC
        """)).fetchall()
    return result  # Total: 1 query!

# BAD: Offset pagination (scans and discards rows)
def get_orders_page_bad(page, page_size=20):
    offset = (page - 1) * page_size  # Page 500 → offset 10000!
    with engine.connect() as conn:
        return conn.execute(
            text("SELECT * FROM orders ORDER BY id OFFSET :off LIMIT :lim"),
            {"off": offset, "lim": page_size}
        ).fetchall()  # Scans 10000 rows, returns 20!

# GOOD: Cursor/Keyset pagination (instant for any page)
def get_orders_page_good(last_id=0, page_size=20):
    with engine.connect() as conn:
        return conn.execute(
            text("SELECT * FROM orders WHERE id > :last_id ORDER BY id LIMIT :lim"),
            {"last_id": last_id, "lim": page_size}
        ).fetchall()  # Uses index, returns 20 instantly!
```

### Java — HikariCP Connection Pool

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import java.sql.*;

public class ConnectionPoolExample {
    
    private static HikariDataSource dataSource;
    
    static {
        HikariConfig config = new HikariConfig();
        
        // Connection settings
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/myapp");
        config.setUsername("admin");
        config.setPassword("secret");
        
        // Pool sizing
        config.setMinimumIdle(5);          // Minimum idle connections
        config.setMaximumPoolSize(20);     // Maximum total connections
        config.setConnectionTimeout(30000); // 30s wait for connection
        config.setIdleTimeout(600000);     // 10min idle before eviction
        config.setMaxLifetime(1800000);    // 30min max connection age
        
        // Validation
        config.setConnectionTestQuery("SELECT 1");
        config.setValidationTimeout(5000);  // 5s to validate connection
        
        // Performance tuning
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        
        // Leak detection (log warning if connection held > 60s)
        config.setLeakDetectionThreshold(60000);
        
        dataSource = new HikariDataSource(config);
    }
    
    // Use the pool
    public List<Order> getRecentOrders(int userId) throws SQLException {
        List<Order> orders = new ArrayList<>();
        
        // try-with-resources: connection auto-returned to pool
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 "SELECT id, amount, created_at FROM orders " +
                 "WHERE user_id = ? AND created_at > NOW() - INTERVAL '30 days' " +
                 "ORDER BY created_at DESC LIMIT 50")) {
            
            stmt.setInt(1, userId);
            try (ResultSet rs = stmt.executeQuery()) {
                while (rs.next()) {
                    orders.add(new Order(
                        rs.getLong("id"),
                        rs.getBigDecimal("amount"),
                        rs.getTimestamp("created_at").toLocalDateTime()
                    ));
                }
            }
        }  // Connection returned to pool here!
        
        return orders;
    }
    
    // Batch insert (100x faster than individual inserts)
    public void insertOrdersBatch(List<Order> orders) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 "INSERT INTO orders (user_id, amount, status) VALUES (?, ?, ?)")) {
            
            conn.setAutoCommit(false);
            
            for (Order order : orders) {
                stmt.setInt(1, order.getUserId());
                stmt.setBigDecimal(2, order.getAmount());
                stmt.setString(3, order.getStatus());
                stmt.addBatch();
                
                // Execute in batches of 1000
                if (orders.indexOf(order) % 1000 == 0) {
                    stmt.executeBatch();
                }
            }
            stmt.executeBatch();  // Remaining items
            conn.commit();
        }
    }
}
```

---

## Infrastructure Examples

### PgBouncer Configuration

```ini
; /etc/pgbouncer/pgbouncer.ini

[databases]
myapp = host=db-primary.internal port=5432 dbname=myapp
myapp_ro = host=db-replica.internal port=5432 dbname=myapp

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

; Pool mode: transaction is best for web apps
pool_mode = transaction

; Pool sizing
default_pool_size = 20       ; Connections per user/database pair
min_pool_size = 5            ; Minimum connections kept open
max_client_conn = 1000       ; Max client connections to PgBouncer
max_db_connections = 50      ; Max connections TO the database

; Timeouts
server_connect_timeout = 5
server_idle_timeout = 60
client_idle_timeout = 300
query_timeout = 30

; Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60
```

### Docker Compose — App + PgBouncer + PostgreSQL

```yaml
version: '3.8'
services:
  app:
    image: my-web-app:latest
    environment:
      # Connect to PgBouncer, NOT directly to PostgreSQL
      DATABASE_URL: "postgresql://admin:secret@pgbouncer:6432/myapp"
    deploy:
      replicas: 5  # 5 app instances, each with its own pool
    depends_on:
      - pgbouncer

  pgbouncer:
    image: edoburu/pgbouncer:latest
    environment:
      DATABASE_URL: "postgresql://admin:secret@postgres:5432/myapp"
      POOL_MODE: transaction
      DEFAULT_POOL_SIZE: 20
      MAX_CLIENT_CONN: 500
      MAX_DB_CONNECTIONS: 50
    ports:
      - "6432:6432"
    depends_on:
      - postgres

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    command:
      - "postgres"
      - "-c" 
      - "max_connections=100"
      - "-c"
      - "shared_buffers=2GB"
      - "-c"
      - "effective_cache_size=6GB"
      - "-c"
      - "work_mem=16MB"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

---

## Real-World Example

### Uber — Connection Pool Management at Scale

```
┌─────────────────────────────────────────────────────────────────┐
│        Uber's Database Connection Architecture                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Challenge: 4000+ microservices, thousands of DB connections     │
│                                                                   │
│  Architecture:                                                   │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Microservice (per instance):                           │     │
│  │   HikariCP pool: 5 connections                        │     │
│  │   × 20 instances = 100 connections per service        │     │
│  └──────────────────────┬─────────────────────────────────┘     │
│                         │                                        │
│  ┌──────────────────────▼─────────────────────────────────┐     │
│  │ Connection Proxy Layer (custom "Schemadoc"):           │     │
│  │   - Multiplexes 100 app connections → 20 DB conns    │     │
│  │   - Automatic read/write routing                      │     │
│  │   - Query timeout enforcement (hard 30s limit)        │     │
│  │   - Connection draining during DB failover            │     │
│  └──────────────────────┬─────────────────────────────────┘     │
│                         │                                        │
│  ┌──────────────────────▼─────────────────────────────────┐     │
│  │ MySQL Cluster:                                         │     │
│  │   max_connections = 3000                              │     │
│  │   Actual active queries at any time: ~200             │     │
│  │   Most connections idle (waiting for requests)        │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                   │
│  Key Optimization:                                               │
│  • Query analysis: slow query log → automatic index suggestions │
│  • Connection leak detection: alert if conn held > 60s         │
│  • Graceful degradation: shed load when pool is exhausted      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Pool too large | DB overwhelmed, context-switching kills performance | Use formula: (cores × 2) + spindles |
| Pool too small | Requests queue up, timeouts, thread starvation | Monitor `pool.waitCount`, increase if high |
| No connection timeout | Thread waits forever for connection | Set pool_timeout (30s typical) |
| Connection leak (not returning) | Pool exhausted, app hangs | Use try-with-resources, leak detection |
| No query timeout | One slow query blocks connection forever | Set statement_timeout = 30s |
| Direct DB connections from many services | Exceeds max_connections | Use PgBouncer/ProxySQL between app and DB |
| OFFSET pagination on large tables | Scans millions of rows for page 500 | Use cursor/keyset pagination |
| SELECT * everywhere | Loads unnecessary columns, wastes memory/bandwidth | Select only needed columns |
| Missing indexes on JOIN/WHERE columns | Full table scans on every query | Run EXPLAIN, add composite indexes |
| Not using prepared statements | Re-parses SQL every time, SQL injection risk | Use parameterized queries always |

---

## When to Use / When NOT to Use

### ✅ Use Connection Pooling When:
- **Any production application** (basically always)
- Multiple requests share the same database
- Application has more threads than database can handle
- Using microservices (many services → many connections)

### ✅ Use External Pooler (PgBouncer/ProxySQL) When:
- **Multiple application instances** connect to same DB
- Need to **multiplex** hundreds of app connections into fewer DB connections
- Need **connection routing** (read/write splitting)
- During **DB failover** (drain connections gracefully)

### ❌ Do NOT Over-Pool When:
- Single-threaded scripts (1 connection is fine)
- Lambda/serverless (connections can't persist — use RDS Proxy)
- Connection count already below DB max_connections

---

## Key Takeaways

1. **Creating a DB connection is expensive** (~30ms) — pooling eliminates this overhead by reusing connections.
2. **Optimal pool size is small** — typically (cores × 2) + 1 on the DB server. More connections = more contention = SLOWER.
3. **Use external poolers** (PgBouncer, ProxySQL) when multiple app instances would exceed DB's max_connections.
4. **Always set timeouts** — connection_timeout, query_timeout, idle_timeout. One hanging query shouldn't kill your pool.
5. **EXPLAIN ANALYZE is your best friend** — it shows exactly where queries are slow and why (Seq Scan = bad, Index Scan = good).
6. **Cursor pagination > OFFSET pagination** — OFFSET scans and discards rows. Keyset pagination uses indexes and is O(1).
7. **Batch operations** — bulk INSERT is 100x faster than individual INSERTs. Use addBatch() in Java, executemany() in Python.
8. **Monitor your pool** — track checked-out connections, wait times, and timeouts. Connection exhaustion = app death.

---

## What's Next?

Next, we'll explore **Polyglot Persistence — Using Multiple Databases Together** (Chapter 9.15), where you'll learn how to choose the right database for each use case in your application and keep them in sync.
