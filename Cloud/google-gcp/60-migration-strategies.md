# Chapter 60 — Migration Strategies

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Migration Fundamentals](#part-1--migration-fundamentals)
- [Part 2: Migration Phases (Assess → Plan → Migrate → Optimize)](#part-2--migration-phases)
- [Part 3: Lift and Shift (Rehost)](#part-3--lift-and-shift-rehost)
- [Part 4: Migrate for Compute Engine](#part-4--migrate-for-compute-engine)
- [Part 5: Database Migration Service (DMS)](#part-5--database-migration-service-dms)
- [Part 6: Migrating to Cloud SQL](#part-6--migrating-to-cloud-sql)
- [Part 7: Migrating to Cloud Spanner & AlloyDB](#part-7--migrating-to-cloud-spanner--alloydb)
- [Part 8: Migrating to GKE (Containerization)](#part-8--migrating-to-gke-containerization)
- [Part 9: Migrate to Containers (Migrate for Anthos)](#part-9--migrate-to-containers)
- [Part 10: Application Modernization (Re-platform / Re-architect)](#part-10--application-modernization)
- [Part 11: VMware Engine Migration](#part-11--vmware-engine-migration)
- [Part 12: Transfer Appliance](#part-12--transfer-appliance)
- [Part 13: Network & DNS Migration](#part-13--network--dns-migration)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [Console Walkthrough: Database Migration Service](#console-walkthrough-database-migration-service)
- [What's Next?](#whats-next)

---

## Overview

Migrating workloads to Google Cloud requires a structured approach — assessing existing systems, planning target architectures, executing the migration with minimal downtime, and optimizing post-migration. GCP provides purpose-built tools for each migration type: VM-to-VM (Migrate for Compute Engine), database-to-managed-database (Database Migration Service), VM-to-container (Migrate to Containers), and bulk data transfer (Transfer Appliance). This chapter covers strategies, tools, and proven patterns for every migration scenario.

---

## Part 1 — Migration Fundamentals

### The 6 Rs of Migration

```
┌────────────────────────────────────────────────────────────────────┐
│         THE 6 Rs OF CLOUD MIGRATION                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐  Move as-is to VMs                              │
│  │  1. Rehost   │  "Lift and Shift"                               │
│  │              │  Fastest, lowest risk                           │
│  └──────────────┘                                                   │
│                                                                      │
│  ┌──────────────┐  Move with minor changes                        │
│  │  2. Replatform│  "Lift, Tinker, and Shift"                    │
│  │              │  e.g., MySQL → Cloud SQL                       │
│  └──────────────┘                                                   │
│                                                                      │
│  ┌──────────────┐  Buy a SaaS equivalent                         │
│  │  3. Repurchase│  e.g., on-prem CRM → Salesforce              │
│  │              │  or email server → Google Workspace             │
│  └──────────────┘                                                   │
│                                                                      │
│  ┌──────────────┐  Re-design for cloud-native                    │
│  │  4. Refactor │  "Re-architect"                                │
│  │              │  Microservices, serverless, containers          │
│  └──────────────┘                                                   │
│                                                                      │
│  ┌──────────────┐  Keep on-premises (not worth migrating)        │
│  │  5. Retain   │  Compliance, latency, or cost reasons          │
│  │              │  May connect via hybrid networking              │
│  └──────────────┘                                                   │
│                                                                      │
│  ┌──────────────┐  Decommission the workload                     │
│  │  6. Retire   │  No longer needed                              │
│  │              │  Save costs by shutting down                    │
│  └──────────────┘                                                   │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP | AWS | Azure |
|---------|-----|-----|-------|
| VM migration | Migrate for Compute Engine | Application Migration Service | Azure Migrate |
| DB migration | Database Migration Service | DMS | Azure DMS |
| Container migration | Migrate to Containers | App2Container | Azure Migrate (containers) |
| VMware migration | Google Cloud VMware Engine | VMware Cloud on AWS | Azure VMware Solution |
| Physical appliance | Transfer Appliance | Snowball / Snowmobile | Data Box |
| Assessment tool | Migration Center | Migration Hub | Azure Migrate Assessment |
| Modernization | Application Modernization | Modernization Accelerator | App Service Migration |

---

## Part 2 — Migration Phases

### Assess → Plan → Migrate → Optimize

```
┌────────────────────────────────────────────────────────────────────┐
│         MIGRATION LIFECYCLE                                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Phase 1: ASSESS                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Inventory all workloads (VMs, DBs, apps, storage)     │     │
│  │  • Map dependencies (network, data, services)            │     │
│  │  • Assess complexity & migration difficulty              │     │
│  │  • Estimate TCO (on-prem vs GCP)                        │     │
│  │  • Tool: Migration Center (StratoZone)                  │     │
│  └──────────────────────────────────────────────────────────┘     │
│       │                                                             │
│       ▼                                                             │
│  Phase 2: PLAN                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Choose migration strategy per workload (6 Rs)         │     │
│  │  • Define target architecture                            │     │
│  │  • Set up landing zone (VPC, IAM, billing)              │     │
│  │  • Create migration waves (group workloads)             │     │
│  │  • Define success criteria & rollback plan              │     │
│  └──────────────────────────────────────────────────────────┘     │
│       │                                                             │
│       ▼                                                             │
│  Phase 3: MIGRATE                                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Provision GCP infrastructure (Terraform)              │     │
│  │  • Execute data migration (continuous replication)       │     │
│  │  • Execute compute migration (VM / container)           │     │
│  │  • Test & validate (functional, performance, security)  │     │
│  │  • Cutover (DNS switch, traffic routing)                │     │
│  └──────────────────────────────────────────────────────────┘     │
│       │                                                             │
│       ▼                                                             │
│  Phase 4: OPTIMIZE                                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Right-size VMs (Recommender)                          │     │
│  │  • Enable autoscaling                                    │     │
│  │  • Purchase committed use discounts                     │     │
│  │  • Migrate to managed services (Cloud SQL, GKE)         │     │
│  │  • Set up monitoring, alerting, logging                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Migration Center (Assessment Tool)

```bash
# Migration Center — discover and assess on-prem workloads
# Install discovery client on on-prem network
# Collects: CPU, memory, disk, network, dependencies

# Create assessment in console:
# Console → Migration Center → Create Assessment
# Select workload group → Choose target GCP region
# Get: recommended VM sizes, cost estimate, migration complexity
```

---

## Part 3 — Lift and Shift (Rehost)

### When to Use

```
┌────────────────────────────────────────────────────────────────────┐
│         LIFT AND SHIFT DECISION                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  GOOD FIT:                                                          │
│  ✓ Data center lease expiring (time pressure)                     │
│  ✓ Hardware end-of-life                                            │
│  ✓ Legacy apps that can't be easily refactored                   │
│  ✓ Regulatory requirement to move to specific region             │
│  ✓ Quick wins to demonstrate cloud value                         │
│                                                                      │
│  NOT IDEAL:                                                         │
│  ✗ App has known scalability issues                              │
│  ✗ App is tightly coupled to hardware (licensing, HSM)           │
│  ✗ App will be retired within 12 months                          │
│  ✗ Cloud-native alternative exists and budget allows refactor    │
│                                                                      │
│  Cost:  On-prem servers → Compute Engine VMs                     │
│  Time:  Fastest (weeks to months)                                 │
│  Risk:  Lowest (minimal changes to app)                          │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 4 — Migrate for Compute Engine

### VM Migration (M4CE)

```
┌────────────────────────────────────────────────────────────────────┐
│         MIGRATE FOR COMPUTE ENGINE                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Source:                                                            │
│  ┌──────────────────┐                                              │
│  │ On-prem / AWS /  │                                              │
│  │ Azure VMs        │                                              │
│  │ ┌──────┐ ┌─────┐│                                              │
│  │ │ VM 1 │ │VM 2 ││                                              │
│  │ └──────┘ └─────┘│                                              │
│  └────────┬─────────┘                                              │
│           │                                                         │
│           │  Continuous replication                                 │
│           │  (block-level, low bandwidth)                          │
│           ▼                                                         │
│  ┌──────────────────┐                                              │
│  │ Migrate for CE   │                                              │
│  │ Manager          │                                              │
│  │ (orchestrates)   │                                              │
│  └────────┬─────────┘                                              │
│           │                                                         │
│           ▼                                                         │
│  Target:                                                            │
│  ┌──────────────────┐                                              │
│  │ Compute Engine   │                                              │
│  │ ┌──────┐ ┌─────┐│                                              │
│  │ │ VM 1 │ │VM 2 ││                                              │
│  │ └──────┘ └─────┘│                                              │
│  └──────────────────┘                                              │
│                                                                      │
│  Features:                                                          │
│  • Continuous replication (minimal downtime cutover)              │
│  • Test clones (validate before cutover)                          │
│  • Supports Windows & Linux                                       │
│  • Source: VMware, AWS, Azure, physical servers                   │
│  • Incremental sync (only changed blocks)                        │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Migration Steps

```bash
# Step 1: Set up Migrate for Compute Engine
# Console → Migrate to Virtual Machines → Setup

# Step 2: Add source (on-prem vCenter or AWS/Azure)
# Install M4CE connector appliance on vCenter
# Or configure AWS/Azure source credentials

# Step 3: Create migration — select VMs to migrate
gcloud migration vms sources create my-source \
    --location=us-central1 \
    --type=vmware \
    --vmware-source-vcenter-ip=10.0.0.5

# Step 4: Start replication
gcloud migration vms migrating-vms create my-vm \
    --source=my-source \
    --location=us-central1 \
    --source-vm-id=vm-12345

gcloud migration vms migrating-vms start-migration my-vm \
    --location=us-central1

# Step 5: Create test clone (validate in GCP)
gcloud migration vms clone-jobs create test-clone \
    --migrating-vm=my-vm \
    --location=us-central1

# Step 6: Cutover (final migration)
gcloud migration vms cutover-jobs create cutover-1 \
    --migrating-vm=my-vm \
    --location=us-central1
```

---

## Part 5 — Database Migration Service (DMS)

### Managed Database Migration

```
┌────────────────────────────────────────────────────────────────────┐
│         DATABASE MIGRATION SERVICE                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Source                        Target                     │     │
│  │  ┌─────────────────┐         ┌──────────────────┐       │     │
│  │  │ MySQL (on-prem) │ ──────► │ Cloud SQL MySQL  │       │     │
│  │  │ PostgreSQL      │ ──────► │ Cloud SQL PG     │       │     │
│  │  │ SQL Server      │ ──────► │ Cloud SQL MSSQL  │       │     │
│  │  │ Oracle          │ ──────► │ Cloud SQL PG     │       │     │
│  │  │ MySQL           │ ──────► │ AlloyDB          │       │     │
│  │  │ PostgreSQL      │ ──────► │ AlloyDB          │       │     │
│  │  │ MySQL/PG        │ ──────► │ Cloud Spanner    │       │     │
│  │  │ AWS RDS         │ ──────► │ Cloud SQL        │       │     │
│  │  │ Azure SQL       │ ──────► │ Cloud SQL        │       │     │
│  │  └─────────────────┘         └──────────────────┘       │     │
│  │                                                            │     │
│  │  How it works:                                            │     │
│  │  1. Full dump (initial data load)                        │     │
│  │  2. Continuous CDC replication (Change Data Capture)     │     │
│  │  3. Test the target database                             │     │
│  │  4. Promote target (cutover — seconds of downtime)      │     │
│  │                                                            │     │
│  │  Migration types:                                         │     │
│  │  • One-time: full dump + restore                         │     │
│  │  • Continuous: ongoing replication until cutover         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Pricing: FREE (no charge for DMS itself)                         │
│  You pay only for the target database instance                    │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Create a connection profile (source)
gcloud database-migration connection-profiles create source-mysql \
    --region=us-central1 \
    --type=MYSQL \
    --host=10.0.0.50 \
    --port=3306 \
    --username=migration_user \
    --password=SECRET

# Create a connection profile (target — Cloud SQL)
gcloud database-migration connection-profiles create target-cloudsql \
    --region=us-central1 \
    --type=CLOUDSQL \
    --cloudsql-instance=my-project:us-central1:target-instance

# Create migration job
gcloud database-migration migration-jobs create migrate-prod-db \
    --region=us-central1 \
    --type=CONTINUOUS \
    --source=source-mysql \
    --destination=target-cloudsql

# Start migration
gcloud database-migration migration-jobs start migrate-prod-db \
    --region=us-central1

# Verify migration job status
gcloud database-migration migration-jobs describe migrate-prod-db \
    --region=us-central1

# Promote (cutover — make target the primary)
gcloud database-migration migration-jobs promote migrate-prod-db \
    --region=us-central1
```

---

## Part 6 — Migrating to Cloud SQL

### MySQL / PostgreSQL / SQL Server

```
┌────────────────────────────────────────────────────────────────────┐
│         CLOUD SQL MIGRATION OPTIONS                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Option 1: Database Migration Service (recommended)               │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Zero-downtime continuous replication                   │     │
│  │  • Managed service — minimal manual work                 │     │
│  │  • Supports MySQL, PostgreSQL, SQL Server                │     │
│  │  • Free to use (pay only for target instance)           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Option 2: Native dump/restore                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  MySQL:  mysqldump → GCS → import to Cloud SQL          │     │
│  │  PG:     pg_dump → GCS → import to Cloud SQL            │     │
│  │  MSSQL:  .bak file → GCS → import to Cloud SQL         │     │
│  │  Downtime: proportional to database size                │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Option 3: External replica + promote                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Configure Cloud SQL as read replica of on-prem       │     │
│  │  • Let it catch up via binlog/WAL replication           │     │
│  │  • Promote Cloud SQL replica to primary                 │     │
│  │  • Minimal downtime at cutover                          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Option 2: mysqldump approach
# On source server:
mysqldump --databases mydb \
    --hex-blob --skip-triggers --single-transaction \
    --default-character-set=utf8mb4 \
    --set-gtid-purged=ON \
    > mydb_dump.sql

# Upload to GCS
gsutil cp mydb_dump.sql gs://my-migration-bucket/

# Import into Cloud SQL
gcloud sql import sql my-cloudsql-instance \
    gs://my-migration-bucket/mydb_dump.sql \
    --database=mydb

# Option 2: pg_dump approach
pg_dump -h source-host -U postgres -Fc mydb > mydb.dump
gsutil cp mydb.dump gs://my-migration-bucket/
gcloud sql import sql my-cloudsql-instance \
    gs://my-migration-bucket/mydb.dump \
    --database=mydb
```

---

## Part 7 — Migrating to Cloud Spanner & AlloyDB

### Spanner Migration Tool (SMT)

```
┌────────────────────────────────────────────────────────────────────┐
│         SPANNER MIGRATION                                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Spanner Migration Tool (Harbourbridge):                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Source               Process              Target        │     │
│  │  ┌─────────┐        ┌─────────────┐      ┌──────────┐  │     │
│  │  │ MySQL   │  ───►  │ Schema      │ ───► │ Cloud    │  │     │
│  │  │ PG      │        │ conversion  │      │ Spanner  │  │     │
│  │  │ Oracle  │        │ + data load │      │          │  │     │
│  │  │ SQL Svr │        │             │      │          │  │     │
│  │  └─────────┘        │ ┌─────────┐ │      └──────────┘  │     │
│  │                      │ │  Web UI │ │                     │     │
│  │                      │ │ review  │ │                     │     │
│  │                      │ │ schema  │ │                     │     │
│  │                      │ └─────────┘ │                     │     │
│  │                      └─────────────┘                     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Schema changes required:                                          │
│  • Auto-increment → UUID-based keys                              │
│  • Foreign keys → interleaved tables (for performance)           │
│  • Data types mapped (MySQL INT → Spanner INT64)                 │
│  • No stored procedures (rewrite as app logic)                   │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### AlloyDB Migration

```bash
# AlloyDB migration uses DMS (same as Cloud SQL migration)
gcloud database-migration connection-profiles create target-alloydb \
    --region=us-central1 \
    --type=ALLOYDB \
    --alloydb-cluster=my-alloydb-cluster

gcloud database-migration migration-jobs create migrate-to-alloydb \
    --region=us-central1 \
    --type=CONTINUOUS \
    --source=source-pg \
    --destination=target-alloydb

gcloud database-migration migration-jobs start migrate-to-alloydb \
    --region=us-central1
```

---

## Part 8 — Migrating to GKE (Containerization)

### Manual Containerization Strategy

```
┌────────────────────────────────────────────────────────────────────┐
│         CONTAINERIZATION APPROACH                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Step 1: Analyze application                                       │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Identify processes, ports, config files               │     │
│  │  • Map filesystem dependencies                           │     │
│  │  • Identify state (stateless vs stateful)               │     │
│  │  • Document environment variables & secrets             │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Step 2: Write Dockerfile                                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  FROM base-image                                          │     │
│  │  COPY application code                                    │     │
│  │  RUN install dependencies                                 │     │
│  │  EXPOSE port                                              │     │
│  │  CMD start application                                    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Step 3: Create Kubernetes manifests                              │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Deployment (replicas, resources, probes)              │     │
│  │  • Service (ClusterIP, LoadBalancer)                     │     │
│  │  • ConfigMap / Secret (config, credentials)              │     │
│  │  • PersistentVolumeClaim (if stateful)                   │     │
│  │  • HPA (autoscaling)                                     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Step 4: Deploy to GKE                                            │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Build image → Artifact Registry                       │     │
│  │  • Deploy to GKE (Autopilot or Standard)                │     │
│  │  • Configure Ingress, TLS, DNS                          │     │
│  │  • Set up CI/CD with Cloud Build                        │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 9 — Migrate to Containers

### Migrate to Containers (M2C)

```
┌────────────────────────────────────────────────────────────────────┐
│         MIGRATE TO CONTAINERS                                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Automated VM → Container migration:                              │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Source VM                                                │     │
│  │  (Linux VM running on-prem, GCE, or AWS)                │     │
│  │       │                                                    │     │
│  │       ▼ Analyze (discover processes, ports, filesystems) │     │
│  │  ┌──────────────────────────────────────────────┐        │     │
│  │  │ Migration Plan                                │        │     │
│  │  │ • Auto-generated Dockerfile                   │        │     │
│  │  │ • Auto-generated Kubernetes manifests         │        │     │
│  │  │ • Data volumes mapped to PVCs                 │        │     │
│  │  │ • Review and customize before proceeding     │        │     │
│  │  └──────────────┬───────────────────────────────┘        │     │
│  │                  │                                         │     │
│  │                  ▼ Generate artifacts                      │     │
│  │  ┌──────────────────────────────────────────────┐        │     │
│  │  │ Docker image + K8s YAML                       │        │     │
│  │  │ → Push to Artifact Registry                   │        │     │
│  │  │ → Deploy to GKE or Cloud Run                 │        │     │
│  │  └──────────────────────────────────────────────┘        │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Supported sources: VMware, Compute Engine, AWS EC2, Azure VMs   │
│  Target: GKE, Cloud Run, Anthos                                  │
│  OS support: Linux (various distros)                              │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Install M2C CLI
# Download from Google Cloud Console → Migrate to Containers

# Step 1: Create a migration source
migctl source create ce my-source --project=my-project --zone=us-central1-a

# Step 2: Create migration
migctl migration create my-migration \
    --source=my-source \
    --vm-id=my-vm-instance \
    --intent=Image

# Step 3: Generate migration plan
migctl migration get-plan my-migration > plan.yaml
# Review and edit the plan

# Step 4: Execute migration
migctl migration generate-artifacts my-migration

# Step 5: Deploy to GKE
kubectl apply -f deployment_spec.yaml
```

---

## Part 10 — Application Modernization

### Re-platform and Re-architect Strategies

```
┌────────────────────────────────────────────────────────────────────┐
│         MODERNIZATION SPECTRUM                                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Least change ─────────────────────────────────► Most change      │
│                                                                      │
│  ┌─────────┐  ┌────────────┐  ┌──────────┐  ┌─────────────┐     │
│  │Rehost   │  │Replatform  │  │Refactor  │  │Rebuild      │     │
│  │(VM→VM)  │  │(VM→managed)│  │(monolith │  │(cloud-native│     │
│  │         │  │            │  │→services)│  │from scratch)│     │
│  └─────────┘  └────────────┘  └──────────┘  └─────────────┘     │
│                                                                      │
│  Common replatform targets:                                        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  On-prem MySQL    → Cloud SQL MySQL (managed)           │     │
│  │  Self-hosted Redis → Memorystore Redis                  │     │
│  │  Apache on VM      → Cloud Run (serverless)             │     │
│  │  Cron jobs on VM   → Cloud Scheduler + Cloud Functions  │     │
│  │  NFS filer         → Filestore (managed NFS)            │     │
│  │  On-prem Kafka     → Pub/Sub (managed messaging)       │     │
│  │  Jenkins on VM     → Cloud Build (managed CI/CD)       │     │
│  │  On-prem K8s       → GKE Autopilot                     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Common refactor patterns:                                         │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Monolith → Microservices (GKE / Cloud Run)             │     │
│  │  Synchronous → Event-driven (Pub/Sub + Cloud Functions) │     │
│  │  Batch processing → Streaming (Dataflow)                │     │
│  │  Single DB → Database-per-service                       │     │
│  │  Session affinity → Stateless + Memorystore             │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 11 — VMware Engine Migration

### Google Cloud VMware Engine (GCVE)

```
┌────────────────────────────────────────────────────────────────────┐
│         GOOGLE CLOUD VMWARE ENGINE                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Run existing VMware workloads natively on GCP:                   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  On-Premises                    Google Cloud              │     │
│  │  ┌───────────────┐            ┌───────────────────┐     │     │
│  │  │ vCenter       │ ──vMotion──│ VMware Engine      │     │     │
│  │  │ ESXi hosts    │ ──HCX────► │ (vCenter + ESXi   │     │     │
│  │  │ vSAN          │            │  + vSAN + NSX-T)  │     │     │
│  │  │ NSX           │            │                    │     │     │
│  │  │ ┌─────┐┌────┐│            │ ┌─────┐ ┌────┐   │     │     │
│  │  │ │ VM1 ││VM2 ││            │ │ VM1 │ │VM2 │   │     │     │
│  │  │ └─────┘└────┘│            │ └─────┘ └────┘   │     │     │
│  │  └───────────────┘            └───────────────────┘     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Features:                                                          │
│  • Same VMware tools (vCenter, vMotion, HCX)                     │
│  • No app changes required                                        │
│  • Dedicated bare-metal infrastructure                            │
│  • Access to GCP services (BigQuery, AI, etc.)                   │
│  • Private connectivity to VPC                                    │
│                                                                      │
│  Use when:                                                          │
│  • VMware-dependent workloads (licensing, tooling)               │
│  • Rapid data center evacuation                                   │
│  • Extend on-prem VMware to cloud (hybrid)                       │
│                                                                      │
│  Pricing: per node-hour (dedicated hosts)                         │
│  Minimum: 3 nodes per private cloud                               │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 12 — Transfer Appliance

### Physical Data Transfer

```
┌────────────────────────────────────────────────────────────────────┐
│         TRANSFER APPLIANCE                                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  For large data transfers where network is too slow:              │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  1. Order appliance from Google                           │     │
│  │  2. Connect to on-prem network                           │     │
│  │  3. Copy data to appliance (encrypted)                   │     │
│  │  4. Ship appliance to Google                             │     │
│  │  5. Google loads data into GCS                           │     │
│  │                                                            │     │
│  │  Sizes:                                                   │     │
│  │  ┌──────────────────────────────┐                        │     │
│  │  │ TA40  — 40 TB usable        │                        │     │
│  │  │ TA300 — 300 TB usable       │                        │     │
│  │  └──────────────────────────────┘                        │     │
│  │                                                            │     │
│  │  When to use:                                             │     │
│  │  • > 20 TB data to transfer                              │     │
│  │  • Upload would take > 1 week over network              │     │
│  │  • Limited or expensive bandwidth                        │     │
│  │                                                            │     │
│  │  Security:                                                │     │
│  │  • AES-256 encryption at rest                            │     │
│  │  • Encryption key stays with you (never sent)           │     │
│  │  • Tamper-evident packaging                              │     │
│  │  • Data wiped after upload confirmed                    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Comparison:                                                        │
│  | Method                | Time (100 TB, 1 Gbps) | Cost       |   │
│  |-----------------------|----------------------|------------|   │
│  | Network transfer      | ~10 days             | Egress fees|   │
│  | Transfer Appliance    | ~5 days (ship)       | Per session|   │
│  | Dedicated Interconnect| ~2 days              | Monthly fee|   │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 13 — Network & DNS Migration

### Hybrid Connectivity Setup

```
┌────────────────────────────────────────────────────────────────────┐
│         NETWORK MIGRATION                                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Phase 1: Establish hybrid connectivity                            │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Option A: Cloud VPN (encrypted over internet)           │     │
│  │  • Quick to set up (hours)                               │     │
│  │  • Up to 3 Gbps per tunnel                              │     │
│  │  • Good for initial migration & testing                 │     │
│  │                                                            │     │
│  │  Option B: Cloud Interconnect (dedicated link)           │     │
│  │  • Higher bandwidth (10–200 Gbps)                       │     │
│  │  • Lower latency, more reliable                         │     │
│  │  • Production workloads & large data transfers          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Phase 2: DNS migration strategy                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Step 1: Create Cloud DNS zones (don't switch yet)       │     │
│  │  Step 2: Replicate all records to Cloud DNS              │     │
│  │  Step 3: Lower TTL on existing DNS (e.g., 60s)         │     │
│  │  Step 4: Update NS records at registrar → Cloud DNS     │     │
│  │  Step 5: Monitor resolution, restore TTL after stable   │     │
│  │                                                            │     │
│  │  Split-horizon DNS (hybrid period):                      │     │
│  │  • On-prem DNS resolves internal services to on-prem    │     │
│  │  • Cloud DNS resolves migrated services to GCP          │     │
│  │  • DNS forwarding zones bridge the two                  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Phase 3: Cutover (move traffic to GCP)                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Update DNS records to point to GCP IPs               │     │
│  │  • Use Cloud Load Balancing for gradual traffic shift   │     │
│  │  • Monitor latency, errors, throughput                  │     │
│  │  • Keep VPN/Interconnect as failback (temporary)        │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Database Migration Service ──────────────────────────────
resource "google_database_migration_service_connection_profile" "source" {
  connection_profile_id = "source-mysql"
  display_name          = "Source MySQL"
  location              = var.region
  project               = var.project_id

  mysql {
    host     = "10.0.0.50"
    port     = 3306
    username = "migration_user"
    password = var.db_password
  }
}

resource "google_database_migration_service_connection_profile" "target" {
  connection_profile_id = "target-cloudsql"
  display_name          = "Target Cloud SQL"
  location              = var.region
  project               = var.project_id

  cloudsql {
    settings {
      database_version = "MYSQL_8_0"
      tier             = "db-n1-standard-4"
      source_id        = "projects/${var.project_id}/locations/${var.region}/connectionProfiles/source-mysql"

      ip_config {
        enable_ipv4    = false
        private_network = google_compute_network.vpc.id
      }
    }
  }
}

resource "google_database_migration_service_migration_job" "migration" {
  migration_job_id = "migrate-prod"
  display_name     = "Migrate Production DB"
  location         = var.region
  project          = var.project_id
  type             = "CONTINUOUS"
  source           = google_database_migration_service_connection_profile.source.name
  destination      = google_database_migration_service_connection_profile.target.name
}

# ─── VPN for Hybrid Connectivity ─────────────────────────────
resource "google_compute_vpn_gateway" "migration" {
  name    = "migration-vpn-gateway"
  network = google_compute_network.vpc.id
  region  = var.region
  project = var.project_id
}

resource "google_compute_vpn_tunnel" "to_onprem" {
  name                  = "to-onprem"
  peer_ip               = var.onprem_vpn_ip
  shared_secret         = var.vpn_shared_secret
  target_vpn_gateway    = google_compute_vpn_gateway.migration.id
  local_traffic_selector  = ["10.128.0.0/20"]
  remote_traffic_selector = ["192.168.0.0/16"]
  region                = var.region
  project               = var.project_id
}

# ─── VMware Engine ────────────────────────────────────────────
resource "google_vmwareengine_private_cloud" "pc" {
  name     = "my-private-cloud"
  location = "${var.region}-a"
  project  = var.project_id

  management_cluster {
    cluster_id = "mgmt-cluster"
    node_type_configs {
      node_type_id = "standard-72"
      node_count   = 3
    }
  }

  network_config {
    management_cidr       = "192.168.0.0/24"
    vmware_engine_network = google_vmwareengine_network.network.id
  }
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# MIGRATE FOR COMPUTE ENGINE
# ═══════════════════════════════════════════════════════════════
gcloud migration vms sources list --location=REGION
gcloud migration vms sources create NAME --location=R --type=TYPE
gcloud migration vms migrating-vms create NAME --source=S --location=R
gcloud migration vms migrating-vms start-migration NAME --location=R
gcloud migration vms clone-jobs create NAME --migrating-vm=VM --location=R
gcloud migration vms cutover-jobs create NAME --migrating-vm=VM --location=R

# ═══════════════════════════════════════════════════════════════
# DATABASE MIGRATION SERVICE
# ═══════════════════════════════════════════════════════════════
gcloud database-migration connection-profiles create NAME \
    --region=R --type=TYPE [--host=H --port=P --username=U]
gcloud database-migration connection-profiles list --region=R
gcloud database-migration migration-jobs create NAME \
    --region=R --type=continuous --source=S --destination=D
gcloud database-migration migration-jobs start NAME --region=R
gcloud database-migration migration-jobs describe NAME --region=R
gcloud database-migration migration-jobs promote NAME --region=R
gcloud database-migration migration-jobs delete NAME --region=R

# ═══════════════════════════════════════════════════════════════
# VMWARE ENGINE
# ═══════════════════════════════════════════════════════════════
gcloud vmware private-clouds create NAME --location=ZONE \
    --cluster-id=CLUSTER --node-type-id=standard-72 --node-count=3 \
    --management-range=192.168.0.0/24 --vmware-engine-network=NET
gcloud vmware private-clouds list --location=ZONE
gcloud vmware private-clouds describe NAME --location=ZONE

# ═══════════════════════════════════════════════════════════════
# CLOUD SQL IMPORT (dump/restore)
# ═══════════════════════════════════════════════════════════════
gcloud sql import sql INSTANCE gs://BUCKET/dump.sql --database=DB
gcloud sql import csv INSTANCE gs://BUCKET/data.csv --database=DB --table=TABLE

# ═══════════════════════════════════════════════════════════════
# TRANSFER APPLIANCE
# ═══════════════════════════════════════════════════════════════
# Order via Console → Transfer Appliance → Create Order
# Track: Console → Transfer Appliance → Orders
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Data Center Evacuation

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: DATA CENTER EXIT (6-MONTH PLAN)                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Month 1-2: ASSESS & PLAN                                           │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • Deploy Migration Center discovery agents               │        │
│  │  • Inventory 200 VMs, 15 databases, 50 TB storage        │        │
│  │  • Categorize workloads: rehost (150), replatform (30),  │        │
│  │    refactor (10), retire (10)                             │        │
│  │  • Set up landing zone (VPC, IAM, billing)               │        │
│  │  • Establish Cloud VPN → later upgrade to Interconnect   │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Month 2-3: WAVE 1 (Non-critical workloads)                         │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • Migrate dev/staging VMs (M4CE)                         │        │
│  │  • Migrate dev databases (DMS — continuous)              │        │
│  │  • Transfer static data (gsutil / Transfer Service)      │        │
│  │  • Validate & test                                        │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Month 3-4: WAVE 2 (Business applications)                          │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • Migrate production VMs (M4CE with test clones)        │        │
│  │  • Migrate prod MySQL → Cloud SQL (DMS continuous)       │        │
│  │  • Replatform: self-hosted Redis → Memorystore          │        │
│  │  • Weekend cutover for each app group                    │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Month 4-5: WAVE 3 (Critical systems)                                │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • Migrate core databases (DMS promote)                   │        │
│  │  • Migrate DNS to Cloud DNS                               │        │
│  │  • Transfer remaining 50 TB bulk data (Transfer Appliance)│       │
│  │  • Final cutover with <1 hour downtime window            │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Month 5-6: OPTIMIZE                                                  │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  • Right-size VMs (Recommender)                           │        │
│  │  • Purchase CUDs for stable workloads                    │        │
│  │  • Set up monitoring & alerting                          │        │
│  │  • Decommission on-prem hardware                         │        │
│  │  • Document & train operations team                      │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Database Migration with Zero Downtime

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: ZERO-DOWNTIME DB MIGRATION                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Week 1: Setup                                             │        │
│  │  ├── Create Cloud SQL target instance (same config)       │        │
│  │  ├── Set up DMS source connection profile                 │        │
│  │  ├── Create continuous migration job                      │        │
│  │  └── Start full dump + CDC replication                    │        │
│  │                                                            │        │
│  │  Week 1-2: Replication                                    │        │
│  │  ├── Monitor replication lag (should be < 1 second)       │        │
│  │  ├── Validate data in target (row counts, checksums)     │        │
│  │  └── Run read-only queries against Cloud SQL to verify   │        │
│  │                                                            │        │
│  │  Week 2: Application preparation                         │        │
│  │  ├── Update app config to support dual-write (optional)  │        │
│  │  ├── Prepare DNS/connection string switch                │        │
│  │  └── Schedule maintenance window (10 min)                │        │
│  │                                                            │        │
│  │  Cutover (10-minute window):                              │        │
│  │  ├── 1. Stop application writes to source (2 min)        │        │
│  │  ├── 2. Verify replication lag = 0 (1 min)               │        │
│  │  ├── 3. Promote Cloud SQL target (DMS promote) (2 min)  │        │
│  │  ├── 4. Update connection strings → Cloud SQL (2 min)    │        │
│  │  ├── 5. Restart application (2 min)                      │        │
│  │  └── 6. Verify writes going to Cloud SQL (1 min)        │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Rollback plan: keep source DB for 1 week as read-only backup     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Strangler Fig (Gradual Modernization)

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: STRANGLER FIG MIGRATION                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Gradually replace monolith with microservices:                     │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │                                                            │        │
│  │  Cloud Load Balancer (traffic router)                     │        │
│  │       │                                                    │        │
│  │       ├── /api/orders  ──► Cloud Run (new microservice)  │        │
│  │       ├── /api/payments ──► Cloud Run (new microservice) │        │
│  │       ├── /api/users   ──► GKE (new microservice)        │        │
│  │       └── /* (everything else) ──► Monolith on GCE       │        │
│  │                                                            │        │
│  │  Over time:                                               │        │
│  │  ┌──────────────────────────────────────────────┐        │        │
│  │  │  Sprint 1: Extract /api/orders               │        │        │
│  │  │  Sprint 2: Extract /api/payments             │        │        │
│  │  │  Sprint 3: Extract /api/users                │        │        │
│  │  │  Sprint 4: Extract /api/products             │        │        │
│  │  │  ...                                          │        │        │
│  │  │  Sprint N: Decommission monolith             │        │        │
│  │  └──────────────────────────────────────────────┘        │        │
│  │                                                            │        │
│  │  Each extraction:                                         │        │
│  │  • Write new service (Cloud Run / GKE)                   │        │
│  │  • Add URL map rule to load balancer                     │        │
│  │  • Shift traffic gradually (10% → 50% → 100%)           │        │
│  │  • Remove code from monolith                             │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Migration Type | Tool | Downtime |
|---------------|------|----------|
| VM → GCE | Migrate for Compute Engine | Minutes (cutover) |
| MySQL/PG → Cloud SQL | Database Migration Service | Seconds (promote) |
| MySQL/PG → AlloyDB | Database Migration Service | Seconds (promote) |
| Any DB → Cloud Spanner | Spanner Migration Tool | Varies |
| VM → Container | Migrate to Containers | Planned window |
| VMware → GCVE | VMware HCX (vMotion) | Zero (live migration) |
| Bulk data → GCS | Transfer Appliance | N/A (offline) |
| DNS | Cloud DNS | TTL-dependent |
| Network | Cloud VPN / Interconnect | Parallel setup |

---

## Console Walkthrough: Database Migration Service

### Step 1 — Create a Connection Profile (Source)

```
┌────────────────────────────────────────────────────────────────────┐
│  Console → Database Migration → Connection Profiles → CREATE      │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Go to Console → Database Migration Service                    │
│     (Navigation menu → Database Migration → Connection profiles)  │
│                                                                      │
│  2. Click "+ CREATE PROFILE"                                      │
│                                                                      │
│  3. Fill in source details:                                        │
│     ┌──────────────────────────────────────────────────────────┐  │
│     │  Profile name:    source-mysql-prod                       │  │
│     │  Database engine: MySQL (or PostgreSQL / SQL Server)      │  │
│     │  Hostname/IP:     10.0.0.50 (or public IP)               │  │
│     │  Port:            3306                                    │  │
│     │  Username:        migration_user                          │  │
│     │  Password:        ●●●●●●●●                               │  │
│     └──────────────────────────────────────────────────────────┘  │
│                                                                      │
│  4. Connectivity method:                                           │
│     • IP allowlist — if source has a public IP                   │
│     • VPN / Interconnect — if source is on-prem (private)        │
│     • Reverse SSH tunnel — if no direct connectivity             │
│                                                                      │
│  5. (Optional) Enable SSL by uploading certificates               │
│                                                                      │
│  6. Click "CREATE" — test connection automatically runs          │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

> **Tip:** The connection profile is reusable — you can use the same source profile for multiple migration jobs.

### Step 2 — Create a Migration Job

```
┌────────────────────────────────────────────────────────────────────┐
│  Console → Database Migration → Migration Jobs → CREATE           │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Go to Console → Database Migration → Migration jobs           │
│                                                                      │
│  2. Click "+ CREATE MIGRATION JOB"                                │
│                                                                      │
│  3. Basic details:                                                 │
│     ┌──────────────────────────────────────────────────────────┐  │
│     │  Job name:        migrate-prod-mysql                      │  │
│     │  Migration type:  Continuous (CDC) ← recommended         │  │
│     │                   One-time (full dump only)               │  │
│     │  Source engine:   MySQL                                   │  │
│     └──────────────────────────────────────────────────────────┘  │
│                                                                      │
│  4. Source connection profile:                                     │
│     • Select "source-mysql-prod" (created in Step 1)             │
│     • Click "TEST CONNECTION" → verify green checkmark           │
│                                                                      │
│  5. Target:                                                        │
│     • Create new Cloud SQL instance (recommended)                 │
│       OR select existing Cloud SQL instance                       │
│     ┌──────────────────────────────────────────────────────────┐  │
│     │  Instance ID:     target-prod-mysql                       │  │
│     │  Region:          us-central1                              │  │
│     │  Machine type:    db-n1-standard-4 (match source sizing) │  │
│     │  Storage:         100 GB SSD (auto-increase enabled)      │  │
│     └──────────────────────────────────────────────────────────┘  │
│                                                                      │
│  6. Databases to migrate:                                          │
│     • All databases (default)                                     │
│     • OR select specific databases                                │
│                                                                      │
│  7. Connectivity method:                                           │
│     • VPC peering (recommended for private connectivity)          │
│     • Reverse SSH tunnel                                          │
│                                                                      │
│  8. Click "CREATE & START" (or "CREATE" to start later)          │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Step 3 — Monitor Migration Progress

```
┌────────────────────────────────────────────────────────────────────┐
│  Console → Database Migration → Migration Jobs → [Your Job]       │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Migration job status indicators:                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  ● NOT STARTED    — job created but not started          │     │
│  │  ● STARTING       — initial setup in progress            │     │
│  │  ● FULL DUMP      — initial data load (can take hours)   │     │
│  │  ● CDC            — continuous replication active ✓      │     │
│  │  ● PROMOTING      — cutover in progress                  │     │
│  │  ● COMPLETED      — migration finished successfully      │     │
│  │  ● FAILED         — check error details                  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  What to monitor:                                                  │
│  • Replication delay — should be near 0 before cutover           │
│  • Data transfer progress — bytes replicated vs total            │
│  • Error count — any replication errors                           │
│                                                                      │
│  When ready to cutover:                                            │
│  1. Verify replication delay is 0 (or near 0)                    │
│  2. Click "PROMOTE" button                                       │
│  3. Confirm promotion — this stops replication and makes         │
│     the target Cloud SQL the primary database                    │
│  4. Downtime: typically just seconds                              │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Step 4 — Post-Migration Validation Checklist

| # | Validation Step | How to Check |
|---|----------------|---------------|
| 1 | Row counts match | Run `SELECT COUNT(*)` on key tables in both source and target |
| 2 | Schema integrity | Compare table schemas — columns, indexes, constraints |
| 3 | Data spot-check | Query random rows from large tables — compare values |
| 4 | Application connectivity | Update app connection strings to point to Cloud SQL |
| 5 | Performance baseline | Run typical queries — compare execution times |
| 6 | Replication stopped | Confirm DMS job shows "COMPLETED" status |
| 7 | Backups enabled | Verify Cloud SQL automated backups are configured |
| 8 | Monitoring set up | Check Cloud Monitoring for CPU, memory, connections |
| 9 | IAM permissions | Verify app service accounts have `cloudsql.client` role |
| 10 | Decommission source | After validation period, shut down source database |

> **Important:** Keep the source database running for a validation period (1–7 days recommended) before decommissioning. This gives you a rollback option if issues are discovered.

---

## What's Next?

Continue to **Chapter 61: Storage Transfer & BigQuery Transfer** → `61-data-transfer.md`
