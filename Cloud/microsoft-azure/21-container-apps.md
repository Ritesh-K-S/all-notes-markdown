# Chapter 21: Azure Container Apps

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Container Apps Fundamentals](#part-1-container-apps-fundamentals)
- [Part 2: Creating a Container App (Full Portal Walkthrough)](#part-2-creating-a-container-app-full-portal-walkthrough)
- [Part 3: Scaling Rules](#part-3-scaling-rules)
- [Part 4: Revisions & Traffic Splitting](#part-4-revisions--traffic-splitting)
- [Part 5: Dapr Integration](#part-5-dapr-integration)
- [Part 6: Container Apps Jobs](#part-6-container-apps-jobs)
- [Part 7: Networking & Custom Domains](#part-7-networking--custom-domains)
- [Part 8: Terraform & Bicep](#part-8-terraform--bicep)
- [Part 9: az CLI Reference](#part-9-az-cli-reference)
- [Part 10: Real-World Patterns](#part-10-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Container Apps is a serverless container platform built on top of Kubernetes (AKS), KEDA (event-driven autoscaling), and Dapr (distributed application runtime). It offers the simplicity of a PaaS with the power of containers — you deploy containers that auto-scale (including to zero) based on HTTP traffic, events, or schedules, without managing any infrastructure.

```
What you'll learn:
├── Container Apps Fundamentals
│   ├── What & why (serverless containers, managed everything)
│   ├── Container Apps vs AKS vs ACI vs App Service
│   └── Key concepts (Environment, App, Revision, Replica)
├── Creating a Container App (Full Portal Walkthrough)
│   ├── Basics (name, region, environment)
│   ├── Container (image, resources, env vars)
│   ├── Ingress (HTTP, TCP, external/internal)
│   ├── Scaling (rules, min/max replicas)
│   └── Dapr, Secrets, Volumes
├── Container Apps Environment
├── Revisions & Traffic Splitting
├── Scaling Rules (HTTP, KEDA, schedule)
├── Dapr Integration
├── Jobs (scheduled/event-driven)
├── Custom Domains & Managed Certificates
├── Networking (VNet, internal, private)
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: Container Apps Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONTAINER APPS CONCEPT                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Container Apps?                                             │
│ ├── Serverless container platform (no infra management!)         │
│ ├── Deploy containers → they auto-scale (including to zero!)   │
│ ├── Built on: AKS + KEDA + Envoy + Dapr (you don't see these) │
│ ├── HTTP services, background workers, event processors        │
│ ├── Built-in service discovery, load balancing, TLS            │
│ └── Dapr integration for microservices (pub/sub, state, etc.)  │
│                                                                       │
│ Key concepts:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Container Apps Environment (shared boundary)                 │  │
│ │ ├── Shared VNet, Log Analytics, Dapr components             │  │
│ │ ├── App 1: frontend (3 replicas)                            │  │
│ │ │   ├── Revision: frontend--abc123 (active, 100% traffic) │  │
│ │ │   └── Revision: frontend--def456 (inactive, 0%)         │  │
│ │ ├── App 2: backend-api (2 replicas)                        │  │
│ │ ├── App 3: worker (0-10 replicas, KEDA)                    │  │
│ │ └── Job: nightly-etl (scheduled, runs at 2 AM)            │  │
│ │                                                              │  │
│ │ ⚡ Apps in same environment can discover each other:         │  │
│ │   http://backend-api (internal DNS, automatic!)            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Hierarchy:                                                           │
│ Environment → App → Revision → Replica (container instance)      │
│                                                                       │
│ ├── Environment: Isolation boundary (like K8s namespace)        │
│ ├── App: Your application (like K8s Deployment)                 │
│ ├── Revision: Immutable snapshot (like K8s ReplicaSet)         │
│ └── Replica: Running container instance (like K8s Pod)         │
│                                                                       │
│ Comparison:                                                          │
│ ┌────────────────┬──────────────┬──────────────┬──────────────┐  │
│ │ Feature         │ Container Apps│ AKS           │ ACI          │  │
│ ├────────────────┼──────────────┼──────────────┼──────────────┤  │
│ │ Complexity      │ Low ✅       │ High          │ Lowest       │  │
│ │ Scale to zero   │ Yes ✅       │ With KEDA    │ N/A          │  │
│ │ Auto-scale      │ KEDA built-in│ Manual setup │ None         │  │
│ │ Service mesh    │ Dapr built-in│ Istio (manual)│ None        │  │
│ │ Traffic split   │ Built-in ✅  │ Ingress config│ None        │  │
│ │ Custom domain   │ Yes ✅       │ Manual        │ None        │  │
│ │ Managed TLS     │ Yes ✅       │ cert-manager  │ None        │  │
│ │ Cron/Jobs       │ Built-in ✅  │ CronJobs     │ None        │  │
│ │ GPU             │ No ❌        │ Yes           │ Yes          │  │
│ │ Windows         │ Yes          │ Yes           │ Yes          │  │
│ │ K8s access      │ No ❌        │ Full kubectl  │ No          │  │
│ │ Sidecar         │ Dapr/custom  │ Any           │ Container grp│  │
│ │ Pricing         │ Per use ✅   │ Per VM        │ Per second   │  │
│ │ Best for        │ Microservices│ Complex K8s   │ Tasks        │  │
│ └────────────────┴──────────────┴──────────────┴──────────────┘  │
│                                                                       │
│ Pricing:                                                             │
│ ├── Consumption plan (default, serverless):                      │
│ │   vCPU: $0.000024/second (active)                            │
│ │   Memory: $0.000003/GiB-second (active)                     │
│ │   Requests: $0.40/million                                    │
│ │   Idle: Free when scaled to zero!                           │
│ │   Free grants: 180K vCPU-sec + 360K GiB-sec/month           │
│ ├── Dedicated plan (workload profiles):                         │
│ │   Pre-defined VM sizes, predictable pricing                 │
│ │   D4 (4 vCPU, 16 GiB), D8, D16, D32, E4, E8, E16         │
│ │   Pay per node allocated (like AKS node pool)              │
│ │   Best for: Sustained workloads, GPU (preview)             │
│ └── ⚡ Consumption is cheapest for variable/burst workloads     │
│                                                                       │
│ ⚡ AWS equivalent: App Runner (simpler) / ECS Fargate (closer)   │
│ ⚡ GCP equivalent: Cloud Run (very similar!) ✅                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Container App (Full Portal Walkthrough)

```
Console → Container Apps → Create

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 1: BASICS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-containers ▼]                                  │
│                                                                       │
│ Container app name: [web-app]                                       │
│ ⚡ Unique within environment. Used in internal DNS.                 │
│                                                                       │
│ Deployment source:                                                   │
│ ● Container image                                                  │
│ ○ GitHub repository (auto-build with GitHub Actions)              │
│                                                                       │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ ── Container Apps Environment ──                                   │
│                                                                       │
│ ● Create new                                                        │
│ ○ Use existing                                                      │
│                                                                       │
│ Environment name: [env-prod]                                        │
│ Environment type:                                                    │
│ ● Consumption only (serverless, scale to zero) ✅                  │
│ ○ Consumption + Dedicated (workload profiles available)          │
│                                                                       │
│ Zone redundancy: ☑ Enabled                                        │
│ ⚡ Replicas spread across availability zones for HA.               │
│                                                                       │
│ ⚡ Environment is the isolation boundary:                            │
│   ├── All apps in same env share a VNet                          │
│   ├── Apps in same env can discover each other by name          │
│   ├── Apps in same env share Log Analytics workspace            │
│   ├── Dapr components are per-environment                       │
│   └── Create separate envs for: prod vs staging vs dev          │
│                                                                       │
│ [Next: Container >]                                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 2: CONTAINER                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ☐ Use quickstart image                                              │
│                                                                       │
│ Container name: [web-app]                                           │
│                                                                       │
│ Image source:                                                        │
│ ● Azure Container Registry                                        │
│ ○ Docker Hub or other registries                                  │
│                                                                       │
│ Registry: [acrmyregistry.azurecr.io ▼]                            │
│ Image: [web-app ▼]                                                 │
│ Image tag: [v1.2 ▼]                                               │
│ ⚡ Full: acrmyregistry.azurecr.io/web-app:v1.2                    │
│                                                                       │
│ ── Resources ──                                                     │
│ CPU cores: [0.5 ▼]                                                 │
│ Memory (Gi): [1 ▼]                                                 │
│                                                                       │
│ ┌────────────────┬───────────────────────────────────────────────┐│
│ │ vCPU           │ Memory options                                ││
│ ├────────────────┼───────────────────────────────────────────────┤│
│ │ 0.25           │ 0.5 GiB                                      ││
│ │ 0.5            │ 1 GiB ✅                                      ││
│ │ 0.75           │ 1.5 GiB                                      ││
│ │ 1.0            │ 2 GiB                                        ││
│ │ 1.25           │ 2.5 GiB                                      ││
│ │ 1.5            │ 3 GiB                                        ││
│ │ 1.75           │ 3.5 GiB                                      ││
│ │ 2.0            │ 4 GiB                                        ││
│ │ 4.0            │ 8 GiB (Dedicated plan only)                  ││
│ └────────────────┴───────────────────────────────────────────────┘│
│ ⚡ Consumption: max 4 vCPU / 8 GiB per container.                  │
│   Dedicated profiles: Up to 32 vCPU / 128 GiB.                  │
│                                                                       │
│ ── Environment variables ──                                        │
│ [+ Add]                                                              │
│ ┌────────────────────────┬──────────────────┬──────────────────┐ │
│ │ Name                    │ Source            │ Value             │ │
│ ├────────────────────────┼──────────────────┼──────────────────┤ │
│ │ NODE_ENV               │ Manual entry     │ production        │ │
│ │ DB_HOST                │ Manual entry     │ db.company.com    │ │
│ │ DB_PASSWORD            │ Secret reference │ db-password ✅    │ │
│ │ API_KEY                │ Secret reference │ api-key           │ │
│ └────────────────────────┴──────────────────┴──────────────────┘ │
│ ⚡ Secrets stored in Container Apps (encrypted at rest).           │
│   Or reference Key Vault secrets via managed identity.          │
│                                                                       │
│ ── Command override ──                                              │
│ Command: [npm, start]  (array format)                             │
│ Args: [--port, 8080]                                               │
│                                                                       │
│ ── Health probes ──                                                │
│ [+ Add]                                                              │
│ Type: ● Liveness  ○ Readiness  ○ Startup                        │
│ Transport: HTTP                                                      │
│ Path: /health                                                        │
│ Port: 8080                                                           │
│ Initial delay: 10 sec                                               │
│ Period: 10 sec                                                       │
│ Failure threshold: 3                                                │
│                                                                       │
│ [Next: Ingress >]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 3: INGRESS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ☑ Enabled                                                           │
│                                                                       │
│ Ingress traffic:                                                     │
│ ● Accepting traffic from anywhere (public internet) ✅             │
│ ○ Limited to Container Apps Environment (internal only)          │
│ ○ Limited to VNet (internal + VNet peering)                      │
│                                                                       │
│ Ingress type:                                                        │
│ ● HTTP (Layer 7 — path routing, host headers) ✅                  │
│ ○ TCP (Layer 4 — raw TCP connections)                            │
│                                                                       │
│ Transport: ● Auto  ○ HTTP/1  ○ HTTP/2                            │
│                                                                       │
│ ☑ Insecure connections (Allow HTTP without redirect to HTTPS)    │
│ ⚡ Uncheck for production: Force HTTPS only.                        │
│                                                                       │
│ Target port: [8080]                                                 │
│ ⚡ Port your container listens on. Ingress routes 443→8080.        │
│                                                                       │
│ Session affinity: ☐ (sticky sessions)                              │
│ ⚡ Same client → same replica. For stateful apps (not recommended).│
│                                                                       │
│ ── IP restrictions ──                                              │
│ [+ Add]                                                              │
│ ┌────────────────────────┬──────────────────┬──────────────────┐ │
│ │ Name                    │ IP / CIDR         │ Action           │ │
│ ├────────────────────────┼──────────────────┼──────────────────┤ │
│ │ Office                  │ 203.0.113.0/24   │ Allow            │ │
│ │ Block all              │ 0.0.0.0/0        │ Deny             │ │
│ └────────────────────────┴──────────────────┴──────────────────┘ │
│                                                                       │
│ Result URL: https://web-app.blueocean-12345.centralindia.          │
│             azurecontainerapps.io                                   │
│ ⚡ Auto-generated HTTPS URL. Custom domain can be added later.     │
│                                                                       │
│ [Next: Scaling >]                                                   │
│ (Scaling rules are set after creation via "Scale" blade)          │
│                                                                       │
│ [Review + create] → [Create]                                      │
│ ⚡ Deploys in 30-60 seconds!                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Scaling Rules

```
┌─────────────────────────────────────────────────────────────────────┐
│           SCALING (KEDA-powered)                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Container App → Scale → Edit                                       │
│                                                                       │
│ Min replicas: [0]  ← Scale to zero! (no cost at idle) ✅         │
│ Max replicas: [10] ← Cap maximum instances                       │
│                                                                       │
│ ⚡ Min=0: App scales to zero when no traffic/events.               │
│   First request: Cold start (~1-3 seconds).                     │
│   Min=1: Always warm, no cold start, but pay 24/7.             │
│                                                                       │
│ Scale rules:                                                        │
│ [+ Add]                                                              │
│                                                                       │
│ ── HTTP scaling (default for web apps) ──                         │
│ Name: [http-rule]                                                   │
│ Type: HTTP scaling                                                  │
│ Concurrent requests: [100]                                          │
│ ⚡ Scale based on concurrent HTTP requests per replica.              │
│   > 100 concurrent requests → add replica.                      │
│   < 100 (per replica avg) → remove replica.                    │
│                                                                       │
│ ── Custom (KEDA) ──                                                │
│ Name: [queue-rule]                                                  │
│ Type: Custom                                                        │
│ KEDA scaler type:                                                    │
│ ┌───────────────────────┬──────────────────────────────────────┐ │
│ │ Scaler                 │ Scale on                              │ │
│ ├───────────────────────┼──────────────────────────────────────┤ │
│ │ azure-servicebus       │ Queue/topic message count           │ │
│ │ azure-queue            │ Storage Queue message count         │ │
│ │ azure-eventhub         │ Event Hub unprocessed events        │ │
│ │ cron                   │ Time schedule (scale up at time X)  │ │
│ │ tcp                    │ Active TCP connections               │ │
│ │ prometheus             │ Prometheus metric value             │ │
│ │ rabbitmq               │ Queue depth                          │ │
│ │ kafka                  │ Consumer lag                         │ │
│ │ postgresql             │ Query result                         │ │
│ └───────────────────────┴──────────────────────────────────────┘ │
│                                                                       │
│ Example — Azure Service Bus:                                       │
│ Metadata:                                                            │
│   queueName: orders                                                 │
│   messageCount: 5    ← 1 replica per 5 messages                 │
│   connectionFromEnv: SB_CONNECTION                                 │
│                                                                       │
│ Scaling flow:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Queue: 0 messages → 0 replicas (scale to zero!)            │  │
│ │ Queue: 5 messages → 1 replica                               │  │
│ │ Queue: 50 messages → 10 replicas (max)                     │  │
│ │ Queue: 0 messages → wait cooldown → 0 replicas            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Example — Cron scale:                                              │
│ Scale to 5 replicas during business hours:                        │
│ Metadata:                                                            │
│   timezone: Asia/Kolkata                                           │
│   start: 0 9 * * *   (9 AM)                                      │
│   end: 0 18 * * *    (6 PM)                                      │
│   desiredReplicas: 5                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Revisions & Traffic Splitting

```
┌─────────────────────────────────────────────────────────────────────┐
│           REVISIONS & TRAFFIC SPLITTING                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Every deployment creates a new REVISION:                            │
│ ├── Revision = immutable snapshot (image + config + secrets)    │
│ ├── Name: web-app--abc123 (auto-generated suffix)              │
│ ├── Multiple revisions can run simultaneously                   │
│ └── Traffic can be split between revisions                      │
│                                                                       │
│ Revision modes:                                                      │
│ ● Single revision (default): New revision gets 100%, old disabled│
│ ○ Multiple revisions: Keep old revisions active for traffic split│
│                                                                       │
│ Traffic splitting (multiple revision mode):                        │
│ Container App → Revisions → Traffic                               │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Revision                   │ Traffic  │ Status                │  │
│ ├───────────────────────────┼──────────┼───────────────────────┤  │
│ │ web-app--abc123 (stable)  │ 90%      │ Active                │  │
│ │ web-app--def456 (canary)  │ 10%      │ Active                │  │
│ └───────────────────────────┴──────────┴───────────────────────┘  │
│                                                                       │
│ Use cases:                                                           │
│ ├── Canary: 5% → 25% → 50% → 100% (gradual rollout)         │
│ ├── A/B testing: 50/50 between feature versions                │
│ ├── Blue-green: 100% to new, instant rollback to old          │
│ └── Testing: Direct URL to specific revision                   │
│                                                                       │
│ Direct revision URL (for testing):                                 │
│ https://web-app--def456.blueocean-12345.centralindia.            │
│   azurecontainerapps.io                                           │
│ ⚡ Access specific revision directly (bypasses traffic split).     │
│                                                                       │
│ Labels (stable traffic distribution):                              │
│ ├── Label "latest" → always points to newest revision           │
│ ├── Label "stable" → you assign manually                        │
│ └── URL: https://web-app---stable.blueocean-12345...             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Dapr Integration

```
┌─────────────────────────────────────────────────────────────────────┐
│           DAPR (Distributed Application Runtime)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Dapr provides microservice building blocks:                  │
│ ├── Service invocation (call other services by name)            │
│ ├── State management (Redis, Cosmos DB, etc.)                   │
│ ├── Pub/sub messaging (Service Bus, Event Hubs, etc.)          │
│ ├── Bindings (input/output to external systems)                 │
│ ├── Secrets (access secrets stores)                              │
│ └── Actors (virtual actors pattern)                              │
│                                                                       │
│ ⚡ Dapr runs as a sidecar — no code changes to use!                │
│   Your app calls Dapr via HTTP/gRPC → Dapr handles the rest.   │
│                                                                       │
│ Enable Dapr on Container App:                                      │
│ Container App → Dapr → Enabled: ☑                                │
│ App ID: [backend-api]                                               │
│ App port: [8080]                                                    │
│ App protocol: [HTTP ▼]  (HTTP or gRPC)                           │
│                                                                       │
│ Example — Service invocation:                                      │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Frontend calls Backend via Dapr:                             │  │
│ │                                                              │  │
│ │ // Instead of: http://backend-api:8080/users                │  │
│ │ // Call Dapr sidecar:                                        │  │
│ │ fetch("http://localhost:3500/v1.0/invoke/backend-api/       │  │
│ │   method/users")                                             │  │
│ │                                                              │  │
│ │ ⚡ Dapr handles:                                              │  │
│ │   ├── Service discovery (find backend-api)                 │  │
│ │   ├── Retries (automatic on failure)                       │  │
│ │   ├── mTLS (encrypted communication)                       │  │
│ │   └── Observability (traces, metrics)                      │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Example — Pub/Sub:                                                  │
│ Dapr component (Environment → Dapr components):                   │
│ componentType: pubsub.azure.servicebus.topics                    │
│ metadata:                                                            │
│   - name: connectionString                                        │
│     secretRef: sb-connection                                      │
│                                                                       │
│ # Publish (POST to Dapr sidecar):                                 │
│ POST http://localhost:3500/v1.0/publish/pubsub/orders             │
│ Body: {"orderId": "123", "amount": 99.99}                        │
│                                                                       │
│ # Subscribe (Dapr calls your app):                                │
│ // Your app exposes POST /orders endpoint                        │
│ // Dapr delivers messages there automatically                   │
│                                                                       │
│ Example — State store:                                              │
│ componentType: state.azure.cosmosdb                               │
│ # Save state:                                                       │
│ POST http://localhost:3500/v1.0/state/statestore                 │
│ Body: [{"key": "user-123", "value": {"name": "Ritesh"}}]       │
│ # Get state:                                                        │
│ GET http://localhost:3500/v1.0/state/statestore/user-123         │
│                                                                       │
│ ⚡ Dapr abstracts the infrastructure:                                │
│   Switch from Redis → Cosmos DB by changing component config!   │
│   Zero code changes in your application.                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Container Apps Jobs

```
┌─────────────────────────────────────────────────────────────────────┐
│           JOBS                                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Jobs = Containers that run to completion (not long-running).       │
│                                                                       │
│ Job types:                                                           │
│ ├── Scheduled: Cron schedule (like Azure Functions timer)        │
│ ├── Event-driven: Triggered by KEDA scaler (queue messages)    │
│ └── Manual: Triggered by API/CLI call                            │
│                                                                       │
│ Console → Container Apps → Jobs → Create                          │
│                                                                       │
│ Trigger type: [Schedule ▼]                                        │
│ Cron expression: [0 2 * * *]  ← Daily at 2 AM                  │
│                                                                       │
│ Or: [Event-driven ▼]                                               │
│ Scaling rule: azure-servicebus (triggered by queue messages)    │
│ Polling interval: 30 seconds                                       │
│ Min executions: 0                                                    │
│ Max executions: 10                                                   │
│                                                                       │
│ Replica timeout: [1800] seconds (30 min max per execution)       │
│ Replica retry limit: [3]                                           │
│ Parallelism: [1]  (run 1 at a time, or parallel)                 │
│                                                                       │
│ ⚡ Jobs vs Apps:                                                      │
│   Apps: Long-running, respond to requests, scale on traffic     │
│   Jobs: Run to completion, exit, scale on events/schedule       │
│                                                                       │
│ Use cases:                                                           │
│ ├── Nightly ETL / data processing                                │
│ ├── Database migrations                                           │
│ ├── Report generation                                             │
│ ├── Queue processing (event-driven)                              │
│ ├── CI/CD build tasks                                             │
│ └── ML training/batch inference                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Networking & Custom Domains

```
┌─────────────────────────────────────────────────────────────────────┐
│           NETWORKING                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ VNet integration (Environment level):                               │
│ ├── Default: No VNet (isolated, public access only)              │
│ ├── Custom VNet: Environment deployed in your VNet subnet       │
│ │   ├── Apps can reach private resources (SQL, Redis, VMs)    │
│ │   ├── Subnet must be /23 or larger (dedicated to Container Apps)│
│ │   └── Subnet delegated to Microsoft.App/environments        │
│ └── Internal only: No public access, VNet-only                  │
│                                                                       │
│ Architecture (VNet integrated, internal + external):               │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ VNet: vnet-prod (10.0.0.0/16)                               │  │
│ │ ├── subnet-apps (10.0.0.0/23) → Container Apps env         │  │
│ │ ├── subnet-db (10.0.4.0/24) → SQL, Redis                  │  │
│ │ └── subnet-other (10.0.8.0/24) → VMs, App Service         │  │
│ │                                                              │  │
│ │ Container Apps:                                              │  │
│ │ ├── web-app (external ingress → public access)             │  │
│ │ ├── api (external ingress → public access)                 │  │
│ │ ├── worker (no ingress — internal only)                    │  │
│ │ └── admin (internal ingress → VNet only)                   │  │
│ │                                                              │  │
│ │ worker → SQL (10.0.4.5) ✅ (same VNet, routable)           │  │
│ │ api → Redis (10.0.4.10) ✅                                  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
├─────────────────────────────────────────────────────────────────────┤
│           CUSTOM DOMAINS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Container App → Custom domains → Add                              │
│                                                                       │
│ Domain: [app.example.com]                                           │
│                                                                       │
│ Certificate:                                                         │
│ ● Managed certificate (free, auto-renewed by Azure) ✅            │
│ ○ Bring your own certificate (upload .pfx)                       │
│                                                                       │
│ DNS validation:                                                      │
│ ┌────────┬──────────────────────────┬───────────────────────────┐│
│ │ Type    │ Name                      │ Value                     ││
│ ├────────┼──────────────────────────┼───────────────────────────┤│
│ │ TXT     │ asuid.app.example.com     │ (verification ID)        ││
│ │ CNAME   │ app.example.com           │ web-app.blueocean-12345 ││
│ │         │                           │ .centralindia.azure...  ││
│ └────────┴──────────────────────────┴───────────────────────────┘│
│                                                                       │
│ ⚡ Managed cert: Free SSL, auto-renewed, takes 5-15 minutes.      │
│   After validation: https://app.example.com → your container app.│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform & Bicep

### Terraform

```hcl
# Container Apps Environment
resource "azurerm_container_app_environment" "prod" {
  name                       = "env-prod"
  location                   = azurerm_resource_group.main.location
  resource_group_name        = azurerm_resource_group.main.name
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  
  infrastructure_subnet_id   = azurerm_subnet.container_apps.id
  zone_redundancy_enabled    = true

  tags = {
    environment = "prod"
  }
}

# Container App — Web frontend
resource "azurerm_container_app" "web" {
  name                         = "web-app"
  container_app_environment_id = azurerm_container_app_environment.prod.id
  resource_group_name          = azurerm_resource_group.main.name
  revision_mode                = "Multiple"

  template {
    min_replicas = 1
    max_replicas = 10

    container {
      name   = "web-app"
      image  = "acrmyregistry.azurecr.io/web-app:v1.2"
      cpu    = 0.5
      memory = "1Gi"

      env {
        name  = "NODE_ENV"
        value = "production"
      }

      env {
        name        = "DB_PASSWORD"
        secret_name = "db-password"
      }

      liveness_probe {
        transport = "HTTP"
        path      = "/health"
        port      = 8080
      }

      readiness_probe {
        transport = "HTTP"
        path      = "/ready"
        port      = 8080
      }
    }

    http_scale_rule {
      name                = "http-scaling"
      concurrent_requests = "100"
    }
  }

  ingress {
    external_enabled = true
    target_port      = 8080
    transport        = "auto"

    traffic_weight {
      latest_revision = true
      percentage      = 100
    }
  }

  secret {
    name  = "db-password"
    value = var.db_password
  }

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.app.id]
  }

  registry {
    server   = azurerm_container_registry.main.login_server
    identity = azurerm_user_assigned_identity.app.id
  }
}

# Container App — Worker (event-driven, scale to zero)
resource "azurerm_container_app" "worker" {
  name                         = "worker"
  container_app_environment_id = azurerm_container_app_environment.prod.id
  resource_group_name          = azurerm_resource_group.main.name
  revision_mode                = "Single"

  template {
    min_replicas = 0
    max_replicas = 20

    container {
      name   = "worker"
      image  = "acrmyregistry.azurecr.io/worker:v1"
      cpu    = 1.0
      memory = "2Gi"

      env {
        name        = "SB_CONNECTION"
        secret_name = "sb-connection"
      }
    }

    custom_scale_rule {
      name             = "queue-scaling"
      custom_rule_type = "azure-servicebus"
      metadata = {
        queueName    = "orders"
        messageCount = "5"
      }
      authentication {
        secret_name       = "sb-connection"
        trigger_parameter = "connection"
      }
    }
  }

  secret {
    name  = "sb-connection"
    value = var.servicebus_connection
  }
}

# Container App — API with Dapr
resource "azurerm_container_app" "api" {
  name                         = "api"
  container_app_environment_id = azurerm_container_app_environment.prod.id
  resource_group_name          = azurerm_resource_group.main.name
  revision_mode                = "Multiple"

  template {
    min_replicas = 2
    max_replicas = 15

    container {
      name   = "api"
      image  = "acrmyregistry.azurecr.io/api:v2"
      cpu    = 1.0
      memory = "2Gi"
    }
  }

  dapr {
    app_id       = "api"
    app_port     = 8080
    app_protocol = "http"
  }

  ingress {
    external_enabled = true
    target_port      = 8080
  }
}

# Dapr Component — Pub/Sub (Service Bus)
resource "azurerm_container_app_environment_dapr_component" "pubsub" {
  name                         = "pubsub"
  container_app_environment_id = azurerm_container_app_environment.prod.id
  component_type               = "pubsub.azure.servicebus.topics"
  version                      = "v1"

  metadata {
    name  = "connectionString"
    secret_name = "sb-connection"
  }

  secret {
    name  = "sb-connection"
    value = var.servicebus_connection
  }

  scopes = ["api", "worker"]
}

# Container App Job — Scheduled
resource "azurerm_container_app_job" "etl" {
  name                         = "nightly-etl"
  container_app_environment_id = azurerm_container_app_environment.prod.id
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location

  replica_timeout_in_seconds = 1800
  replica_retry_limit        = 2

  schedule_trigger_config {
    cron_expression          = "0 2 * * *"
    parallelism              = 1
    replica_completion_count = 1
  }

  template {
    container {
      name   = "etl"
      image  = "acrmyregistry.azurecr.io/etl:v1"
      cpu    = 2.0
      memory = "4Gi"

      command = ["python", "run_etl.py"]
    }
  }
}

# Subnet for Container Apps (must be /23 or larger)
resource "azurerm_subnet" "container_apps" {
  name                 = "subnet-container-apps"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.0.0/23"]

  delegation {
    name = "container-apps-delegation"
    service_delegation {
      name = "Microsoft.App/environments"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/join/action",
      ]
    }
  }
}
```

### Bicep

```bicep
resource environment 'Microsoft.App/managedEnvironments@2024-03-01' = {
  name: 'env-prod'
  location: resourceGroup().location
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalytics.properties.customerId
        sharedKey: logAnalytics.listKeys().primarySharedKey
      }
    }
    vnetConfiguration: {
      infrastructureSubnetId: subnet.id
    }
    zoneRedundant: true
  }
}

resource webApp 'Microsoft.App/containerApps@2024-03-01' = {
  name: 'web-app'
  location: resourceGroup().location
  properties: {
    managedEnvironmentId: environment.id
    configuration: {
      ingress: {
        external: true
        targetPort: 8080
        transport: 'auto'
        traffic: [{ latestRevision: true, weight: 100 }]
      }
      secrets: [{ name: 'db-password', value: dbPassword }]
      registries: [{
        server: acr.properties.loginServer
        identity: managedIdentity.id
      }]
    }
    template: {
      containers: [{
        name: 'web-app'
        image: '${acr.properties.loginServer}/web-app:v1.2'
        resources: { cpu: json('0.5'), memory: '1Gi' }
        env: [
          { name: 'NODE_ENV', value: 'production' }
          { name: 'DB_PASSWORD', secretRef: 'db-password' }
        ]
      }]
      scale: {
        minReplicas: 1
        maxReplicas: 10
        rules: [{ name: 'http', http: { metadata: { concurrentRequests: '100' } } }]
      }
    }
  }
}
```

---

## Part 9: az CLI Reference

```bash
# ═══ Environment ═══

# Create environment (consumption)
az containerapp env create \
  --name env-prod \
  --resource-group rg-containers \
  --location centralindia \
  --logs-workspace-id <LOG_ANALYTICS_WORKSPACE_ID>

# Create environment (with VNet)
az containerapp env create \
  --name env-prod \
  --resource-group rg-containers \
  --location centralindia \
  --infrastructure-subnet-resource-id /subscriptions/.../subnets/subnet-ca \
  --internal-only false \
  --zone-redundant

# ═══ Container App ═══

# Create container app
az containerapp create \
  --name web-app \
  --resource-group rg-containers \
  --environment env-prod \
  --image acrmyregistry.azurecr.io/web-app:v1.2 \
  --registry-server acrmyregistry.azurecr.io \
  --registry-identity <MANAGED_IDENTITY_ID> \
  --target-port 8080 \
  --ingress external \
  --cpu 0.5 \
  --memory 1.0Gi \
  --min-replicas 1 \
  --max-replicas 10 \
  --env-vars "NODE_ENV=production" "DB_PASSWORD=secretref:db-password" \
  --secrets "db-password=MySecretPassword123"

# Update container (new image)
az containerapp update \
  --name web-app \
  --resource-group rg-containers \
  --image acrmyregistry.azurecr.io/web-app:v1.3

# Set scale rules
az containerapp update \
  --name web-app \
  --resource-group rg-containers \
  --scale-rule-name http-rule \
  --scale-rule-type http \
  --scale-rule-http-concurrency 100 \
  --min-replicas 1 \
  --max-replicas 10

# ═══ Revisions & Traffic ═══

# List revisions
az containerapp revision list \
  --name web-app \
  --resource-group rg-containers \
  --output table

# Enable multiple revision mode
az containerapp revision set-mode \
  --name web-app \
  --resource-group rg-containers \
  --mode multiple

# Set traffic split
az containerapp ingress traffic set \
  --name web-app \
  --resource-group rg-containers \
  --revision-weight web-app--abc123=90 web-app--def456=10

# Activate old revision (for rollback)
az containerapp revision activate \
  --name web-app \
  --resource-group rg-containers \
  --revision web-app--abc123

# ═══ Secrets ═══

# Add secret
az containerapp secret set \
  --name web-app \
  --resource-group rg-containers \
  --secrets "api-key=sk-xxxx"

# List secrets
az containerapp secret list \
  --name web-app \
  --resource-group rg-containers

# ═══ Custom domain ═══

# Add custom domain with managed certificate
az containerapp hostname add \
  --name web-app \
  --resource-group rg-containers \
  --hostname app.example.com

az containerapp ssl upload \
  --name web-app \
  --resource-group rg-containers \
  --hostname app.example.com \
  --certificate-type ManagedCertificate

# ═══ Dapr ═══

# Enable Dapr
az containerapp dapr enable \
  --name api \
  --resource-group rg-containers \
  --dapr-app-id api \
  --dapr-app-port 8080 \
  --dapr-app-protocol http

# ═══ Jobs ═══

# Create scheduled job
az containerapp job create \
  --name nightly-etl \
  --resource-group rg-containers \
  --environment env-prod \
  --image acrmyregistry.azurecr.io/etl:v1 \
  --trigger-type Schedule \
  --cron-expression "0 2 * * *" \
  --cpu 2 \
  --memory 4Gi \
  --replica-timeout 1800 \
  --replica-retry-limit 2

# Trigger manual job execution
az containerapp job start \
  --name nightly-etl \
  --resource-group rg-containers

# ═══ Logs ═══

# View app logs
az containerapp logs show \
  --name web-app \
  --resource-group rg-containers \
  --follow

# ═══ Delete ═══
az containerapp delete --name web-app --resource-group rg-containers --yes
az containerapp env delete --name env-prod --resource-group rg-containers --yes
```

---

## Part 10: Real-World Patterns

### Startup

```
Simple microservices on Container Apps:

Environment: env-prod (Consumption, zone-redundant)

Apps:
├── web-app (Next.js frontend)
│   Image: acr/web-app:v1
│   CPU: 0.5, Memory: 1 GiB
│   Scale: 0-5 (HTTP, 100 concurrent)
│   Ingress: External
│   Custom domain: app.startup.com (managed cert)
│   Cost: ~$20-50/month
│
├── api (Node.js backend)
│   Image: acr/api:v2
│   CPU: 1, Memory: 2 GiB
│   Scale: 1-8 (HTTP, 50 concurrent)
│   Ingress: External
│   Custom domain: api.startup.com
│   Env: DB_HOST, secrets from Container Apps secrets
│   Cost: ~$40-100/month
│
└── worker (queue consumer)
    Image: acr/worker:v1
    CPU: 0.5, Memory: 1 GiB
    Scale: 0-5 (Azure Service Bus, 5 messages/replica)
    Ingress: None (no HTTP)
    Cost: ~$5-30/month (often at zero!)

Service discovery: api calls "http://worker" internally
Secrets: Stored in Container Apps (or Key Vault ref)
CI/CD: GitHub Actions → Build → Push ACR → az containerapp update

Total: ~$65-180/month (with scale to zero savings)
```

### Mid-Size

```
Microservices with Dapr:

Environment: env-prod (Consumption + Dedicated, VNet integrated)

Apps:
├── frontend (React SSR, 1-10, external)
├── api-gateway (Node.js, 2-15, external, Dapr enabled)
├── order-service (Python, 2-10, internal, Dapr)
├── payment-service (Node.js, 2-8, internal, Dapr)
├── notification-service (Python, 0-5, internal, Dapr)
├── file-service (Go, 1-5, internal)
└── admin (React SSR, 0-3, internal VNet only)

Workers:
├── email-worker (0-10, Azure Queue scaler, Dapr)
├── report-worker (0-5, Service Bus scaler)
└── sync-worker (0-3, Event Hub scaler)

Jobs:
├── nightly-etl (cron: 0 2 * * *, 2 CPU, 4 GiB)
├── weekly-report (cron: 0 9 * * MON)
└── data-migration (manual trigger)

Dapr components:
├── pubsub: Azure Service Bus (order events)
├── state: Azure Cosmos DB (session state)
├── secrets: Azure Key Vault
└── binding-output: SendGrid (email)

Traffic splitting:
├── api-gateway: 90% stable / 10% canary (per deploy)
├── Promote after 30 min if error rate < 0.1%
└── Rollback: Shift 100% back to stable revision

Networking:
├── VNet integrated (access private SQL, Redis)
├── External: frontend, api-gateway (public)
├── Internal: All services (VNet only, no public)
├── admin: Internal only (access via VPN)
└── NSG: Restrict Container Apps subnet → DB subnet

Observability:
├── Container Insights (Log Analytics)
├── Dapr distributed tracing → Application Insights
├── Alerts: High error rate, scale limits, cold starts
└── Dashboards: Per-app metrics

CI/CD:
├── GitHub Actions: PR → Build → Push ACR → Deploy to staging
├── Main merge → Deploy to prod (traffic split 10%)
├── Manual promote → 100% traffic to new revision
└── Rollback: az containerapp ingress traffic set

Cost: $400-1,500/month
```

### Enterprise

```
Large-scale Container Apps platform:

Environments:
├── env-prod (Dedicated workload profiles, VNet, zone-redundant)
│   Workload profiles: D8 (8 vCPU, 32 GiB) × 5 nodes
├── env-staging (Consumption, same VNet, smaller)
└── env-dev (Consumption, separate VNet)

Apps (30+ container apps):
├── Customer-facing (external):
│   ├── web-app (2-20 replicas)
│   ├── mobile-bff (2-15 replicas)
│   └── api-gateway (3-20 replicas)
├── Internal services (internal):
│   ├── 12 microservices (Dapr enabled)
│   ├── Each: 2-10 replicas, auto-scale
│   └── Service-to-service via Dapr invocation
├── Event workers (scale to zero):
│   ├── 8 workers (Service Bus, Event Hub, Queue scalers)
│   ├── Scale: 0-30 replicas based on queue depth
│   └── Cost: Near-zero when idle
└── Admin/internal tools (internal, VNet only)

Jobs:
├── 5 scheduled jobs (ETL, reports, cleanup)
├── 3 event-driven jobs (file processing)
└── On-demand jobs (migrations, one-off tasks)

Dapr:
├── Pub/Sub: Service Bus Topics (event-driven architecture)
├── State: Cosmos DB (distributed state)
├── Secrets: Key Vault (all secrets centralized)
├── Bindings: Blob Storage, SendGrid, Twilio
└── Actors: Virtual actors for workflow state

Security:
├── VNet integrated (all apps in private subnet)
├── Private endpoints: ACR, Key Vault, SQL, Storage
├── Managed identity: All apps use User-Assigned MI
├── No secrets in config (all from Key Vault)
├── IP restrictions on external ingress
├── Azure Front Door: WAF + DDoS in front
├── Azure Policy: Enforce container app settings
├── Defender for Cloud: Container vulnerability scanning
└── Audit: All operations → Activity Log → SIEM

Multi-region:
├── env-prod-india (Central India)
├── env-prod-eu (West Europe)
├── Azure Front Door: Global routing (latency-based)
├── Each region: Independent Container Apps environment
├── Cosmos DB: Multi-region write (shared state)
└── Service Bus: Geo-DR (message replication)

CI/CD:
├── Azure DevOps Pipelines (or GitHub Actions)
├── Build → Scan (Trivy) → Push ACR → Deploy staging
├── Integration tests in staging → Promote to prod
├── Traffic split: 5% → 25% → 50% → 100% (automated)
├── Rollback: Instant via revision switch
└── GitOps: Infra via Terraform, apps via CI/CD

Cost: $5,000-20,000/month
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Serverless container platform (PaaS) |
| Built on | AKS + KEDA + Envoy + Dapr (abstracted) |
| Scale to zero | Yes (Consumption plan) |
| Scaling | HTTP concurrent requests, KEDA scalers, cron |
| Max resources | 4 vCPU / 8 GiB (Consumption), 32 vCPU / 128 GiB (Dedicated) |
| Revisions | Immutable snapshots, traffic splitting |
| Dapr | Built-in (service invocation, pub/sub, state, secrets) |
| Jobs | Scheduled (cron), event-driven, manual |
| Ingress | External (public), Internal (VNet), TCP/HTTP |
| Custom domain | Yes, with free managed certificates |
| VNet | Environment-level integration |
| Service discovery | By app name within environment |
| Secrets | Built-in + Key Vault references |
| Pricing | Per-second (CPU + memory) or Dedicated profiles |
| Free grants | 180K vCPU-sec + 360K GiB-sec/month |
| AWS equivalent | App Runner (simpler) / ECS Fargate (closer) |
| GCP equivalent | Cloud Run (very similar) |

---

## What's Next?

In the next chapter, we'll cover Azure Functions — serverless event-driven compute for individual functions.

→ Next: [Chapter 22: Azure Storage Accounts](22-storage-accounts.md)

---

*Last Updated: May 2026*
