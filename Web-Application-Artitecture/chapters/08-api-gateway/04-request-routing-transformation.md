# Request Routing, Transformation & Aggregation

> **What you'll learn**: How an API Gateway routes incoming requests to the correct backend service, transforms requests/responses between different formats, and aggregates data from multiple services into a single response — the three superpowers that make gateways indispensable.

---

## Real-Life Analogy: The International Airport

Think of a large international airport:

**Routing** = The departure boards and gate assignments. You look at the board, see "Flight to Tokyo → Gate B7", and walk to the right gate. Different destinations use different gates and terminals.

**Transformation** = The currency exchange counter. You arrive with US Dollars, but you need Japanese Yen. The exchange converts your money into the format your destination accepts.

**Aggregation** = The travel agent who books your flight, hotel, and rental car in one call. Instead of you calling three different companies, one person calls all three and gives you a single itinerary.

```
The API Gateway does all three:

Client Request: GET /dashboard
                    │
                    ▼
           ┌──────────────────┐
           │   API GATEWAY    │
           │                  │
           │  1. ROUTE        │ "Which service(s) handle this?"
           │  2. TRANSFORM    │ "Convert the request format"
           │  3. AGGREGATE    │ "Combine responses from multiple services"
           │                  │
           └──────┬───────────┘
                  │
      ┌───────────┼───────────┐
      ▼           ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ User     │ │ Orders   │ │ Analytics│
│ Service  │ │ Service  │ │ Service  │
└──────────┘ └──────────┘ └──────────┘
      │           │           │
      └───────────┼───────────┘
                  │
                  ▼ Combined response
           ┌──────────────────┐
           │ { user: {...},   │
           │   orders: [...], │
           │   stats: {...} } │
           └──────────────────┘
```

---

## Part 1: Request Routing

### What Is Request Routing?

Request routing is the process of examining an incoming request and deciding which backend service should handle it. The gateway looks at various attributes of the request to make this decision.

### Routing Strategies

```
┌─────────────────────────────────────────────────────────────────┐
│                    ROUTING STRATEGIES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. PATH-BASED ROUTING                                         │
│     /users/*     → User Service                                │
│     /orders/*    → Order Service                               │
│     /payments/*  → Payment Service                             │
│                                                                 │
│  2. HEADER-BASED ROUTING                                       │
│     X-Version: v2        → Service v2                          │
│     X-Client: mobile     → Mobile-optimized Service            │
│                                                                 │
│  3. METHOD-BASED ROUTING                                       │
│     GET /products        → Read Service (replica)              │
│     POST /products       → Write Service (primary)             │
│                                                                 │
│  4. QUERY PARAMETER ROUTING                                    │
│     /search?type=images  → Image Search Service                │
│     /search?type=video   → Video Search Service                │
│                                                                 │
│  5. WEIGHT-BASED ROUTING (Canary)                             │
│     90% traffic → Service v1 (stable)                          │
│     10% traffic → Service v2 (canary)                          │
│                                                                 │
│  6. HOST-BASED ROUTING                                         │
│     api.myapp.com        → API Backend                         │
│     admin.myapp.com      → Admin Backend                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Path-Based Routing (Most Common)

```
Incoming request: GET /api/v1/users/123/orders

Route matching process:
┌─────────────────────────────────────────────┐
│ Route Table (checked in order):             │
│                                             │
│ Priority  Pattern              Service      │
│ ────────  ───────              ───────      │
│    1      /api/v1/users/*      user-svc     │
│    2      /api/v1/orders/*     order-svc    │
│    3      /api/v1/payments/*   payment-svc  │
│    4      /api/v1/products/*   product-svc  │
│    5      /*                   default-svc  │
│                                             │
│ Match: /api/v1/users/123/orders             │
│        ↓ matches /api/v1/users/*            │
│        → Route to user-svc                  │
└─────────────────────────────────────────────┘
```

### Canary Routing (Weight-Based)

```
Deploying a new version? Route only 5% of traffic to it:

                    ┌─────────────┐
                    │   Gateway   │
                    │             │
All Traffic ──────▶ │  95% ────────────▶ Service v1.0 (stable)
                    │             │
                    │   5% ────────────▶ Service v2.0 (canary)
                    │             │
                    └─────────────┘

Gradually increase: 5% → 10% → 25% → 50% → 100%
If errors increase on v2.0 → immediately route 100% back to v1.0
```

### Header-Based Routing (A/B Testing)

```
Route based on custom headers for A/B testing:

Request with: X-Experiment: new-checkout
  → Routes to checkout-service-v2 (new UI)

Request without header or X-Experiment: control
  → Routes to checkout-service-v1 (old UI)

┌──────────┐   X-Experiment: new-checkout   ┌─────────────────────┐
│  Client  │ ──────────────────────────────▶ │ checkout-svc-v2     │
│  (5%)    │                                 │ (new experience)    │
└──────────┘                                 └─────────────────────┘

┌──────────┐   X-Experiment: control         ┌─────────────────────┐
│  Client  │ ──────────────────────────────▶ │ checkout-svc-v1     │
│  (95%)   │                                 │ (current experience)│
└──────────┘                                 └─────────────────────┘
```

---

## Part 2: Request & Response Transformation

### What Is Transformation?

Transformation modifies the request before it reaches the backend, or modifies the response before it reaches the client. This is essential when:

- Clients speak a different "language" than backends
- You need to add/remove headers
- You want to hide internal details from external clients
- You need to convert between formats (JSON ↔ XML, REST ↔ SOAP)

### Types of Transformations

```
┌─────────────────────────────────────────────────────────────┐
│                REQUEST TRANSFORMATIONS                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Header Manipulation:                                       │
│  ├── ADD:    X-Request-ID: uuid-123                        │
│  ├── ADD:    X-User-Id: (from JWT)                         │
│  ├── REMOVE: Cookie (don't send to backend)                │
│  └── RENAME: Authorization → X-Internal-Auth               │
│                                                             │
│  Path Rewriting:                                           │
│  ├── /api/v1/users/123 → /users/123 (strip prefix)        │
│  └── /mobile/feed → /feed?format=compact (rewrite)        │
│                                                             │
│  Body Transformation:                                       │
│  ├── JSON → XML (for legacy SOAP backends)                 │
│  ├── Add default fields: { "region": "us-east-1" }        │
│  └── Remove sensitive fields before forwarding             │
│                                                             │
│  Protocol Translation:                                      │
│  ├── HTTP/REST → gRPC                                      │
│  ├── HTTP/REST → GraphQL                                   │
│  └── WebSocket → HTTP Long Polling                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                RESPONSE TRANSFORMATIONS                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Field Filtering:                                           │
│  ├── Remove internal fields (internal_id, debug_info)      │
│  └── Select only requested fields (?fields=name,email)     │
│                                                             │
│  Format Conversion:                                         │
│  ├── XML → JSON (for modern clients)                       │
│  └── Protobuf → JSON (gRPC service to REST client)        │
│                                                             │
│  Header Manipulation:                                       │
│  ├── ADD: Access-Control-Allow-Origin: *                   │
│  ├── ADD: X-Response-Time: 42ms                            │
│  └── REMOVE: Server: nginx/1.21 (hide server info)        │
│                                                             │
│  Response Wrapping:                                         │
│  └── { "data": {...}, "meta": { "page": 1 } }            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Transformation Example Flow

```
Client sends:
  POST /api/users
  Content-Type: application/json
  Authorization: Bearer eyJhbG...
  X-Client-Version: 2.1.0
  {
    "name": "Alice",
    "email": "alice@example.com"
  }

Gateway transforms → Backend receives:
  POST /users                          ← Path rewritten (prefix stripped)
  Content-Type: application/json
  X-User-Id: user_12345               ← Extracted from JWT
  X-Request-Id: req_abc123            ← Generated for tracing
  X-Forwarded-For: 203.0.113.45      ← Client's real IP
  {
    "name": "Alice",
    "email": "alice@example.com",
    "region": "us-east-1",            ← Added by gateway
    "source": "api-v2"                ← Added by gateway
  }

Backend responds:
  200 OK
  {
    "id": "user_12345",
    "name": "Alice",
    "email": "alice@example.com",
    "internal_score": 85.5,           ← Internal field
    "db_shard": "shard-3",            ← Internal field
    "created_at": "2024-01-15"
  }

Gateway transforms → Client receives:
  200 OK
  X-Request-Id: req_abc123
  X-Response-Time: 42ms
  Access-Control-Allow-Origin: *       ← CORS header added
  {
    "id": "user_12345",
    "name": "Alice",
    "email": "alice@example.com",
    "created_at": "2024-01-15"
  }
  ← internal_score and db_shard REMOVED (not for external consumption)
```

---

## Part 3: Response Aggregation (API Composition)

### What Is Aggregation?

Sometimes a client needs data from multiple services. Instead of the client making 5 separate API calls, the gateway makes them internally and returns one combined response.

```
Without aggregation — Client makes multiple calls:

┌──────────┐  GET /users/123         ┌──────────────┐
│          │ ──────────────────────▶ │ User Service │ → { name, email }
│          │                         └──────────────┘
│          │  GET /users/123/orders   ┌──────────────┐
│  Client  │ ──────────────────────▶ │ Order Service│ → [order1, order2]
│          │                         └──────────────┘
│          │  GET /users/123/payments ┌──────────────┐
│          │ ──────────────────────▶ │ Payment Svc  │ → [payment1, ...]
└──────────┘                         └──────────────┘

3 round trips! On mobile with 200ms latency = 600ms minimum!

With aggregation — Client makes ONE call:

┌──────────┐  GET /users/123/dashboard  ┌─────────────┐
│  Client  │ ─────────────────────────▶ │ API GATEWAY │
└──────────┘                            │             │
                                        │ (parallel)  │
            ┌───────────────────────────┤             │
            │          │                │             │
            ▼          ▼                ▼             │
      ┌──────────┐ ┌──────────┐ ┌──────────┐        │
      │ User Svc │ │Order Svc │ │Payment   │        │
      └──────────┘ └──────────┘ └──────────┘        │
            │          │                │             │
            └───────────────────────────┤             │
                                        │  Combine   │
┌──────────┐  { user, orders, payments }│             │
│  Client  │ ◀─────────────────────────┤             │
└──────────┘  ONE response, ONE trip    └─────────────┘

1 round trip! = 200ms + backend processing time
```

### Aggregation Patterns

```
Pattern 1: Parallel Aggregation
─────────────────────────────────
Call all services simultaneously, combine when all respond.

  Gateway ──┬──▶ Service A ──┐
            ├──▶ Service B ──┤ Wait for ALL
            └──▶ Service C ──┘
                              │
                              ▼ Combine & return

Total time = MAX(latency_A, latency_B, latency_C)


Pattern 2: Sequential Aggregation (Chaining)
─────────────────────────────────────────────
Output of one call is input to the next.

  Gateway ──▶ Service A ──▶ (use result) ──▶ Service B ──▶ (combine)

Example: Get user → use user.region → get local offers for that region

Total time = latency_A + latency_B


Pattern 3: Conditional Aggregation
──────────────────────────────────
Call additional services based on results of previous calls.

  Gateway ──▶ User Service
               │
               ├── if user.isPremium:
               │   └──▶ Premium Offers Service
               │
               └── if user.hasOrders:
                   └──▶ Order Recommendations Service
```

---

## How It Works Internally

### Route Matching Engine

```
How the gateway matches routes (similar to a trie data structure):

Route Trie:
                         /
                        / \
                     api   health
                     /
                    v1
                   / | \
              users orders payments
              /       |
          {id}      {id}
           / \        |
      orders profile items

Request: /api/v1/users/123/orders
Match path: / → api → v1 → users → {id=123} → orders
Result: user-order-service (captures: {id: "123"})

Complexity: O(path_segments) — very fast regardless of # of routes
```

### Request Pipeline for Routing + Transformation

```
Internal Gateway Pipeline:

  ┌─────────────────────────────────────────────────┐
  │               Incoming Request                   │
  └─────────────────────┬───────────────────────────┘
                        │
                        ▼
  ┌─────────────────────────────────────────────────┐
  │  Pre-Route Filters (run before routing)         │
  │  • Authentication                               │
  │  • Rate Limiting                                │
  │  • Request Logging                              │
  └─────────────────────┬───────────────────────────┘
                        │
                        ▼
  ┌─────────────────────────────────────────────────┐
  │  Route Matcher                                   │
  │  • Match path, headers, method                  │
  │  • Select target service + load balance         │
  └─────────────────────┬───────────────────────────┘
                        │
                        ▼
  ┌─────────────────────────────────────────────────┐
  │  Route-Specific Filters (per-route transforms)  │
  │  • Path rewriting                               │
  │  • Header manipulation                          │
  │  • Body transformation                          │
  └─────────────────────┬───────────────────────────┘
                        │
                        ▼
  ┌─────────────────────────────────────────────────┐
  │  Proxy to Backend Service                        │
  │  • Connection pooling                           │
  │  • Timeout handling                             │
  │  • Circuit breaking                             │
  └─────────────────────┬───────────────────────────┘
                        │
                        ▼
  ┌─────────────────────────────────────────────────┐
  │  Post-Route Filters (run after response)        │
  │  • Response transformation                      │
  │  • Add CORS headers                             │
  │  • Response caching                             │
  │  • Metrics collection                           │
  └─────────────────────┬───────────────────────────┘
                        │
                        ▼
  ┌─────────────────────────────────────────────────┐
  │               Response to Client                 │
  └─────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: Request Routing and Transformation

```python
# gateway_routing.py
# API Gateway with routing, transformation, and aggregation

import asyncio
import aiohttp
import uuid
import time
from flask import Flask, request, jsonify

app = Flask(__name__)

# Route configuration
ROUTES = [
    {
        "path_prefix": "/api/v1/users",
        "service": "http://user-service:8001",
        "strip_prefix": "/api/v1",
        "methods": ["GET", "POST", "PUT", "DELETE"],
        "transforms": {
            "add_headers": {"X-Service-Version": "v1"},
            "remove_response_fields": ["internal_id", "db_shard"],
        },
    },
    {
        "path_prefix": "/api/v1/orders",
        "service": "http://order-service:8002",
        "strip_prefix": "/api/v1",
        "methods": ["GET", "POST"],
        "transforms": {
            "add_headers": {"X-Service-Version": "v1"},
        },
    },
    {
        "path_prefix": "/api/v2/orders",
        "service": "http://order-service-v2:8003",
        "strip_prefix": "/api/v2",
        "methods": ["GET", "POST"],
        "weight": 0.1,  # Only 10% of traffic (canary)
    },
]

# Aggregation endpoints
AGGREGATIONS = {
    "/api/v1/dashboard": {
        "calls": [
            {"service": "http://user-service:8001", "path": "/users/{user_id}",
             "response_key": "user"},
            {"service": "http://order-service:8002", "path": "/orders?user_id={user_id}",
             "response_key": "recent_orders"},
            {"service": "http://analytics-service:8004", "path": "/stats/{user_id}",
             "response_key": "stats"},
        ],
        "parallel": True,
    },
}


def match_route(path, method):
    """Find the matching route for the given path and method."""
    for route in ROUTES:
        if path.startswith(route["path_prefix"]):
            if method in route["methods"]:
                return route
    return None


def transform_request(original_request, route):
    """Transform the request before forwarding to backend."""
    # Strip prefix from path
    path = original_request.path
    if "strip_prefix" in route:
        path = path.replace(route["strip_prefix"], "", 1)

    # Build headers
    headers = dict(original_request.headers)
    headers.pop("Host", None)  # Don't forward Host header
    
    # Add request ID for tracing
    headers["X-Request-Id"] = str(uuid.uuid4())
    headers["X-Forwarded-For"] = original_request.remote_addr
    
    # Add route-specific headers
    if "transforms" in route and "add_headers" in route["transforms"]:
        headers.update(route["transforms"]["add_headers"])

    return {
        "url": f"{route['service']}{path}",
        "headers": headers,
        "method": original_request.method,
        "body": original_request.get_data(),
    }


def transform_response(response_data, route):
    """Transform the response before returning to client."""
    if "transforms" not in route:
        return response_data

    # Remove internal fields
    fields_to_remove = route["transforms"].get("remove_response_fields", [])
    if isinstance(response_data, dict):
        for field in fields_to_remove:
            response_data.pop(field, None)

    return response_data


@app.route("/api/<path:path>", methods=["GET", "POST", "PUT", "DELETE"])
def route_request(path):
    """Main routing handler."""
    full_path = f"/api/{path}"
    start_time = time.time()

    # Check if this is an aggregation endpoint
    if full_path in AGGREGATIONS:
        return handle_aggregation(full_path)

    # Find matching route
    route = match_route(full_path, request.method)
    if not route:
        return jsonify({"error": "Route not found"}), 404

    # Transform and forward request
    transformed = transform_request(request, route)

    import requests as http_client
    response = http_client.request(
        method=transformed["method"],
        url=transformed["url"],
        headers=transformed["headers"],
        data=transformed["body"],
    )

    # Transform response
    response_data = response.json()
    response_data = transform_response(response_data, route)

    # Add gateway headers
    result = jsonify(response_data)
    result.headers["X-Response-Time"] = f"{(time.time() - start_time) * 1000:.0f}ms"
    return result, response.status_code


async def fetch_service(session, call, user_id):
    """Fetch data from a single service (used in aggregation)."""
    path = call["path"].format(user_id=user_id)
    url = f"{call['service']}{path}"
    async with session.get(url) as response:
        return {call["response_key"]: await response.json()}


def handle_aggregation(endpoint):
    """Handle aggregation endpoints — call multiple services, combine responses."""
    config = AGGREGATIONS[endpoint]
    user_id = request.args.get("user_id", "unknown")

    async def aggregate():
        async with aiohttp.ClientSession() as session:
            tasks = [fetch_service(session, call, user_id) for call in config["calls"]]
            results = await asyncio.gather(*tasks, return_exceptions=True)

        # Combine all results into one response
        combined = {}
        for result in results:
            if isinstance(result, dict):
                combined.update(result)
            else:
                combined["errors"] = combined.get("errors", [])
                combined["errors"].append(str(result))
        return combined

    result = asyncio.run(aggregate())
    return jsonify(result)
```

### Java: Spring Cloud Gateway with Routing and Transformation

```java
// GatewayRoutingConfig.java
// Advanced routing with path rewriting, header manipulation, and response filtering

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;

@Configuration
public class GatewayRoutingConfig {

    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()

            // Route 1: Path-based routing with prefix stripping
            .route("user-service", r -> r
                .path("/api/v1/users/**")
                .and().method(HttpMethod.GET, HttpMethod.POST)
                .filters(f -> f
                    .stripPrefix(2)                              // Remove /api/v1
                    .addRequestHeader("X-Request-Id",            // Tracing
                        java.util.UUID.randomUUID().toString())
                    .addRequestHeader("X-Source", "api-gateway")
                    .removeResponseHeader("Server")              // Hide internal info
                    .addResponseHeader("X-Gateway", "true")
                )
                .uri("lb://user-service"))  // lb:// = load-balanced lookup

            // Route 2: Canary deployment — weight-based routing
            .route("order-service-canary", r -> r
                .path("/api/v1/orders/**")
                .and().weight("orders", 10)  // 10% to canary
                .filters(f -> f
                    .stripPrefix(2)
                    .addRequestHeader("X-Canary", "true"))
                .uri("lb://order-service-v2"))

            .route("order-service-stable", r -> r
                .path("/api/v1/orders/**")
                .and().weight("orders", 90)  // 90% to stable
                .filters(f -> f.stripPrefix(2))
                .uri("lb://order-service-v1"))

            // Route 3: Header-based routing (mobile vs web)
            .route("mobile-optimized", r -> r
                .path("/api/v1/feed/**")
                .and().header("X-Client-Type", "mobile")
                .filters(f -> f
                    .stripPrefix(2)
                    .addRequestParameter("compact", "true")     // Request less data
                    .modifyResponseBody(String.class, String.class,
                        (exchange, body) -> filterFieldsForMobile(body)))
                .uri("lb://feed-service"))

            // Route 4: Method-based routing (CQRS: reads vs writes)
            .route("product-reads", r -> r
                .path("/api/v1/products/**")
                .and().method(HttpMethod.GET)
                .filters(f -> f.stripPrefix(2))
                .uri("lb://product-read-service"))  // Read replicas

            .route("product-writes", r -> r
                .path("/api/v1/products/**")
                .and().method(HttpMethod.POST, HttpMethod.PUT, HttpMethod.DELETE)
                .filters(f -> f.stripPrefix(2))
                .uri("lb://product-write-service")) // Write primary

            .build();
    }

    private String filterFieldsForMobile(String responseBody) {
        // Remove heavy fields for mobile clients
        // In production, use Jackson to parse and filter JSON
        return responseBody; // Simplified
    }
}
```

---

## Infrastructure Example: Kong Gateway Configuration

```yaml
# kong.yml — Advanced routing and transformation with Kong

_format_version: "3.0"

services:
  # User Service with path-based routing
  - name: user-service
    url: http://user-service:8001
    routes:
      - name: user-api-route
        paths:
          - /api/v1/users
        methods:
          - GET
          - POST
          - PUT
        strip_path: false
    plugins:
      # Request transformation
      - name: request-transformer
        config:
          add:
            headers:
              - "X-Request-Source:api-gateway"
              - "X-Forwarded-Proto:https"
          remove:
            headers:
              - "X-Internal-Debug"   # Strip internal headers from external requests
          rename:
            headers:
              - "Authorization:X-Original-Auth"
      
      # Response transformation
      - name: response-transformer
        config:
          remove:
            headers:
              - "Server"
              - "X-Powered-By"
            json:
              - "internal_id"        # Remove internal fields from response
              - "db_metadata"
          add:
            headers:
              - "X-Gateway-Region:us-east-1"

  # Canary routing for Order Service
  - name: order-service-stable
    url: http://order-service-v1:8002
    routes:
      - name: order-stable-route
        paths:
          - /api/v1/orders
        headers:
          X-Canary:
            - "~*false"            # Match when NOT canary
    
  - name: order-service-canary
    url: http://order-service-v2:8003
    routes:
      - name: order-canary-route
        paths:
          - /api/v1/orders
        headers:
          X-Canary:
            - "~*true"             # Match canary header

upstreams:
  - name: user-service
    algorithm: round-robin
    targets:
      - target: user-service-1:8001
        weight: 100
      - target: user-service-2:8001
        weight: 100
      - target: user-service-3:8001
        weight: 100
```

---

## Real-World Example

### How Amazon Uses API Gateway Routing

Amazon handles millions of product queries per second. Their gateway routes based on:

```
Amazon's Routing Logic:

Request: GET /products/B01DFKC2SO

┌─────────────────────────────────────────────────────────────┐
│                  AMAZON API GATEWAY                          │
│                                                             │
│  Step 1: Region Detection                                   │
│  IP: 203.0.113.45 → Region: India → Route to IN datacenter │
│                                                             │
│  Step 2: Product Type Detection                            │
│  ASIN B01DFKC2SO → Category: Electronics                   │
│  → Route to electronics-catalog-service                    │
│                                                             │
│  Step 3: Personalization                                    │
│  User is Prime member → Include Prime-exclusive pricing    │
│  → Also call pricing-service with prime=true               │
│                                                             │
│  Step 4: Aggregation                                       │
│  Combine: product details + pricing + reviews + "also      │
│  bought" + delivery estimate                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Result: ONE response with data from 5+ services
```

### How Shopify Uses Request Transformation

```
Shopify's GraphQL Gateway:

External GraphQL request → Gateway → Multiple REST backends

Client sends GraphQL:
  query {
    shop { name }
    products(first: 5) { title, price }
    customer { email }
  }

Gateway decomposes into REST calls:
  ┌─────────────────────────────────────┐
  │  GraphQL Parser                     │
  │                                     │
  │  "shop" field → GET /admin/shop.json│
  │  "products"   → GET /admin/products │
  │  "customer"   → GET /admin/customer │
  └─────────────────────────────────────┘

Gateway transforms:
  REST responses → Combined GraphQL response
  {
    "data": {
      "shop": { "name": "My Store" },
      "products": [{ "title": "T-Shirt", "price": "29.99" }],
      "customer": { "email": "bob@example.com" }
    }
  }
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Solution |
|---------|-------------|----------|
| Too many route rules | Hard to debug, slow matching | Use a hierarchical route structure, document well |
| Complex transformations in the gateway | Gateway becomes slow, hard to maintain | Keep transformations simple; complex logic belongs in services |
| Aggregation without timeouts | One slow service blocks entire response | Set per-service timeouts; return partial data if one fails |
| Not handling partial failures in aggregation | Client gets nothing if one of 5 services fails | Return available data + error info for failed parts |
| Path conflicts (overlapping routes) | Ambiguous routing, inconsistent behavior | Define clear priority order, test with edge cases |
| Hardcoding routes | Can't update without redeployment | Use dynamic routing from service registry |
| No circuit breaker on aggregation calls | Repeated calls to dead service adds latency | Circuit break per-service in aggregation |

---

## When to Use / When NOT to Use

### ✅ Use Routing & Transformation When:

- You have **microservices** with different URL structures
- Clients need **aggregated data** from multiple services in one call
- You're doing **API versioning** (v1 routes to old service, v2 to new)
- You need to **hide internal architecture** from external clients
- Different **client types** (mobile/web) need different response formats
- You're doing **canary deployments** or A/B testing

### ❌ Be Cautious When:

- **Simple monolith** — routing is just a reverse proxy, no transformation needed
- **Heavy transformation** — if you're doing complex business logic, put it in a service
- **High-frequency aggregation** — consider moving to BFF pattern (Chapter 8.7) instead of gateway aggregation
- **Real-time streaming** — WebSocket connections shouldn't go through complex transformation pipelines

---

## Key Takeaways

- **Routing** matches incoming requests to backend services using path, headers, method, weight, or host
- **Transformation** modifies requests/responses — adding headers, rewriting paths, filtering fields, converting formats
- **Aggregation** (API Composition) calls multiple services in parallel and returns a combined response
- Use a **trie-based route table** for fast O(n) matching where n = path segments
- For canary deployments, use **weight-based routing** (5% → new version, 95% → stable)
- Always add **timeouts and circuit breakers** to aggregation calls — never let one slow service block everything
- Keep gateway transformations **lightweight** — complex data manipulation belongs in dedicated services

---

## What's Next?

With routing, transformation, and aggregation covered, the next chapter addresses a critical operational challenge: **API Versioning Strategies** — how to evolve your API without breaking existing clients.

Next: [05-api-versioning.md](./05-api-versioning.md)
