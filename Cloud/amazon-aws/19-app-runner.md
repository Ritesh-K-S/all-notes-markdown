# Chapter 19: AWS App Runner

---

## Table of Contents

- [Overview](#overview)
- [Part 1: App Runner Fundamentals](#part-1-app-runner-fundamentals)
- [Part 2: Creating a Service (Full Console Walkthrough)](#part-2-creating-a-service-full-console-walkthrough)
- [Part 3: apprunner.yaml (Configuration File)](#part-3-apprunneryaml-configuration-file)
- [Part 4: Custom Domains](#part-4-custom-domains)
- [Part 5: VPC Connector (Access Private Resources)](#part-5-vpc-connector-access-private-resources)
- [Part 6: Observability](#part-6-observability)
- [Part 7: Terraform](#part-7-terraform)
- [Part 8: AWS CLI Reference](#part-8-aws-cli-reference)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is App Runner? How is it Different from Beanstalk?

> **Real-World Analogy:** App Runner is like Heroku on AWS. You give it your code or container, and it handles everything — building, deploying, scaling, HTTPS certificates. If Lambda is a vending machine (event-driven, per-request), App Runner is a food truck (always ready to serve, scales with the lunch crowd, parks when nobody's hungry).

**Why does this matter?** App Runner is the simplest compute service on AWS. If you just want to deploy a web app or API without learning any infrastructure concepts, this is your answer.

**When NOT to use App Runner:** No cron/scheduled tasks, no GPU workloads, no WebSocket long-polling, limited to web/API workloads. For those needs, use ECS or EC2.

AWS App Runner is the simplest way to deploy containers or source code on AWS. No infrastructure to manage, no Kubernetes knowledge needed — push your code or image and App Runner handles building, deploying, load balancing, scaling (including to zero), TLS, and health checks. It's AWS's answer to "I just want to run my app."

```
What you'll learn:
├── App Runner Fundamentals
│   ├── What & why (fully managed container service)
│   ├── App Runner vs ECS vs EKS vs Elastic Beanstalk vs Lambda
│   └── Source types (container image vs source code)
├── Creating a Service (Full Console Walkthrough)
│   ├── Step 1: Source and deployment
│   ├── Step 2: Configure build (for source code)
│   ├── Step 3: Configure service
│   ├── Step 4: Review and create
│   └── All fields explained
├── Auto Scaling
├── Custom Domains & HTTPS
├── VPC Connector (access private resources)
├── Observability (logs, metrics, tracing)
├── Terraform & CLI
└── Real-world patterns
```

---

## Part 1: App Runner Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           APP RUNNER CONCEPT                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is App Runner?                                                 │
│ ├── Fully managed service to run web apps & APIs                │
│ ├── Give it source code (GitHub) or container image (ECR)       │
│ ├── App Runner handles EVERYTHING:                               │
│ │   ├── Build (if source code)                                  │
│ │   ├── Deploy                                                   │
│ │   ├── HTTPS/TLS certificates (auto)                           │
│ │   ├── Load balancing                                           │
│ │   ├── Auto-scaling (including scale to zero!)                │
│ │   ├── Health checks                                           │
│ │   └── Rolling deployments                                    │
│ ├── No Dockerfiles required for source code (buildpacks)       │
│ └── Pay per compute: vCPU-second + memory-second               │
│                                                                       │
│ Source options:                                                      │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ 1. Container image (ECR)                                     │  │
│ │    ├── Push image to ECR → App Runner deploys it            │  │
│ │    ├── Auto-deploy on new image push (optional)             │  │
│ │    └── You control the build, App Runner runs it            │  │
│ │                                                              │  │
│ │ 2. Source code (GitHub)                                      │  │
│ │    ├── Connect GitHub repo → App Runner builds & deploys    │  │
│ │    ├── Auto-deploy on git push (optional)                   │  │
│ │    ├── Supported runtimes:                                   │  │
│ │    │   ├── Python 3                                         │  │
│ │    │   ├── Node.js 12/14/16/18                              │  │
│ │    │   ├── Java 8/11 (Corretto)                             │  │
│ │    │   ├── .NET 6                                            │  │
│ │    │   ├── PHP 8.1                                           │  │
│ │    │   ├── Ruby 3.1                                          │  │
│ │    │   └── Go 1.18                                           │  │
│ │    └── Build commands: install deps → build → start         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Comparison:                                                          │
│ ┌────────────────┬──────────┬──────────┬──────────┬────────────┐ │
│ │ Feature         │ App Runner│ ECS Farg │ EKS      │ Beanstalk  │ │
│ ├────────────────┼──────────┼──────────┼──────────┼────────────┤ │
│ │ Complexity      │ Lowest ✅│ Medium   │ Highest  │ Low-Med    │ │
│ │ Source code     │ Yes ✅   │ No       │ No       │ Yes        │ │
│ │ Container image │ Yes      │ Yes      │ Yes      │ Yes        │ │
│ │ Auto-scale      │ Yes ✅   │ Yes      │ Yes      │ Yes        │ │
│ │ Scale to zero   │ Yes ✅   │ No       │ No       │ No         │ │
│ │ VPC access      │ Connector│ Native   │ Native   │ Native     │ │
│ │ Custom domain   │ Yes ✅   │ Manual   │ Manual   │ Yes        │ │
│ │ TLS auto        │ Yes ✅   │ Manual   │ Manual   │ Yes        │ │
│ │ CI/CD built-in  │ Yes ✅   │ No       │ No       │ Yes        │ │
│ │ WebSocket       │ Yes      │ Yes      │ Yes      │ Yes        │ │
│ │ Cron/scheduled  │ No ❌    │ Yes      │ Yes      │ Yes        │ │
│ │ GPU             │ No ❌    │ No       │ Yes      │ No         │ │
│ │ K8s native      │ No       │ Partial  │ Yes ✅   │ No         │ │
│ │ Pricing         │ Per use  │ Per task │ Per node │ Per EC2    │ │
│ └────────────────┴──────────┴──────────┴──────────┴────────────┘ │
│                                                                       │
│ ⚡ App Runner pricing:                                               │
│   vCPU: $0.064/vCPU-hour (active) + $0.007/vCPU-hour (paused)  │
│   Memory: $0.007/GB-hour (always)                                │
│   Provisioned instances: Pay for idle capacity                  │
│   Build: $0.005/build-minute                                    │
│   Automatic deployments: Free                                   │
│                                                                       │
│ ⚡ "Paused" = scale to zero. You still pay a small amount for    │
│   provisioned instances to keep them warm for fast startup.    │
│                                                                       │
│ ⚡ GCP equivalent: Cloud Run                                       │
│ ⚡ Azure equivalent: Azure Container Apps                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Service (Full Console Walkthrough)

```
Console → App Runner → Create service

┌─────────────────────────────────────────────────────────────────────┐
│   STEP 1: SOURCE AND DEPLOYMENT                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Source:                                                               │
│ ● Container registry                                                │
│ ○ Source code repository                                            │
│                                                                       │
│ ═══ Container registry ═══                                          │
│                                                                       │
│ Provider:                                                            │
│ ● Amazon ECR                                                        │
│ ○ Amazon ECR Public                                                 │
│                                                                       │
│ Container image URI:                                                 │
│ [123456789.dkr.ecr.ap-south-1.amazonaws.com/web-app:latest ▼]   │
│ ⚡ Browse or type. Must be in same region or ECR Public.            │
│                                                                       │
│ ECR access role:                                                     │
│ ● Create new service role (App Runner creates IAM role to pull)  │
│ ○ Use an existing service role                                    │
│   [AppRunnerECRAccessRole ▼]                                       │
│ ⚡ App Runner needs permission to pull images from ECR.             │
│   The role gets ecr:GetAuthorizationToken + ecr:BatchGetImage.  │
│                                                                       │
│ Deployment settings:                                                 │
│ Deployment trigger:                                                  │
│ ● Automatic ← Push new image → App Runner deploys it ✅          │
│ ○ Manual    ← You trigger deployment manually                   │
│                                                                       │
│ ═══ Source code repository ═══                                      │
│                                                                       │
│ Connect to GitHub:                                                   │
│ [+ Add new] → GitHub App connection                                │
│ ⚡ Creates an AWS Connector for GitHub. Authorize AWS to read repo.│
│                                                                       │
│ Repository: [my-org/web-app ▼]                                     │
│ Branch: [main ▼]                                                    │
│                                                                       │
│ Source directory: [/] (root of repo, or subdirectory like /backend)│
│                                                                       │
│ Deployment trigger:                                                  │
│ ● Automatic (deploy on every push to branch) ✅                   │
│ ○ Manual                                                            │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│   STEP 2: CONFIGURE BUILD (source code only)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚡ This step only appears if you chose "Source code repository".    │
│                                                                       │
│ Configuration source:                                                │
│ ● Configure all settings here                                      │
│ ○ Use a configuration file (apprunner.yaml in repo root)         │
│                                                                       │
│ Runtime: [Node.js 18 ▼]                                            │
│ ├── Python 3                                                       │
│ ├── Node.js 12 / 14 / 16 / 18                                   │
│ ├── Java 11 (Corretto)                                            │
│ ├── .NET 6                                                         │
│ ├── PHP 8.1                                                        │
│ ├── Ruby 3.1                                                       │
│ └── Go 1.18                                                        │
│                                                                       │
│ Build command: [npm ci]                                              │
│ ⚡ Install dependencies. Examples:                                   │
│   Node.js: npm ci                                                │
│   Python:  pip install -r requirements.txt                       │
│   Java:    mvn package -DskipTests                               │
│   .NET:    dotnet publish -c Release -o out                      │
│   Go:      go build -o app .                                     │
│                                                                       │
│ Start command: [npm start]                                          │
│ ⚡ Command to start your application. Examples:                     │
│   Node.js: npm start / node server.js                            │
│   Python:  gunicorn app:app --bind 0.0.0.0:8080                 │
│   Java:    java -jar target/app.jar                              │
│   .NET:    dotnet out/MyApp.dll                                  │
│   Go:      ./app                                                 │
│                                                                       │
│ Port: [8080]                                                        │
│ ⚡ Your app MUST listen on this port. App Runner uses PORT env var.│
│   Default: 8080. App Runner routes HTTPS:443 → your port.       │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│   STEP 3: CONFIGURE SERVICE                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Service name: [web-app-prod]                                        │
│ ⚡ Unique within account/region. Used in default URL.               │
│                                                                       │
│ ── Instance configuration ──                                       │
│                                                                       │
│ CPU: [1 vCPU ▼]                                                    │
│ ├── 0.25 vCPU                                                     │
│ ├── 0.5 vCPU                                                      │
│ ├── 1 vCPU  ← good default ✅                                    │
│ ├── 2 vCPU                                                        │
│ └── 4 vCPU                                                        │
│                                                                       │
│ Memory: [2 GB ▼]                                                   │
│ ├── 0.5 GB (only with 0.25 vCPU)                                │
│ ├── 1 GB                                                          │
│ ├── 2 GB  ← good default ✅                                      │
│ ├── 3 GB                                                          │
│ ├── 4 GB                                                          │
│ ├── 6 GB (with 2+ vCPU)                                          │
│ ├── 8 GB (with 2+ vCPU)                                          │
│ └── 12 GB (with 4 vCPU)                                          │
│                                                                       │
│ ⚡ Valid CPU + Memory combinations are enforced. Not all combos work│
│   0.25 vCPU: 0.5-1 GB                                            │
│   0.5 vCPU: 1 GB                                                  │
│   1 vCPU: 2-4 GB                                                  │
│   2 vCPU: 4-6 GB                                                  │
│   4 vCPU: 8-12 GB                                                 │
│                                                                       │
│ ── Environment variables ──                                        │
│ [+ Add environment variable]                                       │
│ ┌────────────────────────┬──────────────────────────────────────┐ │
│ │ Source                  │ Options                               │ │
│ ├────────────────────────┼──────────────────────────────────────┤ │
│ │ ● Plain text            │ Key: NODE_ENV  Value: production    │ │
│ │ ○ Secrets Manager      │ Key: DB_PASS   ARN: arn:aws:secret  │ │
│ │ ○ SSM Parameter Store  │ Key: API_KEY   ARN: arn:aws:ssm     │ │
│ └────────────────────────┴──────────────────────────────────────┘ │
│ ⚡ Secrets Manager / SSM: App Runner fetches at startup.           │
│   Instance role needs secretsmanager:GetSecretValue or             │
│   ssm:GetParameter permissions.                                   │
│                                                                       │
│ ── Auto scaling ──                                                  │
│                                                                       │
│ Auto scaling configuration:                                         │
│ ● Create new (recommended)                                        │
│ ○ Use an existing configuration                                   │
│                                                                       │
│ Configuration name: [prod-scaling]                                  │
│ Max concurrency: [100]                                              │
│ ⚡ Requests per instance before scaling. Default: 100.              │
│   Lower = more instances (higher cost, better latency).          │
│   Higher = fewer instances (lower cost, higher latency).         │
│                                                                       │
│ Max size: [10]                                                      │
│ ⚡ Maximum number of instances. Default: 25.                       │
│                                                                       │
│ Min size: [1]                                                       │
│ ⚡ Minimum instances. Default: 1.                                   │
│   Set to 0: Scale to zero (no active cost, cold starts).        │
│   Set to 1+: Always warm, no cold starts but pay idle.          │
│                                                                       │
│ Scaling flow:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ 0 requests → Min instances (0 or 1+)                        │  │
│ │ Traffic ↑ → Concurrent requests > 100/instance              │  │
│ │          → New instance added (takes ~5-10 seconds)         │  │
│ │ Traffic ↓ → Instances drain → Scale down to min             │  │
│ │                                                              │  │
│ │ With min=0:                                                  │  │
│ │ 0 requests → 0 active instances (pay provisioned only)     │  │
│ │ First request → Cold start (~2-5 seconds) → Instance up   │  │
│ │                                                              │  │
│ │ With min=1:                                                  │  │
│ │ 0 requests → 1 instance ready (pay active compute)         │  │
│ │ First request → Instant response ✅                         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ── Health check ──                                                  │
│                                                                       │
│ Protocol: [HTTP ▼] (HTTP or TCP)                                  │
│ Path: [/health]                                                     │
│ Interval: [5] seconds   (1-20)                                    │
│ Timeout: [2] seconds    (1-20)                                    │
│ Healthy threshold: [1]  (1-20, checks before healthy)            │
│ Unhealthy threshold: [5] (1-20, checks before unhealthy)        │
│ ⚡ Failed health check → instance replaced automatically.         │
│                                                                       │
│ ── Security ──                                                      │
│                                                                       │
│ Instance role: [AppRunnerInstanceRole ▼]                          │
│ ⚡ IAM role assumed by your running containers.                     │
│   Add permissions for AWS services: S3, DynamoDB, SQS, etc.    │
│   This is NOT the ECR access role (that's for pulling images).  │
│                                                                       │
│ Encryption:                                                          │
│ ● AWS managed key                                                  │
│ ○ Customer managed key (KMS)                                      │
│                                                                       │
│ ── Networking ──                                                    │
│                                                                       │
│ Incoming traffic:                                                    │
│ ● Public (accessible from internet) ✅                              │
│ ○ Private (VPC Ingress — internal only via VPC endpoint)        │
│                                                                       │
│ Outgoing traffic:                                                    │
│ ● Public (can reach internet directly)                             │
│ ○ VPC (route through VPC — access private resources) ✅           │
│   VPC connector: [my-vpc-connector ▼]                             │
│   ⚡ VPC Connector: Lets App Runner access VPC resources.          │
│     RDS, ElastiCache, private APIs — via private subnet.        │
│     Outbound internet: Requires NAT Gateway in VPC.             │
│                                                                       │
│ ── Observability ──                                                 │
│ ☑ Enable tracing (AWS X-Ray)                                      │
│ ⚡ Distributed tracing for request flows.                           │
│                                                                       │
│ [Next]                                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│   STEP 4: REVIEW AND CREATE                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Review all settings and [Create & deploy]                          │
│                                                                       │
│ ⚡ First deployment: ~3-5 min (image) or ~5-10 min (source code). │
│                                                                       │
│ Result:                                                              │
│ ├── Default domain: https://xxxxxxxx.ap-south-1.awsapprunner.com│
│ │   ⚡ Auto-generated, HTTPS by default, always on.              │
│ │                                                                 │
│ ├── Subsequent deploys: Push to ECR or GitHub → auto-deploy     │
│ ├── Logs: CloudWatch Logs (auto-configured)                     │
│ └── Metrics: CloudWatch Metrics (requests, latency, instances)  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: apprunner.yaml (Configuration File)

```yaml
# apprunner.yaml — place in repo root
# Used when source = GitHub + "Use a configuration file"

version: 1.0
runtime: nodejs18

build:
  commands:
    pre-build:
      - echo "Installing dependencies..."
    build:
      - npm ci
      - npm run build
    post-build:
      - echo "Build complete!"

run:
  runtime-version: 18.18.0
  command: npm start
  network:
    port: 8080
    env: PORT           # App Runner sets this env var
  env:
    - name: NODE_ENV
      value: production
    - name: DB_HOST
      value: mydb.cluster-xxx.ap-south-1.rds.amazonaws.com
    - name: DB_PASSWORD
      value: "arn:aws:secretsmanager:ap-south-1:123456789:secret:db-pass"
      # Secrets Manager ARN — auto-fetched at startup
```

```
⚡ apprunner.yaml fields:

version: 1.0 (always 1.0)
runtime: nodejs18 | python3 | java11 | dotnet6 | php81 | ruby31 | go1

build:
  commands:
    pre-build:  ← Before build (install system deps, etc.)
    build:      ← Main build (npm ci, pip install, mvn package)
    post-build: ← After build (cleanup, copy files)

run:
  runtime-version: ← Specific runtime version
  command:          ← Start command
  network:
    port:           ← Port your app listens on
    env:            ← Env var name for port (default: PORT)
  env:              ← Environment variables (plain text or ARN)

⚡ If using container image, you don't need apprunner.yaml.
  Everything is configured in the Dockerfile + console.
```

---

## Part 4: Custom Domains

```
Service → Custom domains → Link domain

┌─────────────────────────────────────────────────────────────────────┐
│           CUSTOM DOMAINS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Domain name: [app.example.com]                                      │
│                                                                       │
│ ☑ Link www.app.example.com also                                    │
│                                                                       │
│ Validation:                                                          │
│ ⚡ App Runner requires DNS validation:                               │
│                                                                       │
│ Add these DNS records:                                               │
│ ┌────────┬──────────────────────────────┬─────────────────────────┐│
│ │ Type    │ Name                          │ Value                   ││
│ ├────────┼──────────────────────────────┼─────────────────────────┤│
│ │ CNAME   │ _xxxxx.app.example.com       │ _xxxxx.acm-validations ││
│ │ CNAME   │ app.example.com              │ xxxxxx.awsapprunner.com││
│ └────────┴──────────────────────────────┴─────────────────────────┘│
│                                                                       │
│ ⚡ First CNAME: Certificate validation (ACM).                       │
│   Second CNAME: Route traffic to App Runner.                     │
│                                                                       │
│ ⚡ TLS certificate: Auto-provisioned by AWS Certificate Manager.   │
│   ├── Free (included with App Runner)                            │
│   ├── Auto-renewed before expiry                                 │
│   ├── Covers your custom domain + www subdomain                 │
│   └── Validation takes 5-30 minutes                             │
│                                                                       │
│ After validation:                                                    │
│ ├── https://app.example.com → your App Runner service          │
│ ├── https://www.app.example.com → your App Runner service      │
│ ├── https://xxxxx.awsapprunner.com → still works (default)    │
│ └── HTTP → auto-redirected to HTTPS                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: VPC Connector (Access Private Resources)

```
┌─────────────────────────────────────────────────────────────────────┐
│           VPC CONNECTOR                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: App Runner runs outside your VPC.                         │
│ How to access RDS, ElastiCache, private APIs in VPC?             │
│                                                                       │
│ Solution: VPC Connector                                             │
│                                                                       │
│ Console → App Runner → VPC connectors → Create                    │
│                                                                       │
│ Name: [vpc-connector-prod]                                          │
│ VPC: [vpc-prod ▼]                                                  │
│ Subnets:                                                             │
│ ☑ subnet-private-1a (10.0.1.0/24)                                │
│ ☑ subnet-private-1b (10.0.2.0/24)                                │
│ ⚡ Use PRIVATE subnets! (Not public.)                               │
│   App Runner creates ENIs in these subnets.                     │
│                                                                       │
│ Security groups: [sg-app-runner ▼]                                 │
│ ⚡ SG controls what the App Runner service can access.              │
│   Allow outbound to: RDS (3306/5432), ElastiCache (6379), etc.  │
│                                                                       │
│ Architecture:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ App Runner service                                          │  │
│ │    │ VPC Connector (ENIs in private subnets)               │  │
│ │    ├───→ RDS (10.0.3.x:5432) ✅                            │  │
│ │    ├───→ ElastiCache (10.0.3.x:6379) ✅                    │  │
│ │    ├───→ Internal ALB (10.0.1.x:443) ✅                    │  │
│ │    └───→ Internet? Only via NAT Gateway!                    │  │
│ │                                                              │  │
│ │ ⚠️ If using VPC connector for outbound:                     │  │
│ │   ALL outbound traffic goes through VPC.                   │  │
│ │   Need NAT Gateway for internet access.                    │  │
│ │   Without NAT: Can't reach external APIs, npm, etc.       │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ VPC Ingress (private App Runner):                                  │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Service → Networking → Incoming: Private                    │  │
│ │                                                              │  │
│ │ Creates an Interface VPC Endpoint for App Runner.          │  │
│ │ Service only accessible from within VPC.                   │  │
│ │ No public URL generated.                                    │  │
│ │                                                              │  │
│ │ Use case: Internal microservices, admin APIs.              │  │
│ │                                                              │  │
│ │ Access via: VPC Endpoint DNS name or private hosted zone.  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Observability

```
┌─────────────────────────────────────────────────────────────────────┐
│           LOGS, METRICS & TRACING                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Logs (CloudWatch Logs — automatic):                                │
│ ├── Application logs: stdout/stderr from your container         │
│ │   Log group: /aws/apprunner/web-app-prod/.../application      │
│ │                                                                 │
│ ├── Service logs: Deployment events, scaling events             │
│ │   Log group: /aws/apprunner/web-app-prod/.../service          │
│ │                                                                 │
│ └── Console → Service → Logs tab → View in real time           │
│                                                                       │
│ Metrics (CloudWatch Metrics — automatic):                          │
│ ├── Requests: Total count                                        │
│ ├── 2xxStatusResponses, 4xxStatusResponses, 5xxStatusResponses │
│ ├── RequestLatency: p50, p90, p99                               │
│ ├── ActiveInstances: Current running instances                  │
│ ├── ConcurrentRequests: Current in-flight requests              │
│ └── CPU/Memory utilization per instance                          │
│                                                                       │
│ Tracing (X-Ray — opt-in):                                          │
│ ├── Enable: Service → Observability → Tracing → AWS X-Ray     │
│ ├── Auto-instruments incoming requests                          │
│ ├── Add X-Ray SDK to app for downstream tracing                │
│ │   (DynamoDB, S3, HTTP calls, SQL queries)                    │
│ └── View traces: CloudWatch → X-Ray → Service Map             │
│                                                                       │
│ Alarms:                                                              │
│ ├── CloudWatch Alarms on App Runner metrics:                    │
│ │   5xxStatusResponses > 10 in 5 min → SNS → PagerDuty       │
│ │   RequestLatency p99 > 1000ms → SNS → Slack                │
│ └── No built-in alerting in App Runner itself.                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform

```hcl
# App Runner Service — from ECR image
resource "aws_apprunner_service" "web" {
  service_name = "web-app-prod"

  source_configuration {
    authentication_configuration {
      access_role_arn = aws_iam_role.apprunner_ecr.arn
    }

    image_repository {
      image_identifier      = "${aws_ecr_repository.web.repository_url}:latest"
      image_repository_type = "ECR"

      image_configuration {
        port = "8080"

        runtime_environment_variables = {
          NODE_ENV = "production"
          DB_HOST  = aws_db_instance.main.address
        }

        runtime_environment_secrets = {
          DB_PASSWORD = aws_secretsmanager_secret.db_password.arn
          API_KEY     = aws_ssm_parameter.api_key.arn
        }
      }
    }

    auto_deployments_enabled = true
  }

  instance_configuration {
    cpu               = "1024"   # 1 vCPU (256, 512, 1024, 2048, 4096)
    memory            = "2048"   # 2 GB  (512, 1024, 2048, 3072, 4096, 6144, 8192, 12288)
    instance_role_arn = aws_iam_role.apprunner_instance.arn
  }

  auto_scaling_configuration_arn = aws_apprunner_auto_scaling_configuration_version.prod.arn

  health_check_configuration {
    protocol            = "HTTP"
    path                = "/health"
    interval            = 5
    timeout             = 2
    healthy_threshold   = 1
    unhealthy_threshold = 5
  }

  network_configuration {
    egress_configuration {
      egress_type       = "VPC"
      vpc_connector_arn = aws_apprunner_vpc_connector.main.arn
    }

    ingress_configuration {
      is_publicly_accessible = true
    }
  }

  observability_configuration {
    observability_configuration_arn = aws_apprunner_observability_configuration.xray.arn
    observability_enabled           = true
  }

  tags = {
    Environment = "prod"
  }
}

# Auto Scaling Configuration
resource "aws_apprunner_auto_scaling_configuration_version" "prod" {
  auto_scaling_configuration_name = "prod-scaling"

  max_concurrency = 100   # Requests per instance before scaling
  max_size        = 10    # Max instances
  min_size        = 1     # Min instances (0 = scale to zero)
}

# VPC Connector
resource "aws_apprunner_vpc_connector" "main" {
  vpc_connector_name = "vpc-connector-prod"
  subnets            = [aws_subnet.private_1a.id, aws_subnet.private_1b.id]
  security_groups    = [aws_security_group.apprunner.id]
}

# Custom Domain
resource "aws_apprunner_custom_domain_association" "app" {
  service_arn          = aws_apprunner_service.web.arn
  domain_name          = "app.example.com"
  enable_www_subdomain = true
}

# Observability (X-Ray)
resource "aws_apprunner_observability_configuration" "xray" {
  observability_configuration_name = "xray-tracing"

  trace_configuration {
    vendor = "AWSXRAY"
  }
}

# ECR Access Role (for pulling images)
resource "aws_iam_role" "apprunner_ecr" {
  name = "AppRunnerECRAccessRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "build.apprunner.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "apprunner_ecr" {
  role       = aws_iam_role.apprunner_ecr.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSAppRunnerServicePolicyForECRAccess"
}

# Instance Role (for app to access AWS services)
resource "aws_iam_role" "apprunner_instance" {
  name = "AppRunnerInstanceRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "tasks.apprunner.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "apprunner_instance" {
  name = "AppRunnerInstancePolicy"
  role = aws_iam_role.apprunner_instance.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
        ]
        Resource = "${aws_s3_bucket.uploads.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
        ]
        Resource = aws_secretsmanager_secret.db_password.arn
      },
      {
        Effect = "Allow"
        Action = [
          "ssm:GetParameter",
        ]
        Resource = aws_ssm_parameter.api_key.arn
      }
    ]
  })
}

# Security Group for VPC Connector
resource "aws_security_group" "apprunner" {
  name   = "sg-apprunner"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = [aws_subnet.db_1a.cidr_block, aws_subnet.db_1b.cidr_block]
    description = "PostgreSQL"
  }

  egress {
    from_port   = 6379
    to_port     = 6379
    protocol    = "tcp"
    cidr_blocks = [aws_subnet.cache_1a.cidr_block, aws_subnet.cache_1b.cidr_block]
    description = "Redis"
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS outbound (via NAT)"
  }
}
```

---

## Part 8: AWS CLI Reference

```bash
# ═══ Service management ═══

# Create service from ECR image
aws apprunner create-service \
  --service-name web-app-prod \
  --source-configuration '{
    "AuthenticationConfiguration": {
      "AccessRoleArn": "arn:aws:iam::123456789:role/AppRunnerECRAccessRole"
    },
    "AutoDeploymentsEnabled": true,
    "ImageRepository": {
      "ImageIdentifier": "123456789.dkr.ecr.ap-south-1.amazonaws.com/web-app:latest",
      "ImageRepositoryType": "ECR",
      "ImageConfiguration": {
        "Port": "8080",
        "RuntimeEnvironmentVariables": {
          "NODE_ENV": "production"
        }
      }
    }
  }' \
  --instance-configuration '{
    "Cpu": "1024",
    "Memory": "2048",
    "InstanceRoleArn": "arn:aws:iam::123456789:role/AppRunnerInstanceRole"
  }'

# List services
aws apprunner list-services

# Describe service
aws apprunner describe-service \
  --service-arn arn:aws:apprunner:ap-south-1:123456789:service/web-app-prod/xxx

# Update service (change CPU/memory)
aws apprunner update-service \
  --service-arn arn:aws:apprunner:ap-south-1:123456789:service/web-app-prod/xxx \
  --instance-configuration '{
    "Cpu": "2048",
    "Memory": "4096"
  }'

# Trigger deployment manually
aws apprunner start-deployment \
  --service-arn arn:aws:apprunner:ap-south-1:123456789:service/web-app-prod/xxx

# Pause service (stop instances, reduce cost)
aws apprunner pause-service \
  --service-arn arn:aws:apprunner:ap-south-1:123456789:service/web-app-prod/xxx

# Resume service
aws apprunner resume-service \
  --service-arn arn:aws:apprunner:ap-south-1:123456789:service/web-app-prod/xxx

# Delete service
aws apprunner delete-service \
  --service-arn arn:aws:apprunner:ap-south-1:123456789:service/web-app-prod/xxx

# ═══ Custom domain ═══

# Associate domain
aws apprunner associate-custom-domain \
  --service-arn arn:aws:apprunner:... \
  --domain-name app.example.com \
  --enable-www-subdomain

# List custom domains
aws apprunner describe-custom-domains \
  --service-arn arn:aws:apprunner:...

# Disassociate domain
aws apprunner disassociate-custom-domain \
  --service-arn arn:aws:apprunner:... \
  --domain-name app.example.com

# ═══ Auto scaling ═══

# Create auto scaling config
aws apprunner create-auto-scaling-configuration \
  --auto-scaling-configuration-name prod-scaling \
  --max-concurrency 100 \
  --min-size 1 \
  --max-size 10

# ═══ VPC connector ═══

# Create VPC connector
aws apprunner create-vpc-connector \
  --vpc-connector-name vpc-connector-prod \
  --subnets subnet-xxx subnet-yyy \
  --security-groups sg-xxx

# ═══ Connections (GitHub) ═══

# Create GitHub connection
aws apprunner create-connection \
  --connection-name github-connection \
  --provider-type GITHUB

# List connections
aws apprunner list-connections
```

---

## Part 9: Real-World Patterns

### Startup

```
Simple web app + API on App Runner:

Services:
├── web-app-prod (frontend — Next.js SSR)
│   ├── Source: GitHub (auto-deploy on push to main)
│   ├── Runtime: Node.js 18
│   ├── CPU: 1 vCPU, Memory: 2 GB
│   ├── Scaling: 1-5 instances, 100 concurrency
│   ├── Custom domain: app.startup.com
│   └── Cost: ~$50-80/month
│
├── api-prod (backend — Express.js)
│   ├── Source: ECR (auto-deploy on image push)
│   ├── CPU: 1 vCPU, Memory: 2 GB
│   ├── Scaling: 1-5 instances, 50 concurrency
│   ├── VPC connector → RDS PostgreSQL, ElastiCache
│   ├── Instance role → S3, SES, Secrets Manager
│   ├── Custom domain: api.startup.com
│   └── Cost: ~$60-120/month

Environment variables:
├── Plain text: NODE_ENV, CORS_ORIGIN, LOG_LEVEL
├── Secrets Manager: DB_PASSWORD, JWT_SECRET
└── SSM: API_KEY, STRIPE_KEY

CI/CD: GitHub Actions → Build image → Push ECR → auto-deploy
No Kubernetes, no ECS task defs, no infra management!

Total: ~$110-200/month
```

### Mid-Size

```
Multiple microservices on App Runner:

Services:
├── web-frontend (Next.js, GitHub auto-deploy)
│   1 vCPU / 2 GB, scale: 2-8
├── api-gateway (Node.js, ECR auto-deploy)
│   2 vCPU / 4 GB, scale: 2-10
├── auth-service (Node.js, ECR)
│   1 vCPU / 2 GB, scale: 1-5
├── notification-service (Python, ECR)
│   0.5 vCPU / 1 GB, scale: 1-3
├── file-service (Go, ECR)
│   1 vCPU / 2 GB, scale: 1-5
└── admin-panel (React SSR, GitHub, private via VPC Ingress)
    0.5 vCPU / 1 GB, scale: 0-2 (scale to zero — rarely used)

Networking:
├── All services: VPC connector → private subnets
├── RDS, ElastiCache, OpenSearch: Private only
├── Public services: web-frontend, api-gateway
├── Private services: admin-panel (VPC Ingress)
├── NAT Gateway: For outbound internet (3rd party APIs)
└── Custom domains: app.company.com, api.company.com

Secrets:
├── All secrets in Secrets Manager (rotated)
├── Feature flags in SSM Parameter Store
└── Instance roles: Least privilege per service

Observability:
├── X-Ray tracing across all services
├── CloudWatch Logs: Structured JSON logging
├── CloudWatch Alarms: 5xx rate, latency, instance count
├── CloudWatch dashboards: Per-service metrics
└── PagerDuty integration: Critical alerts

CI/CD:
├── GitHub Actions → Docker build → ECR push → auto-deploy
├── Staging: Separate App Runner services (staging-xxx)
├── Production: Blue-green via service swap or canary via API
└── Rollback: Redeploy previous ECR image tag

Cost: $500-1,500/month
```

### Enterprise

```
App Runner as part of a larger platform:

App Runner (simple services):
├── Customer-facing API (high-traffic, auto-scale)
├── Webhook receivers (event-driven, scale to zero)
├── Internal tools (admin panels, VPC Ingress private)
├── Docs site (static SSR, low traffic, scale to zero)
└── 15-20 App Runner services total

Combined with:
├── EKS: Complex K8s workloads (data plane, ML platform)
├── ECS Fargate: Long-running workers with custom scheduling
├── Lambda: Event-driven functions (S3 triggers, API Gateway)
└── App Runner: Simple HTTP services (no K8s overhead)

Governance:
├── AWS Organizations: Separate accounts per environment
├── Service Control Policies: Restrict instance sizes
├── AWS Config: Ensure all services use VPC connector
├── CloudTrail: Audit all App Runner API calls
├── Tags: cost-center, team, environment on every service
└── IAM: Federated access, no long-term credentials

Multi-region:
├── Region 1 (ap-south-1): Primary services
├── Region 2 (eu-west-1): EU-specific services
├── Route 53: Latency-based routing between regions
├── Each region: Independent App Runner services
└── RDS: Cross-region read replicas for DR

Security:
├── ECR: Private repos, vulnerability scanning, image signing
├── VPC connectors: All services in private subnets
├── WAF: CloudFront + WAF in front of App Runner
├── Secrets: All in Secrets Manager (auto-rotation)
├── Encryption: KMS customer-managed keys
├── Compliance: SOC 2, HIPAA (via BAA), PCI DSS
└── Network: Private endpoints, no public internet for backend

Cost: $3,000-8,000/month (App Runner portion)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Simplest way to run containers/code on AWS |
| Sources | ECR container image or GitHub source code |
| Build | Automatic for source code (buildpacks) |
| Deploy | Auto on git push or ECR image push |
| Scaling | 0-25 instances, concurrency-based (default 100) |
| Scale to zero | Yes (min_size = 0) |
| CPU | 0.25 - 4 vCPU |
| Memory | 0.5 - 12 GB |
| HTTPS/TLS | Automatic, free (ACM) |
| Custom domain | Yes, with auto TLS certificate |
| VPC access | VPC Connector (ENIs in private subnets) |
| Private service | VPC Ingress (Interface VPC Endpoint) |
| Health checks | HTTP or TCP |
| Logs | CloudWatch Logs (automatic) |
| Metrics | CloudWatch Metrics (automatic) |
| Tracing | AWS X-Ray (opt-in) |
| Config file | apprunner.yaml (for source code) |
| GCP equivalent | Cloud Run |
| Azure equivalent | Container Apps |

---

## What's Next?

In the next chapter, we'll dive deep into AWS Lambda — serverless functions with event-driven compute.

→ Next: [Chapter 20: Lambda Deep Dive](20-lambda-deep-dive.md)

---

*Last Updated: May 2026*
