# Chapter 49: Amazon API Gateway

---

## Table of Contents

- [Overview](#overview)
- [Part 1: API Gateway Fundamentals](#part-1-api-gateway-fundamentals)
- [Part 2: Creating a REST API (Full Portal Walkthrough)](#part-2-creating-a-rest-api-full-portal-walkthrough)
- [Part 3: HTTP API vs REST API](#part-3-http-api-vs-rest-api)
- [Part 4: Authorization & Security](#part-4-authorization--security)
- [Part 5: Stages, Deployments & Throttling](#part-5-stages-deployments--throttling)
- [Part 6: Terraform & CLI Examples](#part-6-terraform--cli-examples)
- [Part 7: Real-World Patterns](#part-7-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is an API Gateway? Why Do We Need It?

Imagine a restaurant. Customers (users) don't walk into the kitchen (backend services) directly. They talk to a **waiter** (API Gateway) who takes their order, brings it to the kitchen, and returns the food.

**API Gateway is the front door to your backend services.** It receives HTTP requests from users/apps, validates them, authenticates the caller, and routes the request to the right backend (Lambda, EC2, ECS, or any HTTP endpoint).

**Why not just expose Lambda/EC2 directly?**
- 🔐 **Security**: API Gateway handles authentication (API keys, Cognito, IAM)
- 🚦 **Rate limiting**: Prevent abuse by limiting requests per second
- 📊 **Monitoring**: Automatic CloudWatch metrics for every API call
- 🔄 **Transformation**: Convert request/response formats
- 🌐 **Custom domains**: Use `api.mycompany.com` instead of ugly AWS URLs

**Simple real-world examples:**
- 📱 Mobile app calls `api.myapp.com/users` → API Gateway → Lambda returns user data
- 🌐 Frontend calls REST API → API Gateway authenticates via Cognito → routes to microservices
- 💬 Chat app uses WebSocket API → API Gateway manages persistent connections

Amazon API Gateway is a fully managed service for creating, publishing, and managing APIs at any scale. It supports REST APIs, HTTP APIs, and WebSocket APIs.

```
What you'll learn:
├── API Gateway fundamentals (REST, HTTP, WebSocket)
├── Creating a REST API (portal walkthrough)
├── HTTP API vs REST API comparison
├── Authorization (IAM, Cognito, Lambda authorizer)
├── Stages, deployments, canary, throttling
└── Terraform & CLI examples
```

---

## Part 1: API Gateway Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW API GATEWAY WORKS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Client → API Gateway → Backend (Lambda, HTTP, AWS Service)        │
│                                                                       │
│ ┌────────┐    ┌──────────────┐    ┌──────────────────────┐       │
│ │ Client │───▶│ API Gateway  │───▶│ Backend              │       │
│ │        │    │              │    │ ├── Lambda Function  │       │
│ │ Mobile │    │ ├── Auth     │    │ ├── HTTP Endpoint    │       │
│ │ Web    │    │ ├── Throttle │    │ ├── AWS Service      │       │
│ │ IoT    │    │ ├── Cache    │    │ │   (DynamoDB, SQS)  │       │
│ │        │    │ ├── Transform│    │ └── Mock             │       │
│ └────────┘    │ └── Log     │    └──────────────────────┘       │
│               └──────────────┘                                      │
│                                                                       │
│ API types:                                                           │
│ ┌──────────────────┬──────────┬──────────┬───────────────┐       │
│ │                  │ REST API │ HTTP API │ WebSocket API  │       │
│ ├──────────────────┼──────────┼──────────┼───────────────┤       │
│ │ Protocol         │ REST     │ REST     │ WebSocket      │       │
│ │ Cost             │ $3.50/M  │ $1.00/M  │ $1.00/M + conn│       │
│ │ Latency          │ Higher   │ Lower    │ Persistent     │       │
│ │ API Keys         │ ✅       │ ❌       │ ❌             │       │
│ │ Usage Plans      │ ✅       │ ❌       │ ❌             │       │
│ │ Request valid.   │ ✅       │ ✅       │ ❌             │       │
│ │ Caching          │ ✅       │ ❌       │ ❌             │       │
│ │ WAF              │ ✅       │ ❌       │ ❌             │       │
│ │ Custom domain    │ ✅       │ ✅       │ ✅             │       │
│ │ Authorizers      │ IAM,Cog, │ IAM,JWT  │ IAM,Lambda     │       │
│ │                  │ Lambda   │          │                │       │
│ │ Transformations  │ ✅ (VTL) │ ❌       │ ❌             │       │
│ │ ⚡ Best for      │ Full feat│ Simple,  │ Real-time,     │       │
│ │                  │ API mgmt │ low cost │ chat, games    │       │
│ └──────────────────┴──────────┴──────────┴───────────────┘       │
│                                                                       │
│ Endpoint types:                                                      │
│ ├── Regional: Deployed in one region (⚡ default)               │
│ ├── Edge-optimized: Via CloudFront (global reach)               │
│ └── Private: Only accessible within VPC                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a REST API (Full Portal Walkthrough)

```
Console → API Gateway → Create API → REST API → Build

┌─────────────────────────────────────────────────────────────────┐
│           CREATE REST API                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Choose the protocol:                                           │
│ ● REST                                                         │
│ ○ WebSocket                                                    │
│                                                                   │
│ Create new API:                                                │
│ ● New API                                                      │
│ ○ Clone from existing API                                     │
│ ○ Import from Swagger/OpenAPI                                 │
│                                                                   │
│ API name: [order-api]                                          │
│ Description: [Order management REST API]                      │
│                                                                   │
│ API endpoint type:                                             │
│ ● Regional                                                     │
│ ○ Edge optimized                                               │
│ ○ Private                                                      │
│                                                                   │
│                    [Create API]                                 │
└─────────────────────────────────────────────────────────────────┘

After creation → Resources editor:

┌─────────────────────────────────────────────────────────────────┐
│           CREATING RESOURCES & METHODS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Actions → Create Resource                                     │
│                                                                   │
│ Resource name: [orders]                                        │
│ Resource path: /orders                                         │
│ ☑ Enable API Gateway CORS                                    │
│ → Automatically adds OPTIONS method for CORS                 │
│                                                                   │
│ Create sub-resource:                                           │
│ Resource path: /orders/{orderId}                              │
│ → {orderId} is a path parameter                              │
│                                                                   │
│ Resource tree:                                                 │
│ /                                                              │
│ ├── /orders                                                   │
│ │   ├── GET (list orders)                                    │
│ │   ├── POST (create order)                                  │
│ │   └── /{orderId}                                           │
│ │       ├── GET (get order)                                  │
│ │       ├── PUT (update order)                               │
│ │       └── DELETE (delete order)                            │
│ └── /health                                                   │
│     └── GET (health check)                                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           CREATING A METHOD (GET /orders)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Select resource /orders → Actions → Create Method → GET      │
│                                                                   │
│ Integration type:                                              │
│ ● Lambda Function                                              │
│   → Most common: API Gateway invokes Lambda                  │
│ ○ HTTP                                                         │
│   → Proxy to another HTTP endpoint                           │
│ ○ Mock                                                         │
│   → Return static response (for testing)                    │
│ ○ AWS Service                                                  │
│   → Direct integration (DynamoDB, SQS, S3, etc.)           │
│   → ⚡ No Lambda needed for simple operations               │
│ ○ VPC Link                                                     │
│   → Private integration with NLB                             │
│                                                                   │
│ Lambda Function:                                               │
│ ☑ Use Lambda Proxy integration (⚡ recommended)              │
│ → Passes entire request to Lambda (headers, body, etc.)     │
│ → Lambda returns {statusCode, headers, body}                │
│ → Simplest integration, no mapping templates                │
│                                                                   │
│ Lambda Function: [us-east-1 ▼] [get-orders ▼]               │
│                                                                   │
│ ☑ Use Default Timeout                                        │
│ → 29 seconds max (hard limit for REST API)                  │
│                                                                   │
│ ⚠️ API Gateway timeout: 29 seconds max                       │
│ → Lambda function must return within 29 seconds              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           METHOD EXECUTION FLOW                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Client ─→ Method Request ─→ Integration Request ─→ Backend   │
│ Client ←─ Method Response ←─ Integration Response ←─ Backend │
│                                                                   │
│ Method Request:                                                │
│ ├── Authorization: [None / AWS_IAM / Cognito / Custom ▼]   │
│ ├── Request Validator: [Validate body ▼]                    │
│ ├── API Key Required: ☐                                     │
│ ├── URL Query String Parameters                              │
│ ├── HTTP Request Headers                                     │
│ └── Request Body (Models/Schema)                             │
│                                                                   │
│ Integration Request (Lambda proxy skips these):              │
│ ├── Mapping Templates (VTL - Velocity Template Language)    │
│ ├── Transform request before sending to backend             │
│ └── Passthrough behavior                                     │
│                                                                   │
│ Integration Response (Lambda proxy skips these):             │
│ ├── Response mapping templates                               │
│ ├── HTTP status regex matching                               │
│ └── Header mappings                                          │
│                                                                   │
│ Method Response:                                               │
│ ├── HTTP Status: 200, 400, 404, 500                        │
│ ├── Response Headers                                         │
│ └── Response Models                                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: HTTP API vs REST API

```
┌─────────────────────────────────────────────────────────────────────┐
│           HTTP API (⚡ Simpler & Cheaper)                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → API Gateway → Create API → HTTP API → Build              │
│                                                                       │
│ Step 1: Integrations                                                │
│ [Add integration]                                                    │
│ ├── Lambda                                                          │
│ ├── HTTP                                                            │
│ └── No integration (routes only)                                   │
│                                                                       │
│ Lambda function: [get-orders ▼]                                     │
│ API name: [order-api-http]                                          │
│                                                                       │
│ Step 2: Configure routes                                            │
│ Method: [GET ▼]  Path: [/orders]  Target: [get-orders]           │
│ Method: [POST ▼] Path: [/orders]  Target: [create-order]         │
│ Method: [GET ▼]  Path: [/orders/{orderId}] Target: [get-order]   │
│                                                                       │
│ Step 3: Define stages                                               │
│ Stage name: [$default]                                              │
│ ☑ Auto-deploy                                                      │
│ → Changes deploy automatically (no manual deployment needed)     │
│                                                                       │
│ When to use HTTP API:                                               │
│ ├── Simple Lambda/HTTP proxy                                     │
│ ├── JWT authorization                                             │
│ ├── Cost-sensitive ($1/M vs $3.50/M)                             │
│ ├── Lower latency needed                                         │
│ └── No need for caching, WAF, API keys, usage plans             │
│                                                                       │
│ When to use REST API:                                               │
│ ├── Need API keys + usage plans                                  │
│ ├── Need response caching                                        │
│ ├── Need WAF integration                                         │
│ ├── Need request/response transformation (VTL)                  │
│ ├── Need Cognito authorizer                                      │
│ └── Need edge-optimized endpoint                                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Authorization & Security

```
┌─────────────────────────────────────────────────────────────────────┐
│           AUTHORIZATION METHODS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. IAM Authorization:                                               │
│    → Client signs request with SigV4                              │
│    → IAM policy controls access                                   │
│    → ⚡ Best for: AWS services, internal apps                     │
│    → Cross-account: Use IAM roles                                │
│                                                                       │
│ 2. Cognito User Pool Authorizer (REST API only):                  │
│    → Client sends JWT token from Cognito                         │
│    → API Gateway validates token automatically                   │
│    → ⚡ Best for: Mobile/web apps with user auth                 │
│    Console: Authorizers → Create → Cognito                       │
│    User Pool: [prod-user-pool ▼]                                 │
│    Token source: Authorization (header)                           │
│                                                                       │
│ 3. Lambda Authorizer (Custom):                                     │
│    → Lambda function validates token/request                     │
│    → Returns IAM policy (allow/deny)                             │
│    → ⚡ Best for: Custom auth, third-party tokens               │
│    Console: Authorizers → Create → Lambda                        │
│    Lambda function: [custom-authorizer ▼]                        │
│    Token source: Authorization (header)                           │
│    Token validation: [regex for token format]                    │
│    Authorization caching: [300] seconds (⚡ reduce Lambda calls)│
│                                                                       │
│ 4. API Keys + Usage Plans (REST API only):                        │
│    → API key required in x-api-key header                       │
│    → Usage plan defines rate/quota per key                      │
│    Console: API Keys → Create API Key                            │
│    Console: Usage Plans → Create                                  │
│    ├── Rate: [100] requests/second (throttle)                   │
│    ├── Burst: [200] requests                                     │
│    └── Quota: [10000] requests/month                            │
│                                                                       │
│ 5. Resource Policy (REST API):                                     │
│    → JSON policy on the API itself                               │
│    → IP allowlist, VPC endpoint restriction                     │
│    → Cross-account access                                        │
│                                                                       │
│ CORS configuration:                                                 │
│ Console → Resources → Enable CORS                                 │
│ Access-Control-Allow-Origin: [*] or [https://example.com]       │
│ Access-Control-Allow-Methods: [GET, POST, PUT, DELETE]          │
│ Access-Control-Allow-Headers: [Content-Type, Authorization]    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Stages, Deployments & Throttling

```
┌─────────────────────────────────────────────────────────────────────┐
│           STAGES & DEPLOYMENTS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → API Gateway → API → Stages                               │
│                                                                       │
│ Stages:                                                              │
│ ├── dev   → https://abc123.execute-api.us-east-1.amazonaws.com/dev│
│ ├── staging → .../staging                                         │
│ └── prod  → .../prod                                               │
│                                                                       │
│ Deploy API:                                                          │
│ Actions → Deploy API                                                │
│ Deployment stage: [prod ▼] or [New Stage]                          │
│ Stage name: [prod]                                                  │
│ Description: [v2.1 - Added order search]                           │
│ → ⚠️ Changes NOT live until deployed                              │
│                                                                       │
│ Stage Variables (per-stage configuration):                          │
│ Console → Stages → prod → Stage Variables                          │
│ [lambdaAlias] = [prod]                                              │
│ [tableName] = [orders-prod]                                         │
│ → Reference in integration: ${stageVariables.lambdaAlias}        │
│ → ⚡ Different Lambda versions per stage                          │
│                                                                       │
│ Canary deployments:                                                  │
│ Console → Stages → prod → Canary                                  │
│ Canary traffic: [10] %                                              │
│ → 10% of traffic goes to new deployment                          │
│ → 90% stays on current deployment                                │
│ → Monitor metrics → Promote or rollback                          │
│                                                                       │
│ Throttling:                                                          │
│ Account-level: 10,000 requests/second (soft limit)                │
│ Stage-level: [5000] requests/second                                │
│ Method-level: [1000] requests/second                               │
│ Burst: [5000] requests                                              │
│ → 429 Too Many Requests when exceeded                             │
│                                                                       │
│ Caching (REST API only):                                            │
│ Cache capacity: [0.5 GB ▼] (0.5 to 237 GB)                       │
│ TTL: [300] seconds (default)                                       │
│ Per-key caching: ☑ (cache by query params)                        │
│ → $0.02/hour per 0.5 GB (even when not used!)                   │
│ → ⚡ Reduces Lambda invocations significantly                    │
│                                                                       │
│ Custom domain:                                                       │
│ Console → Custom domain names → Create                             │
│ Domain name: [api.example.com]                                      │
│ ACM certificate: [*.example.com ▼]                                 │
│ Endpoint: ● Regional ○ Edge                                       │
│ API mapping: API [order-api] Stage [prod] Path [/v1]             │
│ → Route 53: CNAME/A-alias to API Gateway domain                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & CLI Examples

```hcl
# REST API
resource "aws_api_gateway_rest_api" "orders" {
  name        = "order-api"
  description = "Order management API"
  endpoint_configuration { types = ["REGIONAL"] }
}

resource "aws_api_gateway_resource" "orders" {
  rest_api_id = aws_api_gateway_rest_api.orders.id
  parent_id   = aws_api_gateway_rest_api.orders.root_resource_id
  path_part   = "orders"
}

resource "aws_api_gateway_method" "get_orders" {
  rest_api_id   = aws_api_gateway_rest_api.orders.id
  resource_id   = aws_api_gateway_resource.orders.id
  http_method   = "GET"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "lambda" {
  rest_api_id             = aws_api_gateway_rest_api.orders.id
  resource_id             = aws_api_gateway_resource.orders.id
  http_method             = aws_api_gateway_method.get_orders.http_method
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.get_orders.invoke_arn
}

resource "aws_api_gateway_deployment" "prod" {
  rest_api_id = aws_api_gateway_rest_api.orders.id
  depends_on  = [aws_api_gateway_integration.lambda]
}

resource "aws_api_gateway_stage" "prod" {
  rest_api_id   = aws_api_gateway_rest_api.orders.id
  deployment_id = aws_api_gateway_deployment.prod.id
  stage_name    = "prod"
}

# HTTP API (simpler)
resource "aws_apigatewayv2_api" "orders_http" {
  name          = "order-api-http"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda" {
  api_id             = aws_apigatewayv2_api.orders_http.id
  integration_type   = "AWS_PROXY"
  integration_uri    = aws_lambda_function.get_orders.invoke_arn
  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "get_orders" {
  api_id    = aws_apigatewayv2_api.orders_http.id
  route_key = "GET /orders"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}

resource "aws_apigatewayv2_stage" "prod" {
  api_id      = aws_apigatewayv2_api.orders_http.id
  name        = "prod"
  auto_deploy = true
}
```

```bash
# Create REST API
aws apigateway create-rest-api --name order-api --endpoint-configuration types=REGIONAL

# Deploy API
aws apigateway create-deployment --rest-api-id abc123 --stage-name prod

# Test endpoint
curl https://abc123.execute-api.us-east-1.amazonaws.com/prod/orders

# Create HTTP API
aws apigatewayv2 create-api --name order-api-http --protocol-type HTTP

# Get API key usage
aws apigateway get-usage --usage-plan-id plan123 --key-id key123 \
  --start-date 2024-01-01 --end-date 2024-01-31
```

---

## Part 7: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Serverless REST API                                      │
│ API Gateway (REST) → Lambda → DynamoDB                             │
│ ├── Cognito for user authentication                              │
│ ├── API key + usage plan for partner access                     │
│ ├── WAF for security                                              │
│ ├── CloudFront for caching + global distribution               │
│ └── Custom domain: api.example.com                               │
│                                                                       │
│ Pattern 2: Microservices API (HTTP API)                              │
│ API Gateway HTTP → Lambda (per microservice)                      │
│ ├── /users → user-service Lambda                                │
│ ├── /orders → order-service Lambda                              │
│ ├── /products → product-service Lambda                          │
│ └── JWT authorizer for all routes                                │
│                                                                       │
│ Pattern 3: Service Proxy (no Lambda)                                │
│ API Gateway → DynamoDB (direct integration)                       │
│ API Gateway → SQS (direct integration)                            │
│ API Gateway → Step Functions (direct integration)                 │
│ → ⚡ Eliminates Lambda for simple CRUD operations               │
│                                                                       │
│ Pattern 4: WebSocket API (real-time)                                │
│ WebSocket API → Lambda                                              │
│ ├── $connect → store connection ID in DynamoDB                 │
│ ├── $disconnect → remove connection                             │
│ ├── sendMessage → broadcast to all connections                  │
│ └── Use case: Chat, live dashboards, notifications              │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Use HTTP API unless you need REST-specific features          │
│ 2. Enable request validation                                      │
│ 3. Use Lambda proxy integration (simplest)                       │
│ 4. Set appropriate throttling per stage                          │
│ 5. Custom domain + ACM certificate                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting: Common API Gateway Issues

### "CORS errors in the browser"

The #1 API Gateway issue. CORS errors appear when your frontend (e.g., `app.example.com`) calls an API on a different domain.

```
Solutions:
1. For REST API: Enable CORS on each resource
   Console → API Gateway → Resource → Actions → Enable CORS
   Then REDEPLOY the API (people forget this step!)

2. For HTTP API: Add CORS in the API settings
   Console → API → CORS → Configure
   Allow origins: https://app.example.com
   Allow methods: GET, POST, PUT, DELETE, OPTIONS
   Allow headers: Content-Type, Authorization

3. For Lambda proxy: Your Lambda MUST also return CORS headers:
   headers: { "Access-Control-Allow-Origin": "*" }
```

### "502 Bad Gateway"

```
1. ☐ Lambda function timing out?
   API Gateway has a 29-second hard limit for Lambda integration
   Your Lambda timeout must be < 29 seconds

2. ☐ Lambda returning wrong response format?
   Lambda proxy integration requires this exact format:
   { statusCode: 200, headers: {...}, body: JSON.stringify(data) }

3. ☐ Lambda has an unhandled exception?
   Check CloudWatch Logs for the Lambda function
```

### "429 Too Many Requests"

```
1. Default throttle: 10,000 requests/second (account-level)
2. Check if a usage plan is limiting your API key
3. Request a quota increase via Service Quotas console
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Forgetting to deploy after changes | Changes don't take effect | Always Deploy to a stage after editing |
| Lambda timeout > 29s | 502 errors | Keep Lambda < 29s for API Gateway |
| Not enabling CORS | Frontend can't call API | Enable CORS + redeploy |
| Missing Lambda invoke permission | 500 errors | API Gateway needs `lambda:InvokeFunction` |
| Not using stages (dev/prod) | Can't test safely | Create dev + prod stages |

---

## Quick Reference

```
API Gateway Quick Reference:
├── REST API: Full-featured, $3.50/M requests, WAF + caching
├── HTTP API: Simple, $1.00/M requests (⚡ 70% cheaper)
├── WebSocket API: Real-time, persistent connections
├── Endpoint: Regional (default), Edge, Private
├── Integrations: Lambda proxy (⚡), HTTP, AWS service, Mock
├── Auth: IAM, Cognito, Lambda authorizer, API keys, JWT
├── Stages: dev/staging/prod with stage variables
├── Throttling: 10K req/s account limit, configurable per stage
├── Caching: REST API only, 0.5-237 GB ($0.02/hr per 0.5GB)
├── Timeout: 29 seconds max (hard limit for REST)
├── Custom domain: ACM cert + Route 53 alias
├── ⚡ Use HTTP API for simple Lambda APIs
├── ⚡ Use direct AWS service integration to skip Lambda
└── ⚡ Always enable CloudWatch logging per stage
```

---

## What's Next?

In **Chapter 50: ECR (Elastic Container Registry)**, we'll cover container image management, image scanning, lifecycle policies, and cross-account/region replication.
