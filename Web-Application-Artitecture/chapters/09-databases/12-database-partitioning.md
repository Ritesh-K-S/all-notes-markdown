# Database Partitioning (Horizontal & Vertical)

> **What you'll learn**: How partitioning splits large tables within a single database for better performance and manageability, the difference between horizontal and vertical partitioning, partitioning vs sharding, and how to implement time-based partitioning for massive tables.

---

## Real-Life Analogy

Imagine a **massive filing cabinet** with 10 million files. Finding anything takes forever because you're searching through one enormous pile.

**Horizontal partitioning**: Split by year — 2024 files in drawer 1, 2023 files in drawer 2, etc. Now you only search one drawer! (Same columns, different rows per partition)

**Vertical partitioning**: Split by information type — one drawer has names+addresses (lightweight, frequently accessed), another has medical records (heavy, rarely accessed). Now quick lookups don't load unnecessary data! (Same rows, different columns per partition)

Key difference from sharding: All drawers are **in the same building** (same database server). Sharding puts drawers in **different buildings** (different servers).

---

## Core Concept Explained Step-by-Step

### Partitioning vs Sharding

```
PARTITIONING:                            SHARDING:
┌─────────────────────────────────┐     ┌────────────┐  ┌────────────┐
│     SINGLE DATABASE SERVER       │     │  Server 1  │  │  Server 2  │
│                                  │     │  Shard A   │  │  Shard B   │
│  ┌──────────┐ ┌──────────┐     │     │            │  │            │
│  │Partition 1│ │Partition 2│     │     └────────────┘  └────────────┘
│  │(Jan-Jun)  │ │(Jul-Dec)  │     │     
│  └──────────┘ └──────────┘     │     Different SERVERS
│  ┌──────────┐ ┌──────────┐     │     (network boundary between them)
│  │Partition 3│ │Partition 4│     │     
│  │(2023)     │ │(2022)     │     │     ✅ Scales writes + storage
│  └──────────┘ └──────────┘     │     ❌ Complex (distributed system)
│                                  │     
│  Same SERVER, same database     │     
│  ✅ Query optimizer handles it  │     
│  ✅ JOINs work normally         │     
│  ✅ Transactions work normally   │     
│  ❌ Limited by single server    │     
└─────────────────────────────────┘     
```

### Horizontal Partitioning (Row-Based)

Split table by **rows** — each partition contains a subset of rows based on a partition key:

```
ORIGINAL TABLE: orders (100 million rows)
┌─────────┬───────────┬─────────────────┬────────┐
│ id      │ user_id   │ created_at      │ amount │  ← ALL rows in one table
└─────────┴───────────┴─────────────────┴────────┘   (slow scans, huge indexes)

HORIZONTALLY PARTITIONED: (by month)
┌─────────────────────────────────────────────────────────────────┐
│                    orders (parent table)                          │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ orders_2024_01  │ orders_2024_02  │ orders_2024_03  │ ...       │
│ (3M rows)       │ (3.2M rows)     │ (2.8M rows)     │           │
│ Jan 2024 only   │ Feb 2024 only   │ Mar 2024 only   │           │
└─────────────────┴─────────────────┴─────────────────────────────┘

Query: SELECT * FROM orders WHERE created_at BETWEEN '2024-03-01' AND '2024-03-31'
→ Only scans orders_2024_03 partition! (3M rows instead of 100M)
→ Called "partition pruning" — database skips irrelevant partitions
```

### Vertical Partitioning (Column-Based)

Split table by **columns** — separate frequently accessed columns from rarely accessed ones:

```
ORIGINAL TABLE: users
┌─────┬──────┬───────┬──────────────────────────────────────────┐
│ id  │ name │ email │ bio (TEXT, avg 5KB) │ avatar_url │ prefs │
└─────┴──────┴───────┴──────────────────────────────────────────┘
Problem: Loading name/email also loads 5KB bio every time!

VERTICALLY PARTITIONED:
┌─────────────────────────────┐    ┌───────────────────────────────────┐
│ users_core (hot data)       │    │ users_extended (cold data)        │
│ ┌─────┬──────┬───────┐     │    │ ┌─────┬──────────┬────────┬────┐ │
│ │ id  │ name │ email │     │    │ │ id  │ bio      │ avatar │pref│ │
│ │     │      │       │     │    │ │     │ (5KB!)   │        │    │ │
│ └─────┴──────┴───────┘     │    │ └─────┴──────────┴────────┴────┘ │
│ Small rows → fast scans    │    │ Large rows → only loaded when     │
│ Fits in memory/cache       │    │ user views full profile           │
└─────────────────────────────┘    └───────────────────────────────────┘

SELECT name, email FROM users_core WHERE id = 123;  ← Super fast! No bio loaded
SELECT bio FROM users_extended WHERE id = 123;       ← Only when needed
```

### Partition Types (PostgreSQL)

```
1. RANGE PARTITIONING (most common):
────────────────────────────────────
   Partition by date range, numeric range, etc.
   orders_2024_q1: created_at from '2024-01-01' to '2024-04-01'
   orders_2024_q2: created_at from '2024-04-01' to '2024-07-01'
   
   Best for: Time-series data, date-based queries

2. LIST PARTITIONING:
────────────────────────────────────
   Partition by specific values.
   orders_india:    country IN ('IN')
   orders_us:       country IN ('US')
   orders_europe:   country IN ('DE', 'FR', 'GB', 'IT')
   
   Best for: Categorical data, multi-region

3. HASH PARTITIONING:
────────────────────────────────────
   Partition by hash of column value.
   partition_0: hash(user_id) % 4 = 0
   partition_1: hash(user_id) % 4 = 1
   partition_2: hash(user_id) % 4 = 2
   partition_3: hash(user_id) % 4 = 3
   
   Best for: Even distribution when no natural range exists
```

---

## How It Works Internally

### PostgreSQL Partition Pruning

```
Table: orders (partitioned by month)
├── orders_2024_01  (created_at: Jan 2024)
├── orders_2024_02  (created_at: Feb 2024)
├── orders_2024_03  (created_at: Mar 2024)
└── orders_2024_04  (created_at: Apr 2024)

Query: SELECT * FROM orders WHERE created_at = '2024-03-15'

WITHOUT PARTITIONING:
  Seq Scan on orders → scans ALL 100M rows → 10 seconds

WITH PARTITIONING:
  Query Planner sees: created_at = '2024-03-15'
  Partition pruning: Only '2024-03' matches!
  Scans ONLY orders_2024_03 → 3M rows → 0.3 seconds (33x faster!)

EXPLAIN output:
  Append
    → Seq Scan on orders_2024_03  ← Only this partition scanned!
    (all other partitions PRUNED — not even touched)
```

### Partition-Wise Join

```
Query: SELECT o.*, p.name
       FROM orders o JOIN products p ON o.product_id = p.id
       WHERE o.created_at > '2024-03-01'

Without partition-wise join:
  Scan relevant order partitions → JOIN with entire products table

With partition-wise join (if products is also partitioned the same way):
  orders_2024_03 JOIN products_2024_03  ← each partition joined separately
  Much smaller JOINs! Much faster!
```

### Partition Maintenance Operations

```
┌─────────────────────────────────────────────────────────────────┐
│              PARTITION LIFECYCLE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  CREATE new partition (for next month):                          │
│  ─────────────────────────────────────                          │
│  Almost instant, no data movement                               │
│                                                                   │
│  DROP old partition (retention):                                  │
│  ────────────────────────────────                               │
│  DROP TABLE orders_2022_01;  ← INSTANT! (vs DELETE which        │
│                                   generates massive WAL)        │
│                                                                   │
│  DETACH partition (for archival):                                │
│  ────────────────────────────────                               │
│  ALTER TABLE orders DETACH PARTITION orders_2022_01;             │
│  → Table still exists but no longer part of parent              │
│  → Can dump to S3, move to cold storage                         │
│                                                                   │
│  ATTACH partition (restore archived data):                       │
│  ────────────────────────────────────────                       │
│  ALTER TABLE orders ATTACH PARTITION orders_2022_01              │
│  FOR VALUES FROM ('2022-01-01') TO ('2022-02-01');              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python (PostgreSQL Partitioning)

```python
import psycopg2
from datetime import datetime, timedelta

conn = psycopg2.connect("host=localhost dbname=myapp user=admin password=secret")
cursor = conn.cursor()

# Create partitioned table
cursor.execute("""
    CREATE TABLE IF NOT EXISTS orders (
        id BIGSERIAL,
        user_id INTEGER NOT NULL,
        product_name TEXT NOT NULL,
        amount DECIMAL(10,2) NOT NULL,
        status TEXT NOT NULL DEFAULT 'pending',
        created_at TIMESTAMP NOT NULL DEFAULT NOW(),
        PRIMARY KEY (id, created_at)  -- Partition key must be in PK!
    ) PARTITION BY RANGE (created_at)
""")

# Create monthly partitions
def create_monthly_partition(year, month):
    """Create partition for a specific month"""
    start = f"{year}-{month:02d}-01"
    if month == 12:
        end = f"{year+1}-01-01"
    else:
        end = f"{year}-{month+1:02d}-01"
    
    partition_name = f"orders_{year}_{month:02d}"
    cursor.execute(f"""
        CREATE TABLE IF NOT EXISTS {partition_name}
        PARTITION OF orders
        FOR VALUES FROM ('{start}') TO ('{end}')
    """)
    
    # Create indexes on partition (each partition has its own index)
    cursor.execute(f"""
        CREATE INDEX IF NOT EXISTS idx_{partition_name}_user 
        ON {partition_name}(user_id, created_at DESC)
    """)

# Create partitions for 2024
for month in range(1, 13):
    create_monthly_partition(2024, month)

conn.commit()

# Query with partition pruning (only scans March partition)
cursor.execute("""
    SELECT user_id, SUM(amount) as total
    FROM orders
    WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01'
    GROUP BY user_id
    ORDER BY total DESC
    LIMIT 10
""")

# Drop old partition (instant! vs DELETE which generates WAL)
cursor.execute("""
    ALTER TABLE orders DETACH PARTITION orders_2022_01;
    DROP TABLE orders_2022_01;
""")

# Automated partition management (run monthly via cron)
def ensure_future_partitions(months_ahead=3):
    """Ensure partitions exist for the next N months"""
    today = datetime.now()
    for i in range(months_ahead):
        future = today + timedelta(days=30 * (i + 1))
        create_monthly_partition(future.year, future.month)
    conn.commit()

ensure_future_partitions()
```

### Java (JPA with Partitioned Tables)

```java
import jakarta.persistence.*;
import org.springframework.data.jpa.repository.Query;
import org.springframework.scheduling.annotation.Scheduled;
import java.time.LocalDateTime;
import java.time.YearMonth;

@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private Integer userId;
    private String productName;
    private BigDecimal amount;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;  // Partition key
}

// Repository — queries automatically benefit from partition pruning
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // This query prunes to specific partition(s)
    @Query("SELECT o FROM Order o WHERE o.userId = :userId " +
           "AND o.createdAt >= :start AND o.createdAt < :end")
    List<Order> findByUserAndDateRange(Integer userId, 
                                       LocalDateTime start, 
                                       LocalDateTime end);
}

// Partition maintenance service
@Service
public class PartitionMaintenanceService {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    // Run monthly: create next month's partition
    @Scheduled(cron = "0 0 0 25 * *")  // 25th of every month
    public void createNextPartition() {
        YearMonth next = YearMonth.now().plusMonths(1);
        String partName = String.format("orders_%d_%02d", 
            next.getYear(), next.getMonthValue());
        String start = next.atDay(1).toString();
        String end = next.plusMonths(1).atDay(1).toString();
        
        jdbcTemplate.execute(String.format(
            "CREATE TABLE IF NOT EXISTS %s PARTITION OF orders " +
            "FOR VALUES FROM ('%s') TO ('%s')", partName, start, end));
    }

    // Run monthly: drop partitions older than 2 years
    @Scheduled(cron = "0 0 1 1 * *")  // 1st of every month
    public void dropOldPartitions() {
        YearMonth cutoff = YearMonth.now().minusYears(2);
        String partName = String.format("orders_%d_%02d",
            cutoff.getYear(), cutoff.getMonthValue());
        
        jdbcTemplate.execute("ALTER TABLE orders DETACH PARTITION " + partName);
        jdbcTemplate.execute("DROP TABLE " + partName);
    }
}
```

---

## Infrastructure Examples

### PostgreSQL Partitioning Configuration

```sql
-- Enable partition pruning (usually on by default)
SET enable_partition_pruning = on;

-- Check which partitions are scanned
EXPLAIN (ANALYZE, COSTS, VERBOSE)
SELECT * FROM orders WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';

-- Monitor partition sizes
SELECT 
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS size,
    n_live_tup AS row_count
FROM pg_stat_user_tables
WHERE tablename LIKE 'orders_%'
ORDER BY tablename;

-- Auto-create partitions with pg_partman extension
CREATE EXTENSION pg_partman;
SELECT partman.create_parent(
    p_parent_table => 'public.orders',
    p_control => 'created_at',
    p_type => 'native',
    p_interval => '1 month',
    p_premake => 3  -- Create 3 months ahead
);

-- pg_partman handles creation + retention automatically!
SELECT partman.run_maintenance();
```

### MySQL Partitioning

```sql
-- MySQL range partitioning
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT,
    user_id INT NOT NULL,
    amount DECIMAL(10,2),
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (YEAR(created_at) * 100 + MONTH(created_at)) (
    PARTITION p_2024_01 VALUES LESS THAN (202402),
    PARTITION p_2024_02 VALUES LESS THAN (202403),
    PARTITION p_2024_03 VALUES LESS THAN (202404),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Reorganize the catch-all partition when adding new ones
ALTER TABLE orders REORGANIZE PARTITION p_future INTO (
    PARTITION p_2024_04 VALUES LESS THAN (202405),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

---

## Real-World Example

### Shopify — Partitioning for Multi-Tenant Data

```
┌─────────────────────────────────────────────────────────────────┐
│             Shopify's Partitioning Strategy                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Challenge: 1.7M+ merchants, billions of orders                 │
│                                                                   │
│  Strategy: Pod-based architecture with partitioning             │
│                                                                   │
│  ┌─────────────────────────────────────────────────────┐       │
│  │ Per-Pod (each pod = ~5000 shops):                    │       │
│  │                                                      │       │
│  │ orders table:                                        │       │
│  │   Partitioned by shop_id (LIST partitioning)        │       │
│  │   → Each shop's data in its own partition            │       │
│  │   → DROP PARTITION = instant shop data deletion      │       │
│  │   → Migrations per-partition (no global lock)        │       │
│  │                                                      │       │
│  │ events table:                                        │       │
│  │   Partitioned by created_at (RANGE, monthly)        │       │
│  │   → Old months dropped (30-day retention)            │       │
│  │   → Queries always include date range               │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                   │
│  Benefits:                                                       │
│  • Vacuum operates per-partition (not whole table)              │
│  • Index sizes are manageable per partition                      │
│  • DROP partition is instant vs DELETE (no bloat)               │
│  • Easy data lifecycle management                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Partitioning small tables | Overhead without benefit | Only partition tables > 10M rows or > 10GB |
| Queries without partition key | Scans ALL partitions (worse than unpartitioned!) | Ensure WHERE clause includes partition column |
| Too many partitions | Planner overhead, memory for partition metadata | Aim for < 1000 partitions per table |
| Too few partitions | No pruning benefit | Partition so each has 1-10M rows |
| Forgetting partition key in PK/UK | PostgreSQL requires it in unique constraints | Include partition column in PRIMARY KEY |
| Not automating maintenance | Missing partitions for new data | Use pg_partman or cron for creation/cleanup |
| Vertical partitioning with frequent JOINs | Joining split tables negates the benefit | Only split truly cold columns |

---

## When to Use / When NOT to Use

### ✅ Use Horizontal Partitioning When:
- Table is **very large** (>10M rows, >10GB)
- Queries **always filter by a specific column** (date, tenant_id)
- Need to **drop old data efficiently** (retention policies)
- Want to **archive old data** without affecting live queries
- **Index maintenance** on full table is too slow (VACUUM, REINDEX)
- Different partitions need different **storage** (hot SSD vs cold HDD)

### ✅ Use Vertical Partitioning When:
- Table has **wide rows** with rarely-accessed columns (BLOBs, TEXT)
- Some columns are **read frequently** while others are rarely needed
- Want to **reduce I/O** for common queries (only load needed columns)
- **Cache efficiency** — hot columns fit in memory, cold ones on disk

### ❌ Do NOT Partition When:
- Table is **small** (< 1M rows — no benefit, just complexity)
- Queries **don't filter by partition key** (all partitions scanned anyway)
- You need **global unique constraints** across all data
- Already have **good indexes** that serve your query patterns
- Need to **scale writes** (partitioning is on one server — use sharding)

---

## Key Takeaways

1. **Partitioning splits a table into smaller pieces on the SAME server** — unlike sharding which splits across different servers.
2. **Horizontal partitioning** (by rows) is most common — partition by date for time-series data, by tenant for multi-tenant apps.
3. **Partition pruning** is the performance win — the database only scans partitions relevant to your query filter.
4. **DROP PARTITION is instant** — compared to DELETE which generates WAL, creates dead tuples, and requires VACUUM.
5. **Always include the partition key in queries** — without it, the database must scan ALL partitions (full table scan equivalent).
6. **Automate partition lifecycle** — create future partitions ahead of time, drop/archive old ones on schedule.
7. **Partitioning + Sharding together** is common at scale — partition within each shard for additional performance gains.

---

## What's Next?

Next, we'll explore **ACID vs BASE — Consistency Models for Databases** (Chapter 9.13), where you'll learn the fundamental trade-offs between strict consistency and high availability that every distributed database must make.
