# Database Indexing — Making Queries Lightning Fast

> **What you'll learn**: How database indexes work internally (B-Trees, hash indexes, GIN, GiST), when to add indexes, when NOT to, and how the right index can make a query 1000x faster.

---

## Real-Life Analogy

Imagine a **phone book** with 10 million entries:

- **Without an index** (no alphabetical order): To find "Sharma, Rahul" you'd flip through EVERY page until you found it. Average: 5 million pages checked. 😱
- **With an index** (alphabetically sorted): Jump to "S", then "Sh", then "Sha" — found in ~20 lookups! 🚀

That's the difference between a **full table scan** (reading every row) and an **index lookup** (jumping straight to the answer). For a table with 1 billion rows:
- Without index: ~10 seconds
- With index: ~0.001 seconds (1 millisecond)

---

## Core Concept Explained Step-by-Step

### What is a Database Index?

An index is a **separate data structure** that maintains a sorted copy of specific columns, with pointers back to the actual rows:

```
TABLE (unsorted, as data was inserted):
┌──────┬──────────────────┬─────────────┬──────┐
│ id   │ name             │ email       │ age  │
├──────┼──────────────────┼─────────────┼──────┤
│ 1    │ Charlie          │ c@mail.co   │ 22   │
│ 2    │ Alice            │ a@mail.co   │ 28   │
│ 3    │ Dave             │ d@mail.co   │ 35   │
│ 4    │ Bob              │ b@mail.co   │ 30   │
│ 5    │ Eve              │ e@mail.co   │ 25   │
└──────┴──────────────────┴─────────────┴──────┘

INDEX on "name" (sorted, with pointers):
┌──────────────────┬──────────────┐
│ name (sorted)    │ Row pointer  │
├──────────────────┼──────────────┤
│ Alice            │ → Row 2      │
│ Bob              │ → Row 4      │
│ Charlie          │ → Row 1      │
│ Dave             │ → Row 3      │
│ Eve              │ → Row 5      │
└──────────────────┴──────────────┘

Query: SELECT * FROM users WHERE name = 'Dave'
Without index: Scan all 5 rows → O(n)
With index: Binary search on sorted index → O(log n) → Row 3!
```

### Types of Indexes

```
┌────────────────────────────────────────────────────────────────────┐
│                      INDEX TYPES                                     │
├───────────────┬────────────────────────────────────────────────────┤
│ B-Tree        │ Default. Balanced tree for =, <, >, BETWEEN, ORDER│
│               │ BY. Works for most queries.                        │
├───────────────┼────────────────────────────────────────────────────┤
│ Hash          │ Equality only (=). Faster than B-Tree for exact   │
│               │ match, but no range queries.                       │
├───────────────┼────────────────────────────────────────────────────┤
│ GIN           │ Generalized Inverted Index. For arrays, JSONB,    │
│               │ full-text search. "Which rows contain value X?"   │
├───────────────┼────────────────────────────────────────────────────┤
│ GiST          │ For geometric data, range types, nearest-neighbor.│
│               │ "Find all points within 5km."                     │
├───────────────┼────────────────────────────────────────────────────┤
│ BRIN          │ Block Range Index. For naturally ordered data      │
│               │ (timestamps). Tiny index, huge tables.            │
├───────────────┼────────────────────────────────────────────────────┤
│ Composite     │ Multi-column index. (country, city, name).        │
│               │ Order matters! Left-to-right.                     │
├───────────────┼────────────────────────────────────────────────────┤
│ Covering      │ Index includes all query columns — no table       │
│               │ access needed (index-only scan).                   │
├───────────────┼────────────────────────────────────────────────────┤
│ Partial       │ Index only rows matching a condition.             │
│               │ "Only index active users" → smaller, faster.      │
└───────────────┴────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### B-Tree Index (Most Common)

```
B-Tree for index on "age" column:
                                    
                        ┌───────────────┐
                        │   [30, 60]    │         Root
                        └──┬─────┬──┬──┘
                           │     │  │
              ┌────────────┘     │  └────────────┐
              ▼                  ▼                ▼
       ┌───────────┐     ┌───────────┐    ┌───────────┐
       │ [10, 20]  │     │ [40, 50]  │    │ [70, 80]  │    Internal
       └─┬───┬──┬──┘     └─┬───┬──┬──┘    └─┬───┬──┬──┘
         │   │  │           │   │  │          │   │  │
         ▼   ▼  ▼           ▼   ▼  ▼          ▼   ▼  ▼
        Leaf Nodes: Contain actual values + row pointers

Query: WHERE age = 40
Path:  Root [30,60] → go middle → [40,50] → found 40! → row pointer
Comparisons: 3 (vs scanning potentially millions of rows)

For 1 BILLION rows: ~30 comparisons (log₂(1B) ≈ 30)
```

### How B-Tree Supports Different Queries

```
INDEX on "age":

Query                          │ Can Use Index? │ How?
──────────────────────────────┼───────────────┼────────────────────────
WHERE age = 25                │ ✅ Yes        │ Tree traversal to exact value
WHERE age > 30                │ ✅ Yes        │ Find 30, scan right (all larger)
WHERE age BETWEEN 20 AND 40   │ ✅ Yes        │ Find 20, scan until 40
WHERE age IN (25, 30, 35)     │ ✅ Yes        │ Multiple index lookups
ORDER BY age                  │ ✅ Yes        │ Leaf nodes are already sorted!
WHERE age != 25               │ ❌ Mostly no  │ Would scan most of the index
WHERE UPPER(name) = 'ALICE'   │ ❌ No         │ Function on column breaks index
```

### Composite Index (Multi-Column)

```
CREATE INDEX idx_location ON users(country, city, name);

This index is like a phone book sorted by: Country → City → Name

Can use this index:
✅ WHERE country = 'India'                          (leftmost column)
✅ WHERE country = 'India' AND city = 'Mumbai'      (first two columns)
✅ WHERE country = 'India' AND city = 'Mumbai' AND name = 'Alice' (all three)
✅ WHERE country = 'India' ORDER BY city            (prefix + sort)

CANNOT use this index:
❌ WHERE city = 'Mumbai'                            (skipped leftmost column!)
❌ WHERE name = 'Alice'                             (skipped first two columns)
❌ WHERE city = 'Mumbai' AND name = 'Alice'         (leftmost missing)

RULE: Must use columns LEFT TO RIGHT (like a phone book:
      can't search by first name without knowing last name)
```

### Index-Only Scan (Covering Index)

```
-- Without covering index:
SELECT name, email FROM users WHERE age > 25;

Step 1: Search age index → find matching row pointers
Step 2: Go to TABLE to fetch name and email (random I/O!) ← SLOW

-- With covering index:
CREATE INDEX idx_cover ON users(age) INCLUDE (name, email);

Step 1: Search age index → found! Name and email are IN the index!
Step 2: No table access needed! ← FAST (index-only scan)

Visualization:
┌─────────────────────────────────────────────────┐
│ COVERING INDEX on (age) INCLUDE (name, email)   │
├──────┬─────────────┬──────────────┬─────────────┤
│ age  │ name        │ email        │ row_ptr     │
├──────┼─────────────┼──────────────┼─────────────┤
│ 22   │ Charlie     │ c@mail.co    │ (not needed)│
│ 25   │ Eve         │ e@mail.co    │ (not needed)│
│ 28   │ Alice       │ a@mail.co    │ (not needed)│
│ 30   │ Bob         │ b@mail.co    │ (not needed)│
│ 35   │ Dave        │ d@mail.co    │ (not needed)│
└──────┴─────────────┴──────────────┴─────────────┘
All data served directly from index — zero table I/O!
```

### EXPLAIN ANALYZE — Reading Query Plans

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@mail.co';

-- WITHOUT index:
Seq Scan on users  (cost=0.00..18584.00 rows=1 width=72)
  Filter: (email = 'alice@mail.co')
  Rows Removed by Filter: 999999
  Actual Time: 125.432ms

-- WITH index on email:
Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=72)
  Index Cond: (email = 'alice@mail.co')
  Actual Time: 0.032ms

-- 4000x faster! (125ms → 0.032ms)
```

---

## Code Examples

### Python (PostgreSQL with indexes)

```python
import psycopg2
import time

conn = psycopg2.connect("host=localhost dbname=myapp user=admin password=secret")
cursor = conn.cursor()

# Create table and populate with 1 million rows
cursor.execute("""
    CREATE TABLE IF NOT EXISTS orders (
        id SERIAL PRIMARY KEY,
        user_id INTEGER NOT NULL,
        product_name TEXT NOT NULL,
        amount DECIMAL(10,2) NOT NULL,
        status TEXT NOT NULL,
        created_at TIMESTAMP DEFAULT NOW()
    )
""")

# Measure query WITHOUT index
start = time.time()
cursor.execute("""
    SELECT * FROM orders 
    WHERE user_id = 12345 AND status = 'completed'
    ORDER BY created_at DESC LIMIT 10
""")
print(f"Without index: {(time.time() - start)*1000:.1f}ms")

# Create composite index for this query pattern
cursor.execute("""
    CREATE INDEX CONCURRENTLY idx_orders_user_status_date 
    ON orders(user_id, status, created_at DESC)
""")
# CONCURRENTLY = doesn't lock the table during creation!

# Measure query WITH index
start = time.time()
cursor.execute("""
    SELECT * FROM orders 
    WHERE user_id = 12345 AND status = 'completed'
    ORDER BY created_at DESC LIMIT 10
""")
print(f"With index: {(time.time() - start)*1000:.1f}ms")

# Partial index — only index rows you actually query
cursor.execute("""
    CREATE INDEX idx_active_orders ON orders(user_id, created_at)
    WHERE status = 'pending'
""")
# This index is MUCH smaller than indexing all orders!

# Check if index is being used
cursor.execute("""
    EXPLAIN ANALYZE 
    SELECT * FROM orders 
    WHERE user_id = 12345 AND status = 'completed'
    ORDER BY created_at DESC LIMIT 10
""")
for row in cursor.fetchall():
    print(row[0])

# Find unused indexes (wasting space and slowing writes)
cursor.execute("""
    SELECT indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
    FROM pg_stat_user_indexes
    WHERE idx_scan = 0  -- Never used!
    ORDER BY pg_relation_size(indexrelid) DESC
""")
print("\nUnused indexes (candidates for removal):")
for row in cursor.fetchall():
    print(f"  {row[0]}: {row[2]} (0 scans)")

conn.commit()
```

### Java (Index-aware queries)

```java
import java.sql.*;

public class IndexingExample {
    public static void main(String[] args) throws SQLException {
        Connection conn = DriverManager.getConnection(
            "jdbc:postgresql://localhost/myapp", "admin", "secret");

        // Create indexes for common query patterns
        Statement stmt = conn.createStatement();
        
        // Composite index: matches WHERE + ORDER BY
        stmt.execute("""
            CREATE INDEX IF NOT EXISTS idx_orders_user_date 
            ON orders(user_id, created_at DESC)
        """);

        // GIN index for JSONB queries
        stmt.execute("""
            CREATE INDEX IF NOT EXISTS idx_products_specs 
            ON products USING GIN(specs jsonb_path_ops)
        """);

        // Partial index: only active sessions
        stmt.execute("""
            CREATE INDEX IF NOT EXISTS idx_active_sessions 
            ON sessions(user_id, expires_at)
            WHERE expires_at > NOW()
        """);

        // Query that uses the composite index perfectly
        PreparedStatement pstmt = conn.prepareStatement("""
            SELECT id, product_name, amount, created_at
            FROM orders
            WHERE user_id = ? 
            ORDER BY created_at DESC
            LIMIT 20
        """);
        pstmt.setInt(1, 12345);

        long start = System.nanoTime();
        ResultSet rs = pstmt.executeQuery();
        long elapsed = (System.nanoTime() - start) / 1_000_000;
        
        System.out.printf("Query took: %dms%n", elapsed);
        while (rs.next()) {
            System.out.printf("  Order #%d: %s ($%.2f)%n",
                rs.getInt("id"),
                rs.getString("product_name"),
                rs.getDouble("amount"));
        }

        conn.close();
    }
}
```

---

## Infrastructure Examples

### PostgreSQL Index Management Script

```sql
-- Monitor index usage statistics
SELECT 
    schemaname || '.' || tablename AS table,
    indexname,
    idx_scan AS times_used,
    idx_tup_read AS rows_read,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Find slow queries that need indexes
SELECT 
    query,
    calls,
    mean_exec_time AS avg_ms,
    total_exec_time AS total_ms
FROM pg_stat_statements
WHERE mean_exec_time > 100  -- Queries slower than 100ms
ORDER BY total_exec_time DESC
LIMIT 20;

-- Automatic index suggestions (using hypothetical indexes)
-- pg_qualstats extension tracks WHERE clause columns
SELECT * FROM pg_qualstats_indexes;

-- Reindex to fix bloated indexes
REINDEX INDEX CONCURRENTLY idx_orders_user_status_date;
```

### Index Size Impact

```
Table: orders (10 million rows, 2 GB)

Index                              │ Size   │ Impact
───────────────────────────────────┼────────┼────────────────────
PRIMARY KEY (id)                   │ 214 MB │ Auto-created
idx_user_id                        │ 195 MB │ +10% storage
idx_user_status_date (composite)   │ 320 MB │ +16% storage
idx_amount                         │ 195 MB │ +10% storage
idx_all_columns (over-indexed!)    │ 850 MB │ +42% storage

Total table: 2 GB
Total indexes: 1.7 GB (85% of table size!)

More indexes = Faster reads BUT Slower writes
Every INSERT/UPDATE must update ALL indexes
```

---

## Real-World Example

### Shopify — Index Strategy for E-Commerce

```
┌─────────────────────────────────────────────────────────────────┐
│             Shopify Query Patterns & Index Strategy               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Pattern 1: "Get orders for a shop in last 7 days"              │
│  Query: WHERE shop_id = ? AND created_at > ? ORDER BY created_at│
│  Index: (shop_id, created_at DESC)                              │
│                                                                   │
│  Pattern 2: "Find product by SKU in a shop"                     │
│  Query: WHERE shop_id = ? AND sku = ?                           │
│  Index: (shop_id, sku) — UNIQUE                                 │
│                                                                   │
│  Pattern 3: "Count pending orders"                              │
│  Query: WHERE shop_id = ? AND status = 'pending'                │
│  Index: (shop_id, status) WHERE status = 'pending'  ← PARTIAL  │
│                                                                   │
│  Key Insight: Almost every query starts with shop_id            │
│  → shop_id is the leftmost column in every composite index      │
│  → Queries for other shops never touch your shop's index pages  │
│                                                                   │
│  Scale: 100+ million shops, billions of orders                   │
│  Without proper indexes: 10-30 second queries                    │
│  With proper indexes: 1-5 ms queries                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| No indexes at all | Full table scans on every query | Add indexes for WHERE, JOIN, ORDER BY columns |
| Too many indexes | Every INSERT updates all indexes (slow writes) | Only index columns you actually query |
| Wrong column order in composite index | Index can't be used if leftmost column is missing | Put most-filtered column first |
| Functions on indexed columns | `WHERE UPPER(email) = 'X'` bypasses index | Create expression index: `CREATE INDEX ON users(UPPER(email))` |
| Not using EXPLAIN ANALYZE | Guessing instead of measuring | Always check the query plan |
| Indexing low-cardinality columns | Boolean or status columns have few values | Use partial indexes instead |
| Never removing unused indexes | Wastes storage, slows writes | Monitor `pg_stat_user_indexes`, drop unused |
| Not using CONCURRENTLY | `CREATE INDEX` locks the table | Always use `CREATE INDEX CONCURRENTLY` in production |

---

## When to Use / When NOT to Use

### ✅ Create an Index When:
- Column is in **WHERE clauses** frequently
- Column is used in **JOIN conditions**
- Column is used in **ORDER BY** (avoids sort step)
- Table is **large** (>100K rows) and queries return few rows
- Query is **slow** and EXPLAIN shows "Seq Scan"
- Column has **high cardinality** (many distinct values)

### ❌ Do NOT Create an Index When:
- Table is **small** (<10K rows — full scan is fine)
- Column has **low cardinality** (boolean, status with 3 values)
- Table is **write-heavy** with few reads (indexes slow writes)
- You're selecting **most of the table** (index won't help)
- Column is **rarely queried** (wasted space)
- Index already exists that **covers** this column (redundant)

---

## Key Takeaways

1. **Indexes trade write speed for read speed** — every INSERT/UPDATE is slower, but SELECT is orders of magnitude faster.
2. **B-Tree** is the default and handles 90% of cases (equality, range, sorting). Use specialized indexes (GIN, GiST, BRIN) for specific data types.
3. **Composite indexes** must be used left-to-right — `(a, b, c)` helps queries on `a`, `a+b`, and `a+b+c`, but NOT `b` or `c` alone.
4. **EXPLAIN ANALYZE** is your best friend — never guess whether an index is being used; always check the query plan.
5. **Covering indexes** (`INCLUDE` clause) eliminate table lookups entirely — the fastest possible query path.
6. **Partial indexes** are powerful for common filters — index only the rows you care about (smaller index = faster).
7. **Monitor unused indexes** — they waste storage and slow writes. Remove them. A good rule: every index should justify its existence with measurable query speedups.

---

## What's Next?

Next, we'll explore **Database Replication — Master-Slave, Master-Master** (Chapter 9.10), where you'll learn how databases create copies of themselves for high availability, read scaling, and disaster recovery.
