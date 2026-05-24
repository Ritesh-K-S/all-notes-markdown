# Chapter 2.8: Edge Computing for Frontend — Running UI Logic at the Edge

> **Level**: ⭐⭐⭐⭐ Expert  
> **What you'll learn**: How to run frontend logic (not just serve files) on servers closest to users — enabling personalization, A/B testing, geo-routing, and auth at the edge with near-zero latency.

---

## 🧠 Real-Life Analogy: Smart Vending Machines vs Central Factory

A **CDN** is like a vending machine — it holds pre-made items and dispenses them quickly. But it can't customize anything.

**Edge computing** is like a vending machine with a built-in kitchen — it can take your order, customize it (add toppings, adjust size), and serve a personalized product **right there**, without calling the central factory.

```
    CDN (Static Serving):
    ═════════════════════
    
    User ──▶ Edge Server ──▶ Returns SAME pre-cached file
                              (no logic, no customization)
    
    
    EDGE COMPUTING (Smart Serving):
    ═══════════════════════════════
    
    User ──▶ Edge Server ──▶ RUNS CODE at the edge!
                              │
                              ├── Check user's country → serve localized page
                              ├── Check A/B test group → serve variant A or B
                              ├── Check auth cookie → redirect to login
                              ├── Rewrite HTML → inject personalized content
                              ├── Resize images on the fly
                              └── Return CUSTOMIZED response
    
    All this happens at the edge — 5-20ms from the user!
    No round trip to origin server needed.
```

---

## 📖 What is Edge Computing?

```
    TRADITIONAL: All logic runs on your origin server
    ═════════════════════════════════════════════════
    
    User (Mumbai) ──── 200ms ────▶ Origin (US-East) ──── Process ──── 200ms ────▶ User
                                                         logic
                                  Total: ~450ms round trip
    
    
    EDGE COMPUTING: Logic runs on servers NEAR the user
    ═══════════════════════════════════════════════════
    
    User (Mumbai) ──── 5ms ────▶ Edge (Mumbai) ──── Process ──── 5ms ────▶ User
                                                    logic
                                 Total: ~15ms round trip!
    
    
    WHERE THE "EDGE" IS:
    ════════════════════
    
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  Your Origin Server (1 location)                             │
    │           │                                                  │
    │           │ Traditional cloud: 3-5 regions                   │
    │           │                                                  │
    │  CDN Edge Servers (100-300 cities)                           │
    │           │                                                  │
    │           │ Edge Computing: same 100-300 locations            │
    │           │ but now they can RUN CODE, not just serve files!  │
    │           │                                                  │
    │  Users (everywhere)                                          │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
    
    
    Edge Computing Platforms:
    ┌────────────────────┬──────────────────────────────────────────┐
    │  Platform          │  Details                                │
    ├────────────────────┼──────────────────────────────────────────┤
    │  Cloudflare Workers│  310+ cities, V8 isolates, < 5ms cold   │
    │                    │  start. Most popular.                    │
    ├────────────────────┼──────────────────────────────────────────┤
    │  Vercel Edge       │  Runs Next.js middleware at the edge.   │
    │  Functions         │  Built on Cloudflare Workers.           │
    ├────────────────────┼──────────────────────────────────────────┤
    │  AWS CloudFront    │  Lambda@Edge (Node.js, Python at 30+   │
    │  Functions /       │  regions) or CloudFront Functions       │
    │  Lambda@Edge       │  (lighter, 200+ locations).             │
    ├────────────────────┼──────────────────────────────────────────┤
    │  Deno Deploy       │  35+ regions, built on Deno runtime.    │
    ├────────────────────┼──────────────────────────────────────────┤
    │  Fastly Compute    │  WebAssembly-based, ultra-fast.         │
    └────────────────────┴──────────────────────────────────────────┘
```

---

## 🔧 Key Use Cases for Edge Computing in Frontend

### Use Case 1: A/B Testing at the Edge

```
    WITHOUT EDGE (traditional A/B testing):
    ════════════════════════════════════════
    
    Browser loads page → JavaScript runs → Checks experiment →
    Shows variant → FLASH! (user sees original first, then switch)
    
    This is called "Flash of Original Content" (FOOC) — bad UX!
    
    
    WITH EDGE A/B testing:
    ═════════════════════
    
    ┌────────┐     ┌──────────────────────┐     ┌────────────────┐
    │  User  │────▶│  Edge Function       │────▶│  User sees     │
    │ Request│     │                      │     │  Variant B     │
    │        │     │  1. Read A/B cookie  │     │  immediately!  │
    │        │     │  2. No cookie? Assign│     │  No flash! ✅  │
    │        │     │     to group B (50%) │     │                │
    │        │     │  3. Rewrite HTML:    │     │                │
    │        │     │     swap hero image  │     │                │
    │        │     │     change CTA button│     │                │
    │        │     │  4. Set cookie for   │     │                │
    │        │     │     sticky sessions  │     │                │
    └────────┘     └──────────────────────┘     └────────────────┘
    
    The user NEVER sees the original — edge serves the right
    variant from the very first byte!
```

#### Cloudflare Worker — A/B Testing

```javascript
// A/B test at the edge — no origin server needed!
export default {
    async fetch(request) {
        const url = new URL(request.url);
        
        // Only A/B test the homepage
        if (url.pathname !== '/') {
            return fetch(request);
        }
        
        // Check if user already has a test group
        const cookie = request.headers.get('Cookie') || '';
        let variant = getCookieValue(cookie, 'ab_test');
        
        if (!variant) {
            // Assign randomly: 50/50 split
            variant = Math.random() < 0.5 ? 'A' : 'B';
        }
        
        // Fetch the page from origin
        let response = await fetch(request);
        let html = await response.text();
        
        // Modify HTML based on variant
        if (variant === 'B') {
            html = html.replace(
                '<h1>Welcome to MyStore</h1>',
                '<h1>Shop the Best Deals Today!</h1>'
            );
            html = html.replace(
                'btn-primary',
                'btn-danger'  // Red CTA button
            );
        }
        
        // Return modified page with sticky cookie
        const newResponse = new Response(html, response);
        newResponse.headers.set('Set-Cookie', 
            `ab_test=${variant}; Path=/; Max-Age=86400`
        );
        return newResponse;
    }
};
```

### Use Case 2: Geo-Based Content at the Edge

```
    Edge function has access to user's location
    (from the connecting IP / CDN headers):
    
    ┌────────┐     ┌──────────────────────┐     ┌───────────────────┐
    │  User  │────▶│  Edge Function       │────▶│  Response         │
    │  from  │     │                      │     │                   │
    │  India │     │  Country: IN         │     │  Currency: ₹      │
    │        │     │  City: Mumbai        │     │  Language: Hindi   │
    │        │     │  → Set locale        │     │  Store: mystore.in│
    │        │     │  → Redirect to .in   │     │  Shipping: India  │
    │        │     │  → Show ₹ prices     │     │                   │
    └────────┘     └──────────────────────┘     └───────────────────┘
```

```javascript
// Geo-routing at the edge
export default {
    async fetch(request) {
        // Cloudflare provides geo info automatically
        const country = request.cf?.country || 'US';
        const city = request.cf?.city || '';
        
        // Redirect to country-specific domain
        const countryDomains = {
            'IN': 'https://www.mystore.in',
            'UK': 'https://www.mystore.co.uk',
            'DE': 'https://www.mystore.de',
            'JP': 'https://www.mystore.jp',
        };
        
        const targetDomain = countryDomains[country];
        if (targetDomain) {
            const url = new URL(request.url);
            return Response.redirect(
                targetDomain + url.pathname, 302
            );
        }
        
        // Default: serve US site
        return fetch(request);
    }
};
```

### Use Case 3: Authentication at the Edge

```
    Validate auth tokens at the edge — block unauthorized 
    requests BEFORE they reach your origin server:
    
    ┌────────┐     ┌──────────────────────┐
    │ Legit  │────▶│  Edge Function       │──── Token valid ────▶ Origin ✅
    │ User   │     │                      │
    └────────┘     │  Verify JWT token    │
                   │  at the edge         │
    ┌────────┐     │                      │
    │ Hacker │────▶│  Invalid/no token    │──── 401 Unauthorized ❌
    │        │     │  → BLOCKED at edge!  │     (origin never touched!)
    └────────┘     └──────────────────────┘
    
    Benefits:
    • Origin server never sees unauthorized requests
    • Reduces origin load by 30-50% (blocks bots, scrapers)
    • Auth check in 5ms instead of 200ms
```

```javascript
// Edge auth middleware
export default {
    async fetch(request) {
        const url = new URL(request.url);
        
        // Public paths — no auth needed
        const publicPaths = ['/', '/login', '/static/', '/api/health'];
        if (publicPaths.some(p => url.pathname.startsWith(p))) {
            return fetch(request);
        }
        
        // Check for auth token
        const token = request.headers.get('Authorization')?.replace('Bearer ', '');
        if (!token) {
            return new Response('Unauthorized', { status: 401 });
        }
        
        // Verify JWT at the edge (using edge KV or crypto API)
        try {
            const payload = await verifyJWT(token);
            // Add user info to request headers for origin
            const modifiedRequest = new Request(request, {
                headers: {
                    ...Object.fromEntries(request.headers),
                    'X-User-Id': payload.userId,
                    'X-User-Role': payload.role,
                },
            });
            return fetch(modifiedRequest);
        } catch (err) {
            return new Response('Invalid token', { status: 403 });
        }
    }
};
```

### Use Case 4: HTML Streaming / Personalization

```
    Inject personalized content into cached HTML at the edge:
    
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  Cached HTML (same for everyone):                            │
    │  ┌──────────────────────────────────────────────────────┐    │
    │  │  <html>                                              │    │
    │  │    <nav>                                              │    │
    │  │      <span id="greeting"><!-- PERSONALIZE --></span> │    │
    │  │    </nav>                                             │    │
    │  │    <main>                                             │    │
    │  │      <!-- Static product page content -->             │    │
    │  │    </main>                                            │    │
    │  │  </html>                                              │    │
    │  └──────────────────────────────────────────────────────┘    │
    │                                                              │
    │  Edge Function:                                              │
    │  1. Serve cached HTML                                        │
    │  2. Replace <!-- PERSONALIZE --> with "Hello, Ritesh!"       │
    │  3. User gets personalized page from cached content!         │
    │                                                              │
    │  This combines CDN speed with personalization.               │
    │  SSG + Edge Personalization = best of both worlds!           │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## ⚙️ How Edge Computing Works Internally

```
    Traditional Servers vs Edge Functions:
    ══════════════════════════════════════
    
    TRADITIONAL SERVER:
    ┌───────────────────────────────────────┐
    │  Full OS (Linux)                      │
    │  ┌─────────────────────────────────┐  │
    │  │  Node.js / Python runtime       │  │
    │  │  ┌───────────────────────────┐  │  │
    │  │  │  Your Application Code    │  │  │
    │  │  └───────────────────────────┘  │  │
    │  └─────────────────────────────────┘  │
    │  Full file system, network access     │
    │  Cold start: 500ms - 5 seconds        │
    │  Memory: 128MB - 10GB                 │
    └───────────────────────────────────────┘
    
    
    EDGE FUNCTION (V8 Isolate):
    ┌───────────────────────────────────────┐
    │  Shared V8 Engine (Chrome's JS engine)│
    │  ┌─────────────────────┐              │
    │  │  Isolate A (Your    │  Isolated!   │
    │  │  code runs here)    │  Can't access│
    │  └─────────────────────┘  other code  │
    │  ┌─────────────────────┐              │
    │  │  Isolate B (Another │              │
    │  │  customer's code)   │              │
    │  └─────────────────────┘              │
    │                                       │
    │  No OS overhead, no container startup  │
    │  Cold start: < 5ms (!!!)              │
    │  Memory: 128MB max                    │
    │  Execution: < 50ms (CPU time)         │
    │  No file system, limited APIs         │
    └───────────────────────────────────────┘
    
    
    WHY SO FAST?
    V8 Isolates share the same process.
    No boot, no container, no OS startup.
    Just: load code → execute → return.
    
    Comparison:
    ┌─────────────────┬────────────┬─────────────┬───────────────┐
    │                 │  VM (EC2)  │  Container  │  V8 Isolate   │
    │                 │            │  (Lambda)   │  (CF Worker)  │
    ├─────────────────┼────────────┼─────────────┼───────────────┤
    │  Cold start     │  30-60s    │  100-500ms  │  < 5ms        │
    │  Memory         │  GBs      │  128MB-10GB │  128MB        │
    │  Isolation      │  Full OS  │  Container  │  V8 process   │
    │  Locations      │  3-20     │  20-30      │  200-300      │
    │  Cost per req   │  $$$      │  $$         │  $            │
    └─────────────────┴────────────┴─────────────┴───────────────┘
```

---

## 📊 Edge vs Serverless vs Traditional — When to Use What

```
    ┌──────────────────┬──────────────────┬──────────────────┬──────────────────┐
    │  Aspect          │  Edge Function   │  Serverless      │  Traditional     │
    │                  │  (CF Worker)     │  (AWS Lambda)    │  (EC2/Container) │
    ├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
    │  Best for        │  Request routing │  API backends    │  Complex apps    │
    │                  │  Auth, A/B tests │  Data processing │  Databases       │
    │                  │  Geo-redirect    │  Webhooks        │  Long processes  │
    │                  │  HTML rewriting  │  CRON jobs       │  Stateful logic  │
    ├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
    │  Latency         │  5-20ms ⚡       │  50-200ms        │  100-500ms       │
    ├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
    │  Max execution   │  10-30ms CPU     │  15 min          │  Unlimited       │
    ├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
    │  Database access │  Limited (KV,    │  Full (SQL,      │  Full            │
    │                  │  Durable Objects)│  NoSQL, etc.)    │                  │
    ├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
    │  Complexity      │  Simple          │  Medium          │  Full control    │
    └──────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

## 💻 Next.js Middleware — Edge Computing Made Easy

```
    Next.js runs "middleware" at the edge automatically:
    
    Browser ──▶ Edge (middleware runs HERE) ──▶ Next.js Server
                     │
                     ├── Rewrite URL
                     ├── Redirect
                     ├── Set headers
                     ├── Check auth
                     ├── A/B test
                     └── Geo-route
```

```javascript
// middleware.ts (Next.js — runs at the edge!)
import { NextResponse } from 'next/server';

export function middleware(request) {
    const { pathname } = request.nextUrl;
    const country = request.geo?.country || 'US';
    
    // 1. Geo-redirect
    if (country === 'IN' && !pathname.startsWith('/in')) {
        return NextResponse.redirect(new URL(`/in${pathname}`, request.url));
    }
    
    // 2. Auth check for protected routes
    if (pathname.startsWith('/dashboard')) {
        const token = request.cookies.get('auth_token');
        if (!token) {
            return NextResponse.redirect(new URL('/login', request.url));
        }
    }
    
    // 3. A/B testing
    if (pathname === '/pricing') {
        const bucket = request.cookies.get('pricing_test')?.value;
        if (bucket === 'B') {
            return NextResponse.rewrite(new URL('/pricing-v2', request.url));
        }
    }
    
    return NextResponse.next();
}

// Only run middleware for specific routes
export const config = {
    matcher: ['/', '/dashboard/:path*', '/pricing', '/in/:path*'],
};
```

---

## 🏢 Real-World Examples

```
    ┌──────────────────┬──────────────────────────────────────────────┐
    │  Company         │  Edge Computing Use Case                    │
    ├──────────────────┼──────────────────────────────────────────────┤
    │  Shopify         │  Storefront rendering at the edge via       │
    │                  │  Cloudflare Workers. Each store's theme     │
    │                  │  rendered in < 50ms at the nearest edge.    │
    ├──────────────────┼──────────────────────────────────────────────┤
    │  Discord         │  Edge auth validation. Verifies tokens at   │
    │                  │  the edge, blocks unauthorized WebSocket    │
    │                  │  connections before they reach servers.     │
    ├──────────────────┼──────────────────────────────────────────────┤
    │  The New York    │  Paywall enforcement at the edge. Checks   │
    │  Times           │  subscription status at edge, serves       │
    │                  │  cached articles to subscribers instantly.  │
    ├──────────────────┼──────────────────────────────────────────────┤
    │  Vercel          │  Next.js Middleware runs at edge by default.│
    │                  │  Auth checks, redirects, and rewrites      │
    │                  │  happen in < 10ms globally.                │
    ├──────────────────┼──────────────────────────────────────────────┤
    │  Canva           │  Image transformations at the edge.         │
    │                  │  Resizing, format conversion (WebP/AVIF)   │
    │                  │  done at the nearest edge location.        │
    └──────────────────┴──────────────────────────────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Running heavy computation at the edge
       → Edge functions have 10-50ms CPU limits!
       ✅ Keep edge logic lightweight: routing, auth, rewrites

    ❌ Trying to access a database from the edge
       → Traditional databases have high latency from edge locations
       ✅ Use edge-native storage: KV stores, Durable Objects, Turso

    ❌ Deploying entire application as edge functions
       → Edge has limited APIs, no file system, memory constraints
       ✅ Use edge for the "front door" (routing, auth, personalization)
         and traditional servers for heavy backend logic

    ❌ Not considering cold starts on some platforms
       → Lambda@Edge cold starts can be 500ms+
       ✅ Use V8 Isolate-based platforms (Cloudflare Workers) for
         < 5ms cold starts

    ❌ Ignoring edge caching with edge compute
       → Edge function runs on every request unnecessarily
       ✅ Cache edge function responses with Cache API
```

---

## ✅ When to Use / When NOT to Use Edge Computing

```
    USE Edge Computing:                    DON'T USE Edge Computing:
    ═══════════════════                    ════════════════════════
    ✅ Auth/session validation             ❌ Heavy computation
    ✅ A/B testing / feature flags         ❌ Complex database queries
    ✅ Geo-based routing/content           ❌ File processing
    ✅ Bot detection / rate limiting       ❌ Long-running tasks
    ✅ URL rewrites / redirects            ❌ Stateful applications
    ✅ HTML personalization                ❌ Apps needing > 128MB memory
    ✅ API response caching/transformation ❌ WebSocket servers
    ✅ Image optimization on-the-fly
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. Edge computing = running CODE (not just serving files) on       ║
║     servers closest to users. Response in 5-20ms vs 200-500ms.      ║
║                                                                      ║
║  2. Best use cases: A/B testing, geo-routing, auth validation,      ║
║     HTML personalization, bot detection, URL rewrites.               ║
║                                                                      ║
║  3. V8 Isolates (Cloudflare Workers) have < 5ms cold starts —      ║
║     dramatically faster than containers or VMs.                      ║
║                                                                      ║
║  4. Edge functions are lightweight by design. Keep logic simple —   ║
║     heavy processing belongs on traditional servers.                 ║
║                                                                      ║
║  5. Combine edge computing with CDN caching for maximum speed:      ║
║     SSG pages + edge personalization = CDN speed + custom content.   ║
║                                                                      ║
║  6. Next.js Middleware makes edge computing accessible —            ║
║     write middleware that automatically runs at the edge globally.    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

We've covered the entire frontend layer — from how browsers render pages to cutting-edge computing at the edge. Now it's time to go to the other side: **the backend**. In [Part 3: Backend & Server Logic](../03-backend/01-monolith-architecture.md), we'll start with the most fundamental architecture pattern — the monolith — and build up from there.

---

[⬅️ Previous: Micro-Frontends](./07-micro-frontends.md) | [⬆️ Index](../../00-INDEX.md) | [Next: Part 3 — Backend Architecture ➡️](../03-backend/01-monolith-architecture.md)
