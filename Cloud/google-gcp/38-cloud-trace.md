# Chapter 38 — Cloud Trace

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Fundamentals](#part-1--fundamentals)
- [Part 2: Architecture & Core Concepts](#part-2--architecture--core-concepts)
- [Part 3: Trace List & Trace Detail](#part-3--trace-list--trace-detail)
- [Part 4: Latency Analysis](#part-4--latency-analysis)
- [Part 5: Auto-Instrumentation](#part-5--auto-instrumentation)
- [Part 6: Manual Instrumentation with OpenTelemetry](#part-6--manual-instrumentation-with-opentelemetry)
- [Part 7: Cloud Run & GKE Integration](#part-7--cloud-run--gke-integration)
- [Part 8: Trace Context Propagation](#part-8--trace-context-propagation)
- [Part 9: Sampling Strategies](#part-9--sampling-strategies)
- [Part 10: Correlating Traces with Logs & Metrics](#part-10--correlating-traces-with-logs--metrics)
- [Part 11: Trace API & Exporting](#part-11--trace-api--exporting)
- [Part 12: IAM & Security](#part-12--iam--security)
- [Part 13: Troubleshooting & Best Practices](#part-13--troubleshooting--best-practices)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud Trace is a fully managed distributed tracing system that collects latency data from applications, helping you understand how requests propagate through microservices. It provides near-real-time latency analysis, automatic performance bottleneck detection, and tight integration with Cloud Logging and Cloud Monitoring. Cloud Trace supports OpenTelemetry as the primary instrumentation framework and auto-instruments many GCP services.

---

## Part 1 — Fundamentals

### What Is Cloud Trace?

Cloud Trace captures and displays request latency data for distributed applications. When a user request enters your system and passes through multiple services, Cloud Trace records the timing of each step as **spans** and groups them into a **trace**.

```
┌─────────────────────────────────────────────────────────────────┐
│                 CLOUD TRACE OVERVIEW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  User Request                                                    │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Trace (one end-to-end request)                         │    │
│  │                                                         │    │
│  │  ├── Span: API Gateway (0–280ms) ──────────────────    │    │
│  │  │   ├── Span: Auth Service (10–40ms) ─────           │    │
│  │  │   ├── Span: Order Service (50–250ms) ──────────    │    │
│  │  │   │   ├── Span: DB Query (60–90ms) ───            │    │
│  │  │   │   ├── Span: Payment API (100–220ms) ──────    │    │
│  │  │   │   │   └── Span: Fraud Check (120–180ms) ──   │    │
│  │  │   │   └── Span: Send Email (230–245ms) ──        │    │
│  │  │   └── Span: Response (260–275ms) ──               │    │
│  │  │                                                     │    │
│  │  Timeline: 0ms ──────────────────────────── 280ms     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Key Benefits:                                                    │
│  • Identify latency bottlenecks across services                  │
│  • Understand service dependencies                                │
│  • Detect performance regressions                                │
│  • Correlate with logs and metrics                               │
│  • Root-cause analysis for slow requests                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Capabilities

| Capability | Description |
|------------|-------------|
| **Trace list** | View all collected traces with latency data |
| **Trace detail** | Waterfall view of spans within a trace |
| **Latency analysis** | Distribution charts, percentile breakdown |
| **Auto-instrumentation** | Automatic tracing for GCP services |
| **OpenTelemetry support** | Native OTel SDK + Collector integration |
| **Log correlation** | Link trace spans to Cloud Logging entries |
| **Metric correlation** | Link traces to Cloud Monitoring metrics |
| **Sampling** | Configurable sampling rates for cost/volume control |
| **Cross-project** | Traces propagate across project boundaries |

### Cross-Cloud Comparison

| Feature | GCP Cloud Trace | AWS X-Ray | Azure Application Insights |
|---------|----------------|-----------|---------------------------|
| Service | Cloud Trace | X-Ray | App Insights (distributed tracing) |
| Protocol | OpenTelemetry, Cloud Trace API | X-Ray SDK, OTel | OpenTelemetry, AI SDK |
| Auto-instrumentation | Cloud Run, GKE, App Engine, Cloud Functions | Lambda, API Gateway, ECS | App Service, Functions, AKS |
| Trace propagation | W3C Trace Context, X-Cloud-Trace-Context | X-Amzn-Trace-Id | W3C Trace Context |
| Sampling | Configurable (rate, head-based) | Fixed rate / reservoir | Adaptive sampling |
| Log correlation | Trace ID in Cloud Logging | Trace ID in CloudWatch | Operation ID in logs |
| Dependency map | Automatic (via traces) | Service map | Application map |
| Latency analysis | Built-in percentile charts | Latency histograms | Performance blade |
| Pricing | Free (first 2.5M spans/month) | Free tier + $5/M traces | Included in AI pricing |
| Open standard | OpenTelemetry native | OTel supported | OTel supported |
| Retention | 30 days | 30 days | 90 days (configurable) |

### Pricing

| Component | Cost |
|-----------|------|
| Trace ingestion — first 2.5M spans/month | Free |
| Trace ingestion — beyond 2.5M spans | ~$0.20 per million spans |
| Trace storage (30-day retention) | Included |
| Trace API reads | Free |

> **Note**: Cloud Trace is one of the most affordable distributed tracing solutions available. The free tier covers many small-to-medium applications entirely.

---

## Part 2 — Architecture & Core Concepts

### Distributed Tracing Concepts

```
┌──────────────────────────────────────────────────────────────┐
│              TRACING CONCEPTS                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Trace                                                        │
│  ├── A complete request journey through your system           │
│  ├── Identified by a unique Trace ID (128-bit)               │
│  └── Contains one or more spans                               │
│                                                                │
│  Span                                                         │
│  ├── A single unit of work within a trace                     │
│  ├── Has: span ID, parent span ID, name, start/end time     │
│  ├── Can have: attributes, events, links, status             │
│  └── Types:                                                   │
│      ├── Root span: first span in a trace (no parent)        │
│      ├── Child span: has a parent span                        │
│      └── Server/Client/Internal span kinds                   │
│                                                                │
│  Span Attributes                                               │
│  ├── Key-value metadata on a span                             │
│  ├── Standard: http.method, http.status_code, db.system      │
│  └── Custom: order.id, user.tier                             │
│                                                                │
│  Span Events                                                   │
│  ├── Timestamped annotations within a span                    │
│  └── Example: "cache miss at 45ms", "retry attempt 2"       │
│                                                                │
│  Span Status                                                   │
│  ├── OK: successful                                           │
│  ├── ERROR: failed                                            │
│  └── UNSET: not explicitly set                                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### How Cloud Trace Works

```
┌──────────────────────────────────────────────────────────────┐
│             CLOUD TRACE ARCHITECTURE                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Service A    │  │ Service B    │  │ Service C    │       │
│  │ (Cloud Run)  │  │ (GKE)       │  │ (Compute)   │       │
│  │              │  │              │  │              │       │
│  │ OTel SDK     │  │ OTel SDK     │  │ OTel SDK     │       │
│  │ or auto-     │  │ or auto-     │  │ + Ops Agent  │       │
│  │ instrumented │  │ instrumented │  │              │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                  │                  │               │
│         │  Trace context propagated via HTTP headers          │
│         │  (W3C traceparent / X-Cloud-Trace-Context)         │
│         │                  │                  │               │
│         └──────────────────┼──────────────────┘               │
│                            ▼                                  │
│  ┌────────────────────────────────────────────────────┐      │
│  │              Cloud Trace API                        │      │
│  │              (BatchWriteSpans)                      │      │
│  └──────────────────────┬─────────────────────────────┘      │
│                         │                                     │
│                         ▼                                     │
│  ┌────────────────────────────────────────────────────┐      │
│  │              Cloud Trace Backend                    │      │
│  │                                                    │      │
│  │  ┌────────────┐  ┌─────────────┐  ┌────────────┐ │      │
│  │  │ Trace List │  │  Analysis   │  │ Latency    │ │      │
│  │  │ & Detail   │  │  Reports    │  │ Profiles   │ │      │
│  │  └────────────┘  └─────────────┘  └────────────┘ │      │
│  └────────────────────────────────────────────────────┘      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Auto-Instrumented GCP Services

| Service | What Is Traced | Auto? |
|---------|---------------|-------|
| **Cloud Run** | Incoming HTTP requests | Yes (requires OTel for outgoing) |
| **App Engine** | Incoming requests | Yes |
| **Cloud Functions** | Function invocations | Yes (Gen 2 via Cloud Run) |
| **GKE** | Via Istio/Anthos service mesh | Mesh auto-injects |
| **Cloud Load Balancer** | Request routing | Generates trace headers |
| **Pub/Sub** | Message publish/subscribe | Headers propagated |
| **Cloud Tasks** | Task dispatch | Headers propagated |
| **Cloud SQL** | DB calls (via OTel instrumentation) | Requires SDK |
| **Compute Engine** | Not auto-instrumented | Requires SDK + agent |

---

## Part 3 — Trace List & Trace Detail

### Trace List View

```
┌──────────────────────────────────────────────────────────────┐
│                  TRACE LIST                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Console → Trace → Trace list                                 │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Filters:                                             │    │
│  │  Time range: [Last 1 hour          ▼]                │    │
│  │  Root span:  [recv./checkout       ▼]                │    │
│  │  Min latency: [200ms  ]                              │    │
│  │  Max latency: [_______]                              │    │
│  │  HTTP method: [POST               ▼]                │    │
│  │  HTTP status: [500                ▼]                │    │
│  │  Filter label: [service.name = checkout-api]         │    │
│  │  [Apply]                                              │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Latency Distribution (scatter plot)                  │    │
│  │    ·    ·      ·                                      │    │
│  │  · · ·· ···  · · ·     · ·  ·                       │    │
│  │  ····················  ··········  ·    ·    ·        │    │
│  │  ──────────────────────────────────────────────       │    │
│  │  0ms      100ms      200ms      500ms    1s+         │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Trace ID           │ Root Span        │ Latency     │    │
│  │  ─────────────────────────────────────────────────── │    │
│  │  abc123def456...    │ POST /checkout   │ 523ms       │    │
│  │  789ghi012jkl...    │ POST /checkout   │ 1,245ms ⚠  │    │
│  │  mno345pqr678...    │ GET  /products   │ 45ms        │    │
│  │  stu901vwx234...    │ POST /payment    │ 890ms       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Trace Detail (Waterfall) View

```
┌──────────────────────────────────────────────────────────────┐
│             TRACE DETAIL — WATERFALL VIEW                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Trace ID: abc123def456ghi789...                              │
│  Total duration: 523ms                                        │
│  Spans: 7                                                     │
│                                                                │
│  Timeline: 0ms         100ms        200ms        400ms  523ms│
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  API Gateway ████████████████████████████████████████   │ │
│  │  (523ms)                                                │ │
│  │                                                         │ │
│  │    Auth     ███                                         │ │
│  │    (32ms)                                               │ │
│  │                                                         │ │
│  │    Order Service     ████████████████████████████       │ │
│  │    (430ms)                                              │ │
│  │                                                         │ │
│  │      DB Read            ████                            │ │
│  │      (45ms)                                             │ │
│  │                                                         │ │
│  │      Payment API              ██████████████████  ⚠    │ │
│  │      (280ms)                     ← BOTTLENECK           │ │
│  │                                                         │ │
│  │        Fraud Check                 ████████             │ │
│  │        (120ms)                                          │ │
│  │                                                         │ │
│  │      Send Notification                          ███     │ │
│  │      (35ms)                                             │ │
│  │                                                         │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                                │
│  Selected span: Payment API                                   │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Attributes:                                             │ │
│  │    http.method = POST                                   │ │
│  │    http.url = https://payment.internal/charge           │ │
│  │    http.status_code = 200                                │ │
│  │    payment.amount = 99.50                                │ │
│  │    payment.currency = USD                                │ │
│  │  Events:                                                 │ │
│  │    [+120ms] "Fraud check started"                       │ │
│  │    [+240ms] "Fraud check passed"                        │ │
│  │  Related logs: [View in Log Explorer]                   │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Filtering Traces

| Filter | Description | Example |
|--------|-------------|---------|
| Root span | Filter by the root span name | `recv./api/checkout` |
| Latency | Min/max latency range | 200ms – 5000ms |
| HTTP method | GET, POST, PUT, DELETE | POST |
| HTTP status | Response status code | 500 |
| Span name | Any span name in the trace | `pg.query` |
| Label filter | Custom span attributes | `service.name = "checkout"` |
| Time range | Start/end time window | Last 1 hour |

---

## Part 4 — Latency Analysis

### Analysis Reports

```
┌──────────────────────────────────────────────────────────────┐
│            LATENCY ANALYSIS                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Console → Trace → Analysis reports                           │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Request:  POST /api/checkout                         │    │
│  │  Period:   Last 24 hours                              │    │
│  │                                                      │    │
│  │  Latency Distribution:                                │    │
│  │  ┌───────────────────────────────────────────────┐   │    │
│  │  │                                               │   │    │
│  │  │    ▂▅█▇▅▃▂▁            ▂▃▂                   │   │    │
│  │  │  ──────────────────────────────────────────── │   │    │
│  │  │  50ms  100ms  200ms  500ms  1s    2s    5s   │   │    │
│  │  └───────────────────────────────────────────────┘   │    │
│  │                                                      │    │
│  │  Percentiles:                                         │    │
│  │  ┌──────────┬──────────┬──────────┬──────────┐      │    │
│  │  │  p50     │  p75     │  p95     │  p99     │      │    │
│  │  │  120ms   │  180ms   │  450ms   │  1,200ms │      │    │
│  │  └──────────┴──────────┴──────────┴──────────┘      │    │
│  │                                                      │    │
│  │  Comparison: Today vs Yesterday                      │    │
│  │  p50: 120ms → 115ms  (▼ 4% faster)                 │    │
│  │  p99: 1,200ms → 890ms (▼ 26% faster)               │    │
│  │                                                      │    │
│  │  Most common bottleneck spans:                       │    │
│  │  1. payment-service/charge  (avg 280ms, 45% of trace)│   │
│  │  2. db/query                (avg 60ms, 10% of trace) │    │
│  │  3. auth/validate           (avg 35ms, 6% of trace)  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Latency Percentile Breakdown

| Percentile | Meaning | Typical Target |
|-----------|---------|----------------|
| **p50** (median) | Half of requests are faster | < 100ms for APIs |
| **p75** | 75% of requests are faster | < 200ms |
| **p90** | 90% of requests are faster | < 500ms |
| **p95** | 95% of requests are faster | < 1s |
| **p99** | 99% of requests are faster | < 2s |
| **p99.9** | 999 out of 1,000 are faster | < 5s |

### Using Analysis for Performance Investigation

```
Investigation workflow:
1. Check p99 latency — is it within SLO?
2. If elevated → filter trace list by high latency (e.g., > p95)
3. Open a slow trace → inspect waterfall
4. Identify bottleneck span (longest / most time %)
5. Examine span attributes for clues
6. Correlate with logs (click "View logs")
7. Check if bottleneck is consistent across traces
8. Fix root cause (slow query, external API, etc.)
```

---

## Part 5 — Auto-Instrumentation

### Cloud Run Auto-Instrumentation

Cloud Run automatically generates trace spans for incoming requests. To get full end-to-end tracing, you need to:

1. Propagate trace context in outgoing requests
2. Add instrumentation for outgoing calls (HTTP, DB, etc.)

```python
# Cloud Run reads the X-Cloud-Trace-Context header automatically
# and creates a root span for each incoming request.

# To propagate context to downstream services:
import requests

def handle_request(request):
    # Get trace header from incoming request
    trace_header = request.headers.get('X-Cloud-Trace-Context', '')
    
    # Pass it to downstream calls
    response = requests.get(
        'https://payment-service.run.app/charge',
        headers={'X-Cloud-Trace-Context': trace_header}
    )
    return response.json()
```

### App Engine Auto-Instrumentation

```
App Engine Standard and Flexible automatically:
• Create spans for incoming requests
• Propagate trace context
• Record request latency

No configuration needed — traces appear in Cloud Trace
immediately after deploying your app.
```

### Cloud Functions Auto-Instrumentation

```
Cloud Functions (Gen 2, based on Cloud Run):
• Incoming invocations are automatically traced
• Event-driven functions include trigger context
• HTTP functions work like Cloud Run

Cloud Functions (Gen 1):
• Basic tracing via Trace API
• Manual instrumentation recommended
```

### GKE with Istio/Anthos Service Mesh

```yaml
# When Istio sidecar is injected, all inter-service traffic
# is automatically traced via Envoy proxy.

# Enable tracing in Istio mesh config:
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 100.0  # 100% sampling (adjust for production)
        zipkin:
          address: cloudtrace.googleapis.com:443
```

---

## Part 6 — Manual Instrumentation with OpenTelemetry

### Python — OpenTelemetry Setup

```python
# pip install opentelemetry-api opentelemetry-sdk \
#   opentelemetry-exporter-gcp-trace \
#   opentelemetry-instrumentation-requests \
#   opentelemetry-instrumentation-flask \
#   opentelemetry-instrumentation-sqlalchemy

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.resources import Resource

# Configure the tracer
resource = Resource.create({
    "service.name": "checkout-api",
    "service.version": "2.1.0",
    "deployment.environment": "production",
})

provider = TracerProvider(resource=resource)
processor = BatchSpanProcessor(CloudTraceSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)
```

### Python — Creating Spans

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

# Basic span
def process_order(order_id):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        
        # Child span for DB query
        with tracer.start_as_current_span("db.query") as db_span:
            db_span.set_attribute("db.system", "postgresql")
            db_span.set_attribute("db.statement", "SELECT * FROM orders WHERE id = ?")
            order = fetch_order(order_id)
        
        # Child span for payment
        with tracer.start_as_current_span("payment.charge") as pay_span:
            pay_span.set_attribute("payment.amount", order.total)
            pay_span.set_attribute("payment.currency", "USD")
            try:
                result = charge_payment(order)
                pay_span.set_attribute("payment.status", "success")
            except PaymentError as e:
                pay_span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
                pay_span.record_exception(e)
                raise
        
        # Add span event
        span.add_event("Order processed", {
            "order.id": order_id,
            "order.total": str(order.total),
        })
        
        return order
```

### Python — Auto-Instrumentation Libraries

```python
# Auto-instrument common libraries (call once at startup)
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.grpc import GrpcInstrumentorClient

# Flask (incoming HTTP)
FlaskInstrumentor().instrument_app(app)

# requests library (outgoing HTTP)
RequestsInstrumentor().instrument()

# SQLAlchemy (database)
SQLAlchemyInstrumentor().instrument(engine=engine)

# Redis
RedisInstrumentor().instrument()

# gRPC client
GrpcInstrumentorClient().instrument()
```

### Node.js — OpenTelemetry Setup

```javascript
// npm install @opentelemetry/api @opentelemetry/sdk-trace-node
//   @opentelemetry/exporter-trace-otlp-grpc
//   @google-cloud/opentelemetry-cloud-trace-exporter
//   @opentelemetry/instrumentation-http
//   @opentelemetry/instrumentation-express

const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { TraceExporter } = require('@google-cloud/opentelemetry-cloud-trace-exporter');
const { Resource } = require('@opentelemetry/resources');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');

const provider = new NodeTracerProvider({
  resource: new Resource({
    'service.name': 'checkout-api',
    'service.version': '2.1.0',
  }),
});

provider.addSpanProcessor(
  new BatchSpanProcessor(new TraceExporter({ projectId: 'my-project' }))
);
provider.register();

registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ],
});
```

### Go — OpenTelemetry Setup

```go
package main

import (
    "context"

    texporter "github.com/GoogleCloudPlatform/opentelemetry-operations-go/exporter/trace"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

func initTracer() (*sdktrace.TracerProvider, error) {
    exporter, err := texporter.New(texporter.WithProjectID("my-project"))
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("checkout-api"),
            semconv.ServiceVersion("2.1.0"),
        )),
        sdktrace.WithSampler(sdktrace.TraceIDRatioBased(0.1)), // 10% sampling
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}

func processOrder(ctx context.Context, orderID string) {
    tracer := otel.Tracer("checkout")
    ctx, span := tracer.Start(ctx, "process_order")
    defer span.End()

    span.SetAttributes(attribute.String("order.id", orderID))

    // Child span
    ctx, dbSpan := tracer.Start(ctx, "db.query")
    order := fetchOrder(ctx, orderID)
    dbSpan.End()

    // ...
}
```

### Java — OpenTelemetry Setup

```java
// Using the OpenTelemetry Java Agent (zero-code instrumentation)
// java -javaagent:opentelemetry-javaagent.jar \
//   -Dotel.exporter.otlp.endpoint=https://cloudtrace.googleapis.com \
//   -Dotel.resource.attributes=service.name=checkout-api \
//   -Dotel.traces.exporter=google_cloud_trace \
//   -jar myapp.jar

// Or programmatic setup:
import com.google.cloud.opentelemetry.trace.TraceExporter;
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.Span;

TraceExporter exporter = TraceExporter.createWithDefaultConfiguration();
// Configure TracerProvider with exporter...

Tracer tracer = GlobalOpenTelemetry.getTracer("checkout");
Span span = tracer.spanBuilder("process_order")
    .setAttribute("order.id", orderId)
    .startSpan();
try {
    // business logic
} finally {
    span.end();
}
```

---

## Part 7 — Cloud Run & GKE Integration

### Cloud Run Tracing Setup

```python
# For Cloud Run, the simplest setup uses the OTel SDK
# with the Cloud Trace exporter.

# Cloud Run automatically:
# 1. Injects X-Cloud-Trace-Context header
# 2. Creates a root span for the request

# Your job:
# 1. Set up OTel with Cloud Trace exporter (see Part 6)
# 2. Use auto-instrumentation for outgoing calls
# 3. Propagate context via headers

# requirements.txt
# opentelemetry-api
# opentelemetry-sdk
# opentelemetry-exporter-gcp-trace
# opentelemetry-instrumentation-flask
# opentelemetry-instrumentation-requests
# opentelemetry-propagator-gcp

from opentelemetry.propagators.cloud_trace_propagator import CloudTraceFormatPropagator
from opentelemetry import propagate

# Register GCP trace context propagator
propagate.set_global_textmap(CloudTraceFormatPropagator())
```

### GKE Tracing with OpenTelemetry Collector

```yaml
# Deploy OpenTelemetry Collector as a DaemonSet in GKE
# This receives spans from apps and exports to Cloud Trace

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:latest
          ports:
            - containerPort: 4317  # OTLP gRPC
            - containerPort: 4318  # OTLP HTTP
          volumeMounts:
            - name: config
              mountPath: /etc/otel
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
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
        send_batch_size: 256
      resourcedetection:
        detectors: [gcp]

    exporters:
      googlecloud:
        project: my-project
        trace:
          attribute_mappings:
            - key: service.name
              replacement: g.co/r/service_name

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [resourcedetection, batch]
          exporters: [googlecloud]
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  selector:
    app: otel-collector
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
```

```yaml
# Application deployment — point OTel SDK at the collector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-api
spec:
  template:
    spec:
      containers:
        - name: checkout
          image: gcr.io/my-project/checkout-api:v2.1
          env:
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.monitoring:4317"
            - name: OTEL_SERVICE_NAME
              value: "checkout-api"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "service.version=2.1.0,deployment.environment=production"
```

---

## Part 8 — Trace Context Propagation

### Trace Context Headers

```
┌──────────────────────────────────────────────────────────────┐
│          TRACE CONTEXT PROPAGATION                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Two header formats supported:                                 │
│                                                                │
│  1. W3C Trace Context (standard, recommended):                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  traceparent: 00-TRACE_ID-SPAN_ID-FLAGS             │    │
│  │  Example:                                            │    │
│  │  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736   │    │
│  │               -00f067aa0ba902b7-01                   │    │
│  │                                                      │    │
│  │  tracestate: gcp=ADDITIONAL_DATA                    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  2. X-Cloud-Trace-Context (GCP-specific, legacy):             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  X-Cloud-Trace-Context: TRACE_ID/SPAN_ID;o=FLAGS    │    │
│  │  Example:                                            │    │
│  │  X-Cloud-Trace-Context: 4bf92f3577b34da6a3ce929d/   │    │
│  │    1234567890;o=1                                    │    │
│  │                                                      │    │
│  │  o=0 → trace not sampled                            │    │
│  │  o=1 → trace sampled (send to Cloud Trace)          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Propagation flow:                                             │
│  LB → Service A → Service B → Service C                      │
│    header     header     header                               │
│  (generated) (forwarded) (forwarded)                          │
│                                                                │
│  All spans share the same Trace ID                            │
│  Each span has its own Span ID                                │
│  Parent Span ID links child → parent                          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Propagation in Code

```python
# Python — propagating context with OpenTelemetry
from opentelemetry import context, propagate
import requests

def call_downstream(url):
    headers = {}
    # Inject current trace context into headers
    propagate.inject(headers)
    
    # Headers now contain traceparent / X-Cloud-Trace-Context
    response = requests.get(url, headers=headers)
    return response

# With auto-instrumentation (RequestsInstrumentor), this is automatic
```

```go
// Go — propagating context
import (
    "net/http"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
)

func callDownstream(ctx context.Context, url string) (*http.Response, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    
    // Inject trace context into outgoing request headers
    otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))
    
    return http.DefaultClient.Do(req)
}
```

### Cross-Project Trace Propagation

```
Traces automatically propagate across GCP projects when:
• Services pass trace headers between them
• The destination project has Cloud Trace API enabled

The trace appears in both projects:
• Project A sees its spans + reference to Project B
• Project B sees its spans + the parent from Project A

IAM: The calling service does NOT need Trace permissions
in the destination project — each project writes its own spans.
```

---

## Part 9 — Sampling Strategies

### Sampling Overview

```
┌──────────────────────────────────────────────────────────────┐
│               SAMPLING STRATEGIES                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Why sample?                                                   │
│  • High-traffic services can generate millions of traces     │
│  • 100% tracing adds overhead and cost                       │
│  • Sampling keeps representative data at lower cost          │
│                                                                │
│  Types:                                                        │
│                                                                │
│  1. Head-based sampling (decided at trace start)              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Request arrives → Random decision: trace? Y/N       │    │
│  │  • Simple, low overhead                              │    │
│  │  • May miss rare errors (they aren't sampled)       │    │
│  │  • TraceIDRatioBased: hash trace ID, sample X%      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  2. Tail-based sampling (decided after trace completes)       │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Collect all spans → Decide: keep based on criteria  │    │
│  │  • Keep all errors, keep slow requests               │    │
│  │  • Requires OTel Collector with tail_sampling         │    │
│  │  • Higher resource usage (buffer all spans)          │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  3. Always-on for specific conditions                         │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Force trace = 1 for:                                │    │
│  │  • Errors (HTTP 5xx)                                 │    │
│  │  • Slow requests (latency > threshold)               │    │
│  │  • Specific endpoints (/api/checkout)                │    │
│  │  • Debug headers (X-Force-Trace: 1)                  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Configuring Sampling

```python
# Python — head-based sampling (10%)
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased

provider = TracerProvider(
    sampler=TraceIdRatioBased(0.1),  # 10% of traces
)

# Always sample errors (custom sampler)
from opentelemetry.sdk.trace.sampling import ParentBasedTraceIdRatio, ALWAYS_ON, Decision

class ErrorForceSampler:
    """Sample 10% normally, but always sample errors."""
    def __init__(self, ratio=0.1):
        self._ratio_sampler = TraceIdRatioBased(ratio)
    
    def should_sample(self, parent_context, trace_id, name, kind, attributes, links):
        # Always sample if parent says so
        result = self._ratio_sampler.should_sample(
            parent_context, trace_id, name, kind, attributes, links
        )
        
        # Force sample for error indicators
        if attributes and attributes.get("http.status_code", 0) >= 500:
            return SamplingResult(Decision.RECORD_AND_SAMPLE, attributes)
        
        return result
```

```yaml
# OTel Collector — tail-based sampling
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: slow-requests
        type: latency
        latency:
          threshold_ms: 1000
      - name: percentage
        type: probabilistic
        probabilistic:
          sampling_percentage: 10
```

### Recommended Sampling Rates

| Environment | Suggested Rate | Rationale |
|------------|----------------|-----------|
| Development | 100% | See everything while debugging |
| Staging | 50–100% | Catch issues before production |
| Production (low traffic) | 50–100% | Affordable volume |
| Production (medium traffic) | 10–25% | Balance cost vs visibility |
| Production (high traffic) | 1–5% | Enough for statistical analysis |
| Error traces | 100% | Always capture failures |
| Health checks | 0% | No value in tracing probes |

---

## Part 10 — Correlating Traces with Logs & Metrics

### Trace-Log Correlation

```
┌──────────────────────────────────────────────────────────────┐
│         TRACE ↔ LOG CORRELATION                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  How it works:                                                │
│  1. Include trace ID in log entries                           │
│  2. Cloud Logging links entries to Cloud Trace                │
│  3. In Log Explorer: click trace ID → see trace              │
│  4. In Trace detail: click "View logs" → see related logs    │
│                                                                │
│  ┌────────────────────────────┐    ┌───────────────────────┐ │
│  │  Cloud Trace               │    │  Cloud Logging        │ │
│  │                            │    │                       │ │
│  │  Trace: abc123             │◄──►│  Log entry:           │ │
│  │  ├── Span: api-gateway     │    │  trace: abc123        │ │
│  │  │   └── Span: order-svc   │    │  spanId: def456       │ │
│  │  │       └── Span: db      │    │  message: "DB error"  │ │
│  │                            │    │  severity: ERROR       │ │
│  │  [View logs]  ────────────►│    │  [View trace] ───────►│ │
│  └────────────────────────────┘    └───────────────────────┘ │
│                                                                │
│  For Cloud Run / App Engine / GKE (auto):                     │
│  • Trace context is automatically added to logs              │
│                                                                │
│  For custom apps (structured logging):                        │
│  • Add "logging.googleapis.com/trace" field                  │
│  • Add "logging.googleapis.com/spanId" field                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Adding Trace Context to Logs

```python
# Python — add trace context to structured logs
import json
from opentelemetry import trace

def log_with_trace(severity, message, **kwargs):
    span = trace.get_current_span()
    ctx = span.get_span_context()
    
    entry = {
        "severity": severity,
        "message": message,
        **kwargs,
    }
    
    if ctx.is_valid:
        project = "my-project"
        entry["logging.googleapis.com/trace"] = f"projects/{project}/traces/{format(ctx.trace_id, '032x')}"
        entry["logging.googleapis.com/spanId"] = format(ctx.span_id, '016x')
        entry["logging.googleapis.com/trace_sampled"] = ctx.trace_flags.sampled
    
    print(json.dumps(entry))

# Usage
log_with_trace("INFO", "Processing order", orderId="ORD-123")
log_with_trace("ERROR", "Payment failed", orderId="ORD-123", error="Declined")
```

### Trace-Metric Correlation

```
Exemplars connect metrics to traces:
• When a metric data point is recorded, attach a trace ID
• In Monitoring dashboards, click a data point → jump to trace
• Supported via OpenTelemetry exemplars

Cloud Monitoring also auto-correlates:
• Request count metric → filter traces by same time window
• Latency metric → find traces matching that percentile
```

---

## Part 11 — Trace API & Exporting

### Cloud Trace API

```bash
# Write a trace span via REST API
curl -X POST \
  "https://cloudtrace.googleapis.com/v2/projects/my-project/traces:batchWrite" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "spans": [
      {
        "name": "projects/my-project/traces/TRACE_ID/spans/SPAN_ID",
        "spanId": "SPAN_ID",
        "displayName": { "value": "process_order" },
        "startTime": "2026-05-17T10:30:00.000Z",
        "endTime": "2026-05-17T10:30:00.523Z",
        "attributes": {
          "attributeMap": {
            "order.id": { "stringValue": { "value": "ORD-123" } },
            "http.method": { "stringValue": { "value": "POST" } }
          }
        }
      }
    ]
  }'
```

### Exporting Traces to BigQuery

```
Cloud Trace does not have a direct export to BigQuery.
Workaround approaches:

1. Use OpenTelemetry Collector with multiple exporters:
   - Export to Cloud Trace (for UI)
   - Export to BigQuery (for long-term analytics)

2. Use Pub/Sub sink + Dataflow:
   - OTel Collector → Pub/Sub → Dataflow → BigQuery

3. Use Trace API to read and store:
   - Periodic batch reads via API → write to BigQuery
```

```yaml
# OTel Collector — dual export
exporters:
  googlecloud:
    project: my-project
  googlebigquery:
    project: my-project
    dataset: trace_analytics
    table: spans

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [googlecloud, googlebigquery]
```

---

## Part 12 — IAM & Security

### Trace IAM Roles

| Role | Description |
|------|-------------|
| `roles/cloudtrace.user` | Read traces (view in console, API reads) |
| `roles/cloudtrace.agent` | Write trace data (used by applications/agents) |
| `roles/cloudtrace.admin` | Full control (read + write + config) |

### Granting Permissions

```bash
# Allow an app's service account to write traces
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:my-app-sa@my-project.iam.gserviceaccount.com" \
    --role="roles/cloudtrace.agent"

# Allow developers to view traces
gcloud projects add-iam-policy-binding my-project \
    --member="group:developers@example.com" \
    --role="roles/cloudtrace.user"

# For GKE with Workload Identity
gcloud iam service-accounts add-iam-policy-binding \
    my-app-sa@my-project.iam.gserviceaccount.com \
    --member="serviceAccount:my-project.svc.id.goog[default/my-app-ksa]" \
    --role="roles/iam.workloadIdentityUser"
```

### Security Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│         TRACE SECURITY BEST PRACTICES                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Don't put sensitive data in span attributes               │
│     ✗ span.set_attribute("user.password", password)          │
│     ✗ span.set_attribute("credit_card", card_number)         │
│     ✓ span.set_attribute("user.id", hashed_user_id)          │
│                                                                │
│  2. Use least-privilege IAM                                    │
│     • Apps: cloudtrace.agent (write only)                    │
│     • Developers: cloudtrace.user (read only)                │
│     • Admins: cloudtrace.admin                               │
│                                                                │
│  3. Use Workload Identity for GKE (not node SA)              │
│                                                                │
│  4. Redact PII from span names and attributes                │
│     • Use IDs, not names/emails                              │
│     • Hash sensitive identifiers                              │
│                                                                │
│  5. Control sampling to limit data exposure                   │
│     • Fewer traces = less data in the system                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 13 — Troubleshooting & Best Practices

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No traces appearing | API not enabled | Enable `cloudtrace.googleapis.com` |
| No traces appearing | Missing IAM role | Grant `cloudtrace.agent` to SA |
| Broken traces (gaps) | Context not propagated | Ensure headers are forwarded in outgoing calls |
| Missing child spans | No instrumentation | Add OTel auto-instrumentation for libraries |
| All spans in one project | Cross-project not set up | Each service writes to its own project |
| High latency overhead | Synchronous export | Use `BatchSpanProcessor` (not `SimpleSpanProcessor`) |
| Too many traces | No sampling | Configure head/tail-based sampling |
| Spans not linked | Wrong propagator | Use `CloudTraceFormatPropagator` or W3C |

### Best Practices

```
┌──────────────────────────────────────────────────────────────┐
│            TRACING BEST PRACTICES                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Naming:                                                       │
│  • Use descriptive span names: "db.query.orders"             │
│  • Use lowercase with dots: "payment.charge"                 │
│  • Don't include variable data in names:                     │
│    ✗ "GET /users/12345"                                      │
│    ✓ "GET /users/:id"                                        │
│                                                                │
│  Attributes:                                                   │
│  • Follow OpenTelemetry semantic conventions                 │
│  • Include: http.method, http.status_code, db.system         │
│  • Add business context: order.id, payment.status            │
│  • Keep attribute count reasonable (< 32 per span)           │
│                                                                │
│  Spans:                                                        │
│  • Create spans for meaningful operations (not every line)   │
│  • Use auto-instrumentation for common libraries             │
│  • Set span status on errors                                 │
│  • Record exceptions with span.record_exception()            │
│                                                                │
│  Performance:                                                  │
│  • Always use BatchSpanProcessor                              │
│  • Configure appropriate sampling rate                        │
│  • Don't create spans in tight loops                         │
│  • Use async export (don't block request path)               │
│                                                                │
│  Organization:                                                 │
│  • Set service.name on every service                         │
│  • Set service.version for deployment tracking               │
│  • Set deployment.environment (prod/staging/dev)             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform — Enable Cloud Trace

```hcl
# Enable Cloud Trace API
resource "google_project_service" "trace" {
  project = var.project_id
  service = "cloudtrace.googleapis.com"
}

# Service account for tracing
resource "google_service_account" "trace_agent" {
  account_id   = "trace-agent"
  display_name = "Cloud Trace Agent"
  project      = var.project_id
}

# Grant trace agent role
resource "google_project_iam_member" "trace_agent" {
  project = var.project_id
  role    = "roles/cloudtrace.agent"
  member  = "serviceAccount:${google_service_account.trace_agent.email}"
}

# Grant trace viewer to developers
resource "google_project_iam_member" "trace_viewer" {
  project = var.project_id
  role    = "roles/cloudtrace.user"
  member  = "group:developers@example.com"
}
```

### Terraform — GKE with Workload Identity for Tracing

```hcl
# GKE cluster with Workload Identity
resource "google_container_cluster" "primary" {
  name     = "prod-cluster"
  location = "us-central1"
  project  = var.project_id

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
}

# Kubernetes service account
resource "google_service_account" "app_sa" {
  account_id   = "app-trace-sa"
  display_name = "App Tracing SA"
  project      = var.project_id
}

# Grant Trace Agent + Log Writer
resource "google_project_iam_member" "app_trace" {
  project = var.project_id
  role    = "roles/cloudtrace.agent"
  member  = "serviceAccount:${google_service_account.app_sa.email}"
}

resource "google_project_iam_member" "app_logs" {
  project = var.project_id
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.app_sa.email}"
}

# Workload Identity binding
resource "google_service_account_iam_member" "workload_identity" {
  service_account_id = google_service_account.app_sa.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[default/app-ksa]"
}
```

### Terraform — Cloud Run with Tracing

```hcl
resource "google_cloud_run_v2_service" "api" {
  name     = "checkout-api"
  location = "us-central1"
  project  = var.project_id

  template {
    service_account = google_service_account.app_sa.email

    containers {
      image = "gcr.io/${var.project_id}/checkout-api:latest"

      env {
        name  = "OTEL_SERVICE_NAME"
        value = "checkout-api"
      }
      env {
        name  = "OTEL_TRACES_EXPORTER"
        value = "google_cloud_trace"
      }
      env {
        name  = "OTEL_RESOURCE_ATTRIBUTES"
        value = "service.version=2.1.0,deployment.environment=production"
      }
    }
  }
}
```

### gcloud CLI Reference

```bash
# ─── Enable API ──────────────────────────────────────────────
gcloud services enable cloudtrace.googleapis.com

# ─── View Traces ─────────────────────────────────────────────
# List traces (requires gcloud alpha)
gcloud alpha trace traces list \
    --project=my-project \
    --filter="rootSpan.name:checkout" \
    --limit=20

# Get a specific trace
gcloud alpha trace traces describe TRACE_ID \
    --project=my-project

# ─── IAM ─────────────────────────────────────────────────────
# Grant trace agent (write)
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:SA_EMAIL" \
    --role="roles/cloudtrace.agent"

# Grant trace user (read)
gcloud projects add-iam-policy-binding my-project \
    --member="user:dev@example.com" \
    --role="roles/cloudtrace.user"

# ─── Testing ─────────────────────────────────────────────────
# Verify traces are being received
# Console → Trace → Trace list → check for recent traces

# Force a traced request (send with trace header)
curl -H "X-Cloud-Trace-Context: $(python3 -c 'import uuid; print(uuid.uuid4().hex)')/1;o=1" \
    https://my-service.run.app/api/test
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Full-Stack Microservice Tracing

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: END-TO-END MICROSERVICE OBSERVABILITY                  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Architecture:                                                         │
│  User → Cloud LB → API Gateway (Cloud Run)                           │
│                       ├── Auth Service (Cloud Run)                    │
│                       ├── Order Service (GKE)                         │
│                       │   ├── DB (Cloud SQL)                          │
│                       │   └── Cache (Memorystore)                     │
│                       ├── Payment Service (Cloud Run)                 │
│                       │   └── External Payment API                    │
│                       └── Notification Service (Cloud Functions)      │
│                                                                        │
│  Instrumentation strategy:                                             │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │ Service             │ Type             │ Instrumentation    │      │
│  │ API Gateway         │ Cloud Run        │ Auto + OTel SDK    │      │
│  │ Auth Service        │ Cloud Run        │ Auto + OTel SDK    │      │
│  │ Order Service       │ GKE              │ OTel SDK + Collector│     │
│  │ Payment Service     │ Cloud Run        │ Auto + OTel SDK    │      │
│  │ Notification        │ Cloud Functions  │ Auto (Gen 2)       │      │
│  │ DB (Cloud SQL)      │ Managed          │ OTel SQLAlchemy    │      │
│  │ Cache (Memorystore) │ Managed          │ OTel Redis         │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  Result in Cloud Trace:                                                │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Trace: 280ms total                                       │        │
│  │  ├── api-gateway (280ms) ████████████████████████████     │        │
│  │  │   ├── auth-svc (30ms) ███                              │        │
│  │  │   ├── order-svc (200ms) █████████████████████          │        │
│  │  │   │   ├── cloud-sql (40ms) ████                        │        │
│  │  │   │   ├── memorystore (5ms) █                          │        │
│  │  │   │   └── payment-svc (120ms) ████████████             │        │
│  │  │   │       └── ext-payment-api (80ms) ████████          │        │
│  │  │   └── notification (20ms) ██                           │        │
│  │  └───────────────────────────────────────────────────     │        │
│  │  0ms              100ms            200ms          280ms   │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Alerting (from trace data → metrics):                                │
│  • p99 latency > 500ms → PagerDuty                                   │
│  • Error span rate > 1% → Slack                                      │
│  • External API latency > 200ms → ticket                             │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```python
# Shared tracing setup (used by all Python services)
# tracing.py — reusable module

import os
from opentelemetry import trace, propagate
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.propagators.cloud_trace_propagator import CloudTraceFormatPropagator
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

def init_tracing(service_name, version="1.0.0"):
    resource = Resource.create({
        "service.name": service_name,
        "service.version": version,
        "deployment.environment": os.getenv("ENV", "production"),
    })
    
    provider = TracerProvider(resource=resource)
    provider.add_span_processor(
        BatchSpanProcessor(CloudTraceSpanExporter())
    )
    trace.set_tracer_provider(provider)
    propagate.set_global_textmap(CloudTraceFormatPropagator())
    
    # Auto-instrument outgoing HTTP
    RequestsInstrumentor().instrument()
    
    return trace.get_tracer(service_name)

def instrument_flask(app):
    FlaskInstrumentor().instrument_app(app)

def instrument_db(engine):
    SQLAlchemyInstrumentor().instrument(engine=engine)
```

### Pattern 2: Performance Regression Detection

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: AUTOMATED PERFORMANCE REGRESSION DETECTION             │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Goal: Automatically detect when a new deployment increases latency.  │
│                                                                        │
│  Pipeline:                                                             │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  1. Deploy new version (Cloud Deploy / Cloud Build)         │     │
│  │  2. Cloud Trace collects spans with service.version label   │     │
│  │  3. Cloud Monitoring alert compares current vs previous     │     │
│  │     version latency                                         │     │
│  │  4. If p95 increases > 20% → alert + auto-rollback         │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                        │
│  Monitoring setup:                                                     │
│  • Create SLO: 95% of requests under 200ms                           │
│  • Burn rate alert: 6x burn rate over 1h = rollback                  │
│  • Dashboard: latency by version (canary vs stable)                  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Latency Comparison Dashboard                             │        │
│  │                                                            │        │
│  │  Version v2.0 (stable):   p50=80ms   p95=150ms  p99=300ms│        │
│  │  Version v2.1 (canary):   p50=85ms   p95=220ms  p99=800ms│        │
│  │                                      ^^^^^^^^    ^^^^^^^^ │        │
│  │                                      ⚠ +47%     ⚠ +167%  │        │
│  │                                                            │        │
│  │  → ALERT triggered → auto-rollback initiated              │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Debugging Intermittent Latency Spikes

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: ROOT-CAUSE ANALYSIS FOR LATENCY SPIKES                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Symptom: Occasional p99 latency spikes to 5s+ (normal p99 = 300ms) │
│                                                                        │
│  Investigation workflow:                                               │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Step 1: Filter trace list for latency > 3000ms          │        │
│  │                                                          │        │
│  │  Step 2: Examine 5-10 slow traces for common pattern     │        │
│  │  • Same bottleneck span? → Specific service issue        │        │
│  │  • Same time of day? → Cron job / batch contention       │        │
│  │  • Same zone? → Infrastructure issue                     │        │
│  │                                                          │        │
│  │  Step 3: Found pattern — db.query span is 4s+ in slow   │        │
│  │  traces but 20ms normally                                │        │
│  │                                                          │        │
│  │  Step 4: Click "View logs" on slow db.query span         │        │
│  │  → Log shows: "Lock wait timeout exceeded"               │        │
│  │                                                          │        │
│  │  Step 5: Check Cloud SQL metrics at same timestamp       │        │
│  │  → CPU spike + active connections maxed                   │        │
│  │                                                          │        │
│  │  Step 6: Correlate with audit logs                        │        │
│  │  → Found: batch ETL job runs every hour, causing locks   │        │
│  │                                                          │        │
│  │  Step 7: Fix — move ETL to read replica                  │        │
│  │  → p99 stabilizes at 300ms                               │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Key tools used:                                                       │
│  • Cloud Trace: find slow traces + waterfall analysis                 │
│  • Cloud Logging: correlate span with log entries                     │
│  • Cloud Monitoring: check infrastructure metrics                      │
│  • Audit Logs: identify triggering event                              │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command / Location |
|--------|-------------------|
| View traces | Console → Trace → Trace list |
| Enable API | `gcloud services enable cloudtrace.googleapis.com` |
| Grant write access | `--role="roles/cloudtrace.agent"` |
| Grant read access | `--role="roles/cloudtrace.user"` |
| Force trace (curl) | `-H "X-Cloud-Trace-Context: TRACE_ID/1;o=1"` |
| OTel exporter (Python) | `opentelemetry-exporter-gcp-trace` |
| OTel exporter (Node) | `@google-cloud/opentelemetry-cloud-trace-exporter` |
| OTel exporter (Go) | `github.com/GoogleCloudPlatform/opentelemetry-operations-go/exporter/trace` |
| OTel exporter (Java) | `com.google.cloud:google-cloud-opentelemetry-trace` |
| GCP propagator | `CloudTraceFormatPropagator` |
| W3C header | `traceparent: 00-TRACE_ID-SPAN_ID-FLAGS` |
| GCP header | `X-Cloud-Trace-Context: TRACE_ID/SPAN_ID;o=FLAGS` |
| Free tier | 2.5M spans/month |
| Retention | 30 days |

---

## What is Distributed Tracing? (Beginner Explanation)

### The Package Tracking Analogy

Think of **tracing as tracking a package through a delivery system** — you see every stop it makes and how long each step takes.

```
┌──────────────────────────────────────────────────────────────┐
│         TRACING = PACKAGE TRACKING                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Package Delivery              Distributed Tracing            │
│  ─────────────────             ────────────────────            │
│                                                                │
│  Tracking number       ═══►   Trace ID                        │
│  (unique per package)          (unique per request)            │
│                                                                │
│  Warehouse stop        ═══►   Span (a unit of work)           │
│  (one location)                (one service or function)       │
│                                                                │
│  Arrived/departed      ═══►   Span start/end time             │
│  timestamps                    (how long each step took)       │
│                                                                │
│  "Sorting facility"    ═══►   Span name                       │
│  (stop name)                   ("Auth Service", "DB Query")    │
│                                                                │
│  "Delayed: weather"    ═══►   Span attributes                 │
│  (notes on the stop)           (http.status=500, db.query=...) │
│                                                                │
│  Full route map        ═══►   Waterfall view                  │
│  (all stops in order)          (all spans in a trace)          │
│                                                                │
│  "Stuck at customs"    ═══►   Bottleneck span                 │
│  (the slow stop)               (the slow service)              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Breaking It Down

| Concept | Package Analogy | What It Really Means |
|---------|----------------|---------------------|
| **Trace** | One package's full journey from sender to recipient | One user request's journey through all your microservices |
| **Span** | One stop along the way (warehouse, sorting facility, truck) | One unit of work — a service call, a database query, an API request |
| **Root span** | The origin warehouse where the package was first scanned | The first service that receives the user's request (e.g., API Gateway) |
| **Child span** | A sub-stop within a facility (scanning, weighing, loading) | A sub-operation within a service (auth check, cache lookup) |
| **Latency** | How long the package sat at one stop | How long one span took to complete |
| **Waterfall view** | The timeline showing every stop and how long each took | Visual chart showing all spans in a trace, nested and timed |
| **Bottleneck** | The stop where the package got stuck the longest | The span that took the most time — the thing to optimize |

### Why Does Distributed Tracing Matter?

1. **Find the slow service** — When a request takes 5 seconds, tracing shows you exactly which service is responsible
2. **Understand dependencies** — See which services call which other services, and in what order
3. **Debug production issues** — Follow a specific request through your entire system to find where it failed
4. **Measure improvements** — Before/after comparisons show if your optimization actually helped
5. **Detect regressions** — p99 latency charts reveal when a deployment made things slower

> **One-liner**: If your microservices are a delivery network, Cloud Trace is the tracking system that shows every stop a request makes and how long it waited at each one.

---

## Console Walkthrough: Setting Up Tracing

### Enabling Cloud Trace

```
┌──────────────────────────────────────────────────────────────┐
│         ENABLE CLOUD TRACE                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Enable the API                                        │
│  Console → APIs & Services → Library                          │
│  Search for "Cloud Trace API" → Click → ENABLE                │
│                                                                │
│  Or via CLI:                                                   │
│  gcloud services enable cloudtrace.googleapis.com              │
│                                                                │
│  Step 2: Auto-instrumented services (no setup needed)         │
│  • Cloud Run: Incoming requests traced automatically          │
│  • App Engine: Traced automatically                           │
│  • Cloud Functions (Gen 2): Traced automatically              │
│                                                                │
│  Step 3: For custom services (GKE, Compute Engine)            │
│  • Add OpenTelemetry SDK to your application                  │
│  • Use the Cloud Trace exporter                               │
│  • Grant the service account roles/cloudtrace.agent           │
│                                                                │
│  ⚡ For auto-instrumented services, traces appear within       │
│    minutes of sending traffic. No code changes needed!        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Viewing Traces in Console

```
┌──────────────────────────────────────────────────────────────┐
│         VIEW TRACES IN CONSOLE                                │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Trace → Trace list                                 │
│                                                                │
│  Step 2: Filter traces                                         │
│  • Set the time range (e.g., "Last 1 hour")                  │
│  • Filter by root span name (e.g., "recv./api/checkout")     │
│  • Set minimum latency to find slow requests (e.g., 500ms)   │
│  • Filter by HTTP method (GET, POST) or status code (500)    │
│                                                                │
│  Step 3: Browse the scatter plot                               │
│  • Each dot is one trace — higher dots = higher latency       │
│  • Look for outlier dots (unusually slow requests)            │
│  • Click any dot to open the trace detail                     │
│                                                                │
│  Step 4: Open a trace                                          │
│  • Click a trace ID to see the waterfall view                 │
│  • See all spans nested by parent-child relationship          │
│  • Identify the longest span (your bottleneck)                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Analyzing Latency

```
┌──────────────────────────────────────────────────────────────┐
│         ANALYZE LATENCY                                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Trace → Analysis reports                           │
│                                                                │
│  Step 2: Select request type                                   │
│  • Choose the root span (e.g., "POST /api/checkout")         │
│  • Set the time range for analysis                            │
│                                                                │
│  Step 3: Review the latency distribution                      │
│  • Histogram shows how latency is distributed                │
│  • Check percentiles: p50, p75, p95, p99                     │
│  • Compare against your SLO targets                          │
│                                                                │
│  Step 4: Find bottlenecks                                      │
│  • "Most common bottleneck spans" section shows which        │
│    spans contribute most to total latency                     │
│  • Click a bottleneck span to see related traces              │
│                                                                │
│  Step 5: Compare over time                                     │
│  • Compare "Today vs Yesterday" to spot regressions          │
│  • Check if a recent deployment made things slower            │
│                                                                │
│  ⚡ Tip: Focus on p99 latency — it represents your worst      │
│    user experience. If p99 is within SLO, most users are     │
│    happy.                                                      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 39: Error Reporting & Profiler** → `39-error-reporting-profiler.md`
