# CDN Caching — Caching at the Edge

> **What you'll learn**: How CDNs (Content Delivery Networks) cache your content at hundreds of locations worldwide, eliminating latency by serving data from a server just milliseconds away from the user.

---

## Real-Life Analogy

Imagine you own a pizza chain headquartered in New York. If every customer — even those in Tokyo, London, and Mumbai — had to order directly from your NYC kitchen, the pizza would arrive **cold and late**.

Instead, you open **local outlets** in every major city. Each outlet keeps your most popular pizzas **pre-made and warm**. Customers get their pizza from the closest outlet — hot and fast.

A **CDN** works exactly like this. Instead of every user fetching content from your origin server (maybe in Virginia), the CDN caches copies at **edge locations** all over the world, so users get content from the nearest one.

---

## Core Concept Explained Step-by-Step

### Step 1: Without a CDN — Everyone Goes to Origin

```
User in Tokyo ──────────────[200ms]──────────────▶ Origin (Virginia)
User in London ─────────────[100ms]──────────────▶ Origin (Virginia)
User in Mumbai ─────────────[180ms]──────────────▶ Origin (Virginia)
User in New York ───────────[10ms]───────────────▶ Origin (Virginia)
```

**Problem**: Users far from the origin experience high latency. Physics can't be cheated — light takes time to cross continents.

### Step 2: With a CDN — Serve from Nearest Edge

```
User in Tokyo ──────[5ms]──▶ CDN Edge (Tokyo) ✓ CACHED
User in London ─────[3ms]──▶ CDN Edge (London) ✓ CACHED
User in Mumbai ─────[8ms]──▶ CDN Edge (Mumbai) ✓ CACHED
User in New York ───[2ms]──▶ CDN Edge (NYC) ✓ CACHED

                              On MISS, edge fetches from origin:
CDN Edge (Tokyo) ──[200ms]──▶ Origin (Virginia)
                              Then caches it for future requests
```

### Step 3: What Gets Cached on a CDN?

| Content Type | Examples | Cacheable? |
|-------------|----------|------------|
| Static assets | Images, CSS, JS, fonts | ✅ Always |
| HTML pages | Landing pages, blog posts | ✅ With care |
| API responses | Product listings, search results | ✅ If appropriate |
| Video/Audio | Streaming chunks, podcasts | ✅ Common |
| Personalized content | User dashboard, cart | ❌ Usually not |

---

## How It Works Internally

### CDN Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         CDN NETWORK                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │
│  │ Edge PoP   │  │ Edge PoP   │  │ Edge PoP   │  │ Edge PoP   │ │
│  │ Tokyo      │  │ London     │  │ Mumbai     │  │ São Paulo  │ │
│  │ Cache: 2TB │  │ Cache: 2TB │  │ Cache: 2TB │  │ Cache: 2TB │ │
│  └─────┬──────┘  └──────┬─────┘  └─────┬──────┘  └─────┬──────┘ │
│        │                 │               │               │        │
│        └─────────────────┼───────────────┼───────────────┘        │
│                          │               │                         │
│              ┌───────────┴───────────────┴──────────┐             │
│              │        Regional Shield / Mid-Tier     │             │
│              │        (Reduces origin load further)   │             │
│              └───────────────────┬───────────────────┘             │
│                                  │                                  │
│                          ┌───────┴────────┐                        │
│                          │  YOUR ORIGIN   │                        │
│                          │  SERVER        │                        │
│                          └────────────────┘                        │
└──────────────────────────────────────────────────────────────────┘
```

### The CDN Request Flow

```
User Request: GET /images/hero.jpg
       │
       ▼
┌─────────────────────────────────┐
│ DNS resolves to nearest CDN PoP │
│ (GeoDNS / Anycast routing)      │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│      CDN Edge Server            │
│                                  │
│  Is /images/hero.jpg cached?    │
│      │                           │
│   YES│       NO                  │
│      │        │                  │
│      ▼        ▼                  │
│  Return    Fetch from origin     │
│  cached    (or shield tier)      │
│  copy         │                  │
│               ▼                  │
│           Store in local cache   │
│           Return to user         │
└─────────────────────────────────┘
```

### Cache-Control Headers — How Origin Tells CDN What to Cache

The origin server controls CDN behavior using HTTP headers:

```
HTTP/1.1 200 OK
Cache-Control: public, max-age=86400, s-maxage=604800
Vary: Accept-Encoding
ETag: "abc123"
Content-Type: image/jpeg
```

| Header | Meaning |
|--------|---------|
| `Cache-Control: public` | CDN can cache this |
| `max-age=86400` | Browser caches for 1 day |
| `s-maxage=604800` | CDN caches for 7 days (overrides max-age for shared caches) |
| `Vary: Accept-Encoding` | Cache different versions for gzip vs brotli |
| `ETag: "abc123"` | Fingerprint for conditional requests |
| `Cache-Control: private` | Only browser can cache, NOT CDN |
| `Cache-Control: no-store` | Don't cache anywhere |

### Stale-While-Revalidate

A powerful pattern — serve stale content instantly while refreshing in the background:

```
Cache-Control: max-age=60, stale-while-revalidate=3600

Timeline:
0s          60s                    3660s
│───────────│──────────────────────│
│  FRESH    │  STALE BUT SERVED   │  MUST REVALIDATE
│  (serve   │  (serve stale,      │  (fetch new)
│  directly)│   refresh in bg)    │
```

---

## Code Examples

### Python — Setting Cache Headers in Flask

```python
from flask import Flask, make_response, send_file
from datetime import datetime

app = Flask(__name__)

@app.route('/api/products')
def get_products():
    """API response cached at CDN for 5 minutes."""
    products = db.get_all_products()
    
    response = make_response(jsonify(products))
    # CDN caches for 5 min, browser for 1 min
    response.headers['Cache-Control'] = 'public, max-age=60, s-maxage=300'
    response.headers['Vary'] = 'Accept-Encoding'
    return response

@app.route('/static/<path:filename>')
def serve_static(filename):
    """Static assets cached aggressively at CDN."""
    response = make_response(send_file(f'static/{filename}'))
    # CDN and browser cache for 1 year (cache-bust via filename hash)
    response.headers['Cache-Control'] = 'public, max-age=31536000, immutable'
    return response

@app.route('/api/user/profile')
def get_profile():
    """User-specific data — NOT cached at CDN."""
    user = get_current_user()
    response = make_response(jsonify(user.profile))
    response.headers['Cache-Control'] = 'private, no-store'
    return response
```

### Java — Spring Boot Cache Headers

```java
import org.springframework.http.CacheControl;
import org.springframework.http.ResponseEntity;
import java.time.Duration;

@RestController
public class ProductController {

    @GetMapping("/api/products/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable String id) {
        Product product = productService.findById(id);
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl
                .maxAge(Duration.ofMinutes(1))     // Browser: 1 min
                .sMaxAge(Duration.ofMinutes(10))   // CDN: 10 min
                .staleWhileRevalidate(Duration.ofHours(1)))
            .eTag(product.getVersion().toString())  // For conditional requests
            .body(product);
    }

    @GetMapping("/api/user/dashboard")
    public ResponseEntity<Dashboard> getDashboard() {
        // Private — CDN must NOT cache this
        return ResponseEntity.ok()
            .cacheControl(CacheControl.noStore())
            .body(dashboardService.getForCurrentUser());
    }
}
```

### CDN Configuration — CloudFront (AWS)

```json
{
  "DistributionConfig": {
    "Origins": [{
      "DomainName": "api.myapp.com",
      "CustomOriginConfig": {
        "OriginProtocolPolicy": "https-only"
      }
    }],
    "DefaultCacheBehavior": {
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
      "Compress": true,
      "TTL": {
        "DefaultTTL": 86400,
        "MaxTTL": 604800,
        "MinTTL": 0
      }
    },
    "CacheBehaviors": [
      {
        "PathPattern": "/api/*",
        "DefaultTTL": 60,
        "MaxTTL": 300
      },
      {
        "PathPattern": "/static/*",
        "DefaultTTL": 31536000,
        "Compress": true
      }
    ]
  }
}
```

---

## CDN Cache Invalidation (Purging)

When content changes, you need to tell the CDN to drop its cached copy:

```
┌──────────────────────────────────────────────────────────┐
│              CACHE INVALIDATION STRATEGIES                 │
├──────────────────────────────────────────────────────────┤
│                                                            │
│  Strategy 1: PURGE by URL                                 │
│  ─────────────────────────                                │
│  POST /purge HTTP/1.1                                     │
│  Host: cdn-api.cloudflare.com                             │
│  {"files": ["https://mysite.com/images/hero.jpg"]}        │
│                                                            │
│  Strategy 2: CACHE BUSTING (filename hashing)             │
│  ─────────────────────────────────────────                │
│  /static/app.abc123.js  → new build → /static/app.def456.js
│  (Old URL still cached, new URL fetches fresh)            │
│                                                            │
│  Strategy 3: VERSIONED URLs                               │
│  ─────────────────────────                                │
│  /api/v2/products?v=20240115                              │
│                                                            │
│  Strategy 4: SHORT TTL + stale-while-revalidate           │
│  ─────────────────────────────────────────────            │
│  Cache-Control: max-age=60, stale-while-revalidate=86400  │
│  (Content refreshes every minute, stale served if origin   │
│   is slow)                                                 │
└──────────────────────────────────────────────────────────┘
```

> **Best Practice**: Use **filename hashing** for static assets (never need to purge) and **short TTL + stale-while-revalidate** for dynamic content.

---

## Real-World Example

### How Netflix Uses CDN Caching (Open Connect)

Netflix built their **own CDN** called Open Connect, because commercial CDNs couldn't handle their scale (>30% of US internet traffic):

```
┌──────────────────────────────────────────────────────────────┐
│              NETFLIX OPEN CONNECT                              │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  User in Delhi opens Netflix                                   │
│       │                                                        │
│       ▼                                                        │
│  ┌──────────────────────┐                                     │
│  │ Netflix API (AWS)     │ ← Decides WHAT to play              │
│  │ (auth, catalog,       │   (metadata, not video)             │
│  │  recommendations)     │                                     │
│  └──────────┬───────────┘                                     │
│             │ Returns: "Stream from OCA in Delhi"              │
│             ▼                                                  │
│  ┌──────────────────────┐                                     │
│  │ Open Connect          │ ← Physical servers inside ISPs     │
│  │ Appliance (OCA)       │   Pre-loaded with popular content  │
│  │ at Jio/Airtel ISP     │   10-100+ TB SSD per appliance     │
│  │ in Delhi              │                                     │
│  │                        │                                     │
│  │ Cache hit rate: >95%   │ ← Most videos served locally      │
│  └──────────────────────┘                                     │
│                                                                │
│  If not cached locally → fetch from regional hub → origin     │
└──────────────────────────────────────────────────────────────┘
```

**Key stats**:
- **17,000+ servers** in ISP networks worldwide
- **95%+ cache hit rate** — most content served from within the ISP's own network
- User sees **<5ms latency** to the video stream
- Pre-populates popular content at night (off-peak) to avoid daytime bandwidth spikes

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Caching personalized responses | User A sees User B's data | Use `Cache-Control: private` or `Vary: Cookie` |
| No `Vary` header on compressed content | Users get wrong encoding | Add `Vary: Accept-Encoding` |
| Caching error responses (500s) | Users see errors for minutes | Don't cache 5xx, or use very short TTL |
| Too long TTL with no invalidation plan | Stale content for hours/days | Use cache-busting or short TTL |
| Caching `Set-Cookie` responses | Cookie leaks to other users! | Strip `Set-Cookie` or mark `private` |
| Not warming cache after purge | First users after purge get slow response | Use origin shield or pre-warm |

### The Cookie Caching Disaster

```
❌ DANGEROUS: CDN caches response WITH Set-Cookie
   User A requests /page → CDN caches response (includes Set-Cookie: session=abc)
   User B requests /page → CDN serves cached response → User B gets User A's session!

✅ FIX: Never cache responses with Set-Cookie headers
   - Strip Set-Cookie at CDN layer
   - Or use Cache-Control: private for any response with cookies
```

---

## When to Use / When NOT to Use

### ✅ Use CDN Caching When:
- Serving **static assets** (JS, CSS, images, fonts, videos)
- Your users are **geographically distributed**
- You need to reduce **origin server load** (traffic offloading)
- API responses are **cacheable** (same for all users, changes infrequently)
- You want DDoS protection (CDNs absorb attack traffic at edge)

### ❌ Don't Use CDN Caching When:
- Content is **highly personalized** per user (user dashboards, carts)
- Content changes **every second** (real-time stock prices, live scores)
- Data is **security-sensitive** and must not be stored at edge nodes
- Your users are all in **one location** (CDN adds complexity for no benefit)
- POST/PUT/DELETE requests (only cache GETs)

---

## Popular CDN Providers Comparison

| Provider | Edge Locations | Best For | Pricing Model |
|----------|---------------|----------|---------------|
| CloudFront (AWS) | 450+ | AWS-integrated apps | Pay per GB + requests |
| Cloudflare | 300+ | Full-stack (CDN + DNS + WAF) | Free tier available |
| Akamai | 4,000+ | Enterprise, media streaming | Contract-based |
| Fastly | 80+ | Real-time purge, edge compute | Pay per request/GB |
| Google Cloud CDN | 180+ | GCP-integrated apps | Pay per GB |

---

## Key Takeaways

1. **CDNs cache content at edge locations** worldwide — users hit the nearest server instead of crossing continents.
2. Control caching with **HTTP Cache-Control headers** — `public`, `max-age`, `s-maxage`, `stale-while-revalidate`.
3. Use **filename hashing** for static assets (e.g., `app.3fa9b2.js`) — never needs manual purging.
4. **Never cache personalized or cookie-bearing responses** at the CDN layer.
5. **Origin Shield** (mid-tier cache) reduces origin hits further by consolidating edge misses.
6. Netflix, YouTube, and other streaming services run their **own CDN** infrastructure inside ISPs for ultimate performance.
7. CDN caching can offload **80-95% of traffic** from your origin servers — huge cost and performance win.

---

## What's Next?

Next, we'll explore **Database Query Caching** — how to cache the results of expensive database queries so your database doesn't get hammered with the same slow query thousands of times per second.

→ [05-database-query-caching.md](./05-database-query-caching.md)
