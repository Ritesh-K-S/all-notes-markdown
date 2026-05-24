# Chapter 11: Azure Load Balancer & Application Gateway

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Load Balancing Decision Tree](#part-1-azure-load-balancing-decision-tree)
- [Part 2: Azure Load Balancer (Layer 4)](#part-2-azure-load-balancer-layer-4)
- [Part 3: Application Gateway (Layer 7)](#part-3-application-gateway-layer-7)
- [Part 4: Terraform / Bicep Examples](#part-4-terraform--bicep-examples)
- [Part 5: az CLI Examples](#part-5-az-cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference: Azure vs AWS vs GCP](#quick-reference-azure-vs-aws-vs-gcp)
- [What's Next?](#whats-next)

---

## Overview

Azure has two primary load balancing services: **Azure Load Balancer** (Layer 4 — TCP/UDP, like AWS NLB) and **Application Gateway** (Layer 7 — HTTP/S, like AWS ALB). Unlike AWS where ALB and NLB share the "Elastic Load Balancing" umbrella, Azure treats them as completely separate services with different portals, APIs, and pricing.

```
What you'll learn:
├── Azure Load Balancing Decision Tree
├── Azure Load Balancer (Layer 4)
│   ├── Standard vs Basic SKU
│   ├── Public vs Internal
│   ├── Frontend IP, Backend pools
│   ├── Health probes
│   ├── Load balancing rules
│   ├── Inbound NAT rules
│   ├── Outbound rules (SNAT)
│   └── HA ports
├── Application Gateway (Layer 7)
│   ├── SKUs: Standard_v2, WAF_v2
│   ├── Listeners (multi-site, SSL)
│   ├── Routing rules (path-based, basic)
│   ├── Backend pools + HTTP settings
│   ├── Health probes
│   ├── WAF (built-in!)
│   ├── Autoscaling
│   ├── Rewrite rules
│   └── Redirect
├── Comparison table (LB = NLB, AppGW = ALB)
└── Real-world patterns
```

---

## Part 1: Azure Load Balancing Decision Tree

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE LOAD BALANCING DECISION TREE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure has 4 load balancing services (yes, 4!):                     │
│                                                                       │
│ ┌────────────────────────────────────────────────────────────┐     │
│ │                   Is traffic HTTP/HTTPS?                    │     │
│ │                   ┌─────┬─────────┐                        │     │
│ │                   │ Yes │   No    │                        │     │
│ │                   ▼     │         ▼                        │     │
│ │             Is it global?│   Azure Load Balancer           │     │
│ │           ┌─────┬────┐  │   (Layer 4 — TCP/UDP)          │     │
│ │           │ Yes │ No │  │                                  │     │
│ │           ▼     ▼    │  │   Is it global?                 │     │
│ │      Azure    App    │  │   ┌────┬─────┐                  │     │
│ │      Front   Gateway │  │   │Yes │ No  │                  │     │
│ │      Door    (regional) │   ▼    ▼     │                  │     │
│ │      (global) (Layer 7) │ Traffic  Azure                  │     │
│ │      (Layer 7)          │ Manager  Load                   │     │
│ │                         │ (DNS)    Balancer                │     │
│ └────────────────────────────────────────────────────────────┘     │
│                                                                       │
│ Summary:                                                             │
│ ┌────────────────────┬───────┬──────────┬───────────────────────┐ │
│ │ Service            │ Layer │ Scope    │ AWS Equivalent         │ │
│ ├────────────────────┼───────┼──────────┼───────────────────────┤ │
│ │ Azure Load Balancer│ 4     │ Regional │ NLB                    │ │
│ │ Application Gateway│ 7     │ Regional │ ALB                    │ │
│ │ Azure Front Door   │ 7     │ Global   │ CloudFront+ALB(global)│ │
│ │ Traffic Manager    │ DNS   │ Global   │ Route 53 (latency)    │ │
│ └────────────────────┴───────┴──────────┴───────────────────────┘ │
│                                                                       │
│ This chapter: Azure Load Balancer + Application Gateway            │
│ (Front Door was covered in Chapter 10 - CDN)                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Azure Load Balancer (Layer 4)

### Standard vs Basic SKU

```
┌─────────────────────────────────────────────────────────────────────┐
│           STANDARD vs BASIC SKU                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────┬──────────────────┬──────────────────────┐ │
│ │ Feature              │ Standard         │ Basic (LEGACY!)      │ │
│ ├──────────────────────┼──────────────────┼──────────────────────┤ │
│ │ Backend pool size    │ Up to 1,000      │ Up to 300            │ │
│ │ Health probes        │ TCP, HTTP, HTTPS │ TCP, HTTP            │ │
│ │ AZ support           │ Zone-redundant ✅│ Not available ❌     │ │
│ │ HA Ports             │ Yes ✅           │ No ❌               │ │
│ │ Outbound rules       │ Yes ✅           │ No ❌               │ │
│ │ Multiple frontends   │ Yes              │ Yes                  │ │
│ │ SLA                  │ 99.99%           │ No SLA               │ │
│ │ Security             │ Closed by default│ Open by default      │ │
│ │                      │ (NSG required)   │                      │ │
│ │ Pricing              │ Paid             │ Free (retiring!)     │ │
│ │ Global VNet peering  │ Yes ✅           │ No ❌               │ │
│ │ Cross-region (preview)│ Yes             │ No                   │ │
│ └──────────────────────┴──────────────────┴──────────────────────┘ │
│                                                                       │
│ ⚠️ Basic LB is being RETIRED September 2025!                        │
│    ALWAYS use Standard. Never create Basic for new workloads.      │
│                                                                       │
│ ⚡ Standard LB is "secure by default":                               │
│    Backend VMs MUST have an NSG that allows traffic.               │
│    Basic was open by default (security risk).                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Azure Load Balancer (Full Portal Walkthrough)

```
Portal → Load balancers → Create

┌─────────────────────────────────────────────────────────────────┐
│           CREATE LOAD BALANCER                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Basics tab ──                                                │
│ Subscription:     [TechCorp Production ▼]                      │
│ Resource group:   [rg-networking-prod ▼]                       │
│ Name:             [lb-prod-standard]                            │
│ Region:           [Central India ▼]                            │
│ SKU:              ● Standard  ○ Gateway                        │
│ Type:             ● Public    ○ Internal                       │
│ Tier:             ● Regional  ○ Global (cross-region)         │
│                                                                   │
│   Public: Frontend has public IP (internet-facing)             │
│   Internal: Frontend has private IP (internal services)        │
│                                                                   │
│   Gateway SKU: For third-party NVAs (like AWS GLB)             │
│   Cross-region: LBs across regions (global failover)           │
│                                                                   │
│ ── Frontend IP configuration tab ──                             │
│ [+ Add a frontend IP configuration]                             │
│ Name: [fe-prod-public]                                          │
│ IP version: ● IPv4  ○ IPv6                                    │
│ IP type: ● IP address  ○ IP prefix                            │
│ Public IP: [Create new]                                         │
│   Name: [pip-lb-prod]                                           │
│   SKU: Standard                                                │
│   Tier: Regional                                                │
│   Availability zone: ● Zone-redundant (recommended)            │
│                       ○ Zone 1 / Zone 2 / Zone 3              │
│                       ○ No zone                                │
│   Routing: ● Microsoft global network  ○ Internet             │
│                                                                   │
│ ⚡ Unlike AWS NLB which gets per-AZ IPs,                        │
│    Azure LB gets ONE public IP (zone-redundant).               │
│    You can add more frontend IPs if needed.                    │
│                                                                   │
│ ── Backend pools tab ──                                         │
│ [+ Add a backend pool]                                          │
│ Name: [bp-web-servers]                                          │
│ Virtual network: [vnet-prod ▼]                                 │
│ Backend pool configuration:                                    │
│   ● NIC (Network Interface Card)                               │
│   ○ IP address                                                 │
│                                                                   │
│   NIC-based: Add VMs by their network interface                │
│   IP-based: Add any IP address (more flexible)                 │
│                                                                   │
│ Add VMs:                                                        │
│   ☑ vm-web-01 (10.0.1.4) — nic-web-01                        │
│   ☑ vm-web-02 (10.0.1.5) — nic-web-02                        │
│   ☑ vm-web-03 (10.0.1.6) — nic-web-03                        │
│                                                                   │
│ ── Inbound rules tab ──                                         │
│                                                                   │
│ Load balancing rule: [+ Add a load balancing rule]             │
│ ┌─────────────────────────────────────────────────────────┐    │
│ │ Name:              [rule-web-http]                       │    │
│ │ IP Version:        [IPv4 ▼]                            │    │
│ │ Frontend IP:       [fe-prod-public ▼]                  │    │
│ │ Backend pool:      [bp-web-servers ▼]                  │    │
│ │ Protocol:          [TCP ▼]                             │    │
│ │ Port:              [80] (frontend port)                │    │
│ │ Backend port:      [8080] (port on VMs)                │    │
│ │ Health probe:      [Create new]                        │    │
│ │                                                         │    │
│ │ ── Health probe settings ──                             │    │
│ │ Name:       [probe-web-http]                           │    │
│ │ Protocol:   [HTTP ▼]                                   │    │
│ │ Port:       [8080]                                     │    │
│ │ Path:       [/health]                                  │    │
│ │ Interval:   [5] seconds                                │    │
│ │ Unhealthy threshold: [2] consecutive failures         │    │
│ │                                                         │    │
│ │ Session persistence:                                    │    │
│ │ [None ▼]                                               │    │
│ │ ├── None: Distribute evenly                           │    │
│ │ ├── Client IP: Same IP → same VM                     │    │
│ │ └── Client IP and protocol: IP + protocol → same VM  │    │
│ │                                                         │    │
│ │ Idle timeout: [4] minutes (1-30)                      │    │
│ │ TCP reset: ● Enabled (send RST on idle timeout)      │    │
│ │ Floating IP: ○ Disabled (default)                    │    │
│ │              ● Enabled (for SQL AlwaysOn, etc.)       │    │
│ │                                                         │    │
│ │ ⚠️ Floating IP = Direct Server Return (DSR)            │    │
│ │    Backend sees the frontend IP as destination,        │    │
│ │    not its own IP. Required for SQL AlwaysOn,          │    │
│ │    HA setups with virtual IP.                          │    │
│ │                                                         │    │
│ └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│ Inbound NAT rule: [+ Add an inbound NAT rule]                 │
│ ┌─────────────────────────────────────────────────────────┐    │
│ │ Name:              [nat-ssh-web01]                      │    │
│ │ Type:              ○ Azure Virtual Machine              │    │
│ │                    ● Backend pool                       │    │
│ │ Target:            [bp-web-servers ▼]                  │    │
│ │ Frontend IP:       [fe-prod-public ▼]                  │    │
│ │ Frontend port start: [50000]                           │    │
│ │ Backend port:      [22]                                │    │
│ │ Protocol:          [TCP ▼]                             │    │
│ │                                                         │    │
│ │ Result:                                                 │    │
│ │ pip-lb-prod:50000 → vm-web-01:22 (SSH)                │    │
│ │ pip-lb-prod:50001 → vm-web-02:22 (SSH)                │    │
│ │ pip-lb-prod:50002 → vm-web-03:22 (SSH)                │    │
│ │                                                         │    │
│ │ ⚡ Like port forwarding. Common for SSH/RDP access     │    │
│ │    without giving each VM a public IP.                 │    │
│ │    AWS NLB doesn't have this — it's more like          │    │
│ │    iptables DNAT rules.                                │    │
│ │                                                         │    │
│ └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│ ── Outbound rules tab ──                                        │
│ [+ Add an outbound rule]                                       │
│ ┌─────────────────────────────────────────────────────────┐    │
│ │ Name:              [rule-outbound-snat]                 │    │
│ │ Frontend IP:       [fe-prod-public ▼]                  │    │
│ │ Backend pool:      [bp-web-servers ▼]                  │    │
│ │ Protocol:          [All ▼]                             │    │
│ │ Idle timeout:      [4] minutes                         │    │
│ │ TCP reset:         [Enabled ▼]                         │    │
│ │ Port allocation:                                        │    │
│ │   ○ Use the default number of outbound ports           │    │
│ │   ● Manually choose number of outbound ports           │    │
│ │   Ports per instance: [10000]                          │    │
│ │                                                         │    │
│ │ ⚠️ SNAT (Source NAT) = Backend VMs use LB's public IP  │    │
│ │    for outbound internet traffic.                       │    │
│ │                                                         │    │
│ │ ⚠️ SNAT port exhaustion is a REAL problem!              │    │
│ │    Each outbound connection uses a SNAT port.           │    │
│ │    Default: ~1024 ports per VM.                         │    │
│ │    If your app makes many outbound connections          │    │
│ │    (calling APIs, databases), you WILL exhaust ports!  │    │
│ │                                                         │    │
│ │ Solutions for SNAT exhaustion:                          │    │
│ │ ├── NAT Gateway (recommended) — Chapter 7              │    │
│ │ ├── More frontend IPs (each adds 64K ports)            │    │
│ │ ├── Manual port allocation (increase per instance)     │    │
│ │ └── Connection pooling in your application             │    │
│ │                                                         │    │
│ └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│ [Review + create]                                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### HA Ports

```
┌─────────────────────────────────────────────────────────────────────┐
│           HA PORTS                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Load balance ALL ports (all protocols) with a single rule.  │
│                                                                       │
│ Instead of creating rules for port 80, 443, 3306, 5432, etc.,    │
│ HA ports rule covers ALL ports in one rule.                        │
│                                                                       │
│ Use for:                                                             │
│ ├── Network Virtual Appliances (firewalls, routers)               │
│ ├── Multiple ports on same backend pool                            │
│ ├── SQL AlwaysOn Availability Groups                               │
│ └── Any scenario needing all-port forwarding                      │
│                                                                       │
│ Requirements:                                                        │
│ ├── Standard SKU only                                              │
│ ├── Internal LB only (public LB doesn't support HA ports)        │
│ ├── Floating IP required for some scenarios                       │
│ └── Protocol: All                                                  │
│                                                                       │
│ Create:                                                              │
│ LB rule → Protocol: All → Port: 0 (means ALL ports)              │
│ Automatically becomes an HA ports rule.                            │
│                                                                       │
│ ⚠️ AWS NLB doesn't have this — you need one listener per port.     │
│    GCP Internal LB supports all-ports forwarding similarly.       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Internal Load Balancer

```
┌─────────────────────────────────────────────────────────────────────┐
│           INTERNAL LOAD BALANCER                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Same as public LB, but with private frontend IP.                   │
│                                                                       │
│ Create → Type: Internal                                             │
│                                                                       │
│ Frontend IP:                                                         │
│   Name: [fe-internal-db]                                            │
│   VNet: [vnet-prod ▼]                                               │
│   Subnet: [subnet-data ▼]                                           │
│   Assignment: ● Dynamic  ○ Static                                  │
│   Static IP: [10.0.3.100] (optional, pick from subnet range)      │
│   Availability zone: ● Zone-redundant                              │
│                                                                       │
│ Common use cases:                                                    │
│ ├── Database tier: Multiple SQL Server replicas behind ILB        │
│ ├── Middleware: App servers not exposed to internet                │
│ ├── Internal API: Microservices calling each other                │
│ └── NVA: Route traffic through firewall VMs                       │
│                                                                       │
│ ⚡ Internal LB IP acts like a virtual IP (VIP) for your service.   │
│    Other VMs connect to 10.0.3.100:5432, LB distributes to pool. │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Application Gateway (Layer 7)

### Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│           APPLICATION GATEWAY                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Azure's Layer 7 (HTTP/HTTPS) load balancer.                        │
│ = AWS ALB + AWS WAF combined in one service.                       │
│                                                                       │
│ Key features:                                                        │
│ ├── URL-based routing (path + host)                               │
│ ├── Multi-site hosting (multiple domains)                         │
│ ├── SSL termination & end-to-end SSL                              │
│ ├── WAF built-in (not separate like AWS WAF!)                    │
│ ├── Autoscaling (v2 SKU)                                          │
│ ├── Zone-redundant                                                │
│ ├── WebSocket & HTTP/2 support                                    │
│ ├── Session affinity (cookie-based)                               │
│ ├── URL rewrite & redirect                                        │
│ ├── Header rewrite                                                │
│ └── Custom error pages                                            │
│                                                                       │
│ Architecture:                                                        │
│ Client → Frontend IP → Listener → Routing Rule →                 │
│ Backend Pool (via HTTP Settings / Backend Settings) → VMs         │
│                                                                       │
│ Components:                                                          │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ ┌──────────────┐   ┌──────────┐   ┌───────────────────┐   │  │
│ │ │ Frontend IP   │ → │ Listener │ → │ Routing Rule      │   │  │
│ │ │ (Public/Pvt)  │   │ (port +  │   │ (basic or         │   │  │
│ │ │               │   │  cert)   │   │  path-based)      │   │  │
│ │ └──────────────┘   └──────────┘   └────────┬──────────┘   │  │
│ │                                             │               │  │
│ │                                    ┌────────▼──────────┐   │  │
│ │                                    │ Backend Pool      │   │  │
│ │                                    │ (VMs, VMSS, IPs,  │   │  │
│ │                                    │  App Service, etc.)│   │  │
│ │                                    └────────┬──────────┘   │  │
│ │                                             │               │  │
│ │                                    ┌────────▼──────────┐   │  │
│ │                                    │ Backend Settings  │   │  │
│ │                                    │ (protocol, port,  │   │  │
│ │                                    │  timeout, affinity│   │  │
│ │                                    │  host override)   │   │  │
│ │                                    └───────────────────┘   │  │
│ │                                                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚠️ App Gateway is a DEPLOYED RESOURCE — it runs on dedicated      │
│    VMs (instances) in a DEDICATED SUBNET.                          │
│    Unlike Azure LB which is fully managed infrastructure,         │
│    App Gateway needs its own subnet (/24 or /26 recommended).     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### SKUs

```
┌─────────────────────────────────────────────────────────────────────┐
│           APPLICATION GATEWAY SKUs                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────┬──────────────────┬──────────────────────┐ │
│ │ Feature              │ Standard_v2      │ WAF_v2               │ │
│ ├──────────────────────┼──────────────────┼──────────────────────┤ │
│ │ URL routing          │ Yes              │ Yes                  │ │
│ │ SSL termination      │ Yes              │ Yes                  │ │
│ │ Autoscaling          │ Yes              │ Yes                  │ │
│ │ Zone redundancy      │ Yes              │ Yes                  │ │
│ │ Header rewrite       │ Yes              │ Yes                  │ │
│ │ WAF                  │ No ❌            │ Yes ✅              │ │
│ │ Bot protection       │ No               │ Yes                  │ │
│ │ Custom WAF rules     │ No               │ Yes                  │ │
│ │ Cost (base)          │ ~$175/mo         │ ~$262/mo             │ │
│ │ + per capacity unit  │ + CU usage       │ + CU usage           │ │
│ └──────────────────────┴──────────────────┴──────────────────────┘ │
│                                                                       │
│ ⚠️ v1 SKUs (Standard, WAF) are being deprecated. Use v2 ONLY.      │
│                                                                       │
│ ⚡ WAF_v2 vs Standard_v2: Only difference is WAF.                    │
│    If you need WAF, pay the extra ~$87/month.                      │
│    In AWS, WAF is a SEPARATE service ($5/rule/month extra).       │
│    Azure WAF is BUILT-IN to Application Gateway.                  │
│                                                                       │
│ ⚡ App Gateway is MORE EXPENSIVE than AWS ALB!                       │
│    ALB: ~$22/month base                                             │
│    App GW: ~$175/month base (8x more!)                             │
│    But App GW includes features that cost extra in AWS             │
│    (WAF, sticky sessions, more).                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating Application Gateway (Full Portal Walkthrough)

```
Portal → Application gateways → Create

┌─────────────────────────────────────────────────────────────────┐
│           CREATE APPLICATION GATEWAY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Basics tab ──                                                │
│ Subscription:     [TechCorp Production ▼]                      │
│ Resource group:   [rg-networking-prod ▼]                       │
│ Name:             [appgw-prod]                                  │
│ Region:           [Central India ▼]                            │
│ Tier:             [WAF V2 ▼]                                   │
│                   Options: Standard V2, WAF V2                 │
│ Enable autoscaling: ● Yes  ○ No                               │
│   Min instances:  [2] (keep 2 for HA)                          │
│   Max instances:  [10]                                          │
│   If No:                                                        │
│   Instance count: [2]                                           │
│                                                                   │
│ Availability zone: [Zones 1, 2, 3 ▼] (all zones for HA)      │
│ HTTP/2: ● Enabled  ○ Disabled                                 │
│                                                                   │
│ ⚠️ App GW v2 autoscaling: Scale based on traffic.               │
│    Min=0 is possible but cold start is ~6-8 min!               │
│    Keep min=2 for production.                                   │
│                                                                   │
│ Virtual network:                                                │
│   VNet: [vnet-prod ▼]                                          │
│   Subnet: [subnet-appgw ▼] (DEDICATED subnet!)                │
│                                                                   │
│   ⚠️ App GW REQUIRES its own subnet!                            │
│   ⚠️ Only App Gateways can be in this subnet!                   │
│   ⚠️ Recommended: /24 (256 IPs) or at minimum /26             │
│   ⚠️ No other resources in this subnet (no VMs, no NICs)       │
│   ⚠️ NSG on this subnet must allow:                             │
│      - Inbound: 65200-65535 from GatewayManager (required!)   │
│      - Inbound: 80, 443 from Internet (for public LB)         │
│      - Inbound: AzureLoadBalancer (health probes)              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

── Frontends tab ──
┌─────────────────────────────────────────────────────────────────┐
│ Frontend IP type: ● Public  ○ Private  ○ Both                 │
│                                                                   │
│ Public IP address: [Create new]                                │
│   Name: [pip-appgw-prod]                                       │
│   SKU: Standard                                                │
│                                                                   │
│ Private IP (if Both selected):                                 │
│   IP address: [10.0.4.10] (from appgw subnet)                │
│   Use case: Internal + external access                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

── Backends tab ──
┌─────────────────────────────────────────────────────────────────┐
│ [+ Add a backend pool]                                          │
│                                                                   │
│ Backend pool: [bp-web]                                          │
│ Add targets:                                                    │
│ Target type: [Virtual machine ▼]                               │
│   Options:                                                      │
│   ├── Virtual machine                                          │
│   ├── VMSS (Virtual Machine Scale Set)                         │
│   ├── IP address or FQDN                                      │
│   └── App Service                                              │
│                                                                   │
│ ☑ vm-web-01 (nic-web-01)                                      │
│ ☑ vm-web-02 (nic-web-02)                                      │
│                                                                   │
│ Backend pool: [bp-api]                                          │
│ Target type: [App Service ▼]                                   │
│ ☑ api-prod.azurewebsites.net                                  │
│                                                                   │
│ Backend pool: [bp-static]                                       │
│ Target type: [IP address or FQDN ▼]                           │
│ FQDN: [stprodstatic.blob.core.windows.net]                    │
│                                                                   │
│ ⚡ App Gateway can target App Services directly!                │
│    This is like AWS ALB targeting Lambda, but better —         │
│    App Service is a full web hosting platform.                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

── Configuration tab ──
(This is where listeners + routing rules + backend settings come together)

┌─────────────────────────────────────────────────────────────────┐
│ [+ Add a routing rule]                                          │
│                                                                   │
│ Rule name: [rule-https-main]                                   │
│ Priority: [100]                                                 │
│                                                                   │
│ ── Listener tab ──                                              │
│ Listener name:      [listener-https]                           │
│ Frontend IP:        [Public ▼]                                 │
│ Protocol:           [HTTPS ▼]                                  │
│ Port:               [443]                                       │
│                                                                   │
│ HTTPS settings:                                                 │
│ Certificate:                                                    │
│   ○ Choose a certificate from Key Vault                       │
│   ● Upload a certificate (.pfx)                               │
│   Cert name: [cert-techcorp]                                   │
│   PFX file:  [techcorp.com.pfx]                               │
│   Password:  [••••••••]                                        │
│                                                                   │
│   ⚠️ Unlike AWS ACM (free certs), Azure App GW requires:       │
│      PFX certificate OR Key Vault certificate.                 │
│      You can use Key Vault for auto-rotation.                  │
│      No equivalent to ACM auto-managed certs.                  │
│                                                                   │
│ Listener type:                                                  │
│   ● Basic (single site)                                        │
│   ○ Multi-site (host name routing)                            │
│                                                                   │
│ If multi-site:                                                  │
│   Host type: ● Single  ○ Multiple/Wildcard                    │
│   Host name: [techcorp.com]                                    │
│                                                                   │
│ Error page URL:                                                 │
│   ☑ Custom error pages                                         │
│   403 page: [https://stprod.blob.core.windows.net/errors/403] │
│   502 page: [https://stprod.blob.core.windows.net/errors/502] │
│                                                                   │
│ ── Backend targets tab ──                                       │
│ Target type: ● Backend pool  ○ Redirection                    │
│ Backend target: [bp-web ▼]                                     │
│ Backend settings: [Create new]                                 │
│                                                                   │
│ ┌──────────────────────────────────────────────────────────┐   │
│ │ Backend settings:                                         │   │
│ │ Name:               [settings-web]                       │   │
│ │ Backend protocol:   [HTTP ▼] / HTTPS                    │   │
│ │ Backend port:       [8080]                               │   │
│ │ Cookie-based affinity: ● Disabled  ○ Enabled            │   │
│ │ Connection draining: ☑ Enable                           │   │
│ │   Timeout:          [30] seconds                         │   │
│ │ Request timeout:    [30] seconds                         │   │
│ │                                                           │   │
│ │ Override with new host name: ○ No  ● Yes                │   │
│ │   Host name override:                                    │   │
│ │   ○ Pick host name from backend target                  │   │
│ │   ● Override with specific domain name                  │   │
│ │   Host name: [app-prod.azurewebsites.net]               │   │
│ │                                                           │   │
│ │   ⚠️ Host override is CRITICAL for App Service backends! │   │
│ │      App Service routes by host header. If you don't     │   │
│ │      override, it gets the App GW's IP as host →         │   │
│ │      404 or wrong app!                                   │   │
│ │                                                           │   │
│ │ Custom probe: [Create new]                               │   │
│ │ ┌────────────────────────────────────────────────────┐  │   │
│ │ │ Name:       [probe-web]                            │  │   │
│ │ │ Protocol:   [HTTP ▼]                              │  │   │
│ │ │ Host:       [techcorp.com]                        │  │   │
│ │ │ Path:       [/health]                             │  │   │
│ │ │ Interval:   [30] seconds                          │  │   │
│ │ │ Timeout:    [30] seconds                          │  │   │
│ │ │ Unhealthy threshold: [3]                         │  │   │
│ │ │ Use probe matching conditions: ☑ Yes             │  │   │
│ │ │   HTTP response status: [200-399]                │  │   │
│ │ │   HTTP response body: [Healthy]                  │  │   │
│ │ │                                                    │  │   │
│ │ │ ⚡ Body matching! You can check if response body  │  │   │
│ │ │   contains a specific string — more robust than  │  │   │
│ │ │   just status code.                              │  │   │
│ │ │   AWS ALB doesn't have body matching.            │  │   │
│ │ └────────────────────────────────────────────────────┘  │   │
│ └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Path-Based Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│           PATH-BASED ROUTING                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Equivalent to AWS ALB listener rules with path conditions.         │
│                                                                       │
│ Create routing rule → Backend targets tab → "Add multiple targets  │
│ to create a path-based rule"                                       │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Path                │ Backend pool    │ Backend settings      │   │
│ ├─────────────────────┼─────────────────┼───────────────────────┤   │
│ │ /api/*              │ bp-api          │ settings-api          │   │
│ │ /api/v2/*           │ bp-api-v2       │ settings-api          │   │
│ │ /static/*           │ bp-static       │ settings-static       │   │
│ │ /images/*           │ bp-static       │ settings-static       │   │
│ │ /admin/*            │ bp-admin        │ settings-admin        │   │
│ │ /* (default)        │ bp-web          │ settings-web          │   │
│ └─────────────────────┴─────────────────┴───────────────────────┘   │
│                                                                       │
│ Multi-site (host-based) routing:                                    │
│ Create separate listeners per host, each with its own routing rule│
│                                                                       │
│ Listener: listener-api (api.techcorp.com:443)                     │
│   → Rule: rule-api → bp-api                                       │
│                                                                       │
│ Listener: listener-admin (admin.techcorp.com:443)                 │
│   → Rule: rule-admin → bp-admin                                   │
│                                                                       │
│ Listener: listener-web (techcorp.com:443)                         │
│   → Rule: rule-web → bp-web (with path rules)                    │
│                                                                       │
│ ⚡ Key difference from AWS ALB:                                     │
│    ALB: One listener on 443, multiple rules with host conditions  │
│    App GW: Multiple listeners (one per host), each with rules     │
│    Both achieve the same result, just different structure.         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### WAF (Web Application Firewall)

```
┌─────────────────────────────────────────────────────────────────────┐
│           WAF (BUILT INTO APP GATEWAY)                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚡ In AWS, WAF is a SEPARATE service attached to ALB.                │
│    In Azure, WAF is BUILT-IN to Application Gateway (WAF_v2 SKU). │
│    In GCP, Cloud Armor is attached to backend services.            │
│                                                                       │
│ WAF modes:                                                           │
│ ├── Detection: Logs threats but doesn't block                     │
│ └── Prevention: Blocks threats + logs                              │
│                                                                       │
│ WAF policy:                                                          │
│ Portal → WAF policies → Create                                     │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Name: [waf-policy-prod]                                      │   │
│ │ Mode: ● Prevention  ○ Detection                             │   │
│ │                                                               │   │
│ │ Managed rules:                                                │   │
│ │ ☑ OWASP 3.2 rule set (default)                              │   │
│ │   ├── SQL injection protection                               │   │
│ │   ├── Cross-site scripting (XSS)                             │   │
│ │   ├── Remote code execution                                  │   │
│ │   ├── Local file inclusion                                   │   │
│ │   ├── Request smuggling                                      │   │
│ │   └── Protocol violations                                    │   │
│ │ ☑ Microsoft_BotManagerRuleSet_1.0                            │   │
│ │   ├── Good bots (Googlebot — allowed)                       │   │
│ │   ├── Bad bots (scrapers — blocked)                         │   │
│ │   └── Unknown bots (challenged)                              │   │
│ │                                                               │   │
│ │ Custom rules:                                                 │   │
│ │ [+ Add custom rule]                                          │   │
│ │ Name: [block-bad-countries]                                  │   │
│ │ Priority: [1]                                                │   │
│ │ Type: Match                                                   │   │
│ │ Match variable: [RemoteAddr ▼]                              │   │
│ │ Operator: [GeoMatch ▼]                                      │   │
│ │ Values: [CN, RU, KP]                                         │   │
│ │ Action: [Block ▼]                                            │   │
│ │                                                               │   │
│ │ Custom rule: [rate-limit-api]                                │   │
│ │ Priority: [2]                                                │   │
│ │ Type: Rate limit                                              │   │
│ │ Rate limit threshold: [100] requests per [1 minute]         │   │
│ │ Group by: [Client address ▼]                                │   │
│ │ Conditions: RequestUri contains "/api/"                     │   │
│ │ Action: [Block ▼]                                            │   │
│ │                                                               │   │
│ │ Exclusions (false positive handling):                        │   │
│ │ [+ Add exclusion]                                            │   │
│ │ Match variable: [Request body JSON args names ▼]            │   │
│ │ Operator: [Equals ▼]                                        │   │
│ │ Selector: [content]                                          │   │
│ │ Applies to: [OWASP rule 942130]                             │   │
│ │ (Exclude "content" field from SQL injection check)          │   │
│ │                                                               │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Attach to App Gateway:                                               │
│ App Gateway → WAF → Associate WAF policy → waf-policy-prod        │
│                                                                       │
│ WAF logs:                                                            │
│ ├── Diagnostic settings → Log Analytics / Storage Account         │
│ ├── ApplicationGatewayFirewallLog: All WAF events                 │
│ ├── Blocked requests with rule ID + reason                        │
│ └── KQL queries in Log Analytics for analysis                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Rewrite Rules

```
┌─────────────────────────────────────────────────────────────────────┐
│           REWRITE RULES                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Modify HTTP request/response headers and URLs at the App GW level.│
│ Similar to AWS ALB's limited header modification, but much more   │
│ powerful.                                                           │
│                                                                       │
│ App GW → Rewrites → [+ Rewrite set]                                │
│                                                                       │
│ Common rewrites:                                                     │
│                                                                       │
│ 1. Remove server header (security):                                  │
│    Response header: Delete "Server"                                │
│    (Hides "Microsoft-IIS/10.0" from responses)                    │
│                                                                       │
│ 2. Add security headers:                                             │
│    Response header: Set "X-Content-Type-Options" = "nosniff"      │
│    Response header: Set "X-Frame-Options" = "DENY"                │
│    Response header: Set "Strict-Transport-Security" =             │
│      "max-age=31536000; includeSubDomains"                        │
│                                                                       │
│ 3. URL rewrite:                                                      │
│    Condition: URL path = /old-api/*                                │
│    Action: Rewrite URL to /api/v2/{remaining path}                │
│                                                                       │
│ 4. Add client IP header:                                             │
│    Request header: Set "X-Real-IP" = {var_client_ip}              │
│                                                                       │
│ 5. Redirect HTTP to HTTPS:                                          │
│    (Better to use redirect routing rule, but rewrite also works)  │
│                                                                       │
│ Server variables available:                                          │
│ ├── {var_client_ip}: Client IP address                            │
│ ├── {var_client_port}: Client port                                │
│ ├── {var_host}: Host header                                       │
│ ├── {var_request_uri}: Full request URI                           │
│ ├── {var_uri_path}: URI path                                      │
│ ├── {var_query_string}: Query string                              │
│ └── {http_req_HeaderName}: Any request header value              │
│                                                                       │
│ ⚡ AWS ALB has very limited rewrite capability.                      │
│    Azure App GW and GCP URL maps are much more powerful.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Terraform / Bicep Examples

### Terraform

```hcl
# Azure Load Balancer (Layer 4)
resource "azurerm_public_ip" "lb" {
  name                = "pip-lb-prod"
  location            = azurerm_resource_group.networking.location
  resource_group_name = azurerm_resource_group.networking.name
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]
}

resource "azurerm_lb" "prod" {
  name                = "lb-prod-standard"
  location            = azurerm_resource_group.networking.location
  resource_group_name = azurerm_resource_group.networking.name
  sku                 = "Standard"

  frontend_ip_configuration {
    name                 = "fe-prod-public"
    public_ip_address_id = azurerm_public_ip.lb.id
  }
}

resource "azurerm_lb_backend_address_pool" "web" {
  loadbalancer_id = azurerm_lb.prod.id
  name            = "bp-web-servers"
}

resource "azurerm_lb_probe" "web" {
  loadbalancer_id = azurerm_lb.prod.id
  name            = "probe-web-http"
  protocol        = "Http"
  port            = 8080
  request_path    = "/health"
  interval_in_seconds = 5
  number_of_probes    = 2
}

resource "azurerm_lb_rule" "web" {
  loadbalancer_id                = azurerm_lb.prod.id
  name                           = "rule-web-http"
  protocol                       = "Tcp"
  frontend_port                  = 80
  backend_port                   = 8080
  frontend_ip_configuration_name = "fe-prod-public"
  backend_address_pool_ids       = [azurerm_lb_backend_address_pool.web.id]
  probe_id                       = azurerm_lb_probe.web.id
  idle_timeout_in_minutes        = 4
  enable_tcp_reset               = true
}

resource "azurerm_lb_nat_rule" "ssh" {
  resource_group_name            = azurerm_resource_group.networking.name
  loadbalancer_id                = azurerm_lb.prod.id
  name                           = "nat-ssh"
  protocol                       = "Tcp"
  frontend_port_start            = 50000
  frontend_port_end              = 50010
  backend_port                   = 22
  frontend_ip_configuration_name = "fe-prod-public"
  backend_address_pool_id        = azurerm_lb_backend_address_pool.web.id
}

# Associate VMs with backend pool
resource "azurerm_network_interface_backend_address_pool_association" "web_01" {
  network_interface_id    = azurerm_network_interface.web_01.id
  ip_configuration_name   = "internal"
  backend_address_pool_id = azurerm_lb_backend_address_pool.web.id
}

# Application Gateway (Layer 7)
resource "azurerm_application_gateway" "prod" {
  name                = "appgw-prod"
  location            = azurerm_resource_group.networking.location
  resource_group_name = azurerm_resource_group.networking.name
  zones               = ["1", "2", "3"]

  sku {
    name = "WAF_v2"
    tier = "WAF_v2"
  }

  autoscale_configuration {
    min_capacity = 2
    max_capacity = 10
  }

  gateway_ip_configuration {
    name      = "gw-ip-config"
    subnet_id = azurerm_subnet.appgw.id
  }

  # Frontend
  frontend_ip_configuration {
    name                 = "fe-public"
    public_ip_address_id = azurerm_public_ip.appgw.id
  }

  frontend_port {
    name = "port-443"
    port = 443
  }

  frontend_port {
    name = "port-80"
    port = 80
  }

  # SSL certificate
  ssl_certificate {
    name     = "cert-techcorp"
    data     = filebase64("${path.module}/certs/techcorp.pfx")
    password = var.cert_password
  }

  # Backend pools
  backend_address_pool {
    name = "bp-web"
  }

  backend_address_pool {
    name  = "bp-api"
    fqdns = ["api-prod.azurewebsites.net"]
  }

  # Backend settings
  backend_http_settings {
    name                  = "settings-web"
    cookie_based_affinity = "Disabled"
    port                  = 8080
    protocol              = "Http"
    request_timeout       = 30
    connection_draining {
      enabled           = true
      drain_timeout_sec = 30
    }
    probe_name = "probe-web"
  }

  backend_http_settings {
    name                                = "settings-api"
    cookie_based_affinity               = "Disabled"
    port                                = 443
    protocol                            = "Https"
    request_timeout                     = 30
    pick_host_name_from_backend_address = true
    probe_name                          = "probe-api"
  }

  # Health probes
  probe {
    name                = "probe-web"
    protocol            = "Http"
    host                = "techcorp.com"
    path                = "/health"
    interval            = 30
    timeout             = 30
    unhealthy_threshold = 3
    match {
      status_code = ["200-399"]
      body        = "Healthy"
    }
  }

  probe {
    name                                      = "probe-api"
    protocol                                  = "Https"
    pick_host_name_from_backend_http_settings = true
    path                                      = "/api/health"
    interval                                  = 30
    timeout                                   = 30
    unhealthy_threshold                       = 3
  }

  # HTTPS listener
  http_listener {
    name                           = "listener-https"
    frontend_ip_configuration_name = "fe-public"
    frontend_port_name             = "port-443"
    protocol                       = "Https"
    ssl_certificate_name           = "cert-techcorp"
  }

  # HTTP listener (for redirect)
  http_listener {
    name                           = "listener-http"
    frontend_ip_configuration_name = "fe-public"
    frontend_port_name             = "port-80"
    protocol                       = "Http"
  }

  # Routing rule: HTTPS → path-based
  request_routing_rule {
    name               = "rule-https"
    priority           = 100
    rule_type          = "PathBasedRouting"
    http_listener_name = "listener-https"
    url_path_map_name  = "paths-main"
  }

  # Routing rule: HTTP → HTTPS redirect
  request_routing_rule {
    name                        = "rule-http-redirect"
    priority                    = 200
    rule_type                   = "Basic"
    http_listener_name          = "listener-http"
    redirect_configuration_name = "redirect-to-https"
  }

  # URL path map
  url_path_map {
    name                               = "paths-main"
    default_backend_address_pool_name  = "bp-web"
    default_backend_http_settings_name = "settings-web"

    path_rule {
      name                       = "api"
      paths                      = ["/api/*"]
      backend_address_pool_name  = "bp-api"
      backend_http_settings_name = "settings-api"
    }
  }

  # Redirect
  redirect_configuration {
    name                 = "redirect-to-https"
    redirect_type        = "Permanent"
    target_listener_name = "listener-https"
    include_path         = true
    include_query_string = true
  }

  # WAF
  waf_configuration {
    enabled          = true
    firewall_mode    = "Prevention"
    rule_set_type    = "OWASP"
    rule_set_version = "3.2"
  }

  # Rewrite rules
  rewrite_rule_set {
    name = "security-headers"

    rewrite_rule {
      name          = "add-security-headers"
      rule_sequence = 100

      response_header_configuration {
        header_name  = "X-Content-Type-Options"
        header_value = "nosniff"
      }
      response_header_configuration {
        header_name  = "X-Frame-Options"
        header_value = "DENY"
      }
      response_header_configuration {
        header_name  = "Server"
        header_value = ""
      }
    }
  }
}
```

### Bicep

```bicep
// Azure Load Balancer
resource loadBalancer 'Microsoft.Network/loadBalancers@2023-11-01' = {
  name: 'lb-prod-standard'
  location: location
  sku: {
    name: 'Standard'
    tier: 'Regional'
  }
  properties: {
    frontendIPConfigurations: [
      {
        name: 'fe-prod-public'
        properties: {
          publicIPAddress: {
            id: publicIp.id
          }
        }
      }
    ]
    backendAddressPools: [
      {
        name: 'bp-web-servers'
      }
    ]
    probes: [
      {
        name: 'probe-web-http'
        properties: {
          protocol: 'Http'
          port: 8080
          requestPath: '/health'
          intervalInSeconds: 5
          numberOfProbes: 2
        }
      }
    ]
    loadBalancingRules: [
      {
        name: 'rule-web-http'
        properties: {
          frontendIPConfiguration: {
            id: resourceId('Microsoft.Network/loadBalancers/frontendIPConfigurations', 'lb-prod-standard', 'fe-prod-public')
          }
          backendAddressPool: {
            id: resourceId('Microsoft.Network/loadBalancers/backendAddressPools', 'lb-prod-standard', 'bp-web-servers')
          }
          probe: {
            id: resourceId('Microsoft.Network/loadBalancers/probes', 'lb-prod-standard', 'probe-web-http')
          }
          protocol: 'Tcp'
          frontendPort: 80
          backendPort: 8080
          idleTimeoutInMinutes: 4
          enableTcpReset: true
        }
      }
    ]
  }
}

// Application Gateway (simplified — key parts)
resource appGateway 'Microsoft.Network/applicationGateways@2023-11-01' = {
  name: 'appgw-prod'
  location: location
  zones: ['1', '2', '3']
  properties: {
    sku: {
      name: 'WAF_v2'
      tier: 'WAF_v2'
    }
    autoscaleConfiguration: {
      minCapacity: 2
      maxCapacity: 10
    }
    gatewayIPConfigurations: [
      {
        name: 'gw-ip-config'
        properties: {
          subnet: { id: subnetAppGw.id }
        }
      }
    ]
    frontendIPConfigurations: [
      {
        name: 'fe-public'
        properties: {
          publicIPAddress: { id: publicIpAppGw.id }
        }
      }
    ]
    frontendPorts: [
      { name: 'port-443', properties: { port: 443 } }
      { name: 'port-80', properties: { port: 80 } }
    ]
    backendAddressPools: [
      { name: 'bp-web', properties: {} }
      { name: 'bp-api', properties: { backendAddresses: [{ fqdn: 'api-prod.azurewebsites.net' }] } }
    ]
    backendHttpSettingsCollection: [
      {
        name: 'settings-web'
        properties: {
          port: 8080
          protocol: 'Http'
          requestTimeout: 30
          cookieBasedAffinity: 'Disabled'
        }
      }
    ]
    httpListeners: [
      {
        name: 'listener-https'
        properties: {
          frontendIPConfiguration: { id: resourceId('Microsoft.Network/applicationGateways/frontendIPConfigurations', 'appgw-prod', 'fe-public') }
          frontendPort: { id: resourceId('Microsoft.Network/applicationGateways/frontendPorts', 'appgw-prod', 'port-443') }
          protocol: 'Https'
          sslCertificate: { id: resourceId('Microsoft.Network/applicationGateways/sslCertificates', 'appgw-prod', 'cert-techcorp') }
        }
      }
    ]
    requestRoutingRules: [
      {
        name: 'rule-https'
        properties: {
          priority: 100
          ruleType: 'PathBasedRouting'
          httpListener: { id: resourceId('Microsoft.Network/applicationGateways/httpListeners', 'appgw-prod', 'listener-https') }
          urlPathMap: { id: resourceId('Microsoft.Network/applicationGateways/urlPathMaps', 'appgw-prod', 'paths-main') }
        }
      }
    ]
    webApplicationFirewallConfiguration: {
      enabled: true
      firewallMode: 'Prevention'
      ruleSetType: 'OWASP'
      ruleSetVersion: '3.2'
    }
  }
}
```

---

## Part 5: az CLI Examples

```bash
# Create public IP for LB
az network public-ip create \
  --resource-group rg-networking-prod \
  --name pip-lb-prod \
  --sku Standard \
  --zone 1 2 3

# Create Load Balancer
az network lb create \
  --resource-group rg-networking-prod \
  --name lb-prod-standard \
  --sku Standard \
  --frontend-ip-name fe-prod-public \
  --public-ip-address pip-lb-prod \
  --backend-pool-name bp-web-servers

# Add health probe
az network lb probe create \
  --resource-group rg-networking-prod \
  --lb-name lb-prod-standard \
  --name probe-web-http \
  --protocol Http \
  --port 8080 \
  --path /health \
  --interval 5 \
  --threshold 2

# Add LB rule
az network lb rule create \
  --resource-group rg-networking-prod \
  --lb-name lb-prod-standard \
  --name rule-web-http \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 8080 \
  --frontend-ip-name fe-prod-public \
  --backend-pool-name bp-web-servers \
  --probe-name probe-web-http \
  --idle-timeout 4 \
  --enable-tcp-reset true

# Add NAT rule (SSH access)
az network lb inbound-nat-rule create \
  --resource-group rg-networking-prod \
  --lb-name lb-prod-standard \
  --name nat-ssh-web01 \
  --protocol Tcp \
  --frontend-port 50000 \
  --backend-port 22 \
  --frontend-ip-name fe-prod-public

# Add VM NIC to backend pool
az network nic ip-config address-pool add \
  --resource-group rg-networking-prod \
  --nic-name nic-web-01 \
  --ip-config-name ipconfig1 \
  --lb-name lb-prod-standard \
  --address-pool bp-web-servers

# Create Application Gateway (basic)
az network application-gateway create \
  --resource-group rg-networking-prod \
  --name appgw-prod \
  --sku WAF_v2 \
  --capacity 2 \
  --vnet-name vnet-prod \
  --subnet subnet-appgw \
  --public-ip-address pip-appgw-prod \
  --http-settings-port 8080 \
  --http-settings-protocol Http \
  --frontend-port 443 \
  --routing-rule-type PathBasedRouting

# Add URL path map
az network application-gateway url-path-map create \
  --resource-group rg-networking-prod \
  --gateway-name appgw-prod \
  --name paths-main \
  --paths /api/* \
  --address-pool bp-api \
  --http-settings settings-api \
  --default-address-pool bp-web \
  --default-http-settings settings-web

# Enable WAF
az network application-gateway waf-config set \
  --resource-group rg-networking-prod \
  --gateway-name appgw-prod \
  --enabled true \
  --firewall-mode Prevention \
  --rule-set-version 3.2
```

---

## Part 6: Real-World Patterns

### Startup

```
Load balancing: 1 Application Gateway (WAF_v2)

Setup:
├── Frontend: Public IP, HTTPS on 443
├── SSL: PFX cert from Let's Encrypt (uploaded)
├── Backend pool: 2 VMs (or App Service)
├── Path routing: /api/* → App Service, /* → VMs
├── WAF: Prevention mode, OWASP 3.2
├── HTTP → HTTPS redirect rule
├── Autoscale: min=2, max=5
└── DNS: A record → App GW public IP

No Azure Load Balancer needed (all HTTP traffic).
Cost: ~$175/month (App GW) + capacity units

Or cheaper: Just App Service with built-in LB (no App GW)
Cost: ~$55/month (App Service plan)
```

### Mid-Size

```
Load balancing:
├── 1 Application Gateway (WAF_v2) — public web/API
├── 1 Azure Load Balancer (internal) — database tier

Application Gateway:
├── Multi-site listeners:
│   ├── techcorp.com:443 → bp-web (VMSS)
│   ├── api.techcorp.com:443 → bp-api (App Service)
│   └── admin.techcorp.com:443 → bp-admin (VMs)
├── Path-based routing per site
├── WAF: Prevention + custom rate limiting rules
├── Rewrite rules: Security headers, server header removal
├── Cookie-based affinity for web app
├── Custom health probes with body matching
└── Autoscale: min=2, max=10

Internal LB:
├── SQL Server AlwaysOn AG (HA ports + floating IP)
├── Health probe: TCP 1433
└── Backend pool: 2 SQL VMs

Cost: ~$250-400/month
```

### Enterprise

```
Load balancing:
├── Azure Front Door (global) — Chapter 10
│   ├── Global HTTP routing + CDN + WAF
│   ├── Multi-region backends
│   └── DDoS protection
│
├── Application Gateway (per region)
│   ├── WAF_v2 with managed + custom rules
│   ├── 20+ listeners (multi-site)
│   ├── Path-based routing per microservice
│   ├── URL rewrite rules
│   ├── Custom error pages
│   ├── Key Vault integration for SSL certs
│   ├── Private Link for backend App Services
│   └── Autoscale: min=3, max=20
│
├── Azure Load Balancer (public) — non-HTTP
│   ├── Static IP for partner allowlisting
│   ├── TCP services (SFTP, custom protocols)
│   └── Outbound rules with dedicated SNAT ports
│
├── Azure Load Balancer (internal) — per tier
│   ├── Database tier: SQL AlwaysOn (HA ports)
│   ├── Middleware tier: Message queues, cache
│   ├── Cross-region LB: Active-active across regions
│   └── NVA tier: Firewall VMs (HA ports)

Monitoring:
├── App GW diagnostics → Log Analytics
│   ├── Access logs
│   ├── Performance logs
│   ├── WAF logs
│   └── KQL dashboards
├── LB metrics → Azure Monitor
│   ├── Health probe status per backend
│   ├── SNAT connection count
│   ├── Byte/packet counts
│   └── Alerts on probe failures
└── Azure Workbooks for executive dashboards

Cost: ~$500-2000/month (multiple LBs + App GWs)
```

---

## Quick Reference: Azure vs AWS vs GCP

| Feature | Azure LB | Azure App GW | AWS ALB | AWS NLB | GCP App LB |
|---------|----------|--------------|---------|---------|-------------|
| Layer | 4 | 7 | 7 | 4 | 7 |
| Scope | Regional | Regional | Regional | Regional | Global |
| URL routing | No | Yes | Yes | No | Yes |
| WAF | No | Built-in | Separate | No | Separate |
| SSL termination | No | Yes | Yes | Yes | Yes |
| Static IP | Yes | No (v2) | No | Yes | Yes (Anycast) |
| HA ports | Yes | No | No | No | Yes |
| Health body match | No | Yes | No | No | No |
| Autoscaling | N/A | Yes (v2) | Auto | Auto | Auto |
| NAT rules | Yes | No | No | No | No |
| SNAT outbound | Yes | No | No | No | No |
| Cost (base) | ~$18/mo | ~$175/mo | ~$22/mo | ~$22/mo | ~$20/mo |

---

## What's Next?

In the next chapter, we'll cover Virtual Machine Scale Sets (VMSS) — Azure's auto-scaling solution for VMs.

→ Next: [Chapter 12: Application Gateway](12-application-gateway.md)

---

*Last Updated: May 2026*
