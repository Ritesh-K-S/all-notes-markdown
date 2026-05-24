# Chapter 38: X-Ray

---

## Table of Contents

- [Overview](#overview)
- [Part 1: X-Ray Fundamentals](#part-1-x-ray-fundamentals)
- [Part 2: X-Ray Setup & Instrumentation](#part-2-x-ray-setup--instrumentation)
- [Part 3: X-Ray Console Walkthrough](#part-3-x-ray-console-walkthrough)
- [Part 4: Service Map, Traces & Analytics](#part-4-service-map-traces--analytics)
- [Part 5: X-Ray with Lambda, ECS & API Gateway](#part-5-x-ray-with-lambda-ecs--api-gateway)
- [Part 6: Sampling Rules & Groups](#part-6-sampling-rules--groups)
- [Part 7: Terraform & CLI Examples](#part-7-terraform--cli-examples)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Distributed Tracing? Why Do We Need X-Ray?

Imagine you order a package online. You can track it: warehouse → sorting facility → local depot → delivery truck → your door. If the package is delayed, you can see exactly **which stop** caused the delay.

**X-Ray does the same thing for API requests in your application.** When a user makes a request, it might travel through: API Gateway → Lambda → DynamoDB → another Lambda → S3. If the request is slow, X-Ray shows you exactly which service took the longest.

**Why this matters:**
- Without X-Ray: "The API is slow" → no idea which of 10 services is the bottleneck
- With X-Ray: "The API is slow because the DynamoDB query in the Order Service takes 2 seconds" → fix targeted

AWS X-Ray provides distributed tracing for applications. It traces requests as they flow through microservices, helping you identify performance bottlenecks, errors, and dependencies.

```
What you'll learn:
├── X-Ray Fundamentals (traces, segments, subsegments)
├── Instrumentation (SDK, daemon, auto-instrumentation)
├── Console walkthrough (service map, trace details)
├── Service map, traces, and analytics
├── Integration with Lambda, ECS, API Gateway
├── Sampling rules & groups
└── CLI & Terraform examples
```

---

## Part 1: X-Ray Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW X-RAY WORKS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Request flow through microservices:                                 │
│                                                                       │
│ Client → API Gateway → Lambda → DynamoDB                          │
│                           │                                          │
│                           └──→ SQS → Lambda → S3                   │
│                                                                       │
│ X-Ray traces the ENTIRE request path:                              │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Trace (one end-to-end request)                               │  │
│ │ ┌────────────────────────────────────────────────────────┐  │  │
│ │ │ Segment: API Gateway (20ms)                             │  │  │
│ │ │ ┌──────────────────────────────────────────────────┐   │  │  │
│ │ │ │ Segment: Lambda Function (150ms)                  │   │  │  │
│ │ │ │ ┌────────────────────┐ ┌──────────────────────┐ │   │  │  │
│ │ │ │ │ Subseg: DynamoDB   │ │ Subseg: SQS SendMsg  │ │   │  │  │
│ │ │ │ │ (30ms)             │ │ (10ms)               │ │   │  │  │
│ │ │ │ └────────────────────┘ └──────────────────────┘ │   │  │  │
│ │ │ └──────────────────────────────────────────────────┘   │  │  │
│ │ └────────────────────────────────────────────────────────┘  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Trace: Full request journey (unique Trace ID)               │
│ ├── Segment: Work done by one service (API GW, Lambda, etc.)   │
│ ├── Subsegment: Downstream call within a segment (DynamoDB)    │
│ ├── Annotations: Key-value indexed data (searchable)           │
│ ├── Metadata: Non-indexed extra data                            │
│ ├── Sampling: % of requests to trace (not 100%)               │
│ └── X-Ray Daemon: Receives segments, sends to X-Ray API       │
│                                                                       │
│ Components:                                                          │
│ ├── X-Ray SDK: Instrument your code                             │
│ ├── X-Ray Daemon: Runs alongside app, batches+sends data      │
│ ├── X-Ray API: Backend service                                   │
│ └── X-Ray Console: Visualization (service map, traces)         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: X-Ray Setup & Instrumentation

```
┌─────────────────────────────────────────────────────────────────────┐
│           INSTRUMENTATION                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Method 1: X-Ray SDK (manual instrumentation)                       │
│ # Python example                                                     │
│ from aws_xray_sdk.core import xray_recorder                       │
│ from aws_xray_sdk.core import patch_all                             │
│ patch_all()  # Auto-instrument AWS SDK, HTTP, SQL calls            │
│                                                                       │
│ @xray_recorder.capture('my_function')                              │
│ def process_order(order):                                           │
│     # Subsegment created automatically                             │
│     xray_recorder.current_subsegment().put_annotation(            │
│         'order_id', order['id']                                    │
│     )                                                                │
│     result = dynamodb.get_item(...)  # Auto-traced                │
│     return result                                                    │
│                                                                       │
│ Method 2: AWS Distro for OpenTelemetry (ADOT)                     │
│ ├── OpenTelemetry-based (vendor-neutral)                         │
│ ├── ⚡ Recommended for new projects                               │
│ ├── Supports X-Ray + CloudWatch + 3rd party backends            │
│ └── Auto-instrumentation for Java, Python, Node.js, .NET       │
│                                                                       │
│ Method 3: Auto-instrumentation (no code changes)                  │
│ ├── Lambda: Enable "Active tracing" in function config          │
│ ├── API Gateway: Enable tracing in stage settings               │
│ ├── ECS: Add X-Ray daemon as sidecar container                  │
│ └── Elastic Beanstalk: Enable in environment configuration     │
│                                                                       │
│ X-Ray Daemon:                                                        │
│ ├── Runs on EC2, ECS (sidecar), or Beanstalk                   │
│ ├── Listens on UDP port 2000                                     │
│ ├── Batches segments and sends to X-Ray API                     │
│ ├── Not needed for Lambda (built-in)                             │
│ └── IAM permissions: AWSXRayDaemonWriteAccess                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: X-Ray Console Walkthrough

```
Console → CloudWatch → X-Ray traces → Service map

┌─────────────────────────────────────────────────────────────────┐
│           SERVICE MAP                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Visual graph showing your microservice architecture:           │
│                                                                   │
│ ┌────────┐     ┌──────────┐     ┌──────────┐                  │
│ │ Client │────►│API Gateway│────►│ Lambda   │                  │
│ │ (edge) │     │ 50 req/s │     │ 48 req/s │                  │
│ │        │     │ 2ms avg  │     │ 150ms avg│                  │
│ └────────┘     └──────────┘     └──────────┘                  │
│                                    │     │                     │
│                              ┌─────┘     └──────┐             │
│                              ▼                   ▼             │
│                         ┌──────────┐      ┌──────────┐        │
│                         │ DynamoDB │      │   SQS    │        │
│                         │ 45 req/s │      │ 3 req/s  │        │
│                         │ 5ms avg  │      │ 8ms avg  │        │
│                         │ 0% error │      │ 0% error │        │
│                         └──────────┘      └──────────┘        │
│                                                                   │
│ Each node shows:                                               │
│ ├── Service name and type                                     │
│ ├── Request rate (requests/second)                            │
│ ├── Average latency                                            │
│ ├── Error rate (color: green=ok, yellow=4xx, red=5xx)        │
│ └── Click node for detailed metrics                           │
│                                                                   │
│ Filters:                                                       │
│ ├── Time range: Last 5 min to 6 hours                        │
│ ├── Filter by: Service, Edge, Annotation, Status code        │
│ └── Group by: Default, Account, Custom group                 │
└─────────────────────────────────────────────────────────────────┘

Console → CloudWatch → X-Ray traces → Traces

┌─────────────────────────────────────────────────────────────────┐
│           TRACE DETAILS                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Trace ID: 1-65a1b2c3-abcdef012345678901234567                 │
│ Duration: 185ms                                                │
│ Status: 200 OK                                                 │
│                                                                   │
│ Timeline (waterfall view):                                     │
│ ├─ API Gateway ████ (20ms)                                    │
│ ├─ Lambda Init ██ (50ms) [cold start]                        │
│ ├─ Lambda Handler ████████████ (100ms)                        │
│ │  ├─ DynamoDB GetItem ███ (30ms)                            │
│ │  ├─ DynamoDB PutItem ██ (15ms)                             │
│ │  └─ SQS SendMessage █ (10ms)                               │
│ └─ Lambda Overhead █ (5ms)                                    │
│                                                                   │
│ Segment details:                                               │
│ ├── Request: Method, URL, status code                        │
│ ├── Response: Size, status                                    │
│ ├── Annotations: order_id=12345 (searchable!)                │
│ ├── Metadata: Full request/response bodies                   │
│ ├── Exceptions: Stack trace if error                          │
│ └── Subsegments: Each downstream call                         │
│                                                                   │
│ ⚡ Use annotations to search traces by business data:          │
│   "Find all traces for order_id=12345"                        │
│   "Find traces where customer_tier=premium AND error=true"   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Service Map, Traces & Analytics

```
┌─────────────────────────────────────────────────────────────────────┐
│           X-RAY ANALYTICS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → X-Ray traces → Analytics                                 │
│                                                                       │
│ Response time distribution:                                         │
│ ├── Histogram of response times                                  │
│ ├── Identify percentiles (p50, p95, p99)                        │
│ └── Find outlier slow requests                                   │
│                                                                       │
│ Trace comparison:                                                    │
│ ├── Compare fast vs slow traces                                  │
│ ├── See which subsegment is causing slowness                    │
│ └── Root cause analysis                                          │
│                                                                       │
│ Filter expressions:                                                 │
│ # Find error traces                                                │
│ service("my-api") { fault = true }                                │
│                                                                       │
│ # Find slow Lambda traces                                          │
│ service("my-function") { responsetime > 5 }                       │
│                                                                       │
│ # Find traces by annotation                                       │
│ annotation.order_id = "12345"                                     │
│                                                                       │
│ # Combine filters                                                   │
│ service("my-api") AND http.status = 500                           │
│   AND annotation.customer_tier = "premium"                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: X-Ray with Lambda, ECS & API Gateway

```
┌─────────────────────────────────────────────────────────────────────┐
│           SERVICE-SPECIFIC SETUP                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Lambda:                                                              │
│ Console → Lambda → [Function] → Configuration → Monitoring        │
│ Active tracing: ☑ Enable                                           │
│ → No daemon needed (built into Lambda runtime)                   │
│ → Adds ~25ms to cold start                                        │
│ → IAM: AWSXRayDaemonWriteAccess on execution role               │
│ → Use X-Ray SDK to add custom subsegments/annotations           │
│                                                                       │
│ API Gateway:                                                        │
│ Console → API Gateway → [API] → Stages → [Stage]                 │
│ Logs/Tracing → X-Ray Tracing: ☑ Enable                           │
│ → Traces entire API request lifecycle                             │
│ → Shows: Integration latency, Gateway overhead                   │
│                                                                       │
│ ECS:                                                                 │
│ Add X-Ray daemon as sidecar container:                             │
│ {                                                                    │
│   "name": "xray-daemon",                                           │
│   "image": "amazon/aws-xray-daemon",                              │
│   "portMappings": [{"containerPort": 2000, "protocol": "udp"}], │
│   "cpu": 32, "memoryReservation": 256                             │
│ }                                                                    │
│ → App container sends segments to localhost:2000/UDP             │
│                                                                       │
│ EC2:                                                                 │
│ # Install daemon                                                    │
│ curl https://s3.amazonaws.com/aws-xray-assets.us-east-1/         │
│   xray-daemon/aws-xray-daemon-3.x.rpm -o xray.rpm               │
│ sudo rpm -i xray.rpm                                               │
│ sudo systemctl start xray                                          │
│ → IAM role: AWSXRayDaemonWriteAccess                             │
│                                                                       │
│ Elastic Beanstalk:                                                  │
│ Console → Beanstalk → [Env] → Configuration → Software          │
│ X-Ray daemon: ☑ Enabled                                           │
│ → Daemon auto-deployed and managed                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Sampling Rules & Groups

```
┌─────────────────────────────────────────────────────────────────────┐
│           SAMPLING RULES                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → X-Ray → Configuration → Sampling rules                  │
│                                                                       │
│ Why sample: Tracing 100% of requests is expensive and noisy.     │
│ Sampling traces a percentage of requests.                          │
│                                                                       │
│ Default rule:                                                        │
│ ├── Reservoir: 1 request/second (always traced)                 │
│ ├── Fixed rate: 5% of additional requests                       │
│ └── This traces ~5% of traffic + at least 1/sec                │
│                                                                       │
│ Custom rule (example):                                               │
│ Rule name: [trace-errors]                                           │
│ Priority: [100] (lower = higher priority)                          │
│ Reservoir: [10] requests/second                                    │
│ Fixed rate: [100] %                                                 │
│ Service name: [my-api]                                              │
│ Service type: [*]                                                    │
│ HTTP method: [*]                                                     │
│ URL path: [*]                                                        │
│ Host: [*]                                                            │
│ → Traces 100% of requests to "my-api"                            │
│                                                                       │
│ ⚡ Strategy:                                                         │
│ ├── 100% for error responses (trace all failures)               │
│ ├── 10% for normal traffic                                       │
│ ├── 100% for critical API paths (/checkout, /payment)          │
│ └── 1% for health check endpoints                                │
│                                                                       │
│ ── GROUPS ──                                                        │
│ Console → X-Ray → Configuration → Groups                          │
│ Filter expression: service("payment-api") { fault = true }       │
│ → Group traces for focused analysis                              │
│ → Set CloudWatch alarms on group metrics                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & CLI Examples

```hcl
# Lambda with X-Ray tracing
resource "aws_lambda_function" "api" {
  function_name = "my-api"
  # ... other config ...

  tracing_config {
    mode = "Active"  # Active or PassThrough
  }
}

# API Gateway with X-Ray
resource "aws_api_gateway_stage" "prod" {
  stage_name    = "prod"
  rest_api_id   = aws_api_gateway_rest_api.api.id
  deployment_id = aws_api_gateway_deployment.prod.id

  xray_tracing_enabled = true
}

# X-Ray sampling rule
resource "aws_xray_sampling_rule" "errors" {
  rule_name      = "trace-all-errors"
  priority       = 100
  reservoir_size = 10
  fixed_rate     = 1.0
  url_path       = "*"
  host           = "*"
  http_method    = "*"
  service_type   = "*"
  service_name   = "my-api"
  resource_arn   = "*"
  version        = 1
}
```

```bash
# Get service graph
aws xray get-service-graph \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T01:00:00Z

# Get trace summaries
aws xray get-trace-summaries \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T01:00:00Z \
  --filter-expression 'service("my-api") { fault = true }'

# Get trace details
aws xray batch-get-traces \
  --trace-ids "1-65a1b2c3-abcdef012345678901234567"
```

---

## Real-World Patterns

### Pattern 1: Debugging Slow API Responses

```
Problem: Users report "the checkout page takes 10 seconds"

X-Ray Service Map shows:
  API Gateway (50ms) → Lambda (200ms) → DynamoDB (8500ms!) → SNS (100ms)
                                              ↑
                                         BOTTLENECK FOUND

Root cause: Full table scan instead of query on GSI
Fix: Add GSI for the access pattern → DynamoDB drops to 5ms
```

### Pattern 2: Identifying Failing Downstream Services

```
Problem: Intermittent 500 errors in production

X-Ray Trace view shows:
  Order-Service → Payment-Service → Stripe API (502 error, 30% of traces)
  
Action: 
  1. Add retry with exponential backoff for Stripe calls
  2. Add circuit breaker pattern
  3. Alert on X-Ray error rate > 5%
```

### Pattern 3: X-Ray + CloudWatch Integration

```
Monitoring stack:
├── X-Ray: WHERE is the problem? (trace requests across services)
├── CloudWatch Metrics: WHAT is happening? (CPU, memory, error rates)
├── CloudWatch Logs: WHY did it happen? (error messages, stack traces)
└── CloudTrail: WHO did what? (API calls, config changes)

Together they answer: Who changed what, when did errors start,
which service is slow, and why is it failing.
```

---

## Quick Reference

```
X-Ray Quick Reference:
├── What: Distributed tracing for microservices
├── Concepts: Trace → Segments → Subsegments
├── Annotations: Indexed key-value (searchable)
├── Metadata: Non-indexed extra data
├── Sampling: Default 1/sec + 5% (customizable)
├── Lambda: Enable "Active tracing" (no daemon needed)
├── API Gateway: Enable in stage settings
├── ECS: X-Ray daemon as sidecar container
├── EC2: Install X-Ray daemon + SDK
├── Service Map: Visual architecture + latency/errors
├── ADOT: ⚡ Recommended (OpenTelemetry-based)
└── ⚡ Use annotations for business-level trace search
```

---

## What's Next?

In **Chapter 39: AWS Config**, we'll cover resource configuration tracking, compliance rules, and automated remediation for configuration drift.
