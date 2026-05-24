# Stage 1: Single Server — 0 to 100 Users

> **What you'll learn**: How every web application starts its life on a single machine, why this is perfectly fine for early stage, and exactly what happens inside that one server.

---

## Real-Life Analogy

Imagine you open a small **chai stall** on a street corner. You are the owner, the cook, the waiter, and the cashier — all in one person. You take orders, boil chai, serve cups, collect money, and wash cups yourself.

When you have 5-10 customers at a time, this works perfectly. You know everyone by name, the system is simple, and there's nothing to coordinate because it's all YOU.

That's exactly how a web app starts — **one server does everything**: serves the website, runs the business logic, and stores the data.

---

## Core Concept Explained Step-by-Step

### What Does "Single Server" Mean?

It means your **entire application** — the frontend (HTML/CSS/JS), the backend (API logic), and the database — all run on **one physical or virtual machine**.

```
┌──────────────────────────────────────────────────┐
│              SINGLE SERVER (1 Machine)            │
│                                                  │
│  ┌─────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ Frontend │  │   Backend   │  │  Database   │  │
│  │  (HTML/  │  │  (API /     │  │ (PostgreSQL │  │
│  │   CSS/   │  │   Business  │  │  or MySQL)  │  │
│  │   JS)    │  │   Logic)    │  │             │  │
│  └─────────┘  └─────────────┘  └─────────────┘  │
│                                                  │
│  ┌─────────────────────────────────────────────┐  │
│  │         Operating System (Linux)            │  │
│  └─────────────────────────────────────────────┘  │
│                                                  │
│  ┌─────────────────────────────────────────────┐  │
│  │   Hardware: CPU, RAM, Disk, Network Card    │  │
│  └─────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

### How a User Request Flows

```
    User's Browser
         │
         │ (1) Types www.myapp.com
         ▼
    ┌─────────┐
    │   DNS   │ ──── Resolves to single IP: 203.0.113.10
    └─────────┘
         │
         │ (2) HTTP Request
         ▼
┌──────────────────────────────────────┐
│         SINGLE SERVER                │
│         203.0.113.10                 │
│                                      │
│  (3) Nginx receives request          │
│         │                            │
│         ▼                            │
│  (4) If static file → serve directly │
│      If API call → forward to app    │
│         │                            │
│         ▼                            │
│  (5) App processes business logic    │
│         │                            │
│         ▼                            │
│  (6) Query database (same machine)   │
│         │                            │
│         ▼                            │
│  (7) Return response to browser      │
└──────────────────────────────────────┘
```

### Why Start Here?

| Reason | Explanation |
|--------|-------------|
| **Simplicity** | One machine = one place to look when things break |
| **Cost** | $5-20/month on DigitalOcean, AWS Lightsail, or a VPS |
| **Speed of development** | Deploy by just copying files or `git pull` |
| **No coordination** | No network latency between components (they share memory) |
| **Good enough** | 100 users generating ~10-50 requests/second is trivial for modern hardware |

### What's Actually Running on This Server?

A typical single-server setup has these processes:

```
┌─────────────────────────────────────────────────────┐
│                    Linux Server                       │
│                                                     │
│  Process 1: Nginx (Web Server)        Port 80/443   │
│       ↓ reverse proxies to...                       │
│  Process 2: App Server (Gunicorn/Tomcat) Port 8000  │
│       ↓ connects to...                              │
│  Process 3: PostgreSQL               Port 5432      │
│                                                     │
│  Process 4: Redis (optional cache)   Port 6379      │
│  Process 5: Cron jobs (background tasks)            │
│                                                     │
│  RAM: 2-4 GB  │  CPU: 1-2 cores  │  Disk: 50 GB    │
└─────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### The Request Journey (Detailed)

1. **DNS Resolution**: Browser asks DNS "Where is myapp.com?" → Gets back `203.0.113.10`
2. **TCP Connection**: Browser opens a TCP connection to port 443 (HTTPS)
3. **TLS Handshake**: SSL certificate is verified, encrypted channel established
4. **HTTP Request arrives at Nginx**: Nginx checks if it's a static file (images, CSS, JS)
   - If yes → serves directly from disk (very fast)
   - If no → forwards to the application server (reverse proxy)
5. **Application server processes**: Flask/Django/Spring Boot handles the business logic
6. **Database query**: App connects to PostgreSQL on `localhost:5432` (no network hop!)
7. **Response**: App builds the response, Nginx sends it back to the browser

### Why localhost is Fast

When the database is on the same machine:
- Communication happens through **Unix sockets** or **loopback interface** (127.0.0.1)
- Data travels through **shared memory**, not through network cables
- Latency: **< 0.1ms** (compared to 1-5ms over a network)

### Capacity of a Single Server

A modern server with 2 CPU cores and 4 GB RAM can typically handle:

| Metric | Capacity |
|--------|----------|
| Concurrent connections | 500-1,000 |
| Requests per second (simple API) | 500-2,000 |
| Requests per second (with DB queries) | 100-500 |
| Active users at once | 50-200 |
| Monthly active users | 1,000-10,000 |

> **Key Insight**: The bottleneck is almost always the **database**, not the application server.

---

## Code Examples

### Python: Complete Single-Server App (Flask + PostgreSQL)

```python
# app.py — A complete single-server web application
from flask import Flask, jsonify, request
import psycopg2
from psycopg2.extras import RealDictCursor

app = Flask(__name__)

# Database on the SAME machine (localhost)
def get_db():
    return psycopg2.connect(
        host="localhost",      # Same machine!
        port=5432,
        database="myapp",
        user="appuser",
        password="secret",
        cursor_factory=RealDictCursor
    )

@app.route("/")
def home():
    """Serve the homepage — on the same server"""
    return app.send_static_file("index.html")

@app.route("/api/users", methods=["GET"])
def get_users():
    """Fetch users from database on the same machine"""
    conn = get_db()
    cur = conn.cursor()
    cur.execute("SELECT id, name, email FROM users LIMIT 50")
    users = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify(users)

@app.route("/api/users", methods=["POST"])
def create_user():
    """Create a new user"""
    data = request.get_json()
    conn = get_db()
    cur = conn.cursor()
    cur.execute(
        "INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id",
        (data["name"], data["email"])
    )
    new_id = cur.fetchone()["id"]
    conn.commit()
    cur.close()
    conn.close()
    return jsonify({"id": new_id}), 201

# Run with: gunicorn app:app --workers 4 --bind 0.0.0.0:8000
```

### Java: Complete Single-Server App (Spring Boot + PostgreSQL)

```java
// UserController.java — Same concept in Java
@RestController
@RequestMapping("/api/users")
public class UserController {

    // Database on the same machine (localhost)
    // application.properties: spring.datasource.url=jdbc:postgresql://localhost:5432/myapp

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @GetMapping
    public List<Map<String, Object>> getUsers() {
        // Query goes to PostgreSQL on localhost — sub-millisecond latency
        return jdbcTemplate.queryForList(
            "SELECT id, name, email FROM users LIMIT 50"
        );
    }

    @PostMapping
    public ResponseEntity<Map<String, Object>> createUser(@RequestBody Map<String, String> user) {
        KeyHolder keyHolder = new GeneratedKeyHolder();
        jdbcTemplate.update(connection -> {
            PreparedStatement ps = connection.prepareStatement(
                "INSERT INTO users (name, email) VALUES (?, ?)",
                Statement.RETURN_GENERATED_KEYS
            );
            ps.setString(1, user.get("name"));
            ps.setString(2, user.get("email"));
            return ps;
        }, keyHolder);

        Map<String, Object> response = new HashMap<>();
        response.put("id", keyHolder.getKeys().get("id"));
        return ResponseEntity.status(201).body(response);
    }
}
```

---

## Infrastructure Example: Setting Up a Single Server

### Nginx Configuration (Reverse Proxy)

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name myapp.com;

    # Serve static files directly (fast!)
    location /static/ {
        root /var/www/myapp;
        expires 30d;
    }

    # Forward everything else to the app server
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Docker Compose: Single Server with All Components

```yaml
# docker-compose.yml — Everything on one machine
version: '3.8'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app

  app:
    build: .
    expose:
      - "8000"
    environment:
      - DATABASE_URL=postgresql://appuser:secret@db:5432/myapp
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=secret
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

---

## Real-World Example

### Every Startup Starts Here

- **Instagram** in 2010: Started on a single AWS EC2 instance. Served their first 25,000 users from ONE server.
- **Basecamp (37signals)**: Built their flagship product on a single server for years before scaling.
- **Most SaaS MVPs**: Tools like Notion, Linear, and countless others started their journey on a single $20/month VPS.

### Why This Works

Instagram's co-founder Kevin Systrom said: *"We didn't need to think about scaling on day one. We needed to ship."*

The lesson: **Don't over-engineer for scale you don't have yet.** A single server can handle far more than most people think.

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Over-engineering from day 1** | Spending months on microservices for 10 users | Start with a single server, scale later |
| **Not setting up backups** | If the disk dies, everything is gone | Automated daily backups to S3/external storage |
| **Running as root** | Security vulnerability | Create a dedicated app user |
| **No monitoring** | You don't know when things break | Add basic monitoring (uptime checks, disk alerts) |
| **Storing secrets in code** | Leaked passwords if code is public | Use environment variables |
| **Not using HTTPS** | Data is transmitted in plain text | Use Let's Encrypt (free SSL) |
| **Ignoring disk space** | Database fills disk → crash | Monitor disk, set up log rotation |

---

## When to Use / When NOT to Use

### ✅ Use Single Server When:
- You're building an MVP or proof of concept
- You have < 100 daily active users
- Your budget is < $50/month
- Speed of development matters more than reliability
- You're a solo developer or tiny team
- It's a personal project, internal tool, or hackathon

### ❌ Move Away From Single Server When:
- CPU consistently above 70% utilization
- RAM is constantly near full
- Response times exceed 2-3 seconds
- You need high availability (can't afford downtime)
- Single disk failure would cause data loss with no recovery
- You're generating revenue and users depend on uptime

---

## Key Takeaways

1. **Every great system started as a single server** — Instagram, Twitter, Shopify all began on one machine
2. **A $20 server can handle 100+ users easily** — modern hardware is powerful
3. **Simplicity is a feature** — fewer moving parts = fewer things that break
4. **The bottleneck is usually the database** — even on a single server
5. **Always set up backups** — this is the ONE thing you must do from day 1
6. **localhost communication is near-zero latency** — components on the same machine talk fast
7. **Don't scale prematurely** — wait until you have a real scaling problem before adding complexity

---

## What's Next?

When your single server starts struggling — slow responses, high CPU, or you're worried about a single point of failure — it's time for **Stage 2: Separating the Database to its own server**. This is the first and most impactful architectural change you'll make.

Next: [02-stage2-separate-db.md](./02-stage2-separate-db.md)
