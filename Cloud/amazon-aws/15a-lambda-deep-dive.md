# Chapter 15+: AWS Lambda Deep Dive

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Lambda Fundamentals](#part-1-lambda-fundamentals)
- [Part 2: Creating a Function (Full Console Walkthrough)](#part-2-creating-a-function-full-console-walkthrough)
- [Part 3: Triggers & Event Sources](#part-3-triggers--event-sources)
- [Part 4: Layers, Versions & Aliases](#part-4-layers-versions--aliases)
- [Part 5: Lambda@Edge & CloudFront Functions](#part-5-lambdaedge--cloudfront-functions)
- [Part 6: Container Image Support](#part-6-container-image-support)
- [Part 7: Destinations & Error Handling](#part-7-destinations--error-handling)
- [Part 8: Terraform](#part-8-terraform)
- [Part 9: AWS CLI Reference](#part-9-aws-cli-reference)
- [Part 10: Real-World Patterns](#part-10-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### How is This Different from Chapter 15?

Chapter 15 covers Lambda fundamentals — creating functions, triggers, and basic configuration. **This deep-dive chapter** covers production patterns you'll need when building real applications: edge compute (Lambda@Edge), container image packaging, canary deployments, concurrency management, and VPC networking.

> 💡 **Read Chapter 15 first.** Come back here when you need to take your Lambda functions to production.

AWS Lambda is the pioneer of serverless computing — run code without provisioning or managing servers. You upload your function code, define a trigger, and AWS handles everything else: scaling, patching, high availability, and billing per millisecond of execution. Lambda is the backbone of event-driven architectures on AWS.

```
What you'll learn:
├── Lambda Fundamentals
│   ├── What & why (serverless functions)
│   ├── Lambda vs EC2 vs ECS vs App Runner
│   └── Execution model & cold starts
├── Creating a Function (Full Console Walkthrough)
│   ├── Function configuration
│   ├── Runtime settings
│   ├── Memory & timeout
│   ├── Environment variables
│   └── Permissions (execution role)
├── Triggers & Event Sources
│   ├── API Gateway (REST/HTTP APIs)
│   ├── S3, DynamoDB, SQS, SNS, EventBridge
│   ├── Kinesis, CloudWatch Events/Logs
│   └── ALB, IoT, Cognito
├── Layers (shared code/dependencies)
├── Versions & Aliases (deployment)
├── Concurrency (reserved, provisioned)
├── Lambda@Edge & CloudFront Functions
├── Container Image Support
├── Lambda URLs (built-in HTTP endpoint)
├── VPC Access
├── Destinations & Dead Letter Queues
├── Terraform & CLI
└── Real-world patterns
```

---

## Part 1: Lambda Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           LAMBDA CONCEPT                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Lambda?                                                     │
│ ├── Run code in response to events (no servers!)                │
│ ├── Supported: Node.js, Python, Java, Go, .NET, Ruby, custom   │
│ ├── Pay only when code runs (per ms + per request)              │
│ ├── Auto-scales: 0 → thousands of concurrent executions        │
│ ├── Max execution: 15 minutes per invocation                    │
│ └── Stateless: Each invocation is independent                   │
│                                                                       │
│ Execution model:                                                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Event (S3 upload, API call, SQS message, schedule)          │  │
│ │    ↓                                                         │  │
│ │ Lambda Service:                                              │  │
│ │    ├── Is there a warm container? → Reuse it (warm start)  │  │
│ │    └── No? → Create new container (cold start ~100-500ms)  │  │
│ │    ↓                                                         │  │
│ │ Run handler function                                        │  │
│ │    ↓                                                         │  │
│ │ Return response → Container stays warm (~5-15 min)         │  │
│ │                                                              │  │
│ │ Cold start breakdown:                                       │  │
│ │ ├── Download code/image: 0-500ms                           │  │
│ │ ├── Start runtime (Node.js/Python/Java): 50-1000ms        │  │
│ │ ├── Init code (outside handler): varies                    │  │
│ │ └── Total: 100ms (Python) to 3s (Java with VPC)          │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Pricing:                                                             │
│ ├── Requests: $0.20 per 1M requests (first 1M free/month)     │
│ ├── Duration: $0.0000166667 per GB-second                      │
│ │   128 MB × 1 sec = $0.0000021                                │
│ │   1024 MB × 1 sec = $0.0000166667                            │
│ ├── Free tier: 1M requests + 400,000 GB-seconds/month         │
│ └── Provisioned concurrency: $0.0000041667 per GB-second (idle)│
│                                                                       │
│ Limits:                                                              │
│ ├── Memory: 128 MB – 10,240 MB (10 GB)                        │
│ ├── Timeout: 1 sec – 15 min                                    │
│ ├── Package size: 50 MB (zip) / 250 MB (unzipped) / 10 GB img │
│ ├── /tmp storage: 512 MB – 10,240 MB (10 GB)                  │
│ ├── Concurrent executions: 1,000 (default, can increase)      │
│ ├── Payload: 6 MB (sync) / 256 KB (async)                     │
│ └── Layers: 5 per function (total 250 MB unzipped)            │
│                                                                       │
│ Comparison:                                                          │
│ ┌────────────────┬──────────┬──────────┬──────────┬────────────┐ │
│ │ Feature         │ Lambda    │ EC2       │ ECS Farg │ App Runner │ │
│ ├────────────────┼──────────┼──────────┼──────────┼────────────┤ │
│ │ Servers         │ None ✅  │ You manage│ None     │ None       │ │
│ │ Scale to zero   │ Yes ✅   │ No       │ No       │ Yes        │ │
│ │ Max runtime     │ 15 min ⚠️│ Unlimited│ Unlimited│ Unlimited  │ │
│ │ Cold starts     │ Yes ⚠️   │ No       │ No       │ Minimal    │ │
│ │ Concurrency     │ 1000+    │ Unlimited│ Custom   │ Custom     │ │
│ │ Pricing         │ Per ms ✅│ Per hour │ Per sec  │ Per sec    │ │
│ │ WebSocket       │ Via APIGW│ Yes      │ Yes      │ Yes        │ │
│ │ Stateful        │ No ❌    │ Yes      │ Limited  │ No         │ │
│ │ Custom runtime  │ Yes      │ Yes      │ Yes      │ Yes        │ │
│ │ Best for        │ Events   │ General  │ Services │ Web apps   │ │
│ └────────────────┴──────────┴──────────┴──────────┴────────────┘ │
│                                                                       │
│ ⚡ GCP equivalent: Cloud Functions                                  │
│ ⚡ Azure equivalent: Azure Functions                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Function (Full Console Walkthrough)

```
Console → Lambda → Create function

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE FUNCTION                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Create method:                                                       │
│ ● Author from scratch ✅                                            │
│ ○ Use a blueprint (pre-built templates)                            │
│ ○ Container image (deploy from ECR)                                │
│ ○ Browse serverless app repository                                 │
│                                                                       │
│ ── Basic information ──                                            │
│                                                                       │
│ Function name: [process-order]                                      │
│ Runtime: [Node.js 20.x ▼]                                         │
│ ├── Node.js 20.x / 18.x                                          │
│ ├── Python 3.12 / 3.11 / 3.10                                   │
│ ├── Java 21 / 17 / 11                                            │
│ ├── .NET 8 / 6                                                    │
│ ├── Ruby 3.3                                                       │
│ ├── Go 1.x (via provided.al2023)                                 │
│ ├── Rust (via provided.al2023)                                   │
│ └── Custom runtime (provided.al2023 — bring your own)           │
│                                                                       │
│ Architecture:                                                        │
│ ● x86_64 (Intel/AMD — default, most packages support)           │
│ ○ arm64 (Graviton2 — 20% cheaper, better perf for many tasks!) │
│ ⚡ arm64: Same code works, but native deps need ARM builds.        │
│   Use arm64 for cost savings if packages support it.            │
│                                                                       │
│ ── Permissions ──                                                   │
│                                                                       │
│ Execution role:                                                      │
│ ● Create a new role with basic Lambda permissions               │
│ ○ Use an existing role                                            │
│ ○ Create a new role from AWS policy templates                   │
│                                                                       │
│ ⚡ Execution role = IAM role assumed by Lambda:                      │
│   Basic: CloudWatch Logs (write logs)                           │
│   Add: S3, DynamoDB, SQS, etc. as needed                       │
│                                                                       │
│ ── Advanced settings ──                                            │
│                                                                       │
│ ☐ Enable function URL                                               │
│ ⚡ Creates HTTPS endpoint: https://xxxxx.lambda-url.region.on.aws │
│   No API Gateway needed! Good for webhooks, simple APIs.        │
│   Auth: IAM or None (public)                                    │
│                                                                       │
│ ☐ Enable VPC                                                        │
│ ⚡ Place Lambda in VPC to access private resources (RDS, Redis).  │
│   Adds cold start time (~1-2 sec extra with VPC).               │
│   Requires NAT Gateway for internet access!                     │
│                                                                       │
│ [Create function]                                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           FUNCTION CONFIGURATION (after creation)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── General configuration ──                                        │
│                                                                       │
│ Memory: [512] MB  (128 – 10,240 MB)                               │
│ ⚡ More memory = more CPU (proportional).                            │
│   128 MB: Minimal CPU (simple transforms)                        │
│   512 MB: Good for most APIs                                     │
│   1024 MB: Heavy processing                                      │
│   1769 MB: 1 full vCPU equivalent                                │
│   10240 MB: 6 vCPUs (max compute)                                │
│                                                                       │
│ Ephemeral storage (/tmp): [512] MB  (512 – 10,240 MB)           │
│ ⚡ For temp files, downloads, processing. Cleared between invokes. │
│                                                                       │
│ Timeout: [30] seconds  (1 sec – 15 min)                          │
│ ⚡ Function killed at timeout! Set slightly above expected runtime. │
│   API responses: 30s. Processing: 5-15 min.                     │
│                                                                       │
│ ── Environment variables ──                                        │
│ [+ Add environment variable]                                       │
│ ┌────────────────────────┬──────────────────────────────────────┐ │
│ │ Key                     │ Value                                 │ │
│ ├────────────────────────┼──────────────────────────────────────┤ │
│ │ TABLE_NAME             │ orders-prod                           │ │
│ │ BUCKET_NAME            │ my-uploads-prod                       │ │
│ │ QUEUE_URL              │ https://sqs.../orders-queue          │ │
│ │ DB_HOST                │ mydb.xxx.rds.amazonaws.com           │ │
│ └────────────────────────┴──────────────────────────────────────┘ │
│                                                                       │
│ ☑ Enable encryption helpers (KMS encryption for env vars)        │
│ ⚡ Or use Secrets Manager/SSM for sensitive values.                 │
│                                                                       │
│ ── VPC ──                                                           │
│ VPC: [vpc-prod ▼]                                                  │
│ Subnets: [subnet-private-1a, subnet-private-1b ▼]               │
│ Security groups: [sg-lambda ▼]                                    │
│ ⚡ VPC Lambda: Can reach private resources (RDS, ElastiCache).    │
│   Cannot reach internet without NAT Gateway!                    │
│   Only add VPC if you need private resource access.             │
│                                                                       │
│ ── Concurrency ──                                                   │
│ Reserved concurrency: [100]  (or unreserved)                     │
│ ⚡ Reserve = guarantee this many concurrent executions for this fn. │
│   Also CAPS the function (won't exceed this number).            │
│   Unreserved: Uses shared pool (account limit: 1000 default).  │
│                                                                       │
│ Provisioned concurrency: [10]                                      │
│ ⚡ Keep 10 execution environments warm (no cold starts!).          │
│   Costs money even when idle. Use for latency-critical paths.  │
│   Apply to a VERSION or ALIAS (not $LATEST).                    │
│                                                                       │
│ ── Function URL ──                                                  │
│ Auth type:                                                           │
│ ○ AWS_IAM (require SigV4 authentication)                        │
│ ● NONE (public — anyone can call)                                │
│                                                                       │
│ ☐ Configure CORS                                                    │
│ Allowed origins: [https://app.example.com]                        │
│ Allowed methods: [GET, POST]                                       │
│ Allowed headers: [Content-Type, Authorization]                   │
│                                                                       │
│ URL: https://xxxxxxxx.lambda-url.ap-south-1.on.aws              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Triggers & Event Sources

```
┌─────────────────────────────────────────────────────────────────────┐
│           EVENT SOURCES                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Invocation models:                                                   │
│ ├── Synchronous: Caller waits for response (API Gateway, URL)   │
│ ├── Asynchronous: Caller gets 202, Lambda runs later (S3, SNS)  │
│ └── Poll-based: Lambda polls source (SQS, DynamoDB, Kinesis)    │
│                                                                       │
│ ┌────────────────────┬──────────────────────────────────────────┐ │
│ │ Trigger             │ Use case                                  │ │
│ ├────────────────────┼──────────────────────────────────────────┤ │
│ │ API Gateway (REST) │ REST APIs, WebSocket APIs                │ │
│ │ API Gateway (HTTP) │ Simple HTTP APIs (cheaper, faster)       │ │
│ │ Lambda URL          │ Webhooks, simple endpoints (no APIGW)  │ │
│ │ ALB                 │ HTTP behind load balancer               │ │
│ │ S3                  │ File uploads, processing                │ │
│ │ DynamoDB Streams   │ React to DB changes (CDC)               │ │
│ │ SQS                 │ Queue processing, decoupling           │ │
│ │ SNS                 │ Fan-out notifications                   │ │
│ │ EventBridge        │ Scheduled, cross-service events         │ │
│ │ Kinesis            │ Real-time streaming                      │ │
│ │ CloudWatch Logs    │ Log processing, alerts                  │ │
│ │ CloudWatch Events  │ Cron schedules (legacy, use EventBridge)│ │
│ │ Cognito            │ User auth hooks (pre-signup, etc.)      │ │
│ │ IoT                │ IoT device events                        │ │
│ │ CloudFront         │ Edge processing (Lambda@Edge)           │ │
│ │ Step Functions     │ Orchestration workflows                  │ │
│ └────────────────────┴──────────────────────────────────────────┘ │
│                                                                       │
│ API Gateway (most common trigger):                                  │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Client → API Gateway → Lambda → Response                   │  │
│ │                                                              │  │
│ │ REST API: Full featured (auth, throttling, caching, WAF)   │  │
│ │ HTTP API: Simpler, cheaper, faster (70% less cost) ✅       │  │
│ │                                                              │  │
│ │ // Lambda handler (Node.js)                                 │  │
│ │ export const handler = async (event) => {                   │  │
│ │   const body = JSON.parse(event.body);                     │  │
│ │   // process...                                              │  │
│ │   return {                                                   │  │
│ │     statusCode: 200,                                        │  │
│ │     headers: { "Content-Type": "application/json" },       │  │
│ │     body: JSON.stringify({ message: "success" })           │  │
│ │   };                                                         │  │
│ │ };                                                           │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ S3 trigger:                                                          │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ File uploaded to S3 → Lambda invoked (async)               │  │
│ │                                                              │  │
│ │ Event types: s3:ObjectCreated:*, s3:ObjectRemoved:*         │  │
│ │ Filter: prefix=uploads/ suffix=.jpg                        │  │
│ │                                                              │  │
│ │ // event.Records[0].s3.bucket.name                         │  │
│ │ // event.Records[0].s3.object.key                          │  │
│ │ Use for: Image resize, video transcode, CSV import         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ SQS trigger:                                                         │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Messages in SQS → Lambda polls → Processes batch           │  │
│ │                                                              │  │
│ │ Batch size: 1-10,000 messages                              │  │
│ │ Batch window: 0-300 seconds (wait to fill batch)           │  │
│ │                                                              │  │
│ │ // event.Records → array of messages                       │  │
│ │ // Successful: Message auto-deleted from queue             │  │
│ │ // Failed: Message returns to queue (retry or DLQ)         │  │
│ │                                                              │  │
│ │ Partial batch failure: Report which messages failed →      │  │
│ │ Only failed ones return to queue.                           │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ EventBridge (schedule):                                              │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Cron/rate schedule → Lambda                                 │  │
│ │                                                              │  │
│ │ rate(5 minutes)  ← Every 5 minutes                         │  │
│ │ rate(1 hour)     ← Every hour                              │  │
│ │ cron(0 2 * * ? *) ← Daily at 2 AM UTC                    │  │
│ │ cron(0 9 ? * MON-FRI *) ← Weekdays at 9 AM              │  │
│ │                                                              │  │
│ │ Use for: Cleanup, reports, health checks, sync             │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ DynamoDB Streams:                                                    │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ DynamoDB item changes → Stream → Lambda                    │  │
│ │                                                              │  │
│ │ event.Records[].eventName: INSERT | MODIFY | REMOVE         │  │
│ │ event.Records[].dynamodb.NewImage / OldImage               │  │
│ │                                                              │  │
│ │ Use for: Audit trail, search index sync, notifications    │  │
│ │ ⚡ Process in order (per partition key)                      │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Layers, Versions & Aliases

```
┌─────────────────────────────────────────────────────────────────────┐
│           LAYERS                                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Shared code/dependencies packaged separately from function.  │
│ Why: Reduce deployment size, share across functions, separate deps.│
│                                                                       │
│ Layer structure:                                                      │
│ ├── nodejs/node_modules/...   (Node.js dependencies)             │
│ ├── python/lib/python3.12/... (Python packages)                  │
│ ├── java/lib/...              (Java JARs)                        │
│ └── bin/, lib/                (Binaries, shared libraries)       │
│                                                                       │
│ Use cases:                                                           │
│ ├── Heavy dependencies (numpy, pandas, sharp, ffmpeg)           │
│ ├── Shared utility code across multiple functions                │
│ ├── Custom runtimes                                               │
│ └── AWS SDK version pinning                                       │
│                                                                       │
│ # Create layer                                                      │
│ aws lambda publish-layer-version \                                 │
│   --layer-name my-deps \                                           │
│   --compatible-runtimes nodejs20.x \                               │
│   --zip-file fileb://layer.zip                                    │
│                                                                       │
│ Attach: Function → Layers → Add layer → specify ARN + version   │
│ Limit: 5 layers per function, total 250 MB unzipped             │
│                                                                       │
├─────────────────────────────────────────────────────────────────────┤
│           VERSIONS & ALIASES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ $LATEST: Mutable, always points to current code (dev)            │
│                                                                       │
│ Version: Immutable snapshot of function code + config             │
│ ├── Publish: Creates version 1, 2, 3, ...                       │
│ ├── Each version has unique ARN: ...function:process-order:3    │
│ └── Cannot change once published                                 │
│                                                                       │
│ Alias: Named pointer to a version (like a tag)                   │
│ ├── prod → version 5                                             │
│ ├── staging → version 6                                          │
│ ├── ARN: ...function:process-order:prod                         │
│ └── Can shift traffic between versions (canary!)                │
│                                                                       │
│ Traffic shifting (canary deployment):                              │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Alias "prod":                                                │  │
│ │   Version 5: 90% traffic (stable)                           │  │
│ │   Version 6: 10% traffic (canary)                           │  │
│ │                                                              │  │
│ │ aws lambda update-alias --function-name process-order \     │  │
│ │   --name prod --function-version 5 \                        │  │
│ │   --routing-config '{"AdditionalVersionWeights":{"6":0.1}}' │  │
│ │                                                              │  │
│ │ Monitor errors → if OK → shift 100% to version 6           │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ Provisioned concurrency: Apply to VERSION or ALIAS (not $LATEST)│
│   This is why you need versions/aliases for production!         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Lambda@Edge & CloudFront Functions

```
┌─────────────────────────────────────────────────────────────────────┐
│           EDGE COMPUTE                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Lambda@Edge:                                                        │
│ ├── Run Lambda at CloudFront edge locations (200+ worldwide)    │
│ ├── Modify requests/responses at the CDN level                  │
│ ├── 4 trigger points:                                            │
│ │   ├── Viewer request: Before CloudFront checks cache         │
│ │   ├── Origin request: Before CloudFront goes to origin       │
│ │   ├── Origin response: After origin responds                 │
│ │   └── Viewer response: Before CloudFront responds to user   │
│ ├── Max: 5 sec (viewer) / 30 sec (origin)                     │
│ ├── Memory: 128 MB (viewer) / 10 GB (origin)                  │
│ └── Must be in us-east-1!                                       │
│                                                                       │
│ Use cases: A/B testing, auth at edge, URL rewrites, geo-redirect│
│                                                                       │
│ CloudFront Functions (lighter, cheaper):                           │
│ ├── Run at edge (even more locations than Lambda@Edge)          │
│ ├── JavaScript only, no network access                          │
│ ├── Max: 1 ms execution, 2 MB code                             │
│ ├── Viewer request/response only                                │
│ ├── 1/6th the cost of Lambda@Edge                               │
│ └── Use for: URL rewrite, header manipulation, simple redirect │
│                                                                       │
│ ┌──────────────────┬──────────────────┬──────────────────────┐    │
│ │ Feature           │ Lambda@Edge       │ CloudFront Functions │    │
│ ├──────────────────┼──────────────────┼──────────────────────┤    │
│ │ Runtime           │ Node.js, Python  │ JavaScript only       │    │
│ │ Execution time    │ 5-30 sec         │ < 1 ms                │    │
│ │ Memory            │ 128 MB - 10 GB   │ 2 MB                  │    │
│ │ Network access    │ Yes              │ No                    │    │
│ │ Body access       │ Yes              │ No                    │    │
│ │ Trigger points    │ All 4            │ Viewer only           │    │
│ │ Price             │ Higher           │ 1/6th cost            │    │
│ │ Use case          │ Complex logic    │ Simple transforms     │    │
│ └──────────────────┴──────────────────┴──────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Container Image Support

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONTAINER IMAGES                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Deploy Lambda from Docker container image (up to 10 GB!):          │
│                                                                       │
│ # Dockerfile (using AWS base image)                                │
│ FROM public.ecr.aws/lambda/nodejs:20                               │
│ COPY package*.json ./                                               │
│ RUN npm ci --production                                             │
│ COPY index.mjs ./                                                    │
│ CMD ["index.handler"]                                                │
│                                                                       │
│ # Build and push to ECR                                             │
│ docker build -t lambda-app .                                        │
│ docker tag lambda-app:latest 123456789.dkr.ecr.ap-south-1...     │
│ docker push 123456789.dkr.ecr.ap-south-1.../lambda-app:latest    │
│                                                                       │
│ # Create function from image                                       │
│ aws lambda create-function \                                        │
│   --function-name process-order \                                  │
│   --package-type Image \                                            │
│   --code ImageUri=123456789.dkr.ecr.../lambda-app:latest \       │
│   --role arn:aws:iam::123456789:role/lambda-role                  │
│                                                                       │
│ Why use container images?                                           │
│ ├── Larger deployments (10 GB vs 250 MB zip)                    │
│ ├── Custom runtimes (any language)                               │
│ ├── Existing Docker workflows / CI pipelines                    │
│ ├── Local testing with Docker                                    │
│ └── System dependencies (ffmpeg, ImageMagick, ML models)        │
│                                                                       │
│ ⚡ AWS base images: Includes Lambda Runtime Interface Client.       │
│   Custom image: Must implement Lambda Runtime API.               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Destinations & Error Handling

```
┌─────────────────────────────────────────────────────────────────────┐
│           ERROR HANDLING & DESTINATIONS                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Async invocation error handling:                                    │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ S3 upload → Lambda (async)                                  │  │
│ │                                                              │  │
│ │ Success: → On-success destination (SQS, SNS, Lambda, EB)  │  │
│ │ Failure: → Retry (2 attempts) → DLQ or On-failure dest    │  │
│ │                                                              │  │
│ │ Retry behavior (async):                                     │  │
│ │ ├── Attempt 1: Immediate                                   │  │
│ │ ├── Attempt 2: 1 minute delay                              │  │
│ │ ├── Attempt 3: 2 minutes delay (default max = 2 retries)  │  │
│ │ └── If still fails → Dead Letter Queue or Destination     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Dead Letter Queue (DLQ):                                            │
│ ├── SQS queue or SNS topic for failed invocations              │
│ ├── Only for async invocations                                  │
│ ├── Set on function config → DLQ = SQS ARN                    │
│ ├── Failed messages land in DLQ → inspect and replay          │
│ └── ⚡ Destinations are newer and more flexible than DLQ!        │
│                                                                       │
│ Destinations (recommended over DLQ):                               │
│ ├── On success: Route result to SQS/SNS/Lambda/EventBridge    │
│ ├── On failure: Route error to SQS/SNS/Lambda/EventBridge     │
│ ├── Includes full context (request, response, error details)   │
│ └── Better for building event-driven pipelines                 │
│                                                                       │
│ SQS trigger error handling:                                         │
│ ├── Lambda polls SQS → processes messages                      │
│ ├── Success: Message deleted from queue                         │
│ ├── Failure: Message returns to queue (visibility timeout)     │
│ ├── After maxReceiveCount failures → message goes to DLQ      │
│ ├── Partial batch response: Report individual failures         │
│ └── Configure on SQS (redrive policy), not on Lambda           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform

```hcl
# Lambda Function (zip deployment)
resource "aws_lambda_function" "process_order" {
  function_name = "process-order"
  role          = aws_iam_role.lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs20.x"
  architectures = ["arm64"]

  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  memory_size = 512
  timeout     = 30
  
  ephemeral_storage {
    size = 1024  # MB
  }

  environment {
    variables = {
      TABLE_NAME  = aws_dynamodb_table.orders.name
      BUCKET_NAME = aws_s3_bucket.uploads.id
      QUEUE_URL   = aws_sqs_queue.notifications.url
    }
  }

  vpc_config {
    subnet_ids         = [aws_subnet.private_1a.id, aws_subnet.private_1b.id]
    security_group_ids = [aws_security_group.lambda.id]
  }

  dead_letter_config {
    target_arn = aws_sqs_queue.lambda_dlq.arn
  }

  tracing_config {
    mode = "Active"  # X-Ray tracing
  }

  layers = [aws_lambda_layer_version.deps.arn]

  tags = {
    Environment = "prod"
  }
}

# Lambda Function URL
resource "aws_lambda_function_url" "webhook" {
  function_name      = aws_lambda_function.process_order.function_name
  authorization_type = "NONE"

  cors {
    allow_origins = ["https://app.example.com"]
    allow_methods = ["POST"]
    allow_headers = ["Content-Type"]
  }
}

# Lambda Layer
resource "aws_lambda_layer_version" "deps" {
  layer_name          = "shared-deps"
  filename            = "layer.zip"
  compatible_runtimes = ["nodejs20.x"]
  compatible_architectures = ["arm64"]
}

# Version & Alias
resource "aws_lambda_alias" "prod" {
  name             = "prod"
  function_name    = aws_lambda_function.process_order.function_name
  function_version = aws_lambda_function.process_order.version

  routing_config {
    additional_version_weights = {
      # Canary: 10% to new version
      # (aws_lambda_function.process_order_v2.version) = 0.1
    }
  }
}

# Provisioned Concurrency (on alias)
resource "aws_lambda_provisioned_concurrency_config" "prod" {
  function_name                  = aws_lambda_function.process_order.function_name
  qualifier                      = aws_lambda_alias.prod.name
  provisioned_concurrent_executions = 10
}

# API Gateway HTTP API trigger
resource "aws_apigatewayv2_api" "http" {
  name          = "orders-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda" {
  api_id                 = aws_apigatewayv2_api.http.id
  integration_type       = "AWS_PROXY"
  integration_uri        = aws_lambda_function.process_order.invoke_arn
  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "post_orders" {
  api_id    = aws_apigatewayv2_api.http.id
  route_key = "POST /orders"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}

resource "aws_apigatewayv2_stage" "prod" {
  api_id      = aws_apigatewayv2_api.http.id
  name        = "$default"
  auto_deploy = true
}

resource "aws_lambda_permission" "apigw" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.process_order.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.http.execution_arn}/*/*"
}

# S3 trigger
resource "aws_lambda_permission" "s3" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.process_order.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.uploads.arn
}

resource "aws_s3_bucket_notification" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  lambda_function {
    lambda_function_arn = aws_lambda_function.process_order.arn
    events             = ["s3:ObjectCreated:*"]
    filter_prefix      = "uploads/"
    filter_suffix      = ".jpg"
  }
}

# SQS trigger (event source mapping)
resource "aws_lambda_event_source_mapping" "sqs" {
  event_source_arn                   = aws_sqs_queue.orders.arn
  function_name                      = aws_lambda_function.process_order.arn
  batch_size                         = 10
  maximum_batching_window_in_seconds = 5
  
  function_response_types = ["ReportBatchItemFailures"]
}

# EventBridge schedule trigger
resource "aws_scheduler_schedule" "daily_cleanup" {
  name = "daily-cleanup"

  flexible_time_window {
    mode = "OFF"
  }

  schedule_expression = "cron(0 2 * * ? *)"

  target {
    arn      = aws_lambda_function.process_order.arn
    role_arn = aws_iam_role.scheduler.arn
  }
}

# IAM Role
resource "aws_iam_role" "lambda" {
  name = "lambda-process-order"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "lambda" {
  name = "lambda-permissions"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:PutItem",
          "dynamodb:GetItem",
          "dynamodb:Query",
        ]
        Resource = aws_dynamodb_table.orders.arn
      },
      {
        Effect = "Allow"
        Action = ["s3:GetObject", "s3:PutObject"]
        Resource = "${aws_s3_bucket.uploads.arn}/*"
      },
      {
        Effect = "Allow"
        Action = ["sqs:SendMessage"]
        Resource = aws_sqs_queue.notifications.arn
      },
      {
        Effect = "Allow"
        Action = ["sqs:ReceiveMessage", "sqs:DeleteMessage", "sqs:GetQueueAttributes"]
        Resource = aws_sqs_queue.orders.arn
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_vpc" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

resource "aws_iam_role_policy_attachment" "lambda_logs" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

---

## Part 9: AWS CLI Reference

```bash
# ═══ Function management ═══

# Create function (zip)
aws lambda create-function \
  --function-name process-order \
  --runtime nodejs20.x \
  --architectures arm64 \
  --handler index.handler \
  --role arn:aws:iam::123456789:role/lambda-role \
  --zip-file fileb://function.zip \
  --memory-size 512 \
  --timeout 30 \
  --environment "Variables={TABLE_NAME=orders,BUCKET=uploads}"

# Update function code
aws lambda update-function-code \
  --function-name process-order \
  --zip-file fileb://function.zip

# Update function config
aws lambda update-function-configuration \
  --function-name process-order \
  --memory-size 1024 \
  --timeout 60

# Invoke function (test)
aws lambda invoke \
  --function-name process-order \
  --payload '{"key": "value"}' \
  --cli-binary-format raw-in-base64-out \
  response.json

# View logs (last 5 min)
aws logs tail /aws/lambda/process-order --since 5m --follow

# ═══ Versions & Aliases ═══

# Publish version
aws lambda publish-version \
  --function-name process-order

# Create alias
aws lambda create-alias \
  --function-name process-order \
  --name prod \
  --function-version 3

# Update alias (canary — 10% to version 4)
aws lambda update-alias \
  --function-name process-order \
  --name prod \
  --function-version 3 \
  --routing-config '{"AdditionalVersionWeights": {"4": 0.1}}'

# ═══ Concurrency ═══

# Set reserved concurrency
aws lambda put-function-concurrency \
  --function-name process-order \
  --reserved-concurrent-executions 100

# Set provisioned concurrency (on alias)
aws lambda put-provisioned-concurrency-config \
  --function-name process-order \
  --qualifier prod \
  --provisioned-concurrent-executions 10

# ═══ Layers ═══

# Publish layer
aws lambda publish-layer-version \
  --layer-name shared-deps \
  --zip-file fileb://layer.zip \
  --compatible-runtimes nodejs20.x \
  --compatible-architectures arm64

# Add layer to function
aws lambda update-function-configuration \
  --function-name process-order \
  --layers arn:aws:lambda:ap-south-1:123456789:layer:shared-deps:1

# ═══ Event source mappings ═══

# Add SQS trigger
aws lambda create-event-source-mapping \
  --function-name process-order \
  --event-source-arn arn:aws:sqs:ap-south-1:123456789:orders \
  --batch-size 10

# Add DynamoDB Streams trigger
aws lambda create-event-source-mapping \
  --function-name process-order \
  --event-source-arn arn:aws:dynamodb:...:table/orders/stream/... \
  --starting-position LATEST \
  --batch-size 100

# ═══ Function URL ═══

# Create URL
aws lambda create-function-url-config \
  --function-name process-order \
  --auth-type NONE

# Delete function
aws lambda delete-function --function-name process-order
```

---

## Part 10: Real-World Patterns

### Startup

```
Event-driven backend on Lambda:

API Layer:
├── API Gateway (HTTP API) → Lambda functions
│   ├── POST /orders → create-order (512 MB, 30s)
│   ├── GET /orders → list-orders (256 MB, 10s)
│   ├── POST /uploads → presign-url (128 MB, 5s)
│   └── POST /webhooks/stripe → stripe-webhook (256 MB, 30s)
├── Custom domain: api.startup.com
├── Auth: Cognito authorizer (JWT)
└── CORS: Configured for app.startup.com

Event Processing:
├── S3 upload → resize-image (1024 MB, 60s, arm64)
├── DynamoDB Stream (orders) → send-notification (256 MB)
├── SQS (emails) → send-email (256 MB, 30s)
└── EventBridge schedule → daily-report (512 MB, 5 min)

Each function:
├── Separate IAM role (least privilege)
├── Environment vars for config, Secrets Manager for secrets
├── CloudWatch Logs (auto)
├── X-Ray tracing enabled
└── DLQ for async functions

Cost: ~$5-30/month (free tier covers most!)
```

### Mid-Size

```
Lambda microservices:

Services (20+ functions):
├── Orders: create, get, list, update, cancel
├── Users: register, login, profile, delete
├── Payments: charge, refund, webhook
├── Notifications: email, push, sms
├── Files: upload, process, thumbnail
└── Reports: daily, weekly, export

Architecture:
├── API Gateway (HTTP API): All REST endpoints
├── Step Functions: Complex workflows (order processing)
│   ├── Validate order → Charge payment → Send confirmation
│   ├── If payment fails → Retry → Alert → Cancel
│   └── Parallel: Send email + Update inventory + Notify
├── EventBridge: Cross-service events
│   ├── order.created → notification service
│   ├── payment.failed → alert service
│   └── user.registered → onboarding flow
├── SQS: Decouple heavy processing
│   ├── Email queue → batch send (avoid throttling)
│   └── Processing queue → file conversions
└── DynamoDB Streams: CDC to search/analytics

Performance:
├── Provisioned concurrency: 10 (critical API functions)
├── arm64: All functions (20% cost savings)
├── Layers: Shared deps (AWS SDK, utilities)
├── VPC: Only functions needing RDS/Redis
└── Function URLs: Internal webhooks (no API Gateway cost)

Deployment:
├── SAM (Serverless Application Model) or CDK
├── Per-function CI/CD (only deploy changed functions)
├── Aliases: prod, staging per function
├── Canary: 10% traffic → new version → promote
└── Rollback: Shift alias back to old version

Cost: $100-500/month
```

### Enterprise

```
Lambda at enterprise scale:

Platform:
├── 200+ Lambda functions across 10 services
├── Multi-region: ap-south-1 (primary), eu-west-1 (DR)
├── API Gateway: REST API (WAF, usage plans, API keys)
├── Step Functions: 15+ workflows (orders, claims, ETL)
├── EventBridge: 50+ event rules (cross-account, cross-region)
└── Kinesis: Real-time streaming → Lambda

Performance & Reliability:
├── Provisioned concurrency: 50+ (critical paths)
├── Reserved concurrency: Per-function limits
├── Account concurrency: Increased to 10,000
├── Multi-AZ: Automatic (Lambda manages)
├── DLQ + Destinations: All async functions
├── Partial batch response: All stream processors
└── Powertools for Lambda (logging, tracing, metrics)

Security:
├── VPC: All functions accessing internal resources
├── No public internet (NAT Gateway for outbound)
├── IAM: Per-function roles, least privilege
├── Secrets Manager: All credentials (auto-rotation)
├── KMS: Encrypt env vars at rest
├── Resource-based policies: Restrict invoke sources
├── Code signing: Only signed packages can deploy
└── Inspector: Vulnerability scanning on functions

Observability:
├── X-Ray: End-to-end distributed tracing
├── CloudWatch Embedded Metrics: Business metrics in logs
├── CloudWatch Insights: Query across all function logs
├── ServiceLens: Service map + traces + metrics
├── Alarms: Per-function error rate, throttles, duration
├── Dashboards: Per-team, per-service dashboards
└── Cost allocation: Tags per team/service

Governance:
├── AWS Organizations: Separate accounts (dev/staging/prod)
├── Service Control Policies: Restrict runtimes, regions
├── AWS Config: Enforce Lambda settings compliance
├── Deployment: CDK + CodePipeline per account
├── Feature flags: AppConfig integration
└── Approval gates: Manual approval for prod deploys

Cost: $2,000-15,000/month (high-volume event processing)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Serverless event-driven compute |
| Languages | Node.js, Python, Java, .NET, Go, Ruby, Custom |
| Max timeout | 15 minutes |
| Max memory | 10 GB (= 6 vCPU) |
| Max package | 50 MB zip / 250 MB unzipped / 10 GB container |
| Pricing | $0.20/1M requests + GB-second duration |
| Free tier | 1M requests + 400K GB-sec/month |
| Scale to zero | Yes (default behavior) |
| Concurrency | 1,000 default (can increase to 10K+) |
| Cold starts | 100ms (Python) to 3s (Java+VPC) |
| Provisioned | Keep warm instances (no cold start, costs $) |
| Layers | Up to 5, shared dependencies |
| Versions | Immutable snapshots of code |
| Aliases | Named pointers with traffic shifting |
| Function URL | Built-in HTTPS endpoint (no API Gateway) |
| VPC | Optional (adds cold start, needs NAT for internet) |
| Edge | Lambda@Edge + CloudFront Functions |
| GCP equivalent | Cloud Functions |
| Azure equivalent | Azure Functions |

---

## What's Next?

In the next chapter, we'll explore AWS Step Functions — orchestrating complex workflows with state machines.

→ Next: [Chapter 21: Step Functions](21-step-functions.md)

---

*Last Updated: May 2026*
