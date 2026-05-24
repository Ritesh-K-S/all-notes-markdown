# Sticky Sessions (Session Affinity)

> **What you'll learn**: Why some applications need the same user to always reach the same server, how sticky sessions work at the load balancer level, the different implementation methods (cookie-based, IP-based, header-based), and why you should almost always avoid this pattern in favor of externalized state.

---

## Real-Life Analogy — Your Regular Barber

You walk into a barbershop with 5 barbers. The first time, you get assigned to Barber #3. He remembers:
- How you like your hair cut
- Your preferred length
- Your conversation from last time

Next visit, you go to Barber #1 instead. He knows NOTHING about you. You have to explain everything again. Frustrating!

**Sticky sessions = always going to YOUR barber.**

The receptionist (load balancer) remembers: "Ah, you were here before with Barber #3. Let me send you back to him."

But what if Barber #3 calls in sick? You're stuck with no one — or you start over with someone new who doesn't know your preferences.

**The better solution?** Every barber writes your preferences in a SHARED notebook (external session store). Now ANY barber can serve you perfectly.

---

## Core Concept Explained Step-by-Step

### Step 1: The Problem — Why Sticky Sessions Exist

```
SCENARIO: Shopping cart stored in server memory

Request 1: Add iPhone to cart → Server A
Server A memory: { user_123: cart: ["iPhone"] }

Request 2: Add AirPods to cart → Server B (different server!)
Server B memory: { user_123: cart: ["AirPods"] }
(Server B doesn't know about the iPhone!)

Request 3: View cart → Server C
Server C memory: { user_123: cart: [] }
"Your cart is EMPTY!" ← User is confused and angry!

┌────────────┐     ┌────────┐     ┌──────────┐
│  User 123  │     │   LB   │     │ Server A │ cart: ["iPhone"]
│            │────▶│        │────▶│          │
│ "Add iPhone"     │        │     └──────────┘
└────────────┘     │        │     ┌──────────┐
                   │        │────▶│ Server B │ cart: ["AirPods"]
│ "Add AirPods"───▶│        │     │          │
                   │        │     └──────────┘
                   │        │     ┌──────────┐
│ "View cart" ────▶│        │────▶│ Server C │ cart: []  😱
                   └────────┘     └──────────┘
```

### Step 2: The Sticky Session Solution

```
WITH STICKY SESSIONS:
LB remembers: "User 123 was on Server A. Keep sending them there."

Request 1: Add iPhone to cart → Server A
Request 2: Add AirPods to cart → Server A (SAME server!)
Request 3: View cart → Server A (SAME server!)

Server A memory: { user_123: cart: ["iPhone", "AirPods"] }  ✅

┌────────────┐     ┌────────┐     ┌──────────┐
│  User 123  │     │   LB   │     │ Server A │ cart: ["iPhone", "AirPods"]
│            │────▶│  "User │────▶│          │ ✅ All requests here!
│ All requests     │   123  │     └──────────┘
│            │     │   goes │     ┌──────────┐
└────────────┘     │   to A"│     │ Server B │ (not used for this user)
                   │        │     └──────────┘
                   └────────┘     ┌──────────┐
                                  │ Server C │ (not used for this user)
                                  └──────────┘
```

### Step 3: How Sticky Sessions Work — Implementation Methods

```
METHOD 1: Cookie-Based (Most Common)
─────────────────────────────────────
First request (no cookie):
Client ──── GET /cart ────▶ LB ──── (picks Server A) ────▶ Server A
Client ◀─── Response + Set-Cookie: SERVERID=server-a ◀──── LB

Second request (has cookie):
Client ──── GET /cart ────▶ LB
            Cookie: SERVERID=server-a
            LB reads cookie → "Send to Server A"
Client ◀─── Response ◀───── LB ◀──── Server A

METHOD 2: IP-Based (Source IP Hash)
─────────────────────────────────────
hash(client_ip) → always maps to same server
No cookies needed, works for non-HTTP too.
Problem: Users behind NAT/proxy share same IP!

METHOD 3: URL/Header-Based
─────────────────────────────────────
Custom header or URL parameter identifies the session:
GET /api/data?session_server=server-a
X-Sticky-Server: server-a
```

### Step 4: Cookie-Based Sticky Sessions — In Detail

```
LOAD BALANCER COOKIE FLOW:

┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  FIRST REQUEST (no session cookie):                               │
│                                                                    │
│  ┌────────┐   GET /page HTTP/1.1      ┌────────┐     ┌────────┐  │
│  │ Client │──────────────────────────▶│   LB   │────▶│Srv A   │  │
│  │        │   (no SERVERID cookie)    │        │     │        │  │
│  │        │                           │ Picks  │     │        │  │
│  │        │◀──────────────────────────│ Srv A  │◀────│        │  │
│  │        │   HTTP 200 OK             │        │     │        │  │
│  │        │   Set-Cookie: SERVERID=   │        │     └────────┘  │
│  │        │     srv-a-encrypted-id    │        │                  │
│  └────────┘                           └────────┘                  │
│                                                                    │
│  ALL SUBSEQUENT REQUESTS:                                         │
│                                                                    │
│  ┌────────┐   GET /cart HTTP/1.1      ┌────────┐     ┌────────┐  │
│  │ Client │──────────────────────────▶│   LB   │────▶│Srv A   │  │
│  │        │   Cookie: SERVERID=       │        │     │        │  │
│  │        │     srv-a-encrypted-id    │Reads   │     │(same!) │  │
│  │        │                           │cookie  │     │        │  │
│  │        │◀──────────────────────────│→Srv A  │◀────│        │  │
│  └────────┘                           └────────┘     └────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Step 5: What Happens When the Sticky Server Dies?

```
FAILURE SCENARIO:

┌────────┐     ┌────────┐     ┌──────────┐
│ User   │────▶│   LB   │────▶│ Server A │ 💀 DEAD!
│        │     │        │     │ (crashed)│
│ Cookie:│     │ "A is  │     └──────────┘
│ Srv=A  │     │  dead!"│     
└────────┘     └────┬───┘     
                    │
        What happens?
                    │
         ┌──────────┴──────────┐
         │   Two Options:      │
         │                     │
    ┌────┴────┐          ┌────┴────┐
    │OPTION A │          │OPTION B │
    │Fail open│          │Error    │
    │(re-route│          │(return  │
    │to Srv B)│          │  503)   │
    └─────────┘          └─────────┘
    
    Option A (best): Route to another server.
    User loses session state but service continues.
    
    Option B (bad): Return error to user.
    Never do this!
```

---

## How It Works Internally

### Session Affinity Table (LB Internal State)

```
LOAD BALANCER INTERNAL SESSION TABLE:

┌─────────────────────────────────────────────────────────────┐
│                  SESSION AFFINITY TABLE                       │
├──────────────────┬──────────────┬────────────┬──────────────┤
│  Session Key     │  Server      │  Created   │  Expires     │
├──────────────────┼──────────────┼────────────┼──────────────┤
│  cookie:abc123   │  server-a    │  10:00:00  │  10:30:00    │
│  cookie:def456   │  server-b    │  10:02:15  │  10:32:15    │
│  cookie:ghi789   │  server-a    │  10:05:30  │  10:35:30    │
│  ip:203.0.113.50 │  server-c    │  10:07:00  │  10:37:00    │
│  cookie:jkl012   │  server-b    │  10:08:45  │  10:38:45    │
└──────────────────┴──────────────┴────────────┴──────────────┘

The LB maintains this mapping in memory.
When a request arrives, it looks up the session key:
  - Found → route to stored server
  - Not found → use normal algorithm, create new entry
  - Expired → delete entry, use normal algorithm
```

### The Problem: Uneven Load Distribution

```
LOAD IMBALANCE WITH STICKY SESSIONS:

Normal (no stickiness):
Server A: ████████████  (33% traffic)
Server B: ████████████  (33% traffic)
Server C: ████████████  (33% traffic)

With sticky sessions (power users stuck on A):
Server A: ████████████████████████  (60% traffic!) ⚠️
Server B: ████████  (25% traffic)
Server C: █████  (15% traffic)

WHY? Some users make way more requests than others.
If a "whale" user (makes 1000 req/min) is stuck on Server A,
that server is disproportionately loaded.

This DEFEATS the purpose of load balancing!
```

---

## Code Examples

### Python — Implementing Session-Aware Routing

```python
# sticky_session_lb.py — Load balancer with cookie-based session affinity
import hashlib
import time
from flask import Flask, request, Response
import httpx

app = Flask(__name__)

SERVERS = ["http://server-a:8080", "http://server-b:8080", "http://server-c:8080"]
COOKIE_NAME = "LB_STICKY_SERVER"
SESSION_TTL = 1800  # 30 minutes

# In-memory session table (in production: use shared store)
session_table = {}  # {cookie_value: {"server": url, "expires": timestamp}}

def get_sticky_server(cookie_value: str) -> str:
    """Look up existing session affinity."""
    if cookie_value in session_table:
        entry = session_table[cookie_value]
        if time.time() < entry["expires"]:
            return entry["server"]
        else:
            del session_table[cookie_value]  # Expired
    return None

def assign_server(cookie_value: str) -> str:
    """Assign a server and record the affinity."""
    # Use hash-based selection for deterministic assignment
    index = int(hashlib.md5(cookie_value.encode()).hexdigest(), 16) % len(SERVERS)
    server = SERVERS[index]
    session_table[cookie_value] = {
        "server": server,
        "expires": time.time() + SESSION_TTL
    }
    return server

@app.route('/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
def proxy(path):
    """Route request with sticky session support."""
    cookie_value = request.cookies.get(COOKIE_NAME)
    
    if cookie_value:
        # Existing session — try to route to same server
        server = get_sticky_server(cookie_value)
        if server is None:
            # Server died or session expired — reassign
            server = assign_server(cookie_value)
    else:
        # New session — generate cookie and assign server
        cookie_value = hashlib.sha256(f"{time.time()}{request.remote_addr}".encode()).hexdigest()[:16]
        server = assign_server(cookie_value)
    
    # Forward request to chosen server
    resp = httpx.request(method=request.method, url=f"{server}/{path}")
    
    # Build response with sticky cookie
    response = Response(resp.content, status=resp.status_code)
    response.set_cookie(COOKIE_NAME, cookie_value, max_age=SESSION_TTL, httponly=True)
    return response
```

### Java — Sticky Session with Failover

```java
// StickySessionLoadBalancer.java — Cookie-based affinity with failover
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.time.Instant;

public class StickySessionLoadBalancer {
    
    private final List<String> servers;
    private final ConcurrentHashMap<String, SessionEntry> sessionTable = new ConcurrentHashMap<>();
    private final Set<String> healthyServers = ConcurrentHashMap.newKeySet();
    private static final long SESSION_TTL_SECONDS = 1800;
    
    record SessionEntry(String server, Instant expiresAt) {}
    
    public StickySessionLoadBalancer(List<String> servers) {
        this.servers = servers;
        this.healthyServers.addAll(servers);  // All start healthy
    }
    
    public String routeRequest(String sessionCookie) {
        // Try existing affinity
        if (sessionCookie != null && sessionTable.containsKey(sessionCookie)) {
            SessionEntry entry = sessionTable.get(sessionCookie);
            
            // Check if session is still valid AND server is healthy
            if (Instant.now().isBefore(entry.expiresAt()) 
                    && healthyServers.contains(entry.server())) {
                return entry.server();  // Same server as before!
            }
            
            // Server died or session expired — need failover
            sessionTable.remove(sessionCookie);
            System.out.println("Sticky server unavailable, re-routing...");
        }
        
        // Assign new server (round-robin among healthy servers)
        String newServer = pickHealthyServer();
        String newCookie = sessionCookie != null ? sessionCookie : generateCookie();
        
        sessionTable.put(newCookie, new SessionEntry(
            newServer, 
            Instant.now().plusSeconds(SESSION_TTL_SECONDS)
        ));
        
        return newServer;
    }
    
    private String pickHealthyServer() {
        return healthyServers.stream().findFirst()
            .orElseThrow(() -> new RuntimeException("No healthy servers!"));
    }
    
    private String generateCookie() {
        return UUID.randomUUID().toString().substring(0, 16);
    }
    
    // Called by health checker when server goes down
    public void markUnhealthy(String server) {
        healthyServers.remove(server);
        // All sessions on this server will failover on next request
    }
}
```

---

## Infrastructure Examples

### Nginx Sticky Sessions

```nginx
http {
    # Method 1: IP Hash (simple but inaccurate behind NAT)
    upstream backend_iphash {
        ip_hash;
        server 10.0.1.1:8080;
        server 10.0.1.2:8080;
        server 10.0.1.3:8080;
    }
    
    # Method 2: Cookie-based (Nginx Plus only)
    upstream backend_sticky {
        sticky cookie srv_id expires=1h domain=.myapp.com path=/;
        server 10.0.1.1:8080;
        server 10.0.1.2:8080;
        server 10.0.1.3:8080;
    }
    
    # Method 3: Sticky via application cookie (route by existing cookie)
    upstream backend_appcookie {
        sticky route $cookie_jsessionid;
        server 10.0.1.1:8080 route=a;
        server 10.0.1.2:8080 route=b;
        server 10.0.1.3:8080 route=c;
    }
}
```

### AWS ALB Sticky Sessions

```bash
# Enable cookie-based stickiness on target group
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-targets/... \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=3600

# ALB-generated cookie (AWSALB):
# - ALB creates and manages the cookie automatically
# - Cookie is encrypted, contains server mapping
# - Duration configurable (1 second to 7 days)

# Alternative: Application-based stickiness
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-targets/... \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=app_cookie \
    Key=stickiness.app_cookie.cookie_name,Value=JSESSIONID \
    Key=stickiness.app_cookie.duration_seconds,Value=3600
```

---

## Real-World Example

### The Better Alternative: Externalized Session State

```
THE CORRECT SOLUTION (for most apps):

Instead of sticky sessions, store session data EXTERNALLY:

┌────────────┐     ┌────────┐     ┌──────────┐
│  User 123  │────▶│   LB   │────▶│ Server A │──┐
│            │     │(no     │     └──────────┘  │
│            │     │sticky!)│     ┌──────────┐  ├──▶ ┌───────────┐
│            │     │        │────▶│ Server B │──┤    │   Redis   │
│            │     │        │     └──────────┘  │    │  (shared  │
│            │     │        │     ┌──────────┐  │    │  session  │
│            │     │        │────▶│ Server C │──┘    │   store)  │
└────────────┘     └────────┘     └──────────┘      └───────────┘

Request 1 → Server A: reads cart from Redis
Request 2 → Server C: reads SAME cart from Redis
Request 3 → Server B: reads SAME cart from Redis

All servers can serve any user! No stickiness needed!

BENEFITS:
✅ Even load distribution (no hot servers)
✅ Server can die — user doesn't lose state
✅ Can scale servers up/down freely
✅ Zero-downtime deployments (no session migration)

HOW:
- Store sessions in Redis (fast, in-memory)
- Or in PostgreSQL (durable, slower)
- Or in DynamoDB (managed, scalable)
```

```python
# The right way: externalized sessions with Redis
from flask import Flask, session
from flask_session import Session
import redis

app = Flask(__name__)
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_REDIS'] = redis.Redis(host='redis-cluster', port=6379)
Session(app)

@app.route('/cart/add', methods=['POST'])
def add_to_cart():
    # Session automatically stored in Redis — works on ANY server!
    if 'cart' not in session:
        session['cart'] = []
    session['cart'].append(request.json['item'])
    return {"cart": session['cart']}
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using sticky sessions as the default | Causes uneven load, reduces availability | Default to stateless + external session store |
| No failover plan when sticky server dies | User gets error or infinite redirect | Configure LB to route to any healthy server on failure |
| Very long sticky session TTL (24h+) | Users stuck on bad/slow servers for too long | Keep TTL short (30 min) or match session expiry |
| Sticky sessions + auto-scaling | New servers get no traffic (all users stuck to old ones) | Consider session draining or shorter TTL |
| IP-based stickiness with corporate users | Entire office (1000 users) maps to ONE server | Use cookie-based instead of IP-based |
| Not considering WebSocket sticky needs | WebSocket requires same server for the connection lifetime | WebSocket naturally stays connected — stickiness is the TCP connection itself |

---

## When to Use / When NOT to Use

### ❌ DO NOT Use Sticky Sessions When (Most Cases):

- You CAN externalize state (Redis, database, JWT tokens)
- You need even load distribution
- You want zero-downtime deployments
- You're running auto-scaling infrastructure
- You're building a new application (design it stateless!)

### ✅ Use Sticky Sessions ONLY When:

| Scenario | Why |
|----------|-----|
| Legacy application that stores state in memory | Can't refactor to external store yet |
| WebSocket connections | Must maintain connection to same server (but this is natural TCP affinity, not LB stickiness) |
| Local caching optimization | Server has warmed cache for this user's data |
| File uploads in progress | Chunked upload parts must go to same server |
| Specific compliance requirements | Some regulated systems require session isolation |

### Decision Flowchart:

```
Can you store session state externally (Redis/DB)?
├── YES → Don't use sticky sessions. Period. ✓
└── NO  → Can you refactor to externalize?
    ├── YES → Refactor first, then no sticky sessions ✓
    └── NO (legacy/constraint) → Use sticky sessions as temporary solution
        └── Plan migration to external state store!
```

---

## Key Takeaways

1. **Sticky sessions route the same user to the same server** — solving the problem of in-memory session state being lost across servers.

2. **Cookie-based stickiness is the most reliable method** — IP-based breaks behind NAT/proxies, cookies are per-user.

3. **Sticky sessions are an ANTI-PATTERN for modern applications** — they reduce availability, cause uneven load, and complicate scaling.

4. **The correct solution is externalized state** — store sessions in Redis, DynamoDB, or PostgreSQL so ANY server can handle ANY user.

5. **If you must use sticky sessions, always configure failover** — when the sticky server dies, route to another server (user loses session but gets service).

6. **Sticky sessions defeat the purpose of load balancing** — a user "stuck" to a server doesn't benefit from other servers being available.

7. **Design new applications to be stateless** — stateless servers behind a load balancer is the gold standard for scalable architectures.

---

## What's Next?

We've covered single-region load balancing. But what happens when your users are scattered across the globe? In **Chapter 6.6: Global Server Load Balancing (GSLB) & GeoDNS**, we'll explore how to route users to the nearest data center using DNS-level load balancing, reducing latency from hundreds of milliseconds to single digits.
