# Chapter 2.7: Micro-Frontends — Breaking the UI into Independent Pieces

> **Level**: ⭐⭐⭐ Advanced  
> **What you'll learn**: How large companies split their monolithic frontend into independently developed, deployed, and maintained pieces — so multiple teams can ship features without stepping on each other.

---

## 🧠 Real-Life Analogy: Shopping Mall vs Single Store

**Monolith Frontend** = A single giant department store. One owner runs everything — clothing, electronics, food, furniture. If the clothing section needs renovation, the ENTIRE store shuts down.

**Micro-Frontend** = A shopping mall. Each shop is independent — different owners, different staff, different schedules. The food court can renovate without affecting the clothing store. New shops can open without disrupting existing ones.

```
    MONOLITH FRONTEND:
    ═════════════════
    
    ┌──────────────────────────────────────────────┐
    │             ONE Giant Application            │
    │                                              │
    │  Header + Navigation + Search + Products +   │
    │  Cart + Checkout + Reviews + Profile +        │
    │  Recommendations + Footer                    │
    │                                              │
    │  👥 ONE team owns EVERYTHING                 │
    │  🚀 ONE deployment for ANY change            │
    │  🔗 ALL code tightly coupled                 │
    │  💥 Bug in cart breaks the entire site!       │
    └──────────────────────────────────────────────┘
    
    
    MICRO-FRONTEND:
    ═══════════════
    
    ┌──────────────────────────────────────────────┐
    │  App Shell (thin container)                   │
    │  ┌──────────────┐  ┌──────────────────────┐  │
    │  │  Header MFE  │  │  Search MFE          │  │
    │  │  Team: Core  │  │  Team: Discovery     │  │
    │  │  React       │  │  Vue.js              │  │
    │  └──────────────┘  └──────────────────────┘  │
    │  ┌──────────────────────────────────────────┐│
    │  │  Product Listing MFE                     ││
    │  │  Team: Catalog    |  Framework: Angular  ││
    │  └──────────────────────────────────────────┘│
    │  ┌────────────────┐  ┌──────────────────────┐│
    │  │  Cart MFE      │  │  Recommendations MFE ││
    │  │  Team: Orders  │  │  Team: ML/AI         ││
    │  │  React         │  │  Svelte              ││
    │  └────────────────┘  └──────────────────────┘│
    │                                              │
    │  👥 Different teams own different pieces      │
    │  🚀 Deploy independently                     │
    │  🔗 Loosely coupled                          │
    │  💥 Cart bug only affects cart!               │
    └──────────────────────────────────────────────┘
```

---

## 📖 Why Micro-Frontends Exist — The Problem

```
    The Journey of a Growing Frontend:
    ══════════════════════════════════
    
    STARTUP (2 developers):
    ┌────────────────────────┐
    │  Simple SPA            │     Everything in one repo.
    │  React, 50 components  │     2 devs, easy to manage.
    │  Build: 30 seconds     │     ✅ This works fine!
    └────────────────────────┘
    
    GROWTH (20 developers):
    ┌────────────────────────────────────┐
    │  Monolith SPA                      │     20 devs in ONE codebase.
    │  React, 500 components             │     Merge conflicts everywhere!
    │  Build: 15 minutes                 │     "Who broke the build?"
    │  npm install: 10 minutes           │     Deploying is scary.
    │  Test suite: 45 minutes            │     ⚠️ Getting painful...
    └────────────────────────────────────┘
    
    SCALE (200 developers, 15 teams):
    ┌────────────────────────────────────────────┐
    │  Monster Monolith SPA                       │
    │  React, 3000+ components                    │
    │  Build: 1 hour+                             │
    │  Cannot upgrade React (everything breaks!)  │
    │  Teams waiting weeks for deploy slots        │
    │  One bug = entire site goes down             │
    │  New developer onboarding: 2 weeks           │
    │  ❌ THIS IS UNSUSTAINABLE!                  │
    └────────────────────────────────────────────┘
    
    SOLUTION: Break it into Micro-Frontends!
    ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
    │Search│ │ Cart │ │Prods.│ │ Auth │ │ Reco │
    │  MFE │ │ MFE  │ │ MFE  │ │ MFE  │ │ MFE  │
    │ Team │ │ Team │ │ Team │ │ Team │ │ Team │
    │  A   │ │  B   │ │  C   │ │  D   │ │  E   │
    └──────┘ └──────┘ └──────┘ └──────┘ └──────┘
    
    Each team: own repo, own build, own deploy, own framework choice!
```

---

## 🔧 Integration Approaches — How to Combine Micro-Frontends

### Approach 1: Build-Time Integration (npm packages)

```
    Each MFE is published as an npm package:
    
    npm install @mystore/header-mfe
    npm install @mystore/search-mfe
    npm install @mystore/product-mfe
    npm install @mystore/cart-mfe
    
    ┌──────────────────────────────────────────────┐
    │  Container App (imports all MFEs at BUILD)   │
    │                                              │
    │  import Header from '@mystore/header-mfe';   │
    │  import Search from '@mystore/search-mfe';   │
    │  import Products from '@mystore/product-mfe';│
    │  import Cart from '@mystore/cart-mfe';        │
    │                                              │
    │  function App() {                            │
    │    return (                                   │
    │      <>                                      │
    │        <Header />                            │
    │        <Search />                            │
    │        <Products />                          │
    │        <Cart />                              │
    │      </>                                     │
    │    );                                        │
    │  }                                           │
    └──────────────────────────────────────────────┘
    
    ⚠️ Problem: Updating ANY MFE requires rebuilding the container!
       Still coupled at build time. Not truly independent.
    
    ✅ Good for: shared UI libraries, design systems
    ❌ Bad for: independent deployment
```

### Approach 2: Runtime Integration via iframes

```
    Each MFE loads in a separate iframe:
    
    ┌──────────────────────────────────────────────┐
    │  Main Page                                   │
    │  ┌──────────────────────────────────────┐    │
    │  │  <iframe src="header.mystore.com">   │    │
    │  │  Header MFE                          │    │
    │  └──────────────────────────────────────┘    │
    │  ┌──────────────────────────────────────┐    │
    │  │  <iframe src="products.mystore.com"> │    │
    │  │  Product Listing MFE                 │    │
    │  └──────────────────────────────────────┘    │
    │  ┌──────────────────────────────────────┐    │
    │  │  <iframe src="cart.mystore.com">     │    │
    │  │  Cart MFE                            │    │
    │  └──────────────────────────────────────┘    │
    └──────────────────────────────────────────────┘
    
    ✅ Perfect isolation (each iframe = separate browser context)
    ✅ Truly independent (different domains even!)
    ❌ Terrible UX: no shared styling, hard to resize, accessibility issues
    ❌ Poor performance (each iframe = separate page load)
    ❌ Communication between iframes is awkward (postMessage)
```

### Approach 3: Runtime Integration via JavaScript (Module Federation) ⭐ RECOMMENDED

```
    Webpack Module Federation / ES Module Imports:
    Each MFE exposes components at RUNTIME
    
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  Container App (Shell):                                      │
    │  https://mystore.com                                        │
    │                                                              │
    │  At RUNTIME, dynamically loads MFEs from different servers: │
    │                                                              │
    │  ┌──────────────────────────────────────────────────────┐    │
    │  │  const Header = loadRemote('header-mfe/Header')     │    │
    │  │  ↓ Fetches from: https://header.mystore.com/mfe.js  │    │
    │  └──────────────────────────────────────────────────────┘    │
    │                                                              │
    │  ┌──────────────────────────────────────────────────────┐    │
    │  │  const Products = loadRemote('products-mfe/List')   │    │
    │  │  ↓ Fetches from: https://products.mystore.com/mfe.js│    │
    │  └──────────────────────────────────────────────────────┘    │
    │                                                              │
    │  Each MFE is deployed on its OWN server!                    │
    │  Container doesn't need to rebuild when MFEs update!        │
    │                                                              │
    │  ✅ Independent deployment                                   │
    │  ✅ No rebuild of container needed                           │
    │  ✅ Can share libraries (React loaded once)                  │
    │  ✅ Production-proven at scale                               │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Approach 4: Server-Side Composition

```
    Server assembles fragments from different MFE servers:
    
    Browser: GET /products
                │
                ▼
    ┌───────────────────────────────────┐
    │  Composition Server (Nginx/SSI   │
    │  or Podium/Tailor)               │
    │                                   │
    │  For /products page, fetch:       │
    │                                   │
    │  ┌─ GET header.internal/fragment  │───▶ <nav>...</nav>
    │  │                                │
    │  ├─ GET search.internal/fragment  │───▶ <div class="search">...</div>
    │  │                                │
    │  ├─ GET products.internal/fragment│───▶ <div class="products">...</div>
    │  │                                │
    │  └─ GET footer.internal/fragment  │───▶ <footer>...</footer>
    │                                   │
    │  Assembles all fragments into     │
    │  ONE HTML page and returns it     │
    └───────────────────────────────────┘
                │
                ▼
    Browser receives complete HTML page
    (looks like a normal MPA page!)
    
    ✅ SEO-friendly (full server-rendered HTML)
    ✅ Works without JavaScript
    ✅ Fast first paint
    ❌ More server infrastructure needed
    
    Used by: IKEA (Podium), Zalando (Tailor)
```

---

## 💻 Code Example: Module Federation (Webpack 5)

### Product MFE (Team A's repo)

```javascript
// products-mfe/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
    plugins: [
        new ModuleFederationPlugin({
            name: 'products_mfe',
            filename: 'remoteEntry.js',
            // EXPOSE these components to other apps
            exposes: {
                './ProductList': './src/ProductList',
                './ProductDetail': './src/ProductDetail',
            },
            // SHARE React so it's loaded only once
            shared: {
                react: { singleton: true },
                'react-dom': { singleton: true },
            },
        }),
    ],
};
```

```javascript
// products-mfe/src/ProductList.jsx
import React, { useState, useEffect } from 'react';

export default function ProductList() {
    const [products, setProducts] = useState([]);
    
    useEffect(() => {
        fetch('/api/products')
            .then(res => res.json())
            .then(setProducts);
    }, []);
    
    return (
        <div className="product-grid">
            {products.map(p => (
                <div key={p.id} className="product-card">
                    <img src={p.image} alt={p.name} />
                    <h3>{p.name}</h3>
                    <p>₹{p.price.toLocaleString()}</p>
                </div>
            ))}
        </div>
    );
}
```

### Container Shell (Main App)

```javascript
// shell/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
    plugins: [
        new ModuleFederationPlugin({
            name: 'shell',
            // CONSUME remote MFEs at runtime
            remotes: {
                products_mfe: 'products_mfe@https://products.mystore.com/remoteEntry.js',
                cart_mfe: 'cart_mfe@https://cart.mystore.com/remoteEntry.js',
                header_mfe: 'header_mfe@https://header.mystore.com/remoteEntry.js',
            },
            shared: {
                react: { singleton: true },
                'react-dom': { singleton: true },
            },
        }),
    ],
};
```

```javascript
// shell/src/App.jsx
import React, { Suspense, lazy } from 'react';

// Dynamically load MFEs at RUNTIME (not build time!)
const Header = lazy(() => import('header_mfe/Header'));
const ProductList = lazy(() => import('products_mfe/ProductList'));
const Cart = lazy(() => import('cart_mfe/MiniCart'));

export default function App() {
    return (
        <div>
            <Suspense fallback={<div>Loading header...</div>}>
                <Header />
            </Suspense>
            
            <main>
                <Suspense fallback={<div>Loading products...</div>}>
                    <ProductList />
                </Suspense>
            </main>
            
            <aside>
                <Suspense fallback={<div>Loading cart...</div>}>
                    <Cart />
                </Suspense>
            </aside>
        </div>
    );
}
```

---

## 🔄 Communication Between Micro-Frontends

```
    MFEs need to talk to each other sometimes:
    
    Example: User clicks "Add to Cart" in Product MFE
             → Cart MFE needs to update!
    
    
    APPROACH 1: Custom Events (Browser-native)
    ═══════════════════════════════════════════
    
    Product MFE:
    ┌──────────────────────────────────────────────┐
    │  function addToCart(product) {                │
    │    window.dispatchEvent(                      │
    │      new CustomEvent('cart:add', {            │
    │        detail: { productId: 42, qty: 1 }     │
    │      })                                      │
    │    );                                        │
    │  }                                           │
    └──────────────────────────────────────────────┘
                         │
                    Custom Event
                    (through DOM)
                         │
                         ▼
    Cart MFE:
    ┌──────────────────────────────────────────────┐
    │  window.addEventListener('cart:add', (e) => {│
    │    const { productId, qty } = e.detail;      │
    │    updateCart(productId, qty);                │
    │  });                                         │
    └──────────────────────────────────────────────┘
    
    
    APPROACH 2: Shared State (Redux, URL, etc.)
    ═══════════════════════════════════════════
    
    ┌──────────┐     ┌──────────────┐     ┌──────────┐
    │ Product  │────▶│ Shared Store │◀────│  Cart    │
    │   MFE    │     │ (e.g. Redux  │     │   MFE    │
    │          │     │  in shell)   │     │          │
    └──────────┘     └──────────────┘     └──────────┘
    
    ⚠️ Shared state creates coupling — use sparingly!
    
    
    BEST PRACTICE: Keep communication minimal!
    ═══════════════════════════════════════════
    
    MFEs should be as independent as possible.
    Only share: user auth state, cart count, language/theme.
    Each MFE calls its OWN backend API for data.
```

---

## 📊 Deployment Architecture

```
    Each MFE has its own CI/CD pipeline:
    
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │  Header MFE │     │ Product MFE │     │  Cart MFE   │
    │  Git Repo   │     │  Git Repo   │     │  Git Repo   │
    └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │  CI/CD      │     │  CI/CD      │     │  CI/CD      │
    │  Pipeline   │     │  Pipeline   │     │  Pipeline   │
    │  Build+Test │     │  Build+Test │     │  Build+Test │
    └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │  CDN        │     │  CDN        │     │  CDN        │
    │  header.    │     │  products.  │     │  cart.       │
    │  mystore.com│     │  mystore.com│     │  mystore.com │
    └─────────────┘     └─────────────┘     └─────────────┘
           │                   │                   │
           └───────────────────┼───────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Shell / Container   │
                    │  mystore.com         │
                    │                      │
                    │  Loads all MFEs at   │
                    │  runtime from their  │
                    │  respective CDNs     │
                    └──────────────────────┘
    
    Team A can deploy Header changes at 2 PM.
    Team B can deploy Cart changes at 4 PM.
    Neither requires the other to do anything!
```

---

## 🏢 Real-World Examples

```
    ┌──────────────┬──────────────────────────────────────────────────┐
    │  Company     │  How They Use Micro-Frontends                   │
    ├──────────────┼──────────────────────────────────────────────────┤
    │  Amazon      │  Each section (recommendations, search, cart,   │
    │              │  product detail) is an independent MFE owned by │
    │              │  a separate "two-pizza team" (~6-8 people)      │
    ├──────────────┼──────────────────────────────────────────────────┤
    │  IKEA        │  Uses "Podium" framework for server-side        │
    │              │  micro-frontend composition. Each page section  │
    │              │  fetched from different backend services.        │
    ├──────────────┼──────────────────────────────────────────────────┤
    │  Spotify     │  Desktop app built with independent "squads"    │
    │              │  owning separate UI sections (player, browse,   │
    │              │  search, playlists). Each deploys independently.│
    ├──────────────┼──────────────────────────────────────────────────┤
    │  Zalando     │  "Project Mosaic" — server-side composition.    │
    │              │  150+ teams shipping frontend independently.    │
    ├──────────────┼──────────────────────────────────────────────────┤
    │  SAP         │  "Luigi" micro-frontend framework. Enterprise   │
    │              │  apps composed from separate MFE modules.       │
    └──────────────┴──────────────────────────────────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Using micro-frontends with a small team (< 15 devs)
       → Massive overhead for little benefit. Monolith is fine!
       ✅ Only split when you have multiple teams with ownership boundaries
    
    ❌ Sharing too much state between MFEs
       → Tight coupling defeats the purpose!
       ✅ Each MFE should be as independent as possible
    
    ❌ Different MFEs loading duplicate libraries
       → User downloads React 3 times = 400KB wasted!
       ✅ Use shared dependencies (Module Federation's `shared` config)
    
    ❌ Inconsistent UI/UX across MFEs
       → Header looks different from footer, colors don't match
       ✅ Share a design system / component library
    
    ❌ No contract testing between MFEs
       → Cart MFE expects event format A, Product MFE sends format B
       ✅ Define and test event contracts between teams
    
    ❌ Over-engineering from day one
       → "We MIGHT need 50 teams someday!"
       ✅ Start monolith, split when organizational pain demands it
```

---

## ✅ When to Use / When NOT to Use

```
    USE Micro-Frontends:                    DON'T USE Micro-Frontends:
    ════════════════════                    ══════════════════════════
    ✅ 3+ teams working on same frontend   ❌ Small team (< 15 developers)
    ✅ Teams want to deploy independently   ❌ Simple application
    ✅ Different parts have different       ❌ No clear team boundaries
       release cadences                    ❌ Tight UX consistency needed
    ✅ Migrating legacy app incrementally  ❌ Greenfield project (start
       (replace section by section)            with monolith!)
    ✅ Enterprise with many autonomous
       teams
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. Micro-frontends split a monolith frontend into independent      ║
║     pieces, each owned, built, and deployed by a separate team.     ║
║                                                                      ║
║  2. Module Federation (Webpack 5) is the most popular approach —    ║
║     MFEs loaded at RUNTIME from separate servers.                    ║
║                                                                      ║
║  3. Communication between MFEs should be minimal. Use Custom        ║
║     Events for loose coupling. Avoid shared state.                   ║
║                                                                      ║
║  4. Share a design system to keep UI consistent across MFEs.        ║
║     Share core libraries (React) to avoid duplicate downloads.       ║
║                                                                      ║
║  5. This is an ORGANIZATIONAL solution, not a technical one.        ║
║     Only use when you have multiple teams with clear boundaries.    ║
║                                                                      ║
║  6. Start with a monolith. Split into micro-frontends only when     ║
║     team coordination becomes the bottleneck.                        ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

We've been talking about serving frontend from origin servers and CDN edge locations. But what if you could run actual JavaScript **logic** at the edge — not just serve files? In [Chapter 2.8: Edge Computing for Frontend](./08-edge-computing-frontend.md), we'll explore this cutting-edge approach.

---

[⬅️ Previous: SPA vs MPA](./06-spa-vs-mpa.md) | [⬆️ Index](../../00-INDEX.md) | [Next: Edge Computing for Frontend ➡️](./08-edge-computing-frontend.md)
