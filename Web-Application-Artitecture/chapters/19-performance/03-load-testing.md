# Load Testing & Stress Testing (JMeter, k6, Locust)

> **What you'll learn**: How to simulate thousands or millions of users hitting your system to find its breaking point — and the tools professionals use to do it.

---

## Real-Life Analogy

Imagine you built a **bridge** and you need to know: "How many cars can it hold before it collapses?"

You wouldn't just open it and hope for the best. You'd:

1. **Load Test**: Drive 100 cars on it. Then 500. Then 1000. Measure how much it sways. ← "How does it perform under expected load?"
2. **Stress Test**: Keep adding cars until cracks appear. 5000? 10000? ← "At what point does it break?"
3. **Soak Test**: Leave 1000 cars on it for 48 hours straight. ← "Does it weaken over time?"
4. **Spike Test**: Instantly add 5000 cars. ← "Can it handle a sudden rush?"

```
Load Testing:     Stress Testing:    Spike Testing:     Soak Testing:
                                      
Users            Users               Users              Users
 │                │                    │                  │
 │      ╱────    │          ╱╱╱      │     │            │ ────────────
 │    ╱          │        ╱╱         │     │            │
 │  ╱            │      ╱╱           │     │            │
 │╱              │   ╱╱╱             │─────┘            │
 └────── Time    └──────── Time      └────── Time       └────── Time
  "Normal load"   "Until break"      "Sudden burst"    "Extended run"
```

---

## Core Concept Explained Step-by-Step

### Step 1: Types of Performance Tests

| Test Type | What it answers | Load Pattern | Duration |
|-----------|----------------|--------------|----------|
| **Load Test** | "Can we handle expected traffic?" | Ramp up to expected peak | 10-60 min |
| **Stress Test** | "Where does the system break?" | Keep increasing until failure | Until crash |
| **Spike Test** | "Can we handle sudden bursts?" | Instant jump to high load | 5-15 min |
| **Soak Test** | "Any memory leaks under sustained load?" | Constant medium load | 4-72 hours |
| **Capacity Test** | "What's our max throughput?" | Increase until SLA is violated | Variable |
| **Breakpoint Test** | "What's the exact breaking point?" | Incremental steps | Variable |

### Step 2: Key Metrics to Monitor During Tests

```
┌─────────────────────────────────────────────────────────────┐
│                METRICS DASHBOARD (during test)               │
├──────────────────┬──────────────────────────────────────────┤
│  Response Time   │  p50: 45ms  p95: 120ms  p99: 450ms     │
│  Throughput      │  2,340 RPS                               │
│  Error Rate      │  0.02%                                   │
│  Active Users    │  500 virtual users                       │
│  CPU (server)    │  65%                                     │
│  Memory          │  78% (2.4GB / 3GB)                      │
│  DB Connections  │  45/50 pool                              │
│  Network I/O     │  450 Mbps in, 1.2 Gbps out             │
└──────────────────┴──────────────────────────────────────────┘

Pass/Fail Criteria:
  ✅ p95 < 200ms
  ✅ Error rate < 1%
  ✅ Throughput > 2000 RPS
  ❌ p99 > 500ms (FAIL — investigate!)
```

### Step 3: The Load Testing Process

```
┌──────────────────────────────────────────────────────────────┐
│                   LOAD TESTING WORKFLOW                       │
│                                                              │
│  1. Define ──▶ 2. Script ──▶ 3. Execute ──▶ 4. Monitor     │
│     goals        test          test           metrics       │
│     & SLAs       scenarios     (ramp up)      in real-time  │
│                                                              │
│  5. Analyze ──▶ 6. Fix ──▶ 7. Re-test ──▶ 8. Report       │
│     results       bottlenecks   (verify fix)    findings    │
└──────────────────────────────────────────────────────────────┘
```

### Step 4: Open-Loop vs Closed-Loop Testing

This is a **critical** distinction most people get wrong:

```
CLOSED-LOOP (wrong for most tests):
─────────────────────────────────────
  Virtual Users wait for response before sending next request
  
  User 1: [──request──][wait][──request──][wait][──request──]
  User 2: [──request──][wait][──request──][wait]
  
  Problem: If server slows down, arrival RATE decreases!
  You're being "nice" to the server — that's not realistic.

OPEN-LOOP (correct for realistic tests):
────────────────────────────────────────
  Requests arrive at a FIXED RATE regardless of response time
  
  Time:  0s    1s    2s    3s    4s    5s
  Rate:  100   100   100   100   100   100  requests/sec
  
  Even if responses take 5 seconds, new requests keep coming.
  This simulates REAL users who don't wait for each other.
  
  Result: Exposes queuing delays and coordinated omission!
```

> **Coordinated omission**: When your testing tool accidentally hides slow responses by reducing its own request rate. Always use open-loop testing to avoid this.

### Step 5: Ramp-Up Patterns

```
Pattern 1: Linear Ramp                Pattern 2: Step Ramp
Users                                  Users
  │        ╱╱╱╱╱╱╱╱                     │        ┌────┐
  │      ╱╱                             │   ┌────┘    │
  │    ╱╱                               │   │    ┌────┘
  │  ╱╱                                 │   │    │
  │╱╱                                   │───┘    │
  └──────────── Time                    └──────────── Time
  Good for finding                      Good for finding 
  gradual degradation                   exact breakpoints

Pattern 3: Spike                       Pattern 4: Wave (realistic)
Users                                  Users
  │     ███                              │   ╱╲      ╱╲
  │     █ █                              │  ╱  ╲    ╱  ╲
  │     █ █                              │ ╱    ╲  ╱    ╲
  │─────┘ └─────                        │╱      ╲╱      ╲
  └──────────── Time                    └──────────── Time
  Flash sale / viral moment             Simulates daily traffic
```

---

## How It Works Internally

### How Load Testing Tools Generate Load

```
┌─────────────────────────────────────────────────────────────┐
│               LOAD GENERATOR ARCHITECTURE                    │
│                                                             │
│  ┌───────────────────────────────────────────────┐         │
│  │           Load Generator Machine               │         │
│  │                                                │         │
│  │  ┌────────┐ ┌────────┐ ┌────────┐           │         │
│  │  │Thread 1│ │Thread 2│ │Thread N│  (or async)│         │
│  │  │  VU 1  │ │  VU 2  │ │  VU N  │           │         │
│  │  └───┬────┘ └───┬────┘ └───┬────┘           │         │
│  │      │          │          │                  │         │
│  │      └──────────┼──────────┘                  │         │
│  │                 │                             │         │
│  └─────────────────┼─────────────────────────────┘         │
│                    │                                        │
│                    ▼ HTTP/gRPC/WebSocket                    │
│  ┌─────────────────────────────────────────────────┐       │
│  │              Target System                       │       │
│  │  ┌──────┐  ┌──────────┐  ┌────────────┐       │       │
│  │  │  LB  │──│  Servers │──│  Database  │       │       │
│  │  └──────┘  └──────────┘  └────────────┘       │       │
│  └─────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Distributed Load Generation

For high loads (100K+ RPS), a single machine isn't enough:

```
┌──────────────┐
│  Controller  │ ← Orchestrates the test
└──────┬───────┘
       │
       ├────────────────────────────────────┐
       │                │                   │
       ▼                ▼                   ▼
┌──────────────┐ ┌──────────────┐  ┌──────────────┐
│  Generator 1 │ │  Generator 2 │  │  Generator N │
│  (Region A)  │ │  (Region B)  │  │  (Region C)  │
│  10K users   │ │  10K users   │  │  10K users   │
└──────┬───────┘ └──────┬───────┘  └──────┬───────┘
       │                │                   │
       └────────────────┼───────────────────┘
                        │
                        ▼
               ┌──────────────────┐
               │  Target System   │
               │  (30K total VUs) │
               └──────────────────┘
```

---

## Code Examples

### k6 — Modern Load Testing (JavaScript)

```javascript
// load-test.js — k6 load test script
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const responseTime = new Trend('response_time');

// Test configuration — ramp up to 200 users over 5 minutes
export const options = {
  stages: [
    { duration: '2m', target: 50 },   // Ramp up to 50 users
    { duration: '5m', target: 200 },  // Ramp up to 200 users
    { duration: '3m', target: 200 },  // Stay at 200 users
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],  // SLA
    errors: ['rate<0.01'],  // Less than 1% errors
  },
};

export default function () {
  // Simulate a realistic user journey
  const baseUrl = 'http://localhost:8080';

  // Step 1: Get product listing
  let res = http.get(`${baseUrl}/api/products?page=1`);
  check(res, { 'products 200': (r) => r.status === 200 });
  responseTime.add(res.timings.duration);
  errorRate.add(res.status !== 200);

  sleep(1); // Think time — users don't click instantly

  // Step 2: View a product
  res = http.get(`${baseUrl}/api/products/42`);
  check(res, { 'product detail 200': (r) => r.status === 200 });

  sleep(2);

  // Step 3: Add to cart (POST)
  const payload = JSON.stringify({ productId: 42, quantity: 1 });
  res = http.post(`${baseUrl}/api/cart`, payload, {
    headers: { 'Content-Type': 'application/json' },
  });
  check(res, { 'add to cart 201': (r) => r.status === 201 });

  sleep(1);
}

// Run: k6 run load-test.js
// Run distributed: k6 run --out cloud load-test.js
```

### Python — Locust Load Testing

```python
# locustfile.py — Locust load test
from locust import HttpUser, task, between, events
import time

class WebsiteUser(HttpUser):
    """Simulates a user browsing an e-commerce site"""
    wait_time = between(1, 3)  # Random think time 1-3 seconds
    
    def on_start(self):
        """Called once per user — login"""
        self.client.post("/api/login", json={
            "email": "test@example.com",
            "password": "testpass123"
        })
    
    @task(3)  # Weight 3 — happens 3x more often
    def browse_products(self):
        """Browse product listing"""
        self.client.get("/api/products?page=1&limit=20")
    
    @task(2)  # Weight 2
    def view_product(self):
        """View a specific product"""
        product_id = 42  # In real test, pick randomly
        self.client.get(f"/api/products/{product_id}")
    
    @task(1)  # Weight 1 — least common
    def add_to_cart(self):
        """Add item to cart"""
        with self.client.post(
            "/api/cart",
            json={"productId": 42, "quantity": 1},
            catch_response=True
        ) as response:
            if response.status_code == 201:
                response.success()
            elif response.status_code == 429:
                response.failure("Rate limited!")
            else:
                response.failure(f"Unexpected: {response.status_code}")

# Custom event: print stats every 10 seconds
@events.request.add_listener
def on_request(request_type, name, response_time, **kwargs):
    if response_time > 500:
        print(f"⚠️  SLOW: {name} took {response_time}ms")

# Run: locust -f locustfile.py --host=http://localhost:8080
# Then open http://localhost:8089 for the web UI
```

### Java — JMeter-style with Gatling DSL

```java
// GatlingSimulation.java — Gatling load test (Scala DSL shown in Java-like pseudo)
// Gatling uses Scala, but here's the equivalent concept in Java using HttpClient

import java.net.http.*;
import java.net.URI;
import java.time.Duration;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class LoadTestSimulation {
    private static final AtomicInteger successCount = new AtomicInteger(0);
    private static final AtomicInteger errorCount = new AtomicInteger(0);
    private static final ConcurrentLinkedQueue<Long> latencies = new ConcurrentLinkedQueue<>();

    public static void main(String[] args) throws Exception {
        int totalUsers = 200;
        int rampUpSeconds = 60;
        int testDurationSeconds = 300;
        String targetUrl = "http://localhost:8080/api/products";

        HttpClient client = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(5))
            .build();

        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(totalUsers);
        
        // Ramp up users gradually
        for (int i = 0; i < totalUsers; i++) {
            long delayMs = (long) i * rampUpSeconds * 1000 / totalUsers;
            scheduler.schedule(() -> runVirtualUser(client, targetUrl, testDurationSeconds),
                delayMs, TimeUnit.MILLISECONDS);
        }

        // Wait for test to complete
        Thread.sleep((rampUpSeconds + testDurationSeconds) * 1000L);
        scheduler.shutdown();
        
        // Print results
        System.out.printf("Success: %d, Errors: %d, Error Rate: %.2f%%%n",
            successCount.get(), errorCount.get(),
            errorCount.get() * 100.0 / (successCount.get() + errorCount.get()));
    }

    private static void runVirtualUser(HttpClient client, String url, int durationSec) {
        long endTime = System.currentTimeMillis() + durationSec * 1000L;
        while (System.currentTimeMillis() < endTime) {
            try {
                long start = System.nanoTime();
                HttpRequest req = HttpRequest.newBuilder()
                    .uri(URI.create(url)).GET().build();
                HttpResponse<String> resp = client.send(req, 
                    HttpResponse.BodyHandlers.ofString());
                long latencyMs = (System.nanoTime() - start) / 1_000_000;
                latencies.add(latencyMs);
                
                if (resp.statusCode() == 200) successCount.incrementAndGet();
                else errorCount.incrementAndGet();
                
                Thread.sleep(ThreadLocalRandom.current().nextInt(1000, 3000));
            } catch (Exception e) {
                errorCount.incrementAndGet();
            }
        }
    }
}
```

---

## Infrastructure Examples

### Running k6 in CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/load-test.yml
name: Load Test
on:
  pull_request:
    branches: [main]

jobs:
  load-test:
    runs-on: ubuntu-latest
    services:
      app:
        image: my-app:${{ github.sha }}
        ports:
          - 8080:8080
    steps:
      - uses: actions/checkout@v4
      
      - name: Install k6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
            --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D68
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | \
            sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update && sudo apt-get install k6
      
      - name: Run Load Test
        run: k6 run tests/load-test.js
        
      - name: Upload Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: k6-results
          path: results/
```

### Distributed Locust on Kubernetes

```yaml
# locust-master.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: locust-master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: locust
      role: master
  template:
    metadata:
      labels:
        app: locust
        role: master
    spec:
      containers:
        - name: locust
          image: locustio/locust:latest
          args: ["--master", "--host=http://target-service:8080"]
          ports:
            - containerPort: 8089  # Web UI
            - containerPort: 5557  # Worker communication
          volumeMounts:
            - name: test-scripts
              mountPath: /mnt/locust
      volumes:
        - name: test-scripts
          configMap:
            name: locust-scripts
---
# locust-worker.yaml — scale to generate more load
apiVersion: apps/v1
kind: Deployment
metadata:
  name: locust-worker
spec:
  replicas: 10  # 10 workers — can scale to 100+
  selector:
    matchLabels:
      app: locust
      role: worker
  template:
    metadata:
      labels:
        app: locust
        role: worker
    spec:
      containers:
        - name: locust
          image: locustio/locust:latest
          args: ["--worker", "--master-host=locust-master"]
          volumeMounts:
            - name: test-scripts
              mountPath: /mnt/locust
      volumes:
        - name: test-scripts
          configMap:
            name: locust-scripts
```

---

## Real-World Example

### Amazon Prime Day — Load Testing at Scale

Before every Prime Day, Amazon runs load tests simulating **more traffic than they expect**:

```
Amazon Prime Day Preparation:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Expected peak: 100,000 RPS                            │
│  Test target:   200,000 RPS (2x safety margin)         │
│                                                         │
│  Test Infrastructure:                                   │
│  ┌──────────────┐                                      │
│  │ 500+ load    │──────▶ Production-like env           │
│  │ generators   │        (same DB, same services)      │
│  └──────────────┘                                      │
│                                                         │
│  Process:                                               │
│  1. GameDay exercise (team simulates failures)          │
│  2. Ramp to 2x expected peak                           │
│  3. Inject failures (kill servers, slow DB)            │
│  4. Verify auto-scaling kicks in                        │
│  5. Verify circuit breakers activate                    │
│  6. Verify fallbacks work                              │
│                                                         │
│  Result: Confident the system won't crash on the day   │
└─────────────────────────────────────────────────────────┘
```

### Netflix Chaos + Load Testing

Netflix combines load testing with chaos engineering:

```
1. Start load test (normal peak traffic)
2. Kill a random server instance
3. Verify: Does throughput drop? Do errors spike?
4. Verify: Does auto-scaling launch a replacement?
5. Kill an entire availability zone
6. Verify: Does traffic failover to other AZs?

Expected behavior:
  - Users experience ZERO downtime
  - p99 latency increases by <50ms
  - Recovery completes in <2 minutes
```

---

## Tool Comparison

```
┌──────────────┬──────────────┬──────────────┬──────────────┐
│   Feature    │     k6       │   Locust     │   JMeter     │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Language     │ JavaScript   │ Python       │ Java/XML     │
│ Protocol     │ HTTP,gRPC,WS │ HTTP,custom  │ HTTP,JDBC,   │
│              │              │              │ FTP,LDAP     │
│ GUI          │ No (CLI)     │ Web UI       │ Full GUI     │
│ Performance  │ Very high    │ Medium       │ Medium       │
│ Scripting    │ Excellent    │ Excellent    │ Limited      │
│ CI/CD        │ Excellent    │ Good         │ Possible     │
│ Distributed  │ Cloud/k8s    │ Master/Worker│ Remote JMeter│
│ Resource use │ Low (Go)     │ Medium       │ High (JVM)   │
│ Learning     │ Easy         │ Easy         │ Medium       │
│ Best for     │ Developers   │ Python teams │ QA teams     │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Testing from a single machine/network | Network becomes bottleneck, not the app | Use distributed load generators across regions |
| No think time between requests | Unrealistic — real users pause between clicks | Add 1-5 second random delays between actions |
| Ignoring ramp-up time | Slamming all users at once triggers rate limiters | Gradually ramp up over 2-5 minutes |
| Testing against production database | Could corrupt real data or hit rate limits | Use isolated test environment with realistic data |
| Not monitoring during tests | You see failures but don't know WHY | Monitor CPU, memory, DB connections, error logs |
| Closed-loop testing only | Hides latency issues (coordinated omission) | Use constant arrival rate (open-loop) in k6 |
| Testing once and never again | Performance regresses with every deploy | Add load tests to CI/CD pipeline |

---

## When to Use / When NOT to Use

### Run load tests WHEN:
- Before a major launch or marketing campaign
- After significant code changes (new features, refactors)
- When you've changed infrastructure (new DB, more servers)
- Regularly in CI/CD (catch regressions early)
- Before Black Friday, Prime Day, etc.

### Skip load tests WHEN:
- You haven't defined success criteria (what's "fast enough"?)
- The test environment is completely different from production
- You're testing a function that only runs once a day
- You need to test individual function performance (use profiling instead)

---

## Key Takeaways

- **Load testing** verifies performance under expected load; **stress testing** finds the breaking point
- Use **open-loop testing** (constant arrival rate) to avoid coordinated omission bias
- **k6** is best for developer workflows and CI/CD; **Locust** for Python teams; **JMeter** for GUI/protocol variety
- Always include **think time** (1-5s between actions) to simulate real users
- Run **distributed** load generators to avoid the test machine being the bottleneck
- Test at **2x your expected peak** to have safety margin
- Monitor **everything** during tests: application metrics, infrastructure, and database

---

## What's Next?

Now that you know how to test your system's performance, let's dive into one of the most common bottlenecks: the database. In **Chapter 19.4: Database Performance Tuning**, you'll learn query optimization, indexing strategies, and how to squeeze maximum performance from PostgreSQL, MySQL, and MongoDB.
