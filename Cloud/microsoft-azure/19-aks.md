# Chapter 19: AKS - Azure Kubernetes Service

---

## Table of Contents

- [Overview](#overview)
- [Part 1: AKS Fundamentals](#part-1-aks-fundamentals)
- [Part 2: Creating a Cluster (Full Portal Walkthrough)](#part-2-creating-a-cluster-full-portal-walkthrough)
- [Part 3: AKS Networking Deep Dive](#part-3-aks-networking-deep-dive)
- [Part 4: Azure AD (Entra ID) RBAC](#part-4-azure-ad-entra-id-rbac)
- [Part 5: AKS Autoscaling](#part-5-aks-autoscaling)
- [Part 6: Terraform](#part-6-terraform)
- [Part 7: Bicep](#part-7-bicep)
- [Part 8: az CLI Reference](#part-8-az-cli-reference)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Kubernetes Service (AKS) is Azure's fully managed Kubernetes platform. Microsoft manages the control plane (free!), and you manage worker nodes via node pools. AKS deeply integrates with Azure AD (Entra ID), Azure Monitor, Azure Policy, and the entire Azure ecosystem. It's the most popular way to run containers at scale on Azure.

```
What you'll learn:
├── AKS Fundamentals
│   ├── What & why (managed K8s on Azure)
│   ├── AKS vs ACI vs Container Apps vs App Service
│   └── Free tier vs Standard vs Premium tier
├── Creating a Cluster (Full Portal Walkthrough)
│   ├── Basics (name, region, K8s version, tier)
│   ├── Node pools (system + user pools, VM sizes, scaling)
│   ├── Networking (kubenet vs Azure CNI, load balancer, ingress)
│   ├── Integrations (ACR, Azure Monitor, Azure Policy)
│   ├── Security (Azure AD/Entra ID RBAC, managed identity, Defender)
│   └── Advanced (private cluster, upgrade channel)
├── Node Pools Deep Dive
├── AKS Networking
│   ├── Kubenet vs Azure CNI vs Azure CNI Overlay
│   ├── Load Balancer & Ingress Controller
│   └── Network Policies
├── Azure AD (Entra ID) RBAC
├── AKS Autoscaling (HPA, Cluster Autoscaler, KEDA)
├── AKS Monitoring
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: AKS Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           AKS CONCEPT                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is AKS?                                                        │
│ ├── Fully managed Kubernetes control plane (FREE!)               │
│ ├── Microsoft manages: API server, etcd, scheduler, controllers │
│ ├── You manage: Worker nodes (via node pools)                   │
│ ├── Pay only for VM nodes (control plane = free)                │
│ ├── Deep Azure integration (AD, Monitor, Policy, Key Vault)    │
│ └── Supports Windows + Linux nodes in same cluster             │
│                                                                       │
│ AKS Tiers:                                                          │
│ ┌───────────────┬────────────────────────────────────────────────┐│
│ │ Tier           │ Details                                        ││
│ ├───────────────┼────────────────────────────────────────────────┤│
│ │ Free          │ Free control plane, no SLA, no uptime          ││
│ │               │ guarantee. For dev/test.                       ││
│ ├───────────────┼────────────────────────────────────────────────┤│
│ │ Standard      │ $0.10/hr ($73/mo) per cluster.                 ││
│ │               │ 99.95% SLA (with AZs), 99.9% (without AZs).  ││
│ │               │ For production. ✅                              ││
│ ├───────────────┼────────────────────────────────────────────────┤│
│ │ Premium       │ $0.60/hr ($438/mo) per cluster.                ││
│ │               │ Long-term support (LTS), advanced networking, ││
│ │               │ Azure Fleet Manager. For enterprise.          ││
│ └───────────────┴────────────────────────────────────────────────┘│
│                                                                       │
│ Comparison:                                                          │
│ ┌────────────────┬──────────┬──────────┬──────────────┬──────────┐│
│ │ Feature         │ AKS       │ ACI       │ Container Apps│ App Svc ││
│ ├────────────────┼──────────┼──────────┼──────────────┼──────────┤│
│ │ Complexity      │ High     │ Lowest   │ Medium        │ Low     ││
│ │ K8s native      │ Yes ✅   │ No       │ K8s-inspired │ No      ││
│ │ Auto-scale      │ Full ✅  │ None     │ KEDA ✅       │ Yes     ││
│ │ Scale to zero   │ KEDA     │ N/A      │ Yes ✅        │ No      ││
│ │ Multi-container │ Pod ✅   │ Group    │ Sidecar       │ No      ││
│ │ Windows         │ Yes      │ Yes      │ Yes           │ Yes     ││
│ │ GPU             │ Yes ✅   │ Yes      │ No            │ No      ││
│ │ Service mesh    │ Istio ✅ │ No       │ Dapr          │ No      ││
│ │ RBAC            │ K8s+AD ✅│ Azure    │ Azure         │ Azure   ││
│ │ Pricing         │ Per VM   │ Per sec  │ Per use       │ Per plan││
│ │ Best for        │ Complex  │ Tasks    │ Microservices │ Web apps││
│ └────────────────┴──────────┴──────────┴──────────────┴──────────┘│
│                                                                       │
│ ⚡ AWS equivalent: EKS                                               │
│ ⚡ GCP equivalent: GKE                                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Cluster (Full Portal Walkthrough)

```
Console → Kubernetes services → Create

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 1: BASICS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-aks-prod ▼]                                    │
│                                                                       │
│ Cluster preset configuration:                                       │
│ ○ Dev/Test         ← Cheapest, no SLA                            │
│ ● Production Standard ← HA, SLA ✅                                │
│ ○ Production Enterprise ← Premium tier, LTS                      │
│ ○ Production Economy  ← Standard tier, cost-optimized           │
│                                                                       │
│ Cluster details:                                                     │
│ Kubernetes cluster name: [aks-prod]                                │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ Availability zones: [1, 2, 3 ▼] ← Node spread across all AZs  │
│ ⚡ Multi-AZ = HA. Nodes distributed across zones automatically.   │
│   System pods have anti-affinity to spread across zones.         │
│                                                                       │
│ AKS pricing tier:                                                    │
│ ○ Free ← No SLA (dev/test)                                       │
│ ● Standard ← $73/mo, 99.95% SLA ✅                                │
│ ○ Premium ← $438/mo, LTS + advanced features                    │
│                                                                       │
│ Kubernetes version: [1.30.4 ▼]                                    │
│ ⚡ AKS supports N-2 minor versions.                                 │
│   Currently: 1.30, 1.29, 1.28 available.                        │
│                                                                       │
│ Automatic upgrade:                                                   │
│ ● None ← You control upgrades                                    │
│ ○ Patch ← Auto-upgrade patch versions (1.30.3 → 1.30.4)       │
│ ○ Stable ← N-1 minor version (safest auto) ✅                   │
│ ○ Rapid ← Latest K8s version                                    │
│ ○ Node image ← Upgrade node OS images only                      │
│                                                                       │
│ Node security channel:                                              │
│ ● None                                                              │
│ ○ SecurityPatch ← Automatic node image security patches ✅      │
│                                                                       │
│ Authentication and authorization:                                   │
│ ● Microsoft Entra ID authentication with Azure RBAC ✅            │
│ ○ Local accounts with Kubernetes RBAC                            │
│ ⚡ Entra ID: Users/groups from your org can access cluster.        │
│   Azure RBAC: Assign K8s permissions via Azure roles.           │
│   No kubeconfig sharing, no static tokens!                      │
│                                                                       │
│ [Next: Node pools >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 2: NODE POOLS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── System node pool ──                                              │
│ ⚡ System pool: Runs critical system pods (CoreDNS, metrics-server)│
│   Must exist, can't have 0 nodes. Tainted by default.           │
│                                                                       │
│ Node pool name: [systempool]                                       │
│ Mode: System                                                        │
│ OS SKU: [Ubuntu Linux ▼]  (Ubuntu, AzureLinux, Windows)          │
│ ⚡ Azure Linux: Microsoft's container-optimized Linux              │
│   (lighter, faster boot, fewer vulnerabilities) ✅               │
│                                                                       │
│ Availability zones: [1, 2, 3]                                      │
│                                                                       │
│ Node size: [Standard_D4s_v5 ▼]                                    │
│ ┌─────────────────────────┬────────┬─────────┬──────────────────┐│
│ │ VM Size                  │ vCPU   │ Memory   │ Best for         ││
│ ├─────────────────────────┼────────┼─────────┼──────────────────┤│
│ │ Standard_B2s             │ 2      │ 4 GiB   │ Dev/test (burst)││
│ │ Standard_D2s_v5          │ 2      │ 8 GiB   │ Small apps      ││
│ │ Standard_D4s_v5          │ 4      │ 16 GiB  │ General ✅       ││
│ │ Standard_D8s_v5          │ 8      │ 32 GiB  │ Larger apps     ││
│ │ Standard_E4s_v5          │ 4      │ 32 GiB  │ Memory-intensive││
│ │ Standard_F4s_v2          │ 4      │ 8 GiB   │ CPU-intensive   ││
│ │ Standard_NC4as_T4_v3     │ 4      │ 28 GiB  │ GPU (T4)        ││
│ │ Standard_NC24ads_A100_v4 │ 24     │ 220 GiB │ GPU (A100)      ││
│ └─────────────────────────┴────────┴─────────┴──────────────────┘│
│                                                                       │
│ Scale method:                                                        │
│ ● Autoscale ✅                                                      │
│ ○ Manual                                                            │
│                                                                       │
│ Minimum node count: [2]                                             │
│ Maximum node count: [5]                                             │
│ ⚡ System pool min=2 (or 3) for HA across zones.                   │
│                                                                       │
│ Max pods per node: [30 ▼]  (default 30 for Azure CNI, 250 kubenet)│
│ ⚡ Azure CNI: Each pod gets a VNet IP. Need more IPs!              │
│   Plan subnet size: nodes × max_pods IPs needed.                 │
│                                                                       │
│ ── User node pool (add) ──                                         │
│ [+ Add node pool]                                                   │
│                                                                       │
│ Node pool name: [userpool]                                          │
│ Mode: User                                                          │
│ Node size: [Standard_D4s_v5]                                       │
│ Scale: Autoscale, min=2, max=20                                   │
│                                                                       │
│ ⚡ User pools: Run your application workloads.                      │
│   Separate system and user pools (best practice):               │
│   ├── System pool: Small, tainted, system pods only             │
│   ├── User pool(s): App pods, autoscale independently          │
│   └── GPU pool: For ML workloads (with taint)                  │
│                                                                       │
│ Node labels:                                                        │
│ role = general                                                      │
│ env = prod                                                          │
│                                                                       │
│ Node taints:                                                        │
│ [+ Add taint]                                                       │
│ (System pool auto-gets: CriticalAddonsOnly=true:NoSchedule)     │
│                                                                       │
│ Enable Spot VMs: ☐                                                  │
│ ⚡ Spot: 60-90% cheaper, can be evicted. For batch/non-critical.  │
│   Eviction policy: Delete (node removed when preempted).        │
│   Add taint: kubernetes.azure.com/scalesetpriority=spot:NoSched │
│                                                                       │
│ [Next: Networking >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 3: NETWORKING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Network configuration:                                               │
│ ○ Kubenet (basic, overlay, nodes get VNet IPs, pods get private) │
│ ● Azure CNI (advanced, pods get VNet IPs directly) ✅             │
│ ○ Azure CNI Overlay (pods get overlay IPs, nodes get VNet IPs)  │
│ ○ None (BYO CNI — Cilium, Calico)                               │
│                                                                       │
│ ┌────────────────┬──────────────┬───────────────┬──────────────┐ │
│ │ Feature         │ Kubenet       │ Azure CNI      │ CNI Overlay   │ │
│ ├────────────────┼──────────────┼───────────────┼──────────────┤ │
│ │ Pod IPs         │ Overlay (non-│ VNet IPs ✅   │ Overlay      │ │
│ │                │ routable)    │ (routable)    │ (non-routable)│ │
│ │ IP consumption │ Low          │ High ⚠️       │ Low ✅        │ │
│ │ VNet peering   │ UDR needed   │ Native ✅      │ Native       │ │
│ │ Performance    │ Extra hop    │ Best ✅        │ Good         │ │
│ │ Windows nodes  │ No ❌        │ Yes ✅         │ Yes          │ │
│ │ Network Policy │ Calico       │ Azure/Calico  │ Cilium       │ │
│ │ Max pods/node  │ 250          │ 30 (default)  │ 250          │ │
│ │ Subnet size    │ Small OK     │ Large needed! │ Small OK     │ │
│ │ Best for       │ Dev/test     │ Production ✅  │ Large clusters│ │
│ └────────────────┴──────────────┴───────────────┴──────────────┘ │
│                                                                       │
│ ═══ Azure CNI selected ═══                                          │
│                                                                       │
│ Virtual network: [vnet-prod ▼]                                     │
│ Cluster subnet: [subnet-aks (10.0.0.0/16) ▼]                     │
│ ⚡ Azure CNI: Every pod gets an IP from this subnet!               │
│   Need large subnet: nodes × 30 pods × IP each.                 │
│   Example: 20 nodes × 30 pods = 600 IPs → /22 minimum.         │
│                                                                       │
│ Kubernetes service address range: [10.1.0.0/16]                   │
│ ⚡ ClusterIP range. Must NOT overlap with VNet or subnets.         │
│                                                                       │
│ Kubernetes DNS service IP: [10.1.0.10]                             │
│ ⚡ Must be within service address range.                            │
│                                                                       │
│ DNS name prefix: [aks-prod]                                         │
│ ⚡ Used for FQDN: aks-prod.centralindia.azmk8s.io                  │
│                                                                       │
│ Network policy:                                                      │
│ ○ None                                                              │
│ ● Azure (Azure-native NetworkPolicy) ✅                             │
│ ○ Calico (full Calico NetworkPolicy)                              │
│ ⚡ Network policies: Control pod-to-pod traffic. Default: all open.│
│   Azure: Simpler, Azure-managed. Calico: More powerful.          │
│                                                                       │
│ Load balancer: [Standard ▼]                                        │
│ ⚡ Standard LB: Zone-redundant, supports AZs. (Basic deprecated.) │
│                                                                       │
│ ☑ Enable HTTP application routing (preview)                       │
│ ⚡ Or better: Install NGINX Ingress Controller or App Gateway     │
│   Ingress Controller (AGIC) after cluster creation.              │
│                                                                       │
│ ── Private cluster ──                                              │
│ ☐ Enable private cluster                                           │
│ ⚡ Private cluster: API server has no public IP.                    │
│   Access only via private endpoint (VPN, ExpressRoute, bastion).│
│   Optionally: Enable "API server VNet integration" to place     │
│   the API server in your VNet subnet.                           │
│                                                                       │
│ [Next: Integrations >]                                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 4: INTEGRATIONS                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Container registry ──                                           │
│ Azure Container Registry: [acrmyregistry ▼]                       │
│ ⚡ Attaches ACR to AKS (grants AcrPull role to kubelet identity). │
│   No imagePullSecrets needed! Just use image: acr.azurecr.io/.. │
│                                                                       │
│ ── Monitoring ──                                                    │
│ ☑ Enable Container insights                                       │
│ Log Analytics workspace: [law-aks-prod ▼]                         │
│ ⚡ Container insights: CPU, memory, logs for all pods.              │
│   Sends to Log Analytics → query with KQL.                      │
│   Prometheus metrics also available.                             │
│                                                                       │
│ ☑ Enable Managed Prometheus                                        │
│ Azure Monitor workspace: [amw-prod ▼]                             │
│ ⚡ Azure Managed Prometheus: Scrapes metrics, stores centrally.    │
│   Grafana for visualization (Azure Managed Grafana).             │
│                                                                       │
│ ☑ Enable Managed Grafana                                           │
│ Grafana workspace: [grafana-prod ▼]                               │
│                                                                       │
│ ── Azure Policy ──                                                  │
│ ☑ Enable Azure Policy                                              │
│ ⚡ Azure Policy for AKS (Gatekeeper/OPA under the hood):           │
│   ├── Enforce: No privileged containers                          │
│   ├── Enforce: Resource limits required on all pods             │
│   ├── Enforce: Only images from approved ACRs                   │
│   ├── Enforce: No host network/PID namespace                    │
│   └── Audit: Report non-compliant resources                     │
│                                                                       │
│ ── Key Vault ──                                                     │
│ ☑ Enable Secret Store CSI Driver                                  │
│ ⚡ Mount Key Vault secrets as volumes in pods.                      │
│   No need to manage K8s Secrets!                                │
│   SecretProviderClass → maps KV secrets to pod volumes.         │
│                                                                       │
│ [Next: Advanced >]                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 5: ADVANCED                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ── Infrastructure ──                                                │
│ ☑ Enable secret store CSI driver                                  │
│ ☑ Enable OIDC issuer                                               │
│ ☑ Enable Workload Identity                                        │
│ ⚡ Workload Identity: Pods → Azure AD identity (no credentials!)  │
│   K8s ServiceAccount ↔ Azure Managed Identity.                  │
│   Same concept as GKE Workload Identity / AWS IRSA.             │
│                                                                       │
│ ☑ Enable image cleaner (Eraser)                                   │
│ ⚡ Automatically removes unused container images from nodes.       │
│                                                                       │
│ ── Defender ──                                                      │
│ ☑ Enable Microsoft Defender for Containers                       │
│ ⚡ Runtime threat detection, vulnerability scanning,               │
│   suspicious activity alerts.                                   │
│                                                                       │
│ ── API server ──                                                   │
│ API server authorized IP ranges: [203.0.113.0/24]                │
│ ⚡ Restrict who can reach the K8s API server.                      │
│   Set to office IP + CI/CD IPs. Block all else.                 │
│                                                                       │
│ ── Maintenance ──                                                   │
│ Planned maintenance window:                                         │
│ ☑ Enable                                                           │
│ Day: [Tuesday ▼]  Start: [02:00 UTC]  Duration: [4 hours]      │
│ ⚡ Upgrades and patches happen within this window only.            │
│                                                                       │
│ [Review + create] → [Create]                                      │
│ ⚡ Cluster creation: ~5-10 minutes.                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: AKS Networking Deep Dive

```
┌─────────────────────────────────────────────────────────────────────┐
│           INGRESS & LOAD BALANCING                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Exposing services to internet:                                      │
│                                                                       │
│ Option 1: LoadBalancer Service (L4 — TCP/UDP)                     │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ kind: Service                                                │  │
│ │ spec:                                                         │  │
│ │   type: LoadBalancer                                         │  │
│ │   ports:                                                      │  │
│ │     - port: 80                                               │  │
│ │ → Creates Azure Standard Load Balancer with public IP       │  │
│ │ → One LB per service (expensive if many services!)          │  │
│ │                                                              │  │
│ │ Internal LB:                                                 │  │
│ │ metadata:                                                     │  │
│ │   annotations:                                               │  │
│ │     service.beta.kubernetes.io/azure-load-balancer-internal: │  │
│ │       "true"                                                  │  │
│ │ → Internal IP only (no internet access)                     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Option 2: NGINX Ingress Controller (L7 — HTTP/HTTPS) ✅           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ # Install via Helm                                           │  │
│ │ helm install ingress-nginx ingress-nginx/ingress-nginx \    │  │
│ │   --namespace ingress --create-namespace \                   │  │
│ │   --set controller.service.annotations."service\.beta\      │  │
│ │   .kubernetes\.io/azure-load-balancer-health-probe-request\ │  │
│ │   -path"="/healthz"                                          │  │
│ │                                                              │  │
│ │ Ingress resource:                                            │  │
│ │ apiVersion: networking.k8s.io/v1                             │  │
│ │ kind: Ingress                                                │  │
│ │ metadata:                                                     │  │
│ │   annotations:                                               │  │
│ │     cert-manager.io/cluster-issuer: letsencrypt-prod        │  │
│ │ spec:                                                         │  │
│ │   ingressClassName: nginx                                    │  │
│ │   tls:                                                        │  │
│ │     - hosts: [app.example.com]                              │  │
│ │       secretName: tls-app                                    │  │
│ │   rules:                                                      │  │
│ │     - host: app.example.com                                 │  │
│ │       http:                                                   │  │
│ │         paths:                                               │  │
│ │           - path: /                                          │  │
│ │             pathType: Prefix                                 │  │
│ │             backend:                                          │  │
│ │               service:                                       │  │
│ │                 name: web-app                                │  │
│ │                 port: { number: 80 }                         │  │
│ │           - path: /api                                       │  │
│ │             pathType: Prefix                                 │  │
│ │             backend:                                          │  │
│ │               service:                                       │  │
│ │                 name: api-backend                            │  │
│ │                 port: { number: 80 }                         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Option 3: Application Gateway Ingress Controller (AGIC)           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Azure Application Gateway as Ingress:                       │  │
│ │ ├── WAF (Web Application Firewall) built-in                │  │
│ │ ├── SSL offloading                                          │  │
│ │ ├── Cookie-based session affinity                           │  │
│ │ ├── URL path-based routing                                  │  │
│ │ ├── Multi-site hosting                                      │  │
│ │ └── Autoscaling (v2)                                        │  │
│ │                                                              │  │
│ │ annotations:                                                 │  │
│ │   kubernetes.io/ingress.class: azure/application-gateway   │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Network Policies (pod-to-pod traffic):                             │
│ apiVersion: networking.k8s.io/v1                                   │
│ kind: NetworkPolicy                                                 │
│ metadata:                                                            │
│   name: deny-all                                                    │
│   namespace: backend                                                │
│ spec:                                                                │
│   podSelector: {}    # All pods in namespace                      │
│   policyTypes:                                                      │
│     - Ingress                                                       │
│     - Egress                                                        │
│   ingress: []        # Deny all incoming                           │
│   egress: []         # Deny all outgoing                           │
│                                                                       │
│ ⚡ Default deny + explicit allow = Zero Trust networking!          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Azure AD (Entra ID) RBAC

```
┌─────────────────────────────────────────────────────────────────────┐
│           ENTRA ID + KUBERNETES RBAC                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ AKS + Entra ID integration:                                        │
│ ├── Users authenticate with Azure AD credentials (SSO!)          │
│ ├── No kubeconfig files with static tokens                      │
│ ├── Group-based access (AD groups → K8s RBAC roles)             │
│ ├── Conditional Access policies apply (MFA, device compliance)  │
│ └── Azure RBAC or Kubernetes RBAC (or both)                     │
│                                                                       │
│ Azure RBAC roles for AKS:                                          │
│ ┌──────────────────────────────────────────────┬─────────────────┐│
│ │ Role                                          │ Permissions      ││
│ ├──────────────────────────────────────────────┼─────────────────┤│
│ │ Azure Kubernetes Service Cluster Admin Role  │ Full admin       ││
│ │ Azure Kubernetes Service Cluster User Role   │ Get kubeconfig  ││
│ │ Azure Kubernetes Service RBAC Admin          │ K8s RBAC admin  ││
│ │ Azure Kubernetes Service RBAC Cluster Admin  │ Full K8s admin  ││
│ │ Azure Kubernetes Service RBAC Reader         │ Read-only K8s   ││
│ │ Azure Kubernetes Service RBAC Writer         │ Read-write K8s  ││
│ └──────────────────────────────────────────────┴─────────────────┘│
│                                                                       │
│ How it works:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Developer → az aks get-credentials --name aks-prod         │  │
│ │          → kubectl get pods → Browser opens → Azure AD login│  │
│ │          → Token issued → K8s API validates token          │  │
│ │          → Check Azure RBAC or K8s RBAC → Allow/Deny       │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Namespace-level access (common pattern):                           │
│ ├── AD Group: "team-backend" → RoleBinding in namespace "backend"│
│ ├── AD Group: "team-frontend" → RoleBinding in namespace "web"  │
│ ├── AD Group: "sre-team" → ClusterRoleBinding (full access)    │
│ └── AD Group: "readonly" → ClusterRoleBinding (view only)       │
│                                                                       │
│ # Assign Azure RBAC role (namespace-scoped)                       │
│ az role assignment create \                                        │
│   --role "Azure Kubernetes Service RBAC Writer" \                │
│   --assignee-object-id <AD-GROUP-ID> \                            │
│   --scope "/subscriptions/.../managedClusters/aks-prod/          │
│            namespaces/backend"                                     │
│                                                                       │
│ Workload Identity (pods → Azure resources):                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ K8s ServiceAccount: app-sa (namespace: backend)             │  │
│ │     ↕ federated credential                                  │  │
│ │ User-Assigned Managed Identity: mi-app                      │  │
│ │     → role: Storage Blob Contributor                        │  │
│ │     → role: Key Vault Secrets User                          │  │
│ │                                                              │  │
│ │ Pod with serviceAccountName: app-sa                        │  │
│ │ → Gets Azure AD token → Access Storage, Key Vault          │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ # Setup Workload Identity                                          │
│ # 1. Create managed identity                                      │
│ az identity create --name mi-app --resource-group rg-aks-prod   │
│                                                                       │
│ # 2. Create federated credential                                  │
│ az identity federated-credential create \                         │
│   --name fc-app-sa \                                               │
│   --identity-name mi-app \                                         │
│   --resource-group rg-aks-prod \                                  │
│   --issuer $(az aks show -n aks-prod -g rg-aks-prod \            │
│     --query "oidcIssuerProfile.issuerUrl" -o tsv) \               │
│   --subject "system:serviceaccount:backend:app-sa"               │
│                                                                       │
│ # 3. Annotate K8s ServiceAccount                                  │
│ kubectl annotate sa app-sa -n backend \                          │
│   azure.workload.identity/client-id=$(az identity show \         │
│   -n mi-app -g rg-aks-prod --query clientId -o tsv)             │
│                                                                       │
│ # 4. Label the ServiceAccount                                     │
│ kubectl label sa app-sa -n backend \                             │
│   azure.workload.identity/use=true                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: AKS Autoscaling

```
┌─────────────────────────────────────────────────────────────────────┐
│           AUTOSCALING LAYERS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Layer 1: POD autoscaling                                            │
│ ├── HPA (Horizontal Pod Autoscaler): Scale replicas               │
│ │   kubectl autoscale deployment web --cpu-percent=70 \          │
│ │     --min=3 --max=20                                            │
│ │                                                                 │
│ ├── VPA (Vertical Pod Autoscaler): Resize pod resources          │
│ │   AKS addon: az aks update --enable-vpa                       │
│ │                                                                 │
│ └── KEDA (Kubernetes Event Driven Autoscaling): ✅                │
│     AKS addon: az aks update --enable-keda                       │
│     Scale based on: Azure Queue, Service Bus, Prometheus, HTTP  │
│     Can scale to zero! (unlike HPA)                             │
│                                                                       │
│     apiVersion: keda.sh/v1alpha1                                  │
│     kind: ScaledObject                                              │
│     spec:                                                            │
│       scaleTargetRef:                                               │
│         name: worker-deployment                                    │
│       minReplicaCount: 0     ← Scale to zero!                    │
│       maxReplicaCount: 30                                          │
│       triggers:                                                     │
│         - type: azure-servicebus                                  │
│           metadata:                                                 │
│             queueName: orders                                      │
│             messageCount: "5"  ← Scale when > 5 messages        │
│                                                                       │
│ Layer 2: NODE autoscaling                                           │
│ ├── Cluster Autoscaler (built into AKS node pool config):        │
│ │   Pod pending → Add node (2-5 minutes).                       │
│ │   Node empty → Remove node (after scale-down delay).          │
│ │   Configured per pool: min/max node count.                    │
│ │                                                                 │
│ │   Profile settings (cluster-wide):                             │
│ │   az aks update --cluster-autoscaler-profile \                │
│ │     scale-down-delay-after-add=10m \                          │
│ │     scale-down-unneeded-time=10m \                             │
│ │     max-graceful-termination-sec=600                           │
│ │                                                                 │
│ └── Node Autoprovision (NAP — preview):                          │
│     AKS auto-creates node pools based on pending pod needs.     │
│     Similar to GKE NAP / Karpenter on EKS.                     │
│                                                                       │
│ Scaling flow:                                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Traffic/Events ↑ → KEDA/HPA: More pods → No room →         │  │
│ │ Cluster Autoscaler: Add node (~3 min) → Pods scheduled.    │  │
│ │                                                              │  │
│ │ Traffic/Events ↓ → KEDA/HPA: Fewer pods → Nodes empty →   │  │
│ │ Cluster Autoscaler: Remove node (after cooldown).          │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform

```hcl
# AKS Cluster
resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-prod"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "aks-prod"
  kubernetes_version  = "1.30.4"
  sku_tier            = "Standard"  # Free, Standard, Premium

  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v5"
    min_count           = 2
    max_count           = 5
    auto_scaling_enabled = true
    os_sku              = "AzureLinux"
    zones               = [1, 2, 3]
    max_pods            = 30
    vnet_subnet_id      = azurerm_subnet.aks.id
    only_critical_addons_enabled = true  # System pool only

    node_labels = {
      role = "system"
    }

    upgrade_settings {
      max_surge = "33%"
    }
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"         # azure (CNI), kubenet, none
    network_policy    = "azure"         # azure, calico, none
    load_balancer_sku = "standard"
    service_cidr      = "10.1.0.0/16"
    dns_service_ip    = "10.1.0.10"
  }

  azure_active_directory_role_based_access_control {
    azure_rbac_enabled = true
    tenant_id          = data.azurerm_client_config.current.tenant_id
  }

  oidc_issuer_enabled       = true
  workload_identity_enabled = true

  key_vault_secrets_provider {
    secret_rotation_enabled = true
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }

  microsoft_defender {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }

  maintenance_window_auto_upgrade {
    frequency   = "Weekly"
    interval    = 1
    day_of_week = "Tuesday"
    start_time  = "02:00"
    duration    = 4
    utc_offset  = "+05:30"
  }

  auto_scaler_profile {
    scale_down_delay_after_add = "10m"
    scale_down_unneeded        = "10m"
    max_graceful_termination_sec = 600
  }

  tags = {
    environment = "prod"
  }
}

# User Node Pool — General workloads
resource "azurerm_kubernetes_cluster_node_pool" "general" {
  name                  = "general"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v5"
  min_count             = 2
  max_count             = 20
  auto_scaling_enabled  = true
  os_sku                = "AzureLinux"
  zones                 = [1, 2, 3]
  max_pods              = 30
  vnet_subnet_id        = azurerm_subnet.aks.id

  node_labels = {
    role = "general"
    env  = "prod"
  }

  upgrade_settings {
    max_surge = "33%"
  }
}

# User Node Pool — Spot (for batch/workers)
resource "azurerm_kubernetes_cluster_node_pool" "spot" {
  name                  = "spot"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v5"
  min_count             = 0
  max_count             = 20
  auto_scaling_enabled  = true
  priority              = "Spot"
  eviction_policy       = "Delete"
  spot_max_price        = -1  # Pay up to on-demand price
  os_sku                = "AzureLinux"
  zones                 = [1, 2, 3]
  vnet_subnet_id        = azurerm_subnet.aks.id

  node_labels = {
    "kubernetes.azure.com/scalesetpriority" = "spot"
    role = "spot"
  }

  node_taints = [
    "kubernetes.azure.com/scalesetpriority=spot:NoSchedule"
  ]
}

# User Node Pool — GPU
resource "azurerm_kubernetes_cluster_node_pool" "gpu" {
  name                  = "gpu"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_NC4as_T4_v3"
  min_count             = 0
  max_count             = 4
  auto_scaling_enabled  = true
  os_sku                = "AzureLinux"
  zones                 = [1, 2, 3]
  vnet_subnet_id        = azurerm_subnet.aks.id

  node_labels = {
    role = "gpu"
  }

  node_taints = [
    "nvidia.com/gpu=true:NoSchedule"
  ]
}

# AKS Subnet
resource "azurerm_subnet" "aks" {
  name                 = "subnet-aks"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.0.0/16"]
}

# ACR attachment
resource "azurerm_role_assignment" "acr_pull" {
  principal_id         = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
  role_definition_name = "AcrPull"
  scope                = azurerm_container_registry.main.id
}

# Workload Identity — Managed Identity for pods
resource "azurerm_user_assigned_identity" "app" {
  name                = "mi-app"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_federated_identity_credential" "app" {
  name                = "fc-app-sa"
  resource_group_name = azurerm_resource_group.main.name
  parent_id           = azurerm_user_assigned_identity.app.id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = azurerm_kubernetes_cluster.main.oidc_issuer_url
  subject             = "system:serviceaccount:backend:app-sa"
}

resource "azurerm_role_assignment" "app_storage" {
  principal_id         = azurerm_user_assigned_identity.app.principal_id
  role_definition_name = "Storage Blob Data Contributor"
  scope                = azurerm_storage_account.main.id
}

resource "azurerm_role_assignment" "app_keyvault" {
  principal_id         = azurerm_user_assigned_identity.app.principal_id
  role_definition_name = "Key Vault Secrets User"
  scope                = azurerm_key_vault.main.id
}
```

---

## Part 7: Bicep

```bicep
resource aksCluster 'Microsoft.ContainerService/managedClusters@2024-01-01' = {
  name: 'aks-prod'
  location: resourceGroup().location
  sku: { name: 'Base', tier: 'Standard' }
  identity: { type: 'SystemAssigned' }
  properties: {
    kubernetesVersion: '1.30.4'
    dnsPrefix: 'aks-prod'
    agentPoolProfiles: [
      {
        name: 'system'
        count: 2
        vmSize: 'Standard_D4s_v5'
        osSKU: 'AzureLinux'
        mode: 'System'
        enableAutoScaling: true
        minCount: 2
        maxCount: 5
        availabilityZones: ['1', '2', '3']
        vnetSubnetID: aksSubnet.id
        maxPods: 30
        nodeTaints: ['CriticalAddonsOnly=true:NoSchedule']
      }
      {
        name: 'general'
        count: 2
        vmSize: 'Standard_D4s_v5'
        osSKU: 'AzureLinux'
        mode: 'User'
        enableAutoScaling: true
        minCount: 2
        maxCount: 20
        availabilityZones: ['1', '2', '3']
        vnetSubnetID: aksSubnet.id
        maxPods: 30
      }
    ]
    networkProfile: {
      networkPlugin: 'azure'
      networkPolicy: 'azure'
      loadBalancerSku: 'standard'
      serviceCidr: '10.1.0.0/16'
      dnsServiceIP: '10.1.0.10'
    }
    aadProfile: {
      managed: true
      enableAzureRBAC: true
      tenantID: subscription().tenantId
    }
    oidcIssuerProfile: { enabled: true }
    securityProfile: {
      workloadIdentity: { enabled: true }
      defender: {
        securityMonitoring: { enabled: true }
        logAnalyticsWorkspaceResourceId: logAnalytics.id
      }
    }
    addonProfiles: {
      omsagent: {
        enabled: true
        config: { logAnalyticsWorkspaceResourceID: logAnalytics.id }
      }
      azureKeyvaultSecretsProvider: {
        enabled: true
        config: { enableSecretRotation: 'true' }
      }
    }
  }
}
```

---

## Part 8: az CLI Reference

```bash
# ═══ Cluster management ═══

# Create AKS cluster
az aks create \
  --resource-group rg-aks-prod \
  --name aks-prod \
  --location centralindia \
  --kubernetes-version 1.30.4 \
  --tier standard \
  --node-count 2 \
  --node-vm-size Standard_D4s_v5 \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 5 \
  --network-plugin azure \
  --network-policy azure \
  --vnet-subnet-id /subscriptions/.../subnets/subnet-aks \
  --service-cidr 10.1.0.0/16 \
  --dns-service-ip 10.1.0.10 \
  --zones 1 2 3 \
  --enable-aad \
  --enable-azure-rbac \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --attach-acr acrmyregistry \
  --enable-addons monitoring \
  --workspace-resource-id /subscriptions/.../workspaces/law-aks-prod \
  --generate-ssh-keys

# Get credentials (configure kubectl)
az aks get-credentials \
  --resource-group rg-aks-prod \
  --name aks-prod

# List clusters
az aks list --output table

# Show cluster details
az aks show \
  --resource-group rg-aks-prod \
  --name aks-prod

# Upgrade cluster
az aks upgrade \
  --resource-group rg-aks-prod \
  --name aks-prod \
  --kubernetes-version 1.31.0

# Get available upgrades
az aks get-upgrades \
  --resource-group rg-aks-prod \
  --name aks-prod

# Start/stop cluster (dev/test cost saving)
az aks stop --resource-group rg-aks-prod --name aks-dev
az aks start --resource-group rg-aks-prod --name aks-dev

# ═══ Node pools ═══

# Add user node pool
az aks nodepool add \
  --resource-group rg-aks-prod \
  --cluster-name aks-prod \
  --name general \
  --node-count 2 \
  --node-vm-size Standard_D4s_v5 \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 20 \
  --zones 1 2 3 \
  --mode User \
  --labels role=general env=prod

# Add Spot node pool
az aks nodepool add \
  --resource-group rg-aks-prod \
  --cluster-name aks-prod \
  --name spot \
  --node-count 0 \
  --node-vm-size Standard_D4s_v5 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 20 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --labels role=spot \
  --node-taints "kubernetes.azure.com/scalesetpriority=spot:NoSchedule"

# Scale node pool
az aks nodepool scale \
  --resource-group rg-aks-prod \
  --cluster-name aks-prod \
  --name general \
  --node-count 5

# Upgrade node pool
az aks nodepool upgrade \
  --resource-group rg-aks-prod \
  --cluster-name aks-prod \
  --name general \
  --kubernetes-version 1.30.4

# Delete node pool
az aks nodepool delete \
  --resource-group rg-aks-prod \
  --cluster-name aks-prod \
  --name spot

# List node pools
az aks nodepool list \
  --resource-group rg-aks-prod \
  --cluster-name aks-prod \
  --output table

# ═══ Addons ═══

# Enable KEDA
az aks update --resource-group rg-aks-prod --name aks-prod --enable-keda

# Enable VPA
az aks update --resource-group rg-aks-prod --name aks-prod --enable-vpa

# Enable Azure Policy
az aks update --resource-group rg-aks-prod --name aks-prod \
  --enable-addons azure-policy

# Attach ACR
az aks update --resource-group rg-aks-prod --name aks-prod \
  --attach-acr acrmyregistry

# ═══ kubectl (after get-credentials) ═══

# Verify
kubectl get nodes
kubectl get pods -A

# Deploy app
kubectl create namespace backend
kubectl apply -f deployment.yaml -n backend

# ═══ Maintenance ═══

# Set maintenance window
az aks maintenancewindow update \
  --resource-group rg-aks-prod \
  --cluster-name aks-prod \
  --name default \
  --day-of-week Tuesday \
  --start-time 02:00 \
  --duration 4

# ═══ Delete cluster ═══
az aks delete --resource-group rg-aks-prod --name aks-prod --yes
```

---

## Part 9: Real-World Patterns

### Startup

```
Simple AKS for a small team:

Cluster: aks-prod (Standard tier, 99.95% SLA)
├── Region: Central India, Zones: 1, 2, 3
├── K8s version: 1.30 (auto-upgrade: patch)
├── Auth: Entra ID + Azure RBAC
├── Networking: Azure CNI, Standard LB
└── Cost: $73/mo (control plane) + nodes

Node pools:
├── system (Standard_D2s_v5, 2 nodes, autoscale 2-3)
│   ⚡ Small, system pods only (CoreDNS, metrics)
├── general (Standard_D4s_v5, 2 nodes, autoscale 2-8)
│   ⚡ All app workloads
└── Total: ~$400-800/month (4-10 nodes)

Workloads:
├── Namespace: frontend (React SSR, 2-5 pods, HPA on CPU)
├── Namespace: backend (API, 3-8 pods, HPA on CPU)
├── Namespace: workers (queue consumers, 1-5 pods, KEDA)
└── Namespace: ingress (NGINX Ingress Controller)

Ingress: NGINX + cert-manager (Let's Encrypt auto TLS)
Secrets: Key Vault + CSI Secret Store Driver
Images: ACR (attached, no imagePullSecrets)
CI/CD: GitHub Actions → ACR → kubectl apply

Cost: $400-1,000/month
```

### Mid-Size

```
AKS with multiple node pools and advanced features:

Cluster: aks-prod (Standard tier)
├── Azure CNI, Network Policy: Azure
├── Private cluster: No (but API server authorized IPs)
├── Entra ID RBAC: Team-based namespace access
├── KEDA addon enabled
├── Workload Identity enabled
└── Defender for Containers enabled

Node pools:
├── system (Standard_D2s_v5, 3 nodes fixed, AzureLinux)
├── general (Standard_D4s_v5, 3-15 autoscale, AzureLinux)
├── spot (Standard_D4s_v5, 0-20 autoscale, Spot, tainted)
├── highcpu (Standard_F8s_v2, 0-8 autoscale, for compute)
└── Total: ~$2,000-6,000/month

Namespaces & access:
├── frontend: Team Frontend (Entra group → RBAC Writer)
├── backend: Team Backend (Entra group → RBAC Writer)
├── data: Team Data (Entra group → RBAC Writer)
├── workers: Team Backend (shared)
├── monitoring: SRE Team only
├── ingress: SRE Team only
└── SRE: ClusterAdmin (Entra group → RBAC Cluster Admin)

Networking:
├── NGINX Ingress Controller (2 replicas, HA)
│   app.company.com → frontend
│   api.company.com → backend
├── cert-manager: Auto TLS (Let's Encrypt)
├── NetworkPolicies: Default deny per namespace
├── Internal LB: For private services
└── DNS: Azure Private DNS zones for internal

Scaling:
├── HPA: All web services (CPU > 70% → scale)
├── KEDA: Workers (Azure Service Bus queue depth)
│   0 messages → 0 pods (scale to zero!)
│   100 messages → 10 pods (burst)
├── Cluster Autoscaler: Add nodes for pending pods
└── Spot pool: Batch jobs tolerate spot interruption

Observability:
├── Container Insights: CPU, memory, logs → Log Analytics
├── Managed Prometheus + Managed Grafana
├── Alerts: Node NotReady, Pod CrashLoopBackOff, 5xx rate
├── KQL queries for troubleshooting
└── Workbooks: Custom dashboards in Azure Monitor

CI/CD:
├── GitHub Actions → Build → Push to ACR → Deploy
├── Staging cluster: aks-staging (Free tier, smaller)
├── Canary: Deploy to 1 pod → monitor → scale up
└── Rollback: kubectl rollout undo

Cost: $2,000-6,000/month
```

### Enterprise

```
Multi-cluster AKS platform:

Clusters:
├── aks-prod-india (Central India, Standard tier)
│   ├── system pool: 3 × D4s_v5
│   ├── general pool: 5-30 × D8s_v5 (On-Demand)
│   ├── spot pool: 0-50 × D8s_v5 (Spot)
│   ├── gpu pool: 0-8 × NC4as_T4_v3 (GPU, tainted)
│   └── windows pool: 2-6 × D4s_v5 (Windows containers)
│
├── aks-prod-europe (West Europe, Standard tier)
│   └── Same config as India (active-active)
│
├── aks-staging (Central India, Free tier)
│   └── Smaller, for pre-prod testing
│
└── Azure Fleet Manager: Manage all clusters centrally

Networking:
├── Private cluster (API server in VNet, no public endpoint)
├── Azure CNI + Network Policy (Calico)
├── Application Gateway Ingress Controller (AGIC) + WAF
│   ├── WAF v2: OWASP rules, bot protection, rate limiting
│   ├── SSL offloading: Managed certificates
│   └── Multi-site: 20+ domains on one App Gateway
├── Azure Front Door: Global load balancing (India ↔ Europe)
├── Private Link: ACR, Key Vault, SQL via private endpoints
└── ExpressRoute: On-premises connectivity

Security:
├── Entra ID: All access via Azure AD (no local accounts!)
├── Conditional Access: MFA required for cluster access
├── Workload Identity: All pods use managed identities
├── Key Vault: All secrets (CSI driver, auto-rotation)
├── Azure Policy (Gatekeeper):
│   ├── Only images from approved ACR
│   ├── No privileged containers
│   ├── Resource limits required
│   ├── Read-only root filesystem
│   └── No hostNetwork/hostPID
├── Defender for Containers: Runtime protection
├── ACR: Private endpoint, vulnerability scanning
├── Network Policies: Default deny, namespace isolation
├── Pod Security Standards: Restricted
└── Audit: All K8s API calls → Log Analytics → SIEM

GitOps:
├── Flux v2 (AKS GitOps extension):
│   ├── All K8s manifests in Git
│   ├── Changes merged → auto-deployed to clusters
│   ├── Multi-cluster: Same repo → all clusters
│   └── Drift detection: Auto-reconcile
├── Helm charts: Standardized app deployment
└── Kustomize: Environment-specific overlays

Monitoring:
├── Azure Managed Prometheus (all clusters → central)
├── Azure Managed Grafana (dashboards)
├── Container Insights (Log Analytics)
├── SLOs per service (error budget tracking)
├── PagerDuty: Critical alerts
├── Azure Workbooks: Executive dashboards
└── Cost Management: Per-namespace cost allocation

Cost: $15,000-50,000/month (multi-cluster, GPU, multi-region)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Fully managed Kubernetes on Azure |
| Control plane | Free (Free tier), $73/mo (Standard), $438/mo (Premium) |
| SLA | None (Free), 99.95% (Standard + AZs), 99.9% (no AZs) |
| Networking | Kubenet, Azure CNI, Azure CNI Overlay |
| Node types | Linux (Ubuntu/AzureLinux), Windows, Spot, GPU |
| Auth | Entra ID (Azure AD) + Azure RBAC or K8s RBAC |
| Pod IAM | Workload Identity (Managed Identity federation) |
| Autoscaling | HPA, VPA, KEDA (scale to zero), Cluster Autoscaler |
| Ingress | NGINX, AGIC (App Gateway), or any K8s ingress |
| Secrets | Key Vault + CSI Secret Store Driver |
| Policy | Azure Policy (Gatekeeper/OPA) |
| Monitoring | Container Insights, Managed Prometheus + Grafana |
| GitOps | Flux v2 (AKS extension) |
| Private cluster | Yes (API server in VNet) |
| Multi-cluster | Azure Fleet Manager |
| Upgrade | None, Patch, Stable, Rapid, Node Image |
| AWS equivalent | EKS |
| GCP equivalent | GKE |

---

## What's Next?

In the next chapter, we'll cover Azure Container Apps — a serverless container platform built on top of AKS with KEDA and Dapr.

→ Next: [Chapter 20: Azure App Service](20-app-service.md)

---

*Last Updated: May 2026*
