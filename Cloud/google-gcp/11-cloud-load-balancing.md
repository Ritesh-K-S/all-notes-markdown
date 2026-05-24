# Chapter 11: Cloud Load Balancing (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: GCP Load Balancer Types](#part-1-gcp-load-balancer-types)
- [Part 2: External Application Load Balancer (Global HTTP/S)](#part-2-external-application-load-balancer-global-https)
- [Part 3: Network Load Balancers](#part-3-network-load-balancers)
- [Part 4: Backend Services & NEGs](#part-4-backend-services--negs)
- [Part 5: Terraform Example](#part-5-terraform-example)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Console Walkthrough: Deleting a Load Balancer](#console-walkthrough-deleting-a-load-balancer)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud Load Balancing is a fully distributed, software-defined, managed service. Unlike AWS where you choose between ALB/NLB, GCP offers a matrix of load balancer types based on traffic type (HTTP vs TCP/UDP), scope (global vs regional), and facing (external vs internal).

```
What you'll learn:
├── GCP Load Balancer Types (the full matrix)
├── External Application LB (Global HTTP/S) — most common
│   ├── Frontend (IP + cert)
│   ├── URL maps (host/path routing)
│   ├── Backend services & backend buckets
│   ├── Health checks
│   └── All fields explained
├── External Proxy Network LB (Global TCP/SSL)
├── External Passthrough Network LB (Regional TCP/UDP)
├── Internal Application LB (Internal HTTP/S)
├── Internal Passthrough Network LB (Internal TCP/UDP)
├── Cross-Region Internal Application LB
├── Backend Services vs NEGs
├── Traffic Management (advanced routing)
├── Cloud Armor integration
└── Real-world patterns
```

---

## Part 1: GCP Load Balancer Types

```
┌─────────────────────────────────────────────────────────────────────┐
│          GCP LOAD BALANCER MATRIX                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GCP names are verbose. Here's the simplified matrix:               │
│                                                                       │
│ ┌────────────────────────────┬──────────┬──────────┬─────────────┐ │
│ │ Load Balancer              │ Layer    │ Scope    │ Facing      │ │
│ ├────────────────────────────┼──────────┼──────────┼─────────────┤ │
│ │ External Application LB    │ 7 (HTTP) │ Global   │ External    │ │
│ │ (Global HTTP/S LB)         │          │          │             │ │
│ │                            │          │          │             │ │
│ │ Regional External App LB   │ 7 (HTTP) │ Regional │ External    │ │
│ │                            │          │          │             │ │
│ │ Internal Application LB    │ 7 (HTTP) │ Regional │ Internal    │ │
│ │ (Internal HTTP/S LB)       │          │          │             │ │
│ │                            │          │          │             │ │
│ │ Cross-region Internal      │ 7 (HTTP) │ Global   │ Internal    │ │
│ │ Application LB             │          │          │             │ │
│ │                            │          │          │             │ │
│ │ External Proxy Network LB  │ 4 (TCP)  │ Global   │ External    │ │
│ │ (TCP/SSL Proxy LB)         │          │          │             │ │
│ │                            │          │          │             │ │
│ │ Regional Ext Proxy Net LB  │ 4 (TCP)  │ Regional │ External    │ │
│ │                            │          │          │             │ │
│ │ External Passthrough       │ 4 (TCP/  │ Regional │ External    │ │
│ │ Network LB                 │   UDP)   │          │             │ │
│ │ (Network LB)               │          │          │             │ │
│ │                            │          │          │             │ │
│ │ Internal Passthrough       │ 4 (TCP/  │ Regional │ Internal    │ │
│ │ Network LB                 │   UDP)   │          │             │ │
│ │ (Internal TCP/UDP LB)      │          │          │             │ │
│ └────────────────────────────┴──────────┴──────────┴─────────────┘ │
│                                                                       │
│ Quick decision guide:                                                │
│ ├── HTTP/HTTPS traffic → Application LB (Layer 7)                 │
│ ├── TCP/UDP traffic → Network LB (Layer 4)                        │
│ ├── Global (multi-region) → External Application or Proxy Net LB │
│ ├── Single region → Regional variant                              │
│ ├── Internet-facing → External                                    │
│ ├── Internal services → Internal                                  │
│ └── Passthrough = preserves source IP, no proxy                  │
│                                                                       │
│ Mapping to AWS:                                                      │
│ ┌──────────────────────────┬─────────────────────────────────────┐ │
│ │ GCP                      │ AWS Equivalent                      │ │
│ ├──────────────────────────┼─────────────────────────────────────┤ │
│ │ External Application LB  │ ALB (internet-facing) + CloudFront │ │
│ │ Internal Application LB  │ ALB (internal)                     │ │
│ │ External Proxy Network   │ NLB (TLS listener)                 │ │
│ │ External Passthrough Net │ NLB (TCP/UDP listener)             │ │
│ │ Internal Passthrough Net │ NLB (internal)                     │ │
│ └──────────────────────────┴─────────────────────────────────────┘ │
│                                                                       │
│ ⚡ Key GCP difference: Global HTTP/S LB has a SINGLE Anycast IP   │
│    that works worldwide! AWS ALB is regional — you need Route 53  │
│    latency-based routing to achieve similar global reach.          │
│                                                                       │
│ ⚡ GCP's Global HTTP/S LB also includes CDN (Cloud CDN) and       │
│    WAF (Cloud Armor) integration — it's an all-in-one.            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: External Application Load Balancer (Global HTTP/S)

This is the most commonly used GCP load balancer. Covered extensively in Chapter 10 (Cloud CDN), here we focus on the load balancing aspects.

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│     EXTERNAL APPLICATION LB ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Components (bottom-up):                                              │
│                                                                       │
│ ┌──────────────────┐                                                │
│ │ Backend Services  │  Your actual servers/containers               │
│ │ (or Backend       │  ├── Instance Groups (GCE VMs)               │
│ │  Buckets for GCS) │  ├── NEGs (GKE pods, Cloud Run, etc.)       │
│ │                    │  └── Health checks attached                  │
│ └────────┬───────────┘                                               │
│          │                                                            │
│ ┌────────▼───────────┐                                               │
│ │ URL Map            │  Routing rules                               │
│ │ (Host + Path rules)│  ├── api.techcorp.com → bs-api              │
│ │                    │  ├── /static/* → bb-static                   │
│ │                    │  └── /* → bs-web (default)                   │
│ └────────┬───────────┘                                               │
│          │                                                            │
│ ┌────────▼───────────┐                                               │
│ │ Target Proxy       │  Protocol handling                           │
│ │ (HTTPS Proxy)      │  ├── SSL certificate                        │
│ │                    │  ├── SSL policy (TLS version)                │
│ │                    │  └── QUIC/HTTP3                              │
│ └────────┬───────────┘                                               │
│          │                                                            │
│ ┌────────▼───────────┐                                               │
│ │ Forwarding Rule    │  Frontend                                    │
│ │ (Frontend)         │  ├── Global static IP                        │
│ │                    │  ├── Port (80/443)                            │
│ │                    │  └── Entry point for traffic                 │
│ └────────────────────┘                                               │
│                                                                       │
│ Traffic flow:                                                        │
│ User → Google Edge (Anycast IP) → Forwarding Rule →               │
│ Target Proxy (SSL termination) → URL Map (routing) →              │
│ Backend Service → Health-checked targets                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating External Application LB (Full Walkthrough)

```
Console → Network services → Load balancing → Create load balancer

┌─────────────────────────────────────────────────────────────────┐
│           CHOOSE LOAD BALANCER TYPE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Type of traffic:                                                │
│   ● HTTP(S)                                                     │
│   ○ TCP                                                        │
│   ○ UDP                                                        │
│                                                                   │
│ Internet facing or internal:                                    │
│   ● From Internet to my VMs or serverless services             │
│   ○ Only between my VMs or serverless services                │
│                                                                   │
│ Global or regional:                                             │
│   ● Best for global workloads (Global external Application LB)│
│   ○ Best for regional workloads                                │
│                                                                   │
│ [Configure]                                                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Step 1: Frontend Configuration
──────────────────────────────
┌─────────────────────────────────────────────────────────────────┐
│           FRONTEND CONFIGURATION                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:      [fe-prod-https]                                     │
│ Protocol:  [HTTPS ▼]                                           │
│ IP version: [IPv4 ▼]                                           │
│ IP address: [ip-prod-lb ▼] (reserved static IP)               │
│             [Create IP address] (reserves new Anycast IP)      │
│ Port:      [443]                                                │
│                                                                   │
│ Certificate:                                                    │
│   [Create a new certificate ▼]                                 │
│   ● Google-managed certificate                                 │
│   ○ Self-managed certificate                                  │
│                                                                   │
│   Domain: [techcorp.com]                                       │
│   [Add domain]: [www.techcorp.com]                             │
│   [Add domain]: [api.techcorp.com]                             │
│                                                                   │
│   ⚡ Google-managed certs are FREE and auto-renewed!            │
│   ⚠️ DNS must be pointed to LB IP for validation.              │
│                                                                   │
│ SSL policy: [ssl-policy-modern ▼]                              │
│   ├── COMPATIBLE: TLS 1.0+ (widest support)                  │
│   ├── MODERN: TLS 1.2+ (recommended)                         │
│   ├── RESTRICTED: TLS 1.2+ with limited ciphers              │
│   └── CUSTOM: Choose your own                                 │
│                                                                   │
│ QUIC negotiation: ☑ Enabled (HTTP/3)                          │
│                                                                   │
│ [Done]                                                           │
│                                                                   │
│ Add HTTP frontend for redirect:                                 │
│   Name: fe-prod-http                                            │
│   Protocol: HTTP                                                │
│   Port: 80                                                      │
│   IP: same as HTTPS                                             │
│   → URL map set to redirect to HTTPS                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Step 2: Backend Configuration
─────────────────────────────
┌─────────────────────────────────────────────────────────────────┐
│           BACKEND SERVICE                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:            [bs-web-prod]                                  │
│ Backend type:    [Instance group ▼]                            │
│                  Options:                                       │
│                  ├── Instance group (GCE VMs)                  │
│                  ├── Network endpoint group (GKE, Cloud Run)  │
│                  ├── Internet NEG (external origin)            │
│                  └── Serverless NEG (Cloud Functions/Run)      │
│                                                                   │
│ Protocol:        [HTTP ▼] / HTTPS / HTTP/2 / gRPC             │
│ Named port:      [http]                                        │
│ Timeout:         [30] seconds                                   │
│                                                                   │
│ Backends:                                                        │
│ ┌─────────────────────────────────────────────────────────┐    │
│ │ Backend 1:                                               │    │
│ │ Instance group: [ig-web-asia ▼]                        │    │
│ │ Port numbers: [8080]                                    │    │
│ │ Balancing mode: [Utilization ▼]                        │    │
│ │   Options:                                              │    │
│ │   ├── Utilization: Based on CPU usage                  │    │
│ │   ├── Rate: Based on requests per second               │    │
│ │   └── Connection: Based on concurrent connections      │    │
│ │ Max utilization: [80%]                                 │    │
│ │ Max RPS: [10000] (per instance)                        │    │
│ │ Capacity scaler: [1.0] (0-1, to drain: set to 0)     │    │
│ │                                                         │    │
│ │ Backend 2:                                               │    │
│ │ Instance group: [ig-web-us ▼]                          │    │
│ │ (traffic auto-routed to nearest region!)               │    │
│ └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│ Cloud CDN:   ☑ Enable (see Chapter 10)                        │
│ Cloud Armor: [policy-prod ▼]                                  │
│                                                                   │
│ Health check: [Create health check ▼]                          │
│ ┌─────────────────────────────────────────────────────────┐    │
│ │ Name:            [hc-web-https]                         │    │
│ │ Protocol:        [HTTP ▼]                              │    │
│ │ Port:            [8080]                                │    │
│ │ Request path:    [/health]                             │    │
│ │ Check interval:  [10] seconds                          │    │
│ │ Timeout:         [5] seconds                           │    │
│ │ Healthy threshold:   [2] consecutive successes        │    │
│ │ Unhealthy threshold: [3] consecutive failures         │    │
│ │                                                         │    │
│ │ Logs: ☑ Enable health check logs                      │    │
│ │ (Useful for debugging why instances are unhealthy)     │    │
│ └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│ Session affinity:                                                │
│   [None ▼]                                                     │
│   Options:                                                      │
│   ├── None: No stickiness                                     │
│   ├── Client IP: Same IP → same backend                      │
│   ├── Generated cookie: GCLB cookie (like ALB sticky)        │
│   ├── Header field: Custom header value                       │
│   └── HTTP cookie: Your app's cookie                          │
│                                                                   │
│ Connection draining timeout: [300] seconds                     │
│                                                                   │
│ Logging:                                                        │
│   ☑ Enable logging                                             │
│   Sample rate: [1.0] (100% of requests)                       │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Step 3: Host and Path Rules (URL Map)
─────────────────────────────────────
┌─────────────────────────────────────────────────────────────────┐
│           URL MAP (ROUTING RULES)                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Default backend: [bs-web-prod ▼]                               │
│                                                                   │
│ Host and path rules:                                            │
│                                                                   │
│ ┌──────────────────────┬──────────┬────────────────────────┐   │
│ │ Host                 │ Path     │ Backend                │   │
│ ├──────────────────────┼──────────┼────────────────────────┤   │
│ │ api.techcorp.com     │ /*       │ bs-api-prod            │   │
│ │ techcorp.com         │ /api/*   │ bs-api-prod            │   │
│ │ techcorp.com         │ /static/*│ bb-static-assets       │   │
│ │ techcorp.com         │ /*       │ bs-web-prod (default)  │   │
│ │ admin.techcorp.com   │ /*       │ bs-admin               │   │
│ └──────────────────────┴──────────┴────────────────────────┘   │
│                                                                   │
│ ⚡ Equivalent to AWS ALB listener rules!                        │
│    But defined as a separate URL Map resource.                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Advanced Traffic Management

```
┌─────────────────────────────────────────────────────────────────────┐
│           ADVANCED TRAFFIC MANAGEMENT                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ GCP URL maps support advanced routing (via YAML or Terraform):     │
│                                                                       │
│ 1. TRAFFIC SPLITTING (canary/weighted routing):                     │
│    pathRules:                                                        │
│    - paths: ["/api/*"]                                               │
│      routeAction:                                                    │
│        weightedBackendServices:                                      │
│        - backendService: bs-api-v1                                   │
│          weight: 90                                                  │
│        - backendService: bs-api-v2                                   │
│          weight: 10                                                  │
│                                                                       │
│    ⚡ Like AWS weighted target groups, but at URL map level!        │
│    Use for canary deployments, A/B testing.                        │
│                                                                       │
│ 2. URL REWRITE:                                                      │
│    pathRules:                                                        │
│    - paths: ["/api/v2/*"]                                            │
│      routeAction:                                                    │
│        urlRewrite:                                                   │
│          pathPrefixRewrite: "/v2/"                                   │
│                                                                       │
│ 3. HEADER-BASED ROUTING:                                             │
│    routeRules:                                                       │
│    - matchRules:                                                     │
│      - headerMatches:                                                │
│        - headerName: "X-Canary"                                     │
│          exactMatch: "true"                                         │
│      service: bs-api-canary                                         │
│                                                                       │
│ 4. REQUEST MIRRORING:                                                │
│    pathRules:                                                        │
│    - paths: ["/api/*"]                                               │
│      routeAction:                                                    │
│        requestMirrorPolicy:                                          │
│          backendService: bs-api-shadow                               │
│        (sends copy of traffic to shadow service for testing)        │
│                                                                       │
│ 5. RETRY POLICY:                                                     │
│    routeAction:                                                      │
│      retryPolicy:                                                    │
│        retryConditions: ["5xx", "deadline-exceeded"]                │
│        numRetries: 3                                                 │
│        perTryTimeout: "5s"                                           │
│                                                                       │
│ 6. FAULT INJECTION (testing):                                        │
│    routeAction:                                                      │
│      faultInjectionPolicy:                                           │
│        delay:                                                        │
│          fixedDelay: "2s"                                            │
│          percentage: 10                                              │
│        abort:                                                        │
│          httpStatus: 503                                             │
│          percentage: 5                                               │
│                                                                       │
│ 7. CORS:                                                             │
│    routeAction:                                                      │
│      corsPolicy:                                                     │
│        allowOrigins: ["https://techcorp.com"]                       │
│        allowMethods: ["GET", "POST"]                                │
│        allowHeaders: ["Authorization", "Content-Type"]             │
│        maxAge: 3600                                                  │
│                                                                       │
│ ⚡ These features are like a mini service mesh at the LB level!     │
│    AWS ALB doesn't have traffic splitting, mirroring,              │
│    retry policies, or fault injection — you need App Mesh for that.│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Network Load Balancers

### External Passthrough Network LB

```
┌─────────────────────────────────────────────────────────────────────┐
│     EXTERNAL PASSTHROUGH NETWORK LB                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Layer 4, regional, external. Similar to AWS NLB.                   │
│ Traffic passes through (not proxied) — preserves source IP.        │
│                                                                       │
│ Use for:                                                             │
│ ├── TCP/UDP services (databases, gaming, VoIP)                    │
│ ├── Need source IP preservation                                   │
│ ├── Non-HTTP protocols                                             │
│ └── Very high performance                                          │
│                                                                       │
│ Key differences from Application LB:                                │
│ ├── Regional only (not global)                                    │
│ ├── No URL routing (Layer 4, can't see HTTP path)                 │
│ ├── Preserves source IP (no proxy)                                │
│ ├── Supports TCP, UDP, ESP, ICMP                                  │
│ ├── No SSL termination (passthrough)                              │
│ └── Uses target pools or backend services                         │
│                                                                       │
│ Create:                                                              │
│ Console → Load balancing → Create → TCP → From internet →         │
│ No proxy (passthrough)                                              │
│                                                                       │
│ gcloud compute forwarding-rules create nlb-prod-tcp \               │
│   --region=asia-south1 \                                             │
│   --ports=3306 \                                                      │
│   --backend-service=bs-mysql \                                        │
│   --load-balancing-scheme=EXTERNAL                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### External Proxy Network LB (TCP/SSL Proxy)

```
┌─────────────────────────────────────────────────────────────────────┐
│     EXTERNAL PROXY NETWORK LB (TCP/SSL Proxy)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Layer 4, global, proxy-based. SSL termination for TCP.             │
│                                                                       │
│ Use for:                                                             │
│ ├── Global TCP traffic (not HTTP)                                 │
│ ├── SSL termination for non-HTTP (e.g., databases over TLS)      │
│ ├── Global anycast IP for TCP services                            │
│ └── When you need proxy behavior (client IP in PROXY protocol)   │
│                                                                       │
│ Two variants:                                                        │
│ ├── TCP Proxy: Plain TCP, global                                  │
│ └── SSL Proxy: TCP with SSL termination, global                   │
│                                                                       │
│ Example: Global Redis or PostgreSQL access over TLS               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Internal Passthrough Network LB

```
┌─────────────────────────────────────────────────────────────────────┐
│     INTERNAL PASSTHROUGH NETWORK LB                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Layer 4, regional, internal only. Equivalent to AWS internal NLB. │
│                                                                       │
│ Use for:                                                             │
│ ├── Internal TCP/UDP services                                     │
│ ├── Database load balancing (MySQL, PostgreSQL replicas)          │
│ ├── Internal DNS, LDAP, etc.                                      │
│ ├── High-availability for VM-based services                       │
│ └── Next-hop for routes (send all traffic through firewall VMs)  │
│                                                                       │
│ ⚡ Internal TCP LB as next-hop:                                     │
│    Route: 0.0.0.0/0 → next-hop = Internal LB (firewall)         │
│    All traffic from a subnet goes through firewall VMs!          │
│    This is unique to GCP — no AWS equivalent.                    │
│                                                                       │
│ Create:                                                              │
│ gcloud compute forwarding-rules create ilb-db-prod \                │
│   --region=asia-south1 \                                             │
│   --subnet=subnet-private \                                          │
│   --ports=5432 \                                                      │
│   --backend-service=bs-postgres \                                     │
│   --load-balancing-scheme=INTERNAL                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Internal Application LB

```
┌─────────────────────────────────────────────────────────────────────┐
│     INTERNAL APPLICATION LB                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Layer 7, regional, internal. Equivalent to AWS internal ALB.       │
│                                                                       │
│ Use for:                                                             │
│ ├── Internal microservice routing (service mesh alternative)      │
│ ├── Internal API gateway                                           │
│ ├── Host/path-based routing for internal services                 │
│ └── gRPC load balancing between internal services                 │
│                                                                       │
│ Same features as External App LB but internal:                     │
│ ├── URL maps (host/path routing)                                  │
│ ├── Traffic splitting (canary)                                    │
│ ├── Header-based routing                                          │
│ ├── Health checks                                                  │
│ └── Session affinity                                               │
│                                                                       │
│ Cross-region Internal Application LB:                               │
│ ├── Global scope, internal facing                                 │
│ ├── Single internal IP, routes to nearest region                  │
│ ├── Use for: Multi-region internal services                       │
│ └── New (doesn't exist in AWS without custom setup)              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Backend Services & NEGs

```
┌─────────────────────────────────────────────────────────────────────┐
│           BACKEND TYPES                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. INSTANCE GROUP (GCE VMs):                                         │
│    ├── Managed Instance Group (MIG): Auto-scaled VMs              │
│    │   Auto-healing, rolling updates, multi-zone                  │
│    ├── Unmanaged Instance Group: Manually managed VMs             │
│    └── Equivalent to: AWS ASG + Target Group                     │
│                                                                       │
│ 2. NETWORK ENDPOINT GROUP (NEG):                                     │
│    ├── Zonal NEG: Individual endpoints (GKE pods, VM ports)      │
│    ├── Internet NEG: External endpoints (non-GCP origins)        │
│    ├── Serverless NEG: Cloud Run, Cloud Functions, App Engine    │
│    ├── Hybrid NEG: On-premises or multi-cloud endpoints          │
│    └── Private Service Connect NEG: PSC endpoints                │
│                                                                       │
│    Serverless NEG example (Cloud Run):                              │
│    gcloud compute network-endpoint-groups create neg-api \          │
│      --region=asia-south1 \                                          │
│      --network-endpoint-type=serverless \                            │
│      --cloud-run-service=api-prod                                   │
│                                                                       │
│    ⚡ Serverless NEG = Load Balancer → Cloud Run/Functions         │
│       Gets: Custom domain, SSL, CDN, Cloud Armor — all for free! │
│       AWS equivalent: ALB + Lambda target group                   │
│                                                                       │
│ 3. BACKEND BUCKET (GCS):                                             │
│    ├── Direct to Cloud Storage bucket                             │
│    ├── For static content (HTML, CSS, JS, images)                │
│    └── AWS equivalent: CloudFront → S3 origin                    │
│                                                                       │
│ Balancing modes:                                                     │
│ ├── UTILIZATION: Based on CPU % of backend instances             │
│ ├── RATE: Based on RPS (requests per second) per instance       │
│ └── CONNECTION: Based on concurrent connections                  │
│                                                                       │
│ Capacity scaler (0-1):                                               │
│ ├── 1.0: Full capacity (100%)                                    │
│ ├── 0.5: Accept only 50% of max capacity                        │
│ ├── 0.0: Drain — stop sending new requests                      │
│ └── Useful for graceful migration between backends               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Terraform Example

```hcl
# Reserved static IP
resource "google_compute_global_address" "lb_ip" {
  name = "ip-prod-lb"
}

# Health check
resource "google_compute_health_check" "web" {
  name = "hc-web-http"

  http_health_check {
    port         = 8080
    request_path = "/health"
  }

  check_interval_sec  = 10
  timeout_sec         = 5
  healthy_threshold   = 2
  unhealthy_threshold = 3
}

# Backend service (web — GCE instances)
resource "google_compute_backend_service" "web" {
  name                  = "bs-web-prod"
  protocol              = "HTTP"
  port_name             = "http"
  timeout_sec           = 30
  health_checks         = [google_compute_health_check.web.id]
  load_balancing_scheme = "EXTERNAL_MANAGED"
  security_policy       = google_compute_security_policy.prod.id
  enable_cdn            = true

  backend {
    group           = google_compute_region_instance_group_manager.web_asia.instance_group
    balancing_mode  = "UTILIZATION"
    max_utilization = 0.8
    capacity_scaler = 1.0
  }

  backend {
    group           = google_compute_region_instance_group_manager.web_us.instance_group
    balancing_mode  = "UTILIZATION"
    max_utilization = 0.8
    capacity_scaler = 1.0
  }

  session_affinity = "NONE"
  connection_draining_timeout_sec = 300

  log_config {
    enable      = true
    sample_rate = 1.0
  }
}

# Backend service (API — Cloud Run via serverless NEG)
resource "google_compute_region_network_endpoint_group" "api_neg" {
  name                  = "neg-api-prod"
  region                = "asia-south1"
  network_endpoint_type = "SERVERLESS"

  cloud_run {
    service = google_cloud_run_v2_service.api.name
  }
}

resource "google_compute_backend_service" "api" {
  name                  = "bs-api-prod"
  protocol              = "HTTPS"
  timeout_sec           = 30
  load_balancing_scheme = "EXTERNAL_MANAGED"

  backend {
    group = google_compute_region_network_endpoint_group.api_neg.id
  }
}

# URL map (routing rules)
resource "google_compute_url_map" "main" {
  name            = "lb-prod"
  default_service = google_compute_backend_service.web.id

  host_rule {
    hosts        = ["api.techcorp.com"]
    path_matcher = "api"
  }

  host_rule {
    hosts        = ["techcorp.com", "www.techcorp.com"]
    path_matcher = "web"
  }

  path_matcher {
    name            = "api"
    default_service = google_compute_backend_service.api.id
  }

  path_matcher {
    name            = "web"
    default_service = google_compute_backend_service.web.id

    path_rule {
      paths   = ["/api/*"]
      service = google_compute_backend_service.api.id
    }

    path_rule {
      paths   = ["/static/*"]
      service = google_compute_backend_bucket.static.id
    }

    # Traffic splitting (canary)
    path_rule {
      paths = ["/api/v2/*"]
      route_action {
        weighted_backend_services {
          backend_service = google_compute_backend_service.api.id
          weight          = 90
        }
        weighted_backend_services {
          backend_service = google_compute_backend_service.api_v2.id
          weight          = 10
        }
      }
    }
  }
}

# Google-managed SSL certificate
resource "google_compute_managed_ssl_certificate" "main" {
  name = "cert-techcorp"
  managed {
    domains = ["techcorp.com", "www.techcorp.com", "api.techcorp.com"]
  }
}

# SSL policy
resource "google_compute_ssl_policy" "modern" {
  name            = "ssl-policy-modern"
  profile         = "MODERN"
  min_tls_version = "TLS_1_2"
}

# HTTPS proxy
resource "google_compute_target_https_proxy" "main" {
  name             = "proxy-prod-https"
  url_map          = google_compute_url_map.main.id
  ssl_certificates = [google_compute_managed_ssl_certificate.main.id]
  ssl_policy       = google_compute_ssl_policy.modern.id
  quic_override    = "ENABLE"
}

# Forwarding rule (frontend)
resource "google_compute_global_forwarding_rule" "https" {
  name                  = "fe-prod-https"
  ip_address            = google_compute_global_address.lb_ip.address
  port_range            = "443"
  target                = google_compute_target_https_proxy.main.id
  load_balancing_scheme = "EXTERNAL_MANAGED"
}

# HTTP → HTTPS redirect
resource "google_compute_url_map" "redirect" {
  name = "lb-prod-redirect"
  default_url_redirect {
    https_redirect = true
    strip_query    = false
  }
}

resource "google_compute_target_http_proxy" "redirect" {
  name    = "proxy-prod-http-redirect"
  url_map = google_compute_url_map.redirect.id
}

resource "google_compute_global_forwarding_rule" "http" {
  name                  = "fe-prod-http"
  ip_address            = google_compute_global_address.lb_ip.address
  port_range            = "80"
  target                = google_compute_target_http_proxy.redirect.id
  load_balancing_scheme = "EXTERNAL_MANAGED"
}

# Internal TCP LB
resource "google_compute_forwarding_rule" "internal_db" {
  name                  = "ilb-db-prod"
  region                = "asia-south1"
  load_balancing_scheme = "INTERNAL"
  backend_service       = google_compute_region_backend_service.db.id
  ports                 = ["5432"]
  network               = google_compute_network.prod.id
  subnetwork            = google_compute_subnetwork.private.id
}
```

---

## Part 6: Real-World Patterns

### Startup

```
Load balancer: 1 External Application LB (Global HTTP/S)

Setup:
├── 1 global static IP (Anycast)
├── Google-managed SSL cert (free!)
├── Backend bucket: GCS (static SPA)
├── Backend service: Cloud Run via serverless NEG (API)
├── URL map: /api/* → Cloud Run, /* → GCS
├── Cloud CDN: Enabled on GCS backend
└── HTTP → HTTPS redirect

No internal LB (small team).
Cost: ~$20/month (LB minimum) + usage
```

### Mid-Size

```
Load balancers:
├── 1 External Application LB (global)
└── 1 Internal Application LB (regional)

External LB:
├── Backend bucket: GCS (static assets, CDN ON)
├── Backend service: GKE NEG (web app)
├── Backend service: Cloud Run NEG (API)
├── Cloud Armor: Rate limiting + OWASP rules
├── Traffic splitting: 90/10 canary for API v2
└── Session affinity: Generated cookie for web app

Internal LB:
├── user-service → GKE NEG
├── order-service → GKE NEG
├── payment-service → GKE NEG
└── Health checks per service

Cost: ~$40-100/month
```

### Enterprise

```
Load balancers:
├── External Application LB (global) — public traffic
│   ├── Multi-region backends (Asia, US, Europe)
│   ├── Cloud CDN on static backends
│   ├── Cloud Armor (advanced WAF + bot management)
│   ├── Traffic splitting (canary per service)
│   ├── Request mirroring (shadow testing)
│   └── Fault injection (chaos engineering)
│
├── Internal Application LB (regional, per region)
│   ├── Microservice routing (20+ services)
│   ├── gRPC load balancing
│   ├── Header-based routing
│   └── Retry policies
│
├── Cross-region Internal Application LB
│   ├── Global internal services
│   └── Single internal IP, nearest region
│
├── Internal Passthrough Network LB
│   ├── Database load balancing
│   ├── Next-hop for firewall VMs
│   └── UDP services (DNS, syslog)
│
└── External Passthrough Network LB
    ├── Game servers (UDP)
    └── IoT ingestion (MQTT over TCP)

Monitoring:
├── Cloud Monitoring dashboards per LB
├── Request logging → BigQuery analytics
├── Health check logs for debugging
├── Alerts: Error rate, latency, unhealthy backends
└── Load balancer capacity planning

Cost: ~$100-500/month
```

---

## Quick Reference

| Feature | External App LB | Internal App LB | Passthrough Net LB |
|---------|----------------|-----------------|-------------------|
| Layer | 7 (HTTP/S) | 7 (HTTP/S) | 4 (TCP/UDP) |
| Scope | Global | Regional/Global | Regional |
| Anycast IP | Yes | No | No |
| URL routing | Yes | Yes | No |
| SSL termination | Yes | Yes | No (passthrough) |
| CDN | Yes | No | No |
| Cloud Armor | Yes | No | No |
| Traffic splitting | Yes | Yes | No |
| Source IP | Via header | Via header | Preserved |
| Serverless NEG | Yes | Yes | No |
| gRPC | Yes | Yes | No |
| Cost | ~$20/mo + usage | ~$20/mo + usage | ~$20/mo + usage |

---

## Console Walkthrough: Deleting a Load Balancer

```
┌─────────────────────────────────────────────────────────────────────┐
│     DELETING A LOAD BALANCER — ORDER MATTERS!                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚠️ GCP load balancers are built from multiple resources.            │
│    You MUST delete them in reverse order (top → bottom):           │
│                                                                       │
│    1. Forwarding Rule (frontend)                                    │
│    2. Target Proxy (HTTPS/HTTP proxy)                               │
│    3. SSL Certificate (if Google-managed)                           │
│    4. URL Map (routing rules)                                       │
│    5. Backend Service / Backend Bucket                              │
│    6. Health Check                                                   │
│    7. Static IP (if reserved)                                       │
│                                                                       │
│ Why this order?                                                      │
│ ├── Forwarding rule references target proxy                        │
│ ├── Target proxy references URL map + SSL cert                    │
│ ├── URL map references backend services/buckets                   │
│ ├── Backend service references health check                       │
│ └── Can't delete a resource still referenced by another!          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Method 1: Console (Simplified)

```
Console → Network services → Load balancing

┌─────────────────────────────────────────────────────────────────┐
│           DELETE VIA CONSOLE (EASY WAY)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ 1. Console → Network services → Load balancing                 │
│ 2. Select the checkbox next to your load balancer              │
│ 3. Click [Delete] at the top                                   │
│                                                                   │
│ ┌─────────────────────────────────────────────────────────┐    │
│ │ Delete load balancer: lb-prod                            │    │
│ │                                                          │    │
│ │ The following resources will be deleted:                 │    │
│ │ ☑ Forwarding rule: fe-prod-https                       │    │
│ │ ☑ Forwarding rule: fe-prod-http-redirect               │    │
│ │ ☑ Target HTTPS proxy: proxy-prod-https                 │    │
│ │ ☑ Target HTTP proxy: proxy-prod-http-redirect          │    │
│ │ ☑ URL map: lb-prod                                     │    │
│ │ ☑ URL map: lb-prod-redirect                            │    │
│ │                                                          │    │
│ │ Also delete associated resources:                       │    │
│ │ ☑ Backend services (bs-web-prod, bs-api-prod)          │    │
│ │ ☑ Backend buckets (bb-static-assets)                   │    │
│ │ ☑ Health checks (hc-web-http)                          │    │
│ │ ☐ SSL certificates (cert-techcorp) ← review first!    │    │
│ │                                                          │    │
│ │ ⚠️ Review each checkbox carefully!                      │    │
│ │    Backends may be shared across load balancers.        │    │
│ │                                                          │    │
│ │ [Delete]  [Cancel]                                      │    │
│ └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│ ⚡ The Console handles deletion order automatically!             │
│    But does NOT delete: static IPs, instance groups, NEGs,     │
│    Cloud Armor policies, or GCS buckets.                       │
│    Clean these up manually if no longer needed.                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Method 2: gcloud CLI (Step-by-Step)

```
# ──────────────────────────────────────────────
# Step 1: Delete forwarding rules (frontend)
# ──────────────────────────────────────────────
gcloud compute forwarding-rules delete fe-prod-https \
  --global --quiet

gcloud compute forwarding-rules delete fe-prod-http-redirect \
  --global --quiet

# ──────────────────────────────────────────────
# Step 2: Delete target proxies
# ──────────────────────────────────────────────
gcloud compute target-https-proxies delete proxy-prod-https \
  --global --quiet

gcloud compute target-http-proxies delete proxy-prod-http-redirect \
  --global --quiet

# ──────────────────────────────────────────────
# Step 3: Delete SSL certificates
# ──────────────────────────────────────────────
gcloud compute ssl-certificates delete cert-techcorp \
  --global --quiet

# ──────────────────────────────────────────────
# Step 4: Delete URL maps
# ──────────────────────────────────────────────
gcloud compute url-maps delete lb-prod \
  --global --quiet

gcloud compute url-maps delete lb-prod-redirect \
  --global --quiet

# ──────────────────────────────────────────────
# Step 5: Delete backend services and buckets
# ──────────────────────────────────────────────
gcloud compute backend-services delete bs-web-prod \
  --global --quiet

gcloud compute backend-services delete bs-api-prod \
  --global --quiet

gcloud compute backend-buckets delete bb-static-assets \
  --quiet

# ──────────────────────────────────────────────
# Step 6: Delete health checks
# ──────────────────────────────────────────────
gcloud compute health-checks delete hc-web-http \
  --global --quiet

# ──────────────────────────────────────────────
# Step 7: Release static IP (optional)
# ──────────────────────────────────────────────
gcloud compute addresses delete ip-prod-lb \
  --global --quiet

# ──────────────────────────────────────────────
# Step 8: Delete serverless NEGs (if any)
# ──────────────────────────────────────────────
gcloud compute network-endpoint-groups delete neg-api-prod \
  --region=asia-south1 --quiet
```

### Quick Delete Script

```bash
#!/bin/bash
# delete-lb.sh — Delete an entire External Application LB stack
# Usage: ./delete-lb.sh

set -e

LB_NAME="lb-prod"
REGION="asia-south1"

echo "=== Deleting Load Balancer: $LB_NAME ==="

# 1. Forwarding rules
echo "Deleting forwarding rules..."
gcloud compute forwarding-rules list --global \
  --format="value(name)" | while read FR; do
  gcloud compute forwarding-rules delete "$FR" --global --quiet
done

# 2. Target proxies
echo "Deleting target proxies..."
gcloud compute target-https-proxies list \
  --format="value(name)" | while read TP; do
  gcloud compute target-https-proxies delete "$TP" --global --quiet
done
gcloud compute target-http-proxies list \
  --format="value(name)" | while read TP; do
  gcloud compute target-http-proxies delete "$TP" --global --quiet
done

# 3. URL maps
echo "Deleting URL maps..."
gcloud compute url-maps delete "$LB_NAME" --global --quiet

# 4. Backend services
echo "Deleting backend services..."
gcloud compute backend-services list --global \
  --format="value(name)" | while read BS; do
  gcloud compute backend-services delete "$BS" --global --quiet
done

echo "=== Done ==="
```

### Common Errors When Deleting

```
┌─────────────────────────────────────────────────────────────────┐
│           COMMON DELETION ERRORS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Error: "The resource is still in use by another resource"       │
│ Fix:   Delete in the correct order (top → bottom)              │
│        Check: gcloud compute target-https-proxies describe     │
│        to see which URL map it references                      │
│                                                                   │
│ Error: "The backend service is still in use by URL map"        │
│ Fix:   Delete the URL map first, then the backend service      │
│                                                                   │
│ Error: "The health check is still in use"                       │
│ Fix:   Delete all backend services using this health check     │
│        first (health check may be shared across backends)      │
│                                                                   │
│ Error: "Cannot delete address — in use by forwarding rule"     │
│ Fix:   Delete the forwarding rule first, then the static IP    │
│                                                                   │
│ 💡 Tip: Use --quiet flag to skip confirmation prompts.          │
│    Use --format="value(name)" to get clean output for scripts. │
│                                                                   │
│ 💡 To find what references a resource:                           │
│    gcloud compute url-maps describe lb-prod --global           │
│    (Shows all backends referenced by the URL map)              │
│                                                                   │
│    gcloud compute target-https-proxies describe proxy-prod \   │
│      --global                                                   │
│    (Shows which URL map and SSL cert it references)            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover Compute Engine — GCP's virtual machine service.

→ Next: [Chapter 12: Compute Engine](12-compute-engine.md)

---

*Last Updated: May 2026*
