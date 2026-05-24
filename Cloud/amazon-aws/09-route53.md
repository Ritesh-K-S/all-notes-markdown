# Chapter 9: Route 53 - DNS Management (AWS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: DNS Fundamentals (Quick Refresher)](#part-1-dns-fundamentals-quick-refresher)
- [Part 2: Route 53 Components](#part-2-route-53-components)
- [Part 3: Routing Policies](#part-3-routing-policies)
- [Part 4: Health Checks](#part-4-health-checks)
- [Part 5: Domain Registration](#part-5-domain-registration)
- [Part 6: DNSSEC](#part-6-dnssec)

---

## Overview

### What is DNS? Why Do I Need Route 53?

> **Real-World Analogy:** Route 53 is the phone book of the internet. When you type "api.myapp.com" in a browser, Route 53 looks up the "phone number" (IP address) of your server and connects you. But unlike a paper phone book, Route 53 can also act as a smart switchboard — routing calls to the nearest office, switching to a backup when the main line is down, or splitting calls between two offices for load testing.

**Why does this matter?** DNS is the first thing that happens in every web request. If DNS is down, your entire application is unreachable — even if all servers are running perfectly. Route 53 has **100% SLA** (the only AWS service with this guarantee). Understanding DNS routing policies is also essential for multi-region deployments, disaster recovery, and zero-downtime deployments.

Amazon Route 53 is AWS's highly available, scalable DNS web service. It handles domain registration, DNS routing, and health checking. Named after TCP/UDP port 53 used by DNS.

```
What you'll learn:
├── DNS Fundamentals (quick refresher)
├── Route 53 Components
│   ├── Domain Registration
│   ├── Hosted Zones (public & private)
│   ├── Record Types (A, AAAA, CNAME, ALIAS, MX, TXT, etc.)
│   └── All fields explained
├── Routing Policies
│   ├── Simple
│   ├── Weighted
│   ├── Latency-based
│   ├── Failover
│   ├── Geolocation
│   ├── Geoproximity
│   └── Multi-value answer
├── Health Checks
├── Domain Registration
├── DNSSEC
├── Route 53 Resolver (Hybrid DNS)
└── Real-world patterns
```

---

## Part 1: DNS Fundamentals (Quick Refresher)

```
┌─────────────────────────────────────────────────────────────────────┐
│                  DNS BASICS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ DNS = Domain Name System (phone book of the internet)               │
│ Translates: www.example.com → 93.184.216.34                        │
│                                                                       │
│ Resolution flow:                                                     │
│                                                                       │
│ [Browser] → "api.techcorp.com"                                      │
│    │                                                                  │
│    ▼                                                                  │
│ [Local DNS Cache] → Found? Return IP                                │
│    │ Not found                                                       │
│    ▼                                                                  │
│ [ISP Resolver / Recursive Resolver]                                 │
│    │                                                                  │
│    ▼                                                                  │
│ [Root Name Server (.)]  → "Go ask .com TLD"                        │
│    │                                                                  │
│    ▼                                                                  │
│ [TLD Name Server (.com)] → "Go ask techcorp.com NS"               │
│    │                                                                  │
│    ▼                                                                  │
│ [Authoritative NS (Route 53)] → "93.184.216.34"                   │
│    │                                                                  │
│    ▼                                                                  │
│ [Browser] → connects to 93.184.216.34                               │
│                                                                       │
│ Key terms:                                                           │
│ ├── TTL (Time To Live): How long DNS response is cached            │
│ │   300 = 5 minutes, 86400 = 24 hours                              │
│ ├── NS (Name Server): Server that holds DNS records                │
│ ├── SOA (Start of Authority): Primary zone info                    │
│ ├── A Record: Domain → IPv4 address                                │
│ ├── AAAA Record: Domain → IPv6 address                             │
│ ├── CNAME: Domain → another domain (alias)                         │
│ ├── MX: Mail exchange servers                                      │
│ ├── TXT: Text records (SPF, DKIM, verification)                   │
│ ├── SRV: Service location                                          │
│ └── Authoritative: The final source of truth for a domain         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Route 53 Components

### Hosted Zones

```
┌─────────────────────────────────────────────────────────────────────┐
│                  HOSTED ZONES                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A container for DNS records for a specific domain.            │
│                                                                       │
│ Two types:                                                           │
│                                                                       │
│ 1. PUBLIC HOSTED ZONE                                                │
│    ├── Resolves queries from the internet                          │
│    ├── Anyone can query: dig api.techcorp.com                      │
│    ├── Used for: Websites, APIs, public services                   │
│    ├── Gets 4 name servers (NS records) globally distributed       │
│    └── Cost: $0.50/month per hosted zone                           │
│                                                                       │
│ 2. PRIVATE HOSTED ZONE                                               │
│    ├── Resolves queries ONLY within associated VPCs                │
│    ├── Not accessible from the internet                            │
│    ├── Used for: Internal services, databases, microservices       │
│    ├── Must associate with one or more VPCs                        │
│    ├── Can associate with VPCs in different accounts (via RAM)     │
│    ├── Example: db.internal.techcorp.com → 10.0.3.5               │
│    └── Cost: $0.50/month per hosted zone                           │
│                                                                       │
│ ⚡ You can have BOTH for the same domain!                            │
│    public: api.techcorp.com → ALB public IP (internet queries)     │
│    private: api.techcorp.com → ALB private IP (VPC queries)        │
│    This is called "split-horizon DNS"                               │
│                                                                       │
│ Pricing:                                                             │
│ ├── Hosted zone: $0.50/month (first 25 zones)                     │
│ ├── Queries: $0.40 per million (standard)                          │
│ ├── Queries: $0.60 per million (latency-based, geo)               │
│ ├── ALIAS queries to AWS resources: FREE!                          │
│ └── Health checks: $0.50-$2.00/month per check                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Hosted Zone

```
Console → Route 53 → Hosted zones → Create hosted zone

┌─────────────────────────────────────────────────────────────────┐
│           CREATE HOSTED ZONE                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Domain name:  [techcorp.com]                                    │
│ Description:  [Production DNS for techcorp.com]                │
│ Type:         ● Public hosted zone                              │
│               ○ Private hosted zone                             │
│                                                                   │
│ (If Private):                                                    │
│   VPCs to associate:                                            │
│   Region: [ap-south-1 ▼]  VPC: [prod-vpc ▼]                   │
│   [Add VPC]                                                      │
│                                                                   │
│ Tags:                                                            │
│   Environment: production                                        │
│                                                                   │
│ [Create hosted zone]                                             │
│                                                                   │
│ After creation (public zone):                                   │
│ You get 4 name servers:                                         │
│   ns-1234.awsdns-56.org                                         │
│   ns-789.awsdns-01.co.uk                                        │
│   ns-456.awsdns-78.com                                          │
│   ns-321.awsdns-90.net                                          │
│                                                                   │
│ ⚠️ Update your domain registrar's NS records to these!          │
│    Otherwise Route 53 won't serve your DNS.                     │
│    (If domain registered with Route 53, auto-done)              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Public zone
  aws route53 create-hosted-zone \
    --name techcorp.com \
    --caller-reference $(date +%s) \
    --hosted-zone-config Comment="Production DNS"

  # Private zone
  aws route53 create-hosted-zone \
    --name internal.techcorp.com \
    --caller-reference $(date +%s) \
    --vpc VPCRegion=ap-south-1,VPCId=vpc-prod \
    --hosted-zone-config Comment="Internal DNS",PrivateZone=true
```

### Record Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                  DNS RECORD TYPES                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ A RECORD (Address):                                                  │
│   Maps domain to IPv4 address                                       │
│   api.techcorp.com → 13.232.100.50                                  │
│                                                                       │
│ AAAA RECORD:                                                         │
│   Maps domain to IPv6 address                                       │
│   api.techcorp.com → 2600:1f18:abc:def::1                          │
│                                                                       │
│ CNAME RECORD (Canonical Name):                                       │
│   Maps domain to ANOTHER domain                                     │
│   blog.techcorp.com → techcorp.wordpress.com                        │
│   ⚠️ Cannot be used at zone apex! (techcorp.com ≠ CNAME)           │
│   ⚠️ Only for subdomains (www.techcorp.com = CNAME ✅)              │
│                                                                       │
│ ALIAS RECORD (AWS-specific!):                                        │
│   Maps domain to an AWS resource                                    │
│   techcorp.com → d12345.cloudfront.net                              │
│   ⚡ CAN be used at zone apex! (unlike CNAME)                       │
│   ⚡ FREE — no query charges for ALIAS to AWS resources             │
│   Supported targets:                                                │
│   ├── CloudFront distribution                                      │
│   ├── ELB (ALB, NLB, CLB)                                         │
│   ├── S3 website endpoint                                          │
│   ├── API Gateway                                                   │
│   ├── VPC Interface Endpoint                                        │
│   ├── Global Accelerator                                           │
│   ├── Another Route 53 record in same hosted zone                  │
│   └── Elastic Beanstalk environment                                │
│   ⚠️ Cannot ALIAS to an EC2 public IP or non-AWS resource!        │
│                                                                       │
│ MX RECORD (Mail Exchange):                                           │
│   Mail routing for the domain                                       │
│   techcorp.com MX 10 mail.techcorp.com                              │
│   techcorp.com MX 20 mail2.techcorp.com                             │
│   (Lower number = higher priority)                                  │
│                                                                       │
│ TXT RECORD:                                                          │
│   Text data — used for verification and email security              │
│   SPF: "v=spf1 include:_spf.google.com ~all"                       │
│   DKIM: "v=DKIM1; k=rsa; p=MIGfMA0..."                            │
│   Domain verify: "google-site-verification=xxxxx"                   │
│   ⚠️ Max 255 chars per string (split long values)                  │
│                                                                       │
│ NS RECORD (Name Server):                                             │
│   Which name servers are authoritative                              │
│   techcorp.com NS ns-1234.awsdns-56.org                            │
│   Auto-created when you create a hosted zone                       │
│                                                                       │
│ SOA RECORD:                                                          │
│   Zone metadata (serial, refresh, retry, expire, TTL)              │
│   Auto-created, rarely manually edited                             │
│                                                                       │
│ SRV RECORD:                                                          │
│   Service location                                                   │
│   _sip._tcp.techcorp.com SRV 10 60 5060 sip.techcorp.com          │
│                                                                       │
│ CAA RECORD (Certificate Authority Authorization):                    │
│   Which CAs can issue SSL certs for your domain                    │
│   techcorp.com CAA 0 issue "amazon.com"                            │
│   techcorp.com CAA 0 issue "letsencrypt.org"                       │
│                                                                       │
│ PTR RECORD (Reverse DNS):                                            │
│   IP → domain (reverse of A record)                                │
│   Used for email server verification                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### ALIAS vs CNAME

```
┌──────────────────────────┬──────────────────┬──────────────────────┐
│ Feature                  │ CNAME            │ ALIAS (AWS)          │
├──────────────────────────┼──────────────────┼──────────────────────┤
│ Zone apex (techcorp.com) │ ❌ Not allowed   │ ✅ Allowed           │
│ Target                   │ Any domain       │ AWS resources only   │
│ DNS query cost           │ Charged          │ FREE to AWS targets  │
│ Health check             │ Yes              │ Yes                  │
│ TTL                      │ You set it       │ Auto (from target)   │
│ How it works             │ Returns CNAME    │ Returns A/AAAA       │
│                          │ (extra lookup)   │ (direct IP response) │
│ Performance              │ Extra DNS hop    │ Direct resolution    │
└──────────────────────────┴──────────────────┴──────────────────────┘

⚡ ALIAS is almost always better for AWS resources!
   Use CNAME only for non-AWS targets (Shopify, Vercel, etc.)
```

### Creating Records

```
Console → Route 53 → Hosted zones → techcorp.com → Create record

┌─────────────────────────────────────────────────────────────────┐
│              CREATE RECORD                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Record name: [api].techcorp.com                                 │
│              (leave blank for zone apex: techcorp.com)          │
│                                                                   │
│ Record type: [A - Routes traffic to an IPv4 address ▼]         │
│                                                                   │
│ Alias:       ○ No  ● Yes                                       │
│                                                                   │
│ (If Alias = No):                                                │
│   Value: [13.232.100.50]                                        │
│          [13.232.100.51] (multiple IPs for round-robin)        │
│   TTL:   [300] seconds                                          │
│                                                                   │
│ (If Alias = Yes):                                               │
│   Route traffic to:                                              │
│   ├── [Alias to Application/Classic Load Balancer ▼]           │
│   ├── [Alias to CloudFront distribution ▼]                     │
│   ├── [Alias to S3 website endpoint ▼]                         │
│   ├── [Alias to API Gateway ▼]                                 │
│   └── [Alias to another record in this hosted zone ▼]         │
│                                                                   │
│   Region: [ap-south-1 ▼]                                       │
│   Resource: [prod-alb-12345.ap-south-1.elb.amazonaws.com ▼]   │
│                                                                   │
│ Routing policy: [Simple routing ▼]                              │
│   (see routing policies section below)                          │
│                                                                   │
│ [Create records]                                                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  # Simple A record
  aws route53 change-resource-record-sets \
    --hosted-zone-id Z1234567890 \
    --change-batch '{
      "Changes": [{
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.techcorp.com",
          "Type": "A",
          "TTL": 300,
          "ResourceRecords": [{"Value": "13.232.100.50"}]
        }
      }]
    }'

  # ALIAS to ALB
  aws route53 change-resource-record-sets \
    --hosted-zone-id Z1234567890 \
    --change-batch '{
      "Changes": [{
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.techcorp.com",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "ZXXXXXXXX",
            "DNSName": "prod-alb.ap-south-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }]
    }'
```

---

## Part 3: Routing Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│              ROUTE 53 ROUTING POLICIES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: How Route 53 responds to DNS queries.                         │
│       NOT load balancing — this is DNS-level routing.               │
│                                                                       │
│ Available policies:                                                  │
│ ├── 1. Simple                                                       │
│ ├── 2. Weighted                                                     │
│ ├── 3. Latency-based                                               │
│ ├── 4. Failover                                                     │
│ ├── 5. Geolocation                                                  │
│ ├── 6. Geoproximity                                                 │
│ └── 7. Multi-value answer                                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 1. Simple Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│ SIMPLE ROUTING                                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ One record, one or more values. DNS returns all values.             │
│ Client picks one randomly.                                          │
│                                                                       │
│ api.techcorp.com → 13.232.100.50                                    │
│                     13.232.100.51                                    │
│                     13.232.100.52                                    │
│                                                                       │
│ DNS response contains all 3 IPs, client picks one.                  │
│ No health checks possible (use multi-value for that).              │
│                                                                       │
│ Use case: Single resource, no special routing needed.              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. Weighted Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│ WEIGHTED ROUTING                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Route traffic based on weights (percentage-like).                   │
│ Great for: Blue/green deployments, canary releases.                 │
│                                                                       │
│ api.techcorp.com:                                                    │
│   Record 1: 13.232.100.50, Weight: 70  → 70% traffic              │
│   Record 2: 13.232.200.60, Weight: 20  → 20% traffic              │
│   Record 3: 13.232.300.70, Weight: 10  → 10% traffic              │
│                                                                       │
│ Weight 0 = no traffic (useful to temporarily disable)              │
│ All weights 0 = equal distribution                                  │
│                                                                       │
│ Canary deployment:                                                   │
│   Old version: weight 90 (90% of users)                             │
│   New version: weight 10 (10% of users)                             │
│   Gradually shift: 90/10 → 70/30 → 50/50 → 0/100                 │
│                                                                       │
│ Health checks: Can associate → if unhealthy, skip that record.     │
│                                                                       │
│ Create:                                                              │
│   Record name: api.techcorp.com                                     │
│   Routing: Weighted                                                 │
│   Record ID: "prod-v1" (unique identifier)                         │
│   Weight: 90                                                        │
│   Value: 13.232.100.50                                              │
│   Health check: hc-prod-v1                                          │
│                                                                       │
│   (Create another record, same name, different ID):                │
│   Record ID: "prod-v2"                                              │
│   Weight: 10                                                        │
│   Value: 13.232.200.60                                              │
│   Health check: hc-prod-v2                                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. Latency-Based Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│ LATENCY-BASED ROUTING                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Routes to the AWS region with lowest latency for the user.          │
│ AWS maintains a latency database between regions and user locations.│
│                                                                       │
│ api.techcorp.com:                                                    │
│   Record 1: ALB in ap-south-1 (India)                               │
│   Record 2: ALB in us-east-1 (US)                                   │
│   Record 3: ALB in eu-west-1 (Europe)                               │
│                                                                       │
│ User in Mumbai → ap-south-1 (lowest latency)                       │
│ User in New York → us-east-1 (lowest latency)                      │
│ User in London → eu-west-1 (lowest latency)                        │
│                                                                       │
│ Use case: Global applications, multi-region deployments.            │
│ Health checks: If primary region unhealthy, routes to next closest.│
│                                                                       │
│ Create:                                                              │
│   Record name: api.techcorp.com                                     │
│   Routing: Latency                                                  │
│   Region: ap-south-1                                                │
│   Value: prod-alb-india.ap-south-1.elb.amazonaws.com (ALIAS)      │
│   Health check: hc-india                                            │
│   Record ID: "india"                                                │
│                                                                       │
│   (Repeat for us-east-1, eu-west-1)                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 4. Failover Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│ FAILOVER ROUTING                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Active-passive failover. Primary serves traffic;                    │
│ if health check fails, routes to secondary.                        │
│                                                                       │
│ api.techcorp.com:                                                    │
│   Primary:   ALB in ap-south-1  ← Health check attached           │
│   Secondary: S3 static "sorry" page (or another region)            │
│                                                                       │
│ Normal:   Primary healthy → all traffic goes to Primary            │
│ Failure:  Primary unhealthy → all traffic goes to Secondary       │
│ Recovery: Primary healthy again → traffic returns to Primary      │
│                                                                       │
│ Use cases:                                                           │
│ ├── Active/standby DR                                              │
│ ├── Maintenance page during deployments                            │
│ └── "Sorry, we're down" static page as backup                     │
│                                                                       │
│ Create:                                                              │
│   Record 1: Failover type = Primary                                │
│     Value: ALB endpoint                                             │
│     Health check: REQUIRED                                          │
│                                                                       │
│   Record 2: Failover type = Secondary                              │
│     Value: S3 website endpoint                                     │
│     Health check: Optional                                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 5. Geolocation Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│ GEOLOCATION ROUTING                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Routes based on the user's geographic location (country/continent). │
│ Unlike latency-based (which uses network distance), this uses       │
│ actual geographic location.                                         │
│                                                                       │
│ api.techcorp.com:                                                    │
│   India → 13.232.100.50 (India backend)                             │
│   Europe → 52.29.100.50 (EU backend, GDPR-compliant)              │
│   USA → 54.85.100.50 (US backend)                                  │
│   Default → 13.232.100.50 (catch-all, REQUIRED!)                   │
│                                                                       │
│ Use cases:                                                           │
│ ├── Content localization (language, pricing)                       │
│ ├── Legal compliance (GDPR — EU users stay in EU)                 │
│ ├── Content restriction (geo-blocking)                             │
│ └── Different backend per country                                  │
│                                                                       │
│ ⚠️ MUST create a "Default" record for unmatched locations!         │
│    Otherwise, users from unmapped locations get NXDOMAIN.          │
│                                                                       │
│ Geolocation vs Latency:                                             │
│ ├── Geolocation: Based on country/continent (strict boundaries)   │
│ ├── Latency: Based on network performance (best speed)            │
│ └── Use Geo for compliance, Latency for performance               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 6. Geoproximity Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│ GEOPROXIMITY ROUTING                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Routes based on geographic distance + optional bias.                │
│ Bias: Expand or shrink the "catchment area" of a region.           │
│ Requires Route 53 Traffic Flow (visual editor).                    │
│                                                                       │
│ Without bias:                                                        │
│ Users routed to geographically nearest resource.                    │
│                                                                       │
│ With bias (+25 on ap-south-1):                                      │
│ ap-south-1's area expands — attracts users from farther away.      │
│ Users in Middle East might go to ap-south-1 instead of eu-west-1. │
│                                                                       │
│ Bias range: -99 to +99                                              │
│ +99: Maximum expansion (attracts most traffic)                      │
│ -99: Maximum shrinkage (repels traffic)                             │
│   0: No bias (pure geographic distance)                             │
│                                                                       │
│ Use case: Gradually shift traffic between regions.                  │
│           More granular than Geolocation.                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 7. Multi-Value Answer

```
┌─────────────────────────────────────────────────────────────────────┐
│ MULTI-VALUE ANSWER ROUTING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Like Simple routing but with health checks!                         │
│ Returns up to 8 healthy records.                                    │
│                                                                       │
│ api.techcorp.com:                                                    │
│   Record 1: 13.232.100.50 + Health check → healthy ✅               │
│   Record 2: 13.232.100.51 + Health check → unhealthy ❌             │
│   Record 3: 13.232.100.52 + Health check → healthy ✅               │
│                                                                       │
│ DNS response: Returns 13.232.100.50 and 13.232.100.52              │
│               (skips unhealthy .51)                                 │
│                                                                       │
│ ⚠️ NOT a replacement for ELB!                                       │
│    DNS caching means clients may use stale results.                │
│    Use for simple redundancy, not high-traffic load balancing.     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Routing Policy Comparison

```
┌────────────────────┬──────────────────────────────────────────────┐
│ Policy             │ When to Use                                  │
├────────────────────┼──────────────────────────────────────────────┤
│ Simple             │ Single resource, no routing logic            │
│ Weighted           │ Canary, blue/green, A/B testing              │
│ Latency            │ Multi-region, best performance               │
│ Failover           │ Active/passive DR                            │
│ Geolocation        │ Compliance, content localization             │
│ Geoproximity       │ Fine-grained geographic routing with bias    │
│ Multi-value        │ Simple health-checked round robin            │
└────────────────────┴──────────────────────────────────────────────┘
```

---

## Part 4: Health Checks

```
┌─────────────────────────────────────────────────────────────────────┐
│                  HEALTH CHECKS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Route 53 health checks monitor endpoints and determine        │
│       if they're healthy (affects routing decisions).               │
│                                                                       │
│ Three types:                                                         │
│ ├── 1. Endpoint health check (HTTP/HTTPS/TCP)                     │
│ │      Monitor a specific URL or IP                                │
│ │                                                                    │
│ ├── 2. Calculated health check                                     │
│ │      Combine results of other health checks                     │
│ │      "Healthy if 2 out of 3 child checks are healthy"           │
│ │                                                                    │
│ └── 3. CloudWatch alarm health check                               │
│        Monitor a CloudWatch alarm state                            │
│        Useful for: private resources (health checkers can't reach) │
│                                                                       │
│ How endpoint checks work:                                            │
│ ├── 15 global health checkers send requests                        │
│ ├── Interval: 30 seconds (standard) or 10 seconds (fast, $$$)    │
│ ├── Thresholds: 3 consecutive failures = unhealthy (configurable) │
│ ├── Checks: HTTP/HTTPS (can check response body, up to 5120B)    │
│ ├── TCP: Just connection check                                    │
│ └── HTTP: Can check specific path (/health) and status code       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Health Check

```
Console → Route 53 → Health checks → Create health check

┌─────────────────────────────────────────────────────────────────┐
│           CREATE HEALTH CHECK                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [hc-prod-api]                                             │
│                                                                   │
│ What to monitor:                                                │
│ ● Endpoint                                                      │
│ ○ Status of other health checks (calculated)                   │
│ ○ State of CloudWatch alarm                                    │
│                                                                   │
│ Endpoint Configuration:                                          │
│   Monitor endpoint:                                              │
│   ● IP address  ○ Domain name                                  │
│   IP address:    [13.232.100.50]                                │
│   Port:          [443]                                          │
│   Protocol:      [HTTPS ▼]                                     │
│   Path:          [/health]                                      │
│   Host name:     [api.techcorp.com]                            │
│                                                                   │
│ Advanced:                                                        │
│   Request interval:  ○ Standard (30 sec)  ● Fast (10 sec)     │
│   Failure threshold: [3] (unhealthy after 3 failures)          │
│   String matching:   ☑ Enable                                  │
│     Search string:   [OK]                                      │
│     (Body must contain "OK" to be considered healthy)           │
│   Latency graphs:    ☑ Enable                                  │
│                                                                   │
│   Health checker regions: ☑ Use recommended                    │
│   ○ Customize (select specific regions)                        │
│                                                                   │
│ Notification:                                                    │
│   Create alarm: ● Yes  ○ No                                   │
│   Send to: [devops@techcorp.com]                               │
│                                                                   │
│ [Create health check]                                            │
│                                                                   │
│ ⚠️ Health checkers come from AWS IPs!                            │
│    Your SG/firewall must allow traffic from Route 53 health     │
│    checker IP ranges. See: ip-ranges.amazonaws.com              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

CLI:
  aws route53 create-health-check \
    --caller-reference $(date +%s) \
    --health-check-config '{
      "IPAddress": "13.232.100.50",
      "Port": 443,
      "Type": "HTTPS",
      "ResourcePath": "/health",
      "FullyQualifiedDomainName": "api.techcorp.com",
      "RequestInterval": 10,
      "FailureThreshold": 3,
      "EnableSNI": true
    }'
```

---

## Part 5: Domain Registration

```
┌─────────────────────────────────────────────────────────────────────┐
│              DOMAIN REGISTRATION WITH ROUTE 53                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ You can register domains directly through Route 53.                 │
│                                                                       │
│ Console → Route 53 → Registered domains → Register domain          │
│   1. Search for domain availability                                 │
│   2. Select domain(s)                                               │
│   3. Fill in contact info (registrant, admin, tech)                │
│   4. Enable privacy protection (hides WHOIS info, free!)          │
│   5. Auto-renew: ● Enable                                          │
│   6. Review and purchase                                            │
│                                                                       │
│ What happens after registration:                                    │
│ ├── Route 53 automatically creates a public hosted zone            │
│ ├── NS records are auto-set to Route 53 name servers              │
│ ├── No need to manually configure NS at registrar!                │
│ └── Domain is managed entirely within AWS                          │
│                                                                       │
│ Pricing (examples):                                                  │
│ ├── .com: $13/year                                                 │
│ ├── .net: $11/year                                                 │
│ ├── .org: $12/year                                                 │
│ ├── .io: $39/year                                                  │
│ ├── .dev: $14/year                                                 │
│ └── .in: $10/year                                                  │
│                                                                       │
│ Domain transfer (from GoDaddy, Namecheap, etc.):                   │
│ ├── Unlock domain at current registrar                             │
│ ├── Get authorization/EPP code                                     │
│ ├── Route 53 → Transfer domain → Enter code                       │
│ ├── Confirm transfer via email                                     │
│ └── Takes 5-7 days                                                 │
│                                                                       │
│ ⚡ Registering domain with Route 53 is convenient but not required.│
│    You can use any registrar and just point NS to Route 53.        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: DNSSEC

```
┌─────────────────────────────────────────────────────────────────────┐
│                   DNSSEC                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: DNS Security Extensions — cryptographically signs DNS          │
│       records to prevent DNS spoofing/poisoning.                    │
│                                                                       │
│ Without DNSSEC: Attacker can forge DNS responses                    │
│ With DNSSEC: Responses are signed — forgery detected               │
│                                                                       │
│ Enable for a hosted zone:                                            │
│ Console → Route 53 → Hosted zone → DNSSEC signing → Enable        │
│   1. Create KSK (Key Signing Key) in KMS                          │
│   2. Route 53 creates ZSK (Zone Signing Key)                      │
│   3. Establish chain of trust with parent zone (.com)              │
│   4. Add DS record at registrar                                    │
│                                                                       │
│ ⚠️ Requires careful setup. Misconfigured DNSSEC can break DNS!    │
│ Enable only if your security policy requires it.                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Route 53 Resolver (Hybrid DNS)

```
┌─────────────────────────────────────────────────────────────────────┐
│           ROUTE 53 RESOLVER                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Enables DNS resolution between your VPC and on-premises       │
│       networks (hybrid DNS).                                        │
│                                                                       │
│ Problem: VPC can resolve public DNS and private hosted zones,       │
│ but on-prem DNS server can't resolve private hosted zones,         │
│ and VPC can't resolve on-prem DNS domains.                         │
│                                                                       │
│ Solution: Route 53 Resolver Endpoints                               │
│                                                                       │
│ Inbound Endpoint:                                                    │
│ ├── On-prem DNS → forward queries to Route 53 Resolver            │
│ ├── On-prem can now resolve: *.internal.techcorp.com              │
│ └── Creates ENIs in your VPC with IPs for forwarding              │
│                                                                       │
│ Outbound Endpoint:                                                   │
│ ├── Route 53 → forward queries to on-prem DNS                     │
│ ├── VPC can now resolve: *.corp.techcorp.local                    │
│ └── Forwarding rules define which domains go where                │
│                                                                       │
│ ┌──────────────────┐              ┌──────────────────┐            │
│ │ On-Premises       │              │ AWS VPC           │            │
│ │                    │              │                    │            │
│ │ corp.local DNS ◄──┤── Outbound ──│ Route 53          │            │
│ │                    │   Endpoint   │ Resolver          │            │
│ │ corp.local DNS ──►─┤── Inbound ──►│ Private Hosted   │            │
│ │                    │   Endpoint   │ Zones             │            │
│ │ (query .internal   │              │                    │            │
│ │  .techcorp.com)   │              │ (query .corp       │            │
│ │                    │              │  .techcorp.local)  │            │
│ └──────────────────┘              └──────────────────┘            │
│                                                                       │
│ Pricing:                                                             │
│ ├── Endpoint: $0.125/hr per IP (each endpoint needs 2+ IPs)      │
│ ├── Minimum per endpoint: ~$180/month                              │
│ └── Per query: $0.40 per million                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Terraform Example

```hcl
# Public hosted zone
resource "aws_route53_zone" "public" {
  name = "techcorp.com"
}

# Private hosted zone
resource "aws_route53_zone" "private" {
  name = "internal.techcorp.com"

  vpc {
    vpc_id = aws_vpc.prod.id
  }
}

# A record (ALIAS to ALB)
resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "api.techcorp.com"
  type    = "A"

  alias {
    name                   = aws_lb.prod.dns_name
    zone_id                = aws_lb.prod.zone_id
    evaluate_target_health = true
  }
}

# CNAME record
resource "aws_route53_record" "blog" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "blog.techcorp.com"
  type    = "CNAME"
  ttl     = 300
  records = ["techcorp.wordpress.com"]
}

# MX records
resource "aws_route53_record" "mail" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "techcorp.com"
  type    = "MX"
  ttl     = 3600
  records = [
    "10 mail.techcorp.com",
    "20 mail2.techcorp.com",
  ]
}

# Health check
resource "aws_route53_health_check" "api" {
  fqdn              = "api.techcorp.com"
  port               = 443
  type               = "HTTPS"
  resource_path      = "/health"
  failure_threshold  = 3
  request_interval   = 10

  tags = {
    Name = "hc-prod-api"
  }
}

# Weighted routing
resource "aws_route53_record" "api_v1" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "api.techcorp.com"
  type    = "A"

  weighted_routing_policy {
    weight = 90
  }

  set_identifier = "v1"
  ttl            = 60
  records        = ["13.232.100.50"]
  health_check_id = aws_route53_health_check.api.id
}

resource "aws_route53_record" "api_v2" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "api.techcorp.com"
  type    = "A"

  weighted_routing_policy {
    weight = 10
  }

  set_identifier = "v2"
  ttl            = 60
  records        = ["13.232.200.60"]
}

# Private DNS record
resource "aws_route53_record" "db_internal" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "db.internal.techcorp.com"
  type    = "A"
  ttl     = 60
  records = ["10.0.3.5"]
}
```

---

## Part 9: Real-World Patterns

### Startup

```
Hosted zones: 1 public (techcorp.com)
Records:
├── techcorp.com → ALIAS → CloudFront distribution
├── api.techcorp.com → ALIAS → ALB
├── www.techcorp.com → ALIAS → CloudFront
├── MX records for email (Google Workspace / Microsoft 365)
├── TXT: SPF, DKIM, domain verification
└── blog.techcorp.com → CNAME → blog platform

Routing: Simple (single region)
Health checks: 1-2 (API endpoint)
Cost: ~$2/month
```

### Mid-Size

```
Hosted zones:
├── techcorp.com (public)
├── internal.techcorp.com (private)
└── staging.techcorp.com (public, separate zone)

Records:
├── Public: api, www, admin, docs, status
├── Private: db.internal, cache.internal, queue.internal
├── Per-env: api.staging, api.dev
└── Health-checked weighted records for canary deployments

Routing: Weighted (canary), Failover (DR)
Health checks: 5-10 (critical endpoints)
Resolver: Not needed (no on-prem)
Cost: ~$10-20/month
```

### Enterprise

```
Hosted zones:
├── techcorp.com (public, production)
├── internal.techcorp.com (private, prod VPC)
├── staging.internal.techcorp.com (private, staging VPC)
├── dev.internal.techcorp.com (private, dev VPC)
└── Multiple zones for different business units

Records: 100+ across zones
Routing:
├── Latency-based for global users (US, EU, APAC)
├── Failover for DR (primary region → secondary)
├── Geolocation for GDPR compliance (EU users → EU region)
└── Weighted for canary deployments

Health checks: 20-50 (all critical services)
DNSSEC: Enabled on public zones
Resolver: Inbound + Outbound (hybrid with on-prem)
Route 53 Traffic Flow: Visual routing policies
Cost: ~$100-500/month
```

---

## Troubleshooting: Common DNS Issues

### "I changed the record but it still shows the old IP"

This is almost always **TTL caching**. When you set a record with TTL=3600 (1 hour), DNS resolvers worldwide cache that answer for 1 hour. Even after you change the record, users see the old value until the TTL expires.

```
Debugging:
1. Check the current DNS value:
   nslookup myapp.example.com
   dig myapp.example.com

2. Check the TTL:
   dig myapp.example.com | grep -i ttl

3. Force check directly from Route 53 (bypass cache):
   dig @ns-xxx.awsdns-xx.com myapp.example.com

4. Wait for TTL to expire, or:
   - Lower TTL to 60 seconds BEFORE making changes
   - Wait for old TTL to expire
   - Make the change
   - Raise TTL back after confirming
```

### "My domain isn't working at all"

```
1. ☐ Are the NS records at your registrar pointing to Route 53?
   dig NS example.com (should show ns-xxx.awsdns-xx.com)

2. ☐ Did you create the hosted zone in Route 53?
   Console → Route 53 → Hosted zones

3. ☐ Do the NS records at the registrar match the NS records in
   Route 53's hosted zone? (They must match exactly)
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using CNAME at zone apex | DNS spec violation, won't work | Use ALIAS record instead |
| High TTL before migration | Users cached old IP for hours | Lower TTL to 60s days before changes |
| Wrong NS records at registrar | Domain doesn't resolve at all | Copy NS from Route 53 hosted zone |
| Forgetting health checks | Failover routing doesn't work | Always attach health checks to failover records |

---

## Quick Reference

| Feature | Details |
|---------|---------|
| Hosted zone cost | $0.50/month per zone |
| Query cost | $0.40/million (standard) |
| ALIAS queries | FREE to AWS resources |
| Health check | $0.50-$2/month per check |
| ALIAS vs CNAME | ALIAS works at apex, FREE, faster |
| Routing policies | Simple, Weighted, Latency, Failover, Geo, Geoproximity, Multi-value |
| TTL | 60-86400 seconds (recommend 300 for most) |
| DNSSEC | Available, requires KMS key |
| Resolver | ~$180/month per endpoint |
| SLA | 100% availability SLA! |

---

## What's Next?

In the next chapter, we'll cover CloudFront — AWS's CDN for caching content at edge locations globally.

→ Next: [Chapter 10: CloudFront - CDN](10-cloudfront.md)

---

*Last Updated: May 2026*
