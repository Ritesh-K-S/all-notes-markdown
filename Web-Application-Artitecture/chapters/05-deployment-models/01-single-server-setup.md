# Single Server Setup — Everything on One Machine

> **What you'll learn**: How a web application runs when your frontend, backend, and database all live on a single server — the simplest possible deployment — and why every journey to planet-scale starts right here.

---

## Real-Life Analogy

Imagine you open a **tiny restaurant** with just ONE room. In that room, you have:
- The **dining table** (where customers sit) → Frontend
- The **kitchen** (where food is cooked) → Backend/Application
- The **fridge & pantry** (where ingredients are stored) → Database

Everything is in one room. The waiter takes the order, walks three steps to the kitchen, grabs food from the fridge, cooks it, and serves it — all within the same small space.

**Pros**: Dead simple. No coordination needed. Cheap rent. One person can manage everything.

**Cons**: If the kitchen catches fire, the dining area is gone too. If 50 people show up, there's no space. One room = one point of failure.

That's a **single server setup**. Your entire application — HTML/CSS/JS files, application code, and database — all run on one physical or virtual machine.

---

## Core Concept Explained Step-by-Step

### What "Single Server" Means

```
┌─────────────────────────────────────────────────────────┐
│                    ONE SERVER (e.g., a $5 VPS)           │
│                                                         │
│  ┌────────────────┐  ┌─────────────────┐              │
│  │   Web Server   │  │   Application   │              │
│  │   (Nginx)      │  │   (Python/Java) │              │
│  │                │  │                 │              │
│  │ Serves static  │  │ Handles API     │              │
│  │ files (HTML,   │──▶│ requests,       │              │
│  │ CSS, JS)       │  │ business logic  │              │
│  └────────────────┘  └────────┬────────┘              │
│                               │                        │
│                               ▼                        │
│                      ┌─────────────────┐              │
│                      │    Database     │              │
│                      │  (PostgreSQL/   │              │
│                      │   MySQL/SQLite) │              │
│                      └─────────────────┘              │
│                                                         │
│  IP: 203.0.113.42                                      │
│  RAM: 2 GB                                             │
│  Disk: 50 GB                                           │
│  CPU: 1 vCPU                                           │
└─────────────────────────────────────────────────────────┘
```

### The Complete Request Flow

When a user visits your site on a single-server setup:

```
User's Browser
     │
     │  1. DNS lookup: mysite.com → 203.0.113.42
     ▼
┌─────────────────────────────────────────────────────────────┐
│  Server (203.0.113.42)                                      │
│                                                             │
│  ┌──────────────────────────┐                              │
│  │  Nginx (Port 80/443)     │  2. Receives HTTP request    │
│  │                          │                              │
│  │  Static request?         │                              │
│  │  (/style.css, /app.js)  │                              │
│  │  → Serve file directly   │                              │
│  │                          │                              │
│  │  API request?            │                              │
│  │  (/api/users)            │                              │
│  │  → Forward to app ───────┼──┐                           │
│  └──────────────────────────┘  │                           │
│                                ▼                           │
│  ┌──────────────────────────┐                              │
│  │  App Server (Port 8000)  │  3. Process business logic   │
│  │  (Gunicorn / Tomcat)     │                              │
│  │                          │                              │
│  │  Need data? ─────────────┼──┐                           │
│  └──────────────────────────┘  │                           │
│                                ▼                           │
│  ┌──────────────────────────┐                              │
│  │  Database (Port 5432)    │  4. Query/write data         │
│  │  PostgreSQL              │                              │
│  │                          │  5. Return results           │
│  └──────────────────────────┘                              │
│                                                             │
│  6. Response travels back through Nginx → User              │
└─────────────────────────────────────────────────────────────┘
```

### What's Running on This One Machine

| Component | Process | Port | Purpose |
|-----------|---------|------|---------|
| Nginx | `nginx` | 80, 443 | Accept HTTP/HTTPS requests, serve static files, proxy to app |
| App Server | `gunicorn` / `java -jar app.jar` | 8000 / 8080 | Run your application code |
| Database | `postgres` / `mysqld` | 5432 / 3306 | Store and retrieve data |
| OS | Linux (Ubuntu/Debian) | — | Manage resources, run all processes |

### Why Start Here?

Every company that exists today — Google, Amazon, Facebook — started with a single server:
- **Google (1998)**: Ran on a few machines in a Stanford dorm room
- **Facebook (2004)**: One MySQL server in Mark Zuckerberg's dorm
- **Twitter (2006)**: Single Ruby on Rails app on one server

You start here because:
1. It's the **cheapest** option ($5-$20/month on DigitalOcean, Linode, or AWS Lightsail)
2. It's the **simplest** to understand and debug
3. It's **enough** for most small projects (up to ~1,000 daily users easily)
4. It teaches you **every layer** of the stack without abstraction

---

## How It Works Internally

### Process Management

On a Linux server, your application stack runs as separate **processes** managed by the operating system:

```
┌─────────────────────────────────────────────────────┐
│              Linux Kernel (Resource Manager)          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  PID 1: systemd (init system)                       │
│    ├── PID 234: nginx (master)                      │
│    │     ├── PID 235: nginx worker 1                │
│    │     └── PID 236: nginx worker 2                │
│    ├── PID 500: gunicorn (master)                   │
│    │     ├── PID 501: gunicorn worker 1             │
│    │     ├── PID 502: gunicorn worker 2             │
│    │     └── PID 503: gunicorn worker 3             │
│    ├── PID 100: postgres (master)                   │
│    │     ├── PID 101: postgres background writer    │
│    │     ├── PID 102: postgres checkpointer         │
│    │     └── PID 103: postgres stats collector      │
│    └── PID 800: sshd (for you to connect)           │
│                                                     │
│  Shared Resources:                                   │
│    CPU: All processes share 1-2 vCPUs               │
│    RAM: 2 GB total, split among all processes       │
│    Disk I/O: One disk, shared reads/writes          │
│    Network: One NIC, one IP address                 │
└─────────────────────────────────────────────────────┘
```

### Resource Sharing & Contention

The critical challenge of a single server: **everyone shares everything**.

```
RESOURCE COMPETITION:

Total RAM: 2 GB
├── OS + System:     300 MB
├── Nginx:           50 MB
├── App (3 workers): 600 MB (200 MB each)
├── PostgreSQL:      512 MB (shared_buffers)
├── Disk Cache:      ~500 MB (OS file cache)
└── Available:       ~38 MB ← barely anything left!

When traffic spikes:
- App needs more RAM for requests → steals from disk cache
- PostgreSQL needs more RAM for queries → OS starts swapping to disk
- Swapping to disk → everything becomes 100x slower
- Server grinds to a halt → users get timeouts
```

### How Nginx Routes to Your App

Nginx acts as the **gatekeeper** — it decides what to handle itself vs. what to pass to your application:

```
                    Nginx Decision Tree
                          │
                    ┌─────┴─────┐
                    │  Request   │
                    │  arrives   │
                    └─────┬─────┘
                          │
                   ┌──────┴──────┐
                   │ Static file? │
                   │ (.css, .js,  │
                   │  .png, .html)│
                   └──┬───────┬──┘
                      │       │
                    YES       NO
                      │       │
                      ▼       ▼
              ┌─────────┐  ┌─────────────────┐
              │  Serve   │  │  Proxy pass to  │
              │  from    │  │  App Server     │
              │  /var/   │  │  (localhost:8000)│
              │  www/    │  │                 │
              └─────────┘  └─────────────────┘
              (Very fast    (Slower — app must
               ~1ms)        process logic ~50ms)
```

### The Filesystem Layout

```
/
├── etc/
│   ├── nginx/
│   │   └── nginx.conf          ← Nginx configuration
│   ├── systemd/system/
│   │   └── myapp.service       ← App auto-start config
│   └── postgresql/
│       └── postgresql.conf     ← Database config
├── var/
│   ├── www/mysite/
│   │   ├── index.html          ← Static frontend files
│   │   ├── style.css
│   │   └── app.js
│   ├── log/
│   │   ├── nginx/access.log    ← Web server logs
│   │   └── myapp/app.log       ← Application logs
│   └── lib/postgresql/data/    ← Database data files
├── home/deploy/
│   └── myapp/
│       ├── app.py              ← Application source code
│       ├── requirements.txt
│       └── venv/               ← Python virtual environment
└── tmp/                        ← Temporary files
```

---

## Code Examples

### Python (Flask App with Gunicorn + Nginx)

```python
# app.py — A simple Flask application running on a single server
from flask import Flask, jsonify, request
import psycopg2
import os

app = Flask(__name__)

# Database connection — same machine, so we use localhost
def get_db():
    return psycopg2.connect(
        host="localhost",       # Same machine!
        port=5432,
        dbname="myapp",
        user="myapp_user",
        password=os.environ["DB_PASSWORD"]
    )

@app.route("/")
def index():
    """Serve the homepage — Nginx usually handles static files,
    but Flask can serve them too for simplicity."""
    return app.send_static_file("index.html")

@app.route("/api/users", methods=["GET"])
def get_users():
    """API endpoint — fetches users from PostgreSQL on same machine."""
    conn = get_db()
    cur = conn.cursor()
    cur.execute("SELECT id, name, email FROM users LIMIT 50")
    users = [{"id": r[0], "name": r[1], "email": r[2]} for r in cur.fetchall()]
    cur.close()
    conn.close()
    return jsonify(users)

@app.route("/api/users", methods=["POST"])
def create_user():
    """Create a new user — all on the same server."""
    data = request.json
    conn = get_db()
    cur = conn.cursor()
    cur.execute(
        "INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id",
        (data["name"], data["email"])
    )
    user_id = cur.fetchone()[0]
    conn.commit()
    cur.close()
    conn.close()
    return jsonify({"id": user_id}), 201

# Run with: gunicorn app:app --workers 3 --bind 0.0.0.0:8000
```

### Java (Spring Boot with Embedded Tomcat)

```java
// UserController.java — Spring Boot app on a single server
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    // Database is on localhost — same machine
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @GetMapping
    public List<Map<String, Object>> getUsers() {
        // Query PostgreSQL running on the same machine (localhost:5432)
        return jdbcTemplate.queryForList(
            "SELECT id, name, email FROM users LIMIT 50"
        );
    }
    
    @PostMapping
    public ResponseEntity<Map<String, Object>> createUser(@RequestBody Map<String, String> body) {
        KeyHolder keyHolder = new GeneratedKeyHolder();
        jdbcTemplate.update(connection -> {
            PreparedStatement ps = connection.prepareStatement(
                "INSERT INTO users (name, email) VALUES (?, ?)",
                Statement.RETURN_GENERATED_KEYS
            );
            ps.setString(1, body.get("name"));
            ps.setString(2, body.get("email"));
            return ps;
        }, keyHolder);
        
        Map<String, Object> response = new HashMap<>();
        response.put("id", keyHolder.getKey().longValue());
        return ResponseEntity.status(201).body(response);
    }
}

// application.properties:
// spring.datasource.url=jdbc:postgresql://localhost:5432/myapp
// server.port=8080
// Run with: java -jar myapp.jar
```

### Nginx Configuration

```nginx
# /etc/nginx/sites-available/mysite
server {
    listen 80;
    server_name mysite.com;
    
    # Serve static files directly (fast — no app involvement)
    location /static/ {
        alias /var/www/mysite/static/;
        expires 30d;                    # Browser caches for 30 days
        add_header Cache-Control "public, immutable";
    }
    
    # Serve the frontend HTML
    location / {
        root /var/www/mysite;
        try_files $uri $uri/ /index.html;  # SPA fallback
    }
    
    # Proxy API requests to the application server
    location /api/ {
        proxy_pass http://127.0.0.1:8000;   # App runs on same machine
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Systemd Service (Auto-start your app)

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Web Application
After=network.target postgresql.service

[Service]
User=deploy
WorkingDirectory=/home/deploy/myapp
ExecStart=/home/deploy/myapp/venv/bin/gunicorn app:app --workers 3 --bind 0.0.0.0:8000
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## Infrastructure Example

### Setting Up a Single Server (Step-by-Step)

```bash
# 1. Provision a VPS (e.g., DigitalOcean $6/month droplet)
#    - Ubuntu 22.04 LTS
#    - 1 GB RAM, 1 vCPU, 25 GB SSD

# 2. SSH into the server
ssh root@203.0.113.42

# 3. Install everything on ONE machine
apt update && apt upgrade -y
apt install -y nginx postgresql python3-pip python3-venv

# 4. Set up the database
sudo -u postgres createuser myapp_user
sudo -u postgres createdb myapp --owner=myapp_user

# 5. Deploy your application
mkdir -p /home/deploy/myapp
cd /home/deploy/myapp
python3 -m venv venv
source venv/bin/activate
pip install flask gunicorn psycopg2-binary

# 6. Configure Nginx (as shown above)
# 7. Start services
systemctl enable nginx postgresql myapp
systemctl start myapp
```

### Docker Compose (Modern Single-Server Approach)

```yaml
# docker-compose.yml — Run everything on one machine with Docker
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./frontend/build:/usr/share/nginx/html
    depends_on:
      - app

  app:
    build: ./backend
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
    expose:
      - "8000"
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - pgdata:/var/lib/postgresql/data
    expose:
      - "5432"

volumes:
  pgdata:  # Persistent database storage
```

```
All three containers run on the SAME machine:

┌─────────────────────────────────────────────────┐
│  Single Server (Docker Host)                    │
│                                                 │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐     │
│  │  Nginx  │──▶│   App   │──▶│   DB    │     │
│  │ :80/443 │   │  :8000  │   │  :5432  │     │
│  └─────────┘   └─────────┘   └─────────┘     │
│                                                 │
│  They communicate via Docker's internal network │
│  (still localhost from the server's perspective)│
└─────────────────────────────────────────────────┘
```

---

## Real-World Example

### How Small Startups Start

**Basecamp** (the project management tool with millions of users) famously runs a relatively simple server setup. DHH (creator of Ruby on Rails) advocates for this "majestic monolith" approach.

**Typical small startup journey:**

```
Month 1-3: One $20/month server
├── Frontend: React build served by Nginx
├── Backend: Rails/Django/Spring Boot
├── Database: PostgreSQL
├── Users: 0 → 1,000
└── Cost: $20/month

Month 4-6: Upgraded to $80/month server  
├── Same setup, bigger machine (4 GB RAM, 2 vCPU)
├── Added Redis for sessions
├── Users: 1,000 → 10,000
└── Cost: $80/month

Month 7+: Time to split (see Chapter 5.2)
├── Database is the first bottleneck
├── Need separate DB server
└── Traffic outgrowing single machine
```

### DigitalOcean / Hetzner Single-Server Pricing (2024)

| Specs | Monthly Cost | Can Handle |
|-------|-------------|------------|
| 1 vCPU, 1 GB RAM | $6 | ~500 daily users |
| 2 vCPU, 4 GB RAM | $24 | ~5,000 daily users |
| 4 vCPU, 8 GB RAM | $48 | ~20,000 daily users |
| 8 vCPU, 16 GB RAM | $96 | ~50,000 daily users |

> **Note**: These numbers vary wildly based on what your app does. A chat app with WebSockets will hit limits faster than a blog.

---

## Common Mistakes / Pitfalls

### 1. Running the Database Without Backups
❌ **Mistake**: No backup strategy. If the disk dies, ALL your data is gone.
✅ **Fix**: Set up daily `pg_dump` backups stored to a different location (S3 or another machine).

```bash
# Simple backup cron job
0 3 * * * pg_dump myapp | gzip > /backups/myapp-$(date +%Y%m%d).sql.gz
# Also: copy backups OFF the server (to S3, Google Drive, etc.)
```

### 2. No Firewall Configuration
❌ **Mistake**: PostgreSQL port (5432) is exposed to the internet.
✅ **Fix**: Only expose ports 80 (HTTP) and 443 (HTTPS). Block everything else.

```bash
ufw allow 22/tcp     # SSH (consider changing to a non-standard port)
ufw allow 80/tcp     # HTTP
ufw allow 443/tcp    # HTTPS
ufw deny 5432/tcp    # Block direct DB access from internet!
ufw enable
```

### 3. Running Everything as Root
❌ **Mistake**: Your app runs as the `root` user — if compromised, attacker owns everything.
✅ **Fix**: Create a dedicated `deploy` user with minimal permissions.

### 4. Not Monitoring Disk Space
❌ **Mistake**: Logs fill up the disk, database runs out of space, server crashes.
✅ **Fix**: Set up log rotation and disk space monitoring.

### 5. No SSL/HTTPS
❌ **Mistake**: Running on plain HTTP — user data sent in plaintext.
✅ **Fix**: Use Let's Encrypt (free!) with Certbot.

```bash
apt install certbot python3-certbot-nginx
certbot --nginx -d mysite.com
# Auto-renews every 90 days
```

---

## When to Use / When NOT to Use

### ✅ Use a Single Server When:

| Criteria | Why |
|----------|-----|
| **Side project or MVP** | Fast to set up, cheap, no over-engineering |
| **< 10,000 daily active users** | One server can handle this easily |
| **Budget is tight** | $5-$50/month vs. $500+ for a multi-server setup |
| **Solo developer / small team** | Less ops complexity, one place to debug |
| **Read-heavy application** | Blog, docs site, portfolio — Nginx serves static files fast |
| **Learning/prototyping** | See the entire stack on one machine |

### ❌ Don't Use When:

| Criteria | Why |
|----------|-----|
| **High availability required** | One server = one point of failure. If it dies, you're offline |
| **Traffic exceeds single machine capacity** | CPU/RAM maxed out |
| **Database is I/O-bound** | DB and app competing for same disk |
| **Compliance requires separation** | PCI-DSS, HIPAA may require isolated database servers |
| **Team is large** | Multiple devs deploying to one server = conflicts |

---

## Capacity Planning: When to Leave Single Server

```
WARNING SIGNS that you've outgrown a single server:

┌─────────────────────────────────────────────────────────┐
│  CPU consistently > 80%            → Need more compute  │
│  RAM consistently > 85%            → Need more memory   │
│  Disk I/O wait > 20%              → DB needs own disk   │
│  Response times > 2 seconds       → System overloaded   │
│  Database queries slowing down    → DB needs own server  │
│  Can't deploy without downtime    → Need multiple nodes  │
│  Any single failure = total outage → Need redundancy     │
└─────────────────────────────────────────────────────────┘

SOLUTION: Move to Chapter 5.2 → Separate Servers
```

---

## Key Takeaways

- **A single server setup** means your frontend, backend, and database all run on one machine — the simplest deployment model.
- **Every major tech company started here** — don't over-engineer until you have real traffic.
- **Nginx sits in front** as a reverse proxy, serving static files directly and forwarding API requests to your app.
- **Shared resources are the bottleneck** — CPU, RAM, and disk I/O are split between all components.
- **Security basics are critical** — firewall, non-root user, SSL, database backups, even on one server.
- **You can serve 5,000-50,000 daily users** on a well-configured single server (depending on workload).
- **Know when to leave** — when CPU/RAM stays above 80%, response times grow, or you need zero-downtime deployments, it's time to scale.

---

## What's Next?

When your single server starts struggling, the first step is to **separate concerns** — put the database on its own server, maybe serve static files from a CDN. That's exactly what we cover in **Chapter 5.2: Separate Servers for Frontend, Backend & Database**.
