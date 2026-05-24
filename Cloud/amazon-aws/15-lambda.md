# Chapter 15: AWS Lambda - Serverless Compute

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Lambda Fundamentals](#part-1-lambda-fundamentals)
- [Part 2: Creating a Lambda Function (Full Portal Walkthrough)](#part-2-creating-a-lambda-function-full-portal-walkthrough)
- [Part 3: Function Configuration (After Creation)](#part-3-function-configuration-after-creation)
- [Part 4: Triggers & Event Sources](#part-4-triggers--event-sources)
- [Part 5: Lambda Layers](#part-5-lambda-layers)
- [Part 6: Versions & Aliases](#part-6-versions--aliases)
- [Part 7: Concurrency](#part-7-concurrency)
- [Part 8: Destinations](#part-8-destinations)
- [Part 9: Lambda@Edge & CloudFront Functions](#part-9-lambdaedge--cloudfront-functions)
- [Part 10: Container Image Support](#part-10-container-image-support)
- [Part 11: Terraform & CLI Examples](#part-11-terraform--cli-examples)
- [Part 12: Real-World Patterns](#part-12-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Serverless? Why Lambda?

> **Real-World Analogy:** Lambda is like a vending machine for code. Instead of running a 24/7 cafeteria (EC2 server), you put your logic in a vending machine. Someone presses a button (trigger event), the machine dispenses the result, and you're only charged per use. The machine takes no space when nobody's using it.

**Why does this matter?** Lambda is the most cost-effective way to run code that doesn't need to run 24/7. An API that gets 10,000 requests/day might cost $0.20/month on Lambda vs $15/month for the smallest EC2 instance. Lambda also eliminates server management — no patching, no OS updates, no capacity planning.

**When to use Lambda vs EC2:**
- Short tasks (< 15 min), event-driven → **Lambda**
- Long-running processes, custom OS, GPU → **EC2**
- Web APIs with steady traffic → Could be either (Lambda for simplicity, EC2 for cost at scale)

AWS Lambda is a serverless compute service that runs your code in response to events without provisioning or managing servers. You pay only for the compute time consumed — no charge when code is not running. Lambda automatically scales from zero to thousands of concurrent executions.

```
What you'll learn:
├── Lambda Fundamentals
│   ├── Serverless concept (no servers to manage)
│   ├── Event-driven architecture
│   ├── Execution model (cold start vs warm)
│   └── Pricing model (per-request + per-duration)
├── Creating a Function (Full Portal Walkthrough)
│   ├── Author from scratch
│   ├── Use a blueprint
│   ├── Container image
│   └── Browse serverless app repository
├── Function Configuration
│   ├── Runtime (Node.js, Python, Java, .NET, Go, Ruby, custom)
│   ├── Handler
│   ├── Memory (128 MB - 10,240 MB)
│   ├── Timeout (1s - 15 min)
│   ├── Ephemeral storage (/tmp: 512 MB - 10,240 MB)
│   ├── Environment variables
│   ├── Execution role (IAM)
│   └── VPC configuration
├── Triggers & Event Sources
│   ├── API Gateway (REST/HTTP APIs)
│   ├── S3 (object events)
│   ├── DynamoDB Streams
│   ├── SQS / SNS / EventBridge
│   ├── CloudWatch Events/Scheduled
│   ├── Kinesis / Kafka
│   ├── ALB (Application Load Balancer)
│   └── Many more (100+ event sources)
├── Layers (shared code/libraries)
├── Versions & Aliases
├── Concurrency (reserved, provisioned)
├── Destinations (async success/failure routing)
├── Lambda@Edge & CloudFront Functions
├── Container Image Support
├── Terraform & CLI examples
└── Real-world patterns
```

---

## Part 1: Lambda Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           SERVERLESS EXECUTION MODEL                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Traditional (EC2):                                                   │
│ ├── You provision server → runs 24/7 → you pay 24/7             │
│ ├── You manage OS, patching, scaling, availability              │
│ └── Server idle at 2 AM? Still paying.                           │
│                                                                       │
│ Serverless (Lambda):                                                 │
│ ├── You upload code → AWS runs it when triggered                │
│ ├── No servers to manage (AWS handles everything)               │
│ ├── Scales automatically (0 → thousands of instances)           │
│ ├── Pay only for execution time (per ms)                        │
│ └── Idle at 2 AM? $0 cost.                                      │
│                                                                       │
│ Execution flow:                                                      │
│                                                                       │
│ ┌───────────┐    Event    ┌───────────────┐    Invoke    ┌───────┐│
│ │ Trigger   │ ──────────► │ Lambda Service│ ──────────► │ Your  ││
│ │ (S3, API, │             │ (manages      │             │ Code  ││
│ │  SQS...)  │             │  containers)  │             │       ││
│ └───────────┘             └───────────────┘             └───────┘│
│                                                                       │
│ Cold start vs Warm start:                                           │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ COLD START (first invocation or after idle):                 │   │
│ │ ┌──────────┐ ┌──────────────┐ ┌────────────┐ ┌──────────┐ │   │
│ │ │ Download │→│ Create       │→│ Initialize │→│ Run      │ │   │
│ │ │ code     │ │ container    │ │ runtime +  │ │ handler  │ │   │
│ │ │          │ │ (microVM)    │ │ your init  │ │ function │ │   │
│ │ └──────────┘ └──────────────┘ └────────────┘ └──────────┘ │   │
│ │ ↑_____________100-500ms+ overhead__________↑               │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ WARM START (container reused):                               │   │
│ │                              ┌──────────┐                    │   │
│ │                              │ Run      │                    │   │
│ │ Container already exists → │ handler  │  (fast! <10ms)    │   │
│ │                              │ function │                    │   │
│ │                              └──────────┘                    │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Cold start tips:                                                     │
│ ├── Keep deployment package small                                │
│ ├── Use provisioned concurrency for latency-sensitive functions │
│ ├── Python/Node.js: ~100-300ms cold start                       │
│ ├── Java/.NET: ~500ms-3s cold start (JVM/CLR startup)          │
│ ├── Move initialization outside handler (runs once per container)│
│ └── Use SnapStart (Java) for <200ms cold starts                 │
│                                                                       │
│ Limits (key ones):                                                   │
│ ├── Timeout: Max 15 minutes per invocation                      │
│ ├── Memory: 128 MB - 10,240 MB (CPU scales with memory)        │
│ ├── /tmp storage: 512 MB - 10,240 MB (ephemeral)               │
│ ├── Deployment package: 50 MB (zipped), 250 MB (unzipped)      │
│ ├── Container image: Up to 10 GB                                │
│ ├── Concurrent executions: 1,000 default (can increase)        │
│ ├── Payload: 6 MB (sync), 256 KB (async)                       │
│ └── Environment variables: 4 KB total                           │
│                                                                       │
│ Pricing:                                                             │
│ ├── Requests: $0.20 per million requests                        │
│ ├── Duration: $0.0000166667 per GB-second                       │
│ ├── Free tier: 1M requests + 400,000 GB-seconds per month      │
│ ├── Example: 128 MB function, 200ms avg, 1M invocations/month  │
│ │   = 1M × $0.20/M + (128/1024) × 0.2s × 1M × $0.0000166667  │
│ │   = $0.20 + $0.42 = $0.62/month                               │
│ └── ⚡ Extremely cheap for event-driven workloads!               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Lambda Function (Full Portal Walkthrough)

```
Console → Lambda → Create function

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE FUNCTION                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Choose creation method ──                                        │
│                                                                       │
│ ● Author from scratch                                              │
│   (Start with a simple Hello World function)                      │
│                                                                       │
│ ○ Use a blueprint                                                  │
│   (Pre-built templates: S3 thumbnail, DynamoDB trigger, etc.)    │
│                                                                       │
│ ○ Container image                                                  │
│   (Deploy as Docker image from ECR — up to 10 GB)                │
│                                                                       │
│ ○ Browse serverless app repository                                 │
│   (Community templates from AWS SAR)                              │
│                                                                       │
│ ── Basic information ──                                             │
│                                                                       │
│ Function name: [order-processor]                                   │
│ ⚡ Naming convention: {service}-{action} or {project}-{purpose}    │
│   Examples: order-processor, image-resizer, user-auth-verify     │
│                                                                       │
│ Runtime: [Python 3.12 ▼]                                           │
│   ○ Node.js 20.x                                                  │
│   ○ Node.js 18.x                                                  │
│   ● Python 3.12                                                    │
│   ○ Python 3.11                                                    │
│   ○ Java 21                                                        │
│   ○ Java 17                                                        │
│   ○ .NET 8                                                         │
│   ○ .NET 6                                                         │
│   ○ Ruby 3.3                                                       │
│   ○ Go (provided.al2023) — custom runtime                        │
│   ○ Rust (provided.al2023) — custom runtime                      │
│   ○ Custom runtime (provided.al2023)                              │
│   ⚡ Python/Node.js = fastest cold start, most popular            │
│   ⚡ Java = slowest cold start but use SnapStart                  │
│                                                                       │
│ Architecture: [x86_64 ▼]                                           │
│   ● x86_64 (Intel/AMD)                                            │
│   ○ arm64 (AWS Graviton2 — 20% cheaper, better performance)     │
│   ⚡ Use arm64 unless you need x86-specific native libraries     │
│                                                                       │
│ ── Permissions ──                                                   │
│                                                                       │
│ Execution role:                                                      │
│   ● Create a new role with basic Lambda permissions               │
│     (Creates: lambda-role-{function-name})                       │
│     (Permissions: CloudWatch Logs only)                          │
│                                                                       │
│   ○ Use an existing role                                           │
│     [my-lambda-execution-role ▼]                                 │
│     ⚡ Use this for production (pre-created role with right perms)│
│                                                                       │
│   ○ Create a new role from AWS policy templates                  │
│     Policy templates: [S3 read only access ▼]                   │
│                                                                       │
│ ── Advanced settings (expand) ──                                   │
│                                                                       │
│ □ Enable code signing                                              │
│   (Verify code is signed by trusted publisher)                   │
│                                                                       │
│ □ Enable function URL                                              │
│   ⚡ Creates an HTTPS endpoint directly (no API Gateway needed!) │
│   Auth type: ● AWS_IAM  ○ NONE                                  │
│   CORS: □ Configure cross-origin resource sharing               │
│                                                                       │
│ □ Enable VPC                                                       │
│   VPC: [vpc-prod ▼]                                              │
│   Subnets: [private-subnet-1a, private-subnet-1b ▼]            │
│   Security groups: [sg-lambda ▼]                                 │
│   ⚡ ONLY enable VPC if Lambda needs to access VPC resources     │
│     (RDS, ElastiCache, private APIs)                            │
│   ⚠️ VPC adds cold start latency (~1-2s extra)                  │
│     AWS mitigated with Hyperplane ENI (much better now)         │
│                                                                       │
│ □ Enable tags                                                      │
│   environment: prod                                               │
│   team: backend                                                   │
│                                                                       │
│ [Create function]                                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Function Configuration (After Creation)

```
Lambda → order-processor → Configuration tab

┌─────────────────────────────────────────────────────────────────────┐
│           GENERAL CONFIGURATION                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── General configuration ──                                         │
│                                                                       │
│ Memory: [512] MB                                                    │
│ ⚡ 128 MB - 10,240 MB (in 1 MB increments)                         │
│ ⚡ CPU scales proportionally with memory:                           │
│   ├── 128 MB = fractional vCPU                                   │
│   ├── 1,769 MB = 1 full vCPU                                     │
│   ├── 3,538 MB = 2 vCPUs                                         │
│   ├── 5,307 MB = 3 vCPUs                                         │
│   └── 10,240 MB = 6 vCPUs                                        │
│ ⚡ More memory = more CPU = faster execution = may cost LESS!     │
│   (Function runs faster → less duration billed)                  │
│   Use AWS Lambda Power Tuning tool to find optimal memory.       │
│                                                                       │
│ Timeout: [30] seconds                                               │
│ ⚡ 1 second - 15 minutes (900 seconds)                             │
│ ⚡ Default: 3 seconds (too short for most real workloads!)        │
│ ⚡ Best practice: Set to expected max + buffer                    │
│   API handler: 10-30s | Processing: 60-300s | ETL: 900s         │
│ ⚠️ If function hits timeout, it's KILLED (no graceful shutdown)  │
│                                                                       │
│ Ephemeral storage (/tmp): [512] MB                                 │
│ ⚡ 512 MB - 10,240 MB                                              │
│ ⚡ Use for temporary file processing (image resize, PDF gen)     │
│ ⚡ Persists between warm invocations (same container)             │
│ ⚠️ NOT persistent across cold starts                              │
│                                                                       │
│ ── Environment variables ──                                         │
│                                                                       │
│ Key                    │ Value                                     │
│ ──────────────────────┼──────────────────────────────────────     │
│ DB_HOST               │ mydb.cluster-abc.rds.amazonaws.com       │
│ DB_NAME               │ orders                                    │
│ STAGE                 │ prod                                      │
│ S3_BUCKET             │ my-app-uploads-prod                      │
│ LOG_LEVEL             │ INFO                                      │
│                                                                       │
│ ⚡ Max 4 KB total for all env vars                                 │
│ ⚡ For secrets: Use AWS Secrets Manager or SSM Parameter Store    │
│   (Don't put passwords in env vars directly!)                    │
│                                                                       │
│ Encryption configuration:                                           │
│   ● AWS managed key (aws/lambda)                                 │
│   ○ Customer managed key (KMS): [arn:aws:kms:...:key/xxx]       │
│   ⚡ Env vars encrypted at rest. Use KMS for extra control.      │
│                                                                       │
│ ── Execution role ──                                                │
│                                                                       │
│ Role: [arn:aws:iam::123456789012:role/lambda-order-processor]    │
│ ⚡ This IAM role determines what AWS services Lambda can access.  │
│ ⚡ Minimum permissions needed:                                     │
│   ├── AWSLambdaBasicExecutionRole (CloudWatch Logs — always)    │
│   ├── + AWSLambdaVPCAccessExecutionRole (if VPC enabled)        │
│   └── + custom policies for S3, DynamoDB, SQS, etc.            │
│                                                                       │
│ Example trust policy (allows Lambda service to assume role):     │
│ {                                                                    │
│   "Statement": [{                                                   │
│     "Effect": "Allow",                                              │
│     "Principal": {"Service": "lambda.amazonaws.com"},             │
│     "Action": "sts:AssumeRole"                                     │
│   }]                                                                 │
│ }                                                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Triggers & Event Sources

```
┌─────────────────────────────────────────────────────────────────────┐
│           TRIGGERS & EVENT SOURCES                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Lambda → order-processor → Add trigger                             │
│                                                                       │
│ Invocation models:                                                   │
│ ┌────────────────┬────────────────────────────────────────────────┐│
│ │ Model          │ Description                                    ││
│ ├────────────────┼────────────────────────────────────────────────┤│
│ │ Synchronous    │ Caller waits for response (API Gateway, ALB)  ││
│ │ Asynchronous   │ Fire-and-forget (S3, SNS, EventBridge)        ││
│ │ Stream/Poll    │ Lambda polls source (SQS, Kinesis, DynamoDB)  ││
│ └────────────────┴────────────────────────────────────────────────┘│
│                                                                       │
│ ══════════════════════════════════════════════════════════════════  │
│ MOST COMMON TRIGGERS:                                                │
│ ══════════════════════════════════════════════════════════════════  │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ 1. API Gateway (REST/HTTP API)                               │   │
│ │ Client → API Gateway → Lambda → Response                   │   │
│ │ Use: REST APIs, webhooks, microservices                     │   │
│ │                                                              │   │
│ │ Trigger config:                                             │   │
│ │ ├── API type: ● HTTP API (simpler, cheaper)                │   │
│ │ │             ○ REST API (full features, WAF)              │   │
│ │ ├── Security: IAM / API key / JWT / None                  │   │
│ │ └── Stage: $default / prod / dev                          │   │
│ │                                                              │   │
│ │ ⚡ Or use Function URL (simpler, no API Gateway needed):    │   │
│ │    https://abc123.lambda-url.ap-south-1.on.aws/            │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ 2. S3 Event Notifications                                    │   │
│ │ File uploaded → S3 event → Lambda                          │   │
│ │ Use: Image processing, file validation, ETL pipeline       │   │
│ │                                                              │   │
│ │ Trigger config:                                             │   │
│ │ ├── Bucket: [my-app-uploads ▼]                             │   │
│ │ ├── Event types:                                            │   │
│ │ │   ☑ s3:ObjectCreated:*  (any upload)                    │   │
│ │ │   ☐ s3:ObjectCreated:Put                                 │   │
│ │ │   ☐ s3:ObjectCreated:CompleteMultipartUpload            │   │
│ │ │   ☐ s3:ObjectRemoved:*                                   │   │
│ │ ├── Prefix: uploads/ (only trigger for this folder)       │   │
│ │ └── Suffix: .jpg (only trigger for JPG files)             │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ 3. SQS (Simple Queue Service)                                │   │
│ │ Messages in queue → Lambda polls → processes batch         │   │
│ │ Use: Order processing, email sending, async tasks          │   │
│ │                                                              │   │
│ │ Trigger config:                                             │   │
│ │ ├── SQS queue: [orders-queue ▼]                            │   │
│ │ ├── Batch size: [10] (1-10,000 messages per invocation)   │   │
│ │ ├── Batch window: [5] seconds (wait for batch to fill)    │   │
│ │ ├── Max concurrency: [5] (limit parallel executions)      │   │
│ │ └── Report batch item failures: ☑ (partial batch retry)  │   │
│ │ ⚡ Lambda deletes messages after successful processing     │   │
│ │ ⚡ Failed messages → DLQ (Dead Letter Queue)              │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ 4. DynamoDB Streams                                          │   │
│ │ Item change in DynamoDB → Stream record → Lambda          │   │
│ │ Use: Change data capture, sync to Elasticsearch, audit log│   │
│ │                                                              │   │
│ │ Trigger config:                                             │   │
│ │ ├── DynamoDB table: [orders-table ▼]                       │   │
│ │ ├── Batch size: [100]                                       │   │
│ │ ├── Starting position: ● Latest  ○ Trim horizon          │   │
│ │ ├── Batch window: [5] seconds                              │   │
│ │ ├── Retry attempts: [3] (-1 for infinite)                  │   │
│ │ └── Split batch on error: ☑ (isolate bad records)        │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ 5. EventBridge (CloudWatch Events)                           │   │
│ │ Scheduled or event-pattern triggered                       │   │
│ │ Use: Cron jobs, cross-service event routing                │   │
│ │                                                              │   │
│ │ Schedule: rate(5 minutes) or cron(0 8 * * ? *)            │   │
│ │ Event pattern: {"source": ["aws.ec2"],                     │   │
│ │   "detail-type": ["EC2 Instance State-change"]}           │   │
│ │                                                              │   │
│ │ ⚡ Best way to schedule Lambda (replaces CloudWatch Events)│   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ 6. SNS (Simple Notification Service)                         │   │
│ │ Topic message → Lambda (async invocation)                  │   │
│ │ Use: Fan-out, notifications, multi-subscriber processing  │   │
│ │                                                              │   │
│ │ 7. Kinesis Data Streams                                     │   │
│ │ Real-time stream records → Lambda (poll-based)            │   │
│ │ Use: Real-time analytics, IoT data processing             │   │
│ │                                                              │   │
│ │ 8. ALB (Application Load Balancer)                          │   │
│ │ HTTP request → ALB → Lambda (sync)                        │   │
│ │ Use: Lambda behind ALB (multi-target groups)              │   │
│ │                                                              │   │
│ │ 9. CloudFront (Lambda@Edge)                                 │   │
│ │ CDN request → Lambda at edge location                     │   │
│ │ Use: URL rewrite, A/B testing, auth at edge               │   │
│ │                                                              │   │
│ │ 10. Cognito (user pool triggers)                            │   │
│ │ Pre/post sign-up, authentication, token generation        │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Lambda Layers

```
┌─────────────────────────────────────────────────────────────────────┐
│           LAMBDA LAYERS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Reusable packages of code/libraries shared across functions. │
│                                                                       │
│ Without layers:                                                      │
│ ├── Function A: your code + boto3 + requests + pandas (50 MB)   │
│ ├── Function B: your code + boto3 + requests + numpy (45 MB)    │
│ └── Function C: your code + boto3 + requests (20 MB)            │
│     → Duplicated libraries everywhere!                           │
│                                                                       │
│ With layers:                                                         │
│ ├── Layer 1: "common-libs" (boto3, requests) — shared by all    │
│ ├── Layer 2: "data-libs" (pandas, numpy) — used by A & B       │
│ ├── Function A: your code only (1 MB) + Layer 1 + Layer 2      │
│ ├── Function B: your code only (1 MB) + Layer 1 + Layer 2      │
│ └── Function C: your code only (1 MB) + Layer 1                 │
│                                                                       │
│ Creating a layer:                                                    │
│ Lambda → Layers → Create layer                                    │
│                                                                       │
│ Name: [common-python-libs]                                          │
│ Description: [Shared Python libraries - requests, boto3]          │
│ Upload: [.zip file]                                                  │
│                                                                       │
│ ⚡ ZIP structure for Python:                                         │
│    python/                                                           │
│    └── lib/                                                          │
│        └── python3.12/                                               │
│            └── site-packages/                                        │
│                ├── requests/                                          │
│                └── boto3/                                             │
│                                                                       │
│ ⚡ ZIP structure for Node.js:                                        │
│    nodejs/                                                            │
│    └── node_modules/                                                 │
│        ├── axios/                                                    │
│        └── lodash/                                                   │
│                                                                       │
│ Compatible runtimes: [Python 3.11, Python 3.12 ▼]                 │
│ Compatible architectures: [x86_64, arm64 ▼]                       │
│                                                                       │
│ Limits:                                                              │
│ ├── Max 5 layers per function                                    │
│ ├── Total unzipped: 250 MB (function + all layers)              │
│ ├── Layer size: up to 50 MB (zipped)                            │
│ └── Layers are versioned (immutable versions)                    │
│                                                                       │
│ Adding layer to function:                                           │
│ Lambda → function → Layers → Add a layer                         │
│ ├── AWS layers (AWS-provided: AWS SDK, etc.)                    │
│ ├── Custom layers (your layers)                                  │
│ └── Specify ARN (cross-account layer sharing)                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Versions & Aliases

```
┌─────────────────────────────────────────────────────────────────────┐
│           VERSIONS & ALIASES                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ VERSIONS:                                                            │
│ ├── $LATEST: Mutable, always points to latest code               │
│ ├── Version 1, 2, 3...: Immutable snapshots of code + config    │
│ ├── Publish version: Lambda → Actions → Publish new version     │
│ ├── Each version gets unique ARN:                                │
│ │   arn:aws:lambda:region:account:function:order-processor:3    │
│ └── Use for: Stable deployments, rollback capability            │
│                                                                       │
│ ALIASES:                                                             │
│ ├── Named pointer to a version (mutable)                        │
│ ├── Example: "prod" → Version 5, "staging" → Version 6         │
│ ├── Each alias gets stable ARN:                                  │
│ │   arn:aws:lambda:region:account:function:order-processor:prod │
│ ├── Point triggers to alias (not $LATEST or version number!)   │
│ └── Deploy: Update alias to point to new version                │
│                                                                       │
│ WEIGHTED ALIAS (canary deployments):                                │
│ ├── Route % of traffic to different version                     │
│ ├── Example: "prod" alias → 95% Version 5, 5% Version 6       │
│ ├── Monitor → if good → shift to 100% Version 6               │
│ └── ⚡ Built-in canary without any extra infrastructure!         │
│                                                                       │
│ Deployment flow:                                                     │
│ 1. Upload new code to $LATEST                                      │
│ 2. Test $LATEST                                                      │
│ 3. Publish as Version N                                             │
│ 4. Update alias "prod" → Version N (or weighted split)           │
│ 5. Monitor → if issues → point alias back to Version N-1        │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ API Gateway ──► alias "prod" ──► Version 5 (95%)            │  │
│ │                                 └──► Version 6 (5% canary)  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Concurrency

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONCURRENCY MANAGEMENT                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Default: 1,000 concurrent executions per region (account limit).  │
│ All functions in a region SHARE this pool.                         │
│                                                                       │
│ Types of concurrency:                                                │
│                                                                       │
│ 1. Unreserved Concurrency (default):                                │
│    ├── Function uses shared pool                                 │
│    ├── Can be starved by other functions                        │
│    └── No guarantees                                              │
│                                                                       │
│ 2. Reserved Concurrency:                                            │
│    ├── GUARANTEE N concurrent executions for this function       │
│    ├── Also LIMITS function to N (won't exceed)                 │
│    ├── Taken from shared pool (reduces pool for others)         │
│    ├── FREE — no extra cost                                      │
│    └── Use for: Protect critical functions, throttle functions  │
│                                                                       │
│    Lambda → function → Configuration → Concurrency               │
│    Reserved concurrency: [100]                                     │
│    ⚡ Set to 0 = function DISABLED (throttled 100%)              │
│                                                                       │
│ 3. Provisioned Concurrency:                                         │
│    ├── Pre-initialize N execution environments                  │
│    ├── Eliminates cold starts (always warm!)                    │
│    ├── COSTS MONEY ($$$) — you pay for idle provisioned envs   │
│    ├── Applied to a version or alias (not $LATEST)             │
│    └── Use for: Latency-sensitive APIs, predictable traffic     │
│                                                                       │
│    Lambda → function → Configuration → Concurrency               │
│    Provisioned concurrency: Version/Alias → [prod] → [50]      │
│                                                                       │
│    ⚡ Cost: ~$0.015 per GB-hour provisioned                      │
│    Example: 512MB × 50 provisioned = 25 GB provisioned          │
│    = 25 × $0.015 × 24h × 30d = $270/month                      │
│    → Only use for functions where cold start is unacceptable!  │
│                                                                       │
│ Auto Scaling Provisioned Concurrency:                               │
│    ├── Use Application Auto Scaling to schedule provisioned     │
│    ├── Scale up during business hours, scale down at night      │
│    └── Saves cost vs 24/7 provisioned concurrency              │
│                                                                       │
│ Burst concurrency:                                                   │
│ ├── Initial burst: 500-3000 (varies by region)                  │
│ ├── Then scales +500/minute until account limit                 │
│ └── If you hit limit → throttled (429 Too Many Requests)       │
│                                                                       │
│ Request increase: Service Quotas → Lambda → Concurrent executions│
│ (Can get 10,000+ with justification)                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Destinations

```
┌─────────────────────────────────────────────────────────────────────┐
│           DESTINATIONS (Async Invocation Routing)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Route async invocation results to another service.           │
│ Only applies to ASYNCHRONOUS invocations (S3, SNS, EventBridge).  │
│                                                                       │
│ Lambda → function → Configuration → Asynchronous invocation      │
│                                                                       │
│ On success → send to: [SQS queue / SNS topic / Lambda / EventBridge]│
│ On failure → send to: [SQS queue / SNS topic / Lambda / EventBridge]│
│                                                                       │
│ Example:                                                             │
│ ┌───────────┐     success     ┌──────────────────────┐            │
│ │ S3 event  │ → Lambda ──────► │ SNS: notify-success │            │
│ │ (trigger) │        │         └──────────────────────┘            │
│ └───────────┘        │ failure ┌──────────────────────┐            │
│                      └────────► │ SQS: dead-letters   │            │
│                                 └──────────────────────┘            │
│                                                                       │
│ Async retry behavior:                                                │
│ ├── Max retry attempts: [2] (0 = no retries, 2 = default)       │
│ ├── Max event age: [6 hours] (1 min - 6 hours)                  │
│ │   Events older than this are discarded                        │
│ └── Dead-letter queue: [arn:aws:sqs:...:dlq-orders]             │
│     ⚡ DLQ = older approach. Destinations = newer, more flexible │
│                                                                       │
│ Destinations vs DLQ:                                                 │
│ ├── DLQ: Only captures failures, only SQS/SNS                  │
│ ├── Destinations: Success AND failure, more targets            │
│ └── ⚡ Use Destinations for new functions (preferred)            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Lambda@Edge & CloudFront Functions

```
┌─────────────────────────────────────────────────────────────────────┐
│           EDGE COMPUTE                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Lambda@Edge:                                                         │
│ ├── Run Lambda at CloudFront edge locations (globally)           │
│ ├── Must be in us-east-1 (deployed globally by CloudFront)      │
│ ├── Triggers: Viewer Request, Viewer Response,                   │
│ │            Origin Request, Origin Response                    │
│ ├── Limits: 5s (viewer), 30s (origin), 128-10240 MB            │
│ ├── Use for: Auth at edge, URL rewrite, A/B testing, SEO       │
│ └── Runtimes: Node.js, Python only                               │
│                                                                       │
│ CloudFront Functions:                                                │
│ ├── Lightweight JavaScript functions at edge                     │
│ ├── Sub-millisecond execution                                    │
│ ├── Triggers: Viewer Request, Viewer Response ONLY              │
│ ├── Limits: <1ms, 2MB, 10KB function size                      │
│ ├── 1/6th the cost of Lambda@Edge                               │
│ └── Use for: Header manipulation, URL rewrite, simple redirects│
│                                                                       │
│ Decision:                                                            │
│ ├── Simple header/URL manipulation → CloudFront Functions       │
│ ├── Complex logic, network calls, >1ms → Lambda@Edge           │
│ └── Backend processing → Regular Lambda                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Container Image Support

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONTAINER IMAGE DEPLOYMENT                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Deploy Lambda functions as Docker container images.          │
│ Why: Larger packages (up to 10 GB), custom runtimes, existing     │
│      Docker workflows.                                              │
│                                                                       │
│ Requirements:                                                        │
│ ├── Image must be in ECR (Elastic Container Registry)            │
│ ├── Must implement Lambda Runtime API                            │
│ ├── Use AWS base images (easiest) or custom images               │
│ └── Max image size: 10 GB                                        │
│                                                                       │
│ Dockerfile (Python example):                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ FROM public.ecr.aws/lambda/python:3.12                       │  │
│ │                                                              │  │
│ │ COPY requirements.txt .                                      │  │
│ │ RUN pip install -r requirements.txt                          │  │
│ │                                                              │  │
│ │ COPY app.py .                                                │  │
│ │                                                              │  │
│ │ CMD ["app.handler"]                                          │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Deploy:                                                              │
│ docker build -t my-lambda .                                         │
│ docker tag my-lambda:latest 123456789012.dkr.ecr.region.amazonaws.com/my-lambda:latest│
│ docker push 123456789012.dkr.ecr.region.amazonaws.com/my-lambda:latest│
│                                                                       │
│ Then: Create function → Container image → ECR image URI           │
│                                                                       │
│ ⚡ Use container images when:                                        │
│   ├── Package > 250 MB (zip limit)                              │
│   ├── Custom runtime needed (Rust, C++, PHP)                   │
│   ├── Team already uses Docker workflow                         │
│   └── Need ML libraries (PyTorch, TensorFlow = multi-GB)      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Terraform & CLI Examples

### Terraform

```hcl
# Lambda function (zip deployment)
resource "aws_lambda_function" "order_processor" {
  function_name = "order-processor"
  role          = aws_iam_role.lambda_role.arn
  handler       = "app.handler"
  runtime       = "python3.12"
  architectures = ["arm64"]
  
  filename         = "lambda.zip"
  source_code_hash = filebase64sha256("lambda.zip")
  
  memory_size = 512
  timeout     = 30
  
  environment {
    variables = {
      DB_HOST    = aws_rds_cluster.main.endpoint
      S3_BUCKET  = aws_s3_bucket.uploads.id
      STAGE      = "prod"
    }
  }
  
  # VPC (optional)
  vpc_config {
    subnet_ids         = [aws_subnet.private_1.id, aws_subnet.private_2.id]
    security_group_ids = [aws_security_group.lambda.id]
  }
  
  # Dead letter queue
  dead_letter_config {
    target_arn = aws_sqs_queue.lambda_dlq.arn
  }
  
  tags = {
    environment = "prod"
    team        = "backend"
  }
}

# IAM role for Lambda
resource "aws_iam_role" "lambda_role" {
  name = "lambda-order-processor-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "lambda_vpc" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

# S3 trigger
resource "aws_lambda_permission" "s3" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.order_processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.uploads.arn
}

resource "aws_s3_bucket_notification" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  
  lambda_function {
    lambda_function_arn = aws_lambda_function.order_processor.arn
    events              = ["s3:ObjectCreated:*"]
    filter_prefix       = "uploads/"
    filter_suffix       = ".json"
  }
}

# SQS trigger
resource "aws_lambda_event_source_mapping" "sqs" {
  event_source_arn = aws_sqs_queue.orders.arn
  function_name    = aws_lambda_function.order_processor.arn
  batch_size       = 10
  maximum_batching_window_in_seconds = 5
  
  function_response_types = ["ReportBatchItemFailures"]
}

# API Gateway trigger (HTTP API)
resource "aws_apigatewayv2_api" "api" {
  name          = "order-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda" {
  api_id             = aws_apigatewayv2_api.api.id
  integration_type   = "AWS_PROXY"
  integration_uri    = aws_lambda_function.order_processor.invoke_arn
  integration_method = "POST"
}

resource "aws_apigatewayv2_route" "post_order" {
  api_id    = aws_apigatewayv2_api.api.id
  route_key = "POST /orders"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}

# Lambda Layer
resource "aws_lambda_layer_version" "common_libs" {
  layer_name          = "common-python-libs"
  filename            = "layer.zip"
  source_code_hash    = filebase64sha256("layer.zip")
  compatible_runtimes = ["python3.11", "python3.12"]
  compatible_architectures = ["arm64", "x86_64"]
}

# Version & Alias with weighted routing (canary)
resource "aws_lambda_alias" "prod" {
  name             = "prod"
  function_name    = aws_lambda_function.order_processor.function_name
  function_version = aws_lambda_function.order_processor.version
  
  routing_config {
    additional_version_weights = {
      (aws_lambda_function.order_processor_canary.version) = 0.05
    }
  }
}

# Provisioned concurrency
resource "aws_lambda_provisioned_concurrency_config" "prod" {
  function_name                  = aws_lambda_function.order_processor.function_name
  qualifier                      = aws_lambda_alias.prod.name
  provisioned_concurrent_executions = 50
}

# EventBridge schedule (cron)
resource "aws_cloudwatch_event_rule" "every_5_min" {
  name                = "every-5-minutes"
  schedule_expression = "rate(5 minutes)"
}

resource "aws_cloudwatch_event_target" "lambda" {
  rule = aws_cloudwatch_event_rule.every_5_min.name
  arn  = aws_lambda_function.order_processor.arn
}

resource "aws_lambda_permission" "eventbridge" {
  statement_id  = "AllowEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.order_processor.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.every_5_min.arn
}
```

### AWS CLI

```bash
# Create function (zip)
aws lambda create-function \
  --function-name order-processor \
  --runtime python3.12 \
  --architectures arm64 \
  --handler app.handler \
  --role arn:aws:iam::123456789012:role/lambda-order-processor-role \
  --zip-file fileb://lambda.zip \
  --memory-size 512 \
  --timeout 30 \
  --environment "Variables={DB_HOST=mydb.cluster.rds.amazonaws.com,STAGE=prod}"

# Update function code
aws lambda update-function-code \
  --function-name order-processor \
  --zip-file fileb://lambda.zip

# Update configuration
aws lambda update-function-configuration \
  --function-name order-processor \
  --memory-size 1024 \
  --timeout 60

# Invoke function (test)
aws lambda invoke \
  --function-name order-processor \
  --payload '{"key": "value"}' \
  --cli-binary-format raw-in-base64-out \
  output.json

# Publish version
aws lambda publish-version \
  --function-name order-processor \
  --description "v1.2.0 - added retry logic"

# Create/update alias
aws lambda create-alias \
  --function-name order-processor \
  --name prod \
  --function-version 5

# Weighted alias (canary: 95% v5, 5% v6)
aws lambda update-alias \
  --function-name order-processor \
  --name prod \
  --function-version 5 \
  --routing-config '{"AdditionalVersionWeights":{"6":0.05}}'

# Set reserved concurrency
aws lambda put-function-concurrency \
  --function-name order-processor \
  --reserved-concurrent-executions 100

# Set provisioned concurrency
aws lambda put-provisioned-concurrency-config \
  --function-name order-processor \
  --qualifier prod \
  --provisioned-concurrent-executions 50

# Create layer
aws lambda publish-layer-version \
  --layer-name common-python-libs \
  --zip-file fileb://layer.zip \
  --compatible-runtimes python3.11 python3.12 \
  --compatible-architectures arm64 x86_64

# Add layer to function
aws lambda update-function-configuration \
  --function-name order-processor \
  --layers arn:aws:lambda:ap-south-1:123456789012:layer:common-python-libs:1

# List functions
aws lambda list-functions \
  --query "Functions[].{Name:FunctionName,Runtime:Runtime,Memory:MemorySize,Timeout:Timeout}" \
  --output table

# Get function URL
aws lambda create-function-url-config \
  --function-name order-processor \
  --auth-type NONE \
  --cors '{"AllowOrigins":["*"],"AllowMethods":["GET","POST"]}'

# View logs (recent)
aws logs tail /aws/lambda/order-processor --since 1h --follow
```

---

## Part 12: Real-World Patterns

### Startup

```
Lambda-first architecture (minimize infra):

Functions:
├── api-handler (API Gateway HTTP API → Lambda)
│   ├── Runtime: Python 3.12 / Node.js 20
│   ├── Memory: 256-512 MB
│   ├── Timeout: 10s
│   └── Handles: All REST API routes
├── image-processor (S3 trigger)
│   ├── Triggered on upload to S3
│   ├── Resizes images, generates thumbnails
│   └── Memory: 1024 MB (image processing)
├── email-sender (SQS trigger)
│   ├── Processes email queue
│   └── Sends via SES
└── daily-report (EventBridge schedule)
    ├── Cron: 0 8 * * ? * (daily 8 AM)
    └── Generates and emails daily report

Architecture:
API Gateway → Lambda → DynamoDB (or RDS)
S3 → Lambda → S3 (processed)
SQS → Lambda → SES

Cost: $5-50/month for typical startup traffic
(Free tier covers 1M requests/month!)

Best practices:
├── Use arm64 (20% cheaper)
├── Use Function URLs for simple webhooks (no API GW cost)
├── Keep functions focused (single responsibility)
├── Use layers for shared code
└── Monitor with CloudWatch Insights (free tier)
```

### Mid-Size

```
Microservices with Lambda:

Functions (20-50 functions):
├── order-service/
│   ├── create-order (API GW)
│   ├── process-order (SQS)
│   ├── order-stream-handler (DynamoDB Streams)
│   └── order-notifications (SNS)
├── user-service/
│   ├── auth-handler (Cognito triggers)
│   ├── profile-api (API GW)
│   └── user-events (EventBridge)
├── payment-service/
│   ├── process-payment (API GW)
│   └── payment-webhook (Function URL)
└── analytics/
    ├── event-processor (Kinesis)
    └── daily-aggregation (EventBridge schedule)

Deployment:
├── AWS SAM or Serverless Framework
├── CI/CD: GitHub Actions → SAM deploy
├── Aliases: dev, staging, prod per function
├── Canary: 5% weighted alias on deploy → auto-rollback

Concurrency:
├── Reserved: Critical functions (order-processor: 200)
├── Provisioned: API-facing functions (50 during business hours)
└── Auto-scaling provisioned concurrency (schedule-based)

Monitoring:
├── CloudWatch Lambda Insights (enhanced metrics)
├── X-Ray tracing (distributed tracing across functions)
├── CloudWatch Alarms: throttles, errors, duration P99
└── Powertools for Lambda (structured logging, tracing, metrics)

Cost: $100-500/month
```

### Enterprise

```
Large-scale Lambda deployment:

Scale: 200+ Lambda functions, 50M+ invocations/month

Organization:
├── Shared layers (managed by platform team)
│   ├── observability-layer (logging, tracing, metrics)
│   ├── security-layer (secret fetching, auth validation)
│   └── common-utils-layer (shared business logic)
├── Function templates (cookiecutter/yeoman)
├── Centralized CI/CD pipeline (CodePipeline or GitHub Actions)
└── Lambda function factory pattern (standardized creation)

Deployment strategy:
├── AWS SAM + CodeDeploy
├── Canary deployments: 5% → 25% → 100% (auto-rollback on errors)
├── Traffic shifting with CodeDeploy hooks (pre/post traffic)
├── Separate accounts: dev, staging, prod (AWS Organizations)
└── Cross-account deployment roles

Performance:
├── Provisioned concurrency: API-facing (100-500)
├── SnapStart: Java functions (eliminate cold start)
├── arm64 everywhere (cost savings at scale)
├── Lambda Power Tuning: Automated memory optimization
└── Connection pooling: RDS Proxy for database connections

Security:
├── VPC for all DB-accessing functions
├── No env var secrets → Secrets Manager + caching
├── Resource policies (restrict who can invoke)
├── Code signing (ensure only approved code deploys)
├── IAM: Least-privilege per function (no shared roles!)
└── Automated security scanning (Snyk, Checkov)

Observability:
├── CloudWatch Lambda Insights
├── X-Ray: End-to-end distributed tracing
├── Datadog/New Relic Lambda integration
├── Custom metrics (Powertools for Lambda)
├── Centralized logging (CloudWatch → OpenSearch)
└── Anomaly detection on invocation patterns

Cost optimization:
├── arm64 everywhere: ~20% savings
├── Right-size memory (Lambda Power Tuning)
├── Provisioned concurrency only during peak hours
├── Compute Savings Plans (17% savings on Lambda)
├── Review unused functions (automated cleanup)
└── Cost: $2,000-10,000/month at enterprise scale
```

---

## Troubleshooting: Common Lambda Issues

### "My Lambda function times out"

```
1. ☐ Is the timeout set correctly?
   Default is 3 seconds — increase up to 900 seconds (15 minutes)
   Console → Lambda → Function → Configuration → General configuration

2. ☐ Is Lambda in a VPC without internet access?
   Lambda in a VPC needs a NAT Gateway to reach the internet
   or VPC Endpoints for AWS services

3. ☐ Is the downstream service (RDS, API) slow or unreachable?
   Check CloudWatch logs for connection errors
```

### "My Lambda has high latency (cold starts)"

```
Cold start = Lambda creating a new execution environment (~100ms-10s)

Solutions:
1. Use smaller deployment packages (faster to load)
2. Use ARM (Graviton) — faster cold starts than x86
3. Use Provisioned Concurrency for critical functions
4. Keep functions warm with scheduled pings (not recommended for production)
5. Choose higher memory (more memory = more CPU = faster startup)
```

### "AccessDenied when calling AWS services"

```
1. ☐ Does the Lambda execution role have the required permission?
   Console → Lambda → Configuration → Permissions → Execution role
   Click the role → check attached policies

2. ☐ Is there a resource policy on the target service?
   (e.g., S3 bucket policy, SQS queue policy)
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| 3-second default timeout | Functions timeout on API calls | Set timeout to 30-60s for API calls |
| 128 MB default memory | Slow execution, cold starts | Start with 256-512 MB, optimize with Power Tuning |
| Putting Lambda in VPC unnecessarily | Cold starts increase, needs NAT GW | Only use VPC if accessing private resources |
| Not using environment variables | Hardcoded values, can't change per stage | Use env vars for endpoints, table names, etc. |
| Returning large payloads | 6 MB response limit, slow | Return S3 pre-signed URLs for large data |

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Serverless compute — run code without managing servers |
| Pricing | $0.20/million requests + $0.0000166667/GB-second |
| Free tier | 1M requests + 400,000 GB-seconds/month |
| Runtimes | Python, Node.js, Java, .NET, Go, Ruby, Custom, Container |
| Memory | 128 MB - 10,240 MB (CPU scales proportionally) |
| Timeout | 1s - 15 minutes |
| Package | 50 MB zip, 250 MB unzipped, 10 GB container image |
| Concurrency | 1,000 default (account-level, can increase) |
| Triggers | 100+ (API GW, S3, SQS, DynamoDB, EventBridge, etc.) |
| Layers | Shared libraries, max 5 per function |
| Versions | Immutable snapshots of code + config |
| Aliases | Named pointers to versions, weighted routing for canary |
| Provisioned concurrency | Pre-warmed environments (eliminates cold start) |
| Destinations | Route async success/failure to SQS/SNS/Lambda/EventBridge |
| Lambda@Edge | Run at CloudFront edge locations globally |
| GCP equivalent | Cloud Functions / Cloud Run |
| Azure equivalent | Azure Functions |

---

## What's Next?

In the next chapter, we'll cover Amazon ECS — Elastic Container Service for running Docker containers.

→ Next: [Chapter 16: ECS - Elastic Container Service](16-ecs.md)

---

*Last Updated: May 2026*
