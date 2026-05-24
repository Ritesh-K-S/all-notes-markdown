# Chapter 18: Elastic Beanstalk

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Elastic Beanstalk Fundamentals](#part-1-elastic-beanstalk-fundamentals)
- [Part 2: Creating an Application (Full Console Walkthrough)](#part-2-creating-an-application-full-console-walkthrough)
- [Part 3: Environment Configuration Deep Dive](#part-3-environment-configuration-deep-dive)
- [Part 4: Deployment Policies (Visual Comparison)](#part-4-deployment-policies-visual-comparison)
- [Part 5: .ebextensions & Platform Hooks](#part-5-ebextensions--platform-hooks)
- [Part 6: Terraform](#part-6-terraform)
- [Part 7: eb CLI Reference](#part-7-eb-cli-reference)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Elastic Beanstalk? When Should I Use It?

> **Real-World Analogy:** Beanstalk is like moving into a fully furnished apartment. You bring your clothes (code) and everything else — furniture, utilities, internet (servers, load balancers, auto-scaling) — is already set up. You can still rearrange the furniture if you want, but you don't have to.

**Why does this matter?** If you're a developer who just wants to deploy an app without learning VPCs, security groups, ASGs, and ALBs individually, Beanstalk lets you skip all that and get running in minutes. It uses the same AWS services under the hood, but configures them for you.

AWS Elastic Beanstalk is a Platform-as-a-Service (PaaS) that handles infrastructure provisioning, load balancing, auto-scaling, and deployment — you just upload your code. It uses EC2, ALB, ASG, RDS, and other AWS services under the hood, but manages them for you.

```
What you'll learn:
├── Elastic Beanstalk Fundamentals
│   ├── What & why (PaaS, managed infra)
│   ├── EB vs ECS vs EKS vs Lambda
│   └── Applications, Environments, Versions
├── Creating an Application (Full Console Walkthrough)
│   ├── Application setup
│   ├── Environment tier (Web server vs Worker)
│   ├── Platform (Node.js, Python, Java, Docker, etc.)
│   ├── Application code (upload or sample)
│   └── Presets (Single instance, HA, Custom)
├── Environment Configuration (Deep Dive)
│   ├── Instances (type, size, key pair, security groups)
│   ├── Capacity (ASG, scaling triggers, AZs)
│   ├── Load balancer (ALB/NLB/Classic, listeners, health checks)
│   ├── Rolling updates & deployments
│   ├── Security (IAM roles, service role)
│   ├── Monitoring (health reporting, CloudWatch)
│   ├── Managed updates (auto-patching)
│   ├── Notifications (SNS)
│   ├── Network (VPC, subnets)
│   ├── Database (RDS — coupled or decoupled)
│   └── Software (.env, proxy, logs)
├── Deployment Policies
│   ├── All at once
│   ├── Rolling
│   ├── Rolling with additional batch
│   ├── Immutable
│   ├── Traffic splitting (canary)
│   └── Blue/Green (swap URLs)
├── .ebextensions & Platform hooks
├── eb CLI
├── Terraform
└── Real-world patterns
```

---

## Part 1: Elastic Beanstalk Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           ELASTIC BEANSTALK CONCEPT                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Elastic Beanstalk?                                          │
│ ├── Upload your code → EB does the rest                          │
│ ├── Provisions: EC2, ALB, ASG, Security Groups, CloudWatch      │
│ ├── Manages: Deployment, scaling, patching, health monitoring   │
│ ├── You still have FULL ACCESS to underlying AWS resources      │
│ └── Free (you only pay for the underlying resources)            │
│                                                                       │
│ Think of it as:                                                      │
│ "Give me your app code. I'll create servers, load balancers,     │
│  auto-scaling, and deploy it. You can customize everything."     │
│                                                                       │
│ Hierarchy:                                                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Application: my-web-app                                      │  │
│ │ ├── Application Version: v1.0 (source bundle ZIP)           │  │
│ │ ├── Application Version: v1.1                                │  │
│ │ ├── Application Version: v2.0                                │  │
│ │ │                                                            │  │
│ │ ├── Environment: my-web-app-prod (running v1.1)            │  │
│ │ │   ├── ALB + ASG (3 EC2 instances)                        │  │
│ │ │   ├── RDS PostgreSQL                                     │  │
│ │ │   └── URL: my-web-app-prod.us-east-1.elasticbeanstalk.com│  │
│ │ │                                                            │  │
│ │ └── Environment: my-web-app-staging (running v2.0)         │  │
│ │     ├── ALB + ASG (1 EC2 instance)                         │  │
│ │     └── URL: my-web-app-staging...elasticbeanstalk.com     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ EB vs other compute services:                                       │
│ ┌────────────────┬────────────────────────────────────────────┐   │
│ │ Service         │ Best for                                   │   │
│ ├────────────────┼────────────────────────────────────────────┤   │
│ │ Elastic Beanstalk│ Traditional web apps, quick deployment  │   │
│ │                │ Teams that want managed infra but full access│  │
│ │ ECS/EKS        │ Containerized microservices, K8s expertise│   │
│ │ Lambda         │ Event-driven, short-lived functions       │   │
│ │ App Runner     │ Simplest containers (even less config)    │   │
│ │ EC2 (raw)      │ Full control, custom setups               │   │
│ └────────────────┴────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ GCP equivalent: App Engine                                      │
│ ⚡ Azure equivalent: App Service                                   │
│                                                                       │
│ Supported platforms:                                                 │
│ ├── Node.js (14, 16, 18, 20)                                    │
│ ├── Python (3.8, 3.9, 3.11, 3.12)                              │
│ ├── Java (Corretto 8, 11, 17, 21 — Tomcat or SE)              │
│ ├── .NET on Linux (.NET 6, 8)                                   │
│ ├── PHP (8.1, 8.2, 8.3)                                        │
│ ├── Ruby (3.1, 3.2, 3.3)                                       │
│ ├── Go (1.21, 1.22)                                             │
│ ├── Docker (single or multi-container)                          │
│ └── Docker Compose (multi-container on single instance)        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating an Application (Full Console Walkthrough)

```
Console → Elastic Beanstalk → Create application

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 1: CONFIGURE ENVIRONMENT                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Environment tier:                                                    │
│ ● Web server environment                                           │
│   ├── HTTP requests → ALB → EC2 instances                       │
│   └── For: Websites, APIs, web apps                              │
│                                                                       │
│ ○ Worker environment                                                │
│   ├── Processes messages from SQS queue                          │
│   ├── No load balancer, no HTTP endpoint                        │
│   └── For: Background jobs, queue processing, scheduled tasks   │
│                                                                       │
│ ⚡ Common pattern: Web server → SQS queue → Worker                │
│   Web tier handles HTTP, sends async work to queue.             │
│   Worker tier processes queue messages.                          │
│                                                                       │
│ Application name: [my-web-app]                                     │
│ Environment name: [my-web-app-prod]                                │
│                                                                       │
│ Domain: [my-web-app-prod] .us-east-1.elasticbeanstalk.com        │
│ ⚡ Auto-generated URL. Add custom domain via Route 53 later.      │
│                                                                       │
│ Platform:                                                            │
│ Platform type: ● Managed platform                                  │
│ Platform: [Node.js ▼]                                              │
│ Platform branch: [Node.js 20 running on 64bit Amazon Linux 2023]│
│ Platform version: [6.1.5 (Recommended) ▼]                       │
│                                                                       │
│ Application code:                                                    │
│ ○ Sample application (EB provides a default app)                 │
│ ● Upload your code                                                 │
│   Source code origin: [Local file ▼]                              │
│   [Choose file] → my-app-v1.0.zip                                │
│   Version label: [v1.0]                                           │
│                                                                       │
│ ⚡ Source bundle: ZIP file containing your app code.               │
│   Node.js: Must have package.json at root.                      │
│   Python: Must have requirements.txt or Pipfile.                │
│   Docker: Must have Dockerfile at root.                         │
│                                                                       │
│ Presets:                                                             │
│ ○ Single instance (free tier — no ALB, 1 EC2)                   │
│   ⚡ Dev/test. No load balancer = cheapest.                       │
│                                                                       │
│ ● High availability (ALB + ASG + multi-AZ)                      │
│   ⚡ Production. ALB + ASG across AZs.                            │
│                                                                       │
│ ○ Custom configuration (you set everything)                      │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 2: CONFIGURE SERVICE ACCESS                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Service role: [aws-elasticbeanstalk-service-role ▼]              │
│ ⚡ EB service role: Allows EB to manage resources on your behalf. │
│   Policies: AWSElasticBeanstalkEnhancedHealth,                   │
│   AWSElasticBeanstalkManagedUpdatesCustomerRolePolicy            │
│                                                                       │
│ EC2 key pair: [my-key-pair ▼] (optional — for SSH access)       │
│ ⚡ Production: Skip this. Use SSM Session Manager instead.       │
│                                                                       │
│ EC2 instance profile: [aws-elasticbeanstalk-ec2-role ▼]         │
│ ⚡ IAM role that EC2 instances assume. Needs:                     │
│   ├── AWSElasticBeanstalkWebTier (or WorkerTier)               │
│   ├── AWSElasticBeanstalkMulticontainerDocker (if Docker)      │
│   ├── CloudWatchAgentServerPolicy (for enhanced health)        │
│   └── Add your own: S3, DynamoDB, SQS access as needed       │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 3: NETWORKING, DATABASE & TAGS                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── VPC ──                                                           │
│ VPC: [vpc-prod ▼]                                                  │
│                                                                       │
│ Instance subnets (EC2 — PRIVATE):                                 │
│ ☑ subnet-private-1a (10.0.1.0/24)                                │
│ ☑ subnet-private-1b (10.0.2.0/24)                                │
│ ☑ subnet-private-1c (10.0.3.0/24)                                │
│                                                                       │
│ Load balancer subnets (ALB — PUBLIC):                              │
│ ☑ subnet-public-1a (10.0.101.0/24)                               │
│ ☑ subnet-public-1b (10.0.102.0/24)                               │
│ ☑ subnet-public-1c (10.0.103.0/24)                               │
│                                                                       │
│ Public IP: ☐ (disabled — instances in private subnets)           │
│                                                                       │
│ ── Database (RDS) ──                                                │
│                                                                       │
│ ☐ Enable database                                                  │
│ ⚡ WARNING: RDS created here is TIED to the EB environment!       │
│   Delete environment → DELETE DATABASE! Data lost!              │
│   ⚡ NEVER use this for production!                               │
│   Instead: Create RDS separately and pass connection string    │
│   via environment variables.                                    │
│                                                                       │
│ (If checked — DEV ONLY:)                                           │
│ Engine: [postgres ▼]                                               │
│ Engine version: [16.3]                                              │
│ Instance class: [db.t3.micro ▼]                                   │
│ Storage: [20] GB                                                    │
│ Username: [dbadmin]                                                 │
│ Password: [********]                                                │
│ Availability: [Low (one AZ)]                                      │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 4: INSTANCE TRAFFIC & SCALING                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── EC2 Instances ──                                                 │
│                                                                       │
│ Root volume: [General Purpose 3 (SSD) ▼]                         │
│ Size: [10] GB                                                       │
│                                                                       │
│ Instance types: [t3.small ▼]                                      │
│ [Add instance type] → t3.medium, m5.large                        │
│ ⚡ Multiple types for better Spot availability.                    │
│                                                                       │
│ AMI ID: (auto-selected based on platform)                         │
│                                                                       │
│ ── Capacity ──                                                      │
│                                                                       │
│ Auto scaling group:                                                 │
│ Environment type: ● Load balanced                                 │
│                    ○ Single instance                              │
│                                                                       │
│ Instances: min [2], max [8], desired [3]                          │
│                                                                       │
│ Fleet composition:                                                  │
│ ○ On-demand instances                                             │
│ ● Combined (On-demand + Spot)                                    │
│                                                                       │
│ On-demand base: [2] (guaranteed On-Demand)                       │
│ On-demand above base: [30]% (rest can be Spot)                   │
│ Spot max price: [Auto ▼] (per-instance-type pricing)             │
│ ⚡ Base 2 On-Demand + rest is 70% Spot = big savings!             │
│                                                                       │
│ Scaling triggers:                                                    │
│ Metric: [NetworkOut ▼]                                             │
│ ├── NetworkOut (default — good general metric)                  │
│ ├── CPUUtilization                                               │
│ ├── RequestCount                                                 │
│ └── Latency                                                      │
│ Upper threshold: [6000000] bytes  → Scale up                    │
│ Lower threshold: [2000000] bytes  → Scale down                  │
│ Period: [5] minutes                                                │
│ Breach duration: [5] minutes                                      │
│ Scale up increment: [1]                                            │
│ Scale down increment: [-1]                                        │
│ Cooldown: [360] seconds                                            │
│                                                                       │
│ ── Load balancer ──                                                 │
│                                                                       │
│ Load balancer type:                                                 │
│ ● Application Load Balancer (ALB) ← HTTP/HTTPS, recommended   │
│ ○ Network Load Balancer (NLB) ← TCP, ultra-high performance    │
│ ○ Classic Load Balancer ← Legacy, don't use                     │
│                                                                       │
│ Visibility: ● Public  ○ Internal                                 │
│                                                                       │
│ Listeners:                                                           │
│ ├── Port 80 (HTTP) → default process (port 80)                 │
│ ├── [+ Add listener]                                             │
│ │   Port: 443, Protocol: HTTPS                                  │
│ │   SSL certificate: [arn:aws:acm:...certificate/abc ▼]       │
│ │   SSL policy: [ELBSecurityPolicy-TLS13-1-2-2021-06 ▼]      │
│ └── ⚡ Always add HTTPS listener for production!                  │
│                                                                       │
│ Health check:                                                        │
│ Path: [/health]                                                     │
│ Interval: [15] seconds                                              │
│ Healthy threshold: [3]                                              │
│ Unhealthy threshold: [5]                                            │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           STEP 5: UPDATES, MONITORING & LOGGING                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Health reporting ──                                              │
│ System: ● Enhanced ← Detailed per-instance health              │
│          ○ Basic                                                  │
│ ⚡ Enhanced: See HTTP status codes, CPU, response time per        │
│   instance. Requires CloudWatch agent on instances.             │
│                                                                       │
│ ── Managed platform updates ──                                     │
│ ☑ Enabled                                                          │
│ ⚡ Auto-applies platform patches (minor updates, security fixes).│
│   Maintenance window: [MON:04:00] (Monday 4 AM UTC)            │
│   Update level: ● Minor and patch  ○ Patch only                │
│   ⚡ Runs as immutable deployment (safe, zero-downtime).         │
│                                                                       │
│ ── Rolling updates & deployments ──                                │
│                                                                       │
│ Deployment policy: [Rolling with additional batch ▼]              │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ○ All at once                                                │  │
│ │   ├── Deploy to ALL instances simultaneously                │  │
│ │   ├── Fastest but causes downtime!                          │  │
│ │   └── Use for: dev/test only                                │  │
│ │                                                              │  │
│ │ ○ Rolling                                                    │  │
│ │   ├── Deploy in batches (e.g., 30% at a time)              │  │
│ │   ├── Reduced capacity during deployment                   │  │
│ │   └── Use for: Non-critical apps                           │  │
│ │                                                              │  │
│ │ ● Rolling with additional batch ← RECOMMENDED              │  │
│ │   ├── Launch NEW batch first, then update old in batches   │  │
│ │   ├── Capacity NEVER drops below current level             │  │
│ │   └── Use for: Production ✅                                │  │
│ │                                                              │  │
│ │ ○ Immutable                                                  │  │
│ │   ├── Launch entirely NEW ASG with new version              │  │
│ │   ├── Verify healthy → swap to new, terminate old          │  │
│ │   ├── Safest but slowest and most expensive                │  │
│ │   └── Use for: Critical production (safest rollback)       │  │
│ │                                                              │  │
│ │ ○ Traffic splitting                                          │  │
│ │   ├── Canary: Send X% traffic to new version              │  │
│ │   ├── Monitor → promote to 100% or rollback               │  │
│ │   └── Use for: Canary testing in production                │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Batch size: [30]% (for Rolling / Rolling with additional batch)  │
│                                                                       │
│ ── Configuration updates (ASG/LB/instance changes) ──             │
│ Rolling update type: [Rolling based on Health ▼]                 │
│ Max batch size: [1]                                                 │
│ Min instances in service: [1]                                      │
│                                                                       │
│ ── Notifications ──                                                 │
│ Email: [devops@company.com]                                        │
│ ⚡ SNS notifications for environment events (deploy, health, etc.)│
│                                                                       │
│ ── Log streaming ──                                                 │
│ ☑ Stream logs to CloudWatch Logs                                  │
│ Retention: [7 days ▼]                                              │
│ Lifecycle: ☑ Delete logs upon environment termination            │
│                                                                       │
│ [Next → Review → Create]                                           │
│                                                                       │
│ ⚡ Environment creation takes 5-10 minutes.                       │
│   EB provisions: ALB, ASG, EC2, security groups, CloudWatch.    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Environment Configuration Deep Dive

```
┌─────────────────────────────────────────────────────────────────────┐
│           SOFTWARE CONFIGURATION                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Environment → Configuration → Software                             │
│                                                                       │
│ Proxy server: [Nginx ▼]                                            │
│ ├── Nginx (default — reverse proxy in front of your app)        │
│ ├── Apache (alternative reverse proxy)                           │
│ └── None (your app handles connections directly)                │
│                                                                       │
│ ⚡ Nginx handles:                                                    │
│   ├── Static file serving                                        │
│   ├── Gzip compression                                           │
│   ├── Connection buffering                                       │
│   ├── Keep-alive connections                                     │
│   └── Forwards to your app on port 8080 (or $PORT)             │
│                                                                       │
│ ── Environment properties (environment variables) ──              │
│ ┌────────────────────────┬──────────────────────────────────┐    │
│ │ Key                    │ Value                              │    │
│ ├────────────────────────┼──────────────────────────────────┤    │
│ │ NODE_ENV               │ production                        │    │
│ │ DB_HOST                │ mydb.xxx.rds.amazonaws.com        │    │
│ │ DB_NAME                │ myapp                              │    │
│ │ DB_USER                │ admin                              │    │
│ │ DB_PASSWORD            │ ********                          │    │
│ │ REDIS_URL              │ redis://xxx.cache.amazonaws.com   │    │
│ │ S3_BUCKET              │ my-app-uploads                    │    │
│ │ API_KEY                │ sk-xxxx (use Secrets Manager!)    │    │
│ └────────────────────────┴──────────────────────────────────┘    │
│                                                                       │
│ ⚡ Environment variables are available to your app code.           │
│   process.env.DB_HOST (Node.js)                                 │
│   os.environ['DB_HOST'] (Python)                                 │
│                                                                       │
│ ⚡ For secrets: Use AWS Secrets Manager + SDK in your code.       │
│   Don't put secrets as plain-text env vars in production.       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Deployment Policies (Visual Comparison)

```
┌─────────────────────────────────────────────────────────────────────┐
│           DEPLOYMENT STRATEGIES                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Starting state: 4 instances running v1                             │
│                                                                       │
│ 1. ALL AT ONCE (fastest, downtime)                                 │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Time 0: [v1][v1][v1][v1]  ← All running v1                 │  │
│ │ Time 1: [..][..][..][..]  ← All deploying (DOWNTIME!)      │  │
│ │ Time 2: [v2][v2][v2][v2]  ← All running v2                 │  │
│ │ Rollback: Re-deploy v1 (slow)                               │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 2. ROLLING (batch by batch, reduced capacity)                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Time 0: [v1][v1][v1][v1]  ← All running v1                 │  │
│ │ Time 1: [..][v1][v1][v1]  ← Batch 1 deploying (3 capacity)│  │
│ │ Time 2: [v2][v1][v1][v1]  ← Batch 1 done                  │  │
│ │ Time 3: [v2][..][v1][v1]  ← Batch 2 deploying             │  │
│ │ Time 4: [v2][v2][v1][v1]  ← Batch 2 done                  │  │
│ │ ...repeat until all v2...                                   │  │
│ │ Final:  [v2][v2][v2][v2]                                    │  │
│ │ ⚠️ Capacity reduced during deployment!                      │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 3. ROLLING WITH ADDITIONAL BATCH (no capacity loss) ✅            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Time 0: [v1][v1][v1][v1]  ← All running v1                 │  │
│ │ Time 1: [v1][v1][v1][v1][v2]  ← NEW instance with v2 added│  │
│ │ Time 2: [v2][v1][v1][v1][v2]  ← Batch 1 upgraded          │  │
│ │ Time 3: [v2][v2][v1][v1][v2]  ← Batch 2 upgraded          │  │
│ │ ...repeat...                                                │  │
│ │ Final:  [v2][v2][v2][v2]  ← Extra instance removed        │  │
│ │ ⚡ Capacity NEVER drops below 4!                             │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 4. IMMUTABLE (safest, new ASG)                                    │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Time 0: ASG-1: [v1][v1][v1][v1]                             │  │
│ │ Time 1: ASG-1: [v1][v1][v1][v1]                             │  │
│ │         ASG-2: [v2]  ← NEW ASG, 1 instance first          │  │
│ │ Time 2: ASG-2 healthy? → scale to 4                        │  │
│ │         ASG-2: [v2][v2][v2][v2]                             │  │
│ │ Time 3: Move ASG-2 instances to ASG-1                      │  │
│ │         Terminate old v1 instances                         │  │
│ │ Final:  ASG-1: [v2][v2][v2][v2]                             │  │
│ │ ⚡ Rollback: Terminate ASG-2 (old ASG-1 untouched!)        │  │
│ │ ⚡ Most expensive (double capacity during deploy)           │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 5. TRAFFIC SPLITTING (canary)                                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Time 0: ASG-1: [v1][v1][v1][v1] ← 100% traffic            │  │
│ │ Time 1: ASG-1: [v1][v1][v1][v1] ← 90% traffic             │  │
│ │         ASG-2: [v2]              ← 10% traffic (canary)    │  │
│ │ Time 2: Monitor for [evaluation time] minutes              │  │
│ │         Error rate OK? → Shift 100% to v2                  │  │
│ │         Error rate high? → Rollback to v1                  │  │
│ │ Final:  ASG-1: [v2][v2][v2][v2] ← 100% traffic            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 6. BLUE/GREEN (swap environment URLs)                             │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Blue (prod):   my-app-prod.elasticbeanstalk.com → v1       │  │
│ │ Green (new):   my-app-green.elasticbeanstalk.com → v2      │  │
│ │                                                              │  │
│ │ Test green environment → looks good                        │  │
│ │ [Swap environment URLs]                                     │  │
│ │                                                              │  │
│ │ Blue (prod):   my-app-prod.elasticbeanstalk.com → v2 ✅    │  │
│ │ Green (old):   my-app-green.elasticbeanstalk.com → v1      │  │
│ │                                                              │  │
│ │ ⚡ Instant swap! Rollback = swap again.                     │  │
│ │ ⚡ Requires TWO environments (double cost during transition)│  │
│ │ ⚡ Done via: Actions → Swap environment URLs               │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Summary:                                                             │
│ ┌─────────────────────────┬──────┬─────────┬──────────┬─────────┐│
│ │ Policy                  │Speed │Downtime │Rollback  │Cost     ││
│ ├─────────────────────────┼──────┼─────────┼──────────┼─────────┤│
│ │ All at once             │Fast  │Yes ❌   │Slow      │None     ││
│ │ Rolling                 │Med   │No ✅    │Manual    │None     ││
│ │ Rolling + batch         │Med   │No ✅    │Manual    │Small    ││
│ │ Immutable               │Slow  │No ✅    │Fast ✅   │High     ││
│ │ Traffic splitting       │Med   │No ✅    │Fast ✅   │Med      ││
│ │ Blue/green              │Fast  │No ✅    │Instant ✅│2x env   ││
│ └─────────────────────────┴──────┴─────────┴──────────┴─────────┘│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: .ebextensions & Platform Hooks

```
┌─────────────────────────────────────────────────────────────────────┐
│           .ebextensions                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ .ebextensions/ folder in your source bundle = custom config!     │
│ YAML files that run during deployment.                            │
│                                                                       │
│ my-app/                                                              │
│ ├── .ebextensions/                                                 │
│ │   ├── 01-packages.config       ← Install OS packages           │
│ │   ├── 02-nginx.config          ← Custom nginx config           │
│ │   ├── 03-cloudwatch.config     ← Custom CloudWatch metrics    │
│ │   └── 04-https-redirect.config ← HTTP → HTTPS redirect       │
│ ├── .platform/                                                     │
│ │   └── hooks/                                                    │
│ │       ├── predeploy/           ← Scripts before deploy         │
│ │       └── postdeploy/          ← Scripts after deploy          │
│ ├── package.json                                                   │
│ └── server.js                                                      │
│                                                                       │
│ Example: 01-packages.config                                        │
│ packages:                                                            │
│   yum:                                                               │
│     ImageMagick: []                                                │
│     git: []                                                         │
│                                                                       │
│ Example: 02-https-redirect.config                                  │
│ files:                                                               │
│   "/etc/nginx/conf.d/https_redirect.conf":                       │
│     mode: "000644"                                                 │
│     owner: root                                                    │
│     group: root                                                    │
│     content: |                                                     │
│       server {                                                     │
│           listen 80;                                               │
│           if ($http_x_forwarded_proto = "http") {                │
│               return 301 https://$host$request_uri;              │
│           }                                                        │
│       }                                                            │
│                                                                       │
│ Example: .platform/hooks/postdeploy/01-migrate.sh                │
│ #!/bin/bash                                                         │
│ cd /var/app/current                                                 │
│ npx prisma migrate deploy                                          │
│                                                                       │
│ ⚡ .ebextensions order: Files run alphabetically (01, 02, 03...) │
│ ⚡ Platform hooks: predeploy (before app starts),                 │
│   postdeploy (after app starts and health check passes).        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform

```hcl
# Elastic Beanstalk Application
resource "aws_elastic_beanstalk_application" "web" {
  name        = "my-web-app"
  description = "My web application"

  appversion_lifecycle {
    service_role          = aws_iam_role.eb_service.arn
    max_count             = 10
    delete_source_from_s3 = true
  }
}

# Application Version
resource "aws_s3_bucket" "app_versions" {
  bucket = "my-web-app-versions"
}

resource "aws_s3_object" "app_v1" {
  bucket = aws_s3_bucket.app_versions.id
  key    = "v1.0.zip"
  source = "deploy/v1.0.zip"
}

resource "aws_elastic_beanstalk_application_version" "v1" {
  name        = "v1.0"
  application = aws_elastic_beanstalk_application.web.name
  bucket      = aws_s3_bucket.app_versions.id
  key         = aws_s3_object.app_v1.key
}

# Environment (Production — HA)
resource "aws_elastic_beanstalk_environment" "prod" {
  name                = "my-web-app-prod"
  application         = aws_elastic_beanstalk_application.web.name
  solution_stack_name = "64bit Amazon Linux 2023 v6.1.5 running Node.js 20"
  version_label       = aws_elastic_beanstalk_application_version.v1.name
  tier                = "WebServer"

  # Instance settings
  setting {
    namespace = "aws:autoscaling:launchconfiguration"
    name      = "InstanceType"
    value     = "t3.small"
  }
  setting {
    namespace = "aws:autoscaling:launchconfiguration"
    name      = "IamInstanceProfile"
    value     = aws_iam_instance_profile.eb_ec2.name
  }

  # VPC
  setting {
    namespace = "aws:ec2:vpc"
    name      = "VPCId"
    value     = aws_vpc.main.id
  }
  setting {
    namespace = "aws:ec2:vpc"
    name      = "Subnets"
    value     = join(",", aws_subnet.private[*].id)
  }
  setting {
    namespace = "aws:ec2:vpc"
    name      = "ELBSubnets"
    value     = join(",", aws_subnet.public[*].id)
  }

  # Auto Scaling
  setting {
    namespace = "aws:autoscaling:asg"
    name      = "MinSize"
    value     = "2"
  }
  setting {
    namespace = "aws:autoscaling:asg"
    name      = "MaxSize"
    value     = "8"
  }

  # Load Balancer
  setting {
    namespace = "aws:elasticbeanstalk:environment"
    name      = "LoadBalancerType"
    value     = "application"
  }
  setting {
    namespace = "aws:elbv2:listener:443"
    name      = "Protocol"
    value     = "HTTPS"
  }
  setting {
    namespace = "aws:elbv2:listener:443"
    name      = "SSLCertificateArns"
    value     = aws_acm_certificate.main.arn
  }

  # Deployment policy
  setting {
    namespace = "aws:elasticbeanstalk:command"
    name      = "DeploymentPolicy"
    value     = "RollingWithAdditionalBatch"
  }
  setting {
    namespace = "aws:elasticbeanstalk:command"
    name      = "BatchSizeType"
    value     = "Percentage"
  }
  setting {
    namespace = "aws:elasticbeanstalk:command"
    name      = "BatchSize"
    value     = "30"
  }

  # Enhanced health
  setting {
    namespace = "aws:elasticbeanstalk:healthreporting:system"
    name      = "SystemType"
    value     = "enhanced"
  }

  # Managed updates
  setting {
    namespace = "aws:elasticbeanstalk:managedactions"
    name      = "ManagedActionsEnabled"
    value     = "true"
  }
  setting {
    namespace = "aws:elasticbeanstalk:managedactions"
    name      = "PreferredStartTime"
    value     = "MON:04:00"
  }
  setting {
    namespace = "aws:elasticbeanstalk:managedactions:platformupdate"
    name      = "UpdateLevel"
    value     = "minor"
  }

  # Environment variables
  setting {
    namespace = "aws:elasticbeanstalk:application:environment"
    name      = "NODE_ENV"
    value     = "production"
  }
  setting {
    namespace = "aws:elasticbeanstalk:application:environment"
    name      = "DB_HOST"
    value     = aws_db_instance.main.address
  }

  # CloudWatch logs
  setting {
    namespace = "aws:elasticbeanstalk:cloudwatch:logs"
    name      = "StreamLogs"
    value     = "true"
  }
  setting {
    namespace = "aws:elasticbeanstalk:cloudwatch:logs"
    name      = "RetentionInDays"
    value     = "7"
  }

  tags = {
    Environment = "prod"
    Team        = "backend"
  }
}

# IAM roles
resource "aws_iam_role" "eb_service" {
  name = "eb-service-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "elasticbeanstalk.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eb_service" {
  for_each = toset([
    "arn:aws:iam::aws:policy/service-role/AWSElasticBeanstalkEnhancedHealth",
    "arn:aws:iam::aws:policy/AWSElasticBeanstalkManagedUpdatesCustomerRolePolicy",
  ])
  role       = aws_iam_role.eb_service.name
  policy_arn = each.value
}

resource "aws_iam_role" "eb_ec2" {
  name = "eb-ec2-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_instance_profile" "eb_ec2" {
  name = "eb-ec2-profile"
  role = aws_iam_role.eb_ec2.name
}

resource "aws_iam_role_policy_attachment" "eb_ec2" {
  for_each = toset([
    "arn:aws:iam::aws:policy/AWSElasticBeanstalkWebTier",
    "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy",
  ])
  role       = aws_iam_role.eb_ec2.name
  policy_arn = each.value
}
```

---

## Part 7: eb CLI Reference

```bash
# Install EB CLI
pip install awsebcli

# Initialize EB in your project
cd my-app
eb init
# → Select region
# → Select application (or create new)
# → Select platform (Node.js 20)
# → Set up SSH? No (use SSM)

# Create environment
eb create my-web-app-prod \
  --instance-type t3.small \
  --elb-type application \
  --scale 3 \
  --min-instances 2 \
  --max-instances 8 \
  --envvars NODE_ENV=production,DB_HOST=xxx.rds.amazonaws.com \
  --vpc.id vpc-xxx \
  --vpc.elbsubnets subnet-pub1,subnet-pub2 \
  --vpc.ec2subnets subnet-priv1,subnet-priv2 \
  --vpc.elbpublic \
  --tags environment=prod,team=backend

# Deploy new version
eb deploy
# → Zips current directory, uploads, deploys

# Deploy specific version
eb deploy --version v2.0

# Open app in browser
eb open

# Check status
eb status

# View health (enhanced)
eb health

# View logs
eb logs
eb logs --all  # Download all logs
eb logs -g /var/log/web.stdout.log  # Specific log group

# SSH into instance (if key pair set)
eb ssh

# Set environment variables
eb setenv DB_HOST=new-db.rds.amazonaws.com DB_PASSWORD=newpass

# Scale
eb scale 5  # Set desired count to 5

# Swap URLs (blue/green)
eb swap my-web-app-prod --destination_name my-web-app-green

# Terminate environment
eb terminate my-web-app-prod

# List environments
eb list

# Use a saved configuration
eb config save my-web-app-prod --cfg prod-config
eb create my-web-app-staging --cfg prod-config

# Abort in-progress deployment
eb abort
```

---

## Part 8: Real-World Patterns

### Startup

```
Setup: Simple web app on Elastic Beanstalk

Application: my-web-app
Environment: my-web-app-prod
├── Platform: Node.js 20 on Amazon Linux 2023
├── Preset: High availability
├── Instance: t3.small (2 vCPU, 2 GiB)
├── Scaling: min 2, max 4
├── ALB: HTTPS (ACM certificate)
├── Deploy: Rolling with additional batch (30%)
├── Health: Enhanced
├── Managed updates: Enabled (Monday 4 AM)
├── Logs: CloudWatch streaming
└── Custom domain: app.startup.com (Route 53 alias → ALB)

Database: RDS PostgreSQL (SEPARATE from EB!)
├── db.t3.micro, 20 GB
├── Private subnet
└── Connection string via env vars

Deploy workflow:
├── Developer pushes to main
├── GitHub Actions: npm test → npm build → zip
├── eb deploy (via GitHub Actions)
└── Rolling deployment, zero downtime

Cost: ~$80/month (2× t3.small + ALB + RDS)
```

### Mid-Size

```
Architecture: EB web + worker tiers

Web tier: my-app-web-prod
├── t3.medium (2 vCPU, 4 GiB), min 3, max 12
├── ALB + HTTPS + WAF (AWS WAF in front)
├── Deploy: Rolling with additional batch (25%)
├── Spot: Base 3 On-Demand + rest Spot (70%)
├── .ebextensions: Custom nginx config, CloudWatch metrics
├── .platform/hooks: DB migration on postdeploy
├── Env vars: RDS, Redis, S3, Secrets Manager ARN
└── Custom domain: api.company.com

Worker tier: my-app-worker-prod
├── t3.medium, min 2, max 10
├── SQS queue: my-app-orders
├── Processes: Order fulfillment, email sending
├── No ALB (worker tier has SQS daemon built-in)
├── Scaling: Queue depth > 100 → add instance
└── cron.yaml for scheduled tasks:
    version: 1
    cron:
      - name: "daily-report"
        url: "/scheduled/daily-report"
        schedule: "0 2 * * *"

Blue/Green deployment:
├── Create green environment with v2
├── Run smoke tests against green URL
├── eb swap (swap URLs — instant!)
├── Monitor 30 min → if issues, swap back
└── Terminate green (now running v1) after 24h

Monitoring:
├── Enhanced health reporting (per instance)
├── CloudWatch alarms: CPU > 80%, 5xx > 10/min
├── Application logs → CloudWatch Logs
├── X-Ray: Distributed tracing
└── Alerts: SNS → PagerDuty

Cost: ~$400-1000/month (with Spot savings)
```

### Enterprise

```
Multi-environment, multi-region EB platform:

Region 1 (ap-south-1):
├── my-app-web-prod
│   ├── m5.large, min 6, max 20, 3 AZs
│   ├── ALB + WAF + Shield Advanced
│   ├── HTTPS only (redirect HTTP)
│   ├── Deploy: Immutable (safest)
│   ├── Managed updates: Minor + patch
│   ├── .ebextensions:
│   │   ├── CloudWatch custom metrics
│   │   ├── DataDog agent install
│   │   ├── CIS benchmark hardening
│   │   └── Log forwarding to Splunk
│   ├── VPC: Private subnets, NAT gateway
│   └── Secrets Manager for all credentials
│
├── my-app-worker-prod
│   ├── m5.large, min 4, max 15
│   ├── SQS queues: orders, notifications, reports
│   └── Dead-letter queue monitoring
│
├── my-app-web-staging
│   ├── t3.medium, min 1, max 3
│   └── Used for pre-production testing

Region 2 (eu-west-1):
├── my-app-web-eu
│   ├── Same config as prod
│   └── Route 53 latency routing between regions

CI/CD pipeline:
├── GitHub → CodePipeline
│   Stage 1: CodeBuild (npm test, npm build)
│   Stage 2: Deploy to staging (eb deploy)
│   Stage 3: Integration tests against staging
│   Stage 4: Manual approval
│   Stage 5: Deploy to prod (immutable)
│   Stage 6: Canary check (CloudWatch alarms)
│   Stage 7: Promote or rollback

Security:
├── IMDSv2 enforced on all instances
├── No SSH keys (SSM Session Manager only)
├── WAF: OWASP managed rules
├── VPC: Private subnets, no public IPs
├── Security groups: Minimal ports
├── IAM: Least privilege EC2 instance profile
├── Secrets Manager: DB creds, API keys
├── AWS Config: Compliance monitoring
└── GuardDuty: Threat detection

Cost: $3,000-10,000/month (multi-region, multi-env)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | PaaS — upload code, EB manages infra |
| Underlying | EC2, ALB, ASG, S3, CloudWatch, RDS |
| Cost | Free (pay for underlying resources only) |
| Platforms | Node.js, Python, Java, .NET, PHP, Ruby, Go, Docker |
| Tiers | Web server (HTTP) or Worker (SQS) |
| Deploy: All at once | Fastest, has downtime |
| Deploy: Rolling | Batches, reduced capacity |
| Deploy: Rolling + batch | Batches, no capacity loss |
| Deploy: Immutable | New ASG, safest rollback |
| Deploy: Traffic split | Canary with metrics eval |
| Deploy: Blue/Green | Swap URLs, instant rollback |
| .ebextensions | YAML config files in source bundle |
| Platform hooks | Shell scripts (predeploy/postdeploy) |
| Managed updates | Auto-patching on schedule |
| GCP equivalent | App Engine |
| Azure equivalent | App Service |

---

## What's Next?

In the next chapter, we'll cover AWS App Runner — the simplest way to run containers on AWS.

→ Next: [Chapter 19: App Runner](19-app-runner.md)

---

*Last Updated: May 2026*
