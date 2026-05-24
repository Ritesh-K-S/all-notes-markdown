# Strangler Fig Pattern — Migrating from Monolith to Microservices

> **What you'll learn**: How to incrementally replace a legacy monolith with new services — one piece at a time — without ever doing a risky "big bang" rewrite, keeping the system running at all times.

---

## Real-Life Analogy

In nature, a **strangler fig** is a tree that grows around another tree. It starts as a small seed on the host tree, slowly grows roots down and branches up, and gradually wraps around the host tree. Over years, the fig **completely replaces** the original tree — which eventually dies — while the fig stands in its place, fully grown.

At no point is there a "cutover day." The transition happens gradually, piece by piece.

That's exactly how you should migrate from a legacy monolith to microservices:
1. Build new functionality in a **new service** alongside the old system
2. **Route traffic** from the old code to the new service
3. Once the new service handles everything, **remove the old code**
4. Repeat for the next piece

The monolith slowly shrinks while new services grow around it — until the monolith is completely replaced (or only a small core remains).

---

## Core Concept Explained Step-by-Step

### The Problem It Solves

You have a legacy monolith that:
- Is hard to change (brittle, spaghetti code)
- Can't be rewritten from scratch (too risky, too expensive)
- Must keep running (business can't stop)
- Needs to modernize (new features, better scaling)

### Why NOT a Big Bang Rewrite?

```
BIG BANG REWRITE (Dangerous!):

Month 0:         Month 12:          Month 18:
┌──────────┐     ┌──────────┐       ┌──────────┐   ┌──────────┐
│  Legacy  │     │  Legacy  │       │  Legacy  │   │   New    │
│ Monolith │     │  (still  │       │  OFF!    │   │  System  │
│ (running)│     │  running)│       └──────────┘   │  (ON!)   │
└──────────┘     └──────────┘                      └──────────┘
                 ┌──────────┐       
                 │   New    │       RISKS:
                 │ System   │       • Takes 2x longer than estimated
                 │ (building)│       • New system has different bugs
                 │          │       • Business requirements changed during rewrite
                 └──────────┘       • Cutover day = MASSIVE risk
                                    • If new system fails → catastrophe

STRANGLER FIG (Safe!):

Month 0:         Month 6:          Month 12:        Month 18:
┌──────────┐    ┌──────────┐      ┌──────────┐    ┌──────────┐
│  Legacy  │    │  Legacy  │      │  Legacy  │    │  Legacy  │
│ 100%     │    │  85%     │      │  50%     │    │  10%     │
│ traffic  │    │ traffic  │      │ traffic  │    │ traffic  │
└──────────┘    └──────────┘      └──────────┘    └──────────┘
                ┌──────────┐      ┌──────────┐    ┌──────────┐
                │  New Svc │      │ New Svcs │    │ New Svcs │
                │  15%     │      │  50%     │    │  90%     │
                │ traffic  │      │ traffic  │    │ traffic  │
                └──────────┘      └──────────┘    └──────────┘

✅ Always running, always serving users
✅ Migrate one piece at a time
✅ Roll back any single piece if it fails
✅ Deliver value continuously during migration
```

### How It Works

```
STEP 1: Put a PROXY in front of the monolith

┌──────────┐     ┌─────────────┐     ┌──────────────┐
│  Client  │────▶│   PROXY     │────▶│   Monolith   │
│          │◀────│  (router)   │◀────│   (100%)     │
└──────────┘     └─────────────┘     └──────────────┘
                 (routes everything 
                  to monolith initially)


STEP 2: Build first new service, route SOME traffic to it

┌──────────┐     ┌─────────────┐     ┌──────────────┐
│  Client  │────▶│   PROXY     │──┬─▶│   Monolith   │
│          │◀────│             │  │  │   (95%)      │
└──────────┘     └─────────────┘  │  └──────────────┘
                                  │
                                  │  ┌──────────────┐
                                  └─▶│  New User    │
                                     │  Service (5%)│
                                     └──────────────┘


STEP 3: Migration progresses...

┌──────────┐     ┌─────────────┐     ┌──────────────┐
│  Client  │────▶│   PROXY     │──┬─▶│   Monolith   │ (shrinking)
│          │◀────│             │  │  │   (30%)      │
└──────────┘     └─────────────┘  │  └──────────────┘
                                  │
                                  ├─▶┌──────────────┐
                                  │  │ User Service │
                                  │  └──────────────┘
                                  ├─▶┌──────────────┐
                                  │  │Order Service │
                                  │  └──────────────┘
                                  └─▶┌──────────────┐
                                     │Product Service│
                                     └──────────────┘


STEP 4: Monolith fully replaced (or removed)

┌──────────┐     ┌─────────────┐  ┌──────────────┐
│  Client  │────▶│   PROXY /   │─▶│ User Service │
│          │◀────│ API Gateway │─▶│Order Service │
└──────────┘     └─────────────┘─▶│Product Service│
                                   │Payment Service│
                                   └──────────────┘
                                   
                 Monolith is gone! (or minimal shell)
```

---

## How It Works Internally

### The Proxy/Router Layer

The proxy is the **key enabler** — it decides which backend handles each request:

```
┌─────────────────────────────────────────────────────────────────┐
│                       PROXY ROUTING RULES                        │
│                                                                 │
│  Rule 1: /api/users/*           → User Service (new)            │
│  Rule 2: /api/products/*        → Product Service (new)         │
│  Rule 3: /api/orders/*          → Order Service (new)           │
│  Rule 4: /api/reports/*         → Monolith (not migrated yet)   │
│  Rule 5: /api/admin/*           → Monolith (not migrated yet)   │
│  Rule 6: /* (everything else)   → Monolith (default)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Migration Strategies Per Feature

```
STRATEGY 1: Route by URL Path
/api/users → New Service
/api/orders → Monolith (for now)

STRATEGY 2: Route by Feature Flag
if (user.inBetaGroup("new-checkout")):
    route → New Checkout Service
else:
    route → Monolith

STRATEGY 3: Route by Percentage (Canary)
10% of /api/orders → New Order Service
90% of /api/orders → Monolith
(gradually increase %)

STRATEGY 4: Route by Data
orders.created_after("2024-01-01") → New Service
orders.created_before("2024-01-01") → Monolith
```

### Data Migration Challenge

The hardest part of strangler fig is **data**:

```
PROBLEM: Old monolith and new service need the same data

SOLUTION 1: Shared Database (temporary)
┌──────────────┐    ┌──────────────┐
│  Monolith    │    │ New Service  │
└──────┬───────┘    └──────┬───────┘
       │                   │
       └───────┬───────────┘
               ▼
        ┌────────────┐
        │  Shared DB │  ← Temporary! Remove once monolith code is deleted
        └────────────┘

SOLUTION 2: Data Sync (CDC)
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Monolith    │    │  Debezium    │    │ New Service  │
│  (writes to  │───▶│  (CDC)       │───▶│ (own DB,     │
│   old DB)    │    │              │    │  synced data)│
└──────────────┘    └──────────────┘    └──────────────┘

SOLUTION 3: API Calls Back to Monolith
┌──────────────┐                        ┌──────────────┐
│ New Service  │──── GET /api/users ────▶│  Monolith    │
│ (needs user  │◀─── user data ─────────│ (still owns  │
│  data)       │                        │  user data)  │
└──────────────┘                        └──────────────┘
(New service calls back to monolith for data it doesn't own yet)
```

### Event Interception

```
BEFORE (Monolith handles everything internally):
┌──────────────────────────────────────────┐
│            Monolith                       │
│                                          │
│  Order Created → Update Inventory        │
│                → Send Email              │
│                → Update Analytics        │
└──────────────────────────────────────────┘

AFTER (Monolith publishes events, new services consume):
┌──────────────────────────────────────────┐
│            Monolith                       │
│  Order Created → Publish Event ─────────────▶ ┌───────────────┐
│                                          │    │ Kafka/RabbitMQ │
└──────────────────────────────────────────┘    └───────┬───────┘
                                                        │
                                          ┌─────────────┼────────────┐
                                          ▼             ▼            ▼
                                   ┌──────────┐  ┌──────────┐  ┌─────────┐
                                   │Inventory │  │  Email   │  │Analytics│
                                   │ Service  │  │ Service  │  │ Service │
                                   │  (new)   │  │  (new)   │  │ (new)   │
                                   └──────────┘  └──────────┘  └─────────┘
```

---

## Code Examples

### Python (Proxy Router with Feature Flags)

```python
# proxy/router.py — Routes traffic between monolith and new services
from flask import Flask, request, Response
import httpx
import random

app = Flask(__name__)

# Configuration: which routes go where
ROUTING_TABLE = {
    "/api/users": {
        "new_service": "http://user-service:8080",
        "monolith": "http://monolith:8080",
        "percentage_to_new": 100,  # Fully migrated!
    },
    "/api/products": {
        "new_service": "http://product-service:8080",
        "monolith": "http://monolith:8080",
        "percentage_to_new": 50,   # 50% canary
    },
    "/api/orders": {
        "new_service": None,       # Not built yet
        "monolith": "http://monolith:8080",
        "percentage_to_new": 0,    # 100% to monolith
    },
}

@app.route('/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
def proxy(path):
    """Route requests based on migration progress."""
    full_path = f"/{path}"
    
    # Find matching route
    target = determine_target(full_path)
    
    # Forward request to target backend
    resp = httpx.request(
        method=request.method,
        url=f"{target}{full_path}",
        headers=dict(request.headers),
        content=request.get_data(),
        params=request.args,
        timeout=10.0
    )
    
    return Response(
        resp.content,
        status=resp.status_code,
        headers=dict(resp.headers)
    )

def determine_target(path: str) -> str:
    """Decide whether to route to new service or monolith."""
    for prefix, config in ROUTING_TABLE.items():
        if path.startswith(prefix):
            if config["new_service"] and random.randint(1, 100) <= config["percentage_to_new"]:
                return config["new_service"]
            return config["monolith"]
    
    # Default: everything unmapped goes to monolith
    return "http://monolith:8080"
```

### Java (Spring Cloud Gateway as Strangler Proxy)

```java
// GatewayConfig.java — Spring Cloud Gateway routing configuration
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // Fully migrated: all user traffic → User Service
            .route("users-new", r -> r
                .path("/api/users/**")
                .uri("http://user-service:8080"))
            
            // Canary: 30% of product traffic → new Product Service
            .route("products-canary", r -> r
                .path("/api/products/**")
                .and()
                .weight("products", 30)
                .uri("http://product-service:8080"))
            
            // Remaining 70% → monolith
            .route("products-legacy", r -> r
                .path("/api/products/**")
                .and()
                .weight("products", 70)
                .uri("http://monolith:8080"))
            
            // Not migrated yet: orders still go to monolith
            .route("orders-legacy", r -> r
                .path("/api/orders/**")
                .uri("http://monolith:8080"))
            
            // Default: everything else → monolith
            .route("catch-all", r -> r
                .path("/**")
                .uri("http://monolith:8080"))
            
            .build();
    }
}
```

### Nginx Configuration (Simple Proxy Routing)

```nginx
# nginx.conf — Route traffic between old and new systems
upstream monolith {
    server monolith-app:8080;
}

upstream user_service {
    server user-service:8080;
}

upstream product_service {
    server product-service:8080;
}

server {
    listen 80;

    # Fully migrated to new service
    location /api/users/ {
        proxy_pass http://user_service;
        proxy_set_header Host $host;
        proxy_set_header X-Migration-Status "new-service";
    }

    # Canary: split traffic using split_clients
    split_clients "${remote_addr}${uri}" $product_backend {
        30% product_service;  # 30% to new service
        *   monolith;         # 70% to monolith
    }

    location /api/products/ {
        proxy_pass http://$product_backend;
        proxy_set_header Host $host;
    }

    # Not yet migrated — still on monolith
    location /api/orders/ {
        proxy_pass http://monolith;
        proxy_set_header Host $host;
    }

    # Default: everything else goes to monolith
    location / {
        proxy_pass http://monolith;
        proxy_set_header Host $host;
    }
}
```

---

## Infrastructure Example

### Full Migration Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    STRANGLER FIG IN PRODUCTION                            │
│                                                                           │
│  ┌──────────┐     ┌──────────────────────────────────────────────┐       │
│  │  Client  │────▶│            API GATEWAY / PROXY               │       │
│  └──────────┘     │  (Kong / Nginx / AWS ALB / Spring Gateway)   │       │
│                   └─────┬──────────┬──────────┬──────────┬───────┘       │
│                         │          │          │          │                │
│                         ▼          ▼          ▼          ▼                │
│  NEW SERVICES:    ┌──────────┐ ┌────────┐                                │
│                   │  User    │ │Product │                                │
│                   │ Service  │ │Service │     ← Migrated features        │
│                   │  (Go)    │ │(Java)  │                                │
│                   └──────────┘ └────────┘                                │
│                                                                           │
│  LEGACY:                              ┌──────────────────────────┐       │
│                                       │      MONOLITH            │       │
│                                       │   (still handles orders, │       │
│                                       │    reports, admin, etc.) │       │
│                                       └──────────────────────────┘       │
│                                                                           │
│  DATA SYNC:       ┌──────────────────────────────────────────────┐       │
│                   │         Debezium (CDC)                        │       │
│                   │  Syncs data from monolith DB → new service DBs│       │
│                   └──────────────────────────────────────────────┘       │
│                                                                           │
│  EVENTS:          ┌──────────────────────────────────────────────┐       │
│                   │              Kafka                            │       │
│                   │  Monolith publishes events →                  │       │
│                   │  New services consume & react                 │       │
│                   └──────────────────────────────────────────────┘       │
│                                                                           │
│  MONITORING:      ┌──────────────────────────────────────────────┐       │
│                   │  Compare responses: old vs new (shadow mode)  │       │
│                   │  Track error rates per route                  │       │
│                   │  Alert on divergence                          │       │
│                   └──────────────────────────────────────────────┘       │
└───────────────────────────────────────────────────────────────────────────┘
```

### Migration Verification (Shadow Testing)

```python
# shadow_proxy.py — Send to BOTH and compare (verify new service works)
async def shadow_test_route(request):
    """Send request to both old and new service, compare results."""
    
    # Primary: always return response from monolith (safe)
    monolith_response = await call_monolith(request)
    
    # Shadow: also call new service (don't return this to user)
    try:
        new_service_response = await call_new_service(request)
        
        # Compare responses
        if monolith_response.body != new_service_response.body:
            log_divergence(
                path=request.path,
                old_response=monolith_response.body,
                new_response=new_service_response.body
            )
            metrics.increment("shadow.divergence")
        else:
            metrics.increment("shadow.match")
    except Exception as e:
        metrics.increment("shadow.new_service_error")
        log.error(f"New service failed: {e}")
    
    # Always return monolith response (user never sees new service issues)
    return monolith_response
```

---

## Real-World Example

### Amazon — The Original Strangler Migration

Amazon's migration from monolith to services (early 2000s) is the most famous example:

```
2001: Amazon.com = ONE massive C++/Perl monolith

Jeff Bezos' mandate:
"All teams must expose their functionality through service interfaces.
 No exceptions. Anyone who doesn't do this will be fired."

2001-2006: Gradual strangling
┌──────────────────────────────────────────────────┐
│  Year 1: [Product Catalog] extracted as service  │
│  Year 2: [Customer Accounts] extracted           │
│  Year 3: [Order Processing] extracted            │
│  Year 4: [Recommendations] extracted             │
│  ...                                             │
│  Year 5: Monolith is just a thin shell           │
└──────────────────────────────────────────────────┘

Result: Hundreds of independent services → foundation for AWS
```

### Uber — Migrating DISPATCH

```
Uber's dispatch system migration:

Phase 1 (2014): Monolith handles ALL ride dispatch
┌──────────────────────────┐
│  Python Monolith         │
│  (dispatch + everything) │
└──────────────────────────┘

Phase 2 (2015): New Go service handles SOME dispatch
┌──────────┐     ┌──────────────┐
│ Monolith │     │ New Dispatch │ (handles 5% of rides)
│ (95%)    │     │ Service (Go) │
└──────────┘     └──────────────┘

Phase 3 (2016): New service handles ALL dispatch
                 ┌──────────────┐
                 │ New Dispatch │ (100% of rides)
                 │ Service (Go) │
                 └──────────────┘
Monolith dispatch code: DELETED ✓

Repeated this process for 100+ features over 3 years.
```

### Twitter — Rails to JVM

```
2007-2012: Ruby on Rails monolith
2012-2015: Gradually extracted services in Scala/Java

Extracted in order:
1. Timeline service (Scala) — highest traffic
2. Tweet storage (Java) — highest data volume  
3. User service (Scala)
4. Search (Java + Lucene)
...
N. Rails monolith → tiny shell with only legacy admin tools
```

---

## Common Mistakes / Pitfalls

### 1. No Proxy Layer
❌ **Mistake**: Trying to migrate without a routing layer — clients must know about new services directly.
✅ **Fix**: Always put a proxy/gateway in front that routes transparently. Clients never know which backend serves them.

### 2. Migrating Data and Code Simultaneously
❌ **Mistake**: Moving the code AND database schema in one big step.
✅ **Fix**: Separate concerns:
  - Step 1: New code, same database (shared temporarily)
  - Step 2: New code, new database (sync from old)
  - Step 3: Remove old code + old tables

### 3. No Rollback Plan
❌ **Mistake**: Routing 100% to new service immediately with no way back.
✅ **Fix**: Always keep the ability to route back to the monolith. Use percentage-based routing.

```
SAFE ROLLOUT:
Day 1:   1% → new service  (test with real traffic)
Day 3:  10% → new service  (monitor for errors)
Day 7:  50% → new service  (stress test)
Day 14: 100% → new service (fully migrated)

If errors spike at any point → route back to 0% instantly!
```

### 4. Strangling Too Much at Once
❌ **Mistake**: Trying to extract 5 services simultaneously.
✅ **Fix**: One service at a time. Get it fully migrated and stable before starting the next.

### 5. Never Deleting Old Code
❌ **Mistake**: New service works, but monolith still has the old code (dead code accumulates).
✅ **Fix**: Once a route is 100% migrated and stable for 2+ weeks, DELETE the old code from the monolith.

---

## When to Use / When NOT to Use

### ✅ Use Strangler Fig When:

| Criteria | Why |
|---|---|
| **Existing legacy system must stay running** | Can't afford downtime |
| **System too large to rewrite at once** | Incremental is the only realistic approach |
| **Need to deliver features during migration** | Business can't wait 2 years for a rewrite |
| **High-risk system** | Banking, e-commerce, healthcare — can't gamble on big bang |
| **Team is learning microservices** | Migrate one service, learn, then do the next |

### ❌ Avoid When:

| Criteria | Why |
|---|---|
| **System is small enough to rewrite in weeks** | Just rewrite it |
| **Monolith is well-structured** | Modular monolith might be sufficient |
| **No clear domain boundaries** | Can't decide what to extract first |
| **No traffic routing infrastructure** | Need API gateway / proxy capabilities first |
| **Team too small** | Running old + new simultaneously requires capacity |

---

## Migration Playbook (Step-by-Step)

```
┌─────────────────────────────────────────────────────────────────┐
│                STRANGLER FIG PLAYBOOK                            │
│                                                                 │
│  STEP 1: Add Proxy/Gateway                                      │
│     → Route all traffic through it (to monolith initially)      │
│                                                                 │
│  STEP 2: Identify first extraction candidate                    │
│     → Choose something small, well-understood, high-value       │
│     → Ideally with few dependencies on other monolith parts     │
│                                                                 │
│  STEP 3: Build new service                                      │
│     → Implement same functionality + same API contract          │
│     → Write comprehensive tests                                 │
│                                                                 │
│  STEP 4: Shadow test                                            │
│     → Send traffic to BOTH, compare responses                   │
│     → Fix any divergences                                       │
│                                                                 │
│  STEP 5: Canary release                                         │
│     → Route 1% → 10% → 50% → 100% to new service              │
│     → Monitor errors, latency, correctness at each step         │
│                                                                 │
│  STEP 6: Remove old code                                        │
│     → Delete migrated code from monolith                        │
│     → Clean up old database tables (if data fully migrated)     │
│                                                                 │
│  STEP 7: Repeat for next feature                                │
│     → Go back to Step 2                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

- 🌳 **Strangler Fig = incremental migration** — replace a monolith piece by piece, never as a "big bang."
- 🔀 **A proxy/gateway routes traffic** — transparently sending requests to either the old monolith or new services.
- 📊 **Use percentage-based routing** — start at 1%, watch metrics, gradually increase to 100%.
- 🔙 **Always have a rollback plan** — if the new service fails, route back to the monolith instantly.
- 🗑️ **Delete old code** — once a migration is stable, remove the dead code from the monolith. Don't let it accumulate.
- 📦 **Migrate one feature at a time** — don't try to extract 5 services simultaneously. Sequential is safer.
- 🏢 **Amazon, Uber, Twitter, and Netflix all used this pattern** — it's the proven path from monolith to microservices.

---

## What's Next?

Congratulations! You've completed **Part 4: Architecture Patterns**. You now understand the full spectrum from monolith to microservices, including how to evolve between them.

In **Part 5: Deployment Models**, we'll explore how to actually deploy these architectures — from a single server to multi-region global deployments with zero-downtime releases. Start with **Chapter 5.1: Single Server Setup — Everything on One Machine**.
