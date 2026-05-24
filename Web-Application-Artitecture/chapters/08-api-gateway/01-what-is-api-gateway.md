# What is an API Gateway? Why Do We Need One?

> **What you'll learn**: How an API Gateway acts as the single entry point to your system, why every serious web application needs one, and how it simplifies communication between clients and backend services.

---

## Real-Life Analogy: The Hotel Reception Desk

Imagine you arrive at a large luxury hotel. The hotel has:
- A restaurant (on the 3rd floor)
- A spa (in the basement)
- A gym (on the 5th floor)
- Room service (different wing)
- A laundry service (back of the building)

Now, do you run around the building finding each service yourself? **No!** You go to the **reception desk**. The receptionist:

1. **Identifies you** (checks your room key/ID)
2. **Understands your request** ("I'd like to book a spa at 3 PM")
3. **Routes you** to the right service
4. **Handles translation** ("The spa uses a different booking system, let me enter it for you")
5. **Applies policies** ("Sorry, the spa is fully booked. I can put you on a waitlist.")

The **API Gateway** is your system's reception desk. Every request from every client goes through it first. It decides who gets in, where they go, and how they're served.

---

## The Problem: Life Without an API Gateway

Let's say you have a microservices architecture with 10 services:

```
Without API Gateway — Every client must know about every service:

┌──────────────┐
│  Mobile App  │──────▶ User Service (port 8001)
│              │──────▶ Order Service (port 8002)
│              │──────▶ Payment Service (port 8003)
│              │──────▶ Notification Service (port 8004)
│              │──────▶ Inventory Service (port 8005)
└──────────────┘

┌──────────────┐
│  Web App     │──────▶ User Service (port 8001)
│              │──────▶ Order Service (port 8002)
│              │──────▶ Payment Service (port 8003)
│              │──────▶ Notification Service (port 8004)
│              │──────▶ Inventory Service (port 8005)
└──────────────┘

┌──────────────┐
│  3rd Party   │──────▶ User Service (port 8001)
│  Partner     │──────▶ Order Service (port 8002)
└──────────────┘
```

**Problems with this approach:**

| Problem | Impact |
|---------|--------|
| Clients must know all service URLs | Tightly coupled, hard to change |
| No central place for auth | Each service implements its own auth |
| No rate limiting | One abusive client can crash your system |
| No single point for logging | Can't see all traffic in one place |
| SSL/TLS on every service | Operational nightmare |
| Cross-cutting concerns repeated | Every service handles CORS, compression, etc. |

---

## The Solution: API Gateway

```
With API Gateway — One entry point for everything:

┌──────────────┐         ┌─────────────────┐         ┌──────────────────┐
│  Mobile App  │────┐    │                 │────────▶│  User Service    │
└──────────────┘    │    │                 │         └──────────────────┘
                    │    │                 │         ┌──────────────────┐
┌──────────────┐    ├───▶│  API GATEWAY    │────────▶│  Order Service   │
│  Web App     │────┤    │                 │         └──────────────────┘
└──────────────┘    │    │  (Single Entry  │         ┌──────────────────┐
                    │    │   Point)        │────────▶│  Payment Service │
┌──────────────┐    │    │                 │         └──────────────────┘
│  3rd Party   │────┘    │                 │         ┌──────────────────┐
└──────────────┘         └─────────────────┘────────▶│  Notification    │
                                                     └──────────────────┘
```

Now ALL requests go through one place. The gateway handles:

- **Authentication** — "Is this user who they say they are?"
- **Authorization** — "Are they allowed to access this?"
- **Rate Limiting** — "Are they sending too many requests?"
- **Routing** — "Which backend service should handle this?"
- **Load Balancing** — "Which instance of that service is least busy?"
- **Logging & Monitoring** — "Record everything for observability"
- **SSL Termination** — "Handle HTTPS here, talk HTTP internally"
- **Request/Response Transformation** — "Convert this format for the client"

---

## Core Concept: What Exactly IS an API Gateway?

An **API Gateway** is a server that sits between clients and your backend services. It acts as a **reverse proxy** with superpowers — it doesn't just forward requests; it adds a layer of intelligence.

### The Key Responsibilities

```
┌─────────────────────────────────────────────────────────────────┐
│                        API GATEWAY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │   AUTH      │  │  RATE        │  │  REQUEST ROUTING      │  │
│  │   Check     │  │  LIMITING    │  │  /users → User Svc    │  │
│  │   JWT/OAuth │  │  100 req/min │  │  /orders → Order Svc  │  │
│  └─────────────┘  └──────────────┘  └───────────────────────┘  │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │  SSL/TLS    │  │  CACHING     │  │  LOGGING &            │  │
│  │  Terminate  │  │  Responses   │  │  MONITORING           │  │
│  └─────────────┘  └──────────────┘  └───────────────────────┘  │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │  LOAD       │  │  CIRCUIT     │  │  REQUEST/RESPONSE     │  │
│  │  BALANCING  │  │  BREAKING    │  │  TRANSFORMATION       │  │
│  └─────────────┘  └──────────────┘  └───────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Request Flow Through an API Gateway

```
Step-by-step: What happens when a request arrives

Client sends: GET https://api.myapp.com/users/123

  ┌─────────┐
  │ Step 1  │  SSL/TLS Termination
  │         │  Decrypt HTTPS → plain HTTP internally
  └────┬────┘
       │
       ▼
  ┌─────────┐
  │ Step 2  │  Authentication
  │         │  Verify JWT token / API key / OAuth token
  └────┬────┘
       │
       ▼
  ┌─────────┐
  │ Step 3  │  Rate Limiting
  │         │  Check: Has this client exceeded their quota?
  └────┬────┘
       │
       ▼
  ┌─────────┐
  │ Step 4  │  Request Validation
  │         │  Are required headers present? Is the body valid?
  └────┬────┘
       │
       ▼
  ┌─────────┐
  │ Step 5  │  Route Matching
  │         │  /users/* → user-service (at 10.0.1.5:8080)
  └────┬────┘
       │
       ▼
  ┌─────────┐
  │ Step 6  │  Load Balancing
  │         │  Pick one of the 5 user-service instances
  └────┬────┘
       │
       ▼
  ┌─────────┐
  │ Step 7  │  Request Transformation (optional)
  │         │  Add headers, modify path, change body format
  └────┬────┘
       │
       ▼
  ┌─────────┐
  │ Step 8  │  Forward to Backend Service
  │         │  Proxy the request to user-service instance
  └────┬────┘
       │
       ▼
  ┌─────────┐
  │ Step 9  │  Response Transformation (optional)
  │         │  Filter fields, change format, add CORS headers
  └────┬────┘
       │
       ▼
  ┌─────────┐
  │ Step 10 │  Logging & Metrics
  │         │  Record latency, status code, client info
  └────┬────┘
       │
       ▼
  Response sent back to client
```

### Internal Architecture of a Typical API Gateway

```
┌──────────────────────────────────────────────────────┐
│                   API GATEWAY SERVER                  │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │           Plugin/Filter Chain                │    │
│  │                                              │    │
│  │  Request ──▶ [Auth] ──▶ [RateLimit] ──▶     │    │
│  │             [Transform] ──▶ [Route] ──▶     │    │
│  │             [LoadBalance] ──▶ [Proxy]       │    │
│  └──────────────────────────────────────────────┘    │
│                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────┐   │
│  │ Route Table │  │ Service     │  │ Config     │   │
│  │ /users → A  │  │ Registry    │  │ Store      │   │
│  │ /orders → B │  │ (discovers  │  │ (rules,    │   │
│  │ /pay → C   │  │  backends)  │  │  policies) │   │
│  └─────────────┘  └─────────────┘  └────────────┘   │
│                                                      │
│  ┌─────────────────────────────────────────────┐     │
│  │          Connection Pool Manager            │     │
│  │  Keeps persistent connections to backends   │     │
│  └─────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────┘
```

Most API gateways use a **pipeline/filter chain** pattern internally. Each incoming request passes through a series of filters (plugins), each doing one job. This is similar to middleware in Express.js or Django.

---

## Code Examples

### Python: Simple API Gateway with Flask

```python
# simple_api_gateway.py
# A minimal API Gateway that routes requests to different backend services

from flask import Flask, request, jsonify
import requests
import time

app = Flask(__name__)

# Service registry — maps URL prefixes to backend service URLs
SERVICE_REGISTRY = {
    "/users": "http://localhost:8001",
    "/orders": "http://localhost:8002",
    "/payments": "http://localhost:8003",
}

# Simple in-memory rate limiter
rate_limit_store = {}  # {client_ip: [timestamps]}
RATE_LIMIT = 100  # requests per minute


def check_rate_limit(client_ip):
    """Check if client has exceeded rate limit."""
    now = time.time()
    window_start = now - 60  # 1-minute window

    if client_ip not in rate_limit_store:
        rate_limit_store[client_ip] = []

    # Remove old timestamps outside the window
    rate_limit_store[client_ip] = [
        t for t in rate_limit_store[client_ip] if t > window_start
    ]

    if len(rate_limit_store[client_ip]) >= RATE_LIMIT:
        return False  # Rate limit exceeded

    rate_limit_store[client_ip].append(now)
    return True


def authenticate(req):
    """Verify the Authorization header contains a valid token."""
    auth_header = req.headers.get("Authorization")
    if not auth_header or not auth_header.startswith("Bearer "):
        return False
    # In production, verify JWT signature here
    token = auth_header.split(" ")[1]
    return len(token) > 0  # Simplified check


@app.route("/<path:path>", methods=["GET", "POST", "PUT", "DELETE"])
def gateway(path):
    """Main gateway handler — routes all requests."""
    # Step 1: Authentication
    if not authenticate(request):
        return jsonify({"error": "Unauthorized"}), 401

    # Step 2: Rate limiting
    client_ip = request.remote_addr
    if not check_rate_limit(client_ip):
        return jsonify({"error": "Rate limit exceeded"}), 429

    # Step 3: Route matching — find the right backend service
    target_service = None
    for prefix, service_url in SERVICE_REGISTRY.items():
        if f"/{path}".startswith(prefix):
            target_service = service_url
            break

    if not target_service:
        return jsonify({"error": "Service not found"}), 404

    # Step 4: Forward the request to the backend service
    backend_url = f"{target_service}/{path}"
    response = requests.request(
        method=request.method,
        url=backend_url,
        headers={k: v for k, v in request.headers if k != "Host"},
        data=request.get_data(),
        params=request.args,
    )

    # Step 5: Return the response to the client
    return (response.content, response.status_code, dict(response.headers))


if __name__ == "__main__":
    app.run(port=8000)  # Gateway listens on port 8000
```

### Java: Simple API Gateway with Spring Cloud Gateway

```java
// GatewayConfig.java
// Spring Cloud Gateway configuration for routing and filtering

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // Route /users/** to User Service
            .route("user-service", r -> r
                .path("/users/**")
                .filters(f -> f
                    .stripPrefix(0)                           // Keep original path
                    .addRequestHeader("X-Gateway", "true")   // Mark as gateway-routed
                    .retry(3)                                 // Retry up to 3 times
                )
                .uri("http://localhost:8001"))

            // Route /orders/** to Order Service
            .route("order-service", r -> r
                .path("/orders/**")
                .filters(f -> f
                    .stripPrefix(0)
                    .requestRateLimiter(c -> c
                        .setRateLimiter(redisRateLimiter())   // Redis-backed rate limiter
                    )
                )
                .uri("http://localhost:8002"))

            // Route /payments/** to Payment Service
            .route("payment-service", r -> r
                .path("/payments/**")
                .filters(f -> f
                    .circuitBreaker(c -> c                    // Circuit breaker for resilience
                        .setName("paymentCB")
                        .setFallbackUri("forward:/fallback/payments")
                    )
                )
                .uri("http://localhost:8003"))
            .build();
    }
}
```

---

## Infrastructure Example: Nginx as API Gateway

```nginx
# nginx.conf — Using Nginx as a basic API Gateway

upstream user_service {
    server 10.0.1.1:8001;
    server 10.0.1.2:8001;   # Multiple instances for load balancing
    server 10.0.1.3:8001;
}

upstream order_service {
    server 10.0.2.1:8002;
    server 10.0.2.2:8002;
}

# Rate limiting zone: 10 requests/second per IP
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    listen 443 ssl;
    server_name api.myapp.com;

    # SSL Termination
    ssl_certificate /etc/ssl/certs/myapp.crt;
    ssl_certificate_key /etc/ssl/private/myapp.key;

    # Route: /users/* → User Service
    location /users/ {
        limit_req zone=api_limit burst=20 nodelay;   # Rate limiting
        
        proxy_pass http://user_service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Route: /orders/* → Order Service
    location /orders/ {
        limit_req zone=api_limit burst=20 nodelay;
        
        proxy_pass http://order_service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Health check endpoint
    location /health {
        return 200 '{"status": "healthy"}';
        add_header Content-Type application/json;
    }
}
```

---

## Real-World Example

### How Netflix Uses API Gateway (Zuul)

Netflix processes **billions of API requests per day** from over 200 million subscribers on thousands of different device types (Smart TVs, phones, tablets, game consoles).

```
Netflix's API Gateway Architecture:

┌───────────┐  ┌───────────┐  ┌───────────┐
│ iPhone    │  │ Smart TV  │  │ Android   │
│ App       │  │ App       │  │ App       │
└─────┬─────┘  └─────┬─────┘  └─────┬─────┘
      │               │               │
      └───────────────┼───────────────┘
                      │
                      ▼
        ┌──────────────────────────┐
        │   ZUUL (API Gateway)     │
        │                          │
        │  • Auth verification     │
        │  • Device detection      │
        │  • Request routing       │
        │  • Canary testing        │
        │  • Load shedding         │
        │  • DDoS protection       │
        └─────────────┬────────────┘
                      │
          ┌───────────┼───────────┐
          │           │           │
          ▼           ▼           ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ User     │ │ Content  │ │ Streaming│
    │ Profile  │ │ Catalog  │ │ Service  │
    │ Service  │ │ Service  │ │          │
    └──────────┘ └──────────┘ └──────────┘
```

**Why Netflix built Zuul:**
- They needed **dynamic routing** — changing which backend handles requests without redeploying
- **Canary testing** — routing 1% of traffic to a new version to test it
- **Load shedding** — when overwhelmed, gracefully reject some requests
- Each device type gets a **different response format** (a Smart TV gets less data than a phone)

### How Amazon Uses API Gateway

Amazon's API Gateway handles requests from:
- amazon.com (the website)
- Alexa devices
- Mobile apps
- Third-party sellers
- AWS services

They route to over **1,000 microservices** behind a single gateway, handling authentication, rate limiting per seller, and response aggregation.

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Solution |
|---------|-------------|----------|
| Making the gateway a single point of failure | If gateway dies, everything dies | Deploy multiple gateway instances behind a load balancer |
| Putting business logic in the gateway | Gateway becomes bloated and hard to maintain | Gateway should only handle cross-cutting concerns |
| No caching at the gateway | Every request hits backend even for static data | Cache common GET responses at the gateway |
| Ignoring gateway latency | Gateway adds network hop overhead | Monitor gateway p99 latency, keep processing lightweight |
| Not having circuit breakers | One slow backend cascades failures to all clients | Add circuit breakers for each backend route |
| Hardcoding service URLs | Can't scale or move services | Use service discovery (Consul, Eureka) |
| Not rate-limiting internal services | A runaway internal service can overwhelm others | Rate limit even service-to-service traffic |

---

## When to Use / When NOT to Use

### ✅ Use an API Gateway When:

- You have **multiple backend services** (microservices)
- You need **centralized authentication** across services
- You serve **multiple client types** (web, mobile, IoT, partners)
- You want a **single entry point** for monitoring and logging
- You need **rate limiting** to protect your services
- You need to **version your APIs** and manage transitions

### ❌ You Might NOT Need an API Gateway When:

- You have a **single monolithic** backend (just use a reverse proxy like Nginx)
- Your application is **purely internal** with no external clients
- You have **very few services** (2-3) and simple routing needs
- You're building a **prototype/MVP** — keep it simple at first
- Added latency is **unacceptable** (some ultra-low-latency trading systems avoid it)

---

## API Gateway vs Other Patterns

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Comparison Table                                      │
├──────────────────┬─────────────────┬─────────────────┬─────────────────┤
│                  │  Reverse Proxy  │  Load Balancer  │  API Gateway    │
├──────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Primary Job      │ Forward         │ Distribute      │ Manage API      │
│                  │ requests        │ traffic         │ traffic         │
├──────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Auth             │ ❌ No          │ ❌ No           │ ✅ Yes          │
├──────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Rate Limiting    │ Basic          │ ❌ No           │ ✅ Advanced     │
├──────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Request Transform│ ❌ No          │ ❌ No           │ ✅ Yes          │
├──────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ API Versioning   │ ❌ No          │ ❌ No           │ ✅ Yes          │
├──────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Service Discovery│ ❌ No          │ Sometimes       │ ✅ Yes          │
├──────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Analytics        │ Basic logs     │ Basic metrics   │ ✅ Full API     │
│                  │                │                 │   analytics     │
└──────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

> We explore this comparison in detail in Chapter 23.5: "API Gateway vs Reverse Proxy vs Load Balancer."

---

## Key Takeaways

- **API Gateway = single entry point** for all client requests to your backend services
- It handles **cross-cutting concerns** (auth, rate limiting, logging, SSL) in one place so individual services don't have to
- The gateway uses a **filter/plugin chain** internally — each request passes through configurable middleware
- **Never put business logic** in the gateway — it should only do routing, security, and traffic management
- Deploy **multiple gateway instances** behind a load balancer to avoid making it a single point of failure
- Popular tools include **Kong, AWS API Gateway, Nginx, Spring Cloud Gateway, and Envoy**
- At scale, companies like Netflix and Amazon handle **billions of requests/day** through their API gateways

---

## What's Next?

Now that you understand what an API Gateway is and why it exists, the next chapter dives into one of its most critical functions: **Authentication & Authorization at the Gateway** — how the gateway decides who gets in and what they're allowed to do.

Next: [02-auth-at-gateway.md](./02-auth-at-gateway.md)
