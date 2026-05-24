# Chapter 12: Cloud Armor (GCP)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cloud Armor Fundamentals](#part-1-cloud-armor-fundamentals)
- [Part 2: Security Policy Types](#part-2-security-policy-types)
- [Part 3: Creating a Security Policy (Full Portal Walkthrough)](#part-3-creating-a-security-policy-full-portal-walkthrough)
- [Part 4: Adaptive Protection](#part-4-adaptive-protection)
- [Part 5: Bot Management](#part-5-bot-management)
- [Part 6: Named IP Lists](#part-6-named-ip-lists)
- [Part 7: CEL Expression Reference](#part-7-cel-expression-reference)
- [Part 8: Logging & Monitoring](#part-8-logging--monitoring)
- [Part 9: Terraform Example](#part-9-terraform-example)
- [Part 10: gcloud CLI Reference](#part-10-gcloud-cli-reference)
- [Part 11: Real-World Patterns](#part-11-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud Armor is GCP's WAF (Web Application Firewall) and DDoS protection service. It sits in front of your load balancers and protects your applications from web attacks, DDoS, and unwanted traffic. It's attached to backend services on External Application Load Balancers.

```
What you'll learn:
├── What is Cloud Armor & why use it
├── Cloud Armor vs AWS WAF vs Azure WAF
├── Architecture & how it fits with Load Balancing
├── Security Policies
│   ├── Backend security policy (most common)
│   ├── Edge security policy
│   └── Network edge security policy
├── Rules
│   ├── IP allowlist/denylist
│   ├── Geo-based blocking
│   ├── Custom expression rules (CEL)
│   ├── Pre-configured WAF rules (OWASP)
│   └── Rate limiting
├── Adaptive Protection (ML-based DDoS)
├── Bot Management
├── Named IP Lists (CDN providers, Tor, etc.)
├── Creating policies (full walkthrough)
├── Terraform examples
└── Real-world patterns
```

---

## Part 1: Cloud Armor Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUD ARMOR FUNDAMENTALS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: WAF + DDoS protection for GCP load balancers.               │
│                                                                       │
│ Where it sits:                                                       │
│ Client → Google Edge → Cloud Armor → LB → Backend Service → VMs │
│                                                                       │
│ Cloud Armor evaluates EVERY request at Google's edge before       │
│ it reaches your backend. Blocked requests never touch your app.   │
│                                                                       │
│ Protects against:                                                    │
│ ├── DDoS attacks (volumetric, protocol, application layer)       │
│ ├── SQL injection (SQLi)                                          │
│ ├── Cross-site scripting (XSS)                                   │
│ ├── Remote code execution (RCE)                                   │
│ ├── Local/remote file inclusion (LFI/RFI)                        │
│ ├── Bot traffic                                                    │
│ ├── Geographic restrictions                                       │
│ └── Rate-based attacks (brute force, credential stuffing)        │
│                                                                       │
│ Supported load balancers:                                            │
│ ├── External Application LB (Global HTTP/S) ✅ (most common)    │
│ ├── External Proxy Network LB (TCP/SSL) ✅                       │
│ ├── External Passthrough Network LB ✅ (network edge policy)    │
│ └── Internal LBs ❌ (not supported — internal traffic is trusted)│
│                                                                       │
│ Key difference from AWS:                                             │
│ ├── AWS: WAF is a SEPARATE service, attached to ALB/CloudFront  │
│ ├── Azure: WAF is BUILT INTO Application Gateway (WAF SKU)      │
│ ├── GCP: Cloud Armor is a SEPARATE service, attached to backend │
│ │         services on the load balancer                            │
│ └── All three offer OWASP rule sets, but pricing differs         │
│                                                                       │
│ ┌──────────────────────┬──────────────┬─────────────┬────────────┐│
│ │ Feature              │ Cloud Armor  │ AWS WAF     │ Azure WAF  ││
│ ├──────────────────────┼──────────────┼─────────────┼────────────┤│
│ │ Pricing model        │ Per policy + │ Per rule +  │ Built into ││
│ │                      │ per request  │ per request │ App GW SKU ││
│ │ OWASP rules          │ Pre-config'd │ Managed     │ Built-in   ││
│ │ Adaptive protection  │ Yes (ML) ✅  │ No ❌      │ No ❌     ││
│ │ Bot management       │ Yes          │ Bot Control │ Bot protect││
│ │ Rate limiting        │ Yes          │ Yes         │ Custom rule││
│ │ Named IP lists       │ Yes ✅       │ IP sets    │ No         ││
│ │ DDoS protection      │ Included     │ Shield     │ DDoS Prot. ││
│ │ Edge enforcement     │ Yes ✅       │ CloudFront │ Front Door ││
│ │ Custom rules (expr)  │ CEL          │ JSON       │ Custom     ││
│ │ Attach to            │ Backend svc  │ ALB, CF,   │ App GW,    ││
│ │                      │              │ API GW     │ Front Door ││
│ └──────────────────────┴──────────────┴─────────────┴────────────┘│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Security Policy Types

```
┌─────────────────────────────────────────────────────────────────────┐
│           SECURITY POLICY TYPES                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. BACKEND SECURITY POLICY (most common):                            │
│    ├── Attached to: Backend services on External App LB            │
│    ├── Evaluates: HTTP/HTTPS requests (Layer 7)                    │
│    ├── Rules: IP, geo, WAF, rate limiting, custom expressions    │
│    └── Where: At Google's edge (before reaching your backend)    │
│                                                                       │
│ 2. EDGE SECURITY POLICY:                                             │
│    ├── Attached to: Backend services with Cloud CDN enabled        │
│    ├── Evaluates: BEFORE Cloud CDN cache lookup                    │
│    ├── Rules: IP-based and geo-based ONLY                         │
│    ├── No WAF rules, no rate limiting at edge                     │
│    └── Use: Block entire countries/IPs before CDN serves content  │
│                                                                       │
│ 3. NETWORK EDGE SECURITY POLICY:                                     │
│    ├── Attached to: External Passthrough Network LB                │
│    ├── Evaluates: Layer 3/4 (network level)                       │
│    ├── Rules: Byte offset matching, protocol filtering            │
│    └── Use: DDoS protection for non-HTTP traffic                  │
│                                                                       │
│ ⚡ 95% of the time you'll use Backend Security Policy.              │
│    Edge policy is a specialized optimization for CDN.              │
│    Network edge is for non-HTTP DDoS protection.                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Creating a Security Policy (Full Portal Walkthrough)

```
Console → Network Security → Cloud Armor policies → Create policy

┌─────────────────────────────────────────────────────────────────┐
│           CREATE SECURITY POLICY                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Configure policy ──                                          │
│ Name:        [policy-prod]                                      │
│ Description: [Production WAF policy for web applications]      │
│                                                                   │
│ Policy type:                                                    │
│   ● Backend security policy                                    │
│   ○ Edge security policy                                       │
│   ○ Network edge security policy                               │
│                                                                   │
│ Default rule action:                                            │
│   ● Allow  (allow all traffic by default, deny specific)       │
│   ○ Deny   (deny all traffic by default, allow specific)       │
│                                                                   │
│   ⚠️ Start with "Allow" + deny rules.                            │
│      Use "Deny" default only for APIs with known clients.      │
│                                                                   │
│ ── Adaptive Protection ──                                       │
│ ☑ Enable adaptive protection (recommended)                    │
│                                                                   │
│   ⚡ ML-based anomaly detection!                                 │
│   Learns your normal traffic patterns,                         │
│   automatically detects + alerts on DDoS attacks,             │
│   and suggests rules to mitigate.                              │
│   No AWS/Azure equivalent!                                     │
│                                                                   │
│ [Next: Add rules]                                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Adding Rules

```
┌─────────────────────────────────────────────────────────────────────┐
│           RULE TYPES & EXAMPLES                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Rules evaluated by PRIORITY (lower number = higher priority).      │
│ First matching rule wins. Default rule is always last (2147483647).│
│                                                                       │
│ ═══════════════════════════════════════════════════════════════     │
│ RULE 1: Block specific countries                                    │
│ ═══════════════════════════════════════════════════════════════     │
│                                                                       │
│ Description: [Block traffic from high-risk countries]              │
│ Priority:    [1000]                                                 │
│ Match:                                                               │
│   Condition: [Advanced]                                             │
│   Expression:                                                        │
│   origin.region_code == 'CN' ||                                    │
│   origin.region_code == 'RU' ||                                    │
│   origin.region_code == 'KP'                                       │
│                                                                       │
│ Action: [Deny ▼]                                                    │
│ Deny status: [403 ▼] (Forbidden)                                   │
│ Options: 403, 404, 502                                              │
│                                                                       │
│ ═══════════════════════════════════════════════════════════════     │
│ RULE 2: Allow specific IPs (office, VPN)                            │
│ ═══════════════════════════════════════════════════════════════     │
│                                                                       │
│ Description: [Allow office IP ranges]                               │
│ Priority:    [500] (higher priority than block rules)              │
│ Match:                                                               │
│   Condition: [IP address ranges]                                   │
│   IP ranges:                                                        │
│   203.0.113.0/24                                                    │
│   198.51.100.0/24                                                   │
│                                                                       │
│ Action: [Allow ▼]                                                   │
│                                                                       │
│ ⚡ Office IPs are allowed even if their country is blocked.         │
│    Priority 500 < 1000, so office rule evaluates first.           │
│                                                                       │
│ ═══════════════════════════════════════════════════════════════     │
│ RULE 3: OWASP Top 10 WAF rules (pre-configured)                    │
│ ═══════════════════════════════════════════════════════════════     │
│                                                                       │
│ Description: [Block SQL injection attacks]                          │
│ Priority:    [2000]                                                 │
│ Match:                                                               │
│   Condition: [Advanced]                                             │
│   Expression:                                                        │
│   evaluatePreconfiguredExpr('sqli-v33-stable')                    │
│                                                                       │
│ Action: [Deny ▼]                                                    │
│ Deny status: [403]                                                  │
│                                                                       │
│ Available pre-configured expressions:                                │
│ ┌───────────────────────────────────┬───────────────────────────┐  │
│ │ Expression                       │ What it blocks             │  │
│ ├───────────────────────────────────┼───────────────────────────┤  │
│ │ sqli-v33-stable                  │ SQL injection             │  │
│ │ xss-v33-stable                   │ Cross-site scripting      │  │
│ │ lfi-v33-stable                   │ Local file inclusion      │  │
│ │ rfi-v33-stable                   │ Remote file inclusion     │  │
│ │ rce-v33-stable                   │ Remote code execution     │  │
│ │ methodenforcement-v33-stable     │ Method enforcement        │  │
│ │ scannerdetection-v33-stable      │ Scanner detection         │  │
│ │ protocolattack-v33-stable        │ Protocol attacks          │  │
│ │ sessionfixation-v33-stable       │ Session fixation          │  │
│ │ java-v33-stable                  │ Java-specific attacks     │  │
│ │ nodejs-v33-stable                │ Node.js attacks           │  │
│ │ php-v33-stable                   │ PHP-specific attacks      │  │
│ │ cve-canary                       │ Known CVE exploits        │  │
│ │ json-sqli-canary                 │ JSON-based SQL injection  │  │
│ └───────────────────────────────────┴───────────────────────────┘  │
│                                                                       │
│ ⚡ Use 'stable' expressions for production.                         │
│    'canary' expressions are newer, less tested.                    │
│                                                                       │
│ ═══════════════════════════════════════════════════════════════     │
│ RULE 4: XSS protection                                              │
│ ═══════════════════════════════════════════════════════════════     │
│                                                                       │
│ Description: [Block XSS attacks]                                    │
│ Priority:    [2100]                                                 │
│ Expression:  evaluatePreconfiguredExpr('xss-v33-stable')          │
│ Action: Deny 403                                                    │
│                                                                       │
│ ═══════════════════════════════════════════════════════════════     │
│ RULE 5: Rate limiting                                                │
│ ═══════════════════════════════════════════════════════════════     │
│                                                                       │
│ Description: [Rate limit API endpoints]                              │
│ Priority:    [3000]                                                 │
│ Match:                                                               │
│   Expression:                                                        │
│   request.path.matches('/api/.*')                                  │
│                                                                       │
│ Action: [Rate-based ban ▼]                                          │
│ Rate limit options:                                                  │
│   Conform action: [Allow ▼]                                        │
│   Exceed action: [Deny(429) ▼]                                     │
│   Rate limit threshold:                                             │
│     Count: [100] requests                                           │
│     Interval: [60] seconds                                          │
│     (100 requests per minute per key)                              │
│                                                                       │
│   Enforce on key:                                                    │
│     ● IP  (rate limit per source IP)                               │
│     ○ ALL  (aggregate — total requests)                            │
│     ○ HTTP Header                                                  │
│     ○ XFF IP (real IP behind proxy)                                │
│     ○ HTTP Cookie                                                  │
│     ○ Region code                                                  │
│                                                                       │
│   Ban duration: [300] seconds (5 min ban after exceeding)         │
│                                                                       │
│   ⚡ Rate limit options:                                             │
│   ├── Throttle: Just deny excess (no ban)                         │
│   └── Rate-based ban: Ban the key for N seconds after exceeding  │
│                                                                       │
│ ═══════════════════════════════════════════════════════════════     │
│ RULE 6: Custom expression (header-based)                            │
│ ═══════════════════════════════════════════════════════════════     │
│                                                                       │
│ Description: [Block requests without API key]                       │
│ Priority:    [4000]                                                 │
│ Expression:                                                          │
│   request.path.matches('/api/.*') &&                               │
│   !has(request.headers['x-api-key'])                               │
│ Action: Deny 403                                                    │
│                                                                       │
│ ═══════════════════════════════════════════════════════════════     │
│ RULE 7: Block by User-Agent (scanners/bots)                         │
│ ═══════════════════════════════════════════════════════════════     │
│                                                                       │
│ Description: [Block known vulnerability scanners]                   │
│ Priority:    [4500]                                                 │
│ Expression:                                                          │
│   request.headers['user-agent'].matches(                           │
│     '(?i).*(sqlmap|nikto|nmap|masscan|zgrab).*'                   │
│   )                                                                 │
│ Action: Deny 403                                                    │
│                                                                       │
│ ═══════════════════════════════════════════════════════════════     │
│ DEFAULT RULE (always present):                                       │
│ Priority: 2147483647                                                │
│ Match: * (all traffic)                                               │
│ Action: Allow (or Deny if you chose deny-by-default)               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Attaching Policy to Backend Service

```
┌─────────────────────────────────────────────────────────────────────┐
│           ATTACH POLICY TO BACKEND SERVICE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Option 1: During policy creation                                    │
│   Apply policy to targets:                                          │
│   [+ Add target]                                                    │
│   Target type: [Backend service ▼]                                 │
│   Target: [bs-web-prod ▼]                                          │
│                                                                       │
│ Option 2: From the backend service                                  │
│   Load Balancing → Backend services → bs-web-prod → Edit         │
│   Security: [policy-prod ▼]                                       │
│                                                                       │
│ Option 3: gcloud                                                    │
│   gcloud compute backend-services update bs-web-prod \              │
│     --security-policy=policy-prod \                                  │
│     --global                                                         │
│                                                                       │
│ ⚠️ One policy per backend service.                                   │
│    But one policy can be attached to MULTIPLE backend services.    │
│    Common pattern: One "strict" policy for APIs,                   │
│    one "relaxed" policy for static content.                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Adaptive Protection

```
┌─────────────────────────────────────────────────────────────────────┐
│           ADAPTIVE PROTECTION (ML-BASED)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚡ UNIQUE to GCP. No AWS or Azure equivalent!                        │
│                                                                       │
│ What: Machine learning model that learns your normal traffic       │
│ patterns and automatically detects anomalies (potential attacks).  │
│                                                                       │
│ How it works:                                                        │
│ 1. Cloud Armor observes your traffic for ~1 hour (baseline)       │
│ 2. ML model learns: normal volume, sources, patterns, URIs       │
│ 3. When traffic deviates significantly → ALERT                    │
│ 4. Cloud Armor suggests specific rules to block the attack        │
│ 5. You can auto-deploy those rules or review first                │
│                                                                       │
│ Enable:                                                              │
│ Security policy → Edit → Adaptive Protection: ☑ Enable           │
│                                                                       │
│ Auto-deploy:                                                        │
│ ☑ Auto-deploy rules when confidence > 80%                         │
│                                                                       │
│ Auto-deploy settings:                                                │
│ ├── Confidence threshold: [0.8] (80%)                             │
│ ├── Impacted baseline threshold: [0.01] (1% of baseline traffic) │
│ ├── Expiration set: [7200] seconds (2 hours — auto-remove rule)  │
│ └── Load threshold: [0.1] (attack must be 10%+ of capacity)     │
│                                                                       │
│ Alert types:                                                         │
│ ├── "Potential attack detected"                                   │
│ ├── Attack signature (source IPs, regions, patterns)              │
│ ├── Suggested rule (auto-generated CEL expression)                │
│ └── Confidence score (0-1)                                         │
│                                                                       │
│ Example alert:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │ ⚠️ Adaptive Protection Alert                                   │   │
│ │                                                              │   │
│ │ Attack type: HTTP flood                                      │   │
│ │ Confidence: 92%                                              │   │
│ │ Start time: 2026-05-16 14:23:00 UTC                         │   │
│ │ Impact: 10x normal request rate to /api/login               │   │
│ │ Top sources: 45.33.x.x/24, 185.220.x.x/16                 │   │
│ │                                                              │   │
│ │ Suggested rule:                                              │   │
│ │ evaluatePreconfiguredExpr(                                   │   │
│ │   'adaptive-protection-auto-deploy')                        │   │
│ │                                                              │   │
│ │ [Apply rule] [Review first] [Dismiss]                       │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│ Tiers:                                                               │
│ ├── Standard: Alerts only (included in Cloud Armor)               │
│ └── Plus: Alerts + auto-deploy + advanced analysis (paid add-on) │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Bot Management

```
┌─────────────────────────────────────────────────────────────────────┐
│           BOT MANAGEMENT                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud Armor can detect and handle bot traffic with reCAPTCHA       │
│ Enterprise integration.                                             │
│                                                                       │
│ Bot detection methods:                                               │
│ ├── reCAPTCHA score (0.0 to 1.0)                                  │
│ │   0.0 = definitely a bot                                        │
│ │   1.0 = definitely human                                        │
│ ├── Token validation (reCAPTCHA tokens in headers)                │
│ ├── Challenge page (redirect to CAPTCHA)                          │
│ └── Header-based User-Agent analysis                              │
│                                                                       │
│ Rule example (reCAPTCHA):                                           │
│                                                                       │
│ Priority: [5000]                                                    │
│ Expression:                                                          │
│   token.recaptcha_session.score < 0.3 &&                           │
│   request.path.matches('/api/login')                               │
│ Action: Deny 403                                                    │
│ (Block likely bots on login page)                                  │
│                                                                       │
│ Rule example (redirect to CAPTCHA):                                  │
│                                                                       │
│ Priority: [5100]                                                    │
│ Expression:                                                          │
│   token.recaptcha_session.score < 0.5                              │
│ Action: Redirect                                                    │
│ Type: GOOGLE_RECAPTCHA                                              │
│ (Show CAPTCHA challenge to suspicious visitors)                    │
│                                                                       │
│ Setup:                                                               │
│ 1. Enable reCAPTCHA Enterprise in GCP console                     │
│ 2. Create reCAPTCHA site key                                       │
│ 3. Add reCAPTCHA JavaScript to your web pages                     │
│ 4. Cloud Armor evaluates reCAPTCHA token automatically            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Named IP Lists

```
┌─────────────────────────────────────────────────────────────────────┐
│           NAMED IP LISTS                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pre-maintained lists of IP ranges from known providers.            │
│ Google maintains these lists — they auto-update!                   │
│                                                                       │
│ Available named IP lists:                                            │
│ ├── Fastly CDN IPs                                                │
│ ├── Cloudflare IPs                                                 │
│ ├── Imperva (Incapsula) IPs                                       │
│ └── More providers added over time                                 │
│                                                                       │
│ Use case: "Only allow traffic from Cloudflare"                     │
│ (If you're using Cloudflare as CDN in front of GCP)               │
│                                                                       │
│ Rule:                                                                │
│ Priority: [100]                                                     │
│ Expression:                                                          │
│   evaluatePreconfiguredExpr('sourceiplist-cloudflare')             │
│ Action: Allow                                                       │
│                                                                       │
│ Default rule: Deny (block everything not from Cloudflare)          │
│                                                                       │
│ ⚡ No need to manually maintain CDN IP lists!                        │
│    Google updates them automatically.                               │
│    AWS WAF requires manual IP set updates.                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: CEL Expression Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           COMMON EXPRESSIONS (CEL - Common Expression Language)      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud Armor uses CEL for custom rule expressions.                  │
│                                                                       │
│ ── Source matching ──                                                │
│ origin.ip == '1.2.3.4'                                              │
│ inIpRange(origin.ip, '10.0.0.0/8')                                 │
│ origin.region_code == 'IN'                                          │
│ origin.region_code != 'US'                                          │
│ origin.asn == 15169  (Google's ASN)                                 │
│                                                                       │
│ ── Request matching ──                                               │
│ request.path == '/admin'                                            │
│ request.path.matches('/api/v[0-9]+/.*')                            │
│ request.method == 'POST'                                            │
│ request.headers['user-agent'].contains('bot')                      │
│ request.headers['user-agent'].matches('(?i).*googlebot.*')        │
│ has(request.headers['x-api-key'])                                  │
│ request.query.contains('debug=true')                               │
│ request.headers['host'] == 'api.techcorp.com'                     │
│                                                                       │
│ ── Size matching ──                                                  │
│ request.headers['content-length'].asInt() > 1048576  (>1MB)       │
│                                                                       │
│ ── Combining conditions ──                                           │
│ origin.region_code == 'US' && request.path.matches('/api/.*')     │
│ origin.region_code == 'CN' || origin.region_code == 'RU'          │
│ !(origin.region_code == 'IN' || origin.region_code == 'US')       │
│                                                                       │
│ ── Pre-configured WAF ──                                             │
│ evaluatePreconfiguredExpr('sqli-v33-stable')                      │
│ evaluatePreconfiguredExpr('xss-v33-stable')                       │
│ evaluatePreconfiguredExpr(                                          │
│   'sqli-v33-stable',                                                │
│   ['owasp-crs-v030301-id942120-sqli']  // exclude false positive  │
│ )                                                                    │
│                                                                       │
│ ── reCAPTCHA ──                                                      │
│ token.recaptcha_session.score >= 0.5                                │
│ token.recaptcha_action.score < 0.3                                  │
│ token.recaptcha_action.valid                                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Logging & Monitoring

```
┌─────────────────────────────────────────────────────────────────────┐
│           LOGGING & MONITORING                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Cloud Armor logs:                                                    │
│ ├── Logged as part of LB request logs                             │
│ ├── Enable: Backend service → Logging → Sample rate [1.0]        │
│ ├── Log fields:                                                    │
│ │   ├── enforcedSecurityPolicy.name: Policy name                  │
│ │   ├── enforcedSecurityPolicy.priority: Matched rule priority    │
│ │   ├── enforcedSecurityPolicy.outcome: ACCEPT/DENY              │
│ │   ├── enforcedSecurityPolicy.matchedRule: Rule expression       │
│ │   └── adaptiveProtection.autoDeployAlertId: AP alert ID        │
│ └── View in: Cloud Logging                                        │
│                                                                       │
│ Cloud Logging query:                                                 │
│ resource.type="http_load_balancer"                                  │
│ jsonPayload.enforcedSecurityPolicy.outcome="DENY"                  │
│                                                                       │
│ Cloud Monitoring metrics:                                            │
│ ├── request_count (by rule, outcome)                              │
│ ├── preview_request_count (for preview-mode rules)                │
│ └── Dashboards: Pre-built Cloud Armor dashboard                   │
│                                                                       │
│ Alerts:                                                              │
│ ├── Denied requests spike (potential attack)                      │
│ ├── Adaptive Protection alerts (auto)                             │
│ ├── Rate limit threshold exceeded                                  │
│ └── High 403/429 rate                                              │
│                                                                       │
│ Preview mode:                                                        │
│ ├── Any rule can be set to "Preview" mode                         │
│ ├── Logs what WOULD be blocked, but allows traffic through       │
│ ├── Use to test new rules before enforcing                        │
│ └── Check logs → No false positives → Remove preview → Enforce  │
│                                                                       │
│ Enable preview on a rule:                                            │
│ gcloud compute security-policies rules update 2000 \                │
│   --security-policy=policy-prod \                                    │
│   --preview                                                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 9: Terraform Example

```hcl
# Cloud Armor security policy
resource "google_compute_security_policy" "prod" {
  name        = "policy-prod"
  description = "Production WAF policy"

  # Adaptive protection
  adaptive_protection_config {
    layer_7_ddos_defense_config {
      enable = true
    }
    auto_deploy_config {
      load_threshold              = 0.1
      confidence_threshold        = 0.8
      impacted_baseline_threshold = 0.01
      expiration_sec              = 7200
    }
  }

  # Default rule: Allow
  rule {
    action   = "allow"
    priority = 2147483647
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
    description = "Default allow"
  }

  # Rule 1: Allow office IPs (highest priority)
  rule {
    action   = "allow"
    priority = 500
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["203.0.113.0/24", "198.51.100.0/24"]
      }
    }
    description = "Allow office IP ranges"
  }

  # Rule 2: Block specific countries
  rule {
    action   = "deny(403)"
    priority = 1000
    match {
      expr {
        expression = "origin.region_code == 'CN' || origin.region_code == 'RU' || origin.region_code == 'KP'"
      }
    }
    description = "Block high-risk countries"
  }

  # Rule 3: Block SQL injection
  rule {
    action   = "deny(403)"
    priority = 2000
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('sqli-v33-stable')"
      }
    }
    description = "Block SQL injection"
  }

  # Rule 4: Block XSS
  rule {
    action   = "deny(403)"
    priority = 2100
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('xss-v33-stable')"
      }
    }
    description = "Block cross-site scripting"
  }

  # Rule 5: Block RCE + LFI
  rule {
    action   = "deny(403)"
    priority = 2200
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('rce-v33-stable') || evaluatePreconfiguredExpr('lfi-v33-stable')"
      }
    }
    description = "Block RCE and LFI"
  }

  # Rule 6: Rate limit API
  rule {
    action   = "rate_based_ban"
    priority = 3000
    match {
      expr {
        expression = "request.path.matches('/api/.*')"
      }
    }
    rate_limit_options {
      conform_action = "allow"
      exceed_action  = "deny(429)"
      rate_limit_threshold {
        count        = 100
        interval_sec = 60
      }
      ban_duration_sec = 300
      enforce_on_key   = "IP"
    }
    description = "Rate limit API: 100 req/min per IP"
  }

  # Rule 7: Rate limit login (stricter)
  rule {
    action   = "rate_based_ban"
    priority = 3100
    match {
      expr {
        expression = "request.path == '/api/login' && request.method == 'POST'"
      }
    }
    rate_limit_options {
      conform_action = "allow"
      exceed_action  = "deny(429)"
      rate_limit_threshold {
        count        = 10
        interval_sec = 60
      }
      ban_duration_sec = 600
      enforce_on_key   = "IP"
    }
    description = "Rate limit login: 10 req/min per IP"
  }

  # Rule 8: Block vulnerability scanners
  rule {
    action   = "deny(403)"
    priority = 4000
    match {
      expr {
        expression = "request.headers['user-agent'].matches('(?i).*(sqlmap|nikto|nmap|masscan|zgrab|dirbuster).*')"
      }
    }
    description = "Block known scanners"
  }
}

# Attach policy to backend service
resource "google_compute_backend_service" "web" {
  name            = "bs-web-prod"
  security_policy = google_compute_security_policy.prod.id
  # ... other backend service config
}

# Edge security policy (for CDN backends)
resource "google_compute_security_policy" "edge" {
  name        = "policy-edge-cdn"
  description = "Edge policy for CDN backends"
  type        = "CLOUD_ARMOR_EDGE"

  rule {
    action   = "deny(403)"
    priority = 1000
    match {
      expr {
        expression = "origin.region_code == 'KP'"
      }
    }
    description = "Block North Korea at edge"
  }

  rule {
    action   = "allow"
    priority = 2147483647
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
    description = "Default allow"
  }
}
```

---

## Part 10: gcloud CLI Reference

```bash
# Create security policy
gcloud compute security-policies create policy-prod \
  --description="Production WAF policy"

# Add rule: Block countries
gcloud compute security-policies rules create 1000 \
  --security-policy=policy-prod \
  --expression="origin.region_code == 'CN' || origin.region_code == 'RU'" \
  --action=deny-403 \
  --description="Block high-risk countries"

# Add rule: SQLi protection
gcloud compute security-policies rules create 2000 \
  --security-policy=policy-prod \
  --expression="evaluatePreconfiguredExpr('sqli-v33-stable')" \
  --action=deny-403 \
  --description="Block SQL injection"

# Add rule: Rate limiting
gcloud compute security-policies rules create 3000 \
  --security-policy=policy-prod \
  --expression="request.path.matches('/api/.*')" \
  --action=rate-based-ban \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=300 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP \
  --description="Rate limit API"

# Enable adaptive protection
gcloud compute security-policies update policy-prod \
  --enable-layer7-ddos-defense

# Attach to backend service
gcloud compute backend-services update bs-web-prod \
  --security-policy=policy-prod \
  --global

# Preview mode (test without blocking)
gcloud compute security-policies rules update 2000 \
  --security-policy=policy-prod \
  --preview

# Remove preview (enforce)
gcloud compute security-policies rules update 2000 \
  --security-policy=policy-prod \
  --no-preview

# List rules
gcloud compute security-policies rules list \
  --security-policy=policy-prod

# View policy
gcloud compute security-policies describe policy-prod

# Delete a rule
gcloud compute security-policies rules delete 1000 \
  --security-policy=policy-prod
```

---

## Part 11: Real-World Patterns

### Startup

```
Policy: 1 security policy attached to web backend service

Rules:
├── Priority 2000: SQLi protection (sqli-v33-stable)
├── Priority 2100: XSS protection (xss-v33-stable)
├── Priority 3000: Rate limit /api/* (100 req/min per IP)
├── Priority 3100: Rate limit /login (10 req/min per IP)
└── Default: Allow

Adaptive Protection: Enabled (free alerts)
Preview mode: Test new rules for 1 week before enforcing

No geo-blocking (need global users).
No bot management yet (add when needed).

Cost: ~$5/month (1 policy + request volume)
```

### Mid-Size

```
Policies:
├── policy-web (attached to bs-web-prod)
│   ├── Allow office IPs (priority 500)
│   ├── Block high-risk countries (priority 1000)
│   ├── SQLi + XSS + RCE + LFI (priority 2000-2300)
│   ├── Scanner detection (priority 2500)
│   ├── Rate limit: 200 req/min per IP (priority 3000)
│   ├── Rate limit login: 10 req/min per IP (priority 3100)
│   └── Block known scanners by UA (priority 4000)
│
└── policy-api (attached to bs-api-prod)
    ├── Allow only known IP ranges (priority 100)
    ├── SQLi + XSS (priority 2000-2100)
    ├── Rate limit: 500 req/min per IP (priority 3000)
    ├── Require X-API-Key header (priority 4000)
    └── Default: Deny (API is closed by default)

Adaptive Protection: Enabled with auto-deploy
Bot management: reCAPTCHA on login page

Cost: ~$20-50/month
```

### Enterprise

```
Policies:
├── policy-web-prod (web application)
│   ├── Named IP lists (only allow Cloudflare if using CF CDN)
│   ├── Geo-blocking (compliance: block sanctioned countries)
│   ├── Full OWASP rule set (all stable expressions)
│   ├── Custom expressions for business logic
│   ├── Rate limiting per endpoint group
│   ├── Bot management with reCAPTCHA Enterprise
│   └── Adaptive Protection with auto-deploy
│
├── policy-api-external (public API)
│   ├── API key validation at WAF level
│   ├── Strict rate limiting (tiered by API plan)
│   ├── SQLi/XSS protection
│   ├── Request size limits
│   └── IP allowlist for enterprise customers
│
├── policy-admin (admin panel)
│   ├── IP allowlist only (office + VPN)
│   ├── Default: Deny everything else
│   └── Full OWASP protection
│
└── policy-edge-cdn (CDN content)
    ├── Edge policy type
    ├── Geo-blocking at edge (before CDN cache)
    └── Default: Allow

Process:
├── New rules deployed in Preview mode first
├── Monitor for 1 week (check false positives in logs)
├── Review with security team
├── Promote to enforcement
└── Monthly rule review and tuning

Cost: ~$100-500/month (Cloud Armor Enterprise)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| Policy types | Backend, Edge, Network Edge |
| Rule evaluation | Priority (lower = first), first match wins |
| Actions | Allow, Deny (403/404/502), Throttle, Rate-based ban, Redirect |
| Max rules/policy | 200 |
| WAF rules | Pre-configured OWASP expressions (v33-stable) |
| Rate limiting | Per IP, ALL, header, cookie, XFF, region |
| Adaptive Protection | ML-based DDoS detection + auto-deploy |
| Bot management | reCAPTCHA Enterprise integration |
| Named IP lists | CDN provider IPs (auto-maintained by Google) |
| Preview mode | Test rules without blocking traffic |
| Expression language | CEL (Common Expression Language) |
| Cost | ~$5/month per policy + $0.75/million requests |
| AWS equivalent | AWS WAF ($5/rule/month + $0.60/million requests) |
| Azure equivalent | WAF built into App Gateway WAF_v2 SKU |

---

## Console Walkthrough: Deleting Security Policies

You can't delete a Cloud Armor security policy while it's still attached to a backend service. Here's the safe order of operations.

### Step 1: Export the Policy for Backup

Before deleting anything, export the policy so you can recreate it if needed:

```bash
# Export policy as JSON
gcloud compute security-policies describe my-web-policy \
    --format=json > my-web-policy-backup.json

# Export policy as YAML
gcloud compute security-policies describe my-web-policy \
    --format=yaml > my-web-policy-backup.yaml
```

> **💡 Tip:** Always back up security policies before deletion. Recreating complex WAF rules from memory is painful.

### Step 2: Detach the Policy from Backend Services

A policy must be detached from all backend services before it can be deleted.

**From Console:**

1. Go to **Network Security → Cloud Armor policies**
2. Click the policy name
3. Go to the **Targets** tab
4. For each backend service, click **Remove** to detach the policy

**Via CLI:**

```bash
# See which backends use this policy
gcloud compute backend-services list \
    --filter="securityPolicy:my-web-policy" \
    --format="value(name)"

# Detach policy from a backend service
gcloud compute backend-services update my-backend-service \
    --security-policy="" \
    --global
```

> **⚠️ Warning:** Once you detach the policy, the backend service has NO WAF protection. Traffic will flow through unfiltered. Plan for this — either attach a different policy first or do it during a maintenance window.

### Step 3: Delete Individual Rules (Optional)

If you just want to clean up specific rules within a policy (not delete the whole policy):

```
Console → Cloud Armor → select policy → Rules tab → select rule → DELETE
```

```bash
# Delete a specific rule by priority
gcloud compute security-policies rules delete 2000 \
    --security-policy=my-web-policy
```

### Step 4: Delete the Policy

**From Console:**

1. Go to **Network Security → Cloud Armor policies**
2. Select the policy checkbox
3. Click **DELETE** at the top → confirm

**Via CLI:**

```bash
# Delete the policy
gcloud compute security-policies delete my-web-policy --quiet
```

If you get an error like `The security policy resource is still being used`, you missed detaching from a backend — go back to Step 2.

### Complete Cleanup Sequence

```bash
# 1. Backup
gcloud compute security-policies describe my-web-policy \
    --format=json > backup-my-web-policy.json

# 2. Detach from all backends
for BACKEND in $(gcloud compute backend-services list \
    --filter="securityPolicy:my-web-policy" \
    --format="value(name)"); do
  echo "Detaching from $BACKEND..."
  gcloud compute backend-services update "$BACKEND" \
      --security-policy="" --global
done

# 3. Delete the policy
gcloud compute security-policies delete my-web-policy --quiet
```

---

## What's Next?

In the next chapter, we'll cover Cloud Interconnect & VPN — connecting your on-premises network to GCP.

→ Next: [Chapter 13: Cloud Interconnect & VPN](13-interconnect-vpn.md)

---

*Last Updated: May 2026*
