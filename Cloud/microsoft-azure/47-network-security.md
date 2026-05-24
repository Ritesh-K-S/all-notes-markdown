# Chapter 47: Azure Network Security

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Network Security Overview](#part-1-network-security-overview)
- [Part 2: Azure Firewall](#part-2-azure-firewall)
- [Part 3: Web Application Firewall (WAF)](#part-3-web-application-firewall-waf)
- [Part 4: DDoS Protection](#part-4-ddos-protection)
- [Part 5: Private Link & Private Endpoints](#part-5-private-link--private-endpoints)
- [Part 6: Service Endpoints](#part-6-service-endpoints)
- [Part 7: Azure Bastion](#part-7-azure-bastion)
- [Part 8: Terraform & az CLI Reference](#part-8-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure provides multiple layers of network security — from DDoS protection at the network edge, to firewalls for traffic filtering, to private endpoints for eliminating public internet exposure entirely. This chapter covers all the network security services you need to protect your Azure resources.

```
What you'll learn:
├── Network Security Overview (defense in depth)
├── Azure Firewall (network-level firewall)
├── WAF (web application firewall, OWASP protection)
├── DDoS Protection (volumetric attack mitigation)
├── Private Link & Private Endpoints (no public internet)
├── Service Endpoints (VNet-to-PaaS shortcuts)
├── Azure Bastion (secure VM access without public IP)
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Network Security Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│           DEFENSE IN DEPTH                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Internet traffic → Your Azure resources                            │
│                                                                       │
│ Layer 1: DDoS Protection                                            │
│ ├── Absorbs volumetric attacks (terabits of traffic)             │
│ └── Always-on, automatic                                         │
│                                                                       │
│ Layer 2: WAF (Web Application Firewall)                            │
│ ├── Blocks SQL injection, XSS, OWASP Top 10                    │
│ └── On Application Gateway or Front Door                        │
│                                                                       │
│ Layer 3: Azure Firewall                                             │
│ ├── Network-level rules (IP, port, protocol, FQDN)             │
│ └── Central hub firewall for all VNets                          │
│                                                                       │
│ Layer 4: NSG (Network Security Groups)                             │
│ ├── Subnet and NIC-level rules                                  │
│ └── Covered in Chapter 12                                       │
│                                                                       │
│ Layer 5: Private Endpoints                                          │
│ ├── PaaS services accessible only within VNet                   │
│ └── No public internet exposure                                  │
│                                                                       │
│ Layer 6: Application-level security                                 │
│ ├── Authentication (Entra ID)                                    │
│ ├── Authorization (RBAC)                                         │
│ └── Encryption (TLS, Key Vault)                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Azure Firewall

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE FIREWALL                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure Firewall = Managed, cloud-native network firewall            │
│                                                                       │
│ What it does:                                                        │
│ ├── Filter traffic between VNets and internet                    │
│ ├── FQDN filtering (allow *.microsoft.com, block everything else)│
│ ├── Network rules (allow/deny by IP, port, protocol)           │
│ ├── Application rules (HTTP/HTTPS URL filtering)                │
│ ├── Threat intelligence (block known malicious IPs)             │
│ └── Centralized logging of all network traffic                  │
│                                                                       │
│ SKUs:                                                                │
│ ├── Standard: Network/app rules, threat intel, NAT             │
│ │   ~$1.25/hr + $0.016/GB processed                            │
│ ├── Premium: + TLS inspection, IDPS, URL filtering, web categories│
│ │   ~$1.75/hr + $0.016/GB processed                            │
│ └── Basic: Simplified, for SMBs (~$0.395/hr)                   │
│                                                                       │
│ Architecture:                                                        │
│ ┌─────────┐    ┌──────────────────┐    ┌─────────────┐          │
│ │ Internet│───→│ Azure Firewall   │───→│ Spoke VNets│          │
│ │         │    │ (Hub VNet)       │    │ (workloads) │          │
│ │         │    │ Rules:           │    │             │          │
│ │         │    │ Allow: 443, 80   │    │ ├── VNet-A │          │
│ │         │    │ Block: everything│    │ └── VNet-B │          │
│ └─────────┘    └──────────────────┘    └─────────────┘          │
│                                                                       │
│ Console → Firewalls → Create                                     │
│ Name: [fw-hub-prod]                                                │
│ Region: [Central India ▼]                                          │
│ SKU: [Standard ▼]                                                 │
│ VNet: [vnet-hub] (needs a subnet named AzureFirewallSubnet, /26) │
│ Public IP: [pip-fw-hub]                                            │
│ Firewall Policy: [Create new ▼]                                  │
│                                                                       │
│ ⚡ Route all outbound traffic through Azure Firewall using UDR  │
│ (User Defined Routes) on spoke subnet route tables.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Web Application Firewall (WAF)

```
┌─────────────────────────────────────────────────────────────────────┐
│           WAF (Web Application Firewall)                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ WAF protects web apps from common web attacks:                     │
│ ├── SQL Injection ('; DROP TABLE users; --)                      │
│ ├── Cross-Site Scripting (XSS)                                   │
│ ├── Remote Code Execution                                        │
│ ├── Local File Inclusion                                         │
│ ├── HTTP Protocol Violations                                     │
│ └── Other OWASP Top 10 vulnerabilities                           │
│                                                                       │
│ Where to deploy WAF:                                                │
│ ├── Application Gateway WAF v2 (regional, L7 load balancer)   │
│ ├── Azure Front Door WAF (global, CDN + WAF)                   │
│ └── Azure CDN WAF                                                │
│                                                                       │
│ WAF modes:                                                           │
│ ├── Detection: Log attacks but don't block (testing)            │
│ └── Prevention: Block attacks + log (production)                │
│                                                                       │
│ WAF Policy:                                                          │
│ Console → Web Application Firewall policies → Create            │
│ Policy for: [Application Gateway ▼]                               │
│ Mode: [Prevention ▼]                                               │
│ Managed rules: OWASP 3.2 (default rule set)                     │
│ Custom rules:                                                        │
│ ├── Rate limiting (block if > 100 requests/min from same IP)  │
│ ├── Geo-blocking (block traffic from certain countries)        │
│ └── IP allow/block lists                                        │
│                                                                       │
│ ⚡ WAF protects web apps (HTTP/HTTPS layer 7)                   │
│ ⚡ Azure Firewall protects networks (layer 3/4)                 │
│ ⚡ Use BOTH for defense in depth!                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: DDoS Protection

```
┌─────────────────────────────────────────────────────────────────────┐
│           DDoS PROTECTION                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ DDoS = Distributed Denial of Service                                │
│ Attacker floods your app with so much traffic it goes down.       │
│                                                                       │
│ Tiers:                                                               │
│ ├── DDoS Infrastructure Protection (free, default)              │
│ │   ├── Always-on for all Azure resources                       │
│ │   ├── Protects against common network-layer attacks           │
│ │   └── Same protection Microsoft uses for its own services    │
│ │                                                                  │
│ ├── DDoS Network Protection ($2,944/month + overage)           │
│ │   ├── Enhanced mitigation for your VNet resources             │
│ │   ├── Adaptive tuning (learns your traffic patterns)         │
│ │   ├── Attack analytics & metrics                              │
│ │   ├── DDoS Rapid Response (DRR) team support                │
│ │   ├── Cost protection (credit for scale-out during attack)  │
│ │   └── WAF included (no additional WAF cost)                  │
│ │                                                                  │
│ └── DDoS IP Protection ($199/month per public IP)              │
│     ├── Same protection as Network tier                         │
│     ├── Per-IP pricing (cheaper for few resources)             │
│     └── No DRR team or cost protection                         │
│                                                                       │
│ Enable DDoS Network Protection:                                     │
│ Console → DDoS protection plans → Create                        │
│ Then: VNet → DDoS protection → Enable → Select plan           │
│                                                                       │
│ ⚡ Free tier is usually sufficient for most workloads.           │
│ ⚡ Network tier for business-critical, high-traffic apps.       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Private Link & Private Endpoints

```
┌─────────────────────────────────────────────────────────────────────┐
│           PRIVATE LINK & PRIVATE ENDPOINTS                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: PaaS services (Storage, SQL, Key Vault) have public     │
│ endpoints by default. Traffic goes over the internet.             │
│                                                                       │
│ Without Private Endpoint:                                           │
│ VM in VNet → Internet → Storage Account (public endpoint)      │
│ ⚡ Traffic leaves your VNet, goes over public internet!         │
│                                                                       │
│ With Private Endpoint:                                              │
│ VM in VNet → Private Endpoint (private IP) → Storage Account  │
│ ⚡ Traffic stays within Azure backbone, never touches internet! │
│                                                                       │
│ Create Private Endpoint:                                            │
│ Any PaaS resource → Networking → Private endpoint connections  │
│ [+ Private endpoint]                                               │
│ Name: [pe-storage-prod]                                            │
│ VNet: [vnet-prod ▼]                                               │
│ Subnet: [subnet-private-endpoints ▼]                              │
│ Target sub-resource: [blob ▼]                                     │
│ DNS: ☑ Integrate with private DNS zone                          │
│                                                                       │
│ After creation:                                                      │
│ Private endpoint gets a private IP (e.g., 10.0.5.4)             │
│ DNS resolves mystorageaccount.blob.core.windows.net → 10.0.5.4│
│ ⚡ Disable public access on the PaaS resource for full security!│
│                                                                       │
│ Supported services:                                                  │
│ ├── Storage, SQL Database, Cosmos DB, Key Vault                 │
│ ├── App Service, Azure Cache for Redis, Event Hub               │
│ ├── ACR, AKS, Cognitive Services, and many more                │
│ └── Almost every PaaS service supports Private Link             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Service Endpoints

```
┌─────────────────────────────────────────────────────────────────────┐
│           SERVICE ENDPOINTS vs PRIVATE ENDPOINTS                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Service Endpoints:                                                   │
│ ├── Extend VNet identity to Azure PaaS services                 │
│ ├── Traffic uses Azure backbone (not internet)                  │
│ ├── PaaS resource still has public IP                            │
│ ├── Free!                                                         │
│ └── Configure: VNet → Subnet → Service endpoints → Add       │
│                                                                       │
│ Private Endpoints:                                                   │
│ ├── PaaS resource gets a private IP in your VNet                │
│ ├── Traffic never leaves your VNet                               │
│ ├── Can disable public IP entirely                               │
│ ├── Works across VNet peering and VPN                           │
│ ├── Paid (~$7.30/month per endpoint + data processing)         │
│ └── More secure than service endpoints                          │
│                                                                       │
│ ┌─────────────────┬───────────────────┬──────────────────┐       │
│ │ Feature         │ Service Endpoint  │ Private Endpoint │       │
│ ├─────────────────┼───────────────────┼──────────────────┤       │
│ │ Cost            │ Free ✅           │ Paid             │       │
│ │ Public IP       │ Still public      │ Private IP only  │       │
│ │ Cross-VNet      │ No                │ Yes ✅           │       │
│ │ On-premises     │ No                │ Yes (via VPN) ✅ │       │
│ │ DNS             │ No change         │ Private DNS zone │       │
│ │ Setup           │ Very simple       │ More complex     │       │
│ │ Security        │ Good              │ Best ✅          │       │
│ └─────────────────┴───────────────────┴──────────────────┘       │
│                                                                       │
│ ⚡ Use Private Endpoints for production (more secure).           │
│ ⚡ Use Service Endpoints for dev/test (free, simpler).          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Azure Bastion

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE BASTION                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: To RDP/SSH into a VM, you need a public IP on the VM.  │
│ Public IP = exposed to internet attacks (brute force, scanning). │
│                                                                       │
│ Solution: Azure Bastion = Secure RDP/SSH through the browser.   │
│ VM has NO public IP. You connect via Azure Portal over TLS.     │
│                                                                       │
│ Without Bastion:                                                    │
│ You → Internet → VM Public IP (port 3389/22 open) 🔴 Risky   │
│                                                                       │
│ With Bastion:                                                       │
│ You → Azure Portal (HTTPS) → Bastion → VM (private IP) ✅    │
│                                                                       │
│ Create:                                                              │
│ Console → Bastions → Create                                      │
│ Name: [bastion-prod]                                               │
│ VNet: [vnet-prod ▼]                                               │
│ Subnet: AzureBastionSubnet (/26 minimum, auto-created)          │
│ Public IP: [pip-bastion-prod]                                     │
│ SKU: [Standard ▼] (Basic or Standard)                            │
│                                                                       │
│ Connect to VM:                                                       │
│ VM → Connect → Bastion → Enter credentials → [Connect]        │
│ ⚡ Opens RDP/SSH session in your browser tab!                   │
│ ⚡ No RDP client needed, no public IP on VM needed.             │
│                                                                       │
│ Pricing: ~$0.19/hr (Basic), ~$0.29/hr (Standard)               │
│                                                                       │
│ Standard SKU extras:                                                │
│ ├── Native client support (az network bastion rdp/ssh)         │
│ ├── File upload/download                                         │
│ ├── Shareable link (share VM access without portal)             │
│ └── Scale instances (2-50 for more concurrent sessions)        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform & az CLI Reference

### Terraform

```hcl
# Private Endpoint for Storage
resource "azurerm_private_endpoint" "storage" {
  name                = "pe-storage-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  subnet_id           = azurerm_subnet.private_endpoints.id

  private_service_connection {
    name                           = "storage-connection"
    private_connection_resource_id = azurerm_storage_account.main.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "default"
    private_dns_zone_ids = [azurerm_private_dns_zone.blob.id]
  }
}
```

### Bicep

```bicep
// Private Endpoint
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-09-01' = {
  name: 'pe-storage-blob'
  location: resourceGroup().location
  properties: {
    subnet: { id: subnet.id }
    privateLinkServiceConnections: [
      {
        name: 'storage-connection'
        properties: {
          privateLinkServiceId: storageAccount.id
          groupIds: ['blob']
        }
      }
    ]
  }
}

// Azure Firewall
resource firewall 'Microsoft.Network/azureFirewalls@2023-09-01' = {
  name: 'fw-hub-prod'
  location: resourceGroup().location
  properties: {
    sku: { name: 'AZFW_VNet', tier: 'Standard' }
    ipConfigurations: [
      {
        name: 'fw-config'
        properties: {
          subnet: { id: firewallSubnet.id }
          publicIPAddress: { id: firewallPip.id }
        }
      }
    ]
  }
}
```

```bash
# Create Azure Firewall
az network firewall create \
  --name fw-hub-prod \
  --resource-group rg-network \
  --location centralindia \
  --vnet-name vnet-hub

# Create Private Endpoint
az network private-endpoint create \
  --name pe-storage-prod \
  --resource-group rg-myapp-prod \
  --vnet-name vnet-prod \
  --subnet subnet-pe \
  --private-connection-resource-id <storage-account-id> \
  --group-id blob \
  --connection-name storage-connection

# Create DDoS protection plan
az network ddos-protection create \
  --name ddos-plan-prod \
  --resource-group rg-network

# Create Bastion
az network bastion create \
  --name bastion-prod \
  --resource-group rg-network \
  --vnet-name vnet-prod \
  --public-ip-address pip-bastion

# Connect to VM via Bastion (native client)
az network bastion ssh \
  --name bastion-prod \
  --resource-group rg-network \
  --target-resource-id <vm-id> \
  --auth-type password --username azureuser

# Delete Bastion
az network bastion delete --name bastion-prod --resource-group rg-network
```

---

## Real-World Patterns

### Pattern 1: Hub-Spoke with Private Endpoints

```
┌─────────────────────────────────────────────────┐
│  Zero-Trust Network Architecture                │
├─────────────────────────────────────────────────┤
│                                                 │
│  Hub VNet (Shared Services)                     │
│  ├── Azure Firewall (inspect all traffic)       │
│  ├── VPN Gateway (on-prem connectivity)        │
│  └── Private DNS Zones                        │
│       │                                         │
│  ┌────┼──────────┐  VNet Peering              │
│  ▼              ▼                                │
│  Spoke 1       Spoke 2                          │
│  (App VNet)    (Data VNet)                      │
│  ├── App Svc   ├── SQL (Private Endpoint)      │
│  ├── AKS       ├── Storage (Private Endpoint)  │
│  └── NSGs      └── Cosmos (Private Endpoint)   │
│                                                 │
│  All PaaS traffic stays on Microsoft backbone   │
│  No public endpoints exposed                    │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Defense in depth (outside → inside):
  DDoS Protection → WAF → Azure Firewall → NSG → Private Endpoint

Azure Firewall: Network-level (L3/L4), FQDN filtering, central hub
WAF: Web app-level (L7), OWASP Top 10, SQL injection, XSS
DDoS: Volumetric attack mitigation (free basic, paid enhanced)

Private Endpoint: PaaS gets private IP in your VNet (most secure)
Service Endpoint: VNet-to-PaaS shortcut (free, simpler, less secure)
Azure Bastion: Secure RDP/SSH via browser (no public IP on VM)

Cost comparison:
  DDoS Infrastructure: Free
  Service Endpoints: Free
  Azure Bastion: ~$0.19-0.29/hr
  Private Endpoints: ~$7.30/mo per endpoint
  Azure Firewall: ~$1.25/hr
  DDoS Network: ~$2,944/mo
```

---

## What's Next?

Next chapter: [Chapter 48: Azure Service Bus](48-service-bus.md) — Enterprise messaging with queues, topics, and subscriptions.
