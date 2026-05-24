# Server Sizing — How Much CPU, RAM & Disk Do You Need?

> **What you'll learn**: How to determine the right amount of compute resources (CPU, memory, disk, network) for your application so you don't overpay or underperform.

---

## Real-Life Analogy

Imagine you're opening a restaurant. Before you sign a lease, you need to figure out:

- **How many tables do you need?** (CPU — how many things you can do at once)
- **How big should the kitchen be?** (RAM — how much you can work on simultaneously)
- **How large is the pantry?** (Disk — how much stuff you can store)
- **How wide is the entrance door?** (Network — how fast people can come in and out)

If you rent a space meant for 500 guests but only serve 20 per day, you're burning money. If you squeeze 500 guests into a space meant for 20, everyone has a terrible experience.

**Server sizing is the art of matching your infrastructure to your actual workload** — not too big (expensive), not too small (crashes under load).

---

## Core Concept Explained Step-by-Step

### The Four Resources That Matter

Every server has four fundamental resources. Think of them as the four legs of a table — if any one leg is too short, the whole table wobbles.

```
┌─────────────────────────────────────────────────────────────┐
│                    SERVER RESOURCES                          │
├──────────────┬──────────────┬──────────────┬───────────────┤
│     CPU      │     RAM      │     DISK     │   NETWORK     │
│  (Compute)   │   (Memory)   │  (Storage)   │ (Bandwidth)   │
├──────────────┼──────────────┼──────────────┼───────────────┤
│ Calculations │ Working      │ Permanent    │ Data in/out   │
│ per second   │ space        │ storage      │ speed         │
│              │              │              │               │
│ Like: Chef   │ Like: Counter│ Like: Pantry │ Like: Door    │
│ cooking      │ space        │ shelves      │ width         │
└──────────────┴──────────────┴──────────────┴───────────────┘
```

### Step 1: Understand Your Workload Type

Different applications stress different resources:

| Workload Type | Primary Bottleneck | Examples |
|--------------|-------------------|----------|
| **CPU-bound** | CPU cores/speed | Video encoding, image processing, encryption, ML inference |
| **Memory-bound** | RAM | In-memory caches, large dataset processing, JVM apps |
| **I/O-bound** | Disk or Network | Database servers, file servers, media streaming |
| **Network-bound** | Bandwidth/latency | CDN nodes, API gateways, real-time streaming |

### Step 2: Know Your Numbers

Before sizing, gather these metrics:

```
┌─────────────────────────────────────────────────┐
│           QUESTIONS TO ANSWER                    │
├─────────────────────────────────────────────────┤
│ 1. How many requests per second (RPS)?          │
│ 2. How much data per request? (KB/MB)           │
│ 3. How long does each request take? (ms)        │
│ 4. How much data do you store total? (GB/TB)    │
│ 5. How fast is the data growing? (GB/month)     │
│ 6. What's your peak vs average traffic ratio?   │
│ 7. What response time is acceptable? (p99)      │
└─────────────────────────────────────────────────┘
```

### Step 3: Apply the Sizing Formula

Here's a simplified approach:

```
                    SIZING FLOW
                    
    ┌──────────────┐
    │  Gather      │
    │  Requirements│
    │  (RPS, data) │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  Estimate    │
    │  Per-Request │
    │  Resources   │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  Multiply by │
    │  Concurrency │
    │  (peak RPS)  │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  Add Safety  │
    │  Margin      │
    │  (1.5x-3x)  │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  Pick Server │
    │  Instance    │
    │  Type        │
    └──────────────┘
```

---

## How It Works Internally

### CPU Sizing

**Key Formula:**

```
Required vCPUs = (RPS × Average Processing Time per Request) / 1000

Example:
- 1000 RPS
- Each request takes 50ms of CPU time
- Required = (1000 × 50) / 1000 = 50 vCPUs
- With 70% target utilization: 50 / 0.7 ≈ 72 vCPUs
- Across multiple servers: 72 / 8 vCPUs per server = 9 servers
```

**CPU considerations:**
- **Clock speed** matters for single-threaded work (higher GHz = faster per task)
- **Core count** matters for parallel work (more cores = more simultaneous tasks)
- **CPU architecture** — ARM (Graviton) vs x86 (Intel/AMD) affects price-performance
- **Burst vs sustained** — AWS T-series instances give burst credits but throttle under sustained load

### Memory (RAM) Sizing

**Key Formula:**

```
Required RAM = (Concurrent Connections × Memory per Connection)
             + Application Base Memory
             + OS Overhead
             + Buffer/Cache Space

Example (Web API server):
- 5000 concurrent connections × 2 MB each = 10 GB
- Application base (JVM heap, runtime) = 4 GB
- OS + overhead = 2 GB
- Buffers = 2 GB
- Total = 18 GB → Round up to 32 GB instance
```

**Memory considerations:**
- **JVM applications** need heap + metaspace + native memory (plan for 1.5x–2x heap size)
- **Connection pools** each hold memory (DB connections, HTTP clients)
- **Caching in-process** (Guava, Caffeine) lives in RAM
- **Memory leaks** — always plan for some overhead to absorb leaks before restart

### Disk Sizing

**Types of storage:**

```
┌────────────────────────────────────────────────────────┐
│                  STORAGE TYPES                          │
├────────────┬──────────────┬─────────────┬─────────────┤
│   Type     │    Speed     │    Cost     │  Use Case   │
├────────────┼──────────────┼─────────────┼─────────────┤
│ NVMe SSD   │ 3-7 GB/s    │ $$$$        │ Databases   │
│ SSD (gp3)  │ 125-1000    │ $$          │ General     │
│            │  MB/s        │             │             │
│ HDD (st1)  │ 20-500 MB/s │ $           │ Logs, cold  │
│            │              │             │ data        │
│ Object     │ Varies       │ ¢           │ Media,      │
│ (S3)       │              │             │ backups     │
└────────────┴──────────────┴─────────────┴─────────────┘
```

**Key Formula:**

```
Required Disk = Current Data
              + (Growth Rate × Months until next review)
              + Logs & Temp Files
              + Safety Buffer (20-30%)

Example (Database server):
- Current data: 200 GB
- Growth: 20 GB/month × 6 months = 120 GB
- Logs/temp: 50 GB
- Total: 370 GB + 30% buffer = 481 GB → 500 GB SSD
```

**IOPS matter more than size for databases:**

```
Required IOPS = (Read Queries/sec × Avg reads per query)
              + (Write Queries/sec × Avg writes per query)

Example:
- 2000 read queries/sec × 3 random reads each = 6000 read IOPS
- 500 write queries/sec × 2 writes each = 1000 write IOPS
- Total: 7000 IOPS → Choose gp3 with provisioned IOPS
```

### Network Sizing

**Key Formula:**

```
Required Bandwidth = RPS × Average Response Size × 8 (bits per byte)

Example:
- 10,000 RPS
- Average response: 50 KB
- Bandwidth = 10,000 × 50 KB × 8 = 4 Gbps

Also consider:
- Inter-service communication (east-west traffic)
- Database replication traffic
- Backup traffic
- Monitoring/logging data
```

---

## Code Examples

### Python — Server Sizing Calculator

```python
# server_sizing.py — Calculate required server resources

class ServerSizer:
    """Estimates server resources based on workload parameters."""
    
    def __init__(self, rps, avg_request_ms, avg_response_kb,
                 concurrent_users, memory_per_conn_mb=2):
        self.rps = rps                          # Requests per second
        self.avg_request_ms = avg_request_ms    # Avg processing time (ms)
        self.avg_response_kb = avg_response_kb  # Avg response size (KB)
        self.concurrent_users = concurrent_users
        self.memory_per_conn_mb = memory_per_conn_mb
    
    def estimate_cpu(self, target_utilization=0.7):
        """Estimate required vCPUs."""
        # CPU seconds needed per second of wall time
        raw_cpus = (self.rps * self.avg_request_ms) / 1000
        # Don't run CPUs at 100% — leave room for spikes
        return round(raw_cpus / target_utilization)
    
    def estimate_ram_gb(self, app_base_gb=2, os_overhead_gb=1):
        """Estimate required RAM in GB."""
        connection_memory = (self.concurrent_users * self.memory_per_conn_mb) / 1024
        total = connection_memory + app_base_gb + os_overhead_gb
        # Round up to next power of 2 (common instance sizes)
        return self._next_power_of_2(total)
    
    def estimate_bandwidth_gbps(self):
        """Estimate required network bandwidth in Gbps."""
        bits_per_second = self.rps * self.avg_response_kb * 1024 * 8
        return round(bits_per_second / 1_000_000_000, 2)
    
    def estimate_servers(self, cpus_per_server=8, ram_per_server_gb=32):
        """Estimate number of servers needed."""
        cpu_bound = self.estimate_cpu() / cpus_per_server
        ram_bound = self.estimate_ram_gb() / ram_per_server_gb
        # Take the higher constraint
        return max(round(cpu_bound + 0.5), round(ram_bound + 0.5))
    
    def _next_power_of_2(self, n):
        """Round up to next common memory size (8, 16, 32, 64...)."""
        sizes = [4, 8, 16, 32, 64, 128, 256, 512]
        for s in sizes:
            if s >= n:
                return s
        return sizes[-1]
    
    def print_report(self):
        """Print a complete sizing report."""
        print("=" * 50)
        print("SERVER SIZING REPORT")
        print("=" * 50)
        print(f"Workload: {self.rps} RPS, {self.avg_request_ms}ms avg latency")
        print(f"Concurrent users: {self.concurrent_users}")
        print("-" * 50)
        print(f"Estimated vCPUs needed:    {self.estimate_cpu()}")
        print(f"Estimated RAM needed:      {self.estimate_ram_gb()} GB")
        print(f"Estimated bandwidth:       {self.estimate_bandwidth_gbps()} Gbps")
        print(f"Estimated servers (8c/32G): {self.estimate_servers()}")
        print("=" * 50)


# Example: E-commerce API serving 5000 RPS
sizer = ServerSizer(
    rps=5000,
    avg_request_ms=30,
    avg_response_kb=15,
    concurrent_users=10000
)
sizer.print_report()
```

### Java — Server Sizing Calculator

```java
// ServerSizer.java — Estimate server resources for a given workload

public class ServerSizer {
    private final int rps;              // Requests per second
    private final int avgRequestMs;     // Average processing time (ms)
    private final int avgResponseKb;    // Average response size (KB)
    private final int concurrentUsers;
    private final int memoryPerConnMb;

    public ServerSizer(int rps, int avgRequestMs, int avgResponseKb,
                       int concurrentUsers, int memoryPerConnMb) {
        this.rps = rps;
        this.avgRequestMs = avgRequestMs;
        this.avgResponseKb = avgResponseKb;
        this.concurrentUsers = concurrentUsers;
        this.memoryPerConnMb = memoryPerConnMb;
    }

    // Estimate required vCPUs at 70% target utilization
    public int estimateCpu() {
        double rawCpus = (rps * avgRequestMs) / 1000.0;
        return (int) Math.ceil(rawCpus / 0.7);
    }

    // Estimate required RAM in GB
    public int estimateRamGb(int appBaseGb, int osOverheadGb) {
        double connectionMemoryGb = (concurrentUsers * memoryPerConnMb) / 1024.0;
        double total = connectionMemoryGb + appBaseGb + osOverheadGb;
        // Round to next common size
        int[] sizes = {4, 8, 16, 32, 64, 128, 256, 512};
        for (int size : sizes) {
            if (size >= total) return size;
        }
        return sizes[sizes.length - 1];
    }

    // Estimate bandwidth in Gbps
    public double estimateBandwidthGbps() {
        double bitsPerSecond = (double) rps * avgResponseKb * 1024 * 8;
        return Math.round(bitsPerSecond / 1_000_000_000.0 * 100) / 100.0;
    }

    // Estimate number of servers needed
    public int estimateServers(int cpusPerServer, int ramPerServerGb) {
        int cpuBound = (int) Math.ceil((double) estimateCpu() / cpusPerServer);
        int ramBound = (int) Math.ceil((double) estimateRamGb(2, 1) / ramPerServerGb);
        return Math.max(cpuBound, ramBound);
    }

    public void printReport() {
        System.out.println("=".repeat(50));
        System.out.println("SERVER SIZING REPORT");
        System.out.println("=".repeat(50));
        System.out.printf("Workload: %d RPS, %dms avg latency%n", rps, avgRequestMs);
        System.out.printf("Concurrent users: %d%n", concurrentUsers);
        System.out.println("-".repeat(50));
        System.out.printf("Estimated vCPUs needed:     %d%n", estimateCpu());
        System.out.printf("Estimated RAM needed:       %d GB%n", estimateRamGb(2, 1));
        System.out.printf("Estimated bandwidth:        %.2f Gbps%n", estimateBandwidthGbps());
        System.out.printf("Estimated servers (8c/32G): %d%n", estimateServers(8, 32));
        System.out.println("=".repeat(50));
    }

    public static void main(String[] args) {
        // E-commerce API serving 5000 RPS
        ServerSizer sizer = new ServerSizer(5000, 30, 15, 10000, 2);
        sizer.printReport();
    }
}
```

---

## Infrastructure Examples

### AWS Instance Types — Choosing the Right One

```
┌──────────────────────────────────────────────────────────────────┐
│              AWS INSTANCE TYPE CHEAT SHEET                        │
├───────────┬──────────────────────────────────────────────────────┤
│ Family    │ Use Case                                             │
├───────────┼──────────────────────────────────────────────────────┤
│ t3/t4g    │ Burstable. Good for dev/test, low-traffic APIs      │
│ m6i/m7g   │ General purpose. Balanced CPU/RAM. Most web apps    │
│ c6i/c7g   │ Compute-optimized. Video encoding, ML, batch jobs   │
│ r6i/r7g   │ Memory-optimized. Caches, in-memory DBs, analytics │
│ i3/i4i    │ Storage-optimized. Databases, data warehouses       │
│ g5/p4     │ GPU. ML training/inference, video transcoding       │
└───────────┴──────────────────────────────────────────────────────┘

Suffix guide:
  "g" = Graviton (ARM) — 20-40% cheaper, same performance
  "n" = Enhanced networking
  "d" = Local NVMe storage
  "a" = AMD processors (slightly cheaper)
```

### Real Sizing Example — PostgreSQL Database Server

```yaml
# PostgreSQL on AWS — Sizing for 10,000 queries/second

Workload:
  read_qps: 8000
  write_qps: 2000
  avg_row_size: 500 bytes
  total_data: 500 GB
  connections: 200

Sizing Decision:
  instance: r6g.2xlarge  # 8 vCPU, 64 GB RAM (memory-optimized)
  
  rationale:
    cpu: "8 cores handles 10K simple queries/sec easily"
    ram: "64 GB allows ~50 GB shared_buffers (caching most hot data)"
    
  storage:
    type: gp3
    size: 1000 GB        # 500 GB data + growth + WAL + backups
    iops: 10000          # Provisioned for write-heavy workload
    throughput: 500 MB/s

  network: "Up to 10 Gbps (sufficient for replication + client traffic)"

PostgreSQL Config Tuning:
  shared_buffers: 48GB          # ~75% of RAM for caching
  effective_cache_size: 56GB     # How much OS can cache
  work_mem: 64MB                 # Per-query sort memory
  max_connections: 200
  wal_buffers: 64MB
```

### Nginx Web Server Sizing

```nginx
# Nginx sizing for 50,000 concurrent connections

worker_processes auto;           # One per CPU core
worker_connections 10240;        # Connections per worker

# On 8-core machine: 8 × 10240 = ~80K connections capacity
# Rule: Keep actual load at 60-70% of theoretical max

# Memory per connection: ~2-4 KB for static, 10-20 KB for proxying
# 50,000 connections × 20 KB = ~1 GB RAM just for connections
# Total recommended: 4-8 GB RAM for this workload
```

---

## Real-World Example

### How Netflix Sizes Its Servers

Netflix runs thousands of microservices, each sized independently:

```
┌─────────────────────────────────────────────────────┐
│          NETFLIX SIZING STRATEGY                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. Every service starts with a "baseline" instance │
│     type based on its category:                     │
│                                                     │
│     API Services → m5.xlarge (4 CPU, 16 GB)        │
│     Caching      → r5.2xlarge (8 CPU, 64 GB)      │
│     Encoding     → c5.4xlarge (16 CPU, 32 GB)     │
│                                                     │
│  2. Load test with production-like traffic          │
│                                                     │
│  3. Observe: CPU util, memory, latency at p99      │
│                                                     │
│  4. Right-size: Move to smaller/larger instance     │
│     until target 60-70% utilization at peak         │
│                                                     │
│  5. Auto-scale horizontally for traffic variance    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

Netflix uses **Titus** (their container platform) with resource profiles. Each service declares its needs:

```json
{
  "service": "recommendation-engine",
  "resources": {
    "cpu": 4,
    "memoryMB": 8192,
    "diskMB": 10240,
    "networkMbps": 2000
  },
  "scaling": {
    "min": 50,
    "max": 500,
    "target_cpu_percent": 65
  }
}
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | What To Do Instead |
|---------|-------------|-------------------|
| **Sizing for peak on Day 1** | Wastes money for months while traffic is low | Start smaller, auto-scale, resize as you grow |
| **Ignoring memory for JVM apps** | JVM uses 2-3x the heap size in total memory | Set instance RAM to 2.5x your -Xmx setting |
| **Only looking at CPU %** | Can miss memory pressure, disk I/O bottlenecks | Monitor ALL four resources |
| **Using T-series for production** | Burst credits deplete under sustained load → throttling | Use M-series for predictable workloads |
| **Not accounting for spikes** | System crashes during flash sales, viral events | Plan for 2-3x average traffic |
| **Same instance type for everything** | Database needs are different from API server needs | Choose instance family per workload type |
| **Ignoring network limits** | Smaller instances have lower network bandwidth caps | Check instance network specs |

---

## When to Use / When NOT to Use

### When to Invest in Careful Sizing

- ✅ Before launching a new production service
- ✅ When cloud bills are growing faster than revenue
- ✅ Before a big traffic event (Black Friday, product launch)
- ✅ When performance is degrading without obvious code issues
- ✅ During quarterly infrastructure reviews

### When NOT to Overthink Sizing

- ❌ Development and staging environments (use smallest viable)
- ❌ Very early-stage startups with < 100 users (just pick a small general-purpose instance)
- ❌ When you have auto-scaling configured and it's working well
- ❌ Short-lived batch jobs (use spot/preemptible instances)

---

## Key Takeaways

1. **Every application stresses different resources** — identify whether you're CPU-bound, memory-bound, I/O-bound, or network-bound before choosing instance types.

2. **Size for peak × safety margin**, not average — your system must handle the worst-case traffic without falling over.

3. **The four resources are CPU, RAM, Disk (size + IOPS), and Network** — a bottleneck in ANY one of them will limit your entire system.

4. **Start with math, validate with load testing** — formulas give you a starting point, real load tests prove whether it works.

5. **Right-sizing is an ongoing process** — traffic patterns change, code changes, data grows. Review sizing quarterly.

6. **Target 60-70% utilization at peak** — this leaves headroom for unexpected spikes without wasting money at low traffic.

7. **Use ARM instances (Graviton) for 20-40% cost savings** where your stack supports it — most modern runtimes (Java 11+, Python, Node.js, Go) work perfectly on ARM.

---

## What's Next?

Next, we'll explore **Cost Optimization in Cloud (Reserved, Spot, On-Demand)** — where you'll learn the different pricing models cloud providers offer and how to save 30-70% on your infrastructure bill by choosing the right purchasing strategy.
