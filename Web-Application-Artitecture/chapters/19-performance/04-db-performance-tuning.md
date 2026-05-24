# Database Performance Tuning

> **What you'll learn**: How to make your database queries lightning fast — from indexing strategies and query optimization to connection management and hardware considerations that can turn a 5-second query into a 5-millisecond one.

---

## Real-Life Analogy

Imagine a **library with 10 million books** and no catalog system:

- **Without indexing**: "Find me all books by author 'Garcia'" → You walk through EVERY shelf, checking EVERY book. Takes hours.
- **With indexing**: "Find me all books by author 'Garcia'" → You check the author catalog (sorted alphabetically), jump to 'G', find all Garcia books in seconds.
- **With the WRONG index**: You have a catalog sorted by publication year, but you're searching by author. Still useless!

Database performance tuning is about building the **right catalogs** (indexes), asking questions the **right way** (query optimization), and organizing the library efficiently (schema design).

```
WITHOUT tuning:                    WITH tuning:
┌─────────────────────┐           ┌─────────────────────┐
│  Query: Find users  │           │  Query: Find users  │
│  in New York        │           │  in New York        │
│                     │           │                     │
│  Scans: 10,000,000  │           │  Index seek: 45,000 │
│  rows (full scan!)  │           │  rows (targeted!)   │
│                     │           │                     │
│  Time: 4,500ms 🐢   │           │  Time: 3ms ⚡       │
│  CPU: 100%          │           │  CPU: 2%            │
└─────────────────────┘           └─────────────────────┘
```

---

## Core Concept Explained Step-by-Step

### Step 1: Understanding Query Execution

Every database query goes through these stages:

```
SQL Query
    │
    ▼
┌──────────────┐
│   Parser     │ ← Validates syntax, builds parse tree
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Optimizer   │ ← Chooses the BEST execution plan
└──────┬───────┘   (which index? join order? scan type?)
       │
       ▼
┌──────────────┐
│  Executor    │ ← Runs the chosen plan
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Storage    │ ← Fetches data from disk/memory
│   Engine     │
└──────────────┘
```

The **query optimizer** is the most critical piece. It decides:
- Should it use an index or scan the whole table?
- In what order should it join tables?
- Should it sort first or filter first?

### Step 2: EXPLAIN — Your Best Friend

Every database has an `EXPLAIN` command that shows you the query plan:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42 AND status = 'shipped';
```

```
┌─────────────────────────────────────────────────────────────────┐
│ QUERY PLAN                                                      │
├─────────────────────────────────────────────────────────────────┤
│ Index Scan using idx_orders_customer on orders                  │
│   Index Cond: (customer_id = 42)                               │
│   Filter: (status = 'shipped')                                 │
│   Rows Removed by Filter: 3                                    │
│   Planning Time: 0.2ms                                         │
│   Execution Time: 0.8ms  ← FAST! ✅                           │
└─────────────────────────────────────────────────────────────────┘

vs. WITHOUT index:

┌─────────────────────────────────────────────────────────────────┐
│ Seq Scan on orders                                              │
│   Filter: (customer_id = 42 AND status = 'shipped')            │
│   Rows Removed by Filter: 9,999,985                            │
│   Planning Time: 0.1ms                                         │
│   Execution Time: 4,521ms  ← SLOW! ❌                         │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: Index Types and When to Use Each

```
┌──────────────────────────────────────────────────────────────┐
│                    INDEX TYPES                                │
├──────────────┬───────────────────────────────────────────────┤
│ B-Tree       │ Default. Good for: =, <, >, BETWEEN, ORDER BY│
│              │ Most common choice (95% of cases)             │
├──────────────┼───────────────────────────────────────────────┤
│ Hash         │ Good for: = only (exact match)               │
│              │ Faster than B-Tree for equality, but no range │
├──────────────┼───────────────────────────────────────────────┤
│ GIN          │ Good for: Full-text search, JSONB, arrays    │
│(Generalized  │ "Does this array CONTAIN this element?"      │
│ Inverted)    │                                              │
├──────────────┼───────────────────────────────────────────────┤
│ GiST         │ Good for: Geospatial, range types            │
│(Generalized  │ "Find points within this circle"             │
│ Search Tree) │                                              │
├──────────────┼───────────────────────────────────────────────┤
│ BRIN         │ Good for: Large tables with natural ordering │
│(Block Range) │ Time-series data (created_at is always       │
│              │ increasing). Very small index size!           │
└──────────────┴───────────────────────────────────────────────┘
```

### Step 4: Composite Indexes and Column Order

Column order in a composite index MATTERS enormously:

```
CREATE INDEX idx_orders_cust_status ON orders(customer_id, status);

This index is like a phone book sorted by:
  First: customer_id
  Then: status (within same customer)

┌─────────────────────────────────────────┐
│  customer_id  │  status    │  row_ptr   │
├───────────────┼────────────┼────────────┤
│      1        │  pending   │  →row 55   │
│      1        │  shipped   │  →row 12   │
│      2        │  cancelled │  →row 99   │
│      2        │  pending   │  →row 77   │
│      2        │  shipped   │  →row 33   │
│      3        │  shipped   │  →row 44   │
└───────────────┴────────────┴────────────┘

✅ WHERE customer_id = 2 AND status = 'shipped'  → Uses BOTH columns
✅ WHERE customer_id = 2                          → Uses FIRST column
❌ WHERE status = 'shipped'                       → CANNOT use index!
                                                   (leftmost prefix rule)
```

> **Rule**: A composite index on (A, B, C) can satisfy queries on:
> - (A), (A, B), or (A, B, C) ← YES
> - (B), (C), (B, C) ← NO (skips leftmost column)

### Step 5: Covering Indexes (Index-Only Scans)

```
-- Regular index: DB reads index → then fetches row from table
CREATE INDEX idx_orders_customer ON orders(customer_id);

SELECT customer_id, total_amount FROM orders WHERE customer_id = 42;
-- Step 1: Find row pointers in index
-- Step 2: Go to table to get total_amount (EXTRA I/O!)

-- Covering index: Include ALL columns needed by the query
CREATE INDEX idx_orders_customer_covering 
  ON orders(customer_id) INCLUDE (total_amount, status);

SELECT customer_id, total_amount FROM orders WHERE customer_id = 42;
-- Step 1: Find everything in the index itself!
-- Step 2: NONE! (Index-only scan — much faster)

Performance difference:
  Regular index:   Index scan + Heap fetch = 5ms
  Covering index:  Index-only scan        = 0.5ms (10x faster!)
```

### Step 6: Query Anti-Patterns

```
❌ BAD: Functions on indexed columns (kills the index)
────────────────────────────────────────────────────
  SELECT * FROM users WHERE YEAR(created_at) = 2024;
  -- Cannot use index on created_at!

✅ GOOD: Keep the column "naked"
  SELECT * FROM users WHERE created_at >= '2024-01-01' 
                        AND created_at < '2025-01-01';
  -- Uses index perfectly!

❌ BAD: SELECT * (fetches all columns)
────────────────────────────────────────────
  SELECT * FROM orders WHERE status = 'pending';
  -- Fetches 50 columns when you need 3

✅ GOOD: Select only what you need
  SELECT id, customer_id, total FROM orders WHERE status = 'pending';
  -- Less I/O, potential index-only scan

❌ BAD: OR conditions that can't use indexes
────────────────────────────────────────────────
  SELECT * FROM orders WHERE customer_id = 5 OR status = 'pending';
  -- Often results in full table scan

✅ GOOD: UNION ALL for separate index usage
  SELECT * FROM orders WHERE customer_id = 5
  UNION ALL
  SELECT * FROM orders WHERE status = 'pending' AND customer_id != 5;

❌ BAD: Leading wildcard in LIKE
────────────────────────────────────
  SELECT * FROM users WHERE email LIKE '%@gmail.com';
  -- Full scan! Can't use B-tree index

✅ GOOD: Trailing wildcard or full-text search
  SELECT * FROM users WHERE email LIKE 'john%';
  -- Uses index!
```

---

## How It Works Internally

### B-Tree Index Internals

```
B-Tree for index on "age" column:
                    ┌──────────┐
                    │  [30,60] │  ← Root node
                    └──┬───┬───┘
                  ╱    │    ╲
         ╱            │          ╲
┌────────────┐  ┌────────────┐  ┌────────────┐
│ [10,15,25] │  │ [35,42,55] │  │ [65,78,90] │  ← Internal nodes
└─┬──┬──┬──┬─┘  └─┬──┬──┬──┬─┘  └─┬──┬──┬──┬─┘
  │  │  │  │      │  │  │  │      │  │  │  │
  ▼  ▼  ▼  ▼      ▼  ▼  ▼  ▼      ▼  ▼  ▼  ▼    ← Leaf nodes
                                                     (point to rows)

Lookup "age = 42":
  Root: 42 > 30, go right... 42 < 60, go middle
  Internal: 42 > 35, 42 = 42 → FOUND!
  
  Complexity: O(log N) — with 10 million rows, only ~4 node reads!
  
  Table rows: 10,000,000
  B-tree depth: ~4 levels
  Reads needed: 4 (vs 10,000,000 for full scan!)
```

### PostgreSQL Query Optimizer Decision Tree

```
Query arrives
    │
    ├── Estimated rows to return?
    │     │
    │     ├── < 10% of table → Index Scan (if index exists)
    │     ├── 10-30% of table → Bitmap Index Scan
    │     └── > 30% of table → Sequential Scan (full table)
    │
    ├── Join strategy?
    │     │
    │     ├── Small table + indexed → Nested Loop Join
    │     ├── Medium tables → Hash Join
    │     └── Large tables (already sorted) → Merge Join
    │
    └── Sort needed?
          │
          ├── Index already sorted → Use index order (free!)
          └── No index → Sort in memory (or disk if too large)
```

### Connection Pooling Impact

```
WITHOUT connection pool:
  Each request: Connect (30ms) → Query (5ms) → Disconnect (5ms)
  Total per request: 40ms
  Under 1000 RPS: 40,000ms of connection overhead per second!
  DB max_connections hit → ERRORS

WITH connection pool (PgBouncer/HikariCP):
  Startup: Create 20 connections (once)
  Each request: Borrow (0.1ms) → Query (5ms) → Return (0.1ms)
  Total per request: 5.2ms
  Under 1000 RPS: 5,200ms total — 8x improvement!

┌──────────────────────────────────────────────────────┐
│                Connection Pool                        │
│                                                      │
│  App ──▶ ┌─── conn1 ─── conn2 ─── conn3 ───┐      │
│  App ──▶ │        Pool (20 conns)            │──▶ DB│
│  App ──▶ │   conn4 ─── conn5 ─── ... ───    │      │
│  App ──▶ └───────────────────────────────────┘      │
│                                                      │
│  1000 app threads share 20 DB connections            │
└──────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Query Optimization with SQLAlchemy + PostgreSQL

```python
from sqlalchemy import create_engine, text, Index
from sqlalchemy.orm import Session
import time

engine = create_engine(
    "postgresql://user:pass@localhost/mydb",
    pool_size=20,          # Connection pool size
    max_overflow=10,       # Extra connections if pool is full
    pool_pre_ping=True,    # Verify connections are alive
    echo=False             # Set True to see generated SQL
)

# 1. EXPLAIN ANALYZE — check query plan
def analyze_query(query_str, params=None):
    """Run EXPLAIN ANALYZE and print the plan"""
    with engine.connect() as conn:
        result = conn.execute(
            text(f"EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) {query_str}"),
            params or {}
        )
        print("Query Plan:")
        for row in result:
            print(f"  {row[0]}")

# 2. Find slow queries
def find_slow_queries():
    """Query pg_stat_statements for slowest queries"""
    with engine.connect() as conn:
        result = conn.execute(text("""
            SELECT query, calls, mean_exec_time, total_exec_time
            FROM pg_stat_statements
            ORDER BY mean_exec_time DESC
            LIMIT 10;
        """))
        for row in result:
            print(f"  Avg: {row.mean_exec_time:.1f}ms | "
                  f"Calls: {row.calls} | Query: {row.query[:80]}")

# 3. Identify missing indexes
def find_missing_indexes():
    """Find sequential scans on large tables (likely need index)"""
    with engine.connect() as conn:
        result = conn.execute(text("""
            SELECT relname, seq_scan, seq_tup_read, 
                   idx_scan, n_live_tup
            FROM pg_stat_user_tables
            WHERE seq_scan > 100 
              AND n_live_tup > 10000
              AND idx_scan < seq_scan
            ORDER BY seq_tup_read DESC
            LIMIT 10;
        """))
        for row in result:
            print(f"  Table: {row.relname} | "
                  f"Seq scans: {row.seq_scan} | "
                  f"Rows: {row.n_live_tup}")

# 4. Batch operations (avoid N+1)
def update_orders_batch(order_ids, new_status):
    """Update multiple orders in ONE query instead of N queries"""
    with Session(engine) as session:
        # ❌ BAD: N queries
        # for oid in order_ids:
        #     session.execute(text("UPDATE orders SET status=:s WHERE id=:id"), 
        #                     {"s": new_status, "id": oid})
        
        # ✅ GOOD: 1 query
        session.execute(
            text("UPDATE orders SET status = :status WHERE id = ANY(:ids)"),
            {"status": new_status, "ids": order_ids}
        )
        session.commit()
```

### Java — Database Tuning with HikariCP + JDBC

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import java.sql.*;
import java.util.List;

public class DatabaseTuning {

    // Optimized connection pool configuration
    public static HikariDataSource createPool() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("user");
        config.setPassword("pass");
        
        // Pool sizing (rule of thumb: connections = (CPU cores * 2) + disk spindles)
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        
        // Timeouts
        config.setConnectionTimeout(5000);   // 5s to get connection from pool
        config.setIdleTimeout(300000);       // 5min idle before returning
        config.setMaxLifetime(1800000);      // 30min max connection lifetime
        
        // Performance settings
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        
        return new HikariDataSource(config);
    }

    // Batch INSERT for high-throughput writes
    public static void batchInsert(HikariDataSource ds, List<Order> orders) 
            throws SQLException {
        String sql = "INSERT INTO orders (customer_id, amount, status) VALUES (?,?,?)";
        
        try (Connection conn = ds.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            
            conn.setAutoCommit(false); // Start transaction
            
            for (int i = 0; i < orders.size(); i++) {
                Order o = orders.get(i);
                ps.setLong(1, o.customerId());
                ps.setBigDecimal(2, o.amount());
                ps.setString(3, o.status());
                ps.addBatch();
                
                // Execute in batches of 1000
                if (i % 1000 == 0) {
                    ps.executeBatch();
                }
            }
            ps.executeBatch();  // Remaining
            conn.commit();
        }
        // 10,000 rows: ~50ms (batch) vs ~5,000ms (individual inserts)
    }

    // EXPLAIN ANALYZE from Java
    public static void explainQuery(HikariDataSource ds, String query) 
            throws SQLException {
        try (Connection conn = ds.getConnection();
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery("EXPLAIN ANALYZE " + query)) {
            while (rs.next()) {
                System.out.println("  " + rs.getString(1));
            }
        }
    }
}
```

---

## Infrastructure Examples

### PostgreSQL Configuration Tuning

```ini
# postgresql.conf — Performance-focused settings

# Memory (adjust based on available RAM)
shared_buffers = 4GB              # 25% of total RAM (for 16GB server)
effective_cache_size = 12GB       # 75% of total RAM
work_mem = 64MB                   # Per-operation sort/hash memory
maintenance_work_mem = 1GB        # For VACUUM, CREATE INDEX

# WAL (Write-Ahead Log)
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_wal_size = 2GB

# Query Planner
random_page_cost = 1.1            # 1.1 for SSD (default 4.0 for HDD!)
effective_io_concurrency = 200    # For SSD
default_statistics_target = 100   # More stats = better query plans

# Connections
max_connections = 200             # Use connection pooler for more!

# Parallel Queries (PostgreSQL 11+)
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
```

### MongoDB Performance Tuning

```javascript
// MongoDB index creation and optimization

// 1. Create compound index with proper order
db.orders.createIndex(
  { customer_id: 1, created_at: -1, status: 1 },
  { name: "idx_customer_orders" }
);

// 2. Partial index (only index relevant data)
db.orders.createIndex(
  { status: 1, created_at: -1 },
  { 
    partialFilterExpression: { status: "pending" },
    name: "idx_pending_orders"  // Much smaller than indexing ALL orders!
  }
);

// 3. Analyze query with explain()
db.orders.find({ customer_id: 42, status: "shipped" })
  .explain("executionStats");
// Look for: "totalDocsExamined" vs "nReturned"
// If docsExamined >> nReturned, you need a better index!

// 4. Find unused indexes (waste of space + write overhead)
db.orders.aggregate([{ $indexStats: {} }]);
```

### Redis Caching Layer for DB Performance

```yaml
# docker-compose.yml — DB + Redis cache layer
version: '3'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp
    volumes:
      - ./postgres.conf:/etc/postgresql/postgresql.conf
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 2gb --maxmemory-policy allkeys-lru
    
  pgbouncer:
    image: edoburu/pgbouncer
    environment:
      DATABASE_URL: postgres://user:pass@postgres:5432/myapp
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 20
```

---

## Real-World Example

### Instagram — Database Performance at Scale

Instagram's PostgreSQL setup handles billions of reads:

```
Instagram DB Architecture:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  App Servers (thousands)                                │
│       │                                                 │
│       ▼                                                 │
│  ┌──────────────┐                                      │
│  │  PgBouncer   │ ← Connection pooling                 │
│  │  (per host)  │    max 10K app conns → 100 DB conns │
│  └──────┬───────┘                                      │
│         │                                               │
│         ▼                                               │
│  ┌──────────────────────────────────────┐              │
│  │  PostgreSQL (sharded by user_id)     │              │
│  │                                       │              │
│  │  Shard 1: users 1-1M                 │              │
│  │  Shard 2: users 1M-2M               │              │
│  │  ...                                  │              │
│  │  Shard N: users (N-1)M - NM          │              │
│  └──────────────────────────────────────┘              │
│                                                         │
│  Key optimizations:                                     │
│  • Partial indexes on "active" data only               │
│  • Read replicas for timeline queries                   │
│  • Aggressive connection pooling                        │
│  • pg_stat_statements monitoring                        │
│  • Automated VACUUM tuning                              │
└─────────────────────────────────────────────────────────┘
```

### Uber — Database Performance for Ride Matching

```
Uber needs < 100ms for "find nearest drivers":

┌────────────────────────────────────────────────────┐
│  Problem: Find 10 nearest drivers to rider          │
│                                                    │
│  Naive approach (SLOW):                            │
│    SELECT * FROM drivers                           │
│    WHERE is_available = true                       │
│    ORDER BY distance(location, rider_location)     │
│    LIMIT 10;                                       │
│    → Scans ALL available drivers! (millions)       │
│                                                    │
│  Uber's approach (FAST):                           │
│    1. Geospatial index (R-tree / GiST)            │
│    2. Spatial partitioning (H3 hexagonal grid)     │
│    3. Only search nearby hexagons                  │
│    4. In-memory cache of driver locations           │
│    → Returns in < 10ms                             │
└────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Adding indexes on every column | Each index slows down writes + uses disk | Only index columns used in WHERE/JOIN/ORDER BY |
| Not running ANALYZE/VACUUM | Query planner uses stale statistics → bad plans | Schedule regular ANALYZE; autovacuum should be ON |
| Using ORM without checking generated SQL | ORMs often generate inefficient queries | Always check EXPLAIN for critical queries |
| Ignoring connection pool sizing | Too few = waiting; too many = DB memory exhaustion | Formula: connections ≈ (CPU cores × 2) + disk spindles |
| Indexing low-cardinality columns | Index on `gender` (M/F) is useless — not selective | Only index columns with high cardinality (many distinct values) |
| Not using LIMIT with pagination | `SELECT * FROM big_table` returns millions of rows | Always use LIMIT/OFFSET or cursor-based pagination |
| Sequential UUIDs as primary key | Random UUIDs fragment B-tree indexes | Use ULIDs, UUID v7, or auto-increment IDs |

---

## When to Use / When NOT to Use

### Tune the database WHEN:
- Query times exceed your SLA (e.g., p95 > 100ms)
- EXPLAIN shows sequential scans on large tables
- CPU/disk I/O is high on the DB server
- Connection pool wait times are increasing
- Table sizes are growing beyond RAM

### DON'T tune (yet) WHEN:
- The table has < 10,000 rows (small tables are fast regardless)
- The query runs once per day (batch job — latency doesn't matter)
- The real bottleneck is application code or network
- You haven't profiled to confirm DB is the bottleneck

---

## Key Takeaways

- **Always use EXPLAIN ANALYZE** before and after optimization — measure, don't guess
- **B-tree indexes** handle 95% of cases; learn composite index column ordering (leftmost prefix rule)
- **Covering indexes** eliminate heap fetches entirely — massive speedup for read-heavy queries
- **Connection pooling** (PgBouncer, HikariCP) is mandatory for any production system
- **Don't over-index**: each index slows writes and uses storage — only index what queries actually need
- **Keep statistics up-to-date**: stale stats = bad query plans = slow queries
- **Monitor continuously** with pg_stat_statements (PostgreSQL) or slow query log (MySQL)

---

## What's Next?

One of the most common database performance killers is something subtle that ORMs create without you noticing. Next up is **Chapter 19.5: N+1 Query Problem & Batch Loading** — where you'll learn to spot and fix the #1 cause of slow database interactions in web applications.
