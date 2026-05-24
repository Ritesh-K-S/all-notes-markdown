# Chapter 2.3: Static Site Generation (SSG) & Incremental Static Regeneration (ISR)

> **Level**: ⭐⭐ Intermediate  
> **What you'll learn**: How to pre-build HTML pages at deploy time for maximum speed, and how ISR lets you update static pages without rebuilding the entire site.

---

## 🧠 Real-Life Analogy: Printing a Newspaper

**SSR** = A live news channel — generates content on-the-fly for every viewer  
**SSG** = A printed newspaper — prepared once, same copy for everyone  
**ISR** = A newspaper with a live sports ticker — mostly printed, but the scores update automatically

```
    SSR (every request):     SSG (build time):        ISR (best of both):
    ════════════════════     ═══════════════════      ══════════════════════
    
    Request → Server         Build Once               Build Once
    renders → Response       ┌─────────────┐          ┌─────────────┐
                             │ HTML files  │          │ HTML files  │
    Request → Server         │ ready to    │          │ ready to    │
    renders → Response       │ serve       │          │ serve       │
                             └──────┬──────┘          └──────┬──────┘
    Request → Server                │                        │
    renders → Response       All requests                Requests served
                             served from                from cache, but
    (Server works            pre-built files            pages regenerate
     on EVERY request)       (CDN, no server!)          in background
                                                         after X seconds
```

---

## 📖 Static Site Generation (SSG) — Deep Dive

With SSG, HTML pages are generated **at build time** (when you deploy), NOT when a user visits. The result is plain HTML files that can be served from a CDN — no server needed at runtime.

### How SSG Works — Step by Step

```
    BUILD TIME (happens once, before deployment):
    ══════════════════════════════════════════════
    
    ┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐
    │  Source Code  │     │  Database /  │     │  CMS / APIs /        │
    │  (Templates)  │     │  Data Files  │     │  External Sources    │
    └──────┬───────┘     └──────┬───────┘     └──────────┬───────────┘
           │                    │                         │
           └────────────────────┼─────────────────────────┘
                                │
                                ▼
                    ┌──────────────────────┐
                    │   BUILD PROCESS      │
                    │   (Next.js, Gatsby,  │
                    │    Hugo, Jekyll)     │
                    │                      │
                    │   For EACH page:     │
                    │   1. Fetch data      │
                    │   2. Apply template  │
                    │   3. Generate HTML   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   OUTPUT: Static     │
                    │   HTML Files         │
                    │                      │
                    │   /index.html        │
                    │   /about.html        │
                    │   /products/1.html   │
                    │   /products/2.html   │
                    │   /products/3.html   │
                    │   /blog/post-1.html  │
                    │   /blog/post-2.html  │
                    │   ... (one file per  │
                    │       page)          │
                    └──────────┬───────────┘
                               │
                               ▼ Deploy to CDN
    
    
    RUNTIME (when users visit):
    ═══════════════════════════
    
    ┌──────────┐   GET /products/1     ┌───────────┐
    │ Browser  │ ─────────────────────▶│    CDN    │
    │          │                       │           │
    │          │   products/1.html     │ Serves    │
    │ Display! │ ◀─────────────────────│ pre-built │
    │    ✅    │   (already built!)    │ HTML file │
    └──────────┘                       └───────────┘
    
    No server rendering! No database queries!
    Just serving a static file — BLAZING FAST! ⚡⚡⚡
```

### SSG Code Examples

#### Next.js (React) — SSG with `getStaticProps`
```javascript
// pages/products/[id].js

// 1. Tell Next.js WHICH pages to generate at build time
export async function getStaticPaths() {
    const res = await fetch('https://api.mystore.com/products');
    const products = await res.json();
    
    // Generate a page for each product
    const paths = products.map(p => ({
        params: { id: p.id.toString() }
    }));
    
    return { paths, fallback: false };
    //                ^^^^^^^^^^^^^^^^
    //  false = only these pages exist, 404 for anything else
}

// 2. Fetch data at BUILD TIME for each page
export async function getStaticProps({ params }) {
    const res = await fetch(`https://api.mystore.com/products/${params.id}`);
    const product = await res.json();
    
    return {
        props: { product }  // Passed to the component below
    };
}

// 3. The page component (renders at build time)
export default function ProductPage({ product }) {
    return (
        <div>
            <h1>{product.name}</h1>
            <p className="price">₹{product.price}</p>
            <p>{product.description}</p>
        </div>
    );
}
```

#### Python (Pelican / Custom SSG)
```python
"""
Simple custom SSG — generates HTML files from data
"""
import json
import os
from jinja2 import Template

# Template for product pages
PRODUCT_TEMPLATE = Template('''
<html>
<head><title>{{ product.name }} - MyStore</title></head>
<body>
    <h1>{{ product.name }}</h1>
    <p class="price">₹{{ product.price }}</p>
    <p>{{ product.description }}</p>
    <img src="/images/{{ product.image }}">
</body>
</html>
''')

def build_site():
    """Run at BUILD TIME — generates all HTML files"""
    # 1. Load data (from DB, API, or JSON files)
    with open('data/products.json') as f:
        products = json.load(f)
    
    # 2. Generate HTML for each product
    os.makedirs('dist/products', exist_ok=True)
    for product in products:
        html = PRODUCT_TEMPLATE.render(product=product)
        filepath = f"dist/products/{product['id']}.html"
        with open(filepath, 'w') as f:
            f.write(html)
        print(f"Generated: {filepath}")
    
    print(f"Built {len(products)} product pages!")

if __name__ == '__main__':
    build_site()  # Run once during deployment
```

---

## ⚠️ The Problem with SSG — Stale Data

```
    SSG's Achilles Heel: DATA GOES STALE
    ═════════════════════════════════════
    
    Build time:    Product price = ₹5,999
    2 hours later: Admin changes price to ₹4,999
    
    What users see: Still ₹5,999! (the old built page)
    
    To show the new price, you must REBUILD the entire site!
    
    ┌──────────────────────────────────────────────────────┐
    │  For a blog with 50 pages:    Rebuild = ~30 seconds  │
    │  For an e-commerce with 10K:  Rebuild = ~10 minutes  │
    │  For Amazon with 350M items:  Rebuild = IMPOSSIBLE!  │
    └──────────────────────────────────────────────────────┘
    
    This is where ISR comes in...
```

---

## 📖 Incremental Static Regeneration (ISR) — Deep Dive

**ISR** solves the stale data problem by allowing individual pages to **regenerate in the background** after a specified time period — without rebuilding the entire site.

### How ISR Works — Step by Step

```
    ┌──────────────────────────────────────────────────────────────┐
    │  ISR with revalidate: 60 (regenerate after 60 seconds)      │
    │                                                              │
    │  BUILD TIME:                                                 │
    │  Product #42 built with price ₹5,999                        │
    │  Page cached.                                                │
    │                                                              │
    │  ─── TIME PASSES ───                                        │
    │                                                              │
    │  t=0s   Request for /products/42                             │
    │         → Serve cached page (₹5,999) ✅                     │
    │         → Page is fresh (within 60s window)                  │
    │                                                              │
    │  t=30s  Request for /products/42                             │
    │         → Serve cached page (₹5,999) ✅                     │
    │         → Still fresh                                        │
    │                                                              │
    │  t=61s  Request for /products/42                             │
    │         → Serve cached page (₹5,999) ✅ (stale but served!) │
    │         → Page is STALE (>60s old)                           │
    │         → Trigger background regeneration                    │
    │                                                              │
    │         BACKGROUND:                                          │
    │         ┌────────────────────────────────┐                   │
    │         │  Server fetches latest data     │                   │
    │         │  Price is now ₹4,999            │                   │
    │         │  Generates NEW HTML             │                   │
    │         │  Replaces old cached page       │                   │
    │         └────────────────────────────────┘                   │
    │                                                              │
    │  t=62s  Next request for /products/42                        │
    │         → Serve NEW page (₹4,999) ✅                        │
    │         → Fresh for another 60 seconds                       │
    └──────────────────────────────────────────────────────────────┘
    
    Key insight: The USER who triggered regeneration still sees 
    the OLD page. The NEXT user sees the updated page.
    This is called "stale-while-revalidate" strategy.
```

### ISR Visual Timeline

```
    Time ────────────────────────────────────────────────────────▶
    
    Build     t=0    t=30s   t=60s   t=61s        t=62s
      │        │       │       │       │             │
      ▼        ▼       ▼       ▼       ▼             ▼
    ┌─────┐  ┌─────┐ ┌─────┐ ┌─────┐ ┌──────────┐ ┌─────┐
    │Build│  │Serve│ │Serve│ │Fresh│ │Serve STALE│ │Serve│
    │page │  │cache│ │cache│ │ends │ │+ regen in │ │NEW  │
    │     │  │  ✅ │ │  ✅ │ │     │ │background │ │page │
    │₹5999│  │₹5999│ │₹5999│ │     │ │₹5999 → ₹4999│₹4999│
    └─────┘  └─────┘ └─────┘ └─────┘ └──────────┘ └─────┘
    
    ◀──── Cache is FRESH ────▶◀─ STALE ─▶◀── FRESH again ──▶
```

### ISR Code Example (Next.js)

```javascript
// pages/products/[id].js — ISR version

export async function getStaticPaths() {
    // Pre-build the most popular products
    const res = await fetch('https://api.mystore.com/products?popular=true');
    const products = await res.json();
    
    const paths = products.map(p => ({
        params: { id: p.id.toString() }
    }));
    
    return {
        paths,
        fallback: 'blocking'  // Pages not pre-built are generated on first request
        //         ^^^^^^^^^
        // 'blocking' = first visitor waits for generation (SSR-like)
        // true = show loading state, then swap in content
    };
}

export async function getStaticProps({ params }) {
    const res = await fetch(`https://api.mystore.com/products/${params.id}`);
    const product = await res.json();
    
    return {
        props: { product },
        revalidate: 60  // ← THE MAGIC! Regenerate every 60 seconds
        //          ^^
        // After 60 seconds, the next request triggers
        // background regeneration
    };
}

export default function ProductPage({ product }) {
    return (
        <div>
            <h1>{product.name}</h1>
            <p className="price">₹{product.price.toLocaleString()}</p>
            <p>{product.description}</p>
        </div>
    );
}
```

### On-Demand Revalidation (Next.js 12.1+)

```
    Instead of waiting for the timer, you can trigger regeneration
    immediately when data changes:

    Admin updates product price
           │
           ▼
    ┌──────────────────┐    POST /api/revalidate     ┌─────────────┐
    │  Admin Dashboard │ ───── ?path=/products/42 ──▶ │  Next.js    │
    │                  │                              │  Server     │
    └──────────────────┘                              │             │
                                                      │  Immediately│
                                                      │  rebuilds   │
                                                      │  /products/ │
                                                      │  42.html    │
                                                      └─────────────┘
    
    Next visitor gets the updated page instantly!
```

```javascript
// pages/api/revalidate.js — On-demand ISR
export default async function handler(req, res) {
    // Verify the request is from your CMS/admin
    if (req.query.secret !== process.env.REVALIDATION_SECRET) {
        return res.status(401).json({ message: 'Invalid token' });
    }
    
    const path = req.query.path;  // e.g., "/products/42"
    
    await res.revalidate(path);  // Regenerate that specific page NOW
    
    return res.json({ revalidated: true, path });
}
```

---

## 📊 SSR vs SSG vs ISR vs CSR — Complete Comparison

```
    ┌──────────────┬──────────┬──────────┬──────────┬──────────┐
    │  Aspect      │   SSR    │   SSG    │   ISR    │   CSR    │
    ├──────────────┼──────────┼──────────┼──────────┼──────────┤
    │ When HTML    │ Every    │ Build    │ Build +  │ In the   │
    │ is generated │ request  │ time     │ on demand│ browser  │
    ├──────────────┼──────────┼──────────┼──────────┼──────────┤
    │ Speed        │ 🟡 Good  │ ⚡ Best  │ ⚡ Best  │ 🐌 Slow  │
    │ (initial)    │          │          │ (cached) │ (initial)│
    ├──────────────┼──────────┼──────────┼──────────┼──────────┤
    │ Data         │ ✅ Always│ ❌ Stale │ 🟡 Near  │ ✅ Always│
    │ freshness    │ fresh    │ until    │ real-time│ fresh    │
    │              │          │ rebuild  │          │          │
    ├──────────────┼──────────┼──────────┼──────────┼──────────┤
    │ SEO          │ ✅ Great │ ✅ Great │ ✅ Great │ ❌ Poor  │
    ├──────────────┼──────────┼──────────┼──────────┼──────────┤
    │ Server cost  │ 💰 High  │ 💚 Free/ │ 💚 Low   │ 💚 Free/ │
    │              │          │ minimal  │          │ minimal  │
    ├──────────────┼──────────┼──────────┼──────────┼──────────┤
    │ Build time   │ N/A      │ 🔴 Can   │ 💚 Fast  │ 💚 Fast  │
    │              │          │ be slow  │ (partial)│          │
    ├──────────────┼──────────┼──────────┼──────────┼──────────┤
    │ Best for     │ Dynamic  │ Blogs,   │ E-comm,  │ Dash-    │
    │              │ pages    │ docs,    │ news,    │ boards,  │
    │              │ w/ user  │ landing  │ large    │ apps     │
    │              │ data     │ pages    │ sites    │ behind   │
    │              │          │          │          │ login    │
    └──────────────┴──────────┴──────────┴──────────┴──────────┘
```

---

## 🏢 Real-World Examples

```
    ┌──────────────────┬──────────────┬──────────────────────────────┐
    │  Company         │  Approach    │  Why                         │
    ├──────────────────┼──────────────┼──────────────────────────────┤
    │  Vercel Blog     │  SSG         │  Content rarely changes,     │
    │                  │              │  perfect for static pages    │
    ├──────────────────┼──────────────┼──────────────────────────────┤
    │  GitHub Docs     │  SSG         │  Documentation pages,        │
    │                  │              │  rebuilt on git push         │
    ├──────────────────┼──────────────┼──────────────────────────────┤
    │  Hulu            │  ISR         │  Show catalog changes daily, │
    │                  │  (Next.js)   │  ISR keeps pages fresh       │
    ├──────────────────┼──────────────┼──────────────────────────────┤
    │  Notion          │  ISR +       │  Public pages = ISR for      │
    │                  │  On-demand   │  speed, revalidate on edit   │
    ├──────────────────┼──────────────┼──────────────────────────────┤
    │  Target.com      │  ISR         │  350K+ product pages,        │
    │                  │  (Next.js)   │  can't rebuild all at once   │
    └──────────────────┴──────────────┴──────────────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Using SSG for pages with user-specific data (cart, profile)
       → SSG pages are the SAME for everyone. Use CSR or SSR for
         personalized content.
    
    ❌ Setting revalidate too low (e.g., 1 second)
       → Basically becomes SSR with extra complexity.
         Use SSR if you need real-time data.
    
    ❌ Forgetting that SSG build time grows with page count
       → 100K products × 500ms per page = 14 hours build!
         Use ISR with fallback: 'blocking' to build on demand.
    
    ❌ Not using fallback for new pages
       → New products added after build get 404.
         Use fallback: 'blocking' or true.
    
    ❌ Putting dynamic data in SSG pages
       → Prices that change hourly shouldn't be in SSG.
         Fetch dynamic data client-side after static shell loads.
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. SSG = Build HTML once at deploy time. Fastest possible serving.  ║
║     Perfect for content that rarely changes (blogs, docs, marketing) ║
║                                                                      ║
║  2. ISR = SSG + automatic background regeneration. Individual pages  ║
║     refresh without rebuilding the entire site.                      ║
║                                                                      ║
║  3. On-demand revalidation = Trigger page regeneration immediately   ║
║     when content changes (from CMS webhook, admin action, etc.)      ║
║                                                                      ║
║  4. fallback: 'blocking' = Pages not pre-built are generated on     ║
║     first request (SSR-like), then cached as static.                 ║
║                                                                      ║
║  5. SSG pages are the same for ALL users — don't use for            ║
║     personalized content. Fetch user-specific data client-side.      ║
║                                                                      ║
║  6. The sweet spot for most sites: SSG for stable pages +           ║
║     ISR for frequently changing pages + CSR for user-specific data   ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

SSG and ISR pages are just static files — perfect for serving from a **CDN (Content Delivery Network)**. In [Chapter 2.4: CDN for Frontend](./04-cdn-for-frontend.md), we'll learn how CDNs distribute your pages globally so users everywhere get lightning-fast load times.

---

[⬅️ Previous: CSR vs SSR](./02-csr-vs-ssr.md) | [⬆️ Index](../../00-INDEX.md) | [Next: CDN for Frontend ➡️](./04-cdn-for-frontend.md)
