# Relational Databases (SQL) — PostgreSQL, MySQL

> **What you'll learn**: How relational databases organize data into tables with rows and columns, enforce relationships, guarantee data integrity, and why they've been the backbone of applications for 50+ years.

---

## Real-Life Analogy

Imagine a **perfectly organized library**. Every book has a card in the catalog, organized by author, title, and genre. Each book sits on a specific shelf, in a specific row. If you want to find all books by "J.K. Rowling," the librarian doesn't scan every book — she goes straight to the author index, looks up the shelf number, and hands you the exact books.

Now imagine the library has rules:
- Every book MUST have a unique ID number (no duplicates)
- Every book MUST belong to an author who exists in the "Authors" catalog
- If you borrow 3 books, either ALL 3 are marked as borrowed, or NONE are (no half-updates)

This is exactly how a **relational database** works — organized, indexed, and rule-enforced.

---

## Core Concept Explained Step-by-Step

### What is a Relational Database?

A **relational database** stores data in **tables** (also called **relations**). Each table has:
- **Columns** (fields) — define what data is stored (name, email, age)
- **Rows** (records/tuples) — each row is one instance of data (one user, one order)

```
┌─────────────────────────────────────────────────────┐
│                    USERS TABLE                        │
├────────┬──────────────┬───────────────┬─────────────┤
│ id     │ name         │ email         │ age         │
├────────┼──────────────┼───────────────┼─────────────┤
│ 1      │ Alice        │ alice@mail.co │ 28          │
│ 2      │ Bob          │ bob@mail.co   │ 35          │
│ 3      │ Charlie      │ char@mail.co  │ 22          │
└────────┴──────────────┴───────────────┴─────────────┘
```

### Tables, Relationships, and Keys

The magic of relational databases is **relationships between tables**:

```
USERS TABLE                          ORDERS TABLE
┌────┬─────────┐                    ┌──────────┬─────────┬────────┬───────┐
│ id │ name    │                    │ order_id │ user_id │ item   │ price │
├────┼─────────┤                    ├──────────┼─────────┼────────┼───────┤
│ 1  │ Alice   │◄───────────────────│ 101      │ 1       │ Laptop │ 999   │
│ 2  │ Bob     │◄──────────┐       │ 102      │ 1       │ Mouse  │ 29    │
│ 3  │ Charlie │           └───────│ 103      │ 2       │ Phone  │ 699   │
└────┴─────────┘                    └──────────┴─────────┴────────┴───────┘
         ▲                                        │
         │          PRIMARY KEY ◄─────────────────┘
         │                         FOREIGN KEY
         └── PRIMARY KEY
```

**Key types:**
- **Primary Key (PK)** — Uniquely identifies each row (like your Aadhaar/SSN number)
- **Foreign Key (FK)** — Points to a primary key in another table (creates the relationship)
- **Unique Key** — Ensures no duplicates in a column (like email)
- **Composite Key** — Two or more columns together form a unique identifier

### Types of Relationships

```
ONE-TO-ONE                    ONE-TO-MANY                    MANY-TO-MANY
┌──────┐     ┌──────────┐   ┌──────┐     ┌──────────┐    ┌──────────┐     ┌───────────────┐     ┌──────────┐
│ User │────▶│ Profile  │   │ User │────▶│ Order 1  │    │ Student  │────▶│ Enrollment    │◀────│ Course   │
│      │     │          │   │      │────▶│ Order 2  │    │          │     │ (Join Table)  │     │          │
└──────┘     └──────────┘   │      │────▶│ Order 3  │    └──────────┘     └───────────────┘     └──────────┘
                             └──────┘     └──────────┘
```

### SQL — The Language of Relational Databases

**SQL (Structured Query Language)** is how you talk to a relational database:

```sql
-- CREATE a table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- INSERT data
INSERT INTO users (name, email) VALUES ('Alice', 'alice@mail.co');

-- SELECT (read) data
SELECT name, email FROM users WHERE age > 25;

-- UPDATE data
UPDATE users SET name = 'Alice Smith' WHERE id = 1;

-- DELETE data
DELETE FROM users WHERE id = 3;

-- JOIN tables (the power of relational DBs!)
SELECT u.name, o.item, o.price
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.name = 'Alice';
```

### Normalization — Reducing Data Duplication

**Normalization** organizes data to eliminate redundancy:

```
❌ BAD (Denormalized — data repeated):
┌────┬───────┬─────────────┬──────────────────┐
│ id │ name  │ order_item  │ shipping_address │
├────┼───────┼─────────────┼──────────────────┤
│ 1  │ Alice │ Laptop      │ 123 Main St      │
│ 1  │ Alice │ Mouse       │ 123 Main St      │  ← "Alice" & address repeated!
│ 1  │ Alice │ Keyboard    │ 123 Main St      │  ← 3x duplication
└────┴───────┴─────────────┴──────────────────┘

✅ GOOD (Normalized — no repetition):
USERS:  id=1, name="Alice", address="123 Main St"
ORDERS: order_id=101, user_id=1, item="Laptop"
        order_id=102, user_id=1, item="Mouse"
        order_id=103, user_id=1, item="Keyboard"
```

**Normal forms (simplified):**
- **1NF**: Each cell has one value, each row is unique
- **2NF**: 1NF + every non-key column depends on the ENTIRE primary key
- **3NF**: 2NF + no transitive dependencies (no column depends on another non-key column)

---

## How It Works Internally

### Storage Engine Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     CLIENT APPLICATION                        │
└─────────────────────────┬───────────────────────────────────┘
                          │ SQL Query
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    QUERY PROCESSOR                            │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │   Parser    │─▶│  Optimizer   │─▶│ Execution Engine  │  │
│  │ (SQL→Tree)  │  │ (Best Plan)  │  │ (Run the Plan)    │  │
│  └─────────────┘  └──────────────┘  └───────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    STORAGE ENGINE                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Buffer Pool  │  │   WAL Log    │  │  B-Tree Index    │  │
│  │ (In-Memory   │  │(Write-Ahead  │  │ (Fast Lookups)   │  │
│  │  Cache)      │  │  Log)        │  │                  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                       DISK STORAGE                            │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│  │  Data     │  │  Index    │  │  WAL      │               │
│  │  Files    │  │  Files    │  │  Files    │               │
│  └───────────┘  └───────────┘  └───────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### How a Query Executes

When you run `SELECT * FROM users WHERE email = 'alice@mail.co'`:

1. **Parser**: Converts SQL text into an Abstract Syntax Tree (AST)
2. **Optimizer**: Decides the best execution plan
   - "Should I scan the whole table or use the email index?"
   - Compares cost estimates for different approaches
3. **Executor**: Runs the chosen plan
4. **Buffer Pool**: Checks if data pages are already in memory (cache hit)
5. **Disk I/O**: Fetches pages from disk if not in memory

### Write-Ahead Logging (WAL)

Every write goes to the **WAL log FIRST** before modifying actual data:

```
Step 1: Write to WAL (sequential, fast)
Step 2: Acknowledge to client ("Write successful!")
Step 3: Eventually flush to actual data files (background)

If crash happens between Step 2 and Step 3:
→ On restart, replay WAL to recover lost writes
→ ZERO DATA LOSS guaranteed!
```

### B-Tree Index Structure

Most relational databases use **B-Trees** for indexes:

```
                        ┌───────────────┐
                        │   [30, 60]    │         ← Root Node
                        └──┬─────┬──┬──┘
                           │     │  │
              ┌────────────┘     │  └────────────┐
              ▼                  ▼                ▼
       ┌───────────┐     ┌───────────┐    ┌───────────┐
       │ [10, 20]  │     │ [40, 50]  │    │ [70, 80]  │   ← Internal
       └─┬───┬──┬──┘     └─┬───┬──┬──┘    └─┬───┬──┬──┘
         │   │  │           │   │  │          │   │  │
         ▼   ▼  ▼           ▼   ▼  ▼          ▼   ▼  ▼
       [5,8][12,15][22,28][35,38][42,48][55,58][65,68][75,78][85,90]
                                                              ← Leaf Nodes
                                                              (Actual data pointers)
```

**Lookup time**: O(log n) — finding 1 record among 1 billion takes ~30 comparisons!

### MVCC (Multi-Version Concurrency Control)

PostgreSQL uses **MVCC** so readers never block writers:

```
Transaction A (READ):         Transaction B (WRITE):
┌─────────────────────┐       ┌─────────────────────┐
│ Sees snapshot at    │       │ Creates NEW version  │
│ time T1             │       │ of the row           │
│                     │       │                      │
│ Old version: ───────┼───────┤ New version:         │
│ name = "Alice"      │       │ name = "Alice Smith" │
└─────────────────────┘       └─────────────────────┘

Both transactions proceed WITHOUT blocking each other!
```

---

## Code Examples

### Python (using psycopg2 with PostgreSQL)

```python
import psycopg2
from psycopg2.extras import RealDictCursor

# Connect to PostgreSQL
conn = psycopg2.connect(
    host="localhost",
    database="myapp",
    user="admin",
    password="secret"
)

# Use RealDictCursor for dict-like row access
cursor = conn.cursor(cursor_factory=RealDictCursor)

# Create table
cursor.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        email VARCHAR(255) UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT NOW()
    )
""")

# Insert with parameterized query (prevents SQL injection!)
cursor.execute(
    "INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id",
    ("Alice", "alice@example.com")
)
user_id = cursor.fetchone()["id"]
print(f"Created user with ID: {user_id}")

# Query with JOIN
cursor.execute("""
    SELECT u.name, o.item, o.price
    FROM users u
    JOIN orders o ON u.id = o.user_id
    WHERE u.id = %s
    ORDER BY o.created_at DESC
""", (user_id,))

orders = cursor.fetchall()
for order in orders:
    print(f"{order['name']} bought {order['item']} for ${order['price']}")

# Transaction example — transfer money atomically
try:
    cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
    cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2")
    conn.commit()  # Both succeed or neither does
except Exception as e:
    conn.rollback()  # Undo everything on failure
    print(f"Transaction failed: {e}")
finally:
    cursor.close()
    conn.close()
```

### Java (using JDBC with PostgreSQL)

```java
import java.sql.*;

public class RelationalDBExample {
    public static void main(String[] args) {
        String url = "jdbc:postgresql://localhost:5432/myapp";
        String user = "admin";
        String password = "secret";

        try (Connection conn = DriverManager.getConnection(url, user, password)) {
            // Disable auto-commit for transaction control
            conn.setAutoCommit(false);

            // Insert with PreparedStatement (prevents SQL injection!)
            String insertSQL = "INSERT INTO users (name, email) VALUES (?, ?) RETURNING id";
            try (PreparedStatement pstmt = conn.prepareStatement(insertSQL)) {
                pstmt.setString(1, "Bob");
                pstmt.setString(2, "bob@example.com");
                ResultSet rs = pstmt.executeQuery();
                if (rs.next()) {
                    System.out.println("Created user ID: " + rs.getInt("id"));
                }
            }

            // Query with JOIN
            String query = """
                SELECT u.name, o.item, o.price
                FROM users u
                JOIN orders o ON u.id = o.user_id
                WHERE u.name = ?
            """;
            try (PreparedStatement pstmt = conn.prepareStatement(query)) {
                pstmt.setString(1, "Bob");
                ResultSet rs = pstmt.executeQuery();
                while (rs.next()) {
                    System.out.printf("%s bought %s for $%.2f%n",
                        rs.getString("name"),
                        rs.getString("item"),
                        rs.getDouble("price"));
                }
            }

            conn.commit(); // Commit transaction
        } catch (SQLException e) {
            System.err.println("Database error: " + e.getMessage());
        }
    }
}
```

---

## Infrastructure Examples

### PostgreSQL Docker Setup

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    deploy:
      resources:
        limits:
          memory: 2G

volumes:
  pgdata:
```

### PostgreSQL Configuration (postgresql.conf) for Production

```ini
# Memory Settings
shared_buffers = 4GB              # 25% of total RAM
effective_cache_size = 12GB       # 75% of total RAM
work_mem = 256MB                  # Per-query sort memory
maintenance_work_mem = 1GB        # For VACUUM, CREATE INDEX

# WAL Settings
wal_level = replica               # Enable replication
max_wal_senders = 5               # Number of replication slots
wal_buffers = 64MB

# Connection Settings
max_connections = 200
```

### AWS RDS Setup (Terraform)

```hcl
resource "aws_db_instance" "main" {
  identifier           = "myapp-postgres"
  engine               = "postgres"
  engine_version       = "16.1"
  instance_class       = "db.r6g.xlarge"
  allocated_storage    = 100
  max_allocated_storage = 500  # Auto-scaling storage

  db_name  = "myapp"
  username = "admin"
  password = var.db_password

  multi_az               = true   # High availability
  backup_retention_period = 7
  storage_encrypted      = true

  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
}
```

---

## PostgreSQL vs MySQL — Quick Comparison

| Feature | PostgreSQL | MySQL |
|---------|-----------|-------|
| **ACID Compliance** | Full | Full (InnoDB) |
| **JSON Support** | Excellent (JSONB) | Good (JSON) |
| **Concurrency** | MVCC (no read locks) | MVCC (InnoDB) |
| **Extensions** | 100+ (PostGIS, pg_trgm) | Limited |
| **Complex Queries** | Excellent (CTEs, Window Functions) | Good |
| **Replication** | Streaming + Logical | Binary Log |
| **Best For** | Complex apps, analytics | Simple web apps, read-heavy |
| **Used By** | Apple, Instagram, Uber | Facebook, Twitter, Airbnb |

---

## Real-World Example

### Instagram (PostgreSQL at Scale)

Instagram serves 2+ billion monthly active users using PostgreSQL:

```
┌─────────────────────────────────────────────────────────┐
│                   Instagram Architecture                  │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  App Server ──▶ PgBouncer (Connection Pool)             │
│                      │                                   │
│                      ▼                                   │
│              ┌───────────────┐                           │
│              │  Primary DB   │──▶ WAL Streaming          │
│              │  (Writes)     │                           │
│              └───────┬───────┘                           │
│                      │                                   │
│         ┌────────────┼────────────┐                     │
│         ▼            ▼            ▼                     │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐            │
│  │ Replica 1 │ │ Replica 2 │ │ Replica 3 │            │
│  │ (Reads)   │ │ (Reads)   │ │ (Reads)   │            │
│  └───────────┘ └───────────┘ └───────────┘            │
│                                                          │
│  Sharding: Users split across 64 PostgreSQL shards      │
│  Each shard: 1 Primary + 3 Replicas                     │
│  Total: 256 PostgreSQL instances                         │
└─────────────────────────────────────────────────────────┘
```

**Key decisions:**
- **PgBouncer** for connection pooling (reduces PostgreSQL connection overhead)
- **Sharding by user_id** to distribute load
- **Read replicas** handle 90% of traffic (reads)
- **Partitioning** large tables by date (older photos → cold storage)

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Not using indexes | Full table scans on every query | Add indexes on frequently queried columns |
| Too many indexes | Slows down writes (every INSERT updates all indexes) | Only index what you actually query |
| Not using parameterized queries | SQL injection vulnerability | Always use `?` or `%s` placeholders |
| Opening too many connections | Each connection uses ~10MB RAM | Use connection pooling (PgBouncer) |
| Over-normalization | Too many JOINs = slow queries | Denormalize read-heavy data strategically |
| Not using transactions | Inconsistent data on failure | Wrap related operations in BEGIN/COMMIT |
| Ignoring EXPLAIN ANALYZE | Blind performance issues | Always analyze slow query plans |
| Storing large blobs in DB | Bloats database, slows backups | Use object storage (S3) + store URL in DB |

---

## When to Use / When NOT to Use

### ✅ Use Relational Databases When:
- Data has clear relationships (users → orders → products)
- You need **strong consistency** (banking, inventory)
- You need complex queries with JOINs, aggregations
- ACID transactions are critical
- Data structure is well-defined and won't change often
- You need to enforce data integrity (constraints, foreign keys)

### ❌ Do NOT Use When:
- Schema changes frequently (use Document DB)
- Data is hierarchical/graph-like (use Graph DB)
- You need extreme write throughput at scale (use Cassandra/DynamoDB)
- Data is unstructured (logs, sensor data → use Time-Series/NoSQL)
- Horizontal scaling beyond a few TB is required (consider NewSQL)
- Simple key-value lookups only (use Redis/DynamoDB)

---

## Key Takeaways

1. **Relational databases** store data in tables with defined relationships, enforced by primary and foreign keys.
2. **SQL** is the universal language for querying relational data — learn JOINs, subqueries, and window functions.
3. **ACID transactions** guarantee that operations are atomic, consistent, isolated, and durable — critical for financial and inventory systems.
4. **Indexes** use B-Tree structures to make lookups O(log n) instead of O(n) — the single biggest performance lever.
5. **PostgreSQL** is the most feature-rich open-source RDBMS — choose it for complex apps; MySQL for simpler, read-heavy workloads.
6. **Connection pooling** (PgBouncer, HikariCP) is essential in production — never let your app open unlimited connections.
7. **Normalization** reduces duplication but too much hurts read performance — find the right balance for your workload.

---

## What's Next?

Next, we'll explore **NoSQL Databases — Types & When to Use What** (Chapter 9.2), where you'll learn about the alternatives to relational databases and when they make sense over traditional SQL databases.
