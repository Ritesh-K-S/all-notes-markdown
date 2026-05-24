# Tools — Kong, AWS API Gateway, Apigee, Zuul, Spring Cloud Gateway

> **What you'll learn**: A hands-on comparison of the most popular API Gateway tools — their architectures, strengths, weaknesses, configuration examples, and when to choose each one for your project.

---

## Real-Life Analogy: Choosing a Car

Picking an API Gateway is like choosing a car:

- **Kong** = A highly customizable sports car — fast, open-source, tons of aftermarket plugins
- **AWS API Gateway** = A self-driving electric car — zero maintenance, just push a button, but locked to one ecosystem
- **Apigee (Google)** = A luxury business sedan — enterprise features, analytics dashboard, premium price
- **Zuul (Netflix)** = A hand-built race car — built for extreme conditions, but you assemble it yourself
- **Spring Cloud Gateway** = A reliable family car — if you're already in the Spring ecosystem, it just fits
- **Envoy/Istio** = A fleet management system — not just one car, but managing all vehicles in your company

---

## Quick Comparison Table

```
┌───────────────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│                   │  Kong    │ AWS API  │ Apigee   │  Zuul    │ Spring   │  Envoy   │
│                   │          │ Gateway  │ (Google) │(Netflix) │ Cloud GW │          │
├───────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Type              │ Open Src │ Managed  │ Managed  │ Open Src │ Open Src │ Open Src │
│ Language          │ Lua/C    │ AWS      │ Java     │ Java     │ Java     │ C++      │
│ Deployment        │ Self/Cloud│ Serverless│ Cloud   │ Self     │ Self     │ Self/Mesh│
│ Performance       │ ⭐⭐⭐⭐⭐│ ⭐⭐⭐   │ ⭐⭐⭐  │ ⭐⭐⭐⭐ │ ⭐⭐⭐⭐ │ ⭐⭐⭐⭐⭐│
│ Plugin Ecosystem  │ ⭐⭐⭐⭐⭐│ ⭐⭐⭐   │ ⭐⭐⭐⭐ │ ⭐⭐⭐   │ ⭐⭐⭐  │ ⭐⭐⭐⭐ │
│ Learning Curve    │ Medium   │ Low      │ High     │ High     │ Medium   │ High     │
│ Cost              │ Free/Paid│ Pay/use  │ $$$$$    │ Free     │ Free     │ Free     │
│ Best For          │ Multi-   │ AWS-     │ Enterpri-│ Netflix- │ Spring   │ Service  │
│                   │ cloud    │ native   │ se API   │ scale    │ apps     │ mesh     │
│                   │          │          │ mgmt     │          │          │          │
└───────────────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

---

## 1. Kong Gateway

### Overview

Kong is the most popular open-source API Gateway. Built on top of **Nginx** and **OpenResty** (Lua), it's extremely fast and extensible.

```
Kong Architecture:

┌─────────────────────────────────────────────────────────┐
│                     KONG GATEWAY                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │            Nginx / OpenResty Core                │    │
│  │         (handles HTTP/HTTPS traffic)            │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │  Auth    │ │  Rate    │ │ Logging  │ │ Transform│  │
│  │  Plugin  │ │  Limit   │ │  Plugin  │ │  Plugin  │  │
│  │          │ │  Plugin  │ │          │ │          │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │     PostgreSQL / Cassandra (Config Store)       │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │     Admin API (port 8001) — Manage routes       │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Configuration Example

```yaml
# kong.yml — Declarative configuration
_format_version: "3.0"

services:
  - name: user-service
    url: http://user-service:8080
    connect_timeout: 5000
    write_timeout: 60000
    read_timeout: 60000
    retries: 3
    routes:
      - name: user-routes
        paths:
          - /api/v1/users
        methods:
          - GET
          - POST
          - PUT
        strip_path: false
        preserve_host: false

plugins:
  # JWT Authentication
  - name: jwt
    service: user-service
    config:
      header_names:
        - Authorization
      claims_to_verify:
        - exp

  # Rate Limiting (Redis-backed)
  - name: rate-limiting
    service: user-service
    config:
      minute: 100
      hour: 5000
      policy: redis
      redis_host: redis
      redis_port: 6379

  # Request Transformer
  - name: request-transformer
    service: user-service
    config:
      add:
        headers:
          - "X-Gateway: kong"
      remove:
        headers:
          - "X-Internal-Debug"

  # Prometheus metrics
  - name: prometheus
    config:
      per_consumer: true

  # Correlation ID for tracing
  - name: correlation-id
    config:
      header_name: X-Request-ID
      generator: uuid
```

```bash
# Managing Kong via Admin API
# Add a service
curl -X POST http://localhost:8001/services \
  -d name=user-service \
  -d url=http://user-backend:8080

# Add a route
curl -X POST http://localhost:8001/services/user-service/routes \
  -d paths[]=/api/users \
  -d methods[]=GET \
  -d methods[]=POST

# Enable rate limiting plugin
curl -X POST http://localhost:8001/services/user-service/plugins \
  -d name=rate-limiting \
  -d config.minute=100
```

**When to choose Kong:**
- You want open-source with enterprise option
- You need multi-cloud or hybrid cloud deployment
- You need extensive plugin ecosystem (100+ plugins)
- You want Kubernetes-native (Kong Ingress Controller)

---

## 2. AWS API Gateway

### Overview

A fully managed, serverless API Gateway from AWS. You don't manage any servers — just configure routes and deploy.

```
AWS API Gateway Architecture:

┌─────────────────────────────────────────────────────────────┐
│                 AWS API GATEWAY (Managed)                    │
│                                                             │
│  Client ──▶ CloudFront (Edge) ──▶ API Gateway              │
│                                     │                       │
│                    ┌────────────────┼────────────────┐      │
│                    │                │                │      │
│                    ▼                ▼                ▼      │
│            ┌──────────────┐ ┌──────────────┐ ┌──────────┐  │
│            │   Lambda     │ │   EC2/ECS    │ │   HTTP   │  │
│            │   Functions  │ │   Backend    │ │   Proxy  │  │
│            └──────────────┘ └──────────────┘ └──────────┘  │
│                                                             │
│  Types:                                                    │
│  • REST API — Full features, more expensive                │
│  • HTTP API — Simpler, faster, cheaper                     │
│  • WebSocket API — Real-time connections                   │
└─────────────────────────────────────────────────────────────┘
```

### Configuration Example (Serverless Framework)

```yaml
# serverless.yml — AWS API Gateway with Lambda backend
service: my-api

provider:
  name: aws
  runtime: python3.11
  region: us-east-1

functions:
  getUsers:
    handler: handlers.get_users
    events:
      - http:
          path: /api/v1/users
          method: get
          cors: true
          authorizer:
            name: jwtAuthorizer
            type: TOKEN
            identitySource: method.request.header.Authorization

  createUser:
    handler: handlers.create_user
    events:
      - http:
          path: /api/v1/users
          method: post
          cors: true
          authorizer: jwtAuthorizer
          request:
            schemas:
              application/json: ${file(schemas/create-user.json)}

  # Rate limiting via usage plans
resources:
  Resources:
    UsagePlan:
      Type: AWS::ApiGateway::UsagePlan
      Properties:
        UsagePlanName: BasicPlan
        Throttle:
          BurstLimit: 100        # Max concurrent requests
          RateLimit: 50          # Requests per second
        Quota:
          Limit: 10000           # Requests per month
          Period: MONTH

    # API Key for the usage plan
    ApiKey:
      Type: AWS::ApiGateway::ApiKey
      Properties:
        Name: partner-api-key
        Enabled: true
```

```python
# handlers.py — Lambda handler behind API Gateway
import json
import boto3

def get_users(event, context):
    """Handler for GET /api/v1/users"""
    # event contains: path, headers, queryStringParameters, body
    user_id = event.get('pathParameters', {}).get('id')
    
    # API Gateway already verified JWT (via authorizer)
    # User info available in event['requestContext']['authorizer']
    claims = event['requestContext']['authorizer']['claims']
    
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*',
        },
        'body': json.dumps({
            'users': [{'id': '123', 'name': 'Alice'}],
            'requestedBy': claims['sub'],
        })
    }
```

**When to choose AWS API Gateway:**
- You're **all-in on AWS** (Lambda, DynamoDB, etc.)
- You want **zero infrastructure management**
- You need **WebSocket** support out of the box
- Budget is flexible (can get expensive at high volume)
- You want built-in **WAF, CloudFront, and Cognito** integration

---

## 3. Google Apigee

### Overview

Apigee is Google's enterprise API management platform. It goes beyond a gateway — it's a full API lifecycle management tool with developer portal, analytics, monetization, and governance.

```
Apigee Architecture:

┌──────────────────────────────────────────────────────────────┐
│                        APIGEE PLATFORM                        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │            Management Plane                           │    │
│  │  • API Design Studio                                 │    │
│  │  • Developer Portal (self-service docs)              │    │
│  │  • Analytics Dashboard (traffic, errors, latency)    │    │
│  │  • Monetization (pay-per-call billing)               │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │            Runtime Plane (Gateway)                    │    │
│  │                                                      │    │
│  │  Client ──▶ [ProxyEndpoint] ──▶ [Policies] ──▶      │    │
│  │             [TargetEndpoint] ──▶ Backend             │    │
│  │                                                      │    │
│  │  Policies: Auth, Quota, Transform, Cache, Mediation  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Configuration Example

```xml
<!-- Apigee API Proxy Configuration (policies in XML) -->
<!-- apiproxy/proxies/default.xml -->
<ProxyEndpoint name="default">
    <PreFlow>
        <Request>
            <!-- Verify API Key -->
            <Step><Name>Verify-API-Key</Name></Step>
            <!-- Check Rate Limit -->
            <Step><Name>Spike-Arrest</Name></Step>
            <!-- Quota Check -->
            <Step><Name>Check-Quota</Name></Step>
        </Request>
    </PreFlow>

    <Flows>
        <Flow name="GetUsers">
            <Condition>(proxy.pathsuffix MatchesPath "/users") and (request.verb = "GET")</Condition>
            <Request>
                <Step><Name>Cache-Lookup</Name></Step>
            </Request>
            <Response>
                <Step><Name>Cache-Store</Name></Step>
            </Response>
        </Flow>
    </Flows>

    <RouteRule name="default">
        <TargetEndpoint>backend</TargetEndpoint>
    </RouteRule>
</ProxyEndpoint>
```

```xml
<!-- apiproxy/policies/Spike-Arrest.xml -->
<SpikeArrest name="Spike-Arrest">
    <Rate>100ps</Rate>  <!-- 100 per second -->
    <Identifier ref="request.header.X-API-Key"/>
</SpikeArrest>

<!-- apiproxy/policies/Check-Quota.xml -->
<Quota name="Check-Quota">
    <Interval>1</Interval>
    <TimeUnit>month</TimeUnit>
    <Allow count="100000"/>  <!-- 100K requests/month -->
    <Identifier ref="developer.app.name"/>
</Quota>
```

**When to choose Apigee:**
- You're a **large enterprise** with many API products
- You need **developer portal** with self-service API key provisioning
- You want **API monetization** (charge per call)
- You need **deep analytics** and compliance reporting
- Budget is not a constraint (Apigee is expensive)

---

## 4. Netflix Zuul / Zuul 2

### Overview

Zuul was Netflix's API Gateway, handling all traffic for 200+ million subscribers. Zuul 1 used blocking I/O; Zuul 2 uses non-blocking (Netty-based).

```
Zuul Architecture:

┌──────────────────────────────────────────────────────────┐
│                    ZUUL GATEWAY                           │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │              Filter Chain (Zuul Filters)            │  │
│  │                                                    │  │
│  │  PRE-FILTERS     ROUTING        POST-FILTERS      │  │
│  │  ┌──────────┐    FILTERS       ┌──────────┐      │  │
│  │  │ Auth     │   ┌──────────┐   │ Logging  │      │  │
│  │  │ Filter   │   │ Route to │   │ Filter   │      │  │
│  │  ├──────────┤   │ Backend  │   ├──────────┤      │  │
│  │  │ Rate     │   │ Service  │   │ Metrics  │      │  │
│  │  │ Limit    │   └──────────┘   │ Filter   │      │  │
│  │  ├──────────┤                  ├──────────┤      │  │
│  │  │ Debug    │   ERROR          │ CORS     │      │  │
│  │  │ Filter   │   FILTERS        │ Filter   │      │  │
│  │  └──────────┘   ┌──────────┐   └──────────┘      │  │
│  │                  │ Error    │                      │  │
│  │                  │ Handler  │                      │  │
│  │                  └──────────┘                      │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Backed by: Eureka (Service Discovery)                  │
│             Ribbon (Client-Side Load Balancing)          │
│             Hystrix (Circuit Breaking)                   │
└──────────────────────────────────────────────────────────┘
```

### Configuration Example (Zuul with Spring Boot)

```java
// ZuulApplication.java
@SpringBootApplication
@EnableZuulProxy
@EnableDiscoveryClient
public class ZuulApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZuulApplication.class, args);
    }
}
```

```yaml
# application.yml — Zuul configuration
zuul:
  routes:
    user-service:
      path: /api/users/**
      serviceId: user-service
      strip-prefix: true
    order-service:
      path: /api/orders/**
      serviceId: order-service
      strip-prefix: true

  # Timeout settings
  host:
    connect-timeout-millis: 5000
    socket-timeout-millis: 30000

  # Ribbon load balancing
  ribbon:
    eager-load:
      enabled: true

# Eureka service discovery
eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka:8761/eureka/

# Hystrix circuit breaker
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 10000
```

```java
// CustomAuthFilter.java — Zuul pre-filter for authentication
import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import javax.servlet.http.HttpServletRequest;

public class CustomAuthFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }  // Runs before routing

    @Override
    public int filterOrder() { return 1; }  // First filter to run

    @Override
    public boolean shouldFilter() { return true; }  // Always run

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        String token = request.getHeader("Authorization");
        if (token == null || !isValidToken(token)) {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            ctx.setResponseBody("{\"error\": \"Unauthorized\"}");
        } else {
            // Add user info for downstream services
            ctx.addZuulRequestHeader("X-User-Id", extractUserId(token));
        }
        return null;
    }
}
```

**When to choose Zuul:**
- You're already in the **Netflix OSS / Spring Cloud** ecosystem
- You need **dynamic filter deployment** (Zuul can hot-reload Groovy filters)
- You want proven technology that handles **Netflix-scale** traffic

> **Note:** Zuul 1 is in maintenance mode. For new projects, consider Spring Cloud Gateway instead.

---

## 5. Spring Cloud Gateway

### Overview

The modern replacement for Zuul in the Spring ecosystem. Built on Spring WebFlux (Project Reactor), it's non-blocking and reactive.

```
Spring Cloud Gateway Architecture:

┌──────────────────────────────────────────────────────────┐
│              SPRING CLOUD GATEWAY                         │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │        Netty Server (Non-Blocking I/O)             │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │              Route Predicate Engine                 │  │
│  │  path, header, method, query, host, cookie, time   │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │              Gateway Filter Chain                   │  │
│  │  Pre-filters → Proxy → Post-filters               │  │
│  │  (AddHeader, RateLimit, CircuitBreaker, Retry...)  │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Integrates with:                                       │
│  • Spring Security (Auth)                               │
│  • Resilience4j (Circuit Breaker)                       │
│  • Spring Cloud LoadBalancer                            │
│  • Spring Cloud Discovery (Eureka/Consul)               │
└──────────────────────────────────────────────────────────┘
```

### Configuration Example

```yaml
# application.yml — Spring Cloud Gateway configuration
spring:
  cloud:
    gateway:
      routes:
        # User service with load balancing
        - id: user-service
          uri: lb://user-service  # "lb://" = service discovery + load balancing
          predicates:
            - Path=/api/v1/users/**
            - Method=GET,POST,PUT
          filters:
            - StripPrefix=2
            - AddRequestHeader=X-Gateway, spring-cloud
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/users
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
                methods: GET

        # Order service with rate limiting
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - StripPrefix=2
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10    # 10 req/sec steady
                redis-rate-limiter.burstCapacity: 20    # 20 req/sec burst
                key-resolver: "#{@userKeyResolver}"     # Rate limit per user

        # Canary deployment — weight-based
        - id: payment-service-v2
          uri: http://payment-v2:8003
          predicates:
            - Path=/api/v1/payments/**
            - Weight=payment, 10              # 10% traffic
          filters:
            - StripPrefix=2

        - id: payment-service-v1
          uri: http://payment-v1:8002
          predicates:
            - Path=/api/v1/payments/**
            - Weight=payment, 90              # 90% traffic
          filters:
            - StripPrefix=2

      # Global filters applied to all routes
      default-filters:
        - AddResponseHeader=X-Response-Time, ${spring.cloud.gateway.response-time}
        - RemoveResponseHeader=Server

  # Redis for rate limiting
  redis:
    host: localhost
    port: 6379
```

```java
// KeyResolverConfig.java — Custom key resolver for rate limiting
import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import reactor.core.publisher.Mono;

@Configuration
public class KeyResolverConfig {

    @Bean
    public KeyResolver userKeyResolver() {
        // Rate limit by API key (or IP if no key)
        return exchange -> {
            String apiKey = exchange.getRequest().getHeaders()
                .getFirst("X-API-Key");
            if (apiKey != null) {
                return Mono.just(apiKey);
            }
            return Mono.just(
                exchange.getRequest().getRemoteAddress()
                    .getAddress().getHostAddress()
            );
        };
    }
}
```

**When to choose Spring Cloud Gateway:**
- You're building a **Spring Boot / Spring Cloud** application
- You want tight integration with **Spring Security, Eureka, Resilience4j**
- You need **reactive/non-blocking** performance
- You prefer **Java configuration** over YAML/declarative config

---

## 6. Envoy Proxy (+ Istio Service Mesh)

### Overview

Envoy is a high-performance proxy built by Lyft. It's the data plane for the Istio service mesh and is also used standalone as an API gateway.

```
Envoy as API Gateway:

┌────────────────────────────────────────────────────────────┐
│                    ENVOY PROXY                              │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Listeners (port bindings)                   │    │
│  │         ├── 0.0.0.0:443 (HTTPS)                    │    │
│  │         └── 0.0.0.0:80 (HTTP → redirect HTTPS)     │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Filter Chains                               │    │
│  │         ├── TLS Inspector                           │    │
│  │         ├── HTTP Connection Manager                 │    │
│  │         │   ├── Router Filter                       │    │
│  │         │   ├── Rate Limit Filter                   │    │
│  │         │   ├── JWT Auth Filter                     │    │
│  │         │   └── CORS Filter                         │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Clusters (backend service groups)           │    │
│  │         ├── user-service (3 instances)              │    │
│  │         ├── order-service (5 instances)             │    │
│  │         └── payment-service (2 instances)           │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  Control Plane: Istio Pilot / xDS API                      │
│  (Dynamic configuration without restart)                   │
└────────────────────────────────────────────────────────────┘
```

### Configuration Example

```yaml
# envoy.yaml — Envoy as API Gateway
static_resources:
  listeners:
    - name: main_listener
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 8080
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains: ["*"]
                      routes:
                        # Route /api/users → user-service cluster
                        - match:
                            prefix: "/api/v1/users"
                          route:
                            cluster: user_service
                            prefix_rewrite: "/users"
                            timeout: 5s
                            retry_policy:
                              retry_on: "5xx,reset,connect-failure"
                              num_retries: 3

                        # Route /api/orders → order-service cluster
                        - match:
                            prefix: "/api/v1/orders"
                          route:
                            cluster: order_service
                            prefix_rewrite: "/orders"

                http_filters:
                  # JWT Authentication
                  - name: envoy.filters.http.jwt_authn
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
                      providers:
                        my_provider:
                          issuer: "https://auth.myapp.com"
                          remote_jwks:
                            http_uri:
                              uri: "https://auth.myapp.com/.well-known/jwks.json"
                              cluster: auth_service
                              timeout: 5s
                      rules:
                        - match:
                            prefix: "/api/"
                          requires:
                            provider_name: "my_provider"

                  # Rate Limiting
                  - name: envoy.filters.http.ratelimit
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
                      domain: "my_api"
                      rate_limit_service:
                        grpc_service:
                          envoy_grpc:
                            cluster_name: rate_limit_service

                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: user_service
      connect_timeout: 2s
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: user_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: user-service
                      port_value: 8001

    - name: order_service
      connect_timeout: 2s
      type: STRICT_DNS
      lb_policy: LEAST_REQUEST
      circuit_breakers:
        thresholds:
          - max_connections: 1000
            max_pending_requests: 500
            max_retries: 3
      load_assignment:
        cluster_name: order_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: order-service
                      port_value: 8002
```

**When to choose Envoy:**
- You're using **Kubernetes with Istio/Linkerd** (service mesh)
- You need **extreme performance** (C++, handles millions of req/sec)
- You want **dynamic configuration** without restarts (xDS API)
- You need both **north-south** (external) and **east-west** (internal) traffic management

---

## Decision Flowchart

```
Which API Gateway should you choose?

                    START
                      │
                      ▼
              ┌──────────────┐
              │ Using AWS    │──── Yes ──▶ AWS API Gateway
              │ exclusively? │              (serverless, zero ops)
              └──────┬───────┘
                     │ No
                     ▼
              ┌──────────────┐
              │ Enterprise   │──── Yes ──▶ Apigee
              │ API mgmt +   │              (portal, monetization,
              │ monetization?│              analytics)
              └──────┬───────┘
                     │ No
                     ▼
              ┌──────────────┐
              │ Spring Boot  │──── Yes ──▶ Spring Cloud Gateway
              │ ecosystem?   │              (native integration)
              └──────┬───────┘
                     │ No
                     ▼
              ┌──────────────┐
              │ Service mesh │──── Yes ──▶ Envoy + Istio
              │ (K8s)?       │              (mesh + gateway)
              └──────┬───────┘
                     │ No
                     ▼
              ┌──────────────┐
              │ Multi-cloud  │──── Yes ──▶ Kong
              │ or hybrid?   │              (portable, plugin-rich)
              └──────┬───────┘
                     │ No
                     ▼
              Kong (default best choice for most)
```

---

## Performance Comparison

```
Benchmark: 10,000 concurrent connections, simple proxy pass-through

┌─────────────────────┬───────────────┬─────────────┬──────────────┐
│ Gateway             │ Requests/sec  │ p99 Latency │ Memory Usage │
├─────────────────────┼───────────────┼─────────────┼──────────────┤
│ Envoy               │ ~150,000      │ 2ms         │ 50 MB        │
│ Kong (DB-less)      │ ~100,000      │ 3ms         │ 100 MB       │
│ Spring Cloud GW     │ ~50,000       │ 8ms         │ 256 MB       │
│ Zuul 2 (Netty)     │ ~40,000       │ 10ms        │ 512 MB       │
│ AWS API Gateway     │ ~10,000*      │ 15ms        │ N/A (managed)│
│ Apigee              │ ~10,000*      │ 20ms        │ N/A (managed)│
└─────────────────────┴───────────────┴─────────────┴──────────────┘

* Managed services have different scaling models — they auto-scale
  but add latency due to their internal processing.

Note: Real-world numbers depend heavily on plugin configuration,
      payload size, and backend response time.
```

---

## Real-World Example

### Who Uses What?

```
┌────────────────────┬───────────────────────────────────┐
│ Company            │ API Gateway                       │
├────────────────────┼───────────────────────────────────┤
│ Netflix            │ Zuul → migrating to custom        │
│ Uber               │ Custom (built on Envoy)           │
│ Airbnb             │ Custom + Envoy                    │
│ Stripe             │ Custom (HAProxy + custom logic)   │
│ Shopify            │ Custom + Nginx + Lua              │
│ LinkedIn           │ Custom (Rest.li framework)        │
│ Twilio             │ Kong                              │
│ Skyscanner         │ Kong                              │
│ Expedia            │ Apigee                            │
│ Walgreens          │ Apigee                            │
│ Many startups      │ AWS API Gateway                   │
│ Spring shops       │ Spring Cloud Gateway              │
│ Lyft               │ Envoy (they built it!)            │
│ Google (GKE)       │ Envoy (via Istio)                 │
└────────────────────┴───────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Solution |
|---------|-------------|----------|
| Choosing based on hype | You might not need Envoy's complexity | Match tool to your team's skills and actual needs |
| Starting with enterprise tool for MVP | Overkill, slow iteration, expensive | Start with Kong or AWS API Gateway; upgrade later |
| Not considering operational burden | Self-hosted gateways need monitoring, upgrades, HA | Factor in team capacity for ops |
| Vendor lock-in | Migrating from AWS API Gateway to Kong is non-trivial | Use gateway-agnostic configuration where possible |
| Ignoring plugin ecosystem | Writing custom auth from scratch when a plugin exists | Check plugin marketplace first |
| Single gateway instance | Single point of failure | Always deploy 2+ instances with a load balancer in front |
| Not load-testing the gateway | Gateway becomes the bottleneck under traffic | Benchmark your specific configuration before production |

---

## Key Takeaways

- **Kong** is the best general-purpose open-source gateway — fast, extensible, multi-cloud
- **AWS API Gateway** is ideal for serverless AWS-native architectures with zero ops
- **Apigee** is for enterprises needing full API lifecycle management (portal, analytics, monetization)
- **Zuul** is legacy Netflix tech — powerful but being replaced by Spring Cloud Gateway
- **Spring Cloud Gateway** is the natural choice for Spring Boot microservice architectures
- **Envoy** is the highest-performance option and the standard for service mesh (Istio)
- At **large scale**, companies (Netflix, Uber, Stripe) build custom gateways tailored to their needs
- Always deploy **multiple gateway instances** — the gateway must never be a single point of failure

---

## What's Next?

Now that you know the tools, the final chapter in this part covers an advanced architectural pattern that changes how you think about API gateways: **Backend for Frontend (BFF) Pattern** — creating specialized gateway layers for different client types.

Next: [07-bff-pattern.md](./07-bff-pattern.md)
