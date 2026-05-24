# Chapter 32: Artifact Registry

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Artifact Registry Fundamentals](#part-1-artifact-registry-fundamentals)
- [Part 2: Architecture & How It Works](#part-2-architecture--how-it-works)
- [Part 3: Docker Repositories](#part-3-docker-repositories)
- [Part 4: Pushing & Pulling Docker Images](#part-4-pushing--pulling-docker-images)
- [Part 5: npm Repositories](#part-5-npm-repositories)
- [Part 6: Maven Repositories](#part-6-maven-repositories)
- [Part 7: Python Repositories](#part-7-python-repositories)
- [Part 8: Other Formats — Go, Apt, Yum, KubeFlow](#part-8-other-formats--go-apt-yum-kubeflow)
- [Part 9: Remote & Virtual Repositories](#part-9-remote--virtual-repositories)
- [Part 10: Vulnerability Scanning](#part-10-vulnerability-scanning)
- [Part 11: Cleanup Policies & Lifecycle](#part-11-cleanup-policies--lifecycle)
- [Part 12: IAM & Security](#part-12-iam--security)
- [Part 13: Cross-Region & Multi-Project](#part-13-cross-region--multi-project)
- [Part 14: Terraform & CLI](#part-14-terraform--cli)
- [Part 15: Real-World Patterns](#part-15-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What is a Container Registry? (Beginner Explanation)](#what-is-a-container-registry-beginner-explanation)
- [Console Walkthrough: Managing & Deleting](#console-walkthrough-managing--deleting)
- [What's Next?](#whats-next)

---

## Overview

Artifact Registry is Google Cloud's universal package manager — a single place to store, manage, and secure all your build artifacts. Whether you're building Docker containers, npm packages, Maven JARs, Python wheels, or Go modules, Artifact Registry provides private, IAM-controlled repositories with integrated vulnerability scanning. It's the successor to Container Registry (GCR) and is the recommended artifact storage for all GCP workloads.

```
┌─────────────────────────────────────────────────────────────────────┐
│ WHAT YOU'LL LEARN                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ├── 1. What is Artifact Registry and why it replaced GCR           │
│ ├── 2. Architecture — repositories, formats, and regions           │
│ ├── 3. Docker repositories (create, configure, tag strategies)     │
│ ├── 4. Pushing and pulling Docker images                           │
│ ├── 5. npm repositories (Node.js packages)                         │
│ ├── 6. Maven repositories (Java/Kotlin JARs)                      │
│ ├── 7. Python repositories (PyPI packages)                         │
│ ├── 8. Other formats (Go, Apt, Yum, KubeFlow pipelines)           │
│ ├── 9. Remote and virtual repositories (proxies + aggregation)     │
│ ├── 10. Vulnerability scanning (automatic + on-demand)             │
│ ├── 11. Cleanup policies and lifecycle management                  │
│ ├── 12. IAM, encryption, and security                              │
│ ├── 13. Cross-region replication and multi-project access          │
│ ├── 14. Terraform and gcloud CLI                                   │
│ └── 15. Real-world architecture patterns                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Artifact Registry Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           WHAT IS ARTIFACT REGISTRY?                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Artifact Registry = Universal package management                   │
│                     + IAM-controlled access                         │
│                     + vulnerability scanning                        │
│                     + multi-format support                          │
│                                                                       │
│ Key characteristics:                                                │
│ ├── Multi-format: Docker, npm, Maven, Python, Go, Apt, Yum, etc. │
│ ├── Regional: repos live in a specific GCP region                  │
│ ├── IAM-integrated: fine-grained access control per repository     │
│ ├── Vulnerability scanning: automatic or on-demand (Container      │
│ │   Analysis API)                                                   │
│ ├── Cleanup policies: auto-delete old/untagged images              │
│ ├── Remote repos: cache/proxy for upstream registries              │
│ ├── Virtual repos: aggregate multiple repos behind one endpoint   │
│ ├── CMEK: customer-managed encryption keys                        │
│ ├── Immutable tags: prevent overwriting image tags                 │
│ ├── Native integration: Cloud Build, Cloud Run, GKE, Cloud Deploy│
│ └── Successor to Container Registry (gcr.io → pkg.dev)           │
│                                                                       │
│ Container Registry (gcr.io) vs Artifact Registry (pkg.dev):       │
│ ┌──────────────────────┬────────────────────┬────────────────────┐│
│ │ Feature              │ Container Registry │ Artifact Registry  ││
│ │                      │ (gcr.io) ⚠ Legacy  │ (pkg.dev) ✅ New   ││
│ ├──────────────────────┼────────────────────┼────────────────────┤│
│ │ Formats              │ Docker only        │ Docker, npm, Maven,││
│ │                      │                    │ Python, Go, Apt,   ││
│ │                      │                    │ Yum, KubeFlow      ││
│ │ Repository granularity│ Per-project (flat) │ Named repos per    ││
│ │                      │                    │ project (organized)││
│ │ IAM                  │ Bucket-level only  │ Per-repo IAM       ││
│ │ Location             │ Multi-region only  │ Region or multi-   ││
│ │                      │ (us, eu, asia)     │ region             ││
│ │ Vulnerability scan   │ ✅ Basic            │ ✅ Advanced         ││
│ │ Cleanup policies     │ ❌ Manual only      │ ✅ Automated        ││
│ │ Remote repos         │ ❌ No               │ ✅ Yes              ││
│ │ Virtual repos        │ ❌ No               │ ✅ Yes              ││
│ │ CMEK                 │ ❌ No               │ ✅ Yes              ││
│ │ Immutable tags       │ ❌ No               │ ✅ Yes              ││
│ │ Pricing              │ GCS storage pricing│ $0.10/GB/month     ││
│ │ Status               │ ⚠ Deprecated       │ ✅ Active           ││
│ └──────────────────────┴────────────────────┴────────────────────┘│
│                                                                       │
│ ⚡ ALWAYS use Artifact Registry for new projects.                  │
│ ⚡ Migrate from gcr.io to pkg.dev (gcr.io redirects supported).   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

```
┌──────────────────────┬───────────────┬───────────────┬───────────────┐
│ Feature              │ GCP Artifact  │ AWS ECR       │ Azure ACR     │
│                      │ Registry      │               │               │
├──────────────────────┼───────────────┼───────────────┼───────────────┤
│ Docker images        │ ✅ Yes         │ ✅ Yes         │ ✅ Yes         │
│ npm packages         │ ✅ Yes         │ ✅ CodeArtifact│ ✅ Azure       │
│                      │               │               │ Artifacts     │
│ Maven packages       │ ✅ Yes         │ ✅ CodeArtifact│ ✅ Azure       │
│                      │               │               │ Artifacts     │
│ Python packages      │ ✅ Yes         │ ✅ CodeArtifact│ ✅ Azure       │
│                      │               │               │ Artifacts     │
│ Go modules           │ ✅ Yes         │ ❌ No          │ ❌ No          │
│ OS packages (Apt/Yum)│ ✅ Yes         │ ❌ No          │ ❌ No          │
│ Unified service      │ ✅ Single      │ ❌ ECR +       │ ✅ ACR +       │
│                      │ service       │ CodeArtifact  │ Artifacts     │
│ Remote repos (proxy) │ ✅ Yes         │ ✅ CodeArtifact│ ❌ No          │
│ Virtual repos        │ ✅ Yes         │ ✅ CodeArtifact│ ❌ No          │
│ Vulnerability scan   │ ✅ Built-in    │ ✅ Inspector   │ ✅ Defender    │
│ Cleanup policies     │ ✅ Yes         │ ✅ Lifecycle   │ ✅ Retention   │
│ Geo-replication      │ ✅ Multi-region│ ✅ Cross-region│ ✅ Geo-repl    │
│ Image signing        │ ✅ Binary Auth │ ✅ Cosign/Notary│ ✅ Notation   │
│ Pricing (storage)    │ $0.10/GB/month│ $0.10/GB/month│ $0.003/GB/day │
└──────────────────────┴───────────────┴───────────────┴───────────────┘
```

### Pricing Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│           ARTIFACT REGISTRY PRICING                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌─────────────────────┬──────────────────────────────────────────┐ │
│ │ Component           │ Price                                     │ │
│ ├─────────────────────┼──────────────────────────────────────────┤ │
│ │ Storage             │ $0.10/GB/month                            │ │
│ │                     │ First 0.5 GB FREE                        │ │
│ │                     │                                          │ │
│ │ Network egress      │ Standard GCP egress pricing              │ │
│ │ (to internet)       │ Same-region = free                       │ │
│ │                     │ Inter-region = egress charges             │ │
│ │                     │                                          │ │
│ │ Vulnerability       │ Container scanning:                      │ │
│ │ scanning            │ $0.26/image scanned (first scan)         │ │
│ │                     │ Continuous monitoring: included           │ │
│ └─────────────────────┴──────────────────────────────────────────┘ │
│                                                                       │
│ Cost example (medium project):                                      │
│ ├── 50 Docker images × ~200 MB avg = 10 GB storage               │
│ ├── 100 npm packages × ~5 MB avg = 0.5 GB storage               │
│ ├── Storage cost: 10.5 GB × $0.10 = $1.05/month                 │
│ ├── Scanning: 50 images × $0.26 = $13/month                     │
│ └── Total: ~$14/month (very affordable)                           │
│                                                                       │
│ ⚡ Egress within same region is FREE (Cloud Build, GKE, Cloud Run │
│    pulling images from same-region Artifact Registry = no charge). │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Architecture & How It Works

```
┌─────────────────────────────────────────────────────────────────────┐
│           ARTIFACT REGISTRY ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                       GCP PROJECT                                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                                                               │  │
│  │  Artifact Registry                                           │  │
│  │                                                               │  │
│  │  ┌── REGION: us-central1 ──────────────────────────────┐    │  │
│  │  │                                                       │    │  │
│  │  │  ┌─────────────────────────────────────────────────┐ │    │  │
│  │  │  │ DOCKER REPO: "docker-prod"                       │ │    │  │
│  │  │  │ Format: Docker                                   │ │    │  │
│  │  │  │ Mode: Standard                                    │ │    │  │
│  │  │  │                                                   │ │    │  │
│  │  │  │ Images:                                           │ │    │  │
│  │  │  │ ├── backend-api:v1.2.3, :latest, :abc123f        │ │    │  │
│  │  │  │ ├── frontend-app:v2.0.0, :latest                 │ │    │  │
│  │  │  │ └── worker:v1.0.0                                │ │    │  │
│  │  │  └─────────────────────────────────────────────────┘ │    │  │
│  │  │                                                       │    │  │
│  │  │  ┌─────────────────────────────────────────────────┐ │    │  │
│  │  │  │ NPM REPO: "npm-internal"                         │ │    │  │
│  │  │  │ Format: npm                                       │ │    │  │
│  │  │  │                                                   │ │    │  │
│  │  │  │ Packages:                                         │ │    │  │
│  │  │  │ ├── @myorg/shared-utils@1.0.0                    │ │    │  │
│  │  │  │ └── @myorg/ui-components@2.3.1                   │ │    │  │
│  │  │  └─────────────────────────────────────────────────┘ │    │  │
│  │  │                                                       │    │  │
│  │  │  ┌─────────────────────────────────────────────────┐ │    │  │
│  │  │  │ PYTHON REPO: "python-internal"                   │ │    │  │
│  │  │  │ Format: Python                                    │ │    │  │
│  │  │  │                                                   │ │    │  │
│  │  │  │ Packages:                                         │ │    │  │
│  │  │  │ └── my-ml-utils==0.5.0                           │ │    │  │
│  │  │  └─────────────────────────────────────────────────┘ │    │  │
│  │  │                                                       │    │  │
│  │  │  ┌─────────────────────────────────────────────────┐ │    │  │
│  │  │  │ REMOTE REPO: "dockerhub-cache"                   │ │    │  │
│  │  │  │ Format: Docker                                    │ │    │  │
│  │  │  │ Mode: Remote (proxies Docker Hub)                 │ │    │  │
│  │  │  └─────────────────────────────────────────────────┘ │    │  │
│  │  │                                                       │    │  │
│  │  └───────────────────────────────────────────────────────┘    │  │
│  │                                                               │  │
│  │  ┌── REGION: europe-west1 ─────────────────────────────┐    │  │
│  │  │  ┌─────────────────────────────────────────────────┐ │    │  │
│  │  │  │ DOCKER REPO: "docker-eu"                         │ │    │  │
│  │  │  │ (For EU-based workloads)                          │ │    │  │
│  │  │  └─────────────────────────────────────────────────┘ │    │  │
│  │  └───────────────────────────────────────────────────────┘    │  │
│  │                                                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Repository URL format:                                             │
│ {REGION}-docker.pkg.dev/{PROJECT}/{REPO}/{IMAGE}:{TAG}             │
│ e.g., us-central1-docker.pkg.dev/my-proj/docker-prod/api:v1.2.3   │
│                                                                       │
│ Repository modes:                                                   │
│ ┌────────────────┬───────────────────────────────────────────────┐ │
│ │ Mode           │ Description                                    │ │
│ ├────────────────┼───────────────────────────────────────────────┤ │
│ │ Standard       │ You push/pull your own artifacts. Default.    │ │
│ │ Remote         │ Proxy/cache for an upstream registry          │ │
│ │                │ (Docker Hub, npm registry, Maven Central,     │ │
│ │                │ PyPI, etc.)                                    │ │
│ │ Virtual        │ Aggregates multiple repos (standard + remote) │ │
│ │                │ behind a single endpoint                      │ │
│ └────────────────┴───────────────────────────────────────────────┘ │
│                                                                       │
│ Supported formats:                                                  │
│ ├── Docker (OCI container images)                                  │
│ ├── npm (Node.js packages)                                         │
│ ├── Maven (Java/Kotlin/Scala JARs)                                │
│ ├── Python (PyPI packages / wheels)                                │
│ ├── Go (Go modules)                                                │
│ ├── Apt (Debian packages)                                          │
│ ├── Yum (RPM packages)                                             │
│ └── KubeFlow (ML pipelines)                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Docker Repositories

### Console Walkthrough

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONSOLE → Artifact Registry → Create Repository                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Name:              [docker-prod]                              │   │
│ │   ├── Unique within project+region                           │   │
│ │   └── Lowercase, hyphens, numbers                            │   │
│ │                                                               │   │
│ │ Format:            [Docker ▼]                                 │   │
│ │   Options: Docker, Maven, npm, Python, Apt, Yum, Go,        │   │
│ │            KubeFlow Pipelines                                │   │
│ │                                                               │   │
│ │ Mode:              [Standard ▼]                               │   │
│ │   ○ Standard — store your own artifacts                      │   │
│ │   ○ Remote — proxy upstream registry                        │   │
│ │   ○ Virtual — aggregate multiple repos                      │   │
│ │                                                               │   │
│ │ Location type:                                                │   │
│ │   ○ Region:        [us-central1]                             │   │
│ │     └── Single region (lowest latency, cheapest egress)     │   │
│ │   ○ Multi-region:  [us] / [europe] / [asia]                 │   │
│ │     └── Replicated across regions (higher availability)     │   │
│ │                                                               │   │
│ │ Encryption:                                                   │   │
│ │   ○ Google-managed encryption key ✅ (default)               │   │
│ │   ○ Customer-managed encryption key (CMEK)                   │   │
│ │     → Select Cloud KMS key in same location                 │   │
│ │                                                               │   │
│ │ Immutable image tags: ☐                                      │   │
│ │   → When enabled, tags cannot be overwritten                │   │
│ │   → Prevents "latest" tag from being updated                │   │
│ │   → Forces unique tags (SHA, version, timestamp)            │   │
│ │                                                               │   │
│ │ Cleanup policies: (configure after creation)                 │   │
│ │                                                               │   │
│ │ Labels: [environment: production]                             │   │
│ │                                                               │   │
│ │ [CREATE]                                                      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ After creation, the repo endpoint is:                              │
│ us-central1-docker.pkg.dev/my-proj/docker-prod                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Docker Tag Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│           TAG STRATEGIES                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────┬──────────────────────────────────────────┐│
│ │ Strategy             │ Example                                   ││
│ ├──────────────────────┼──────────────────────────────────────────┤│
│ │ Git SHA (recommended)│ api:abc123f                               ││
│ │                      │ Immutable, traceable to exact commit     ││
│ │                      │                                          ││
│ │ Semantic version     │ api:v1.2.3                               ││
│ │                      │ Human-readable, follows release cycle    ││
│ │                      │                                          ││
│ │ latest (caution)     │ api:latest                               ││
│ │                      │ Mutable — always points to newest build ││
│ │                      │ ⚠️ Avoid in production (non-deterministic)││
│ │                      │                                          ││
│ │ Branch-based         │ api:main-abc123f                         ││
│ │                      │ Useful for development environments     ││
│ │                      │                                          ││
│ │ Multi-tag            │ api:v1.2.3 AND api:abc123f AND api:latest││
│ │ (recommended)        │ Tag same image digest with multiple tags ││
│ └──────────────────────┴──────────────────────────────────────────┘│
│                                                                       │
│ ⚡ Best practice: always tag with git SHA + semantic version.      │
│ ⚡ Use immutable tags in production repos to prevent overwrites.   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Pushing & Pulling Docker Images

```
┌─────────────────────────────────────────────────────────────────────┐
│           DOCKER PUSH / PULL                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Configure Docker authentication                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # One-time setup: configure Docker to use gcloud credentials │   │
│ │ gcloud auth configure-docker us-central1-docker.pkg.dev      │   │
│ │                                                               │   │
│ │ # For multiple regions:                                       │   │
│ │ gcloud auth configure-docker \                                │   │
│ │   us-central1-docker.pkg.dev,europe-west1-docker.pkg.dev     │   │
│ │                                                               │   │
│ │ # This updates ~/.docker/config.json to use gcloud as        │   │
│ │ # the credential helper for *.pkg.dev domains                │   │
│ │                                                               │   │
│ │ # Verify:                                                     │   │
│ │ cat ~/.docker/config.json                                     │   │
│ │ # Should show: "credHelpers": {                              │   │
│ │ #   "us-central1-docker.pkg.dev": "gcloud"                  │   │
│ │ # }                                                           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 2: Build, tag, and push                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Set variables                                               │   │
│ │ REGION=us-central1                                            │   │
│ │ PROJECT=my-gcp-project                                        │   │
│ │ REPO=docker-prod                                              │   │
│ │ IMAGE=backend-api                                             │   │
│ │ TAG=v1.2.3                                                    │   │
│ │                                                               │   │
│ │ # Full image path                                             │   │
│ │ FULL_IMAGE=${REGION}-docker.pkg.dev/${PROJECT}/${REPO}/${IMAGE}│  │
│ │                                                               │   │
│ │ # Build                                                       │   │
│ │ docker build -t ${FULL_IMAGE}:${TAG} .                        │   │
│ │                                                               │   │
│ │ # Also tag as latest                                          │   │
│ │ docker tag ${FULL_IMAGE}:${TAG} ${FULL_IMAGE}:latest          │   │
│ │                                                               │   │
│ │ # Push                                                        │   │
│ │ docker push ${FULL_IMAGE}:${TAG}                              │   │
│ │ docker push ${FULL_IMAGE}:latest                              │   │
│ │                                                               │   │
│ │ # Or push all tags at once                                    │   │
│ │ docker push --all-tags ${FULL_IMAGE}                          │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Step 3: Pull                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ docker pull ${FULL_IMAGE}:${TAG}                              │   │
│ │                                                               │   │
│ │ # Pull by digest (immutable, most secure)                    │   │
│ │ docker pull ${FULL_IMAGE}@sha256:abcdef1234567890...          │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Cloud Build shortcut (no Docker setup needed):                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Build and push in one command                               │   │
│ │ gcloud builds submit --tag \                                  │   │
│ │   us-central1-docker.pkg.dev/my-proj/docker-prod/api:v1.2.3  │   │
│ │                                                               │   │
│ │ # Cloud Build handles auth automatically                     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ GKE / Cloud Run pulling images:                                    │
│ ├── Same-project: automatic (default SA has reader access)         │
│ ├── Cross-project: grant roles/artifactregistry.reader to the     │
│ │   GKE/Cloud Run service account                                  │
│ └── Same-region pull: no egress charges                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: npm Repositories

```
┌─────────────────────────────────────────────────────────────────────┐
│           NPM REPOSITORIES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Host private npm packages for your organization.                   │
│                                                                       │
│ Create:                                                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ gcloud artifacts repositories create npm-internal \           │   │
│ │   --repository-format=npm \                                   │   │
│ │   --location=us-central1 \                                    │   │
│ │   --description="Internal npm packages"                       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Configure npm to use the repo:                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Option A: Project-level .npmrc                              │   │
│ │ # Create .npmrc in your project root:                        │   │
│ │                                                               │   │
│ │ @myorg:registry=https://us-central1-npm.pkg.dev/my-proj/\   │   │
│ │   npm-internal/                                               │   │
│ │ //us-central1-npm.pkg.dev/my-proj/npm-internal/:always-auth= │   │
│ │   true                                                        │   │
│ │                                                               │   │
│ │ # Option B: Use gcloud to set up auth                        │   │
│ │ npx google-artifactregistry-auth                              │   │
│ │ # Or:                                                         │   │
│ │ gcloud artifacts print-settings npm \                         │   │
│ │   --repository=npm-internal \                                 │   │
│ │   --location=us-central1 \                                    │   │
│ │   --scope=@myorg                                              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Publish a package:                                                  │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # package.json                                                │   │
│ │ {                                                             │   │
│ │   "name": "@myorg/shared-utils",                              │   │
│ │   "version": "1.0.0",                                        │   │
│ │   "publishConfig": {                                          │   │
│ │     "registry": "https://us-central1-npm.pkg.dev/\           │   │
│ │       my-proj/npm-internal/"                                  │   │
│ │   }                                                           │   │
│ │ }                                                             │   │
│ │                                                               │   │
│ │ npm publish                                                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Install a package:                                                  │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ npm install @myorg/shared-utils                               │   │
│ │ # npm reads .npmrc and routes @myorg scope to Artifact Reg.  │   │
│ │ # Public packages (no @myorg scope) still go to npmjs.com    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Maven Repositories

```
┌─────────────────────────────────────────────────────────────────────┐
│           MAVEN REPOSITORIES                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Host private Java/Kotlin/Scala artifacts (JARs, WARs, POMs).      │
│                                                                       │
│ Create:                                                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ gcloud artifacts repositories create maven-internal \         │   │
│ │   --repository-format=maven \                                 │   │
│ │   --location=us-central1 \                                    │   │
│ │   --description="Internal Maven artifacts"                    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Configure Maven (pom.xml):                                         │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ <!-- Get settings with gcloud: -->                            │   │
│ │ <!-- gcloud artifacts print-settings mvn \                   │   │
│ │      --repository=maven-internal \                            │   │
│ │      --location=us-central1 -->                               │   │
│ │                                                               │   │
│ │ <!-- In pom.xml (for publishing): -->                        │   │
│ │ <distributionManagement>                                      │   │
│ │   <repository>                                                │   │
│ │     <id>artifact-registry</id>                                │   │
│ │     <url>artifactregistry://us-central1-maven.pkg.dev/\     │   │
│ │       my-proj/maven-internal</url>                            │   │
│ │   </repository>                                               │   │
│ │ </distributionManagement>                                     │   │
│ │                                                               │   │
│ │ <!-- In pom.xml (for pulling dependencies): -->              │   │
│ │ <repositories>                                                │   │
│ │   <repository>                                                │   │
│ │     <id>artifact-registry</id>                                │   │
│ │     <url>artifactregistry://us-central1-maven.pkg.dev/\     │   │
│ │       my-proj/maven-internal</url>                            │   │
│ │     <releases><enabled>true</enabled></releases>              │   │
│ │     <snapshots><enabled>true</enabled></snapshots>            │   │
│ │   </repository>                                               │   │
│ │ </repositories>                                               │   │
│ │                                                               │   │
│ │ <!-- Maven extension for auth (in pom.xml or .mvn/): -->    │   │
│ │ <extensions>                                                  │   │
│ │   <extension>                                                 │   │
│ │     <groupId>com.google.cloud.artifactregistry</groupId>     │   │
│ │     <artifactId>artifactregistry-maven-wagon</artifactId>    │   │
│ │     <version>2.2.1</version>                                  │   │
│ │   </extension>                                                │   │
│ │ </extensions>                                                 │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Publish:                                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ mvn deploy                                                    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Gradle alternative:                                                │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ // build.gradle                                               │   │
│ │ plugins {                                                     │   │
│ │   id "com.google.cloud.artifactregistry.gradle-plugin"       │   │
│ │     version "2.2.1"                                           │   │
│ │ }                                                             │   │
│ │                                                               │   │
│ │ publishing {                                                  │   │
│ │   repositories {                                              │   │
│ │     maven {                                                   │   │
│ │       url "artifactregistry://us-central1-maven.pkg.dev/\   │   │
│ │         my-proj/maven-internal"                              │   │
│ │     }                                                         │   │
│ │   }                                                           │   │
│ │ }                                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Python Repositories

```
┌─────────────────────────────────────────────────────────────────────┐
│           PYTHON REPOSITORIES                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Host private Python packages (wheels, sdists).                     │
│                                                                       │
│ Create:                                                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ gcloud artifacts repositories create python-internal \        │   │
│ │   --repository-format=python \                                │   │
│ │   --location=us-central1 \                                    │   │
│ │   --description="Internal Python packages"                    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Configure pip/twine:                                               │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Get configuration:                                          │   │
│ │ gcloud artifacts print-settings python \                      │   │
│ │   --repository=python-internal \                              │   │
│ │   --location=us-central1                                      │   │
│ │                                                               │   │
│ │ # pip.conf (for installing):                                 │   │
│ │ [global]                                                      │   │
│ │ extra-index-url = https://us-central1-python.pkg.dev/\       │   │
│ │   my-proj/python-internal/simple/                             │   │
│ │                                                               │   │
│ │ # .pypirc (for publishing):                                  │   │
│ │ [distutils]                                                   │   │
│ │ index-servers = artifact-registry                             │   │
│ │                                                               │   │
│ │ [artifact-registry]                                           │   │
│ │ repository = https://us-central1-python.pkg.dev/\            │   │
│ │   my-proj/python-internal/                                    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Authentication:                                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Install the keyring backend                                 │   │
│ │ pip install keyring                                            │   │
│ │ pip install keyrings.google-artifactregistry-auth              │   │
│ │                                                               │   │
│ │ # Now pip install and twine upload authenticate automatically │   │
│ │ # using your gcloud credentials                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Publish:                                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Build the package                                           │   │
│ │ python -m build                                               │   │
│ │                                                               │   │
│ │ # Upload to Artifact Registry                                 │   │
│ │ twine upload --repository artifact-registry dist/*            │   │
│ │                                                               │   │
│ │ # Or via gcloud (Cloud Build artifact upload):               │   │
│ │ # artifacts.pythonPackages in cloudbuild.yaml                 │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Install:                                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # With pip.conf configured:                                   │   │
│ │ pip install my-ml-utils                                       │   │
│ │                                                               │   │
│ │ # Or explicitly:                                              │   │
│ │ pip install --extra-index-url \                               │   │
│ │   https://us-central1-python.pkg.dev/my-proj/\              │   │
│ │   python-internal/simple/ my-ml-utils                        │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Other Formats — Go, Apt, Yum, KubeFlow

```
┌─────────────────────────────────────────────────────────────────────┐
│           OTHER ARTIFACT FORMATS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. GO MODULES                                                      │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ gcloud artifacts repositories create go-internal \            │   │
│ │   --repository-format=go \                                    │   │
│ │   --location=us-central1                                      │   │
│ │                                                               │   │
│ │ # Upload a Go module                                          │   │
│ │ gcloud artifacts go upload \                                  │   │
│ │   --repository=go-internal \                                  │   │
│ │   --location=us-central1 \                                    │   │
│ │   --module-path=example.com/myorg/mymodule \                 │   │
│ │   --version=v1.0.0 \                                          │   │
│ │   --source=.                                                  │   │
│ │                                                               │   │
│ │ # Configure Go to use Artifact Registry:                     │   │
│ │ export GONOSUMCHECK="example.com/myorg/*"                    │   │
│ │ export GOFLAGS="-mod=mod"                                     │   │
│ │ export GOPROXY="https://us-central1-go.pkg.dev/my-proj/\    │   │
│ │   go-internal,https://proxy.golang.org,direct"               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. APT (DEBIAN PACKAGES)                                          │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ gcloud artifacts repositories create apt-internal \           │   │
│ │   --repository-format=apt \                                   │   │
│ │   --location=us-central1                                      │   │
│ │                                                               │   │
│ │ # Upload .deb package                                         │   │
│ │ gcloud artifacts apt upload apt-internal \                    │   │
│ │   --location=us-central1 \                                    │   │
│ │   --source=my-tool_1.0.0_amd64.deb                           │   │
│ │                                                               │   │
│ │ # Add to apt sources on VMs:                                 │   │
│ │ # /etc/apt/sources.list.d/artifact-registry.list             │   │
│ │ deb ar+https://us-central1-apt.pkg.dev/projects/my-proj \   │   │
│ │   apt-internal main                                           │   │
│ │                                                               │   │
│ │ # Install                                                     │   │
│ │ apt-get update && apt-get install my-tool                    │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. YUM (RPM PACKAGES)                                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ gcloud artifacts repositories create yum-internal \           │   │
│ │   --repository-format=yum \                                   │   │
│ │   --location=us-central1                                      │   │
│ │                                                               │   │
│ │ # Upload .rpm package                                         │   │
│ │ gcloud artifacts yum upload yum-internal \                   │   │
│ │   --location=us-central1 \                                    │   │
│ │   --source=my-tool-1.0.0-1.x86_64.rpm                        │   │
│ │                                                               │   │
│ │ # Add to yum repos on VMs:                                  │   │
│ │ # /etc/yum.repos.d/artifact-registry.repo                   │   │
│ │ [artifact-registry]                                           │   │
│ │ name=Artifact Registry                                        │   │
│ │ baseurl=https://us-central1-yum.pkg.dev/projects/my-proj/\  │   │
│ │   yum-internal                                                │   │
│ │ enabled=1                                                     │   │
│ │ gpgcheck=0                                                    │   │
│ │                                                               │   │
│ │ yum install my-tool                                           │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 4. KUBEFLOW PIPELINES                                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Store and version KubeFlow pipeline templates (YAML).        │   │
│ │                                                               │   │
│ │ gcloud artifacts repositories create kfp-pipelines \          │   │
│ │   --repository-format=kfp \                                   │   │
│ │   --location=us-central1                                      │   │
│ │                                                               │   │
│ │ # Upload pipeline                                             │   │
│ │ gcloud artifacts kfp upload \                                │   │
│ │   --repository=kfp-pipelines \                               │   │
│ │   --location=us-central1 \                                    │   │
│ │   --package-name=my-training-pipeline \                      │   │
│ │   --version=1.0.0 \                                           │   │
│ │   --source=pipeline.yaml                                      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Remote & Virtual Repositories

```
┌─────────────────────────────────────────────────────────────────────┐
│           REMOTE REPOSITORIES (PROXY / CACHE)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ A remote repo acts as a caching proxy for an upstream registry.    │
│ Instead of pulling directly from Docker Hub / npm / Maven Central, │
│ clients pull from Artifact Registry, which caches the result.      │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Application                                                  │   │
│ │       │                                                       │   │
│ │       │ docker pull us-central1-docker.pkg.dev/              │   │
│ │       │   my-proj/dockerhub-cache/nginx:latest               │   │
│ │       ▼                                                       │   │
│ │  ┌──────────────────────┐                                    │   │
│ │  │ Remote Repo          │    cache    ┌──────────────────┐  │   │
│ │  │ "dockerhub-cache"    │────miss────→│ Docker Hub        │  │   │
│ │  │                      │←───cache────│ (upstream)        │  │   │
│ │  │ Cached images:       │    fill     └──────────────────┘  │   │
│ │  │ ├── nginx:latest     │                                    │   │
│ │  │ ├── python:3.11      │                                    │   │
│ │  │ └── node:18          │                                    │   │
│ │  └──────────────────────┘                                    │   │
│ │                                                               │   │
│ │  On cache hit: returns artifact from Artifact Registry       │   │
│ │  On cache miss: fetches from upstream, caches, returns       │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Why use remote repos:                                              │
│ ├── Reduced internet egress (cached locally in GCP)                │
│ ├── Faster pulls (GCP internal network vs internet)                │
│ ├── Protection from upstream outages (Docker Hub rate limits)      │
│ ├── Vulnerability scanning on cached images                        │
│ ├── Centralized security policy (VPC SC, IAM)                      │
│ └── Audit trail of all packages used                               │
│                                                                       │
│ Create remote Docker repo (Docker Hub):                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ gcloud artifacts repositories create dockerhub-cache \        │   │
│ │   --repository-format=docker \                                │   │
│ │   --location=us-central1 \                                    │   │
│ │   --mode=remote-repository \                                  │   │
│ │   --remote-repo-config-desc="Docker Hub cache" \              │   │
│ │   --remote-docker-repo=DOCKER-HUB                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Create remote npm repo (npmjs.com):                                │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ gcloud artifacts repositories create npmjs-cache \            │   │
│ │   --repository-format=npm \                                   │   │
│ │   --location=us-central1 \                                    │   │
│ │   --mode=remote-repository \                                  │   │
│ │   --remote-npm-repo=NPMJS                                     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Supported upstream registries:                                     │
│ ├── Docker: Docker Hub, custom registries                          │
│ ├── npm: npmjs.com, custom registries                              │
│ ├── Maven: Maven Central, custom registries                       │
│ ├── Python: PyPI, custom registries                                │
│ └── Apt/Yum: upstream OS package mirrors                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           VIRTUAL REPOSITORIES (AGGREGATION)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ A virtual repo combines multiple repos behind ONE endpoint.        │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Application                                                  │   │
│ │       │                                                       │   │
│ │       │ docker pull us-central1-docker.pkg.dev/              │   │
│ │       │   my-proj/docker-all/backend-api:v1.2.3              │   │
│ │       ▼                                                       │   │
│ │  ┌──────────────────────┐                                    │   │
│ │  │ Virtual Repo          │                                    │   │
│ │  │ "docker-all"          │                                    │   │
│ │  │                       │                                    │   │
│ │  │ Searches in order:    │                                    │   │
│ │  │ ├── 1. docker-prod   ─────→ (standard repo, priority 100)│   │
│ │  │ ├── 2. docker-staging─────→ (standard repo, priority 90) │   │
│ │  │ └── 3. dockerhub-cache───→ (remote repo, priority 80)   │   │
│ │  └──────────────────────┘                                    │   │
│ │                                                               │   │
│ │  Resolution: tries repos in priority order, returns first    │   │
│ │  match. If not in docker-prod, checks docker-staging, then   │   │
│ │  falls through to Docker Hub cache.                          │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Create virtual Docker repo:                                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ gcloud artifacts repositories create docker-all \             │   │
│ │   --repository-format=docker \                                │   │
│ │   --location=us-central1 \                                    │   │
│ │   --mode=virtual-repository \                                 │   │
│ │   --upstream-policy-file=policy.json                          │   │
│ │                                                               │   │
│ │ # policy.json:                                                │   │
│ │ [                                                             │   │
│ │   {                                                           │   │
│ │     "id": "docker-prod",                                      │   │
│ │     "repository": "projects/my-proj/locations/us-central1/\  │   │
│ │       repositories/docker-prod",                              │   │
│ │     "priority": 100                                           │   │
│ │   },                                                          │   │
│ │   {                                                           │   │
│ │     "id": "dockerhub-cache",                                  │   │
│ │     "repository": "projects/my-proj/locations/us-central1/\  │   │
│ │       repositories/dockerhub-cache",                          │   │
│ │     "priority": 80                                            │   │
│ │   }                                                           │   │
│ │ ]                                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Use case: Single endpoint for developers                           │
│ ├── Private images found first (docker-prod)                       │
│ ├── Public images fall through to Docker Hub cache                 │
│ ├── Developers use ONE registry URL for everything                 │
│ └── Security team controls what's accessible                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Vulnerability Scanning

```
┌─────────────────────────────────────────────────────────────────────┐
│           VULNERABILITY SCANNING                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Artifact Registry integrates with Container Analysis (also called  │
│ Artifact Analysis) to scan container images for known CVEs.        │
│                                                                       │
│ Two scanning modes:                                                │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │ 1. AUTOMATIC SCANNING                                        │   │
│ │    ├── Scans images on push to Artifact Registry             │   │
│ │    ├── Continuous monitoring (re-scans when new CVEs found)  │   │
│ │    ├── Enabled per project (Container Scanning API)          │   │
│ │    ├── Results visible in Console and via API                │   │
│ │    └── $0.26/image (first scan), continuous monitoring free  │   │
│ │                                                               │   │
│ │ 2. ON-DEMAND SCANNING                                        │   │
│ │    ├── Scan local images before pushing                      │   │
│ │    ├── Scan images in CI pipeline (fail build if critical)   │   │
│ │    ├── gcloud artifacts docker images scan IMAGE_URL          │   │
│ │    └── $0.26/scan                                            │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Enable automatic scanning:                                         │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Enable Container Scanning API                               │   │
│ │ gcloud services enable containerscanning.googleapis.com       │   │
│ │                                                               │   │
│ │ # Once enabled, all new image pushes are auto-scanned        │   │
│ │ # Results appear in:                                          │   │
│ │ # Console → Artifact Registry → [repo] → [image] →         │   │
│ │ #   Vulnerabilities tab                                      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ View scan results:                                                  │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # List vulnerabilities for an image                           │   │
│ │ gcloud artifacts docker images list-vulnerabilities \         │   │
│ │   us-central1-docker.pkg.dev/my-proj/docker-prod/\          │   │
│ │   backend-api:v1.2.3                                          │   │
│ │                                                               │   │
│ │ # Console view:                                               │   │
│ │ ┌────────────────────────────────────────────────────────┐   │   │
│ │ │ Vulnerability Summary for backend-api:v1.2.3            │   │   │
│ │ │                                                         │   │   │
│ │ │ ┌──────────┬───────┐                                   │   │   │
│ │ │ │ Severity │ Count │                                   │   │   │
│ │ │ ├──────────┼───────┤                                   │   │   │
│ │ │ │ CRITICAL │     0 │ ✅                                 │   │   │
│ │ │ │ HIGH     │     2 │ ⚠️                                │   │   │
│ │ │ │ MEDIUM   │     8 │                                   │   │   │
│ │ │ │ LOW      │    15 │                                   │   │   │
│ │ │ └──────────┴───────┘                                   │   │   │
│ │ │                                                         │   │   │
│ │ │ CVE-2024-1234 │ HIGH │ openssl 3.0.9 → fix: 3.0.12   │   │   │
│ │ │ CVE-2024-5678 │ HIGH │ curl 8.1.0 → fix: 8.4.0       │   │   │
│ │ │ ...                                                     │   │   │
│ │ └────────────────────────────────────────────────────────┘   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ On-demand scan in CI pipeline:                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # In cloudbuild.yaml — scan before deploy                    │   │
│ │ steps:                                                        │   │
│ │   - id: "scan"                                                │   │
│ │     name: "gcr.io/cloud-builders/gcloud"                      │   │
│ │     entrypoint: "bash"                                        │   │
│ │     args:                                                     │   │
│ │       - "-c"                                                  │   │
│ │       - |                                                     │   │
│ │         gcloud artifacts docker images scan \                │   │
│ │           ${_REGION}-docker.pkg.dev/${PROJECT_ID}/\          │   │
│ │             ${_REPO}/${_IMAGE}:${SHORT_SHA} \                │   │
│ │           --format='value(response.scan)' > /tmp/scan_id     │   │
│ │                                                               │   │
│ │         gcloud artifacts docker images \                     │   │
│ │           list-vulnerabilities $(cat /tmp/scan_id) \         │   │
│ │           --format='value(vulnerability.effectiveSeverity)' \│   │
│ │           | grep -q CRITICAL && \                            │   │
│ │           echo "CRITICAL vulnerability found!" && exit 1 || \│   │
│ │           echo "No critical vulnerabilities."                │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Binary Authorization integration:                                  │
│ ├── Only deploy images that passed vulnerability scan              │
│ ├── Require images to have zero CRITICAL CVEs                      │
│ ├── Enforce image signing (attestation)                            │
│ └── Works with GKE and Cloud Run                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Cleanup Policies & Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLEANUP POLICIES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Automatically delete old or unwanted artifacts to save storage     │
│ costs and keep repositories clean.                                 │
│                                                                       │
│ Policy types:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │ 1. DELETE policy — remove matching artifacts                 │   │
│ │    ├── By age: "Delete images older than 30 days"            │   │
│ │    ├── By tag: "Delete untagged images"                      │   │
│ │    ├── By version count: "Keep only last 10 versions"        │   │
│ │    └── By tag prefix: "Delete images tagged dev-*"           │   │
│ │                                                               │   │
│ │ 2. KEEP policy — protect matching artifacts from deletion    │   │
│ │    ├── By tag: "Always keep images tagged v*"                │   │
│ │    ├── By count: "Keep most recent 5 versions"               │   │
│ │    └── Keep policies take precedence over delete policies    │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Console: Artifact Registry → [repo] → Cleanup Policies            │
│                                                                       │
│ Example policy (JSON):                                             │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ [                                                             │   │
│ │   {                                                           │   │
│ │     "name": "keep-release-tags",                              │   │
│ │     "action": { "type": "Keep" },                            │   │
│ │     "condition": {                                            │   │
│ │       "tagState": "TAGGED",                                  │   │
│ │       "tagPrefixes": ["v"]                                   │   │
│ │     }                                                         │   │
│ │   },                                                          │   │
│ │   {                                                           │   │
│ │     "name": "keep-recent",                                    │   │
│ │     "action": { "type": "Keep" },                            │   │
│ │     "mostRecentVersions": {                                  │   │
│ │       "keepCount": 5                                          │   │
│ │     }                                                         │   │
│ │   },                                                          │   │
│ │   {                                                           │   │
│ │     "name": "delete-untagged",                                │   │
│ │     "action": { "type": "Delete" },                          │   │
│ │     "condition": {                                            │   │
│ │       "tagState": "UNTAGGED",                                │   │
│ │       "olderThan": "7d"                                      │   │
│ │     }                                                         │   │
│ │   },                                                          │   │
│ │   {                                                           │   │
│ │     "name": "delete-old-dev",                                 │   │
│ │     "action": { "type": "Delete" },                          │   │
│ │     "condition": {                                            │   │
│ │       "tagState": "TAGGED",                                  │   │
│ │       "tagPrefixes": ["dev-", "pr-"],                        │   │
│ │       "olderThan": "14d"                                     │   │
│ │     }                                                         │   │
│ │   }                                                           │   │
│ │ ]                                                             │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Apply cleanup policy:                                              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Set cleanup policies from a JSON file                       │   │
│ │ gcloud artifacts repositories set-cleanup-policies \           │   │
│ │   docker-prod \                                               │   │
│ │   --location=us-central1 \                                    │   │
│ │   --policy=cleanup-policy.json                                │   │
│ │                                                               │   │
│ │ # Dry run (preview what would be deleted)                    │   │
│ │ gcloud artifacts repositories set-cleanup-policies \           │   │
│ │   docker-prod \                                               │   │
│ │   --location=us-central1 \                                    │   │
│ │   --policy=cleanup-policy.json \                              │   │
│ │   --dry-run                                                   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ ⚡ Always add a KEEP policy first to protect important artifacts. │
│ ⚡ Use dry-run before applying to preview what gets deleted.      │
│ ⚡ Cleanup runs periodically (not immediately on policy set).     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: IAM & Security

```
┌─────────────────────────────────────────────────────────────────────┐
│           IAM ROLES FOR ARTIFACT REGISTRY                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌─────────────────────────────────┬──────────────────────────────┐ │
│ │ Role                            │ Permissions                   │ │
│ ├─────────────────────────────────┼──────────────────────────────┤ │
│ │ artifactregistry.reader         │ Pull/download artifacts      │ │
│ │                                 │ List repos and packages      │ │
│ │                                 │ View vulnerability results   │ │
│ │                                 │                              │ │
│ │ artifactregistry.writer         │ All reader permissions +     │ │
│ │                                 │ Push/upload artifacts        │ │
│ │                                 │ Delete artifact versions     │ │
│ │                                 │                              │ │
│ │ artifactregistry.repoAdmin     │ All writer permissions +     │ │
│ │                                 │ Update repo settings         │ │
│ │                                 │ Set cleanup policies         │ │
│ │                                 │                              │ │
│ │ artifactregistry.admin          │ All repoAdmin permissions +  │ │
│ │                                 │ Create/delete repos          │ │
│ │                                 │ Set IAM policies on repos    │ │
│ └─────────────────────────────────┴──────────────────────────────┘ │
│                                                                       │
│ Per-repo IAM (key advantage over Container Registry):              │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Grant reader to GKE SA for a specific repo only            │   │
│ │ gcloud artifacts repositories add-iam-policy-binding \        │   │
│ │   docker-prod \                                               │   │
│ │   --location=us-central1 \                                    │   │
│ │   --member="serviceAccount:gke-sa@my-proj.iam.\             │   │
│ │     gserviceaccount.com" \                                    │   │
│ │   --role="roles/artifactregistry.reader"                     │   │
│ │                                                               │   │
│ │ # Grant writer to Cloud Build SA for pushing                  │   │
│ │ gcloud artifacts repositories add-iam-policy-binding \        │   │
│ │   docker-prod \                                               │   │
│ │   --location=us-central1 \                                    │   │
│ │   --member="serviceAccount:123@cloudbuild.gserviceaccount.\  │   │
│ │     com" \                                                    │   │
│ │   --role="roles/artifactregistry.writer"                     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Security features:                                                  │
│ ├── Encryption at rest: Google-managed or CMEK                     │
│ ├── Encryption in transit: TLS (HTTPS)                             │
│ ├── Immutable tags: prevent overwriting released images            │
│ ├── VPC Service Controls: restrict registry to service perimeter  │
│ ├── Binary Authorization: enforce image signing for deployment    │
│ ├── Vulnerability scanning: automatic CVE detection                │
│ ├── Audit logging: who pushed/pulled what, when                   │
│ └── Tag history: see when tags were created/moved                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Cross-Region & Multi-Project

```
┌─────────────────────────────────────────────────────────────────────┐
│           CROSS-REGION & MULTI-PROJECT ACCESS                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. MULTI-REGION REPOSITORIES                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # Create a multi-region repo (replicated across region)      │   │
│ │ gcloud artifacts repositories create docker-global \          │   │
│ │   --repository-format=docker \                                │   │
│ │   --location=us \                                             │   │
│ │   --description="Multi-region Docker repo"                    │   │
│ │                                                               │   │
│ │ Available multi-region locations: us, europe, asia            │   │
│ │                                                               │   │
│ │ URL: us-docker.pkg.dev/my-proj/docker-global/image:tag       │   │
│ │                                                               │   │
│ │ ✅ Automatic replication across regions within the area       │   │
│ │ ✅ Lower latency for global pulls                             │   │
│ │ ⚠️ Slightly higher storage cost                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 2. CROSS-PROJECT ACCESS                                            │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Scenario: Central "platform" project hosts images,           │   │
│ │ "app-dev" and "app-prod" projects consume them.              │   │
│ │                                                               │   │
│ │ ┌──────────────┐    pull     ┌──────────────────────────┐   │   │
│ │ │ app-dev proj │────────────→│ platform proj            │   │   │
│ │ │ (GKE / Run)  │            │ Artifact Registry         │   │   │
│ │ └──────────────┘            │ "docker-shared"           │   │   │
│ │                              │                           │   │   │
│ │ ┌──────────────┐    pull     │ Images:                   │   │   │
│ │ │ app-prod proj│────────────→│ ├── base-image:latest    │   │   │
│ │ │ (GKE / Run)  │            │ ├── api:v1.2.3           │   │   │
│ │ └──────────────┘            │ └── worker:v1.0.0        │   │   │
│ │                              └──────────────────────────┘   │   │
│ │                                                               │   │
│ │ Grant cross-project access:                                  │   │
│ │ # Grant GKE SA in app-dev read access to platform repo      │   │
│ │ gcloud artifacts repositories add-iam-policy-binding \        │   │
│ │   docker-shared \                                             │   │
│ │   --project=platform-proj \                                   │   │
│ │   --location=us-central1 \                                    │   │
│ │   --member="serviceAccount:gke-sa@app-dev.iam.\             │   │
│ │     gserviceaccount.com" \                                    │   │
│ │   --role="roles/artifactregistry.reader"                     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ 3. REGION-TO-REGION COPY                                           │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ # No automatic cross-region replication for standard repos.  │   │
│ │ # Options:                                                    │   │
│ │ # A. Use multi-region repo (us, europe, asia)                │   │
│ │ # B. Use crane/skopeo to copy images between repos           │   │
│ │                                                               │   │
│ │ # Copy image from us-central1 to europe-west1                │   │
│ │ crane copy \                                                  │   │
│ │   us-central1-docker.pkg.dev/my-proj/docker-prod/api:v1.2.3 \│  │
│ │   europe-west1-docker.pkg.dev/my-proj/docker-eu/api:v1.2.3   │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 14: Terraform & CLI

### Terraform

```hcl
# ─────────────────────────────────────────────────────────────
# Docker Repository (Standard)
# ─────────────────────────────────────────────────────────────

resource "google_artifact_registry_repository" "docker_prod" {
  location      = "us-central1"
  repository_id = "docker-prod"
  format        = "DOCKER"
  description   = "Production Docker images"
  project       = "my-gcp-project"

  docker_config {
    immutable_tags = true  # Prevent tag overwriting
  }

  cleanup_policies {
    id     = "keep-release-tags"
    action = "KEEP"
    condition {
      tag_state    = "TAGGED"
      tag_prefixes = ["v"]
    }
  }

  cleanup_policies {
    id     = "keep-recent"
    action = "KEEP"
    most_recent_versions {
      keep_count = 10
    }
  }

  cleanup_policies {
    id     = "delete-untagged"
    action = "DELETE"
    condition {
      tag_state  = "UNTAGGED"
      older_than = "604800s"  # 7 days
    }
  }

  cleanup_policies {
    id     = "delete-old-dev"
    action = "DELETE"
    condition {
      tag_state    = "TAGGED"
      tag_prefixes = ["dev-", "pr-"]
      older_than   = "1209600s"  # 14 days
    }
  }

  labels = {
    environment = "production"
  }
}

# ─────────────────────────────────────────────────────────────
# npm Repository
# ─────────────────────────────────────────────────────────────

resource "google_artifact_registry_repository" "npm_internal" {
  location      = "us-central1"
  repository_id = "npm-internal"
  format        = "NPM"
  description   = "Internal npm packages"
  project       = "my-gcp-project"
}

# ─────────────────────────────────────────────────────────────
# Python Repository
# ─────────────────────────────────────────────────────────────

resource "google_artifact_registry_repository" "python_internal" {
  location      = "us-central1"
  repository_id = "python-internal"
  format        = "PYTHON"
  description   = "Internal Python packages"
  project       = "my-gcp-project"
}

# ─────────────────────────────────────────────────────────────
# Maven Repository
# ─────────────────────────────────────────────────────────────

resource "google_artifact_registry_repository" "maven_internal" {
  location      = "us-central1"
  repository_id = "maven-internal"
  format        = "MAVEN"
  description   = "Internal Maven artifacts"
  project       = "my-gcp-project"

  maven_config {
    allow_snapshot_overwrites = false
    version_policy            = "RELEASE"  # RELEASE, SNAPSHOT, or NONE
  }
}

# ─────────────────────────────────────────────────────────────
# Remote Repository (Docker Hub Cache)
# ─────────────────────────────────────────────────────────────

resource "google_artifact_registry_repository" "dockerhub_cache" {
  location      = "us-central1"
  repository_id = "dockerhub-cache"
  format        = "DOCKER"
  mode          = "REMOTE_REPOSITORY"
  description   = "Docker Hub caching proxy"
  project       = "my-gcp-project"

  remote_repository_config {
    description = "Docker Hub"
    docker_repository {
      public_repository = "DOCKER_HUB"
    }
  }
}

# ─────────────────────────────────────────────────────────────
# Remote Repository (npm cache)
# ─────────────────────────────────────────────────────────────

resource "google_artifact_registry_repository" "npmjs_cache" {
  location      = "us-central1"
  repository_id = "npmjs-cache"
  format        = "NPM"
  mode          = "REMOTE_REPOSITORY"
  project       = "my-gcp-project"

  remote_repository_config {
    description = "npmjs.com cache"
    npm_repository {
      public_repository = "NPMJS"
    }
  }
}

# ─────────────────────────────────────────────────────────────
# Virtual Repository (aggregates standard + remote)
# ─────────────────────────────────────────────────────────────

resource "google_artifact_registry_repository" "docker_virtual" {
  location      = "us-central1"
  repository_id = "docker-all"
  format        = "DOCKER"
  mode          = "VIRTUAL_REPOSITORY"
  project       = "my-gcp-project"

  virtual_repository_config {
    upstream_policies {
      id         = "docker-prod"
      repository = google_artifact_registry_repository.docker_prod.id
      priority   = 100
    }
    upstream_policies {
      id         = "dockerhub-cache"
      repository = google_artifact_registry_repository.dockerhub_cache.id
      priority   = 80
    }
  }
}

# ─────────────────────────────────────────────────────────────
# IAM — Per-Repo Access
# ─────────────────────────────────────────────────────────────

resource "google_artifact_registry_repository_iam_member" "gke_reader" {
  project    = "my-gcp-project"
  location   = "us-central1"
  repository = google_artifact_registry_repository.docker_prod.name
  role       = "roles/artifactregistry.reader"
  member     = "serviceAccount:gke-sa@my-gcp-project.iam.gserviceaccount.com"
}

resource "google_artifact_registry_repository_iam_member" "build_writer" {
  project    = "my-gcp-project"
  location   = "us-central1"
  repository = google_artifact_registry_repository.docker_prod.name
  role       = "roles/artifactregistry.writer"
  member     = "serviceAccount:cloud-build-deployer@my-gcp-project.iam.gserviceaccount.com"
}

# Cross-project: grant read access to another project's SA
resource "google_artifact_registry_repository_iam_member" "cross_project_reader" {
  project    = "my-gcp-project"
  location   = "us-central1"
  repository = google_artifact_registry_repository.docker_prod.name
  role       = "roles/artifactregistry.reader"
  member     = "serviceAccount:gke-sa@other-project.iam.gserviceaccount.com"
}
```

### gcloud CLI Reference

```bash
# ─────────────────────────────────────────────────────────────
# Repository Management
# ─────────────────────────────────────────────────────────────

# Create Docker repo
gcloud artifacts repositories create docker-prod \
  --repository-format=docker \
  --location=us-central1 \
  --description="Production Docker images" \
  --immutable-tags

# Create npm repo
gcloud artifacts repositories create npm-internal \
  --repository-format=npm \
  --location=us-central1

# Create Python repo
gcloud artifacts repositories create python-internal \
  --repository-format=python \
  --location=us-central1

# Create Maven repo
gcloud artifacts repositories create maven-internal \
  --repository-format=maven \
  --location=us-central1

# List repos
gcloud artifacts repositories list --location=us-central1

# Describe repo
gcloud artifacts repositories describe docker-prod \
  --location=us-central1

# Delete repo (⚠️ deletes all contents!)
gcloud artifacts repositories delete docker-prod \
  --location=us-central1

# ─────────────────────────────────────────────────────────────
# Docker Images
# ─────────────────────────────────────────────────────────────

# Configure Docker auth
gcloud auth configure-docker us-central1-docker.pkg.dev

# List images in a repo
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/my-proj/docker-prod

# List tags for an image
gcloud artifacts docker tags list \
  us-central1-docker.pkg.dev/my-proj/docker-prod/backend-api

# Delete an image tag
gcloud artifacts docker tags delete \
  us-central1-docker.pkg.dev/my-proj/docker-prod/backend-api:dev-old

# Delete an image (by digest)
gcloud artifacts docker images delete \
  us-central1-docker.pkg.dev/my-proj/docker-prod/\
backend-api@sha256:abc123...

# ─────────────────────────────────────────────────────────────
# Vulnerability Scanning
# ─────────────────────────────────────────────────────────────

# Enable scanning API
gcloud services enable containerscanning.googleapis.com

# On-demand scan
gcloud artifacts docker images scan \
  us-central1-docker.pkg.dev/my-proj/docker-prod/backend-api:v1.2.3

# List vulnerabilities
gcloud artifacts docker images list-vulnerabilities \
  us-central1-docker.pkg.dev/my-proj/docker-prod/backend-api:v1.2.3

# ─────────────────────────────────────────────────────────────
# Cleanup Policies
# ─────────────────────────────────────────────────────────────

# Set cleanup policies
gcloud artifacts repositories set-cleanup-policies docker-prod \
  --location=us-central1 \
  --policy=cleanup-policy.json

# Dry run (preview)
gcloud artifacts repositories set-cleanup-policies docker-prod \
  --location=us-central1 \
  --policy=cleanup-policy.json \
  --dry-run

# Delete all cleanup policies
gcloud artifacts repositories delete-cleanup-policies docker-prod \
  --location=us-central1

# ─────────────────────────────────────────────────────────────
# Remote Repositories
# ─────────────────────────────────────────────────────────────

# Create Docker Hub remote
gcloud artifacts repositories create dockerhub-cache \
  --repository-format=docker \
  --location=us-central1 \
  --mode=remote-repository \
  --remote-docker-repo=DOCKER-HUB

# Create npmjs remote
gcloud artifacts repositories create npmjs-cache \
  --repository-format=npm \
  --location=us-central1 \
  --mode=remote-repository \
  --remote-npm-repo=NPMJS

# Create PyPI remote
gcloud artifacts repositories create pypi-cache \
  --repository-format=python \
  --location=us-central1 \
  --mode=remote-repository \
  --remote-python-repo=PYPI

# Create Maven Central remote
gcloud artifacts repositories create maven-cache \
  --repository-format=maven \
  --location=us-central1 \
  --mode=remote-repository \
  --remote-mvn-repo=MAVEN-CENTRAL

# ─────────────────────────────────────────────────────────────
# IAM (Per-Repo)
# ─────────────────────────────────────────────────────────────

# Grant reader on a specific repo
gcloud artifacts repositories add-iam-policy-binding docker-prod \
  --location=us-central1 \
  --member="serviceAccount:gke-sa@my-proj.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"

# Grant writer
gcloud artifacts repositories add-iam-policy-binding docker-prod \
  --location=us-central1 \
  --member="serviceAccount:build@my-proj.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

# ─────────────────────────────────────────────────────────────
# Print Settings (generate client config)
# ─────────────────────────────────────────────────────────────

# npm .npmrc
gcloud artifacts print-settings npm \
  --repository=npm-internal \
  --location=us-central1 \
  --scope=@myorg

# Maven pom.xml / settings.xml
gcloud artifacts print-settings mvn \
  --repository=maven-internal \
  --location=us-central1

# Python pip.conf / .pypirc
gcloud artifacts print-settings python \
  --repository=python-internal \
  --location=us-central1

# Gradle build.gradle
gcloud artifacts print-settings gradle \
  --repository=maven-internal \
  --location=us-central1
```

---

## Part 15: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 1: COMPLETE CI/CD ARTIFACT FLOW                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Build → Scan → Push → Deploy with artifact governance.  │
│                                                                       │
│ Architecture:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  GitHub push → Cloud Build trigger                           │   │
│ │       │                                                       │   │
│ │       ▼                                                       │   │
│ │  ┌──────────────────────────┐                                │   │
│ │  │ Step 1: Build image       │                                │   │
│ │  │ (Kaniko + layer caching)  │                                │   │
│ │  └──────────┬────────────────┘                                │   │
│ │             ▼                                                  │   │
│ │  ┌──────────────────────────┐                                │   │
│ │  │ Step 2: Run tests         │                                │   │
│ │  │ (inside built image)      │                                │   │
│ │  └──────────┬────────────────┘                                │   │
│ │             ▼                                                  │   │
│ │  ┌──────────────────────────┐                                │   │
│ │  │ Step 3: Vuln scan         │                                │   │
│ │  │ (fail if CRITICAL CVEs)  │                                │   │
│ │  └──────────┬────────────────┘                                │   │
│ │             ▼                                                  │   │
│ │  ┌──────────────────────────┐    ┌────────────────────────┐  │   │
│ │  │ Step 4: Push image        │───→│ Artifact Registry       │  │   │
│ │  │ Tags: $SHA, v1.2.3, latest│    │ "docker-prod"           │  │   │
│ │  └──────────┬────────────────┘    │ (immutable tags)        │  │   │
│ │             │                     └────────────────────────┘  │   │
│ │             ▼                                                  │   │
│ │  ┌──────────────────────────┐    ┌────────────────────────┐  │   │
│ │  │ Step 5: Deploy to        │───→│ Cloud Run              │  │   │
│ │  │ Cloud Run                │    │ (pulls from AR)        │  │   │
│ │  └──────────────────────────┘    └────────────────────────┘  │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Cleanup policy on docker-prod:                                     │
│ ├── Keep: images tagged with v* (release tags)                     │
│ ├── Keep: most recent 10 versions per image                        │
│ ├── Delete: untagged images older than 7 days                      │
│ └── Delete: images tagged dev-* or pr-* older than 14 days        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 2: CENTRALIZED REGISTRY (MULTI-PROJECT)             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Platform team manages a central registry; app teams     │
│ push and pull images across projects.                              │
│                                                                       │
│ Architecture:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  Platform Project (platform-shared)                          │   │
│ │  ┌──────────────────────────────────────────────────────┐   │   │
│ │  │ Artifact Registry (us-central1)                       │   │   │
│ │  │                                                       │   │   │
│ │  │ ├── docker-base   ← Base images (python, node, etc.) │   │   │
│ │  │ ├── docker-prod   ← Production app images            │   │   │
│ │  │ ├── docker-staging← Staging app images                │   │   │
│ │  │ ├── npm-internal  ← Shared npm packages              │   │   │
│ │  │ └── dockerhub-cache ← Remote (Docker Hub proxy)      │   │   │
│ │  │                                                       │   │   │
│ │  │ IAM:                                                  │   │   │
│ │  │ ├── build-dev@dev-proj → writer on docker-staging    │   │   │
│ │  │ ├── build-prod@prod-proj → writer on docker-prod     │   │   │
│ │  │ ├── gke-dev@dev-proj → reader on docker-staging      │   │   │
│ │  │ └── gke-prod@prod-proj → reader on docker-prod       │   │   │
│ │  └──────────────────────────────────────────────────────┘   │   │
│ │                                                               │   │
│ │  Dev Project               Prod Project                      │   │
│ │  ┌──────────────┐          ┌──────────────┐                  │   │
│ │  │ Cloud Build  │──push──→ │ Cloud Build  │──push──→         │   │
│ │  │ (dev CI/CD)  │ staging  │ (prod CI/CD) │ prod             │   │
│ │  │              │          │              │                  │   │
│ │  │ GKE ←─pull── │          │ GKE ←─pull── │                  │   │
│ │  │ (staging)    │ staging  │ (production) │ prod             │   │
│ │  └──────────────┘          └──────────────┘                  │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Benefits:                                                           │
│ ├── Single source of truth for all images                          │
│ ├── Per-repo IAM controls who can push/pull what                   │
│ ├── Centralized vulnerability scanning and cleanup policies        │
│ ├── Consistent base images across all teams                        │
│ └── Platform team governs security; app teams own their images     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           PATTERN 3: SUPPLY CHAIN SECURITY                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Scenario: Secure the entire software supply chain from source      │
│ to deployment.                                                     │
│                                                                       │
│ Architecture:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                               │   │
│ │  1. Source (GitHub)                                           │   │
│ │       │ signed commits, branch protection                    │   │
│ │       ▼                                                       │   │
│ │  2. Build (Cloud Build)                                      │   │
│ │       │ generates SLSA provenance                            │   │
│ │       ▼                                                       │   │
│ │  3. Scan (Artifact Analysis)                                 │   │
│ │       │ vulnerability scan, fail on CRITICAL                 │   │
│ │       ▼                                                       │   │
│ │  4. Store (Artifact Registry)                                │   │
│ │       │ immutable tags, CMEK encryption                      │   │
│ │       │ cleanup policies for retention                       │   │
│ │       ▼                                                       │   │
│ │  5. Attest (Binary Authorization)                            │   │
│ │       │ sign image with attestor key                         │   │
│ │       │ verify: built by Cloud Build + passed scan           │   │
│ │       ▼                                                       │   │
│ │  6. Deploy (GKE / Cloud Run)                                 │   │
│ │       │ Binary Authorization enforcer:                       │   │
│ │       │ only deploy images with valid attestation            │   │
│ │       │ → Blocks unsigned or untested images                │   │
│ │       ▼                                                       │   │
│ │  7. Monitor (Container Analysis)                             │   │
│ │       └── Continuous CVE monitoring post-deployment          │   │
│ │           Alert if new CVE affects running images             │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ SLSA (Supply-chain Levels for Software Artifacts):                 │
│ ├── Cloud Build automatically generates SLSA Level 3 provenance   │
│ ├── Provenance = metadata proving WHERE and HOW image was built   │
│ ├── Stored alongside the image in Artifact Registry               │
│ └── Binary Authorization can require provenance for deployment    │
│                                                                       │
│ ⚡ This pattern implements SLSA Level 3 supply chain security.    │
│ ⚡ Relevant for SOC 2, FedRAMP, PCI-DSS compliance.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           ARTIFACT REGISTRY QUICK REFERENCE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Universal package manager (Docker, npm, Maven, Python, etc.)│
│ Replaces: Container Registry (gcr.io) → Artifact Registry (pkg.dev)│
│                                                                       │
│ URL format: {REGION}-{FORMAT}.pkg.dev/{PROJECT}/{REPO}/{PACKAGE}   │
│ Docker: us-central1-docker.pkg.dev/my-proj/docker-prod/api:v1     │
│ npm:    us-central1-npm.pkg.dev/my-proj/npm-internal/              │
│                                                                       │
│ Repository modes:                                                   │
│ ├── Standard — host your own artifacts                             │
│ ├── Remote — cache/proxy upstream registries                      │
│ └── Virtual — aggregate multiple repos into one endpoint          │
│                                                                       │
│ Supported formats:                                                  │
│ Docker, npm, Maven, Python, Go, Apt, Yum, KubeFlow               │
│                                                                       │
│ Key features:                                                       │
│ ├── Per-repo IAM (unlike GCR's per-project)                        │
│ ├── Immutable tags (prevent overwriting release tags)              │
│ ├── Cleanup policies (auto-delete old/untagged artifacts)          │
│ ├── Vulnerability scanning (automatic + on-demand)                 │
│ ├── CMEK encryption                                                │
│ └── VPC Service Controls                                           │
│                                                                       │
│ IAM roles:                                                          │
│ ├── artifactregistry.reader — pull/download                       │
│ ├── artifactregistry.writer — push/upload + reader                │
│ ├── artifactregistry.repoAdmin — settings + writer                │
│ └── artifactregistry.admin — create/delete repos + all            │
│                                                                       │
│ Key CLI:                                                            │
│ ├── gcloud artifacts repositories create NAME --format=docker ...  │
│ ├── gcloud artifacts docker images list REPO_URL                   │
│ ├── gcloud auth configure-docker REGION-docker.pkg.dev            │
│ ├── gcloud artifacts print-settings npm|mvn|python ...            │
│ ├── gcloud artifacts repositories set-cleanup-policies ...        │
│ └── gcloud artifacts docker images scan IMAGE_URL                 │
│                                                                       │
│ Pricing: $0.10/GB/month storage + $0.26/image scan                │
│ Free tier: 0.5 GB storage                                          │
│ Same-region pulls: free (no egress)                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What is a Container Registry? (Beginner Explanation)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONTAINER REGISTRY — THE SIMPLE EXPLANATION                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 💡 A container registry is like an "app store" for your Docker     │
│    images — a central place to store, version, and distribute      │
│    the containers your team builds.                                 │
│                                                                       │
│ Think of it this way:                                               │
│ ├── You write code                                                 │
│ ├── You package it into a Docker image (like a shipping container) │
│ ├── You push it to the registry (like uploading to the app store)  │
│ └── Other services pull it from the registry to run it             │
│                                                                       │
│ Why do you need one?                                                │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ WITHOUT a registry:                                           │   │
│ │ ├── Docker images live only on your laptop                   │   │
│ │ ├── Other developers can't access your images                │   │
│ │ ├── Your servers can't pull images to run                    │   │
│ │ └── No version tracking — chaos                              │   │
│ │                                                               │   │
│ │ WITH a registry (Artifact Registry):                          │   │
│ │ ├── Central storage — everyone pushes/pulls from one place   │   │
│ │ ├── Version control — tag images (v1.0, v2.0, latest)       │   │
│ │ ├── Access control — IAM decides who can push vs pull        │   │
│ │ ├── Security scanning — auto-detect vulnerabilities          │   │
│ │ └── CI/CD integration — automate build → push → deploy      │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ How it fits in your CI/CD pipeline:                                │
│                                                                       │
│   Developer         Cloud Build          Artifact Registry          │
│   pushes code       builds image         stores image               │
│       │                  │                    │                      │
│       ▼                  ▼                    ▼                      │
│   ┌───────┐         ┌────────┐          ┌──────────┐               │
│   │  Git  │────────▶│  CI    │─────────▶│ Registry │               │
│   │ Repo  │  trigger│ Build  │  push    │ (pkg.dev)│               │
│   └───────┘         └────────┘          └────┬─────┘               │
│                                              │ pull                 │
│                          ┌───────────────────┼──────────────┐      │
│                          ▼                   ▼              ▼      │
│                     ┌─────────┐        ┌──────────┐   ┌────────┐  │
│                     │   GKE   │        │Cloud Run │   │  GCE   │  │
│                     │ cluster │        │ service  │   │   VM   │  │
│                     └─────────┘        └──────────┘   └────────┘  │
│                                                                       │
│ ⚡ In GCP, Artifact Registry is the standard container registry.   │
│ ⚡ It also stores npm, Maven, Python packages — not just Docker.   │
│ ⚡ Old "Container Registry" (gcr.io) is deprecated — use pkg.dev. │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Managing & Deleting

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONSOLE → Artifact Registry → Repositories                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ VIEWING IMAGES AND TAGS                                             │
│ ─────────────────────                                               │
│                                                                       │
│ 1. Go to: Console → Artifact Registry → Repositories               │
│ 2. You see a list of all your repositories:                        │
│    ┌──────────────────────────────────────────────────────────┐    │
│    │ Repository         │ Format │ Location    │ Mode        │    │
│    ├────────────────────┼────────┼─────────────┼─────────────┤    │
│    │ docker-prod        │ Docker │ us-central1 │ Standard    │    │
│    │ npm-internal       │ npm    │ us-central1 │ Standard    │    │
│    │ dockerhub-cache    │ Docker │ us-central1 │ Remote      │    │
│    └──────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 3. Click a Docker repository (e.g., "docker-prod")                 │
│    → Shows all images in that repository                           │
│    ┌──────────────────────────────────────────────────────────┐    │
│    │ Image              │ Tags   │ Last pushed   │ Size      │    │
│    ├────────────────────┼────────┼───────────────┼───────────┤    │
│    │ backend-api        │ 5 tags │ 2 hours ago   │ 245 MB    │    │
│    │ frontend-app       │ 3 tags │ 1 day ago     │ 180 MB    │    │
│    │ worker             │ 2 tags │ 3 days ago    │ 120 MB    │    │
│    └──────────────────────────────────────────────────────────┘    │
│                                                                       │
│ 4. Click an image (e.g., "backend-api")                            │
│    → Shows all tags/digests for that image                         │
│    ┌──────────────────────────────────────────────────────────┐    │
│    │ Tag       │ Digest (sha256)   │ Created        │ Vulns  │    │
│    ├───────────┼───────────────────┼────────────────┼────────┤    │
│    │ v1.2.3    │ abc123...         │ 2 hours ago    │ 0 ✅    │    │
│    │ latest    │ abc123...         │ 2 hours ago    │ 0 ✅    │    │
│    │ v1.2.2    │ def456...         │ 1 day ago      │ 2 ⚠️    │    │
│    │ v1.2.1    │ ghi789...         │ 3 days ago     │ 5 🔴    │    │
│    │ (untagged)│ jkl012...         │ 5 days ago     │ —      │    │
│    └──────────────────────────────────────────────────────────┘    │
│                                                                       │
│                                                                       │
│ DELETING IMAGES AND TAGS                                            │
│ ────────────────────────                                            │
│                                                                       │
│ Delete a specific tag:                                              │
│ 1. Navigate to the image → find the tag                            │
│ 2. Check the box next to the tag (e.g., "v1.2.1")                 │
│ 3. Click [DELETE] at the top                                       │
│ 4. Confirm deletion                                                │
│    ⚠️ This only removes the tag — the digest remains if other      │
│       tags point to it                                              │
│                                                                       │
│ Delete an image (all tags + digests):                               │
│ 1. Go back to the repository view                                  │
│ 2. Check the box next to the image name (e.g., "worker")          │
│ 3. Click [DELETE] at the top                                       │
│ 4. Confirm → deletes ALL tags and ALL digests for that image       │
│    ⚠️ This is permanent — make sure no services are pulling this   │
│                                                                       │
│ Delete untagged images:                                             │
│ 1. Navigate to the image                                           │
│ 2. Check the box next to "(untagged)" entries                     │
│ 3. Click [DELETE]                                                  │
│ 4. Or better — set up a cleanup policy to auto-delete untagged    │
│                                                                       │
│                                                                       │
│ DELETING A REPOSITORY                                               │
│ ─────────────────────                                               │
│                                                                       │
│ 1. Go to: Console → Artifact Registry → Repositories               │
│ 2. Check the box next to the repository (e.g., "docker-prod")     │
│ 3. Click [DELETE] at the top                                       │
│ 4. Type the repository name to confirm                             │
│    ⚠️ This deletes ALL images, tags, and artifacts in the repo     │
│    ⚠️ Cannot be undone — make sure nothing depends on this repo    │
│                                                                       │
│                                                                       │
│ VIEWING VULNERABILITY SCAN RESULTS                                  │
│ ──────────────────────────────────                                  │
│                                                                       │
│ 1. Navigate to an image → click a specific tag/digest              │
│ 2. Click the "Vulnerabilities" tab (or the vuln count badge)      │
│    ┌──────────────────────────────────────────────────────────┐    │
│    │ Vulnerability Scan Results                                │    │
│    ├──────────────────────────────────────────────────────────┤    │
│    │                                                           │    │
│    │ Summary:  Critical: 0  High: 1  Medium: 3  Low: 8       │    │
│    │                                                           │    │
│    │ ┌─────────────┬──────────┬────────────┬───────────────┐  │    │
│    │ │ CVE ID      │ Severity │ Package    │ Fix Available │  │    │
│    │ ├─────────────┼──────────┼────────────┼───────────────┤  │    │
│    │ │ CVE-2024-xxx│ HIGH     │ openssl    │ ✅ 3.0.12      │  │    │
│    │ │ CVE-2024-yyy│ MEDIUM   │ curl       │ ✅ 8.5.0       │  │    │
│    │ │ CVE-2024-zzz│ MEDIUM   │ libc       │ ❌ No fix      │  │    │
│    │ └─────────────┴──────────┴────────────┴───────────────┘  │    │
│    │                                                           │    │
│    │ ⚡ Click any CVE to see full details + remediation        │    │
│    │ ⚡ Scanning is automatic if Container Analysis API is     │    │
│    │    enabled — no manual trigger needed                     │    │
│    │ ⚡ To enable: APIs & Services → Enable                   │    │
│    │    "Container Scanning API"                               │    │
│    └──────────────────────────────────────────────────────────┘    │
│                                                                       │
│ To trigger an on-demand scan (CLI):                                │
│ gcloud artifacts docker images scan \                              │
│   REGION-docker.pkg.dev/PROJECT/REPO/IMAGE:TAG                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 33: Cloud Deploy** → `33-cloud-deploy.md`
