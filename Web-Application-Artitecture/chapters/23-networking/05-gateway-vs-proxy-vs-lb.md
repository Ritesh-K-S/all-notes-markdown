# API Gateway vs Reverse Proxy vs Load Balancer — What's the Difference?

> **What you'll learn**: The distinct roles of API Gateways, Reverse Proxies, and Load Balancers — where they overlap, where they differ, and how to decide which combination your architecture needs.

---

## Real-Life Analogy

Imagine a **luxury hotel**:

**Load Balancer = The Parking Valet**
- Cars (requests) arrive at the hotel entrance
- The valet's ONLY job: direct each car to an available parking spot (server)
- He doesn't care who you are, what you're wearing, or why you're here
- He just distributes cars evenly across available spaces
- "Go to spot A... next car go to spot B... next car go to spot C..."

**Reverse Proxy = The Doorman / Concierge**
- Stands at the front door and handles incoming guests
- Checks if you look presentable (SSL termination, basic security)
- Might hand you a cached brochure (caching)
- Directs you to the right area (routing)
- Compresses your luggage (response compression)
- The hotel's internals are hidden from you (hides backend)

**API Gateway = The Hotel Receptionist**
- Checks your ID and reservation (authentication)
- Verifies your room access (authorization)
- Tracks how many towels you've requested (rate limiting)
- Translates your requests ("I need room service" → calls kitchen, housekeeping)
- Converts your foreign currency (request/response transformation)
- Logs everything for billing (analytics, metering)
- Knows all hotel services and their rules (API management)

```
                    Overlap Diagram:

    ┌───────────────────────────────────────────────┐
    │              LOAD BALANCER                     │
    │  • Distribute traffic                         │
    │  • Health checks                              │
    │  • High availability                          │
    │         ┌─────────────────────────────┐       │
    │         │       REVERSE PROXY         │       │
    │         │  • SSL termination          │       │
    │         │  • Caching                  │       │
    │         │  • Compression              │       │
    │         │  • URL routing              │       │
    │         │       ┌─────────────────┐   │       │
    │         │       │  API GATEWAY    │   │       │
    │         │       │  • Auth/AuthZ   │   │       │
    │         │       │  • Rate limiting│   │       │
    │         │       │  • API versioning│  │       │
    │         │       │  • Transformation│  │       │
    │         │       │  • Analytics    │   │       │
    │         │       └─────────────────┘   │       │
    │         └─────────────────────────────┘       │
    └───────────────────────────────────────────────┘

    API Gateway ⊃ Reverse Proxy ⊃ Load Balancer
    (most features)              (fewest features)
```

---

## Core Concept Explained Step-by-Step

### Step 1: Load Balancer — Pure Traffic Distribution

A **Load Balancer** has ONE primary job: **distribute incoming traffic across multiple servers**.

```
What a Load Balancer does:

  Client Requests
       │ │ │ │ │
       ▼ ▼ ▼ ▼ ▼
  ┌─────────────────┐
  │  LOAD BALANCER  │
  │                 │
  │  Algorithm:     │
  │  • Round Robin  │
  │  • Least Conn   │
  │  • IP Hash      │
  └───────┬─────────┘
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
  Srv 1  Srv 2  Srv 3

Core responsibilities:
  ✓ Distribute traffic evenly
  ✓ Health check servers (remove dead ones)
  ✓ High availability (if one server dies, others handle traffic)
  
Does NOT do:
  ✗ Authentication
  ✗ Request transformation
  ✗ Rate limiting
  ✗ API versioning
  ✗ Response caching
```

**Types:**
- **L4 Load Balancer** — Works at TCP level. Doesn't look at HTTP content. Fastest.
- **L7 Load Balancer** — Works at HTTP level. Can route based on URL/headers. More flexible.

### Step 2: Reverse Proxy — Traffic Management + Extras

A **Reverse Proxy** sits in front of servers and adds several capabilities beyond just distribution.

```
What a Reverse Proxy does:

  Client (HTTPS)
       │
       ▼
  ┌─────────────────────────────┐
  │       REVERSE PROXY          │
  │                              │
  │  1. SSL Termination          │ (decrypt HTTPS → HTTP)
  │  2. Caching                  │ (serve cached responses)
  │  3. Compression              │ (gzip responses)
  │  4. URL Routing              │ (/api → backend, /static → CDN)
  │  5. Load Balancing           │ (distribute across servers)
  │  6. Request Buffering        │ (absorb slow clients)
  │  7. Security Headers         │ (add HSTS, CSP, etc.)
  │  8. WebSocket Support        │ (protocol upgrade)
  │                              │
  └───────────┬─────────────────┘
              │
        ┌─────┼─────┐
        ▼     ▼     ▼
      Srv 1  Srv 2  Srv 3 (HTTP, internal)

Note: A reverse proxy CAN do load balancing, but it's not ONLY a load balancer.
It adds management, security, and optimization features.
```

**Tools:** Nginx, HAProxy, Envoy, Traefik, Caddy

### Step 3: API Gateway — Full API Lifecycle Management

An **API Gateway** is a reverse proxy on steroids — specifically designed for managing APIs.

```
What an API Gateway does:

  Client
       │
       ▼
  ┌─────────────────────────────────────┐
  │           API GATEWAY                │
  │                                      │
  │  Everything a reverse proxy does     │
  │  PLUS:                               │
  │                                      │
  │  1. Authentication                   │ (validate JWT, API keys)
  │  2. Authorization                    │ (check permissions/scopes)
  │  3. Rate Limiting                    │ (100 req/min per user)
  │  4. Request Transformation           │ (modify headers, body)
  │  5. Response Transformation          │ (filter fields, change format)
  │  6. API Versioning                   │ (/v1/users → service A, /v2/users → service B)
  │  7. Request Aggregation              │ (combine multiple backend calls)
  │  8. Analytics & Monitoring           │ (usage metrics per API)
  │  9. Developer Portal                 │ (API docs, key management)
  │  10. Protocol Translation            │ (REST → gRPC, SOAP → REST)
  │  11. Circuit Breaking                │ (stop calling failing services)
  │  12. Request Validation              │ (schema validation)
  │                                      │
  └───────────────┬─────────────────────┘
                  │
          ┌───────┼────────┐
          ▼       ▼        ▼
      User Svc  Order Svc  Payment Svc
      (gRPC)    (REST)     (REST)
```

**Tools:** Kong, AWS API Gateway, Apigee, Zuul, Spring Cloud Gateway, Tyk

### Step 4: Side-by-Side Comparison

```
┌──────────────────────┬──────────────┬───────────────┬──────────────────┐
│ Feature              │ Load Balancer│ Reverse Proxy │ API Gateway      │
├──────────────────────┼──────────────┼───────────────┼──────────────────┤
│ Traffic distribution │ ✅ Primary    │ ✅ Yes         │ ✅ Yes            │
│ Health checks        │ ✅ Yes        │ ✅ Yes         │ ✅ Yes            │
│ SSL termination      │ ⚠️ Some (L7) │ ✅ Yes         │ ✅ Yes            │
│ Caching              │ ❌ No         │ ✅ Yes         │ ✅ Yes            │
│ Compression          │ ❌ No         │ ✅ Yes         │ ✅ Yes            │
│ URL routing          │ ⚠️ Basic (L7)│ ✅ Yes         │ ✅ Advanced       │
│ Authentication       │ ❌ No         │ ⚠️ Basic      │ ✅ Full (JWT,OAuth)│
│ Authorization        │ ❌ No         │ ❌ No          │ ✅ Yes            │
│ Rate limiting        │ ❌ No         │ ⚠️ Basic      │ ✅ Advanced       │
│ Request transform    │ ❌ No         │ ⚠️ Headers    │ ✅ Full body/header│
│ API versioning       │ ❌ No         │ ❌ No          │ ✅ Yes            │
│ Analytics/metrics    │ ⚠️ Basic     │ ⚠️ Logs       │ ✅ Detailed       │
│ Developer portal     │ ❌ No         │ ❌ No          │ ✅ Yes            │
│ Protocol translation │ ❌ No         │ ❌ No          │ ✅ Yes            │
│ Request aggregation  │ ❌ No         │ ❌ No          │ ✅ Yes            │
│ Circuit breaking     │ ❌ No         │ ⚠️ Retry only │ ✅ Yes            │
│ Schema validation    │ ❌ No         │ ❌ No          │ ✅ Yes            │
├──────────────────────┼──────────────┼───────────────┼──────────────────┤
│ Performance overhead │ Minimal      │ Low           │ Medium-High      │
│ Complexity           │ Low          │ Medium        │ High             │
│ Latency added        │ ~0.1-0.5ms   │ ~0.5-2ms      │ ~2-10ms          │
└──────────────────────┴──────────────┴───────────────┴──────────────────┘
```

### Step 5: How They Work Together in Production

```
Typical production architecture — ALL THREE working together:

Internet
    │
    ▼
┌──────────────────────┐
│  L4 Load Balancer    │  ← Pure TCP distribution (NLB)
│  (AWS NLB / F5)      │     Distributes to API Gateway instances
└──────────┬───────────┘
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
┌───────┐ ┌───────┐ ┌───────┐
│API GW │ │API GW │ │API GW │  ← API Gateway cluster (Kong/Apigee)
│ inst1 │ │ inst2 │ │ inst3 │     Auth, rate limiting, routing
└───┬───┘ └───┬───┘ └───┬───┘
    │         │         │
    └─────────┼─────────┘
              │
              ▼
┌──────────────────────┐
│  L7 Load Balancer    │  ← Application-level routing (ALB/Nginx)
│  (Nginx / ALB)       │     SSL termination, caching, path routing
└──────────┬───────────┘
           │
    ┌──────┼──────────────┐
    ▼      ▼              ▼
┌──────┐ ┌──────┐     ┌──────┐
│User  │ │Order │     │Pay   │  ← Backend microservices
│Svc   │ │Svc   │     │Svc   │
└──────┘ └──────┘     └──────┘
```

---

## How It Works Internally

### Request Flow Through All Three

```
Complete request lifecycle: Client → All three components → Backend

1. CLIENT sends: GET https://api.shop.com/v2/orders/123
         │
         ▼
2. DNS resolves api.shop.com → Load Balancer IP (Anycast)
         │
         ▼
3. ┌── L4 LOAD BALANCER (NLB) ─────────────────────────┐
   │ • Receives TCP connection                          │
   │ • Selects one of 3 API Gateway instances           │
   │ • Algorithm: Least connections                     │
   │ • Does NOT inspect HTTP content (fast!)            │
   │ • Adds: no headers (transparent at TCP level)      │
   └────────────────────┬──────────────────────────────┘
                        │
                        ▼
4. ┌── API GATEWAY (Kong) ──────────────────────────────┐
   │ • Terminates SSL → reads HTTP request              │
   │ • Authenticates: Validates JWT token               │
   │   → Token expired? Return 401 immediately          │
   │ • Authorizes: Does user have "read:orders" scope?  │
   │   → No permission? Return 403                      │
   │ • Rate limit check: Has user exceeded 100 req/min? │
   │   → Over limit? Return 429                         │
   │ • Route matching: /v2/orders/* → order-service     │
   │ • Transform: Add X-User-Id header from JWT claims  │
   │ • Log: Record API call for analytics               │
   └────────────────────┬──────────────────────────────┘
                        │
                        ▼
5. ┌── REVERSE PROXY / L7 LB (Nginx/Envoy) ────────────┐
   │ • Receives pre-authenticated request               │
   │ • Checks cache: Is this response cached?           │
   │   → Cache hit? Return immediately (skip backend)   │
   │ • Load balance: Select healthiest order-service    │
   │ • Forward: proxy_pass to order-service:8080        │
   │ • Compress: gzip the response                      │
   │ • Cache: Store response for next time              │
   └────────────────────┬──────────────────────────────┘
                        │
                        ▼
6. ┌── BACKEND SERVICE (order-service) ─────────────────┐
   │ • Trusts X-User-Id header (came from API GW)      │
   │ • Queries database for order #123                  │
   │ • Returns JSON response                            │
   └────────────────────────────────────────────────────┘
```

### What Each Component Adds to the Request

```
Headers progression through the stack:

Original client request:
  GET /v2/orders/123 HTTP/1.1
  Host: api.shop.com
  Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...

After L4 Load Balancer (passes through unchanged):
  GET /v2/orders/123 HTTP/1.1
  Host: api.shop.com
  Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...

After API Gateway (enriches the request):
  GET /orders/123 HTTP/1.1              ← Path rewritten (removed /v2)
  Host: order-service.internal
  X-User-Id: user_12345                 ← Extracted from JWT
  X-Request-Id: abc-123-def            ← Added for tracing
  X-RateLimit-Remaining: 87            ← Rate limit info
  # Authorization header REMOVED (already validated)

After Reverse Proxy (routing headers):
  GET /orders/123 HTTP/1.1
  Host: order-service.internal
  X-User-Id: user_12345
  X-Request-Id: abc-123-def
  X-Real-IP: 203.0.113.50              ← Client's real IP
  X-Forwarded-For: 203.0.113.50        ← Forwarding chain
  X-Forwarded-Proto: https             ← Original protocol
```

---

## Code Examples

### Python — Simple API Gateway Middleware

```python
# api_gateway.py — Demonstrates what an API Gateway does
# This is a simplified version showing key responsibilities

from flask import Flask, request, jsonify, g
from functools import wraps
import jwt
import time
import redis
from collections import defaultdict

app = Flask(__name__)
redis_client = redis.Redis(host='localhost', port=6379)

# 1. AUTHENTICATION middleware
def authenticate(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.headers.get('Authorization', '').replace('Bearer ', '')
        if not token:
            return jsonify({"error": "Missing token"}), 401
        
        try:
            # Validate JWT token
            payload = jwt.decode(token, 'secret-key', algorithms=['HS256'])
            g.user_id = payload['user_id']
            g.scopes = payload.get('scopes', [])
        except jwt.ExpiredSignatureError:
            return jsonify({"error": "Token expired"}), 401
        except jwt.InvalidTokenError:
            return jsonify({"error": "Invalid token"}), 401
        
        return f(*args, **kwargs)
    return decorated

# 2. RATE LIMITING middleware
def rate_limit(max_requests=100, window_seconds=60):
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            key = f"rate_limit:{g.user_id}"
            current = redis_client.incr(key)
            if current == 1:
                redis_client.expire(key, window_seconds)
            
            if current > max_requests:
                return jsonify({"error": "Rate limit exceeded"}), 429
            
            # Add rate limit headers to response
            g.rate_limit_remaining = max_requests - current
            return f(*args, **kwargs)
        return decorated
    return decorator

# 3. AUTHORIZATION check
def require_scope(scope):
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            if scope not in g.scopes:
                return jsonify({"error": f"Missing scope: {scope}"}), 403
            return f(*args, **kwargs)
        return decorated
    return decorator

# 4. REQUEST ROUTING (API Gateway routes to microservices)
import requests as http_client

SERVICES = {
    "users": "http://user-service:8001",
    "orders": "http://order-service:8002",
    "payments": "http://payment-service:8003",
}

@app.route('/api/v1/<service>/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
@authenticate
@rate_limit(max_requests=100, window_seconds=60)
def gateway_route(service, path):
    """Route requests to the appropriate microservice."""
    if service not in SERVICES:
        return jsonify({"error": "Service not found"}), 404
    
    # Forward request to backend service
    target_url = f"{SERVICES[service]}/{path}"
    
    # Transform request — add user context, remove auth header
    headers = {
        'X-User-Id': str(g.user_id),
        'X-Request-Id': request.headers.get('X-Request-Id', 'generated-id'),
        'Content-Type': request.content_type or 'application/json',
    }
    
    # Forward to backend
    resp = http_client.request(
        method=request.method,
        url=target_url,
        headers=headers,
        data=request.get_data(),
        timeout=30
    )
    
    # Transform response — add gateway headers
    response = jsonify(resp.json()) if resp.headers.get('content-type', '').startswith('application/json') else resp.content
    return response, resp.status_code

if __name__ == '__main__':
    app.run(port=8000)
```

### Java — Spring Cloud Gateway Configuration

```java
// GatewayConfig.java — Spring Cloud Gateway with all features
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.cloud.gateway.filter.ratelimit.RedisRateLimiter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            
            // Route 1: User service (with rate limiting)
            .route("user-service", r -> r
                .path("/api/v1/users/**")
                .filters(f -> f
                    // Authentication (JWT validation)
                    .filter(new JwtAuthFilter())
                    // Rate limiting (100 req/sec per user)
                    .requestRateLimiter(c -> c
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver()))
                    // Remove Authorization header before forwarding
                    .removeRequestHeader("Authorization")
                    // Add user context from JWT
                    .addRequestHeader("X-User-Id", "#{jwt.userId}")
                    // Strip /api/v1 prefix
                    .stripPrefix(2)
                    // Circuit breaker
                    .circuitBreaker(cb -> cb
                        .setName("user-service-cb")
                        .setFallbackUri("forward:/fallback/users"))
                )
                .uri("lb://user-service"))  // Load-balanced URI
            
            // Route 2: Order service (with request transformation)
            .route("order-service", r -> r
                .path("/api/v2/orders/**")
                .filters(f -> f
                    .filter(new JwtAuthFilter())
                    .requestRateLimiter(c -> c.setRateLimiter(redisRateLimiter()))
                    .stripPrefix(2)
                    // Response transformation: remove internal fields
                    .modifyResponseBody(String.class, String.class,
                        (exchange, body) -> removeInternalFields(body))
                )
                .uri("lb://order-service"))
            
            // Route 3: Legacy service (protocol translation)
            .route("legacy-soap", r -> r
                .path("/api/v1/legacy/**")
                .filters(f -> f
                    .filter(new RestToSoapTranslationFilter()))
                .uri("http://legacy-system:9090"))
            
            .build();
    }

    @Bean
    public RedisRateLimiter redisRateLimiter() {
        // 100 requests per second, burst of 200
        return new RedisRateLimiter(100, 200);
    }
}
```

---

## Infrastructure Examples

### Architecture Pattern 1: Small Startup (Combined)

```
When you're small, ONE tool does all three jobs:

Internet ──▶ [Nginx] ──▶ App Server 1
                       ──▶ App Server 2

Nginx acts as:
  • Load Balancer (upstream with round_robin)
  • Reverse Proxy (proxy_pass, caching, compression)
  • Basic API Gateway (rate limiting, auth_basic, path routing)

nginx.conf:
  upstream api { server app1:8080; server app2:8080; }
  limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
  
  location /api/ {
    limit_req zone=api burst=20;
    proxy_pass http://api;
    proxy_cache api_cache;
  }

This is FINE for:
  ✓ Early-stage startups
  ✓ < 1000 req/sec
  ✓ Single team
  ✓ Simple auth needs
```

### Architecture Pattern 2: Growing Company (Separate Concerns)

```
As you grow, separate the roles:

Internet
    │
    ▼
┌──────────────────────┐
│ AWS ALB              │  ← Load Balancer (managed, no maintenance)
│ (Application LB)     │     SSL termination, health checks
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Kong API Gateway     │  ← API Gateway (auth, rate limit, versioning)
│ (3 instances)        │     Handles API lifecycle
└──────────┬───────────┘
           │
    ┌──────┼────────┐
    ▼      ▼        ▼
┌──────┐┌──────┐┌──────┐
│Svc A ││Svc B ││Svc C │  ← Backend services
└──────┘└──────┘└──────┘

Good for:
  ✓ Multiple teams/services
  ✓ Need proper API management
  ✓ 1K-100K req/sec
  ✓ External API consumers (partners, mobile apps)
```

### Architecture Pattern 3: Enterprise / Planet-Scale (Full Stack)

```
At scale, every layer is distinct and horizontally scaled:

Internet (millions of users)
    │
    ▼
┌──────────────────────────┐
│ Global Load Balancer     │  ← GeoDNS / Anycast (Route 53, Cloudflare)
│ (multi-region routing)   │     Routes to nearest region
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│ Regional NLB (L4)        │  ← Network Load Balancer (AWS NLB)
│ (TCP distribution)       │     Ultra-fast, no HTTP parsing
└──────────┬───────────────┘
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
┌──────┐┌──────┐┌──────┐
│API GW││API GW││API GW│  ← API Gateway Cluster (Kong/Apigee)
│inst 1││inst 2││inst 3│     Auth, rate limiting, metering
└──┬───┘└──┬───┘└──┬───┘
   │       │       │
   └───────┼───────┘
           │
           ▼
┌──────────────────────────┐
│ Service Mesh (Envoy/Istio)│  ← Sidecar proxies on every pod
│ (per-service LB + mTLS)  │     Internal load balancing, retries
└──────────┬───────────────┘
           │
    ┌──────┼────────────┐
    ▼      ▼            ▼
┌──────┐┌──────┐    ┌──────┐
│Svc A ││Svc B │    │Svc N │  ← Hundreds of microservices
│(50   ││(20   │    │(100  │     Each with own scaling
│ pods)││ pods)│    │ pods)│
└──────┘└──────┘    └──────┘
```

### Comparison: When Each Tool is Used

```yaml
# docker-compose.yml — Showing all three in a single deployment

version: '3.8'

services:
  # LOAD BALANCER — distributes to API Gateway instances
  load-balancer:
    image: haproxy:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    # HAProxy config routes to kong-1, kong-2, kong-3

  # API GATEWAY — authentication, rate limiting, routing
  kong-1:
    image: kong:latest
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong-db
    expose:
      - "8000"
  
  kong-2:
    image: kong:latest
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong-db
    expose:
      - "8000"

  # REVERSE PROXY — caching, compression (per-service sidecar)
  # In Kubernetes, this would be Envoy sidecars
  nginx-cache:
    image: nginx:latest
    volumes:
      - ./nginx-cache.conf:/etc/nginx/nginx.conf
    # Caches responses from backend services

  # BACKEND SERVICES
  user-service:
    image: myapp/user-service:latest
    deploy:
      replicas: 3
    expose:
      - "8080"

  order-service:
    image: myapp/order-service:latest
    deploy:
      replicas: 5
    expose:
      - "8080"
```

---

## Real-World Example

### Amazon's Architecture

```
Amazon uses ALL THREE at different layers:

Customer Browser/App
        │
        ▼
┌─────────────────────────┐
│ AWS CloudFront (CDN)     │  ← Reverse Proxy + Cache (edge)
│ + AWS WAF               │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ AWS NLB/ALB             │  ← Load Balancer (per region)
│ (Network/App LB)        │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Internal API Gateway     │  ← API Gateway (custom-built)
│ (authentication,         │     Handles 1M+ TPS
│  rate limiting,          │
│  request routing)        │
└───────────┬─────────────┘
            │
     ┌──────┼────────────────┐
     ▼      ▼                ▼
 Product  Cart    Search    Payment
 Service  Service  Service  Service
 
Each service INTERNALLY also has:
  • Its own load balancer (internal ALB)
  • Service mesh (App Mesh / Envoy sidecars)
```

### Stripe's API Architecture

```
Stripe (payment processing) — API Gateway is CRITICAL:

Developer's App
      │ (API Key in header)
      ▼
┌───────────────────────────────────┐
│  Stripe API Gateway               │
│                                   │
│  • Validate API key               │ (every request authenticated)
│  • Check permissions              │ (can this key create charges?)
│  • Rate limit per API key         │ (100 req/sec per key)
│  • Idempotency key check          │ (prevent duplicate charges!)
│  • Request logging                │ (for audit trail)
│  • API version routing            │ (/v1 → current, /v2 → beta)
│  • PCI compliance boundary        │ (sensitive data stops here)
│                                   │
└────────────────┬──────────────────┘
                 │ (only tokenized data passes through)
                 ▼
          Internal Services

Why API Gateway is essential for Stripe:
  1. Every API call involves MONEY — must authenticate perfectly
  2. Idempotency prevents charging a customer twice
  3. PCI-DSS compliance requires a clear boundary
  4. Multiple API versions run simultaneously
  5. Rate limiting prevents abuse
```

### Uber — Service Mesh (Beyond Traditional LB)

```
Uber evolved from traditional LB → Service Mesh:

Before (2015):
  Client → Nginx LB → Monolith

After (2020+):
  Client → Edge Gateway → Envoy Mesh → 4000+ microservices

Each microservice pod:
┌──────────────────────────────────────┐
│  Pod                                  │
│  ┌──────────┐    ┌──────────────┐    │
│  │  Envoy   │───▶│  App         │    │
│  │  Sidecar │    │  Container   │    │
│  │          │    │              │    │
│  │ • mTLS   │    │  (business   │    │
│  │ • Retry  │    │   logic)     │    │
│  │ • LB     │    │              │    │
│  │ • Metrics│    │              │    │
│  └──────────┘    └──────────────┘    │
└──────────────────────────────────────┘

Envoy sidecar handles:
  • Load balancing (to other services)
  • Mutual TLS (encryption between services)
  • Retries and circuit breaking
  • Observability (metrics, tracing)
  
This is DIFFERENT from API Gateway:
  • API Gateway = North-South traffic (external → internal)
  • Service Mesh = East-West traffic (internal ↔ internal)
```

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using API Gateway for everything | Adds 5-10ms latency to EVERY request, even internal ones | Use API GW for external/north-south traffic only. Use service mesh for internal. |
| No load balancer in front of API Gateway | API Gateway becomes single point of failure | Always put an LB in front of your gateway cluster |
| Rate limiting only at application level | Each backend instance has different count | Rate limit at the gateway (single point of enforcement) |
| Running Nginx as "API Gateway" | Missing auth, rate limiting, analytics | If you need API management features, use a real API Gateway |
| One API Gateway for all services | Becomes bottleneck and blast radius | Consider one gateway per domain (BFF pattern) |
| Not caching at the reverse proxy | Every request hits backend (wasted compute) | Cache GET responses at proxy layer (cache-control headers) |
| Skipping the L4 load balancer | API Gateway instances unevenly loaded | Use NLB/HAProxy (L4) to distribute to gateway instances |

---

## When to Use / When NOT to Use

### Use Load Balancer Alone When:
- ✅ Single application with multiple instances (simple horizontal scaling)
- ✅ Internal service-to-service communication (within trusted network)
- ✅ TCP/non-HTTP workloads (databases, Redis, Kafka)
- ✅ You just need traffic distribution + health checks

### Use Reverse Proxy When:
- ✅ You need SSL termination in front of HTTP servers
- ✅ You want to cache responses at the proxy layer
- ✅ You need compression, static file serving, or WebSocket support
- ✅ Single team, simple routing, no complex API management

### Use API Gateway When:
- ✅ You have external API consumers (mobile apps, third-party developers)
- ✅ You need proper authentication/authorization at the edge
- ✅ Multiple microservices that clients need a unified entry point for
- ✅ You need rate limiting per user/API key
- ✅ You need API versioning, request transformation, or protocol translation
- ✅ You need analytics, metering, or billing per API call

### Decision Matrix:

```
Question: How complex is your setup?

Solo app, 1-3 servers?
  └── Nginx (acts as all three) ✓

Multiple services, single team?
  └── ALB + Nginx ✓ (LB + Reverse Proxy)

Multiple services, external API consumers?
  └── ALB + Kong + Nginx ✓ (LB + API GW + Reverse Proxy)

Hundreds of microservices?
  └── NLB + API Gateway + Service Mesh (Envoy/Istio) ✓
```

---

## Key Takeaways

1. **Load Balancer** = distributes traffic. That's it. Fast, simple, does one job well.
2. **Reverse Proxy** = load balancing + SSL + caching + compression + routing. A superset of load balancer.
3. **API Gateway** = reverse proxy + auth + rate limiting + transformation + analytics. A superset of reverse proxy.
4. In small systems, **one tool (Nginx) can do all three**. In large systems, you'll have distinct components for each role.
5. **External traffic** (north-south) flows through API Gateway. **Internal traffic** (east-west) uses service mesh or direct load balancers.
6. The **latency cost** increases as you add layers: LB (~0.1ms) < Reverse Proxy (~1ms) < API Gateway (~5ms). Only add what you actually need.
7. **Don't over-engineer**: If you have 2 services and 1 team, Nginx is probably all you need. Add Kong/Apigee when you have external consumers or complex auth requirements.

---

## What's Next?

This completes Part 23: Networking Deep Dive. You now understand the core networking components that connect everything in modern web architecture.

Next up is **Part 24: Real-World System Designs** — where we'll see how companies like Google, Amazon, Netflix, and Uber put ALL of these concepts together to build planet-scale systems.

→ [../24-real-world/01-google-search.md](../24-real-world/01-google-search.md)
