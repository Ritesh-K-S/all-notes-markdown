# Authentication & Authorization at the Gateway

> **What you'll learn**: How an API Gateway verifies who you are (authentication) and what you're allowed to do (authorization), why centralizing these at the gateway simplifies your entire system, and the common patterns used in production.

---

## Real-Life Analogy: The Nightclub Bouncer

Think of a nightclub with multiple dance floors, a VIP section, a rooftop bar, and a lounge.

**The bouncer at the front door** does two things:

1. **Authentication** — "Are you who you say you are?" (checks your ID)
2. **Authorization** — "Are you allowed in the VIP section?" (checks if your name is on the VIP list)

Without the bouncer, every bartender and DJ would need to check IDs themselves. That's slow, inconsistent, and error-prone.

In the same way, the **API Gateway** acts as the bouncer for your entire system. It verifies identity ONCE at the entrance, stamps a "verified" mark on the request, and then backend services can trust that anyone who arrives has already been checked.

```
Without centralized auth:          With auth at gateway:

Client ──▶ Service A (check auth)  Client ──▶ [GATEWAY: Auth] ──▶ Service A ✓
Client ──▶ Service B (check auth)                             ──▶ Service B ✓
Client ──▶ Service C (check auth)                             ──▶ Service C ✓

Each service duplicates auth logic  Auth done ONCE, services trust the gateway
```

---

## Core Concepts: Authentication vs Authorization

Before diving into implementation, let's be crystal clear on the difference:

| | Authentication (AuthN) | Authorization (AuthZ) |
|---|---|---|
| **Question** | "WHO are you?" | "WHAT can you do?" |
| **Analogy** | Showing your ID card | Checking your permissions badge |
| **Example** | Logging in with username/password | Checking if user can delete an order |
| **When it fails** | 401 Unauthorized | 403 Forbidden |
| **Happens** | First | After authentication |

```
Request Flow:

Client ──▶ Gateway
              │
              ▼
         ┌──────────────────┐
         │  AUTHENTICATION  │  "Who is this person?"
         │  Verify JWT /    │
         │  API Key / OAuth │
         └────────┬─────────┘
                  │ ✅ Identity confirmed
                  ▼
         ┌──────────────────┐
         │  AUTHORIZATION   │  "Can they do this action?"
         │  Check roles /   │
         │  permissions /   │
         │  policies        │
         └────────┬─────────┘
                  │ ✅ Permission granted
                  ▼
         Forward to backend service
```

---

## Authentication Methods at the Gateway

### 1. JWT (JSON Web Tokens) — The Most Popular

```
How JWT works at the gateway:

┌──────────┐    1. Login (username/password)    ┌──────────────┐
│  Client  │ ─────────────────────────────────▶ │  Auth Server │
│          │ ◀───────────────────────────────── │ (issues JWT) │
└──────────┘    2. Returns JWT token            └──────────────┘
     │
     │  3. Sends request with JWT in header
     │     Authorization: Bearer eyJhbGciO...
     ▼
┌─────────────────────────────────────────┐
│            API GATEWAY                   │
│                                         │
│  4. Extract JWT from Authorization header│
│  5. Verify signature (using public key) │
│  6. Check expiration (exp claim)        │
│  7. Extract user info (sub, roles)      │
│  8. Attach user info to request headers │
│                                         │
└────────────────────┬────────────────────┘
                     │
                     ▼  X-User-Id: 12345
                        X-User-Role: admin
                     │
                     ▼
              Backend Service
              (trusts gateway's headers)
```

**Why JWT is great at the gateway:**
- Gateway can verify the token **without calling another service** (stateless)
- The token contains user info (claims) — no database lookup needed
- The signature proves the token hasn't been tampered with

### 2. API Keys — Simple but Limited

```
Client sends: GET /api/products
              X-API-Key: sk_live_abc123def456

Gateway checks:
  ┌─────────────────────────────────┐
  │ API Key Store (Redis/DB)        │
  │                                 │
  │ sk_live_abc123... → {           │
  │   client: "partner-corp",       │
  │   tier: "premium",             │
  │   rate_limit: 10000/hour,      │
  │   allowed_routes: ["/products"] │
  │ }                               │
  └─────────────────────────────────┘
```

### 3. OAuth 2.0 Token Introspection

```
Client sends: GET /api/profile
              Authorization: Bearer opaque-token-xyz

Gateway process:
  ┌──────────┐                      ┌─────────────────┐
  │ Gateway  │ ──── Introspect ───▶ │  OAuth Server   │
  │          │ ◀─── {active: true,  │  (Keycloak,     │
  │          │       sub: "user1",  │   Auth0, etc.)  │
  │          │       scope: "read"} │                 │
  └──────────┘                      └─────────────────┘
```

> **Note:** Token introspection adds a network call for every request. Most production systems prefer JWT for performance and use introspection only for opaque tokens.

### 4. Mutual TLS (mTLS) — Service-to-Service

```
For internal service communication:

Service A ──── mTLS (both sides present certificates) ────▶ Gateway
                                                              │
Gateway verifies:                                             │
  • Client certificate is valid                               │
  • Certificate is not revoked                                │
  • CN (Common Name) matches allowed services                 │
                                                              ▼
                                                       Backend Service
```

---

## How It Works Internally

### JWT Verification Deep Dive

When the gateway receives a JWT, here's exactly what happens:

```
JWT Token Structure:
──────────────────────────────────────────────────
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIxMjM0NSIsInJ...

  Header          Payload              Signature
  ┌─────┐         ┌─────────────┐     ┌──────────┐
  │{    │         │{            │     │ HMAC or  │
  │ alg:│    .    │ sub: "123", │  .  │ RSA sig  │
  │ RS256│        │ role: "admin"│    │ of header│
  │}    │         │ exp: 17321..│     │ + payload│
  └─────┘         │}            │     └──────────┘
                  └─────────────┘
──────────────────────────────────────────────────

Gateway Verification Steps:

  1. Split token by "." → [header, payload, signature]
  2. Base64-decode header → get algorithm (e.g., RS256)
  3. Base64-decode payload → get claims
  4. Using the public key, verify signature matches header+payload
  5. Check "exp" claim > current time (not expired)
  6. Check "iss" claim = expected issuer
  7. Check "aud" claim = our service (audience)
  8. ✅ Token is valid — extract user info from claims
```

### Authorization Patterns

**Pattern 1: Role-Based Access Control (RBAC)**

```
Route configuration with roles:

/admin/*       → requires role: "admin"
/users/*/edit  → requires role: "admin" OR "manager"  
/users/*/view  → requires role: "admin" OR "manager" OR "user"
/public/*      → no role required (public)

Decision Matrix:
┌──────────────────┬───────┬─────────┬──────┬────────┐
│ Route            │ Admin │ Manager │ User │ Public │
├──────────────────┼───────┼─────────┼──────┼────────┤
│ GET /admin/dash  │  ✅   │   ❌    │  ❌  │   ❌   │
│ PUT /users/edit  │  ✅   │   ✅    │  ❌  │   ❌   │
│ GET /users/view  │  ✅   │   ✅    │  ✅  │   ❌   │
│ GET /public/docs │  ✅   │   ✅    │  ✅  │   ✅   │
└──────────────────┴───────┴─────────┴──────┴────────┘
```

**Pattern 2: Scope-Based (OAuth 2.0 Scopes)**

```
Token has scopes: ["read:users", "write:orders"]

Gateway checks:
  GET  /users/123     → needs "read:users"   → ✅ ALLOWED
  POST /orders        → needs "write:orders" → ✅ ALLOWED
  DELETE /users/123   → needs "delete:users" → ❌ FORBIDDEN (403)
```

**Pattern 3: Policy-Based (OPA — Open Policy Agent)**

```
Gateway ──── "Can user X do action Y on resource Z?" ────▶ OPA Server
       ◀──── { "allow": true/false, "reason": "..." } ────

OPA evaluates Rego policies:
  allow {
    input.method == "GET"
    input.path[0] == "users"
    input.user.department == "engineering"
  }
```

---

## Code Examples

### Python: JWT Verification at Gateway

```python
# gateway_auth.py
# JWT authentication middleware for an API Gateway

import jwt
import time
from functools import wraps
from flask import Flask, request, jsonify, g

app = Flask(__name__)

# In production, load from secure config / key management service
PUBLIC_KEY = """-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
-----END PUBLIC KEY-----"""

# Simpler setup: shared secret for HMAC (HS256)
JWT_SECRET = "your-256-bit-secret"  # Use env variable in production!


def authenticate(f):
    """Decorator that verifies JWT token before allowing access."""
    @wraps(f)
    def decorated(*args, **kwargs):
        # Step 1: Extract token from Authorization header
        auth_header = request.headers.get("Authorization")
        if not auth_header or not auth_header.startswith("Bearer "):
            return jsonify({"error": "Missing or invalid token"}), 401

        token = auth_header.split(" ")[1]

        try:
            # Step 2: Decode and verify the JWT
            payload = jwt.decode(
                token,
                JWT_SECRET,
                algorithms=["HS256"],
                options={
                    "verify_exp": True,      # Check expiration
                    "verify_iss": True,      # Check issuer
                    "require": ["exp", "sub", "roles"],  # Required claims
                },
                issuer="auth.myapp.com",     # Expected issuer
            )

            # Step 3: Attach user info to request context
            g.user_id = payload["sub"]
            g.user_roles = payload.get("roles", [])
            g.user_email = payload.get("email", "")

        except jwt.ExpiredSignatureError:
            return jsonify({"error": "Token expired"}), 401
        except jwt.InvalidTokenError as e:
            return jsonify({"error": f"Invalid token: {str(e)}"}), 401

        return f(*args, **kwargs)
    return decorated


def authorize(required_roles):
    """Decorator that checks if user has required roles."""
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            user_roles = getattr(g, "user_roles", [])
            
            # Check if user has ANY of the required roles
            if not any(role in user_roles for role in required_roles):
                return jsonify({
                    "error": "Forbidden",
                    "message": f"Requires one of: {required_roles}"
                }), 403

            return f(*args, **kwargs)
        return decorated
    return decorator


# Example routes with auth
@app.route("/api/users/<user_id>")
@authenticate
def get_user(user_id):
    """Any authenticated user can view profiles."""
    return jsonify({"user_id": user_id, "requested_by": g.user_id})


@app.route("/api/admin/dashboard")
@authenticate
@authorize(required_roles=["admin", "super_admin"])
def admin_dashboard():
    """Only admins can access the dashboard."""
    return jsonify({"message": "Welcome, admin!", "user": g.user_id})
```

### Java: JWT Authentication Filter for Spring Cloud Gateway

```java
// JwtAuthFilter.java
// Custom gateway filter that validates JWT tokens

import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;

@Component
public class JwtAuthFilter extends AbstractGatewayFilterFactory<JwtAuthFilter.Config> {

    private static final String SECRET = "your-256-bit-secret-key-here-min-32-chars!";

    public JwtAuthFilter() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // Step 1: Extract Authorization header
            String authHeader = exchange.getRequest().getHeaders()
                .getFirst("Authorization");

            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }

            String token = authHeader.substring(7);

            try {
                // Step 2: Parse and verify JWT
                Claims claims = Jwts.parserBuilder()
                    .setSigningKey(Keys.hmacShaKeyFor(SECRET.getBytes()))
                    .build()
                    .parseClaimsJws(token)
                    .getBody();

                // Step 3: Add user info as headers for downstream services
                exchange = exchange.mutate()
                    .request(r -> r
                        .header("X-User-Id", claims.getSubject())
                        .header("X-User-Roles", claims.get("roles", String.class))
                        .header("X-User-Email", claims.get("email", String.class))
                    )
                    .build();

            } catch (Exception e) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }

            // Step 4: Continue the filter chain
            return chain.filter(exchange);
        };
    }

    public static class Config {
        // Configuration properties (required roles, etc.)
        private String requiredRole;

        public String getRequiredRole() { return requiredRole; }
        public void setRequiredRole(String role) { this.requiredRole = role; }
    }
}
```

---

## Infrastructure Example: Kong API Gateway with JWT Plugin

```yaml
# kong.yml — Declarative configuration for Kong API Gateway

_format_version: "3.0"

services:
  - name: user-service
    url: http://user-service:8001
    routes:
      - name: user-routes
        paths:
          - /users
        strip_path: false

  - name: order-service
    url: http://order-service:8002
    routes:
      - name: order-routes
        paths:
          - /orders
        strip_path: false

plugins:
  # Global JWT authentication plugin
  - name: jwt
    config:
      uri_param_names:        # Where to look for the token
        - jwt
      header_names:
        - Authorization
      claims_to_verify:
        - exp                 # Verify expiration
      key_claim_name: iss     # Which claim identifies the consumer

  # Rate limiting per consumer
  - name: rate-limiting
    config:
      minute: 100
      hour: 5000
      policy: redis
      redis_host: redis-server

  # Access Control List (authorization)
  - name: acl
    service: user-service
    config:
      allow:
        - admin-group
        - user-group

consumers:
  - username: mobile-app
    jwt_secrets:
      - key: "mobile-app-issuer"
        secret: "shared-secret-for-mobile"
    acls:
      - group: user-group

  - username: admin-dashboard
    jwt_secrets:
      - key: "admin-dashboard-issuer"
        secret: "shared-secret-for-admin"
    acls:
      - group: admin-group
```

---

## Real-World Example

### How Uber Handles Auth at the Gateway

Uber's system processes millions of requests per second from riders, drivers, and internal services.

```
Uber's Authentication Architecture:

┌───────────┐     ┌───────────┐     ┌──────────────┐
│  Rider    │     │  Driver   │     │  Internal    │
│  App      │     │  App      │     │  Services    │
└─────┬─────┘     └─────┬─────┘     └──────┬───────┘
      │                  │                   │
      │  OAuth2 Token    │  OAuth2 Token     │  mTLS Certificate
      │                  │                   │
      └──────────────────┼───────────────────┘
                         │
                         ▼
          ┌──────────────────────────────┐
          │       EDGE GATEWAY           │
          │                              │
          │  External traffic:           │
          │  • Verify OAuth2 token       │
          │  • Check token scopes        │
          │  • Rate limit per user       │
          │  • Inject user context       │
          │                              │
          │  Internal traffic:           │
          │  • Verify mTLS certificate   │
          │  • Check service identity    │
          │  • Apply service-level ACLs  │
          └──────────────┬───────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │   User Context       │
              │   Propagation        │
              │                      │
              │   Headers added:     │
              │   X-Uber-UUID: abc   │
              │   X-Uber-Scope: ride │
              │   X-Uber-City: NYC   │
              └──────────┬───────────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
      ┌──────────┐ ┌──────────┐ ┌──────────┐
      │ Ride     │ │ Pricing  │ │ Maps     │
      │ Service  │ │ Service  │ │ Service  │
      └──────────┘ └──────────┘ └──────────┘
```

**Key lessons from Uber:**
- Auth tokens are **verified ONCE** at the edge gateway
- User context is **propagated as headers** to all downstream services
- Internal services use **mTLS** (mutual TLS) — no tokens needed between services
- They separate **external auth** (user-facing) from **internal auth** (service-to-service)

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Solution |
|---------|-------------|----------|
| Verifying tokens in every microservice | Duplicated logic, inconsistent implementations | Verify once at gateway, pass verified user info as headers |
| Not validating JWT expiration | Stolen tokens work forever | Always check `exp` claim; use short-lived tokens (15 min) |
| Storing secrets in code/config files | Secrets leak via git, logs, or config dumps | Use a secrets manager (Vault, AWS Secrets Manager) |
| No token refresh mechanism | Users get logged out frequently | Use refresh tokens with longer expiry |
| Using symmetric keys (HS256) at scale | Every service needs the shared secret | Use asymmetric keys (RS256) — only auth server has private key |
| Not revoking tokens on logout | Logged-out users still have valid tokens | Use a token blacklist in Redis, or short expiry + refresh |
| Trusting client-provided headers | Attackers can forge X-User-Id headers | Gateway should STRIP all internal headers from external requests |

---

## Security Best Practices

```
Security Layers at the Gateway:

┌─────────────────────────────────────────────────────────┐
│                    GATEWAY SECURITY                      │
│                                                         │
│  Layer 1: TLS Termination                              │
│  ├── Only accept HTTPS                                  │
│  ├── Minimum TLS 1.2                                   │
│  └── Strong cipher suites only                         │
│                                                         │
│  Layer 2: Input Validation                             │
│  ├── Strip unknown headers from external requests      │
│  ├── Validate Content-Type                             │
│  └── Reject oversized payloads                         │
│                                                         │
│  Layer 3: Authentication                               │
│  ├── Verify token signature                            │
│  ├── Check expiration + issuer + audience              │
│  └── Validate token hasn't been revoked                │
│                                                         │
│  Layer 4: Authorization                                │
│  ├── Check roles/scopes against route requirements     │
│  ├── Resource-level access (user can only edit own data)│
│  └── IP allowlisting for admin routes                  │
│                                                         │
│  Layer 5: Propagation                                  │
│  ├── Inject verified user context as trusted headers   │
│  ├── Sign the internal headers (HMAC)                  │
│  └── Log auth decisions for audit trail                │
└─────────────────────────────────────────────────────────┘
```

---

## When to Use / When NOT to Use

### ✅ Centralize Auth at Gateway When:

- You have **multiple services** that all need the same auth logic
- You want to **change auth strategy** without touching every service
- You need **consistent security policies** across all APIs
- You want a **single audit log** for all auth decisions
- Services are written in **different languages** (auth logic shouldn't be reimplemented)

### ❌ Cases Where Services Still Need Auth Logic:

- **Fine-grained authorization** — "Can this user edit THIS specific document?" (resource-level access control is better in the service that owns the data)
- **Service-to-service auth** — Internal services calling each other may use mTLS or service mesh
- **Sensitive operations** — Payment processing might double-check auth as defense-in-depth

> **Best practice:** Gateway handles *coarse-grained* auth (is this user authenticated? do they have the right role?). Services handle *fine-grained* auth (can this user access this specific resource?).

---

## Key Takeaways

- **Authentication = "Who are you?"** → verified via JWT, API key, or OAuth token at the gateway
- **Authorization = "What can you do?"** → checked via roles, scopes, or policy engines (OPA)
- **Verify once at the gateway**, then propagate user identity via trusted headers to backend services
- Use **asymmetric keys (RS256/ES256)** for JWT in production — only the auth server needs the private key
- **Never trust headers from external requests** — the gateway must strip and replace them
- Implement **token revocation** via blacklists or short expiry + refresh tokens
- Combine gateway auth (coarse-grained) with service auth (fine-grained) for defense-in-depth

---

## What's Next?

With authentication and authorization handled, the next critical gateway function is protecting your system from abuse. Next up: **Rate Limiting & Throttling** — how to prevent clients from overwhelming your services.

Next: [03-rate-limiting-throttling.md](./03-rate-limiting-throttling.md)
