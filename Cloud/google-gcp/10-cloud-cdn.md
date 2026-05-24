# Chapter 10: Cloud CDN (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud CDN Fundamentals](#part-1-cloud-cdn-fundamentals)
- [Part 2: Architecture (CDN + Load Balancer)](#part-2-architecture-cdn--load-balancer)
- [Part 3: Cache Modes](#part-3-cache-modes)
- [Part 4: Cache Keys](#part-4-cache-keys)
- [Part 5: Signed URLs & Signed Cookies](#part-5-signed-urls--signed-cookies)
- [Part 6: Cache Invalidation](#part-6-cache-invalidation)
- [Part 7: Advanced Features](#part-7-advanced-features)
- [Part 8: Media CDN (Large-Scale Streaming)](#part-8-media-cdn-large-scale-streaming)
- [Part 9: Terraform Example](#part-9-terraform-example)
- [Part 10: Real-World Patterns](#part-10-real-world-patterns)
- [Console Walkthrough: Managing & Deleting CDN](#console-walkthrough-managing--deleting-cdn)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud CDN uses Google's globally distributed edge points of presence to cache content close to users. It's tightly integrated with external Application Load Balancers and Cloud Storage, leveraging the same infrastructure that serves YouTube, Gmail, and Google Search.

```
What you'll learn:
├── Cloud CDN Fundamentals
├── Architecture (CDN + Load Balancer)
│   ├── Why CDN needs a Load Balancer in GCP
│   ├── Backend services & backend buckets
│   └── URL maps
├── Cache Modes
│   ├── USE_ORIGIN_HEADERS
│   ├── CACHE_ALL_STATIC
│   └── FORCE_CACHE_ALL
├── Cache Keys
│   ├── Query string inclusion/exclusion
│   ├── Header-based cache keys
│   └── Cookie-based cache keys
├── Signed URLs & Signed Cookies
├── Cache Invalidation
├── CDN Policies & Advanced Features
├── Media CDN (for large-scale streaming)
└── Real-world patterns
```

---

## Part 1: Cloud CDN Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CLOUD CDN                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Key difference from AWS:                                             │
│ AWS CloudFront = standalone CDN service                             │
│ GCP Cloud CDN = feature of the Load Balancer!                      │
│                                                                       │
│ You can't create a Cloud CDN "distribution" independently.         │
│ You enable CDN on a Load Balancer backend.                         │
│                                                                       │
│ Architecture:                                                        │
│                                                                       │
│ [User] → [Google Edge PoP] → [Global HTTP(S) LB] → [Backend]     │
│           CDN cache here       URL map routing      origin         │
│                                                                       │
│ Google's CDN network:                                                │
│ ├── 180+ edge locations in 200+ cities                             │
│ ├── Uses same network as YouTube/Google Search                     │
│ ├── Anycast IP (single global IP)                                  │
│ └── Integrated with Cloud Armor (WAF)                              │
│                                                                       │
│ Cloud CDN vs CloudFront:                                             │
│ ┌──────────────────────┬──────────────────┬────────────────────┐   │
│ │ Feature              │ CloudFront       │ Cloud CDN          │   │
│ ├──────────────────────┼──────────────────┼────────────────────┤   │
│ │ Standalone service   │ Yes              │ No (needs LB)      │   │
│ │ Edge locations       │ 450+             │ 180+               │   │
│ │ Edge compute         │ Lambda@Edge, CF  │ No (use Cloud Run) │   │
│ │                      │ Functions        │                    │   │
│ │ WAF                  │ AWS WAF          │ Cloud Armor        │   │
│ │ Signed URLs          │ Yes              │ Yes                │   │
│ │ Cache invalidation   │ Yes (path-based) │ Yes (path/tag)     │   │
│ │ Cache modes          │ Policy-based     │ 3 modes            │   │
│ │ Origin types         │ S3, ALB, custom  │ Backend svc/bucket │   │
│ │ Custom domains       │ ACM cert         │ Google-managed cert│   │
│ │ HTTP/3               │ Yes              │ Yes                │   │
│ │ Price (per GB)       │ $0.085           │ $0.08              │   │
│ └──────────────────────┴──────────────────┴────────────────────┘   │
│                                                                       │
│ Pricing:                                                             │
│ ├── Cache egress: $0.02-$0.20/GB (varies by region)               │
│ ├── Cache fill (origin fetch): Regular egress rates                │
│ ├── HTTP requests: $0.0075/10,000 requests                        │
│ ├── Cache invalidation: Free (up to rate limits)                   │
│ └── Cheaper than CloudFront for high-traffic sites                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Architecture (CDN + Load Balancer)

```
┌─────────────────────────────────────────────────────────────────────┐
│           GCP CDN ARCHITECTURE                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ To use Cloud CDN, you need:                                         │
│ 1. External Application Load Balancer (HTTP/S)                     │
│ 2. Backend service or backend bucket                               │
│ 3. Enable CDN on the backend                                       │
│                                                                       │
│ ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌──────────────┐   │
│ │ Users    │──►│ Google   │──►│ URL Map   │──►│ Backend      │   │
│ │          │   │ Frontend │   │ (routing) │   │ (with CDN)   │   │
│ │          │   │ (Global  │   │           │   │              │   │
│ │          │   │  IP+SSL) │   │ /api/*→BS │   │ Backend Svc  │   │
│ │          │   │          │   │ /*→BB     │   │ (GCE, GKE,   │   │
│ │          │   │          │   │           │   │  Cloud Run)  │   │
│ │          │   │          │   │           │   │              │   │
│ │          │   │          │   │           │   │ Backend Bucket│   │
│ │          │   │          │   │           │   │ (GCS)        │   │
│ └──────────┘   └──────────┘   └───────────┘   └──────────────┘   │
│                     ▲                                                │
│              CDN cache here                                          │
│              (at Google edge)                                        │
│                                                                       │
│ Backend types:                                                       │
│                                                                       │
│ 1. BACKEND SERVICE (dynamic content):                                │
│    ├── Instance groups (GCE VMs)                                   │
│    ├── Network Endpoint Groups (NEGs) — GKE pods, Cloud Run       │
│    ├── Enable CDN ☑ (checkbox on backend service)                 │
│    └── Used for: APIs, SSR apps, dynamic pages                    │
│                                                                       │
│ 2. BACKEND BUCKET (static content):                                  │
│    ├── Points directly to a GCS bucket                             │
│    ├── Enable CDN ☑ (checkbox on backend bucket)                  │
│    ├── No need for a separate server                               │
│    └── Used for: Static websites, assets (CSS, JS, images)        │
│                                                                       │
│ Comparison with AWS:                                                 │
│ AWS CloudFront:                                                      │
│   S3 origin → CloudFront → user                                    │
│   ALB origin → CloudFront → user                                   │
│                                                                       │
│ GCP Cloud CDN:                                                       │
│   GCS → Backend Bucket → LB (CDN enabled) → user                  │
│   GCE → Backend Service → LB (CDN enabled) → user                 │
│                                                                       │
│ ⚡ GCP approach = more configuration steps                          │
│    but gives you LB features (health checks, routing) for free!    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Setting Up Cloud CDN (Full Walkthrough)

```
Step 1: Reserve a static IP
──────────────────────────
Console → VPC network → IP addresses → Reserve external static address
  Name: [ip-prod-lb]
  Network tier: Premium
  IP version: IPv4
  Type: Global
  [Reserve]

Step 2: Create a GCS bucket for static content
──────────────────────────────────────────────
Console → Cloud Storage → Create bucket
  Name: [tc-prod-static]
  Location: Multi-region (us or asia)
  Access: Uniform (not fine-grained)
  Public access: Allow public (allUsers reader) OR use signed URLs

Step 3: Create Backend Bucket
─────────────────────────────
Console → Network services → Load balancing → Backends → Create backend bucket

┌─────────────────────────────────────────────────────────────────┐
│              CREATE BACKEND BUCKET                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:             [bb-static-assets]                            │
│ Cloud Storage bucket: [tc-prod-static ▼]                       │
│ Enable Cloud CDN: ☑                                             │
│                                                                   │
│ CDN settings (when enabled):                                    │
│   Cache mode: [Cache static content ▼]                         │
│   (see cache modes section below)                               │
│                                                                   │
│   Client TTL:  [3600] seconds (1 hour)                         │
│   Default TTL: [86400] seconds (24 hours)                      │
│   Max TTL:     [2592000] seconds (30 days)                     │
│                                                                   │
│   Negative caching: ☑ Enable                                   │
│   404: 120 seconds                                              │
│   410: 120 seconds                                              │
│                                                                   │
│   Serve stale content: ☐                                        │
│   (Serve cached content while revalidating with origin)        │
│                                                                   │
│   Signed URL cache: ☐                                           │
│   (Include/exclude query params)                               │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Step 4: Create Backend Service (for API)
───────────────────────────────────────
Console → Load balancing → Backends → Create backend service

┌─────────────────────────────────────────────────────────────────┐
│              CREATE BACKEND SERVICE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:               [bs-api]                                    │
│ Backend type:       [Instance group ▼] / [NEG ▼]              │
│ Protocol:           [HTTPS ▼]                                  │
│ Named port:         [https]                                     │
│ Timeout:            [30] seconds                                │
│                                                                   │
│ Backends:                                                        │
│   Instance group: [ig-api-prod ▼]                              │
│   Port numbers: [443]                                           │
│   Balancing mode: [Utilization ▼]                              │
│   Max utilization: [80%]                                        │
│                                                                   │
│ Health check: [hc-api-https ▼]                                 │
│                                                                   │
│ Enable Cloud CDN: ☑ (or ☐ for API that shouldn't cache)       │
│   Cache mode: [Use origin headers ▼]                           │
│                                                                   │
│ Cloud Armor: [policy-prod ▼] (WAF)                             │
│                                                                   │
│ [Create]                                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Step 5: Create URL Map
─────────────────────
Console → Load balancing → Create load balancer → HTTP(S)

┌─────────────────────────────────────────────────────────────────┐
│              URL MAP (ROUTING RULES)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Default backend: [bb-static-assets ▼]                          │
│                                                                   │
│ Host and path rules:                                            │
│   Host: techcorp.com                                            │
│   ├── /api/*    → bs-api (backend service)                    │
│   ├── /static/* → bb-static-assets (backend bucket)           │
│   └── /*        → bb-static-assets (default)                  │
│                                                                   │
│   Host: api.techcorp.com                                        │
│   └── /*        → bs-api                                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Step 6: Create Frontend (HTTPS)
──────────────────────────────
  Name: [fe-prod-https]
  Protocol: HTTPS
  IP address: [ip-prod-lb ▼] (reserved static IP)
  Port: 443
  Certificate: [cert-techcorp ▼] (Google-managed or self-managed)
  SSL policy: [ssl-policy-modern ▼] (TLS 1.2+)
  HTTP/3 (QUIC): ☑ Enable

Step 7: DNS record
──────────────────
  techcorp.com A → [static IP from Step 1]
  (In Cloud DNS: simple A record, no ALIAS needed!)
```

---

## Part 3: Cache Modes

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CACHE MODES                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. USE_ORIGIN_HEADERS                                                │
│    ├── Respects Cache-Control headers from origin                  │
│    ├── If origin sends no cache headers → NOT cached              │
│    ├── Most control for the developer                              │
│    ├── Use for: APIs, dynamic content with varying cacheability   │
│    └── Equivalent to CloudFront "honor origin headers"            │
│                                                                       │
│    Origin sends: Cache-Control: public, max-age=3600 → cached 1h │
│    Origin sends: Cache-Control: no-store → not cached             │
│    Origin sends: nothing → not cached                              │
│                                                                       │
│ 2. CACHE_ALL_STATIC (default for backend buckets)                    │
│    ├── Caches static content types automatically                   │
│    ├── Static = images, CSS, JS, fonts, video, audio, etc.        │
│    ├── Identified by Content-Type header or file extension        │
│    ├── Dynamic content follows Cache-Control headers              │
│    ├── Use for: Mixed content (static assets + dynamic pages)     │
│    └── Most common mode                                            │
│                                                                       │
│    Image (image/jpeg): Cached with Default TTL                     │
│    HTML (text/html) with no Cache-Control: NOT cached              │
│    HTML with Cache-Control: public → cached per header            │
│                                                                       │
│ 3. FORCE_CACHE_ALL                                                   │
│    ├── Caches EVERYTHING regardless of Cache-Control              │
│    ├── Ignores no-store, no-cache, private directives!            │
│    ├── ⚠️ DANGEROUS for dynamic/personalized content              │
│    ├── Use for: Fully static sites, documentation sites           │
│    └── Make sure no user-specific content is served!              │
│                                                                       │
│ ⚠️ FORCE_CACHE_ALL can leak private data!                           │
│    User A's profile page cached → User B sees User A's data!     │
│    Only use for truly static, non-personalized content.           │
│                                                                       │
│ Cache mode comparison:                                               │
│ ┌────────────────────┬─────────────────┬──────────────────────┐   │
│ │ Content            │ USE_ORIGIN      │ CACHE_ALL_STATIC     │   │
│ ├────────────────────┼─────────────────┼──────────────────────┤   │
│ │ image.jpg          │ Per C-C header  │ Auto-cached          │   │
│ │ style.css          │ Per C-C header  │ Auto-cached          │   │
│ │ page.html (no C-C) │ NOT cached      │ NOT cached           │   │
│ │ page.html (public) │ Cached per C-C  │ Cached per C-C       │   │
│ │ api response       │ Per C-C header  │ NOT cached (dynamic) │   │
│ └────────────────────┴─────────────────┴──────────────────────┘   │
│                                                                       │
│ TTL Settings (per cache mode):                                       │
│ ├── Client TTL: Max-age sent to browser (viewer-side caching)    │
│ ├── Default TTL: Used when origin has no max-age header           │
│ │   (only for CACHE_ALL_STATIC and FORCE_CACHE_ALL)              │
│ ├── Max TTL: Upper bound, overrides origin's max-age if higher   │
│ └── Defaults: Client=3600, Default=3600, Max=86400               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Cache Keys

```
┌─────────────────────────────────────────────────────────────────────┐
│                  CACHE KEYS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cache key determines what makes a cached object unique.             │
│                                                                       │
│ Default cache key: Protocol + Host + Path                           │
│   https://techcorp.com/images/logo.png                              │
│                                                                       │
│ You can include/exclude:                                             │
│ ├── Query strings                                                  │
│ ├── HTTP headers                                                    │
│ └── Named cookies                                                   │
│                                                                       │
│ Query string configuration:                                          │
│ ├── Include all (default): ?v=1&color=red → part of cache key    │
│ ├── Include specific: Only ?v= matters for cache key              │
│ ├── Exclude all: Query strings ignored in cache key               │
│ └── Exclude specific: Ignore ?utm_source= in cache key           │
│                                                                       │
│ Header-based cache key:                                              │
│ ├── Include specific headers: Accept-Language, Device-Type        │
│ ├── Different cache per language or device                         │
│ └── ⚠️ More headers = more cache variations = lower hit rate      │
│                                                                       │
│ Cookie-based cache key:                                              │
│ ├── Include specific cookies                                       │
│ ├── Example: Include "country" cookie for localized content       │
│ └── ⚠️ Never include session cookies (per-user = no caching)     │
│                                                                       │
│ Configure:                                                           │
│ Backend service/bucket → CDN settings → Cache key policy          │
│                                                                       │
│ gcloud compute backend-services update bs-api \                      │
│   --global \                                                          │
│   --cache-key-include-query-string \                                  │
│   --cache-key-query-string-whitelist=version,lang                   │
│                                                                       │
│ gcloud compute backend-buckets update bb-static \                    │
│   --cache-key-include-http-header=Accept-Language                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Signed URLs & Signed Cookies

```
┌─────────────────────────────────────────────────────────────────────┐
│           SIGNED URLs & SIGNED COOKIES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Restrict access to CDN content with time-limited signatures. │
│       Same concept as AWS CloudFront signed URLs/cookies.          │
│                                                                       │
│ Signed URL:                                                          │
│ https://techcorp.com/premium/video.mp4                              │
│   ?Expires=1716000000                                               │
│   &KeyName=cdn-key-prod                                             │
│   &Signature=BASE64_SIGNATURE                                      │
│                                                                       │
│ Signed Cookie:                                                       │
│ Set-Cookie: Cloud-CDN-Cookie=URLPrefix=BASE64:Expires=...:         │
│             KeyName=...:Signature=...                               │
│                                                                       │
│ Setup:                                                               │
│ 1. Create a signing key:                                            │
│    # Generate random key (exactly 128 bits for HMAC-SHA1)          │
│    head -c 16 /dev/urandom | base64 | tr +/ -_ > cdn-key.txt      │
│                                                                       │
│ 2. Add key to backend:                                              │
│    gcloud compute backend-services \                                 │
│      add-signed-url-key bs-api \                                    │
│      --global \                                                       │
│      --key-name cdn-key-prod \                                       │
│      --key-file cdn-key.txt                                          │
│                                                                       │
│ 3. Enable signed URL requirement:                                   │
│    Backend bucket/service → CDN → Require signed URL               │
│                                                                       │
│ 4. Generate signed URL in your app:                                 │
│    gcloud compute sign-url \                                         │
│      "https://techcorp.com/premium/video.mp4" \                     │
│      --key-name cdn-key-prod \                                       │
│      --key-file cdn-key.txt \                                        │
│      --expires-in 1h                                                  │
│                                                                       │
│ ⚠️ GCP uses HMAC-SHA1 keys (symmetric)                              │
│    AWS uses RSA keys (asymmetric)                                   │
│    GCP approach is simpler to manage.                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Cache Invalidation

```
┌─────────────────────────────────────────────────────────────────────┐
│           CACHE INVALIDATION                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Force Cloud CDN to evict cached content.                     │
│                                                                       │
│ Two methods:                                                         │
│                                                                       │
│ 1. Path-based invalidation (like AWS):                              │
│    gcloud compute url-maps invalidate-cdn-cache lb-prod \           │
│      --path="/images/logo.png"        (single file)                │
│                                                                       │
│    gcloud compute url-maps invalidate-cdn-cache lb-prod \           │
│      --path="/static/*"               (wildcard)                   │
│                                                                       │
│    gcloud compute url-maps invalidate-cdn-cache lb-prod \           │
│      --path="/*"                      (everything)                 │
│                                                                       │
│ 2. Tag-based invalidation (GCP-unique!):                            │
│    ├── Tag cached responses with custom cache tags                 │
│    ├── Origin sends: Cache-Tag: product-123, category-shoes       │
│    ├── Invalidate by tag: all items with tag "product-123"        │
│    └── Much more precise than path-based!                          │
│                                                                       │
│    gcloud compute url-maps invalidate-cdn-cache lb-prod \           │
│      --tag="product-123"                                            │
│                                                                       │
│    Use case: Product updated in DB →                               │
│    invalidate cache tag "product-123" →                             │
│    all cached pages showing that product are refreshed.            │
│                                                                       │
│ Limits:                                                              │
│ ├── 1 invalidation request per minute per URL map                  │
│ ├── No cost for invalidations                                      │
│ ├── Takes up to a few minutes to propagate                         │
│ └── Like AWS, prefer cache busting (versioned filenames)           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Advanced Features

```
┌─────────────────────────────────────────────────────────────────────┐
│           ADVANCED FEATURES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Negative Caching                                                  │
│    ├── Cache error responses (404, 410) for a short time           │
│    ├── Prevents origin from being hammered with 404 requests      │
│    ├── Default: 404 cached for 120 seconds                        │
│    └── Configurable per error code                                 │
│                                                                       │
│ 2. Serve Stale Content                                               │
│    ├── Serve cached content while revalidating with origin         │
│    ├── If origin is down, serve stale instead of error            │
│    ├── Configure max-stale duration                                │
│    └── Better user experience during origin issues                 │
│                                                                       │
│ 3. Bypass Cache (for debugging)                                      │
│    ├── Send request with: Pragma: no-cache                        │
│    ├── Or: Cache-Control: no-cache                                │
│    ├── ⚠️ Only works if you DON'T use FORCE_CACHE_ALL            │
│    └── Useful for testing cache behavior                           │
│                                                                       │
│ 4. Custom Response Headers                                           │
│    ├── Add headers to CDN responses                                │
│    ├── Security headers, CORS, custom headers                     │
│    ├── Configure on backend service/bucket                        │
│    └── gcloud compute backend-services update bs-api \             │
│          --custom-response-header="X-Frame-Options: SAMEORIGIN"   │
│                                                                       │
│ 5. Cloud Armor Integration (WAF)                                     │
│    ├── Attach Cloud Armor security policy to backend service      │
│    ├── IP allowlisting/denylisting                                 │
│    ├── Rate limiting                                                │
│    ├── WAF rules (SQL injection, XSS protection)                  │
│    ├── Geo-based access control                                    │
│    ├── Bot management                                              │
│    └── Edge security policy (evaluated at CDN edge)               │
│                                                                       │
│ 6. Logging                                                           │
│    ├── Cloud CDN logs integrated with Cloud Logging               │
│    ├── Automatic, no separate configuration                       │
│    ├── Fields: cacheHit, cacheLookup, cacheFillBytes              │
│    ├── Export to BigQuery for analysis                             │
│    └── Filter: resource.type="http_load_balancer"                 │
│         httpRequest.cacheHit=true                                  │
│                                                                       │
│ 7. Monitoring                                                        │
│    ├── Cloud Monitoring dashboard (automatic)                     │
│    ├── Metrics: cdn/backend_latencies, cdn/fill_bytes             │
│    ├── Cache hit ratio metric                                      │
│    └── Set alerts: Cache hit rate < 70%                           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Media CDN (Large-Scale Streaming)

```
┌─────────────────────────────────────────────────────────────────────┐
│           MEDIA CDN                                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Separate CDN service for large-scale media delivery.         │
│       Uses the same infrastructure as YouTube.                     │
│                                                                       │
│ When to use:                                                         │
│ ├── Cloud CDN: General web content (sites, APIs, assets)          │
│ ├── Media CDN: Video streaming, large file downloads, gaming      │
│ │   assets, software distribution                                  │
│ └── If you're serving TB/PB of video → Media CDN                  │
│                                                                       │
│ Key features:                                                        │
│ ├── YouTube-grade infrastructure                                   │
│ ├── Origin shielding (prevent origin overload)                     │
│ ├── Token authentication (like signed URLs, more features)        │
│ ├── Prefetching (predict and cache before request)                │
│ ├── Live streaming support                                        │
│ └── Much deeper edge caching hierarchy                             │
│                                                                       │
│ ⚠️ Media CDN is a separate product with different pricing.         │
│    For most web applications, standard Cloud CDN is sufficient.   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Terraform Example

```hcl
# Reserved static IP
resource "google_compute_global_address" "lb_ip" {
  name = "ip-prod-lb"
}

# GCS bucket for static content
resource "google_storage_bucket" "static" {
  name     = "tc-prod-static"
  location = "ASIA"

  website {
    main_page_suffix = "index.html"
    not_found_page   = "404.html"
  }
}

# Make bucket publicly readable (for CDN)
resource "google_storage_bucket_iam_member" "public_read" {
  bucket = google_storage_bucket.static.name
  role   = "roles/storage.objectViewer"
  member = "allUsers"
}

# Backend bucket (GCS + CDN)
resource "google_compute_backend_bucket" "static" {
  name        = "bb-static-assets"
  bucket_name = google_storage_bucket.static.name
  enable_cdn  = true

  cdn_policy {
    cache_mode                   = "CACHE_ALL_STATIC"
    default_ttl                  = 86400
    max_ttl                      = 2592000
    client_ttl                   = 3600
    negative_caching             = true
    serve_while_stale            = 86400
    signed_url_cache_max_age_sec = 3600

    negative_caching_policy {
      code = 404
      ttl  = 120
    }
  }
}

# Health check
resource "google_compute_health_check" "api" {
  name = "hc-api-https"

  https_health_check {
    port         = 443
    request_path = "/health"
  }

  check_interval_sec  = 10
  timeout_sec         = 5
  healthy_threshold   = 2
  unhealthy_threshold = 3
}

# Backend service (API + CDN disabled for API)
resource "google_compute_backend_service" "api" {
  name                  = "bs-api"
  protocol              = "HTTPS"
  timeout_sec           = 30
  health_checks         = [google_compute_health_check.api.id]
  enable_cdn            = false  # No CDN for API
  security_policy       = google_compute_security_policy.prod.id

  backend {
    group           = google_compute_instance_group_manager.api.instance_group
    balancing_mode  = "UTILIZATION"
    max_utilization = 0.8
  }
}

# URL map (routing)
resource "google_compute_url_map" "main" {
  name            = "lb-prod"
  default_service = google_compute_backend_bucket.static.id

  host_rule {
    hosts        = ["techcorp.com", "www.techcorp.com"]
    path_matcher = "main"
  }

  path_matcher {
    name            = "main"
    default_service = google_compute_backend_bucket.static.id

    path_rule {
      paths   = ["/api/*"]
      service = google_compute_backend_service.api.id
    }

    path_rule {
      paths   = ["/static/*"]
      service = google_compute_backend_bucket.static.id
    }
  }
}

# Google-managed SSL certificate
resource "google_compute_managed_ssl_certificate" "main" {
  name = "cert-techcorp"
  managed {
    domains = ["techcorp.com", "www.techcorp.com"]
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

# Frontend (forwarding rule)
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

resource "google_compute_global_forwarding_rule" "http_redirect" {
  name                  = "fe-prod-http-redirect"
  ip_address            = google_compute_global_address.lb_ip.address
  port_range            = "80"
  target                = google_compute_target_http_proxy.redirect.id
  load_balancing_scheme = "EXTERNAL_MANAGED"
}

# DNS record
resource "google_dns_record_set" "apex" {
  name         = "techcorp.com."
  type         = "A"
  ttl          = 300
  managed_zone = google_dns_managed_zone.public.name
  rrdatas      = [google_compute_global_address.lb_ip.address]
}
```

---

## Part 10: Real-World Patterns

### Startup

```
Setup: Global HTTP(S) LB + Cloud CDN

Backends:
├── Backend bucket: GCS (React SPA, static assets)
│   CDN: CACHE_ALL_STATIC, TTL 24h
└── Backend service: Cloud Run (API)
    CDN: Disabled

URL map:
├── /api/* → Cloud Run
└── /* → GCS bucket

SSL: Google-managed cert (free, auto-renewed!)
Cloud Armor: Basic rate limiting
Cost: ~$20/month (LB minimum) + $0-5/month CDN
```

### Mid-Size

```
Setup: Global HTTP(S) LB + Cloud CDN + Cloud Armor

Backends:
├── Backend bucket: GCS (static assets, CDN ON)
│   Cache mode: CACHE_ALL_STATIC
│   TTL: 365 days for versioned assets
├── Backend service: GKE NEG (API, CDN OFF)
├── Backend service: Cloud Run (docs site, CDN ON)
│   Cache mode: USE_ORIGIN_HEADERS
└── Backend bucket: GCS (media uploads, CDN ON)
    Signed URLs for premium content

Cloud Armor:
├── Rate limiting (100 req/min per IP)
├── Geo-blocking (block high-risk countries)
└── WAF rules (OWASP top 10)

Monitoring:
├── Cache hit ratio dashboard
├── Alert: Hit ratio < 70%
├── BigQuery export of CDN logs
└── Monthly cost review

Cost: ~$30-100/month
```

### Enterprise

```
Setup: Multiple LBs per product + Cloud CDN + Cloud Armor

Main site LB:
├── Backend bucket: Multi-region GCS (global static assets)
├── Backend service: Regional MIG (SSR app)
└── CDN: CACHE_ALL_STATIC with cache tags

API LB:
├── Backend service: GKE NEGs (API pods)
├── CDN: USE_ORIGIN_HEADERS (selective caching)
└── Cloud Armor: Advanced WAF + bot management

Media delivery:
├── Media CDN (video streaming)
├── Token authentication
└── Origin shielding

Cache strategy:
├── Versioned assets: max TTL (365 days)
├── HTML: Short TTL (5 min) or no cache
├── API: Per-endpoint cache policies
├── Cache tags for fine-grained invalidation
└── Serve stale enabled for resilience

Security:
├── Cloud Armor edge security policies
├── Signed URLs for premium content
├── Google-managed certs + custom SSL policy
└── HTTP/3 enabled globally

Cost: ~$200-2,000/month
```

---

## Quick Reference

| Feature | Details |
|---------|---------|
| Standalone CDN? | No — enable on LB backends |
| Edge locations | 180+ PoPs in 200+ cities |
| Cache modes | USE_ORIGIN_HEADERS, CACHE_ALL_STATIC, FORCE_CACHE_ALL |
| Cache key | URL + optional query/headers/cookies |
| Cache invalidation | Path-based + tag-based (free!) |
| Signed URLs | HMAC-SHA1 symmetric keys |
| Edge compute | None (use Cloud Run / serverless) |
| WAF | Cloud Armor (separate service) |
| SSL certs | Google-managed (free, auto-renew) |
| HTTP/3 | Yes (QUIC) |
| Compression | Automatic (gzip, brotli) |
| Media CDN | Separate product for video/streaming |
| SLA | Covered under LB SLA (99.99%) |

---

## Console Walkthrough: Managing & Deleting CDN

### Disabling CDN on a Backend

```
Console → Network services → Load balancing → [your load balancer] → Edit

┌─────────────────────────────────────────────────────────────────┐
│           DISABLE CDN ON BACKEND                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ For Backend Service:                                            │
│ 1. Click "Backend configuration"                               │
│ 2. Click the pencil icon (edit) on the backend service         │
│ 3. Scroll to "Cloud CDN"                                       │
│ 4. Uncheck ☐ "Enable Cloud CDN"                                │
│ 5. Click [Update]                                               │
│ 6. Click [Update] on the load balancer                         │
│                                                                   │
│ For Backend Bucket:                                             │
│ 1. Console → Network services → Load balancing                 │
│ 2. Go to "Backends" tab → Backend buckets                     │
│ 3. Click your backend bucket → [Edit]                          │
│ 4. Uncheck ☐ "Enable Cloud CDN"                                │
│ 5. Click [Update]                                               │
│                                                                   │
│ CLI:                                                             │
│   gcloud compute backend-services update bs-api \               │
│     --global --no-enable-cdn                                    │
│                                                                   │
│   gcloud compute backend-buckets update bb-static \             │
│     --no-enable-cdn                                             │
│                                                                   │
│ ⚠️ Disabling CDN does NOT invalidate existing cache.            │
│    Cached content expires naturally based on TTL.               │
│    To clear immediately: invalidate cache first, then disable. │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Invalidating Cache from Console

```
Console → Network services → Load balancing → [your load balancer]
→ Click "Caching" tab (or look for cache invalidation option)

┌─────────────────────────────────────────────────────────────────┐
│           INVALIDATE CACHE                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Method 1: From Load Balancer details                            │
│ 1. Console → Network services → Load balancing                 │
│ 2. Click your load balancer name                                │
│ 3. Click [Cache invalidation] button at the top               │
│ 4. Enter path pattern:                                          │
│    ├── /images/logo.png     (single file)                     │
│    ├── /static/*            (wildcard — all in /static/)      │
│    └── /*                   (everything)                       │
│ 5. Click [Invalidate]                                           │
│                                                                   │
│ Method 2: Using gcloud                                          │
│   gcloud compute url-maps invalidate-cdn-cache lb-prod \        │
│     --path="/static/*"                                          │
│                                                                   │
│   # Tag-based invalidation                                      │
│   gcloud compute url-maps invalidate-cdn-cache lb-prod \        │
│     --tag="product-123"                                         │
│                                                                   │
│ ⚠️ Limit: 1 invalidation request per minute per URL map.        │
│    Takes a few minutes to propagate to all edge locations.      │
│    Free of charge (unlike some other CDN providers).            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Deleting Backend Buckets from Console

```
┌─────────────────────────────────────────────────────────────────┐
│           DELETE BACKEND BUCKET                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ⚠️ You cannot delete a backend bucket that is still            │
│    referenced by a URL map. Remove references first!            │
│                                                                   │
│ Step 1: Remove from URL map                                    │
│ 1. Console → Network services → Load balancing                 │
│ 2. Click your load balancer → [Edit]                           │
│ 3. Click "Routing rules" (URL map)                              │
│ 4. Remove or change any path rules pointing to this            │
│    backend bucket                                               │
│ 5. Change default backend if it's this backend bucket          │
│ 6. Click [Update]                                               │
│                                                                   │
│ Step 2: Delete the backend bucket                               │
│ 1. Console → Network services → Load balancing                 │
│ 2. Go to "Backends" tab                                        │
│ 3. Find your backend bucket in the list                        │
│ 4. Click the checkbox next to it                                │
│ 5. Click [Delete] at the top                                   │
│ 6. Confirm deletion                                             │
│                                                                   │
│ CLI:                                                             │
│   # Check if backend bucket is in use                           │
│   gcloud compute url-maps describe lb-prod --global             │
│                                                                   │
│   # Delete backend bucket                                       │
│   gcloud compute backend-buckets delete bb-static-assets        │
│                                                                   │
│ ⚠️ Deleting backend bucket does NOT delete the GCS bucket.     │
│    The underlying Cloud Storage bucket remains intact.          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Checking Cache Hit Rates in Cloud Monitoring

```
┌─────────────────────────────────────────────────────────────────┐
│           MONITORING CACHE HIT RATES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Method 1: Load Balancer details page                            │
│ 1. Console → Network services → Load balancing                 │
│ 2. Click your load balancer                                     │
│ 3. Click "Monitoring" tab                                      │
│ 4. View "CDN cache hit ratio" chart                            │
│    ├── Shows % of requests served from cache                  │
│    ├── Healthy target: > 70-80% for static content            │
│    └── Adjust time range as needed                             │
│                                                                   │
│ Method 2: Cloud Monitoring Metrics Explorer                    │
│ 1. Console → Monitoring → Metrics Explorer                     │
│ 2. Search for metric:                                           │
│    ├── loadbalancing.googleapis.com/https/                     │
│    │   request_count (filter by cache_result)                  │
│    ├── cache_result values:                                    │
│    │   ├── HIT: Served from edge cache                        │
│    │   ├── MISS: Fetched from origin                          │
│    │   ├── DISABLED: CDN not enabled                          │
│    │   └── REVALIDATED: Cache revalidated with origin         │
│    └── cdn/fill_bytes_count (origin fetch volume)             │
│ 3. Create a chart or dashboard                                 │
│                                                                   │
│ Method 3: Cloud Logging query                                   │
│   resource.type="http_load_balancer"                            │
│   httpRequest.cacheHit=true                                     │
│   (Shows individual requests served from cache)                │
│                                                                   │
│ Setting up an alert:                                             │
│ 1. Console → Monitoring → Alerting → Create policy             │
│ 2. Metric: CDN cache hit ratio                                 │
│ 3. Condition: Cache hit ratio < 70% for 15 minutes            │
│ 4. Notification: Email / Slack / PagerDuty                    │
│                                                                   │
│ 💡 Low cache hit rate? Check:                                    │
│    ├── Cache mode set correctly?                               │
│    ├── Origin sending proper Cache-Control headers?            │
│    ├── Cache key too specific (too many query params)?        │
│    ├── Content too dynamic (short TTL)?                        │
│    └── Vary header causing excessive cache variations?        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

In the next chapter, we'll cover Cloud Load Balancing — GCP's suite of load balancing options.

→ Next: [Chapter 11: Cloud Load Balancing](11-cloud-load-balancing.md)

---

*Last Updated: May 2026*
