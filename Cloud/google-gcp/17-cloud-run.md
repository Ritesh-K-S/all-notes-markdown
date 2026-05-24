# Chapter 17: Cloud Run

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud Run Fundamentals](#part-1-cloud-run-fundamentals)
- [Part 2: Creating a Cloud Run Service (Full Console Walkthrough)](#part-2-creating-a-cloud-run-service-full-console-walkthrough)
- [Part 3: Revisions & Traffic Splitting](#part-3-revisions--traffic-splitting)
- [Part 4: Cloud Run Jobs](#part-4-cloud-run-jobs)
- [Part 5: Custom Domains](#part-5-custom-domains)
- [Part 6: Terraform](#part-6-terraform)
- [Part 7: gcloud CLI Reference](#part-7-gcloud-cli-reference)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [Console Walkthrough: Managing & Deleting Services](#console-walkthrough-managing--deleting-services)
- [Cloud Run + Cloud SQL Connection](#cloud-run--cloud-sql-connection)
- [Pub/Sub Push to Cloud Run](#pubsub-push-to-cloud-run)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)
- [What's Next?](#whats-next)

---

## Overview

Cloud Run is Google Cloud's fully managed platform for running containers. You give it a container image, Cloud Run runs it, scales it (including to zero), handles HTTPS, and charges you only when your code is handling requests. No servers, no clusters, no node management.

```
What you'll learn:
├── Cloud Run Fundamentals
│   ├── What & why (managed containers, scale to zero)
│   ├── Cloud Run vs Cloud Functions vs GKE
│   └── Pricing model (per request + CPU/memory)
├── Cloud Run Services (Full Console Walkthrough)
│   ├── Container image, port, command
│   ├── Revisions (immutable deployments)
│   ├── Autoscaling (min/max instances, concurrency)
│   ├── CPU allocation (request-based vs always-on)
│   ├── Environment variables & secrets
│   ├── Networking (VPC, ingress, egress)
│   ├── Security (IAM, service account, auth)
│   └── Traffic splitting (canary deployments)
├── Cloud Run Jobs
│   ├── Run-to-completion tasks
│   ├── Parallelism, task count, retries
│   └── Scheduled jobs with Cloud Scheduler
├── Custom Domains
├── Terraform & gcloud CLI
└── Real-world patterns
```

---

## Part 1: Cloud Run Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD RUN CONCEPT                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Cloud Run?                                                  │
│ ├── You give it a container image (any language/runtime)         │
│ ├── Cloud Run runs it behind HTTPS                               │
│ ├── Scales from 0 to thousands of instances automatically       │
│ ├── You pay only when handling requests (or always-allocated)   │
│ └── No servers, VMs, clusters, or orchestration to manage       │
│                                                                       │
│ Think of it as:                                                      │
│ "Docker containers as a service, with auto-scaling & HTTPS"      │
│                                                                       │
│ Key concepts:                                                        │
│ ┌────────────────────────────────────────────────────────────┐    │
│ │ Service   │ Long-running HTTP endpoint. Scale up/down.      │    │
│ │           │ Each deploy creates a new Revision.             │    │
│ ├───────────┼────────────────────────────────────────────────-┤    │
│ │ Revision  │ Immutable snapshot of code + config.            │    │
│ │           │ Traffic can split across revisions.             │    │
│ ├───────────┼────────────────────────────────────────────────-┤    │
│ │ Instance  │ A running container handling requests.           │    │
│ │           │ Auto-created, auto-destroyed.                   │    │
│ ├───────────┼────────────────────────────────────────────────-┤    │
│ │ Job       │ Run-to-completion task (not HTTP). Execute      │    │
│ │           │ once or on schedule. Exits when done.           │    │
│ └───────────┴────────────────────────────────────────────────-┘    │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                 Cloud Run Service                            │  │
│ │                                                              │  │
│ │ Request → HTTPS endpoint → Cloud Run                       │  │
│ │                              │                              │  │
│ │              ┌───────────────┼───────────────┐              │  │
│ │              ▼               ▼               ▼              │  │
│ │         [Instance 1]   [Instance 2]   [Instance 3]         │  │
│ │         (Revision 3)   (Revision 3)   (Revision 3)         │  │
│ │                                                              │  │
│ │ No requests? → Scale to 0 instances (pay nothing!)         │  │
│ │ 1000 requests/sec? → Scale to 100+ instances               │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Comparison:                                                          │
│ ┌────────────────┬──────────────┬──────────────┬──────────────┐  │
│ │ Feature         │ Cloud Run     │ Cloud Funcs   │ GKE          │  │
│ ├────────────────┼──────────────┼──────────────┼──────────────┤  │
│ │ Unit            │ Container     │ Function      │ Pod/Cluster  │  │
│ │ Scale to zero   │ Yes ✅        │ Yes ✅        │ No ❌        │  │
│ │ Any language    │ Yes (Docker) │ Select runtimes│ Yes (Docker)│  │
│ │ Concurrency     │ 1-1000/inst  │ 1 (Gen1)      │ Unlimited   │  │
│ │                │              │ 1-1000 (Gen2) │             │  │
│ │ Max timeout     │ 60 min       │ 60 min (Gen2) │ Unlimited   │  │
│ │ Infra mgmt      │ None ✅      │ None ✅       │ Cluster mgmt│  │
│ │ Pricing         │ Per-request  │ Per-invocation│ Per-node 24/7│  │
│ │ Complexity      │ Low ✅       │ Lowest ✅     │ High         │  │
│ │ Best for        │ APIs, web    │ Events, glue  │ Complex apps │  │
│ └────────────────┴──────────────┴──────────────┴──────────────┘  │
│                                                                       │
│ ⚡ AWS equivalent: App Runner (simple) or Fargate (more control)  │
│ ⚡ Azure equivalent: Container Apps                                │
│                                                                       │
│ Pricing:                                                             │
│ ├── CPU: $0.00002400/vCPU-second (request-based allocation)    │
│ ├── Memory: $0.00000250/GiB-second                              │
│ ├── Requests: $0.40 per million                                  │
│ ├── Free tier: 2 million requests, 360K vCPU-seconds/month     │
│ └── Always-allocated CPU: 2.5× more expensive but always warm  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Cloud Run Service (Full Console Walkthrough)

```
Console → Cloud Run → Create Service

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE SERVICE — SOURCE                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Deploy from:                                                        │
│   ● Deploy one revision from an existing container image          │
│   ○ Continuously deploy from a repository (Cloud Build)          │
│                                                                       │
│ ═══ OPTION 1: Container image ═══                                  │
│ Container image URL:                                                │
│ [asia-south1-docker.pkg.dev/my-project/images/web-app:v1.2 ▼]  │
│ [SELECT] → Browse Artifact Registry or Container Registry         │
│                                                                       │
│ ⚡ Image sources:                                                    │
│   ├── Artifact Registry (recommended): region-docker.pkg.dev/.. │
│   ├── Container Registry (legacy): gcr.io/project/image:tag     │
│   ├── Docker Hub: docker.io/library/nginx:latest                │
│   └── Any public/private registry                                │
│                                                                       │
│ ═══ OPTION 2: Continuous deployment ═══                            │
│ Repository: [GitHub ▼]                                              │
│ ├── Connect GitHub account                                        │
│ ├── Select repository                                             │
│ ├── Branch: main                                                  │
│ ├── Build type: Dockerfile or Buildpacks                         │
│ ├── Dockerfile path: /Dockerfile                                  │
│ └── ⚡ Auto-deploys on every push to branch!                      │
│     Uses Cloud Build behind the scenes.                          │
│                                                                       │
│ Service name: [web-app]                                             │
│ Region: [asia-south1 ▼]                                           │
│ ⚡ Choose region closest to users. Cloud Run is regional          │
│   (not global). For multi-region: Deploy to multiple regions    │
│   + Cloud Load Balancing in front.                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           CONFIGURE — FIRST REVISION                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Container(s) ──                                                  │
│                                                                       │
│ Container port: [8080]                                              │
│ ⚡ Cloud Run sends HTTP requests to this port.                     │
│   Your app MUST listen on this port.                              │
│   Default: 8080. Set via $PORT env var (auto-injected).          │
│                                                                       │
│ Command: [] (optional — override Docker ENTRYPOINT)               │
│ Arguments: [] (optional — override Docker CMD)                    │
│                                                                       │
│ ── Resources ──                                                     │
│                                                                       │
│ CPU: [1 ▼]                                                         │
│ ┌────────────┬──────────────────────────────────────────────────┐ │
│ │ CPU        │ Memory options                                    │ │
│ ├────────────┼──────────────────────────────────────────────────┤ │
│ │ 1 vCPU     │ 128 MiB - 4 GiB                                 │ │
│ │ 2 vCPU     │ 256 MiB - 8 GiB                                 │ │
│ │ 4 vCPU     │ 512 MiB - 16 GiB                                │ │
│ │ 8 vCPU     │ 1 GiB - 32 GiB                                  │ │
│ └────────────┴──────────────────────────────────────────────────┘ │
│                                                                       │
│ Memory: [512 MiB ▼]                                               │
│                                                                       │
│ CPU allocation:                                                      │
│ ● CPU is only allocated during request processing                │
│ ○ CPU is always allocated                                         │
│                                                                       │
│ ⚡ CPU allocation modes:                                             │
│   Request-based (default):                                        │
│   ├── CPU throttled to near-zero between requests               │
│   ├── Cheaper (pay only during processing)                      │
│   ├── Cold start possible (instance idle → request arrives)    │
│   └── Best for: HTTP APIs, web apps, intermittent traffic      │
│                                                                       │
│   Always-allocated:                                                │
│   ├── CPU runs at full speed even between requests              │
│   ├── More expensive (2.5× CPU cost)                            │
│   ├── No cold start (CPU always warm)                           │
│   ├── Can do background work (process queues, websockets)      │
│   └── Best for: WebSockets, background processing, low-latency│
│                                                                       │
│ Startup CPU boost: ☑ Enabled                                      │
│ ⚡ Temporarily gives more CPU during startup for faster boot.     │
│                                                                       │
│ ── Execution environment ──                                        │
│ ○ Default (Gen 1 — gVisor sandbox)                                │
│ ● Second generation (Gen 2 — full Linux, better perf) ✅         │
│ ⚡ Gen 2: Full Linux compatibility, network file systems,         │
│   better CPU perf. Use for most workloads.                      │
│                                                                       │
│ ── Environment variables ──                                        │
│ [+ Add variable]                                                    │
│ Name: NODE_ENV          Value: production                         │
│ Name: API_BASE_URL      Value: https://api.example.com           │
│                                                                       │
│ ── Secrets ──                                                       │
│ [+ Reference a secret]                                              │
│ Secret: [db-password ▼] (from Secret Manager)                    │
│ Reference method:                                                   │
│   ● Exposed as environment variable                               │
│     Name: DB_PASSWORD                                              │
│     Version: [latest ▼] or [3]                                   │
│   ○ Mounted as volume                                             │
│     Mount path: /secrets/db                                       │
│                                                                       │
│ ── Volume mounts ──                                                 │
│ [+ Add volume]                                                      │
│ Volume type:                                                        │
│   ○ Cloud Storage bucket (FUSE mount — read/write)               │
│   ○ In-memory (emptyDir — for temp files, shared between sidecars)│
│   ○ NFS (network file system — Gen 2 only)                      │
│                                                                       │
│ ── Health checks ──                                                 │
│ Startup probe:                                                      │
│   Type: [HTTP ▼]  Path: [/health]  Port: [8080]                 │
│   Initial delay: [0] sec  Period: [10] sec  Timeout: [1] sec   │
│   Failure threshold: [3]                                          │
│ Liveness probe:                                                     │
│   Type: [HTTP ▼]  Path: [/health]  Period: [10] sec            │
│                                                                       │
│ ── Sidecar containers ──                                           │
│ [+ Add container] ← Add sidecar (proxy, log agent, etc.)        │
│ ⚡ Multi-container support. Main container + sidecars share       │
│   networking (localhost) and volumes.                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           AUTOSCALING                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Minimum number of instances: [0]                                   │
│ ⚡ 0 = Scale to zero (save cost, but cold start on first request)│
│   1+ = Always keep N warm (no cold start, but pay 24/7)        │
│   Production API: Set 1-2 minimum to avoid cold starts.        │
│                                                                       │
│ Maximum number of instances: [100]                                 │
│ ⚡ Cost safety cap. Cloud Run will not scale beyond this.         │
│   Set based on your backend capacity (DB connections, etc.)    │
│                                                                       │
│ Maximum concurrent requests per instance: [80]                    │
│ ⚡ How many simultaneous requests ONE instance handles.           │
│   ├── Low (1-10): CPU-intensive work (image processing)        │
│   ├── Medium (50-80): Typical web apps (default: 80)           │
│   ├── High (200-1000): I/O-heavy apps (many DB queries)       │
│   └── When all instances are at max → new instance created    │
│                                                                       │
│ Scaling math:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ 500 concurrent requests, concurrency = 80                   │  │
│ │ Instances needed: 500 / 80 = ~7 instances                  │  │
│ │                                                              │  │
│ │ 500 concurrent requests, concurrency = 1                    │  │
│ │ Instances needed: 500 / 1 = 500 instances (expensive!)     │  │
│ │                                                              │  │
│ │ Higher concurrency = fewer instances = lower cost!          │  │
│ │ But: Each instance shares CPU across concurrent requests.   │  │
│ │ Too high concurrency + CPU-heavy work = slow responses.    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           NETWORKING                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Ingress (who can reach your service) ──                         │
│                                                                       │
│ ● All (allow direct access from the internet)                    │
│ ○ Internal (only from same project VPC or Shared VPC)           │
│ ○ Internal and Cloud Load Balancing                              │
│                                                                       │
│ ⚡ "Internal" = Only VPC traffic + other Cloud Run/GCF services. │
│   Use for: Backend APIs that should not be publicly accessible. │
│   "Internal and CLB" = Internal + traffic from external GCLB.  │
│                                                                       │
│ ── Egress (connecting OUT from your service) ──                    │
│                                                                       │
│ Connect to a VPC for outbound traffic:                             │
│ Network: [default ▼]                                               │
│ Subnet: [cloud-run-subnet ▼]  [Create new subnet]               │
│                                                                       │
│ Traffic routing:                                                     │
│ ○ Route only requests to private IPs through the VPC            │
│ ● Route all traffic through the VPC                              │
│                                                                       │
│ ⚡ Direct VPC Egress (recommended over Serverless VPC Access):   │
│   ├── Cloud Run instances get IPs from a VPC subnet             │
│   ├── Can reach private IPs (Cloud SQL, Memorystore, GKE)     │
│   ├── Route all traffic: Egress goes through VPC (NAT, FW)    │
│   └── No separate VPC connector needed!                        │
│                                                                       │
│ ⚡ Legacy: Serverless VPC Access connector (still supported):    │
│   Connector: [vpc-connector-prod ▼]                              │
│   ├── Separate resource, /28 subnet, runs on e2-micro VMs     │
│   └── Direct VPC Egress is simpler and cheaper.                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           SECURITY                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Authentication ──                                                │
│                                                                       │
│ ○ Allow unauthenticated invocations ← Public (website/API)     │
│ ● Require authentication ← Private (internal services)          │
│                                                                       │
│ ⚡ "Require auth": Callers must provide:                            │
│   ├── Identity token (OIDC token from service account)          │
│   ├── IAM role: roles/run.invoker must be granted              │
│   ├── Service-to-service: Auto-handled with IAM                │
│   └── External: Use API Gateway or Cloud Endpoints in front   │
│                                                                       │
│ ── Service account ──                                               │
│ [sa-web-app@project.iam.gserviceaccount.com ▼]                   │
│ ⚡ The identity of your running container.                         │
│   ├── Cloud Run uses this SA to access other GCP services       │
│   ├── Example: Cloud SQL, Cloud Storage, Pub/Sub, BigQuery    │
│   ├── Default: Compute Engine default SA (too broad!)          │
│   └── Best practice: Create dedicated SA per service ✅          │
│                                                                       │
│ ── Binary Authorization ──                                         │
│ ☐ Enable                                                           │
│ ⚡ Verify container images are from trusted sources.               │
│   Enterprise feature for supply chain security.                 │
│                                                                       │
│ ── CMEK (Customer-Managed Encryption Key) ──                      │
│ ☐ Enable                                                           │
│ KMS key: [projects/.../keys/cloud-run-key]                       │
│                                                                       │
│ [CREATE]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Revisions & Traffic Splitting

```
┌─────────────────────────────────────────────────────────────────────┐
│           REVISIONS & TRAFFIC                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Every deployment creates a new REVISION (immutable):               │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Service: web-app                                             │  │
│ │                                                              │  │
│ │ Revisions:                                                   │  │
│ │ ├── web-app-00001-abc (v1.0, deployed May 1)  ← old       │  │
│ │ ├── web-app-00002-def (v1.1, deployed May 8)  ← old       │  │
│ │ ├── web-app-00003-ghi (v1.2, deployed May 15) ← current   │  │
│ │ └── web-app-00004-jkl (v2.0, deployed May 16) ← canary    │  │
│ │                                                              │  │
│ │ Traffic:                                                     │  │
│ │   web-app-00003-ghi: 90%                                   │  │
│ │   web-app-00004-jkl: 10%  ← Canary deployment!            │  │
│ │                                                              │  │
│ │ URL: https://web-app-xxxxx-xx.a.run.app                    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Traffic splitting (Console):                                        │
│ Cloud Run → web-app → Revisions → Manage Traffic                 │
│                                                                       │
│ ┌──────────────────────────────────────────────┐                   │
│ │ Revision                    │ Traffic %       │                   │
│ ├─────────────────────────────┼─────────────────┤                   │
│ │ web-app-00003-ghi (v1.2)    │ [90] %         │                   │
│ │ web-app-00004-jkl (v2.0)    │ [10] %         │                   │
│ │ + Add revision               │                │                   │
│ └──────────────────────────────┴─────────────────┘                   │
│                                                                       │
│ [Save]                                                               │
│                                                                       │
│ Canary deployment strategy:                                         │
│ 1. Deploy v2.0 → new revision created                             │
│ 2. Split: 95% v1.2, 5% v2.0 → monitor errors/latency           │
│ 3. Looks good → 50% v1.2, 50% v2.0                              │
│ 4. All clear → 0% v1.2, 100% v2.0                               │
│ 5. Problem? → 100% v1.2, 0% v2.0 (instant rollback!)           │
│                                                                       │
│ Revision URL (tag-based):                                           │
│ ├── Latest: https://web-app-xxxxx.a.run.app                    │
│ ├── Tagged: https://canary---web-app-xxxxx.a.run.app            │
│ └── ⚡ Tags let you test specific revisions without traffic.      │
│                                                                       │
│ Rollback:                                                            │
│ ├── Send 100% traffic to previous revision (instant!)           │
│ ├── No redeployment needed — old revision still exists          │
│ └── ⚡ This is why immutable revisions are powerful!              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Cloud Run Jobs

```
Console → Cloud Run → Jobs → Create Job

┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD RUN JOBS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Run containers to completion (not HTTP).                     │
│ Use for: Data processing, migrations, reports, backups.            │
│                                                                       │
│ Services vs Jobs:                                                    │
│ ├── Service: Listens for HTTP requests, runs continuously       │
│ └── Job: Runs, does work, exits. No HTTP endpoint.              │
│                                                                       │
│ Container image: [region-docker.pkg.dev/project/repo/job:v1 ▼] │
│                                                                       │
│ Job name: [data-export-job]                                        │
│ Region: [asia-south1 ▼]                                           │
│                                                                       │
│ ── Task configuration ──                                           │
│                                                                       │
│ Number of tasks: [10]                                               │
│ ⚡ How many parallel tasks (containers) to run.                    │
│   Each task runs the SAME container image.                       │
│   Use CLOUD_RUN_TASK_INDEX (0-9) to shard work.                │
│   Task 0: Process records 0-999                                 │
│   Task 1: Process records 1000-1999                             │
│   ... etc.                                                       │
│                                                                       │
│ Parallelism: [5]                                                    │
│ ⚡ How many tasks run at the same time.                             │
│   10 tasks, parallelism 5 → 2 batches of 5.                    │
│   0 = Run as many as possible simultaneously.                   │
│                                                                       │
│ Max retries per task: [3]                                          │
│ ⚡ If a task fails (non-zero exit), retry up to 3 times.          │
│                                                                       │
│ Task timeout: [600] seconds (10 minutes)                          │
│ ⚡ Max time per task. Job-level timeout also available.            │
│                                                                       │
│ CPU: [2 ▼]   Memory: [2 GiB ▼]                                  │
│                                                                       │
│ Environment variables:                                              │
│ BATCH_SIZE: 1000                                                    │
│ OUTPUT_BUCKET: gs://my-project-exports                             │
│                                                                       │
│ ⚡ Auto-injected env vars:                                          │
│   CLOUD_RUN_TASK_INDEX → 0, 1, 2, ..., 9 (task number)         │
│   CLOUD_RUN_TASK_COUNT → 10 (total tasks)                       │
│   CLOUD_RUN_TASK_ATTEMPT → 0, 1, 2, 3 (retry count)           │
│                                                                       │
│ [CREATE]                                                             │
│ Then: [EXECUTE] to run the job.                                    │
│                                                                       │
│ ── Schedule with Cloud Scheduler ──                                │
│ Cloud Scheduler → Create job:                                      │
│   Name: daily-export                                               │
│   Frequency: 0 2 * * * (2 AM daily)                              │
│   Target type: HTTP                                                │
│   URL: https://asia-south1-run.googleapis.com/apis/run.googleapis│
│        .com/v1/namespaces/PROJECT/jobs/data-export-job:run       │
│   Auth header: OAuth token                                        │
│   Service account: sa-scheduler@project.iam                      │
│                                                                       │
│ Or via gcloud:                                                      │
│ gcloud scheduler jobs create http daily-export \                  │
│   --schedule="0 2 * * *" \                                        │
│   --uri="https://..../jobs/data-export-job:run" \                │
│   --http-method=POST \                                             │
│   --oauth-service-account-email=sa-scheduler@project.iam        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

Example job code (Python):

import os

task_index = int(os.environ.get("CLOUD_RUN_TASK_INDEX", 0))
task_count = int(os.environ.get("CLOUD_RUN_TASK_COUNT", 1))
batch_size = int(os.environ.get("BATCH_SIZE", 1000))

start = task_index * batch_size
end = start + batch_size
print(f"Task {task_index}: Processing records {start} to {end}")

# Process your data here...
# Upload results to GCS...

print(f"Task {task_index} complete!")
```

---

## Part 5: Custom Domains

```
┌─────────────────────────────────────────────────────────────────────┐
│           CUSTOM DOMAINS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Option A: Cloud Run domain mapping (simple, single region):       │
│ Cloud Run → web-app → Integrations → Custom domains              │
│                                                                       │
│ [Add mapping]                                                       │
│ Domain: [app.example.com]                                          │
│ ⚡ Cloud Run verifies domain ownership + provisions SSL cert.     │
│   Add DNS CNAME: app.example.com → ghs.googlehosted.com         │
│                                                                       │
│ Option B: Global Load Balancer + Cloud Run (multi-region) ✅:     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Internet → Global HTTPS LB → Cloud Run (multi-region)     │  │
│ │                                                              │  │
│ │ Frontend: Global IP + SSL cert + custom domain              │  │
│ │ Backend: Serverless NEG → Cloud Run service                │  │
│ │                                                              │  │
│ │ Benefits:                                                    │  │
│ │ ├── Custom domain with Google-managed SSL                  │  │
│ │ ├── Cloud CDN integration                                  │  │
│ │ ├── Cloud Armor (WAF, DDoS protection)                    │  │
│ │ ├── Multi-region routing (closest region)                  │  │
│ │ └── URL rewriting, header modifications                   │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Setup (gcloud):                                                      │
│ # Create serverless NEG                                            │
│ gcloud compute network-endpoint-groups create neg-web-app \      │
│   --region=asia-south1 \                                          │
│   --network-endpoint-type=serverless \                             │
│   --cloud-run-service=web-app                                     │
│                                                                       │
│ # Create backend service                                           │
│ gcloud compute backend-services create bs-web-app \               │
│   --global \                                                        │
│   --load-balancing-scheme=EXTERNAL_MANAGED                        │
│                                                                       │
│ # Add NEG to backend                                               │
│ gcloud compute backend-services add-backend bs-web-app \         │
│   --global \                                                        │
│   --network-endpoint-group=neg-web-app \                          │
│   --network-endpoint-group-region=asia-south1                    │
│                                                                       │
│ # Create URL map + HTTPS proxy + forwarding rule                  │
│ # ... (similar to Ch14 Load Balancing setup)                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform

```hcl
# Cloud Run Service
resource "google_cloud_run_v2_service" "web_app" {
  name     = "web-app"
  location = "asia-south1"

  template {
    service_account = google_service_account.web_app.email

    scaling {
      min_instance_count = 1    # Keep 1 warm (no cold start)
      max_instance_count = 100
    }

    containers {
      image = "asia-south1-docker.pkg.dev/my-project/images/web-app:v1.2"

      ports {
        container_port = 8080
      }

      resources {
        limits = {
          cpu    = "2"
          memory = "1Gi"
        }
        cpu_idle          = true   # Request-based CPU allocation
        startup_cpu_boost = true
      }

      env {
        name  = "NODE_ENV"
        value = "production"
      }

      env {
        name  = "API_KEY"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.api_key.secret_id
            version = "latest"
          }
        }
      }

      startup_probe {
        http_get {
          path = "/health"
          port = 8080
        }
        initial_delay_seconds = 0
        period_seconds        = 10
        failure_threshold     = 3
      }

      liveness_probe {
        http_get {
          path = "/health"
          port = 8080
        }
        period_seconds = 30
      }
    }

    # VPC access for private resources
    vpc_access {
      network_interfaces {
        network    = google_compute_network.main.id
        subnetwork = google_compute_subnetwork.cloud_run.id
      }
      egress = "PRIVATE_RANGES_ONLY"
    }

    execution_environment = "EXECUTION_ENVIRONMENT_GEN2"

    max_instance_request_concurrency = 80
    timeout                          = "300s"
  }

  traffic {
    type    = "TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST"
    percent = 100
  }
}

# Allow unauthenticated access (public)
resource "google_cloud_run_v2_service_iam_member" "public" {
  location = google_cloud_run_v2_service.web_app.location
  name     = google_cloud_run_v2_service.web_app.name
  role     = "roles/run.invoker"
  member   = "allUsers"
}

# Service Account
resource "google_service_account" "web_app" {
  account_id   = "sa-web-app"
  display_name = "Web App Service Account"
}

# Grant SA access to Cloud SQL
resource "google_project_iam_member" "web_app_sql" {
  project = var.project_id
  role    = "roles/cloudsql.client"
  member  = "serviceAccount:${google_service_account.web_app.email}"
}

# Grant SA access to Secret Manager
resource "google_project_iam_member" "web_app_secrets" {
  project = var.project_id
  role    = "roles/secretmanager.secretAccessor"
  member  = "serviceAccount:${google_service_account.web_app.email}"
}

# Cloud Run Service (internal — backend API)
resource "google_cloud_run_v2_service" "api" {
  name     = "api-backend"
  location = "asia-south1"
  ingress  = "INGRESS_TRAFFIC_INTERNAL_ONLY"

  template {
    service_account = google_service_account.api.email

    scaling {
      min_instance_count = 2
      max_instance_count = 50
    }

    containers {
      image = "asia-south1-docker.pkg.dev/my-project/images/api:v2.0"
      ports {
        container_port = 8080
      }
      resources {
        limits = {
          cpu    = "2"
          memory = "2Gi"
        }
      }
    }
  }
}

# Cloud Run Job
resource "google_cloud_run_v2_job" "data_export" {
  name     = "data-export-job"
  location = "asia-south1"

  template {
    task_count  = 10
    parallelism = 5

    template {
      service_account = google_service_account.job.email
      timeout         = "600s"
      max_retries     = 3

      containers {
        image = "asia-south1-docker.pkg.dev/my-project/images/exporter:v1"

        resources {
          limits = {
            cpu    = "2"
            memory = "2Gi"
          }
        }

        env {
          name  = "BATCH_SIZE"
          value = "1000"
        }

        env {
          name  = "OUTPUT_BUCKET"
          value = "gs://my-project-exports"
        }
      }
    }
  }
}

# Schedule the job
resource "google_cloud_scheduler_job" "daily_export" {
  name     = "daily-export"
  schedule = "0 2 * * *"
  time_zone = "Asia/Kolkata"

  http_target {
    http_method = "POST"
    uri = "https://asia-south1-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/${var.project_id}/jobs/data-export-job:run"

    oauth_token {
      service_account_email = google_service_account.scheduler.email
    }
  }
}

# Traffic splitting (canary)
resource "google_cloud_run_v2_service" "web_canary" {
  name     = "web-app"
  location = "asia-south1"

  template {
    revision = "web-app-v2"
    containers {
      image = "asia-south1-docker.pkg.dev/my-project/images/web-app:v2.0"
      ports {
        container_port = 8080
      }
    }
  }

  traffic {
    type     = "TRAFFIC_TARGET_ALLOCATION_TYPE_REVISION"
    revision = "web-app-v1"
    percent  = 90
  }

  traffic {
    type     = "TRAFFIC_TARGET_ALLOCATION_TYPE_REVISION"
    revision = "web-app-v2"
    percent  = 10
    tag      = "canary"
  }
}
```

---

## Part 7: gcloud CLI Reference

```bash
# ═══ Cloud Run Services ═══

# Deploy a service
gcloud run deploy web-app \
  --image=asia-south1-docker.pkg.dev/my-project/images/web-app:v1.2 \
  --region=asia-south1 \
  --platform=managed \
  --port=8080 \
  --cpu=2 \
  --memory=1Gi \
  --min-instances=1 \
  --max-instances=100 \
  --concurrency=80 \
  --timeout=300 \
  --execution-environment=gen2 \
  --cpu-boost \
  --service-account=sa-web-app@my-project.iam.gserviceaccount.com \
  --set-env-vars="NODE_ENV=production,LOG_LEVEL=info" \
  --set-secrets="DB_PASSWORD=db-password:latest" \
  --allow-unauthenticated

# Deploy (internal only — no public access)
gcloud run deploy api-backend \
  --image=asia-south1-docker.pkg.dev/my-project/images/api:v2.0 \
  --region=asia-south1 \
  --ingress=internal \
  --no-allow-unauthenticated \
  --min-instances=2

# Deploy with VPC egress (Direct VPC)
gcloud run deploy web-app \
  --image=asia-south1-docker.pkg.dev/my-project/images/web-app:v1.2 \
  --region=asia-south1 \
  --network=default \
  --subnet=cloud-run-subnet \
  --vpc-egress=private-ranges-only

# Deploy with always-on CPU
gcloud run deploy worker \
  --image=...worker:v1 \
  --no-cpu-throttling \
  --min-instances=1

# List services
gcloud run services list --region=asia-south1

# Describe service
gcloud run services describe web-app --region=asia-south1

# Get URL
gcloud run services describe web-app \
  --region=asia-south1 \
  --format="value(status.url)"

# ═══ Traffic splitting ═══

# Send 10% to latest, 90% to previous
gcloud run services update-traffic web-app \
  --region=asia-south1 \
  --to-revisions=web-app-00003-ghi=90,web-app-00004-jkl=10

# Send 100% to latest (after canary is good)
gcloud run services update-traffic web-app \
  --region=asia-south1 \
  --to-latest

# Rollback: Send 100% to old revision
gcloud run services update-traffic web-app \
  --region=asia-south1 \
  --to-revisions=web-app-00003-ghi=100

# Tag a revision for testing
gcloud run services update-traffic web-app \
  --region=asia-south1 \
  --set-tags=canary=web-app-00004-jkl
# Access: https://canary---web-app-xxxxx.a.run.app

# ═══ Revisions ═══

# List revisions
gcloud run revisions list --service=web-app --region=asia-south1

# Delete old revision
gcloud run revisions delete web-app-00001-abc --region=asia-south1

# ═══ Cloud Run Jobs ═══

# Create job
gcloud run jobs create data-export-job \
  --image=asia-south1-docker.pkg.dev/my-project/images/exporter:v1 \
  --region=asia-south1 \
  --tasks=10 \
  --parallelism=5 \
  --max-retries=3 \
  --task-timeout=600 \
  --cpu=2 \
  --memory=2Gi \
  --set-env-vars="BATCH_SIZE=1000,OUTPUT_BUCKET=gs://exports"

# Execute job
gcloud run jobs execute data-export-job --region=asia-south1

# Execute with overrides
gcloud run jobs execute data-export-job \
  --region=asia-south1 \
  --tasks=20 \
  --update-env-vars="BATCH_SIZE=500"

# List job executions
gcloud run jobs executions list \
  --job=data-export-job \
  --region=asia-south1

# View logs
gcloud run jobs executions describe exec-xxxxx \
  --job=data-export-job \
  --region=asia-south1

# ═══ Logs ═══

# Stream service logs
gcloud run services logs read web-app --region=asia-south1 --limit=50

# Tail logs (real-time)
gcloud run services logs tail web-app --region=asia-south1

# ═══ Domain mapping ═══

# Map custom domain
gcloud run domain-mappings create \
  --service=web-app \
  --domain=app.example.com \
  --region=asia-south1

# ═══ IAM ═══

# Grant invoke permission
gcloud run services add-iam-policy-binding web-app \
  --region=asia-south1 \
  --member="serviceAccount:sa-frontend@my-project.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

# Delete service
gcloud run services delete web-app --region=asia-south1
```

---

## Part 8: Real-World Patterns

### Startup

```
Setup: Simple web app on Cloud Run

Service: web-app (public, scale-to-zero)
├── Image: Node.js/Python/Go app from Artifact Registry
├── CPU: 1 vCPU, Memory: 512 MiB
├── CPU allocation: Request-based (cheapest)
├── Min instances: 0 (scale to zero!)
├── Max instances: 10
├── Concurrency: 80
├── Port: 8080
├── Auth: Allow unauthenticated
├── Custom domain: app.startup.com (domain mapping)
├── Secrets: DB password from Secret Manager
├── VPC egress: Direct VPC (→ Cloud SQL private IP)
└── CI/CD: Cloud Build on git push → auto-deploy

Service: api-backend (internal)
├── Ingress: Internal only
├── Auth: Require authentication
├── Service account: sa-api → Cloud SQL, Storage
└── Called by: web-app (service-to-service with IAM)

Cost: ~$5-30/month (scale-to-zero, free tier covers most)
```

### Mid-Size

```
Architecture: Multi-service Cloud Run

Services:
├── web-app (public)
│   ├── 2 vCPU, 1 GiB, Gen 2
│   ├── Min 2, Max 50 (always warm)
│   ├── Concurrency: 100
│   ├── Behind Global HTTPS LB + Cloud Armor
│   ├── Custom domain: app.company.com
│   └── Canary deploys: 5% → 25% → 100%
│
├── api-gateway (internal + LB)
│   ├── 2 vCPU, 1 GiB
│   ├── Min 2, Max 30
│   ├── Auth: Require auth (IAM invoker)
│   └── Routes to microservices
│
├── user-service (internal)
│   ├── 1 vCPU, 512 MiB
│   ├── Cloud SQL (PostgreSQL) via VPC
│   └── IRSA: Cloud SQL Client + Secret Accessor
│
├── notification-service (internal)
│   ├── Pub/Sub push subscription triggers this
│   ├── Sends emails (SendGrid), push notifications
│   └── Always-on CPU (processes async)
│
└── worker (always-on CPU)
    ├── Pub/Sub pull, long-running processing
    ├── Min 1, Max 20
    └── No HTTP ingress needed

Jobs:
├── daily-report (Cloud Scheduler, 2 AM daily)
│   Tasks: 1, timeout: 30 min
│   Generates PDF reports → Cloud Storage
│
└── data-migration (manual execution)
    Tasks: 20, parallelism: 10
    Processes millions of records in parallel

CI/CD:
├── GitHub → Cloud Build trigger on push to main
├── Build → Test → Push to Artifact Registry
├── Deploy with traffic split: 5% canary
├── Monitor 30 min → promote to 100%
└── Rollback: One command to revert traffic

Cost: $200-800/month
```

### Enterprise

```
Multi-region Cloud Run platform:

Region 1 (asia-south1):
├── web-app, api-gateway, 8 microservices
├── All behind Global HTTPS LB
├── Cloud Armor: WAF + rate limiting + geo-blocking
├── Cloud CDN: Cache static responses
└── Min instances: 5-10 per service (warm)

Region 2 (europe-west1):
├── Same services deployed (identical)
├── Global LB routes to closest region
└── Active-active: Both regions serve traffic

Global Load Balancer:
├── app.company.com → Global IP
├── SSL: Google-managed certificate
├── Routing: Nearest region (latency-based)
├── Failover: Auto-redirect if region unhealthy
├── Cloud Armor: OWASP top 10 rules
└── Cloud CDN: Cache API responses (where safe)

Security:
├── All internal services: Require auth (IAM)
├── Dedicated service accounts per service (least privilege)
├── Binary Authorization: Only signed images from CI/CD
├── VPC Service Controls: Restrict API access perimeter
├── CMEK: Customer-managed encryption keys
├── Org Policy: constraints/run.allowedIngress = internal-and-cloud-load-balancing
├── Secret Manager: All secrets, auto-rotation
├── Container scanning: Artifact Registry vulnerability scanning
└── Audit: Cloud Audit Logs → BigQuery → SIEM

Deployment:
├── GitOps: Config repo with service YAML definitions
├── Cloud Build: Multi-stage (build → test → scan → deploy)
├── Canary: 1% → 5% → 25% → 50% → 100% (automated)
├── Rollback: Automated on error rate spike (Cloud Monitoring alert)
└── Multi-region deploy: Deploy region 1 → verify → deploy region 2

Monitoring:
├── Cloud Monitoring: Latency, error rate, instance count
├── SLOs: 99.9% availability, p99 < 500ms
├── Alerts: PagerDuty on SLO burn rate
├── Logging: Structured JSON logs → Cloud Logging → BigQuery
├── Tracing: Cloud Trace (distributed tracing across services)
└── Dashboards: Per-service Grafana dashboards

Cost: $3,000-15,000/month (multi-region, always-warm)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Fully managed container platform |
| Unit | Container (any language, any runtime) |
| Scale to zero | Yes (min instances = 0) |
| Max instances | 1000 (can request increase) |
| Concurrency | 1-1000 requests per instance |
| CPU | 1-8 vCPU |
| Memory | 128 MiB - 32 GiB |
| Timeout | Up to 60 minutes |
| CPU modes | Request-based (cheap) or always-on (background work) |
| Execution env | Gen 1 (gVisor) or Gen 2 (full Linux) |
| Traffic splitting | Yes (canary, A/B testing) |
| Jobs | Run-to-completion tasks with parallelism |
| Custom domains | Domain mapping or Global HTTPS LB |
| Free tier | 2M requests, 360K vCPU-seconds/month |
| AWS equivalent | App Runner or ECS Fargate |
| Azure equivalent | Container Apps |

---

## Console Walkthrough: Managing & Deleting Services

```
┌─────────────────────────────────────────────────────────────────────┐
│           EDITING AN EXISTING SERVICE                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Cloud Run → click on service name → Edit & Deploy      │
│ New Revision                                                        │
│                                                                       │
│ What you can change:                                                │
│ ├── Container image (deploy new version)                         │
│ ├── Environment variables (add/edit/remove)                     │
│ ├── Secrets (add new, change version)                            │
│ ├── CPU & Memory (scale up/down resources)                      │
│ ├── Concurrency (requests per instance)                         │
│ ├── Min/Max instances (scaling behavior)                        │
│ ├── CPU allocation mode (request-based ↔ always-on)           │
│ ├── Timeout, health checks                                      │
│ ├── Service account                                              │
│ └── VPC/networking settings                                     │
│                                                                       │
│ Steps:                                                               │
│ 1. Click "Edit & Deploy New Revision" at the top                │
│ 2. Modify settings (e.g., change memory from 512 MiB → 1 GiB) │
│ 3. Scroll down → Review changes                                 │
│ 4. Click [Deploy] → Creates a new revision                      │
│ 5. Traffic automatically shifts to new revision (unless split)  │
│                                                                       │
│ ⚡ Every change creates a new immutable revision.                  │
│   You can always roll back to a previous revision.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           DELETING A CLOUD RUN SERVICE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Cloud Run → select checkbox next to service → Delete  │
│                                                                       │
│ Steps:                                                               │
│ 1. Go to Cloud Run in Console                                    │
│ 2. Check the box next to the service you want to delete          │
│ 3. Click [DELETE] button at the top                               │
│ 4. Confirm by typing the service name                            │
│ 5. Click [Delete]                                                  │
│                                                                       │
│ Or from inside the service:                                        │
│ 1. Click on the service name to open it                          │
│ 2. Click the ⋮ (three dots) menu at the top right                │
│ 3. Select "Delete"                                                │
│ 4. Confirm deletion                                                │
│                                                                       │
│ ⚡ What happens when you delete:                                    │
│   ├── All revisions are deleted                                  │
│   ├── The HTTPS URL stops working immediately                   │
│   ├── Custom domain mappings are removed                        │
│   ├── IAM bindings for the service are removed                  │
│   └── ⚠️ This is permanent! No undo!                              │
│                                                                       │
│ ⚡ Things NOT deleted automatically:                                │
│   ├── Container images in Artifact Registry (delete separately)│
│   ├── Secrets in Secret Manager                                  │
│   ├── Service account used by the service                       │
│   └── VPC connectors or subnets                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           DELETING A CLOUD RUN JOB                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → Cloud Run → Jobs tab → select job → Delete             │
│                                                                       │
│ Steps:                                                               │
│ 1. Go to Cloud Run in Console                                    │
│ 2. Click the "Jobs" tab (next to "Services" tab)                │
│ 3. Check the box next to the job you want to delete              │
│ 4. Click [DELETE] button at the top                               │
│ 5. Confirm by typing the job name → Click [Delete]              │
│                                                                       │
│ ⚡ Deleting a job also deletes all its execution history.          │
│   If a Cloud Scheduler job triggers it, delete that too.        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           VIEWING LOGS & METRICS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ═══ Logs ═══                                                        │
│ Console → Cloud Run → click service name → Logs tab              │
│                                                                       │
│ What you see:                                                       │
│ ├── All stdout/stderr from your container                        │
│ ├── Request logs (method, path, status code, latency)           │
│ ├── System logs (instance startup, scaling events)              │
│ └── Filter by severity: Default, Info, Warning, Error           │
│                                                                       │
│ Filters:                                                             │
│ ├── Severity: [All ▼] [Error ▼] [Warning ▼]                   │
│ ├── Time range: Last 1 hour, 6 hours, 1 day, custom           │
│ └── Search: Free text search across log entries                 │
│                                                                       │
│ ⚡ Click "View in Logs Explorer" for advanced queries.             │
│   Logs Explorer query example:                                   │
│   resource.type="cloud_run_revision"                              │
│   resource.labels.service_name="web-app"                         │
│   severity>=ERROR                                                 │
│                                                                       │
│ ═══ Metrics ═══                                                     │
│ Console → Cloud Run → click service name → Metrics tab           │
│                                                                       │
│ Built-in charts:                                                    │
│ ├── Request count (per second, by response code)                │
│ ├── Request latency (p50, p95, p99)                             │
│ ├── Container instance count (current running instances)       │
│ ├── CPU utilization (% of allocated CPU)                        │
│ ├── Memory utilization (% of allocated memory)                  │
│ ├── Container startup latency                                    │
│ └── Billable instance time                                       │
│                                                                       │
│ ⚡ Spot check:                                                      │
│   High latency? → Check CPU util (may need more CPU/instances)│
│   OOM errors? → Increase memory allocation                      │
│   503 errors? → Check max instances limit / concurrency         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Cloud Run + Cloud SQL Connection

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD RUN → CLOUD SQL                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud Run connects to Cloud SQL using the Cloud SQL Auth Proxy,  │
│ which runs as a sidecar container. The proxy handles:              │
│ ├── Secure connection (SSL/TLS) without managing certs          │
│ ├── IAM-based authentication (no passwords needed!)             │
│ └── Connection via Unix socket (fast, local-like)               │
│                                                                       │
│ How it works:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Cloud Run Instance                                           │  │
│ │ ┌──────────────┐    Unix socket    ┌──────────────────────┐ │  │
│ │ │ Your App     │ ──────────────── │ Cloud SQL Auth Proxy│ │  │
│ │ │ (main cont.) │  /cloudsql/...   │ (sidecar container) │ │  │
│ │ └──────────────┘                   └────────┬─────────────┘ │  │
│ │                                              │               │  │
│ └──────────────────────────────────────────────│───────────────┘  │
│                                                │                    │
│                                    Secure tunnel (IAM auth)       │
│                                                │                    │
│                                    ┌───────────▼──────────────┐   │
│                                    │ Cloud SQL (PostgreSQL/  │   │
│                                    │ MySQL/SQL Server)        │   │
│                                    └──────────────────────────┘   │
│                                                                       │
│ ═══ Setup Steps ═══                                                 │
│                                                                       │
│ Step 1: Grant IAM role to Cloud Run's service account            │
│   Role: roles/cloudsql.client                                     │
│   This lets the proxy connect to Cloud SQL.                      │
│                                                                       │
│ Step 2: Add Cloud SQL connection in Console                       │
│   Cloud Run → Edit service → Connections tab                     │
│   Cloud SQL connections: [+ Add connection]                      │
│   Select instance: [my-project:asia-south1:my-db ▼]             │
│                                                                       │
│   ⚡ This auto-adds the Cloud SQL Auth Proxy as a sidecar.       │
│     The proxy mounts a Unix socket at:                           │
│     /cloudsql/PROJECT:REGION:INSTANCE                            │
│                                                                       │
│ Step 3: Configure your app to connect via Unix socket            │
│   Set env var: DB_SOCKET_PATH=/cloudsql/my-project:asia-south1:my-db │
│                                                                       │
│ Or via gcloud:                                                      │
│ gcloud run deploy web-app \                                       │
│   --add-cloudsql-instances=my-project:asia-south1:my-db \       │
│   --set-env-vars="DB_SOCKET_PATH=/cloudsql/my-project:asia-south1:my-db" │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

Example connection code (Python with SQLAlchemy):

```python
import os
import sqlalchemy

def connect_to_cloud_sql():
    """Connect to Cloud SQL via Unix socket (Auth Proxy sidecar)."""

    db_user = os.environ.get("DB_USER", "app_user")
    db_pass = os.environ.get("DB_PASSWORD")        # From Secret Manager
    db_name = os.environ.get("DB_NAME", "mydb")
    db_socket = os.environ.get(
        "DB_SOCKET_PATH",
        "/cloudsql/my-project:asia-south1:my-db"
    )

    # PostgreSQL connection via Unix socket
    engine = sqlalchemy.create_engine(
        sqlalchemy.engine.url.URL.create(
            drivername="postgresql+pg8000",
            username=db_user,
            password=db_pass,
            database=db_name,
            query={"unix_sock": f"{db_socket}/.s.PGSQL.5432"},
        ),
        pool_size=5,       # Keep 5 connections open
        max_overflow=2,    # Allow 2 extra under load
        pool_timeout=30,   # Wait 30s for available connection
        pool_recycle=1800, # Recycle connections every 30 min
    )
    return engine

# Usage in a Flask/FastAPI app:
engine = connect_to_cloud_sql()

# For MySQL, change drivername and socket path:
# drivername="mysql+pymysql"
# query={"unix_socket": f"{db_socket}"}
```

```
⚡ Best practices for Cloud Run + Cloud SQL:
  ├── Always use connection pooling (pool_size=5 is a good start)
  ├── Set max instances on Cloud Run carefully:
  │   Max instances × pool_size ≤ Cloud SQL max connections
  │   Example: 50 instances × 5 pool = 250 connections
  ├── Use Secret Manager for DB password (not env vars)
  ├── Use a dedicated service account with roles/cloudsql.client
  └── For private IP: Use Direct VPC Egress + private IP
      (skips the Auth Proxy, connects directly via VPC)
```

---

## Pub/Sub Push to Cloud Run

```
┌─────────────────────────────────────────────────────────────────────┐
│           PUB/SUB PUSH → CLOUD RUN                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pub/Sub can push messages directly to a Cloud Run service.        │
│ Every message triggers an HTTP POST to your service endpoint.     │
│                                                                       │
│ ┌────────────┐     ┌────────────┐     ┌──────────────────────┐   │
│ │ Publisher   │────│ Pub/Sub    │────│ Cloud Run Service   │   │
│ │ (any app)   │    │ Topic +    │    │ POST /               │   │
│ │             │    │ Push Sub   │    │ (handles message)    │   │
│ └────────────┘     └────────────┘     └──────────────────────┘   │
│                                                                       │
│ ═══ Setup Steps ═══                                                 │
│                                                                       │
│ Step 1: Deploy your Cloud Run service                              │
│   ├── Must accept POST requests at / (or a custom path)        │
│   ├── Auth: "Require authentication" (recommended)              │
│   └── Note the service URL                                       │
│                                                                       │
│ Step 2: Create a Pub/Sub topic                                     │
│   Console → Pub/Sub → Create topic                                │
│   Topic ID: [order-events]                                        │
│                                                                       │
│ Step 3: Create a push subscription                                 │
│   Console → Pub/Sub → order-events → Create subscription         │
│   Subscription ID: [order-events-push-to-run]                    │
│   Delivery type: ● Push                                           │
│   Endpoint URL: [https://order-processor-xxxxx.a.run.app]       │
│                                                                       │
│   ☑ Enable authentication                                        │
│   Service account: [sa-pubsub@project.iam.gserviceaccount.com] │
│   ⚡ This SA needs roles/run.invoker on the Cloud Run service.   │
│                                                                       │
│ Step 4: Grant Pub/Sub SA the invoker role on Cloud Run            │
│   gcloud run services add-iam-policy-binding order-processor \  │
│     --region=asia-south1 \                                        │
│     --member="serviceAccount:sa-pubsub@project.iam..." \       │
│     --role="roles/run.invoker"                                    │
│                                                                       │
│ ⚡ How the push message looks:                                      │
│   POST / HTTP/1.1                                                 │
│   Content-Type: application/json                                  │
│   Authorization: Bearer <OIDC token>                              │
│                                                                       │
│   {                                                                  │
│     "message": {                                                    │
│       "data": "<base64 encoded message>",                       │
│       "messageId": "123456789",                                  │
│       "publishTime": "2026-05-22T10:00:00Z",                   │
│       "attributes": {"key": "value"}                            │
│     },                                                               │
│     "subscription": "projects/my-project/subscriptions/..."     │
│   }                                                                  │
│                                                                       │
│ ⚡ Response codes:                                                   │
│   ├── 200 or 204 → Message acknowledged (won't be redelivered) │
│   ├── 102, 2xx → Success, message acknowledged                  │
│   └── Any other code → Pub/Sub retries delivery                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

Example handler code (Python with Flask):

```python
import base64
import json
import os
from flask import Flask, request

app = Flask(__name__)

@app.route("/", methods=["POST"])
def handle_pubsub_message():
    """Handle incoming Pub/Sub push message."""

    envelope = request.get_json()
    if not envelope:
        return "Bad Request: no JSON body", 400

    if "message" not in envelope:
        return "Bad Request: missing message", 400

    # Decode the message data (base64 encoded)
    message = envelope["message"]
    if "data" in message:
        data = base64.b64decode(message["data"]).decode("utf-8")
        payload = json.loads(data)
    else:
        payload = {}

    # Get message attributes (optional metadata)
    attributes = message.get("attributes", {})
    message_id = message.get("messageId", "unknown")

    print(f"Received message {message_id}: {payload}")
    print(f"Attributes: {attributes}")

    # ── Process the message here ──
    # Example: order-events topic
    event_type = attributes.get("event_type", "unknown")
    if event_type == "order_created":
        process_new_order(payload)
    elif event_type == "order_cancelled":
        process_cancellation(payload)
    else:
        print(f"Unknown event type: {event_type}")

    # Return 200 to acknowledge the message
    # Any non-2xx response → Pub/Sub will retry!
    return "OK", 200


def process_new_order(order):
    print(f"Processing new order: {order.get('order_id')}")
    # Send confirmation email, update inventory, etc.


def process_cancellation(order):
    print(f"Cancelling order: {order.get('order_id')}")
    # Refund payment, restock items, etc.


if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port)
```

---

## Troubleshooting Common Issues

```
┌─────────────────────────────────────────────────────────────────────┐
│           TROUBLESHOOTING CLOUD RUN                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ═══ Issue 1: Container fails to start ═══                          │
│                                                                       │
│ Symptoms:                                                            │
│   "Container failed to start" or revision stuck in "Deploying"   │
│                                                                       │
│ Common causes:                                                      │
│ ┌──────────────────────┬──────────────────────────────────────┐   │
│ │ Cause                │ Fix                                   │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Wrong port           │ Your app must listen on the port     │   │
│ │                      │ specified in Cloud Run (default 8080)│   │
│ │                      │ Use $PORT env var: auto-injected.   │   │
│ │                      │ app.listen(process.env.PORT || 8080)│   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ App crashes at start │ Check logs for stack traces.         │   │
│ │                      │ Missing env vars? Missing secrets?  │   │
│ │                      │ Test locally: docker run -p 8080:8080│   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Health check fails   │ Startup probe path returns non-200? │   │
│ │                      │ Increase failure threshold or remove│   │
│ │                      │ startup probe for initial debugging.│   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Out of memory (OOM)  │ App needs more memory than allocated.│   │
│ │                      │ Increase memory in service settings.│   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Image pull error     │ Check image URL. Is Artifact Registry│   │
│ │                      │ in same project? SA has permission? │   │
│ └──────────────────────┴──────────────────────────────────────┘   │
│                                                                       │
│ Debug steps:                                                        │
│ 1. Console → Cloud Run → service → Logs tab                      │
│ 2. Look for error messages at startup                              │
│ 3. Test container locally:                                         │
│    docker run -p 8080:8080 -e PORT=8080 your-image:tag          │
│ 4. Verify port: curl http://localhost:8080/health                 │
│                                                                       │
│ ═══ Issue 2: Cold start optimization ═══                           │
│                                                                       │
│ Symptoms:                                                            │
│   First request after idle period is slow (1-10+ seconds)        │
│                                                                       │
│ What causes cold starts:                                            │
│ ├── Min instances = 0 → scale to zero when idle                 │
│ ├── Cloud Run must: pull image → start container → app ready   │
│ └── Large images or slow app startup = longer cold starts       │
│                                                                       │
│ Optimization strategies:                                            │
│ ┌──────────────────────┬──────────────────────────────────────┐   │
│ │ Strategy             │ Impact                                │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Min instances ≥ 1    │ ✅ Eliminates cold starts (best fix) │   │
│ │                      │ Cost: Pay for idle instance 24/7    │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Startup CPU boost    │ ✅ Enable in settings. Gives extra   │   │
│ │                      │ CPU during startup (free!).         │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Smaller images       │ ✅ Use slim/distroless base images.  │   │
│ │                      │ Alpine: ~5 MB vs Ubuntu: ~70 MB.   │   │
│ │                      │ Multi-stage builds strip build deps.│   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Lazy initialization  │ ✅ Don't load everything at startup. │   │
│ │                      │ Connect to DB on first request.    │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Gen 2 execution env  │ ✅ Faster startup than Gen 1.        │   │
│ └──────────────────────┴──────────────────────────────────────┘   │
│                                                                       │
│ ═══ Issue 3: 503 errors under load ═══                             │
│                                                                       │
│ Symptoms:                                                            │
│   "503 Service Unavailable" when traffic spikes                  │
│                                                                       │
│ Common causes:                                                      │
│ ┌──────────────────────┬──────────────────────────────────────┐   │
│ │ Cause                │ Fix                                   │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Max instances hit    │ Increase max instances limit.         │   │
│ │                      │ Default: 100. Check current usage.  │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Concurrency too low  │ If concurrency = 1, Cloud Run needs │   │
│ │                      │ 1 instance per request. Set higher  │   │
│ │                      │ (50-80) for I/O-bound apps.         │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Scaling too slow     │ Set min instances ≥ 1 to have warm  │   │
│ │                      │ instances ready. Enable CPU boost.  │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ App is crashing      │ Check logs for errors/OOM. App might│   │
│ │                      │ crash under load → instances restart │   │
│ │                      │ → fewer available to serve traffic. │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Backend overloaded   │ Cloud SQL / external API can't keep │   │
│ │                      │ up. Add connection pooling, retries,│   │
│ │                      │ or circuit breakers.                │   │
│ └──────────────────────┴──────────────────────────────────────┘   │
│                                                                       │
│ ═══ Issue 4: Permission denied when invoking ═══                   │
│                                                                       │
│ Symptoms:                                                            │
│   "403 Forbidden" or "Your client does not have permission"      │
│                                                                       │
│ Common causes & fixes:                                              │
│ ┌──────────────────────┬──────────────────────────────────────┐   │
│ │ Cause                │ Fix                                   │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Service requires     │ If public: redeploy with             │   │
│ │ auth but caller      │ --allow-unauthenticated.             │   │
│ │ sends no token       │ If internal: caller must send OIDC  │   │
│ │                      │ identity token in Authorization hdr.│   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Missing IAM role     │ Grant roles/run.invoker to the      │   │
│ │                      │ caller's service account or user.   │   │
│ │                      │ gcloud run services                 │   │
│ │                      │   add-iam-policy-binding SERVICE \  │   │
│ │                      │   --member="serviceAccount:SA" \   │   │
│ │                      │   --role="roles/run.invoker"        │   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Wrong token type     │ Must use identity token (OIDC),     │   │
│ │                      │ NOT access token (OAuth2).           │   │
│ │                      │ Common mistake with gcloud/curl.    │   │
│ │                      │ Correct:                             │   │
│ │                      │ curl -H "Authorization: Bearer      │   │
│ │                      │ $(gcloud auth print-identity-token)"│   │
│ ├──────────────────────┼──────────────────────────────────────┤   │
│ │ Ingress = internal   │ Service set to internal-only but    │   │
│ │ but calling from     │ caller is on public internet.        │   │
│ │ public internet      │ Change ingress to "All" or call    │   │
│ │                      │ from within VPC / other GCP service.│   │
│ └──────────────────────┴──────────────────────────────────────┘   │
│                                                                       │
│ Quick debug checklist:                                              │
│ 1. Is the service set to "Allow unauthenticated"?                │
│ 2. Does the caller have roles/run.invoker?                        │
│ 3. Is the caller sending an identity token (not access token)?   │
│ 4. Is ingress set correctly (all vs internal)?                    │
│ 5. Check: Console → Cloud Run → service → Security tab          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover Google Kubernetes Engine (GKE) — fully managed Kubernetes on GCP.

→ Next: [Chapter 18: GKE - Google Kubernetes Engine](18-gke.md)

---

*Last Updated: May 2026*
