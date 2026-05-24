# Chapter 9: Azure DNS & Private DNS Zones

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure DNS Fundamentals](#part-1-azure-dns-fundamentals)
- [Part 2: Alias Records](#part-2-alias-records)
- [Part 3: Private DNS Zones](#part-3-private-dns-zones)
- [Part 4: Azure DNS Private Resolver](#part-4-azure-dns-private-resolver)
- [Part 5: Azure Traffic Manager (DNS-Based Routing)](#part-5-azure-traffic-manager-dns-based-routing)
- [Part 6: DNSSEC](#part-6-dnssec)
- [Part 7: Bicep Example](#part-7-bicep-example)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure DNS hosts your DNS domains and provides name resolution using Microsoft Azure's global infrastructure. Azure also provides Private DNS Zones for VNet-internal resolution, Alias records for pointing to Azure resources, and Azure DNS Private Resolver for hybrid DNS scenarios.

```
What you'll learn:
├── Azure DNS Components
│   ├── DNS Zones (public)
│   ├── Record Sets (A, AAAA, CNAME, MX, TXT, etc.)
│   ├── Alias Records (Azure's answer to AWS ALIAS)
│   └── All fields explained
├── Private DNS Zones
│   ├── VNet linking (resolution & auto-registration)
│   ├── Private Endpoint DNS (privatelink zones)
│   └── Cross-VNet resolution
├── Azure DNS Private Resolver
│   ├── Inbound endpoints
│   ├── Outbound endpoints
│   └── DNS forwarding rulesets
├── Azure Traffic Manager (DNS routing)
│   ├── Routing methods (weighted, priority, geographic, etc.)
│   ├── Health probes
│   └── Nested profiles
├── Domain Registration (App Service Domains)
└── Real-world patterns
```

---

## Part 1: Azure DNS Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│                 AZURE DNS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed, authoritative DNS hosting on Azure's global network. │
│                                                                       │
│ Key facts:                                                           │
│ ├── Uses Azure's global Anycast name server network                │
│ ├── 100% availability SLA                                          │
│ ├── Supports public and private DNS zones                          │
│ ├── Alias records for Azure resources (like AWS ALIAS)            │
│ ├── Role-based access control (RBAC) on zones & records           │
│ ├── Activity log & Azure Monitor integration                      │
│ ├── Azure Policy support for DNS governance                       │
│ └── Cannot register domains (use App Service Domains or external) │
│                                                                       │
│ ⚠️ Azure DNS is for HOSTING only. NOT a registrar!                  │
│    (Unlike AWS Route 53 which is both)                              │
│                                                                       │
│ Azure DNS vs AWS Route 53 vs GCP Cloud DNS:                         │
│ ┌──────────────────┬──────────────┬──────────────┬───────────────┐ │
│ │ Feature          │ Route 53     │ Cloud DNS    │ Azure DNS     │ │
│ │ SLA              │ 100%         │ 100%         │ 100%          │ │
│ │ Alias record     │ ALIAS        │ None (use A) │ Alias         │ │
│ │ DNS routing      │ 7 policies   │ None         │ Traffic Mgr   │ │
│ │ Health checks    │ Built-in     │ None         │ Traffic Mgr   │ │
│ │ Domain register  │ Yes          │ Yes          │ Separate*     │ │
│ │ Private zones    │ Yes          │ Yes          │ Yes           │ │
│ │ Auto-registration│ No           │ No           │ Yes!          │ │
│ │ DNSSEC           │ Yes          │ Yes          │ Yes           │ │
│ │ Hybrid DNS       │ Resolver     │ DNS policies │ DNS Resolver  │ │
│ │ Response policy   │ No           │ Yes          │ No            │ │
│ │ RBAC per zone    │ IAM policies │ IAM          │ Azure RBAC    │ │
│ │ Zone cost        │ $0.50/mo     │ $0.20/mo     │ $0.50/mo      │ │
│ └──────────────────┴──────────────┴──────────────┴───────────────┘ │
│ * App Service Domains for registration                              │
│                                                                       │
│ ⚡ Unique Azure feature: Auto-registration!                         │
│    Private DNS zones can automatically register A records for      │
│    all VMs in a linked VNet. No manual record creation needed.     │
│                                                                       │
│ Pricing:                                                             │
│ ├── DNS zone: $0.50/month per zone (first 25, then $0.10)        │
│ ├── Queries: $0.40/million (first billion)                        │
│ ├── Alias queries to Azure resources: Charged normally            │
│ │   (unlike AWS where ALIAS queries are free)                     │
│ ├── Private DNS zone: $0.25/month per zone                        │
│ └── Private DNS queries: $0.40/million                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### DNS Zones (Public)

```
┌─────────────────────────────────────────────────────────────────────┐
│               PUBLIC DNS ZONE                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Hosts DNS records for a domain accessible from internet.      │
│                                                                       │
│ Key points:                                                          │
│ ├── Lives in a resource group (Azure-specific!)                    │
│ ├── RBAC at zone or record level                                   │
│ ├── Gets 4 Azure name servers                                     │
│ ├── Zone name = domain name                                       │
│ ├── Activity log for all DNS changes (audit trail)                │
│ └── Can delegate subdomains to child zones                        │
│                                                                       │
│ Subdomain delegation:                                                │
│   Parent zone: techcorp.com                                         │
│   Child zone: dev.techcorp.com (separate zone, separate RBAC)     │
│   Add NS record in parent pointing to child's name servers        │
│   Dev team manages dev.techcorp.com independently!                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a DNS Zone

```
Azure Portal → DNS zones → + Create

┌─────────────────────────────────────────────────────────────────┐
│              CREATE DNS ZONE                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Basics:                                                          │
│   Subscription:     [tc-production ▼]                           │
│   Resource group:   [rg-networking ▼]  [Create new]            │
│   Name:             [techcorp.com]                              │
│   Resource group location: (auto from RG)                      │
│                                                                   │
│ Tags:                                                            │
│   Environment: production                                        │
│   Team: platform                                                │
│                                                                   │
│ [Review + create] → [Create]                                    │
│                                                                   │
│ After creation, you get 4 name servers:                         │
│   ns1-04.azure-dns.com                                          │
│   ns2-04.azure-dns.net                                          │
│   ns3-04.azure-dns.org                                          │
│   ns4-04.azure-dns.info                                         │
│                                                                   │
│ ⚠️ Update NS at your domain registrar to these!                 │
│    Azure DNS zones ALWAYS use 4 name servers across 4 TLDs     │
│    (.com, .net, .org, .info) for maximum redundancy.            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create DNS zone
  az network dns zone create \
    --resource-group rg-networking \
    --name techcorp.com \
    --tags Environment=production Team=platform

  # List name servers
  az network dns zone show \
    --resource-group rg-networking \
    --name techcorp.com \
    --query nameServers \
    --output tsv
```

### Record Sets

```
┌─────────────────────────────────────────────────────────────────────┐
│                 RECORD SETS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure calls them "Record Sets" — a record set can contain          │
│ multiple records of the same type for the same name.               │
│                                                                       │
│ Supported types:                                                     │
│ ├── A: Domain → IPv4                                               │
│ ├── AAAA: Domain → IPv6                                            │
│ ├── CNAME: Domain → another domain                                 │
│ │   ⚠️ Cannot be at zone apex (same restriction as AWS/GCP)       │
│ │   ⚠️ Only ONE CNAME per record set (unlike A which can have     │
│ │      multiple IPs)                                               │
│ ├── MX: Mail exchange                                              │
│ ├── TXT: Text (SPF, DKIM, verification)                           │
│ │   Max 1024 chars per string, 4096 chars total per record set    │
│ ├── NS: Name servers (auto-created at apex)                       │
│ ├── SOA: Start of Authority (auto-created)                        │
│ ├── SRV: Service location                                          │
│ ├── CAA: Certificate Authority Authorization                      │
│ └── PTR: Reverse DNS (in special reverse zones)                   │
│                                                                       │
│ Record set limits:                                                   │
│ ├── Max 10,000 record sets per zone                                │
│ ├── Max 20 records per record set                                  │
│ └── Max TTL: 2,147,483,647 seconds                                │
│                                                                       │
│ Wildcard records supported:                                          │
│   *.techcorp.com A 20.100.50.60                                    │
│   Any subdomain not explicitly defined → matches wildcard          │
│                                                                       │
│ @ symbol = zone apex (techcorp.com itself)                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Records

```
Azure Portal → DNS zone → + Record set

┌─────────────────────────────────────────────────────────────────┐
│              ADD RECORD SET                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:     [api]  (creates api.techcorp.com)                    │
│           [@]    (for zone apex techcorp.com)                  │
│           [*]    (wildcard)                                     │
│                                                                   │
│ Type:     [A ▼]                                                │
│                                                                   │
│ Alias record set: ○ No  ● Yes                                 │
│                                                                   │
│ (If Alias = No):                                                │
│   TTL:    [300]                                                 │
│   TTL unit: [Seconds ▼]                                        │
│   IP address: [20.100.50.60]                                   │
│   [Add a record] (for round-robin)                             │
│                                                                   │
│ (If Alias = Yes):                                               │
│   Alias type:                                                    │
│   ├── [Azure resource ▼]                                       │
│   ├── Azure resource: [tc-prod-lb ▼]                           │
│   └── Auto-populates target                                    │
│                                                                   │
│ [OK]                                                             │
│                                                                   │
│ CNAME example:                                                   │
│   Name: blog                                                    │
│   Type: CNAME                                                   │
│   TTL: 3600                                                     │
│   Alias: techcorp.wordpress.com                                │
│                                                                   │
│ MX example:                                                      │
│   Name: @                                                       │
│   Type: MX                                                      │
│   TTL: 3600                                                     │
│   Mail exchange:                                                │
│     10 mail.techcorp.com                                        │
│     20 mail2.techcorp.com                                       │
│                                                                   │
│ TXT example:                                                     │
│   Name: @                                                       │
│   Type: TXT                                                     │
│   TTL: 3600                                                     │
│   Value: v=spf1 include:spf.protection.outlook.com ~all        │
│                                                                   │
│ CAA example:                                                     │
│   Name: @                                                       │
│   Type: CAA                                                     │
│   Flags: 0                                                      │
│   Tag: issue                                                    │
│   Value: letsencrypt.org                                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # A record
  az network dns record-set a add-record \
    --resource-group rg-networking \
    --zone-name techcorp.com \
    --record-set-name api \
    --ipv4-address 20.100.50.60 \
    --ttl 300

  # CNAME
  az network dns record-set cname set-record \
    --resource-group rg-networking \
    --zone-name techcorp.com \
    --record-set-name blog \
    --cname techcorp.wordpress.com

  # MX
  az network dns record-set mx add-record \
    --resource-group rg-networking \
    --zone-name techcorp.com \
    --record-set-name "@" \
    --exchange mail.techcorp.com \
    --preference 10

  # TXT
  az network dns record-set txt add-record \
    --resource-group rg-networking \
    --zone-name techcorp.com \
    --record-set-name "@" \
    --value "v=spf1 include:spf.protection.outlook.com ~all"

  # CAA
  az network dns record-set caa add-record \
    --resource-group rg-networking \
    --zone-name techcorp.com \
    --record-set-name "@" \
    --flags 0 \
    --tag issue \
    --value "letsencrypt.org"
```

---

## Part 2: Alias Records

```
┌─────────────────────────────────────────────────────────────────────┐
│                  ALIAS RECORDS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Point a DNS record directly to an Azure resource.             │
│       Azure's equivalent of AWS ALIAS records.                      │
│                                                                       │
│ Works with record types: A, AAAA, CNAME                            │
│ ⚡ CAN be used at zone apex (A/AAAA alias at @)!                    │
│                                                                       │
│ Supported target Azure resources:                                    │
│ ├── Azure Public IP (attached to LB, App Gateway, etc.)           │
│ ├── Azure Traffic Manager profile                                  │
│ ├── Azure CDN endpoint                                             │
│ ├── Azure Front Door                                               │
│ └── Another DNS record set in the SAME zone                       │
│                                                                       │
│ ⚠️ Cannot alias to:                                                 │
│ ├── Azure Private IP                                               │
│ ├── Cross-zone record sets                                         │
│ ├── Non-Azure resources                                            │
│ └── Azure App Service (use CNAME for subdomain or TXT verify)     │
│                                                                       │
│ Benefits vs standard records:                                        │
│ ├── Zone apex support (A alias at @)                               │
│ ├── Auto-updates: If Azure resource IP changes, DNS follows       │
│ │   (standard A records would become stale!)                      │
│ ├── Health-aware: If Traffic Manager endpoint is unhealthy,       │
│ │   alias won't return that IP                                     │
│ └── No dangling DNS: If resource is deleted, alias returns empty  │
│                                                                       │
│ Alias record vs standard record:                                     │
│ ┌────────────────────┬─────────────────────┬───────────────────┐   │
│ │ Feature            │ Standard Record     │ Alias Record      │   │
│ │ Zone apex          │ Only A/AAAA         │ A/AAAA at apex ✅ │   │
│ │ Target             │ IP address          │ Azure resource ID │   │
│ │ Auto-update        │ No (manual)         │ Yes (automatic)   │   │
│ │ Dangling check     │ No                  │ Yes               │   │
│ │ TTL                │ You set             │ Inherited from    │   │
│ │                    │                     │ target resource   │   │
│ └────────────────────┴─────────────────────┴───────────────────┘   │
│                                                                       │
│ Create (CLI):                                                        │
│ az network dns record-set a create \                                 │
│   --resource-group rg-networking \                                    │
│   --zone-name techcorp.com \                                          │
│   --name "@" \                                                        │
│   --target-resource /subscriptions/{sub-id}/resourceGroups/\         │
│     rg-networking/providers/Microsoft.Network/publicIPAddresses/\    │
│     pip-prod-lb                                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Private DNS Zones

```
┌─────────────────────────────────────────────────────────────────────┐
│              PRIVATE DNS ZONES                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: DNS zones that resolve ONLY within linked VNets.              │
│                                                                       │
│ Key features:                                                        │
│ ├── Linked to one or more VNets                                    │
│ ├── Auto-registration: VMs auto-register A records! ⚡             │
│ ├── Cross-VNet resolution (link multiple VNets)                    │
│ ├── Cross-subscription (link VNets from other subscriptions)       │
│ └── Used extensively for Private Endpoints                         │
│                                                                       │
│ VNet link types:                                                     │
│                                                                       │
│ 1. RESOLUTION LINK (auto-registration OFF):                         │
│    ├── VNet CAN resolve records in this private zone               │
│    ├── VNet CANNOT auto-register records                           │
│    └── Read-only access to DNS zone                                │
│                                                                       │
│ 2. REGISTRATION LINK (auto-registration ON):                        │
│    ├── VNet CAN resolve records                                    │
│    ├── VMs in VNet AUTO-REGISTER A + PTR records                  │
│    ├── VM name → private IP (A record)                             │
│    ├── Private IP → VM name (PTR record)                           │
│    ├── Only ONE registration link per VNet per zone!               │
│    └── Records auto-removed when VM deleted                        │
│                                                                       │
│ Auto-registration example:                                           │
│ Private zone: internal.techcorp.com                                 │
│ VNet: prod-vnet (linked with auto-registration ON)                 │
│                                                                       │
│ You create VM "web-server-01" in prod-vnet:                        │
│ ├── Auto-created: web-server-01.internal.techcorp.com A 10.0.1.5  │
│ ├── Auto-created: 5.1.0.10.in-addr.arpa PTR web-server-01...     │
│ └── All VMs in prod-vnet can resolve web-server-01 by name!      │
│                                                                       │
│ ⚡ This is UNIQUE to Azure! AWS and GCP don't have this.           │
│    In AWS, you manually create Route 53 private records.           │
│    In GCP, you manually create Cloud DNS private records.          │
│    Azure does it automatically!                                    │
│                                                                       │
│ Limits:                                                              │
│ ├── 1 registration link per VNet per zone                          │
│ ├── 1000 resolution links per zone                                 │
│ ├── 25,000 records per zone                                        │
│ └── 1000 private zones per subscription                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Private DNS Zone

```
Azure Portal → Private DNS zones → + Create

┌─────────────────────────────────────────────────────────────────┐
│           CREATE PRIVATE DNS ZONE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Basics:                                                          │
│   Subscription:     [tc-production ▼]                           │
│   Resource group:   [rg-networking ▼]                           │
│   Name:             [internal.techcorp.com]                    │
│   Resource group location: (global)                            │
│                                                                   │
│ Tags:                                                            │
│   Environment: production                                        │
│                                                                   │
│ [Review + create] → [Create]                                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Then add VNet link:
Private DNS zone → Virtual network links → + Add

┌─────────────────────────────────────────────────────────────────┐
│           ADD VIRTUAL NETWORK LINK                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Link name:           [link-prod-vnet]                           │
│ Subscription:        [tc-production ▼]                          │
│ Virtual network:     [prod-vnet ▼]                              │
│ Enable auto registration: ☑ (for VM auto-registration)        │
│                                                                   │
│ [OK]                                                             │
│                                                                   │
│ After linking:                                                   │
│ ├── All VMs in prod-vnet → auto-registered                     │
│ ├── Any VM in prod-vnet can resolve *.internal.techcorp.com    │
│ └── Link shows "Completed" when provisioned                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create private DNS zone
  az network private-dns zone create \
    --resource-group rg-networking \
    --name internal.techcorp.com

  # Link VNet with auto-registration
  az network private-dns link vnet create \
    --resource-group rg-networking \
    --zone-name internal.techcorp.com \
    --name link-prod-vnet \
    --virtual-network prod-vnet \
    --registration-enabled true

  # Link another VNet (resolution only, no auto-registration)
  az network private-dns link vnet create \
    --resource-group rg-networking \
    --zone-name internal.techcorp.com \
    --name link-dev-vnet \
    --virtual-network dev-vnet \
    --registration-enabled false

  # Add manual record
  az network private-dns record-set a add-record \
    --resource-group rg-networking \
    --zone-name internal.techcorp.com \
    --record-set-name db \
    --ipv4-address 10.0.3.5
```

### Private Endpoint DNS (privatelink zones)

```
┌─────────────────────────────────────────────────────────────────────┐
│          PRIVATE ENDPOINT DNS INTEGRATION                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ When you create a Private Endpoint, you need DNS to resolve        │
│ the public FQDN to the private IP.                                 │
│                                                                       │
│ Example: Azure SQL Database                                         │
│ Public FQDN: tc-prod-sql.database.windows.net                      │
│ Public IP: 40.79.153.12 (internet-accessible)                      │
│ Private Endpoint IP: 10.0.5.4 (in your VNet)                      │
│                                                                       │
│ You need: tc-prod-sql.database.windows.net → 10.0.5.4             │
│           (inside your VNet)                                        │
│                                                                       │
│ How it works (DNS chain):                                            │
│                                                                       │
│ VM queries: tc-prod-sql.database.windows.net                       │
│    │                                                                  │
│    ▼                                                                  │
│ Public DNS returns CNAME:                                            │
│ tc-prod-sql.database.windows.net                                    │
│  → tc-prod-sql.privatelink.database.windows.net                    │
│    │                                                                  │
│    ▼                                                                  │
│ Azure Private DNS zone: privatelink.database.windows.net           │
│ tc-prod-sql → 10.0.5.4                                             │
│    │                                                                  │
│    ▼                                                                  │
│ VM connects to 10.0.5.4 (private!)                                 │
│                                                                       │
│ Common privatelink zones:                                            │
│ ┌──────────────────────────┬──────────────────────────────────────┐│
│ │ Service                  │ Private DNS Zone                     ││
│ ├──────────────────────────┼──────────────────────────────────────┤│
│ │ Azure SQL                │ privatelink.database.windows.net     ││
│ │ Blob Storage             │ privatelink.blob.core.windows.net    ││
│ │ Azure Files              │ privatelink.file.core.windows.net    ││
│ │ Cosmos DB                │ privatelink.documents.azure.com      ││
│ │ Key Vault                │ privatelink.vaultcore.azure.net      ││
│ │ Azure Cache for Redis    │ privatelink.redis.cache.windows.net  ││
│ │ Container Registry       │ privatelink.azurecr.io               ││
│ │ App Service              │ privatelink.azurewebsites.net        ││
│ │ Event Hubs               │ privatelink.servicebus.windows.net   ││
│ │ Azure Monitor            │ privatelink.monitor.azure.com        ││
│ └──────────────────────────┴──────────────────────────────────────┘│
│                                                                       │
│ Setup steps:                                                         │
│ 1. Create private DNS zone (e.g., privatelink.database.windows.net)│
│ 2. Link zone to your VNet                                          │
│ 3. Create Private Endpoint with "Integrate with private DNS zone"  │
│    → Azure auto-creates the A record!                              │
│                                                                       │
│ ⚠️ If using hub-and-spoke, create privatelink zones in hub         │
│    and link all spoke VNets for centralized DNS management.        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Azure DNS Private Resolver

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE DNS PRIVATE RESOLVER                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Managed DNS forwarder for hybrid DNS scenarios.               │
│       Replaces custom DNS VMs/forwarders in Azure.                  │
│                                                                       │
│ Components:                                                          │
│ ├── 1. Inbound Endpoint                                            │
│ │      On-prem DNS → forward to this IP → resolves Azure DNS      │
│ │      Gets a private IP in your VNet subnet                       │
│ │      On-prem can resolve: *.internal.techcorp.com               │
│ │      On-prem can resolve: *.privatelink.database.windows.net    │
│ │                                                                    │
│ ├── 2. Outbound Endpoint                                           │
│ │      Azure DNS → forward specific domains to on-prem DNS       │
│ │      VMs can resolve: *.corp.techcorp.local                     │
│ │                                                                    │
│ └── 3. DNS Forwarding Ruleset                                      │
│        Rules defining which domains go where                       │
│        Attached to outbound endpoint                               │
│        Linked to VNets                                              │
│                                                                       │
│ Architecture:                                                        │
│                                                                       │
│ ┌──────────────────┐              ┌──────────────────────────┐     │
│ │ On-Premises       │              │ Azure VNet                │     │
│ │                    │   VPN/ER     │                            │     │
│ │ AD DNS ◄───────────┤◄─ Outbound ─│ DNS Private Resolver     │     │
│ │ (.corp.local)      │   Endpoint   │   ├── Outbound Endpoint  │     │
│ │                    │              │   │   + Forwarding Rules  │     │
│ │ AD DNS ───────────►│── Inbound ──►│   └── Inbound Endpoint   │     │
│ │ (query .internal   │   Endpoint   │                            │     │
│ │  .techcorp.com)   │              │ ┌─ Private DNS Zones ──┐ │     │
│ │                    │              │ │ internal.techcorp.com │ │     │
│ │                    │              │ │ privatelink.*.net     │ │     │
│ │                    │              │ └──────────────────────┘ │     │
│ └──────────────────┘              └──────────────────────────┘     │
│                                                                       │
│ Pricing:                                                             │
│ ├── Inbound endpoint: $0.18/hour (~$130/month)                    │
│ ├── Outbound endpoint: $0.18/hour (~$130/month)                   │
│ └── Queries: $0.40/million (first billion)                         │
│                                                                       │
│ Comparison:                                                          │
│ ├── AWS Route 53 Resolver: ~$180/month per endpoint               │
│ ├── Azure DNS Private Resolver: ~$130/month per endpoint          │
│ └── GCP DNS Policy inbound: ~$0.40/month (much cheaper!)          │
│                                                                       │
│ Before DNS Private Resolver:                                         │
│ People ran custom DNS forwarder VMs (BIND, Windows DNS)            │
│ in Azure. DNS Private Resolver replaces these with a managed       │
│ service. No VMs to manage!                                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating DNS Private Resolver

```
Azure Portal → DNS Private Resolvers → + Create

┌─────────────────────────────────────────────────────────────────┐
│        CREATE DNS PRIVATE RESOLVER                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Basics:                                                          │
│   Subscription:     [tc-production ▼]                           │
│   Resource group:   [rg-networking ▼]                           │
│   Name:             [resolver-prod]                             │
│   Region:           [Central India ▼]                           │
│   Virtual Network:  [hub-vnet ▼]                               │
│                                                                   │
│ Inbound Endpoints:                                               │
│   Name:   [ie-prod]                                             │
│   Subnet: [snet-dns-inbound ▼]                                 │
│   (/28 minimum, dedicated subnet for resolver!)                │
│   Private IP: (auto-assigned, e.g., 10.0.254.4)               │
│                                                                   │
│ Outbound Endpoints:                                              │
│   Name:   [oe-prod]                                             │
│   Subnet: [snet-dns-outbound ▼]                                │
│   (/28 minimum, different subnet from inbound!)                │
│                                                                   │
│ [Review + create] → [Create]                                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Then create Forwarding Ruleset:
DNS Private Resolver → DNS forwarding rulesets → + Create

┌─────────────────────────────────────────────────────────────────┐
│        CREATE DNS FORWARDING RULESET                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:             [frs-onprem]                                  │
│ Outbound endpoint: [oe-prod ▼]                                 │
│                                                                   │
│ Rules:                                                           │
│   Rule name: forward-corp-local                                 │
│   Domain:    corp.techcorp.local.                               │
│   State:     Enabled                                            │
│   Destination IP: 192.168.1.53:53                              │
│                   192.168.1.54:53                               │
│                                                                   │
│ Virtual Network Links:                                           │
│   [hub-vnet]                                                    │
│   [spoke-prod-vnet]                                             │
│   [spoke-dev-vnet]                                              │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create resolver
  az dns-resolver create \
    --resource-group rg-networking \
    --name resolver-prod \
    --location centralindia \
    --id /subscriptions/{sub}/resourceGroups/rg-networking/\
    providers/Microsoft.Network/virtualNetworks/hub-vnet

  # Create inbound endpoint
  az dns-resolver inbound-endpoint create \
    --resource-group rg-networking \
    --dns-resolver-name resolver-prod \
    --name ie-prod \
    --location centralindia \
    --ip-configurations "[{\"private-ip-allocation-method\":\"Dynamic\",\
    \"id\":\"/subscriptions/{sub}/resourceGroups/rg-networking/\
    providers/Microsoft.Network/virtualNetworks/hub-vnet/subnets/snet-dns-inbound\"}]"
```

---

## Part 5: Azure Traffic Manager (DNS-Based Routing)

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE TRAFFIC MANAGER                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: DNS-based traffic distribution service.                       │
│       Azure's equivalent to AWS Route 53 routing policies.         │
│                                                                       │
│ ⚠️ Traffic Manager is NOT part of Azure DNS!                        │
│    It's a separate service, but works at DNS level.                │
│    Returns DNS responses directing clients to the best endpoint.   │
│                                                                       │
│ Key concept:                                                         │
│ ├── You get a Traffic Manager FQDN:                                │
│ │   techcorp.trafficmanager.net                                    │
│ ├── CNAME your domain to it:                                      │
│ │   api.techcorp.com CNAME techcorp.trafficmanager.net            │
│ ├── Or use Alias record at apex:                                  │
│ │   techcorp.com ALIAS → Traffic Manager profile                  │
│ └── Traffic Manager returns the best endpoint IP                  │
│                                                                       │
│ Routing Methods:                                                     │
│                                                                       │
│ 1. PRIORITY (= AWS Failover)                                        │
│    ├── Route to primary endpoint                                   │
│    ├── If unhealthy → route to secondary                          │
│    ├── Priority 1 = highest (primary)                              │
│    ├── Active-passive DR                                           │
│    └── Like AWS Failover routing                                   │
│                                                                       │
│ 2. WEIGHTED (= AWS Weighted)                                        │
│    ├── Distribute traffic by weight                                │
│    ├── Weight 1-1000 per endpoint                                  │
│    ├── Canary deployments, A/B testing                             │
│    └── Same as AWS Weighted routing                                │
│                                                                       │
│ 3. PERFORMANCE (= AWS Latency-based)                                │
│    ├── Route to lowest-latency endpoint                            │
│    ├── Uses Internet Latency Table                                 │
│    ├── Best for global multi-region apps                           │
│    └── Same as AWS Latency-based routing                           │
│                                                                       │
│ 4. GEOGRAPHIC (= AWS Geolocation)                                    │
│    ├── Route based on user's location                              │
│    ├── Compliance (GDPR), content localization                     │
│    ├── Must have "All (World)" for catch-all                      │
│    └── Same as AWS Geolocation routing                             │
│                                                                       │
│ 5. MULTIVALUE (= AWS Multi-value answer)                            │
│    ├── Return multiple healthy endpoints                           │
│    ├── Client picks one                                            │
│    ├── Only for IPv4/IPv6 endpoints                                │
│    └── Up to 8 endpoints per response                              │
│                                                                       │
│ 6. SUBNET (Azure-unique!)                                            │
│    ├── Route based on client's IP subnet                           │
│    ├── Map specific IP ranges to specific endpoints                │
│    ├── Use case: Internal users → internal endpoint                │
│    │             External users → public endpoint                  │
│    └── No AWS/GCP equivalent at DNS level                          │
│                                                                       │
│ Comparison to AWS:                                                   │
│ ┌──────────────────┬──────────────────┬────────────────────────┐   │
│ │ Azure TM         │ AWS Route 53     │ GCP Equivalent         │   │
│ │ Priority          │ Failover         │ Global LB + failover  │   │
│ │ Weighted          │ Weighted         │ Global LB + weight    │   │
│ │ Performance       │ Latency-based    │ Global LB (default)   │   │
│ │ Geographic        │ Geolocation      │ Global LB + header    │   │
│ │ MultiValue        │ Multi-value      │ N/A                   │   │
│ │ Subnet            │ N/A              │ N/A                   │   │
│ │ N/A               │ Geoproximity     │ N/A                   │   │
│ └──────────────────┴──────────────────┴────────────────────────┘   │
│                                                                       │
│ Nested profiles:                                                     │
│ ├── Combine routing methods by nesting TM profiles                │
│ ├── Example: Performance (global) → Weighted (per region)         │
│ │   Global users → nearest region → canary within that region    │
│ └── Very powerful for complex routing scenarios                    │
│                                                                       │
│ Health probes:                                                       │
│ ├── Protocol: HTTP, HTTPS, TCP                                    │
│ ├── Path: /health                                                 │
│ ├── Interval: 10 or 30 seconds                                   │
│ ├── Timeout: 5 or 10 seconds                                     │
│ ├── Tolerated failures: 3 (configurable)                          │
│ ├── Custom headers supported (Host header, etc.)                  │
│ └── Expected status codes: 200 (or custom range)                  │
│                                                                       │
│ Pricing:                                                             │
│ ├── First billion DNS queries/month: $0.54/million                │
│ ├── Health checks: Included!                                      │
│ ├── Real User Measurements: $2/month per million measurements    │
│ └── No base fee for the profile                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Traffic Manager Profile

```
Azure Portal → Traffic Manager profiles → + Create

┌─────────────────────────────────────────────────────────────────┐
│        CREATE TRAFFIC MANAGER PROFILE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:             [tc-prod] (.trafficmanager.net)               │
│ Routing method:   [Performance ▼]                               │
│ Subscription:     [tc-production ▼]                             │
│ Resource group:   [rg-networking ▼]                             │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
│ Then add endpoints:                                              │
│ Traffic Manager → Endpoints → + Add                             │
│                                                                   │
│   Type:         [Azure endpoint ▼]                              │
│                 [External endpoint ▼]                            │
│                 [Nested endpoint ▼]                              │
│                                                                   │
│   Name:         [ep-india]                                      │
│   Target type:  [Public IP address ▼]                           │
│   Target:       [pip-prod-lb-india ▼]                           │
│   Status:       ● Enabled                                       │
│                                                                   │
│   (For Weighted):                                                │
│   Weight:       [100]                                           │
│                                                                   │
│   (For Priority):                                                │
│   Priority:     [1]                                             │
│                                                                   │
│   [Add]                                                          │
│                                                                   │
│ Configure health probe:                                          │
│ Traffic Manager → Configuration                                 │
│   Protocol:        [HTTPS ▼]                                   │
│   Port:            [443]                                        │
│   Path:            [/health]                                    │
│   Probing interval: [10 seconds]                               │
│   Tolerated failures: [3]                                      │
│   Probe timeout:   [5 seconds]                                 │
│   DNS TTL:         [60 seconds]                                │
│                                                                   │
│ Point DNS:                                                       │
│ api.techcorp.com CNAME tc-prod.trafficmanager.net              │
│ (or Alias record at zone apex)                                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Create profile
  az network traffic-manager profile create \
    --resource-group rg-networking \
    --name tc-prod \
    --routing-method Performance \
    --unique-dns-name tc-prod \
    --monitor-protocol HTTPS \
    --monitor-port 443 \
    --monitor-path /health \
    --ttl 60

  # Add endpoint
  az network traffic-manager endpoint create \
    --resource-group rg-networking \
    --profile-name tc-prod \
    --name ep-india \
    --type azureEndpoints \
    --target-resource-id /subscriptions/{sub}/resourceGroups/\
    rg-networking/providers/Microsoft.Network/publicIPAddresses/pip-prod-lb-india

  # CNAME in DNS
  az network dns record-set cname set-record \
    --resource-group rg-networking \
    --zone-name techcorp.com \
    --record-set-name api \
    --cname tc-prod.trafficmanager.net
```

---

## Part 6: DNSSEC

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DNSSEC IN AZURE DNS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure DNS supports DNSSEC for public zones (GA since 2024).        │
│                                                                       │
│ Enable:                                                              │
│ Azure Portal → DNS zone → DNSSEC → Enable                          │
│                                                                       │
│ az network dns dnssec-config create \                                │
│   --resource-group rg-networking \                                    │
│   --zone-name techcorp.com                                            │
│                                                                       │
│ After enabling:                                                      │
│ 1. Azure generates DNSKEY records                                   │
│ 2. Get DS record information                                        │
│ 3. Add DS record at your domain registrar                          │
│ 4. Chain of trust is established                                   │
│                                                                       │
│ ⚠️ Same caution as AWS/GCP: misconfigured DNSSEC breaks DNS.      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Bicep Example

```bicep
// Public DNS zone
resource dnsZone 'Microsoft.Network/dnsZones@2023-07-01-preview' = {
  name: 'techcorp.com'
  location: 'global'
  tags: {
    Environment: 'production'
  }
}

// A record (standard)
resource apiRecord 'Microsoft.Network/dnsZones/A@2023-07-01-preview' = {
  parent: dnsZone
  name: 'api'
  properties: {
    TTL: 300
    ARecords: [
      { ipv4Address: '20.100.50.60' }
    ]
  }
}

// A record (alias to Public IP)
resource apexAlias 'Microsoft.Network/dnsZones/A@2023-07-01-preview' = {
  parent: dnsZone
  name: '@'
  properties: {
    TTL: 300
    targetResource: {
      id: publicIp.id
    }
  }
}

// CNAME record
resource blogRecord 'Microsoft.Network/dnsZones/CNAME@2023-07-01-preview' = {
  parent: dnsZone
  name: 'blog'
  properties: {
    TTL: 3600
    CNAMERecord: {
      cname: 'techcorp.wordpress.com'
    }
  }
}

// MX record
resource mxRecord 'Microsoft.Network/dnsZones/MX@2023-07-01-preview' = {
  parent: dnsZone
  name: '@'
  properties: {
    TTL: 3600
    MXRecords: [
      { preference: 10, exchange: 'mail.techcorp.com' }
      { preference: 20, exchange: 'mail2.techcorp.com' }
    ]
  }
}

// TXT record (SPF)
resource txtRecord 'Microsoft.Network/dnsZones/TXT@2023-07-01-preview' = {
  parent: dnsZone
  name: '@'
  properties: {
    TTL: 3600
    TXTRecords: [
      { value: ['v=spf1 include:spf.protection.outlook.com ~all'] }
    ]
  }
}

// Private DNS zone
resource privateDnsZone 'Microsoft.Network/privateDnsZones@2024-06-01' = {
  name: 'internal.techcorp.com'
  location: 'global'
}

// VNet link with auto-registration
resource vnetLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2024-06-01' = {
  parent: privateDnsZone
  name: 'link-prod-vnet'
  location: 'global'
  properties: {
    virtualNetwork: {
      id: prodVnet.id
    }
    registrationEnabled: true
  }
}

// Private DNS record
resource dbRecord 'Microsoft.Network/privateDnsZones/A@2024-06-01' = {
  parent: privateDnsZone
  name: 'db'
  properties: {
    ttl: 60
    aRecords: [
      { ipv4Address: '10.0.3.5' }
    ]
  }
}

// Private DNS zone for Private Endpoints (SQL)
resource sqlPrivateLink 'Microsoft.Network/privateDnsZones@2024-06-01' = {
  name: 'privatelink.database.windows.net'
  location: 'global'
}

resource sqlVnetLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2024-06-01' = {
  parent: sqlPrivateLink
  name: 'link-hub-vnet'
  location: 'global'
  properties: {
    virtualNetwork: {
      id: hubVnet.id
    }
    registrationEnabled: false
  }
}

// Traffic Manager profile
resource trafficManager 'Microsoft.Network/trafficmanagerprofiles@2022-04-01' = {
  name: 'tc-prod'
  location: 'global'
  properties: {
    profileStatus: 'Enabled'
    trafficRoutingMethod: 'Performance'
    dnsConfig: {
      relativeName: 'tc-prod'
      ttl: 60
    }
    monitorConfig: {
      protocol: 'HTTPS'
      port: 443
      path: '/health'
      intervalInSeconds: 10
      toleratedNumberOfFailures: 3
      timeoutInSeconds: 5
    }
    endpoints: [
      {
        name: 'ep-india'
        type: 'Microsoft.Network/trafficManagerProfiles/azureEndpoints'
        properties: {
          targetResourceId: publicIpIndia.id
          endpointStatus: 'Enabled'
        }
      }
      {
        name: 'ep-europe'
        type: 'Microsoft.Network/trafficManagerProfiles/azureEndpoints'
        properties: {
          targetResourceId: publicIpEurope.id
          endpointStatus: 'Enabled'
        }
      }
    ]
  }
}
```

---

## Part 8: Real-World Patterns

### Startup

```
DNS zones: 1 public (techcorp.com)
Records:
├── techcorp.com A → Front Door/App Gateway public IP (alias)
├── api.techcorp.com A → App Gateway
├── www.techcorp.com CNAME → techcorp.com
├── MX → Microsoft 365
├── TXT → SPF, DKIM, domain verification
└── blog.techcorp.com CNAME → blog platform

Private DNS: None (or 1 zone for Private Endpoints)
Traffic Manager: Not needed (single region)
Cost: ~$2/month
```

### Mid-Size

```
DNS zones:
├── techcorp.com (public)
├── internal.techcorp.com (private, auto-registration ON)
├── privatelink.database.windows.net (for SQL Private Endpoints)
├── privatelink.blob.core.windows.net (for Storage)
└── privatelink.vaultcore.azure.net (for Key Vault)

Private DNS:
├── internal.techcorp.com linked to prod-vnet + dev-vnet
├── Auto-registration: All VMs auto-registered
├── Manual records: db, cache, queue
└── All privatelink zones linked to all VNets

Traffic Manager: Not needed (single region)
DNS Private Resolver: Not needed (no on-prem)
Cost: ~$5-10/month
```

### Enterprise

```
DNS zones:
├── techcorp.com (public, DNSSEC ON)
├── internal.techcorp.com (private, hub-vnet)
├── staging.internal.techcorp.com (private, staging-vnet)
├── dev.internal.techcorp.com (private, dev-vnet)
├── 10+ privatelink zones (SQL, Storage, KV, ACR, etc.)
└── Per-BU zones (finance.internal, hr.internal)

VNet links:
├── All privatelink zones → hub-vnet (centralized)
├── All spoke VNets → hub VNet link for resolution
├── Auto-registration in each environment zone
└── Cross-subscription links for shared services

Traffic Manager:
├── Performance profile: Global users → nearest region
├── Priority profile: DR (primary India → secondary Europe)
├── Nested: Performance → Weighted (canary per region)
└── Health probes on all endpoints (HTTPS /health)

DNS Private Resolver (in hub-vnet):
├── Inbound endpoint: On-prem → resolve Azure private zones
├── Outbound endpoint: Azure → resolve corp.techcorp.local
├── Forwarding rules: corp.local → on-prem AD DNS
└── All spoke VNets linked to forwarding ruleset

Cost: ~$200-400/month (resolver is the big cost)
```

---

## Quick Reference

| Feature | Details |
|---------|---------|
| Public zone cost | $0.50/month (first 25 zones) |
| Private zone cost | $0.25/month |
| Query cost | $0.40/million |
| Alias records | A, AAAA, CNAME → Azure resources |
| Auto-registration | Yes! (unique to Azure) |
| DNS routing | Azure Traffic Manager (separate service) |
| TM routing methods | Priority, Weighted, Performance, Geographic, MultiValue, Subnet |
| Health probes | Via Traffic Manager |
| DNSSEC | Yes (public zones) |
| Private Resolver | ~$130/month per endpoint |
| Privatelink zones | One per Azure service type |
| SLA | 100% |

---

## What's Next?

In the next chapter, we'll cover Azure CDN & Front Door — Azure's content delivery and global application acceleration services.

→ Next: [Chapter 10: Azure CDN & Front Door](10-cdn-front-door.md)

---

*Last Updated: May 2026*
