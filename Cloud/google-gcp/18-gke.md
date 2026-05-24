# Chapter 18: GKE - Google Kubernetes Engine

---

## Table of Contents

- [Overview](#overview)
- [Part 1: GKE Fundamentals](#part-1-gke-fundamentals)
- [Part 2: Cluster Creation — Autopilot Mode](#part-2-cluster-creation--autopilot-mode)
- [Part 3: Cluster Creation — Standard Mode](#part-3-cluster-creation--standard-mode)
- [Part 4: Node Pools (Standard Mode)](#part-4-node-pools-standard-mode)
- [Part 5: GKE Networking & Ingress](#part-5-gke-networking--ingress)
- [Part 6: Workload Identity Federation](#part-6-workload-identity-federation)
- [Part 7: GKE Autoscaling](#part-7-gke-autoscaling)
- [Part 8: Terraform](#part-8-terraform)
- [Part 9: gcloud CLI Reference](#part-9-gcloud-cli-reference)
- [Part 10: Real-World Patterns](#part-10-real-world-patterns)
- [Quick Reference](#quick-reference)
- [Kubernetes Basics for Beginners](#kubernetes-basics-for-beginners)
- [Deploying Your First App on GKE](#deploying-your-first-app-on-gke)
- [Console Walkthrough: Managing & Deleting Clusters](#console-walkthrough-managing--deleting-clusters)
- [Connecting GKE to Cloud SQL](#connecting-gke-to-cloud-sql)
- [What's Next?](#whats-next)

---

## Overview

Google Kubernetes Engine (GKE) is Google Cloud's fully managed Kubernetes service. Google literally invented Kubernetes (Borg → Kubernetes), and GKE is considered the most mature, feature-rich managed Kubernetes offering. GKE comes in two modes: **Autopilot** (Google manages everything including nodes) and **Standard** (you manage node pools).

```
What you'll learn:
├── GKE Fundamentals
│   ├── What & why (managed K8s, Google-built)
│   ├── Autopilot vs Standard mode
│   └── GKE vs Cloud Run vs Cloud Functions
├── Cluster Creation (Full Console Walkthrough)
│   ├── Autopilot mode (fully managed)
│   ├── Standard mode (node pool management)
│   ├── Networking (VPC-native, private cluster)
│   ├── Security (Workload Identity, Shielded Nodes)
│   └── Features (auto-upgrade, auto-repair, release channels)
├── Node Pools (Standard mode)
│   ├── Machine types, scaling, Spot VMs
│   ├── Labels, taints, image types
│   └── GPU node pools
├── Deploying Workloads
│   ├── Deployments, Services, Ingress
│   ├── GKE Ingress (Google Cloud Load Balancer)
│   ├── Gateway API
│   └── Config Connector
├── Workload Identity Federation
├── GKE Autoscaling (HPA, VPA, Cluster Autoscaler, NAP)
├── Terraform & gcloud CLI
└── Real-world patterns
```

---

## Part 1: GKE Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           GKE CONCEPT                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is GKE?                                                        │
│ ├── Fully managed Kubernetes control plane                       │
│ ├── Google manages: API server, etcd, scheduler, controllers    │
│ ├── Auto-upgrades, auto-repairs, auto-scales                    │
│ ├── Integrated with GCP (IAM, networking, monitoring, logging)  │
│ └── Most mature K8s offering (Google created Kubernetes!)       │
│                                                                       │
│ Two modes:                                                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ AUTOPILOT (recommended for most) ✅                         │  │
│ │ ├── Google manages EVERYTHING (nodes, scaling, security)   │  │
│ │ ├── You deploy pods → Google provisions nodes              │  │
│ │ ├── Pay per pod (CPU + memory), not per node               │  │
│ │ ├── Pre-configured security best practices                 │  │
│ │ ├── No node SSH, no privileged containers                  │  │
│ │ ├── Auto-scales nodes based on pod demand                  │  │
│ │ └── Best for: Most workloads, teams without K8s ops        │  │
│ │                                                              │  │
│ │ STANDARD (full control)                                     │  │
│ │ ├── You manage node pools (machine type, count, OS)        │  │
│ │ ├── Pay per node (VM) regardless of pod utilization        │  │
│ │ ├── Full access to nodes (SSH, DaemonSets, privileged)    │  │
│ │ ├── Node pool autoscaling (cluster autoscaler)             │  │
│ │ ├── Custom node images, GPUs, Windows nodes               │  │
│ │ └── Best for: Custom requirements, GPUs, Windows, legacy  │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Comparison:                                                          │
│ ┌────────────────┬──────────────┬──────────────┬──────────────┐  │
│ │ Feature         │ Autopilot     │ Standard      │ Cloud Run    │  │
│ ├────────────────┼──────────────┼──────────────┼──────────────┤  │
│ │ Node management│ Google       │ You           │ N/A          │  │
│ │ Pod to zero    │ Yes (scale)  │ Yes (Karpenter)│ Yes ✅      │  │
│ │ GPU support    │ Yes (A100)   │ Yes ✅        │ No ❌        │  │
│ │ DaemonSets     │ Limited      │ Yes ✅        │ No ❌        │  │
│ │ Privileged pods│ No ❌        │ Yes           │ No ❌        │  │
│ │ Pricing        │ Per pod      │ Per node      │ Per request  │  │
│ │ K8s expertise  │ Medium       │ High          │ Low ✅       │  │
│ │ Control plane  │ Free ✅      │ Free (zonal)  │ N/A          │  │
│ │                │              │ $73/mo (reg)  │              │  │
│ │ Best for       │ Most K8s     │ Custom, GPU   │ Simple HTTP  │  │
│ └────────────────┴──────────────┴──────────────┴──────────────┘  │
│                                                                       │
│ ⚡ Control plane pricing:                                            │
│   Autopilot: Free!                                                │
│   Standard zonal: Free!                                           │
│   Standard regional: $0.10/hr ($73/mo) per cluster              │
│   You pay for nodes (Standard) or pods (Autopilot).             │
│                                                                       │
│ ⚡ AWS equivalent: EKS (no autopilot mode — closest: Fargate)    │
│ ⚡ Azure equivalent: AKS                                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Cluster Creation — Autopilot Mode

```
Console → Kubernetes Engine → Clusters → Create → Autopilot

┌─────────────────────────────────────────────────────────────────────┐
│           AUTOPILOT CLUSTER                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Cluster basics ──                                                │
│                                                                       │
│ Name: [gke-prod]                                                    │
│ Region: [asia-south1 ▼]                                           │
│ ⚡ Autopilot is always regional (multi-zone HA).                   │
│   Control plane and pods span 3 zones automatically.            │
│                                                                       │
│ Release channel:                                                     │
│ ○ Rapid    ← Latest K8s, early features, fast upgrades          │
│ ● Regular  ← Balanced stability & features (default) ✅          │
│ ○ Stable   ← Most tested, slowest to get new features          │
│ ○ Extended ← 24 months support (vs 14 for Regular)             │
│                                                                       │
│ ⚡ Release channels control:                                        │
│   ├── Which K8s version you get                                 │
│   ├── When auto-upgrades happen                                 │
│   ├── Rapid: 1-2 weeks behind upstream K8s release             │
│   ├── Regular: 2-3 months behind                                │
│   └── Stable: 3-5 months behind                                │
│                                                                       │
│ ── Networking ──                                                    │
│                                                                       │
│ Network: [default ▼]                                               │
│ Node subnet: [default ▼]                                          │
│ Pod IPv4 address range: [Auto ▼] (secondary range for pods)     │
│ Service address range: [Auto ▼] (secondary range for services)  │
│                                                                       │
│ ⚡ GKE uses VPC-native (alias IPs) networking:                     │
│   ├── Pods get IPs from a secondary range on the subnet         │
│   ├── Services get IPs from another secondary range             │
│   ├── Pods can communicate with VPC resources directly          │
│   └── Required for Autopilot, recommended for Standard         │
│                                                                       │
│ ☑ Enable private cluster                                          │
│ ├── ☑ Access control plane using its external IP address       │
│ │   ⚡ Public endpoint but restrict to CIDRs below.             │
│ │   Master authorized networks: [203.0.113.0/24] (office IP)  │
│ │   Or: Disable for fully private (kubectl from VPC only).    │
│ ├── Control plane IP range: [172.16.0.0/28] (default)        │
│ └── ⚡ Private cluster = nodes have NO public IPs.               │
│     Pods reach internet via Cloud NAT.                          │
│                                                                       │
│ ── Security ──                                                      │
│                                                                       │
│ ☑ Workload Identity Federation                                    │
│ ⚡ Pods get GCP IAM identities (like IRSA on EKS).               │
│   K8s ServiceAccount ↔ GCP Service Account.                     │
│   No service account key files needed!                          │
│   Always enabled on Autopilot.                                  │
│                                                                       │
│ ☑ Shielded GKE Nodes                                              │
│ ⚡ Secure boot, vTPM, integrity monitoring on all nodes.         │
│   Always enabled on Autopilot.                                  │
│                                                                       │
│ ☐ Enable Binary Authorization                                     │
│ ⚡ Enforce that only signed/trusted images run on cluster.       │
│                                                                       │
│ Encryption:                                                          │
│ ● Google-managed encryption key                                   │
│ ○ Customer-managed encryption key (CMEK)                        │
│                                                                       │
│ ── Advanced ──                                                      │
│                                                                       │
│ Maintenance window:                                                 │
│ ☑ Set maintenance window                                          │
│ Start time: [02:00] UTC   Duration: [4 hours]                   │
│ Days: [Any ▼]                                                     │
│ ⚡ Upgrades and maintenance happen within this window.            │
│                                                                       │
│ Maintenance exclusion:                                              │
│ [+ Add exclusion]                                                   │
│ Name: black-friday                                                  │
│ Start: 2026-11-25  End: 2026-11-30                                │
│ ⚡ No upgrades during critical business periods.                  │
│                                                                       │
│ [CREATE]                                                             │
│ ⚡ Cluster creation: ~5 minutes.                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Cluster Creation — Standard Mode

```
Console → Kubernetes Engine → Clusters → Create → Standard

┌─────────────────────────────────────────────────────────────────────┐
│           STANDARD CLUSTER — CLUSTER CONFIG                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [gke-prod-std]                                                │
│                                                                       │
│ Location type:                                                       │
│ ○ Zonal (control plane in 1 zone — free, but SPOF!)            │
│ ● Regional (control plane in 3 zones — $73/mo, HA!) ✅          │
│                                                                       │
│ Region: [asia-south1 ▼]                                           │
│ Specify default node locations:                                    │
│ ☑ asia-south1-a                                                   │
│ ☑ asia-south1-b                                                   │
│ ☑ asia-south1-c                                                   │
│                                                                       │
│ Release channel: [Regular ▼]                                      │
│ Control plane version: [1.30.5-gke.1200 (default) ▼]            │
│                                                                       │
│ ── Features ──                                                      │
│ ☑ Enable Workload Identity Federation                             │
│ ☑ Enable Dataplane V2 (eBPF-based networking)                   │
│ ⚡ Dataplane V2: Better performance, built-in NetworkPolicy       │
│   enforcement, no need for Calico.                              │
│                                                                       │
│ ☑ Enable Gateway API                                              │
│ ⚡ Next-gen Ingress. More expressive routing rules.               │
│                                                                       │
│ ☐ Enable Config Connector                                         │
│ ⚡ Manage GCP resources via K8s manifests (optional).             │
│                                                                       │
│ ── Security ──                                                      │
│ ☑ Shielded GKE Nodes                                              │
│ ☑ Enable Workload Vulnerability Scanning                        │
│ ☐ Binary Authorization                                            │
│ Encryption: Google-managed or CMEK                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Node Pools (Standard Mode)

```
Cluster → Node pools → Create node pool  (or edit default-pool)

┌─────────────────────────────────────────────────────────────────────┐
│           NODE POOL CONFIGURATION                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [pool-general]                                                │
│                                                                       │
│ ── Node pool details ──                                            │
│                                                                       │
│ Number of nodes (per zone): [2]                                    │
│ ⚡ If regional cluster with 3 zones: 2 × 3 = 6 nodes total.      │
│                                                                       │
│ ☑ Enable autoscaling                                               │
│ Minimum nodes per zone: [1]                                        │
│ Maximum nodes per zone: [5]                                        │
│ ⚡ Cluster Autoscaler watches pending pods →                       │
│   No room for pod → add node. Nodes empty → remove.            │
│                                                                       │
│ ☐ Enable Spot VMs                                                  │
│ ⚡ Spot = 60-91% cheaper, can be preempted (2-24 hours).         │
│   Best for: Batch, non-critical, fault-tolerant workloads.     │
│   Add taint: cloud.google.com/gke-spot=true:NoSchedule          │
│                                                                       │
│ Surge upgrade:                                                       │
│ Max surge: [1]                                                      │
│ Max unavailable: [0]                                                │
│ ⚡ During node upgrades:                                            │
│   Surge 1 + Unavail 0 = Create 1 new → drain 1 old → repeat.  │
│   Always have full capacity. Zero downtime.                    │
│                                                                       │
│ ── Nodes config ──                                                  │
│                                                                       │
│ Machine type: [e2-standard-4 ▼]                                   │
│ ┌──────────────────────┬────────┬──────────┬────────────────────┐ │
│ │ Type                  │ vCPU   │ Memory    │ Best for           │ │
│ ├──────────────────────┼────────┼──────────┼────────────────────┤ │
│ │ e2-standard-2         │ 2      │ 8 GiB    │ Small workloads   │ │
│ │ e2-standard-4         │ 4      │ 16 GiB   │ General purpose   │ │
│ │ e2-standard-8         │ 8      │ 32 GiB   │ Larger services   │ │
│ │ n2-standard-4         │ 4      │ 16 GiB   │ Better perf      │ │
│ │ n2-highmem-4          │ 4      │ 32 GiB   │ Memory-intensive │ │
│ │ t2a-standard-4 (Arm)  │ 4      │ 16 GiB   │ Cost-efficient   │ │
│ │ a2-highgpu-1g (GPU)   │ 12     │ 85 GiB   │ ML/AI (A100 GPU) │ │
│ └──────────────────────┴────────┴──────────┴────────────────────┘ │
│                                                                       │
│ Boot disk type: [SSD persistent disk ▼]                           │
│ Boot disk size: [100] GB (default — increase for many images)    │
│                                                                       │
│ Node image: [Container-Optimized OS with containerd (cos_containerd)│
│              ▼]                                                      │
│ ├── cos_containerd (default, minimal, secure, auto-update) ✅    │
│ ├── Ubuntu with containerd (full Ubuntu, more packages)         │
│ └── Windows Server (Windows containers)                          │
│                                                                       │
│ ── Networking ──                                                    │
│                                                                       │
│ Max pods per node: [110] (default, GKE uses /24 per node)       │
│ ⚡ Each node gets a /24 from the pod CIDR (256 IPs).              │
│   Ensure pod CIDR is large enough! /14 = 262K IPs.             │
│                                                                       │
│ ── Node metadata ──                                                 │
│                                                                       │
│ Kubernetes labels:                                                   │
│ role = general                                                      │
│ env = prod                                                          │
│                                                                       │
│ Node taints:                                                        │
│ [+ Add taint]                                                       │
│ Key: gpu  Value: true  Effect: NoSchedule                        │
│ ⚡ Taints prevent pods without matching tolerations.               │
│                                                                       │
│ ── Security ──                                                      │
│ Service account: [gke-nodes@project.iam.gserviceaccount.com ▼] │
│ ⚡ Create dedicated SA for nodes with minimal permissions.        │
│   Default Compute Engine SA has too many permissions!           │
│   Node SA needs: logging.logWriter, monitoring.metricWriter,   │
│   monitoring.viewer, storage.objectViewer (pull images).        │
│                                                                       │
│ ☑ Enable Secure Boot                                               │
│ ☑ Enable Integrity Monitoring                                     │
│                                                                       │
│ [CREATE]                                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: GKE Networking & Ingress

```
┌─────────────────────────────────────────────────────────────────────┐
│           GKE INGRESS & SERVICES                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GKE Ingress = Google Cloud Load Balancer (automatic!)             │
│                                                                       │
│ Service types:                                                       │
│ ├── ClusterIP: Internal only (pod-to-pod, default)              │
│ ├── NodePort: Expose on each node's IP:port                     │
│ ├── LoadBalancer: Provision a GCP Network LB (L4, TCP/UDP)     │
│ └── Ingress: Provision a GCP HTTP(S) LB (L7, path routing) ✅  │
│                                                                       │
│ External HTTP(S) Load Balancer via Ingress:                       │
│                                                                       │
│ apiVersion: networking.k8s.io/v1                                   │
│ kind: Ingress                                                       │
│ metadata:                                                            │
│   name: web-ingress                                                 │
│   annotations:                                                      │
│     kubernetes.io/ingress.global-static-ip-name: "web-ip"        │
│     networking.gke.io/managed-certificates: "web-cert"           │
│     kubernetes.io/ingress.class: "gce"  # External HTTPS LB     │
│ spec:                                                                │
│   rules:                                                             │
│     - host: app.example.com                                       │
│       http:                                                         │
│         paths:                                                      │
│           - path: /                                                │
│             pathType: Prefix                                      │
│             backend:                                               │
│               service:                                             │
│                 name: web-app                                     │
│                 port:                                               │
│                   number: 80                                      │
│           - path: /api                                             │
│             pathType: Prefix                                      │
│             backend:                                               │
│               service:                                             │
│                 name: api-backend                                 │
│                 port:                                               │
│                   number: 80                                      │
│                                                                       │
│ ⚡ This creates a Google Cloud HTTPS LB with:                      │
│   ├── Global anycast IP                                          │
│   ├── Google-managed SSL certificate                             │
│   ├── Cloud CDN (optional)                                       │
│   ├── Cloud Armor WAF (optional)                                │
│   ├── Health checks auto-configured                             │
│   └── Backend services pointing to K8s node ports or NEGs      │
│                                                                       │
│ Internal Load Balancer (for internal services):                   │
│   annotations:                                                      │
│     kubernetes.io/ingress.class: "gce-internal"                  │
│                                                                       │
│ Google-managed SSL certificate:                                    │
│ apiVersion: networking.gke.io/v1                                   │
│ kind: ManagedCertificate                                           │
│ metadata:                                                            │
│   name: web-cert                                                    │
│ spec:                                                                │
│   domains:                                                          │
│     - app.example.com                                              │
│                                                                       │
│ ⚡ Gateway API (next-gen, more powerful than Ingress):             │
│ apiVersion: gateway.networking.k8s.io/v1                          │
│ kind: Gateway                                                       │
│ metadata:                                                            │
│   name: external-http                                               │
│ spec:                                                                │
│   gatewayClassName: gke-l7-global-external-managed               │
│   listeners:                                                        │
│     - name: https                                                  │
│       protocol: HTTPS                                              │
│       port: 443                                                    │
│       tls:                                                          │
│         mode: Terminate                                            │
│         certificateRefs:                                           │
│           - name: web-cert                                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Workload Identity Federation

```
┌─────────────────────────────────────────────────────────────────────┐
│           WORKLOAD IDENTITY FEDERATION                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: Pods need access to GCP services (Cloud SQL, GCS, etc.) │
│                                                                       │
│ ❌ Bad: Download service account key JSON → mount in pod.         │
│   Key never expires, can be leaked, hard to rotate.             │
│                                                                       │
│ ✅ Good: Workload Identity — K8s SA ↔ GCP SA (no keys!)          │
│   Pod uses K8s SA → auto-gets GCP SA credentials (short-lived).│
│                                                                       │
│ How it works:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ K8s ServiceAccount: app-sa (namespace: backend)             │  │
│ │     ↕ bound via IAM policy                                 │  │
│ │ GCP Service Account: sa-app@project.iam.gserviceaccount.com│  │
│ │     → roles/cloudsql.client                                │  │
│ │     → roles/storage.objectViewer                           │  │
│ │                                                              │  │
│ │ Pod with serviceAccountName: app-sa                        │  │
│ │ → Automatically gets GCP SA credentials                    │  │
│ │ → Can access Cloud SQL, GCS as sa-app                     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Setup:                                                               │
│                                                                       │
│ # 1. Create GCP service account                                    │
│ gcloud iam service-accounts create sa-app \                      │
│   --display-name="App Service Account"                            │
│                                                                       │
│ # 2. Grant GCP roles to the service account                       │
│ gcloud projects add-iam-policy-binding PROJECT_ID \              │
│   --member="serviceAccount:sa-app@PROJECT.iam.gserviceaccount" \│
│   --role="roles/cloudsql.client"                                  │
│                                                                       │
│ # 3. Allow K8s SA to impersonate GCP SA                           │
│ gcloud iam service-accounts add-iam-policy-binding \             │
│   sa-app@PROJECT.iam.gserviceaccount.com \                       │
│   --role="roles/iam.workloadIdentityUser" \                      │
│   --member="serviceAccount:PROJECT.svc.id.goog[backend/app-sa]"│
│                                                                       │
│ # 4. Annotate K8s ServiceAccount                                   │
│ kubectl annotate serviceaccount app-sa \                         │
│   -n backend \                                                    │
│   iam.gke.io/gcp-service-account=sa-app@PROJECT.iam             │
│                                                                       │
│ # 5. Use in pod                                                    │
│ spec:                                                                │
│   serviceAccountName: app-sa                                      │
│                                                                       │
│ ⚡ AWS equivalent: IRSA (IAM Roles for Service Accounts)          │
│ ⚡ Always enabled on Autopilot. Must enable on Standard.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: GKE Autoscaling

```
┌─────────────────────────────────────────────────────────────────────┐
│           AUTOSCALING LAYERS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Layer 1: POD autoscaling                                            │
│ ├── HPA (Horizontal Pod Autoscaler): Add/remove pod replicas    │
│ │   kubectl autoscale deployment web --cpu-percent=70            │
│ │   --min=3 --max=20                                             │
│ │                                                                 │
│ │   Or: Custom metrics (Pub/Sub messages, request count)        │
│ │   Or: KEDA (event-driven autoscaling)                         │
│ │                                                                 │
│ ├── VPA (Vertical Pod Autoscaler): Adjust pod CPU/memory        │
│ │   ⚡ Watches actual usage → recommends or auto-resizes.        │
│ │   apiVersion: autoscaling.k8s.io/v1                           │
│ │   kind: VerticalPodAutoscaler                                 │
│ │   spec:                                                        │
│ │     targetRef:                                                 │
│ │       apiVersion: apps/v1                                     │
│ │       kind: Deployment                                        │
│ │       name: web-app                                            │
│ │     updatePolicy:                                              │
│ │       updateMode: "Auto"  # or "Off" (recommend only)        │
│ │                                                                 │
│ └── MPA (Multidimensional Pod Autoscaler — GKE-specific):       │
│     Combines HPA + VPA: Scale replicas AND resize pods.          │
│                                                                       │
│ Layer 2: NODE autoscaling (Standard mode)                          │
│ ├── Cluster Autoscaler (built into node pool config):            │
│ │   Pod pending (no room) → Add node (1-5 min).                │
│ │   Node empty → Remove node (after 10 min idle).              │
│ │   Configured per node pool: min/max nodes per zone.          │
│ │                                                                 │
│ ├── Node Auto-Provisioning (NAP):                                │
│ │   ⚡ Automatically creates NEW node pools based on demand!     │
│ │   Pod needs GPU? → NAP creates GPU node pool.                │
│ │   Pod needs ARM? → NAP creates ARM node pool.                │
│ │   You set resource limits (max CPU, max memory, GPU types).  │
│ │                                                                 │
│ │   Console: Cluster → Automation → Node auto-provisioning     │
│ │   Resource limits: Max CPU: [100], Max Memory: [400 GiB]    │
│ │   GPU: ☑ nvidia-tesla-t4, max: [8]                          │
│ │                                                                 │
│ └── Autopilot: All node scaling is automatic (you set nothing!) │
│                                                                       │
│ Autoscaling flow (Standard):                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Traffic ↑ → HPA: More pods → No room on nodes →           │  │
│ │ Cluster Autoscaler: Add node → Pods scheduled.             │  │
│ │                                                              │  │
│ │ Traffic ↓ → HPA: Fewer pods → Nodes empty →               │  │
│ │ Cluster Autoscaler: Remove node (after cooldown).          │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Autoscaling flow (Autopilot):                                      │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Traffic ↑ → HPA: More pods → GKE auto-provisions nodes →  │  │
│ │ Pods scheduled. You see nothing, it just works!            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform

```hcl
# GKE Autopilot Cluster
resource "google_container_cluster" "autopilot" {
  name     = "gke-prod"
  location = "asia-south1"

  enable_autopilot = true

  release_channel {
    channel = "REGULAR"
  }

  network    = google_compute_network.main.id
  subnetwork = google_compute_subnetwork.gke.id

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false  # Public endpoint + auth networks
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "203.0.113.0/24"
      display_name = "Office IP"
    }
  }

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  maintenance_policy {
    recurring_window {
      start_time = "2026-01-01T02:00:00Z"
      end_time   = "2026-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR"
    }
  }

  deletion_protection = true
}

# GKE Standard Cluster
resource "google_container_cluster" "standard" {
  name     = "gke-prod-std"
  location = "asia-south1"

  remove_default_node_pool = true
  initial_node_count       = 1

  release_channel {
    channel = "REGULAR"
  }

  network    = google_compute_network.main.id
  subnetwork = google_compute_subnetwork.gke.id

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  datapath_provider = "ADVANCED_DATAPATH"  # Dataplane V2

  gateway_api_config {
    channel = "CHANNEL_STANDARD"
  }

  deletion_protection = true
}

# Node Pool — General
resource "google_container_node_pool" "general" {
  name     = "pool-general"
  cluster  = google_container_cluster.standard.id
  location = "asia-south1"

  initial_node_count = 2

  autoscaling {
    min_node_count = 1
    max_node_count = 5
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  upgrade_settings {
    max_surge       = 1
    max_unavailable = 0
  }

  node_config {
    machine_type    = "e2-standard-4"
    disk_size_gb    = 100
    disk_type       = "pd-ssd"
    image_type      = "COS_CONTAINERD"
    service_account = google_service_account.gke_nodes.email

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform",
    ]

    labels = {
      role = "general"
      env  = "prod"
    }

    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    workload_metadata_config {
      mode = "GKE_METADATA"  # Workload Identity
    }
  }
}

# Node Pool — Spot (for batch/workers)
resource "google_container_node_pool" "spot" {
  name     = "pool-spot"
  cluster  = google_container_cluster.standard.id
  location = "asia-south1"

  initial_node_count = 0

  autoscaling {
    min_node_count = 0
    max_node_count = 10
  }

  node_config {
    machine_type    = "e2-standard-4"
    spot            = true
    disk_size_gb    = 100
    service_account = google_service_account.gke_nodes.email

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform",
    ]

    labels = {
      role = "spot"
    }

    taint {
      key    = "cloud.google.com/gke-spot"
      value  = "true"
      effect = "NO_SCHEDULE"
    }
  }
}

# Node Service Account (minimal permissions)
resource "google_service_account" "gke_nodes" {
  account_id   = "gke-nodes"
  display_name = "GKE Node Service Account"
}

resource "google_project_iam_member" "gke_node_roles" {
  for_each = toset([
    "roles/logging.logWriter",
    "roles/monitoring.metricWriter",
    "roles/monitoring.viewer",
    "roles/stackdriver.resourceMetadata.writer",
    "roles/artifactregistry.reader",
  ])

  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

# Workload Identity — App Service Account
resource "google_service_account" "app" {
  account_id   = "sa-app"
  display_name = "App Service Account"
}

resource "google_project_iam_member" "app_sql" {
  project = var.project_id
  role    = "roles/cloudsql.client"
  member  = "serviceAccount:${google_service_account.app.email}"
}

resource "google_service_account_iam_binding" "app_workload_identity" {
  service_account_id = google_service_account.app.name
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "serviceAccount:${var.project_id}.svc.id.goog[backend/app-sa]",
  ]
}

# Subnet with secondary ranges for pods/services
resource "google_compute_subnetwork" "gke" {
  name          = "subnet-gke"
  ip_cidr_range = "10.0.0.0/20"
  region        = "asia-south1"
  network       = google_compute_network.main.id

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.4.0.0/14"
  }

  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.8.0.0/20"
  }
}
```

---

## Part 9: gcloud CLI Reference

```bash
# ═══ Cluster management ═══

# Create Autopilot cluster
gcloud container clusters create-auto gke-prod \
  --region=asia-south1 \
  --release-channel=regular \
  --network=vpc-prod \
  --subnetwork=subnet-gke \
  --enable-private-nodes \
  --master-ipv4-cidr=172.16.0.0/28 \
  --enable-master-authorized-networks \
  --master-authorized-networks=203.0.113.0/24

# Create Standard cluster
gcloud container clusters create gke-prod-std \
  --region=asia-south1 \
  --release-channel=regular \
  --network=vpc-prod \
  --subnetwork=subnet-gke \
  --enable-ip-alias \
  --enable-private-nodes \
  --master-ipv4-cidr=172.16.0.0/28 \
  --enable-shielded-nodes \
  --workload-pool=PROJECT_ID.svc.id.goog \
  --enable-dataplane-v2 \
  --num-nodes=2 \
  --machine-type=e2-standard-4 \
  --disk-type=pd-ssd \
  --disk-size=100

# Get credentials (configure kubectl)
gcloud container clusters get-credentials gke-prod \
  --region=asia-south1

# List clusters
gcloud container clusters list

# Describe cluster
gcloud container clusters describe gke-prod --region=asia-south1

# Upgrade cluster
gcloud container clusters upgrade gke-prod \
  --region=asia-south1 \
  --master \
  --cluster-version=1.31.1-gke.1000

# ═══ Node pools (Standard) ═══

# Create node pool
gcloud container node-pools create pool-general \
  --cluster=gke-prod-std \
  --region=asia-south1 \
  --machine-type=e2-standard-4 \
  --num-nodes=2 \
  --min-nodes=1 \
  --max-nodes=5 \
  --enable-autoscaling \
  --disk-type=pd-ssd \
  --disk-size=100 \
  --enable-autorepair \
  --enable-autoupgrade \
  --max-surge-upgrade=1 \
  --max-unavailable-upgrade=0 \
  --node-labels=role=general,env=prod \
  --workload-metadata=GKE_METADATA

# Create Spot node pool
gcloud container node-pools create pool-spot \
  --cluster=gke-prod-std \
  --region=asia-south1 \
  --machine-type=e2-standard-4 \
  --spot \
  --num-nodes=0 \
  --min-nodes=0 \
  --max-nodes=10 \
  --enable-autoscaling \
  --node-labels=role=spot \
  --node-taints=cloud.google.com/gke-spot=true:NoSchedule

# Resize node pool
gcloud container clusters resize gke-prod-std \
  --region=asia-south1 \
  --node-pool=pool-general \
  --num-nodes=4

# Upgrade node pool
gcloud container node-pools upgrade pool-general \
  --cluster=gke-prod-std \
  --region=asia-south1

# Delete node pool
gcloud container node-pools delete pool-spot \
  --cluster=gke-prod-std \
  --region=asia-south1

# ═══ kubectl (after get-credentials) ═══

# Verify connection
kubectl get nodes
kubectl get pods -A

# Deploy application
kubectl create namespace backend
kubectl apply -f deployment.yaml -n backend

# HPA
kubectl autoscale deployment web-app \
  -n backend --cpu-percent=70 --min=3 --max=20

# View cluster autoscaler status
kubectl get configmap cluster-autoscaler-status \
  -n kube-system -o yaml

# ═══ Workload Identity setup ═══

# Create GCP SA
gcloud iam service-accounts create sa-app

# Grant roles
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:sa-app@PROJECT.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

# Bind K8s SA to GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  sa-app@PROJECT.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:PROJECT.svc.id.goog[backend/app-sa]"

# Annotate K8s SA
kubectl annotate serviceaccount app-sa -n backend \
  iam.gke.io/gcp-service-account=sa-app@PROJECT.iam.gserviceaccount.com

# Delete cluster
gcloud container clusters delete gke-prod --region=asia-south1
```

---

## Part 10: Real-World Patterns

### Startup

```
GKE Autopilot for a small team:

Cluster: gke-prod (Autopilot, regional)
├── Release channel: Regular
├── Private cluster (public endpoint + authorized networks)
├── Workload Identity enabled
└── Control plane cost: Free (Autopilot)

Workloads:
├── Namespace: frontend (React SSR, 2 replicas)
├── Namespace: backend (API, 3 replicas)
├── Namespace: workers (queue consumers, 2 replicas)
├── HPA: CPU > 70% → scale pods
└── Autopilot: Nodes auto-provisioned for pods

Networking:
├── GKE Ingress → Google HTTPS LB
├── ManagedCertificate for app.startup.com
├── Cloud Armor: Basic WAF rules
└── Direct VPC: Cloud SQL via private IP

Workload Identity:
├── sa-api → Cloud SQL Client + Storage Viewer
├── sa-worker → Pub/Sub Subscriber + Storage Writer
└── No service account keys!

Cost: ~$150-300/month (Autopilot pay-per-pod)
```

### Mid-Size

```
GKE Standard with node pools:

Cluster: gke-prod-std (Standard, regional)
├── Dataplane V2, Gateway API enabled
├── Release channel: Regular
├── Private cluster, authorized networks
└── Cost: $73/mo (regional control plane)

Node pools:
├── pool-general (e2-standard-4, 3-12 per zone, On-Demand)
│   Labels: role=general
│   For: Web services, APIs
│
├── pool-spot (e2-standard-4, 0-10 per zone, Spot)
│   Labels: role=spot, Taint: spot=true:NoSchedule
│   For: Batch, queue workers, non-critical
│
├── pool-highcpu (c2-standard-8, 0-4, On-Demand)
│   Labels: role=compute
│   For: CPU-intensive workloads
│
└── Node Auto-Provisioning: Enabled
    Max CPU: 200, Max Memory: 800 GiB

Namespaces & workloads:
├── frontend: SSR app (3-10 pods, HPA on CPU)
├── backend: 6 microservices (3-20 pods each)
├── workers: Pub/Sub consumers (2-30 pods, KEDA)
├── batch: CronJobs (nightly ETL, scale to zero)
├── monitoring: Prometheus, Grafana
└── ingress: GKE Gateway API

Networking:
├── Gateway API: Multi-host routing
│   app.company.com → frontend
│   api.company.com → api-gateway
├── Internal services: ClusterIP + DNS
├── NetworkPolicies: Namespace isolation
├── Cloud Armor: WAF + DDoS protection
└── Cloud NAT: Egress for private nodes

CI/CD:
├── Cloud Build → Artifact Registry → GKE
├── Config Sync (GitOps): K8s manifests from Git
├── Canary: GKE Gateway traffic splitting
└── Rollback: kubectl rollout undo

Cost: $1,500-4,000/month (with Spot savings)
```

### Enterprise

```
Multi-cluster, multi-region GKE platform:

Region 1 (asia-south1):
├── gke-prod-apps (Standard, regional)
│   ├── pool-web (n2-standard-8, 6-30 On-Demand)
│   ├── pool-api (n2-standard-4, 10-40 On-Demand)
│   ├── pool-spot (n2-standard-4, 0-50 Spot)
│   └── pool-gpu (a2-highgpu-1g, 0-4 On-Demand, ML inference)
│
├── gke-prod-data (Autopilot, regional)
│   ├── Spark jobs on K8s (auto-scale nodes)
│   └── Apache Airflow (Cloud Composer alternatively)

Region 2 (europe-west1):
├── gke-prod-eu (same config as asia-south1)
└── Multi-cluster services: Cross-region service discovery

Fleet management:
├── GKE Hub (Fleet): Manage all clusters centrally
├── Config Sync: GitOps for all clusters from one repo
├── Policy Controller: OPA Gatekeeper policies
│   ├── Require resource limits on all pods
│   ├── Block privileged containers
│   ├── Require approved container registries
│   └── Enforce network policies per namespace
├── Multi-Cluster Ingress: Single global IP → route to nearest
└── Binary Authorization: Only signed images from CI/CD

Security:
├── Private clusters (no public endpoints)
├── Workload Identity everywhere (no SA keys)
├── Shielded nodes + Secure Boot
├── Container-Optimized OS (minimal surface)
├── GKE Sandbox (gVisor — pod-level kernel isolation)
├── NetworkPolicies (default deny, explicit allow)
├── Secret Manager (External Secrets Operator)
├── Artifact Registry: Vulnerability scanning + block critical
├── Binary Authorization: Attestation required
├── Audit: GKE audit logs → Cloud Logging → BigQuery
└── Compliance: PCI DSS, HIPAA, SOC 2

Monitoring:
├── Google Cloud Managed Prometheus
├── GKE dashboards (Cloud Monitoring)
├── Cloud Logging: All pod logs, structured JSON
├── Cloud Trace: Distributed tracing
├── SLOs: Error budgets per service
├── Alerts: PagerDuty integration
└── Cost: GKE cost allocation per namespace/label

Cost: $15,000-60,000/month (multi-cluster, GPU, Spot savings)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Fully managed Kubernetes |
| Modes | Autopilot (Google manages nodes) or Standard (you manage) |
| Control plane | Free (Autopilot, Standard zonal), $73/mo (Standard regional) |
| Node scaling | Cluster Autoscaler, NAP, or Autopilot auto |
| Pod scaling | HPA, VPA, MPA |
| Networking | VPC-native (alias IPs), Dataplane V2 (eBPF) |
| Ingress | GKE Ingress (Google Cloud LB), Gateway API |
| Pod IAM | Workload Identity Federation (K8s SA ↔ GCP SA) |
| Node types | Standard, Spot, GPU, ARM (Tau T2A) |
| Node OS | Container-Optimized OS, Ubuntu, Windows |
| Release channels | Rapid, Regular, Stable, Extended |
| Auto-repair | Yes (detects unhealthy nodes, recreates) |
| Auto-upgrade | Yes (within release channel + maintenance window) |
| AWS equivalent | EKS |
| Azure equivalent | AKS |

---

## Kubernetes Basics for Beginners

```
┌─────────────────────────────────────────────────────────────────────┐
│           KUBERNETES CORE CONCEPTS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Before using GKE, you need to understand 4 basic Kubernetes       │
│ building blocks. Think of them as layers:                          │
│                                                                       │
│ 1. POD (smallest unit)                                              │
│ ├── One or more containers running together                      │
│ ├── Containers in a pod share the same network (localhost)      │
│ ├── A pod gets ONE IP address                                    │
│ ├── Pods are ephemeral — they can be killed and recreated       │
│ ├── You almost never create pods directly                       │
│ └── Think of a pod as a single apartment                        │
│                                                                       │
│ 2. DEPLOYMENT (manages pods)                                       │
│ ├── Tells Kubernetes: "I want 3 copies of this pod running"     │
│ ├── If a pod dies, Deployment creates a new one automatically   │
│ ├── Handles rolling updates (update without downtime)           │
│ ├── Handles rollbacks (go back to previous version)             │
│ ├── You define: container image, replicas, CPU/memory limits    │
│ └── Think of a Deployment as an apartment building              │
│     (manages multiple apartments/pods)                           │
│                                                                       │
│ 3. SERVICE (exposes pods to the network)                           │
│ ├── Pods have random IPs that change when pods restart          │
│ ├── A Service gives your pods a stable IP and DNS name          │
│ ├── Routes traffic to healthy pods automatically (load balance) │
│ ├── Types:                                                       │
│ │   ├── ClusterIP: Internal only (pod-to-pod) — default        │
│ │   ├── NodePort: Expose on each node's port                   │
│ │   └── LoadBalancer: Create a cloud load balancer (public)    │
│ └── Think of a Service as the building's street address         │
│     (people find the building by address, not apt number)       │
│                                                                       │
│ 4. NAMESPACE (logical separation)                                  │
│ ├── Groups related resources together                            │
│ ├── Like folders for your Kubernetes objects                     │
│ ├── Each team or app gets its own namespace                     │
│ ├── Examples: "frontend", "backend", "monitoring"              │
│ └── Default namespace: "default" (don't use for production!)   │
│                                                                       │
│ ── Simple Analogy ──                                               │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ NAMESPACE = City neighborhood ("Downtown", "Uptown")       │  │
│ │   └── DEPLOYMENT = Apartment building                      │  │
│ │         └── POD = One apartment                             │  │
│ │               └── Container = Person living in the apt     │  │
│ │   └── SERVICE = Street address of the building             │  │
│ │         (so visitors know where to go)                     │  │
│ │                                                              │  │
│ │ You tell K8s: "I need 3 apartments (pods) in this building │  │
│ │ (deployment), and here's the address (service) to find them"│  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ How they connect:                                                   │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ User → Service (stable IP) → Pod 1 ┐                       │  │
│ │                              → Pod 2 ├─ Deployment manages │  │
│ │                              → Pod 3 ┘                       │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ You don't need to be a Kubernetes expert for GKE Autopilot!    │
│   Just understand these 4 concepts, write YAML, and deploy.     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Deploying Your First App on GKE

```
┌─────────────────────────────────────────────────────────────────────┐
│           STEP-BY-STEP: FIRST APP ON GKE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Prerequisites:                                                       │
│ ├── GKE cluster created (Autopilot or Standard)                  │
│ ├── gcloud CLI installed                                          │
│ ├── kubectl installed (gcloud components install kubectl)        │
│ └── Connected to cluster:                                         │
│     gcloud container clusters get-credentials gke-prod \        │
│       --region=asia-south1                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 1: Create a Deployment YAML

Save this as `deployment.yaml`:

```yaml
# deployment.yaml — Tells GKE to run 3 copies of your app
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: default
  labels:
    app: hello-app
spec:
  replicas: 3                    # Run 3 pods (copies of your app)
  selector:
    matchLabels:
      app: hello-app             # Deployment manages pods with this label
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
        - name: hello-app
          image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0
          ports:
            - containerPort: 8080  # App listens on port 8080
          resources:
            requests:              # Minimum resources per pod
              cpu: "250m"          # 0.25 CPU cores
              memory: "128Mi"      # 128 MB memory
            limits:                # Maximum resources per pod
              cpu: "500m"
              memory: "256Mi"
```

> ⚡ **Why resource requests/limits?** Autopilot requires them (billing is based on pod resources). Standard mode uses them for scheduling decisions. Always set them!

### Step 2: Create a Service YAML

Save this as `service.yaml`:

```yaml
# service.yaml — Exposes your app to the internet via a Load Balancer
apiVersion: v1
kind: Service
metadata:
  name: hello-app-service
  namespace: default
spec:
  type: LoadBalancer              # Creates a Google Cloud Network LB
  selector:
    app: hello-app                # Route traffic to pods with this label
  ports:
    - protocol: TCP
      port: 80                    # External port (what users access)
      targetPort: 8080            # Pod port (where app listens)
```

> ⚡ `type: LoadBalancer` tells GKE to automatically create a Google Cloud Network Load Balancer with a public IP.

### Step 3: Deploy to GKE

```bash
# Apply the Deployment (creates 3 pods)
kubectl apply -f deployment.yaml

# Apply the Service (creates a load balancer)
kubectl apply -f service.yaml

# Watch pods come up (wait for STATUS: Running)
kubectl get pods -w

# Output:
# NAME                         READY   STATUS    RESTARTS   AGE
# hello-app-6b8d95c4f7-abc12   1/1     Running   0          30s
# hello-app-6b8d95c4f7-def34   1/1     Running   0          30s
# hello-app-6b8d95c4f7-ghi56   1/1     Running   0          30s
```

### Step 4: View Your Running App

```bash
# Get the external IP of the load balancer
# (may take 1-2 minutes to provision)
kubectl get service hello-app-service

# Output:
# NAME                TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)
# hello-app-service   LoadBalancer   10.8.0.42      34.93.120.55    80:31234/TCP

# Visit http://34.93.120.55 in your browser!
# You'll see: "Hello, world! Version: 1.0.0"

# ── Useful commands ──

# View deployment details
kubectl describe deployment hello-app

# View pod logs
kubectl logs -l app=hello-app --tail=50

# Scale to 5 replicas
kubectl scale deployment hello-app --replicas=5

# Update to a new image version (rolling update)
kubectl set image deployment/hello-app \
  hello-app=us-docker.pkg.dev/google-samples/containers/gke/hello-app:2.0

# Watch the rolling update
kubectl rollout status deployment/hello-app

# Rollback if something went wrong
kubectl rollout undo deployment/hello-app

# Clean up (delete everything)
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
```

---

## Console Walkthrough: Managing & Deleting Clusters

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING CLUSTERS FROM CONSOLE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Viewing Workloads ──                                             │
│                                                                       │
│ Console → Kubernetes Engine → Workloads                            │
│ ├── Shows all Deployments, StatefulSets, DaemonSets, Jobs        │
│ ├── Click a workload to see:                                      │
│ │   ├── Pod status (Running, Pending, Failed)                   │
│ │   ├── Events (scheduling, image pull, crashes)                │
│ │   ├── YAML (live configuration)                                │
│ │   ├── Revision history (rollback from here!)                  │
│ │   └── Managed pods list                                        │
│ └── Filter by namespace, cluster, status                          │
│                                                                       │
│ ── Viewing Logs ──                                                  │
│                                                                       │
│ Console → Kubernetes Engine → Workloads → [your workload]        │
│ → Logs tab                                                         │
│ ├── Shows container stdout/stderr                                 │
│ ├── Filter by severity (Error, Warning, Info)                    │
│ ├── Link to Cloud Logging for advanced queries                   │
│ │   resource.type="k8s_container"                                │
│ │   resource.labels.cluster_name="gke-prod"                     │
│ │   resource.labels.namespace_name="backend"                    │
│ └── Or: kubectl logs <pod-name> -n <namespace>                  │
│                                                                       │
│ ── Viewing Services & Ingress ──                                   │
│                                                                       │
│ Console → Kubernetes Engine → Services & Ingress                  │
│ ├── Shows all Services with their type and external IPs          │
│ ├── Shows Ingress resources with load balancer details           │
│ └── Click to see endpoints, port mappings, backend health        │
│                                                                       │
│ ── Adding a Node Pool (Standard mode) ──                           │
│                                                                       │
│ Console → Kubernetes Engine → Clusters → [your cluster]          │
│ → Nodes tab → [+ Add Node Pool]                                   │
│ ├── Configure machine type, disk, autoscaling                    │
│ ├── Set labels and taints                                         │
│ ├── Takes 2-5 minutes to provision nodes                         │
│ └── New pool appears alongside existing pools                     │
│                                                                       │
│ ── Removing a Node Pool (Standard mode) ──                         │
│                                                                       │
│ Console → Kubernetes Engine → Clusters → [your cluster]          │
│ → Nodes tab → [pool name] → [Delete]                              │
│ ├── ⚠️  Pods on those nodes are evicted (moved to other pools)   │
│ ├── If no other pool can fit the pods → pods go Pending!        │
│ ├── Ensure another pool has capacity before deleting             │
│ └── Cannot delete the last node pool in a Standard cluster      │
│                                                                       │
│ ── Deleting a Cluster ──                                           │
│                                                                       │
│ Console → Kubernetes Engine → Clusters → [your cluster]          │
│ → [DELETE] button (top of page)                                    │
│                                                                       │
│ ⚠️  What happens when you delete a cluster:                       │
│ ├── ALL workloads (pods, deployments, services) are destroyed   │
│ ├── ALL node VMs are deleted                                      │
│ ├── Load balancers created by Services/Ingress are deleted      │
│ ├── Persistent disks from PVCs are KEPT by default              │
│ │   (reclaimPolicy: Retain — you must delete manually)          │
│ ├── External static IPs are KEPT (must release manually)        │
│ ├── Container images in Artifact Registry are KEPT              │
│ ├── Cloud Logging logs are KEPT (based on retention settings)   │
│ └── Action is IRREVERSIBLE — there is no undo!                  │
│                                                                       │
│ ⚡ Deletion protection (Terraform/gcloud):                         │
│   If deletion_protection = true, you must disable it first.     │
│   gcloud: --no-deletion-protection flag.                         │
│   This prevents accidental deletion.                             │
│                                                                       │
│ ⚡ Tip: Before deleting, run:                                      │
│   kubectl get svc -A  (check for external LoadBalancers)        │
│   kubectl get pvc -A  (check for persistent volume claims)      │
│   These resources may incur cost even after cluster is deleted. │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Connecting GKE to Cloud SQL

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD SQL AUTH PROXY (SIDECAR PATTERN)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: Your GKE app needs to connect to Cloud SQL.              │
│                                                                       │
│ ❌ Bad approach: Whitelist pod IPs, use public IP, store password │
│   in plain text. IPs change, credentials leak, insecure.        │
│                                                                       │
│ ✅ Best approach: Cloud SQL Auth Proxy as a sidecar container.    │
│                                                                       │
│ How it works:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ Pod                                                          │  │
│ │ ┌──────────────────┐   ┌─────────────────────────┐         │  │
│ │ │ Your App         │   │ Cloud SQL Auth Proxy    │         │  │
│ │ │                  │   │ (sidecar container)     │         │  │
│ │ │ connects to      │──→│ localhost:5432          │         │  │
│ │ │ localhost:5432   │   │                         │         │  │
│ │ └──────────────────┘   │ Handles:               │         │  │
│ │                        │ ├── IAM authentication │         │  │
│ │                        │ ├── TLS encryption     │         │  │
│ │                        │ └── Connection pooling │         │  │
│ │                        │          │              │         │  │
│ │                        └──────────┼──────────────┘         │  │
│ │                                   │                          │  │
│ └───────────────────────────────────┼──────────────────────────┘  │
│                                     │ Encrypted tunnel            │
│                                     ▼                              │
│                              Cloud SQL Instance                   │
│                              (Private IP)                          │
│                                                                       │
│ ⚡ The proxy runs as a second container in the SAME pod.           │
│   Your app connects to localhost — the proxy handles the rest.  │
│   Uses Workload Identity for auth (no passwords in YAML!).      │
│                                                                       │
│ Prerequisites:                                                       │
│ ├── Workload Identity set up (see Part 6)                        │
│ ├── GCP SA has roles/cloudsql.client role                        │
│ ├── Cloud SQL instance connection name:                          │
│ │   PROJECT_ID:REGION:INSTANCE_NAME                              │
│ └── Database password stored in Secret Manager or K8s Secret    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Deployment YAML with Cloud SQL Auth Proxy Sidecar

```yaml
# app-with-cloudsql.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
  namespace: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-api
  template:
    metadata:
      labels:
        app: my-api
    spec:
      serviceAccountName: app-sa       # K8s SA bound to GCP SA via Workload Identity
      containers:
        # ── Your application container ──
        - name: my-api
          image: asia-south1-docker.pkg.dev/my-project/my-repo/my-api:1.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: "127.0.0.1"         # Connect to proxy on localhost
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: "mydb"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials   # Store in K8s Secret or use Secret Manager
                  key: password
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

        # ── Cloud SQL Auth Proxy sidecar ──
        - name: cloud-sql-proxy
          image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.14.1
          args:
            - "--structured-logs"                              # JSON logs
            - "--auto-iam-authn"                               # Use Workload Identity
            - "my-project:asia-south1:my-db-instance"          # Connection name
            # Format: PROJECT_ID:REGION:INSTANCE_NAME
          securityContext:
            runAsNonRoot: true
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
---
# K8s Secret for database credentials
# ⚡ In production, use Secret Manager + External Secrets Operator instead
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: backend
type: Opaque
stringData:
  username: "app_user"
  password: "use-secret-manager-instead"   # Don't hardcode in production!
```

```bash
# Deploy the app with Cloud SQL proxy
kubectl apply -f app-with-cloudsql.yaml

# Verify both containers are running in each pod
kubectl get pods -n backend
# NAME                      READY   STATUS    RESTARTS   AGE
# my-api-7f8b9c6d5-abc12    2/2     Running   0          30s
#                            ^^^ 2/2 means both containers (app + proxy) are running

# Check proxy logs
kubectl logs -n backend -l app=my-api -c cloud-sql-proxy --tail=10
```

> ⚡ **Why a sidecar?** The proxy runs in the same pod as your app, so your app connects to `localhost:5432` — no network hops, no firewall rules, no IP whitelisting. The proxy handles IAM auth, TLS encryption, and automatic reconnection. This is the Google-recommended approach.

---

## What's Next?

In the next chapter, we'll cover Google App Engine — a PaaS for deploying apps without managing servers.

→ Next: [Chapter 19: App Engine](19-app-engine.md)

---

*Last Updated: May 2026*
