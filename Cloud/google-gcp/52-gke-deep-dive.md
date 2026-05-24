# Chapter 52 — GKE Deep Dive

---

## Table of Contents

- [Overview](#overview)
- [Part 1: GKE Architecture Deep Dive](#part-1--gke-architecture-deep-dive)
- [Part 2: Network Policies](#part-2--network-policies)
- [Part 3: Workload Identity](#part-3--workload-identity)
- [Part 4: Config Connector](#part-4--config-connector)
- [Part 5: Multi-Cluster with Fleet](#part-5--multi-cluster-with-fleet)
- [Part 6: GKE Enterprise Features](#part-6--gke-enterprise-features)
- [Part 7: Binary Authorization](#part-7--binary-authorization)
- [Part 8: Backup for GKE](#part-8--backup-for-gke)
- [Part 9: GKE Cost Optimization](#part-9--gke-cost-optimization)
- [Part 10: Node Auto-Provisioning](#part-10--node-auto-provisioning)
- [Part 11: Vertical Pod Autoscaler](#part-11--vertical-pod-autoscaler)
- [Part 12: Custom Metrics Autoscaling](#part-12--custom-metrics-autoscaling)
- [Part 13: GKE Security Posture](#part-13--gke-security-posture)
- [Part 14: Console Walkthrough — Advanced GKE Configuration](#part-14-console-walkthrough--advanced-gke-configuration)
- [Part 15: Terraform & gcloud CLI Reference](#part-15--terraform--gcloud-cli-reference)
- [Part 16: Real-World Patterns](#part-16--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

While Chapter 18 covered GKE fundamentals, this chapter dives deep into advanced GKE features — network policies for pod-level traffic control, Workload Identity for secure GCP access, Config Connector for managing GCP resources via Kubernetes, multi-cluster fleet management, Binary Authorization for supply chain security, and cost optimization strategies. These features transform GKE from a basic Kubernetes runtime into an enterprise-grade platform.

---

## Part 1 — GKE Architecture Deep Dive

### Control Plane & Node Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│         GKE ARCHITECTURE                                            │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────┐        │
│  │ CONTROL PLANE (Google-managed)                          │        │
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │        │
│  │ │ API      │ │ etcd     │ │ Scheduler│ │ Controller│  │        │
│  │ │ Server   │ │ (HA)     │ │          │ │ Manager   │  │        │
│  │ └────┬─────┘ └──────────┘ └──────────┘ └──────────┘  │        │
│  │      │  Regional HA: replicated across 3 zones        │        │
│  └──────┼─────────────────────────────────────────────────┘        │
│         │                                                           │
│  ┌──────┼─────────────────────────────────────────────────┐        │
│  │ DATA PLANE (Customer-managed nodes)                     │        │
│  │      │                                                   │        │
│  │  ┌───▼────────────────────────────┐                     │        │
│  │  │ Node Pool: default-pool        │                     │        │
│  │  │ ┌──────┐ ┌──────┐ ┌──────┐   │                     │        │
│  │  │ │Node 1│ │Node 2│ │Node 3│   │  e2-standard-4     │        │
│  │  │ │ pods │ │ pods │ │ pods │   │  autoscale: 1-10   │        │
│  │  │ └──────┘ └──────┘ └──────┘   │                     │        │
│  │  └───────────────────────────────┘                     │        │
│  │                                                          │        │
│  │  ┌────────────────────────────────┐                     │        │
│  │  │ Node Pool: gpu-pool            │                     │        │
│  │  │ ┌──────┐ ┌──────┐             │  n1-standard-8     │        │
│  │  │ │Node 1│ │Node 2│             │  + NVIDIA T4       │        │
│  │  │ │ GPU  │ │ GPU  │             │  autoscale: 0-4    │        │
│  │  │ └──────┘ └──────┘             │                     │        │
│  │  └────────────────────────────────┘                     │        │
│  │                                                          │        │
│  │  ┌────────────────────────────────┐                     │        │
│  │  │ Node Pool: spot-pool           │                     │        │
│  │  │ ┌──────┐ ┌──────┐ ┌──────┐   │  Spot VMs          │        │
│  │  │ │Node 1│ │Node 2│ │Node 3│   │  60-91% discount   │        │
│  │  │ │batch │ │batch │ │batch │   │  preemptible       │        │
│  │  │ └──────┘ └──────┘ └──────┘   │                     │        │
│  │  └────────────────────────────────┘                     │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                      │
│  Cluster modes:                                                     │
│  • Standard: you manage nodes, node pools, upgrades                │
│  • Autopilot: Google manages nodes, pay per pod                    │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP GKE | AWS EKS | Azure AKS |
|---------|---------|---------|-----------|
| Managed K8s | GKE | EKS | AKS |
| Serverless mode | Autopilot | Fargate profiles | Virtual Nodes (ACI) |
| Control plane cost | Free (Standard) / $0.10/hr (Autopilot) | $0.10/hr | Free |
| Node auto-provisioning | Yes | Karpenter | KEDA + VPA |
| Workload identity | Workload Identity | IRSA / Pod Identity | Workload Identity |
| Multi-cluster | Fleet / Hub | None (use Argo) | Fleet Manager |
| Config-as-code GCP | Config Connector | ACK | ASO |
| Binary Authorization | Yes | None (use Kyverno) | None |
| Service mesh | Anthos Service Mesh | App Mesh | Istio add-on |

---

## Part 2 — Network Policies

### Kubernetes Network Policies in GKE

```
┌────────────────────────────────────────────────────────────────────┐
│         NETWORK POLICIES                                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Network policies control pod-to-pod traffic within a cluster:    │
│                                                                      │
│  Without network policy (default):                                 │
│  ┌──────────────────────────────────────────────┐                 │
│  │  All pods can talk to all pods (flat network) │                 │
│  │  frontend ◄──► backend ◄──► database          │                 │
│  │  frontend ◄──────────────► database  ← BAD   │                 │
│  └──────────────────────────────────────────────┘                 │
│                                                                      │
│  With network policy:                                               │
│  ┌──────────────────────────────────────────────┐                 │
│  │  frontend ──► backend ──► database            │                 │
│  │  frontend ──X──────────► database  ← BLOCKED │                 │
│  └──────────────────────────────────────────────┘                 │
│                                                                      │
│  Prerequisites:                                                     │
│  • Enable network policy on GKE cluster                           │
│  • Uses Calico CNI (Dataplane V2 uses Cilium)                    │
│  • Autopilot: network policies enabled by default                 │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Enable network policy on existing cluster
gcloud container clusters update my-cluster \
    --enable-network-policy \
    --location=us-central1
```

```yaml
# Default deny all ingress for a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}  # selects all pods in namespace
  policyTypes:
    - Ingress

---
# Allow frontend → backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080

---
# Allow backend → database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432

---
# Allow egress only to specific CIDR (e.g., external API)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8          # internal
        - ipBlock:
            cidr: 203.0.113.0/24      # external API
      ports:
        - protocol: TCP
          port: 443
    - to:                               # DNS
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

### GKE Dataplane V2 (Cilium)

```bash
# Create cluster with Dataplane V2 (recommended)
gcloud container clusters create my-cluster \
    --enable-dataplane-v2 \
    --location=us-central1

# Dataplane V2 benefits:
# • eBPF-based (faster than iptables)
# • Built-in network policy enforcement
# • Network policy logging
# • No separate Calico installation needed
```

---

## Part 3 — Workload Identity

### Secure GCP Access from Pods

```
┌────────────────────────────────────────────────────────────────────┐
│         WORKLOAD IDENTITY                                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Problem: How do pods access GCP services securely?                │
│                                                                      │
│  ❌ Bad: Export SA key JSON → mount as secret                      │
│  ❌ Bad: Use node SA (all pods get same permissions)               │
│  ✅ Good: Workload Identity (per-pod GCP identity)                 │
│                                                                      │
│  How it works:                                                      │
│  ┌─────────────────────────────────────────────────────────┐      │
│  │                                                           │      │
│  │  K8s Service Account ──bind──► GCP Service Account       │      │
│  │  (namespace/sa)                (project/sa@gsa)          │      │
│  │       │                             │                     │      │
│  │       │  pod runs as               │  has IAM roles      │      │
│  │       ▼                             ▼                     │      │
│  │  ┌─────────┐              ┌──────────────────┐          │      │
│  │  │  Pod    │──requests──►│ GKE Metadata     │          │      │
│  │  │         │  token      │ Server           │          │      │
│  │  │         │◄──token─────│ (intercepts      │          │      │
│  │  │         │             │  169.254.169.254) │          │      │
│  │  └─────────┘              └──────────────────┘          │      │
│  │       │                                                   │      │
│  │       │ uses token to access                             │      │
│  │       ▼                                                   │      │
│  │  ┌──────────────────────────────────────────┐           │      │
│  │  │ GCS, BigQuery, Pub/Sub, etc.              │           │      │
│  │  └──────────────────────────────────────────┘           │      │
│  └─────────────────────────────────────────────────────────┘      │
│                                                                      │
│  No keys! No secrets! Automatic token rotation!                    │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# 1. Enable Workload Identity on cluster
gcloud container clusters update my-cluster \
    --workload-pool=my-project.svc.id.goog \
    --location=us-central1

# 2. Create GCP service account
gcloud iam service-accounts create app-sa \
    --display-name="Application SA"

# 3. Grant GCP permissions to the SA
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:app-sa@my-project.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# 4. Create Kubernetes service account
kubectl create serviceaccount app-ksa --namespace=default

# 5. Bind K8s SA → GCP SA
gcloud iam service-accounts add-iam-policy-binding \
    app-sa@my-project.iam.gserviceaccount.com \
    --role="roles/iam.workloadIdentityUser" \
    --member="serviceAccount:my-project.svc.id.goog[default/app-ksa]"

# 6. Annotate K8s SA with GCP SA email
kubectl annotate serviceaccount app-ksa \
    --namespace=default \
    iam.gke.io/gcp-service-account=app-sa@my-project.iam.gserviceaccount.com
```

```yaml
# Pod using Workload Identity
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
spec:
  serviceAccountName: app-ksa    # K8s SA linked to GCP SA
  containers:
    - name: app
      image: gcr.io/my-project/my-app:latest
      # No SA key needed — GCP libraries auto-detect WI token
```

---

## Part 4 — Config Connector

### Manage GCP Resources via Kubernetes

```
┌────────────────────────────────────────────────────────────────────┐
│         CONFIG CONNECTOR                                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Config Connector lets you manage GCP resources as K8s objects:   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  kubectl apply -f bucket.yaml                             │     │
│  │       │                                                    │     │
│  │       ▼                                                    │     │
│  │  ┌──────────────┐     ┌───────────────────────────┐      │     │
│  │  │ Config       │────►│ GCP APIs                   │      │     │
│  │  │ Connector    │     │                           │      │     │
│  │  │ (CRD        │◄────│ Cloud Storage bucket      │      │     │
│  │  │  controller) │     │ BigQuery dataset          │      │     │
│  │  └──────────────┘     │ Pub/Sub topic             │      │     │
│  │                        │ SQL instance              │      │     │
│  │  Reconciliation loop: │ IAM policy                │      │     │
│  │  K8s desired state    │ + 200 more resources      │      │     │
│  │  = GCP actual state   └───────────────────────────┘      │     │
│  │                                                            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Benefits:                                                          │
│  • Unified GitOps — manage infra + apps in same repo              │
│  • K8s-native — use kubectl, Kustomize, Helm                     │
│  • Drift detection — auto-corrects manual changes                 │
│  • Namespace-scoped — team isolation                              │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Install Config Connector add-on
gcloud container clusters update my-cluster \
    --update-addons=ConfigConnector=ENABLED \
    --location=us-central1
```

```yaml
# Config Connector identity (per namespace)
apiVersion: core.cnrm.cloud.google.com/v1beta1
kind: ConfigConnectorContext
metadata:
  name: configconnectorcontext.core.cnrm.cloud.google.com
  namespace: team-a
spec:
  googleServiceAccount: "cnrm-sa@my-project.iam.gserviceaccount.com"

---
# Create a GCS bucket via K8s manifest
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  name: team-a-data-bucket
  namespace: team-a
  annotations:
    cnrm.cloud.google.com/project-id: "my-project"
spec:
  location: US
  storageClass: STANDARD
  versioning:
    enabled: true
  lifecycleRule:
    - action:
        type: Delete
      condition:
        age: 90

---
# Create a Pub/Sub topic
apiVersion: pubsub.cnrm.cloud.google.com/v1beta1
kind: PubSubTopic
metadata:
  name: orders-topic
  namespace: team-a
spec:
  messageRetentionDuration: "86400s"

---
# Create a BigQuery dataset
apiVersion: bigquery.cnrm.cloud.google.com/v1beta1
kind: BigQueryDataset
metadata:
  name: analytics
  namespace: team-a
spec:
  friendlyName: Analytics Dataset
  location: US
  defaultTableExpirationMs: "7776000000"

---
# Create a Cloud SQL instance
apiVersion: sql.cnrm.cloud.google.com/v1beta1
kind: SQLInstance
metadata:
  name: app-database
  namespace: team-a
spec:
  databaseVersion: POSTGRES_15
  region: us-central1
  settings:
    tier: db-custom-2-7680
    backupConfiguration:
      enabled: true
      startTime: "02:00"
    ipConfiguration:
      privateNetworkRef:
        name: my-vpc
```

---

## Part 5 — Multi-Cluster with Fleet

### Fleet Management (Hub)

```
┌────────────────────────────────────────────────────────────────────┐
│         FLEET (MULTI-CLUSTER MANAGEMENT)                            │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Fleet = logical grouping of Kubernetes clusters:                  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Fleet (Hub)                                               │     │
│  │  ┌──────────────────────────────────────────────┐         │     │
│  │  │                                                │         │     │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │         │     │
│  │  │  │  GKE     │  │  GKE     │  │  GKE     │   │         │     │
│  │  │  │ us-east  │  │ eu-west  │  │ ap-south │   │         │     │
│  │  │  │          │  │          │  │          │   │         │     │
│  │  │  └──────────┘  └──────────┘  └──────────┘   │         │     │
│  │  │                                                │         │     │
│  │  │  Shared across fleet:                          │         │     │
│  │  │  • Config sync (GitOps)                       │         │     │
│  │  │  • Policy controller (OPA Gatekeeper)         │         │     │
│  │  │  • Service mesh (ASM)                         │         │     │
│  │  │  • Multi-cluster Ingress                      │         │     │
│  │  │  • Multi-cluster Services                     │         │     │
│  │  │  • Fleet-wide logging & monitoring            │         │     │
│  │  └──────────────────────────────────────────────┘         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Fleet can include:                                                 │
│  • GKE clusters (any region)                                       │
│  • Anthos on bare metal                                            │
│  • Anthos on VMware                                                │
│  • Attached clusters (EKS, AKS)                                   │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Register a cluster to fleet
gcloud container fleet memberships register my-cluster \
    --gke-cluster=us-central1/my-cluster \
    --enable-workload-identity

# List fleet members
gcloud container fleet memberships list

# Enable Config Sync for fleet
gcloud container fleet config-management apply \
    --membership=my-cluster \
    --config=config-sync.yaml

# Enable Multi-Cluster Ingress
gcloud container fleet ingress enable \
    --config-membership=my-cluster
```

### Multi-Cluster Services

```yaml
# Export a service across clusters
apiVersion: net.gke.io/v1
kind: ServiceExport
metadata:
  name: backend-service
  namespace: production

---
# Import: automatically available as
# <service>.<namespace>.svc.clusterset.local
# e.g., backend-service.production.svc.clusterset.local
```

---

## Part 6 — GKE Enterprise Features

### GKE Enterprise Overview

```
┌────────────────────────────────────────────────────────────────────┐
│         GKE ENTERPRISE                                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  GKE Enterprise = GKE Standard + advanced features:               │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  Config Sync                                              │     │
│  │  ├── GitOps-based configuration management               │     │
│  │  ├── Sync K8s manifests from Git repos                   │     │
│  │  └── Namespace-scoped or cluster-scoped                  │     │
│  │                                                            │     │
│  │  Policy Controller                                        │     │
│  │  ├── OPA Gatekeeper-based policy enforcement             │     │
│  │  ├── Pre-built policy bundles (CIS, PCI-DSS, etc.)       │     │
│  │  └── Custom constraint templates                         │     │
│  │                                                            │     │
│  │  Anthos Service Mesh (ASM)                                │     │
│  │  ├── Managed Istio service mesh                          │     │
│  │  ├── mTLS, traffic management, observability             │     │
│  │  └── Multi-cluster mesh                                  │     │
│  │                                                            │     │
│  │  Connect Gateway                                          │     │
│  │  ├── kubectl access to any fleet cluster                 │     │
│  │  └── Via Google Cloud IAM (no direct cluster access)     │     │
│  │                                                            │     │
│  │  Fleet-wide Security Posture                              │     │
│  │  ├── Vulnerability scanning                              │     │
│  │  ├── Compliance reporting                                │     │
│  │  └── Configuration auditing                              │     │
│  │                                                            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Pricing: $0.01667/vCPU/hr ($12.17/vCPU/month)                   │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Config Sync

```yaml
# config-sync.yaml
applySpecVersion: 1
spec:
  configSync:
    enabled: true
    sourceFormat: unstructured
    git:
      repo: https://github.com/myorg/k8s-configs
      branch: main
      dir: clusters/production
      secretType: token
    preventDrift: true

# Policy Controller
  policyController:
    enabled: true
    templateLibraryInstalled: true
    referentialRulesEnabled: true
```

---

## Part 7 — Binary Authorization

### Container Image Verification

```
┌────────────────────────────────────────────────────────────────────┐
│         BINARY AUTHORIZATION                                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Ensure only trusted container images run on GKE:                 │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  Build ──► Sign ──► Verify ──► Deploy                     │     │
│  │                                                            │     │
│  │  1. Cloud Build builds image                              │     │
│  │  2. Attestor signs image digest                           │     │
│  │  3. Binary Auth policy checks attestation                 │     │
│  │  4. Only signed images are admitted to cluster            │     │
│  │                                                            │     │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐            │     │
│  │  │ Cloud     │  │ Container │  │ Binary    │            │     │
│  │  │ Build     │─►│ Analysis  │─►│ Auth      │            │     │
│  │  │           │  │ (signing) │  │ (verify)  │            │     │
│  │  └───────────┘  └───────────┘  └─────┬─────┘            │     │
│  │                                        │ admit/deny       │     │
│  │                                   ┌────▼─────┐            │     │
│  │                                   │ GKE      │            │     │
│  │                                   │ Cluster  │            │     │
│  │                                   └──────────┘            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Policy modes:                                                      │
│  • ALLOW_ALL: no enforcement (audit only)                          │
│  • DENY_ALL: block all images                                      │
│  • REQUIRE_ATTESTATION: only signed images allowed                 │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Enable Binary Authorization on cluster
gcloud container clusters update my-cluster \
    --enable-binauthz \
    --location=us-central1

# Create attestor
gcloud container binauthz attestors create build-attestor \
    --attestation-authority-note=build-note \
    --attestation-authority-note-project=my-project

# Set policy
gcloud container binauthz policy export > policy.yaml
# Edit policy to require attestation
gcloud container binauthz policy import policy.yaml
```

```yaml
# Binary Authorization policy
admissionWhitelistPatterns:
  - namePattern: "gcr.io/google-containers/*"
  - namePattern: "gke.gcr.io/*"
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  requireAttestationsBy:
    - projects/my-project/attestors/build-attestor
clusterAdmissionRules:
  us-central1.staging-cluster:
    evaluationMode: ALWAYS_ALLOW      # relaxed for staging
    enforcementMode: DRYRUN_AUDIT_LOG_ONLY
```

---

## Part 8 — Backup for GKE

### Cluster Backup & Restore

```bash
# Enable Backup for GKE API
gcloud services enable gkebackup.googleapis.com

# Create backup plan
gcloud beta container backup-restore backup-plans create my-backup-plan \
    --project=my-project \
    --location=us-central1 \
    --cluster=projects/my-project/locations/us-central1/clusters/my-cluster \
    --all-namespaces \
    --include-volume-data \
    --cron-schedule="0 2 * * *" \
    --backup-retain-days=30

# Create manual backup
gcloud beta container backup-restore backups create manual-backup-001 \
    --project=my-project \
    --location=us-central1 \
    --backup-plan=my-backup-plan \
    --wait-for-completion

# Restore from backup
gcloud beta container backup-restore restores create restore-001 \
    --project=my-project \
    --location=us-central1 \
    --restore-plan=my-restore-plan \
    --backup=projects/my-project/locations/us-central1/backupPlans/my-backup-plan/backups/manual-backup-001
```

---

## Part 9 — GKE Cost Optimization

### Cost Optimization Strategies

```
┌────────────────────────────────────────────────────────────────────┐
│         COST OPTIMIZATION                                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Strategy 1: RIGHT-SIZE NODES                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • Use e2-standard (cost-effective general purpose)       │     │
│  │ • Match machine type to workload needs                   │     │
│  │ • Avoid over-provisioning (check CPU/mem utilization)    │     │
│  │ • Use custom machine types for exact fit                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Strategy 2: SPOT VMs (up to 91% discount)                        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • Batch processing, CI/CD, dev/test                     │     │
│  │ • Use taints/tolerations to isolate spot workloads      │     │
│  │ • Combine with on-demand pool for critical workloads    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Strategy 3: COMMITTED USE DISCOUNTS (1yr: 20%, 3yr: 52%)        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • Commit to stable baseline usage                       │     │
│  │ • Use CUDs for predictable workloads                    │     │
│  │ • Combine with Spot VMs for burst capacity              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Strategy 4: AUTOPILOT (pay per pod)                               │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • No node management overhead                           │     │
│  │ • Pay only for pod resource requests                    │     │
│  │ • Automatic bin-packing                                 │     │
│  │ • Best for varied workloads                             │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Strategy 5: AUTOSCALING                                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • HPA: scale pods by CPU/memory/custom metrics          │     │
│  │ • VPA: right-size pod resource requests                 │     │
│  │ • Cluster autoscaler: scale nodes                       │     │
│  │ • NAP: auto-create node pools by workload needs         │     │
│  │ • Scale to zero for non-critical workloads              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Create spot VM node pool
gcloud container node-pools create spot-pool \
    --cluster=my-cluster \
    --location=us-central1 \
    --machine-type=e2-standard-4 \
    --spot \
    --enable-autoscaling \
    --min-nodes=0 \
    --max-nodes=20 \
    --num-nodes=0

# Taint spot nodes
gcloud container node-pools update spot-pool \
    --cluster=my-cluster \
    --location=us-central1 \
    --node-taints="cloud.google.com/gke-spot=true:NoSchedule"
```

```yaml
# Schedule batch jobs on spot nodes
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processor
spec:
  template:
    spec:
      tolerations:
        - key: "cloud.google.com/gke-spot"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      nodeSelector:
        cloud.google.com/gke-spot: "true"
      containers:
        - name: processor
          image: gcr.io/my-project/batch:latest
          resources:
            requests:
              cpu: "2"
              memory: "4Gi"
      restartPolicy: OnFailure
```

---

## Part 10 — Node Auto-Provisioning

### Automatic Node Pool Creation

```
┌────────────────────────────────────────────────────────────────────┐
│         NODE AUTO-PROVISIONING (NAP)                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  NAP automatically creates and deletes node pools based on        │
│  pending pod requirements:                                          │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Pod needs GPU?                                           │     │
│  │  → NAP creates a GPU node pool                           │     │
│  │                                                            │     │
│  │  Pod needs high-memory?                                   │     │
│  │  → NAP creates n2-highmem node pool                      │     │
│  │                                                            │     │
│  │  No pending pods for a pool?                              │     │
│  │  → NAP scales pool to 0 and eventually deletes it        │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  NAP vs Cluster Autoscaler:                                        │
│  • Cluster Autoscaler: scales existing node pools                 │
│  • NAP: creates NEW node pools to match pod requirements          │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Enable NAP
gcloud container clusters update my-cluster \
    --enable-autoprovisioning \
    --min-cpu=1 \
    --max-cpu=100 \
    --min-memory=1 \
    --max-memory=400 \
    --autoprovisioning-scopes=\
https://www.googleapis.com/auth/devstorage.read_only,\
https://www.googleapis.com/auth/logging.write,\
https://www.googleapis.com/auth/monitoring \
    --location=us-central1

# Enable NAP with GPU support
gcloud container clusters update my-cluster \
    --enable-autoprovisioning \
    --max-cpu=100 \
    --max-memory=400 \
    --min-accelerator type=nvidia-tesla-t4,count=0 \
    --max-accelerator type=nvidia-tesla-t4,count=8 \
    --location=us-central1
```

---

## Part 11 — Vertical Pod Autoscaler

### Right-Size Pod Resources

```
┌────────────────────────────────────────────────────────────────────┐
│         VERTICAL POD AUTOSCALER (VPA)                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  VPA automatically adjusts pod CPU and memory requests:           │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Current:  requests: cpu=1000m, memory=2Gi              │     │
│  │  Actual:   uses cpu=200m, memory=500Mi                  │     │
│  │  VPA:      recommends cpu=250m, memory=600Mi            │     │
│  │  Savings:  75% CPU, 70% memory → smaller nodes          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Modes:                                                             │
│  • Off: recommendations only (no changes)                         │
│  • Initial: set on pod creation only                               │
│  • Auto: update running pods (restarts pods)                      │
│                                                                      │
│  ⚠ VPA and HPA should not target the same metric (CPU/memory)    │
│  Use HPA for scaling replicas, VPA for right-sizing resources     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Enable VPA on cluster
gcloud container clusters update my-cluster \
    --enable-vertical-pod-autoscaling \
    --location=us-central1
```

```yaml
# VPA configuration
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: backend-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: backend
  updatePolicy:
    updateMode: "Auto"         # Off | Initial | Auto
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: "50m"
          memory: "64Mi"
        maxAllowed:
          cpu: "2000m"
          memory: "4Gi"
        controlledResources: ["cpu", "memory"]

---
# VPA recommendation-only mode (safe to start with)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: backend-vpa-recommend
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: backend
  updatePolicy:
    updateMode: "Off"          # just generate recommendations

# Check recommendations:
# kubectl describe vpa backend-vpa-recommend
```

---

## Part 12 — Custom Metrics Autoscaling

### HPA with Custom Metrics

```yaml
# HPA based on Pub/Sub queue depth (custom metric)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: queue-worker
  minReplicas: 1
  maxReplicas: 50
  metrics:
    - type: External
      external:
        metric:
          name: pubsub.googleapis.com|subscription|num_undelivered_messages
          selector:
            matchLabels:
              resource.labels.subscription_id: worker-subscription
        target:
          type: AverageValue
          averageValue: "100"      # 100 messages per pod

---
# HPA based on custom Prometheus metric
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "500"      # 500 RPS per pod
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100               # double pods per 30s
          periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10                # slow scale-down
          periodSeconds: 60
```

```bash
# Install custom metrics adapter (Stackdriver)
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/\
k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/\
production/adapter.yaml

# Verify custom metrics are available
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1" | jq .
```

---

## Part 13 — GKE Security Posture

### Security Best Practices

```
┌────────────────────────────────────────────────────────────────────┐
│         GKE SECURITY POSTURE                                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Layer 1: CLUSTER                                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • Private cluster (no public endpoint)                    │     │
│  │ • Master authorized networks                             │     │
│  │ • Shielded GKE nodes (Secure Boot, vTPM)                 │     │
│  │ • Auto-upgrade enabled                                    │     │
│  │ • Release channels (Regular or Stable)                    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Layer 2: NETWORK                                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • Network policies (default deny)                         │     │
│  │ • Dataplane V2 (eBPF/Cilium)                             │     │
│  │ • Private Google Access (no internet for nodes)           │     │
│  │ • VPC-native cluster (alias IPs)                          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Layer 3: IDENTITY                                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • Workload Identity (no SA keys)                          │     │
│  │ • Least-privilege GCP SA per workload                     │     │
│  │ • RBAC for K8s API access                                 │     │
│  │ • Groups for GKE (Google Groups RBAC)                     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Layer 4: WORKLOAD                                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • Binary Authorization (signed images only)               │     │
│  │ • Container image scanning (Artifact Registry)            │     │
│  │ • Pod Security Standards (restricted profile)             │     │
│  │ • No privileged containers                                │     │
│  │ • Read-only root filesystem                               │     │
│  │ • Non-root user                                           │     │
│  │ • GKE Sandbox (gVisor) for untrusted workloads           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Layer 5: DATA                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ • CMEK for boot/data disks                                │     │
│  │ • Secret Manager (not K8s secrets)                        │     │
│  │ • Application-layer encryption for etcd                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Create a hardened cluster
gcloud container clusters create secure-cluster \
    --location=us-central1 \
    --enable-private-nodes \
    --master-ipv4-cidr=172.16.0.0/28 \
    --enable-master-authorized-networks \
    --master-authorized-networks=10.0.0.0/8 \
    --enable-shielded-nodes \
    --shielded-secure-boot \
    --shielded-integrity-monitoring \
    --enable-dataplane-v2 \
    --workload-pool=my-project.svc.id.goog \
    --enable-binauthz \
    --enable-vertical-pod-autoscaling \
    --release-channel=regular \
    --enable-autorepair \
    --enable-autoupgrade \
    --enable-network-policy
```

```yaml
# Pod Security Standard (restricted)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# Secure pod spec
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      serviceAccountName: app-ksa
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: us-docker.pkg.dev/my-project/repo/app@sha256:abc123
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
            runAsUser: 1000
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

---

## Part 14: Console Walkthrough — Advanced GKE Configuration

### Enabling Workload Identity on a Cluster

```
Console → Kubernetes Engine → Clusters → Click cluster → EDIT

┌─────────────────────────────────────────────────────────────────┐
│           CLUSTER: production-cluster                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Security tab ──                                               │
│                                                                   │
│ Workload Identity:                                              │
│ ☑ Enable Workload Identity                                      │
│ Workload pool: [project-id.svc.id.goog]                        │
│ ⚡ Auto-set based on project ID.                                 │
│                                                                   │
│ ⚡ After enabling, configure K8s service accounts to             │
│   impersonate GCP service accounts:                             │
│   1. Create KSA in namespace                                   │
│   2. Annotate KSA with GSA email                               │
│   3. Bind GSA → KSA via IAM                                   │
│                                                                   │
│                              [SAVE]                             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Configuring Network Policies

```
Console → Kubernetes Engine → Clusters → Click cluster → EDIT

┌─────────────────────────────────────────────────────────────────┐
│           NETWORKING tab                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ☑ Enable network policy                                         │
│ Network policy provider: [Calico ▼]                            │
│ ⚡ Enables Kubernetes NetworkPolicy enforcement.                 │
│   Calico is the default and only provider for GKE.             │
│                                                                   │
│ ☑ Enable Dataplane V2                                           │
│ ⚡ Uses eBPF-based networking (Cilium). Recommended.             │
│   Replaces Calico for network policy enforcement.              │
│   Better performance and observability.                        │
│                                                                   │
│ ⚡ Network policies are applied via kubectl                     │
│   (K8s NetworkPolicy YAML), NOT through the console.          │
│   Console only enables/disables the feature on the cluster.   │
│                                                                   │
│                              [SAVE]                             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Enabling Binary Authorization

```
Console → Security → Binary Authorization → CONFIGURE POLICY

┌─────────────────────────────────────────────────────────────────┐
│           BINARY AUTHORIZATION POLICY                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Default rule ──                                              │
│ Default rule:                                                   │
│ ○ Allow all images                                             │
│ ○ Disallow all images                                          │
│ ● Require attestations                                         │
│                                                                   │
│ ── Attestors ──                                                  │
│ [ADD ATTESTOR]                                                  │
│ Name: [build-verified]                                          │
│ Attestor type:                                                  │
│ ● PKIX (public key)                                            │
│ ○ PGP                                                          │
│ Public key: [paste PEM-encoded public key]                     │
│                                                                   │
│ ── Exemptions (optional) ──                                     │
│ Exempt images:                                                  │
│ [gcr.io/google-containers/*]                                   │
│ [us-docker.pkg.dev/google-samples/*]                           │
│ ⚡ Google system images are typically exempted.                  │
│                                                                   │
│                          [SAVE POLICY]                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Then enable on cluster:
Console → Kubernetes Engine → Clusters → Edit → Security tab
☑ Enable Binary Authorization
```

### Creating a Backup Plan (Backup for GKE)

```
Console → Kubernetes Engine → Backup for GKE → CREATE BACKUP PLAN

┌─────────────────────────────────────────────────────────────────┐
│           CREATE BACKUP PLAN                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Plan details ──                                              │
│ Name: [daily-prod-backup]                                       │
│ Region: [us-central1 ▼]                                         │
│ Cluster: [production-cluster ▼]                                 │
│                                                                   │
│ ── Backup scope ──                                              │
│ ● Entire cluster                                               │
│ ○ Selected namespaces:                                         │
│   [production, monitoring]                                     │
│ ○ Selected applications (ProtectedApplication CRs)             │
│                                                                   │
│ ── Schedule ──                                                   │
│ Cron schedule: [0 2 * * *]  (daily at 2 AM)                   │
│                                                                   │
│ ── Retention ──                                                  │
│ Retain backups for: [30] days                                   │
│ Minimum retained backups: [3]                                   │
│                                                                   │
│ ── Backup config ──                                             │
│ ☑ Include Persistent Volume data                                │
│ ☑ Include secrets                                               │
│ Encryption: [● Google-managed | ○ CMEK]                        │
│                                                                   │
│                              [CREATE]                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 15 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── GKE Cluster (Production-Ready) ──────────────────────────
resource "google_container_cluster" "primary" {
  name     = "production-cluster"
  location = var.region
  project  = var.project_id

  # Remove default node pool
  remove_default_node_pool = true
  initial_node_count       = 1

  # Networking
  network    = google_compute_network.vpc.id
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

  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "10.0.0.0/8"
      display_name = "internal"
    }
  }

  # Security
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }

  datapath_provider = "ADVANCED_DATAPATH"  # Dataplane V2

  release_channel {
    channel = "REGULAR"
  }

  # Addons
  addons_config {
    config_connector_config {
      enabled = true
    }
  }

  # Maintenance window
  maintenance_policy {
    recurring_window {
      start_time = "2024-01-01T02:00:00Z"
      end_time   = "2024-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA"
    }
  }
}

# ─── Primary Node Pool ───────────────────────────────────────
resource "google_container_node_pool" "primary" {
  name     = "primary-pool"
  location = var.region
  cluster  = google_container_cluster.primary.name
  project  = var.project_id

  autoscaling {
    min_node_count = 1
    max_node_count = 10
  }

  node_config {
    machine_type    = "e2-standard-4"
    service_account = google_service_account.gke_nodes.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]

    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    metadata = {
      disable-legacy-endpoints = "true"
    }
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }
}

# ─── Spot Node Pool ──────────────────────────────────────────
resource "google_container_node_pool" "spot" {
  name     = "spot-pool"
  location = var.region
  cluster  = google_container_cluster.primary.name
  project  = var.project_id

  autoscaling {
    min_node_count = 0
    max_node_count = 20
  }

  node_config {
    machine_type    = "e2-standard-4"
    spot            = true
    service_account = google_service_account.gke_nodes.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]

    taint {
      key    = "cloud.google.com/gke-spot"
      value  = "true"
      effect = "NO_SCHEDULE"
    }

    labels = {
      "cloud.google.com/gke-spot" = "true"
    }
  }
}

# ─── Workload Identity Setup ─────────────────────────────────
resource "google_service_account" "app_sa" {
  account_id   = "app-workload-sa"
  display_name = "App Workload Identity SA"
  project      = var.project_id
}

resource "google_project_iam_member" "app_sa_storage" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.app_sa.email}"
}

resource "google_service_account_iam_member" "workload_identity" {
  service_account_id = google_service_account.app_sa.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[default/app-ksa]"
}

resource "kubernetes_service_account" "app_ksa" {
  metadata {
    name      = "app-ksa"
    namespace = "default"
    annotations = {
      "iam.gke.io/gcp-service-account" = google_service_account.app_sa.email
    }
  }
}

# ─── Fleet Registration ──────────────────────────────────────
resource "google_gke_hub_membership" "primary" {
  membership_id = "production-cluster"
  project       = var.project_id

  endpoint {
    gke_cluster {
      resource_link = google_container_cluster.primary.id
    }
  }
  authority {
    issuer = "https://container.googleapis.com/v1/${google_container_cluster.primary.id}"
  }
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# CLUSTER
# ═══════════════════════════════════════════════════════════════
gcloud container clusters create NAME --location=LOC [options]
gcloud container clusters update NAME --location=LOC [options]
gcloud container clusters describe NAME --location=LOC
gcloud container clusters delete NAME --location=LOC
gcloud container clusters get-credentials NAME --location=LOC

# ═══════════════════════════════════════════════════════════════
# NODE POOLS
# ═══════════════════════════════════════════════════════════════
gcloud container node-pools create NAME --cluster=C --location=LOC [opts]
gcloud container node-pools update NAME --cluster=C --location=LOC [opts]
gcloud container node-pools list --cluster=C --location=LOC
gcloud container node-pools delete NAME --cluster=C --location=LOC

# ═══════════════════════════════════════════════════════════════
# FLEET
# ═══════════════════════════════════════════════════════════════
gcloud container fleet memberships register NAME --gke-cluster=LOC/C
gcloud container fleet memberships list
gcloud container fleet memberships unregister NAME

# ═══════════════════════════════════════════════════════════════
# BINARY AUTHORIZATION
# ═══════════════════════════════════════════════════════════════
gcloud container binauthz policy export
gcloud container binauthz policy import FILE
gcloud container binauthz attestors create NAME [opts]
gcloud container binauthz attestors list

# ═══════════════════════════════════════════════════════════════
# BACKUP
# ═══════════════════════════════════════════════════════════════
gcloud beta container backup-restore backup-plans create NAME [opts]
gcloud beta container backup-restore backups create NAME [opts]
gcloud beta container backup-restore restores create NAME [opts]
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Multi-Environment Platform

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: MULTI-ENVIRONMENT PLATFORM                            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Fleet                                                     │        │
│  │                                                            │        │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │        │
│  │  │ Dev Cluster   │  │ Staging      │  │ Production   │   │        │
│  │  │ (Autopilot)   │  │ (Standard)   │  │ (Standard)   │   │        │
│  │  │               │  │              │  │              │   │        │
│  │  │ us-central1   │  │ us-central1  │  │ Multi-region │   │        │
│  │  │ Spot nodes    │  │ Mixed nodes  │  │ Regional HA  │   │        │
│  │  │ No BinAuth    │  │ BinAuth dry  │  │ BinAuth enf  │   │        │
│  │  └──────┬────────┘  └──────┬───────┘  └──────┬───────┘   │        │
│  │         │                   │                  │            │        │
│  │  ┌──────▼───────────────────▼──────────────────▼───────┐  │        │
│  │  │ Config Sync (GitOps)                                 │  │        │
│  │  │ Git repo: k8s-configs/                              │  │        │
│  │  │ ├── base/          (shared configs)                 │  │        │
│  │  │ ├── dev/           (dev overrides)                  │  │        │
│  │  │ ├── staging/       (staging overrides)              │  │        │
│  │  │ └── production/    (prod overrides + policies)      │  │        │
│  │  └─────────────────────────────────────────────────────┘  │        │
│  │                                                            │        │
│  │  Policy Controller:                                       │        │
│  │  • Dev: warn only                                        │        │
│  │  • Staging: audit                                        │        │
│  │  • Production: enforce (block non-compliant resources)   │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Cost-Optimized ML Platform

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: ML PLATFORM ON GKE                                     │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  GKE Cluster with NAP                                      │        │
│  │                                                            │        │
│  │  Node Pool: system (always on)                            │        │
│  │  ├── e2-standard-2, 2 nodes                              │        │
│  │  ├── Runs: kube-system, monitoring, ingress              │        │
│  │  └── CUD discount applied                                │        │
│  │                                                            │        │
│  │  Node Pool: inference (on-demand)                         │        │
│  │  ├── n1-standard-4 + T4 GPU                              │        │
│  │  ├── Autoscale 0-8 based on request queue                │        │
│  │  ├── VPA right-sizes memory                              │        │
│  │  └── Workload Identity → access model in GCS             │        │
│  │                                                            │        │
│  │  Node Pool: training (spot)                               │        │
│  │  ├── a2-highgpu-1g (A100 GPU)                            │        │
│  │  ├── Spot VMs (60-91% discount)                          │        │
│  │  ├── NAP auto-creates when training jobs submitted       │        │
│  │  ├── Checkpoint to GCS every 10 minutes                  │        │
│  │  └── Auto-retry on preemption                            │        │
│  │                                                            │        │
│  │  HPA: scale inference pods on custom metric              │        │
│  │  (pubsub queue depth = pending predictions)              │        │
│  │                                                            │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Monthly savings vs all on-demand:                                   │
│  • Spot training: ~70% savings on GPU compute                      │
│  • VPA inference: ~40% savings (right-sized)                       │
│  • CUD system: ~20% savings on baseline                            │
│  • Scale-to-zero: no cost when idle                                │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Global Multi-Cluster Application

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: GLOBAL MULTI-CLUSTER                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │                  Global Load Balancer                      │        │
│  │              (Multi-Cluster Ingress / Gateway)             │        │
│  │  ┌──────────────────────────────────────────────┐         │        │
│  │  │ users → closest healthy cluster               │         │        │
│  │  └────┬──────────────────┬──────────────┬────────┘         │        │
│  │       │                  │              │                   │        │
│  │  ┌────▼──────┐  ┌───────▼────┐  ┌─────▼───────┐          │        │
│  │  │ us-east1  │  │ europe-w1  │  │ asia-east1  │          │        │
│  │  │ GKE       │  │ GKE        │  │ GKE         │          │        │
│  │  │           │  │            │  │             │          │        │
│  │  │ frontend  │  │ frontend   │  │ frontend    │          │        │
│  │  │ backend   │  │ backend    │  │ backend     │          │        │
│  │  │ cache     │  │ cache      │  │ cache       │          │        │
│  │  └────┬──────┘  └─────┬──────┘  └──────┬──────┘          │        │
│  │       │               │                │                   │        │
│  │  ┌────▼───────────────▼────────────────▼──────┐           │        │
│  │  │ Multi-Cluster Services (ServiceExport)      │           │        │
│  │  │ backend.prod.svc.clusterset.local           │           │        │
│  │  └───────────────────────────────────────────  ┘           │        │
│  │                                                            │        │
│  │  Shared across fleet:                                     │        │
│  │  • Config Sync: same configs deployed to all clusters    │        │
│  │  • ASM: cross-cluster mTLS + traffic management          │        │
│  │  • Policy Controller: consistent security policies        │        │
│  │  • Cloud SQL (regional) + Spanner (global) for data      │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Benefits:                                                            │
│  • Low latency globally (users hit closest cluster)                 │
│  • High availability (survive full regional outage)                 │
│  • Consistent policies and configs via GitOps                       │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Feature | Command / Config |
|---------|-----------------|
| Enable network policy | `--enable-network-policy` or `--enable-dataplane-v2` |
| Enable Workload Identity | `--workload-pool=PROJECT.svc.id.goog` |
| Bind KSA → GSA | `gcloud iam service-accounts add-iam-policy-binding GSA --role=roles/iam.workloadIdentityUser --member=serviceAccount:PROJECT.svc.id.goog[NS/KSA]` |
| Enable Config Connector | `--update-addons=ConfigConnector=ENABLED` |
| Register to fleet | `gcloud container fleet memberships register NAME --gke-cluster=LOC/C` |
| Enable Binary Auth | `--enable-binauthz` |
| Create spot pool | `--spot --enable-autoscaling --min-nodes=0` |
| Enable NAP | `--enable-autoprovisioning --max-cpu=N --max-memory=N` |
| Enable VPA | `--enable-vertical-pod-autoscaling` |
| Private cluster | `--enable-private-nodes --master-ipv4-cidr=CIDR` |
| Shielded nodes | `--enable-shielded-nodes --shielded-secure-boot` |
| Dataplane V2 | `--enable-dataplane-v2` (uses Cilium/eBPF) |

---

## What's Next?

Continue to **Chapter 53: Anthos** → `53-anthos.md`
