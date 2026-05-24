# ⚡ Load Balancer — Complete Guide (All Architecture Types)

> Everything about Load Balancers: What, Why, How, Tools, Configs, Flows & Real-World Setup

---

## 📌 TABLE OF CONTENTS

1. [What is a Load Balancer?](#1-what-is-a-load-balancer)
2. [Why Do We Need It?](#2-why-do-we-need-it)
3. [Key Components](#3-key-components)
4. [LB Algorithms](#4-lb-algorithms)
5. [Popular LB Tools](#5-popular-lb-tools)
6. [SETUP 1 — Monolithic Coupled App (UI + API + DB in one app)](#6-setup-1--monolithic-coupled-app)
7. [SETUP 2 — Detached Monolithic (UI & API on separate servers)](#7-setup-2--detached-monolithic)
8. [SETUP 3 — Microservices (Multiple services, multiple instances, multiple servers)](#8-setup-3--microservices)
9. [Health Checks](#9-health-checks)
10. [SSL Termination](#10-ssl-termination)
11. [Sticky Sessions](#11-sticky-sessions)
12. [Summary Comparison Table](#12-summary-comparison-table)

---

## 1. What is a Load Balancer?

A Load Balancer (LB) is a server/device that sits BETWEEN the client (browser) and your application
servers. Its job is to DISTRIBUTE incoming traffic across multiple servers so that:
- No single server gets overwhelmed
- If one server dies, traffic goes to healthy ones
- Users get faster response times

```
Simple Idea:

    [ 1000 Users ]
          |
    [ Load Balancer ]          <-- Distributes traffic
       /     |     \
  [Server1] [Server2] [Server3]   <-- Each handles ~333 users
```

---

## 2. Why Do We Need It?

| Problem Without LB                | Solution With LB                          |
|------------------------------------|-------------------------------------------|
| Single server crashes = site down  | Traffic shifts to healthy servers          |
| One server handles ALL traffic     | Traffic split across many servers          |
| Can't scale horizontally           | Add more servers behind LB easily         |
| No failover                        | Automatic failover via health checks      |
| Uneven load on servers             | Even distribution via algorithms          |

---

## 3. Key Components

```
┌──────────────────────────────────────────────────────────────────┐
│                     LOAD BALANCER ECOSYSTEM                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. LISTENER        → The "ear" of LB. Listens on a port        │
│                        (e.g., port 80 for HTTP, 443 for HTTPS)  │
│                                                                  │
│  2. TARGET GROUP    → A group of servers (targets) that will     │
│                        receive the traffic                       │
│                        e.g., Target Group "web-servers" has:     │
│                             - 10.0.1.10:3000                     │
│                             - 10.0.1.11:3000                     │
│                             - 10.0.1.12:3000                     │
│                                                                  │
│  3. RULES           → Conditions that decide WHICH target group  │
│                        gets the request                          │
│                        e.g., /api/* → API target group           │
│                             /*     → UI target group             │
│                                                                  │
│  4. HEALTH CHECK    → LB pings each server every X seconds       │
│                        to check if it is alive                   │
│                        e.g., GET /health every 30s               │
│                        If 3 checks fail → server marked unhealthy│
│                                                                  │
│  5. BACKEND SERVERS → The actual app instances receiving traffic │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. LB Algorithms

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ ALGORITHM           │ HOW IT WORKS                                         │
├─────────────────────┼───────────────────────────────────────────────────────┤
│ Round Robin         │ Request 1 → Server A                                 │
│                     │ Request 2 → Server B                                 │
│                     │ Request 3 → Server C                                 │
│                     │ Request 4 → Server A  (cycles back)                  │
│                     │ Simple rotation. Most common.                        │
├─────────────────────┼───────────────────────────────────────────────────────┤
│ Weighted Round Robin│ Server A (weight 5) gets 5 requests                  │
│                     │ Server B (weight 3) gets 3 requests                  │
│                     │ Server C (weight 2) gets 2 requests                  │
│                     │ Use when servers have different capacity.            │
├─────────────────────┼───────────────────────────────────────────────────────┤
│ Least Connections   │ Sends request to the server with the FEWEST         │
│                     │ active connections right now.                        │
│                     │ Good for long-lived connections (WebSocket).         │
├─────────────────────┼───────────────────────────────────────────────────────┤
│ IP Hash             │ Hash of client IP decides the server.               │
│                     │ Same user always goes to same server.               │
│                     │ Good for session-based apps (sticky sessions).      │
├─────────────────────┼───────────────────────────────────────────────────────┤
│ Random              │ Picks a random server. Simple but less predictable. │
└─────────────────────┴───────────────────────────────────────────────────────┘
```

---

## 5. Popular LB Tools

### A. Nginx (Most Popular — Software LB)
- Open source, very fast, widely used
- Acts as reverse proxy + load balancer
- Config file based
- Used by Netflix, Airbnb, Dropbox

### B. HAProxy (High Availability Proxy)
- Purpose-built for load balancing
- Extremely fast and lightweight
- Used by GitHub, Stack Overflow, Reddit
- TCP + HTTP load balancing

### C. AWS ALB (Application Load Balancer)
- Cloud-managed LB by Amazon
- Layer 7 (HTTP/HTTPS) — can route based on URL path, headers, etc.
- Has built-in target groups, listeners, rules
- No server to manage — fully managed service

### D. AWS NLB (Network Load Balancer)
- Layer 4 (TCP/UDP) — ultra fast, millions of requests/sec
- Use for non-HTTP traffic or extreme performance needs

### E. Traefik
- Modern LB built for microservices & containers
- Auto-discovers services in Docker/Kubernetes
- Built-in Let's Encrypt SSL

### F. Envoy (used inside Service Mesh like Istio)
- Sidecar proxy for microservices
- Used alongside Kubernetes

---

---

# 6. SETUP 1 — Monolithic Coupled App

## (UI + API + DB Connection — All in ONE codebase, ONE app)

### What is this?
```
Your single app does EVERYTHING:
  - Serves the HTML/CSS/JS (UI)
  - Handles API requests (/api/users, /api/orders)
  - Connects to Database directly

Example: A Node.js Express app that serves React build AND has API routes AND connects to MongoDB
```

### The Problem
```
ONE server running your app:

    [ 50,000 Users ]
          |
    [ Server: 10.0.1.10:3000 ]   ← Dies under load! 💀
          |
    [ MongoDB: 10.0.2.50:27017 ]
```

### The Solution — Add LB + Multiple Instances

```
FULL FLOW DIAGRAM:
==================

    [ Users from Internet ]
            |
            | (DNS: www.myapp.com → 52.20.100.1)
            ▼
    ┌──────────────────────┐
    │   LOAD BALANCER       │
    │   (Nginx)             │
    │   IP: 52.20.100.1     │
    │   Listening: Port 80  │
    │   Listening: Port 443 │
    └──────────┬───────────┘
               │
               │  (Distributes via Round Robin)
               │
     ┌─────────┼─────────────┐
     │         │             │
     ▼         ▼             ▼
  ┌────────┐ ┌────────┐  ┌────────┐
  │ App 1  │ │ App 2  │  │ App 3  │
  │10.0.1. │ │10.0.1. │  │10.0.1. │
  │10:3000 │ │11:3000 │  │12:3000 │
  │        │ │        │  │        │
  │ UI +   │ │ UI +   │  │ UI +   │
  │ API +  │ │ API +  │  │ API +  │
  │ DB conn│ │ DB conn│  │ DB conn│
  └───┬────┘ └───┬────┘  └───┬────┘
      │          │            │
      └──────────┼────────────┘
                 │
                 ▼
        ┌────────────────┐
        │   MongoDB       │
        │  10.0.2.50:27017│
        │  (Single DB or  │
        │   Replica Set)  │
        └────────────────┘
```

### Step-by-Step Flow

```
1. User types www.myapp.com in browser
2. DNS resolves www.myapp.com → 52.20.100.1 (LB's public IP)
3. Browser sends HTTP request to 52.20.100.1:80
4. Nginx LB receives the request on port 80
5. Nginx checks its upstream server list:
     - 10.0.1.10:3000 (healthy ✅)
     - 10.0.1.11:3000 (healthy ✅)
     - 10.0.1.12:3000 (healthy ✅)
6. Using Round Robin, sends 1st request → 10.0.1.10:3000
7. App 1 processes the request (serves UI or handles API)
8. App 1 queries MongoDB at 10.0.2.50:27017
9. Response goes back: App 1 → Nginx LB → User's browser
10. Next request from another user → 10.0.1.11:3000 (Round Robin)
```

### Nginx Config for Monolithic Coupled App

```nginx
# File: /etc/nginx/nginx.conf

# ─── STEP 1: Define the TARGET GROUP (upstream block) ───
# This is like AWS "Target Group" — a pool of backend servers

upstream monolith_app {
    # Algorithm: Round Robin (default)
    # Each server gets requests in rotation

    server 10.0.1.10:3000;    # App Instance 1
    server 10.0.1.11:3000;    # App Instance 2
    server 10.0.1.12:3000;    # App Instance 3

    # Optional: If one server is more powerful, give it more weight
    # server 10.0.1.10:3000 weight=3;
    # server 10.0.1.11:3000 weight=2;
    # server 10.0.1.12:3000 weight=1;
}

# ─── STEP 2: Define the LISTENER (server block) ───
# This listens for incoming requests on port 80

server {
    listen 80;                          # Listen on port 80 (HTTP)
    server_name www.myapp.com;          # Domain name

    # ─── STEP 3: Define the RULE (location block) ───
    # All requests (/) go to our upstream target group

    location / {
        proxy_pass http://monolith_app;            # Forward to target group
        proxy_set_header Host $host;                # Pass original host header
        proxy_set_header X-Real-IP $remote_addr;    # Pass client's real IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ─── HEALTH CHECK (Nginx Plus or use passive checks) ───
    # Nginx OSS does passive health checks by default:
    # If a server returns error, it's temporarily marked as down
}
```

### HAProxy Config for Monolithic Coupled App

```haproxy
# File: /etc/haproxy/haproxy.cfg

global
    log         127.0.0.1 local0
    maxconn     4096

defaults
    mode        http          # Layer 7 (HTTP mode)
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms
    option httplog
    option dontlognull

# ─── LISTENER (frontend) ───
# Like AWS ALB Listener — listens on port 80

frontend http_front
    bind *:80                                 # Listen on port 80
    default_backend monolith_servers          # Send to target group

# ─── TARGET GROUP (backend) ───
# Like AWS Target Group — pool of servers

backend monolith_servers
    balance roundrobin                         # Algorithm: Round Robin
    option httpchk GET /health                 # Health check: GET /health

    server app1 10.0.1.10:3000 check inter 5s fall 3 rise 2
    server app2 10.0.1.11:3000 check inter 5s fall 3 rise 2
    server app3 10.0.1.12:3000 check inter 5s fall 3 rise 2

    # check       → Enable health checks
    # inter 5s    → Check every 5 seconds
    # fall 3      → Mark unhealthy after 3 failed checks
    # rise 2      → Mark healthy after 2 successful checks
```

### AWS ALB Setup for Monolithic

```
In AWS Console / CLI:

STEP 1: Create Target Group
  Name: monolith-tg
  Protocol: HTTP
  Port: 3000
  Health Check Path: /health
  Health Check Interval: 30s
  Healthy Threshold: 3
  Unhealthy Threshold: 3

STEP 2: Register Targets (your EC2 instances)
  - i-abc123 (10.0.1.10:3000)
  - i-def456 (10.0.1.11:3000)
  - i-ghi789 (10.0.1.12:3000)

STEP 3: Create ALB
  Name: myapp-alb
  Scheme: internet-facing
  Listeners:
    - Port 80 (HTTP) → Forward to monolith-tg
    - Port 443 (HTTPS) → Forward to monolith-tg (with SSL cert)

STEP 4: DNS
  Create CNAME: www.myapp.com → myapp-alb-1234567.us-east-1.elb.amazonaws.com
```

---

---

# 7. SETUP 2 — Detached Monolithic

## (UI on separate servers, API on separate servers)

### What is this?
```
The frontend (React/Angular build) is served by its own servers.
The backend (Express/Spring API) runs on its own servers.
They are SEPARATE deployments but still "monolithic" (one big API, one big UI).

UI Server  → Serves HTML/CSS/JS (static files or SSR)
API Server → Handles /api/* routes, connects to DB
```

### Why Separate?
```
- Scale UI and API independently
  (maybe UI needs 2 servers, API needs 5 servers)
- Different teams can deploy independently
- UI can be cached on CDN, API cannot
- Different resource needs (UI = CPU light, API = CPU heavy)
```

### FULL FLOW DIAGRAM

```
    [ Users from Internet ]
            |
            | (DNS: www.myapp.com → 52.20.100.1)
            ▼
    ┌────────────────────────────┐
    │     MAIN LOAD BALANCER     │
    │         (Nginx / ALB)      │
    │      IP: 52.20.100.1       │
    │    Listening: Port 80/443  │
    │                            │
    │   RULES:                   │
    │   ┌──────────────────────┐ │
    │   │ /api/*  → API TG     │ │
    │   │ /*      → UI TG      │ │
    │   └──────────────────────┘ │
    └─────────┬──────────────────┘
              │
     ┌────────┴──────────┐
     │ PATH-BASED ROUTING│
     │                   │
     ▼                   ▼
  ┌──────────┐      ┌──────────┐
  │ UI       │      │ API      │
  │ TARGET   │      │ TARGET   │
  │ GROUP    │      │ GROUP    │
  └────┬─────┘      └────┬─────┘
       │                  │
  ┌────┼────┐        ┌────┼────┐
  │    │    │        │    │    │
  ▼    ▼    ▼        ▼    ▼    ▼

┌────┐┌────┐┌────┐ ┌────┐┌────┐┌────┐
│UI-1││UI-2││UI-3│ │API1││API2││API3│
│    ││    ││    │ │    ││    ││    │
│10. ││10. ││10. │ │10. ││10. ││10. │
│0.1.││0.1.││0.1.│ │0.2.││0.2.││0.2.│
│20: ││21: ││22: │ │30: ││31: ││32: │
│8080││8080││8080│ │4000││4000││4000│
│    ││    ││    │ │    ││    ││    │
│Reac││Reac││Reac│ │Expr││Expr││Expr│
│t   ││t   ││t   │ │ess ││ess ││ess │
│App ││App ││App │ │API ││API ││API │
└────┘└────┘└────┘ └──┬─┘└──┬─┘└──┬─┘
                      │     │     │
                      └─────┼─────┘
                            │
                            ▼
                   ┌────────────────┐
                   │   PostgreSQL    │
                   │  10.0.3.50:5432│
                   │   (Database)   │
                   └────────────────┘
```

### Step-by-Step Flow

```
FLOW 1 — User loads the website (UI request):
==============================================
1. User types www.myapp.com in browser
2. DNS → 52.20.100.1 (LB IP)
3. Browser sends GET / to LB
4. LB checks rules:
     Path "/"  matches  "/*"  → route to UI Target Group
5. LB picks UI server (Round Robin): 10.0.1.20:8080
6. UI server returns HTML + CSS + JS (React build)
7. Browser renders the UI

FLOW 2 — UI makes API call (API request):
==========================================
8. React app in browser calls: fetch("https://www.myapp.com/api/users")
9. Request goes to LB at 52.20.100.1
10. LB checks rules:
     Path "/api/users"  matches  "/api/*"  → route to API Target Group
11. LB picks API server (Round Robin): 10.0.2.30:4000
12. API server processes request, queries PostgreSQL at 10.0.3.50:5432
13. Response: DB → API server → LB → Browser
14. React app displays the data
```

### Nginx Config for Detached Monolithic

```nginx
# File: /etc/nginx/nginx.conf

# ─── TARGET GROUP 1: UI Servers ───
upstream ui_servers {
    server 10.0.1.20:8080;    # UI Instance 1
    server 10.0.1.21:8080;    # UI Instance 2
    server 10.0.1.22:8080;    # UI Instance 3
}

# ─── TARGET GROUP 2: API Servers ───
upstream api_servers {
    server 10.0.2.30:4000;    # API Instance 1
    server 10.0.2.31:4000;    # API Instance 2
    server 10.0.2.32:4000;    # API Instance 3
}

# ─── LISTENER ───
server {
    listen 80;
    server_name www.myapp.com;

    # ─── RULE 1: API requests → API Target Group ───
    # IMPORTANT: More specific paths MUST come first!
    location /api/ {
        proxy_pass http://api_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout settings for API (may take longer)
        proxy_read_timeout 60s;
        proxy_connect_timeout 10s;
    }

    # ─── RULE 2: Everything else → UI Target Group ───
    location / {
        proxy_pass http://ui_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Cache static assets
        proxy_cache_valid 200 1h;
    }
}
```

### HAProxy Config for Detached Monolithic

```haproxy
# File: /etc/haproxy/haproxy.cfg

global
    log         127.0.0.1 local0
    maxconn     4096

defaults
    mode        http
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms

# ─── LISTENER ───
frontend http_front
    bind *:80

    # ─── RULES: Path-based routing ───
    # ACL = Access Control List (a condition)
    acl is_api path_beg /api/

    # If path starts with /api/ → send to API backend
    use_backend api_servers if is_api

    # Everything else → send to UI backend
    default_backend ui_servers

# ─── TARGET GROUP: UI ───
backend ui_servers
    balance roundrobin
    option httpchk GET /
    server ui1 10.0.1.20:8080 check inter 10s fall 3 rise 2
    server ui2 10.0.1.21:8080 check inter 10s fall 3 rise 2
    server ui3 10.0.1.22:8080 check inter 10s fall 3 rise 2

# ─── TARGET GROUP: API ───
backend api_servers
    balance leastconn                       # Use least connections for API
    option httpchk GET /api/health
    server api1 10.0.2.30:4000 check inter 5s fall 3 rise 2
    server api2 10.0.2.31:4000 check inter 5s fall 3 rise 2
    server api3 10.0.2.32:4000 check inter 5s fall 3 rise 2
```

### AWS ALB Setup for Detached Monolithic

```
STEP 1: Create TWO Target Groups

  Target Group 1: ui-tg
    Protocol: HTTP, Port: 8080
    Health Check: GET /
    Targets:
      - i-ui001 (10.0.1.20:8080)
      - i-ui002 (10.0.1.21:8080)
      - i-ui003 (10.0.1.22:8080)

  Target Group 2: api-tg
    Protocol: HTTP, Port: 4000
    Health Check: GET /api/health
    Targets:
      - i-api001 (10.0.2.30:4000)
      - i-api002 (10.0.2.31:4000)
      - i-api003 (10.0.2.32:4000)

STEP 2: Create ALB with Listener Rules

  Listener on Port 80:
    Rule 1: IF path is /api/*   → Forward to api-tg
    Rule 2: IF path is /*       → Forward to ui-tg (default)

  Listener on Port 443 (HTTPS):
    Same rules, with SSL certificate attached

STEP 3: DNS
  CNAME: www.myapp.com → myapp-alb-xxxxx.us-east-1.elb.amazonaws.com
```

---

---

# 8. SETUP 3 — Microservices

## (Multiple services, multiple instances, different ports, different servers)

### What is this?
```
The application is broken into MANY small services, each doing ONE thing:
  - User Service       → handles user registration, login, profiles
  - Order Service      → handles orders, cart, checkout
  - Payment Service    → handles payments, refunds
  - Notification Service → handles emails, SMS, push notifications
  - Product Service    → handles product catalog, search

Each service:
  - Has its OWN codebase
  - Has its OWN database (usually)
  - Runs on its OWN port
  - Can have MULTIPLE instances (for scaling)
  - Can run on the SAME server or DIFFERENT servers
```

### FULL ARCHITECTURE DIAGRAM

```
                        [ Users from Internet ]
                                 |
                                 | DNS: www.myapp.com → 52.20.100.1
                                 ▼
                  ┌───────────────────────────────┐
                  │      EXTERNAL LOAD BALANCER    │
                  │         (AWS ALB / Nginx)       │
                  │        IP: 52.20.100.1          │
                  │       Port: 80 / 443            │
                  │                                 │
                  │  RULES:                         │
                  │    /api/*  → API Gateway         │
                  │    /*      → UI Target Group     │
                  └────────────┬──────────────────┘
                               │
                ┌──────────────┴──────────────┐
                │                             │
                ▼                             ▼
     ┌──────────────────┐          ┌──────────────────┐
     │   UI TARGET GROUP │          │  API GATEWAY      │
     │                   │          │  (Kong / Nginx /  │
     │ 10.0.1.20:3000   │          │   AWS API GW /    │
     │ 10.0.1.21:3000   │          │   Traefik)        │
     │ 10.0.1.22:3000   │          │                   │
     │                   │          │  IP: 10.0.5.10    │
     │ (React/Next.js)   │          │  Port: 8080       │
     └──────────────────┘          └────────┬──────────┘
                                            │
                                            │  PATH-BASED ROUTING
                                            │  TO INDIVIDUAL SERVICES
                                            │
            ┌───────────────┬───────────────┼───────────────┬───────────────┐
            │               │               │               │               │
            ▼               ▼               ▼               ▼               ▼
     /api/users/*     /api/orders/*   /api/payments/*  /api/products/*  /api/notify/*
            │               │               │               │               │
            ▼               ▼               ▼               ▼               ▼
    ┌──────────────┐┌──────────────┐┌──────────────┐┌──────────────┐┌──────────────┐
    │INTERNAL LB /││INTERNAL LB /││INTERNAL LB /││INTERNAL LB /││INTERNAL LB /│
    │SERVICE DISC.││SERVICE DISC.││SERVICE DISC.││SERVICE DISC.││SERVICE DISC.│
    │(User Svc TG)││(Order Svc TG)││(Pay Svc TG)││(Prod Svc TG)││(Notif Svc TG)│
    └──────┬───────┘└──────┬───────┘└──────┬───────┘└──────┬───────┘└──────┬───────┘
           │               │               │               │               │
     ┌─────┼─────┐   ┌─────┼─────┐   ┌─────┼─────┐   ┌────┼────┐    ┌────┼────┐
     │     │     │   │     │     │   │     │     │   │    │    │    │    │    │
     ▼     ▼     ▼   ▼     ▼     ▼   ▼     ▼     ▼   ▼    ▼    ▼    ▼    ▼    ▼
   ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐
   │U-1││U-2││U-3││O-1││O-2││O-3││P-1││P-2││P-3││Pr1││Pr2││Pr3││N-1││N-2││N-3│
   └─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘
     │     │     │   │     │    │    │    │    │    │    │    │    │    │    │
     └─────┼─────┘   └─────┼────┘    └────┼────┘    └────┼────┘    └────┼────┘
           ▼               ▼              ▼              ▼              ▼
       ┌──────┐        ┌──────┐       ┌──────┐       ┌──────┐       ┌──────┐
       │UserDB│        │OrdDB │       │PayDB │       │ProdDB│       │Queue │
       │Mongo │        │Postgr│       │Postgr│       │Elastic│      │Redis/│
       │:27017│        │:5432 │       │:5432 │       │:9200 │       │Kafka │
       └──────┘        └──────┘       └──────┘       └──────┘       └──────┘
```

### Instance Details — Where Each Service Runs

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        SERVICE INSTANCE MAP                                         │
├──────────────────┬───────────────────────────────────────────────────────────────────┤
│ User Service     │ Instance 1: Server-A (10.0.3.10:5001)                            │
│                  │ Instance 2: Server-A (10.0.3.10:5002)  ← Same server, diff port  │
│                  │ Instance 3: Server-B (10.0.3.11:5001)  ← Different server         │
├──────────────────┼───────────────────────────────────────────────────────────────────┤
│ Order Service    │ Instance 1: Server-C (10.0.4.10:6001)                            │
│                  │ Instance 2: Server-C (10.0.4.10:6002)  ← Same server, diff port  │
│                  │ Instance 3: Server-D (10.0.4.11:6001)  ← Different server         │
├──────────────────┼───────────────────────────────────────────────────────────────────┤
│ Payment Service  │ Instance 1: Server-E (10.0.5.10:7001)                            │
│                  │ Instance 2: Server-F (10.0.5.11:7001)  ← Different server         │
│                  │ Instance 3: Server-F (10.0.5.11:7002)  ← Same server, diff port  │
├──────────────────┼───────────────────────────────────────────────────────────────────┤
│ Product Service  │ Instance 1: Server-G (10.0.6.10:8001)                            │
│                  │ Instance 2: Server-G (10.0.6.10:8002)                            │
│                  │ Instance 3: Server-H (10.0.6.11:8001)                            │
├──────────────────┼───────────────────────────────────────────────────────────────────┤
│ Notification Svc │ Instance 1: Server-I (10.0.7.10:9001)                            │
│                  │ Instance 2: Server-I (10.0.7.10:9002)                            │
│                  │ Instance 3: Server-J (10.0.7.11:9001)                            │
└──────────────────┴───────────────────────────────────────────────────────────────────┘
```

### What is an API Gateway? (And Why Do We Need It for Microservices?)

```
WITHOUT API Gateway:
====================
The external LB would need to know about EVERY microservice and EVERY instance.
If you add a new service, you'd have to update LB rules.
If services move or scale, you'd have to update IPs/ports.

This gets VERY messy with 50+ microservices!


WITH API Gateway:
=================
External LB only talks to the API Gateway.
API Gateway knows ALL the services, their instances, and routes.

API Gateway does:
  ✅ Route requests to correct microservice
  ✅ Rate limiting (max 100 requests/min per user)
  ✅ Authentication (verify JWT token)
  ✅ Request transformation (modify headers/body)
  ✅ Response aggregation (combine multiple service responses)
  ✅ Circuit breaking (if service is down, return fallback)
  ✅ Logging & monitoring
  ✅ API versioning (/v1/users, /v2/users)


WHO DOES WHAT:
==============
  External LB  →  Handles internet traffic, SSL, distributes to API Gateway instances
  API Gateway  →  Routes /api/users to User Service, /api/orders to Order Service, etc.
  Internal LB  →  Distributes load across instances of each microservice
                   (Could be a software LB, service mesh, or service discovery)
```

### Step-by-Step Flow for Microservices

```
FULL FLOW: User places an order
=================================

1.  User clicks "Place Order" on the website (www.myapp.com)

2.  Browser sends: POST https://www.myapp.com/api/orders
    with body: { productId: "P123", quantity: 2, paymentMethod: "card" }

3.  DNS resolves www.myapp.com → 52.20.100.1 (External LB)

4.  EXTERNAL LB (52.20.100.1:443) receives the request
    ├── Terminates SSL (HTTPS → HTTP internally)
    ├── Checks rules: path /api/* → forward to API Gateway target group
    └── Picks API Gateway instance: 10.0.5.10:8080

5.  API GATEWAY (10.0.5.10:8080) receives the request
    ├── Validates JWT token from Authorization header
    ├── Rate limit check: user hasn't exceeded 100 req/min ✅
    ├── Checks route table: POST /api/orders → Order Service
    └── Forwards to Order Service (via internal LB or service discovery)

6.  INTERNAL LB for Order Service picks instance:
    └── 10.0.4.10:6001 (Order Service Instance 1)

7.  ORDER SERVICE (10.0.4.10:6001) processes the order:
    ├── a) Calls User Service (to get user details):
    │       → Internal LB picks: 10.0.3.11:5001 (User Service Instance 3)
    │       → GET http://10.0.3.11:5001/users/U456
    │       → Returns: { name: "Ritesh", email: "ritesh@email.com" }
    │
    ├── b) Calls Product Service (to check stock):
    │       → Internal LB picks: 10.0.6.10:8002 (Product Service Instance 2)
    │       → GET http://10.0.6.10:8002/products/P123
    │       → Returns: { name: "Laptop", price: 999, stock: 5 }
    │
    ├── c) Calls Payment Service (to charge the user):
    │       → Internal LB picks: 10.0.5.11:7001 (Payment Service Instance 2)
    │       → POST http://10.0.5.11:7001/payments/charge
    │       → Body: { userId: "U456", amount: 1998 }
    │       → Returns: { status: "success", txnId: "TXN789" }
    │
    ├── d) Saves order to Order DB (10.0.4.50:5432)
    │
    └── e) Calls Notification Service (to send confirmation):
            → Internal LB picks: 10.0.7.10:9001 (Notification Instance 1)
            → POST http://10.0.7.10:9001/notify/email
            → Body: { to: "ritesh@email.com", subject: "Order Confirmed" }

8.  Response flows back:
    Order Service → API Gateway → External LB → User's Browser
    Response: { orderId: "ORD999", status: "confirmed", txnId: "TXN789" }

9.  User sees: "Order placed successfully! 🎉"
```

### Inter-Service Communication Diagram

```
    ┌──────────────────────────────────────────────────────────────────────┐
    │               INTER-SERVICE COMMUNICATION FLOW                      │
    │                                                                      │
    │  Order Service (10.0.4.10:6001)                                     │
    │       │                                                              │
    │       ├──► [Internal LB] ──► User Service (10.0.3.11:5001)          │
    │       │         GET /users/U456                                      │
    │       │                                                              │
    │       ├──► [Internal LB] ──► Product Service (10.0.6.10:8002)       │
    │       │         GET /products/P123                                   │
    │       │                                                              │
    │       ├──► [Internal LB] ──► Payment Service (10.0.5.11:7001)       │
    │       │         POST /payments/charge                                │
    │       │                                                              │
    │       └──► [Internal LB] ──► Notification Service (10.0.7.10:9001)  │
    │                 POST /notify/email                                   │
    │                                                                      │
    │  Each arrow goes through an Internal LB or Service Discovery        │
    │  to pick the right instance of each service                         │
    └──────────────────────────────────────────────────────────────────────┘
```

### Nginx Config — External LB + API Gateway + Internal LBs

```nginx
# ═══════════════════════════════════════════════════════════
# FILE 1: EXTERNAL LOAD BALANCER (runs on 52.20.100.1)
# /etc/nginx/nginx.conf on the External LB server
# ═══════════════════════════════════════════════════════════

# Target Group: UI Servers
upstream ui_servers {
    server 10.0.1.20:3000;
    server 10.0.1.21:3000;
    server 10.0.1.22:3000;
}

# Target Group: API Gateway Instances
upstream api_gateway {
    server 10.0.5.10:8080;    # API Gateway Instance 1
    server 10.0.5.11:8080;    # API Gateway Instance 2
}

server {
    listen 80;
    server_name www.myapp.com;

    # Redirect HTTP → HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name www.myapp.com;

    ssl_certificate     /etc/ssl/certs/myapp.crt;
    ssl_certificate_key /etc/ssl/private/myapp.key;

    # API requests → API Gateway
    location /api/ {
        proxy_pass http://api_gateway;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # UI requests → UI Servers
    location / {
        proxy_pass http://ui_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```nginx
# ═══════════════════════════════════════════════════════════
# FILE 2: API GATEWAY CONFIG (runs on 10.0.5.10 and 10.0.5.11)
# /etc/nginx/nginx.conf on API Gateway servers
# This acts as BOTH the API Gateway AND Internal Load Balancer
# ═══════════════════════════════════════════════════════════

# ─── Internal Target Groups (one per microservice) ───

upstream user_service {
    least_conn;                          # Least connections algorithm
    server 10.0.3.10:5001;              # User Svc Instance 1 (Server-A)
    server 10.0.3.10:5002;              # User Svc Instance 2 (Server-A, diff port)
    server 10.0.3.11:5001;              # User Svc Instance 3 (Server-B)
}

upstream order_service {
    least_conn;
    server 10.0.4.10:6001;              # Order Svc Instance 1 (Server-C)
    server 10.0.4.10:6002;              # Order Svc Instance 2 (Server-C, diff port)
    server 10.0.4.11:6001;              # Order Svc Instance 3 (Server-D)
}

upstream payment_service {
    least_conn;
    server 10.0.5.10:7001;              # Payment Svc Instance 1 (Server-E)
    server 10.0.5.11:7001;              # Payment Svc Instance 2 (Server-F)
    server 10.0.5.11:7002;              # Payment Svc Instance 3 (Server-F, diff port)
}

upstream product_service {
    least_conn;
    server 10.0.6.10:8001;              # Product Svc Instance 1 (Server-G)
    server 10.0.6.10:8002;              # Product Svc Instance 2 (Server-G, diff port)
    server 10.0.6.11:8001;              # Product Svc Instance 3 (Server-H)
}

upstream notification_service {
    least_conn;
    server 10.0.7.10:9001;              # Notification Svc Instance 1 (Server-I)
    server 10.0.7.10:9002;              # Notification Svc Instance 2 (Server-I, diff port)
    server 10.0.7.11:9001;              # Notification Svc Instance 3 (Server-J)
}

server {
    listen 8080;

    # ─── Rate Limiting ───
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;

    # ─── Route: User Service ───
    location /api/users/ {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://user_service/users/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # ─── Route: Order Service ───
    location /api/orders/ {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://order_service/orders/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # ─── Route: Payment Service ───
    location /api/payments/ {
        limit_req zone=api_limit burst=10 nodelay;
        proxy_pass http://payment_service/payments/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # ─── Route: Product Service ───
    location /api/products/ {
        limit_req zone=api_limit burst=30 nodelay;
        proxy_pass http://product_service/products/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # ─── Route: Notification Service ───
    location /api/notify/ {
        limit_req zone=api_limit burst=10 nodelay;
        proxy_pass http://notification_service/notify/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # ─── Health Check Endpoint for API Gateway itself ───
    location /health {
        return 200 '{"status": "ok"}';
        add_header Content-Type application/json;
    }
}
```

### Kong API Gateway Config (Popular for Microservices)

```yaml
# ═══════════════════════════════════════════════════════════
# Kong Declarative Config (kong.yml)
# Kong is a dedicated API Gateway — more powerful than Nginx
# for microservices routing, auth, rate limiting, etc.
# ═══════════════════════════════════════════════════════════

_format_version: "3.0"

# ─── SERVICES (each microservice) ───
services:
  - name: user-service
    url: http://user-service-upstream
    routes:
      - name: user-route
        paths:
          - /api/users
        strip_path: false

  - name: order-service
    url: http://order-service-upstream
    routes:
      - name: order-route
        paths:
          - /api/orders
        strip_path: false

  - name: payment-service
    url: http://payment-service-upstream
    routes:
      - name: payment-route
        paths:
          - /api/payments
        strip_path: false

  - name: product-service
    url: http://product-service-upstream
    routes:
      - name: product-route
        paths:
          - /api/products
        strip_path: false

# ─── UPSTREAMS (Target Groups with Load Balancing) ───
upstreams:
  - name: user-service-upstream
    algorithm: least-connections
    healthchecks:
      active:
        http_path: /health
        healthy:
          interval: 5
          successes: 2
        unhealthy:
          interval: 5
          http_failures: 3
    targets:
      - target: 10.0.3.10:5001
        weight: 100
      - target: 10.0.3.10:5002
        weight: 100
      - target: 10.0.3.11:5001
        weight: 100

  - name: order-service-upstream
    algorithm: least-connections
    targets:
      - target: 10.0.4.10:6001
        weight: 100
      - target: 10.0.4.10:6002
        weight: 100
      - target: 10.0.4.11:6001
        weight: 100

  - name: payment-service-upstream
    algorithm: least-connections
    targets:
      - target: 10.0.5.10:7001
        weight: 100
      - target: 10.0.5.11:7001
        weight: 100
      - target: 10.0.5.11:7002
        weight: 100

  - name: product-service-upstream
    algorithm: round-robin
    targets:
      - target: 10.0.6.10:8001
        weight: 100
      - target: 10.0.6.10:8002
        weight: 100
      - target: 10.0.6.11:8001
        weight: 100

# ─── PLUGINS (API Gateway Features) ───
plugins:
  - name: rate-limiting
    config:
      minute: 100
      policy: local

  - name: jwt
    config:
      uri_param_names:
        - jwt

  - name: cors
    config:
      origins:
        - "https://www.myapp.com"
      methods:
        - GET
        - POST
        - PUT
        - DELETE
```

### AWS Full Setup for Microservices

```
═══════════════════════════════════════════════════════════════════
AWS ARCHITECTURE FOR MICROSERVICES (Full Production Setup)
═══════════════════════════════════════════════════════════════════

LAYER 1: DNS
─────────────
  Route 53:
    www.myapp.com → ALB DNS (myapp-alb-xxx.us-east-1.elb.amazonaws.com)

LAYER 2: External ALB (Internet-Facing)
────────────────────────────────────────
  Name: myapp-external-alb
  Scheme: internet-facing
  Security Group: Allow 80, 443 from 0.0.0.0/0

  Listeners:
    Port 443 (HTTPS):
      SSL Cert: *.myapp.com (from ACM)
      Rules:
        Rule 1: path /api/*  → Forward to api-gateway-tg
        Rule 2: path /*      → Forward to ui-tg (default)

  Target Groups:
    ui-tg:
      Targets: 10.0.1.20:3000, 10.0.1.21:3000, 10.0.1.22:3000
      Health: GET / every 30s

    api-gateway-tg:
      Targets: 10.0.5.10:8080, 10.0.5.11:8080
      Health: GET /health every 10s

LAYER 3: API Gateway (AWS API Gateway or Kong on ECS/EKS)
──────────────────────────────────────────────────────────
  Routes requests to internal services via internal ALBs

LAYER 4: Internal ALBs (One per service or shared)
───────────────────────────────────────────────────
  Internal ALB: user-service-alb (internal, not internet-facing)
    Listener Port 80 → user-service-tg
    Target Group: user-service-tg
      - 10.0.3.10:5001
      - 10.0.3.10:5002
      - 10.0.3.11:5001

  Internal ALB: order-service-alb
    Listener Port 80 → order-service-tg
    Target Group: order-service-tg
      - 10.0.4.10:6001
      - 10.0.4.10:6002
      - 10.0.4.11:6001

  Internal ALB: payment-service-alb
    Listener Port 80 → payment-service-tg
    Target Group: payment-service-tg
      - 10.0.5.10:7001
      - 10.0.5.11:7001
      - 10.0.5.11:7002

  Internal ALB: product-service-alb
    Listener Port 80 → product-service-tg
    Target Group: product-service-tg
      - 10.0.6.10:8001
      - 10.0.6.10:8002
      - 10.0.6.11:8001

  Internal ALB: notification-service-alb
    Listener Port 80 → notification-service-tg
    Target Group: notification-service-tg
      - 10.0.7.10:9001
      - 10.0.7.10:9002
      - 10.0.7.11:9001

LAYER 5: Services (ECS Tasks / EKS Pods / EC2 Instances)
─────────────────────────────────────────────────────────
  Each service runs as containers or on EC2 instances.
  Auto Scaling Groups scale instances based on CPU/memory.

LAYER 6: Databases (each service has its own)
──────────────────────────────────────────────
  User Service    → MongoDB (10.0.8.10:27017)
  Order Service   → PostgreSQL RDS (10.0.8.20:5432)
  Payment Service → PostgreSQL RDS (10.0.8.30:5432)
  Product Service → Elasticsearch (10.0.8.40:9200)
  Notification    → Redis Queue (10.0.8.50:6379)
```

### Traefik Config (Modern LB for Docker/Kubernetes Microservices)

```yaml
# ═══════════════════════════════════════════════════════════
# Traefik — Auto-discovery LB for Docker containers
# docker-compose.yml with Traefik
# ═══════════════════════════════════════════════════════════

version: "3.8"

services:
  # ─── TRAEFIK (Load Balancer + API Gateway) ───
  traefik:
    image: traefik:v2.10
    command:
      - "--api.insecure=true"                    # Dashboard on :8080
      - "--providers.docker=true"                # Auto-discover Docker containers
      - "--entrypoints.web.address=:80"          # Listen on port 80
      - "--entrypoints.websecure.address=:443"   # Listen on port 443
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"    # Traefik Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # Docker socket for auto-discovery

  # ─── USER SERVICE (3 instances) ───
  user-service:
    image: myapp/user-service:latest
    deploy:
      replicas: 3                                 # 3 instances automatically!
    labels:
      - "traefik.http.routers.users.rule=PathPrefix(`/api/users`)"
      - "traefik.http.services.users.loadbalancer.server.port=5001"
      - "traefik.http.services.users.loadbalancer.healthcheck.path=/health"
      - "traefik.http.services.users.loadbalancer.healthcheck.interval=10s"

  # ─── ORDER SERVICE (3 instances) ───
  order-service:
    image: myapp/order-service:latest
    deploy:
      replicas: 3
    labels:
      - "traefik.http.routers.orders.rule=PathPrefix(`/api/orders`)"
      - "traefik.http.services.orders.loadbalancer.server.port=6001"
      - "traefik.http.services.orders.loadbalancer.healthcheck.path=/health"

  # ─── PAYMENT SERVICE (2 instances) ───
  payment-service:
    image: myapp/payment-service:latest
    deploy:
      replicas: 2
    labels:
      - "traefik.http.routers.payments.rule=PathPrefix(`/api/payments`)"
      - "traefik.http.services.payments.loadbalancer.server.port=7001"

  # ─── PRODUCT SERVICE (3 instances) ───
  product-service:
    image: myapp/product-service:latest
    deploy:
      replicas: 3
    labels:
      - "traefik.http.routers.products.rule=PathPrefix(`/api/products`)"
      - "traefik.http.services.products.loadbalancer.server.port=8001"

  # ─── NOTIFICATION SERVICE (2 instances) ───
  notification-service:
    image: myapp/notification-service:latest
    deploy:
      replicas: 2
    labels:
      - "traefik.http.routers.notify.rule=PathPrefix(`/api/notify`)"
      - "traefik.http.services.notify.loadbalancer.server.port=9001"

# Traefik AUTOMATICALLY discovers all containers and load balances!
# No manual IP/port config needed! It reads Docker labels.
```

---

---

## 9. Health Checks

```
WHAT: LB periodically checks if each server is alive.
WHY:  So it doesn't send traffic to dead servers.

HOW IT WORKS:
─────────────
Every X seconds, LB sends a request to each server:

  LB ──── GET /health ────► Server 1  → Returns 200 OK     ✅ Healthy
  LB ──── GET /health ────► Server 2  → Returns 200 OK     ✅ Healthy
  LB ──── GET /health ────► Server 3  → Connection refused  ❌ Unhealthy (after 3 fails)

After 3 consecutive failures → Server is marked UNHEALTHY
LB stops sending traffic to that server.

When Server 3 comes back → after 2 consecutive successes → marked HEALTHY again
LB starts sending traffic to it.


EXAMPLE HEALTH CHECK ENDPOINT (Node.js):
─────────────────────────────────────────
```

```javascript
// Simple health check endpoint in Express.js
app.get('/health', (req, res) => {
  // Check if app can connect to DB
  const dbStatus = mongoose.connection.readyState === 1;

  if (dbStatus) {
    res.status(200).json({
      status: 'healthy',
      uptime: process.uptime(),
      timestamp: Date.now()
    });
  } else {
    res.status(503).json({
      status: 'unhealthy',
      reason: 'Database connection failed'
    });
  }
});
```

---

## 10. SSL Termination

```
WHAT: The LB handles HTTPS (SSL/TLS) decryption.
WHY:  So your backend servers don't need to deal with SSL.
      Saves CPU on app servers. Easier certificate management.

FLOW:
─────
  Browser ──── HTTPS (encrypted) ────► LB
  LB ──── HTTP (plain, fast) ────► App Servers (internal network, safe)


                    ENCRYPTED              PLAIN (internal network)
  [Browser] ═══════════════► [LB] ─────────────────────► [App Server]
              HTTPS/443           HTTP/3000
              SSL cert here       No cert needed


Nginx SSL Termination:
──────────────────────
```

```nginx
server {
    listen 443 ssl;
    server_name www.myapp.com;

    # SSL Certificate (from Let's Encrypt or purchased)
    ssl_certificate     /etc/ssl/certs/myapp.crt;
    ssl_certificate_key /etc/ssl/private/myapp.key;

    # Modern SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        # Forward as plain HTTP to backend (internal network)
        proxy_pass http://monolith_app;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name www.myapp.com;
    return 301 https://$host$request_uri;
}
```

---

## 11. Sticky Sessions

```
WHAT: Ensure the SAME user always goes to the SAME server.
WHY:  When sessions are stored in server memory (not in Redis/DB),
      user must hit the same server or they'll lose their session.

HOW:
────
  1st request from User A → LB assigns Server 2
  LB sets cookie: SERVERID=server2
  All future requests from User A → Forced to Server 2 (because of cookie)


  User A ──► LB ──► Server 2 (always!)
  User B ──► LB ──► Server 1 (always!)
  User C ──► LB ──► Server 3 (always!)


BETTER ALTERNATIVE:
───────────────────
  Don't use sticky sessions!
  Store sessions in Redis/database instead.
  Then ANY server can handle ANY user.
  This is the preferred approach in production.
```

```nginx
# Nginx sticky session (using ip_hash)
upstream monolith_app {
    ip_hash;                     # Same client IP → Same server
    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
    server 10.0.1.12:3000;
}
```

```haproxy
# HAProxy sticky session (using cookie)
backend monolith_servers
    balance roundrobin
    cookie SERVERID insert indirect nocache
    server app1 10.0.1.10:3000 cookie s1 check
    server app2 10.0.1.11:3000 cookie s2 check
    server app3 10.0.1.12:3000 cookie s3 check
```

---

## 12. Summary Comparison Table

```
┌──────────────────────────┬────────────────────────┬────────────────────────┬────────────────────────────┐
│ ASPECT                   │ MONOLITHIC COUPLED      │ DETACHED MONOLITHIC    │ MICROSERVICES              │
├──────────────────────────┼────────────────────────┼────────────────────────┼────────────────────────────┤
│ LB Count                 │ 1 (External only)      │ 1 (with path rules)   │ 2+ (External + Internal)   │
│                          │                        │ or 2 (separate LBs)   │ + API Gateway              │
├──────────────────────────┼────────────────────────┼────────────────────────┼────────────────────────────┤
│ Target Groups            │ 1                      │ 2 (UI + API)          │ Many (1 per service)       │
├──────────────────────────┼────────────────────────┼────────────────────────┼────────────────────────────┤
│ API Gateway Needed?      │ No                     │ No                    │ YES (essential)            │
├──────────────────────────┼────────────────────────┼────────────────────────┼────────────────────────────┤
│ Routing Type             │ Simple (all → backend) │ Path-based (/api, /)  │ Path-based + Service       │
│                          │                        │                        │ Discovery                  │
├──────────────────────────┼────────────────────────┼────────────────────────┼────────────────────────────┤
│ Scaling                  │ Scale entire app       │ Scale UI & API         │ Scale each service         │
│                          │                        │ independently          │ independently              │
├──────────────────────────┼────────────────────────┼────────────────────────┼────────────────────────────┤
│ Complexity               │ Low                    │ Medium                 │ High                       │
├──────────────────────────┼────────────────────────┼────────────────────────┼────────────────────────────┤
│ Best Tools               │ Nginx / HAProxy        │ Nginx / ALB            │ Kong + Traefik / ALB +     │
│                          │                        │                        │ API Gateway + Service Mesh │
├──────────────────────────┼────────────────────────┼────────────────────────┼────────────────────────────┤
│ SSL Termination          │ At LB                  │ At LB                 │ At External LB             │
├──────────────────────────┼────────────────────────┼────────────────────────┼────────────────────────────┤
│ Health Checks            │ 1 endpoint             │ 2 endpoints            │ N endpoints (1 per service)│
├──────────────────────────┼────────────────────────┼────────────────────────┼────────────────────────────┤
│ Inter-Service Calls      │ None (all in one app)  │ UI calls API via LB   │ Services call each other   │
│                          │                        │                        │ via Internal LBs           │
├──────────────────────────┼────────────────────────┼────────────────────────┼────────────────────────────┤
│ Example Companies        │ Small startups         │ Medium companies       │ Netflix, Amazon, Uber      │
│                          │ (< 10K users)          │ (10K-1M users)        │ (1M+ users)                │
└──────────────────────────┴────────────────────────┴────────────────────────┴────────────────────────────┘
```

---

## BONUS: Quick Reference — What Goes Where

```
DECISION TREE:
═══════════════

Q: Are UI and API in the SAME codebase?
├── YES → MONOLITHIC COUPLED → Setup 1
└── NO  → Q: Is it ONE big API or MANY small services?
          ├── ONE big API → DETACHED MONOLITHIC → Setup 2
          └── MANY small services → MICROSERVICES → Setup 3

WHAT TO USE:
═══════════════

Small project, just starting?
  → Nginx as LB + 2-3 app instances

Medium project, growing fast?
  → AWS ALB + Auto Scaling Group + 2 Target Groups

Large project, 50+ developers, millions of users?
  → AWS ALB (external) + Kong/Traefik (API Gateway) + Internal ALBs
  → OR Kubernetes with Ingress Controller + Service Mesh (Istio/Envoy)
```

---

> **Remember**: Load balancer is NOT just about splitting traffic.
> It's about **reliability** (failover), **scalability** (add more servers),
> and **performance** (distribute load evenly).
