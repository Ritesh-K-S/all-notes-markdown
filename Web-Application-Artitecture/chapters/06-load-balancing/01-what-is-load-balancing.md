# What is Load Balancing?

> **What you'll learn**: Why one server is never enough for real traffic, how load balancers distribute incoming requests across multiple servers, and why this is the single most important concept for building reliable web applications.

---

## Real-Life Analogy — The Restaurant Host

Imagine you walk into a busy restaurant. There's a **host** standing at the entrance. Their job? To look at all available tables (servers) and seat you at the one that will give you the best experience — maybe the one with the least number of customers, or the one closest to the kitchen for faster service.

Without the host:
- Everyone crowds around one waiter → slow service, dropped orders
- Some tables are empty while others are overwhelmed
- If a waiter calls in sick, customers standing in their area are stuck

With the host:
- Customers are distributed evenly across all available waiters
- If one waiter is overwhelmed, new customers go to a less busy one
- If a waiter goes home sick, the host simply stops seating people there

**A load balancer IS that restaurant host — but for web traffic.**

```
WITHOUT Load Balancer:                   WITH Load Balancer:

All users → One Server                   Users → Load Balancer → Server 1
            (overloaded!)                                      → Server 2
            (crashes!)                                         → Server 3
                                                              → Server 4
                                         (traffic spread evenly!)
```

---

## Core Concept Explained Step-by-Step

### Step 1: The Single Server Problem

When you launch a website on ONE server, everything works fine... until it doesn't.

```
Day 1: 100 users/day
┌────────┐         ┌──────────┐
│ Users  │────────▶│  Server  │   ✅ Happy server, fast responses
│ (100)  │         │          │
└────────┘         └──────────┘

Day 30: 10,000 users/day
┌────────┐         ┌──────────┐
│ Users  │════════▶│  Server  │   ❌ CPU at 100%, RAM full
│(10,000)│         │ (dying!) │      Response time: 8 seconds
└────────┘         └──────────┘      Some requests DROPPED!

Day 31: Server crashes completely
┌────────┐         ┌──────────┐
│ Users  │────X───▶│  Server  │   💀 503 Service Unavailable
│(10,000)│         │  (dead)  │      ALL users affected
└────────┘         └──────────┘
```

**Problems with a single server:**

| Problem | Impact |
|---------|--------|
| Limited capacity | Can only handle so many requests per second |
| Single point of failure | If it dies, EVERYONE is affected |
| No maintenance window | Can't update software without downtime |
| No scalability | To handle more traffic, you need a bigger server ($$$) |

### Step 2: The Solution — Multiple Servers + Load Balancer

Instead of one beefy server, use multiple smaller servers with a **load balancer** in front:

```
                         ┌──────────────┐
                    ┌───▶│   Server 1   │
                    │    │  (healthy)   │
┌────────┐    ┌────┴────┐    └──────────────┘
│        │    │  LOAD   │    ┌──────────────┐
│ Users  │───▶│BALANCER │───▶│   Server 2   │
│(10,000)│    │         │    │  (healthy)   │
└────────┘    └────┬────┘    └──────────────┘
                    │    ┌──────────────┐
                    └───▶│   Server 3   │
                         │  (healthy)   │
                         └──────────────┘

Each server handles ~3,333 users. Easy! 😊
```

### Step 3: What Exactly Does a Load Balancer Do?

A load balancer performs these core functions:

```
┌─────────────────────────────────────────────────────────────┐
│              LOAD BALANCER RESPONSIBILITIES                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. DISTRIBUTE TRAFFIC                                      │
│     Spread incoming requests across multiple servers        │
│                                                             │
│  2. HEALTH CHECKING                                         │
│     Continuously verify servers are alive and responding    │
│                                                             │
│  3. REMOVE SICK SERVERS                                     │
│     Stop sending traffic to servers that are down           │
│                                                             │
│  4. SSL TERMINATION                                         │
│     Handle HTTPS encryption/decryption                      │
│                                                             │
│  5. SESSION MANAGEMENT                                      │
│     Optionally route same user to same server               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Step 4: How a Request Flows Through a Load Balancer

```
Step-by-step: User visits www.myapp.com

1. Browser looks up DNS:
   www.myapp.com → 52.23.186.100 (Load Balancer's IP)

2. Browser sends HTTP request to Load Balancer:
   GET /api/products HTTP/1.1
   Host: www.myapp.com

3. Load Balancer picks a server (using an algorithm):
   "Server 2 has the fewest active connections, send it there"

4. Load Balancer forwards request to Server 2:
   GET /api/products HTTP/1.1
   Host: www.myapp.com
   X-Forwarded-For: 203.0.113.50 (original user IP)

5. Server 2 processes request, returns response to LB

6. Load Balancer returns response to user's browser

┌──────────┐    ┌─────────────┐    ┌──────────────┐    ┌────────┐
│  Browser │───▶│     DNS     │    │              │    │Server 1│
│          │    │ (resolves   │    │              │───▶│        │
│          │    │  to LB IP)  │    │    LOAD      │    └────────┘
│          │    └─────────────┘    │   BALANCER   │    ┌────────┐
│          │──────────────────────▶│              │───▶│Server 2│ ✓
│          │◀──────────────────────│              │    └────────┘
│          │                       │              │    ┌────────┐
└──────────┘                       │              │───▶│Server 3│
                                   └──────────────┘    └────────┘
```

### Step 5: Load Balancer vs Reverse Proxy — What's the Difference?

```
┌─────────────────────────────────────────────────────────┐
│ Reverse Proxy (1 or few servers):                       │
│ - Sits between client and servers                       │
│ - Caching, compression, SSL                            │
│ - Can forward to just ONE server                       │
│                                                         │
│ Load Balancer (multiple servers):                       │
│ - Sits between client and MANY servers                 │
│ - Distributes traffic using algorithms                 │
│ - Health checks, failover                              │
│                                                         │
│ In practice: MOST load balancers are also reverse      │
│ proxies. Nginx, HAProxy, and ALB do BOTH.             │
└─────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### The Two Main Types (Preview)

Load balancers operate at different network layers:

```
┌──────────────────────────────────────────────────────────────┐
│                    OSI MODEL (Simplified)                      │
├──────────────────────────────────────────────────────────────┤
│  Layer 7: Application  (HTTP, HTTPS, WebSocket)              │
│           → L7 Load Balancer: reads URLs, headers, cookies   │
│                                                              │
│  Layer 4: Transport    (TCP, UDP)                            │
│           → L4 Load Balancer: sees only IP + port            │
│                                                              │
│  Layer 3: Network      (IP)                                  │
│  Layer 2: Data Link    (Ethernet)                            │
│  Layer 1: Physical     (cables)                              │
└──────────────────────────────────────────────────────────────┘
```

> We'll dive deep into L4 vs L7 in Chapter 6.2.

### Connection Handling

When a load balancer receives a connection:

```
OPTION A: Connection Termination (most common)
┌────────┐   Connection 1   ┌────────┐  Connection 2   ┌────────┐
│ Client │──────────────────▶│   LB   │────────────────▶│ Server │
│        │◀──────────────────│        │◀────────────────│        │
└────────┘                   └────────┘                 └────────┘
LB terminates client connection, opens NEW connection to server.
Two separate TCP connections exist.

OPTION B: Connection Passthrough (L4, faster)
┌────────┐          Single Connection            ┌────────┐
│ Client │──────────────────────────────────────▶│ Server │
│        │◀──────────────────────────────────────│        │
└────────┘        LB just routes packets         └────────┘
LB acts as a transparent router. Doesn't terminate connection.
```

### Where Load Balancers Sit in Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    TYPICAL ARCHITECTURE                           │
│                                                                 │
│   Internet                                                      │
│      │                                                          │
│      ▼                                                          │
│  ┌──────────────────┐                                          │
│  │  DNS (Route 53)  │  → Resolves to LB's IP address           │
│  └────────┬─────────┘                                          │
│           ▼                                                     │
│  ┌──────────────────┐                                          │
│  │  External LB     │  → Internet-facing (handles SSL)         │
│  │  (AWS ALB)       │                                          │
│  └────────┬─────────┘                                          │
│           │                                                     │
│    ┌──────┼──────┐                                             │
│    ▼      ▼      ▼                                             │
│  ┌────┐ ┌────┐ ┌────┐                                         │
│  │Web │ │Web │ │Web │  → Frontend/API servers                  │
│  │ 1  │ │ 2  │ │ 3  │                                         │
│  └──┬─┘ └──┬─┘ └──┬─┘                                         │
│     └───────┼──────┘                                            │
│             ▼                                                   │
│  ┌──────────────────┐                                          │
│  │  Internal LB     │  → For backend-to-backend communication  │
│  └────────┬─────────┘                                          │
│           │                                                     │
│    ┌──────┼──────┐                                             │
│    ▼      ▼      ▼                                             │
│  ┌────┐ ┌────┐ ┌────┐                                         │
│  │Svc │ │Svc │ │Svc │  → Microservices                        │
│  │ A  │ │ B  │ │ C  │                                         │
│  └────┘ └────┘ └────┘                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Simple Load Balancer Concept

```python
# simple_load_balancer.py — Demonstrates round-robin load balancing concept
import itertools
import requests

# List of backend servers
SERVERS = [
    "http://server1:8080",
    "http://server2:8080",
    "http://server3:8080",
]

# Round-robin iterator: cycles through servers endlessly
server_cycle = itertools.cycle(SERVERS)

def get_next_server():
    """Pick the next server using round-robin."""
    return next(server_cycle)

def handle_request(path: str) -> dict:
    """Forward a request to the next available server."""
    server = get_next_server()
    print(f"Routing request to: {server}")
    
    response = requests.get(f"{server}{path}", timeout=5)
    return {
        "status": response.status_code,
        "body": response.json(),
        "served_by": server,
    }

# Simulate 6 requests — watch them distribute evenly:
# Request 1 → server1, Request 2 → server2, Request 3 → server3
# Request 4 → server1, Request 5 → server2, Request 6 → server3
for i in range(6):
    result = handle_request("/api/data")
    print(f"Request {i+1}: served by {result['served_by']}")
```

### Java — Simple Load Balancer Concept

```java
// SimpleLoadBalancer.java — Round-robin load balancing concept
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;
import java.net.http.*;
import java.net.URI;

public class SimpleLoadBalancer {
    private final List<String> servers;
    private final AtomicInteger currentIndex = new AtomicInteger(0);
    
    public SimpleLoadBalancer(List<String> servers) {
        this.servers = servers;
    }
    
    // Thread-safe round-robin server selection
    public String getNextServer() {
        int index = currentIndex.getAndUpdate(i -> (i + 1) % servers.size());
        return servers.get(index);
    }
    
    public HttpResponse<String> forwardRequest(String path) throws Exception {
        String server = getNextServer();
        System.out.println("Routing to: " + server);
        
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(server + path))
            .header("X-Forwarded-For", "client-ip")  // Preserve original IP
            .build();
        
        return client.send(request, HttpResponse.BodyHandlers.ofString());
    }
    
    public static void main(String[] args) throws Exception {
        var lb = new SimpleLoadBalancer(List.of(
            "http://server1:8080",
            "http://server2:8080",
            "http://server3:8080"
        ));
        
        // Requests distribute: server1, server2, server3, server1...
        for (int i = 0; i < 6; i++) {
            System.out.println("Request " + (i+1) + " → " + lb.getNextServer());
        }
    }
}
```

---

## Infrastructure Examples

### Nginx as a Load Balancer (Simplest Setup)

```nginx
# /etc/nginx/nginx.conf — Basic load balancer configuration
http {
    # Define a group of backend servers
    upstream backend_servers {
        server 10.0.1.1:8080;   # Server 1
        server 10.0.1.2:8080;   # Server 2
        server 10.0.1.3:8080;   # Server 3
    }

    server {
        listen 80;
        server_name www.myapp.com;

        location / {
            # Forward all requests to the upstream group
            proxy_pass http://backend_servers;
            
            # Pass original client information to backend
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

### AWS Application Load Balancer (via CLI)

```bash
# Create a load balancer on AWS
aws elbv2 create-load-balancer \
  --name my-app-lb \
  --subnets subnet-abc123 subnet-def456 \
  --security-groups sg-12345678 \
  --type application

# Create a target group (the servers to balance across)
aws elbv2 create-target-group \
  --name my-app-targets \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-abc123 \
  --health-check-path /health

# Register your servers in the target group
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-app-targets/... \
  --targets Id=i-server1 Id=i-server2 Id=i-server3
```

---

## Real-World Example

### How Netflix Uses Load Balancing

```
Netflix Architecture (simplified):

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  User's Device (TV, Phone, Browser)                             │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────────┐                                            │
│  │  AWS Route 53   │  DNS-level: routes to nearest region       │
│  └────────┬────────┘                                            │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │  AWS ALB/NLB    │  Regional LB: distributes across AZs      │
│  └────────┬────────┘                                            │
│           ▼                                                      │
│  ┌─────────────────┐                                            │
│  │  Zuul Gateway   │  Netflix's own L7 LB + gateway             │
│  │  (custom LB)    │  Rate limiting, auth, routing              │
│  └────────┬────────┘                                            │
│           ▼                                                      │
│  ┌────┐ ┌────┐ ┌────┐                                          │
│  │Svc │ │Svc │ │Svc │  Hundreds of microservices               │
│  │    │ │    │ │    │  Each with their own internal LB          │
│  └────┘ └────┘ └────┘  (Eureka + Ribbon for service-to-service)│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

Netflix handles 250+ million subscribers with MULTIPLE layers of
load balancing at different levels of the stack.
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| No health checks | Traffic sent to dead servers → errors | Always configure health checks (Chapter 6.4) |
| Single load balancer | LB itself becomes single point of failure | Use multiple LBs (active-passive or active-active) |
| Ignoring session state | User's cart disappears on next request | Use sticky sessions or external session store (Redis) |
| Not forwarding client IP | Backend can't see real user IP | Set `X-Forwarded-For` header |
| Over-provisioning LB | Expensive, wasted resources | Right-size based on actual traffic metrics |
| Under-provisioning servers | All servers overloaded despite LB | LB can't fix insufficient backend capacity |

---

## When to Use / When NOT to Use

### ✅ Use a Load Balancer When:

- You have **more than one server** (or plan to scale to more than one)
- You need **high availability** (if one server dies, others handle traffic)
- You need **zero-downtime deployments** (take servers out one at a time)
- Your application is **stateless** (or uses external session storage)
- Traffic is **unpredictable** and you want to add/remove servers dynamically

### ❌ You DON'T Need a Load Balancer When:

- You have a **single server** and don't plan to scale (hobby project)
- Your traffic is **trivially small** (< 100 requests/second on a decent server)
- You're building a **local development** environment
- Cost is a concern and you're still in **very early stages** (but plan to add one soon!)

---

## Key Takeaways

1. **A load balancer distributes incoming traffic** across multiple servers, preventing any single server from becoming overwhelmed.

2. **It solves three critical problems**: capacity (handle more traffic), availability (survive server failures), and maintainability (deploy without downtime).

3. **The load balancer itself should never be a single point of failure** — use cloud-managed LBs (AWS ALB/NLB) or redundant pairs.

4. **Every major website uses load balancing** — from simple round-robin for small sites to complex multi-layer setups for Netflix-scale traffic.

5. **Load balancers also handle SSL termination, health checking, and request routing** — they're much more than simple traffic splitters.

6. **Your application must be stateless** to work properly behind a load balancer — store sessions in Redis/database, not in server memory.

7. **This is the #1 architectural pattern for scaling web applications** — you'll use it everywhere from here on.

---

## What's Next?

Now that you understand WHAT a load balancer does, it's time to go deeper. In **Chapter 6.2: Layer 4 vs Layer 7 Load Balancing**, we'll explore the two fundamentally different TYPES of load balancers — one that looks at network packets (L4) and one that understands HTTP (L7) — and when to use each one.
