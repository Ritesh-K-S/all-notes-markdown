# Stage 3: Load Balancer + Multiple Servers — 1K to 10K Users

> **What you'll learn**: How to run multiple copies of your application behind a load balancer, achieving both higher capacity AND high availability (if one server dies, others keep serving).

---

## Real-Life Analogy

Your chai stall is now a **chai shop** with a growing line of customers. Even with the dishwasher working separately (separate DB), one chai maker can't keep up.

**Solution**: You hire 3 chai makers, all making the same chai using the same recipe. At the door, you place a **host/receptionist** who directs each customer to whichever chai maker is least busy.

If one chai maker calls in sick, the host simply stops sending customers to that counter. Customers never even notice — they still get their chai from the other two.

That host is your **Load Balancer**. The chai makers are your **application server instances**.

---

## Core Concept Explained Step-by-Step

### The Architecture

```
                         Internet
                            │
                            ▼
                   ┌─────────────────┐
                   │  LOAD BALANCER  │
                   │  (Single entry  │
                   │   point)        │
                   │                 │
                   │  Public IP:     │
                   │  203.0.113.1    │
                   └────────┬────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │  App Server  │ │  App Server  │ │  App Server  │
     │  Instance 1  │ │  Instance 2  │ │  Instance 3  │
     │  10.0.1.1    │ │  10.0.1.2    │ │  10.0.1.3    │
     │              │ │              │ │              │
     │  Same code   │ │  Same code   │ │  Same code   │
     │  Same config │ │  Same config │ │  Same config │
     └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
              │             │             │
              └─────────────┼─────────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │    DATABASE     │
                   │  (PostgreSQL)   │
                   │   10.0.2.1      │
                   └─────────────────┘
```

### What Does the Load Balancer Do?

```
User A ──▶ ┌─────────────────┐ ──▶ Server 1
User B ──▶ │  Load Balancer  │ ──▶ Server 2
User C ──▶ │                 │ ──▶ Server 3
User D ──▶ │  Distributes    │ ──▶ Server 1  (round-robin)
User E ──▶ │  traffic evenly │ ──▶ Server 2
User F ──▶ └─────────────────┘ ──▶ Server 3
```

The load balancer:
1. **Receives ALL incoming requests** (it's the only thing with a public IP)
2. **Picks a healthy server** to forward the request to
3. **Gets the response** from that server
4. **Sends it back** to the user

The user never knows which server handled their request.

### Two Major Benefits

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  BENEFIT 1: SCALABILITY (Handle more traffic)                │
│                                                              │
│  1 server  = 500 requests/second                             │
│  3 servers = 1,500 requests/second  (linear scaling!)        │
│  Need more? Just add more servers.                           │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  BENEFIT 2: HIGH AVAILABILITY (No single point of failure)   │
│                                                              │
│  Server 2 crashes:                                           │
│                                                              │
│  User ──▶ LB ──▶ Server 1  ✓ (still working)               │
│                ──▶ Server 2  ✗ (dead — LB stops sending)     │
│                ──▶ Server 3  ✓ (still working)               │
│                                                              │
│  Users experience ZERO downtime!                             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Health Checks: How the LB Knows Who's Alive

```
Every 10 seconds:

Load Balancer ──── GET /health ────▶ Server 1 ──── 200 OK ✓
                                     Server 2 ──── TIMEOUT ✗ (remove!)
                                     Server 3 ──── 200 OK ✓

LB's internal list:
┌────────────┬────────┬──────────────┐
│ Server     │ Status │ Last Check   │
├────────────┼────────┼──────────────┤
│ 10.0.1.1   │ UP ✓   │ 2s ago       │
│ 10.0.1.2   │ DOWN ✗ │ 12s ago      │
│ 10.0.1.3   │ UP ✓   │ 2s ago       │
└────────────┴────────┴──────────────┘
```

---

## How It Works Internally

### Load Balancing Algorithms

```
┌──────────────────────────────────────────────────────────────────────┐
│ ALGORITHM          │ HOW IT WORKS                │ BEST FOR           │
├────────────────────┼─────────────────────────────┼────────────────────┤
│ Round Robin        │ 1→2→3→1→2→3→...            │ Equal servers      │
│ Least Connections  │ Send to server with fewest  │ Varying request    │
│                    │ active connections           │ durations          │
│ IP Hash            │ Same user IP → same server  │ Session affinity   │
│ Weighted Round     │ Server with 2x CPU gets     │ Mixed hardware     │
│ Robin              │ 2x traffic                  │                    │
│ Random             │ Pick any healthy server     │ Large clusters     │
└────────────────────┴─────────────────────────────┴────────────────────┘
```

### The Stateless Requirement

For load balancing to work, your app servers must be **stateless** — they can't store user sessions in memory.

```
PROBLEM: Stateful servers (sessions in memory)

Request 1: User logs in ──▶ Server 1 (stores session in RAM)
Request 2: User loads page ──▶ Server 2 (no session here! → "Please log in again")

                    User: "Why do I keep getting logged out?!" 😡
```

```
SOLUTION: Externalize state (sessions in Redis/database)

Request 1: User logs in ──▶ Server 1 ──▶ Stores session in Redis
Request 2: User loads page ──▶ Server 2 ──▶ Reads session from Redis ✓

┌──────────┐      ┌──────────┐      ┌──────────┐
│ Server 1 │      │ Server 2 │      │ Server 3 │
└─────┬────┘      └─────┬────┘      └─────┬────┘
      │                  │                  │
      └──────────────────┼──────────────────┘
                         │
                    ┌────▼────┐
                    │  Redis  │  ← Shared session store
                    │(sessions│
                    │ + cache)│
                    └─────────┘
```

### SSL/TLS Termination at the Load Balancer

```
                    HTTPS (encrypted)           HTTP (plain, private network)
User ═══════════════════════════════▶ LB ──────────────────────────▶ App Servers

The load balancer:
1. Handles the expensive SSL/TLS encryption
2. Forwards plain HTTP to app servers (they don't need SSL)
3. This saves CPU on app servers (~15-20% reduction)
```

### What Happens During a Deploy?

```
Rolling deploy (zero downtime):

Step 1: Remove Server 1 from LB pool
        LB ──▶ Server 2, Server 3 only

Step 2: Deploy new code to Server 1
        (Server 1 starts with v2.0)

Step 3: Add Server 1 back to LB pool
        LB ──▶ Server 1 (v2.0), Server 2 (v1.0), Server 3 (v1.0)

Step 4: Repeat for Server 2, then Server 3

Result: Zero downtime! Users never see an error.
```

---

## Code Examples

### Python: Stateless App with Redis Sessions

```python
# app.py — Stateless application (works behind a load balancer)
from flask import Flask, jsonify, request, session
from flask_session import Session
import redis
import os

app = Flask(__name__)

# Sessions stored in Redis (NOT in server memory!)
# Any server can read any user's session
app.config["SESSION_TYPE"] = "redis"
app.config["SESSION_REDIS"] = redis.Redis(
    host="10.0.3.1",    # Shared Redis server
    port=6379,
    db=0
)
app.config["SECRET_KEY"] = os.environ["SECRET_KEY"]
Session(app)

@app.route("/api/login", methods=["POST"])
def login():
    data = request.get_json()
    # ... validate credentials ...
    session["user_id"] = data["user_id"]    # Stored in Redis!
    session["username"] = data["username"]
    return jsonify({"message": "Logged in"})

@app.route("/api/profile")
def profile():
    # Works regardless of which server handles this request!
    user_id = session.get("user_id")
    if not user_id:
        return jsonify({"error": "Not logged in"}), 401
    return jsonify({"user_id": user_id, "username": session["username"]})

@app.route("/health")
def health():
    """Health check endpoint — load balancer hits this every 10 seconds"""
    return jsonify({"status": "healthy", "server": os.environ.get("HOSTNAME")})
```

### Java: Stateless Spring Boot with Redis Sessions

```java
// Application with externalized sessions (works with load balancer)
@SpringBootApplication
@EnableRedisHttpSession    // Store sessions in Redis, not in-memory!
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}

// application.yml
// spring:
//   redis:
//     host: 10.0.3.1
//     port: 6379
//   session:
//     store-type: redis

@RestController
public class AuthController {

    @PostMapping("/api/login")
    public ResponseEntity<Map<String, String>> login(
            HttpSession session, @RequestBody LoginRequest request) {
        // Session stored in Redis — accessible from ANY server
        session.setAttribute("userId", request.getUserId());
        session.setAttribute("username", request.getUsername());
        return ResponseEntity.ok(Map.of("message", "Logged in"));
    }

    @GetMapping("/api/profile")
    public ResponseEntity<?> profile(HttpSession session) {
        String userId = (String) session.getAttribute("userId");
        if (userId == null) {
            return ResponseEntity.status(401).body(Map.of("error", "Not logged in"));
        }
        return ResponseEntity.ok(Map.of("userId", userId));
    }

    @GetMapping("/health")
    public Map<String, String> health() {
        return Map.of("status", "healthy",
                      "hostname", InetAddress.getLocalHost().getHostName());
    }
}
```

---

## Infrastructure Example

### Nginx as Load Balancer

```nginx
# /etc/nginx/nginx.conf — Nginx as a simple load balancer
http {
    # Define the pool of app servers
    upstream app_servers {
        # Round-robin by default
        server 10.0.1.1:8000;
        server 10.0.1.2:8000;
        server 10.0.1.3:8000;

        # Health check: mark server as down after 3 failures
        # Try again after 30 seconds
    }

    server {
        listen 80;
        listen 443 ssl;
        server_name myapp.com;

        ssl_certificate /etc/ssl/myapp.crt;
        ssl_certificate_key /etc/ssl/myapp.key;

        location / {
            proxy_pass http://app_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Timeout settings
            proxy_connect_timeout 5s;
            proxy_read_timeout 60s;

            # If one server fails, try the next
            proxy_next_upstream error timeout http_502 http_503;
        }
    }
}
```

### AWS Application Load Balancer (Terraform)

```hcl
# alb.tf — AWS Application Load Balancer with auto-scaling

resource "aws_lb" "app" {
  name               = "myapp-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
}

resource "aws_lb_target_group" "app" {
  name     = "myapp-targets"
  port     = 8000
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  # Health check configuration
  health_check {
    path                = "/health"
    interval            = 10
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.app.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# Auto Scaling Group: maintains 3 app servers
resource "aws_autoscaling_group" "app" {
  desired_capacity = 3
  min_size         = 2
  max_size         = 10

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  target_group_arns = [aws_lb_target_group.app.arn]

  tag {
    key                 = "Name"
    value               = "myapp-instance"
    propagate_at_launch = true
  }
}
```

### Docker Compose: Simulating Multiple Servers Locally

```yaml
# docker-compose.yml — Test load balancing locally
version: '3.8'
services:
  nginx-lb:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx-lb.conf:/etc/nginx/nginx.conf
    depends_on:
      - app1
      - app2
      - app3

  app1:
    build: .
    environment:
      - HOSTNAME=app-server-1
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
      - REDIS_URL=redis://redis:6379

  app2:
    build: .
    environment:
      - HOSTNAME=app-server-2
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
      - REDIS_URL=redis://redis:6379

  app3:
    build: .
    environment:
      - HOSTNAME=app-server-3
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
      - REDIS_URL=redis://redis:6379

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass

  redis:
    image: redis:7-alpine
```

---

## Real-World Example

### Netflix's Approach

Netflix runs **thousands of instances** behind their load balancers:
- They use **AWS Elastic Load Balancer (ELB)** at the edge
- Internal services use **Eureka** (service discovery) + **Ribbon** (client-side load balancing)
- Each microservice has 10-100 instances behind a load balancer

### Flipkart's Big Billion Days

During India's biggest sale event:
- Normal traffic: 50 servers behind the load balancer
- Sale day: Auto-scales to **500+ servers** within minutes
- The load balancer is the single entry point that makes this seamless

### GitHub

- Uses **HAProxy** as their load balancer
- Routes millions of `git push` and `git pull` requests across hundreds of backend servers
- If a server becomes unhealthy, HAProxy removes it in < 5 seconds

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Storing sessions in memory** | Users get logged out when hitting a different server | Use Redis/database for sessions |
| **Storing uploaded files on one server** | File available on server 1, not on server 2 | Use shared storage (S3, NFS) |
| **No health checks** | LB sends traffic to dead servers → errors | Always configure `/health` endpoint |
| **Load balancer as single point of failure** | If LB dies, everything is down | Use HA pair or managed LB (AWS ALB) |
| **Not setting connection draining** | In-flight requests get killed during deploys | Enable connection draining (wait for active requests) |
| **Forgetting X-Forwarded-For header** | App sees LB's IP instead of client's real IP | Pass X-Forwarded-For in proxy config |
| **Sticky sessions without need** | Uneven load distribution | Only use sticky sessions if absolutely necessary |

---

## When to Use / When NOT to Use

### ✅ Add Load Balancer + Multiple Servers When:
- Single server can't handle the traffic (CPU > 70% sustained)
- You need high availability (zero downtime)
- You want to deploy without downtime (rolling deploys)
- Your traffic is growing and predictable
- You're serving paying customers who expect reliability

### ❌ Don't Add Yet If:
- You have fewer than 500-1000 daily active users
- You haven't exhausted vertical scaling (bigger server)
- Your bottleneck is the database, not the app server
- You can't afford the complexity of stateless design
- Your app has heavy local state that's hard to externalize

---

## Key Takeaways

1. **Load balancer = single entry point that distributes traffic across multiple servers** — users never know which server they hit
2. **Your app must be stateless** — no sessions in memory, no files on local disk. Put everything in Redis/S3/database.
3. **Health checks are critical** — the LB must know which servers are alive
4. **This gives you BOTH scalability AND availability** — handle more traffic AND survive server failures
5. **SSL termination at the LB** saves CPU on app servers and centralizes certificate management
6. **Rolling deploys become possible** — update servers one at a time with zero downtime
7. **The database becomes the next bottleneck** — all servers now hit one DB (solved in Stage 5)

---

## What's Next?

With multiple app servers, you can handle 10K users. But every user request still hits the database for every page load. At this scale, the database starts to struggle. The next step is adding **Caching and a CDN** to dramatically reduce load on both servers and database.

Next: [04-stage4-caching-cdn.md](./04-stage4-caching-cdn.md)
