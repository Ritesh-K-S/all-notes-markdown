# Rate Limiting & Throttling — Protecting Your System

> **What you'll learn**: How rate limiting and throttling protect your APIs from abuse, the algorithms behind them (Token Bucket, Leaky Bucket, Sliding Window), how to implement them at the gateway level, and how companies like Stripe and GitHub use them in production.

---

## Real-Life Analogy: The Water Faucet and the Bucket

Imagine you have a water tank on your roof that fills slowly. The tank can hold 100 liters, and it refills at 1 liter per second.

- If you turn on the faucet gently (2 liters/sec), the tank keeps up fine
- If you open it full blast (50 liters/sec), the tank empties quickly
- Once the tank is empty, **no more water** — you have to wait for it to refill

This is exactly how a **Token Bucket** rate limiter works:
- The "bucket" holds a maximum number of **tokens** (allowed requests)
- Tokens refill at a fixed rate
- Each request consumes one token
- When the bucket is empty → request is **rejected** (HTTP 429: Too Many Requests)

**Throttling** is slightly different — instead of rejecting, you **slow down** the requester. Like a speed bump on a road that forces cars to go 20 mph instead of crashing into a wall.

---

## Why Rate Limiting Matters

```
Without rate limiting:

Abusive Client ──────▶ 100,000 req/sec ──────▶ Your API ──────▶ 💥 DEAD
                                                (overwhelmed)

Legitimate users: "Why is the site down?!"

With rate limiting:

Abusive Client ──────▶ 100,000 req/sec ──▶ ┌──────────┐  100 req/sec  ┌──────────┐
                                            │  RATE    │──────────────▶│ Your API │
Legit Client ────────▶ 10 req/sec ────────▶ │  LIMITER │               │ (healthy)│
                                            └──────────┘               └──────────┘
                                                 │
                                                 │ 99,900 req/sec
                                                 ▼
                                            HTTP 429 ❌
                                            "Too Many Requests"
```

**Rate limiting protects against:**

| Threat | How Rate Limiting Helps |
|--------|------------------------|
| DDoS attacks | Limits requests per IP/client |
| Brute-force login attempts | Limits login tries per account |
| Web scraping | Prevents bots from crawling all your data |
| Runaway automation | A buggy script sending millions of requests |
| API abuse | Free-tier users trying to get premium usage |
| Cascading failures | One service flooding another |

---

## Rate Limiting vs Throttling

```
┌───────────────────────────────────────────────────────────────────────┐
│                                                                       │
│  RATE LIMITING                          THROTTLING                    │
│  ─────────────                          ──────────                    │
│                                                                       │
│  "You've exceeded your limit."         "Slow down, please."          │
│                                                                       │
│  Action: REJECT (429 response)         Action: DELAY (queue request) │
│                                                                       │
│  Hard stop — request denied            Soft stop — request queued    │
│  immediately                           and processed later           │
│                                                                       │
│  Client gets error instantly           Client waits longer but       │
│                                         eventually gets a response    │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

In practice, most systems use **rate limiting** (reject) because throttling (queuing) can lead to memory issues and unpredictable latency.

---

## Rate Limiting Algorithms

### Algorithm 1: Fixed Window Counter

The simplest approach — count requests in fixed time windows.

```
Fixed Window: 100 requests per minute

Timeline:
│◀────── Minute 1 ──────▶│◀────── Minute 2 ──────▶│
│                          │                          │
│  Request count: 0→100    │  Request count: 0→...   │
│  ✅✅✅...✅ (100th)     │  ✅ (counter resets)    │
│  ❌ (101st → REJECTED)  │                          │

Problem: "Boundary burst"
│         ...90 requests──▶│◀──90 requests...        │
│     at 0:59              │    at 1:00              │
│                          │                          │
│  180 requests in 2 seconds (even though limit is 100/min!)
```

**Pros:** Simple, low memory  
**Cons:** Allows bursts at window boundaries

### Algorithm 2: Sliding Window Log

Keep a log of all request timestamps. Count requests in the last N seconds.

```
Sliding Window Log: 100 requests per 60 seconds

Timestamp log: [10:00:01, 10:00:03, 10:00:05, ..., 10:00:58]

New request at 10:01:02:
  1. Remove entries older than 10:00:02 (60 sec ago)
  2. Count remaining entries
  3. If count < 100 → ALLOW (add timestamp)
  4. If count >= 100 → REJECT

┌─────────────────────────────────────────────────────┐
│  ◀──────────── 60-second sliding window ──────────▶ │
│  [removed]  │  ✅ ✅ ✅ ✅ ✅ ... (counted)        │
│             │  ←── window slides with time ───▶     │
└─────────────────────────────────────────────────────┘
```

**Pros:** Precise, no boundary issues  
**Cons:** High memory (stores every timestamp)

### Algorithm 3: Sliding Window Counter (Hybrid)

Combines fixed window + sliding window for accuracy without high memory.

```
Sliding Window Counter: 100 requests per minute

Previous window count: 80 requests
Current window count: 30 requests  
Current window elapsed: 25% (15 seconds into 60-second window)

Estimated count = (previous × remaining%) + current
                = (80 × 75%) + 30
                = 60 + 30 = 90

90 < 100 → ALLOW ✅

│◀── Previous Window ──▶│◀── Current Window ────────────────▶│
│      80 requests       │ 30 requests so far                 │
│                   75%  │ 25% elapsed                        │
│               ┌────────┼──┐                                 │
│               │  Sliding Window Estimate = 90               │
│               └────────┼──┘                                 │
```

**Pros:** Memory efficient, accurate  
**Cons:** Slightly approximate (but good enough for production)

### Algorithm 4: Token Bucket ⭐ (Most Popular)

```
Token Bucket: capacity=10, refill_rate=2 tokens/sec

Time 0:  Bucket = [●●●●●●●●●●] (10 tokens, full)
         Request arrives → consume 1 token
         Bucket = [●●●●●●●●●○] (9 tokens)

Time 1s: +2 tokens refilled
         Bucket = [●●●●●●●●●●] (10 tokens, capped at capacity)

Burst scenario:
Time 0:  10 requests arrive at once
         Bucket = [○○○○○○○○○○] (0 tokens - all consumed)
         11th request → REJECTED ❌

Time 1s: +2 tokens
         Bucket = [●●○○○○○○○○] (2 tokens available)
         2 requests can be served

Key insight: Token Bucket ALLOWS bursts up to bucket capacity
             while maintaining average rate = refill_rate
```

**Pros:** Allows controlled bursts, simple, widely used  
**Cons:** Can still allow short bursts

### Algorithm 5: Leaky Bucket

```
Leaky Bucket: capacity=10, drain_rate=2 requests/sec

Requests enter the bucket (queue) from the top.
They "leak" out the bottom at a fixed rate.

       Request  Request  Request
         │        │        │
         ▼        ▼        ▼
    ┌──────────────────────────┐
    │    ● ● ● ● ● ● ●        │ ← Bucket (queue)
    │    ● ● ●                 │    capacity = 10
    └──────────┬───────────────┘
               │
               ▼ (leaks at fixed rate: 2/sec)
         Processed requests

If bucket is full → new requests OVERFLOW → REJECTED ❌

Key insight: Leaky Bucket produces CONSTANT output rate
             regardless of input burstiness
```

**Pros:** Smooth, constant output rate  
**Cons:** Doesn't handle legitimate bursts well

### Algorithm Comparison

```
┌──────────────────┬────────────┬───────────────┬────────────────┐
│ Algorithm        │ Burst      │ Memory        │ Accuracy       │
│                  │ Handling   │ Usage         │                │
├──────────────────┼────────────┼───────────────┼────────────────┤
│ Fixed Window     │ Poor       │ Very Low      │ Low            │
│ Sliding Log      │ Good       │ High          │ Very High      │
│ Sliding Counter  │ Good       │ Low           │ High           │
│ Token Bucket     │ Excellent  │ Very Low      │ High           │
│ Leaky Bucket     │ None       │ Low           │ High           │
└──────────────────┴────────────┴───────────────┴────────────────┘

Most used in production: Token Bucket (AWS, Stripe, Kong)
```

---

## How It Works Internally

### Distributed Rate Limiting Challenge

With a single gateway instance, rate limiting is simple (in-memory counter). But with multiple gateway instances, you need **shared state**:

```
Problem: Distributed gateway without shared state

Client sends 100 requests:
  ├── 34 requests → Gateway Instance 1 (thinks: "only 34, under limit")
  ├── 33 requests → Gateway Instance 2 (thinks: "only 33, under limit")
  └── 33 requests → Gateway Instance 3 (thinks: "only 33, under limit")

Total: 100 requests got through! (limit was supposed to be 50)

Solution: Shared counter in Redis

                    ┌─────────────────────┐
                    │       REDIS          │
                    │  key: "user:123"     │
                    │  value: 47           │
                    │  ttl: 60 sec         │
                    └──────────┬──────────┘
                         ▲     │     ▲
                    ┌────┘     │     └────┐
                    │          │          │
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Gateway  │ │ Gateway  │ │ Gateway  │
              │ Instance │ │ Instance │ │ Instance │
              │    1     │ │    2     │ │    3     │
              └──────────┘ └──────────┘ └──────────┘

Each instance atomically increments the same Redis counter.
Redis INCR is atomic — no race conditions!
```

### Rate Limit Headers

APIs communicate rate limit status via response headers:

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100         ← Your total allowance
X-RateLimit-Remaining: 67      ← Requests remaining in this window
X-RateLimit-Reset: 1623456789  ← Unix timestamp when limit resets
Retry-After: 30                ← (on 429) seconds to wait before retrying

HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1623456789
Retry-After: 30

Body:
{
  "error": "rate_limit_exceeded",
  "message": "You have exceeded 100 requests per minute. Please retry after 30 seconds."
}
```

---

## Code Examples

### Python: Token Bucket Rate Limiter with Redis

```python
# rate_limiter.py
# Distributed Token Bucket rate limiter using Redis

import time
import redis

# Connect to Redis
redis_client = redis.Redis(host="localhost", port=6379, db=0)

# Lua script for atomic token bucket operation in Redis
# This runs entirely on the Redis server — no race conditions!
TOKEN_BUCKET_SCRIPT = """
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- Get current bucket state
local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1])
local last_refill = tonumber(bucket[2])

-- Initialize bucket if it doesn't exist
if tokens == nil then
    tokens = capacity
    last_refill = now
end

-- Calculate tokens to add based on time elapsed
local elapsed = now - last_refill
local new_tokens = elapsed * refill_rate
tokens = math.min(capacity, tokens + new_tokens)

-- Try to consume tokens
local allowed = false
if tokens >= requested then
    tokens = tokens - requested
    allowed = true
end

-- Save state back to Redis
redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
redis.call('EXPIRE', key, 120)  -- Auto-cleanup after 2 minutes of inactivity

if allowed then
    return {1, tokens}  -- {allowed, remaining_tokens}
else
    return {0, tokens}
end
"""


class TokenBucketRateLimiter:
    def __init__(self, capacity=100, refill_rate=10):
        """
        capacity: Maximum tokens in the bucket (burst size)
        refill_rate: Tokens added per second (sustained rate)
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.script = redis_client.register_script(TOKEN_BUCKET_SCRIPT)

    def is_allowed(self, client_id: str, tokens_needed: int = 1) -> dict:
        """Check if request is allowed for this client."""
        key = f"ratelimit:{client_id}"
        now = time.time()

        result = self.script(
            keys=[key],
            args=[self.capacity, self.refill_rate, now, tokens_needed],
        )

        allowed = bool(result[0])
        remaining = int(result[1])

        return {
            "allowed": allowed,
            "remaining": remaining,
            "limit": self.capacity,
            "reset_in_seconds": (self.capacity - remaining) / self.refill_rate,
        }


# Usage in Flask gateway middleware
from flask import Flask, request, jsonify

app = Flask(__name__)

# Different limits for different tiers
limiters = {
    "free": TokenBucketRateLimiter(capacity=60, refill_rate=1),       # 60/min
    "pro": TokenBucketRateLimiter(capacity=600, refill_rate=10),      # 600/min
    "enterprise": TokenBucketRateLimiter(capacity=6000, refill_rate=100),  # 6000/min
}


@app.before_request
def rate_limit_middleware():
    """Apply rate limiting before every request."""
    client_id = request.headers.get("X-API-Key", request.remote_addr)
    tier = get_client_tier(client_id)  # Look up client's tier
    
    limiter = limiters.get(tier, limiters["free"])
    result = limiter.is_allowed(client_id)

    if not result["allowed"]:
        response = jsonify({
            "error": "rate_limit_exceeded",
            "retry_after": result["reset_in_seconds"],
        })
        response.status_code = 429
        response.headers["X-RateLimit-Limit"] = str(result["limit"])
        response.headers["X-RateLimit-Remaining"] = "0"
        response.headers["Retry-After"] = str(int(result["reset_in_seconds"]))
        return response
```

### Java: Rate Limiter with Resilience4j

```java
// RateLimitConfig.java
// Rate limiting configuration using Resilience4j with Spring Cloud Gateway

import io.github.resilience4j.ratelimiter.RateLimiter;
import io.github.resilience4j.ratelimiter.RateLimiterConfig;
import io.github.resilience4j.ratelimiter.RateLimiterRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.time.Duration;

@Configuration
public class RateLimitConfig {

    @Bean
    public RateLimiterRegistry rateLimiterRegistry() {
        // Free tier: 60 requests per minute
        RateLimiterConfig freeConfig = RateLimiterConfig.custom()
            .limitForPeriod(60)                        // Max requests in period
            .limitRefreshPeriod(Duration.ofMinutes(1)) // Period duration
            .timeoutDuration(Duration.ofMillis(0))     // Don't wait, reject immediately
            .build();

        // Pro tier: 600 requests per minute
        RateLimiterConfig proConfig = RateLimiterConfig.custom()
            .limitForPeriod(600)
            .limitRefreshPeriod(Duration.ofMinutes(1))
            .timeoutDuration(Duration.ofMillis(0))
            .build();

        RateLimiterRegistry registry = RateLimiterRegistry.of(freeConfig);
        registry.addConfiguration("pro", proConfig);
        return registry;
    }
}

// RateLimitFilter.java — Gateway filter applying rate limits
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.http.HttpStatus;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

public class RateLimitFilter implements GatewayFilter {

    private final RateLimiterRegistry registry;

    public RateLimitFilter(RateLimiterRegistry registry) {
        this.registry = registry;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // Get client identifier (API key or IP)
        String clientId = extractClientId(exchange);
        
        // Get or create a rate limiter for this client
        RateLimiter limiter = registry.rateLimiter(clientId);

        // Try to acquire permission
        if (limiter.acquirePermission()) {
            // Add rate limit headers to response
            exchange.getResponse().getHeaders().add(
                "X-RateLimit-Remaining",
                String.valueOf(limiter.getMetrics().getAvailablePermissions())
            );
            return chain.filter(exchange);  // Request allowed
        } else {
            // Rate limit exceeded — return 429
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            exchange.getResponse().getHeaders().add("Retry-After", "60");
            return exchange.getResponse().setComplete();
        }
    }

    private String extractClientId(ServerWebExchange exchange) {
        String apiKey = exchange.getRequest().getHeaders().getFirst("X-API-Key");
        if (apiKey != null) return apiKey;
        return exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
    }
}
```

---

## Infrastructure Example: Redis + Nginx Rate Limiting

```nginx
# nginx.conf — Multi-tier rate limiting

# Define rate limit zones
# Zone 1: Per IP — 10 requests/second
limit_req_zone $binary_remote_addr zone=per_ip:10m rate=10r/s;

# Zone 2: Per API key — extracted from header
map $http_x_api_key $api_key_zone {
    default $http_x_api_key;
    ""      $binary_remote_addr;
}
limit_req_zone $api_key_zone zone=per_api_key:50m rate=100r/s;

# Zone 3: Global — protect the entire system
limit_req_zone $server_name zone=global:1m rate=10000r/s;

server {
    listen 443 ssl;
    server_name api.myapp.com;

    # Apply all three rate limit zones
    location /api/ {
        # Per-IP limit: 10 req/sec, allow burst of 20
        limit_req zone=per_ip burst=20 nodelay;
        
        # Per-API-key limit: 100 req/sec, allow burst of 200
        limit_req zone=per_api_key burst=200 nodelay;
        
        # Global limit: 10,000 req/sec total
        limit_req zone=global burst=5000;
        
        # Custom error page for 429
        limit_req_status 429;
        error_page 429 = @rate_limited;
        
        proxy_pass http://backend;
    }

    location @rate_limited {
        default_type application/json;
        return 429 '{"error": "rate_limit_exceeded", "retry_after": 1}';
    }
}
```

```yaml
# docker-compose.yml — Redis for distributed rate limiting
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data

  api-gateway:
    build: ./gateway
    ports:
      - "8000:8000"
    environment:
      REDIS_URL: redis://redis:6379
      RATE_LIMIT_FREE: "60/minute"
      RATE_LIMIT_PRO: "600/minute"
      RATE_LIMIT_ENTERPRISE: "6000/minute"
    depends_on:
      - redis
```

---

## Real-World Example

### How Stripe Handles Rate Limiting

Stripe processes billions of dollars in payments. Their rate limiting is sophisticated:

```
Stripe's Multi-Dimensional Rate Limiting:

┌─────────────────────────────────────────────────────┐
│              STRIPE API GATEWAY                      │
│                                                     │
│  Dimension 1: Per API Key                          │
│  ├── Test mode: 25 requests/sec                    │
│  └── Live mode: 100 requests/sec                   │
│                                                     │
│  Dimension 2: Per Endpoint                         │
│  ├── GET /charges: 100/sec (reads are cheap)       │
│  ├── POST /charges: 25/sec (writes are expensive)  │
│  └── POST /refunds: 10/sec (very sensitive)        │
│                                                     │
│  Dimension 3: Per Resource                         │
│  └── Same card charged: max 5 times/hour           │
│                                                     │
│  Dimension 4: Global System Protection             │
│  └── Total system: shed load at 95% capacity       │
└─────────────────────────────────────────────────────┘

Stripe's response headers:
HTTP/1.1 429 Too Many Requests
Stripe-RateLimit-Limit: 100
Stripe-RateLimit-Remaining: 0
Stripe-RateLimit-Reset: 1623456789
```

### How GitHub Handles Rate Limiting

```
GitHub API Rate Limits:

Unauthenticated:  60 requests/hour (per IP)
Authenticated:    5,000 requests/hour (per user)
GitHub Apps:      15,000 requests/hour (per installation)
Search API:       30 requests/minute (separate limit)

GitHub's approach: Different limits per endpoint type
┌────────────────────────────────────────────────────┐
│  Core API      │  5,000/hour                       │
│  Search API    │  30/minute                        │
│  GraphQL       │  5,000 points/hour (cost-based)   │
│  Code Scanning │  1,000/hour                       │
└────────────────────────────────────────────────────┘

Cost-based limiting (GraphQL):
  Simple query:   costs 1 point
  Complex query:  costs 10 points  
  Nested query:   costs 50 points
  Total budget:   5,000 points/hour
```

---

## Advanced: Rate Limiting Strategies

### Strategy 1: Tiered Rate Limiting

```
Different limits for different user tiers:

┌──────────────┬──────────────┬──────────────┬──────────────┐
│    Free      │    Basic     │    Pro       │  Enterprise  │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ 60 req/min   │ 300 req/min  │ 3000 req/min │ 30000 req/min│
│ 1000 req/day │ 10K req/day  │ 100K req/day │ Unlimited    │
│ No burst     │ 2x burst     │ 5x burst     │ 10x burst    │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

### Strategy 2: Adaptive Rate Limiting

```
Adjust limits based on system health:

System Load < 50%:  Allow 100% of configured limits
System Load 50-75%: Allow 75% of configured limits  
System Load 75-90%: Allow 50% of configured limits
System Load > 90%:  Allow 25% of configured limits (emergency)

┌────────────────────────────────────────────────┐
│  System Health Monitor                         │
│                                                │
│  CPU: 82%  ████████░░  [HIGH]                 │
│  MEM: 64%  ██████░░░░  [NORMAL]              │
│  Queue: 91% █████████░  [CRITICAL]            │
│                                                │
│  Decision: Reduce rate limits to 50%           │
│  Reason: Queue depth exceeds threshold         │
└────────────────────────────────────────────────┘
```

### Strategy 3: Cost-Based Rate Limiting

```
Not all requests are equal. Assign costs:

GET  /users         → cost: 1 point   (cheap, cacheable)
GET  /reports       → cost: 5 points  (expensive query)
POST /upload        → cost: 10 points (heavy I/O)
POST /bulk-import   → cost: 50 points (very expensive)

User budget: 1000 points/minute

Example:
  20 × GET /users    = 20 points
  5  × GET /reports  = 25 points
  2  × POST /upload  = 20 points
  ─────────────────────────────
  Total used:          65 points
  Remaining:          935 points
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Solution |
|---------|-------------|----------|
| Rate limiting only by IP | Many users share an IP (NAT, corporate proxy) | Use API keys, user IDs, or combination |
| Not rate limiting internal services | A runaway internal service crashes everything | Apply limits even for service-to-service calls |
| Using local (per-instance) counters | Clients hit different instances, bypass limits | Use Redis for shared counters |
| Not returning rate limit headers | Clients can't implement backoff | Always return X-RateLimit-* headers |
| Same limit for all endpoints | Expensive endpoints get as much traffic as cheap ones | Use cost-based or per-endpoint limits |
| No graceful degradation | Users get cryptic 429 errors | Return helpful error message with retry-after |
| Rate limiting after authentication | DDoS can still overwhelm the auth layer | Apply basic IP rate limiting BEFORE auth |
| Not monitoring rate limit hits | You don't know if limits are too strict or lenient | Alert on rate limit spikes, review weekly |

---

## When to Use / When NOT to Use

### ✅ Use Rate Limiting When:

- You have **public-facing APIs** (always!)
- You need to **protect against DDoS** at the application layer
- You want to **enforce fair usage** across different clients
- You have **expensive operations** (complex queries, file processing)
- You need to **monetize API access** (different tiers)
- You want to prevent **brute-force attacks** on login endpoints

### ❌ Be Careful When:

- **Internal-only services** — rate limiting adds latency; use lighter approaches
- **Real-time systems** — strict rate limits may cause unacceptable delays
- **Rate limits are too strict** — legitimate users get blocked → revenue loss
- **No retry-after headers** — clients don't know when to retry → thundering herd

---

## Key Takeaways

- **Rate limiting = hard stop** (reject request), **Throttling = soft stop** (slow down/queue)
- The **Token Bucket** algorithm is the most widely used — it allows controlled bursts while maintaining average rate
- Use **Redis** for distributed rate limiting across multiple gateway instances (Lua scripts for atomicity)
- Always return **X-RateLimit-*** headers so clients can implement intelligent backoff
- Apply **different limits** per endpoint, per tier, and per operation cost
- Rate limit **before authentication** at the IP level to protect against DDoS
- Monitor rate limit hit rates — too many 429s means your limits might be too strict (or you're under attack)

---

## What's Next?

Now that your gateway can verify identity and protect against abuse, the next chapter covers what happens to the request itself: **Request Routing, Transformation & Aggregation** — how the gateway decides where to send requests and how to modify them along the way.

Next: [04-request-routing-transformation.md](./04-request-routing-transformation.md)
