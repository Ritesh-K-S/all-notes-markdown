# Chapter 1: Introduction to Google Cloud Platform (GCP)

---

## Table of Contents

- [What is GCP?](#what-is-gcp)
- [Brief History](#brief-history)
- [Why GCP? (For a Full-Stack Developer / DevOps Engineer)](#why-gcp-for-a-full-stack-developer--devops-engineer)
- [GCP Global Infrastructure](#gcp-global-infrastructure)
- [How GCP Services Are Organized](#how-gcp-services-are-organized)
- [GCP Console - What You See](#gcp-console---what-you-see)
- [How GCP Works in Real Companies](#how-gcp-works-in-real-companies)
- [GCP Pricing Model](#gcp-pricing-model)
- [GCP vs AWS Naming Comparison](#gcp-vs-aws-naming-comparison)
- [Key Terminology](#key-terminology)
- [Accessing GCP (4 Ways)](#accessing-gcp-4-ways)

---

## What is GCP?

Google Cloud Platform (GCP) is a suite of cloud computing services provided by Google that runs on the same infrastructure that Google uses internally for its end-user products (Google Search, Gmail, YouTube, Google Maps).

> **In Simple Terms:** You get access to the same world-class infrastructure that powers Google's own products — their network, their hardware, their data centers — and you only pay for what you use.

---

## Brief History

```
Timeline:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2008 → Google App Engine launched (PaaS, first GCP service)
2010 → Cloud Storage and BigQuery introduced
2012 → Compute Engine (IaaS VMs) launched
2013 → Google Cloud Platform brand established
2014 → Kubernetes open-sourced by Google
2015 → GKE (Google Kubernetes Engine), Cloud Pub/Sub
2017 → Cloud Spanner, Cloud Functions launched
2018 → Thomas Kurian becomes Google Cloud CEO
2019 → Anthos (multi-cloud) launched, aggressive enterprise push
2020 → Cloud Run GA, massive growth in enterprise adoption
2022 → BigQuery ML, Vertex AI matured
2024 → Gemini AI integration, 40+ regions
2026 → 40+ regions, competitive enterprise offering (~12% market share)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Why GCP? (For a Full-Stack Developer / DevOps Engineer)

| Need | GCP Solution |
|------|-------------|
| Host my web application | Compute Engine, Cloud Run, GKE, App Engine |
| Store files/images/videos | Cloud Storage (GCS) |
| Run a database | Cloud SQL, Firestore, Spanner, AlloyDB |
| Deploy containers | Cloud Run (simplest), GKE (full K8s) |
| CI/CD Pipeline | Cloud Build, Cloud Deploy, Artifact Registry |
| Monitor everything | Cloud Monitoring, Cloud Logging, Cloud Trace |
| Secure my infrastructure | IAM, VPC Firewall, Cloud Armor, KMS |
| Serve globally with low latency | Cloud CDN, Global Load Balancer |
| Run code without servers | Cloud Functions, Cloud Run |
| Message queues & events | Pub/Sub, Eventarc, Cloud Tasks |
| Big Data & Analytics | BigQuery (game-changer), Dataflow, Dataproc |

### GCP's Unique Strengths

```
What Makes GCP Different:
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  1. GOOGLE'S PRIVATE GLOBAL NETWORK                           │
│     └── Your traffic travels on Google's fiber, not public    │
│         internet → faster, more secure, more reliable         │
│                                                                │
│  2. KUBERNETES NATIVE                                          │
│     └── Google invented Kubernetes. GKE is the best           │
│         managed K8s service. Cloud Run = serverless K8s.      │
│                                                                │
│  3. BIGQUERY                                                   │
│     └── Serverless data warehouse. Query petabytes in         │
│         seconds. Nothing else comes close in simplicity.      │
│                                                                │
│  4. PER-SECOND BILLING                                        │
│     └── VMs billed per second (after 1 min minimum)           │
│         → More granular than hourly billing                   │
│                                                                │
│  5. SUSTAINED USE DISCOUNTS (Automatic)                       │
│     └── Use a VM > 25% of month → auto discount up to 30%   │
│         No commitment needed!                                  │
│                                                                │
│  6. LIVE MIGRATION                                             │
│     └── VMs are live-migrated during host maintenance         │
│         → Zero downtime (AWS reboots your instance)           │
│                                                                │
│  7. AI/ML LEADERSHIP                                           │
│     └── TPUs, Vertex AI, Gemini, pre-trained models          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## GCP Global Infrastructure

### Overview Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GCP GLOBAL INFRASTRUCTURE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                        REGIONS (40+)                         │    │
│  │  Independent geographic areas                                │    │
│  │                                                               │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │              ZONES (3+ per Region)                     │   │    │
│  │  │  Isolated deployment areas within a region            │   │    │
│  │  │                                                        │   │    │
│  │  │  ┌────────────┐  ┌────────────┐  ┌────────────┐     │   │    │
│  │  │  │ Zone A     │  │ Zone B     │  │ Zone C     │     │   │    │
│  │  │  │ us-cntrl1-a│  │ us-cntrl1-b│  │ us-cntrl1-c│     │   │    │
│  │  │  │ Compute    │  │ Compute    │  │ Compute    │     │   │    │
│  │  │  │ Storage    │  │ Storage    │  │ Storage    │     │   │    │
│  │  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘     │   │    │
│  │  │        │                │                │             │   │    │
│  │  │        └──── High-bandwidth encrypted links ──┘        │   │    │
│  │  │         (sub-millisecond latency between zones)        │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                 MULTI-REGIONS (3 main)                       │    │
│  │  ┌──────┐  ┌──────┐  ┌──────┐                              │    │
│  │  │  US  │  │  EU  │  │ ASIA │                              │    │
│  │  └──────┘  └──────┘  └──────┘                              │    │
│  │  Used for: Cloud Storage, Spanner (geo-redundant data)      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │            POINTS OF PRESENCE / EDGE (180+)                  │    │
│  │  Cloud CDN caching, Cloud Interconnect peering points        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │          GOOGLE'S PRIVATE GLOBAL NETWORK                     │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  Subsea cables (owned/leased): 20+ cable systems      │   │    │
│  │  │  Land-based fiber: Spans all continents               │   │    │
│  │  │  Traffic between regions stays on Google's network    │   │    │
│  │  │  (Never touches the public internet!)                  │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Concepts Explained

#### 1. Region
A **Region** is a specific geographical location where GCP resources are hosted.

```
Examples of Popular Regions:
┌─────────────────────────────────────────────────────────┐
│ Region Code          │ Region Name                       │
├─────────────────────────────────────────────────────────┤
│ us-central1          │ Iowa, USA                         │ ← Common default
│ us-east1             │ South Carolina, USA               │
│ europe-west1         │ Belgium                           │
│ asia-south1          │ Mumbai, India                     │ ← India
│ asia-southeast1      │ Singapore                         │
│ europe-west3         │ Frankfurt, Germany                │
│ asia-northeast1      │ Tokyo, Japan                      │
│ australia-southeast1 │ Sydney, Australia                 │
└─────────────────────────────────────────────────────────┘
```

**How to choose a region:**
1. **Proximity to users** — Closest to your users for lower latency
2. **Compliance** — Data sovereignty requirements
3. **Service availability** — Some services are region-specific
4. **Cost** — Prices vary by region (US regions often cheapest)
5. **Carbon footprint** — Some regions run on cleaner energy

#### 2. Zone
A **Zone** is an isolated deployment area within a region. Think of it as an independent failure domain.

```
Why Zones matter (High Availability):

    Region: asia-south1 (Mumbai)
    ┌──────────────────────────────────────────────────┐
    │                                                  │
    │  ┌────────────┐  ┌────────────┐  ┌────────────┐│
    │  │ Zone A     │  │ Zone B     │  │ Zone C     ││
    │  │asia-south1-│  │asia-south1-│  │asia-south1-││
    │  │     a      │  │     b      │  │     c      ││
    │  │            │  │            │  │            ││
    │  │ App ✓      │  │ App ✓      │  │ App ✓      ││
    │  │ DB Primary │  │ DB Replica │  │            ││
    │  └────────────┘  └────────────┘  └────────────┘│
    │                                                  │
    │  If Zone A fails → App continues in Zone B & C  │
    │  DB auto-failovers to replica in Zone B         │
    └──────────────────────────────────────────────────┘

Key difference from AWS:
- GCP zone names are deterministic (us-central1-a is the same for everyone)
- AWS AZ names are randomized per account (us-east-1a might be different physical DC for you vs me)
```

#### 3. Multi-Region
A **Multi-Region** location allows storing data redundantly across multiple regions:

```
Multi-Region: US
├── Data replicated across regions within the US
├── Used for: Cloud Storage (geo-redundant)
├── Use case: Highest availability for data
└── Trade-off: Higher cost, ~200ms consistency

Multi-Region: EU
├── Data replicated across regions within Europe
└── Use case: GDPR compliance + high availability

Multi-Region: ASIA
├── Data replicated across Asia-Pacific regions
└── Use case: Asia-focused applications
```

#### 4. Google's Private Network (Premium vs Standard Tier)

```
PREMIUM TIER (Default):
User → Nearest Google Edge → Google's Private Network → Your GCP resource
(Traffic enters Google's network ASAP, stays on private fiber)
→ Lower latency, higher reliability, slightly higher cost

STANDARD TIER:
User → Public Internet → ... → Google's Network (at destination region) → Your resource
(Traffic travels mostly on public internet)
→ Higher latency, lower cost (useful for non-latency-sensitive workloads)

                    ┌─── Premium Tier ─────────────────────────┐
                    │                                           │
User (India) → Edge (Mumbai) ═══ Google Fiber ═══> GCP VM (US)│
                    │                                           │
                    └──────────────────────────────────────────┘

                    ┌─── Standard Tier ────────────────────────┐
                    │                                           │
User (India) → Public Internet → ... → GCP Region (US) → VM  │
                    │                                           │
                    └──────────────────────────────────────────┘
```

---

## How GCP Services Are Organized

When you open the GCP Console, services are grouped in the left sidebar:

```
GCP Console Navigation (Left Sidebar):
├── Dashboard (Project overview)
├── APIs & Services
├── IAM & Admin
│   ├── IAM, Service Accounts, Organization Policies, Roles
├── Compute
│   ├── Compute Engine (VMs)
│   ├── GKE (Kubernetes)
│   ├── Cloud Run (Serverless containers)
│   ├── Cloud Functions (FaaS)
│   ├── App Engine (PaaS)
├── Storage
│   ├── Cloud Storage (Object storage)
│   ├── Filestore (Managed NFS)
│   ├── Persistent Disk (Block storage)
├── Databases
│   ├── Cloud SQL (MySQL/PostgreSQL/SQL Server)
│   ├── Firestore (Document DB)
│   ├── Bigtable (Wide-column)
│   ├── Spanner (Global relational)
│   ├── Memorystore (Redis/Memcached)
│   ├── AlloyDB (PostgreSQL-compatible)
├── Networking
│   ├── VPC Networks, Firewall, Routes
│   ├── Load Balancing, Cloud CDN, Cloud DNS
│   ├── Cloud NAT, Cloud Armor, Cloud Router
│   ├── Connectivity (VPN, Interconnect)
├── CI/CD
│   ├── Cloud Build, Artifact Registry, Cloud Deploy
├── Operations (Monitoring)
│   ├── Monitoring, Logging, Trace, Error Reporting, Profiler
├── Security
│   ├── Security Command Center, Secret Manager, KMS
│   ├── BinaryAuthorization, Certificate Manager
├── Serverless
│   ├── Cloud Functions, Cloud Run, App Engine, Workflows
├── Analytics
│   ├── BigQuery, Dataflow, Dataproc, Composer, Pub/Sub
├── AI & Machine Learning
│   ├── Vertex AI, AutoML, AI Platform
└── Migration
    ├── Migration Center, Database Migration Service
```

---

## GCP Console - What You See

When you log into the GCP Console (https://console.cloud.google.com):

```
┌─────────────────────────────────────────────────────────────────────┐
│ ☰ │ Google Cloud  │ [Project Selector ▼] │ [Search 🔍] │ [⚙️] [👤]│
├────┬────────────────────────────────────────────────────────────────┤
│    │                                                                 │
│ N  │  ┌─── Project Dashboard ────────────────────────────────────┐ │
│ A  │  │                                                          │ │
│ V  │  │  Project Info              API Requests                  │ │
│    │  │  ├── Project name          ├── Requests/sec graph        │ │
│ B  │  │  ├── Project ID            └── Error rate                │ │
│ A  │  │  └── Project number                                      │ │
│ R  │  │                                                          │ │
│    │  │  Resources                 Billing                       │ │
│ (  │  │  ├── Compute Engine: 3     ├── Current charges           │ │
│ L  │  │  ├── Cloud Storage: 5      └── Estimated monthly         │ │
│ E  │  │  └── Cloud SQL: 2                                        │ │
│ F  │  │                                                          │ │
│ T  │  │  Monitoring                Google Cloud Status           │ │
│ )  │  │  ├── Error count           └── All services normal ✓    │ │
│    │  │  └── Alert policies                                      │ │
│    │  │                                                          │ │
│    │  └──────────────────────────────────────────────────────────┘ │
│    │                                                                 │
└────┴────────────────────────────────────────────────────────────────┘
```

### Important UI Elements:

| Element | Location | Purpose |
|---------|----------|---------|
| **Project Selector** | Top bar | Switch between projects (CRITICAL: wrong project = can't see resources) |
| **Navigation Menu (☰)** | Top-left | Access all services (hamburger menu) |
| **Search Bar** | Top-center | Search services, resources, docs, even settings |
| **Cloud Shell** | Top-right (terminal icon) | Browser-based terminal with `gcloud` CLI pre-configured |
| **Notifications** | Bell icon | Alerts, operations progress |
| **Pin services** | Nav sidebar | Pin frequently used services to sidebar |

### Critical Difference: Projects in GCP

```
⚠️  IMPORTANT CONCEPT:

In GCP, EVERYTHING lives inside a PROJECT.
- A project is the fundamental organizing entity
- Each project has: Name, ID (unique globally), Number
- Resources in one project can't see resources in another (by default)
- Billing is tracked per project
- APIs must be enabled per project

This is different from AWS where resources live in a Region within an Account.
In GCP: Organization → Folder → Project → Resources

You ALWAYS need to select the correct project before doing anything.
```

---

## How GCP Works in Real Companies

### Typical Company Setup

```
Real-World GCP Organization Structure:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Company: "TechCorp Inc." (domain: techcorp.com)

Organization: techcorp.com (tied to Google Workspace / Cloud Identity)
│
├── Folder: Production
│   ├── Project: techcorp-prod-frontend
│   │   ├── Cloud Run services (web apps)
│   │   ├── Cloud CDN + Global LB
│   │   └── Cloud Storage (static assets)
│   ├── Project: techcorp-prod-backend
│   │   ├── GKE cluster (microservices)
│   │   ├── Cloud SQL (databases)
│   │   └── Memorystore (caching)
│   ├── Project: techcorp-prod-data
│   │   ├── BigQuery (analytics)
│   │   ├── Pub/Sub (event streaming)
│   │   └── Dataflow (ETL pipelines)
│   └── Project: techcorp-prod-ml
│       ├── Vertex AI (ML models)
│       └── Cloud Storage (training data)
│
├── Folder: Shared Services
│   ├── Project: techcorp-shared-networking
│   │   ├── Shared VPC (host project)
│   │   ├── Cloud DNS
│   │   └── Cloud Interconnect
│   ├── Project: techcorp-shared-cicd
│   │   ├── Cloud Build (pipelines)
│   │   ├── Artifact Registry (Docker/npm)
│   │   └── Secret Manager
│   └── Project: techcorp-shared-monitoring
│       ├── Cloud Monitoring (metrics scope)
│       └── Cloud Logging (aggregated)
│
├── Folder: Non-Production
│   ├── Project: techcorp-staging
│   └── Project: techcorp-dev
│
└── Folder: Sandbox
    ├── Project: techcorp-sandbox-john
    └── Project: techcorp-sandbox-jane
```

### Why This Structure?

| Concept | Purpose |
|---------|---------|
| **Organization** | Top-level node, tied to company domain, central policy management |
| **Folders** | Group projects by environment/team/function, inherit policies |
| **Projects** | Isolation boundary for billing, access, and resources |
| **Shared VPC** | One networking project shared across multiple service projects |
| **Separate data project** | Tight access control on sensitive data |

### Day-to-Day Workflow (Developer/DevOps)

```
Your daily interaction with GCP:

Morning:
  1. Open Cloud Console → select your project
  2. Check Cloud Monitoring dashboard → uptime, error rates
  3. Review Cloud Build → any failed builds?

Development:
  4. Push code to GitHub/Cloud Source Repos
  5. Cloud Build trigger fires automatically
  6. Build runs: test → build Docker image → push to Artifact Registry
  7. Cloud Deploy promotes to staging environment

Release:
  8. Cloud Deploy promotion requires approval
  9. Approved → rolls out to production (Cloud Run/GKE)
  10. Canary/blue-green deployment with traffic splitting

Troubleshooting:
  11. Cloud Logging → filter and search application logs
  12. Cloud Trace → visualize request latency across services
  13. Error Reporting → auto-grouped errors with stack traces
  14. Cloud Audit Logs → "who did what, when?"
```

---

## GCP Pricing Model

### Core Principles

```
GCP Pricing Philosophy:
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  1. PAY-AS-YOU-GO (Per-Second Billing)                        │
│     └── VMs billed per second (1 min minimum)                 │
│         Most granular billing among cloud providers            │
│                                                                │
│  2. SUSTAINED USE DISCOUNTS (Automatic!)                      │
│     └── Use a VM > 25% of month → automatic discount          │
│         Up to 30% off — no commitment required                │
│                                                                │
│  3. COMMITTED USE DISCOUNTS (CUDs)                            │
│     └── Commit to 1 or 3 years → up to 57% off               │
│         For predictable workloads                             │
│                                                                │
│  4. PREEMPTIBLE / SPOT VMs                                    │
│     └── Up to 60-91% off for fault-tolerant workloads         │
│         Can be terminated with 30s notice                     │
│                                                                │
│  5. FREE TIER                                                  │
│     ├── $300 credit for 90 days (new accounts)               │
│     └── Always Free tier (many services)                      │
│                                                                │
│  6. FLAT-RATE & ON-DEMAND (BigQuery)                          │
│     └── Choose between per-query pricing or flat monthly      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Free Tier Highlights (Useful for Learning)

| Service | Free Tier Offer | Duration |
|---------|----------------|----------|
| Compute Engine | 1 e2-micro VM (us regions) | Always free |
| Cloud Storage | 5 GB (US regions, Standard) | Always free |
| BigQuery | 1 TB queries, 10 GB storage / month | Always free |
| Cloud Functions | 2M invocations / month | Always free |
| Cloud Run | 2M requests / month | Always free |
| Firestore | 1 GB storage, 50K reads/day | Always free |
| Pub/Sub | 10 GB messages / month | Always free |
| Cloud Build | 120 build-minutes / day | Always free |
| Artifact Registry | 0.5 GB storage | Always free |
| **New Account Credit** | **$300 for 90 days** | One-time |

---

## GCP vs AWS Naming Comparison

| AWS Service | GCP Equivalent | Notes |
|-------------|---------------|-------|
| EC2 | Compute Engine | VMs |
| Lambda | Cloud Functions / Cloud Run | Cloud Run is more flexible |
| ECS/EKS | GKE | GKE is considered best K8s |
| S3 | Cloud Storage | Similar feature set |
| RDS | Cloud SQL | Managed relational DBs |
| DynamoDB | Firestore / Bigtable | Firestore for documents, Bigtable for wide-column |
| Aurora | AlloyDB / Cloud Spanner | Spanner = globally distributed |
| CloudFront | Cloud CDN | Part of Load Balancing |
| Route 53 | Cloud DNS | Similar |
| VPC | VPC Network | GCP VPC is global by default! |
| IAM | IAM | Similar but different model |
| CloudWatch | Cloud Monitoring + Logging | Split into separate services |
| CloudFormation | Deployment Manager / Terraform | Terraform preferred on GCP |
| SQS/SNS | Pub/Sub | Unified messaging |
| CodePipeline | Cloud Build + Cloud Deploy | |
| API Gateway | API Gateway / Apigee | Apigee for enterprise |

---

## Key Terminology

| Term | Meaning |
|------|---------|
| **Project** | Fundamental organizing entity. All resources belong to a project |
| **Project ID** | Globally unique identifier (e.g., `my-project-12345`) — cannot change |
| **Organization** | Top-level node tied to Google Workspace domain |
| **Folder** | Grouping mechanism for projects (like nested folders) |
| **Region** | Geographic area (e.g., `us-central1`) |
| **Zone** | Isolated area within a region (e.g., `us-central1-a`) |
| **gcloud** | GCP command-line tool (like AWS CLI) |
| **gsutil** | Cloud Storage command-line tool (legacy, now part of gcloud) |
| **bq** | BigQuery command-line tool |
| **Service Account** | Identity for applications/services (not humans) |
| **API** | Must be enabled per project before using a service |
| **Label** | Key-value metadata for organizing resources (like AWS Tags) |

---

## Accessing GCP (4 Ways)

```
┌─────────────────────────────────────────────────────────────────┐
│                  WAYS TO INTERACT WITH GCP                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Google Cloud Console (Web Browser)                           │
│     └── GUI → Best for learning, exploring, one-off tasks       │
│     └── URL: https://console.cloud.google.com                   │
│                                                                   │
│  2. gcloud CLI (Command Line Interface)                          │
│     └── Terminal commands → Best for scripting, automation       │
│     └── Example: gcloud compute instances list                  │
│                                                                   │
│  3. Client Libraries / SDKs (Programmatic)                       │
│     └── Code libraries → Best for application integration       │
│     └── Languages: Python, Node.js, Java, Go, C#, Ruby, PHP    │
│                                                                   │
│  4. Cloud Shell (Browser-based Terminal)                          │
│     └── Pre-authenticated, 5GB home directory, free             │
│     └── Includes: gcloud, kubectl, terraform, docker pre-installed│
│                                                                   │
│  (All methods call the same underlying GCP REST APIs)            │
│                                                                   │
│  Bonus: Terraform (IaC) → Most common in production             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## GCP's Unique Concepts (vs AWS/Azure)

### 1. Global VPC
```
AWS/Azure: VPC/VNet is REGIONAL (one per region)
GCP:       VPC is GLOBAL (spans all regions automatically!)

GCP VPC:
┌──────────────────────────────────────────────────────────┐
│  VPC: "my-vpc" (GLOBAL)                                  │
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │ Subnet: us-central1│  │ Subnet: asia-south1│           │
│  │ 10.0.1.0/24     │  │ 10.0.2.0/24     │               │
│  │ (Regional)      │  │ (Regional)      │               │
│  └─────────────────┘  └─────────────────┘               │
│                                                          │
│  VMs in different regions can communicate via internal   │
│  IPs without any peering or special setup!              │
└──────────────────────────────────────────────────────────┘
```

### 2. Projects as Isolation Boundary
```
AWS:  Account = Isolation boundary
GCP:  Project = Isolation boundary

One GCP Organization can have thousands of projects.
Each project has its own:
- Billing
- IAM policies
- Enabled APIs
- Quotas
- Resources
```

### 3. API Enablement
```
⚠️  In GCP, you must ENABLE an API before using a service!

Example: Before creating a VM, you need:
  gcloud services enable compute.googleapis.com

Common APIs to enable:
- compute.googleapis.com (Compute Engine)
- container.googleapis.com (GKE)
- run.googleapis.com (Cloud Run)
- cloudbuild.googleapis.com (Cloud Build)
- sqladmin.googleapis.com (Cloud SQL)
```

---

## What's Next?

In the next chapter, we'll set up your GCP account (personal and organization-level), create your first project, understand the resource hierarchy in depth, and learn how companies structure their GCP Organizations.

→ Next: [Chapter 2: Account Setup & Organization](02-account-setup-and-organization.md)

---

*Last Updated: May 2026*
