# Bulkhead Pattern — Isolating Failures

> **What you'll learn**: How to partition your system into isolated compartments so that a failure in one area can't spread and sink the entire ship — inspired by the watertight compartments in ocean liners.

---

## Real-Life Analogy

The **Titanic** sank because water flooded from one compartment to the next — the bulkheads didn't go all the way to the top. Modern ships have **watertight bulkheads** — if one compartment floods, the doors seal shut and the rest of the ship stays dry.

```
SHIP WITHOUT PROPER BULKHEADS (Titanic):
┌─────────────────────────────────────────────┐
│  ~~~~WATER~~~~  →  →  →  →  →  →  →  →  → │
│  Compartment 1   Compartment 2   Comp 3     │
│  FLOODED         FLOODING...     NEXT...    │
└─────────────────────────────────────────────┘
Result: Entire ship sinks


SHIP WITH PROPER BULKHEADS:
┌───────────┐│┌───────────┐│┌───────────┐
│ ~~WATER~~ │││           │││           │
│ FLOODED   │││  DRY ✓    │││  DRY ✓    │
│           │││           │││           │
└───────────┘│└───────────┘│└───────────┘
             ▲             ▲
        SEALED DOOR   SEALED DOOR
Result: One compartment floods, rest stays operational
```

In software, a **Bulkhead** isolates resources (threads, connections, memory) between different parts of your system. If one service consumes all its allocated resources, it can't steal from others.

---

## Core Concept Explained Step-by-Step

### The Problem: Resource Exhaustion Spreads

Without bulkheads, all services share the same resource pool:

```
WITHOUT BULKHEADS — Shared Thread Pool (100 threads total):

┌─────────────────────────────────────────────────────────────┐
│                SHARED THREAD POOL (100 threads)              │
│                                                             │
│  Payment API ──────────▶  ████████████████████████████████  │
│  (Service is SLOW)         60 threads BLOCKED waiting...    │
│                                                             │
│  User API ─────────────▶  █████████████████████████████     │
│  (needs threads too!)      35 threads used                  │
│                                                             │
│  Search API ───────────▶  █████                             │
│  (STARVED!)                5 threads left!                  │
│                                                             │
│  Catalog API ──────────▶  (NO THREADS! REJECTED!)           │
│                            0 threads available              │
│                                                             │
│  Result: Payment service being slow KILLS Search & Catalog! │
└─────────────────────────────────────────────────────────────┘
```

### With Bulkheads: Isolated Resource Pools

```
WITH BULKHEADS — Separate Thread Pools:

┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ PAYMENT POOL     │  │ USER POOL        │  │ SEARCH POOL      │
│ (30 threads max) │  │ (30 threads max) │  │ (20 threads max) │
│                  │  │                  │  │                  │
│ ████████████████ │  │ ████████         │  │ ████             │
│ ALL 30 BLOCKED!  │  │ 8/30 used        │  │ 4/20 used       │
│                  │  │                  │  │                  │
│ ⚠️ Payment is   │  │ ✓ Working fine   │  │ ✓ Working fine   │
│   degraded      │  │                  │  │                  │
└──────────────────┘  └──────────────────┘  └──────────────────┘
         │                     │                     │
         ▼                     ▼                     ▼
   Payment fails        Users work fine      Search works fine
   (isolated!)          (unaffected!)        (unaffected!)

Result: Payment service failure stays CONTAINED.
        Rest of the system operates normally.
```

### Types of Bulkheads

```
┌──────────────────────────────────────────────────────────────────┐
│                     BULKHEAD TYPES                                │
├──────────────────┬───────────────────────────────────────────────┤
│ Thread Pool      │ Each service gets its own dedicated thread    │
│ Isolation        │ pool. Exhausting one pool doesn't affect     │
│                  │ others.                                       │
├──────────────────┼───────────────────────────────────────────────┤
│ Semaphore        │ Limit concurrent calls to a service using     │
│ Isolation        │ a counter (lighter weight than thread pools). │
├──────────────────┼───────────────────────────────────────────────┤
│ Connection Pool  │ Separate database connection pools per        │
│ Isolation        │ service or feature.                           │
├──────────────────┼───────────────────────────────────────────────┤
│ Process          │ Run services in separate processes/containers.│
│ Isolation        │ One crashing doesn't kill others.             │
├──────────────────┼───────────────────────────────────────────────┤
│ Infrastructure   │ Separate clusters, databases, or regions      │
│ Isolation        │ for critical vs non-critical workloads.       │
└──────────────────┴───────────────────────────────────────────────┘
```

---

## How It Works Internally

### Thread Pool Bulkhead

```
┌─────────────────────────────────────────────────────────────────┐
│                  THREAD POOL BULKHEAD                            │
│                                                                 │
│  Incoming Request                                               │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────┐                                          │
│  │ Which service is  │                                          │
│  │ this call for?    │                                          │
│  └────┬──────┬───────┘                                          │
│       │      │      │                                           │
│       ▼      ▼      ▼                                           │
│  ┌────────┐ ┌────────┐ ┌────────┐                              │
│  │Pool: A │ │Pool: B │ │Pool: C │                              │
│  │Max: 20 │ │Max: 15 │ │Max: 10 │                              │
│  │Queue: 5│ │Queue: 5│ │Queue: 3│                              │
│  └───┬────┘ └───┬────┘ └───┬────┘                              │
│      │          │          │                                    │
│      ▼          ▼          ▼                                    │
│  ┌────────┐ ┌────────┐ ┌────────┐                              │
│  │Service │ │Service │ │Service │                              │
│  │   A    │ │   B    │ │   C    │                              │
│  └────────┘ └────────┘ └────────┘                              │
│                                                                 │
│  If Pool A is full (20 threads busy + 5 in queue):              │
│    → Request to A is REJECTED immediately                       │
│    → Pools B and C are completely unaffected                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Semaphore Bulkhead

```
┌─────────────────────────────────────────────────────────────────┐
│                  SEMAPHORE BULKHEAD                              │
│                                                                 │
│  Semaphore = Counter of available permits                       │
│                                                                 │
│  Service A Semaphore: max_concurrent = 10                       │
│                                                                 │
│  Request 1 arrives → acquire permit → count: 9 remaining       │
│  Request 2 arrives → acquire permit → count: 8 remaining       │
│  ...                                                            │
│  Request 10 arrives → acquire permit → count: 0 remaining      │
│  Request 11 arrives → NO PERMITS! → REJECTED immediately       │
│                                                                 │
│  Request 1 completes → release permit → count: 1 remaining     │
│  Request 11 retries → acquire permit → count: 0 remaining ✓   │
│                                                                 │
│  ┌─────────────────────────────────────────────┐               │
│  │  Thread Pool vs Semaphore:                   │               │
│  │                                              │               │
│  │  Thread Pool:                                │               │
│  │  + Calls execute on a SEPARATE thread        │               │
│  │  + True isolation (can timeout the thread)   │               │
│  │  - Higher overhead (thread creation/context) │               │
│  │                                              │               │
│  │  Semaphore:                                  │               │
│  │  + Lightweight (just a counter)              │               │
│  │  + Runs on the CALLER's thread              │               │
│  │  - Can't timeout a stuck call               │               │
│  │  - Less isolation than thread pools          │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Connection Pool Bulkhead

```
┌─────────────────────────────────────────────────────────────────┐
│             CONNECTION POOL ISOLATION                            │
│                                                                 │
│  Application Server                                             │
│  ┌────────────────────────────────────────┐                    │
│  │                                        │                    │
│  │  ┌─────────────────┐                  │                    │
│  │  │ Order Service    │                  │                    │
│  │  │ DB Pool: 20 conn │──────────────────┼──▶ PostgreSQL     │
│  │  └─────────────────┘                  │    (Orders DB)     │
│  │                                        │                    │
│  │  ┌─────────────────┐                  │                    │
│  │  │ Analytics Service│                  │                    │
│  │  │ DB Pool: 5 conn  │──────────────────┼──▶ PostgreSQL     │
│  │  └─────────────────┘                  │    (Analytics DB)  │
│  │                                        │                    │
│  │  ┌─────────────────┐                  │                    │
│  │  │ Cache Client     │                  │                    │
│  │  │ Pool: 30 conn    │──────────────────┼──▶ Redis          │
│  │  └─────────────────┘                  │                    │
│  │                                        │                    │
│  └────────────────────────────────────────┘                    │
│                                                                 │
│  If Analytics runs a slow query that takes all 5 connections:   │
│  → Order Service still has its own 20 connections (unaffected!) │
│  → Redis cache still has its own 30 connections (unaffected!)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Semaphore Bulkhead

```python
import asyncio
import aiohttp
from typing import Any

class BulkheadSemaphore:
    """Limits concurrent calls to a downstream service."""
    
    def __init__(self, name: str, max_concurrent: int):
        self.name = name
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.max_concurrent = max_concurrent
        self.active_calls = 0
    
    async def execute(self, func, *args, **kwargs) -> Any:
        """Execute function within bulkhead limits."""
        # Try to acquire permit without blocking
        if self.semaphore.locked():
            # All permits taken — reject immediately (don't queue!)
            raise BulkheadFullError(
                f"Bulkhead '{self.name}' is full "
                f"({self.max_concurrent} concurrent calls active)")
        
        async with self.semaphore:
            self.active_calls += 1
            try:
                return await func(*args, **kwargs)
            finally:
                self.active_calls -= 1

class BulkheadFullError(Exception):
    pass

# --- Usage: Separate bulkheads per service ---

payment_bulkhead = BulkheadSemaphore("payment-service", max_concurrent=10)
inventory_bulkhead = BulkheadSemaphore("inventory-service", max_concurrent=20)
notification_bulkhead = BulkheadSemaphore("notification-service", max_concurrent=5)

async def process_order(order_id: str):
    async with aiohttp.ClientSession() as session:
        # Each service call is isolated in its own bulkhead
        try:
            payment = await payment_bulkhead.execute(
                charge_payment, session, order_id)
        except BulkheadFullError:
            return {"error": "Payment service overwhelmed, try again"}
        
        try:
            stock = await inventory_bulkhead.execute(
                reserve_stock, session, order_id)
        except BulkheadFullError:
            # Inventory bulkhead full, but payment was ok
            await refund_payment(order_id)
            return {"error": "Inventory service busy, try again"}
        
        # Notifications are non-critical — fail silently
        try:
            await notification_bulkhead.execute(
                send_confirmation, session, order_id)
        except BulkheadFullError:
            pass  # Non-critical, skip it

async def charge_payment(session, order_id):
    async with session.post(f"http://payment/charge/{order_id}") as resp:
        return await resp.json()

async def reserve_stock(session, order_id):
    async with session.post(f"http://inventory/reserve/{order_id}") as resp:
        return await resp.json()

async def send_confirmation(session, order_id):
    async with session.post(f"http://notify/email/{order_id}") as resp:
        return await resp.json()
```

### Python — Thread Pool Bulkhead

```python
from concurrent.futures import ThreadPoolExecutor, TimeoutError
import requests
from functools import partial

class ThreadPoolBulkhead:
    """Isolates service calls in dedicated thread pools."""
    
    def __init__(self, name: str, max_workers: int, queue_size: int = 10):
        self.name = name
        self.executor = ThreadPoolExecutor(
            max_workers=max_workers,
            thread_name_prefix=f"bulkhead-{name}"
        )
        self.max_workers = max_workers
        self.queue_size = queue_size
    
    def execute(self, func, *args, timeout: float = 10.0, **kwargs):
        """Execute function in isolated thread pool with timeout."""
        future = self.executor.submit(func, *args, **kwargs)
        try:
            return future.result(timeout=timeout)
        except TimeoutError:
            future.cancel()
            raise BulkheadTimeoutError(
                f"Call via bulkhead '{self.name}' timed out after {timeout}s")

class BulkheadTimeoutError(Exception):
    pass

# Create isolated pools for each service
payment_pool = ThreadPoolBulkhead("payment", max_workers=10)
user_pool = ThreadPoolBulkhead("user", max_workers=15)
recommendation_pool = ThreadPoolBulkhead("recommendation", max_workers=5)

def get_user_homepage(user_id: str):
    """Each service call uses its own isolated thread pool."""
    
    # User data — critical, 15 threads available
    user = user_pool.execute(
        requests.get, f"http://user-service/users/{user_id}", timeout=5.0)
    
    # Payment info — 10 threads available
    try:
        payments = payment_pool.execute(
            requests.get, f"http://payment-service/history/{user_id}", timeout=5.0)
    except (BulkheadTimeoutError, Exception):
        payments = None  # Non-critical, show page without payments
    
    # Recommendations — only 5 threads (non-critical)
    try:
        recs = recommendation_pool.execute(
            requests.get, f"http://rec-service/recs/{user_id}", timeout=3.0)
    except (BulkheadTimeoutError, Exception):
        recs = None  # Show default recommendations
    
    return build_homepage(user, payments, recs)
```

### Java — Resilience4j Bulkhead

```java
import io.github.resilience4j.bulkhead.Bulkhead;
import io.github.resilience4j.bulkhead.BulkheadConfig;
import io.github.resilience4j.bulkhead.BulkheadRegistry;
import io.github.resilience4j.bulkhead.ThreadPoolBulkhead;
import io.github.resilience4j.bulkhead.ThreadPoolBulkheadConfig;

import java.time.Duration;
import java.util.concurrent.CompletionStage;

public class BulkheadExample {
    
    // --- Semaphore Bulkhead (lightweight) ---
    public static Bulkhead createSemaphoreBulkhead() {
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(10)           // Max 10 concurrent calls
            .maxWaitDuration(Duration.ZERO)   // Don't queue, reject immediately
            .build();
        
        return BulkheadRegistry.of(config).bulkhead("paymentService");
    }
    
    // --- Thread Pool Bulkhead (stronger isolation) ---
    public static ThreadPoolBulkhead createThreadPoolBulkhead() {
        ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)            // Max 10 threads
            .coreThreadPoolSize(5)            // Core 5 threads
            .queueCapacity(20)               // Queue up to 20 requests
            .keepAliveDuration(Duration.ofSeconds(60))
            .build();
        
        return ThreadPoolBulkhead.of("paymentService", config);
    }
    
    public static void main(String[] args) {
        Bulkhead bulkhead = createSemaphoreBulkhead();
        
        // Wrap a service call with bulkhead protection
        var decoratedCall = Bulkhead.decorateSupplier(bulkhead, () -> {
            // This call is limited to 10 concurrent executions
            return callPaymentService("order-123");
        });
        
        try {
            String result = decoratedCall.get();
            System.out.println("Payment result: " + result);
        } catch (io.github.resilience4j.bulkhead.BulkheadFullException e) {
            // Bulkhead is full — all 10 slots taken
            System.out.println("Payment service overloaded, try later");
        }
    }
}
```

### Java — Spring Boot with Bulkhead Annotation

```java
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    
    // Semaphore bulkhead — max 10 concurrent calls
    @Bulkhead(name = "paymentBulkhead", 
              fallbackMethod = "paymentFallback",
              type = Bulkhead.Type.SEMAPHORE)
    public PaymentResult processPayment(String orderId) {
        return paymentClient.charge(orderId);
    }
    
    // Thread pool bulkhead — runs on isolated threads
    @Bulkhead(name = "inventoryBulkhead", 
              fallbackMethod = "inventoryFallback",
              type = Bulkhead.Type.THREADPOOL)
    public CompletionStage<StockResult> checkStock(String productId) {
        return CompletableFuture.supplyAsync(() -> 
            inventoryClient.getStock(productId));
    }
    
    // Fallback when bulkhead is full
    private PaymentResult paymentFallback(String orderId, Exception ex) {
        return new PaymentResult("queued", "Payment will be processed shortly");
    }
    
    private CompletionStage<StockResult> inventoryFallback(String productId, Exception ex) {
        return CompletableFuture.completedFuture(
            new StockResult(productId, -1, "Stock check unavailable"));
    }
}
```

---

## Infrastructure Examples

### Kubernetes — Pod-Level Isolation

```yaml
# Separate deployments = process-level bulkhead
# payment-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: payment
        resources:
          requests:
            cpu: "500m"        # Guaranteed 0.5 CPU cores
            memory: "512Mi"    # Guaranteed 512MB RAM
          limits:
            cpu: "1000m"       # Max 1 CPU core (can't steal more)
            memory: "1Gi"      # Max 1GB RAM (OOMKilled if exceeded)
---
# notification-deployment.yaml — completely isolated resources
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: notification
        resources:
          requests:
            cpu: "200m"        # Less resources (non-critical)
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

### Kubernetes — Namespace-Level Isolation with Resource Quotas

```yaml
# critical-services-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: critical-services-quota
  namespace: critical-services   # Payment, Orders, etc.
spec:
  hard:
    requests.cpu: "8"           # Total 8 CPUs reserved for critical
    requests.memory: "16Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"
    pods: "50"
---
# non-critical-services-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: non-critical-quota
  namespace: non-critical        # Analytics, Notifications, etc.
spec:
  hard:
    requests.cpu: "4"            # Only 4 CPUs for non-critical
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "30"
```

### Envoy Proxy — Connection-Level Bulkheads

```yaml
# Envoy circuit breaker limits per upstream cluster
clusters:
  - name: payment-service
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections: 100         # Max 100 TCP connections
          max_pending_requests: 50     # Max 50 requests in queue
          max_requests: 200           # Max 200 active requests
          max_retries: 3              # Max 3 concurrent retries
        - priority: HIGH
          max_connections: 200         # High-priority gets more
          max_pending_requests: 100
          max_requests: 500
          
  - name: recommendation-service
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections: 30          # Less resources (non-critical)
          max_pending_requests: 10
          max_requests: 50
          max_retries: 1
```

### Spring Boot — Bulkhead Configuration

```yaml
# application.yml
resilience4j:
  bulkhead:
    instances:
      paymentService:
        max-concurrent-calls: 10      # Max 10 simultaneous calls
        max-wait-duration: 0          # Don't queue, reject immediately
      inventoryService:
        max-concurrent-calls: 20
        max-wait-duration: 500ms      # Wait up to 500ms for a slot
      notificationService:
        max-concurrent-calls: 5       # Non-critical, limited resources
        max-wait-duration: 0
  
  thread-pool-bulkhead:
    instances:
      paymentService:
        max-thread-pool-size: 10
        core-thread-pool-size: 5
        queue-capacity: 20
        keep-alive-duration: 60s
```

---

## Real-World Example

### Netflix's Bulkhead Architecture

Netflix processes billions of API calls per day. They use bulkheads at every level:

```
┌─────────────────────────────────────────────────────────────────────┐
│              NETFLIX BULKHEAD ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Level 1: SERVICE ISOLATION                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Streaming│  │ User Mgmt│  │   Search │  │  Billing │          │
│  │ Service  │  │ Service  │  │  Service │  │  Service │          │
│  │ (own pods│  │ (own pods│  │ (own pods│  │ (own pods│          │
│  │  own DB) │  │  own DB) │  │  own DB) │  │  own DB) │          │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘          │
│                                                                     │
│  Level 2: THREAD POOL ISOLATION (within each service)               │
│  ┌────────────────────────────────────────────┐                    │
│  │  API Service                                │                    │
│  │  ┌────────────┐ ┌────────────┐ ┌─────────┐│                    │
│  │  │Pool: User  │ │Pool: Movie │ │Pool: Rec││                    │
│  │  │Threads: 20 │ │Threads: 30 │ │Threads:10││                   │
│  │  └────────────┘ └────────────┘ └─────────┘│                    │
│  └────────────────────────────────────────────┘                    │
│                                                                     │
│  Level 3: REGION ISOLATION                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                   │
│  │  US-EAST   │  │  US-WEST   │  │  EU-WEST   │                   │
│  │ (isolated  │  │ (isolated  │  │ (isolated  │                   │
│  │  cluster)  │  │  cluster)  │  │  cluster)  │                   │
│  └────────────┘  └────────────┘  └────────────┘                   │
│                                                                     │
│  If the Recommendation engine in US-EAST crashes:                   │
│  - Streaming service: UNAFFECTED ✓                                  │
│  - User management: UNAFFECTED ✓                                    │
│  - Search: UNAFFECTED ✓                                             │
│  - Recommendations in US-WEST: UNAFFECTED ✓                        │
│  - Only US-EAST users get default recommendations                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Amazon's Blast Radius Reduction

Amazon calls it "blast radius" — the amount of damage a single failure can cause:

```
Goal: Minimize blast radius at every level

┌────────────────────────────────────────────────────┐
│  Isolation Level    │  Blast Radius                 │
├─────────────────────┼─────────────────────────────────┤
│  Shared everything  │  💥 ONE failure → ENTIRE site  │
│  Process isolation  │  💥 ONE failure → one service  │
│  Thread pool isolation │ 💥 ONE failure → one feature │
│  Cell-based arch.   │  💥 ONE failure → one cell     │
│                     │     (fraction of customers)    │
└─────────────────────┴─────────────────────────────────┘

Amazon's Cell Architecture:
┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
│Cell1│ │Cell2│ │Cell3│ │Cell4│ │Cell5│
│ 20% │ │ 20% │ │ 20% │ │ 20% │ │ 20% │
│users│ │users│ │users│ │users│ │users│
└─────┘ └─────┘ └─────┘ └─────┘ └─────┘
   💥
  Cell1 failure only affects 20% of users!
```

---

## Common Mistakes / Pitfalls

### 1. Bulkhead Size Too Large

```
❌ BAD: Payment bulkhead = 1000 concurrent calls
   → Might as well have no bulkhead
   → Service still gets overwhelmed

✅ GOOD: Payment bulkhead = 10-20 concurrent calls
   → Based on what the downstream service can actually handle
   → Protects the downstream service too
```

### 2. Bulkhead Size Too Small

```
❌ BAD: Payment bulkhead = 2 concurrent calls
   → Under normal load, requests get rejected constantly
   → Artificial bottleneck causing false failures

✅ GOOD: Size based on downstream capacity + traffic patterns
   → If payment service handles 50 req/s and avg latency is 200ms:
   → Concurrent calls ≈ 50 × 0.2 = 10 slots minimum
```

### 3. Not Having a Fallback When Bulkhead Rejects

```
❌ BAD: Bulkhead full → HTTP 500 → user sees error

✅ GOOD: Bulkhead full → queue for later / return cached data / degrade
```

### 4. Same Bulkhead for Critical and Non-Critical Calls

```
❌ BAD: 
   Payment (critical) and Analytics (non-critical) share one pool
   → Analytics flood blocks payment processing!

✅ GOOD:
   Separate bulkheads with priority:
   - Payment: 20 threads (always available)
   - Analytics: 5 threads (limited, non-critical)
```

### 5. Only Applying Bulkheads at Application Level

```
Bulkheads should exist at EVERY layer:

✅ Application level: Thread pools, semaphores
✅ Infrastructure level: Container resource limits
✅ Network level: Connection limits per service
✅ Database level: Separate connection pools
✅ Cluster level: Separate node groups
```

---

## When to Use / When NOT to Use

### ✅ Use Bulkheads When

| Scenario | Type |
|----------|------|
| Multiple downstream services in one app | Thread pool per service |
| Critical vs non-critical workloads | Separate resource pools |
| Shared database between services | Connection pool isolation |
| Multi-tenant system | Per-tenant resource limits |
| Microservices with varying reliability | Isolate unreliable ones |
| Mix of fast and slow operations | Separate pools by latency |

### ❌ When NOT to Use

| Scenario | Why |
|----------|-----|
| Single downstream dependency | Circuit breaker is sufficient |
| Extremely low traffic | Overhead not worth the isolation |
| Stateless functions (serverless) | Already isolated by platform |
| Simple CRUD apps | Over-engineering for basic needs |

---

## Bulkhead + Circuit Breaker Together

```
┌────────────────────────────────────────────────────────────┐
│         COMPLETE RESILIENCE STACK                           │
│                                                            │
│  Request                                                    │
│    │                                                        │
│    ▼                                                        │
│  ┌──────────────────┐                                      │
│  │  BULKHEAD         │  "Do we have capacity?"             │
│  │  (10 slots max)   │  If NO → reject immediately        │
│  └────────┬──────────┘                                      │
│           │ YES                                              │
│           ▼                                                  │
│  ┌──────────────────┐                                      │
│  │  CIRCUIT BREAKER  │  "Is service healthy?"              │
│  │                   │  If OPEN → fallback                 │
│  └────────┬──────────┘                                      │
│           │ CLOSED                                           │
│           ▼                                                  │
│  ┌──────────────────┐                                      │
│  │  RETRY            │  Try up to 3 times                  │
│  │  (with backoff)   │                                      │
│  └────────┬──────────┘                                      │
│           │                                                  │
│           ▼                                                  │
│  ┌──────────────────┐                                      │
│  │  TIMEOUT          │  Max 5 seconds                      │
│  │                   │                                      │
│  └────────┬──────────┘                                      │
│           │                                                  │
│           ▼                                                  │
│  ┌──────────────────┐                                      │
│  │  SERVICE CALL     │  Actual HTTP/gRPC request           │
│  └──────────────────┘                                      │
│                                                            │
│  Order: Bulkhead → Circuit Breaker → Retry → Timeout       │
│  (Outermost to innermost)                                   │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

- **Bulkheads prevent total system failure** — one overloaded service can't consume all resources and take everything down.
- **Thread pool isolation** gives the strongest guarantees but has overhead; **semaphore isolation** is lightweight but less powerful.
- **Size bulkheads based on downstream capacity** — not too big (useless) and not too small (artificial bottleneck).
- **Apply at multiple levels** — thread pools, connection pools, containers, clusters, and regions.
- **Always pair with fallbacks** — rejected requests should degrade gracefully, not crash.
- **Separate critical from non-critical** — give more resources to payment processing than to analytics.
- **Amazon's cell architecture** is the ultimate bulkhead — isolating failures to a fraction of customers.

---

## What's Next?

When a service call fails — whether due to timeout, circuit breaker tripping, or bulkhead rejection — what do you DO? That's where **Fallback Strategies** come in. See [05-fallback-strategies.md](./05-fallback-strategies.md).
