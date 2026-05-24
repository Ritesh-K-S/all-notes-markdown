# Chapter 10: CloudFront - Content Delivery Network (AWS)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: CDN Fundamentals](#part-1-cdn-fundamentals)
- [Part 2: CloudFront Distribution](#part-2-cloudfront-distribution)
- [Part 3: Cache Policies](#part-3-cache-policies)
- [Part 4: SSL/TLS & Custom Domains](#part-4-ssltls--custom-domains)
- [Part 5: Origin Access Control (OAC) for S3](#part-5-origin-access-control-oac-for-s3)
- [Part 6: Signed URLs & Signed Cookies](#part-6-signed-urls--signed-cookies)
- [Part 7: Lambda@Edge & CloudFront Functions](#part-7-lambdaedge--cloudfront-functions)
- [Part 8: Cache Invalidation](#part-8-cache-invalidation)
- [Part 9: Security Features](#part-9-security-features)
- [Part 10: Monitoring & Logs](#part-10-monitoring--logs)
- [Part 11: Terraform Example](#part-11-terraform-example)
- [Part 12: Real-World Patterns](#part-12-real-world-patterns)

---

## Overview

### What is a CDN? Why Do I Need CloudFront?

> **Real-World Analogy:** Imagine a pizza chain. Without CloudFront, every customer orders from one central kitchen (your origin server in us-east-1). A customer in Mumbai waits 200ms for each item. With CloudFront, you open local pickup counters (edge locations) in every major city. The first Mumbai customer's pizza is made at the central kitchen, but a copy is kept at the Mumbai counter. Every subsequent Mumbai customer gets it in 5ms.

**Why does this matter?** Page load time directly impacts business: Amazon found that every 100ms of latency costs 1% in sales. CloudFront can reduce your global page load time from 2 seconds to 200ms. It also reduces your AWS bill — data transfer through CloudFront is **cheaper** than direct transfer from S3/EC2, and it offloads ~90% of requests from your origin servers.

Amazon CloudFront is AWS's global Content Delivery Network (CDN) that caches and delivers content from edge locations worldwide. It reduces latency by serving content from the location closest to the user.

```
What you'll learn:
├── CDN Fundamentals
├── CloudFront Components
│   ├── Distributions (Web)
│   ├── Origins (S3, ALB, EC2, Custom)
│   ├── Behaviors (path-based routing + cache settings)
│   ├── Cache Policies & Origin Request Policies
│   └── All fields explained
├── SSL/TLS & Custom Domains
├── Origin Access Control (OAC) for S3
├── Signed URLs & Signed Cookies
├── Lambda@Edge & CloudFront Functions
├── Cache Invalidation
├── Security (WAF, Geo Restriction, Field-Level Encryption)
├── Real-Time Logs & Monitoring
└── Real-world patterns
```

---

## Part 1: CDN Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CDN BASICS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: A network of globally distributed servers that cache          │
│       content close to users for faster delivery.                   │
│                                                                       │
│ Without CDN:                                                         │
│ User (Mumbai) ──── internet ────► Origin (us-east-1)               │
│                    ~200ms latency                                    │
│                                                                       │
│ With CDN:                                                            │
│ User (Mumbai) ──► Edge (Mumbai) ──► Origin (us-east-1)             │
│                    ~5ms (cached!)   (only on cache miss)            │
│                                                                       │
│ What CloudFront caches:                                              │
│ ├── Static files: HTML, CSS, JS, images, fonts, videos            │
│ ├── API responses (when cacheable)                                  │
│ ├── Dynamic content (accelerated, not cached)                      │
│ └── Live & on-demand video streaming                               │
│                                                                       │
│ CloudFront network:                                                  │
│ ├── 450+ Points of Presence (PoPs) in 100+ cities                 │
│ ├── 13 Regional Edge Caches (larger, longer-lived cache)           │
│ ├── Uses AWS backbone network (not public internet)                │
│ └── Anycast routing to nearest edge                                │
│                                                                       │
│ Flow:                                                                │
│ User → Edge Location (PoP) → Regional Edge Cache → Origin         │
│         cache hit? return     cache hit? return    fetch & cache    │
│                                                                       │
│ ┌──────────┐   miss    ┌───────────────┐   miss   ┌──────────┐    │
│ │ Edge PoP │ ────────► │ Regional Edge │ ───────► │ Origin   │    │
│ │ (Mumbai) │           │ Cache (India) │          │ (S3/ALB) │    │
│ │          │ ◄──cache──│               │◄──cache──│          │    │
│ └──────────┘           └───────────────┘          └──────────┘    │
│       ▲                                                              │
│       │ cache hit (fast!)                                            │
│  [User Request]                                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: CloudFront Distribution

### Creating a Distribution

```
Console → CloudFront → Create distribution

┌─────────────────────────────────────────────────────────────────┐
│           CREATE DISTRIBUTION                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ┌─── ORIGIN SETTINGS ───────────────────────────────────────┐  │
│ │                                                             │  │
│ │ Origin domain: [tc-prod-assets.s3.amazonaws.com ▼]        │  │
│ │   Auto-populates from S3, ALB, EC2, MediaStore, etc.     │  │
│ │   Or type custom origin: api.techcorp.com                │  │
│ │                                                             │  │
│ │ Origin path: [/production]  (optional subfolder)          │  │
│ │   If set, CloudFront appends this to origin requests      │  │
│ │   Request for /logo.png → origin gets /production/logo.png│  │
│ │                                                             │  │
│ │ Name: [tc-prod-assets-origin]  (identifier)              │  │
│ │                                                             │  │
│ │ ── S3 Origin Only ──                                      │  │
│ │ Origin access:                                            │  │
│ │   ○ Public (S3 is publicly accessible)                   │  │
│ │   ● Origin access control settings (recommended) ← OAC  │  │
│ │   ○ Legacy access identity (OAI, deprecated)             │  │
│ │                                                             │  │
│ │ Origin access control:                                    │  │
│ │   [Create new OAC ▼]                                     │  │
│ │   Name: [oac-prod-assets]                                │  │
│ │   Origin type: [S3]                                      │  │
│ │   Signing behavior: [Sign requests (recommended)]        │  │
│ │                                                             │  │
│ │ ── Custom Origin Only ──                                  │  │
│ │ Protocol: ● HTTPS only  ○ HTTP only  ○ Match viewer     │  │
│ │ HTTP port: [80]    HTTPS port: [443]                     │  │
│ │ Minimum SSL protocol: [TLSv1.2 ▼]                       │  │
│ │                                                             │  │
│ │ ── All Origins ──                                         │  │
│ │ Enable Origin Shield: ○ Yes  ● No                        │  │
│ │   (Extra caching layer, reduces origin load)              │  │
│ │   Region: [ap-south-1 ▼] (closest to origin)            │  │
│ │                                                             │  │
│ │ Connection attempts: [3]  (1-3)                          │  │
│ │ Connection timeout: [10]  seconds (1-10)                 │  │
│ │ Response timeout: [30]    seconds (1-60)                 │  │
│ │ Keep-alive timeout: [5]   seconds (1-60)                 │  │
│ │                                                             │  │
│ │ Add custom header:                                        │  │
│ │   [X-Custom-Origin-Verify] : [secret-key-abc123]        │  │
│ │   (Origin can verify requests come from CloudFront)      │  │
│ │                                                             │  │
│ └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Origin Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                  ORIGIN TYPES                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. S3 BUCKET                                                         │
│    ├── Most common origin for static assets                        │
│    ├── Use OAC to keep bucket private (CloudFront-only access)    │
│    ├── S3 Transfer Acceleration NOT needed (CloudFront does it)   │
│    └── Can use S3 website endpoint (for redirects/index docs)     │
│                                                                       │
│ 2. ALB / ELB                                                         │
│    ├── For dynamic content (APIs, SSR apps)                        │
│    ├── ALB MUST be public-facing (CF connects via internet)       │
│    ├── Set custom header to verify requests from CF               │
│    ├── ALB SG: Allow CloudFront managed prefix list               │
│    └── Multiple behaviors: /api/* → ALB, /* → S3                  │
│                                                                       │
│ 3. EC2 INSTANCE                                                      │
│    ├── Must have public IP                                         │
│    ├── Less common (prefer ALB)                                    │
│    └── Instance SG: Allow CloudFront IPs                           │
│                                                                       │
│ 4. CUSTOM ORIGIN (any HTTP/S endpoint)                               │
│    ├── Your own server, another CDN, third-party API              │
│    ├── On-premises server via internet                             │
│    └── Any HTTP/HTTPS endpoint                                     │
│                                                                       │
│ 5. MEDIASTORE / MEDIAPACKAGE                                        │
│    ├── Video streaming origins                                     │
│    └── Live and on-demand video                                    │
│                                                                       │
│ 6. ORIGIN GROUP (failover)                                           │
│    ├── Primary origin + secondary origin                           │
│    ├── If primary returns 5xx or times out → try secondary        │
│    ├── Example: Primary S3 → Secondary S3 (different region)     │
│    └── Automatic failover, no health checks needed                │
│                                                                       │
│ Multiple origins per distribution:                                   │
│    /static/* → S3 bucket (images, CSS, JS)                         │
│    /api/*    → ALB (API server)                                     │
│    /media/*  → MediaStore (video)                                   │
│    /*        → S3 bucket (default: SPA index.html)                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Default Cache Behavior Settings

```
┌─────────────────────────────────────────────────────────────────┐
│     DEFAULT CACHE BEHAVIOR SETTINGS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Path pattern: Default (*)   (can't change for default)          │
│                                                                   │
│ Compress objects automatically: ● Yes  ○ No                    │
│   (Gzip/Brotli compression for text-based content)             │
│   Reduces file size by 60-80%!                                 │
│                                                                   │
│ Viewer protocol policy:                                          │
│   ○ HTTP and HTTPS                                              │
│   ○ Redirect HTTP to HTTPS  ← recommended                     │
│   ○ HTTPS only                                                  │
│                                                                   │
│ Allowed HTTP methods:                                            │
│   ● GET, HEAD                  (static content)                │
│   ○ GET, HEAD, OPTIONS         (CORS preflight)                │
│   ○ GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE  (API)        │
│                                                                   │
│ Restrict viewer access:  ○ No  ○ Yes                           │
│   (Yes = require signed URLs/cookies)                          │
│   Trusted key groups: [kg-prod ▼]                              │
│                                                                   │
│ ── Cache key and origin requests ──                             │
│                                                                   │
│ ● Cache policy and origin request policy (recommended)         │
│ ○ Legacy cache settings                                        │
│                                                                   │
│ Cache policy: [CachingOptimized ▼]                             │
│   AWS managed policies:                                         │
│   ├── CachingOptimized: Default TTL 24h, honors Cache-Control │
│   ├── CachingOptimizedForUncompressedObjects: No gzip          │
│   ├── CachingDisabled: No caching (pass-through)               │
│   ├── Elemental-MediaPackage: For video streaming              │
│   └── Custom: You define TTLs, query strings, headers, cookies│
│                                                                   │
│ Origin request policy: [AllViewerExceptHostHeader ▼]           │
│   What to send to origin:                                      │
│   ├── None: Minimal request                                    │
│   ├── AllViewer: Forward all viewer headers                    │
│   ├── AllViewerExceptHostHeader: All except Host               │
│   ├── CORS-S3Origin: CORS headers for S3                      │
│   └── Custom: You choose headers, query strings, cookies      │
│                                                                   │
│ Response headers policy: [SecurityHeadersPolicy ▼]             │
│   Add security headers to responses:                           │
│   ├── X-Content-Type-Options: nosniff                          │
│   ├── X-Frame-Options: SAMEORIGIN                              │
│   ├── X-XSS-Protection: 1; mode=block                         │
│   ├── Strict-Transport-Security: max-age=31536000             │
│   ├── Content-Security-Policy                                  │
│   └── CORS headers                                              │
│                                                                   │
│ ── Function associations ──                                     │
│ Viewer request:  [none ▼] (CloudFront Function or Lambda@Edge)│
│ Viewer response: [none ▼]                                      │
│ Origin request:  [none ▼] (Lambda@Edge only)                  │
│ Origin response: [none ▼] (Lambda@Edge only)                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Additional Behaviors (Path-Based Routing)

```
┌─────────────────────────────────────────────────────────────────────┐
│           CACHE BEHAVIORS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Each behavior = path pattern + origin + cache settings.             │
│ CloudFront matches behaviors top to bottom (first match wins).     │
│                                                                       │
│ Example SPA + API setup:                                             │
│                                                                       │
│ ┌───────────────┬────────────┬──────────────────────────────────┐  │
│ │ Path Pattern  │ Origin     │ Cache Settings                   │  │
│ ├───────────────┼────────────┼──────────────────────────────────┤  │
│ │ /api/*        │ ALB        │ CachingDisabled, all methods    │  │
│ │ /static/*     │ S3 assets  │ CachingOptimized, TTL 365 days │  │
│ │ /media/*      │ S3 media   │ CachingOptimized, TTL 30 days  │  │
│ │ Default (*)   │ S3 SPA     │ CachingOptimized, TTL 24h      │  │
│ └───────────────┴────────────┴──────────────────────────────────┘  │
│                                                                       │
│ Request: /api/users → matches /api/* → goes to ALB (no cache)     │
│ Request: /static/logo.png → matches /static/* → S3 (cached 1yr)  │
│ Request: /dashboard → matches * → S3 (serves index.html for SPA) │
│                                                                       │
│ Add behavior:                                                        │
│ CloudFront → Distribution → Behaviors → Create behavior           │
│   Path pattern: [/api/*]                                            │
│   Origin: [prod-alb ▼]                                              │
│   Cache policy: [CachingDisabled ▼]                                 │
│   Allowed methods: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE    │
│   Viewer protocol: Redirect HTTP to HTTPS                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Distribution Settings (continued)

```
┌─────────────────────────────────────────────────────────────────┐
│     DISTRIBUTION SETTINGS                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Price class:                                                     │
│   ● Use all edge locations (best performance)                  │
│   ○ Use only North America and Europe                          │
│   ○ Use North America, Europe, Asia, Middle East, Africa       │
│   (Lower price class = fewer edges = lower cost)               │
│                                                                   │
│ AWS WAF web ACL: [none ▼] or [waf-prod ▼]                     │
│   Attach WAF for protection against attacks.                   │
│   See WAF chapter for details.                                 │
│                                                                   │
│ Alternate domain names (CNAMEs):                                │
│   [cdn.techcorp.com]                                            │
│   [www.techcorp.com]                                            │
│   [techcorp.com]                                                │
│   (Custom domains instead of d12345.cloudfront.net)            │
│                                                                   │
│ Custom SSL certificate:                                          │
│   [*.techcorp.com (ACM) ▼]                                     │
│   ⚠️ MUST be in us-east-1! CloudFront only uses us-east-1 certs│
│   Use ACM to provision a free certificate.                     │
│                                                                   │
│ Security policy:                                                 │
│   [TLSv1.2_2021 ▼] ← recommended                              │
│   Options: TLSv1, TLSv1.1, TLSv1.2_2019, TLSv1.2_2021       │
│                                                                   │
│ Supported HTTP versions:                                         │
│   ☑ HTTP/2  ☑ HTTP/3  ← both recommended                     │
│                                                                   │
│ Default root object: [index.html]                               │
│   When someone visits https://techcorp.com/ → serves index.html│
│                                                                   │
│ Standard logging: ○ On  ● Off                                  │
│   S3 bucket: [tc-prod-cf-logs ▼]                               │
│   Log prefix: [cloudfront/prod/]                               │
│   Cookie logging: ☐                                             │
│                                                                   │
│ IPv6: ● Enabled                                                 │
│                                                                   │
│ Description: [Production CDN for techcorp.com]                 │
│                                                                   │
│ [Create distribution]                                            │
│                                                                   │
│ After creation:                                                  │
│ Distribution domain: d12345abcdef.cloudfront.net               │
│ Status: Deploying → Deployed (takes 5-15 minutes)             │
│                                                                   │
│ Then in Route 53:                                                │
│ techcorp.com → ALIAS → d12345abcdef.cloudfront.net             │
│ www.techcorp.com → ALIAS → d12345abcdef.cloudfront.net         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Cache Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CACHE POLICIES                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Define what's included in the cache key and TTL settings.    │
│                                                                       │
│ Cache key = what makes a cached object unique.                      │
│ Same cache key → same cached response.                              │
│                                                                       │
│ Cache key components (you choose):                                   │
│ ├── URL path (always included)                                     │
│ ├── Query strings (none, specific, all)                            │
│ ├── Headers (none, specific — NOT all!)                            │
│ ├── Cookies (none, specific, all)                                  │
│ └── Compression (gzip, brotli)                                     │
│                                                                       │
│ ⚠️ More things in cache key = more cache variations = lower hit %  │
│    Include ONLY what actually changes the response!                │
│                                                                       │
│ TTL settings:                                                        │
│ ├── Minimum TTL: 0 seconds (can't cache less than this)           │
│ ├── Maximum TTL: 31536000 (365 days, can't cache more)            │
│ ├── Default TTL: 86400 (24 hours, used if origin sends no TTL)   │
│ └── Origin Cache-Control/Expires headers respected within bounds  │
│                                                                       │
│ TTL priority:                                                        │
│ 1. Origin sends Cache-Control: max-age=3600                       │
│    → CloudFront uses 3600s (within min/max bounds)                │
│ 2. Origin sends no cache header                                    │
│    → CloudFront uses Default TTL                                  │
│ 3. Origin sends Cache-Control: no-cache or no-store               │
│    → CloudFront still caches for Minimum TTL!                     │
│    (Set Minimum TTL = 0 to honor no-cache)                        │
│                                                                       │
│ Create custom cache policy:                                          │
│ CloudFront → Policies → Cache → Create cache policy                │
│                                                                       │
│   Name: [cp-api-responses]                                          │
│   Min TTL: [0]  Default TTL: [60]  Max TTL: [3600]                │
│   Headers: [Authorization] (different response per user)           │
│   Query strings: [All]                                              │
│   Cookies: [session_id]                                             │
│   Compression: ☑ Gzip ☑ Brotli                                    │
│                                                                       │
│ Common patterns:                                                     │
│ ┌─────────────────┬───────────────────────────────────────────┐    │
│ │ Content Type    │ Cache Policy                               │    │
│ ├─────────────────┼───────────────────────────────────────────┤    │
│ │ Static assets   │ Max TTL, no QS/headers/cookies            │    │
│ │ (CSS, JS, imgs) │ Cache for 365 days, bust with filename   │    │
│ │ HTML pages      │ Short TTL (5 min), no QS                  │    │
│ │ API responses   │ CachingDisabled or short TTL + auth header│    │
│ │ Personalized    │ CachingDisabled (pass through)            │    │
│ └─────────────────┴───────────────────────────────────────────┘    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Origin Request Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│           ORIGIN REQUEST POLICIES                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Control what CloudFront sends to the origin                   │
│       (separate from cache key!).                                   │
│                                                                       │
│ You might NOT include a header in the cache key                     │
│ but still need to FORWARD it to the origin.                        │
│                                                                       │
│ Example:                                                             │
│ ├── Cache key: URL only (same cache for all users)                │
│ ├── Origin request: Forward Accept-Language header                 │
│ │   (Origin needs it but it shouldn't vary the cache)             │
│ └── Result: Cached, but origin gets the header on miss            │
│                                                                       │
│ Common origin request policies:                                      │
│ ├── AllViewer: Forward everything viewer sent                      │
│ ├── AllViewerExceptHostHeader: Forward all except Host             │
│ ├── CORS-S3Origin: Forward Origin, Access-Control-* headers       │
│ ├── CORS-CustomOrigin: CORS headers for non-S3                    │
│ └── UserAgentRefererHeaders: Forward User-Agent and Referer       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: SSL/TLS & Custom Domains

```
┌─────────────────────────────────────────────────────────────────────┐
│           SSL/TLS SETUP                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Get SSL certificate from ACM                                │
│ ⚠️ MUST be in us-east-1 region!                                     │
│                                                                       │
│ Console (switch to us-east-1) → ACM → Request certificate         │
│   Domain names: techcorp.com, *.techcorp.com                       │
│   Validation: DNS (add CNAME to Route 53, auto-validates)         │
│   → Certificate ARN: arn:aws:acm:us-east-1:123:cert/abc           │
│                                                                       │
│ Step 2: Add alternate domain names to distribution                  │
│   CNAMEs: techcorp.com, www.techcorp.com, cdn.techcorp.com        │
│   SSL certificate: Select the ACM cert from us-east-1             │
│                                                                       │
│ Step 3: Create DNS records                                          │
│   techcorp.com     → ALIAS → d12345.cloudfront.net                │
│   www.techcorp.com → ALIAS → d12345.cloudfront.net                │
│   cdn.techcorp.com → CNAME → d12345.cloudfront.net                │
│                                                                       │
│ Viewer ↔ CloudFront: Uses your custom SSL cert                     │
│ CloudFront ↔ Origin: Uses origin's cert (must be valid!)          │
│                                                                       │
│ Two SSL modes:                                                       │
│ ├── SNI (Server Name Indication): FREE, modern browsers only      │
│ │   ⚡ Use this (99.9% of browsers support SNI)                   │
│ └── Dedicated IP: $600/month, for legacy browser support          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Origin Access Control (OAC) for S3

```
┌─────────────────────────────────────────────────────────────────────┐
│           ORIGIN ACCESS CONTROL (OAC)                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: You want S3 content served ONLY through CloudFront,       │
│          not directly via S3 URL.                                   │
│                                                                       │
│ Without OAC:                                                         │
│ ├── https://d12345.cloudfront.net/logo.png ✅ (through CF)        │
│ └── https://bucket.s3.amazonaws.com/logo.png ✅ (direct = bad!)   │
│                                                                       │
│ With OAC:                                                            │
│ ├── https://d12345.cloudfront.net/logo.png ✅ (through CF)        │
│ └── https://bucket.s3.amazonaws.com/logo.png ❌ (403 Forbidden)   │
│                                                                       │
│ Setup:                                                               │
│ 1. Create OAC in CloudFront                                        │
│ 2. Attach OAC to S3 origin in distribution                        │
│ 3. Update S3 bucket policy to allow CloudFront                     │
│                                                                       │
│ S3 bucket policy (CloudFront provides this):                        │
│ {                                                                    │
│   "Statement": [{                                                   │
│     "Sid": "AllowCloudFrontServicePrincipal",                      │
│     "Effect": "Allow",                                              │
│     "Principal": {                                                  │
│       "Service": "cloudfront.amazonaws.com"                        │
│     },                                                               │
│     "Action": "s3:GetObject",                                      │
│     "Resource": "arn:aws:s3:::tc-prod-assets/*",                   │
│     "Condition": {                                                  │
│       "StringEquals": {                                              │
│         "AWS:SourceArn":                                            │
│           "arn:aws:cloudfront::123456:distribution/E12345"         │
│       }                                                              │
│     }                                                                │
│   }]                                                                 │
│ }                                                                    │
│                                                                       │
│ OAC vs OAI (legacy):                                                 │
│ ├── OAC: Newer, supports SSE-KMS, all S3 features, recommended   │
│ └── OAI: Legacy, limited, being deprecated                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Signed URLs & Signed Cookies

```
┌─────────────────────────────────────────────────────────────────────┐
│           SIGNED URLs & SIGNED COOKIES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Restrict content access to authorized users with             │
│       time-limited, signed requests.                                │
│                                                                       │
│ Signed URL:                                                          │
│ ├── One URL per resource                                           │
│ ├── Includes expiration, IP restriction, signature                │
│ ├── Use for: Individual file downloads, premium content           │
│ └── Example: https://d12345.cloudfront.net/video.mp4              │
│     ?Expires=1716000000&Signature=abc123&Key-Pair-Id=APKA...     │
│                                                                       │
│ Signed Cookie:                                                       │
│ ├── One cookie covers many resources                               │
│ ├── Set 3 cookies: CloudFront-Policy, CloudFront-Signature,      │
│ │   CloudFront-Key-Pair-Id                                        │
│ ├── Use for: Multiple files (streaming video segments),           │
│ │   entire website areas                                          │
│ └── No URL modification needed                                    │
│                                                                       │
│ When to use which:                                                   │
│ ├── Signed URL: Single file, direct link sharing                  │
│ ├── Signed Cookie: Multiple files, website sections, streaming    │
│ └── Signed URL overrides Signed Cookie if both present            │
│                                                                       │
│ Setup:                                                               │
│ 1. Create key pair (RSA 2048-bit)                                  │
│ 2. Upload public key to CloudFront → Key groups                   │
│ 3. Attach key group to behavior (Restrict viewer access: Yes)     │
│ 4. Application signs URLs/cookies with private key                │
│                                                                       │
│ Generate signed URL (Node.js example):                              │
│ const { getSignedUrl } = require('@aws-sdk/cloudfront-signer');   │
│ const signedUrl = getSignedUrl({                                    │
│   url: 'https://cdn.techcorp.com/premium/video.mp4',              │
│   keyPairId: 'APKAXXXXXXXXX',                                     │
│   dateLessThan: new Date(Date.now() + 3600 * 1000),  // 1 hour  │
│   privateKey: fs.readFileSync('private_key.pem'),                  │
│ });                                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Lambda@Edge & CloudFront Functions

```
┌─────────────────────────────────────────────────────────────────────┐
│       EDGE COMPUTE                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Run code at CloudFront edge locations!                              │
│ Two options: CloudFront Functions and Lambda@Edge.                  │
│                                                                       │
│ ┌──────────────────────┬────────────────────┬───────────────────┐  │
│ │ Feature              │ CF Functions       │ Lambda@Edge       │  │
│ ├──────────────────────┼────────────────────┼───────────────────┤  │
│ │ Runtime              │ JavaScript only    │ Node.js, Python   │  │
│ │ Runs at              │ Edge locations     │ Regional edge     │  │
│ │ Execution time       │ < 1 ms             │ 5-30 seconds      │  │
│ │ Memory               │ 2 MB               │ 128-10,240 MB     │  │
│ │ Network access       │ No                 │ Yes               │  │
│ │ Request body access  │ No                 │ Yes               │  │
│ │ Triggers             │ Viewer req/res     │ All 4 events      │  │
│ │ Cost                 │ $0.10/million      │ $0.60/million     │  │
│ │ Scale                │ 10M+ req/sec       │ Thousands/sec     │  │
│ └──────────────────────┴────────────────────┴───────────────────┘  │
│                                                                       │
│ 4 trigger points:                                                    │
│                                                                       │
│ [Viewer] ─Viewer Request─► [CloudFront] ─Origin Request─► [Origin]│
│ [Viewer] ◄Viewer Response─ [CloudFront] ◄Origin Response─ [Origin]│
│                                                                       │
│ CF Functions:  Viewer Request, Viewer Response only                 │
│ Lambda@Edge:   All 4 events                                         │
│                                                                       │
│ Common use cases:                                                    │
│ ├── URL rewrite: /products/123 → /index.html (SPA routing)       │
│ ├── Header manipulation: Add security headers                      │
│ ├── A/B testing: Route 10% users to new version                   │
│ ├── Auth: Validate JWT at edge (viewer request)                   │
│ ├── Bot detection: Block scrapers at edge                          │
│ ├── Image optimization: Resize/format at edge (origin response)   │
│ └── Redirect: www → non-www, HTTP → HTTPS                        │
│                                                                       │
│ SPA URL rewrite (CloudFront Function):                              │
│ function handler(event) {                                            │
│   var request = event.request;                                      │
│   var uri = request.uri;                                            │
│   // If no file extension, serve index.html (SPA routing)          │
│   if (!uri.includes('.')) {                                         │
│     request.uri = '/index.html';                                   │
│   }                                                                  │
│   return request;                                                    │
│ }                                                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Cache Invalidation

```
┌─────────────────────────────────────────────────────────────────────┐
│           CACHE INVALIDATION                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Force CloudFront to remove cached content before TTL expires.│
│                                                                       │
│ Console → Distribution → Invalidations → Create invalidation      │
│   Object paths:                                                      │
│   /index.html          (single file)                               │
│   /static/css/*        (all CSS files)                             │
│   /*                   (everything — expensive!)                   │
│                                                                       │
│ Pricing:                                                             │
│ ├── First 1,000 paths/month: FREE                                 │
│ ├── Additional: $0.005 per path                                    │
│ ├── Wildcard (/*) counts as ONE path                              │
│ └── Takes 5-10 minutes to propagate                                │
│                                                                       │
│ CLI:                                                                 │
│ aws cloudfront create-invalidation \                                 │
│   --distribution-id E12345 \                                         │
│   --paths "/index.html" "/static/css/*"                             │
│                                                                       │
│ Better approach: Cache busting!                                      │
│ ├── Instead of invalidating, change the filename                   │
│ ├── /static/app.abc123.js (content hash in filename)               │
│ ├── New deploy → /static/app.def456.js                             │
│ ├── Old file stays cached until TTL (no harm)                      │
│ ├── New file fetched from origin (cache miss)                      │
│ └── Most build tools do this automatically (Webpack, Vite)        │
│                                                                       │
│ ⚡ Use cache busting for assets, invalidation for HTML/index files  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Security Features

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECURITY                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. AWS WAF Integration                                               │
│    ├── Attach WAF Web ACL to distribution                          │
│    ├── Rate limiting, IP blocking, SQL injection protection        │
│    ├── Bot control, account takeover protection                    │
│    └── See WAF chapter for details                                 │
│                                                                       │
│ 2. AWS Shield                                                        │
│    ├── Standard: Included free (Layer 3/4 DDoS protection)       │
│    ├── Advanced: $3,000/month (enhanced DDoS, 24/7 DRT support)  │
│    └── CloudFront has Shield Standard automatically               │
│                                                                       │
│ 3. Geo Restriction                                                   │
│    ├── Allow list: Only specific countries can access              │
│    ├── Block list: Specific countries blocked                      │
│    ├── Based on GeoIP database                                     │
│    ├── Returns 403 to blocked countries                            │
│    └── Configure: Distribution → Geographic restrictions          │
│                                                                       │
│ 4. Field-Level Encryption                                            │
│    ├── Encrypt specific form fields at edge                        │
│    ├── Only your application can decrypt (with private key)       │
│    ├── Intermediate services can't read sensitive data             │
│    └── Use case: Credit card numbers, SSNs                        │
│                                                                       │
│ 5. Response Headers Policy                                           │
│    ├── Add security headers at edge                                │
│    ├── HSTS, X-Frame-Options, CSP, CORS, etc.                    │
│    ├── No need to configure at origin                              │
│    └── Consistent security headers across all origins              │
│                                                                       │
│ 6. Origin Custom Headers                                             │
│    ├── Add secret header to origin requests                        │
│    ├── Origin validates: X-Origin-Verify: secret123               │
│    ├── Prevents direct access bypassing CloudFront                │
│    └── Defense in depth with OAC/OAI                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 10: Monitoring & Logs

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. CloudFront Console Metrics (free):                                │
│    ├── Requests, bytes downloaded/uploaded                         │
│    ├── Error rate (4xx, 5xx)                                       │
│    ├── Cache hit ratio (target: 80%+)                              │
│    └── Available in CloudFront → Monitoring                       │
│                                                                       │
│ 2. CloudWatch Metrics:                                               │
│    ├── Requests, BytesDownloaded, BytesUploaded                   │
│    ├── TotalErrorRate, 4xxErrorRate, 5xxErrorRate                  │
│    ├── CacheHitRate                                                 │
│    ├── OriginLatency                                                │
│    └── Set alarms: CacheHitRate < 70%, 5xxErrorRate > 1%          │
│                                                                       │
│ 3. Standard Logs (S3):                                               │
│    ├── Delivered to S3 bucket (gzip compressed)                    │
│    ├── ~5 minute delay                                              │
│    ├── Contains: timestamp, edge, IP, method, URI, status, etc.   │
│    ├── Analyze with Athena (SQL queries on log files)              │
│    └── Retain as long as you want                                  │
│                                                                       │
│ 4. Real-Time Logs (Kinesis):                                        │
│    ├── Stream to Kinesis Data Streams                              │
│    ├── Seconds delay (near real-time)                              │
│    ├── Choose specific fields to log                               │
│    ├── Analyze with Kinesis → Lambda / OpenSearch                 │
│    └── More expensive but immediate                                │
│                                                                       │
│ Key metric: Cache Hit Ratio                                          │
│ ├── Above 90%: Excellent                                           │
│ ├── 70-90%: Good                                                   │
│ ├── Below 70%: Review cache policies, TTLs, cache key             │
│ └── Improve: Reduce cache key components, increase TTL            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 11: Terraform Example

```hcl
# S3 origin bucket
resource "aws_s3_bucket" "assets" {
  bucket = "tc-prod-assets"
}

# OAC
resource "aws_cloudfront_origin_access_control" "oac" {
  name                              = "oac-prod-assets"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# Cache policy (static assets)
resource "aws_cloudfront_cache_policy" "static" {
  name        = "cp-static-assets"
  default_ttl = 86400
  max_ttl     = 31536000
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    headers_config {
      header_behavior = "none"
    }
    cookies_config {
      cookie_behavior = "none"
    }
    query_strings_config {
      query_string_behavior = "none"
    }
    enable_accept_encoding_gzip   = true
    enable_accept_encoding_brotli = true
  }
}

# Distribution
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Production CDN for techcorp.com"
  default_root_object = "index.html"
  price_class         = "PriceClass_All"
  http_version        = "http2and3"
  aliases             = ["techcorp.com", "www.techcorp.com"]
  web_acl_id          = aws_wafv2_web_acl.prod.arn

  # S3 origin (static assets)
  origin {
    domain_name              = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id                = "s3-assets"
    origin_access_control_id = aws_cloudfront_origin_access_control.oac.id
  }

  # ALB origin (API)
  origin {
    domain_name = aws_lb.prod.dns_name
    origin_id   = "alb-api"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }

    custom_header {
      name  = "X-Origin-Verify"
      value = "secret-key-abc123"
    }
  }

  # Default behavior (S3 SPA)
  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "s3-assets"
    cache_policy_id        = aws_cloudfront_cache_policy.static.id
    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.spa_rewrite.arn
    }
  }

  # API behavior
  ordered_cache_behavior {
    path_pattern           = "/api/*"
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "alb-api"
    cache_policy_id        = data.aws_cloudfront_cache_policy.disabled.id
    viewer_protocol_policy = "redirect-to-https"
  }

  # Static assets behavior (long cache)
  ordered_cache_behavior {
    path_pattern           = "/static/*"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "s3-assets"
    cache_policy_id        = aws_cloudfront_cache_policy.static.id
    viewer_protocol_policy = "redirect-to-https"
    compress               = true
  }

  # SSL certificate
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.wildcard.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  # Geo restriction
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  # Custom error page (SPA 404 → index.html)
  custom_error_response {
    error_code            = 404
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 10
  }

  custom_error_response {
    error_code            = 403
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 10
  }
}

# S3 bucket policy for OAC
resource "aws_s3_bucket_policy" "assets" {
  bucket = aws_s3_bucket.assets.id
  policy = jsonencode({
    Statement = [{
      Sid       = "AllowCloudFrontServicePrincipal"
      Effect    = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.assets.arn}/*"
      Condition = {
        StringEquals = {
          "AWS:SourceArn" = aws_cloudfront_distribution.main.arn
        }
      }
    }]
  })
}

# CloudFront Function (SPA rewrite)
resource "aws_cloudfront_function" "spa_rewrite" {
  name    = "spa-url-rewrite"
  runtime = "cloudfront-js-2.0"
  code    = <<-EOF
    function handler(event) {
      var request = event.request;
      var uri = request.uri;
      if (!uri.includes('.')) {
        request.uri = '/index.html';
      }
      return request;
    }
  EOF
}

# Route 53 ALIAS record
resource "aws_route53_record" "apex" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "techcorp.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false
  }
}
```

---

## Part 12: Real-World Patterns

### Startup (SPA + API)

```
Distribution: 1
Origins:
├── S3 bucket (React/Vue/Angular SPA)
└── ALB (Node.js/Python API)

Behaviors:
├── /api/* → ALB (no caching)
└── * → S3 (cache 24h, SPA rewrite function)

SSL: ACM wildcard cert (*.techcorp.com)
WAF: Basic rate limiting
Cache hit ratio: ~85%
Cost: ~$5-20/month (depends on traffic)

Cache busting: Vite/Webpack content hashes
Invalidation: Only /index.html on deploys
```

### Mid-Size (Multi-origin, Staging)

```
Distributions: 2 (prod + staging)

Prod distribution:
├── S3 assets (CSS, JS, images) → cache 365 days
├── ALB API → no caching, all methods
├── S3 media (uploads) → cache 30 days
└── S3 SPA → cache 1 hour, SPA rewrite

Features:
├── OAC for all S3 origins
├── WAF with rate limiting + IP blocking
├── Response headers policy (security headers)
├── CloudFront Functions for URL rewrite + redirects
├── Standard logging to S3 + Athena queries
└── CloudWatch alarms for error rate + cache hit ratio

Signed URLs: For premium content downloads
Cost: ~$50-200/month
```

### Enterprise (Global, Multi-Region)

```
Distributions: 5+ (per product/service)

Architecture:
├── Origin groups for failover (S3 cross-region replication)
├── Lambda@Edge for:
│   ├── A/B testing (route % users to different origins)
│   ├── Auth validation (JWT check at edge)
│   ├── Image optimization (resize on-the-fly)
│   └── Custom logging
├── Origin Shield enabled (reduce origin load)
├── Real-time logs → Kinesis → OpenSearch
├── WAF Advanced (bot control, account takeover protection)
├── Shield Advanced ($3,000/month)
├── Geo restriction for compliance
├── Field-level encryption for sensitive forms
└── Multiple cache policies per behavior

Monitoring:
├── Cache hit ratio dashboards per behavior
├── Origin latency alarms
├── Error rate alarms with SNS notifications
├── Cost anomaly detection
└── Monthly optimization reviews

Cost: ~$500-5,000/month (traffic dependent)
```

---

## Troubleshooting: Common CloudFront Issues

### "I deployed new content but still see the old version"

```
Solutions (in order of preference):
1. Cache busting: Use versioned file names (app.v2.js, style.abc123.css)
   → Best practice — no invalidation needed

2. Create an invalidation:
   Console → CloudFront → Distribution → Invalidations → Create
   Path: /* (all files) or /specific/file.js
   → 1,000 free paths/month, then $0.005 per path

3. Wait for TTL to expire (default 24 hours)
```

### "403 Forbidden when accessing S3 through CloudFront"

```
1. ☐ Is OAC (Origin Access Control) configured?
   Distribution → Origins → Edit → Origin access: "Origin access control settings"

2. ☐ Is the S3 bucket policy updated with OAC permissions?
   CloudFront gives you the policy to copy — paste it in S3 bucket policy

3. ☐ Is S3 Block Public Access enabled? (It should be — OAC works with it)
```

### "CORS errors in the browser"

```
1. ☐ Does the S3 bucket have a CORS configuration?
2. ☐ Is CloudFront forwarding the Origin header?
   Cache policy must include "Origin" in cache key headers
3. ☐ Are the response headers (Access-Control-Allow-Origin) being cached
   for the wrong origin? Include Origin in cache key.
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| ACM cert not in us-east-1 | CloudFront can't use it | Create cert in us-east-1 specifically |
| Using OAI instead of OAC | OAI is deprecated | Migrate to OAC |
| Caching error pages (4xx/5xx) | Users see errors long after fix | Set error caching TTL to 0-60s |
| Not enabling compression | 60-70% larger responses | Enable automatic compression in cache policy |

---

## Quick Reference

| Feature | Details |
|---------|---------|
| Edge locations | 450+ PoPs in 100+ cities |
| Default domain | d12345.cloudfront.net |
| Custom domain | Add CNAME + ACM cert (us-east-1!) |
| Price classes | All / NA+EU / NA+EU+Asia+ME+Africa |
| Cache invalidation | 1,000 free/month, then $0.005/path |
| Signed URLs | Time-limited access to content |
| CF Functions | JS only, <1ms, viewer events, $0.10/M |
| Lambda@Edge | Node/Python, 5-30s, all events, $0.60/M |
| OAC | Restrict S3 access to CloudFront only |
| Origin Shield | Extra cache layer, reduce origin load |
| HTTP versions | HTTP/1.1, HTTP/2, HTTP/3 (QUIC) |
| WAF | Attach Web ACL for L7 protection |
| Compression | Gzip + Brotli (automatic) |
| SLA | 99.9% availability |

---

## What's Next?

In the next chapter, we'll cover Elastic Load Balancing — AWS's load balancing services (ALB, NLB, GLB).

→ Next: [Chapter 11: Elastic Load Balancing](11-elb.md)

---

*Last Updated: May 2026*
