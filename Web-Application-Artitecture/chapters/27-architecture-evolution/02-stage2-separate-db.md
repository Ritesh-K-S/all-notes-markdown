# Stage 2: Separate Database — 100 to 1,000 Users

> **What you'll learn**: Why the very first scaling move is pulling the database out to its own server, how this simple change transforms your capacity, and exactly how to do it without downtime.

---

## Real-Life Analogy

Remember our chai stall? Business is growing. You now have 50 customers waiting in line at peak hours. The problem? You're spending most of your time **washing cups and managing supplies** (that's the database work) instead of actually making and serving chai.

**Solution**: You hire a dedicated dishwasher/supply manager who works at a separate counter. Now YOU can focus entirely on making chai (application logic), and they focus entirely on managing cups and ingredients (data storage).

The chai flows faster because each person does ONE job well, even though they now need to shout across the room to communicate (network latency).

---

## Core Concept Explained Step-by-Step

### What Changes?

The database moves from the same machine to its own dedicated server.

```
BEFORE (Stage 1):                    AFTER (Stage 2):
┌─────────────────────┐             ┌──────────────────┐    ┌──────────────────┐
│    SINGLE SERVER    │             │   APP SERVER     │    │   DB SERVER      │
│                     │             │                  │    │                  │
│  ┌───┐ ┌───┐ ┌───┐ │             │  ┌───┐  ┌───┐   │    │  ┌────────────┐  │
│  │App│ │DB │ │Web│ │    ──▶      │  │Web│  │App│   │◄──▶│  │ PostgreSQL │  │
│  └───┘ └───┘ └───┘ │             │  └───┘  └───┘   │    │  └────────────┘  │
│                     │             │                  │    │                  │
│  CPU: shared        │             │  CPU: all for app│    │  CPU: all for DB │
│  RAM: shared        │             │  RAM: all for app│    │  RAM: all for DB │
│  Disk: shared       │             │  Disk: SSD       │    │  Disk: Fast SSD  │
└─────────────────────┘             └──────────────────┘    └──────────────────┘
                                              │                       │
                                              └───── Private Network ─┘
                                                    (1-2 ms latency)
```

### Why Separate the Database FIRST?

This is universally the **first scaling move** because:

| Reason | Explanation |
|--------|-------------|
| **Resource contention** | DB and app fight for the same CPU/RAM. A heavy query starves your API. |
| **Independent scaling** | DB needs lots of RAM (for caching). App needs lots of CPU (for processing). Different hardware profiles. |
| **Independent backups** | Backing up the DB doesn't slow down the app server |
| **Security** | DB is no longer exposed to the internet — only accessible from app server |
| **Specialization** | DB server can use specialized storage (NVMe SSDs, RAID) |

### The Architecture After Separation

```
                    Internet
                       │
                       ▼
              ┌─────────────────┐
              │   App Server    │
              │  (Nginx + App)  │
              │                 │
              │  Public IP:     │
              │  203.0.113.10   │
              │                 │
              │  RAM: 4 GB      │
              │  (for app logic)│
              └────────┬────────┘
                       │
            Private Network (10.0.0.x)
            (NOT accessible from internet)
                       │
              ┌────────▼────────┐
              │   DB Server     │
              │  (PostgreSQL)   │
              │                 │
              │  Private IP:    │
              │  10.0.0.50      │
              │                 │
              │  RAM: 8 GB      │
              │  (for DB cache) │
              │  Disk: NVMe SSD │
              └─────────────────┘
```

### What Changes in Your Code?

Only ONE line changes — the database connection string:

```
BEFORE: DATABASE_URL = "postgresql://user:pass@localhost:5432/myapp"
AFTER:  DATABASE_URL = "postgresql://user:pass@10.0.0.50:5432/myapp"
```

That's it. Your application code stays exactly the same.

---

## How It Works Internally

### Network Communication Between Servers

When the database was on localhost:
- Communication via **Unix sockets** or **loopback** (127.0.0.1)
- Latency: **< 0.1 ms**
- Bandwidth: **unlimited** (memory-to-memory)

When the database is on a separate server:
- Communication via **TCP over private network**
- Latency: **0.5 - 2 ms** (same datacenter)
- Bandwidth: **1-10 Gbps** (typical private network)

```
App Server                                DB Server
┌────────────┐                           ┌────────────┐
│            │  (1) TCP SYN              │            │
│  App sends ├──────────────────────────▶│ PostgreSQL │
│  SQL query │                           │ listens on │
│            │  (2) TCP SYN-ACK          │ port 5432  │
│            │◀──────────────────────────┤            │
│            │                           │            │
│            │  (3) SQL Query sent       │            │
│            ├──────────────────────────▶│ Executes   │
│            │                           │ query      │
│            │  (4) Result rows          │            │
│            │◀──────────────────────────┤ Returns    │
│            │                           │ data       │
└────────────┘                           └────────────┘

         Round-trip: ~1-2 ms (same datacenter)
         Compare to localhost: ~0.05 ms
```

### Why the Extra Latency is Worth It

| Metric | Single Server | Separated |
|--------|--------------|-----------|
| DB query latency | 0.05 ms | 1-2 ms |
| Max concurrent DB connections | 50-100 (shared RAM) | 200-500 (dedicated RAM) |
| App throughput | 200 req/s | 500-1000 req/s |
| DB can cache data in RAM | 1-2 GB | 6-8 GB |
| Backup impact on app | Heavy (slows app) | Zero (separate machine) |

The **20x increase in latency per query** is compensated by **5x improvement in overall throughput** because the systems no longer compete for resources.

### Connection Pooling Becomes Critical

With network overhead, opening a new connection for each query is expensive:

```
Without Connection Pool:
Request → Open TCP → TLS handshake → Authenticate → Query → Close
         (50 ms setup for each request!)

With Connection Pool:
Request → Grab existing connection → Query → Return to pool
         (0 ms setup — connection already exists!)
```

```
┌──────────────────────────────────────────────────────────┐
│                    App Server                              │
│                                                          │
│  Request 1 ──┐                                           │
│  Request 2 ──┤    ┌────────────────────┐                 │
│  Request 3 ──┼───▶│  Connection Pool   │────▶ DB Server  │
│  Request 4 ──┤    │  (10-20 connections│                 │
│  Request 5 ──┘    │   kept alive)      │                 │
│                   └────────────────────┘                 │
│                                                          │
│  50 requests share 10 connections efficiently            │
└──────────────────────────────────────────────────────────┘
```

### Security: Private Network Isolation

```
┌─────────────────────────────────────────────────────────────┐
│                     CLOUD VPC (Virtual Private Cloud)         │
│                                                             │
│  Public Subnet                  Private Subnet              │
│  ┌──────────────┐              ┌──────────────┐             │
│  │  App Server  │──────────────│  DB Server   │             │
│  │  (public IP) │  10.0.0.x    │  (NO public  │             │
│  └──────┬───────┘              │   IP!)       │             │
│         │                      └──────────────┘             │
│         │                                                   │
│  ┌──────▼───────┐                                           │
│  │ Security Grp │  Only allows:                             │
│  │ (Firewall)   │  - Port 80/443 from internet              │
│  └──────────────┘  - Port 5432 ONLY from app server IP      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
         │
    Internet (hackers can't reach DB directly!)
```

---

## Code Examples

### Python: Connecting to Remote Database with Connection Pool

```python
# app.py — Connecting to a SEPARATE database server
from flask import Flask, jsonify
import psycopg2
from psycopg2 import pool

app = Flask(__name__)

# Connection pool to the REMOTE database server
# Key difference: host is now a private IP, not localhost
db_pool = psycopg2.pool.ThreadedConnectionPool(
    minconn=5,          # Keep 5 connections always ready
    maxconn=20,         # Max 20 connections to DB
    host="10.0.0.50",   # Private IP of DB server (NOT localhost!)
    port=5432,
    database="myapp",
    user="appuser",
    password="secret",
    # Connection keepalive to detect dead connections
    keepalives=1,
    keepalives_idle=30,
    keepalives_interval=10,
    keepalives_count=5
)

@app.route("/api/users")
def get_users():
    conn = db_pool.getconn()      # Grab from pool (instant!)
    try:
        cur = conn.cursor()
        cur.execute("SELECT id, name, email FROM users LIMIT 50")
        users = [
            {"id": row[0], "name": row[1], "email": row[2]}
            for row in cur.fetchall()
        ]
        return jsonify(users)
    finally:
        db_pool.putconn(conn)     # Return to pool (don't close!)

@app.route("/health")
def health():
    """Health check endpoint — verify DB is reachable"""
    try:
        conn = db_pool.getconn()
        cur = conn.cursor()
        cur.execute("SELECT 1")
        db_pool.putconn(conn)
        return jsonify({"status": "healthy", "db": "connected"})
    except Exception as e:
        return jsonify({"status": "unhealthy", "error": str(e)}), 503
```

### Java: Connecting to Remote Database with HikariCP Pool

```java
// DatabaseConfig.java — Connection pool to separate DB server
@Configuration
public class DatabaseConfig {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();

        // Key: pointing to the PRIVATE IP of the separate DB server
        config.setJdbcUrl("jdbc:postgresql://10.0.0.50:5432/myapp");
        config.setUsername("appuser");
        config.setPassword("secret");

        // Connection pool settings
        config.setMinimumIdle(5);           // Keep 5 ready
        config.setMaximumPoolSize(20);      // Max 20 connections
        config.setConnectionTimeout(5000);  // Fail fast if DB unreachable
        config.setIdleTimeout(300000);      // Close idle connections after 5 min
        config.setMaxLifetime(600000);      // Refresh connections every 10 min

        // Keepalive: detect dead connections
        config.setKeepaliveTime(30000);

        return new HikariDataSource(config);
    }
}

// UserController.java — Same app code, different DB location
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @GetMapping
    public List<Map<String, Object>> getUsers() {
        // This query now travels over the network to 10.0.0.50
        // But connection pooling makes it fast
        return jdbcTemplate.queryForList(
            "SELECT id, name, email FROM users LIMIT 50"
        );
    }
}
```

---

## Infrastructure Example: Setting Up Separate DB on AWS

### Terraform: Provision App Server + RDS Database

```hcl
# main.tf — Separate app server and managed database

# VPC with public + private subnets
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# App server in public subnet (accessible from internet)
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"  # Ubuntu 22.04
  instance_type = "t3.medium"               # 2 CPU, 4 GB RAM
  subnet_id     = aws_subnet.public.id

  tags = { Name = "app-server" }
}

# Database in private subnet (NOT accessible from internet)
resource "aws_db_instance" "postgres" {
  engine         = "postgres"
  engine_version = "15"
  instance_class = "db.t3.medium"   # 2 CPU, 4 GB RAM (dedicated to DB)
  allocated_storage = 100            # 100 GB SSD

  db_name  = "myapp"
  username = "appuser"
  password = var.db_password

  # CRITICAL: Private subnet only!
  db_subnet_group_name   = aws_db_subnet_group.private.name
  vpc_security_group_ids = [aws_security_group.db.id]
  publicly_accessible    = false    # No public IP!

  # Backups (automated, no impact on app server)
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
}

# Security group: Only allow app server to connect
resource "aws_security_group" "db" {
  ingress {
    from_port       = 5432
    to_port         = 5432
    security_groups = [aws_security_group.app.id]  # Only from app!
  }
}
```

### Migration Script: Moving Data Without Downtime

```bash
#!/bin/bash
# migrate-db.sh — Move database to a separate server

# Step 1: Set up the new database server
echo "Creating database on new server..."
PGPASSWORD=secret psql -h 10.0.0.50 -U appuser -c "CREATE DATABASE myapp;"

# Step 2: Dump data from old server (local)
echo "Dumping data from local database..."
pg_dump -h localhost -U appuser myapp > /tmp/myapp_dump.sql

# Step 3: Restore to new server
echo "Restoring data to new server..."
PGPASSWORD=secret psql -h 10.0.0.50 -U appuser myapp < /tmp/myapp_dump.sql

# Step 4: Update app configuration (change DATABASE_URL)
echo "Updating environment variable..."
# Change from localhost to 10.0.0.50 in your .env or config

# Step 5: Restart app to use new connection
echo "Restarting application..."
sudo systemctl restart myapp

# Step 6: Verify
echo "Verifying..."
curl -s http://localhost/health | jq .
```

---

## Real-World Example

### How Companies Handle This Transition

**Amazon RDS (Relational Database Service)** exists precisely because this is the most common scaling move:
- You get a managed PostgreSQL/MySQL server on its own dedicated hardware
- Automated backups, patching, and failover — you don't manage the OS
- Used by Netflix, Airbnb, Pinterest, and thousands of startups

**Heroku's architecture** does this automatically:
- Your app runs on "dynos" (containers)
- Your database runs on a separate managed PostgreSQL instance
- They separated them from day one so you never have to think about it

**Instagram's first scale move (2010)**:
- Started everything on one EC2 instance
- First change: moved PostgreSQL to a dedicated EC2 instance with more RAM
- This alone gave them 3x more capacity (from ~25K to ~75K users)

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **No connection pooling** | Each request opens a new TCP connection (50ms overhead) | Use pgBouncer, HikariCP, or built-in pools |
| **DB publicly accessible** | Hackers can try to connect directly | Put DB in private subnet, no public IP |
| **Not monitoring network** | Network issues silently degrade performance | Monitor latency between app and DB |
| **Too many connections** | PostgreSQL struggles above 200 connections | Use connection pooler (pgBouncer) |
| **No connection timeout** | App hangs forever if DB is down | Set `connect_timeout=5` and `statement_timeout=30s` |
| **Large result sets** | Sending 1M rows across network is slow | Use pagination (LIMIT/OFFSET) |
| **Not using private network** | Going through public internet adds latency + security risk | Use VPC private subnets |

---

## When to Use / When NOT to Use

### ✅ Separate the Database When:
- Your server's RAM is mostly used by the database
- Database backup (pg_dump) slows down your application
- You want the ability to upgrade app and DB independently
- You need better security isolation
- CPU is maxed because app and DB compete

### ❌ Keep Single Server When:
- You have fewer than 50-100 active users
- Your budget is extremely tight (< $10/month)
- The extra 1-2ms latency would be unacceptable (rare)
- It's a personal/hobby project with no reliability requirements
- The operational complexity isn't worth it yet

---

## Key Takeaways

1. **Separating the database is the #1 first scaling move** — universally recommended
2. **Only ONE line of code changes** — the connection string (localhost → private IP)
3. **You trade 1-2ms latency for massive resource gains** — each server gets dedicated CPU/RAM
4. **Connection pooling becomes mandatory** — opening TCP connections per request is too expensive
5. **Always use a private network** — the database should NEVER have a public IP
6. **Managed databases (RDS, Cloud SQL) handle this automatically** — backups, patching, monitoring
7. **This single change can 3-5x your capacity** — the most impactful change per effort invested

---

## What's Next?

Your app server can now serve more requests because it has all the RAM and CPU to itself. But what happens when even that one app server isn't enough? When you need **high availability** (no downtime if one server crashes)?

That's when you add a **Load Balancer and Multiple App Servers** — Stage 3.

Next: [03-stage3-load-balancer.md](./03-stage3-load-balancer.md)
