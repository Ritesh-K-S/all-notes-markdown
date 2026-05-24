# Request Timeouts — Don't Wait Forever

> **What you'll learn**: How to set smart time limits on network calls so your application doesn't hang indefinitely when things go wrong.

---

## Real-Life Analogy

Imagine you're at a restaurant and you've placed your order. You wait 5 minutes — fine. 10 minutes — a bit long. 30 minutes — something's clearly wrong. At some point, you decide: "I'm not going to wait any longer. I'll go somewhere else."

That's exactly what a **timeout** does in software. It says: *"If I don't get a response within X seconds, I'll stop waiting and do something else."*

Without a timeout, your application is like a person who sits in that restaurant forever — blocking their entire evening, never eating, never moving on.

---

## Core Concept Explained Step-by-Step

### What is a Timeout?

A **timeout** is a maximum amount of time your application is willing to wait for a response from another system (a database, an API, a microservice, etc.).

```
┌─────────────┐         Request          ┌─────────────┐
│   Service A │ ───────────────────────▶  │   Service B │
│  (Caller)   │                           │  (Callee)   │
│             │   ⏱️ Timer starts...      │  (Slow/Down)│
│             │                           │             │
│             │   ⏱️ TIMEOUT! (3 sec)     │             │
│             │ ◀── No response ────────  │             │
│  "Give up!" │                           │             │
└─────────────┘                           └─────────────┘
```

### Why Are Timeouts Critical?

Without timeouts, a single slow dependency can take down your entire system:

```
WITHOUT TIMEOUTS:
                                                    
User ──▶ Web Server ──▶ Payment Service (HUNG!)
              │
              │  Thread is blocked... waiting...
              │  More requests pile up...
              │  All threads used up!
              │
              ▼
         SERVER CRASH! (Thread pool exhausted)


WITH TIMEOUTS:

User ──▶ Web Server ──▶ Payment Service (SLOW!)
              │
              │  Timer: 3 seconds...
              │  TIMEOUT! Return error to user
              │  Thread freed immediately
              │
              ▼
         Server stays healthy ✓
```

### Types of Timeouts

```
┌────────────────────────────────────────────────────────────────┐
│                    TIMEOUT TYPES                                │
├────────────────────┬───────────────────────────────────────────┤
│ Connection Timeout │ Max time to ESTABLISH a connection        │
│                    │ (TCP handshake)                           │
├────────────────────┼───────────────────────────────────────────┤
│ Read Timeout       │ Max time to RECEIVE data after           │
│ (Socket Timeout)   │ connection is established                 │
├────────────────────┼───────────────────────────────────────────┤
│ Write Timeout      │ Max time to SEND data to the server      │
├────────────────────┼───────────────────────────────────────────┤
│ Request Timeout    │ Total time for the ENTIRE request         │
│ (Overall)          │ (connection + send + receive)             │
├────────────────────┼───────────────────────────────────────────┤
│ Idle Timeout       │ Max time a connection can sit unused      │
│                    │ before being closed                       │
└────────────────────┴───────────────────────────────────────────┘
```

### The Timeline of a Request

```
|◀── Connection Timeout ──▶|◀────── Read Timeout ──────▶|
|                           |                             |
|  DNS + TCP Handshake      |  Waiting for response body  |
|  + TLS Negotiation        |                             |
|                           |                             |
|◀──────────────── Request Timeout (Overall) ───────────▶|
```

---

## How It Works Internally

### 1. Connection Timeout (TCP Level)

When your application tries to connect to another service, the TCP handshake must complete:

```
Client                          Server
  │                               │
  │──── SYN ─────────────────────▶│  ← Timer starts
  │                               │
  │◀─── SYN-ACK ─────────────────│  ← Timer check
  │                               │
  │──── ACK ─────────────────────▶│  ← Connection established!
  │                               │     Timer stopped
```

If the server is unreachable (firewall dropping packets, server down), the SYN-ACK never comes. Without a timeout, the OS will retry SYN packets with exponential backoff for up to **2+ minutes** on Linux (controlled by `tcp_syn_retries`).

**Connection timeout says**: "If I don't complete the handshake in 2 seconds, give up."

### 2. Read Timeout (Application Level)

After the connection is established, you send your request and wait for data:

```
Client                          Server
  │                               │
  │── GET /api/users ────────────▶│  ← Read timer starts
  │                               │
  │    (Server processing...)     │  ← Server is busy
  │    (Still processing...)      │
  │    (5 seconds pass...)        │
  │                               │
  │  ⏱️ READ TIMEOUT!            │  ← Timer expires!
  │                               │
  │  Client closes connection     │
```

### 3. How the OS Manages Timeouts

Under the hood, timeouts use OS-level mechanisms:

- **`select()`/`poll()`/`epoll()`** — The application tells the OS: "Watch this socket for data, but only wait X milliseconds"
- **`SO_RCVTIMEO`** — Socket option that sets the receive timeout at the kernel level
- **`SO_SNDTIMEO`** — Socket option that sets the send timeout
- **Application-level timers** — Libraries use threads or event loops to track elapsed time

### 4. Timeout in Different Layers

```
┌─────────────────────────────────────────────────────────┐
│  Layer                │  Timeout Controls               │
├───────────────────────┼─────────────────────────────────┤
│  Browser/Client       │  XMLHttpRequest.timeout         │
│                       │  fetch AbortController          │
├───────────────────────┼─────────────────────────────────┤
│  Load Balancer        │  proxy_read_timeout (Nginx)     │
│                       │  timeout server (HAProxy)       │
├───────────────────────┼─────────────────────────────────┤
│  API Gateway          │  route-level timeout config     │
├───────────────────────┼─────────────────────────────────┤
│  Application Code     │  HTTP client timeout settings   │
├───────────────────────┼─────────────────────────────────┤
│  Database Client      │  statement_timeout (PostgreSQL) │
│                       │  socketTimeoutMS (MongoDB)      │
├───────────────────────┼─────────────────────────────────┤
│  Connection Pool      │  maxWaitMillis / checkout time  │
└───────────────────────┴─────────────────────────────────┘
```

---

## Code Examples

### Python — Using `requests` with Timeouts

```python
import requests
from requests.exceptions import ConnectTimeout, ReadTimeout, Timeout

# ALWAYS set timeouts! Never use requests without them.
try:
    response = requests.get(
        "https://api.payment-service.com/charge",
        timeout=(3, 10)  # (connection_timeout, read_timeout) in seconds
    )
    print(f"Payment successful: {response.json()}")
    
except ConnectTimeout:
    # Could not establish connection within 3 seconds
    print("Payment service is unreachable. Try again later.")
    
except ReadTimeout:
    # Connected, but response took longer than 10 seconds
    # DANGER: The payment might have gone through!
    print("Payment service is slow. Check payment status before retrying.")
    
except Timeout:
    # Generic timeout (catches both)
    print("Request timed out.")
```

### Python — Using `httpx` with Total Timeout

```python
import httpx

# httpx gives you more granular timeout control
timeout_config = httpx.Timeout(
    connect=5.0,    # Time to establish connection
    read=10.0,      # Time to receive response
    write=5.0,      # Time to send request body
    pool=2.0        # Time to acquire connection from pool
)

async def call_payment_service(order_id: str):
    async with httpx.AsyncClient(timeout=timeout_config) as client:
        try:
            response = await client.post(
                "https://api.payment.com/charge",
                json={"order_id": order_id, "amount": 99.99}
            )
            return response.json()
        except httpx.TimeoutException as e:
            # Log which timeout was hit
            print(f"Timeout calling payment service: {type(e).__name__}")
            raise
```

### Java — Using HttpClient with Timeouts

```java
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.net.URI;

public class PaymentClient {
    
    // Configure timeouts at the client level
    private final HttpClient client = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(3))  // Connection timeout
        .build();
    
    public String chargePayment(String orderId) {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://api.payment.com/charge"))
            .timeout(Duration.ofSeconds(10))  // Overall request timeout
            .header("Content-Type", "application/json")
            .POST(HttpRequest.BodyPublishers.ofString(
                "{\"order_id\": \"" + orderId + "\", \"amount\": 99.99}"))
            .build();
        
        try {
            HttpResponse<String> response = client.send(
                request, HttpResponse.BodyHandlers.ofString());
            return response.body();
        } catch (java.net.http.HttpTimeoutException e) {
            // Request timed out — service too slow or down
            System.err.println("Payment service timeout: " + e.getMessage());
            throw new RuntimeException("Payment service unavailable", e);
        } catch (Exception e) {
            throw new RuntimeException("Payment call failed", e);
        }
    }
}
```

### Java — Using Spring RestTemplate with Timeouts

```java
import org.springframework.http.client.SimpleClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.client.ResourceAccessException;

public class TimeoutConfigExample {
    
    public RestTemplate createRestTemplateWithTimeouts() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(3000);  // 3 seconds to connect
        factory.setReadTimeout(10000);    // 10 seconds to read
        
        return new RestTemplate(factory);
    }
    
    public void callService() {
        RestTemplate restTemplate = createRestTemplateWithTimeouts();
        try {
            String result = restTemplate.getForObject(
                "http://inventory-service/api/stock/{id}", 
                String.class, "SKU-123");
            System.out.println("Stock: " + result);
        } catch (ResourceAccessException e) {
            // This catches timeout exceptions
            System.err.println("Service call timed out: " + e.getMessage());
        }
    }
}
```

---

## Infrastructure Examples

### Nginx — Proxy Timeouts

```nginx
# /etc/nginx/conf.d/upstream.conf
upstream backend_api {
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
}

server {
    listen 80;
    
    location /api/ {
        proxy_pass http://backend_api;
        
        # Time to establish connection to upstream
        proxy_connect_timeout 5s;
        
        # Time to receive response headers from upstream
        proxy_read_timeout 30s;
        
        # Time to send request body to upstream
        proxy_send_timeout 10s;
        
        # If upstream doesn't respond, try next server
        proxy_next_upstream timeout error;
        proxy_next_upstream_timeout 60s;
        proxy_next_upstream_tries 3;
    }
}
```

### PostgreSQL — Query Timeouts

```sql
-- Set query timeout for current session (prevents runaway queries)
SET statement_timeout = '5000';  -- 5 seconds

-- Set timeout for all connections from a specific role
ALTER ROLE web_app SET statement_timeout = '30000';  -- 30 seconds

-- Lock wait timeout (don't wait forever for a lock)
SET lock_timeout = '3000';  -- 3 seconds

-- Idle connection timeout (close connections idle too long)
SET idle_in_transaction_session_timeout = '60000';  -- 60 seconds
```

### Redis — Timeout Configuration

```bash
# redis.conf
timeout 300          # Close idle connections after 300 seconds
tcp-keepalive 60     # Send TCP keepalive every 60 seconds

# In application:
# redis-py client with timeout
```

```python
import redis

# Redis client with timeouts
r = redis.Redis(
    host='redis-cluster.internal',
    port=6379,
    socket_timeout=2,        # Read/write timeout: 2 seconds
    socket_connect_timeout=1, # Connection timeout: 1 second
    retry_on_timeout=True     # Auto-retry on timeout
)
```

### Kubernetes — Liveness & Readiness Probe Timeouts

```yaml
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: api-server
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          timeoutSeconds: 3        # Each probe must respond in 3s
          periodSeconds: 10
          failureThreshold: 3      # 3 failures = restart container
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          timeoutSeconds: 2        # Must respond in 2s to receive traffic
          periodSeconds: 5
```

---

## Real-World Example

### Amazon's Microservices Timeout Strategy

Amazon has thousands of microservices. Their approach:

```
┌─────────────────────────────────────────────────────────────┐
│              AMAZON'S TIMEOUT PHILOSOPHY                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. EVERY call has a timeout. No exceptions.                │
│                                                             │
│  2. Timeouts are set based on p99 latency + buffer          │
│     (If p99 = 200ms, timeout = 500ms)                       │
│                                                             │
│  3. Cascading timeouts decrease through the chain:          │
│                                                             │
│     API Gateway (30s) → Service A (10s) → Service B (3s)   │
│                                                             │
│  4. Total timeout < Upstream timeout                        │
│     (So the caller can still respond with an error)         │
│                                                             │
│  5. Read timeouts are separate from connection timeouts     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Netflix's Timeout Configuration

Netflix uses **Hystrix** (now replaced by **Resilience4j**) to manage timeouts:

```
Request from User
       │
       ▼
┌──────────────┐    timeout: 30s
│  API Gateway │ ──────────────────────┐
└──────────────┘                       │
       │                               │
       ▼                               ▼
┌──────────────┐    timeout: 5s    ┌────────────┐
│ Recommendation│──────────────────│  User      │
│ Service      │                   │  Profile   │
└──────────────┘                   └────────────┘
       │                               │
       ▼ timeout: 2s                   ▼ timeout: 1s
┌──────────────┐                   ┌────────────┐
│  ML Model    │                   │   Cache    │
│  Service     │                   │  (Redis)   │
└──────────────┘                   └────────────┘

Rule: Each downstream call has a SHORTER timeout than
      the caller's own timeout to the outside world.
```

---

## Common Mistakes / Pitfalls

### 1. No Timeout Set At All (The #1 Mistake!)

```python
# ❌ NEVER do this! Default timeout is often infinite.
response = requests.get("http://slow-service.com/data")

# ✅ ALWAYS set a timeout
response = requests.get("http://slow-service.com/data", timeout=(3, 10))
```

### 2. Timeout Too Long

Setting a 60-second timeout means a dead service blocks threads for 60 seconds. Under load, your thread pool exhausts in seconds.

### 3. Timeout Too Short

If your database query legitimately takes 5 seconds and your timeout is 2 seconds, you'll get false failures. **Always base timeouts on measured p99 latency**.

### 4. Same Timeout for Everything

```
❌ BAD: All services get 30s timeout

✅ GOOD:
  - Redis cache lookup:     200ms timeout
  - Database query:         5s timeout
  - External payment API:   15s timeout
  - File upload:            60s timeout
```

### 5. Not Handling Timeout Ambiguity

```
DANGER ZONE:
You call a payment API with a 5s timeout.
The timeout fires at 5s.

But did the payment go through? You don't know!
The server might have processed it at 4.9 seconds.

Solution: Use idempotency keys (Chapter 12.7)
          and check payment status before retrying.
```

### 6. Cascading Timeout Mismatch

```
❌ BAD:
API Gateway timeout: 10s
  └─ Service A timeout to Service B: 30s  ← LONGER than gateway!

What happens:
- Gateway kills the request at 10s
- Service A keeps waiting until 30s (wasted resources!)
- User already got an error

✅ GOOD:
API Gateway timeout: 30s
  └─ Service A timeout to Service B: 10s  ← SHORTER
       └─ Service B timeout to DB: 3s     ← EVEN SHORTER
```

---

## When to Use / When NOT to Use

### ✅ When to Use Timeouts

| Scenario | Recommended Timeout |
|----------|-------------------|
| Internal microservice call | 1-5 seconds |
| Database query | 5-30 seconds |
| External third-party API | 10-30 seconds |
| File upload/download | 30-120 seconds |
| Health checks | 1-3 seconds |
| Cache lookup (Redis) | 100-500 milliseconds |
| DNS resolution | 2-5 seconds |

### ⚠️ When to Be Careful

| Scenario | Consideration |
|----------|---------------|
| Long-running batch jobs | Use job-level monitoring, not HTTP timeouts |
| WebSocket connections | Use heartbeats instead of simple timeouts |
| File streaming | Use idle timeout, not total timeout |
| Database migrations | Temporarily increase or remove statement_timeout |

### ❌ When NOT to Use Short Timeouts

- **Fire-and-forget async operations** — Use message queues instead
- **Large data transfers** — Use chunked uploads with per-chunk timeouts
- **WebSocket/SSE streams** — Use heartbeat/ping mechanisms

---

## How to Choose the Right Timeout Value

```
Step 1: Measure actual latency
        └─ Use p50, p95, p99 from monitoring

Step 2: Add safety buffer
        └─ Timeout = p99 × 2 (or p99 + fixed buffer)

Step 3: Consider cascade effects
        └─ Downstream timeout < Your timeout < Upstream timeout

Step 4: Monitor timeout rates
        └─ If >1% requests timeout → investigate root cause
        └─ If <0.01% → timeout might be too generous

Step 5: Adjust over time
        └─ Latency changes as traffic grows
        └─ Re-evaluate timeouts quarterly
```

---

## Key Takeaways

- **Every network call MUST have a timeout.** No exceptions. Libraries often default to infinity.
- **Connection timeouts should be short** (1-5s) — if you can't connect in 5 seconds, the server is likely down.
- **Read timeouts depend on the operation** — fast lookups need short timeouts, complex queries need longer ones.
- **Timeouts should cascade downward** — each hop in your service chain should have a shorter timeout than the previous one.
- **Timeout ≠ failure** — a timeout means "I don't know what happened." Handle the ambiguity, especially for writes.
- **Base timeouts on real metrics** — use p99 latency plus a buffer, not guesswork.
- **Monitor timeout rates** — a spike in timeouts is an early warning of system degradation.

---

## What's Next?

Timeouts tell you when to give up on a single request. But what do you do after a timeout? You **retry** — but naively retrying can make things worse. In the next chapter, [02-retry-mechanism.md](./02-retry-mechanism.md), we'll learn how to retry smartly using exponential backoff, jitter, and retry budgets.
