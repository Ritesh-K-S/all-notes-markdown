# Forward Proxy vs Reverse Proxy

> **What you'll learn**: The difference between forward and reverse proxies — what they are, how they work internally, and when to use each one in real-world web architecture.

---

## Real-Life Analogy

Imagine you live in a gated apartment complex:

**Forward Proxy = The Security Guard at YOUR Gate (Outgoing)**

You want to order food from 10 different restaurants. Instead of giving each restaurant your home address, you tell the security guard: "Order for me." The restaurants only see the guard's address — they never know who you are or where you live exactly.

**Reverse Proxy = The Receptionist at a Company (Incoming)**

You call a company's main number. You don't know which employee will answer — the receptionist routes your call to the right department. You never speak directly to the internal team; the receptionist handles everything.

```
FORWARD PROXY (protects the CLIENT):

  You ──▶ [Forward Proxy] ──▶ Internet ──▶ Website
  (Client)   (hides you)                    (Server)

REVERSE PROXY (protects the SERVER):

  Internet ──▶ [Reverse Proxy] ──▶ Server 1
  (Client)      (hides servers)  ──▶ Server 2
                                 ──▶ Server 3
```

---

## Core Concept Explained Step-by-Step

### Step 1: What is a Proxy?

A **proxy** is simply a middleman — something that sits between two parties and acts on behalf of one of them.

In networking, a proxy sits between a client and a server and forwards requests/responses between them.

```
Without Proxy:
  Client ────────────────────────▶ Server

With Proxy:
  Client ──▶ [PROXY] ──▶ Server
```

### Step 2: Forward Proxy (Client-Side Proxy)

A **forward proxy** sits in front of **clients** and forwards their requests to the internet.

The server never sees the real client — it only sees the proxy.

```
┌─────────────────────────────────────────────────────────┐
│                     FORWARD PROXY                        │
│                                                         │
│  Client A ─┐                                           │
│  Client B ─┼──▶ [Forward Proxy] ──▶ Internet ──▶ Server│
│  Client C ─┘    (1 IP visible)                         │
│                                                         │
│  Server sees: Proxy's IP (not A, B, or C's IP)         │
└─────────────────────────────────────────────────────────┘
```

**What a Forward Proxy Does:**
- **Hides client identity** — Server only sees the proxy's IP
- **Content filtering** — Block access to certain websites (used in offices/schools)
- **Caching** — Cache frequently accessed content to speed things up
- **Bypassing restrictions** — Access geo-blocked content via a proxy in another country
- **Logging & monitoring** — Track what employees browse

**Real Examples:**
- Corporate firewalls (Squid Proxy)
- VPNs (they act as forward proxies)
- School/library internet filters
- Tor network (multi-hop forward proxy)

### Step 3: Reverse Proxy (Server-Side Proxy)

A **reverse proxy** sits in front of **servers** and forwards client requests to the appropriate backend server.

The client never sees the real server — it only sees the proxy.

```
┌──────────────────────────────────────────────────────────┐
│                     REVERSE PROXY                         │
│                                                          │
│                                    ┌──▶ Server 1 (App)   │
│  Internet ──▶ [Reverse Proxy] ────┼──▶ Server 2 (App)   │
│  (Clients)    (single entry)      └──▶ Server 3 (App)   │
│                                                          │
│  Client sees: Proxy's IP (not Server 1, 2, or 3's IP)   │
└──────────────────────────────────────────────────────────┘
```

**What a Reverse Proxy Does:**
- **Hides server identity** — Clients only see the proxy's IP
- **Load balancing** — Distribute requests across multiple servers
- **SSL termination** — Handle HTTPS encryption/decryption
- **Caching** — Cache responses to reduce server load
- **Compression** — Compress responses before sending to client
- **Security** — Protect against DDoS, hide server architecture
- **Rate limiting** — Throttle abusive clients

**Real Examples:**
- Nginx as a reverse proxy
- Cloudflare (sits in front of your servers)
- AWS ALB (Application Load Balancer)
- CDNs (act as reverse proxies for static content)

### Step 4: Key Difference — WHO is Being Protected?

| Aspect | Forward Proxy | Reverse Proxy |
|--------|--------------|---------------|
| **Sits in front of** | Clients | Servers |
| **Protects** | Client identity | Server identity |
| **Client knows about it?** | Yes (configured by client) | No (transparent to client) |
| **Server knows about it?** | No (sees proxy IP) | Yes (configured by server admin) |
| **Direction** | Client → Internet | Internet → Server |
| **Configured by** | Client/Network admin | Server/Infrastructure admin |
| **Main purpose** | Privacy, filtering, caching | Security, load balancing, SSL |

---

## How It Works Internally

### Forward Proxy — Internal Flow

```
Step-by-step: Client requests example.com through a forward proxy

1. Client ──[HTTP CONNECT example.com:443]──▶ Forward Proxy
2. Forward Proxy checks ACL (Access Control List)
   ├── Allowed? ──▶ Continue
   └── Blocked? ──▶ Return 403 Forbidden
3. Forward Proxy ──[DNS lookup]──▶ Resolves example.com
4. Forward Proxy ──[TCP connection]──▶ example.com:443
5. Forward Proxy ──[Forwards request]──▶ example.com
   (replaces client IP with proxy IP in headers)
6. example.com ──[Response]──▶ Forward Proxy
7. Forward Proxy ──[Response]──▶ Client
   (optionally caches the response)
```

**Headers modified by Forward Proxy:**
```
Original request from client:
  GET /page HTTP/1.1
  Host: example.com

What the server sees:
  GET /page HTTP/1.1
  Host: example.com
  X-Forwarded-For: <client-IP>    ← (optional, proxy may add or hide)
  Via: 1.1 proxy.company.com       ← (optional)
```

### Reverse Proxy — Internal Flow

```
Step-by-step: Client requests api.myapp.com (reverse proxy)

1. Client ──[DNS resolves api.myapp.com]──▶ Gets Reverse Proxy IP
2. Client ──[HTTPS request]──▶ Reverse Proxy (port 443)
3. Reverse Proxy:
   ├── Terminates SSL (decrypts HTTPS → HTTP)
   ├── Reads request path/headers
   ├── Applies routing rules:
   │   ├── /api/* ──▶ Backend Server Pool
   │   ├── /static/* ──▶ CDN/File Server
   │   └── /ws/* ──▶ WebSocket Server
   ├── Selects backend server (load balancing algorithm)
   └── Forwards request to chosen backend
4. Reverse Proxy ──[HTTP request]──▶ Backend Server (port 8080)
5. Backend Server ──[HTTP response]──▶ Reverse Proxy
6. Reverse Proxy:
   ├── Compresses response (gzip/brotli)
   ├── Adds security headers (HSTS, CSP, etc.)
   ├── Caches response (if cacheable)
   └── Encrypts response (HTTP → HTTPS)
7. Reverse Proxy ──[HTTPS response]──▶ Client
```

**Headers added by Reverse Proxy:**
```
What the backend server receives:
  GET /api/users HTTP/1.1
  Host: api.myapp.com
  X-Forwarded-For: 203.0.113.50       ← Real client IP
  X-Forwarded-Proto: https            ← Original protocol
  X-Real-IP: 203.0.113.50             ← Real client IP
  X-Request-ID: abc-123-def           ← Tracing ID
```

---

## Code Examples

### Python — Simple Forward Proxy

```python
# forward_proxy.py — A minimal forward proxy using Python
# This handles HTTP CONNECT method for HTTPS tunneling

import socket
import threading

def handle_client(client_socket):
    """Handle a single client connection through the forward proxy."""
    # Read the client's request
    request = client_socket.recv(4096).decode('utf-8')
    
    # Parse the first line: "CONNECT example.com:443 HTTP/1.1"
    first_line = request.split('\n')[0]
    method = first_line.split(' ')[0]
    
    if method == 'CONNECT':
        # HTTPS tunneling — just forward raw bytes
        host_port = first_line.split(' ')[1]
        host, port = host_port.split(':')
        port = int(port)
        
        # Connect to the target server
        server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server_socket.connect((host, port))
        
        # Tell client the tunnel is established
        client_socket.send(b"HTTP/1.1 200 Connection Established\r\n\r\n")
        
        # Forward data in both directions
        relay(client_socket, server_socket)
    else:
        # HTTP request — forward as-is
        url = first_line.split(' ')[1]
        host = request.split('Host: ')[1].split('\r\n')[0]
        
        server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server_socket.connect((host, 80))
        server_socket.send(request.encode())
        
        # Get response and forward to client
        response = server_socket.recv(65536)
        client_socket.send(response)
        server_socket.close()
    
    client_socket.close()

def relay(client_sock, server_sock):
    """Relay data between client and server (bidirectional tunnel)."""
    def forward(src, dst):
        while True:
            data = src.recv(4096)
            if not data:
                break
            dst.send(data)
    
    t1 = threading.Thread(target=forward, args=(client_sock, server_sock))
    t2 = threading.Thread(target=forward, args=(server_sock, client_sock))
    t1.start()
    t2.start()

# Start the forward proxy on port 8888
proxy = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
proxy.bind(('0.0.0.0', 8888))
proxy.listen(100)
print("Forward Proxy running on port 8888")

while True:
    client, addr = proxy.accept()
    print(f"Connection from {addr}")
    threading.Thread(target=handle_client, args=(client,)).start()
```

### Python — Simple Reverse Proxy (using Flask)

```python
# reverse_proxy.py — A minimal reverse proxy that routes to backend servers
import requests
from flask import Flask, request, Response

app = Flask(__name__)

# Backend servers pool
BACKENDS = [
    "http://localhost:8001",
    "http://localhost:8002",
    "http://localhost:8003",
]
current_backend = 0  # Simple round-robin counter

def get_next_backend():
    """Round-robin backend selection."""
    global current_backend
    backend = BACKENDS[current_backend % len(BACKENDS)]
    current_backend += 1
    return backend

@app.route('/', defaults={'path': ''})
@app.route('/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
def proxy(path):
    """Forward all requests to a backend server."""
    backend = get_next_backend()
    target_url = f"{backend}/{path}"
    
    # Forward the request with original headers
    headers = {key: value for key, value in request.headers if key != 'Host'}
    headers['X-Forwarded-For'] = request.remote_addr
    headers['X-Forwarded-Proto'] = request.scheme
    
    # Make the request to the backend
    resp = requests.request(
        method=request.method,
        url=target_url,
        headers=headers,
        data=request.get_data(),
        params=request.args,
        allow_redirects=False
    )
    
    # Return backend's response to the client
    return Response(resp.content, status=resp.status_code, headers=dict(resp.headers))

if __name__ == '__main__':
    app.run(port=80)  # Reverse proxy on port 80
```

### Java — Reverse Proxy with Spring Boot

```java
// ReverseProxyController.java — Routes requests to backend servers
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;
import jakarta.servlet.http.HttpServletRequest;
import java.util.concurrent.atomic.AtomicInteger;

@RestController
public class ReverseProxyController {

    private final RestTemplate restTemplate = new RestTemplate();
    private final String[] backends = {
        "http://backend1:8080",
        "http://backend2:8080",
        "http://backend3:8080"
    };
    private final AtomicInteger counter = new AtomicInteger(0);

    // Round-robin backend selection
    private String getNextBackend() {
        int index = counter.getAndIncrement() % backends.length;
        return backends[Math.abs(index)];
    }

    @RequestMapping("/**")
    public ResponseEntity<byte[]> proxy(HttpServletRequest request) {
        String backend = getNextBackend();
        String targetUrl = backend + request.getRequestURI();
        
        // Build headers — forward client's real IP
        HttpHeaders headers = new HttpHeaders();
        headers.set("X-Forwarded-For", request.getRemoteAddr());
        headers.set("X-Forwarded-Proto", request.getScheme());
        
        HttpEntity<byte[]> entity = new HttpEntity<>(headers);
        
        // Forward request to backend
        ResponseEntity<byte[]> response = restTemplate.exchange(
            targetUrl,
            HttpMethod.valueOf(request.getMethod()),
            entity,
            byte[].class
        );
        
        return ResponseEntity
            .status(response.getStatusCode())
            .headers(response.getHeaders())
            .body(response.getBody());
    }
}
```

---

## Infrastructure Examples

### Nginx as a Reverse Proxy

```nginx
# /etc/nginx/nginx.conf — Nginx reverse proxy configuration

upstream backend_servers {
    # Load balance across 3 backend servers
    server 10.0.1.10:8080 weight=3;   # Gets 3x traffic
    server 10.0.1.11:8080 weight=2;   # Gets 2x traffic
    server 10.0.1.12:8080 weight=1;   # Gets 1x traffic
    
    # Health check — remove unhealthy servers
    # max_fails=3 means after 3 failures, mark as down
    server 10.0.1.13:8080 backup;     # Only used when others are down
}

server {
    listen 443 ssl;
    server_name api.myapp.com;
    
    # SSL termination (reverse proxy handles HTTPS)
    ssl_certificate /etc/ssl/certs/myapp.crt;
    ssl_certificate_key /etc/ssl/private/myapp.key;
    
    # Forward all requests to backend
    location / {
        proxy_pass http://backend_servers;
        
        # Pass real client info to backend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }
    
    # Cache static assets
    location /static/ {
        proxy_pass http://backend_servers;
        proxy_cache_valid 200 1h;
        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

### Squid as a Forward Proxy

```bash
# /etc/squid/squid.conf — Forward proxy configuration

# Define who can use this proxy
acl internal_network src 10.0.0.0/8
acl allowed_sites dstdomain .github.com .stackoverflow.com

# Access rules
http_access allow internal_network allowed_sites
http_access deny all

# Listen on port 3128
http_port 3128

# Cache settings
cache_mem 512 MB
maximum_object_size 100 MB

# Logging
access_log /var/log/squid/access.log
```

### Docker Compose — Reverse Proxy + App Servers

```yaml
# docker-compose.yml — Nginx reverse proxy with multiple app instances
version: '3.8'

services:
  reverse-proxy:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/ssl/certs
    depends_on:
      - app1
      - app2
      - app3

  app1:
    image: myapp:latest
    expose:
      - "8080"
    environment:
      - INSTANCE_ID=1

  app2:
    image: myapp:latest
    expose:
      - "8080"
    environment:
      - INSTANCE_ID=2

  app3:
    image: myapp:latest
    expose:
      - "8080"
    environment:
      - INSTANCE_ID=3
```

---

## Real-World Example

### Cloudflare as a Reverse Proxy

Every website behind Cloudflare uses a reverse proxy:

```
User (anywhere in world)
    │
    ▼
┌──────────────────────┐
│  Cloudflare Edge PoP │ ← Reverse proxy closest to user
│  (300+ locations)    │
│                      │
│  • SSL termination   │
│  • DDoS protection   │
│  • Caching           │
│  • WAF rules         │
│  • Rate limiting     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Origin Server       │ ← Your actual server (hidden)
│  (your-server.com)   │
│  IP: 10.0.1.5        │ ← Never exposed to public
└──────────────────────┘
```

**How Cloudflare works:**
1. You point your DNS to Cloudflare (they become the reverse proxy)
2. All traffic hits Cloudflare's edge first
3. Cloudflare filters attacks, caches content, and forwards clean requests
4. Your origin server IP is never exposed to the public

### Corporate Forward Proxy (Zscaler)

Companies like Zscaler provide cloud-based forward proxies:

```
Employee Laptop
    │
    ▼
┌──────────────────────┐
│  Zscaler Cloud Proxy │ ← Forward proxy
│                      │
│  • URL filtering     │ (block social media)
│  • Malware scanning  │ (scan downloads)
│  • DLP              │ (prevent data leaks)
│  • SSL inspection    │ (decrypt & inspect HTTPS)
│  • Logging          │ (compliance/audit)
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│    Internet          │
│    (websites)        │
└──────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Not forwarding `X-Forwarded-For` header | Backend sees proxy IP instead of real client IP | Always set `proxy_set_header X-Forwarded-For` |
| Trusting `X-Forwarded-For` blindly | Clients can spoof this header | Only trust it from known proxies (use `set_real_ip_from`) |
| No timeouts on reverse proxy | One slow backend blocks all connections | Set `proxy_connect_timeout` and `proxy_read_timeout` |
| SSL termination without re-encryption | Traffic between proxy and backend is plaintext | Use internal TLS or keep within trusted network (VPC) |
| Single reverse proxy (no HA) | Proxy itself becomes single point of failure | Use multiple proxies behind a VIP (Virtual IP) or DNS failover |
| Caching POST/PUT responses | Stale data gets served | Only cache GET requests with proper cache-control |
| Not handling WebSockets | WebSocket upgrades fail through proxy | Configure `proxy_http_version 1.1` and `Upgrade` headers |

---

## When to Use / When NOT to Use

### Forward Proxy — Use When:
- ✅ You need to control/filter outbound internet access
- ✅ You want to hide client identities (privacy)
- ✅ Caching frequently accessed external resources
- ✅ Bypassing geo-restrictions (VPN-like behavior)
- ✅ Corporate compliance and logging

### Forward Proxy — Don't Use When:
- ❌ You need to protect your servers (that's a reverse proxy)
- ❌ You need load balancing
- ❌ Performance is critical and proxy adds latency you can't afford

### Reverse Proxy — Use When:
- ✅ You have multiple backend servers to distribute load
- ✅ You want to terminate SSL in one place
- ✅ You need to hide your internal server architecture
- ✅ You want to add security (WAF, rate limiting) without modifying app code
- ✅ You want caching at the edge
- ✅ You need to serve different backends based on URL path

### Reverse Proxy — Don't Use When:
- ❌ You have a single server with low traffic (unnecessary complexity)
- ❌ Latency is critical and every millisecond matters (adds 1-5ms)
- ❌ You just need a simple static file server

---

## Key Takeaways

1. **Forward proxy** protects clients; **reverse proxy** protects servers. The "direction" matters.
2. **VPNs are forward proxies** — they hide your IP from websites you visit.
3. **Cloudflare, Nginx, and CDNs are reverse proxies** — they hide your server from the internet.
4. Reverse proxies enable **SSL termination**, **load balancing**, **caching**, and **security** without changing your application code.
5. Always forward the **real client IP** (`X-Forwarded-For`) to your backend servers, but never trust it blindly from untrusted sources.
6. A reverse proxy is almost always present in production architectures — it's rare to expose application servers directly to the internet.
7. The **same software** (Nginx, HAProxy) can act as both a forward proxy and a reverse proxy depending on configuration.

---

## What's Next?

Next, we'll take a deep dive into **Nginx & HAProxy** — the two most popular reverse proxies/load balancers in the industry. You'll learn their architecture, configuration, and when to choose one over the other.

→ [02-nginx-haproxy.md](./02-nginx-haproxy.md)
