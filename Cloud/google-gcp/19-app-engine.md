# Chapter 19: Google App Engine

---

## Table of Contents

- [Overview](#overview)
- [Part 1: App Engine Fundamentals](#part-1-app-engine-fundamentals)
- [Part 2: app.yaml Configuration](#part-2-appyaml-configuration)
- [Part 3: Scaling Configurations](#part-3-scaling-configurations)
- [Part 4: Versions & Traffic Splitting](#part-4-versions--traffic-splitting)
- [Part 5: Services & Dispatch Rules](#part-5-services--dispatch-rules)
- [Part 6: Custom Domains, SSL & Firewall](#part-6-custom-domains-ssl--firewall)
- [Part 7: Cron Jobs & Task Queues](#part-7-cron-jobs--task-queues)
- [Part 8: Terraform & gcloud CLI](#part-8-terraform--gcloud-cli)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Console Walkthrough: Deploying & Managing App Engine](#console-walkthrough-deploying--managing-app-engine)
- [App Engine Pricing](#app-engine-pricing)
- [Limitations & Gotchas](#limitations--gotchas)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google App Engine (GAE) is Google Cloud's original PaaS — the OG of serverless. Launched in 2008, it lets you deploy web apps and APIs without managing any infrastructure. App Engine comes in two environments: **Standard** (sandbox, fast scale to zero) and **Flexible** (custom runtimes, Compute Engine VMs). It's ideal when you want a platform that just runs your code.

```
What you'll learn:
├── App Engine Fundamentals
│   ├── What & why (original PaaS, fully managed)
│   ├── Standard vs Flexible environment
│   └── App Engine vs Cloud Run vs GKE vs Cloud Functions
├── App Engine Architecture
│   ├── Application → Services → Versions → Instances
│   ├── Traffic splitting
│   └── Dispatch rules
├── Creating an App (Full Console + app.yaml Walkthrough)
│   ├── Standard environment config
│   ├── Flexible environment config
│   └── All fields explained
├── Scaling Configuration
│   ├── Automatic scaling (default)
│   ├── Basic scaling
│   └── Manual scaling
├── Services & Microservices
├── Versions & Traffic Splitting
├── Custom Domains & SSL
├── Firewall Rules
├── Cron Jobs
├── Task Queues (Cloud Tasks integration)
├── Terraform & gcloud CLI
└── Real-world patterns
```

---

## Part 1: App Engine Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           APP ENGINE CONCEPT                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is App Engine?                                                 │
│ ├── Google's original PaaS (2008 — predates everything!)        │
│ ├── Deploy web apps by uploading code (no containers needed)    │
│ ├── Google manages ALL infrastructure                            │
│ ├── Built-in versioning and traffic splitting                   │
│ ├── Scale to zero (Standard env) or scale up to thousands      │
│ └── One App Engine app per GCP project (can have many services) │
│                                                                       │
│ Architecture:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ GCP Project: my-project                                      │  │
│ │                                                              │  │
│ │ App Engine Application (1 per project, 1 region — permanent)│  │
│ │ ├── Service: default (every app has this)                   │  │
│ │ │   ├── Version: v1 (serving 90% traffic)                  │  │
│ │ │   │   └── Instances: 3 (auto-scaled)                    │  │
│ │ │   └── Version: v2 (serving 10% traffic — canary!)       │  │
│ │ │       └── Instances: 1                                   │  │
│ │ │                                                           │  │
│ │ ├── Service: api                                            │  │
│ │ │   └── Version: v5 (serving 100%)                         │  │
│ │ │       └── Instances: 5                                   │  │
│ │ │                                                           │  │
│ │ └── Service: worker (background tasks)                     │  │
│ │     └── Version: v3 (serving 100%)                         │  │
│ │         └── Instances: 2                                   │  │
│ │                                                              │  │
│ │ URLs:                                                        │  │
│ │ https://my-project.appspot.com → default service           │  │
│ │ https://api-dot-my-project.appspot.com → api service       │  │
│ │ https://v2-dot-my-project.appspot.com → v2 of default     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚠️ IMPORTANT:                                                       │
│   App Engine region is PERMANENT once set. Cannot change!        │
│   Choose wisely: asia-south1 (Mumbai), us-central1, etc.       │
│   Only ONE App Engine app per GCP project.                      │
│                                                                       │
│ Standard vs Flexible:                                                │
│ ┌────────────────┬───────────────────┬───────────────────────┐    │
│ │ Feature         │ Standard           │ Flexible               │    │
│ ├────────────────┼───────────────────┼───────────────────────┤    │
│ │ Runtime        │ Sandbox (Google-   │ Docker container on   │    │
│ │                │ managed runtimes)  │ Compute Engine VMs    │    │
│ │ Languages      │ Python, Java,      │ Any language/runtime │    │
│ │                │ Node.js, Go, PHP,  │ (custom Dockerfile)  │    │
│ │                │ Ruby               │                       │    │
│ │ Scale to zero  │ Yes ✅ (free!)     │ No ❌ (min 1 VM)     │    │
│ │ Startup time   │ Seconds ✅         │ Minutes (VM boot)    │    │
│ │ Pricing        │ Per instance-hour  │ Per VM-hour           │    │
│ │ Free tier      │ 28 instance-hrs/day│ None                 │    │
│ │ Max request    │ 10 min (gen1)      │ 60 min                │    │
│ │ timeout        │ 60 min (gen2)      │                       │    │
│ │ SSH to instance│ No ❌              │ Yes ✅ (it's a VM)    │    │
│ │ Background     │ Limited (gen2 OK)  │ Yes ✅                │    │
│ │ threads        │                    │                       │    │
│ │ Local disk     │ Read-only (tmpfs)  │ Yes (ephemeral disk) │    │
│ │ Custom OS pkgs │ No ❌              │ Yes ✅ (Dockerfile)   │    │
│ │ WebSocket      │ No (gen1)          │ Yes ✅                │    │
│ │                │ Yes (gen2)         │                       │    │
│ │ Network        │ No VPC access (gen1)│ VPC access ✅        │    │
│ │                │ VPC Connector (gen2)│                      │    │
│ │ Best for       │ Web apps, APIs     │ Custom runtimes,     │    │
│ │                │ (cost-effective)   │ legacy apps, Docker  │    │
│ └────────────────┴───────────────────┴───────────────────────┘    │
│                                                                       │
│ Standard gen1 vs gen2:                                              │
│ ├── gen1: Original sandbox, Python 2.7/3.7, Java 8/11, Go 1.11 │
│ │   ├── Proprietary APIs (ndb, taskqueue, memcache)            │
│ │   ├── 10 min request timeout                                  │
│ │   └── Being phased out                                        │
│ ├── gen2: Modern runtime, Python 3.9+, Node.js 18+, Java 17+  │
│ │   ├── Uses standard libraries (no proprietary APIs)          │
│ │   ├── 60 min request timeout                                  │
│ │   ├── VPC Connector support                                   │
│ │   └── Recommended for all new apps ✅                         │
│ └── Flexible: Compute Engine VMs running Docker containers      │
│                                                                       │
│ App Engine vs alternatives:                                         │
│ ┌────────────────┬──────────┬──────────┬──────────┬────────────┐ │
│ │ Feature         │ App Eng  │ Cloud Run│ GKE      │ Cloud Func │ │
│ ├────────────────┼──────────┼──────────┼──────────┼────────────┤ │
│ │ Source deploy   │ Yes ✅   │ Yes      │ No       │ Yes ✅     │ │
│ │ Container       │ Optional │ Required │ Required │ No         │ │
│ │ Scale to zero   │ Std only │ Yes ✅   │ No       │ Yes ✅     │ │
│ │ Traffic split   │ Built-in │ Built-in │ Ingress  │ No         │ │
│ │ Cron built-in   │ Yes ✅   │ Scheduler│ CronJobs │ Scheduler  │ │
│ │ Versioning      │ Built-in │ Revisions│ Deploys  │ No         │ │
│ │ WebSocket       │ gen2/Flex│ Yes      │ Yes      │ No         │ │
│ │ Lock-in risk    │ Higher   │ Lower ✅ │ Lowest   │ Medium     │ │
│ │ Recommendation  │ Legacy ⚠️│ New ✅   │ Complex  │ Events     │ │
│ └────────────────┴──────────┴──────────┴──────────┴────────────┘ │
│                                                                       │
│ ⚡ For new projects, Google recommends Cloud Run over App Engine.  │
│   App Engine is still great for existing apps & simple web apps. │
│                                                                       │
│ ⚡ AWS equivalent: Elastic Beanstalk / App Runner                  │
│ ⚡ Azure equivalent: App Service                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: app.yaml Configuration

```
┌─────────────────────────────────────────────────────────────────────┐
│           app.yaml — STANDARD ENVIRONMENT                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ # app.yaml — The deployment config for App Engine                  │
│ # Place in project root                                             │
│                                                                       │
│ runtime: python312     ← Runtime (python312, nodejs20, java21,    │
│                          go122, php83, ruby33)                    │
│                                                                       │
│ service: default       ← Service name (omit for "default")       │
│                                                                       │
│ instance_class: F2     ← Instance class                           │
│ ┌──────────┬────────┬───────────┬──────────────────────────────┐  │
│ │ Class     │ Memory │ CPU        │ Notes                        │  │
│ ├──────────┼────────┼───────────┼──────────────────────────────┤  │
│ │ F1 (def) │ 256 MB │ 600 MHz   │ Free tier, basic apps       │  │
│ │ F2       │ 512 MB │ 1.2 GHz   │ Most web apps ✅             │  │
│ │ F4       │ 1 GB   │ 2.4 GHz   │ Larger apps                 │  │
│ │ F4_1G    │ 2 GB   │ 2.4 GHz   │ Memory-intensive            │  │
│ │ B1       │ 256 MB │ 600 MHz   │ Basic/manual scaling only   │  │
│ │ B2       │ 512 MB │ 1.2 GHz   │ Basic/manual scaling only   │  │
│ │ B4       │ 1 GB   │ 2.4 GHz   │ Basic/manual scaling only   │  │
│ │ B4_1G    │ 2 GB   │ 2.4 GHz   │ Basic/manual scaling only   │  │
│ │ B8       │ 2 GB   │ 4.8 GHz   │ Basic/manual scaling only   │  │
│ └──────────┴────────┴───────────┴──────────────────────────────┘  │
│ ⚡ F = Automatic scaling, B = Basic/Manual scaling                 │
│                                                                       │
│ entrypoint: gunicorn -b :$PORT app:app                           │
│ ⚡ How to start your app. PORT env var is set by App Engine.       │
│   Node.js: node server.js                                        │
│   Python:  gunicorn -b :$PORT app:app                            │
│   Java:    (auto-detected from pom.xml/build.gradle)             │
│   Go:      (auto-detected from go.mod)                           │
│                                                                       │
│ env_variables:                                                       │
│   NODE_ENV: "production"                                           │
│   DB_HOST: "10.0.0.5"                                             │
│   REDIS_HOST: "10.0.1.10"                                        │
│ ⚡ Plain text env vars. For secrets: use Secret Manager.           │
│                                                                       │
│ automatic_scaling:                                                   │
│   min_instances: 0         ← 0 = scale to zero ✅                │
│   max_instances: 10        ← Max instances (prevent cost spike) │
│   min_idle_instances: 1    ← Always keep 1 warm (optional)      │
│   max_idle_instances: 2    ← Don't keep more than 2 idle        │
│   target_cpu_utilization: 0.65    ← Scale up at 65% CPU        │
│   target_throughput_utilization: 0.6                             │
│   max_concurrent_requests: 80     ← Requests per instance      │
│   min_pending_latency: 30ms       ← Wait before new instance  │
│   max_pending_latency: automatic                                 │
│                                                                       │
│ handlers:                                                            │
│   - url: /static                                                   │
│     static_dir: static                                             │
│     ⚡ Serve static files directly (fast, no instance needed)     │
│                                                                       │
│   - url: /.*                                                       │
│     script: auto                                                   │
│     secure: always         ← Force HTTPS                         │
│                                                                       │
│ vpc_access_connector:                                                │
│   name: projects/PROJECT/locations/REGION/connectors/connector   │
│   egress_setting: private-ranges-only                             │
│ ⚡ Access private VPC resources (Cloud SQL private IP, Redis).    │
│   egress_setting:                                                 │
│   ├── private-ranges-only: Only RFC 1918 traffic goes through  │
│   └── all-traffic: ALL traffic routes through VPC               │
│                                                                       │
│ inbound_services:                                                    │
│   - warmup                                                          │
│ ⚡ App Engine sends /_ah/warmup request before routing traffic.   │
│   Pre-load caches, establish DB connections, etc.                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           app.yaml — FLEXIBLE ENVIRONMENT                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ runtime: custom          ← or python, nodejs, java, go, etc.     │
│ env: flex                ← THIS makes it flexible!               │
│                                                                       │
│ service: api                                                        │
│                                                                       │
│ # If runtime: custom, provide Dockerfile                          │
│ # If runtime: python/nodejs/etc, App Engine builds for you       │
│                                                                       │
│ resources:                                                           │
│   cpu: 2                 ← vCPUs (1, 2, 4, 8)                   │
│   memory_gb: 2.5         ← Memory in GB                          │
│   disk_size_gb: 20       ← Persistent disk (ephemeral)          │
│                                                                       │
│ automatic_scaling:                                                   │
│   min_num_instances: 1    ← Minimum VMs (can't be 0!)           │
│   max_num_instances: 10   ← Maximum VMs                          │
│   cool_down_period_sec: 120                                      │
│   cpu_utilization:                                                  │
│     target_utilization: 0.6  ← Scale at 60% CPU                │
│                                                                       │
│ network:                                                             │
│   name: vpc-prod                                                   │
│   subnetwork_name: subnet-flex                                   │
│   instance_ip_mode: internal  ← No public IP on VMs            │
│   forwarded_ports:                                                  │
│     - 8080/tcp                                                    │
│   session_affinity: true     ← Sticky sessions                 │
│                                                                       │
│ liveness_check:                                                      │
│   path: /health                                                    │
│   check_interval_sec: 30                                          │
│   timeout_sec: 4                                                   │
│   failure_threshold: 4                                             │
│   success_threshold: 2                                             │
│   initial_delay_sec: 300     ← Wait 5 min after deploy          │
│                                                                       │
│ readiness_check:                                                     │
│   path: /ready                                                     │
│   check_interval_sec: 5                                           │
│   timeout_sec: 4                                                   │
│   failure_threshold: 2                                             │
│   success_threshold: 2                                             │
│   app_start_timeout_sec: 300                                     │
│                                                                       │
│ env_variables:                                                       │
│   NODE_ENV: "production"                                           │
│                                                                       │
│ ⚡ Flexible env uses Compute Engine VMs under the hood.            │
│   Bigger, heavier, more expensive — but more flexible.           │
│   Use Standard gen2 if possible. Use Flex only when needed.     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Scaling Configurations

```
┌─────────────────────────────────────────────────────────────────────┐
│           SCALING TYPES                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. AUTOMATIC SCALING (Standard & Flexible) ✅ default              │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ automatic_scaling:                                           │  │
│ │   min_instances: 0       ← Scale to zero (0 cost at idle!) │  │
│ │   max_instances: 10      ← Cap to control cost             │  │
│ │   min_idle_instances: 1  ← Keep 1 warm (no cold starts)   │  │
│ │   max_idle_instances: 3  ← Don't keep too many idle        │  │
│ │   target_cpu_utilization: 0.65                              │  │
│ │   max_concurrent_requests: 80                               │  │
│ │                                                              │  │
│ │ Flow:                                                        │  │
│ │ Traffic ↑ → CPU > 65% or requests > 80/instance           │  │
│ │          → App Engine adds instances (seconds)              │  │
│ │ Traffic ↓ → Instances idle → Scale down (after cooldown)  │  │
│ │ No traffic → Scale to 0 (if min_instances=0)              │  │
│ │                                                              │  │
│ │ ⚡ Use F1-F4_1G instance classes with automatic scaling     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 2. BASIC SCALING (Standard only)                                   │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ basic_scaling:                                               │  │
│ │   max_instances: 5                                          │  │
│ │   idle_timeout: 5m       ← Shut down after 5 min idle     │  │
│ │                                                              │  │
│ │ ⚡ Instances created on demand, shut down when idle.         │  │
│ │   For: Long-running tasks, background processing.          │  │
│ │   Use B1-B8 instance classes.                               │  │
│ │   No concurrent request limit (1 request = 1 instance).    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 3. MANUAL SCALING (Standard & Flexible)                            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ manual_scaling:                                              │  │
│ │   instances: 3           ← Fixed count, always running     │  │
│ │                                                              │  │
│ │ ⚡ Fixed number of instances. No auto-scaling.               │  │
│ │   For: Predictable load, stateful apps, workers.           │  │
│ │   Use B1-B8 instance classes.                               │  │
│ │   Pay 24/7 for all instances.                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Which to choose:                                                     │
│ ├── Web apps/APIs: Automatic ✅ (most cases)                     │
│ ├── Background workers: Basic or Manual                          │
│ ├── Predictable load: Manual (cheaper, no scaling overhead)    │
│ └── Burst/variable: Automatic (handles spikes)                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Versions & Traffic Splitting

```
┌─────────────────────────────────────────────────────────────────────┐
│           VERSIONS & TRAFFIC SPLITTING                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Every deployment creates a new VERSION:                             │
│ ├── gcloud app deploy → Version: 20260516-v1                    │
│ ├── Old version still exists (can roll back)                    │
│ ├── By default, new version gets 100% traffic                  │
│ ├── Keep old versions: For rollback & traffic splitting         │
│ └── Delete old versions to save resources (if manual/basic)    │
│                                                                       │
│ Traffic Splitting:                                                   │
│ Console → App Engine → Versions → Split traffic                  │
│                                                                       │
│ Split by:                                                            │
│ ● IP address ← Same user → same version (default)              │
│ ○ Cookie    ← More accurate (GOOGAPPUID cookie)                │
│ ○ Random    ← Each request randomly routed                     │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Version       │ Traffic                                       │  │
│ ├──────────────┼──────────────────────────────────────────────┤  │
│ │ v1 (stable)  │ ████████████████████████████████████████ 90% │  │
│ │ v2 (canary)  │ ████ 10%                                     │  │
│ └──────────────┴──────────────────────────────────────────────┘  │
│                                                                       │
│ Use cases:                                                           │
│ ├── Canary deploy: 5% → 25% → 50% → 100%                    │
│ ├── A/B testing: 50/50 split between versions                  │
│ ├── Gradual rollout: Slowly shift traffic to new version      │
│ └── Instant rollback: Shift 100% back to old version          │
│                                                                       │
│ # CLI: Split traffic                                                │
│ gcloud app services set-traffic default \                        │
│   --splits=v1=0.9,v2=0.1 \                                     │
│   --split-by=ip                                                   │
│                                                                       │
│ # Migrate all traffic to new version                              │
│ gcloud app services set-traffic default \                        │
│   --splits=v2=1.0 \                                              │
│   --migrate                                                        │
│                                                                       │
│ # Direct URL to specific version (testing before split):         │
│ https://v2-dot-my-project.appspot.com                            │
│                                                                       │
│ Version lifecycle:                                                   │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Deploy v1 → 100% traffic                                   │  │
│ │ Deploy v2 → Split 90/10 (canary)                           │  │
│ │ Monitor v2 → No errors? → Split 50/50                     │  │
│ │ Confirm v2 → Migrate 100% to v2                            │  │
│ │ Cleanup → Delete v1 (stop instances, save money)           │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Services & Dispatch Rules

```
┌─────────────────────────────────────────────────────────────────────┐
│           SERVICES (MICROSERVICES)                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ One App Engine app can have MULTIPLE services:                     │
│                                                                       │
│ ├── default (required, main entry point)                         │
│ ├── api (REST API)                                                │
│ ├── admin (admin panel)                                           │
│ ├── worker (background processing)                               │
│ └── Each service has its own app.yaml + separate scaling         │
│                                                                       │
│ Each service is an independent app with its own:                   │
│ ├── Instance class (F2 for web, B2 for worker)                  │
│ ├── Scaling config (automatic, basic, manual)                   │
│ ├── Versions and traffic splitting                               │
│ └── URL: https://SERVICE-dot-PROJECT.appspot.com                │
│                                                                       │
│ # Deploy a service:                                                 │
│ # api/app.yaml:                                                     │
│ service: api                                                        │
│ runtime: nodejs20                                                   │
│ instance_class: F2                                                  │
│                                                                       │
│ gcloud app deploy api/app.yaml                                     │
│                                                                       │
│ Dispatch rules (route by URL path):                                │
│ # dispatch.yaml — URL routing to services                         │
│ dispatch:                                                            │
│   - url: "*/api/*"                                                 │
│     service: api                                                    │
│   - url: "*/admin/*"                                               │
│     service: admin                                                  │
│   - url: "*/*"                                                     │
│     service: default                                                │
│                                                                       │
│ gcloud app deploy dispatch.yaml                                    │
│                                                                       │
│ ⚡ Dispatch routes by URL path:                                      │
│   app.example.com/api/users → api service                        │
│   app.example.com/admin/dashboard → admin service                │
│   app.example.com/ → default service                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Custom Domains, SSL & Firewall

```
┌─────────────────────────────────────────────────────────────────────┐
│           CUSTOM DOMAINS & SSL                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → App Engine → Settings → Custom domains                  │
│                                                                       │
│ Add custom domain:                                                   │
│ 1. Verify domain ownership (TXT record or Google Domains)        │
│ 2. Map domain to App Engine                                       │
│ 3. Add DNS records (CNAME or A records)                          │
│                                                                       │
│ DNS records (provided by Google):                                   │
│ ┌────────┬──────────────────────────┬───────────────────────────┐│
│ │ Type    │ Name                      │ Value                     ││
│ ├────────┼──────────────────────────┼───────────────────────────┤│
│ │ CNAME   │ www.example.com           │ ghs.googlehosted.com     ││
│ │ A       │ example.com               │ 216.239.32.21            ││
│ │ A       │ example.com               │ 216.239.34.21            ││
│ │ A       │ example.com               │ 216.239.36.21            ││
│ │ A       │ example.com               │ 216.239.38.21            ││
│ └────────┴──────────────────────────┴───────────────────────────┘│
│                                                                       │
│ SSL certificate:                                                     │
│ ● Google-managed SSL (free, auto-renewed) ✅                      │
│ ○ Upload your own certificate                                     │
│                                                                       │
│ # CLI: Map domain                                                   │
│ gcloud app domain-mappings create example.com \                  │
│   --certificate-management=AUTOMATIC                               │
│                                                                       │
├─────────────────────────────────────────────────────────────────────┤
│           FIREWALL RULES                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → App Engine → Firewall rules                              │
│                                                                       │
│ Default rule: Allow all traffic (priority 2147483647)             │
│                                                                       │
│ Add rules:                                                           │
│ ┌──────────┬────────────────────┬──────────────────────────────┐ │
│ │ Priority  │ IP Range            │ Action                       │ │
│ ├──────────┼────────────────────┼──────────────────────────────┤ │
│ │ 1000      │ 203.0.113.0/24      │ Allow (office IP)           │ │
│ │ 2000      │ 0.0.0.0/0           │ Deny (block all else)      │ │
│ └──────────┴────────────────────┴──────────────────────────────┘ │
│                                                                       │
│ # CLI: Create firewall rule                                        │
│ gcloud app firewall-rules create 1000 \                          │
│   --action=ALLOW \                                                  │
│   --source-range="203.0.113.0/24" \                               │
│   --description="Office IP"                                        │
│                                                                       │
│ gcloud app firewall-rules update default \                       │
│   --action=DENY                                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Cron Jobs & Task Queues

```
┌─────────────────────────────────────────────────────────────────────┐
│           CRON JOBS                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ # cron.yaml — Scheduled tasks                                      │
│ cron:                                                                │
│   - description: "Daily report"                                    │
│     url: /tasks/daily-report                                       │
│     schedule: every day 02:00                                      │
│     timezone: Asia/Kolkata                                         │
│     target: worker                  ← Service to handle it       │
│     retry_parameters:                                               │
│       job_retry_limit: 3                                           │
│       min_backoff_seconds: 60                                      │
│                                                                       │
│   - description: "Hourly cleanup"                                  │
│     url: /tasks/cleanup                                            │
│     schedule: every 1 hours                                        │
│     target: worker                                                  │
│                                                                       │
│   - description: "Weekly backup"                                   │
│     url: /tasks/backup                                             │
│     schedule: every monday 03:00                                   │
│     timezone: Asia/Kolkata                                         │
│     target: worker                                                  │
│                                                                       │
│ gcloud app deploy cron.yaml                                        │
│                                                                       │
│ ⚡ Cron sends HTTP GET to your URL at scheduled times.              │
│   Header: X-Appengine-Cron: true (verify in your handler!)      │
│   Only App Engine can set this header → prevents spoofing.       │
│                                                                       │
│ Schedule formats:                                                    │
│ ├── every 5 minutes                                               │
│ ├── every 1 hours                                                 │
│ ├── every day 14:00 (2 PM)                                       │
│ ├── every monday 09:00                                            │
│ ├── 1st of month 00:00                                            │
│ └── every 24 hours from 00:00 to 23:59                          │
│                                                                       │
├─────────────────────────────────────────────────────────────────────┤
│           CLOUD TASKS (Queue integration)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚡ App Engine integrates with Cloud Tasks for async processing:     │
│                                                                       │
│ 1. Create a Cloud Tasks queue                                      │
│ 2. Push tasks from your app (HTTP target = App Engine URL)       │
│ 3. Cloud Tasks delivers to your App Engine handler               │
│ 4. Retry on failure, rate limiting, deduplication                 │
│                                                                       │
│ # Create queue                                                      │
│ gcloud tasks queues create email-queue \                          │
│   --max-dispatches-per-second=10 \                                │
│   --max-concurrent-dispatches=5                                    │
│                                                                       │
│ # Push task (from your app code — Python example)                 │
│ from google.cloud import tasks_v2                                  │
│ client = tasks_v2.CloudTasksClient()                               │
│ parent = client.queue_path(project, region, "email-queue")       │
│ task = {"app_engine_http_request": {                               │
│     "http_method": "POST",                                         │
│     "relative_uri": "/tasks/send-email",                          │
│     "body": json.dumps({"to": "user@example.com"}).encode(),    │
│ }}                                                                    │
│ client.create_task(parent=parent, task=task)                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform & gcloud CLI

### Terraform

```hcl
# App Engine Application (one per project, permanent region!)
resource "google_app_engine_application" "app" {
  project     = var.project_id
  location_id = "asia-south1"

  database_type = "CLOUD_FIRESTORE"  # or CLOUD_DATASTORE_COMPATIBILITY
}

# Standard version — default service
resource "google_app_engine_standard_app_version" "default" {
  project    = var.project_id
  service    = "default"
  version_id = "v1"
  runtime    = "python312"

  entrypoint {
    shell = "gunicorn -b :$PORT app:app"
  }

  deployment {
    zip {
      source_url = "https://storage.googleapis.com/${google_storage_bucket.deploy.name}/${google_storage_bucket_object.app_zip.name}"
    }
  }

  env_variables = {
    NODE_ENV = "production"
    DB_HOST  = google_sql_database_instance.main.private_ip_address
  }

  instance_class = "F2"

  automatic_scaling {
    min_idle_instances = 1
    max_idle_instances = 3
    min_instances      = 0
    max_instances      = 10
    standard_scheduler_settings {
      target_cpu_utilization        = 0.65
      target_throughput_utilization = 0.65
      max_concurrent_requests      = 80
    }
  }

  vpc_access_connector {
    name           = google_vpc_access_connector.connector.id
    egress_setting = "PRIVATE_RANGES_ONLY"
  }

  delete_service_on_destroy = false
  noop_on_destroy           = true
}

# Flexible version — API service
resource "google_app_engine_flexible_app_version" "api" {
  project    = var.project_id
  service    = "api"
  version_id = "v1"
  runtime    = "custom"

  deployment {
    container {
      image = "gcr.io/${var.project_id}/api:latest"
    }
  }

  env_variables = {
    NODE_ENV = "production"
  }

  resources {
    cpu       = 2
    memory_gb = 2
    disk_gb   = 20
  }

  automatic_scaling {
    min_total_instances = 1
    max_total_instances = 10
    cpu_utilization {
      target_utilization = 0.6
    }
  }

  liveness_check {
    path = "/health"
  }

  readiness_check {
    path = "/ready"
  }

  network {
    name       = google_compute_network.main.name
    subnetwork = google_compute_subnetwork.flex.name
  }

  noop_on_destroy = true
}

# Firewall rule
resource "google_app_engine_firewall_rule" "office" {
  project      = var.project_id
  priority     = 1000
  action       = "ALLOW"
  source_range = "203.0.113.0/24"
  description  = "Office IP"
}

# Domain mapping
resource "google_app_engine_domain_mapping" "app" {
  project     = var.project_id
  domain_name = "app.example.com"

  ssl_settings {
    ssl_management_type = "AUTOMATIC"
  }
}

# Dispatch rules
resource "google_app_engine_application_url_dispatch_rules" "dispatch" {
  project = var.project_id

  dispatch_rules {
    domain  = "*"
    path    = "/api/*"
    service = "api"
  }

  dispatch_rules {
    domain  = "*"
    path    = "/admin/*"
    service = "admin"
  }
}

# Serverless VPC Access Connector
resource "google_vpc_access_connector" "connector" {
  name          = "appengine-connector"
  region        = "asia-south1"
  network       = google_compute_network.main.name
  ip_cidr_range = "10.8.0.0/28"
  min_instances = 2
  max_instances = 10

  machine_type = "e2-micro"
}
```

### gcloud CLI

```bash
# ═══ App & Deployment ═══

# Initialize App Engine (one-time, sets region permanently!)
gcloud app create --region=asia-south1

# Deploy application
gcloud app deploy app.yaml
# ⚡ Uploads code, builds, creates new version, routes traffic.

# Deploy specific version (no traffic)
gcloud app deploy app.yaml --version=v2 --no-promote
# ⚡ --no-promote: Deploy but don't route traffic.
#   Test at: https://v2-dot-PROJECT.appspot.com

# Deploy multiple config files
gcloud app deploy app.yaml cron.yaml dispatch.yaml

# Deploy a service
gcloud app deploy api/app.yaml

# ═══ Browse & Logs ═══

# Open app in browser
gcloud app browse

# Browse specific service
gcloud app browse --service=api

# View logs
gcloud app logs tail
gcloud app logs tail --service=api

# ═══ Versions ═══

# List versions
gcloud app versions list

# Describe version
gcloud app versions describe v2 --service=default

# Delete old version
gcloud app versions delete v1 --service=default

# ═══ Traffic Splitting ═══

# Canary: 90/10 split
gcloud app services set-traffic default \
  --splits=v1=0.9,v2=0.1 \
  --split-by=ip

# Migrate all traffic to v2
gcloud app services set-traffic default \
  --splits=v2=1.0 \
  --migrate

# ═══ Services ═══

# List services
gcloud app services list

# Delete service (⚠️ can't delete "default")
gcloud app services delete worker

# ═══ Instances ═══

# List instances
gcloud app instances list

# SSH into Flexible env instance
gcloud app instances ssh INSTANCE_ID \
  --service=api \
  --version=v1

# Delete specific instance (for debugging/restart)
gcloud app instances delete INSTANCE_ID \
  --service=default \
  --version=v1

# ═══ Domain & Firewall ═══

# Map custom domain
gcloud app domain-mappings create app.example.com \
  --certificate-management=AUTOMATIC

# List domain mappings
gcloud app domain-mappings list

# Create firewall rule
gcloud app firewall-rules create 1000 \
  --action=ALLOW \
  --source-range="203.0.113.0/24" \
  --description="Office IP"

# Block all except office
gcloud app firewall-rules update default --action=DENY

# List firewall rules
gcloud app firewall-rules list

# ═══ Describe App ═══

# Describe app (region, URL, etc.)
gcloud app describe

# Open console
gcloud app open-console
```

---

## Part 9: Real-World Patterns

### Startup

```
Simple web app on App Engine Standard:

Structure:
├── default service (Frontend — Next.js SSR)
│   app.yaml:
│     runtime: nodejs20
│     instance_class: F2
│     automatic_scaling:
│       min_instances: 0  ← Scale to zero (free at night!)
│       max_instances: 5
│   Cost: ~$0-30/month (free tier covers light traffic!)
│
├── api service (Backend — Python Flask)
│   app.yaml:
│     runtime: python312
│     service: api
│     instance_class: F2
│     automatic_scaling:
│       min_instances: 1  ← Always warm
│       max_instances: 5
│   Cost: ~$20-60/month
│
├── dispatch.yaml: /api/* → api service
├── cron.yaml: Daily cleanup, weekly report
└── Custom domain: app.startup.com (Google-managed SSL)

Database: Cloud SQL (private IP via VPC Connector)
Cache: Memorystore Redis (via VPC Connector)
Storage: Cloud Storage (for uploads)

CI/CD: GitHub Actions → gcloud app deploy
Total: ~$30-100/month (with free tier)
```

### Mid-Size

```
Multi-service App Engine + Flexible:

Services:
├── default (Frontend — React SSR, Standard, F4, 2-10 instances)
├── api (REST API — Node.js, Standard, F4, 3-15 instances)
├── admin (Admin panel — Python, Standard, F2, 0-3 instances)
├── worker (Background jobs — Python, Standard Basic, B4, 1-5)
├── ml-api (ML inference — Custom Docker, Flexible, 2 CPU, 1-4)
└── websocket (Real-time — Node.js, Flexible, 2 CPU, 2-6)

Dispatch:
├── /api/* → api
├── /admin/* → admin
├── /ws/* → websocket
├── /ml/* → ml-api
└── /* → default

Traffic splitting (api service):
├── v5 (stable): 95%
├── v6 (canary): 5%
└── Gradually promote v6 over 2 days

Cron jobs:
├── /tasks/daily-report (daily 2 AM)
├── /tasks/cleanup (every 6 hours)
├── /tasks/sync-data (every 30 min)
└── All target: worker service

VPC Connector: All services → Cloud SQL, Redis, Elasticsearch
Firewall: Admin service restricted to office IPs
Secrets: Secret Manager (accessed via env vars or API)

CI/CD:
├── main → deploy to staging (auto)
├── Tag v* → deploy to production (manual approve)
├── Each service deployed independently
└── Traffic split for gradual rollout

Cost: $300-1,200/month
```

### Enterprise

```
App Engine in enterprise (often legacy, migrating to Cloud Run):

Current state (App Engine):
├── 8 Standard services (web, API, admin, 5 workers)
├── 3 Flexible services (custom runtimes, WebSocket)
├── 200+ versions across services
├── Cron: 15 scheduled jobs
├── Cloud Tasks: 5 queues for async processing
├── Traffic splitting: Canary for every deployment
└── VPC Connector: Access to private Cloud SQL, Redis

Migration plan (App Engine → Cloud Run):
├── New services → Cloud Run (not App Engine)
├── Existing services → Migrate one by one:
│   1. Containerize (add Dockerfile)
│   2. Deploy to Cloud Run (same VPC connector)
│   3. Test with internal traffic
│   4. Switch DNS/dispatch to Cloud Run
│   5. Decommission App Engine version
├── Why migrate?
│   ├── Cloud Run: More portable (standard containers)
│   ├── Cloud Run: Better pricing (per-request)
│   ├── Cloud Run: Multi-region (App Engine = single region!)
│   ├── Cloud Run: Knative-compatible (less vendor lock-in)
│   └── App Engine: Still works, just less investment from Google
└── Timeline: 6-12 months

Security:
├── VPC Connector (private resources only)
├── Identity-Aware Proxy (IAP) for admin services
│   ⚡ IAP: Google login required to access admin panel
│   No firewall rules needed — Google handles auth!
├── Firewall rules: Office IPs + VPN only for admin
├── Secret Manager: All credentials
├── Audit logs: Cloud Audit Logs for all deployments
└── DDoS: Cloud Armor (via Cloud Load Balancing)

Cost: $2,000-8,000/month
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Google's original PaaS for web apps |
| Environments | Standard (sandbox, fast, scale-to-zero) and Flexible (Docker VMs) |
| Languages | Python, Node.js, Java, Go, PHP, Ruby (Standard); Any (Flexible) |
| Scale to zero | Standard only (Free tier: 28 instance-hrs/day) |
| Config file | app.yaml (per service) |
| Scaling | Automatic, Basic, Manual |
| Services | Multiple per project (microservices) |
| Versioning | Built-in, with traffic splitting |
| Traffic splitting | IP-based, Cookie-based, or Random |
| Cron | Built-in via cron.yaml |
| Custom domain | Google-managed SSL (free) |
| Firewall | Built-in IP-based rules |
| VPC access | VPC Connector (Standard gen2 + Flexible) |
| Region | Permanent! Cannot change once set |
| One per project | Yes (1 App Engine app per GCP project) |
| AWS equivalent | Elastic Beanstalk / App Runner |
| Azure equivalent | App Service |
| Recommendation | Use Cloud Run for new projects |

---

## Console Walkthrough: Deploying & Managing App Engine

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONSOLE: DEPLOYING & MANAGING APP ENGINE                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. VIEW DEPLOYED VERSIONS                                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Console → App Engine → Versions                             │  │
│ │                                                              │  │
│ │ You'll see a table with:                                    │  │
│ │ ├── Version ID (e.g., 20260522-v1)                         │  │
│ │ ├── Traffic Allocation (% of traffic each version gets)    │  │
│ │ ├── Instances (current running count)                      │  │
│ │ ├── Runtime (python312, nodejs20, etc.)                    │  │
│ │ ├── Environment (Standard / Flexible)                      │  │
│ │ ├── Size (deployed artifact size)                          │  │
│ │ └── Deployed (timestamp)                                    │  │
│ │                                                              │  │
│ │ ⚡ Use the "Service" dropdown at the top to switch between │  │
│ │   services (default, api, worker, etc.)                    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 2. SPLIT TRAFFIC BETWEEN VERSIONS                                  │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Console → App Engine → Versions                             │  │
│ │                                                              │  │
│ │ Step-by-step:                                                │  │
│ │ 1. Select 2+ versions using the checkboxes ☑               │  │
│ │ 2. Click "Split traffic" button at the top                 │  │
│ │ 3. Choose split method:                                     │  │
│ │    ├── IP address (default — same IP → same version)      │  │
│ │    ├── Cookie (GOOGAPPUID — more precise)                  │  │
│ │    └── Random (each request randomly routed)               │  │
│ │ 4. Set percentage for each version:                        │  │
│ │    ├── v1: 90%                                              │  │
│ │    └── v2: 10% (canary)                                    │  │
│ │ 5. Click "Save"                                             │  │
│ │                                                              │  │
│ │ ⚡ Percentages must add up to 100%.                          │  │
│ │ ⚡ Changes take effect within seconds.                       │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 3. STOP SERVING A VERSION                                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Console → App Engine → Versions                             │  │
│ │                                                              │  │
│ │ 1. Select the version(s) you want to stop ☑                │  │
│ │ 2. Click "Stop" button at the top                          │  │
│ │ 3. Confirm the action                                       │  │
│ │                                                              │  │
│ │ ⚡ Stopped versions no longer serve traffic or run          │  │
│ │   instances. You can start them again later.                │  │
│ │ ⚠️ You CANNOT stop a version that's serving traffic!       │  │
│ │   First migrate traffic to another version, then stop.     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 4. VIEW LOGS FOR APP ENGINE                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Option A: App Engine → Versions → click version → Logs    │  │
│ │           (shows logs for that specific version)            │  │
│ │                                                              │  │
│ │ Option B: Console → Logging → Logs Explorer                │  │
│ │           Resource type → "GAE Application"                │  │
│ │           Filter by service, version, severity              │  │
│ │                                                              │  │
│ │ Option C: gcloud app logs tail --service=default           │  │
│ │           (live tail in terminal)                            │  │
│ │                                                              │  │
│ │ Log types:                                                   │  │
│ │ ├── Request logs (HTTP requests — method, path, status)   │  │
│ │ ├── App logs (your print/console.log statements)          │  │
│ │ ├── Stderr (errors from your application)                  │  │
│ │ └── System logs (scaling events, instance start/stop)      │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 5. ⚠️ CANNOT DELETE AN APP ENGINE APPLICATION!                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ You CANNOT delete an App Engine application once created!  │  │
│ │                                                              │  │
│ │ ⚡ Once you create an App Engine app in a project, it's    │  │
│ │   there FOREVER. The only way to truly remove it is to     │  │
│ │   delete the entire GCP project.                            │  │
│ │                                                              │  │
│ │ What you CAN do:                                            │  │
│ │ ├── Disable the application (stops serving traffic)       │  │
│ │ ├── Delete individual versions (free up resources)        │  │
│ │ ├── Delete non-default services                            │  │
│ │ └── But the "default" service always exists                │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 6. DISABLE THE APPLICATION                                          │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Console → App Engine → Settings → Disable Application     │  │
│ │                                                              │  │
│ │ 1. Go to App Engine → Settings                              │  │
│ │ 2. Click "Disable application"                              │  │
│ │ 3. Type the App ID to confirm                               │  │
│ │ 4. Click "Disable"                                          │  │
│ │                                                              │  │
│ │ What happens:                                                │  │
│ │ ├── All traffic returns 404                                 │  │
│ │ ├── All instances are shut down                            │  │
│ │ ├── No more billing for App Engine instances                │  │
│ │ ├── App Engine app still exists (can re-enable later)      │  │
│ │ └── Cron jobs stop running                                  │  │
│ │                                                              │  │
│ │ To re-enable: Settings → Enable Application                │  │
│ │                                                              │  │
│ │ ⚡ This is the closest thing to "deleting" App Engine.      │  │
│ │   Disable when you don't need it but don't want to        │  │
│ │   delete the whole project.                                 │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## App Engine Pricing

```
┌─────────────────────────────────────────────────────────────────────┐
│           APP ENGINE PRICING                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ STANDARD ENVIRONMENT — Per instance-hour                           │
│ ┌────────────┬────────┬───────────┬──────────────────────────┐    │
│ │ Instance   │ Memory │ CPU        │ Cost / instance-hour     │    │
│ │ Class      │        │            │ (USD approx.)            │    │
│ ├────────────┼────────┼───────────┼──────────────────────────┤    │
│ │ F1         │ 256 MB │ 600 MHz   │ $0.05                    │    │
│ │ F2         │ 512 MB │ 1.2 GHz   │ $0.10                    │    │
│ │ F4         │ 1 GB   │ 2.4 GHz   │ $0.20                    │    │
│ │ F4_1G      │ 2 GB   │ 2.4 GHz   │ $0.30                    │    │
│ │ B1         │ 256 MB │ 600 MHz   │ $0.05                    │    │
│ │ B2         │ 512 MB │ 1.2 GHz   │ $0.10                    │    │
│ │ B4         │ 1 GB   │ 2.4 GHz   │ $0.20                    │    │
│ │ B4_1G      │ 2 GB   │ 2.4 GHz   │ $0.30                    │    │
│ │ B8         │ 2 GB   │ 4.8 GHz   │ $0.40                    │    │
│ └────────────┴────────┴───────────┴──────────────────────────┘    │
│                                                                       │
│ ⚡ F classes = Automatic scaling                                    │
│ ⚡ B classes = Basic / Manual scaling                               │
│                                                                       │
│ FLEXIBLE ENVIRONMENT — Per resource-hour                           │
│ ┌──────────────────────────────┬─────────────────────────────┐    │
│ │ Resource                      │ Cost / hour (USD approx.)    │    │
│ ├──────────────────────────────┼─────────────────────────────┤    │
│ │ vCPU                          │ $0.0526                      │    │
│ │ Memory (per GB)               │ $0.0071                      │    │
│ │ Persistent disk (per GB/mo)  │ $0.0400                      │    │
│ └──────────────────────────────┴─────────────────────────────┘    │
│                                                                       │
│ Example: 1 Flex instance (2 vCPU, 4 GB RAM, 20 GB disk)          │
│ ├── vCPU:   2 × $0.0526 = $0.1052/hr                           │
│ ├── Memory: 4 × $0.0071 = $0.0284/hr                           │
│ ├── Disk:   20 × $0.04/730 = $0.0011/hr                        │
│ └── Total:  ~$0.13/hr ≈ ~$97/month (24/7)                     │
│                                                                       │
│ FREE TIER (Standard only):                                         │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ✅ 28 instance-hours/day for F instances (F1-F4_1G)        │  │
│ │ ✅ 9 instance-hours/day for B instances (B1-B8)            │  │
│ │ ✅ 1 GB egress/day                                          │  │
│ │ ✅ Shared Memcache                                          │  │
│ │ ✅ 1,000 search operations/day                              │  │
│ │ ✅ 5 GB Cloud Storage                                       │  │
│ │                                                              │  │
│ │ ⚡ With F1 + auto scaling + scale to zero:                   │  │
│ │   Light-traffic apps can run essentially FREE!              │  │
│ │   28 F1 instance-hours ≈ 1 instance running all day.       │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ Flexible environment has NO free tier.                           │
│ ⚡ Always set max_instances to prevent surprise bills!              │
│ ⚡ Pricing varies by region — check the latest pricing page.       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Limitations & Gotchas

```
┌─────────────────────────────────────────────────────────────────────┐
│           LIMITATIONS & GOTCHAS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. ONE App Engine app per project                                  │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ├── Each GCP project can have exactly ONE App Engine app   │  │
│ │ ├── You can have multiple services within that app         │  │
│ │ ├── But you cannot create a second App Engine app          │  │
│ │ └── Need another app? Create a new GCP project!            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 2. CANNOT change region after creation                             │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ├── Region is set PERMANENTLY when you run                 │  │
│ │ │   gcloud app create --region=<REGION>                    │  │
│ │ ├── Picked the wrong region? Too bad — can't change it!   │  │
│ │ ├── Only option: Delete the project & start over            │  │
│ │ └── Choose wisely! (asia-south1, us-central1, etc.)        │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 3. CANNOT delete the app (only disable)                            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ├── Once created, App Engine app exists forever in project │  │
│ │ ├── You can DISABLE it (Settings → Disable Application)   │  │
│ │ ├── You can delete versions and non-default services       │  │
│ │ ├── But the app itself? Stuck with it.                      │  │
│ │ └── To truly remove: Delete the entire GCP project          │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 4. Standard gen1: No WebSockets, 60s request timeout             │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ├── gen1 runtimes (Python 2.7, Java 8, Go 1.11):          │  │
│ │ │   ├── No WebSocket support ❌                            │  │
│ │ │   ├── Request timeout: 60 seconds (not 60 min!)         │  │
│ │ │   ├── Proprietary APIs (ndb, taskqueue, memcache)       │  │
│ │ │   └── Being deprecated — migrate to gen2!               │  │
│ │ ├── gen2 runtimes (Python 3.9+, Node.js 18+, Java 17+):  │  │
│ │ │   ├── WebSocket support ✅                               │  │
│ │ │   ├── Request timeout: up to 60 minutes ✅               │  │
│ │ │   └── Standard libraries (no proprietary APIs)          │  │
│ │ └── ⚡ Always use gen2 for new apps!                        │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 5. File system is READ-ONLY (Standard environment)                │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ├── Standard env: Cannot write to disk ❌                   │  │
│ │ │   ├── /tmp is available (in-memory tmpfs, limited)      │  │
│ │ │   ├── Use Cloud Storage for file uploads                 │  │
│ │ │   └── Use Memorystore/Redis for caching                  │  │
│ │ ├── Flexible env: Ephemeral disk available (read/write)   │  │
│ │ │   ├── Data lost when instance restarts                   │  │
│ │ │   └── Still use Cloud Storage for persistent files       │  │
│ │ └── ⚡ Never rely on local disk for persistent data!        │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Quick summary of gotchas:                                           │
│ ┌──────────────────────┬────────────────────────────────────────┐ │
│ │ Gotcha                │ Workaround                              │ │
│ ├──────────────────────┼────────────────────────────────────────┤ │
│ │ 1 app per project    │ Use multiple GCP projects              │ │
│ │ Region is permanent  │ Choose carefully at creation time     │ │
│ │ Can't delete app     │ Disable it, or delete the project     │ │
│ │ No WebSockets (gen1) │ Use gen2 Standard or Flexible env     │ │
│ │ Read-only disk (Std) │ Use /tmp, Cloud Storage, Memorystore  │ │
│ │ No VPC access (gen1) │ Use gen2 with VPC Connector            │ │
│ │ Cold starts (Std)    │ Set min_instances: 1 or warmup handler│ │
│ │ 60s timeout (gen1)   │ Use gen2 (60 min) or Flexible          │ │
│ │ Vendor lock-in       │ Consider Cloud Run for new projects   │ │
│ └──────────────────────┴────────────────────────────────────────┘ │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover Google Cloud's container orchestration features with Anthos — multi-cloud and hybrid Kubernetes management.

→ Next: [Chapter 20: Anthos](20-anthos.md)

---

*Last Updated: May 2026*
