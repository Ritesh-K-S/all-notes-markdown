# Latency, Throughput & Response Time — Key Metrics

> **What you'll learn**: The three fundamental metrics that define how "fast" your system really is — and why improving one can sometimes hurt another.

---

## Real-Life Analogy

Imagine a **highway** connecting two cities:

- **Latency** = How long it takes ONE car to travel from City A to City B (travel time for a single trip)
- **Throughput** = How many cars can pass through the highway per hour (total capacity)
- **Response Time** = The entire time from when a driver decides to leave until they arrive — including waiting at traffic lights, toll booths, and the actual drive

A highway with 10 lanes has **high throughput** (many cars at once), but each car still takes 2 hours to travel (latency doesn't change). You can widen the highway (more throughput), but you can't shorten the distance (latency stays).

```
City A ════════════════════════════════ City B
         ← Latency: time for ONE car →

  🚗🚗🚗🚗🚗🚗🚗🚗🚗🚗 ──────────▶
  🚗🚗🚗🚗🚗🚗🚗🚗🚗🚗 ──────────▶   ← Throughput: cars per hour
  🚗🚗🚗🚗🚗🚗🚗🚗🚗🚗 ──────────▶

Response Time = Wait in queue + Travel (latency) + Processing at destination
```

---

## Core Concept Explained Step-by-Step

### Step 1: What is Latency?

**Latency** is the time delay between a request being sent and the first byte of the response being received. It's the "travel time" of data.

```
┌──────────┐           Network            ┌──────────┐
│  Client  │ ──── Request ──────────────▶  │  Server  │
│          │                               │          │
│          │ ◀──── First Byte ───────────  │          │
└──────────┘                               └──────────┘
             │◀──── Latency (ms) ────────▶│
```

**Types of Latency:**

| Type | What it measures | Typical values |
|------|-----------------|----------------|
| Network Latency | Time for data to travel over the wire | 1-200ms |
| Disk Latency | Time to read/write from disk | 0.1-10ms (SSD), 5-20ms (HDD) |
| Application Latency | Time spent in your code processing | 1-500ms |
| Database Latency | Time for a DB query to return | 1-100ms |

### Step 2: What is Throughput?

**Throughput** is the total number of operations (requests, transactions, queries) your system can handle per unit of time.

```
Time ──────────────────────────────────────────▶

     │ req │ req │ req │ req │ req │ req │ req │
     └─────┴─────┴─────┴─────┴─────┴─────┴─────┘
     
     Throughput = 7 requests per second (RPS)
```

Common units:
- **RPS** (Requests Per Second)
- **TPS** (Transactions Per Second)
- **QPS** (Queries Per Second)
- **Mbps / Gbps** (for network throughput)

### Step 3: What is Response Time?

**Response Time** is the TOTAL time from when the client sends a request until it receives the complete response. It includes EVERYTHING:

```
Response Time = Network Latency (to server)
             + Queue Wait Time
             + Processing Time  
             + Network Latency (back to client)

┌────────┐                                    ┌────────┐
│ Client │                                    │ Server │
└───┬────┘                                    └───┬────┘
    │                                             │
    │──── Request sent ─────────────────────────▶│  ← Network latency
    │                                             │
    │                                      ┌──────┤  ← Queuing time
    │                                      │ Queue│
    │                                      └──────┤
    │                                             │
    │                                      ┌──────┤  ← Processing time
    │                                      │ Work │
    │                                      └──────┤
    │                                             │
    │◀──── Response received ────────────────────│  ← Network latency
    │                                             │
    │◀───────── Total Response Time ────────────▶│
```

### Step 4: The Relationship Between All Three

```
                    ┌─────────────────────────────────────┐
                    │         The Performance Triangle     │
                    │                                     │
                    │          RESPONSE TIME              │
                    │           /          \              │
                    │          /            \             │
                    │     LATENCY ──────── THROUGHPUT     │
                    │                                     │
                    │  • Low latency + High throughput    │
                    │    = Fast response time             │
                    │                                     │
                    │  • High latency OR Low throughput   │
                    │    = Slow response time             │
                    └─────────────────────────────────────┘
```

**Key Insight**: Under light load, response time ≈ latency + processing time. But as load increases and throughput is maxed out, requests start QUEUING, and response time shoots up.

```
Response Time (ms)
    │
500 │                                          ╱
    │                                        ╱
400 │                                      ╱
    │                                    ╱
300 │                                  ╱
    │                               ╱╱
200 │                           ╱╱╱
    │                       ╱╱╱
100 │─────────────────╱╱╱╱
    │    (flat zone)
 50 │─────────────
    │
    └──────────────────────────────────────── Load (RPS)
    0    100   200   300   400   500   600   700

    ◀── Comfortable ──▶◀── Stressed ──▶◀── Overloaded ──▶
```

### Step 5: Percentiles — The Right Way to Measure

**Average response time is misleading!** Use percentiles instead:

| Percentile | Meaning | What it tells you |
|-----------|---------|-------------------|
| p50 (median) | 50% of requests are faster than this | Typical user experience |
| p90 | 90% of requests are faster | Most users' experience |
| p95 | 95% are faster | Almost everyone |
| p99 | 99% are faster | Worst-case (except extreme outliers) |
| p99.9 | 99.9% are faster | The "tail latency" — 1 in 1000 |

```
Number of
Requests
    │
    │  ██
    │  ██ ██
    │  ██ ██ ██
    │  ██ ██ ██ ██
    │  ██ ██ ██ ██ ██
    │  ██ ██ ██ ██ ██ ██
    │  ██ ██ ██ ██ ██ ██ ██ ▒▒ ▒▒      ░░
    └──────────────────────────────────────── Response Time
       50ms   100ms  150ms 200ms 300ms  800ms
              ▲          ▲         ▲       ▲
            p50        p90       p95     p99
```

> **Why p99 matters**: If you serve 1 million requests/day, p99 = 800ms means 10,000 users EVERY DAY have a terrible experience.

---

## How It Works Internally

### Little's Law — The Universal Throughput Formula

**Little's Law** connects the three metrics mathematically:

```
L = λ × W

Where:
  L = Average number of requests in the system (concurrency)
  λ = Arrival rate (throughput — requests per second)
  W = Average time each request spends in the system (response time)
```

**Example**: If your server handles 100 RPS (λ) and average response time is 200ms (W = 0.2s):
- L = 100 × 0.2 = **20 concurrent requests** in the system at any time

**Practical implication**: To handle 1000 RPS with 100ms response time, you need capacity for 100 concurrent requests.

### Queuing Theory — Why Systems Get Slow

When a server is 0% utilized, latency is just the processing time. As utilization goes up:

```
                    1
Response Time ∝  ─────────
                 1 - U

Where U = utilization (0 to 1)

U = 0.5 (50%) → Response time = 2x baseline
U = 0.8 (80%) → Response time = 5x baseline  
U = 0.9 (90%) → Response time = 10x baseline ⚠️
U = 0.99 (99%) → Response time = 100x baseline 💀
```

This is why you should **never run servers above 70-80% utilization** in production!

```
Response Time
Multiplier
    │
100x│                                              │
    │                                              │
 50x│                                            ╱ │
    │                                          ╱   │
 20x│                                       ╱╱    │
    │                                    ╱╱       │
 10x│                                ╱╱╱          │
    │                           ╱╱╱╱              │
  5x│                      ╱╱╱╱                   │
    │                ╱╱╱╱╱                        │
  2x│          ╱╱╱╱╱                              │
  1x│─────╱╱╱╱                                    │
    └─────────────────────────────────────────────── Utilization
    0%   20%   40%   60%   80%   90%  95%  99%
                                   ▲
                            DANGER ZONE
```

### Tail Latency Amplification

In microservices, one slow backend call makes the ENTIRE request slow:

```
Client Request
    │
    ├──▶ Service A (5ms)   ✓ fast
    ├──▶ Service B (3ms)   ✓ fast
    ├──▶ Service C (500ms) ✗ SLOW (p99 hit)
    ├──▶ Service D (4ms)   ✓ fast
    │
    └──▶ Total: 500ms+ (entire request is slow!)

If each service has p99 = 100ms, and you call 5 services:
  P(all fast) = 0.99^5 = 95%
  P(at least one slow) = 5% ← 1 in 20 requests is slow!
```

With 20 backend calls:
- P(all fast) = 0.99^20 = **82%** — 18% of user requests hit p99 latency!

This is why **tail latency matters enormously in distributed systems**.

---

## Code Examples

### Python — Measuring Latency and Throughput

```python
import time
import statistics
import asyncio
import aiohttp

# Simple response time measurement
def measure_response_time(func):
    """Decorator to measure function execution time"""
    def wrapper(*args, **kwargs):
        start = time.perf_counter()  # High-precision timer
        result = func(*args, **kwargs)
        elapsed_ms = (time.perf_counter() - start) * 1000
        print(f"{func.__name__}: {elapsed_ms:.2f}ms")
        return result
    return wrapper

# Collecting percentile metrics
class LatencyTracker:
    def __init__(self):
        self.samples = []
    
    def record(self, duration_ms):
        self.samples.append(duration_ms)
    
    def report(self):
        if not self.samples:
            return
        sorted_samples = sorted(self.samples)
        n = len(sorted_samples)
        print(f"Requests: {n}")
        print(f"  p50: {sorted_samples[int(n * 0.50)]:.2f}ms")
        print(f"  p90: {sorted_samples[int(n * 0.90)]:.2f}ms")
        print(f"  p95: {sorted_samples[int(n * 0.95)]:.2f}ms")
        print(f"  p99: {sorted_samples[int(n * 0.99)]:.2f}ms")
        print(f"  max: {sorted_samples[-1]:.2f}ms")
        print(f"  avg: {statistics.mean(sorted_samples):.2f}ms")

# Measuring throughput with concurrent requests
async def measure_throughput(url, total_requests=1000):
    tracker = LatencyTracker()
    start = time.perf_counter()
    
    async with aiohttp.ClientSession() as session:
        tasks = []
        for _ in range(total_requests):
            tasks.append(fetch_one(session, url, tracker))
        await asyncio.gather(*tasks)
    
    elapsed = time.perf_counter() - start
    throughput = total_requests / elapsed
    print(f"\nThroughput: {throughput:.0f} RPS")
    tracker.report()

async def fetch_one(session, url, tracker):
    start = time.perf_counter()
    async with session.get(url) as resp:
        await resp.read()
    elapsed_ms = (time.perf_counter() - start) * 1000
    tracker.record(elapsed_ms)

# Usage: asyncio.run(measure_throughput("http://localhost:8080/api"))
```

### Java — Measuring Latency and Throughput

```java
import java.net.http.*;
import java.net.URI;
import java.time.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.IntStream;

public class PerformanceMetrics {
    private final List<Long> latencies = new CopyOnWriteArrayList<>();

    // Record a single request's response time
    public long measureRequest(HttpClient client, String url) throws Exception {
        long start = System.nanoTime();
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .GET().build();
        client.send(request, HttpResponse.BodyHandlers.ofString());
        long durationMs = (System.nanoTime() - start) / 1_000_000;
        latencies.add(durationMs);
        return durationMs;
    }

    // Run throughput test with concurrent requests
    public void runLoadTest(String url, int totalRequests, int concurrency) 
            throws Exception {
        HttpClient client = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(5)).build();
        ExecutorService pool = Executors.newFixedThreadPool(concurrency);

        long start = System.nanoTime();
        List<Future<?>> futures = IntStream.range(0, totalRequests)
            .mapToObj(i -> pool.submit(() -> {
                try { measureRequest(client, url); }
                catch (Exception e) { e.printStackTrace(); }
            }))
            .toList();

        for (Future<?> f : futures) f.get(); // Wait for all
        double elapsedSec = (System.nanoTime() - start) / 1_000_000_000.0;

        // Calculate percentiles
        List<Long> sorted = latencies.stream().sorted().toList();
        int n = sorted.size();
        System.out.printf("Throughput: %.0f RPS%n", n / elapsedSec);
        System.out.printf("  p50: %dms%n", sorted.get((int)(n * 0.50)));
        System.out.printf("  p90: %dms%n", sorted.get((int)(n * 0.90)));
        System.out.printf("  p95: %dms%n", sorted.get((int)(n * 0.95)));
        System.out.printf("  p99: %dms%n", sorted.get((int)(n * 0.99)));
        System.out.printf("  max: %dms%n", sorted.get(n - 1));

        pool.shutdown();
    }
}
```

---

## Infrastructure Examples

### Prometheus — Collecting Latency Metrics in Production

```yaml
# prometheus.yml - scrape config
scrape_configs:
  - job_name: 'web-app'
    metrics_path: '/metrics'
    scrape_interval: 15s
    static_configs:
      - targets: ['app:8080']
```

```python
# Python app exposing Prometheus histogram for response time
from prometheus_client import Histogram, start_http_server
import time

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'Request latency in seconds',
    ['method', 'endpoint'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

def handle_request(method, endpoint, handler_func):
    start = time.time()
    result = handler_func()
    duration = time.time() - start
    REQUEST_LATENCY.labels(method=method, endpoint=endpoint).observe(duration)
    return result
```

### Grafana Dashboard Query (PromQL)

```promql
# p99 latency over the last 5 minutes
histogram_quantile(0.99, 
  rate(http_request_duration_seconds_bucket[5m])
)

# Throughput (requests per second)
rate(http_request_duration_seconds_count[1m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m]) 
  / rate(http_requests_total[5m]) * 100
```

---

## Real-World Example

### Amazon — The 100ms Rule

Amazon found that **every 100ms of added latency costs 1% in sales**. At their scale, that's hundreds of millions of dollars per year.

```
Amazon's Latency Targets:
┌─────────────────────────────────────────────────┐
│  Page Load:        < 2 seconds                  │
│  API p50:          < 20ms                       │
│  API p99:          < 200ms                      │
│  Database queries: < 5ms (from cache)           │
│  Service-to-service: < 10ms within same AZ      │
└─────────────────────────────────────────────────┘
```

### Google — Tail Latency at Scale

Google's Jeff Dean published research showing that at Google's scale:
- A single user request touches **hundreds of servers**
- If each server has p99 = 10ms, with 100 servers → **63% of requests** will be slow
- They use techniques like "hedged requests" (send duplicate requests to multiple servers, take the first response)

### Netflix — Throughput at Scale

```
Netflix Production Numbers (approximate):
┌─────────────────────────────────────────────────┐
│  Peak streaming:       ~400 Gbps throughput     │
│  API requests:         ~500,000 RPS             │
│  p99 latency target:   < 100ms for API calls    │
│  Microservices calls:  Billions per day         │
└─────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Only measuring average latency | Hides tail latency (p99 can be 10x average) | Always track p50, p90, p95, p99 |
| Confusing latency with response time | Response time includes queuing + processing + latency | Be precise about which metric you mean |
| Optimizing throughput without checking latency | Adding more threads can increase throughput but also latency | Monitor both simultaneously |
| Testing with low load only | Real performance issues appear under high load | Test at expected peak + 2x |
| Ignoring coordinated omission | Load testing tools can miss measuring slow requests | Use open-loop testing (constant arrival rate) |
| Running servers at 90%+ utilization | Queuing theory: response time explodes above 80% | Keep max utilization at 70-80% |

---

## When to Use / When NOT to Use

### When to focus on **Latency**:
- Real-time applications (gaming, trading, video calls)
- User-facing APIs where perceived speed matters
- Geographically distributed users (CDN decisions)

### When to focus on **Throughput**:
- Batch processing systems
- Data pipelines and ETL jobs
- Background workers processing queues

### When to focus on **Response Time**:
- E-commerce (every 100ms = lost revenue)
- Search engines (users leave after 3 seconds)
- Mobile apps on slow networks

---

## Key Takeaways

- **Latency** = time for one request to travel; **Throughput** = requests per unit time; **Response Time** = total wait from the user's perspective
- **Always use percentiles** (p50, p90, p95, p99) — averages lie
- **Little's Law** (L = λ × W) connects concurrency, throughput, and response time mathematically
- **Queuing theory** explains why response time explodes above 80% utilization — never run production at 90%+
- **Tail latency amplification** means even p99 = 10ms becomes a problem when you call many services
- At scale, companies like Amazon lose millions per 100ms of added latency
- Measure these metrics **continuously in production** using Prometheus/Grafana, not just during load tests

---

## What's Next?

Next, we'll learn how to actually **find** what's causing poor latency and low throughput in your application. That's **Chapter 19.2: Profiling Applications — Finding Bottlenecks**, where we'll use profilers, flame graphs, and tracing tools to pinpoint exactly where your code is slow.
