# Chapter 12: Azure Application Gateway (Deep Dive)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Architecture Deep Dive](#part-1-architecture-deep-dive)
- [Part 2: Subnet & NSG Requirements](#part-2-subnet--nsg-requirements)
- [Part 3: Listeners (Deep Dive)](#part-3-listeners-deep-dive)
- [Part 4: Backend Pools & Backend Settings](#part-4-backend-pools--backend-settings)
- [Part 5: Health Probes](#part-5-health-probes)
- [Part 6: Routing Rules & URL Path Maps](#part-6-routing-rules--url-path-maps)
- [Part 7: Rewrite Rules](#part-7-rewrite-rules)
- [Part 8: WAF Policy (Deep Dive)](#part-8-waf-policy-deep-dive)
- [Part 9: Private Application Gateway & AKS Ingress](#part-9-private-application-gateway--aks-ingress)
- [Part 10: Monitoring & Diagnostics](#part-10-monitoring--diagnostics)
- [Part 11: Terraform Example](#part-11-terraform-example)
- [Part 12: Real-World Patterns](#part-12-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Application Gateway is a Layer 7 (HTTP/HTTPS) load balancer and web application firewall. Chapter 11 covered the basics alongside Azure Load Balancer. This chapter is the complete, in-depth reference for Application Gateway — every setting, every field, every scenario.

```
What you'll learn:
├── Application Gateway Architecture (deep dive)
├── SKUs & Tiers (Standard_v2, WAF_v2, Basic)
├── Subnet & NSG Requirements
├── Frontend IP Configurations
├── Listeners
│   ├── Basic vs Multi-site
│   ├── SSL/TLS termination
│   ├── SSL profiles & policies
│   └── Mutual TLS (mTLS)
├── Backend Pools
│   ├── VMs, VMSS, App Service, IPs/FQDNs
│   └── Private endpoints as backends
├── Backend Settings (HTTP Settings)
│   ├── Protocol, port, timeout
│   ├── Cookie-based affinity
│   ├── Connection draining
│   ├── Host override
│   ├── Custom probes
│   └── Backend authentication
├── Routing Rules
│   ├── Basic routing
│   ├── Path-based routing (URL path maps)
│   └── Priority
├── Redirect Configuration
├── Rewrite Rules (headers & URL)
├── WAF Policies (OWASP, custom, bot, exclusions)
├── Autoscaling
├── Private Application Gateway
├── Ingress Controller for AKS
├── Monitoring & Diagnostics
├── Terraform / Bicep / az CLI
└── Real-world patterns
```

---

## Part 1: Architecture Deep Dive

```
┌─────────────────────────────────────────────────────────────────────┐
│           APPLICATION GATEWAY ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ App Gateway is a DEPLOYED RESOURCE — it runs on dedicated          │
│ instances (VMs) managed by Azure in YOUR dedicated subnet.         │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐    │
│ │                    YOUR VNet (vnet-prod)                      │    │
│ │                                                               │    │
│ │ ┌─────────────────────────────────────────────────────────┐ │    │
│ │ │ subnet-appgw (/24 or /26 — DEDICATED)                   │ │    │
│ │ │                                                          │ │    │
│ │ │ ┌──────────────────────────────────────────────────┐   │ │    │
│ │ │ │          APPLICATION GATEWAY                      │   │ │    │
│ │ │ │                                                    │   │ │    │
│ │ │ │ Frontend IP ──► Listener ──► Routing Rule          │   │ │    │
│ │ │ │ (Public/Pvt)    (port+cert)   (basic/path-based)  │   │ │    │
│ │ │ │                                    │               │   │ │    │
│ │ │ │                               ┌────▼────┐         │   │ │    │
│ │ │ │                               │URL Path │         │   │ │    │
│ │ │ │                               │  Map    │         │   │ │    │
│ │ │ │                               └────┬────┘         │   │ │    │
│ │ │ │                                    │               │   │ │    │
│ │ │ │                            ┌───────▼──────┐       │   │ │    │
│ │ │ │                            │Backend Pool  │       │   │ │    │
│ │ │ │                            │+ Backend     │       │   │ │    │
│ │ │ │                            │  Settings    │       │   │ │    │
│ │ │ │                            └──────────────┘       │   │ │    │
│ │ │ │ Instance 1 ◄──┐                                   │   │ │    │
│ │ │ │ Instance 2 ◄──┤ (auto-scaled v2 instances)       │   │ │    │
│ │ │ │ Instance N ◄──┘                                   │   │ │    │
│ │ │ └──────────────────────────────────────────────────┘   │ │    │
│ │ └─────────────────────────────────────────────────────────┘ │    │
│ │                                                               │    │
│ │ ┌───────────────┐  ┌───────────────┐  ┌──────────────────┐ │    │
│ │ │subnet-web     │  │subnet-api     │  │subnet-data       │ │    │
│ │ │(backend VMs)  │  │(App Service)  │  │(SQL Server)      │ │    │
│ │ └───────────────┘  └───────────────┘  └──────────────────┘ │    │
│ └───────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ Traffic flow:                                                        │
│ Client → Public IP → App GW Frontend →                             │
│ Listener (SSL termination) → Routing Rule →                        │
│ URL Path Map (if path-based) →                                      │
│ Backend Pool (selected) →                                           │
│ Backend Settings (protocol, port, host override) →                 │
│ Health Probe (only healthy backends) →                              │
│ Backend VM/App Service/VMSS                                         │
│                                                                       │
│ ⚠️ Key concept: Listener + Routing Rule + Backend Pool +            │
│    Backend Settings = one complete routing chain.                  │
│    Each chain can have different SSL certs, WAF, backends.        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Subnet & NSG Requirements

```
┌─────────────────────────────────────────────────────────────────────┐
│           SUBNET REQUIREMENTS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Application Gateway MUST have its own dedicated subnet.            │
│                                                                       │
│ Subnet sizing:                                                       │
│ ├── Minimum: /26 (64 IPs) — fits ~32 instances                   │
│ ├── Recommended: /24 (256 IPs) — room to grow                    │
│ ├── Each App GW instance uses 1 private IP                        │
│ ├── Each private frontend IP uses 1 IP                             │
│ ├── Azure reserves 5 IPs per subnet                               │
│ └── Formula: (max_instances × 1) + (private_frontends × 1) + 5  │
│                                                                       │
│ Subnet restrictions:                                                 │
│ ├── ONLY Application Gateways can be in this subnet               │
│ ├── No other VMs, NICs, private endpoints                          │
│ ├── Multiple App GWs CAN share same subnet (same SKU)            │
│ ├── No route table with 0.0.0.0/0 to non-internet next hop       │
│ └── No NSG rules that deny GatewayManager or AzureLoadBalancer   │
│                                                                       │
│ Required NSG rules (if NSG is on the subnet):                      │
│ ┌────────┬───────────┬──────────────────┬───────┬─────────┐       │
│ │ Dir    │ Port      │ Source           │ Dest  │ Action  │       │
│ ├────────┼───────────┼──────────────────┼───────┼─────────┤       │
│ │ Inbound│ 65200-    │ GatewayManager   │ Any   │ Allow ⚠️│       │
│ │        │ 65535     │                  │       │ (MUST!) │       │
│ │ Inbound│ 80, 443  │ Internet (or     │ Any   │ Allow   │       │
│ │        │           │ your CIDRs)     │       │         │       │
│ │ Inbound│ Any      │ AzureLoadBalancer│ Any   │ Allow   │       │
│ └────────┴───────────┴──────────────────┴───────┴─────────┘       │
│                                                                       │
│ ⚠️ If you block ports 65200-65535 from GatewayManager,               │
│    App GW health probes will FAIL and it will show as UNHEALTHY!  │
│    This is the #1 most common App GW configuration error.         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Listeners (Deep Dive)

```
┌─────────────────────────────────────────────────────────────────────┐
│           LISTENERS                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ A listener "listens" for incoming connections on a frontend IP     │
│ + port + protocol combination.                                     │
│                                                                       │
│ BASIC LISTENER:                                                      │
│ ├── Single site — accepts ALL hostnames on that port              │
│ ├── One basic listener per port per frontend IP                   │
│ └── Use when: Single domain, or you route by path only           │
│                                                                       │
│ MULTI-SITE LISTENER:                                                 │
│ ├── Listens for specific hostname(s)                              │
│ ├── Multiple multi-site listeners can share same port             │
│ ├── Each handles different domain                                  │
│ └── Use when: Multiple domains on same App GW                    │
│                                                                       │
│ Example — Multi-site setup on port 443:                             │
│ ┌───────────────────────────────────────────────────────────┐      │
│ │ Listener: listener-web                                    │      │
│ │   Type: Multi-site                                        │      │
│ │   Frontend: Public IP                                     │      │
│ │   Port: 443 (HTTPS)                                       │      │
│ │   Host names: techcorp.com, www.techcorp.com              │      │
│ │   Certificate: cert-techcorp                              │      │
│ │   → Routing rule → bp-web                                │      │
│ │                                                           │      │
│ │ Listener: listener-api                                    │      │
│ │   Type: Multi-site                                        │      │
│ │   Frontend: Same public IP                                │      │
│ │   Port: 443 (HTTPS)                                       │      │
│ │   Host names: api.techcorp.com                            │      │
│ │   Certificate: cert-techcorp (same wildcard cert)        │      │
│ │   → Routing rule → bp-api                                │      │
│ │                                                           │      │
│ │ Listener: listener-admin                                  │      │
│ │   Type: Multi-site                                        │      │
│ │   Frontend: Same public IP                                │      │
│ │   Port: 443 (HTTPS)                                       │      │
│ │   Host names: admin.techcorp.com                          │      │
│ │   Certificate: cert-techcorp                              │      │
│ │   → Routing rule → bp-admin                              │      │
│ └───────────────────────────────────────────────────────────┘      │
│                                                                       │
│ ⚡ Wildcard hostnames supported: *.techcorp.com                     │
│                                                                       │
│ ═══════════════════════════════════════════════════════════════     │
│                                                                       │
│ SSL/TLS configuration:                                               │
│                                                                       │
│ Certificate sources:                                                 │
│ ├── Key Vault: Store PFX/PEM in Azure Key Vault                  │
│ │   ├── Auto-renewal when Key Vault cert is updated              │
│ │   ├── App GW needs managed identity with GET access            │
│ │   ├── Recommended for production!                               │
│ │   └── Portal: Listeners → HTTPS → Key Vault certificate      │
│ │                                                                 │
│ ├── Upload PFX: Upload directly to App GW                        │
│ │   ├── Requires PFX format with password                        │
│ │   ├── Must manually re-upload on renewal                       │
│ │   └── Portal: Listeners → HTTPS → Upload certificate         │
│ │                                                                 │
│ └── ⚠️ No equivalent to AWS ACM (free auto-managed certs).        │
│       You must bring your own certificate.                        │
│       Use Let's Encrypt + Key Vault for free certs.              │
│                                                                       │
│ SSL policy:                                                          │
│ ├── Predefined:                                                   │
│ │   ├── AppGwSslPolicy20220101: TLS 1.2 (recommended)           │
│ │   ├── AppGwSslPolicy20220101S: TLS 1.2 (stricter ciphers)    │
│ │   └── AppGwSslPolicy20170401S: TLS 1.2                        │
│ ├── Custom: Choose TLS version + cipher suites                   │
│ └── Set per listener or globally on App GW                       │
│                                                                       │
│ SSL profile (per-listener override):                                │
│ ├── Different SSL policies per listener                           │
│ ├── Different trusted client CAs (for mTLS)                     │
│ └── Portal: Listeners → SSL profile                              │
│                                                                       │
│ Mutual TLS (mTLS):                                                  │
│ ├── Client must present certificate                               │
│ ├── App GW validates client cert against trusted CA              │
│ ├── Use for: B2B APIs, zero-trust, high-security apps           │
│ ├── Portal: SSL profile → Client authentication → Upload CA    │
│ └── Client cert info forwarded to backend via headers            │
│                                                                       │
│ End-to-end SSL:                                                      │
│ ├── Client → App GW (HTTPS) → Backend (HTTPS)                   │
│ ├── App GW re-encrypts traffic to backend                       │
│ ├── Backend Settings: Protocol = HTTPS, Port = 443              │
│ ├── Upload backend CA cert (for App GW to verify backend)       │
│ ├── Or: ☑ "Use well known CA certificate" (for trusted CAs)    │
│ └── ⚡ Not needed for most scenarios (HTTP to backend is fine    │
│       since traffic stays within VNet — already private)         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Backend Pools & Backend Settings

```
┌─────────────────────────────────────────────────────────────────────┐
│           BACKEND POOLS                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Backend types:                                                       │
│                                                                       │
│ 1. Virtual Machine (NIC-based):                                      │
│    ├── Select VMs by their network interface                       │
│    ├── VMs must be in same VNet (or peered VNet)                  │
│    └── Health probed directly on VM IP                             │
│                                                                       │
│ 2. VMSS (Virtual Machine Scale Sets):                                │
│    ├── Auto-scaling VM pool                                        │
│    ├── App GW auto-discovers scale set members                    │
│    └── Works with both Uniform and Flexible orchestration         │
│                                                                       │
│ 3. App Service:                                                      │
│    ├── Azure App Service / Web App                                │
│    ├── Add by FQDN: myapp.azurewebsites.net                     │
│    ├── ⚠️ CRITICAL: Must override host header!                     │
│    │   Backend Settings → Override with host name → Yes          │
│    │   → Pick from backend target                                │
│    │   Without this: App Service gets App GW IP as host →        │
│    │   returns 404 or wrong app!                                  │
│    └── Multi-tenant: Multiple apps on same infrastructure        │
│                                                                       │
│ 4. IP address or FQDN:                                               │
│    ├── Any IP address (on-premises, other clouds)                │
│    ├── FQDN: External origins, custom DNS names                   │
│    ├── Use for: Hybrid/multi-cloud load balancing                │
│    └── ⚠️ Must be reachable from App GW subnet                    │
│                                                                       │
│ 5. Private Endpoint (preview):                                       │
│    ├── Backend is a private endpoint                               │
│    ├── For privately-connected PaaS services                      │
│    └── Use for: Private App Service, private API Management      │
│                                                                       │
│ Empty backend pool:                                                  │
│ ├── No targets → App GW returns 502 Bad Gateway                  │
│ ├── Use with redirect rules (no backend needed for redirect)     │
│ └── Use with fixed response rewrites                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           BACKEND SETTINGS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Backend Settings (formerly "HTTP Settings") define HOW App GW     │
│ communicates with the backend.                                     │
│                                                                       │
│ Portal → App Gateway → Backend settings → Add                     │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Name:               [settings-web-prod]                      │   │
│ │                                                              │   │
│ │ Backend protocol:   [HTTP ▼]  / HTTPS                       │   │
│ │   HTTP: App GW → Backend on HTTP (most common)             │   │
│ │   HTTPS: End-to-end encryption (App GW re-encrypts)        │   │
│ │                                                              │   │
│ │ Backend port:       [8080]                                   │   │
│ │   Port your application listens on                          │   │
│ │                                                              │   │
│ │ Cookie-based affinity:                                       │   │
│ │   ○ Disabled (default — recommended for stateless apps)     │   │
│ │   ● Enabled                                                 │   │
│ │   Affinity cookie name: [ApplicationGatewayAffinity]        │   │
│ │   Cookie type: ○ Default  ● Custom                         │   │
│ │   If custom: Set-Cookie attributes (SameSite, Secure, etc.)│   │
│ │                                                              │   │
│ │ Connection draining:                                         │   │
│ │   ☑ Enable                                                  │   │
│ │   Drain timeout: [30] seconds                               │   │
│ │   (Time to complete in-flight requests during backend removal)│  │
│ │                                                              │   │
│ │ Request timeout:    [30] seconds                             │   │
│ │   How long App GW waits for backend response                │   │
│ │   Range: 1-86400 seconds                                    │   │
│ │   ⚠️ For long-running APIs (file upload, reports):            │   │
│ │      Increase to 120 or 300 seconds                         │   │
│ │                                                              │   │
│ │ Override backend path: [blank]                               │   │
│ │   Rewrite the URL path sent to backend                      │   │
│ │   Example: /api/* → backend receives /v2/*                 │   │
│ │                                                              │   │
│ │ Override with new host name:                                 │   │
│ │   ○ No                                                      │   │
│ │   ● Yes                                                     │   │
│ │   ● Pick host name from backend target                     │   │
│ │     (Auto-sets Host header to backend's FQDN)              │   │
│ │   ○ Override with specific domain name                     │   │
│ │     Host name: [app-prod.azurewebsites.net]                │   │
│ │                                                              │   │
│ │ ⚠️ Host override is REQUIRED for:                             │   │
│ │   ├── App Service backends (multi-tenant routing)           │   │
│ │   ├── Azure Functions backends                              │   │
│ │   └── Any backend that routes by Host header                │   │
│ │                                                              │   │
│ │ Use custom probe: [probe-web ▼]                             │   │
│ │                                                              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Health Probes

```
┌─────────────────────────────────────────────────────────────────────┐
│           HEALTH PROBES                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ App GW probes each backend to determine health.                    │
│ Unhealthy backends are removed from rotation.                      │
│                                                                       │
│ Default probe (auto-created):                                       │
│ ├── Protocol: HTTP or HTTPS (matches backend settings)            │
│ ├── Host: 127.0.0.1 (or backend settings host)                   │
│ ├── Path: / (root)                                                 │
│ ├── Interval: 30 seconds                                          │
│ ├── Timeout: 30 seconds                                           │
│ ├── Unhealthy threshold: 3                                         │
│ └── Match: HTTP 200-399 (any 2xx/3xx is healthy)                 │
│                                                                       │
│ Custom probe (recommended):                                         │
│                                                                       │
│ Portal → App Gateway → Health probes → Add                        │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Name:          [probe-web-prod]                              │   │
│ │ Protocol:      [HTTP ▼] / HTTPS                             │   │
│ │ Host:          [techcorp.com]                                │   │
│ │   ○ Pick from backend settings                              │   │
│ │   ● Specific host name                                     │   │
│ │                                                              │   │
│ │ Path:          [/health]                                     │   │
│ │ Port:          [● Use backend setting port]                 │   │
│ │                [○ Override: 8080]                            │   │
│ │                                                              │   │
│ │ Interval:      [10] seconds (how often to probe)            │   │
│ │ Timeout:       [30] seconds (wait for response)             │   │
│ │ Unhealthy threshold: [3] (failures before marking unhealthy)│   │
│ │                                                              │   │
│ │ ☑ Use probe matching conditions                             │   │
│ │   Body match:                                                │   │
│ │   Body contains: [Healthy]                                  │   │
│ │   HTTP response status code match:                          │   │
│ │   Status code ranges: [200-399]                             │   │
│ │                                                              │   │
│ │   ⚡ Body matching is POWERFUL!                               │   │
│ │   Your /health endpoint returns: {"status": "Healthy"}     │   │
│ │   Probe checks for "Healthy" in body.                      │   │
│ │   If your app is degraded, return {"status": "Degraded"}   │   │
│ │   → App GW marks it unhealthy → removes from pool.        │   │
│ │   AWS ALB does NOT have body matching.                     │   │
│ │                                                              │   │
│ │ Associated backend settings: [settings-web-prod ▼]         │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Troubleshooting unhealthy probes:                                   │
│ Portal → App Gateway → Backend health                              │
│ Shows: Each backend + health status + probe results               │
│                                                                       │
│ Common reasons for unhealthy:                                        │
│ 1. NSG blocking App GW subnet → backend subnet                   │
│ 2. Backend app not listening on expected port                     │
│ 3. /health returns 500 or non-2xx                                 │
│ 4. Timeout (app too slow to respond in 30s)                       │
│ 5. SSL certificate mismatch (HTTPS backends)                      │
│ 6. Host header wrong (App Service returns 404)                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Routing Rules & URL Path Maps

```
┌─────────────────────────────────────────────────────────────────────┐
│           ROUTING RULES                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Each listener MUST have exactly ONE routing rule.                  │
│ Routing rules link: Listener → Backend (or Redirect).             │
│                                                                       │
│ BASIC ROUTING:                                                       │
│ Listener → ALL traffic → one backend pool                         │
│ Simple, no path-based routing.                                     │
│                                                                       │
│ PATH-BASED ROUTING:                                                  │
│ Listener → URL Path Map → different backends per path             │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Routing rule: [rule-https-main]                              │   │
│ │ Priority: [100] (lower = evaluated first)                   │   │
│ │                                                              │   │
│ │ Listener: [listener-https ▼]                                │   │
│ │                                                              │   │
│ │ Backend targets:                                             │   │
│ │ Target type: ● Backend pool  ○ Redirection                 │   │
│ │                                                              │   │
│ │ ☑ Path-based routing (creates URL path map)                │   │
│ │                                                              │   │
│ │ Default backend: [bp-web ▼]                                 │   │
│ │ Default settings: [settings-web ▼]                          │   │
│ │                                                              │   │
│ │ Path rules:                                                  │   │
│ │ ┌──────────────┬───────────┬──────────────────────────┐    │   │
│ │ │ Path         │ Backend   │ Settings                  │    │   │
│ │ ├──────────────┼───────────┼──────────────────────────┤    │   │
│ │ │ /api/*       │ bp-api    │ settings-api             │    │   │
│ │ │ /api/v2/*    │ bp-api-v2 │ settings-api             │    │   │
│ │ │ /static/*    │ bp-static │ settings-static          │    │   │
│ │ │ /images/*    │ bp-cdn    │ settings-cdn             │    │   │
│ │ │ /admin/*     │ bp-admin  │ settings-admin           │    │   │
│ │ │ /ws/*        │ bp-ws     │ settings-ws (timeout=300)│    │   │
│ │ │ /* (default) │ bp-web    │ settings-web             │    │   │
│ │ └──────────────┴───────────┴──────────────────────────┘    │   │
│ │                                                              │   │
│ │ ⚡ Each path rule can have DIFFERENT backend settings!        │   │
│ │    Different ports, timeouts, host overrides per path.      │   │
│ │    AWS ALB can only change the target group, not settings.  │   │
│ │                                                              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Priority (v2):                                                       │
│ ├── Rules evaluated by priority (1-20000)                        │
│ ├── Lower number = higher priority                                │
│ ├── Within a rule: Path rules evaluated by specificity           │
│ │   /api/v2/* matches before /api/*                               │
│ └── Default backend catches unmatched paths                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Redirect Configuration

```
┌─────────────────────────────────────────────────────────────────────┐
│           REDIRECT                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ App Gateway can redirect traffic without any backend.              │
│                                                                       │
│ Redirect types:                                                      │
│ ├── Permanent (301): SEO-friendly permanent redirect              │
│ ├── Found (302): Temporary redirect                                │
│ ├── See Other (303): Redirect POST to GET                         │
│ └── Temporary Redirect (307): Preserve HTTP method                │
│                                                                       │
│ Redirect targets:                                                    │
│ ├── Listener: Redirect to another listener (HTTP→HTTPS)          │
│ ├── External site: Redirect to any URL                            │
│ └── Path rules can also redirect                                  │
│                                                                       │
│ Common setup — HTTP to HTTPS redirect:                              │
│                                                                       │
│ Listener: listener-http (port 80)                                   │
│ Routing rule: rule-redirect                                         │
│   Target type: Redirection                                          │
│   Redirect type: Permanent (301)                                    │
│   Redirect target: listener-https                                   │
│   ☑ Include path                                                    │
│   ☑ Include query string                                           │
│                                                                       │
│ Result:                                                              │
│ http://techcorp.com/api/users?page=2                               │
│ → 301 → https://techcorp.com/api/users?page=2                    │
│                                                                       │
│ Path-level redirect:                                                 │
│ /old-blog/* → 301 → https://blog.techcorp.com/{remaining}        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Rewrite Rules

```
┌─────────────────────────────────────────────────────────────────────┐
│           REWRITE RULES                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Modify HTTP request/response headers and URLs.                     │
│                                                                       │
│ Portal → App Gateway → Rewrites → Add rewrite set                 │
│                                                                       │
│ Rewrite set name: [rewrite-security-headers]                       │
│ Associate with routing rule: [rule-https-main ▼]                  │
│                                                                       │
│ ── Rewrite rule 1: Add security headers ──                         │
│ Rule name: [add-security-headers]                                   │
│ Sequence: [100]                                                     │
│                                                                       │
│ Conditions (optional):                                               │
│   Variable: [none — apply to all responses]                        │
│                                                                       │
│ Actions:                                                             │
│ Response header: Set                                                │
│   "Strict-Transport-Security" = "max-age=31536000"                │
│ Response header: Set                                                │
│   "X-Content-Type-Options" = "nosniff"                             │
│ Response header: Set                                                │
│   "X-Frame-Options" = "DENY"                                       │
│ Response header: Set                                                │
│   "Content-Security-Policy" = "default-src 'self'"                │
│ Response header: Delete                                             │
│   "Server" (remove IIS server header)                              │
│ Response header: Delete                                             │
│   "X-Powered-By" (remove ASP.NET header)                          │
│                                                                       │
│ ── Rewrite rule 2: URL rewrite ──                                  │
│ Rule name: [rewrite-api-path]                                       │
│ Sequence: [200]                                                     │
│                                                                       │
│ Conditions:                                                          │
│   Variable: var_uri_path                                            │
│   Operator: matches pattern                                        │
│   Pattern: /api/legacy/(.*)                                        │
│                                                                       │
│ Actions:                                                             │
│ URL rewrite:                                                        │
│   URL path: /api/v2/{var_uri_path_1}                               │
│   Query string: {var_query_string}                                 │
│                                                                       │
│ ── Rewrite rule 3: Add client IP ──                                │
│ Rule name: [add-client-ip]                                          │
│ Actions:                                                             │
│ Request header: Set                                                 │
│   "X-Real-IP" = {var_client_ip}                                   │
│ Request header: Set                                                 │
│   "X-Client-Port" = {var_client_port}                              │
│                                                                       │
│ Available server variables:                                          │
│ ├── {var_client_ip}: Client IP                                    │
│ ├── {var_client_port}: Client port                                │
│ ├── {var_host}: Host header                                       │
│ ├── {var_uri_path}: URI path                                      │
│ ├── {var_uri_path_1}: First capture group from condition          │
│ ├── {var_query_string}: Query string                              │
│ ├── {var_request_uri}: Full URI                                   │
│ ├── {var_server_port}: Server port                                │
│ ├── {http_req_HeaderName}: Any request header                    │
│ └── {http_resp_HeaderName}: Any response header                  │
│                                                                       │
│ ⚡ Conditional rewrites:                                              │
│    Only apply rewrite if condition matches.                        │
│    Example: Only add HSTS header for HTTPS responses.             │
│    Condition: var_server_port equals 443                           │
│    Action: Set Strict-Transport-Security header                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: WAF Policy (Deep Dive)

```
┌─────────────────────────────────────────────────────────────────────┐
│           WAF POLICY                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ WAF is available only with WAF_v2 SKU.                             │
│ WAF Policy is now a SEPARATE resource (not inline on App GW).     │
│                                                                       │
│ Portal → WAF policies → Create                                     │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ Policy for: [Application Gateway ▼]                          │   │
│ │   Options: Application Gateway, Front Door, CDN             │   │
│ │                                                              │   │
│ │ ── Basics ──                                                 │   │
│ │ Name: [waf-policy-prod]                                     │   │
│ │ Policy state: ☑ Enabled                                    │   │
│ │ Policy mode:                                                 │   │
│ │   ○ Detection (log only — don't block)                      │   │
│ │   ● Prevention (log + block)                                │   │
│ │                                                              │   │
│ │ ⚡ Start with Detection for 1-2 weeks.                       │   │
│ │    Review logs for false positives.                         │   │
│ │    Switch to Prevention after tuning.                       │   │
│ │                                                              │   │
│ │ ── Managed rules ──                                         │   │
│ │ Managed rule set:                                            │   │
│ │   ☑ OWASP 3.2 (default — latest stable)                   │   │
│ │   ☑ Microsoft_BotManagerRuleSet 1.1                        │   │
│ │                                                              │   │
│ │ OWASP 3.2 rule groups:                                      │   │
│ │ ┌────────────────────────────────────┬──────────┐          │   │
│ │ │ Rule group                         │ Status   │          │   │
│ │ ├────────────────────────────────────┼──────────┤          │   │
│ │ │ REQUEST-911-METHOD-ENFORCEMENT     │ Enabled  │          │   │
│ │ │ REQUEST-913-SCANNER-DETECTION      │ Enabled  │          │   │
│ │ │ REQUEST-920-PROTOCOL-ENFORCEMENT   │ Enabled  │          │   │
│ │ │ REQUEST-921-PROTOCOL-ATTACK        │ Enabled  │          │   │
│ │ │ REQUEST-930-APPLICATION-ATTACK-LFI │ Enabled  │          │   │
│ │ │ REQUEST-931-APPLICATION-ATTACK-RFI │ Enabled  │          │   │
│ │ │ REQUEST-932-APPLICATION-ATTACK-RCE │ Enabled  │          │   │
│ │ │ REQUEST-933-APPLICATION-ATTACK-PHP │ Enabled  │          │   │
│ │ │ REQUEST-941-APPLICATION-ATTACK-XSS │ Enabled  │          │   │
│ │ │ REQUEST-942-APPLICATION-ATTACK-SQLI│ Enabled  │          │   │
│ │ │ REQUEST-943-APPLICATION-ATTACK-SF  │ Enabled  │          │   │
│ │ │ REQUEST-944-APPLICATION-ATTACK-JAVA│ Enabled  │          │   │
│ │ └────────────────────────────────────┴──────────┘          │   │
│ │                                                              │   │
│ │ Per-rule override:                                           │   │
│ │ Expand any group → Toggle individual rules:                 │   │
│ │ ├── Enabled (default)                                       │   │
│ │ ├── Disabled (permanently skip this rule)                   │   │
│ │ ├── Log (detect but don't block — useful for noisy rules)  │   │
│ │ └── Anomaly score only                                      │   │
│ │                                                              │   │
│ │ ── Custom rules ──                                          │   │
│ │                                                              │   │
│ │ Custom rule 1: Geo-blocking                                 │   │
│ │ Name: [BlockCountries]                                      │   │
│ │ Priority: [1]                                                │   │
│ │ Rule type: ● Match                                          │   │
│ │ Match variables:                                             │   │
│ │   Variable: [RemoteAddr ▼]                                  │   │
│ │   Operator: [GeoMatch ▼]                                    │   │
│ │   Values: [CN, RU, KP]                                      │   │
│ │ Action: [Block ▼] (deny with 403)                          │   │
│ │                                                              │   │
│ │ Custom rule 2: Rate limiting                                │   │
│ │ Name: [RateLimitAPI]                                        │   │
│ │ Priority: [5]                                                │   │
│ │ Rule type: ● Rate limit                                     │   │
│ │ Rate limit duration: [1 minute ▼]                           │   │
│ │ Rate limit threshold: [100] requests                        │   │
│ │ Group rate limit by: [Client address ▼]                    │   │
│ │ Match conditions:                                            │   │
│ │   Variable: [RequestUri ▼]                                  │   │
│ │   Operator: [Contains ▼]                                    │   │
│ │   Value: [/api/]                                             │   │
│ │ Action: [Block ▼]                                           │   │
│ │                                                              │   │
│ │ Custom rule 3: IP allowlist                                 │   │
│ │ Name: [AllowOfficeIPs]                                      │   │
│ │ Priority: [0] (highest — evaluated first)                  │   │
│ │ Rule type: Match                                             │   │
│ │ Match variables:                                             │   │
│ │   Variable: [RemoteAddr ▼]                                  │   │
│ │   Operator: [IPMatch ▼]                                     │   │
│ │   Values: [203.0.113.0/24, 198.51.100.0/24]               │   │
│ │ Negate: ☑ (NOT matching these IPs → block)                │   │
│ │ Action: [Block ▼]                                           │   │
│ │ (Only office IPs allowed — all others blocked)             │   │
│ │                                                              │   │
│ │ ── Exclusions ──                                            │   │
│ │ (Handle false positives without disabling rules globally)  │   │
│ │                                                              │   │
│ │ Exclusion 1:                                                 │   │
│ │ Applies to: [REQUEST-942-SQLI ▼]                           │   │
│ │ Match variable: [Request body post args names ▼]           │   │
│ │ Operator: [Equals ▼]                                        │   │
│ │ Selector: [content]                                          │   │
│ │                                                              │   │
│ │ ⚡ This excludes the "content" POST field from SQL injection │   │
│ │    checks. Common for CMS apps where content may contain   │   │
│ │    SQL-like syntax (SELECT, FROM, WHERE).                  │   │
│ │                                                              │   │
│ │ ── Associations ──                                          │   │
│ │ Associate with:                                              │   │
│ │ ● Entire Application Gateway                                │   │
│ │ ○ Specific listeners (per-listener WAF)                    │   │
│ │ ○ Specific path rules (per-path WAF)                       │   │
│ │                                                              │   │
│ │ ⚡ Per-listener WAF: Different WAF rules for admin vs web!   │   │
│ │    admin.techcorp.com → strict WAF                          │   │
│ │    techcorp.com → standard WAF                              │   │
│ │                                                              │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Private Application Gateway & AKS Ingress

```
┌─────────────────────────────────────────────────────────────────────┐
│           PRIVATE APPLICATION GATEWAY                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ App GW can be internal-only (private frontend IP).                 │
│                                                                       │
│ Use for:                                                             │
│ ├── Internal web applications (not internet-facing)               │
│ ├── Internal API gateway                                           │
│ ├── Behind Azure Front Door or Azure Firewall                     │
│ └── Multi-tier architectures (Front Door → App GW → VMs)        │
│                                                                       │
│ Setup:                                                               │
│ Frontend IP: Private only                                           │
│ IP address: 10.0.4.10 (from App GW subnet)                        │
│ No public IP needed                                                 │
│                                                                       │
│ Private DNS:                                                         │
│ Register internal.techcorp.com → 10.0.4.10 in Private DNS Zone   │
│ Internal clients connect to internal.techcorp.com                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           AKS INGRESS CONTROLLER (AGIC)                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Application Gateway Ingress Controller (AGIC) uses App GW as      │
│ the Kubernetes Ingress controller for AKS.                         │
│                                                                       │
│ Instead of nginx-ingress or traefik, App GW handles Ingress.      │
│                                                                       │
│ Benefits:                                                            │
│ ├── WAF at ingress level                                          │
│ ├── No in-cluster ingress controller needed                       │
│ ├── Auto-scales with AKS pod changes                              │
│ ├── SSL termination at App GW                                     │
│ └── Path-based routing to K8s services                            │
│                                                                       │
│ Setup:                                                               │
│ AKS → Networking → Enable Application Gateway Ingress Controller │
│                                                                       │
│ Kubernetes Ingress resource:                                        │
│ apiVersion: networking.k8s.io/v1                                   │
│ kind: Ingress                                                       │
│ metadata:                                                            │
│   name: app-ingress                                                 │
│   annotations:                                                       │
│     kubernetes.io/ingress.class: azure/application-gateway        │
│ spec:                                                                │
│   rules:                                                             │
│   - host: app.techcorp.com                                         │
│     http:                                                            │
│       paths:                                                         │
│       - path: /api/*                                                │
│         pathType: Prefix                                             │
│         backend:                                                     │
│           service:                                                   │
│             name: api-service                                       │
│             port: {number: 8080}                                   │
│       - path: /*                                                    │
│         pathType: Prefix                                             │
│         backend:                                                     │
│           service:                                                   │
│             name: web-service                                       │
│             port: {number: 80}                                     │
│                                                                       │
│ AGIC translates K8s Ingress → App GW listeners, rules, backends  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Monitoring & Diagnostics

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Diagnostic settings:                                                 │
│ App GW → Diagnostic settings → Add diagnostic setting             │
│ ├── ApplicationGatewayAccessLog: All requests (hits, misses)     │
│ ├── ApplicationGatewayPerformanceLog: Performance data            │
│ ├── ApplicationGatewayFirewallLog: WAF events (blocks/allows)   │
│ ├── Send to: Log Analytics / Storage Account / Event Hub         │
│ └── Log Analytics recommended for queries (KQL)                   │
│                                                                       │
│ Key KQL queries:                                                     │
│                                                                       │
│ // WAF blocked requests                                              │
│ AzureDiagnostics                                                    │
│ | where ResourceType == "APPLICATIONGATEWAYS"                      │
│ | where Category == "ApplicationGatewayFirewallLog"                │
│ | where action_s == "Blocked"                                       │
│ | summarize count() by ruleId_s, bin(TimeGenerated, 1h)           │
│ | order by count_ desc                                              │
│                                                                       │
│ // Backend health status                                             │
│ AzureDiagnostics                                                    │
│ | where Category == "ApplicationGatewayAccessLog"                  │
│ | where httpStatus_d >= 500                                         │
│ | summarize count() by serverRouted_s, bin(TimeGenerated, 5m)     │
│                                                                       │
│ // Top client IPs                                                    │
│ AzureDiagnostics                                                    │
│ | where Category == "ApplicationGatewayAccessLog"                  │
│ | summarize count() by clientIP_s                                  │
│ | top 20 by count_                                                  │
│                                                                       │
│ Azure Monitor Metrics:                                               │
│ ├── Throughput: Bytes per second                                  │
│ ├── Total Requests: Request count                                 │
│ ├── Current Connections: Active connections                       │
│ ├── Healthy/Unhealthy Host Count: Per backend pool               │
│ ├── Response Status: 2xx, 3xx, 4xx, 5xx breakdown               │
│ ├── Backend Response Status: From backends                       │
│ ├── Backend Last Byte Response Time: Backend latency             │
│ ├── Failed Requests: Count of failed requests                    │
│ ├── Capacity Units: Current CU consumption                       │
│ └── Compute Units: Current compute consumption                   │
│                                                                       │
│ Key alerts:                                                          │
│ ├── Unhealthy Host Count > 0: Backend failure                    │
│ ├── 5xx Response > threshold: Application errors                  │
│ ├── Backend Response Time > 5s: Performance degradation          │
│ ├── Capacity Units > 75% of max: Scaling needed                  │
│ └── WAF blocked requests spike: Potential attack                  │
│                                                                       │
│ Backend Health blade:                                                │
│ Portal → App Gateway → Backend health                              │
│ Shows real-time status of each backend in each pool.               │
│ Includes: Status, server IP, port, health probe result.           │
│ ⚡ First place to check when things go wrong!                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Terraform Example

```hcl
# Dedicated subnet for App GW
resource "azurerm_subnet" "appgw" {
  name                 = "subnet-appgw"
  resource_group_name  = azurerm_resource_group.networking.name
  virtual_network_name = azurerm_virtual_network.prod.name
  address_prefixes     = ["10.0.4.0/24"]
}

# Public IP for App GW
resource "azurerm_public_ip" "appgw" {
  name                = "pip-appgw-prod"
  location            = azurerm_resource_group.networking.location
  resource_group_name = azurerm_resource_group.networking.name
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]
}

# WAF Policy
resource "azurerm_web_application_firewall_policy" "prod" {
  name                = "waf-policy-prod"
  location            = azurerm_resource_group.networking.location
  resource_group_name = azurerm_resource_group.networking.name

  policy_settings {
    enabled                     = true
    mode                        = "Prevention"
    request_body_check          = true
    max_request_body_size_in_kb = 128
    file_upload_limit_in_mb     = 100
  }

  managed_rules {
    managed_rule_set {
      type    = "OWASP"
      version = "3.2"

      rule_group_override {
        rule_group_name = "REQUEST-942-APPLICATION-ATTACK-SQLI"
        rule {
          id      = "942130"
          enabled = true
          action  = "Log"  # Log instead of block (false positive prone)
        }
      }
    }

    managed_rule_set {
      type    = "Microsoft_BotManagerRuleSet"
      version = "1.0"
    }

    exclusion {
      match_variable          = "RequestBodyPostArgNames"
      selector                = "content"
      selector_match_operator = "Equals"

      excluded_rule_set {
        type    = "OWASP"
        version = "3.2"
        rule_group {
          rule_group_name = "REQUEST-942-APPLICATION-ATTACK-SQLI"
          excluded_rules  = ["942130", "942440"]
        }
      }
    }
  }

  custom_rules {
    name      = "BlockCountries"
    priority  = 1
    rule_type = "MatchRule"
    action    = "Block"

    match_conditions {
      match_variables {
        variable_name = "RemoteAddr"
      }
      operator           = "GeoMatch"
      match_values       = ["CN", "RU", "KP"]
      negation_condition = false
    }
  }

  custom_rules {
    name      = "RateLimitAPI"
    priority  = 5
    rule_type = "RateLimitRule"
    action    = "Block"

    rate_limit_duration_in_minutes = 1
    rate_limit_threshold           = 100
    group_rate_limit_by            = "ClientAddr"

    match_conditions {
      match_variables {
        variable_name = "RequestUri"
      }
      operator     = "Contains"
      match_values = ["/api/"]
    }
  }
}

# Application Gateway
resource "azurerm_application_gateway" "prod" {
  name                = "appgw-prod"
  location            = azurerm_resource_group.networking.location
  resource_group_name = azurerm_resource_group.networking.name
  zones               = ["1", "2", "3"]
  firewall_policy_id  = azurerm_web_application_firewall_policy.prod.id
  enable_http2        = true

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

  # SSL Certificate (Key Vault)
  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.appgw.id]
  }

  ssl_certificate {
    name                = "cert-techcorp"
    key_vault_secret_id = azurerm_key_vault_certificate.techcorp.secret_id
  }

  ssl_policy {
    policy_type = "Predefined"
    policy_name = "AppGwSslPolicy20220101"
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
    probe_name            = "probe-web"

    connection_draining {
      enabled           = true
      drain_timeout_sec = 30
    }
  }

  backend_http_settings {
    name                                = "settings-api"
    cookie_based_affinity               = "Disabled"
    port                                = 443
    protocol                            = "Https"
    request_timeout                     = 60
    pick_host_name_from_backend_address = true
    probe_name                          = "probe-api"
  }

  # Health probes
  probe {
    name                = "probe-web"
    protocol            = "Http"
    host                = "techcorp.com"
    path                = "/health"
    interval            = 10
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

  # HTTPS listener (multi-site: techcorp.com)
  http_listener {
    name                           = "listener-web-https"
    frontend_ip_configuration_name = "fe-public"
    frontend_port_name             = "port-443"
    protocol                       = "Https"
    ssl_certificate_name           = "cert-techcorp"
    host_names                     = ["techcorp.com", "www.techcorp.com"]
  }

  # HTTPS listener (multi-site: api.techcorp.com)
  http_listener {
    name                           = "listener-api-https"
    frontend_ip_configuration_name = "fe-public"
    frontend_port_name             = "port-443"
    protocol                       = "Https"
    ssl_certificate_name           = "cert-techcorp"
    host_names                     = ["api.techcorp.com"]
  }

  # HTTP listener (redirect)
  http_listener {
    name                           = "listener-http"
    frontend_ip_configuration_name = "fe-public"
    frontend_port_name             = "port-80"
    protocol                       = "Http"
  }

  # Routing: Web (path-based)
  request_routing_rule {
    name               = "rule-web"
    priority           = 100
    rule_type          = "PathBasedRouting"
    http_listener_name = "listener-web-https"
    url_path_map_name  = "paths-web"
  }

  # Routing: API
  request_routing_rule {
    name                       = "rule-api"
    priority                   = 200
    rule_type                  = "Basic"
    http_listener_name         = "listener-api-https"
    backend_address_pool_name  = "bp-api"
    backend_http_settings_name = "settings-api"
  }

  # Routing: HTTP → HTTPS redirect
  request_routing_rule {
    name                        = "rule-http-redirect"
    priority                    = 300
    rule_type                   = "Basic"
    http_listener_name          = "listener-http"
    redirect_configuration_name = "redirect-https"
  }

  # URL Path Map
  url_path_map {
    name                               = "paths-web"
    default_backend_address_pool_name  = "bp-web"
    default_backend_http_settings_name = "settings-web"

    path_rule {
      name                       = "api"
      paths                      = ["/api/*"]
      backend_address_pool_name  = "bp-api"
      backend_http_settings_name = "settings-api"
    }
  }

  # Redirect config
  redirect_configuration {
    name                 = "redirect-https"
    redirect_type        = "Permanent"
    target_listener_name = "listener-web-https"
    include_path         = true
    include_query_string = true
  }

  # Rewrite rules
  rewrite_rule_set {
    name = "security-headers"

    rewrite_rule {
      name          = "add-hsts"
      rule_sequence = 100

      response_header_configuration {
        header_name  = "Strict-Transport-Security"
        header_value = "max-age=31536000; includeSubDomains"
      }

      response_header_configuration {
        header_name  = "X-Content-Type-Options"
        header_value = "nosniff"
      }

      response_header_configuration {
        header_name  = "Server"
        header_value = ""
      }
    }
  }
}
```

---

## Part 12: Real-World Patterns

### Startup

```
Setup: 1 Application Gateway (WAF_v2)

Frontend: 1 public IP
Listeners:
├── HTTPS:443 (basic — single domain)
└── HTTP:80 → redirect to HTTPS

Backend pools:
├── bp-web: 2 VMs or App Service
└── bp-api: App Service (FQDN)

Routing: Path-based
├── /api/* → bp-api (settings with host override)
└── /* → bp-web

WAF: Detection mode for 2 weeks → Prevention
├── OWASP 3.2 managed rules
├── Rate limit: 100 req/min per IP
└── No geo-blocking (need global users)

Autoscale: min=2, max=5
Monitoring: Diagnostic settings → Log Analytics

Cost: ~$200/month (WAF_v2 min=2)

Alternative: Just App Service (no App GW) = ~$55/month
Add App GW when you need WAF or multi-backend routing.
```

### Mid-Size

```
Setup: 1 Application Gateway (WAF_v2)

Listeners (multi-site):
├── techcorp.com:443 → path-based routing
├── api.techcorp.com:443 → bp-api
├── admin.techcorp.com:443 → bp-admin
└── :80 → redirect to HTTPS

Path-based routing (techcorp.com):
├── /api/* → bp-api (App Service, host override)
├── /static/* → bp-cdn (Storage, settings-cdn)
├── /ws/* → bp-websocket (settings with 300s timeout)
└── /* → bp-web (VMSS)

WAF policy:
├── Prevention mode
├── OWASP 3.2 + Bot Manager 1.1
├── Custom: Block high-risk countries
├── Custom: Rate limit /api/ (100 req/min)
├── Custom: Rate limit /api/login (10 req/min)
├── Exclusions: Content fields from SQLi checks
└── Per-listener: Stricter policy on admin

Rewrite rules:
├── Security headers (HSTS, X-Content-Type-Options)
├── Remove Server + X-Powered-By headers
└── Add X-Real-IP header for backends

SSL: Key Vault certs (auto-rotation)
Autoscale: min=2, max=15

Cost: ~$300-500/month
```

### Enterprise

```
Setup:
├── Azure Front Door (global) → regional App GWs
├── App GW per region (WAF_v2, 2-3 regions)
├── Private App GW for internal APIs
└── AGIC for AKS clusters

Public App GW (per region):
├── 20+ multi-site listeners
├── Path-based routing per microservice
├── WAF policy:
│   ├── OWASP 3.2 + Bot Manager
│   ├── Geo-blocking (compliance)
│   ├── Rate limiting (tiered by endpoint)
│   ├── Custom rules for business logic
│   └── Per-listener WAF overrides
├── mTLS on partner API listeners
├── End-to-end SSL for PCI compliance
├── Key Vault certs (auto-rotation)
├── Rewrite rules: Security headers, URL rewrites
└── Autoscale: min=3, max=30

Private App GW:
├── Internal frontend IP only
├── Microservice routing (20+ backend pools)
├── Backend: AKS via AGIC
└── No WAF needed (internal traffic)

AGIC (AKS):
├── App GW as K8s Ingress controller
├── Auto-configures listeners from Ingress resources
├── WAF protection for K8s workloads
└── SSL termination at App GW

Monitoring:
├── Diagnostic logs → Log Analytics
├── WAF logs → custom KQL dashboards
├── Azure Workbooks for executive reports
├── Alerts: Unhealthy backends, WAF blocks, latency
├── Backend Health monitoring (automated checks)
└── Monthly WAF rule tuning review

Cost: ~$1000-3000/month (multiple App GWs)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| SKUs | Standard_v2, WAF_v2 (v1 deprecated) |
| Layer | 7 (HTTP/HTTPS) |
| Scope | Regional |
| Subnet | Dedicated subnet required (/24 recommended) |
| Autoscaling | min/max instances (v2 only) |
| Zones | 1, 2, 3 (zone-redundant) |
| Listeners | Basic (single site), Multi-site (host-based) |
| Routing | Basic, Path-based (URL path maps) |
| WAF | OWASP 3.2, Bot Manager, custom rules, exclusions |
| SSL | PFX upload or Key Vault (no ACM equivalent) |
| mTLS | Supported (SSL profiles) |
| Backends | VM, VMSS, App Service, IP/FQDN |
| Health probes | HTTP/HTTPS with body matching |
| Rewrite | Headers + URL (request & response) |
| WebSocket | Supported (natively) |
| HTTP/2 | Supported (frontend only) |
| AGIC | AKS Ingress Controller |
| Cost | ~$175/mo (Standard_v2) or ~$262/mo (WAF_v2) + CU |
| AWS equivalent | ALB + WAF (separate) |
| GCP equivalent | External Application LB + Cloud Armor (separate) |

---

## What's Next?

In the next chapter, we'll cover Azure Firewall — the managed network firewall service.

→ Next: [Chapter 13: Azure Firewall](13-azure-firewall.md)

---

*Last Updated: May 2026*
