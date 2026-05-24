# Chapter 1: Introduction to Microsoft Azure

---

## Table of Contents

- [What is Azure?](#what-is-azure)
- [Brief History](#brief-history)
- [Why Azure? (For a Full-Stack Developer / DevOps Engineer)](#why-azure-for-a-full-stack-developer--devops-engineer)
- [Azure Global Infrastructure](#azure-global-infrastructure)
- [How Azure Services Are Organized](#how-azure-services-are-organized)
- [Azure Portal - What You See](#azure-portal---what-you-see)
- [How Azure Works in Real Companies](#how-azure-works-in-real-companies)
- [Azure Pricing Model](#azure-pricing-model)
- [Azure vs AWS Naming Comparison](#azure-vs-aws-naming-comparison)
- [Key Terminology](#key-terminology)
- [Accessing Azure (5 Ways)](#accessing-azure-5-ways)
- [Azure's Unique Concepts (vs AWS/GCP)](#azures-unique-concepts-vs-awsgcp)
- [What's Next?](#whats-next)

---

## What is Azure?

Microsoft Azure is a cloud computing platform and service created by Microsoft that provides a wide range of cloud services (compute, storage, networking, databases, AI, DevOps, etc.) integrated with Microsoft's enterprise ecosystem (Active Directory, Office 365, Windows Server, .NET).

> **In Simple Terms:** Azure is Microsoft's cloud. If your company already uses Microsoft products (Windows, Office 365, Active Directory, SQL Server, .NET), Azure integrates seamlessly. You get the same enterprise-grade infrastructure Microsoft uses for Xbox, LinkedIn, GitHub, and Office 365.

---

## Brief History

```
Timeline:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2008 → "Project Red Dog" announced at Microsoft PDC
2010 → Windows Azure officially launched (PaaS-focused)
2012 → IaaS (Virtual Machines) added, renamed to "Microsoft Azure"
2014 → Became #2 cloud provider, enterprise features expanded
2015 → Azure AD, Resource Manager (ARM) model introduced
2017 → Cosmos DB, AKS launched
2018 → Azure surpasses $30B annual run rate
2019 → Azure Arc (hybrid/multi-cloud), GitHub acquisition integration
2020 → Massive growth during pandemic, Teams/cloud adoption
2022 → Azure OpenAI Service launched
2023 → Copilot integrations, AI-first strategy
2024 → 60+ regions, deepest enterprise integration
2026 → 60+ regions, #2 cloud provider (~23% market share)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Why Azure? (For a Full-Stack Developer / DevOps Engineer)

| Need | Azure Solution |
|------|---------------|
| Host my web application | App Service, VMs, AKS, Container Apps |
| Store files/images/videos | Blob Storage |
| Run a database | Azure SQL, Cosmos DB, PostgreSQL Flexible Server |
| Deploy containers | AKS, Container Apps, ACI |
| CI/CD Pipeline | Azure DevOps Pipelines, GitHub Actions |
| Monitor everything | Azure Monitor, Application Insights, Log Analytics |
| Secure my infrastructure | Entra ID, NSGs, Key Vault, Defender for Cloud |
| Serve globally with low latency | Front Door, Azure CDN |
| Run code without servers | Azure Functions, Container Apps |
| Message queues & events | Service Bus, Event Grid, Event Hubs |

### Azure's Unique Strengths

```
What Makes Azure Different:
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  1. ENTERPRISE INTEGRATION (Microsoft Ecosystem)              │
│     └── Seamless with Active Directory, Office 365,           │
│         Windows Server, SQL Server, .NET, Visual Studio       │
│     └── If company uses Microsoft → Azure is natural choice  │
│                                                                │
│  2. HYBRID CLOUD LEADER                                       │
│     └── Azure Arc: Manage on-premises + multi-cloud from Azure│
│     └── Azure Stack: Run Azure in YOUR data center            │
│     └── Best story for gradual cloud migration               │
│                                                                │
│  3. AZURE DEVOPS (Complete Platform)                          │
│     └── Repos + Pipelines + Boards + Artifacts + Test Plans  │
│     └── All-in-one DevOps platform (not just CI/CD)          │
│                                                                │
│  4. MICROSOFT ENTRA ID (formerly Azure AD)                    │
│     └── Enterprise identity management                        │
│     └── SSO, MFA, Conditional Access, PIM                    │
│     └── Most enterprises already have it                     │
│                                                                │
│  5. AZURE OPENAI SERVICE                                      │
│     └── GPT-4, DALL-E, Whisper hosted in Azure               │
│     └── Enterprise compliance + private endpoints             │
│                                                                │
│  6. COMPLIANCE CERTIFICATIONS                                 │
│     └── Most compliance offerings of any cloud (100+)         │
│     └── Government clouds, healthcare, financial              │
│                                                                │
│  7. GITHUB INTEGRATION                                        │
│     └── Microsoft owns GitHub → tight integration             │
│     └── GitHub Actions + Azure deployment = seamless          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Azure Global Infrastructure

### Overview Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AZURE GLOBAL INFRASTRUCTURE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    GEOGRAPHIES (60+ countries)                │    │
│  │  Contain one or more regions, define data residency          │    │
│  │  Examples: United States, Europe, Asia Pacific, India        │    │
│  │                                                               │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │                   REGIONS (60+)                        │   │    │
│  │  │  A set of data centers connected by low-latency network│   │    │
│  │  │                                                        │   │    │
│  │  │  ┌─────────────────────────────────────────────────┐  │   │    │
│  │  │  │       AVAILABILITY ZONES (3 per enabled region)  │  │   │    │
│  │  │  │  Physically separate buildings with independent   │  │   │    │
│  │  │  │  power, cooling, and networking                  │  │   │    │
│  │  │  │                                                   │  │   │    │
│  │  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐        │  │   │    │
│  │  │  │  │ Zone 1  │  │ Zone 2  │  │ Zone 3  │        │  │   │    │
│  │  │  │  │ DC(s)   │  │ DC(s)   │  │ DC(s)   │        │  │   │    │
│  │  │  │  └────┬────┘  └────┬────┘  └────┬────┘        │  │   │    │
│  │  │  │       └───── < 2ms latency ──────┘             │  │   │    │
│  │  │  └─────────────────────────────────────────────────┘  │   │    │
│  │  │                                                        │   │    │
│  │  │  ┌─────────────────────────────────────────────────┐  │   │    │
│  │  │  │              REGION PAIRS                         │  │   │    │
│  │  │  │  Each region paired with another in same geography│  │   │    │
│  │  │  │  For disaster recovery & platform updates         │  │   │    │
│  │  │  │                                                   │  │   │    │
│  │  │  │  East US ←──paired──→ West US                    │  │   │    │
│  │  │  │  North Europe ←paired→ West Europe               │  │   │    │
│  │  │  │  Central India ←paired→ South India              │  │   │    │
│  │  │  └─────────────────────────────────────────────────┘  │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              POINTS OF PRESENCE (190+)                       │    │
│  │  Azure CDN, Azure Front Door                                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              SOVEREIGN CLOUDS                                │    │
│  │  ┌──────────────────┐  ┌──────────────────┐                │    │
│  │  │ Azure Government  │  │ Azure China       │                │    │
│  │  │ (US Gov only)     │  │ (Operated by      │                │    │
│  │  │ FedRAMP, DoD      │  │  21Vianet)        │                │    │
│  │  └──────────────────┘  └──────────────────┘                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              AZURE EDGE ZONES                                │    │
│  │  Azure services deployed at the edge (carrier networks)      │    │
│  │  For ultra-low latency applications (5G, gaming, media)      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              AZURE STACK (Hybrid)                             │    │
│  │  ├── Azure Stack Hub (Full Azure in your DC)                 │    │
│  │  ├── Azure Stack HCI (Hyperconverged infrastructure)         │    │
│  │  └── Azure Stack Edge (AI/ML at the edge)                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Concepts Explained

#### 1. Geography
A **Geography** is a discrete market that contains one or more Azure regions. Geographies preserve data residency and compliance boundaries.

```
Examples:
┌────────────────────────────────────────────────────────────┐
│ Geography     │ Regions                                     │
├────────────────────────────────────────────────────────────┤
│ United States │ East US, East US 2, Central US, West US... │
│ Europe        │ North Europe, West Europe, UK South...     │
│ Asia Pacific  │ East Asia, Southeast Asia, Japan East...   │
│ India         │ Central India, South India, West India     │
│ Australia     │ Australia East, Australia Southeast        │
└────────────────────────────────────────────────────────────┘
```

#### 2. Region
A **Region** is a set of data centers deployed within a latency-defined perimeter, connected through a dedicated regional low-latency network.

```
Examples of Popular Regions:
┌─────────────────────────────────────────────────────────┐
│ Region Name          │ Location                          │
├─────────────────────────────────────────────────────────┤
│ East US              │ Virginia, USA                     │ ← Common default
│ East US 2            │ Virginia, USA                     │
│ West Europe          │ Netherlands                       │
│ North Europe         │ Ireland                           │
│ Central India        │ Pune, India                       │ ← India
│ Southeast Asia       │ Singapore                         │
│ West US 2            │ Washington, USA                   │
│ UK South             │ London                            │
└─────────────────────────────────────────────────────────┘
```

**How to choose a region:**
1. **Proximity to users** — Lower latency
2. **Compliance** — Data residency requirements (geography boundary)
3. **Service availability** — Not all services in all regions
4. **Pricing** — Varies by region
5. **Region pair** — Consider DR strategy with paired region
6. **Availability Zones** — Not all regions have AZs

#### 3. Availability Zones
An **Availability Zone** is a physically separate location within a region with independent power, cooling, and networking.

```
Why AZs matter (High Availability):

    Region: Central India (Pune)
    ┌──────────────────────────────────────────────────┐
    │                                                  │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
    │  │ Zone 1   │  │ Zone 2   │  │ Zone 3   │     │
    │  │          │  │          │  │          │     │
    │  │ App VM ✓ │  │ App VM ✓ │  │ App VM ✓ │     │
    │  │ SQL (Pri)│  │ SQL (Sec)│  │          │     │
    │  └──────────┘  └──────────┘  └──────────┘     │
    │                                                  │
    │  Load Balancer distributes across all 3 zones   │
    │  If Zone 1 fails → traffic goes to Zone 2 & 3  │
    │  SQL auto-failover to Zone 2                    │
    └──────────────────────────────────────────────────┘
```

#### 4. Region Pairs (Unique to Azure)
Each Azure region is paired with another region within the same geography:

```
Region Pairs:
┌──────────────────────────────────────────────────────┐
│ Primary Region    │ Paired Region                    │
├──────────────────────────────────────────────────────┤
│ East US           │ West US                          │
│ North Europe      │ West Europe                      │
│ Central India     │ South India                      │
│ UK South          │ UK West                          │
│ Southeast Asia    │ East Asia                        │
└──────────────────────────────────────────────────────┘

Benefits of Region Pairs:
├── Platform updates are rolled out to one region at a time (not both)
├── In case of broad outage, one region is prioritized for recovery
├── GRS (Geo-Redundant Storage) replicates to paired region
└── Some services offer automatic failover to paired region
```

#### 5. Sovereign Clouds

```
┌─────────────────────────────────────────────────────────┐
│ Azure Government                                         │
│ ├── Physically isolated from commercial Azure           │
│ ├── Only US government entities and their partners      │
│ ├── Meets FedRAMP High, DoD IL4/IL5, ITAR              │
│ └── Separate portal: portal.azure.us                    │
├─────────────────────────────────────────────────────────┤
│ Azure China                                              │
│ ├── Operated by 21Vianet (not Microsoft directly)       │
│ ├── Physically and logically separated                  │
│ ├── Complies with Chinese regulations                   │
│ └── Separate portal: portal.azure.cn                    │
└─────────────────────────────────────────────────────────┘
```

---

## How Azure Services Are Organized

When you open the Azure Portal, services are accessible via the left sidebar and search:

```
Azure Service Categories (Portal Navigation):
├── Compute
│   ├── Virtual Machines, VM Scale Sets, App Service, Functions
│   ├── Container Instances, AKS, Container Apps, Batch
├── Networking
│   ├── Virtual Networks, Load Balancer, Application Gateway
│   ├── VPN Gateway, Azure DNS, CDN, Front Door, ExpressRoute
│   ├── Firewall, NSG, Traffic Manager, Private Link
├── Storage
│   ├── Storage Accounts (Blob, Files, Queue, Table)
│   ├── Managed Disks, Data Lake Storage
├── Databases
│   ├── Azure SQL, Cosmos DB, PostgreSQL, MySQL
│   ├── Cache for Redis, SQL Managed Instance
├── Identity
│   ├── Microsoft Entra ID (Azure AD), App Registrations
│   ├── Enterprise Applications, Conditional Access
├── Security
│   ├── Key Vault, Defender for Cloud, DDoS Protection
│   ├── Azure Policy, Managed Identities
├── Integration
│   ├── Service Bus, Event Grid, Event Hubs, Logic Apps
│   ├── API Management, Azure Functions
├── DevOps
│   ├── Azure DevOps (separate portal: dev.azure.com)
│   ├── Container Registry, GitHub integration
├── Management + Governance
│   ├── Azure Monitor, Log Analytics, Application Insights
│   ├── Azure Policy, Blueprints, Cost Management
│   ├── Resource Graph, Service Health, Advisor
├── Analytics
│   ├── Synapse Analytics, Data Factory, Databricks
│   ├── Stream Analytics, HDInsight, Power BI
├── AI + Machine Learning
│   ├── Azure OpenAI, Machine Learning, Cognitive Services
│   ├── Bot Service, Document Intelligence
├── Migration
│   ├── Azure Migrate, Database Migration Service
│   ├── Azure Arc, Azure Stack
└── IoT
    ├── IoT Hub, IoT Central, Digital Twins
```

---

## Azure Portal - What You See

When you log into the Azure Portal (https://portal.azure.com):

```
┌─────────────────────────────────────────────────────────────────────┐
│  [☰] Microsoft Azure  [🔍 Search resources, services, docs]   [👤] │
├──────┬──────────────────────────────────────────────────────────────┤
│      │                                                               │
│  F   │  ┌─── Azure Portal Home ─────────────────────────────────┐  │
│  A   │  │                                                        │  │
│  V   │  │  Azure services (icons)                                │  │
│  O   │  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐         │  │
│  R   │  │  │ VM │ │ App│ │SQL │ │K8s │ │Func│ │More│         │  │
│  I   │  │  │    │ │Svc │ │    │ │    │ │    │ │ >> │         │  │
│  T   │  │  └────┘ └────┘ └────┘ └────┘ └────┘ └────┘         │  │
│  E   │  │                                                        │  │
│  S   │  │  Recent resources                                      │  │
│      │  │  ├── my-web-app (App Service)                          │  │
│  B   │  │  ├── prod-sql-server (SQL Database)                    │  │
│  A   │  │  └── rg-production (Resource Group)                    │  │
│  R   │  │                                                        │  │
│      │  │  Navigate                                              │  │
│      │  │  ├── Subscriptions       ├── Resource Groups           │  │
│      │  │  ├── All resources       ├── Azure Active Directory    │  │
│      │  │  └── Dashboard           └── Cost Management           │  │
│      │  │                                                        │  │
│      │  │  Tools                                                 │  │
│      │  │  ├── Azure Monitor       ├── Azure Advisor             │  │
│      │  │  └── Microsoft Learn     └── Service Health            │  │
│      │  │                                                        │  │
│      │  └────────────────────────────────────────────────────────┘  │
│      │                                                               │
└──────┴──────────────────────────────────────────────────────────────┘
```

### Important UI Elements:

| Element | Location | Purpose |
|---------|----------|---------|
| **Search Bar** | Top-center | Search any resource, service, or documentation (VERY powerful) |
| **Directory + Subscription** | Top-right (gear icon) | Switch between AD tenants and subscriptions |
| **Cloud Shell** | Top bar (terminal icon) | Browser-based Bash/PowerShell with Azure CLI |
| **Notifications** | Bell icon | Deployment progress, alerts |
| **Favorites sidebar** | Left panel | Pin frequently used services |
| **Resource Groups** | Portal navigation | Group related resources (CRITICAL concept) |

### Critical Difference: Resource Groups in Azure

```
⚠️  IMPORTANT CONCEPT:

In Azure, EVERY resource must belong to exactly ONE Resource Group.

Resource Group properties:
├── Logical container for related resources
├── All resources in a group share the same lifecycle
├── Can contain resources from different regions
├── Deletion of RG deletes ALL resources inside
├── RBAC can be applied at RG level (inherited by all resources)
├── Tags at RG level for cost tracking
└── Cannot be nested (flat structure)

Example:
Resource Group: rg-prod-webapp
├── App Service Plan (West Europe)
├── Web App (West Europe)
├── SQL Database (West Europe)
├── Storage Account (West Europe)
├── Application Insights (West Europe)
└── Key Vault (West Europe)

This is different from:
- AWS: No mandatory grouping (resources are just in a region/account)
- GCP: Resources belong to a Project (similar concept but at higher level)
```

---

## How Azure Works in Real Companies

### Typical Company Setup

```
Real-World Azure Organization Structure:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Company: "TechCorp Inc."

Microsoft Entra ID Tenant: techcorp.onmicrosoft.com (or techcorp.com)
│
└── Management Group: Tenant Root Group
    │
    ├── Management Group: Platform
    │   ├── Subscription: Identity & Access
    │   │   └── RG: rg-identity
    │   │       └── Entra ID Premium, Conditional Access policies
    │   ├── Subscription: Management
    │   │   ├── RG: rg-monitoring
    │   │   │   └── Log Analytics Workspace, Azure Monitor
    │   │   └── RG: rg-automation
    │   │       └── Automation Account, Update Management
    │   └── Subscription: Connectivity
    │       ├── RG: rg-networking-hub
    │       │   └── Hub VNet, Azure Firewall, VPN Gateway
    │       └── RG: rg-dns
    │           └── Private DNS Zones, DNS Resolver
    │
    ├── Management Group: Landing Zones
    │   ├── Management Group: Production
    │   │   ├── Subscription: Prod-App1
    │   │   │   ├── RG: rg-prod-app1-compute
    │   │   │   │   └── AKS Cluster, Container Registry
    │   │   │   ├── RG: rg-prod-app1-data
    │   │   │   │   └── Azure SQL, Cosmos DB, Redis Cache
    │   │   │   └── RG: rg-prod-app1-web
    │   │   │       └── Front Door, App Service, CDN
    │   │   └── Subscription: Prod-App2
    │   │       └── ...
    │   └── Management Group: Non-Production
    │       ├── Subscription: Development
    │       │   └── RG: rg-dev-*
    │       └── Subscription: Staging
    │           └── RG: rg-staging-*
    │
    └── Management Group: Sandbox
        └── Subscription: Sandbox-Developers
            └── RG: rg-sandbox-*

Azure Policy applied at Management Group level → inherited by all below
```

### Why This Structure? (Azure Landing Zone Architecture)

| Component | Purpose |
|-----------|---------|
| **Tenant Root Group** | Top of hierarchy, apply org-wide policies |
| **Platform MG** | Shared infrastructure — networking, identity, monitoring |
| **Landing Zones MG** | Where actual workloads live |
| **Hub VNet** | Central networking with firewall, spoke VNets for workloads |
| **Separate Subscriptions** | Billing isolation, quota isolation, access boundary |
| **Management Groups** | Apply Azure Policy at scale across subscriptions |

### Day-to-Day Workflow (Developer/DevOps)

```
Your daily interaction with Azure:

Morning:
  1. Log in via Entra ID SSO → access portal, DevOps, GitHub
  2. Check Azure Monitor dashboard → health, performance
  3. Check Azure DevOps Pipelines → build/deploy status

Development:
  4. Push code to Azure Repos or GitHub
  5. Azure Pipeline / GitHub Action triggers automatically
  6. Pipeline: build → test → Docker build → push to ACR
  7. Deploy to staging (App Service slot / AKS namespace)

Release:
  8. PR-based approval / Environment approval gates
  9. Swap deployment slots (App Service) or Helm upgrade (AKS)
  10. Canary/blue-green via Traffic Manager or Front Door

Troubleshooting:
  11. Application Insights → Live Metrics, Transaction search
  12. Log Analytics (KQL) → query logs across all resources
  13. Azure Monitor Alerts → triggered on metric/log conditions
  14. Activity Log → "who did what" audit trail (via ARM)
```

---

## Azure Pricing Model

### Core Principles

```
Azure Pricing Philosophy:
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  1. PAY-AS-YOU-GO                                             │
│     └── No upfront, per-minute billing for most services      │
│                                                                │
│  2. RESERVED INSTANCES (1 or 3 year)                          │
│     └── Commit to 1-3 years → up to 72% discount             │
│         Applies to VMs, SQL, Cosmos DB, etc.                  │
│                                                                │
│  3. AZURE SAVINGS PLAN                                        │
│     └── Commit to $/hour spend → flexible across services     │
│         More flexible than reservations                       │
│                                                                │
│  4. SPOT VMS                                                   │
│     └── Up to 90% discount for evictable workloads            │
│                                                                │
│  5. AZURE HYBRID BENEFIT                                      │
│     └── Already have Windows Server / SQL Server licenses?    │
│         Use them on Azure → save up to 85%                   │
│         (Unique to Azure — big deal for enterprises!)         │
│                                                                │
│  6. DEV/TEST PRICING                                          │
│     └── Discounted rates for development & testing            │
│         (No Windows license charges on Dev/Test subs)         │
│                                                                │
│  7. FREE TIER                                                  │
│     ├── $200 credit for 30 days (new accounts)               │
│     └── 12-month free + always-free services                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Free Tier Highlights (Useful for Learning)

| Service | Free Tier Offer | Duration |
|---------|----------------|----------|
| Virtual Machines | 750 hrs B1s (Linux + Windows) | 12 months |
| App Service | 10 web/mobile/API apps (F1 tier) | Always free |
| Azure SQL | 250 GB S0 instance | 12 months |
| Cosmos DB | 1000 RU/s, 25 GB storage | Always free |
| Functions | 1M executions / month | Always free |
| Blob Storage | 5 GB LRS, 20K read, 10K write | 12 months |
| Azure DevOps | 5 users, unlimited private repos | Always free |
| Key Vault | 10,000 transactions / month | 12 months |
| Azure Monitor | Basic metrics (free for all resources) | Always free |
| **New Account Credit** | **$200 for 30 days** | One-time |

---

## Azure vs AWS Naming Comparison

| Azure Service | AWS Equivalent | Notes |
|---------------|---------------|-------|
| Virtual Machines | EC2 | VMs |
| Azure Functions | Lambda | Serverless functions |
| AKS | EKS | Managed Kubernetes |
| Container Apps | App Runner / Fargate | Serverless containers |
| Blob Storage | S3 | Object storage |
| Azure SQL | RDS (SQL Server) | Managed SQL |
| Cosmos DB | DynamoDB | Multi-model NoSQL (more flexible) |
| VNet | VPC | Virtual networks |
| NSG | Security Groups | Network access control |
| Azure Firewall | Network Firewall | Managed firewall |
| Application Gateway | ALB + WAF | L7 load balancer with WAF |
| Front Door | CloudFront + Global Accelerator | Global LB + CDN + WAF |
| Entra ID (Azure AD) | IAM + Cognito + SSO | Identity platform |
| Key Vault | Secrets Manager + KMS | Combined secrets + keys |
| Azure Monitor | CloudWatch | Monitoring platform |
| Application Insights | X-Ray + CloudWatch RUM | APM |
| Azure DevOps | CodePipeline + CodeBuild + CodeCommit | Full DevOps platform |
| ARM/Bicep | CloudFormation | IaC |
| Service Bus | SQS + SNS | Enterprise messaging |
| Event Grid | EventBridge | Event routing |
| Logic Apps | Step Functions | Workflow orchestration |
| API Management | API Gateway | API management |
| Azure Policy | AWS Config + SCPs | Governance |

---

## Key Terminology

| Term | Meaning |
|------|---------|
| **Tenant** | An instance of Microsoft Entra ID representing your organization |
| **Subscription** | Billing container and access boundary for Azure resources |
| **Resource Group** | Logical container for resources that share the same lifecycle |
| **Management Group** | Container for managing access/policy across multiple subscriptions |
| **ARM (Azure Resource Manager)** | Deployment and management layer for all Azure resources |
| **Resource Provider** | Service that supplies resources (e.g., `Microsoft.Compute` for VMs) |
| **RBAC** | Role-Based Access Control (Owner, Contributor, Reader, custom) |
| **Resource ID** | Unique identifier (e.g., `/subscriptions/{id}/resourceGroups/{rg}/providers/Microsoft.Compute/virtualMachines/{vm}`) |
| **SKU** | Stock Keeping Unit — defines tier/size/capabilities of a resource |
| **Entra ID** | Microsoft's identity platform (formerly Azure Active Directory) |
| **Tag** | Key-value metadata for organization, billing, governance |
| **Azure CLI** | Command-line tool (`az` command) |
| **Azure PowerShell** | PowerShell module (`Az` commands) |

---

## Accessing Azure (5 Ways)

```
┌─────────────────────────────────────────────────────────────────┐
│                  WAYS TO INTERACT WITH AZURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Azure Portal (Web Browser)                                   │
│     └── GUI → Best for learning, exploring, one-off tasks       │
│     └── URL: https://portal.azure.com                           │
│                                                                   │
│  2. Azure CLI (Command Line Interface)                           │
│     └── Cross-platform terminal commands (Bash-style)           │
│     └── Example: az vm list --output table                      │
│                                                                   │
│  3. Azure PowerShell (PowerShell Module)                         │
│     └── PowerShell-native cmdlets                               │
│     └── Example: Get-AzVM | Format-Table                       │
│     └── Preferred by Windows admins & .NET developers           │
│                                                                   │
│  4. Azure SDKs (Programmatic)                                    │
│     └── Code libraries → Application integration                │
│     └── Languages: .NET, Python, Java, JavaScript, Go           │
│                                                                   │
│  5. Azure Cloud Shell (Browser-based Terminal)                    │
│     └── Pre-authenticated Bash or PowerShell in browser         │
│     └── Includes: az, terraform, kubectl, docker, git           │
│     └── 5 GB persistent storage                                 │
│                                                                   │
│  (All methods interact with Azure Resource Manager / ARM APIs)   │
│                                                                   │
│  Bonus: Bicep / Terraform (IaC) → Infrastructure as Code        │
│  Bonus: Azure DevOps (dev.azure.com) → Separate DevOps portal  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Azure's Unique Concepts (vs AWS/GCP)

### 1. Resource Groups (Mandatory)
```
AWS/GCP: Resources exist independently (no mandatory grouping)
Azure:   EVERY resource MUST be in a Resource Group

Benefits:
├── Deploy together, manage together, delete together
├── Apply RBAC at group level
├── Track costs per resource group
├── Export ARM template from entire group
└── Apply locks (CanNotDelete, ReadOnly) at group level
```

### 2. Subscriptions as Boundaries
```
Azure Subscription ≈ AWS Account ≈ GCP Project (loosely)

But Azure is unique because:
├── One Entra ID tenant can have MULTIPLE subscriptions
├── Subscriptions define billing boundary + service limits
├── Types: Pay-As-You-Go, Enterprise Agreement, CSP, Free, Dev/Test
├── Management Groups organize subscriptions hierarchically
└── Azure Policy applied at subscription or management group level
```

### 3. Azure Resource Manager (ARM)
```
ALL interactions with Azure go through ARM:

          Portal ──┐
          CLI ─────┤
          PowerShell┼──→ ARM (Authentication + Authorization) ──→ Resource Provider
          SDK ─────┤              │                                    │
          Terraform┘              ├── Validates request               ├── Creates/updates resource
                                  ├── Checks RBAC                     └── Returns response
                                  └── Applies Azure Policy

Why this matters:
- Consistent behavior regardless of which tool you use
- Every action is logged (Activity Log)
- Idempotent deployments (deploy same template = same result)
```

### 4. Hybrid Benefit (Cost Advantage)
```
If your company already has:
├── Windows Server licenses → Use on Azure VMs (save ~40%)
├── SQL Server licenses → Use on Azure SQL (save ~55%)
└── Combined → Save up to 85% vs pay-as-you-go

This is a MASSIVE cost advantage for enterprises already invested in Microsoft.
No equivalent exists in AWS or GCP.
```

---

## What's Next?

In the next chapter, we'll set up your Azure account (free and enterprise), understand tenants, subscriptions, and management groups in depth, and learn how companies configure their Azure landing zones.

→ Next: [Chapter 2: Account Setup & Tenant Configuration](02-account-setup-and-tenant.md)

---

*Last Updated: May 2026*
