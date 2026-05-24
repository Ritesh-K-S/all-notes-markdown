# HTTP/2 & HTTP/3 (QUIC) — Faster Protocols

> **What you'll learn**: How HTTP/2 and HTTP/3 fundamentally redesigned web communication to eliminate bottlenecks that plagued HTTP/1.1 for decades — multiplexing, header compression, server push, and the revolutionary move from TCP to UDP.

---

## Real-Life Analogy

Imagine ordering food at a restaurant:

**HTTP/1.1** = One waiter, one order at a time:
- "I'd like soup." → Waiter goes to kitchen, comes back with soup.
- "Now I'd like salad." → Waiter goes to kitchen, comes back with salad.
- "Now the main course." → ...
- If the kitchen is slow on one dish, EVERYTHING waits!

**HTTP/2** = One waiter, takes ALL orders at once:
- "I'd like soup, salad, main course, and dessert." → Waiter sends ALL orders to kitchen simultaneously.
- Kitchen delivers each dish as it's ready — no waiting in sequence!
- But if the waiter trips (connection error), ALL dishes are delayed.

**HTTP/3** = Multiple waiters, each carrying one dish independently:
- Each dish travels its own path from kitchen to table.
- If one waiter trips, only THAT dish is delayed — the others still arrive fine!

```
HTTP/1.1: Sequential (one at a time, blocking)
──────────────────────────────────────────────
Client ──▶[Request 1]──────────────▶[Response 1]
                                              ──▶[Request 2]──────▶[Response 2]
                                                                            ──▶[Request 3]─▶[Resp 3]

HTTP/2: Multiplexed (all at once, one connection)
──────────────────────────────────────────────────
Client ──▶[Req 1]──▶[Resp 1]
       ──▶[Req 2]─────────▶[Resp 2]
       ──▶[Req 3]──▶[Resp 3]
       (All on ONE TCP connection, interleaved!)

HTTP/3: Multiplexed + Independent streams (no HOL blocking)
─────────────────────────────────────────────────────────────
Stream 1: ──▶[Req 1]──▶[Resp 1] ✓
Stream 2: ──▶[Req 2]───────────▶[Resp 2] ✓  (slow but independent)
Stream 3: ──▶[Req 3]──▶[Resp 3] ✓           (not blocked by Stream 2!)
(Each stream independent — packet loss on one doesn't affect others!)
```

---

## Core Concept Explained Step-by-Step

### Step 1: The Problem with HTTP/1.1

HTTP/1.1 was designed in 1997. Its major problems:

```
Problem 1: HEAD-OF-LINE (HOL) BLOCKING
──────────────────────────────────────
  One slow response blocks everything behind it:
  
  Client ──▶ [GET image.jpg] ────(1000ms)────▶ [Response]
              [GET style.css]  ← BLOCKED waiting for image!
              [GET app.js]     ← BLOCKED!
              [GET font.woff]  ← BLOCKED!

Problem 2: MULTIPLE CONNECTIONS NEEDED
──────────────────────────────────────
  Workaround: Open 6 connections per domain!
  
  Connection 1: [GET image1.jpg]
  Connection 2: [GET image2.jpg]
  Connection 3: [GET style.css]
  Connection 4: [GET app.js]
  Connection 5: [GET image3.jpg]
  Connection 6: [GET font.woff]
  
  Each connection = separate TCP handshake + TLS handshake
  = 6 × (50ms + 100ms) = 900ms overhead! 😱

Problem 3: REPETITIVE HEADERS
─────────────────────────────
  Every request sends the SAME headers (cookies, User-Agent, etc.)
  Typical header size: 500-2000 bytes PER REQUEST
  
  100 requests × 1KB headers = 100KB of redundant data!
```

### Step 2: HTTP/2 — The Major Upgrade (2015)

```
┌──────────────────────────────────────────────────────────────┐
│                   HTTP/2 KEY FEATURES                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. MULTIPLEXING                                             │
│     Multiple requests/responses on ONE connection            │
│     No more 6-connection hack!                               │
│                                                              │
│  2. BINARY FRAMING                                           │
│     Binary protocol (not text like HTTP/1.1)                 │
│     More efficient parsing by machines                       │
│                                                              │
│  3. HEADER COMPRESSION (HPACK)                               │
│     Compresses headers, caches repeated headers              │
│     500 bytes → 30 bytes for subsequent requests!            │
│                                                              │
│  4. SERVER PUSH                                              │
│     Server can send resources before client asks             │
│     "I know you'll need style.css, here it is!"             │
│                                                              │
│  5. STREAM PRIORITIZATION                                    │
│     Tell the server which resources are most important       │
│     CSS > images (render CSS first!)                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Step 3: How HTTP/2 Multiplexing Works

```
HTTP/1.1: 6 TCP connections (wasteful)
┌─────────┐    ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
│ Browser │────│TCP 1│─│TCP 2│─│TCP 3│─│TCP 4│─│TCP 5│─│TCP 6│──▶ Server
└─────────┘    └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘

HTTP/2: 1 TCP connection, many streams
┌─────────┐    ┌──────────────────────────────────────────┐
│ Browser │────│ Single TCP Connection                     │──▶ Server
└─────────┘    │ ┌──────────────────────────────────────┐ │
               │ │ Stream 1: [HEADERS][DATA][DATA]      │ │
               │ │ Stream 3: [HEADERS][DATA]            │ │
               │ │ Stream 5: [HEADERS][DATA][DATA][DATA]│ │
               │ │ Stream 7: [HEADERS][DATA]            │ │
               │ └──────────────────────────────────────┘ │
               └──────────────────────────────────────────┘

Frames are INTERLEAVED on the wire:
  [S1:HEADER][S3:HEADER][S1:DATA][S5:HEADER][S3:DATA][S1:DATA][S5:DATA]...
  
  Each frame has a stream ID → demultiplexed at the receiver
```

### Step 4: HTTP/2's Remaining Problem — TCP HOL Blocking

```
The TCP layer doesn't know about HTTP/2 streams!

TCP sees: [packet1][packet2][packet3][packet4][packet5]

If packet 2 is lost:
  TCP: "I must deliver packets IN ORDER!"
  
  [packet1] ✓ delivered
  [packet2] ✗ LOST — retransmit!
  [packet3] ⏸ buffered, waiting for packet 2
  [packet4] ⏸ buffered, waiting for packet 2
  [packet5] ⏸ buffered, waiting for packet 2
  
  Even though packets 3,4,5 might be for DIFFERENT streams,
  TCP blocks ALL of them until packet 2 is retransmitted!
  
  This is TCP-level HOL blocking → HTTP/3 solves this!
```

### Step 5: HTTP/3 — Built on QUIC (UDP)

```
┌──────────────────────────────────────────────────────────────┐
│                   HTTP/3 ARCHITECTURE                         │
│                                                              │
│  HTTP/1.1          HTTP/2            HTTP/3                  │
│  ┌────────┐        ┌────────┐        ┌────────┐            │
│  │ HTTP   │        │ HTTP/2 │        │ HTTP/3 │            │
│  ├────────┤        ├────────┤        ├────────┤            │
│  │  TLS   │        │  TLS   │        │  QUIC  │ ← Includes │
│  ├────────┤        ├────────┤        │(TLS 1.3│   TLS!     │
│  │  TCP   │        │  TCP   │        │ built  │            │
│  ├────────┤        ├────────┤        │  in)   │            │
│  │   IP   │        │   IP   │        ├────────┤            │
│  └────────┘        └────────┘        │  UDP   │            │
│                                      ├────────┤            │
│                                      │   IP   │            │
│                                      └────────┘            │
│                                                              │
│  3 handshakes:     3 handshakes:     1 handshake:           │
│  TCP + TLS         TCP + TLS         QUIC (0-RTT possible!) │
│  = 3 RTT           = 3 RTT           = 1 RTT (or 0!)       │
└──────────────────────────────────────────────────────────────┘
```

### Step 6: QUIC's Key Innovations

```
1. INDEPENDENT STREAMS (no TCP HOL blocking):
─────────────────────────────────────────────
   Stream 1: [pkt1][pkt2✗ lost][pkt3]
   Stream 2: [pkt4][pkt5][pkt6]     ← NOT blocked by Stream 1!
   Stream 3: [pkt7][pkt8]           ← NOT blocked by Stream 1!
   
   Only Stream 1 waits for retransmission. Others continue!

2. FASTER CONNECTION SETUP (0-RTT):
────────────────────────────────────
   First connection:  1 RTT (QUIC handshake + TLS combined)
   Repeat connection: 0 RTT! (cached credentials)
   
   TCP + TLS 1.3: Minimum 2 RTT
   QUIC:          Minimum 1 RTT (0 for repeat visits!)
   
   On mobile (100ms RTT): saves 100-200ms per connection!

3. CONNECTION MIGRATION:
────────────────────────
   TCP: Connection = (src IP, src port, dst IP, dst port)
        Change WiFi → new IP → connection DIES → must reconnect
   
   QUIC: Connection = Connection ID (random token)
         Change WiFi → same Connection ID → stream continues!
   
   Perfect for mobile: switch WiFi ↔ cellular seamlessly!
```

---

## How It Works Internally

### HTTP/2 Binary Framing Layer

```
HTTP/1.1 (text):
  GET /index.html HTTP/1.1\r\n
  Host: example.com\r\n
  Accept: text/html\r\n
  \r\n

HTTP/2 (binary frames):
  ┌───────────────────────────────────┐
  │ Length (24 bits) │ Type (8 bits)  │
  ├───────────────────────────────────┤
  │ Flags (8 bits)  │ Stream ID (31) │
  ├───────────────────────────────────┤
  │              Payload              │
  └───────────────────────────────────┘

Frame Types:
  DATA (0x0)     — Response body
  HEADERS (0x1)  — HTTP headers (compressed)
  PRIORITY (0x2) — Stream priority
  RST_STREAM(0x3)— Cancel a stream
  SETTINGS (0x4) — Connection parameters
  PUSH_PROMISE(0x5) — Server push notification
  PING (0x6)     — Keep-alive / measure RTT
  GOAWAY (0x7)   — Graceful shutdown
  WINDOW_UPDATE(0x8) — Flow control
```

### HPACK Header Compression

```
First Request Headers (full):
  :method: GET
  :path: /api/users
  :authority: example.com
  cookie: session=abc123xyz...  (800 bytes!)
  user-agent: Mozilla/5.0 (Windows NT 10.0; ...)
  accept: application/json
  
  Total: ~1200 bytes

Second Request Headers (HPACK compressed):
  Static table index: 2 (:method: GET)     → 1 byte
  Static table index: 4 (:path: /api/users)→ encoded path
  Dynamic table ref: "same authority"      → 1 byte
  Dynamic table ref: "same cookie"         → 1 byte
  Dynamic table ref: "same user-agent"     → 1 byte
  Static table index: 19 (accept: json)    → 1 byte
  
  Total: ~30 bytes! (97% smaller!)

HPACK uses:
  1. Static table: 61 common header name/value pairs
  2. Dynamic table: Headers from previous requests (per-connection)
  3. Huffman encoding: Compress header string values
```

### QUIC Packet Structure

```
QUIC Packet (on top of UDP):
┌─────────────────────────────────────────────────────┐
│  Header (unencrypted)                               │
│  ├── Header Form (1 bit): Long/Short               │
│  ├── Connection ID (0-20 bytes)                    │
│  ├── Packet Number (1-4 bytes)                     │
│  └── Version (for Long header)                     │
├─────────────────────────────────────────────────────┤
│  Payload (encrypted with TLS 1.3)                  │
│  ├── STREAM frames (actual HTTP data)              │
│  ├── ACK frames (acknowledgments)                  │
│  ├── CRYPTO frames (TLS handshake)                │
│  └── Other frames (PING, CONNECTION_CLOSE, etc.)  │
└─────────────────────────────────────────────────────┘

Key difference from TCP:
  TCP packet loss → retransmit exact packet → blocks all data
  QUIC packet loss → retransmit only that stream's data → others unblocked!
```

### Connection Establishment Comparison

```
TCP + TLS 1.3 (HTTP/2):                QUIC (HTTP/3):
─────────────────────────              ─────────────────────────
Client          Server                 Client          Server
  │                │                     │                │
  │──── SYN ──────▶│ }                   │                │
  │◀── SYN-ACK ───│ } TCP               │── QUIC Initial▶│ } 1 RTT
  │──── ACK ──────▶│ } 1 RTT            │  (+ TLS hello) │ } combined!
  │                │                     │◀── QUIC Handshk│ }
  │── ClientHello ▶│ }                   │── Finished ───▶│
  │◀─ ServerHello ─│ } TLS              │                │
  │── Finished ───▶│ } 1 RTT            │── DATA ───────▶│  ← Send data!
  │                │                     │                │
  │── DATA ───────▶│  ← Send data       Total: 1 RTT (100ms saved!)
  │                │                     
  Total: 2 RTT                          Repeat visit: 0-RTT!
  (On 100ms RTT: 200ms before data)     (On 100ms RTT: 0ms!)
```

---

## Code Examples

### Python — HTTP/2 Client with httpx

```python
import httpx
import asyncio
import time

# HTTP/2 client (multiplexed requests)
async def fetch_with_http2():
    """Fetch multiple resources concurrently over HTTP/2"""
    async with httpx.AsyncClient(http2=True) as client:
        urls = [
            "https://api.example.com/users",
            "https://api.example.com/products",
            "https://api.example.com/orders",
            "https://api.example.com/categories",
            "https://api.example.com/reviews",
        ]
        
        start = time.perf_counter()
        
        # All requests multiplexed on ONE connection!
        responses = await asyncio.gather(
            *[client.get(url) for url in urls]
        )
        
        elapsed = (time.perf_counter() - start) * 1000
        print(f"HTTP/2: {len(responses)} requests in {elapsed:.0f}ms")
        print(f"  Protocol: {responses[0].http_version}")
        # Output: "HTTP/2: 5 requests in 85ms" (vs ~500ms with HTTP/1.1)

# HTTP/2 server with hypercorn (ASGI)
# pip install hypercorn quart
from quart import Quart, jsonify

app = Quart(__name__)

@app.route('/api/data')
async def get_data():
    return jsonify({"message": "Served over HTTP/2!"})

# Run: hypercorn app:app --bind 0.0.0.0:443 --certfile cert.pem --keyfile key.pem
# Hypercorn natively supports HTTP/2

# Measuring HTTP/2 vs HTTP/1.1 performance
async def benchmark_protocols():
    """Compare HTTP/1.1 vs HTTP/2 for multiple requests"""
    url = "https://api.example.com/data"
    n_requests = 20
    
    # HTTP/1.1 (sequential due to HOL blocking)
    async with httpx.AsyncClient(http2=False) as client:
        start = time.perf_counter()
        for _ in range(n_requests):
            await client.get(url)
        http1_time = (time.perf_counter() - start) * 1000
    
    # HTTP/2 (multiplexed)
    async with httpx.AsyncClient(http2=True) as client:
        start = time.perf_counter()
        await asyncio.gather(*[client.get(url) for _ in range(n_requests)])
        http2_time = (time.perf_counter() - start) * 1000
    
    print(f"HTTP/1.1: {http1_time:.0f}ms")
    print(f"HTTP/2:   {http2_time:.0f}ms")
    print(f"Speedup:  {http1_time/http2_time:.1f}x")

asyncio.run(benchmark_protocols())
```

### Java — HTTP/2 Client (built into JDK 11+)

```java
import java.net.URI;
import java.net.http.*;
import java.net.http.HttpClient.Version;
import java.time.Duration;
import java.util.*;
import java.util.concurrent.*;

public class Http2Example {
    
    public static void main(String[] args) throws Exception {
        // Java HttpClient natively supports HTTP/2!
        HttpClient client = HttpClient.newBuilder()
            .version(Version.HTTP_2)           // Prefer HTTP/2
            .connectTimeout(Duration.ofSeconds(5))
            .build();

        // Single request — automatically upgrades to HTTP/2
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://api.example.com/users"))
            .GET()
            .build();

        HttpResponse<String> response = client.send(request, 
            HttpResponse.BodyHandlers.ofString());
        System.out.println("Protocol: " + response.version()); // HTTP_2

        // Multiplexed concurrent requests (HTTP/2 advantage!)
        List<HttpRequest> requests = List.of(
            HttpRequest.newBuilder(URI.create("https://api.example.com/users")).build(),
            HttpRequest.newBuilder(URI.create("https://api.example.com/products")).build(),
            HttpRequest.newBuilder(URI.create("https://api.example.com/orders")).build(),
            HttpRequest.newBuilder(URI.create("https://api.example.com/reviews")).build(),
            HttpRequest.newBuilder(URI.create("https://api.example.com/categories")).build()
        );

        long start = System.nanoTime();
        
        // All sent concurrently on ONE connection via HTTP/2!
        List<CompletableFuture<HttpResponse<String>>> futures = requests.stream()
            .map(req -> client.sendAsync(req, HttpResponse.BodyHandlers.ofString()))
            .toList();

        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        
        long elapsed = (System.nanoTime() - start) / 1_000_000;
        System.out.printf("HTTP/2: %d requests in %dms%n", requests.size(), elapsed);
        // vs HTTP/1.1: each request sequential = 5 × RTT time
    }
}
```

---

## Infrastructure Examples

### Nginx — HTTP/2 and HTTP/3 Configuration

```nginx
# nginx.conf — HTTP/2 + HTTP/3 setup

# HTTP/2 (over TLS — required)
server {
    listen 443 ssl http2;           # Enable HTTP/2
    listen [::]:443 ssl http2;      # IPv6
    
    # HTTP/3 (QUIC over UDP)
    listen 443 quic reuseport;      # Enable HTTP/3/QUIC
    listen [::]:443 quic reuseport;
    
    server_name example.com;
    
    ssl_certificate /etc/ssl/certs/example.com.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    
    # TLS 1.3 (required for HTTP/3)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    
    # Tell browsers HTTP/3 is available (Alt-Svc header)
    add_header Alt-Svc 'h3=":443"; ma=86400';  # Valid for 24 hours
    
    # HTTP/2 specific settings
    http2_max_concurrent_streams 128;    # Max parallel streams
    http2_initial_window_size 1048576;   # 1MB flow control window
    
    # HTTP/2 Server Push (use sparingly — being deprecated)
    location / {
        http2_push /static/css/style.css;
        http2_push /static/js/app.js;
        proxy_pass http://backend;
    }
}

# Redirect HTTP to HTTPS (HTTP/2 requires TLS)
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

### Caddy Server — Automatic HTTP/3

```Caddyfile
# Caddy automatically enables HTTP/2 and HTTP/3!
example.com {
    # HTTP/3 enabled by default (with auto TLS)
    
    # Reverse proxy to backend
    reverse_proxy localhost:8080
    
    # Static files with compression
    handle /static/* {
        root * /var/www/static
        file_server {
            precompressed br gzip
        }
    }
}
```

### Kubernetes Ingress — HTTP/2 with NGINX Ingress Controller

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    # Enable HTTP/2 for backend connections too
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"  # Uses HTTP/2
    nginx.ingress.kubernetes.io/proxy-http-version: "2"
    # HTTP/2 settings
    nginx.ingress.kubernetes.io/http2-max-concurrent-streams: "128"
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

---

## Real-World Example

### Google (Inventors of QUIC/HTTP/3)

```
Google's HTTP/3 deployment:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Google invented QUIC in 2012, standardized as HTTP/3   │
│                                                         │
│  Results after deploying HTTP/3:                        │
│  • YouTube: 30% reduction in rebuffering events        │
│  • Google Search: 2% faster page loads (huge at scale!)│
│  • Google Maps: 25% faster on poor networks            │
│                                                         │
│  Why it helps on bad networks:                          │
│  • 0-RTT connection resumption                         │
│  • No TCP HOL blocking (packet loss doesn't stall all) │
│  • Connection migration (WiFi → cellular seamless)     │
│                                                         │
│  Today: 30%+ of all Google traffic uses HTTP/3          │
└─────────────────────────────────────────────────────────┘
```

### Cloudflare — HTTP/3 at the Edge

```
Cloudflare's measurements (HTTP/2 vs HTTP/3):
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Time to First Byte (TTFB):                            │
│    HTTP/2: 180ms average                               │
│    HTTP/3: 120ms average (-33% improvement!)            │
│                                                         │
│  On high packet-loss networks (3G mobile):              │
│    HTTP/2: 500ms+ (TCP retransmission stalls)          │
│    HTTP/3: 200ms (only affected stream stalls)          │
│            60% improvement on lossy networks!           │
│                                                         │
│  Connection establishment:                              │
│    HTTP/2: 3 RTT (TCP SYN + TLS 1.3)                  │
│    HTTP/3: 1 RTT (0 RTT for repeat visits!)            │
│                                                         │
│  Adoption: 25%+ of all web traffic uses HTTP/3 (2024)  │
└─────────────────────────────────────────────────────────┘
```

---

## Protocol Comparison Table

```
┌───────────────────┬──────────────┬──────────────┬──────────────┐
│    Feature        │  HTTP/1.1    │   HTTP/2     │   HTTP/3     │
├───────────────────┼──────────────┼──────────────┼──────────────┤
│ Year              │ 1997         │ 2015         │ 2022         │
│ Transport         │ TCP          │ TCP          │ QUIC (UDP)   │
│ Multiplexing      │ No           │ Yes          │ Yes          │
│ HOL blocking      │ HTTP-level   │ TCP-level    │ None!        │
│ Header compress   │ None         │ HPACK        │ QPACK        │
│ Connection setup  │ 3 RTT        │ 3 RTT       │ 1 RTT (0!)  │
│ Server push       │ No           │ Yes          │ Yes          │
│ Encryption        │ Optional     │ Practical:   │ Always       │
│                   │              │ required     │ (built-in)   │
│ Connection migrate│ No           │ No           │ Yes!         │
│ Format            │ Text         │ Binary       │ Binary       │
│ Max streams       │ 1/connection │ ~100/conn    │ ~100/conn    │
│ Best for          │ Legacy       │ Most sites   │ Mobile/lossy │
└───────────────────┴──────────────┴──────────────┴──────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Domain sharding with HTTP/2 | Defeats multiplexing! Multiple connections = wasted | Use one domain; HTTP/2 multiplexes on one connection |
| Concatenating/spriting for HTTP/2 | No longer needed — multiplexing handles many small files | Use individual files; let HTTP/2 stream them |
| Not enabling TLS | HTTP/2 requires HTTPS in practice (browsers mandate it) | Always use TLS 1.2+ for HTTP/2 |
| Server Push without measurement | Often wastes bandwidth (pushes things already cached) | Measure push benefit; most teams remove it |
| Not advertising HTTP/3 (Alt-Svc) | Browsers won't use HTTP/3 unless told it exists | Add `Alt-Svc: h3=":443"` header |
| Blocking QUIC in firewall | UDP port 443 might be blocked by enterprise firewalls | Ensure UDP:443 is open; HTTP/3 falls back to HTTP/2 |
| Ignoring HTTP/2 for backend calls | Microservice-to-microservice communication benefits too | Use gRPC (HTTP/2 native) for internal services |

---

## When to Use / When NOT to Use

### Upgrade to HTTP/2 WHEN:
- You serve any modern website (it's the default now!)
- You have many assets per page (CSS, JS, images, fonts)
- You use gRPC (requires HTTP/2)
- Your users are on mobile networks

### Upgrade to HTTP/3 WHEN:
- Your users are on mobile/unstable networks (3G, WiFi switching)
- Latency is critical (0-RTT saves 100-200ms on repeat visits)
- You experience TCP HOL blocking under packet loss
- You're behind Cloudflare/CDN (they handle it for you!)

### HTTP/3 is less critical WHEN:
- All traffic is on reliable internal networks (datacenter)
- Users have fast, stable connections (enterprise LAN)
- Your infrastructure doesn't support UDP (some firewalls block it)

---

## Key Takeaways

- **HTTP/2** multiplexes all requests over ONE TCP connection — eliminates 6-connection overhead
- **HPACK** compresses HTTP headers by 90%+ using static/dynamic tables
- HTTP/2 still has **TCP-level HOL blocking** — one lost packet stalls ALL streams
- **HTTP/3 uses QUIC** (over UDP) — each stream is independent, loss on one doesn't affect others
- **0-RTT connection resumption** in HTTP/3 eliminates 100-200ms for repeat visitors
- **Connection migration** means mobile users seamlessly switch WiFi ↔ cellular
- HTTP/3 is most impactful on **lossy mobile networks** (30-60% faster!)
- For most sites, HTTP/2 is mandatory; HTTP/3 is a bonus especially for mobile

---

## What's Next?

You now understand how to make individual requests faster (compression, better protocols). But how do you ensure the ENTIRE user experience stays fast? In **Chapter 19.8: Performance Budgets & Core Web Vitals**, you'll learn how to set, measure, and enforce performance targets — the way Google uses them to rank websites.
