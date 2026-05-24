# Chapter 6: Virtual Networks (VNet) - Azure

---

## Table of Contents

- [Overview](#overview)
- [Part 1: What is a VNet?](#part-1-what-is-a-vnet)
- [Part 2: No Default VNet!](#part-2-no-default-vnet)
- [Part 3: Address Space and CIDR](#part-3-address-space-and-cidr)
- [Part 4: Subnets](#part-4-subnets)
- [Part 5: Network Interfaces (NICs)](#part-5-network-interfaces-nics)
- [Part 6: Public and Private IP Addresses](#part-6-public-and-private-ip-addresses)
- [Part 7: DNS Settings](#part-7-dns-settings)
- [Part 8: Service Endpoints](#part-8-service-endpoints)
- [Part 9: VNet Creation — Portal Walkthrough](#part-9-vnet-creation--portal-walkthrough)
- [Part 10: VNet Creation — CLI](#part-10-vnet-creation--cli)
- [Part 11: VNet Creation — Bicep](#part-11-vnet-creation--bicep)
- [Part 12: Multi-Tier Architecture](#part-12-multi-tier-architecture)
- [Part 13: VNet Limits and Quotas](#part-13-vnet-limits-and-quotas)
- [Part 14: Real-World Patterns](#part-14-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Virtual Network (VNet) is the fundamental building block for your private network in Azure. VNets enable Azure resources to securely communicate with each other, the internet, and on-premises networks. VNets are **regional** (like AWS VPCs) and scoped to a **subscription**.

```
What you'll learn:
├── What is a VNet and why you need it
├── Default behavior (no default VNet!)
├── Address space and CIDR blocks
├── Subnets (regular, gateway, firewall, bastion, etc.)
├── Special subnets and delegated subnets
├── Network interfaces (NICs)
├── Public and Private IP addresses
├── DNS settings in VNet
├── Service Endpoints
├── VNet creation walkthrough (Portal + CLI + Bicep)
├── Multi-tier architecture design
├── VNet limits and quotas
└── Real-world patterns (startup, mid-size, enterprise)
```

---

## Part 1: What is a VNet?

### The Big Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AZURE CLOUD                                   │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │           YOUR VNET (10.0.0.0/16) — Regional                  │  │
│  │           Region: Central India                                │  │
│  │                                                                 │  │
│  │  ┌──────────────────┐  ┌──────────────────┐                    │  │
│  │  │  Subnet: web      │  │  Subnet: app      │                   │  │
│  │  │  10.0.1.0/24      │  │  10.0.2.0/24      │                   │  │
│  │  │                    │  │                    │                   │  │
│  │  │  ┌──────────────┐ │  │  ┌──────────────┐ │                   │  │
│  │  │  │ Web VM       │ │  │  │ App VM       │ │                   │  │
│  │  │  │ + Load       │ │  │  │              │ │                   │  │
│  │  │  │   Balancer   │ │  │  │              │ │                   │  │
│  │  │  └──────────────┘ │  │  └──────────────┘ │                   │  │
│  │  └──────────────────┘  └──────────────────┘                    │  │
│  │                                                                 │  │
│  │  ┌──────────────────┐                                           │  │
│  │  │  Subnet: db       │                                           │  │
│  │  │  10.0.3.0/24      │                                           │  │
│  │  │  ┌──────────────┐ │                                           │  │
│  │  │  │ Azure SQL     │ │                                           │  │
│  │  │  │ (Private EP)  │ │                                           │  │
│  │  │  └──────────────┘ │                                           │  │
│  │  └──────────────────┘                                           │  │
│  │                                                                 │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ INTERNET ←→ Public IP ←→ NSG ←→ NIC ←→ VM                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

KEY CONCEPTS:
├── VNet = Regional resource (scoped to a single Azure region)
├── Subnet = Segment of VNet (no AZ constraint — spans all AZs!)
│   └── ⚡ Unlike AWS: Azure subnets span ALL AZs in a region
├── NSG = Network Security Group (firewall at subnet/NIC level)
├── NIC = Network Interface Card (connects VM to subnet)
├── Public IP = Optional, assigned to NIC or LB
├── No default VNet! You MUST create one before deploying resources
├── Service Endpoints = Private access to Azure PaaS services
└── Private Endpoints = Private IP for Azure PaaS services
```

### Azure vs AWS vs GCP Comparison

```
┌──────────────────────┬────────────────┬──────────────┬─────────────┐
│ Feature              │ Azure VNet     │ AWS VPC      │ GCP VPC     │
├──────────────────────┼────────────────┼──────────────┼─────────────┤
│ Scope                │ Regional       │ Regional     │ Global      │
│ Subnet scope         │ Regional (all  │ AZ-specific  │ Regional    │
│                      │ AZs in region!)│              │             │
│ Default network      │ NONE           │ Default VPC  │ default     │
│ Firewall             │ NSG (subnet/NIC│ SG + NACL    │ FW rules    │
│                      │ level)         │              │ (VPC level) │
│ Internet access      │ Public IP + NSG│ IGW + route  │ External IP │
│                      │                │ + public IP  │ + FW rule   │
│ NAT                  │ NAT Gateway    │ NAT Gateway  │ Cloud NAT   │
│ Cross-region connect │ VNet Peering   │ VPC Peering  │ Same VPC!   │
│ Route management     │ Route tables   │ Route tables │ Routes      │
│ Address space change │ Can add more!  │ Can add sec. │ Can expand  │
│                      │                │ CIDR blocks  │ subnet CIDR │
│ Subnet resize        │ NO (delete &   │ NO           │ YES!        │
│                      │ recreate)      │              │             │
└──────────────────────┴────────────────┴──────────────┴─────────────┘
```

---

## Part 2: No Default VNet!

```
┌─────────────────────────────────────────────────────────────────────┐
│              AZURE: NO DEFAULT VNET                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Unlike AWS (default VPC in every region) and GCP (default network), │
│ Azure does NOT create a default VNet!                                │
│                                                                       │
│ What this means:                                                     │
│ ├── You MUST create a VNet before deploying VMs, AKS, etc.         │
│ ├── Forces you to think about networking upfront (good!)            │
│ ├── No "accidental" public resources in a default network           │
│ ├── When creating a VM, Portal can auto-create a VNet               │
│ │   (but don't rely on this for production!)                        │
│ └── Always plan your network architecture first                     │
│                                                                       │
│ When you create a VM through the portal, Azure OFFERS to create:    │
│ ├── A VNet (named like <vm-name>-vnet)                             │
│ ├── A subnet (named "default", 10.0.0.0/24)                       │
│ ├── A public IP                                                     │
│ ├── An NSG                                                          │
│ └── A NIC                                                           │
│                                                                       │
│ ⚠️ For production: Create VNet separately, then deploy resources!    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Address Space and CIDR

```
┌─────────────────────────────────────────────────────────────────────┐
│                   ADDRESS SPACE                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure VNet uses "address spaces" (CIDR blocks)                       │
│                                                                       │
│ Rules:                                                               │
│ ├── Minimum subnet: /29 (8 IPs, 3 usable)                         │
│ ├── Maximum VNet: /8 (or even /2 for address space)                │
│ ├── Use RFC 1918 private ranges:                                    │
│ │   ├── 10.0.0.0/8                                                 │
│ │   ├── 172.16.0.0/12                                              │
│ │   └── 192.168.0.0/16                                             │
│ ├── ⚡ Can add MULTIPLE address spaces to one VNet!                 │
│ │   (10.0.0.0/16 AND 172.16.0.0/16 in same VNet)                  │
│ ├── Cannot overlap with peered VNets                               │
│ ├── Cannot overlap with on-premises if using VPN/ExpressRoute      │
│ └── Can add/remove address spaces after creation!                  │
│     (if no subnet is using the range being removed)                │
│                                                                       │
│ Azure reserved IPs per subnet (5 IPs reserved):                     │
│ ┌──────────────┬──────────────────────────────────────────────────┐│
│ │ IP Address   │ Purpose                                          ││
│ ├──────────────┼──────────────────────────────────────────────────┤│
│ │ x.x.x.0      │ Network address                                  ││
│ │ x.x.x.1      │ Default gateway                                  ││
│ │ x.x.x.2      │ Azure DNS (mapped to VNet)                       ││
│ │ x.x.x.3      │ Azure DNS (mapped to VNet)                       ││
│ │ x.x.x.255    │ Broadcast                                        ││
│ ├──────────────┼──────────────────────────────────────────────────┤│
│ │ Usable /24   │ 251 IPs (256 - 5)                                ││
│ │ Usable /29   │ 3 IPs (8 - 5) — absolute minimum!               ││
│ └──────────────┴──────────────────────────────────────────────────┘│
│                                                                       │
│ ⚠️ /29 gives only 3 usable IPs. For gateway subnets this works.     │
│    For regular subnets, use at least /27 (27 usable) or /24 (251). │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Recommended CIDR Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│              CIDR PLANNING STRATEGY                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Company-wide allocation:                                             │
│ ┌─────────────────────────────────────────────────────────────┐    │
│ │ Overall range:   10.0.0.0/8                                  │    │
│ │                                                               │    │
│ │ Prod VNet (Central India):    10.0.0.0/16                    │    │
│ │ Staging VNet (Central India): 10.1.0.0/16                    │    │
│ │ Dev VNet (Central India):     10.2.0.0/16                    │    │
│ │ Shared VNet (Central India):  10.3.0.0/16                    │    │
│ │ DR VNet (South India):        10.10.0.0/16                   │    │
│ │ On-premises:                  192.168.0.0/16                 │    │
│ └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Within a VNet (10.0.0.0/16):                                        │
│ ┌─────────────────────────────────────────────────────────────┐    │
│ │ Web tier:            10.0.1.0/24  (251 IPs)                  │    │
│ │ App tier:            10.0.2.0/24  (251 IPs)                  │    │
│ │ Database tier:       10.0.3.0/24  (251 IPs)                  │    │
│ │ AKS nodes:           10.0.10.0/24 (251 IPs)                  │    │
│ │ AKS pods (if Azure CNI): 10.0.64.0/18 (16K IPs)             │    │
│ │ GatewaySubnet:       10.0.255.0/27 (27 IPs, for VPN GW)     │    │
│ │ AzureBastionSubnet:  10.0.254.0/26 (59 IPs, for Bastion)    │    │
│ │ AzureFirewallSubnet: 10.0.253.0/26 (59 IPs, for Firewall)   │    │
│ └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Subnets

### How Subnets Work in Azure

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AZURE SUBNETS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚡ KEY: Azure subnets span ALL Availability Zones in a region!      │
│    (Unlike AWS where each subnet is AZ-specific)                    │
│                                                                       │
│ VNet: prod-vnet (Central India)                                      │
│ │                                                                    │
│ ├── Subnet: web (10.0.1.0/24)                                     │
│ │   ├── VM in AZ 1 ← uses this subnet                             │
│ │   ├── VM in AZ 2 ← uses this SAME subnet                        │
│ │   └── VM in AZ 3 ← uses this SAME subnet                        │
│ │   (No need to create per-AZ subnets for HA!)                     │
│ │                                                                    │
│ ├── Subnet: app (10.0.2.0/24)                                     │
│ │   └── App VMs in any AZ                                          │
│ │                                                                    │
│ └── Subnet: db (10.0.3.0/24)                                      │
│     └── SQL instances in any AZ                                    │
│                                                                       │
│ Subnet properties:                                                   │
│ ├── Name: Must be unique within the VNet                            │
│ ├── Address range: Must be within VNet address space               │
│ ├── NSG: Can attach one NSG (optional but recommended)             │
│ ├── Route table: Can associate one UDR (optional)                  │
│ ├── Service endpoints: Enable access to Azure PaaS                 │
│ ├── Delegation: Dedicate subnet to a specific service              │
│ ├── Private endpoint policy: Control Private Endpoints             │
│ └── NAT gateway: Attach for outbound internet                     │
│                                                                       │
│ Subnet rules:                                                        │
│ ├── CIDR must be within VNet address space                         │
│ ├── Cannot overlap with other subnets                               │
│ ├── Cannot be resized (must delete and recreate with new CIDR)     │
│ │   ⚠️ This means all resources in subnet must be deleted first!   │
│ ├── Can be deleted if empty (no resources)                         │
│ └── Some subnets have naming requirements (see special subnets)    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Special Subnets

```
┌─────────────────────────────────────────────────────────────────────┐
│                   SPECIAL SUBNETS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure has several subnets with REQUIRED names:                       │
│                                                                       │
│ 1. GatewaySubnet (MUST be named exactly "GatewaySubnet")           │
│    ├── For: VPN Gateway, ExpressRoute Gateway                      │
│    ├── Min size: /27 recommended (some features need it)           │
│    ├── Do NOT add NSG to this subnet                               │
│    └── Do NOT deploy VMs in this subnet                            │
│                                                                       │
│ 2. AzureBastionSubnet (MUST be named exactly "AzureBastionSubnet") │
│    ├── For: Azure Bastion (secure SSH/RDP without public IP)       │
│    ├── Min size: /26                                                │
│    └── Replaces need for bastion/jump hosts                        │
│                                                                       │
│ 3. AzureFirewallSubnet (MUST be named "AzureFirewallSubnet")       │
│    ├── For: Azure Firewall                                         │
│    ├── Min size: /26                                                │
│    └── Do NOT deploy other resources here                          │
│                                                                       │
│ 4. AzureFirewallManagementSubnet                                    │
│    ├── For: Azure Firewall forced tunneling                        │
│    └── Min size: /26                                                │
│                                                                       │
│ 5. RouteServerSubnet                                                │
│    ├── For: Azure Route Server                                     │
│    └── Min size: /27                                                │
│                                                                       │
│ Delegated subnets (not specific names, but dedicated):              │
│ ├── Azure SQL Managed Instance: Dedicated subnet, /27 minimum     │
│ ├── Azure Container Instances: Delegated subnet                    │
│ ├── Azure App Service VNet Integration: Delegated subnet          │
│ └── Azure NetApp Files: Delegated subnet                           │
│                                                                       │
│ 💡 When planning subnets, reserve space for these special subnets!  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Network Interfaces (NICs)

```
┌─────────────────────────────────────────────────────────────────────┐
│                NETWORK INTERFACE CARDS (NICs)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ In Azure, NICs are SEPARATE resources (unlike AWS ENIs):             │
│                                                                       │
│ VM ←→ NIC ←→ Subnet                                                 │
│        │                                                              │
│        ├── Has private IP (from subnet CIDR)                        │
│        ├── Can have public IP (optional)                            │
│        ├── Can have NSG attached                                    │
│        └── Can have multiple IPs per NIC                            │
│                                                                       │
│ NIC properties:                                                      │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ Resource group:     [rg-prod-networking ▼]                    │    │
│ │ Name:               [prod-web-vm1-nic]                        │    │
│ │ Region:             [Central India]                           │    │
│ │ Virtual network:    [prod-vnet ▼]                             │    │
│ │ Subnet:             [web ▼]                                   │    │
│ │                                                                │    │
│ │ Private IP:                                                    │    │
│ │   Allocation:  ○ Dynamic  ○ Static                            │    │
│ │   Address:     [10.0.1.4] (auto or specified)                 │    │
│ │                                                                │    │
│ │ Public IP:                                                     │    │
│ │   [prod-web-vm1-pip ▼] or [None]                              │    │
│ │                                                                │    │
│ │ NSG:                                                           │    │
│ │   [web-nsg ▼] (or None)                                       │    │
│ │                                                                │    │
│ │ Accelerated networking: ☑ Enable (better performance)         │    │
│ │   └── Available on most D/E/F/M series VMs                   │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Multiple NICs per VM:                                                │
│ ├── Depends on VM size (B1s: 2, D2s: 2, D4s: 2, D8s: 4, etc.)   │
│ ├── Each NIC can be in a DIFFERENT subnet                          │
│ ├── Use case: Network virtual appliance (NVA), firewall            │
│ └── All NICs must be in the SAME VNet                              │
│                                                                       │
│ ⚡ Dynamic private IP = stays same unless you deallocate the VM      │
│    Static private IP = guaranteed to never change                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Public and Private IP Addresses

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IP ADDRESSES IN AZURE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. PRIVATE IP ADDRESSES                                              │
│ ├── From subnet CIDR range                                          │
│ ├── Every NIC has at least one                                      │
│ ├── Dynamic (default): Assigned by Azure, stable during VM lifetime │
│ │   └── Changes only on deallocation                                │
│ ├── Static: You choose the IP, guaranteed to not change            │
│ └── Used for: All internal communication                            │
│                                                                       │
│ 2. PUBLIC IP ADDRESSES (separate resource!)                          │
│ ├── Azure creates a Public IP resource                              │
│ ├── Associated with: NIC, Load Balancer, App Gateway, etc.         │
│ │                                                                    │
│ ├── SKU: Basic (being retired!) vs Standard                        │
│ │   ┌──────────────────┬──────────────────┬───────────────────┐    │
│ │   │ Feature          │ Basic (retiring) │ Standard           │    │
│ │   ├──────────────────┼──────────────────┼───────────────────┤    │
│ │   │ Assignment       │ Dynamic/Static   │ Static only        │    │
│ │   │ Availability zone│ Not zone-aware   │ Zone-redundant    │    │
│ │   │ Security         │ Open by default  │ Closed by default │    │
│ │   │ Routing          │ Internet only    │ Internet + routing│    │
│ │   │ Price            │ Cheaper          │ ~$3.65/month      │    │
│ │   │ ⚠️ Retirement    │ Sept 30, 2025    │ Use this!         │    │
│ │   └──────────────────┴──────────────────┴───────────────────┘    │
│ │                                                                    │
│ ├── Standard Public IP:                                             │
│ │   ├── Static only (always fixed IP)                              │
│ │   ├── Zone-redundant by default                                  │
│ │   ├── ⚠️ Closed by default! Need NSG to allow traffic           │
│ │   └── $0.005/hour ≈ $3.65/month                                 │
│ │                                                                    │
│ └── ⚠️ Public IPs cost money even when not associated!              │
│     Always delete unused Public IPs                                 │
│                                                                       │
│ CLI:                                                                 │
│ # Create static public IP                                           │
│ az network public-ip create \                                        │
│   --name prod-web-pip \                                              │
│   --resource-group rg-prod \                                         │
│   --location centralindia \                                          │
│   --sku Standard \                                                   │
│   --allocation-method Static \                                       │
│   --zone 1 2 3                                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: DNS Settings

```
┌─────────────────────────────────────────────────────────────────────┐
│                   DNS IN VNET                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Every VNet has DNS settings:                                         │
│                                                                       │
│ 1. AZURE-PROVIDED DNS (Default)                                      │
│ ├── Automatic, no configuration needed                               │
│ ├── Resolves: Azure resource names within VNet                     │
│ ├── Resolves: Public DNS names (internet)                          │
│ ├── VM hostname registered automatically                           │
│ ├── DNS server: 168.63.129.16 (Azure's DNS resolver)               │
│ ├── ⚠️ Cannot resolve between VNets (only within same VNet)        │
│ └── Sufficient for simple setups                                    │
│                                                                       │
│ 2. CUSTOM DNS SERVERS                                                │
│ ├── Point to your own DNS server (in Azure or on-prem)             │
│ ├── Configure at VNet level: VNet → DNS servers → Custom           │
│ ├── All VMs in VNet use these DNS servers                          │
│ ├── Use case: Hybrid (cloud + on-prem DNS resolution)              │
│ └── Must restart VMs after changing DNS settings!                   │
│                                                                       │
│ 3. AZURE PRIVATE DNS ZONES (recommended for VNet-to-VNet)           │
│ ├── Create: privatelink.database.windows.net                       │
│ ├── Link to VNets → auto-registration of VM DNS records            │
│ ├── Enables DNS resolution across peered VNets                     │
│ ├── Required for Private Endpoints                                  │
│ └── No need to manage DNS servers!                                  │
│                                                                       │
│ Configure DNS:                                                       │
│ Portal → VNet → DNS servers                                         │
│ ○ Default (Azure-provided)                                          │
│ ○ Custom: [10.0.3.10] [Add DNS server IP]                          │
│                                                                       │
│ CLI:                                                                 │
│ az network vnet update \                                              │
│   --name prod-vnet \                                                  │
│   --resource-group rg-prod \                                          │
│   --dns-servers 10.0.3.10                                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Service Endpoints

```
┌─────────────────────────────────────────────────────────────────────┐
│                  SERVICE ENDPOINTS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Allow traffic to Azure PaaS services to stay on Azure          │
│       backbone network (not go through the internet)                 │
│                                                                       │
│ Without Service Endpoints:                                           │
│ VM (10.0.1.5) → Internet → Azure Storage (public endpoint)         │
│ ⚠️ Traffic leaves VNet, goes over internet, comes back              │
│                                                                       │
│ With Service Endpoints:                                              │
│ VM (10.0.1.5) → Azure backbone → Azure Storage                     │
│ ✅ Traffic stays on Azure network, faster, more secure              │
│                                                                       │
│ Available for:                                                       │
│ ├── Microsoft.Storage (Azure Storage)                               │
│ ├── Microsoft.Sql (Azure SQL Database)                              │
│ ├── Microsoft.KeyVault (Key Vault)                                  │
│ ├── Microsoft.ServiceBus                                            │
│ ├── Microsoft.EventHub                                              │
│ ├── Microsoft.AzureCosmosDB                                        │
│ ├── Microsoft.ContainerRegistry                                    │
│ ├── Microsoft.Web (App Service)                                     │
│ └── Microsoft.AzureActiveDirectory                                  │
│                                                                       │
│ How to enable:                                                       │
│ Portal → VNet → Subnets → [subnet] → Service endpoints             │
│ ☑ Microsoft.Storage                                                 │
│ ☑ Microsoft.Sql                                                     │
│ ☑ Microsoft.KeyVault                                                │
│                                                                       │
│ CLI:                                                                 │
│ az network vnet subnet update \                                      │
│   --name app \                                                        │
│   --vnet-name prod-vnet \                                             │
│   --resource-group rg-prod \                                          │
│   --service-endpoints Microsoft.Storage Microsoft.Sql                │
│                                                                       │
│ ⚠️ Service Endpoints vs Private Endpoints:                           │
│ ┌──────────────────────┬──────────────────┬───────────────────────┐ │
│ │                      │ Service Endpoint │ Private Endpoint       │ │
│ ├──────────────────────┼──────────────────┼───────────────────────┤ │
│ │ IP used              │ Public IP of PaaS│ Private IP in your VNet│ │
│ │ Traffic path         │ Azure backbone   │ Through your VNet      │ │
│ │ DNS                  │ Public DNS       │ Private DNS zone       │ │
│ │ Access from on-prem  │ No              │ Yes (via VPN/ER)       │ │
│ │ Granularity          │ Entire service   │ Specific resource      │ │
│ │ Cost                 │ FREE             │ $0.01/hr + data        │ │
│ │ Recommended          │ Good start       │ Best for security      │ │
│ └──────────────────────┴──────────────────┴───────────────────────┘ │
│                                                                       │
│ 💡 Use Service Endpoints for simple setups (free!)                   │
│    Use Private Endpoints for zero-trust / on-prem access            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: VNet Creation — Portal Walkthrough

```
Portal → Virtual networks → Create

┌─────────────────────────────────────────────────────────────────┐
│              CREATE VIRTUAL NETWORK                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ TAB 1: BASICS                                                    │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Subscription:      [Visual Studio Enterprise ▼]            │   │
│ │ Resource group:    [rg-prod-networking ▼] [Create new]    │   │
│ │ Name:              [prod-vnet]                             │   │
│ │ Region:            [Central India ▼]                       │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ TAB 2: SECURITY                                                  │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Azure Bastion:                                             │   │
│ │ ☐ Enable Azure Bastion                                    │   │
│ │   └── If enabled: Creates AzureBastionSubnet + Bastion    │   │
│ │       Name: [prod-bastion]                                │   │
│ │       Public IP: [prod-bastion-pip]                        │   │
│ │       ⚠️ Cost: ~$140/month (Standard SKU)                 │   │
│ │                                                            │   │
│ │ Azure Firewall:                                            │   │
│ │ ☐ Enable Azure Firewall                                   │   │
│ │   └── If enabled: Creates AzureFirewallSubnet + Firewall  │   │
│ │       ⚠️ Cost: ~$912/month (Standard)                     │   │
│ │                                                            │   │
│ │ Azure DDoS Protection:                                     │   │
│ │ ☐ Enable Azure DDoS Protection                            │   │
│ │   └── ⚠️ Cost: ~$2,944/month (very expensive!)           │   │
│ │       Only for large enterprises                           │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ TAB 3: IP ADDRESSES                                              │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ IPv4 address space: [10.0.0.0/16]                         │   │
│ │ [+ Add address space]                                      │   │
│ │                                                            │   │
│ │ Subnets:                                                   │   │
│ │ ┌─────────────────────────────────────────────────────┐   │   │
│ │ │ Name         │ Address range │ Size   │ Delegation  │   │   │
│ │ ├─────────────────────────────────────────────────────┤   │   │
│ │ │ default      │ 10.0.0.0/24   │ 251    │ None        │   │   │
│ │ │              │               │        │             │   │   │
│ │ │ [+ Add subnet]                                      │   │   │
│ │ └─────────────────────────────────────────────────────┘   │   │
│ │                                                            │   │
│ │ Edit "default" subnet → rename to "web":                  │   │
│ │ ┌─────────────────────────────────────────────────────┐   │   │
│ │ │ Subnet name:       [web]                             │   │   │
│ │ │ Starting address:  [10.0.1.0]                        │   │   │
│ │ │ Subnet size:       [/24 (256 addresses) ▼]          │   │   │
│ │ │                                                      │   │   │
│ │ │ Add IPv6: ☐                                          │   │   │
│ │ │                                                      │   │   │
│ │ │ NAT gateway: [None ▼] (attach later or during)      │   │   │
│ │ │                                                      │   │   │
│ │ │ Service endpoints: ☐ Microsoft.Storage               │   │   │
│ │ │                    ☐ Microsoft.Sql                   │   │   │
│ │ │                    ☐ Microsoft.KeyVault              │   │   │
│ │ │                                                      │   │   │
│ │ │ Subnet delegation: [None ▼]                          │   │   │
│ │ │   (Microsoft.Web/serverFarms, Microsoft.Sql/...)     │   │   │
│ │ │                                                      │   │   │
│ │ │ Network security group: [None ▼] (attach after)     │   │   │
│ │ │ Route table: [None ▼]                                │   │   │
│ │ │                                                      │   │   │
│ │ │ Private endpoint network policy: [Disabled ▼]       │   │   │
│ │ └─────────────────────────────────────────────────────┘   │   │
│ │                                                            │   │
│ │ Add more subnets: app (10.0.2.0/24), db (10.0.3.0/24)   │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ TAB 4: TAGS                                                      │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Environment: production                                    │   │
│ │ Team:        devops                                        │   │
│ │ CostCenter:  CC-1001                                       │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│ TAB 5: REVIEW + CREATE                                           │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

What you get:
├── VNet with address space 10.0.0.0/16
├── Subnets (web, app, db) with specified CIDRs
├── Azure-provided DNS
├── No NSGs (must create and attach separately!)
├── No route tables (uses system routes)
├── No NAT Gateway (must create separately if needed)
└── No Internet Gateway needed (Azure handles this differently)

⚠️ Post-creation steps:
├── Create NSGs and attach to subnets
├── Create NAT Gateway for outbound internet (private subnets)
├── Create Azure Bastion for secure SSH/RDP
├── Set up Service Endpoints or Private Endpoints
└── Create route tables (UDRs) if needed
```

---

## Part 10: VNet Creation — CLI

```bash
# 1. Create resource group
az group create \
  --name rg-prod-networking \
  --location centralindia

# 2. Create VNet with first subnet
az network vnet create \
  --name prod-vnet \
  --resource-group rg-prod-networking \
  --location centralindia \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name web \
  --subnet-prefixes 10.0.1.0/24 \
  --tags Environment=production Team=devops

# 3. Add more subnets
az network vnet subnet create \
  --name app \
  --vnet-name prod-vnet \
  --resource-group rg-prod-networking \
  --address-prefixes 10.0.2.0/24 \
  --service-endpoints Microsoft.Storage Microsoft.Sql

az network vnet subnet create \
  --name db \
  --vnet-name prod-vnet \
  --resource-group rg-prod-networking \
  --address-prefixes 10.0.3.0/24 \
  --service-endpoints Microsoft.Sql Microsoft.KeyVault

# Gateway subnet for VPN Gateway
az network vnet subnet create \
  --name GatewaySubnet \
  --vnet-name prod-vnet \
  --resource-group rg-prod-networking \
  --address-prefixes 10.0.255.0/27

# Bastion subnet
az network vnet subnet create \
  --name AzureBastionSubnet \
  --vnet-name prod-vnet \
  --resource-group rg-prod-networking \
  --address-prefixes 10.0.254.0/26

# 4. Create NSGs
az network nsg create \
  --name web-nsg \
  --resource-group rg-prod-networking \
  --location centralindia

# Allow HTTP/HTTPS inbound
az network nsg rule create \
  --nsg-name web-nsg \
  --resource-group rg-prod-networking \
  --name AllowHTTP \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 80 443 \
  --source-address-prefixes '*'

# Allow SSH from Bastion subnet only
az network nsg rule create \
  --nsg-name web-nsg \
  --resource-group rg-prod-networking \
  --name AllowSSHFromBastion \
  --priority 200 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 22 \
  --source-address-prefixes 10.0.254.0/26

# 5. Attach NSG to subnet
az network vnet subnet update \
  --name web \
  --vnet-name prod-vnet \
  --resource-group rg-prod-networking \
  --network-security-group web-nsg

# 6. Create NAT Gateway for outbound internet
az network public-ip create \
  --name nat-pip \
  --resource-group rg-prod-networking \
  --location centralindia \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3

az network nat gateway create \
  --name prod-nat-gateway \
  --resource-group rg-prod-networking \
  --location centralindia \
  --public-ip-addresses nat-pip \
  --idle-timeout 10

# Attach NAT Gateway to subnets (for outbound internet)
az network vnet subnet update \
  --name app \
  --vnet-name prod-vnet \
  --resource-group rg-prod-networking \
  --nat-gateway prod-nat-gateway

az network vnet subnet update \
  --name db \
  --vnet-name prod-vnet \
  --resource-group rg-prod-networking \
  --nat-gateway prod-nat-gateway

# 7. Add additional address space (if needed later)
az network vnet update \
  --name prod-vnet \
  --resource-group rg-prod-networking \
  --address-prefixes 10.0.0.0/16 172.16.0.0/16

# 8. List VNet details
az network vnet show \
  --name prod-vnet \
  --resource-group rg-prod-networking \
  --output table

az network vnet subnet list \
  --vnet-name prod-vnet \
  --resource-group rg-prod-networking \
  --output table
```

---

## Part 11: VNet Creation — Bicep

```bicep
// infra/vnet.bicep

param location string = 'centralindia'
param environment string = 'production'

// ──────────────────────────────────────
// Virtual Network
// ──────────────────────────────────────
resource vnet 'Microsoft.Network/virtualNetworks@2023-09-01' = {
  name: 'prod-vnet'
  location: location
  tags: {
    Environment: environment
    Team: 'devops'
  }
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: 'web'
        properties: {
          addressPrefix: '10.0.1.0/24'
          networkSecurityGroup: {
            id: webNsg.id
          }
          serviceEndpoints: [
            { service: 'Microsoft.Storage' }
          ]
        }
      }
      {
        name: 'app'
        properties: {
          addressPrefix: '10.0.2.0/24'
          networkSecurityGroup: {
            id: appNsg.id
          }
          natGateway: {
            id: natGateway.id
          }
          serviceEndpoints: [
            { service: 'Microsoft.Storage' }
            { service: 'Microsoft.Sql' }
          ]
        }
      }
      {
        name: 'db'
        properties: {
          addressPrefix: '10.0.3.0/24'
          networkSecurityGroup: {
            id: dbNsg.id
          }
          natGateway: {
            id: natGateway.id
          }
          serviceEndpoints: [
            { service: 'Microsoft.Sql' }
            { service: 'Microsoft.KeyVault' }
          ]
        }
      }
      {
        name: 'GatewaySubnet'
        properties: {
          addressPrefix: '10.0.255.0/27'
        }
      }
      {
        name: 'AzureBastionSubnet'
        properties: {
          addressPrefix: '10.0.254.0/26'
        }
      }
    ]
  }
}

// ──────────────────────────────────────
// Network Security Groups
// ──────────────────────────────────────
resource webNsg 'Microsoft.Network/networkSecurityGroups@2023-09-01' = {
  name: 'web-nsg'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowHTTP'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRanges: ['80', '443']
        }
      }
      {
        name: 'AllowSSHFromBastion'
        properties: {
          priority: 200
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: '10.0.254.0/26'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRanges: ['22']
        }
      }
    ]
  }
}

resource appNsg 'Microsoft.Network/networkSecurityGroups@2023-09-01' = {
  name: 'app-nsg'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowFromWeb'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: '10.0.1.0/24'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRanges: ['8080', '3000']
        }
      }
    ]
  }
}

resource dbNsg 'Microsoft.Network/networkSecurityGroups@2023-09-01' = {
  name: 'db-nsg'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowSQLFromApp'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: '10.0.2.0/24'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRanges: ['1433']
        }
      }
    ]
  }
}

// ──────────────────────────────────────
// NAT Gateway
// ──────────────────────────────────────
resource natPip 'Microsoft.Network/publicIPAddresses@2023-09-01' = {
  name: 'nat-pip'
  location: location
  sku: {
    name: 'Standard'
  }
  zones: ['1', '2', '3']
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

resource natGateway 'Microsoft.Network/natGateways@2023-09-01' = {
  name: 'prod-nat-gateway'
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    idleTimeoutInMinutes: 10
    publicIpAddresses: [
      { id: natPip.id }
    ]
  }
}

// ──────────────────────────────────────
// Outputs
// ──────────────────────────────────────
output vnetId string = vnet.id
output webSubnetId string = vnet.properties.subnets[0].id
output appSubnetId string = vnet.properties.subnets[1].id
output dbSubnetId string = vnet.properties.subnets[2].id
```

---

## Part 12: Multi-Tier Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│            STANDARD 3-TIER AZURE ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                         INTERNET                                     │
│                            │                                          │
│                    ┌───────┴────────┐                                │
│                    │ Azure Load     │                                 │
│                    │ Balancer (Std) │                                 │
│                    │ Public IP      │                                 │
│                    └───────┬────────┘                                │
│                            │                                          │
│  ┌─────────────────────────┼─────────────────────────────────────┐  │
│  │         VNet: prod-vnet (10.0.0.0/16)                         │  │
│  │         Region: Central India                                  │  │
│  │         (Subnets span ALL AZs automatically)                  │  │
│  │                         │                                       │  │
│  │  ┌─────────────────────┼──────────────────────────────────┐  │  │
│  │  │    web subnet: 10.0.1.0/24                  [web-nsg]   │  │  │
│  │  │                      │                                   │  │  │
│  │  │    AZ 1              │  AZ 2             AZ 3           │  │  │
│  │  │    ┌──────────┐     │  ┌──────────┐    ┌──────────┐   │  │  │
│  │  │    │ Web VM   │     │  │ Web VM   │    │ Web VM   │   │  │  │
│  │  │    │ (VMSS)   │     │  │ (VMSS)   │    │ (VMSS)   │   │  │  │
│  │  │    └──────────┘     │  └──────────┘    └──────────┘   │  │  │
│  │  └─────────────────────┼──────────────────────────────────┘  │  │
│  │                         │                                       │  │
│  │  ┌─────────────────────┼──────────────────────────────────┐  │  │
│  │  │    app subnet: 10.0.2.0/24                  [app-nsg]   │  │  │
│  │  │                      │                                   │  │  │
│  │  │    ┌──────────┐     │  ┌──────────┐    ┌──────────┐   │  │  │
│  │  │    │ App VM   │     │  │ App VM   │    │ App VM   │   │  │  │
│  │  │    │ (VMSS)   │     │  │ (VMSS)   │    │ (VMSS)   │   │  │  │
│  │  │    └──────────┘     │  └──────────┘    └──────────┘   │  │  │
│  │  └─────────────────────┼──────────────────────────────────┘  │  │
│  │                         │                                       │  │
│  │  ┌─────────────────────┼──────────────────────────────────┐  │  │
│  │  │    db subnet: 10.0.3.0/24                   [db-nsg]    │  │  │
│  │  │                      │                                   │  │  │
│  │  │    ┌──────────────────────────────────────────────┐    │  │  │
│  │  │    │ Azure SQL (Private Endpoint → 10.0.3.10)     │    │  │  │
│  │  │    │ Zone-redundant HA                            │    │  │  │
│  │  │    └──────────────────────────────────────────────┘    │  │  │
│  │  └─────────────────────┼──────────────────────────────────┘  │  │
│  │                         │                                       │  │
│  │  ┌──────────────────────────────────┐                          │  │
│  │  │ AzureBastionSubnet: 10.0.254.0/26│                          │  │
│  │  │ Azure Bastion (secure SSH/RDP)    │                          │  │
│  │  └──────────────────────────────────┘                          │  │
│  │                                                                 │  │
│  │  ┌──────────────────────────┐                                   │  │
│  │  │ NAT Gateway (outbound)  │                                    │  │
│  │  │ Attached to app + db    │                                    │  │
│  │  │ subnets                 │                                    │  │
│  │  └──────────────────────────┘                                   │  │
│  │                                                                 │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Key differences from AWS:                                            │
│ ├── No IGW needed — outbound works with Public IP or NAT GW        │
│ ├── Subnets span ALL AZs (no per-AZ subnets needed!)               │
│ ├── NSGs instead of Security Groups + NACLs                        │
│ ├── Azure Bastion replaces bastion hosts (managed service)         │
│ ├── Private Endpoint for Azure SQL (private IP in your VNet)       │
│ ├── NAT Gateway attaches to subnets (not route tables)             │
│ └── VMSS across AZs in same subnet                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: VNet Limits and Quotas

```
┌────────────────────────────────────┬────────────────┬──────────────┐
│ Resource                           │ Default Limit  │ Max Limit    │
├────────────────────────────────────┼────────────────┼──────────────┤
│ VNets per subscription per region  │ 1,000          │ 1,000        │
│ Subnets per VNet                   │ 3,000          │ 3,000        │
│ Address spaces per VNet            │ 500            │ 500          │
│ VNet peerings per VNet             │ 500            │ 500          │
│ Private Endpoints per VNet         │ 1,000          │ Yes          │
│ Public IPs per subscription        │ Varies by SKU  │ Yes          │
│ NSGs per subscription              │ 5,000          │ 5,000        │
│ Rules per NSG                      │ 1,000          │ 1,000        │
│ NICs per subscription              │ 65,536         │ 65,536       │
│ Route tables per subscription      │ 200            │ Yes          │
│ Routes per route table             │ 400            │ 400          │
│ NAT Gateways per subscription      │ 100            │ Yes          │
│ DNS servers per VNet               │ 25             │ 25           │
│ Service endpoints per subnet       │ 25             │ 25           │
│ Private endpoints per subnet       │ 1,000          │ Yes          │
└────────────────────────────────────┴────────────────┴──────────────┘

💡 Azure generally has higher default limits than AWS.
   Request increases via Azure Portal → Quotas.
```

---

## Part 14: Real-World Patterns

### Startup (5-10 developers)

```
Architecture:
├── 1 VNet per environment (prod + dev) = 2 VNets
│   Or 1 VNet with separate subnets per tier
├── VNet CIDR: 10.0.0.0/16 (prod), 10.1.0.0/16 (dev)
├── 3 subnets: web, app, db
├── NSG per subnet (not per NIC)
├── NAT Gateway for outbound ($32/month)
├── Azure Bastion (if SSH needed) or just use Azure Serial Console
├── Service Endpoints for Storage + SQL (free!)
├── No VPN Gateway (not needed yet)
└── Total VNet cost: ~$32/month (NAT GW) + ~$4/month (public IPs)

Do:
├── Create VNet before deploying anything
├── Use NSGs on every subnet
├── Use Azure Bastion or Serial Console (no public SSH!)
├── Enable Service Endpoints (free, improves security)
├── Use Private IPs for VMs (NAT GW for outbound)
├── Tag everything
└── Use Standard SKU for public IPs

Don't:
├── Let portal auto-create VNets per VM
├── Put public IPs on every VM
├── Open RDP/SSH to 0.0.0.0/0
├── Skip NSGs
└── Use Basic public IPs (being retired)
```

### Mid-Size (50-100 developers)

```
Architecture:
├── Hub-and-spoke topology:
│   ├── Hub VNet: Shared services (Bastion, Firewall, VPN GW)
│   ├── Spoke: Prod VNet
│   ├── Spoke: Staging VNet
│   ├── Spoke: Dev VNet
│   └── All peered to Hub (or Azure Virtual WAN)
├── Azure Firewall in Hub ($912/month — but centralized security)
│   Or NVA (Network Virtual Appliance) like Palo Alto
├── Azure Bastion in Hub (shared across spokes via peering)
├── VPN Gateway in Hub (connect to on-premises)
├── NSGs on every subnet
├── Private Endpoints for PaaS services (SQL, Storage, Key Vault)
├── Private DNS Zones linked to VNets
├── UDRs (User Defined Routes) to force traffic through Firewall
└── Total: ~$1,000-2,000/month networking

Do:
├── Use hub-and-spoke topology
├── Centralize security (Firewall) in hub
├── Use Private Endpoints for all PaaS
├── Azure Private DNS Zones for name resolution
├── UDR tables for traffic inspection
├── VNet Flow Logs (via NSG flow logs)
├── Azure Network Watcher for diagnostics
└── Plan CIDR ranges carefully (no overlap for peering!)
```

### Enterprise (500+ developers)

```
Architecture:
├── Azure Virtual WAN (replaces manual hub-and-spoke)
│   ├── Virtual WAN Hub per region
│   ├── Spokes auto-connect (VNet connections)
│   ├── Built-in VPN, ExpressRoute, Firewall integration
│   └── Global transit routing
├── Or manual hub-and-spoke with Azure Firewall Premium
├── ExpressRoute for on-premises (private circuit)
│   ├── 50 Mbps to 10 Gbps
│   ├── Private peering for VNet access
│   ├── Microsoft peering for M365/Azure services
│   └── Redundant circuits (different providers)
├── Azure Firewall Premium (TLS inspection, IDPS)
├── Azure DDoS Protection Standard (for public-facing)
├── Azure Private Link everywhere (zero-trust network)
├── Multiple regions with global VNet peering
├── Azure DNS Private Resolver for hybrid DNS
├── Network security perimeter (preview)
└── Total: $3,000-15,000+/month networking

Advanced:
├── Azure Landing Zone (Cloud Adoption Framework)
├── Centralized network management via Azure Policy
├── Cross-region load balancing (Front Door / Traffic Manager)
├── ExpressRoute Global Reach (connect two on-prem sites via Azure)
├── Azure Route Server for NVA BGP peering
├── Service chaining through NVAs
├── Encrypted VNet peering (preview)
└── IP Group and Application Rule Collections for Firewall
```

---

## Quick Reference

| Component | Purpose | Cost |
|-----------|---------|------|
| VNet | Virtual network | FREE |
| Subnet | Network segment (all AZs) | FREE |
| NSG | Firewall rules | FREE |
| Route table (UDR) | Custom routing | FREE |
| Public IP (Standard) | Static public address | $3.65/mo |
| NAT Gateway | Outbound internet for private subnets | $32/mo + $0.045/GB |
| Azure Bastion | Secure SSH/RDP (no public IPs) | $140/mo (Standard) |
| VNet Peering | Connect two VNets | $0.01/GB |
| VPN Gateway | Site-to-site VPN | $140-1,600/mo |
| ExpressRoute | Private circuit to on-prem | $55-16,000/mo |
| Azure Firewall | Central network firewall | $912/mo (Standard) |
| Service Endpoint | Private path to PaaS | FREE |
| Private Endpoint | Private IP for PaaS resource | $7.30/mo |

---

## What's Next?

In the next chapter, we'll cover VNet Advanced Networking — VNet Peering, VPN Gateway, ExpressRoute, Virtual WAN, and Private Link.

→ Next: [Chapter 7: VNet Advanced Networking](07-vnet-advanced.md)

---

*Last Updated: May 2026*
