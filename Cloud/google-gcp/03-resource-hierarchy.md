# Chapter 3: Resource Hierarchy & Projects (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: GCP Resource Hierarchy (Deep Dive)](#part-1-gcp-resource-hierarchy-deep-dive)
- [Part 2: How Resources Are Identified in GCP](#part-2-how-resources-are-identified-in-gcp)
- [Part 3: Resource Naming Conventions](#part-3-resource-naming-conventions)
- [Part 4: Labels Strategy (GCP's Tags)](#part-4-labels-strategy-gcps-tags)
- [Part 5: Project Management at Scale](#part-5-project-management-at-scale)
- [Part 6: Quotas & Limits](#part-6-quotas--limits)
- [Part 10: Console Walkthrough — Projects & Folders](#part-10-console-walkthrough--projects--folders)

---

## Overview

This chapter covers how resources are organized, identified, and managed within GCP — including resource IDs, labels, the project-centric model, resource manager, quotas, and real-world resource management patterns.

```
What you'll learn:
├── GCP resource hierarchy in depth
├── How resources are identified (self-links, resource names)
├── Resource naming conventions
├── Labels strategy (GCP's version of tags)
├── Project management at scale
├── Quotas & limits
├── Asset Inventory (organization-wide visibility)
├── Organization Policy constraints in practice
├── Resource lifecycle management
└── Real-world patterns for resource organization
```

---

## Part 1: GCP Resource Hierarchy (Deep Dive)

### The Four Levels

```
┌─────────────────────────────────────────────────────────────────────┐
│              GCP RESOURCE HIERARCHY - COMPLETE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Level 1: ORGANIZATION (optional, but recommended)                   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Organization: techcorp.com                                   │    │
│  │ Org ID: 123456789012                                        │    │
│  │                                                               │    │
│  │ What lives here:                                             │    │
│  │ ├── Organization Policies (guardrails)                      │    │
│  │ ├── IAM Policies (inherited by all below)                   │    │
│  │ ├── Access Context Manager (VPC Service Controls)           │    │
│  │ └── Organization Admin role                                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                              │
│       ▼                                                              │
│  Level 2: FOLDERS (optional, nestable up to 10 levels)               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Folder: Production (folder ID: 111222333444)                 │    │
│  │                                                               │    │
│  │ What lives here:                                             │    │
│  │ ├── Organization Policies (scoped to this folder)           │    │
│  │ ├── IAM Policies (inherited by child folders & projects)    │    │
│  │ └── Grouping logic (no resources directly in folders)       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                              │
│       ▼                                                              │
│  Level 3: PROJECTS (required — all resources live here)              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Project: prod-backend (ID: prod-backend-a1b2c3)             │    │
│  │ Project Number: 123456789012                                │    │
│  │                                                               │    │
│  │ What lives here:                                             │    │
│  │ ├── All resources (VMs, buckets, databases...)              │    │
│  │ ├── Enabled APIs                                            │    │
│  │ ├── Service accounts                                        │    │
│  │ ├── IAM policies (project-level)                            │    │
│  │ ├── Labels                                                  │    │
│  │ ├── Billing link (to billing account)                       │    │
│  │ └── Quotas (per project)                                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                              │
│       ▼                                                              │
│  Level 4: RESOURCES (the actual cloud services)                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Resources within project:                                    │    │
│  │ ├── Compute Engine VM: prod-web-01 (zone: asia-south1-a)   │    │
│  │ ├── Cloud SQL: prod-db-main (region: asia-south1)          │    │
│  │ ├── GCS Bucket: prod-assets (multi-region: asia)           │    │
│  │ ├── GKE Cluster: prod-cluster (region: asia-south1)        │    │
│  │ └── Cloud Run: prod-api (region: asia-south1)              │    │
│  │                                                               │    │
│  │ Resources have their own IAM (some services):               │    │
│  │ ├── Bucket-level IAM                                        │    │
│  │ ├── Dataset-level IAM (BigQuery)                            │    │
│  │ └── Instance-level IAM (rare, usually project-level)       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### IAM Inheritance Flow

```
IAM Policy Inheritance (additive, cannot restrict inherited):

Organization ──── roles/viewer → auditor@company.com
    │              (Can view EVERYTHING in the org)
    │
    ▼ INHERITS DOWN
Folder: Prod ──── roles/editor → devops-group@company.com
    │              (Can edit everything in Production)
    │
    ▼ INHERITS DOWN
Project ────────── roles/cloudsql.admin → dba@company.com
    │              (Can manage Cloud SQL in this project)
    │
    ▼ INHERITS DOWN  
Resource ────────── (All above policies apply here)
    (Cloud SQL Instance)

EFFECTIVE on the Cloud SQL instance:
├── auditor@company.com → Viewer (from Organization)
├── devops-group@company.com → Editor (from Folder)
└── dba@company.com → Cloud SQL Admin (from Project)

⚠️ KEY RULE: You CANNOT remove inherited permissions!
   Only ADD more permissions or use IAM Deny policies (preview).
   To restrict access: Use separate projects/folders.
```

---

## Part 2: How Resources Are Identified in GCP

### Resource Names (Self-Links)

Every GCP resource has a unique identifier called a **resource name** (or self-link).

```
Resource Name Format:
━━━━━━━━━━━━━━━━━━━━━
//SERVICE.googleapis.com/RESOURCE_PATH

Full formats vary by service:

Compute Engine VM:
  //compute.googleapis.com/projects/PROJECT_ID/zones/ZONE/instances/INSTANCE_NAME
  Example:
  //compute.googleapis.com/projects/prod-backend-a1b2c3/zones/asia-south1-a/instances/prod-web-01

Cloud Storage Bucket:
  //storage.googleapis.com/projects/_/buckets/BUCKET_NAME
  Example:
  //storage.googleapis.com/projects/_/buckets/techcorp-prod-assets

Cloud SQL:
  //sqladmin.googleapis.com/projects/PROJECT_ID/instances/INSTANCE_NAME
  Example:
  //sqladmin.googleapis.com/projects/prod-backend-a1b2c3/instances/prod-db-main

BigQuery Dataset:
  //bigquery.googleapis.com/projects/PROJECT_ID/datasets/DATASET_ID
  
GKE Cluster:
  //container.googleapis.com/projects/PROJECT_ID/locations/LOCATION/clusters/CLUSTER_NAME

Pub/Sub Topic:
  //pubsub.googleapis.com/projects/PROJECT_ID/topics/TOPIC_NAME

Cloud Run Service:
  //run.googleapis.com/projects/PROJECT_ID/locations/LOCATION/services/SERVICE_NAME
```

### Short Resource References (Used in CLI & APIs)

```
In gcloud CLI and API calls, you use shorter references:

VM:
  projects/PROJECT_ID/zones/ZONE/instances/INSTANCE_NAME
  gcloud compute instances describe INSTANCE_NAME --zone=ZONE

Bucket:
  gs://BUCKET_NAME
  gsutil ls gs://techcorp-prod-assets/

Cloud SQL:
  projects/PROJECT_ID/instances/INSTANCE_NAME
  gcloud sql instances describe INSTANCE_NAME

GKE:
  projects/PROJECT_ID/locations/LOCATION/clusters/CLUSTER_NAME
  gcloud container clusters describe CLUSTER_NAME --region=REGION

Cloud Run:
  projects/PROJECT_ID/locations/LOCATION/services/SERVICE_NAME
  gcloud run services describe SERVICE_NAME --region=REGION
```

### Project Identifiers (Three Ways to Reference)

```
┌─────────────────────────────────────────────────────────────────┐
│           THREE WAYS TO IDENTIFY A PROJECT                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. PROJECT NAME (human-readable, not unique, changeable)       │
│     "My Production Backend"                                      │
│     └── Only used for display in Console                        │
│                                                                   │
│  2. PROJECT ID (globally unique, immutable, you choose)         │
│     "prod-backend-a1b2c3"                                       │
│     └── Used in: CLI commands, APIs, URLs, IAM policies         │
│     └── Rules: 6-30 chars, lowercase, letters/digits/hyphens   │
│     └── ⚠️ CANNOT be changed after creation!                    │
│     └── ⚠️ CANNOT be reused after project deletion (30 days)   │
│                                                                   │
│  3. PROJECT NUMBER (auto-generated, numeric, immutable)         │
│     "123456789012"                                               │
│     └── Used internally by GCP (service accounts, APIs)         │
│     └── Default service account: 123456789012@cloudservices.gsa │
│                                                                   │
│  Where each is used:                                             │
│  ├── Console URL: console.cloud.google.com/home?project=PROJECT_ID│
│  ├── gcloud CLI: gcloud config set project PROJECT_ID            │
│  ├── API calls: projects/PROJECT_ID/...                         │
│  ├── Service account: PROJECT_NUMBER-compute@developer.gsa      │
│  └── Billing: Reports show PROJECT_ID                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Resource Naming Conventions

### Naming Rules per Service

```
⚠️ Important constraints per service:

Projects:
├── Globally unique
├── 6-30 characters
├── Lowercase letters, digits, hyphens
├── Must start with lowercase letter
├── Cannot end with hyphen
└── Cannot be reused after deletion

Compute Engine VMs:
├── 1-63 characters
├── Lowercase letters, digits, hyphens
├── Must start with lowercase letter
├── Must be unique within project+zone
└── Cannot contain underscores or periods

Cloud Storage Buckets:
├── Globally unique across all GCP
├── 3-63 characters (up to 222 with dots)
├── Lowercase letters, digits, hyphens, dots
├── Must start and end with letter or number
├── Cannot look like IP addresses
└── 💡 Prefix with project-id for uniqueness

Cloud SQL Instances:
├── Unique within project
├── Lowercase letters, digits, hyphens
├── Must start with letter
├── 1-98 characters
└── ⚠️ Cannot reuse name for 1 week after deletion!

GKE Clusters:
├── Unique within project+location
├── 1-40 characters
├── Lowercase letters, digits, hyphens
└── Must start with letter, end with alphanumeric

Cloud Run Services:
├── Unique within project+region
├── 1-63 characters
├── Lowercase letters, digits, hyphens
└── Must start with letter
```

### Recommended Naming Convention

```
Pattern: <env>-<app>-<component>-<qualifier>

┌──────────────────────────────────────────────────────────────────────┐
│ Resource         │ Convention                     │ Example           │
├──────────────────────────────────────────────────────────────────────┤
│ Project          │ <company>-<env>-<service>      │ tc-prod-backend   │
│ VM Instance      │ <env>-<app>-<role>-<##>        │ prod-api-web-01   │
│ GCS Bucket       │ <company>-<env>-<purpose>      │ tc-prod-uploads   │
│ Cloud SQL        │ <env>-<app>-<engine>           │ prod-users-pg     │
│ GKE Cluster      │ <env>-<app>-cluster            │ prod-backend-cluster│
│ Cloud Run        │ <env>-<app>-<service>          │ prod-api-orders   │
│ Pub/Sub Topic    │ <env>-<app>-<event>            │ prod-orders-created│
│ VPC Network      │ <env>-vpc-<purpose>            │ prod-vpc-main     │
│ Subnet           │ <env>-subnet-<region>-<tier>   │ prod-subnet-asia-s1-private│
│ Firewall Rule    │ <env>-fw-<allow/deny>-<what>   │ prod-fw-allow-https│
│ Service Account  │ <app>-<purpose>-sa             │ api-cloudrun-sa   │
│ Cloud Function   │ <env>-<trigger>-<action>       │ prod-pubsub-process-order│
│ BigQuery Dataset │ <env>_<domain>                 │ prod_analytics    │
│ Load Balancer    │ <env>-lb-<app>-<type>          │ prod-lb-frontend-ext│
└──────────────────────────────────────────────────────────────────────┘

💡 BigQuery uses underscores (hyphens not allowed in dataset names)
```

---

## Part 4: Labels Strategy (GCP's Tags)

### What are Labels?

Labels are **key-value pairs** attached to GCP resources for organization, filtering, and cost allocation.

```
Label structure:
┌─────────────────────────────────────────────┐
│ Key              │ Value                     │
├─────────────────────────────────────────────┤
│ env              │ production                │
│ team             │ backend                   │
│ app              │ order-service             │
│ cost-center      │ cc-1234                   │
│ owner            │ john-doe                  │
│ managed-by       │ terraform                 │
│ created-date     │ 2026-05-16               │
└─────────────────────────────────────────────┘

⚠️ Label constraints:
├── Max 64 labels per resource
├── Key: 1-63 characters, lowercase letters, digits, hyphens, underscores
├── Value: 0-63 characters (can be empty!), same char rules
├── Keys MUST start with lowercase letter
├── Both key and value: lowercase only
└── No uppercase, no special characters, no spaces
```

### Labels vs Tags (GCP Terminology)

```
⚠️ IMPORTANT DISTINCTION IN GCP:

LABELS (key-value pairs on resources):
├── Used for: Organization, filtering, cost allocation
├── Applied to: Most GCP resources
├── Format: Key-value pairs (lowercase only)
├── Example: env=production, team=backend
└── Similar to: AWS Tags, Azure Tags

TAGS (network tags on VMs):
├── Used for: Firewall rule targeting ONLY
├── Applied to: VM instances only
├── Format: Simple strings (not key-value)
├── Example: "web-server", "allow-ssh", "internal-only"
└── Purpose: Apply firewall rules to VMs with matching tags

RESOURCE TAGS (newer feature):
├── Used for: IAM conditions, organization policies
├── Applied to: Projects, folders, organizations
├── Format: Tag key + tag value (managed centrally)
├── Example: env/production, cost-center/cc-1234
└── Created at org level, applied to resources

Don't confuse them! In this chapter, we focus on LABELS.
```

### Label Categories

```
┌─────────────────────────────────────────────────────────────────────┐
│                        LABEL CATEGORIES                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. BUSINESS LABELS (Cost allocation & ownership)                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Key               │ Values           │ Purpose               │    │
│  ├─────────────────────────────────────────────────────────────┤    │
│  │ cost-center       │ cc-1234          │ Billing chargeback    │    │
│  │ team              │ backend, frontend│ Team ownership        │    │
│  │ owner             │ john-doe         │ Who's responsible     │    │
│  │ department        │ engineering      │ Department tracking   │    │
│  │ business-unit     │ retail           │ BU attribution        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  2. TECHNICAL LABELS (Operations)                                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Key               │ Values           │ Purpose               │    │
│  ├─────────────────────────────────────────────────────────────┤    │
│  │ env               │ prod, staging, dev│ Environment          │    │
│  │ app               │ order-service    │ Application name      │    │
│  │ component         │ api, worker, web │ Component within app  │    │
│  │ managed-by        │ terraform, manual│ How it's managed      │    │
│  │ version           │ v1-2-3           │ Deployed version      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  3. AUTOMATION LABELS (Trigger actions)                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Key               │ Values           │ Purpose               │    │
│  ├─────────────────────────────────────────────────────────────┤    │
│  │ auto-shutdown     │ true             │ Stop at night         │    │
│  │ backup-schedule   │ daily, weekly    │ Backup automation     │    │
│  │ auto-delete-after │ 2026-08-16       │ Temp resource cleanup │    │
│  │ patch-group       │ group-a          │ Maintenance window    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  4. SECURITY LABELS                                                  │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Key               │ Values           │ Purpose               │    │
│  ├─────────────────────────────────────────────────────────────┤    │
│  │ data-classification│ public, internal │ Data sensitivity     │    │
│  │ compliance        │ pci, hipaa       │ Regulatory scope      │    │
│  │ security-zone     │ dmz, internal    │ Network tier          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Applying Labels

```
Console: Any resource → Labels section → Add label

CLI:
  # Label a VM
  gcloud compute instances update prod-web-01 \
    --zone=asia-south1-a \
    --update-labels=env=production,team=backend,app=api

  # Label a GCS bucket
  gcloud storage buckets update gs://my-bucket \
    --update-labels=env=production,team=data

  # Label a Cloud SQL instance
  gcloud sql instances patch prod-db \
    --update-labels=env=production,team=backend

  # Label a project
  gcloud projects update prod-backend-a1b2c3 \
    --update-labels=env=production,team=backend

  # Remove a label
  gcloud compute instances update prod-web-01 \
    --zone=asia-south1-a \
    --remove-labels=old-label

Terraform:
  resource "google_compute_instance" "web" {
    name         = "prod-web-01"
    machine_type = "e2-medium"
    zone         = "asia-south1-a"
    
    labels = {
      env         = "production"
      team        = "backend"
      app         = "api"
      managed-by  = "terraform"
      cost-center = "cc-1234"
    }
  }
```

### Using Labels for Cost Reporting

```
Console → Billing → Reports

┌─────────────────────────────────────────────────────────────────┐
│                    BILLING REPORTS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Group by: [Label: team ▼]                                       │
│                                                                   │
│ Results:                                                         │
│ ┌───────────────────────────────────────────────────┐           │
│ │ Team          │ This Month  │ Last Month  │ Trend │           │
│ ├───────────────────────────────────────────────────┤           │
│ │ backend       │ $2,340      │ $2,100      │ ↑12%  │           │
│ │ frontend      │ $890        │ $850        │ ↑5%   │           │
│ │ data          │ $3,100      │ $2,900      │ ↑7%   │           │
│ │ platform      │ $1,200      │ $1,200      │ →0%   │           │
│ │ (no label)    │ $450        │ $500        │ ↓10%  │           │
│ └───────────────────────────────────────────────────┘           │
│                                                                   │
│ Filter by: Label: env = production                              │
│ Group by: Label: app                                            │
│ → Shows per-application costs in production                     │
│                                                                   │
│ 💡 Labels must be applied to resources to show up here!          │
│ 💡 Unlabeled resources show as "(no label)"                     │
│ 💡 BigQuery billing export provides most detailed analysis      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Project Management at Scale

### Creating Projects Programmatically

```
For organizations with many projects:

CLI:
  gcloud projects create tc-prod-newservice \
    --name="New Service Production" \
    --folder=FOLDER_ID \
    --labels=env=production,team=backend

  # Link to billing
  gcloud billing projects link tc-prod-newservice \
    --billing-account=BILLING_ACCOUNT_ID

  # Enable required APIs
  gcloud services enable \
    compute.googleapis.com \
    container.googleapis.com \
    cloudsql.googleapis.com \
    run.googleapis.com \
    --project=tc-prod-newservice

Terraform (recommended for production):
  resource "google_project" "new_service" {
    name            = "New Service Production"
    project_id      = "tc-prod-newservice"
    folder_id       = google_folder.production.name
    billing_account = var.billing_account_id
    
    labels = {
      env  = "production"
      team = "backend"
    }
  }

  resource "google_project_service" "apis" {
    for_each = toset([
      "compute.googleapis.com",
      "container.googleapis.com",
      "run.googleapis.com",
    ])
    project = google_project.new_service.project_id
    service = each.value
  }
```

### Project Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROJECT LIFECYCLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  CREATE ──→ ACTIVE ──→ DELETE_REQUESTED ──→ DELETED              │
│                              │                                   │
│                              │ (30-day recovery window)          │
│                              ▼                                   │
│                         RESTORE possible                         │
│                         (within 30 days)                          │
│                                                                   │
│  After 30 days: Project permanently deleted                     │
│  ├── All resources destroyed                                    │
│  ├── Project ID cannot be reused                                │
│  └── Billing stops immediately on delete request                │
│                                                                   │
│  ⚠️ Deleting a project = DELETING ALL RESOURCES inside it!      │
│     This is like "rm -rf" for your cloud resources.             │
│     Always double-check before deleting a project.              │
│                                                                   │
│  Shut down a project:                                            │
│  Console → IAM & Admin → Manage Resources → Select → Delete    │
│  gcloud projects delete PROJECT_ID                              │
│                                                                   │
│  Restore within 30 days:                                         │
│  gcloud projects undelete PROJECT_ID                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Quotas & Limits

### What are Quotas?

Every GCP project has quotas (limits) on resource usage. These protect against unexpected spending and ensure fair resource sharing.

```
Console → IAM & Admin → Quotas (per project)

┌─────────────────────────────────────────────────────────────────┐
│                   COMMON GCP QUOTAS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Compute Engine:                                                  │
│ ├── CPUs (all regions): 24 (default, per project)               │
│ ├── CPUs per region: 8-24 (varies)                              │
│ ├── Persistent disk (GB): 2,048 per project                    │
│ ├── Local SSD (GB): 0 (must request)                            │
│ ├── In-use IP addresses: 8 per region                          │
│ ├── Static IP addresses: 8 per region                          │
│ ├── Instances per zone: varies                                  │
│ └── GPUs: 0 (must request)                                      │
│                                                                   │
│ VPC:                                                             │
│ ├── Networks per project: 15                                    │
│ ├── Subnets per network: 100                                   │
│ ├── Firewall rules per project: 200                            │
│ ├── Routes per network: 200                                    │
│ └── Internal forwarding rules per VPC: 75                      │
│                                                                   │
│ Cloud Storage:                                                   │
│ ├── Buckets per project: No limit                              │
│ ├── Objects per bucket: No limit                               │
│ ├── Object size: 5 TB max                                      │
│ └── Write ops: 1 request per second per object name            │
│                                                                   │
│ Cloud SQL:                                                       │
│ ├── Instances per project: 100                                 │
│ ├── Storage per instance: 64 TB                                │
│ └── Read replicas: 10 per primary                              │
│                                                                   │
│ GKE:                                                             │
│ ├── Clusters per zone/region: 100                              │
│ ├── Nodes per cluster: 15,000                                  │
│ ├── Pods per node: 110                                         │
│ └── Nodes per node pool: 1,000                                 │
│                                                                   │
│ Cloud Run:                                                       │
│ ├── Services per project per region: 1,000                     │
│ ├── Revisions per service: 1,000                               │
│ └── Max instances per service: 1,000 (adjustable)              │
│                                                                   │
│ Cloud Functions:                                                 │
│ ├── Functions per project per region: 1,000                    │
│ ├── Concurrent executions: 3,000                               │
│ └── Build time: 120 minutes max                                │
│                                                                   │
│ BigQuery:                                                        │
│ ├── Datasets per project: No limit                             │
│ ├── Tables per dataset: No limit                               │
│ ├── Concurrent queries: 100                                    │
│ └── Query size: 1 MB (query text), result: 10 GB (flat)       │
│                                                                   │
│ Pub/Sub:                                                         │
│ ├── Topics per project: 10,000                                 │
│ ├── Subscriptions per project: 10,000                          │
│ └── Message size: 10 MB                                        │
│                                                                   │
│ API Requests (all services):                                     │
│ ├── Rate limits vary by API                                    │
│ └── Typically: 100-1,000 requests/second                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Requesting Quota Increases

```
Console → IAM & Admin → Quotas → Filter → Select quota → Edit Quotas

┌─────────────────────────────────────────────────────────────────┐
│ Service: Compute Engine                                          │
│ Quota: CPUs (asia-south1)                                       │
│ Current limit: 24                                               │
│ New limit: [100]                                                │
│                                                                   │
│ Request description:                                             │
│ "Scaling production cluster for Q3 release.                     │
│  Expected to need 80 vCPUs across 20 VMs."                      │
│                                                                   │
│ [Submit request]                                                 │
│                                                                   │
│ ⏱️ Processing: Usually 2-3 business days                         │
│ 💡 Some quotas auto-approve up to certain limits                │
│ 💡 Tip: Request ahead of time!                                  │
│                                                                   │
│ CLI:                                                             │
│ gcloud compute project-info describe --project=PROJECT_ID        │
│ (Shows current quotas)                                          │
│                                                                   │
│ gcloud quotas info list --service=compute.googleapis.com         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Cloud Asset Inventory

### What is Cloud Asset Inventory?

Cloud Asset Inventory provides a unified view of all resources across your entire organization.

```
┌─────────────────────────────────────────────────────────────────┐
│                   CLOUD ASSET INVENTORY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What it does:                                                    │
│ ├── Search all resources across all projects in org             │
│ ├── View resource metadata and relationships                    │
│ ├── Track configuration history (changes over time)             │
│ ├── Export entire inventory to BigQuery/GCS                     │
│ ├── Analyze IAM policies across organization                   │
│ └── Monitor for configuration drift                            │
│                                                                   │
│ Use cases:                                                       │
│ ├── "Show me all VMs without labels across all projects"        │
│ ├── "Who has Owner role in any project?"                        │
│ ├── "List all public Cloud SQL instances"                       │
│ ├── "What changed in the last 24 hours?"                        │
│ └── "Export all resources for compliance audit"                  │
│                                                                   │
│ CLI examples:                                                    │
│                                                                   │
│ # Search all VMs across org                                      │
│ gcloud asset search-all-resources \                              │
│   --scope=organizations/ORG_ID \                                │
│   --asset-types=compute.googleapis.com/Instance                 │
│                                                                   │
│ # Search IAM policies                                            │
│ gcloud asset search-all-iam-policies \                           │
│   --scope=organizations/ORG_ID \                                │
│   --query="policy:roles/owner"                                  │
│                                                                   │
│ # Export to BigQuery (for analysis)                              │
│ gcloud asset export \                                            │
│   --organization=ORG_ID \                                       │
│   --content-type=resource \                                     │
│   --bigquery-table=projects/PROJECT/datasets/DS/tables/TBL     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Resource Tags (Newer Feature — IAM Integration)

### What are Resource Tags?

Resource Tags (different from labels!) are managed at the organization level and can be used in IAM Conditions and Organization Policies.

```
┌─────────────────────────────────────────────────────────────────┐
│                    RESOURCE TAGS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Labels vs Resource Tags:                                         │
│ ┌──────────────┬───────────────────┬──────────────────────────┐ │
│ │              │ Labels            │ Resource Tags             │ │
│ ├──────────────┼───────────────────┼──────────────────────────┤ │
│ │ Managed by   │ Anyone with access│ Org Admin (centrally)    │ │
│ │ Applied to   │ Most resources    │ Projects, folders, orgs  │ │
│ │ Values       │ Free-form text    │ Pre-defined allowed vals │ │
│ │ Used in IAM  │ No                │ Yes (conditions)         │ │
│ │ Used in Org  │ No                │ Yes (conditionally)      │ │
│ │  Policies    │                   │                          │ │
│ │ Inheritance  │ No                │ Yes (parent to child)    │ │
│ │ Governance   │ Loose             │ Strict (controlled)      │ │
│ └──────────────┴───────────────────┴──────────────────────────┘ │
│                                                                   │
│ Example:                                                         │
│ Tag Key: "env" (created at org level)                           │
│ Tag Values: "production", "staging", "development"              │
│                                                                   │
│ Applied: Project "prod-backend" → tagged env:production         │
│                                                                   │
│ IAM Condition:                                                   │
│ "Grant roles/compute.admin to devops-team@company.com          │
│  ONLY on resources tagged env:production"                       │
│                                                                   │
│ Organization Policy Condition:                                   │
│ "Deny external IPs on VMs ONLY in projects tagged              │
│  env:production (allow in dev)"                                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Complete Resource Hierarchy Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│              COMPLETE GCP RESOURCE HIERARCHY                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Organization: techcorp.com                                          │
│  │  ├── Org Policies (constraints)                                  │
│  │  ├── IAM (inherited by all)                                     │
│  │  ├── Resource Tags (tag keys defined here)                      │
│  │  └── Cloud Asset Inventory (org-wide view)                      │
│  │                                                                   │
│  ├── Folder: Production                                             │
│  │   │  ├── Org Policies (scoped)                                  │
│  │   │  ├── IAM (inherited down)                                   │
│  │   │  └── Resource Tags (applied here, inherited to projects)    │
│  │   │                                                               │
│  │   └── Project: tc-prod-backend                                   │
│  │       │  ├── Project Settings                                    │
│  │       │  ├── Enabled APIs                                        │
│  │       │  ├── Service Accounts                                    │
│  │       │  ├── IAM Policies                                        │
│  │       │  ├── Labels (on the project itself)                     │
│  │       │  ├── Linked Billing Account                             │
│  │       │  └── Quotas                                             │
│  │       │                                                           │
│  │       ├── Regional Resources:                                    │
│  │       │   ├── Compute Engine (zonal)                            │
│  │       │   │   └── VM instances, disks, snapshots                │
│  │       │   ├── Cloud SQL (regional)                              │
│  │       │   │   └── Instances, replicas, backups                  │
│  │       │   ├── GKE (regional/zonal)                              │
│  │       │   │   └── Clusters, node pools, workloads              │
│  │       │   ├── Cloud Run (regional)                              │
│  │       │   │   └── Services, revisions                          │
│  │       │   └── Memorystore (regional)                            │
│  │       │       └── Redis/Memcached instances                     │
│  │       │                                                           │
│  │       ├── Multi-Regional/Global Resources:                       │
│  │       │   ├── Cloud Storage (multi-region/region)               │
│  │       │   │   └── Buckets, objects                              │
│  │       │   ├── BigQuery (multi-region)                           │
│  │       │   │   └── Datasets, tables, views                      │
│  │       │   ├── VPC Network (global!)                             │
│  │       │   │   └── Subnets (regional), FW rules, routes        │
│  │       │   ├── Cloud DNS (global)                                │
│  │       │   │   └── Zones, record sets                           │
│  │       │   └── Load Balancers (global/regional)                  │
│  │       │                                                           │
│  │       └── LABELS on every resource:                             │
│  │           env=production, team=backend, app=api...              │
│  │                                                                   │
│  ├── Folder: Shared Services                                       │
│  │   └── Project: tc-shared-networking (Shared VPC Host)           │
│  │       └── VPC shared to other projects (service projects)       │
│  │                                                                   │
│  └── Folder: Sandbox                                               │
│      └── Project: tc-sandbox                                       │
│                                                                       │
│  BILLING:                                                            │
│  Billing Account → linked to projects                               │
│  Costs tracked by: project, service, SKU, label                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Console Walkthrough — Projects & Folders

### Creating a New Project

```
Console → IAM & Admin → Manage Resources → CREATE PROJECT
(or use the project dropdown at top of page → NEW PROJECT)

┌─────────────────────────────────────────────────────────────────┐
│           NEW PROJECT                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Project name ──                                               │
│ Name: [my-web-app]                                               │
│ ⚡ Display name — can contain spaces, can be changed later.      │
│                                                                   │
│ ── Project ID ──                                                 │
│ ID: [my-web-app-a1b2c3]                                         │
│ ⚡ Auto-generated from name. Click EDIT to customize.            │
│ ⚠️ CANNOT be changed after creation! Must be globally unique.    │
│    Rules: 6-30 chars, lowercase letters, digits, hyphens only.  │
│                                                                   │
│ ── Organization ──                                               │
│ Organization: [techcorp.com ▼]                                   │
│ ⚡ Only visible if you have a Google Workspace / Cloud Identity  │
│   organization. Otherwise shows "No organization".              │
│                                                                   │
│ ── Location (Parent) ──                                          │
│ Location: [techcorp.com ▼]                                       │
│ [BROWSE] → Select Organization or Folder as parent              │
│   ├── techcorp.com (Organization)                               │
│   │   ├── Production (Folder)                                   │
│   │   ├── Development (Folder)                                  │
│   │   └── Sandbox (Folder)                                      │
│ ⚡ Choose the folder where this project should live.             │
│   If no folder, it goes directly under the Organization.        │
│                                                                   │
│                                   [CREATE]                       │
│                                                                   │
│ After creation:                                                  │
│ ├── Project appears in the project selector dropdown            │
│ ├── Billing must be linked (Settings → Billing)                 │
│ ├── APIs must be enabled (APIs & Services → Enable APIs)        │
│ └── Project number is auto-assigned (cannot be changed)         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a Folder

```
Console → IAM & Admin → Manage Resources → CREATE FOLDER
(Requires Organization — folders need a parent org or folder)

┌─────────────────────────────────────────────────────────────────┐
│           NEW FOLDER                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Folder name ──                                                │
│ Name: [Production]                                               │
│ ⚡ Descriptive name for the folder. Can be changed later.        │
│                                                                   │
│ ── Organization or folder (Parent) ──                            │
│ Parent: [techcorp.com ▼]                                         │
│ [BROWSE] → Select Organization or existing Folder               │
│   ├── techcorp.com (Organization — top level)                   │
│   │   ├── Engineering (Folder)                                  │
│   │   └── Finance (Folder)                                      │
│ ⚡ Folders can be nested up to 10 levels deep.                   │
│                                                                   │
│                                   [CREATE]                       │
│                                                                   │
│ Required permissions:                                            │
│ ├── resourcemanager.folders.create on the parent                │
│ └── Typically: roles/resourcemanager.folderCreator              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Managing Labels on a Project

```
Console → IAM & Admin → Manage Resources → Select project → Click "⋮" → Edit labels

┌─────────────────────────────────────────────────────────────────┐
│           EDIT LABELS                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Key                    Value                                    │
│  ┌──────────────┐      ┌──────────────┐                        │
│  │ environment  │      │ production   │                        │
│  ├──────────────┤      ├──────────────┤                        │
│  │ team         │      │ backend      │                        │
│  ├──────────────┤      ├──────────────┤                        │
│  │ cost-center  │      │ eng-001      │                        │
│  └──────────────┘      └──────────────┘                        │
│  [+ Add label]                                                  │
│                                                                   │
│ ⚡ Label rules:                                                  │
│   Key: 1-63 chars, lowercase letters/numbers/hyphens/underscores│
│   Value: 0-63 chars, same character rules                       │
│   Max 64 labels per resource                                    │
│                                                                   │
│                      [SAVE]   [CANCEL]                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Requesting Quota Increase

```
Console → IAM & Admin → Quotas & System Limits

┌─────────────────────────────────────────────────────────────────┐
│           QUOTAS & SYSTEM LIMITS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Filter: [Service ▼] [Metric ▼] [Location ▼]                    │
│                                                                   │
│ ┌────────────────────────────────────────────────────────────┐  │
│ │ Service          │ Metric              │ Limit │ Usage    │  │
│ ├────────────────────────────────────────────────────────────┤  │
│ │ Compute Engine   │ CPUs (all regions)  │  24   │ 8 (33%) │  │
│ │ Compute Engine   │ GPUs (all regions)  │   1   │ 0 (0%)  │  │
│ │ Compute Engine   │ In-use IP addresses │  24   │ 3 (13%) │  │
│ │ Cloud Storage    │ Requests/sec        │ 5000  │ 200     │  │
│ └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│ To request increase:                                             │
│ 1. Select the quota(s) with checkbox ☑                          │
│ 2. Click [EDIT QUOTAS] at the top                               │
│ 3. Fill in new limit request and justification                  │
│ 4. Submit — Google reviews (usually approved within 24-48h)    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Real-World Resource Management Patterns

### Pattern 1: Label-Based Cost Attribution

```
Company requirement: "Show spending by team and application"

Implementation:
1. Define mandatory labels in a labeling policy document:
   ├── env (production/staging/development)
   ├── team (backend/frontend/data/platform)
   ├── app (service name)
   └── cost-center (department code)

2. Enforce via Org Policy + Cloud Function:
   ├── Cloud Function triggered on resource creation
   ├── Checks if mandatory labels exist
   ├── If missing → Notify owner + add "untagged=true" label
   └── Weekly report: "Resources missing labels"

3. Billing Export to BigQuery:
   Console → Billing → Billing Export → BigQuery export (enable)
   
   Query:
   SELECT labels.value as team, SUM(cost) as total_cost
   FROM `billing_export.gcp_billing_export_v1_XXXXX`
   WHERE labels.key = "team"
   GROUP BY team
   ORDER BY total_cost DESC

4. Looker Studio Dashboard → Visual reports for leadership
```

### Pattern 2: Automated Cleanup with Labels

```
Scenario: Delete temporary resources after expiration

Label: auto-delete-after=2026-08-16

Cloud Scheduler (daily at midnight):
  → Triggers Cloud Function
  → Function logic:
    1. List all resources with "auto-delete-after" label
    2. Check if date is past today
    3. If past → Delete resource
    4. If within 3 days → Send warning email
    5. Log all actions

Saves money on forgotten dev/test resources.
```

### Pattern 3: Project Factory (Automated Project Creation)

```
For large organizations that create projects frequently:

Project Factory pattern (Terraform module):
┌─────────────────────────────────────────────────────┐
│ Input:                                               │
│ ├── project_name: "new-microservice"                │
│ ├── environment: "production"                       │
│ ├── team: "backend"                                 │
│ ├── billing_account: "XXXXX"                        │
│ └── apis_needed: ["run", "sql", "pubsub"]          │
│                                                      │
│ Output (automatically created):                      │
│ ├── Project: tc-prod-new-microservice               │
│ ├── Placed in: Production folder                    │
│ ├── Billing linked                                  │
│ ├── APIs enabled                                    │
│ ├── Default labels applied                          │
│ ├── Service accounts created                        │
│ ├── IAM bindings (team access)                      │
│ ├── VPC subnets (from Shared VPC)                   │
│ ├── Budget alerts configured                        │
│ └── Monitoring dashboard created                    │
└─────────────────────────────────────────────────────┘

Self-service: Team fills a form → PR generated → Approved → Terraform applies
Result: New project ready in ~10 minutes
```

---

## Quick Reference: Console Navigation

| Task | Console Path |
|------|-------------|
| Create Project | Project Selector (top bar) → New Project |
| View all resources | IAM & Admin → Manage Resources |
| Project settings | IAM & Admin → Settings |
| Manage labels (project) | IAM & Admin → Settings → Labels |
| View quotas | IAM & Admin → Quotas |
| Request quota increase | Quotas → Filter → Edit Quotas |
| Enable APIs | APIs & Services → Enable APIs & Services |
| Asset Inventory | Asset Inventory (search) |
| Billing labels | Billing → Reports → Group by label |
| Organization Policies | IAM & Admin → Organization Policies |

---

## What's Next?

In the next chapter, we'll cover IAM (Identity & Access Management) in depth — members, roles, policies, service accounts, workload identity, and least-privilege patterns.

→ Next: [Chapter 4: IAM - Identity & Access Management](04-iam.md)

---

*Last Updated: May 2026*
