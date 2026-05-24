# Chapter 35 — Infrastructure Manager

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Fundamentals](#part-1--fundamentals)
- [Part 2: Architecture & Core Concepts](#part-2--architecture--core-concepts)
- [Part 3: Creating Deployments](#part-3--creating-deployments)
- [Part 4: Previews](#part-4--previews)
- [Part 5: Revisions](#part-5--revisions)
- [Part 6: Service Accounts & IAM](#part-6--service-accounts--iam)
- [Part 7: Blueprint Configurations](#part-7--blueprint-configurations)
- [Part 8: Updating & Deleting Deployments](#part-8--updating--deleting-deployments)
- [Part 9: Console Walkthrough](#part-9--console-walkthrough)
- [Part 10: Error Handling & Troubleshooting](#part-10--error-handling--troubleshooting)
- [Part 11: Integration with CI/CD](#part-11--integration-with-cicd)
- [Part 12: Advanced Features](#part-12--advanced-features)
- [Part 13: Terraform Provider for Infrastructure Manager](#part-13--terraform-provider-for-infrastructure-manager)
- [Part 14: gcloud CLI Reference](#part-14--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud Infrastructure Manager (Infra Manager) is a fully managed service that automates the deployment and management of infrastructure using Terraform configurations. It provides a Google-native wrapper around Terraform—handling state, execution, previews, and revisions—without requiring you to manage Terraform CLI, backends, or CI/CD pipelines yourself. Think of it as "managed Terraform as a Service" on GCP.

---

## Part 1 — Fundamentals

### What Is Infrastructure Manager?

Infrastructure Manager lets you deploy and manage GCP resources using standard Terraform HCL configurations, but Google handles the execution environment, state storage, locking, and revision history. You submit Terraform configs, and Infra Manager runs `plan` and `apply` for you.

```
┌─────────────────────────────────────────────────────────────────┐
│            INFRASTRUCTURE MANAGER OVERVIEW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────┐         ┌──────────────────────────┐     │
│  │  You provide:     │         │  Infra Manager handles:   │     │
│  │                  │         │                          │     │
│  │  • Terraform HCL │────────▶│  • terraform init        │     │
│  │  • Variables     │         │  • terraform plan        │     │
│  │  • Input values  │         │  • terraform apply       │     │
│  │                  │         │  • State storage (GCS)   │     │
│  └──────────────────┘         │  • Locking               │     │
│                               │  • Revision history      │     │
│                               │  • IAM integration       │     │
│                               │  • Audit logging         │     │
│                               └──────────────────────────┘     │
│                                                                   │
│  Key Concepts:                                                    │
│  • Deployment = a managed Terraform workspace                    │
│  • Revision = a specific version of a deployment                 │
│  • Preview = a dry-run showing planned changes                   │
│  • Blueprint = Terraform configuration source                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Infrastructure Manager vs Self-Managed Terraform

| Feature | Infrastructure Manager | Self-Managed Terraform |
|---------|----------------------|----------------------|
| Execution | Google-managed | Your CI/CD or local |
| State storage | Automatic (GCS, managed) | You configure backend |
| State locking | Automatic | You configure |
| Revision history | Built-in | Git history + state versions |
| Preview/Plan | API-based with results stored | CLI `terraform plan` |
| Approval workflow | IAM-based | CI/CD pipeline gates |
| Provider versions | Managed, updated | You pin and update |
| Custom providers | Limited | Full flexibility |
| Complex modules | Supported | Full flexibility |
| Multi-cloud | GCP only | Any provider |
| Cost | Free (pay for resources) | Free (pay for resources) |
| Learning curve | Lower (no backend setup) | Higher (full Terraform knowledge) |

### Cross-Cloud Comparison

| Feature | GCP Infrastructure Manager | AWS CloudFormation | Azure Deployment Stacks |
|---------|---------------------------|-------------------|------------------------|
| IaC language | Terraform HCL | JSON/YAML (CFN) | Bicep/ARM JSON |
| Execution | Managed | Managed | Managed |
| State | GCS (managed) | AWS-managed | Azure-managed |
| Preview | Preview resource | Change sets | What-if |
| Revisions | Built-in history | Stack versions | Deployment history |
| Drift | Planned | Drift detection | No built-in |
| Rollback | Redeploy previous revision | Auto-rollback | Redeploy |
| Nested/Modules | Terraform modules | Nested stacks | Module specs |
| Deletion protection | Lock deployment | Termination protection | Deny settings |

### When to Use Infrastructure Manager

| Scenario | Recommendation |
|----------|---------------|
| Teams new to Terraform, want managed experience | Infrastructure Manager |
| Need simple deployment without CI/CD setup | Infrastructure Manager |
| Want Google-managed state and execution | Infrastructure Manager |
| Need multi-cloud (AWS + GCP + Azure) | Self-managed Terraform |
| Complex custom providers or provisioners | Self-managed Terraform |
| Already have mature Terraform CI/CD | Continue self-managed |
| Want tight GCP IAM integration for approvals | Infrastructure Manager |
| Need programmatic infrastructure deployment (API) | Infrastructure Manager |

### Pricing

Infrastructure Manager itself is **free**. You only pay for:
- Resources created by the deployment
- Cloud Storage for state/artifacts (minimal)
- Cloud Build execution time (for running Terraform)

---

## Part 2 — Architecture & Core Concepts

### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│          INFRASTRUCTURE MANAGER ARCHITECTURE                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  User / CI System                                                  │
│       │                                                            │
│       │  gcloud infra-manager deployments apply                    │
│       │  (or REST API / Terraform provider)                        │
│       ▼                                                            │
│  ┌─────────────────────────────────────────────────┐              │
│  │         Infrastructure Manager Service           │              │
│  │                                                 │              │
│  │  ┌───────────────┐  ┌───────────────────────┐ │              │
│  │  │  Deployment   │  │  Revision Manager      │ │              │
│  │  │  Controller   │  │  (tracks all versions) │ │              │
│  │  └───────┬───────┘  └───────────────────────┘ │              │
│  │          │                                      │              │
│  │          ▼                                      │              │
│  │  ┌───────────────┐                             │              │
│  │  │ Cloud Build   │  (execution environment)    │              │
│  │  │ • terraform   │                             │              │
│  │  │   init/plan/  │                             │              │
│  │  │   apply       │                             │              │
│  │  └───────┬───────┘                             │              │
│  │          │                                      │              │
│  └──────────┼──────────────────────────────────────┘              │
│             │                                                      │
│             ▼                                                      │
│  ┌────────────────────────────────────────────┐                   │
│  │           GCP Resource APIs                 │                   │
│  │  Compute │ GKE │ Cloud SQL │ IAM │ ...     │                   │
│  └────────────────────────────────────────────┘                   │
│                                                                    │
│  State stored in:                                                  │
│  ┌────────────────────────────────────────────┐                   │
│  │  GCS Bucket (auto-managed, locked)          │                   │
│  └────────────────────────────────────────────┘                   │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Deployment** | A named, managed Terraform workspace. Contains revisions, state, and configuration. |
| **Revision** | A specific point-in-time version of a deployment. Created each time you apply changes. |
| **Preview** | A dry-run that shows what changes would occur without applying them. |
| **Blueprint** | The Terraform configuration source (Git repo, GCS path, or local upload). |
| **Service Account** | The identity used to create/manage resources (execution SA). |
| **Annotations** | User-defined metadata on deployments. |
| **Labels** | Key-value pairs for organizing deployments. |
| **Lock** | Prevents modifications to a deployment. |

### Deployment Lifecycle

```
┌──────────────────────────────────────────────────────────────┐
│                 DEPLOYMENT LIFECYCLE                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  CREATE DEPLOYMENT                                             │
│       │                                                        │
│       ▼                                                        │
│  ┌──────────────────────────────────────┐                     │
│  │  Revision 1 (initial)                │                     │
│  │  • Fetch blueprint (Terraform HCL)   │                     │
│  │  • terraform init                    │                     │
│  │  • terraform plan                    │                     │
│  │  • terraform apply                   │                     │
│  │  • Store state                       │                     │
│  │  Status: ACTIVE                      │                     │
│  └────────────────┬─────────────────────┘                     │
│                   │                                            │
│                   ▼  (update deployment)                      │
│  ┌──────────────────────────────────────┐                     │
│  │  Revision 2 (update)                 │                     │
│  │  • Fetch updated blueprint           │                     │
│  │  • terraform plan (diff from state)  │                     │
│  │  • terraform apply (incremental)     │                     │
│  │  • Update state                      │                     │
│  │  Status: ACTIVE                      │                     │
│  └────────────────┬─────────────────────┘                     │
│                   │                                            │
│                   ▼  (update again)                           │
│  ┌──────────────────────────────────────┐                     │
│  │  Revision N ...                      │                     │
│  └────────────────┬─────────────────────┘                     │
│                   │                                            │
│                   ▼  (delete deployment)                      │
│  ┌──────────────────────────────────────┐                     │
│  │  terraform destroy                   │                     │
│  │  • Removes all managed resources     │                     │
│  │  • Deletes state                     │                     │
│  │  Status: DELETED                     │                     │
│  └──────────────────────────────────────┘                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Deployment States

| State | Description |
|-------|-------------|
| `CREATING` | Initial deployment in progress |
| `ACTIVE` | Deployment succeeded, resources exist |
| `UPDATING` | Applying changes to existing deployment |
| `DELETING` | Destroying resources |
| `DELETED` | All resources destroyed |
| `FAILED` | Last operation failed |
| `LOCKED` | Deployment is locked, no changes allowed |

---

## Part 3 — Creating Deployments

### Blueprint Sources

```
┌──────────────────────────────────────────────────────────────┐
│                  BLUEPRINT SOURCES                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │   Git Repo     │  │   GCS Bucket   │  │  Local Upload  │ │
│  ├────────────────┤  ├────────────────┤  ├────────────────┤ │
│  │ • GitHub       │  │ • .tf files    │  │ • gcloud CLI   │ │
│  │ • GitLab       │  │   in bucket    │  │   --local-     │ │
│  │ • Cloud Source │  │ • Zipped or    │  │   source       │ │
│  │   Repos        │  │   directory    │  │ • Direct       │ │
│  │ • Bitbucket    │  │               │  │   upload       │ │
│  │ • Any Git URL  │  │               │  │               │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating from Git Repository

```bash
# Create deployment from GitHub repo
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --git-source-repo="https://github.com/my-org/infra-configs" \
    --git-source-directory="environments/prod" \
    --git-source-ref="main" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com \
    --input-values="project_id=my-project,region=us-central1,environment=prod"
```

### Creating from GCS

```bash
# Upload Terraform configs to GCS first
gsutil cp -r ./terraform/ gs://my-project-infra-blueprints/prod/

# Create deployment from GCS
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --gcs-source="gs://my-project-infra-blueprints/prod" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com \
    --input-values="project_id=my-project,region=us-central1"
```

### Creating from Local Source

```bash
# Create deployment from local directory
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --local-source="./terraform/environments/prod" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com \
    --input-values="project_id=my-project,region=us-central1,environment=prod"
```

### Input Values

```bash
# Pass variables via --input-values (comma-separated key=value)
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --git-source-repo="https://github.com/my-org/infra" \
    --git-source-directory="prod" \
    --git-source-ref="v1.2.0" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com \
    --input-values="project_id=my-project,region=us-central1,machine_type=e2-standard-4,node_count=3"

# Or use a tfvars file in the blueprint directory
# Infrastructure Manager automatically picks up terraform.tfvars
```

### Labels and Annotations

```bash
# Create with labels
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --local-source="./terraform/prod" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com \
    --labels="env=prod,team=platform,cost-center=engineering" \
    --annotations="owner=platform-team,ticket=INFRA-1234"
```

---

## Part 4 — Previews

### What Is a Preview?

A preview runs `terraform plan` against your blueprint and stores the results. It shows what resources would be created, modified, or destroyed—without making any changes.

```
┌──────────────────────────────────────────────────────────────┐
│                    PREVIEW WORKFLOW                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────┐                                           │
│  │ Create Preview │                                           │
│  └───────┬────────┘                                           │
│          │                                                     │
│          ▼                                                     │
│  ┌──────────────────────────────────────┐                    │
│  │  Infrastructure Manager:             │                    │
│  │  • Fetches blueprint                 │                    │
│  │  • Runs terraform init               │                    │
│  │  • Runs terraform plan               │                    │
│  │  • Stores plan artifacts             │                    │
│  └───────┬──────────────────────────────┘                    │
│          │                                                     │
│          ▼                                                     │
│  ┌──────────────────────────────────────┐                    │
│  │  Preview Results:                    │                    │
│  │  • Resources to create: 5           │                    │
│  │  • Resources to update: 2           │                    │
│  │  • Resources to delete: 1           │                    │
│  │  • Full plan output available       │                    │
│  └───────┬──────────────────────────────┘                    │
│          │                                                     │
│     ┌────┴────┐                                               │
│     ▼         ▼                                               │
│  ┌──────┐  ┌──────────┐                                     │
│  │Apply │  │ Discard  │                                     │
│  │(go)  │  │ (cancel) │                                     │
│  └──────┘  └──────────┘                                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating a Preview

```bash
# Create a preview for a new deployment
gcloud infra-manager previews create my-preview \
    --location=us-central1 \
    --deployment=my-deployment \
    --git-source-repo="https://github.com/my-org/infra" \
    --git-source-directory="prod" \
    --git-source-ref="feature/add-redis" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com \
    --input-values="project_id=my-project,region=us-central1"

# Create preview from local source
gcloud infra-manager previews create my-preview \
    --location=us-central1 \
    --deployment=my-deployment \
    --local-source="./terraform/prod" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com
```

### Viewing Preview Results

```bash
# Describe the preview (see plan summary)
gcloud infra-manager previews describe my-preview \
    --location=us-central1

# Export preview results
gcloud infra-manager previews export my-preview \
    --location=us-central1 \
    --output-file=plan-output.json
```

### Applying from a Preview

```bash
# Apply changes based on a preview (skip re-planning)
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --preview=my-preview \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com
```

### Deleting a Preview

```bash
# Delete/discard a preview without applying
gcloud infra-manager previews delete my-preview \
    --location=us-central1
```

### Preview States

| State | Description |
|-------|-------------|
| `CREATING` | Plan is being executed |
| `SUCCEEDED` | Plan completed, results available |
| `FAILED` | Plan failed (config errors, auth issues) |
| `STALE` | Deployment changed since preview was created |
| `DELETING` | Preview being removed |

---

## Part 5 — Revisions

### What Are Revisions?

Every time you apply changes to a deployment, a new revision is created. Revisions provide a complete history of your infrastructure changes.

```
┌──────────────────────────────────────────────────────────────┐
│                    REVISION HISTORY                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Deployment: "prod-infrastructure"                            │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Revision 1 (2024-01-15)                             │    │
│  │  Action: CREATE                                      │    │
│  │  Resources: VPC + 3 subnets + firewall               │    │
│  │  Status: SUCCEEDED → superseded                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Revision 2 (2024-02-01)                             │    │
│  │  Action: UPDATE                                      │    │
│  │  Changes: Added GKE cluster + node pool              │    │
│  │  Status: SUCCEEDED → superseded                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Revision 3 (2024-03-10)  ← CURRENT                 │    │
│  │  Action: UPDATE                                      │    │
│  │  Changes: Scaled node pool, added Cloud SQL          │    │
│  │  Status: ACTIVE                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  To rollback: Re-apply the blueprint from Revision 1 or 2    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Listing Revisions

```bash
# List all revisions for a deployment
gcloud infra-manager revisions list \
    --deployment=my-deployment \
    --location=us-central1

# Describe a specific revision
gcloud infra-manager revisions describe revision-abc123 \
    --deployment=my-deployment \
    --location=us-central1

# Get revision with full details
gcloud infra-manager revisions describe revision-abc123 \
    --deployment=my-deployment \
    --location=us-central1 \
    --format=json
```

### Revision Details

Each revision contains:

| Field | Description |
|-------|-------------|
| `name` | Unique revision identifier |
| `createTime` | When the revision was created |
| `action` | CREATE or UPDATE |
| `state` | APPLYING, ACTIVE, FAILED |
| `stateDetail` | Detailed status message |
| `blueprint` | Source configuration used |
| `inputValues` | Variables passed |
| `serviceAccount` | SA used for execution |
| `logs` | Cloud Build logs URL |
| `tfErrors` | Terraform error details (if failed) |
| `applyResults` | Resource changes summary |

### Rolling Back to a Previous Revision

There's no built-in "rollback" command. Instead, re-apply the previous blueprint:

```bash
# Option 1: Re-apply from the same Git ref as a previous revision
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --git-source-repo="https://github.com/my-org/infra" \
    --git-source-directory="prod" \
    --git-source-ref="v1.0.0" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com

# Option 2: Use Git tags for versioning
# Tag v1.0.0 → initial infra
# Tag v1.1.0 → added GKE
# Tag v1.2.0 → current (broken)
# Rollback: deploy from v1.1.0
```

---

## Part 6 — Service Accounts & IAM

### Service Account Requirements

```
┌──────────────────────────────────────────────────────────────┐
│           SERVICE ACCOUNT ARCHITECTURE                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────┐            │
│  │  Infrastructure Manager Service Agent         │            │
│  │  (service-PROJ_NUM@gcp-sa-config.iam.        │            │
│  │   gserviceaccount.com)                        │            │
│  │                                              │            │
│  │  Auto-created. Needs:                         │            │
│  │  • roles/iam.serviceAccountUser on exec SA   │            │
│  │  • Access to blueprint source (GCS/Git)      │            │
│  └──────────────────────────────────────────────┘            │
│                                                                │
│  ┌──────────────────────────────────────────────┐            │
│  │  Execution Service Account (you create)       │            │
│  │  (infra-mgr@PROJECT.iam.gserviceaccount.com) │            │
│  │                                              │            │
│  │  Needs permissions for ALL resources          │            │
│  │  Terraform will create/modify/delete:         │            │
│  │  • roles/compute.admin                        │            │
│  │  • roles/container.admin                      │            │
│  │  • roles/iam.serviceAccountAdmin              │            │
│  │  • roles/storage.admin                        │            │
│  │  • (whatever your Terraform creates)          │            │
│  └──────────────────────────────────────────────┘            │
│                                                                │
│  ┌──────────────────────────────────────────────┐            │
│  │  User / CI Service Account                    │            │
│  │  (the one calling infra-manager API)          │            │
│  │                                              │            │
│  │  Needs:                                       │            │
│  │  • roles/config.admin (full control)          │            │
│  │  OR                                           │            │
│  │  • roles/config.editor (create/update)        │            │
│  │  • roles/config.viewer (read-only)            │            │
│  └──────────────────────────────────────────────┘            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Setting Up Service Accounts

```bash
# Create execution service account
gcloud iam service-accounts create infra-mgr \
    --display-name="Infrastructure Manager Execution SA" \
    --project=my-project

# Grant resource creation permissions
ROLES=(
  "roles/compute.admin"
  "roles/container.admin"
  "roles/iam.serviceAccountAdmin"
  "roles/iam.serviceAccountUser"
  "roles/resourcemanager.projectIamAdmin"
  "roles/storage.admin"
  "roles/cloudsql.admin"
  "roles/run.admin"
  "roles/secretmanager.admin"
  "roles/dns.admin"
  "roles/logging.admin"
)

for ROLE in "${ROLES[@]}"; do
  gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:infra-mgr@my-project.iam.gserviceaccount.com" \
    --role="$ROLE"
done

# Allow Infrastructure Manager service agent to use the execution SA
PROJECT_NUM=$(gcloud projects describe my-project --format='value(projectNumber)')

gcloud iam service-accounts add-iam-policy-binding \
    infra-mgr@my-project.iam.gserviceaccount.com \
    --member="serviceAccount:service-${PROJECT_NUM}@gcp-sa-config.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountUser"
```

### IAM Roles for Infrastructure Manager

| Role | Description | Use Case |
|------|-------------|----------|
| `roles/config.admin` | Full control over deployments, previews | Platform admins |
| `roles/config.editor` | Create/update/delete deployments | Developers, CI/CD |
| `roles/config.viewer` | Read-only access to deployments | Auditors |
| `roles/config.agent` | Service agent role (auto-granted) | Service agent |

### Granting User Access

```bash
# Allow a user to manage deployments
gcloud projects add-iam-policy-binding my-project \
    --member="user:developer@example.com" \
    --role="roles/config.editor"

# Allow a CI/CD SA to manage deployments
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:ci-cd@my-project.iam.gserviceaccount.com" \
    --role="roles/config.admin"

# Read-only for auditors
gcloud projects add-iam-policy-binding my-project \
    --member="group:auditors@example.com" \
    --role="roles/config.viewer"
```

---

## Part 7 — Blueprint Configurations

### Standard Terraform Structure for Infra Manager

```
┌──────────────────────────────────────────────────────────────┐
│         BLUEPRINT STRUCTURE (Terraform HCL)                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  my-blueprint/                                                 │
│  ├── main.tf              (resource definitions)              │
│  ├── variables.tf         (input variables)                   │
│  ├── outputs.tf           (deployment outputs)                │
│  ├── versions.tf          (provider requirements)             │
│  ├── terraform.tfvars     (default values — optional)         │
│  └── modules/             (local modules — optional)          │
│      └── network/                                             │
│          ├── main.tf                                          │
│          ├── variables.tf                                     │
│          └── outputs.tf                                       │
│                                                                │
│  Rules:                                                        │
│  • Standard Terraform HCL syntax                              │
│  • Do NOT include backend block (Infra Manager manages it)   │
│  • Provider block optional (defaults to google provider)      │
│  • Terraform Registry modules supported                       │
│  • Google provider version managed by Infra Manager          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Example Blueprint — Web Application Infrastructure

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  # NOTE: Do NOT specify backend — Infra Manager handles it
}

# variables.tf
variable "project_id" {
  type        = string
  description = "GCP project ID"
}

variable "region" {
  type        = string
  description = "Deployment region"
  default     = "us-central1"
}

variable "environment" {
  type        = string
  description = "Environment (dev/staging/prod)"
}

variable "min_instances" {
  type        = number
  description = "Minimum Cloud Run instances"
  default     = 0
}

variable "container_image" {
  type        = string
  description = "Container image to deploy"
}
```

```hcl
# main.tf
provider "google" {
  project = var.project_id
  region  = var.region
}

# VPC
resource "google_compute_network" "main" {
  name                    = "${var.environment}-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "app" {
  name          = "${var.environment}-app"
  ip_cidr_range = "10.0.1.0/24"
  region        = var.region
  network       = google_compute_network.main.id
}

# Cloud Run service
resource "google_cloud_run_v2_service" "app" {
  name     = "${var.environment}-app"
  location = var.region

  template {
    scaling {
      min_instance_count = var.min_instances
      max_instance_count = 100
    }
    containers {
      image = var.container_image
      ports {
        container_port = 8080
      }
      env {
        name  = "ENVIRONMENT"
        value = var.environment
      }
    }
  }
}

# Make Cloud Run publicly accessible
resource "google_cloud_run_v2_service_iam_member" "public" {
  name     = google_cloud_run_v2_service.app.name
  location = var.region
  role     = "roles/run.invoker"
  member   = "allUsers"
}
```

```hcl
# outputs.tf
output "service_url" {
  description = "Cloud Run service URL"
  value       = google_cloud_run_v2_service.app.uri
}

output "network_id" {
  description = "VPC network ID"
  value       = google_compute_network.main.id
}
```

### Using Terraform Registry Modules

```hcl
# Blueprint using Google Foundation Toolkit modules
module "vpc" {
  source  = "terraform-google-modules/network/google"
  version = "~> 9.0"

  project_id   = var.project_id
  network_name = "${var.environment}-vpc"

  subnets = [
    {
      subnet_name   = "app"
      subnet_ip     = "10.0.1.0/24"
      subnet_region = var.region
    }
  ]
}

module "gke" {
  source  = "terraform-google-modules/kubernetes-engine/google"
  version = "~> 30.0"

  project_id = var.project_id
  name       = "${var.environment}-cluster"
  region     = var.region
  network    = module.vpc.network_name
  subnetwork = module.vpc.subnets_names[0]
}
```

---

## Part 8 — Updating & Deleting Deployments

### Updating an Existing Deployment

```bash
# Update with new configuration (same deployment name = update)
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --git-source-repo="https://github.com/my-org/infra" \
    --git-source-directory="prod" \
    --git-source-ref="v1.2.0" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com \
    --input-values="project_id=my-project,region=us-central1,node_count=5"

# Update with changed variables only (same source)
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --git-source-repo="https://github.com/my-org/infra" \
    --git-source-directory="prod" \
    --git-source-ref="main" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com \
    --input-values="project_id=my-project,region=us-central1,machine_type=e2-standard-8"
```

### Deleting a Deployment

```bash
# Delete deployment (destroys all managed resources)
gcloud infra-manager deployments delete my-deployment \
    --location=us-central1

# Force delete (skip confirmation)
gcloud infra-manager deployments delete my-deployment \
    --location=us-central1 \
    --quiet
```

> **Warning**: Deleting a deployment runs `terraform destroy`, which removes all resources managed by that deployment.

### Locking a Deployment

```bash
# Lock deployment to prevent changes
gcloud infra-manager deployments lock my-deployment \
    --location=us-central1

# Unlock deployment
gcloud infra-manager deployments unlock my-deployment \
    --location=us-central1
```

### Exporting State

```bash
# Export Terraform state from a deployment
gcloud infra-manager deployments export-statefile my-deployment \
    --location=us-central1 \
    --output-file=terraform.tfstate

# Export to migrate to self-managed Terraform
gcloud infra-manager deployments export-statefile my-deployment \
    --location=us-central1 \
    --output-file=./self-managed/terraform.tfstate
```

### Importing State

```bash
# Import existing state into a deployment
gcloud infra-manager deployments import-statefile my-deployment \
    --location=us-central1 \
    --input-file=terraform.tfstate
```

---

## Part 9 — Console Walkthrough

### Navigating to Infrastructure Manager

```
┌──────────────────────────────────────────────────────────────┐
│              CONSOLE WALKTHROUGH                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Go to: Console → Infrastructure Manager                   │
│     (or search "Infrastructure Manager" in console)           │
│                                                                │
│  2. Deployments List View:                                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Name          │ Location    │ Status │ Last Updated │    │
│  │  prod-infra    │ us-central1 │ ACTIVE │ 2024-03-10  │    │
│  │  staging-infra │ us-central1 │ ACTIVE │ 2024-03-08  │    │
│  │  dev-infra     │ us-central1 │ ACTIVE │ 2024-03-12  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  3. Deployment Detail View:                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Deployment: prod-infra                              │    │
│  │  ┌──────────────────────────────────────────────┐   │    │
│  │  │  Overview │ Revisions │ Resources │ Outputs  │   │    │
│  │  └──────────────────────────────────────────────┘   │    │
│  │                                                      │    │
│  │  • Status: ACTIVE                                   │    │
│  │  • Current Revision: revision-3                     │    │
│  │  • Service Account: infra-mgr@proj.iam...          │    │
│  │  • Blueprint: github.com/org/repo/prod              │    │
│  │  • Resources: 12 managed                            │    │
│  │  • Last applied: 2024-03-10 14:32 UTC              │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  4. Create Deployment:                                         │
│     [+ CREATE DEPLOYMENT] button                              │
│     → Choose source (Git/GCS/Upload)                          │
│     → Configure variables                                      │
│     → Select service account                                   │
│     → Review and create                                        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Console Features

| Feature | Description |
|---------|-------------|
| Deployment list | View all deployments with status/filters |
| Revision timeline | See history of all changes |
| Resource view | See all Terraform-managed resources |
| Output values | View Terraform outputs |
| Logs link | Direct link to Cloud Build execution logs |
| Preview | Create and view previews from console |
| Lock/Unlock | Lock deployments from console |
| Delete | Destroy deployment from console |

---

## Part 10 — Error Handling & Troubleshooting

### Common Errors

```
┌──────────────────────────────────────────────────────────────┐
│            COMMON ERRORS & SOLUTIONS                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Error: Permission denied                                      │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Cause: Execution SA lacks permissions                 │    │
│  │ Fix: Grant required roles to execution SA            │    │
│  │ Check: gcloud iam service-accounts get-iam-policy    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Error: Service agent cannot act as service account            │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Cause: Config service agent needs actAs permission   │    │
│  │ Fix: Grant iam.serviceAccountUser to service agent   │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Error: Blueprint source not accessible                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Cause: Git repo private or GCS bucket inaccessible    │    │
│  │ Fix: Grant source.repos.get or storage.objects.get   │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Error: Terraform configuration invalid                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Cause: HCL syntax error or missing variable          │    │
│  │ Fix: Run terraform validate locally first            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Error: State lock                                             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Cause: Previous operation didn't release lock        │    │
│  │ Fix: Wait for timeout or unlock via console/CLI      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Viewing Logs

```bash
# Get Cloud Build logs for a revision
gcloud infra-manager revisions describe REVISION_ID \
    --deployment=my-deployment \
    --location=us-central1 \
    --format="value(build)"

# Then view the build
gcloud builds describe BUILD_ID --region=us-central1

# Or view logs directly
gcloud builds log BUILD_ID --region=us-central1

# Filter Infrastructure Manager logs
gcloud logging read \
    'resource.type="infra_manager_deployment"' \
    --limit=50 \
    --project=my-project
```

### Handling Failed Deployments

```bash
# Check what failed
gcloud infra-manager deployments describe my-deployment \
    --location=us-central1 \
    --format="yaml(state, stateDetail, errorCode)"

# Get the latest revision's error
gcloud infra-manager revisions list \
    --deployment=my-deployment \
    --location=us-central1 \
    --limit=1 \
    --format="yaml(state, tfErrors)"

# Fix and re-apply
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --local-source="./fixed-terraform" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com
```

### Recovering from Stuck State

```bash
# Export state to fix locally
gcloud infra-manager deployments export-statefile my-deployment \
    --location=us-central1 \
    --output-file=state.tfstate

# Fix state locally (remove problematic resource)
terraform state rm -state=state.tfstate google_compute_instance.broken

# Re-import fixed state
gcloud infra-manager deployments import-statefile my-deployment \
    --location=us-central1 \
    --input-file=state.tfstate

# Re-apply
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --local-source="./terraform" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com
```

---

## Part 11 — Integration with CI/CD

### Cloud Build Integration

```yaml
# cloudbuild.yaml — Deploy infrastructure via Infra Manager
steps:
  # Validate Terraform locally first
  - id: 'validate'
    name: 'hashicorp/terraform:1.7'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        cd environments/${_ENV}
        terraform init -backend=false
        terraform validate

  # Create preview
  - id: 'preview'
    name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'infra-manager'
      - 'previews'
      - 'create'
      - 'preview-${SHORT_SHA}'
      - '--location=${_REGION}'
      - '--deployment=${_DEPLOYMENT}'
      - '--local-source=environments/${_ENV}'
      - '--service-account=projects/${PROJECT_ID}/serviceAccounts/infra-mgr@${PROJECT_ID}.iam.gserviceaccount.com'
      - '--input-values=project_id=${PROJECT_ID},region=${_REGION},environment=${_ENV}'

  # Apply (only on main branch)
  - id: 'apply'
    name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'infra-manager'
      - 'deployments'
      - 'apply'
      - '${_DEPLOYMENT}'
      - '--location=${_REGION}'
      - '--local-source=environments/${_ENV}'
      - '--service-account=projects/${PROJECT_ID}/serviceAccounts/infra-mgr@${PROJECT_ID}.iam.gserviceaccount.com'
      - '--input-values=project_id=${PROJECT_ID},region=${_REGION},environment=${_ENV}'

substitutions:
  _ENV: 'prod'
  _REGION: 'us-central1'
  _DEPLOYMENT: 'prod-infrastructure'
```

### GitHub Actions Integration

```yaml
# .github/workflows/infra-manager.yml
name: Infrastructure Manager Deploy
on:
  push:
    branches: [main]
    paths: ['terraform/**']
  pull_request:
    paths: ['terraform/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.WIF_PROVIDER }}
          service_account: ${{ vars.CI_SERVICE_ACCOUNT }}

      - uses: google-github-actions/setup-gcloud@v2

      - name: Preview (PRs only)
        if: github.event_name == 'pull_request'
        run: |
          gcloud infra-manager previews create preview-${{ github.event.pull_request.number }} \
            --location=us-central1 \
            --deployment=prod-infrastructure \
            --local-source=./terraform/environments/prod \
            --service-account=projects/${{ vars.PROJECT_ID }}/serviceAccounts/infra-mgr@${{ vars.PROJECT_ID }}.iam.gserviceaccount.com \
            --input-values="project_id=${{ vars.PROJECT_ID }},region=us-central1"

      - name: Apply (main only)
        if: github.ref == 'refs/heads/main'
        run: |
          gcloud infra-manager deployments apply prod-infrastructure \
            --location=us-central1 \
            --local-source=./terraform/environments/prod \
            --service-account=projects/${{ vars.PROJECT_ID }}/serviceAccounts/infra-mgr@${{ vars.PROJECT_ID }}.iam.gserviceaccount.com \
            --input-values="project_id=${{ vars.PROJECT_ID }},region=us-central1"
```

### Automated Drift Detection

```yaml
# Cloud Scheduler + Cloud Function for drift detection
# Schedule preview creation and compare with current state

# cloudbuild-drift.yaml — Runs on schedule
steps:
  - id: 'drift-check'
    name: 'gcr.io/cloud-builders/gcloud'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        # Create a preview to detect drift
        gcloud infra-manager previews create drift-check-$(date +%Y%m%d) \
          --location=us-central1 \
          --deployment=prod-infrastructure \
          --git-source-repo="https://github.com/my-org/infra" \
          --git-source-directory="prod" \
          --git-source-ref="main" \
          --service-account=projects/${PROJECT_ID}/serviceAccounts/infra-mgr@${PROJECT_ID}.iam.gserviceaccount.com
        
        # Check if preview shows changes (drift detected)
        PREVIEW=$(gcloud infra-manager previews describe drift-check-$(date +%Y%m%d) \
          --location=us-central1 --format=json)
        
        echo "$PREVIEW" | jq '.applyResults'
```

---

## Part 12 — Advanced Features

### Quota & Limits

| Limit | Value |
|-------|-------|
| Deployments per project per region | 1,000 |
| Revisions per deployment | 1,000 |
| Concurrent operations per project | 10 |
| Blueprint size (compressed) | 10 MB |
| Input values size | 512 KB |
| Terraform execution timeout | 2 hours |
| Preview retention | 7 days |

### Annotations for Metadata

```bash
# Add annotations for tracking
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --local-source="./terraform" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com \
    --annotations="jira-ticket=INFRA-456,approved-by=jane@example.com,change-window=2024-03-15"
```

### Working with Outputs

```bash
# Get deployment outputs (Terraform outputs)
gcloud infra-manager deployments describe my-deployment \
    --location=us-central1 \
    --format="yaml(outputs)"

# Get specific output value
gcloud infra-manager deployments describe my-deployment \
    --location=us-central1 \
    --format="value(outputs.service_url.value)"
```

### Cross-Project Deployments

```bash
# Deploy resources in a different project
# The execution SA needs permissions in the target project

# Grant execution SA roles in target project
gcloud projects add-iam-policy-binding target-project \
    --member="serviceAccount:infra-mgr@management-project.iam.gserviceaccount.com" \
    --role="roles/compute.admin"

# Deploy specifying target project in variables
gcloud infra-manager deployments apply cross-project-deploy \
    --location=us-central1 \
    --project=management-project \
    --local-source="./terraform" \
    --service-account=projects/management-project/serviceAccounts/infra-mgr@management-project.iam.gserviceaccount.com \
    --input-values="project_id=target-project,region=us-central1"
```

### Using with Private Git Repos

```bash
# For Cloud Source Repos — automatic authentication via service agent

# For GitHub/GitLab — use Secret Manager for tokens
# 1. Store token in Secret Manager
echo -n "ghp_YOUR_TOKEN" | gcloud secrets create github-token \
    --data-file=- --project=my-project

# 2. Grant service agent access to secret
PROJECT_NUM=$(gcloud projects describe my-project --format='value(projectNumber)')
gcloud secrets add-iam-policy-binding github-token \
    --member="serviceAccount:service-${PROJECT_NUM}@gcp-sa-config.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"

# 3. Use authenticated Git URL
gcloud infra-manager deployments apply my-deployment \
    --location=us-central1 \
    --git-source-repo="https://github.com/my-org/private-infra" \
    --git-source-directory="prod" \
    --git-source-ref="main" \
    --service-account=projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com
```

---

## Part 13 — Terraform Provider for Infrastructure Manager

### Managing Infra Manager with Terraform

You can manage Infrastructure Manager deployments themselves using the Google Terraform provider:

```hcl
# Deploy infrastructure using Infra Manager... managed by Terraform
# (meta-IaC: Terraform managing Terraform deployments)

resource "google_infra_manager_deployment" "prod" {
  deployment_id = "prod-infrastructure"
  location      = "us-central1"
  project       = var.project_id

  labels = {
    environment = "prod"
    team        = "platform"
  }

  annotations = {
    "owner" = "platform-team"
  }

  service_account = "projects/${var.project_id}/serviceAccounts/infra-mgr@${var.project_id}.iam.gserviceaccount.com"

  terraform_blueprint {
    git_source {
      repo      = "https://github.com/my-org/infra-configs"
      directory = "environments/prod"
      ref       = "main"
    }

    input_values = {
      project_id  = var.project_id
      region      = "us-central1"
      environment = "prod"
    }
  }
}
```

### Terraform Provider — GCS Source

```hcl
resource "google_infra_manager_deployment" "staging" {
  deployment_id = "staging-infrastructure"
  location      = "us-central1"
  project       = var.project_id

  service_account = "projects/${var.project_id}/serviceAccounts/infra-mgr@${var.project_id}.iam.gserviceaccount.com"

  terraform_blueprint {
    gcs_source = "gs://${var.project_id}-blueprints/staging"

    input_values = {
      project_id  = var.project_id
      region      = "us-central1"
      environment = "staging"
    }
  }
}
```

### Terraform Provider — Preview Resource

```hcl
resource "google_infra_manager_preview" "prod_preview" {
  preview_id    = "prod-preview-${formatdate("YYYYMMDD", timestamp())}"
  location      = "us-central1"
  project       = var.project_id
  deployment    = google_infra_manager_deployment.prod.id

  service_account = "projects/${var.project_id}/serviceAccounts/infra-mgr@${var.project_id}.iam.gserviceaccount.com"

  terraform_blueprint {
    git_source {
      repo      = "https://github.com/my-org/infra-configs"
      directory = "environments/prod"
      ref       = "feature/new-changes"
    }

    input_values = {
      project_id  = var.project_id
      region      = "us-central1"
      environment = "prod"
    }
  }
}
```

### Full Terraform Setup for Infra Manager

```hcl
# Enable API
resource "google_project_service" "infra_manager" {
  service = "config.googleapis.com"
  project = var.project_id
}

# Execution Service Account
resource "google_service_account" "infra_mgr" {
  account_id   = "infra-mgr"
  display_name = "Infrastructure Manager Execution SA"
  project      = var.project_id
}

# Grant permissions to execution SA
resource "google_project_iam_member" "infra_mgr_roles" {
  for_each = toset([
    "roles/compute.admin",
    "roles/container.admin",
    "roles/storage.admin",
    "roles/run.admin",
    "roles/iam.serviceAccountAdmin",
    "roles/resourcemanager.projectIamAdmin",
  ])

  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.infra_mgr.email}"
}

# Allow service agent to use execution SA
data "google_project" "current" {
  project_id = var.project_id
}

resource "google_service_account_iam_member" "agent_actas" {
  service_account_id = google_service_account.infra_mgr.name
  role               = "roles/iam.serviceAccountUser"
  member             = "serviceAccount:service-${data.google_project.current.number}@gcp-sa-config.iam.gserviceaccount.com"

  depends_on = [google_project_service.infra_manager]
}

# Create the deployment
resource "google_infra_manager_deployment" "app" {
  deployment_id = "${var.environment}-app-infra"
  location      = var.region
  project       = var.project_id

  service_account = "projects/${var.project_id}/serviceAccounts/${google_service_account.infra_mgr.email}"

  terraform_blueprint {
    git_source {
      repo      = var.blueprint_repo
      directory = "environments/${var.environment}"
      ref       = var.blueprint_ref
    }

    input_values = {
      project_id  = var.project_id
      region      = var.region
      environment = var.environment
    }
  }

  labels = {
    environment = var.environment
    managed-by  = "terraform"
  }

  depends_on = [
    google_project_service.infra_manager,
    google_service_account_iam_member.agent_actas,
    google_project_iam_member.infra_mgr_roles,
  ]
}
```

---

## Part 14 — gcloud CLI Reference

### Complete CLI Reference

```bash
# ─── Enable API ───────────────────────────────────────────────
gcloud services enable config.googleapis.com --project=PROJECT_ID

# ─── Deployments ──────────────────────────────────────────────
# Create or update deployment (from Git)
gcloud infra-manager deployments apply DEPLOYMENT_ID \
    --location=REGION \
    --git-source-repo="REPO_URL" \
    --git-source-directory="PATH" \
    --git-source-ref="BRANCH_OR_TAG" \
    --service-account=projects/PROJECT/serviceAccounts/SA_EMAIL \
    --input-values="key1=val1,key2=val2" \
    --labels="env=prod,team=platform" \
    --annotations="owner=team-a"

# Create or update deployment (from GCS)
gcloud infra-manager deployments apply DEPLOYMENT_ID \
    --location=REGION \
    --gcs-source="gs://BUCKET/PATH" \
    --service-account=projects/PROJECT/serviceAccounts/SA_EMAIL \
    --input-values="key1=val1"

# Create or update deployment (from local)
gcloud infra-manager deployments apply DEPLOYMENT_ID \
    --location=REGION \
    --local-source="./LOCAL_PATH" \
    --service-account=projects/PROJECT/serviceAccounts/SA_EMAIL \
    --input-values="key1=val1"

# List deployments
gcloud infra-manager deployments list --location=REGION
gcloud infra-manager deployments list --location=REGION --filter="labels.env=prod"

# Describe deployment
gcloud infra-manager deployments describe DEPLOYMENT_ID --location=REGION
gcloud infra-manager deployments describe DEPLOYMENT_ID --location=REGION --format=json

# Get outputs
gcloud infra-manager deployments describe DEPLOYMENT_ID \
    --location=REGION --format="yaml(outputs)"

# Delete deployment (destroys resources)
gcloud infra-manager deployments delete DEPLOYMENT_ID --location=REGION

# Lock deployment
gcloud infra-manager deployments lock DEPLOYMENT_ID --location=REGION

# Unlock deployment
gcloud infra-manager deployments unlock DEPLOYMENT_ID --location=REGION

# Export state file
gcloud infra-manager deployments export-statefile DEPLOYMENT_ID \
    --location=REGION --output-file=STATE_FILE

# Import state file
gcloud infra-manager deployments import-statefile DEPLOYMENT_ID \
    --location=REGION --input-file=STATE_FILE

# ─── Previews ────────────────────────────────────────────────
# Create preview
gcloud infra-manager previews create PREVIEW_ID \
    --location=REGION \
    --deployment=DEPLOYMENT_ID \
    --local-source="./PATH" \
    --service-account=projects/PROJECT/serviceAccounts/SA_EMAIL

# Create preview (Git source)
gcloud infra-manager previews create PREVIEW_ID \
    --location=REGION \
    --deployment=DEPLOYMENT_ID \
    --git-source-repo="REPO_URL" \
    --git-source-directory="PATH" \
    --git-source-ref="BRANCH" \
    --service-account=projects/PROJECT/serviceAccounts/SA_EMAIL

# List previews
gcloud infra-manager previews list --location=REGION

# Describe preview (see plan results)
gcloud infra-manager previews describe PREVIEW_ID --location=REGION

# Export preview
gcloud infra-manager previews export PREVIEW_ID \
    --location=REGION --output-file=PLAN_FILE

# Delete preview
gcloud infra-manager previews delete PREVIEW_ID --location=REGION

# ─── Revisions ───────────────────────────────────────────────
# List revisions
gcloud infra-manager revisions list \
    --deployment=DEPLOYMENT_ID --location=REGION

# Describe revision
gcloud infra-manager revisions describe REVISION_ID \
    --deployment=DEPLOYMENT_ID --location=REGION

# Get revision logs
gcloud infra-manager revisions describe REVISION_ID \
    --deployment=DEPLOYMENT_ID --location=REGION \
    --format="value(build)"

# ─── Resources ───────────────────────────────────────────────
# List resources in a revision
gcloud infra-manager resources list \
    --revision=REVISION_ID \
    --deployment=DEPLOYMENT_ID \
    --location=REGION

# Describe a managed resource
gcloud infra-manager resources describe RESOURCE_NAME \
    --revision=REVISION_ID \
    --deployment=DEPLOYMENT_ID \
    --location=REGION
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Environment Promotion Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: ENVIRONMENT PROMOTION WITH INFRA MANAGER              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Git Repository Structure:                                             │
│  infra-configs/                                                        │
│  ├── modules/          (shared Terraform modules)                     │
│  │   ├── network/                                                     │
│  │   ├── gke/                                                         │
│  │   └── cloud-sql/                                                   │
│  ├── environments/                                                     │
│  │   ├── dev/          (dev-specific values)                          │
│  │   ├── staging/      (staging-specific values)                      │
│  │   └── prod/         (prod-specific values)                         │
│  └── .github/workflows/                                               │
│                                                                        │
│  Promotion Flow:                                                       │
│  ┌─────────┐    merge    ┌──────────┐   merge    ┌──────────┐       │
│  │   Dev   │───────────▶│ Staging  │───────────▶│   Prod   │       │
│  │         │             │          │            │          │       │
│  │ Branch: │             │ Branch:  │            │ Branch:  │       │
│  │  dev    │             │ staging  │            │  main    │       │
│  └─────────┘             └──────────┘            └──────────┘       │
│       │                       │                       │               │
│       ▼                       ▼                       ▼               │
│  Infra Manager           Infra Manager          Infra Manager        │
│  Deployment:             Deployment:            Deployment:          │
│  "dev-infra"             "staging-infra"        "prod-infra"         │
│                                                                        │
│  Each environment:                                                     │
│  • Same modules, different variables                                  │
│  • Separate Infra Manager deployments                                 │
│  • Independent state and revisions                                    │
│  • Preview on PR, apply on merge                                      │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```yaml
# .github/workflows/deploy.yml
name: Infrastructure Deployment
on:
  push:
    branches: [dev, staging, main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - id: set-env
        run: |
          if [ "${{ github.ref_name }}" = "dev" ]; then
            echo "env=dev" >> $GITHUB_OUTPUT
            echo "deployment=dev-infra" >> $GITHUB_OUTPUT
          elif [ "${{ github.ref_name }}" = "staging" ]; then
            echo "env=staging" >> $GITHUB_OUTPUT
            echo "deployment=staging-infra" >> $GITHUB_OUTPUT
          else
            echo "env=prod" >> $GITHUB_OUTPUT
            echo "deployment=prod-infra" >> $GITHUB_OUTPUT
          fi

      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.WIF_PROVIDER }}
          service_account: ${{ vars.CI_SA }}

      - uses: google-github-actions/setup-gcloud@v2

      - name: Deploy
        run: |
          gcloud infra-manager deployments apply ${{ steps.set-env.outputs.deployment }} \
            --location=us-central1 \
            --local-source=./environments/${{ steps.set-env.outputs.env }} \
            --service-account=projects/${{ vars.PROJECT_ID }}/serviceAccounts/infra-mgr@${{ vars.PROJECT_ID }}.iam.gserviceaccount.com \
            --input-values="project_id=${{ vars.PROJECT_ID }},region=us-central1,environment=${{ steps.set-env.outputs.env }}"
```

### Pattern 2: Self-Service Infrastructure Platform

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: SELF-SERVICE PLATFORM WITH INFRA MANAGER              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  Internal Developer Portal (e.g., Backstage)               │      │
│  │                                                            │      │
│  │  Developer selects:                                         │      │
│  │  • Template: "Web Application" / "Data Pipeline" / "API"   │      │
│  │  • Environment: dev / staging / prod                       │      │
│  │  • Size: small / medium / large                            │      │
│  │  • Region: us-central1 / europe-west1                      │      │
│  └───────────────────────────┬────────────────────────────────┘      │
│                              │                                         │
│                              ▼                                         │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  API / Cloud Function                                       │      │
│  │  • Validates request                                        │      │
│  │  • Selects blueprint template                               │      │
│  │  • Calls Infra Manager API                                  │      │
│  └───────────────────────────┬────────────────────────────────┘      │
│                              │                                         │
│                              ▼                                         │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  Infrastructure Manager                                     │      │
│  │  • Creates deployment from pre-approved blueprint           │      │
│  │  • Applies with team-specific service account               │      │
│  │  • Returns outputs (URLs, connection strings)               │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  Blueprint Library (GCS):                                              │
│  gs://blueprints/                                                      │
│  ├── web-app/          (Cloud Run + VPC + SQL)                        │
│  ├── data-pipeline/    (Dataflow + BigQuery + GCS)                    │
│  ├── api-service/      (GKE + Cloud Armor + LB)                       │
│  └── ml-platform/      (Vertex AI + Notebooks + GCS)                  │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**API Implementation:**

```python
# Cloud Function — Self-service infrastructure provisioning
from google.cloud import config_v1
import functions_framework

BLUEPRINTS = {
    "web-app": "gs://my-project-blueprints/web-app",
    "data-pipeline": "gs://my-project-blueprints/data-pipeline",
    "api-service": "gs://my-project-blueprints/api-service",
}

SIZE_CONFIGS = {
    "small": {"machine_type": "e2-small", "min_instances": "0", "max_instances": "5"},
    "medium": {"machine_type": "e2-medium", "min_instances": "1", "max_instances": "20"},
    "large": {"machine_type": "e2-standard-4", "min_instances": "2", "max_instances": "100"},
}

@functions_framework.http
def provision_infrastructure(request):
    """Provision infrastructure via Infrastructure Manager."""
    data = request.get_json()
    
    template = data["template"]
    environment = data["environment"]
    size = data["size"]
    team = data["team"]
    region = data.get("region", "us-central1")
    
    deployment_id = f"{team}-{template}-{environment}"
    
    client = config_v1.ConfigClient()
    
    input_values = {
        "project_id": "my-project",
        "region": region,
        "environment": environment,
        "team": team,
        **SIZE_CONFIGS[size],
    }
    
    deployment = config_v1.Deployment(
        terraform_blueprint=config_v1.TerraformBlueprint(
            gcs_source=BLUEPRINTS[template],
            input_values=input_values,
        ),
        service_account=f"projects/my-project/serviceAccounts/infra-mgr@my-project.iam.gserviceaccount.com",
        labels={"team": team, "environment": environment, "template": template},
    )
    
    operation = client.create_deployment(
        parent=f"projects/my-project/locations/{region}",
        deployment_id=deployment_id,
        deployment=deployment,
    )
    
    return {"deployment_id": deployment_id, "status": "creating"}
```

### Pattern 3: Multi-Project Infrastructure with Centralized Management

```
┌──────────────────────────────────────────────────────────────────────┐
│   PATTERN 3: CENTRALIZED INFRA MANAGEMENT ACROSS PROJECTS            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │  Management Project (infra-mgmt)                             │     │
│  │                                                              │     │
│  │  • All Infra Manager deployments live here                   │     │
│  │  • Centralized state and revision history                    │     │
│  │  • Single execution SA with cross-project permissions        │     │
│  │  • Platform team manages blueprints                          │     │
│  │                                                              │     │
│  │  Deployments:                                                │     │
│  │  ├── team-a-dev        (targets: team-a-dev-project)        │     │
│  │  ├── team-a-prod       (targets: team-a-prod-project)       │     │
│  │  ├── team-b-dev        (targets: team-b-dev-project)        │     │
│  │  ├── team-b-prod       (targets: team-b-prod-project)       │     │
│  │  └── shared-services   (targets: shared-project)            │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                    │                                                    │
│         ┌──────────┼──────────┐                                       │
│         ▼          ▼          ▼                                        │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐                          │
│  │ Team A    │ │ Team B    │ │ Shared    │                          │
│  │ Projects  │ │ Projects  │ │ Services  │                          │
│  │           │ │           │ │ Project   │                          │
│  │ VPC, GKE, │ │ VPC, Run, │ │ VPC, LB,  │                          │
│  │ SQL       │ │ Functions │ │ DNS       │                          │
│  └───────────┘ └───────────┘ └───────────┘                          │
│                                                                        │
│  Benefits:                                                             │
│  • Single pane of glass for all infrastructure                        │
│  • Consistent blueprints across teams                                  │
│  • Centralized audit trail (all revisions in one project)             │
│  • Team-specific IAM (config.viewer per team)                         │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```hcl
# Management project Terraform — creates deployments for all teams
locals {
  teams = {
    "team-a" = {
      dev_project  = "team-a-dev"
      prod_project = "team-a-prod"
      blueprint    = "web-app"
    }
    "team-b" = {
      dev_project  = "team-b-dev"
      prod_project = "team-b-prod"
      blueprint    = "data-pipeline"
    }
  }
}

# Create Infra Manager deployments for each team/environment
resource "google_infra_manager_deployment" "team_deployments" {
  for_each = { for pair in flatten([
    for team, config in local.teams : [
      { key = "${team}-dev", team = team, project = config.dev_project, env = "dev", blueprint = config.blueprint },
      { key = "${team}-prod", team = team, project = config.prod_project, env = "prod", blueprint = config.blueprint },
    ]
  ]) : pair.key => pair }

  deployment_id = each.key
  location      = "us-central1"
  project       = "infra-mgmt"  # All deployments in management project

  service_account = "projects/infra-mgmt/serviceAccounts/infra-mgr@infra-mgmt.iam.gserviceaccount.com"

  terraform_blueprint {
    gcs_source = "gs://infra-mgmt-blueprints/${each.value.blueprint}"

    input_values = {
      project_id  = each.value.project
      region      = "us-central1"
      environment = each.value.env
      team        = each.value.team
    }
  }

  labels = {
    team        = each.value.team
    environment = each.value.env
    blueprint   = each.value.blueprint
  }
}
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Enable API | `gcloud services enable config.googleapis.com` |
| Create/update deployment (Git) | `gcloud infra-manager deployments apply NAME --location=REGION --git-source-repo=URL --git-source-directory=PATH --git-source-ref=REF --service-account=SA` |
| Create/update deployment (GCS) | `gcloud infra-manager deployments apply NAME --location=REGION --gcs-source=gs://BUCKET/PATH --service-account=SA` |
| Create/update deployment (local) | `gcloud infra-manager deployments apply NAME --location=REGION --local-source=./PATH --service-account=SA` |
| List deployments | `gcloud infra-manager deployments list --location=REGION` |
| Describe deployment | `gcloud infra-manager deployments describe NAME --location=REGION` |
| Delete deployment | `gcloud infra-manager deployments delete NAME --location=REGION` |
| Lock deployment | `gcloud infra-manager deployments lock NAME --location=REGION` |
| Unlock deployment | `gcloud infra-manager deployments unlock NAME --location=REGION` |
| Export state | `gcloud infra-manager deployments export-statefile NAME --location=REGION --output-file=FILE` |
| Create preview | `gcloud infra-manager previews create NAME --location=REGION --deployment=DEPLOY --local-source=PATH --service-account=SA` |
| List revisions | `gcloud infra-manager revisions list --deployment=NAME --location=REGION` |

---

## What's Next?

Continue to **Chapter 36: Cloud Monitoring** → `36-cloud-monitoring.md`
