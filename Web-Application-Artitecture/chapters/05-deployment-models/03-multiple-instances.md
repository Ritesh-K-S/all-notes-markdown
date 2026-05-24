# Multiple Instances of the Same Application (Horizontal Scaling)

> **What you'll learn**: How to run multiple copies of your application behind a load balancer to handle more traffic, achieve high availability, and deploy without downtime — the foundation of modern scalable systems.

---

## Real-Life Analogy

Your restaurant's kitchen (backend server) is getting overwhelmed — one chef can't cook fast enough for 200 orders. What do you do?

**Option A (Vertical Scaling)**: Hire a SUPERHERO chef who cooks 5x faster. Expensive, and there's a limit to how fast one person can cook.

**Option B (Horizontal Scaling)**: Hire 5 regular chefs, each with their own cooking station, all making the same menu. A **host** (load balancer) directs incoming orders to whichever chef is least busy.

Option B is **horizontal scaling** — instead of making one server bigger, you run multiple copies of the same server and distribute traffic among them.

```
WITHOUT horizontal scaling:                WITH horizontal scaling:

  200 orders                                 200 orders
      │                                          │
      ▼                                          ▼
┌──────────┐                              ┌──────────┐
│  1 Chef  │ ← Overwhelmed!              │   Host   │ (Load Balancer)
│ (1 server)│                              └────┬─────┘
└──────────┘                                   │
                                    ┌──────────┼──────────┐
                                    ▼          ▼          ▼
                              ┌────────┐ ┌────────┐ ┌────────┐
                              │ Chef 1 │ │ Chef 2 │ │ Chef 3 │
                              │(67 ord)│ │(67 ord)│ │(66 ord)│
                              └────────┘ └────────┘ └────────┘
                              
                              Each handles ~67 orders. Easy!
```

---

## Core Concept Explained Step-by-Step

### What Are "Multiple Instances"?

An **instance** is one running copy of your application. Multiple instances means:
- Same code
- Same configuration
- Running on different machines (or same machine, different ports)
- All handling the same type of requests

```
┌─────────────────────────────────────────────────────────────────┐
│                     HORIZONTAL SCALING                           │
│                                                                 │
│   Clients                                                       │
│   │ │ │ │ │ │ │ │ │ │                                          │
│   └─┴─┴─┴─┴─┴─┴─┴─┴─┘                                          │
│           │                                                     │
│           ▼                                                     │
│   ┌───────────────┐                                             │
│   │ Load Balancer │  Distributes requests across instances      │
│   └───────┬───────┘                                             │
│           │                                                     │
│     ┌─────┼─────┬─────────┐                                    │
│     ▼     ▼     ▼         ▼                                    │
│   ┌───┐ ┌───┐ ┌───┐    ┌───┐                                  │
│   │ 1 │ │ 2 │ │ 3 │    │ N │   ← All running SAME code       │
│   └───┘ └───┘ └───┘    └───┘                                  │
│     │     │     │         │                                    │
│     └─────┴─────┴─────────┘                                    │
│                 │                                               │
│                 ▼                                               │
│         ┌─────────────┐                                         │
│         │  Database   │  ← Shared! (single source of truth)    │
│         └─────────────┘                                         │
└─────────────────────────────────────────────────────────────────┘
```

### Key Requirements for Multiple Instances

For your app to run as multiple instances, it must be **stateless**:

```
STATELESS APPLICATION (✅ can scale horizontally):
- No data stored in local memory between requests
- No files written to local disk that other instances need
- Session data stored in Redis/database, NOT in-memory
- Any instance can handle any request

STATEFUL APPLICATION (❌ can't easily scale):
- User sessions stored in server memory (HttpSession)
- Uploaded files stored on local disk
- In-memory caches that aren't shared
- WebSocket connections tied to specific instance
```

### The 12-Factor Rule: Store Nothing Locally

```
BAD (Stateful — can't scale):
┌──────────┐   ┌──────────┐
│Instance 1│   │Instance 2│
│          │   │          │
│ sessions │   │ sessions │   ← Each has DIFFERENT sessions!
│ {user:A} │   │ {user:B} │   ← If user A hits Instance 2, 
└──────────┘   └──────────┘     their session is GONE!

GOOD (Stateless — scales perfectly):
┌──────────┐   ┌──────────┐
│Instance 1│   │Instance 2│
│          │   │          │
│ (no local│   │ (no local│   ← Both are identical
│  state)  │   │  state)  │   ← Any request → any instance
└──────────┘   └──────────┘
      │              │
      └──────┬───────┘
             ▼
      ┌─────────────┐
      │   Redis     │   ← Shared session store
      │ {user:A}    │   ← All instances read from here
      │ {user:B}    │
      └─────────────┘
```

---

## How It Works Internally

### The Load Balancer's Job

```
INCOMING REQUESTS (1000/second):
    │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │

                    │
                    ▼
    ┌───────────────────────────────────┐
    │         LOAD BALANCER             │
    │                                   │
    │  Algorithm: Round Robin           │
    │  Health Check: /health every 10s  │
    │                                   │
    │  Instance 1: HEALTHY ✓ (250 rps)  │
    │  Instance 2: HEALTHY ✓ (250 rps)  │
    │  Instance 3: HEALTHY ✓ (250 rps)  │
    │  Instance 4: HEALTHY ✓ (250 rps)  │
    └───────────────────────────────────┘
              │    │    │    │
              ▼    ▼    ▼    ▼
          ┌────┐┌────┐┌────┐┌────┐
          │ I1 ││ I2 ││ I3 ││ I4 │
          └────┘└────┘└────┘└────┘

    Each instance handles 250 requests/second instead of one handling 1000.
```

### What Happens When an Instance Dies?

```
BEFORE (Instance 3 crashes):
              │    │    │    │
              ▼    ▼    ▼    ▼
          ┌────┐┌────┐┌────┐┌────┐
          │ I1 ││ I2 ││ I3 ││ I4 │
          │ ✓  ││ ✓  ││ ✗  ││ ✓  │  ← Instance 3 fails health check!
          └────┘└────┘└────┘└────┘

AFTER (Load balancer removes Instance 3):
              │    │         │
              ▼    ▼         ▼
          ┌────┐┌────┐    ┌────┐
          │ I1 ││ I2 │    │ I4 │
          │ ✓  ││ ✓  │    │ ✓  │  ← Traffic redistributed!
          └────┘└────┘    └────┘
          (333)  (333)    (333)   ← Each now handles 333 rps

USER IMPACT: ZERO! Users never knew Instance 3 existed.
The load balancer routed around it automatically.
```

### Session Management Across Instances

```
PROBLEM: User logs in on Instance 1, next request goes to Instance 2

Request 1 (Login):           Request 2 (Get Profile):
    │                            │
    ▼                            ▼
┌────────┐                   ┌────────┐
│  LB    │                   │  LB    │
└───┬────┘                   └───┬────┘
    ▼                            ▼
┌────────┐                   ┌────────┐
│ Inst 1 │ ← "Login OK!"    │ Inst 2 │ ← "Who are you??" 😱
└────────┘                   └────────┘


SOLUTION: External session store (Redis)

Request 1 (Login):           Request 2 (Get Profile):
    │                            │
    ▼                            ▼
┌────────┐                   ┌────────┐
│  LB    │                   │  LB    │
└───┬────┘                   └───┬────┘
    ▼                            ▼
┌────────┐                   ┌────────┐
│ Inst 1 │                   │ Inst 2 │
│ Save   │──┐                │ Check  │──┐
│ session│  │                │ session│  │
└────────┘  ▼                └────────┘  ▼
         ┌───────┐                    ┌───────┐
         │ Redis │ ← Store here      │ Redis │ ← Found it! ✓
         │session│                    │session│
         │ :abc  │                    │ :abc  │
         └───────┘                    └───────┘
```

### Deployment to Multiple Instances

```
DEPLOY NEW VERSION (v2) to 4 instances:

OPTION 1: All at once (downtime!)
┌────┐┌────┐┌────┐┌────┐     ┌────┐┌────┐┌────┐┌────┐
│ v1 ││ v1 ││ v1 ││ v1 │ ──▶ │ v2 ││ v2 ││ v2 ││ v2 │
└────┘└────┘└────┘└────┘     └────┘└────┘└────┘└────┘
         ↑ DOWNTIME ↑            All updated at once

OPTION 2: Rolling update (zero downtime!) ← Covered in Chapter 5.6
Step 1: ┌────┐┌────┐┌────┐┌────┐
         │ v2 ││ v1 ││ v1 ││ v1 │  ← Update one at a time
         └────┘└────┘└────┘└────┘
Step 2: ┌────┐┌────┐┌────┐┌────┐
         │ v2 ││ v2 ││ v1 ││ v1 │
         └────┘└────┘└────┘└────┘
Step 3: ┌────┐┌────┐┌────┐┌────┐
         │ v2 ││ v2 ││ v2 ││ v1 │
         └────┘└────┘└────┘└────┘
Step 4: ┌────┐┌────┐┌────┐┌────┐
         │ v2 ││ v2 ││ v2 ││ v2 │  ← Done! Zero downtime!
         └────┘└────┘└────┘└────┘
```

---

## Code Examples

### Python (Stateless App with Redis Session Store)

```python
# app.py — Stateless application ready for multiple instances
from flask import Flask, jsonify, request
import redis
import os
import uuid
import json

app = Flask(__name__)

# Shared Redis for sessions — ALL instances use the same Redis
session_store = redis.Redis(
    host=os.environ.get("REDIS_HOST", "redis-server"),  # Shared server
    port=6379,
    decode_responses=True
)

@app.route("/api/login", methods=["POST"])
def login():
    """Login creates a session token stored in Redis (not locally!)."""
    data = request.json
    # ... validate credentials ...
    
    # Create session in REDIS (shared across all instances)
    session_id = str(uuid.uuid4())
    session_data = {"user_id": data["user_id"], "role": "user"}
    session_store.setex(
        f"session:{session_id}", 
        3600,  # Expires in 1 hour
        json.dumps(session_data)
    )
    
    return jsonify({"token": session_id})

@app.route("/api/profile")
def get_profile():
    """Any instance can serve this — session is in Redis, not local memory."""
    token = request.headers.get("Authorization", "").replace("Bearer ", "")
    
    # Look up session from shared Redis
    session_data = session_store.get(f"session:{token}")
    if not session_data:
        return jsonify({"error": "Unauthorized"}), 401
    
    session = json.loads(session_data)
    return jsonify({"user_id": session["user_id"], "role": session["role"]})

@app.route("/health")
def health():
    """Health check endpoint — load balancer calls this every 10 seconds."""
    try:
        session_store.ping()
        return jsonify({"status": "healthy", "instance": os.environ.get("HOSTNAME")}), 200
    except Exception:
        return jsonify({"status": "unhealthy"}), 503

# Run on each instance:
# Instance 1: gunicorn app:app --bind 0.0.0.0:8000
# Instance 2: gunicorn app:app --bind 0.0.0.0:8000  (different machine)
# Instance 3: gunicorn app:app --bind 0.0.0.0:8000  (different machine)
```

### Java (Spring Boot Stateless with Redis Sessions)

```java
// Application ready for horizontal scaling with Spring Session + Redis

// pom.xml dependency: spring-boot-starter-data-redis, spring-session-data-redis

// application.yml:
// spring:
//   session:
//     store-type: redis        ← Sessions stored in Redis, not memory!
//   redis:
//     host: redis-server       ← Shared Redis instance
//     port: 6379
// server:
//   port: 8080

@RestController
public class ProfileController {
    
    @GetMapping("/api/profile")
    public ResponseEntity<Map<String, Object>> getProfile(HttpSession session) {
        // Spring Session automatically stores this in Redis
        // Any instance can read it!
        Object userId = session.getAttribute("user_id");
        if (userId == null) {
            return ResponseEntity.status(401).body(Map.of("error", "Unauthorized"));
        }
        return ResponseEntity.ok(Map.of(
            "user_id", userId,
            "instance", System.getenv("HOSTNAME")  // Shows which instance handled this
        ));
    }
    
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        return ResponseEntity.ok(Map.of(
            "status", "healthy",
            "instance", System.getenv("HOSTNAME")
        ));
    }
}
```

### Nginx Load Balancer Configuration

```nginx
# /etc/nginx/nginx.conf — Load balancing across multiple instances
http {
    # Define the group of backend instances
    upstream backend_pool {
        # Round-robin by default
        server 10.0.2.10:8080;    # Instance 1
        server 10.0.2.11:8080;    # Instance 2
        server 10.0.2.12:8080;    # Instance 3
        server 10.0.2.13:8080;    # Instance 4
        
        # Health check: remove unhealthy servers automatically
        # max_fails=3: after 3 failed attempts, mark as down
        # fail_timeout=30s: try again after 30 seconds
    }

    server {
        listen 80;
        server_name api.myapp.com;

        location / {
            proxy_pass http://backend_pool;  # Distribute across all instances
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            
            # Timeouts for backend connections
            proxy_connect_timeout 5s;
            proxy_read_timeout 30s;
        }
        
        location /health {
            proxy_pass http://backend_pool;
            access_log off;  # Don't log health checks
        }
    }
}
```

---

## Infrastructure Example

### Docker Compose (Multiple Instances Locally)

```yaml
# docker-compose.yml — Running 3 instances with load balancer
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app1
      - app2
      - app3

  app1:
    build: ./app
    environment:
      - INSTANCE_ID=1
      - REDIS_HOST=redis
      - DB_HOST=postgres
    expose:
      - "8080"

  app2:
    build: ./app
    environment:
      - INSTANCE_ID=2
      - REDIS_HOST=redis
      - DB_HOST=postgres
    expose:
      - "8080"

  app3:
    build: ./app
    environment:
      - INSTANCE_ID=3
      - REDIS_HOST=redis
      - DB_HOST=postgres
    expose:
      - "8080"

  redis:
    image: redis:7-alpine
    expose:
      - "6379"

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### Kubernetes Deployment (Production)

```yaml
# deployment.yaml — Run 5 replicas of your app in Kubernetes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
spec:
  replicas: 5  # ← 5 instances of the same app!
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: app
        image: myapp:v2.1.0
        ports:
        - containerPort: 8080
        env:
        - name: REDIS_HOST
          value: "redis-service"
        - name: DB_HOST
          value: "postgres-service"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        # Health checks — Kubernetes restarts unhealthy pods
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# Service — acts as the load balancer within Kubernetes
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP  # Internal load balancing
```

### AWS Auto Scaling Group

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS Auto Scaling Group                            │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Application Load Balancer (ALB)                              │  │
│  │  Endpoint: myapp-lb-123.us-east-1.elb.amazonaws.com          │  │
│  └──────────────────────────┬───────────────────────────────────┘  │
│                             │                                       │
│        ┌────────────────────┼────────────────────┐                 │
│        ▼                    ▼                    ▼                  │
│  ┌───────────┐       ┌───────────┐       ┌───────────┐            │
│  │  EC2 #1   │       │  EC2 #2   │       │  EC2 #3   │            │
│  │  t3.medium│       │  t3.medium│       │  t3.medium│            │
│  │  (v2.1.0) │       │  (v2.1.0) │       │  (v2.1.0) │            │
│  └───────────┘       └───────────┘       └───────────┘            │
│                                                                     │
│  Min instances: 2                                                   │
│  Max instances: 10                                                  │
│  Desired: 3                                                         │
│  Scale up when: CPU > 70% for 5 min                                │
│  Scale down when: CPU < 30% for 10 min                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Netflix — Thousands of Instances

```
Netflix runs THOUSANDS of instances of each microservice:

┌────────────────────────────────────────────────────┐
│  Service              │  Instances   │  Traffic     │
├───────────────────────┼──────────────┼──────────────┤
│  API Gateway (Zuul)   │  ~800        │  2B req/day  │
│  User Service         │  ~200        │  500M/day    │
│  Recommendation Svc   │  ~500        │  1B/day      │
│  Video Streaming      │  ~2000       │  15B/day     │
│  ...                  │  ...         │  ...         │
└────────────────────────────────────────────────────┘

Each service auto-scales based on traffic:
- Weekend evening (peak): 2000 streaming instances
- Tuesday 3 AM (low):    200 streaming instances
- Money saved by scaling DOWN: millions $/year
```

### Instagram — Scaling Django

```
Instagram (2012, pre-Facebook acquisition):
- 30+ million users
- 25+ photos/second uploaded
- Django + PostgreSQL + Redis + Memcached

How they scaled:
┌───────────────────────────────────────────────────────┐
│  25 Django app servers (each on Amazon EC2)           │
│  ├── Gunicorn with 16 workers per server              │
│  ├── Total: 400 worker processes handling requests    │
│  ├── All stateless (sessions in Redis)                │
│  └── Behind Amazon ELB (Elastic Load Balancer)        │
│                                                       │
│  When traffic grew:                                   │
│  → Add more EC2 instances (horizontal scaling)        │
│  → NOT: buy bigger servers (vertical scaling)         │
└───────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

### 1. Storing State in Application Memory
❌ **Mistake**: In-memory session store, local file uploads, in-memory counters.
✅ **Fix**: Move ALL state to external stores (Redis, S3, Database).

```python
# BAD: In-memory (works on 1 instance, fails on many)
sessions = {}  # This dict exists only on THIS instance!

# GOOD: Redis (shared across all instances)
redis_client.set(f"session:{token}", json.dumps(data))
```

### 2. Writing Files to Local Disk
❌ **Mistake**: User uploads a file → saved to `/tmp/uploads/` on Instance 1 → request to serve it hits Instance 2 → FILE NOT FOUND!
✅ **Fix**: Upload to shared storage (S3, NFS, or a shared volume).

### 3. No Health Checks
❌ **Mistake**: Load balancer sends traffic to a crashed instance → users get 502 errors.
✅ **Fix**: Implement `/health` endpoint; configure load balancer to check every 10s.

### 4. Database Connection Exhaustion
❌ **Mistake**: 10 instances × 20 connections each = 200 connections to PostgreSQL (default max is 100!).
✅ **Fix**: Use connection pooling (PgBouncer) or reduce connections per instance.

```
WITHOUT PgBouncer:
Instance 1 ──20 connections──┐
Instance 2 ──20 connections──┼──▶ PostgreSQL (max_connections=100)
Instance 3 ──20 connections──┤     ← FULL at 5 instances!
Instance 4 ──20 connections──┤
Instance 5 ──20 connections──┘

WITH PgBouncer (connection multiplexer):
Instance 1 ──20 connections──┐
Instance 2 ──20 connections──┼──▶ PgBouncer ──20 connections──▶ PostgreSQL
Instance 3 ──20 connections──┤    (multiplexes                   (stays at 20!)
Instance 4 ──20 connections──┤     100 → 20)
Instance 5 ──20 connections──┘
```

### 5. Logging to Local Files
❌ **Mistake**: Logs written to `/var/log/app.log` on each instance — need to SSH into each to debug.
✅ **Fix**: Send logs to a centralized service (ELK, CloudWatch, Datadog).

---

## When to Use / When NOT to Use

### ✅ Use Multiple Instances When:

| Criteria | Why |
|----------|-----|
| **Single server can't handle traffic** | Distribute load across machines |
| **Need high availability** | If one instance dies, others keep serving |
| **Zero-downtime deployments** | Update instances one at a time |
| **Traffic is unpredictable** | Auto-scale up during spikes, down during quiet times |
| **SLA requires 99.9%+ uptime** | Single server = single point of failure |

### ❌ Don't Use When:

| Criteria | Why |
|----------|-----|
| **Traffic easily fits on one server** | Added complexity without benefit |
| **Application is stateful and hard to refactor** | Fix statefulness first |
| **Budget is very tight** | Load balancer + multiple servers costs more |
| **No automated deployment pipeline** | Manual deployment to N servers is painful |

---

## Scaling Math

```
HOW MANY INSTANCES DO YOU NEED?

Given:
- 1 instance handles 500 requests/second
- Expected traffic: 2,000 requests/second peak
- Want 50% headroom (don't run at 100%)

Calculation:
  Instances needed = (Peak Traffic / Capacity per Instance) × Safety Factor
  Instances needed = (2000 / 500) × 1.5 = 6 instances

  Add 1 extra for redundancy (what if one dies?)
  Total: 7 instances

┌──────────────────────────────────────────────────┐
│  Capacity planning:                               │
│                                                  │
│  Instances:  7                                   │
│  Each:       t3.medium ($30/month)               │
│  Monthly:    7 × $30 = $210                      │
│                                                  │
│  vs. one giant server:                           │
│  1 × c5.4xlarge (16 CPU, 32 GB) = $500/month    │
│                                                  │
│  Multiple small > one big:                       │
│  ✓ Cheaper ($210 vs $500)                        │
│  ✓ Fault tolerant (1 dies → 6 remain)            │
│  ✓ Can scale further (add more instances)        │
│  ✓ Zero-downtime deployments                     │
└──────────────────────────────────────────────────┘
```

---

## Key Takeaways

- **Horizontal scaling** means running multiple copies of your application behind a load balancer — not making one server bigger.
- **Your application must be stateless** — no in-memory sessions, no local file storage, no machine-specific state.
- **The load balancer** distributes traffic and removes failed instances automatically — users never see downtime.
- **All shared state goes to external stores** — Redis for sessions/cache, S3 for files, PostgreSQL for data.
- **Health checks are critical** — the load balancer must know which instances are alive.
- **Watch database connections** — N instances × M connections can overwhelm your database. Use connection pooling.
- **This is how every major website operates** — from Instagram (25 Django servers) to Netflix (thousands of instances per service).

---

## What's Next?

Running multiple instances in one datacenter is great, but what happens when you need to serve users across the globe — with low latency in every country? That's **Chapter 5.4: Multi-Region Deployment — Serving Users Across the Globe**, where we replicate your entire stack across continents.
