# Chapter 31: Cloud Build

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud Build Fundamentals](#part-1-cloud-build-fundamentals)
- [Part 2: Architecture & How It Works](#part-2-architecture--how-it-works)
- [Part 3: Build Triggers](#part-3-build-triggers)
- [Part 4: cloudbuild.yaml — Build Configuration](#part-4-cloudbuildyaml--build-configuration)
- [Part 5: Build Steps & Builders](#part-5-build-steps--builders)
- [Part 6: Substitutions & Variables](#part-6-substitutions--variables)
- [Part 7: Building Docker Images](#part-7-building-docker-images)
- [Part 8: Artifacts & Storage](#part-8-artifacts--storage)
- [Part 9: Source Connections — GitHub, Bitbucket, GitLab](#part-9-source-connections--github-bitbucket-gitlab)
- [Part 10: Service Accounts & Permissions](#part-10-service-accounts--permissions)
- [Part 11: Private Pools & Networking](#part-11-private-pools--networking)
- [Part 12: Secrets & Environment Variables](#part-12-secrets--environment-variables)
- [Part 13: Notifications & Approvals](#part-13-notifications--approvals)
- [Part 14: Console Walkthrough — Build Triggers & Connections](#part-14-console-walkthrough--build-triggers--connections)
- [Part 15: Terraform & CLI](#part-15-terraform--cli)
- [Part 16: Real-World Patterns](#part-16-real-world-patterns)
- [Quick Reference](#quick-reference)
- [Console Walkthrough: Managing Builds & Triggers](#console-walkthrough-managing-builds--triggers)
- [Build Caching](#build-caching)
- [What's Next?](#whats-next)

---

## Overview

Cloud Build is Google Cloud's fully managed continuous integration and delivery (CI/CD) platform. It executes your builds as a series of steps, each running in a Docker container, giving you complete flexibility to build, test, and deploy anything — containers, JAR files, Go binaries, Terraform plans, or full application deployments. It's the backbone of CI/CD on GCP, analogous to GitHub Actions, AWS CodeBuild, or Azure Pipelines.

```
┌─────────────────────────────────────────────────────────────────────┐
│ WHAT YOU'LL LEARN                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ├── 1. What is Cloud Build and how it fits in GCP CI/CD            │
│ ├── 2. Architecture — build workers, steps, workspace              │
│ ├── 3. Build triggers (push, PR, tag, manual, scheduled)           │
│ ├── 4. cloudbuild.yaml — the build configuration file              │
│ ├── 5. Build steps and cloud builders (pre-built + custom)         │
│ ├── 6. Substitutions and dynamic variables                         │
│ ├── 7. Building and pushing Docker images                          │
│ ├── 8. Artifacts — storing build outputs                           │
│ ├── 9. Source connections (GitHub, Bitbucket, GitLab)              │
│ ├── 10. Service accounts and permissions                           │
│ ├── 11. Private pools and VPC networking                           │
│ ├── 12. Secrets and environment variables                          │
│ ├── 13. Notifications and approval gates                           │
│ ├── 14. Terraform and gcloud CLI                                   │
│ └── 15. Real-world architecture patterns                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Cloud Build Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT IS CLOUD BUILD?                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud Build = Serverless CI/CD platform                             │
│               + Docker-based build steps                            │
│               + native GCP integrations                             │
│                                                                       │
│ Key characteristics:                                                │
│ ├── Serverless — no build servers to manage                        │
│ ├── Each build step = a Docker container                           │
│ ├── Steps share a workspace volume (/workspace)                    │
│ ├── Runs on Google-managed infrastructure (or private pools)       │
│ ├── Free tier: 120 build-minutes/day (e1-medium)                   │
│ ├── Parallel builds — many builds can run simultaneously           │
│ ├── Language-agnostic — build anything that runs in Docker         │
│ ├── Native triggers from GitHub, Bitbucket, GitLab, CSR           │
│ ├── Integrates with Artifact Registry, Cloud Deploy, GKE, etc.    │
│ └── Build history, logs, and metrics in Console                    │
│                                                                       │
│ What Cloud Build does:                                              │
│ ├── Compile code (Java, Go, Python, Node.js, etc.)                │
│ ├── Run tests (unit, integration, e2e)                             │
│ ├── Build Docker images                                            │
│ ├── Push images to Artifact Registry / Container Registry          │
│ ├── Deploy to Cloud Run, GKE, App Engine, Cloud Functions         │
│ ├── Run Terraform apply                                            │
│ ├── Generate and publish documentation                             │
│ ├── Run security scans (SAST, container scanning)                  │
│ └── Any custom automation (shell scripts, tools)                   │
│                                                                       │
│ What Cloud Build does NOT do:                                      │
│ ├── Source control (use GitHub, GitLab, Bitbucket)                 │
│ ├── PR reviews / code review (use source platform)                 │
│ ├── Progressive delivery (use Cloud Deploy for canary/blue-green) │
│ ├── Artifact hosting (use Artifact Registry)                       │
│ └── Runtime monitoring (use Cloud Monitoring)                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

```
┌──────────────────────┬───────────────┬───────────────┬───────────────┐
│ Feature              │ GCP Cloud     │ AWS           │ Azure         │
│                      │ Build         │ CodeBuild     │ Pipelines     │
├──────────────────────┼───────────────┼───────────────┼───────────────┤
│ Type                 │ Serverless CI │ Serverless CI │ Managed CI/CD │
│ Build config         │ cloudbuild.   │ buildspec.yml │ azure-        │
│                      │ yaml          │               │ pipelines.yml │
│ Build environment    │ Docker        │ Docker        │ Docker/VM     │
│ Free tier            │ 120 min/day   │ 100 min/month │ 1800 min/month│
│ Parallel builds      │ ✅ Unlimited   │ ✅ Configurable│ ✅ By plan     │
│ Private networking   │ ✅ Private pool│ ✅ VPC support │ ✅ Self-hosted │
│ Built-in caching     │ ✅ kaniko cache│ ✅ S3 cache    │ ✅ Pipeline    │
│                      │ + GCS cache   │               │ cache         │
│ Approval gates       │ ✅ Yes         │ ✅ CodePipeline│ ✅ Yes         │
│ Matrix builds        │ ❌ Manual      │ ✅ Batch build │ ✅ Strategy    │
│ Self-hosted runners  │ ✅ Private pool│ ❌ No          │ ✅ Agents      │
│ Native container     │ ✅ Direct      │ ✅ ECR push    │ ✅ ACR push    │
│ registry push        │               │               │               │
│ Deploy integration   │ Cloud Deploy, │ CodeDeploy,   │ Azure Release │
│                      │ Cloud Run     │ ECS, Lambda   │ Pipelines     │
└──────────────────────┴───────────────┴───────────────┴───────────────┘
```

### Pricing Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD BUILD PRICING                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌────────────────────────┬─────────────────────────────────────┐   │
│ │ Machine Type           │ Price per build-minute               │   │
│ ├────────────────────────┼─────────────────────────────────────┤   │
│ │ e2-medium (default)    │ $0.003/min                          │   │
│ │ (1 vCPU, 4 GB RAM)     │ FREE: 120 min/day                  │   │
│ │                        │                                     │   │
│ │ e2-highcpu-8           │ $0.016/min                          │   │
│ │ (8 vCPU, 8 GB RAM)     │                                     │   │
│ │                        │                                     │   │
│ │ e2-highcpu-32          │ $0.064/min                          │   │
│ │ (32 vCPU, 32 GB RAM)   │                                     │   │
│ │                        │                                     │   │
│ │ Private pool (custom)  │ Varies by machine config            │   │
│ │                        │ + idle time charges                 │   │
│ └────────────────────────┴─────────────────────────────────────┘   │
│                                                                       │
│ Additional costs:                                                   │
│ ├── Network egress: Standard GCP egress pricing                    │
│ ├── Storage: Build logs in Cloud Logging (free tier applies)       │
│ ├── Artifact Registry: Image storage ($0.10/GB/month)             │
│ └── Private pools: Machine + idle time charges                     │
│                                                                       │
│ Free tier (per billing account per day):                           │
│ ├── 120 build-minutes on e2-medium (default machine)               │
│ ├── ≈ 2 hours of free builds daily                                │
│ └── Sufficient for small teams / personal projects                 │
│                                                                       │
│ Example monthly cost (medium team):                                │
│ ├── 500 builds/month × 5 min avg = 2,500 build-minutes           │
│ ├── Free: 120 min/day × 30 = 3,600 min free                     │
│ ├── Billable: 0 minutes (within free tier!)                       │
│ └── Most small-medium teams pay $0 for Cloud Build compute        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Architecture & How It Works

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD BUILD ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                    BUILD EXECUTION                             │   │
│ │                                                               │   │
│ │  Trigger fires (push, manual, etc.)                          │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  ┌──────────────────────────────────────────────────────┐    │   │
│ │  │ 1. SOURCE FETCH                                       │    │   │
│ │  │    ├── Clone repo (or download source archive)        │    │   │
│ │  │    ├── Placed in /workspace directory                 │    │   │
│ │  │    └── All steps share this /workspace volume        │    │   │
│ │  └──────────────────────────────────────────────────────┘    │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  ┌──────────────────────────────────────────────────────┐    │   │
│ │  │ 2. BUILD STEPS (sequential by default)                │    │   │
│ │  │                                                        │    │   │
│ │  │  Step 1: [gcr.io/cloud-builders/docker]               │    │   │
│ │  │  ┌──────────────────────────────────┐                 │    │   │
│ │  │  │ docker build -t my-app .          │                 │    │   │
│ │  │  │ /workspace ← source code here    │                 │    │   │
│ │  │  └──────────────────────────────────┘                 │    │   │
│ │  │       │ /workspace persists                           │    │   │
│ │  │       ▼                                                │    │   │
│ │  │  Step 2: [gcr.io/cloud-builders/docker]               │    │   │
│ │  │  ┌──────────────────────────────────┐                 │    │   │
│ │  │  │ docker push gcr.io/proj/my-app    │                 │    │   │
│ │  │  └──────────────────────────────────┘                 │    │   │
│ │  │       │                                                │    │   │
│ │  │       ▼                                                │    │   │
│ │  │  Step 3: [gcr.io/cloud-builders/gcloud]               │    │   │
│ │  │  ┌──────────────────────────────────┐                 │    │   │
│ │  │  │ gcloud run deploy my-app ...      │                 │    │   │
│ │  │  └──────────────────────────────────┘                 │    │   │
│ │  │                                                        │    │   │
│ │  └──────────────────────────────────────────────────────┘    │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  ┌──────────────────────────────────────────────────────┐    │   │
│ │  │ 3. ARTIFACTS (optional)                               │    │   │
│ │  │    ├── Upload images to Artifact Registry             │    │   │
│ │  │    ├── Upload files to Cloud Storage                  │    │   │
│ │  │    └── Generate provenance metadata (SLSA)            │    │   │
│ │  └──────────────────────────────────────────────────────┘    │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  Build status: SUCCESS / FAILURE / TIMEOUT                   │   │
│ │  Logs → Cloud Logging                                        │   │
│ │  Notifications → Pub/Sub / Slack / email                     │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Key concepts:                                                       │
│                                                                       │
│ /workspace:                                                         │
│ ├── Shared volume mounted in ALL build steps                       │
│ ├── Source code is cloned here at the start                        │
│ ├── Build outputs (binaries, artifacts) remain between steps       │
│ ├── Ephemeral — destroyed after build completes                    │
│ └── Default dir for each step (can override with 'dir' field)     │
│                                                                       │
│ Build step:                                                         │
│ ├── A Docker container that executes a command                     │
│ ├── Runs to completion, then next step starts                      │
│ ├── Exit code 0 = success, non-zero = failure (build stops)       │
│ ├── Can run in parallel using 'waitFor' field                     │
│ └── Has access to /workspace and mounted volumes                   │
│                                                                       │
│ Builder image:                                                      │
│ ├── The Docker image used for a build step                         │
│ ├── Google provides pre-built builders (docker, gcloud, npm, etc.)│
│ ├── Community builders (GitHub: GoogleCloudPlatform/cloud-builders)│
│ └── Custom builders (your own Docker image for specialized tools) │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Build Triggers

```
┌─────────────────────────────────────────────────────────────────────┐
│           BUILD TRIGGERS                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Triggers = rules that automatically start builds when events occur.│
│                                                                       │
│ Trigger types:                                                      │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │ 1. PUSH TO BRANCH                                            │   │
│ │    ├── Fires when code is pushed to a matching branch        │   │
│ │    ├── Branch pattern: regex (e.g., ^main$, ^release/.*)    │   │
│ │    └── Most common trigger type                              │   │
│ │                                                               │   │
│ │ 2. PUSH NEW TAG                                              │   │
│ │    ├── Fires when a new Git tag is pushed                    │   │
│ │    ├── Tag pattern: regex (e.g., ^v\d+\.\d+\.\d+$)         │   │
│ │    └── Common for release builds                             │   │
│ │                                                               │   │
│ │ 3. PULL REQUEST (GitHub / GitLab only)                       │   │
│ │    ├── Fires when a PR is opened or updated                  │   │
│ │    ├── Can post build status back to the PR                  │   │
│ │    ├── Branch pattern: target branch regex                   │   │
│ │    └── Use for CI checks before merge                        │   │
│ │                                                               │   │
│ │ 4. MANUAL INVOCATION                                         │   │
│ │    ├── Triggered manually from Console or CLI                │   │
│ │    ├── Can specify branch, tag, or commit SHA                │   │
│ │    └── Useful for one-off builds or debugging                │   │
│ │                                                               │   │
│ │ 5. SCHEDULED (via Cloud Scheduler)                           │   │
│ │    ├── Run builds on a cron schedule                          │   │
│ │    ├── Cloud Scheduler → HTTP call → Cloud Build API        │   │
│ │    └── Useful for nightly builds, periodic scans             │   │
│ │                                                               │   │
│ │ 6. PUB/SUB (event-driven)                                    │   │
│ │    ├── Trigger build from any Pub/Sub message                │   │
│ │    └── Advanced: chain builds, react to GCR events, etc.    │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Console Walkthrough: Create Trigger                                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Console → Cloud Build → Triggers → Create Trigger           │   │
│ │                                                               │   │
│ │ Name:            [deploy-main]                                │   │
│ │ Description:     [Deploy on push to main]                    │   │
│ │ Region:          [global] or [us-central1] ← regional builds │   │
│ │                                                               │   │
│ │ Event:                                                        │   │
│ │ ○ Push to a branch                                           │   │
│ │ ○ Push new tag                                                │   │
│ │ ○ Pull request (2nd gen only)                                │   │
│ │ ○ Manual invocation                                          │   │
│ │ ○ Pub/Sub message                                            │   │
│ │                                                               │   │
│ │ Source:                                                       │   │
│ │ ├── Repository: [my-org/backend-api (GitHub)] ← connected   │   │
│ │ └── Branch:     [^main$]  (regex pattern)                   │   │
│ │                                                               │   │
│ │ Configuration:                                                │   │
│ │ ○ Cloud Build configuration file (YAML or JSON)              │   │
│ │   └── Location: [/cloudbuild.yaml]                           │   │
│ │ ○ Dockerfile                                                  │   │
│ │   └── Location: [/Dockerfile]                                │   │
│ │   └── Image name: [us-docker.pkg.dev/proj/repo/app]          │   │
│ │ ○ Buildpacks                                                  │   │
│ │   └── Auto-detect language and build                         │   │
│ │                                                               │   │
│ │ Advanced:                                                     │   │
│ │ ├── Substitution variables: _DEPLOY_ENV = production         │   │
│ │ ├── Included files: src/**, Dockerfile                       │   │
│ │ ├── Ignored files: docs/**, *.md, .gitignore                 │   │
│ │ ├── Service account: (custom or default)                     │   │
│ │ ├── Approval required: ☐ (for manual gates)                 │   │
│ │ └── Build logs: Cloud Logging or Cloud Storage bucket        │   │
│ │                                                               │   │
│ │ [CREATE]                                                      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 1st Gen vs 2nd Gen Triggers:                                       │
│ ┌──────────────────────┬───────────────────┬────────────────────┐  │
│ │                      │ 1st Gen            │ 2nd Gen            │  │
│ ├──────────────────────┼───────────────────┼────────────────────┤  │
│ │ Source               │ CSR, GitHub (app) │ GitHub, GitLab,    │  │
│ │                      │                   │ Bitbucket (direct) │  │
│ │ PR triggers          │ ❌ Limited          │ ✅ Full support     │  │
│ │ PR comment trigger   │ ❌ No               │ ✅ /gcbrun          │  │
│ │ Regional builds      │ ❌ Global only      │ ✅ Regional         │  │
│ │ Repository connection│ CSR mirror or     │ Direct connection  │  │
│ │                      │ GitHub App        │ (no mirror needed) │  │
│ │ Recommended          │ Legacy            │ ✅ Use this          │  │
│ └──────────────────────┴───────────────────┴────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: cloudbuild.yaml — Build Configuration

```
┌─────────────────────────────────────────────────────────────────────┐
│           cloudbuild.yaml STRUCTURE                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ The cloudbuild.yaml (or cloudbuild.json) defines what Cloud Build  │
│ should do. It's the build recipe.                                  │
│                                                                       │
│ Basic structure:                                                    │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │ steps:          ← List of build steps (required)             │   │
│ │ images:         ← Docker images to push after build          │   │
│ │ artifacts:      ← Non-container artifacts to upload          │   │
│ │ substitutions:  ← Default variable values                    │   │
│ │ options:        ← Build options (machine type, logging, etc.)│   │
│ │ timeout:        ← Max build duration                         │   │
│ │ tags:           ← Labels for filtering builds                │   │
│ │ serviceAccount: ← Custom SA for the build                    │   │
│ │ availableSecrets: ← Secret Manager references               │   │
│ │ logsBucket:     ← GCS bucket for build logs                 │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Example: Complete cloudbuild.yaml

```yaml
# cloudbuild.yaml — Full example
# Builds a Docker image, runs tests, pushes to Artifact Registry,
# and deploys to Cloud Run.

steps:
  # Step 1: Build Docker image
  - id: "build"
    name: "gcr.io/cloud-builders/docker"
    args:
      - "build"
      - "-t"
      - "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_IMAGE}:${SHORT_SHA}"
      - "-t"
      - "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_IMAGE}:latest"
      - "."

  # Step 2: Run unit tests
  - id: "test"
    name: "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_IMAGE}:${SHORT_SHA}"
    entrypoint: "sh"
    args:
      - "-c"
      - "python -m pytest tests/ -v"
    waitFor: ["build"]

  # Step 3: Push image to Artifact Registry
  - id: "push"
    name: "gcr.io/cloud-builders/docker"
    args:
      - "push"
      - "--all-tags"
      - "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_IMAGE}"
    waitFor: ["test"]

  # Step 4: Deploy to Cloud Run
  - id: "deploy"
    name: "gcr.io/cloud-builders/gcloud"
    args:
      - "run"
      - "deploy"
      - "${_SERVICE_NAME}"
      - "--image=${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_IMAGE}:${SHORT_SHA}"
      - "--region=${_REGION}"
      - "--platform=managed"
      - "--allow-unauthenticated"
    waitFor: ["push"]

# Images to push (alternative to explicit docker push step)
images:
  - "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_IMAGE}:${SHORT_SHA}"
  - "${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_IMAGE}:latest"

# Default substitution values (overridable by trigger)
substitutions:
  _REGION: "us-central1"
  _REPO: "my-docker-repo"
  _IMAGE: "backend-api"
  _SERVICE_NAME: "backend-api"

# Build options
options:
  machineType: "E2_HIGHCPU_8"
  logging: CLOUD_LOGGING_ONLY
  dynamicSubstitutions: true

# Maximum build time (default: 10 minutes)
timeout: "1200s"

# Tags for filtering in Console
tags:
  - "backend"
  - "production"
```

### Minimal Examples

```yaml
# ─── Minimal: just build and push a Docker image ───
steps:
  - name: "gcr.io/cloud-builders/docker"
    args: ["build", "-t", "gcr.io/$PROJECT_ID/my-app:$COMMIT_SHA", "."]
images:
  - "gcr.io/$PROJECT_ID/my-app:$COMMIT_SHA"
```

```yaml
# ─── Minimal: run a shell script ───
steps:
  - name: "ubuntu"
    entrypoint: "bash"
    args:
      - "-c"
      - |
        echo "Hello from Cloud Build!"
        echo "Project: $PROJECT_ID"
        echo "Commit: $COMMIT_SHA"
        ls -la /workspace
```

```yaml
# ─── Minimal: npm build and test ───
steps:
  - name: "node:18"
    entrypoint: "npm"
    args: ["ci"]

  - name: "node:18"
    entrypoint: "npm"
    args: ["test"]

  - name: "node:18"
    entrypoint: "npm"
    args: ["run", "build"]

artifacts:
  objects:
    location: "gs://${PROJECT_ID}-build-artifacts/"
    paths: ["dist/**"]
```

---

## Part 5: Build Steps & Builders

```
┌─────────────────────────────────────────────────────────────────────┐
│           BUILD STEPS                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Each step is a Docker container that executes a command:           │
│                                                                       │
│ steps:                                                              │
│   - id: "step-name"           ← Unique identifier (optional)      │
│     name: "builder-image"     ← Docker image to run (required)    │
│     args: ["arg1", "arg2"]    ← Arguments to entrypoint           │
│     entrypoint: "/bin/bash"   ← Override image's ENTRYPOINT       │
│     env: ["KEY=value"]        ← Environment variables             │
│     dir: "subdir"             ← Working directory (relative to    │
│                                  /workspace)                       │
│     waitFor: ["step-id"]      ← Dependencies (for parallel exec) │
│     timeout: "300s"           ← Max time for this step            │
│     secretEnv: ["SECRET"]     ← Secret Manager secrets            │
│     volumes:                  ← Additional volumes                 │
│       - name: "cache"                                              │
│         path: "/cache"                                             │
│                                                                       │
│ Step execution model:                                               │
│ ├── Sequential by default (step 1 → step 2 → step 3)            │
│ ├── Parallel with 'waitFor' (explained below)                     │
│ ├── Step fails (non-zero exit) → build fails immediately         │
│ ├── /workspace is shared across all steps                          │
│ ├── Docker socket is available (can run docker-in-docker)          │
│ └── Each step pulls its builder image (cached locally)             │
│                                                                       │
│ Parallel execution with waitFor:                                   │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ steps:                                                        │   │
│ │   - id: "build"                                               │   │
│ │     name: "gcr.io/cloud-builders/docker"                      │   │
│ │     args: ["build", ...]                                      │   │
│ │                                                               │   │
│ │   - id: "lint"             ← Runs in PARALLEL with "test"    │   │
│ │     name: "node:18"                                           │   │
│ │     entrypoint: "npm"                                         │   │
│ │     args: ["run", "lint"]                                     │   │
│ │     waitFor: ["build"]     ← Waits for "build" only          │   │
│ │                                                               │   │
│ │   - id: "test"             ← Runs in PARALLEL with "lint"    │   │
│ │     name: "node:18"                                           │   │
│ │     entrypoint: "npm"                                         │   │
│ │     args: ["test"]                                            │   │
│ │     waitFor: ["build"]     ← Waits for "build" only          │   │
│ │                                                               │   │
│ │   - id: "deploy"           ← Waits for BOTH lint + test      │   │
│ │     name: "gcr.io/cloud-builders/gcloud"                      │   │
│ │     args: [...]                                               │   │
│ │     waitFor: ["lint", "test"]                                 │   │
│ │                                                               │   │
│ │ Execution flow:                                               │   │
│ │   build → ┬─ lint  ─┬→ deploy                               │   │
│ │           └─ test  ─┘                                        │   │
│ │                                                               │   │
│ │ Special: waitFor: ["-"]  ← Start immediately (don't wait     │   │
│ │                              for previous steps)              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Pre-Built Cloud Builders

```
┌─────────────────────────────────────────────────────────────────────┐
│           GOOGLE-PROVIDED BUILDERS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────────────────────────┬──────────────────────┐│
│ │ Builder Image                            │ Use                   ││
│ ├──────────────────────────────────────────┼──────────────────────┤│
│ │ gcr.io/cloud-builders/docker             │ Build/push Docker    ││
│ │ gcr.io/cloud-builders/gcloud             │ gcloud CLI commands  ││
│ │ gcr.io/cloud-builders/kubectl            │ Kubernetes deployments││
│ │ gcr.io/cloud-builders/gsutil             │ Cloud Storage ops    ││
│ │ gcr.io/cloud-builders/npm                │ Node.js / npm        ││
│ │ gcr.io/cloud-builders/yarn               │ Node.js / yarn       ││
│ │ gcr.io/cloud-builders/go                 │ Go builds            ││
│ │ gcr.io/cloud-builders/mvn                │ Maven (Java)         ││
│ │ gcr.io/cloud-builders/gradle             │ Gradle (Java/Kotlin) ││
│ │ gcr.io/cloud-builders/bazel              │ Bazel builds         ││
│ │ gcr.io/cloud-builders/git                │ Git operations       ││
│ │ gcr.io/cloud-builders/wget               │ Download files       ││
│ │ gcr.io/cloud-builders/curl               │ HTTP requests        ││
│ │ gcr.io/kaniko-project/executor           │ Kaniko (rootless Docker)││
│ └──────────────────────────────────────────┴──────────────────────┘│
│                                                                       │
│ Using ANY Docker image as a builder:                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Use official Docker Hub images                              │   │
│ │ - name: "python:3.11"                                         │   │
│ │   entrypoint: "pip"                                           │   │
│ │   args: ["install", "-r", "requirements.txt"]                │   │
│ │                                                               │   │
│ │ - name: "node:18-alpine"                                      │   │
│ │   entrypoint: "npm"                                           │   │
│ │   args: ["ci"]                                                │   │
│ │                                                               │   │
│ │ - name: "hashicorp/terraform:1.5"                             │   │
│ │   args: ["apply", "-auto-approve"]                           │   │
│ │                                                               │   │
│ │ - name: "alpine"                                              │   │
│ │   entrypoint: "sh"                                            │   │
│ │   args: ["-c", "echo Hello World"]                           │   │
│ │                                                               │   │
│ │ # Use your own custom builder from Artifact Registry         │   │
│ │ - name: "us-central1-docker.pkg.dev/proj/builders/my-tool"   │   │
│ │   args: ["run", "--config=prod"]                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Any public or private Docker image can be a builder.            │
│ ⚡ First-party builders (gcr.io/cloud-builders/*) are cached       │
│    locally for faster startup.                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Substitutions & Variables

```
┌─────────────────────────────────────────────────────────────────────┐
│           SUBSTITUTIONS & VARIABLES                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Built-in variables (automatically available):                      │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Variable           │ Description                              │   │
│ ├────────────────────┼──────────────────────────────────────────│   │
│ │ $PROJECT_ID        │ GCP project ID                           │   │
│ │ $PROJECT_NUMBER    │ GCP project number                       │   │
│ │ $BUILD_ID          │ Unique build ID (UUID)                   │   │
│ │ $COMMIT_SHA        │ Full Git commit SHA                      │   │
│ │ $SHORT_SHA         │ First 7 chars of commit SHA              │   │
│ │ $REVISION_ID       │ Same as COMMIT_SHA                       │   │
│ │ $BRANCH_NAME       │ Git branch name                          │   │
│ │ $TAG_NAME          │ Git tag name (if tag trigger)            │   │
│ │ $REPO_NAME         │ Repository name                          │   │
│ │ $REPO_FULL_NAME    │ Full repo name (owner/repo)              │   │
│ │ $TRIGGER_NAME      │ Name of the trigger                      │   │
│ │ $TRIGGER_BUILD_     │                                          │   │
│ │   CONFIG_PATH      │ Path to build config file                │   │
│ │ $SERVICE_ACCOUNT_   │                                          │   │
│ │   EMAIL            │ Build service account email              │   │
│ │ $LOCATION          │ Build region (or "global")               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ User-defined substitutions:                                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # In cloudbuild.yaml — define defaults                       │   │
│ │ substitutions:                                                │   │
│ │   _DEPLOY_ENV: "staging"       ← Must start with _           │   │
│ │   _REGION: "us-central1"      ← Uppercase convention        │   │
│ │   _IMAGE_NAME: "my-app"       ← Default value               │   │
│ │                                                               │   │
│ │ # Use in steps:                                               │   │
│ │ steps:                                                        │   │
│ │   - name: "gcr.io/cloud-builders/gcloud"                     │   │
│ │     args:                                                     │   │
│ │       - "run"                                                 │   │
│ │       - "deploy"                                              │   │
│ │       - "${_IMAGE_NAME}"                                      │   │
│ │       - "--region=${_REGION}"                                 │   │
│ │                                                               │   │
│ │ # Override in trigger settings or CLI:                        │   │
│ │ gcloud builds submit --substitutions=_DEPLOY_ENV=production  │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Dynamic substitutions (bash-style parameter expansion):            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Requires: options.dynamicSubstitutions = true               │   │
│ │                                                               │   │
│ │ options:                                                      │   │
│ │   dynamicSubstitutions: true                                  │   │
│ │                                                               │   │
│ │ substitutions:                                                │   │
│ │   _IMAGE_TAG: "${SHORT_SHA}"                                  │   │
│ │   _FULL_IMAGE: "${_REGION}-docker.pkg.dev/${PROJECT_ID}/\    │   │
│ │     ${_REPO}/${_IMAGE}:${_IMAGE_TAG}"                        │   │
│ │                                                               │   │
│ │ # Now $_FULL_IMAGE resolves to the full path automatically   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Rules for substitutions:                                           │
│ ├── User-defined MUST start with underscore: _MY_VAR              │
│ ├── Built-in vars do NOT have underscore: $PROJECT_ID             │
│ ├── Use ${VAR} syntax (curly braces) for clarity                  │
│ ├── Empty/undefined substitution = empty string (no error)        │
│ ├── Can be set in: cloudbuild.yaml, trigger config, CLI flag      │
│ └── Trigger-level substitutions override yaml defaults             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Building Docker Images

```
┌─────────────────────────────────────────────────────────────────────┐
│           BUILDING & PUSHING DOCKER IMAGES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Method 1: docker builder (most common)                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ steps:                                                        │   │
│ │   - name: "gcr.io/cloud-builders/docker"                      │   │
│ │     args:                                                     │   │
│ │       - "build"                                               │   │
│ │       - "-t"                                                  │   │
│ │       - "us-central1-docker.pkg.dev/$PROJECT_ID/\            │   │
│ │           my-repo/my-app:$SHORT_SHA"                         │   │
│ │       - "-t"                                                  │   │
│ │       - "us-central1-docker.pkg.dev/$PROJECT_ID/\            │   │
│ │           my-repo/my-app:latest"                             │   │
│ │       - "-f"                                                  │   │
│ │       - "Dockerfile"                                          │   │
│ │       - "."                                                   │   │
│ │                                                               │   │
│ │ # Auto-push these images after build succeeds                │   │
│ │ images:                                                       │   │
│ │   - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/\       │   │
│ │       my-app:$SHORT_SHA"                                     │   │
│ │   - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/\       │   │
│ │       my-app:latest"                                         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Method 2: Kaniko (no Docker daemon, better layer caching)          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ steps:                                                        │   │
│ │   - name: "gcr.io/kaniko-project/executor:latest"             │   │
│ │     args:                                                     │   │
│ │       - "--destination=us-central1-docker.pkg.dev/\          │   │
│ │           $PROJECT_ID/my-repo/my-app:$SHORT_SHA"             │   │
│ │       - "--destination=us-central1-docker.pkg.dev/\          │   │
│ │           $PROJECT_ID/my-repo/my-app:latest"                 │   │
│ │       - "--cache=true"                                        │   │
│ │       - "--cache-ttl=168h"                                    │   │
│ │       - "--dockerfile=Dockerfile"                             │   │
│ │       - "--context=."                                         │   │
│ │                                                               │   │
│ │ ✅ Better layer caching (stores in registry)                  │   │
│ │ ✅ No privileged mode needed                                  │   │
│ │ ✅ Builds AND pushes in one step                              │   │
│ │ ⚠️ Slightly different behavior from regular docker build     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Method 3: Buildpacks (no Dockerfile needed)                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ steps:                                                        │   │
│ │   - name: "gcr.io/k8s-skaffold/pack"                         │   │
│ │     args:                                                     │   │
│ │       - "build"                                               │   │
│ │       - "us-central1-docker.pkg.dev/$PROJECT_ID/\            │   │
│ │           my-repo/my-app:$SHORT_SHA"                         │   │
│ │       - "--builder=gcr.io/buildpacks/builder:v1"             │   │
│ │       - "--publish"                                           │   │
│ │                                                               │   │
│ │ ✅ Auto-detects language (Go, Node, Python, Java, etc.)      │   │
│ │ ✅ No Dockerfile maintenance                                  │   │
│ │ ✅ Security patches applied automatically to base image      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Quick build (CLI — no cloudbuild.yaml needed):                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Build from Dockerfile in current directory                  │   │
│ │ gcloud builds submit --tag \                                  │   │
│ │   us-central1-docker.pkg.dev/my-proj/my-repo/my-app:v1.0     │   │
│ │                                                               │   │
│ │ # This uploads source, builds, and pushes — all in one cmd  │   │
│ │ # No cloudbuild.yaml needed for simple Docker builds         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Artifacts & Storage

```
┌─────────────────────────────────────────────────────────────────────┐
│           BUILD ARTIFACTS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Two types of artifacts:                                             │
│                                                                       │
│ 1. CONTAINER IMAGES (via 'images' field)                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Pushed automatically after build succeeds                   │   │
│ │ images:                                                       │   │
│ │   - "us-central1-docker.pkg.dev/$PROJECT_ID/repo/app:$SHA"  │   │
│ │   - "us-central1-docker.pkg.dev/$PROJECT_ID/repo/app:latest"│   │
│ │                                                               │   │
│ │ • Images are pushed AFTER all steps succeed                  │   │
│ │ • Generates provenance metadata (for supply chain security)  │   │
│ │ • Recorded in build history (visible in Console)             │   │
│ │ • Alternative: push in a step (for more control)             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. NON-CONTAINER ARTIFACTS (via 'artifacts' field)                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Upload files to Cloud Storage                               │   │
│ │ artifacts:                                                    │   │
│ │   objects:                                                    │   │
│ │     location: "gs://my-build-artifacts/$BUILD_ID/"           │   │
│ │     paths:                                                    │   │
│ │       - "build/output/*.jar"                                 │   │
│ │       - "dist/**"                                            │   │
│ │       - "coverage/report.html"                               │   │
│ │                                                               │   │
│ │ # Upload to Maven in Artifact Registry                       │   │
│ │ artifacts:                                                    │   │
│ │   mavenArtifacts:                                            │   │
│ │     - repository: "https://us-central1-maven.pkg.dev/\      │   │
│ │         my-proj/my-maven-repo"                               │   │
│ │       path: "target/my-app-1.0.jar"                          │   │
│ │       artifactId: "my-app"                                   │   │
│ │       groupId: "com.example"                                 │   │
│ │       version: "1.0.0"                                       │   │
│ │                                                               │   │
│ │ # Upload to Python (PyPI) in Artifact Registry               │   │
│ │ artifacts:                                                    │   │
│ │   pythonPackages:                                            │   │
│ │     - repository: "https://us-central1-python.pkg.dev/\     │   │
│ │         my-proj/my-python-repo"                              │   │
│ │       paths: ["dist/*"]                                      │   │
│ │                                                               │   │
│ │ # Upload to npm in Artifact Registry                         │   │
│ │ artifacts:                                                    │   │
│ │   npmPackages:                                               │   │
│ │     - repository: "https://us-central1-npm.pkg.dev/\        │   │
│ │         my-proj/my-npm-repo"                                 │   │
│ │       packagePath: "."                                       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Build logs:                                                         │
│ ├── Default: Cloud Logging (searchable, integrated)                │
│ ├── Optional: GCS bucket (for long-term retention)                 │
│ ├── Configured via: options.logging or logsBucket field            │
│ │                                                                   │
│ │ options:                                                          │
│ │   logging: CLOUD_LOGGING_ONLY          # Only Cloud Logging     │
│ │   # OR                                                           │
│ │   logging: GCS_ONLY                    # Only GCS bucket        │
│ │   # OR                                                           │
│ │   logging: LEGACY                      # Both (default)         │
│ │                                                                   │
│ │ logsBucket: "gs://my-build-logs"       # Custom log bucket      │
│ └──                                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Source Connections — GitHub, Bitbucket, GitLab

```
┌─────────────────────────────────────────────────────────────────────┐
│           REPOSITORY CONNECTIONS (2ND GEN)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 2nd-gen triggers connect DIRECTLY to source platforms              │
│ (no CSR mirror needed).                                            │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │ ┌─────────────┐     ┌────────────────────┐                  │   │
│ │ │ GitHub      │────→│ Cloud Build         │                  │   │
│ │ │ (webhook)   │     │ 2nd Gen Trigger     │                  │   │
│ │ └─────────────┘     └────────────────────┘                  │   │
│ │                                                               │   │
│ │ ┌─────────────┐     ┌────────────────────┐                  │   │
│ │ │ GitLab      │────→│ Cloud Build         │                  │   │
│ │ │ (webhook)   │     │ 2nd Gen Trigger     │                  │   │
│ │ └─────────────┘     └────────────────────┘                  │   │
│ │                                                               │   │
│ │ ┌─────────────┐     ┌────────────────────┐                  │   │
│ │ │ Bitbucket   │────→│ Cloud Build         │                  │   │
│ │ │ (webhook)   │     │ 2nd Gen Trigger     │                  │   │
│ │ └─────────────┘     └────────────────────┘                  │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Setup process (GitHub — most common):                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Console → Cloud Build → Repositories → Create Host Connection│  │
│ │                                                               │   │
│ │ Step 1: Create a host connection                              │   │
│ │ ├── Provider: GitHub / GitLab / Bitbucket                    │   │
│ │ ├── Region: us-central1                                      │   │
│ │ ├── Name: "my-github-connection"                             │   │
│ │ └── Authorize: OAuth flow → install GitHub App              │   │
│ │                                                               │   │
│ │ Step 2: Link a repository                                    │   │
│ │ ├── Select the host connection                               │   │
│ │ ├── Choose which repos the connection can access             │   │
│ │ ├── Name: "my-org-backend-api"                               │   │
│ │ └── Each linked repo can be used in triggers                 │   │
│ │                                                               │   │
│ │ Step 3: Create a trigger using the linked repo               │   │
│ │ ├── Event: Push / Tag / PR                                   │   │
│ │ ├── Repository: my-org-backend-api                           │   │
│ │ ├── Branch/tag pattern                                       │   │
│ │ └── Build config (cloudbuild.yaml)                           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ GitHub-specific features:                                          │
│ ├── PR status checks (pass/fail posted to PR)                      │
│ ├── Comment trigger: post /gcbrun on PR to trigger build          │
│ ├── GitHub App integration (org-level, not personal OAuth)         │
│ └── Webhook-based (instant trigger, no polling)                   │
│                                                                       │
│ GitLab-specific features:                                          │
│ ├── Merge request triggers                                         │
│ ├── Pipeline status posted back to MR                              │
│ ├── GitLab.com and self-managed supported                         │
│ └── Personal access token or project access token                 │
│                                                                       │
│ Bitbucket-specific features:                                       │
│ ├── Push and PR triggers                                           │
│ ├── Bitbucket Cloud only (not Server/Data Center)                 │
│ └── App password authentication                                    │
│                                                                       │
│ CLI alternative (submit source directly):                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Submit local source code directly (no repo needed)         │   │
│ │ gcloud builds submit . --config=cloudbuild.yaml              │   │
│ │                                                               │   │
│ │ # Submit from a GCS archive                                   │   │
│ │ gcloud builds submit gs://my-bucket/source.tar.gz \           │   │
│ │   --config=cloudbuild.yaml                                    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Service Accounts & Permissions

```
┌─────────────────────────────────────────────────────────────────────┐
│           SERVICE ACCOUNTS FOR CLOUD BUILD                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Default Cloud Build service account:                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ {PROJECT_NUMBER}@cloudbuild.gserviceaccount.com               │   │
│ │                                                               │   │
│ │ Default roles (automatically granted):                       │   │
│ │ ├── cloudbuild.builds.builder                                │   │
│ │ ├── Logs to Cloud Logging                                    │   │
│ │ ├── Push/pull from Artifact Registry (same project)          │   │
│ │ └── Read source code                                         │   │
│ │                                                               │   │
│ │ Does NOT have by default:                                    │   │
│ │ ├── ❌ Deploy to Cloud Run                                   │   │
│ │ ├── ❌ Deploy to GKE                                         │   │
│ │ ├── ❌ Access Secret Manager                                  │   │
│ │ ├── ❌ Modify IAM policies                                   │   │
│ │ ├── ❌ Create GCP resources (Compute, SQL, etc.)             │   │
│ │ └── ❌ Access other projects                                  │   │
│ │                                                               │   │
│ │ ⚠️ Must grant additional roles for deployment steps!         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Common permissions to grant:                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Deploy to Cloud Run                                         │   │
│ │ gcloud projects add-iam-policy-binding my-proj \              │   │
│ │   --member="serviceAccount:123456@cloudbuild.gserviceaccount\ │   │
│ │     .com" \                                                   │   │
│ │   --role="roles/run.admin"                                    │   │
│ │                                                               │   │
│ │ # Act as Cloud Run service account                            │   │
│ │ gcloud iam service-accounts add-iam-policy-binding \          │   │
│ │   my-run-sa@my-proj.iam.gserviceaccount.com \                 │   │
│ │   --member="serviceAccount:123456@cloudbuild.gserviceaccount\ │   │
│ │     .com" \                                                   │   │
│ │   --role="roles/iam.serviceAccountUser"                       │   │
│ │                                                               │   │
│ │ # Deploy to GKE                                               │   │
│ │ --role="roles/container.developer"                            │   │
│ │                                                               │   │
│ │ # Access Secret Manager                                       │   │
│ │ --role="roles/secretmanager.secretAccessor"                   │   │
│ │                                                               │   │
│ │ # Run Terraform (create resources)                            │   │
│ │ --role="roles/editor"  (or more granular roles)              │   │
│ │                                                               │   │
│ │ # Push to Artifact Registry in another project                │   │
│ │ --role="roles/artifactregistry.writer"                       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Custom service account (recommended for production):               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Create a dedicated build SA with minimal permissions       │   │
│ │ gcloud iam service-accounts create cloud-build-deployer \     │   │
│ │   --display-name="Cloud Build Deployer"                       │   │
│ │                                                               │   │
│ │ # Grant only what's needed                                    │   │
│ │ gcloud projects add-iam-policy-binding my-proj \              │   │
│ │   --member="serviceAccount:cloud-build-deployer@my-proj.\   │   │
│ │     iam.gserviceaccount.com" \                                │   │
│ │   --role="roles/run.admin"                                    │   │
│ │                                                               │   │
│ │ # Use in cloudbuild.yaml:                                    │   │
│ │ serviceAccount: "projects/my-proj/serviceAccounts/\          │   │
│ │   cloud-build-deployer@my-proj.iam.gserviceaccount.com"      │   │
│ │                                                               │   │
│ │ # Or set in trigger configuration                            │   │
│ │                                                               │   │
│ │ ✅ Principle of least privilege                               │   │
│ │ ✅ Different SAs for different triggers (dev vs prod)         │   │
│ │ ✅ Audit trail per SA                                         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Private Pools & Networking

```
┌─────────────────────────────────────────────────────────────────────┐
│           PRIVATE POOLS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ By default, Cloud Build runs on Google-managed shared workers.     │
│ Private pools = dedicated workers in YOUR VPC for:                 │
│ ├── Accessing private resources (databases, APIs, registries)      │
│ ├── Compliance requirements (dedicated compute)                    │
│ ├── Custom machine types (more CPU/RAM)                            │
│ └── Network isolation (no shared infra)                            │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Default (shared) pool:          Private pool:               │   │
│ │                                                               │   │
│ │  ┌──────────────────┐          ┌──────────────────────────┐ │   │
│ │  │ Google-managed   │          │ Your VPC                  │ │   │
│ │  │ workers          │          │                           │ │   │
│ │  │                  │          │ ┌──────────────────────┐ │ │   │
│ │  │ ├── Shared infra │          │ │ Private pool workers │ │ │   │
│ │  │ ├── No VPC access│          │ │                      │ │ │   │
│ │  │ ├── Internet only│          │ │ ├── In your VPC     │ │ │   │
│ │  │ └── Standard mach│          │ │ ├── Access private  │ │ │   │
│ │  │                  │          │ │ │   resources        │ │ │   │
│ │  └──────────────────┘          │ │ ├── Custom machines │ │ │   │
│ │                                │ │ └── Static IP (opt.) │ │ │   │
│ │                                │ └──────────────────────┘ │ │   │
│ │                                │         │                 │ │   │
│ │                                │    ┌────┴────┐            │ │   │
│ │                                │    ▼         ▼            │ │   │
│ │                                │ Private   Private         │ │   │
│ │                                │ DB        API              │ │   │
│ │                                └──────────────────────────┘ │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Create private pool:                                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Create a worker pool with VPC peering                       │   │
│ │ gcloud builds worker-pools create my-private-pool \           │   │
│ │   --region=us-central1 \                                      │   │
│ │   --peered-network=projects/my-proj/global/networks/my-vpc \ │   │
│ │   --peered-network-ip-range=/24                               │   │
│ │                                                               │   │
│ │ # Use in cloudbuild.yaml:                                    │   │
│ │ options:                                                      │   │
│ │   pool:                                                       │   │
│ │     name: "projects/my-proj/locations/us-central1/\          │   │
│ │       workerPools/my-private-pool"                            │   │
│ │                                                               │   │
│ │ # Or set in trigger configuration                            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Use cases for private pools:                                       │
│ ├── Build needs to connect to Cloud SQL (private IP)               │
│ ├── Build pulls base images from private Artifact Registry         │
│ ├── Build runs integration tests against private services          │
│ ├── Compliance: builds must run on dedicated infrastructure        │
│ ├── Need larger machines (up to 32 vCPU)                          │
│ └── Need static egress IP (for allowlisting)                      │
│                                                                       │
│ ⚠️ Private pools have idle charges (even when not building).      │
│ ⚠️ Private pools are regional (build runs in that region).        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Secrets & Environment Variables

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECRETS IN CLOUD BUILD                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Method 1: Secret Manager integration (recommended)                 │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Define secrets available to the build                       │   │
│ │ availableSecrets:                                             │   │
│ │   secretManager:                                              │   │
│ │     - versionName: "projects/my-proj/secrets/\               │   │
│ │         DB_PASSWORD/versions/latest"                          │   │
│ │       env: "DB_PASSWORD"                                      │   │
│ │     - versionName: "projects/my-proj/secrets/\               │   │
│ │         API_KEY/versions/latest"                              │   │
│ │       env: "API_KEY"                                          │   │
│ │                                                               │   │
│ │ steps:                                                        │   │
│ │   - name: "ubuntu"                                            │   │
│ │     entrypoint: "bash"                                        │   │
│ │     args:                                                     │   │
│ │       - "-c"                                                  │   │
│ │       - |                                                     │   │
│ │         echo "Connecting to database..."                     │   │
│ │         ./migrate --password=$$DB_PASSWORD                   │   │
│ │     secretEnv: ["DB_PASSWORD"]  ← Must declare per step      │   │
│ │                                                               │   │
│ │   - name: "gcr.io/cloud-builders/gcloud"                     │   │
│ │     entrypoint: "bash"                                        │   │
│ │     args:                                                     │   │
│ │       - "-c"                                                  │   │
│ │       - "curl -H 'X-API-Key: $$API_KEY' https://api.ext.com"│   │
│ │     secretEnv: ["API_KEY"]                                   │   │
│ │                                                               │   │
│ │ ⚠️ Use $$ (double dollar) to reference secretEnv vars        │   │
│ │    in bash args (single $ is for substitutions)              │   │
│ │ ⚠️ Secrets are NEVER printed in build logs                   │   │
│ │ ⚠️ Build SA needs roles/secretmanager.secretAccessor         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Method 2: Inline environment variables (non-sensitive only!)       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ steps:                                                        │   │
│ │   - name: "node:18"                                           │   │
│ │     entrypoint: "npm"                                         │   │
│ │     args: ["test"]                                            │   │
│ │     env:                                                      │   │
│ │       - "NODE_ENV=test"                                       │   │
│ │       - "CI=true"                                             │   │
│ │       - "COVERAGE=true"                                       │   │
│ │                                                               │   │
│ │ ⚠️ env values are visible in build logs and config!          │   │
│ │    NEVER put secrets in env — use secretEnv instead.         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Method 3: Volume-based secret (mount as file)                      │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ steps:                                                        │   │
│ │   # Step 1: Fetch secret and write to shared volume          │   │
│ │   - name: "gcr.io/cloud-builders/gcloud"                     │   │
│ │     entrypoint: "bash"                                        │   │
│ │     args:                                                     │   │
│ │       - "-c"                                                  │   │
│ │       - |                                                     │   │
│ │         gcloud secrets versions access latest \              │   │
│ │           --secret=SERVICE_ACCOUNT_KEY > /secrets/sa-key.json│   │
│ │     volumes:                                                  │   │
│ │       - name: "secrets"                                       │   │
│ │         path: "/secrets"                                      │   │
│ │                                                               │   │
│ │   # Step 2: Use the secret file                              │   │
│ │   - name: "hashicorp/terraform:1.5"                           │   │
│ │     entrypoint: "sh"                                          │   │
│ │     args:                                                     │   │
│ │       - "-c"                                                  │   │
│ │       - |                                                     │   │
│ │         export GOOGLE_CREDENTIALS=/secrets/sa-key.json       │   │
│ │         terraform apply -auto-approve                        │   │
│ │     volumes:                                                  │   │
│ │       - name: "secrets"                                       │   │
│ │         path: "/secrets"                                      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Notifications & Approvals

```
┌─────────────────────────────────────────────────────────────────────┐
│           NOTIFICATIONS & APPROVAL GATES                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. PUB/SUB NOTIFICATIONS (automatic)                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Cloud Build automatically publishes to:                       │   │
│ │ Topic: cloud-builds                                           │   │
│ │                                                               │   │
│ │ Events published:                                             │   │
│ │ ├── Build QUEUED                                             │   │
│ │ ├── Build WORKING (started)                                  │   │
│ │ ├── Build SUCCESS                                            │   │
│ │ ├── Build FAILURE                                            │   │
│ │ ├── Build TIMEOUT                                            │   │
│ │ └── Build CANCELLED                                          │   │
│ │                                                               │   │
│ │ # Create subscription to react to build events               │   │
│ │ gcloud pubsub subscriptions create build-notifications \      │   │
│ │   --topic=cloud-builds \                                      │   │
│ │   --push-endpoint=https://my-webhook.example.com/build-event │   │
│ │                                                               │   │
│ │ # Or use Cloud Functions to send Slack notifications         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. SLACK / WEBHOOK NOTIFICATIONS                                   │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Option A: Use Cloud Build Notifiers (official)             │   │
│ │ # Deploy a pre-built notifier to Cloud Run:                  │   │
│ │ # - Slack notifier                                           │   │
│ │ # - SMTP (email) notifier                                    │   │
│ │ # - BigQuery notifier                                        │   │
│ │ # - HTTP notifier (custom webhook)                           │   │
│ │                                                               │   │
│ │ # Setup: deploy notifier image + create notifier config     │   │
│ │ # See: github.com/GoogleCloudPlatform/cloud-build-notifiers │   │
│ │                                                               │   │
│ │ # Option B: Cloud Function triggered by cloud-builds topic  │   │
│ │ # (Custom: parse Pub/Sub message, send to Slack/Teams/etc.) │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. APPROVAL GATES (manual approval before build proceeds)          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Enable approval on a trigger:                                 │   │
│ │                                                               │   │
│ │ Console → Cloud Build → Triggers → [trigger] → Edit         │   │
│ │   → ☑ Require approval before build executes                │   │
│ │                                                               │   │
│ │ Workflow:                                                     │   │
│ │ 1. Push triggers the build                                   │   │
│ │ 2. Build enters PENDING state (does not start)               │   │
│ │ 3. Approver gets notification                                 │   │
│ │ 4. Console → Cloud Build → Builds → Approve/Reject          │   │
│ │ 5. If approved: build executes                               │   │
│ │ 6. If rejected: build cancelled                              │   │
│ │                                                               │   │
│ │ Approver role: roles/cloudbuild.builds.approver               │   │
│ │                                                               │   │
│ │ Use cases:                                                    │   │
│ │ ├── Production deployments needing sign-off                  │   │
│ │ ├── Infrastructure changes (Terraform)                       │   │
│ │ ├── Compliance-required approvals                            │   │
│ │ └── Database migration reviews                               │   │
│ │                                                               │   │
│ │ # CLI: approve a pending build                               │   │
│ │ gcloud builds approve BUILD_ID --comment="LGTM, deploying"  │   │
│ │                                                               │   │
│ │ # CLI: reject a pending build                                │   │
│ │ gcloud builds reject BUILD_ID --comment="Needs fix"          │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 4. GITHUB STATUS CHECKS (2nd gen triggers)                         │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ When using GitHub 2nd-gen connection:                         │   │
│ │ ├── Build status is automatically posted to the PR/commit    │   │
│ │ ├── Shows ✅ pass or ❌ fail on the PR                        │   │
│ │ ├── Can be set as a required check for branch protection     │   │
│ │ └── Click "Details" on GitHub → links to Cloud Build logs   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 14: Console Walkthrough — Build Triggers & Connections

### Connecting a GitHub Repository (2nd Gen)

```
Console → Cloud Build → Repositories (2nd gen) → CREATE HOST CONNECTION

┌─────────────────────────────────────────────────────────────────┐
│           CREATE HOST CONNECTION                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Provider ──                                                   │
│ ● GitHub                                                        │
│ ○ GitHub Enterprise                                              │
│ ○ GitLab                                                         │
│ ○ Bitbucket Cloud                                                │
│ ○ Bitbucket Data Center                                          │
│                                                                   │
│ ── Region ──                                                     │
│ Region: [us-central1 ▼]                                         │
│ ⚡ Must match where your triggers will run.                      │
│                                                                   │
│ ── Name ──                                                       │
│ Name: [my-github-connection]                                    │
│                                                                   │
│                              [CONNECT]                          │
│                                                                   │
│ ⚡ This opens a GitHub OAuth flow:                                │
│   1. Authorize Google Cloud Build GitHub App                    │
│   2. Select repositories to grant access to                     │
│   3. Connection appears in Cloud Build                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

After connecting → LINK REPOSITORY

┌─────────────────────────────────────────────────────────────────┐
│           LINK REPOSITORY                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Connection: [my-github-connection ▼]                            │
│ Repository: [my-org/my-repo ▼]                                  │
│ ⚡ Shows repos the GitHub App has access to.                     │
│   If repo is missing → update GitHub App permissions.           │
│                                                                   │
│ Repository name: [my-repo]                                      │
│                                                                   │
│                              [LINK]                             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a Build Trigger

```
Console → Cloud Build → Triggers → CREATE TRIGGER

┌─────────────────────────────────────────────────────────────────┐
│           CREATE TRIGGER                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Name ──                                                       │
│ Name: [deploy-to-prod]                                          │
│ Description: [Build and deploy on push to main]                 │
│                                                                   │
│ ── Region ──                                                     │
│ Region: [us-central1 ▼]  (global also available)               │
│                                                                   │
│ ── Event ──                                                      │
│ ● Push to a branch                                              │
│ ○ Push new tag                                                   │
│ ○ Pull request                                                   │
│ ○ Manual invocation                                              │
│ ○ Pub/Sub message                                                │
│ ○ Webhook event                                                  │
│                                                                   │
│ ── Source ──                                                     │
│ Repository generation:                                          │
│ ● 2nd gen (recommended)                                        │
│ ○ 1st gen                                                       │
│                                                                   │
│ Repository: [my-github-connection/my-repo ▼]                   │
│ Branch: [^main$]  ← regex pattern                              │
│ ⚡ Use regex: ^main$ for exact match, ^release-.* for pattern  │
│                                                                   │
│ ── Configuration ──                                              │
│ Type:                                                           │
│ ● Cloud Build configuration file (yaml or json)                │
│ ○ Dockerfile                                                    │
│ ○ Buildpacks                                                    │
│                                                                   │
│ Location:                                                       │
│ ● Repository (cloudbuild.yaml in repo root)                    │
│ ○ Inline (enter YAML directly)                                 │
│                                                                   │
│ Cloud Build configuration file location:                        │
│ [cloudbuild.yaml]  ← path relative to repo root               │
│                                                                   │
│ ── Substitution variables (optional) ──                         │
│ Variable: [_ENVIRONMENT]     Value: [production]               │
│ [+ ADD VARIABLE]                                                │
│                                                                   │
│ ── Service account ──                                            │
│ ○ Default Cloud Build service account                          │
│ ● Custom service account:                                      │
│   [build-sa@project.iam.gserviceaccount.com ▼]                 │
│ ⚡ Use custom SA for least-privilege builds.                     │
│                                                                   │
│ ── Approval (optional) ──                                       │
│ ☐ Require approval before build executes                       │
│ ⚡ Useful for production deploy triggers.                        │
│                                                                   │
│                              [CREATE]                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Creating a Private Pool (Worker Pool)

```
Console → Cloud Build → Worker pools → CREATE

┌─────────────────────────────────────────────────────────────────┐
│           CREATE PRIVATE POOL                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Name ──                                                       │
│ Name: [prod-build-pool]                                         │
│ Region: [us-central1 ▼]                                         │
│                                                                   │
│ ── Machine type ──                                               │
│ Machine type: [e2-standard-4 ▼]                                 │
│   Options: e2-medium, e2-standard-4, e2-highcpu-8,             │
│            e2-highcpu-32, n1-highcpu-32                         │
│ ⚡ Larger machines = faster builds but higher cost.              │
│                                                                   │
│ ── Disk size ──                                                  │
│ Disk size (GB): [100] (min 100, default 100)                   │
│                                                                   │
│ ── Network (optional) ──                                        │
│ ☑ Assign to a VPC network for private access                   │
│ Network: [projects/my-project/global/networks/my-vpc ▼]        │
│ ⚡ Required for accessing private resources (Artifact Registry  │
│   in VPC, private Git repos, internal APIs).                    │
│                                                                   │
│                              [CREATE]                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 15: Terraform & CLI

### Terraform

```hcl
# ─────────────────────────────────────────────────────────────
# GitHub Connection (2nd Gen)
# ─────────────────────────────────────────────────────────────

resource "google_cloudbuildv2_connection" "github" {
  project  = "my-gcp-project"
  location = "us-central1"
  name     = "my-github-connection"

  github_config {
    app_installation_id = 12345678  # GitHub App installation ID
    authorizer_credential {
      oauth_token_secret_version = "projects/my-gcp-project/secrets/github-token/versions/latest"
    }
  }
}

resource "google_cloudbuildv2_repository" "backend_api" {
  project           = "my-gcp-project"
  location          = "us-central1"
  name              = "backend-api"
  parent_connection = google_cloudbuildv2_connection.github.name
  remote_uri        = "https://github.com/my-org/backend-api.git"
}

# ─────────────────────────────────────────────────────────────
# Build Trigger — Push to Branch (2nd Gen)
# ─────────────────────────────────────────────────────────────

resource "google_cloudbuild_trigger" "deploy_main" {
  project  = "my-gcp-project"
  location = "us-central1"
  name     = "deploy-on-push-main"

  repository_event_config {
    repository = google_cloudbuildv2_repository.backend_api.id
    push {
      branch = "^main$"
    }
  }

  filename = "cloudbuild.yaml"

  included_files = ["src/**", "Dockerfile"]
  ignored_files  = ["docs/**", "*.md"]

  substitutions = {
    _DEPLOY_ENV = "production"
    _REGION     = "us-central1"
  }

  service_account = google_service_account.cloud_build_deployer.id
}

# ─────────────────────────────────────────────────────────────
# Build Trigger — Pull Request
# ─────────────────────────────────────────────────────────────

resource "google_cloudbuild_trigger" "pr_check" {
  project  = "my-gcp-project"
  location = "us-central1"
  name     = "pr-ci-check"

  repository_event_config {
    repository = google_cloudbuildv2_repository.backend_api.id
    pull_request {
      branch = "^main$"
    }
  }

  filename = "cloudbuild-test.yaml"

  service_account = google_service_account.cloud_build_ci.id
}

# ─────────────────────────────────────────────────────────────
# Build Trigger — Tag (Release)
# ─────────────────────────────────────────────────────────────

resource "google_cloudbuild_trigger" "release_tag" {
  project  = "my-gcp-project"
  location = "us-central1"
  name     = "release-on-tag"

  repository_event_config {
    repository = google_cloudbuildv2_repository.backend_api.id
    push {
      tag = "^v\\d+\\.\\d+\\.\\d+$"
    }
  }

  filename = "cloudbuild-release.yaml"

  substitutions = {
    _DEPLOY_ENV = "production"
  }

  approval_config {
    approval_required = true
  }
}

# ─────────────────────────────────────────────────────────────
# Private Worker Pool
# ─────────────────────────────────────────────────────────────

resource "google_cloudbuild_worker_pool" "private_pool" {
  name     = "my-private-pool"
  location = "us-central1"
  project  = "my-gcp-project"

  worker_config {
    disk_size_gb   = 100
    machine_type   = "e2-highcpu-8"
    no_external_ip = false
  }

  network_config {
    peered_network          = "projects/my-gcp-project/global/networks/my-vpc"
    peered_network_ip_range = "/29"
  }
}

# ─────────────────────────────────────────────────────────────
# Service Account for Cloud Build
# ─────────────────────────────────────────────────────────────

resource "google_service_account" "cloud_build_deployer" {
  account_id   = "cloud-build-deployer"
  display_name = "Cloud Build Deployer"
  project      = "my-gcp-project"
}

resource "google_project_iam_member" "build_run_admin" {
  project = "my-gcp-project"
  role    = "roles/run.admin"
  member  = "serviceAccount:${google_service_account.cloud_build_deployer.email}"
}

resource "google_project_iam_member" "build_sa_user" {
  project = "my-gcp-project"
  role    = "roles/iam.serviceAccountUser"
  member  = "serviceAccount:${google_service_account.cloud_build_deployer.email}"
}

resource "google_project_iam_member" "build_logs_writer" {
  project = "my-gcp-project"
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.cloud_build_deployer.email}"
}

resource "google_project_iam_member" "build_ar_writer" {
  project = "my-gcp-project"
  role    = "roles/artifactregistry.writer"
  member  = "serviceAccount:${google_service_account.cloud_build_deployer.email}"
}

resource "google_service_account" "cloud_build_ci" {
  account_id   = "cloud-build-ci"
  display_name = "Cloud Build CI (read-only)"
  project      = "my-gcp-project"
}

resource "google_project_iam_member" "ci_logs_writer" {
  project = "my-gcp-project"
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.cloud_build_ci.email}"
}
```

### gcloud CLI Reference

```bash
# ─────────────────────────────────────────────────────────────
# Submit Builds (Manual / One-off)
# ─────────────────────────────────────────────────────────────

# Build from current directory using cloudbuild.yaml
gcloud builds submit . --config=cloudbuild.yaml

# Quick Docker build (no cloudbuild.yaml needed)
gcloud builds submit --tag us-central1-docker.pkg.dev/my-proj/repo/app:v1.0

# Build with substitutions
gcloud builds submit . \
  --config=cloudbuild.yaml \
  --substitutions=_DEPLOY_ENV=staging,_REGION=us-east1

# Build from GCS source
gcloud builds submit gs://my-bucket/source.tar.gz \
  --config=cloudbuild.yaml

# Build in a specific region
gcloud builds submit . \
  --config=cloudbuild.yaml \
  --region=us-central1

# Build using a private worker pool
gcloud builds submit . \
  --config=cloudbuild.yaml \
  --worker-pool=projects/my-proj/locations/us-central1/\
workerPools/my-private-pool

# Dry run (validate config without executing)
gcloud builds submit . \
  --config=cloudbuild.yaml \
  --no-source

# ─────────────────────────────────────────────────────────────
# Trigger Management
# ─────────────────────────────────────────────────────────────

# Create trigger (2nd gen, GitHub)
gcloud builds triggers create github \
  --name=deploy-on-push \
  --repository=projects/my-proj/locations/us-central1/\
connections/my-github/repositories/backend-api \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml \
  --region=us-central1

# Create trigger (tag-based)
gcloud builds triggers create github \
  --name=release-on-tag \
  --repository=projects/my-proj/locations/us-central1/\
connections/my-github/repositories/backend-api \
  --tag-pattern="^v.*" \
  --build-config=cloudbuild-release.yaml \
  --region=us-central1 \
  --require-approval

# List triggers
gcloud builds triggers list --region=us-central1

# Describe trigger
gcloud builds triggers describe deploy-on-push --region=us-central1

# Run trigger manually
gcloud builds triggers run deploy-on-push \
  --region=us-central1 \
  --branch=main

# Delete trigger
gcloud builds triggers delete deploy-on-push --region=us-central1

# ─────────────────────────────────────────────────────────────
# Build History & Logs
# ─────────────────────────────────────────────────────────────

# List recent builds
gcloud builds list --limit=10

# List builds for a specific trigger
gcloud builds list --filter="trigger_id=TRIGGER_ID" --limit=5

# Describe a specific build
gcloud builds describe BUILD_ID

# Stream build logs (follow in real-time)
gcloud builds log BUILD_ID --stream

# Cancel a running build
gcloud builds cancel BUILD_ID

# Approve a pending build
gcloud builds approve BUILD_ID --comment="Approved for production"

# Reject a pending build
gcloud builds reject BUILD_ID --comment="Needs changes"

# ─────────────────────────────────────────────────────────────
# Worker Pools
# ─────────────────────────────────────────────────────────────

# Create private worker pool
gcloud builds worker-pools create my-private-pool \
  --region=us-central1 \
  --peered-network=projects/my-proj/global/networks/my-vpc \
  --worker-machine-type=e2-highcpu-8 \
  --worker-disk-size=100

# List worker pools
gcloud builds worker-pools list --region=us-central1

# Describe worker pool
gcloud builds worker-pools describe my-private-pool \
  --region=us-central1

# Update worker pool
gcloud builds worker-pools update my-private-pool \
  --region=us-central1 \
  --worker-machine-type=e2-highcpu-32

# Delete worker pool
gcloud builds worker-pools delete my-private-pool \
  --region=us-central1

# ─────────────────────────────────────────────────────────────
# Connections (2nd Gen)
# ─────────────────────────────────────────────────────────────

# List connections
gcloud builds connections list --region=us-central1

# Describe connection
gcloud builds connections describe my-github-connection \
  --region=us-central1

# List repositories in a connection
gcloud builds repositories list \
  --connection=my-github-connection \
  --region=us-central1

# Delete connection
gcloud builds connections delete my-github-connection \
  --region=us-central1
```

---

## Part 15: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 1: FULL CI/CD — BUILD, TEST, DEPLOY TO CLOUD RUN    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Python API deployed to Cloud Run on every push to main.  │
│                                                                       │
│ Architecture:                                                       │
│ ┌──────────┐   push   ┌──────────┐  trigger  ┌──────────────────┐ │
│ │ GitHub   │─────────→│ Cloud    │──────────→│ Cloud Build       │ │
│ │ (main)   │          │ Build    │           │                   │ │
│ └──────────┘          │ 2nd Gen  │           │ 1. docker build   │ │
│                       │ Trigger  │           │ 2. pytest         │ │
│                       └──────────┘           │ 3. docker push    │ │
│                                              │ 4. gcloud run     │ │
│                                              │    deploy         │ │
│                                              └────────┬──────────┘ │
│                                                       │            │
│                                                       ▼            │
│ ┌──────────────────────┐    ┌──────────────────────────────────┐  │
│ │ Artifact Registry    │    │ Cloud Run                         │  │
│ │ (stores image)       │    │ (serves traffic)                  │  │
│ └──────────────────────┘    └──────────────────────────────────┘  │
│                                                                       │
│ cloudbuild.yaml:                                                   │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ steps:                                                        │   │
│ │   - id: "build"                                               │   │
│ │     name: "gcr.io/cloud-builders/docker"                      │   │
│ │     args: ["build", "-t",                                     │   │
│ │       "${_REGION}-docker.pkg.dev/${PROJECT_ID}/\             │   │
│ │         ${_REPO}/${_IMAGE}:${SHORT_SHA}", "."]               │   │
│ │                                                               │   │
│ │   - id: "test"                                                │   │
│ │     name: "${_REGION}-docker.pkg.dev/${PROJECT_ID}/\         │   │
│ │       ${_REPO}/${_IMAGE}:${SHORT_SHA}"                       │   │
│ │     entrypoint: "pytest"                                      │   │
│ │     args: ["tests/", "-v", "--tb=short"]                     │   │
│ │     waitFor: ["build"]                                        │   │
│ │                                                               │   │
│ │   - id: "push"                                                │   │
│ │     name: "gcr.io/cloud-builders/docker"                      │   │
│ │     args: ["push", "${_REGION}-docker.pkg.dev/\             │   │
│ │       ${PROJECT_ID}/${_REPO}/${_IMAGE}:${SHORT_SHA}"]        │   │
│ │     waitFor: ["test"]                                         │   │
│ │                                                               │   │
│ │   - id: "deploy"                                              │   │
│ │     name: "gcr.io/cloud-builders/gcloud"                      │   │
│ │     args: ["run", "deploy", "${_IMAGE}",                     │   │
│ │       "--image=${_REGION}-docker.pkg.dev/${PROJECT_ID}/\     │   │
│ │         ${_REPO}/${_IMAGE}:${SHORT_SHA}",                    │   │
│ │       "--region=${_REGION}",                                  │   │
│ │       "--platform=managed"]                                   │   │
│ │     waitFor: ["push"]                                         │   │
│ │                                                               │   │
│ │ substitutions:                                                │   │
│ │   _REGION: us-central1                                       │   │
│ │   _REPO: docker-repo                                         │   │
│ │   _IMAGE: backend-api                                        │   │
│ │ options:                                                      │   │
│ │   logging: CLOUD_LOGGING_ONLY                                │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 2: MULTI-ENVIRONMENT (DEV → STAGING → PROD)        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Different triggers for different environments.           │
│                                                                       │
│ Git workflow:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  feature/* ───→ develop ───→ main ───→ v1.2.3 (tag)         │   │
│ │                    │           │            │                 │   │
│ │                    ▼           ▼            ▼                 │   │
│ │              Trigger:     Trigger:     Trigger:              │   │
│ │              "dev-deploy" "staging"    "prod-release"        │   │
│ │                    │           │            │                 │   │
│ │                    ▼           ▼            ▼                 │   │
│ │              Cloud Run    Cloud Run    Cloud Run             │   │
│ │              (dev)        (staging)    (prod)                │   │
│ │                                        ⚠️ Approval required │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Trigger configurations:                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Trigger 1: "dev-deploy"                                       │   │
│ │ ├── Branch: ^develop$                                        │   │
│ │ ├── Substitutions: _DEPLOY_ENV=dev, _SERVICE=backend-api-dev│   │
│ │ ├── Approval: No                                             │   │
│ │ └── SA: cloud-build-dev@...                                  │   │
│ │                                                               │   │
│ │ Trigger 2: "staging-deploy"                                  │   │
│ │ ├── Branch: ^main$                                           │   │
│ │ ├── Substitutions: _DEPLOY_ENV=staging,                      │   │
│ │ │     _SERVICE=backend-api-staging                           │   │
│ │ ├── Approval: No                                             │   │
│ │ └── SA: cloud-build-staging@...                              │   │
│ │                                                               │   │
│ │ Trigger 3: "prod-release"                                    │   │
│ │ ├── Tag: ^v\d+\.\d+\.\d+$                                   │   │
│ │ ├── Substitutions: _DEPLOY_ENV=production,                   │   │
│ │ │     _SERVICE=backend-api                                   │   │
│ │ ├── Approval: ✅ REQUIRED                                    │   │
│ │ └── SA: cloud-build-prod@... (restricted permissions)        │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Single cloudbuild.yaml (environment determined by substitutions):  │
│   - Same build config, different trigger substitutions             │
│   - _DEPLOY_ENV drives which Cloud Run service to deploy to       │
│   - Different SAs for different privilege levels                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 3: INFRASTRUCTURE AS CODE (TERRAFORM)               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Terraform changes go through plan → approve → apply.    │
│                                                                       │
│ Architecture:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  PR opened ──→ Trigger: "tf-plan"                            │   │
│ │                 ├── terraform init                            │   │
│ │                 ├── terraform plan                            │   │
│ │                 └── Post plan output to PR comment            │   │
│ │                                                               │   │
│ │  PR merged to main ──→ Trigger: "tf-apply"                  │   │
│ │                         ├── ⚠️ Approval required             │   │
│ │                         ├── terraform init                   │   │
│ │                         └── terraform apply -auto-approve    │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ cloudbuild-plan.yaml:                                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ steps:                                                        │   │
│ │   - id: "tf-init"                                             │   │
│ │     name: "hashicorp/terraform:1.5"                           │   │
│ │     args: ["init", "-backend-config=bucket=${_TF_BUCKET}"]   │   │
│ │     dir: "infrastructure/"                                    │   │
│ │                                                               │   │
│ │   - id: "tf-validate"                                        │   │
│ │     name: "hashicorp/terraform:1.5"                           │   │
│ │     args: ["validate"]                                        │   │
│ │     dir: "infrastructure/"                                    │   │
│ │                                                               │   │
│ │   - id: "tf-plan"                                            │   │
│ │     name: "hashicorp/terraform:1.5"                           │   │
│ │     args: ["plan", "-out=tfplan",                            │   │
│ │       "-var=environment=${_DEPLOY_ENV}"]                      │   │
│ │     dir: "infrastructure/"                                    │   │
│ │                                                               │   │
│ │ substitutions:                                                │   │
│ │   _TF_BUCKET: "my-proj-terraform-state"                      │   │
│ │   _DEPLOY_ENV: "production"                                  │   │
│ │                                                               │   │
│ │ options:                                                      │   │
│ │   pool:                                                       │   │
│ │     name: "projects/my-proj/locations/us-central1/\          │   │
│ │       workerPools/my-private-pool"                            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ cloudbuild-apply.yaml:                                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ steps:                                                        │   │
│ │   - id: "tf-init"                                             │   │
│ │     name: "hashicorp/terraform:1.5"                           │   │
│ │     args: ["init", "-backend-config=bucket=${_TF_BUCKET}"]   │   │
│ │     dir: "infrastructure/"                                    │   │
│ │                                                               │   │
│ │   - id: "tf-apply"                                           │   │
│ │     name: "hashicorp/terraform:1.5"                           │   │
│ │     args: ["apply", "-auto-approve",                         │   │
│ │       "-var=environment=${_DEPLOY_ENV}"]                      │   │
│ │     dir: "infrastructure/"                                    │   │
│ │                                                               │   │
│ │ # Private pool needed to access state bucket and             │   │
│ │ # create resources in the VPC                                │   │
│ │ options:                                                      │   │
│ │   pool:                                                       │   │
│ │     name: "projects/my-proj/locations/us-central1/\          │   │
│ │       workerPools/my-private-pool"                            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Security best practices for IaC builds:                            │
│ ├── Dedicated SA with only needed IAM roles (not roles/editor)    │
│ ├── Private pool (access state bucket and VPC resources)          │
│ ├── Approval gate on apply trigger (human review required)        │
│ ├── Terraform state in GCS with versioning + locking              │
│ └── Plan output posted to PR for review before merge              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD BUILD QUICK REFERENCE                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Serverless CI/CD platform — build, test, deploy anything    │
│ Config: cloudbuild.yaml (or .json) defines build steps             │
│ Steps: Each step = Docker container executing a command             │
│ Workspace: /workspace — shared volume across all steps             │
│                                                                       │
│ Triggers:                                                           │
│ ├── Push to branch / tag                                           │
│ ├── Pull request (2nd gen)                                         │
│ ├── Manual invocation                                              │
│ ├── Scheduled (via Cloud Scheduler)                                │
│ └── Pub/Sub event                                                  │
│                                                                       │
│ Key builders:                                                       │
│ ├── gcr.io/cloud-builders/docker — Docker builds                  │
│ ├── gcr.io/cloud-builders/gcloud — gcloud CLI                     │
│ ├── gcr.io/cloud-builders/kubectl — Kubernetes                    │
│ ├── gcr.io/kaniko-project/executor — Rootless Docker build        │
│ └── Any Docker image (node:18, python:3.11, hashicorp/terraform)  │
│                                                                       │
│ Built-in variables: $PROJECT_ID, $COMMIT_SHA, $SHORT_SHA,          │
│   $BRANCH_NAME, $TAG_NAME, $BUILD_ID                               │
│ User variables: Must start with _ (e.g., $_DEPLOY_ENV)             │
│                                                                       │
│ Secrets: availableSecrets + secretEnv (Secret Manager)             │
│ Artifacts: images (Docker) or artifacts.objects (GCS)              │
│ Private pools: VPC-peered workers for private resource access      │
│ Approval: require-approval flag on triggers                        │
│                                                                       │
│ Free tier: 120 build-minutes/day (e2-medium)                       │
│                                                                       │
│ Key CLI:                                                            │
│ ├── gcloud builds submit . --config=cloudbuild.yaml               │
│ ├── gcloud builds submit --tag REGION-docker.pkg.dev/P/R/I:TAG    │
│ ├── gcloud builds triggers create github ...                      │
│ ├── gcloud builds triggers run TRIGGER_NAME --branch=main         │
│ ├── gcloud builds list                                             │
│ └── gcloud builds log BUILD_ID --stream                           │
│                                                                       │
│ 1st gen vs 2nd gen triggers:                                       │
│ ├── 2nd gen: Direct GitHub/GitLab/Bitbucket connections           │
│ ├── 2nd gen: PR triggers, regional builds, no CSR needed          │
│ └── Always use 2nd gen for new projects                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Managing Builds & Triggers

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING BUILDS & TRIGGERS IN CONSOLE                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Once you have triggers set up and builds running, you'll spend    │
│ most of your time viewing build history, checking logs, retrying  │
│ failed builds, and managing triggers. Here's how to do all of     │
│ that from the Google Cloud Console.                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Viewing Build History and Logs

```
Console → Cloud Build → History

┌─────────────────────────────────────────────────────────────────┐
│           BUILD HISTORY                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ Filter by: Region | Status | Trigger | Date range           │ │
│ ├─────────┬──────────┬───────────┬──────────┬────────────────┤ │
│ │ Build   │ Status   │ Trigger   │ Duration │ Commit         │ │
│ ├─────────┼──────────┼───────────┼──────────┼────────────────┤ │
│ │ abc123  │ ✅ Success│ deploy-   │ 2m 34s   │ feat: add     │ │
│ │         │          │ main      │          │ auth (a1b2c3)  │ │
│ │ def456  │ ❌ Failed │ deploy-   │ 1m 12s   │ fix: typo     │ │
│ │         │          │ main      │          │ (d4e5f6)       │ │
│ │ ghi789  │ ⏳ Running│ pr-check  │ 0m 45s   │ refactor:     │ │
│ │         │          │           │          │ cleanup        │ │
│ └─────────┴──────────┴───────────┴──────────┴────────────────┘ │
│                                                                   │
│ Click any build to see details:                                 │
│ ├── Build Summary — status, duration, trigger, commit info     │
│ ├── Build Log — real-time streaming output from each step      │
│ ├── Build Details — source, images, artifacts, substitutions   │
│ ├── Execution Details — step-by-step timing, exit codes        │
│ └── Artifacts — links to pushed images, uploaded files         │
│                                                                   │
│ Reading the build log:                                          │
│ ├── Each step's output is shown in order                       │
│ ├── Step name and ID shown in the left sidebar                 │
│ ├── Click a step → jumps to that step's logs                  │
│ ├── Errors are highlighted in red                               │
│ ├── Expand/collapse individual steps                           │
│ └── Download full log or view in Cloud Logging                 │
│                                                                   │
│ ⚡ Tip: Use the "Execution details" tab to see which step      │
│   failed and how long each step took — great for debugging.    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Retrying a Failed Build from Console

```
┌─────────────────────────────────────────────────────────────────────┐
│           RETRY A FAILED BUILD                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Sometimes a build fails due to a transient issue (network timeout,│
│ flaky test, rate limit, etc.). You can retry it without pushing   │
│ new code.                                                          │
│                                                                       │
│ From Console:                                                      │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ 1. Console → Cloud Build → History                           │   │
│ │ 2. Click the failed build (❌ status)                        │   │
│ │ 3. Click the "RETRY" button (top right of build details)     │   │
│ │ 4. Cloud Build re-runs the SAME build with:                  │   │
│ │    ├── Same source code / commit                             │   │
│ │    ├── Same substitutions                                    │   │
│ │    ├── Same cloudbuild.yaml                                  │   │
│ │    └── A new Build ID                                        │   │
│ │ 5. The retried build appears as a new entry in history       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ From CLI:                                                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # You can also trigger the same build manually:              │   │
│ │ gcloud builds triggers run TRIGGER_NAME \                    │   │
│ │   --branch=main                                              │   │
│ │                                                               │   │
│ │ # Or re-submit the same source:                              │   │
│ │ gcloud builds submit . --config=cloudbuild.yaml              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ The Retry button uses the exact same source snapshot.          │
│    If you need the latest code, trigger a new build instead.      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Deleting a Build Trigger from Console

```
┌─────────────────────────────────────────────────────────────────────┐
│           DELETE A BUILD TRIGGER                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ From Console:                                                      │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ 1. Console → Cloud Build → Triggers                          │   │
│ │ 2. Find the trigger you want to delete                       │   │
│ │ 3. Click the three-dot menu (⋮) on the right side            │   │
│ │ 4. Select "Delete"                                           │   │
│ │ 5. Confirm the deletion in the dialog                        │   │
│ │                                                               │   │
│ │ ⚠️ This does NOT delete:                                     │   │
│ │    ├── Past build history (still visible in History)          │   │
│ │    ├── The repository connection                             │   │
│ │    ├── The cloudbuild.yaml in your repo                      │   │
│ │    └── Any deployed resources                                │   │
│ │                                                               │   │
│ │ ⚠️ This DOES stop:                                           │   │
│ │    └── Future builds from this trigger (immediately)         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ From CLI:                                                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Delete a trigger by name                                   │   │
│ │ gcloud builds triggers delete TRIGGER_NAME \                 │   │
│ │   --region=us-central1                                       │   │
│ │                                                               │   │
│ │ # List triggers first if you're not sure of the name         │   │
│ │ gcloud builds triggers list --region=us-central1             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Cancelling an In-Progress Build

```
┌─────────────────────────────────────────────────────────────────────┐
│           CANCEL A RUNNING BUILD                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ If a build is taking too long, stuck, or you pushed the wrong     │
│ code, you can cancel it immediately.                               │
│                                                                       │
│ From Console:                                                      │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ 1. Console → Cloud Build → History                           │   │
│ │ 2. Find the running build (⏳ status)                        │   │
│ │ 3. Click the build to open details                           │   │
│ │ 4. Click the "CANCEL" button (top right)                     │   │
│ │ 5. Build status changes to CANCELLED                         │   │
│ │                                                               │   │
│ │ What happens when you cancel:                                │   │
│ │ ├── Currently running step is terminated                     │   │
│ │ ├── Remaining steps are skipped                              │   │
│ │ ├── No artifacts are uploaded (images/objects)               │   │
│ │ ├── Build is marked CANCELLED in history                     │   │
│ │ ├── You are NOT billed for time after cancellation           │   │
│ │ └── Pub/Sub notification sent (status: CANCELLED)            │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ From CLI:                                                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Cancel a running build by its Build ID                     │   │
│ │ gcloud builds cancel BUILD_ID                                │   │
│ │                                                               │   │
│ │ # Find the Build ID from the list of running builds          │   │
│ │ gcloud builds list --ongoing                                 │   │
│ │                                                               │   │
│ │ # Example:                                                   │   │
│ │ gcloud builds list --ongoing                                 │   │
│ │ # ID              STATUS   ...                               │   │
│ │ # abc123-def456   WORKING  ...                               │   │
│ │                                                               │   │
│ │ gcloud builds cancel abc123-def456                           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Tip: If the same commit triggered multiple builds (e.g.,       │
│   different triggers), you need to cancel each build separately.  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Build Caching

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHY CACHING MATTERS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Without caching, every build starts from scratch:                  │
│ ├── Download all dependencies (npm install, pip install, etc.)     │
│ ├── Rebuild every Docker layer (even unchanged ones)               │
│ ├── Re-compile everything from zero                                │
│                                                                       │
│ With caching, builds reuse previous work:                          │
│ ├── Skip unchanged Docker layers (seconds instead of minutes)      │
│ ├── Reuse downloaded dependencies                                  │
│ ├── Only rebuild what actually changed                             │
│                                                                       │
│ Impact:                                                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │  Without cache:  npm install ← 2 minutes                     │   │
│ │                  docker build ← 5 minutes                    │   │
│ │                  Total: ~7 minutes                            │   │
│ │                                                               │   │
│ │  With cache:     npm install ← 15 seconds (cached)           │   │
│ │                  docker build ← 45 seconds (layers cached)   │   │
│ │                  Total: ~1 minute                             │   │
│ │                                                               │   │
│ │  Faster builds = less cost (billed per build-minute)         │   │
│ │  Faster builds = faster feedback for developers              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Cloud Build offers two main caching strategies:                    │
│ ├── 1. Docker layer caching (with kaniko) — for Docker builds     │
│ └── 2. GCS-based caching — for non-Docker builds (npm, pip, etc.) │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Docker Layer Caching with Kaniko

```
┌─────────────────────────────────────────────────────────────────────┐
│           KANIKO DOCKER LAYER CACHING                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: The default docker builder does NOT cache layers between  │
│ builds. Every build pulls and rebuilds ALL layers from scratch.    │
│                                                                       │
│ Solution: Use kaniko — it stores Docker layers in a container      │
│ registry (Artifact Registry) and reuses them in future builds.     │
│                                                                       │
│ How it works:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Build 1 (no cache):                                         │   │
│ │  ┌──────────────────────────────────────────┐                │   │
│ │  │ Layer 1: FROM node:18         → BUILD     │                │   │
│ │  │ Layer 2: COPY package.json    → BUILD     │                │   │
│ │  │ Layer 3: RUN npm install      → BUILD     │ ← slow        │   │
│ │  │ Layer 4: COPY . .             → BUILD     │                │   │
│ │  │ Layer 5: RUN npm run build    → BUILD     │                │   │
│ │  └──────────────────────────────────────────┘                │   │
│ │  → All layers saved to registry cache                        │   │
│ │                                                               │   │
│ │  Build 2 (only source code changed):                         │   │
│ │  ┌──────────────────────────────────────────┐                │   │
│ │  │ Layer 1: FROM node:18         → CACHED ✅ │                │   │
│ │  │ Layer 2: COPY package.json    → CACHED ✅ │                │   │
│ │  │ Layer 3: RUN npm install      → CACHED ✅ │ ← fast!       │   │
│ │  │ Layer 4: COPY . .             → REBUILD   │ ← changed     │   │
│ │  │ Layer 5: RUN npm run build    → REBUILD   │                │   │
│ │  └──────────────────────────────────────────┘                │   │
│ │  → Layers 1-3 pulled from cache (seconds, not minutes)       │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Kaniko caches layers in the same registry as your image.        │
│ ⚡ No Docker daemon needed — runs unprivileged (more secure).      │
│ ⚡ Use --cache-ttl to control how long cache layers are kept.      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Example: cloudbuild.yaml with Kaniko Caching

```yaml
# cloudbuild.yaml — Docker build with kaniko layer caching
# Kaniko caches Docker layers in Artifact Registry for faster rebuilds.

steps:
  # Build and push with kaniko (caching enabled)
  - id: "build-and-push"
    name: "gcr.io/kaniko-project/executor:latest"
    args:
      - "--destination=${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_IMAGE}:${SHORT_SHA}"
      - "--destination=${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_IMAGE}:latest"
      - "--cache=true"                    # Enable layer caching
      - "--cache-ttl=168h"                # Cache layers for 7 days
      - "--dockerfile=Dockerfile"
      - "--context=."

  # Deploy to Cloud Run
  - id: "deploy"
    name: "gcr.io/cloud-builders/gcloud"
    args:
      - "run"
      - "deploy"
      - "${_IMAGE}"
      - "--image=${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO}/${_IMAGE}:${SHORT_SHA}"
      - "--region=${_REGION}"
      - "--platform=managed"
    waitFor: ["build-and-push"]

substitutions:
  _REGION: "us-central1"
  _REPO: "docker-repo"
  _IMAGE: "my-app"

options:
  logging: CLOUD_LOGGING_ONLY
```

### GCS-Based Caching for Non-Docker Builds

```
┌─────────────────────────────────────────────────────────────────────┐
│           GCS-BASED CACHING                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ For non-Docker builds (Node.js, Python, Go, Java, etc.), there's  │
│ no built-in cache. But you can cache dependencies in a GCS bucket │
│ and restore them at the start of each build.                       │
│                                                                       │
│ How it works:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Step 1: RESTORE — download cached files from GCS            │   │
│ │  ┌────────────────────────────────────────────────────────┐  │   │
│ │  │ gsutil cp gs://my-cache/node_modules.tar.gz /workspace │  │   │
│ │  │ tar -xzf node_modules.tar.gz                           │  │   │
│ │  └────────────────────────────────────────────────────────┘  │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  Step 2: BUILD — install + build (cache speeds up install)   │   │
│ │  ┌────────────────────────────────────────────────────────┐  │   │
│ │  │ npm ci           ← much faster with cached modules     │  │   │
│ │  │ npm run build                                          │  │   │
│ │  │ npm test                                               │  │   │
│ │  └────────────────────────────────────────────────────────┘  │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  Step 3: SAVE — upload updated cache back to GCS             │   │
│ │  ┌────────────────────────────────────────────────────────┐  │   │
│ │  │ tar -czf node_modules.tar.gz node_modules/             │  │   │
│ │  │ gsutil cp node_modules.tar.gz gs://my-cache/           │  │   │
│ │  └────────────────────────────────────────────────────────┘  │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ What to cache (common examples):                                   │
│ ├── Node.js: node_modules/                                         │
│ ├── Python: .venv/ or pip cache (~/.cache/pip)                     │
│ ├── Go: $GOPATH/pkg/mod/                                           │
│ ├── Java/Maven: ~/.m2/repository/                                  │
│ ├── Java/Gradle: ~/.gradle/caches/                                 │
│ └── General: any large directory that rarely changes               │
│                                                                       │
│ ⚡ Use lifecycle rules on the GCS bucket to auto-delete old caches│
│   (e.g., delete objects older than 14 days).                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Example: cloudbuild.yaml with GCS Caching (Node.js)

```yaml
# cloudbuild.yaml — Node.js build with GCS dependency caching
# Caches node_modules in a GCS bucket to speed up npm ci.

steps:
  # Step 1: Restore cache from GCS (if it exists)
  - id: "restore-cache"
    name: "gcr.io/cloud-builders/gsutil"
    entrypoint: "bash"
    args:
      - "-c"
      - |
        echo "Restoring cache..."
        gsutil cp gs://${PROJECT_ID}-build-cache/node_modules.tar.gz /workspace/node_modules.tar.gz || echo "No cache found, starting fresh."
        if [ -f /workspace/node_modules.tar.gz ]; then
          tar -xzf /workspace/node_modules.tar.gz
          echo "Cache restored!"
        fi

  # Step 2: Install dependencies (much faster with cache)
  - id: "install"
    name: "node:18"
    entrypoint: "npm"
    args: ["ci"]

  # Step 3: Run tests
  - id: "test"
    name: "node:18"
    entrypoint: "npm"
    args: ["test"]

  # Step 4: Build the app
  - id: "build"
    name: "node:18"
    entrypoint: "npm"
    args: ["run", "build"]

  # Step 5: Save cache back to GCS
  - id: "save-cache"
    name: "gcr.io/cloud-builders/gsutil"
    entrypoint: "bash"
    args:
      - "-c"
      - |
        echo "Saving cache..."
        tar -czf /workspace/node_modules.tar.gz node_modules/
        gsutil cp /workspace/node_modules.tar.gz gs://${PROJECT_ID}-build-cache/node_modules.tar.gz
        echo "Cache saved!"

  # Step 6: Deploy (optional)
  - id: "deploy"
    name: "gcr.io/cloud-builders/gcloud"
    args:
      - "run"
      - "deploy"
      - "my-frontend"
      - "--source=."
      - "--region=us-central1"

options:
  logging: CLOUD_LOGGING_ONLY

# ─── Pre-requisite: create the cache bucket once ───
# gsutil mb gs://YOUR_PROJECT_ID-build-cache
# gsutil lifecycle set lifecycle.json gs://YOUR_PROJECT_ID-build-cache
#
# lifecycle.json:
# { "rule": [{ "action": { "type": "Delete" },
#   "condition": { "age": 14 } }] }
```

```
┌─────────────────────────────────────────────────────────────────────┐
│           CACHING — QUICK SUMMARY                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌────────────────────┬───────────────────┬────────────────────────┐│
│ │ Approach           │ Best for          │ How                     ││
│ ├────────────────────┼───────────────────┼────────────────────────┤│
│ │ Kaniko             │ Docker image      │ --cache=true flag       ││
│ │ (layer caching)    │ builds            │ Layers stored in        ││
│ │                    │                   │ Artifact Registry       ││
│ │                    │                   │                         ││
│ │ GCS bucket         │ Non-Docker builds │ gsutil cp to/from       ││
│ │ (manual caching)   │ (npm, pip, maven) │ GCS bucket at start/   ││
│ │                    │                   │ end of build            ││
│ │                    │                   │                         ││
│ │ Docker --cache-from│ Docker builds     │ Pull previous image as  ││
│ │ (alternative)      │ (simpler setup)   │ cache source with       ││
│ │                    │                   │ --cache-from flag       ││
│ └────────────────────┴───────────────────┴────────────────────────┘│
│                                                                       │
│ Rules of thumb:                                                     │
│ ├── Building Docker images? → Use kaniko with --cache=true         │
│ ├── Non-Docker build (npm/pip/maven)? → Use GCS bucket caching    │
│ ├── Cache things that rarely change (dependencies, base layers)    │
│ ├── Don't cache things that change every build (source code)       │
│ └── Set a TTL / lifecycle policy so stale caches get cleaned up    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 32: Artifact Registry** → `32-artifact-registry.md`
