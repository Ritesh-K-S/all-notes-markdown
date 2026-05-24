# Chapter 35: Azure Container Registry (ACR)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: ACR Fundamentals](#part-1-acr-fundamentals)
- [Part 2: Creating an ACR (Portal Walkthrough)](#part-2-creating-an-acr-portal-walkthrough)
- [Part 3: Pushing & Pulling Images](#part-3-pushing--pulling-images)
- [Part 4: ACR Tasks (Build in the Cloud)](#part-4-acr-tasks-build-in-the-cloud)
- [Part 5: Geo-Replication](#part-5-geo-replication)
- [Part 6: Security & Scanning](#part-6-security--scanning)
- [Part 7: Terraform & Bicep](#part-7-terraform--bicep)
- [Part 8: az CLI Reference](#part-8-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Container Registry (ACR) is a private Docker registry for storing and managing container images. Think of it like Docker Hub but private and integrated with Azure services (AKS, App Service, Container Instances, etc.). It supports Docker images, Helm charts, OCI artifacts, and automated image builds.

```
What you'll learn:
├── ACR Fundamentals
│   ├── What is a container registry
│   ├── SKU comparison (Basic, Standard, Premium)
│   └── ACR vs Docker Hub vs GitHub Container Registry
├── Creating an ACR (Portal)
├── Pushing & Pulling Images (docker push/pull)
├── ACR Tasks (build images in the cloud)
├── Geo-Replication (Premium SKU)
├── Security & Vulnerability Scanning
├── Terraform, Bicep, az CLI
└── Quick reference
```

---

## Part 1: ACR Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACR CONCEPT                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is a Container Registry?                                       │
│ ├── A storage service for Docker/container images                │
│ ├── Like a library of pre-built application packages             │
│ ├── You push (upload) images and pull (download) them           │
│ └── Used by AKS, App Service, CI/CD pipelines                   │
│                                                                       │
│ Registry URL: myregistry.azurecr.io                                │
│ Full image: myregistry.azurecr.io/myapp:v1.0                     │
│                                                                       │
│ SKU comparison:                                                      │
│ ┌──────────────┬─────────┬──────────┬──────────────┐             │
│ │ Feature      │ Basic   │ Standard │ Premium      │             │
│ ├──────────────┼─────────┼──────────┼──────────────┤             │
│ │ Storage      │ 10 GB   │ 100 GB   │ 500 GB       │             │
│ │ Throughput   │ Low     │ Medium   │ High         │             │
│ │ Webhooks     │ 2       │ 10       │ 500          │             │
│ │ Geo-repl     │ No      │ No       │ Yes ✅       │             │
│ │ Private link │ No      │ No       │ Yes ✅       │             │
│ │ Content trust│ No      │ No       │ Yes ✅       │             │
│ │ Zone redund  │ No      │ No       │ Yes ✅       │             │
│ │ Price/month  │ ~$5     │ ~$20     │ ~$50+        │             │
│ └──────────────┴─────────┴──────────┴──────────────┘             │
│                                                                       │
│ When to use which:                                                   │
│ ├── Basic: Dev/test, small teams, low usage                      │
│ ├── Standard: Production, medium usage                            │
│ └── Premium: Enterprise, geo-replication, private link           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating an ACR (Portal Walkthrough)

```
Console → Container registries → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE CONTAINER REGISTRY                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Basics ──                                                        │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-containers-prod ▼]                             │
│                                                                       │
│ Registry name: [mycompanyacr]                                       │
│ ⚡ Must be globally unique. Creates: mycompanyacr.azurecr.io     │
│ ⚡ Only alphanumeric (no hyphens, no dots)                       │
│                                                                       │
│ Location: [Central India ▼]                                        │
│ Availability zones: ☐ (Premium only)                              │
│                                                                       │
│ SKU: [Standard ▼]                                                 │
│ ├── Basic ($5/mo)                                                │
│ ├── Standard ($20/mo) ← recommended for production             │
│ └── Premium ($50/mo) ← geo-replication, private link           │
│                                                                       │
│ ── Networking (Premium only) ──                                    │
│ Connectivity: ● Public endpoint  ○ Private endpoint              │
│                                                                       │
│ ── Encryption (Premium only) ──                                    │
│ Customer-managed key: ○ Disabled  ○ Enabled                     │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Pushing & Pulling Images

```
┌─────────────────────────────────────────────────────────────────────┐
│           PUSH & PULL IMAGES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Login to registry                                           │
│ az acr login --name mycompanyacr                                   │
│ ⚡ Uses your Azure credentials (no password needed)              │
│                                                                       │
│ Step 2: Tag your local image                                        │
│ docker tag myapp:latest mycompanyacr.azurecr.io/myapp:v1.0       │
│                                                                       │
│ Image naming convention:                                             │
│ {registry}.azurecr.io/{repository}:{tag}                           │
│ mycompanyacr.azurecr.io/myapp:v1.0                                │
│ mycompanyacr.azurecr.io/frontend/react-app:latest                 │
│ mycompanyacr.azurecr.io/backend/api:2024.01.15                   │
│                                                                       │
│ Step 3: Push to ACR                                                 │
│ docker push mycompanyacr.azurecr.io/myapp:v1.0                   │
│                                                                       │
│ Step 4: Pull from ACR (on another machine, AKS, etc.)             │
│ docker pull mycompanyacr.azurecr.io/myapp:v1.0                   │
│                                                                       │
│ View in Portal:                                                      │
│ Container registry → Repositories → myapp → Tags → v1.0        │
│                                                                       │
│ Multi-arch images:                                                   │
│ docker buildx build --platform linux/amd64,linux/arm64 \          │
│   -t mycompanyacr.azurecr.io/myapp:v1.0 --push .                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: ACR Tasks (Build in the Cloud)

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACR TASKS                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Build images in the cloud WITHOUT a local Docker installation!    │
│                                                                       │
│ Quick build (one-off):                                               │
│ az acr build --registry mycompanyacr --image myapp:v1.0 .        │
│ ⚡ Sends your source code to ACR, builds there, pushes image.   │
│ ⚡ No Docker Desktop needed on your machine!                     │
│                                                                       │
│ Scheduled tasks (automated builds):                                  │
│ az acr task create \                                                │
│   --registry mycompanyacr \                                        │
│   --name build-myapp \                                             │
│   --image myapp:{{.Run.ID}} \                                    │
│   --context https://github.com/mycompany/myapp.git \             │
│   --file Dockerfile \                                               │
│   --git-access-token $GITHUB_TOKEN \                               │
│   --commit-trigger-enabled true                                    │
│                                                                       │
│ Task triggers:                                                       │
│ ├── Source commit: Build when code is pushed to GitHub           │
│ ├── Base image update: Rebuild when base image changes           │
│ │   ⚡ If node:20 gets a security patch, your image auto-rebuilds│
│ ├── Schedule: Build on a cron schedule                            │
│ └── Manual: az acr task run --name build-myapp                  │
│                                                                       │
│ Multi-step tasks (acr-task.yaml):                                   │
│ steps:                                                               │
│   - build: -t mycompanyacr.azurecr.io/myapp:{{.Run.ID}} .       │
│   - push: [mycompanyacr.azurecr.io/myapp:{{.Run.ID}}]           │
│   - cmd: mycompanyacr.azurecr.io/myapp:{{.Run.ID}} npm test    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Geo-Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│           GEO-REPLICATION (Premium only)                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Container registry → Geo-replications → [+ Add]                  │
│                                                                       │
│ ┌──────────────────────────────────────────────────┐              │
│ │ Region: [East US ▼]                              │              │
│ │ Zone redundancy: ☑                               │              │
│ │                                                    │              │
│ │ [Create]                                           │              │
│ └──────────────────────────────────────────────────┘              │
│                                                                       │
│ How it works:                                                        │
│ ├── Push image once → Automatically replicated to all regions  │
│ ├── Pull from nearest region (faster!)                           │
│ ├── Same image URL works everywhere                              │
│ └── If one region goes down, pull from another                  │
│                                                                       │
│ Example:                                                             │
│ Registry: mycompanyacr.azurecr.io                                  │
│ ├── Central India (primary) ✅                                   │
│ ├── East US (replica) ✅                                         │
│ └── West Europe (replica) ✅                                     │
│                                                                       │
│ AKS in East US pulls from East US replica (fast!)               │
│ AKS in India pulls from Central India (fast!)                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Security & Scanning

```
Authentication methods:
├── az acr login (Azure CLI - interactive dev)
├── Service Principal (CI/CD pipelines)
├── Managed Identity (AKS, App Service - no credentials!)
├── Admin user (quick testing only, not for production!)
└── Repository-scoped tokens (limited access)

Enable admin user (not recommended for prod):
  Container registry → Access keys → Admin user: Enable

Vulnerability scanning:
├── Microsoft Defender for Containers
│   ├── Scans images on push and periodically
│   ├── Finds OS and language vulnerabilities
│   └── Shows CVE details and remediation
├── Enable: Security Center → Defender plans → Containers → On
└── View: Container registry → Security → Vulnerability assessment

Content trust (Premium):
├── Image signing with Docker Content Trust
├── Ensures images haven't been tampered with
└── az acr config content-trust update --registry mycompanyacr --status enabled

Private link (Premium):
├── Access ACR from within your VNet only
├── No public internet exposure
└── Container registry → Networking → Private endpoint
```

---

## Part 7: Terraform & Bicep

### Terraform

```hcl
resource "azurerm_container_registry" "main" {
  name                = "mycompanyacr"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard"
  admin_enabled       = false

  # Premium features
  # georeplications {
  #   location = "East US"
  #   zone_redundancy_enabled = true
  # }
}

# Grant AKS access to pull images
resource "azurerm_role_assignment" "aks_acr" {
  principal_id         = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
  role_definition_name = "AcrPull"
  scope                = azurerm_container_registry.main.id
}
```

### Bicep

```bicep
resource acr 'Microsoft.ContainerRegistry/registries@2023-07-01' = {
  name: 'mycompanyacr'
  location: resourceGroup().location
  sku: {
    name: 'Standard'
  }
  properties: {
    adminUserEnabled: false
  }
}
```

---

## Part 8: az CLI Reference

```bash
# Create container registry
az acr create \
  --name mycompanyacr \
  --resource-group rg-containers-prod \
  --sku Standard

# Login to registry
az acr login --name mycompanyacr

# Build image in ACR (no local Docker needed!)
az acr build --registry mycompanyacr --image myapp:v1.0 .

# List repositories
az acr repository list --name mycompanyacr --output table

# List tags for a repository
az acr repository show-tags --name mycompanyacr --repository myapp --output table

# Show image details
az acr repository show --name mycompanyacr --repository myapp

# Delete an image tag
az acr repository delete --name mycompanyacr --image myapp:v1.0 --yes

# Attach ACR to AKS (grant pull access)
az aks update \
  --name myaks-cluster \
  --resource-group rg-aks \
  --attach-acr mycompanyacr

# List registries
az acr list --resource-group rg-containers-prod --output table

# Delete registry
az acr delete --name mycompanyacr --resource-group rg-containers-prod --yes
```

---

## Real-World Patterns

### Pattern 1: CI/CD with ACR and AKS

```
┌─────────────────────────────────────────────────┐
│  Container CI/CD Pipeline                       │
├─────────────────────────────────────────────────┤
│                                                 │
│  Git Push ─→ Build Pipeline                     │
│                  │                              │
│                  ▼                              │
│          Docker Build + Push to ACR            │
│          (acr.azurecr.io/myapp:v1.2)            │
│                  │                              │
│                  ▼                              │
│          ACR Task: Vulnerability Scan            │
│                  │                              │
│                  ▼                              │
│          AKS Deployment (kubectl apply)          │
│          (pulls image via managed identity)      │
│                                                 │
│  Geo-replication: ACR replicated to 2 regions   │
│  for fast pulls from regional AKS clusters.     │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
ACR URL: {name}.azurecr.io
Image format: {registry}.azurecr.io/{repo}:{tag}

SKUs: Basic ($5), Standard ($20), Premium ($50+)
Premium extras: Geo-replication, Private link, Content trust

Push: docker tag + docker push  OR  az acr build (cloud build)
Pull: docker pull  OR  az acr login + docker pull
Auth: az acr login (dev), Managed Identity (AKS), Service Principal (CI)

ACR Tasks: Build images in cloud, auto-rebuild on base image update
Scanning: Microsoft Defender for Containers (CVE detection)

AKS integration: az aks update --attach-acr (grants AcrPull role)
```

---

## What's Next?

Next chapter: [Chapter 36: ARM Templates & Bicep](36-arm-bicep.md) — Infrastructure as Code with ARM template structure, Bicep language, parameters, modules, and deployment stacks.
