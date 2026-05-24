# Profiling Applications — Finding Bottlenecks

> **What you'll learn**: How to use profiling tools and flame graphs to pinpoint exactly where your application is spending time and memory — like a detective finding the one slow cog in a machine.

---

## Real-Life Analogy

Imagine you're a **detective** investigating why a factory is producing goods slowly. You could guess: "Maybe the paint machine is slow?" But that's just a guess. Instead, you:

1. **Attach a stopwatch** to every single machine and worker
2. **Record** how long each step takes
3. **Find the bottleneck** — maybe 80% of the time is spent waiting for one slow conveyor belt

That's exactly what a **profiler** does for your code. It attaches timers to every function, measures where your program spends its time (and memory), and shows you a clear picture of the bottleneck.

```
Without profiling:          With profiling:
┌───────────────┐           ┌───────────────────────────────┐
│  "The app is  │           │  Function         Time        │
│   slow...     │           │  ─────────────────────────    │
│   maybe the   │           │  db_query()       72% ████▓  │
│   database?   │           │  serialize()      15% ██     │
│   maybe the   │           │  network_call()    8% █      │
│   network?"   │           │  validation()      5% ▒      │
│               │           │                               │
│  😕 GUESSING  │           │  🎯 FOUND IT: db_query()!    │
└───────────────┘           └───────────────────────────────┘
```

---

## Core Concept Explained Step-by-Step

### Step 1: What is Profiling?

**Profiling** is the act of measuring your program's runtime behavior to find:
- **Where time is spent** (CPU profiling)
- **Where memory is allocated** (Memory profiling)
- **Where I/O waits happen** (I/O profiling)
- **Where threads block** (Concurrency profiling)

### Step 2: Types of Profilers

```
┌─────────────────────────────────────────────────────────────┐
│                    PROFILER TYPES                            │
├─────────────────┬───────────────────────────────────────────┤
│                 │                                           │
│  CPU Profiler   │  Measures time spent in functions         │
│                 │  "Where is the CPU busy?"                 │
│                 │                                           │
├─────────────────┼───────────────────────────────────────────┤
│                 │                                           │
│  Memory         │  Tracks allocations & heap usage          │
│  Profiler       │  "What's eating all my RAM?"              │
│                 │                                           │
├─────────────────┼───────────────────────────────────────────┤
│                 │                                           │
│  I/O Profiler   │  Measures disk reads, network waits       │
│                 │  "Why is the app waiting?"                │
│                 │                                           │
├─────────────────┼───────────────────────────────────────────┤
│                 │                                           │
│  Concurrency    │  Finds lock contention, deadlocks         │
│  Profiler       │  "Why are threads waiting for each other?"│
│                 │                                           │
└─────────────────┴───────────────────────────────────────────┘
```

### Step 3: Sampling vs Instrumentation Profiling

There are two fundamental approaches:

```
INSTRUMENTATION PROFILING:
──────────────────────────
  Inserts measurement code at EVERY function entry/exit
  
  func doWork() {
    START_TIMER()        ← injected
    ... actual code ...
    STOP_TIMER()         ← injected
  }

  ✅ Exact measurements (captures every call)
  ❌ High overhead (slows program by 2-10x)
  ❌ Changes program behavior (observer effect)

SAMPLING PROFILING:
───────────────────
  Periodically interrupts the program and records the call stack

  Time ──▶  │  │  │  │  │  │  │  │  │  │
            ▼  ▼  ▼  ▼  ▼  ▼  ▼  ▼  ▼  ▼  ← samples (every 10ms)
  
  Sample 1: main() → processOrder() → queryDB()
  Sample 2: main() → processOrder() → queryDB()
  Sample 3: main() → processOrder() → serialize()
  Sample 4: main() → processOrder() → queryDB()
  
  Result: queryDB() seen in 75% of samples → it's the bottleneck!

  ✅ Low overhead (1-5%)
  ✅ Safe for production
  ❌ Statistical (might miss short functions)
```

### Step 4: Flame Graphs — The Most Powerful Visualization

A **flame graph** shows a visual stack trace — wider = more time spent:

```
┌─────────────────────────────────────────────────────────────┐
│                        main()                                │
├──────────────────────────────────┬──────────────────────────┤
│        handleRequest()           │      logMetrics()        │
├─────────────────────┬────────────┤                          │
│    processOrder()   │ validate() │                          │
├───────────┬─────────┤            │                          │
│ queryDB() │ cache() │            │                          │
├───────────┤         │            │                          │
│  execute  │         │            │                          │
│  _sql()   │         │            │                          │
└───────────┴─────────┴────────────┴──────────────────────────┘

Reading a Flame Graph:
  • X-axis = percentage of time (wider = slower)
  • Y-axis = call stack depth (bottom = entry point, top = leaf function)
  • Look for WIDE bars at the TOP — those are your bottlenecks
  
  In this example: queryDB() → execute_sql() is the widest at top
  = 🎯 Database queries are the #1 bottleneck
```

### Step 5: The Profiling Workflow

```
┌─────────────────────────────────────────────────────────────┐
│              THE PROFILING WORKFLOW                          │
│                                                             │
│  1. REPRODUCE ──▶ 2. PROFILE ──▶ 3. ANALYZE ──▶ 4. FIX    │
│     the issue       the code       flame graphs     code   │
│                                                             │
│  5. VERIFY ──▶ 6. REPEAT (next bottleneck)                 │
│     improvement                                             │
└─────────────────────────────────────────────────────────────┘

Important: After fixing one bottleneck, ANOTHER function
becomes the new bottleneck. Profile again!

Before fix:                    After fix:
  queryDB()      72% ████▓     serialize()    45% ███▓
  serialize()    15% ██        network()      30% ██▓
  network()       8% █         queryDB()      15% █▓
  validation()    5% ▒         validation()   10% █
```

---

## How It Works Internally

### How Sampling Profilers Work

1. A **timer interrupt** fires every N milliseconds (typically 10ms)
2. The OS pauses the program
3. The profiler captures the **current call stack**
4. The OS resumes the program
5. After collecting thousands of samples, the profiler counts how often each function appears

```
OS Timer Signal (SIGPROF on Linux)
    │
    ▼
┌──────────────────────┐
│  Pause Application   │
│  Read Stack Pointer  │─────▶ Stack Frame:
│  Walk Call Stack     │        main()
│  Record Sample       │        └─ handleRequest()
│  Resume Application  │            └─ processOrder()
└──────────────────────┘                └─ queryDB()  ← captured!
```

### How JVM Profiling Works (Java)

The JVM provides two profiling interfaces:

```
┌──────────────────────────────────────────────────────────┐
│                  JVM Profiling Stack                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Application Code                                        │
│       │                                                  │
│       ▼                                                  │
│  ┌────────────────┐    ┌──────────────────┐             │
│  │ JVM TI (Tool   │    │  JFR (Java       │             │
│  │  Interface)    │    │  Flight Recorder) │             │
│  │                │    │                   │             │
│  │ • JVMTI agent  │    │ • Built into JVM  │             │
│  │ • High overhead│    │ • Very low overhead│             │
│  │ • Very detailed│    │ • Always-on capable│             │
│  └────────────────┘    └──────────────────┘             │
│       │                        │                         │
│       ▼                        ▼                         │
│  ┌────────────────────────────────────────┐             │
│  │  async-profiler (best of both worlds)  │             │
│  │  • Uses perf_events (Linux)            │             │
│  │  • No safepoint bias                   │             │
│  │  • Production-safe (~1% overhead)      │             │
│  └────────────────────────────────────────┘             │
└──────────────────────────────────────────────────────────┘
```

### Memory Profiling — Finding Leaks

```
Heap Snapshot Timeline:
                                              ← LEAK!
Memory                                     ╱
(MB)    ╱╲    ╱╲    ╱╲    ╱╲    ╱╲     ╱╲
100 │  ╱  ╲  ╱  ╲  ╱  ╲  ╱  ╲  ╱  ╲  ╱  ╲──────
    │ ╱    ╲╱    ╲╱    ╲╱    ╲╱    ╲╱
 50 │╱        Normal GC pattern
    │         (allocate → collect → allocate)
    └──────────────────────────────────────────── Time

VS. Memory Leak:
Memory                                    ╱╱╱╱
(MB)                                   ╱╱╱
200 │                               ╱╱╱
    │                            ╱╱╱      ← Never goes down!
150 │                         ╱╱╱          Objects accumulating
    │                      ╱╱╱
100 │                   ╱╱╱
    │                ╱╱╱
 50 │           ╱╱╱╱╱
    │      ╱╱╱╱╱
    │ ╱╱╱╱╱
    └──────────────────────────────────────────── Time
                                         💀 OOM Kill
```

---

## Code Examples

### Python — CPU Profiling with cProfile and Flame Graphs

```python
import cProfile
import pstats
from io import StringIO

# Method 1: Profile a function with cProfile
def profile_function(func, *args, **kwargs):
    """Profile a function and print top 10 time-consuming calls"""
    profiler = cProfile.Profile()
    profiler.enable()
    
    result = func(*args, **kwargs)  # Run the function
    
    profiler.disable()
    stream = StringIO()
    stats = pstats.Stats(profiler, stream=stream)
    stats.sort_stats('cumulative')  # Sort by total time
    stats.print_stats(10)  # Top 10 functions
    print(stream.getvalue())
    return result

# Method 2: Using py-spy for flame graphs (production-safe)
# Install: pip install py-spy
# Usage from command line:
#   py-spy record -o flamegraph.svg --pid 12345
#   py-spy record -o flamegraph.svg -- python app.py

# Method 3: Memory profiling with tracemalloc
import tracemalloc

def find_memory_leaks():
    tracemalloc.start()
    
    # ... run your code ...
    leaked_data = []
    for i in range(100000):
        leaked_data.append(" " * 1000)  # Simulated leak
    
    # Take snapshot and find top memory consumers
    snapshot = tracemalloc.take_snapshot()
    top_stats = snapshot.statistics('lineno')
    
    print("Top 5 memory allocations:")
    for stat in top_stats[:5]:
        print(f"  {stat}")

# Method 4: Line-by-line profiling with line_profiler
# Install: pip install line_profiler
# Decorate with @profile, then run:
#   kernprof -l -v script.py

# @profile  # Uncomment when using kernprof
def slow_function():
    total = 0
    for i in range(1000000):      # 40% time here
        total += i * i
    result = expensive_db_call()   # 55% time here  
    formatted = format_output(result)  # 5% time here
    return formatted
```

### Java — Profiling with async-profiler and JFR

```java
import java.lang.management.*;
import java.util.*;

public class ProfilingExample {
    
    // Method 1: Manual timing (basic profiling)
    public static void profileMethod(Runnable task, String name) {
        long start = System.nanoTime();
        task.run();
        long durationMs = (System.nanoTime() - start) / 1_000_000;
        System.out.printf("[PROFILE] %s: %dms%n", name, durationMs);
    }

    // Method 2: JVM Memory monitoring
    public static void printMemoryUsage() {
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heap = memBean.getHeapMemoryUsage();
        
        System.out.printf("Heap: used=%dMB, max=%dMB, ratio=%.1f%%%n",
            heap.getUsed() / (1024 * 1024),
            heap.getMax() / (1024 * 1024),
            (double) heap.getUsed() / heap.getMax() * 100);
    }

    // Method 3: GC monitoring
    public static void printGCStats() {
        for (GarbageCollectorMXBean gc : 
                ManagementFactory.getGarbageCollectorMXBeans()) {
            System.out.printf("GC [%s]: count=%d, time=%dms%n",
                gc.getName(), gc.getCollectionCount(), 
                gc.getCollectionTime());
        }
    }

    // Method 4: Thread dump for deadlock detection
    public static void detectDeadlocks() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        long[] deadlocked = threadBean.findDeadlockedThreads();
        if (deadlocked != null) {
            ThreadInfo[] infos = threadBean.getThreadInfo(deadlocked, true, true);
            for (ThreadInfo info : infos) {
                System.out.println("DEADLOCK: " + info);
            }
        }
    }
}

// Using async-profiler (command line):
// java -agentpath:/path/to/libasyncProfiler.so=start,event=cpu,
//   file=flamegraph.html -jar app.jar

// Using JFR (built into JDK 11+):
// java -XX:StartFlightRecording=duration=60s,filename=recording.jfr 
//   -jar app.jar
```

---

## Infrastructure Examples

### Continuous Profiling in Production (Pyroscope)

```yaml
# docker-compose.yml — Pyroscope continuous profiler
version: '3'
services:
  pyroscope:
    image: pyroscope/pyroscope:latest
    ports:
      - "4040:4040"
    command: ["server"]

  app:
    build: .
    environment:
      - PYROSCOPE_SERVER_ADDRESS=http://pyroscope:4040
      - PYROSCOPE_APPLICATION_NAME=my-web-app
      - PYROSCOPE_AUTH_TOKEN=${PYROSCOPE_TOKEN}
    depends_on:
      - pyroscope
```

```python
# Python app with Pyroscope continuous profiling
import pyroscope

pyroscope.configure(
    application_name="order-service",
    server_address="http://pyroscope:4040",
    sample_rate=100,  # 100 Hz sampling
    tags={"region": "us-east-1", "env": "production"}
)

# Now every function is automatically profiled!
# View flame graphs at http://localhost:4040
```

### async-profiler with Kubernetes

```yaml
# Kubernetes pod with profiling sidecar
apiVersion: v1
kind: Pod
metadata:
  name: app-with-profiler
spec:
  shareProcessNamespace: true  # Allow profiler to see app process
  containers:
    - name: app
      image: my-java-app:latest
      ports:
        - containerPort: 8080
    - name: profiler
      image: async-profiler:latest
      command: ["/bin/sh", "-c", "sleep 30 && ./profiler.sh -d 60 -f /output/flamegraph.html 1"]
      volumeMounts:
        - name: profile-output
          mountPath: /output
  volumes:
    - name: profile-output
      emptyDir: {}
```

---

## Real-World Example

### Netflix — Continuous Profiling in Production

Netflix runs **continuous profiling** across all their microservices (thousands of instances):

```
Netflix Profiling Architecture:
┌────────────────────────────────────────────────────────┐
│                                                        │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐              │
│  │Service A│  │Service B│  │Service C│  (thousands)  │
│  │ + agent │  │ + agent │  │ + agent │              │
│  └────┬────┘  └────┬────┘  └────┬────┘              │
│       │             │             │                    │
│       └─────────────┼─────────────┘                   │
│                     │                                  │
│                     ▼                                  │
│         ┌───────────────────┐                         │
│         │  Profile Collector │                         │
│         │  (centralized)     │                         │
│         └─────────┬─────────┘                         │
│                   │                                    │
│                   ▼                                    │
│         ┌───────────────────┐                         │
│         │  Flame Graph UI    │                         │
│         │  (compare before/  │                         │
│         │   after deploys)   │                         │
│         └───────────────────┘                         │
└────────────────────────────────────────────────────────┘

They can diff flame graphs between versions:
  "After deploy v2.3, serialize() went from 5% → 25% CPU"
  → Instant detection of performance regressions!
```

### Google — Wide Profiling (Google-Wide Profiling)

Google profiles ALL production servers continuously with <1% overhead, collecting:
- CPU profiles every 10 seconds
- Heap profiles every 30 seconds  
- Contention profiles for lock debugging

This led to discoveries like: "2% of all Google CPU is spent in protobuf serialization."

---

## Common Mistakes / Pitfalls

| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Profiling only in development | Dev environment ≠ production behavior | Use production-safe profilers (async-profiler, py-spy) |
| Optimizing before profiling | You'll optimize the wrong thing 90% of the time | Always profile first, then optimize |
| Ignoring GC pauses | GC can add 50-200ms latency spikes | Profile GC separately (JFR, GC logs) |
| Profiling too short a period | Misses intermittent bottlenecks | Profile for at least 60 seconds under load |
| Only looking at CPU | I/O wait, lock contention, GC are often worse | Profile CPU, memory, I/O, and locks |
| Using instrumentation profilers in production | 2-10x overhead makes results unreliable | Use sampling profilers (<2% overhead) |

---

## When to Use / When NOT to Use

### Profile WHEN:
- Response time has increased after a deploy
- CPU utilization is high but you don't know why
- Memory keeps growing (potential leak)
- You need to optimize a hot path
- Before and after optimization (to prove improvement)

### DON'T Profile WHEN:
- The bottleneck is obviously the network (ping first!)
- The issue is infrastructure (disk full, OOM killed by OS)
- You haven't reproduced the problem yet (profile + reproduce together)
- You're premature-optimizing code that runs once a day

---

## Key Takeaways

- **Never guess** where the bottleneck is — always profile first
- **Sampling profilers** (py-spy, async-profiler) are production-safe with <2% overhead
- **Flame graphs** are the single best visualization for finding CPU bottlenecks — look for wide bars at the top
- **Profile continuously** in production (Pyroscope, Google Cloud Profiler) to catch regressions immediately
- **Memory profiling** finds leaks by comparing heap snapshots over time
- After fixing one bottleneck, **profile again** — a new bottleneck will emerge
- The **80/20 rule** applies: usually 20% of functions consume 80% of resources

---

## What's Next?

Now that you can find bottlenecks, the next step is to simulate real production traffic and see how your system behaves under pressure. That's **Chapter 19.3: Load Testing & Stress Testing (JMeter, k6, Locust)** — where you'll learn to break your system on purpose before your users do.
