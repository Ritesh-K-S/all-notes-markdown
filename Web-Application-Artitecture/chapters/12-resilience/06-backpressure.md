# Backpressure — Slowing Down When Overwhelmed

> **What you'll learn**: How systems communicate "I'm overwhelmed, slow down!" to upstream producers — preventing cascading overloads, memory exhaustion, and system crashes through intelligent flow control.

---

## Real-Life Analogy

Imagine a **factory assembly line**:

- **Station A** produces widgets at 100/minute
- **Station B** paints them at 50/minute
- **Station C** packages them at 50/minute

What happens without backpressure?

```
WITHOUT BACKPRESSURE:

Station A (100/min) ──▶ [████████████████████████] ──▶ Station B (50/min)
                         ↑ PILE-UP! ↑
                         Widgets falling off
                         the conveyor belt!
                         
Eventually: Floor covered in unpainted widgets.
            Workers tripping. Factory shuts down.


WITH BACKPRESSURE:

Station A detects Station B is full
Station A SLOWS DOWN to 50/min to match Station B

Station A (50/min) ──▶ [████       ] ──▶ Station B (50/min)
                        ↑ Manageable ↑
                        Smooth flow!
```

In software, **backpressure** is the mechanism by which a slow consumer tells a fast producer: "I can't keep up — slow down or I'll crash."

---

## Core Concept Explained Step-by-Step

### The Problem: Producer-Consumer Speed Mismatch

```
┌─────────────────────────────────────────────────────────────────┐
│            THE SPEED MISMATCH PROBLEM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Producer: 10,000 events/sec                                    │
│  Consumer: 2,000 events/sec                                     │
│  Gap: 8,000 events/sec piling up!                               │
│                                                                 │
│  What happens to the excess 8,000 events every second?          │
│                                                                 │
│  Option A: Buffer in memory                                     │
│    → Memory fills up → OutOfMemoryError → CRASH                │
│                                                                 │
│  Option B: Buffer on disk (queue)                               │
│    → Disk fills up → Messages lost → System broken             │
│    → Latency grows from seconds → minutes → hours              │
│                                                                 │
│  Option C: Drop messages                                        │
│    → Data loss → Incorrect business state                      │
│                                                                 │
│  Option D: BACKPRESSURE                                         │
│    → Tell producer to SLOW DOWN to 2,000/sec                   │
│    → System stays stable ✓                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Backpressure Strategies

```
┌─────────────────────────────────────────────────────────────────┐
│           BACKPRESSURE STRATEGIES                                │
├───────────────┬─────────────────────────────────────────────────┤
│ Strategy      │ How it works                                    │
├───────────────┼─────────────────────────────────────────────────┤
│ DROP          │ Discard excess messages/requests                 │
│               │ (newest or oldest)                              │
│               │ Use: Metrics, non-critical events               │
├───────────────┼─────────────────────────────────────────────────┤
│ BUFFER        │ Store excess in a bounded queue                  │
│               │ Once buffer is full → apply another strategy    │
│               │ Use: Short bursts that even out                 │
├───────────────┼─────────────────────────────────────────────────┤
│ THROTTLE      │ Slow down the producer (rate limiting)          │
│ (Rate Limit)  │ Return 429 Too Many Requests                    │
│               │ Use: API endpoints, user-facing services        │
├───────────────┼─────────────────────────────────────────────────┤
│ BLOCK         │ Block the producer until consumer catches up    │
│               │ Producer thread waits (synchronous)             │
│               │ Use: Pipelines where data loss is unacceptable  │
├───────────────┼─────────────────────────────────────────────────┤
│ SIGNAL        │ Consumer signals capacity to producer           │
│ (Flow Control)│ Producer adjusts its sending rate dynamically   │
│               │ Use: Reactive streams, TCP flow control         │
└───────────────┴─────────────────────────────────────────────────┘
```

### Where Backpressure Applies

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Internet          Your System         Downstream               │
│                                                                 │
│  Users ──▶ Load Balancer ──▶ App Server ──▶ Database           │
│  (fast)    (rate limit)     (queue)       (slow)               │
│                                                                 │
│            ◀── Backpressure ──  ◀── Backpressure ──            │
│            "429 Too Many        "Connection pool               │
│             Requests"            full, wait"                    │
│                                                                 │
│  Event Producer ──▶ Kafka ──▶ Consumer ──▶ Database            │
│  (fast)            (buffer)   (slow)      (slow)               │
│                                                                 │
│                    ◀── Backpressure ──                          │
│                    "Consumer lag                                │
│                     increasing,                                  │
│                     pause producer"                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### TCP Flow Control — The Original Backpressure

TCP has built-in backpressure at the network level:

```
┌─────────────────────────────────────────────────────────────────┐
│              TCP WINDOW-BASED FLOW CONTROL                       │
│                                                                 │
│  Sender                              Receiver                   │
│    │                                    │                       │
│    │  ── Data (1000 bytes) ──────────▶  │                       │
│    │                                    │ Buffer: [████____]    │
│    │  ◀── ACK, Window=4000 ──────────  │ (4KB free)            │
│    │                                    │                       │
│    │  ── Data (4000 bytes) ──────────▶  │                       │
│    │                                    │ Buffer: [████████]    │
│    │  ◀── ACK, Window=0 ─────────────  │ (FULL!)              │
│    │                                    │                       │
│    │  (Sender STOPS sending!)           │                       │
│    │  (Waits for window update)         │ App reads buffer...  │
│    │                                    │                       │
│    │  ◀── Window Update, Window=2000 ─  │ Buffer: [████____]    │
│    │                                    │ (2KB free)            │
│    │  ── Data (2000 bytes) ──────────▶  │ (Sending resumes)    │
│    │                                    │                       │
│                                                                 │
│  Key: The "Window" field tells the sender how much buffer space │
│  the receiver has. When window=0, sender MUST stop.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Reactive Streams — Application-Level Backpressure

```
┌─────────────────────────────────────────────────────────────────┐
│            REACTIVE STREAMS BACKPRESSURE                         │
│                                                                 │
│  Publisher                         Subscriber                   │
│    │                                    │                       │
│    │  ◀── subscribe() ──────────────── │                       │
│    │                                    │                       │
│    │  ── onSubscribe(subscription) ──▶  │                       │
│    │                                    │                       │
│    │  ◀── request(5) ──────────────── │  ← "Give me 5 items" │
│    │                                    │                       │
│    │  ── onNext(item1) ─────────────▶  │                       │
│    │  ── onNext(item2) ─────────────▶  │                       │
│    │  ── onNext(item3) ─────────────▶  │                       │
│    │  ── onNext(item4) ─────────────▶  │                       │
│    │  ── onNext(item5) ─────────────▶  │                       │
│    │                                    │                       │
│    │  (Publisher WAITS — subscriber     │  ← Processing...     │
│    │   hasn't requested more)           │                       │
│    │                                    │                       │
│    │  ◀── request(3) ──────────────── │  ← "Ready for 3 more"│
│    │                                    │                       │
│    │  ── onNext(item6) ─────────────▶  │                       │
│    │  ── onNext(item7) ─────────────▶  │                       │
│    │  ── onNext(item8) ─────────────▶  │                       │
│                                                                 │
│  PULL-based: Consumer controls the speed!                       │
│  Publisher never overwhelms subscriber.                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Message Queue Backpressure

```
┌─────────────────────────────────────────────────────────────────┐
│           KAFKA CONSUMER LAG AS BACKPRESSURE SIGNAL              │
│                                                                 │
│  Producer → [Kafka Topic] → Consumer                            │
│             (offset tracking)                                   │
│                                                                 │
│  Latest Offset:  1,000,000  (newest message)                    │
│  Consumer Offset:   900,000  (last processed)                   │
│  LAG:              100,000  messages behind!                    │
│                                                                 │
│  Actions based on lag:                                          │
│                                                                 │
│  Lag < 1000    → Normal ✓ (consumer keeping up)                │
│  Lag 1K-10K   → ⚠️ Scale up consumers                         │
│  Lag > 100K   → 🚨 ALERT! Consider:                           │
│                   - Pause producer                              │
│                   - Add more consumer instances                 │
│                   - Increase batch size                         │
│                   - Drop non-critical events                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Bounded Queue with Backpressure

```python
import asyncio
import logging
from typing import Any, Callable

logger = logging.getLogger(__name__)

class BackpressureQueue:
    """
    A bounded async queue that applies backpressure
    when the consumer can't keep up.
    """
    
    def __init__(self, max_size: int = 1000, high_watermark: float = 0.8):
        self.queue = asyncio.Queue(maxsize=max_size)
        self.max_size = max_size
        self.high_watermark = int(max_size * high_watermark)
        self._paused = False
    
    async def produce(self, item: Any) -> bool:
        """
        Add item to queue. Blocks if queue is full (backpressure).
        Returns False if queue is at high watermark (warning signal).
        """
        current_size = self.queue.qsize()
        
        if current_size >= self.high_watermark and not self._paused:
            self._paused = True
            logger.warning(
                f"Queue at {current_size}/{self.max_size} "
                f"({current_size/self.max_size*100:.0f}%). "
                f"Backpressure active!")
        
        # This will BLOCK if queue is full — natural backpressure
        await self.queue.put(item)
        return not self._paused
    
    async def consume(self) -> Any:
        """Get next item. Resumes producer if queue drops below watermark."""
        item = await self.queue.get()
        
        if self._paused and self.queue.qsize() < self.high_watermark * 0.5:
            self._paused = False
            logger.info("Queue below low watermark. Backpressure released.")
        
        return item
    
    @property
    def is_under_pressure(self) -> bool:
        return self._paused


# --- Usage: Event processing pipeline ---

async def event_producer(queue: BackpressureQueue):
    """Produces events. Naturally slowed by queue backpressure."""
    event_count = 0
    while True:
        event = {"id": event_count, "type": "page_view", "data": "..."}
        
        # This blocks when queue is full — producer automatically slows down
        can_continue = await queue.produce(event)
        
        if not can_continue:
            # Optional: log that we're being throttled
            await asyncio.sleep(0.1)  # Brief pause to help consumer catch up
        
        event_count += 1

async def event_consumer(queue: BackpressureQueue):
    """Consumes events at its own pace. Slow processing is fine."""
    while True:
        event = await queue.consume()
        # Simulate slow processing (e.g., DB write)
        await asyncio.sleep(0.05)  # 50ms per event = 20 events/sec
        process_event(event)

async def main():
    queue = BackpressureQueue(max_size=500, high_watermark=0.8)
    await asyncio.gather(
        event_producer(queue),
        event_consumer(queue),
        event_consumer(queue),  # Two consumers to increase throughput
    )
```

### Python — HTTP API Rate Limiting (Backpressure via 429)

```python
import time
from collections import defaultdict
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse

app = FastAPI()

class TokenBucketRateLimiter:
    """
    Token bucket rate limiter — applies backpressure to HTTP clients.
    Returns 429 when bucket is empty, telling clients to slow down.
    """
    
    def __init__(self, rate: float, capacity: int):
        self.rate = rate          # Tokens added per second
        self.capacity = capacity  # Max tokens (burst size)
        self.buckets = defaultdict(lambda: {"tokens": capacity, "last": time.time()})
    
    def allow_request(self, client_id: str) -> tuple[bool, dict]:
        bucket = self.buckets[client_id]
        now = time.time()
        
        # Refill tokens based on elapsed time
        elapsed = now - bucket["last"]
        bucket["tokens"] = min(
            self.capacity,
            bucket["tokens"] + elapsed * self.rate
        )
        bucket["last"] = now
        
        if bucket["tokens"] >= 1:
            bucket["tokens"] -= 1
            return True, {}
        else:
            # Calculate when client can retry
            retry_after = (1 - bucket["tokens"]) / self.rate
            return False, {"retry_after": retry_after}

limiter = TokenBucketRateLimiter(rate=10, capacity=20)  # 10 req/s, burst 20

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    client_id = request.client.host
    allowed, info = limiter.allow_request(client_id)
    
    if not allowed:
        # BACKPRESSURE: Tell client to slow down
        return JSONResponse(
            status_code=429,
            content={"error": "Too Many Requests", "message": "Slow down!"},
            headers={
                "Retry-After": str(int(info["retry_after"] + 1)),
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(int(time.time() + info["retry_after"]))
            }
        )
    
    return await call_next(request)
```

### Java — Reactive Streams Backpressure (Project Reactor)

```java
import reactor.core.publisher.Flux;
import reactor.core.scheduler.Schedulers;
import java.time.Duration;

public class ReactiveBackpressureExample {
    
    public static void main(String[] args) throws InterruptedException {
        
        // Fast producer: emits 1000 items/second
        Flux<Long> fastProducer = Flux.interval(Duration.ofMillis(1))
            .doOnNext(i -> {
                if (i % 1000 == 0) System.out.println("Produced: " + i);
            });
        
        // Slow consumer: processes 10 items/second
        fastProducer
            .onBackpressureBuffer(500)  // Buffer up to 500 items
            .onBackpressureDrop(item -> {
                // When buffer is full, DROP excess items
                System.out.println("DROPPED (backpressure): " + item);
            })
            .publishOn(Schedulers.boundedElastic())
            .subscribe(item -> {
                // Slow processing (100ms per item)
                try { Thread.sleep(100); } catch (Exception e) {}
                System.out.println("Consumed: " + item);
            });
        
        Thread.sleep(10000);  // Run for 10 seconds
    }
}
```

### Java — Kafka Consumer with Pause/Resume Backpressure

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;
import java.time.Duration;
import java.util.*;

public class BackpressureKafkaConsumer {
    
    private static final int MAX_PENDING = 1000;
    private final Queue<ConsumerRecord<String, String>> processingQueue = 
        new LinkedList<>();
    private boolean paused = false;
    
    public void consume() {
        Properties props = new Properties();
        props.put("bootstrap.servers", "kafka:9092");
        props.put("group.id", "order-processor");
        props.put("enable.auto.commit", "false");
        props.put("max.poll.records", "100");  // Limit batch size
        
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(List.of("orders"));
        
        while (true) {
            // BACKPRESSURE: If processing queue is too full, pause consuming
            if (processingQueue.size() > MAX_PENDING && !paused) {
                System.out.println("⚠️ Backpressure: PAUSING Kafka consumption");
                consumer.pause(consumer.assignment());
                paused = true;
            }
            
            // RESUME: If queue drains below threshold, resume
            if (processingQueue.size() < MAX_PENDING / 2 && paused) {
                System.out.println("✓ Backpressure released: RESUMING consumption");
                consumer.resume(consumer.assignment());
                paused = false;
            }
            
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
            
            for (ConsumerRecord<String, String> record : records) {
                processingQueue.add(record);
            }
            
            // Process items from queue (at the consumer's own pace)
            processNextBatch();
            
            // Commit after successful processing
            if (!records.isEmpty()) {
                consumer.commitSync();
            }
        }
    }
    
    private void processNextBatch() {
        int batchSize = Math.min(10, processingQueue.size());
        for (int i = 0; i < batchSize; i++) {
            ConsumerRecord<String, String> record = processingQueue.poll();
            if (record != null) {
                processOrder(record);  // Slow operation
            }
        }
    }
}
```

---

## Infrastructure Examples

### Nginx — Connection Limiting (HTTP Backpressure)

```nginx
# Limit concurrent connections per client IP
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
limit_req_zone $binary_remote_addr zone=req_limit:10m rate=10r/s;

server {
    listen 80;
    
    # Max 10 concurrent connections per IP
    limit_conn conn_limit 10;
    
    # Max 10 requests/second per IP, burst of 20
    limit_req zone=req_limit burst=20 nodelay;
    
    # When limit is hit, return 429 (backpressure signal)
    limit_req_status 429;
    limit_conn_status 429;
    
    location /api/ {
        proxy_pass http://backend;
    }
}
```

### Kafka — Producer Backpressure Configuration

```properties
# kafka-producer.properties

# Buffer memory for unsent records — when full, producer blocks
buffer.memory=33554432          # 32MB buffer
# How long producer blocks when buffer is full (backpressure)
max.block.ms=60000             # Block for up to 60 seconds

# Batch settings — larger batches = less pressure on broker
batch.size=16384               # 16KB batch
linger.ms=5                    # Wait 5ms to fill batch

# Limit in-flight requests (flow control)
max.in.flight.requests.per.connection=5

# If broker can't keep up:
# - buffer.memory fills up
# - send() blocks for up to max.block.ms
# - If still blocked → throws TimeoutException (explicit backpressure)
```

### RabbitMQ — Publisher Confirms & Flow Control

```python
import pika

# RabbitMQ has built-in flow control:
# When queues fill up, it stops accepting messages from publishers

connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='rabbitmq',
        # Connection will block if RabbitMQ applies flow control
        blocked_connection_timeout=30  # Wait up to 30s if flow-controlled
    )
)

channel = connection.channel()

# Enable publisher confirms — broker acknowledges each message
channel.confirm_delivery()

# Prefetch = consumer-side backpressure
# "Only send me 10 unacknowledged messages at a time"
channel.basic_qos(prefetch_count=10)

def publish_with_backpressure(message):
    try:
        channel.basic_publish(
            exchange='orders',
            routing_key='new-orders',
            body=message,
            mandatory=True  # Return message if no queue can accept it
        )
    except pika.exceptions.ConnectionBlockedTimeout:
        # RabbitMQ is applying backpressure — we must slow down
        print("RabbitMQ flow control active! Slowing down...")
        time.sleep(5)
```

### Kubernetes — Resource Quotas as Backpressure

```yaml
# Limit total requests a namespace can make to the API server
apiVersion: v1
kind: LimitRange
metadata:
  name: request-limits
  namespace: high-traffic-service
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
    max:
      cpu: "2"
      memory: "2Gi"
---
# HPA with stabilization — prevents too-rapid scaling (implicit backpressure)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # Wait 60s before scaling up more
      policies:
      - type: Percent
        value: 50                      # Scale up max 50% at a time
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
```

---

## Real-World Example

### Netflix — Adaptive Concurrency Limits

Netflix developed **concurrency-limits** — a library that dynamically adjusts how many requests a server accepts based on latency signals:

```
┌─────────────────────────────────────────────────────────────────────┐
│           NETFLIX ADAPTIVE CONCURRENCY LIMITS                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Concept: Server measures its own latency.                          │
│           If latency increases → reduce concurrency limit           │
│           If latency is good → increase concurrency limit           │
│                                                                     │
│  Time 0:   Concurrency limit = 100, Latency = 20ms  ✓             │
│  Time 1:   Concurrency limit = 100, Latency = 50ms  ⚠️            │
│  Time 2:   Concurrency limit = 80,  Latency = 30ms  ← reduced!   │
│  Time 3:   Concurrency limit = 80,  Latency = 25ms  ✓             │
│  Time 4:   Concurrency limit = 90,  Latency = 22ms  ← growing    │
│  Time 5:   Concurrency limit = 100, Latency = 20ms  ✓ stable     │
│                                                                     │
│  Algorithm: Vegas-style (based on TCP Vegas congestion control)      │
│                                                                     │
│  limit = current_limit × (1 - gradient)                             │
│  gradient = (RTT_noload - RTT_actual) / RTT_actual                 │
│                                                                     │
│  When requests exceed the limit → return 503 (backpressure)         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Uber — Load Shedding with QALM

```
┌─────────────────────────────────────────────────────────────────────┐
│              UBER'S QALM (Queue And Load Management)                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Request arrives                                                    │
│       │                                                             │
│       ▼                                                             │
│  ┌──────────────┐                                                  │
│  │ Check load    │                                                  │
│  │ (CPU, queue   │                                                  │
│  │  depth, p99)  │                                                  │
│  └──────┬────────┘                                                  │
│         │                                                            │
│    Load OK? ──────── YES ──▶ Process request normally              │
│         │                                                            │
│        NO (overloaded)                                              │
│         │                                                            │
│         ▼                                                            │
│  ┌──────────────┐                                                  │
│  │ Priority?     │                                                  │
│  └──────┬────────┘                                                  │
│         │                                                            │
│    HIGH? ──────────── YES ──▶ Process (critical path)              │
│         │                                                            │
│        LOW                                                           │
│         │                                                            │
│         ▼                                                            │
│  Return 503 + Retry-After                                           │
│  (Shed non-critical load to protect critical paths)                 │
│                                                                     │
│  Example:                                                           │
│  - Ride matching: HIGH priority (never shed)                       │
│  - ETA estimates: MEDIUM priority (shed under extreme load)         │
│  - Analytics tracking: LOW priority (shed first)                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

### 1. Unbounded Buffers/Queues

```
❌ BAD: Queue with no max size
   queue = []   # List grows forever!
   
   Result: Memory grows until OOM → crash

✅ GOOD: Bounded queue with explicit overflow strategy
   queue = asyncio.Queue(maxsize=1000)  # Blocks when full
```

### 2. Backpressure Only at One Layer

```
❌ BAD: Rate limit at API Gateway, but nothing between services

  API Gateway (rate limited) → Service A → Service B → Database
                                            ↑
                                     No backpressure here!
                                     Service B overwhelms DB

✅ GOOD: Backpressure at EVERY boundary

  API Gateway → Service A → Service B → Database
  (rate limit)  (bulkhead)   (conn pool)  (max connections)
```

### 3. Drop Strategy Without Monitoring

```
❌ BAD: Silently dropping messages when overwhelmed
   → You don't know how much data you're losing
   → Business impact invisible until customer complains

✅ GOOD: Track and alert on drop rate
   counter("messages_dropped_total").increment()
   if drop_rate > 5%: alert("High message drop rate!")
```

### 4. Not Propagating Backpressure Upstream

```
❌ BAD: Consumer is slow, but producer doesn't know
   Producer → [HUGE BUFFER] → Consumer (slow)
              Gets bigger and bigger...

✅ GOOD: Consumer communicates back to producer
   Consumer → "I'm full! Pause!" → Producer (pauses)
   Consumer → "OK, ready!" → Producer (resumes)
```

### 5. Aggressive Backpressure Causing Cascading Rejection

```
❌ BAD: Service B returns 503 → Service A retries immediately → B gets more traffic

✅ GOOD: Service B returns 503 + Retry-After: 30
         Service A respects Retry-After header
         Service A activates circuit breaker after repeated 503s
```

---

## When to Use / When NOT to Use

### ✅ Use Backpressure When

| Scenario | Strategy |
|----------|----------|
| API receiving too many requests | Rate limiting (429) |
| Event stream consumer falling behind | Kafka pause/resume |
| Database connection pool full | Block/reject new queries |
| Memory growing due to buffered messages | Bounded queues |
| Microservice call chain overloaded | Adaptive concurrency |
| Stream processing falling behind | Drop/sample old events |

### ❌ When NOT to Use (or Use Carefully)

| Scenario | Why |
|----------|-----|
| Critical financial transactions | Can't drop or delay payments |
| Real-time safety systems | Latency budget is zero |
| Very short bursts (<1 second) | Buffer handles it naturally |
| Already using async queue (Kafka) | Kafka IS your backpressure buffer |

---

## Key Takeaways

- **Backpressure is how systems avoid drowning** — it's the "slow down" signal between components with different speeds.
- **TCP has built-in backpressure** (window sizing) — application layers need their own mechanisms.
- **Bounded queues are essential** — unbounded queues just move the OOM crash to later.
- **The 4 strategies**: Drop, Buffer, Throttle, Block — choose based on your data's criticality.
- **Apply at every boundary** — between services, between threads, between producers and consumers.
- **Monitor queue depth and lag** — they're the earliest indicators of an approaching overload.
- **Reactive Streams (Project Reactor, RxJava)** formalize backpressure with request(n) semantics.

---

## What's Next?

Backpressure helps when systems are overwhelmed. But when you DO process requests (especially retries), how do you ensure that processing the same request twice doesn't cause damage? That's **Idempotency** — see [07-idempotency.md](./07-idempotency.md).
