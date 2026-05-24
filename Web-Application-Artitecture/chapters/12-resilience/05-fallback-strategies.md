# Fallback Strategies — What to Do When Things Fail

> **What you'll learn**: How to design graceful fallback responses when a service is unavailable — from serving cached data to returning defaults to queuing for later — so your users never see a blank error page.

---

## Real-Life Analogy

You walk into a restaurant and order the special pasta dish. The waiter comes back and says:

- **No fallback**: "The kitchen is closed. Go away." (Terrible experience)
- **Bad fallback**: "Here's a raw potato." (Irrelevant, unhelpful)
- **Good fallback**: "The pasta is unavailable tonight, but I can offer you a similar dish from our regular menu." (Useful alternative)
- **Great fallback**: "Let me check if we have yesterday's pasta reheated — it won't be as fresh, but you'll still enjoy it." (Slightly stale but relevant)

In software, a **fallback** is what your system returns when the primary source of data or functionality is unavailable. The goal: **give the user something useful, not an error**.

---

## Core Concept Explained Step-by-Step

### Why Do We Need Fallbacks?

```
┌─────────────────────────────────────────────────────────────────┐
│                  WITHOUT FALLBACKS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User → App → Payment Service (DOWN!)                           │
│                      │                                          │
│                      ▼                                          │
│  User sees: "500 Internal Server Error"                        │
│             "Something went wrong. Please try again."           │
│                                                                 │
│  Result: User leaves. Revenue lost. Trust broken.               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                  WITH FALLBACKS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User → App → Payment Service (DOWN!)                           │
│                      │                                          │
│                      ▼                                          │
│  Fallback activates: "Your order is confirmed!                  │
│                       Payment will be processed shortly."       │
│  (Order queued for payment when service recovers)               │
│                                                                 │
│  Result: User is happy. Order captured. Payment retried later.  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Types of Fallback Strategies

```
┌─────────────────────────────────────────────────────────────────┐
│              FALLBACK STRATEGY HIERARCHY                         │
│         (From best experience to worst)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 🥇 CACHED DATA (stale but recent)                          │
│     "Here's the data from 5 minutes ago"                        │
│     User might not even notice!                                 │
│                                                                 │
│  2. 🥈 DEFAULT/STATIC DATA                                     │
│     "Here are our most popular items"                           │
│     Generic but useful                                          │
│                                                                 │
│  3. 🥉 DEGRADED FUNCTIONALITY                                  │
│     "Search works, but without spell correction"                │
│     Core feature available, extras missing                      │
│                                                                 │
│  4. 🏅 QUEUE FOR LATER                                         │
│     "Your request is saved, we'll process it soon"              │
│     Async fulfillment when service recovers                     │
│                                                                 │
│  5. 📋 ALTERNATIVE SERVICE / PROVIDER                          │
│     "Primary payment failed, trying backup provider"            │
│     Redundancy at the vendor level                              │
│                                                                 │
│  6. ⚠️ GRACEFUL ERROR MESSAGE                                  │
│     "This feature is temporarily unavailable"                   │
│     Last resort — at least explain what's happening             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Fallback Decision Tree

```
┌──────────────────┐
│  Primary Call     │
│  (e.g., API call)│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     ✓ Success
│  Did it succeed?  │────────────────▶ Return Response
└────────┬─────────┘
         │ ✗ Failed (timeout, circuit open, 5xx, bulkhead full)
         ▼
┌──────────────────┐     ✓ Fresh cache exists
│  Check Cache?     │────────────────▶ Return Cached Data
└────────┬─────────┘                   (with "stale" indicator)
         │ ✗ No cache / expired
         ▼
┌──────────────────┐     ✓ Default available
│  Has Default?     │────────────────▶ Return Default Data
└────────┬─────────┘
         │ ✗ No default makes sense
         ▼
┌──────────────────┐     ✓ Can queue
│  Can Queue?       │────────────────▶ Queue & Confirm Later
└────────┬─────────┘
         │ ✗ Must respond now
         ▼
┌──────────────────┐     ✓ Backup exists
│  Alt Provider?    │────────────────▶ Try Backup Provider
└────────┬─────────┘
         │ ✗ No alternative
         ▼
┌──────────────────┐
│ Graceful Error    │────────────────▶ "Temporarily unavailable"
│ (user-friendly)   │
└──────────────────┘
```

### Fallback Patterns by Use Case

```
┌────────────────────┬───────────────────────┬──────────────────────────┐
│ Feature            │ Primary Source         │ Fallback Strategy         │
├────────────────────┼───────────────────────┼──────────────────────────┤
│ Product recs       │ ML Recommendation     │ Cached recs / Popular    │
│                    │ Service               │ items / "Trending Now"   │
├────────────────────┼───────────────────────┼──────────────────────────┤
│ User profile       │ User Service          │ Cached profile / "Guest" │
├────────────────────┼───────────────────────┼──────────────────────────┤
│ Payment            │ Stripe/Razorpay       │ Queue for retry / Alt    │
│ processing         │                       │ payment provider         │
├────────────────────┼───────────────────────┼──────────────────────────┤
│ Search results     │ Elasticsearch         │ Basic DB search / Cached │
│                    │                       │ popular searches         │
├────────────────────┼───────────────────────┼──────────────────────────┤
│ Price calculation  │ Pricing Service       │ Last known price from    │
│                    │                       │ cache                    │
├────────────────────┼───────────────────────┼──────────────────────────┤
│ Inventory count    │ Inventory Service     │ "In Stock" (optimistic)  │
│                    │                       │ or "Check availability"  │
├────────────────────┼───────────────────────┼──────────────────────────┤
│ Notifications      │ Email/Push Service    │ Queue for later delivery │
├────────────────────┼───────────────────────┼──────────────────────────┤
│ Analytics/Tracking │ Analytics Service     │ Drop silently (non-      │
│                    │                       │ critical)                │
└────────────────────┴───────────────────────┴──────────────────────────┘
```

---

## Code Examples

### Python — Multi-Level Fallback Chain

```python
import redis
import json
import logging
from typing import Optional, List
from dataclasses import dataclass

logger = logging.getLogger(__name__)
redis_client = redis.Redis(host='redis', port=6379, socket_timeout=1)

@dataclass
class Product:
    id: str
    name: str
    price: float

def get_recommendations(user_id: str) -> List[Product]:
    """
    Multi-level fallback for product recommendations.
    
    Chain:  ML Service → Redis Cache → Static Defaults
    """
    
    # Level 1: Try the primary source (ML recommendation service)
    try:
        recs = call_recommendation_service(user_id)
        # Cache successful results for future fallback use
        cache_recommendations(user_id, recs)
        return recs
    except (TimeoutError, ConnectionError, ServiceUnavailableError) as e:
        logger.warning(f"Rec service failed for {user_id}: {e}")
    
    # Level 2: Try cached recommendations (stale but personalized)
    try:
        cached = get_cached_recommendations(user_id)
        if cached:
            logger.info(f"Serving cached recs for {user_id}")
            return cached
    except redis.RedisError as e:
        logger.warning(f"Redis fallback failed: {e}")
    
    # Level 3: Static popular items (not personalized, but useful)
    logger.info(f"Serving default recs for {user_id}")
    return get_popular_items()

def call_recommendation_service(user_id: str) -> List[Product]:
    """Call the ML recommendation service."""
    import requests
    response = requests.get(
        f"http://rec-service/recommendations/{user_id}",
        timeout=(2, 5)
    )
    response.raise_for_status()
    return [Product(**p) for p in response.json()["products"]]

def cache_recommendations(user_id: str, recs: List[Product]):
    """Cache recommendations for fallback use. TTL = 1 hour."""
    try:
        data = json.dumps([vars(r) for r in recs])
        redis_client.setex(f"recs:{user_id}", 3600, data)
    except redis.RedisError:
        pass  # Best effort — don't fail if caching fails

def get_cached_recommendations(user_id: str) -> Optional[List[Product]]:
    """Get cached recommendations (may be stale)."""
    data = redis_client.get(f"recs:{user_id}")
    if data:
        return [Product(**p) for p in json.loads(data)]
    return None

def get_popular_items() -> List[Product]:
    """Static fallback — hardcoded popular items."""
    return [
        Product("pop-1", "Best Seller #1", 29.99),
        Product("pop-2", "Best Seller #2", 19.99),
        Product("pop-3", "Trending Item", 39.99),
    ]
```

### Python — Fallback with Queue for Later Processing

```python
import json
from datetime import datetime
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers='kafka:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

class PaymentService:
    """Payment with queue-based fallback."""
    
    def process_payment(self, order_id: str, amount: float, user_id: str):
        """
        Try to process payment immediately.
        If payment service is down, queue it for later.
        """
        try:
            # Try primary payment
            result = self._charge_immediately(order_id, amount)
            return {"status": "completed", "transaction_id": result["id"]}
            
        except (TimeoutError, ServiceUnavailableError) as e:
            # Fallback: Queue for async processing
            self._queue_payment(order_id, amount, user_id)
            return {
                "status": "pending",
                "message": "Payment will be processed shortly",
                "estimated_completion": "< 5 minutes"
            }
    
    def _charge_immediately(self, order_id, amount):
        import requests
        response = requests.post(
            "http://payment-service/charge",
            json={"order_id": order_id, "amount": amount},
            timeout=(3, 10)
        )
        response.raise_for_status()
        return response.json()
    
    def _queue_payment(self, order_id, amount, user_id):
        """Send to Kafka for retry by a background worker."""
        event = {
            "type": "payment_retry",
            "order_id": order_id,
            "amount": amount,
            "user_id": user_id,
            "queued_at": datetime.utcnow().isoformat(),
            "max_retries": 5
        }
        producer.send("payment-retry-queue", value=event)
        producer.flush()
```

### Java — Fallback with Circuit Breaker

```java
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

@Service
public class ProductCatalogService {
    
    private final ProductServiceClient productClient;
    private final CacheService cacheService;
    private final StaticDataProvider staticData;
    
    // Primary call protected by circuit breaker
    @CircuitBreaker(name = "productService", fallbackMethod = "getProductsFallback")
    public List<Product> getProducts(String category) {
        List<Product> products = productClient.getByCategory(category);
        // Cache successful results for fallback use
        cacheService.cacheProducts(category, products);
        return products;
    }
    
    // Fallback method — called when circuit is open or call fails
    private List<Product> getProductsFallback(String category, Exception ex) {
        System.out.println("Primary failed: " + ex.getMessage() + 
                          ". Trying fallback chain...");
        
        // Fallback Level 1: Try cache
        Optional<List<Product>> cached = cacheService.getCachedProducts(category);
        if (cached.isPresent()) {
            System.out.println("Serving cached products for: " + category);
            return cached.get();
        }
        
        // Fallback Level 2: Static/default data
        System.out.println("Serving static popular products");
        return staticData.getPopularProducts();
    }
}
```

### Java — Multiple Fallback Providers

```java
import java.util.List;

@Service
public class PaymentGateway {
    
    private final List<PaymentProvider> providers;  // Ordered by priority
    
    public PaymentGateway() {
        // Multiple payment providers as fallbacks
        this.providers = List.of(
            new StripeProvider(),      // Primary
            new RazorpayProvider(),    // Fallback 1
            new PayPalProvider()       // Fallback 2
        );
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        Exception lastException = null;
        
        for (PaymentProvider provider : providers) {
            try {
                if (!provider.isHealthy()) {
                    System.out.println(provider.getName() + " is unhealthy, skipping");
                    continue;
                }
                
                PaymentResult result = provider.charge(request);
                System.out.println("Payment processed via: " + provider.getName());
                return result;
                
            } catch (PaymentException e) {
                lastException = e;
                System.out.println(provider.getName() + " failed: " + e.getMessage());
                // Try next provider
            }
        }
        
        // All providers failed — queue for later
        queuePaymentForRetry(request);
        return PaymentResult.pending(
            "Payment queued for processing. " +
            "You'll receive confirmation shortly.");
    }
}
```

---

## Infrastructure Examples

### Nginx — Fallback to Static Content

```nginx
# If backend is down, serve a static cached page
upstream backend {
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
}

server {
    listen 80;
    
    location /api/products {
        proxy_pass http://backend;
        proxy_connect_timeout 3s;
        proxy_read_timeout 5s;
        
        # If ALL backends fail, serve cached JSON
        proxy_intercept_errors on;
        error_page 502 503 504 = @fallback_cache;
    }
    
    location @fallback_cache {
        # Serve pre-generated static JSON as fallback
        root /var/www/fallback-cache;
        try_files /products-cache.json =503;
        add_header X-Fallback "true";  # Signal to client: this is cached
        add_header Cache-Control "public, max-age=60";
    }
}
```

### Redis — Cache-Aside Pattern for Fallback

```python
import redis
import json

r = redis.Redis(host='redis-cluster', port=6379, socket_timeout=1)

class CacheFallbackMixin:
    """Mixin that adds cache-based fallback to any service client."""
    
    def call_with_cache_fallback(self, cache_key: str, primary_func, ttl=3600):
        """
        Try primary function. If it fails, return cached result.
        If primary succeeds, update cache for future fallbacks.
        """
        try:
            # Try primary source
            result = primary_func()
            
            # Success! Cache it for future fallback use
            self._update_cache(cache_key, result, ttl)
            return result, "live"
            
        except Exception as primary_error:
            # Primary failed — try cache
            cached = self._get_from_cache(cache_key)
            if cached is not None:
                return cached, "cached"
            
            # No cache either — re-raise
            raise primary_error
    
    def _update_cache(self, key, data, ttl):
        try:
            r.setex(f"fallback:{key}", ttl, json.dumps(data))
        except redis.RedisError:
            pass  # Don't fail if cache update fails
    
    def _get_from_cache(self, key):
        try:
            data = r.get(f"fallback:{key}")
            return json.loads(data) if data else None
        except (redis.RedisError, json.JSONDecodeError):
            return None
```

### Kubernetes — Readiness-Based Traffic Shifting

```yaml
# When a service becomes unhealthy, K8s removes it from service endpoint
# Combined with a fallback service:

# Primary service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recommendation-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: recommendation
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          periodSeconds: 5
          failureThreshold: 2   # Mark unready after 2 failures
---
# Fallback service (serves cached/static recommendations)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recommendation-fallback
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: recommendation-static
        image: rec-service-static:latest  # Pre-built static data
```

---

## Real-World Example

### Netflix — "The Netflix Effect"

Netflix shows how fallbacks create seamless user experience:

```
┌─────────────────────────────────────────────────────────────────────┐
│            NETFLIX FALLBACK STRATEGY PER FEATURE                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐                                               │
│  │ HOMEPAGE ROWS    │                                               │
│  ├─────────────────┤                                               │
│  │ Primary: ML recs │──failed──▶ Cached recs (last successful)     │
│  │                  │──failed──▶ Popular in your country            │
│  │                  │──failed──▶ Generic "Top 10" list              │
│  └─────────────────┘                                               │
│                                                                     │
│  ┌─────────────────┐                                               │
│  │ SEARCH           │                                               │
│  ├─────────────────┤                                               │
│  │ Primary: Search  │──failed──▶ Pre-indexed popular queries       │
│  │ Service          │──failed──▶ "Search is temporarily slow"      │
│  └─────────────────┘                                               │
│                                                                     │
│  ┌─────────────────┐                                               │
│  │ VIDEO STREAMING  │                                               │
│  ├─────────────────┤                                               │
│  │ Primary: Optimal │──failed──▶ Lower quality stream              │
│  │ quality (4K)     │──failed──▶ SD stream from CDN cache          │
│  │                  │──failed──▶ "Try again in a moment"           │
│  └─────────────────┘                                               │
│                                                                     │
│  ┌─────────────────┐                                               │
│  │ USER PROFILE     │                                               │
│  ├─────────────────┤                                               │
│  │ Primary: User    │──failed──▶ Cached profile (stale)            │
│  │ Service          │──failed──▶ Generic profile (default avatar)  │
│  └─────────────────┘                                               │
│                                                                     │
│  KEY INSIGHT: Most Netflix outages are invisible to users           │
│  because EVERY feature has a fallback chain.                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Amazon E-Commerce Fallback Strategy

```
When Amazon's Recommendation Engine goes down:

Normal:     "Customers who bought X also bought Y, Z, W..."
Fallback 1: "Based on your recent purchases..."  (cached recs)
Fallback 2: "Best sellers in this category..."    (static popular)
Fallback 3:  Hide the recommendation section entirely (degrade UI)
             Main product page still works perfectly!

Result: User can still browse & buy. Revenue impact minimized.
```

---

## Common Mistakes / Pitfalls

### 1. Fallback Returns Completely Irrelevant Data

```
❌ BAD: User searches "iPhone 15" → Fallback shows "Popular garden tools"
   (Misleading and confusing)

✅ GOOD: User searches "iPhone 15" → Fallback shows "iPhone category page"
   or "Search is temporarily limited, try browsing categories"
```

### 2. Fallback Has the Same Dependency as Primary

```
❌ BAD:
   Primary: Call recommendation-service
   Fallback: Call recommendation-service (different endpoint)
   
   If the service is DOWN, both fail!

✅ GOOD:
   Primary: Call recommendation-service
   Fallback: Read from Redis cache (independent system)
```

### 3. Silent Failures — Nobody Knows Fallback is Active

```
❌ BAD: Fallback activates silently, no logging, no alerting
   → Service is down for hours, nobody notices
   → Users get stale data without anyone investigating

✅ GOOD: 
   - Log every fallback activation
   - Emit metrics: "fallback_activated_total" counter
   - Alert if fallback rate > 5% for > 5 minutes
   - Set response header: "X-Data-Source: cache-fallback"
```

### 4. Stale Cache Without Expiration

```
❌ BAD: Cache price=$100. Service is down for 3 days.
   Price changed to $150 on day 2.
   Users see $100 for days — orders at wrong price!

✅ GOOD: 
   - Cache TTL: 1 hour (re-validate regularly)
   - Max stale limit: 24 hours (after that, show "unavailable")
   - Show "prices last updated: 2 hours ago" to user
```

### 5. Fallback for WRITE Operations That Loses Data

```
❌ BAD: User places order → Service down → Return "success" with no actual processing
   (User thinks order went through, but it's lost!)

✅ GOOD: User places order → Service down → Queue in Kafka/SQS
   → Return "Order received, confirming shortly"
   → Background worker processes when service recovers
   → Send confirmation email when actually complete
```

---

## When to Use / When NOT to Use

### ✅ Use Fallbacks When

| Scenario | Fallback Type |
|----------|--------------|
| Non-critical features (recs, ads) | Cached/default data |
| Read operations (product catalog) | Stale cache |
| Notification sending | Queue for later |
| Third-party API (weather, exchange rate) | Cached last value |
| Search functionality | Simpler search / category browse |
| Payment processing | Alternative provider or queue |
| Image/video serving | Lower quality / placeholder |

### ❌ When NOT to Use / Be Very Careful

| Scenario | Why |
|----------|-----|
| Financial transactions | Can't fake a bank transfer |
| Legal compliance checks | Must have real-time accuracy |
| Security authentication | Can't assume someone is authorized |
| Medical/safety systems | Wrong data could harm people |
| Real-time stock trading | Stale prices = financial loss |

### ⚠️ Fallback Decision Matrix

```
                    ┌─────────────────────────────────────┐
                    │ Can user tolerate stale/default data?│
                    └───────┬─────────────────┬───────────┘
                      YES   │                 │  NO
                            ▼                 ▼
                  ┌─────────────────┐ ┌─────────────────────┐
                  │ Use cache/default│ │ Can we process later?│
                  │ fallback        │ └──────┬──────────┬────┘
                  └─────────────────┘   YES  │          │ NO
                                             ▼          ▼
                                  ┌────────────────┐ ┌─────────┐
                                  │ Queue for later│ │ Show     │
                                  │ processing     │ │ graceful │
                                  └────────────────┘ │ error    │
                                                     └─────────┘
```

---

## Key Takeaways

- **Every external dependency should have a fallback plan** — the question is not "if" it fails, but "when."
- **Cache successful responses** specifically for fallback use — this is your insurance policy.
- **Match fallback to feature criticality** — payment needs queuing, recommendations can use defaults.
- **Never silently return wrong data** — indicate to users and monitoring that a fallback is active.
- **Fallbacks must use DIFFERENT infrastructure** than the primary — if both share a database and it's down, both fail.
- **Monitor fallback activation** — a spike means something is broken and needs investigation.
- **Test your fallbacks regularly** — untested fallbacks often have bugs and fail when you need them most.

---

## What's Next?

Fallbacks help you respond to individual failures. But what happens when your system is receiving MORE traffic than it can handle? You need to push back — that's **Backpressure**. See [06-backpressure.md](./06-backpressure.md).
