# Health Checks — How Load Balancers Know a Server is Alive

> **What you'll learn**: How load balancers continuously monitor backend servers to detect failures, the difference between active and passive health checks, what makes a good health check endpoint, and how to configure checks that catch real problems without causing false alarms.

---

## Real-Life Analogy — The Night Security Guard

Imagine a security guard patrolling a building every 30 minutes:

- **Active health check**: The guard walks to each room, knocks on the door, and waits for a response. "Room 3, you okay?" → "All good!" If no response after 3 knocks, mark that room as "problem — don't send anyone there."

- **Passive health check**: The guard doesn't patrol. Instead, they listen for screams or alarms. If they hear something bad from Room 3 three times in a row, they mark it as unsafe.

- **Deep health check**: The guard not only knocks but also checks: "Is your plumbing working? Is your electricity on? Do you have supplies?" — validates EVERYTHING is working, not just that someone answered the door.

A load balancer does exactly this — it constantly checks if servers are healthy and stops sending traffic to sick ones.

---

## Core Concept Explained Step-by-Step

### Step 1: Why Health Checks Matter

Without health checks:
```
┌────────┐         ┌────────┐         ┌──────────┐
│  User  │────────▶│   LB   │────────▶│ Server 1 │ ✅ Healthy
└────────┘         │        │         └──────────┘
                   │        │         ┌──────────┐
                   │        │────────▶│ Server 2 │ 💀 DEAD (crashed!)
                   │        │         └──────────┘
                   │        │         ┌──────────┐
                   │        │────────▶│ Server 3 │ ✅ Healthy
                   └────────┘         └──────────┘

Without health checks:
- 33% of users get routed to the dead Server 2
- They see 503 errors, timeouts, blank pages
- LB keeps sending traffic there because it doesn't KNOW it's dead!
```

With health checks:
```
┌────────┐         ┌────────┐    "You alive?"     ┌──────────┐
│  User  │────────▶│   LB   │───────────────────▶│ Server 1 │ ✅ "Yes!"
└────────┘         │        │                     └──────────┘
                   │        │    "You alive?"     ┌──────────┐
                   │        │────────X───────────▶│ Server 2 │ 💀 (no response)
                   │        │  ↑                  └──────────┘
                   │        │  └─ "3 failures in a row → REMOVE from pool!"
                   │        │                     ┌──────────┐
                   │        │    "You alive?"     │ Server 3 │ ✅ "Yes!"
                   └────────┘───────────────────▶└──────────┘

Now: 0% of users hit the dead server!
Traffic splits 50/50 between Server 1 and Server 3.
```

### Step 2: Active vs Passive Health Checks

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  ACTIVE HEALTH CHECKS:                                             │
│  LB proactively sends test requests to servers on a schedule       │
│                                                                     │
│  LB ──── GET /health ──── every 10 seconds ────▶ Server            │
│      ◀─── HTTP 200 OK ─────────────────────────── Server           │
│                                                                     │
│  ✅ Detects failures BEFORE users are affected                      │
│  ✅ Can check deep functionality (DB, cache, dependencies)          │
│  ❌ Adds extra traffic/load to servers                              │
│  ❌ Slight delay between failure and detection (check interval)     │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PASSIVE HEALTH CHECKS:                                            │
│  LB monitors actual user traffic responses for errors              │
│                                                                     │
│  User ──── request ────▶ LB ──── forward ────▶ Server              │
│                          LB ◀─── 500 Error ──── Server             │
│                          LB: "That's 3 errors in 60s → UNHEALTHY!" │
│                                                                     │
│  ✅ No extra traffic — uses real requests                           │
│  ✅ Catches issues that health endpoints don't test                 │
│  ❌ Users experience errors BEFORE server is marked down            │
│  ❌ Low-traffic servers might not get checked often enough          │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  BEST PRACTICE: Use BOTH!                                          │
│  Active: catches server crashes/network issues immediately          │
│  Passive: catches application bugs/slowness in real traffic         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 3: Health Check Types

```
┌──────────────────────────────────────────────────────────────────┐
│                    HEALTH CHECK DEPTH LEVELS                       │
│                                                                  │
│  LEVEL 1: TCP Check (Layer 4)                                    │
│  ─────────────────────────────                                   │
│  "Can I open a TCP connection to port 8080?"                    │
│  LB ──── SYN ────▶ Server                                       │
│  LB ◀── SYN-ACK ── Server  → HEALTHY                           │
│  Catches: server completely down, port not listening             │
│  Misses: app frozen, returning errors, dependencies down         │
│                                                                  │
│  LEVEL 2: HTTP Check (Layer 7)                                   │
│  ─────────────────────────────                                   │
│  "Does GET /health return HTTP 200?"                            │
│  LB ──── GET /health ────▶ Server                               │
│  LB ◀── HTTP 200 OK ────── Server  → HEALTHY                   │
│  Catches: app crashed, web server misconfigured                  │
│  Misses: database down, cache unavailable                        │
│                                                                  │
│  LEVEL 3: Deep Application Check (Layer 7 + Dependencies)        │
│  ─────────────────────────────────────────────────               │
│  "Does GET /health/deep confirm DB, cache, and queues work?"    │
│  LB ──── GET /health/deep ────▶ Server                          │
│     Server checks internally:                                    │
│       ├── Database: SELECT 1        ✅                           │
│       ├── Redis: PING               ✅                           │
│       ├── Queue: connection alive    ✅                           │
│       └── Disk: free space > 1GB    ✅                           │
│  LB ◀── HTTP 200 + JSON ──────────── Server  → HEALTHY          │
│  Catches: dependency failures, resource exhaustion               │
│  Risk: slow checks, cascading failures                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Step 4: Health Check Parameters

```
KEY CONFIGURATION PARAMETERS:

┌─────────────────────┬────────────────────────────────────────────┐
│ Parameter           │ Meaning                                     │
├─────────────────────┼────────────────────────────────────────────┤
│ Interval            │ How often to check (e.g., every 10s)       │
│ Timeout             │ How long to wait for response (e.g., 5s)   │
│ Healthy threshold   │ Checks to pass before marking UP (e.g., 2)│
│ Unhealthy threshold │ Failures before marking DOWN (e.g., 3)    │
│ Path                │ URL to check (e.g., /health)               │
│ Expected status     │ What HTTP code means "healthy" (e.g., 200) │
│ Expected body       │ Optional string to find in response body   │
└─────────────────────┴────────────────────────────────────────────┘

TIMELINE: How a server gets marked unhealthy

Time 0s:    Health check → ✅ 200 OK
Time 10s:   Health check → ✅ 200 OK
Time 20s:   Health check → ❌ Timeout (failure 1/3)
Time 30s:   Health check → ❌ 500 Error (failure 2/3)
Time 40s:   Health check → ❌ Timeout (failure 3/3)
                           └→ SERVER MARKED UNHEALTHY! Traffic removed.

Time 50s:   Health check → ✅ 200 OK (recovery 1/2)
Time 60s:   Health check → ✅ 200 OK (recovery 2/2)
                           └→ SERVER MARKED HEALTHY! Traffic resumed.

Total detection time: 3 × 10s = 30 seconds
(interval × unhealthy_threshold)
```

### Step 5: Graceful Degradation

```
WHAT HAPPENS WHEN A SERVER FAILS:

Phase 1: Detection (during health check failures)
┌───────────────────────────────┐
│ Pool: [A ✅, B ✅, C ✅]       │  Traffic: 33% each
└───────────────────────────────┘

Phase 2: Server C fails health checks
┌───────────────────────────────┐
│ Pool: [A ✅, B ✅, C ⚠️]       │  Traffic: still 33% each
│                               │  (threshold not yet met)
└───────────────────────────────┘

Phase 3: C exceeds failure threshold
┌───────────────────────────────┐
│ Pool: [A ✅, B ✅, C ❌]       │  Traffic: 50% A, 50% B
│                               │  C gets ZERO traffic
└───────────────────────────────┘

Phase 4: C recovers and passes checks
┌───────────────────────────────┐
│ Pool: [A ✅, B ✅, C ✅]       │  Traffic: 33% each again
│                               │  (C passes healthy threshold)
└───────────────────────────────┘
```

---

## How It Works Internally

### Connection Draining During Health Failure

```
When a server is marked unhealthy, existing connections need handling:

IMMEDIATE REMOVAL (aggressive):
┌─────────────────────────────────────────────────────┐
│ Server C marked unhealthy at T=0                     │
│                                                     │
│ T=0:  New requests → STOP going to C               │
│ T=0:  Existing connections → IMMEDIATELY CUT        │
│                                                     │
│ Problem: Users with active requests get errors!     │
└─────────────────────────────────────────────────────┘

CONNECTION DRAINING (graceful — best practice):
┌─────────────────────────────────────────────────────┐
│ Server C marked unhealthy at T=0                     │
│                                                     │
│ T=0:    New requests → STOP going to C              │
│ T=0-30: Existing connections → ALLOWED TO FINISH    │
│ T=30:   Drain timeout reached → remaining cut       │
│                                                     │
│ Result: In-flight requests complete normally!        │
└─────────────────────────────────────────────────────┘
```

### Flapping Prevention

```
PROBLEM: Server keeps toggling between healthy/unhealthy

Time:  0  10  20  30  40  50  60  70  80  90  100
State: ✅  ❌  ✅  ❌  ✅  ❌  ✅  ❌  ✅  ❌  ✅

This is "flapping" — causes constant pool changes, connection resets,
and inconsistent user experience.

SOLUTION: Thresholds + cooldown

Unhealthy threshold: 3 consecutive failures
Healthy threshold:   5 consecutive successes (higher to prevent flapping)
Cooldown period:     30 seconds after marking unhealthy before re-checking

Now: Server must be stable for 50 seconds straight before rejoining pool.
```

---

## Code Examples

### Python — Health Check Endpoint (Application Side)

```python
# health_check.py — What your APPLICATION server should expose
from flask import Flask, jsonify
import psycopg2
import redis
import time

app = Flask(__name__)

# --- Simple health check (Level 2: HTTP check) ---
@app.route('/health')
def basic_health():
    """Fast check — just confirms the app process is alive."""
    return jsonify({"status": "healthy", "timestamp": time.time()}), 200

# --- Deep health check (Level 3: dependency check) ---
@app.route('/health/deep')
def deep_health():
    """Comprehensive check — validates all dependencies work."""
    checks = {}
    overall_healthy = True
    
    # Check database connectivity
    try:
        conn = psycopg2.connect("postgresql://localhost/mydb", connect_timeout=3)
        conn.execute("SELECT 1")
        conn.close()
        checks["database"] = "healthy"
    except Exception as e:
        checks["database"] = f"unhealthy: {str(e)}"
        overall_healthy = False
    
    # Check Redis connectivity
    try:
        r = redis.Redis(host='localhost', socket_timeout=2)
        r.ping()
        checks["redis"] = "healthy"
    except Exception as e:
        checks["redis"] = f"unhealthy: {str(e)}"
        overall_healthy = False
    
    # Check disk space (must have > 500MB free)
    import shutil
    disk = shutil.disk_usage("/")
    free_mb = disk.free / (1024 * 1024)
    if free_mb > 500:
        checks["disk"] = f"healthy ({free_mb:.0f}MB free)"
    else:
        checks["disk"] = f"unhealthy ({free_mb:.0f}MB free)"
        overall_healthy = False
    
    status_code = 200 if overall_healthy else 503
    return jsonify({
        "status": "healthy" if overall_healthy else "unhealthy",
        "checks": checks,
        "timestamp": time.time()
    }), status_code
```

### Java — Health Check Endpoint (Spring Boot)

```java
// HealthCheckController.java — Spring Boot health endpoint
import org.springframework.web.bind.annotation.*;
import org.springframework.http.ResponseEntity;
import javax.sql.DataSource;
import java.sql.Connection;
import java.util.*;

@RestController
public class HealthCheckController {
    
    private final DataSource dataSource;
    
    public HealthCheckController(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    // Simple liveness check — "Is the process alive?"
    @GetMapping("/health")
    public ResponseEntity<Map<String, Object>> basicHealth() {
        return ResponseEntity.ok(Map.of(
            "status", "healthy",
            "timestamp", System.currentTimeMillis()
        ));
    }
    
    // Deep readiness check — "Can this server handle real requests?"
    @GetMapping("/health/deep")
    public ResponseEntity<Map<String, Object>> deepHealth() {
        Map<String, String> checks = new LinkedHashMap<>();
        boolean allHealthy = true;
        
        // Check database
        try (Connection conn = dataSource.getConnection()) {
            conn.createStatement().execute("SELECT 1");
            checks.put("database", "healthy");
        } catch (Exception e) {
            checks.put("database", "unhealthy: " + e.getMessage());
            allHealthy = false;
        }
        
        // Check available memory (must have > 100MB free)
        long freeMemory = Runtime.getRuntime().freeMemory() / (1024 * 1024);
        if (freeMemory > 100) {
            checks.put("memory", "healthy (" + freeMemory + "MB free)");
        } else {
            checks.put("memory", "unhealthy (" + freeMemory + "MB free)");
            allHealthy = false;
        }
        
        Map<String, Object> response = Map.of(
            "status", allHealthy ? "healthy" : "unhealthy",
            "checks", checks
        );
        
        int status = allHealthy ? 200 : 503;
        return ResponseEntity.status(status).body(response);
    }
}
```

---

## Infrastructure Examples

### Nginx Health Check Configuration

```nginx
http {
    upstream backend {
        zone backend_zone 64k;  # Shared memory for health state
        
        server 10.0.1.1:8080;
        server 10.0.1.2:8080;
        server 10.0.1.3:8080;
    }

    # Active health check (Nginx Plus / OpenResty)
    # Note: Nginx OSS only supports passive checks
    match health_check {
        status 200;
        header Content-Type = "application/json";
        body ~ "\"status\":\"healthy\"";
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://backend;
            
            # Passive health checks (Nginx OSS supports these)
            proxy_next_upstream error timeout http_500 http_502 http_503;
            proxy_next_upstream_tries 3;
            proxy_connect_timeout 5s;
            proxy_read_timeout 10s;
        }
        
        # Health check endpoint for the LB itself
        location /health {
            health_check interval=10 fails=3 passes=2;
            health_check_timeout 5s;
            proxy_pass http://backend;
        }
    }
}
```

### AWS ALB Health Check Configuration

```bash
# Create target group with health check settings
aws elbv2 create-target-group \
  --name my-app-targets \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-abc123 \
  --health-check-protocol HTTP \
  --health-check-path "/health" \
  --health-check-interval-seconds 15 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --matcher '{"HttpCode": "200"}'

# View target health status
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-app-targets/...
```

### HAProxy Comprehensive Health Checks

```haproxy
backend api_servers
    balance roundrobin
    option httpchk GET /health HTTP/1.1\r\nHost:\ api.myapp.com
    
    # Health check parameters
    default-server inter 10s fall 3 rise 2 slowstart 30s
    
    #   inter 10s  = check every 10 seconds
    #   fall 3     = 3 failures → mark down
    #   rise 2     = 2 successes → mark up
    #   slowstart  = gradually increase traffic for 30s after recovery
    
    server api1 10.0.1.1:8080 check
    server api2 10.0.1.2:8080 check
    server api3 10.0.1.3:8080 check
    
    # Backup server (only used when all primaries are down)
    server api_backup 10.0.1.99:8080 check backup
```

---

## Real-World Example

### How Kubernetes Health Checks Work

```
Kubernetes has THREE types of probes:

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  1. LIVENESS PROBE: "Is the container alive?"                    │
│     Failure action: RESTART the container                        │
│     Use for: detecting deadlocks, infinite loops                │
│                                                                  │
│  2. READINESS PROBE: "Can this container handle traffic?"       │
│     Failure action: REMOVE from Service (LB stops routing)      │
│     Use for: dependency checks, startup initialization          │
│                                                                  │
│  3. STARTUP PROBE: "Has the container finished starting?"       │
│     Failure action: Wait (disable liveness/readiness)           │
│     Use for: slow-starting applications                         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

YAML Example:
```

```yaml
# kubernetes-deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: my-app
        image: my-app:1.0
        ports:
        - containerPort: 8080
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15    # Wait 15s before first check
          periodSeconds: 10          # Check every 10s
          timeoutSeconds: 3          # Timeout after 3s
          failureThreshold: 3        # 3 failures → restart container
        
        readinessProbe:
          httpGet:
            path: /health/deep
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2        # 2 failures → remove from LB
          successThreshold: 2        # 2 successes → add back to LB
        
        startupProbe:
          httpGet:
            path: /health
            port: 8080
          failureThreshold: 30       # Allow 30 × 10s = 300s to start
          periodSeconds: 10
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Health check endpoint is too expensive | Checking DB on every probe adds load, can cause cascading failures | Keep `/health` cheap; use `/health/deep` with longer interval |
| Check interval too aggressive (1s) | Generates excessive traffic, may trigger rate limits | Use 10-30s intervals for most applications |
| No timeout on health check | LB waits forever for a hung server | Always set timeout < interval |
| Healthy threshold = 1 | Server rejoins pool on one lucky response, then fails again (flapping) | Use threshold of 2-3 for stability |
| Health check doesn't test real functionality | Returns 200 even when DB is down | Check critical dependencies in deep checks |
| Deep health check on every probe | External dependency slow? ALL servers marked unhealthy! | Use shallow for liveness, deep less frequently |
| Not handling graceful shutdown | Server removed from pool while processing requests | Implement connection draining (deregistration delay) |

---

## When to Use / When NOT to Use

### ✅ Always Use Health Checks When:

- Running more than one server behind a load balancer
- In any production environment (no exceptions!)
- Using auto-scaling (new instances need health validation before receiving traffic)
- Deploying updates (new version must pass health before old is removed)

### Health Check Type Selection:

```
TCP checks:
├── Good for: L4 load balancers, databases, Redis, non-HTTP services
├── Fast and lightweight
└── Can't detect application-level issues

HTTP checks (shallow):
├── Good for: Liveness probes, frequent checks (every 5-10s)
├── Validates app process is running and accepting HTTP
└── Endpoint: GET /health → 200 OK

HTTP checks (deep):
├── Good for: Readiness probes, less frequent (every 30-60s)
├── Validates full application functionality
└── Endpoint: GET /health/deep → 200 OK with dependency status
```

---

## Key Takeaways

1. **Health checks are non-negotiable in production** — without them, your load balancer will happily send traffic to dead or broken servers.

2. **Use BOTH active and passive health checks** — active catches server crashes before users notice; passive catches subtle errors in real traffic.

3. **Create two health endpoints**: a fast `/health` (just "am I alive?") and a comprehensive `/health/deep` (test all dependencies).

4. **Tune your thresholds carefully**: too sensitive = flapping (constant up/down); too tolerant = slow detection of real failures.

5. **Connection draining is critical** — when marking a server unhealthy, let existing requests finish before cutting traffic completely.

6. **Health checks should be fast** — if your health endpoint takes 2 seconds to respond, it's too expensive. Target < 100ms for shallow, < 1s for deep.

7. **In Kubernetes, understand the difference between liveness and readiness** — liveness failure restarts the container, readiness failure just removes it from the load balancer.

---

## What's Next?

Health checks keep traffic away from dead servers. But what about users who have ongoing sessions — like a shopping cart? In **Chapter 6.5: Sticky Sessions**, we'll explore how to keep a user's requests going to the same server when your application requires it, and why you should usually avoid this pattern.
