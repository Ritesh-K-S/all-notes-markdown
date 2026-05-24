# Chapter 4: IAM - Identity & Access Management (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: GCP IAM Model](#part-1-gcp-iam-model)
- [Part 2: Members (Identities)](#part-2-members-identities)
- [Part 3: Roles](#part-3-roles)
- [Part 4: IAM Policies (Bindings)](#part-4-iam-policies-bindings)
- [Part 5: Service Accounts](#part-5-service-accounts)
- [Part 6: Workload Identity Federation](#part-6-workload-identity-federation)
- [Part 7: IAM Conditions](#part-7-iam-conditions)
- [Part 8: IAM Deny Policies](#part-8-iam-deny-policies)
- [Part 11: Console Walkthrough — IAM, Service Accounts & Custom Roles](#part-11-console-walkthrough--iam-service-accounts--custom-roles)

---

## Overview

GCP IAM controls **who** (identity) has **what access** (role) to **which resource**. Unlike AWS where policies are JSON documents attached to identities, GCP uses a binding model: you bind a member to a role on a resource.

```
What you'll learn:
├── GCP IAM model (members + roles + resources)
├── Members (Google accounts, groups, service accounts, domains)
├── Roles (Basic, Predefined, Custom)
├── IAM Policies (bindings on resources)
├── Service Accounts (machine identities)
├── Workload Identity Federation (external workloads)
├── IAM Conditions (attribute-based access)
├── Organization Policies vs IAM (when to use each)
├── IAM Best Practices
├── IAM Deny Policies (new feature)
└── Real-world patterns (startup, mid-size, enterprise)
```

---

## Part 1: GCP IAM Model

### The Three-Part Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GCP IAM MODEL                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  WHO (Member)  +  WHAT (Role)  +  WHERE (Resource)                   │
│       │                │                    │                         │
│       ▼                ▼                    ▼                         │
│  "john@co.com"   "roles/compute.admin"   "project: prod-backend"    │
│                                                                       │
│  This creates a BINDING:                                             │
│  "john@co.com has Compute Admin permissions on project prod-backend" │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │              IAM POLICY (on a resource)                     │     │
│  ├────────────────────────────────────────────────────────────┤     │
│  │ Binding 1:                                                  │     │
│  │   Role: roles/compute.admin                                │     │
│  │   Members:                                                  │     │
│  │   ├── user:john@company.com                                │     │
│  │   └── group:devops@company.com                             │     │
│  │                                                             │     │
│  │ Binding 2:                                                  │     │
│  │   Role: roles/viewer                                       │     │
│  │   Members:                                                  │     │
│  │   ├── user:intern@company.com                              │     │
│  │   └── serviceAccount:monitoring@proj.iam.gserviceaccount   │     │
│  │                                                             │     │
│  │ Binding 3:                                                  │     │
│  │   Role: roles/storage.objectViewer                         │     │
│  │   Members:                                                  │     │
│  │   └── allUsers (public — be careful!)                      │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                       │
│  KEY DIFFERENCE FROM AWS:                                            │
│  ├── AWS: Policies attached TO users/roles                          │
│  ├── GCP: Policies attached TO resources (binding members to roles) │
│  └── GCP asks: "Who has what role on THIS resource?"                │
│      AWS asks: "What can THIS user do?"                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Where IAM Policies Live

```
IAM policies can be set at ANY level of the hierarchy:

Organization ─── IAM Policy (applies to everything)
    │
    ├── Folder ─── IAM Policy (applies to all projects in folder)
    │       │
    │       └── Project ─── IAM Policy (applies to all resources)
    │               │
    │               └── Resource ─── IAM Policy (applies to this resource)
    │                   (bucket, dataset, VM, etc.)
    │
    └── Folder ─── ...

INHERITANCE: Policies are ADDITIVE and inherited downward
├── If you have Editor at org level → you're Editor EVERYWHERE
├── Cannot remove inherited permissions at lower levels
├── Can only ADD more permissions at lower levels
└── ⚠️ Be very careful with org/folder level bindings!
```

---

## Part 2: Members (Identities)

### Types of Members

```
┌─────────────────────────────────────────────────────────────────────┐
│                       MEMBER TYPES                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Google Account (individual user)                                  │
│    Format: user:email@gmail.com or user:email@company.com            │
│    Example: user:john.doe@techcorp.com                               │
│    → Represents a single person with a Google identity               │
│                                                                       │
│ 2. Google Group                                                      │
│    Format: group:groupname@domain.com                                │
│    Example: group:devops-team@techcorp.com                           │
│    → All members of the Google Group get the role                    │
│    → ✅ RECOMMENDED: Always use groups (not individual users)        │
│                                                                       │
│ 3. Service Account (machine identity)                                │
│    Format: serviceAccount:name@project-id.iam.gserviceaccount.com   │
│    Example: serviceAccount:api-sa@prod-backend.iam.gserviceaccount  │
│    → For applications, VMs, Cloud Run, Cloud Functions               │
│                                                                       │
│ 4. Google Workspace Domain                                           │
│    Format: domain:company.com                                        │
│    Example: domain:techcorp.com                                      │
│    → ALL users in the Workspace domain get the role                  │
│    → ⚠️ Very broad! Use for viewer-level at most                    │
│                                                                       │
│ 5. Cloud Identity Domain                                             │
│    Format: domain:company.com (same syntax)                          │
│    → All users in Cloud Identity get the role                        │
│                                                                       │
│ 6. allUsers (PUBLIC — anyone on internet)                            │
│    Format: allUsers                                                   │
│    → ⚠️ Makes resource PUBLIC! Use only for intentionally public    │
│    → Example: public website bucket, public API                      │
│                                                                       │
│ 7. allAuthenticatedUsers (any Google account)                        │
│    Format: allAuthenticatedUsers                                     │
│    → Any logged-in Google user (not just your org!)                  │
│    → ⚠️ Still very broad! Rarely what you want                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Roles

### Role Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ROLE TYPES                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. BASIC ROLES (formerly Primitive — very broad!)                    │
│ ┌────────────────────────────────────────────────────────────────┐  │
│ │ Role           │ Permissions                                    │  │
│ ├────────────────────────────────────────────────────────────────┤  │
│ │ roles/viewer   │ Read-only access to all resources             │  │
│ │ roles/editor   │ Viewer + create/modify resources              │  │
│ │                │ ⚠️ Does NOT include IAM management            │  │
│ │ roles/owner    │ Editor + manage IAM + billing                 │  │
│ │                │ ⚠️ Full control! Very dangerous at org/folder │  │
│ │ roles/browser  │ List projects & folders (minimal)             │  │
│ └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚠️ DO NOT use Basic roles in production!                             │
│    They grant way too many permissions (thousands of actions)        │
│    Example: roles/editor grants 2000+ permissions across all services│
│                                                                       │
│ 2. PREDEFINED ROLES (created by Google, service-specific)            │
│ ┌────────────────────────────────────────────────────────────────┐  │
│ │ Format: roles/SERVICE.ROLE_NAME                                │  │
│ │                                                                 │  │
│ │ Compute Engine:                                                │  │
│ │ ├── roles/compute.admin         (full compute access)         │  │
│ │ ├── roles/compute.instanceAdmin (manage instances only)       │  │
│ │ ├── roles/compute.networkAdmin  (manage networking)           │  │
│ │ ├── roles/compute.viewer        (read-only compute)           │  │
│ │ └── roles/compute.osLogin       (SSH to VMs)                  │  │
│ │                                                                 │  │
│ │ Cloud Storage:                                                 │  │
│ │ ├── roles/storage.admin         (full storage access)         │  │
│ │ ├── roles/storage.objectAdmin   (manage objects)              │  │
│ │ ├── roles/storage.objectViewer  (read objects)                │  │
│ │ ├── roles/storage.objectCreator (create objects only)         │  │
│ │ └── roles/storage.hmacKeyAdmin  (manage HMAC keys)            │  │
│ │                                                                 │  │
│ │ Cloud SQL:                                                     │  │
│ │ ├── roles/cloudsql.admin        (full SQL access)             │  │
│ │ ├── roles/cloudsql.editor       (modify instances)            │  │
│ │ ├── roles/cloudsql.viewer       (read-only)                   │  │
│ │ └── roles/cloudsql.client       (connect to databases)        │  │
│ │                                                                 │  │
│ │ BigQuery:                                                      │  │
│ │ ├── roles/bigquery.admin        (full BQ access)              │  │
│ │ ├── roles/bigquery.dataEditor   (create/modify data)          │  │
│ │ ├── roles/bigquery.dataViewer   (read data)                   │  │
│ │ ├── roles/bigquery.jobUser      (run queries)                 │  │
│ │ └── roles/bigquery.user         (list datasets, run jobs)     │  │
│ │                                                                 │  │
│ │ GKE:                                                           │  │
│ │ ├── roles/container.admin       (full GKE access)             │  │
│ │ ├── roles/container.clusterAdmin(manage clusters)             │  │
│ │ ├── roles/container.developer   (deploy workloads)            │  │
│ │ └── roles/container.viewer      (read-only)                   │  │
│ │                                                                 │  │
│ │ IAM:                                                           │  │
│ │ ├── roles/iam.securityAdmin     (manage IAM policies)         │  │
│ │ ├── roles/iam.roleAdmin         (create custom roles)         │  │
│ │ └── roles/iam.serviceAccountAdmin(manage service accounts)    │  │
│ │                                                                 │  │
│ │ Cloud Run:                                                     │  │
│ │ ├── roles/run.admin             (full Cloud Run access)       │  │
│ │ ├── roles/run.developer         (deploy services)             │  │
│ │ ├── roles/run.invoker           (invoke services)             │  │
│ │ └── roles/run.viewer            (read-only)                   │  │
│ │                                                                 │  │
│ │ 900+ predefined roles available across all services!          │  │
│ └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 3. CUSTOM ROLES (you define the exact permissions)                   │
│ ┌────────────────────────────────────────────────────────────────┐  │
│ │ Created at: Organization level or Project level               │  │
│ │ Format: projects/PROJECT/roles/ROLE_NAME                      │  │
│ │     or: organizations/ORG/roles/ROLE_NAME                     │  │
│ │                                                                 │  │
│ │ When to use:                                                   │  │
│ │ ├── Predefined role is too broad                              │  │
│ │ ├── Need to combine permissions from multiple services        │  │
│ │ └── Compliance requires exact permission list                 │  │
│ │                                                                 │  │
│ │ Limitations:                                                   │  │
│ │ ├── Some permissions cannot be used in custom roles           │  │
│ │ ├── You must maintain them (new permissions won't auto-add)   │  │
│ │ └── Max 300 custom roles per project, 300 per org             │  │
│ └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Custom Role

```
Console → IAM & Admin → Roles → Create Role

┌─────────────────────────────────────────────────────────────────┐
│                   CREATE CUSTOM ROLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Title:       [Cloud SQL Developer]                               │
│ Description: [Can connect to Cloud SQL and manage databases,     │
│               but cannot delete instances or change networking]   │
│ ID:          [cloudSqlDeveloper]                                  │
│ Role launch stage: [General Availability ▼]                      │
│                                                                   │
│ Add permissions:                                                 │
│ ☑ cloudsql.databases.create                                     │
│ ☑ cloudsql.databases.get                                        │
│ ☑ cloudsql.databases.list                                       │
│ ☑ cloudsql.databases.update                                     │
│ ☑ cloudsql.instances.connect                                    │
│ ☑ cloudsql.instances.get                                        │
│ ☑ cloudsql.instances.list                                       │
│ ☐ cloudsql.instances.delete    ← NOT included                   │
│ ☐ cloudsql.instances.create    ← NOT included                   │
│ ☑ cloudsql.users.create                                         │
│ ☑ cloudsql.users.list                                           │
│ ☑ cloudsql.users.update                                         │
│                                                                   │
│ [CREATE]                                                         │
└─────────────────────────────────────────────────────────────────┘

CLI:
  gcloud iam roles create cloudSqlDeveloper \
    --project=prod-backend \
    --title="Cloud SQL Developer" \
    --description="Connect and manage databases, no delete" \
    --permissions=cloudsql.databases.create,cloudsql.databases.get,\
cloudsql.databases.list,cloudsql.instances.connect,cloudsql.instances.get,\
cloudsql.instances.list,cloudsql.users.create,cloudsql.users.list

  # Or from YAML file
  gcloud iam roles create cloudSqlDeveloper \
    --project=prod-backend \
    --file=role-definition.yaml
```

---

## Part 4: IAM Policies (Bindings)

### Granting Access

```
Console → IAM & Admin → IAM → Grant Access

┌─────────────────────────────────────────────────────────────────┐
│                    GRANT ACCESS                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Resource: Project "prod-backend" (can also be org, folder, etc) │
│                                                                   │
│ New principals:                                                  │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ [group:devops-team@techcorp.com]                           │   │
│ │                                                            │   │
│ │ Assign roles:                                             │   │
│ │ Role 1: [Compute Instance Admin ▼]                        │   │
│ │         roles/compute.instanceAdmin.v1                    │   │
│ │ [+ Add another role]                                      │   │
│ │ Role 2: [Storage Object Admin ▼]                          │   │
│ │         roles/storage.objectAdmin                         │   │
│ │                                                            │   │
│ │ IAM Condition (optional):                                 │   │
│ │ [+ Add IAM condition]                                     │   │
│ │ Title: "Only during business hours"                       │   │
│ │ Expression: request.time.getHours("Asia/Kolkata") >= 9 && │   │
│ │            request.time.getHours("Asia/Kolkata") <= 18    │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Save]                                                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Grant a role to a user on a project
  gcloud projects add-iam-policy-binding prod-backend \
    --member="user:john@techcorp.com" \
    --role="roles/compute.instanceAdmin.v1"

  # Grant a role to a group
  gcloud projects add-iam-policy-binding prod-backend \
    --member="group:devops-team@techcorp.com" \
    --role="roles/editor"

  # Grant a role on a specific resource (bucket)
  gcloud storage buckets add-iam-policy-binding gs://prod-data-bucket \
    --member="serviceAccount:api-sa@prod-backend.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

  # Grant with condition
  gcloud projects add-iam-policy-binding prod-backend \
    --member="group:contractors@techcorp.com" \
    --role="roles/compute.viewer" \
    --condition='expression=request.time.getHours("Asia/Kolkata") >= 9 && request.time.getHours("Asia/Kolkata") <= 18,title=Business hours only'

  # View current IAM policy
  gcloud projects get-iam-policy prod-backend

  # Remove a binding
  gcloud projects remove-iam-policy-binding prod-backend \
    --member="user:john@techcorp.com" \
    --role="roles/compute.instanceAdmin.v1"
```

---

## Part 5: Service Accounts

### What is a Service Account?

A Service Account is a **special Google account for machines** (applications, VMs, services). It acts as both an identity and a resource.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SERVICE ACCOUNTS                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  What: A non-human identity for applications                         │
│  Format: NAME@PROJECT_ID.iam.gserviceaccount.com                    │
│  Example: api-backend@prod-backend.iam.gserviceaccount.com          │
│                                                                       │
│  Types:                                                              │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ Type              │ Description                             │     │
│  ├────────────────────────────────────────────────────────────┤     │
│  │ User-managed      │ You create and manage (recommended)    │     │
│  │ Default           │ Auto-created per service (e.g.,        │     │
│  │                   │ Compute Engine default SA)              │     │
│  │                   │ ⚠️ Has Editor role — too broad!         │     │
│  │ Google-managed    │ Internal to GCP (you don't see them)   │     │
│  │                   │ e.g., Google APIs Service Agent         │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                       │
│  Default service accounts (auto-created):                            │
│  ├── Compute Engine: PROJECT_NUMBER-compute@developer.gserviceaccount│
│  │   ⚠️ Has Editor role by default — REMOVE THIS!                   │
│  ├── App Engine: PROJECT_ID@appspot.gserviceaccount.com              │
│  └── Cloud Build: PROJECT_NUMBER@cloudbuild.gserviceaccount.com     │
│                                                                       │
│  DUAL NATURE of Service Accounts:                                    │
│  1. As Identity: SA can call APIs (like a user)                     │
│  2. As Resource: Users can be granted permission to USE the SA      │
│     ├── roles/iam.serviceAccountUser → Attach SA to resources       │
│     ├── roles/iam.serviceAccountTokenCreator → Generate tokens      │
│     └── roles/iam.serviceAccountKeyAdmin → Manage SA keys           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating & Using Service Accounts

```
Console → IAM & Admin → Service Accounts → Create Service Account

┌─────────────────────────────────────────────────────────────────┐
│              CREATE SERVICE ACCOUNT                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Step 1: Service account details                                  │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Name:        [api-backend]                                │   │
│ │ ID:          [api-backend] (auto-generated from name)     │   │
│ │ Description: [Service account for backend API running      │   │
│ │               on Cloud Run to access Cloud SQL & GCS]     │   │
│ │                                                            │   │
│ │ Email (auto): api-backend@prod-backend.iam.gserviceaccount│   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Step 2: Grant this service account access to project (optional) │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Role: Cloud SQL Client (roles/cloudsql.client)            │   │
│ │ Role: Storage Object Viewer (roles/storage.objectViewer)  │   │
│ │                                                            │   │
│ │ ⚠️ Or skip and grant later at resource level for least    │   │
│ │    privilege (e.g., only on specific bucket)              │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ Step 3: Grant users access to this service account (optional)   │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Service account users:                                    │   │
│ │   group:devops-team@techcorp.com → Service Account User   │   │
│ │                                                            │   │
│ │ This means devops team can deploy workloads using this SA │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ [Done]                                                           │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create service account
  gcloud iam service-accounts create api-backend \
    --display-name="API Backend Service Account" \
    --description="For Cloud Run backend accessing SQL and GCS"

  # Grant role to SA on project
  gcloud projects add-iam-policy-binding prod-backend \
    --member="serviceAccount:api-backend@prod-backend.iam.gserviceaccount.com" \
    --role="roles/cloudsql.client"

  # Grant role to SA on specific bucket
  gcloud storage buckets add-iam-policy-binding gs://prod-uploads \
    --member="serviceAccount:api-backend@prod-backend.iam.gserviceaccount.com" \
    --role="roles/storage.objectAdmin"
```

### Service Account Keys vs Better Alternatives

```
┌─────────────────────────────────────────────────────────────────────┐
│            SERVICE ACCOUNT AUTHENTICATION                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Method 1: SA Keys (JSON key files) — AVOID if possible              │
│ ├── Creates a long-lived private key (never expires)                │
│ ├── ⚠️ If leaked, attacker has full SA permissions                  │
│ ├── ⚠️ Hard to track where keys are used                           │
│ ├── Must rotate manually                                            │
│ └── Only use for: on-premises/external apps with no other option    │
│                                                                       │
│ Method 2: Attached Service Account (preferred for GCP resources)    │
│ ├── VM: Attach SA → VM gets SA identity automatically              │
│ ├── Cloud Run: Set SA → Service gets SA identity                   │
│ ├── Cloud Function: Set SA → Function gets SA identity             │
│ ├── GKE: Workload Identity (pod → SA mapping)                      │
│ ├── No key files! Credentials auto-managed by GCP                  │
│ └── ✅ ALWAYS use this for workloads running ON GCP                 │
│                                                                       │
│ Method 3: Workload Identity Federation (for external workloads)     │
│ ├── GitHub Actions → GCP (no SA keys!)                              │
│ ├── AWS workloads → GCP                                             │
│ ├── Azure workloads → GCP                                           │
│ ├── On-premises with OIDC provider → GCP                           │
│ └── ✅ Best for CI/CD and multi-cloud                                │
│                                                                       │
│ Method 4: Impersonation                                              │
│ ├── User/SA can "impersonate" another SA                            │
│ ├── Gets short-lived token for target SA                            │
│ ├── Requires: roles/iam.serviceAccountTokenCreator                  │
│ └── Use for: Admin tasks that need specific SA context              │
│                                                                       │
│ Decision tree:                                                       │
│ Running on GCP? → Attached Service Account                          │
│ CI/CD (GitHub, GitLab)? → Workload Identity Federation             │
│ Multi-cloud? → Workload Identity Federation                         │
│ On-premises (no OIDC)? → SA Key (last resort, rotate often)        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Workload Identity Federation

### What is Workload Identity Federation?

Allows external identities (AWS, Azure, GitHub, OIDC) to access GCP resources **without service account keys**.

```
┌─────────────────────────────────────────────────────────────────────┐
│              WORKLOAD IDENTITY FEDERATION                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Flow: External → Identity Pool → GCP Service Account → GCP Resources│
│                                                                       │
│ GitHub Actions example:                                              │
│                                                                       │
│  GitHub Action ──→ Workload Identity Pool ──→ SA ──→ GCS, Cloud Run │
│      (OIDC token)    (validates token)     (impersonate)             │
│                                                                       │
│ Setup:                                                               │
│ 1. Create Workload Identity Pool                                    │
│ 2. Create Provider (GitHub OIDC)                                    │
│ 3. Grant SA impersonation to pool                                   │
│ 4. Configure GitHub Action with pool details                        │
│                                                                       │
│ CLI:                                                                 │
│ # Create pool                                                        │
│ gcloud iam workload-identity-pools create "github-pool" \            │
│   --location="global" \                                              │
│   --description="GitHub Actions" \                                   │
│   --display-name="GitHub Pool"                                       │
│                                                                       │
│ # Create provider                                                    │
│ gcloud iam workload-identity-pools providers create-oidc "github" \  │
│   --location="global" \                                              │
│   --workload-identity-pool="github-pool" \                           │
│   --issuer-uri="https://token.actions.githubusercontent.com" \       │
│   --attribute-mapping="google.subject=assertion.sub,\                │
│     attribute.repository=assertion.repository"                       │
│                                                                       │
│ # Allow pool to impersonate SA                                       │
│ gcloud iam service-accounts add-iam-policy-binding \                 │
│   deploy-sa@prod-backend.iam.gserviceaccount.com \                   │
│   --member="principalSet://iam.googleapis.com/projects/PROJECT_NUM/\ │
│     locations/global/workloadIdentityPools/github-pool/\             │
│     attribute.repository/techcorp/backend-repo" \                    │
│   --role="roles/iam.workloadIdentityUser"                            │
│                                                                       │
│ GitHub Actions YAML:                                                 │
│ - uses: google-github-actions/auth@v2                                │
│   with:                                                              │
│     workload_identity_provider: 'projects/123/locations/global/...'  │
│     service_account: 'deploy-sa@prod-backend.iam.gserviceaccount'   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: IAM Conditions

### What are IAM Conditions?

Conditions add **attribute-based access control** — grant access ONLY when certain conditions are met.

```
Console → IAM → Edit binding → Add IAM condition

┌─────────────────────────────────────────────────────────────────┐
│                   IAM CONDITION                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Condition builder:                                               │
│                                                                   │
│ Available attributes:                                            │
│ ├── resource.name (resource being accessed)                     │
│ ├── resource.type (type of resource)                            │
│ ├── resource.service (which API)                                │
│ ├── request.time (current time)                                 │
│ ├── resource.name.startsWith() / .endsWith()                    │
│ └── Access level (VPC Service Controls)                         │
│                                                                   │
│ Example conditions:                                              │
│                                                                   │
│ 1. Time-based access (business hours only):                     │
│    request.time.getHours("Asia/Kolkata") >= 9 &&                │
│    request.time.getHours("Asia/Kolkata") <= 18 &&               │
│    request.time.getDayOfWeek("Asia/Kolkata") >= 1 &&            │
│    request.time.getDayOfWeek("Asia/Kolkata") <= 5               │
│                                                                   │
│ 2. Resource name prefix (only access staging buckets):          │
│    resource.name.startsWith("projects/_/buckets/staging-")      │
│                                                                   │
│ 3. Temporary access (expires on a date):                        │
│    request.time < timestamp("2026-08-01T00:00:00Z")             │
│                                                                   │
│ 4. Resource type restriction (only manage VMs, not disks):      │
│    resource.type == "compute.googleapis.com/Instance"            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 8: IAM Deny Policies

### What are Deny Policies?

Deny policies allow you to **explicitly deny access** — even if an allow policy grants it. This solves the problem of "IAM is additive" (normally you can't reduce inherited permissions).

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IAM DENY POLICIES                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Normal IAM: Allow-only, additive, cannot remove inherited            │
│ Deny Policies: Can BLOCK specific actions (overrides Allow!)         │
│                                                                       │
│ Evaluation order:                                                    │
│ 1. Check Deny policies → if denied → DENIED (stop)                 │
│ 2. Check Allow policies → if allowed → ALLOWED                     │
│ 3. Default: DENIED (implicit deny)                                  │
│                                                                       │
│ Example: Deny project deletion for everyone except org admin        │
│                                                                       │
│ gcloud iam policies create deny-project-delete \                     │
│   --attachment-point="cloudresourcemanager.googleapis.com/\           │
│     organizations/ORG_ID" \                                          │
│   --kind=denypolicies \                                              │
│   --policy-file=deny-policy.json                                     │
│                                                                       │
│ deny-policy.json:                                                    │
│ {                                                                    │
│   "displayName": "Deny project deletion",                           │
│   "rules": [{                                                        │
│     "denyRule": {                                                    │
│       "deniedPrincipals": ["principalSet://goog/public:all"],       │
│       "exceptionPrincipals": [                                      │
│         "principal://goog/subject/orgadmin@techcorp.com"            │
│       ],                                                             │
│       "deniedPermissions": [                                        │
│         "cloudresourcemanager.googleapis.com/projects.delete"       │
│       ]                                                              │
│     }                                                                │
│   }]                                                                 │
│ }                                                                    │
│                                                                       │
│ Use cases:                                                           │
│ ├── Block specific dangerous actions (delete prod resources)        │
│ ├── Prevent privilege escalation (deny IAM changes)                 │
│ ├── Restrict inherited Editor role (deny certain services)          │
│ └── Temporary block during maintenance windows                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Organization Policies vs IAM

```
┌─────────────────────────────────────────────────────────────────────┐
│              ORGANIZATION POLICIES vs IAM                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ IAM answers: "WHO can do WHAT?"                                      │
│ Org Policies answer: "WHAT is allowed to exist?" (regardless of who)│
│                                                                       │
│ ┌─────────────────────┬─────────────────────────────────────────┐   │
│ │ IAM                  │ Organization Policies                   │   │
│ ├─────────────────────┼─────────────────────────────────────────┤   │
│ │ Controls who can act │ Controls what's possible               │   │
│ │ "John can create VMs"│ "No one can create external IPs"       │   │
│ │ Per-member control   │ Per-resource/config control            │   │
│ │ Can be granted to    │ Applies to EVERYONE (even admins)      │   │
│ │ specific users       │                                        │   │
│ │ Additive (allow)     │ Restrictive (constrain)               │   │
│ └─────────────────────┴─────────────────────────────────────────┘   │
│                                                                       │
│ Common Org Policy constraints:                                       │
│ ├── constraints/compute.vmExternalIpAccess → Deny external IPs     │
│ ├── constraints/iam.allowedPolicyMemberDomains → Only your domain  │
│ ├── constraints/gcp.resourceLocations → Only certain regions        │
│ ├── constraints/compute.requireShieldedVm → Force shielded VMs     │
│ ├── constraints/sql.restrictPublicIp → No public Cloud SQL         │
│ └── constraints/storage.uniformBucketLevelAccess → Force uniform    │
│                                                                       │
│ Example: "Even if someone has Owner role, they CANNOT create         │
│  a VM with external IP in production projects"                       │
│ → This is Org Policy, not IAM                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: IAM Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IAM BEST PRACTICES                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. LEAST PRIVILEGE                                                  │
│  ├── Use predefined roles (not basic roles like Editor)             │
│  ├── Grant at the narrowest scope (resource > project > folder)     │
│  ├── Use IAM Recommender to identify over-permissioned accounts     │
│  ├── Review permissions quarterly                                   │
│  └── Use custom roles when predefined are too broad                 │
│                                                                       │
│  2. USE GROUPS (not individual users)                                │
│  ├── Create Google Groups for each team/role                        │
│  ├── Add IAM bindings to groups, not users                          │
│  ├── When person leaves → remove from group → access revoked        │
│  └── Easier to audit (fewer bindings to review)                     │
│                                                                       │
│  3. SERVICE ACCOUNTS                                                 │
│  ├── Create dedicated SA per workload (not shared)                  │
│  ├── Never use default compute SA (has Editor role!)                │
│  ├── Avoid SA keys — use attached SA or Workload Identity           │
│  ├── Disable unused service accounts                                │
│  └── Monitor SA key usage with Security Command Center             │
│                                                                       │
│  4. AUDIT & MONITOR                                                  │
│  ├── Enable Cloud Audit Logs (Admin Activity = always on)           │
│  ├── Enable Data Access logs for sensitive resources                │
│  ├── Use IAM Recommender for right-sizing permissions              │
│  ├── Policy Analyzer: "Who has access to X?"                        │
│  └── Policy Troubleshooter: "Why was access denied?"                │
│                                                                       │
│  5. SEPARATION OF DUTIES                                             │
│  ├── Different teams for networking vs compute vs data              │
│  ├── No single person should have full admin everywhere             │
│  ├── Use folders to isolate environments                            │
│  └── Separate billing admin from resource admin                     │
│                                                                       │
│  6. SECURITY                                                         │
│  ├── Enable 2FA/MFA for all users (via Workspace/Cloud Identity)   │
│  ├── Use BeyondCorp Enterprise for zero-trust access               │
│  ├── Constrain who can impersonate service accounts                 │
│  ├── Use VPC Service Controls for data exfiltration prevention     │
│  └── Set up alerts for IAM policy changes                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Console Walkthrough — IAM, Service Accounts & Custom Roles

### Granting IAM Role to a Member

```
Console → IAM & Admin → IAM → GRANT ACCESS

┌─────────────────────────────────────────────────────────────────┐
│           GRANT ACCESS                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── New principals ──                                             │
│ [john@techcorp.com]                                              │
│ ⚡ Can add multiple members at once (comma-separated).           │
│   Accepts: user:, group:, serviceAccount:, domain:              │
│                                                                   │
│ ── Assign roles ──                                               │
│ Role: [Cloud SQL Admin ▼]                                       │
│ ⚡ Search by name or filter by service.                          │
│   Recommended: Use predefined roles (NOT basic like Editor).    │
│                                                                   │
│ [+ ADD ANOTHER ROLE]  ← Can assign multiple roles at once      │
│   Role: [Storage Object Viewer ▼]                               │
│                                                                   │
│ ── IAM Condition (optional) ──                                   │
│ [+ ADD IAM CONDITION]                                            │
│   Title: [Allow weekdays only]                                  │
│   Condition type:                                               │
│     ○ Condition editor (visual builder)                        │
│     ○ Condition expression (CEL expression)                    │
│   Expression: request.time.getHours("America/New_York") >= 9   │
│               && request.time.getHours("America/New_York") <= 17│
│                                                                   │
│                              [SAVE]   [CANCEL]                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a Service Account

```
Console → IAM & Admin → Service Accounts → CREATE SERVICE ACCOUNT

┌─────────────────────────────────────────────────────────────────┐
│           CREATE SERVICE ACCOUNT (Step 1 of 3)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Service account details ──                                    │
│ Service account name: [backend-api-sa]                          │
│ ⚡ Human-readable name.                                          │
│                                                                   │
│ Service account ID: [backend-api-sa]                            │
│ ⚡ Auto-generated from name. This becomes the email:             │
│   backend-api-sa@PROJECT_ID.iam.gserviceaccount.com             │
│ ⚠️ Cannot be changed after creation!                             │
│                                                                   │
│ Service account description:                                    │
│ [Service account for backend API server]                        │
│                                                                   │
│                    [CREATE AND CONTINUE]                         │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│           STEP 2: Grant this service account access (optional)    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Grant roles to this service account on the PROJECT ──        │
│ Role: [Cloud SQL Client ▼]                                      │
│ [+ ADD ANOTHER ROLE]                                             │
│ Role: [Storage Object Admin ▼]                                  │
│                                                                   │
│ ⚡ These roles are granted ON this project.                      │
│   For cross-project access, grant roles in the other project.  │
│                                                                   │
│                          [CONTINUE]                             │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│           STEP 3: Grant users access to this SA (optional)        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Service account users role ──                                 │
│ Members: [devops-team@techcorp.com]                             │
│ ⚡ Who can impersonate (use) this service account.               │
│                                                                   │
│ ── Service account admins role ──                                │
│ Members: [admin@techcorp.com]                                   │
│ ⚡ Who can manage (edit/delete) this service account.            │
│                                                                   │
│                           [DONE]                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a Custom Role

```
Console → IAM & Admin → Roles → CREATE ROLE

┌─────────────────────────────────────────────────────────────────┐
│           CREATE ROLE                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Title ──                                                      │
│ Title: [Custom Cloud SQL Reader]                                │
│                                                                   │
│ ── Description ──                                                │
│ Description: [Read-only access to Cloud SQL instances and       │
│  databases, without ability to modify or delete]                │
│                                                                   │
│ ── ID ──                                                         │
│ ID: [customCloudSqlReader]                                      │
│ ⚡ Auto-generated. Used in IAM bindings:                         │
│   projects/PROJECT_ID/roles/customCloudSqlReader                │
│ ⚠️ Cannot be changed after creation!                             │
│                                                                   │
│ ── Role launch stage ──                                          │
│ Launch stage: [General Availability ▼]                          │
│   Options: Alpha → Beta → General Availability → Disabled       │
│ ⚡ Use Alpha/Beta for testing, GA when ready for production.     │
│                                                                   │
│ ── Permissions ──                                                │
│ [ADD PERMISSIONS]                                                │
│ Filter: [cloudsql] ← search by service name                    │
│                                                                   │
│ ☑ cloudsql.instances.get                                        │
│ ☑ cloudsql.instances.list                                       │
│ ☑ cloudsql.databases.get                                        │
│ ☑ cloudsql.databases.list                                       │
│ ☐ cloudsql.instances.create   (do NOT include for read-only)   │
│ ☐ cloudsql.instances.delete   (do NOT include for read-only)   │
│ ☐ cloudsql.instances.update   (do NOT include for read-only)   │
│                                                                   │
│ ⚡ Pick only the permissions you need (least privilege).         │
│   Tip: Start from a predefined role → "CREATE ROLE FROM ROLE"  │
│   to copy its permissions, then remove what you don't need.    │
│                                                                   │
│                              [CREATE]                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Managing Service Account Keys

```
Console → IAM & Admin → Service Accounts → Click SA → KEYS tab → ADD KEY

┌─────────────────────────────────────────────────────────────────┐
│           CREATE PRIVATE KEY                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Key type:                                                       │
│ ● JSON (recommended)                                           │
│ ○ P12 (legacy)                                                 │
│                                                                   │
│                     [CREATE]   [CANCEL]                         │
│                                                                   │
│ ⚠️ The JSON key file downloads automatically.                   │
│    This is the ONLY time you can download it!                   │
│    Store it securely — treat it like a password.                │
│                                                                   │
│ ⚡ BEST PRACTICE: Avoid using SA keys when possible!            │
│   ├── Use Workload Identity Federation instead (external)      │
│   ├── Use attached service accounts (VMs, Cloud Run)           │
│   └── Keys are a security risk (can be leaked/shared)          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Real-World Patterns

### Startup (5-10 Developers)

```
Setup:
├── Google Workspace (techcorp.com) → identities
├── 1 Organization
├── Folders: Production, Development
├── Groups:
│   ├── gcp-admins@techcorp.com → Organization Admin
│   ├── gcp-developers@techcorp.com → roles/editor on Dev folder
│   └── gcp-viewers@techcorp.com → roles/viewer on Prod folder
│
├── Service Accounts (per workload):
│   ├── backend-api-sa → Cloud SQL Client + Storage Object Admin
│   ├── frontend-deploy-sa → Cloud Run Developer
│   └── ci-cd-sa → Workload Identity Federation with GitHub
│
├── Org Policies:
│   ├── Restrict regions to asia-south1 (save costs)
│   └── Restrict domain (only techcorp.com can be granted access)
│
└── Monitoring:
    └── Cloud Audit Logs → Alert on Owner role grants
```

### Mid-Size Company (50-100 Developers)

```
Setup:
├── Cloud Identity Premium (MFA, device management)
├── Organization with nested folders:
│   ├── Platform (shared networking, monitoring)
│   ├── Production (per-team folders)
│   ├── Staging
│   ├── Development
│   └── Sandbox (auto-delete projects after 30 days)
│
├── Groups (many, fine-grained):
│   ├── team-backend-admins → Compute Admin on backend projects
│   ├── team-backend-devs → Compute Viewer + Cloud Run Developer
│   ├── team-data-admins → BigQuery Admin + Storage Admin
│   ├── team-data-analysts → BigQuery Data Viewer + Job User
│   ├── platform-team → Network Admin + Security Admin
│   └── security-auditors → Security Reviewer (org-wide)
│
├── Service Accounts:
│   ├── Per microservice (10+ SAs)
│   ├── All using Workload Identity (no keys)
│   ├── CI/CD via Workload Identity Federation
│   └── Weekly report: unused SAs → disable
│
├── Advanced controls:
│   ├── IAM Conditions: Contractors → expires in 90 days
│   ├── VPC Service Controls: Prod data cannot leave perimeter
│   ├── Org Policies: No public IPs in production
│   └── Deny Policies: No one can delete prod projects
│
└── Monitoring & Compliance:
    ├── IAM Recommender: Weekly review of suggestions
    ├── Security Command Center: SA key alerts
    ├── Monthly access review by team leads
    └── Quarterly security audit
```

### Enterprise (500+ Developers)

```
Setup:
├── Cloud Identity Premium + BeyondCorp Enterprise
├── Federated identity from corporate AD/Okta
│   ├── Users sync automatically
│   ├── Group membership drives GCP access
│   └── Offboarding = remove from AD = instant GCP revocation
│
├── Hierarchical groups with role-based access:
│   ├── L1: Platform → Full admin (5-10 people)
│   ├── L2: Team leads → Admin within their team's projects
│   ├── L3: Developers → Read + deploy within their team
│   ├── L4: Viewers → Read-only (QA, stakeholders)
│   └── L5: Emergency → Break-glass admin (time-limited)
│
├── Automation:
│   ├── Terraform manages all IAM bindings (no manual changes)
│   ├── Custom role per microservice (exact permissions)
│   ├── Project Factory auto-creates projects with correct IAM
│   └── Auto-cleanup: Unused SAs disabled after 90 days
│
├── Governance:
│   ├── Deny policies: Block data exfiltration APIs
│   ├── Org policies: 50+ constraints across org
│   ├── VPC SC: Multi-perimeter for different data classes
│   ├── Access Approval: Critical changes need approval
│   └── Key Access Justification: Why SA key was needed
│
└── Compliance:
    ├── SOC2, ISO 27001, PCI-DSS scope defined
    ├── Automated compliance checks via Security Command Center
    ├── All IAM changes in immutable audit log (exported to SIEM)
    └── Annual external penetration test includes IAM review
```

---

## Quick Reference: CLI Commands

```bash
# View IAM policy
gcloud projects get-iam-policy PROJECT_ID
gcloud organizations get-iam-policy ORG_ID

# Grant access
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="MEMBER" --role="ROLE"

# Revoke access
gcloud projects remove-iam-policy-binding PROJECT_ID \
  --member="MEMBER" --role="ROLE"

# Service Accounts
gcloud iam service-accounts create NAME --display-name="DISPLAY"
gcloud iam service-accounts list --project=PROJECT_ID
gcloud iam service-accounts keys list --iam-account=SA_EMAIL
gcloud iam service-accounts disable SA_EMAIL

# Custom Roles
gcloud iam roles create ROLE_ID --project=PROJECT --file=role.yaml
gcloud iam roles list --project=PROJECT
gcloud iam roles describe ROLE_ID --project=PROJECT

# Test permissions (troubleshooting)
gcloud asset analyze-iam-policy --organization=ORG_ID \
  --identity="user:john@company.com" --full-resource-name="..."

# IAM Recommender
gcloud recommender recommendations list \
  --recommender=google.iam.policy.Recommender \
  --project=PROJECT_ID --location=global
```

---

## What's Next?

In the next chapter, we'll cover GCP Billing & Cost Management — pricing models, committed use discounts, budgets, billing export to BigQuery, and cost optimization.

→ Next: [Chapter 5: Billing & Cost Management](05-billing.md)

---

*Last Updated: May 2026*
