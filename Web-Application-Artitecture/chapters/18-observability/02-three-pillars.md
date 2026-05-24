# The Three Pillars of Observability: Logs, Metrics & Traces

> **What you'll learn**: The three fundamental types of telemetry data — logs, metrics, and traces — what each one does, how they differ, and how they work together to give you complete visibility into your system.

---

## Real-Life Analogy

Imagine you're a detective investigating why a restaurant lost customers last Tuesday night.

You have three sources of evidence:

1. **📝 Logs (The Diary)** — The restaurant's detailed diary: "7:03 PM - Order #456 placed. 7:15 PM - Chef burned the steak for order #456. 7:18 PM - Customer complained. 7:20 PM - Manager offered free dessert." Every individual event, in detail.

2. **📊 Metrics (The Dashboard)** — The restaurant's dashboard numbers: "Average wait time: 45 min (normally 20). Orders per hour: 12 (normally 40). Customer complaints: 8 (normally 1)." Numbers that tell you something is wrong at a glance.

3. **🔍 Traces (The Journey Map)** — Following one specific customer's experience from start to finish: "Entered → waited 10 min for table → ordered → waited 35 min for food → food was cold → complained → left bad review." A single request's full journey through the system.

**Together**, these three give you complete visibility:
- **Metrics** tell you SOMETHING is wrong (wait times are up!)
- **Logs** tell you WHAT went wrong (the chef burned 8 orders)
- **Traces** tell you HOW it affected specific customers (customer X waited 35 min)

---

## The Three Pillars at a Glance

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE THREE PILLARS OF OBSERVABILITY               │
│                                                                   │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐   │
│  │     LOGS      │    │    METRICS    │    │    TRACES     │   │
│  │               │    │               │    │               │   │
│  │  "What        │    │  "How is      │    │  "What path   │   │
│  │   happened?"  │    │   the system  │    │   did this    │   │
│  │               │    │   performing?"│    │   request     │   │
│  │  Individual   │    │               │    │   take?"      │   │
│  │  events with  │    │  Numeric      │    │               │   │
│  │  rich context │    │  measurements │    │  End-to-end   │   │
│  │               │    │  over time    │    │  journey of   │   │
│  │  Text-based   │    │               │    │  a single     │   │
│  │  Immutable    │    │  Aggregatable │    │  request      │   │
│  │  High-volume  │    │  Compact      │    │               │   │
│  └───────────────┘    └───────────────┘    └───────────────┘   │
│                                                                   │
│  BEST FOR:            BEST FOR:            BEST FOR:            │
│  Debugging,           Alerting,            Understanding        │
│  Auditing,            Dashboards,          cross-service        │
│  Forensics            Trends               dependencies         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Pillar 1: Logs — The Detailed Record

### What Are Logs?

A **log** is a timestamped record of a discrete event. Every time something noteworthy happens in your application, you write a log.

```
2024-03-15T14:32:01.234Z [INFO]  user_service | User "alice" logged in from IP 192.168.1.5
2024-03-15T14:32:01.567Z [ERROR] payment_service | Payment failed for order #789 - Card declined
2024-03-15T14:32:02.001Z [WARN]  db_connection | Connection pool at 85% capacity (17/20 connections used)
```

### Log Levels

```
┌──────────┬─────────────────────────────────────────────────────────┐
│  Level   │  When to Use                                            │
├──────────┼─────────────────────────────────────────────────────────┤
│  TRACE   │  Ultra-verbose: every function call (dev only)          │
│  DEBUG   │  Detailed info for debugging (usually off in prod)      │
│  INFO    │  Normal operations: "User logged in", "Order placed"    │
│  WARN    │  Something unexpected but handled: "Retry succeeded"    │
│  ERROR   │  Something failed: "Payment declined", "DB timeout"     │
│  FATAL   │  System is crashing: "Out of memory", "Cannot start"   │
└──────────┴─────────────────────────────────────────────────────────┘

Production typically logs: INFO, WARN, ERROR, FATAL
Development typically logs: DEBUG and above
```

### Structured vs Unstructured Logs

```
UNSTRUCTURED (hard to parse):
  "User alice from 192.168.1.5 failed login attempt #3"

STRUCTURED (easy to query and filter):
  {
    "timestamp": "2024-03-15T14:32:01Z",
    "level": "WARN",
    "service": "auth-service",
    "event": "login_failed",
    "user": "alice",
    "ip": "192.168.1.5",
    "attempt": 3,
    "reason": "invalid_password"
  }
```

> **Always use structured logs in production.** They allow you to search, filter, and aggregate: "Show me all failed logins from IP 192.168.1.5 in the last hour."

---

## Pillar 2: Metrics — The Numbers

### What Are Metrics?

A **metric** is a numeric measurement captured at a specific point in time. Unlike logs (text about individual events), metrics are numbers that you can aggregate, graph, and alert on.

```
http_requests_total{method="GET", status="200"} = 1,523,847
http_request_duration_seconds{quantile="0.99"} = 0.45
system_cpu_usage_percent = 73.2
active_database_connections = 18
```

### Types of Metrics

```
┌──────────────────────────────────────────────────────────────────┐
│                      METRIC TYPES                                 │
│                                                                    │
│  COUNTER          GAUGE            HISTOGRAM        SUMMARY       │
│  ────────         ─────            ─────────        ───────       │
│  Only goes UP     Goes UP or DOWN  Buckets of       Pre-computed  │
│                                    values           percentiles   │
│     ╱              /\                                             │
│    ╱              /  \  /\         [0-100ms]: 45%                 │
│   ╱              /    \/  \        [100-500ms]: 35%  p50: 120ms  │
│  ╱                                 [500ms-1s]: 15%   p99: 890ms  │
│                                    [1s+]: 5%                      │
│                                                                    │
│  Examples:        Examples:        Examples:         Examples:     │
│  • Total         • CPU usage      • Response        • Latency    │
│    requests      • Memory used      time             percentiles │
│  • Total         • Active           distribution                  │
│    errors          connections    • Request                       │
│  • Bytes         • Queue depth      size                         │
│    transferred                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Why Metrics Are Powerful

Metrics are compact, fast, and aggregatable:

```
1 million log entries about requests = ~500 MB of storage
1 million requests as metrics       = ~50 KB of storage (just the numbers)

This is why metrics are used for:
  • Real-time dashboards (fast to query)
  • Alerting (cheap to evaluate rules)
  • Trend analysis (weeks/months of history)
```

---

## Pillar 3: Traces — The Journey Map

### What Are Traces?

A **trace** follows a single request as it travels through your entire system — across multiple services, databases, caches, and queues.

```
User clicks "Place Order"
        │
        ▼
┌─── Trace ID: abc-123 ──────────────────────────────────────────┐
│                                                                  │
│  [API Gateway] ──(4ms)──▶ [Order Service] ──(15ms)──▶ [DB]    │
│                                │                                │
│                                ├──(8ms)──▶ [Inventory Service] │
│                                │               │                │
│                                │               └──(3ms)──▶[DB] │
│                                │                                │
│                                └──(120ms)──▶ [Payment Service] │
│                                                   │             │
│                                                   └──(95ms)──▶ │
│                                                   [Stripe API] │
│                                                                  │
│  Total Duration: 245ms                                          │
│  Bottleneck: Payment Service → Stripe API (95ms)               │
└──────────────────────────────────────────────────────────────────┘
```

### Trace Anatomy: Spans

A trace is made up of **spans**. Each span represents one unit of work:

```
Trace (abc-123)
├── Span 1: API Gateway (4ms)
│   └── Span 2: Order Service (150ms)
│       ├── Span 3: Validate Order (2ms)
│       ├── Span 4: Check Inventory (25ms)
│       │   └── Span 5: DB Query (3ms)
│       ├── Span 6: Process Payment (120ms)
│       │   └── Span 7: Call Stripe (95ms)  ← BOTTLENECK!
│       └── Span 8: Save Order to DB (15ms)
```

Each span contains:
- **Operation name** (what it did)
- **Start time & duration** (how long it took)
- **Tags/attributes** (metadata like user_id, http.status)
- **Parent span ID** (to build the tree)

---

## How the Three Pillars Work Together

```
SCENARIO: "Users report slow checkout"

Step 1: METRICS tell you there's a problem
  ┌─────────────────────────────────────┐
  │ Dashboard shows:                     │
  │ • p99 latency jumped from 200ms     │
  │   to 3,000ms at 14:30              │
  │ • Payment service error rate: 12%   │
  └─────────────────────────────────────┘
         │
         ▼ "Let me look at traces for slow requests"

Step 2: TRACES show you WHERE the problem is
  ┌─────────────────────────────────────┐
  │ Trace shows:                         │
  │ • API Gateway: 5ms ✓               │
  │ • Order Service: 10ms ✓            │
  │ • Payment Service: 2,900ms ✗       │
  │   └── DB Query: 2,850ms ← HERE!   │
  └─────────────────────────────────────┘
         │
         ▼ "Let me check the payment DB logs"

Step 3: LOGS tell you WHAT went wrong
  ┌─────────────────────────────────────┐
  │ Logs show:                           │
  │ 14:29:55 [WARN] Connection pool     │
  │   exhausted, waiting for free conn  │
  │ 14:30:01 [ERROR] Lock timeout on    │
  │   payments table, retrying...       │
  │ 14:30:02 [INFO] Long-running        │
  │   migration query blocking locks    │
  └─────────────────────────────────────┘

ROOT CAUSE: A database migration locked the payments table!
```

---

## How It Works Internally

### Correlation: Tying the Three Pillars Together

The magic is in **correlation IDs** that link logs, metrics, and traces:

```
Every request gets a unique Trace ID (e.g., "abc-123")

This ID appears in:
  • LOGS:    {"trace_id": "abc-123", "message": "Payment failed..."}
  • METRICS: http_errors{trace_id="abc-123"} (as an exemplar)
  • TRACES:  Trace abc-123 → Span 1 → Span 2 → ...

This lets you:
  1. See a spike in error metrics
  2. Click a specific error → jump to the trace
  3. From the trace → jump to related logs
```

### OpenTelemetry: The Unified Standard

**OpenTelemetry (OTel)** is the industry standard for collecting all three signals:

```
┌──────────────────────────────────────────────────────────────┐
│                    YOUR APPLICATION                            │
│                                                                │
│  ┌─────────────────────────────────────────────────┐         │
│  │         OpenTelemetry SDK                        │         │
│  │                                                   │         │
│  │  Auto-instrumentation:                           │         │
│  │  • HTTP requests → traces + metrics             │         │
│  │  • DB queries → traces + metrics                │         │
│  │  • Log bridge → structured logs                 │         │
│  └───────────────────────┬───────────────────────────┘         │
│                          │                                     │
└──────────────────────────┼─────────────────────────────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │   OTel Collector       │
              │   (receives, processes,│
              │    exports telemetry)  │
              └─────────┬──────────────┘
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
   ┌────────────┐ ┌──────────┐ ┌──────────┐
   │   Jaeger   │ │Prometheus│ │   Loki   │
   │  (traces)  │ │(metrics) │ │  (logs)  │
   └────────────┘ └──────────┘ └──────────┘
```

---

## Code Examples

### Python: Emitting All Three Signals with OpenTelemetry

```python
# three_pillars_example.py
# Shows how to emit logs, metrics, and traces from one application

import logging
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider

# --- Setup ---
tracer = trace.get_tracer("order-service")
meter = metrics.get_meter("order-service")
logger = logging.getLogger("order-service")

# Define metrics
order_counter = meter.create_counter(
    "orders.placed.total",
    description="Total orders placed"
)
order_duration = meter.create_histogram(
    "orders.processing.duration",
    description="Time to process an order (ms)"
)

def place_order(user_id: str, items: list):
    """Process an order — demonstrates all three pillars."""
    
    # TRACE: Start a new span for this operation
    with tracer.start_as_current_span("place_order") as span:
        span.set_attribute("user.id", user_id)
        span.set_attribute("order.item_count", len(items))
        
        # LOG: Record what's happening
        logger.info(
            "Processing order",
            extra={"user_id": user_id, "items": len(items)}
        )
        
        # Simulate sub-operations (child spans)
        with tracer.start_as_current_span("validate_inventory"):
            # Check stock for each item
            logger.debug(f"Checking inventory for {len(items)} items")
        
        with tracer.start_as_current_span("charge_payment"):
            # Process payment
            logger.info(f"Charging payment for user {user_id}")
        
        # METRIC: Increment the counter
        order_counter.add(1, {"user_id": user_id, "status": "success"})
        
        # METRIC: Record processing duration
        order_duration.record(245, {"user_id": user_id})
        
        logger.info(f"Order completed for user {user_id}")

# Usage
place_order("user_42", ["laptop", "mouse", "keyboard"])
```

### Java: All Three Pillars with OpenTelemetry

```java
// OrderService.java
// Demonstrates logs, metrics, and traces in Java with OpenTelemetry

import io.opentelemetry.api.trace.*;
import io.opentelemetry.api.metrics.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OrderService {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);
    private static final Tracer tracer = GlobalOpenTelemetry.getTracer("order-service");
    private static final Meter meter = GlobalOpenTelemetry.getMeter("order-service");
    
    // Metrics
    private final LongCounter orderCounter = meter
        .counterBuilder("orders.placed.total")
        .setDescription("Total orders placed")
        .build();
    
    private final DoubleHistogram orderDuration = meter
        .histogramBuilder("orders.processing.duration")
        .setDescription("Order processing time in ms")
        .build();
    
    public void placeOrder(String userId, List<String> items) {
        // TRACE: Create a span for the entire operation
        Span span = tracer.spanBuilder("place_order")
            .setAttribute("user.id", userId)
            .setAttribute("order.item_count", items.size())
            .startSpan();
        
        try (var scope = span.makeCurrent()) {
            long startTime = System.currentTimeMillis();
            
            // LOG: Structured log entry
            logger.info("Processing order | user={} items={}", userId, items.size());
            
            // Child span: validate inventory
            Span inventorySpan = tracer.spanBuilder("validate_inventory").startSpan();
            validateInventory(items);
            inventorySpan.end();
            
            // Child span: process payment
            Span paymentSpan = tracer.spanBuilder("charge_payment").startSpan();
            chargePayment(userId);
            paymentSpan.end();
            
            // METRIC: Record success
            long duration = System.currentTimeMillis() - startTime;
            orderCounter.add(1, Attributes.of(AttributeKey.stringKey("status"), "success"));
            orderDuration.record(duration);
            
            logger.info("Order completed | user={} duration={}ms", userId, duration);
            
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            logger.error("Order failed | user={} error={}", userId, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## Comparison Table: When to Use What

| Scenario | Use Logs | Use Metrics | Use Traces |
|----------|----------|-------------|------------|
| "Is the system healthy right now?" | | ✅ | |
| "What exactly went wrong at 3:15 PM?" | ✅ | | |
| "Which service is the bottleneck?" | | | ✅ |
| "How has latency changed over 30 days?" | | ✅ | |
| "Show me the full path of this failed request" | | | ✅ |
| "What was the error message?" | ✅ | | |
| "Are we getting more errors than yesterday?" | | ✅ | |
| "What did user X experience?" | ✅ | | ✅ |
| "Set up an alert for error rate > 5%" | | ✅ | |
| "Audit trail for compliance" | ✅ | | |

---

## Real-World Example

### How Google Uses the Three Pillars (Dapper, Monarch, Cloud Logging)

Google invented modern distributed tracing with **Dapper** (2010), which inspired Zipkin, Jaeger, and OpenTelemetry.

```
┌─────────────────────────────────────────────────────────┐
│              GOOGLE'S OBSERVABILITY STACK                 │
│                                                          │
│  LOGS:     Cloud Logging (formerly Stackdriver)         │
│            • Petabytes of logs per day                  │
│            • Sub-second search across all services      │
│                                                          │
│  METRICS:  Monarch (internal) / Cloud Monitoring        │
│            • Billions of time series                    │
│            • Custom query language (MQL)               │
│                                                          │
│  TRACES:   Dapper → Cloud Trace                         │
│            • Traces every single request                │
│            • Adaptive sampling at scale                 │
│                                                          │
│  CORRELATION:                                           │
│  • Every request gets a trace ID                       │
│  • Logs automatically tagged with trace ID             │
│  • Metrics linked to trace exemplars                   │
│  • One click from metric spike → trace → logs          │
└─────────────────────────────────────────────────────────┘
```

### Uber's Observability at Scale

Uber processes **100+ million trips** and monitors:
- **Metrics**: M3 (custom time-series platform) — 500 billion data points/day
- **Traces**: Jaeger (which Uber created!) — samples 0.1% of traces in production
- **Logs**: Centralized ELK → migrated to custom solution for cost

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| **Logging everything at DEBUG in production** | Storage costs explode, signal buried in noise | Use INFO in prod, DEBUG only when investigating |
| **Not using structured logs** | Can't search or filter effectively | Always log as JSON with consistent fields |
| **Only using metrics, skipping traces** | Know something is slow but not WHERE | Add tracing to find bottlenecks across services |
| **No correlation between pillars** | Can't jump from alert → trace → logs | Add trace_id to all log entries |
| **Sampling traces too aggressively** | Miss rare but important errors | Keep 100% of error traces, sample success |
| **Storing raw logs forever** | Costs go through the roof | Set retention policies (7 days hot, 30 days cold) |
| **Metrics with too many labels** | Cardinality explosion → slow queries | Never use user_id or request_id as metric labels |

---

## When to Use / When NOT to Use

### Logs: Use When...
- ✅ You need detailed context about a specific event
- ✅ Compliance/audit requirements (who did what, when)
- ✅ Debugging a specific user's issue
- ❌ Don't use for real-time alerting (too slow to query at scale)
- ❌ Don't log sensitive data (PII, passwords, tokens)

### Metrics: Use When...
- ✅ You need to detect anomalies quickly (alerting)
- ✅ You want trends over time (dashboards)
- ✅ You need to compare before/after a deployment
- ❌ Don't use for detailed debugging (not enough context)
- ❌ Don't add high-cardinality labels (user_id, email)

### Traces: Use When...
- ✅ You have multiple services (microservices)
- ✅ You need to find performance bottlenecks
- ✅ You want to understand request flow
- ❌ Don't use in simple monoliths (overkill)
- ❌ Don't trace 100% of traffic at high scale (too expensive)

---

## Key Takeaways

1. **Three pillars = complete picture**: Logs give detail, Metrics give trends, Traces give flow. No single pillar is sufficient alone.

2. **Correlation is critical** — a trace_id that links all three pillars lets you jump from a dashboard spike to the root cause in seconds.

3. **Metrics are your first line of defense** — they're cheap to store, fast to query, and perfect for alerting.

4. **Logs are for context** — when something goes wrong, logs tell you exactly what happened and why.

5. **Traces are essential for microservices** — when a request crosses 5+ services, you NEED traces to understand the journey.

6. **OpenTelemetry is the industry standard** — instrument once, export to any backend (Jaeger, Prometheus, Datadog, etc.).

7. **Be deliberate about what you collect** — more data isn't better if you can't find what matters. Focus on actionable signals.

---

## What's Next?

Now that you understand the three pillars conceptually, let's dive deep into the first pillar — **Centralized Logging** — and learn how to collect, store, and query logs across hundreds of services using the ELK Stack, Fluentd, and Loki.

**Next: [03-centralized-logging.md](./03-centralized-logging.md)** — Centralized Logging (ELK Stack, Fluentd, Loki)
