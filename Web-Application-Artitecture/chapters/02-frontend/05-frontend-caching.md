# Chapter 2.5: Frontend Caching Strategies (Browser Cache, Service Workers, ETags)

> **Level**: ⭐⭐ Intermediate  
> **What you'll learn**: The full frontend caching stack — how browsers cache files, how ETags validate freshness, and how Service Workers enable offline-capable apps.

---

## 🧠 Real-Life Analogy: Keeping Food at Different Locations

Think of caching as storing food at different distances from you:

```
    ┌─────────────────────────────────────────────────────────────┐
    │                                                             │
    │  🏪 Store (Origin Server)                                   │
    │     Far away, has EVERYTHING, always fresh                  │
    │     → Slow to get food (200-500ms)                          │
    │                                                             │
    │  🧊 Neighborhood Fridge (CDN Cache)                         │
    │     Pre-stocked with popular items                          │
    │     → Faster (5-20ms)                                       │
    │                                                             │
    │  🏠 Home Fridge (Browser HTTP Cache)                        │
    │     Items you recently bought                               │
    │     → Very fast (1-5ms, disk read)                          │
    │                                                             │
    │  📋 Kitchen Counter (Memory Cache / Service Worker)         │
    │     Items you're using RIGHT NOW                            │
    │     → Instant (< 1ms, already in memory)                   │
    │                                                             │
    │  The closer the cache, the FASTER it is!                    │
    │  But closer caches have LESS space.                         │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘
```

---

## 📖 The Full Caching Stack — Overview

```
    Browser requests: GET /js/app.abc123.js
    
    ┌────────────────────────────────────────────────────────────────┐
    │  Layer 1: Memory Cache (in-memory, per-tab)                   │
    │  → Checked FIRST. Cleared when tab closes.                    │
    │  → Stores: images, scripts loaded in current page             │
    │  → Speed: ~0ms                                                │
    │  → Miss? ↓                                                    │
    ├────────────────────────────────────────────────────────────────┤
    │  Layer 2: Service Worker Cache (programmatic, persistent)     │
    │  → Custom JavaScript code intercepts requests                 │
    │  → Can serve cached responses even OFFLINE                    │
    │  → Speed: ~1-2ms                                              │
    │  → Miss? ↓                                                    │
    ├────────────────────────────────────────────────────────────────┤
    │  Layer 3: HTTP Disk Cache (browser's built-in cache)          │
    │  → Uses Cache-Control / Expires / ETag headers                │
    │  → Survives browser restart (stored on disk)                  │
    │  → Speed: ~1-5ms                                              │
    │  → Miss? ↓                                                    │
    ├────────────────────────────────────────────────────────────────┤
    │  Layer 4: CDN Edge Cache (covered in Chapter 2.4)             │
    │  → Nearest CDN edge server                                    │
    │  → Speed: ~5-20ms                                             │
    │  → Miss? ↓                                                    │
    ├────────────────────────────────────────────────────────────────┤
    │  Layer 5: Origin Server                                       │
    │  → Your actual server                                         │
    │  → Speed: ~100-500ms                                          │
    └────────────────────────────────────────────────────────────────┘
    
    You can see which layer served a request in DevTools:
    Network tab → "Size" column shows:
    • "(memory cache)"   — Layer 1
    • "(ServiceWorker)"  — Layer 2
    • "(disk cache)"     — Layer 3
    • File size (200 OK) — Layers 4 or 5 (fetched over network)
```

---

## 📦 Layer 1: Browser HTTP Cache (Cache-Control & Expires)

This is the browser's built-in caching mechanism. Controlled by HTTP headers.

### How Cache-Control Works

```
    FIRST REQUEST: Browser has nothing cached
    ══════════════════════════════════════════
    
    Browser                            Server
       │                                  │
       │  GET /css/main.abc123.css        │
       │ ────────────────────────────────▶│
       │                                  │
       │  200 OK                          │
       │  Cache-Control: max-age=31536000 │
       │  Content: .body { color: #333 }  │
       │◀─────────────────────────────────│
       │                                  │
       │  Browser stores in disk cache:   │
       │  Key: /css/main.abc123.css       │
       │  Value: .body { color: #333 }    │
       │  Expires: NOW + 1 year           │
       │                                  │
    
    
    SECOND REQUEST (within 1 year): Cache HIT!
    ═══════════════════════════════════════════
    
    Browser                            Server
       │                                  │
       │  GET /css/main.abc123.css        │
       │                                  │
       │  Check disk cache...             │
       │  Found! Expires in 364 days.     │
       │                                  │
       │  ✅ Serve from cache instantly!  │
       │  No network request at all!      │
       │                                  │
       │  (Server never even knows about  │
       │   this request!)                 │
```

### Cache-Control Directives — Complete Guide

```
    ┌────────────────────────┬─────────────────────────────────────────┐
    │  Directive             │  What it means                         │
    ├────────────────────────┼─────────────────────────────────────────┤
    │  max-age=3600          │  Cache for 3600 seconds (1 hour)       │
    │                        │  Browser won't even ASK server         │
    │                        │  until time expires                    │
    ├────────────────────────┼─────────────────────────────────────────┤
    │  s-maxage=60           │  CDN cache time (overrides max-age     │
    │                        │  for shared/CDN caches only)           │
    ├────────────────────────┼─────────────────────────────────────────┤
    │  public                │  ANY cache can store (CDN, browser)    │
    ├────────────────────────┼─────────────────────────────────────────┤
    │  private               │  ONLY browser cache (NOT CDN)          │
    │                        │  Use for: user-specific data           │
    ├────────────────────────┼─────────────────────────────────────────┤
    │  no-cache              │  Cache, but ALWAYS check with server   │
    │                        │  before using (revalidation)           │
    ├────────────────────────┼─────────────────────────────────────────┤
    │  no-store              │  Don't cache AT ALL. Not in memory,    │
    │                        │  not on disk. For sensitive data.      │
    ├────────────────────────┼─────────────────────────────────────────┤
    │  immutable             │  NEVER revalidate, even on reload.     │
    │                        │  Use with hashed filenames.            │
    ├────────────────────────┼─────────────────────────────────────────┤
    │  stale-while-          │  Serve stale cache while fetching      │
    │  revalidate=60         │  fresh copy in background              │
    ├────────────────────────┼─────────────────────────────────────────┤
    │  must-revalidate       │  Once expired, MUST revalidate.        │
    │                        │  Don't serve stale content.            │
    └────────────────────────┴─────────────────────────────────────────┘
    
    
    RECIPE: Recommended Cache-Control for different file types:
    ══════════════════════════════════════════════════════════════
    
    ┌──────────────────┬───────────────────────────────────────────────┐
    │  File Type       │  Cache-Control Header                        │
    ├──────────────────┼───────────────────────────────────────────────┤
    │  JS/CSS (hashed) │  public, max-age=31536000, immutable         │
    │  main.abc123.js  │  (1 year, never revalidate)                  │
    ├──────────────────┼───────────────────────────────────────────────┤
    │  Images (hashed) │  public, max-age=31536000, immutable         │
    │  logo.d4e5f6.png │  (1 year, never revalidate)                  │
    ├──────────────────┼───────────────────────────────────────────────┤
    │  HTML pages      │  no-cache                                    │
    │  index.html      │  (always check for updates)                  │
    ├──────────────────┼───────────────────────────────────────────────┤
    │  API responses   │  private, no-cache                           │
    │  /api/user       │  (never cache publicly, always revalidate)   │
    ├──────────────────┼───────────────────────────────────────────────┤
    │  Sensitive data   │  no-store                                    │
    │  /api/payment    │  (never cache anywhere)                      │
    └──────────────────┴───────────────────────────────────────────────┘
```

---

## 🏷️ Layer 2: ETags — Smart Validation

ETags let the browser ask: *"Has this file changed since I last downloaded it?"*

```
    FIRST REQUEST:
    ══════════════
    
    Browser                              Server
       │  GET /api/products               │
       │ ────────────────────────────────▶│
       │                                  │
       │  200 OK                          │
       │  ETag: "abc123xyz"               │  ← Server generates a hash
       │  Cache-Control: no-cache         │     of the response content
       │  Body: [{id:1, name:"Phone"}]    │
       │◀─────────────────────────────────│
       │                                  │
       │  Browser saves:                  │
       │  URL: /api/products              │
       │  ETag: "abc123xyz"               │
       │  Body: [{id:1, name:"Phone"}]    │
    
    
    SECOND REQUEST (data NOT changed):
    ══════════════════════════════════
    
    Browser                              Server
       │  GET /api/products               │
       │  If-None-Match: "abc123xyz"      │  ← "I have this version"
       │ ────────────────────────────────▶│
       │                                  │  Server checks: same hash?
       │  304 Not Modified                │  YES → send 304 (no body!)
       │  (no body — saves bandwidth!)    │
       │◀─────────────────────────────────│
       │                                  │
       │  Browser uses cached version ✅   │
       │                                  │
       │  Saved: Full response body NOT   │
       │  sent again! Only 304 header.    │
       │  Response: ~100 bytes vs ~50KB   │
    
    
    SECOND REQUEST (data HAS changed):
    ═════════════════════════════════
    
    Browser                              Server
       │  GET /api/products               │
       │  If-None-Match: "abc123xyz"      │
       │ ────────────────────────────────▶│
       │                                  │  Server checks: different hash!
       │  200 OK                          │  Send full new response
       │  ETag: "def456uvw"              │  ← New ETag
       │  Body: [{id:1, name:"Phone"},   │
       │         {id:2, name:"Laptop"}]  │
       │◀─────────────────────────────────│
       │                                  │
       │  Browser updates cache           │
```

### ETag vs Last-Modified

```
    ┌─────────────────┬──────────────────────┬────────────────────────┐
    │                 │  ETag                │  Last-Modified         │
    ├─────────────────┼──────────────────────┼────────────────────────┤
    │  Header sent    │  ETag: "abc123"      │  Last-Modified:        │
    │  by server      │                      │  Mon, 15 Jan 2024     │
    ├─────────────────┼──────────────────────┼────────────────────────┤
    │  Header sent    │  If-None-Match:      │  If-Modified-Since:    │
    │  by browser     │  "abc123"            │  Mon, 15 Jan 2024     │
    ├─────────────────┼──────────────────────┼────────────────────────┤
    │  Precision      │  Byte-level exact    │  Second-level only     │
    ├─────────────────┼──────────────────────┼────────────────────────┤
    │  Best for       │  API responses,      │  Static files          │
    │                 │  dynamic content     │                        │
    └─────────────────┴──────────────────────┴────────────────────────┘
```

### ETag Implementation

#### Python (Flask)
```python
from flask import Flask, request, jsonify, make_response
import hashlib
import json

app = Flask(__name__)

@app.route('/api/products')
def get_products():
    products = fetch_products_from_db()
    
    # Generate ETag from response content
    content = json.dumps(products, sort_keys=True)
    etag = hashlib.md5(content.encode()).hexdigest()
    
    # Check if client's cached version matches
    client_etag = request.headers.get('If-None-Match')
    if client_etag == f'"{etag}"':
        # Content hasn't changed — return 304 (no body!)
        return make_response('', 304)
    
    # Content changed — send full response with new ETag
    response = make_response(jsonify(products))
    response.headers['ETag'] = f'"{etag}"'
    response.headers['Cache-Control'] = 'no-cache'  # Always validate
    return response
```

#### Java (Spring Boot)
```java
@RestController
public class ProductController {

    @GetMapping("/api/products")
    public ResponseEntity<List<Product>> getProducts(
            WebRequest request) {
        
        List<Product> products = productRepo.findAll();
        
        // Generate ETag from content hash
        String content = products.toString();
        String etag = DigestUtils.md5DigestAsHex(
            content.getBytes(StandardCharsets.UTF_8)
        );
        
        // Check if client has current version
        if (request.checkNotModified(etag)) {
            // Spring automatically returns 304 Not Modified
            return null;
        }
        
        return ResponseEntity.ok()
            .eTag(etag)
            .cacheControl(CacheControl.noCache())
            .body(products);
    }
}
```

---

## 🔧 Layer 3: Service Workers — Programmable Cache

A **Service Worker** is a JavaScript file that runs **between** your browser and the network. It intercepts every request and decides: serve from cache or fetch from network?

```
    WITHOUT Service Worker:
    ══════════════════════
    Browser  ─────▶  Network  ─────▶  Server
    
    
    WITH Service Worker:
    ═══════════════════
    Browser  ─────▶  Service Worker  ─────▶  Network  ─────▶  Server
                          │
                          ├── Check cache
                          ├── Modify request
                          ├── Serve cached response
                          └── Work OFFLINE! ✨
    
    
    Service Worker Lifecycle:
    ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
    │ Register │────▶│ Install  │────▶│ Activate │────▶│ Running  │
    │          │     │(download │     │(clean old│     │(intercept│
    │          │     │ & cache  │     │  caches) │     │ requests)│
    └──────────┘     └──────────┘     └──────────┘     └──────────┘
                          │
                          ▼
                    Pre-cache critical
                    assets during install
```

### Service Worker Code Example

```javascript
// sw.js — Service Worker file

const CACHE_NAME = 'mystore-v2';
const CRITICAL_ASSETS = [
    '/',
    '/css/main.abc123.css',
    '/js/app.def456.js',
    '/images/logo.svg',
    '/offline.html'
];

// ═══════════════════════════════════════════
// INSTALL: Pre-cache critical assets
// ═══════════════════════════════════════════
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME).then((cache) => {
            console.log('Pre-caching critical assets');
            return cache.addAll(CRITICAL_ASSETS);
        })
    );
});

// ═══════════════════════════════════════════
// ACTIVATE: Clean up old caches
// ═══════════════════════════════════════════
self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then((cacheNames) => {
            return Promise.all(
                cacheNames
                    .filter(name => name !== CACHE_NAME)
                    .map(name => caches.delete(name))
            );
        })
    );
});

// ═══════════════════════════════════════════
// FETCH: Intercept every request
// ═══════════════════════════════════════════
self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request).then((cachedResponse) => {
            if (cachedResponse) {
                // ✅ Found in cache — serve immediately!
                return cachedResponse;
            }
            
            // ❌ Not in cache — fetch from network
            return fetch(event.request).then((networkResponse) => {
                // Cache the response for next time
                const responseClone = networkResponse.clone();
                caches.open(CACHE_NAME).then((cache) => {
                    cache.put(event.request, responseClone);
                });
                return networkResponse;
            }).catch(() => {
                // 🔌 Network failed (offline!) — show offline page
                return caches.match('/offline.html');
            });
        })
    );
});
```

### Service Worker Caching Strategies

```
    STRATEGY 1: Cache First (best for static assets)
    ═════════════════════════════════════════════════
    
    Request ──▶ Cache ──┬── HIT ──▶ Return cached ✅
                        │
                        └── MISS ──▶ Network ──▶ Cache + Return
    
    Best for: CSS, JS, images, fonts
    
    
    STRATEGY 2: Network First (best for API data)
    ══════════════════════════════════════════════
    
    Request ──▶ Network ──┬── SUCCESS ──▶ Cache + Return ✅
                          │
                          └── FAIL ──▶ Cache (stale fallback) ✅
    
    Best for: API calls, HTML pages, real-time data
    
    
    STRATEGY 3: Stale While Revalidate (balanced)
    ═════════════════════════════════════════════
    
    Request ──▶ Cache ──▶ Return cached IMMEDIATELY ✅
                   │
                   └──▶ Network ──▶ Update cache in background
                                     (next request gets fresh data)
    
    Best for: Avatars, social feeds, non-critical API data
    
    
    STRATEGY 4: Cache Only (offline apps)
    ═════════════════════════════════════
    
    Request ──▶ Cache ──▶ Return ✅ (never goes to network)
    
    Best for: Offline-first apps, pre-cached assets
    
    
    STRATEGY 5: Network Only (no caching)
    ═════════════════════════════════════
    
    Request ──▶ Network ──▶ Return ✅ (never touches cache)
    
    Best for: Analytics, auth tokens, payments
```

### Registering a Service Worker

```javascript
// main.js — Register service worker in your app
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js')
        .then((registration) => {
            console.log('SW registered:', registration.scope);
            
            // Check for updates every hour
            setInterval(() => {
                registration.update();
            }, 60 * 60 * 1000);
        })
        .catch((error) => {
            console.log('SW registration failed:', error);
        });
}
```

---

## 🔄 How All Layers Work Together

```
    User visits mystore.com for the SECOND TIME:
    ═════════════════════════════════════════════
    
    Browser: "I need /js/app.abc123.js"
    
    ┌──────────────────────┐
    │ 1. Memory Cache      │──── Already loaded this page? 
    │    (per-tab)         │     YES → serve from memory (0ms)
    └──────────┬───────────┘     NO ↓
               │
    ┌──────────▼───────────┐
    │ 2. Service Worker    │──── SW intercepts request
    │    Cache (CacheAPI)  │     Has it? YES → return (1ms)
    └──────────┬───────────┘     NO ↓ 
               │
    ┌──────────▼───────────┐
    │ 3. HTTP Disk Cache   │──── Check Cache-Control headers
    │    (automatic)       │     max-age valid? YES → return (3ms)
    └──────────┬───────────┘     Expired? → Send request with ETag
               │
    ┌──────────▼───────────┐
    │ 4. Network Request   │──── Goes to CDN or origin
    │    (with ETag)       │     ETag matches? → 304 (no body, 20ms)
    └──────────┬───────────┘     Changed? → 200 + full response (200ms)
               │
    ┌──────────▼───────────┐
    │ 5. Origin Server     │──── Database query, render response
    │    (if CDN miss)     │     Return new content (500ms)
    └──────────────────────┘
```

---

## 🏢 Real-World Examples

```
    ┌──────────────────┬──────────────────────────────────────────────┐
    │  Company         │  Caching Strategy                           │
    ├──────────────────┼──────────────────────────────────────────────┤
    │  Twitter/X       │  Service Worker + aggressive browser cache  │
    │                  │  for offline timeline reading               │
    ├──────────────────┼──────────────────────────────────────────────┤
    │  Google Maps     │  Service Worker caches map tiles.           │
    │                  │  Works offline for pre-loaded areas!        │
    ├──────────────────┼──────────────────────────────────────────────┤
    │  Flipkart Lite   │  Full PWA with Service Worker.              │
    │                  │  3x increase in time-on-site.               │
    │                  │  Works on 2G networks.                      │
    ├──────────────────┼──────────────────────────────────────────────┤
    │  Pinterest       │  Aggressive asset caching.                  │
    │                  │  40% reduction in wait times.               │
    │                  │  15% increase in sign-ups.                  │
    ├──────────────────┼──────────────────────────────────────────────┤
    │  Starbucks       │  Offline-capable PWA.                       │
    │                  │  Full menu browsing and ordering            │
    │                  │  works without internet.                    │
    └──────────────────┴──────────────────────────────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Caching HTML with long max-age
       → Users stuck on old version, can't see new features!
       ✅ Use no-cache for HTML, long max-age only for hashed assets
    
    ❌ Service Worker caching everything
       → Cache fills up, important assets evicted
       ✅ Be selective: only cache critical assets
    
    ❌ Not versioning Service Worker cache
       → Old SW serves stale assets forever
       ✅ Change CACHE_NAME on each deploy ('v1' → 'v2')
    
    ❌ Forgetting Cache-Control: private for user data
       → CDN caches User A's dashboard, shows to User B!
       ✅ Always use private for authenticated/personalized responses
    
    ❌ Using ETags on load-balanced servers
       → Server A and Server B generate different ETags!
       ✅ Use content-based ETags (hash of body), not inode-based
    
    ❌ Not handling Service Worker updates
       → Users stuck on old cached version forever
       ✅ Use skipWaiting() + prompt user to refresh
```

---

## ✅ When to Use Each Caching Layer

```
    ┌──────────────────────┬──────────────────────────────────────────┐
    │  Caching Layer       │  Use When                               │
    ├──────────────────────┼──────────────────────────────────────────┤
    │  Cache-Control       │  ALWAYS. Every response should have it. │
    │  (HTTP Headers)      │  It's free, automatic, and essential.   │
    ├──────────────────────┼──────────────────────────────────────────┤
    │  ETag / Last-Modified│  API responses, dynamic content where   │
    │  (Validation)        │  you want freshness + bandwidth saving  │
    ├──────────────────────┼──────────────────────────────────────────┤
    │  Service Worker      │  PWAs, offline support, push notifs,    │
    │  (Programmable)      │  background sync, fine-grained control  │
    ├──────────────────────┼──────────────────────────────────────────┤
    │  CDN Cache           │  Global audience, static assets, SSG    │
    │  (Edge)              │  pages, high-traffic sites              │
    └──────────────────────┴──────────────────────────────────────────┘
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. Frontend caching has 5 layers: Memory → Service Worker →        ║
║     HTTP disk cache → CDN → Origin. Each layer is checked in order. ║
║                                                                      ║
║  2. Cache-Control headers control browser + CDN caching behavior.   ║
║     Use max-age for freshness period, no-cache for always validate. ║
║                                                                      ║
║  3. ETags save bandwidth — server returns 304 (empty body) when     ║
║     content hasn't changed. Always use for API responses.            ║
║                                                                      ║
║  4. Service Workers enable offline-capable apps. They intercept     ║
║     requests and can serve from cache even without internet.         ║
║                                                                      ║
║  5. GOLDEN RULE: Long cache for hashed assets (CSS/JS/images),     ║
║     no-cache for HTML, no-store for sensitive data.                   ║
║                                                                      ║
║  6. Version your cache! Change cache name on deploy to avoid        ║
║     serving stale assets forever.                                    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

Now that you understand how content is rendered (SSR/CSR/SSG) and cached (CDN, browser, Service Worker), let's explore the two fundamental frontend architecture styles. In [Chapter 2.6: SPA vs MPA](./06-spa-vs-mpa.md), we'll compare Single Page Applications and Multi Page Applications.

---

[⬅️ Previous: CDN for Frontend](./04-cdn-for-frontend.md) | [⬆️ Index](../../00-INDEX.md) | [Next: SPA vs MPA ➡️](./06-spa-vs-mpa.md)
