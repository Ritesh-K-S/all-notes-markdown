# Chapter 50 — API Gateway & Apigee

---

## Table of Contents

- [Overview](#overview)
- [Part 1: API Management Fundamentals](#part-1--api-management-fundamentals)
- [Part 2: API Gateway — Overview](#part-2--api-gateway--overview)
- [Part 3: API Gateway — API Config (OpenAPI)](#part-3--api-gateway--api-config-openapi)
- [Part 4: API Gateway — Deploying & Managing](#part-4--api-gateway--deploying--managing)
- [Part 5: API Gateway — Authentication](#part-5--api-gateway--authentication)
- [Part 6: API Gateway — Rate Limiting & Quotas](#part-6--api-gateway--rate-limiting--quotas)
- [Part 7: Apigee — Overview & Architecture](#part-7--apigee--overview--architecture)
- [Part 8: Apigee — API Proxies](#part-8--apigee--api-proxies)
- [Part 9: Apigee — Policies](#part-9--apigee--policies)
- [Part 10: Apigee — Developer Portal & API Products](#part-10--apigee--developer-portal--api-products)
- [Part 11: Apigee — Analytics & Monitoring](#part-11--apigee--analytics--monitoring)
- [Part 12: Apigee — Security Policies](#part-12--apigee--security-policies)
- [Part 13: API Gateway vs Apigee — When to Use Which](#part-13--api-gateway-vs-apigee--when-to-use-which)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud provides two API management services: **API Gateway** — a lightweight, serverless gateway for exposing backend services as managed APIs with authentication, rate limiting, and monitoring; and **Apigee** — an enterprise-grade, full-lifecycle API management platform with advanced policies, developer portals, analytics, monetization, and API product packaging. API Gateway is for simple API fronting, while Apigee is for organizations that treat APIs as products.

---

## Part 1 — API Management Fundamentals

### What Is API Management?

```
┌─────────────────────────────────────────────────────────────────┐
│              API MANAGEMENT                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  API management sits between API consumers and your backend:    │
│                                                                   │
│  ┌──────────┐    ┌──────────────────┐    ┌──────────────┐      │
│  │ Mobile   │    │                  │    │              │      │
│  │ App      │───►│   API Gateway /  │───►│  Backend     │      │
│  ├──────────┤    │   Apigee         │    │  Services    │      │
│  │ Web App  │───►│                  │───►│  (Cloud Run, │      │
│  ├──────────┤    │  • Auth          │    │   Functions, │      │
│  │ Partner  │───►│  • Rate limiting │    │   GKE, etc.) │      │
│  │ API      │    │  • Monitoring    │    │              │      │
│  └──────────┘    │  • Transforms    │    └──────────────┘      │
│                  │  • Caching       │                           │
│                  │  • Analytics     │                           │
│                  └──────────────────┘                           │
│                                                                   │
│  Capabilities:                                                   │
│  ├── Authentication & authorization                             │
│  ├── Rate limiting & quotas                                     │
│  ├── Request/response transformation                            │
│  ├── API versioning                                              │
│  ├── Monitoring, logging & analytics                            │
│  ├── Developer portal & documentation                           │
│  ├── API key management                                          │
│  └── Traffic management & caching                               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP API Gateway | GCP Apigee | AWS API Gateway | Azure API Management |
|---------|----------------|-----------|-----------------|---------------------|
| Type | Lightweight | Enterprise | Full-featured | Enterprise |
| Serverless | Yes | Hybrid | Yes | No (dedicated) |
| OpenAPI spec | Yes | Yes | Yes | Yes |
| Developer portal | No | Yes | No (use portal) | Yes |
| Analytics | Basic (Cloud Monitoring) | Advanced | CloudWatch | Built-in |
| Monetization | No | Yes | No | Yes |
| API products | No | Yes | Usage plans | Products |
| Policy engine | No | Yes (extensive) | Limited | Yes |
| Caching | No | Yes | Yes | Yes |
| Pricing | Per-call | Subscription + per-call | Per-call | Tier-based |

---

## Part 2 — API Gateway — Overview

### API Gateway Architecture

```
┌──────────────────────────────────────────────────────────────┐
│         API GATEWAY                                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Serverless API front door for your backends:                │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Components:                                          │    │
│  │                                                      │    │
│  │  API                                                 │    │
│  │  └── API Config (OpenAPI spec)                       │    │
│  │      └── Gateway (deployment)                        │    │
│  │          └── endpoint URL                            │    │
│  │                                                      │    │
│  │  API = logical container                             │    │
│  │  API Config = OpenAPI spec defining routes + backend │    │
│  │  Gateway = deployed instance serving traffic         │    │
│  │  Endpoint = public URL for consumers                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Supported backends:                                           │
│  • Cloud Functions (HTTP trigger)                             │
│  • Cloud Run (HTTPS URL)                                      │
│  • App Engine                                                  │
│  • Any HTTP endpoint                                          │
│  • GKE (via Ingress URL)                                      │
│                                                                │
│  Features:                                                     │
│  • OpenAPI 2.0 (Swagger) spec                                │
│  • API key validation                                         │
│  • JWT validation (Firebase, Auth0, Google)                   │
│  • Automatic Cloud Monitoring metrics                         │
│  • Automatic Cloud Logging                                    │
│  • Per-API rate limiting                                      │
│  • Serverless (no infra to manage)                           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Pricing (API Gateway)

| Component | Cost |
|-----------|------|
| API calls | First 2M/month free, then $3/million |
| Data transfer | Standard GCP egress rates |
| No per-gateway charge | Free |

---

## Part 3 — API Gateway — API Config (OpenAPI)

### OpenAPI Specification

```yaml
# openapi-spec.yaml
swagger: "2.0"
info:
  title: "Orders API"
  version: "1.0.0"
  description: "API for managing orders"

host: "my-api-gateway-abc123.apigateway.my-project.cloud.goog"
basePath: "/"

schemes:
  - "https"

produces:
  - "application/json"

paths:
  /orders:
    get:
      summary: "List all orders"
      operationId: "listOrders"
      x-google-backend:
        address: "https://orders-service-abc123.run.app/orders"
      security:
        - api_key: []
      responses:
        200:
          description: "List of orders"

    post:
      summary: "Create an order"
      operationId: "createOrder"
      x-google-backend:
        address: "https://orders-service-abc123.run.app/orders"
      security:
        - jwt_auth: []
      responses:
        201:
          description: "Order created"

  /orders/{orderId}:
    get:
      summary: "Get order by ID"
      operationId: "getOrder"
      parameters:
        - name: orderId
          in: path
          required: true
          type: string
      x-google-backend:
        address: "https://orders-service-abc123.run.app/orders/{orderId}"
        path_translation: APPEND_PATH_TO_ADDRESS
      security:
        - api_key: []
      responses:
        200:
          description: "Order details"

  /health:
    get:
      summary: "Health check"
      operationId: "healthCheck"
      x-google-backend:
        address: "https://orders-service-abc123.run.app/health"
      # No security — public endpoint
      responses:
        200:
          description: "OK"

securityDefinitions:
  api_key:
    type: "apiKey"
    name: "x-api-key"
    in: "header"

  jwt_auth:
    authorizationUrl: ""
    flow: "implicit"
    type: "oauth2"
    x-google-issuer: "https://accounts.google.com"
    x-google-jwks_uri: "https://www.googleapis.com/oauth2/v3/certs"
    x-google-audiences: "my-api-client-id.apps.googleusercontent.com"
```

### Key x-google Extensions

| Extension | Purpose |
|-----------|---------|
| `x-google-backend.address` | Backend service URL |
| `x-google-backend.path_translation` | `APPEND_PATH_TO_ADDRESS` or `CONSTANT_ADDRESS` |
| `x-google-backend.deadline` | Backend timeout (seconds) |
| `x-google-issuer` | JWT issuer URL |
| `x-google-jwks_uri` | JWT public keys URL |
| `x-google-audiences` | Expected JWT audience |
| `x-google-management.quota` | Rate limiting config |

---

## Part 4 — API Gateway — Deploying & Managing

### Deployment Flow

```bash
# Step 1: Create the API
gcloud api-gateway apis create orders-api \
    --project=my-project

# Step 2: Create an API config (upload OpenAPI spec)
gcloud api-gateway api-configs create orders-config-v1 \
    --api=orders-api \
    --openapi-spec=openapi-spec.yaml \
    --project=my-project \
    --backend-auth-service-account=gateway-sa@my-project.iam.gserviceaccount.com

# Step 3: Create a gateway (deploy)
gcloud api-gateway gateways create orders-gateway \
    --api=orders-api \
    --api-config=orders-config-v1 \
    --location=us-central1 \
    --project=my-project

# Get the gateway URL
gcloud api-gateway gateways describe orders-gateway \
    --location=us-central1 \
    --project=my-project \
    --format="value(defaultHostname)"
# Output: orders-gateway-abc123.uc.gateway.dev

# Step 4: Test it
curl https://orders-gateway-abc123.uc.gateway.dev/health

# Update to new config version
gcloud api-gateway api-configs create orders-config-v2 \
    --api=orders-api \
    --openapi-spec=openapi-spec-v2.yaml \
    --project=my-project \
    --backend-auth-service-account=gateway-sa@my-project.iam.gserviceaccount.com

gcloud api-gateway gateways update orders-gateway \
    --api=orders-api \
    --api-config=orders-config-v2 \
    --location=us-central1 \
    --project=my-project
```

---

## Part 5 — API Gateway — Authentication

### API Key Authentication

```yaml
# In OpenAPI spec
securityDefinitions:
  api_key:
    type: "apiKey"
    name: "x-api-key"
    in: "header"

paths:
  /orders:
    get:
      security:
        - api_key: []
      x-google-backend:
        address: "https://orders-service.run.app/orders"
```

```bash
# Create API key in Google Cloud Console
# APIs & Services → Credentials → Create API Key

# Test with API key
curl -H "x-api-key: AIzaSyB..." \
    https://orders-gateway.uc.gateway.dev/orders
```

### JWT Authentication

```yaml
# In OpenAPI spec — Google Identity
securityDefinitions:
  google_jwt:
    authorizationUrl: ""
    flow: "implicit"
    type: "oauth2"
    x-google-issuer: "https://accounts.google.com"
    x-google-jwks_uri: "https://www.googleapis.com/oauth2/v3/certs"
    x-google-audiences: "CLIENT_ID.apps.googleusercontent.com"

# Firebase Auth
securityDefinitions:
  firebase:
    authorizationUrl: ""
    flow: "implicit"
    type: "oauth2"
    x-google-issuer: "https://securetoken.google.com/PROJECT_ID"
    x-google-jwks_uri: "https://www.googleapis.com/service_accounts/v1/metadata/x509/securetoken@system.gserviceaccount.com"
    x-google-audiences: "PROJECT_ID"

# Auth0
securityDefinitions:
  auth0:
    authorizationUrl: ""
    flow: "implicit"
    type: "oauth2"
    x-google-issuer: "https://YOUR_DOMAIN.auth0.com/"
    x-google-jwks_uri: "https://YOUR_DOMAIN.auth0.com/.well-known/jwks.json"
    x-google-audiences: "YOUR_API_IDENTIFIER"
```

---

## Part 6 — API Gateway — Rate Limiting & Quotas

### Configuring Quotas

```yaml
# In OpenAPI spec
x-google-management:
  metrics:
    - name: "read-requests"
      displayName: "Read requests"
      valueType: INT64
      metricKind: DELTA
    - name: "write-requests"
      displayName: "Write requests"
      valueType: INT64
      metricKind: DELTA
  quota:
    limits:
      - name: "read-limit"
        metric: "read-requests"
        unit: "1/min/{project}"
        values:
          STANDARD: 1000  # 1000 reads per minute per project
      - name: "write-limit"
        metric: "write-requests"
        unit: "1/min/{project}"
        values:
          STANDARD: 100  # 100 writes per minute per project

paths:
  /orders:
    get:
      x-google-quota:
        metricCosts:
          read-requests: 1
    post:
      x-google-quota:
        metricCosts:
          write-requests: 1
```

---

## Part 7 — Apigee — Overview & Architecture

### Apigee Architecture

```
┌──────────────────────────────────────────────────────────────┐
│         APIGEE ARCHITECTURE                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                    APIGEE                              │    │
│  │                                                      │    │
│  │  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │    │
│  │  │ Management │  │ Runtime    │  │ Analytics    │  │    │
│  │  │ Plane      │  │ Plane      │  │ Plane        │  │    │
│  │  │            │  │            │  │              │  │    │
│  │  │ API design │  │ API Proxy  │  │ Traffic      │  │    │
│  │  │ Config     │  │ execution  │  │ Performance  │  │    │
│  │  │ Portal     │  │ Policies   │  │ Developer    │  │    │
│  │  │ Products   │  │ Routing    │  │ Engagement   │  │    │
│  │  │            │  │ Caching    │  │ Custom       │  │    │
│  │  └────────────┘  └────────────┘  └──────────────┘  │    │
│  │                                                      │    │
│  │  Environments: dev → test → staging → prod          │    │
│  │                                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Apigee Editions:                                              │
│  ┌──────────────────────┬──────────────────────────────┐     │
│  │ Apigee X             │ Apigee hybrid               │     │
│  │ (fully managed)      │ (management in cloud,       │     │
│  │ Runtime on Google    │  runtime on-prem/GKE)       │     │
│  │ VPC peering          │ For regulatory/latency       │     │
│  │ Recommended          │ requirements                 │     │
│  └──────────────────────┴──────────────────────────────┘     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Pricing (Apigee)

| Plan | Cost | Included |
|------|------|----------|
| Evaluation | Free (60 days) | Limited features |
| Standard | ~$500/month | Basic features |
| Enterprise | Custom | Full features |
| Enterprise Plus | Custom | Premium support + SLA |

---

## Part 8 — Apigee — API Proxies

### API Proxy Structure

```
┌──────────────────────────────────────────────────────────────┐
│         API PROXY FLOW                                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Client request → Apigee Proxy → Backend                     │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  API Proxy: "orders-api"                              │    │
│  │                                                      │    │
│  │  ProxyEndpoint (client-facing):                      │    │
│  │  ┌──────────────────────────────────────────────┐   │    │
│  │  │ PreFlow:                                      │   │    │
│  │  │   → Verify API Key                           │   │    │
│  │  │   → Spike Arrest (rate limit)                │   │    │
│  │  │   → Quota enforcement                        │   │    │
│  │  │                                               │   │    │
│  │  │ Conditional Flows:                            │   │    │
│  │  │   → GET /orders → read policies              │   │    │
│  │  │   → POST /orders → write policies            │   │    │
│  │  │                                               │   │    │
│  │  │ PostFlow:                                     │   │    │
│  │  │   → Transform response                       │   │    │
│  │  │   → Set CORS headers                         │   │    │
│  │  └──────────────────────────────────────────────┘   │    │
│  │                                                      │    │
│  │  TargetEndpoint (backend-facing):                   │    │
│  │  ┌──────────────────────────────────────────────┐   │    │
│  │  │ PreFlow:                                      │   │    │
│  │  │   → Set auth headers for backend              │   │    │
│  │  │                                               │   │    │
│  │  │ Target URL:                                   │   │    │
│  │  │   → https://orders-service.run.app            │   │    │
│  │  │                                               │   │    │
│  │  │ PostFlow:                                     │   │    │
│  │  │   → Parse backend response                    │   │    │
│  │  └──────────────────────────────────────────────┘   │    │
│  │                                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 9 — Apigee — Policies

### Common Policies

```
┌──────────────────────────────────────────────────────────────┐
│         APIGEE POLICIES                                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Security:                                                     │
│  ├── VerifyAPIKey — validate API key in request              │
│  ├── OAuthV2 — generate/validate OAuth tokens                │
│  ├── JWT — generate/verify/decode JWTs                       │
│  ├── BasicAuthentication — basic auth encode/decode          │
│  ├── HMAC — compute/verify HMAC signatures                   │
│  └── CORS — set Cross-Origin Resource Sharing headers        │
│                                                                │
│  Traffic Management:                                           │
│  ├── SpikeArrest — prevent traffic spikes                    │
│  ├── Quota — enforce usage limits per time period            │
│  ├── ConcurrentRateLimit — limit concurrent connections      │
│  └── ResponseCache — cache backend responses                 │
│                                                                │
│  Mediation:                                                    │
│  ├── AssignMessage — modify request/response                 │
│  ├── XMLToJSON / JSONToXML — format conversion               │
│  ├── ExtractVariables — parse data from messages             │
│  ├── XSLTransform — XSLT transformations                    │
│  └── MessageLogging — send logs to external systems          │
│                                                                │
│  Extension:                                                    │
│  ├── JavaScript — custom JavaScript logic                    │
│  ├── Python — custom Python scripts                          │
│  ├── JavaCallout — custom Java logic                         │
│  ├── ServiceCallout — call external services mid-flow        │
│  └── RaiseFault — generate custom error responses            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Policy Example — Spike Arrest + Quota

```xml
<!-- SpikeArrest — prevent traffic spikes -->
<SpikeArrest name="SA-RateLimit">
    <Rate>100ps</Rate>  <!-- 100 per second -->
    <Identifier ref="request.header.x-api-key"/>
</SpikeArrest>

<!-- Quota — enforce monthly limits per API key -->
<Quota name="Q-MonthlyLimit">
    <Identifier ref="request.header.x-api-key"/>
    <Allow count="10000"/>
    <Interval>1</Interval>
    <TimeUnit>month</TimeUnit>
</Quota>

<!-- Response Cache -->
<ResponseCache name="RC-GetOrders">
    <CacheKey>
        <KeyFragment ref="request.uri"/>
    </CacheKey>
    <ExpirySettings>
        <TimeoutInSec>300</TimeoutInSec>
    </ExpirySettings>
</ResponseCache>
```

---

## Part 10 — Apigee — Developer Portal & API Products

### API Products

```
┌──────────────────────────────────────────────────────────────┐
│         API PRODUCTS                                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  API Products bundle APIs into packages for consumers:       │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Product: "Free Tier"                                 │    │
│  │  ├── API Proxies: orders-api (GET only)              │    │
│  │  ├── Quota: 1,000 calls/month                        │    │
│  │  ├── Rate limit: 10 calls/minute                     │    │
│  │  └── Price: Free                                     │    │
│  │                                                      │    │
│  │  Product: "Pro Tier"                                  │    │
│  │  ├── API Proxies: orders-api (all methods)           │    │
│  │  ├── Quota: 100,000 calls/month                      │    │
│  │  ├── Rate limit: 100 calls/minute                    │    │
│  │  └── Price: $99/month                                │    │
│  │                                                      │    │
│  │  Product: "Enterprise Tier"                           │    │
│  │  ├── API Proxies: orders-api + analytics-api          │    │
│  │  ├── Quota: Unlimited                                 │    │
│  │  ├── Rate limit: 1000 calls/minute                   │    │
│  │  └── Price: Custom                                   │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Developer Portal:                                             │
│  • Self-service registration for API consumers               │
│  • API documentation (auto-generated from OpenAPI spec)      │
│  • API key management                                         │
│  • Usage analytics dashboard                                 │
│  • Customizable branding                                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 11 — Apigee — Analytics & Monitoring

### Built-in Analytics

```
┌──────────────────────────────────────────────────────────────┐
│         APIGEE ANALYTICS                                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Pre-built dashboards:                                        │
│  ├── API traffic: requests/sec, response times, errors       │
│  ├── Developer engagement: active developers, new signups    │
│  ├── API product usage: which products are popular           │
│  ├── Error analysis: error codes, error rates by API        │
│  ├── Geo-distribution: traffic by region/country             │
│  └── Latency analysis: p50, p95, p99 response times         │
│                                                                │
│  Custom analytics:                                             │
│  • Custom reports with dimensions and metrics                │
│  • Export to BigQuery for deep analysis                       │
│  • Real-time monitoring alerts                               │
│                                                                │
│  Dimensions: api_proxy, developer_app, client_ip,            │
│              response_status_code, target_url, etc.          │
│                                                                │
│  Metrics: total_response_time, target_response_time,         │
│           message_count, error_count, etc.                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 12 — Apigee — Security Policies

### OAuth 2.0 Flow

```
┌──────────────────────────────────────────────────────────────┐
│         APIGEE OAUTH 2.0                                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Apigee can act as the OAuth 2.0 authorization server:       │
│                                                                │
│  1. Client Credentials flow (service-to-service):            │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Client → POST /oauth/token                          │    │
│  │    (client_id + client_secret)                       │    │
│  │  Apigee → validates credentials                      │    │
│  │  Apigee → returns access_token                       │    │
│  │  Client → GET /orders (Authorization: Bearer token)  │    │
│  │  Apigee → validates token → proxies to backend       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  2. Authorization Code flow (user-facing apps):              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  User → login page → consent → auth code             │    │
│  │  App → exchanges code for access + refresh tokens    │    │
│  │  App → calls API with access token                   │    │
│  │  Apigee → validates token, checks scopes             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Token management:                                             │
│  • Token expiry configurable                                 │
│  • Refresh token support                                     │
│  • Token revocation                                          │
│  • Scope-based access control                                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 13 — API Gateway vs Apigee — When to Use Which

```
┌──────────────────────────────────────────────────────────────┐
│         API GATEWAY vs APIGEE                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────────────┬─────────────────────────────┐    │
│  │ API Gateway            │ Apigee                       │    │
│  ├────────────────────────┼─────────────────────────────┤    │
│  │ Simple API fronting    │ Full API lifecycle mgmt     │    │
│  │ Serverless             │ Managed / hybrid            │    │
│  │ OpenAPI spec only      │ Full policy engine          │    │
│  │ API key / JWT auth     │ OAuth 2.0, SAML, LDAP      │    │
│  │ Basic rate limiting    │ Advanced quotas + monetize  │    │
│  │ No developer portal    │ Full developer portal       │    │
│  │ No response caching    │ Response caching            │    │
│  │ No response transform  │ Full mediation/transform    │    │
│  │ Cloud Monitoring only  │ Rich built-in analytics     │    │
│  │ $3/million calls       │ Subscription-based pricing  │    │
│  ├────────────────────────┼─────────────────────────────┤    │
│  │ USE WHEN:              │ USE WHEN:                    │    │
│  │ • Internal APIs        │ • APIs are a product         │    │
│  │ • Simple auth needs    │ • External partner APIs      │    │
│  │ • Small-medium traffic │ • Need analytics/monetize   │    │
│  │ • Serverless backends  │ • Complex security needs    │    │
│  │ • Quick setup          │ • Enterprise requirements   │    │
│  └────────────────────────┴─────────────────────────────┘    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform — API Gateway

```hcl
# ─── API ──────────────────────────────────────────────────────
resource "google_api_gateway_api" "orders" {
  provider = google-beta
  api_id   = "orders-api"
  project  = var.project_id
}

# ─── API Config ───────────────────────────────────────────────
resource "google_api_gateway_api_config" "orders_v1" {
  provider      = google-beta
  api           = google_api_gateway_api.orders.api_id
  api_config_id = "orders-config-v1"
  project       = var.project_id

  openapi_documents {
    document {
      path     = "openapi.yaml"
      contents = base64encode(file("${path.module}/openapi-spec.yaml"))
    }
  }

  gateway_config {
    backend_config {
      google_service_account = google_service_account.gateway.email
    }
  }

  lifecycle {
    create_before_destroy = true
  }
}

# ─── Gateway ─────────────────────────────────────────────────
resource "google_api_gateway_gateway" "orders" {
  provider   = google-beta
  api_config = google_api_gateway_api_config.orders_v1.id
  gateway_id = "orders-gateway"
  region     = var.region
  project    = var.project_id
}

# ─── Service Account ─────────────────────────────────────────
resource "google_service_account" "gateway" {
  account_id   = "api-gateway-sa"
  display_name = "API Gateway Service Account"
  project      = var.project_id
}

resource "google_cloud_run_service_iam_member" "gateway_invoker" {
  project  = var.project_id
  location = var.region
  service  = google_cloud_run_v2_service.orders.name
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.gateway.email}"
}

# ─── Output gateway URL ──────────────────────────────────────
output "gateway_url" {
  value = google_api_gateway_gateway.orders.default_hostname
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# API GATEWAY
# ═══════════════════════════════════════════════════════════════

# APIs
gcloud api-gateway apis create API_ID
gcloud api-gateway apis list
gcloud api-gateway apis describe API_ID
gcloud api-gateway apis delete API_ID

# API Configs
gcloud api-gateway api-configs create CONFIG_ID \
    --api=API_ID \
    --openapi-spec=SPEC.yaml \
    --backend-auth-service-account=SA
gcloud api-gateway api-configs list --api=API_ID
gcloud api-gateway api-configs describe CONFIG_ID --api=API_ID
gcloud api-gateway api-configs delete CONFIG_ID --api=API_ID

# Gateways
gcloud api-gateway gateways create GW_ID \
    --api=API_ID --api-config=CONFIG_ID --location=LOC
gcloud api-gateway gateways list --location=LOC
gcloud api-gateway gateways describe GW_ID --location=LOC
gcloud api-gateway gateways update GW_ID \
    --api=API_ID --api-config=NEW_CONFIG_ID --location=LOC
gcloud api-gateway gateways delete GW_ID --location=LOC

# ═══════════════════════════════════════════════════════════════
# APIGEE (via gcloud or Apigee CLI)
# ═══════════════════════════════════════════════════════════════
gcloud apigee organizations list
gcloud apigee apis list --organization=ORG
gcloud apigee apis deploy --api=API --environment=ENV --organization=ORG
gcloud apigee products list --organization=ORG
gcloud apigee developers list --organization=ORG
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Microservices API Gateway

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: API GATEWAY FOR MICROSERVICES                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Single API Gateway fronting multiple Cloud Run services:            │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  API Gateway: "my-app-api"                                │        │
│  │                                                           │        │
│  │  /users/*   → users-service.run.app                      │        │
│  │  /orders/*  → orders-service.run.app                     │        │
│  │  /products/* → products-service.run.app                   │        │
│  │  /payments/* → payments-service.run.app                   │        │
│  │  /health    → health-check (no auth)                     │        │
│  │                                                           │        │
│  │  Auth: JWT validation (Firebase Auth) on all routes      │        │
│  │  except /health                                           │        │
│  │                                                           │        │
│  │  Rate limit: 1000 req/min per API key                    │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Benefits:                                                            │
│  • Single URL for all microservices                                  │
│  • Centralized auth and rate limiting                                │
│  • Backend services don't need to handle auth                        │
│  • Easy to add/remove services (update OpenAPI spec)                 │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Apigee for Partner API Program

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: PARTNER API PROGRAM WITH APIGEE                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Company exposes APIs to external partners:                          │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Apigee:                                                  │        │
│  │                                                           │        │
│  │  Developer Portal:                                        │        │
│  │  → Partners self-register                                │        │
│  │  → Browse API documentation                              │        │
│  │  → Generate API keys                                     │        │
│  │  → View usage analytics                                  │        │
│  │                                                           │        │
│  │  API Products:                                            │        │
│  │  ├── Bronze: 1K calls/day, basic endpoints, free         │        │
│  │  ├── Silver: 10K calls/day, all endpoints, $499/mo       │        │
│  │  └── Gold: unlimited, priority support, custom pricing   │        │
│  │                                                           │        │
│  │  Policies:                                                │        │
│  │  ├── OAuth 2.0 (client credentials flow)                 │        │
│  │  ├── Quota per product tier                              │        │
│  │  ├── Spike arrest (prevent bursts)                       │        │
│  │  ├── Response cache (reduce backend load)                │        │
│  │  └── Request validation (JSON schema)                    │        │
│  │                                                           │        │
│  │  Analytics:                                               │        │
│  │  → Track per-partner usage for billing                   │        │
│  │  → Identify integration issues                           │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: API Versioning Strategy

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: API VERSIONING WITH API GATEWAY                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Multiple API versions running simultaneously:                       │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Gateway: "orders-v1-gateway" → config-v1               │        │
│  │  URL: orders-v1.gateway.dev                              │        │
│  │  Backend: orders-v1.run.app                              │        │
│  │  Status: Deprecated (sunset date: 2024-06-01)            │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Gateway: "orders-v2-gateway" → config-v2               │        │
│  │  URL: orders-v2.gateway.dev                              │        │
│  │  Backend: orders-v2.run.app                              │        │
│  │  Status: Current (recommended)                           │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Alternative: Single gateway with path-based versioning:             │
│  /v1/orders → orders-v1.run.app                                     │
│  /v2/orders → orders-v2.run.app                                     │
│                                                                        │
│  Migration strategy:                                                  │
│  1. Deploy v2 gateway alongside v1                                   │
│  2. Notify consumers of deprecation                                  │
│  3. Monitor v1 traffic until it drops to zero                        │
│  4. Delete v1 gateway                                                │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create API | `gcloud api-gateway apis create API_ID` |
| Create config | `gcloud api-gateway api-configs create C --api=A --openapi-spec=S.yaml` |
| Create gateway | `gcloud api-gateway gateways create G --api=A --api-config=C --location=L` |
| Update gateway | `... gateways update G --api-config=NEW_CONFIG` |
| API key auth | `securityDefinitions.api_key` in OpenAPI spec |
| JWT auth | `x-google-issuer` + `x-google-jwks_uri` |
| Backend routing | `x-google-backend.address` per path |
| Rate limiting | `x-google-management.quota` in OpenAPI spec |
| API Gateway pricing | $3/million calls (first 2M free) |
| Apigee | Enterprise API management + portal + analytics |
| Apigee policies | SpikeArrest, Quota, OAuth, JWT, Cache, Transform |

---

## What is an API Gateway? (Beginner Explanation)

### Simple Analogy

```
┌──────────────────────────────────────────────────────────────┐
│         HOTEL FRONT DESK ANALOGY                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  An API Gateway is like a FRONT DESK at a hotel:             │
│                                                                │
│  ┌──────────┐    ┌──────────────────┐    ┌──────────────┐    │
│  │ Guest 1  │    │   FRONT DESK     │    │ Room 101     │    │
│  │ (Mobile  │───►│                  │───►│ (Users       │    │
│  │  App)    │    │ • Checks ID      │    │  Service)    │    │
│  ├──────────┤    │   (authentication)│    ├──────────────┤    │
│  │ Guest 2  │───►│ • Assigns room   │───►│ Room 202     │    │
│  │ (Web App)│    │   (routing)      │    │ (Orders      │    │
│  ├──────────┤    │ • Limits visitors │    │  Service)    │    │
│  │ Guest 3  │───►│   (rate limiting)│───►├──────────────┤    │
│  │ (Partner)│    │ • Tracks entries  │    │ Room 303     │    │
│  └──────────┘    │   (monitoring)   │    │ (Payments    │    │
│                  └──────────────────┘    │  Service)    │    │
│                                          └──────────────┘    │
│                                                                │
│  WITHOUT a front desk (no API Gateway):                      │
│  • Every room handles its own check-in (auth in every svc)  │
│  • Guests wander the building looking for rooms (no routing) │
│  • No one tracks who comes and goes (no monitoring)          │
│  • Anyone can enter unlimited times (no rate limiting)       │
│                                                                │
│  WITH a front desk (API Gateway):                             │
│  • One place handles all check-ins (centralized auth)        │
│  • Front desk directs guests to the right room (routing)     │
│  • Entry log tracks all visitors (monitoring & logging)      │
│  • Front desk limits how often guests can request things     │
│    (rate limiting)                                            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Why Do You Need an API Gateway?

```
┌──────────────────────────────────────────────────────────────┐
│         WHY USE AN API GATEWAY?                                │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Problem without API Gateway:                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  • Each microservice needs its own auth logic        │    │
│  │  • Clients must know every service URL               │    │
│  │  • No centralized rate limiting — one bad client     │    │
│  │    can overwhelm your services                       │    │
│  │  • No single place to monitor all API traffic        │    │
│  │  • Changing backends means updating all clients      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Solution with API Gateway:                                   │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ✓ Centralized authentication — handle auth once    │    │
│  │  ✓ Single URL — clients call one endpoint           │    │
│  │  ✓ Rate limiting — protect backends from abuse      │    │
│  │  ✓ Monitoring — see all traffic in one dashboard    │    │
│  │  ✓ Decoupling — change backends without breaking    │    │
│  │    clients (just update the OpenAPI spec)            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  In GCP:                                                       │
│  • API Gateway = lightweight, serverless, quick setup        │
│  • Apigee = enterprise-grade, full API lifecycle management  │
│  • Start with API Gateway → graduate to Apigee when needed   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Creating & Managing API Gateway

### Step 1 — Create an API

```
┌──────────────────────────────────────────────────────────────┐
│  CONSOLE: Create an API                                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. Go to Console → search "API Gateway" → click on it      │
│     (or navigate: API Gateway in the left sidebar)           │
│                                                                │
│  2. Click "CREATE GATEWAY" button at the top                 │
│                                                                │
│  3. In the "API" section:                                     │
│     ┌──────────────────────────────────────────────────┐     │
│     │ ○ Create a new API                                │     │
│     │                                                    │     │
│     │ Display Name: [ Orders API               ]        │     │
│     │ API ID:       [ orders-api                ]        │     │
│     │                                                    │     │
│     │ (API ID is auto-generated from display name,      │     │
│     │  but you can edit it)                              │     │
│     └──────────────────────────────────────────────────┘     │
│                                                                │
│  Equivalent gcloud command:                                   │
│  gcloud api-gateway apis create orders-api                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 2 — Create an API Config (Upload OpenAPI Spec)

```
┌──────────────────────────────────────────────────────────────┐
│  CONSOLE: Create API Config                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Still on the same "Create Gateway" page:                    │
│                                                                │
│  4. In the "API Config" section:                              │
│     ┌──────────────────────────────────────────────────┐     │
│     │ ○ Create a new API config                         │     │
│     │                                                    │     │
│     │ Upload an API Spec:                                │     │
│     │   [ Browse ] ← click and select your               │     │
│     │               openapi-spec.yaml file               │     │
│     │                                                    │     │
│     │ Display Name: [ orders-config-v1          ]        │     │
│     │                                                    │     │
│     │ Select a Service Account:                          │     │
│     │   [ gateway-sa@my-project.iam.gserviceacco... ▼ ]  │     │
│     │                                                    │     │
│     │ (This SA needs roles/run.invoker or                │     │
│     │  roles/cloudfunctions.invoker to call backends)    │     │
│     └──────────────────────────────────────────────────┘     │
│                                                                │
│  Important:                                                    │
│  • The OpenAPI spec MUST be version 2.0 (Swagger format)    │
│  • Include x-google-backend in each path to define           │
│    which backend service handles each route                  │
│  • The service account is used by the gateway to              │
│    authenticate with your backend services                   │
│                                                                │
│  Equivalent gcloud command:                                   │
│  gcloud api-gateway api-configs create orders-config-v1 \    │
│      --api=orders-api \                                       │
│      --openapi-spec=openapi-spec.yaml \                       │
│      --backend-auth-service-account=gateway-sa@proj.iam...   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 3 — Create a Gateway

```
┌──────────────────────────────────────────────────────────────┐
│  CONSOLE: Create a Gateway                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Still on the same "Create Gateway" page:                    │
│                                                                │
│  5. In the "Gateway details" section:                         │
│     ┌──────────────────────────────────────────────────┐     │
│     │ Display Name: [ orders-gateway            ]        │     │
│     │ Gateway ID:   [ orders-gateway             ]       │     │
│     │ Location:     [ us-central1                ▼ ]     │     │
│     └──────────────────────────────────────────────────┘     │
│                                                                │
│  6. Click "CREATE GATEWAY"                                    │
│                                                                │
│  7. Wait for deployment (can take 5-10 minutes)              │
│     Status will change from "Creating" → "Active"            │
│                                                                │
│  8. Once created, you'll see your gateway with:              │
│     • Gateway URL: orders-gateway-abc123.uc.gateway.dev      │
│     • Status: Active                                          │
│     • API Config: orders-config-v1                            │
│     • Location: us-central1                                   │
│                                                                │
│  Equivalent gcloud command:                                   │
│  gcloud api-gateway gateways create orders-gateway \         │
│      --api=orders-api \                                       │
│      --api-config=orders-config-v1 \                          │
│      --location=us-central1                                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 4 — Test the API

```
┌──────────────────────────────────────────────────────────────┐
│  CONSOLE: Test Your API                                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  After the gateway is active, test it:                       │
│                                                                │
│  1. Copy the Gateway URL from the gateway details page       │
│                                                                │
│  2. Test from terminal or browser:                            │
│     ┌──────────────────────────────────────────────────┐     │
│     │ # Public endpoint (no auth)                       │     │
│     │ curl https://orders-gateway-abc123.uc.gateway.dev │     │
│     │      /health                                      │     │
│     │                                                    │     │
│     │ # With API key                                     │     │
│     │ curl -H "x-api-key: YOUR_API_KEY" \               │     │
│     │   https://orders-gateway-abc123.uc.gateway.dev    │     │
│     │   /orders                                          │     │
│     │                                                    │     │
│     │ # With JWT token                                   │     │
│     │ curl -H "Authorization: Bearer YOUR_JWT" \        │     │
│     │   https://orders-gateway-abc123.uc.gateway.dev    │     │
│     │   /orders                                          │     │
│     └──────────────────────────────────────────────────┘     │
│                                                                │
│  3. Check logs:                                               │
│     Console → API Gateway → click your gateway               │
│     → "LOGS" tab shows request/response details              │
│                                                                │
│  4. Check metrics:                                             │
│     Console → API Gateway → click your gateway               │
│     → "METRICS" tab shows request count, latency, errors     │
│                                                                │
│  Common errors:                                                │
│  • 403: API key missing or invalid                           │
│  • 401: JWT invalid or expired                                │
│  • 500: Backend service unreachable (check SA permissions)   │
│  • 429: Rate limit exceeded                                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 5 — Delete Gateway / Config / API (Must Delete in Order)

```
┌──────────────────────────────────────────────────────────────┐
│  CONSOLE: Deleting API Gateway Resources                      │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ⚠️  You MUST delete in this exact order:                    │
│  Gateway → API Config → API                                  │
│                                                                │
│  (You can't delete an API if it has configs attached,        │
│   and you can't delete a config if a gateway uses it)        │
│                                                                │
│  Step 1: Delete the GATEWAY first                             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Console → API Gateway → Gateways tab                │    │
│  │  → Find your gateway → click ⋮ (three dots)          │    │
│  │  → "Delete" → confirm                                │    │
│  │  → Wait for deletion to complete                     │    │
│  │                                                      │    │
│  │  gcloud api-gateway gateways delete orders-gateway \ │    │
│  │      --location=us-central1                           │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Step 2: Delete the API CONFIG                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Console → API Gateway → APIs tab                    │    │
│  │  → Click your API → "Configs" tab                    │    │
│  │  → Find your config → click ⋮ → "Delete"            │    │
│  │  → Confirm deletion                                  │    │
│  │                                                      │    │
│  │  gcloud api-gateway api-configs delete \             │    │
│  │      orders-config-v1 --api=orders-api                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Step 3: Delete the API                                       │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Console → API Gateway → APIs tab                    │    │
│  │  → Find your API → click ⋮ → "Delete"               │    │
│  │  → Confirm deletion                                  │    │
│  │                                                      │    │
│  │  gcloud api-gateway apis delete orders-api            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Deletion order:  Gateway ──► API Config ──► API             │
│  (reverse of creation order)                                  │
│                                                                │
│  If you try to delete out of order, you'll get an error:     │
│  "Resource has children and cannot be deleted"                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Continue to **Chapter 51: Eventarc** → `51-eventarc.md`
