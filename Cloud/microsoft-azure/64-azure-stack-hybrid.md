# Chapter 64: Azure Stack & Hybrid Solutions

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Hybrid Overview](#part-1-azure-hybrid-overview)
- [Part 2: Azure Stack Hub](#part-2-azure-stack-hub)
- [Part 3: Azure Stack HCI](#part-3-azure-stack-hci)
- [Part 4: Azure Stack Edge](#part-4-azure-stack-edge)
- [Part 5: Hybrid Connectivity](#part-5-hybrid-connectivity)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Stack brings Azure services into your own data center, edge locations, or disconnected environments. There are three products: Stack Hub (full Azure in a box), Stack HCI (hyper-converged infrastructure), and Stack Edge (AI at the edge).

```
What you'll learn:
├── Azure Hybrid Overview
│   ├── Why hybrid (regulations, latency, disconnected)
│   └── Azure Stack family comparison
├── Azure Stack Hub (Azure in your datacenter)
├── Azure Stack HCI (modern hyperconverged infra)
├── Azure Stack Edge (AI/ML at the edge)
├── Hybrid Connectivity patterns
└── Quick reference
```

---

## Part 1: Azure Hybrid Overview

```
Why hybrid?
├── Data sovereignty: Data must stay in-country
├── Regulation: Healthcare, government, finance
├── Latency: Process data close to the source
├── Disconnected: Ships, mines, military, remote sites
├── Gradual migration: Move to cloud incrementally
└── Existing investment: Leverage on-prem hardware

Azure Stack family:
┌──────────────────┬──────────────────┬──────────────────┬──────────────────┐
│                  │ Stack Hub        │ Stack HCI        │ Stack Edge       │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ What             │ Azure in a box   │ Hyperconverged   │ Edge appliance   │
│                  │ (full cloud)     │ infrastructure   │ (AI/compute)     │
│ Hardware         │ Integrated system│ Validated hardware│ Microsoft device│
│                  │ (Dell, HPE, etc.)│ (Dell, Lenovo,..)│ (shipped to you)│
│ Use case         │ Run Azure APIs   │ Modernize VMs    │ ML inference,   │
│                  │ on-premises      │ and containers   │ IoT, edge data  │
│ Disconnected     │ Yes ✅          │ No (needs Azure) │ Partially       │
│ Self-service     │ Full Azure Portal│ Azure Portal     │ Azure Portal    │
│ Services         │ VMs, Storage, K8s│ VMs, AKS, Azure  │ VMs, containers │
│                  │ App Service, SQL │ services         │ ML models       │
│ Cost             │ $$$$ (hardware)  │ $$ (software sub)│ $ (monthly rent)│
│ Target           │ Enterprise DC    │ Branch/DC        │ Edge locations  │
└──────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

## Part 2: Azure Stack Hub

```
Azure Stack Hub = Complete Azure in your data center

What you get:
├── Azure Portal (local) for self-service
├── Azure Resource Manager (ARM) templates work
├── Azure services:
│   ├── Virtual Machines (Windows/Linux)
│   ├── Azure App Service
│   ├── Azure Kubernetes Service (AKS)
│   ├── Azure Storage (Blob, Table, Queue)
│   ├── Azure SQL
│   ├── Key Vault
│   ├── Event Hubs
│   └── Azure Functions
├── Same APIs and tools as Azure public cloud
└── Works fully disconnected (no internet required!)

Hardware:
├── Integrated systems from Dell, HPE, Lenovo, Cisco
├── 4-16 nodes per "scale unit"
├── Pre-configured and validated
├── Starts at ~$100,000+
└── Microsoft manages software updates

Who uses it:
├── Government agencies (classified data)
├── Military (disconnected operations)
├── Cruise ships (intermittent connectivity)
├── Mining operations (remote locations)
├── Healthcare (patient data regulations)
└── Manufacturing (factory floor, air-gapped)

Operator vs User model:
├── Operator: IT admin who manages the hardware/infrastructure
│   ├── Update firmware, manage capacity
│   ├── Create "offers" and "plans" (what users can use)
│   └── Monitor health of the stamp
└── User (Tenant): Developer who deploys workloads
    ├── Subscribe to plans
    ├── Create VMs, apps, databases
    └── Same experience as Azure public cloud
```

---

## Part 3: Azure Stack HCI

```
Azure Stack HCI = Hyperconverged infrastructure with Azure integration

What it is:
├── Runs on YOUR validated hardware (Dell, HPE, Lenovo, etc.)
├── Hyperconverged: Compute + Storage + Networking in one cluster
├── Managed from Azure Portal
├── Monthly subscription per physical core (~$10/core/month)
└── Requires Azure connectivity (not for disconnected)

What you can run:
├── Windows and Linux VMs
├── AKS hybrid (Kubernetes on-premises)
├── Azure Virtual Desktop
├── Azure Arc services
├── Azure Benefits (use Azure licenses on-prem)
└── Stretched clusters (across two sites for DR)

Azure Stack HCI vs traditional Hyper-V:
├── HCI: Azure-managed, auto-updates, Azure Portal
├── Hyper-V: Self-managed, manual updates, SCVMM
└── HCI is the future direction for Microsoft virtualization

Setup:
├── Buy validated hardware (2-16 nodes)
├── Install Azure Stack HCI OS
├── Register with Azure
├── Manage from Azure Portal → Azure Stack HCI
└── Deploy VMs, AKS, storage from Azure
```

---

## Part 4: Azure Stack Edge

```
Azure Stack Edge = Microsoft-owned edge device shipped to you

Models:
├── Edge Pro (GPU): NVIDIA T4 GPU for ML inference
├── Edge Pro (FPGA): Intel FPGA for specific workloads
├── Edge Mini R: Ruggedized, portable (military, field)
└── Edge Pro R: Ruggedized rack unit

How it works:
1. Order from Azure Portal (Console → Azure Stack Edge)
2. Microsoft ships device to you
3. Connect to your network, activate from Azure
4. Deploy workloads (VMs, containers, ML models)
5. Data processed at edge, results sent to cloud

Use cases:
├── ML inference at the edge (quality inspection in factory)
├── IoT data processing (process locally, send summary to cloud)
├── Data transfer (cache + transfer large data to Azure)
├── Remote office compute (VMs at branch office)
└── Healthcare imaging (process medical images locally)

Features:
├── Azure IoT Edge integration
├── GPU compute for ML models
├── Kubernetes for containerized apps
├── Local Storage (cache + tiered to Azure Blob)
├── Azure Portal management
└── Hardware refresh cycle (Microsoft maintains)

Pricing:
├── Monthly rental: ~$446/month (Pro) to ~$2,198/month (Pro GPU)
├── No upfront hardware purchase
└── Microsoft handles hardware replacement
```

---

## Part 5: Hybrid Connectivity

```
Connecting on-premises to Azure:

VPN Gateway:
├── Site-to-Site VPN (IPsec tunnel over internet)
├── Up to 10 Gbps (VpnGw5)
├── Encrypted, cost-effective
└── Good for: Most hybrid scenarios

ExpressRoute:
├── Private, dedicated connection (not over internet)
├── Up to 100 Gbps
├── Lower latency, higher reliability
├── Providers: Airtel, Tata, Equinix, etc.
├── Expensive: ~$200-$5000+/month
└── Good for: Mission-critical, high-bandwidth

ExpressRoute + VPN (failover):
├── ExpressRoute as primary (fast, private)
├── VPN as backup (if ExpressRoute fails)
└── Best for: Maximum reliability

Azure Virtual WAN:
├── Hub-and-spoke networking at scale
├── Connect branches, VPNs, ExpressRoute in one hub
├── Centralized routing and security
└── Good for: Many branch offices
```

---

## Part 6: az CLI Reference

```bash
# --- Azure Stack HCI ---

# Register Azure Stack HCI cluster
az stack-hci cluster create \
  --name hci-cluster-01 \
  --resource-group rg-hybrid \
  --location centralindia

# List HCI clusters
az stack-hci cluster list --resource-group rg-hybrid -o table

# Show HCI cluster details
az stack-hci cluster show --name hci-cluster-01 --resource-group rg-hybrid

# Delete HCI cluster registration
az stack-hci cluster delete --name hci-cluster-01 --resource-group rg-hybrid --yes

# --- Azure Stack Edge ---

# Create Stack Edge order
az databoxedge device create \
  --name edge-device-01 \
  --resource-group rg-edge \
  --location centralindia \
  --sku EdgeP_Base

# List Stack Edge devices
az databoxedge device list --resource-group rg-edge -o table

# Show device details
az databoxedge device show --name edge-device-01 --resource-group rg-edge

# Get activation key
az databoxedge device show \
  --name edge-device-01 \
  --resource-group rg-edge \
  --query "deviceProperties"

# List shares on device
az databoxedge share list --device-name edge-device-01 --resource-group rg-edge -o table

# Delete device
az databoxedge device delete --name edge-device-01 --resource-group rg-edge --yes

# --- VPN Gateway (hybrid connectivity) ---

# Create VPN gateway
az network vnet-gateway create \
  --name gw-hybrid \
  --resource-group rg-hybrid \
  --vnet vnet-hub \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1 \
  --public-ip-address pip-vpn

# Create site-to-site connection
az network vpn-connection create \
  --name conn-onprem \
  --resource-group rg-hybrid \
  --vnet-gateway1 gw-hybrid \
  --local-gateway2 lgw-onprem \
  --shared-key "MySharedKey123!"
```

---

## Real-World Patterns

### Pattern 1: Hybrid Manufacturing (Stack Edge + Cloud)

```
┌─────────────────────────────────────────────────┐
│  Factory Floor + Azure Cloud                    │
├─────────────────────────────────────────────────┤
│                                                 │
│  Factory (On-Premises)                          │
│  ┌─────────────────────────────────────┐        │
│  │ Azure Stack Edge (GPU)              │        │
│  │ ├── Camera feeds → ML inference    │        │
│  │ │   (quality inspection in <10ms)  │        │
│  │ ├── IoT Edge modules               │        │
│  │ └── Local storage (cache)          │        │
│  └────────────┬────────────────────────┘        │
│               │ ExpressRoute / VPN              │
│               ▼                                 │
│  Azure Cloud                                    │
│  ├── ML model training (retrain weekly)         │
│  ├── IoT Hub (device management)                │
│  ├── Data Lake (historical analytics)           │
│  └── Power BI (dashboards for management)       │
│                                                 │
│  Why: <10ms inference needed on factory floor,  │
│  internet latency too high for real-time QA     │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Azure Stack Family:
├── Stack Hub: Full Azure in your datacenter (disconnected OK)
│   Hardware: $100K+ integrated system, 4-16 nodes
│   Services: VMs, App Service, AKS, Storage, SQL, Functions
│
├── Stack HCI: Modern hyperconverged infra (Azure-managed)
│   Hardware: Your validated servers, ~$10/core/month
│   Services: VMs, AKS hybrid, AVD, Arc services
│
└── Stack Edge: Microsoft-owned edge device (shipped to you)
    Hardware: Rented ~$446-$2198/month
    Services: VMs, containers, ML inference, IoT

Hybrid Connectivity:
├── VPN Gateway: Encrypted tunnel over internet (up to 10 Gbps)
├── ExpressRoute: Private dedicated link (up to 100 Gbps)
└── Virtual WAN: Hub-and-spoke for many branches

Use when: Data sovereignty, disconnected, latency, regulations
Azure Arc: Manage on-prem from Azure (lighter than Stack)
Azure Stack: Run Azure ON-PREMISES (heavier, full Azure experience)
```

---

## What's Next?

Next chapter: [Chapter 65: Azure Well-Architected Framework](65-well-architected.md) — The five pillars of building excellent cloud solutions.
