# Chapter 18: Azure Container Instances (ACI)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: ACI Fundamentals](#part-1-aci-fundamentals)
- [Part 2: Creating a Container Instance (Full Portal Walkthrough)](#part-2-creating-a-container-instance-full-portal-walkthrough)
- [Part 3: Container Groups (Multi-Container)](#part-3-container-groups-multi-container)
- [Part 4: Restart Policies](#part-4-restart-policies)
- [Part 5: Virtual Network Integration](#part-5-virtual-network-integration)
- [Part 6: Terraform & Bicep](#part-6-terraform--bicep)
- [Part 7: az CLI Reference](#part-7-az-cli-reference)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Container Instances (ACI) is the fastest and simplest way to run containers in Azure. No VMs, no clusters, no orchestration — just give it a container image and it runs. ACI is ideal for short-lived tasks, burst workloads, and scenarios where you need a container without the overhead of managing infrastructure.

```
What you'll learn:
├── ACI Fundamentals
│   ├── What & why (serverless containers)
│   ├── ACI vs AKS vs Container Apps vs App Service
│   └── Container groups concept
├── Creating a Container Instance (Full Portal Walkthrough)
│   ├── Basics (name, image, size, OS, region)
│   ├── Networking (public IP, ports, DNS, VNet)
│   ├── Advanced (restart policy, env vars, command)
│   ├── Volumes (Azure Files, Git repo, emptyDir, secret)
│   └── Tags
├── Container Groups (Multi-Container)
│   ├── Sidecar pattern
│   ├── Ambassador, Adapter patterns
│   └── Shared networking & volumes
├── Restart Policies
├── GPU Container Instances
├── Virtual Network Integration
├── Confidential Containers
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: ACI Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           ACI CONCEPT                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is ACI?                                                        │
│ ├── Run a container WITHOUT managing VMs or clusters             │
│ ├── Start in seconds (not minutes like VMs)                     │
│ ├── Pay per second of usage (CPU + memory)                      │
│ ├── Linux and Windows containers supported                      │
│ └── No orchestration (single container group, not a fleet)      │
│                                                                       │
│ Think of it as:                                                      │
│ "docker run" in the cloud — instant, no setup.                   │
│                                                                       │
│ Container Group (deployment unit):                                  │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Container Group: my-app-group                                │  │
│ │ ├── IP: 20.81.xxx.xxx (public or private)                  │  │
│ │ ├── DNS: my-app.centralindia.azurecontainer.io             │  │
│ │ │                                                           │  │
│ │ ├── Container 1: web-app (nginx:latest)                    │  │
│ │ │   CPU: 1, Memory: 1.5 GiB, Port: 80                    │  │
│ │ │                                                           │  │
│ │ ├── Container 2: log-shipper (fluentd sidecar)            │  │
│ │ │   CPU: 0.5, Memory: 0.5 GiB                            │  │
│ │ │                                                           │  │
│ │ └── Shared: Network (localhost), Volumes                   │  │
│ │                                                              │  │
│ │ ⚡ Like a Kubernetes Pod — containers share network & vols │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Comparison:                                                          │
│ ┌────────────────┬──────────────┬──────────────┬──────────────┐  │
│ │ Feature         │ ACI           │ AKS           │ Container Apps│  │
│ ├────────────────┼──────────────┼──────────────┼──────────────┤  │
│ │ Complexity      │ Lowest ✅    │ Highest       │ Medium        │  │
│ │ Orchestration   │ None         │ Full K8s      │ Managed       │  │
│ │ Auto-scaling    │ None ❌      │ HPA/CA ✅     │ KEDA ✅       │  │
│ │ Scale to zero   │ N/A          │ With Karpenter│ Yes ✅        │  │
│ │ Service mesh    │ No           │ Yes (Istio)   │ Dapr built-in │  │
│ │ Load balancing  │ No (single IP)│ Yes ✅       │ Yes ✅        │  │
│ │ Startup time    │ Seconds ✅   │ Minutes       │ Seconds       │  │
│ │ GPU             │ Yes ✅       │ Yes           │ No            │  │
│ │ Windows         │ Yes          │ Yes           │ Yes           │  │
│ │ VNet integration│ Yes          │ Yes           │ Yes           │  │
│ │ Best for        │ Simple tasks │ Complex apps  │ Microservices │  │
│ └────────────────┴──────────────┴──────────────┴──────────────┘  │
│                                                                       │
│ Use ACI for:                                                        │
│ ├── One-off tasks (data migration, ETL, builds)                │
│ ├── CI/CD build agents                                          │
│ ├── Burst workloads from AKS (virtual kubelet)                 │
│ ├── Event-driven tasks (run container, exit when done)         │
│ ├── Dev/test environments                                       │
│ ├── GPU workloads (ML inference)                                │
│ └── Sidecar patterns (proxy + app together)                    │
│                                                                       │
│ DON'T use ACI for:                                                  │
│ ├── Long-running production services (use AKS/Container Apps) │
│ ├── Auto-scaling fleet of containers                            │
│ ├── Complex multi-service architectures                        │
│ └── Services needing load balancing across instances           │
│                                                                       │
│ ⚡ AWS equivalent: ECS Fargate (single task)                      │
│ ⚡ GCP equivalent: Cloud Run Jobs (for tasks)                     │
│                                                                       │
│ Pricing:                                                             │
│ ├── Linux: $0.0000135/sec per vCPU + $0.0000015/sec per GiB   │
│ │   ≈ $0.049/hr per vCPU + $0.0054/hr per GiB                │
│ ├── Windows: ~2× Linux pricing                                  │
│ ├── GPU: $0.68-3.60/hr per GPU (depends on type)              │
│ └── No charge when container group is stopped (deallocated)   │
│                                                                       │
│ Limits:                                                              │
│ ├── Max CPU per container group: 4 vCPU (Linux), 4 (Windows) │
│ ├── Max memory per container group: 16 GiB                     │
│ ├── Max containers per group: 60                                │
│ ├── Max GPU: 4 (K80, P100, V100)                               │
│ └── Max volume mounts: 5 per container                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Container Instance (Full Portal Walkthrough)

```
Console → Container instances → Create

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 1: BASICS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-containers ▼]                                  │
│                                                                       │
│ Container name: [my-web-app]                                       │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ Availability zones: [None ▼]                                      │
│ ⚡ ACI supports zone selection in some regions.                    │
│   Deploy to specific zone for zone-level resilience.            │
│                                                                       │
│ SKU: ● Standard  ○ Dedicated  ○ Confidential                    │
│ ⚡ Standard: Shared infra (cheaper, most common).                  │
│   Dedicated: Isolated hardware (compliance, performance).       │
│   Confidential: Hardware-based encryption (SEV-SNP enclaves).  │
│                                                                       │
│ ── Image source ──                                                  │
│ ● Azure Container Registry                                        │
│ ○ Docker Hub or other registry                                    │
│ ○ Quickstart images                                                │
│                                                                       │
│ ═══ Azure Container Registry ═══                                   │
│ Registry: [myregistry.azurecr.io ▼]                              │
│ Image: [web-app ▼]                                                │
│ Image tag: [v1.2 ▼]                                               │
│ ⚡ Full image: myregistry.azurecr.io/web-app:v1.2                 │
│                                                                       │
│ ═══ Docker Hub ═══                                                  │
│ Image type: ● Public  ○ Private                                  │
│ Image: [nginx:latest]                                              │
│ (If private:)                                                       │
│ Image registry login server: [https://index.docker.io/v1/]     │
│ Username: [myuser]                                                  │
│ Password: [********]                                                │
│                                                                       │
│ ── Size ──                                                          │
│                                                                       │
│ Number of CPU cores: [1 ▼]  (0.25, 0.5, 1, 2, 4)              │
│ Memory (GiB): [1.5 ▼]      (0.5, 1, 1.5, 2, 4, 8, 16)        │
│                                                                       │
│ ⚡ Size determines cost. Start small, increase if needed.         │
│   1 vCPU + 1.5 GiB ≈ $38/month (if running 24/7).              │
│                                                                       │
│ ── GPU (optional) ──                                                │
│ GPU type: [None ▼]                                                 │
│ ├── None (default)                                                │
│ ├── K80 (1, 2, or 4 GPUs)                                       │
│ ├── P100 (1, 2, or 4 GPUs)                                      │
│ └── V100 (1, 2, or 4 GPUs)                                      │
│ GPU count: [1 ▼]                                                  │
│ ⚡ GPU available in limited regions. Use for ML inference.        │
│                                                                       │
│ [Next: Networking >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 2: NETWORKING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Networking type:                                                     │
│ ● Public                                                             │
│ ○ Private                                                            │
│ ○ None (no network — isolated container)                           │
│                                                                       │
│ ═══ PUBLIC ═══                                                      │
│ DNS name label: [my-web-app]                                       │
│ ⚡ Creates: my-web-app.centralindia.azurecontainer.io              │
│                                                                       │
│ Ports:                                                               │
│ ┌────────────┬──────────────┐                                      │
│ │ Port        │ Protocol      │                                      │
│ ├────────────┼──────────────┤                                      │
│ │ 80          │ TCP           │                                      │
│ │ 443         │ TCP           │                                      │
│ │ [+ Add]     │               │                                      │
│ └────────────┴──────────────┘                                      │
│                                                                       │
│ ═══ PRIVATE (VNet integration) ═══                                  │
│ Virtual network: [vnet-prod ▼]                                     │
│ Subnet: [subnet-aci (10.0.10.0/24) ▼]                            │
│ ⚡ Private = container gets a VPC private IP.                      │
│   ├── No public IP, not accessible from internet                │
│   ├── Can talk to other VNet resources (SQL, Redis, VMs)       │
│   ├── Subnet must be delegated to Microsoft.ContainerInstance  │
│   └── Use for: Internal tasks, accessing private resources     │
│                                                                       │
│ ⚡ Subnet delegation (required):                                    │
│   Subnet → Properties → Delegation:                              │
│   Microsoft.ContainerInstance/containerGroups                    │
│   ⚠️ Once delegated, only ACI can use this subnet!              │
│                                                                       │
│ [Next: Advanced >]                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 3: ADVANCED                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Restart policy ──                                                │
│                                                                       │
│ ● Always (container exits → restart immediately)                 │
│   ⚡ For long-running services (web servers, APIs).               │
│                                                                       │
│ ○ On failure (restart only on non-zero exit code)               │
│   ⚡ For tasks that should retry on error.                        │
│                                                                       │
│ ○ Never (run once, stop when container exits)                   │
│   ⚡ For one-off tasks: migrations, builds, reports.             │
│   Container exits → container group status = "Terminated".     │
│   ⚠️ You still pay until you DELETE the container group!        │
│                                                                       │
│ ── Environment variables ──                                        │
│ [+ Add]                                                              │
│ ┌────────────────────────┬──────────────────┬──────────────────┐ │
│ │ Name                    │ Value              │ Secure           │ │
│ ├────────────────────────┼──────────────────┼──────────────────┤ │
│ │ NODE_ENV               │ production        │ ☐                 │ │
│ │ DB_HOST                │ db.company.com    │ ☐                 │ │
│ │ DB_PASSWORD            │ ********          │ ☑ (hidden!) ✅   │ │
│ │ API_KEY                │ sk-xxxx           │ ☑ (hidden!)      │ │
│ └────────────────────────┴──────────────────┴──────────────────┘ │
│                                                                       │
│ ⚡ Secure = value never shown in portal, CLI, or API responses.  │
│   Use for secrets. But for production, use Key Vault instead.   │
│                                                                       │
│ ── Command override ──                                              │
│ ["/bin/sh", "-c", "npm run migrate && npm start"]                │
│ ⚡ Overrides the Docker CMD/ENTRYPOINT.                            │
│   Leave empty to use the image's default command.               │
│                                                                       │
│ ── Volume mounts ──                                                 │
│ [+ Add volume]                                                      │
│                                                                       │
│ Volume type:                                                        │
│ ├── Azure file share                                              │
│ │   Share name: [my-share]                                       │
│ │   Storage account: [stmydata ▼]                                │
│ │   Storage account key: [********]                              │
│ │   Mount path: [/mnt/data]                                     │
│ │   Read only: ☐                                                 │
│ │   ⚡ Persistent! Data survives container restarts.              │
│ │   ⚡ Shared across containers in the group.                     │
│ │                                                                 │
│ ├── Empty directory (emptyDir)                                   │
│ │   Mount path: [/tmp/cache]                                    │
│ │   ⚡ Temp storage. Shared between sidecars. Lost on restart.  │
│ │                                                                 │
│ ├── Git repo (clone git repo into volume)                       │
│ │   Repository URL: [https://github.com/org/config.git]        │
│ │   Directory: [.]                                               │
│ │   Mount path: [/mnt/config]                                   │
│ │   ⚡ Cloned at startup. Good for config files.                  │
│ │                                                                 │
│ └── Secret                                                        │
│     Secrets are mounted as files.                                │
│     Secret name: db-password                                     │
│     Value: base64-encoded                                        │
│     Mount path: /mnt/secrets/db-password                        │
│     ⚡ Each secret becomes a file. Cat /mnt/secrets/db-password  │
│                                                                       │
│ [Next: Tags > ] → [Review + create] → [Create]                   │
│                                                                       │
│ ⚡ Container starts in 5-30 seconds!                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Container Groups (Multi-Container)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONTAINER GROUPS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Container Group = Multiple containers sharing:                     │
│ ├── Network (same IP, communicate via localhost)                 │
│ ├── Volumes (mount same Azure File share)                       │
│ ├── Lifecycle (start/stop together)                              │
│ └── Resources (CPU/memory pool)                                  │
│                                                                       │
│ ⚡ Like a Kubernetes Pod, but simpler.                             │
│                                                                       │
│ Sidecar pattern:                                                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Container Group: app-with-logging                            │  │
│ │                                                              │  │
│ │ ┌───────────────┐  ┌────────────────────┐                   │  │
│ │ │ Main: web-app │  │ Sidecar: log-agent │                   │  │
│ │ │ Port: 80      │  │ Reads /var/log/*   │                   │  │
│ │ │ Writes logs to│  │ Ships to Log       │                   │  │
│ │ │ /var/log/     │  │ Analytics          │                   │  │
│ │ └───────┬───────┘  └────────┬───────────┘                   │  │
│ │         │                    │                               │  │
│ │         └──── Shared Volume ─┘                              │  │
│ │              (emptyDir: /var/log)                            │  │
│ │                                                              │  │
│ │ Shared IP: 20.81.xxx.xxx:80 → web-app                     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Ambassador pattern (API proxy):                                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ┌───────────────┐  ┌────────────────────┐                   │  │
│ │ │ Main: app     │  │ Ambassador: envoy  │                   │  │
│ │ │ → localhost   │  │ Port: 443 (TLS)    │                   │  │
│ │ │   :8080       │──│ → localhost:8080   │                   │  │
│ │ │ (no TLS)      │  │ (handles TLS)      │                   │  │
│ │ └───────────────┘  └────────────────────┘                   │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Multi-container YAML (deploy with az CLI):                         │
│ apiVersion: '2023-05-01'                                           │
│ type: Microsoft.ContainerInstance/containerGroups                  │
│ location: centralindia                                              │
│ properties:                                                          │
│   osType: Linux                                                     │
│   restartPolicy: Always                                             │
│   containers:                                                        │
│     - name: web-app                                                │
│       properties:                                                   │
│         image: myregistry.azurecr.io/web-app:v1.2                │
│         resources:                                                  │
│           requests:                                                 │
│             cpu: 1.0                                                │
│             memoryInGb: 1.5                                        │
│         ports:                                                      │
│           - port: 80                                                │
│         volumeMounts:                                               │
│           - name: logs                                              │
│             mountPath: /var/log/app                                │
│     - name: log-agent                                              │
│       properties:                                                   │
│         image: fluent/fluentd:latest                               │
│         resources:                                                  │
│           requests:                                                 │
│             cpu: 0.5                                                │
│             memoryInGb: 0.5                                        │
│         volumeMounts:                                               │
│           - name: logs                                              │
│             mountPath: /var/log/app                                │
│             readOnly: true                                         │
│   volumes:                                                           │
│     - name: logs                                                    │
│       emptyDir: {}                                                  │
│   ipAddress:                                                        │
│     type: Public                                                    │
│     ports:                                                           │
│       - port: 80                                                    │
│         protocol: TCP                                               │
│     dnsNameLabel: my-web-app                                       │
│   imageRegistryCredentials:                                         │
│     - server: myregistry.azurecr.io                               │
│       identity: /subscriptions/.../identity/aci-identity          │
│                                                                       │
│ Deploy:                                                              │
│ az container create --resource-group rg-containers \              │
│   --file container-group.yaml                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Restart Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│           RESTART POLICIES                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌─────────────┬──────────────────────────────────────────────────┐│
│ │ Policy       │ Behavior                                         ││
│ ├─────────────┼──────────────────────────────────────────────────┤│
│ │ Always       │ Container exits → restart immediately           ││
│ │ (default)    │ Use for: Web servers, long-running services    ││
│ │              │ ⚡ Backoff: Restart delay increases if crashing ││
│ ├─────────────┼──────────────────────────────────────────────────┤│
│ │ OnFailure    │ Exit code 0 → stop (success)                   ││
│ │              │ Exit code ≠ 0 → restart (failure)              ││
│ │              │ Use for: Tasks that should retry on error      ││
│ ├─────────────┼──────────────────────────────────────────────────┤│
│ │ Never        │ Run once, never restart                         ││
│ │              │ Exit code 0 → Terminated (success)             ││
│ │              │ Exit code ≠ 0 → Terminated (failure)           ││
│ │              │ Use for: One-off tasks, CI/CD, migrations      ││
│ │              │ ⚠️ Delete container group to stop billing!      ││
│ └─────────────┴──────────────────────────────────────────────────┘│
│                                                                       │
│ Container group lifecycle:                                          │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Pending → Running → [container exits]                      │  │
│ │                                                              │  │
│ │ Always:    → Waiting → Running → [exits] → Waiting → ...  │  │
│ │ OnFailure: → Running → [exit 0] → Terminated (Succeeded)  │  │
│ │            → Running → [exit 1] → Waiting → Running → ... │  │
│ │ Never:     → Running → [exit] → Terminated                │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ Stop/Start container group:                                      │
│   az container stop --name my-app --resource-group rg-aci        │
│   → Stops all containers, deallocates resources (no billing!)   │
│   az container start --name my-app --resource-group rg-aci       │
│   → Restarts with same config (new IP if public)                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Virtual Network Integration

```
┌─────────────────────────────────────────────────────────────────────┐
│           VNET INTEGRATION                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Deploy ACI into a VNet for:                                         │
│ ├── Accessing private resources (SQL, Redis, Storage)           │
│ ├── No public IP (secure internal workloads)                    │
│ ├── Network segmentation and NSG rules                          │
│ └── Communication with other VNet resources                     │
│                                                                       │
│ Requirements:                                                        │
│ ├── Subnet MUST be delegated to Microsoft.ContainerInstance     │
│ ├── Subnet needs enough IPs (each group gets 1 IP)             │
│ ├── /24 subnet = 251 container groups max                      │
│ └── Cannot be the same subnet as other resources                │
│                                                                       │
│ Setup:                                                               │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ VNet: vnet-prod (10.0.0.0/16)                               │  │
│ │ ├── subnet-app (10.0.1.0/24) → VMs, App Service            │  │
│ │ ├── subnet-db (10.0.2.0/24) → SQL, Redis                  │  │
│ │ └── subnet-aci (10.0.10.0/24) → ACI (delegated!)          │  │
│ │                                                              │  │
│ │ ACI (10.0.10.5) → subnet-db (10.0.2.x) → SQL Server     │  │
│ │ ⚡ ACI can reach SQL on private IP, no public endpoint!     │  │
│ │                                                              │  │
│ │ NSG on subnet-aci:                                          │  │
│ │ ├── Allow outbound to subnet-db:1433 (SQL)                │  │
│ │ ├── Allow outbound to subnet-db:6379 (Redis)              │  │
│ │ └── Deny all other outbound (least privilege)             │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ VNet ACI limitations:                                             │
│   ├── No public IP (use Application Gateway or Azure Front Door)│
│   ├── Cannot use DNS name label                                 │
│   ├── Some regions only                                          │
│   └── Windows containers: Limited VNet support                  │
│                                                                       │
│ ACI + Application Gateway (public access to VNet ACI):            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Internet → App Gateway (public IP, WAF)                    │  │
│ │           → Backend: ACI private IP (10.0.10.5:80)        │  │
│ │                                                              │  │
│ │ ⚡ App Gateway provides: SSL, WAF, path routing.            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & Bicep

### Terraform

```hcl
# Simple ACI (public, single container)
resource "azurerm_container_group" "web" {
  name                = "aci-web-app"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  ip_address_type     = "Public"
  dns_name_label      = "my-web-app"
  restart_policy      = "Always"

  container {
    name   = "web-app"
    image  = "myregistry.azurecr.io/web-app:v1.2"
    cpu    = 1
    memory = 1.5

    ports {
      port     = 80
      protocol = "TCP"
    }

    environment_variables = {
      NODE_ENV = "production"
      DB_HOST  = "db.company.com"
    }

    secure_environment_variables = {
      DB_PASSWORD = var.db_password
      API_KEY     = var.api_key
    }

    volume {
      name                 = "data"
      mount_path           = "/mnt/data"
      storage_account_name = azurerm_storage_account.main.name
      storage_account_key  = azurerm_storage_account.main.primary_access_key
      share_name           = azurerm_storage_share.data.name
    }

    liveness_probe {
      http_get {
        path   = "/health"
        port   = 80
        scheme = "Http"
      }
      initial_delay_seconds = 30
      period_seconds        = 10
    }
  }

  image_registry_credential {
    server                    = "myregistry.azurecr.io"
    user_assigned_identity_id = azurerm_user_assigned_identity.aci.id
  }

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.aci.id]
  }

  tags = {
    environment = "prod"
  }
}

# Multi-container (sidecar pattern)
resource "azurerm_container_group" "multi" {
  name                = "aci-app-with-sidecar"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  ip_address_type     = "Public"
  dns_name_label      = "my-app-sidecar"
  restart_policy      = "Always"

  # Main container
  container {
    name   = "web-app"
    image  = "myregistry.azurecr.io/web-app:v1.2"
    cpu    = 1
    memory = 1

    ports {
      port     = 80
      protocol = "TCP"
    }

    volume {
      name       = "logs"
      mount_path = "/var/log/app"
      empty_dir  = true
    }
  }

  # Sidecar container
  container {
    name   = "log-shipper"
    image  = "fluent/fluentd:latest"
    cpu    = 0.5
    memory = 0.5

    volume {
      name       = "logs"
      mount_path = "/var/log/app"
      read_only  = true
      empty_dir  = true
    }
  }
}

# VNet-integrated ACI (private)
resource "azurerm_container_group" "private" {
  name                = "aci-worker"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  ip_address_type     = "Private"
  subnet_ids          = [azurerm_subnet.aci.id]
  restart_policy      = "Never"

  container {
    name   = "data-migration"
    image  = "myregistry.azurecr.io/migrator:v1"
    cpu    = 2
    memory = 4

    commands = ["/bin/sh", "-c", "python migrate.py"]

    environment_variables = {
      DB_HOST = azurerm_mssql_server.main.fully_qualified_domain_name
    }

    secure_environment_variables = {
      DB_PASSWORD = var.db_password
    }
  }
}

# Subnet delegation for ACI
resource "azurerm_subnet" "aci" {
  name                 = "subnet-aci"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.10.0/24"]

  delegation {
    name = "aci-delegation"
    service_delegation {
      name    = "Microsoft.ContainerInstance/containerGroups"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/action",
      ]
    }
  }
}

# One-off task (run and delete)
resource "azurerm_container_group" "task" {
  name                = "aci-batch-${formatdate("YYYYMMDDhhmm", timestamp())}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  ip_address_type     = "None"
  restart_policy      = "Never"

  container {
    name   = "batch-job"
    image  = "myregistry.azurecr.io/batch:v1"
    cpu    = 2
    memory = 4

    commands = ["python", "process_data.py", "--batch-size=1000"]
  }
}
```

### Bicep

```bicep
resource containerGroup 'Microsoft.ContainerInstance/containerGroups@2023-05-01' = {
  name: 'aci-web-app'
  location: resourceGroup().location
  properties: {
    osType: 'Linux'
    restartPolicy: 'Always'
    containers: [
      {
        name: 'web-app'
        properties: {
          image: '${acrLoginServer}/web-app:v1.2'
          resources: {
            requests: {
              cpu: 1
              memoryInGB: json('1.5')
            }
          }
          ports: [{ port: 80, protocol: 'TCP' }]
          environmentVariables: [
            { name: 'NODE_ENV', value: 'production' }
            { name: 'DB_PASSWORD', secureValue: dbPassword }
          ]
        }
      }
    ]
    ipAddress: {
      type: 'Public'
      dnsNameLabel: 'my-web-app'
      ports: [{ port: 80, protocol: 'TCP' }]
    }
    imageRegistryCredentials: [
      {
        server: acrLoginServer
        identity: managedIdentity.id
      }
    ]
  }
}
```

---

## Part 7: az CLI Reference

```bash
# Create simple container (public, from Docker Hub)
az container create \
  --resource-group rg-containers \
  --name aci-nginx \
  --image nginx:latest \
  --cpu 1 \
  --memory 1.5 \
  --ports 80 443 \
  --dns-name-label my-nginx-app \
  --restart-policy Always

# Create container from ACR (with managed identity)
az container create \
  --resource-group rg-containers \
  --name aci-web-app \
  --image myregistry.azurecr.io/web-app:v1.2 \
  --acr-identity /subscriptions/.../identity/aci-identity \
  --assign-identity /subscriptions/.../identity/aci-identity \
  --cpu 2 \
  --memory 4 \
  --ports 80 \
  --dns-name-label my-web-app \
  --environment-variables NODE_ENV=production DB_HOST=db.company.com \
  --secure-environment-variables DB_PASSWORD=s3cret123 \
  --restart-policy Always

# Create one-off task (run once, exit)
az container create \
  --resource-group rg-containers \
  --name aci-migration \
  --image myregistry.azurecr.io/migrator:v1 \
  --cpu 2 \
  --memory 4 \
  --restart-policy Never \
  --command-line "python migrate.py --env production" \
  --no-wait

# Create in VNet (private)
az container create \
  --resource-group rg-containers \
  --name aci-worker \
  --image myregistry.azurecr.io/worker:v1 \
  --cpu 1 \
  --memory 2 \
  --vnet vnet-prod \
  --subnet subnet-aci \
  --restart-policy OnFailure

# Mount Azure Files volume
az container create \
  --resource-group rg-containers \
  --name aci-with-storage \
  --image myregistry.azurecr.io/app:v1 \
  --cpu 1 \
  --memory 1.5 \
  --azure-file-volume-account-name stmydata \
  --azure-file-volume-account-key "xxxx" \
  --azure-file-volume-share-name myshare \
  --azure-file-volume-mount-path /mnt/data

# Create from YAML (multi-container)
az container create \
  --resource-group rg-containers \
  --file container-group.yaml

# Create with GPU
az container create \
  --resource-group rg-containers \
  --name aci-ml \
  --image myregistry.azurecr.io/ml-model:v1 \
  --cpu 4 \
  --memory 16 \
  --gpu-count 1 \
  --gpu-sku V100 \
  --restart-policy Never

# ═══ Management ═══

# List containers
az container list --resource-group rg-containers --output table

# Show container details
az container show \
  --resource-group rg-containers \
  --name aci-web-app \
  --output table

# View logs
az container logs \
  --resource-group rg-containers \
  --name aci-web-app \
  --container-name web-app

# Follow logs (stream)
az container logs \
  --resource-group rg-containers \
  --name aci-web-app \
  --follow

# Exec into container (interactive shell)
az container exec \
  --resource-group rg-containers \
  --name aci-web-app \
  --container-name web-app \
  --exec-command "/bin/sh"

# Stop container group (deallocate — no billing!)
az container stop \
  --resource-group rg-containers \
  --name aci-web-app

# Start container group
az container start \
  --resource-group rg-containers \
  --name aci-web-app

# Restart container group
az container restart \
  --resource-group rg-containers \
  --name aci-web-app

# Delete container group
az container delete \
  --resource-group rg-containers \
  --name aci-web-app \
  --yes

# Export to YAML (backup config)
az container export \
  --resource-group rg-containers \
  --name aci-web-app \
  --file aci-export.yaml
```

---

## Part 8: Real-World Patterns

### Startup

```
Use case: Quick API + batch tasks

API (long-running):
├── aci-api-prod
├── Image: Node.js API from ACR
├── CPU: 1, Memory: 1.5 GiB
├── Public IP + DNS: api.centralindia.azurecontainer.io
├── Restart: Always
├── Env vars: DB connection, API keys (secure)
├── Azure Files mount: /mnt/uploads (persistent storage)
└── Cost: ~$38/month

Batch tasks (on-demand):
├── aci-migrate (restart: Never, run DB migrations)
├── aci-report (restart: Never, generate monthly reports)
├── aci-import (restart: Never, import CSV data)
└── Created via CI/CD or manually, deleted after completion

Dev environments:
├── aci-dev-john (per-developer containers)
├── Quick spin-up for testing
├── Delete when done (pay only when running)
└── Cost: ~$0.05/hour each

Total: ~$50-80/month
```

### Mid-Size

```
ACI as burst compute from AKS:

AKS cluster: aks-prod (main workloads)
├── Normal pods run on AKS node pools
├── Burst traffic → AKS Virtual Nodes (ACI!)
│   ⚡ Virtual Kubelet: Pods overflow to ACI
│   ├── Pod scheduled on virtual node → ACI container created
│   ├── No node provisioning delay (seconds, not minutes)
│   ├── Scale to hundreds of pods instantly
│   └── Pods destroyed when traffic subsides
└── Nodepool: virtual-node (tolerations: virtual-kubelet.io/provider)

Standalone ACI (tasks):
├── CI/CD build agents
│   ├── aci-build-{run-id} (restart: Never)
│   ├── Created by Azure DevOps / GitHub Actions
│   ├── Run build + tests in isolated container
│   └── Deleted after completion
│
├── Data processing
│   ├── aci-etl-nightly (restart: Never)
│   ├── VNet integrated → access private SQL
│   ├── Triggered by Logic App timer
│   └── 2 vCPU, 8 GiB, runs ~30 min
│
└── ML inference (GPU)
    ├── aci-ml-predict (GPU: V100)
    ├── Process batch of images/documents
    ├── Private VNet → read from Storage, write to Cosmos
    └── Cost: ~$3.60/hr × hours used

Cost: $200-600/month (burst + tasks)
```

### Enterprise

```
ACI for enterprise use cases:

Regulated workloads:
├── Confidential containers (ACI Confidential SKU)
│   ├── SEV-SNP hardware enclaves
│   ├── Code + data encrypted in memory
│   ├── Even Azure operators can't access
│   └── Use for: Financial data, healthcare, PII processing

CI/CD pipeline agents:
├── Self-hosted build agents on ACI
├── Azure DevOps agent pools → ACI
├── Each build: Fresh container (clean env)
├── VNet: Access private repos, package registries
├── Scale: 0 to 50 concurrent builds
└── Auto-cleanup: Container deleted after build

Scheduled tasks:
├── aci-report-daily (Logic App timer → create ACI → delete)
├── aci-etl-hourly (Event Grid → create ACI → process → delete)
├── aci-compliance-scan (weekly, scan infra, generate report)
└── All in private VNet → access internal resources

Burst from AKS:
├── AKS production cluster → Virtual Nodes
├── Black Friday: Burst 100+ pods to ACI (seconds!)
├── Normal day: 0 ACI pods (pay nothing for burst capacity)
├── Pod anti-affinity: Keep critical pods on real nodes
└── Tolerations: Only batch/stateless pods go to ACI

Security:
├── Managed identity (no passwords in container config)
├── ACR with private endpoint (images stay in VNet)
├── VNet integration for all internal tasks
├── NSG: Restrict ACI subnet to needed ports only
├── Azure Policy: Enforce allowed images, CPU/memory limits
├── Diagnostic logs → Log Analytics
└── Microsoft Defender for Containers

Cost: $500-3,000/month (tasks + burst + GPU)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Serverless containers (no VMs, no clusters) |
| Startup time | 5-30 seconds |
| Max CPU | 4 vCPU per container group |
| Max memory | 16 GiB per container group |
| Max containers | 60 per group |
| GPU | K80, P100, V100 (1-4 per group) |
| OS | Linux and Windows |
| Restart policies | Always, OnFailure, Never |
| Networking | Public IP, Private (VNet), None |
| Volumes | Azure Files, emptyDir, Git repo, Secret |
| Multi-container | Yes (sidecar pattern, shared network/volumes) |
| Billing | Per-second (CPU + memory), stopped = free |
| VNet | Requires delegated subnet |
| Confidential | SEV-SNP enclaves (hardware encryption) |
| AKS integration | Virtual Nodes (burst to ACI) |
| AWS equivalent | ECS Fargate (single task) |
| GCP equivalent | Cloud Run Jobs |

---

## What's Next?

In the next chapter, we'll cover Azure Kubernetes Service (AKS) — fully managed Kubernetes on Azure.

→ Next: [Chapter 19: AKS - Azure Kubernetes Service](19-aks.md)

---

*Last Updated: May 2026*
