# Graceful Degradation — Keep Core Features Working

> **What you'll learn**: How to design systems that progressively shed non-essential functionality during failures, keeping core user experiences intact — like a car that loses air conditioning but keeps driving.

---

## Real-Life Analogy

Imagine a **commercial airplane** experiencing an engine problem mid-flight:

- **No graceful degradation**: Engine fails → entire plane shuts down → crash. (Terrible design!)
- **Graceful degradation**: Engine fails → plane flies on remaining engines → non-critical systems (entertainment, overhead lights) power down → pilot diverts to nearest airport → all passengers land safely.

The plane **degrades gracefully** — it sheds non-essential features to keep the critical function (flying + landing safely) working.

Your web application should work the same way:

```
FULL FUNCTIONALITY (everything working):
┌──────────────────────────────────────────────┐
│  🛒 Shopping  │  ⭐ Reviews │  📦 Tracking  │
│  ✓ Search     │  ✓ Ratings │  ✓ Real-time  │
│  ✓ Cart       │  ✓ Photos  │  ✓ Map view   │
│  ✓ Checkout   │  ✓ Q&A     │  ✓ Estimates  │
│  ✓ Recommendations │ ✓ AI Summary │          │
└──────────────────────────────────────────────┘

DEGRADED (review service down):
┌──────────────────────────────────────────────┐
│  🛒 Shopping  │  ⭐ Reviews │  📦 Tracking  │
│  ✓ Search     │  ⚠️ Cached │  ✓ Real-time  │
│  ✓ Cart       │  ⚠️ No new │  ✓ Map view   │
│  ✓ Checkout   │  ✗ No Q&A  │  ✓ Estimates  │
│  ✓ Recommendations │ ✗ No AI   │             │
└──────────────────────────────────────────────┘
Core shopping works! Users can still buy things.


SEVERELY DEGRADED (multiple services down):
┌──────────────────────────────────────────────┐
│  🛒 Shopping  │  ⭐ Reviews │  📦 Tracking  │
│  ✓ Search     │  ✗ Hidden  │  ⚠️ Last known│
│  ✓ Cart       │            │               │
│  ✓ Checkout   │            │               │
│  ✗ No recs    │            │               │
└──────────────────────────────────────────────┘
Absolute core (browse + buy) still works!
```

---

## Core Concept Explained Step-by-Step

### What is Graceful Degradation?

**Graceful degradation** is a design philosophy where a system continues to operate at reduced functionality rather than failing completely when some components fail.

```
┌─────────────────────────────────────────────────────────────────┐
│         GRACEFUL DEGRADATION SPECTRUM                            │
│                                                                 │
│  Full Service ──────────────────────────▶ Complete Outage       │
│       │                                          │              │
│       ▼                                          ▼              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐         │
│  │ FULL    │  │ PARTIAL │  │ MINIMAL │  │ COMPLETE │         │
│  │ SERVICE │  │ DEGRADED│  │ DEGRADED│  │ OUTAGE   │         │
│  │         │  │         │  │         │  │          │         │
│  │ All     │  │ Some    │  │ Only    │  │ Nothing  │         │
│  │ features│  │ features│  │ core    │  │ works    │         │
│  │ work    │  │ limited │  │ works   │  │          │         │
│  └─────────┘  └─────────┘  └─────────┘  └──────────┘         │
│      100%         70%          30%            0%               │
│                                                                 │
│  GOAL: Stay in the first 3 boxes. Never reach "Complete Outage"│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Feature Priority Classification

The first step is classifying every feature by business criticality:

```
┌────────────────────────────────────────────────────────────────────┐
│           FEATURE PRIORITY TIERS                                   │
├────────┬──────────────────────────────────┬────────────────────────┤
│ Tier   │ Features                         │ Degradation Strategy   │
├────────┼──────────────────────────────────┼────────────────────────┤
│ CRITICAL│ Login, Search, Add to Cart,     │ NEVER degrade          │
│ (Tier 0)│ Checkout, Payment              │ Scale/failover instead │
├────────┼──────────────────────────────────┼────────────────────────┤
│ IMPORTANT│ Product pages, Order history,  │ Serve cached/stale     │
│ (Tier 1) │ Account settings, Notifications│ data if needed         │
├────────┼──────────────────────────────────┼────────────────────────┤
│ NICE TO │ Recommendations, Reviews,       │ Hide section / show    │
│ HAVE    │ "Customers also bought",        │ defaults when down     │
│ (Tier 2)│ Recently viewed, Wishlist       │                        │
├────────┼──────────────────────────────────┼────────────────────────┤
│ OPTIONAL│ Analytics tracking, A/B tests,  │ Silently disable       │
│ (Tier 3)│ Personalized banners, Surveys  │ (user doesn't notice)  │
└────────┴──────────────────────────────────┴────────────────────────┘
```

### Degradation Decision Flow

```
System health check detects: Service X is DOWN
                    │
                    ▼
┌─────────────────────────────────┐
│ What tier is this service?       │
└────────┬──────────┬──────┬──────┘
         │          │      │
    Tier 0      Tier 1   Tier 2/3
         │          │      │
         ▼          ▼      ▼
┌────────────┐ ┌──────────┐ ┌─────────────────┐
│ PANIC!      │ │ Serve    │ │ Hide feature    │
│ Failover!   │ │ cached   │ │ Show default    │
│ Page ops!   │ │ or stale │ │ Silently skip   │
│ ALL HANDS!  │ │ data     │ │                 │
└────────────┘ └──────────┘ └─────────────────┘
```

---

## How It Works Internally

### Feature Flag-Driven Degradation

```
┌─────────────────────────────────────────────────────────────────┐
│         FEATURE FLAGS FOR DEGRADATION                            │
│                                                                 │
│  Configuration Store (Consul / LaunchDarkly / Redis)             │
│  ┌───────────────────────────────────────────────┐             │
│  │  feature.recommendations.enabled = false       │             │
│  │  feature.reviews.enabled = false               │             │
│  │  feature.search.mode = "basic"  (not "ml")    │             │
│  │  feature.checkout.enabled = true               │             │
│  │  feature.payments.provider = "backup"          │             │
│  └───────────────────────────────────────────────┘             │
│                                                                 │
│  Application reads flags on each request:                       │
│                                                                 │
│  if feature_flags.get("recommendations.enabled"):               │
│      show_recommendations()                                    │
│  else:                                                          │
│      show_popular_items()   # Degraded but functional          │
│                                                                 │
│  Flags can be:                                                  │
│  - Set MANUALLY by on-call engineer during incidents            │
│  - Set AUTOMATICALLY by health check system                    │
│  - Changed in SECONDS (no deploy needed!)                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Health-Based Automatic Degradation

```
┌─────────────────────────────────────────────────────────────────┐
│       AUTOMATIC DEGRADATION SYSTEM                              │
│                                                                 │
│  ┌──────────────┐    health status    ┌─────────────────┐      │
│  │  Health Check │ ─────────────────▶  │  Degradation    │      │
│  │  Service      │                     │  Controller     │      │
│  └──────────────┘                     └────────┬────────┘      │
│         │                                       │               │
│    Monitors:                              Updates:               │
│    - Service A: ✓ healthy                 - Feature flags       │
│    - Service B: ✗ unhealthy              - Load balancer rules  │
│    - Service C: ⚠️ degraded              - CDN configs          │
│    - DB: ✓ healthy                       - Alert channels       │
│    - Redis: ✗ down                                              │
│                                                                 │
│  Rules engine:                                                  │
│  ┌────────────────────────────────────────────────────┐        │
│  │ IF rec-service.health == DOWN:                      │        │
│  │   SET feature.recommendations.enabled = false       │        │
│  │   SET feature.homepage.layout = "simplified"        │        │
│  │                                                     │        │
│  │ IF redis.health == DOWN:                            │        │
│  │   SET feature.sessions.store = "database"           │        │
│  │   SET feature.caching.enabled = false               │        │
│  │                                                     │        │
│  │ IF payment-primary.health == DOWN:                  │        │
│  │   SET feature.payment.provider = "backup-razorpay"  │        │
│  └────────────────────────────────────────────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Progressive Load Shedding

```
System Load Level:    NORMAL    HIGH      CRITICAL   EXTREME
                        │        │           │          │
                        ▼        ▼           ▼          ▼
                    
Tier 3 (Optional):  ✓ ON     ✗ OFF      ✗ OFF      ✗ OFF
  - Analytics
  - A/B tests
  
Tier 2 (Nice):      ✓ ON     ✓ ON       ✗ OFF      ✗ OFF
  - Recommendations
  - Reviews
  
Tier 1 (Important): ✓ ON     ✓ ON       ⚠️ CACHED  ✗ LIMITED
  - Product details
  - User profiles
  
Tier 0 (Critical):  ✓ ON     ✓ ON       ✓ ON       ✓ ON
  - Search
  - Cart + Checkout
  - Payment

As load increases, we shed features starting from the BOTTOM tier.
Critical features are the LAST to be affected.
```

---

## Code Examples

### Python — Degradation-Aware Service Layer

```python
from enum import Enum
from typing import Optional, Any
import logging

logger = logging.getLogger(__name__)

class SystemHealth(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    CRITICAL = "critical"

class DegradationManager:
    """Manages feature degradation based on system health."""
    
    def __init__(self, config_client):
        self.config = config_client  # Consul / Redis / Feature flag service
    
    def is_feature_enabled(self, feature: str) -> bool:
        """Check if a feature should be active given current health."""
        return self.config.get_bool(f"feature.{feature}.enabled", default=True)
    
    def get_degradation_level(self) -> SystemHealth:
        """Get current system health level."""
        level = self.config.get("system.health.level", "healthy")
        return SystemHealth(level)


class ProductPageService:
    """Builds product page with graceful degradation."""
    
    def __init__(self, degradation: DegradationManager):
        self.degradation = degradation
        self.product_client = ProductClient()
        self.review_client = ReviewClient()
        self.rec_client = RecommendationClient()
        self.cache = RedisCache()
    
    def get_product_page(self, product_id: str) -> dict:
        """
        Build product page. Non-critical sections degrade gracefully.
        Core product info is ALWAYS returned.
        """
        
        # TIER 0: Product details (CRITICAL — must always work)
        product = self._get_product_details(product_id)
        
        page = {
            "product": product,
            "reviews": None,
            "recommendations": None,
            "recently_viewed": None,
            "degraded_features": []
        }
        
        # TIER 2: Reviews (nice to have)
        if self.degradation.is_feature_enabled("reviews"):
            try:
                page["reviews"] = self.review_client.get_reviews(product_id)
            except Exception as e:
                logger.warning(f"Reviews degraded for {product_id}: {e}")
                page["reviews"] = self._get_cached_reviews(product_id)
                page["degraded_features"].append("reviews")
        else:
            page["degraded_features"].append("reviews")
        
        # TIER 2: Recommendations (nice to have)
        if self.degradation.is_feature_enabled("recommendations"):
            try:
                page["recommendations"] = self.rec_client.get_recs(product_id)
            except Exception as e:
                logger.warning(f"Recommendations degraded: {e}")
                page["recommendations"] = self._get_popular_in_category(
                    product["category"])
                page["degraded_features"].append("recommendations")
        else:
            page["degraded_features"].append("recommendations")
        
        # TIER 3: Recently viewed (optional — silently skip)
        if self.degradation.is_feature_enabled("recently_viewed"):
            try:
                page["recently_viewed"] = self._get_recently_viewed()
            except Exception:
                pass  # Silently skip — user won't miss it
        
        return page
    
    def _get_product_details(self, product_id: str):
        """CRITICAL — try primary, then cache, then basic DB."""
        try:
            return self.product_client.get(product_id)
        except Exception:
            cached = self.cache.get(f"product:{product_id}")
            if cached:
                return cached
            raise  # If we can't show product at all, propagate error
    
    def _get_cached_reviews(self, product_id):
        """Return stale cached reviews as fallback."""
        return self.cache.get(f"reviews:{product_id}") or {
            "average_rating": None,
            "count": 0,
            "reviews": [],
            "note": "Reviews temporarily unavailable"
        }
    
    def _get_popular_in_category(self, category):
        """Static fallback for recommendations."""
        return self.cache.get(f"popular:{category}") or []
```

### Python — Load-Based Degradation Middleware

```python
import time
import psutil
from fastapi import FastAPI, Request, Response
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

class GracefulDegradationMiddleware(BaseHTTPMiddleware):
    """Automatically sheds features based on system load."""
    
    # Features in order of importance (shed from bottom first)
    FEATURE_PRIORITY = [
        ("analytics", 3),       # First to shed
        ("ab_tests", 3),
        ("recommendations", 2),
        ("reviews", 2),
        ("notifications", 2),
        ("search_autocomplete", 1),
        ("core", 0),            # Never shed
    ]
    
    THRESHOLDS = {
        "normal": {"cpu": 60, "memory": 70, "latency_p99_ms": 500},
        "high": {"cpu": 75, "memory": 80, "latency_p99_ms": 1000},
        "critical": {"cpu": 90, "memory": 90, "latency_p99_ms": 3000},
    }
    
    async def dispatch(self, request: Request, call_next):
        # Determine current load level
        load_level = self._get_load_level()
        
        # Determine which tier to shed
        shed_tier = self._get_shed_tier(load_level)
        
        # Add degradation context to request
        request.state.shed_tier = shed_tier
        request.state.load_level = load_level
        
        response = await call_next(request)
        
        # Add headers so clients know about degradation
        if shed_tier > 0:
            response.headers["X-Degraded"] = "true"
            response.headers["X-Degraded-Tier"] = str(shed_tier)
            response.headers["X-Load-Level"] = load_level
        
        return response
    
    def _get_load_level(self) -> str:
        cpu = psutil.cpu_percent()
        memory = psutil.virtual_memory().percent
        
        if cpu > 90 or memory > 90:
            return "critical"
        elif cpu > 75 or memory > 80:
            return "high"
        return "normal"
    
    def _get_shed_tier(self, load_level: str) -> int:
        """Return the minimum tier that should still be active."""
        if load_level == "critical":
            return 0  # Only Tier 0 (critical) features
        elif load_level == "high":
            return 1  # Tier 0 + Tier 1
        return 3  # All features active

app.add_middleware(GracefulDegradationMiddleware)

@app.get("/api/products/{product_id}")
async def get_product(product_id: str, request: Request):
    shed_tier = request.state.shed_tier
    
    product = get_product_from_db(product_id)  # Always
    
    result = {"product": product}
    
    if shed_tier <= 2:  # Include recommendations only if tier ≤ 2 is active
        result["recommendations"] = get_recommendations(product_id)
    
    if shed_tier <= 1:  # Include reviews only if tier ≤ 1 is active
        result["reviews"] = get_reviews(product_id)
    
    return result
```

### Java — Spring Boot Degradation with Health Indicators

```java
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class DegradationAwareController {
    
    private final DegradationManager degradationManager;
    private final ProductService productService;
    private final ReviewService reviewService;
    private final RecommendationService recService;
    
    @GetMapping("/api/products/{id}")
    public ResponseEntity<ProductPageResponse> getProductPage(
            @PathVariable String id) {
        
        ProductPageResponse.Builder builder = ProductPageResponse.builder();
        
        // TIER 0: Always fetch product (critical)
        builder.product(productService.getProduct(id));
        
        // TIER 1: Reviews — degrade if service unhealthy
        if (degradationManager.isEnabled("reviews")) {
            try {
                builder.reviews(reviewService.getReviews(id));
            } catch (ServiceException e) {
                builder.reviews(reviewService.getCachedReviews(id));
                builder.addDegradedFeature("reviews");
            }
        } else {
            builder.addDegradedFeature("reviews");
        }
        
        // TIER 2: Recommendations — skip if load is high
        if (degradationManager.isEnabled("recommendations")) {
            try {
                builder.recommendations(recService.getRecommendations(id));
            } catch (ServiceException e) {
                builder.recommendations(recService.getPopularItems());
                builder.addDegradedFeature("recommendations");
            }
        } else {
            builder.addDegradedFeature("recommendations");
        }
        
        ProductPageResponse response = builder.build();
        
        HttpHeaders headers = new HttpHeaders();
        if (!response.getDegradedFeatures().isEmpty()) {
            headers.add("X-Degraded-Features", 
                String.join(",", response.getDegradedFeatures()));
        }
        
        return ResponseEntity.ok().headers(headers).body(response);
    }
}

@Service
public class DegradationManager {
    
    private final HealthRegistry healthRegistry;
    private final FeatureFlagClient featureFlags;
    
    public boolean isEnabled(String feature) {
        // Check feature flag first (manual override)
        if (!featureFlags.isEnabled("feature." + feature)) {
            return false;
        }
        
        // Check if dependent service is healthy
        String dependentService = getDependentService(feature);
        if (dependentService != null) {
            Health health = healthRegistry.getHealth(dependentService);
            return health.getStatus() != Status.DOWN;
        }
        
        return true;
    }
}
```

---

## Infrastructure Examples

### Feature Flags with LaunchDarkly / Consul

```python
# Consul-based degradation flags
# Can be toggled instantly during incidents

# PUT to Consul KV to degrade a feature:
# curl -X PUT -d 'false' http://consul:8500/v1/kv/features/recommendations/enabled
# curl -X PUT -d 'basic' http://consul:8500/v1/kv/features/search/mode

import consul

class ConsulDegradationConfig:
    def __init__(self):
        self.client = consul.Consul(host='consul', port=8500)
    
    def is_feature_enabled(self, feature: str) -> bool:
        _, data = self.client.kv.get(f'features/{feature}/enabled')
        if data is None:
            return True  # Default: enabled
        return data['Value'].decode() == 'true'
    
    def get_feature_mode(self, feature: str) -> str:
        _, data = self.client.kv.get(f'features/{feature}/mode')
        if data is None:
            return 'full'
        return data['Value'].decode()
```

### Nginx — Static Fallback Pages During Outage

```nginx
# Serve a static "degraded" version when backend is completely down
upstream backend {
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
}

server {
    listen 80;
    root /var/www/static-fallback;
    
    # Try backend first, fall back to static page
    location / {
        proxy_pass http://backend;
        proxy_connect_timeout 2s;
        proxy_read_timeout 5s;
        
        # If backend returns 5xx or is unreachable:
        proxy_intercept_errors on;
        error_page 500 502 503 504 = @static_fallback;
    }
    
    location @static_fallback {
        # Serve pre-rendered static version of the page
        try_files /degraded$uri /degraded/index.html =503;
        add_header X-Served-By "static-fallback";
        add_header Cache-Control "public, max-age=60";
    }
}
```

### Kubernetes — Priority-Based Resource Allocation

```yaml
# PriorityClass ensures critical services get resources first
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-service
value: 1000000        # Highest priority
description: "Critical services (checkout, payment)"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: standard-service
value: 100000
description: "Standard services (product catalog)"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: non-critical-service
value: 1000           # Lowest priority — evicted first under pressure
description: "Non-critical (analytics, recommendations)"
---
# Under memory pressure, K8s evicts pods in priority order:
# non-critical → standard → critical (last to be evicted)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recommendation-service
spec:
  template:
    spec:
      priorityClassName: non-critical-service  # Evicted first!
      containers:
      - name: recs
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
```

---

## Real-World Example

### Netflix's "The Netflix Tax"

Netflix is the gold standard for graceful degradation:

```
┌─────────────────────────────────────────────────────────────────────┐
│            NETFLIX'S DEGRADATION PHILOSOPHY                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PRINCIPLE: "A user watching a movie with a bad recommendation     │
│  is better than a user staring at an error page."                   │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Service Down          │  User Experience                   │    │
│  ├────────────────────────┼────────────────────────────────────┤    │
│  │  Personalization       │  Show "Popular in [Country]"       │    │
│  │  Service               │  instead of personalized rows      │    │
│  ├────────────────────────┼────────────────────────────────────┤    │
│  │  Artwork Service       │  Show default poster instead of    │    │
│  │  (personalized art)    │  personalized artwork              │    │
│  ├────────────────────────┼────────────────────────────────────┤    │
│  │  Search Service        │  Show "Search is slow" + basic     │    │
│  │                        │  title matching (no fuzzy search)   │    │
│  ├────────────────────────┼────────────────────────────────────┤    │
│  │  Streaming CDN         │  Reduce quality (4K → HD → SD)    │    │
│  │  (partial failure)     │  rather than stop playback         │    │
│  ├────────────────────────┼────────────────────────────────────┤    │
│  │  API Server            │  Serve stale data from edge cache  │    │
│  │  (backend region down) │  (Zuul/edge cache fallback)        │    │
│  └────────────────────────┴────────────────────────────────────┘    │
│                                                                     │
│  Result: 99.99% of the time, users don't know anything is wrong.   │
│  They might get slightly less personalized content, but they can    │
│  still watch their shows.                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Amazon's "One-Box" Degradation Testing

```
Amazon tests degradation in production:

1. Take ONE server box out of rotation
2. Inject failure for specific services on that box
3. Verify: users on that box still have a functioning experience
4. Measure: what's the impact on conversion rate?

┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Box 1   │  │ Box 2   │  │ Box 3   │  │ Box 4   │
│ Normal  │  │ Normal  │  │ TEST!   │  │ Normal  │
│         │  │         │  │ Rec svc │  │         │
│         │  │         │  │ disabled│  │         │
└─────────┘  └─────────┘  └─────────┘  └─────────┘
                              ↑
                    "Does checkout still work
                     without recommendations?"
                              
Answer: YES → Safe to degrade in real outage
Answer: NO  → Fix dependency before next incident
```

---

## Common Mistakes / Pitfalls

### 1. Not Defining Feature Priorities in Advance

```
❌ BAD: Incident happens → scramble to decide what to turn off
   "Should we disable search or recommendations?!" 
   (Panic decisions during 3 AM outage)

✅ GOOD: Feature tiers documented and agreed upon BEFORE incidents
   Runbook says: "If load > 80%, disable tier 3 features first"
   Decision is pre-made and automated
```

### 2. Degradation That Breaks Dependent Features

```
❌ BAD: Disable "user preferences" service
   → But checkout NEEDS user preferences (shipping address!)
   → Checkout breaks too!

✅ GOOD: Map feature dependencies before degradation
   → If "user preferences" is down, checkout uses CACHED address
   → Verify fallbacks exist for ALL downstream impacts
```

### 3. No UI Indication of Degraded State

```
❌ BAD: Feature silently shows stale data / empty sections
   → Users confused: "Why are there no reviews?"
   → Support tickets flood in

✅ GOOD: UI shows gentle indicator
   → "Reviews are temporarily unavailable" (small banner)
   → "Showing cached results from 2 hours ago"
   → Users understand and don't panic
```

### 4. Testing Degradation Only During Incidents

```
❌ BAD: First time you test degradation is during a real outage
   → Discover bugs in fallback code
   → Degradation makes things WORSE

✅ GOOD: Regular degradation testing (chaos engineering!)
   → Monthly "game days" — simulate failures
   → Verify fallbacks work correctly
   → Fix issues before real incidents
```

### 5. All-or-Nothing Degradation

```
❌ BAD: Everything works OR everything fails
   No middle ground — one bad service takes down entire page

✅ GOOD: Progressive degradation
   - First: serve cached data
   - Then: serve default data
   - Then: hide the section
   - Last resort: show error message for THAT feature only
```

---

## When to Use / When NOT to Use

### ✅ Use Graceful Degradation When

| Scenario | Approach |
|----------|----------|
| Non-critical feature service down | Hide section or show defaults |
| High load / traffic spike | Shed non-essential features progressively |
| Third-party API outage | Use cached data or alternatives |
| Database read replica lag | Serve slightly stale data |
| CDN cache miss spike | Serve lower-quality content |
| Region-level failure | Redirect to another region with fewer features |

### ❌ When NOT to Use

| Scenario | Why |
|----------|-----|
| Core feature failure | Fix it, don't degrade it |
| Security service down | Don't bypass auth — that's dangerous! |
| Data integrity at risk | Don't serve potentially wrong financial data |
| Compliance requirements | Regulations may require all-or-nothing |

---

## Key Takeaways

- **Classify every feature by priority** (Tier 0-3) BEFORE incidents happen. Pre-decide what to shed.
- **Degradation should be invisible** to most users — they get a slightly worse experience, not an error page.
- **Use feature flags** to toggle features instantly without deployments during incidents.
- **Automate degradation** based on health checks and load metrics — don't rely on humans at 3 AM.
- **Test your degradation regularly** — untested fallbacks often fail when needed most.
- **Core business must ALWAYS work** — users can't search, browse, or buy? That's a total outage, not degradation.
- **Communicate degraded state** to both users (subtle UI hints) and ops teams (dashboards + alerts).

---

## What's Next?

Graceful degradation helps you survive real failures. But how do you KNOW your system degrades gracefully before a real incident? You **test** it — by intentionally breaking things. That's **Chaos Engineering** — see [09-chaos-engineering.md](./09-chaos-engineering.md).
