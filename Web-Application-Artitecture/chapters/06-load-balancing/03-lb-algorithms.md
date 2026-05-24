# Load Balancing Algorithms

> **What you'll learn**: The exact algorithms load balancers use to decide which server gets the next request — from simple Round Robin to intelligent Least Connections and Weighted strategies — with visual examples showing how each distributes traffic differently.

---

## Real-Life Analogy — Assigning Customers at a Bank

Imagine you're managing a bank with 4 tellers:

| Algorithm | Bank Analogy |
|-----------|-------------|
| **Round Robin** | Give ticket #1 to Teller A, #2 to B, #3 to C, #4 to D, #5 back to A... |
| **Weighted Round Robin** | Teller A is senior (handles 3 customers per rotation), Teller B is new (handles 1) |
| **Least Connections** | Look up and see who has the shortest line RIGHT NOW. Send customer there |
| **IP Hash** | "Customers whose last name starts with A-G → Teller A, H-M → Teller B..." (same customer always goes to same teller) |
| **Random** | Close your eyes and point at a teller. Send them there |

Each approach has trade-offs. Let's explore all of them.

---

## Core Concept Explained Step-by-Step

### Algorithm 1: Round Robin

The simplest algorithm. Requests are distributed to servers in sequential, circular order.

```
Servers: A, B, C

Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (starts over)
Request 5 → Server B
Request 6 → Server C
Request 7 → Server A  (and so on...)

Timeline:
─────────────────────────────────────────────────
  Req1   Req2   Req3   Req4   Req5   Req6   Req7
   │      │      │      │      │      │      │
   ▼      ▼      ▼      ▼      ▼      ▼      ▼
   A      B      C      A      B      C      A
─────────────────────────────────────────────────

Server A: 3 requests
Server B: 2 requests
Server C: 2 requests   ← Nearly equal distribution!
```

**Pros:** Dead simple, no state to maintain, perfectly fair for identical servers  
**Cons:** Doesn't account for server capacity differences or current load

**Best for:** Identical servers with similar request complexity

---

### Algorithm 2: Weighted Round Robin

Like Round Robin but servers get a "weight" based on their capacity.

```
Servers: A (weight=5), B (weight=3), C (weight=2)
Total weight = 10

For every 10 requests:
- Server A gets 5 requests (50%)
- Server B gets 3 requests (30%)
- Server C gets 2 requests (20%)

Sequence: A, A, A, A, A, B, B, B, C, C, A, A, A, A, A, B, B, B, C, C...

Visual distribution:
┌─────────────────────────────────────────────┐
│ Server A (8 CPU cores):  █████████████████   50% │
│ Server B (4 CPU cores):  ██████████         30% │
│ Server C (2 CPU cores):  ██████             20% │
└─────────────────────────────────────────────┘

Why? Because Server A is a powerful machine (8 cores),
Server C is small (2 cores). Fair ≠ Equal.
```

**Pros:** Accounts for different server sizes  
**Cons:** Static weights don't reflect real-time load

**Best for:** Heterogeneous servers (different hardware)

---

### Algorithm 3: Least Connections

Send each request to the server with the fewest active connections RIGHT NOW.

```
Current state:
┌──────────┬────────────────────┬───────┐
│  Server  │ Active Connections │ Pick? │
├──────────┼────────────────────┼───────┤
│ Server A │         12         │       │
│ Server B │          3         │  ✓    │ ← Least loaded!
│ Server C │          8         │       │
└──────────┴────────────────────┴───────┘

New request → goes to Server B (only 3 active connections)

After assignment:
┌──────────┬────────────────────┐
│ Server A │         12         │
│ Server B │          4         │  (was 3, now 4)
│ Server C │          8         │
└──────────┴────────────────────┘

Next request → STILL goes to Server B (it's still the least!)

After:
┌──────────┬────────────────────┐
│ Server A │         11         │  (one connection finished)
│ Server B │          5         │
│ Server C │          8         │
└──────────┴────────────────────┘

Next request → Server B again (still least at 5)
```

**Why it's better than Round Robin:**

```
Scenario: Some requests are fast (5ms), some are slow (5000ms)

ROUND ROBIN:
Request 1 (slow, 5s)   → Server A  (busy for 5 seconds!)
Request 2 (fast, 5ms)  → Server B  (done instantly)
Request 3 (fast, 5ms)  → Server C  (done instantly)
Request 4 (slow, 5s)   → Server A  (STILL BUSY from Request 1! Now has 2!)
Request 5 (slow, 5s)   → Server B  (fine, it's free)
Request 6 (fast, 5ms)  → Server C  (fine)

Server A is overloaded because Round Robin doesn't know it's still busy!

LEAST CONNECTIONS:
Request 1 (slow, 5s)   → Server A  [A:1, B:0, C:0]
Request 2 (fast, 5ms)  → Server B  [A:1, B:1, C:0] → instantly [A:1, B:0, C:0]
Request 3 (fast, 5ms)  → Server B  (B now has 0!) [A:1, B:0, C:0]
Request 4 (slow, 5s)   → Server B  [A:1, B:1, C:0] wait... → Server C! [A:1, B:0, C:0]
                                     (B finished already, C was also 0, either works)
```

**Pros:** Adapts to real-time load, handles varying request durations  
**Cons:** Requires tracking connection counts per server

**Best for:** Requests with varying processing times (APIs that mix fast reads with slow reports)

---

### Algorithm 4: Weighted Least Connections

Combines Weighted + Least Connections. Accounts for both server capacity AND current load.

```
Formula: score = active_connections / weight
Pick the server with the LOWEST score.

┌──────────┬────────┬─────────────────────┬───────────┬───────┐
│  Server  │ Weight │ Active Connections  │   Score   │ Pick? │
├──────────┼────────┼─────────────────────┼───────────┼───────┤
│ Server A │   5    │         10          │ 10/5 = 2.0│       │
│ Server B │   3    │          3          │  3/3 = 1.0│  ✓    │
│ Server C │   1    │          1          │  1/1 = 1.0│  ✓    │
└──────────┴────────┴─────────────────────┴───────────┴───────┘

Server A has more connections BUT also more capacity.
Score normalizes: "connections per unit of capacity"
B and C are tied at 1.0 — LB picks either (often the first).
```

---

### Algorithm 5: IP Hash

Hash the client's IP address to determine which server they go to. Same IP → always same server.

```
Hash function: server_index = hash(client_ip) % num_servers

Client 203.0.113.50:
  hash("203.0.113.50") = 928374261
  928374261 % 3 = 0   → Server A   (ALWAYS Server A!)

Client 198.51.100.23:
  hash("198.51.100.23") = 571629384
  571629384 % 3 = 1   → Server B   (ALWAYS Server B!)

Client 192.0.2.77:
  hash("192.0.2.77") = 103947552
  103947552 % 3 = 2   → Server C   (ALWAYS Server C!)

┌────────────────────┐                  ┌──────────┐
│ Client 203.0.113.50│──────────────────▶│ Server A │ (always!)
└────────────────────┘                  └──────────┘
┌────────────────────┐                  ┌──────────┐
│ Client 198.51.100.23│─────────────────▶│ Server B │ (always!)
└────────────────────┘                  └──────────┘
┌────────────────────┐                  ┌──────────┐
│ Client 192.0.2.77  │──────────────────▶│ Server C │ (always!)
└────────────────────┘                  └──────────┘
```

**Pros:** Built-in session persistence without cookies or state  
**Cons:** Uneven distribution if some IPs send way more traffic; breaks if server count changes

**Best for:** Stateful applications that can't use external session stores (legacy systems)

---

### Algorithm 6: Least Response Time

Send requests to the server with the fastest recent response time.

```
Server health metrics (measured continuously):
┌──────────┬─────────────────────┬────────────────────┬───────┐
│  Server  │ Avg Response Time   │ Active Connections │ Pick? │
├──────────┼─────────────────────┼────────────────────┼───────┤
│ Server A │       45ms          │         5          │  ✓    │
│ Server B │      120ms          │         3          │       │
│ Server C │       80ms          │         4          │       │
└──────────┴─────────────────────┴────────────────────┴───────┘

Server A is fastest even with more connections → gets the request!
```

**Best for:** When you want the fastest user experience (combines speed + load)

---

### Algorithm 7: Random

Pick a server at random. Surprisingly effective with many servers!

```
With 3 servers and 1000 requests:
- Server A: ~333 requests
- Server B: ~334 requests
- Server C: ~333 requests

With large numbers, random approaches round-robin's fairness.
No state needed, no coordination between LB instances.
```

**Best for:** Multiple LB instances that can't share state

---

### Algorithm Comparison Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│            ALGORITHM COMPARISON MATRIX                                │
├──────────────────────┬────────┬───────────┬──────────┬──────────────┤
│     Algorithm        │ Speed  │ Fairness  │ State    │ Best For     │
├──────────────────────┼────────┼───────────┼──────────┼──────────────┤
│ Round Robin          │ ★★★★★ │ ★★★☆☆    │ Minimal  │ Identical    │
│                      │        │           │          │ servers      │
├──────────────────────┼────────┼───────────┼──────────┼──────────────┤
│ Weighted Round Robin │ ★★★★★ │ ★★★★☆    │ Minimal  │ Mixed        │
│                      │        │           │          │ hardware     │
├──────────────────────┼────────┼───────────┼──────────┼──────────────┤
│ Least Connections    │ ★★★★☆ │ ★★★★★    │ Per-conn │ Variable     │
│                      │        │           │ tracking │ request time │
├──────────────────────┼────────┼───────────┼──────────┼──────────────┤
│ IP Hash              │ ★★★★★ │ ★★☆☆☆    │ None     │ Session      │
│                      │        │           │          │ stickiness   │
├──────────────────────┼────────┼───────────┼──────────┼──────────────┤
│ Least Response Time  │ ★★★☆☆ │ ★★★★★    │ Metrics  │ Latency-     │
│                      │        │           │ tracking │ sensitive    │
├──────────────────────┼────────┼───────────┼──────────┼──────────────┤
│ Random               │ ★★★★★ │ ★★★☆☆    │ None     │ Distributed  │
│                      │        │           │          │ LB clusters  │
└──────────────────────┴────────┴───────────┴──────────┴──────────────┘
```

---

## How It Works Internally

### Consistent Hashing (Used by IP Hash and Modern LBs)

Regular hashing breaks when servers are added/removed:

```
PROBLEM with simple hash:
hash(ip) % 3 servers:  Client A → Server 0
hash(ip) % 4 servers:  Client A → Server 2  ← DIFFERENT SERVER!

When you add server 4, MOST clients get remapped! Bad!

SOLUTION: Consistent Hashing (hash ring)

Imagine a circular number line (0 to 2^32):

         Server A (hash=1000)
              │
              ▼
    ──────────●──────────────────
   │                              │
   │          Hash Ring           │
   │    (0 ──────── 2^32)        │
   │                              │
    ──────────────────●──────────
                      │
                      ▼
                Server B (hash=3000)

Client X (hash=1500) → walks clockwise → hits Server B (3000)
Client Y (hash=800)  → walks clockwise → hits Server A (1000)

Adding Server C (hash=2000):
- Only clients between 1000-2000 get remapped (from B to C)
- Everyone else stays on their current server!
- Minimal disruption!
```

---

## Code Examples

### Python — All Major Algorithms Implemented

```python
# lb_algorithms.py — Implementations of common load balancing algorithms
import hashlib
import random
import time
from collections import defaultdict

class Server:
    def __init__(self, name: str, weight: int = 1):
        self.name = name
        self.weight = weight
        self.active_connections = 0
        self.response_times = []  # Recent response times
    
    @property
    def avg_response_time(self) -> float:
        if not self.response_times:
            return 0
        return sum(self.response_times[-10:]) / len(self.response_times[-10:])

# --- Round Robin ---
class RoundRobinLB:
    def __init__(self, servers: list):
        self.servers = servers
        self.index = 0
    
    def next_server(self) -> Server:
        server = self.servers[self.index % len(self.servers)]
        self.index += 1
        return server

# --- Weighted Round Robin ---
class WeightedRoundRobinLB:
    def __init__(self, servers: list):
        # Expand servers by weight: A(w=3) → [A, A, A]
        self.pool = []
        for server in servers:
            self.pool.extend([server] * server.weight)
        self.index = 0
    
    def next_server(self) -> Server:
        server = self.pool[self.index % len(self.pool)]
        self.index += 1
        return server

# --- Least Connections ---
class LeastConnectionsLB:
    def __init__(self, servers: list):
        self.servers = servers
    
    def next_server(self) -> Server:
        return min(self.servers, key=lambda s: s.active_connections)

# --- IP Hash ---
class IPHashLB:
    def __init__(self, servers: list):
        self.servers = servers
    
    def next_server(self, client_ip: str) -> Server:
        hash_val = int(hashlib.md5(client_ip.encode()).hexdigest(), 16)
        index = hash_val % len(self.servers)
        return self.servers[index]

# --- Demo ---
servers = [Server("A", weight=3), Server("B", weight=2), Server("C", weight=1)]
lb = LeastConnectionsLB(servers)
servers[0].active_connections = 5
servers[1].active_connections = 2
servers[2].active_connections = 8
print(f"Next server: {lb.next_server().name}")  # → B (least connections)
```

### Java — Least Connections with Thread Safety

```java
// LeastConnectionsLoadBalancer.java — Thread-safe implementation
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.CopyOnWriteArrayList;

public class LeastConnectionsLoadBalancer {
    
    static class Server {
        final String address;
        final AtomicInteger activeConnections = new AtomicInteger(0);
        
        Server(String address) { this.address = address; }
        
        int getConnections() { return activeConnections.get(); }
        void connect()       { activeConnections.incrementAndGet(); }
        void disconnect()    { activeConnections.decrementAndGet(); }
    }
    
    private final CopyOnWriteArrayList<Server> servers;
    
    public LeastConnectionsLoadBalancer(List<Server> servers) {
        this.servers = new CopyOnWriteArrayList<>(servers);
    }
    
    // Thread-safe: find server with minimum active connections
    public Server chooseServer() {
        return servers.stream()
            .min(Comparator.comparingInt(Server::getConnections))
            .orElseThrow(() -> new RuntimeException("No servers available"));
    }
    
    // Simulate handling a request
    public void handleRequest(String requestId) {
        Server chosen = chooseServer();
        chosen.connect();
        System.out.printf("Request %s → %s (connections: %d)%n",
            requestId, chosen.address, chosen.getConnections());
        
        // Simulate processing... (in real code, this would be async)
        // After response:
        chosen.disconnect();
    }
    
    public static void main(String[] args) {
        var lb = new LeastConnectionsLoadBalancer(List.of(
            new Server("10.0.1.1:8080"),
            new Server("10.0.1.2:8080"),
            new Server("10.0.1.3:8080")
        ));
        
        // Simulate 10 concurrent requests
        for (int i = 1; i <= 10; i++) {
            lb.handleRequest("REQ-" + i);
        }
    }
}
```

---

## Infrastructure Examples

### Nginx — Configuring Different Algorithms

```nginx
# Round Robin (default — no directive needed)
upstream backend_rr {
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
    server 10.0.1.3:8080;
}

# Weighted Round Robin
upstream backend_weighted {
    server 10.0.1.1:8080 weight=5;   # Gets 50% of traffic
    server 10.0.1.2:8080 weight=3;   # Gets 30% of traffic
    server 10.0.1.3:8080 weight=2;   # Gets 20% of traffic
}

# Least Connections
upstream backend_leastconn {
    least_conn;
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
    server 10.0.1.3:8080;
}

# IP Hash (session persistence)
upstream backend_iphash {
    ip_hash;
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
    server 10.0.1.3:8080;
}
```

### HAProxy — Algorithm Configuration

```haproxy
# /etc/haproxy/haproxy.cfg

backend web_servers
    # Round Robin
    balance roundrobin
    server web1 10.0.1.1:8080 check weight 1
    server web2 10.0.1.2:8080 check weight 1

backend api_servers
    # Least Connections
    balance leastconn
    server api1 10.0.2.1:8080 check
    server api2 10.0.2.2:8080 check
    server api3 10.0.2.3:8080 check

backend session_servers
    # Source IP hash (sticky sessions)
    balance source
    hash-type consistent    # Use consistent hashing
    server app1 10.0.3.1:8080 check
    server app2 10.0.3.2:8080 check
```

---

## Real-World Example

### How Cloudflare Chooses Algorithms

```
Cloudflare's traffic routing (simplified):

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  EXTERNAL (User → Cloudflare):                                  │
│  Algorithm: Geo-based + Least connections                        │
│  "Route to nearest data center, then to least loaded server"    │
│                                                                  │
│  INTERNAL (Between Cloudflare services):                        │
│  Algorithm: Weighted Least Connections + Health metrics          │
│  "Account for server capacity AND current load AND response     │
│   time — pick the server that'll respond fastest"               │
│                                                                  │
│  FOR WEBSOCKET (persistent connections):                        │
│  Algorithm: IP Hash                                             │
│  "Same client must reconnect to same server for state"          │
│                                                                  │
│  FOR API GATEWAY:                                               │
│  Algorithm: Least Response Time                                  │
│  "Pick the server that responded fastest in the last 10s"       │
│                                                                  │
│  They DON'T use one algorithm for everything.                   │
│  Different workloads need different strategies!                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using Round Robin with varying request times | Slow requests pile up on unlucky servers | Use Least Connections |
| Using IP Hash when servers change frequently | Adding/removing a server remaps most clients | Use Consistent Hashing |
| Not setting weights for different hardware | Weak servers get same traffic as strong ones → bottleneck | Use Weighted algorithms |
| Using Least Connections without health checks | Sends traffic to servers that are slow because they're failing | Combine with health checks (Ch 6.4) |
| Choosing algorithm and never revisiting | Traffic patterns change over time | Monitor and adjust quarterly |

---

## When to Use / When NOT to Use

### Decision Guide

```
What's your situation?
│
├── All servers are identical hardware?
│   ├── Requests take roughly same time? → Round Robin ✓
│   └── Requests vary wildly in duration? → Least Connections ✓
│
├── Servers have different capacities?
│   └── → Weighted Round Robin or Weighted Least Connections ✓
│
├── Need same user to hit same server?
│   └── → IP Hash (simple) or Sticky Sessions (Chapter 6.5) ✓
│
├── Latency is the #1 priority?
│   └── → Least Response Time ✓
│
└── Multiple LB instances that can't share state?
    └── → Random or IP Hash ✓
```

---

## Key Takeaways

1. **Round Robin** is the default and works well for identical servers with uniform request times — start here unless you have a reason not to.

2. **Least Connections** is the most adaptive algorithm — it naturally handles varying request durations by always choosing the least-loaded server.

3. **Weighted algorithms** exist for heterogeneous infrastructure — assign weights proportional to server capacity (CPU cores, RAM).

4. **IP Hash** provides session stickiness without cookies — same client always hits the same server, but breaks when servers are added/removed.

5. **No single algorithm is "best"** — different workloads need different strategies. Many production systems use different algorithms for different backend pools.

6. **Consistent hashing** solves the IP Hash remapping problem — only a fraction of clients are remapped when servers change.

7. **Always combine with health checks** — no algorithm helps if you're routing to a dead server (covered in Chapter 6.4).

---

## What's Next?

Algorithms decide WHERE to send traffic, but how does the load balancer know if a server is even alive? In **Chapter 6.4: Health Checks**, we'll explore how load balancers continuously monitor server health and automatically remove failing servers from the pool.
