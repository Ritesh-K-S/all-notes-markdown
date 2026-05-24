# API Security Best Practices

> **What you'll learn**: How to secure REST APIs against common attacks — including authentication, input validation, rate limiting, and protecting sensitive data — with production-ready patterns used by companies like Stripe, Twilio, and GitHub.

---

## Real-Life Analogy

Think of your API as a **bank teller window**:

- **Authentication** = Showing your ID before accessing your account
- **Authorization** = The teller checking if you're allowed to withdraw from THAT account
- **Input validation** = The teller verifying the withdrawal slip is filled out correctly
- **Rate limiting** = "Maximum 5 transactions per hour per customer"
- **Logging** = Security cameras recording every transaction
- **Encryption** = Speaking through a private booth so nobody else hears

```
┌─────────────────────────────────────────────────────────────────┐
│                     API SECURITY LAYERS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ Layer 1: Transport Security (HTTPS/TLS)             │ ← Ch14.4│
│  │ ┌─────────────────────────────────────────────────┐ │        │
│  │ │ Layer 2: Authentication (WHO are you?)          │ │        │
│  │ │ ┌─────────────────────────────────────────────┐ │ │        │
│  │ │ │ Layer 3: Authorization (WHAT can you do?)   │ │ │        │
│  │ │ │ ┌─────────────────────────────────────────┐ │ │ │        │
│  │ │ │ │ Layer 4: Input Validation               │ │ │ │        │
│  │ │ │ │ ┌─────────────────────────────────────┐ │ │ │ │        │
│  │ │ │ │ │ Layer 5: Rate Limiting              │ │ │ │ │        │
│  │ │ │ │ │ ┌─────────────────────────────────┐ │ │ │ │ │        │
│  │ │ │ │ │ │ Layer 6: Your Business Logic    │ │ │ │ │ │        │
│  │ │ │ │ │ └─────────────────────────────────┘ │ │ │ │ │        │
│  │ │ │ │ └─────────────────────────────────────┘ │ │ │ │        │
│  │ │ │ └─────────────────────────────────────────┘ │ │ │        │
│  │ │ └─────────────────────────────────────────────┘ │ │        │
│  │ └─────────────────────────────────────────────────┘ │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Concept Explained Step-by-Step

### The 10 Pillars of API Security

```
┌────┬─────────────────────────────┬────────────────────────────────┐
│ #  │ Principle                   │ What It Means                   │
├────┼─────────────────────────────┼────────────────────────────────┤
│ 1  │ Always use HTTPS            │ Encrypt all traffic              │
│ 2  │ Authenticate every request  │ Know WHO is calling              │
│ 3  │ Authorize every action      │ Check WHAT they can do           │
│ 4  │ Validate all input          │ Never trust the client           │
│ 5  │ Rate limit everything       │ Prevent abuse                    │
│ 6  │ Return minimal data         │ Don't leak extra info            │
│ 7  │ Log security events         │ Detect attacks                   │
│ 8  │ Version your APIs           │ Safely deprecate old versions    │
│ 9  │ Handle errors safely        │ Don't leak internals in errors   │
│ 10 │ Use security headers        │ Browser-level protection         │
└────┴─────────────────────────────┴────────────────────────────────┘
```

---

## How It Works Internally

### API Request Security Pipeline

```
┌────────────────────────────────────────────────────────────────────┐
│                    API REQUEST PIPELINE                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Client Request                                                      │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────┐  Reject if not HTTPS                               │
│  │ TLS Check   │──────────────────────▶ 301 Redirect to HTTPS       │
│  └──────┬──────┘                                                     │
│         ▼                                                            │
│  ┌─────────────┐  No token / invalid token                           │
│  │ AuthN Check │──────────────────────▶ 401 Unauthorized             │
│  └──────┬──────┘                                                     │
│         ▼                                                            │
│  ┌─────────────┐  Over limit                                        │
│  │ Rate Limit  │──────────────────────▶ 429 Too Many Requests       │
│  └──────┬──────┘                                                     │
│         ▼                                                            │
│  ┌─────────────┐  Invalid / malicious input                          │
│  │ Validation  │──────────────────────▶ 400 Bad Request              │
│  └──────┬──────┘                                                     │
│         ▼                                                            │
│  ┌─────────────┐  Insufficient permissions                           │
│  │ AuthZ Check │──────────────────────▶ 403 Forbidden                │
│  └──────┬──────┘                                                     │
│         ▼                                                            │
│  ┌─────────────┐                                                     │
│  │ Business    │──────────────────────▶ 200 OK (minimal response)    │
│  │ Logic       │                                                     │
│  └─────────────┘                                                     │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Complete Secure API

```python
# secure_api.py — Production-ready API security patterns
from flask import Flask, request, jsonify, abort
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from marshmallow import Schema, fields, validate, ValidationError
from functools import wraps
import jwt
import re

app = Flask(__name__)

# ─── RATE LIMITING ───────────────────────────────────────────
limiter = Limiter(
    app=app,
    key_func=get_remote_address,
    default_limits=["100 per hour", "10 per minute"],
    storage_uri="redis://localhost:6379"
)

# ─── INPUT VALIDATION SCHEMA ─────────────────────────────────
class CreateUserSchema(Schema):
    """Strict validation — reject anything unexpected"""
    email = fields.Email(required=True)
    name = fields.String(
        required=True,
        validate=[
            validate.Length(min=2, max=100),
            validate.Regexp(r'^[a-zA-Z\s\-]+$', error="Name contains invalid characters")
        ]
    )
    age = fields.Integer(validate=validate.Range(min=13, max=150))

# ─── AUTHENTICATION MIDDLEWARE ─────────────────────────────────
SECRET_KEY = "loaded-from-env-or-vault"  # Never hardcode!

def require_auth(f):
    """Verify JWT token on protected endpoints"""
    @wraps(f)
    def decorated(*args, **kwargs):
        auth_header = request.headers.get("Authorization", "")
        
        if not auth_header.startswith("Bearer "):
            return jsonify({"error": "Missing authorization header"}), 401
        
        token = auth_header[7:]  # Remove "Bearer " prefix
        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
            request.user = payload
        except jwt.ExpiredSignatureError:
            return jsonify({"error": "Token expired"}), 401
        except jwt.InvalidTokenError:
            return jsonify({"error": "Invalid token"}), 401
        
        return f(*args, **kwargs)
    return decorated

# ─── AUTHORIZATION ─────────────────────────────────────────────
def require_role(role):
    """Check if authenticated user has required role"""
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            user_roles = request.user.get("roles", [])
            if role not in user_roles:
                return jsonify({"error": "Insufficient permissions"}), 403
            return f(*args, **kwargs)
        return wrapper
    return decorator

# ─── SECURE ENDPOINTS ─────────────────────────────────────────

@app.route("/api/v1/users", methods=["POST"])
@limiter.limit("5 per minute")  # Stricter limit for user creation
@require_auth
@require_role("admin")
def create_user():
    """Create user — validates input, checks auth, rate limited"""
    # Input validation
    schema = CreateUserSchema()
    try:
        data = schema.load(request.json)
    except ValidationError as err:
        return jsonify({"errors": err.messages}), 400
    
    # Business logic (data is validated and safe)
    user = create_user_in_db(data)
    
    # Return ONLY necessary fields (minimize data exposure)
    return jsonify({
        "id": user.id,
        "email": user.email,
        "name": user.name,
    }), 201

@app.route("/api/v1/users/<int:user_id>", methods=["GET"])
@require_auth
def get_user(user_id):
    """Get user — ownership check (can only view own data)"""
    # Authorization: users can only view their own profile
    if request.user["sub"] != str(user_id) and "admin" not in request.user.get("roles", []):
        return jsonify({"error": "Forbidden"}), 403
    
    user = get_user_from_db(user_id)
    if not user:
        # Don't reveal whether user exists or not
        return jsonify({"error": "Not found"}), 404
    
    return jsonify({"id": user.id, "email": user.email, "name": user.name})

# ─── ERROR HANDLING ─────────────────────────────────────────────
@app.errorhandler(500)
def internal_error(e):
    """Never expose internal details in error responses"""
    # Log the real error internally
    app.logger.error(f"Internal error: {e}")
    # Return generic message to client
    return jsonify({"error": "Internal server error"}), 500

@app.errorhandler(404)
def not_found(e):
    return jsonify({"error": "Resource not found"}), 404
```

### Java — Secure REST API with Spring Boot

```java
// SecureApiController.java
@RestController
@RequestMapping("/api/v1")
public class SecureApiController {

    // Input validation with Bean Validation
    @PostMapping("/users")
    @PreAuthorize("hasRole('ADMIN')")  // Authorization check
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        
        User user = userService.create(request);
        
        // Return only necessary fields (DTO pattern)
        return ResponseEntity.status(201).body(
            new UserResponse(user.getId(), user.getEmail(), user.getName())
        );
    }

    @GetMapping("/users/{id}")
    @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(user -> ResponseEntity.ok(new UserResponse(user)))
            .orElse(ResponseEntity.notFound().build());
    }
}

// CreateUserRequest.java — Input validation
public record CreateUserRequest(
    @NotBlank @Email 
    String email,
    
    @NotBlank @Size(min = 2, max = 100) 
    @Pattern(regexp = "^[a-zA-Z\\s\\-]+$", message = "Invalid characters")
    String name,
    
    @Min(13) @Max(150)
    Integer age
) {}

// RateLimitConfig.java — Rate limiting with Bucket4j
@Configuration
public class RateLimitConfig {
    
    @Bean
    public FilterRegistrationBean<RateLimitFilter> rateLimitFilter() {
        FilterRegistrationBean<RateLimitFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new RateLimitFilter());
        bean.addUrlPatterns("/api/*");
        return bean;
    }
}
```

### API Key Authentication Pattern

```python
# api_key_auth.py — For service-to-service or public APIs
import hashlib
import secrets
from datetime import datetime

class ApiKeyManager:
    """Manage API keys with scopes and rate limits"""
    
    def generate_key(self, client_name: str, scopes: list) -> dict:
        """Generate a new API key for a client"""
        # Generate random key
        raw_key = secrets.token_urlsafe(32)
        
        # Store HASH of key (never store raw key!)
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        
        key_record = {
            "key_hash": key_hash,
            "prefix": raw_key[:8],  # For identification in logs
            "client": client_name,
            "scopes": scopes,
            "rate_limit": 1000,  # requests per hour
            "created_at": datetime.utcnow(),
        }
        
        # Store in database
        save_to_db(key_record)
        
        # Return raw key ONCE — client must save it
        return {"api_key": raw_key, "prefix": raw_key[:8]}
    
    def validate_key(self, raw_key: str) -> dict | None:
        """Validate an API key and return client info"""
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        record = find_by_hash(key_hash)
        
        if not record:
            return None
        if record.get("revoked"):
            return None
        
        return record

# Usage in middleware:
def api_key_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        api_key = request.headers.get("X-API-Key")
        if not api_key:
            return jsonify({"error": "API key required"}), 401
        
        client = api_key_manager.validate_key(api_key)
        if not client:
            return jsonify({"error": "Invalid API key"}), 401
        
        request.client = client
        return f(*args, **kwargs)
    return decorated
```

---

## Infrastructure Examples

### Rate Limiting with Redis (Sliding Window)

```python
# rate_limiter.py — Redis-based sliding window rate limiter
import redis
import time

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def is_rate_limited(client_id: str, limit: int = 100, window: int = 3600) -> bool:
    """
    Sliding window rate limiter.
    Returns True if the client has exceeded their limit.
    """
    key = f"rate_limit:{client_id}"
    now = time.time()
    window_start = now - window
    
    pipe = redis_client.pipeline()
    
    # Remove old entries outside the window
    pipe.zremrangebyscore(key, 0, window_start)
    
    # Count entries in current window
    pipe.zcard(key)
    
    # Add current request
    pipe.zadd(key, {f"{now}": now})
    
    # Set expiry on the key
    pipe.expire(key, window)
    
    results = pipe.execute()
    request_count = results[1]
    
    return request_count >= limit
```

### Nginx — API Security Configuration

```nginx
# Rate limiting at Nginx level (before hitting your app)
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
limit_req_zone $http_x_api_key zone=api_key:10m rate=100r/s;

server {
    listen 443 ssl;
    server_name api.example.com;

    # Block common attack patterns
    location /api/ {
        # Rate limiting
        limit_req zone=api burst=20 nodelay;
        
        # Limit request body size (prevent large payload attacks)
        client_max_body_size 1m;
        
        # Timeout settings
        proxy_read_timeout 30s;
        proxy_connect_timeout 5s;
        
        # Hide server info
        proxy_hide_header X-Powered-By;
        proxy_hide_header Server;
        
        # Security headers
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-Frame-Options "DENY" always;
        add_header Cache-Control "no-store" always;  # Don't cache API responses
        
        proxy_pass http://backend:8080;
    }
    
    # Block access to sensitive paths
    location ~ /\.(git|env|htaccess) {
        deny all;
        return 404;
    }
}
```

### API Response — What to Return (and What NOT to)

```python
# ❌ BAD — Leaks internal details
{
    "user": {
        "id": 42,
        "email": "alice@example.com",
        "password_hash": "$2b$12$...",         # NEVER expose!
        "internal_notes": "VIP customer",       # Internal data
        "social_security": "123-45-6789",       # PII not needed
        "database_id": "ObjectId('abc...')",    # Internal ID
        "created_by_employee": "john@corp.com"  # Internal info
    },
    "debug": {"sql_query": "SELECT * FROM...", "time_ms": 23}  # Debug info!
}

# ✓ GOOD — Minimal, safe response
{
    "user": {
        "id": 42,
        "email": "alice@example.com",
        "name": "Alice Smith"
    }
}
```

---

## Real-World Example

### How Stripe Secures Its API

```
┌──────────────────────────────────────────────────────────────┐
│                STRIPE API SECURITY MODEL                       │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Authentication: API keys (publishable + secret keys)      │
│     - Publishable key: pk_live_... (safe for frontend)        │
│     - Secret key: sk_live_... (server-only, never expose!)    │
│                                                               │
│  2. HTTPS only — HTTP requests rejected entirely              │
│                                                               │
│  3. Idempotency keys — Prevents duplicate charges             │
│     Header: Idempotency-Key: unique-request-id                │
│                                                               │
│  4. Webhook signatures — Verify events are from Stripe        │
│     Stripe-Signature: t=timestamp,v1=signature                │
│                                                               │
│  5. Restricted keys — Create keys with limited permissions    │
│     "This key can only create charges, nothing else"          │
│                                                               │
│  6. Rate limiting — 100 requests/second per key               │
│     Returns: 429 + Retry-After header                         │
│                                                               │
│  7. Request logging — Every API call logged and auditable     │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### How GitHub API Uses Scopes

```
Token with scope "repo":
  ✓ Read/write code
  ✓ Read/write issues
  ✓ Read/write pull requests
  
Token with scope "read:user":
  ✓ Read profile info
  ✗ Cannot modify anything
  
Fine-grained PAT:
  ✓ Access ONLY specific repositories
  ✓ Read-only on some, write on others
  ✓ Expires automatically
```

---

## Common Mistakes / Pitfalls

| Mistake | Risk | Fix |
|---------|------|-----|
| API key in URL query params | Logged in server access logs, browser history | Use `Authorization` header |
| No input size limits | DoS via huge payloads | Set `Content-Length` limits |
| Verbose error messages | Info disclosure | Generic errors + internal logging |
| Sequential/predictable IDs | Enumeration attacks | Use UUIDs or random IDs |
| No pagination limits | Memory exhaustion | Max page size (e.g., 100) |
| Returning all fields | Data leak | Use DTOs / response schemas |
| Rate limiting by IP only | Bypassed with rotating IPs | Limit by API key + IP |
| Not validating Content-Type | Unexpected data formats | Check `Content-Type` header |

### Error Response Anti-Patterns

```python
# ❌ BAD — Information disclosure
{
    "error": "PostgreSQL error: relation 'users' column 'ssn' does not exist",
    "stack_trace": "File /app/models.py, line 42...",
    "query": "SELECT * FROM users WHERE id = '1 OR 1=1'"
}

# ✓ GOOD — Safe error response
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid request parameters",
        "details": [
            {"field": "email", "message": "Must be a valid email address"}
        ]
    },
    "request_id": "req_abc123"  # For support/debugging
}
```

---

## When to Use / When NOT to Use

### Authentication Method Decision:

| Scenario | Method |
|----------|--------|
| Public API for developers | API keys + OAuth for user data |
| First-party mobile/web app | OAuth 2.0 + refresh tokens |
| Service-to-service (internal) | mTLS or OAuth Client Credentials |
| Webhooks (inbound) | HMAC signature verification |
| Serverless/Lambda | IAM roles + short-lived tokens |

### Rate Limiting Strategy:

| API Type | Limit |
|----------|-------|
| Free tier | 100 requests/hour |
| Paid tier | 10,000 requests/hour |
| Authentication endpoints | 5 per minute (prevent brute force) |
| Search/expensive queries | 30 per minute |
| Webhooks/callbacks | 1000 per hour |

---

## Key Takeaways

- **Always use HTTPS** — there's no valid reason for HTTP APIs in production
- **Validate ALL input** server-side — never trust the client, even your own frontend
- **Return minimal data** — only include fields the client actually needs (DTO pattern)
- **Rate limit by multiple dimensions** — IP + API key + user ID for best protection
- **Use proper HTTP status codes** — 401 (not authenticated), 403 (not authorized), 429 (rate limited)
- **Never expose internal errors** — log details internally, return generic messages to clients
- **Implement idempotency** for create/payment operations (Chapter 12.7)
- **Version your API** to safely deprecate insecure endpoints (Chapter 8.5)

---

## What's Next?

Next, we'll explore **Secrets Management** — how to securely store and access passwords, API keys, database credentials, and encryption keys using tools like HashiCorp Vault and AWS Secrets Manager. That's Chapter 14.8.
