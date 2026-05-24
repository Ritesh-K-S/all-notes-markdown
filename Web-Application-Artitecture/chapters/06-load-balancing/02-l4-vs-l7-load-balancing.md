# Layer 4 vs Layer 7 Load Balancing

> **What you'll learn**: The two fundamentally different types of load balancers — one that only looks at network addresses (Layer 4) and one that reads your actual HTTP requests (Layer 7) — their trade-offs in speed, intelligence, and cost, and when to pick each.

---

## Real-Life Analogy — Post Office vs Personal Assistant

**Layer 4 Load Balancer = Post Office Sorter**

Imagine a post office sorting machine. It looks at the **address on the envelope** (IP + port) and routes it to the right mailbox. It NEVER opens the envelope to read what's inside. It's incredibly fast because it doesn't need to understand the contents — just the destination.

**Layer 7 Load Balancer = Personal Assistant**

Now imagine a personal assistant who opens every letter, reads the content, and decides what to do:
- "This is a billing question → route to the Finance team"
- "This is a technical problem → route to the Engineering team"
- "This is spam → throw it away"

The assistant is smarter but slower — they have to read every letter.

```
LAYER 4 (Post Office):                LAYER 7 (Personal Assistant):
┌──────────────┐                      ┌──────────────┐
│ Sees ONLY:   │                      │ Sees ALL:    │
│ • Source IP  │                      │ • URL path   │
│ • Dest IP    │                      │ • Headers    │
│ • Source Port│                      │ • Cookies    │
│ • Dest Port  │                      │ • Body       │
│ • Protocol   │                      │ • Method     │
│              │                      │ • Host name  │
│ Does NOT see:│                      │              │
│ • URL        │                      │ Can make     │
│ • Headers    │                      │ smart routing│
│ • Body       │                      │ decisions!   │
└──────────────┘                      └──────────────┘
```

---

## Core Concept Explained Step-by-Step

### The OSI Model — Where These Layers Live

```
┌─────────────────────────────────────────────────────────────────────┐
│                         OSI NETWORK MODEL                            │
│                                                                     │
│  Layer 7: APPLICATION   ← HTTP, HTTPS, WebSocket, gRPC             │
│           "What is the user actually asking for?"                    │
│           URL: /api/users, Header: Authorization: Bearer xyz        │
│                                                                     │
│  Layer 6: Presentation  (encoding, encryption)                      │
│  Layer 5: Session       (session management)                        │
│                                                                     │
│  Layer 4: TRANSPORT     ← TCP, UDP                                  │
│           "Where is this packet going?"                             │
│           Source port: 52431, Dest port: 443                        │
│                                                                     │
│  Layer 3: Network       (IP addressing, routing)                    │
│  Layer 2: Data Link     (MAC addresses, switches)                   │
│  Layer 1: Physical      (cables, electrical signals)                │
│                                                                     │
│  ────────────────────────────────────────────────────               │
│  L4 Load Balancer operates HERE ───────────── Layer 4               │
│  L7 Load Balancer operates HERE ───────────── Layer 7               │
└─────────────────────────────────────────────────────────────────────┘
```

### Layer 4 Load Balancing — Fast but Blind

```
HOW L4 WORKS:

Client sends a TCP packet:
┌─────────────────────────────────────────────────────────┐
│ TCP Packet Header:                                       │
│   Source IP:   203.0.113.50                             │
│   Dest IP:     52.23.186.100 (LB's IP)                 │
│   Source Port: 52431                                     │
│   Dest Port:   443                                       │
│   Protocol:    TCP                                       │
├─────────────────────────────────────────────────────────┤
│ Payload: [encrypted blob — LB cannot read this!]        │
└─────────────────────────────────────────────────────────┘

L4 Load Balancer decision:
"I see a packet going to port 443. I'll forward it to Server 2."
(It CANNOT see: URL, HTTP method, headers, cookies, or body)

┌────────┐         ┌─────────┐         ┌──────────┐
│ Client │──TCP───▶│  L4 LB  │──TCP───▶│ Server 2 │
│        │         │         │         │          │
│        │         │ Sees:   │         │          │
│        │         │ IP+Port │         │          │
│        │         │ ONLY    │         │          │
└────────┘         └─────────┘         └──────────┘
```

**What L4 CAN do:**
- Route based on source/destination IP
- Route based on port number
- Very fast (no need to parse HTTP)
- Handle ANY protocol (not just HTTP): TCP, UDP, SMTP, FTP, database connections
- NAT (Network Address Translation)

**What L4 CANNOT do:**
- Route `/api/users` to Service A and `/api/orders` to Service B
- Read cookies or headers
- Modify HTTP requests/responses
- Terminate SSL (it doesn't understand SSL — just forwards raw bytes)
- Make content-based routing decisions

### Layer 7 Load Balancing — Smart but Slower

```
HOW L7 WORKS:

Client sends HTTP request (after TCP+TLS handshake):
┌─────────────────────────────────────────────────────────┐
│ HTTP Request (fully visible to L7 LB):                   │
│                                                         │
│   GET /api/users?page=1 HTTP/1.1                        │
│   Host: api.myapp.com                                   │
│   Authorization: Bearer eyJhbGc...                      │
│   Cookie: session_id=abc123                             │
│   Content-Type: application/json                        │
│   X-Request-ID: req-456                                 │
│                                                         │
└─────────────────────────────────────────────────────────┘

L7 Load Balancer decision:
"I see this is a GET to /api/users. I'll route it to the User Service."
"The cookie says premium_user=true, route to premium servers."
"The header says mobile-app, route to mobile-optimized servers."

┌────────┐         ┌─────────┐         ┌──────────────┐
│ Client │──HTTP──▶│  L7 LB  │────────▶│ User Service │
│        │         │         │         └──────────────┘
│        │         │ Sees:   │         ┌──────────────┐
│        │         │ FULL    │────────▶│Order Service │
│        │         │ HTTP    │         └──────────────┘
│        │         │ Request │         ┌──────────────┐
│        │         │         │────────▶│ Static Files │
└────────┘         └─────────┘         └──────────────┘
```

**What L7 CAN do (that L4 cannot):**
- Route based on URL path (`/api/*` → API servers, `/static/*` → CDN)
- Route based on HTTP headers (mobile vs desktop, premium vs free)
- Route based on cookies (A/B testing, canary releases)
- Terminate SSL/TLS (decrypt → inspect → re-encrypt)
- Modify requests (add headers, rewrite URLs)
- Rate limit specific endpoints
- Serve cached responses directly (without hitting backend)
- Content-based health checks (check HTTP 200 vs just TCP connection)

---

## How It Works Internally

### L4 Internal Mechanics

```
L4 PACKET ROUTING METHODS:

METHOD 1: NAT (Network Address Translation)
┌────────┐  Src: Client    ┌────────┐  Src: LB         ┌────────┐
│ Client │  Dst: LB     ──▶│  L4 LB │  Dst: Server  ──▶│ Server │
│        │                  │        │  (rewrites IP)   │        │
└────────┘                  └────────┘                  └────────┘
LB changes destination IP from its own to the server's IP.
Response goes back through LB (which un-NATs it).

METHOD 2: DSR (Direct Server Return) — even faster!
┌────────┐                  ┌────────┐                  ┌────────┐
│ Client │────────────────▶│  L4 LB │─────────────────▶│ Server │
│        │                  │  (fwd) │                  │        │
│        │◀──────────────────────────────────────────────│        │
└────────┘                  └────────┘                  └────────┘
                         (response goes DIRECTLY back to client!)
                         (LB never sees the response — super fast!)

DSR: Server responds directly to client, bypassing LB on return path.
Used in high-performance scenarios (gaming, streaming).
```

### L7 Internal Mechanics

```
L7 CONNECTION HANDLING:

┌────────┐                  ┌────────────┐              ┌────────┐
│ Client │                  │   L7 LB    │              │ Server │
│        │                  │            │              │        │
│ 1. TCP │──── SYN ────────▶│            │              │        │
│    handshake               │            │              │        │
│        │◀─── SYN-ACK ─────│            │              │        │
│        │──── ACK ─────────▶│            │              │        │
│        │                  │            │              │        │
│ 2. TLS │──── ClientHello ─▶│ (Decrypt!) │              │        │
│    handshake               │            │              │        │
│        │◀─── Certificate ──│            │              │        │
│        │──── Finished ────▶│            │              │        │
│        │                  │            │              │        │
│ 3. HTTP│──── GET /api ────▶│ (Read URL) │              │        │
│    request                 │ (Decide)  │              │        │
│        │                  │            │──── HTTP ────▶│        │
│        │                  │            │◀─── Response──│        │
│        │◀─── Response ─────│            │              │        │
│        │                  │            │              │        │
└────────┘                  └────────────┘              └────────┘

Two separate connections:
- Client ↔ LB (encrypted)
- LB ↔ Server (can be unencrypted in private network)

L7 LB holds BOTH connections simultaneously.
This is called "connection termination" or "proxy mode."
```

### Performance Comparison

```
┌─────────────────────────────────────────────────────────────────────┐
│                  PERFORMANCE: L4 vs L7                               │
│                                                                     │
│  Metric              │    Layer 4        │     Layer 7               │
│  ────────────────────┼───────────────────┼────────────────────────   │
│  Throughput          │  Very High        │  High (but less than L4) │
│                      │  (millions pps)   │  (hundreds of k rps)     │
│                      │                   │                          │
│  Latency added       │  < 1 ms           │  1-5 ms                  │
│                      │                   │  (parse + decrypt)       │
│                      │                   │                          │
│  CPU usage           │  Very Low         │  Higher                  │
│                      │  (just routing)   │  (SSL + parsing)         │
│                      │                   │                          │
│  Memory usage        │  Low              │  Higher                  │
│                      │  (connection      │  (HTTP buffers,          │
│                      │   tracking)       │   SSL sessions)          │
│                      │                   │                          │
│  Connection handling │  Millions         │  Tens of thousands       │
│                      │  concurrent       │  concurrent              │
│                      │                   │                          │
│  Routing intelligence│  None (IP+port)   │  Full (URL, headers,     │
│                      │                   │   cookies, body)         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — L7 Routing Logic (URL-Based)

```python
# l7_router.py — Demonstrates L7 load balancer routing decisions
from flask import Flask, request, redirect
import httpx

app = Flask(__name__)

# Different backend pools for different paths
ROUTES = {
    "/api/users":    ["http://user-svc-1:8080", "http://user-svc-2:8080"],
    "/api/orders":   ["http://order-svc-1:8080", "http://order-svc-2:8080"],
    "/api/products": ["http://product-svc-1:8080", "http://product-svc-2:8080"],
    "/static":       ["http://cdn-1:80", "http://cdn-2:80"],
}

def find_backend(path: str) -> list:
    """L7 routing: match URL path to backend pool."""
    for prefix, servers in ROUTES.items():
        if path.startswith(prefix):
            return servers
    return ["http://default-svc:8080"]  # fallback

@app.route('/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
def route_request(path):
    """L7 LB: reads the HTTP request and routes based on content."""
    full_path = f"/{path}"
    backends = find_backend(full_path)
    
    # Route based on header (e.g., canary routing)
    if request.headers.get("X-Canary") == "true":
        target = backends[-1]  # Last server = canary
    else:
        # Simple round-robin among healthy backends
        target = backends[hash(request.remote_addr) % len(backends)]
    
    # Forward the full request to chosen backend
    resp = httpx.request(
        method=request.method,
        url=f"{target}{full_path}",
        headers=dict(request.headers),
        content=request.get_data(),
    )
    return (resp.content, resp.status_code, dict(resp.headers))
```

### Java — L4 vs L7 Connection Handling

```java
// L4LoadBalancer.java — TCP-level load balancing (no HTTP awareness)
import java.net.*;
import java.io.*;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

public class L4LoadBalancer {
    private final List<String> backends;  // ["10.0.1.1:8080", "10.0.1.2:8080"]
    private final AtomicInteger counter = new AtomicInteger(0);
    
    public L4LoadBalancer(List<String> backends) {
        this.backends = backends;
    }
    
    // L4: Just forward raw TCP bytes — don't look inside!
    public void handleConnection(Socket clientSocket) throws IOException {
        // Pick a backend (round-robin)
        String backend = backends.get(counter.getAndIncrement() % backends.size());
        String[] parts = backend.split(":");
        
        // Open connection to backend
        Socket backendSocket = new Socket(parts[0], Integer.parseInt(parts[1]));
        
        // Forward bytes both ways (bidirectional pipe)
        // L4 never reads/modifies the content — just pipes bytes through
        Thread clientToBackend = new Thread(() -> pipe(clientSocket, backendSocket));
        Thread backendToClient = new Thread(() -> pipe(backendSocket, clientSocket));
        
        clientToBackend.start();
        backendToClient.start();
    }
    
    private void pipe(Socket from, Socket to) {
        try (InputStream in = from.getInputStream();
             OutputStream out = to.getOutputStream()) {
            byte[] buffer = new byte[4096];
            int bytesRead;
            while ((bytesRead = in.read(buffer)) != -1) {
                out.write(buffer, 0, bytesRead);  // Raw bytes, no parsing!
                out.flush();
            }
        } catch (IOException e) { /* connection closed */ }
    }
}
```

---

## Infrastructure Examples

### AWS: NLB (Layer 4) vs ALB (Layer 7)

```
AWS Network Load Balancer (NLB) — Layer 4:
┌─────────────────────────────────────────┐
│ • Handles millions of requests/sec      │
│ • Ultra-low latency (~100 microseconds) │
│ • Supports TCP, UDP, TLS               │
│ • Static IP addresses                  │
│ • Cannot read HTTP content             │
│ • Use for: databases, gaming, IoT, gRPC│
│ • Cost: ~$0.006/hour + per LCU         │
└─────────────────────────────────────────┘

AWS Application Load Balancer (ALB) — Layer 7:
┌─────────────────────────────────────────┐
│ • Content-based routing (URL, headers)  │
│ • SSL termination                       │
│ • WebSocket support                     │
│ • HTTP/2 support                        │
│ • Path-based routing (/api, /admin)     │
│ • Host-based routing (a.com, b.com)     │
│ • Use for: web apps, APIs, microservices│
│ • Cost: ~$0.0225/hour + per LCU         │
└─────────────────────────────────────────┘
```

### Nginx: L7 Content-Based Routing

```nginx
# L7 routing: different paths → different backends
http {
    upstream user_service {
        server 10.0.1.1:8080;
        server 10.0.1.2:8080;
    }
    
    upstream order_service {
        server 10.0.2.1:8080;
        server 10.0.2.2:8080;
    }
    
    upstream static_files {
        server 10.0.3.1:80;
    }

    server {
        listen 443 ssl;
        server_name api.myapp.com;
        
        # SSL termination happens HERE (at L7 LB)
        ssl_certificate     /etc/ssl/cert.pem;
        ssl_certificate_key /etc/ssl/key.pem;

        # Route based on URL path (this is L7 intelligence!)
        location /api/users {
            proxy_pass http://user_service;
        }
        
        location /api/orders {
            proxy_pass http://order_service;
        }
        
        location /static {
            proxy_pass http://static_files;
            # Add caching headers
            expires 30d;
        }
        
        # Route based on header (A/B testing)
        location /api/checkout {
            if ($http_x_experiment = "new_checkout") {
                proxy_pass http://checkout_v2;
            }
            proxy_pass http://checkout_v1;
        }
    }
}
```

---

## Real-World Example

### How Google Handles Both Layers

```
Google's Load Balancing Stack:

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  LAYER 4 (Maglev — Google's custom L4 LB):                     │
│  ├── Custom hardware/software LB                                 │
│  ├── Handles 10+ million packets per second per machine          │
│  ├── Uses consistent hashing (no state between nodes)            │
│  ├── ECMP (Equal-Cost Multi-Path) for LB redundancy             │
│  └── Routes raw TCP/UDP to the right backend cluster            │
│                                                                  │
│  LAYER 7 (GFE — Google Front End):                              │
│  ├── Terminates TLS connections                                  │
│  ├── Reads HTTP requests                                         │
│  ├── Routes based on URL path, host, etc.                       │
│  ├── Serves cached content directly (no backend hit)             │
│  ├── Enforces rate limits and DDoS protection                   │
│  └── Adds security headers                                      │
│                                                                  │
│  TRAFFIC FLOW:                                                   │
│  Client → Maglev (L4) → GFE (L7) → Backend Services            │
│                                                                  │
│  Why BOTH?                                                       │
│  - L4 for raw speed and DDoS absorption                         │
│  - L7 for intelligent routing and SSL                           │
└──────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using L7 for database traffic | L7 can't understand database protocols. Adds unnecessary latency | Use L4 (NLB) for databases, Redis, Kafka |
| Using L4 when you need URL routing | L4 can't see URLs — all traffic goes to same pool | Use L7 (ALB/Nginx) for HTTP-based routing |
| Forgetting L7 adds latency | SSL termination + HTTP parsing = 1-5ms extra | Acceptable for web apps; use L4 if microseconds matter |
| Not enabling keep-alive on L7 | New TCP+TLS handshake per request = terrible performance | Enable HTTP keep-alive (connections reused) |
| Running L7 for WebSocket without config | WebSocket needs special upgrade handling | Configure `proxy_http_version 1.1` + upgrade headers in Nginx |

---

## When to Use / When NOT to Use

### Use Layer 4 When:

| Scenario | Why L4 |
|----------|--------|
| Database connections (PostgreSQL, MySQL) | L4 handles any TCP protocol |
| Raw TCP/UDP services (gaming, VoIP) | L7 doesn't understand game protocols |
| Extreme performance requirements | Lowest possible latency |
| TLS passthrough needed | Client talks directly to server's TLS |
| Non-HTTP protocols (SMTP, FTP, custom) | L7 is HTTP-specific |

### Use Layer 7 When:

| Scenario | Why L7 |
|----------|--------|
| Microservices with path-based routing | `/users` → Service A, `/orders` → Service B |
| Canary deployments / A/B testing | Route by header or cookie |
| SSL termination at the edge | Offload encryption from application servers |
| Web applications and APIs | Need HTTP awareness |
| Rate limiting per endpoint | Different limits for different URLs |
| WebSocket with fallback | Can handle upgrade + regular HTTP |

### Decision Flowchart:

```
Is it HTTP/HTTPS traffic?
├── YES → Is routing intelligence needed? (paths, headers, cookies)
│         ├── YES → Layer 7 ✓
│         └── NO (just round-robin) → Layer 7 still fine, or L4 for speed
└── NO (TCP/UDP, database, custom protocol)
          └── Layer 4 ✓
```

---

## Key Takeaways

1. **Layer 4** operates at the transport level (TCP/UDP) — it sees only IPs and ports, making it blazingly fast but unable to make content-based routing decisions.

2. **Layer 7** operates at the application level (HTTP) — it can read URLs, headers, cookies, and make intelligent routing decisions, but adds latency.

3. **L4 is for speed and protocol-agnostic load balancing** — use it for databases, gaming, gRPC, or when every microsecond counts.

4. **L7 is for intelligent HTTP routing** — use it for web apps, microservices, canary deployments, and anything that needs content-aware decisions.

5. **Most production systems use BOTH** — L4 in front for raw traffic distribution, L7 behind it for application-level routing (Google's Maglev + GFE pattern).

6. **AWS NLB = Layer 4, AWS ALB = Layer 7** — pick based on your needs, not just what's popular.

7. **L7 load balancers are also reverse proxies** — they terminate connections, enabling SSL offload, request modification, and caching.

---

## What's Next?

You now know the two types of load balancers. But HOW does each one choose which server to send a request to? In **Chapter 6.3: Load Balancing Algorithms**, we'll explore Round Robin, Least Connections, IP Hash, Weighted algorithms, and more — the actual decision-making logic inside every load balancer.
