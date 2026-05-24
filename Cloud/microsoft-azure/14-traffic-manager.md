# Chapter 14: Azure Traffic Manager

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Traffic Manager Fundamentals](#part-1-traffic-manager-fundamentals)
- [Part 2: Creating a Traffic Manager Profile (Full Walkthrough)](#part-2-creating-a-traffic-manager-profile-full-walkthrough)
- [Part 3: Routing Methods (Deep Dive)](#part-3-routing-methods-deep-dive)
- [Part 4: Endpoints](#part-4-endpoints)
- [Part 5: Health Monitoring](#part-5-health-monitoring)
- [Part 6: Nested Profiles](#part-6-nested-profiles)
- [Part 7: Real IP & Traffic View](#part-7-real-ip--traffic-view)
- [Part 8: Comparison with AWS & GCP](#part-8-comparison-with-aws--gcp)
- [Part 9: Terraform & Bicep Examples](#part-9-terraform--bicep-examples)
- [Part 10: az CLI Reference](#part-10-az-cli-reference)
- [Part 11: Real-World Patterns](#part-11-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Traffic Manager is a DNS-based global traffic load balancer. Unlike Azure Load Balancer (Layer 4) and Application Gateway (Layer 7) which work at the network/transport level, Traffic Manager works at the DNS level — it directs client requests to the most appropriate endpoint based on routing methods (performance, priority, weighted, geographic, etc.). It's used for multi-region deployments, disaster recovery, and global traffic distribution.

```
What you'll learn:
├── Traffic Manager Fundamentals
│   ├── DNS-based load balancing
│   ├── How it works (DNS resolution flow)
│   ├── Traffic Manager vs Load Balancer vs Front Door
│   └── When to use Traffic Manager
├── Profiles (Full Portal Walkthrough)
│   ├── Routing methods
│   ├── DNS settings (TTL)
│   ├── Endpoints
│   └── Health checks (monitors)
├── Routing Methods (Deep Dive)
│   ├── Priority (active/passive failover)
│   ├── Weighted (traffic splitting)
│   ├── Performance (latency-based)
│   ├── Geographic (geo-fencing)
│   ├── Multivalue (multiple healthy IPs)
│   └── Subnet (client IP → specific endpoint)
├── Endpoints
│   ├── Azure endpoints
│   ├── External endpoints
│   ├── Nested profiles
│   └── Endpoint status & weights
├── Health Monitoring
│   ├── Health check configuration
│   ├── Endpoint failover
│   └── Fast failover settings
├── Nested Profiles (hierarchical routing)
├── Real IP & Traffic View
├── Comparison with AWS & GCP
├── az CLI, Terraform, Bicep examples
└── Real-world patterns
```

---

## Part 1: Traffic Manager Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW TRAFFIC MANAGER WORKS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Traffic Manager is a DNS-based service. It does NOT proxy traffic.│
│ It only resolves DNS names to direct clients to the right endpoint.│
│                                                                       │
│ Flow:                                                                │
│                                                                       │
│ 1. Client → DNS query: myapp.trafficmanager.net                   │
│ 2. DNS → Traffic Manager: "Which endpoint is best?"               │
│ 3. Traffic Manager checks:                                         │
│    ├── Routing method (priority? performance? weighted?)         │
│    ├── Endpoint health (is endpoint healthy?)                    │
│    └── Returns: IP of best endpoint                               │
│ 4. DNS response → Client gets: 20.204.xx.xx (India endpoint)    │
│ 5. Client → connects DIRECTLY to 20.204.xx.xx                   │
│    ⚡ Traffic does NOT flow through Traffic Manager!               │
│    Traffic Manager only resolved the DNS name.                    │
│                                                                       │
│ ┌──────────┐     DNS query     ┌──────────────────┐              │
│ │  Client   │ ──────────────► │ Traffic Manager   │              │
│ │ (browser) │ ◄────────────── │ (DNS response)    │              │
│ └─────┬─────┘    IP address    └──────────────────┘              │
│       │                                                            │
│       │  Direct connection (HTTP/HTTPS/TCP)                       │
│       │  (Traffic Manager is NOT in the data path!)               │
│       │                                                            │
│       ▼                                                            │
│ ┌──────────────┐                                                  │
│ │ Endpoint     │ (App Service, VM, External IP, etc.)            │
│ │ (India)      │                                                  │
│ └──────────────┘                                                  │
│                                                                       │
│ Key point:                                                           │
│ ├── Traffic Manager = DNS resolver (not a proxy)                 │
│ ├── No data flows through Traffic Manager                        │
│ ├── No encryption/SSL termination (endpoint handles that)       │
│ ├── Works with ANY protocol (HTTP, HTTPS, TCP, UDP, gRPC)       │
│ ├── Works with any endpoint (Azure, on-prem, other clouds!)    │
│ └── Global service (not region-specific)                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TRAFFIC MANAGER vs LOAD BALANCER vs FRONT DOOR              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌────────────────┬────────────┬──────────────┬──────────────────┐ │
│ │ Feature        │ Traffic    │ Azure Load   │ Azure Front Door │ │
│ │                │ Manager    │ Balancer     │                  │ │
│ ├────────────────┼────────────┼──────────────┼──────────────────┤ │
│ │ Layer          │ DNS (L7*)  │ L4 (TCP/UDP) │ L7 (HTTP/HTTPS)  │ │
│ │ Scope          │ Global     │ Regional     │ Global           │ │
│ │ Data path      │ DNS only   │ In-path      │ In-path (proxy) │ │
│ │ Protocols      │ Any        │ TCP/UDP      │ HTTP/HTTPS       │ │
│ │ SSL offload    │ No         │ No           │ Yes ✅           │ │
│ │ WAF            │ No         │ No           │ Yes ✅           │ │
│ │ Caching        │ No         │ No           │ Yes ✅           │ │
│ │ Path routing   │ No         │ No           │ Yes ✅           │ │
│ │ Session affin. │ No         │ Yes          │ Yes              │ │
│ │ Failover speed │ DNS TTL    │ Seconds      │ Seconds          │ │
│ │                │ (30-300s)  │              │                  │ │
│ │ Best for       │ Multi-     │ High-perf    │ Global HTTP      │ │
│ │                │ region DNS │ within region│ apps with WAF    │ │
│ │ Cost           │ ~$0.75/    │ ~$18/mo +    │ ~$35/mo +        │ │
│ │                │ million    │ $0.005/GB    │ $0.01/GB         │ │
│ │                │ queries    │              │                  │ │
│ └────────────────┴────────────┴──────────────┴──────────────────┘ │
│                                                                       │
│ When to use which:                                                   │
│ ├── Traffic Manager: Multi-region failover for ANY protocol     │
│ │   (non-HTTP, legacy apps, multi-cloud, on-prem)               │
│ ├── Azure Load Balancer: High-performance L4 within a region    │
│ ├── Application Gateway: L7 within a region (WAF, path routing) │
│ ├── Azure Front Door: Global HTTP/HTTPS with CDN + WAF          │
│ └── ⚡ Use Front Door for HTTP apps (better than TM for HTTP)    │
│     Use Traffic Manager for non-HTTP or multi-cloud scenarios   │
│                                                                       │
│ Common combo:                                                        │
│ Traffic Manager (global DNS) → Application Gateway (per region) │
│ → VM/App Service (backend)                                        │
│ Or: Front Door (global) → App Service/VM (backend)               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Traffic Manager Profile (Full Walkthrough)

```
Console → Search "Traffic Manager profiles" → Create

┌─────────────────────────────────────────────────────────────────┐
│           CREATE TRAFFIC MANAGER PROFILE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Basics ──                                                    │
│                                                                   │
│ Name: [tm-myapp-global]                                         │
│ ⚡ Creates DNS: tm-myapp-global.trafficmanager.net               │
│ ⚡ You CNAME your domain to this:                                │
│    www.myapp.com → CNAME → tm-myapp-global.trafficmanager.net  │
│                                                                   │
│ Routing method: [Priority ▼]                                   │
│   ● Priority (active/passive failover)                         │
│   ○ Weighted (traffic splitting)                               │
│   ○ Performance (latency-based)                                │
│   ○ Geographic (geo-fencing)                                   │
│   ○ Multivalue (return multiple healthy IPs)                  │
│   ○ Subnet (map client IPs to endpoints)                      │
│                                                                   │
│ Subscription: [Pay-As-You-Go ▼]                                │
│ Resource group: [rg-networking-global ▼]                       │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Configure After Creation

```
Traffic Manager profile → tm-myapp-global

┌─────────────────────────────────────────────────────────────────┐
│           CONFIGURATION                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Configuration ──                                             │
│                                                                   │
│ Routing method: [Priority ▼]                                   │
│ (Can change anytime without downtime)                          │
│                                                                   │
│ DNS time to live (TTL): [30] seconds                           │
│ ⚡ How long DNS resolvers cache the response.                   │
│ ├── Lower TTL (10-30s) = faster failover, more DNS queries    │
│ ├── Higher TTL (300s) = slower failover, fewer queries, cheaper│
│ ├── Default: 300s (5 minutes) — too slow for failover!        │
│ └── Recommended: 30-60s for production failover scenarios     │
│ ⚠️ Lower TTL = higher cost (billed per million DNS queries)    │
│                                                                   │
│ Protocol: [HTTPS ▼]                                             │
│   ○ HTTP                                                       │
│   ● HTTPS                                                      │
│   ○ TCP                                                        │
│   ⚡ HTTPS recommended for web apps (health check protocol)    │
│                                                                   │
│ Port: [443]                                                     │
│                                                                   │
│ Path: [/health]                                                 │
│ ⚡ Health check endpoint on your app.                           │
│   Must return 200 OK when healthy.                             │
│                                                                   │
│ ── Health check settings ──                                     │
│                                                                   │
│ Probing interval: [10] seconds                                 │
│   ○ 30 seconds (normal probing)                               │
│   ● 10 seconds (fast probing — recommended)                   │
│   ⚡ 10s = faster failure detection, slightly higher cost      │
│                                                                   │
│ Tolerated number of failures: [3]                              │
│ ⚡ After 3 consecutive failures, endpoint marked unhealthy.    │
│   With 10s interval + 3 failures = unhealthy in 30 seconds.  │
│   With 30s interval + 3 failures = unhealthy in 90 seconds.  │
│                                                                   │
│ Probe timeout: [5] seconds                                     │
│ (How long to wait for health check response)                  │
│                                                                   │
│ Expected status code ranges: [200-299]                         │
│ (HTTP status codes considered healthy)                         │
│                                                                   │
│ [Save]                                                           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Routing Methods (Deep Dive)

### Priority Routing (Active/Passive Failover)

```
┌─────────────────────────────────────────────────────────────────────┐
│           PRIORITY ROUTING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ How: All traffic goes to highest priority (lowest number)         │
│ endpoint. If it fails health check, traffic shifts to next.      │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────┐      │
│ │ Priority 1 (India) ← ALL traffic goes here              │      │
│ │ Status: Online ✅                                        │      │
│ ├──────────────────────────────────────────────────────────┤      │
│ │ Priority 2 (US) ← Only if India fails                   │      │
│ │ Status: Online ✅ (standby)                              │      │
│ ├──────────────────────────────────────────────────────────┤      │
│ │ Priority 3 (EU) ← Only if India AND US fail             │      │
│ │ Status: Online ✅ (standby)                              │      │
│ └──────────────────────────────────────────────────────────┘      │
│                                                                       │
│ Use case: Disaster recovery (primary + failover regions)         │
│                                                                       │
│ ⚡ Most common routing method for DR failover.                     │
│    AWS equivalent: Route 53 Failover routing                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Weighted Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│           WEIGHTED ROUTING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ How: Distribute traffic across endpoints based on weight.         │
│ Weight range: 1-1000 per endpoint.                                │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────┐      │
│ │ India (weight: 70) ← 70% of traffic                     │      │
│ ├──────────────────────────────────────────────────────────┤      │
│ │ US (weight: 20) ← 20% of traffic                        │      │
│ ├──────────────────────────────────────────────────────────┤      │
│ │ EU (weight: 10) ← 10% of traffic                        │      │
│ └──────────────────────────────────────────────────────────┘      │
│                                                                       │
│ Weight 0: Endpoint gets NO traffic (but stays monitored)         │
│ Equal weights: Traffic distributed equally                        │
│                                                                       │
│ Use cases:                                                           │
│ ├── Canary deployments (5% to new version, 95% to stable)      │
│ ├── Gradual migration (shift traffic to new region slowly)      │
│ ├── A/B testing                                                   │
│ └── Load distribution across regions                              │
│                                                                       │
│ ⚡ Canary example:                                                   │
│    Stable endpoint: weight 95                                     │
│    Canary endpoint: weight 5                                      │
│    Monitor → if good → change to 50/50 → then 0/100             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Performance Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│           PERFORMANCE ROUTING                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ How: Direct users to the CLOSEST (lowest latency) endpoint.       │
│ Traffic Manager maintains a latency table mapping client IPs     │
│ to Azure regions.                                                  │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────┐      │
│ │ User in India → India endpoint (10ms)                    │      │
│ │ User in US → US endpoint (15ms)                          │      │
│ │ User in Europe → EU endpoint (12ms)                      │      │
│ └──────────────────────────────────────────────────────────┘      │
│                                                                       │
│ How it decides:                                                      │
│ ├── Microsoft maintains Internet latency table                   │
│ ├── Maps client IP ranges to Azure region latencies              │
│ ├── Returns endpoint with lowest latency to client               │
│ └── Updated periodically based on actual measurements            │
│                                                                       │
│ Use case: Global application, serve users from nearest region    │
│                                                                       │
│ ⚡ AWS equivalent: Route 53 Latency-based routing                  │
│ ⚡ If endpoint is unhealthy, routes to next closest               │
│ ⚡ Works for both Azure and external endpoints                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Geographic Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│           GEOGRAPHIC ROUTING                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ How: Route users based on their geographic location (country,     │
│ continent, or region). Used for geo-fencing and compliance.       │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────┐      │
│ │ India users → India endpoint (data stays in India)       │      │
│ │ EU users → EU endpoint (GDPR compliance)                 │      │
│ │ US users → US endpoint                                   │      │
│ │ All others → Default endpoint (fallback)                 │      │
│ └──────────────────────────────────────────────────────────┘      │
│                                                                       │
│ Geo mapping:                                                        │
│ ├── World (catch-all)                                              │
│ ├── Continent: Asia, Europe, North America, etc.                 │
│ ├── Country/Region: India, Germany, United States, etc.          │
│ └── State/Province (US/Canada/Australia only)                    │
│                                                                       │
│ Important:                                                           │
│ ├── Each geo can map to ONLY ONE endpoint                        │
│ ├── MUST have a "World" mapping as fallback!                    │
│ │   (If no mapping matches the user's location)                 │
│ ├── Geo is determined by client's DNS resolver IP                │
│ │   (Not perfect — corporate DNS may be in different location) │
│ └── If endpoint unhealthy → NO failover to another geo!        │
│     (Unlike priority/performance — strict geo isolation)        │
│     Solution: Use nested profiles for per-geo failover          │
│                                                                       │
│ Use cases:                                                           │
│ ├── Data sovereignty (keep EU data in EU)                        │
│ ├── GDPR/regulatory compliance                                   │
│ ├── Content localization (language, currency by region)          │
│ └── Licensing restrictions (content only available in US)       │
│                                                                       │
│ ⚡ AWS equivalent: Route 53 Geolocation routing                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Multivalue & Subnet Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│           MULTIVALUE & SUBNET ROUTING                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Multivalue:                                                          │
│ ├── Returns MULTIPLE healthy endpoint IPs in DNS response        │
│ ├── Max addresses to return: 1-8 (configurable)                  │
│ ├── Client picks one (client-side load balancing)               │
│ ├── Only returns healthy endpoints                                │
│ ├── Like DNS round-robin but with health checks                 │
│ └── Use case: Simple multi-IP failover without Azure LB         │
│                                                                       │
│ Subnet:                                                              │
│ ├── Map specific client IP ranges to specific endpoints          │
│ ├── "Requests from 10.0.0.0/8 → endpoint A"                    │
│ ├── "Requests from 192.168.0.0/16 → endpoint B"                │
│ ├── Use case: Different experience for corporate vs public      │
│ ├── Use case: Route internal users to staging, external to prod │
│ └── ⚠️ Based on DNS resolver IP, not actual client IP            │
│                                                                       │
│ ⚡ These are less common. Most deployments use Priority,           │
│    Performance, or Weighted routing.                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Endpoints

```
Console → Traffic Manager profile → Endpoints → Add

┌─────────────────────────────────────────────────────────────────┐
│           ADD ENDPOINT                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Type:                                                            │
│   ● Azure endpoint                                              │
│   ○ External endpoint                                           │
│   ○ Nested endpoint                                             │
│                                                                   │
│ ── Azure endpoint ──                                            │
│ Name: [ep-india-appservice]                                     │
│ Target resource type: [App Service ▼]                          │
│   ○ App Service                                                │
│   ○ App Service (Slots)                                        │
│   ○ Cloud Service                                              │
│   ○ Public IP address                                          │
│   ○ Application Gateway (for path-based routing combo)        │
│                                                                   │
│ Target resource: [app-myapp-india ▼]                           │
│                                                                   │
│ Priority: [1] (for Priority routing — lower = higher priority)│
│ Weight: [100] (for Weighted routing)                           │
│ Custom header: Host: www.myapp.com                             │
│                                                                   │
│ Geo mapping (for Geographic routing only):                     │
│   [Asia-Pacific → India ▼]                                     │
│                                                                   │
│ ── External endpoint ──                                         │
│ Name: [ep-onprem-dc]                                            │
│ FQDN or IP: [203.0.113.50] or [dc.mycompany.com]             │
│ ⚡ Use for: On-prem servers, other cloud providers (AWS, GCP) │
│                                                                   │
│ ── Nested endpoint ──                                           │
│ Name: [ep-nested-india]                                         │
│ Target resource: [tm-india-failover ▼]                         │
│ (Another Traffic Manager profile)                              │
│ Minimum child endpoints: [1]                                   │
│ ⚡ Enables hierarchical routing (covered in Part 6)             │
│                                                                   │
│ ── Common fields ──                                             │
│ Endpoint status: ● Enabled  ○ Disabled                        │
│ ⚡ Disable to take endpoint out of rotation without deleting   │
│                                                                   │
│ [Add]                                                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Example setup (Priority routing):
┌──────────────────┬──────────┬──────────┬────────────────────────┐
│ Endpoint         │ Type     │ Priority │ Target                 │
├──────────────────┼──────────┼──────────┼────────────────────────┤
│ ep-india-primary │ Azure    │ 1        │ app-myapp-india        │
│ ep-us-secondary  │ Azure    │ 2        │ app-myapp-us           │
│ ep-eu-tertiary   │ Azure    │ 3        │ app-myapp-eu           │
│ ep-onprem-last   │ External │ 4        │ 203.0.113.50           │
└──────────────────┴──────────┴──────────┴────────────────────────┘

Endpoint status indicators:
├── Online ✅: Healthy, receiving traffic
├── Degraded ⚠️: Health check failing, NOT receiving traffic
├── Disabled 🚫: Manually disabled
├── Stopped ⛔: Profile stopped
└── CheckingEndpoint 🔄: Just added, initial health check
```

---

## Part 5: Health Monitoring

```
┌─────────────────────────────────────────────────────────────────────┐
│           HEALTH MONITORING                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Traffic Manager continuously probes endpoints to check health.    │
│                                                                       │
│ Probe flow:                                                          │
│ Traffic Manager probes → GET /health → expects 200-299            │
│                                                                       │
│ Probe settings:                                                      │
│ ┌──────────────────────┬─────────────┬──────────────────────────┐ │
│ │ Setting              │ Normal      │ Fast (recommended)       │ │
│ ├──────────────────────┼─────────────┼──────────────────────────┤ │
│ │ Probing interval     │ 30 seconds  │ 10 seconds               │ │
│ │ Failures tolerated   │ 3           │ 3                        │ │
│ │ Probe timeout        │ 10 seconds  │ 5 seconds                │ │
│ │ Time to failover     │ ~120 sec    │ ~40 sec                  │ │
│ │ Cost                 │ Lower       │ Slightly higher          │ │
│ └──────────────────────┴─────────────┴──────────────────────────┘ │
│                                                                       │
│ ⚡ Use fast probing (10s interval) for production!                  │
│    Failover in ~40 seconds vs ~120 seconds with normal probing.  │
│                                                                       │
│ Health check endpoint best practices:                                │
│ ├── Dedicated /health or /healthz endpoint                       │
│ ├── Check dependencies (database, cache, external APIs)         │
│ ├── Return 200 if healthy, 503 if unhealthy                     │
│ ├── Lightweight — don't do expensive operations                 │
│ ├── Include custom headers if needed (Host header for App Svc)  │
│ └── Monitor the health check endpoint itself (inception!)       │
│                                                                       │
│ Failover timeline (fast probing, 3 failures):                      │
│ 0s:   Endpoint goes down                                          │
│ 10s:  First failed probe                                          │
│ 20s:  Second failed probe                                         │
│ 30s:  Third failed probe → endpoint marked DEGRADED              │
│ 30s+: DNS response changes to next healthy endpoint               │
│ +TTL: Clients pick up new DNS (depends on TTL setting)           │
│                                                                       │
│ Total failover time: ~30s (probing) + TTL (30-300s)               │
│ ⚡ With TTL=30s: Total failover in ~60 seconds                     │
│ ⚠️ With TTL=300s: Total failover in ~330 seconds (5.5 minutes!)   │
│                                                                       │
│ ⚡ For fastest failover:                                             │
│    Probing interval: 10s                                           │
│    Tolerated failures: 2 (faster, but more false positives)      │
│    TTL: 10-30s (higher DNS query cost)                            │
│    → Failover in ~30-50 seconds                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Nested Profiles

```
┌─────────────────────────────────────────────────────────────────────┐
│           NESTED PROFILES                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Use one Traffic Manager profile as an endpoint in another.  │
│ Enables complex, hierarchical routing strategies.                 │
│                                                                       │
│ Example: Performance routing (global) + Priority routing (per region)│
│                                                                       │
│              ┌──────────────────────────┐                           │
│              │ Parent: tm-global        │                           │
│              │ Routing: Performance     │                           │
│              └──────┬───────┬───────────┘                           │
│                     │       │                                        │
│            ┌────────┘       └────────┐                               │
│            ▼                         ▼                               │
│ ┌────────────────────┐  ┌────────────────────┐                     │
│ │ Child: tm-india    │  │ Child: tm-us       │                     │
│ │ Routing: Priority  │  │ Routing: Priority  │                     │
│ │ ├── ep-india-1 (P1)│  │ ├── ep-us-east (P1)│                     │
│ │ └── ep-india-2 (P2)│  │ └── ep-us-west (P2)│                     │
│ └────────────────────┘  └────────────────────┘                     │
│                                                                       │
│ Result:                                                              │
│ ├── Indian user → Performance picks tm-india (closest)           │
│ │   → tm-india uses Priority → sends to ep-india-1               │
│ │   → If ep-india-1 down → failover to ep-india-2               │
│ ├── US user → Performance picks tm-us (closest)                  │
│ │   → tm-us uses Priority → sends to ep-us-east                 │
│ │   → If ep-us-east down → failover to ep-us-west               │
│ └── ⚡ Global performance routing + per-region failover!          │
│                                                                       │
│ Another example: Geographic + Weighted (canary per region)        │
│                                                                       │
│              ┌──────────────────────────┐                           │
│              │ Parent: tm-global-geo    │                           │
│              │ Routing: Geographic      │                           │
│              └──────┬───────┬───────────┘                           │
│                     │       │                                        │
│            India    │       │   EU                                   │
│            ┌────────┘       └────────┐                               │
│            ▼                         ▼                               │
│ ┌────────────────────┐  ┌────────────────────┐                     │
│ │ Child: tm-india-wt │  │ Child: tm-eu-wt    │                     │
│ │ Routing: Weighted  │  │ Routing: Weighted  │                     │
│ │ ├── stable (95)    │  │ ├── stable (90)    │                     │
│ │ └── canary (5)     │  │ └── canary (10)    │                     │
│ └────────────────────┘  └────────────────────┘                     │
│                                                                       │
│ Result:                                                              │
│ ├── Indian users geo-fenced to India + 5% canary                 │
│ ├── EU users geo-fenced to EU + 10% canary                       │
│ └── ⚡ Per-region canary deployments with geo compliance!          │
│                                                                       │
│ Nested endpoint settings:                                            │
│ Minimum child endpoints: [1]                                       │
│ ⚡ How many child endpoints must be healthy for the nested        │
│    endpoint to be considered healthy in the parent.               │
│    Set to 1: If any child is healthy → parent considers          │
│    the nested endpoint healthy.                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Real IP & Traffic View

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL USER MEASUREMENTS & TRAFFIC VIEW                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Real User Measurements (RUM):                                       │
│ ├── Measures actual latency from real users to Azure regions     │
│ ├── Improves Performance routing accuracy                        │
│ ├── Add JavaScript snippet to your web pages                     │
│ ├── Browser measures latency to Azure endpoints                  │
│ ├── Traffic Manager uses real data for routing decisions         │
│ └── Better than Microsoft's default latency table                │
│                                                                       │
│ Traffic View:                                                        │
│ ├── Visual map of where your DNS queries come from               │
│ ├── Shows: Query volume by region, latency to endpoints          │
│ ├── Helps decide: Where to add new endpoints/regions            │
│ ├── Dashboard: Traffic Manager → Traffic View                   │
│ └── ⚡ Great for capacity planning!                                │
│                                                                       │
│ Enable:                                                              │
│ Traffic Manager → Real User Measurements → Generate key          │
│ Add to your HTML:                                                    │
│ <script src="https://www.trafficmanagerpool.net/apc/                │
│   rum.js?key=YOUR_RUM_KEY" async></script>                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Comparison with AWS & GCP

```
┌─────────────────────────────────────────────────────────────────────┐
│           CROSS-CLOUD COMPARISON                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌────────────────────┬───────────────┬───────────┬────────────────┐│
│ │ Feature            │ Azure TM      │ AWS       │ GCP            ││
│ ├────────────────────┼───────────────┼───────────┼────────────────┤│
│ │ DNS-based LB       │ Traffic       │ Route 53  │ Cloud DNS      ││
│ │                    │ Manager       │ routing   │ routing        ││
│ │                    │               │ policies  │ policies       ││
│ │ Priority/Failover  │ ✅            │ ✅        │ ✅ (failover)  ││
│ │ Weighted           │ ✅            │ ✅        │ ✅ (weighted)  ││
│ │ Performance/Latency│ ✅            │ ✅        │ ✅ (geolocation)││
│ │ Geographic         │ ✅            │ ✅        │ ✅             ││
│ │ Multivalue         │ ✅            │ ✅ (multi)│ Limited        ││
│ │ Subnet/IP-based    │ ✅            │ ✅ (IP)   │ Limited        ││
│ │ Nested profiles    │ ✅            │ Alias     │ N/A            ││
│ │ Health checks      │ ✅            │ ✅        │ ✅             ││
│ │ External endpoints │ ✅            │ ✅        │ External DNS   ││
│ │ Global proxy (L7)  │ Front Door    │ CloudFront│ Cloud LB       ││
│ │ Cost               │ ~$0.75/M     │ ~$0.50/M  │ ~$0.40/M       ││
│ │                    │ queries      │ queries   │ queries        ││
│ └────────────────────┴───────────────┴───────────┴────────────────┘│
│                                                                       │
│ Key differences:                                                     │
│ ├── Azure TM is a standalone DNS LB service                      │
│ ├── AWS Route 53 is a full DNS service WITH routing policies     │
│ ├── GCP Cloud DNS has routing policies built into DNS zones      │
│ ├── Azure Front Door is the L7 global proxy (replaces TM for HTTP)│
│ └── All three achieve similar outcomes with different approaches  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Terraform & Bicep Examples

### Terraform

```hcl
# Traffic Manager Profile (Priority routing)
resource "azurerm_traffic_manager_profile" "global" {
  name                   = "tm-myapp-global"
  resource_group_name    = azurerm_resource_group.networking.name
  traffic_routing_method = "Priority"

  dns_config {
    relative_name = "tm-myapp-global"
    ttl           = 30
  }

  monitor_config {
    protocol                     = "HTTPS"
    port                         = 443
    path                         = "/health"
    interval_in_seconds          = 10
    timeout_in_seconds           = 5
    tolerated_number_of_failures = 3
    expected_status_code_ranges  = ["200-299"]

    custom_header {
      name  = "Host"
      value = "www.myapp.com"
    }
  }

  tags = {
    environment = "prod"
  }
}

# Azure endpoint - Primary (India)
resource "azurerm_traffic_manager_azure_endpoint" "india" {
  name               = "ep-india-primary"
  profile_id         = azurerm_traffic_manager_profile.global.id
  target_resource_id = azurerm_linux_web_app.india.id
  priority           = 1
  weight             = 100
}

# Azure endpoint - Secondary (US)
resource "azurerm_traffic_manager_azure_endpoint" "us" {
  name               = "ep-us-secondary"
  profile_id         = azurerm_traffic_manager_profile.global.id
  target_resource_id = azurerm_linux_web_app.us.id
  priority           = 2
  weight             = 100
}

# External endpoint (on-premises)
resource "azurerm_traffic_manager_external_endpoint" "onprem" {
  name       = "ep-onprem-dr"
  profile_id = azurerm_traffic_manager_profile.global.id
  target     = "dc.mycompany.com"
  priority   = 3
  weight     = 100

  custom_header {
    name  = "Host"
    value = "www.myapp.com"
  }
}

# ──────────────────────────────────────────────
# Performance routing example
# ──────────────────────────────────────────────

resource "azurerm_traffic_manager_profile" "perf" {
  name                   = "tm-myapp-perf"
  resource_group_name    = azurerm_resource_group.networking.name
  traffic_routing_method = "Performance"

  dns_config {
    relative_name = "tm-myapp-perf"
    ttl           = 30
  }

  monitor_config {
    protocol                     = "HTTPS"
    port                         = 443
    path                         = "/health"
    interval_in_seconds          = 10
    timeout_in_seconds           = 5
    tolerated_number_of_failures = 3
  }
}

resource "azurerm_traffic_manager_azure_endpoint" "india_perf" {
  name               = "ep-india"
  profile_id         = azurerm_traffic_manager_profile.perf.id
  target_resource_id = azurerm_linux_web_app.india.id
  endpoint_location  = "Central India"
}

resource "azurerm_traffic_manager_azure_endpoint" "us_perf" {
  name               = "ep-us"
  profile_id         = azurerm_traffic_manager_profile.perf.id
  target_resource_id = azurerm_linux_web_app.us.id
  endpoint_location  = "East US"
}

resource "azurerm_traffic_manager_azure_endpoint" "eu_perf" {
  name               = "ep-eu"
  profile_id         = azurerm_traffic_manager_profile.perf.id
  target_resource_id = azurerm_linux_web_app.eu.id
  endpoint_location  = "West Europe"
}

# ──────────────────────────────────────────────
# Nested profile (Performance parent + Priority children)
# ──────────────────────────────────────────────

# Parent profile
resource "azurerm_traffic_manager_profile" "parent" {
  name                   = "tm-global-parent"
  resource_group_name    = azurerm_resource_group.networking.name
  traffic_routing_method = "Performance"

  dns_config {
    relative_name = "tm-global-parent"
    ttl           = 30
  }

  monitor_config {
    protocol                     = "HTTPS"
    port                         = 443
    path                         = "/health"
    interval_in_seconds          = 10
    timeout_in_seconds           = 5
    tolerated_number_of_failures = 3
  }
}

# Child profile (India region failover)
resource "azurerm_traffic_manager_profile" "india_child" {
  name                   = "tm-india-failover"
  resource_group_name    = azurerm_resource_group.networking.name
  traffic_routing_method = "Priority"

  dns_config {
    relative_name = "tm-india-failover"
    ttl           = 30
  }

  monitor_config {
    protocol                     = "HTTPS"
    port                         = 443
    path                         = "/health"
    interval_in_seconds          = 10
    timeout_in_seconds           = 5
    tolerated_number_of_failures = 3
  }
}

resource "azurerm_traffic_manager_azure_endpoint" "india_primary" {
  name               = "ep-india-primary"
  profile_id         = azurerm_traffic_manager_profile.india_child.id
  target_resource_id = azurerm_linux_web_app.india_1.id
  priority           = 1
}

resource "azurerm_traffic_manager_azure_endpoint" "india_secondary" {
  name               = "ep-india-secondary"
  profile_id         = azurerm_traffic_manager_profile.india_child.id
  target_resource_id = azurerm_linux_web_app.india_2.id
  priority           = 2
}

# Nested endpoint in parent
resource "azurerm_traffic_manager_nested_endpoint" "india_nested" {
  name                    = "ep-nested-india"
  profile_id              = azurerm_traffic_manager_profile.parent.id
  target_resource_id      = azurerm_traffic_manager_profile.india_child.id
  minimum_child_endpoints = 1
  endpoint_location       = "Central India"
}

# CNAME your domain (in your DNS provider)
# www.myapp.com → CNAME → tm-global-parent.trafficmanager.net
```

### Bicep

```bicep
resource trafficManager 'Microsoft.Network/trafficManagerProfiles@2022-04-01' = {
  name: 'tm-myapp-global'
  location: 'global'
  properties: {
    profileStatus: 'Enabled'
    trafficRoutingMethod: 'Priority'
    dnsConfig: {
      relativeName: 'tm-myapp-global'
      ttl: 30
    }
    monitorConfig: {
      protocol: 'HTTPS'
      port: 443
      path: '/health'
      intervalInSeconds: 10
      timeoutInSeconds: 5
      toleratedNumberOfFailures: 3
      expectedStatusCodeRanges: [
        { min: 200, max: 299 }
      ]
    }
    endpoints: [
      {
        name: 'ep-india-primary'
        type: 'Microsoft.Network/trafficManagerProfiles/azureEndpoints'
        properties: {
          targetResourceId: appServiceIndia.id
          priority: 1
          weight: 100
          endpointStatus: 'Enabled'
        }
      }
      {
        name: 'ep-us-secondary'
        type: 'Microsoft.Network/trafficManagerProfiles/azureEndpoints'
        properties: {
          targetResourceId: appServiceUs.id
          priority: 2
          weight: 100
          endpointStatus: 'Enabled'
        }
      }
    ]
  }
}
```

---

## Part 10: az CLI Reference

```bash
# Create profile (Priority routing)
az network traffic-manager profile create \
  --name tm-myapp-global \
  --resource-group rg-networking-global \
  --routing-method Priority \
  --unique-dns-name tm-myapp-global \
  --ttl 30 \
  --protocol HTTPS \
  --port 443 \
  --path "/health" \
  --interval 10 \
  --timeout 5 \
  --max-failures 3

# Add Azure endpoint
az network traffic-manager endpoint create \
  --name ep-india-primary \
  --profile-name tm-myapp-global \
  --resource-group rg-networking-global \
  --type azureEndpoints \
  --target-resource-id /subscriptions/.../Microsoft.Web/sites/app-myapp-india \
  --priority 1 \
  --endpoint-status Enabled

# Add external endpoint
az network traffic-manager endpoint create \
  --name ep-onprem \
  --profile-name tm-myapp-global \
  --resource-group rg-networking-global \
  --type externalEndpoints \
  --target dc.mycompany.com \
  --priority 3 \
  --endpoint-status Enabled

# Add nested endpoint
az network traffic-manager endpoint create \
  --name ep-nested-india \
  --profile-name tm-global-parent \
  --resource-group rg-networking-global \
  --type nestedEndpoints \
  --target-resource-id /subscriptions/.../trafficManagerProfiles/tm-india-failover \
  --min-child-endpoints 1 \
  --endpoint-location "Central India"

# Check profile status
az network traffic-manager profile show \
  --name tm-myapp-global \
  --resource-group rg-networking-global \
  --query "{name:name, status:profileStatus, routing:trafficRoutingMethod, fqdn:dnsConfig.fqdn}" \
  --output table

# List endpoints and status
az network traffic-manager endpoint list \
  --profile-name tm-myapp-global \
  --resource-group rg-networking-global \
  --type azureEndpoints \
  --query "[].{name:name, status:endpointStatus, monitor:endpointMonitorStatus, priority:priority}" \
  --output table

# Change routing method
az network traffic-manager profile update \
  --name tm-myapp-global \
  --resource-group rg-networking-global \
  --routing-method Performance

# Disable endpoint (maintenance)
az network traffic-manager endpoint update \
  --name ep-india-primary \
  --profile-name tm-myapp-global \
  --resource-group rg-networking-global \
  --type azureEndpoints \
  --endpoint-status Disabled

# Test DNS resolution
nslookup tm-myapp-global.trafficmanager.net
dig tm-myapp-global.trafficmanager.net
```

---

## Part 11: Real-World Patterns

### Startup

```
Setup: Single region, no Traffic Manager needed yet.

When to add Traffic Manager:
├── Expanding to second region (DR)
├── Need failover to on-prem backup
└── Multi-cloud deployment

First TM setup (when ready):
├── Profile: Priority routing
├── TTL: 30 seconds
├── Health checks: HTTPS /health, 10s interval
├── Endpoints:
│   ├── Priority 1: App Service (India)
│   └── Priority 2: App Service (US) — cold standby
└── CNAME: www.myapp.com → tm-myapp.trafficmanager.net

Cost: ~$5-15/month (based on DNS query volume)
```

### Mid-Size

```
Architecture: 2 active regions + Performance routing

Profile: tm-myapp-perf
├── Routing: Performance (latency-based)
├── TTL: 30 seconds
├── Health: HTTPS /health, 10s interval, 3 failures
├── Endpoints:
│   ├── ep-india (Central India) — App Service
│   ├── ep-us (East US) — App Service
│   └── ep-eu (West Europe) — App Service
└── DNS: www.myapp.com → CNAME → tm-myapp-perf.trafficmanager.net

Indian users → India endpoint (lowest latency)
US users → US endpoint
EU users → EU endpoint
If any endpoint unhealthy → next closest

Deployment (canary):
├── Switch to Weighted routing temporarily
├── New version: weight 5 (canary)
├── Stable version: weight 95
├── Monitor → increase to 50/50 → then 0/100
├── Switch back to Performance routing
└── Or: Use nested profiles for per-region canary

Monitoring:
├── Traffic View dashboard (query patterns)
├── Azure Monitor alerts on endpoint degradation
├── Log Analytics: DNS query logs
└── Real User Measurements (RUM) for accurate latency

Cost: ~$20-50/month
```

### Enterprise

```
Architecture: Nested profiles — Performance + Priority per region

Parent profile: tm-global-parent
├── Routing: Performance
├── TTL: 10 seconds (fastest failover)
├── Endpoints (nested):
│   ├── ep-nested-india → tm-india-failover
│   ├── ep-nested-us → tm-us-failover
│   └── ep-nested-eu → tm-eu-failover

Child: tm-india-failover
├── Routing: Priority
├── Endpoints:
│   ├── P1: App Gateway (Central India) — active
│   └── P2: App Gateway (South India) — passive

Child: tm-us-failover
├── Routing: Priority
├── Endpoints:
│   ├── P1: App Gateway (East US) — active
│   └── P2: App Gateway (West US) — passive

Child: tm-eu-failover
├── Routing: Priority
├── Endpoints:
│   ├── P1: App Gateway (West Europe) — active
│   ├── P2: App Gateway (North Europe) — passive
│   └── P3: External (on-prem EU DC) — last resort

Result:
├── User routed to nearest region (Performance)
├── Within region, primary → secondary failover (Priority)
├── Global: 6 endpoints across 3 regions
├── Regional: Per-region HA with failover
└── Total failover time: ~40-50 seconds

Additional TM profiles:
├── Geographic profile for GDPR (EU data stays in EU)
├── Weighted profiles for per-region canary deployments
├── Subnet routing for internal vs external users

Monitoring:
├── Real User Measurements (JavaScript on all pages)
├── Traffic View (capacity planning)
├── Azure Monitor: Endpoint status alerts → PagerDuty
├── Sentinel: Anomaly detection on DNS patterns
└── Monthly review: Traffic View for new region planning

DNS setup:
├── www.myapp.com → CNAME → tm-global-parent.trafficmanager.net
├── api.myapp.com → CNAME → tm-api-global.trafficmanager.net
├── Custom domain verification on all App Services
└── SSL certificates on all endpoints (Let's Encrypt / managed)

Cost: ~$50-200/month (multiple profiles + high query volume)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | DNS-based global traffic load balancer |
| How | Returns IP of best endpoint via DNS resolution |
| Data path | DNS only — traffic goes directly to endpoint |
| Protocols | Any (HTTP, HTTPS, TCP, UDP — DNS-level routing) |
| Routing: Priority | Active/passive failover (DR) |
| Routing: Weighted | Traffic splitting (canary, A/B testing) |
| Routing: Performance | Latency-based (nearest endpoint) |
| Routing: Geographic | Geo-fencing (compliance, localization) |
| Routing: Multivalue | Return multiple healthy IPs |
| Routing: Subnet | Client IP → specific endpoint |
| Endpoints | Azure, External (on-prem/other cloud), Nested profiles |
| Health checks | HTTP/HTTPS/TCP, 10-30s interval, configurable failures |
| TTL | 10-300s (lower = faster failover, more queries, higher cost) |
| Nested profiles | Hierarchical routing (combine routing methods) |
| Traffic View | Visual dashboard of DNS query patterns |
| Cost | ~$0.75/million DNS queries |
| AWS equivalent | Route 53 routing policies |
| GCP equivalent | Cloud DNS routing policies |

---

## What's Next?

In the next chapter, we'll cover Azure Virtual Machines — the core compute service.

→ Next: [Chapter 15: Virtual Machines](15-virtual-machines.md)

---

*Last Updated: May 2026*
