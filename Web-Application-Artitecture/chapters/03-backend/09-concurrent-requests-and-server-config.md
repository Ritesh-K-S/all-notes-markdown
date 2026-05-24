# Chapter 3.9: How Many Concurrent Requests Can a Server Handle?

> **Level**: ⭐⭐⭐ Advanced  
> **What you'll learn**: The real factors that determine how many users your server can serve simultaneously — from hardware limits to software configuration — with actual numbers and tuning strategies.

---

## 🧠 Real-Life Analogy: A Highway Toll Booth

```
    Think of your server as a HIGHWAY TOLL BOOTH:
    
    ┌────────────────────────────────────────────────────────────┐
    │  Cars = Requests                                          │
    │  Toll booths = Threads/Workers                            │
    │  Road lanes = Network bandwidth                           │
    │  Parking lot = Memory (RAM)                               │
    │  Toll operator speed = CPU speed                          │
    │                                                            │
    │  Traffic:                                                  │
    │  ═══════                                                  │
    │  🚗🚗🚗🚗🚗🚗🚗🚗  (incoming cars = requests)           │
    │         │                                                  │
    │    ┌────┼────┐                                            │
    │    │    │    │  Toll booths (3 open)                       │
    │    ▼    ▼    ▼                                            │
    │  [🧑‍💼1] [🧑‍💼2] [🧑‍💼3]                                     │
    │    │    │    │                                             │
    │    ▼    ▼    ▼                                            │
    │  🚗   🚗   🚗  (processed cars)                          │
    │                                                            │
    │  Q: How many cars per hour?                               │
    │  Depends on:                                               │
    │  1. Number of toll booths (threads)                       │
    │  2. How fast each operator works (CPU speed)              │
    │  3. How many lanes approach (network bandwidth)           │
    │  4. Is there parking for waiting cars? (queue/memory)     │
    │  5. Does operator need to call HQ? (DB/external calls)   │
    └────────────────────────────────────────────────────────────┘
```

---

## 📖 The Formula: Concurrent Requests

```
    ┌──────────────────────────────────────────────────────────────┐
    │  THE FUNDAMENTAL FORMULA:                                    │
    │                                                              │
    │                    Concurrent Requests                       │
    │  Throughput = ──────────────────────────                     │
    │                Average Response Time                         │
    │                                                              │
    │                                                              │
    │  Example:                                                    │
    │  - You have 200 threads                                     │
    │  - Each request takes 100ms (0.1s) on average               │
    │                                                              │
    │  Throughput = 200 / 0.1 = 2,000 requests/second            │
    │                                                              │
    │  If response time increases to 500ms:                       │
    │  Throughput = 200 / 0.5 = 400 requests/second              │
    │                                                              │
    │  → Slow responses KILL throughput!                          │
    │  → A slow DB query affects ALL users, not just the one.    │
    └──────────────────────────────────────────────────────────────┘
    
    
    MAX CONCURRENT CONNECTIONS ≠ MAX CONCURRENT REQUESTS:
    ═════════════════════════════════════════════════════
    
    ┌──────────────────────────────────────────────────────┐
    │  Connection: TCP socket between client and server.  │
    │  Request: An actual HTTP request being processed.    │
    │                                                      │
    │  Nginx can have 10,000 CONNECTIONS open              │
    │  but only 1,000 REQUESTS being processed.           │
    │  The other 9,000 are idle (keepalive).              │
    │                                                      │
    │  Think of it as:                                     │
    │  - 10,000 phone lines installed (connections)       │
    │  - Only 1,000 people actively talking (requests)    │
    │  - 9,000 lines open but silent (keepalive/idle)     │
    └──────────────────────────────────────────────────────┘
```

---

## 🔧 Real Server Configurations & Their Limits

### Nginx (Reverse Proxy / Static Files)

```
    ┌──────────────────────────────────────────────────────────────┐
    │  NGINX — Event-driven, non-blocking                         │
    │                                                              │
    │  # /etc/nginx/nginx.conf                                    │
    │  worker_processes auto;       # = number of CPU cores       │
    │  events {                                                    │
    │      worker_connections 10240; # connections PER worker      │
    │  }                                                           │
    │                                                              │
    │  Max connections = worker_processes × worker_connections    │
    │  Example: 4 cores × 10,240 = 40,960 connections!           │
    │                                                              │
    │  Typical capacity:                                           │
    │  ┌──────────────────┬──────────────────────────────┐       │
    │  │  Scenario        │  Requests/second             │       │
    │  ├──────────────────┼──────────────────────────────┤       │
    │  │  Static files    │  50,000 - 100,000+ req/s    │       │
    │  │  Reverse proxy   │  20,000 - 50,000 req/s      │       │
    │  │  SSL termination │  10,000 - 30,000 req/s      │       │
    │  └──────────────────┴──────────────────────────────┘       │
    │                                                              │
    │  Memory usage: ~2.5 MB per 1,000 connections               │
    │  = 40K connections ≈ 100 MB RAM. Incredibly efficient!     │
    └──────────────────────────────────────────────────────────────┘
```

### Tomcat / Spring Boot (Java)

```
    ┌──────────────────────────────────────────────────────────────┐
    │  TOMCAT — Thread pool based                                 │
    │                                                              │
    │  # application.yml                                          │
    │  server:                                                     │
    │    tomcat:                                                    │
    │      threads:                                                │
    │        min-spare: 25         # Min idle threads              │
    │        max: 200              # Max worker threads            │
    │      max-connections: 10000  # Max TCP connections           │
    │      accept-count: 100      # Queue when all threads busy   │
    │      connection-timeout: 20000  # 20 seconds                │
    │                                                              │
    │  Flow when under load:                                      │
    │                                                              │
    │  Requests    Threads   Queue     Result                     │
    │  ─────────   ───────   ─────     ──────                     │
    │  1-25        25        0         ✅ Handled immediately     │
    │  26-200      200       0         ✅ New threads created     │
    │  201-300     200       100       ⏳ Queued, waiting         │
    │  301+        200       FULL      ❌ CONNECTION REFUSED!     │
    │                                                              │
    │  Typical capacity with 200 threads:                         │
    │  ┌──────────────────┬──────────────────────────────┐       │
    │  │  Response time   │  Throughput                  │       │
    │  ├──────────────────┼──────────────────────────────┤       │
    │  │  10ms (fast API) │  200 / 0.01 = 20,000 req/s  │       │
    │  │  100ms (DB query)│  200 / 0.1  = 2,000 req/s   │       │
    │  │  1s (slow query) │  200 / 1.0  = 200 req/s     │       │
    │  │  5s (timeout)    │  200 / 5.0  = 40 req/s ⚠️   │       │
    │  └──────────────────┴──────────────────────────────┘       │
    └──────────────────────────────────────────────────────────────┘
```

### Gunicorn + Flask/Django (Python)

```
    ┌──────────────────────────────────────────────────────────────┐
    │  GUNICORN — Process-based (because of Python GIL)           │
    │                                                              │
    │  # Start command                                             │
    │  gunicorn app:app \                                          │
    │    --workers 4 \           # 4 worker processes              │
    │    --threads 4 \           # 4 threads per process           │
    │    --worker-class gthread  # Thread-based workers            │
    │                                                              │
    │  Total concurrency = workers × threads = 4 × 4 = 16        │
    │                                                              │
    │  Formula for workers:                                        │
    │  workers = (2 × CPU_CORES) + 1                              │
    │  4 cores → 9 workers                                         │
    │                                                              │
    │  With async workers (uvicorn):                              │
    │  gunicorn app:app -k uvicorn.workers.UvicornWorker \        │
    │    --workers 4                                               │
    │  Each worker handles thousands of concurrent connections!   │
    │  Total: 4 workers × 10K each = 40K concurrent!             │
    │                                                              │
    │  Memory per worker process: ~50-150 MB                     │
    │  9 workers × 100 MB = ~900 MB RAM                          │
    └──────────────────────────────────────────────────────────────┘
```

### Node.js

```
    ┌──────────────────────────────────────────────────────────────┐
    │  NODE.JS — Single-threaded event loop                       │
    │                                                              │
    │  Single process:    ~10,000-40,000 concurrent connections   │
    │  Cluster mode (4):  ~40,000-160,000 concurrent connections  │
    │                                                              │
    │  // cluster mode                                             │
    │  const cluster = require('cluster');                         │
    │  const numCPUs = require('os').cpus().length;               │
    │                                                              │
    │  if (cluster.isPrimary) {                                   │
    │    for (let i = 0; i < numCPUs; i++) {                     │
    │      cluster.fork();  // Fork worker processes              │
    │    }                                                         │
    │  } else {                                                    │
    │    // Each worker runs event loop independently             │
    │    app.listen(3000);                                         │
    │  }                                                           │
    │                                                              │
    │  Throughput (I/O-bound):                                    │
    │  ┌──────────────────┬──────────────────────────────┐       │
    │  │  Setup           │  Requests/second             │       │
    │  ├──────────────────┼──────────────────────────────┤       │
    │  │  Single process  │  10,000 - 30,000 req/s      │       │
    │  │  4-core cluster  │  30,000 - 100,000 req/s     │       │
    │  │  PM2 cluster     │  Same, but managed            │       │
    │  └──────────────────┴──────────────────────────────┘       │
    └──────────────────────────────────────────────────────────────┘
```

---

## 💻 Code Examples

### Python — Load Testing Your Server

```python
"""
Simple load tester to measure concurrent request handling.
Shows how response time degrades under load.
"""
import asyncio
import aiohttp
import time

async def send_request(session, url, request_id):
    """Send a single request and measure response time."""
    start = time.time()
    try:
        async with session.get(url) as response:
            await response.text()
            elapsed = time.time() - start
            return {"id": request_id, "status": response.status, 
                    "time_ms": round(elapsed * 1000)}
    except Exception as e:
        return {"id": request_id, "error": str(e), 
                "time_ms": round((time.time() - start) * 1000)}

async def load_test(url, concurrent_requests, total_requests):
    """Run a load test with specified concurrency."""
    print(f"\nLoad Test: {total_requests} requests, "
          f"{concurrent_requests} concurrent")
    print("=" * 50)
    
    results = []
    semaphore = asyncio.Semaphore(concurrent_requests)
    
    async def limited_request(session, req_id):
        async with semaphore:
            return await send_request(session, url, req_id)
    
    start = time.time()
    async with aiohttp.ClientSession() as session:
        tasks = [limited_request(session, i) 
                 for i in range(total_requests)]
        results = await asyncio.gather(*tasks)
    
    total_time = time.time() - start
    
    # Calculate statistics
    times = [r["time_ms"] for r in results if "time_ms" in r]
    errors = [r for r in results if "error" in r]
    
    print(f"Total time:    {total_time:.2f}s")
    print(f"Throughput:    {total_requests / total_time:.0f} req/s")
    print(f"Avg response:  {sum(times) / len(times):.0f}ms")
    print(f"Min response:  {min(times)}ms")
    print(f"Max response:  {max(times)}ms")
    print(f"Errors:        {len(errors)}")

# Test with increasing concurrency
asyncio.run(load_test("http://localhost:8080/api/users", 10, 100))
asyncio.run(load_test("http://localhost:8080/api/users", 100, 1000))
asyncio.run(load_test("http://localhost:8080/api/users", 500, 5000))
```

### Java — Monitoring Thread Pool Utilization

```java
/**
 * Spring Boot actuator + custom metrics for monitoring
 * thread pool saturation in production.
 */
@RestController
public class ServerCapacityController {

    @GetMapping("/api/server-status")
    public Map<String, Object> getServerStatus() {
        // Get Tomcat thread pool info
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        Runtime runtime = Runtime.getRuntime();
        
        return Map.of(
            "threads", Map.of(
                "active", threadBean.getThreadCount(),
                "peak", threadBean.getPeakThreadCount(),
                "daemon", threadBean.getDaemonThreadCount()
            ),
            "memory", Map.of(
                "total_mb", runtime.totalMemory() / (1024 * 1024),
                "free_mb", runtime.freeMemory() / (1024 * 1024),
                "used_mb", (runtime.totalMemory() - runtime.freeMemory()) 
                           / (1024 * 1024),
                "max_mb", runtime.maxMemory() / (1024 * 1024)
            ),
            "cpu", Map.of(
                "available_processors", runtime.availableProcessors()
            )
        );
    }
}

// Spring Boot application.yml — tuning for high traffic:
// server:
//   tomcat:
//     threads:
//       max: 400        # Increase for I/O-heavy workloads
//     max-connections: 20000
//     accept-count: 200
//   compression:
//     enabled: true     # Compress responses (less bandwidth)
```

---

## 🔧 The Bottleneck Chain

```
    A server's capacity is limited by the WEAKEST link:
    
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  Request flow and potential bottlenecks:                    │
    │                                                              │
    │  Internet ──▶ [Network Bandwidth]                          │
    │               │                                              │
    │               ▼ Bottleneck? Max 1 Gbps = ~125 MB/s         │
    │                                                              │
    │  Nginx ────▶ [Connection Limit]                            │
    │               │                                              │
    │               ▼ Bottleneck? worker_connections × workers    │
    │                                                              │
    │  App Server ──▶ [Thread/Worker Pool]                       │
    │               │                                              │
    │               ▼ Bottleneck? max 200 threads in Tomcat      │
    │                                                              │
    │  Application ──▶ [CPU Processing]                          │
    │               │                                              │
    │               ▼ Bottleneck? 4 cores = 4 things at once     │
    │                                                              │
    │  Database ──▶ [DB Connection Pool]                         │
    │               │                                              │
    │               ▼ Bottleneck? max 100 DB connections          │
    │                                                              │
    │  Database ──▶ [Disk I/O]                                   │
    │               │                                              │
    │               ▼ Bottleneck? SSD: 500K IOPS, HDD: 200 IOPS │
    │                                                              │
    │  Memory ──▶ [RAM]                                          │
    │               │                                              │
    │               ▼ Bottleneck? 16 GB shared across everything  │
    │                                                              │
    │                                                              │
    │  REAL EXAMPLE:                                              │
    │  ═════════════                                              │
    │  Tomcat: 200 threads, each request takes 100ms             │
    │  Theoretical: 200 / 0.1 = 2,000 req/s                     │
    │                                                              │
    │  BUT your DB pool has only 20 connections!                 │
    │  200 threads fight over 20 DB connections.                 │
    │  180 threads WAIT for a DB connection.                     │
    │  Actual: ~200 req/s (10x less than theoretical!)           │
    │                                                              │
    │  The DB pool is the bottleneck, not Tomcat.                │
    └──────────────────────────────────────────────────────────────┘
```

---

## 📊 Real Numbers: What Companies Handle

```
    ┌──────────────────┬───────────────────┬───────────────────────┐
    │  Company         │  Scale            │  How                  │
    ├──────────────────┼───────────────────┼───────────────────────┤
    │  Google Search   │  99,000 queries/s │  1000s of servers     │
    │                  │                   │  custom infra         │
    ├──────────────────┼───────────────────┼───────────────────────┤
    │  Twitter/X       │  500,000 tweets/s │  Scala + custom       │
    │                  │  (peak: >140K/s)  │  message queues       │
    ├──────────────────┼───────────────────┼───────────────────────┤
    │  Netflix         │  100,000+ req/s   │  Zuul (Netty) gateway │
    │                  │  per gateway      │  event-driven         │
    ├──────────────────┼───────────────────┼───────────────────────┤
    │  Amazon          │  1,000,000+ req/s │  100s of microservices│
    │                  │  during Prime Day │  auto-scaling         │
    ├──────────────────┼───────────────────┼───────────────────────┤
    │  WhatsApp        │  50 billion       │  Erlang/BEAM VM       │
    │                  │  messages/day     │  2 million conn/server│
    ├──────────────────┼───────────────────┼───────────────────────┤
    │  Discord         │  1M+ concurrent   │  Elixir + Rust        │
    │                  │  per voice server │  custom BEAM          │
    └──────────────────┴───────────────────┴───────────────────────┘
    
    
    FOR YOUR APP (typical single server):
    ═════════════════════════════════════
    
    ┌──────────────────┬────────────┬────────────────────────┐
    │  Server Stack    │  1 Server  │  With Load Balancer    │
    ├──────────────────┼────────────┼────────────────────────┤
    │  Flask + Gunicorn│  500-2K/s  │  5K-20K/s (10 servers)│
    │  Spring Boot     │  2K-10K/s  │  20K-100K/s            │
    │  Node.js         │  10K-30K/s │  100K-300K/s           │
    │  Go (net/http)   │  50K-100K/s│  500K-1M/s             │
    │  Rust (Actix)    │  100K-200K+│  1M+/s                 │
    └──────────────────┴────────────┴────────────────────────┘
    
    These are I/O-bound estimates. CPU-bound workloads = much less.
```

---

## 🛠️ Tuning Checklist

```
    ┌──────────────────────────────────────────────────────────────┐
    │  STEP-BY-STEP SERVER TUNING:                                │
    │                                                              │
    │  1. ⬜ Identify the bottleneck first!                       │
    │     Don't guess. Use metrics: CPU%, memory, thread pool     │
    │     utilization, DB connection pool, response times.        │
    │                                                              │
    │  2. ⬜ Tune the bottleneck (not random settings!)          │
    │     - CPU at 100%? → Add more servers or optimize code     │
    │     - All threads busy? → Increase thread pool size        │
    │     - DB pool exhausted? → Increase pool + add replicas   │
    │     - Memory full? → Reduce per-request allocation         │
    │                                                              │
    │  3. ⬜ Set proper timeouts                                  │
    │     - Connection timeout: 5-10s                             │
    │     - Read timeout: 30s                                     │
    │     - Idle timeout: 60s                                     │
    │     Without timeouts, slow clients consume resources!       │
    │                                                              │
    │  4. ⬜ Enable keepalive connections                         │
    │     Reuse TCP connections instead of creating new ones.     │
    │     Saves the 3-way handshake (~1ms) per request.          │
    │                                                              │
    │  5. ⬜ Compress responses                                   │
    │     gzip/brotli reduces payload by 70-90%.                 │
    │     Less data = faster transfer = more throughput.          │
    │                                                              │
    │  6. ⬜ Load test before going to production!               │
    │     Tools: wrk, k6, Apache Bench (ab), Locust, Gatling     │
    └──────────────────────────────────────────────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Guessing the bottleneck instead of measuring
       → "Let's add more servers!" when the DB is the bottleneck
       ✅ Use monitoring tools (Prometheus, Grafana, New Relic)
    
    ❌ Setting thread pool way too high
       → 10,000 threads on a 4-core server = context switch hell
       ✅ Start with (2 × CPU cores) + 1 for CPU-bound work
    
    ❌ No timeout on upstream connections
       → One slow DB query holds a thread for 60+ seconds
       ✅ Set connection + query timeouts (5-30 seconds)
    
    ❌ Ignoring the database connection pool
       → 200 app threads but only 10 DB connections = 190 threads waiting
       ✅ DB pool should be ~25-50% of your thread pool size
    
    ❌ Testing on localhost and assuming production will be the same
       → Localhost has zero network latency, unlimited bandwidth
       ✅ Test with realistic network conditions and data volumes
    
    ❌ Optimizing code before identifying the bottleneck
       → Spending a week optimizing a function that takes 1ms
          when the DB query takes 500ms
       ✅ Profile first. Optimize the slowest part.
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. Throughput = Concurrent Workers / Avg Response Time.            ║
║     200 threads at 100ms = 2,000 req/s. At 1s = only 200 req/s.   ║
║                                                                      ║
║  2. Your server's capacity is limited by the WEAKEST link:         ║
║     network → connections → threads → CPU → DB pool → disk I/O.   ║
║                                                                      ║
║  3. Different servers have different models:                        ║
║     Nginx: 40K+ connections (event-driven, very efficient).        ║
║     Tomcat: ~200 threads (increase for I/O-heavy workloads).       ║
║     Node.js: 10K-30K/s per process (cluster for more).             ║
║                                                                      ║
║  4. ALWAYS identify the bottleneck BEFORE tuning. Use monitoring   ║
║     tools, not guesses.                                              ║
║                                                                      ║
║  5. Set timeouts everywhere. A slow request without a timeout      ║
║     can take down your entire server.                                ║
║                                                                      ║
║  6. Load test before production! Tools: k6, Locust, wrk, Gatling. ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

We mentioned the DB connection pool as a common bottleneck. Let's learn what connection pooling is and why it's essential for any production server. Next: [Chapter 3.10: Connection Pooling](./10-connection-pooling.md).

---

[⬅️ Previous: Concurrency & Parallelism](./08-concurrency-and-parallelism.md) | [⬆️ Index](../../00-INDEX.md) | [Next: Connection Pooling ➡️](./10-connection-pooling.md)
