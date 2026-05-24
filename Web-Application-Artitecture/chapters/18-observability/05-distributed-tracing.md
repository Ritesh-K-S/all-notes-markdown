# Distributed Tracing — Jaeger, Zipkin & OpenTelemetry

> **What you'll learn**: How to follow a single request as it travels across multiple microservices, find performance bottlenecks, and understand complex service dependencies using distributed tracing.

---

## Real-Life Analogy

Imagine you order a package online and get a **tracking number**. With that number, you can see:
- Package picked up at warehouse (2 min)
- Transported to sorting facility (3 hours)
- Sorted and dispatched (45 min)
- On delivery truck (2 hours)
- **STUCK** at customs for 4 days ← bottleneck!
- Delivered to your door

Without that tracking number, you'd just know: "I ordered 5 days ago and it hasn't arrived." With it, you can pinpoint EXACTLY where the delay happened.

**Distributed tracing gives every request a tracking number.** As the request moves through Service A → Service B → Database → Cache → Service C, you can see exactly how long each hop took and where things got slow.

---

## The Problem: Lost in Microservices

In a monolith, finding slow code is easy — just profile one process. In microservices:

```
WITHOUT TRACING:

User: "The checkout page is slow!"

Developer: "Let me check..."
  → API Gateway: looks fine (10ms) ✓
  → Order Service: looks fine (15ms) ✓
  → Inventory Service: looks fine (20ms) ✓
  → Payment Service: looks fine (30ms) ✓
  → Notification Service: looks fine (5ms) ✓

Developer: "Everything looks fine individually... but the user waited 8 seconds?! 🤔"

THE MYSTERY: Where did those 8 seconds go?
  → Network latency between services? Retries? Queue delays?
  → We can't see the FULL picture of one request!
```

```
WITH TRACING:

Trace ID: abc-123-def

┌──────────────────────────────────────────────────────────────────┐
│  API Gateway          ████ (12ms)                                │
│  └─ Order Service     ██████████████████████████████ (145ms)     │
│     ├─ Inventory DB   ██████████ (45ms)                          │
│     ├─ Payment Svc    ████████████████████████████████████ (8.2s)│ ← HERE!
│     │  └─ Stripe API  ████████████████████████████████ (8.1s)    │ ← BOTTLENECK
│     └─ Notification   ██ (5ms)                                   │
└──────────────────────────────────────────────────────────────────┘

Total: 8.4 seconds
Root Cause: Stripe API timeout/slow response

FOUND IT in 5 seconds! 🎯
```

---

## Core Concepts

### Traces, Spans, and Context

```
┌─────────────────────────────────────────────────────────────────┐
│                    TRACING TERMINOLOGY                            │
│                                                                   │
│  TRACE:                                                          │
│  The ENTIRE journey of one request, from start to finish.        │
│  Has a unique Trace ID (e.g., "abc-123-def-456")                │
│                                                                   │
│  SPAN:                                                           │
│  ONE operation within a trace. Each service creates a span.      │
│  Has: name, start time, duration, parent span, tags             │
│                                                                   │
│  CONTEXT PROPAGATION:                                            │
│  Passing the Trace ID from service to service                    │
│  (via HTTP headers, message metadata, etc.)                      │
│                                                                   │
│  Example:                                                        │
│  ┌─── Trace: abc-123 ──────────────────────────────────┐        │
│  │                                                       │        │
│  │  Span A: API Gateway (parent=none)                   │        │
│  │    └── Span B: Order Service (parent=A)              │        │
│  │          ├── Span C: DB Query (parent=B)             │        │
│  │          ├── Span D: Payment Service (parent=B)      │        │
│  │          │     └── Span E: Stripe API (parent=D)     │        │
│  │          └── Span F: Send Email (parent=B)           │        │
│  │                                                       │        │
│  └───────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

### How Context Propagation Works

```
Service A sends request to Service B:

HTTP Request:
  GET /api/inventory/check HTTP/1.1
  Host: inventory-service:8080
  
  ┌─── TRACE HEADERS (W3C Trace Context standard) ───┐
  │ traceparent: 00-abc123def456-span789-01          │
  │              ──  ────────────  ──────  ──        │
  │              │   Trace ID      Span ID  Sampled  │
  │              version                              │
  └──────────────────────────────────────────────────┘

When Service B receives this:
  1. Extracts trace_id and parent_span_id from headers
  2. Creates a NEW span with parent = span789
  3. Does its work
  4. Passes headers to any downstream calls
  5. Reports its span to the tracing backend
```

---

## The Distributed Tracing Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                DISTRIBUTED TRACING SYSTEM                         │
│                                                                   │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                   │
│  │Service A │   │Service B │   │Service C │                   │
│  │          │   │          │   │          │                   │
│  │[OTel SDK]│   │[OTel SDK]│   │[OTel SDK]│                   │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘                   │
│       │               │               │                          │
│       │  Span data    │  Span data    │  Span data              │
│       ▼               ▼               ▼                          │
│  ┌────────────────────────────────────────────┐                 │
│  │         OpenTelemetry Collector             │                 │
│  │   (receives, batches, exports spans)       │                 │
│  └─────────────────────┬──────────────────────┘                 │
│                         │                                        │
│                         ▼                                        │
│  ┌────────────────────────────────────────────┐                 │
│  │         Tracing Backend                     │                 │
│  │   (Jaeger / Zipkin / Tempo / X-Ray)        │                 │
│  │                                             │                 │
│  │   • Stores spans                           │                 │
│  │   • Assembles traces (links spans)         │                 │
│  │   • Provides query API                     │                 │
│  │   • Visualizes trace waterfall             │                 │
│  └────────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Popular Tracing Tools

| Tool | Created By | Best For | Storage Backend |
|------|-----------|----------|-----------------|
| **Jaeger** | Uber (2017) | Kubernetes, large-scale microservices | Elasticsearch, Cassandra, Kafka |
| **Zipkin** | Twitter (2012) | Simple setup, wide language support | Elasticsearch, MySQL, Cassandra |
| **Grafana Tempo** | Grafana Labs | Cost-effective, integrates with Grafana | Object storage (S3/GCS) |
| **AWS X-Ray** | Amazon | AWS-native applications | AWS managed |
| **Google Cloud Trace** | Google | GCP-native applications | GCP managed |
| **Datadog APM** | Datadog | Full-stack observability (SaaS) | Datadog managed |

---

## How It Works Internally

### Span Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPAN LIFECYCLE                                  │
│                                                                   │
│  1. REQUEST ARRIVES                                              │
│     → Extract trace context from headers                        │
│     → OR generate new Trace ID (if first service)              │
│                                                                   │
│  2. SPAN CREATED                                                 │
│     → Record: span_id, trace_id, parent_span_id, start_time   │
│     → Set operation name (e.g., "HTTP GET /api/orders")        │
│                                                                   │
│  3. WORK HAPPENS                                                 │
│     → Add events/logs: ("Querying database", timestamp)        │
│     → Add attributes: (user_id="42", db.statement="SELECT...") │
│     → Create child spans for sub-operations                    │
│                                                                   │
│  4. SPAN ENDS                                                    │
│     → Record: end_time, status (OK/ERROR)                      │
│     → Calculate duration                                        │
│                                                                   │
│  5. SPAN EXPORTED                                                │
│     → Batched with other spans                                  │
│     → Sent to collector (async, non-blocking)                   │
│     → Collector forwards to storage backend                     │
└─────────────────────────────────────────────────────────────────┘
```

### Sampling Strategies

At high traffic, tracing EVERY request is too expensive. You sample:

```
┌─────────────────────────────────────────────────────────────────┐
│                   SAMPLING STRATEGIES                             │
│                                                                   │
│  1. HEAD-BASED SAMPLING (decision at start)                     │
│     ─────────────────────────────────────                       │
│     • Decide at the FIRST service whether to trace             │
│     • "Trace 10% of all requests"                               │
│     • Simple but may miss interesting traces                    │
│                                                                   │
│     Request arrives → Random(0,1) < 0.1? → Trace it!          │
│                                                                   │
│  2. TAIL-BASED SAMPLING (decision at end)                       │
│     ─────────────────────────────────────                       │
│     • Collect ALL spans, then decide what to KEEP              │
│     • "Keep all traces with errors or latency > 2s"           │
│     • More useful but more complex/expensive                    │
│                                                                   │
│     All spans → Collector → [Error?] → Keep!                   │
│                             [Slow?]  → Keep!                    │
│                             [Normal?] → Sample 1%              │
│                                                                   │
│  3. ADAPTIVE SAMPLING                                            │
│     ─────────────────────────────────────                       │
│     • Adjust rate based on traffic                              │
│     • Low traffic → 100% sampling                              │
│     • High traffic → 0.1% sampling                             │
│     • Guarantees minimum traces per endpoint                    │
└─────────────────────────────────────────────────────────────────┘
```

### How Jaeger Stores and Queries Traces

```
┌─────────────────────────────────────────────────────────────────┐
│                     JAEGER ARCHITECTURE                           │
│                                                                   │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────────┐     │
│  │  Services│───▶│ Jaeger Agent │───▶│ Jaeger Collector  │     │
│  │ (OTel SDK│    │ (UDP recv,   │    │ (validates,       │     │
│  │  or Jaeger│    │  batches)    │    │  indexes, stores)│     │
│  │  client) │    └──────────────┘    └────────┬──────────┘     │
│  └──────────┘                                  │                 │
│                                                │                 │
│                                    ┌───────────┼──────────┐     │
│                                    ▼           ▼          ▼     │
│                              Elasticsearch  Cassandra   Kafka   │
│                              (search by     (high       (buffer │
│                               service,       write      before  │
│                               operation,     volume)    store)  │
│                               tags)                             │
│                                                                   │
│  ┌───────────────────────────────────────────────────────┐      │
│  │               Jaeger Query UI                          │      │
│  │                                                         │      │
│  │  Search: service=order-service, min_duration=2s        │      │
│  │  Result: Trace abc-123 (8.4s total)                   │      │
│  │                                                         │      │
│  │  [API Gateway]  ████ (12ms)                           │      │
│  │  [Order Svc]    ██████████████ (145ms)                │      │
│  │  [Payment Svc]  ██████████████████████████ (8.2s)     │      │
│  │  [Stripe API]   ████████████████████████ (8.1s) ⚠️    │      │
│  └───────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: Distributed Tracing with OpenTelemetry

```python
# order_service.py
# Service that traces requests across multiple downstream calls

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.propagate import inject
from flask import Flask
import requests

# --- Setup OpenTelemetry ---
provider = TracerProvider()
# Export spans to OTel Collector (which sends to Jaeger/Tempo)
processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317"))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("order-service")

app = Flask(__name__)
# Auto-instrument Flask (creates spans for every HTTP request)
FlaskInstrumentor().instrument_app(app)
# Auto-instrument outgoing HTTP calls (propagates trace context)
RequestsInstrumentor().instrument()

@app.route('/api/orders', methods=['POST'])
def create_order():
    """Creates an order — calls inventory and payment services."""
    
    # This span is automatically created by FlaskInstrumentor
    # Let's add custom spans for business logic
    
    with tracer.start_as_current_span("validate_order") as span:
        span.set_attribute("order.items_count", 3)
        # Validation logic here...
    
    # Call Inventory Service (trace context propagated automatically!)
    with tracer.start_as_current_span("check_inventory"):
        resp = requests.get("http://inventory-service:8080/api/stock/check",
                           params={"items": "laptop,mouse,keyboard"})
        if resp.status_code != 200:
            span = trace.get_current_span()
            span.set_status(trace.Status(trace.StatusCode.ERROR))
            span.record_exception(Exception("Inventory check failed"))
            return {"error": "Out of stock"}, 409
    
    # Call Payment Service (trace context propagated automatically!)
    with tracer.start_as_current_span("process_payment") as span:
        span.set_attribute("payment.amount", 999.99)
        span.set_attribute("payment.currency", "USD")
        resp = requests.post("http://payment-service:8080/api/charge",
                            json={"amount": 999.99, "user_id": "user_42"})
    
    # Add event (log within the trace)
    current_span = trace.get_current_span()
    current_span.add_event("order_created", {"order_id": "ORD-12345"})
    
    return {"order_id": "ORD-12345", "status": "confirmed"}, 201

if __name__ == '__main__':
    app.run(port=8080)
```

```python
# payment_service.py
# Downstream service — receives and continues the trace

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from flask import Flask, request
import requests
import time

provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317"))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("payment-service")

app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

@app.route('/api/charge', methods=['POST'])
def charge():
    """Processes payment — the trace continues from order-service."""
    # The trace context was automatically extracted from incoming headers!
    # This span is a CHILD of the order-service span
    
    with tracer.start_as_current_span("call_stripe_api") as span:
        span.set_attribute("stripe.amount", request.json['amount'])
        # Call external payment provider
        resp = requests.post("https://api.stripe.com/v1/charges",
                            json={"amount": request.json['amount']})
        span.set_attribute("stripe.charge_id", "ch_1234")
    
    return {"charge_id": "ch_1234", "status": "succeeded"}

if __name__ == '__main__':
    app.run(port=8081)
```

### Java: Distributed Tracing with OpenTelemetry

```java
// OrderService.java - Spring Boot with OpenTelemetry auto-instrumentation

import io.opentelemetry.api.trace.*;
import io.opentelemetry.api.common.*;
import io.opentelemetry.context.Scope;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;

@RestController
public class OrderController {
    
    private final Tracer tracer;
    private final RestTemplate restTemplate; // Auto-instrumented by OTel agent
    
    public OrderController(Tracer tracer, RestTemplate restTemplate) {
        this.tracer = tracer;
        this.restTemplate = restTemplate;
    }
    
    @PostMapping("/api/orders")
    public Map<String, String> createOrder(@RequestBody OrderRequest request) {
        // Parent span automatically created by OTel auto-instrumentation
        
        // Custom child span for business logic
        Span inventorySpan = tracer.spanBuilder("check_inventory")
            .setAttribute("items.count", request.getItems().size())
            .startSpan();
        
        try (Scope scope = inventorySpan.makeCurrent()) {
            // Trace context automatically injected into outgoing HTTP headers!
            var stockResponse = restTemplate.getForObject(
                "http://inventory-service:8080/api/stock/check?items={items}",
                StockResponse.class,
                String.join(",", request.getItems())
            );
            inventorySpan.setAttribute("inventory.available", stockResponse.isAvailable());
        } finally {
            inventorySpan.end();
        }
        
        // Custom span for payment
        Span paymentSpan = tracer.spanBuilder("process_payment")
            .setAttribute("payment.amount", request.getAmount())
            .setAttribute("payment.currency", "USD")
            .startSpan();
        
        try (Scope scope = paymentSpan.makeCurrent()) {
            var paymentResponse = restTemplate.postForObject(
                "http://payment-service:8080/api/charge",
                new ChargeRequest(request.getAmount(), request.getUserId()),
                PaymentResponse.class
            );
            paymentSpan.addEvent("payment_completed", Attributes.of(
                AttributeKey.stringKey("charge_id"), paymentResponse.getChargeId()
            ));
        } catch (Exception e) {
            paymentSpan.setStatus(StatusCode.ERROR, e.getMessage());
            paymentSpan.recordException(e);
            throw e;
        } finally {
            paymentSpan.end();
        }
        
        return Map.of("order_id", "ORD-12345", "status", "confirmed");
    }
}
```

---

## Infrastructure Example: Jaeger with OpenTelemetry Collector

```yaml
# docker-compose-tracing.yml
version: '3.8'

services:
  # OpenTelemetry Collector - receives spans from all services
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.96.0
    ports:
      - "4317:4317"   # OTLP gRPC receiver
      - "4318:4318"   # OTLP HTTP receiver
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol/config.yaml

  # Jaeger - stores and visualizes traces
  jaeger:
    image: jaegertracing/all-in-one:1.54
    ports:
      - "16686:16686"  # Jaeger UI
      - "14250:14250"  # gRPC from collector
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200

  # Elasticsearch - trace storage
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
```

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1000
  
  # Tail-based sampling: keep errors and slow traces
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-traces
        type: latency
        latency: {threshold_ms: 2000}
      - name: probabilistic
        type: probabilistic
        probabilistic: {sampling_percentage: 10}

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  
  # Also export to Prometheus for trace-based metrics
  prometheus:
    endpoint: "0.0.0.0:8889"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling, batch]
      exporters: [jaeger]
```

---

## Real-World Example

### How Uber Built Jaeger for 4,000+ Microservices

```
┌─────────────────────────────────────────────────────────┐
│              UBER'S TRACING AT SCALE                      │
│                                                          │
│  Scale:                                                 │
│  • 4,000+ microservices                                 │
│  • Millions of spans per second                        │
│  • Average trace spans 10+ services                    │
│                                                          │
│  Architecture:                                          │
│  [Services] → [Jaeger Agent] → [Kafka] → [Flink] →   │
│            → [Cassandra] → [Jaeger Query UI]           │
│                                                          │
│  Key Design Decisions:                                  │
│  • Agent runs as sidecar (one per host)                │
│  • Kafka buffers spans (handles bursts)                │
│  • Flink processes & enriches spans in real-time       │
│  • Cassandra stores (high write throughput)            │
│  • Adaptive sampling: 0.1% normal, 100% errors        │
│                                                          │
│  Impact:                                                │
│  • MTTR (Mean Time To Resolve) dropped 50%            │
│  • Engineers find bottlenecks in < 5 minutes           │
│  • Dependency graph auto-generated from traces         │
└─────────────────────────────────────────────────────────┘
```

### How Google Uses Tracing (Dapper)

Google's internal system **Dapper** (2010 paper) traced:
- Every single request across all Google services
- Sampling rate: 1 in 1024 (0.1%) for low-overhead
- Still catches all slow/error traces
- Used to build automatic service dependency maps
- Inspired Zipkin, Jaeger, and the entire tracing ecosystem

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| **No context propagation** | Traces break at service boundaries | Use OTel auto-instrumentation for HTTP clients |
| **Tracing 100% of traffic** | Massive storage costs, performance overhead | Use sampling: 1-10% normal, 100% errors |
| **Not tracing async operations** | Kafka consumers lose trace context | Pass trace context in message headers |
| **Too many spans** | Traces become unreadable noise | One span per meaningful operation, not per function |
| **Not adding attributes** | Spans say "HTTP GET" but not what happened | Add user_id, order_id, relevant business context |
| **Forgetting external calls** | Can't see time spent in 3rd party APIs | Instrument HTTP clients (auto-instrumentation helps) |
| **No tail-based sampling** | Miss rare but important error traces | Use OTel Collector with tail_sampling processor |

---

## When to Use / When NOT to Use

### Use Distributed Tracing When:
- ✅ You have 3+ services that call each other
- ✅ Users complain about latency and you can't find why
- ✅ You need to understand service dependencies
- ✅ You're debugging intermittent failures across services
- ✅ You want to find which service is the bottleneck

### Distributed Tracing Is Overkill When:
- ❌ You have a single monolith (use a profiler instead)
- ❌ You have 2 services that rarely interact
- ❌ Your system has < 100 requests per minute
- ❌ You're just starting a project (add it when you hit microservices)

### Tool Selection Guide:

```
Need free + self-hosted?          → Jaeger or Zipkin
Already using Grafana?            → Grafana Tempo
On AWS?                           → AWS X-Ray
Want zero setup (SaaS)?           → Datadog APM or Honeycomb
Need to keep costs minimal?       → Grafana Tempo (S3 backend)
```

---

## Key Takeaways

1. **Distributed tracing follows ONE request** across all services, showing exactly where time is spent — it's like a GPS tracker for requests.

2. **Context propagation is critical** — the trace ID must pass through every HTTP call, queue message, and async operation, or the trace breaks.

3. **OpenTelemetry is the standard** — instrument once with OTel, export to any backend (Jaeger, Zipkin, Tempo, Datadog).

4. **Sampling is mandatory at scale** — you can't store every trace at high traffic. Use tail-based sampling to always keep errors and slow traces.

5. **Auto-instrumentation covers 80%** — most frameworks have OTel libraries that automatically trace HTTP, DB, and cache calls.

6. **Combine with logs and metrics** — traces show WHERE the problem is, metrics show WHEN it started, logs show WHY it happened.

7. **Start simple** — even basic tracing across 3-4 services will save you hours of debugging time per week.

---

## What's Next?

Now that you can see inside your system with logs, metrics, and traces, the next step is knowing when something goes wrong — **Alerting & On-Call**. We'll cover how to set up intelligent alerts that page the right person at the right time.

**Next: [06-alerting-oncall.md](./06-alerting-oncall.md)** — Alerting & On-Call (PagerDuty, OpsGenie)
