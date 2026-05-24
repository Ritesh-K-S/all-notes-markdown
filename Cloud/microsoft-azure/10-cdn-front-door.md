# Chapter 10: Azure CDN & Azure Front Door

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure CDN vs Azure Front Door](#part-1-azure-cdn-vs-azure-front-door)
- [Part 2: Azure Front Door — Components](#part-2-azure-front-door--components)
- [Part 3: Rules Engine](#part-3-rules-engine)
- [Part 4: Custom Domains & SSL](#part-4-custom-domains--ssl)
- [Part 5: Caching](#part-5-caching)
- [Part 6: WAF Integration (Premium)](#part-6-waf-integration-premium)
- [Part 7: Private Link Origins (Premium)](#part-7-private-link-origins-premium)
- [Part 8: Azure CDN Classic (Brief Reference)](#part-8-azure-cdn-classic-brief-reference)
- [Part 9: Bicep Example](#part-9-bicep-example)
- [Part 10: Real-World Patterns](#part-10-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure offers two main content delivery services: **Azure CDN** (classic CDN) and **Azure Front Door** (global load balancer + CDN + WAF). Microsoft is converging these into a unified platform under the **Azure Front Door** brand, with CDN tiers.

```
What you'll learn:
├── Azure CDN vs Azure Front Door (which to use)
├── Azure Front Door (Standard/Premium)
│   ├── Profiles, endpoints, origins, origin groups
│   ├── Routes and routing rules
│   ├── Caching & cache policies
│   ├── Rules engine (URL rewrite, redirect, headers)
│   ├── Custom domains & SSL
│   └── WAF integration
├── Azure CDN (Classic)
│   ├── Profiles and endpoints
│   ├── Caching rules
│   ├── Microsoft, Akamai, Verizon tiers
│   └── When to still use classic CDN
├── Private Link origins
├── Security features
└── Real-world patterns
```

---

## Part 1: Azure CDN vs Azure Front Door

```
┌─────────────────────────────────────────────────────────────────────┐
│          AZURE CDN vs AZURE FRONT DOOR                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ History:                                                             │
│ ├── Azure CDN: Classic CDN with multiple provider tiers            │
│ ├── Azure Front Door: Global LB + CDN + WAF (newer)              │
│ └── Microsoft is merging them: Front Door = the future            │
│                                                                       │
│ Current tiers (Front Door based):                                    │
│ ├── Azure Front Door Standard: CDN + basic routing                │
│ ├── Azure Front Door Premium: CDN + advanced routing + WAF        │
│ └── Azure CDN (Classic): Legacy, still available                  │
│                                                                       │
│ ┌──────────────────────┬────────────┬────────────┬──────────────┐ │
│ │ Feature              │ CDN Classic│ FD Standard│ FD Premium    │ │
│ ├──────────────────────┼────────────┼────────────┼──────────────┤ │
│ │ Static caching       │ ✅          │ ✅          │ ✅            │ │
│ │ Dynamic acceleration │ ❌          │ ✅          │ ✅            │ │
│ │ Global load balancing│ ❌          │ ✅          │ ✅            │ │
│ │ Health probes        │ ❌          │ ✅          │ ✅            │ │
│ │ Failover             │ ❌          │ ✅          │ ✅            │ │
│ │ Rules engine         │ Basic      │ ✅          │ ✅            │ │
│ │ WAF                  │ ❌          │ ❌          │ ✅            │ │
│ │ Private Link origins │ ❌          │ ❌          │ ✅            │ │
│ │ Bot protection       │ ❌          │ ❌          │ ✅            │ │
│ │ Custom domains       │ ✅          │ ✅          │ ✅            │ │
│ │ Managed certs        │ ✅          │ ✅          │ ✅            │ │
│ │ HTTP/2               │ ✅          │ ✅          │ ✅            │ │
│ │ Compression          │ ✅          │ ✅          │ ✅            │ │
│ │ Geo-filtering        │ ✅          │ ✅ (rules)  │ ✅ (WAF)     │ │
│ │ Edge locations       │ 100+       │ 180+        │ 180+         │ │
│ └──────────────────────┴────────────┴────────────┴──────────────┘ │
│                                                                       │
│ Recommendation:                                                      │
│ ├── New projects: Use Azure Front Door Standard or Premium        │
│ ├── Need WAF: Front Door Premium                                  │
│ ├── Simple static CDN: Front Door Standard                        │
│ └── Legacy/existing: Azure CDN Classic (migrate when possible)   │
│                                                                       │
│ Comparison with AWS & GCP:                                           │
│ ┌──────────────────────┬──────────────────┬────────────────────┐   │
│ │ Azure               │ AWS Equivalent   │ GCP Equivalent     │   │
│ ├──────────────────────┼──────────────────┼────────────────────┤   │
│ │ Front Door           │ CloudFront       │ Cloud CDN + LB     │   │
│ │ Front Door routing   │ Route 53 routing │ Global LB           │   │
│ │ Front Door WAF       │ AWS WAF          │ Cloud Armor        │   │
│ │ Front Door rules     │ CF Functions     │ URL map rules      │   │
│ │ CDN Classic          │ (no equivalent)  │ (no equivalent)    │   │
│ └──────────────────────┴──────────────────┴────────────────────┘   │
│                                                                       │
│ ⚡ Azure Front Door = CloudFront + Route 53 routing + WAF          │
│    combined into ONE service. Very powerful!                        │
│                                                                       │
│ Pricing (Front Door):                                                │
│ ├── Standard: ~$35/month base + $0.065/GB egress                  │
│ ├── Premium: ~$330/month base + $0.065/GB egress                  │
│ ├── Requests: $0.01/10,000 (HTTPS)                                │
│ └── WAF (Premium): Included in base!                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Azure Front Door — Components

```
┌─────────────────────────────────────────────────────────────────────┐
│           FRONT DOOR ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Components:                                                          │
│                                                                       │
│ ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐    │
│ │ Endpoint │──►│ Route    │──►│ Origin   │──►│ Origin       │    │
│ │ (domain) │   │ (rules)  │   │ Group    │   │ (backend)    │    │
│ │          │   │          │   │          │   │              │    │
│ │ techcorp │   │ /api/*→  │   │ Health   │   │ App Service  │    │
│ │ .com     │   │ api-grp  │   │ probes   │   │ Storage      │    │
│ │          │   │ /*→      │   │ LB mode  │   │ Custom host  │    │
│ │ FD URL:  │   │ web-grp  │   │ Failover │   │              │    │
│ │ xxx.z01  │   │          │   │          │   │              │    │
│ │ .azurefd │   │ Cache    │   │          │   │              │    │
│ │ .net     │   │ Rules    │   │          │   │              │    │
│ └──────────┘   └──────────┘   └──────────┘   └──────────────┘    │
│                                                                       │
│ PROFILE                                                              │
│ ├── The top-level resource                                         │
│ ├── Choose tier: Standard or Premium                               │
│ ├── Contains endpoints, origin groups, routes, rule sets          │
│ └── One profile can serve multiple domains/applications           │
│                                                                       │
│ ENDPOINT                                                             │
│ ├── The entry point (gets a *.azurefd.net domain)                 │
│ ├── Add custom domains to it                                      │
│ ├── One endpoint can have multiple routes                         │
│ └── Enable/disable without deleting                                │
│                                                                       │
│ ORIGIN GROUP                                                         │
│ ├── Collection of origins (backends)                               │
│ ├── Health probe configuration                                     │
│ ├── Load balancing settings                                       │
│ └── Failover between origins                                      │
│                                                                       │
│ ORIGIN                                                               │
│ ├── The actual backend server                                     │
│ ├── Types: App Service, Storage, Cloud service, Custom            │
│ ├── Priority + weight for routing                                 │
│ ├── Private Link support (Premium only)                           │
│ └── Can be in any region or even non-Azure                        │
│                                                                       │
│ ROUTE                                                                │
│ ├── Maps endpoint + patterns to origin group                      │
│ ├── Path pattern matching (/api/*, /static/*, /*)                 │
│ ├── Accepted protocols (HTTP, HTTPS)                              │
│ ├── Redirect HTTPS enforcement                                    │
│ ├── Caching settings                                               │
│ └── Rule set association                                           │
│                                                                       │
│ RULE SET                                                             │
│ ├── Conditions + actions (like AWS CF Functions/L@E)              │
│ ├── URL rewrite, redirect, header modification                    │
│ ├── Cache expiration override                                      │
│ └── Request/response manipulation                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Front Door Profile

```
Azure Portal → Front Door and CDN profiles → + Create

┌─────────────────────────────────────────────────────────────────┐
│           CREATE FRONT DOOR PROFILE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Compare offerings:                                               │
│   ○ Azure Front Door (classic) — Legacy                        │
│   ○ Explore other offerings                                    │
│   ● Quick create  ○ Custom create                              │
│                                                                   │
│ (Quick Create):                                                  │
│   Subscription:     [tc-production ▼]                           │
│   Resource group:   [rg-networking ▼]                           │
│   Name:             [fd-prod-techcorp]                          │
│                                                                   │
│   Tier:             ○ Standard  ● Premium                      │
│     Standard: CDN + routing                                    │
│     Premium: CDN + routing + WAF + Private Link               │
│                                                                   │
│   Endpoint name:    [techcorp]                                  │
│   → Gets: techcorp.z01.azurefd.net                             │
│                                                                   │
│   Origin type:      [App Service ▼]                            │
│   Origin host name: [tc-prod-app.azurewebsites.net ▼]         │
│                                                                   │
│   Private Link:     ☐ Enable (Premium only)                    │
│   Caching:          ☑ Enable caching                           │
│   WAF policy:       [waf-prod ▼] (Premium only)               │
│                                                                   │
│ [Review + create] → [Create]                                    │
│                                                                   │
│ ⚡ Quick create sets up endpoint + origin group + origin + route│
│    in one step. Custom create lets you configure each part.    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Custom Create (Detailed Walkthrough)

```
Step 1: Create Profile
─────────────────────
  Name: fd-prod-techcorp
  Tier: Premium
  [Create]

Step 2: Add Endpoint
────────────────────
Front Door → Endpoints → + Add endpoint

┌─────────────────────────────────────────────────────────────────┐
│   Endpoint name: [techcorp]                                     │
│   Status: Enabled                                                │
│   → techcorp.z01.azurefd.net                                   │
│   [Add]                                                          │
└─────────────────────────────────────────────────────────────────┘

Step 3: Add Origin Group
────────────────────────
Front Door → Origin groups → + Add

┌─────────────────────────────────────────────────────────────────┐
│           ADD ORIGIN GROUP                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [og-web-app]                                              │
│                                                                   │
│ Health probes:                                                   │
│   Status:      ● Enabled                                       │
│   Protocol:    [HTTPS ▼]                                       │
│   Path:        [/health]                                        │
│   Interval:    [30 seconds ▼]  (30, 60, 90, 120)             │
│   Method:      [HEAD ▼]  (HEAD or GET)                        │
│                                                                   │
│ Load balancing:                                                  │
│   Sample size:                    [4]                           │
│   (How many probes to check)                                   │
│   Successful samples required:    [3]                           │
│   (How many must pass to be healthy)                           │
│   Latency sensitivity (ms):      [0]                           │
│   (0 = pure priority/weight, >0 = consider latency)           │
│                                                                   │
│ Origins: (add in next step)                                     │
│   [+ Add an origin]                                             │
│                                                                   │
│ [Add]                                                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Step 4: Add Origins
──────────────────
Origin group → + Add an origin

┌─────────────────────────────────────────────────────────────────┐
│              ADD ORIGIN                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:          [origin-app-india]                               │
│                                                                   │
│ Origin type:   [App Service ▼]                                 │
│   Options:                                                      │
│   ├── App Service                                              │
│   ├── Storage (Blob)                                           │
│   ├── Storage (Static Website)                                 │
│   ├── Cloud service                                            │
│   ├── Application Gateway                                      │
│   ├── API Management                                           │
│   ├── Public IP address                                        │
│   ├── Custom                                                   │
│   └── Azure Spring Apps                                        │
│                                                                   │
│ Host name:     [tc-prod-app.azurewebsites.net ▼]              │
│ Origin host header: [tc-prod-app.azurewebsites.net]           │
│   (Host header sent to origin — important for multi-tenant    │
│    services like App Service!)                                 │
│                                                                   │
│ HTTP port:     [80]                                             │
│ HTTPS port:    [443]                                            │
│                                                                   │
│ Priority:      [1]   (1 = highest, 1-5)                       │
│   Lower priority = preferred. Same priority = load balanced.  │
│   Origin with priority 2 only used if all priority 1 are down.│
│                                                                   │
│ Weight:        [1000]  (1-1000)                                │
│   Within same priority, traffic split by weight.              │
│   Weight 500 + 500 = 50/50 split.                             │
│                                                                   │
│ Private Link:  ☐ Enable (Premium only)                         │
│   (Connect to origin privately, no public access needed!)     │
│                                                                   │
│ Status:        ● Enabled                                       │
│                                                                   │
│ [Add]                                                            │
│                                                                   │
│ Add second origin for failover:                                 │
│   Name: origin-app-europe                                      │
│   Host: tc-prod-app-eu.azurewebsites.net                      │
│   Priority: 2 (failover)                                       │
│   Weight: 1000                                                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Step 5: Add Route
────────────────
Front Door → Routes → + Add route

┌─────────────────────────────────────────────────────────────────┐
│              ADD ROUTE                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name:          [route-default]                                  │
│ Endpoint:      [techcorp ▼]                                    │
│                                                                   │
│ Domains:                                                         │
│   ☑ techcorp.z01.azurefd.net                                  │
│   ☑ techcorp.com (custom domain, add first)                   │
│   ☑ www.techcorp.com                                           │
│                                                                   │
│ Patterns to match:  [/*]                                       │
│   (path patterns: /*, /api/*, /static/*, etc.)                │
│                                                                   │
│ Accepted protocols: ● HTTP and HTTPS                           │
│                     ○ HTTPS only                                │
│                                                                   │
│ Redirect:          ☑ Redirect all traffic to use HTTPS        │
│                                                                   │
│ Origin group:      [og-web-app ▼]                              │
│ Origin path:       [/] (optional, prefix for origin requests) │
│ Forwarding protocol: [HTTPS only ▼]                            │
│   (Match incoming request / HTTP only / HTTPS only)           │
│                                                                   │
│ Caching:           ☑ Enable caching                            │
│   Query string caching behavior:                                │
│   ├── [Use query string ▼]  (include in cache key)           │
│   ├── Ignore query string  (better cache hit ratio)           │
│   └── Cache every unique URL  (include everything)            │
│                                                                   │
│   Compression:     ☑ Enable compression                       │
│   (Gzip/Brotli for text content types)                        │
│                                                                   │
│ Rule sets:         [rs-security-headers ▼]                     │
│   (associate rule sets for this route)                         │
│                                                                   │
│ [Add]                                                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Rules Engine

```
┌─────────────────────────────────────────────────────────────────────┐
│           RULES ENGINE                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Customize request/response handling at the edge.             │
│       Azure's equivalent to CloudFront Functions / Lambda@Edge.    │
│                                                                       │
│ Rule Set → Rules → Conditions + Actions                            │
│                                                                       │
│ CONDITIONS (when to apply):                                          │
│ ├── Request URL: Path, query string, extension                    │
│ ├── Request headers: Any header value                              │
│ ├── Request method: GET, POST, etc.                                │
│ ├── Request scheme: HTTP or HTTPS                                  │
│ ├── Remote address: Client IP/CIDR                                │
│ ├── Host name: Which domain was requested                         │
│ ├── SSL protocol: TLS version                                     │
│ ├── Socket address: Edge location IP                              │
│ ├── Server port: 80 or 443                                        │
│ ├── Cookie: Specific cookie values                                │
│ └── Post args: Form POST field values                             │
│                                                                       │
│ ACTIONS (what to do):                                                │
│ ├── URL Redirect (301/302/307/308 + rewrite URL)                 │
│ ├── URL Rewrite (change path/query before sending to origin)     │
│ ├── Route configuration override                                  │
│ │   ├── Override origin group                                     │
│ │   ├── Override caching behavior                                 │
│ │   ├── Set cache duration                                        │
│ │   └── Set compression                                           │
│ ├── Modify request header (add/overwrite/delete)                 │
│ └── Modify response header (add/overwrite/delete)                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Creating a Rule Set

```
Front Door → Rule sets → + Add

┌─────────────────────────────────────────────────────────────────┐
│           CREATE RULE SET                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [rs-security-headers]                                     │
│                                                                   │
│ Rule 1: Add security headers                                    │
│   Name: [add-security-headers]                                  │
│   Order: [1]                                                    │
│                                                                   │
│   Conditions:                                                    │
│     [If] [Always] (no condition, apply to all requests)        │
│                                                                   │
│   Actions:                                                       │
│     [Modify response header]                                    │
│     ├── Action: [Append ▼]                                     │
│     │   Header: [X-Content-Type-Options]                       │
│     │   Value:  [nosniff]                                      │
│     ├── Action: [Append ▼]                                     │
│     │   Header: [X-Frame-Options]                              │
│     │   Value:  [SAMEORIGIN]                                   │
│     ├── Action: [Append ▼]                                     │
│     │   Header: [Strict-Transport-Security]                    │
│     │   Value:  [max-age=31536000; includeSubDomains]         │
│     └── Action: [Append ▼]                                     │
│         Header: [X-XSS-Protection]                             │
│         Value:  [1; mode=block]                                │
│                                                                   │
│ Rule 2: Redirect www to apex                                    │
│   Name: [www-to-apex]                                           │
│   Conditions:                                                    │
│     [If] [Host name] [Equal] [www.techcorp.com]               │
│   Actions:                                                       │
│     [URL redirect]                                              │
│     Redirect type: [Moved (301) ▼]                             │
│     Protocol: [HTTPS ▼]                                        │
│     Hostname: [techcorp.com]                                   │
│     Path: [Preserve]                                            │
│                                                                   │
│ Rule 3: SPA routing (rewrite to index.html)                    │
│   Name: [spa-rewrite]                                           │
│   Conditions:                                                    │
│     [If] [URL file extension]                                  │
│     [Not Equal] [css js png jpg svg ico json woff2 woff ttf]  │
│   Actions:                                                       │
│     [URL rewrite]                                               │
│     Source pattern: [/]                                         │
│     Destination: [/index.html]                                 │
│     Preserve unmatched path: No                                │
│                                                                   │
│ Rule 4: Cache override for static assets                        │
│   Name: [cache-static-assets]                                   │
│   Conditions:                                                    │
│     [If] [URL path] [Begins with] [/static/]                 │
│   Actions:                                                       │
│     [Route configuration override]                             │
│     Caching: Override                                           │
│     Cache duration: 365 days                                   │
│                                                                   │
│ [Save]                                                           │
│                                                                   │
│ Then associate rule set to a route:                             │
│ Routes → route-default → Rule sets → [rs-security-headers]    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Custom Domains & SSL

```
┌─────────────────────────────────────────────────────────────────────┐
│           CUSTOM DOMAINS & SSL                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Step 1: Add custom domain                                            │
│ Front Door → Domains → + Add                                       │
│                                                                       │
│   Domain type:                                                       │
│   ├── Azure pre-validated domain (from Azure DNS)                 │
│   │   ⚡ Auto-validates if domain in Azure DNS!                   │
│   └── Non-Azure validated domain                                   │
│       Requires DNS TXT record for verification                    │
│                                                                       │
│   DNS management:                                                    │
│   ├── Azure managed DNS (zone in Azure DNS)                       │
│   └── All other DNS services (manual CNAME)                       │
│                                                                       │
│   Domain name: [techcorp.com]                                       │
│                                                                       │
│ Step 2: SSL certificate                                              │
│   Certificate type:                                                  │
│   ├── Front Door managed ← recommended (free, auto-renewed!)     │
│   │   Uses DigiCert, auto-provisioned                             │
│   └── Bring your own certificate                                   │
│       From Azure Key Vault                                         │
│                                                                       │
│   Minimum TLS version: [TLS 1.2 ▼]                                │
│                                                                       │
│ Step 3: DNS configuration                                            │
│   For subdomain (www.techcorp.com):                                │
│     CNAME → techcorp.z01.azurefd.net                               │
│                                                                       │
│   For apex (techcorp.com):                                          │
│     Option 1: Azure DNS Alias → Front Door endpoint               │
│     Option 2: CNAME flattening (if non-Azure DNS supports it)    │
│                                                                       │
│ Step 4: Associate domain with route                                  │
│   Routes → route-default → Domains → ☑ techcorp.com              │
│                                                                       │
│ ⚠️ Unlike AWS (cert must be in us-east-1), Azure Front Door       │
│    manages certs globally. No region restriction!                  │
│                                                                       │
│ ⚡ Front Door managed certs are FREE and auto-renew!               │
│    AWS ACM is also free. GCP Google-managed is also free.          │
│    All three clouds offer free managed SSL.                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Caching

```
┌─────────────────────────────────────────────────────────────────────┐
│           CACHING                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Caching is configured per route and can be overridden by rules.    │
│                                                                       │
│ Query string caching behavior:                                       │
│ ├── Use query string: /page?v=1 and /page?v=2 cached separately  │
│ ├── Ignore query string: Both → same cache entry                  │
│ │   (higher hit ratio, use for content that doesn't vary by QS)  │
│ └── Cache every unique URL: Include all parameters                │
│                                                                       │
│ Default caching behavior:                                            │
│ ├── Honor origin Cache-Control headers                             │
│ ├── If origin sends no cache header → not cached by default       │
│ ├── Override with rules engine to force caching                   │
│ └── Compression auto-enabled for text types                       │
│                                                                       │
│ Cache purge (invalidation):                                          │
│ Front Door → Purge                                                  │
│   Content type:                                                      │
│   ├── Single path: /images/logo.png                               │
│   ├── Wildcard: /static/*                                         │
│   └── Root: / (purge all)                                         │
│   Domain: techcorp.com                                              │
│                                                                       │
│ CLI:                                                                 │
│ az afd endpoint purge-content \                                      │
│   --resource-group rg-networking \                                    │
│   --profile-name fd-prod-techcorp \                                   │
│   --endpoint-name techcorp \                                          │
│   --content-paths "/static/*" "/index.html"                         │
│                                                                       │
│ ⚡ Purge is free, takes ~2 minutes to propagate.                    │
│    Same as AWS/GCP, prefer cache busting over purging.             │
│                                                                       │
│ Cache by rules engine:                                               │
│ ├── Override cache duration per path pattern                      │
│ ├── /static/* → cache 365 days                                   │
│ ├── /api/* → caching disabled                                    │
│ ├── /images/* → cache 30 days                                    │
│ └── Much more flexible than basic route-level caching             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: WAF Integration (Premium)

```
┌─────────────────────────────────────────────────────────────────────┐
│           WAF (FRONT DOOR PREMIUM)                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Web Application Firewall attached to Front Door.             │
│       Same WAF policy as Azure Application Gateway WAF.            │
│                                                                       │
│ Features:                                                            │
│ ├── Managed rule sets (OWASP 3.2, Bot Manager, etc.)             │
│ ├── Custom rules (IP, geo, rate limiting, header matching)        │
│ ├── Rate limiting per IP or per geo                               │
│ ├── Bot protection (bad bot blocking)                             │
│ ├── Geo-filtering (allow/block countries)                         │
│ └── Anomaly scoring mode                                          │
│                                                                       │
│ Create WAF policy:                                                   │
│ Azure Portal → WAF policies → + Create                            │
│   Policy for: [Azure Front Door ▼]                                │
│   Tier: [Premium ▼]                                                │
│   Name: [waf-prod-fd]                                              │
│                                                                       │
│ Managed rules:                                                       │
│   ☑ DefaultRuleSet_2.1 (OWASP protection)                        │
│   ☑ BotProtection (bad bots)                                      │
│                                                                       │
│ Custom rules:                                                        │
│   Rate limit: 100 requests/minute per IP                          │
│   Block IPs: Specific CIDR ranges                                 │
│   Geo-block: Block traffic from specific countries                │
│                                                                       │
│ Attach to Front Door:                                                │
│ Front Door → Security policies → + Add                            │
│   Name: [sp-prod]                                                  │
│   WAF policy: [waf-prod-fd ▼]                                    │
│   Domains: ☑ techcorp.com                                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Private Link Origins (Premium)

```
┌─────────────────────────────────────────────────────────────────────┐
│           PRIVATE LINK ORIGINS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Connect Front Door to origins privately (no public internet).│
│       Front Door Premium exclusive feature.                        │
│                                                                       │
│ Without Private Link:                                                │
│ Front Door ──internet──► App Service (public endpoint)             │
│ ⚠️ Origin must be publicly accessible                               │
│                                                                       │
│ With Private Link:                                                   │
│ Front Door ──Private Link──► App Service (private only!)          │
│ ⚡ Origin can disable public access entirely                       │
│                                                                       │
│ Supported origins:                                                   │
│ ├── App Service / Functions                                        │
│ ├── Azure Storage (Blob)                                           │
│ ├── Azure Application Gateway (v2)                                │
│ ├── Azure API Management                                          │
│ ├── Internal Load Balancer                                        │
│ └── Custom (via Private Link Service)                             │
│                                                                       │
│ Setup:                                                               │
│ When adding origin, check "Enable Private Link":                   │
│   Private Link: ☑ Enable                                          │
│   Region: [Central India ▼]                                       │
│   Target: [App Service ▼]                                         │
│   Resource: [tc-prod-app ▼]                                       │
│   Request message: [FD prod origin access]                        │
│                                                                       │
│ After creation → origin gets a pending Private Endpoint approval. │
│ Go to origin resource → Networking → Approve the PE request.      │
│                                                                       │
│ ⚠️ No equivalent in AWS CloudFront or GCP Cloud CDN!               │
│    CloudFront can access ALB only if ALB is public.               │
│    Front Door Premium + Private Link = truly private origin.      │
│    This is a significant security advantage.                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Azure CDN Classic (Brief Reference)

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE CDN CLASSIC                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚠️ Legacy service. Use Front Door for new projects.                 │
│                                                                       │
│ Tiers:                                                               │
│ ├── Azure CDN Standard from Microsoft                              │
│ ├── Azure CDN Standard from Akamai (being retired)                │
│ ├── Azure CDN Standard from Verizon                                │
│ └── Azure CDN Premium from Verizon                                 │
│                                                                       │
│ Structure:                                                           │
│ CDN Profile → CDN Endpoints → Origins → Caching Rules             │
│                                                                       │
│ When to use classic:                                                 │
│ ├── Existing deployments (not yet migrated)                        │
│ ├── Specific Verizon/Akamai edge feature needed                   │
│ └── Otherwise, migrate to Front Door Standard/Premium             │
│                                                                       │
│ Migration path:                                                      │
│ Azure CDN → Azure Front Door Standard or Premium                  │
│ Microsoft provides migration tools in the portal.                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Bicep Example

```bicep
// Front Door profile (Premium with WAF)
resource frontDoor 'Microsoft.Cdn/profiles@2024-02-01' = {
  name: 'fd-prod-techcorp'
  location: 'global'
  sku: {
    name: 'Premium_AzureFrontDoor'
  }
}

// Endpoint
resource endpoint 'Microsoft.Cdn/profiles/afdEndpoints@2024-02-01' = {
  parent: frontDoor
  name: 'techcorp'
  location: 'global'
  properties: {
    enabledState: 'Enabled'
  }
}

// Origin group (web app)
resource originGroupWeb 'Microsoft.Cdn/profiles/originGroups@2024-02-01' = {
  parent: frontDoor
  name: 'og-web-app'
  properties: {
    loadBalancingSettings: {
      sampleSize: 4
      successfulSamplesRequired: 3
      additionalLatencyInMilliseconds: 0
    }
    healthProbeSettings: {
      probePath: '/health'
      probeRequestType: 'HEAD'
      probeProtocol: 'Https'
      probeIntervalInSeconds: 30
    }
  }
}

// Origin (App Service in India)
resource originIndia 'Microsoft.Cdn/profiles/originGroups/origins@2024-02-01' = {
  parent: originGroupWeb
  name: 'origin-app-india'
  properties: {
    hostName: 'tc-prod-app.azurewebsites.net'
    originHostHeader: 'tc-prod-app.azurewebsites.net'
    httpPort: 80
    httpsPort: 443
    priority: 1
    weight: 1000
    enabledState: 'Enabled'
  }
}

// Origin (App Service in Europe — failover)
resource originEurope 'Microsoft.Cdn/profiles/originGroups/origins@2024-02-01' = {
  parent: originGroupWeb
  name: 'origin-app-europe'
  properties: {
    hostName: 'tc-prod-app-eu.azurewebsites.net'
    originHostHeader: 'tc-prod-app-eu.azurewebsites.net'
    httpPort: 80
    httpsPort: 443
    priority: 2
    weight: 1000
    enabledState: 'Enabled'
  }
}

// Origin group (static storage)
resource originGroupStatic 'Microsoft.Cdn/profiles/originGroups@2024-02-01' = {
  parent: frontDoor
  name: 'og-static'
  properties: {
    loadBalancingSettings: {
      sampleSize: 4
      successfulSamplesRequired: 3
    }
  }
}

// Origin (Storage Account)
resource originStorage 'Microsoft.Cdn/profiles/originGroups/origins@2024-02-01' = {
  parent: originGroupStatic
  name: 'origin-storage'
  properties: {
    hostName: 'tcprodstatic.blob.core.windows.net'
    originHostHeader: 'tcprodstatic.blob.core.windows.net'
    httpPort: 80
    httpsPort: 443
    priority: 1
    weight: 1000
    enabledState: 'Enabled'
  }
}

// Route (default — web app)
resource routeDefault 'Microsoft.Cdn/profiles/afdEndpoints/routes@2024-02-01' = {
  parent: endpoint
  name: 'route-default'
  properties: {
    originGroup: {
      id: originGroupWeb.id
    }
    patternsToMatch: ['/*']
    supportedProtocols: ['Http', 'Https']
    httpsRedirect: 'Enabled'
    forwardingProtocol: 'HttpsOnly'
    cacheConfiguration: {
      queryStringCachingBehavior: 'UseQueryString'
      compressionSettings: {
        isCompressionEnabled: true
        contentTypesToCompress: [
          'text/html'
          'text/css'
          'application/javascript'
          'application/json'
          'image/svg+xml'
        ]
      }
    }
    linkToDefaultDomain: 'Enabled'
  }
}

// Route (static assets)
resource routeStatic 'Microsoft.Cdn/profiles/afdEndpoints/routes@2024-02-01' = {
  parent: endpoint
  name: 'route-static'
  properties: {
    originGroup: {
      id: originGroupStatic.id
    }
    patternsToMatch: ['/static/*']
    supportedProtocols: ['Https']
    httpsRedirect: 'Enabled'
    forwardingProtocol: 'HttpsOnly'
    cacheConfiguration: {
      queryStringCachingBehavior: 'IgnoreQueryString'
      compressionSettings: {
        isCompressionEnabled: true
        contentTypesToCompress: [
          'text/css'
          'application/javascript'
          'image/svg+xml'
        ]
      }
    }
    linkToDefaultDomain: 'Enabled'
  }
}

// Rule set (security headers)
resource ruleSet 'Microsoft.Cdn/profiles/ruleSets@2024-02-01' = {
  parent: frontDoor
  name: 'rsSecurityHeaders'
}

resource ruleSecHeaders 'Microsoft.Cdn/profiles/ruleSets/rules@2024-02-01' = {
  parent: ruleSet
  name: 'addSecurityHeaders'
  properties: {
    order: 1
    actions: [
      {
        name: 'ModifyResponseHeader'
        parameters: {
          typeName: 'DeliveryRuleHeaderActionParameters'
          headerAction: 'Append'
          headerName: 'X-Content-Type-Options'
          value: 'nosniff'
        }
      }
      {
        name: 'ModifyResponseHeader'
        parameters: {
          typeName: 'DeliveryRuleHeaderActionParameters'
          headerAction: 'Append'
          headerName: 'X-Frame-Options'
          value: 'SAMEORIGIN'
        }
      }
      {
        name: 'ModifyResponseHeader'
        parameters: {
          typeName: 'DeliveryRuleHeaderActionParameters'
          headerAction: 'Append'
          headerName: 'Strict-Transport-Security'
          value: 'max-age=31536000; includeSubDomains'
        }
      }
    ]
  }
}

// Custom domain
resource customDomain 'Microsoft.Cdn/profiles/customDomains@2024-02-01' = {
  parent: frontDoor
  name: 'techcorp-com'
  properties: {
    hostName: 'techcorp.com'
    tlsSettings: {
      certificateType: 'ManagedCertificate'
      minimumTlsVersion: 'TLS12'
    }
  }
}

// DNS Alias record (Azure DNS)
resource dnsAlias 'Microsoft.Network/dnsZones/A@2023-07-01-preview' = {
  parent: dnsZone
  name: '@'
  properties: {
    TTL: 60
    targetResource: {
      id: endpoint.id
    }
  }
}

// WAF policy
resource wafPolicy 'Microsoft.Network/FrontDoorWebApplicationFirewallPolicies@2024-02-01' = {
  name: 'wafProdFd'
  location: 'global'
  sku: {
    name: 'Premium_AzureFrontDoor'
  }
  properties: {
    policySettings: {
      enabledState: 'Enabled'
      mode: 'Prevention'
    }
    managedRules: {
      managedRuleSets: [
        {
          ruleSetType: 'Microsoft_DefaultRuleSet'
          ruleSetVersion: '2.1'
        }
        {
          ruleSetType: 'Microsoft_BotManagerRuleSet'
          ruleSetVersion: '1.1'
        }
      ]
    }
    customRules: {
      rules: [
        {
          name: 'RateLimitRule'
          priority: 1
          ruleType: 'RateLimitRule'
          rateLimitDurationInMinutes: 1
          rateLimitThreshold: 100
          matchConditions: [
            {
              matchVariable: 'RemoteAddr'
              operator: 'IPMatch'
              matchValue: ['0.0.0.0/0']
            }
          ]
          action: 'Block'
        }
      ]
    }
  }
}

// Security policy (attach WAF to domain)
resource securityPolicy 'Microsoft.Cdn/profiles/securityPolicies@2024-02-01' = {
  parent: frontDoor
  name: 'sp-prod'
  properties: {
    parameters: {
      type: 'WebApplicationFirewall'
      wafPolicy: {
        id: wafPolicy.id
      }
      associations: [
        {
          domains: [
            { id: customDomain.id }
            { id: endpoint.id }
          ]
          patternsToMatch: ['/*']
        }
      ]
    }
  }
}
```

---

## Part 10: Real-World Patterns

### Startup

```
Service: Azure Front Door Standard
Tier: Standard ($35/month base)

Setup:
├── 1 endpoint: techcorp.z01.azurefd.net
├── 1 origin group: App Service
├── 1 origin: tc-prod-app.azurewebsites.net
├── 1 route: /* → App Service (caching ON)
├── Custom domain: techcorp.com (managed cert)
└── Rules: HTTPS redirect, security headers

No WAF (Standard tier).
No failover (single region).
Cost: ~$40-60/month
```

### Mid-Size

```
Service: Azure Front Door Premium
Tier: Premium ($330/month base)

Setup:
├── 1 endpoint
├── 2 origin groups:
│   ├── og-web: App Service India (pri 1) + Europe (pri 2)
│   └── og-static: Storage Account
├── 3 routes:
│   ├── /api/* → og-web (no caching)
│   ├── /static/* → og-static (cache 365 days)
│   └── /* → og-web (cache 1 hour)
├── Rule sets:
│   ├── Security headers
│   ├── www → apex redirect
│   ├── SPA URL rewrite
│   └── Cache overrides per path
├── WAF: OWASP rules + rate limiting
├── Custom domains: techcorp.com, www, api, admin
└── Managed certs (free, auto-renewed)

Private Link: App Service origin (private access only)
Health probes: HTTPS /health every 30s
Cost: ~$400-600/month
```

### Enterprise

```
Service: Azure Front Door Premium
Multiple profiles per product/BU

Architecture:
├── Main site profile:
│   ├── Multi-region origin groups (India, Europe, US)
│   ├── Priority-based failover (1→2→3)
│   ├── Weight-based canary within same priority
│   └── Private Link to all origins
├── API profile:
│   ├── API Management as origin
│   ├── No caching (or selective per endpoint)
│   └── WAF with API-specific rules
├── Static/CDN profile:
│   ├── Storage origins (multi-region)
│   ├── Maximum cache TTL
│   └── Compression for all text types

WAF (per profile):
├── OWASP managed rules
├── Bot Manager
├── Custom rate limiting per endpoint
├── Geo-blocking for compliance
├── IP allowlisting for admin areas
└── Custom rules for business logic

Rules engine:
├── A/B testing (route % to different origins)
├── Header-based routing (mobile vs desktop)
├── Cache strategy per content type
├── Custom error pages
└── Request/response header manipulation

Monitoring:
├── Front Door diagnostics → Log Analytics
├── WAF logs → Log Analytics
├── Dashboard: Cache hit %, latency, errors
├── Alerts: Error rate > 1%, WAF blocks spike
└── Cost anomaly detection

Cost: ~$1,000-5,000/month
```

---

## Quick Reference

| Feature | Details |
|---------|---------|
| Tiers | Standard ($35/mo) / Premium ($330/mo) |
| Edge locations | 180+ PoPs globally |
| Custom domains | Unlimited, managed certs (free!) |
| WAF | Premium only (included in base) |
| Private Link origins | Premium only |
| Rules engine | Conditions + actions (URL rewrite, redirect, headers) |
| Caching | Per-route + rules engine overrides |
| Cache purge | Path-based, wildcard, free |
| Health probes | Configurable per origin group |
| Failover | Priority-based within origin group |
| Load balancing | Weight-based within same priority |
| Compression | Gzip/Brotli (automatic) |
| HTTP versions | HTTP/2 (HTTP/3 preview) |
| SLA | 99.99% |

---

## What's Next?

In the next chapter, we'll cover Azure Load Balancer & Application Gateway — Azure's load balancing services.

→ Next: [Chapter 11: Azure Load Balancer](11-load-balancer.md)

---

*Last Updated: May 2026*
