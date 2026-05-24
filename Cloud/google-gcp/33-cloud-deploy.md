# Chapter 33 — Cloud Deploy

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Fundamentals](#part-1--fundamentals)
- [Part 2: Architecture & Core Concepts](#part-2--architecture--core-concepts)
- [Part 3: Delivery Pipeline Configuration](#part-3--delivery-pipeline-configuration)
- [Part 4: Targets](#part-4--targets)
- [Part 5: Releases & Rollouts](#part-5--releases--rollouts)
- [Part 6: Approval Gates](#part-6--approval-gates)
- [Part 7: Deployment Strategies](#part-7--deployment-strategies)
- [Part 8: Deploy Verification](#part-8--deploy-verification)
- [Part 9: Rollbacks](#part-9--rollbacks)
- [Part 10: Automation & Hooks](#part-10--automation--hooks)
- [Part 11: Notifications & Integrations](#part-11--notifications--integrations)
- [Part 12: IAM & Security](#part-12--iam--security)
- [Part 13: Cloud Run Deployments](#part-13--cloud-run-deployments)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What is Continuous Delivery? (Beginner Explanation)](#what-is-continuous-delivery-beginner-explanation)
- [Console Walkthrough: Managing Releases & Rollouts](#console-walkthrough-managing-releases--rollouts)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud Deploy is a fully managed continuous delivery service that automates deployments to GKE, Cloud Run, and Anthos clusters. It provides opinionated, secure delivery pipelines with built-in approval gates, rollback capabilities, and promotion workflows across environments—bringing GitOps-style progressive delivery without managing your own CD infrastructure.

---

## Part 1 — Fundamentals

### What Is Cloud Deploy?

Cloud Deploy is Google's managed CD (Continuous Delivery) service that handles the "last mile" of deployment—taking built artifacts from CI (Cloud Build / Artifact Registry) and promoting them through environments (dev → staging → prod) with safety controls.

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLOUD DEPLOY OVERVIEW                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────┐    ┌──────────────────────────────────────────┐   │
│  │  Cloud   │    │         Delivery Pipeline                 │   │
│  │  Build   │───▶│                                           │   │
│  │  (CI)    │    │  ┌─────┐    ┌─────────┐    ┌──────┐     │   │
│  └──────────┘    │  │ Dev │───▶│ Staging │───▶│ Prod │     │   │
│                  │  └─────┘    └─────────┘    └──────┘     │   │
│  ┌──────────┐    │     │            │             │          │   │
│  │ Artifact │    │  Rollout     Rollout       Rollout       │   │
│  │ Registry │    │  (auto)     (auto)        (approval)     │   │
│  └──────────┘    └──────────────────────────────────────────┘   │
│                                                                   │
│  Key Concepts:                                                    │
│  • Pipeline = ordered sequence of targets                        │
│  • Target = where to deploy (GKE cluster, Cloud Run service)     │
│  • Release = what to deploy (container image + manifests)        │
│  • Rollout = actual deployment to a specific target              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cloud Deploy vs Other CD Tools

| Feature | Cloud Deploy | Spinnaker | ArgoCD | AWS CodeDeploy |
|---------|-------------|-----------|--------|----------------|
| Managed | Fully managed | Self-managed | Self-managed | Fully managed |
| Targets | GKE, Cloud Run, Anthos | Multi-cloud | Kubernetes | EC2, ECS, Lambda |
| Approach | Push-based promotion | Push-based | GitOps (pull) | Push-based |
| Approval | Built-in IAM-based | Pipeline stages | Sync policies | Built-in |
| Rollback | One-click | Manual | Git revert | Automatic |
| Canary | Built-in | Built-in | Progressive | Built-in |
| Cost | Per delivery pipeline | Infrastructure | Infrastructure | Per deployment |

### Cross-Cloud Comparison

| Feature | GCP Cloud Deploy | AWS CodeDeploy + CodePipeline | Azure DevOps Pipelines |
|---------|-----------------|-------------------------------|------------------------|
| Service | Cloud Deploy | CodeDeploy + CodePipeline | Azure Pipelines |
| Targets | GKE, Cloud Run | EC2, ECS, Lambda, EKS | AKS, App Service, VMs |
| Config | YAML (declarative) | appspec.yml + pipeline JSON | YAML pipelines |
| Environments | Targets in pipeline | Stages in pipeline | Environments with gates |
| Approvals | IAM-based | Manual approval action | Environment approvals |
| Canary | Native support | Native (EC2/ECS) | Needs extension |
| Rollback | Built-in | Automatic on failure | Manual or scripted |

### Pricing Overview

| Component | Cost |
|-----------|------|
| Delivery pipeline | ~$3.00/month per active pipeline |
| Target | Included in pipeline cost |
| Releases | No additional charge |
| Rollouts | No additional charge |
| Render operations | Cloud Build charges apply |
| Deploy operations | Cloud Build charges apply |
| Storage | Artifacts stored in GCS (standard rates) |

> **Note**: Cloud Deploy uses Cloud Build under the hood for rendering manifests and executing deploys. Those build minutes are billed separately.

---

## Part 2 — Architecture & Core Concepts

### Delivery Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                   CLOUD DEPLOY ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │              Delivery Pipeline Definition                 │        │
│  │              (clouddeploy.yaml)                          │        │
│  └────────────────────────┬────────────────────────────────┘        │
│                           │                                          │
│         ┌─────────────────┼─────────────────┐                       │
│         ▼                 ▼                 ▼                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │   Target:   │  │   Target:   │  │   Target:   │                 │
│  │     dev     │  │   staging   │  │    prod     │                 │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤                 │
│  │ GKE Cluster │  │ GKE Cluster │  │ GKE Cluster │                 │
│  │ us-central1 │  │ us-central1 │  │ us-east1    │                 │
│  │             │  │             │  │             │                 │
│  │ No approval │  │ No approval │  │ Approval    │                 │
│  │ Auto-promote│  │ Auto-promote│  │ required    │                 │
│  └─────────────┘  └─────────────┘  └─────────────┘                 │
│                                                                       │
│  Release Flow:                                                        │
│  ┌────────┐  render  ┌──────────┐  deploy  ┌─────────┐             │
│  │ Create │─────────▶│ Rendered │─────────▶│ Rollout │             │
│  │Release │          │Manifests │          │ (per    │             │
│  └────────┘          └──────────┘          │ target) │             │
│                                             └─────────┘             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Core Concepts Detailed

| Concept | Description | Analogy |
|---------|-------------|---------|
| **Delivery Pipeline** | Ordered sequence of targets representing environments | The assembly line |
| **Target** | A specific deployment destination (cluster/service) | A workstation on the line |
| **Release** | A specific version of your application to deploy | A product version |
| **Rollout** | The act of deploying a release to a target | Actually building the product |
| **Manifest** | Kubernetes YAML or Cloud Run service definition | The blueprint |
| **Render** | Process of substituting values into manifest templates | Customizing the blueprint |
| **Promotion** | Moving a release from one target to the next | Passing QA |
| **Approval** | Gate requiring human sign-off before rollout proceeds | Quality checkpoint |

### Release Lifecycle

```
┌──────────────────────────────────────────────────────────────┐
│                    RELEASE LIFECYCLE                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  CREATE RELEASE                                                │
│       │                                                        │
│       ▼                                                        │
│  ┌──────────┐                                                  │
│  │ RENDER   │──── Cloud Build renders manifests per target     │
│  └────┬─────┘                                                  │
│       │                                                        │
│       ▼                                                        │
│  ┌──────────────────────────────────────────────┐             │
│  │  ROLLOUT TO FIRST TARGET (e.g., dev)         │             │
│  │  Status: IN_PROGRESS → SUCCEEDED             │             │
│  └────┬─────────────────────────────────────────┘             │
│       │                                                        │
│       ▼  (promote or auto-promote)                            │
│  ┌──────────────────────────────────────────────┐             │
│  │  ROLLOUT TO SECOND TARGET (e.g., staging)    │             │
│  │  Status: IN_PROGRESS → SUCCEEDED             │             │
│  └────┬─────────────────────────────────────────┘             │
│       │                                                        │
│       ▼  (promote — requires approval)                        │
│  ┌──────────────────────────────────────────────┐             │
│  │  APPROVAL GATE                                │             │
│  │  Status: PENDING_APPROVAL → APPROVED          │             │
│  └────┬─────────────────────────────────────────┘             │
│       │                                                        │
│       ▼                                                        │
│  ┌──────────────────────────────────────────────┐             │
│  │  ROLLOUT TO THIRD TARGET (e.g., prod)        │             │
│  │  Status: IN_PROGRESS → SUCCEEDED             │             │
│  └──────────────────────────────────────────────┘             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Rollout States

| State | Description |
|-------|-------------|
| `PENDING_RELEASE` | Waiting for render to complete |
| `PENDING_APPROVAL` | Waiting for approval (if required) |
| `APPROVED` | Approved, waiting to deploy |
| `IN_PROGRESS` | Deployment underway |
| `SUCCEEDED` | Deployment completed successfully |
| `FAILED` | Deployment failed |
| `CANCELLING` | Being cancelled |
| `CANCELLED` | Cancelled by user |
| `HALTED` | Halted (canary/verify failure) |

---

## Part 3 — Delivery Pipeline Configuration

### Pipeline Definition (clouddeploy.yaml)

The delivery pipeline is defined in a `clouddeploy.yaml` file that describes the pipeline and its targets:

```yaml
# clouddeploy.yaml — Pipeline + Targets in one file
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: my-app-pipeline
description: "Production delivery pipeline for my-app"
serialPipeline:
  stages:
    - targetId: dev
      profiles: [dev]
    - targetId: staging
      profiles: [staging]
    - targetId: prod
      profiles: [prod]
      strategy:
        standard:
          verify: true
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: dev
description: "Development GKE cluster"
gke:
  cluster: projects/my-project/locations/us-central1/clusters/dev-cluster
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: staging
description: "Staging GKE cluster"
gke:
  cluster: projects/my-project/locations/us-central1/clusters/staging-cluster
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod
description: "Production GKE cluster"
requireApproval: true
gke:
  cluster: projects/my-project/locations/us-east1/clusters/prod-cluster
```

### Registering the Pipeline

```bash
# Apply pipeline and targets to Cloud Deploy
gcloud deploy apply --file=clouddeploy.yaml \
    --region=us-central1 \
    --project=my-project
```

### Pipeline with Multiple Parallel Targets

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: multi-region-pipeline
serialPipeline:
  stages:
    - targetId: dev
      profiles: [dev]
    - targetId: staging
      profiles: [staging]
    - targetId: prod-us
      profiles: [prod]
    - targetId: prod-eu
      profiles: [prod]
```

### Pipeline with Canary Strategy

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: canary-pipeline
serialPipeline:
  stages:
    - targetId: dev
      profiles: [dev]
    - targetId: staging
      profiles: [staging]
    - targetId: prod
      profiles: [prod]
      strategy:
        canary:
          runtimeConfig:
            kubernetes:
              serviceNetworking:
                service: "my-app-service"
                deployment: "my-app"
          canaryDeployment:
            percentages: [10, 25, 50, 75]
            verify: true
```

---

## Part 4 — Targets

### Target Types

```
┌──────────────────────────────────────────────────────────────┐
│                      TARGET TYPES                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  │
│  │      GKE       │  │   Cloud Run    │  │    Anthos      │  │
│  ├────────────────┤  ├────────────────┤  ├────────────────┤  │
│  │ • Standard     │  │ • Services     │  │ • Attached     │  │
│  │ • Autopilot    │  │ • Jobs         │  │   clusters     │  │
│  │ • Private      │  │                │  │ • On-prem      │  │
│  │ • Multi-zone   │  │                │  │ • Multi-cloud  │  │
│  └────────────────┘  └────────────────┘  └────────────────┘  │
│                                                                │
│  ┌────────────────┐                                           │
│  │  Custom Target │                                           │
│  ├────────────────┤                                           │
│  │ • Any platform │                                           │
│  │ • User-defined │                                           │
│  │   actions      │                                           │
│  └────────────────┘                                           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### GKE Target

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod-gke
description: "Production GKE cluster"
requireApproval: true
gke:
  cluster: projects/my-project/locations/us-central1/clusters/prod-cluster
executionConfigs:
  - usages:
      - RENDER
      - DEPLOY
    serviceAccount: deploy-sa@my-project.iam.gserviceaccount.com
    workerPool: projects/my-project/locations/us-central1/workerPools/my-pool
```

### Cloud Run Target

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod-cloudrun
description: "Production Cloud Run service"
requireApproval: true
run:
  location: projects/my-project/locations/us-central1
```

### Anthos Target

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: on-prem-cluster
description: "On-premises Anthos cluster"
anthosCluster:
  membership: projects/my-project/locations/global/memberships/on-prem-cluster-1
```

### Custom Target

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: custom-vm-target
description: "Deploy to VMs using custom scripts"
customTarget:
  customTargetType: projects/my-project/locations/us-central1/customTargetTypes/vm-deploy
```

### Multi-Target (Parallel Deployment)

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod-multi
description: "Multi-region production"
multiTarget:
  targetIds:
    - prod-us-central1
    - prod-us-east1
    - prod-europe-west1
```

### Target with Private GKE Cluster

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: private-gke
gke:
  cluster: projects/my-project/locations/us-central1/clusters/private-cluster
  internalIp: true
executionConfigs:
  - usages: [RENDER, DEPLOY]
    workerPool: projects/my-project/locations/us-central1/workerPools/private-pool
```

---

## Part 5 — Releases & Rollouts

### Creating a Release

```bash
# Create a release from Skaffold config
gcloud deploy releases create release-001 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1 \
    --source=./ \
    --images=my-app=us-central1-docker.pkg.dev/my-project/repo/my-app:v1.2.3

# Create release with description
gcloud deploy releases create release-002 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1 \
    --source=./ \
    --images=my-app=us-central1-docker.pkg.dev/my-project/repo/my-app:v1.2.4 \
    --description="Feature: Add user dashboard"

# Create release with labels
gcloud deploy releases create release-003 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1 \
    --source=./ \
    --images=my-app=us-central1-docker.pkg.dev/my-project/repo/my-app:v1.3.0 \
    --labels=team=backend,sprint=42
```

### Skaffold Configuration

Cloud Deploy uses Skaffold to define how to render and deploy manifests:

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta7
kind: Config
metadata:
  name: my-app
profiles:
  - name: dev
    manifests:
      rawYaml:
        - k8s/dev/*.yaml
  - name: staging
    manifests:
      rawYaml:
        - k8s/staging/*.yaml
  - name: prod
    manifests:
      rawYaml:
        - k8s/prod/*.yaml
deploy:
  kubectl: {}
```

### Skaffold with Helm

```yaml
# skaffold.yaml — Helm renderer
apiVersion: skaffold/v4beta7
kind: Config
metadata:
  name: my-app-helm
profiles:
  - name: dev
    manifests:
      helm:
        releases:
          - name: my-app
            chartPath: charts/my-app
            valuesFiles:
              - values/dev.yaml
  - name: prod
    manifests:
      helm:
        releases:
          - name: my-app
            chartPath: charts/my-app
            valuesFiles:
              - values/prod.yaml
deploy:
  kubectl: {}
```

### Skaffold with Kustomize

```yaml
# skaffold.yaml — Kustomize renderer
apiVersion: skaffold/v4beta7
kind: Config
metadata:
  name: my-app-kustomize
profiles:
  - name: dev
    manifests:
      kustomize:
        paths:
          - k8s/overlays/dev
  - name: prod
    manifests:
      kustomize:
        paths:
          - k8s/overlays/prod
deploy:
  kubectl: {}
```

### Promoting a Release

```bash
# Promote release to next target in pipeline
gcloud deploy releases promote \
    --release=release-001 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Promote to specific target
gcloud deploy releases promote \
    --release=release-001 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1 \
    --to-target=prod
```

### Rollout Details

```bash
# List rollouts for a release
gcloud deploy rollouts list \
    --release=release-001 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Describe a specific rollout
gcloud deploy rollouts describe rollout-001 \
    --release=release-001 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Check rollout status
gcloud deploy rollouts list \
    --release=release-001 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1 \
    --filter="state=SUCCEEDED"
```

---

## Part 6 — Approval Gates

### Configuring Approvals

```
┌──────────────────────────────────────────────────────────────┐
│                    APPROVAL WORKFLOW                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Release promoted to prod target                              │
│         │                                                      │
│         ▼                                                      │
│  ┌─────────────────────────────────────┐                      │
│  │  ROLLOUT STATUS: PENDING_APPROVAL   │                      │
│  │                                     │                      │
│  │  • Notification sent (Pub/Sub)      │                      │
│  │  • Visible in Console               │                      │
│  │  • Awaiting IAM-authorized user     │                      │
│  └──────────────────┬──────────────────┘                      │
│                     │                                          │
│         ┌───────────┴───────────┐                             │
│         ▼                       ▼                              │
│  ┌─────────────┐        ┌─────────────┐                      │
│  │  APPROVED   │        │  REJECTED   │                      │
│  │             │        │             │                      │
│  │  Rollout    │        │  Rollout    │                      │
│  │  proceeds   │        │  cancelled  │                      │
│  └─────────────┘        └─────────────┘                      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Target with Approval Required

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod
description: "Production — requires approval"
requireApproval: true
gke:
  cluster: projects/my-project/locations/us-central1/clusters/prod
```

### Approving/Rejecting a Rollout

```bash
# Approve a pending rollout
gcloud deploy rollouts approve rollout-001 \
    --release=release-001 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Reject a pending rollout
gcloud deploy rollouts reject rollout-001 \
    --release=release-001 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1
```

### IAM Roles for Approval

| Role | Permission | Use Case |
|------|-----------|----------|
| `roles/clouddeploy.approver` | Approve/reject rollouts | Release managers |
| `roles/clouddeploy.operator` | Create releases, promote | CI/CD service accounts |
| `roles/clouddeploy.viewer` | Read-only access | Developers, auditors |
| `roles/clouddeploy.admin` | Full management | Platform team |

### Approval Policy (Advanced)

```yaml
# Target with approval policy
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod
  annotations:
    deploy.cloud.google.com/approval-notification-topic: projects/my-project/topics/deploy-approvals
requireApproval: true
gke:
  cluster: projects/my-project/locations/us-central1/clusters/prod
```

---

## Part 7 — Deployment Strategies

### Standard Deployment

The default strategy — full replacement of the previous version:

```yaml
serialPipeline:
  stages:
    - targetId: prod
      profiles: [prod]
      strategy:
        standard:
          verify: false   # Set true to enable verification
```

### Canary Deployment

```
┌──────────────────────────────────────────────────────────────┐
│                    CANARY DEPLOYMENT                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Phase 1: 10% canary                                          │
│  ┌───────────────────────────────────────────┐                │
│  │ ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │  10% new      │
│  └───────────────────────────────────────────┘                │
│         │ verify ✓                                            │
│         ▼                                                      │
│  Phase 2: 25% canary                                          │
│  ┌───────────────────────────────────────────┐                │
│  │ ██████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │  25% new      │
│  └───────────────────────────────────────────┘                │
│         │ verify ✓                                            │
│         ▼                                                      │
│  Phase 3: 50% canary                                          │
│  ┌───────────────────────────────────────────┐                │
│  │ ████████████████████░░░░░░░░░░░░░░░░░░░░░ │  50% new      │
│  └───────────────────────────────────────────┘                │
│         │ verify ✓                                            │
│         ▼                                                      │
│  Phase 4: 100% (stable)                                       │
│  ┌───────────────────────────────────────────┐                │
│  │ ████████████████████████████████████████░░ │  100% new     │
│  └───────────────────────────────────────────┘                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Canary for GKE (Service Networking)

```yaml
serialPipeline:
  stages:
    - targetId: prod
      profiles: [prod]
      strategy:
        canary:
          runtimeConfig:
            kubernetes:
              serviceNetworking:
                service: "my-app-service"
                deployment: "my-app"
                podSelectorLabel: "app"
          canaryDeployment:
            percentages: [10, 25, 50, 75]
            verify: true
```

### Canary for GKE (Gateway API)

```yaml
serialPipeline:
  stages:
    - targetId: prod
      profiles: [prod]
      strategy:
        canary:
          runtimeConfig:
            kubernetes:
              gatewayServiceMesh:
                httpRoute: "my-app-route"
                service: "my-app-service"
                deployment: "my-app"
          canaryDeployment:
            percentages: [10, 30, 50, 80]
            verify: true
```

### Canary for Cloud Run

```yaml
serialPipeline:
  stages:
    - targetId: prod-run
      profiles: [prod]
      strategy:
        canary:
          runtimeConfig:
            cloudRun:
              automaticTrafficControl: true
          canaryDeployment:
            percentages: [10, 25, 50, 75]
            verify: true
```

### Custom Canary with Manual Phases

```yaml
serialPipeline:
  stages:
    - targetId: prod
      profiles: [prod]
      strategy:
        canary:
          customCanaryDeployment:
            phaseConfigs:
              - phaseId: "canary-10"
                percentage: 10
                profiles: [canary]
                verify: true
              - phaseId: "canary-50"
                percentage: 50
                profiles: [canary]
                verify: true
              - phaseId: "stable"
                percentage: 100
                profiles: [prod]
                verify: true
```

---

## Part 8 — Deploy Verification

### What Is Verification?

Verification runs automated tests after a deployment to confirm the release is healthy before promoting further:

```
┌──────────────────────────────────────────────────────────────┐
│                  VERIFICATION WORKFLOW                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Rollout Deploys                                               │
│       │                                                        │
│       ▼                                                        │
│  ┌─────────────────────────────────────┐                      │
│  │  VERIFICATION PHASE                  │                      │
│  │                                     │                      │
│  │  • Runs Cloud Build job             │                      │
│  │  • Executes test container          │                      │
│  │  • Checks exit code                 │                      │
│  │                                     │                      │
│  │  Exit 0 = PASS → proceed            │                      │
│  │  Exit !0 = FAIL → halt rollout      │                      │
│  └────────────────┬────────────────────┘                      │
│                   │                                            │
│       ┌───────────┴───────────┐                               │
│       ▼                       ▼                                │
│  ┌─────────┐           ┌──────────┐                          │
│  │  PASS   │           │  FAIL    │                          │
│  │ Promote │           │ Rollback │                          │
│  └─────────┘           └──────────┘                          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Enabling Verification in Pipeline

```yaml
serialPipeline:
  stages:
    - targetId: staging
      profiles: [staging]
      strategy:
        standard:
          verify: true
    - targetId: prod
      profiles: [prod]
      strategy:
        standard:
          verify: true
```

### Skaffold Verify Configuration

```yaml
# skaffold.yaml with verify stanza
apiVersion: skaffold/v4beta7
kind: Config
metadata:
  name: my-app
profiles:
  - name: staging
    manifests:
      rawYaml:
        - k8s/staging/*.yaml
    verify:
      - name: smoke-test
        container:
          name: smoke-test
          image: us-central1-docker.pkg.dev/my-project/repo/smoke-tests:latest
          command: ["./run-smoke-tests.sh"]
          args: ["--env=staging"]
      - name: integration-test
        container:
          name: integration-test
          image: us-central1-docker.pkg.dev/my-project/repo/integration-tests:latest
          command: ["./run-integration-tests.sh"]
  - name: prod
    manifests:
      rawYaml:
        - k8s/prod/*.yaml
    verify:
      - name: health-check
        container:
          name: health-check
          image: curlimages/curl:latest
          command: ["sh"]
          args: ["-c", "curl -f https://my-app.example.com/health || exit 1"]
deploy:
  kubectl: {}
```

### Verification with Environment Variables

```yaml
verify:
  - name: api-test
    container:
      name: api-test
      image: us-central1-docker.pkg.dev/my-project/repo/api-tests:latest
      command: ["./run-tests.sh"]
      env:
        - name: TARGET_URL
          value: "https://staging.my-app.example.com"
        - name: TEST_TIMEOUT
          value: "120"
```

---

## Part 9 — Rollbacks

### Rollback Methods

```
┌──────────────────────────────────────────────────────────────┐
│                    ROLLBACK OPTIONS                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Method 1: Roll back to previous release                      │
│  ┌──────────────────────────────────────┐                    │
│  │  gcloud deploy targets rollback      │                    │
│  │  • Deploys the previous successful   │                    │
│  │    release to the target             │                    │
│  └──────────────────────────────────────┘                    │
│                                                                │
│  Method 2: Create new release with old image                  │
│  ┌──────────────────────────────────────┐                    │
│  │  gcloud deploy releases create       │                    │
│  │  --images=app:old-image-tag          │                    │
│  │  • Creates fresh release with known  │                    │
│  │    good version                      │                    │
│  └──────────────────────────────────────┘                    │
│                                                                │
│  Method 3: Canary rollback (halt + rollback)                  │
│  ┌──────────────────────────────────────┐                    │
│  │  gcloud deploy rollouts cancel       │                    │
│  │  • Stops canary at current phase     │                    │
│  │  • Routes 100% to stable version     │                    │
│  └──────────────────────────────────────┘                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Rolling Back a Target

```bash
# Roll back to previous successful release
gcloud deploy targets rollback prod \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Roll back to a specific release
gcloud deploy targets rollback prod \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1 \
    --release=release-001
```

### Cancelling an In-Progress Rollout

```bash
# Cancel a rollout (useful during canary)
gcloud deploy rollouts cancel rollout-003 \
    --release=release-002 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1
```

### Advancing/Retrying a Rollout Phase

```bash
# Advance canary to next phase manually
gcloud deploy rollouts advance rollout-003 \
    --release=release-002 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1 \
    --phase-id=canary-50

# Retry a failed rollout phase
gcloud deploy rollouts retry-job rollout-003 \
    --release=release-002 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1 \
    --phase-id=canary-10 \
    --job-id=deploy
```

---

## Part 10 — Automation & Hooks

### Deploy Hooks (Pre/Post Actions)

```yaml
# skaffold.yaml with deploy hooks
apiVersion: skaffold/v4beta7
kind: Config
metadata:
  name: my-app
profiles:
  - name: prod
    manifests:
      rawYaml:
        - k8s/prod/*.yaml
    deploy:
      kubectl:
        hooks:
          before:
            - host:
                command: ["sh", "-c", "echo 'Pre-deploy: draining connections'"]
            - container:
                name: db-migrate
                image: us-central1-docker.pkg.dev/my-project/repo/migrator:latest
                command: ["./migrate.sh", "--env=prod"]
          after:
            - host:
                command: ["sh", "-c", "echo 'Post-deploy: warming caches'"]
```

### Automation Rules

Cloud Deploy Automation allows defining rules that automatically trigger actions:

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: Automation
metadata:
  name: auto-promote-staging
description: "Auto-promote successful dev rollouts to staging"
selector:
  targets:
    - id: dev
rules:
  - promoteReleaseRule:
      name: promote-to-staging
      targetId: staging
      wait: 5m   # Wait 5 minutes after success before promoting
---
apiVersion: deploy.cloud.google.com/v1
kind: Automation
metadata:
  name: auto-advance-canary
description: "Auto-advance canary phases"
selector:
  targets:
    - id: prod
rules:
  - advanceRolloutRule:
      name: advance-canary
      sourcePhases: ["canary-10", "canary-25"]
      wait: 10m
```

### Automation for Repair (Auto-Rollback)

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: Automation
metadata:
  name: auto-repair-prod
description: "Auto-rollback failed prod deployments"
selector:
  targets:
    - id: prod
rules:
  - repairRolloutRule:
      name: auto-rollback
      repairPhases:
        - retry:
            attempts: 1
            wait: 2m
        - rollback: {}
```

### Registering Automation

```bash
# Apply automation rules
gcloud deploy apply --file=automation.yaml \
    --region=us-central1 \
    --project=my-project

# List automations
gcloud deploy automations list \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Delete automation
gcloud deploy automations delete auto-promote-staging \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1
```

---

## Part 11 — Notifications & Integrations

### Pub/Sub Notifications

Cloud Deploy publishes events to Pub/Sub topics automatically:

```
┌──────────────────────────────────────────────────────────────┐
│               NOTIFICATION ARCHITECTURE                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Cloud Deploy                                                  │
│       │                                                        │
│       ▼                                                        │
│  ┌──────────────────────────────┐                             │
│  │  Pub/Sub Topics (auto)       │                             │
│  │                              │                             │
│  │  • clouddeploy-resources     │── Pipeline/target changes   │
│  │  • clouddeploy-operations    │── Release/rollout events    │
│  │  • clouddeploy-approvals     │── Approval requests         │
│  └──────────────┬───────────────┘                             │
│                 │                                              │
│       ┌─────────┼─────────┐                                   │
│       ▼         ▼         ▼                                   │
│  ┌────────┐ ┌────────┐ ┌──────────┐                         │
│  │ Cloud  │ │ Slack  │ │ Custom   │                         │
│  │Function│ │Webhook │ │ Handler  │                         │
│  └────────┘ └────────┘ └──────────┘                         │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Pub/Sub Topic Structure

| Topic | Events Published |
|-------|-----------------|
| `clouddeploy-resources` | Pipeline created/updated/deleted, target changes |
| `clouddeploy-operations` | Release created, rollout started/succeeded/failed |
| `clouddeploy-approvals` | Rollout pending approval |

### Subscribing to Notifications

```bash
# Create subscription for approvals
gcloud pubsub subscriptions create deploy-approval-sub \
    --topic=clouddeploy-approvals \
    --push-endpoint=https://my-webhook.example.com/approvals

# Create subscription for operations
gcloud pubsub subscriptions create deploy-ops-sub \
    --topic=clouddeploy-operations \
    --message-filter='attributes.Action="Succeed"'
```

### Cloud Function for Slack Notification

```python
# Cloud Function triggered by Pub/Sub
import base64
import json
import requests

def notify_slack(event, context):
    """Send Cloud Deploy events to Slack."""
    pubsub_message = base64.b64decode(event['data']).decode('utf-8')
    message = json.loads(pubsub_message)
    
    action = event.get('attributes', {}).get('Action', 'Unknown')
    resource = event.get('attributes', {}).get('Resource', 'Unknown')
    
    slack_message = {
        "text": f":rocket: Cloud Deploy: {action} on {resource}",
        "blocks": [
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*Cloud Deploy Event*\n"
                            f"Action: `{action}`\n"
                            f"Resource: `{resource}`"
                }
            }
        ]
    }
    
    webhook_url = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
    requests.post(webhook_url, json=slack_message)
```

### Integration with Cloud Build Triggers

```yaml
# cloudbuild.yaml — Trigger Cloud Deploy release after build
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', '${_IMAGE}:${SHORT_SHA}', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '${_IMAGE}:${SHORT_SHA}']
  # Create Cloud Deploy release
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'deploy'
      - 'releases'
      - 'create'
      - 'release-${SHORT_SHA}'
      - '--delivery-pipeline=my-app-pipeline'
      - '--region=us-central1'
      - '--images=my-app=${_IMAGE}:${SHORT_SHA}'
substitutions:
  _IMAGE: us-central1-docker.pkg.dev/my-project/repo/my-app
```

---

## Part 12 — IAM & Security

### Cloud Deploy IAM Roles

| Role | Description | Typical User |
|------|-------------|--------------|
| `roles/clouddeploy.admin` | Full control over all resources | Platform admin |
| `roles/clouddeploy.developer` | Create/manage pipelines & releases | Developers |
| `roles/clouddeploy.operator` | Create releases, promote, manage rollouts | CI/CD systems |
| `roles/clouddeploy.approver` | Approve/reject rollouts | Release managers |
| `roles/clouddeploy.viewer` | Read-only access | Auditors |
| `roles/clouddeploy.customTargetTypeAdmin` | Manage custom target types | Platform team |

### Service Account Configuration

```
┌──────────────────────────────────────────────────────────────┐
│              SERVICE ACCOUNT ARCHITECTURE                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────┐                 │
│  │        Cloud Deploy Service Agent         │                 │
│  │  (service-PROJECT_NUM@gcp-sa-             │                 │
│  │   clouddeploy.iam.gserviceaccount.com)    │                 │
│  │                                           │                 │
│  │  Needs:                                   │                 │
│  │  • actAs on execution SA                  │                 │
│  │  • cloudbuild.builds.create               │                 │
│  └──────────────────────────────────────────┘                 │
│                                                                │
│  ┌──────────────────────────────────────────┐                 │
│  │        Execution Service Account          │                 │
│  │  (deploy-sa@PROJECT.iam.gserviceaccount)  │                 │
│  │                                           │                 │
│  │  Needs:                                   │                 │
│  │  • container.developer (for GKE)          │                 │
│  │  • run.developer (for Cloud Run)          │                 │
│  │  • storage.objectViewer (for artifacts)   │                 │
│  │  • logging.logWriter                      │                 │
│  └──────────────────────────────────────────┘                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Setting Up Service Accounts

```bash
# Create execution service account
gcloud iam service-accounts create cloud-deploy-exec \
    --display-name="Cloud Deploy Execution SA"

# Grant GKE deploy permissions
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:cloud-deploy-exec@my-project.iam.gserviceaccount.com" \
    --role="roles/container.developer"

# Grant Cloud Run deploy permissions
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:cloud-deploy-exec@my-project.iam.gserviceaccount.com" \
    --role="roles/run.developer"

# Allow Cloud Deploy service agent to actAs the execution SA
gcloud iam service-accounts add-iam-policy-binding \
    cloud-deploy-exec@my-project.iam.gserviceaccount.com \
    --member="serviceAccount:service-PROJECT_NUM@gcp-sa-clouddeploy.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountUser"

# Grant Cloud Build permissions to execution SA
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:cloud-deploy-exec@my-project.iam.gserviceaccount.com" \
    --role="roles/cloudbuild.workerPoolUser"
```

### Execution Environment Configuration

```yaml
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod
gke:
  cluster: projects/my-project/locations/us-central1/clusters/prod
executionConfigs:
  - usages:
      - RENDER
      - DEPLOY
      - VERIFY
      - PREDEPLOY
      - POSTDEPLOY
    serviceAccount: cloud-deploy-exec@my-project.iam.gserviceaccount.com
    artifactStorage: gs://my-deploy-artifacts
    executionTimeout: 1800s   # 30 minutes
```

### Audit Logging

Cloud Deploy writes Admin Activity logs automatically and Data Access logs when enabled:

```bash
# View deploy audit logs
gcloud logging read \
    'resource.type="clouddeploy.googleapis.com/DeliveryPipeline"' \
    --limit=50 \
    --format=json
```

---

## Part 13 — Cloud Run Deployments

### Cloud Run Delivery Pipeline

```yaml
# clouddeploy.yaml for Cloud Run
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: cloudrun-pipeline
description: "Cloud Run delivery pipeline"
serialPipeline:
  stages:
    - targetId: run-dev
      profiles: [dev]
    - targetId: run-staging
      profiles: [staging]
    - targetId: run-prod
      profiles: [prod]
      strategy:
        canary:
          runtimeConfig:
            cloudRun:
              automaticTrafficControl: true
          canaryDeployment:
            percentages: [25, 50, 75]
            verify: true
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: run-dev
run:
  location: projects/my-project/locations/us-central1
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: run-staging
run:
  location: projects/my-project/locations/us-central1
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: run-prod
requireApproval: true
run:
  location: projects/my-project/locations/us-central1
```

### Cloud Run Service Manifest

```yaml
# k8s/prod/service.yaml (Cloud Run uses Knative-style YAML)
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-cloudrun-app
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "100"
        autoscaling.knative.dev/minScale: "2"
    spec:
      serviceAccountName: app-sa@my-project.iam.gserviceaccount.com
      containers:
        - image: my-app  # Placeholder — replaced by Cloud Deploy
          ports:
            - containerPort: 8080
          resources:
            limits:
              memory: "512Mi"
              cpu: "1"
          env:
            - name: ENV
              value: "production"
```

### Skaffold for Cloud Run

```yaml
# skaffold.yaml for Cloud Run
apiVersion: skaffold/v4beta7
kind: Config
metadata:
  name: cloudrun-app
profiles:
  - name: dev
    manifests:
      rawYaml:
        - service-dev.yaml
  - name: staging
    manifests:
      rawYaml:
        - service-staging.yaml
  - name: prod
    manifests:
      rawYaml:
        - service-prod.yaml
    verify:
      - name: smoke-test
        container:
          name: smoke
          image: curlimages/curl:latest
          command: ["sh", "-c", "curl -f $CLOUD_RUN_SERVICE_URLS/health"]
deploy:
  cloudrun: {}
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform — Delivery Pipeline

```hcl
# Enable Cloud Deploy API
resource "google_project_service" "clouddeploy" {
  service = "clouddeploy.googleapis.com"
}

# Delivery Pipeline
resource "google_clouddeploy_delivery_pipeline" "pipeline" {
  name     = "my-app-pipeline"
  location = "us-central1"
  project  = var.project_id

  description = "Production delivery pipeline"

  serial_pipeline {
    stages {
      target_id = google_clouddeploy_target.dev.name
      profiles  = ["dev"]
    }
    stages {
      target_id = google_clouddeploy_target.staging.name
      profiles  = ["staging"]
    }
    stages {
      target_id = google_clouddeploy_target.prod.name
      profiles  = ["prod"]
      strategy {
        standard {
          verify = true
        }
      }
    }
  }

  labels = {
    team        = "platform"
    environment = "all"
  }
}
```

### Terraform — GKE Targets

```hcl
# Dev Target
resource "google_clouddeploy_target" "dev" {
  name     = "dev"
  location = "us-central1"
  project  = var.project_id

  description      = "Development GKE cluster"
  require_approval = false

  gke {
    cluster = "projects/${var.project_id}/locations/us-central1/clusters/dev-cluster"
  }

  execution_configs {
    usages          = ["RENDER", "DEPLOY"]
    service_account = google_service_account.deploy_exec.email
  }
}

# Staging Target
resource "google_clouddeploy_target" "staging" {
  name     = "staging"
  location = "us-central1"
  project  = var.project_id

  description      = "Staging GKE cluster"
  require_approval = false

  gke {
    cluster = "projects/${var.project_id}/locations/us-central1/clusters/staging-cluster"
  }

  execution_configs {
    usages          = ["RENDER", "DEPLOY", "VERIFY"]
    service_account = google_service_account.deploy_exec.email
  }
}

# Prod Target (with approval)
resource "google_clouddeploy_target" "prod" {
  name     = "prod"
  location = "us-central1"
  project  = var.project_id

  description      = "Production GKE cluster"
  require_approval = true

  gke {
    cluster = "projects/${var.project_id}/locations/us-east1/clusters/prod-cluster"
  }

  execution_configs {
    usages            = ["RENDER", "DEPLOY", "VERIFY"]
    service_account   = google_service_account.deploy_exec.email
    execution_timeout = "1800s"
  }
}
```

### Terraform — Cloud Run Targets

```hcl
resource "google_clouddeploy_target" "run_prod" {
  name     = "run-prod"
  location = "us-central1"
  project  = var.project_id

  description      = "Production Cloud Run"
  require_approval = true

  run {
    location = "projects/${var.project_id}/locations/us-central1"
  }

  execution_configs {
    usages          = ["RENDER", "DEPLOY", "VERIFY"]
    service_account = google_service_account.deploy_exec.email
  }
}
```

### Terraform — Canary Pipeline

```hcl
resource "google_clouddeploy_delivery_pipeline" "canary_pipeline" {
  name     = "canary-pipeline"
  location = "us-central1"
  project  = var.project_id

  serial_pipeline {
    stages {
      target_id = google_clouddeploy_target.dev.name
      profiles  = ["dev"]
    }
    stages {
      target_id = google_clouddeploy_target.prod.name
      profiles  = ["prod"]
      strategy {
        canary {
          runtime_config {
            kubernetes {
              service_networking {
                service    = "my-app-service"
                deployment = "my-app"
              }
            }
          }
          canary_deployment {
            percentages = [10, 25, 50, 75]
            verify      = true
          }
        }
      }
    }
  }
}
```

### Terraform — Automation

```hcl
resource "google_clouddeploy_automation" "auto_promote" {
  name              = "auto-promote-staging"
  location          = "us-central1"
  project           = var.project_id
  delivery_pipeline = google_clouddeploy_delivery_pipeline.pipeline.name

  description     = "Auto-promote to staging after dev success"
  service_account = google_service_account.deploy_exec.email

  selector {
    targets {
      id = "dev"
    }
  }

  rules {
    promote_release_rule {
      id        = "promote-to-staging"
      target_id = "staging"
      wait      = "300s"
    }
  }
}

resource "google_clouddeploy_automation" "auto_repair" {
  name              = "auto-repair-prod"
  location          = "us-central1"
  project           = var.project_id
  delivery_pipeline = google_clouddeploy_delivery_pipeline.pipeline.name

  description     = "Auto-repair failed prod deployments"
  service_account = google_service_account.deploy_exec.email

  selector {
    targets {
      id = "prod"
    }
  }

  rules {
    repair_rollout_rule {
      id = "auto-rollback"
      repair_phases {
        retry {
          attempts = 1
          wait     = "120s"
        }
      }
      repair_phases {
        rollback {}
      }
    }
  }
}
```

### Terraform — Service Account

```hcl
resource "google_service_account" "deploy_exec" {
  account_id   = "cloud-deploy-exec"
  display_name = "Cloud Deploy Execution SA"
  project      = var.project_id
}

# GKE permissions
resource "google_project_iam_member" "deploy_gke" {
  project = var.project_id
  role    = "roles/container.developer"
  member  = "serviceAccount:${google_service_account.deploy_exec.email}"
}

# Cloud Run permissions
resource "google_project_iam_member" "deploy_run" {
  project = var.project_id
  role    = "roles/run.developer"
  member  = "serviceAccount:${google_service_account.deploy_exec.email}"
}

# Allow Cloud Deploy agent to act as execution SA
resource "google_service_account_iam_member" "deploy_agent_actas" {
  service_account_id = google_service_account.deploy_exec.name
  role               = "roles/iam.serviceAccountUser"
  member             = "serviceAccount:service-${data.google_project.current.number}@gcp-sa-clouddeploy.iam.gserviceaccount.com"
}

# Logging
resource "google_project_iam_member" "deploy_logging" {
  project = var.project_id
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.deploy_exec.email}"
}
```

### gcloud CLI Reference

```bash
# ─── Pipeline Management ─────────────────────────────────────
# Apply pipeline configuration
gcloud deploy apply --file=clouddeploy.yaml --region=us-central1

# List pipelines
gcloud deploy delivery-pipelines list --region=us-central1

# Describe pipeline
gcloud deploy delivery-pipelines describe my-app-pipeline --region=us-central1

# Delete pipeline
gcloud deploy delivery-pipelines delete my-app-pipeline --region=us-central1

# ─── Target Management ────────────────────────────────────────
# List targets
gcloud deploy targets list --region=us-central1

# Describe target
gcloud deploy targets describe prod --region=us-central1

# Delete target
gcloud deploy targets delete prod --region=us-central1

# ─── Release Management ───────────────────────────────────────
# Create release
gcloud deploy releases create release-v1 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1 \
    --source=./ \
    --images=my-app=us-central1-docker.pkg.dev/proj/repo/app:v1

# List releases
gcloud deploy releases list \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Describe release
gcloud deploy releases describe release-v1 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Promote release
gcloud deploy releases promote \
    --release=release-v1 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# ─── Rollout Management ───────────────────────────────────────
# List rollouts
gcloud deploy rollouts list \
    --release=release-v1 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Approve rollout
gcloud deploy rollouts approve rollout-001 \
    --release=release-v1 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Reject rollout
gcloud deploy rollouts reject rollout-001 \
    --release=release-v1 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Cancel rollout
gcloud deploy rollouts cancel rollout-001 \
    --release=release-v1 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Rollback target
gcloud deploy targets rollback prod \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Advance canary phase
gcloud deploy rollouts advance rollout-001 \
    --release=release-v1 \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# ─── Automation Management ────────────────────────────────────
# List automations
gcloud deploy automations list \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Describe automation
gcloud deploy automations describe auto-promote-staging \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1

# Delete automation
gcloud deploy automations delete auto-promote-staging \
    --delivery-pipeline=my-app-pipeline \
    --region=us-central1
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Full CI/CD Pipeline (Build → Deploy → Promote)

```
┌──────────────────────────────────────────────────────────────────────┐
│         PATTERN 1: END-TO-END CI/CD WITH CLOUD DEPLOY                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Developer pushes code                                                 │
│       │                                                                │
│       ▼                                                                │
│  ┌──────────────┐    ┌───────────────┐    ┌──────────────────┐       │
│  │  Cloud Build │───▶│   Artifact    │───▶│   Cloud Deploy   │       │
│  │  (CI)        │    │   Registry    │    │   (CD)           │       │
│  │              │    │               │    │                  │       │
│  │ • Lint       │    │ • Docker img  │    │ • Create release │       │
│  │ • Test       │    │ • Scan vuln   │    │ • Render         │       │
│  │ • Build      │    │ • Sign        │    │ • Deploy to dev  │       │
│  │ • Push       │    │               │    │ • Auto-promote   │       │
│  └──────────────┘    └───────────────┘    │   to staging     │       │
│                                           │ • Verify          │       │
│                                           │ • Approval gate   │       │
│                                           │ • Deploy to prod  │       │
│                                           │ • Canary 10→50→100│       │
│                                           └──────────────────┘       │
│                                                                        │
│  Pipeline Definition:                                                  │
│  cloudbuild.yaml → build/test/push → create Cloud Deploy release      │
│  clouddeploy.yaml → dev (auto) → staging (verify) → prod (canary)    │
│                                                                        │
│  Key files:                                                            │
│  ├── cloudbuild.yaml          (CI pipeline)                           │
│  ├── clouddeploy.yaml         (CD pipeline + targets)                 │
│  ├── skaffold.yaml            (render + deploy config)                │
│  ├── automation.yaml          (auto-promote + auto-repair)            │
│  └── k8s/                                                             │
│      ├── base/                (shared manifests)                      │
│      ├── overlays/dev/        (dev customization)                     │
│      ├── overlays/staging/    (staging customization)                 │
│      └── overlays/prod/       (prod customization)                    │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```yaml
# cloudbuild.yaml — CI triggers Cloud Deploy
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', '${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_APP}:${SHORT_SHA}', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_APP}:${SHORT_SHA}']
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'deploy'
      - 'releases'
      - 'create'
      - 'rel-${SHORT_SHA}'
      - '--delivery-pipeline=${_PIPELINE}'
      - '--region=${_REGION}'
      - '--source=./'
      - '--images=${_APP}=${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_APP}:${SHORT_SHA}'
substitutions:
  _REGION: us-central1
  _REPO: docker-repo
  _APP: my-app
  _PIPELINE: my-app-pipeline
```

### Pattern 2: Multi-Region Progressive Delivery

```
┌──────────────────────────────────────────────────────────────────────┐
│       PATTERN 2: MULTI-REGION PROGRESSIVE DELIVERY                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────┐                                                         │
│  │ Release  │                                                         │
│  └────┬─────┘                                                         │
│       │                                                                │
│       ▼                                                                │
│  ┌──────────┐     Auto-promote                                        │
│  │   Dev    │─────────────────┐                                       │
│  │ (canary) │                 │                                       │
│  └──────────┘                 ▼                                       │
│                          ┌──────────┐     Approval                    │
│                          │ Staging  │────────────────┐                │
│                          │ (verify) │                │                │
│                          └──────────┘                ▼                │
│                                                ┌──────────┐           │
│                                                │ Prod     │           │
│                                                │ Canary   │           │
│                                                │ (multi)  │           │
│                                                └────┬─────┘           │
│                                                     │                  │
│                                    ┌────────────────┼────────────┐    │
│                                    ▼                ▼            ▼    │
│                              ┌──────────┐    ┌──────────┐ ┌────────┐ │
│                              │ US-EAST  │    │ US-WEST  │ │ EU     │ │
│                              │ (canary  │    │ (canary  │ │(canary │ │
│                              │  10→100) │    │  10→100) │ │ 10→100)│ │
│                              └──────────┘    └──────────┘ └────────┘ │
│                                                                        │
│  Uses multi-target for parallel regional deployment                   │
│  Each region runs independent canary with verification                │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```yaml
# clouddeploy.yaml — Multi-region
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: multi-region-pipeline
serialPipeline:
  stages:
    - targetId: dev
      profiles: [dev]
    - targetId: staging
      profiles: [staging]
      strategy:
        standard:
          verify: true
    - targetId: prod-multi
      profiles: [prod]
      strategy:
        canary:
          runtimeConfig:
            kubernetes:
              serviceNetworking:
                service: "my-app-svc"
                deployment: "my-app"
          canaryDeployment:
            percentages: [10, 50]
            verify: true
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod-multi
requireApproval: true
multiTarget:
  targetIds:
    - prod-us-east1
    - prod-us-west1
    - prod-europe-west1
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod-us-east1
gke:
  cluster: projects/my-project/locations/us-east1/clusters/prod-east
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod-us-west1
gke:
  cluster: projects/my-project/locations/us-west1/clusters/prod-west
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod-europe-west1
gke:
  cluster: projects/my-project/locations/europe-west1/clusters/prod-eu
```

### Pattern 3: Microservices with Shared Pipeline Template

```
┌──────────────────────────────────────────────────────────────────────┐
│       PATTERN 3: MICROSERVICES — PIPELINE PER SERVICE                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │                  Shared Infrastructure                       │     │
│  │  • Same GKE clusters (namespaced per service)               │     │
│  │  • Common execution SA with per-namespace RBAC              │     │
│  │  • Shared automation rules (promote + repair)               │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │ user-svc    │  │ order-svc   │  │ payment-svc │                  │
│  │ pipeline    │  │ pipeline    │  │ pipeline    │                  │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤                  │
│  │ dev → stg → │  │ dev → stg → │  │ dev → stg → │                  │
│  │ prod        │  │ prod        │  │ prod (canary)│                  │
│  │ (standard)  │  │ (standard)  │  │ (approval)  │                  │
│  └─────────────┘  └─────────────┘  └─────────────┘                  │
│                                                                        │
│  Terraform module pattern:                                            │
│  module "user_svc_pipeline" {                                         │
│    source    = "./modules/deploy-pipeline"                            │
│    name      = "user-svc"                                             │
│    namespace = "user-service"                                         │
│    strategy  = "standard"                                             │
│  }                                                                    │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Terraform Module:**

```hcl
# modules/deploy-pipeline/main.tf
variable "name" {}
variable "project_id" {}
variable "region" {}
variable "dev_cluster" {}
variable "staging_cluster" {}
variable "prod_cluster" {}
variable "require_prod_approval" { default = true }
variable "canary_percentages" { default = [] }

resource "google_clouddeploy_delivery_pipeline" "pipeline" {
  name     = "${var.name}-pipeline"
  location = var.region
  project  = var.project_id

  serial_pipeline {
    stages {
      target_id = google_clouddeploy_target.dev.name
      profiles  = ["dev"]
    }
    stages {
      target_id = google_clouddeploy_target.staging.name
      profiles  = ["staging"]
      strategy {
        standard {
          verify = true
        }
      }
    }
    stages {
      target_id = google_clouddeploy_target.prod.name
      profiles  = ["prod"]

      dynamic "strategy" {
        for_each = length(var.canary_percentages) > 0 ? [1] : []
        content {
          canary {
            canary_deployment {
              percentages = var.canary_percentages
              verify      = true
            }
          }
        }
      }
    }
  }
}

resource "google_clouddeploy_target" "dev" {
  name     = "${var.name}-dev"
  location = var.region
  project  = var.project_id
  gke { cluster = var.dev_cluster }
}

resource "google_clouddeploy_target" "staging" {
  name     = "${var.name}-staging"
  location = var.region
  project  = var.project_id
  gke { cluster = var.staging_cluster }
}

resource "google_clouddeploy_target" "prod" {
  name             = "${var.name}-prod"
  location         = var.region
  project          = var.project_id
  require_approval = var.require_prod_approval
  gke { cluster = var.prod_cluster }
}
```

**Usage:**

```hcl
module "user_svc" {
  source          = "./modules/deploy-pipeline"
  name            = "user-svc"
  project_id      = var.project_id
  region          = "us-central1"
  dev_cluster     = "projects/${var.project_id}/locations/us-central1/clusters/dev"
  staging_cluster = "projects/${var.project_id}/locations/us-central1/clusters/staging"
  prod_cluster    = "projects/${var.project_id}/locations/us-east1/clusters/prod"
}

module "payment_svc" {
  source               = "./modules/deploy-pipeline"
  name                 = "payment-svc"
  project_id           = var.project_id
  region               = "us-central1"
  dev_cluster          = "projects/${var.project_id}/locations/us-central1/clusters/dev"
  staging_cluster      = "projects/${var.project_id}/locations/us-central1/clusters/staging"
  prod_cluster         = "projects/${var.project_id}/locations/us-east1/clusters/prod"
  canary_percentages   = [10, 25, 50, 75]
}
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Apply pipeline | `gcloud deploy apply --file=clouddeploy.yaml --region=REGION` |
| Create release | `gcloud deploy releases create NAME --delivery-pipeline=PIPE --region=REGION --source=./` |
| Promote | `gcloud deploy releases promote --release=NAME --delivery-pipeline=PIPE --region=REGION` |
| Approve | `gcloud deploy rollouts approve NAME --release=REL --delivery-pipeline=PIPE --region=REGION` |
| Rollback | `gcloud deploy targets rollback TARGET --delivery-pipeline=PIPE --region=REGION` |
| List pipelines | `gcloud deploy delivery-pipelines list --region=REGION` |
| List releases | `gcloud deploy releases list --delivery-pipeline=PIPE --region=REGION` |
| Cancel rollout | `gcloud deploy rollouts cancel NAME --release=REL --delivery-pipeline=PIPE --region=REGION` |
| Advance canary | `gcloud deploy rollouts advance NAME --release=REL --delivery-pipeline=PIPE --region=REGION` |

---

## What is Continuous Delivery? (Beginner Explanation)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONTINUOUS DELIVERY — THE SIMPLE EXPLANATION                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 💡 Continuous Delivery (CD) automatically moves your code from      │
│    dev → staging → production — so you don't have to manually      │
│    SSH into servers and deploy things yourself.                      │
│                                                                       │
│ Think of it like a conveyor belt in a factory:                      │
│ ├── Your code gets built (CI — Continuous Integration)              │
│ ├── The built artifact (Docker image) lands on the conveyor belt   │
│ ├── It automatically moves through quality checkpoints             │
│ │   (dev → staging → prod)                                         │
│ └── At each checkpoint, someone can approve or reject              │
│                                                                       │
│ Without CD:                                                         │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ 1. Developer finishes code                                    │   │
│ │ 2. Manually builds Docker image                               │   │
│ │ 3. Manually deploys to dev cluster — tests it                │   │
│ │ 4. Manually deploys to staging — tests again                 │   │
│ │ 5. Manually deploys to production — hopes nothing breaks     │   │
│ │ 6. ⚠️ Error-prone, slow, inconsistent                        │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ With CD (Cloud Deploy):                                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ 1. Developer pushes code → Cloud Build creates image         │   │
│ │ 2. Cloud Deploy auto-deploys to dev — runs tests             │   │
│ │ 3. Auto-promotes to staging — runs more tests                │   │
│ │ 4. Waits for approval → one-click promote to production     │   │
│ │ 5. ✅ Consistent, auditable, rollback-ready                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ How Cloud Deploy fits in the full pipeline:                         │
│                                                                       │
│   ┌────────┐    ┌────────────┐    ┌───────────────────────┐        │
│   │  Git   │───▶│ Cloud Build│───▶│    Cloud Deploy       │        │
│   │ (code) │    │ (CI: build │    │    (CD: promote       │        │
│   └────────┘    │  + test)   │    │     through envs)     │        │
│                 └────────────┘    └───────┬───────────────┘        │
│                                          │                          │
│                    ┌─────────────────────┼─────────────────┐       │
│                    ▼                     ▼                 ▼       │
│               ┌─────────┐         ┌──────────┐      ┌────────┐   │
│               │   Dev   │────────▶│ Staging  │─────▶│  Prod  │   │
│               │ (auto)  │ promote │ (auto)   │approve│(gated) │   │
│               └─────────┘         └──────────┘      └────────┘   │
│                                                                       │
│ ⚡ Cloud Deploy handles ONLY the CD part (deployment + promotion). │
│ ⚡ Cloud Build handles the CI part (building + testing).           │
│ ⚡ Together they form a complete CI/CD pipeline.                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Managing Releases & Rollouts

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONSOLE → Cloud Deploy → Delivery Pipelines                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ VIEWING PIPELINE STATUS                                             │
│ ───────────────────────                                             │
│                                                                       │
│ 1. Go to: Console → Cloud Deploy → Delivery pipelines              │
│ 2. You see all your pipelines:                                      │
│    ┌──────────────────────────────────────────────────────────┐    │
│    │ Pipeline            │ Last release  │ Status             │    │
│    ├─────────────────────┼───────────────┼────────────────────┤    │
│    │ my-app-pipeline     │ release-005   │ ✅ Deployed to prod │    │
│    │ payment-svc-pipeline│ release-012   │ ⏳ Pending approval │    │
│    └──────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 3. Click a pipeline → see the visual promotion flow:               │
│    ┌─────────────────────────────────────────────────────────┐     │
│    │                                                          │     │
│    │   ┌─────┐    ┌─────────┐    ┌──────┐                   │     │
│    │   │ Dev │───▶│ Staging │───▶│ Prod │                   │     │
│    │   │ ✅   │    │ ✅       │    │ ⏳    │                   │     │
│    │   └─────┘    └─────────┘    └──────┘                   │     │
│    │                                                          │     │
│    │   release-005: v1.3.0                                   │     │
│    │   Last updated: 15 minutes ago                          │     │
│    └─────────────────────────────────────────────────────────┘     │
│                                                                       │
│ 4. Click a target (e.g., "Staging") → see rollout history          │
│    ┌──────────────────────────────────────────────────────────┐    │
│    │ Rollout              │ Release      │ Status  │ Time    │    │
│    ├──────────────────────┼──────────────┼─────────┼─────────┤    │
│    │ rollout-staging-001  │ release-005  │ ✅ Done  │ 3m ago  │    │
│    │ rollout-staging-002  │ release-004  │ ✅ Done  │ 1d ago  │    │
│    │ rollout-staging-003  │ release-003  │ ❌ Failed│ 3d ago  │    │
│    └──────────────────────────────────────────────────────────┘    │
│                                                                       │
│                                                                       │
│ APPROVING A ROLLOUT                                                 │
│ ───────────────────                                                 │
│                                                                       │
│ When a target has "requireApproval: true":                          │
│ 1. Navigate to the pipeline → click the target with ⏳ status      │
│ 2. You'll see a banner:                                             │
│    ┌──────────────────────────────────────────────────────────┐    │
│    │ ⏳ Rollout "rollout-prod-001" is pending approval         │    │
│    │                                                           │    │
│    │ Release: release-005 (v1.3.0)                            │    │
│    │ Target: prod                                              │    │
│    │ Requested by: dev@company.com                             │    │
│    │                                                           │    │
│    │ [APPROVE]  [REJECT]                                       │    │
│    └──────────────────────────────────────────────────────────┘    │
│ 3. Click [APPROVE] → add an optional comment → confirm             │
│ 4. The rollout begins automatically after approval                  │
│    ⚠️ You need the "clouddeploy.approver" IAM role to approve     │
│                                                                       │
│                                                                       │
│ ROLLING BACK FROM CONSOLE                                           │
│ ─────────────────────────                                           │
│                                                                       │
│ If a deployment goes wrong and you need to revert:                  │
│ 1. Navigate to the pipeline → click the target (e.g., "Prod")     │
│ 2. In the rollout history, find the last successful rollout         │
│ 3. Click [ROLLBACK] button at the top of the target view           │
│    ┌──────────────────────────────────────────────────────────┐    │
│    │ Rollback target "prod"                                    │    │
│    │                                                           │    │
│    │ Current release: release-005 (v1.3.0) ❌ issues           │    │
│    │ Rollback to:     release-004 (v1.2.4) ✅ was stable       │    │
│    │                                                           │    │
│    │ This will create a new rollout deploying release-004     │    │
│    │ to the "prod" target.                                    │    │
│    │                                                           │    │
│    │ [ROLLBACK]  [CANCEL]                                      │    │
│    └──────────────────────────────────────────────────────────┘    │
│ 4. Click [ROLLBACK] → a new rollout is created with the previous   │
│    release                                                          │
│ 5. Monitor the rollback rollout until status = SUCCEEDED            │
│    ⚡ Rollback creates a NEW rollout — it doesn't delete the        │
│       failed one. Full audit trail is preserved.                    │
│                                                                       │
│                                                                       │
│ DELETING PIPELINE AND TARGETS                                       │
│ ─────────────────────────────                                       │
│                                                                       │
│ Delete a delivery pipeline:                                         │
│ 1. Go to: Console → Cloud Deploy → Delivery pipelines              │
│ 2. Click the three-dot menu (⋮) next to the pipeline               │
│ 3. Select "Delete"                                                 │
│ 4. Type the pipeline name to confirm                               │
│    ⚠️ You must delete all releases first — pipeline with active    │
│       releases cannot be deleted                                    │
│    ⚠️ This does NOT delete the targets — they are separate         │
│                                                                       │
│ Delete a target:                                                    │
│ 1. Go to: Console → Cloud Deploy → Targets                        │
│ 2. Click the three-dot menu (⋮) next to the target                 │
│ 3. Select "Delete"                                                 │
│ 4. Confirm deletion                                                │
│    ⚠️ Target must not be referenced by any active pipeline         │
│    ⚠️ This does NOT delete the actual GKE cluster or Cloud Run     │
│       service — only the Cloud Deploy target definition            │
│                                                                       │
│ Delete releases (cleanup old releases):                             │
│ 1. Navigate to the pipeline → "Releases" tab                      │
│ 2. Check the boxes next to old releases                            │
│ 3. Click [DELETE]                                                  │
│ 4. Confirm → removes release metadata and rendered manifests       │
│    ⚡ Good practice: periodically clean up old releases to          │
│       reduce clutter and storage costs                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 34: Deployment Manager & Terraform** → `34-deployment-manager.md`
