# Separate Servers for Frontend, Backend & Database

> **What you'll learn**: Why splitting your application across multiple servers is the first real scaling step, how to architect it, and how this separation gives you independent scaling, better security, and higher reliability.

---

## Real-Life Analogy

Remember our tiny restaurant with everything in one room? Now business is booming, so you **expand**:

- You move the **dining area** to a beautiful front building with lots of tables → **Frontend Server** (or CDN)
- The **kitchen** moves to a separate building behind it — focused purely on cooking → **Backend/Application Server**
- The **warehouse/cold storage** moves to a secure building with heavy locks → **Database Server**

Now:
- The dining area can be renovated without shutting down the kitchen.
- If the kitchen gets too busy, you can add more cooks without touching the dining room.
- The warehouse has its own security — even if someone breaks into the dining area, they can't reach the ingredients.
- Each building can be sized independently — big dining room, medium kitchen, huge warehouse.

That's the **separate servers** model — each concern runs on its own machine, communicating over the network.

---

## Core Concept Explained Step-by-Step

### The Separation

```
BEFORE (Single Server — Chapter 5.1):
┌─────────────────────────────────┐
│         ONE SERVER              │
│  ┌───────┐ ┌─────┐ ┌────────┐ │
│  │ Nginx │ │ App │ │  DB    │ │
│  └───────┘ └─────┘ └────────┘ │
└─────────────────────────────────┘

AFTER (Separate Servers):
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Frontend   │     │   Backend    │     │   Database   │
│   Server     │────▶│   Server     │────▶│   Server     │
│              │     │              │     │              │
│  Nginx/CDN   │     │  App Code    │     │  PostgreSQL  │
│  Static HTML │     │  API Logic   │     │  Data Files  │
│  CSS, JS     │     │  Auth, etc.  │     │  Indexes     │
└──────────────┘     └──────────────┘     └──────────────┘
   IP: 10.0.1.1        IP: 10.0.1.2        IP: 10.0.1.3
```

### Why Separate?

```
PROBLEM with single server:

┌────────────────────────────────────────────┐
│  Single Server: 4 GB RAM                   │
│                                            │
│  App needs 2 GB (spike!) ──┐               │
│  DB needs 2 GB (big query)──┼── CONFLICT!  │
│  Nginx needs 200 MB ────────┘              │
│                                            │
│  Total needed: 4.2 GB > 4 GB available     │
│  Result: OOM killer → process dies         │
└────────────────────────────────────────────┘

SOLUTION with separate servers:

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Frontend: 1 GB  │  │ Backend: 4 GB   │  │ Database: 8 GB  │
│                 │  │                 │  │                 │
│ Just Nginx +    │  │ All RAM for     │  │ All RAM for     │
│ static files    │  │ app processing  │  │ data caching    │
│                 │  │                 │  │ (shared_buffers) │
└─────────────────┘  └─────────────────┘  └─────────────────┘

Each component gets DEDICATED resources!
```

### The Three Common Separation Models

```
MODEL 1: Frontend on CDN + Backend + Database

┌────────────┐      ┌────────────────┐      ┌────────────────┐
│    CDN     │      │   Backend      │      │   Database     │
│ (Cloudflare│      │   Server       │      │   Server       │
│  Vercel,   │      │                │      │                │
│  Netlify)  │      │  REST API      │      │  PostgreSQL    │
│            │─────▶│  Business Logic│─────▶│  or MongoDB    │
│ Static HTML│      │  Auth          │      │                │
│ CSS, JS    │      │                │      │                │
└────────────┘      └────────────────┘      └────────────────┘
 Cost: Free-$20       Cost: $20-$100          Cost: $50-$200


MODEL 2: Web Server + App Server + Database

┌────────────┐      ┌────────────────┐      ┌────────────────┐
│ Web Server │      │  App Server    │      │  DB Server     │
│  (Nginx)   │      │                │      │                │
│            │─────▶│  Gunicorn/     │─────▶│  PostgreSQL    │
│ SSL termi- │      │  Tomcat/       │      │                │
│ nation +   │      │  Node.js       │      │  + Redis       │
│ static     │      │                │      │  (cache)       │
└────────────┘      └────────────────┘      └────────────────┘


MODEL 3: Frontend + Backend + DB + Cache (4-tier)

┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│Frontend │───▶│ Backend │───▶│  Cache  │───▶│Database │
│ Server  │    │ Server  │    │ (Redis) │    │ Server  │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
                              (check cache     (only hit if
                               first!)          cache miss)
```

---

## How It Works Internally

### Network Communication Between Servers

When servers are separated, they communicate over the **network** instead of localhost:

```
BEFORE (same machine):
App connects to DB via: localhost:5432 (memory/unix socket — FAST)
Latency: ~0.1 ms

AFTER (separate servers):
App connects to DB via: 10.0.1.3:5432 (network TCP — slower)
Latency: ~0.5-2 ms (same datacenter)
Latency: ~10-50 ms (different datacenter)

┌─────────────────────────────────────────────────────────────┐
│                     Private Network (VPC)                     │
│                                                             │
│  ┌──────────┐    TCP/IP    ┌──────────┐    TCP/IP          │
│  │  Backend │◄────────────▶│ Database │                     │
│  │ 10.0.1.2 │  port 5432  │ 10.0.1.3 │                     │
│  └──────────┘              └──────────┘                     │
│       ▲                                                     │
│       │ TCP/IP port 443                                      │
│       │                                                     │
└───────┼─────────────────────────────────────────────────────┘
        │
   ┌────┴─────┐
   │  Client  │ (public internet)
   └──────────┘
```

### Private Network (VPC) — Security Boundary

```
┌─────────────────────────────────────────────────────────────────┐
│                    VPC (Virtual Private Cloud)                    │
│                    10.0.0.0/16                                   │
│                                                                 │
│  ┌───────────────────────────────────┐                          │
│  │     PUBLIC SUBNET (10.0.1.0/24)   │                          │
│  │                                   │                          │
│  │  ┌──────────────────────────────┐ │                          │
│  │  │  Frontend/Load Balancer      │ │  ← Accessible from      │
│  │  │  (public IP: 54.23.xx.xx)   │ │    the internet          │
│  │  └──────────────────────────────┘ │                          │
│  └────────────────┬──────────────────┘                          │
│                   │                                              │
│  ┌────────────────┼──────────────────┐                          │
│  │     PRIVATE SUBNET (10.0.2.0/24)  │                          │
│  │                │                  │                          │
│  │  ┌─────────────▼────────────────┐ │                          │
│  │  │  Backend Server              │ │  ← NOT accessible        │
│  │  │  (private IP: 10.0.2.10)    │ │    from internet!         │
│  │  └─────────────┬───────────────┘ │                          │
│  └────────────────┼──────────────────┘                          │
│                   │                                              │
│  ┌────────────────┼──────────────────┐                          │
│  │     DATABASE SUBNET (10.0.3.0/24) │                          │
│  │                │                  │                          │
│  │  ┌─────────────▼────────────────┐ │                          │
│  │  │  Database Server             │ │  ← Only backend can      │
│  │  │  (private IP: 10.0.3.10)    │ │    reach this!            │
│  │  └─────────────────────────────┘ │                          │
│  └───────────────────────────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Security rules:
- Internet → Frontend: ALLOWED (ports 80, 443)
- Internet → Backend: BLOCKED
- Internet → Database: BLOCKED
- Frontend → Backend: ALLOWED (port 8080)
- Backend → Database: ALLOWED (port 5432)
- Database → Internet: BLOCKED (no outbound!)
```

### How the Frontend Talks to the Backend

```
OPTION A: Backend serves API, Frontend is on CDN

User's Browser
     │
     │ 1. GET https://myapp.com/
     ▼
┌──────────┐
│   CDN    │ 2. Returns HTML + JS bundle (cached at edge)
└──────────┘
     │
     │ 3. JS app loads, makes API call:
     │    fetch("https://api.myapp.com/users")
     ▼
┌──────────┐
│ Backend  │ 4. Processes request, queries DB, returns JSON
│ Server   │
└──────────┘


OPTION B: Nginx reverse proxy routes to backend

User's Browser
     │
     │  All requests go to same domain: myapp.com
     ▼
┌──────────────┐
│    Nginx     │
│   (Server 1) │
│              │
│  /static/*  ──── → Serve files from disk (fast)
│  /api/*     ──── → Proxy to backend server (Server 2)
└──────────────┘
```

---

## Code Examples

### Python (Backend connecting to remote database)

```python
# app.py — Backend server connecting to a SEPARATE database server
from flask import Flask, jsonify
import psycopg2
from psycopg2 import pool
import os

app = Flask(__name__)

# Connection pool to REMOTE database server
# Key difference: host is NO LONGER "localhost"!
db_pool = psycopg2.pool.ThreadedConnectionPool(
    minconn=5,
    maxconn=20,
    host=os.environ["DB_HOST"],        # e.g., "10.0.3.10" or "mydb.abc.us-east-1.rds.amazonaws.com"
    port=5432,
    dbname="myapp",
    user=os.environ["DB_USER"],
    password=os.environ["DB_PASSWORD"],
    # Important for remote connections:
    connect_timeout=5,                  # Don't hang if DB is unreachable
    options="-c statement_timeout=30000"  # 30s query timeout
)

@app.route("/api/products")
def get_products():
    conn = db_pool.getconn()
    try:
        cur = conn.cursor()
        cur.execute("SELECT id, name, price FROM products WHERE active = true")
        products = [{"id": r[0], "name": r[1], "price": float(r[2])} 
                   for r in cur.fetchall()]
        return jsonify(products)
    finally:
        db_pool.putconn(conn)  # Return connection to pool

@app.route("/health")
def health():
    """Health check — load balancer pings this to verify server is alive."""
    try:
        conn = db_pool.getconn()
        cur = conn.cursor()
        cur.execute("SELECT 1")
        db_pool.putconn(conn)
        return jsonify({"status": "healthy", "db": "connected"})
    except Exception as e:
        return jsonify({"status": "unhealthy", "error": str(e)}), 503
```

### Java (Spring Boot with remote database + connection pool)

```java
// application.yml — Spring Boot connecting to a separate DB server
// spring:
//   datasource:
//     url: jdbc:postgresql://10.0.3.10:5432/myapp   ← REMOTE server!
//     username: ${DB_USER}
//     password: ${DB_PASSWORD}
//     hikari:
//       maximum-pool-size: 20        # Pool connections (expensive over network)
//       minimum-idle: 5
//       connection-timeout: 5000     # 5s timeout if DB unreachable
//       idle-timeout: 300000         # Close idle connections after 5 min

@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductRepository productRepository;
    
    @GetMapping
    public ResponseEntity<List<Product>> getProducts() {
        // This query goes over the NETWORK to the database server
        // Network latency adds ~1-2ms per query (vs ~0.1ms on localhost)
        List<Product> products = productRepository.findByActiveTrue();
        return ResponseEntity.ok(products);
    }
    
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        // Health endpoint for load balancer
        try {
            productRepository.count(); // Simple query to verify DB connectivity
            return ResponseEntity.ok(Map.of("status", "healthy"));
        } catch (Exception e) {
            return ResponseEntity.status(503)
                .body(Map.of("status", "unhealthy", "error", e.getMessage()));
        }
    }
}
```

### Frontend (Deployed separately, calling backend API)

```javascript
// frontend/src/api.js — Frontend on CDN calling backend on separate server
const API_BASE = process.env.REACT_APP_API_URL; // "https://api.myapp.com"

export async function fetchProducts() {
    const response = await fetch(`${API_BASE}/api/products`, {
        headers: {
            'Authorization': `Bearer ${getToken()}`,
            'Content-Type': 'application/json'
        }
    });
    
    if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
    }
    
    return response.json();
}

// The frontend doesn't know or care WHERE the backend server is.
// It just calls the API URL. The backend could be:
// - One server
// - Multiple servers behind a load balancer
// - Serverless functions
// Frontend doesn't need to change!
```

---

## Infrastructure Example

### AWS Setup (Typical 3-Tier)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AWS Architecture                             │
│                                                                     │
│  ┌──────────┐     ┌──────────────────┐     ┌──────────────────┐   │
│  │CloudFront│     │     EC2          │     │    RDS           │   │
│  │  (CDN)   │     │  (Backend)       │     │  (PostgreSQL)    │   │
│  │          │     │                  │     │                  │   │
│  │  React   │────▶│  Flask/Spring    │────▶│  Managed DB      │   │
│  │  build   │     │  Boot App        │     │  Auto-backups    │   │
│  │  files   │     │                  │     │  Multi-AZ        │   │
│  │          │     │  t3.medium       │     │  db.t3.medium    │   │
│  │  ~$0     │     │  ~$30/month      │     │  ~$50/month      │   │
│  └──────────┘     └──────────────────┘     └──────────────────┘   │
│                                                                     │
│  Frontend URL: https://d1234.cloudfront.net (or custom domain)     │
│  Backend URL:  https://api.myapp.com (points to EC2)               │
│  DB Endpoint:  mydb.abc123.us-east-1.rds.amazonaws.com             │
│                (only accessible within VPC — not from internet!)    │
└─────────────────────────────────────────────────────────────────────┘
```

### Docker Compose with Separate "Servers" (Simulated Locally)

```yaml
# docker-compose.yml — Simulates separate servers using containers
version: '3.8'

services:
  # "Server 1" — Frontend/Web Server
  frontend:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./frontend/build:/usr/share/nginx/html
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    networks:
      - public_net
    deploy:
      resources:
        limits:
          memory: 256M    # Frontend needs minimal resources

  # "Server 2" — Backend Application
  backend:
    build: ./backend
    environment:
      - DATABASE_URL=postgresql://user:pass@database:5432/myapp
      - REDIS_URL=redis://cache:6379
    networks:
      - public_net     # Can receive requests from frontend
      - private_net    # Can reach database and cache
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'

  # "Server 3" — Database
  database:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - private_net    # ONLY accessible from backend!
    deploy:
      resources:
        limits:
          memory: 2G

  # "Server 4" — Cache
  cache:
    image: redis:7-alpine
    networks:
      - private_net
    deploy:
      resources:
        limits:
          memory: 512M

networks:
  public_net:     # Frontend + Backend can communicate
  private_net:    # Backend + Database + Cache (isolated!)

volumes:
  pgdata:
```

### Terraform Configuration (AWS 3-Tier)

```hcl
# main.tf — Infrastructure as Code for separate servers
resource "aws_instance" "backend" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  subnet_id     = aws_subnet.private.id    # Private subnet!
  
  vpc_security_group_ids = [aws_security_group.backend_sg.id]
  
  tags = { Name = "backend-server" }
}

resource "aws_db_instance" "database" {
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.medium"
  
  db_subnet_group_name   = aws_db_subnet_group.db_subnets.name
  vpc_security_group_ids = [aws_security_group.db_sg.id]
  
  publicly_accessible = false  # KEY: Not accessible from internet!
  multi_az           = false   # Enable for production (costs 2x)
  
  allocated_storage  = 100
  storage_encrypted  = true
}

resource "aws_security_group" "db_sg" {
  name   = "database-sg"
  vpc_id = aws_vpc.main.id
  
  # Only allow connections FROM the backend server
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.backend_sg.id]  # Only backend!
  }
  
  # No egress to internet
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["10.0.0.0/16"]  # Only within VPC
  }
}
```

---

## Real-World Example

### How Flipkart Evolved (India's Amazon)

```
2007: Single PHP server on shared hosting
2008: Separate DB server (MySQL on its own machine)
2010: Frontend on CDN + Backend cluster + DB cluster
2014: Full microservices with hundreds of servers

The FIRST thing they separated was the database.
Why? The database was the bottleneck:
- App server was using 30% CPU
- Database server was at 95% CPU
- Separating let them give the DB a bigger machine (16 GB RAM → 64 GB)
  without touching the app server
```

### Typical Migration Path

```
Step 1: Separate the Database (biggest win!)
┌──────────┐         ┌──────────┐
│  Server  │         │  Server  │
│  (App +  │  ──▶    │  (App)   │─────┐
│   DB)    │         └──────────┘     │
└──────────┘                          ▼
                                ┌──────────┐
                                │  Server  │
                                │  (DB)    │
                                └──────────┘

Step 2: Serve static files from CDN
                     ┌─────────┐
              ┌─────▶│  CDN    │ (static files)
              │      └─────────┘
┌──────────┐  │      ┌──────────┐
│  Client  │──┤      │  App     │
└──────────┘  └─────▶│  Server  │──────┐
                     └──────────┘      │
                                       ▼
                                 ┌──────────┐
                                 │  DB      │
                                 │  Server  │
                                 └──────────┘

Step 3: Add a cache layer
┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐
│  CDN    │    │  App     │    │  Redis  │    │  DB      │
│(static) │    │  Server  │───▶│ (cache) │───▶│  Server  │
└─────────┘    └──────────┘    └─────────┘    └──────────┘
```

---

## Common Mistakes / Pitfalls

### 1. Hardcoding `localhost` in Database Connections
❌ **Mistake**: After separating servers, app still connects to `localhost:5432` — connection refused!
✅ **Fix**: Use environment variables for all connection strings.

```python
# Bad: hardcoded
conn = psycopg2.connect(host="localhost", port=5432)

# Good: environment variable
conn = psycopg2.connect(host=os.environ["DB_HOST"], port=5432)
```

### 2. Not Using Connection Pooling
❌ **Mistake**: Opening a new TCP connection to the remote DB for every request (each takes ~5ms to establish).
✅ **Fix**: Use connection pools (PgBouncer, HikariCP, or application-level pooling).

### 3. Database Accessible from the Internet
❌ **Mistake**: Database server has a public IP and port 5432 open to the world.
✅ **Fix**: Database in a private subnet, accessible ONLY from the backend's security group.

### 4. No Network Failure Handling
❌ **Mistake**: Code assumes the DB is always reachable (like it was on localhost).
✅ **Fix**: Add connection timeouts, retries, and health checks.

```python
# Always set timeouts for remote connections
connection = psycopg2.connect(
    host=DB_HOST,
    connect_timeout=5,    # Fail fast if network is down
)
```

### 5. CORS Issues When Frontend and Backend Are on Different Domains
❌ **Mistake**: Frontend on `myapp.com`, backend on `api.myapp.com` — browser blocks requests.
✅ **Fix**: Configure CORS on the backend.

```python
from flask_cors import CORS
app = Flask(__name__)
CORS(app, origins=["https://myapp.com"])  # Allow frontend domain
```

---

## When to Use / When NOT to Use

### ✅ Separate Servers When:

| Criteria | Why |
|----------|-----|
| **Database is the bottleneck** | Give it dedicated RAM and CPU |
| **Different scaling needs** | Scale backend (CPU-heavy) and DB (memory/IO-heavy) independently |
| **Security compliance** | Isolate database in private network |
| **Team separation** | Frontend team deploys independently from backend team |
| **Need specialized hardware** | DB on high-IOPS SSD, backend on compute-optimized |
| **Growing past ~10,000 daily users** | Single server is getting maxed |

### ❌ Stay on Single Server When:

| Criteria | Why |
|----------|-----|
| **Traffic is very low** | Extra servers = extra cost + complexity |
| **Budget is extremely tight** | One $20 server < three $20 servers |
| **Solo developer with simple app** | Ops overhead isn't worth it yet |
| **Development/staging environment** | Single server is fine for non-prod |

---

## Cost Comparison

```
SINGLE SERVER:
┌────────────────────────────────┐
│  1 x $48/month (4 CPU, 8 GB)  │
│  Total: $48/month              │
└────────────────────────────────┘

SEPARATE SERVERS (same total resources):
┌────────────────────────────────────────────────────────┐
│  Frontend: CDN (Cloudflare free tier)    $0            │
│  Backend:  1 x $24/month (2 CPU, 4 GB)  $24           │
│  Database: 1 x $48/month (2 CPU, 8 GB)  $48           │
│  Redis:    1 x $15/month (1 GB)         $15           │
│  Total: $87/month                                      │
│                                                        │
│  MORE expensive! But:                                  │
│  ✓ DB gets 8 GB dedicated RAM (vs shared 8 GB)        │
│  ✓ DB failure doesn't kill the app server              │
│  ✓ Can upgrade DB independently                       │
│  ✓ Better security isolation                          │
│  ✓ Ready for next scaling step (multiple instances)    │
└────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

- **Separating the database** is the first and most impactful scaling step — it gets dedicated resources and better security.
- **Network latency appears** when components are on different machines (~1-2ms per call in the same datacenter vs ~0.1ms on localhost).
- **Connection pooling becomes essential** — opening new TCP connections per request is too expensive over the network.
- **Private networking (VPC)** keeps your database invisible to the internet — only the backend can reach it.
- **Frontend on a CDN** eliminates an entire server and gives users faster load times globally.
- **Each component scales independently** — need more app power? Upgrade the backend. Need more storage? Upgrade the DB.
- **This is where 90% of production applications live** — it's the sweet spot between simplicity and scalability.

---

## What's Next?

Your backend server is now separated, but what happens when ONE backend server can't handle all the traffic? You need to run **multiple copies** of it. That's **Chapter 5.3: Multiple Instances of the Same Application (Horizontal Scaling)** — where we add a load balancer and run 2, 5, or 50 copies of your backend.
