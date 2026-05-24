# Chapter 63: Azure Arc

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Arc Fundamentals](#part-1-azure-arc-fundamentals)
- [Part 2: Arc-Enabled Servers](#part-2-arc-enabled-servers)
- [Part 3: Arc-Enabled Kubernetes](#part-3-arc-enabled-kubernetes)
- [Part 4: Arc-Enabled Data Services](#part-4-arc-enabled-data-services)
- [Part 5: Governance at Scale](#part-5-governance-at-scale)
- [Part 6: az CLI Reference](#part-6-az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Arc extends Azure management to any infrastructure — on-premises servers, other clouds (AWS, GCP), and edge locations. It lets you manage non-Azure resources using the same Azure tools (Portal, Policy, RBAC, Monitor, Defender).

```
What you'll learn:
├── Azure Arc Fundamentals
│   ├── Why Arc (single pane of glass)
│   ├── What can be Arc-enabled
│   └── How it works (Connected Machine Agent)
├── Arc-Enabled Servers
├── Arc-Enabled Kubernetes
├── Arc-Enabled Data Services
├── Governance at Scale (Policy, RBAC across everything)
├── az CLI reference
└── Quick reference
```

---

## Part 1: Azure Arc Fundamentals

```
Problem: You have servers everywhere
├── Azure VMs
├── On-premises data center
├── AWS EC2 instances
├── GCP Compute instances
├── Edge locations (stores, factories)
└── Different management tools for each!

Solution: Azure Arc
├── Connect ALL servers to Azure
├── Manage everything from Azure Portal
├── Apply Azure Policy everywhere
├── Use Defender for Cloud everywhere
├── Deploy with Azure tools everywhere
└── One pane of glass!

How it works:
┌──────────────────────────────────────────────┐
│ Azure                                          │
│ ┌──────────────────────────────────────────┐ │
│ │ Azure Resource Manager                     │ │
│ │ (Portal, CLI, Policy, RBAC, Monitor)     │ │
│ └────────────────┬───────────────────────────┘ │
│                  │ (outbound HTTPS)             │
│ ┌────────────────┼───────────────────────────┐ │
│ │ Arc-enabled    │                             │ │
│ │ resources      ▼                             │ │
│ │ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐      │ │
│ │ │On-prem│ │ AWS  │ │ GCP  │ │ Edge │      │ │
│ │ │server │ │ EC2  │ │ VM   │ │device│      │ │
│ │ │[Agent]│ │[Agent]│ │[Agent]│ │[Agent]│    │ │
│ │ └──────┘ └──────┘ └──────┘ └──────┘      │ │
│ └────────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘

Agent: Connected Machine Agent (lightweight)
├── Outbound HTTPS only (no inbound ports needed)
├── Registers server as Azure resource
├── Sends metadata (OS, CPU, hostname)
└── Receives configuration (policies, extensions)

What can be Arc-enabled:
├── Servers (Windows/Linux, any location)
├── Kubernetes clusters (any K8s distribution)
├── SQL Server instances (on-premises)
├── Data services (SQL MI, PostgreSQL on any K8s)
├── VMware vSphere (manage VMs from Azure)
└── SCVMM (System Center Virtual Machine Manager)
```

---

## Part 2: Arc-Enabled Servers

```
Console → Azure Arc → Servers → [+ Add]

Methods to onboard:
├── Single server: Generate script → Run on server
├── Multiple servers: Script with service principal
├── At scale: Group Policy, Ansible, ConfigMgr
└── AWS/GCP: Use Connector (auto-onboard)

Single server onboard:
1. Azure Arc → Servers → [+ Add] → Generate script
2. Run the script on the server:

   # Windows (PowerShell)
   Invoke-WebRequest -Uri https://aka.ms/azcmagent-windows -OutFile install.ps1
   .\install.ps1
   azcmagent connect --resource-group rg-arc \
     --tenant-id <tenant> --subscription-id <sub>

   # Linux
   wget https://aka.ms/azcmagent -O install.sh
   bash install.sh
   azcmagent connect --resource-group rg-arc \
     --tenant-id <tenant> --subscription-id <sub>

3. Server appears in Azure Portal as a resource!

What you can do with Arc-enabled servers:
├── Azure Policy: Apply policies (require tags, audit configs)
├── Azure Monitor: Collect metrics and logs
├── Defender for Cloud: Security recommendations, threat detection
├── Update Management: Patch management
├── Azure Automation: Run scripts, DSC configurations
├── VM extensions: Install Log Analytics, Custom Script, etc.
├── RBAC: Control who can manage the server
├── Tags: Organize and track costs
└── Microsoft Sentinel: SIEM integration
```

---

## Part 3: Arc-Enabled Kubernetes

```
Connect any Kubernetes cluster to Azure:
├── On-premises K8s (kubeadm, Rancher, etc.)
├── AWS EKS
├── GCP GKE
├── Edge K3s clusters
└── Any CNCF-conformant K8s

Onboard:
  az connectedk8s connect \
    --name my-onprem-cluster \
    --resource-group rg-arc \
    --location centralindia

What you can do:
├── GitOps (Flux): Deploy apps from Git repo
│   az k8s-configuration flux create ...
├── Azure Policy for K8s: Enforce pod security, images
├── Azure Monitor: Container Insights for the cluster
├── Defender for Containers: Threat detection
├── Open Service Mesh: Service mesh management
├── Azure Key Vault Secrets Provider: Mount secrets
└── Azure App Service / Functions / Logic Apps on K8s!
    Deploy Azure PaaS services on YOUR cluster!

Arc-enabled App Service:
├── Run App Service on your own K8s cluster
├── Use Azure Portal to manage apps
├── Same deployment slots, scaling features
└── Great for edge scenarios (data sovereignty)
```

---

## Part 4: Arc-Enabled Data Services

```
Run Azure data services on YOUR infrastructure:

Available services:
├── Azure Arc-enabled SQL Managed Instance
│   └── Full SQL MI running on your K8s cluster
├── Azure Arc-enabled PostgreSQL (Hyperscale)
│   └── PostgreSQL with scale-out on your K8s
└── Azure Arc-enabled SQL Server
    └── Connect existing SQL Server instances

Benefits:
├── Always current: Auto-updates, always latest version
├── Elastic scale: Scale up/down on demand
├── Unified management: Azure Portal for all databases
├── Cloud billing: Pay via Azure subscription
└── Disconnected scenarios: Works with limited connectivity

Deployment:
  az arcdata dc create \
    --name arc-dc \
    --k8s-namespace arc \
    --connectivity-mode indirect \
    --resource-group rg-arc \
    --location centralindia

Connectivity modes:
├── Direct: Always connected to Azure (full features)
└── Indirect: Intermittent connection (edge scenarios)
```

---

## Part 5: Governance at Scale

```
Apply Azure governance to ALL infrastructure:

Azure Policy across everything:
├── "All servers must have antivirus" (Arc + Azure VMs)
├── "Tag all resources" (Arc servers + Azure resources)
├── "Audit servers without backup" (everywhere)
└── One policy → Applies to Azure VMs AND Arc servers

RBAC across everything:
├── "SRE team can manage all servers" (Azure + on-prem)
├── Same role assignments work for Arc resources
└── Consistent access control everywhere

Defender for Cloud:
├── Secure Score includes Arc-enabled servers
├── Recommendations for on-prem servers
├── Vulnerability scanning
└── Threat protection alerts

Update Manager:
├── Azure Update Manager → Works with Arc servers
├── Schedule patches for on-prem servers
├── Compliance reporting
└── Pre/post scripts for maintenance windows

Example: Govern hybrid environment
  Management Group
  └── Subscription
      ├── Azure VMs (native)           ← Policy + RBAC + Monitor
      ├── Arc Servers (on-prem)        ← Same Policy + RBAC + Monitor!
      └── Arc K8s (AWS EKS)            ← Same governance!
```

---

## Part 6: az CLI Reference

```bash
# Install Arc extension
az extension add --name connectedmachine
az extension add --name connectedk8s

# Onboard server (generates script)
az connectedmachine connect \
  --resource-group rg-arc \
  --name my-server \
  --location centralindia

# List Arc servers
az connectedmachine list --resource-group rg-arc -o table

# Show Arc server details
az connectedmachine show --name my-server --resource-group rg-arc

# Connect Kubernetes cluster
az connectedk8s connect \
  --name my-k8s-cluster \
  --resource-group rg-arc

# List Arc K8s clusters
az connectedk8s list --resource-group rg-arc -o table

# Enable Azure Monitor for Arc server
az connectedmachine extension create \
  --machine-name my-server \
  --resource-group rg-arc \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --type AzureMonitorLinuxAgent

# List extensions on Arc server
az connectedmachine extension list \
  --machine-name my-server \
  --resource-group rg-arc -o table

# Disconnect (remove) Arc server
az connectedmachine delete --name my-server --resource-group rg-arc --yes

# Disconnect Arc K8s cluster
az connectedk8s delete --name my-k8s-cluster --resource-group rg-arc --yes
```

---

## Real-World Patterns

### Pattern 1: Unified Governance Across Clouds

```
┌─────────────────────────────────────────────────┐
│  Multi-Cloud Governance with Azure Arc          │
├─────────────────────────────────────────────────┤
│                                                 │
│  Azure Portal (single pane of glass)            │
│       │                                         │
│  ┌────┼────────────┬────────────┐               │
│  ▼    ▼            ▼            ▼               │
│  Azure VMs   AWS EC2       GCP VMs   On-prem   │
│  (native)    (Arc agent)   (Arc)     (Arc)     │
│                                                 │
│  Apply uniformly:                               │
│  ├── Azure Policy (compliance)                  │
│  ├── RBAC (access control)                      │
│  ├── Azure Monitor (monitoring)                 │
│  ├── Defender for Cloud (security)              │
│  └── Update Management (patching)               │
│                                                 │
│  Result: Same governance for ALL servers,       │
│  regardless of where they run.                  │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Azure Arc = Extend Azure management to any infrastructure

Arc-enabled Servers: Manage on-prem/other-cloud VMs from Azure
├── Install Connected Machine Agent
├── Get: Policy, RBAC, Monitor, Defender, Update Management
└── Supported: Windows, Linux (physical or virtual)

Arc-enabled Kubernetes: Manage any K8s from Azure
├── Connect with az connectedk8s connect
├── Get: GitOps, Policy, Monitor, Defender
└── Supported: AKS, EKS, GKE, Rancher, on-prem K8s

Arc-enabled Data Services: Run Azure SQL/PostgreSQL anywhere
├── Azure SQL Managed Instance (on-prem)
├── Azure Database for PostgreSQL (on-prem)
└── Managed by Azure, runs on your infrastructure

Arc-enabled App Services: Run App Service/Functions/Logic Apps on K8s

Use when: Multi-cloud, hybrid, on-prem governance
Not a replacement for: Azure Stack (full Azure on-prem)
```

---

## What's Next?

Next chapter: [Chapter 64: Azure Stack & Hybrid Solutions](64-azure-stack-hybrid.md) — Run Azure services in your datacenter with Azure Stack Hub, HCI, and Edge.
az connectedmachine list --resource-group rg-arc -o table

# Connect K8s cluster
az connectedk8s connect \
  --name my-k8s-cluster \
  --resource-group rg-arc \
  --location centralindia

# Enable GitOps on Arc K8s
az k8s-configuration flux create \
  --name gitops-config \
  --cluster-name my-k8s-cluster \
  --resource-group rg-arc \
  --cluster-type connectedClusters \
  --url https://github.com/myorg/k8s-manifests \
  --branch main

# Install extension on Arc server
az connectedmachine extension create \
  --name AzureMonitorLinuxAgent \
  --machine-name my-server \
  --resource-group rg-arc \
  --type AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor

# Delete Arc server
az connectedmachine delete --name my-server --resource-group rg-arc --yes
```

---

## Quick Reference

```
Azure Arc = Extend Azure management to any infrastructure

Arc-enabled resources:
├── Servers: Windows/Linux anywhere (on-prem, AWS, GCP, edge)
├── Kubernetes: Any K8s cluster (EKS, GKE, on-prem, edge)
├── Data Services: SQL MI, PostgreSQL on your K8s
├── SQL Server: Connect on-prem SQL instances
└── VMware/SCVMM: Manage VMs from Azure

How: Install Connected Machine Agent → Server appears in Azure Portal

What you get:
├── Azure Policy (governance everywhere)
├── RBAC (consistent access control)
├── Azure Monitor (metrics/logs from everywhere)
├── Defender for Cloud (security everywhere)
├── Update Manager (patching everywhere)
├── GitOps (deploy to any K8s from Git)
└── Azure PaaS on K8s (App Service, Functions on your cluster!)

Agent: Outbound HTTPS only, no inbound ports needed
Pricing: Arc control plane is FREE; pay for add-on services
```

---

## What's Next?

Next chapter: [Chapter 64: Azure Stack & Hybrid Solutions](64-azure-stack-hybrid.md) — Run Azure services in your datacenter with Azure Stack Hub, HCI, and Edge.
