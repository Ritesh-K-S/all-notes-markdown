# Chapter 33: Azure DevOps - Pipelines

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Pipeline Fundamentals](#part-1-pipeline-fundamentals)
- [Part 2: Creating a Pipeline (Portal Walkthrough)](#part-2-creating-a-pipeline-portal-walkthrough)
- [Part 3: YAML Pipeline Deep Dive](#part-3-yaml-pipeline-deep-dive)
- [Part 4: Variables & Variable Groups](#part-4-variables--variable-groups)
- [Part 5: Environments & Approvals](#part-5-environments--approvals)
- [Part 6: Templates & Reuse](#part-6-templates--reuse)
- [Part 7: Triggers](#part-7-triggers)
- [Part 8: Service Connections](#part-8-service-connections)
- [Part 9: az CLI Reference](#part-9-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Pipelines is a CI/CD service that automatically builds, tests, and deploys your code. You define your pipeline as code in a YAML file (`azure-pipelines.yml`) that lives in your repo. Pipelines can deploy to any cloud (Azure, AWS, GCP) or on-premises.

```
What you'll learn:
├── Pipeline Fundamentals
│   ├── CI (Continuous Integration) vs CD (Continuous Deployment)
│   ├── YAML vs Classic (UI) pipelines
│   └── Pipeline concepts (stages, jobs, steps, tasks)
├── Creating a Pipeline (Portal)
├── YAML Pipeline Deep Dive (full syntax)
├── Variables & Variable Groups
├── Environments & Approvals (deployment gates)
├── Templates (reuse across pipelines)
├── Triggers (CI, PR, scheduled)
├── Service Connections (connect to Azure, Docker Hub, etc.)
└── az CLI reference
```

---

## Part 1: Pipeline Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           PIPELINE CONCEPTS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ CI = Continuous Integration                                          │
│ ├── Automatically build + test when code is pushed               │
│ └── Catch bugs early, every commit is verified                   │
│                                                                       │
│ CD = Continuous Deployment / Delivery                                │
│ ├── Automatically deploy to staging/production after CI passes  │
│ ├── Delivery = deploy to staging, manual approval for prod      │
│ └── Deployment = fully automated, no manual steps               │
│                                                                       │
│ Pipeline hierarchy:                                                  │
│                                                                       │
│ Pipeline (azure-pipelines.yml)                                      │
│ ├── Stage: Build                                                  │
│ │   └── Job: BuildApp                                            │
│ │       ├── Step: Install dependencies (npm install)            │
│ │       ├── Step: Run tests (npm test)                          │
│ │       └── Step: Build artifact (npm run build)                │
│ ├── Stage: Deploy to Staging                                     │
│ │   └── Job: DeployStaging                                      │
│ │       ├── Step: Download artifact                              │
│ │       └── Step: Deploy to App Service                         │
│ └── Stage: Deploy to Production                                  │
│     └── Job: DeployProd (requires approval!)                    │
│         ├── Step: Download artifact                              │
│         └── Step: Deploy to App Service                         │
│                                                                       │
│ Agent Pools:                                                         │
│ ├── Microsoft-hosted: ubuntu-latest, windows-latest, macos-latest│
│ │   ⚡ Pre-configured VMs, no maintenance. Free tier: 1800 min/mo│
│ └── Self-hosted: Your own VMs (more control, faster, no limits) │
│                                                                       │
│ YAML vs Classic:                                                     │
│ ├── YAML (recommended): Pipeline as code, version controlled   │
│ └── Classic (UI editor): Visual designer, being deprecated      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Pipeline (Portal Walkthrough)

```
Project → Pipelines → Create Pipeline

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE PIPELINE                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Where is your code?                                         │
│ ├── Azure Repos Git ← (if using Azure DevOps)                   │
│ ├── GitHub ← (most common)                                      │
│ ├── Bitbucket Cloud                                               │
│ └── Other Git                                                     │
│                                                                       │
│ Step 2: Select repository                                           │
│ [myapp ▼]                                                          │
│                                                                       │
│ Step 3: Configure your pipeline                                     │
│ ├── Starter pipeline (minimal YAML)                              │
│ ├── Node.js (npm install, build, test)                           │
│ ├── .NET Core                                                     │
│ ├── Python                                                        │
│ ├── Docker (build & push image)                                  │
│ ├── Existing Azure Pipelines YAML file                           │
│ └── Many more templates...                                       │
│                                                                       │
│ Step 4: Review your pipeline YAML                                   │
│ ⚡ Edit the YAML, then click [Save and run]                     │
│                                                                       │
│ [Save and run] → Commits azure-pipelines.yml to your repo       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: YAML Pipeline Deep Dive

### Minimal Pipeline

```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - script: echo Hello, world!
    displayName: 'Run a one-line script'
```

### Full Multi-Stage Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - release/*
    exclude:
      - feature/experimental

pr:
  branches:
    include:
      - main

variables:
  - name: buildConfiguration
    value: 'Release'
  - group: 'production-secrets'  # Variable group from Library

stages:
  # ──── STAGE 1: BUILD ────
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: BuildJob
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'
            displayName: 'Install Node.js'

          - script: npm ci
            displayName: 'Install dependencies'

          - script: npm run test -- --coverage
            displayName: 'Run tests'

          - script: npm run build
            displayName: 'Build application'

          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: '$(Build.ArtifactStagingDirectory)'
              artifactName: 'drop'

  # ──── STAGE 2: DEPLOY TO STAGING ────
  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: DeployStagingJob
        environment: 'staging'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    appType: 'webAppLinux'
                    appName: 'myapp-staging'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'

  # ──── STAGE 3: DEPLOY TO PRODUCTION ────
  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: DeployProdJob
        environment: 'production'     # Has approval gate!
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'Azure-ServiceConnection'
                    appType: 'webAppLinux'
                    appName: 'myapp-prod'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
```

### Key Tasks

```
Common built-in tasks:
├── script / bash / powershell → Run scripts
├── task: NodeTool@0 → Install Node.js
├── task: UseDotNet@2 → Install .NET
├── task: UsePythonVersion@0 → Install Python
├── task: Docker@2 → Build & push Docker images
├── task: AzureWebApp@1 → Deploy to App Service
├── task: AzureCLI@2 → Run az CLI commands
├── task: KubernetesManifest@0 → Deploy to AKS
├── task: PublishBuildArtifacts@1 → Save build output
├── task: DownloadBuildArtifacts@1 → Get build output
├── task: AzureKeyVault@2 → Fetch secrets from Key Vault
└── task: CopyFiles@2 → Copy files between directories
```

---

## Part 4: Variables & Variable Groups

```
┌─────────────────────────────────────────────────────────────────────┐
│           VARIABLES                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Inline variables (in YAML):                                      │
│    variables:                                                        │
│      appName: 'myapp'                                              │
│      buildConfig: 'Release'                                        │
│    Usage: $(appName)                                                │
│                                                                       │
│ 2. Pipeline variables (in UI):                                      │
│    Pipeline → Edit → Variables → [+ Add]                        │
│    Name: [DOCKER_PASSWORD]                                         │
│    Value: [●●●●●●●●]                                              │
│    ☑ Keep this value secret ← won't show in logs!              │
│                                                                       │
│ 3. Variable Groups (shared across pipelines):                       │
│    Pipelines → Library → Variable groups → [+ Variable group]  │
│    Name: [production-secrets]                                      │
│    Variables:                                                        │
│      DB_CONNECTION_STRING = ●●●●●●●●                             │
│      API_KEY = ●●●●●●●●                                          │
│    Link to Azure Key Vault: ☑                                    │
│    ⚡ Fetches secrets directly from Key Vault at runtime!        │
│                                                                       │
│    Reference in YAML:                                               │
│    variables:                                                        │
│      - group: 'production-secrets'                                │
│                                                                       │
│ 4. Predefined variables:                                            │
│    $(Build.BuildId) → Unique build number                        │
│    $(Build.SourceBranch) → refs/heads/main                      │
│    $(Build.ArtifactStagingDirectory) → staging dir              │
│    $(System.DefaultWorkingDirectory) → source code dir          │
│    $(Pipeline.Workspace) → workspace root                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Environments & Approvals

```
┌─────────────────────────────────────────────────────────────────────┐
│           ENVIRONMENTS & APPROVALS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pipelines → Environments → [New environment]                     │
│                                                                       │
│ Name: [production]                                                  │
│ Description: [Production environment]                              │
│ Resource: None / Kubernetes / Virtual machines                    │
│                                                                       │
│ After creating, add approval gates:                                 │
│ Environment → production → ⋮ → Approvals and checks            │
│                                                                       │
│ ☑ Approvals:                                                       │
│   Approvers: [team-lead@company.com, devops@company.com]        │
│   Minimum approvals: [1]                                          │
│   Timeout: [72 hours]                                             │
│   ⚡ Pipeline pauses at production stage until someone approves!│
│                                                                       │
│ ☑ Branch control:                                                  │
│   Allowed branches: [refs/heads/main, refs/heads/release/*]    │
│   ⚡ Only deployments from main/release branches allowed!       │
│                                                                       │
│ ☑ Business hours:                                                  │
│   Deploy only: Mon-Fri, 9 AM - 5 PM IST                        │
│   ⚡ Prevents weekend/night deployments!                         │
│                                                                       │
│ Deployment flow:                                                    │
│ Push code → Build passes → Deploy staging (auto)                │
│          → Waiting for approval... ⏳                             │
│          → Manager approves ✅                                    │
│          → Deploy production 🚀                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Templates & Reuse

```yaml
# templates/build-template.yml
parameters:
  - name: nodeVersion
    default: '20.x'
  - name: buildCommand
    default: 'npm run build'

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: ${{ parameters.nodeVersion }}
  - script: npm ci
    displayName: 'Install dependencies'
  - script: npm test
    displayName: 'Run tests'
  - script: ${{ parameters.buildCommand }}
    displayName: 'Build'
```

```yaml
# azure-pipelines.yml — using the template
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - template: templates/build-template.yml
    parameters:
      nodeVersion: '20.x'
      buildCommand: 'npm run build:prod'
```

```
Templates save you from copy-pasting pipeline code!
├── Step templates → Reuse steps
├── Job templates → Reuse entire jobs
├── Stage templates → Reuse stages
└── Variable templates → Reuse variable definitions
```

---

## Part 7: Triggers

```
Trigger types:
├── CI trigger (push):
│   trigger:
│     branches:
│       include: [main, release/*]
│       exclude: [feature/experimental]
│     paths:
│       include: [src/*, api/*]
│       exclude: [docs/*]
│
├── PR trigger (pull request):
│   pr:
│     branches:
│       include: [main]
│     paths:
│       include: [src/*]
│
├── Scheduled trigger:
│   schedules:
│     - cron: "0 0 * * *"   # midnight daily
│       displayName: 'Nightly build'
│       branches:
│         include: [main]
│       always: true   # run even if no changes
│
└── Pipeline trigger (after another pipeline):
    resources:
      pipelines:
        - pipeline: buildPipeline
          source: 'Build Pipeline'
          trigger:
            branches:
              include: [main]
```

---

## Part 8: Service Connections

```
┌─────────────────────────────────────────────────────────────────────┐
│           SERVICE CONNECTIONS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Project Settings → Service connections → [New service connection]│
│                                                                       │
│ Types:                                                               │
│ ├── Azure Resource Manager ← Most common                        │
│ │   Authentication: Service principal (auto) / Managed identity │
│ │   Scope: Subscription or Resource Group                       │
│ │   Name: [Azure-ServiceConnection]                             │
│ │   ⚡ Pipeline gets permission to deploy to your Azure sub!   │
│ │                                                                  │
│ ├── Docker Registry                                               │
│ │   Registry type: Azure Container Registry / Docker Hub        │
│ │                                                                  │
│ ├── Kubernetes                                                     │
│ │   Cluster: AKS / kubeconfig                                   │
│ │                                                                  │
│ ├── GitHub                                                         │
│ │   Personal access token / OAuth                                │
│ │                                                                  │
│ └── Generic (webhook, SSH, etc.)                                 │
│                                                                       │
│ Reference in YAML:                                                  │
│   azureSubscription: 'Azure-ServiceConnection'                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: az CLI Reference

```bash
# List pipelines
az pipelines list --output table

# Create pipeline from YAML
az pipelines create \
  --name "Build Pipeline" \
  --repository myapp \
  --branch main \
  --yml-path azure-pipelines.yml

# Run pipeline manually
az pipelines run --name "Build Pipeline"

# Show pipeline runs
az pipelines runs list --output table

# Show specific run details
az pipelines runs show --id 42

# Create variable group
az pipelines variable-group create \
  --name "production-secrets" \
  --variables DB_HOST=mydb.database.windows.net API_VERSION=v2

# Delete pipeline
az pipelines delete --id 5 --yes
```

---

## Real-World Patterns

### Pattern 1: Multi-Stage CI/CD Pipeline

```
┌─────────────────────────────────────────────────┐
│  Build ─→ Dev ─→ Staging ─→ Prod               │
├─────────────────────────────────────────────────┤
│  Stage 1: Build                                 │
│  - Restore packages, compile, run unit tests   │
│  - Publish artifact                            │
│                                                 │
│  Stage 2: Deploy to Dev (auto)                  │
│  - Deploy to App Service (dev slot)             │
│  - Run integration tests                       │
│                                                 │
│  Stage 3: Deploy to Staging (auto)              │
│  - Deploy to App Service (staging slot)         │
│  - Run smoke tests + load tests                │
│                                                 │
│  Stage 4: Deploy to Prod (manual approval)      │
│  - Approval gate: Release Manager              │
│  - Blue/green swap deployment slots             │
│  - Post-deployment health check                │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Pipeline YAML file: azure-pipelines.yml (lives in repo root)
Hierarchy: Pipeline → Stage → Job → Step

Triggers: CI (push), PR (pull request), Scheduled (cron), Pipeline
Agent pools: ubuntu-latest, windows-latest, macos-latest
Variables: Inline (YAML), UI secrets, Variable Groups, Key Vault

Environments: staging, production (with approval gates)
Service Connections: Connect pipeline to Azure subscription

Key tasks: AzureWebApp@1, Docker@2, AzureCLI@2, KubernetesManifest@0
Templates: Reusable YAML fragments (step/job/stage/variable)
```

---

## What's Next?

Next chapter: [Chapter 34: Azure DevOps - Artifacts](34-azure-devops-artifacts.md) — Package management with feeds, upstream sources, and multiple package types.
