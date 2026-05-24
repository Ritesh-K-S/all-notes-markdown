# Chapter 3.8: Concurrency & Parallelism — Serving Thousands at Once

> **Level**: ⭐⭐⭐ Advanced  
> **What you'll learn**: The fundamental difference between concurrency and parallelism, how operating systems manage multiple tasks, and why this matters for building high-performance servers.

---

## 🧠 Real-Life Analogy: One Cook vs Many Cooks

```
    CONCURRENCY = One cook, many dishes (multitasking)
    ═══════════════════════════════════════════════════
    
    One cook has 3 dishes to prepare:
    
    🧑‍🍳 Start boiling pasta → while it boils...
    🧑‍🍳 Start chopping vegetables → while they cook...
    🧑‍🍳 Start grilling chicken → while it grills...
    🧑‍🍳 Check pasta → it's done! Drain it.
    🧑‍🍳 Check veggies → done! Plate them.
    🧑‍🍳 Check chicken → done! Serve it.
    
    ONE person, THREE dishes, done in ~15 minutes.
    Not at the same time — but INTERLEAVED cleverly.
    
    
    PARALLELISM = Three cooks, each on one dish (simultaneous)
    ═════════════════════════════════════════════════════════
    
    🧑‍🍳 Cook 1: Making pasta         → Done in 15 min
    🧑‍🍳 Cook 2: Making vegetables    → Done in 12 min
    🧑‍🍳 Cook 3: Grilling chicken     → Done in 10 min
    
    THREE people, THREE dishes, ALL done in ~15 minutes.
    Actually happening AT THE SAME TIME.
    
    
    KEY INSIGHT:
    ════════════
    Concurrency = DEALING with many things at once (structure)
    Parallelism  = DOING many things at once (execution)
    
    You can have concurrency WITHOUT parallelism!
    (One cook switching between tasks on a single stove)
    
    You can have parallelism WITHOUT concurrency!
    (Three cooks each doing only ONE task)
    
    Best: Concurrency WITH Parallelism!
    (Three cooks, each managing multiple dishes)
```

---

## 📖 Concurrency vs Parallelism — Visualized

```
    CONCURRENCY (Single CPU Core):
    ══════════════════════════════
    
    Time ─────────────────────────────────────────────▶
    
    CPU:  ┃ T1 ┃ T2 ┃ T1 ┃ T3 ┃ T2 ┃ T1 ┃ T3 ┃ T2 ┃
    
    ONE core rapidly switches between Task 1, 2, and 3.
    LOOKS like they're running at the same time.
    Actually taking turns (time-slicing / context switching).
    
    
    PARALLELISM (Multiple CPU Cores):
    ═════════════════════════════════
    
    Time ─────────────────────────────────────────────▶
    
    Core 1:  ┃ Task 1 ┃ Task 1 ┃ Task 1 ┃ Task 1 ┃
    Core 2:  ┃ Task 2 ┃ Task 2 ┃ Task 2 ┃ Task 2 ┃
    Core 3:  ┃ Task 3 ┃ Task 3 ┃ Task 3 ┃ Task 3 ┃
    
    THREE cores running three tasks TRULY simultaneously.
    
    
    CONCURRENCY + PARALLELISM (Real Production):
    ════════════════════════════════════════════
    
    Time ─────────────────────────────────────────────▶
    
    Core 1:  ┃ T1 ┃ T4 ┃ T1 ┃ T7 ┃ T4 ┃ T1 ┃
    Core 2:  ┃ T2 ┃ T5 ┃ T2 ┃ T8 ┃ T5 ┃ T2 ┃
    Core 3:  ┃ T3 ┃ T6 ┃ T3 ┃ T9 ┃ T6 ┃ T3 ┃
    
    Multiple cores, each handling multiple tasks concurrently.
    9 tasks across 3 cores = both concurrent AND parallel!
```

---

## 🔧 How It Works Internally

### Processes vs Threads vs Coroutines

```
    ┌──────────────────────────────────────────────────────────────┐
    │                    PROCESS                                   │
    │  ┌──────────────────────────────────────────────────────┐   │
    │  │  Own memory space (isolated)                         │   │
    │  │  Own file descriptors, network sockets               │   │
    │  │  Heavy to create (~10-100ms)                        │   │
    │  │  1-100 MB of memory                                 │   │
    │  │                                                      │   │
    │  │  ┌──────────────────────────────────────────────┐   │   │
    │  │  │  THREAD 1          THREAD 2         THREAD 3 │   │   │
    │  │  │  ┌────────────┐   ┌────────────┐  ┌────────┐│   │   │
    │  │  │  │ Own stack  │   │ Own stack  │  │Own stack││   │   │
    │  │  │  │ (1-8 MB)   │   │ (1-8 MB)   │  │(1-8 MB)││   │   │
    │  │  │  └────────────┘   └────────────┘  └────────┘│   │   │
    │  │  │                                              │   │   │
    │  │  │  SHARED: Heap memory, code, file descriptors │   │   │
    │  │  │  Create: ~100μs each                        │   │   │
    │  │  └──────────────────────────────────────────────┘   │   │
    │  └──────────────────────────────────────────────────────┘   │
    │                                                              │
    │  Coroutines/Green Threads (inside a thread):                │
    │  ┌──────────────────────────────────────────────────────┐   │
    │  │  Coroutine 1  Coroutine 2  Coroutine 3  ... 100,000 │   │
    │  │  (~1 KB each) (~1 KB each) (~1 KB each)              │   │
    │  │                                                       │   │
    │  │  Managed by runtime, NOT by OS                       │   │
    │  │  Create: ~1μs each (100x faster than threads!)      │   │
    │  │  Cooperative scheduling (yield voluntarily)           │   │
    │  └──────────────────────────────────────────────────────┘   │
    └──────────────────────────────────────────────────────────────┘
    
    
    COMPARISON:
    ═══════════
    
    ┌──────────────┬────────────┬────────────┬──────────────┐
    │              │  Process   │  Thread    │  Coroutine   │
    ├──────────────┼────────────┼────────────┼──────────────┤
    │  Memory      │  100+ MB   │  1-8 MB    │  ~1 KB       │
    ├──────────────┼────────────┼────────────┼──────────────┤
    │  Create time │  10-100ms  │  ~100μs    │  ~1μs        │
    ├──────────────┼────────────┼────────────┼──────────────┤
    │  Isolation   │  Full      │  Shared    │  Shared      │
    │              │  (safe)    │  memory    │  memory      │
    ├──────────────┼────────────┼────────────┼──────────────┤
    │  Max count   │  ~100s     │  ~1000s    │  ~1,000,000  │
    ├──────────────┼────────────┼────────────┼──────────────┤
    │  Managed by  │  OS        │  OS        │  Runtime     │
    ├──────────────┼────────────┼────────────┼──────────────┤
    │  Used in     │  Gunicorn  │  Java/C++  │  Python      │
    │              │  Nginx     │  Tomcat    │  asyncio,    │
    │              │  PHP-FPM   │  .NET      │  Go routines │
    │              │            │            │  Kotlin      │
    └──────────────┴────────────┴────────────┴──────────────┘
```

### Context Switching — The Hidden Cost

```
    When the OS switches from one thread to another:
    
    ┌──────────────────────────────────────────────────────────────┐
    │  CONTEXT SWITCH (Thread A → Thread B):                      │
    │                                                              │
    │  1. Save Thread A's state:                                  │
    │     - CPU registers (program counter, stack pointer)        │
    │     - Stack pointer                                         │
    │     - Memory mappings                                       │
    │                                                              │
    │  2. Store state in Thread Control Block (TCB)               │
    │                                                              │
    │  3. Load Thread B's state:                                  │
    │     - Restore registers                                     │
    │     - Restore stack pointer                                 │
    │     - Restore memory mappings                               │
    │                                                              │
    │  4. CPU cache is now INVALID! (cache miss = SLOW)           │
    │                                                              │
    │  Cost per context switch: ~1-10 microseconds                │
    │  With 10,000 threads: 10K switches/sec = 10-100ms wasted!  │
    │                                                              │
    │  This is why TOO MANY threads = SLOWER, not faster!        │
    └──────────────────────────────────────────────────────────────┘
    
    
    ┌──────────────────────────────────────────────────┐
    │           Thread count vs Performance            │
    │                                                  │
    │  Performance                                     │
    │  ▲                                               │
    │  │        ╱╲                                     │
    │  │      ╱    ╲                                   │
    │  │    ╱        ╲                                 │
    │  │  ╱            ╲                               │
    │  │╱                ╲────────────────             │
    │  ├─────┬─────┬─────┬─────┬─────────▶ Threads   │
    │  │     │     │     │     │                       │
    │  0    50   100   200   500                       │
    │                                                  │
    │  Sweet spot: ~100-200 threads (depends on CPU)  │
    │  After that: context switching kills performance │
    └──────────────────────────────────────────────────┘
```

---

## 💻 Code Examples

### Python — Concurrency with Threads

```python
"""
Demonstrating concurrency with Python threads.
Threads are great for I/O-bound tasks (network, disk, DB).
"""
import threading
import time
import requests

def fetch_url(url):
    """Fetch a URL — I/O bound task."""
    start = time.time()
    response = requests.get(url, timeout=10)
    elapsed = time.time() - start
    print(f"  {url}: {response.status_code} ({elapsed:.2f}s)")

urls = [
    "https://httpbin.org/delay/1",  # 1 second delay
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
]

# ── Sequential (no concurrency) ──
print("Sequential:")
start = time.time()
for url in urls:
    fetch_url(url)
print(f"Total: {time.time() - start:.2f}s")  # ~4 seconds!

# ── Concurrent with threads ──
print("\nConcurrent (threads):")
start = time.time()
threads = [threading.Thread(target=fetch_url, args=(url,)) for url in urls]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(f"Total: {time.time() - start:.2f}s")  # ~1 second! (all parallel)
```

### Python — Parallelism with Multiprocessing

```python
"""
True parallelism with Python multiprocessing.
Processes run on SEPARATE CPU cores — true simultaneous execution.
"""
import multiprocessing
import time
import math

def cpu_heavy_task(n):
    """CPU-bound task — calculates prime numbers."""
    count = 0
    for num in range(2, n):
        if all(num % i != 0 for i in range(2, int(math.sqrt(num)) + 1)):
            count += 1
    return count

numbers = [100000, 100000, 100000, 100000]

# ── Sequential (one core) ──
print("Sequential (1 core):")
start = time.time()
results = [cpu_heavy_task(n) for n in numbers]
print(f"Total: {time.time() - start:.2f}s")  # ~12 seconds

# ── Parallel (multiple cores) ──
print("\nParallel (4 cores):")
start = time.time()
with multiprocessing.Pool(processes=4) as pool:
    results = pool.map(cpu_heavy_task, numbers)
print(f"Total: {time.time() - start:.2f}s")  # ~3 seconds! (4x faster)

# WHY NOT USE THREADS FOR CPU WORK IN PYTHON?
# Python has the GIL (Global Interpreter Lock) —
# only ONE thread can execute Python code at a time!
# Threads help for I/O (waiting), but NOT for CPU (computing).
# Use multiprocessing for CPU-bound tasks in Python.
```

### Java — Concurrency with ExecutorService

```java
/**
 * Java concurrency using ExecutorService (thread pool).
 * Java has TRUE parallelism — no GIL like Python!
 */
import java.util.concurrent.*;
import java.util.List;

public class ConcurrencyDemo {

    public static void main(String[] args) throws Exception {
        // Create a thread pool with 4 threads
        ExecutorService executor = Executors.newFixedThreadPool(4);

        // Submit 10 tasks to run concurrently
        List<Future<String>> futures = new java.util.ArrayList<>();
        
        for (int i = 1; i <= 10; i++) {
            final int taskId = i;
            Future<String> future = executor.submit(() -> {
                String thread = Thread.currentThread().getName();
                System.out.println("Task " + taskId + " on " + thread);
                
                // Simulate I/O work
                Thread.sleep(1000);
                return "Task " + taskId + " done";
            });
            futures.add(future);
        }

        // Collect results
        for (Future<String> f : futures) {
            System.out.println(f.get());  // Blocks until done
        }

        executor.shutdown();
        
        // 10 tasks × 1 second each, but 4 threads:
        // Batch 1 (tasks 1-4): 1 second
        // Batch 2 (tasks 5-8): 1 second  
        // Batch 3 (tasks 9-10): 1 second
        // Total: ~3 seconds instead of 10!
    }
}
```

### Java — Virtual Threads (Java 21+ — Game Changer!)

```java
/**
 * Java 21 Virtual Threads — lightweight threads (like coroutines).
 * Can create MILLIONS of them! (vs ~thousands for platform threads)
 */
import java.util.concurrent.*;

public class VirtualThreadDemo {

    public static void main(String[] args) throws Exception {
        
        // OLD way: Platform threads (heavy, limited)
        // ExecutorService exec = Executors.newFixedThreadPool(200);
        
        // NEW way (Java 21+): Virtual threads (lightweight!)
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            // Create 100,000 tasks — each gets its own virtual thread!
            for (int i = 0; i < 100_000; i++) {
                final int taskId = i;
                executor.submit(() -> {
                    // Each virtual thread uses ~1 KB (not 1 MB!)
                    Thread.sleep(1000);  // Doesn't block OS thread!
                    if (taskId % 10000 == 0) {
                        System.out.println("Task " + taskId + " done on " +
                            Thread.currentThread());
                    }
                    return null;
                });
            }
        }
        // 100,000 concurrent tasks with minimal memory!
        // Platform threads: 100,000 × 1MB = 100 GB RAM needed!
        // Virtual threads:  100,000 × 1KB = 100 MB RAM needed!
    }
}
```

---

## 📊 I/O-Bound vs CPU-Bound — Choosing the Right Approach

```
    ┌────────────────────────────────────────────────────────────┐
    │              What is your task doing?                      │
    │                                                            │
    │  I/O-Bound (WAITING for external things):                 │
    │  ├── Network calls (HTTP, DB queries, API calls)          │
    │  ├── Disk reads/writes (file operations)                  │
    │  ├── Waiting for user input                               │
    │  └── DNS lookups, email sending                           │
    │                                                            │
    │  CPU-Bound (COMPUTING with the processor):                │
    │  ├── Math calculations, encryption                        │
    │  ├── Image/video processing                               │
    │  ├── JSON parsing large payloads                          │
    │  ├── Machine learning inference                           │
    │  └── Compression, sorting large datasets                  │
    └────────────────────────────────────────────────────────────┘
    
    
    DECISION GUIDE:
    ═══════════════
    
    ┌──────────────────┬──────────────────┬──────────────────────┐
    │  Task Type       │  Best Approach   │  Why                 │
    ├──────────────────┼──────────────────┼──────────────────────┤
    │  I/O-Bound       │  Async/Await     │  Thread sleeps during│
    │  (Python)        │  (asyncio)       │  I/O → waste. Use    │
    │                  │  or Threads      │  event loop instead. │
    ├──────────────────┼──────────────────┼──────────────────────┤
    │  I/O-Bound       │  Virtual Threads │  Lightweight, can    │
    │  (Java)          │  (Java 21+) or   │  have millions.      │
    │                  │  CompletableFuture│                      │
    ├──────────────────┼──────────────────┼──────────────────────┤
    │  CPU-Bound       │  Multiprocessing │  Python GIL blocks   │
    │  (Python)        │  (separate       │  threads from true   │
    │                  │  processes)      │  parallelism.        │
    ├──────────────────┼──────────────────┼──────────────────────┤
    │  CPU-Bound       │  Thread Pool     │  Java threads have   │
    │  (Java)          │  (ForkJoinPool)  │  true parallelism    │
    │                  │                  │  across cores.       │
    ├──────────────────┼──────────────────┼──────────────────────┤
    │  Mixed           │  Hybrid: Async   │  Async for I/O,      │
    │                  │  I/O + Thread    │  offload CPU work to │
    │                  │  pool for CPU    │  separate threads.   │
    └──────────────────┴──────────────────┴──────────────────────┘
```

---

## 🏗️ Python's GIL — The Elephant in the Room

```
    ┌──────────────────────────────────────────────────────────────┐
    │  PYTHON'S GIL (Global Interpreter Lock)                     │
    │                                                              │
    │  Python has a lock that allows only ONE thread to execute   │
    │  Python bytecode at a time, even on multi-core CPUs!       │
    │                                                              │
    │  4-Core CPU with Python threads:                            │
    │                                                              │
    │  Core 1: [Python Thread 1 running]                          │
    │  Core 2: [Thread 2 BLOCKED by GIL]  ← Wasted!             │
    │  Core 3: [Thread 3 BLOCKED by GIL]  ← Wasted!             │
    │  Core 4: [Thread 4 BLOCKED by GIL]  ← Wasted!             │
    │                                                              │
    │  HOWEVER, GIL is RELEASED during I/O operations!           │
    │                                                              │
    │  Thread 1: [Python] [I/O wait - GIL released] [Python]     │
    │  Thread 2: [BLOCKED] [Python runs!] [BLOCKED]               │
    │                                                              │
    │  So threads WORK for I/O-bound, NOT for CPU-bound!         │
    │                                                              │
    │  Solutions:                                                  │
    │  1. multiprocessing — separate processes (each has own GIL)│
    │  2. C extensions — NumPy releases GIL for math operations  │
    │  3. Python 3.13+ — experimental "free-threaded" mode!      │
    └──────────────────────────────────────────────────────────────┘
```

---

## 🏢 Real-World Architecture Patterns

```
    GOOGLE SEARCH (handles 99,000 queries/second):
    ═══════════════════════════════════════════════
    
    Query: "best pizza near me"
    
    ┌──────────────────────────────────────────────────────┐
    │  Fan-Out Pattern (parallel processing):             │
    │                                                      │
    │  Query ──▶ Load Balancer                            │
    │                 │                                    │
    │     ┌───────────┼───────────┐  All run              │
    │     ▼           ▼           ▼  in PARALLEL!         │
    │  ┌───────┐  ┌───────┐  ┌───────┐                   │
    │  │Web    │  │Maps   │  │ Ads   │                    │
    │  │Index  │  │Search │  │Engine │                    │
    │  └───┬───┘  └───┬───┘  └───┬───┘                   │
    │      └──────────┼──────────┘                        │
    │                 ▼                                    │
    │         Merge Results                               │
    │                 ▼                                    │
    │         Response (< 200ms!)                         │
    └──────────────────────────────────────────────────────┘
    
    
    AMAZON PRODUCT PAGE:
    ══════════════════
    
    Loading amazon.com/product/123
    
    ┌──────────────────────────────────────────────────────┐
    │  Concurrent requests (all at once):                 │
    │                                                      │
    │  ├── Product details ────────────▶ Product Service  │
    │  ├── Price + offers ─────────────▶ Pricing Service  │
    │  ├── Reviews ────────────────────▶ Review Service   │
    │  ├── Recommendations ────────────▶ ML Service       │
    │  ├── Inventory / shipping ───────▶ Inventory Svc    │
    │  └── Seller info ────────────────▶ Seller Service   │
    │                                                      │
    │  All 6 requests run CONCURRENTLY, not sequentially! │
    │                                                      │
    │  Sequential: 50ms + 30ms + 40ms + 80ms + 20ms + 15ms│
    │            = 235ms total                             │
    │                                                      │
    │  Concurrent: max(50, 30, 40, 80, 20, 15) = 80ms!   │
    │            3x faster!                                │
    └──────────────────────────────────────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Race conditions — two threads modifying the same data
       balance = 100
       Thread A: reads balance (100), adds 50 → writes 150
       Thread B: reads balance (100), adds 30 → writes 130
       Expected: 180. Got: 130! Thread A's update was LOST!
       ✅ Use locks, mutexes, or atomic operations
    
    ❌ Deadlocks — two threads waiting for each other forever
       Thread A: holds Lock 1, waiting for Lock 2
       Thread B: holds Lock 2, waiting for Lock 1
       Both wait FOREVER. Application hangs!
       ✅ Always acquire locks in the same order
    
    ❌ Using Python threads for CPU-bound work
       → GIL prevents true parallelism. No speedup!
       ✅ Use multiprocessing for CPU tasks in Python
    
    ❌ Creating too many threads
       → Context switching overhead > actual work done
       ✅ Use thread pools with a sensible max size
    
    ❌ Shared mutable state across threads
       → Source of almost ALL concurrency bugs
       ✅ Prefer immutable data, message passing, or thread-local storage
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. Concurrency = dealing with many things at once (interleaving).  ║
║     Parallelism = doing many things at once (simultaneous on        ║
║     multiple cores). Both are needed for high-performance servers.  ║
║                                                                      ║
║  2. Processes are heavy (100 MB), threads are medium (1-8 MB),     ║
║     coroutines are light (~1 KB). Choose based on your needs.      ║
║                                                                      ║
║  3. I/O-bound → use async/await or threads.                        ║
║     CPU-bound → use multiple processes or parallel threads.         ║
║                                                                      ║
║  4. Python GIL limits threads to one CPU core for Python code.     ║
║     Use multiprocessing for CPU-bound tasks in Python.              ║
║                                                                      ║
║  5. Java 21 Virtual Threads are a game-changer — millions of       ║
║     lightweight threads with ~1 KB each. Best of both worlds.      ║
║                                                                      ║
║  6. Race conditions and deadlocks are the biggest dangers of       ║
║     concurrent programming. Use locks carefully, prefer            ║
║     immutable data and message passing.                              ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

Now that you understand concurrency and parallelism, let's answer the practical question: exactly how many concurrent requests can YOUR server handle? Next: [Chapter 3.9: How Many Concurrent Requests Can a Server Handle?](./09-concurrent-requests-and-server-config.md).

---

[⬅️ Previous: Thread Models](./07-thread-models.md) | [⬆️ Index](../../00-INDEX.md) | [Next: Concurrent Requests & Server Config ➡️](./09-concurrent-requests-and-server-config.md)
