# Chapter 16: GCP Cloud Functions

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud Functions Fundamentals](#part-1-cloud-functions-fundamentals)
- [Part 2: Creating a Cloud Function (Full Walkthrough — Gen 2)](#part-2-creating-a-cloud-function-full-walkthrough--gen-2)
- [Part 3: Triggers (Deep Dive)](#part-3-triggers-deep-dive)
- [Part 4: Concurrency & Scaling](#part-4-concurrency--scaling)
- [Part 5: VPC Connector & Networking](#part-5-vpc-connector--networking)
- [Part 6: Terraform & gcloud CLI](#part-6-terraform--gcloud-cli)
- [Part 7: Real-World Patterns](#part-7-real-world-patterns)
- [Console Walkthrough: Managing & Deleting Functions](#console-walkthrough-managing--deleting-functions)
- [Local Development & Testing](#local-development--testing)
- [Connecting Cloud Functions to Cloud SQL](#connecting-cloud-functions-to-cloud-sql)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Cloud Functions is Google Cloud's serverless compute platform for running event-driven code without managing servers. You write a function, deploy it, and GCP handles scaling (from zero to thousands), infrastructure, patching, and availability. Cloud Functions has two generations: Gen 1 (original) and Gen 2 (built on Cloud Run, recommended for new projects).

```
What you'll learn:
├── Cloud Functions Fundamentals
│   ├── Serverless concept (event-driven, pay-per-use)
│   ├── Gen 1 vs Gen 2 (differences & migration)
│   ├── Execution model (cold start, instance reuse)
│   └── Pricing
├── Creating a Function (Full Walkthrough)
│   ├── Gen 2 creation (recommended)
│   ├── Trigger configuration
│   ├── Runtime settings
│   ├── Source code (inline, upload, Cloud Source Repo, GitHub)
│   └── Networking & security
├── Triggers
│   ├── HTTP triggers (REST APIs, webhooks)
│   ├── Eventarc triggers (Cloud Events)
│   ├── Cloud Storage events
│   ├── Pub/Sub messages
│   ├── Firestore/Firebase events
│   ├── Cloud Scheduler (cron)
│   └── Audit Log triggers
├── Runtimes (Node.js, Python, Go, Java, .NET, Ruby, PHP)
├── Environment Variables & Secrets
├── Concurrency & Scaling
├── VPC Connector (access private resources)
├── Terraform & gcloud CLI
└── Real-world patterns
```

---

## Part 1: Cloud Functions Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD FUNCTIONS: HOW IT WORKS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────┐   Event    ┌────────────────┐  Invoke  ┌──────────┐  │
│ │ Trigger  │ ─────────► │ Cloud Functions│ ───────► │ Your     │  │
│ │ (HTTP,   │            │ Service        │          │ Function │  │
│ │  Storage,│            │ (auto-scales)  │          │ Code     │  │
│ │  Pub/Sub)│            └────────────────┘          └──────────┘  │
│ └──────────┘                                                       │
│                                                                       │
│ Key characteristics:                                                 │
│ ├── No servers to manage (GCP handles everything)               │
│ ├── Scales automatically: 0 → thousands of instances           │
│ ├── Pay only when running (per invocation + per second)        │
│ ├── Event-driven: Triggered by HTTP, events, schedules         │
│ └── Stateless: Each invocation is independent                   │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ GEN 1 vs GEN 2                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ ┌──────────────────────┬────────────────┬────────────────────────┐│
│ │ Feature              │ Gen 1          │ Gen 2 (recommended)   ││
│ ├──────────────────────┼────────────────┼────────────────────────┤│
│ │ Built on             │ Custom infra   │ Cloud Run ✅           ││
│ │ Concurrency          │ 1 req/instance │ Up to 1000 req/inst ✅││
│ │ Max timeout          │ 9 minutes      │ 60 minutes ✅          ││
│ │ Max memory           │ 8 GB           │ 32 GB ✅               ││
│ │ Max vCPUs            │ 2              │ 8 ✅                   ││
│ │ Min instances        │ No             │ Yes ✅ (warm start)    ││
│ │ Traffic splitting    │ No             │ Yes ✅ (canary/A-B)    ││
│ │ Triggers             │ HTTP, Pub/Sub, │ HTTP, Eventarc         ││
│ │                      │ Storage, Fire. │ (100+ event sources) ✅││
│ │ Pricing              │ Per invocation │ Per invocation + CPU   ││
│ │ Revision management  │ No             │ Yes ✅ (Cloud Run)     ││
│ └──────────────────────┴────────────────┴────────────────────────┘│
│                                                                       │
│ ⚡ Gen 2 advantages:                                                  │
│ ├── Concurrency: Handle 1000 requests per instance (Gen 1 = 1!)│
│ │   → Fewer instances, lower cold starts, lower cost           │
│ ├── Longer timeout: 60 min vs 9 min (process large files!)    │
│ ├── More resources: 32 GB RAM, 8 vCPUs                        │
│ ├── Min instances: Keep instances warm (no cold starts!)       │
│ ├── Traffic splitting: Route % to different revisions (canary) │
│ └── Eventarc: 100+ event sources (Audit Logs, BigQuery, etc.)│
│                                                                       │
│ ⚡ ALWAYS use Gen 2 for new functions. Gen 1 is maintained but   │
│    not recommended.                                                │
│                                                                       │
│ Execution model:                                                     │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ COLD START: New instance created                             │   │
│ │ ┌──────────┐ ┌──────────────┐ ┌──────────┐ ┌──────────┐  │   │
│ │ │ Download │→│ Start        │→│ Initialize│→│ Run      │  │   │
│ │ │ code     │ │ runtime      │ │ global    │ │ function │  │   │
│ │ │          │ │ (container)  │ │ scope     │ │          │  │   │
│ │ └──────────┘ └──────────────┘ └──────────┘ └──────────┘  │   │
│ │ ↑___________100-500ms overhead____________↑               │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ WARM START: Instance reused (Gen 2 + concurrency!)          │   │
│ │                              ┌──────────┐                    │   │
│ │ Instance already exists  →  │ Run      │  (fast <10ms)     │   │
│ │ + handles concurrent reqs   │ function │                    │   │
│ │                              └──────────┘                    │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Pricing (Gen 2):                                                    │
│ ├── Invocations: $0.40 per million (first 2M free!)            │
│ ├── Compute time: $0.0000025/GHz-sec + $0.0000025/GB-sec      │
│ ├── Networking: Egress charges (outbound data)                 │
│ ├── Free tier: 2M invocations/month, 400K GB-seconds           │
│ └── ⚡ Very cheap for event-driven workloads                    │
│                                                                       │
│ Limits (Gen 2):                                                     │
│ ├── Timeout: 60 minutes (HTTP), 9 min (event-triggered)       │
│ ├── Memory: 128 MB - 32 GB                                     │
│ ├── vCPUs: Up to 8                                              │
│ ├── Concurrent requests per instance: 1-1000                   │
│ ├── Max instances: 100 default (can increase to 3000)         │
│ ├── Deploy source: 100 MB (compressed), 500 MB (uncompressed) │
│ └── Request/response size: 32 MB                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Cloud Function (Full Walkthrough — Gen 2)

```
Console → Cloud Functions → Create Function

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE FUNCTION (Gen 2)                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Basics ──                                                        │
│                                                                       │
│ Environment: [2nd gen ▼] ← ALWAYS use 2nd gen                    │
│                                                                       │
│ Function name: [order-processor]                                   │
│ ⚡ Naming: {service}-{action} or {project}-{purpose}               │
│                                                                       │
│ Region: [asia-south1 (Mumbai) ▼]                                  │
│ ⚡ Deploy close to your users and data.                             │
│                                                                       │
│ ── Trigger ──                                                       │
│                                                                       │
│ Trigger type:                                                       │
│   ● HTTPS                                                          │
│   ○ Cloud Pub/Sub                                                  │
│   ○ Cloud Storage                                                  │
│   ○ Eventarc (all other events)                                   │
│                                                                       │
│ ═══ HTTPS trigger ═══                                               │
│ URL: https://asia-south1-myproject.cloudfunctions.net/order-processor│
│ ⚡ Auto-generated HTTPS endpoint.                                   │
│                                                                       │
│ Authentication:                                                     │
│   ○ Require authentication (IAM — recommended for internal APIs)│
│   ● Allow unauthenticated invocations (public endpoint)         │
│   ⚡ "Require auth" = caller needs roles/cloudfunctions.invoker  │
│   ⚡ "Allow unauth" = anyone can call (public webhooks, APIs)    │
│                                                                       │
│ ═══ Cloud Pub/Sub trigger ═══                                      │
│ Topic: [orders-topic ▼]  [Create a topic]                        │
│ Retry on failure: ☑                                                │
│ ⚡ Function invoked for each message published to the topic.      │
│                                                                       │
│ ═══ Cloud Storage trigger ═══                                      │
│ Bucket: [my-uploads-bucket ▼]                                     │
│ Event type:                                                         │
│   ● google.cloud.storage.object.v1.finalized (object created)   │
│   ○ google.cloud.storage.object.v1.deleted                       │
│   ○ google.cloud.storage.object.v1.archived                     │
│   ○ google.cloud.storage.object.v1.metadataUpdated              │
│ Retry on failure: ☑                                                │
│                                                                       │
│ ═══ Eventarc trigger ═══                                           │
│ Event provider: [Cloud Audit Logs ▼]                              │
│   ○ Cloud Audit Logs (any API call as event!)                    │
│   ○ Cloud Storage                                                 │
│   ○ BigQuery                                                      │
│   ○ Firestore                                                     │
│   ○ Cloud Memorystore                                             │
│   ○ Workflows                                                     │
│   ○ Custom (any CloudEvents source)                              │
│                                                                       │
│ Event: [google.cloud.bigquery.v2.JobService.InsertJob]            │
│ ⚡ Eventarc turns ANY GCP API call into an event trigger!         │
│   "When someone creates a BigQuery job → run my function"       │
│                                                                       │
│ [Next: Runtime, build, connections and security]                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           RUNTIME, BUILD, CONNECTIONS AND SECURITY                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Runtime ──                                                       │
│                                                                       │
│ Memory allocated: [256 MiB ▼]                                     │
│   128 MiB │ 256 MiB │ 512 MiB │ 1 GiB │ 2 GiB │ 4 GiB │ 8 GiB│
│   │ 16 GiB │ 32 GiB                                               │
│ ⚡ CPU scales with memory:                                          │
│   ├── 128-256 MB = 0.083 vCPU                                   │
│   ├── 512 MB = 0.333 vCPU                                        │
│   ├── 1 GB = 0.583 vCPU                                          │
│   ├── 2 GB = 1 vCPU                                              │
│   ├── 4 GB = 2 vCPU                                              │
│   ├── 8 GB = 2 vCPU                                              │
│   └── 16-32 GB = 4-8 vCPU                                       │
│                                                                       │
│ CPU: [auto ▼]                                                      │
│   ● auto (scales with memory)                                    │
│   ○ 1 vCPU (override)                                            │
│   ○ 2 vCPU                                                        │
│   ○ 4 vCPU                                                        │
│   ○ 8 vCPU                                                        │
│                                                                       │
│ Timeout: [60] seconds                                               │
│ ⚡ Max: 3600s (60 min) for HTTP, 540s (9 min) for event-triggered│
│   Set to expected max execution + buffer.                        │
│                                                                       │
│ Maximum concurrent requests per instance: [80]                    │
│ ⚡ Gen 2 KILLER FEATURE! One instance handles 80 concurrent reqs.│
│   Gen 1 = always 1 request per instance.                        │
│   Set based on your function's ability to handle concurrent     │
│   work. CPU-bound: lower (1-10). I/O-bound: higher (80-1000).  │
│                                                                       │
│ Minimum instances: [0]                                              │
│ ⚡ Set to 1+ to eliminate cold starts (keeps instances warm).     │
│   0 = scale to zero (cheapest, but cold starts occur).          │
│   1 = always 1 warm instance (fast response, constant cost).    │
│                                                                       │
│ Maximum instances: [100]                                            │
│ ⚡ Cap to control costs and prevent runaway scaling.              │
│   Default: 100. Can increase to 3000 with quota request.        │
│                                                                       │
│ ── Environment variables ──                                         │
│                                                                       │
│ Runtime environment variables:                                      │
│ ┌──────────────────┬──────────────────────────────────────────┐   │
│ │ Name             │ Value                                      │   │
│ ├──────────────────┼──────────────────────────────────────────┤   │
│ │ DB_HOST          │ 10.0.1.5                                  │   │
│ │ ENVIRONMENT      │ production                                │   │
│ │ LOG_LEVEL        │ info                                      │   │
│ └──────────────────┴──────────────────────────────────────────┘   │
│ ⚡ Available as process.env.DB_HOST (Node) or os.environ (Python)│
│                                                                       │
│ Build environment variables:                                       │
│ (Available during build only, not at runtime)                     │
│ GOOGLE_FUNCTION_SOURCE: main.py                                   │
│                                                                       │
│ ── Secrets ──                                                       │
│                                                                       │
│ [+ Reference a secret]                                              │
│ Secret: [db-password ▼] (from Secret Manager)                    │
│ Reference method:                                                   │
│   ● Exposed as environment variable                               │
│     Environment variable: DB_PASSWORD                             │
│   ○ Mounted as volume                                             │
│     Mount path: /secrets/db-password                             │
│ Version: [latest ▼] or specific [3]                               │
│                                                                       │
│ ⚡ Secrets are fetched from Secret Manager at function start.      │
│   Function's service account needs:                               │
│   roles/secretmanager.secretAccessor                             │
│                                                                       │
│ ── Connections ──                                                   │
│                                                                       │
│ VPC connector:                                                      │
│ ├── [None ▼] or [connector-prod ▼]                              │
│ ├── ⚡ Required to access private VPC resources (Cloud SQL,      │
│ │   Memorystore Redis, internal VMs, GKE).                     │
│ ├── Egress settings:                                              │
│ │   ● Route only private IP traffic through VPC connector      │
│ │   ○ Route ALL traffic through VPC connector                   │
│ └── ⚡ Without connector: Function can only reach public internet│
│                                                                       │
│ Ingress settings:                                                   │
│   ● Allow all traffic                                             │
│   ○ Allow internal traffic only (VPC + Cloud Interconnect)      │
│   ○ Allow internal traffic and traffic from Cloud Load Balancing│
│   ⚡ For internal APIs: "Allow internal traffic only" (secure)   │
│                                                                       │
│ ── Security ──                                                      │
│                                                                       │
│ Service account: [sa-order-processor@project.iam.gserviceaccount.com ▼]│
│ ⚡ Use dedicated service account per function (least privilege!) │
│ ⚠️ Don't use default compute service account in production!     │
│                                                                       │
│ [Next: Code]                                                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           CODE                                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Runtime: [Python 3.12 ▼]                                           │
│   ○ Node.js 20                                                     │
│   ○ Node.js 18                                                     │
│   ● Python 3.12                                                    │
│   ○ Python 3.11                                                    │
│   ○ Go 1.22                                                       │
│   ○ Java 17                                                        │
│   ○ Java 11                                                        │
│   ○ .NET 8                                                         │
│   ○ Ruby 3.2                                                       │
│   ○ PHP 8.3                                                        │
│                                                                       │
│ Source code:                                                        │
│   ● Inline editor                                                  │
│   ○ ZIP upload                                                     │
│   ○ ZIP from Cloud Storage                                        │
│   ○ Cloud Source Repositories                                     │
│   ○ GitHub / Bitbucket (via Cloud Build trigger)                 │
│                                                                       │
│ Entry point: [order_processor]                                     │
│ ⚡ The function name in your code that handles requests.           │
│                                                                       │
│ ── main.py (Python example) ──                                     │
│                                                                       │
│ import functions_framework                                          │
│ import os                                                           │
│                                                                       │
│ @functions_framework.http                                           │
│ def order_processor(request):                                      │
│     """HTTP Cloud Function."""                                     │
│     db_host = os.environ.get('DB_HOST')                           │
│     request_json = request.get_json(silent=True)                 │
│                                                                       │
│     if request_json and 'order_id' in request_json:              │
│         order_id = request_json['order_id']                      │
│         # Process order...                                        │
│         return f'Order {order_id} processed', 200                │
│                                                                       │
│     return 'No order_id provided', 400                            │
│                                                                       │
│ ── requirements.txt ──                                              │
│ functions-framework==3.*                                            │
│ google-cloud-storage==2.*                                          │
│ google-cloud-firestore==2.*                                        │
│                                                                       │
│ ── index.js (Node.js example) ──                                   │
│                                                                       │
│ const functions = require('@google-cloud/functions-framework');   │
│                                                                       │
│ functions.http('orderProcessor', (req, res) => {                  │
│   const orderId = req.body.order_id;                              │
│   if (!orderId) {                                                  │
│     res.status(400).send('No order_id provided');                │
│     return;                                                        │
│   }                                                                │
│   // Process order...                                              │
│   res.send(`Order ${orderId} processed`);                        │
│ });                                                                 │
│                                                                       │
│ [Deploy]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Triggers (Deep Dive)

```
┌─────────────────────────────────────────────────────────────────────┐
│           TRIGGER TYPES                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. HTTP Trigger:                                                    │
│    ├── Direct HTTPS endpoint                                     │
│    ├── URL: https://REGION-PROJECT.cloudfunctions.net/FUNCTION   │
│    ├── Methods: GET, POST, PUT, DELETE, etc.                    │
│    ├── Auth: IAM (require invoker role) or public (unauth)     │
│    ├── Use for: REST APIs, webhooks, web endpoints             │
│    └── ⚡ AWS equivalent: API Gateway + Lambda                   │
│                                                                       │
│ 2. Cloud Pub/Sub Trigger:                                          │
│    ├── Function invoked per Pub/Sub message                     │
│    ├── Automatic retry on failure                                │
│    ├── Message data: base64 encoded                              │
│    ├── Use for: Async processing, event pipelines, fan-out     │
│    └── ⚡ AWS equivalent: SNS/SQS → Lambda                       │
│                                                                       │
│    @functions_framework.cloud_event                               │
│    def process_pubsub(cloud_event):                               │
│        import base64                                               │
│        data = base64.b64decode(                                   │
│            cloud_event.data["message"]["data"]                   │
│        ).decode()                                                  │
│        print(f"Received: {data}")                                 │
│                                                                       │
│ 3. Cloud Storage Trigger:                                          │
│    ├── Events: finalized (created), deleted, archived, metadata │
│    ├── Function invoked per object event                        │
│    ├── Event data: bucket, name, size, content type             │
│    ├── Use for: Image processing, file validation, ETL         │
│    └── ⚡ AWS equivalent: S3 event → Lambda                      │
│                                                                       │
│    @functions_framework.cloud_event                               │
│    def process_storage(cloud_event):                              │
│        data = cloud_event.data                                    │
│        bucket = data["bucket"]                                    │
│        name = data["name"]                                        │
│        print(f"File: gs://{bucket}/{name}")                      │
│                                                                       │
│ 4. Eventarc Triggers (Gen 2 only):                                 │
│    ├── Cloud Audit Log events (ANY GCP API call!)               │
│    │   "When someone creates a VM" → trigger function           │
│    │   "When someone creates BigQuery dataset" → trigger        │
│    ├── Cloud Storage (via Eventarc — preferred for Gen 2)      │
│    ├── Firestore document changes                                │
│    ├── Firebase Authentication events                           │
│    ├── Custom events (your own CloudEvents)                    │
│    └── ⚡ 100+ event sources! Way more than Gen 1.               │
│                                                                       │
│ 5. Cloud Scheduler (cron jobs):                                    │
│    ├── Not a direct trigger type — Scheduler → HTTP or Pub/Sub │
│    ├── Scheduler calls your HTTP function on a schedule         │
│    ├── Or: Scheduler publishes to Pub/Sub → triggers function  │
│    ├── Cron syntax: "0 8 * * *" (daily at 8 AM)               │
│    └── ⚡ AWS equivalent: EventBridge schedule → Lambda           │
│                                                                       │
│    Console → Cloud Scheduler → Create job:                       │
│    Name: daily-report                                             │
│    Frequency: 0 8 * * * (daily 8 AM)                            │
│    Timezone: Asia/Kolkata                                        │
│    Target: HTTP                                                   │
│    URL: https://..../daily-report                                │
│    Method: POST                                                   │
│    Auth header: Add OIDC token (service account)               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Concurrency & Scaling

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONCURRENCY & SCALING                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Gen 1 Concurrency:                                                   │
│ ├── 1 request per instance (always!)                            │
│ ├── 100 concurrent requests → 100 instances                    │
│ ├── More cold starts, higher cost                               │
│ └── Simple but wasteful                                          │
│                                                                       │
│ Gen 2 Concurrency (GAME CHANGER):                                  │
│ ├── Up to 1000 requests per instance                            │
│ ├── 100 concurrent requests → maybe just 2 instances!          │
│ ├── Fewer cold starts, lower cost                               │
│ └── Set based on function type:                                  │
│     ├── CPU-bound (image processing): 1-10 concurrent          │
│     ├── I/O-bound (API calls, DB queries): 50-200              │
│     └── Lightweight (logging, forwarding): 200-1000            │
│                                                                       │
│ Scaling:                                                             │
│ ┌───────────────────────────────────────────────────────────┐     │
│ │ Incoming requests: 500 concurrent                          │     │
│ │ Concurrency per instance: 80                              │     │
│ │ Instances needed: ceil(500/80) = 7 instances              │     │
│ │ vs Gen 1: 500 instances! (80× more)                      │     │
│ └───────────────────────────────────────────────────────────┘     │
│                                                                       │
│ Minimum instances:                                                   │
│ ├── 0: Scale to zero (cheapest, cold starts on first request)  │
│ ├── 1: Always 1 warm instance (eliminates most cold starts)    │
│ ├── N: Keep N instances warm (for high-traffic functions)      │
│ └── ⚡ Cost: You pay for idle min instances (CPU + memory)       │
│                                                                       │
│ Maximum instances:                                                   │
│ ├── Default: 100 (account-level quota)                         │
│ ├── Can increase to 3000 (quota request)                       │
│ ├── Set to limit costs and prevent runaway scaling             │
│ └── Requests beyond max → 429 Too Many Requests               │
│                                                                       │
│ CPU allocation:                                                      │
│ ├── CPU allocated only during request processing (default)     │
│ │   → Cheapest, but background work stops between requests    │
│ ├── CPU always allocated                                        │
│ │   → Background work continues, min-instance keeps running   │
│ │   → More expensive, needed for: WebSocket, background tasks │
│ └── ⚡ "Always allocated" needed if function does background work│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: VPC Connector & Networking

```
┌─────────────────────────────────────────────────────────────────────┐
│           VPC CONNECTOR                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ By default: Cloud Functions runs outside your VPC.                 │
│ Problem: Can't access private resources (Cloud SQL, Redis, VMs). │
│ Solution: Serverless VPC Access Connector.                        │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────┐      │
│ │ Without VPC Connector:                                    │      │
│ │ Cloud Function → Internet → Public Cloud SQL ❌          │      │
│ │ (Can't reach private IP resources)                       │      │
│ │                                                          │      │
│ │ With VPC Connector:                                      │      │
│ │ Cloud Function → VPC Connector → Private VPC             │      │
│ │                                   ├── Cloud SQL (10.0.1.5)│     │
│ │                                   ├── Redis (10.0.2.10)  │      │
│ │                                   └── GKE (10.0.3.0/24)  │      │
│ └──────────────────────────────────────────────────────────┘      │
│                                                                       │
│ Create VPC Connector:                                               │
│ Console → VPC network → Serverless VPC access → Create connector │
│                                                                       │
│ Name: [connector-prod]                                              │
│ Region: [asia-south1] (must match function region!)               │
│ Network: [vpc-prod]                                                 │
│ Subnet:                                                              │
│   ● Custom IP range: [10.8.0.0/28] (dedicated /28 range)       │
│   ○ Use existing subnet                                          │
│ ⚡ The /28 range (16 IPs) is used for connector instances.       │
│   Must not overlap with existing subnets!                        │
│                                                                       │
│ Scaling:                                                             │
│ Min instances: [2] (for availability)                              │
│ Max instances: [3]                                                  │
│ Instance type: [e2-micro ▼] (or f1-micro for cheapest)           │
│                                                                       │
│ ⚡ Alternative for Gen 2: Direct VPC Egress (no connector needed!)│
│   Cloud Functions (Gen 2) → can attach directly to VPC subnet   │
│   Saves connector cost ($8-15/month per connector).              │
│                                                                       │
│ AWS equivalent: Lambda VPC configuration                          │
│ Azure equivalent: VNet Integration                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & gcloud CLI

### Terraform

```hcl
# Cloud Function (Gen 2) — HTTP trigger
resource "google_cloudfunctions2_function" "order_processor" {
  name     = "order-processor"
  location = "asia-south1"
  
  build_config {
    runtime     = "python312"
    entry_point = "order_processor"
    
    source {
      storage_source {
        bucket = google_storage_bucket.source.name
        object = google_storage_bucket_object.source.name
      }
    }
  }
  
  service_config {
    max_instance_count               = 100
    min_instance_count               = 1
    available_memory                 = "256Mi"
    available_cpu                    = "1"
    timeout_seconds                  = 60
    max_instance_request_concurrency = 80
    
    environment_variables = {
      DB_HOST     = "10.0.1.5"
      ENVIRONMENT = "production"
      LOG_LEVEL   = "info"
    }
    
    secret_environment_variables {
      key        = "DB_PASSWORD"
      project_id = var.project_id
      secret     = google_secret_manager_secret.db_password.secret_id
      version    = "latest"
    }
    
    service_account_email = google_service_account.function_sa.email
    
    vpc_connector                 = google_vpc_access_connector.connector.id
    vpc_connector_egress_settings = "PRIVATE_RANGES_ONLY"
    ingress_settings              = "ALLOW_ALL"
  }
  
  labels = {
    environment = "prod"
    team        = "backend"
  }
}

# Make function publicly accessible (for HTTP trigger)
resource "google_cloud_run_service_iam_member" "invoker" {
  location = google_cloudfunctions2_function.order_processor.location
  service  = google_cloudfunctions2_function.order_processor.name
  role     = "roles/run.invoker"
  member   = "allUsers"
}

# Service account for the function
resource "google_service_account" "function_sa" {
  account_id   = "sa-order-processor"
  display_name = "Order Processor Function SA"
}

resource "google_project_iam_member" "function_storage" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.function_sa.email}"
}

# Source code bucket
resource "google_storage_bucket" "source" {
  name     = "${var.project_id}-function-source"
  location = "ASIA-SOUTH1"
}

resource "google_storage_bucket_object" "source" {
  name   = "order-processor-${data.archive_file.source.output_md5}.zip"
  bucket = google_storage_bucket.source.name
  source = data.archive_file.source.output_path
}

data "archive_file" "source" {
  type        = "zip"
  source_dir  = "${path.module}/src/order-processor"
  output_path = "${path.module}/tmp/order-processor.zip"
}

# Cloud Function — Pub/Sub trigger
resource "google_cloudfunctions2_function" "notification_handler" {
  name     = "notification-handler"
  location = "asia-south1"
  
  build_config {
    runtime     = "nodejs20"
    entry_point = "handleNotification"
    
    source {
      storage_source {
        bucket = google_storage_bucket.source.name
        object = google_storage_bucket_object.notification_source.name
      }
    }
  }
  
  service_config {
    max_instance_count               = 50
    min_instance_count               = 0
    available_memory                 = "256Mi"
    timeout_seconds                  = 60
    max_instance_request_concurrency = 1
    service_account_email            = google_service_account.function_sa.email
  }
  
  event_trigger {
    trigger_region = "asia-south1"
    event_type     = "google.cloud.pubsub.topic.v1.messagePublished"
    pubsub_topic   = google_pubsub_topic.notifications.id
    retry_policy   = "RETRY_POLICY_RETRY"
  }
}

# Cloud Function — Cloud Storage trigger
resource "google_cloudfunctions2_function" "image_processor" {
  name     = "image-processor"
  location = "asia-south1"
  
  build_config {
    runtime     = "python312"
    entry_point = "process_image"
    
    source {
      storage_source {
        bucket = google_storage_bucket.source.name
        object = google_storage_bucket_object.image_source.name
      }
    }
  }
  
  service_config {
    max_instance_count    = 20
    available_memory      = "1Gi"
    available_cpu         = "1"
    timeout_seconds       = 120
    service_account_email = google_service_account.function_sa.email
  }
  
  event_trigger {
    trigger_region        = "asia-south1"
    event_type            = "google.cloud.storage.object.v1.finalized"
    retry_policy          = "RETRY_POLICY_RETRY"
    service_account_email = google_service_account.function_sa.email
    
    event_filters {
      attribute = "bucket"
      value     = google_storage_bucket.uploads.name
    }
  }
}

# VPC Access Connector
resource "google_vpc_access_connector" "connector" {
  name          = "connector-prod"
  region        = "asia-south1"
  ip_cidr_range = "10.8.0.0/28"
  network       = google_compute_network.vpc.name
  
  min_instances = 2
  max_instances = 3
  machine_type  = "e2-micro"
}

# Cloud Scheduler (cron → HTTP function)
resource "google_cloud_scheduler_job" "daily_report" {
  name     = "daily-report"
  schedule = "0 8 * * *"
  time_zone = "Asia/Kolkata"
  
  http_target {
    uri         = google_cloudfunctions2_function.order_processor.url
    http_method = "POST"
    body        = base64encode("{\"action\": \"daily-report\"}")
    
    headers = {
      "Content-Type" = "application/json"
    }
    
    oidc_token {
      service_account_email = google_service_account.scheduler_sa.email
    }
  }
}
```

### gcloud CLI

```bash
# Deploy HTTP function (Gen 2)
gcloud functions deploy order-processor \
  --gen2 \
  --region=asia-south1 \
  --runtime=python312 \
  --entry-point=order_processor \
  --source=./src/order-processor \
  --trigger-http \
  --allow-unauthenticated \
  --memory=256Mi \
  --cpu=1 \
  --timeout=60s \
  --max-instances=100 \
  --min-instances=1 \
  --concurrency=80 \
  --service-account=sa-order-processor@project-id.iam.gserviceaccount.com \
  --set-env-vars="DB_HOST=10.0.1.5,ENVIRONMENT=production" \
  --set-secrets="DB_PASSWORD=db-password:latest" \
  --vpc-connector=connector-prod \
  --egress-settings=private-ranges-only

# Deploy Pub/Sub triggered function
gcloud functions deploy notification-handler \
  --gen2 \
  --region=asia-south1 \
  --runtime=nodejs20 \
  --entry-point=handleNotification \
  --source=./src/notifications \
  --trigger-topic=notifications-topic \
  --retry \
  --memory=256Mi \
  --max-instances=50 \
  --service-account=sa-notifications@project-id.iam.gserviceaccount.com

# Deploy Cloud Storage triggered function
gcloud functions deploy image-processor \
  --gen2 \
  --region=asia-south1 \
  --runtime=python312 \
  --entry-point=process_image \
  --source=./src/image-processor \
  --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
  --trigger-event-filters="bucket=my-uploads-bucket" \
  --retry \
  --memory=1Gi \
  --timeout=120s \
  --max-instances=20

# List functions
gcloud functions list --gen2 --region=asia-south1

# Describe function
gcloud functions describe order-processor \
  --gen2 --region=asia-south1

# View logs
gcloud functions logs read order-processor \
  --gen2 --region=asia-south1 --limit=50

# Test/call function
gcloud functions call order-processor \
  --gen2 --region=asia-south1 \
  --data='{"order_id": "12345"}'

# Update function
gcloud functions deploy order-processor \
  --gen2 --region=asia-south1 \
  --update-env-vars="LOG_LEVEL=debug"

# Delete function
gcloud functions delete order-processor \
  --gen2 --region=asia-south1

# Traffic splitting (canary — Gen 2 only)
# Gen 2 functions are Cloud Run services underneath
gcloud run services update-traffic order-processor \
  --region=asia-south1 \
  --to-revisions=order-processor-00002-abc=10,order-processor-00001-xyz=90

# 10% canary → monitor → shift to 100%
gcloud run services update-traffic order-processor \
  --region=asia-south1 \
  --to-latest
```

---

## Part 7: Real-World Patterns

### Startup

```
Functions-first architecture:

Functions:
├── api-handler (HTTP trigger — public API)
│   ├── Runtime: Node.js 20 or Python 3.12
│   ├── Memory: 256 Mi, concurrency: 80
│   ├── Min instances: 1 (no cold start for API)
│   └── Handles REST API routes
├── image-processor (Cloud Storage trigger)
│   ├── Triggered on upload to GCS bucket
│   ├── Resizes images, generates thumbnails
│   ├── Memory: 1 GiB (image processing)
│   └── Min instances: 0 (event-driven, can cold start)
├── email-sender (Pub/Sub trigger)
│   ├── Processes email queue messages
│   └── Min instances: 0
└── daily-report (Cloud Scheduler → HTTP)
    ├── Cron: 0 8 * * * (daily 8 AM IST)
    └── Generates and sends daily report

Architecture:
HTTP → Cloud Function → Firestore (or Cloud SQL)
GCS → Cloud Function → GCS (processed)
Pub/Sub → Cloud Function → SendGrid/Mailgun

Cost: $5-30/month (free tier covers a LOT)
```

### Mid-Size

```
Microservices with Cloud Functions:

Functions (15-30 functions):
├── user-api (HTTP, concurrency 80, min 1)
├── order-api (HTTP, concurrency 80, min 1)
├── order-processor (Pub/Sub, concurrency 10)
├── payment-webhook (HTTP, concurrency 50)
├── image-processor (GCS trigger)
├── notification-sender (Pub/Sub)
├── audit-logger (Eventarc — Cloud Audit Logs)
├── daily-reports (Cloud Scheduler)
└── data-pipeline-trigger (Eventarc — BigQuery)

Scaling:
├── API functions: min 1, max 100, concurrency 80
├── Workers: min 0, max 50, concurrency 10
├── All Gen 2 (concurrency is the key advantage)
└── VPC Connector for Cloud SQL/Redis access

Deployment:
├── CI/CD: Cloud Build → deploy function
├── Traffic splitting: 10% canary → monitor → 100%
├── Separate service accounts per function
└── Secrets from Secret Manager

Monitoring:
├── Cloud Logging (structured JSON logs)
├── Cloud Monitoring (invocation count, latency, errors)
├── Error Reporting (automatic error grouping)
├── Cloud Trace (distributed tracing)
└── Alerts: Error rate > 1%, latency P99 > 2s

Cost: $100-400/month
```

### Enterprise

```
Large-scale Cloud Functions platform:

Scale: 100+ functions, 100M+ invocations/month

Organization:
├── Shared per project/team (GCP project per team)
├── Platform team manages: VPC connectors, secrets, CI/CD
├── Each team deploys own functions
└── Terraform modules for standardized function creation

Deployment:
├── Cloud Build CI/CD (GitHub → Cloud Build → deploy)
├── Traffic splitting: 5% → 25% → 100% (automated)
├── Rollback: gcloud run services update-traffic → previous
├── Multi-region: Deploy to asia-south1 + us-central1
└── Global HTTP LB in front of Cloud Run (Gen 2) endpoints

Security:
├── Dedicated service account per function
├── VPC connector (or Direct VPC Egress) for private access
├── Ingress: Internal only (for internal functions)
├── Binary Authorization (ensure trusted builds only)
├── Secret Manager for all secrets (auto-rotation)
├── CMEK encryption for function source and data
└── VPC Service Controls (data exfiltration prevention)

Performance:
├── Min instances: Based on traffic patterns (scheduled)
├── Concurrency tuning: Profile each function
├── Connection pooling: Reuse DB connections across requests
├── Global scope initialization: DB clients, SDK clients
└── Monitoring: Latency P50/P95/P99 per function

Cost optimization:
├── Gen 2 concurrency: 10-50× fewer instances than Gen 1
├── Min instances only during business hours (scheduled)
├── Max instances capped per function (prevent runaway)
├── Committed use discounts (Cloud Run underlying)
├── Review unused functions monthly
└── Cost: $2,000-8,000/month at enterprise scale
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Serverless event-driven functions (FaaS) |
| Gen 2 | Built on Cloud Run, recommended for all new functions |
| Concurrency | Gen 1: 1/instance, Gen 2: up to 1000/instance |
| Runtimes | Node.js, Python, Go, Java, .NET, Ruby, PHP |
| Timeout | Gen 2: 60 min (HTTP), 9 min (event) |
| Memory | 128 MB - 32 GB |
| Max instances | 100 default, up to 3000 |
| Min instances | 0 (scale to zero) or N (warm) |
| HTTP trigger | Auto-generated HTTPS URL |
| Pub/Sub trigger | Message → function |
| Storage trigger | Object event → function |
| Eventarc | 100+ event sources (Audit Logs, BigQuery, etc.) |
| Secrets | Secret Manager integration (env var or volume) |
| VPC access | VPC Connector or Direct VPC Egress (Gen 2) |
| Traffic splitting | Canary/A-B testing (Gen 2 via Cloud Run) |
| Free tier | 2M invocations/month + 400K GB-seconds |
| AWS equivalent | Lambda |
| Azure equivalent | Azure Functions |

---

## Console Walkthrough: Managing & Deleting Functions

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING EXISTING FUNCTIONS FROM CONSOLE                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Cloud Functions → Click function name                    │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ EDIT ENVIRONMENT VARIABLES, MEMORY, TIMEOUT                          │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ 1. Click [Edit] at the top of the function details page           │
│ 2. You'll see the same creation wizard with current settings      │
│ 3. Navigate to "Runtime, build, connections and security":        │
│                                                                       │
│    Memory allocated: [512 MiB ▼] ← Change from 256 to 512       │
│    Timeout: [120] seconds         ← Increase from 60 to 120     │
│    Min instances: [1]             ← Add warm instance            │
│    Max instances: [200]           ← Increase scaling limit       │
│                                                                       │
│    Runtime environment variables:                                   │
│    ┌──────────────────┬──────────────────────────────────────┐    │
│    │ Name             │ Value                                  │    │
│    ├──────────────────┼──────────────────────────────────────┤    │
│    │ LOG_LEVEL        │ debug  ← Changed from "info"          │    │
│    │ NEW_VAR          │ some_value ← Added new variable       │    │
│    └──────────────────┴──────────────────────────────────────┘    │
│    [+ Add variable]                                                │
│                                                                       │
│ 4. Click [Next] to proceed to the Code step                      │
│ 5. Click [Deploy] to apply changes                                │
│ ⚡ Editing triggers a new deployment (takes 1-2 minutes).         │
│ ⚡ No downtime — new revision is created, traffic shifts after   │
│    health check passes.                                          │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ UPDATE CODE ON AN EXISTING FUNCTION                                   │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ 1. Click [Edit] on the function details page                      │
│ 2. Click [Next] to skip to the Code step                         │
│ 3. Choose source:                                                  │
│    ● Inline editor — edit main.py / index.js directly            │
│    ○ ZIP upload — upload a new .zip with updated code            │
│    ○ ZIP from Cloud Storage — point to a new .zip in GCS        │
│    ○ Cloud Source Repositories / GitHub                           │
│ 4. Edit code in the inline editor:                                │
│    ┌─── main.py ─────────────────────────────────────────────┐   │
│    │ import functions_framework                                │   │
│    │                                                            │   │
│    │ @functions_framework.http                                  │   │
│    │ def order_processor(request):                              │   │
│    │     # Updated logic here...                               │   │
│    │     return 'Updated response', 200                        │   │
│    └────────────────────────────────────────────────────────────┘   │
│ 5. Verify Entry point matches your function name                  │
│ 6. Click [Deploy]                                                  │
│ ⚡ For production: Use gcloud CLI or CI/CD instead of inline     │
│    editor. The inline editor is great for testing/prototyping.   │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ DELETE A FUNCTION                                                     │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ Method 1: From function list                                       │
│ 1. Console → Cloud Functions                                      │
│ 2. Check the box ☑ next to the function name                    │
│ 3. Click [Delete] at the top                                      │
│ 4. Confirm deletion                                                │
│                                                                       │
│ Method 2: From function details                                    │
│ 1. Click on the function name to open details                    │
│ 2. Click [Delete] button at the top                               │
│ 3. Confirm deletion                                                │
│                                                                       │
│ ⚠️ Deletion is permanent! The function, its revisions, and its   │
│    trigger are all removed. Source code in GCS is NOT deleted.   │
│ ⚡ The underlying Cloud Run service (Gen 2) is also deleted.      │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ VIEW FUNCTION LOGS                                                    │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ Method 1: From function details                                    │
│ 1. Console → Cloud Functions → Click function name               │
│ 2. Click the [Logs] tab                                           │
│ 3. Logs appear with:                                               │
│    ├── Timestamp                                                  │
│    ├── Severity (INFO, WARNING, ERROR, DEBUG)                   │
│    ├── Execution ID (links all logs from one invocation)        │
│    └── Message (your print/console.log output)                  │
│ 4. Use the filter bar to search by:                               │
│    ├── Severity level (e.g., show only ERRORs)                 │
│    ├── Time range (last 1 hour, 24 hours, custom)              │
│    └── Text search (search within log messages)                 │
│                                                                       │
│ Method 2: Cloud Logging (more powerful)                            │
│ 1. Console → Logging → Logs Explorer                              │
│ 2. Query:                                                          │
│    resource.type="cloud_run_revision"                             │
│    resource.labels.service_name="order-processor"                │
│    severity>=ERROR                                                │
│ ⚡ Cloud Logging gives you advanced filtering, log-based metrics, │
│    and alerting. The function's Logs tab is a quick shortcut.    │
│                                                                       │
│ Method 3: gcloud CLI                                               │
│    gcloud functions logs read order-processor \                   │
│      --gen2 --region=asia-south1 --limit=50                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Local Development & Testing

```
┌─────────────────────────────────────────────────────────────────────┐
│           LOCAL DEVELOPMENT WITH FUNCTIONS FRAMEWORK                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ The Functions Framework lets you run Cloud Functions locally on   │
│ your machine — no deployment needed! Great for development and   │
│ debugging before deploying to GCP.                                 │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ PYTHON                                                                │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ 1. Install the framework:                                          │
│    pip install functions-framework                                │
│                                                                       │
│ 2. Write your function (main.py):                                 │
│    import functions_framework                                     │
│                                                                       │
│    @functions_framework.http                                      │
│    def hello_http(request):                                       │
│        name = request.args.get('name', 'World')                  │
│        return f'Hello, {name}!'                                   │
│                                                                       │
│ 3. Run locally:                                                    │
│    functions-framework --target=hello_http --port=8080            │
│                                                                       │
│ 4. Test it:                                                        │
│    curl http://localhost:8080?name=Cloud                          │
│    → Hello, Cloud!                                                │
│                                                                       │
│ ⚡ The --target flag must match your function name exactly.       │
│ ⚡ Set environment variables locally:                               │
│    export DB_HOST=localhost                                       │
│    functions-framework --target=hello_http                       │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ NODE.JS                                                               │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ 1. Run directly with npx (no install needed):                     │
│    npx @google-cloud/functions-framework --target=helloHttp      │
│                                                                       │
│ 2. Or install as a dependency:                                     │
│    npm install @google-cloud/functions-framework                 │
│                                                                       │
│ 3. Write your function (index.js):                                │
│    const functions = require('@google-cloud/functions-framework');│
│                                                                       │
│    functions.http('helloHttp', (req, res) => {                   │
│      const name = req.query.name || 'World';                    │
│      res.send(`Hello, ${name}!`);                                │
│    });                                                             │
│                                                                       │
│ 4. Run locally:                                                    │
│    npx @google-cloud/functions-framework --target=helloHttp \   │
│      --port=8080                                                  │
│                                                                       │
│ 5. Test it:                                                        │
│    curl http://localhost:8080?name=Cloud                          │
│    → Hello, Cloud!                                                │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ UNIT TESTING                                                          │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ Python (pytest):                                                    │
│ ── test_main.py ──                                                 │
│                                                                       │
│ import pytest                                                      │
│ from unittest.mock import Mock                                    │
│ from main import hello_http                                       │
│                                                                       │
│ def test_hello_http_with_name():                                  │
│     # Create a mock request with query parameter                 │
│     req = Mock()                                                   │
│     req.args = {'name': 'Cloud'}                                  │
│     req.get_json.return_value = None                              │
│                                                                       │
│     result = hello_http(req)                                      │
│     assert result == 'Hello, Cloud!'                              │
│                                                                       │
│ def test_hello_http_without_name():                               │
│     req = Mock()                                                   │
│     req.args = {}                                                  │
│     req.get_json.return_value = None                              │
│                                                                       │
│     result = hello_http(req)                                      │
│     assert result == 'Hello, World!'                              │
│                                                                       │
│ Run tests:                                                         │
│   pip install pytest                                              │
│   pytest test_main.py -v                                          │
│                                                                       │
│ ⚡ Mock the request object — Cloud Functions receives a Flask     │
│    request (Python) or Express request (Node.js).                │
│ ⚡ Test locally first, then deploy. Saves time and money!         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Connecting Cloud Functions to Cloud SQL

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD FUNCTIONS → CLOUD SQL (UNIX SOCKET)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud Functions can connect to Cloud SQL using a Unix socket —   │
│ no VPC Connector required! GCP provides a built-in proxy.        │
│                                                                       │
│ How it works:                                                       │
│ ┌──────────────┐   Unix socket    ┌───────────────────────┐      │
│ │ Cloud        │ ───────────────► │ Cloud SQL Proxy       │      │
│ │ Function     │   (built-in)     │ (auto-managed by GCP) │      │
│ │              │                   │       ↓               │      │
│ │              │                   │ Cloud SQL instance    │      │
│ └──────────────┘                   └───────────────────────┘      │
│                                                                       │
│ ⚡ No VPC Connector needed for Cloud SQL! The proxy is automatic. │
│ ⚡ You DO need a VPC Connector for Redis, Memorystore, or VMs.    │
│                                                                       │
│ Setup steps:                                                        │
│ 1. Get your Cloud SQL instance connection name:                   │
│    Console → Cloud SQL → Instance → Overview                     │
│    Connection name: project-id:region:instance-name              │
│    Example: my-project:asia-south1:mydb                          │
│                                                                       │
│ 2. Grant Cloud SQL Client role to the function's service account:│
│    gcloud projects add-iam-policy-binding my-project \           │
│      --member="serviceAccount:sa-func@my-project.iam..." \      │
│      --role="roles/cloudsql.client"                              │
│                                                                       │
│ 3. Add the Cloud SQL connection in the function settings:        │
│    Console → Cloud Functions → Edit → Connections tab            │
│    Cloud SQL connections: [+ Add connection]                     │
│    Select: my-project:asia-south1:mydb                           │
│                                                                       │
│ 4. Python code example (using pg8000 + SQLAlchemy):              │
│                                                                       │
│ ── main.py ──                                                      │
│ import os                                                          │
│ import sqlalchemy                                                  │
│ import functions_framework                                         │
│                                                                       │
│ def connect_unix_socket():                                        │
│     db_user = os.environ["DB_USER"]                               │
│     db_pass = os.environ["DB_PASS"]                               │
│     db_name = os.environ["DB_NAME"]                               │
│     instance_connection = os.environ["INSTANCE_CONNECTION_NAME"] │
│     unix_socket_path = f"/cloudsql/{instance_connection}"        │
│                                                                       │
│     pool = sqlalchemy.create_engine(                              │
│         sqlalchemy.engine.url.URL.create(                        │
│             drivername="postgresql+pg8000",                       │
│             username=db_user,                                     │
│             password=db_pass,                                     │
│             database=db_name,                                     │
│             query={"unix_sock": f"{unix_socket_path}/.s.PGSQL.5432"}│
│         ),                                                        │
│         pool_size=5,                                              │
│         max_overflow=2,                                            │
│         pool_pre_ping=True,                                       │
│     )                                                              │
│     return pool                                                    │
│                                                                       │
│ # Initialize OUTSIDE the function (reused across invocations)   │
│ db = connect_unix_socket()                                        │
│                                                                       │
│ @functions_framework.http                                          │
│ def get_users(request):                                           │
│     with db.connect() as conn:                                    │
│         result = conn.execute(                                    │
│             sqlalchemy.text("SELECT name FROM users LIMIT 10")  │
│         )                                                         │
│         users = [row[0] for row in result]                       │
│     return {"users": users}                                       │
│                                                                       │
│ ── requirements.txt ──                                              │
│ functions-framework==3.*                                            │
│ SQLAlchemy==2.*                                                     │
│ pg8000==1.*                                                         │
│ cloud-sql-python-connector==1.*                                    │
│                                                                       │
│ ── Environment variables to set ──                                 │
│ DB_USER=myuser                                                      │
│ DB_PASS=*** (use Secret Manager!)                                  │
│ DB_NAME=mydb                                                        │
│ INSTANCE_CONNECTION_NAME=my-project:asia-south1:mydb              │
│                                                                       │
│ ⚡ Initialize the DB connection pool in GLOBAL SCOPE (outside     │
│    your function). This reuses the pool across invocations.     │
│ ⚡ Use Secret Manager for DB_PASS — never hardcode passwords!     │
│ ⚡ For MySQL: Use pymysql driver instead of pg8000.               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting Common Issues

```
┌─────────────────────────────────────────────────────────────────────┐
│           TROUBLESHOOTING COMMON ISSUES                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. COLD START DELAYS                                                │
│ ────────────────────                                               │
│ Symptom: First request after idle period takes 500ms-10s extra.  │
│                                                                       │
│ Why: GCP must download code, start container, and initialize     │
│      your global scope before handling the first request.        │
│                                                                       │
│ Fixes:                                                              │
│ ├── Set min instances to 1+ (keeps instances warm)              │
│ │   gcloud functions deploy my-func --min-instances=1           │
│ ├── Use Gen 2 with concurrency (fewer instances needed overall) │
│ ├── Keep deployment package small (remove unused dependencies)  │
│ ├── Use lighter runtimes (Python/Node.js start faster than Java)│
│ ├── Move heavy initialization to global scope (runs once, not  │
│ │   on every request)                                           │
│ └── Use Cloud Scheduler to ping the function every few minutes  │
│     (cheap keep-alive trick if min instances is too expensive)  │
│                                                                       │
│ 2. FUNCTION TIMEOUT                                                  │
│ ────────────────────                                               │
│ Symptom: Function returns error after timeout period.             │
│ Error: "Function execution took X ms, finished with status:     │
│         timeout"                                                   │
│                                                                       │
│ Why: Function execution exceeded the configured timeout.         │
│                                                                       │
│ Fixes:                                                              │
│ ├── Increase timeout value:                                      │
│ │   gcloud functions deploy my-func --timeout=300s              │
│ │   Max: 3600s (HTTP Gen 2), 540s (event-triggered)            │
│ ├── Optimize your code:                                          │
│ │   ├── Use connection pooling (don't create new DB connection │
│ │   │   per request)                                            │
│ │   ├── Process data in batches instead of one-by-one          │
│ │   └── Use async/await for I/O-bound operations               │
│ ├── Break large tasks into smaller functions chained via        │
│ │   Pub/Sub or Workflows                                        │
│ └── If processing large files: Stream instead of loading entire │
│     file into memory                                             │
│                                                                       │
│ 3. PERMISSION DENIED ERRORS                                         │
│ ────────────────────────────                                       │
│ Symptom: 403 Forbidden or "Permission denied" in logs.           │
│ Error: "Permission 'X' denied on resource 'Y'"                  │
│                                                                       │
│ Common causes & fixes:                                              │
│ ├── Invoking the function:                                       │
│ │   ├── Caller needs roles/cloudfunctions.invoker (Gen 1) or   │
│ │   │   roles/run.invoker (Gen 2)                              │
│ │   └── For public access: Set "Allow unauthenticated"         │
│ ├── Function accessing GCP services (Storage, BigQuery, etc.): │
│ │   ├── Check the function's SERVICE ACCOUNT                   │
│ │   ├── Grant the needed role to the service account:          │
│ │   │   gcloud projects add-iam-policy-binding my-project \   │
│ │   │     --member="serviceAccount:sa@project.iam..." \       │
│ │   │     --role="roles/storage.objectViewer"                  │
│ │   └── ⚠️ Don't use the default compute SA in production!    │
│ ├── Accessing secrets:                                           │
│ │   └── Function SA needs roles/secretmanager.secretAccessor  │
│ └── Cloud SQL access:                                            │
│     └── Function SA needs roles/cloudsql.client                │
│                                                                       │
│ ⚡ Always check: What service account is the function using?      │
│    Console → Cloud Functions → function → Details → Service acct│
│                                                                       │
│ 4. MEMORY EXCEEDED                                                   │
│ ──────────────────                                                 │
│ Symptom: Function crashes or is killed mid-execution.             │
│ Error: "Memory limit of X MB exceeded" or "Container killed     │
│         due to memory usage"                                      │
│                                                                       │
│ Why: Your function used more RAM than the configured limit.      │
│                                                                       │
│ Fixes:                                                              │
│ ├── Increase memory allocation:                                  │
│ │   gcloud functions deploy my-func --memory=1Gi                │
│ │   Options: 128Mi, 256Mi, 512Mi, 1Gi, 2Gi, 4Gi, 8Gi, 16Gi, │
│ │   32Gi                                                        │
│ ├── Optimize memory usage:                                       │
│ │   ├── Stream large files instead of loading entirely          │
│ │   ├── Process data in chunks/batches                          │
│ │   ├── Release references to large objects when done          │
│ │   └── Use generators (Python) for large data sets            │
│ ├── Watch for memory leaks:                                      │
│ │   ├── Global variables that grow across invocations          │
│ │   ├── Event listeners not cleaned up                          │
│ │   └── Caches without size limits                              │
│ └── Monitor memory usage:                                        │
│     Console → Cloud Functions → function → Metrics tab          │
│     Look at "Memory utilization" chart                          │
│     ⚡ If consistently above 80%, increase the limit.           │
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ QUICK DEBUGGING CHECKLIST                                            │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ ☐ Check logs first: Console → Cloud Functions → Logs tab         │
│ ☐ Verify the correct runtime and entry point                     │
│ ☐ Confirm environment variables are set correctly                │
│ ☐ Check the function's service account and its IAM roles         │
│ ☐ Verify VPC Connector if accessing private resources            │
│ ☐ Test locally with Functions Framework before deploying         │
│ ☐ Check Cloud Error Reporting for grouped errors                 │
│ ☐ Review memory and timeout metrics in the Metrics tab           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover GCP Cloud Run — the fully managed container runtime.

→ Next: [Chapter 17: Cloud Run](17-cloud-run.md)

---

*Last Updated: May 2026*
