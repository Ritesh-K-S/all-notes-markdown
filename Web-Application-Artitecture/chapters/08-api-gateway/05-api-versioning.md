# API Versioning Strategies

> **What you'll learn**: Why APIs need versioning, the different approaches (URL path, query parameter, header, content negotiation), how to manage multiple versions simultaneously, and how companies like Stripe and Twilio handle versioning at scale without breaking millions of integrations.

---

## Real-Life Analogy: The Restaurant Menu Overhaul

Imagine you run a popular restaurant. You want to redesign the entire menu — new dishes, new pricing, new layout.

**Bad approach:** One day, you swap all the menus overnight. Regular customers come in, can't find their favorite "Chicken Parmesan" (it's now called "Pollo alla Parmigiana"), and leave confused and angry.

**Good approach:** You introduce a "New Menu" alongside the old one. Regulars can still order from the old menu, while new customers get the new one. After 6 months, you retire the old menu (with advance notice).

This is **API versioning** — you change your API's structure or behavior without breaking existing clients who depend on the old format.

```
Without versioning:
  Day 1: API returns { "name": "Alice" }
  Day 2: API returns { "full_name": "Alice Smith" }   ← BREAKING CHANGE!
  
  All existing clients break instantly! 💥

With versioning:
  /v1/users/123 → { "name": "Alice" }           ← Still works!
  /v2/users/123 → { "full_name": "Alice Smith" } ← New format
  
  Old clients keep working. New clients use v2.
```

---

## Why API Versioning Matters

```
The Reality of API Evolution:

Year 1: Simple API
  GET /users → { "name": "Alice", "email": "alice@mail.com" }

Year 2: Need to support multiple emails
  GET /users → { "name": "Alice", "emails": ["alice@mail.com", "a@work.com"] }
                                   ↑
                     "email" (string) → "emails" (array) = BREAKING CHANGE

Year 3: Need to split name into first/last
  GET /users → { "first_name": "Alice", "last_name": "Smith", "emails": [...] }
                  ↑
                  "name" → "first_name" + "last_name" = BREAKING CHANGE

You can't avoid API evolution. You CAN avoid breaking existing clients.
```

**Breaking vs Non-Breaking Changes:**

| Non-Breaking (Safe) | Breaking (Dangerous) |
|---------------------|---------------------|
| Adding a new field to response | Removing a field from response |
| Adding a new optional parameter | Making an optional param required |
| Adding a new endpoint | Changing response data type |
| Adding new enum values | Renaming a field |
| Loosening validation | Tightening validation |
| Increasing rate limits | Decreasing rate limits |

---

## Versioning Strategies

### Strategy 1: URL Path Versioning ⭐ (Most Popular)

```
Format: /api/v{N}/resource

Examples:
  GET /api/v1/users/123     ← Version 1
  GET /api/v2/users/123     ← Version 2
  GET /api/v3/users/123     ← Version 3

How the gateway routes:

Client ──▶ GET /api/v1/users/123
                    │
                    ▼
           ┌──────────────────┐
           │   API GATEWAY    │
           │                  │
           │  /api/v1/* → user-service-v1 (port 8001)
           │  /api/v2/* → user-service-v2 (port 8002)
           │  /api/v3/* → user-service-v3 (port 8003)
           └──────────────────┘
```

**Pros:**
- Very clear and explicit — version is visible in every request
- Easy to cache (CDNs cache by URL)
- Simple to implement in any API gateway
- Easy to test (just change the URL)

**Cons:**
- URL "pollution" — version is in every URL
- Hard to share links (URL changes between versions)
- Client libraries need updating for every version bump

**Who uses it:** Google, Twitter/X, Facebook Graph API

### Strategy 2: Query Parameter Versioning

```
Format: /api/resource?version=N

Examples:
  GET /api/users/123?version=1
  GET /api/users/123?version=2
  GET /api/users/123                ← defaults to latest? or v1?

Gateway routing:
  Extract ?version param → route to appropriate service version
```

**Pros:**
- Base URL stays clean
- Easy to default to a version
- Optional parameter — can be omitted

**Cons:**
- Easy to forget (what happens with no version param?)
- Harder to cache (query params complicate CDN caching)
- Not very common in practice

**Who uses it:** Amazon (some APIs), Google (some APIs)

### Strategy 3: Header Versioning

```
Format: Custom header or Accept header

Option A — Custom Header:
  GET /api/users/123
  X-API-Version: 2

Option B — Accept Header (Content Negotiation):
  GET /api/users/123
  Accept: application/vnd.myapp.v2+json

Gateway routing:
  ┌──────────────────────────────────────────────────────┐
  │ Request:                                             │
  │   GET /api/users/123                                │
  │   Accept: application/vnd.myapp.v2+json             │
  │                                                      │
  │ Gateway reads Accept header:                         │
  │   "vnd.myapp.v2" → route to user-service-v2        │
  └──────────────────────────────────────────────────────┘
```

**Pros:**
- Clean URLs (version not in path)
- Follows REST/HTTP content negotiation standards
- URL stays stable across versions

**Cons:**
- Harder to test (can't just paste URL in browser)
- Headers are "invisible" — easy to forget or misconfigure
- More complex for API consumers to implement

**Who uses it:** GitHub API (Accept header), Stripe (Stripe-Version header)

### Strategy 4: Date-Based Versioning (Stripe's Approach) ⭐

```
Format: Version = date when API behavior was locked

  GET /v1/charges
  Stripe-Version: 2023-10-16     ← "Give me the API as it behaved on Oct 16, 2023"

How it works internally:
  ┌─────────────────────────────────────────────────────────┐
  │                     API SERVICE                          │
  │                                                         │
  │  Stripe-Version: 2023-10-16                            │
  │                                                         │
  │  Change log:                                           │
  │  ├── 2024-01-15: "amount" field renamed to "total"    │
  │  ├── 2023-11-01: "source" deprecated, use "payment_method" │
  │  └── 2023-10-16: baseline                              │
  │                                                         │
  │  For version 2023-10-16:                               │
  │  → Apply compatibility transforms for all changes      │
  │    made AFTER this date                                │
  │  → Return response in the format that existed on       │
  │    2023-10-16                                          │
  └─────────────────────────────────────────────────────────┘
```

**Pros:**
- Extremely precise — client knows exactly what behavior to expect
- No "v2 vs v3" confusion — just pick a date
- Stripe can make frequent changes without bumping major versions
- Gradual migration — each change is independent

**Cons:**
- Complex to implement (must maintain compatibility layers for every date)
- Hard for API consumers to know which date gives which features
- Operational burden to test all versions

**Who uses it:** Stripe (the gold standard of API versioning)

---

## Comparison Table

```
┌──────────────────┬────────────┬──────────────┬───────────────┬──────────────┐
│                  │ URL Path   │ Query Param  │ Header        │ Date-Based   │
│                  │ /v1/users  │ ?version=1   │ X-API-V: 1    │ 2023-10-16   │
├──────────────────┼────────────┼──────────────┼───────────────┼──────────────┤
│ Visibility       │ ⭐⭐⭐     │ ⭐⭐        │ ⭐            │ ⭐           │
│ Cache-friendly   │ ⭐⭐⭐     │ ⭐          │ ⭐⭐          │ ⭐⭐         │
│ Clean URLs       │ ⭐         │ ⭐⭐        │ ⭐⭐⭐        │ ⭐⭐⭐       │
│ Ease of testing  │ ⭐⭐⭐     │ ⭐⭐⭐      │ ⭐            │ ⭐           │
│ Implementation   │ Simple     │ Simple       │ Moderate      │ Complex      │
│ Granularity      │ Coarse     │ Coarse       │ Fine          │ Very Fine    │
│ Industry Usage   │ Most common│ Rare         │ Common        │ Stripe only  │
└──────────────────┴────────────┴──────────────┴───────────────┴──────────────┘
```

---

## How It Works Internally

### Version Routing at the Gateway

```
Gateway's Version Resolution Process:

  ┌─────────────────────────────────────────────────────────┐
  │              VERSION RESOLUTION                          │
  │                                                         │
  │  1. Check URL path: /api/v2/users → version = 2        │
  │     OR                                                  │
  │  2. Check header: X-API-Version: 2 → version = 2       │
  │     OR                                                  │
  │  3. Check query: ?version=2 → version = 2              │
  │     OR                                                  │
  │  4. Use default: configured default version             │
  │                                                         │
  │  Once version is determined:                            │
  │                                                         │
  │  Option A: Route to different service instance          │
  │  ┌────────────────────────────────────────┐            │
  │  │ v1 → user-service-v1 (old code)       │            │
  │  │ v2 → user-service-v2 (new code)       │            │
  │  └────────────────────────────────────────┘            │
  │                                                         │
  │  Option B: Route to same service, pass version header  │
  │  ┌────────────────────────────────────────┐            │
  │  │ All versions → user-service            │            │
  │  │ Service reads X-API-Version header     │            │
  │  │ and adjusts response format            │            │
  │  └────────────────────────────────────────┘            │
  └─────────────────────────────────────────────────────────┘
```

### Managing Multiple Versions (Version Lifecycle)

```
Version Lifecycle:

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  ALPHA   │───▶│  BETA    │───▶│  STABLE  │───▶│  SUNSET  │──▶ RETIRED
  │          │    │          │    │          │    │          │
  │ Internal │    │ Partner  │    │ General  │    │ Warning  │
  │ testing  │    │ testing  │    │ available│    │ headers  │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘

Timeline example:
  v1: STABLE (Jan 2022) → SUNSET (Jan 2024) → RETIRED (Jul 2024)
  v2: STABLE (Jun 2023) → current
  v3: BETA (Jan 2024) → STABLE (Mar 2024)

Sunset response headers:
  HTTP/1.1 200 OK
  Sunset: Sat, 01 Jul 2024 00:00:00 GMT
  Deprecation: true
  Link: <https://docs.myapp.com/migration-v2>; rel="successor-version"
```

---

## Code Examples

### Python: API Versioning at the Gateway

```python
# versioned_gateway.py
# API Gateway that handles multiple versioning strategies

from flask import Flask, request, jsonify
import requests as http_client

app = Flask(__name__)

# Version → service mapping
VERSION_MAP = {
    "v1": {
        "users": "http://user-service-v1:8001",
        "orders": "http://order-service-v1:8002",
    },
    "v2": {
        "users": "http://user-service-v2:8011",
        "orders": "http://order-service-v2:8012",
    },
}

# Version lifecycle status
VERSION_STATUS = {
    "v1": {"status": "sunset", "sunset_date": "2024-07-01",
            "successor": "v2", "docs": "https://docs.myapp.com/migrate-v2"},
    "v2": {"status": "stable"},
    "v3": {"status": "beta", "note": "Not for production use"},
}

DEFAULT_VERSION = "v2"


def resolve_version(req):
    """Determine API version from request using multiple strategies."""
    # Strategy 1: URL path (/api/v2/users)
    path_parts = req.path.split("/")
    for part in path_parts:
        if part.startswith("v") and part[1:].isdigit():
            return part

    # Strategy 2: Custom header (X-API-Version: v2)
    header_version = req.headers.get("X-API-Version")
    if header_version:
        return header_version

    # Strategy 3: Accept header (application/vnd.myapp.v2+json)
    accept = req.headers.get("Accept", "")
    if "vnd.myapp." in accept:
        # Extract version from "application/vnd.myapp.v2+json"
        import re
        match = re.search(r"vnd\.myapp\.(v\d+)", accept)
        if match:
            return match.group(1)

    # Strategy 4: Query parameter (?version=v2)
    query_version = req.args.get("version")
    if query_version:
        return query_version

    # Default
    return DEFAULT_VERSION


def add_sunset_headers(response, version):
    """Add deprecation/sunset headers for old API versions."""
    status = VERSION_STATUS.get(version, {})
    
    if status.get("status") == "sunset":
        response.headers["Sunset"] = status["sunset_date"]
        response.headers["Deprecation"] = "true"
        response.headers["Link"] = (
            f'<{status["docs"]}>; rel="successor-version"'
        )
        # Also add a warning header
        response.headers["Warning"] = (
            f'299 - "API {version} is deprecated. '
            f'Please migrate to {status["successor"]}. '
            f'Sunset date: {status["sunset_date"]}"'
        )

    if status.get("status") == "beta":
        response.headers["Warning"] = (
            f'199 - "API {version} is in beta. Not for production use."'
        )

    return response


@app.route("/api/<path:path>", methods=["GET", "POST", "PUT", "DELETE"])
def versioned_route(path):
    """Route requests based on resolved API version."""
    version = resolve_version(request)
    
    # Validate version exists
    if version not in VERSION_MAP:
        return jsonify({
            "error": "invalid_version",
            "message": f"Version '{version}' not found. Available: {list(VERSION_MAP.keys())}",
        }), 400

    # Check if version is retired
    status = VERSION_STATUS.get(version, {})
    if status.get("status") == "retired":
        return jsonify({
            "error": "version_retired",
            "message": f"Version '{version}' has been retired.",
            "migration_guide": status.get("docs"),
        }), 410  # HTTP 410 Gone

    # Determine the service to route to
    # Strip version prefix from path if present
    clean_path = path
    if clean_path.startswith(f"{version}/"):
        clean_path = clean_path[len(f"{version}/"):]

    # Find service for this resource
    resource = clean_path.split("/")[0]  # e.g., "users" from "users/123"
    service_url = VERSION_MAP[version].get(resource)

    if not service_url:
        return jsonify({"error": "route_not_found"}), 404

    # Forward request
    backend_url = f"{service_url}/{clean_path}"
    resp = http_client.request(
        method=request.method,
        url=backend_url,
        headers={k: v for k, v in request.headers if k != "Host"},
        data=request.get_data(),
    )

    # Build response with version headers
    response = jsonify(resp.json())
    response.status_code = resp.status_code
    response.headers["X-API-Version"] = version
    response = add_sunset_headers(response, version)

    return response


if __name__ == "__main__":
    app.run(port=8000)
```

### Java: API Versioning with Spring Cloud Gateway

```java
// VersionRoutingConfig.java
// Dynamic version-based routing with sunset headers

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.http.server.reactive.ServerHttpResponse;
import reactor.core.publisher.Mono;

@Configuration
public class VersionRoutingConfig {

    @Bean
    public RouteLocator versionedRoutes(RouteLocatorBuilder builder) {
        return builder.routes()

            // V1 routes (sunset — will be retired)
            .route("users-v1", r -> r
                .path("/api/v1/users/**")
                .filters(f -> f
                    .stripPrefix(2)  // Remove /api/v1
                    .filter(addSunsetHeaders("2024-07-01", "v2"))
                )
                .uri("http://user-service-v1:8001"))

            // V2 routes (current stable)
            .route("users-v2", r -> r
                .path("/api/v2/users/**")
                .filters(f -> f
                    .stripPrefix(2)  // Remove /api/v2
                    .addResponseHeader("X-API-Version", "v2")
                )
                .uri("http://user-service-v2:8011"))

            // Header-based versioning (alternative)
            .route("users-header-v2", r -> r
                .path("/api/users/**")
                .and().header("X-API-Version", "v2")
                .filters(f -> f
                    .stripPrefix(1)  // Remove /api
                    .addResponseHeader("X-API-Version", "v2")
                )
                .uri("http://user-service-v2:8011"))

            // Default (no version specified) → latest stable
            .route("users-default", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addResponseHeader("X-API-Version", "v2")
                    .addResponseHeader("X-Default-Version", "true")
                )
                .uri("http://user-service-v2:8011"))

            .build();
    }

    /**
     * Creates a filter that adds sunset/deprecation headers to responses.
     */
    private GatewayFilter addSunsetHeaders(String sunsetDate, String successor) {
        return (exchange, chain) -> chain.filter(exchange).then(Mono.fromRunnable(() -> {
            ServerHttpResponse response = exchange.getResponse();
            response.getHeaders().add("Sunset", sunsetDate);
            response.getHeaders().add("Deprecation", "true");
            response.getHeaders().add("X-API-Version", "v1");
            response.getHeaders().add("Warning",
                "299 - \"This API version is deprecated. " +
                "Migrate to " + successor + " before " + sunsetDate + "\"");
        }));
    }
}
```

---

## Infrastructure Example: Nginx Version Routing

```nginx
# nginx.conf — API versioning with Nginx

# Upstream definitions for each version
upstream user_service_v1 {
    server user-svc-v1-1:8001;
    server user-svc-v1-2:8001;
}

upstream user_service_v2 {
    server user-svc-v2-1:8011;
    server user-svc-v2-2:8011;
    server user-svc-v2-3:8011;
}

# Map header to version (for header-based versioning)
map $http_x_api_version $api_version {
    default "v2";      # Default version if no header
    "v1"    "v1";
    "v2"    "v2";
}

server {
    listen 443 ssl;
    server_name api.myapp.com;

    # URL path-based versioning
    location /api/v1/users {
        # Add sunset headers for deprecated version
        add_header Sunset "2024-07-01" always;
        add_header Deprecation "true" always;
        add_header Warning '299 - "v1 is deprecated, use v2"' always;
        
        proxy_pass http://user_service_v1/users;
        proxy_set_header Host $host;
    }

    location /api/v2/users {
        add_header X-API-Version "v2" always;
        
        proxy_pass http://user_service_v2/users;
        proxy_set_header Host $host;
    }

    # Header-based versioning (same URL, different version via header)
    location /api/users {
        # Route based on X-API-Version header
        if ($api_version = "v1") {
            proxy_pass http://user_service_v1/users;
            break;
        }
        # Default to v2
        proxy_pass http://user_service_v2/users;
        proxy_set_header Host $host;
        add_header X-API-Version $api_version always;
    }

    # Version discovery endpoint
    location /api/versions {
        default_type application/json;
        return 200 '{
            "versions": [
                {"version": "v1", "status": "sunset", "sunset_date": "2024-07-01"},
                {"version": "v2", "status": "stable", "released": "2023-06-01"},
                {"version": "v3", "status": "beta", "docs": "/api/v3/docs"}
            ],
            "default": "v2"
        }';
    }
}
```

---

## Real-World Example

### How Stripe Handles API Versioning

Stripe is considered the **gold standard** for API versioning. They serve millions of businesses and can never break integrations.

```
Stripe's Date-Based Versioning:

  Every Stripe account is pinned to the API version
  that existed when they signed up.

  Account created on 2022-03-15:
    → Automatically uses API version "2022-02-15" (nearest release)
    → All responses formatted as of that date

  Stripe makes ~15-20 API changes per year:
    2024-01-15: "source" field removed
    2023-11-01: "payment_intent" response structure changed
    2023-08-15: Pagination changed from offset to cursor
    ...

  How it works internally:
  ┌──────────────────────────────────────────────────────────┐
  │                  STRIPE API SERVER                        │
  │                                                          │
  │  Request: GET /v1/charges/ch_123                        │
  │  Stripe-Version: 2022-02-15                             │
  │                                                          │
  │  1. Fetch charge data from database (latest format)     │
  │  2. Look up ALL changes between 2022-02-15 and today    │
  │  3. Apply REVERSE transforms for each change:           │
  │     ├── Un-apply change from 2024-01-15                 │
  │     ├── Un-apply change from 2023-11-01                 │
  │     └── Un-apply change from 2023-08-15                 │
  │  4. Return response in 2022-02-15 format                │
  │                                                          │
  └──────────────────────────────────────────────────────────┘

  Merchants can upgrade at their own pace:
    1. Read the changelog for their current version → latest
    2. Test in sandbox with the new version
    3. Update their Stripe-Version header (or dashboard setting)
```

**Why this works brilliantly:**
- Merchants are NEVER broken — they stay on their version until they choose to upgrade
- Stripe can innovate rapidly without coordinating with millions of merchants
- Each change is small and independent (not big "v2 → v3" migrations)

### How Twilio Handles Versioning

```
Twilio uses date-based URL paths:

  https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Messages

  The "2010-04-01" is the API version date!

  When Twilio releases a new version:
    https://api.twilio.com/2024-03-15/Accounts/{AccountSid}/Messages

  Old URLs keep working FOREVER.
  New features only available on new date paths.
```

---

## Advanced: Compatibility Layer Pattern

```
Instead of running multiple service versions, use ONE service with
a compatibility transformation layer:

Client ──▶ Gateway ──▶ ┌─────────────────────────────────┐
                       │       Version Compatibility      │
                       │            Layer                 │
                       │                                 │
                       │  v1 request → transform → v3   │
                       │  v2 request → transform → v3   │
                       │  v3 request → pass through     │
                       │                                 │
                       └──────────────┬──────────────────┘
                                      │
                                      ▼
                              ┌──────────────┐
                              │ Service v3   │
                              │ (latest code)│
                              └──────────────┘
                                      │
                       ┌──────────────┴──────────────────┐
                       │       Response Compatibility     │
                       │            Layer                 │
                       │                                 │
                       │  v3 response → transform → v1  │
                       │  v3 response → transform → v2  │
                       │  v3 response → pass through    │
                       └─────────────────────────────────┘

Benefit: Only ONE service to maintain and deploy!
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Solution |
|---------|-------------|----------|
| No versioning from day one | Adding it later is painful (existing clients have no version) | Always start with v1, even for your first API |
| Too many active versions | Each version multiplies testing/maintenance burden | Limit to 2-3 active versions; sunset aggressively |
| No sunset timeline | Old versions live forever, increasing tech debt | Announce sunset dates upfront (6-12 months notice) |
| Breaking changes in minor versions | Clients don't expect breaking changes in v1.2 vs v1.1 | Breaking = new major version, always |
| Not documenting changes | Clients don't know what changed between versions | Maintain a detailed changelog with migration guides |
| Versioning too frequently | Clients are confused, always behind | Version only for breaking changes; additive changes don't need new versions |
| No default version | Clients without version param get unpredictable results | Always have a stable default, and never change it silently |

---

## When to Use Each Strategy

### ✅ Use URL Path Versioning When:

- You want maximum **clarity and visibility**
- Your API is **public** and used by many third parties
- You want easy **CDN caching**
- You prefer **simplicity** over elegance

### ✅ Use Header Versioning When:

- You want **clean, stable URLs**
- Your clients are **SDKs/libraries** (easy to set headers programmatically)
- You're following **REST purist** practices
- You want to **evolve the API** without URL changes

### ✅ Use Date-Based Versioning When:

- You make **frequent, small changes** to the API
- You have **thousands of integrators** who upgrade at different paces
- You want each merchant/client to be **pinned to a stable experience**
- You can invest in building a **compatibility layer**

### ❌ Don't Version When:

- The change is **additive** (new optional fields) — just add them
- It's an **internal API** with only a few consumers you control
- You're in **very early stage** (MVP) — just iterate fast and coordinate with clients directly

---

## Key Takeaways

- **API versioning prevents breaking changes** from disrupting existing clients — it's a contract
- **URL path versioning** (/api/v1/) is the most common and easiest to understand
- **Header versioning** keeps URLs clean but is harder to test and discover
- **Date-based versioning** (Stripe's approach) is the most granular but hardest to implement
- Always **start with v1** even on day one — retrofitting versioning is painful
- Limit **active versions to 2-3** and sunset old versions with 6-12 months notice
- Use **sunset headers** (HTTP `Sunset`, `Deprecation`) to communicate deprecation in-band
- The **compatibility layer** pattern lets you run one service and transform responses per version

---

## What's Next?

Now that you understand how to version your APIs cleanly, the next chapter explores the actual tools that power API gateways in production: **Tools — Kong, AWS API Gateway, Apigee, Zuul, Spring Cloud Gateway** — a hands-on comparison of the most popular options.

Next: [06-api-gateway-tools.md](./06-api-gateway-tools.md)
