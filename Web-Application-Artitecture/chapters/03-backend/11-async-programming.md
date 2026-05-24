# Chapter 3.11: Async Programming — Non-Blocking I/O (Event Loop, Coroutines)

> **Level**: ⭐⭐⭐ Advanced  
> **What you'll learn**: How async/await works under the hood — event loops, coroutines, non-blocking I/O — and why this paradigm lets a single thread handle thousands of concurrent operations.

---

## 🧠 Real-Life Analogy: The Smart Waiter

```
    SYNCHRONOUS (Blocking) WAITER:
    ══════════════════════════════
    
    🧑‍🍳 Waiter takes Order 1 → walks to kitchen
    → stands at kitchen counter WAITING for food (10 minutes!)
    → food ready → delivers to Table 1
    → NOW takes Order 2 → walks to kitchen
    → WAITS again (10 minutes!)...
    
    10 tables × 10 minutes = 100 minutes to serve everyone!
    Waiter spends 90% of time STANDING and WAITING. 🤦
    
    
    ASYNC (Non-Blocking) WAITER:
    ════════════════════════════
    
    🧑‍🍳 Waiter takes Order 1 → sends to kitchen → DOESN'T WAIT!
    → Takes Order 2 → sends to kitchen → DOESN'T WAIT!
    → Takes Order 3 → sends to kitchen → DOESN'T WAIT!
    → 🔔 Kitchen bell: "Order 1 ready!" → delivers to Table 1
    → Takes Order 4 → sends to kitchen
    → 🔔 Kitchen bell: "Order 3 ready!" → delivers to Table 3
    ...
    
    10 tables served in ~15 minutes with ONE waiter!
    The waiter NEVER stands idle. Always doing useful work.
    
    
    ┌────────────────────────────────────────────────────┐
    │  SYNC:                                            │
    │  Waiter: [Order1][WAIT 10min][Deliver][Order2]... │
    │  Time:    ████████░░░░░░░░░░████████████░░░░░░░░  │
    │           work    idle      work       idle        │
    │                                                    │
    │  ASYNC:                                           │
    │  Waiter: [O1][O2][O3][D1][O4][D3][O5][D2][D4]... │
    │  Time:    ████████████████████████████████████████  │
    │           ALL useful work! No idle time!            │
    └────────────────────────────────────────────────────┘
```

---

## 📖 What is Non-Blocking I/O?

```
    Every time your code does something "outside" the CPU,
    it's doing I/O (Input/Output):
    
    ┌────────────────────────────────────────────────────────────┐
    │  Common I/O Operations and their latencies:               │
    │                                                            │
    │  ┌─────────────────────────────┬───────────────────────┐  │
    │  │  Operation                  │  Time                 │  │
    │  ├─────────────────────────────┼───────────────────────┤  │
    │  │  CPU: add two numbers       │  ~0.3 nanoseconds     │  │
    │  │  RAM: read from memory      │  ~100 nanoseconds     │  │
    │  │  SSD: read from disk        │  ~100 microseconds    │  │
    │  │  Network: DB query          │  ~1-100 milliseconds  │  │
    │  │  Network: API call          │  ~50-500 milliseconds │  │
    │  │  Network: slow API          │  ~1-5 seconds         │  │
    │  └─────────────────────────────┴───────────────────────┘  │
    │                                                            │
    │  CPU operations: nanoseconds (billionths of a second)     │
    │  I/O operations: milliseconds (thousandths of a second)   │
    │                                                            │
    │  I/O is 1,000,000× SLOWER than CPU operations!           │
    │                                                            │
    │  In BLOCKING I/O: your thread SLEEPS during that 50ms.   │
    │  In NON-BLOCKING I/O: your thread does OTHER WORK!       │
    └────────────────────────────────────────────────────────────┘
    
    
    BLOCKING vs NON-BLOCKING:
    ═════════════════════════
    
    Blocking (Synchronous):
    ┌──────┐  query()  ┌──────┐              ┌──────┐
    │ Code │──────────▶│  DB  │──── 50ms ───▶│ Code │ continues
    │      │  BLOCKS   │      │              │      │
    │ 💤  │  here!    │      │              │      │
    └──────┘           └──────┘              └──────┘
    Thread does NOTHING for 50ms. Wasted.
    
    Non-Blocking (Asynchronous):
    ┌──────┐  query()  ┌──────┐              ┌──────┐
    │ Code │──────────▶│  DB  │──── 50ms ───▶│ Code │ callback!
    │      │           │      │              │      │
    │ Does │ other     │      │              │      │
    │ work!│ tasks     │      │              │      │
    └──────┘           └──────┘              └──────┘
    Thread serves OTHER requests during those 50ms!
```

---

## 🔧 The Event Loop — Heart of Async Programming

```
    The EVENT LOOP is a single thread that continuously checks:
    "Is any I/O operation complete? If yes, run its callback."
    
    ┌──────────────────────────────────────────────────────────────┐
    │                     THE EVENT LOOP                           │
    │                                                              │
    │         ┌─────────────────────────────────┐                 │
    │         │                                 │                 │
    │         ▼                                 │                 │
    │  ┌─────────────┐    ┌─────────────┐      │                 │
    │  │ Check event │───▶│ Run callback│──────┘                 │
    │  │   queue     │    │ for ready   │                         │
    │  └─────────────┘    │ events      │                         │
    │                     └─────────────┘                         │
    │                                                              │
    │                                                              │
    │  DETAILED VIEW:                                             │
    │  ─────────────                                              │
    │                                                              │
    │  Event Queue:                                                │
    │  ┌────────────────────────────────────────┐                 │
    │  │ [DB result ready] [File read done]     │                 │
    │  │ [HTTP response] [Timer expired]        │                 │
    │  └────────────────────────────────────────┘                 │
    │         │                                                    │
    │         ▼                                                    │
    │  Event Loop picks one event:                                │
    │  "DB result ready for Request #42"                          │
    │         │                                                    │
    │         ▼                                                    │
    │  Runs the callback/continuation for Request #42:            │
    │  → Processes the DB result                                  │
    │  → Sends HTTP response to client                            │
    │  → Done! Picks next event from queue.                       │
    │                                                              │
    │                                                              │
    │  TIMELINE EXAMPLE (3 concurrent requests):                  │
    │  ═════════════════════════════════════════                   │
    │                                                              │
    │  Time →  0ms    5ms    10ms    50ms   55ms   100ms  105ms  │
    │                                                              │
    │  Loop:  [Parse] [Parse] [Parse]  [CB1]  [CB2]  [CB3] [idle]│
    │         Req 1   Req 2   Req 3   Req1   Req2   Req3         │
    │                                                              │
    │  DB:           [────── query 1 ──────]                      │
    │                       [────── query 2 ──────]               │
    │                              [────── query 3 ──────]        │
    │                                                              │
    │  One thread served 3 requests concurrently!                 │
    │  Total time: ~105ms (not 150ms = 3 × 50ms sequentially)   │
    └──────────────────────────────────────────────────────────────┘
```

---

## 📖 Evolution: Callbacks → Promises → Async/Await

```
    Async programming evolved through 3 generations:
    
    
    GEN 1: CALLBACKS (Callback Hell 🔥)
    ════════════════════════════════════
    
    getUser(userId, function(user) {
        getOrders(user.id, function(orders) {
            getPayment(orders[0].id, function(payment) {
                sendEmail(user.email, payment, function(result) {
                    console.log("Done!");
                });
            });
        });
    });
    
    ↪ Nested deeper and deeper = "Pyramid of Doom"
    ↪ Error handling is a nightmare
    ↪ Hard to read and maintain
    
    
    GEN 2: PROMISES / FUTURES
    ═════════════════════════
    
    getUser(userId)
        .then(user => getOrders(user.id))
        .then(orders => getPayment(orders[0].id))
        .then(payment => sendEmail(user.email, payment))
        .then(result => console.log("Done!"))
        .catch(error => console.error(error));
    
    ↪ Flat chain instead of nesting
    ↪ Centralized error handling
    ↪ Still somewhat awkward to read
    
    
    GEN 3: ASYNC/AWAIT (Modern Standard ✅)
    ═══════════════════════════════════════
    
    async function processOrder(userId) {
        const user = await getUser(userId);
        const orders = await getOrders(user.id);
        const payment = await getPayment(orders[0].id);
        const result = await sendEmail(user.email, payment);
        console.log("Done!");
    }
    
    ↪ Reads like synchronous code!
    ↪ But runs ASYNCHRONOUSLY under the hood!
    ↪ Easy error handling with try/catch
    ↪ This is what Python, Java, JavaScript all use today
```

---

## 💻 Code Examples

### Python — Async with asyncio (Deep Dive)

```python
"""
Async programming in Python with asyncio.
Shows how one thread handles many operations concurrently.
"""
import asyncio
import aiohttp  # Async HTTP client
import time

# ─── Understanding coroutines ───

async def fetch_data(url, delay_name):
    """A coroutine — looks sync, runs async!"""
    print(f"  Starting {delay_name}...")
    
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            data = await response.text()  # Non-blocking!
            # While waiting for this response,
            # the event loop runs OTHER coroutines!
            print(f"  Finished {delay_name}: {len(data)} bytes")
            return data

async def main():
    """Run multiple I/O operations concurrently."""
    
    urls = [
        ("https://httpbin.org/delay/2", "API Call 1"),
        ("https://httpbin.org/delay/2", "API Call 2"),
        ("https://httpbin.org/delay/2", "API Call 3"),
    ]
    
    # ── WRONG WAY: Sequential (one after another) ──
    print("Sequential (slow):")
    start = time.time()
    for url, name in urls:
        await fetch_data(url, name)  # Waits for each one!
    print(f"Total: {time.time() - start:.1f}s\n")  # ~6 seconds
    
    # ── RIGHT WAY: Concurrent (all at once!) ──
    print("Concurrent (fast):")
    start = time.time()
    tasks = [fetch_data(url, name) for url, name in urls]
    results = await asyncio.gather(*tasks)  # All run together!
    print(f"Total: {time.time() - start:.1f}s")  # ~2 seconds!

asyncio.run(main())

# Output:
# Sequential (slow):
#   Starting API Call 1... Finished (2s)
#   Starting API Call 2... Finished (2s)
#   Starting API Call 3... Finished (2s)
#   Total: 6.1s
#
# Concurrent (fast):
#   Starting API Call 1...
#   Starting API Call 2...   ← All start immediately!
#   Starting API Call 3...
#   Finished API Call 2...
#   Finished API Call 1...   ← All finish around same time!
#   Finished API Call 3...
#   Total: 2.1s              ← 3x faster!
```

### Python — Async Web Server (FastAPI)

```python
"""
FastAPI — modern async Python web framework.
Handles thousands of concurrent requests with one process.
"""
from fastapi import FastAPI
import asyncio
import httpx  # Async HTTP client

app = FastAPI()

@app.get("/sync-endpoint")
def sync_handler():
    """SYNC handler — blocks the thread during I/O!"""
    import requests
    # This BLOCKS the worker thread for 2 seconds!
    response = requests.get("https://httpbin.org/delay/2")
    return {"data": response.json()}

@app.get("/async-endpoint")
async def async_handler():
    """ASYNC handler — doesn't block! Other requests served!"""
    async with httpx.AsyncClient() as client:
        # This is NON-BLOCKING! Event loop is free!
        response = await client.get("https://httpbin.org/delay/2")
    return {"data": response.json()}

@app.get("/parallel-calls")
async def parallel_handler():
    """Make multiple external calls in PARALLEL."""
    async with httpx.AsyncClient() as client:
        # Fire all 3 requests at once!
        tasks = [
            client.get("https://httpbin.org/delay/1"),
            client.get("https://httpbin.org/delay/1"),
            client.get("https://httpbin.org/delay/1"),
        ]
        responses = await asyncio.gather(*tasks)
        
    # All 3 completed in ~1 second, not 3 seconds!
    return {"results": [r.status_code for r in responses]}

# Run: uvicorn app:app --workers 4
# 4 workers × event loop = handles 10K+ concurrent connections!
```

### Java — Async with CompletableFuture

```java
/**
 * Async programming in Java with CompletableFuture.
 * Non-blocking composition of async operations.
 */
@Service
public class AsyncOrderService {

    @Autowired
    private WebClient webClient;  // Non-blocking HTTP client

    public CompletableFuture<OrderDetails> getOrderDetails(Long orderId) {
        
        // Fire ALL three requests in PARALLEL!
        CompletableFuture<Order> orderFuture = 
            CompletableFuture.supplyAsync(() -> fetchOrder(orderId));
        
        CompletableFuture<User> userFuture = 
            orderFuture.thenComposeAsync(order ->
                CompletableFuture.supplyAsync(
                    () -> fetchUser(order.getUserId())));
        
        CompletableFuture<Payment> paymentFuture = 
            orderFuture.thenComposeAsync(order ->
                CompletableFuture.supplyAsync(
                    () -> fetchPayment(order.getPaymentId())));

        // Combine results when ALL are done
        return orderFuture.thenCombine(userFuture, (order, user) -> {
            return paymentFuture.thenApply(payment -> 
                new OrderDetails(order, user, payment));
        }).thenCompose(f -> f);
    }
}

// Modern Java (21+) with Virtual Threads — much simpler!
@Service 
public class ModernOrderService {
    
    public OrderDetails getOrderDetails(Long orderId) 
            throws Exception {
        
        // With virtual threads, just write BLOCKING code!
        // The JVM makes it non-blocking automatically!
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            var orderFuture = executor.submit(() -> fetchOrder(orderId));
            Order order = orderFuture.get();
            
            // These run in parallel on virtual threads
            var userFuture = executor.submit(
                () -> fetchUser(order.getUserId()));
            var paymentFuture = executor.submit(
                () -> fetchPayment(order.getPaymentId()));
            
            return new OrderDetails(
                order, userFuture.get(), paymentFuture.get()
            );
        }
    }
}
```

### Java — Reactive with Spring WebFlux

```java
/**
 * Spring WebFlux — fully reactive (non-blocking) web framework.
 * Uses Reactor's Mono (single value) and Flux (stream of values).
 */
@RestController
public class ReactiveController {

    @Autowired
    private WebClient webClient;

    @GetMapping("/api/dashboard")
    public Mono<DashboardResponse> getDashboard(@RequestParam Long userId) {
        
        // All three API calls execute in PARALLEL!
        Mono<User> userMono = webClient.get()
            .uri("/users/{id}", userId)
            .retrieve()
            .bodyToMono(User.class);
        
        Mono<List<Order>> ordersMono = webClient.get()
            .uri("/orders?userId={id}", userId)
            .retrieve()
            .bodyToFlux(Order.class)
            .collectList();
        
        Mono<List<Notification>> notifMono = webClient.get()
            .uri("/notifications?userId={id}", userId)
            .retrieve()
            .bodyToFlux(Notification.class)
            .collectList();

        // Combine when all complete — total time = max of the three
        return Mono.zip(userMono, ordersMono, notifMono)
            .map(tuple -> new DashboardResponse(
                tuple.getT1(),   // user
                tuple.getT2(),   // orders  
                tuple.getT3()    // notifications
            ));
    }
}
// 3 API calls × 100ms each:
// Sequential: 300ms
// Parallel (above): 100ms! 3x faster!
```

---

## 🔄 When Async Helps vs When It Doesn't

```
    ┌──────────────────────────────────────────────────────────────┐
    │  ASYNC IS GREAT FOR (I/O-bound):                           │
    │                                                              │
    │  ✅ Database queries (waiting for DB response)              │
    │  ✅ HTTP API calls (waiting for external service)           │
    │  ✅ File I/O (waiting for disk read/write)                  │
    │  ✅ WebSocket connections (many idle connections)            │
    │  ✅ Email sending (waiting for SMTP server)                 │
    │  ✅ Cache lookups (waiting for Redis response)              │
    │                                                              │
    │  → The thread is mostly WAITING. Async lets it do          │
    │    useful work instead of sleeping!                          │
    │                                                              │
    │                                                              │
    │  ASYNC DOESN'T HELP FOR (CPU-bound):                       │
    │                                                              │
    │  ❌ Image/video processing (CPU is computing)               │
    │  ❌ Cryptography / hashing (CPU is computing)               │
    │  ❌ Machine learning inference (CPU/GPU is computing)       │
    │  ❌ Large JSON parsing (CPU is computing)                   │
    │  ❌ Compression (CPU is computing)                          │
    │                                                              │
    │  → The CPU is 100% busy. There's nothing to "not block."  │
    │    Use threads/processes/parallelism instead.               │
    └──────────────────────────────────────────────────────────────┘
    
    
    VISUAL PROOF:
    ═════════════
    
    I/O-bound task (DB query):
    Thread: [Send query] [💤 WAITING 50ms 💤] [Process result]
                          ▲
                          └── Async saves this time!
    
    CPU-bound task (image resize):
    Thread: [🔥 COMPUTING 50ms 🔥🔥🔥🔥🔥🔥🔥🔥]
                          ▲
                          └── No waiting time to save!
                              Async doesn't help here.
```

---

## 📊 Sync vs Async Performance Comparison

```
    BENCHMARK: 1000 requests to server, each makes a 50ms DB query
    
    ┌──────────────────────┬───────────┬───────────┬─────────────┐
    │  Server Config       │  Threads  │  Time     │  Throughput │
    ├──────────────────────┼───────────┼───────────┼─────────────┤
    │  Flask (sync)        │  1        │  50,000ms │  20 req/s   │
    │  4 workers           │           │  = 50s    │             │
    ├──────────────────────┼───────────┼───────────┼─────────────┤
    │  Flask (sync)        │  4        │  12,500ms │  80 req/s   │
    │  4 workers           │           │  = 12.5s  │             │
    ├──────────────────────┼───────────┼───────────┼─────────────┤
    │  Gunicorn            │  16       │  3,125ms  │  320 req/s  │
    │  4 workers × 4 thr   │           │  = 3.1s   │             │
    ├──────────────────────┼───────────┼───────────┼─────────────┤
    │  FastAPI (async)     │  1 event  │  ~55ms    │  18,000+    │
    │  1 worker            │  loop     │  = 0.05s! │  req/s!     │
    ├──────────────────────┼───────────┼───────────┼─────────────┤
    │  FastAPI (async)     │  4 event  │  ~55ms    │  72,000+    │
    │  4 workers           │  loops    │  = 0.05s! │  req/s!     │
    └──────────────────────┴───────────┴───────────┴─────────────┘
    
    Async is 225x more throughput with FEWER resources!
    (For I/O-bound workloads. CPU-bound would be similar.)
```

---

## 🏢 Real-World Examples

```
    ┌──────────────────┬───────────────────┬─────────────────────────┐
    │  Company         │  Async Framework  │  Why Async?             │
    ├──────────────────┼───────────────────┼─────────────────────────┤
    │  Instagram       │  Django (async    │  Millions of API calls  │
    │                  │  views in Django  │  per second, I/O heavy  │
    │                  │  4.1+)            │                         │
    ├──────────────────┼───────────────────┼─────────────────────────┤
    │  Netflix         │  Spring WebFlux   │  Zuul gateway handles   │
    │                  │  (Reactor/Netty)  │  100K+ concurrent conn  │
    ├──────────────────┼───────────────────┼─────────────────────────┤
    │  Discord         │  Tokio (Rust      │  Millions of WS conns   │
    │                  │  async runtime)   │  with minimal servers   │
    ├──────────────────┼───────────────────┼─────────────────────────┤
    │  Uber            │  Node.js +        │  Real-time location     │
    │                  │  Go goroutines    │  tracking, high I/O     │
    ├──────────────────┼───────────────────┼─────────────────────────┤
    │  LinkedIn        │  Async Servlet    │  Feed generation makes  │
    │                  │  (Java)           │  many parallel API calls│
    ├──────────────────┼───────────────────┼─────────────────────────┤
    │  FastAPI (Python) │  asyncio +       │  Modern Python APIs     │
    │  projects        │  uvicorn          │  competing with Go/Node │
    └──────────────────┴───────────────────┴─────────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Mixing sync and async code incorrectly
       async def handler():
           data = requests.get(url)  # BLOCKING! Defeats async!
       ✅ Use async libraries: httpx, aiohttp, asyncpg (not requests)
    
    ❌ Blocking the event loop with CPU work
       async def handler():
           result = heavy_computation()  # Blocks ALL other requests!
       ✅ Offload CPU work: await asyncio.to_thread(heavy_computation)
    
    ❌ Creating too many concurrent connections
       tasks = [fetch(url) for url in 100000_urls]
       await asyncio.gather(*tasks)  # 100K connections at once! 💥
       ✅ Use asyncio.Semaphore to limit concurrency:
       semaphore = asyncio.Semaphore(100)  # Max 100 at a time
    
    ❌ Not handling async exceptions properly
       → Unhandled exceptions in coroutines are silently swallowed!
       ✅ Always use try/except inside coroutines and gather
    
    ❌ Using async when you don't need it
       → Simple CRUD app with 10 users doesn't need async
       → Adds complexity with no benefit
       ✅ Use async when you have high concurrency + I/O-heavy work
    
    ❌ Forgetting that async is NOT parallel
       → async runs on ONE thread. It's concurrent, not parallel.
       → For CPU parallelism, use multiprocessing or thread pools.
       ✅ Combine async (I/O) with multiprocessing (CPU) for max perf
```

---

## 🗺️ The Async Landscape Across Languages

```
    ┌──────────────────┬────────────────────┬──────────────────────┐
    │  Language        │  Async Runtime     │  Syntax              │
    ├──────────────────┼────────────────────┼──────────────────────┤
    │  Python          │  asyncio           │  async def / await   │
    │                  │  (uvloop for speed)│                      │
    ├──────────────────┼────────────────────┼──────────────────────┤
    │  JavaScript      │  V8 event loop     │  async / await       │
    │  (Node.js)       │  (libuv)           │  (or Promises)       │
    ├──────────────────┼────────────────────┼──────────────────────┤
    │  Java            │  Virtual Threads   │  Thread.ofVirtual()  │
    │  (21+)           │  (Project Loom)    │  or CompletableFuture│
    ├──────────────────┼────────────────────┼──────────────────────┤
    │  Java            │  Reactor / RxJava  │  Mono / Flux         │
    │  (Reactive)      │  (Netty event loop)│  .map() .flatMap()   │
    ├──────────────────┼────────────────────┼──────────────────────┤
    │  Go              │  Goroutines        │  go func() { ... }   │
    │                  │  (built-in!)       │  channels for comm.  │
    ├──────────────────┼────────────────────┼──────────────────────┤
    │  Rust            │  Tokio / async-std │  async fn / .await   │
    ├──────────────────┼────────────────────┼──────────────────────┤
    │  Kotlin          │  Coroutines        │  suspend fun / launch│
    │                  │  (structured)      │  withContext()       │
    ├──────────────────┼────────────────────┼──────────────────────┤
    │  C#              │  Task-based (TPL)  │  async / await       │
    │  (.NET)          │                    │  Task<T>             │
    └──────────────────┴────────────────────┴──────────────────────┘
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. Sync code BLOCKS the thread during I/O (50ms of sleeping).     ║
║     Async code RELEASES the thread during I/O (does other work).   ║
║                                                                      ║
║  2. The event loop is a single thread that multiplexes thousands   ║
║     of I/O operations. It never sleeps — always doing useful work. ║
║                                                                      ║
║  3. async/await is syntactic sugar that makes async code read      ║
║     like synchronous code. It's the modern standard everywhere.    ║
║                                                                      ║
║  4. Async is game-changing for I/O-bound workloads (APIs, DBs).   ║
║     It does NOT help for CPU-bound workloads (math, compression).  ║
║                                                                      ║
║  5. Always use async-compatible libraries. One blocking call       ║
║     in async code blocks ALL concurrent operations.                 ║
║                                                                      ║
║  6. Java 21 Virtual Threads make blocking code non-blocking        ║
║     automatically — the best of both worlds. This is the future.   ║
║                                                                      ║
║  7. Don't use async for simple apps with low traffic. It adds     ║
║     complexity. Use it when concurrency + I/O is your bottleneck.  ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

Congratulations! You've completed Part 3: Backend Fundamentals! 🎉

You now understand:
- How servers work (web servers, app servers, reverse proxies)
- Communication protocols (REST, GraphQL, gRPC, WebSocket, SSE)
- How servers handle load (threads, concurrency, parallelism)
- Performance essentials (connection pooling, async programming)

Next up: **Part 4 — Databases & Storage** — where we'll learn how data is stored, queried, and scaled. [Go to Part 4: Databases](../04-databases/).

---

[⬅️ Previous: Connection Pooling](./10-connection-pooling.md) | [⬆️ Index](../../00-INDEX.md) | [Next: Part 4 — Databases ➡️](../04-databases/)
