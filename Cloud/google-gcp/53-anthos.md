# Chapter 53: Google Anthos

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Anthos Fundamentals](#part-1-anthos-fundamentals)
- [Part 2: GKE Hub (Fleet) — Cluster Registration](#part-2-gke-hub-fleet--cluster-registration)
- [Part 3: Config Sync (GitOps)](#part-3-config-sync-gitops)
- [Part 4: Policy Controller (OPA Gatekeeper)](#part-4-policy-controller-opa-gatekeeper)
- [Part 5: Anthos Service Mesh (ASM)](#part-5-anthos-service-mesh-asm)
- [Part 6: Multi-Cluster Services & Ingress](#part-6-multi-cluster-services--ingress)
- [Part 7: Anthos on Bare Metal / VMware](#part-7-anthos-on-bare-metal--vmware)
- [Part 8: Terraform & gcloud CLI](#part-8-terraform--gcloud-cli)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Anthos is Google Cloud's platform for managing Kubernetes clusters and workloads across multiple environments — GKE on Google Cloud, on-premises data centers, AWS, and Azure. It provides a single control plane for fleet management, configuration as code (GitOps), service mesh, and policy enforcement — regardless of where your clusters run.

```
What you'll learn:
├── Anthos Fundamentals
│   ├── What & why (multi-cloud/hybrid Kubernetes management)
│   ├── Anthos components overview
│   └── When to use Anthos vs just GKE
├── Anthos Architecture
│   ├── GKE Hub (Fleet)
│   ├── Attached clusters (EKS, AKS, on-prem)
│   ├── Anthos on Bare Metal / VMware
│   └── Connect gateway
├── Config Sync (GitOps)
│   ├── What & how (repo → clusters)
│   ├── Source types (Git, OCI, Helm)
│   └── Multi-repo mode
├── Policy Controller (OPA Gatekeeper)
│   ├── Constraint templates
│   ├── Pre-built policy bundles
│   └── Audit mode vs Enforce mode
├── Anthos Service Mesh (ASM / Istio)
│   ├── Traffic management
│   ├── Security (mTLS, authorization)
│   └── Observability
├── Multi-Cluster Services & Ingress
├── Config Controller (managed GCP resources)
├── Terraform & gcloud CLI
└── Real-world patterns
```

---

## Part 1: Anthos Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           ANTHOS CONCEPT                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Anthos?                                                     │
│ ├── Platform for managing K8s clusters EVERYWHERE:               │
│ │   ├── GKE on Google Cloud                                     │
│ │   ├── On-premises (VMware vSphere or bare metal)             │
│ │   ├── AWS (EKS or Anthos-managed)                             │
│ │   ├── Azure (AKS or Anthos-managed)                          │
│ │   └── Any CNCF-conformant K8s cluster                        │
│ ├── Single pane of glass: Manage all from Google Cloud Console  │
│ ├── Consistent policies, security, and config across all        │
│ └── Not a separate product — it's GKE Enterprise tier           │
│                                                                       │
│ ⚡ Naming update (2024):                                             │
│   "Anthos" brand → now called "GKE Enterprise"                  │
│   Same features, rebranded under GKE umbrella.                  │
│   Anthos = GKE Enterprise = same thing.                         │
│                                                                       │
│ Why Anthos / GKE Enterprise?                                       │
│ ├── Hybrid cloud: Apps in both cloud AND on-premises            │
│ ├── Multi-cloud: Apps on GCP + AWS + Azure                     │
│ ├── Consistency: Same tools, policies, mesh everywhere         │
│ ├── Migration: Gradually move workloads to cloud               │
│ ├── Compliance: Data residency (must stay on-prem)             │
│ └── Avoid lock-in: K8s is portable, Anthos adds consistency   │
│                                                                       │
│ Components:                                                          │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ GKE Hub (Fleet)                                              │  │
│ │ ├── Register clusters (GKE, EKS, AKS, on-prem)            │  │
│ │ ├── Central inventory of all clusters                       │  │
│ │ └── Enable features per fleet/cluster                       │  │
│ │                                                              │  │
│ │ Config Sync                                                  │  │
│ │ ├── GitOps: K8s manifests in Git → auto-deployed           │  │
│ │ ├── Multi-cluster: Same config → all clusters              │  │
│ │ └── Drift detection: Auto-reconcile deviations             │  │
│ │                                                              │  │
│ │ Policy Controller                                            │  │
│ │ ├── OPA Gatekeeper (policy as code)                        │  │
│ │ ├── Enforce: No privileged pods, required labels, etc.     │  │
│ │ └── Audit: Report violations without blocking              │  │
│ │                                                              │  │
│ │ Anthos Service Mesh (ASM)                                    │  │
│ │ ├── Managed Istio service mesh                             │  │
│ │ ├── mTLS between all services (zero-trust)                 │  │
│ │ ├── Traffic management (canary, fault injection)           │  │
│ │ └── Observability (traces, metrics, topology)              │  │
│ │                                                              │  │
│ │ Multi-Cluster Ingress                                        │  │
│ │ ├── Single global IP → route to nearest cluster            │  │
│ │ └── Google Cloud Load Balancer across clusters             │  │
│ │                                                              │  │
│ │ Connect Gateway                                              │  │
│ │ ├── kubectl access to any registered cluster               │  │
│ │ └── Through Google Cloud (no direct network needed)        │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Pricing (GKE Enterprise):                                           │
│ ├── $0.040/vCPU/hour for all managed cluster vCPUs            │
│ │   (100 vCPU cluster = $0.04 × 100 × 730 hr = $2,920/mo)   │
│ ├── Includes: Config Sync, Policy Controller, ASM, Fleet     │
│ ├── On-prem/attached: Same per-vCPU pricing                  │
│ └── ⚡ GKE Standard tier: Doesn't include these features       │
│                                                                       │
│ When to use Anthos/GKE Enterprise:                                  │
│ ├── ✅ Multi-cloud (apps on GCP + AWS + Azure)                    │
│ ├── ✅ Hybrid (cloud + on-premises)                                │
│ ├── ✅ Enterprise governance (policy, compliance, audit)          │
│ ├── ✅ Service mesh needed                                         │
│ ├── ❌ Single-cloud, single-cluster (just use GKE Standard)     │
│ ├── ❌ Small team, few services (overkill)                        │
│ └── ❌ Non-K8s workloads (Anthos is K8s-focused)                 │
│                                                                       │
│ ⚡ AWS equivalent: No direct equivalent (EKS Anywhere is closest) │
│ ⚡ Azure equivalent: Azure Arc (similar multi-cloud management)    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: GKE Hub (Fleet) — Cluster Registration

```
┌─────────────────────────────────────────────────────────────────────┐
│           GKE HUB / FLEET                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Fleet = Collection of clusters managed together                    │
│                                                                       │
│ Register clusters:                                                   │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Fleet: prod-fleet                                            │  │
│ │ ├── gke-prod-asia (GKE, asia-south1) — auto-registered     │  │
│ │ ├── gke-prod-eu (GKE, europe-west1) — auto-registered      │  │
│ │ ├── eks-prod (EKS, ap-south-1) — attached                  │  │
│ │ ├── aks-prod (AKS, centralindia) — attached                │  │
│ │ └── on-prem-dc1 (bare metal, Mumbai DC) — attached         │  │
│ │                                                              │  │
│ │ All clusters:                                                │  │
│ │ ├── Same Config Sync repo (consistent config)              │  │
│ │ ├── Same Policy Controller (same guardrails)               │  │
│ │ ├── Same service mesh (mTLS everywhere)                    │  │
│ │ └── Managed from one console!                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Console → Kubernetes Engine → Fleets → Register cluster          │
│                                                                       │
│ Register GKE cluster (automatic):                                  │
│ ⚡ GKE clusters in same project auto-register to fleet.            │
│                                                                       │
│ Register external cluster (EKS, AKS, on-prem):                   │
│ # 1. Install Connect Agent on the cluster                        │
│ gcloud container fleet memberships register eks-prod \            │
│   --context=arn:aws:eks:ap-south-1:123456789:cluster/prod \     │
│   --kubeconfig=~/.kube/eks-config \                               │
│   --enable-workload-identity                                       │
│                                                                       │
│ # 2. Connect Agent runs in the cluster → calls home to GCP      │
│ # No inbound ports needed! Agent initiates outbound connection. │
│                                                                       │
│ Connect Gateway:                                                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Your laptop → gcloud → Connect Gateway → Any cluster       │  │
│ │                                                              │  │
│ │ # Access any fleet cluster with kubectl                    │  │
│ │ gcloud container fleet memberships get-credentials eks-prod│  │
│ │ kubectl get pods -A  # Works! Even for EKS/AKS/on-prem   │  │
│ │                                                              │  │
│ │ ⚡ No VPN, no direct network connectivity needed!           │  │
│ │   Connect Agent tunnels through Google Cloud.              │  │
│ │   IAM controls who can access which cluster.              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Fleet features (enable per fleet):                                  │
│ ├── Config Sync: GitOps across fleet                             │
│ ├── Policy Controller: Guardrails across fleet                  │
│ ├── Service Mesh: Istio across fleet                            │
│ ├── Multi-Cluster Services: Cross-cluster service discovery     │
│ ├── Multi-Cluster Ingress: Global load balancing                │
│ └── Fleet logging/monitoring: Centralized observability        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Config Sync (GitOps)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CONFIG SYNC                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Git repo = source of truth → auto-deployed to clusters      │
│                                                                       │
│ How it works:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Git repo (GitHub/GitLab/Cloud Source Repos)                 │  │
│ │ ├── namespaces/backend/deployment.yaml                     │  │
│ │ ├── namespaces/backend/service.yaml                        │  │
│ │ ├── policies/require-labels.yaml                           │  │
│ │ └── cluster/rbac.yaml                                      │  │
│ │                                                              │  │
│ │     ↓ Config Sync watches (every 15s by default)           │  │
│ │                                                              │  │
│ │ All fleet clusters:                                         │  │
│ │ ├── gke-prod-asia: Synced ✅                                │  │
│ │ ├── gke-prod-eu: Synced ✅                                  │  │
│ │ ├── eks-prod: Synced ✅                                     │  │
│ │ └── aks-prod: Synced ✅                                     │  │
│ │                                                              │  │
│ │ Drift detected?                                              │  │
│ │ Someone manually edited a resource → Config Sync reverts!  │  │
│ │ Git is the single source of truth.                         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Source types:                                                        │
│ ├── Git repository (GitHub, GitLab, Bitbucket, CSR)             │
│ ├── OCI image (Artifact Registry, Docker Hub)                   │
│ └── Helm chart (Helm repository)                                 │
│                                                                       │
│ Multi-repo mode:                                                     │
│ ├── Root repo: Cluster-level config (namespaces, RBAC, policies)│
│ ├── Namespace repos: Per-team config (their own deployments)    │
│ └── Separation of concerns: Platform team vs app teams          │
│                                                                       │
│ Enable Config Sync:                                                 │
│ Console → Fleet → Features → Config Sync → Enable               │
│                                                                       │
│ # Or via gcloud:                                                    │
│ gcloud container fleet config-management apply \                  │
│   --membership=gke-prod-asia \                                     │
│   --config=config-sync.yaml                                        │
│                                                                       │
│ # config-sync.yaml:                                                 │
│ applySpecVersion: 1                                                 │
│ spec:                                                                │
│   configSync:                                                        │
│     enabled: true                                                    │
│     sourceFormat: unstructured                                      │
│     git:                                                             │
│       syncRepo: https://github.com/org/k8s-config                │
│       syncBranch: main                                              │
│       secretType: token                                             │
│       policyDir: clusters/prod                                    │
│     preventDrift: true                                              │
│     sourceType: git                                                  │
│                                                                       │
│ Repo structure (unstructured):                                      │
│ k8s-config/                                                          │
│ ├── clusters/                                                       │
│ │   ├── prod/                                                      │
│ │   │   ├── namespaces.yaml                                      │
│ │   │   ├── rbac.yaml                                             │
│ │   │   └── network-policies.yaml                                │
│ │   └── staging/                                                   │
│ │       └── ...                                                    │
│ ├── apps/                                                           │
│ │   ├── frontend/                                                  │
│ │   │   ├── deployment.yaml                                      │
│ │   │   ├── service.yaml                                          │
│ │   │   └── ingress.yaml                                         │
│ │   └── backend/                                                   │
│ │       └── ...                                                    │
│ └── policies/                                                       │
│     ├── require-labels.yaml                                        │
│     └── no-privileged.yaml                                         │
│                                                                       │
│ ⚡ Benefit: Review K8s changes via Pull Requests!                   │
│   PR → review → merge → auto-deploy to all clusters.            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Policy Controller (OPA Gatekeeper)

```
┌─────────────────────────────────────────────────────────────────────┐
│           POLICY CONTROLLER                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Enforce policies on all K8s resources across fleet.          │
│ How: OPA Gatekeeper (admission controller) + pre-built bundles.   │
│                                                                       │
│ Flow:                                                                │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ kubectl apply -f pod.yaml                                    │  │
│ │     ↓                                                        │  │
│ │ K8s API → Admission Webhook → Policy Controller            │  │
│ │     ↓                                                        │  │
│ │ Evaluate constraints:                                        │  │
│ │ ├── Has required labels? ✅                                 │  │
│ │ ├── Has resource limits? ✅                                 │  │
│ │ ├── Image from approved registry? ✅                        │  │
│ │ ├── No privileged container? ✅                             │  │
│ │ └── All pass → ADMIT                                       │  │
│ │                                                              │  │
│ │ If any fails:                                                │  │
│ │ ├── Enforce mode: REJECT (pod not created)                 │  │
│ │ └── Audit mode: ALLOW but report violation                 │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Pre-built policy bundles:                                           │
│ ├── Pod Security Standards (PSS):                                │
│ │   ├── No privileged containers                                │
│ │   ├── No host networking/PID/IPC                             │
│ │   ├── Read-only root filesystem                              │
│ │   ├── No privilege escalation                                 │
│ │   └── Run as non-root                                        │
│ │                                                                 │
│ ├── CIS Kubernetes Benchmark:                                    │
│ │   ├── RBAC settings                                           │
│ │   ├── Network policies required                               │
│ │   └── Audit logging enabled                                  │
│ │                                                                 │
│ └── Custom constraints:                                           │
│     ├── Only images from gcr.io/my-project                     │
│     ├── All deployments must have PodDisruptionBudget          │
│     ├── Namespace must have resource quotas                    │
│     └── Labels: team, env required on all resources            │
│                                                                       │
│ Example constraint (require labels):                               │
│ apiVersion: constraints.gatekeeper.sh/v1beta1                     │
│ kind: K8sRequiredLabels                                            │
│ metadata:                                                            │
│   name: require-team-label                                         │
│ spec:                                                                │
│   enforcementAction: deny    # or dryrun (audit only)           │
│   match:                                                             │
│     kinds:                                                           │
│       - apiGroups: ["apps"]                                       │
│         kinds: ["Deployment"]                                     │
│   parameters:                                                        │
│     labels:                                                          │
│       - key: team                                                  │
│       - key: env                                                   │
│                                                                       │
│ Example (restrict registries):                                      │
│ apiVersion: constraints.gatekeeper.sh/v1beta1                     │
│ kind: K8sAllowedRepos                                              │
│ metadata:                                                            │
│   name: allowed-repos                                               │
│ spec:                                                                │
│   enforcementAction: deny                                          │
│   match:                                                             │
│     kinds:                                                           │
│       - apiGroups: [""]                                             │
│         kinds: ["Pod"]                                              │
│   parameters:                                                        │
│     repos:                                                           │
│       - "gcr.io/my-project/"                                      │
│       - "asia-south1-docker.pkg.dev/my-project/"                 │
│                                                                       │
│ ⚡ Policies deployed via Config Sync → all clusters get them!      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Anthos Service Mesh (ASM)

```
┌─────────────────────────────────────────────────────────────────────┐
│           ANTHOS SERVICE MESH                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed Istio service mesh across all fleet clusters.        │
│                                                                       │
│ Features:                                                            │
│ ├── mTLS: All service-to-service traffic encrypted automatically│
│ ├── Traffic management: Canary, A/B, fault injection, retries  │
│ ├── Authorization policies: Which service can call which       │
│ ├── Observability: Topology map, traces, metrics (golden signals)│
│ └── Multi-cluster mesh: Services across clusters communicate   │
│                                                                       │
│ Architecture:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Pod A (frontend)          Pod B (backend)                   │  │
│ │ ┌─────────────────┐      ┌─────────────────┐              │  │
│ │ │ App container   │      │ App container   │              │  │
│ │ │ (React SSR)     │      │ (Node.js API)   │              │  │
│ │ │                 │      │                 │              │  │
│ │ │ Envoy sidecar   │──────│ Envoy sidecar   │              │  │
│ │ │ (injected auto) │ mTLS │ (injected auto) │              │  │
│ │ └─────────────────┘      └─────────────────┘              │  │
│ │                                                              │  │
│ │ Envoy sidecars handle:                                      │  │
│ │ ├── TLS termination + mutual authentication               │  │
│ │ ├── Load balancing (round-robin, least-conn, random)      │  │
│ │ ├── Retries, timeouts, circuit breaking                    │  │
│ │ ├── Metrics collection (Prometheus format)                 │  │
│ │ ├── Distributed tracing (propagate trace headers)          │  │
│ │ └── Authorization (allow/deny based on identity)          │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Traffic management:                                                  │
│ # Canary deployment (90/10 traffic split)                         │
│ apiVersion: networking.istio.io/v1beta1                            │
│ kind: VirtualService                                                │
│ metadata:                                                            │
│   name: backend-vs                                                  │
│ spec:                                                                │
│   hosts: [backend]                                                  │
│   http:                                                              │
│     - route:                                                        │
│         - destination:                                               │
│             host: backend                                            │
│             subset: stable                                           │
│           weight: 90                                                 │
│         - destination:                                               │
│             host: backend                                            │
│             subset: canary                                           │
│           weight: 10                                                 │
│                                                                       │
│ Authorization policy (zero-trust):                                  │
│ # Only frontend can call backend                                   │
│ apiVersion: security.istio.io/v1beta1                              │
│ kind: AuthorizationPolicy                                          │
│ metadata:                                                            │
│   name: backend-authz                                               │
│   namespace: backend                                                │
│ spec:                                                                │
│   selector:                                                          │
│     matchLabels:                                                     │
│       app: backend-api                                              │
│   rules:                                                             │
│     - from:                                                          │
│         - source:                                                    │
│             principals:                                              │
│               - "cluster.local/ns/frontend/sa/frontend-sa"        │
│       to:                                                            │
│         - operation:                                                 │
│             methods: ["GET", "POST"]                                │
│             paths: ["/api/*"]                                       │
│                                                                       │
│ Observability (Console → Anthos → Service Mesh):                  │
│ ├── Topology map: Visual graph of all service communications   │
│ ├── Golden signals per service: Latency, Traffic, Errors, Sat  │
│ ├── SLOs: Define and track per-service error budgets          │
│ └── Traces: End-to-end request flow (Cloud Trace integration) │
│                                                                       │
│ Enable ASM:                                                          │
│ gcloud container fleet mesh enable                                 │
│ gcloud container fleet mesh update \                               │
│   --management automatic \                                          │
│   --memberships gke-prod-asia,gke-prod-eu                         │
│ ⚡ "automatic" = Google manages Istio control plane for you.       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Multi-Cluster Services & Ingress

```
┌─────────────────────────────────────────────────────────────────────┐
│           MULTI-CLUSTER                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Multi-Cluster Services (MCS):                                      │
│ ├── Services in one cluster → discoverable from other clusters  │
│ ├── Export a service → other clusters can call it by DNS        │
│ ├── Automatic load balancing across clusters                    │
│ └── Same as calling a local service (transparent!)             │
│                                                                       │
│ # Export service from cluster A                                    │
│ apiVersion: net.gke.io/v1                                          │
│ kind: ServiceExport                                                 │
│ metadata:                                                            │
│   name: backend-api                                                 │
│   namespace: backend                                                │
│                                                                       │
│ # In cluster B, call it:                                           │
│ # backend-api.backend.svc.clusterset.local                        │
│ # → Routed to cluster A's backend-api (or both if exported)    │
│                                                                       │
│ Multi-Cluster Ingress:                                              │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ User (India) ──→ Google Cloud LB (global anycast IP)       │  │
│ │                      ↓ nearest healthy cluster              │  │
│ │              ┌───────┴───────┐                              │  │
│ │              ↓               ↓                              │  │
│ │     gke-prod-asia      gke-prod-eu                         │  │
│ │     (asia-south1)      (europe-west1)                      │  │
│ │                                                              │  │
│ │ Config:                                                      │  │
│ │ apiVersion: networking.gke.io/v1                            │  │
│ │ kind: MultiClusterIngress                                   │  │
│ │ metadata:                                                    │  │
│ │   name: global-ingress                                      │  │
│ │   namespace: frontend                                       │  │
│ │ spec:                                                        │  │
│ │   template:                                                  │  │
│ │     spec:                                                    │  │
│ │       backend:                                               │  │
│ │         serviceName: web-app                                │  │
│ │         servicePort: 80                                     │  │
│ │                                                              │  │
│ │ ⚡ Google handles: Global LB, health checks, failover.      │  │
│ │   If asia-south1 cluster unhealthy → traffic to eu.        │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Anthos on Bare Metal / VMware

```
┌─────────────────────────────────────────────────────────────────────┐
│           ON-PREMISES CLUSTERS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Anthos clusters on bare metal:                                     │
│ ├── Run K8s on your own servers (no VMware needed!)             │
│ ├── Google provides the K8s distribution + management          │
│ ├── Supported OS: Ubuntu, RHEL, CentOS                          │
│ ├── Networking: Bundled LB (MetalLB) or manual                 │
│ ├── Storage: Local PVs, external CSI drivers                   │
│ └── Upgrades: Managed by Anthos (gcloud initiated)             │
│                                                                       │
│ Anthos clusters on VMware:                                         │
│ ├── Run K8s on vSphere (existing VMware infrastructure)        │
│ ├── Creates VMs in vSphere for K8s nodes                       │
│ ├── Integrates with vCenter, VSAN, NSX-T                       │
│ └── Admin cluster + user clusters model                         │
│                                                                       │
│ Architecture (on-prem):                                             │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Data Center                                                  │  │
│ │ ├── Admin workstation (manage clusters)                     │  │
│ │ ├── Admin cluster (manages user clusters)                   │  │
│ │ │   └── Runs: Cluster API, Connect Agent, logging/mon     │  │
│ │ └── User cluster(s) (run workloads)                        │  │
│ │     ├── Node 1 (control plane + worker)                    │  │
│ │     ├── Node 2 (worker)                                     │  │
│ │     └── Node 3 (worker)                                     │  │
│ │                                                              │  │
│ │ Connect Agent → Google Cloud (outbound HTTPS only)         │  │
│ │ ├── Fleet registration                                      │  │
│ │ ├── Config Sync (pull from Git)                            │  │
│ │ ├── Policy Controller (pull policies)                      │  │
│ │ ├── Logging → Cloud Logging                               │  │
│ │ └── Monitoring → Cloud Monitoring                          │  │
│ │                                                              │  │
│ │ ⚡ No inbound ports needed from Google Cloud!                │  │
│ │   Agent calls out to Google (443 only).                    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Use cases for on-prem:                                              │
│ ├── Data sovereignty (data must stay in-country/on-prem)       │
│ ├── Latency-sensitive (edge locations, factories)               │
│ ├── Existing hardware investment (use what you have)           │
│ ├── Air-gapped environments (with limited connectivity)       │
│ └── Gradual cloud migration (run same platform everywhere)     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform & gcloud CLI

### Terraform

```hcl
# Enable GKE Enterprise (Anthos) on the project
resource "google_project_service" "anthos" {
  for_each = toset([
    "anthos.googleapis.com",
    "gkehub.googleapis.com",
    "anthosconfigmanagement.googleapis.com",
    "meshconfig.googleapis.com",
    "multiclusteringress.googleapis.com",
    "multiclusterservicediscovery.googleapis.com",
  ])

  project = var.project_id
  service = each.value
}

# Fleet membership (GKE clusters auto-register)
resource "google_gke_hub_membership" "gke_asia" {
  membership_id = "gke-prod-asia"
  project       = var.project_id

  endpoint {
    gke_cluster {
      resource_link = google_container_cluster.asia.id
    }
  }
}

resource "google_gke_hub_membership" "gke_eu" {
  membership_id = "gke-prod-eu"
  project       = var.project_id

  endpoint {
    gke_cluster {
      resource_link = google_container_cluster.eu.id
    }
  }
}

# Config Sync feature
resource "google_gke_hub_feature" "configmanagement" {
  name     = "configmanagement"
  location = "global"
  project  = var.project_id
}

resource "google_gke_hub_feature_membership" "config_sync_asia" {
  location   = "global"
  feature    = google_gke_hub_feature.configmanagement.name
  membership = google_gke_hub_membership.gke_asia.membership_id
  project    = var.project_id

  configmanagement {
    version = "1.18.0"

    config_sync {
      source_format = "unstructured"

      git {
        sync_repo   = "https://github.com/org/k8s-config"
        sync_branch = "main"
        policy_dir  = "clusters/prod"
        secret_type = "token"
      }

      prevent_drift = true
    }

    policy_controller {
      enabled                    = true
      template_library_installed = true
      audit_interval_seconds     = 60
    }
  }
}

# Service Mesh feature
resource "google_gke_hub_feature" "mesh" {
  name     = "servicemesh"
  location = "global"
  project  = var.project_id
}

resource "google_gke_hub_feature_membership" "mesh_asia" {
  location   = "global"
  feature    = google_gke_hub_feature.mesh.name
  membership = google_gke_hub_membership.gke_asia.membership_id
  project    = var.project_id

  mesh {
    management = "MANAGEMENT_AUTOMATIC"
  }
}

# Multi-Cluster Ingress feature
resource "google_gke_hub_feature" "mci" {
  name     = "multiclusteringress"
  location = "global"
  project  = var.project_id

  spec {
    multiclusteringress {
      config_membership = google_gke_hub_membership.gke_asia.id
    }
  }
}
```

### gcloud CLI

```bash
# ═══ Fleet management ═══

# List fleet memberships
gcloud container fleet memberships list

# Register GKE cluster to fleet
gcloud container fleet memberships register gke-prod-asia \
  --gke-cluster=asia-south1/gke-prod-asia \
  --enable-workload-identity

# Register non-GKE cluster (EKS, AKS, on-prem)
gcloud container fleet memberships register eks-prod \
  --context=arn:aws:eks:... \
  --kubeconfig=~/.kube/eks-config \
  --enable-workload-identity

# Get credentials for any fleet cluster
gcloud container fleet memberships get-credentials gke-prod-asia
kubectl get pods -A  # Works!

# ═══ Config Sync ═══

# Enable Config Management feature
gcloud container fleet config-management enable

# Apply Config Sync to a cluster
gcloud container fleet config-management apply \
  --membership=gke-prod-asia \
  --config=config-sync-config.yaml

# Check sync status
gcloud container fleet config-management status

# ═══ Policy Controller ═══

# View policy violations
gcloud container fleet config-management status \
  --format="table(name,policy_controller_state)"

# Install policy bundle
kubectl apply -f \
  https://raw.githubusercontent.com/GoogleCloudPlatform/\
  acm-policy-controller-library/main/bundles/\
  pod-security-standards/restricted.yaml

# ═══ Service Mesh ═══

# Enable mesh feature
gcloud container fleet mesh enable

# Set automatic management
gcloud container fleet mesh update \
  --management automatic \
  --memberships gke-prod-asia,gke-prod-eu

# Check mesh status
gcloud container fleet mesh describe

# ═══ Multi-Cluster Ingress ═══

# Enable MCI
gcloud container fleet ingress enable \
  --config-membership=gke-prod-asia

# Check MCI status
gcloud container fleet ingress describe

# ═══ Anthos on bare metal ═══

# Create bare metal cluster
gcloud container bare-metal clusters create on-prem-prod \
  --project=my-project \
  --location=asia-south1 \
  --admin-cluster-membership=admin-cluster \
  --version=1.30.0 \
  --control-plane-node-configs='node-ip=10.0.0.10' \
  --control-plane-vip=10.0.0.100 \
  --ingress-vip=10.0.0.101 \
  --metal-lb-address-pools='pool1=10.0.0.200-10.0.0.220'
```

---

## Part 9: Real-World Patterns

### Startup (Probably Not Anthos)

```
⚡ Startups usually DON'T need Anthos/GKE Enterprise.
   Use regular GKE (Autopilot or Standard).
   Anthos is for multi-cloud/hybrid/enterprise governance.

When a startup might use it:
├── FinTech with data residency requirements
│   (some data must stay on-prem or in specific regions)
├── Gaming company with edge servers
│   (on-prem + cloud fleet)
└── Regulated industry (healthcare, banking)
    (compliance requires on-prem + audit)

Cost: $2,920/mo per 100 vCPUs — expensive for startups!
```

### Mid-Size

```
Hybrid cloud (GKE + on-premises):

Fleet:
├── gke-prod (GKE, asia-south1) — main workloads
├── on-prem-dc1 (bare metal, Mumbai DC) — sensitive data
└── Fleet features: Config Sync, Policy Controller

Why hybrid?
├── Banking data must stay on-prem (RBI regulations)
├── Customer-facing APIs in cloud (scale, global)
├── On-prem: Core banking, ledger, PII processing
└── Cloud: Mobile API, analytics, ML models

Config Sync:
├── Root repo: Namespaces, RBAC, network policies
├── shared-policies/: Applied to ALL clusters
│   ├── require-labels.yaml
│   ├── no-privileged.yaml
│   └── allowed-registries.yaml
└── Per-cluster dirs: Cluster-specific config

Policy Controller:
├── Enforce: Only internal registry images
├── Enforce: All pods must have resource limits
├── Enforce: No external load balancers on on-prem cluster
├── Audit: Pod security standards compliance
└── Dashboard: View violations across fleet

Observability:
├── Cloud Logging + Cloud Monitoring (all clusters)
├── Connect Agent: Ships logs from on-prem → GCP
├── Single dashboard: All clusters, all metrics
└── Alerts: Same alerts regardless of cluster location

Cost: ~$5,000-15,000/month (GKE Enterprise)
```

### Enterprise

```
Multi-cloud platform (GCP + AWS + Azure + on-prem):

Fleet:
├── gke-prod-asia (GKE, asia-south1) — primary
├── gke-prod-eu (GKE, europe-west1) — EU data residency
├── eks-prod-us (EKS, us-east-1) — US customers
├── aks-prod-uk (AKS, UK South) — UK regulations
├── on-prem-mumbai (bare metal) — core banking
├── on-prem-delhi (bare metal) — disaster recovery
└── edge-locations (3 small clusters) — low latency

Config Sync (GitOps):
├── Platform repo: Fleet-wide config (all clusters)
│   ├── namespaces/
│   ├── rbac/
│   ├── network-policies/
│   └── policy-constraints/
├── App repos: Per-team (separate repos, multi-repo mode)
│   ├── team-payments/ → their namespace manifests
│   ├── team-mobile/ → their namespace manifests
│   └── team-data/ → their namespace manifests
├── Deployment: PR → Review → Merge → All clusters sync
└── Drift prevention: Manual changes auto-reverted

Policy Controller (fleet-wide):
├── CIS Kubernetes Benchmark (full compliance)
├── Pod Security Standards (restricted profile)
├── Custom:
│   ├── Only approved container registries
│   ├── No public load balancers (except designated ingress)
│   ├── All containers: Non-root, read-only root FS
│   ├── Require pod disruption budgets
│   ├── Maximum resource limits (cost control)
│   └── Required annotations: team, cost-center, SLA
├── Audit dashboard: Fleet-wide compliance percentage
└── Exception process: Specific namespaces can be exempted

Service Mesh (ASM):
├── mTLS everywhere (zero-trust networking)
├── Service-to-service authorization policies
├── Traffic management: Canary deployments
├── Cross-cluster service discovery (MCS)
├── Observability: Golden signals per service
├── SLOs per service (error budget tracking)
└── Multi-cluster mesh: Services span GKE + on-prem

Multi-Cluster Ingress:
├── Global anycast IP: app.enterprise.com
├── Routes to nearest healthy cluster
├── Failover: asia down → route to EU
├── Cloud Armor: WAF rules at the edge
├── Cloud CDN: Static content caching
└── Managed certificates: Auto-renewed

Security:
├── Binary Authorization: Only signed images deploy
├── Workload Identity: All pods use GCP/AWS/Azure identities
├── Private clusters everywhere
├── Network policies: Default deny, explicit allow
├── Secret Manager: All secrets (synced via External Secrets)
├── Audit: All API server calls → central SIEM
├── Compliance: PCI DSS, HIPAA, SOC 2, ISO 27001
└── Incident: PagerDuty integration, runbooks

Cost: $30,000-100,000/month (GKE Enterprise across fleet)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Multi-cloud/hybrid Kubernetes management platform |
| New name | GKE Enterprise (Anthos is legacy brand) |
| Supported clusters | GKE, EKS, AKS, on-prem (bare metal/VMware) |
| Fleet | Central inventory of all clusters |
| Config Sync | GitOps (Git → clusters, drift prevention) |
| Policy Controller | OPA Gatekeeper (admit/deny K8s resources) |
| Service Mesh (ASM) | Managed Istio (mTLS, traffic, observability) |
| Multi-Cluster Ingress | Global LB across clusters |
| Multi-Cluster Services | Cross-cluster service discovery |
| Connect Gateway | kubectl access to any fleet cluster |
| On-prem | Bare metal or VMware (agent calls out to GCP) |
| Pricing | $0.04/vCPU/hour (all managed cluster vCPUs) |
| AWS equivalent | EKS Anywhere (partial), no full equivalent |
| Azure equivalent | Azure Arc |
| Best for | Enterprise multi-cloud/hybrid, governance |
| Overkill for | Single cloud, small teams, non-K8s |

---

## When to Consider Anthos (Beginner Guide)

### The Simple Decision Guide

Anthos (now GKE Enterprise) is powerful but not for everyone. Here's a straightforward guide to help you decide:

```
┌──────────────────────────────────────────────────────────────┐
│         DO I NEED ANTHOS?                                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  You probably DON'T need Anthos if:                           │
│  ─────────────────────────────────                            │
│  ❌ You're a small team (< 20 developers)                     │
│  ❌ You only use GCP (no AWS, Azure, or on-prem)              │
│  ❌ You have 1-3 GKE clusters                                 │
│  ❌ You don't need a service mesh                             │
│  ❌ You don't have strict multi-cloud compliance requirements │
│  ❌ Your budget is tight (Anthos costs $0.04/vCPU/hour)       │
│                                                                │
│  → Just use GKE Standard. It's simpler and cheaper.          │
│                                                                │
│  ───────────────────────────────────────────────              │
│                                                                │
│  Consider Anthos / GKE Enterprise if:                         │
│  ─────────────────────────────────────                        │
│  ✅ You run K8s on MULTIPLE clouds (GCP + AWS + Azure)        │
│  ✅ You have on-premises data centers running Kubernetes      │
│  ✅ You need consistent policies across 10+ clusters          │
│  ✅ You need service mesh (mTLS, traffic management)          │
│  ✅ You need GitOps at scale (Config Sync)                    │
│  ✅ Compliance requires centralized policy enforcement        │
│  ✅ You're migrating workloads from on-prem to cloud          │
│                                                                │
│  → Anthos gives you ONE control plane for everything.        │
│                                                                │
│  ───────────────────────────────────────────────              │
│                                                                │
│  Cost reality check:                                           │
│  • 50 vCPU cluster: ~$1,460/month just for Anthos fee        │
│  • 200 vCPU across 5 clusters: ~$5,840/month                 │
│  • This is ON TOP of regular compute costs                    │
│  • Worth it for enterprise-scale. Overkill for startups.     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Decision Flowchart

```
  Are you using Kubernetes?
  ├── NO → Anthos is not for you (it's K8s-focused)
  └── YES
      ├── Only on GCP?
      │   ├── YES → GKE Standard is probably enough
      │   └── NO (multi-cloud or on-prem)
      │       ├── Need consistent policies across all clusters?
      │       │   ├── YES → ✅ Anthos is a strong fit
      │       │   └── NO → Manage clusters independently
      │       └── Need service mesh across clouds?
      │           ├── YES → ✅ Anthos Service Mesh
      │           └── NO → Consider open-source Istio
      └── More than 5-10 clusters?
          ├── YES → Anthos Fleet management helps
          └── NO → GKE Standard + manual management
```

> **One-liner**: Anthos is the "enterprise glue" for organizations running Kubernetes everywhere. If you're a small team on just GCP, save your money and use GKE Standard.

---

## Console Walkthrough: Fleet Management

### Viewing Your Fleet

```
┌──────────────────────────────────────────────────────────────┐
│         VIEW FLEET IN CONSOLE                                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Kubernetes Engine → Fleets                         │
│                                                                │
│  Step 2: Review registered clusters                            │
│  • All fleet members listed with:                             │
│    - Cluster name and location                                │
│    - Type (GKE, Attached — EKS/AKS, On-prem)                │
│    - Status (Ready, Degraded, Not Connected)                  │
│    - Version (K8s version)                                    │
│  • Click a cluster name to see its detail page                │
│                                                                │
│  Step 3: Check fleet-wide status                               │
│  • "Overview" tab shows health summary across all clusters   │
│  • "Features" tab shows which features are enabled            │
│    (Config Sync, Policy Controller, Service Mesh)             │
│                                                                │
│  ⚡ GKE clusters in the same project auto-register to the     │
│    fleet. External clusters appear here after registration.   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Enabling Fleet Features

```
┌──────────────────────────────────────────────────────────────┐
│         ENABLE FLEET FEATURES                                 │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Kubernetes Engine → Fleets → Features             │
│                                                                │
│  Step 2: Enable a feature                                      │
│  • Config Sync: Click ENABLE → configure Git repo details    │
│  • Policy Controller: Click ENABLE → choose policy bundles   │
│  • Service Mesh: Click ENABLE → select managed/in-cluster    │
│                                                                │
│  Step 3: Select target clusters                                │
│  • Choose which fleet members get the feature                 │
│  • Can be all clusters or a specific subset                   │
│                                                                │
│  Step 4: Verify                                                │
│  • Feature status shows per cluster:                          │
│    ✅ Synced / Active / Healthy                               │
│    ⚠ Pending / Configuring                                   │
│    ❌ Error (click to see details)                             │
│                                                                │
│  ⚡ Features are enabled at the fleet level but applied       │
│    per cluster. You can enable Config Sync for some clusters  │
│    and not others.                                             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# CLI equivalent — enable Config Sync
gcloud container fleet config-management enable

# CLI equivalent — enable Service Mesh
gcloud container fleet mesh enable
```

### Unregistering Clusters

```
┌──────────────────────────────────────────────────────────────┐
│         UNREGISTER CLUSTERS FROM FLEET                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Kubernetes Engine → Fleets                         │
│                                                                │
│  Step 2: Find the cluster                                      │
│  • Locate the cluster you want to remove from the fleet       │
│                                                                │
│  Step 3: Unregister                                            │
│  • Click the ⋮ (three-dot menu) next to the cluster          │
│  • Select "Unregister cluster"                                │
│  • Confirm the action                                         │
│                                                                │
│  What happens:                                                 │
│  • Cluster is removed from fleet management                   │
│  • Config Sync stops syncing to this cluster                  │
│  • Policy Controller is removed                               │
│  • Service mesh sidecar injection stops                       │
│  • The cluster itself is NOT deleted — it keeps running       │
│                                                                │
│  ⚠ Unregistering does NOT delete the cluster. It just         │
│    removes it from centralized fleet management. The cluster  │
│    and its workloads continue to run independently.            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# CLI equivalent
gcloud container fleet memberships unregister CLUSTER_NAME \
    --context=KUBECTL_CONTEXT \
    --kubeconfig=KUBECONFIG_PATH

# For GKE clusters
gcloud container fleet memberships unregister gke-prod-asia \
    --gke-cluster=us-central1/gke-prod-asia
```

---

## What's Next?

In the next chapter, we'll cover Google Cloud's serverless compute with Cloud Functions — event-driven functions without servers.

→ Next: [Chapter 21: Cloud Functions Deep Dive](21-cloud-functions-deep-dive.md)

---

*Last Updated: May 2026*
