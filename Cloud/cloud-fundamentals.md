# ☁️ Cloud Computing — AWS vs Azure vs GCP — Production-Level Notes

> **Goal:** Master cloud concepts across all three providers with deep comparisons, flows, and production-grade understanding.

---

## 📌 Table of Contents

1. [What is Cloud Computing](#1-what-is-cloud-computing)
2. [Cloud Service Models (IaaS / PaaS / SaaS)](#2-cloud-service-models)
3. [Cloud Deployment Models](#3-cloud-deployment-models)
4. [Global Infrastructure Hierarchy](#4-global-infrastructure-hierarchy)
5. [Account & Resource Hierarchy](#5-account--resource-hierarchy)
6. [Practical Company Setup — From Day Zero to Production](#6-practical-company-setup--from-day-zero-to-production)
7. [Core Service Categories — Full Comparison](#7-core-service-categories--full-comparison)
8. [Identity & Access Management (IAM)](#8-identity--access-management)
9. [Networking Deep Dive](#9-networking-deep-dive)
10. [Compute Deep Dive](#10-compute-deep-dive)
11. [Storage Deep Dive](#11-storage-deep-dive)
12. [Database Deep Dive](#12-database-deep-dive)
13. [Serverless & Event-Driven](#13-serverless--event-driven)
14. [Containers & Orchestration](#14-containers--orchestration)
15. [Monitoring, Logging & Observability](#15-monitoring-logging--observability)
16. [Security & Compliance](#16-security--compliance)
17. [CI/CD & DevOps](#17-cicd--devops)
18. [Cost Management](#18-cost-management)
19. [Production Architecture Flow](#19-production-architecture-flow)
20. [Shared Responsibility Model](#20-shared-responsibility-model)
21. [Well-Architected Framework](#21-well-architected-framework)
22. [High Availability & Disaster Recovery](#22-high-availability--disaster-recovery)
23. [Data Transfer & Egress Costs](#23-data-transfer--egress-costs)
24. [Compliance Frameworks & SLAs](#24-compliance-frameworks--slas)

---

## 1. What is Cloud Computing

Cloud computing is **on-demand delivery** of IT resources (compute, storage, networking, databases) over the internet with **pay-as-you-go** pricing.

### Five Essential Characteristics (NIST Definition)

```
┌─────────────────────────────────────────────────────────────────┐
│                 5 Essential Characteristics                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. On-Demand Self-Service                                      │
│     └── Provision resources without human interaction           │
│                                                                 │
│  2. Broad Network Access                                        │
│     └── Available over the network (internet/private)           │
│                                                                 │
│  3. Resource Pooling                                            │
│     └── Multi-tenant model, shared infrastructure               │
│                                                                 │
│  4. Rapid Elasticity                                            │
│     └── Scale up/down automatically based on demand             │
│                                                                 │
│  5. Measured Service                                            │
│     └── Pay only for what you use (metered billing)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Cloud Service Models

```
┌──────────────────────────────────────────────────────────────────────────┐
│          YOU MANAGE ◄────────────────────────────── PROVIDER MANAGES    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  On-Premises    IaaS           PaaS           SaaS                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │App       │  │App       │  │App       │  │██████████│               │
│  │Data      │  │Data      │  │Data      │  │██████████│               │
│  │Runtime   │  │Runtime   │  │██████████│  │██████████│               │
│  │Middleware│  │Middleware│  │██████████│  │██████████│               │
│  │OS        │  │OS        │  │██████████│  │██████████│               │
│  │Virtualize│  │██████████│  │██████████│  │██████████│               │
│  │Servers   │  │██████████│  │██████████│  │██████████│               │
│  │Storage   │  │██████████│  │██████████│  │██████████│               │
│  │Network   │  │██████████│  │██████████│  │██████████│               │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘               │
│   You manage    You manage    You manage    Provider manages           │
│   everything    App+Data+RT   App+Data      everything                 │
│                                                                          │
│  ██ = Managed by Cloud Provider                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

### Service Model Mapping

| Layer | IaaS Example (AWS / Azure / GCP) | PaaS Example | SaaS Example |
|-------|----------------------------------|--------------|--------------|
| **AWS** | EC2, EBS, VPC | Elastic Beanstalk, RDS, Lambda | WorkMail, Chime, QuickSight |
| **Azure** | Virtual Machines, Managed Disks, VNet | App Service, Azure SQL, Functions | Microsoft 365, Dynamics 365 |
| **GCP** | Compute Engine, Persistent Disk, VPC | App Engine, Cloud SQL, Cloud Functions | Google Workspace, Looker |

---

## 3. Cloud Deployment Models

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Cloud Deployment Models                            │
├──────────┬──────────┬──────────────┬────────────────────────────────────┤
│  Public  │ Private  │   Hybrid     │   Multi-Cloud                     │
│  Cloud   │ Cloud    │   Cloud      │                                    │
├──────────┼──────────┼──────────────┼────────────────────────────────────┤
│          │          │              │                                    │
│ Shared   │ Dedicated│ Public +     │ Multiple public                   │
│ infra    │ to one   │ Private      │ cloud providers                   │
│ over     │ org      │ connected    │ used together                     │
│ internet │          │              │                                    │
│          │          │              │                                    │
│ AWS/     │ VMware/  │ AWS Outposts │ AWS + Azure + GCP                 │
│ Azure/   │ OpenStack│ Azure Arc    │ with Terraform/                   │
│ GCP      │ Private  │ Anthos       │ Pulumi                           │
│          │ VPC      │              │                                    │
└──────────┴──────────┴──────────────┴────────────────────────────────────┘
```

---

## 4. Global Infrastructure Hierarchy

This is **critical** — understanding how each provider organizes its physical infrastructure.

### AWS Global Infrastructure

```
AWS Global Infrastructure
│
├── Regions (33+)                          ← Geographically isolated
│   ├── Availability Zones (AZs) (105+)   ← 1 or more data centers
│   │   └── Data Centers                   ← Physical buildings
│   └── Local Zones                        ← Extension of a Region
│
├── Edge Locations (600+)                  ← CDN / CloudFront PoPs
│   └── Regional Edge Caches              ← Larger cache layer
│
└── Wavelength Zones                       ← 5G edge computing
    └── Embedded in Telecom networks
```

### Azure Global Infrastructure

```
Azure Global Infrastructure
│
├── Geographies (60+ regions across 140+ countries)
│   ├── Region Pairs                       ← Two regions paired for DR
│   │   ├── Region (e.g., East US)         ← Set of data centers
│   │   │   └── Availability Zones (3)     ← Physically separate DCs
│   │   └── Region (e.g., West US)
│   │       └── Availability Zones (3)
│   │
│   └── Sovereign Clouds                   ← Gov, China (isolated)
│
├── Edge Zones                             ← Low latency edge
│   └── Azure CDN PoPs (190+)
│
└── Azure Orbital                          ← Satellite ground stations
```

### GCP Global Infrastructure

```
GCP Global Infrastructure
│
├── Multi-Regions                          ← e.g., US, EU, Asia
│   └── Regions (40+)                      ← Independent geographic areas
│       └── Zones (121+)                   ← Isolated deployment areas
│           └── Clusters → Racks → Machines
│
├── Edge PoPs (187+)                       ← Cloud CDN / Interconnect
│   └── Edge Nodes                         ← Caching nodes
│
└── Subsea Cable Network                   ← Google-owned fiber
    └── Private Global Backbone (owned)
```

### Comparison Table — Global Infrastructure

| Concept | AWS | Azure | GCP |
|---------|-----|-------|-----|
| **Top Level** | Region | Geography → Region | Multi-Region → Region |
| **Isolation Unit** | Availability Zone (AZ) | Availability Zone | Zone |
| **# of Regions** | 33+ | 60+ | 40+ |
| **# of Zones** | 105+ | 3 per region (where available) | 121+ |
| **Edge/CDN** | Edge Locations (CloudFront) | Azure CDN PoPs | Cloud CDN Edge PoPs |
| **Paired DR** | No built-in pairing (manual) | Region Pairs (automatic) | Multi-Region (manual) |
| **Private Backbone** | AWS Global Network | Microsoft Global Network | Google Private Backbone |
| **Sovereign** | GovCloud | Azure Government, Azure China | Assured Workloads |

### How a Request Flows Through Global Infra

```
User Request
    │
    ▼
┌─────────────────┐
│  DNS Resolution  │  (Route 53 / Azure DNS / Cloud DNS)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Edge Location   │  Cache hit? ──YES──► Return cached content
│  (CDN PoP)       │
└────────┬────────┘
         │ NO (cache miss)
         ▼
┌─────────────────┐
│  Regional Edge   │  Larger cache layer
│  Cache           │
└────────┬────────┘
         │ MISS
         ▼
┌─────────────────────────────────────────┐
│  Region (e.g., us-east-1)              │
│  ┌──────────┐ ┌──────────┐ ┌────────┐  │
│  │  AZ-1a   │ │  AZ-1b   │ │ AZ-1c  │  │
│  │ ┌──────┐ │ │ ┌──────┐ │ │┌──────┐│  │
│  │ │ EC2  │ │ │ │ EC2  │ │ ││ EC2  ││  │
│  │ │ RDS  │ │ │ │ RDS  │ │ ││ RDS  ││  │
│  │ │ EBS  │ │ │ │ EBS  │ │ ││ EBS  ││  │
│  │ └──────┘ │ │ └──────┘ │ │└──────┘│  │
│  └──────────┘ └──────────┘ └────────┘  │
│         ▲            ▲          ▲       │
│         └────────────┼──────────┘       │
│              Load Balancer (ALB/NLB)    │
└─────────────────────────────────────────┘
```

---

## 5. Account & Resource Hierarchy

This defines **how you organize projects, billing, and permissions** — extremely important in production.

### AWS Account Hierarchy

```
AWS Organizations (Management Account)
│
├── Root
│   │
│   ├── OU: Production
│   │   ├── AWS Account: prod-app-1 (Account ID: 111111111111)
│   │   │   ├── Region: us-east-1
│   │   │   │   ├── VPC
│   │   │   │   ├── EC2 Instances
│   │   │   │   ├── RDS Databases
│   │   │   │   └── S3 Buckets (global namespace, regional storage)
│   │   │   └── Region: eu-west-1
│   │   │       └── ...
│   │   └── AWS Account: prod-app-2
│   │
│   ├── OU: Staging
│   │   └── AWS Account: staging-app-1
│   │
│   ├── OU: Development
│   │   └── AWS Account: dev-app-1
│   │
│   ├── OU: Security
│   │   └── AWS Account: security-audit
│   │
│   └── OU: Shared Services
│       ├── AWS Account: networking-hub
│       └── AWS Account: logging-central
│
├── Service Control Policies (SCPs)         ← Guardrails at OU/Account level
├── Consolidated Billing                     ← Single payer, volume discounts
└── AWS IAM Identity Center (SSO)           ← Centralized access
```

### Azure Resource Hierarchy

```
Azure Active Directory (Entra ID) Tenant
│
├── Management Groups (up to 6 levels deep)
│   │
│   ├── MG: Production
│   │   ├── Subscription: prod-sub-1 (Subscription ID: xxxx-xxxx)
│   │   │   ├── Resource Group: rg-app-eastus
│   │   │   │   ├── Virtual Machine
│   │   │   │   ├── Azure SQL Database
│   │   │   │   ├── Storage Account
│   │   │   │   └── Virtual Network
│   │   │   └── Resource Group: rg-app-westus
│   │   │       └── ...
│   │   └── Subscription: prod-sub-2
│   │
│   ├── MG: Staging
│   │   └── Subscription: staging-sub-1
│   │
│   ├── MG: Development
│   │   └── Subscription: dev-sub-1
│   │
│   └── MG: Platform
│       ├── Subscription: connectivity-sub
│       └── Subscription: identity-sub
│
├── Azure Policy                             ← Governance guardrails
├── Azure Blueprints                         ← Repeatable env setup
├── Cost Management + Billing                ← Billing accounts
└── Azure RBAC                               ← Role-based access
```

### GCP Resource Hierarchy

```
GCP Organization (linked to Cloud Identity / Workspace domain)
│
├── Folders (nestable, up to 10 levels)
│   │
│   ├── Folder: Production
│   │   ├── Project: prod-app-1 (Project ID: prod-app-1-a1b2c3)
│   │   │   ├── Region: us-central1
│   │   │   │   ├── Compute Engine VMs
│   │   │   │   ├── Cloud SQL Instances
│   │   │   │   ├── GKE Clusters
│   │   │   │   └── VPC Networks (global resource)
│   │   │   └── Region: europe-west1
│   │   │       └── ...
│   │   └── Project: prod-app-2
│   │
│   ├── Folder: Staging
│   │   └── Project: staging-app-1
│   │
│   ├── Folder: Development
│   │   └── Project: dev-app-1
│   │
│   └── Folder: Shared
│       ├── Project: shared-vpc-host
│       └── Project: logging-project
│
├── Organization Policies                    ← Constraints (like SCPs)
├── Billing Account                          ← Linked to projects
└── IAM (at Org / Folder / Project level)   ← Inherited permissions
```

### Side-by-Side Hierarchy Comparison

```
            AWS                    Azure                    GCP
            ───                    ─────                    ───

     ┌──────────────┐      ┌──────────────────┐     ┌──────────────┐
     │ Organization │      │   Entra ID       │     │ Organization │
     │ (Mgmt Acct)  │      │   Tenant         │     │ (Domain)     │
     └──────┬───────┘      └────────┬─────────┘     └──────┬───────┘
            │                       │                       │
     ┌──────┴───────┐      ┌───────┴──────────┐     ┌──────┴───────┐
     │     OUs      │      │ Management       │     │   Folders    │
     │ (Org Units)  │      │ Groups           │     │ (Nestable)   │
     └──────┬───────┘      └───────┬──────────┘     └──────┬───────┘
            │                       │                       │
     ┌──────┴───────┐      ┌───────┴──────────┐     ┌──────┴───────┐
     │   Accounts   │      │  Subscriptions   │     │   Projects   │
     │ (12-digit ID)│      │  (GUID)          │     │ (Project ID) │
     └──────┬───────┘      └───────┬──────────┘     └──────┬───────┘
            │                       │                       │
            │               ┌──────┴───────┐                │
            │               │  Resource    │                │
            │               │  Groups      │                │
            │               └──────┬───────┘                │
            │                       │                       │
     ┌──────┴───────┐      ┌───────┴──────────┐     ┌──────┴───────┐
     │  Resources   │      │   Resources      │     │  Resources   │
     │ (per region) │      │   (per RG)       │     │ (per project)│
     └──────────────┘      └──────────────────┘     └──────────────┘

     Policy: SCPs          Policy: Azure Policy     Policy: Org Policy
     Billing: Consolidated Billing: Cost Mgmt       Billing: Billing Acct
     IAM: Per-Account      IAM: RBAC (inherited)    IAM: Per-level
```

### Key Differences Table — Hierarchy

| Aspect | AWS | Azure | GCP |
|--------|-----|-------|-----|
| **Top-level entity** | Organization (Management Account) | Entra ID Tenant | Organization (Domain) |
| **Grouping mechanism** | Organizational Units (OUs) | Management Groups (6 levels) | Folders (10 levels) |
| **Billing boundary** | Account | Subscription | Project |
| **Resource container** | Account + Region | Resource Group | Project |
| **Policy mechanism** | Service Control Policies (SCPs) | Azure Policy + Blueprints | Organization Policies |
| **Identity provider** | IAM (per account) + Identity Center | Entra ID (centralized) | Cloud Identity / Workspace |
| **Resource group** | ❌ No (tags-based grouping) | ✅ Required (mandatory) | ❌ No (project = boundary) |
| **Cross-account access** | IAM Roles (AssumeRole) | RBAC (inherited across MGs) | IAM (inherited down hierarchy) |

---

## 6. Practical Company Setup — From Day Zero to Production

> This section walks through **exactly what happens in real life** when a company adopts cloud — from registering your organization to onboarding employees, enabling SSO, granting file storage, and how developers consume APIs.

---

### 6.1 Step 0 — How a Company Registers on Cloud (Day Zero)

Before **anything** happens — compute, storage, apps — you first **create your company's identity on the cloud**. This is the single most important step.

#### What Happens Behind the Scenes

```
Company: "Acme Corp" wants to move to cloud
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        STEP 0: REGISTER YOUR COMPANY                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─── Microsoft / Azure Path ──────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  1. Go to portal.azure.com or admin.microsoft.com                    │   │
│  │  2. Sign up with company email (admin@acmecorp.com)                  │   │
│  │  3. This creates an "Entra ID Tenant" (formerly Azure AD Tenant)     │   │
│  │     └── Tenant = Your company's identity universe                    │   │
│  │     └── Tenant ID: a unique GUID (e.g., 72f988bf-86f1-41af-...)     │   │
│  │  4. Add your custom domain: acmecorp.com                             │   │
│  │     └── Verify via DNS TXT record                                    │   │
│  │  5. Buy licenses: Microsoft 365 E3/E5 (includes Office apps,         │   │
│  │     SharePoint, OneDrive, Teams, Exchange, Entra ID P1/P2)           │   │
│  │  6. Optionally: Create Azure Subscription for IaaS/PaaS              │   │
│  │                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─── Google / GCP Path ───────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  1. Go to workspace.google.com (for productivity) or                  │   │
│  │     console.cloud.google.com (for infra)                              │   │
│  │  2. Sign up with company domain (admin@acmecorp.com)                  │   │
│  │  3. This creates a "Google Workspace Organization"                    │   │
│  │     └── Org = Your company's identity universe                        │   │
│  │  4. Verify domain ownership via DNS TXT record                        │   │
│  │  5. Choose plan: Business Starter/Standard/Plus or Enterprise         │   │
│  │     └── Includes Gmail, Drive, Docs, Sheets, Meet, Calendar          │   │
│  │  6. For GCP infra: Link Workspace domain → GCP Organization          │   │
│  │     └── Or use Cloud Identity (free, no productivity apps)            │   │
│  │                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─── AWS Path ────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  1. Go to aws.amazon.com → Create Account                            │   │
│  │  2. This creates a "Root Account" (Management Account)                │   │
│  │     └── Uses an email address (aws-root@acmecorp.com)                │   │
│  │  3. Enable AWS Organizations → create OUs                            │   │
│  │  4. AWS does NOT have a built-in productivity suite                   │   │
│  │     └── WorkMail exists but rarely used                              │   │
│  │     └── Most companies use Microsoft 365 or Google Workspace          │   │
│  │  5. For SSO: Set up IAM Identity Center                               │   │
│  │     └── Connect to external IdP (Entra ID / Okta / Google)           │   │
│  │  6. AWS is primarily IaaS/PaaS — not a productivity provider         │   │
│  │                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### The Fundamental Difference — Productivity vs Infrastructure

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  TWO LAYERS OF "CLOUD"                                   │
│                                                                          │
│  LAYER 1: PRODUCTIVITY & IDENTITY (SaaS)                                │
│  ════════════════════════════════════════                                │
│  "Where my employees get email, files, apps, and identity"              │
│                                                                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────┐     │
│  │ Microsoft 365     │  │ Google Workspace  │  │ AWS               │     │
│  │                   │  │                   │  │ (No native suite) │     │
│  │ • Outlook (Email) │  │ • Gmail (Email)   │  │                   │     │
│  │ • SharePoint      │  │ • Google Drive    │  │ • WorkMail (basic)│     │
│  │ • OneDrive        │  │ • Google Docs     │  │ • WorkDocs (basic)│     │
│  │ • Teams           │  │ • Google Meet     │  │ • Chime (meet)    │     │
│  │ • Excel/Word/PPT  │  │ • Sheets/Docs/    │  │                   │     │
│  │ • Entra ID (IdP)  │  │   Slides          │  │ Usually paired    │     │
│  │ • Intune (MDM)    │  │ • Google Chat     │  │ with M365 or      │     │
│  │                   │  │ • Cloud Identity   │  │ Google Workspace  │     │
│  └──────────────────┘  └──────────────────┘  └───────────────────┘     │
│                                                                          │
│  LAYER 2: INFRASTRUCTURE & PLATFORM (IaaS / PaaS)                      │
│  ════════════════════════════════════════════════                        │
│  "Where my developers build, deploy, and run applications"              │
│                                                                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────┐     │
│  │ Azure             │  │ GCP               │  │ AWS               │     │
│  │ VMs, AKS, SQL,   │  │ GCE, GKE, BigQ,  │  │ EC2, EKS, RDS,   │     │
│  │ Functions, etc.  │  │ Functions, etc.   │  │ Lambda, etc.     │     │
│  └──────────────────┘  └──────────────────┘  └───────────────────┘     │
│                                                                          │
│  ⚠️  MOST COMPANIES USE A COMBINATION:                                  │
│  ────────────────────────────────────────                                │
│  Common Combos:                                                          │
│  • Microsoft 365 (identity+productivity) + Azure (infra)  ← Most common │
│  • Microsoft 365 (identity+productivity) + AWS (infra)    ← Very common │
│  • Google Workspace (identity+productivity) + GCP (infra) ← Common      │
│  • Google Workspace + AWS                                  ← Common      │
│  • Okta (identity) + Any Cloud (infra)                     ← Enterprise │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

#### What Gets Created When You Register

| What | Microsoft (Azure) | Google (GCP) | AWS |
|------|-------------------|--------------|-----|
| **Identity Container** | Entra ID Tenant | Google Workspace Org / Cloud Identity | Root Account |
| **Unique ID** | Tenant ID (GUID) | Organization ID (numeric) | Account ID (12-digit) |
| **Admin Account** | Global Administrator | Super Admin | Root User |
| **Domain** | acmecorp.onmicrosoft.com (default) + custom | acmecorp.com (verified) | N/A (email-based) |
| **User Directory** | Entra ID | Google Directory | IAM (no directory) |
| **Email** | Exchange Online (Outlook) | Gmail | WorkMail (optional) |
| **File Storage** | OneDrive (personal) + SharePoint (shared) | Google Drive (personal + shared) | S3 / WorkDocs (optional) |
| **Office Apps** | Word, Excel, PPT (desktop + web) | Docs, Sheets, Slides (web-first) | ❌ None |
| **Chat/Meet** | Microsoft Teams | Google Chat + Meet | Chime (optional) |
| **License Needed** | M365 E3/E5 ($36-$57/user/mo) | Workspace Business ($7-$18/user/mo) | Free (AWS account) |
| **Infra Console** | portal.azure.com | console.cloud.google.com | console.aws.amazon.com |

---

### 6.2 Onboarding a New Employee — Full Real-World Flow

> **Scenario:** John Doe just joined Acme Corp as a Software Engineer. Here's exactly what happens from HR click to "John is working."

#### The Complete Onboarding Flow

```
HR System (Workday / BambooHR / SAP SuccessFactors)
│
│  HR creates employee record:
│  Name: John Doe
│  Email: john.doe@acmecorp.com
│  Department: Engineering
│  Title: Software Engineer
│  Start Date: 2026-05-01
│  Manager: jane.smith@acmecorp.com
│
│  ──── AUTOMATIC PROVISIONING (via SCIM / HR Connector) ────
│
▼
┌─────────────────────────────────────────────────────────────────────┐
│              IDENTITY PROVIDER (IdP) — The Source of Truth          │
│                                                                     │
│  ┌─── Microsoft Entra ID ────────────────────────────────────────┐ │
│  │                                                                │ │
│  │  1. User object created:                                       │ │
│  │     • UPN: john.doe@acmecorp.com                               │ │
│  │     • Object ID: a1b2c3d4-e5f6-7890-...                       │ │
│  │     • Department: Engineering                                  │ │
│  │     • Manager: jane.smith@acmecorp.com                         │ │
│  │                                                                │ │
│  │  2. Added to Security Groups:                                  │ │
│  │     • SG-All-Employees                                         │ │
│  │     • SG-Engineering                                           │ │
│  │     • SG-Developers                                            │ │
│  │                                                                │ │
│  │  3. License assigned (via group-based licensing):              │ │
│  │     • Microsoft 365 E5                                         │ │
│  │     • Azure DevOps Basic                                       │ │
│  │     • Power BI Pro                                             │ │
│  │                                                                │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌─── OR: Google Workspace Admin ────────────────────────────────┐ │
│  │                                                                │ │
│  │  1. User created in Google Admin Console:                      │ │
│  │     • Primary email: john.doe@acmecorp.com                     │ │
│  │     • Org Unit: /Engineering                                   │ │
│  │                                                                │ │
│  │  2. Added to Google Groups:                                    │ │
│  │     • all-employees@acmecorp.com                               │ │
│  │     • engineering@acmecorp.com                                 │ │
│  │                                                                │ │
│  │  3. License assigned:                                          │ │
│  │     • Google Workspace Enterprise Plus                         │ │
│  │                                                                │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌─── OR: Okta / External IdP ──────────────────────────────────┐ │
│  │                                                                │ │
│  │  1. User profile created in Okta Universal Directory          │ │
│  │  2. Assigned to groups → triggers app provisioning            │ │
│  │  3. SCIM pushes user to downstream apps                       │ │
│  │                                                                │ │
│  └────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
         ┌─────────────────────────┼──────────────────────────┐
         │                         │                          │
         ▼                         ▼                          ▼
┌─────────────────┐   ┌──────────────────┐   ┌─────────────────────┐
│ PRODUCTIVITY    │   │ CLOUD INFRA      │   │ THIRD-PARTY APPS    │
│ APPS ENABLED    │   │ ACCESS GRANTED   │   │ PROVISIONED (SCIM)  │
│                 │   │                  │   │                     │
│ ✅ Email        │   │ ✅ AWS Console   │   │ ✅ Slack            │
│ ✅ Calendar     │   │   (via SSO)     │   │ ✅ Jira             │
│ ✅ Drive/OneDrv │   │ ✅ Azure Portal  │   │ ✅ GitHub           │
│ ✅ Office/Docs  │   │   (via Entra ID)│   │ ✅ Confluence       │
│ ✅ Chat/Teams   │   │ ✅ GCP Console   │   │ ✅ Salesforce       │
│ ✅ SharePoint   │   │   (via SSO)     │   │ ✅ Figma            │
│                 │   │                  │   │                     │
└─────────────────┘   └──────────────────┘   └─────────────────────┘
```

#### How the Automated Provisioning Works (SCIM Protocol)

```
┌──────────────┐        ┌──────────────┐         ┌──────────────┐
│   HR System   │──SCIM──►   Identity    │──SCIM───►  Target Apps  │
│  (Workday)    │  Push   │   Provider   │  Push    │              │
│               │        │ (Entra/Okta/ │         │  Slack       │
│ "Create User" │        │  Google)     │         │  GitHub      │
│               │        │              │         │  Jira        │
│               │        │ Groups →     │         │  AWS SSO     │
│               │        │ Licenses →   │         │  Salesforce  │
└──────────────┘        └──────────────┘         └──────────────┘
                                │
                  SCIM = System for Cross-domain
                  Identity Management
                  ─────────────────────────
                  REST API standard for:
                  • POST /Users       (create user)
                  • PATCH /Users/{id} (update user)
                  • DELETE /Users/{id}(deactivate user)
                  • POST /Groups      (manage groups)
```

#### What John Gets on Day 1 — Complete Breakdown

```
John Doe opens his laptop on Day 1:
│
├── 1. FIRST LOGIN
│   │   Goes to: login.microsoftonline.com  OR  accounts.google.com
│   │   Enters: john.doe@acmecorp.com
│   │   Sets up: Password + MFA (Authenticator app / Security Key)
│   │   Accepts: Terms of Use / Company Policy
│   │
│   └── He now has a CLOUD IDENTITY ✅
│
├── 2. EMAIL
│   │   Microsoft: Outlook Web (outlook.office.com) or Desktop app
│   │   Google:    Gmail (mail.google.com)
│   │   Address:   john.doe@acmecorp.com
│   │
│   └── Auto-configured, mailbox already created ✅
│
├── 3. PERSONAL FILE STORAGE
│   │   Microsoft: OneDrive (1 TB per user)
│   │   Google:    Google Drive (varies by plan: 1TB-5TB)
│   │   Path:      onedrive.com or drive.google.com
│   │
│   └── Personal space, only John can see ✅
│
├── 4. SHARED FILE STORAGE
│   │   Microsoft: SharePoint sites
│   │   │  └── Engineering Team Site: acmecorp.sharepoint.com/sites/engineering
│   │   │  └── Company Wiki: acmecorp.sharepoint.com/sites/wiki
│   │   Google:    Shared Drives
│   │   │  └── Engineering Shared Drive
│   │   │  └── Company Documents Shared Drive
│   │   │
│   │   Access controlled by: Group membership (SG-Engineering)
│   │
│   └── John sees shared files because he's in Engineering group ✅
│
├── 5. OFFICE APPLICATIONS
│   │   Microsoft: Word, Excel, PowerPoint, OneNote
│   │   │  └── Web: office.com  |  Desktop: installed via Intune/MDM
│   │   Google:    Docs, Sheets, Slides, Keep
│   │   │  └── Web-only (native browser apps)
│   │
│   └── Licensed automatically via group membership ✅
│
├── 6. COMMUNICATION
│   │   Microsoft: Teams (chat, video, channels)
│   │   Google:    Google Chat + Google Meet
│   │   │
│   │   Auto-added to channels:
│   │   • #general  • #engineering  • #new-hires
│   │
│   └── Ready to chat on Day 1 ✅
│
├── 7. CALENDAR
│   │   Microsoft: Outlook Calendar
│   │   Google:    Google Calendar
│   │   │
│   │   Can see: Team calendars, meeting rooms, colleagues' free/busy
│   │
│   └── Manager's meetings already shared ✅
│
├── 8. COMPANY INTRANET / PORTAL
│   │   Microsoft: SharePoint Intranet site
│   │   Google:    Google Sites / custom portal
│   │   │
│   │   Contains: HR policies, org chart, benefits info, IT help
│   │
│   └── Accessible immediately ✅
│
├── 9. DEVELOPER TOOLS (because John is an engineer)
│   │   Code: GitHub / Azure DevOps / GitLab (provisioned via SCIM)
│   │   Cloud: AWS Console / Azure Portal / GCP Console (via SSO)
│   │   IDE: VS Code (company settings synced)
│   │   CI/CD: Jenkins / GitHub Actions / Azure Pipelines
│   │
│   └── Access based on "SG-Developers" group membership ✅
│
└── 10. SSO APP PORTAL
    │   Microsoft: myapps.microsoft.com (App Launcher)
    │   Google:    workspace.google.com (App Launcher)
    │   Okta:      acmecorp.okta.com (Dashboard)
    │
    │   All company apps in one place with single sign-on:
    │   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
    │   │Slack │ │Jira  │ │GitHub│ │Sales │ │AWS   │
    │   │      │ │      │ │      │ │force │ │Consol│
    │   └──────┘ └──────┘ └──────┘ └──────┘ └──────┘
    │
    └── One password, one MFA — access everything ✅
```

---

### 6.3 SSO (Single Sign-On) — How It Actually Works

> When John clicks on any company app, how does he magically get logged in without entering credentials again?

#### SSO Flow — Step by Step

```
John clicks "Jira" in his app portal
│
│  ── STEP 1: Browser redirects to Jira ──
│
▼
┌────────────────────────────────────────────────────────────────────┐
│  Jira (Service Provider / SP / Relying Party)                     │
│                                                                    │
│  "I don't know who you are. Let me redirect you to your           │
│   company's Identity Provider to prove your identity."            │
│                                                                    │
│  Jira checks the email domain: @acmecorp.com                     │
│  Jira knows: acmecorp.com uses Entra ID for SSO                  │
│  (This was configured by IT admin during Jira setup)              │
│                                                                    │
│  HTTP 302 REDIRECT ──►  login.microsoftonline.com/                │
│                         {tenant-id}/saml2?SAMLRequest=...         │
│                         OR                                         │
│                         accounts.google.com/o/saml2/...           │
└────────────────────────────┬───────────────────────────────────────┘
                             │
                             ▼
┌────────────────────────────────────────────────────────────────────┐
│  Identity Provider (IdP) — Entra ID / Google / Okta              │
│                                                                    │
│  CASE A: John already has an active session (browser cookie)       │
│  ────────────────────────────────────────────────────────          │
│  → IdP says: "I know John. He authenticated 2 hours ago.          │
│    His session is still valid. No need to ask for password."       │
│  → IdP creates a SAML Assertion / OIDC Token                      │
│                                                                    │
│  CASE B: No active session                                         │
│  ─────────────────────────                                         │
│  → IdP shows login page                                            │
│  → John enters: john.doe@acmecorp.com + password                  │
│  → MFA challenge: Approve on Authenticator app                     │
│  → IdP validates credentials against directory                     │
│  → IdP creates a session cookie + SAML Assertion / OIDC Token     │
│                                                                    │
│  ┌──────────────────────────────────────────────────────┐          │
│  │            SAML ASSERTION (simplified)                │          │
│  │                                                       │          │
│  │  Subject: john.doe@acmecorp.com                       │          │
│  │  Issuer:  login.microsoftonline.com/{tenant-id}       │          │
│  │  Audience: https://acmecorp.atlassian.net             │          │
│  │  Attributes:                                          │          │
│  │    - displayName: John Doe                            │          │
│  │    - email: john.doe@acmecorp.com                     │          │
│  │    - groups: [SG-Engineering, SG-Developers]          │          │
│  │    - department: Engineering                          │          │
│  │  Signature: (RSA-SHA256 signed by IdP's cert)        │          │
│  │  NotBefore: 2026-05-01T09:00:00Z                     │          │
│  │  NotOnOrAfter: 2026-05-01T09:05:00Z (5 min window)  │          │
│  │                                                       │          │
│  └──────────────────────────────────────────────────────┘          │
│                                                                    │
│  HTTP POST REDIRECT ──► back to Jira with SAML Response           │
└────────────────────────────┬───────────────────────────────────────┘
                             │
                             ▼
┌────────────────────────────────────────────────────────────────────┐
│  Jira (SP) receives the SAML Response                             │
│                                                                    │
│  1. Validates the XML signature (using IdP's public certificate)  │
│  2. Checks: Is the audience correct? (yes, it's for Jira)         │
│  3. Checks: Is the assertion still valid? (within 5 min window)   │
│  4. Extracts: user email, name, groups                            │
│  5. Looks up or creates local user: john.doe@acmecorp.com         │
│  6. Maps groups → Jira roles (SG-Engineering → jira-developers)   │
│  7. Creates a Jira session cookie                                  │
│                                                                    │
│  ✅ John is now logged into Jira without typing Jira credentials  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

#### SSO Protocols — SAML vs OIDC vs OAuth 2.0

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     SSO PROTOCOL COMPARISON                              │
│                                                                          │
│  ┌────────────────┐  ┌─────────────────┐  ┌─────────────────┐          │
│  │     SAML 2.0    │  │   OIDC (OpenID   │  │   OAuth 2.0     │          │
│  │                 │  │   Connect)       │  │                 │          │
│  ├────────────────┤  ├─────────────────┤  ├─────────────────┤          │
│  │ Format: XML     │  │ Format: JSON/JWT │  │ Format: JSON    │          │
│  │ Token: Assertion│  │ Token: ID Token  │  │ Token: Access   │          │
│  │                 │  │  + Access Token  │  │  Token          │          │
│  │ Use: Enterprise │  │ Use: Modern apps │  │ Use: API access │          │
│  │ SSO (web apps)  │  │ SSO + API        │  │ (delegation)    │          │
│  │                 │  │                  │  │                 │          │
│  │ Examples:       │  │ Examples:        │  │ Examples:       │          │
│  │ • Jira          │  │ • SPA apps       │  │ • "Login with   │          │
│  │ • Salesforce    │  │ • Mobile apps    │  │    Google"      │          │
│  │ • AWS Console   │  │ • Custom apps    │  │ • API-to-API    │          │
│  │ • ServiceNow   │  │ • Azure Portal   │  │ • GitHub OAuth  │          │
│  └────────────────┘  └─────────────────┘  └─────────────────┘          │
│                                                                          │
│  KEY DIFFERENCE:                                                         │
│  • SAML   = "WHO is this user?"        (Authentication only)            │
│  • OIDC   = "WHO is this user?"        (Authentication) +               │
│              "Give me a token for API"  (Authorization)                  │
│  • OAuth  = "Can this app access my    (Authorization only,             │
│              data?"                     NOT authentication)              │
│                                                                          │
│  In Production:                                                          │
│  • Enterprise web apps → SAML 2.0 (Jira, Salesforce, AWS Console)      │
│  • Modern apps/SPAs    → OIDC (React apps, mobile apps)                │
│  • API access          → OAuth 2.0 (service-to-service)                │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

#### SSO Services — Which Service Provides SSO?

| Role | Microsoft | Google | AWS | Third-Party |
|------|-----------|--------|-----|-------------|
| **Identity Provider (IdP)** | Entra ID (Azure AD) | Google Workspace / Cloud Identity | IAM Identity Center | Okta, Ping Identity, OneLogin |
| **SSO Protocol** | SAML 2.0 + OIDC | SAML 2.0 + OIDC | SAML 2.0 (to apps) | SAML 2.0 + OIDC |
| **MFA** | Entra ID MFA (Authenticator, FIDO2) | Google 2-Step (Authenticator, Titan Key) | MFA via IdP | Okta Verify, Duo |
| **App Catalog** | Entra ID Enterprise Apps (3000+) | Google SAML Apps | IAM Identity Center Apps | Okta Integration Network (7000+) |
| **User Portal** | myapps.microsoft.com | workspace.google.com | AWS access portal | acmecorp.okta.com |
| **Conditional Access** | ✅ Conditional Access Policies | ✅ Context-Aware Access | ❌ (via IdP) | ✅ (varies) |
| **Device Trust** | Intune + Entra ID Join | Endpoint Verification | ❌ (via IdP) | ✅ (varies) |

---

### 6.4 How Users Get SharePoint / Google Drive / OneDrive

> This is NOT manual. It's all **license-based + group-based** — automatic.

#### The Storage Provisioning Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│               HOW SHARED STORAGE IS PROVISIONED                          │
│                                                                          │
│  ── MICROSOFT PATH ──────────────────────────────────────────────────── │
│                                                                          │
│  User Created in Entra ID                                                │
│       │                                                                  │
│       ├── Assigned M365 E3/E5 License (includes OneDrive + SharePoint)  │
│       │       │                                                          │
│       │       ├── OneDrive provisioned automatically (1 TB personal)    │
│       │       │   URL: acmecorp-my.sharepoint.com/personal/john_doe    │
│       │       │                                                          │
│       │       └── SharePoint: Access controlled by Security Groups      │
│       │           │                                                      │
│       │           ├── "SG-All-Employees" → Company Intranet site       │
│       │           ├── "SG-Engineering"   → Engineering Team Site        │
│       │           └── "SG-Project-Alpha" → Project Alpha site          │
│       │                                                                  │
│       └── Teams: Each Team = SharePoint site + Mailbox + Chat           │
│           └── "Engineering" Team → auto-creates SharePoint site        │
│               Files tab in Teams = SharePoint document library          │
│                                                                          │
│  ── GOOGLE PATH ─────────────────────────────────────────────────────── │
│                                                                          │
│  User Created in Google Admin                                            │
│       │                                                                  │
│       ├── Workspace License assigned                                     │
│       │       │                                                          │
│       │       ├── Google Drive provisioned automatically                 │
│       │       │   └── "My Drive" = personal storage (1TB-5TB by plan)  │
│       │       │                                                          │
│       │       └── Shared Drives: Access controlled by Google Groups     │
│       │           │                                                      │
│       │           ├── engineering@acmecorp.com → "Engineering" Shared Dr│
│       │           ├── all@acmecorp.com → "Company Documents" Shared Dr  │
│       │           └── project-alpha@acmecorp.com → "Alpha" Shared Dr   │
│       │                                                                  │
│       └── Google Drive = unified storage for Docs, Sheets, Slides      │
│           └── No separate service; all files live in Drive             │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Storage Comparison — Personal vs Shared

```
┌──────────────────────────────────────────────────────────────────┐
│           PERSONAL STORAGE         vs      SHARED STORAGE        │
│                                                                   │
│  ┌─── Microsoft ───────────┐    ┌─── Microsoft ──────────────┐  │
│  │ OneDrive                │    │ SharePoint                  │  │
│  │ • 1 TB per user         │    │ • 1 TB + 10GB per user     │  │
│  │ • Only owner can see    │    │ • Team/group accessible    │  │
│  │ • Syncs to desktop      │    │ • Versioning, workflows   │  │
│  │ • Auto-save Office docs │    │ • Permissions inherited    │  │
│  │ • Recycle bin (93 days) │    │ • Sites = Departments      │  │
│  └─────────────────────────┘    └────────────────────────────┘  │
│                                                                   │
│  ┌─── Google ──────────────┐    ┌─── Google ─────────────────┐  │
│  │ My Drive                │    │ Shared Drives              │  │
│  │ • 1-5 TB per user       │    │ • Org-owned (not personal) │  │
│  │ • Only owner can see    │    │ • Members can add/remove   │  │
│  │ • Syncs via Drive app   │    │ • Files stay when user     │  │
│  │ • Native Docs/Sheets    │    │   leaves company           │  │
│  │ • Trash (30 days)       │    │ • Up to 400K files per     │  │
│  └─────────────────────────┘    └────────────────────────────┘  │
│                                                                   │
│  ⚠️ KEY DIFFERENCE:                                              │
│  SharePoint files = owned by the SITE (persist after user leaves)│
│  OneDrive files   = owned by the USER (deleted after user leaves)│
│  Shared Drives    = owned by the ORG  (persist after user leaves)│
│  My Drive files   = owned by the USER (deleted after user leaves)│
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

### 6.5 Developer API Access — How Developers Call Service APIs

> **Scenario:** A developer at Acme Corp wants to build an internal tool that reads files from SharePoint / Google Drive, creates calendar events, and posts messages to Teams / Chat. How does this work?

#### The Complete API Access Flow

```
Developer wants to call SharePoint API / Google Drive API
│
│  "I need to read files from the Engineering SharePoint site
│   programmatically in my Python/Node.js application."
│
│  ── STEP 1: Register the Application ──
│
▼
┌──────────────────────────────────────────────────────────────────────┐
│                  APP REGISTRATION                                     │
│                                                                       │
│  ┌─── Microsoft (Entra ID) ──────────────────────────────────────┐  │
│  │                                                                │  │
│  │  Go to: portal.azure.com → Entra ID → App Registrations      │  │
│  │  Create: "Acme Internal Tool"                                  │  │
│  │                                                                │  │
│  │  What you get:                                                 │  │
│  │  ├── Application (Client) ID: 1a2b3c4d-5e6f-...              │  │
│  │  ├── Directory (Tenant) ID:   72f988bf-86f1-...              │  │
│  │  ├── Client Secret OR Certificate (for auth)                  │  │
│  │  └── Redirect URI: https://myapp.acmecorp.com/callback       │  │
│  │                                                                │  │
│  │  Configure API Permissions:                                    │  │
│  │  ├── Microsoft Graph API                                       │  │
│  │  │   ├── Files.Read.All (read SharePoint files)               │  │
│  │  │   ├── Calendars.ReadWrite (manage calendar)                │  │
│  │  │   ├── Chat.ReadWrite (Teams messages)                      │  │
│  │  │   ├── User.Read (read user profile)                        │  │
│  │  │   └── Mail.Send (send emails)                              │  │
│  │  └── Admin must GRANT CONSENT for these permissions           │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─── Google (Cloud Console) ────────────────────────────────────┐  │
│  │                                                                │  │
│  │  Go to: console.cloud.google.com → APIs & Services             │  │
│  │       → Credentials → Create OAuth 2.0 Client ID              │  │
│  │                                                                │  │
│  │  What you get:                                                 │  │
│  │  ├── Client ID: 123456789-abc.apps.googleusercontent.com     │  │
│  │  ├── Client Secret: GOCSPX-...                                │  │
│  │  └── Redirect URI: https://myapp.acmecorp.com/callback       │  │
│  │                                                                │  │
│  │  Enable APIs:                                                  │  │
│  │  ├── Google Drive API                                          │  │
│  │  ├── Google Calendar API                                       │  │
│  │  ├── Gmail API                                                 │  │
│  │  ├── Google Chat API                                           │  │
│  │  └── Admin SDK (for user management)                           │  │
│  │                                                                │  │
│  │  Configure OAuth Consent Screen:                               │  │
│  │  ├── Internal (org users only) or External                    │  │
│  │  └── Define requested scopes                                   │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘

│  ── STEP 2: Get an Access Token ──
│
▼
┌──────────────────────────────────────────────────────────────────────┐
│              AUTHENTICATION FLOW (OAuth 2.0)                          │
│                                                                       │
│  Two main patterns:                                                   │
│                                                                       │
│  PATTERN A: User-Delegated (act on behalf of a user)                 │
│  ─────────────────────────────────────────────────────                │
│                                                                       │
│  User (John) ──► App ──► IdP Login Page ──► User consents            │
│       │                                          │                    │
│       │    ┌─────────────────────────────────────┘                   │
│       │    │  Authorization Code returned                            │
│       │    ▼                                                          │
│       │  App exchanges code for tokens:                               │
│       │  POST /oauth2/v2.0/token                                     │
│       │  ├── access_token  (expires in 1 hour, used to call API)    │
│       │  ├── refresh_token (used to get new access_token)            │
│       │  └── id_token      (user identity info, JWT)                │
│       │                                                               │
│       └── App calls API WITH user's access_token                     │
│           → API sees: "John is reading his own files"                │
│                                                                       │
│  PATTERN B: Application-Only (app acts as itself, no user)           │
│  ─────────────────────────────────────────────────────────           │
│                                                                       │
│  App ──► POST /oauth2/v2.0/token                                    │
│          Body: client_id + client_secret + scope                     │
│          (Client Credentials Grant)                                   │
│       │                                                               │
│       └── Returns: access_token (app-level, not user-level)         │
│           → App calls API as ITSELF                                   │
│           → Can access ALL users' data (if admin consented)          │
│           → Used for: background jobs, daemons, microservices        │
│                                                                       │
│  ⚠️ In Google: Use Service Account + Domain-Wide Delegation          │
│     instead of OAuth Client Credentials                               │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘

│  ── STEP 3: Call the API ──
│
▼
┌──────────────────────────────────────────────────────────────────────┐
│                    CALLING THE API                                     │
│                                                                       │
│  ┌─── Microsoft Graph API ───────────────────────────────────────┐  │
│  │                                                                │  │
│  │  Base URL: https://graph.microsoft.com/v1.0                   │  │
│  │                                                                │  │
│  │  List SharePoint files:                                        │  │
│  │  GET /sites/{site-id}/drive/root/children                     │  │
│  │  Authorization: Bearer {access_token}                         │  │
│  │                                                                │  │
│  │  Create Calendar Event:                                        │  │
│  │  POST /me/events                                               │  │
│  │  Authorization: Bearer {access_token}                         │  │
│  │  Body: { "subject": "Team Standup", "start": {...} }          │  │
│  │                                                                │  │
│  │  Send Teams Message:                                           │  │
│  │  POST /teams/{team-id}/channels/{channel-id}/messages         │  │
│  │  Authorization: Bearer {access_token}                         │  │
│  │  Body: { "body": { "content": "Hello team!" } }              │  │
│  │                                                                │  │
│  │  Read User Profile:                                            │  │
│  │  GET /me                                                       │  │
│  │  Returns: { displayName, mail, jobTitle, department, ... }    │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─── Google Workspace APIs ─────────────────────────────────────┐  │
│  │                                                                │  │
│  │  List Drive Files:                                             │  │
│  │  GET https://www.googleapis.com/drive/v3/files                │  │
│  │  Authorization: Bearer {access_token}                         │  │
│  │                                                                │  │
│  │  Create Calendar Event:                                        │  │
│  │  POST https://www.googleapis.com/calendar/v3/                 │  │
│  │       calendars/primary/events                                │  │
│  │  Authorization: Bearer {access_token}                         │  │
│  │  Body: { "summary": "Team Standup", "start": {...} }         │  │
│  │                                                                │  │
│  │  Send Chat Message (Google Chat API):                          │  │
│  │  POST https://chat.googleapis.com/v1/spaces/{space}/messages  │  │
│  │  Authorization: Bearer {access_token}                         │  │
│  │                                                                │  │
│  │  Read User Profile (Admin SDK):                                │  │
│  │  GET https://admin.googleapis.com/admin/directory/v1/         │  │
│  │      users/{userKey}                                           │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─── AWS APIs ──────────────────────────────────────────────────┐  │
│  │                                                                │  │
│  │  AWS doesn't have productivity APIs like Graph/Google.         │  │
│  │  AWS APIs are for infrastructure:                              │  │
│  │                                                                │  │
│  │  List S3 Objects:                                              │  │
│  │  GET https://my-bucket.s3.amazonaws.com/?list-type=2          │  │
│  │  Authorization: AWS Signature V4                               │  │
│  │                                                                │  │
│  │  Auth Method: NOT OAuth — uses IAM credentials:               │  │
│  │  ├── Access Key ID + Secret Access Key (long-lived ⚠️)       │  │
│  │  ├── STS Temporary Credentials (best practice)                │  │
│  │  └── IAM Role (for services like EC2, Lambda)                 │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

#### Who Provides What — API Access Summary

| Question | Microsoft | Google | AWS |
|----------|-----------|--------|-----|
| **Where to register app?** | Entra ID → App Registrations | GCP Console → APIs & Services | IAM → Create User/Role |
| **What credentials do you get?** | Client ID + Client Secret | Client ID + Client Secret | Access Key + Secret Key |
| **Unified API?** | ✅ Microsoft Graph (single endpoint for all M365) | ❌ Separate APIs per service | ❌ Separate API per service |
| **API Base URL** | `graph.microsoft.com/v1.0` | `googleapis.com/{service}` | `{service}.amazonaws.com` |
| **Auth protocol** | OAuth 2.0 / OIDC | OAuth 2.0 / OIDC | AWS Signature V4 |
| **Tokens** | JWT access tokens (1hr) | JWT access tokens (1hr) | Temporary STS tokens (1-12hr) |
| **Admin consent** | Required for app-level perms | Domain-wide delegation | IAM policy attachment |
| **SDK/Libraries** | Microsoft Graph SDK (Python/Node/Java/C#) | Google Client Libraries (Python/Node/Java/Go) | AWS SDK (Boto3/JS/Java/Go) |
| **Playground** | Graph Explorer (developer.microsoft.com/graph) | APIs Explorer (developers.google.com) | CLI (aws cli) |
| **Rate Limits** | Per-app throttling | Per-project quotas | Per-account/service limits |

#### Code Example — Reading Files (Python)

```python
# ── MICROSOFT: Read SharePoint Files via Graph API ──────────
from azure.identity import ClientSecretCredential
from msgraph import GraphServiceClient

credential = ClientSecretCredential(
    tenant_id="72f988bf-...",
    client_id="1a2b3c4d-...",
    client_secret="your-secret"
)
client = GraphServiceClient(credential)

# List files in a SharePoint site's document library
files = await client.sites.by_site_id("site-id") \
    .drive.root.children.get()

for file in files.value:
    print(f"{file.name} - {file.size} bytes")


# ── GOOGLE: Read Drive Files via Drive API ──────────────────
from google.oauth2 import service_account
from googleapiclient.discovery import build

creds = service_account.Credentials.from_service_account_file(
    'service-account-key.json',
    scopes=['https://www.googleapis.com/auth/drive.readonly'],
    subject='admin@acmecorp.com'  # domain-wide delegation
)
service = build('drive', 'v3', credentials=creds)

results = service.files().list(
    pageSize=10,
    fields="files(id, name, mimeType)"
).execute()

for file in results.get('files', []):
    print(f"{file['name']} ({file['mimeType']})")


# ── AWS: List S3 Objects (no equivalent productivity API) ───
import boto3

s3 = boto3.client('s3')  # uses ~/.aws/credentials or IAM role
response = s3.list_objects_v2(Bucket='acme-corp-documents')

for obj in response.get('Contents', []):
    print(f"{obj['Key']} - {obj['Size']} bytes")
```

---

### 6.6 What Can Users Do By Default? (Permissions Baseline)

> When a new user is created, what can they do **out of the box** before any admin grants extra permissions?

#### Default User Capabilities

```
┌──────────────────────────────────────────────────────────────────────────┐
│          DEFAULT USER PERMISSIONS (No admin action needed)               │
│                                                                          │
│  ┌─── Microsoft 365 (with E3/E5 license) ────────────────────────────┐ │
│  │                                                                    │ │
│  │  ✅ CAN DO:                                                       │ │
│  │  ├── Send/receive email (Outlook)                                 │ │
│  │  ├── Create/edit documents (Word, Excel, PPT)                    │ │
│  │  ├── Store files in OneDrive (1 TB)                              │ │
│  │  ├── Chat and video call (Teams)                                 │ │
│  │  ├── View Global Address List (see all users in company)         │ │
│  │  ├── Schedule meetings, see free/busy of colleagues              │ │
│  │  ├── Access SharePoint sites they're added to                    │ │
│  │  ├── Create Teams (if allowed by admin policy)                   │ │
│  │  ├── Install mobile apps (Outlook, Teams, OneDrive)              │ │
│  │  ├── Share files from OneDrive (with company users by default)  │ │
│  │  ├── Register apps in Entra ID (if allowed — often disabled)    │ │
│  │  ├── Read their own profile, update phone/photo                 │ │
│  │  └── Access My Apps portal (myapps.microsoft.com)               │ │
│  │                                                                    │ │
│  │  ❌ CANNOT DO (needs admin/elevated role):                        │ │
│  │  ├── Access Azure Portal resources (needs Azure RBAC role)      │ │
│  │  ├── Create/manage users                                         │ │
│  │  ├── Assign licenses                                              │ │
│  │  ├── View audit logs                                              │ │
│  │  ├── Manage security groups                                       │ │
│  │  ├── Access other users' mailboxes                               │ │
│  │  ├── Create SharePoint sites (configurable)                      │ │
│  │  ├── Share files externally (configurable by admin)              │ │
│  │  └── Admin center access (admin.microsoft.com)                   │ │
│  │                                                                    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌─── Google Workspace ──────────────────────────────────────────────┐ │
│  │                                                                    │ │
│  │  ✅ CAN DO:                                                       │ │
│  │  ├── Send/receive email (Gmail)                                   │ │
│  │  ├── Create/edit documents (Docs, Sheets, Slides)                │ │
│  │  ├── Store files in My Drive (1-5 TB by plan)                    │ │
│  │  ├── Chat and video call (Chat, Meet)                            │ │
│  │  ├── View company directory                                       │ │
│  │  ├── Schedule meetings, see free/busy                            │ │
│  │  ├── Access Shared Drives they're added to                       │ │
│  │  ├── Create Google Groups (if allowed)                           │ │
│  │  ├── Share files with company users                              │ │
│  │  ├── Use Google Keep, Google Sites                               │ │
│  │  ├── Install mobile apps (Gmail, Drive)                          │ │
│  │  └── Access GCP Console (needs project-level IAM roles)         │ │
│  │                                                                    │ │
│  │  ❌ CANNOT DO:                                                    │ │
│  │  ├── Access Google Admin Console                                  │ │
│  │  ├── Create/manage users                                          │ │
│  │  ├── Create Shared Drives (configurable)                         │ │
│  │  ├── Share externally (configurable)                              │ │
│  │  ├── Access audit/security reports                                │ │
│  │  └── Change org-wide settings                                     │ │
│  │                                                                    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌─── AWS (No productivity suite — pure infra) ──────────────────────┐ │
│  │                                                                    │ │
│  │  ✅ New IAM User CAN DO:                                          │ │
│  │  ├── NOTHING by default! (zero permissions)                       │ │
│  │  ├── Must be explicitly granted every permission                 │ │
│  │  └── AWS follows "deny by default" model                         │ │
│  │                                                                    │ │
│  │  After SSO login (via Identity Center):                           │ │
│  │  ├── Can access only what their Permission Set allows            │ │
│  │  ├── Common: ReadOnlyAccess for dev accounts                     │ │
│  │  ├── Common: PowerUserAccess for dev accounts                    │ │
│  │  └── Common: ViewOnlyAccess for prod accounts                    │ │
│  │                                                                    │ │
│  │  ⚠️ AWS IAM = ZERO TRUST by default                              │ │
│  │     Every single permission must be explicitly granted            │ │
│  │     This is opposite to M365/Google where license = access       │ │
│  │                                                                    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### 6.7 Complete Day-Zero to Day-One Flow — The Big Picture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    COMPANY CLOUD JOURNEY: DAY ZERO → DAY ONE                    │
│                                                                                  │
│  DAY ZERO (IT Admin / CTO sets up):                                             │
│  ═══════════════════════════════════                                             │
│                                                                                  │
│  ① Register company domain with cloud provider                                  │
│     └── Creates: Tenant (Microsoft) / Organization (Google/GCP) / Account (AWS)│
│                                                                                  │
│  ② Verify domain ownership (DNS TXT record)                                     │
│     └── Proves: acmecorp.com belongs to you                                    │
│                                                                                  │
│  ③ Configure Identity Provider (IdP)                                            │
│     └── Entra ID / Google Workspace / Okta                                      │
│     └── Enable MFA, set password policies, conditional access                   │
│                                                                                  │
│  ④ Buy licenses (M365 E3/E5 or Google Workspace Business/Enterprise)           │
│     └── This enables: email, storage, office apps, chat, calendar              │
│                                                                                  │
│  ⑤ Create Security Groups / Google Groups for departments                      │
│     └── SG-Engineering, SG-Marketing, SG-HR, SG-All-Employees                 │
│                                                                                  │
│  ⑥ Set up SSO for third-party apps (Jira, Slack, GitHub, Salesforce)           │
│     └── SAML/OIDC configuration in IdP + each app                              │
│                                                                                  │
│  ⑦ Configure SCIM provisioning (auto-create users in Slack, Jira, etc.)        │
│     └── When user added to IdP → automatically appears in Slack/Jira/GitHub    │
│                                                                                  │
│  ⑧ Set up cloud infrastructure accounts (if needed)                            │
│     └── AWS Organizations / Azure Subscriptions / GCP Projects                 │
│     └── Connect SSO: IdP → AWS Identity Center / Azure RBAC / GCP IAM         │
│                                                                                  │
│  ⑨ Create shared resources                                                      │
│     └── SharePoint sites / Shared Drives for each department                   │
│     └── Teams channels / Google Chat spaces                                     │
│                                                                                  │
│  ⑩ Set up device management (optional but recommended)                          │
│     └── Microsoft Intune / Google Endpoint Management                           │
│     └── Enforce: encryption, screen lock, app restrictions                     │
│                                                                                  │
│  ─────────────────────────────────────────────────────────────────────────────── │
│                                                                                  │
│  DAY ONE (New Employee John Doe joins):                                         │
│  ═══════════════════════════════════════                                         │
│                                                                                  │
│  HR creates John in Workday ──► SCIM syncs to Entra ID / Google Admin           │
│        │                                                                         │
│        ├── ① Identity Created (john.doe@acmecorp.com)                           │
│        ├── ② Added to Groups (SG-Engineering, SG-All-Employees)                │
│        ├── ③ License Assigned (M365 E5 / Google Workspace Enterprise)          │
│        ├── ④ Email Provisioned (Outlook mailbox / Gmail inbox)                  │
│        ├── ⑤ Personal Storage Ready (OneDrive 1TB / My Drive 5TB)              │
│        ├── ⑥ Shared Storage Accessible (SharePoint / Shared Drives)            │
│        ├── ⑦ Apps Provisioned via SCIM (Slack, Jira, GitHub accounts)          │
│        ├── ⑧ SSO Portal Ready (myapps.microsoft.com / okta dashboard)          │
│        ├── ⑨ Cloud Console Access (AWS/Azure/GCP via SSO if developer)         │
│        └── ⑩ Welcome Email Sent (with setup instructions + MFA guide)          │
│                                                                                  │
│  John opens laptop → logs in once → has access to EVERYTHING                   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### 6.8 User Offboarding — What Happens When Someone Leaves

> Just as important as onboarding — and often more critical for security.

```
HR marks John as "Terminated" in Workday
│
▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AUTOMATED OFFBOARDING FLOW                       │
│                                                                     │
│  ① IMMEDIATE (within minutes):                                     │
│     ├── Entra ID / Google: Account DISABLED (can't log in)         │
│     ├── All active sessions REVOKED (tokens invalidated)           │
│     ├── MFA devices removed                                        │
│     ├── SSO access cut off → ALL apps (Slack, Jira, GitHub...)    │
│     └── SCIM sends DEACTIVATE to all connected apps               │
│                                                                     │
│  ② WITHIN 24 HOURS:                                                │
│     ├── Manager gets access to John's OneDrive / Drive files      │
│     ├── Email forwarding set to manager (optional)                 │
│     ├── Out-of-office auto-reply enabled                           │
│     ├── Shared resources (SharePoint/Shared Drives) unaffected    │
│     │   (because they're owned by the org, not the user)          │
│     └── Cloud IAM roles/permissions revoked                        │
│                                                                     │
│  ③ AFTER RETENTION PERIOD (30-90 days):                            │
│     ├── Account DELETED permanently                                │
│     ├── OneDrive / My Drive data deleted (or archived)             │
│     ├── Mailbox deleted (or converted to shared mailbox)           │
│     └── Licenses freed up (can be reassigned)                      │
│                                                                     │
│  ⚠️ SECURITY CRITICAL:                                             │
│     ├── Disable account BEFORE collecting laptop                   │
│     ├── Revoke any API keys / service account keys                │
│     ├── Rotate shared secrets the user had access to              │
│     └── Review audit logs for unusual activity pre-departure      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 6.9 Real-World Architecture — How It All Connects

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ACME CORP — FULL CLOUD ARCHITECTURE                      │
│                                                                              │
│                        ┌───────────────────┐                                │
│                        │   HR SYSTEM        │                                │
│                        │   (Workday)        │                                │
│                        └─────────┬─────────┘                                │
│                                  │ SCIM                                      │
│                                  ▼                                           │
│                        ┌───────────────────┐                                │
│                        │   IDENTITY         │                                │
│                        │   PROVIDER         │                                │
│                        │   (Entra ID/       │                                │
│                        │    Google/Okta)     │                                │
│                        └──┬──────┬───────┬──┘                               │
│                           │      │       │                                   │
│               ┌───────────┘      │       └────────────┐                     │
│               │ SAML/OIDC        │ SAML/OIDC          │ SCIM               │
│               ▼                  ▼                     ▼                     │
│  ┌─────────────────┐ ┌─────────────────┐  ┌─────────────────────┐          │
│  │  PRODUCTIVITY   │ │   CLOUD INFRA   │  │  THIRD-PARTY APPS   │          │
│  │                 │ │                 │  │                     │          │
│  │ ┌─────────────┐ │ │ ┌─────────────┐ │  │ ┌───┐ ┌───┐ ┌───┐ │          │
│  │ │ Email       │ │ │ │ AWS         │ │  │ │Slk│ │Jra│ │Git│ │          │
│  │ │ Calendar    │ │ │ │ EC2, S3,    │ │  │ │   │ │   │ │Hub│ │          │
│  │ │ Drive/SPO   │ │ │ │ Lambda, RDS │ │  │ └───┘ └───┘ └───┘ │          │
│  │ │ Office/Docs │ │ │ ├─────────────┤ │  │ ┌───┐ ┌───┐ ┌───┐ │          │
│  │ │ Teams/Chat  │ │ │ │ Azure       │ │  │ │SFo│ │Fig│ │1Ps│ │          │
│  │ │ Meet/Video  │ │ │ │ VMs, AKS,   │ │  │ │rce│ │ma │ │wd │ │          │
│  │ └─────────────┘ │ │ │ Functions   │ │  │ └───┘ └───┘ └───┘ │          │
│  │                 │ │ ├─────────────┤ │  │                     │          │
│  │ Microsoft 365   │ │ │ GCP         │ │  │  All accessible via│          │
│  │     OR          │ │ │ GCE, GKE,   │ │  │  SSO — one login   │          │
│  │ Google Workspace│ │ │ BigQuery    │ │  │  for everything     │          │
│  └─────────────────┘ │ └─────────────┘ │  └─────────────────────┘          │
│                       │                 │                                    │
│                       │ Connected via:  │                                    │
│                       │ • IAM Identity  │                                    │
│                       │   Center (AWS)  │                                    │
│                       │ • Entra ID RBAC │                                    │
│                       │   (Azure)       │                                    │
│                       │ • Workforce ID  │                                    │
│                       │   (GCP)         │                                    │
│                       └─────────────────┘                                    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                       DEVELOPER FLOW                                │    │
│  │                                                                     │    │
│  │  Developer ──SSO──► Cloud Console ──► Deploy Code ──► APIs          │    │
│  │      │                                                  │           │    │
│  │      └── Registers App in IdP ──► Gets Client ID ──► Calls         │    │
│  │          Graph API / Google APIs / AWS APIs                         │    │
│  │          with OAuth 2.0 tokens                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Core Service Categories — Full Comparison

### Master Comparison Table

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| **── COMPUTE ──** | | | |
| Virtual Machines | EC2 | Virtual Machines | Compute Engine |
| Auto Scaling | Auto Scaling Groups | VM Scale Sets (VMSS) | Managed Instance Groups (MIG) |
| Spot/Preemptible | Spot Instances | Spot VMs | Spot VMs (Preemptible) |
| Bare Metal | EC2 Bare Metal | Dedicated Hosts | Sole-Tenant Nodes |
| **── CONTAINERS ──** | | | |
| Container Orchestration | ECS / EKS | AKS | GKE |
| Serverless Containers | Fargate | Container Apps | Cloud Run |
| Container Registry | ECR | ACR | Artifact Registry |
| **── SERVERLESS ──** | | | |
| Functions | Lambda | Azure Functions | Cloud Functions |
| API Gateway | API Gateway | API Management | API Gateway / Apigee |
| Event Bus | EventBridge | Event Grid | Eventarc |
| Step Functions | Step Functions | Logic Apps / Durable Functions | Workflows |
| **── STORAGE ──** | | | |
| Object Storage | S3 | Blob Storage | Cloud Storage |
| Block Storage | EBS | Managed Disks | Persistent Disk |
| File Storage | EFS / FSx | Azure Files / NetApp Files | Filestore |
| Archive | S3 Glacier | Archive Storage | Archive Storage |
| **── DATABASES ──** | | | |
| Relational (Managed) | RDS / Aurora | Azure SQL / PostgreSQL Flexible | Cloud SQL / AlloyDB |
| NoSQL Document | DynamoDB | Cosmos DB | Firestore / Bigtable |
| In-Memory Cache | ElastiCache (Redis/Memcached) | Azure Cache for Redis | Memorystore |
| Data Warehouse | Redshift | Synapse Analytics | BigQuery |
| **── NETWORKING ──** | | | |
| Virtual Network | VPC | VNet | VPC |
| Load Balancer (L4) | NLB | Azure Load Balancer | Network Load Balancer |
| Load Balancer (L7) | ALB | Application Gateway | HTTP(S) Load Balancer |
| CDN | CloudFront | Azure CDN / Front Door | Cloud CDN |
| DNS | Route 53 | Azure DNS | Cloud DNS |
| VPN | Site-to-Site VPN | VPN Gateway | Cloud VPN |
| Direct Connect | Direct Connect | ExpressRoute | Cloud Interconnect |
| Service Mesh | App Mesh | Open Service Mesh | Traffic Director |
| **── SECURITY & IDENTITY ──** | | | |
| IAM | IAM | Entra ID + RBAC | Cloud IAM |
| Secrets | Secrets Manager | Key Vault | Secret Manager |
| Key Management | KMS | Key Vault | Cloud KMS |
| WAF | AWS WAF | Azure WAF | Cloud Armor |
| DDoS Protection | Shield / Shield Advanced | DDoS Protection | Cloud Armor |
| **── MONITORING ──** | | | |
| Metrics | CloudWatch Metrics | Azure Monitor Metrics | Cloud Monitoring |
| Logs | CloudWatch Logs | Log Analytics | Cloud Logging |
| Tracing | X-Ray | Application Insights | Cloud Trace |
| Dashboards | CloudWatch Dashboards | Azure Dashboards | Cloud Monitoring Dashboards |
| **── CI/CD & DEVOPS ──** | | | |
| Source Control | CodeCommit (deprecated) | Azure Repos | Cloud Source Repos |
| Build | CodeBuild | Azure Pipelines | Cloud Build |
| Deploy | CodeDeploy | Azure Pipelines | Cloud Deploy |
| IaC | CloudFormation | ARM / Bicep | Deployment Manager / Terraform |
| **── AI/ML ──** | | | |
| ML Platform | SageMaker | Azure ML | Vertex AI |
| Pre-trained AI | Rekognition, Comprehend | Cognitive Services | Vision AI, Natural Language |
| LLM / GenAI | Bedrock | Azure OpenAI | Gemini API / Vertex AI |
| **── MESSAGING ──** | | | |
| Message Queue | SQS | Azure Queue / Service Bus | Pub/Sub |
| Pub/Sub | SNS | Service Bus Topics | Pub/Sub |
| Streaming | Kinesis | Event Hubs | Dataflow (Pub/Sub) |

---

## 8. Identity & Access Management

### IAM Concept Flow — All Three Providers

```
WHO (Principal/Identity)  ──►  WHAT (Action/Permission)  ──►  WHICH (Resource)
         │                              │                            │
         │                              │                            │
    ┌────┴──────────────┐    ┌─────────┴───────────┐    ┌──────────┴───────────┐
    │ AWS:              │    │ AWS:                 │    │ AWS:                 │
    │  - IAM User       │    │  - IAM Policy (JSON) │    │  - ARN               │
    │  - IAM Role       │    │  - Action: s3:Get*   │    │  - arn:aws:s3:::*    │
    │  - Federated User │    │                      │    │                      │
    ├───────────────────┤    ├──────────────────────┤    ├──────────────────────┤
    │ Azure:            │    │ Azure:               │    │ Azure:               │
    │  - User           │    │  - RBAC Role         │    │  - Resource ID       │
    │  - Service Princ. │    │  - Role Definition   │    │  - /subscriptions/.. │
    │  - Managed Ident. │    │  - Actions: */read   │    │                      │
    ├───────────────────┤    ├──────────────────────┤    ├──────────────────────┤
    │ GCP:              │    │ GCP:                 │    │ GCP:                 │
    │  - Google Account  │    │  - IAM Role          │    │  - Resource path     │
    │  - Service Account│    │  - Binding            │    │  - projects/xxx/..   │
    │  - Group          │    │  - Permission          │    │                      │
    └───────────────────┘    └──────────────────────┘    └──────────────────────┘
```

### IAM Policy Structure Comparison

**AWS IAM Policy (JSON-based, attached to principal or resource):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        }
      }
    }
  ]
}
```

**Azure RBAC Role Assignment:**
```json
{
  "properties": {
    "roleDefinitionId": "/providers/Microsoft.Authorization/roleDefinitions/acdd72a7-...",
    "principalId": "user-object-id",
    "scope": "/subscriptions/sub-id/resourceGroups/rg-name"
  }
}
```

**GCP IAM Policy Binding:**
```json
{
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "user:alice@example.com",
        "serviceAccount:sa@project.iam.gserviceaccount.com"
      ],
      "condition": {
        "expression": "resource.name.startsWith('projects/_/buckets/my-bucket')"
      }
    }
  ]
}
```

### IAM Comparison Table

| Concept | AWS | Azure | GCP |
|---------|-----|-------|-----|
| **Human Identity** | IAM User | Entra ID User | Google Account |
| **Machine Identity** | IAM Role (for services) | Managed Identity / Service Principal | Service Account |
| **Group** | IAM Group | Entra ID Group | Google Group |
| **Policy Model** | Identity-based + Resource-based | RBAC (Role Assignments) | Binding (Role + Member + Resource) |
| **Policy Format** | JSON (Effect/Action/Resource) | Role Definition (Actions/DataActions) | Binding (Role + Members) |
| **Temporary Creds** | STS AssumeRole | Managed Identity Token | Workload Identity / SA Key |
| **Federation** | SAML / OIDC → IAM Roles | Entra ID (native SAML/OIDC) | Workforce Identity Federation |
| **SSO** | IAM Identity Center | Entra ID SSO | Cloud Identity SSO |
| **Permission Boundary** | Permission Boundary | Blueprint constraints | Org Policy Constraints |
| **Cross-account** | AssumeRole (Trust Policy) | Lighthouse / Guest Access | Cross-project IAM binding |
| **Deny Policy** | Explicit Deny in policy | Deny Assignments | IAM Deny Policies |

### Authentication Flow — Production (Federated SSO)

```
Corporate Employee
       │
       ▼
┌──────────────┐
│   Identity   │  (Okta / Active Directory / Google Workspace)
│   Provider   │
│   (IdP)      │
└──────┬───────┘
       │ SAML / OIDC Assertion
       ▼
┌──────────────────────────────────────────────────────────────┐
│                    Cloud Provider SSO                         │
│                                                              │
│  AWS:  IAM Identity Center  ──► AssumeRole ──► Temp Creds   │
│  Azure: Entra ID            ──► Token      ──► RBAC Access  │
│  GCP:  Workforce Identity   ──► Token      ──► IAM Access   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│  Temporary   │  (No long-lived keys in production!)
│  Credentials │
│  (STS Token) │
└──────┬───────┘
       │
       ▼
   Access Resources (S3 / Blob / GCS etc.)
```

---

## 9. Networking Deep Dive

### Virtual Network Architecture — All Three

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         REGION                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              VPC / VNet / VPC Network                              │  │
│  │              CIDR: 10.0.0.0/16                                    │  │
│  │                                                                    │  │
│  │  ┌─────────────────────────┐  ┌─────────────────────────┐        │  │
│  │  │     PUBLIC SUBNET       │  │     PUBLIC SUBNET       │        │  │
│  │  │     10.0.1.0/24         │  │     10.0.2.0/24         │        │  │
│  │  │     AZ-1 / Zone-a       │  │     AZ-2 / Zone-b       │        │  │
│  │  │  ┌───────┐ ┌────────┐  │  │  ┌───────┐ ┌────────┐  │        │  │
│  │  │  │ Web   │ │  NAT   │  │  │  │ Web   │ │  NAT   │  │        │  │
│  │  │  │Server │ │Gateway │  │  │  │Server │ │Gateway │  │        │  │
│  │  │  └───┬───┘ └───┬────┘  │  │  └───┬───┘ └───┬────┘  │        │  │
│  │  └──────┼─────────┼───────┘  └──────┼─────────┼───────┘        │  │
│  │         │         │                  │         │                 │  │
│  │  ┌──────┼─────────┼───────┐  ┌──────┼─────────┼───────┐        │  │
│  │  │  PRIVATE SUBNET        │  │  PRIVATE SUBNET        │        │  │
│  │  │  10.0.3.0/24           │  │  10.0.4.0/24           │        │  │
│  │  │  AZ-1 / Zone-a        │  │  AZ-2 / Zone-b         │        │  │
│  │  │  ┌────────┐ ┌──────┐  │  │  ┌────────┐ ┌──────┐  │        │  │
│  │  │  │  App   │ │ App  │  │  │  │  App   │ │ App  │  │        │  │
│  │  │  │Server 1│ │Srvr 2│  │  │  │Server 3│ │Srvr 4│  │        │  │
│  │  │  └────────┘ └──────┘  │  │  └────────┘ └──────┘  │        │  │
│  │  └────────────────────────┘  └────────────────────────┘        │  │
│  │                                                                    │  │
│  │  ┌────────────────────────┐  ┌────────────────────────┐        │  │
│  │  │  DATA SUBNET           │  │  DATA SUBNET           │        │  │
│  │  │  10.0.5.0/24           │  │  10.0.6.0/24           │        │  │
│  │  │  ┌────────┐ ┌──────┐  │  │  ┌────────┐ ┌──────┐  │        │  │
│  │  │  │  RDS   │ │Redis │  │  │  │  RDS   │ │Redis │  │        │  │
│  │  │  │Primary │ │Cache │  │  │  │Standby │ │Replca│  │        │  │
│  │  │  └────────┘ └──────┘  │  │  └────────┘ └──────┘  │        │  │
│  │  └────────────────────────┘  └────────────────────────┘        │  │
│  │                                                                    │  │
│  │  ┌──────────────────────────────────────────────────────────┐  │  │
│  │  │  Internet Gateway (IGW)  ◄──► Route Table (Public)       │  │  │
│  │  │  NAT Gateway             ◄──► Route Table (Private)      │  │  │
│  │  │  VPC Endpoints           ◄──► Private access to services │  │  │
│  │  └──────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Network Security Layers

```
Internet Traffic Incoming
        │
        ▼
┌───────────────────┐
│   DDoS Protection │  AWS Shield / Azure DDoS / Cloud Armor
└───────┬───────────┘
        │
        ▼
┌───────────────────┐
│   WAF (Layer 7)   │  AWS WAF / Azure WAF / Cloud Armor Rules
│   SQL injection,  │
│   XSS, Rate limit │
└───────┬───────────┘
        │
        ▼
┌───────────────────┐
│   Load Balancer   │  ALB / App Gateway / HTTP(S) LB
│   (Layer 7)       │  SSL Termination, Routing Rules
└───────┬───────────┘
        │
        ▼
┌───────────────────┐
│  Network ACL /    │  AWS: NACL (stateless, subnet-level)
│  NSG              │  Azure: NSG (stateful, subnet/NIC)
│  (Subnet level)   │  GCP: Firewall Rules (VPC-level)
└───────┬───────────┘
        │
        ▼
┌───────────────────┐
│  Security Group / │  AWS: Security Group (stateful, instance)
│  Firewall Rule    │  Azure: NSG (same, NIC-level)
│  (Instance level) │  GCP: Firewall Rules (tags/SA)
└───────┬───────────┘
        │
        ▼
   ┌─────────┐
   │ Instance │
   └─────────┘
```

### Networking Comparison Table

| Concept | AWS | Azure | GCP |
|---------|-----|-------|-----|
| **Virtual Network** | VPC (regional) | VNet (regional) | VPC (global!) |
| **Subnet** | Per-AZ | Per-Region (any AZ) | Per-Region |
| **Public/Private** | Route table + IGW | Route table + Public IP | Route + Cloud NAT |
| **Firewall (Instance)** | Security Groups (stateful) | NSG (stateful) | Firewall Rules (tags) |
| **Firewall (Subnet)** | NACL (stateless) | NSG (at subnet) | Firewall Rules (VPC) |
| **NAT** | NAT Gateway (per AZ) | NAT Gateway | Cloud NAT (per region) |
| **Peering** | VPC Peering (non-transitive) | VNet Peering (non-transitive) | VPC Peering (non-transitive) |
| **Transit** | Transit Gateway | Virtual WAN / vHub | NCC (Network Connectivity Center) |
| **Private Endpoints** | VPC Endpoints (Gateway/Interface) | Private Endpoints | Private Service Connect |
| **DNS** | Route 53 (public + private) | Azure DNS + Private DNS | Cloud DNS (public + private) |
| **VPC Scope** | Regional | Regional | **Global** (unique to GCP!) |
| **Subnet Scope** | Zonal | Regional | Regional |

> **🔑 Key Insight:** GCP VPC is **global** — one VPC spans all regions. AWS and Azure VPCs/VNets are regional and need peering to connect across regions.

---

## 10. Compute Deep Dive

### VM Instance Lifecycle

```
         ┌─────────┐
         │ Pending  │  (Instance launching, hardware allocated)
         └────┬─────┘
              │
              ▼
         ┌─────────┐
    ┌───►│ Running  │◄──── Start/Resume
    │    └────┬─────┘
    │         │
    │    ┌────┴─────────────────┐
    │    │                      │
    │    ▼                      ▼
    │ ┌─────────┐        ┌──────────┐
    │ │ Stopped │        │Terminated│  (AWS: instance gone)
    │ │(Deallocd)│        │          │  (Azure: VM deleted)
    │ └────┬────┘        └──────────┘  (GCP: instance deleted)
    │      │
    └──────┘
      Start
```

### Compute Comparison Table

| Feature | AWS EC2 | Azure VM | GCP Compute Engine |
|---------|---------|----------|-------------------|
| **Instance Types** | 750+ (m5, c5, r5...) | 700+ (D-series, E-series...) | 100+ (n2, c2, e2...) |
| **Custom Machine** | ❌ Fixed types | ❌ Fixed types (Constrained avail.) | ✅ Custom vCPU/RAM |
| **GPU** | P4, G5, P5 | NC, ND series | A2, G2 (NVIDIA) |
| **Pricing Model** | On-Demand, Reserved (1/3yr), Spot, Savings Plans | Pay-as-you-go, Reserved (1/3yr), Spot | On-Demand, Committed Use (1/3yr), Spot |
| **Spot Savings** | Up to 90% | Up to 90% | Up to 91% |
| **Per-second billing** | ✅ (Linux), hourly (Windows) | ✅ (per second) | ✅ (per second, min 1 min) |
| **Sustained Discount** | ❌ (use Savings Plans) | ❌ (use Reserved) | ✅ Automatic (up to 30%) |
| **Auto Scaling** | Auto Scaling Groups (ASG) | VM Scale Sets (VMSS) | Managed Instance Groups (MIG) |
| **Placement** | Placement Groups (cluster/spread/partition) | Proximity Placement Groups | Sole-Tenant Nodes |
| **OS** | Linux, Windows, macOS | Linux, Windows | Linux, Windows |
| **Key Pair** | SSH Key Pair (create/import) | SSH Key (create/import) | SSH Key (project/instance metadata) |
| **Metadata** | Instance Metadata Service (IMDS v2) | Instance Metadata Service (IMDS) | Metadata Server |

> **🔑 Key Insight:** GCP offers **automatic sustained use discounts** (no commitment needed) — unique among the three. GCP also allows **custom machine types** (pick exact vCPU + RAM).

---

## 11. Storage Deep Dive

### Object Storage Tiers — Comparison

```
                    ACCESS FREQUENCY
    ◄──── Frequent ──────────────────────── Rare ────►

AWS S3:
    ┌──────────┬───────────┬────────────┬──────────────┬──────────────────┐
    │S3 Standard│S3 Standard│S3 One Zone │ S3 Glacier   │S3 Glacier Deep   │
    │           │    -IA    │    -IA     │Instant/Flex. │    Archive       │
    │ Frequent  │ Infrequent│ Infrequent │   Archive    │ Long-term        │
    │           │ (30-day)  │ (30-day)   │              │ (180-day min)    │
    │ ~$0.023/GB│ ~$0.0125  │ ~$0.01     │ ~$0.004      │ ~$0.00099/GB     │
    └──────────┴───────────┴────────────┴──────────────┴──────────────────┘

Azure Blob:
    ┌──────────┬───────────┬────────────┬──────────────┐
    │  Hot     │   Cool    │   Cold     │   Archive    │
    │          │ (30-day)  │ (90-day)   │ (180-day)    │
    │~$0.018/GB│ ~$0.01    │ ~$0.0045   │ ~$0.00099/GB │
    └──────────┴───────────┴────────────┴──────────────┘

GCP Cloud Storage:
    ┌──────────┬───────────┬────────────┬──────────────┐
    │ Standard │  Nearline │  Coldline  │   Archive    │
    │          │ (30-day)  │ (90-day)   │ (365-day)    │
    │~$0.020/GB│ ~$0.01    │ ~$0.004    │ ~$0.0012/GB  │
    └──────────┴───────────┴────────────┴──────────────┘
```

### Storage Comparison Table

| Feature | AWS S3 | Azure Blob Storage | GCP Cloud Storage |
|---------|--------|-------------------|-------------------|
| **Max Object Size** | 5 TB | 4.75 TB (block blob) | 5 TB |
| **Bucket/Container** | Bucket (global unique name) | Container (in Storage Account) | Bucket (global unique name) |
| **Versioning** | ✅ Per bucket | ✅ Soft delete + versioning | ✅ Object versioning |
| **Lifecycle** | ✅ Lifecycle policies | ✅ Lifecycle management | ✅ Lifecycle rules |
| **Encryption at Rest** | SSE-S3, SSE-KMS, SSE-C | Microsoft-managed, CMK | Google-managed, CMEK, CSEK |
| **Static Website** | ✅ S3 Static Hosting | ✅ $web container | ✅ Cloud Storage website |
| **Access Control** | Bucket Policy + ACL + IAM | Shared Access Signature (SAS) + RBAC | IAM + ACL + Signed URLs |
| **Replication** | Cross-Region Replication (CRR) | Geo-Redundant (GRS/GZRS) | Dual/Multi-Region buckets |
| **Event Trigger** | S3 Events → Lambda/SQS/SNS | Blob Events → Event Grid | GCS Events → Pub/Sub/Functions |

---

## 12. Database Deep Dive

### Database Decision Tree

```
What type of data?
│
├── Structured (SQL) ──────────────────────────────────────────────┐
│   │                                                              │
│   ├── OLTP (Transactional)                                       │
│   │   ├── AWS:  RDS (MySQL/PostgreSQL/Oracle/SQL Server)         │
│   │   │         Aurora (MySQL/PostgreSQL compatible, 5x faster)  │
│   │   ├── Azure: Azure SQL Database, PostgreSQL Flexible Server  │
│   │   └── GCP:  Cloud SQL, AlloyDB (PostgreSQL compatible)       │
│   │                                                              │
│   └── OLAP (Analytical / Warehouse)                              │
│       ├── AWS:  Redshift                                         │
│       ├── Azure: Synapse Analytics                               │
│       └── GCP:  BigQuery (serverless!)                           │
│                                                                  │
├── Semi-Structured (NoSQL) ───────────────────────────────────────┤
│   │                                                              │
│   ├── Key-Value                                                  │
│   │   ├── AWS:  DynamoDB                                         │
│   │   ├── Azure: Cosmos DB (Table API)                           │
│   │   └── GCP:  Firestore / Bigtable                            │
│   │                                                              │
│   ├── Document                                                   │
│   │   ├── AWS:  DynamoDB / DocumentDB (MongoDB compatible)       │
│   │   ├── Azure: Cosmos DB (Core/SQL API, MongoDB API)           │
│   │   └── GCP:  Firestore (Native mode)                         │
│   │                                                              │
│   ├── Wide-Column                                                │
│   │   ├── AWS:  Keyspaces (Cassandra)                            │
│   │   ├── Azure: Cosmos DB (Cassandra API)                       │
│   │   └── GCP:  Bigtable                                        │
│   │                                                              │
│   └── Graph                                                      │
│       ├── AWS:  Neptune                                          │
│       ├── Azure: Cosmos DB (Gremlin API)                         │
│       └── GCP:  No native (use Neo4j on GCE)                    │
│                                                                  │
├── In-Memory Cache ───────────────────────────────────────────────┤
│   ├── AWS:  ElastiCache (Redis / Memcached)                      │
│   ├── Azure: Azure Cache for Redis                               │
│   └── GCP:  Memorystore (Redis / Memcached)                     │
│                                                                  │
└── Time-Series / Search ──────────────────────────────────────────┘
    ├── AWS:  Timestream (TS), OpenSearch (Search)                 │
    ├── Azure: ADX (TS), Cognitive Search (Search)                 │
    └── GCP:  Bigtable (TS), Vertex AI Search (Search)            │
```

---

## 13. Serverless & Event-Driven

### Serverless Compute Comparison

| Feature | AWS Lambda | Azure Functions | GCP Cloud Functions |
|---------|-----------|-----------------|-------------------|
| **Languages** | Python, Node, Java, Go, .NET, Ruby, Custom | C#, Python, Node, Java, PowerShell, Custom | Python, Node, Java, Go, .NET, Ruby, PHP |
| **Max Timeout** | 15 minutes | Consumption: 10min, Premium: unlimited | 1st gen: 9min, 2nd gen: 60min |
| **Max Memory** | 10,240 MB | 1,536 MB (Consumption) | 32 GB (2nd gen) |
| **Concurrency** | 1000 (default, adjustable) | 200 (per instance) | 1000 (per region) |
| **Cold Start** | ~100ms-1s | ~1-3s (Consumption) | ~100ms-1s |
| **Pricing** | Per request + duration (GB-sec) | Per execution + duration | Per invocation + duration |
| **VPC Access** | ✅ (VPC config) | ✅ (VNet Integration) | ✅ (VPC Connector) |
| **Container Image** | ✅ Up to 10GB | ✅ Custom containers | ✅ (2nd gen via Cloud Run) |
| **Triggers** | 200+ AWS service triggers | HTTP, Timer, Queue, Blob, etc. | HTTP, Pub/Sub, GCS, Firestore |

### Event-Driven Architecture Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Event Source │     │  Event Bus   │     │  Handler     │
│              │────►│              │────►│  (Function)  │
│  S3 Upload   │     │ EventBridge  │     │  Lambda      │
│  Blob Change │     │ Event Grid   │     │  Az Function │
│  GCS Object  │     │ Eventarc     │     │  Cloud Func  │
└──────────────┘     └──────┬───────┘     └──────┬───────┘
                            │                     │
                     ┌──────┴───────┐     ┌──────┴───────┐
                     │  Filter/Rule │     │  Downstream  │
                     │  Route by    │     │  DB Write    │
                     │  event type  │     │  Notification│
                     └──────────────┘     │  API Call    │
                                          └──────────────┘
```

---

## 14. Containers & Orchestration

### Container Services Hierarchy

```
                        Container Services
                              │
            ┌─────────────────┼──────────────────┐
            │                 │                  │
      ┌─────┴─────┐    ┌─────┴─────┐     ┌─────┴─────┐
      │  Registry  │    │Orchestrat.│     │ Serverless │
      │            │    │           │     │ Containers │
      ├────────────┤    ├───────────┤     ├────────────┤
      │AWS: ECR    │    │AWS: EKS   │     │AWS: Fargate│
      │Azure: ACR  │    │    ECS    │     │Azure:      │
      │GCP:Artifact│    │Azure: AKS │     │ Container  │
      │   Registry │    │GCP: GKE   │     │ Apps       │
      └────────────┘    └───────────┘     │GCP:Cloud   │
                                          │ Run        │
                                          └────────────┘
```

### Kubernetes (Managed) Comparison

| Feature | AWS EKS | Azure AKS | GCP GKE |
|---------|---------|-----------|---------|
| **Control Plane Cost** | ~$73/month | FREE | FREE (Standard), $73 (Enterprise) |
| **Max Nodes** | 5,000 per cluster | 5,000 per cluster | 15,000 per cluster |
| **Auto-scaling** | Cluster Autoscaler / Karpenter | Cluster Autoscaler / KEDA | Cluster Autoscaler / Autopilot |
| **Serverless Mode** | Fargate Profiles | Virtual Nodes (ACI) | GKE Autopilot |
| **Service Mesh** | App Mesh / Istio | Istio (addon) | Anthos Service Mesh (Istio) |
| **Networking** | VPC CNI (native) | Azure CNI / kubenet | GKE VPC-native |
| **GPU Support** | ✅ | ✅ | ✅ |
| **Windows Nodes** | ✅ | ✅ | ✅ |
| **Multi-cluster** | EKS Anywhere | AKS Fleet Manager | GKE Enterprise (Anthos) |

> **🔑 Key Insight:** GKE is widely considered the most mature managed Kubernetes service (Google invented Kubernetes). AKS has free control plane. EKS charges for control plane.

---

## 15. Monitoring, Logging & Observability

### Observability Stack — All Three

```
                    ┌──────────────────────────────────┐
                    │       OBSERVABILITY PILLARS       │
                    ├──────────┬──────────┬────────────┤
                    │ METRICS  │  LOGS    │  TRACES    │
                    ├──────────┼──────────┼────────────┤
                    │          │          │            │
          AWS       │CloudWatch│CloudWatch│  X-Ray     │
                    │ Metrics  │  Logs    │            │
                    ├──────────┼──────────┼────────────┤
                    │          │          │            │
         Azure      │ Monitor  │  Log     │Application │
                    │ Metrics  │Analytics │ Insights   │
                    ├──────────┼──────────┼────────────┤
                    │          │          │            │
          GCP       │  Cloud   │  Cloud   │  Cloud     │
                    │Monitoring│ Logging  │  Trace     │
                    └──────────┴──────────┴────────────┘
                              │
                              ▼
                    ┌──────────────────────┐
                    │    ALERTING          │
                    │ AWS: CloudWatch Alarms│
                    │ Azure: Monitor Alerts│
                    │ GCP: Alerting Policies│
                    └──────────┬───────────┘
                              │
                              ▼
                    ┌──────────────────────┐
                    │   NOTIFICATION       │
                    │ AWS: SNS → Email/SMS │
                    │ Azure: Action Groups │
                    │ GCP: Notification Ch.│
                    └──────────────────────┘
```

---

## 16. Security & Compliance

### Defense in Depth — All Providers

```
Layer 1: Physical Security
│   └── Data center access, biometrics (Provider managed)
│
├── Layer 2: Identity & Access
│   └── IAM / Entra ID / Cloud IAM + MFA + Federation
│
├── Layer 3: Network Security
│   ├── VPC/VNet isolation
│   ├── Security Groups / NSG / Firewall Rules
│   ├── WAF / DDoS protection
│   └── Private endpoints / VPC endpoints
│
├── Layer 4: Compute Security
│   ├── Patching (SSM / Update Manager / OS Patch)
│   ├── Hardened images (AMI / Managed Image / Custom Image)
│   └── Confidential computing (Nitro / SGX / Confidential VMs)
│
├── Layer 5: Application Security
│   ├── Secrets management (Secrets Manager / Key Vault / Secret Manager)
│   ├── Certificate management (ACM / App Service Certs / Certificate Manager)
│   └── Runtime protection (GuardDuty / Defender / SCC)
│
├── Layer 6: Data Security
│   ├── Encryption at rest (KMS / Key Vault / Cloud KMS)
│   ├── Encryption in transit (TLS everywhere)
│   └── Data classification (Macie / Purview / DLP)
│
└── Layer 7: Governance & Compliance
    ├── AWS: Config, Security Hub, Control Tower
    ├── Azure: Policy, Defender for Cloud, Sentinel
    └── GCP: Org Policy, Security Command Center, Chronicle
```

---

## 17. CI/CD & DevOps

### CI/CD Pipeline — Cross-Cloud Comparison

```
Developer
    │
    ▼
┌─────────────────┐     ┌────────────────┐     ┌──────────────────┐
│  Source Control  │────►│    Build        │────►│    Test           │
│                 │     │                │     │                  │
│ AWS: CodeCommit │     │ AWS: CodeBuild  │     │ Unit + Integration│
│ Azure: Repos    │     │ Azure: Pipelines│     │ + Security Scan  │
│ GCP: CSR        │     │ GCP: Cloud Build│     │                  │
│ (All: GitHub)   │     │                │     │                  │
└─────────────────┘     └────────────────┘     └────────┬─────────┘
                                                         │
                        ┌────────────────┐     ┌────────┴─────────┐
                        │   Deploy        │◄───│  Artifact Store   │
                        │                │     │                  │
                        │ AWS: CodeDeploy │     │ AWS: ECR / S3    │
                        │ Azure: Pipelines│     │ Azure: ACR / Blob│
                        │ GCP: Cloud Dep. │     │ GCP: Artifact Reg│
                        └───────┬────────┘     └──────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                  ▼
        ┌──────────┐    ┌──────────┐       ┌──────────┐
        │   Dev    │    │  Staging  │       │   Prod   │
        │  Env     │    │   Env     │       │   Env    │
        └──────────┘    └──────────┘       └──────────┘
```

### Infrastructure as Code (IaC) Comparison

| Tool | AWS | Azure | GCP | Multi-Cloud |
|------|-----|-------|-----|-------------|
| **Native** | CloudFormation | ARM Templates / Bicep | Deployment Manager | ❌ |
| **Terraform** | ✅ AWS Provider | ✅ AzureRM Provider | ✅ Google Provider | ✅ |
| **Pulumi** | ✅ | ✅ | ✅ | ✅ |
| **CDK** | AWS CDK | Azure CDK (preview) | ❌ | CDKTF |
| **Crossplane** | ✅ | ✅ | ✅ | ✅ (K8s native) |

---

## 18. Cost Management

### Cost Optimization Strategies

| Strategy | AWS | Azure | GCP |
|----------|-----|-------|-----|
| **Reserved/Committed** | Reserved Instances, Savings Plans | Reserved VMs, Savings Plans | Committed Use Discounts |
| **Spot/Preemptible** | Spot Instances (up to 90% off) | Spot VMs (up to 90% off) | Spot VMs (up to 91% off) |
| **Auto-Sustained** | ❌ | ❌ | ✅ Sustained Use Discounts |
| **Right-sizing** | Compute Optimizer | Azure Advisor | Recommender |
| **Cost Visibility** | Cost Explorer, Budgets | Cost Management + Billing | Billing Reports, Budgets |
| **Free Tier** | 12 months + Always Free | 12 months + Always Free | Always Free (generous) |
| **Calculator** | AWS Pricing Calculator | Azure Pricing Calculator | GCP Pricing Calculator |

---

## 19. Production Architecture Flow

### Full Production-Grade Architecture (3-Tier Web App)

```
                                    INTERNET
                                       │
                                       ▼
                              ┌─────────────────┐
                              │   DNS            │
                              │ Route53/AzDNS/   │
                              │ CloudDNS         │
                              └────────┬─────────┘
                                       │
                              ┌────────┴─────────┐
                              │   CDN             │
                              │ CloudFront/       │
                              │ Front Door/       │
                              │ Cloud CDN         │
                              └────────┬─────────┘
                                       │
                              ┌────────┴─────────┐
                              │   WAF + DDoS      │
                              │ AWS WAF+Shield/   │
                              │ Az WAF+DDoS/      │
                              │ Cloud Armor       │
                              └────────┬─────────┘
                                       │
                              ┌────────┴─────────┐
                              │  Load Balancer    │
                              │  (L7 - HTTPS)     │
                              │  ALB/AppGW/       │
                              │  HTTPS LB         │
                              └────────┬─────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                   │
                    ▼                  ▼                   ▼
            ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
            │  Web Tier     │  │  Web Tier     │  │  Web Tier     │
            │  (AZ-1)       │  │  (AZ-2)       │  │  (AZ-3)       │
            │  ┌──────────┐ │  │  ┌──────────┐ │  │  ┌──────────┐ │
            │  │Container │ │  │  │Container │ │  │  │Container │ │
            │  │or VM     │ │  │  │or VM     │ │  │  │or VM     │ │
            │  └─────┬────┘ │  │  └─────┬────┘ │  │  └─────┬────┘ │
            └────────┼──────┘  └────────┼──────┘  └────────┼──────┘
                     │                  │                   │
            ┌────────┴──────────────────┴───────────────────┤
            │          Internal Load Balancer (L4)          │
            │          NLB / ILB / Internal TCP LB          │
            └───────────────────┬───────────────────────────┘
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
            ┌────────────┐ ┌────────────┐ ┌────────────┐
            │  App Tier   │ │  App Tier   │ │  App Tier   │
            │  (AZ-1)     │ │  (AZ-2)     │ │  (AZ-3)     │
            │  ┌────────┐ │ │  ┌────────┐ │ │  ┌────────┐ │
            │  │API Pods│ │ │  │API Pods│ │ │  │API Pods│ │
            │  │or VMs  │ │ │  │or VMs  │ │ │  │or VMs  │ │
            │  └───┬────┘ │ │  └───┬────┘ │ │  └───┬────┘ │
            └──────┼──────┘ └──────┼──────┘ └──────┼──────┘
                   │               │               │
            ┌──────┴───────────────┴───────────────┤
            │                                      │
            │    ┌───────────┐  ┌───────────┐      │
            │    │  Cache     │  │  Queue     │      │
            │    │ Redis      │  │ SQS/SBus/  │      │
            │    │ ElastiCache│  │ Pub/Sub    │      │
            │    └───────────┘  └───────────┘      │
            │                                      │
            ├──────────────────────────────────────┤
            │          Data Tier                    │
            │  ┌──────────┐    ┌──────────┐        │
            │  │ Primary  │───►│ Replica  │        │
            │  │  DB      │    │   DB     │        │
            │  │ (AZ-1)   │    │ (AZ-2)   │        │
            │  │RDS/AzSQL/│    │          │        │
            │  │CloudSQL  │    │          │        │
            │  └──────────┘    └──────────┘        │
            │                                      │
            │  ┌──────────────────────────┐        │
            │  │   Object Storage          │        │
            │  │   S3 / Blob / GCS         │        │
            │  │   (Static assets, backups)│        │
            │  └──────────────────────────┘        │
            └──────────────────────────────────────┘
                                │
                                ▼
            ┌──────────────────────────────────────┐
            │         Monitoring & Logging          │
            │  CloudWatch / Azure Monitor / Cloud   │
            │  Monitoring + Logging + Tracing       │
            │  ┌──────────┐  ┌───────┐ ┌────────┐  │
            │  │ Metrics  │  │ Logs  │ │ Traces │  │
            │  │ Alerts   │  │ Query │ │ X-Ray/ │  │
            │  │ Dashboard│  │ Retain│ │ Insight│  │
            │  └──────────┘  └───────┘ └────────┘  │
            └──────────────────────────────────────┘
```

---

## 20. Shared Responsibility Model

> This is arguably **the most important concept in cloud computing** — understanding what YOU are responsible for vs what the CLOUD PROVIDER manages.

### The Model Visualized

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     SHARED RESPONSIBILITY MODEL                              │
│                                                                              │
│   CUSTOMER RESPONSIBILITY ("Security IN the Cloud")                         │
│   ════════════════════════════════════════════════                           │
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                                                                      │  │
│   │  • Customer Data (encryption, classification, access control)       │  │
│   │  • Identity & Access Management (IAM policies, MFA, passwords)      │  │
│   │  • Application Security (code, dependencies, runtime config)        │  │
│   │  • OS Patching & Hardening (for IaaS — VMs)                         │  │
│   │  • Network Configuration (Security Groups, NACLs, firewall rules)   │  │
│   │  • Client-Side Encryption & Data Integrity                          │  │
│   │  • Firewall / Network Traffic Rules                                  │  │
│   │                                                                      │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   ─── Responsibility shifts based on service model ───                      │
│                                                                              │
│   IaaS (EC2/VM/GCE):     Customer manages OS, runtime, app, data           │
│   PaaS (RDS/App Svc):    Customer manages app + data only                   │
│   SaaS (M365/Workspace): Customer manages data + access only               │
│                                                                              │
│   ═══════════════════════════════════════════════════════════════════════    │
│                                                                              │
│   PROVIDER RESPONSIBILITY ("Security OF the Cloud")                         │
│   ═════════════════════════════════════════════════                          │
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                                                                      │  │
│   │  • Physical Data Centers (access, power, cooling, fire suppression) │  │
│   │  • Hardware (servers, storage arrays, networking equipment)          │  │
│   │  • Hypervisor / Virtualization Layer                                 │  │
│   │  • Global Network Infrastructure (fiber, edge locations)            │  │
│   │  • Host OS & Firmware Patching (underlying infra)                   │  │
│   │  • Managed Service Internals (RDS engine patching, etc.)            │  │
│   │  • Compliance Certifications (SOC 2, ISO 27001, etc.)               │  │
│   │                                                                      │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Responsibility Breakdown by Service Model

```
                  On-Prem    IaaS        PaaS         SaaS
                  ───────    ────        ────         ────
Data              YOU        YOU         YOU          YOU ← Always yours
Identity/Access   YOU        YOU         YOU          YOU ← Always yours
Application       YOU        YOU         YOU          PROVIDER
OS / Runtime      YOU        YOU         PROVIDER     PROVIDER
Networking Cfg    YOU        YOU (VPC)   SHARED       PROVIDER
Virtualization    YOU        PROVIDER    PROVIDER     PROVIDER
Physical Servers  YOU        PROVIDER    PROVIDER     PROVIDER
Storage HW        YOU        PROVIDER    PROVIDER     PROVIDER
Physical DC       YOU        PROVIDER    PROVIDER     PROVIDER
```

### Provider-Specific Names & Documentation

| Aspect | AWS | Azure | GCP |
|--------|-----|-------|-----|
| **Model Name** | Shared Responsibility Model | Shared Responsibility Model | Shared Fate Model |
| **Philosophy** | Clear boundary — your side/our side | Clear boundary — your side/our side | **Shared Fate** — Google actively helps you stay secure |
| **Security Tools** | GuardDuty, Inspector, Security Hub | Defender for Cloud, Sentinel | Security Command Center (SCC) |
| **Best Practice Guide** | Well-Architected (Security Pillar) | Cloud Adoption Framework | Architecture Framework |
| **Compliance Dashboard** | AWS Artifact | Trust Center | Compliance Reports Manager |

> **🔑 Key Insight:** GCP calls their model **Shared Fate** instead of Shared Responsibility — emphasizing that Google takes an active role in helping customers stay secure (via Assured Workloads, Security Command Center, etc.) rather than just drawing a line.

---

## 21. Well-Architected Framework

> All three providers publish a set of **design principles and best practices** for building reliable, secure, efficient, and cost-effective workloads in the cloud.

### Framework Pillars — Comparison

```
┌──────────────────────────────────────────────────────────────────────────────┐
│              WELL-ARCHITECTED FRAMEWORK — PILLARS                            │
├──────────────────┬──────────────────┬────────────────────────────────────────┤
│      AWS (6)     │    Azure (5)     │         GCP (6)                        │
│  Well-Architected│  Well-Architected│   Architecture Framework               │
├──────────────────┼──────────────────┼────────────────────────────────────────┤
│                  │                  │                                        │
│ 1. Operational   │ 1. Operational   │ 1. Operational Excellence             │
│    Excellence    │    Excellence    │                                        │
│                  │                  │                                        │
│ 2. Security      │ 2. Security      │ 2. Security, Privacy &               │
│                  │                  │    Compliance                          │
│                  │                  │                                        │
│ 3. Reliability   │ 3. Reliability   │ 3. Reliability                        │
│                  │                  │                                        │
│ 4. Performance   │ 4. Performance   │ 4. Performance Optimization           │
│    Efficiency    │    Efficiency    │                                        │
│                  │                  │                                        │
│ 5. Cost          │ 5. Cost          │ 5. Cost Optimization                  │
│    Optimization  │    Optimization  │                                        │
│                  │                  │                                        │
│ 6. Sustainability│                  │ 6. System Design                      │
│    (added 2021)  │                  │    (includes sustainability)           │
│                  │                  │                                        │
└──────────────────┴──────────────────┴────────────────────────────────────────┘
```

### Key Principles Per Pillar

```
┌─────────────────────────────────────────────────────────────────────────┐
│  OPERATIONAL EXCELLENCE                                                 │
│  ├── Automate everything (IaC, CI/CD, runbooks)                        │
│  ├── Make frequent, small, reversible changes                          │
│  ├── Anticipate failure — run game days / chaos engineering            │
│  └── Learn from operational events → improve                          │
│                                                                         │
│  SECURITY                                                               │
│  ├── Implement least privilege access                                  │
│  ├── Enable traceability (audit logs everywhere)                       │
│  ├── Encrypt data at rest AND in transit                               │
│  ├── Automate security best practices                                  │
│  └── Protect data in all layers (defense in depth)                     │
│                                                                         │
│  RELIABILITY                                                            │
│  ├── Design for failure (assume everything fails)                      │
│  ├── Test recovery procedures regularly                                │
│  ├── Scale horizontally to increase availability                       │
│  ├── Stop guessing capacity — auto-scale                               │
│  └── Manage change through automation                                  │
│                                                                         │
│  PERFORMANCE EFFICIENCY                                                 │
│  ├── Use the right resource type for your workload                     │
│  ├── Go global in minutes (multi-region)                               │
│  ├── Use serverless where possible                                     │
│  ├── Experiment more often (A/B test architectures)                    │
│  └── Consider mechanical sympathy (understand underlying tech)         │
│                                                                         │
│  COST OPTIMIZATION                                                      │
│  ├── Implement cloud financial management (FinOps)                     │
│  ├── Adopt a consumption model (pay for what you use)                  │
│  ├── Measure overall efficiency (cost per transaction)                 │
│  ├── Stop spending on undifferentiated heavy lifting                   │
│  └── Analyze and attribute expenditure (tagging, cost allocation)      │
│                                                                         │
│  SUSTAINABILITY (AWS) / SYSTEM DESIGN (GCP)                            │
│  ├── Understand your impact (carbon footprint)                         │
│  ├── Maximize utilization (right-size, spot instances)                 │
│  ├── Use managed services (provider optimizes efficiency)              │
│  └── Reduce downstream impact (data transfer optimization)            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Review Tools

| Feature | AWS | Azure | GCP |
|---------|-----|-------|-----|
| **Review Tool** | AWS Well-Architected Tool | Azure Advisor + WAF Review | Architecture Framework checklists |
| **Free Review** | ✅ Self-service in console | ✅ Azure Advisor (automated) | ✅ Self-service checklists |
| **Partner Review** | AWS Partner WAFR | Azure Expert MSP | Google Cloud Partner |
| **Lens/Workloads** | Lenses (SaaS, Serverless, ML, etc.) | Azure Workbooks | Workload-specific guides |

---

## 22. High Availability & Disaster Recovery

> HA = **preventing downtime.** DR = **recovering from catastrophic failure.** They are related but distinct concepts.

### HA vs DR Explained

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  HIGH AVAILABILITY (HA)                    DISASTER RECOVERY (DR)       │
│  ═══════════════════════                   ═══════════════════════      │
│                                                                          │
│  Goal: Minimize downtime                   Goal: Recover from           │
│        (keep running)                             catastrophic failure  │
│                                                                          │
│  Scope: Within a region                    Scope: Cross-region          │
│         (multi-AZ/zone)                           (or cross-provider)   │
│                                                                          │
│  Metric: Availability %                    Metric: RTO + RPO            │
│          (99.9%, 99.99%)                                                 │
│                                                                          │
│  Example: 2 VMs behind a                  Example: Full replica in     │
│           load balancer                             another region      │
│           in different AZs                                               │
│                                                                          │
│  Cost: Moderate (2x compute)              Cost: Varies by strategy     │
│                                                                          │
│  RTO = Recovery Time Objective     (How LONG can you be down?)          │
│  RPO = Recovery Point Objective    (How much DATA can you lose?)        │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### DR Strategies — From Cheapest to Fastest Recovery

```
     COST ──────────────────────────────────────────────────► HIGH
     RTO  ──────────────────────────────────────────────────► LOW

 ┌────────────┐  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐
 │   Backup   │  │  Pilot      │  │    Warm      │  │  Multi-Site   │
 │  & Restore │  │  Light      │  │   Standby    │  │ Active-Active │
 ├────────────┤  ├─────────────┤  ├──────────────┤  ├───────────────┤
 │            │  │             │  │              │  │               │
 │ RPO: Hours │  │ RPO: Minutes│  │ RPO: Seconds │  │ RPO: Near-Zero│
 │ RTO: Hours │  │ RTO: 10min+ │  │ RTO: Minutes │  │ RTO: Near-Zero│
 │            │  │             │  │              │  │               │
 │ Backups    │  │ Core infra  │  │ Scaled-down  │  │ Full replica  │
 │ stored in  │  │ running     │  │ copy running │  │ in 2+ regions │
 │ another    │  │ (DB replica,│  │ in DR region │  │ both serving  │
 │ region     │  │  min compute│  │ ready to     │  │ traffic       │
 │            │  │  rest OFF)  │  │ scale up     │  │               │
 │ Restore    │  │ Scale up    │  │ Quick        │  │ DNS failover  │
 │ from       │  │ when needed │  │ failover     │  │ automatic     │
 │ backup     │  │             │  │              │  │               │
 │            │  │             │  │              │  │               │
 │ Cost: $    │  │ Cost: $$    │  │ Cost: $$$    │  │ Cost: $$$$    │
 └────────────┘  └─────────────┘  └──────────────┘  └───────────────┘
```

### DR Services — Provider Comparison

| DR Capability | AWS | Azure | GCP |
|--------------|-----|-------|-----|
| **Cross-Region Replication** | S3 CRR, RDS Read Replicas, DynamoDB Global Tables | Geo-Redundant Storage (GRS), Azure SQL Geo-Replication | Multi-Region Cloud Storage, Cloud SQL Cross-Region Replicas |
| **DB Failover** | RDS Multi-AZ, Aurora Global Database | Azure SQL Failover Groups, Cosmos DB Multi-Region | Cloud SQL HA, Spanner (global) |
| **Site Recovery / DR Service** | Elastic Disaster Recovery (DRS) | Azure Site Recovery (ASR) | No native equivalent (use Terraform/scripts) |
| **DNS Failover** | Route 53 Health Checks + Failover Routing | Azure Traffic Manager / Front Door | Cloud DNS + Global LB Health Checks |
| **Backup Service** | AWS Backup | Azure Backup | Backup and DR Service |
| **Built-in Region Pairing** | ❌ Manual | ✅ Automatic Region Pairs | ❌ Manual |
| **Globally Distributed DB** | DynamoDB Global Tables, Aurora Global | Cosmos DB (multi-region writes) | Spanner (globally consistent!) |

### Availability Tiers — What the 9s Mean

```
 Availability %    Downtime/Year     Downtime/Month    Typical Use
 ──────────────    ──────────────    ──────────────    ────────────
 99%    (two 9s)   3.65 days         7.3 hours         Internal tools
 99.9%  (three 9s) 8.76 hours        43.8 minutes      Business apps
 99.95%            4.38 hours        21.9 minutes      Important apps
 99.99% (four 9s)  52.6 minutes      4.38 minutes      Critical apps
 99.999%(five 9s)  5.26 minutes      26.3 seconds      Financial/Health
```

---

## 23. Data Transfer & Egress Costs

> This is the **#1 surprise cost** for cloud newcomers. Data going INTO the cloud is usually free. Data going OUT is NOT.

### The Golden Rule of Cloud Networking Costs

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    DATA TRANSFER COST RULES                              │
│                                                                          │
│                          ┌──────────────┐                                │
│                          │    CLOUD      │                                │
│                          │              │                                │
│        INGRESS ─────────►│   Region     │────────► EGRESS                │
│        (data IN)         │              │          (data OUT)            │
│        FREE ✅           │              │          PAID 💰               │
│                          └──────────────┘                                │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                                                                   │  │
│  │  ✅ FREE:                                                        │  │
│  │  ├── Internet → Cloud (ingress / upload)                        │  │
│  │  ├── Same AZ/Zone traffic (within same AZ)                      │  │
│  │  ├── Traffic to managed services via private endpoint (varies)  │  │
│  │                                                                   │  │
│  │  💰 COSTS MONEY:                                                 │  │
│  │  ├── Cloud → Internet (egress / download)        ~$0.08-0.12/GB │  │
│  │  ├── Cross-AZ/Zone traffic (same region)         ~$0.01-0.02/GB │  │
│  │  ├── Cross-Region traffic                        ~$0.02-0.08/GB │  │
│  │  ├── Cross-Cloud traffic (e.g., AWS → GCP)       Full egress    │  │
│  │  └── VPN / Interconnect data transfer             Varies         │  │
│  │                                                                   │  │
│  │  ⚠️ COMMON GOTCHAS:                                              │  │
│  │  ├── Multi-AZ deployments = 2x cross-AZ charges on traffic     │  │
│  │  ├── CDN can REDUCE egress (cheaper edge pricing)               │  │
│  │  ├── VPC Peering cross-region = egress-like pricing             │  │
│  │  └── NAT Gateway charges per GB processed (AWS: $0.045/GB)     │  │
│  │                                                                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Egress Pricing Comparison (Approximate — First 10 TB/month)

| Transfer Type | AWS | Azure | GCP |
|--------------|-----|-------|-----|
| **Ingress (Internet → Cloud)** | FREE | FREE | FREE |
| **Egress (Cloud → Internet)** | ~$0.09/GB | ~$0.087/GB | ~$0.12/GB (Premium) / ~$0.085 (Standard) |
| **Cross-AZ (same region)** | $0.01/GB (each direction) | FREE (within VNet) | $0.01/GB |
| **Cross-Region (same cloud)** | $0.02/GB | $0.02-0.08/GB | $0.01-0.08/GB |
| **First 100 GB/month** | FREE (free tier) | FREE (5 GB) | FREE (free tier) |
| **CDN Egress** | Cheaper than direct (~$0.085/GB) | Cheaper via Front Door | Cheaper via Cloud CDN |
| **NAT Gateway processing** | $0.045/GB | Included | Included |

### Cost Optimization Tips for Data Transfer

```
┌─────────────────────────────────────────────────────────────────────┐
│  HOW TO REDUCE DATA TRANSFER COSTS                                  │
│                                                                     │
│  1. Use CDN for static content (cheaper egress rates)              │
│  2. Compress data before transfer (gzip/brotli)                    │
│  3. Use VPC/Private Endpoints instead of public internet           │
│  4. Keep tightly-coupled services in the SAME AZ                   │
│  5. Use same-region replicas instead of cross-region when possible │
│  6. Batch API calls to reduce number of transfers                  │
│  7. Cache aggressively (Redis/CloudFront/CDN)                      │
│  8. Consider GCP Standard Tier (cheaper egress, less PoPs)         │
│  9. Use Direct Connect / ExpressRoute / Interconnect for high      │
│     volume (cheaper per-GB than internet egress)                   │
│  10. Monitor egress with cost allocation tags                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 24. Compliance Frameworks & SLAs

### Common Compliance Frameworks

```
┌──────────────────────────────────────────────────────────────────────────┐
│               COMPLIANCE FRAMEWORKS IN CLOUD                             │
│                                                                          │
│  ┌─── Industry-Agnostic ─────────────────────────────────────────────┐ │
│  │ SOC 1 / SOC 2 / SOC 3     Financial & operational controls       │ │
│  │ ISO 27001 / 27017 / 27018  Information security management       │ │
│  │ CSA STAR                   Cloud-specific security assessment     │ │
│  │ ISO 9001                   Quality management                    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌─── Industry-Specific ─────────────────────────────────────────────┐ │
│  │ HIPAA / HITECH             Healthcare (US)                        │ │
│  │ PCI-DSS                    Payment card processing                │ │
│  │ FedRAMP                    US Federal government                  │ │
│  │ GDPR                       EU data privacy                        │ │
│  │ CCPA / CPRA                California privacy                    │ │
│  │ FIPS 140-2/3               Cryptographic module validation       │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌─── Regional / Sovereign ──────────────────────────────────────────┐ │
│  │ AWS GovCloud               US Gov workloads                       │ │
│  │ Azure Government            US Gov (IL4/IL5/IL6)                  │ │
│  │ GCP Assured Workloads      Regulatory compliance (US/EU/etc.)    │ │
│  │ Azure China (21Vianet)     China data residency                   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ⚠️ IMPORTANT: Cloud providers are CERTIFIED — but your USE of the     │
│  cloud must also be compliant. The provider gives you the tools and    │
│  certified infrastructure; you must configure them correctly.           │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Compliance Resources

| Resource | AWS | Azure | GCP |
|----------|-----|-------|-----|
| **Compliance Reports** | AWS Artifact | Azure Trust Center / Service Trust Portal | Compliance Reports Manager |
| **Compliance Programs** | 143+ | 100+ | 100+ |
| **Audit-Ready** | ✅ SOC 2, ISO, PCI, HIPAA | ✅ SOC 2, ISO, PCI, HIPAA | ✅ SOC 2, ISO, PCI, HIPAA |
| **Data Residency Controls** | Region selection + SCPs | Azure Policy + Sovereign Regions | Org Policy + Assured Workloads |
| **Privacy Tool** | Macie (PII detection) | Purview (data governance) | DLP API (sensitive data) |

### SLA (Service Level Agreement) Comparison

| Service Type | AWS SLA | Azure SLA | GCP SLA |
|-------------|---------|-----------|---------|
| **Single VM** | 99.5% (EBS-backed) | 99.9% (Premium SSD) | 99.99% (single instance) |
| **Multi-AZ VM** | 99.99% | 99.99% (Avail. Zones) | 99.99% (regional MIG) |
| **Managed DB (Multi-AZ)** | 99.95% (RDS) | 99.99% (Azure SQL) | 99.95% (Cloud SQL HA) |
| **Object Storage** | 99.99% (S3 Standard) | 99.9% (Hot LRS) | 99.95% (Standard Regional) |
| **Serverless Functions** | 99.95% (Lambda) | 99.95% (Functions) | 99.99% (Cloud Functions) |
| **Kubernetes** | 99.95% (EKS) | 99.95% (AKS with SLA) | 99.95% (GKE Standard) |
| **Global DB** | 99.999% (DynamoDB Global) | 99.999% (Cosmos DB) | 99.999% (Spanner) |

> **🔑 Key Insight:** GCP offers **99.99% SLA for a single VM instance** — the highest among all three providers for single instances. Spanner and Cosmos DB both offer 99.999% (five 9s) for globally distributed databases.

### How SLA Credits Work

```
┌─────────────────────────────────────────────────────────────────────┐
│  SLA BREACH → SERVICE CREDIT (bill reduction, NOT cash refund)     │
│                                                                     │
│  Typical Credit Schedule:                                           │
│  ├── < SLA but ≥ 99.0%     →  10% credit                          │
│  ├── < 99.0% but ≥ 95.0%  →  25% credit                          │
│  └── < 95.0%               →  100% credit (full month)            │
│                                                                     │
│  ⚠️ You must FILE A CLAIM — credits are NOT automatic              │
│  ⚠️ Credits apply to FUTURE bills — no cash refunds                │
│  ⚠️ SLA only applies if you follow the architecture guidelines     │
│     (e.g., multi-AZ deployment for 99.99% SLA)                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference — Certification Paths

| Level | AWS | Azure | GCP |
|-------|-----|-------|-----|
| **Foundational** | Cloud Practitioner | AZ-900 Fundamentals | Cloud Digital Leader |
| **Associate** | Solutions Architect Associate | AZ-104 Administrator | Associate Cloud Engineer |
| **Professional** | Solutions Architect Professional | AZ-305 Solutions Architect | Professional Cloud Architect |
| **DevOps** | DevOps Engineer Professional | AZ-400 DevOps Engineer | Professional Cloud DevOps |
| **Security** | Security Specialty | AZ-500 Security Engineer | Professional Cloud Security |
| **Data** | Data Analytics Specialty | DP-203 Data Engineer | Professional Data Engineer |

---

> **📝 Notes:**
> - Prices mentioned are approximate (US regions) and change frequently
> - Always check the official pricing pages for current rates
> - This document will be continuously updated as we dive deeper into each topic

---

*Last Updated: 2026-05-22*
