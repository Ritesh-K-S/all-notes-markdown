# Microsoft Azure - Complete Guide

> A comprehensive, hands-on guide to Azure for full-stack developers and DevOps engineers.  
> Goal: Master every aspect of Azure to independently manage cloud infrastructure for your company.

---

## How to Use This Guide

- Each chapter is a separate `.md` file linked below.
- Chapters follow a logical learning path — from account setup to advanced services.
- Each service file includes: portal walkthrough, all fields/options explained, diagrams, and real-world usage.

---

## Table of Contents

### Part 1: Foundation & Account Management

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 1 | Introduction to Azure | `01-introduction-to-azure.md` | What is Azure, global infrastructure, regions, availability zones, geographies |
| 2 | Account Setup & Tenant Configuration | `02-account-setup-and-tenant.md` | Personal vs Enterprise, Azure AD tenant, subscriptions, Microsoft Entra ID |
| 3 | Resource Hierarchy & Management | `03-resource-hierarchy.md` | Management Groups → Subscriptions → Resource Groups → Resources, tagging |
| 4 | Microsoft Entra ID (Azure AD) | `04-entra-id.md` | Users, groups, app registrations, enterprise apps, roles, conditional access, PIM |
| 5 | Billing, Cost Management & Budgets | `05-billing-and-cost-management.md` | Cost analysis, budgets, advisor recommendations, reservations, savings plans |

### Part 2: Networking

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 6 | Virtual Networks (VNet) | `06-virtual-networks.md` | VNet creation, address space, subnets, DNS settings, service endpoints |
| 7 | VNet Advanced Networking | `07-vnet-advanced.md` | VNet Peering, VPN Gateway, ExpressRoute, Virtual WAN, Private Link |
| 8 | Network Security Groups (NSG) | `08-nsg.md` | Inbound/outbound rules, priority, ASGs, flow logs, NSG diagnostics |
| 9 | Azure DNS | `09-azure-dns.md` | DNS zones (public/private), record sets, alias records, Private DNS zones |
| 10 | Azure CDN & Front Door | `10-cdn-front-door.md` | CDN profiles, endpoints, rules engine, Front Door routing, caching, WAF |
| 11 | Azure Load Balancer | `11-load-balancer.md` | Public/Internal LB, backend pools, health probes, load balancing rules, NAT rules |
| 12 | Application Gateway | `12-application-gateway.md` | SKUs, listeners, routing rules, backend pools, WAF policies, URL path-based routing |
| 13 | Azure Firewall | `13-azure-firewall.md` | Firewall policies, rule collections (NAT/Network/Application), threat intelligence |
| 14 | Traffic Manager | `14-traffic-manager.md` | Routing methods, endpoints, health checks, nested profiles |

### Part 3: Compute

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 15 | Virtual Machines | `15-virtual-machines.md` | VM sizes/series, images, disks, availability sets/zones, Spot VMs, custom images |
| 16 | VM Scale Sets (VMSS) | `16-vmss.md` | Scaling policies, upgrade policies, instance repair, orchestration modes |
| 17 | Azure Functions | `17-azure-functions.md` | Function apps, triggers/bindings, hosting plans, Durable Functions, deployment slots |
| 18 | Azure Container Instances (ACI) | `18-aci.md` | Container groups, restart policies, volumes, GPU, virtual network |
| 19 | Azure Kubernetes Service (AKS) | `19-aks.md` | Clusters, node pools, networking (kubenet/Azure CNI), add-ons, workload identity |
| 20 | App Service | `20-app-service.md` | App Service Plans, web apps, deployment slots, custom domains, SSL, scaling |
| 21 | Azure Container Apps | `21-container-apps.md` | Environments, containers, scaling rules, Dapr, revisions, traffic splitting |

### Part 4: Storage

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 22 | Storage Accounts | `22-storage-accounts.md` | Account types, performance tiers, replication (LRS/ZRS/GRS), access tiers |
| 23 | Blob Storage | `23-blob-storage.md` | Containers, blobs (block/append/page), lifecycle management, versioning, soft delete |
| 24 | Azure Files & File Sync | `24-azure-files.md` | File shares, SMB/NFS, tiers, snapshots, Azure File Sync |
| 25 | Managed Disks | `25-managed-disks.md` | Disk types (Ultra/Premium SSD/Standard SSD/Standard HDD), snapshots, encryption |
| 26 | Azure Data Lake Storage Gen2 | `26-data-lake-storage.md` | Hierarchical namespace, ACLs, data lifecycle, integration with analytics |

### Part 5: Databases

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 27 | Azure SQL Database | `27-azure-sql.md` | Single DB, elastic pools, managed instance, serverless, DTU vs vCore, geo-replication |
| 28 | Azure Database for PostgreSQL/MySQL | `28-postgresql-mysql.md` | Flexible Server, HA, read replicas, server parameters, backups |
| 29 | Cosmos DB | `29-cosmos-db.md` | APIs (NoSQL/MongoDB/Cassandra/Gremlin/Table), RUs, partitioning, consistency levels, global distribution |
| 30 | Azure Cache for Redis | `30-azure-cache-redis.md` | Tiers (Basic/Standard/Premium/Enterprise), clustering, persistence, geo-replication |
| 31 | Azure SQL Managed Instance | `31-sql-managed-instance.md` | VNet integration, instance pools, migration from on-prem, link feature |

### Part 6: CI/CD & Developer Tools

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 32 | Azure DevOps - Repos | `32-azure-devops-repos.md` | Git repos, branch policies, pull requests, code review |
| 33 | Azure DevOps - Pipelines | `33-azure-devops-pipelines.md` | YAML pipelines, stages, jobs, tasks, variable groups, environments, approvals |
| 34 | Azure DevOps - Artifacts | `34-azure-devops-artifacts.md` | Feeds, upstream sources, package types (npm/NuGet/Maven/Python) |
| 35 | Azure Container Registry (ACR) | `35-acr.md` | SKUs, repositories, tasks, geo-replication, content trust, vulnerability scanning |
| 36 | ARM Templates & Bicep | `36-arm-bicep.md` | Template structure, parameters, modules, what-if, deployment stacks |
| 37 | Terraform on Azure | `37-terraform-azure.md` | AzureRM provider, state management, modules, CI/CD integration |

### Part 7: Monitoring & Operations

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 38 | Azure Monitor | `38-azure-monitor.md` | Metrics, logs, alerts, action groups, diagnostic settings, autoscale |
| 39 | Log Analytics & KQL | `39-log-analytics.md` | Workspaces, data collection rules, KQL queries, workbooks |
| 40 | Application Insights | `40-application-insights.md` | Instrumentation, live metrics, transaction diagnostics, availability tests, smart detection |
| 41 | Azure Service Health & Advisor | `41-service-health-advisor.md` | Service issues, planned maintenance, health alerts, Advisor recommendations |

### Part 8: Security & Compliance

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 42 | Azure Key Vault | `42-key-vault.md` | Secrets, keys, certificates, access policies vs RBAC, soft delete, purge protection |
| 43 | Microsoft Defender for Cloud | `43-defender-for-cloud.md` | Security posture, recommendations, regulatory compliance, workload protection |
| 44 | Azure Policy | `44-azure-policy.md` | Policy definitions, initiatives, assignments, compliance, remediation tasks |
| 45 | Azure DDoS Protection | `45-ddos-protection.md` | DDoS Network Protection, IP protection, telemetry, rapid response |
| 46 | Azure Private Link & Private Endpoints | `46-private-link.md` | Private endpoints, private link services, DNS integration, approval workflow |
| 47 | Managed Identities | `47-managed-identities.md` | System-assigned vs user-assigned, token acquisition, role assignments |

### Part 9: Application Integration & Messaging

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 48 | Azure Service Bus | `48-service-bus.md` | Queues, topics/subscriptions, sessions, dead-letter queue, message processing |
| 49 | Azure Event Grid | `49-event-grid.md` | Topics, event subscriptions, system/custom topics, domains, dead-lettering |
| 50 | Azure Event Hubs | `50-event-hubs.md` | Namespaces, event hubs, consumer groups, partitions, capture, Kafka interface |
| 51 | Azure Queue Storage | `51-queue-storage.md` | Queues, messages, visibility timeout, poison messages |
| 52 | Azure Logic Apps | `52-logic-apps.md` | Workflows, triggers, actions, connectors, Standard vs Consumption |
| 53 | API Management (APIM) | `53-api-management.md` | Services, APIs, operations, policies, products, subscriptions, developer portal |

### Part 10: Containers & Orchestration (Advanced)

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 54 | AKS Deep Dive | `54-aks-deep-dive.md` | Network policies, AGIC, KEDA, pod identity, GitOps with Flux |
| 55 | Azure Service Fabric | `55-service-fabric.md` | Clusters, applications, services (stateful/stateless), reliable collections |

### Part 11: Analytics & Big Data

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 56 | Azure Synapse Analytics | `56-synapse-analytics.md` | Workspaces, SQL pools (dedicated/serverless), Spark pools, pipelines |
| 57 | Azure Data Factory | `57-data-factory.md` | Pipelines, activities, datasets, linked services, triggers, data flows |
| 58 | Azure Databricks | `58-databricks.md` | Workspaces, clusters, notebooks, jobs, Unity Catalog, Delta Lake |
| 59 | Azure Stream Analytics | `59-stream-analytics.md` | Jobs, inputs, outputs, query language, windowing functions |

### Part 12: Machine Learning & AI

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 60 | Azure Machine Learning | `60-azure-ml.md` | Workspaces, compute, datasets, experiments, models, endpoints, pipelines |
| 61 | Azure AI Services | `61-ai-services.md` | OpenAI Service, Cognitive Services (Vision, Speech, Language), Bot Service |

### Part 13: Migration & Hybrid

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 62 | Azure Migrate | `62-azure-migrate.md` | Assessment, server migration, database migration, app migration |
| 63 | Azure Arc | `63-azure-arc.md` | Arc-enabled servers, Kubernetes, data services, multi-cloud management |
| 64 | Azure Stack & Hybrid Solutions | `64-azure-stack-hybrid.md` | Azure Stack Hub/HCI/Edge, hybrid connectivity patterns |

### Part 14: Architecture & Best Practices

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 65 | Azure Well-Architected Framework | `65-well-architected.md` | 5 pillars, design principles, assessment, trade-offs |
| 66 | Real-World Architecture Patterns | `66-real-world-patterns.md` | N-tier, microservices on AKS, serverless, event-driven, CQRS |
| 67 | Cost Optimization Strategies | `67-cost-optimization.md` | Reservations, Spot VMs, auto-shutdown, right-sizing, hybrid benefit |
| 68 | Disaster Recovery & BCDR | `68-disaster-recovery.md` | Azure Site Recovery, backup, geo-redundancy, RPO/RTO planning |

---

## Quick Reference

### Azure Global Infrastructure
```
Azure Global Infrastructure
├── Geographies (60+ countries)
│   ├── Regions (60+)
│   │   ├── Availability Zones (3+ per zone-enabled region)
│   │   │   └── Data Centers (1+ per AZ)
│   │   └── Region Pairs (for disaster recovery)
│   └── Sovereign Clouds (Government, China)
├── Edge Zones
│   └── Azure Edge Zones with carrier
└── Points of Presence (190+)
    └── Azure CDN / Front Door
```

### Azure Resource Hierarchy
```
Azure Active Directory Tenant (Microsoft Entra ID)
└── Management Group: Root (Tenant Root Group)
    ├── Management Group: Production
    │   ├── Subscription: Prod-Core
    │   │   ├── Resource Group: rg-prod-networking
    │   │   │   ├── Virtual Network
    │   │   │   ├── Network Security Groups
    │   │   │   └── Azure Firewall
    │   │   ├── Resource Group: rg-prod-compute
    │   │   │   ├── Virtual Machines / AKS Cluster
    │   │   │   └── App Service Plans
    │   │   └── Resource Group: rg-prod-data
    │   │       ├── Azure SQL Database
    │   │       ├── Storage Accounts
    │   │       └── Cosmos DB
    │   └── Subscription: Prod-Analytics
    │       └── Resource Group: rg-analytics
    ├── Management Group: Non-Production
    │   ├── Subscription: Development
    │   │   ├── Resource Group: rg-dev-app
    │   │   └── Resource Group: rg-dev-infra
    │   └── Subscription: Staging
    │       └── Resource Group: rg-staging
    ├── Management Group: Security
    │   └── Subscription: Security-Monitoring
    │       ├── Resource Group: rg-security-logs
    │       └── Resource Group: rg-security-tools
    └── Management Group: Sandbox
        └── Subscription: Developer-Sandbox

RBAC & Azure Policy inherited: Management Group → Subscription → Resource Group → Resource
```

---

## Learning Path Recommendation

1. **Weeks 1-2**: Chapters 1-5 (Foundation)
2. **Weeks 3-4**: Chapters 6-14 (Networking)
3. **Weeks 5-6**: Chapters 15-21 (Compute)
4. **Weeks 7-8**: Chapters 22-26 (Storage)
5. **Weeks 9-10**: Chapters 27-31 (Databases)
6. **Weeks 11-12**: Chapters 32-37 (CI/CD)
7. **Weeks 13-14**: Chapters 38-47 (Monitoring & Security)
8. **Weeks 15-16**: Chapters 48-55 (Integration & Containers)
9. **Weeks 17-18**: Chapters 56-68 (Analytics, ML, Architecture)

---

*Last Updated: May 2026*
