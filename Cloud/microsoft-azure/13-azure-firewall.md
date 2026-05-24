# Chapter 13: Azure Firewall

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Firewall Fundamentals](#part-1-azure-firewall-fundamentals)
- [Part 2: Creating Azure Firewall (Portal Walkthrough)](#part-2-creating-azure-firewall-portal-walkthrough)
- [Part 3: Firewall Policy](#part-3-firewall-policy)
- [Part 4: Rule Types (Full Walkthrough)](#part-4-rule-types-full-walkthrough)
- [Part 5: Threat Intelligence](#part-5-threat-intelligence)
- [Part 6: Premium Features](#part-6-premium-features)
- [Part 7: DNS Proxy](#part-7-dns-proxy)
- [Part 8: Forced Tunneling](#part-8-forced-tunneling)
- [Part 9: Route Table (UDR) Setup](#part-9-route-table-udr-setup)
- [Part 10: Azure Firewall Manager](#part-10-azure-firewall-manager)
- [Part 11: Monitoring & Logging](#part-11-monitoring--logging)
- [Part 12: Comparison with AWS & GCP](#part-12-comparison-with-aws--gcp)
- [Part 13: Terraform & Bicep Examples](#part-13-terraform--bicep-examples)
- [Part 14: az CLI Reference](#part-14-az-cli-reference)
- [Part 15: Real-World Patterns](#part-15-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Firewall is a cloud-native, fully managed network security service that protects your Azure Virtual Network resources. It's a stateful firewall-as-a-service with built-in high availability and unrestricted cloud scalability. Unlike NSGs (Layer 4 per-subnet/NIC), Azure Firewall provides centralized network and application-level (Layer 4 + Layer 7) filtering across VNets and subscriptions.

```
What you'll learn:
├── Azure Firewall Fundamentals
│   ├── What it does vs NSG vs third-party NVA
│   ├── SKUs (Basic, Standard, Premium)
│   └── Where to place it (hub-spoke topology)
├── Firewall Policy
│   ├── Policy hierarchy (parent → child)
│   ├── Rule Collection Groups
│   ├── Rule Collections
│   └── Rules (NAT, Network, Application)
├── Rule Types (Full Walkthrough)
│   ├── DNAT Rules (inbound port forwarding)
│   ├── Network Rules (Layer 4: IP/port/protocol)
│   ├── Application Rules (Layer 7: FQDN/URL filtering)
│   └── Priority & processing order
├── Threat Intelligence
├── IDPS (Intrusion Detection & Prevention) — Premium
├── TLS Inspection — Premium
├── DNS Proxy
├── Forced Tunneling
├── Azure Firewall Manager
├── Hub-Spoke Topology with Firewall
├── Comparison with AWS & GCP
├── az CLI, Terraform, Bicep examples
└── Real-world patterns
```

---

## Part 1: Azure Firewall Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE FIREWALL vs NSG vs NVA                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌────────────────┬──────────────┬──────────────┬──────────────────┐│
│ │ Feature        │ NSG          │ Azure FW     │ Third-party NVA ││
│ ├────────────────┼──────────────┼──────────────┼──────────────────┤│
│ │ Layer          │ L3/L4        │ L3/L4/L7     │ L3/L4/L7        ││
│ │ Scope          │ Subnet/NIC   │ Centralized  │ Centralized     ││
│ │                │              │ (hub VNet)   │ (hub VNet)      ││
│ │ FQDN filter    │ No           │ Yes ✅       │ Yes              ││
│ │ TLS inspect    │ No           │ Premium ✅   │ Yes              ││
│ │ IDPS           │ No           │ Premium ✅   │ Yes              ││
│ │ Threat intel   │ No           │ Yes ✅       │ Yes              ││
│ │ Managed        │ Yes (basic)  │ Yes (full)   │ No (you manage) ││
│ │ HA/scaling     │ Built-in     │ Built-in     │ You configure   ││
│ │ Cost           │ Free         │ ~$912/month  │ License + VM    ││
│ │ Best for       │ Micro-rules  │ Centralized  │ Advanced/       ││
│ │                │ per subnet   │ egress/ingress│ compliance     ││
│ └────────────────┴──────────────┴──────────────┴──────────────────┘│
│                                                                       │
│ Use BOTH: NSG for subnet-level micro-segmentation                 │
│           + Azure Firewall for centralized traffic inspection     │
│                                                                       │
│ SKUs:                                                                │
│ ┌──────────┬──────────────────────────────────────────────────────┐│
│ │ SKU      │ Features                                             ││
│ ├──────────┼──────────────────────────────────────────────────────┤│
│ │ Basic    │ SMB/small workloads, 250 Mbps throughput,           ││
│ │ ~$274/mo │ NAT + Network + App rules, Threat Intel (alert),   ││
│ │          │ ⚠️ No IDPS, no TLS inspection, limited scale        ││
│ │          │                                                      ││
│ │ Standard │ Most workloads, multi-Gbps throughput,               ││
│ │ ~$912/mo │ NAT + Network + App rules, Threat Intel             ││
│ │          │ (alert + deny), FQDN filtering, FQDN tags,         ││
│ │          │ Web categories, DNS proxy                            ││
│ │          │ ⚡ Best value for most companies                      ││
│ │          │                                                      ││
│ │ Premium  │ Everything in Standard PLUS:                         ││
│ │~$1,825/mo│ TLS inspection (decrypt/inspect HTTPS),             ││
│ │          │ IDPS (signature-based intrusion detection),          ││
│ │          │ URL filtering (path-level, not just FQDN),          ││
│ │          │ Web categories (granular)                            ││
│ │          │ ⚡ For regulated industries, PCI-DSS, HIPAA          ││
│ └──────────┴──────────────────────────────────────────────────────┘│
│                                                                       │
│ ⚡ Firewall placement — ALWAYS in a hub VNet:                        │
│                                                                       │
│             ┌────────────────────────┐                              │
│             │     Hub VNet           │                              │
│             │ ┌──────────────────┐   │                              │
│             │ │ AzureFirewall    │   │                              │
│             │ │ Subnet           │   │                              │
│             │ │ (dedicated /26)  │   │                              │
│             │ └────────┬─────────┘   │                              │
│             │          │             │                              │
│       ┌─────┤     VNet peering      ├─────┐                       │
│       │     │          │             │     │                       │
│       ▼     └──────────┼─────────────┘     ▼                       │
│ ┌──────────┐           │            ┌──────────┐                   │
│ │ Spoke 1  │           │            │ Spoke 2  │                   │
│ │ (Web)    │           │            │ (API)    │                   │
│ └──────────┘           │            └──────────┘                   │
│                   ┌────▼─────┐                                     │
│                   │ Internet │                                      │
│                   └──────────┘                                      │
│                                                                       │
│ AzureFirewallSubnet: Dedicated subnet, MUST be /26 minimum       │
│ (e.g., 10.0.0.0/26) — Azure creates it automatically if you     │
│ name it exactly "AzureFirewallSubnet".                             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating Azure Firewall (Portal Walkthrough)

```
Console → Search "Firewalls" → Create

┌─────────────────────────────────────────────────────────────────┐
│           CREATE A FIREWALL                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Basics ──                                                    │
│                                                                   │
│ Subscription: [Pay-As-You-Go ▼]                                │
│ Resource group: [rg-network-prod ▼]                            │
│ Name: [fw-hub-prod]                                             │
│ Region: [Central India ▼]                                      │
│                                                                   │
│ Availability zone:                                              │
│   ○ None                                                       │
│   ● Zones 1, 2, 3 (recommended — spans all zones for HA)     │
│   ⚡ Always select all zones for production!                    │
│                                                                   │
│ Firewall tier:                                                  │
│   ○ Basic ($274/mo)                                            │
│   ● Standard ($912/mo)                                         │
│   ○ Premium ($1825/mo)                                         │
│                                                                   │
│ Firewall management:                                            │
│   ● Use a Firewall Policy to manage this firewall              │
│   ○ Use classic Firewall rules (legacy — don't use)           │
│                                                                   │
│ Firewall policy:                                                │
│   [Create new ▼]                                               │
│   Name: [policy-fw-prod]                                        │
│   Region: [Central India]                                       │
│   Policy tier: [Standard ▼]                                    │
│   ⚡ Policy tier MUST match or exceed Firewall tier              │
│                                                                   │
│ Virtual network:                                                │
│   ● Use existing: [vnet-hub-prod ▼]                           │
│   ○ Create new                                                 │
│   ⚠️ VNet MUST have a subnet named "AzureFirewallSubnet"       │
│      with at least /26 prefix (64 IPs)                         │
│      e.g., 10.0.0.0/26                                         │
│                                                                   │
│ Public IP address:                                              │
│   [Create new ▼]                                               │
│   Name: [pip-fw-hub-prod]                                       │
│   ⚡ This IP is used for DNAT (inbound) and SNAT (outbound)    │
│   You can add up to 250 public IPs for SNAT port exhaustion   │
│                                                                   │
│ Forced tunneling:                                               │
│   ☐ Enable                                                     │
│   (Routes ALL traffic including internet through on-prem       │
│   firewall — requires AzureFirewallManagementSubnet)           │
│                                                                   │
│ ── Tags ──                                                      │
│ environment: prod                                               │
│ managed-by: network-team                                       │
│ cost-center: security                                           │
│                                                                   │
│ [Review + create] → [Create]                                   │
│                                                                   │
│ ⚠️ Deployment takes 5-10 minutes!                                │
│                                                                   │
│ After creation:                                                  │
│ ├── Firewall gets private IP (e.g., 10.0.0.4) in hub VNet    │
│ ├── Firewall gets public IP (e.g., 20.204.xx.xx)             │
│ ├── Create UDR (Route Table) to force spoke traffic through   │
│ │   firewall (next-hop = 10.0.0.4)                            │
│ └── Associate UDR with spoke subnets                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Firewall Policy

```
┌─────────────────────────────────────────────────────────────────────┐
│           FIREWALL POLICY                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Firewall Policy is the MODERN way to manage Azure Firewall rules. │
│ One policy can be shared across multiple firewalls.                │
│                                                                       │
│ Hierarchy:                                                           │
│                                                                       │
│ Firewall Policy                                                      │
│ ├── Rule Collection Group (priority: 100-65000)                  │
│ │   ├── Rule Collection (action: Allow/Deny, priority)          │
│ │   │   ├── Rule 1 (source, dest, protocol, port)             │
│ │   │   ├── Rule 2                                              │
│ │   │   └── Rule 3                                              │
│ │   ├── Rule Collection 2                                        │
│ │   └── Rule Collection 3                                        │
│ ├── Rule Collection Group 2                                       │
│ └── Rule Collection Group 3                                       │
│                                                                       │
│ Example structure:                                                   │
│ policy-fw-prod                                                       │
│ ├── rcg-dnat (priority: 100) — inbound rules                    │
│ │   └── rc-dnat-web (action: DNAT)                               │
│ │       ├── Forward 443 → web server                            │
│ │       └── Forward 8080 → API server                           │
│ ├── rcg-network (priority: 200) — internal traffic              │
│ │   ├── rc-allow-internal (action: Allow)                       │
│ │   │   ├── Spoke1 → Spoke2 on port 443                       │
│ │   │   └── All → DNS on port 53                               │
│ │   └── rc-deny-internal (action: Deny)                         │
│ │       └── Spoke2 → Spoke1 on all ports                       │
│ └── rcg-application (priority: 300) — outbound internet         │
│     ├── rc-allow-web (action: Allow)                             │
│     │   ├── All → *.microsoft.com                               │
│     │   ├── All → *.ubuntu.com                                   │
│     │   └── All → github.com                                    │
│     └── rc-deny-all (action: Deny)                               │
│         └── All → * (deny everything else)                      │
│                                                                       │
│ Processing order (CRITICAL):                                         │
│ 1. DNAT rules (lowest priority number first)                      │
│ 2. Network rules (lowest priority number first)                   │
│ 3. Application rules (lowest priority number first)               │
│ ⚠️ Lower priority NUMBER = processed FIRST (100 before 200)       │
│ ⚠️ If Network rule matches, Application rules are NOT evaluated   │
│                                                                       │
│ Parent-Child policy (inheritance):                                   │
│ ┌─────────────────────┐                                            │
│ │ Parent policy       │ ← Base rules (org-wide)                  │
│ │ (policy-base-org)   │   e.g., Block malware FQDNs              │
│ └─────────┬───────────┘                                            │
│           │ inherits                                                │
│ ┌─────────▼───────────┐                                            │
│ │ Child policy        │ ← Environment-specific rules             │
│ │ (policy-fw-prod)    │   e.g., Allow prod subnets               │
│ └─────────────────────┘                                            │
│                                                                       │
│ Parent rules are processed FIRST (lower priority).                │
│ Child CANNOT override parent deny rules.                          │
│ ⚡ Great for org-wide security baselines + team customization.     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Rule Types (Full Walkthrough)

### DNAT Rules (Inbound Port Forwarding)

```
┌─────────────────────────────────────────────────────────────────────┐
│           DNAT RULES                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Translate inbound traffic from firewall public IP to        │
│ internal private IP. Used for exposing internal services.          │
│                                                                       │
│ Internet user → Firewall public IP:443 → Web server 10.1.1.4:443│
│                                                                       │
│ Firewall → Firewall policy → Rule Collection Groups              │
│ → Add a rule collection group                                      │
│                                                                       │
│ Rule collection group:                                               │
│   Name: [rcg-dnat-inbound]                                         │
│   Priority: [100]                                                   │
│                                                                       │
│ Add rule collection:                                                 │
│   Name: [rc-dnat-web]                                               │
│   Rule collection type: [DNAT ▼]                                  │
│   Priority: [100]                                                   │
│                                                                       │
│ Rules:                                                               │
│ ┌──────────┬──────────┬──────────┬───────┬──────────┬───────────┐ │
│ │ Name     │ Source   │ Dest     │ Proto │ Dest Port│ Translated│ │
│ │          │          │ (FW IP)  │       │          │ addr:port │ │
│ ├──────────┼──────────┼──────────┼───────┼──────────┼───────────┤ │
│ │ web-https│ *        │ 20.204.  │ TCP   │ 443      │ 10.1.1.4 │ │
│ │          │          │ xx.xx    │       │          │ :443      │ │
│ │          │          │          │       │          │           │ │
│ │ api-8080 │ 203.0.   │ 20.204.  │ TCP   │ 8080     │ 10.2.1.10│ │
│ │          │ 113.0/24 │ xx.xx    │       │          │ :8080     │ │
│ │          │          │          │       │          │           │ │
│ │ ssh-mgmt │ My IP    │ 20.204.  │ TCP   │ 2222     │ 10.0.1.5 │ │
│ │          │          │ xx.xx    │       │          │ :22       │ │
│ └──────────┴──────────┴──────────┴───────┴──────────┴───────────┘ │
│                                                                       │
│ Fields explained:                                                    │
│ ├── Source: Who can access (* = anyone, or IP/CIDR/IP Group)     │
│ ├── Destination: Firewall's public IP(s)                          │
│ ├── Protocol: TCP, UDP, or Any                                    │
│ ├── Destination port: Port on the public IP                       │
│ ├── Translated address: Internal server private IP               │
│ ├── Translated port: Port on the internal server                 │
│ └── ⚠️ DNAT automatically adds a matching Network rule           │
│     (you don't need a separate Allow Network rule)              │
│                                                                       │
│ ⚡ Better alternative: Use Application Gateway or Azure Front Door │
│    for HTTP/HTTPS instead of DNAT (they provide WAF, SSL offload)│
│    Use DNAT for non-HTTP protocols (SSH, custom TCP, RDP)        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Network Rules (Layer 4)

```
┌─────────────────────────────────────────────────────────────────────┐
│           NETWORK RULES                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Layer 4 filtering based on source/dest IP, port, protocol.  │
│ Like NSG rules but centralized at the firewall.                    │
│                                                                       │
│ Add rule collection group:                                           │
│   Name: [rcg-network]                                               │
│   Priority: [200]                                                   │
│                                                                       │
│ Add rule collection:                                                 │
│   Name: [rc-allow-internal]                                         │
│   Rule collection type: [Network ▼]                                │
│   Priority: [100]                                                   │
│   Action: [Allow ▼]                                                │
│                                                                       │
│ Rules:                                                               │
│ ┌───────────────┬─────────────┬──────────────┬───────┬──────────┐ │
│ │ Name          │ Source      │ Destination  │ Proto │ Dest Port│ │
│ ├───────────────┼─────────────┼──────────────┼───────┼──────────┤ │
│ │ spoke1-to-    │ 10.1.0.0/16│ 10.2.0.0/16 │ TCP   │ 443,8080│ │
│ │ spoke2        │ (spoke 1)  │ (spoke 2)    │       │          │ │
│ │               │             │              │       │          │ │
│ │ all-to-dns    │ 10.0.0.0/8 │ 168.63.129.16│ TCP,  │ 53       │ │
│ │               │             │ (Azure DNS)  │ UDP   │          │ │
│ │               │             │              │       │          │ │
│ │ all-to-kms    │ 10.0.0.0/8 │ kms.core.    │ TCP   │ 1688     │ │
│ │               │             │ windows.net  │       │          │ │
│ │               │             │ (FQDN ok in  │       │          │ │
│ │               │             │ network rule)│       │          │ │
│ │               │             │              │       │          │ │
│ │ vpn-to-onprem │ 10.0.0.0/8 │ 192.168.0.   │ Any   │ *        │ │
│ │               │             │ 0/16         │       │          │ │
│ │               │             │ (on-premises)│       │          │ │
│ └───────────────┴─────────────┴──────────────┴───────┴──────────┘ │
│                                                                       │
│ Source types:                                                        │
│ ├── IP Address: CIDR (10.1.0.0/16)                                │
│ ├── IP Group: Reusable group of IPs (reference by name)          │
│ ├── Service Tag: AzureCloud, Internet, VirtualNetwork, etc.      │
│ └── ⚡ Service Tags are powerful — no need to track Azure IPs!     │
│                                                                       │
│ Destination types:                                                   │
│ ├── IP Address: CIDR                                               │
│ ├── IP Group                                                       │
│ ├── Service Tag: AzureCloud, Sql, Storage, AzureMonitor, etc.   │
│ └── FQDN: *.database.windows.net (resolved at rule eval time)   │
│                                                                       │
│ Deny collection (catch-all):                                        │
│   Name: [rc-deny-all-network]                                      │
│   Action: [Deny ▼]                                                 │
│   Priority: [1000] (processed after allow rules)                  │
│   Rule: * → * → Any → * (deny everything else)                  │
│                                                                       │
│ ⚠️ Azure Firewall denies by default. You only need explicit deny   │
│    rules for logging purposes (to see what's being blocked).      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Application Rules (Layer 7)

```
┌─────────────────────────────────────────────────────────────────────┐
│           APPLICATION RULES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Layer 7 filtering. Filter by FQDN, URL, HTTP/HTTPS headers.│
│ This is the MOST USED rule type — controls outbound internet.    │
│                                                                       │
│ Add rule collection group:                                           │
│   Name: [rcg-application]                                           │
│   Priority: [300]                                                   │
│                                                                       │
│ Add rule collection:                                                 │
│   Name: [rc-allow-internet]                                         │
│   Rule collection type: [Application ▼]                            │
│   Priority: [100]                                                   │
│   Action: [Allow ▼]                                                │
│                                                                       │
│ Rules:                                                               │
│ ┌───────────────┬──────────────┬──────────────────┬───────┬──────┐│
│ │ Name          │ Source       │ Target FQDNs     │ Proto │ Port ││
│ ├───────────────┼──────────────┼──────────────────┼───────┼──────┤│
│ │ allow-updates │ 10.0.0.0/8  │ *.ubuntu.com     │ HTTPS │ 443  ││
│ │               │              │ *.debian.org     │       │      ││
│ │               │              │ *.microsoft.com  │       │      ││
│ │               │              │                  │       │      ││
│ │ allow-github  │ 10.1.0.0/16 │ github.com       │ HTTPS │ 443  ││
│ │               │ (dev spoke)  │ *.github.com     │       │      ││
│ │               │              │ *.githubusercontent│     │      ││
│ │               │              │ .com              │       │      ││
│ │               │              │                  │       │      ││
│ │ allow-npm     │ 10.1.0.0/16 │ registry.npmjs.  │ HTTPS │ 443  ││
│ │               │              │ org              │       │      ││
│ │               │              │ *.npmjs.com      │       │      ││
│ │               │              │                  │       │      ││
│ │ allow-docker  │ 10.1.0.0/16 │ *.docker.io      │ HTTPS │ 443  ││
│ │               │              │ *.docker.com     │       │      ││
│ │               │              │ production.      │       │      ││
│ │               │              │ cloudflare.      │       │      ││
│ │               │              │ docker.com       │       │      ││
│ │               │              │                  │       │      ││
│ │ allow-azure   │ 10.0.0.0/8  │ FQDN Tag:        │ HTTPS │ 443  ││
│ │ -services     │              │ AzureKubernetes  │       │      ││
│ │               │              │ Service          │       │      ││
│ │               │              │ WindowsUpdate    │       │      ││
│ │               │              │ AzureBackup      │       │      ││
│ └───────────────┴──────────────┴──────────────────┴───────┴──────┘│
│                                                                       │
│ Target types:                                                        │
│ ├── FQDN: Specific domain (github.com)                            │
│ ├── FQDN wildcard: *.github.com (any subdomain)                  │
│ ├── FQDN Tag: Pre-defined FQDN group by Microsoft                │
│ │   ├── WindowsUpdate: All Windows Update URLs                   │
│ │   ├── AzureBackup: All Azure Backup endpoints                  │
│ │   ├── AzureKubernetesService: AKS required URLs                │
│ │   ├── HDInsight, AppServiceEnvironment, etc.                   │
│ │   └── ⚡ FQDN Tags auto-update when Microsoft adds/changes URLs│
│ └── Web categories (Standard/Premium):                            │
│     ├── Social Networking                                          │
│     ├── Gambling                                                   │
│     ├── Adult Content                                              │
│     ├── Streaming Media                                            │
│     ├── Peer-to-Peer                                               │
│     └── ⚡ Block by category instead of individual FQDNs          │
│                                                                       │
│ Protocol:Port options:                                               │
│ ├── Http:80                                                        │
│ ├── Https:443                                                      │
│ ├── Mssql:1433 (Azure SQL)                                       │
│ └── Custom protocol:port                                           │
│                                                                       │
│ Deny collection (explicit deny for logging):                        │
│   Name: [rc-deny-all-internet]                                     │
│   Action: [Deny ▼]                                                 │
│   Priority: [1000]                                                  │
│   Rule: * → * (FQDN: *) → HTTPS:443, HTTP:80                    │
│   ⚡ This logs all denied outbound attempts!                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Threat Intelligence

```
┌─────────────────────────────────────────────────────────────────────┐
│           THREAT INTELLIGENCE                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Microsoft-curated feed of known malicious IPs, domains,     │
│ and URLs. Azure Firewall checks all traffic against this feed.    │
│                                                                       │
│ Modes:                                                               │
│ ├── Off: No threat intelligence checking                          │
│ ├── Alert only: Log but allow traffic (monitoring phase)         │
│ └── Alert and deny: Block + log (RECOMMENDED for production)    │
│                                                                       │
│ Configure:                                                           │
│ Firewall policy → Threat intelligence →                           │
│   Mode: ● Alert and deny                                          │
│   Allowlist IPs: [add IPs to exclude from checking]              │
│   Allowlist FQDNs: [add FQDNs to exclude]                        │
│                                                                       │
│ What it blocks:                                                      │
│ ├── Known malware command-and-control (C2) servers               │
│ ├── Known phishing domains                                        │
│ ├── Known botnet IPs                                               │
│ ├── Crypto mining pools                                            │
│ └── Microsoft Threat Intelligence feed (updated continuously)    │
│                                                                       │
│ ⚡ Enable "Alert and deny" on ALL production firewalls.             │
│    No extra cost — included in Standard and Premium.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Premium Features

### IDPS (Intrusion Detection & Prevention System)

```
┌─────────────────────────────────────────────────────────────────────┐
│           IDPS (Premium Only)                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Signature-based inspection of network traffic to detect     │
│ exploits, vulnerabilities, and malicious activity.                │
│                                                                       │
│ Modes:                                                               │
│ ├── Off: No IDPS                                                  │
│ ├── Alert only: Detect + log (no blocking)                       │
│ └── Alert and deny: Detect + block + log                         │
│                                                                       │
│ Configure:                                                           │
│ Firewall policy → IDPS →                                          │
│   Mode: ● Alert and deny                                          │
│   Signature rules: 67,000+ signatures                             │
│   Categories:                                                      │
│   ├── Malware                                                      │
│   ├── Phishing                                                     │
│   ├── Trojan                                                       │
│   ├── DOS/DDoS                                                     │
│   ├── Exploit kit                                                  │
│   ├── Web attacks (SQLi, XSS, SSRF)                              │
│   ├── Protocol anomalies                                          │
│   └── And many more...                                             │
│                                                                       │
│   Custom rules: Override specific signatures                      │
│   ├── Set to Alert, Deny, or Off per signature                   │
│   ├── Bypass list (for false positives)                          │
│   └── Private IP ranges for internal traffic inspection          │
│                                                                       │
│ ⚠️ IDPS on inbound: Inspects traffic AFTER DNAT rule matches      │
│ ⚠️ IDPS on outbound: Inspects all outbound traffic                 │
│ ⚠️ East-West: Inspects spoke-to-spoke traffic through firewall   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### TLS Inspection

```
┌─────────────────────────────────────────────────────────────────────┐
│           TLS INSPECTION (Premium Only)                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Decrypt HTTPS traffic → inspect content → re-encrypt →    │
│ forward. Without this, firewall can only see the FQDN (SNI),    │
│ NOT the URL path or payload.                                      │
│                                                                       │
│ Without TLS inspection:                                              │
│ ├── Can filter: *.github.com ✅                                   │
│ └── Cannot see: github.com/malicious/payload ❌                  │
│                                                                       │
│ With TLS inspection:                                                 │
│ ├── Can filter: github.com/malicious/* ✅                         │
│ ├── Can inspect: Payload content ✅                                │
│ └── IDPS can scan decrypted content ✅                             │
│                                                                       │
│ How it works (man-in-the-middle):                                   │
│                                                                       │
│ Client → Firewall → decrypts → inspects → re-encrypts → Server │
│                                                                       │
│ Requirements:                                                        │
│ ├── Intermediate CA certificate in Azure Key Vault               │
│ ├── Firewall generates on-the-fly certificates signed by your CA│
│ ├── Clients MUST trust your intermediate CA                      │
│ │   (Deploy CA cert via GPO, Intune, or MDM)                    │
│ ├── Managed identity for Key Vault access                        │
│ └── ⚠️ Breaks certificate pinning (some apps may fail)            │
│                                                                       │
│ Configure:                                                           │
│ Firewall policy → TLS inspection →                                │
│   ☑ Enable TLS inspection                                         │
│   Key Vault: [kv-firewall-prod ▼]                                │
│   Certificate: [cert-fw-intermediate-ca ▼]                       │
│   Managed identity: [mi-fw-prod ▼]                               │
│                                                                       │
│ Which traffic to inspect:                                            │
│ ├── In Application rules, add "TLS inspection" toggle per rule  │
│ ├── Don't inspect: Banking, healthcare, certificate-pinned apps │
│ └── Do inspect: General web browsing, downloads, unknown sites  │
│                                                                       │
│ ⚡ TLS inspection is required for:                                   │
│    URL path filtering, payload inspection, IDPS on HTTPS         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: DNS Proxy

```
┌─────────────────────────────────────────────────────────────────────┐
│           DNS PROXY                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Azure Firewall acts as a DNS proxy. VMs send DNS queries    │
│ to the firewall, which forwards to the configured DNS server.    │
│                                                                       │
│ Why:                                                                 │
│ ├── FQDN-based rules REQUIRE DNS proxy for correct resolution   │
│ ├── Firewall resolves FQDNs and caches results                   │
│ ├── Required for FQDN filtering in Network rules                │
│ ├── Centralized DNS logging (see what domains VMs resolve)      │
│ └── Works with Azure Private DNS zones                            │
│                                                                       │
│ Configure:                                                           │
│ Firewall policy → DNS →                                            │
│   ☑ Enable DNS Proxy                                               │
│   DNS Servers:                                                      │
│     ● Azure DNS (default — 168.63.129.16)                         │
│     ○ Custom: [10.0.1.4, 10.0.1.5] (your DNS servers)           │
│                                                                       │
│ VNet DNS setting:                                                    │
│ After enabling, point your VNet DNS to the firewall's private IP:│
│ VNet → DNS servers → Custom: 10.0.0.4 (firewall IP)             │
│                                                                       │
│ ⚡ ALWAYS enable DNS proxy if using FQDN-based rules.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Forced Tunneling

```
┌─────────────────────────────────────────────────────────────────────┐
│           FORCED TUNNELING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Send ALL internet-bound traffic from Azure Firewall to      │
│ your on-premises firewall (or NVA) instead of directly to the    │
│ internet. Even the firewall's own management traffic.             │
│                                                                       │
│ Normal:                                                              │
│ Azure VMs → Azure Firewall → Internet ✅                          │
│                                                                       │
│ Forced tunneling:                                                    │
│ Azure VMs → Azure Firewall → VPN/ExpressRoute → On-prem FW →   │
│ Internet                                                            │
│                                                                       │
│ Requirement:                                                         │
│ ├── AzureFirewallManagementSubnet (/26 minimum)                  │
│ ├── Separate public IP for management traffic                    │
│ ├── Management traffic goes directly to internet (not tunneled) │
│ └── User traffic follows forced tunnel to on-prem                │
│                                                                       │
│ Use case:                                                            │
│ ├── Regulatory: All internet traffic MUST go through on-prem    │
│ │   inspection appliance                                          │
│ ├── Single exit point: All internet through on-prem proxy       │
│ └── ⚠️ Adds latency and complexity — use only if required         │
│                                                                       │
│ Enable during firewall creation:                                     │
│ ☑ Enable forced tunneling                                          │
│ Management subnet: AzureFirewallManagementSubnet (10.0.0.64/26) │
│ Management public IP: pip-fw-mgmt                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Route Table (UDR) Setup

```
┌─────────────────────────────────────────────────────────────────────┐
│           USER-DEFINED ROUTES (UDR) FOR FIREWALL                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure Firewall does NOT automatically intercept traffic.           │
│ You MUST create route tables to force traffic through it.         │
│                                                                       │
│ Create Route Table:                                                  │
│ Search "Route tables" → Create                                     │
│   Name: [rt-spoke-to-firewall]                                     │
│   Region: [Central India]                                          │
│   Propagate gateway routes: ● Yes                                 │
│                                                                       │
│ Add routes:                                                          │
│ ┌────────────────┬─────────────────┬──────────────────┬──────────┐│
│ │ Route name     │ Address prefix  │ Next hop type    │ Next hop ││
│ ├────────────────┼─────────────────┼──────────────────┼──────────┤│
│ │ to-internet    │ 0.0.0.0/0       │ Virtual appliance│ 10.0.0.4 ││
│ │                │ (all internet)  │ (firewall IP)    │          ││
│ │                │                 │                  │          ││
│ │ to-spoke2      │ 10.2.0.0/16     │ Virtual appliance│ 10.0.0.4 ││
│ │                │ (spoke 2 CIDR)  │ (firewall IP)    │          ││
│ │                │                 │                  │          ││
│ │ to-onprem      │ 192.168.0.0/16  │ Virtual appliance│ 10.0.0.4 ││
│ │                │ (on-prem CIDR)  │ (firewall IP)    │          ││
│ └────────────────┴─────────────────┴──────────────────┴──────────┘│
│                                                                       │
│ Associate with spoke subnets:                                        │
│ Route table → Subnets → Associate                                  │
│   VNet: [vnet-spoke1-prod ▼]                                      │
│   Subnet: [snet-app ▼]                                             │
│                                                                       │
│ ⚡ Associate with EVERY subnet that needs firewall inspection.      │
│    Without UDR, traffic bypasses the firewall completely!          │
│                                                                       │
│ ⚠️ Do NOT associate with AzureFirewallSubnet or GatewaySubnet!     │
│                                                                       │
│ Full flow:                                                           │
│ VM (10.1.1.5) → wants google.com                                  │
│ 1. UDR: 0.0.0.0/0 → next hop 10.0.0.4 (firewall)               │
│ 2. Traffic hits Azure Firewall                                     │
│ 3. Firewall checks: DNAT → Network rules → Application rules    │
│ 4. Application rule: *.google.com → Allow                        │
│ 5. Traffic exits via firewall public IP (SNAT)                   │
│ 6. Response comes back to firewall → DNAT reverse → VM          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Azure Firewall Manager

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE FIREWALL MANAGER                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Central management for multiple Azure Firewalls across      │
│ subscriptions and regions.                                         │
│                                                                       │
│ Features:                                                            │
│ ├── Centralized policy management (one policy → many firewalls) │
│ ├── Hub-and-spoke topology management                             │
│ ├── Secured virtual hubs (Azure Virtual WAN integration)         │
│ ├── Third-party SECaaS (Zscaler, Check Point, iboss)            │
│ └── DDoS protection plan association                              │
│                                                                       │
│ Two architectures:                                                   │
│                                                                       │
│ 1. Hub VNet (standard):                                             │
│    Your VNet with Azure Firewall                                  │
│    You manage peering, route tables, firewall                     │
│                                                                       │
│ 2. Secured Virtual Hub (Virtual WAN):                               │
│    Azure Firewall deployed INSIDE Virtual WAN hub               │
│    Routing is automatic (no manual UDRs!)                        │
│    ⚡ Simpler for large-scale deployments                          │
│                                                                       │
│ Access: Search "Firewall Manager" in portal                       │
│ ├── Overview: All firewalls, policies, status                     │
│ ├── Azure Firewall Policies: Create/manage policies              │
│ ├── Virtual Hubs: Manage secured Virtual WAN hubs                │
│ ├── Virtual Networks: Associate hub VNets                        │
│ └── Security Partner Providers: Third-party integration          │
│                                                                       │
│ ⚡ Use Firewall Manager when you have 2+ firewalls.                 │
│    Single firewall → just use the policy directly.                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Monitoring & Logging

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Diagnostic settings (MUST configure):                                │
│ Firewall → Diagnostic settings → Add diagnostic setting           │
│                                                                       │
│ Logs to send:                                                        │
│ ├── ☑ Azure Firewall Application Rule Log                        │
│ ├── ☑ Azure Firewall Network Rule Log                            │
│ ├── ☑ Azure Firewall NAT Rule Log                                │
│ ├── ☑ Azure Firewall Threat Intel Log                            │
│ ├── ☑ Azure Firewall IDPS Log (Premium)                         │
│ ├── ☑ Azure Firewall DNS Proxy Log                               │
│ └── ☑ Azure Firewall TLS Inspection Log (Premium)               │
│                                                                       │
│ Send to:                                                             │
│ ├── ☑ Log Analytics workspace (for KQL queries, dashboards)     │
│ ├── ☐ Storage account (long-term retention, cheap)               │
│ └── ☐ Event Hub (stream to SIEM like Splunk, Sentinel)          │
│                                                                       │
│ Log Analytics → Resource-specific (Structured logs):               │
│ ├── AZFWApplicationRule: Outbound FQDN matches                   │
│ ├── AZFWNetworkRule: IP/port matches                              │
│ ├── AZFWNatRule: DNAT matches                                    │
│ ├── AZFWThreatIntel: Threat intelligence matches                │
│ ├── AZFWIdpsSignature: IDPS signature matches                   │
│ └── AZFWDnsQuery: DNS proxy queries                               │
│                                                                       │
│ Example KQL queries:                                                 │
│                                                                       │
│ // Top denied outbound FQDNs                                       │
│ AZFWApplicationRule                                                  │
│ | where Action == "Deny"                                            │
│ | summarize count() by Fqdn                                        │
│ | top 20 by count_                                                  │
│                                                                       │
│ // Threat intelligence hits                                          │
│ AZFWThreatIntel                                                      │
│ | where ThreatDescription != ""                                     │
│ | project TimeGenerated, SourceIp, DestinationIp,                  │
│   ThreatDescription, Action                                         │
│                                                                       │
│ // IDPS alerts                                                       │
│ AZFWIdpsSignature                                                    │
│ | where Severity <= 2                                               │
│ | project TimeGenerated, SourceIp, SignatureId, Description        │
│                                                                       │
│ Azure Firewall Workbook:                                             │
│ Monitor → Workbooks → Azure Firewall Workbook (built-in)          │
│ ├── Application rule hits                                          │
│ ├── Network rule hits                                              │
│ ├── Threat intelligence hits                                      │
│ ├── IDPS alerts                                                    │
│ ├── DNS proxy activity                                             │
│ └── ⚡ Best pre-built dashboard — use it!                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 12: Comparison with AWS & GCP

```
┌─────────────────────────────────────────────────────────────────────┐
│           CROSS-CLOUD COMPARISON                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌────────────────┬──────────────────┬──────────┬─────────────────┐ │
│ │ Feature        │ Azure Firewall   │ AWS      │ GCP             │ │
│ ├────────────────┼──────────────────┼──────────┼─────────────────┤ │
│ │ Managed FW     │ Azure Firewall   │ AWS      │ Cloud Firewall  │ │
│ │                │                  │ Network  │ (L4) +          │ │
│ │                │                  │ Firewall │ Cloud Armor (L7)│ │
│ │ FQDN filtering │ Yes ✅          │ Yes ✅   │ Limited         │ │
│ │ L7 inspection  │ Premium ✅      │ Suricata │ Cloud Armor     │ │
│ │ TLS inspection │ Premium ✅      │ Yes ✅   │ No ❌          │ │
│ │ IDPS           │ Premium ✅      │ Suricata │ Cloud IDS       │ │
│ │                │                  │ (via NFW)│ (separate svc)  │ │
│ │ Threat intel   │ Built-in ✅     │ No ❌    │ Cloud Armor     │ │
│ │                │                  │          │ (threat intel)  │ │
│ │ Policy inherit │ Parent→child ✅ │ No       │ Hierarchical    │ │
│ │                │                  │          │ fw policies     │ │
│ │ HA             │ Multi-AZ ✅     │ Multi-AZ │ Global ✅       │ │
│ │ Cost (standard)│ ~$912/mo        │ ~$328/mo │ Per-rule pricing│ │
│ └────────────────┴──────────────────┴──────────┴─────────────────┘ │
│                                                                       │
│ Key differences:                                                     │
│ ├── Azure: All-in-one managed firewall (L4+L7+IDPS+ThreatIntel)│
│ ├── AWS: Network Firewall (Suricata-based) + WAF (separate L7) │
│ └── GCP: Cloud Firewall (VPC rules) + Cloud Armor (WAF/DDoS)   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 13: Terraform & Bicep Examples

### Terraform

```hcl
# Azure Firewall
resource "azurerm_firewall" "hub" {
  name                = "fw-hub-prod"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  sku_name            = "AZFW_VNet"
  sku_tier            = "Standard"
  zones               = ["1", "2", "3"]
  firewall_policy_id  = azurerm_firewall_policy.prod.id
  threat_intel_mode   = "Deny"

  ip_configuration {
    name                 = "fw-ipconfig"
    subnet_id            = azurerm_subnet.fw.id
    public_ip_address_id = azurerm_public_ip.fw.id
  }

  tags = {
    environment = "prod"
    managed-by  = "network-team"
  }
}

# Firewall subnet (MUST be named "AzureFirewallSubnet")
resource "azurerm_subnet" "fw" {
  name                 = "AzureFirewallSubnet"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.hub.name
  address_prefixes     = ["10.0.0.0/26"]
}

# Public IP for firewall
resource "azurerm_public_ip" "fw" {
  name                = "pip-fw-hub-prod"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]
}

# Firewall Policy
resource "azurerm_firewall_policy" "prod" {
  name                     = "policy-fw-prod"
  location                 = azurerm_resource_group.network.location
  resource_group_name      = azurerm_resource_group.network.name
  sku                      = "Standard"
  threat_intelligence_mode = "Deny"

  dns {
    proxy_enabled = true
    servers       = ["168.63.129.16"]
  }
}

# Rule Collection Group: DNAT
resource "azurerm_firewall_policy_rule_collection_group" "dnat" {
  name               = "rcg-dnat-inbound"
  firewall_policy_id = azurerm_firewall_policy.prod.id
  priority           = 100

  nat_rule_collection {
    name     = "rc-dnat-web"
    priority = 100
    action   = "Dnat"

    rule {
      name                = "web-https"
      protocols           = ["TCP"]
      source_addresses    = ["*"]
      destination_address = azurerm_public_ip.fw.ip_address
      destination_ports   = ["443"]
      translated_address  = "10.1.1.4"
      translated_port     = "443"
    }
  }
}

# Rule Collection Group: Network rules
resource "azurerm_firewall_policy_rule_collection_group" "network" {
  name               = "rcg-network"
  firewall_policy_id = azurerm_firewall_policy.prod.id
  priority           = 200

  network_rule_collection {
    name     = "rc-allow-internal"
    priority = 100
    action   = "Allow"

    rule {
      name                  = "spoke1-to-spoke2"
      protocols             = ["TCP"]
      source_addresses      = ["10.1.0.0/16"]
      destination_addresses = ["10.2.0.0/16"]
      destination_ports     = ["443", "8080"]
    }

    rule {
      name                  = "all-to-dns"
      protocols             = ["TCP", "UDP"]
      source_addresses      = ["10.0.0.0/8"]
      destination_addresses = ["168.63.129.16"]
      destination_ports     = ["53"]
    }
  }
}

# Rule Collection Group: Application rules
resource "azurerm_firewall_policy_rule_collection_group" "application" {
  name               = "rcg-application"
  firewall_policy_id = azurerm_firewall_policy.prod.id
  priority           = 300

  application_rule_collection {
    name     = "rc-allow-internet"
    priority = 100
    action   = "Allow"

    rule {
      name = "allow-updates"
      protocols {
        type = "Https"
        port = 443
      }
      source_addresses  = ["10.0.0.0/8"]
      destination_fqdns = [
        "*.ubuntu.com",
        "*.microsoft.com",
        "*.windowsupdate.com"
      ]
    }

    rule {
      name = "allow-github"
      protocols {
        type = "Https"
        port = 443
      }
      source_addresses  = ["10.1.0.0/16"]
      destination_fqdns = [
        "github.com",
        "*.github.com",
        "*.githubusercontent.com"
      ]
    }
  }

  application_rule_collection {
    name     = "rc-deny-all-internet"
    priority = 1000
    action   = "Deny"

    rule {
      name = "deny-all"
      protocols {
        type = "Https"
        port = 443
      }
      protocols {
        type = "Http"
        port = 80
      }
      source_addresses  = ["*"]
      destination_fqdns = ["*"]
    }
  }
}

# Route table for spoke subnets
resource "azurerm_route_table" "spoke" {
  name                = "rt-spoke-to-firewall"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name

  route {
    name                   = "to-internet"
    address_prefix         = "0.0.0.0/0"
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = azurerm_firewall.hub.ip_configuration[0].private_ip_address
  }

  route {
    name                   = "to-spoke2"
    address_prefix         = "10.2.0.0/16"
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = azurerm_firewall.hub.ip_configuration[0].private_ip_address
  }
}

resource "azurerm_subnet_route_table_association" "spoke1" {
  subnet_id      = azurerm_subnet.spoke1_app.id
  route_table_id = azurerm_route_table.spoke.id
}
```

### Bicep

```bicep
resource firewall 'Microsoft.Network/azureFirewalls@2023-09-01' = {
  name: 'fw-hub-prod'
  location: location
  zones: ['1', '2', '3']
  properties: {
    sku: {
      name: 'AZFW_VNet'
      tier: 'Standard'
    }
    threatIntelMode: 'Deny'
    firewallPolicy: {
      id: firewallPolicy.id
    }
    ipConfigurations: [
      {
        name: 'fw-ipconfig'
        properties: {
          subnet: { id: fwSubnet.id }
          publicIPAddress: { id: fwPublicIp.id }
        }
      }
    ]
  }
}

resource firewallPolicy 'Microsoft.Network/firewallPolicies@2023-09-01' = {
  name: 'policy-fw-prod'
  location: location
  properties: {
    sku: { tier: 'Standard' }
    threatIntelMode: 'Deny'
    dnsSettings: {
      enableProxy: true
      servers: ['168.63.129.16']
    }
  }
}
```

---

## Part 14: az CLI Reference

```bash
# Create firewall
az network firewall create \
  --name fw-hub-prod \
  --resource-group rg-network-prod \
  --location centralindia \
  --tier Standard \
  --zones 1 2 3 \
  --firewall-policy policy-fw-prod \
  --vnet-name vnet-hub-prod

# Create IP config
az network firewall ip-config create \
  --name fw-ipconfig \
  --firewall-name fw-hub-prod \
  --resource-group rg-network-prod \
  --vnet-name vnet-hub-prod \
  --public-ip-address pip-fw-hub-prod

# Create firewall policy
az network firewall policy create \
  --name policy-fw-prod \
  --resource-group rg-network-prod \
  --location centralindia \
  --sku Standard \
  --threat-intel-mode Deny

# Enable DNS proxy
az network firewall policy update \
  --name policy-fw-prod \
  --resource-group rg-network-prod \
  --enable-dns-proxy true \
  --dns-servers 168.63.129.16

# Create rule collection group
az network firewall policy rule-collection-group create \
  --name rcg-application \
  --policy-name policy-fw-prod \
  --resource-group rg-network-prod \
  --priority 300

# Add application rule collection
az network firewall policy rule-collection-group collection add-filter-collection \
  --name rc-allow-internet \
  --policy-name policy-fw-prod \
  --resource-group rg-network-prod \
  --rule-collection-group-name rcg-application \
  --collection-priority 100 \
  --action Allow \
  --rule-type ApplicationRule \
  --rule-name allow-github \
  --source-addresses "10.1.0.0/16" \
  --target-fqdns "github.com" "*.github.com" \
  --protocols Https=443

# Add network rule collection
az network firewall policy rule-collection-group collection add-filter-collection \
  --name rc-allow-internal \
  --policy-name policy-fw-prod \
  --resource-group rg-network-prod \
  --rule-collection-group-name rcg-network \
  --collection-priority 100 \
  --action Allow \
  --rule-type NetworkRule \
  --rule-name spoke1-to-spoke2 \
  --source-addresses "10.1.0.0/16" \
  --destination-addresses "10.2.0.0/16" \
  --destination-ports 443 8080 \
  --ip-protocols TCP

# Check firewall status
az network firewall show \
  --name fw-hub-prod \
  --resource-group rg-network-prod \
  --query "{name:name, provisioningState:provisioningState, privateIp:ipConfigurations[0].privateIPAddress}" \
  --output table

# List all rules in a policy
az network firewall policy rule-collection-group list \
  --policy-name policy-fw-prod \
  --resource-group rg-network-prod \
  --output table
```

---

## Part 15: Real-World Patterns

### Startup

```
Architecture: Single VNet (no hub-spoke yet)
Firewall: Azure Firewall Basic ($274/month)

Setup:
├── AzureFirewallSubnet in main VNet
├── Basic tier (250 Mbps, enough for small workloads)
├── Firewall policy with:
│   ├── Application rules: Allow *.microsoft.com, github.com,
│   │   registry.npmjs.org, *.docker.io, *.ubuntu.com
│   ├── Network rules: Allow Azure DNS (53)
│   └── Threat intelligence: Alert and deny
├── UDR: 0.0.0.0/0 → firewall for app subnets
├── Diagnostic logs → Log Analytics
└── No DNAT (use Application Gateway or Azure Front Door)

Cost: ~$274/month (Basic SKU) + data processing
```

### Mid-Size

```
Architecture: Hub-spoke (hub VNet + 3-5 spoke VNets)
Firewall: Azure Firewall Standard ($912/month)

Hub VNet (10.0.0.0/16):
├── AzureFirewallSubnet (10.0.0.0/26)
├── GatewaySubnet (10.0.1.0/27) — VPN/ExpressRoute
├── Azure Bastion subnet (10.0.2.0/26)
└── Shared services subnet (10.0.3.0/24)

Spoke VNets (peered to hub):
├── spoke-web (10.1.0.0/16)
├── spoke-api (10.2.0.0/16)
└── spoke-data (10.3.0.0/16)

Firewall policy:
├── DNAT: Port 443 → Application Gateway (not direct to VMs)
├── Network rules:
│   ├── spoke-web → spoke-api (443, 8080)
│   ├── spoke-api → spoke-data (1433, 5432)
│   ├── All → Azure DNS (53)
│   └── All → on-prem (192.168.0.0/16) via VPN
├── Application rules:
│   ├── Dev spoke → github.com, npm, Docker Hub
│   ├── All → Windows Update (FQDN tag)
│   ├── All → Azure Backup (FQDN tag)
│   └── Deny all other outbound
├── Threat intelligence: Alert and deny
├── DNS proxy: Enabled
└── Diagnostics: Log Analytics + Storage (90 days)

Route tables:
├── rt-spoke-to-fw: 0.0.0.0/0 → firewall
├── Applied to all spoke subnets
└── Spoke-to-spoke traffic forced through firewall

Cost: ~$912/month + ~$200 data processing = ~$1,100/month
```

### Enterprise

```
Architecture: Virtual WAN with Secured Hubs (multi-region)
Firewall: Azure Firewall Premium (per hub)

Virtual WAN Hub 1 (Central India):
├── Azure Firewall Premium (IDPS + TLS inspection)
├── ExpressRoute gateway (on-prem connectivity)
├── 10 spoke VNets (prod workloads)
└── S2S VPN (branch offices)

Virtual WAN Hub 2 (West Europe):
├── Azure Firewall Premium
├── ExpressRoute gateway
├── 5 spoke VNets (EU workloads)
└── S2S VPN (EU offices)

Firewall Policy hierarchy:
├── Parent: policy-base-org (org-wide rules)
│   ├── Block all threat intel (deny)
│   ├── IDPS: Alert and deny
│   ├── Block gambling, adult, P2P web categories
│   └── Allow Azure services (FQDN tags)
├── Child India: policy-fw-india-prod
│   ├── India-specific app rules
│   ├── On-prem routes for India DC
│   └── TLS inspection for outbound HTTPS
└── Child EU: policy-fw-eu-prod
    ├── EU-specific GDPR compliance rules
    ├── On-prem routes for EU DC
    └── TLS inspection + URL filtering

Azure Firewall Manager:
├── Single pane of glass for all firewalls
├── Policy deployment across regions
├── Compliance dashboards
└── Integration with Microsoft Sentinel (SIEM)

Monitoring:
├── Log Analytics per region
├── Microsoft Sentinel: Correlate firewall logs with identity,
│   endpoint, cloud app events
├── Azure Firewall Workbook (per region)
├── Alerts: Threat intel hits, IDPS critical alerts,
│   unusual traffic patterns
└── Data retention: 1 year (compliance)

Cost: ~$3,650-5,000/month per hub (Premium + data processing)
      + Sentinel costs
      ~$10,000-15,000/month total (2 hubs)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Managed cloud-native firewall (L3/L4/L7) |
| SKUs | Basic ($274/mo), Standard ($912/mo), Premium ($1,825/mo) |
| Subnet | AzureFirewallSubnet (dedicated, /26 minimum) |
| Placement | Hub VNet or Secured Virtual Hub |
| Policy | Rule Collection Groups → Rule Collections → Rules |
| Rule types | DNAT (inbound), Network (L4), Application (L7/FQDN) |
| Processing order | DNAT → Network → Application (lower priority # first) |
| Threat Intel | Microsoft feed, Alert or Alert+Deny, included |
| IDPS | Premium — 67,000+ signatures, Alert or Alert+Deny |
| TLS inspection | Premium — decrypt/inspect HTTPS, requires CA cert |
| DNS proxy | Forward DNS through firewall, required for FQDN rules |
| Forced tunneling | Route all internet via on-prem, needs mgmt subnet |
| UDR | Required — route table pointing 0.0.0.0/0 to firewall |
| Monitoring | Diagnostic logs → Log Analytics, Workbook, Sentinel |
| Firewall Manager | Multi-firewall management, policy hierarchy |
| AWS equivalent | AWS Network Firewall + WAF |
| GCP equivalent | Cloud Firewall + Cloud Armor + Cloud IDS |

---

## What's Next?

In the next chapter, we'll cover Azure Traffic Manager — global DNS-based load balancing across regions.

→ Next: [Chapter 14: Traffic Manager](14-traffic-manager.md)

---

*Last Updated: May 2026*
