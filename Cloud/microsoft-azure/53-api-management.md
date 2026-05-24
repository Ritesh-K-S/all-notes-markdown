# Chapter 53: API Management (APIM)

---

## Table of Contents

- [Overview](#overview)
- [Part 1: APIM Fundamentals](#part-1-apim-fundamentals)
- [Part 2: Creating APIM (Portal Walkthrough)](#part-2-creating-apim-portal-walkthrough)
- [Part 3: APIs, Operations & Policies](#part-3-apis-operations--policies)
- [Part 4: Products & Subscriptions](#part-4-products--subscriptions)
- [Part 5: Developer Portal](#part-5-developer-portal)
- [Part 6: Security & Authentication](#part-6-security--authentication)
- [Part 7: Terraform & az CLI Reference](#part-7-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure API Management (APIM) is a fully managed API gateway that sits in front of your APIs. It handles authentication, rate limiting, caching, transformation, monitoring, and provides a developer portal where consumers can discover and test your APIs.

```
What you'll learn:
├── APIM Fundamentals (API gateway pattern)
├── Creating APIM (Portal)
├── APIs, Operations & Policies
├── Products & Subscriptions (access control)
├── Developer Portal (API documentation)
├── Security & Authentication
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: APIM Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           API MANAGEMENT OVERVIEW                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ APIM = API Gateway + Management Plane + Developer Portal          │
│                                                                       │
│ Without APIM:                                                        │
│ Client → API (no auth, no rate limit, no monitoring)             │
│                                                                       │
│ With APIM:                                                           │
│ Client → APIM Gateway → Backend API                              │
│            │                                                        │
│            ├── Authentication (API keys, OAuth, JWT)             │
│            ├── Rate limiting (100 calls/min per user)            │
│            ├── Caching (reduce backend load)                     │
│            ├── Request/Response transformation                   │
│            ├── Monitoring & analytics                             │
│            ├── Versioning (v1, v2)                                │
│            └── Developer portal (docs, try-it)                   │
│                                                                       │
│ Tiers:                                                               │
│ ├── Consumption: Serverless, pay per call, no developer portal   │
│ ├── Developer: Non-prod, dev portal, ~$49/month                  │
│ ├── Basic: Production entry, ~$152/month                        │
│ ├── Standard: Medium traffic, ~$703/month                       │
│ ├── Premium: Multi-region, VNet, ~$2,825/month                  │
│ └── v2 tiers: Basic v2, Standard v2 (newer, VNet support)       │
│                                                                       │
│ ⚡ Developer tier takes ~30 min to provision                     │
│ ⚡ Premium can take 45+ min                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating APIM (Portal Walkthrough)

```
Console → API Management services → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE API MANAGEMENT                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-api ▼]                                         │
│ Region: [Central India ▼]                                          │
│ Resource name: [apim-mycompany]                                    │
│   → Gateway URL: https://apim-mycompany.azure-api.net            │
│ Organization name: [My Company]                                    │
│ Administrator email: [admin@mycompany.com]                        │
│                                                                       │
│ Pricing tier: [Developer ▼]                                       │
│   Developer: For development/testing ($49/mo)                     │
│   Consumption: Serverless, no developer portal                    │
│   Basic v2: Production, 100 req/sec                               │
│   Standard v2: Production + VNet, 200 req/sec                     │
│                                                                       │
│ [Review + Create]                                                   │
│ ⚡ Provisioning takes 30-45 minutes!                              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: APIs, Operations & Policies

### Adding an API

```
APIM → APIs → [+ Add API]

Import options:
├── OpenAPI (Swagger) specification ← Most common
├── WSDL (SOAP services)
├── WADL
├── Azure Function App
├── Azure App Service
├── Logic App
└── Blank API (define manually)

Example: Import from OpenAPI
├── OpenAPI specification URL: https://api.example.com/swagger.json
├── Display name: [Order API]
├── Name: [order-api]
├── API URL suffix: [orders] → https://apim-mycompany.azure-api.net/orders
├── Products: [Starter ▼]
└── [Create]
```

### Policies

```
Policies = XML rules applied to requests/responses

Policy scopes (applied in order):
├── Global (all APIs)
├── Product (all APIs in a product)
├── API (all operations in an API)
└── Operation (single operation)

Policy structure:
<policies>
  <inbound>     <!-- Before sending to backend -->
    <rate-limit calls="100" renewal-period="60" />
    <set-header name="X-Custom" exists-action="override">
      <value>my-value</value>
    </set-header>
  </inbound>
  <backend>     <!-- When calling backend -->
    <forward-request />
  </backend>
  <outbound>    <!-- Before returning to caller -->
    <set-header name="X-Powered-By" exists-action="delete" />
  </outbound>
  <on-error>    <!-- When error occurs -->
    <set-status code="500" reason="Internal Error" />
  </on-error>
</policies>

Popular policies:
├── rate-limit: Limit calls per subscription (e.g., 100/min)
├── rate-limit-by-key: Limit by IP, header, etc.
├── quota: Limit total calls per period (e.g., 10000/month)
├── cache-lookup / cache-store: Cache responses
├── validate-jwt: Validate JWT tokens
├── set-header: Add/modify/remove headers
├── rewrite-uri: Change URL path
├── set-backend-service: Route to different backends
├── cors: Handle CORS headers
├── ip-filter: Allow/deny IP addresses
├── mock-response: Return mock data (no backend needed)
└── retry: Auto-retry failed backend calls
```

---

## Part 4: Products & Subscriptions

```
Products = Packages of APIs for consumers

APIM → Products → [+ Add]

Name: [Starter]
Description: [Free tier - 100 calls/day]
State: Published
APIs: [Order API, Product API]
☑ Requires subscription (API key)
☐ Requires approval

Built-in products:
├── Starter: Rate limited, good for trial
└── Unlimited: No rate limits

Subscriptions:
├── When developer signs up → Gets subscription key (API key)
├── Pass key in header: Ocp-Apim-Subscription-Key: <key>
├── Or in query: ?subscription-key=<key>
├── Each subscription has primary and secondary keys
└── Rotate keys without downtime (use secondary while rotating primary)

APIM → Subscriptions → [+ Add subscription]
Name: [mobile-app-team]
Scope: Product [Starter]
→ Generates API keys for this team
```

---

## Part 5: Developer Portal

```
Developer Portal = Self-service API documentation site

APIM → Developer portal → Portal overview → [Enable]

URL: https://apim-mycompany.developer.azure-api.net

Features:
├── API documentation (auto-generated from OpenAPI)
├── Interactive "Try It" console (test APIs in browser)
├── Subscription sign-up (self-service API key)
├── Code samples (curl, C#, Java, Python, etc.)
├── Rate limit/quota visibility
└── Customizable UI (branding, pages, widgets)

Customization:
├── APIM → Developer portal → [Edit] (visual editor)
├── Add pages, menus, content blocks
├── Custom CSS/HTML
├── Add company logo and branding
└── Publish changes to make them live

⚡ Not available on Consumption tier!
⚡ Available on Developer, Basic, Standard, Premium tiers
```

---

## Part 6: Security & Authentication

```
Authentication methods:
├── Subscription keys (API keys) — simplest
│   Header: Ocp-Apim-Subscription-Key: <key>
│
├── OAuth 2.0 / JWT validation — recommended for production
│   <validate-jwt header-name="Authorization">
│     <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration" />
│     <required-claims>
│       <claim name="aud" match="all">
│         <value>api://my-api</value>
│       </claim>
│     </required-claims>
│   </validate-jwt>
│
├── Client certificates (mutual TLS)
│   <authentication-certificate thumbprint="..." />
│
├── IP filtering
│   <ip-filter action="allow">
│     <address>10.0.0.0/24</address>
│   </ip-filter>
│
└── Managed Identity (APIM → Backend)
    APIM uses managed identity to call backend securely
```

---

## Part 7: Terraform & az CLI Reference

### Terraform

```hcl
resource "azurerm_api_management" "main" {
  name                = "apim-mycompany"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  publisher_name      = "My Company"
  publisher_email     = "admin@mycompany.com"
  sku_name            = "Developer_1"
}

resource "azurerm_api_management_api" "orders" {
  name                = "order-api"
  resource_group_name = azurerm_resource_group.main.name
  api_management_name = azurerm_api_management.main.name
  revision            = "1"
  display_name        = "Order API"
  path                = "orders"
  protocols           = ["https"]

  import {
    content_format = "openapi-link"
    content_value  = "https://api.example.com/swagger.json"
  }
}

resource "azurerm_api_management_product" "starter" {
  product_id            = "starter"
  resource_group_name   = azurerm_resource_group.main.name
  api_management_name   = azurerm_api_management.main.name
  display_name          = "Starter"
  subscription_required = true
  published             = true
}
```

### Bicep

```bicep
// API Management
resource apim 'Microsoft.ApiManagement/service@2023-03-01-preview' = {
  name: 'apim-mycompany'
  location: resourceGroup().location
  sku: { name: 'Developer', capacity: 1 }
  properties: {
    publisherEmail: 'admin@mycompany.com'
    publisherName: 'My Company'
  }
}

// API
resource api 'Microsoft.ApiManagement/service/apis@2023-03-01-preview' = {
  parent: apim
  name: 'order-api'
  properties: {
    displayName: 'Order API'
    path: 'orders'
    protocols: ['https']
    format: 'openapi-link'
    value: 'https://api.example.com/swagger.json'
  }
}

// Product
resource product 'Microsoft.ApiManagement/service/products@2023-03-01-preview' = {
  parent: apim
  name: 'starter'
  properties: {
    displayName: 'Starter'
    subscriptionRequired: true
    state: 'published'
  }
}
```
  --resource-group rg-api \
  --publisher-name "My Company" \
  --publisher-email admin@mycompany.com \
  --sku-name Developer

# Import API from OpenAPI
az apim api import \
  --resource-group rg-api \
  --service-name apim-mycompany \
  --api-id order-api \
  --path orders \
  --specification-format OpenApi \
  --specification-url https://api.example.com/swagger.json

# Create subscription
az apim subscription create \
  --resource-group rg-api \
  --service-name apim-mycompany \
  --subscription-id mobile-team \
  --display-name "Mobile Team" \
  --scope /apis/order-api

# List APIs
az apim api list --resource-group rg-api --service-name apim-mycompany -o table

# Delete APIM
az apim delete --name apim-mycompany --resource-group rg-api --yes
```

---

## Real-World Patterns

### Pattern 1: API Gateway for Microservices

```
┌─────────────────────────────────────────────────┐
│       Unified API Gateway                       │
├─────────────────────────────────────────────────┤
│                                                 │
│  Mobile App / Web App / Partners                │
│            │                                    │
│            ▼                                    │
│  ┌──────────────────────┐                       │
│  │   API Management     │                       │
│  │   (api.company.com)  │                       │
│  ├──────────────────────┤                       │
│  │ Policies:            │                       │
│  │ - Rate limit: 100/min│                       │
│  │ - JWT validation     │                       │
│  │ - Response caching   │                       │
│  │ - Request transform  │                       │
│  └────────┬─────────────┘                       │
│      ┌────┼────┐                                │
│      ▼    ▼    ▼                                │
│  Orders Users Products                          │
│  (AKS)  (App   (Azure                           │
│         Svc)   Functions)                       │
│                                                 │
│  Developer Portal: Self-service API keys,       │
│  documentation, and testing                     │
└─────────────────────────────────────────────────┘
```

### Pattern 2: API Versioning Strategy

```
┌─────────────────────────────────────────────────┐
│       API Version Management                    │
├─────────────────────────────────────────────────┤
│                                                 │
│  /api/v1/orders  ──→ Backend v1 (legacy)        │
│  /api/v2/orders  ──→ Backend v2 (current)       │
│  /api/v3/orders  ──→ Backend v3 (preview)       │
│                                                 │
│  Products:                                      │
│  ┌───────────┬───────────┬───────────┐          │
│  │ Free      │ Standard  │ Premium   │          │
│  ├───────────┤───────────┤───────────┤          │
│  │ 100/hour  │ 1000/hour │ Unlimited │          │
│  │ v1 only   │ v1 + v2   │ All vers  │          │
│  │ No SLA    │ 99.9% SLA │ 99.99%   │          │
│  └───────────┘───────────┘───────────┘          │
│                                                 │
│  Sunset policy: v1 deprecated after 6 months    │
│  Revision: Non-breaking changes within version  │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
APIM = API Gateway + Management + Developer Portal
URL: https://<name>.azure-api.net/<api-path>

Components: APIs → Operations → Policies
Products: Package APIs together with rate limits
Subscriptions: API keys for consumers (Ocp-Apim-Subscription-Key)

Policies (XML): rate-limit, quota, validate-jwt, cache, cors, ip-filter
Scopes: Global → Product → API → Operation

Developer Portal: Self-service docs, try-it, sign-up
Authentication: API keys, OAuth/JWT, client certs, IP filter

Tiers: Consumption | Developer ($49) | Basic ($152) | Standard ($703) | Premium ($2,825)
v2 tiers: Basic v2, Standard v2 (newer, faster provisioning)

⚡ Provisioning takes 30-45 minutes!
```

---

## What's Next?

Next chapter: [Chapter 54: AKS Deep Dive](54-aks-deep-dive.md) — Advanced Kubernetes patterns: network policies, AGIC, KEDA, pod identity, and GitOps.
