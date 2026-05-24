# Google Cloud Platform (GCP) - Complete Guide

> A comprehensive, hands-on guide to GCP for full-stack developers and DevOps engineers.  
> Goal: Master every aspect of GCP to independently manage cloud infrastructure for your company.

---

## How to Use This Guide

- Each chapter is a separate `.md` file linked below.
- Chapters follow a logical learning path — from account setup to advanced services.
- Each service file includes: console walkthrough, all fields/options explained, diagrams, and real-world usage.

---

## Table of Contents

### Part 1: Foundation & Account Management

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 1 | Introduction to GCP | `01-introduction-to-gcp.md` | What is GCP, global infrastructure, regions, zones, points of presence |
| 2 | Account Setup & Organization | `02-account-setup-and-organization.md` | Personal vs Organization accounts, Google Workspace integration, Cloud Identity |
| 3 | Resource Hierarchy & Projects | `03-resource-hierarchy.md` | Organization → Folders → Projects → Resources, project creation, labels |
| 4 | IAM - Identity & Access Management | `04-iam.md` | Members, roles (basic/predefined/custom), policies, conditions, service accounts, Workload Identity |
| 5 | Billing & Cost Management | `05-billing-and-cost-management.md` | Billing accounts, budgets, alerts, committed use discounts, cost breakdown, export |

### Part 2: Networking

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 6 | VPC Networks | `06-vpc-networks.md` | VPC creation, subnets (auto/custom mode), IP ranges, firewall rules, routes |
| 7 | VPC Advanced Networking | `07-vpc-advanced.md` | VPC Peering, Shared VPC, Private Google Access, Cloud NAT, Cloud Router |
| 8 | Firewall Rules & Policies | `08-firewall-rules.md` | Ingress/egress rules, priority, tags, service accounts, hierarchical firewall policies |
| 9 | Cloud DNS | `09-cloud-dns.md` | Managed zones (public/private), record sets, DNSSEC, DNS policies, peering |
| 10 | Cloud CDN | `10-cloud-cdn.md` | Backend configuration, cache modes, cache keys, signed URLs, cache invalidation |
| 11 | Cloud Load Balancing | `11-cloud-load-balancing.md` | HTTP(S) LB, TCP/UDP LB, Internal LB, backend services, health checks, URL maps |
| 12 | Cloud Armor | `12-cloud-armor.md` | Security policies, rules, WAF rules, adaptive protection, rate limiting |
| 13 | Cloud Interconnect & VPN | `13-interconnect-vpn.md` | Dedicated/Partner Interconnect, Cloud VPN (HA/Classic), Cloud Router |

### Part 3: Compute

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 14 | Compute Engine | `14-compute-engine.md` | VM instances, machine types, images, disks, preemptible/spot VMs, sole-tenant nodes |
| 15 | Instance Groups & Autoscaling | `15-instance-groups-autoscaling.md` | Managed/unmanaged instance groups, instance templates, autoscaling policies |
| 16 | Cloud Functions | `16-cloud-functions.md` | Gen 1 vs Gen 2, triggers, runtimes, environment variables, secrets, concurrency |
| 17 | Cloud Run | `17-cloud-run.md` | Services, revisions, traffic splitting, concurrency, min/max instances, jobs |
| 18 | GKE - Google Kubernetes Engine | `18-gke.md` | Cluster modes (Autopilot/Standard), node pools, workloads, services, ingress |
| 19 | App Engine | `19-app-engine.md` | Standard vs Flexible, app.yaml, versions, traffic splitting, services |

### Part 4: Storage

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 20 | Cloud Storage (GCS) | `20-cloud-storage.md` | Buckets, objects, storage classes, lifecycle rules, versioning, retention policies |
| 21 | Cloud Storage Advanced | `21-cloud-storage-advanced.md` | IAM vs ACLs, signed URLs, notifications, transfer service, compose objects |
| 22 | Persistent Disk & Local SSD | `22-persistent-disk.md` | Disk types (pd-standard, pd-balanced, pd-ssd, pd-extreme), snapshots, images |
| 23 | Filestore | `23-filestore.md` | Instances, tiers (Basic/Enterprise), capacity, access configuration, backups |

### Part 5: Databases

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 24 | Cloud SQL | `24-cloud-sql.md` | MySQL/PostgreSQL/SQL Server, instance configuration, HA, read replicas, backups |
| 25 | Cloud Spanner | `25-cloud-spanner.md` | Instances, databases, schema design, interleaving, multi-region |
| 26 | Firestore & Datastore | `26-firestore.md` | Native vs Datastore mode, collections, documents, indexes, security rules |
| 27 | Bigtable | `27-bigtable.md` | Instances, clusters, tables, column families, row key design, replication |
| 28 | Memorystore | `28-memorystore.md` | Redis vs Memcached, instances, tiers, configuration, maintenance |
| 29 | AlloyDB | `29-alloydb.md` | Clusters, instances (primary/read pool), columnar engine, cross-region replication |

### Part 6: CI/CD & Developer Tools

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 30 | Cloud Source Repositories | `30-cloud-source-repos.md` | Repositories, mirroring from GitHub/Bitbucket |
| 31 | Cloud Build | `31-cloud-build.md` | Build triggers, cloudbuild.yaml, substitutions, builders, artifacts |
| 32 | Artifact Registry | `32-artifact-registry.md` | Repositories (Docker, npm, Maven, Python), cleanup policies, vulnerability scanning |
| 33 | Cloud Deploy | `33-cloud-deploy.md` | Delivery pipelines, targets, releases, rollouts, approval gates |
| 34 | Deployment Manager & Terraform | `34-deployment-manager.md` | Templates, configurations, deployments, Terraform on GCP |
| 35 | Infrastructure Manager | `35-infrastructure-manager.md` | Terraform-based deployments, previews, revisions |

### Part 7: Monitoring & Operations

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 36 | Cloud Monitoring | `36-cloud-monitoring.md` | Metrics, dashboards, alerting policies, uptime checks, groups, custom metrics |
| 37 | Cloud Logging | `37-cloud-logging.md` | Log Explorer, log router, sinks, log-based metrics, exclusion filters |
| 38 | Cloud Trace | `38-cloud-trace.md` | Trace list, latency analysis, auto-instrumentation |
| 39 | Error Reporting & Profiler | `39-error-reporting-profiler.md` | Error grouping, notifications, continuous profiling |

### Part 8: Security & Compliance

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 40 | Cloud KMS | `40-cloud-kms.md` | Key rings, keys, key versions, rotation, encryption/decryption, CMEK/CSEK |
| 41 | Secret Manager | `41-secret-manager.md` | Secrets, versions, rotation, access control, replication policies |
| 42 | Security Command Center | `42-security-command-center.md` | Sources, findings, assets, security marks, standard/premium tier |
| 43 | VPC Service Controls | `43-vpc-service-controls.md` | Service perimeters, access levels, bridges, dry-run mode |
| 44 | Certificate Manager | `44-certificate-manager.md` | Certificates (Google-managed/self-managed), certificate maps, DNS authorization |
| 45 | Identity-Aware Proxy (IAP) | `45-identity-aware-proxy.md` | Securing apps, tunnel resources, access levels, programmatic access |

### Part 9: Application Integration & Messaging

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 46 | Pub/Sub | `46-pubsub.md` | Topics, subscriptions (pull/push), dead-letter topics, ordering, filtering, schemas |
| 47 | Cloud Tasks | `47-cloud-tasks.md` | Queues, tasks, HTTP/App Engine targets, rate limits, retry config |
| 48 | Cloud Scheduler | `48-cloud-scheduler.md` | Jobs, cron syntax, targets (HTTP/Pub/Sub/App Engine), retry policies |
| 49 | Workflows | `49-workflows.md` | Workflow definitions, steps, connectors, error handling, callbacks |
| 50 | API Gateway & Apigee | `50-api-gateway-apigee.md` | API configs, gateways, Apigee for enterprise API management |
| 51 | Eventarc | `51-eventarc.md` | Triggers, providers, event types, channels, routing |

### Part 10: Containers (Advanced)

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 52 | GKE Deep Dive | `52-gke-deep-dive.md` | Network policies, workload identity, config connector, multi-cluster |
| 53 | Anthos | `53-anthos.md` | Multi-cloud/hybrid, service mesh, config management, Anthos clusters |

### Part 11: Analytics & Big Data

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 54 | BigQuery | `54-bigquery.md` | Datasets, tables, jobs, partitioning, clustering, materialized views, BI Engine |
| 55 | Dataflow | `55-dataflow.md` | Pipelines, templates, Apache Beam, streaming vs batch |
| 56 | Dataproc | `56-dataproc.md` | Clusters, jobs (Spark/Hadoop/Hive), autoscaling, serverless |
| 57 | Composer (Airflow) | `57-cloud-composer.md` | Environments, DAGs, variables, connections, Airflow UI |

### Part 12: Machine Learning & AI

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 58 | Vertex AI | `58-vertex-ai.md` | Datasets, training, models, endpoints, pipelines, experiments |
| 59 | AI/ML APIs | `59-ai-ml-apis.md` | Vision, Speech, NLP, Translation, Generative AI (Gemini), Document AI |

### Part 13: Migration & Transfer

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 60 | Migration Strategies | `60-migration-strategies.md` | Migrate for Compute Engine, Database Migration Service, Transfer Appliance |
| 61 | Storage Transfer & BigQuery Transfer | `61-data-transfer.md` | Storage Transfer Service, BigQuery Data Transfer, Transfer Appliance |

### Part 14: Architecture & Best Practices

| # | Chapter | File | Description |
|---|---------|------|-------------|
| 62 | Google Cloud Architecture Framework | `62-architecture-framework.md` | Design principles, operational excellence, security, reliability, performance, cost |
| 63 | Real-World Architecture Patterns | `63-real-world-patterns.md` | Three-tier, microservices on GKE, serverless event-driven, data analytics pipelines |
| 64 | Cost Optimization Strategies | `64-cost-optimization.md` | Committed use, sustained use discounts, preemptible VMs, right-sizing recommendations |
| 65 | Disaster Recovery on GCP | `65-disaster-recovery.md` | Cold, warm, hot standby patterns, multi-region strategies |

---

## Quick Reference

### GCP Global Infrastructure
```
GCP Global Infrastructure
├── Regions (40+)
│   ├── Zones (3+ per region, e.g., us-central1-a, us-central1-b)
│   │   └── Physical clusters of resources
│   └── Multi-Region (US, EU, Asia)
├── Points of Presence (180+)
│   └── Edge caching for Cloud CDN
└── Private Network (Google's global fiber network)
```

### GCP Resource Hierarchy
```
Organization (company-domain.com)
├── Folder: Production
│   ├── Project: prod-frontend (project-id: prod-frontend-12345)
│   │   ├── Compute Engine instances
│   │   ├── Cloud Run services
│   │   └── Cloud Storage buckets
│   ├── Project: prod-backend (project-id: prod-backend-67890)
│   │   ├── GKE clusters
│   │   ├── Cloud SQL instances
│   │   └── Pub/Sub topics
│   └── Project: prod-data (project-id: prod-data-11111)
│       ├── BigQuery datasets
│       └── Dataflow pipelines
├── Folder: Development
│   ├── Project: dev-app
│   └── Project: staging-app
├── Folder: Shared Services
│   ├── Project: shared-networking (Shared VPC Host)
│   ├── Project: shared-monitoring
│   └── Project: shared-security
└── Folder: Sandbox
    └── Project: developer-sandbox
    
IAM Policies inherited: Organization → Folder → Project → Resource
```

---

## Learning Path Recommendation

1. **Weeks 1-2**: Chapters 1-5 (Foundation)
2. **Weeks 3-4**: Chapters 6-13 (Networking)
3. **Weeks 5-6**: Chapters 14-19 (Compute)
4. **Weeks 7-8**: Chapters 20-23 (Storage)
5. **Weeks 9-10**: Chapters 24-29 (Databases)
6. **Weeks 11-12**: Chapters 30-35 (CI/CD)
7. **Weeks 13-14**: Chapters 36-45 (Monitoring & Security)
8. **Weeks 15-16**: Chapters 46-53 (Integration & Containers)
9. **Weeks 17-18**: Chapters 54-65 (Analytics, ML, Architecture)

---

*Last Updated: May 2026*
