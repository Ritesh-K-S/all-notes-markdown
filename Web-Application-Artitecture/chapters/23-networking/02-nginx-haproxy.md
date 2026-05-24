# Nginx & HAProxy Deep Dive

> **What you'll learn**: How Nginx and HAProxy work internally, their architecture differences, configuration patterns, and when to choose one over the other for reverse proxying, load balancing, and high-performance traffic management.

---

## Real-Life Analogy

Think of a **massive hospital reception desk**:

**Nginx** is like a receptionist who's also a nurse — she can:
- Direct patients to the right doctor (reverse proxy / load balancing)
- Hand out pamphlets from a shelf (serve static files)
- Fill out paperwork on behalf of the patient (SSL termination)
- Handle 10,000 people in the waiting room without breaking a sweat (event-driven)

**HAProxy** is like a highly specialized traffic controller at a highway junction:
- His ONLY job is directing cars to the right lane (pure proxying/load balancing)
- He doesn't serve coffee, he doesn't sell maps — he just routes traffic
- He's extremely fast at this one job
- He can track every car's speed, lane, and ETA (detailed metrics)

```
Nginx = Swiss Army Knife (web server + reverse proxy + LB + static files + SSL)

HAProxy = Laser-focused traffic director (proxy + LB + health checks + metrics)
```

---

## Core Concept Explained Step-by-Step

### Step 1: What is Nginx?

**Nginx** (pronounced "engine-X") is an open-source web server that also functions as:
- Reverse proxy
- Load balancer
- HTTP cache
- Static file server
- Mail proxy
- SSL/TLS terminator

Created by Igor Sysoev in 2004 to solve the **C10K problem** (handling 10,000+ concurrent connections).

### Step 2: What is HAProxy?

**HAProxy** (High Availability Proxy) is a free, open-source TCP/HTTP load balancer and proxy server.

Created by Willy Tarreau in 2000, it's focused on:
- Being the fastest and most reliable proxy
- Layer 4 (TCP) and Layer 7 (HTTP) load balancing
- Advanced health checking
- Extremely detailed logging and metrics
- Zero-downtime reloads

### Step 3: Architecture Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                      NGINX ARCHITECTURE                         │
│                                                                 │
│  ┌──────────────┐                                              │
│  │ Master Process│ (reads config, manages workers)             │
│  └──────┬───────┘                                              │
│         │ fork()                                                │
│    ┌────┼────────────┐                                         │
│    ▼    ▼            ▼                                         │
│  ┌────┐ ┌────┐    ┌────┐                                      │
│  │ W1 │ │ W2 │    │ W4 │  ← Worker Processes (1 per CPU core) │
│  └────┘ └────┘    └────┘                                      │
│    │                                                            │
│    ▼                                                            │
│  Event Loop (epoll/kqueue)                                     │
│  Each worker handles THOUSANDS of connections simultaneously   │
│  using non-blocking I/O                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     HAPROXY ARCHITECTURE                        │
│                                                                 │
│  ┌──────────────────┐                                          │
│  │  Single Process   │ (or multi-threaded since v1.8)          │
│  └────────┬─────────┘                                          │
│           │                                                     │
│    ┌──────┼──────────────┐                                     │
│    ▼      ▼              ▼                                     │
│  ┌──────┐ ┌──────┐    ┌──────┐                                │
│  │ T1   │ │ T2   │    │ T4   │  ← Threads (since v1.8+)      │
│  └──────┘ └──────┘    └──────┘                                │
│    │                                                            │
│    ▼                                                            │
│  Event-driven (epoll/kqueue)                                   │
│  Focuses ONLY on proxying — no file serving, no app logic      │
│  Lock-free data structures for maximum throughput              │
└─────────────────────────────────────────────────────────────────┘
```

### Step 4: How Nginx Handles a Request

```
Request lifecycle through Nginx:

1. Client connects (TCP handshake)
         │
         ▼
2. Nginx worker picks up connection (via epoll event)
         │
         ▼
3. SSL/TLS handshake (if HTTPS)
         │
         ▼
4. Read HTTP request headers
         │
         ▼
5. Match server block (virtual host)
         │
         ▼
6. Match location block (URL path)
         │
         ▼
7. Execute directives:
   ├── Static file? ──▶ Read from disk, send response
   ├── Proxy pass? ──▶ Forward to upstream server
   ├── FastCGI? ──▶ Forward to PHP-FPM / app
   └── Redirect? ──▶ Return 301/302
         │
         ▼
8. Process response (compress, add headers, cache)
         │
         ▼
9. Send response to client
         │
         ▼
10. Log the request (access.log)
```

### Step 5: How HAProxy Handles a Request

```
Request lifecycle through HAProxy:

1. Client connects (TCP handshake)
         │
         ▼
2. HAProxy maps connection to a "frontend"
         │
         ▼
3. Frontend applies ACLs (access control rules)
   ├── Check source IP
   ├── Check request rate
   └── Check headers/URL patterns
         │
         ▼
4. Select a "backend" based on ACL rules
         │
         ▼
5. Choose a server from the backend pool
   (using configured algorithm: roundrobin, leastconn, etc.)
         │
         ▼
6. Forward request to chosen server
         │
         ▼
7. Wait for response (with timeout)
         │
         ▼
8. Forward response to client
   (modify headers if configured)
         │
         ▼
9. Log detailed metrics (response time, server chosen, retries)
```

---

## How It Works Internally

### Nginx Internals: The Event-Driven Model

Nginx uses an **event-driven, asynchronous, non-blocking** architecture:

```
Traditional Server (Apache with prefork):
┌──────────────────────────────────┐
│  Connection 1 → Thread 1 (busy) │
│  Connection 2 → Thread 2 (busy) │
│  Connection 3 → Thread 3 (busy) │
│  ...                             │
│  Connection 10000 → OUT OF THREADS! │
└──────────────────────────────────┘

Nginx (Event Loop):
┌──────────────────────────────────┐
│  Single Worker Process:          │
│                                  │
│  epoll_wait() → events ready:    │
│  ├── Conn 1: data arrived → read │
│  ├── Conn 500: socket writable → write │
│  ├── Conn 3000: new conn → accept │
│  └── Conn 8000: upstream ready → forward │
│                                  │
│  One worker handles ALL of them! │
│  No thread switching overhead    │
└──────────────────────────────────┘
```

**Key internals:**
- Uses **epoll** (Linux) or **kqueue** (BSD/macOS) for I/O multiplexing
- Each worker process is single-threaded
- Worker count = number of CPU cores (optimal)
- Shared memory zones for caching and rate limiting
- File descriptors are shared across connections in same worker

### HAProxy Internals: The Proxy-Optimized Model

HAProxy is optimized for one thing: **moving bytes between connections as fast as possible**.

```
HAProxy Connection Handling:

Frontend (listener)                    Backend (servers)
┌─────────────────┐                   ┌─────────────────┐
│ Bind *:80       │                   │ Server pool:    │
│ Bind *:443      │                   │ ├── srv1:8080   │
│                 │    ┌───────┐      │ ├── srv2:8080   │
│ Accept conn ────┼───▶│ Stick │      │ └── srv3:8080   │
│                 │    │ Table │      │                 │
│ Apply ACLs      │    └───┬───┘      │ Health checks   │
│ Rate limit      │        │          │ (active + agent)│
└─────────────────┘        │          └─────────────────┘
                           │
                    ┌──────▼──────┐
                    │   Session   │
                    │  (stream)   │
                    │             │
                    │ Client Buf ←┼── reads from client
                    │ Server Buf ─┼──▶ writes to server
                    │             │
                    │ Splice/     │
                    │ Zero-copy   │ ← kernel bypasses user space
                    └─────────────┘
```

**Key internals:**
- **Splice system call** — moves data between sockets without copying to user space (zero-copy)
- **Stick tables** — in-memory key-value store for session persistence, rate limiting, counters
- **Multi-threaded** since v1.8 (one thread per CPU core)
- **Lock-free queues** for inter-thread communication
- **Connection pooling** to backends (reuse connections)
- **Seamless reload** — new process takes over, old one drains connections

---

## Code Examples

### Python — Health Check Server (for backends)

```python
# health_check.py — A backend server with proper health endpoints
# Both Nginx and HAProxy can check this endpoint

from flask import Flask, jsonify
import psycopg2
import redis

app = Flask(__name__)

@app.route('/health')
def health_check():
    """Basic health check — is the app running?"""
    return jsonify({"status": "healthy"}), 200

@app.route('/health/ready')
def readiness_check():
    """Readiness check — can the app serve traffic?
    Checks all dependencies (DB, cache, etc.)"""
    checks = {}
    
    # Check database connection
    try:
        conn = psycopg2.connect("postgresql://localhost/mydb")
        conn.close()
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"failed: {str(e)}"
        return jsonify({"status": "not_ready", "checks": checks}), 503
    
    # Check Redis connection
    try:
        r = redis.Redis(host='localhost', port=6379)
        r.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"failed: {str(e)}"
        return jsonify({"status": "not_ready", "checks": checks}), 503
    
    return jsonify({"status": "ready", "checks": checks}), 200

@app.route('/health/live')
def liveness_check():
    """Liveness check — is the process alive?
    Simple response, no dependency checks."""
    return "OK", 200

if __name__ == '__main__':
    app.run(port=8080)
```

### Java — Custom Health Indicator (Spring Boot)

```java
// CustomHealthIndicator.java — For HAProxy/Nginx health checks
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import javax.sql.DataSource;
import java.sql.Connection;

@Component
public class CustomHealthIndicator implements HealthIndicator {

    private final DataSource dataSource;

    public CustomHealthIndicator(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Health health() {
        // Check database connectivity
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(2)) {  // 2-second timeout
                return Health.up()
                    .withDetail("database", "connected")
                    .withDetail("pool_active", getActiveConnections())
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("database", "disconnected")
                .withException(e)
                .build();
        }
        return Health.down().build();
    }

    private int getActiveConnections() {
        // Return active connection count from pool
        // (implementation depends on pool: HikariCP, etc.)
        return 0;
    }
}
```

---

## Infrastructure Examples

### Complete Nginx Configuration (Production-Ready)

```nginx
# /etc/nginx/nginx.conf — Production Nginx configuration

# Worker processes = CPU cores
worker_processes auto;

# Max file descriptors per worker
worker_rlimit_nofile 65535;

events {
    # Max simultaneous connections per worker
    worker_connections 16384;
    
    # Use epoll on Linux for efficiency
    use epoll;
    
    # Accept multiple connections at once
    multi_accept on;
}

http {
    # Basic settings
    sendfile on;                  # Kernel-level file sending
    tcp_nopush on;               # Send headers and file in one packet
    tcp_nodelay on;              # Don't buffer small packets
    keepalive_timeout 65;        # Keep connections open for reuse
    types_hash_max_size 2048;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1000;
    gzip_types text/plain text/css application/json application/javascript;
    
    # Rate limiting zone (10MB shared memory, 10 req/sec per IP)
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    
    # Upstream backend pool
    upstream api_servers {
        least_conn;  # Send to server with fewest active connections
        
        server 10.0.1.10:8080 weight=5 max_fails=3 fail_timeout=30s;
        server 10.0.1.11:8080 weight=3 max_fails=3 fail_timeout=30s;
        server 10.0.1.12:8080 weight=2 max_fails=3 fail_timeout=30s;
        server 10.0.1.13:8080 backup;  # Only used when others fail
        
        # Keep connections to backends alive (connection pooling)
        keepalive 32;
    }
    
    # Caching configuration
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=app_cache:10m
                     max_size=1g inactive=60m use_temp_path=off;
    
    # HTTPS server
    server {
        listen 443 ssl http2;
        server_name api.myapp.com;
        
        # SSL configuration
        ssl_certificate /etc/ssl/certs/myapp.crt;
        ssl_certificate_key /etc/ssl/private/myapp.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        
        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Strict-Transport-Security "max-age=31536000" always;
        
        # API endpoints — reverse proxy to backend
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            
            proxy_pass http://api_servers;
            proxy_http_version 1.1;
            proxy_set_header Connection "";  # Enable keepalive to upstream
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 30s;
            
            # Retry on failure
            proxy_next_upstream error timeout http_502 http_503;
            proxy_next_upstream_tries 2;
        }
        
        # WebSocket endpoint
        location /ws/ {
            proxy_pass http://api_servers;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_read_timeout 3600s;  # Keep WS alive for 1 hour
        }
        
        # Static files — serve directly from disk
        location /static/ {
            root /var/www/myapp;
            expires 30d;
            add_header Cache-Control "public, immutable";
        }
    }
    
    # Redirect HTTP to HTTPS
    server {
        listen 80;
        server_name api.myapp.com;
        return 301 https://$server_name$request_uri;
    }
}
```

### Complete HAProxy Configuration (Production-Ready)

```bash
# /etc/haproxy/haproxy.cfg — Production HAProxy configuration

global
    # Process settings
    maxconn 100000               # Max total connections
    nbthread 4                   # Threads (match CPU cores)
    
    # Logging
    log /dev/log local0 info
    log /dev/log local1 notice
    
    # Security
    chroot /var/lib/haproxy
    user haproxy
    group haproxy
    
    # SSL settings
    ssl-default-bind-ciphers HIGH:!aNULL:!MD5
    ssl-default-bind-options ssl-min-ver TLSv1.2
    tune.ssl.default-dh-param 2048

defaults
    mode http                    # Layer 7 (HTTP mode)
    log global
    
    # Timeouts
    timeout connect 5s           # Time to connect to backend
    timeout client 30s           # Time waiting for client data
    timeout server 30s           # Time waiting for server response
    timeout http-request 10s     # Time for complete HTTP request
    timeout http-keep-alive 10s  # Keep-alive timeout
    timeout queue 30s            # Time in backend queue
    
    # Retries
    retries 3
    option redispatch            # Retry on different server if one fails
    
    # Logging
    option httplog               # Detailed HTTP logging
    option dontlognull           # Don't log health checks

#───────────────────────────────────────────────
# FRONTEND: Where clients connect
#───────────────────────────────────────────────
frontend http_front
    bind *:80
    # Redirect all HTTP to HTTPS
    http-request redirect scheme https unless { ssl_fc }

frontend https_front
    bind *:443 ssl crt /etc/ssl/certs/myapp.pem
    
    # Request inspection ACLs
    acl is_api path_beg /api/
    acl is_websocket hdr(Upgrade) -i websocket
    acl is_static path_beg /static/
    acl is_health path /health
    
    # Rate limiting (using stick-table)
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 100 }
    
    # Route to appropriate backend
    use_backend ws_servers if is_websocket
    use_backend static_servers if is_static
    use_backend api_servers if is_api
    default_backend api_servers

#───────────────────────────────────────────────
# BACKENDS: Where requests are forwarded
#───────────────────────────────────────────────
backend api_servers
    balance leastconn            # Send to least busy server
    option httpchk GET /health   # Active health check
    http-check expect status 200
    
    # Servers with health checking
    server api1 10.0.1.10:8080 check inter 5s fall 3 rise 2 weight 100
    server api2 10.0.1.11:8080 check inter 5s fall 3 rise 2 weight 100
    server api3 10.0.1.12:8080 check inter 5s fall 3 rise 2 weight 50
    server api4 10.0.1.13:8080 check inter 5s fall 3 rise 2 backup
    
    # Connection pooling to backends
    http-reuse safe
    
    # Add headers
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request add-header X-Real-IP %[src]

backend ws_servers
    balance source              # Sticky (same client → same server)
    option httpchk GET /health/live
    
    server ws1 10.0.2.10:8080 check inter 10s
    server ws2 10.0.2.11:8080 check inter 10s
    
    timeout tunnel 1h           # Keep WebSocket connections alive

backend static_servers
    balance roundrobin
    
    server static1 10.0.3.10:80 check
    server static2 10.0.3.11:80 check
    
    # Cache static content
    http-response set-header Cache-Control "public, max-age=86400"

#───────────────────────────────────────────────
# STATS: Built-in monitoring dashboard
#───────────────────────────────────────────────
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if TRUE          # Allow drain/disable from UI
```

### Performance Comparison

```
┌─────────────────────────────────────────────────────────────┐
│          NGINX vs HAPROXY — Performance Characteristics      │
├─────────────────────┬──────────────────┬────────────────────┤
│ Feature             │ Nginx            │ HAProxy            │
├─────────────────────┼──────────────────┼────────────────────┤
│ Max connections     │ 100K+ per worker │ 100K+ total        │
│ Latency added       │ ~0.5-2ms         │ ~0.1-0.5ms         │
│ Throughput (HTTP)   │ ~500K req/s      │ ~800K req/s        │
│ SSL performance     │ Excellent        │ Excellent          │
│ Static file serving │ YES (fast!)      │ NO                 │
│ WebSocket support   │ YES              │ YES                │
│ gRPC support        │ YES (v1.13+)     │ YES (v2.0+)        │
│ Built-in cache      │ YES              │ NO                 │
│ Hot reload          │ YES (graceful)   │ YES (seamless)     │
│ Stats dashboard     │ Commercial only  │ FREE built-in      │
│ Config complexity   │ Medium           │ Low-Medium         │
│ Memory usage        │ Low-Medium       │ Very Low           │
└─────────────────────┴──────────────────┴────────────────────┘
```

---

## Real-World Example

### Netflix — Nginx + Custom Extensions (OpenResty)

Netflix uses a custom build of Nginx (via OpenResty/Lua) called **Zuul** for some workloads and Nginx-based proxies for others:

```
Netflix Traffic Flow:

User → AWS CloudFront (CDN)
         │
         ▼
   ┌─────────────────┐
   │ Nginx (OpenResty)│ ← Edge proxy with Lua scripting
   │                  │
   │ • A/B testing    │ (route 5% to new version)
   │ • Auth check     │ (validate Netflix token)
   │ • Rate limiting  │ (per-user request limits)
   │ • Routing logic  │ (device type → appropriate backend)
   └────────┬─────────┘
            │
      ┌─────┼──────────┐
      ▼     ▼          ▼
   API-1  API-2     API-N  (microservices)
```

### GitHub — HAProxy at Scale

GitHub uses HAProxy as their primary load balancer:

```
GitHub's architecture:

Internet → Anycast DNS
             │
             ▼
   ┌──────────────────┐
   │ HAProxy (Layer 4) │ ← First layer: TCP load balancing
   │  (multiple nodes) │    Fast, no HTTP parsing overhead
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │ HAProxy (Layer 7) │ ← Second layer: HTTP routing
   │                   │    Route by path: /api, /git, /web
   └────────┬─────────┘
            │
      ┌─────┼──────────┐
      ▼     ▼          ▼
   Web App  Git Server  API

Key numbers:
• 100+ Gbps of traffic
• Millions of Git operations/day
• Sub-millisecond routing decisions
```

### Cloudflare — Nginx → Custom (Pingora)

Cloudflare used Nginx for years but built their own proxy (Pingora) in Rust:

```
Evolution:
  2010-2022: Cloudflare used Nginx (worker-per-core model)
  2022+: Moved to Pingora (custom proxy in Rust)

Why they switched:
• Nginx workers don't share connection pools → memory waste
• Needed custom protocols (QUIC/HTTP3) before Nginx supported them
• Worker-per-core means N copies of every connection table
• Wanted multi-threaded shared-memory architecture

Lesson: At EXTREME scale, even Nginx has limitations.
Most companies (99.9%) will never hit these limits.
```

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not tuning `worker_connections` in Nginx | Server runs out of file descriptors | Set to `worker_rlimit_nofile` × `worker_processes` |
| Using Nginx `proxy_pass` without trailing slash | URL path gets mangled | Be consistent: `proxy_pass http://backend/;` |
| HAProxy `timeout server` too low | Backend responses get cut off | Set based on your slowest API endpoint |
| Not enabling `keepalive` to upstreams | New TCP connection per request (slow) | Add `keepalive 32` in Nginx upstream block |
| Running HAProxy in HTTP mode for TCP traffic | Protocol mismatch, data corruption | Use `mode tcp` for non-HTTP protocols |
| Not monitoring HAProxy stats page | Blind to backend health | Always enable `listen stats` in production |
| Nginx `proxy_buffering off` globally | High memory usage under load | Only disable for streaming/SSE endpoints |
| No connection limits per backend | One backend can consume all connections | Set `maxconn` per server in HAProxy |

---

## When to Use / When NOT to Use

### Choose Nginx When:
- ✅ You need a **web server + reverse proxy** combined
- ✅ You're serving **static files** alongside dynamic content
- ✅ You need **built-in caching** for API responses
- ✅ You want **one tool** for everything (simplicity)
- ✅ You need **Lua scripting** for custom logic (OpenResty)
- ✅ Your team already knows Nginx configuration

### Choose HAProxy When:
- ✅ You need the **absolute fastest** proxy performance
- ✅ You need **advanced health checking** (agent checks, intervals)
- ✅ You want a **free built-in stats dashboard**
- ✅ You need **Layer 4 (TCP) load balancing** (databases, Redis, etc.)
- ✅ You need **stick tables** for rate limiting/session tracking
- ✅ You need **seamless reloads** with zero dropped connections
- ✅ You're building a **dedicated load balancer tier**

### Choose Both Together:
```
Common pattern in production:

Internet ──▶ [HAProxy] ──▶ [Nginx] ──▶ Application
              (L4/L7 LB)    (SSL, cache,    (your code)
                             static files)
```

- ✅ HAProxy for pure load balancing between multiple Nginx instances
- ✅ Nginx for SSL termination, caching, and static file serving
- ✅ This combination gives you the best of both worlds

### Don't Use Either When:
- ❌ You're running on a managed platform (AWS ALB handles it for you)
- ❌ You have a service mesh (Istio/Envoy already does this)
- ❌ Traffic is so low that any solution works fine

---

## Key Takeaways

1. **Nginx** is a Swiss Army Knife — web server, reverse proxy, cache, and load balancer in one. Perfect for most teams.
2. **HAProxy** is a laser-focused proxy — faster raw performance, better health checks, and a free stats dashboard. Ideal as a dedicated load balancer.
3. Both use **event-driven I/O** (epoll/kqueue) and handle 100K+ concurrent connections easily.
4. Nginx shines when you need to **serve static files** and **cache responses** at the proxy layer.
5. HAProxy shines when you need **Layer 4 (TCP) proxying**, **stick tables**, and **detailed connection metrics**.
6. In large production systems, you'll often see **both used together** — HAProxy for load balancing, Nginx for SSL/caching/static files.
7. For 95% of startups and medium-scale applications, **Nginx alone** is more than sufficient.

---

## What's Next?

Next, we'll explore the fundamental transport protocols of the internet: **TCP vs UDP** — understanding when reliability matters vs when speed matters, and how this choice affects your architecture.

→ [03-tcp-vs-udp.md](./03-tcp-vs-udp.md)
