# DDoS Protection & WAF (Web Application Firewall)

> **What you'll learn**: How Distributed Denial of Service attacks overwhelm applications, how Web Application Firewalls block malicious requests, and how companies like Cloudflare and AWS Shield protect against attacks handling millions of requests per second.

---

## Real-Life Analogy

### DDoS Attack = Flash Mob at a Restaurant

Imagine you own a popular restaurant with 50 seats.

**Normal day**: 200 customers come throughout the day. Everyone gets served.

**DDoS attack**: Your competitor pays 10,000 people to show up at once, sit in all seats, order water, and never leave. REAL customers can't get in. The restaurant is "up" but completely unusable.

**DDoS Protection**: You hire a bouncer (WAF/CDN) who:
1. Recognizes that 10,000 people arriving simultaneously is abnormal
2. Checks if they're real customers (CAPTCHA, rate check)
3. Blocks the fake crowd at the door
4. Lets legitimate customers through

```
WITHOUT PROTECTION:                     WITH PROTECTION:
                                        
10,000 bots ──────▶ YOUR SERVER 💥      10,000 bots ──▶ CLOUDFLARE ──▶ 🗑️ (blocked)
                    (overwhelmed)                           │
100 real users ──▶ TIMEOUT ❌           100 real users ──▶ │ ──▶ YOUR SERVER ✓
                                                           (filters traffic)
```

### WAF = Security Guard with a Rulebook

A **WAF** is like a security guard who reads every letter (HTTP request) before delivering it:
- "This letter contains a bomb threat (SQL injection)" → BLOCKED
- "This letter asks for the master key (path traversal)" → BLOCKED
- "This is a normal package from a known sender" → ALLOWED

---

## Core Concept Explained Step-by-Step

### Step 1: Types of DDoS Attacks

```
┌─────────────────────────────────────────────────────────────────┐
│                     DDoS ATTACK TYPES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  LAYER 3/4 — NETWORK/TRANSPORT (Volumetric)                      │
│  ────────────────────────────────────────────                    │
│  Goal: Saturate bandwidth / exhaust network resources            │
│  │                                                               │
│  ├── UDP Flood: Send massive amounts of UDP packets              │
│  ├── SYN Flood: Send millions of TCP SYN (never complete)        │
│  ├── ICMP Flood: Overwhelm with ping packets                     │
│  └── Amplification: Abuse DNS/NTP to multiply traffic 50-100x   │
│      (Send small request to DNS, it sends large response to you) │
│                                                                  │
│  LAYER 7 — APPLICATION (Sophisticated)                           │
│  ──────────────────────────────────────                          │
│  Goal: Exhaust application resources (CPU, memory, DB)           │
│  │                                                               │
│  ├── HTTP Flood: Millions of legitimate-looking GET/POST         │
│  ├── Slowloris: Keep connections open as long as possible        │
│  ├── API Abuse: Call expensive endpoints repeatedly              │
│  └── Cache-Busting: Unique URLs to bypass CDN cache              │
│                                                                  │
│  Layer 3/4 = "Flood the highway with cars"                       │
│  Layer 7   = "Send millions of valid-looking customers"          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: DDoS Attack Anatomy

```
┌─────────────────────────────────────────────────────────────────┐
│                    DDoS ATTACK INFRASTRUCTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│         ┌─────────────┐                                          │
│         │  Attacker   │  (Command & Control)                     │
│         └──────┬──────┘                                          │
│                │ Commands                                         │
│    ┌───────────┼───────────────┐                                 │
│    ▼           ▼               ▼                                 │
│  ┌────┐    ┌────┐          ┌────┐                               │
│  │Bot │    │Bot │  ...     │Bot │   Botnet: 100,000+            │
│  │ 1  │    │ 2  │          │ N  │   compromised devices          │
│  └──┬─┘    └──┬─┘          └──┬─┘                               │
│     │         │               │                                  │
│     └─────────┼───────────────┘                                  │
│               │ Millions of requests                             │
│               ▼                                                  │
│         ┌──────────┐                                             │
│         │  TARGET  │  ← Overwhelmed!                             │
│         │  SERVER  │                                             │
│         └──────────┘                                             │
│                                                                  │
│  Scale: Largest recorded attacks exceed 3 Tbps (terabits/sec)   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: Multi-Layer Defense Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  DEFENSE-IN-DEPTH ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Internet Traffic                                                │
│       │                                                          │
│       ▼                                                          │
│  ┌──────────────────┐  Layer 1: DNS-level protection            │
│  │  CDN / Anycast   │  - Absorb volumetric attacks              │
│  │  (Cloudflare,    │  - Distribute across global network        │
│  │   AWS CloudFront)│  - Block known bad IPs                     │
│  └────────┬─────────┘                                            │
│           ▼                                                      │
│  ┌──────────────────┐  Layer 2: Edge protection                  │
│  │  WAF             │  - Block SQL injection, XSS               │
│  │  (AWS WAF,       │  - Rate limiting per IP/region            │
│  │   Cloudflare WAF)│  - Bot detection (CAPTCHA)                │
│  └────────┬─────────┘                                            │
│           ▼                                                      │
│  ┌──────────────────┐  Layer 3: Load balancer                    │
│  │  Load Balancer   │  - Connection limiting                     │
│  │  (ALB/NLB)       │  - Health checks (drop unhealthy)         │
│  └────────┬─────────┘                                            │
│           ▼                                                      │
│  ┌──────────────────┐  Layer 4: Application level                │
│  │  Application     │  - Request validation                      │
│  │  (Rate limiting, │  - Authentication checks                   │
│  │   input checks)  │  - Graceful degradation                    │
│  └──────────────────┘                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 4: WAF — How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    WAF REQUEST INSPECTION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Incoming HTTP Request                                           │
│  ┌─────────────────────────────────────────────────┐            │
│  │ POST /api/users HTTP/1.1                         │            │
│  │ Host: app.example.com                            │            │
│  │ User-Agent: Mozilla/5.0                          │            │
│  │ Content-Type: application/json                   │            │
│  │                                                  │            │
│  │ {"name": "'; DROP TABLE users; --"}              │            │
│  └─────────────────────────────────────────────────┘            │
│                          │                                       │
│                          ▼                                       │
│  ┌─────────────────────────────────────────────────┐            │
│  │              WAF RULE ENGINE                      │            │
│  ├─────────────────────────────────────────────────┤            │
│  │                                                  │            │
│  │  Rule 1: SQL Injection patterns ──▶ MATCH! 🚨    │            │
│  │    Pattern: '; DROP|UNION SELECT|OR 1=1          │            │
│  │                                                  │            │
│  │  Rule 2: XSS patterns ──▶ No match              │            │
│  │    Pattern: <script>|javascript:|onerror=        │            │
│  │                                                  │            │
│  │  Rule 3: Path traversal ──▶ No match            │            │
│  │    Pattern: \.\./|/etc/passwd|/proc/             │            │
│  │                                                  │            │
│  │  Rule 4: Rate limit ──▶ Under limit             │            │
│  │    Limit: 100 req/min per IP                     │            │
│  │                                                  │            │
│  │  VERDICT: BLOCK (Rule 1 matched)                 │            │
│  │  Response: 403 Forbidden                         │            │
│  │  Log: SQL injection attempt from 1.2.3.4         │            │
│  │                                                  │            │
│  └─────────────────────────────────────────────────┘            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### SYN Flood Attack and Defense

```
NORMAL TCP Handshake:          SYN FLOOD Attack:
┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐
│ Client │     │ Server │     │Attacker│     │ Server │
└───┬────┘     └───┬────┘     └───┬────┘     └───┬────┘
    │ SYN          │               │ SYN (fake IP) │
    │─────────────▶│               │──────────────▶│
    │              │               │               │ Allocates memory
    │ SYN-ACK     │               │ SYN-ACK ───▶ nowhere (spoofed)
    │◀─────────────│               │               │ Waits...
    │              │               │ SYN (fake IP) │
    │ ACK          │               │──────────────▶│
    │─────────────▶│               │               │ More memory...
    │              │               │ SYN ×1000000  │
    │ Connection!  │               │──────────────▶│ 💥 OUT OF MEMORY
                                   
DEFENSE: SYN Cookies — Don't allocate memory until ACK received
```

### DNS Amplification Attack

```
┌────────────────────────────────────────────────────────────┐
│                DNS AMPLIFICATION                             │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Attacker sends small DNS query (64 bytes)               │
│     Source IP: spoofed to VICTIM's IP                        │
│                                                             │
│  2. DNS server responds with large answer (3000 bytes)      │
│     Destination: VICTIM's IP (spoofed source)               │
│                                                             │
│  Amplification factor: 50x                                  │
│  Attacker: 1 Gbps out → Victim receives: 50 Gbps! 💥       │
│                                                             │
│  ┌─────────┐   64B query    ┌─────────┐                   │
│  │Attacker │───────────────▶│  Open   │                   │
│  │         │  src=VICTIM     │  DNS    │                   │
│  └─────────┘                 │ Resolver│                   │
│                              └────┬────┘                   │
│                                   │ 3000B response         │
│                                   │ dst=VICTIM             │
│                              ┌────▼────┐                   │
│                              │ VICTIM  │ 💥 50x traffic    │
│                              │ SERVER  │                   │
│                              └─────────┘                   │
│                                                             │
│  Defense: Rate-limit DNS, disable open resolvers,           │
│           upstream DDoS protection (Cloudflare/AWS Shield) │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### WAF Rule Types

```
┌────────────────────────────────────────────────────────────┐
│                    WAF RULE CATEGORIES                       │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  1. MANAGED RULES (Pre-built by provider)                   │
│     ├── OWASP Core Rule Set (CRS)                          │
│     ├── Known CVE signatures                               │
│     ├── Bot detection patterns                              │
│     └── IP reputation lists                                 │
│                                                             │
│  2. CUSTOM RULES (You write them)                           │
│     ├── Block specific countries                            │
│     ├── Rate limit /api/login to 5/min                     │
│     ├── Block if User-Agent is empty                        │
│     └── Allow only specific IPs to /admin                   │
│                                                             │
│  3. RATE-BASED RULES                                        │
│     ├── Block IP after 1000 requests in 5 minutes          │
│     ├── Block if > 50 4xx errors in 1 minute               │
│     └── Challenge (CAPTCHA) above threshold                 │
│                                                             │
│  4. IP REPUTATION                                           │
│     ├── Known malicious IPs (threat intelligence)          │
│     ├── Tor exit nodes                                      │
│     ├── Known VPN/proxy IPs                                │
│     └── Cloud provider IPs (unusual for human traffic)      │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Application-Level Rate Limiting + DDoS Defense

```python
# ddos_defense.py — Application-level protections
from flask import Flask, request, abort, jsonify
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
import redis
import time

app = Flask(__name__)
redis_client = redis.Redis(host='localhost', port=6379)

# Multi-tier rate limiting
limiter = Limiter(
    app=app,
    key_func=get_remote_address,
    storage_uri="redis://localhost:6379",
    default_limits=["200 per minute", "50 per second"]
)

# --- ADVANCED: Sliding window with burst detection ---
def detect_ddos(ip: str) -> bool:
    """Detect potential DDoS by tracking request patterns"""
    key = f"ddos:counter:{ip}"
    window = 60  # 1 minute window
    threshold = 500  # requests per minute = suspicious
    
    pipe = redis_client.pipeline()
    now = time.time()
    
    pipe.zadd(key, {str(now): now})
    pipe.zremrangebyscore(key, 0, now - window)
    pipe.zcard(key)
    pipe.expire(key, window)
    
    results = pipe.execute()
    count = results[2]
    
    if count > threshold:
        # Block this IP for 10 minutes
        redis_client.setex(f"ddos:blocked:{ip}", 600, "1")
        return True
    return False

@app.before_request
def ddos_check():
    """Check if IP is blocked or behaving suspiciously"""
    ip = request.remote_addr
    
    # Check blocklist
    if redis_client.exists(f"ddos:blocked:{ip}"):
        abort(429, "Too many requests. Please try again later.")
    
    # Check pattern
    if detect_ddos(ip):
        abort(429, "Rate limit exceeded.")

# Endpoint-specific limits
@app.route("/api/login", methods=["POST"])
@limiter.limit("5 per minute")  # Very strict on auth endpoints
def login():
    return jsonify({"status": "ok"})

@app.route("/api/search")
@limiter.limit("30 per minute")  # Expensive operations
def search():
    return jsonify({"results": []})
```

### Python — WAF-like Request Filtering

```python
# waf_middleware.py — Simple application-level WAF patterns
import re
from flask import Flask, request, abort

app = Flask(__name__)

# WAF rule patterns
SQL_INJECTION_PATTERNS = [
    r"(\b(UNION|SELECT|INSERT|UPDATE|DELETE|DROP|ALTER)\b.*\b(FROM|INTO|TABLE|SET)\b)",
    r"(--|#|/\*)",
    r"(\bOR\b\s+\d+\s*=\s*\d+)",
    r"('\s*(OR|AND)\s+')",
]

XSS_PATTERNS = [
    r"(<script[\s>])",
    r"(javascript\s*:)",
    r"(on\w+\s*=)",
    r"(<iframe|<object|<embed)",
]

PATH_TRAVERSAL_PATTERNS = [
    r"(\.\./|\.\.\\)",
    r"(/etc/passwd|/proc/self)",
    r"(\\\\[a-zA-Z]+\\)",
]

def check_waf_rules(value: str) -> str | None:
    """Check a string against WAF rules. Returns rule name if matched."""
    for pattern in SQL_INJECTION_PATTERNS:
        if re.search(pattern, value, re.IGNORECASE):
            return "SQL_INJECTION"
    for pattern in XSS_PATTERNS:
        if re.search(pattern, value, re.IGNORECASE):
            return "XSS"
    for pattern in PATH_TRAVERSAL_PATTERNS:
        if re.search(pattern, value, re.IGNORECASE):
            return "PATH_TRAVERSAL"
    return None

@app.before_request
def waf_filter():
    """Inspect all request parameters for malicious patterns"""
    # Check query parameters
    for key, value in request.args.items():
        rule = check_waf_rules(value)
        if rule:
            app.logger.warning(
                f"WAF BLOCKED: rule={rule} ip={request.remote_addr} "
                f"path={request.path} param={key}"
            )
            abort(403, "Request blocked by security policy")
    
    # Check request body (for POST/PUT)
    if request.is_json and request.json:
        for key, value in flatten_dict(request.json).items():
            if isinstance(value, str):
                rule = check_waf_rules(value)
                if rule:
                    app.logger.warning(f"WAF BLOCKED: rule={rule} body_param={key}")
                    abort(403, "Request blocked by security policy")
```

### Java — Rate Limiting with Resilience4j

```java
import io.github.resilience4j.ratelimiter.RateLimiter;
import io.github.resilience4j.ratelimiter.RateLimiterConfig;
import org.springframework.web.bind.annotation.*;

import java.time.Duration;

@RestController
public class ProtectedController {
    
    private final RateLimiter rateLimiter = RateLimiter.of("api",
        RateLimiterConfig.custom()
            .limitForPeriod(100)           // 100 requests
            .limitRefreshPeriod(Duration.ofMinutes(1))  // per minute
            .timeoutDuration(Duration.ofMillis(0))      // Don't wait, reject
            .build()
    );
    
    @GetMapping("/api/data")
    public ResponseEntity<?> getData() {
        // Check rate limit
        if (!rateLimiter.acquirePermission()) {
            return ResponseEntity.status(429)
                .header("Retry-After", "60")
                .body(Map.of("error", "Rate limit exceeded"));
        }
        
        return ResponseEntity.ok(fetchData());
    }
}
```

---

## Infrastructure Examples

### AWS WAF + Shield + CloudFront

```yaml
# CloudFormation — Complete DDoS protection stack
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  # WAF Web ACL with rules
  WebACL:
    Type: AWS::WAFv2::WebACL
    Properties:
      Name: MyAppWAF
      Scope: CLOUDFRONT
      DefaultAction:
        Allow: {}
      Rules:
        # Block SQL injection
        - Name: AWSManagedRulesSQLi
          Priority: 1
          OverrideAction:
            None: {}
          Statement:
            ManagedRuleGroupStatement:
              VendorName: AWS
              Name: AWSManagedRulesSQLiRuleSet
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: SQLiRule
            
        # Block common exploits
        - Name: AWSManagedRulesCommonRuleSet
          Priority: 2
          OverrideAction:
            None: {}
          Statement:
            ManagedRuleGroupStatement:
              VendorName: AWS
              Name: AWSManagedRulesCommonRuleSet
              
        # Rate limiting (2000 requests per 5 minutes per IP)
        - Name: RateLimit
          Priority: 3
          Action:
            Block: {}
          Statement:
            RateBasedStatement:
              Limit: 2000
              AggregateKeyType: IP
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: RateLimit

        # Geo-blocking (optional)
        - Name: GeoBlock
          Priority: 4
          Action:
            Block: {}
          Statement:
            GeoMatchStatement:
              CountryCodes:
                - CN
                - RU
```

### Nginx — DDoS Mitigation Configuration

```nginx
# /etc/nginx/conf.d/ddos-protection.conf

# Rate limiting zones
limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;

# Connection limiting
limit_conn_zone $binary_remote_addr zone=perip:10m;

# Block known bad User-Agents
map $http_user_agent $bad_bot {
    default 0;
    ~*(bot|crawler|spider|scraper) 0;  # Allow legitimate bots
    ~*(nikto|sqlmap|nmap|masscan) 1;   # Block attack tools
    "" 1;                               # Block empty UA
}

server {
    listen 443 ssl http2;
    server_name app.example.com;
    
    # Connection limits
    limit_conn perip 50;           # Max 50 connections per IP
    limit_conn_status 429;
    
    # Block bad bots
    if ($bad_bot) {
        return 403;
    }
    
    # Slow client protection (Slowloris defense)
    client_header_timeout 10s;
    client_body_timeout 10s;
    send_timeout 10s;
    keepalive_timeout 30s;
    
    # General endpoints
    location / {
        limit_req zone=general burst=20 nodelay;
        proxy_pass http://backend;
    }
    
    # Login — very strict
    location /api/login {
        limit_req zone=login burst=3 nodelay;
        proxy_pass http://backend;
    }
    
    # API endpoints
    location /api/ {
        limit_req zone=api burst=50 nodelay;
        proxy_pass http://backend;
    }
}
```

### Cloudflare WAF Rules (Terraform)

```hcl
# terraform — Cloudflare WAF configuration
resource "cloudflare_ruleset" "waf" {
  zone_id     = var.zone_id
  name        = "Custom WAF Rules"
  kind        = "zone"
  phase       = "http_request_firewall_custom"

  # Block SQL injection attempts
  rules {
    action      = "block"
    expression  = "(http.request.uri.query contains \"UNION SELECT\" or http.request.body contains \"'; DROP\")"
    description = "Block SQL injection"
  }

  # Rate limit login endpoint
  rules {
    action      = "block"
    expression  = "(http.request.uri.path eq \"/api/login\" and rate(5m) > 10)"
    description = "Rate limit login"
  }

  # Challenge suspicious traffic
  rules {
    action      = "managed_challenge"
    expression  = "(cf.threat_score > 30)"
    description = "Challenge high threat score"
  }
  
  # Block known attack tools
  rules {
    action      = "block"
    expression  = "(http.user_agent contains \"sqlmap\" or http.user_agent contains \"nikto\")"
    description = "Block attack tools"
  }
}
```

---

## Real-World Example

### How Cloudflare Handles DDoS at Scale

```
┌────────────────────────────────────────────────────────────────┐
│              CLOUDFLARE DDoS PROTECTION                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Network capacity: 248+ Tbps (can absorb any known attack)     │
│  Points of presence: 310+ cities worldwide                     │
│  Average mitigation time: < 3 seconds                          │
│                                                                 │
│  Architecture:                                                  │
│  ┌─────────┐    ┌──────────────┐    ┌───────────┐             │
│  │ Attack  │───▶│  Anycast     │───▶│ Filtering │             │
│  │ Traffic │    │  Network     │    │ Engine    │             │
│  │(Tbps)   │    │(distributed  │    │           │             │
│  └─────────┘    │ across 310+  │    │ ML-based  │             │
│                 │ locations)    │    │ detection │             │
│                 └──────────────┘    └─────┬─────┘             │
│                                           │                    │
│                                    ┌──────▼──────┐             │
│                                    │ Clean       │             │
│                                    │ Traffic     │──▶ Origin    │
│                                    │ (only)      │    Server    │
│                                    └─────────────┘             │
│                                                                 │
│  Key: Traffic is absorbed across the entire global network      │
│       No single point gets overwhelmed                          │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### GitHub DDoS Attack (2018) — Largest Ever at the Time

- **Attack size**: 1.35 Tbps (memcached amplification)
- **Duration**: ~10 minutes of impact
- **Defense**: Traffic routed to Akamai Prolexic (DDoS protection)
- **Result**: Site down for ~5 minutes, then fully mitigated
- **Lesson**: Even GitHub needs upstream DDoS protection

### AWS Shield Advanced Features

```
AWS Shield Standard (free):
  ✓ Layer 3/4 DDoS protection
  ✓ Automatic, always-on
  ✓ No additional charge

AWS Shield Advanced ($3,000/month):
  ✓ Everything in Standard
  ✓ Layer 7 protection
  ✓ DDoS Response Team (DRT) — AWS experts help you
  ✓ Cost protection (AWS credits if DDoS causes scaling costs)
  ✓ Real-time attack visibility
  ✓ Health-based detection
  ✓ Advanced metrics and reporting
```

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| No rate limiting on auth endpoints | Brute force attacks | 5-10 attempts per minute max |
| WAF in detection-only mode forever | Attacks logged but not blocked | Start blocking after tuning |
| Overly strict WAF rules | Blocking legitimate users | Test with real traffic first |
| No DDoS protection for APIs | API DDoS harder to detect | Use rate limiting + WAF |
| Relying only on application-level defense | Bandwidth saturation | Use CDN/upstream protection |
| No alerting on WAF blocks | Attacks go unnoticed | Alert on spike in 403/429 |
| Blocking by country alone | VPNs bypass it | Layer with other techniques |
| No graceful degradation plan | Complete outage during attack | Serve static pages under attack |

### WAF False Positives — The Silent Killer

```
Common false positives:
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  Rule: SQL injection                                          │
│  Blocked: User named "O'Brien" (apostrophe triggers rule!)   │
│  Fix: Context-aware rules, allowlist patterns                 │
│                                                               │
│  Rule: XSS detection                                          │
│  Blocked: Developer posting code with <script> in tutorial   │
│  Fix: Exclude specific paths (/blog/*, /docs/*)              │
│                                                               │
│  Rule: Path traversal                                         │
│  Blocked: Legitimate file path "../assets/image.png"         │
│  Fix: More specific patterns, path-aware rules               │
│                                                               │
│  LESSON: Always start WAF in "log only" mode, tune rules,    │
│          then switch to blocking after validating no FPs      │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## When to Use / When NOT to Use

### DDoS Protection Tiers:

| App Size | Protection Level |
|----------|-----------------|
| Side project / small blog | Cloudflare Free tier (basic) |
| Startup / medium traffic | Cloudflare Pro + Rate Limiting |
| Enterprise / high traffic | AWS Shield Advanced + WAF |
| Critical infrastructure | Multi-vendor (Cloudflare + AWS) + on-premise scrubbing |

### WAF Decision Guide:

| Need | Solution |
|------|----------|
| Basic protection (OWASP Top 10) | Managed WAF rules (AWS/Cloudflare) |
| Custom business logic rules | Custom WAF rules + rate limiting |
| Bot protection (scrapers) | Bot management (Cloudflare Bot Management) |
| API-specific protection | API Gateway + WAF + rate limiting |
| Compliance (PCI-DSS, SOC2) | WAF + logging + regular rule updates |

---

## Key Takeaways

- **DDoS attacks** overwhelm resources at network (L3/4) or application (L7) layers
- **You cannot defend against volumetric DDoS alone** — you need upstream protection (CDN, cloud provider)
- **WAFs** inspect HTTP requests and block malicious patterns (SQLi, XSS, etc.)
- **Rate limiting** is your first line of defense — implement at multiple levels (CDN → LB → App)
- **Start WAF in log-only mode** — tune rules before blocking to avoid false positives
- **Layer your defenses**: CDN + WAF + Rate Limiting + Application validation
- **Plan for degradation** — have a static fallback page ready if your origin is overwhelmed
- **Alerting is critical** — detect attacks in seconds, not hours

---

## What's Next?

Next, we'll explore **Zero Trust Architecture** — the security model that says "never trust, always verify" — even for traffic inside your own network. No more "trusted internal network." That's Chapter 14.10.
