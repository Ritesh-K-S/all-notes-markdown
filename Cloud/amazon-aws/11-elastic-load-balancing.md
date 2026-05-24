# Chapter 11: Elastic Load Balancing (AWS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Load Balancing Fundamentals](#part-1-load-balancing-fundamentals)
- [Part 2: ELB Types Comparison](#part-2-elb-types-comparison)
- [Part 3: Application Load Balancer (ALB) Deep Dive](#part-3-application-load-balancer-alb-deep-dive)
- [Part 4: Network Load Balancer (NLB) Deep Dive](#part-4-network-load-balancer-nlb-deep-dive)
- [Part 5: Gateway Load Balancer (GLB)](#part-5-gateway-load-balancer-glb)
- [Part 6: Access Logs & Monitoring](#part-6-access-logs--monitoring)
- [Part 7: Terraform Example](#part-7-terraform-example)
- [Part 8: Real-World Patterns](#part-8-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Load Balancing? Why Do I Need It?

> **Real-World Analogy:** Think of a load balancer like a restaurant host — when customers (requests) arrive, the host directs each party to a different available table (server) so no single waiter gets overwhelmed. If a waiter calls in sick (server crashes), the host simply stops seating people at their table.

**Why does this matter?** Without a load balancer, your entire app runs on one server — if it crashes, everything is down. With a load balancer, traffic is spread across multiple servers and unhealthy ones are automatically removed. It's the foundation of high availability.

**Which load balancer should I use?**

| Use Case | Type |
|----------|------|
| Web apps, APIs (HTTP/HTTPS) | **ALB** (Application LB) |
| Gaming, IoT, extreme performance | **NLB** (Network LB) |
| Existing third-party appliances | **GWLB** (Gateway LB) |
| Legacy (avoid for new projects) | Classic LB |

Elastic Load Balancing (ELB) automatically distributes incoming traffic across multiple targets (EC2 instances, containers, IPs, Lambda functions). AWS offers four types of load balancers, each designed for different use cases.

```
What you'll learn:
├── Load Balancing Fundamentals
├── ELB Types
│   ├── Application Load Balancer (ALB) — Layer 7
│   ├── Network Load Balancer (NLB) — Layer 4
│   ├── Gateway Load Balancer (GLB) — Layer 3
│   └── Classic Load Balancer (CLB) — Legacy
├── Target Groups
│   ├── Instance, IP, Lambda, ALB targets
│   └── Health checks
├── ALB Deep Dive
│   ├── Listeners & rules (host/path routing)
│   ├── Fixed response, redirect, forward actions
│   ├── Sticky sessions
│   ├── Authentication (Cognito/OIDC)
│   ├── WebSocket & HTTP/2
│   └── WAF integration
├── NLB Deep Dive
│   ├── Static IP / Elastic IP
│   ├── TCP/UDP/TLS listeners
│   ├── Cross-zone load balancing
│   └── PrivateLink (NLB as VPC endpoint service)
├── GLB Deep Dive
├── Access Logs & Monitoring
└── Real-world patterns
```

---

## Part 1: Load Balancing Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│                  LOAD BALANCING BASICS                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Distribute traffic across multiple servers so no single      │
│       server gets overwhelmed.                                      │
│                                                                       │
│ Without LB:                                                          │
│ Users ──► Single server → overloaded → crash!                      │
│                                                                       │
│ With LB:                                                             │
│ Users ──► [Load Balancer] ──► Server 1                             │
│                            ──► Server 2                             │
│                            ──► Server 3                             │
│                                                                       │
│ Benefits:                                                            │
│ ├── High availability: One server dies → traffic goes to others   │
│ ├── Scalability: Add/remove servers based on demand               │
│ ├── Health checks: Automatically stop sending to unhealthy       │
│ ├── SSL termination: Handle SSL at LB, not every server          │
│ └── Single entry point: One DNS name for multiple servers         │
│                                                                       │
│ OSI Layer context:                                                   │
│ ├── Layer 4 (Transport): TCP/UDP — sees IP + port                │
│ │   Can route based on: source IP, port                           │
│ │   Cannot see: URL path, headers, cookies, body                 │
│ │                                                                    │
│ ├── Layer 7 (Application): HTTP/HTTPS — sees everything          │
│ │   Can route based on: URL path, host header, query strings,    │
│ │   HTTP headers, cookies, HTTP method                            │
│ │                                                                    │
│ └── Layer 3 (Network): IP packets — for inline appliances        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: ELB Types Comparison

```
┌─────────────────────────────────────────────────────────────────────┐
│                  ELB TYPE COMPARISON                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────┬──────────┬──────────┬──────────┬────────────┐ │
│ │ Feature          │ ALB      │ NLB      │ GLB      │ CLB(legacy)│ │
│ ├──────────────────┼──────────┼──────────┼──────────┼────────────┤ │
│ │ OSI Layer        │ 7        │ 4        │ 3        │ 4 & 7     │ │
│ │ Protocols        │ HTTP,    │ TCP, UDP,│ IP       │ HTTP, TCP  │ │
│ │                  │ HTTPS,   │ TLS,     │ (GENEVE) │            │ │
│ │                  │ gRPC,    │ TCP_UDP  │          │            │ │
│ │                  │ WebSocket│          │          │            │ │
│ │ Static IP        │ No ❌    │ Yes ✅   │ No       │ No         │ │
│ │ Elastic IP       │ No ❌    │ Yes ✅   │ No       │ No         │ │
│ │ Path routing     │ Yes ✅   │ No ❌    │ No       │ No         │ │
│ │ Host routing     │ Yes ✅   │ No ❌    │ No       │ No         │ │
│ │ SSL termination  │ Yes      │ Yes(TLS) │ No       │ Yes        │ │
│ │ WAF              │ Yes ✅   │ No ❌    │ No       │ No         │ │
│ │ Auth (Cognito)   │ Yes ✅   │ No ❌    │ No       │ No         │ │
│ │ Lambda target    │ Yes ✅   │ No ❌    │ No       │ No         │ │
│ │ Sticky sessions  │ Yes      │ Yes      │ Yes      │ Yes        │ │
│ │ Cross-zone       │ Yes(free)│ Yes(paid)│ No       │ Yes(free)  │ │
│ │ PrivateLink      │ No       │ Yes ✅   │ Yes      │ No         │ │
│ │ Preserve src IP  │ No(X-FF) │ Yes ✅   │ Yes      │ No         │ │
│ │ Performance      │ Good     │ Millions │ N/A      │ Good       │ │
│ │                  │          │ req/sec  │          │            │ │
│ │ Fixed response   │ Yes ✅   │ No ❌    │ No       │ No         │ │
│ │ Redirect         │ Yes ✅   │ No ❌    │ No       │ No         │ │
│ │ Cost (approx)    │ $22/mo + │ $22/mo + │ $22/mo + │ $22/mo +   │ │
│ │                  │ LCU      │ NLCU     │ GLCU     │ usage      │ │
│ └──────────────────┴──────────┴──────────┴──────────┴────────────┘ │
│                                                                       │
│ When to use which:                                                   │
│ ├── ALB: HTTP/HTTPS traffic (web apps, APIs, microservices)       │
│ ├── NLB: TCP/UDP traffic (gaming, IoT, extreme performance)       │
│ ├── GLB: Inline security appliances (firewalls, IDS/IPS)          │
│ └── CLB: Legacy only — migrate to ALB or NLB                     │
│                                                                       │
│ ⚡ 90% of the time, you want ALB.                                   │
│    Use NLB only if you need: static IP, extreme perf, or TCP/UDP. │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Application Load Balancer (ALB) Deep Dive

### Creating an ALB

```
Console → EC2 → Load Balancers → Create load balancer → Application Load Balancer

┌─────────────────────────────────────────────────────────────────┐
│           CREATE APPLICATION LOAD BALANCER                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Basic configuration ──                                       │
│ Name:              [alb-prod-web]                               │
│ Scheme:            ● Internet-facing  ○ Internal               │
│   Internet-facing: Has public IP, accessible from internet     │
│   Internal: Private IP only, for internal services             │
│                                                                   │
│ IP address type:   ● IPv4  ○ Dual-stack (IPv4+IPv6)           │
│                                                                   │
│ ── Network mapping ──                                           │
│ VPC:               [prod-vpc ▼]                                │
│ Mappings:          (select AZs + subnets)                      │
│   ☑ ap-south-1a   [pub-subnet-1a ▼]                          │
│   ☑ ap-south-1b   [pub-subnet-1b ▼]                          │
│   ☐ ap-south-1c                                                │
│                                                                   │
│   ⚠️ Internet-facing ALB MUST be in PUBLIC subnets!            │
│   ⚠️ Select at least 2 AZs for high availability!             │
│   ⚠️ Internal ALB goes in PRIVATE subnets.                     │
│                                                                   │
│ ── Security groups ──                                           │
│   [sg-alb-prod ▼]                                              │
│   Inbound: 80 (HTTP) from 0.0.0.0/0                           │
│            443 (HTTPS) from 0.0.0.0/0                          │
│   Outbound: All to target SG                                   │
│                                                                   │
│ ── Listeners and routing ──                                     │
│   Listener 1:                                                    │
│   Protocol: [HTTPS ▼]   Port: [443]                           │
│   Default action: Forward to → [tg-web-prod ▼]                │
│   Certificate: [*.techcorp.com (ACM) ▼]                       │
│   Security policy: [ELBSecurityPolicy-TLS13-1-2-2021-06 ▼]   │
│                                                                   │
│   Listener 2:                                                    │
│   Protocol: [HTTP ▼]    Port: [80]                             │
│   Default action: Redirect to → HTTPS://#{host}:443/#{path}   │
│   Status code: 301                                              │
│                                                                   │
│ [Create load balancer]                                           │
│                                                                   │
│ After creation:                                                  │
│ DNS: alb-prod-web-1234567890.ap-south-1.elb.amazonaws.com     │
│ ⚠️ ALB DNS name changes. Use Route 53 ALIAS record!            │
│ ⚠️ ALB has NO static IP. If you need static IP → use NLB.     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Target Groups

```
┌─────────────────────────────────────────────────────────────────────┐
│                  TARGET GROUPS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A group of targets (servers) that receive traffic.           │
│       Load balancer routes to target groups.                       │
│                                                                       │
│ Target types:                                                        │
│ ├── Instance: EC2 instances (by instance ID)                      │
│ ├── IP: Specific IP addresses (for containers, on-prem, etc.)    │
│ ├── Lambda: Lambda function (ALB only)                            │
│ └── ALB: Another ALB (NLB can target ALB!)                       │
│                                                                       │
│ Create target group:                                                 │
│ EC2 → Target Groups → Create target group                         │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ Target type:   ● Instances  ○ IP  ○ Lambda  ○ ALB        │   │
│ │ Name:          [tg-web-prod]                               │   │
│ │ Protocol:      [HTTP ▼]                                    │   │
│ │ Port:          [8080]  (port your app listens on)          │   │
│ │ IP type:       [IPv4 ▼]                                    │   │
│ │ VPC:           [prod-vpc ▼]                                │   │
│ │ Protocol version: [HTTP1 ▼] / HTTP2 / gRPC                │   │
│ │                                                             │   │
│ │ ── Health check settings ──                                │   │
│ │ Protocol:      [HTTP ▼]                                    │   │
│ │ Path:          [/health]                                    │   │
│ │                                                             │   │
│ │ Advanced:                                                   │   │
│ │ Port:           ● Traffic port  ○ Override: [8080]        │   │
│ │ Healthy threshold:   [3] (consecutive successes)          │   │
│ │ Unhealthy threshold: [3] (consecutive failures)           │   │
│ │ Timeout:        [5] seconds                                │   │
│ │ Interval:       [30] seconds                               │   │
│ │ Success codes:  [200]  (or 200-299)                       │   │
│ │                                                             │   │
│ │ ── Stickiness ──                                           │   │
│ │ Type: ○ Disabled                                          │   │
│ │       ● Load balancer generated cookie                    │   │
│ │       ○ Application-based cookie                          │   │
│ │ Duration: [3600] seconds (1 hour)                         │   │
│ │ Cookie name: AWSALB (auto) or custom                     │   │
│ │                                                             │   │
│ │ ── Deregistration delay ──                                 │   │
│ │ [300] seconds (5 min — time to finish in-flight requests)│   │
│ │ Reduce to [30] for faster deploys                         │   │
│ │                                                             │   │
│ │ ── Slow start duration ──                                  │   │
│ │ [0] seconds (0 = disabled)                                │   │
│ │ Set to [60] to gradually increase traffic to new targets  │   │
│ │                                                             │   │
│ │ ── Load balancing algorithm ──                             │   │
│ │ ● Round robin                                             │   │
│ │ ○ Least outstanding requests                              │   │
│ │ ○ Weighted random                                         │   │
│ │                                                             │   │
│ │ [Create target group]                                      │   │
│ │                                                             │   │
│ │ Then register targets:                                     │   │
│ │ ☑ i-abc123 (web-server-01)  Port: 8080                   │   │
│ │ ☑ i-def456 (web-server-02)  Port: 8080                   │   │
│ │ ☑ i-ghi789 (web-server-03)  Port: 8080                   │   │
│ │ [Include as pending below] → [Register pending targets]  │   │
│ │                                                             │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Health check flow:                                                   │
│ ALB → GET /health on each target every 30s                         │
│ ├── Returns 200 → healthy (after 3 consecutive successes)        │
│ ├── Returns 500 → unhealthy (after 3 consecutive failures)       │
│ ├── Timeout → unhealthy                                           │
│ └── Unhealthy targets removed from rotation                       │
│                                                                       │
│ ⚠️ Target SG must allow traffic FROM the ALB SG!                   │
│    Target SG inbound: Port 8080 from sg-alb-prod                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### ALB Listener Rules (Advanced Routing)

```
┌─────────────────────────────────────────────────────────────────────┐
│           ALB LISTENER RULES                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Rules are evaluated in order (priority). First match wins.         │
│ Last rule = default action (cannot be deleted).                    │
│                                                                       │
│ Rule conditions:                                                     │
│ ├── Host header: api.techcorp.com, admin.techcorp.com             │
│ ├── Path pattern: /api/*, /admin/*, /static/*                     │
│ ├── HTTP header: X-Custom-Header = specific-value                 │
│ ├── HTTP method: GET, POST, PUT, DELETE                            │
│ ├── Query string: ?version=2                                      │
│ └── Source IP: 10.0.0.0/8 (internal only)                         │
│                                                                       │
│ Rule actions:                                                        │
│ ├── Forward to: Target group                                      │
│ ├── Redirect to: URL (301/302)                                    │
│ ├── Fixed response: Return static HTML/JSON (no origin needed)   │
│ └── Authenticate: Cognito or OIDC before forwarding              │
│                                                                       │
│ Example listener rules (HTTPS:443):                                  │
│                                                                       │
│ ┌──────┬──────────────────────────┬──────────────────────────────┐ │
│ │ Pri  │ Condition                │ Action                       │ │
│ ├──────┼──────────────────────────┼──────────────────────────────┤ │
│ │ 1    │ Path: /health            │ Fixed response: 200 "OK"    │ │
│ │ 10   │ Host: api.techcorp.com   │ Forward → tg-api-prod       │ │
│ │      │ Path: /v2/*              │                              │ │
│ │ 20   │ Host: api.techcorp.com   │ Forward → tg-api-v1         │ │
│ │ 30   │ Host: admin.techcorp.com │ Authenticate (Cognito)      │ │
│ │      │                          │ → Forward → tg-admin         │ │
│ │ 40   │ Path: /api/*             │ Forward → tg-api-prod       │ │
│ │ 50   │ Path: /static/*          │ Forward → tg-static         │ │
│ │ 60   │ Host: old.techcorp.com   │ Redirect 301 → techcorp.com│ │
│ │ 99   │ Default                  │ Forward → tg-web-prod       │ │
│ └──────┴──────────────────────────┴──────────────────────────────┘ │
│                                                                       │
│ ⚡ ALB rules are incredibly powerful!                                │
│    You can do microservice routing, API versioning, A/B testing,   │
│    authentication, maintenance pages — all at the LB level.       │
│                                                                       │
│ Add rule:                                                            │
│ Load Balancer → Listeners → HTTPS:443 → Manage rules → Add rule  │
│                                                                       │
│ CLI:                                                                 │
│ aws elbv2 create-rule \                                              │
│   --listener-arn arn:aws:elasticloadbalancing:...:listener/... \   │
│   --priority 10 \                                                     │
│   --conditions '[{                                                   │
│     "Field": "host-header",                                         │
│     "Values": ["api.techcorp.com"]                                  │
│   },{                                                                │
│     "Field": "path-pattern",                                        │
│     "Values": ["/v2/*"]                                              │
│   }]' \                                                               │
│   --actions '[{                                                      │
│     "Type": "forward",                                               │
│     "TargetGroupArn": "arn:aws:elasticloadbalancing:...:tg-api"    │
│   }]'                                                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### ALB Authentication

```
┌─────────────────────────────────────────────────────────────────────┐
│           ALB AUTHENTICATION                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ALB can authenticate users BEFORE traffic reaches your app!        │
│                                                                       │
│ Supported:                                                           │
│ ├── Amazon Cognito User Pools                                      │
│ └── Any OIDC-compliant IdP (Google, Azure AD, Okta, Auth0)       │
│                                                                       │
│ Flow:                                                                │
│ User → ALB → "Not authenticated" → Redirect to login             │
│ User → Cognito/OIDC login → token → ALB validates → forward      │
│ ALB adds headers: x-amzn-oidc-data, x-amzn-oidc-identity         │
│                                                                       │
│ Rule action:                                                         │
│   Action 1: Authenticate (Cognito or OIDC)                        │
│   Action 2: Forward to target group                                │
│                                                                       │
│ Use case: Protect admin panel without adding auth code.            │
│ admin.techcorp.com → ALB authenticates via Cognito →              │
│ only authenticated users reach the admin app.                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### ALB Additional Features

```
┌─────────────────────────────────────────────────────────────────────┐
│           ALB FEATURES                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ WebSocket support:                                                   │
│ ├── Natively supported on ALB                                     │
│ ├── Long-lived connections (ws://, wss://)                        │
│ ├── Sticky sessions recommended for WebSocket                     │
│ └── No special configuration needed                                │
│                                                                       │
│ HTTP/2:                                                              │
│ ├── Supported on frontend (client ↔ ALB)                          │
│ ├── ALB ↔ targets: HTTP/1.1 or HTTP/2 (configure target group)  │
│ ├── Multiplexing, header compression, binary protocol             │
│ └── Enabled by default for HTTPS listeners                        │
│                                                                       │
│ gRPC:                                                                │
│ ├── Supported (target group protocol version = gRPC)              │
│ ├── Health checks use gRPC health check protocol                  │
│ ├── Content-based routing for gRPC services                       │
│ └── Requires HTTP/2 and HTTPS                                     │
│                                                                       │
│ X-Forwarded-For header:                                              │
│ ├── ALB adds client's real IP to X-Forwarded-For header          │
│ ├── Your app sees ALB's IP as source, not client's               │
│ ├── Read X-Forwarded-For to get the real client IP               │
│ └── X-Forwarded-Proto: HTTP or HTTPS (original protocol)        │
│                                                                       │
│ Cross-zone load balancing:                                           │
│ ├── Distributes traffic evenly across ALL targets in all AZs     │
│ ├── Enabled by default for ALB (free!)                            │
│ ├── Without: Each AZ gets equal share (unbalanced if uneven)     │
│ └── With: Each target gets equal share regardless of AZ          │
│                                                                       │
│ Desync mitigation mode:                                              │
│ ├── Protects against HTTP desync attacks                          │
│ ├── Modes: Defensive (default), Strictest, Monitor               │
│ └── Keep default unless you have specific needs                   │
│                                                                       │
│ Connection draining (deregistration delay):                          │
│ ├── When target is removed, wait for in-flight requests          │
│ ├── Default: 300 seconds (5 minutes)                              │
│ ├── Set to 30s for faster rolling deployments                     │
│ └── 0 = immediately stop (may drop requests!)                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Network Load Balancer (NLB) Deep Dive

```
┌─────────────────────────────────────────────────────────────────────┐
│           NETWORK LOAD BALANCER (NLB)                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Layer 4 load balancer. Ultra-high performance.                     │
│ Handles millions of requests per second with ultra-low latency.    │
│                                                                       │
│ Key features:                                                        │
│ ├── Static IP per AZ (auto-assigned or Elastic IP)                │
│ ├── Preserves source IP (no X-Forwarded-For needed)              │
│ ├── TCP, UDP, TLS protocols                                       │
│ ├── Millions of requests/sec, ~100µs latency                     │
│ ├── VPC Endpoint Service (PrivateLink provider)                  │
│ ├── Long-lived connections (TCP keep-alive)                       │
│ └── Can target ALB (NLB → ALB pattern!)                          │
│                                                                       │
│ When NLB over ALB:                                                   │
│ ├── Need static IP or Elastic IP (allowlisting by partners)      │
│ ├── Need extreme performance (gaming, financial, IoT)            │
│ ├── Non-HTTP protocols (TCP, UDP — databases, gaming, DNS)       │
│ ├── Need to expose as PrivateLink service                        │
│ ├── Need source IP preservation                                   │
│ └── Need TLS passthrough (not termination)                       │
│                                                                       │
│ NLB → ALB pattern:                                                   │
│ ├── Need static IP (NLB) + Layer 7 routing (ALB)                │
│ ├── NLB target group = ALB                                       │
│ ├── Client → NLB (static IP) → ALB (path/host routing) → targets│
│ └── Common for enterprises needing static IPs for firewall rules │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating an NLB

```
Console → EC2 → Load Balancers → Create → Network Load Balancer

┌─────────────────────────────────────────────────────────────────┐
│           CREATE NETWORK LOAD BALANCER                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:       [nlb-prod-tcp]                                      │
│ Scheme:     ● Internet-facing  ○ Internal                      │
│ IP type:    ● IPv4                                              │
│                                                                   │
│ Network mapping:                                                │
│ VPC: [prod-vpc ▼]                                              │
│ ☑ ap-south-1a  [pub-subnet-1a ▼]                              │
│   IPv4: ○ Assigned by AWS  ● Use Elastic IP: [eip-nlb-1a ▼] │
│ ☑ ap-south-1b  [pub-subnet-1b ▼]                              │
│   IPv4: ○ Assigned by AWS  ● Use Elastic IP: [eip-nlb-1b ▼] │
│                                                                   │
│ ⚡ Each AZ gets its own static IP!                               │
│    Partners can allowlist these specific IPs.                   │
│                                                                   │
│ Listeners:                                                       │
│   Protocol: [TCP ▼]   Port: [443]                              │
│   Forward to: [tg-tcp-prod ▼]                                  │
│                                                                   │
│   (Or TLS for SSL termination at NLB):                         │
│   Protocol: [TLS ▼]   Port: [443]                              │
│   Certificate: [*.techcorp.com (ACM) ▼]                       │
│   ALPN policy: [HTTP2Optional ▼]                               │
│   Forward to: [tg-tcp-prod ▼]                                  │
│                                                                   │
│ Cross-zone: ○ Disabled (default)  ● Enabled (recommended)     │
│   ⚠️ Cross-zone costs extra for NLB!                            │
│   $0.006 per GB of cross-AZ data.                              │
│   ALB cross-zone is free.                                      │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### NLB as PrivateLink Provider

```
┌─────────────────────────────────────────────────────────────────────┐
│           NLB + PRIVATELINK                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Expose your service to other AWS accounts privately          │
│       via VPC Endpoint Service.                                    │
│                                                                       │
│ Your VPC (Service Provider):                                        │
│ [NLB] → [Your App] → registered as VPC Endpoint Service           │
│                                                                       │
│ Customer's VPC (Service Consumer):                                   │
│ [VPC Endpoint (Interface)] → connects to your NLB privately       │
│                                                                       │
│ No VPC peering, no internet, no public IPs needed!                 │
│                                                                       │
│ Use case: SaaS provider offering private connectivity to customers.│
│                                                                       │
│ Setup:                                                               │
│ VPC → Endpoint services → Create                                   │
│   Load balancer: [nlb-prod-tcp ▼]                                  │
│   Require acceptance: ☑ (you approve each consumer)               │
│   Allowed principals: arn:aws:iam::CUSTOMER_ACCOUNT:root          │
│                                                                       │
│ Customer creates VPC Interface Endpoint → your service name        │
│ You approve → private connection established!                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Gateway Load Balancer (GLB)

```
┌─────────────────────────────────────────────────────────────────────┐
│           GATEWAY LOAD BALANCER (GLB)                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Layer 3 (IP level). For inline virtual network appliances.         │
│                                                                       │
│ Use case: Route ALL traffic through security appliances            │
│ ├── Firewalls (Palo Alto, Fortinet, Check Point)                  │
│ ├── Intrusion detection/prevention (IDS/IPS)                     │
│ ├── Deep packet inspection                                        │
│ └── Traffic analytics/mirroring                                   │
│                                                                       │
│ Flow:                                                                │
│ Internet → GLB Endpoint → GLB → [Firewall] → GLB → your app     │
│                                                                       │
│ Uses GENEVE protocol (port 6081) for encapsulation.                │
│ Traffic goes through appliance transparently — appliance sees     │
│ original packets, inspects/filters, returns to GLB.               │
│                                                                       │
│ ⚠️ Most companies don't use GLB directly.                           │
│    It's used by network security vendors.                          │
│    You'll encounter it if using third-party security appliances.  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Access Logs & Monitoring

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. CloudWatch Metrics (automatic):                                   │
│    ALB metrics:                                                      │
│    ├── RequestCount: Total requests                                │
│    ├── ActiveConnectionCount: Current connections                  │
│    ├── TargetResponseTime: Avg response time from targets         │
│    ├── HTTPCode_Target_2XX_Count: Successful responses            │
│    ├── HTTPCode_Target_5XX_Count: Server errors                   │
│    ├── HTTPCode_ELB_5XX_Count: LB errors (capacity, etc.)        │
│    ├── HealthyHostCount: Per target group                         │
│    ├── UnHealthyHostCount: Per target group                       │
│    └── RejectedConnectionCount: Exceeded limit                    │
│                                                                       │
│    Key alarms:                                                       │
│    ├── UnHealthyHostCount > 0: Alert immediately                  │
│    ├── HTTPCode_Target_5XX > threshold: Application errors        │
│    ├── TargetResponseTime > 5s: Performance issue                 │
│    └── HTTPCode_ELB_5XX > 0: LB itself has issues                │
│                                                                       │
│ 2. Access Logs (S3):                                                 │
│    ├── Detailed log of every request                               │
│    ├── Client IP, request URL, response code, timing              │
│    ├── Stored in S3 (gzip compressed)                              │
│    ├── Analyze with Athena                                         │
│    ├── Enable: LB → Attributes → Access logs → Enable            │
│    └── S3 bucket must allow ELB service principal                 │
│                                                                       │
│ 3. Connection Logs (ALB):                                            │
│    ├── TLS connection details                                      │
│    ├── Cipher suites, handshake times                              │
│    └── Useful for TLS debugging                                    │
│                                                                       │
│ 4. Request Tracing:                                                  │
│    ├── ALB adds X-Amzn-Trace-Id header                            │
│    ├── Trace requests across ALB → targets                        │
│    └── Integrates with AWS X-Ray                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform Example

```hcl
# ALB
resource "aws_lb" "prod" {
  name               = "alb-prod-web"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = [aws_subnet.pub_1a.id, aws_subnet.pub_1b.id]

  enable_deletion_protection = true

  access_logs {
    bucket  = aws_s3_bucket.lb_logs.id
    prefix  = "alb/prod"
    enabled = true
  }
}

# Target group (web)
resource "aws_lb_target_group" "web" {
  name                 = "tg-web-prod"
  port                 = 8080
  protocol             = "HTTP"
  vpc_id               = aws_vpc.prod.id
  target_type          = "instance"
  deregistration_delay = 30

  health_check {
    enabled             = true
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    healthy_threshold   = 3
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    matcher             = "200"
  }

  stickiness {
    type            = "lb_cookie"
    cookie_duration = 3600
    enabled         = false
  }
}

# Target group (API)
resource "aws_lb_target_group" "api" {
  name                 = "tg-api-prod"
  port                 = 3000
  protocol             = "HTTP"
  vpc_id               = aws_vpc.prod.id
  target_type          = "instance"
  deregistration_delay = 30

  health_check {
    path    = "/api/health"
    matcher = "200"
  }
}

# HTTPS listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.prod.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.wildcard.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# HTTP → HTTPS redirect
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.prod.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# Listener rule: API routing
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 10

  condition {
    host_header {
      values = ["api.techcorp.com"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}

# Listener rule: path-based routing
resource "aws_lb_listener_rule" "api_path" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 20

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}

# Listener rule: health check (fixed response)
resource "aws_lb_listener_rule" "health" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 1

  condition {
    path_pattern {
      values = ["/health"]
    }
  }

  action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "OK"
      status_code  = "200"
    }
  }
}

# Register targets
resource "aws_lb_target_group_attachment" "web_1" {
  target_group_arn = aws_lb_target_group.web.arn
  target_id        = aws_instance.web_1.id
  port             = 8080
}

# NLB (for static IP use case)
resource "aws_lb" "nlb" {
  name               = "nlb-prod-tcp"
  internal           = false
  load_balancer_type = "network"
  
  subnet_mapping {
    subnet_id     = aws_subnet.pub_1a.id
    allocation_id = aws_eip.nlb_1a.id
  }

  subnet_mapping {
    subnet_id     = aws_subnet.pub_1b.id
    allocation_id = aws_eip.nlb_1b.id
  }

  enable_cross_zone_load_balancing = true
}

# Route 53 ALIAS to ALB
resource "aws_route53_record" "app" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "app.techcorp.com"
  type    = "A"

  alias {
    name                   = aws_lb.prod.dns_name
    zone_id                = aws_lb.prod.zone_id
    evaluate_target_health = true
  }
}
```

---

## Part 8: Real-World Patterns

### Startup

```
Load balancer: 1 ALB (internet-facing)
Scheme: Internet-facing
Subnets: 2 public subnets (2 AZs)

Listeners:
├── HTTPS:443 → tg-web (port 8080)
└── HTTP:80 → redirect to HTTPS

Target group: 2-3 EC2 instances
Health check: /health (HTTP 200)
Deregistration delay: 30 seconds

DNS: api.techcorp.com → ALIAS → ALB
SSL: ACM wildcard cert

No NLB, no GLB.
Cost: ~$25/month + data processing
```

### Mid-Size

```
Load balancers:
├── ALB (internet-facing) — public traffic
└── ALB (internal) — service-to-service

Public ALB rules:
├── api.techcorp.com → tg-api
├── admin.techcorp.com → Cognito auth → tg-admin
├── /ws/* → tg-websocket (sticky sessions)
├── /static/* → tg-static
└── * → tg-web (SPA)

Internal ALB rules:
├── user-svc.internal → tg-user-service
├── order-svc.internal → tg-order-service
└── payment-svc.internal → tg-payment-service

Target groups: Mix of EC2 + ECS containers
Health checks: Per-service /health endpoints
Access logs: Enabled → S3 → Athena queries

Cost: ~$50-100/month (2 ALBs)
```

### Enterprise

```
Load balancers:
├── ALB (internet-facing) — public web/API
├── ALB (internal) — microservices (10+ target groups)
├── NLB (internet-facing) — TCP services + static IP
├── NLB (internal) — PrivateLink service provider
└── GLB — inline firewall appliances

Public ALB:
├── WAF attached (rate limiting, OWASP rules)
├── Host-based routing (20+ subdomains)
├── Path-based routing per microservice
├── Cognito auth on admin paths
├── gRPC support for ML services
└── Cross-zone enabled

NLB:
├── Elastic IPs per AZ (partner allowlisting)
├── TLS termination with ACM cert
├── VPC Endpoint Service for customers (SaaS)
└── NLB → ALB pattern for static IP + L7 routing

GLB:
├── Palo Alto VM-Series inline firewall
├── All ingress traffic inspected
└── Managed by security team

Monitoring:
├── CloudWatch dashboards per ALB/target group
├── Alarms: Unhealthy hosts, 5XX rate, response time
├── Access logs → Athena → weekly reports
├── X-Ray tracing across ALB → microservices
└── Connection logs for TLS audit

Cost: ~$200-800/month (multiple LBs)
```

---

## Quick Reference

| Feature | ALB | NLB |
|---------|-----|-----|
| Layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP) |
| Static IP | No | Yes (Elastic IP) |
| Path/host routing | Yes | No |
| SSL termination | Yes | Yes (TLS listener) |
| WAF | Yes | No |
| Authentication | Yes (Cognito/OIDC) | No |
| WebSocket | Yes | Yes (TCP) |
| gRPC | Yes | No |
| PrivateLink | No | Yes |
| Cross-zone cost | Free | $0.006/GB |
| Performance | Good | Millions req/sec |
| Source IP | X-Forwarded-For | Preserved |
| Lambda target | Yes | No |
| Fixed response | Yes | No |
| Cost | ~$22/mo + LCU | ~$22/mo + NLCU |

---

## What's Next?

In the next chapter, we'll cover Auto Scaling — automatically scaling EC2 instances based on demand.

→ Next: [Chapter 12: Auto Scaling](12-auto-scaling.md)

---

*Last Updated: May 2026*
