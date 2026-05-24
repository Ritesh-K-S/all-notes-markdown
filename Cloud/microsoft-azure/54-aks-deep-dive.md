# Chapter 54: AKS Deep Dive

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Advanced Networking](#part-1-advanced-networking)
- [Part 2: Application Gateway Ingress Controller (AGIC)](#part-2-application-gateway-ingress-controller-agic)
- [Part 3: KEDA (Event-Driven Autoscaling)](#part-3-keda-event-driven-autoscaling)
- [Part 4: Workload Identity](#part-4-workload-identity)
- [Part 5: GitOps with Flux](#part-5-gitops-with-flux)
- [Part 6: Monitoring & Observability](#part-6-monitoring--observability)
- [Part 7: Security Best Practices](#part-7-security-best-practices)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

This chapter goes deeper into AKS beyond the basics covered in Chapter 19. We explore advanced networking, event-driven scaling, workload identity, GitOps, and production-grade security patterns.

```
What you'll learn:
├── Advanced Networking
│   ├── Network Policies (Calico/Azure)
│   ├── Azure CNI Overlay vs Pod Subnet
│   └── Service Mesh (Istio add-on)
├── AGIC (Application Gateway as Ingress)
├── KEDA (scale from 0 based on events)
├── Workload Identity (Azure AD for pods)
├── GitOps with Flux (declarative deployments)
├── Monitoring & Observability
└── Security best practices
```

---

## Part 1: Advanced Networking

```
Network Policies (control pod-to-pod traffic):
├── By default: ALL pods can talk to ALL other pods
├── Network policies restrict traffic (like NSGs for pods)
├── Two providers:
│   ├── Azure NPM: Azure's implementation
│   └── Calico: Open-source, more features
│
│ Example: Only allow frontend → backend, block everything else
│ apiVersion: networking.k8s.io/v1
│ kind: NetworkPolicy
│ metadata:
│   name: allow-frontend-to-backend
│ spec:
│   podSelector:
│     matchLabels:
│       app: backend
│   ingress:
│   - from:
│     - podSelector:
│         matchLabels:
│           app: frontend
│     ports:
│     - port: 8080

Azure CNI options:
├── Azure CNI: Each pod gets VNet IP (IP exhaustion risk)
├── Azure CNI Overlay: Pods get private IPs, overlay network
│   └── Best for large clusters (no IP exhaustion)
├── Azure CNI with Pod Subnet: Pods in dedicated subnet
└── Kubenet: Simple, NAT-based (limited features)

Service Mesh (Istio add-on):
├── AKS → Service mesh → [Enable Istio]
├── Provides: mTLS, traffic management, observability
├── No need to install Istio manually
└── Managed by Azure (auto-upgrades)
```

---

## Part 2: Application Gateway Ingress Controller (AGIC)

```
AGIC = Use Azure Application Gateway as Kubernetes Ingress

Without AGIC:
  Internet → Load Balancer → nginx ingress pod → Service → Pods

With AGIC:
  Internet → Application Gateway (L7, WAF) → Pods directly
  ├── No extra ingress controller pod needed
  ├── WAF protection built-in
  ├── SSL termination at App Gateway
  └── Auto-configures App Gateway from K8s Ingress resources

Enable:
  az aks enable-addons \
    --name aks-prod \
    --resource-group rg-k8s \
    --addons ingress-appgw \
    --appgw-name appgw-aks \
    --appgw-subnet-cidr 10.225.0.0/16

Ingress manifest:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: my-ingress
    annotations:
      kubernetes.io/ingress.class: azure/application-gateway
  spec:
    rules:
    - host: myapp.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: my-service
              port:
                number: 80
```

---

## Part 3: KEDA (Event-Driven Autoscaling)

```
KEDA = Kubernetes Event-Driven Autoscaling
Scale pods from 0 to N based on external events

Without KEDA: HPA scales based on CPU/memory only
With KEDA: Scale based on queue length, event hub lag, cron, etc.

Enable:
  az aks update --name aks-prod --resource-group rg-k8s --enable-keda

Example: Scale based on Azure Service Bus queue length
  apiVersion: keda.sh/v1alpha1
  kind: ScaledObject
  metadata:
    name: order-processor-scaler
  spec:
    scaleTargetRef:
      name: order-processor        # Deployment name
    minReplicaCount: 0             # Scale to zero!
    maxReplicaCount: 20
    triggers:
    - type: azure-servicebus
      metadata:
        queueName: orders
        namespace: sb-myapp
        messageCount: "5"          # 1 pod per 5 messages
      authenticationRef:
        name: sb-trigger-auth

Supported scalers (50+):
├── Azure: Service Bus, Event Hub, Storage Queue, Cosmos DB
├── AWS: SQS, CloudWatch, Kinesis
├── Databases: PostgreSQL, MySQL, Redis
├── Messaging: Kafka, RabbitMQ, NATS
├── HTTP: Prometheus metrics, external APIs
└── Cron: Schedule-based scaling
```

---

## Part 4: Workload Identity

```
Workload Identity = Give pods an Azure AD identity (no secrets!)

Old way: Store connection strings/keys in pod (insecure)
New way: Pod authenticates to Azure AD with federated token

Setup:
1. Enable on cluster:
   az aks update --name aks-prod --resource-group rg-k8s \
     --enable-oidc-issuer --enable-workload-identity

2. Create managed identity:
   az identity create --name id-order-api --resource-group rg-k8s

3. Create federated credential:
   az identity federated-credential create \
     --name fc-order-api \
     --identity-name id-order-api \
     --resource-group rg-k8s \
     --issuer <AKS_OIDC_ISSUER_URL> \
     --subject system:serviceaccount:default:sa-order-api

4. Create K8s service account with annotation:
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: sa-order-api
     annotations:
       azure.workload.identity/client-id: <MANAGED_IDENTITY_CLIENT_ID>

5. Use in pod:
   spec:
     serviceAccountName: sa-order-api
     containers:
     - name: order-api
       # Pod can now access Azure resources using DefaultAzureCredential!

6. Grant RBAC to managed identity:
   az role assignment create --assignee <IDENTITY_ID> \
     --role "Storage Blob Data Reader" \
     --scope /subscriptions/.../storageAccounts/stmyapp
```

---

## Part 5: GitOps with Flux

```
GitOps = Git repo is the source of truth for cluster state
Push to Git → Flux syncs changes to cluster automatically

Enable Flux:
  az k8s-extension create \
    --name flux \
    --cluster-name aks-prod \
    --resource-group rg-k8s \
    --cluster-type managedClusters \
    --extension-type microsoft.flux

Create Flux configuration:
  az k8s-configuration flux create \
    --name gitops-config \
    --cluster-name aks-prod \
    --resource-group rg-k8s \
    --cluster-type managedClusters \
    --url https://github.com/myorg/k8s-manifests \
    --branch main \
    --kustomization name=infra path=./clusters/prod

How it works:
  Developer pushes YAML to Git
       ↓
  Flux detects changes (polls every 1 min)
       ↓
  Flux applies changes to AKS cluster
       ↓
  Cluster state matches Git (self-healing!)

Benefits:
├── Auditable: Git history = deployment history
├── Rollback: git revert = cluster rollback
├── Multi-cluster: Same repo, different paths per cluster
└── Self-healing: Drift detected → auto-corrected
```

---

## Part 6: Monitoring & Observability

```
Container Insights (built-in):
├── AKS → Insights → Enable
├── Node/pod CPU, memory, disk metrics
├── Container logs (stdout/stderr)
├── Live data (tail logs in real-time)
└── Queries in Log Analytics (KQL)

Prometheus + Grafana (managed):
├── Azure Monitor managed service for Prometheus
├── AKS → Monitoring → Enable Prometheus
├── Grafana dashboards: Azure Managed Grafana
├── Pre-built dashboards for K8s workloads
└── Custom PromQL queries

Key KQL queries:
  // Pod restarts
  KubePodInventory
  | where RestartCount > 0
  | summarize count() by Name, Namespace
  | order by count_ desc

  // Container CPU usage
  Perf
  | where ObjectName == "K8SContainer"
  | where CounterName == "cpuUsageNanoCores"
  | summarize avg(CounterValue) by InstanceName, bin(TimeGenerated, 5m)

  // Failed pods
  KubePodInventory
  | where PodStatus == "Failed"
  | project PodName, Namespace, PodStatus, TimeGenerated
```

---

## Quick Reference

```
AKS Deep Dive Checklist:
├── Networking: Azure CNI (production) or Kubenet (dev/test)
├── Ingress: AGIC (Application Gateway) or NGINX
├── Autoscaling: HPA (pods) + Cluster Autoscaler (nodes) + KEDA (events)
├── Security: Workload Identity (not pod identity), Network Policies
├── GitOps: Flux v2 via Arc-enabled K8s extensions
├── Monitoring: Container Insights + Prometheus + Grafana
├── Node Pools: System (small, system pods) + User (workloads)
├── Upgrades: Planned maintenance windows, blue/green node pools

KEDA: Scale to/from zero based on queue depth, HTTP, custom metrics
Workload Identity: Azure AD tokens for pods (replaces pod-managed identity)
Network Policy: Calico (Azure or Calico) for pod-to-pod firewall rules
```

---

## What's Next?

Next chapter: [Chapter 55: Azure Service Fabric](55-service-fabric.md) — Microservices platform for stateful and stateless services.
  | summarize Restarts=max(RestartCount) by Name, Namespace

  // OOMKilled containers
  ContainerLog
  | where LogEntry contains "OOMKilled"

  // Node CPU pressure
  Perf
  | where ObjectName == "K8SNode" and CounterName == "cpuUsageNanoCores"
  | summarize AvgCPU=avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
```

---

## Part 7: Security Best Practices

```
Production AKS security checklist:
├── ☑ Private cluster (API server not public)
│   az aks create ... --enable-private-cluster
├── ☑ Azure AD authentication (disable local accounts)
│   az aks create ... --enable-aad --disable-local-accounts
├── ☑ Workload Identity (no secrets in pods)
├── ☑ Network policies (restrict pod traffic)
├── ☑ Azure Policy for AKS (enforce standards)
│   "Containers should not run as root"
│   "Containers should use allowed images only"
├── ☑ Defender for Containers (threat detection)
├── ☑ Image scanning (ACR vulnerability scanning)
├── ☑ Pod Security Admission (restrict privileged containers)
├── ☑ Secrets in Key Vault (CSI driver, not K8s secrets)
│   az aks enable-addons --addons azure-keyvault-secrets-provider
└── ☑ Restrict egress (Azure Firewall or NSG)
```

---

## Quick Reference

```
AKS Advanced Features:
├── Network Policies: Calico/Azure NPM (pod firewall rules)
├── AGIC: App Gateway as ingress (WAF, SSL, L7)
├── KEDA: Scale 0→N based on events (queue length, etc.)
├── Workload Identity: Pod → Azure AD → Azure resources (no secrets)
├── GitOps/Flux: Git repo → auto-sync to cluster
├── Service Mesh: Istio add-on (mTLS, traffic mgmt)
└── Monitoring: Container Insights + Managed Prometheus + Grafana

CNI choices: Kubenet | Azure CNI | CNI Overlay | CNI Pod Subnet

Security: Private cluster + AAD + Network policies + Workload Identity
         + Defender + Image scanning + Key Vault CSI + Pod Security
```

---

## What's Next?

Next chapter: [Chapter 55: Azure Service Fabric](55-service-fabric.md) — Microservices platform for stateful and stateless services.
