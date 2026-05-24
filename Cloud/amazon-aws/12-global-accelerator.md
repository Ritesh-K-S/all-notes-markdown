# Chapter 12: AWS Global Accelerator

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Why Global Accelerator?](#part-1-why-global-accelerator)
- [Part 2: Architecture](#part-2-architecture)
- [Part 3: Creating a Global Accelerator (Full Portal Walkthrough)](#part-3-creating-a-global-accelerator-full-portal-walkthrough)
- [Part 4: Standard vs Custom Routing](#part-4-standard-vs-custom-routing)
- [Part 5: Traffic Dials & Weights](#part-5-traffic-dials--weights)
- [Part 6: Failover & Health Checks](#part-6-failover--health-checks)
- [Part 7: Endpoint Types & Client IP Preservation](#part-7-endpoint-types--client-ip-preservation)
- [Part 8: Monitoring & Flow Logs](#part-8-monitoring--flow-logs)
- [Part 9: Terraform Example](#part-9-terraform-example)
- [Part 10: Real-World Patterns](#part-10-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Global Accelerator? How is it Different from CloudFront?

> **Real-World Analogy:** Imagine flying from Mumbai to New York. The public internet is like a budget airline with 3 layovers through random cities. Global Accelerator is like a direct flight on a private jet — you board at the nearest airport (AWS edge location) and fly the private highway (AWS backbone network) straight to your destination.

**Why does this matter?** For latency-sensitive, non-cacheable traffic (like gaming, VoIP, APIs with unique responses), CloudFront can't help because there's nothing to cache. Global Accelerator speeds up **every request** by routing through AWS's private network instead of the unpredictable public internet.

> 💡 **Anycast** = the same IP address is advertised from multiple locations worldwide; your traffic is automatically routed to the nearest one.

AWS Global Accelerator is a networking service that improves the performance and availability of your applications by routing traffic through AWS's global network instead of the public internet. It gives you two static Anycast IP addresses that act as a fixed entry point to your application endpoints in one or more AWS regions.

```
What you'll learn:
├── What is Global Accelerator & why use it
├── Global Accelerator vs CloudFront
├── Architecture (Accelerator → Listeners → Endpoint Groups → Endpoints)
├── Creating a Global Accelerator (full walkthrough)
├── Listeners (TCP/UDP)
├── Endpoint Groups (per region)
│   ├── Traffic dials (% traffic to a region)
│   └── Health checks
├── Endpoints
│   ├── ALB, NLB, EC2, Elastic IP
│   └── Endpoint weights
├── Client Affinity
├── Custom routing accelerators
├── Monitoring & Flow Logs
├── Terraform examples
└── Real-world patterns
```

---

## Part 1: Why Global Accelerator?

```
┌─────────────────────────────────────────────────────────────────────┐
│           THE PROBLEM                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Without Global Accelerator:                                         │
│                                                                       │
│ User (India) ──► Public Internet ──► [Many hops] ──► App (US)     │
│                                                                       │
│ Public internet routing:                                             │
│ ├── Unpredictable latency (varies by ISP, congestion)             │
│ ├── Multiple hops across ISPs                                      │
│ ├── Packets may take suboptimal paths                              │
│ ├── No SLA on internet backbone                                    │
│ └── Higher jitter, more packet loss                                │
│                                                                       │
│ With Global Accelerator:                                             │
│                                                                       │
│ User (India) ──► Nearest AWS Edge ──► AWS Backbone ──► App (US)  │
│                                                                       │
│ AWS backbone routing:                                                │
│ ├── Traffic enters AWS network at nearest edge location            │
│ ├── Travels on AWS's private fiber backbone                       │
│ ├── Consistent, low-latency path                                   │
│ ├── 60% better performance in many cases                           │
│ └── AWS controls the entire path                                   │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │                                                              │   │
│ │   User (Mumbai)                                              │   │
│ │     │                                                        │   │
│ │     │ (short hop to nearest edge)                            │   │
│ │     ▼                                                        │   │
│ │   ┌──────────────────┐                                       │   │
│ │   │ AWS Edge Location │ (Mumbai)                             │   │
│ │   │ (Anycast IP)      │                                      │   │
│ │   └────────┬─────────┘                                       │   │
│ │            │                                                  │   │
│ │            │ AWS Private Backbone (fast, reliable)           │   │
│ │            │                                                  │   │
│ │   ┌────────▼─────────┐                                       │   │
│ │   │ AWS Region (US)   │                                      │   │
│ │   │ ALB/NLB/EC2       │                                      │   │
│ │   └──────────────────┘                                       │   │
│ │                                                              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Global Accelerator vs CloudFront

```
┌─────────────────────────────────────────────────────────────────────┐
│           GLOBAL ACCELERATOR vs CLOUDFRONT                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Both use AWS edge locations. But for VERY different purposes.      │
│                                                                       │
│ ┌────────────────────┬───────────────────┬─────────────────────┐   │
│ │ Feature            │ Global Accelerator│ CloudFront           │   │
│ ├────────────────────┼───────────────────┼─────────────────────┤   │
│ │ Purpose            │ Network perf +    │ Content delivery    │   │
│ │                    │ availability      │ (CDN, caching)      │   │
│ │ Layer              │ 4 (TCP/UDP)       │ 7 (HTTP/HTTPS)      │   │
│ │ Caching            │ No ❌             │ Yes ✅              │   │
│ │ Static IPs         │ Yes ✅ (2 IPs)   │ No ❌               │   │
│ │ Protocols          │ TCP, UDP          │ HTTP, HTTPS,        │   │
│ │                    │                   │ WebSocket            │   │
│ │ Use for            │ Gaming, VoIP,     │ Websites, APIs,     │   │
│ │                    │ IoT, non-HTTP,    │ video streaming,    │   │
│ │                    │ failover          │ static assets        │   │
│ │ Health checks      │ Yes (instant      │ No (origin only)    │   │
│ │                    │ failover)         │                      │   │
│ │ Multi-region       │ Active-active/    │ Origin groups       │   │
│ │ failover           │ active-passive    │ (failover only)     │   │
│ │ Traffic dials      │ Yes (% per region)│ No                  │   │
│ │ Client affinity    │ Yes (source IP)   │ Cookies             │   │
│ │ DDoS protection    │ AWS Shield Adv    │ AWS Shield Adv      │   │
│ │ Cost               │ $0.025/hr +       │ Per request +       │   │
│ │                    │ DT premium        │ data transfer        │   │
│ └────────────────────┴───────────────────┴─────────────────────┘   │
│                                                                       │
│ Decision:                                                            │
│ ├── HTTP content delivery/caching → CloudFront                    │
│ ├── Non-HTTP (TCP/UDP, gaming, IoT) → Global Accelerator          │
│ ├── Need static IPs → Global Accelerator                          │
│ ├── Multi-region failover → Global Accelerator                    │
│ ├── Both → CloudFront for web + Global Accelerator for TCP/UDP   │
│ └── Just better performance for HTTP → Usually CloudFront is     │
│     enough (it also uses AWS backbone)                             │
│                                                                       │
│ ⚡ CloudFront ALSO routes through AWS backbone.                      │
│    Global Accelerator is mainly for TCP/UDP, static IPs,          │
│    and multi-region active-active with instant failover.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│           GLOBAL ACCELERATOR ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┐                                                │
│ │ ACCELERATOR       │  Top-level resource                           │
│ │ (2 static IPs)   │  1.2.3.4 and 5.6.7.8 (Anycast)             │
│ └──────┬───────────┘                                                │
│        │                                                             │
│ ┌──────▼───────────┐                                                │
│ │ LISTENER          │  Protocol + port range                        │
│ │ (TCP:443)         │  What traffic to accept                      │
│ │ (UDP:9000-9100)   │  One or more listeners per accelerator      │
│ └──────┬───────────┘                                                │
│        │                                                             │
│ ┌──────▼───────────┐  ┌──────────────────┐                         │
│ │ ENDPOINT GROUP    │  │ ENDPOINT GROUP    │                        │
│ │ (us-east-1)       │  │ (ap-south-1)      │                        │
│ │ Traffic dial: 70% │  │ Traffic dial: 30% │                        │
│ │ Health check: ✅  │  │ Health check: ✅  │                        │
│ └──────┬───────────┘  └──────┬───────────┘                         │
│        │                      │                                      │
│ ┌──────▼───────────┐  ┌──────▼───────────┐                         │
│ │ ENDPOINTS         │  │ ENDPOINTS         │                        │
│ │ ├── ALB (w:128)  │  │ ├── ALB (w:128)  │                        │
│ │ ├── NLB (w:128)  │  │ └── EC2 (w:64)   │                        │
│ │ └── EIP (w:64)   │  │                    │                        │
│ └──────────────────┘  └──────────────────┘                         │
│                                                                       │
│ Hierarchy:                                                           │
│ Accelerator                                                          │
│ └── Listener (protocol + port)                                     │
│     ├── Endpoint Group (region, traffic dial, health check)       │
│     │   ├── Endpoint (ALB/NLB/EC2/EIP, weight)                   │
│     │   └── Endpoint (ALB/NLB/EC2/EIP, weight)                   │
│     └── Endpoint Group (another region)                            │
│         └── Endpoint (ALB/NLB/EC2/EIP, weight)                    │
│                                                                       │
│ Traffic flow:                                                        │
│ Client → Anycast IP → Nearest edge → AWS backbone →              │
│ Endpoint group (by region proximity + traffic dial) →             │
│ Endpoint (by weight) → Your app                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Creating a Global Accelerator (Full Portal Walkthrough)

```
Console → Global Accelerator → Create accelerator

┌─────────────────────────────────────────────────────────────────┐
│           CREATE ACCELERATOR                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Step 1: Name your accelerator ──                             │
│ Accelerator type:                                               │
│   ● Standard    (most common — routes to optimal endpoint)     │
│   ○ Custom routing (you control which endpoint gets traffic)   │
│                                                                   │
│   Standard: GA picks best endpoint based on health, proximity  │
│   Custom: You map traffic to specific endpoints (gaming, VoIP) │
│                                                                   │
│ Name: [ga-prod-global]                                          │
│                                                                   │
│ IP address type: ● IPv4  ○ Dual-stack                          │
│                                                                   │
│ IP addresses:                                                    │
│   ○ Provide IP addresses from your IP address pool             │
│   ● Use AWS-provided IP addresses                              │
│                                                                   │
│   ⚡ You get 2 static Anycast IPv4 addresses.                    │
│   ⚡ These IPs NEVER change (even during failover).              │
│   ⚡ Share with partners for allowlisting!                       │
│                                                                   │
│ [Next]                                                           │
│                                                                   │
│ ── Step 2: Add listeners ──                                     │
│ [Add listener]                                                   │
│                                                                   │
│ Ports:                                                           │
│   From port: [443]   To port: [443]                            │
│   [Add port range]                                              │
│   From port: [80]    To port: [80]                             │
│                                                                   │
│ Protocol: ● TCP  ○ UDP                                         │
│                                                                   │
│ Client affinity:                                                │
│   [None ▼]                                                     │
│   ├── None: Each request can go to any endpoint                │
│   ├── Source IP: Same client IP → same endpoint                │
│   └── ⚠️ Use Source IP for stateful apps (WebSocket, gaming)    │
│                                                                   │
│ [Next]                                                           │
│                                                                   │
│ ── Step 3: Add endpoint groups ──                               │
│ [Add endpoint group]                                            │
│                                                                   │
│ Region: [ap-south-1 (Mumbai) ▼]                                │
│                                                                   │
│ Traffic dial: [100] %                                           │
│   Range: 0-100%                                                 │
│   100% = full traffic to this region                           │
│   50% = half the traffic that would normally go here           │
│   0% = no traffic (drain this region)                          │
│                                                                   │
│   ⚡ Traffic dials are AMAZING for migrations!                   │
│   Start at 10%, monitor, increase to 50%, then 100%.          │
│   Instant rollback: Set to 0%.                                 │
│                                                                   │
│ Health check:                                                    │
│   Port: [8080]                                                  │
│   Protocol: [HTTP ▼] / TCP                                     │
│   Path: [/health] (HTTP only)                                  │
│   Interval: [30] seconds (10 or 30)                            │
│   Threshold: [3] consecutive health checks                     │
│                                                                   │
│   ⚠️ These health checks are for the ENDPOINT GROUP.            │
│      If all endpoints in a group fail → traffic moves to      │
│      the next nearest healthy group (instant failover!).      │
│                                                                   │
│ [Add endpoint group] (second region)                            │
│ Region: [us-east-1 (N. Virginia) ▼]                            │
│ Traffic dial: [100] %                                           │
│ Health check: same settings                                     │
│                                                                   │
│ [Next]                                                           │
│                                                                   │
│ ── Step 4: Add endpoints ──                                     │
│                                                                   │
│ For endpoint group: ap-south-1                                  │
│ [Add endpoint]                                                   │
│   Endpoint type:                                                │
│   ○ Application Load Balancer                                  │
│   ○ Network Load Balancer                                      │
│   ○ EC2 instance                                               │
│   ○ Elastic IP address                                         │
│   ● Application Load Balancer                                  │
│                                                                   │
│   Endpoint: [alb-prod-mumbai ▼]                                │
│   Weight: [128]                                                 │
│     Range: 0-255                                                │
│     128 = default (equal weight)                               │
│     255 = maximum traffic                                      │
│     0 = no traffic to this endpoint                            │
│                                                                   │
│   Preserve client IP: ☑                                        │
│     ⚡ ALB/NLB target sees real client IP!                      │
│     Without: Target sees Global Accelerator IP.                │
│     With: X-Forwarded-For has real client IP.                  │
│     ⚠️ Only for ALB and EC2 endpoints.                          │
│     ⚠️ NLB always preserves client IP.                          │
│                                                                   │
│ For endpoint group: us-east-1                                   │
│ [Add endpoint]                                                   │
│   Type: Application Load Balancer                               │
│   Endpoint: [alb-prod-virginia ▼]                              │
│   Weight: [128]                                                 │
│                                                                   │
│ [Create accelerator]                                             │
│                                                                   │
│ After creation:                                                  │
│ Static IPs:                                                      │
│   75.2.xxx.xxx (Anycast)                                       │
│   99.83.xxx.xxx (Anycast)                                      │
│ DNS: a1b2c3d4e5f6.awsglobalaccelerator.com                    │
│ Status: IN_PROGRESS → DEPLOYED (takes a few minutes)          │
│                                                                   │
│ ⚡ Point your DNS to these static IPs!                            │
│    api.techcorp.com → A record → 75.2.xxx.xxx                  │
│    api.techcorp.com → A record → 99.83.xxx.xxx                 │
│    Or CNAME → a1b2c3d4e5f6.awsglobalaccelerator.com           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Standard vs Custom Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│           STANDARD vs CUSTOM ROUTING                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ STANDARD ACCELERATOR:                                                │
│ ├── GA decides which endpoint receives traffic                    │
│ ├── Based on: health, proximity, traffic dials, weights           │
│ ├── Automatic failover between endpoints/regions                  │
│ ├── Good for: Web apps, APIs, most applications                  │
│ └── You control: traffic dials, weights, health checks            │
│                                                                       │
│ CUSTOM ROUTING ACCELERATOR:                                          │
│ ├── YOU decide which endpoint receives which traffic              │
│ ├── Maps port ranges to specific EC2 destinations                 │
│ ├── Deterministic routing (same port → same destination)         │
│ ├── Good for: Gaming (match each player to specific server),     │
│ │   VoIP, media workflows, custom application logic              │
│ └── Endpoint group → subnet → GA maps port to each EC2 instance │
│                                                                       │
│ Custom routing example (gaming):                                     │
│                                                                       │
│ Listener: UDP 10000-20000                                           │
│ Endpoint group: us-east-1, subnet: subnet-game-servers             │
│ GA auto-maps:                                                        │
│ ├── Port 10000-10099 → EC2 game-server-1                         │
│ ├── Port 10100-10199 → EC2 game-server-2                         │
│ ├── Port 10200-10299 → EC2 game-server-3                         │
│ └── ...                                                            │
│                                                                       │
│ Your matchmaker service:                                             │
│ 1. Find available game server (e.g., game-server-2)               │
│ 2. Look up GA port mapping → port 10150                           │
│ 3. Tell client: "connect to 75.2.xxx.xxx:10150"                  │
│ 4. Client connects → GA routes to game-server-2 directly        │
│                                                                       │
│ ⚠️ Custom routing only supports EC2 instances as endpoints.        │
│    Not ALB, NLB, or EIP.                                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Traffic Dials & Weights

```
┌─────────────────────────────────────────────────────────────────────┐
│           TRAFFIC DIALS vs ENDPOINT WEIGHTS                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ These are TWO different controls at TWO different levels:           │
│                                                                       │
│ TRAFFIC DIAL (Endpoint Group level):                                 │
│ ├── Controls % of traffic that goes to a REGION                   │
│ ├── Range: 0-100%                                                  │
│ ├── Applies to all endpoints in that group                        │
│ └── Use for: Blue/green region migration, regional traffic control│
│                                                                       │
│ ENDPOINT WEIGHT (Endpoint level):                                    │
│ ├── Controls % of traffic within an endpoint group                │
│ ├── Range: 0-255 (proportional)                                   │
│ ├── Applies to individual endpoints                                │
│ └── Use for: Canary between ALBs, gradual endpoint rollout        │
│                                                                       │
│ Example:                                                             │
│                                                                       │
│ ┌───────────────────────────────────────────────────────────┐      │
│ │ Accelerator (100% total traffic)                          │      │
│ │                                                           │      │
│ │ Endpoint Group: Mumbai (Traffic Dial: 70%)               │      │
│ │ ├── ALB-Mumbai-A  (weight 200) → 200/300 = 67% of 70%  │      │
│ │ └── ALB-Mumbai-B  (weight 100) → 100/300 = 33% of 70%  │      │
│ │                                                           │      │
│ │ Endpoint Group: Virginia (Traffic Dial: 30%)             │      │
│ │ └── ALB-Virginia   (weight 128) → 100% of 30%           │      │
│ │                                                           │      │
│ │ Final distribution:                                       │      │
│ │ ALB-Mumbai-A:  70% × 67% = ~47% of total traffic        │      │
│ │ ALB-Mumbai-B:  70% × 33% = ~23% of total traffic        │      │
│ │ ALB-Virginia:  30% × 100% = 30% of total traffic        │      │
│ └───────────────────────────────────────────────────────────┘      │
│                                                                       │
│ Blue/Green with traffic dials:                                       │
│                                                                       │
│ Step 1: Mumbai=100%, Virginia=0% (all traffic to Mumbai)           │
│ Step 2: Mumbai=90%, Virginia=10% (canary to Virginia)              │
│ Step 3: Mumbai=50%, Virginia=50% (split)                           │
│ Step 4: Mumbai=0%, Virginia=100% (fully migrated)                  │
│                                                                       │
│ ⚡ Changes take effect in seconds! No DNS propagation delays.       │
│    This is MUCH faster than Route 53 DNS-based failover.          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Failover & Health Checks

```
┌─────────────────────────────────────────────────────────────────────┐
│           FAILOVER                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Global Accelerator provides INSTANT failover (<30 seconds):        │
│                                                                       │
│ Scenario 1: Endpoint fails                                          │
│ ├── ALB-Mumbai-A goes unhealthy                                   │
│ ├── GA detects via health check (within 10-30 seconds)           │
│ ├── Traffic shifts to ALB-Mumbai-B (within same endpoint group)  │
│ └── No DNS change, no client-side change                          │
│                                                                       │
│ Scenario 2: Entire region fails                                     │
│ ├── All endpoints in Mumbai endpoint group unhealthy              │
│ ├── GA detects within 10-30 seconds                               │
│ ├── Traffic shifts to Virginia endpoint group (next nearest)      │
│ └── Again, no DNS change — same static IPs work                  │
│                                                                       │
│ Scenario 3: Planned maintenance                                     │
│ ├── Set Mumbai traffic dial to 0%                                 │
│ ├── All traffic instantly goes to Virginia                        │
│ ├── Do maintenance on Mumbai                                      │
│ ├── Set Mumbai traffic dial back to 100%                          │
│ └── Traffic flows back                                             │
│                                                                       │
│ Health check flow:                                                   │
│                                                                       │
│ GA → Health check (HTTP GET /health on port 8080) every 10/30s   │
│ ├── 200 OK → healthy                                              │
│ ├── Non-200 or timeout → count as failure                        │
│ ├── After [threshold] consecutive failures → unhealthy           │
│ └── After [threshold] consecutive successes → healthy again      │
│                                                                       │
│ ⚠️ GA health checks are IN ADDITION to ALB/NLB health checks.     │
│    GA checks endpoint group health (macro).                       │
│    ALB checks individual targets health (micro).                  │
│                                                                       │
│ Compare failover speeds:                                             │
│ ┌────────────────────────────┬─────────────────────────┐          │
│ │ Service                    │ Failover time            │          │
│ ├────────────────────────────┼─────────────────────────┤          │
│ │ Route 53 DNS failover      │ 30-120 seconds (TTL)    │          │
│ │ Global Accelerator          │ <30 seconds             │          │
│ │ CloudFront origin failover │ Immediate (per request) │          │
│ └────────────────────────────┴─────────────────────────┘          │
│                                                                       │
│ ⚡ GA failover is faster than DNS because it doesn't depend on     │
│    DNS TTL expiry or client DNS cache.                              │
│    Same IPs, traffic just routes differently at network level.    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Endpoint Types & Client IP Preservation

```
┌─────────────────────────────────────────────────────────────────────┐
│           ENDPOINT TYPES                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┬──────────────────────────────────────────┐    │
│ │ Endpoint Type    │ Notes                                    │    │
│ ├──────────────────┼──────────────────────────────────────────┤    │
│ │ ALB              │ Most common. Can be internet-facing or   │    │
│ │                  │ internal. Client IP via X-Forwarded-For. │    │
│ │                  │ Can preserve client IP.                  │    │
│ │                  │                                          │    │
│ │ NLB              │ For TCP/UDP. Always preserves client IP. │    │
│ │                  │ Can be internet-facing or internal.      │    │
│ │                  │                                          │    │
│ │ EC2 instance     │ Direct to instance (needs public IP or   │    │
│ │                  │ EIP). Can preserve client IP.            │    │
│ │                  │                                          │    │
│ │ Elastic IP       │ Points to an EIP (and its associated     │    │
│ │                  │ EC2 instance or ENI).                    │    │
│ └──────────────────┴──────────────────────────────────────────┘    │
│                                                                       │
│ Client IP preservation:                                              │
│                                                                       │
│ Without preservation:                                                │
│ Client (1.2.3.4) → GA → ALB → Target sees GA IP (internal)      │
│ X-Forwarded-For: <GA internal IP>                                  │
│                                                                       │
│ With preservation enabled:                                           │
│ Client (1.2.3.4) → GA → ALB → Target sees real client IP        │
│ X-Forwarded-For: 1.2.3.4                                           │
│                                                                       │
│ ⚠️ When enabling client IP preservation on ALB:                     │
│    ALB security group must allow traffic from client IPs           │
│    (not just GA IPs). Use 0.0.0.0/0 or known CIDR ranges.        │
│                                                                       │
│ Cross-account endpoints:                                             │
│ ├── GA in Account A can point to ALB in Account B                 │
│ ├── Requires resource-based policy on the ALB                     │
│ └── Great for shared services or multi-account architectures     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Monitoring & Flow Logs

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ CloudWatch Metrics:                                                  │
│ ├── NewFlowCount: New connections per minute                      │
│ ├── ProcessedBytesIn/Out: Bytes in/out                            │
│ ├── ConsumedLCUs: Capacity consumption                            │
│ └── HealthyEndpointCount: Per endpoint group                     │
│                                                                       │
│ Flow Logs:                                                           │
│ ├── Detailed per-connection logs                                  │
│ ├── Source/destination IP and port                                │
│ ├── Protocol, bytes transferred                                   │
│ ├── Stored in S3                                                  │
│ ├── Enable: Accelerator → Flow logs → Enable                    │
│ │   S3 bucket: [s3://ga-flow-logs-prod/]                         │
│ │   Prefix: [ga/prod]                                             │
│ └── Analyze with Athena or third-party tools                     │
│                                                                       │
│ Key alarms:                                                          │
│ ├── HealthyEndpointCount < expected: Endpoint failure             │
│ ├── NewFlowCount spike: Potential DDoS or traffic surge           │
│ └── ProcessedBytesIn/Out: Capacity planning                       │
│                                                                       │
│ AWS Shield Advanced:                                                 │
│ ├── GA is eligible for AWS Shield Advanced protection             │
│ ├── DDoS protection + 24/7 DDoS Response Team                   │
│ ├── Cost protection (credits for DDoS-induced scaling)           │
│ └── Enable via Shield console → Protected resources → Add GA    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Terraform Example

```hcl
# Global Accelerator
resource "aws_globalaccelerator_accelerator" "prod" {
  name            = "ga-prod-global"
  ip_address_type = "IPV4"
  enabled         = true

  attributes {
    flow_logs_enabled   = true
    flow_logs_s3_bucket = aws_s3_bucket.ga_logs.id
    flow_logs_s3_prefix = "ga/prod"
  }
}

# Listener (HTTPS)
resource "aws_globalaccelerator_listener" "https" {
  accelerator_arn = aws_globalaccelerator_accelerator.prod.id
  protocol        = "TCP"
  client_affinity = "SOURCE_IP"

  port_range {
    from_port = 443
    to_port   = 443
  }

  port_range {
    from_port = 80
    to_port   = 80
  }
}

# Endpoint group — Mumbai
resource "aws_globalaccelerator_endpoint_group" "mumbai" {
  listener_arn = aws_globalaccelerator_listener.https.id

  endpoint_group_region         = "ap-south-1"
  traffic_dial_percentage       = 100
  health_check_port             = 8080
  health_check_protocol         = "HTTP"
  health_check_path             = "/health"
  health_check_interval_seconds = 30
  threshold_count               = 3

  endpoint_configuration {
    endpoint_id                    = aws_lb.alb_mumbai.arn
    weight                         = 128
    client_ip_preservation_enabled = true
  }
}

# Endpoint group — Virginia
resource "aws_globalaccelerator_endpoint_group" "virginia" {
  listener_arn = aws_globalaccelerator_listener.https.id

  endpoint_group_region         = "us-east-1"
  traffic_dial_percentage       = 100
  health_check_port             = 8080
  health_check_protocol         = "HTTP"
  health_check_path             = "/health"
  health_check_interval_seconds = 30
  threshold_count               = 3

  endpoint_configuration {
    endpoint_id                    = aws_lb.alb_virginia.arn
    weight                         = 128
    client_ip_preservation_enabled = true
  }
}

# DNS pointing to GA
resource "aws_route53_record" "app" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "app.techcorp.com"
  type    = "A"

  alias {
    name                   = aws_globalaccelerator_accelerator.prod.dns_name
    zone_id                = aws_globalaccelerator_accelerator.prod.hosted_zone_id
    evaluate_target_health = true
  }
}

# Output the static IPs
output "ga_static_ips" {
  value = aws_globalaccelerator_accelerator.prod.ip_sets[0].ip_addresses
}
```

---

## Part 10: Real-World Patterns

### Startup

```
Not needed. Use CloudFront for CDN + Route 53 for DNS.
Global Accelerator adds cost with limited benefit for small scale.
Only consider if you have UDP/gaming workloads.
```

### Mid-Size (Multi-Region)

```
Accelerator: 1 standard accelerator

Listener: TCP 80, 443 (source IP affinity)

Endpoint groups:
├── ap-south-1 (Mumbai): Traffic dial 100%
│   └── ALB (weight 128) — primary region
│
└── us-east-1 (Virginia): Traffic dial 100%
    └── ALB (weight 128) — DR region

DNS: app.techcorp.com → GA static IPs
Failover: <30 seconds between regions

Benefits:
├── 2 static IPs for partner allowlisting
├── Instant regional failover (no DNS TTL)
├── Better latency for global users
└── No client-side DNS cache issues

Cost: ~$18/month (fixed) + $0.015-0.035/GB DT premium
```

### Enterprise

```
Accelerators:
├── ga-prod-web (standard)
│   ├── TCP 80, 443 → ALBs in 3 regions
│   ├── Traffic dials: 40% Asia, 40% US, 20% EU
│   ├── Canary: Reduce new region to 5%, observe, scale up
│   └── Shield Advanced enabled
│
├── ga-prod-api (standard)
│   ├── TCP 443 → NLBs in 2 regions
│   ├── Source IP affinity (stateful API sessions)
│   └── Client IP preservation enabled
│
└── ga-prod-gaming (custom routing)
    ├── UDP 10000-20000 → EC2 game servers
    ├── Matchmaker service maps players to ports
    └── Deterministic routing to specific servers

Monitoring:
├── Flow logs → S3 → Athena (traffic analysis)
├── CloudWatch dashboards per accelerator
├── Alarms: Unhealthy endpoints, traffic spikes
├── Shield Advanced: DDoS alerts + response team
└── Cost: ~$100-300/month
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| Static IPs | 2 Anycast IPv4 addresses per accelerator |
| Protocols | TCP, UDP |
| Endpoint types | ALB, NLB, EC2, Elastic IP |
| Failover time | <30 seconds |
| Traffic dials | 0-100% per endpoint group (region) |
| Weights | 0-255 per endpoint |
| Client affinity | None or Source IP |
| Health checks | TCP or HTTP, 10/30s interval |
| Custom routing | Map port ranges to specific EC2 instances |
| DDoS | Shield Standard (free) or Shield Advanced |
| Cost | ~$0.025/hr (~$18/mo) + DT premium ($0.015-0.035/GB) |
| GCP equivalent | Global LB with Anycast (built-in, no separate service) |
| Azure equivalent | Azure Front Door (for HTTP) or Traffic Manager (DNS) |

---

## What's Next?

In the next chapter, we'll cover EC2 — Elastic Compute Cloud, AWS's virtual machine service.

→ Next: [Chapter 13: EC2](13-ec2.md)

---

*Last Updated: May 2026*
